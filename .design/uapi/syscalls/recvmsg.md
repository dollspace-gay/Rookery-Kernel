# Tier-5 syscall: recvmsg(2) — receive a message with ancillary data (syscall 47)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - net/socket.c (__sys_recvmsg, ___sys_recvmsg, recvmsg_copy_msghdr)
  - net/compat.c (get_compat_msghdr) [compat path]
  - net/core/scm.c (scm_recv, __scm_install_fd)
  - include/linux/socket.h (struct msghdr, struct cmsghdr, struct user_msghdr)
-->

## Summary

`recvmsg(2)` is the most general receive primitive: it gathers a message into a scatter/gather buffer set (`iovec` array), populates the source address (`msg_name`), and delivers any ancillary control data the peer sent (`msg_control`, carrying SCM_RIGHTS FD passing, IP_PKTINFO, IPV6_PKTINFO, SO_TIMESTAMP*, etc.). It is syscall number **47** on x86-64.

The path is `sys_recvmsg(fd, umsg, flags)` → `__sys_recvmsg(fd, umsg, flags, /*forbid_cmsg_compat=*/false)` → `___sys_recvmsg`: (1) copies user_msghdr in, (2) imports iov, (3) dispatches `sock.ops.recvmsg`, (4) writes back name and cmsg, (5) writes back msg_flags including `MSG_TRUNC`/`MSG_CTRUNC` indicators.

Critical for: UNIX-domain FD passing receivers (containers, systemd), every RX timestamping consumer (PTP), every multicast pktinfo demuxer.

This Tier-5 covers syscall **47** `recvmsg(int sockfd, struct msghdr *msg, int flags)`.

## Signature

```c
ssize_t recvmsg(int sockfd, struct msghdr *msg, int flags);
```

```rust
pub fn sys_recvmsg(sockfd: i32, msg: UserPtr<UserMsghdr>, flags: i32) -> SyscallResult<isize>;
```

x86-64 entry: `__x64_sys_recvmsg` → `__sys_recvmsg(fd, umsg, flags, /*forbid_cmsg_compat=*/false)`.

## Parameters

- **`sockfd`** — socket fd.
- **`msg`** — pointer to `struct user_msghdr`, output mostly:
  - `msg_name`/`msg_namelen` — output: source sockaddr if non-NULL.
  - `msg_iov`/`msg_iovlen` — input: scatter buffers.
  - `msg_control`/`msg_controllen` — output: ancillary cmsg buffer.
  - `msg_flags` — output: kernel sets `MSG_TRUNC`/`MSG_CTRUNC`/`MSG_OOB`/`MSG_EOR`/`MSG_ERRQUEUE`/`MSG_CMSG_CLOEXEC`.
- **`flags`** — `MSG_*` flags: `MSG_OOB`, `MSG_PEEK`, `MSG_WAITALL`, `MSG_DONTWAIT`, `MSG_TRUNC`, `MSG_ERRQUEUE`, `MSG_CMSG_CLOEXEC`, `MSG_CMSG_COMPAT` (compat-only).

## Return

Number of bytes received (≥ 0) on success. For DGRAM and `MSG_TRUNC` flag: returns true packet length even if larger than iov buffer (with msg_flags |= MSG_TRUNC). `-1`/`-errno` on error.

## Errors

- `EBADF` — sockfd not open.
- `ENOTSOCK` — fd not a socket.
- `EFAULT` — msg or component pointer fault.
- `EINVAL` — msg_iovlen > UIO_MAXIOV; msg_controllen too large; flags invalid for family.
- `EAGAIN` / `EWOULDBLOCK` — non-blocking and no data.
- `ENOTCONN` — recvmsg on unconnected stream socket.
- `ECONNRESET` — peer reset.
- `EINTR` — blocking recv interrupted by signal.
- `ENOMEM` — slab alloc fail.
- `EACCES` / `EPERM` — LSM denial.
- `ENOBUFS` — kernel ran out of receive buffer space for ancillary.
- `EOPNOTSUPP` — flag not supported for family.

## ABI surface

- Output `msg_flags` semantics:
  - `MSG_TRUNC` — datagram larger than iov buffer; trailing bytes dropped.
  - `MSG_CTRUNC` — ancillary data larger than `msg_controllen`; trailing cmsg dropped.
  - `MSG_OOB` — message contains out-of-band data.
  - `MSG_EOR` — end of record (SEQPACKET, etc.).
  - `MSG_ERRQUEUE` — message dequeued from socket error queue.
  - `MSG_CMSG_CLOEXEC` — echo of caller flag indicating CLOEXEC was applied to passed FDs.
- `MSG_PEEK` — leaves data in queue (sk_receive_queue not advanced).
- `MSG_WAITALL` — block until full iov filled (stream only); ignored for SOCK_DGRAM.
- `MSG_TRUNC` (as **input** flag for DGRAM): return actual datagram length even when larger than iov; useful for sizing.

## Compatibility contract

REQ-1: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-2: msghdr copy:
- err = copy_msghdr_from_user(&msg, umsg, &uaddr_storage, &iov, /*save_addr=*/false).
- if err: return err.
- if msg.msg_iovlen > UIO_MAXIOV: return `-EMSGSIZE`/`-EINVAL`.

REQ-3: iov import:
- err = import_iovec(READ, msg.msg_iov, msg.msg_iovlen, ARRAY_SIZE(iovstack), &iov, &msg.msg_iter).
- if err: return err.

REQ-4: msg_namelen sanity:
- if uaddr (user msg_name) ∧ msg_namelen < 0: return `-EINVAL`.

REQ-5: LSM:
- err = security_socket_recvmsg(sock, &msg, total_len, flags); if err: return err.

REQ-6: Dispatch:
- msg.msg_flags = 0.
- err = sock.ops.recvmsg(sock, &msg, total_len, flags).
- if err < 0: return err.

REQ-7: Address writeback:
- if uaddr (msg_name not NULL):
  - move_addr_to_user(&msg.msg_name, msg.msg_namelen, uaddr, &umsg.msg_namelen).

REQ-8: cmsg writeback:
- if umsg.msg_control non-NULL:
  - copy kernel-scratch cmsg buffer to user, possibly truncated; if truncated set MSG_CTRUNC.

REQ-9: SCM install:
- scm_recv handles SCM_RIGHTS / SCM_CREDENTIALS:
  - For SCM_RIGHTS: install fds via __scm_install_fd; honor MSG_CMSG_CLOEXEC (each new fd close-on-exec if set).
  - For SCM_CREDENTIALS: validated ucred written into cmsg.

REQ-10: msg_flags writeback:
- copy msg.msg_flags back to umsg.msg_flags (MSG_TRUNC, MSG_CTRUNC, MSG_OOB, MSG_EOR, MSG_ERRQUEUE, MSG_CMSG_CLOEXEC).

REQ-11: PEEK semantics:
- if flags & MSG_PEEK: data not removed from sk_receive_queue; reads idempotent.
- SCM_RIGHTS under PEEK: ambiguous historically — Linux current: peek does NOT install fds (skipped) to prevent dup-on-peek. Kernel sets MSG_CTRUNC if cmsg was present but skipped.

REQ-12: MSG_TRUNC return semantics:
- DGRAM: if flags & MSG_TRUNC: return actual packet length; else min(actual, buf_len).
- STREAM: not applicable; STREAM never truncates a "message".

REQ-13: Audit:
- audit_log_socketcall(SYS_RECVMSG, fd, &msg.msg_name?, msg.msg_namelen, ret, flags).

## Acceptance Criteria

- [ ] AC-1: recvmsg on UDP socket: returns packet length; iov filled.
- [ ] AC-2: recvmsg on UDP with iov < packet, without MSG_TRUNC flag: return min(iov, packet); msg_flags & MSG_TRUNC set.
- [ ] AC-3: recvmsg on UDP with iov < packet, with MSG_TRUNC flag: return packet length; iov holds only iov_len bytes; msg_flags & MSG_TRUNC.
- [ ] AC-4: recvmsg with MSG_PEEK: data still in queue; subsequent recv returns same data.
- [ ] AC-5: recvmsg with controllen < actual cmsg: msg_flags & MSG_CTRUNC; trailing cmsg dropped.
- [ ] AC-6: recvmsg with MSG_CMSG_CLOEXEC and SCM_RIGHTS payload: each new fd has FD_CLOEXEC.
- [ ] AC-7: recvmsg without MSG_CMSG_CLOEXEC: new fds do NOT have FD_CLOEXEC.
- [ ] AC-8: recvmsg on closed peer with empty queue (stream): returns 0 (EOF).
- [ ] AC-9: recvmsg blocking + SIGINT: -EINTR.
- [ ] AC-10: recvmsg non-blocking with empty queue: -EAGAIN.
- [ ] AC-11: MSG_PEEK + SCM_RIGHTS: msg_flags & MSG_CTRUNC; no FDs installed (peek does not consume).
- [ ] AC-12: msg.msg_iovlen > UIO_MAXIOV: -EMSGSIZE.

## Architecture

```
SysSocket::sys_recvmsg(fd, umsg, flags) -> SyscallResult<isize>
  return Self::sys_recvmsg_inner(fd, umsg, flags, /*forbid_cmsg_compat=*/false).

SysSocket::sys_recvmsg_inner(fd, umsg, flags, forbid_compat) -> SyscallResult<isize>
  1. sock = SocketFd::lookup_light(fd)?;
  2. msg = Msghdr::default();
  3. let (uaddr_storage, mut iov) = Self::copy_msghdr_from_user(umsg, &mut msg)?;
  4. if msg.msg_iovlen as usize > UIO_MAXIOV { return Err(EMSGSIZE); }
  5. ImportIovec::read(msg.msg_iov, msg.msg_iovlen, &mut iov, &mut msg.msg_iter)?;
  6. let total_len = msg.msg_iter.count();
  7. Lsm::socket_recvmsg(&sock, &msg, total_len, flags)?;
  8. msg.msg_flags = 0;
  9. let received = sock.ops.recvmsg(&sock, &mut msg, total_len, flags)?;
 10. if let Some(uaddr_ptr) = umsg.msg_name.option() {
        UserCopy::move_addr_to_user(&msg.msg_name, msg.msg_namelen, uaddr_ptr,
                                    &mut umsg.msg_namelen)?;
     }
 11. if !umsg.msg_control.is_null() {
        Self::put_cmsg_buf(&msg, umsg.msg_control, &mut umsg.msg_controllen, flags)?;
     }
 12. UserCopy::put_user(msg.msg_flags, &mut umsg.msg_flags)?;
 13. Audit::log_socketcall(SYS_RECVMSG, fd, ..., received, flags);
 14. Ok(received)
```

`Scm::recv(sock, msg, scm, flags)`:
1. for each scm cmsg:
   - if SOL_SOCKET, SCM_RIGHTS:
     - if flags & MSG_PEEK: set msg.msg_flags |= MSG_CTRUNC; skip install (mark fp[] for later).
     - else: for each fp file: fd = __scm_install_fd(file, flags & MSG_CMSG_CLOEXEC).
     - copy fd array into user cmsg buffer.
   - if SOL_SOCKET, SCM_CREDENTIALS: copy ucred.
   - else: per-AF cmsg handler.
2. if cmsg total > msg_controllen: msg.msg_flags |= MSG_CTRUNC; truncate.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `recvmsg_iovlen_bounds` | INVARIANT | per-sys_recvmsg: msg_iovlen ≤ UIO_MAXIOV. |
| `recvmsg_msg_flags_clear` | INVARIANT | per-sys_recvmsg: msg.msg_flags zeroed before dispatch. |
| `recvmsg_truncate_signal` | INVARIANT | per-DGRAM: iov_len < packet ⟹ MSG_TRUNC set. |
| `recvmsg_ctrunc_signal` | INVARIANT | per-cmsg: kernel-cmsg-bytes > msg_controllen ⟹ MSG_CTRUNC set. |
| `recvmsg_peek_idempotent` | INVARIANT | per-MSG_PEEK: sk_receive_queue head unchanged. |
| `recvmsg_cmsg_cloexec_atomic` | INVARIANT | per-MSG_CMSG_CLOEXEC: installed fds have FD_CLOEXEC pre-visibility. |
| `recvmsg_peek_no_fd_install` | INVARIANT | per-MSG_PEEK + SCM_RIGHTS: no fd installed. |

### Layer 2: TLA+

`uapi/syscalls/recvmsg.tla`:
- States: validate, msghdr-copy, iov-import, dispatch, name-copy, cmsg-copy, flags-copy, scm-install, return.
- Properties:
  - `safety_peek_no_consume` — MSG_PEEK ⟹ sk_receive_queue unchanged.
  - `safety_trunc_visibility` — DGRAM iov_len < packet ⟹ MSG_TRUNC in returned flags.
  - `safety_ctrunc_visibility` — cmsg overflow ⟹ MSG_CTRUNC.
  - `safety_cloexec_atomic` — MSG_CMSG_CLOEXEC ⟹ each new fd visible only with CLOEXEC bit set.
  - `liveness_blocking_returns` — blocking recv eventually returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: msg_iovlen ≤ UIO_MAXIOV | `sys_recvmsg` |
| post: ret ≥ 0 ⟹ bytes copied ≤ min(iov_len, packet_len) | `sys_recvmsg` |
| post: DGRAM ∧ ret < packet_len ⟹ MSG_TRUNC | `sys_recvmsg` |
| post: cmsg overflow ⟹ MSG_CTRUNC | `Scm::recv` |
| post: MSG_PEEK ⟹ no fd installs, no queue dequeue | `sys_recvmsg` |
| post: MSG_CMSG_CLOEXEC ⟹ all new fds have close_on_exec | `Scm::recv` |

### Layer 4: Verus/Creusot functional

Per-`__sys_recvmsg → ___sys_recvmsg → scm_recv → __scm_install_fd` semantic equivalence with `net/socket.c` and `net/core/scm.c`. SCM_RIGHTS fd-install order matches sender's fp[] order; MSG_CMSG_CLOEXEC bit applied atomically before fd visibility.

## Hardening

- **msg_iovlen ≤ UIO_MAXIOV (1024)** — defense against per-iov-array explosion.
- **`msg_flags` zeroed before dispatch** — defense against per-uninitialized-flags info-leak.
- **MSG_TRUNC always signaled when DGRAM payload exceeded** — defense against per-silent-truncation bug.
- **MSG_CTRUNC always signaled when cmsg overflowed** — defense against per-missing-cmsg bug.
- **MSG_PEEK + SCM_RIGHTS does NOT install fds** — defense against per-peek-dup-fd amplification.
- **MSG_CMSG_CLOEXEC atomic FD install** — defense against per-recvmsg-then-fcntl race.
- **LSM `socket_recvmsg`** — defense against per-MAC bypass.
- **`__scm_install_fd` uses `get_unused_fd_flags`** — defense against per-fd-leak on partial failure.
- **Cmsg buffer kmalloc bounded by msg_controllen** — defense against per-kmalloc-bomb.
- **Source-address writeback respects caller `msg_namelen`** — defense against per-OOB-write into user buffer.

## Grsecurity-PaX

- **PAX_RANDKSTACK** — per-syscall kstack base randomization; on-stack iov[] / cmsg scratch displaced.
- **PaX UDEREF** — every msg / iov / control userland deref goes through `copy_from/to_user`. Particularly important here because the kernel WRITES to `msg_name`, `msg_control`, `msg_namelen`, `msg_controllen`, `msg_flags` — five distinct writeback sites all under UDEREF protection.
- **GRKERNSEC_NO_SIMULT_CONNECT** — irrelevant on recvmsg path.
- **GRKERNSEC_BLACKHOLE** — peer-side; recvmsg simply sees what arrived.
- **GRKERNSEC_RANDNET** — irrelevant on recvmsg path.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — raw-socket recv requires CAP_NET_RAW (gated at socket creation).
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — UNIX-domain recvmsg validates SCM_CREDENTIALS pid against connected peer's verified pid; cross-namespace abstract-ns SCM_RIGHTS rejected under hardened config.
- **SOCK_CLOEXEC mandatory under suid** — grsec `suid_strict_cloexec=1` policy forces `MSG_CMSG_CLOEXEC` OR'd into flags when current task has recent SUID transition; prevents SCM_RIGHTS FD leak across `execve`. This is the most load-bearing grsec rule on recvmsg.
- **SCM_RIGHTS recursion bound** — receiver-side: hardened kernel tracks `current.scm_inflight`; oversize burst returns `-EMFILE`.
- **getsockopt info-leak prevention** — `msg.msg_flags` is initialized to 0 before dispatch; never returns stack garbage. cmsg buffer is zero-padded before copy_to_user.

## Open Questions

- For `MSG_PEEK + MSG_TRUNC` combination on DGRAM: report packet length without consuming; upstream confirmed. Document.
- For `recvmsg(... MSG_ERRQUEUE)`: should we route through generic dispatch or a dedicated `sock_recv_errqueue`? Upstream: dedicated path; we follow.

## Out of Scope

- `sendmsg(2)` — separate Tier-5.
- `recvmmsg(2)` — separate Tier-5.
- Per-AF `sock.ops.recvmsg` implementations — Tier-3 per family.
- `MSG_ERRQUEUE` socket error queue mechanics — Tier-3 `net/core/sock_err.md`.
- Implementation code.
