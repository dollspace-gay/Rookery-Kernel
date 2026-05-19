# Tier-5 syscall: recvmmsg(2) — receive multiple messages from a socket in one syscall (syscall 299)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - net/socket.c (__sys_recvmmsg, sys_recvmmsg, do_recvmmsg, ___sys_recvmsg)
  - net/socket.c (recvmsg_copy_msghdr)
  - net/core/scm.c (scm_recv, __scm_install_fd)
  - include/linux/socket.h (struct mmsghdr, struct msghdr)
  - include/uapi/asm-generic/socket.h (MSG_WAITFORONE 0x10000)
-->

## Summary

`recvmmsg(2)` is the compound-receive primitive: it gathers multiple messages from a socket into an array of `struct mmsghdr` in a single syscall, returning the number of messages received. It supports an optional `timeout` and the `MSG_WAITFORONE` flag, which causes the call to block for the first message but switch to non-blocking for subsequent messages. It is syscall number **299** on x86-64.

The path is `sys_recvmmsg(fd, mmsg, vlen, flags, timeout)` → `__sys_recvmmsg()` → `do_recvmmsg()` loop: per-iteration call `___sys_recvmsg(sock, &msg, &iov, flags', /*forbid_compat=*/false)` (where `flags'` may be `flags | MSG_DONTWAIT` for iterations after the first when `MSG_WAITFORONE` was set), write per-msg `msg_len` and `msg_hdr.msg_flags` back, and respect `timeout` between iterations.

Critical for: QUIC servers, DNS resolvers, packet receivers, log ingestors; any workload with many short inbound packets. Amortizes syscall entry cost over many receives.

This Tier-5 covers syscall **299** `recvmmsg(int sockfd, struct mmsghdr *msgvec, unsigned int vlen, int flags, struct timespec *timeout)`.

## Signature

```c
struct mmsghdr {
    struct msghdr msg_hdr;
    unsigned int  msg_len;   // bytes received per message (kernel-written)
};

int recvmmsg(int sockfd, struct mmsghdr *msgvec, unsigned int vlen, int flags,
             struct __kernel_timespec *timeout);
```

```rust
pub fn sys_recvmmsg(sockfd: i32, msgvec: UserPtr<Mmsghdr>, vlen: u32, flags: i32,
                    timeout: UserPtr<Timespec>) -> SyscallResult<i32>;
```

x86-64 entry: `__x64_sys_recvmmsg` → `__sys_recvmmsg(fd, mmsg, vlen, flags, timeout)`.

## Parameters

- **`sockfd`** — fd from `socket(2)` / `accept(2)`.
- **`msgvec`** — userland array of `struct mmsghdr`; kernel writes per-message `msg_hdr` fields and `msg_len`.
- **`vlen`** — number of slots in `msgvec`. Capped to `UIO_MAXIOV` (1024).
- **`flags`** — `MSG_*` flags applied to every message: `MSG_DONTWAIT`, `MSG_PEEK`, `MSG_TRUNC`, `MSG_WAITALL`, `MSG_ERRQUEUE`, `MSG_CMSG_CLOEXEC`, plus the compound-specific `MSG_WAITFORONE` (0x10000).
- **`timeout`** — userland `struct timespec`; if non-NULL, the loop terminates when the timeout expires (consumed). NULL = block indefinitely.

## Return

Number of messages received on success (≥ 1) when at least one arrived, `0` if the loop ended (e.g. timeout or `MSG_DONTWAIT`-style early-return) with no messages, or `-errno` on error before any message was received.

## Errors

- `EBADF` — sockfd not open.
- `ENOTSOCK` — fd not a socket.
- `EFAULT` — msgvec or timeout pointer fault.
- `EINVAL` — vlen == 0; per-msg msg_iovlen > UIO_MAXIOV; flags invalid; timeout has invalid tv_nsec.
- `EAGAIN` / `EWOULDBLOCK` — first message would block.
- `ENOTCONN` — first message recvmsg on unconnected stream.
- `ECONNRESET` — peer reset (first message).
- `EINTR` — first-message blocking recv interrupted by signal (mid-loop interrupt returns count).
- `ENOMEM` — slab alloc fail.
- `EOPNOTSUPP` — flag not supported.
- `EACCES` / `EPERM` — LSM denial.

## ABI surface

- `struct mmsghdr` is 4 bytes larger than `struct msghdr` (the `msg_len` writeback field).
- `vlen` is `u32`; clamped to `UIO_MAXIOV` (1024) per call.
- `MSG_WAITFORONE` (`0x10000`): block for first message, then set `MSG_DONTWAIT` for all subsequent iterations. Drains everything currently queued; returns once queue is empty.
- `timeout` is `struct __kernel_timespec` (64-bit on all archs); decremented in-place during loop; consumed value reflected back to user.
- Compatibility variant: `compat_sys_recvmmsg_time32` and `compat_sys_recvmmsg_time64` use compat structures.

## Compatibility contract

REQ-1: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-2: vlen clamp:
- if vlen == 0: return 0.
- if vlen > UIO_MAXIOV (1024): vlen = UIO_MAXIOV.

REQ-3: timeout copy:
- if timeout != NULL:
  - copy_from_user(&ts, timeout, sizeof(ts)); if err: return `-EFAULT`.
  - if ts.tv_nsec < 0 ∨ ts.tv_nsec ≥ 1_000_000_000 ∨ ts.tv_sec < 0: return `-EINVAL`.
  - end_time = ktime_get() + timespec64_to_ktime(ts).

REQ-4: For i in 0..vlen:
- per_msg_flags = flags;
- if (flags & MSG_WAITFORONE) ∧ i > 0: per_msg_flags |= MSG_DONTWAIT.
- err = ___sys_recvmsg(sock, &mmsg[i].msg_hdr, &iov, per_msg_flags).
- if err < 0:
  - if i == 0: return err.
  - else: break.
- put_user(err, &mmsg[i].msg_len).
- write back msg_hdr fields (msg_namelen, msg_controllen, msg_flags).

REQ-5: timeout check:
- after each iteration, if timeout != NULL ∧ ktime_get() ≥ end_time: break.

REQ-6: signal check:
- per-iteration: if signal_pending(current) ∧ i > 0: break.
- if signal_pending ∧ i == 0: return -EINTR.

REQ-7: timeout writeback:
- if timeout != NULL: copy_to_user(&timeout, &remaining, sizeof(ts)).

REQ-8: Audit:
- audit_log_socketcall(SYS_RECVMMSG, fd, NULL, 0, ret, flags).

## Acceptance Criteria

- [ ] AC-1: recvmmsg(UDP, mmsg×8, 8, 0) blocking, with 8 packets queued: returns 8.
- [ ] AC-2: recvmmsg(UDP, mmsg×8, 8, MSG_WAITFORONE) with 3 packets queued: returns 3; 4th iteration returns -EAGAIN, count > 0 ⟹ break ⟹ return 3.
- [ ] AC-3: recvmmsg(UDP, mmsg×8, 8, 0, timeout=100ms) with no packets: returns 0 (or -EAGAIN if non-blocking).
- [ ] AC-4: recvmmsg(UDP, mmsg×8, 8, MSG_WAITFORONE, timeout=NULL) blocking, 1 packet arrives: returns 1.
- [ ] AC-5: recvmmsg vlen = 0: returns 0.
- [ ] AC-6: recvmmsg vlen = 2000: clamped to 1024.
- [ ] AC-7: recvmmsg first iteration -EAGAIN non-blocking: returns -EAGAIN.
- [ ] AC-8: recvmmsg iter 0..2 succeed, iter 3 -EAGAIN: returns 3.
- [ ] AC-9: recvmmsg interrupted by signal after 2: returns 2; before any: -EINTR.
- [ ] AC-10: recvmmsg timeout with tv_nsec = 1_000_000_000: -EINVAL.
- [ ] AC-11: recvmmsg with MSG_CMSG_CLOEXEC: each new fd from SCM_RIGHTS has FD_CLOEXEC.
- [ ] AC-12: recvmmsg msgvec pointer fault: -EFAULT.
- [ ] AC-13: recvmmsg with MSG_PEEK on UDP: queue not consumed; subsequent recv returns same packets.

## Architecture

```
SysSocket::sys_recvmmsg(fd, msgvec, vlen, flags, timeout) -> SyscallResult<i32>
  1. sock = SocketFd::lookup_light(fd)?;
  2. if vlen == 0 { return Ok(0); }
  3. let n = core::cmp::min(vlen as usize, UIO_MAXIOV);
  4. let end_time = if !timeout.is_null() {
        let ts: Timespec = UserCopy::from_user_struct(timeout)?;
        if ts.tv_nsec < 0 || ts.tv_nsec >= 1_000_000_000 || ts.tv_sec < 0 {
            return Err(EINVAL);
        }
        Some(KTime::now() + ts.into())
     } else { None };
  5. let mut received: i32 = 0;
  6. for i in 0..n {
        // signal check
        if Task::signal_pending() {
            if received > 0 { break; }
            return Err(EINTR);
        }
        let per_msg_flags = if (flags & MSG_WAITFORONE) != 0 && i > 0 {
            flags | MSG_DONTWAIT
        } else { flags };
        let mmsg_ptr = unsafe { msgvec.add(i) };
        let bytes = match Self::sys_recvmsg_inner(&sock, &mut mmsg_ptr.msg_hdr,
                                                  per_msg_flags, false) {
            Ok(b) => b,
            Err(_) if received > 0 => break,
            Err(e) => return Err(e),
        };
        UserCopy::put_user(bytes as u32, &mut mmsg_ptr.msg_len)?;
        received += 1;
        if let Some(end) = end_time {
            if KTime::now() >= end { break; }
        }
     }
  7. if let Some(end) = end_time {
        let remaining = end.saturating_sub(KTime::now());
        UserCopy::to_user_struct(timeout, &remaining.into_timespec())?;
     }
  8. Audit::log_socketcall(SYS_RECVMMSG, fd, &(), 0, received, flags);
  9. Ok(received)
```

Per-iteration inner `sys_recvmsg_inner` shares body with `sys_recvmsg` Tier-5 minus the fd lookup; writes back `msg_namelen`, `msg_controllen`, `msg_flags` via `copy_msghdr_writeback`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `recvmmsg_vlen_clamp` | INVARIANT | per-sys_recvmmsg: vlen clamped to UIO_MAXIOV (1024). |
| `recvmmsg_first_error_propagated` | INVARIANT | per-loop: i == 0 ∧ err < 0 ⟹ return -err. |
| `recvmmsg_partial_success` | INVARIANT | per-loop: i > 0 ∧ err < 0 ⟹ break ⟹ return i. |
| `recvmmsg_msg_len_writeback` | INVARIANT | per-iteration: success ⟹ put_user(bytes, msg_len) before received++. |
| `recvmmsg_waitforone_semantics` | INVARIANT | per-MSG_WAITFORONE: i > 0 ⟹ MSG_DONTWAIT OR'd into per-msg flags. |
| `recvmmsg_timeout_validation` | INVARIANT | per-sys_recvmmsg: timeout.tv_nsec ∈ [0, 1_000_000_000) ∧ tv_sec ≥ 0. |
| `recvmmsg_per_msg_iov_bound` | INVARIANT | per-iteration: msg.msg_iovlen ≤ UIO_MAXIOV. |
| `recvmmsg_msg_flags_zeroed` | INVARIANT | per-iteration: msg.msg_flags zeroed before per-AF recvmsg. |
| `recvmmsg_signal_check` | INVARIANT | per-loop: signal_pending checked at every iteration boundary. |

### Layer 2: TLA+

`uapi/syscalls/recvmmsg.tla`:
- States: validate, timeout-copy, per-message-recv, per-message-writeback, loop-advance, timeout-check, signal-check, timeout-writeback, return.
- Properties:
  - `safety_partial_progress_monotonic` — received count never decreases mid-call.
  - `safety_first_failure_propagated` — i == 0 failure ⟹ return -err.
  - `safety_late_failure_returns_count` — i > 0 failure ⟹ return i.
  - `safety_waitforone_breaks_on_empty` — MSG_WAITFORONE ⟹ first iter blocks, subsequent skips when empty.
  - `safety_timeout_writeback` — non-NULL timeout ⟹ caller observes consumed time.
  - `liveness_terminates` — recvmmsg returns within bounded steps (when timeout finite or vlen consumed).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: vlen clamped to UIO_MAXIOV | `sys_recvmmsg` |
| pre: timeout.tv_nsec ∈ [0, 1e9) ∧ tv_sec ≥ 0 | `sys_recvmmsg` |
| pre: per-iteration msg.msg_iovlen ≤ UIO_MAXIOV | `sys_recvmsg_inner` |
| pre: per-iteration msg.msg_flags == 0 pre-dispatch | `sys_recvmsg_inner` |
| post: ret ≥ 0 ⟹ first ret messages successfully received | `sys_recvmmsg` |
| post: ret > 0 ⟹ msg_len[0..ret] and msg_hdr[0..ret] populated | `sys_recvmmsg` |
| post: ret < 0 ⟹ zero messages received | `sys_recvmmsg` |
| post: timeout writeback ⟹ remaining time written | `sys_recvmmsg` |

### Layer 4: Verus/Creusot functional

Per-`__sys_recvmmsg → do_recvmmsg → ___sys_recvmsg` semantic equivalence with `net/socket.c`. `MSG_WAITFORONE` flag handling matches upstream: blocking on first iteration only. Timeout consumption uses `ktime_get` against `end_time` per upstream.

## Hardening

- **`vlen` clamp to UIO_MAXIOV (1024)** — defense against per-vlen DoS.
- **`timeout` field validation pre-loop** — defense against per-overflow on bogus tv_nsec/tv_sec.
- **Per-iteration signal check** — defense against per-uninterruptible-loop.
- **Per-message `msg_iovlen ≤ UIO_MAXIOV` revalidation** — defense against per-iov-explosion.
- **Per-iteration `msg.msg_flags` zeroed pre-dispatch** — defense against per-stack-leak into per-msg writeback.
- **Per-iteration `msg.msg_namelen` and `msg.msg_controllen` writeback under UDEREF** — defense against per-userspace-write OOB.
- **`MSG_WAITFORONE` causes `MSG_DONTWAIT` on iter ≥ 1** — defense against per-stall when queue dries up.
- **First-message-failure propagates errno** — defense against per-silent-failure.
- **Late-message-failure returns count** — defense against per-state-loss.
- **`put_user(msg_len)` before `received++`** — defense against per-state-skew between kernel counter and user-visible msg_len.
- **`MSG_CMSG_CLOEXEC` propagated to each message's SCM_RIGHTS fds** — defense against per-FD-leak-across-exec.
- **Timeout writeback uses `to_user` clamp** — defense against per-OOB-write into caller's struct timespec.
- **Per-iteration LSM `socket_recvmsg`** — defense against per-MAC bypass.

## Grsecurity-PaX

- **PAX_RANDKSTACK** — kstack base randomized; per-iteration `iov[]` and `cmsg` scratch placements vary.
- **PaX UDEREF** — every per-iteration `mmsg[i].msg_hdr`, `msg_name`, `msg_iov`, `msg_control` writeback via `copy_to_user`; `msg_len` via `put_user`. **Compound nature means UDEREF fires vlen × ~5 times per call.**
- **GRKERNSEC_NO_SIMULT_CONNECT** — irrelevant on recvmmsg.
- **GRKERNSEC_BLACKHOLE** — recvmmsg sees what arrived; blackhole policy applies at peer.
- **GRKERNSEC_RANDNET** — irrelevant on recvmmsg.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — raw-socket recvmmsg requires `CAP_NET_RAW` (gated at socket creation).
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — UNIX-domain recvmmsg with SCM_RIGHTS in multiple messages: each message's fd-install independently subject to `MAX_FD_NESTING`. Cross-namespace SCM_RIGHTS rejected per-message under hardened policy.
- **SOCK_CLOEXEC mandatory under suid** — hardened `suid_strict_cloexec=1` policy forces `MSG_CMSG_CLOEXEC` OR'd into `flags` (applies to ALL messages in the compound) when current task has recent SUID transition; defeats batched SCM_RIGHTS-via-execve leakage.
- **SCM_RIGHTS recursion bound** — receiver-side `current.scm_inflight` re-checked per message; oversize burst at any message returns either `-EMFILE` (first) or count-so-far (later).
- **sendmmsg compound-bound limit** — symmetric rule applies to recvmmsg: vlen clamped to UIO_MAXIOV (1024). Hardened builds may set lower (e.g. 64). Configurable `grsec.recvmmsg_max_vlen`.
- **splice fd-permission boundary** — irrelevant on recvmmsg.
- **getsockopt info-leak prevention** — `msg.msg_flags` zeroed per iteration; per-msg storage zeroed before per-AF write; cmsg buffer zero-padded before copy_to_user. **No inter-message stack residue can leak**, even across UIO_MAXIOV iterations.

## Open Questions

- Should `grsec.recvmmsg_max_vlen` be per-cgroup configurable? Upstream grsec: global. We follow.
- For `MSG_PEEK` across the compound: every iteration peeks, queue never advances. Document behavior; matches upstream.

## Out of Scope

- `recvmsg(2)` — separate Tier-5 (single-message variant; shares `___sys_recvmsg`).
- `sendmmsg(2)` — separate Tier-5 (compound-send analog).
- `recvfrom(2)` / `recv(2)` — separate Tier-5s.
- Per-AF `sock.ops.recvmsg` implementations — Tier-3 per family.
- `SCM_RIGHTS` mechanics — covered in `recvmsg.md`.
- `MSG_ERRQUEUE` semantics — Tier-3 `net/core/sock_err.md`.
- Implementation code.
