# Tier-5 syscall: recvfrom(2) — receive a message from a socket and capture source (syscall 45)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - net/socket.c (__sys_recvfrom, sys_recvfrom, sys_recv, move_addr_to_user)
  - net/socket.c (sock_recvmsg, sock_recvmsg_nosec)
  - net/ipv4/udp.c (udp_recvmsg)
  - net/ipv4/tcp.c (tcp_recvmsg)
  - net/unix/af_unix.c (unix_dgram_recvmsg, unix_stream_recvmsg)
  - include/linux/socket.h (struct msghdr)
-->

## Summary

`recvfrom(2)` is the workhorse datagram-receive primitive: it reads bytes from a socket into a contiguous user buffer (`buf`, `len`), and — when `src`/`addrlen` are non-NULL — also writes the sender's address back. It is syscall number **45** on x86-64.

The path is `sys_recvfrom(fd, buf, len, flags, src, addrlen)` → `__sys_recvfrom()` builds an internal `struct msghdr` with a single-iovec `[buf, len]` and optional address writeback slot, then calls `sock_recvmsg(sock, &msg, flags)` → `sock.ops.recvmsg(sock, &msg, len, flags)`. `recv(2)` is implemented as `recvfrom(fd, buf, len, flags, NULL, NULL)` and shares all code paths.

Critical for: every UDP server (need source address for replies), every connected-TCP `read`-equivalent that accepts `MSG_*` flags, every classic BSD-style `recv`/`recvfrom` user code, every raw-socket packet inspector.

This Tier-5 covers syscall **45** `recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src, socklen_t *addrlen)` and subordinate `recv(2)`.

## Signature

```c
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                 struct sockaddr *src, socklen_t *addrlen);
ssize_t recv    (int sockfd, void *buf, size_t len, int flags);  // == recvfrom(..., NULL, NULL)
```

```rust
pub fn sys_recvfrom(sockfd: i32, buf: UserPtr<u8>, len: usize, flags: i32,
                    src: UserPtr<Sockaddr>, addrlen: UserPtr<u32>)
    -> SyscallResult<isize>;
```

x86-64 entry: `__x64_sys_recvfrom` → `__sys_recvfrom(fd, ubuf, len, flags, usrc, usrclen)`.

## Parameters

- **`sockfd`** — fd from `socket(2)` / `accept(2)`.
- **`buf`** — userland buffer to fill.
- **`len`** — buffer capacity (`size_t`).
- **`flags`** — `MSG_*` flags: `MSG_OOB`, `MSG_PEEK`, `MSG_WAITALL`, `MSG_DONTWAIT`, `MSG_TRUNC`, `MSG_ERRQUEUE`.
- **`src`** — userland sockaddr buffer (output, may be NULL).
- **`addrlen`** — IN/OUT: on entry, capacity of `src`; on exit, kernel-side namelen of the source address (may exceed capacity).

## Return

Number of bytes received (≥ 0) on success. For DGRAM with `MSG_TRUNC` flag, returns actual packet length even when larger than `len`. `-1`/`-errno` on error.

## Errors

- `EBADF` — sockfd not open.
- `ENOTSOCK` — fd not a socket.
- `EFAULT` — buf, src, addrlen pointer fault.
- `EINVAL` — `*addrlen` negative on entry; flags invalid for family.
- `EAGAIN` / `EWOULDBLOCK` — non-blocking and no data.
- `ENOTCONN` — recv on unconnected stream socket.
- `ECONNRESET` — peer reset.
- `EINTR` — blocking recv interrupted by signal.
- `ENOMEM` — slab alloc fail.
- `EACCES` / `EPERM` — LSM denial.
- `EOPNOTSUPP` — flag not supported for family.

## ABI surface

- `src` == NULL: source address not returned; `addrlen` need not be valid (must also be NULL).
- `src` != NULL ∧ `addrlen` == NULL: `-EFAULT`.
- `addrlen` IN/OUT semantics identical to `getsockname(2)`: kernel writes back kernel-side namelen even when input was smaller (truncation indicator).
- `MSG_PEEK`: data remains in `sk_receive_queue`.
- `MSG_WAITALL`: block until full `len` filled (stream only).
- `MSG_TRUNC` as input flag: return actual datagram length for DGRAM.
- internally builds `struct msghdr` with `msg_iov = &[(buf, len)]`, `msg_iovlen = 1`, `msg_control = NULL`.

## Compatibility contract

REQ-1: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-2: addrlen read-in:
- if usrc != NULL ∧ usrclen != NULL:
  - err = get_user(input_len, usrclen); if err: return `-EFAULT`.
  - if (int)input_len < 0: return `-EINVAL`.
- elif usrc != NULL ∧ usrclen == NULL: return `-EFAULT`.

REQ-3: iov build:
- iov = [(ubuf, len)].
- err = import_single_range(READ, ubuf, len, &iov, &msg.msg_iter).
- if err: return err.

REQ-4: msghdr scaffold:
- msg.msg_name = &storage (if usrc); else NULL.
- msg.msg_namelen = 0.
- msg.msg_control = NULL; msg.msg_controllen = 0.
- msg.msg_flags = 0.

REQ-5: LSM:
- err = security_socket_recvmsg(sock, &msg, len, flags); if err: return err.

REQ-6: Dispatch:
- err = sock.ops.recvmsg(sock, &msg, len, flags).
- per-AF writes msg.msg_namelen.

REQ-7: Address writeback:
- if usrc != NULL:
  - move_addr_to_user(&storage, msg.msg_namelen, usrc, usrclen).

REQ-8: Audit:
- audit_log_socketcall(SYS_RECVFROM, fd, &storage?, msg.msg_namelen, ret, flags).

## Acceptance Criteria

- [ ] AC-1: recvfrom(UDP, buf, len, 0, &src, &slen): returns packet bytes; src filled; slen == 16.
- [ ] AC-2: recvfrom(connected-TCP, buf, len, 0, NULL, NULL): returns stream bytes.
- [ ] AC-3: recvfrom(unconnected-TCP, buf, len, 0, NULL, NULL): -ENOTCONN.
- [ ] AC-4: recvfrom(UDP, buf=4, len=4, MSG_TRUNC, &src, &slen) on 1500-byte packet: returns 1500; iov[0..4] filled; src filled.
- [ ] AC-5: recvfrom with MSG_PEEK: data remains; subsequent recv returns same bytes.
- [ ] AC-6: recvfrom non-blocking + empty queue: -EAGAIN.
- [ ] AC-7: recvfrom blocking + SIGINT: -EINTR.
- [ ] AC-8: recvfrom on closed peer stream with empty queue: returns 0 (EOF).
- [ ] AC-9: recvfrom with src != NULL ∧ addrlen == NULL: -EFAULT.
- [ ] AC-10: recvfrom with *addrlen = (u32)-1: -EINVAL.
- [ ] AC-11: recvfrom with input addrlen = 8 (UDP source truncated): copies 8 bytes; *addrlen out = 16.
- [ ] AC-12: recvfrom with MSG_ERRQUEUE on socket with queued error: returns ICMP/TCP error.

## Architecture

```
SysSocket::sys_recvfrom(fd, ubuf, len, flags, usrc, usrclen) -> SyscallResult<isize>
  1. sock = SocketFd::lookup_light(fd)?;
  2. let mut msg = Msghdr::default();
  3. let mut storage = SockaddrStorage::zeroed();
  4. let writeback_addr = !usrc.is_null();
  5. if writeback_addr {
        if usrclen.is_null() { return Err(EFAULT); }
        let input_len: u32 = UserCopy::get_user(usrclen)?;
        if (input_len as i32) < 0 { return Err(EINVAL); }
        msg.msg_name = &mut storage as *mut _ as *mut c_void;
     }
  6. let mut iov = [Iovec { base: ubuf.as_ptr() as *mut _, len }];
  7. ImportIov::single(IoDirection::Read, ubuf, len, &mut iov[0], &mut msg.msg_iter)?;
  8. msg.msg_control = ptr::null_mut(); msg.msg_controllen = 0; msg.msg_flags = 0;
  9. Lsm::socket_recvmsg(&sock, &msg, len, flags)?;
 10. let received = sock.ops.recvmsg(&sock, &mut msg, len, flags)?;
 11. if writeback_addr {
        UserCopy::move_addr_to_user(&storage, msg.msg_namelen as usize, usrc, usrclen)?;
     }
 12. Audit::log_socketcall(SYS_RECVFROM, fd, &storage, msg.msg_namelen, received, flags);
 13. Ok(received)
```

`sock_recvmsg(sock, msg, flags)`:
1. msg.msg_iocb = ptr::null_mut().
2. err = security_socket_recvmsg(sock, msg, msg.msg_iter.count(), flags) — already done by caller.
3. return sock.ops.recvmsg(sock, msg, msg.msg_iter.count(), flags).

`move_addr_to_user(storage, kernel_namelen, usrc, usrclen)`:
1. let input_len: u32 = UserCopy::get_user(usrclen)?.
2. let copy_len = core::cmp::min(kernel_namelen, input_len as usize).
3. UserCopy::to_user(usrc, &storage.as_bytes()[..copy_len])?.
4. UserCopy::put_user(kernel_namelen as u32, usrclen)?.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `recvfrom_iov_count_eq` | INVARIANT | per-sys_recvfrom: msg_iter.count() == len pre-dispatch. |
| `recvfrom_no_control` | INVARIANT | per-sys_recvfrom: msg_control == NULL ∧ msg_controllen == 0. |
| `recvfrom_msg_flags_zeroed` | INVARIANT | per-sys_recvfrom: msg_flags zeroed before dispatch. |
| `recvfrom_truncate_signal` | INVARIANT | per-DGRAM: iov_len < packet_len ⟹ returned bytes ≤ iov_len; with MSG_TRUNC flag the return value is full packet_len. |
| `recvfrom_storage_zeroed` | INVARIANT | per-sys_recvfrom: storage zeroed before dispatch (when writeback enabled). |
| `recvfrom_address_writeback_bounded` | INVARIANT | per-move_addr_to_user: bytes ≤ min(kernel_namelen, input_len). |
| `recvfrom_addrlen_full_reported` | INVARIANT | per-sys_recvfrom: *addrlen on return == kernel-side namelen. |

### Layer 2: TLA+

`uapi/syscalls/recvfrom.tla`:
- States: validate, addrlen-read, iov-build, lsm, dispatch, addr-writeback, return.
- Properties:
  - `safety_no_addr_writeback_without_src` — usrc == NULL ⟹ no copy_to_user of storage.
  - `safety_addrlen_pointer_required_when_src` — usrc != NULL ∧ usrclen == NULL ⟹ EFAULT.
  - `safety_namelen_reported_full` — *addrlen on return == kernel-side namelen.
  - `safety_no_stack_leak` — storage bytes beyond namelen never written to user.
  - `safety_peek_no_consume` — MSG_PEEK ⟹ sk_receive_queue unchanged.
  - `liveness_blocking_returns` — blocking recv eventually returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: usrc != NULL ⟹ usrclen != NULL | `sys_recvfrom` |
| pre: msg_iter.count() == len | `sys_recvfrom` |
| pre: storage zeroed when usrc != NULL | `sys_recvfrom` |
| post: ret ≥ 0 ⟹ bytes copied ≤ min(len, packet_len) | `sys_recvfrom` |
| post: usrc != NULL ⟹ *addrlen out == msg.msg_namelen | `sys_recvfrom` |
| post: bytes copied to user addr ≤ min(msg.msg_namelen, input_len) | `move_addr_to_user` |

### Layer 4: Verus/Creusot functional

Per-`__sys_recvfrom → sock_recvmsg → sock.ops.recvmsg → move_addr_to_user` semantic equivalence with `net/socket.c`. `import_single_range` builds equivalent iov_iter to a single-element `import_iovec`. Per-AF recvmsg writes msg.msg_namelen exactly per upstream.

## Hardening

- **`storage` zero-initialized when usrc != NULL** — defense against per-info-leak of prior stack contents.
- **`storage` is the full 128-byte `sockaddr_storage`** — defense against per-AF over-write into smaller object.
- **`move_addr_to_user` clamps copy_len** — defense against per-OOB-write into caller's buffer.
- **`*addrlen` reports full kernel length** — defense against per-silent-truncation API surprise.
- **`msg.msg_flags = 0` zeroed pre-dispatch** — defense against per-stack-leak via msg.msg_flags.
- **`msg_control = NULL` enforced** — defense against per-stray-ancillary read.
- **`input_len` signed-check pre-dispatch** — defense against per-overflow when input_len > INT_MAX.
- **LSM `socket_recvmsg`** — defense against per-MAC bypass.
- **Per-AF recvmsg writes to caller buffer via iov_iter** — defense against per-OOB-write (iov_iter enforces caller's `len`).
- **`MSG_PEEK` honored at per-AF layer** — defense against per-double-consume of pending data.
- **Source-address writeback respects `*addrlen` capacity** — defense against per-OOB-write into caller's src buffer.

## Grsecurity-PaX

- **PAX_RANDKSTACK** — kstack base randomized; `storage` + scratch iov[1] placements vary.
- **PaX UDEREF** — userland `ubuf`, `usrc`, `usrclen` derefs only via `copy_to_user` / `get_user` / `put_user` / `move_addr_to_user`. Direct kernel deref of any traps. Critical here because the kernel WRITES to `ubuf`, `usrc`, AND `*usrclen` — three distinct user pointers all under UDEREF.
- **GRKERNSEC_NO_SIMULT_CONNECT** — irrelevant on recvfrom path.
- **GRKERNSEC_BLACKHOLE** — receive side simply sees whatever arrived; blackhole policy applies at egress.
- **GRKERNSEC_RANDNET** — irrelevant on recvfrom (no port allocation).
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — raw-socket recvfrom requires CAP_NET_RAW (already gated at socket creation).
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — UNIX-DGRAM recvfrom validates SCM_CREDENTIALS (if peer attached) against connected-peer pid; cross-namespace UNIX-DGRAM source addresses returned by recvfrom may be sanitized under hardened policy.
- **SOCK_CLOEXEC mandatory under suid** — irrelevant on recvfrom (no fd creation; SCM_RIGHTS path is on recvmsg).
- **SCM_RIGHTS recursion bound** — irrelevant on recvfrom (no cmsg path).
- **sendmmsg compound-bound limit** — irrelevant.
- **splice fd-permission boundary** — irrelevant.
- **getsockopt info-leak prevention** — `storage` zeroed; `sin_zero[8]` always zero; sockaddr_un padding zeroed; netlink `nl_pad` zeroed. **This is the central grsec invariant for recvfrom** — peer addresses must never leak unrelated stack state. **`msg.msg_flags` is zeroed before dispatch and never written back here**, eliminating an MSG_*-flag info-leak vector.

## Open Questions

- For `MSG_TRUNC | MSG_PEEK` on DGRAM: peek packet length without consuming; upstream supports. We follow.
- For `MSG_ERRQUEUE` with `usrc != NULL`: source is the icmp/tcp error originator. Document interaction; matches upstream.

## Out of Scope

- `recvmsg(2)` — separate Tier-5 (general scatter-gather + cmsg path).
- `recvmmsg(2)` — separate Tier-5.
- `read(2)` on a socket fd — separate Tier-5 (no flags, no source address).
- Per-AF `sock.ops.recvmsg` implementations — Tier-3 per family.
- `MSG_ERRQUEUE` mechanics — Tier-3 `net/core/sock_err.md`.
- Implementation code.
