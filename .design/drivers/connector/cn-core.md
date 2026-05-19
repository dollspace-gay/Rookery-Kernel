# Tier-3: drivers/connector/{connector,cn_proc,cn_queue}.c — netlink connector + proc-events

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/connector/00-overview.md
upstream-paths:
  - drivers/connector/connector.c
  - drivers/connector/cn_proc.c
  - drivers/connector/cn_queue.c
  - include/uapi/linux/connector.h
  - include/uapi/linux/cn_proc.h
  - include/linux/connector.h
-->

## Summary

The connector is a thin abstraction on top of NETLINK_CONNECTOR (netlink family 11) that lets in-kernel producers ship structured messages — identified by a two-u32 `cb_id = (idx, val)` — to userspace listeners, unicast or multicast. Its only mainline consumer of real interest is `cn_proc` (CN_IDX_PROC), which broadcasts every `fork`/`exec`/`uid-change`/`gid-change`/`sid`/`ptrace-attach`/`comm-change`/`coredump`/`exit` as a `struct proc_event` to subscribed listeners — the kernel-side substrate behind `taskstats`-style accounting, antivirus daemons, container managers (`pid`-ns reconciliation), and host-IDS deployments.

This Tier-3 covers `drivers/connector/connector.c` (~315 lines: netlink socket setup, send_mult/unicast/multicast, callback bus), `cn_proc.c` (~480 lines: per-event `proc_*_connector` emitters + filter), and `cn_queue.c` (per-callback work queue).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cb_id` (`uapi/linux/connector.h`) | (idx, val) tuple identifying a connector channel | `drivers::connector::CbId` |
| `struct cn_msg` (`uapi/linux/connector.h`) | per-message header (id + seq + ack + len + flags + data[]) | `drivers::connector::CnMsg` |
| `struct cn_dev` (`drivers/connector/connector.c`) | per-subsystem netlink socket state | `drivers::connector::CnDev` |
| `cn_add_callback(id, name, callback)` | register kernel-side listener for an idx/val | `Subsystem::add_callback` |
| `cn_del_callback(id)` | unregister | `Subsystem::del_callback` |
| `cn_netlink_send(msg, portid, group, gfp_mask)` | unicast (portid != 0) or multicast (group != 0) | `Subsystem::send` |
| `cn_netlink_send_mult(msg, len, portid, group, gfp_mask, filter, filter_data)` | send-with-filter | `Subsystem::send_mult` |
| `proc_fork_connector(task)` (`cn_proc.c`) | broadcast PROC_EVENT_FORK | `cn_proc::fork` |
| `proc_exec_connector(task)` | broadcast PROC_EVENT_EXEC | `cn_proc::exec` |
| `proc_id_connector(task, which)` | broadcast PROC_EVENT_UID/GID | `cn_proc::id_change` |
| `proc_sid_connector(task)` | broadcast PROC_EVENT_SID | `cn_proc::sid` |
| `proc_ptrace_connector(task, ptrace_id)` | broadcast PROC_EVENT_PTRACE | `cn_proc::ptrace` |
| `proc_comm_connector(task)` | broadcast PROC_EVENT_COMM | `cn_proc::comm` |
| `proc_coredump_connector(task)` | broadcast PROC_EVENT_COREDUMP | `cn_proc::coredump` |
| `proc_exit_connector(task)` | broadcast PROC_EVENT_EXIT (+ PROC_EVENT_NONZERO_EXIT if exit_code != 0) | `cn_proc::exit` |

## Compatibility contract

REQ-1: NETLINK_CONNECTOR is a kernel-only netlink family (constant 11, defined in `uapi/linux/netlink.h`) — userspace creates `socket(AF_NETLINK, SOCK_DGRAM, NETLINK_CONNECTOR)`, binds to multicast group `nl_pid=0, nl_groups=1<<(CN_IDX_PROC-1)`, and reads `struct cn_msg`-wrapped events.

REQ-2: `cb_id` allocation is centrally registered in `uapi/linux/connector.h`: `CN_IDX_PROC=0x1`, `_CIFS=0x2`, `CN_W1_IDX=0x3`, `_V86D=0x4`, `_BB=0x5`, `CN_DST_IDX=0x6`, `_DM=0x7`, `_DRBD=0x8`, `CN_KVP_IDX=0x9`, `CN_VSS_IDX=0xa`. `CN_NETLINK_USERS=11`. The connector core treats unregistered idx values as listener-only; multicast groups are sparse.

REQ-3: Per-message payload bounded by `CONNECTOR_MAX_MSG_SIZE = 16384` bytes; longer messages refused at `cn_netlink_send_mult`.

REQ-4: Per-kernel-callback dispatch — `cn_add_callback(id, name, cb)` registers `cb(msg, nsp)`; inbound user→kernel message with matching `cb_id` is dispatched to the per-callback workqueue queued via `cn_queue.c`.

REQ-5: Multicast send (`cn_netlink_send` with `group=cb_id.idx`) reaches all listeners subscribed to that group; per-listener delivery best-effort under GFP_ATOMIC / GFP_NOWAIT (proc-events are emitted from atomic contexts like `fork`).

REQ-6: `cn_proc` is gated by `CONFIG_PROC_EVENTS=y`. Per-event emitter is a no-op when `proc_event_num_listeners == 0` (avoid the alloc + send when nobody is listening); the listener count is tracked via `PROC_CN_MCAST_LISTEN`/`_IGNORE` cn_proc-control messages.

REQ-7: `proc_event` carries `(what, cpu, timestamp_ns, event_data)` where `event_data` is a per-event union (`fork_proc_event`, `exec_proc_event`, `id_proc_event`, `sid_proc_event`, `ptrace_proc_event`, `comm_proc_event`, `coredump_proc_event`, `exit_proc_event`).

REQ-8: cn_proc emits with `pid` and `tgid` from the **init** PID namespace; per-pid-ns translation is the listener's responsibility (current upstream behavior; debated for container-aware use).

REQ-9: Per-listener filter at `cn_netlink_send_mult`: callback `filter(group, filter_data, ev_what, exit_code)` lets the listener pre-filter at the socket level via `setsockopt(SOL_NETLINK, NETLINK_LISTEN_ALL_NSID, …)` and `BPF_PROG_TYPE_NETLINK_FILTER`-style hooks.

REQ-10: Subscription control via `PROC_CN_MCAST_LISTEN`/`_IGNORE` messages — userspace explicitly opts in/out, incrementing `proc_event_num_listeners` atomic counter.

## Acceptance Criteria

- [ ] AC-1: `CONFIG_CONNECTOR=y` + `CONFIG_PROC_EVENTS=y` build succeeds; `lsmod | grep cn_` (or builtin) confirms presence.
- [ ] AC-2: Test program subscribes to NETLINK_CONNECTOR group `1<<(CN_IDX_PROC-1)`, sends `PROC_CN_MCAST_LISTEN`, observes PROC_EVENT_FORK on `fork()` from another shell.
- [ ] AC-3: PROC_EVENT_EXEC fires on `execve()`; carries correct `process_pid`/`process_tgid`.
- [ ] AC-4: PROC_EVENT_EXIT + PROC_EVENT_NONZERO_EXIT fires on `exit(1)`; carries correct `exit_code`/`exit_signal`.
- [ ] AC-5: PROC_EVENT_UID/GID fires on `setuid()`/`setgid()`.
- [ ] AC-6: PROC_EVENT_PTRACE fires on `ptrace(PTRACE_ATTACH)`.
- [ ] AC-7: Subscribing as unprivileged user (no CAP_NET_ADMIN) refused (`bind` returns -EPERM under hardened policy; cross-ref grsec section).
- [ ] AC-8: `CONNECTOR_MAX_MSG_SIZE+1` byte send returns -EMSGSIZE; no kernel oops.
- [ ] AC-9: Listener-count gate observed: with `proc_event_num_listeners==0`, no `proc_*_connector` emission allocates skb (verified via tracepoint).

## Architecture

`Subsystem` lives in `drivers::connector::Subsystem`:

```
struct Subsystem {
  dev: CnDev,
  callback_list: RwLock<Vec<Callback>>, // per-cb_id kernel-listener registry
  wq: Workqueue,
}

struct CnDev {
  nls: NonNull<NetlinkSock>,           // NETLINK_CONNECTOR sock
  input: fn(skb: SkbRef),              // cn_rx_skb
}

struct Callback {
  id: CbId,
  name: KString,
  cb: fn(msg: &CnMsg, nsp: &NetlinkSkbParms),
}

struct CnProc {
  num_listeners: AtomicI32,            // proc_event_num_listeners
  msg_buffer: CpuLocal<[u8; CN_PROC_MSG_SIZE]>,  // 8-byte-aligned per-CPU staging
}
```

Initialization (`cn_init`):
1. Create NETLINK_CONNECTOR socket via `netlink_kernel_create(net, NETLINK_CONNECTOR, &cfg)`.
2. Register `cn_rx_skb` as input handler — dispatches inbound user→kernel messages to per-`cb_id` callback.
3. Allocate per-cpu work-queue for callback dispatch.

`cn_proc` initialization (`cn_proc_init`):
1. `cn_add_callback({CN_IDX_PROC, CN_VAL_PROC}, "cn_proc", cn_proc_mcast_ctl)` — registers the listener-control handler.
2. Initialize `proc_event_num_listeners` to 0.

Per-event emit (e.g. `proc_fork_connector(task)`):
1. Read `atomic_read(&proc_event_num_listeners)` — return early if zero.
2. Get per-CPU `msg_buffer` (CN_PROC_MSG_SIZE = `sizeof(cn_msg) + sizeof(proc_event) + 4`).
3. Populate `cn_msg.id = {CN_IDX_PROC, CN_VAL_PROC}`, `seq = atomic_inc_return(&seq)`, `ack = 0`, `len = sizeof(proc_event)`.
4. Populate `proc_event.what = PROC_EVENT_FORK`, `.cpu = smp_processor_id()`, `.timestamp_ns = ktime_get_ns()`, `.fork.parent_pid = task->real_parent->pid`, `.fork.parent_tgid = task->real_parent->tgid`, `.fork.child_pid = task->pid`, `.fork.child_tgid = task->tgid`.
5. `cn_netlink_send_mult(msg, msg->len, 0, CN_IDX_PROC, GFP_NOWAIT, cn_filter, filter_data)`.

`cn_netlink_send_mult(msg, len, portid, group, gfp_mask, filter, filter_data)`:
1. Validate `len + sizeof(*msg) ≤ CONNECTOR_MAX_MSG_SIZE`.
2. Build outer `nlmsghdr` with `NLMSG_DONE` type; embed `cn_msg` + payload.
3. If `portid != 0`: `netlink_unicast(nls, skb, portid, MSG_DONTWAIT)`.
4. Else: `netlink_broadcast_filtered(nls, skb, 0, group, gfp_mask, filter, filter_data)`.

Inbound dispatch (`cn_rx_skb`):
1. Validate `nlmsghdr` size + type.
2. Extract `cn_msg` header; reject if `msg->len > CONNECTOR_MAX_MSG_SIZE`.
3. Look up callback by `cn_msg.id`; refuse if not registered.
4. Queue work to per-callback workqueue: `cb(msg, nsp)`.

`cn_proc_mcast_ctl(msg, nsp)`:
1. Permission check — CAP_NET_ADMIN in nsp's user-ns (cross-ref grsec hardening below; mainline post-5.x).
2. Decode `enum proc_cn_mcast_op` — `PROC_CN_MCAST_LISTEN` increments `proc_event_num_listeners`; `_IGNORE` decrements (floored at 0).

`cn_filter(dsk, skb, group, data)` — per-listener filter applied to each broadcast recipient:
1. Decode listener's `nl_groups` to confirm group subscription.
2. Look up per-listener filter array (set via `setsockopt`).
3. Match against `filter_data[0]` (event `what`) and `filter_data[1]` (exit_code for EXIT events).
4. Return true → deliver; false → skip.

cn_proc emitter table (per-event helper):

| Helper | Event | Fields populated |
|---|---|---|
| `proc_fork_connector` | PROC_EVENT_FORK | parent_pid/tgid, child_pid/tgid |
| `proc_exec_connector` | PROC_EVENT_EXEC | process_pid/tgid |
| `proc_id_connector` | PROC_EVENT_UID or _GID | process_pid/tgid + ruid/euid (or rgid/egid) |
| `proc_sid_connector` | PROC_EVENT_SID | process_pid/tgid |
| `proc_ptrace_connector` | PROC_EVENT_PTRACE | process_pid/tgid + tracer_pid/tgid |
| `proc_comm_connector` | PROC_EVENT_COMM | process_pid/tgid + comm[16] |
| `proc_coredump_connector` | PROC_EVENT_COREDUMP | process_pid/tgid + parent_pid/tgid |
| `proc_exit_connector` | PROC_EVENT_EXIT (+ NONZERO_EXIT if nonzero) | process_pid/tgid + exit_code + exit_signal + parent_pid/tgid |

## Hardening

- **CN_IDX allowlist** — kernel-side `cn_add_callback` callers limited to `CN_NETLINK_USERS` defined channels; unknown idx refused.
- **Per-message bound** — `CONNECTOR_MAX_MSG_SIZE = 16384` enforced on both send paths.
- **Listener-count gate** — emitters short-circuit at `proc_event_num_listeners == 0` to avoid kernel-side cost when nobody is listening.
- **Per-callback workqueue** — inbound user-msg dispatch decoupled from netlink-rx softirq.
- **Subscription control gated** — `PROC_CN_MCAST_LISTEN`/`_IGNORE` require CAP_NET_ADMIN (mainline hardening, lockdown patches).
- **Per-emit GFP_NOWAIT** — proc-events emitted from atomic contexts (fork/exec exit paths); refuse blocking alloc.
- **Filter-pre-deliver** — per-listener BPF filter prevents wakeup-storm from a chatty event-stream.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `cn_msg`, `proc_event` skb head/tail, and per-callback work-item allocations; per-comm-event 16-byte `comm[]` copy bounded.
- **PAX_KERNEXEC** — connector + cn_proc code in W^X kernel text; `callback_list` registry, `cn_proc_event_id`, and per-emitter dispatch live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `cn_netlink_send_mult`, `proc_*_connector` emitters, and `cn_rx_skb` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on netlink-sock state, per-callback work-items, and `proc_event_num_listeners` (saturating, never wraps); overflow trap defeats subscribe/unsubscribe race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for proc_event staging buffers, per-cpu emit scratch, and netlink skbs so PID/comm/exit_code traces cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on inbound `cn_rx_skb` user-buffer dereferences; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `cb_id` → callback dispatch, `nls->sk_user_data->input` netlink-input function, and `filter` callback indirect dispatch marked `__ro_after_init` with kCFI-typed signatures.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-`cn_dev` pointer disclosure behind CAP_SYSLOG; suppress `%p` in connector tracepoints.
- **GRKERNSEC_DMESG** — restrict connector registration banners + listener-count probes to CAP_SYSLOG so attackers cannot trivially probe which subsystems use the connector.
- **CN_IDX allowlist enforced** — `cn_add_callback` and inbound dispatch refuse idx values outside the `CN_NETLINK_USERS` table; refuse runtime addition outside boot path.
- **Payload PAX_USERCOPY whitelist** — connector skb cache marked usercopy-whitelist only over the precise `cn_msg + payload` window; refuse out-of-window copies.
- **Multicast subscription CAP_NET_ADMIN** — `bind` to NETLINK_CONNECTOR group `1<<(CN_IDX_PROC-1)` (and every other group) requires CAP_NET_ADMIN in the binder's user-ns; `PROC_CN_MCAST_LISTEN`/`_IGNORE` from non-CAP_NET_ADMIN sockets refused.
- **proc-events GRKERNSEC_HIDESYM (PID hiding)** — `proc_event.process_pid`/`_tgid`/`parent_pid`/`tracer_pid`/`comm[16]`/`exit_code` disclosure restricted to listeners whose effective uid can already see the source task under `/proc` hidepid policy; cross-uid pid leakage refused. Subscribing in non-init PID-ns sees only ns-local PIDs (or nothing).
- **Per-listener-filter mandatory for non-CAP** — BPF filter array must be installed before broadcast group bind; refuse subscription otherwise.

Rationale: cn_proc is essentially a fanout of every process-lifecycle event in the kernel — fork, exec, uid/gid change, ptrace, exit, comm, coredump. Without CAP_NET_ADMIN on the multicast bind, any unprivileged process can passively observe the global process table, including PIDs from other containers, comm[] changes (which leak process identity transitions), and exit codes. Combined with GRKERNSEC_HIDESYM-style PID hiding and per-listener BPF filter mandatory, the connector becomes a privileged-only observability channel. RAP/kCFI on the callback dispatch, refcount-overflow on subscription state, and PAX_USERCOPY whitelist on the cn_msg+payload window close the rest of the surface.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cn_msg_len_bounded` | OOB | `cn_msg.len + sizeof(cn_msg) ≤ CONNECTOR_MAX_MSG_SIZE`. |
| `cb_lookup_no_oob` | OOB | callback list indexed by `cb_id`; refuse unregistered. |
| `proc_event_buffer_no_overflow` | OOB | per-cpu `msg_buffer` sized `CN_PROC_MSG_SIZE`; per-event struct fits. |
| `listener_count_no_underflow` | OOB | `proc_event_num_listeners` floored at 0 on IGNORE. |

### Layer 2: TLA+

`models/connector/listener_subscribe.tla` (this doc declares it): proves PROC_CN_MCAST_LISTEN ↔ IGNORE counter never underflows, even under concurrent subscribe/unsubscribe and concurrent emit-with-listener-check.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Subsystem::send` post: `len ≤ CONNECTOR_MAX_MSG_SIZE` ∧ skb-cb_id matches input | `cn_netlink_send_mult` |
| Per-callback dispatch pre: `cb_id` registered ∧ `msg->len ≤ CONNECTOR_MAX_MSG_SIZE` | `cn_rx_skb` |
| `proc_*_connector` post: emission no-op iff `proc_event_num_listeners == 0` | every `cn_proc` emitter |

### Layer 4: Verus/Creusot functional

`fork()` → `proc_fork_connector(task)` → listener-count > 0 → cn_msg + proc_event populated → `cn_netlink_send_mult(group=CN_IDX_PROC)` → `netlink_broadcast_filtered` → per-listener BPF filter → recv on subscribed socket → userspace ev. Encoded as Verus invariant chained with `kernel/fork.md` and `fs/exec.md`.

## Out of Scope

- Other connector consumers (`cn_cifs`, `w1`, `dm`, `drbd`, HyperV KVP/VSS) — future per-consumer Tier-3s if needed
- Netlink core (covered by `net-netlink-core.md` future Tier-3)
- BPF socket filter (covered by `bpf-socket-filter.md` future Tier-3)
- 32-bit-only paths
- Implementation code
