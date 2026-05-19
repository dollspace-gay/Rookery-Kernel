# Tier-5 syscall: sendmmsg(2) — send multiple messages on a socket in one syscall (syscall 307)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - net/socket.c (__sys_sendmmsg, sys_sendmmsg, ___sys_sendmsg)
  - net/socket.c (copy_msghdr_from_user, sendmsg_copy_msghdr)
  - net/core/scm.c (scm_send)
  - include/linux/socket.h (struct mmsghdr, struct msghdr)
-->

## Summary

`sendmmsg(2)` is the compound-send primitive: it takes an array of `struct mmsghdr` (each wraps a `struct msghdr` + a per-message `msg_len` writeback field) and sends them in a single syscall, returning the number of messages successfully sent. This amortizes one syscall entry/exit over many sends, dramatically improving throughput for high-rate UDP/QUIC/DPDK-style workloads. It is syscall number **307** on x86-64.

The path is `sys_sendmmsg(fd, mmsg, vlen, flags)` → `__sys_sendmmsg()` → loop: per-iteration call `___sys_sendmsg(sock, &msg, &iov, flags, /*forbid_compat=*/false)` (i.e. the inner `sendmsg(2)` body sans fd lookup) and write `msg_len` back to user; on error after at least one message, return the count sent so far.

Critical for: QUIC servers, DNS resolvers, log-shippers, multicast publishers; any workload with many short outbound packets. Modern container networking stacks rely on sendmmsg for performance.

This Tier-5 covers syscall **307** `sendmmsg(int sockfd, struct mmsghdr *msgvec, unsigned int vlen, int flags)`.

## Signature

```c
struct mmsghdr {
    struct msghdr msg_hdr;
    unsigned int  msg_len;   // bytes sent for this message (kernel-written)
};

int sendmmsg(int sockfd, struct mmsghdr *msgvec, unsigned int vlen, int flags);
```

```rust
pub fn sys_sendmmsg(sockfd: i32, msgvec: UserPtr<Mmsghdr>, vlen: u32, flags: i32)
    -> SyscallResult<i32>;
```

x86-64 entry: `__x64_sys_sendmmsg` → `__sys_sendmmsg(fd, mmsg, vlen, flags)`.

## Parameters

- **`sockfd`** — fd from `socket(2)`.
- **`msgvec`** — userland array of `struct mmsghdr`; kernel reads each `msg_hdr` and writes `msg_len`.
- **`vlen`** — number of entries in `msgvec`. Capped to `UIO_MAXIOV` (1024) — values above are clamped silently to 1024 (upstream Linux behavior).
- **`flags`** — `MSG_*` flags applied to every message: `MSG_DONTWAIT`, `MSG_NOSIGNAL`, `MSG_CONFIRM`, `MSG_MORE`, `MSG_DONTROUTE`, `MSG_ZEROCOPY`, plus the compound-only `MSG_BATCH`.

## Return

Number of messages sent on success (≥ 1), `-errno` on error when zero messages were sent. **Partial-success semantics**: if some messages succeeded and a later one failed, the return value is the count of successful messages (≥ 1) — the error is silently swallowed for this call but the socket's `SO_ERROR` may reflect it.

## Errors

- `EBADF` — sockfd not open.
- `ENOTSOCK` — fd not a socket.
- `EFAULT` — msgvec pointer fault.
- `EINVAL` — vlen == 0; per-message msg_iovlen > UIO_MAXIOV; msg_controllen > INT_MAX; flag invalid.
- `EAGAIN` / `EWOULDBLOCK` — non-blocking send would block on first message.
- `EMSGSIZE` — datagram too large.
- `EPIPE` — stream peer closed.
- `ENOMEM` — slab alloc fail.
- `ENOBUFS` — socket buffer full.
- `EHOSTUNREACH` / `ENETUNREACH` — route error.
- `EINTR` — interrupted on first message (mid-loop interrupt yields partial count).
- `EPERM` — cgroup-BPF denial.
- `ETOOMANYREFS` — SCM_RIGHTS recursion limit exceeded on first message.

## ABI surface

- `struct mmsghdr` is 4 bytes larger than `struct msghdr` (the `msg_len` writeback field).
- `vlen` is `u32`; clamped to `UIO_MAXIOV` (1024) **per call**. This is the **sendmmsg compound-bound limit** (central grsec consideration).
- `flags` apply to all messages uniformly.
- On the first message's failure: return `-errno`. On later message's failure: return count-so-far (≥ 1).
- Compatibility variant: `compat_sys_sendmmsg` uses `compat_mmsghdr` with 32-bit pointer widths.

## Compatibility contract

REQ-1: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-2: vlen clamp:
- if vlen == 0: return 0 (Linux behavior; not an error).
- if vlen > UIO_MAXIOV (1024): vlen = UIO_MAXIOV.

REQ-3: For i in 0..vlen:
- err = ___sys_sendmsg(sock, &mmsg[i].msg_hdr, &iov, flags, /*forbid_compat=*/false).
- if err < 0:
  - if i == 0: return err.
  - else: return i.  // partial success
- put_user(err, &mmsg[i].msg_len).

REQ-4: Per-message inner ___sys_sendmsg:
- err = copy_msghdr_from_user(&msg, &mmsg[i].msg_hdr, &uaddr_storage, &iov, /*save_addr=*/true).
- if err: return err.
- if msg.msg_iovlen > UIO_MAXIOV: return -EMSGSIZE.
- err = import_iovec(WRITE, msg.msg_iov, msg.msg_iovlen, ARRAY_SIZE(iovstack), &iov, &msg.msg_iter).
- if err: return err.
- err = scm_send for cmsg.
- err = security_socket_sendmsg.
- err = BPF_CGROUP_RUN_PROG_INET_EGRESS.
- err = sock.ops.sendmsg(sock, &msg, total_len).
- on error: return err.

REQ-5: Audit:
- audit_log_socketcall(SYS_SENDMMSG, fd, NULL, 0, ret, flags).

REQ-6: signal handling:
- per-iteration: if signal_pending(current) ∧ i > 0: break loop; return i.
- if signal_pending ∧ i == 0: return -EINTR.

## Acceptance Criteria

- [ ] AC-1: sendmmsg(UDP, mmsg×8, 8, 0): all 8 sent; return 8; each msg_len populated.
- [ ] AC-2: sendmmsg(connected-TCP, mmsg×4, 4, 0): all 4 sent; return 4.
- [ ] AC-3: sendmmsg vlen = 0: return 0.
- [ ] AC-4: sendmmsg vlen = 2000: clamped to 1024.
- [ ] AC-5: sendmmsg msgvec pointer fault: -EFAULT (before any send).
- [ ] AC-6: sendmmsg first message fails (-EAGAIN): return -EAGAIN.
- [ ] AC-7: sendmmsg messages 0..2 succeed, message 3 fails: return 3.
- [ ] AC-8: sendmmsg interrupted by signal after 2 sends: return 2.
- [ ] AC-9: sendmmsg with MSG_FASTOPEN: only first message uses TFO; subsequent on TCP are EISCONN.
- [ ] AC-10: sendmmsg per-message msg_iovlen > UIO_MAXIOV on i=0: -EMSGSIZE.
- [ ] AC-11: sendmmsg with SCM_RIGHTS in message i, fd-recursion-limit hit: i==0 ⟹ -ETOOMANYREFS; else return i.
- [ ] AC-12: cgroup-BPF egress denial of message 1: messages 0 sent; return 1.

## Architecture

```
SysSocket::sys_sendmmsg(fd, msgvec, vlen, flags) -> SyscallResult<i32>
  1. sock = SocketFd::lookup_light(fd)?;
  2. if vlen == 0 { return Ok(0); }
  3. let n = core::cmp::min(vlen as usize, UIO_MAXIOV);
  4. let mut sent: i32 = 0;
  5. for i in 0..n {
        // signal check before each iteration
        if Task::signal_pending() {
            if sent > 0 { return Ok(sent); }
            return Err(EINTR);
        }
        let mmsg_ptr = unsafe { msgvec.add(i) };
        let bytes = match Self::sys_sendmsg_inner(&sock, &mmsg_ptr.msg_hdr, flags, false) {
            Ok(b) => b,
            Err(e) if sent > 0 => return Ok(sent),
            Err(e) => return Err(e),
        };
        UserCopy::put_user(bytes as u32, &mut mmsg_ptr.msg_len)?;
        sent += 1;
     }
  6. Audit::log_socketcall(SYS_SENDMMSG, fd, &(), 0, sent, flags);
  7. Ok(sent)
```

Per-iteration inner `sys_sendmsg_inner(sock, msg_hdr, flags, forbid_compat)`:
- equivalent to the body of `sys_sendmsg` but without the fd lookup (caller already holds sock).
- builds local iov[] on stack (UIO_FASTIOV = 8 fast-path slots).
- on overflow allocates iov heap with `kmalloc(msg_iovlen * sizeof(iovec), GFP_KERNEL)`.
- on return, free iov heap.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sendmmsg_vlen_clamp` | INVARIANT | per-sys_sendmmsg: vlen clamped to UIO_MAXIOV (1024). |
| `sendmmsg_first_error_propagated` | INVARIANT | per-loop: i == 0 ∧ err < 0 ⟹ return -err. |
| `sendmmsg_partial_success` | INVARIANT | per-loop: i > 0 ∧ err < 0 ⟹ return i. |
| `sendmmsg_msg_len_writeback` | INVARIANT | per-iteration: success ⟹ put_user(bytes, &mmsg[i].msg_len) before incrementing sent. |
| `sendmmsg_per_msg_iov_bound` | INVARIANT | per-iteration: msg.msg_iovlen ≤ UIO_MAXIOV. |
| `sendmmsg_per_msg_controllen_bound` | INVARIANT | per-iteration: msg.msg_controllen ≤ INT_MAX. |
| `sendmmsg_scm_rights_bound` | INVARIANT | per-iteration: SCM_RIGHTS fd count ≤ SCM_MAX_FD. |
| `sendmmsg_signal_check` | INVARIANT | per-loop: signal_pending checked at every iteration boundary. |

### Layer 2: TLA+

`uapi/syscalls/sendmmsg.tla`:
- States: validate, per-message-copy, per-message-iov, per-message-scm, per-message-dispatch, per-message-writeback, loop-advance, signal-check, return.
- Properties:
  - `safety_partial_progress_atomic` — sent count never decreases mid-call.
  - `safety_first_failure_propagated` — i == 0 failure ⟹ return -err; never partial.
  - `safety_late_failure_returns_count` — i > 0 failure ⟹ return i.
  - `safety_signal_interleave` — signal_pending at iteration ⟹ either break-with-count or EINTR-when-empty.
  - `liveness_terminates` — sendmmsg returns within bounded vlen steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: vlen clamped to UIO_MAXIOV | `sys_sendmmsg` |
| pre: per-iteration msg.msg_iovlen ≤ UIO_MAXIOV | `sys_sendmsg_inner` |
| post: ret ≥ 0 ⟹ first ret messages successfully sent | `sys_sendmmsg` |
| post: ret > 0 ⟹ msg_len[0..ret] populated | `sys_sendmmsg` |
| post: ret < 0 ⟹ zero messages sent (no msg_len writeback) | `sys_sendmmsg` |
| post: signal-interrupt mid-loop ⟹ return current sent count or EINTR if zero | `sys_sendmmsg` |

### Layer 4: Verus/Creusot functional

Per-`__sys_sendmmsg` semantic equivalence with `net/socket.c`. Per-iteration `___sys_sendmsg` call shares all sub-invariants with the standalone `sendmsg(2)` Tier-5. Signal-check timing matches upstream (between messages, not within a single dispatch).

## Hardening

- **`vlen` clamp to UIO_MAXIOV (1024)** — defense against per-vlen DoS (caller can't pin kernel in loop indefinitely). **This is the sendmmsg compound-bound limit.**
- **Per-iteration signal check** — defense against per-uninterruptible-loop with adversarial flag combinations (e.g. MSG_MORE with full queue).
- **Per-message `msg_iovlen ≤ UIO_MAXIOV` revalidation** — defense against per-message-iov-explosion (each loop iteration must independently re-check).
- **Per-message `msg_controllen ≤ INT_MAX` revalidation** — defense against per-message-cmsg-bomb.
- **Per-message `SCM_MAX_FD` revalidation** — defense against per-message fd-blast.
- **Per-message `MAX_FD_NESTING` recursion bound revalidation** — defense against per-message FD-passing-loop DoS.
- **First-message-failure propagates errno** — defense against per-silent-failure (caller sees the real error).
- **Late-message-failure returns count** — defense against per-state-loss when callers cannot replay (partial-success semantics).
- **`put_user(msg_len)` before incrementing sent** — defense against per-state-skew between kernel sent counter and user-visible msg_len[].
- **Per-iteration `cgroup-BPF inet_egress`** — defense against per-policy-bypass via batched sends.
- **Per-iteration LSM `socket_sendmsg`** — defense against per-MAC bypass via batched sends.
- **No fd-table mutation between iterations** — defense against per-TOCTOU on the socket fd mid-call.

## Grsecurity-PaX

- **PAX_RANDKSTACK** — kstack base randomized; per-iteration `iov[]` and `cmsg` scratch placements vary.
- **PaX UDEREF** — every per-iteration `mmsg[i].msg_hdr`, `msg_name`, `msg_iov`, `msg_control` deref via `copy_from_user`. `msg_len` writeback via `put_user`. Direct kernel deref traps. **The compound nature means UDEREF protections fire vlen × 4 times per call** — a stress point.
- **GRKERNSEC_NO_SIMULT_CONNECT** — combined with `MSG_FASTOPEN`: only the first message may carry MSG_FASTOPEN per call under hardened policy (defeats batched TFO flooding).
- **GRKERNSEC_BLACKHOLE** — irrelevant on egress; per-message blackhole behavior is unchanged from sendmsg.
- **GRKERNSEC_RANDNET** — per-message IP_ID selection re-randomized; cannot guess sequence.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — raw-socket sendmmsg with IP_HDRINCL revalidates `CAP_NET_RAW` per call (not per message).
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — UNIX-domain sendmmsg with SCM_RIGHTS in multiple messages: each message's fd-pass independently subject to `MAX_FD_NESTING` and grsec rate-limit. **Rate-limited at messages-per-second per task**, not per call.
- **SOCK_CLOEXEC mandatory under suid** — irrelevant on sendmmsg (no fd creation).
- **SCM_RIGHTS recursion bound** — `current.scm_depth` re-checked per message; oversize burst at any message returns either `-ETOOMANYREFS` (first) or count-so-far (later).
- **sendmmsg compound-bound limit** — central grsec rule: vlen clamped to UIO_MAXIOV (1024); hardened builds may set `grsec.sendmmsg_max_vlen` to a lower value (e.g. 64) to limit burst amplification.
- **splice fd-permission boundary** — irrelevant on sendmmsg.
- **getsockopt info-leak prevention** — `msg.msg_flags` zeroed per iteration; per-msg storage zeroed before move_addr_to_kernel; no inter-message stack residue can leak.

## Open Questions

- Should the `grsec.sendmmsg_max_vlen` clamp be configurable per-cgroup? Upstream grsec: global. We follow.
- For `MSG_ZEROCOPY` across the compound: each message's completion lands on the socket error queue independently. Documented.

## Out of Scope

- `sendmsg(2)` — separate Tier-5 (single-message variant; shares `___sys_sendmsg`).
- `recvmmsg(2)` — separate Tier-5 (compound-receive analog).
- `sendto(2)` / `sendfile(2)` / `splice(2)` — separate Tier-5s.
- Per-AF `sock.ops.sendmsg` implementations — Tier-3 per family.
- `SCM_RIGHTS` mechanics — covered in `sendmsg.md`.
- Implementation code.
