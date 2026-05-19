# Tier-5 syscall: listen(2) — listen for connections on a socket (syscall 50)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - net/socket.c (__sys_listen, sys_listen)
  - net/ipv4/af_inet.c (inet_listen)
  - net/ipv4/inet_connection_sock.c (inet_csk_listen_start)
  - net/unix/af_unix.c (unix_listen)
  - include/uapi/linux/socket.h (SOMAXCONN)
-->

## Summary

`listen(2)` marks a bound socket as passive — i.e., one that will accept incoming connection requests — and sets the maximum length of the pending-connection queue. It is syscall number **50** on x86-64.

The path is `sys_listen(fd, backlog)` → `__sys_listen()` → `sock.ops.listen(sock, backlog)`. The kernel clamps `backlog` to `sysctl_somaxconn` (default 4096) before invoking the per-AF listener-start handler. For TCP this transitions `sk_state` from `TCP_CLOSE` to `TCP_LISTEN` and initializes `icsk.icsk_accept_queue`.

Critical for: every server (`bind` → `listen` → `accept` loop), every container's listener fd, every socket-activated systemd unit (which performs `listen` in the init process and passes the fd).

This Tier-5 covers syscall **50** `listen(int sockfd, int backlog)`.

## Signature

```c
int listen(int sockfd, int backlog);
```

```rust
pub fn sys_listen(sockfd: i32, backlog: i32) -> SyscallResult<i32>;
```

x86-64 entry: `__x64_sys_listen` → `__sys_listen(fd, backlog)`.

## Parameters

- **`sockfd`** — bound socket fd of type `SOCK_STREAM` or `SOCK_SEQPACKET`.
- **`backlog`** — maximum number of pending connections (incomplete + completed). Clamped to `sysctl_somaxconn`. Negative values are clamped to 0 (and 0 means "implementation-defined small value", in practice the kernel uses backlog=0 as a literal "no queue beyond established" hint).

## Return

0 on success; `-1`/`-errno` on error.

## Errors

- `EBADF` — sockfd not open.
- `ENOTSOCK` — fd not a socket.
- `EOPNOTSUPP` — socket type doesn't support listen (SOCK_DGRAM, SOCK_RAW).
- `EADDRINUSE` — another socket is already listening on the same address (rare; usually caught at bind).
- `EINVAL` — socket already shut down; not bound; backlog deeply negative on some legacy paths; family mismatch.
- `EACCES` / `EPERM` — LSM denial.
- `ENOMEM` — accept-queue allocation failure.

## ABI surface

- `backlog` is an `int`. The kernel clamps via `(unsigned int)backlog > somaxconn ? somaxconn : backlog`, treating negative values as huge unsigned and thus clamping them down — so negative `backlog` is folded to `sysctl_somaxconn`.
- `SOMAXCONN` is `4096` in `include/uapi/linux/socket.h` (raised from 128 in v5.4); `sysctl_somaxconn` is the per-netns runtime ceiling.
- The "pending queue" combines incomplete (`SYN_RCVD`) and completed (`ESTABLISHED`-but-not-accepted) connections; tcp_max_syn_backlog further caps incompletes.

## Compatibility contract

REQ-1: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-2: somaxconn clamp:
- somaxconn = READ_ONCE(net.core.sysctl_somaxconn).
- if (unsigned)backlog > somaxconn: backlog = somaxconn.
- (negative-int reinterpreted as huge unsigned ⟹ clamped to somaxconn.)

REQ-3: Ops dispatch:
- if !sock.ops.listen: return `-EOPNOTSUPP`.
- err = security_socket_listen(sock, backlog); if err: return err.
- err = sock.ops.listen(sock, backlog); return err.

REQ-4: inet_listen (TCP):
- if sock.state != SS_UNCONNECTED ∨ sock.type != SOCK_STREAM: return `-EINVAL`.
- old_state = sk.sk_state.
- if old_state ∉ {TCP_CLOSE, TCP_LISTEN}: return `-EINVAL`.
- if !inet.inet_num: return `-EINVAL` (must be bound).
- WRITE_ONCE(sk.sk_max_ack_backlog, backlog).
- if old_state != TCP_LISTEN:
  - err = inet_csk_listen_start(sk, backlog); if err: return err.
  - tcp_call_bpf(sk, BPF_SOCK_OPS_TCP_LISTEN_CB, ...).
- return 0.

REQ-5: inet_csk_listen_start:
- reqsk_queue_alloc(&icsk.icsk_accept_queue).
- inet.inet_num and bind_hash retained.
- sk.sk_state = TCP_LISTEN.
- sk.sk_ack_backlog = 0.
- inet_csk_delack_init(sk).
- err = sk.sk_prot.hash(sk); if err:
  - reqsk_queue_destroy.
  - sk_state = TCP_CLOSE.
  - return err.
- return 0.

REQ-6: unix_listen:
- if sock.type != SOCK_STREAM ∧ sock.type != SOCK_SEQPACKET: return `-EOPNOTSUPP`.
- if u.addr == NULL: return `-EINVAL` (not bound).
- sk.sk_state = TCP_LISTEN.
- sk.sk_max_ack_backlog = backlog.

REQ-7: Audit:
- audit_log_socketcall(SYS_LISTEN, fd, NULL, 0, ret, backlog).

REQ-8: cgroup-bpf:
- BPF_SOCK_OPS_TCP_LISTEN_CB invoked after successful listen-start.

REQ-9: tcp_max_syn_backlog:
- separate sysctl bounds the SYN-RCVD half-open queue; not the same as somaxconn.

## Acceptance Criteria

- [ ] AC-1: listen on bound TCP socket: returns 0; sk_state == TCP_LISTEN.
- [ ] AC-2: listen with backlog > SOMAXCONN: clamped to sysctl_somaxconn.
- [ ] AC-3: listen with backlog == -1: clamped to sysctl_somaxconn.
- [ ] AC-4: listen on unbound socket: -EINVAL.
- [ ] AC-5: listen on SOCK_DGRAM: -EOPNOTSUPP.
- [ ] AC-6: listen on connected socket: -EINVAL.
- [ ] AC-7: second listen on same socket with new backlog: succeeds, updates sk_max_ack_backlog.
- [ ] AC-8: listen on UNIX SOCK_STREAM after bind: success; subsequent connect by client joins queue.
- [ ] AC-9: LSM denial: -EACCES.
- [ ] AC-10: hash insertion failure: state restored to TCP_CLOSE.
- [ ] AC-11: cgroup BPF TCP_LISTEN_CB invoked after success.

## Architecture

```
SysSocket::sys_listen(fd, backlog) -> SyscallResult<i32>
  1. sock = SocketFd::lookup_light(fd)?;
  2. somaxconn = net::sysctl_somaxconn(current_net());
  3. if (backlog as u32) > somaxconn: backlog = somaxconn as i32;
  4. listen_op = sock.ops.listen.ok_or(EOPNOTSUPP)?;
  5. Lsm::socket_listen(&sock, backlog)?;
  6. err = listen_op(&sock, backlog);
  7. Audit::log_socketcall(SYS_LISTEN, fd, None, 0, err, backlog);
  8. err.into()
```

`inet_listen(sock, backlog)`:
1. lock_sock(sk).
2. err = -EINVAL; if sock.state != SS_UNCONNECTED ∨ sock.type != SOCK_STREAM: goto out.
3. old = sk.sk_state; if old ∉ {TCP_CLOSE, TCP_LISTEN}: goto out.
4. if !inet.inet_num: goto out.
5. WRITE_ONCE(sk.sk_max_ack_backlog, backlog).
6. if old != TCP_LISTEN:
   - err = inet_csk_listen_start(sk, backlog); if err: goto out.
   - tcp_call_bpf(sk, BPF_SOCK_OPS_TCP_LISTEN_CB, ...).
7. err = 0.
8. out: release_sock(sk); return err.

`inet_csk_listen_start(sk, backlog)`:
1. reqsk_queue_alloc(&icsk.icsk_accept_queue, backlog).
2. inet_csk_delack_init(sk).
3. sk.sk_state = TCP_LISTEN.
4. sk.sk_ack_backlog = 0.
5. err = sk.sk_prot.hash(sk); if err:
   - reqsk_queue_destroy.
   - sk.sk_state = TCP_CLOSE.
   - return err.
6. return 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `listen_backlog_clamp` | INVARIANT | per-sys_listen: backlog passed to ops ≤ sysctl_somaxconn. |
| `listen_negative_clamped` | INVARIANT | per-sys_listen: (i32 < 0) cast to u32 > somaxconn ⟹ clamped. |
| `listen_must_be_bound` | INVARIANT | per-inet_listen: inet_num != 0 else EINVAL. |
| `listen_state_atomic` | INVARIANT | per-inet_csk_listen_start: hash failure ⟹ sk_state restored. |
| `listen_lockheld` | INVARIANT | per-inet_listen: lock_sock held across state transition. |

### Layer 2: TLA+

`uapi/syscalls/listen.tla`:
- States: validate, clamp, lsm, dispatch, hash, return.
- Models sk_state transitions TCP_CLOSE → TCP_LISTEN and TCP_LISTEN → TCP_CLOSE on hash failure.
- Properties:
  - `safety_listen_idempotent` — listen on already-listening socket only updates backlog; state stays TCP_LISTEN.
  - `safety_clamp_enforced` — never see effective backlog > somaxconn.
  - `safety_state_rollback_on_failure` — hash() failure ⟹ sk_state == TCP_CLOSE.
  - `liveness_terminates` — listen returns within bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| post: ret == 0 ∧ stream ⟹ sk.sk_state == TCP_LISTEN | `inet_listen` |
| post: ret == 0 ⟹ sk.sk_max_ack_backlog ≤ sysctl_somaxconn | `sys_listen` |
| post: hash failure ⟹ sk.sk_state == TCP_CLOSE | `inet_csk_listen_start` |
| pre: lock_sock held in inet_listen body | `inet_listen` |

### Layer 4: Verus/Creusot functional

Per-`inet_listen → inet_csk_listen_start → reqsk_queue_alloc → hash` semantic equivalence with `net/ipv4/inet_connection_sock.c`. SOMAXCONN-clamp matches `net/socket.c:__sys_listen` line-for-line.

## Hardening

- **`backlog` cast through `unsigned int` for clamp** — defense against per-negative-overflow into massive queue alloc.
- **`sysctl_somaxconn` is per-netns** — defense against per-cross-ns DOS on accept queue.
- **`tcp_max_syn_backlog` separate cap on half-open** — defense against per-SYN-flood.
- **State rollback on hash failure** — defense against per-half-listening sock.
- **LSM `socket_listen`** — defense against per-MAC bypass.
- **`inet_num` required before listen** — defense against per-listen-without-bind footgun.
- **`reqsk_queue_alloc` sized to clamped backlog** — defense against per-OOM-via-huge-queue.
- **`lock_sock` held across state transition** — defense against per-race-with-close.
- **BPF `TCP_LISTEN_CB` hook** — defense against per-uncontrolled-listener (cgroup-scoped policy).

## Grsecurity-PaX

- **PAX_RANDKSTACK** — per-syscall kstack base randomized.
- **PaX UDEREF** — no userland deref in `sys_listen` (only `sockfd`, `backlog` integers).
- **GRKERNSEC_NO_SIMULT_CONNECT** — peer-side rate limit prevents listener's accept-queue from being saturated by a single attacker; complements the listen backlog ceiling.
- **GRKERNSEC_BLACKHOLE** — bound-but-not-listening ports get no RST; only listening ports respond. Reduces port-scan signal.
- **GRKERNSEC_RANDNET on accept queue** — accept-queue dequeue order randomized within reqsk_queue_remove; combined with SYN-cookie path it prevents queue-position leak. The listener salt is initialized at `inet_csk_listen_start`.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — irrelevant on listen (already gated at socket/bind).
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — UNIX-domain `listen` strict: abstract-ns listener only accept'd by tasks in same user-ns under grsec hardened build; pathname listener subject to filesystem ACL + grsec chroot policy.
- **SOCK_CLOEXEC mandatory under suid** — listen does not create an fd; CLOEXEC enforced on the accepted children via `accept4(SOCK_CLOEXEC)` and grsec's `suid_strict_cloexec` policy.
- **SCM_RIGHTS recursion bound** — irrelevant on listen.
- **getsockopt info-leak prevention** — `SO_ACCEPTCONN` readback is a single boolean; no leak.

## Open Questions

- Should we lower `SOMAXCONN` ceiling default in `sysctl_somaxconn` for hardened builds (e.g. 512 instead of 4096)? Upstream remains 4096.
- For UNIX-domain SOCK_SEQPACKET listen, do we share the AF_UNIX listener path or split per-type? Upstream: shared `unix_listen` with per-type validation.

## Out of Scope

- `accept(2)` / `accept4(2)` — separate Tier-5s.
- TCP SYN-cookie path — Tier-3 `net/ipv4/tcp_input.md`.
- `inet_csk_accept` queue dequeue — Tier-3 `net/ipv4/inet_connection_sock.md`.
- Implementation code.
