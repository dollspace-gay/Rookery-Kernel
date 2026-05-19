# Tier-3: net/netfilter/nf-queue — userspace verdict handoff (NFQUEUE)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/netfilter/nf_queue.c
  - net/netfilter/nfnetlink_queue.c
  - net/netfilter/nfnetlink.c
  - include/uapi/linux/netfilter/nfnetlink_queue.h
-->

## Summary
Tier-3 design for the NFQUEUE infrastructure — the netfilter mechanism that hands off in-flight packets to userspace daemons via NETLINK_NETFILTER + NFNL_SUBSYS_QUEUE for verdict (accept / drop / mangle). When a netfilter rule emits `NF_QUEUE` verdict (with embedded queue number), kernel suspends the skb in a per-netns per-queue waiter list, sends an `NFQA_*`-encoded packet to subscribed userspace, and waits for the daemon's verdict reply. Used for:

- userspace IDS/IPS daemons (Suricata, Snort with NFQUEUE backend)
- per-packet machine-learning classification
- application-layer proxies that need per-packet authorization
- nftables `queue` expression / iptables `-j NFQUEUE` target
- libnetfilter_queue / scapy `nfqueue` testing harnesses

Two-layer architecture:
- `nf_queue.c` — generic queue framework: `nf_queue_entry` per-suspended-skb, per-netns per-queue waiter lists, verdict-application
- `nfnetlink_queue.c` — NETLINK transport: NFNL_SUBSYS_QUEUE messages, packet→userspace serialization, verdict reception

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/core.md` (NF_QUEUE verdict dispatch from hook framework), `net/netfilter/nfnetlink.c` shared NFNL transport.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Generic queue framework: `nf_queue_entry`, suspend/resume, per-netns per-queue waiters | `net/netfilter/nf_queue.c` |
| NETLINK transport: NFNL_SUBSYS_QUEUE, packet→userspace + verdict reception | `net/netfilter/nfnetlink_queue.c` |
| Generic NFNL framework | `net/netfilter/nfnetlink.c` |
| UAPI: `NFQA_*`, `NFQNL_MSG_*`, `NFQNL_CFG_CMD_*` | `include/uapi/linux/netfilter/nfnetlink_queue.h` |

## Compatibility contract

### NFQUEUE verdict encoding

Per `include/linux/netfilter.h`, NFQUEUE verdict is `NF_QUEUE_NR(num) << 16 | NF_QUEUE` where `num` is the 16-bit queue number. Used by:
- nftables: `queue num 5 bypass` expression
- iptables-legacy: `-j NFQUEUE --queue-num 5`

Userspace daemon binds to queue 5 to receive packets matching that rule.

### Per-netns per-queue waiter list

Each (netns, queue_num) tuple has:
- `nfnl_q.queue_lock` (spinlock)
- `nfnl_q.queue_total` (current packet count)
- `nfnl_q.copy_mode` (NFQNL_COPY_NONE / META / PACKET)
- `nfnl_q.copy_range` (max bytes per packet to send to userspace)
- `nfnl_q.flags` (FAIL_OPEN / CONNTRACK / GSO / UID_GID / SECCTX / VLAN)
- `nfnl_q.queue_maxlen` (default 1024 packets)
- `nfnl_q.peer_portid` (netlink portid of bound daemon)
- `nfnl_q.queue_list` (list of pending nf_queue_entry)

Identical layout for the first cache-line.

### `struct nf_queue_entry` (per-suspended-skb)

```c
struct nf_queue_entry {
    struct list_head list;
    struct sk_buff *skb;
    unsigned int id;             /* per-queue monotonic ID */
    unsigned int hook_index;
    struct nf_hook_state state;
    u16 size;
};
```

Layout-equivalent.

### NFQNL_MSG_* message types

Per `include/uapi/linux/netfilter/nfnetlink_queue.h`:

| Message | Direction | Operation |
|---|---|---|
| `NFQNL_MSG_PACKET` | kernel → user | Per-packet handoff |
| `NFQNL_MSG_VERDICT` | user → kernel | Per-packet verdict |
| `NFQNL_MSG_CONFIG` | user → kernel | Bind/unbind/configure queue |
| `NFQNL_MSG_VERDICT_BATCH` | user → kernel | Bulk verdict for performance |

Wire format byte-identical so libnetfilter_queue + Suricata + Snort + `nfq` Python bindings work unchanged.

### NFQA_* NLA attributes (per NFQNL_MSG_PACKET)

- `NFQA_PACKET_HDR` (struct nfqnl_msg_packet_hdr — packet ID + hw_protocol + hook)
- `NFQA_VERDICT_HDR` (verdict reply)
- `NFQA_MARK` (skb mark)
- `NFQA_TIMESTAMP` (skb tstamp)
- `NFQA_IFINDEX_INDEV` / `_OUTDEV` / `_PHYSINDEV` / `_PHYSOUTDEV`
- `NFQA_HWADDR` (sender MAC for received frames)
- `NFQA_PAYLOAD` (packet bytes — variable length per `copy_range`)
- `NFQA_CT` / `_CT_INFO` (conntrack metadata if FLAG_CONNTRACK)
- `NFQA_CAP_LEN` (original packet length when truncated)
- `NFQA_SKB_INFO` (skb metadata flags)
- `NFQA_EXP` (expectation tuple if applicable)
- `NFQA_UID` / `_GID` (sender uid/gid if FLAG_UID_GID)
- `NFQA_SECCTX` (LSM secctx if FLAG_SECCTX)
- `NFQA_VLAN` (VLAN tag if FLAG_VLAN)
- `NFQA_L2HDR` (L2 header if applicable)
- `NFQA_PRIORITY` (skb priority)
- `NFQA_CGROUPv1` / `_CGROUPv2` (cgroup classid + path)

Wire format byte-identical.

### Verdict semantics

```c
struct nfqnl_msg_verdict_hdr {
    __be32 verdict;     /* NF_ACCEPT / DROP / STOP / QUEUE / REPEAT / STOLEN */
    __be32 id;          /* matches NFQA_PACKET_HDR.packet_id */
};
```

Optional NFQA_PAYLOAD on verdict → packet content modified by daemon (mangling).

Verdict result applied to the suspended skb:
- ACCEPT → resume processing at next NF hook position (continue traversal)
- DROP → free skb
- QUEUE | new-num → re-queue to different queue
- REPEAT → re-run callback chain at this hook
- STOLEN → daemon owns it (don't free)

Identical mechanism.

### NFQNL_CFG_CMD_* (config commands)

```
NFQNL_CFG_CMD_NONE       = 0
NFQNL_CFG_CMD_BIND       = 1   /* bind a daemon to a queue number */
NFQNL_CFG_CMD_UNBIND     = 2
NFQNL_CFG_CMD_PF_BIND    = 3   /* bind to all queues for AF (deprecated) */
NFQNL_CFG_CMD_PF_UNBIND  = 4
```

Plus per-queue config:
- `NFQA_CFG_PARAMS` (copy_mode + copy_range)
- `NFQA_CFG_QUEUE_MAXLEN`
- `NFQA_CFG_MASK` (flags mask)
- `NFQA_CFG_FLAGS` (FAIL_OPEN / CONNTRACK / GSO / UID_GID / SECCTX / VLAN)

Identical UAPI.

### `NFQA_CFG_F_FAIL_OPEN`

When set, if queue is full (queue_total == queue_maxlen), kernel applies NF_ACCEPT and drops the packet from the queue (rather than NF_DROP). Used by daemons that prefer "fail open" behavior on overload.

### `NFQA_CFG_F_GSO`

Allow per-skb GSO segmentation to occur in userspace; kernel sends GSO-aggregated skb (faster but daemon must handle GSO).

### `nfnetlink_queue` and `nfnetlink_log` co-existence

Both subsystems share the NFNL framework (`nfnetlink.c`). NFNL_SUBSYS_QUEUE = 3; NFNL_SUBSYS_ULOG = 4 (cross-ref `nf-log.md`). Per-netns per-subsys handler registry.

## Requirements

- REQ-1: NFQUEUE verdict encoding `NF_QUEUE_NR(num) << 16 | NF_QUEUE` byte-identical.
- REQ-2: Per-netns per-queue waiter list with `queue_lock` + `queue_total` + `queue_maxlen`; identical defaults.
- REQ-3: `struct nf_queue_entry` first-cache-line layout-equivalent.
- REQ-4: NFQNL_MSG_* message types (PACKET / VERDICT / CONFIG / VERDICT_BATCH) parsed identically.
- REQ-5: NFQA_* NLA attributes (full list per the table) byte-identical wire format.
- REQ-6: Verdict semantics (ACCEPT / DROP / STOP / QUEUE-rebind / REPEAT / STOLEN) per upstream.
- REQ-7: NFQNL_CFG_CMD_* config commands + NFQA_CFG_PARAMS / QUEUE_MAXLEN / FLAGS / MASK identical.
- REQ-8: NFQA_CFG_F_FAIL_OPEN: when queue full, NF_ACCEPT instead of NF_DROP.
- REQ-9: NFQA_CFG_F_GSO: send GSO-aggregated skb without pre-segmentation.
- REQ-10: NFQA_CFG_F_CONNTRACK / UID_GID / SECCTX / VLAN: per-flag adds corresponding NFQA_* attribute to PACKET messages.
- REQ-11: Per-queue VERDICT_BATCH: bulk-verdict array with single-message ack; identical batch semantics.
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `iptables-legacy -A FORWARD -j NFQUEUE --queue-num 5` test: verdict-encoded NF_QUEUE | (5 << 16); userspace daemon bound to queue 5 receives PACKET messages with byte-identical NFQA_* encoding. (covers REQ-1, REQ-4, REQ-5)
- [ ] AC-2: ACCEPT verdict round-trip: send packet → daemon receives PACKET → daemon sends NFQNL_MSG_VERDICT NF_ACCEPT → packet resumes processing at next hook. (covers REQ-6)
- [ ] AC-3: DROP verdict test: daemon sends NF_DROP → kernel frees skb; tcpdump downstream of NF hook does not see packet. (covers REQ-6)
- [ ] AC-4: Mangling test: daemon receives PACKET with NFQA_PAYLOAD, modifies bytes, returns NFQA_PAYLOAD modified + NF_ACCEPT → kernel applies new payload to skb. (covers REQ-6)
- [ ] AC-5: queue_maxlen test: queue_maxlen=10; flood 11 packets → 11th: with FAIL_OPEN=0 dropped + counter; with FAIL_OPEN=1 ACCEPTed instead. (covers REQ-7, REQ-8)
- [ ] AC-6: GSO test: NFQA_CFG_F_GSO=1; GSO-aggregated skb of 64KB delivered as single PACKET message; daemon handles segmentation. (covers REQ-9)
- [ ] AC-7: CONNTRACK flag test: FLAG_CONNTRACK=1; PACKET messages include NFQA_CT + NFQA_CT_INFO with conntrack tuple + IP_CT_* state. (covers REQ-10)
- [ ] AC-8: VERDICT_BATCH test: daemon sends batch of 100 verdicts in a single NFQNL_MSG_VERDICT_BATCH message; all 100 packets resumed. (covers REQ-11)
- [ ] AC-9: SECCTX test: FLAG_SECCTX=1; PACKET messages include NFQA_SECCTX with LSM-supplied security context. (covers REQ-10)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::net::netfilter::nf_queue::Queue` — per-(netns, queue_num) waiter list
- `kernel::net::netfilter::nf_queue::Entry` — `struct nf_queue_entry` wrapper
- `kernel::net::netfilter::nf_queue::Suspend` — `nf_queue` (suspend skb in waiter list)
- `kernel::net::netfilter::nf_queue::Resume` — `nf_reinject` (apply verdict + continue)
- `kernel::net::netfilter::nf_queue::nfnl::Transport` — NFNL_SUBSYS_QUEUE handlers
- `kernel::net::netfilter::nf_queue::nfnl::PacketBuilder` — NFQA_* NLA build
- `kernel::net::netfilter::nf_queue::nfnl::VerdictParser` — NFQNL_MSG_VERDICT parser
- `kernel::net::netfilter::nf_queue::nfnl::VerdictBatch` — bulk-verdict
- `kernel::net::netfilter::nf_queue::nfnl::Config` — NFQNL_CFG_CMD_* + NFQA_CFG_*

### Locking and concurrency

- **Per-(netns, queue_num) `queue_lock`** (spinlock): protects waiter list + counters
- **`nfnl_lock`** (per-netns mutex): NFNL framework's per-subsys handler register
- **RCU**: per-queue handler lookup RCU-side
- **Per-skb refcount**: held while suspended in queue

### Error handling

- `Err(EINVAL)` — bad NLA / queue not bound
- `Err(EOVERFLOW)` — queue full (without FAIL_OPEN)
- `Err(ENOENT)` — verdict for unknown packet ID
- `Err(EBUSY)` — queue already bound to different daemon
- `Err(EPERM)` — non-CAP_NET_ADMIN

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-queue suspend (refcount-acquire on skb; list-add under queue_lock) | `kani::proofs::net::netfilter::nf_queue::suspend_safety` |
| Per-queue resume + verdict apply (no use-after-free of suspended skb) | `kani::proofs::net::netfilter::nf_queue::resume_safety` |
| NFQA_* NLA build (bound-checked accessors; payload truncation per copy_range) | `kani::proofs::net::netfilter::nf_queue::build_safety` |
| Verdict parser (per-NLA bounds) | `kani::proofs::net::netfilter::nf_queue::verdict_parse_safety` |
| FAIL_OPEN drop policy (queue overflow → ACCEPT vs DROP) | `kani::proofs::net::netfilter::nf_queue::fail_open_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-queue waiter list | `queue_total` = list-length always; never exceeds `queue_maxlen` (with FAIL_OPEN modulating drop policy) | `kani::proofs::net::netfilter::nf_queue::queue_invariants` |
| Per-entry `id` | per-queue monotonic increment; verdict reply must match a current pending id | `kani::proofs::net::netfilter::nf_queue::id_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Verdict-soundness theorem** via Verus — proves: ∀ suspended skb, exactly one of {verdict applied + skb resumed, queue removed via daemon-disconnect, drop on queue overflow} occurs in finite time.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-skb refcount held while suspended (queue holds extra ref) uses `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | NFNL_SUBSYS_QUEUE handler tables `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed-after-emit packet skbs cleared (carries flow data + LSM secctx) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-skb (cross-ref `net/skbuff.md`); per-queue_entry slab cache
- **CONSTIFY**: see above
- **USERCOPY**: NLA parsing uses bound-checked accessors (CVE-class verdict-overflow defense)
- **SIZE_OVERFLOW**: queue_total + copy_range arithmetic uses checked operators
- **KERNEXEC**: dispatch via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for NFQNL_CFG_CMD_BIND.
- LSM hook `security_skb_classify_flow` (already standard) applies to suspended skbs.
- Default useful GR-RBAC policy: deny NFQUEUE bind outside gradm-marked `firewall_admin` role; NFQUEUE bypass via daemon controlling verdicts is a powerful privilege.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

- PAX_USERCOPY: NFQNL_COPY_PACKET payload copy to userspace must use a whitelisted slab; size bounded by `queue->copy_range` before `copy_to_user()` so an oversized verdict cannot bleed adjacent skb pages.
- PAX_KERNEXEC: nf-queue verdict handlers and netlink dispatch run from .text only; no W^X violation; W+X mappings under nfnetlink are denied.
- PAX_RANDKSTACK: per-syscall stack base offset randomized for `setsockopt(NETLINK_LISTEN_ALL_NSID)` and verdict ioctl paths to defeat stack pivots from a malicious userspace verdict producer.
- PAX_REFCOUNT: `instance->use`, `nfnl_callback` and per-queue refcounts use saturating refcount_t; overflow on rapid NFQA_VERDICT_HDR replay aborts and SIGKILLs the producer.
- PAX_MEMORY_SANITIZE: skb clones queued to userspace are zeroed on free; `nfqnl_zcopy()` buffers scrubbed after `nfqnl_recv_verdict()` to prevent payload re-read via slab reuse.
- PAX_UDEREF: NFQA_PAYLOAD and NFQA_CT attribute pointers from `nlmsg_data()` validated under uderef before deref; no implicit kernel/user alias.
- PAX_RAP / kCFI: `struct nfqnl_msg_verdict_hdr` and `queue_handler` indirect calls (`outfn`) are kCFI-typed; an attacker substituting outfn with a non-matching prototype is trapped on call.
- GRKERNSEC_HIDESYM: `/proc/net/netfilter/nfnetlink_queue` hides kernel pointers; `instance` addresses redacted under `kptr_restrict=2`.
- GRKERNSEC_DMESG: nf_queue overflow / verdict-timeout messages rate-limited and gated behind CAP_SYSLOG so a flooder cannot probe drop policy from dmesg.
- Queue-handler PAX_RAP: `nf_register_queue_handler()` requires `const struct nf_queue_handler *` with kCFI-typed `outfn`/`nf_hook_drop`; runtime swap to a forged ops table is rejected.
- NFQNL_COPY_PACKET PAX_USERCOPY: skb linear+frag copy to nlmsg uses explicit usercopy whitelisted allocations; `nla_put()` boundary enforced against `queue->copy_range`.
- Per-net `nfnl_queue_net` and instance hash isolated per netns so a low-privileged userns producer cannot affect host queues.

Rationale: nf-queue exports raw network payloads to userspace and accepts authoritative verdicts back; combined PAX_USERCOPY, kCFI-typed `queue_handler` ops, and saturating refcounts close the canonical UAF/typo-confusion vectors documented across CVE-2017-7184-class bugs.

## Open Questions

(none — NFQUEUE wire format exhaustively specified by upstream + libnetfilter_queue regression-test corpus + Suricata interop coverage)

## Out of Scope

- Per-IDS daemon implementation details (userspace; out of scope)
- 32-bit-only paths
- Implementation code
