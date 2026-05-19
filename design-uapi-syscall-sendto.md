---
title: "Tier-5 syscall: sendto(2) — send a message on a socket with explicit destination (syscall 44)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`sendto(2)` is the workhorse datagram-send primitive: it transmits a contiguous buffer (`buf`, `len`) on a socket, optionally to an explicit `(dest, addrlen)` destination — which is required for unconnected `SOCK_DGRAM` and optional/ignored for connected `SOCK_STREAM`. It is syscall number **44** on x86-64.

The path is `sys_sendto(fd, buf, len, flags, dest, addrlen)` → `__sys_sendto()` builds an internal `struct msghdr` with a single-iovec `[buf, len]` and an optional `msg_name`, then calls `sock_sendmsg(sock, &msg)` → `sock.ops.sendmsg(sock, &msg, len)`. `send(2)` is implemented as `sendto(fd, buf, len, flags, NULL, 0)` and shares all code paths.

Critical for: every UDP send, every connected-TCP write that bypasses the writev/sendmsg overhead, every classic BSD-style `send`/`sendto` user code, every raw-socket header-include emission.

This Tier-5 covers syscall **44** `sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest, socklen_t addrlen)` and the subordinate `send(2)`.

### Acceptance Criteria

- [ ] AC-1: sendto(UDP, buf, len, 0, &dest, sizeof(dest)): returns len.
- [ ] AC-2: sendto(connected-TCP, buf, len, 0, NULL, 0): returns bytes sent.
- [ ] AC-3: sendto(connected-TCP, buf, len, 0, &dest, sizeof(dest)): -EISCONN.
- [ ] AC-4: sendto(unconnected-TCP, buf, len, 0, NULL, 0): -ENOTCONN.
- [ ] AC-5: sendto(UDP, 65508-byte buf): -EMSGSIZE.
- [ ] AC-6: sendto(closed-peer-TCP, buf, len, 0, NULL, 0): -EPIPE + SIGPIPE.
- [ ] AC-7: sendto(closed-peer-TCP, buf, len, MSG_NOSIGNAL, NULL, 0): -EPIPE, no SIGPIPE.
- [ ] AC-8: sendto(UDP, buf, len, 0, NULL, 0) unconnected: -EDESTADDRREQ.
- [ ] AC-9: sendto with addrlen > 128: -EINVAL.
- [ ] AC-10: sendto with addrlen = sizeof(sa_family_t) - 1: -EINVAL.
- [ ] AC-11: sendto with dest sa_family mismatched: -EAFNOSUPPORT.
- [ ] AC-12: sendto on raw socket with MSG_DONTROUTE and CAP_NET_RAW: success.
- [ ] AC-13: sendto with MSG_DONTWAIT on full sndbuf: -EAGAIN.
- [ ] AC-14: cgroup-BPF egress denial: -EPERM.

### Architecture

```
SysSocket::sys_sendto(fd, ubuf, len, flags, udest, addrlen) -> SyscallResult<isize>
  1. sock = SocketFd::lookup_light(fd)?;
  2. let mut msg = Msghdr::default();
  3. let mut storage = SockaddrStorage::zeroed();
  4. if !udest.is_null() && addrlen != 0 {
        if !(size_of::<SaFamilyT>() ..= 128).contains(&(addrlen as usize)) {
            return Err(EINVAL);
        }
        UserCopy::move_addr_to_kernel(udest, addrlen, &mut storage)?;
        msg.msg_name = &mut storage as *mut _ as *mut c_void;
        msg.msg_namelen = addrlen;
     }
  5. let mut iov = [Iovec { base: ubuf.as_ptr() as *mut _, len }];
  6. ImportIov::single(IoDirection::Write, ubuf, len, &mut iov[0], &mut msg.msg_iter)?;
  7. msg.msg_control = ptr::null_mut(); msg.msg_controllen = 0; msg.msg_flags = 0;
  8. Lsm::socket_sendmsg(&sock, &msg, len)?;
  9. CgroupBpf::inet_egress(&sock, &msg)?;
 10. let sent = sock.ops.sendmsg(&sock, &mut msg, len)?;
 11. if !(flags & MSG_NOSIGNAL) && sent == -EPIPE { Signal::send(SIGPIPE, current()); }
 12. Audit::log_socketcall(SYS_SENDTO, fd, &storage, addrlen, sent, flags);
 13. Ok(sent)
```

`sock_sendmsg(sock, msg)` (delegated to sendto path):
1. msg.msg_iocb = ptr::null_mut() (synchronous).
2. err = security_socket_sendmsg(sock, msg, len) — already done by caller; per-`sock_sendmsg_nosec` is alias.
3. return sock.ops.sendmsg(sock, msg, len).

### Out of Scope

- `sendmsg(2)` — separate Tier-5 (general scatter-gather + cmsg path).
- `sendmmsg(2)` — separate Tier-5.
- `sendfile(2)` — separate Tier-5 (single-call zero-copy fd→fd).
- `splice(2)` — separate Tier-5.
- Per-AF `sock.ops.sendmsg` implementations — Tier-3 per family.
- Implementation code.

### signature

```c
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
               const struct sockaddr *dest, socklen_t addrlen);
ssize_t send  (int sockfd, const void *buf, size_t len, int flags);  // == sendto(..., NULL, 0)
```

```rust
pub fn sys_sendto(sockfd: i32, buf: UserPtr<u8>, len: usize, flags: i32,
                  dest: UserPtr<Sockaddr>, addrlen: u32) -> SyscallResult<isize>;
```

x86-64 entry: `__x64_sys_sendto` → `__sys_sendto(fd, ubuf, len, flags, udest, addrlen)`.

### parameters

- **`sockfd`** — fd from `socket(2)`.
- **`buf`** — userland buffer to send.
- **`len`** — bytes to send (`size_t`).
- **`flags`** — `MSG_*` flags: `MSG_OOB`, `MSG_DONTROUTE`, `MSG_DONTWAIT`, `MSG_NOSIGNAL`, `MSG_CONFIRM`, `MSG_MORE`, `MSG_EOR`, `MSG_FASTOPEN`, `MSG_ZEROCOPY`.
- **`dest`** — userland sockaddr destination; may be NULL (with `addrlen == 0`).
- **`addrlen`** — bytes at `dest`; if 0, `dest` is NULL/ignored.

### return

Number of bytes sent (≥ 0) on success. May be less than `len` for stream sockets in non-blocking mode (partial send). `-1`/`-errno` on error.

### errors

- `EBADF` — sockfd not open.
- `ENOTSOCK` — fd not a socket.
- `EFAULT` — buf, dest pointer fault.
- `EINVAL` — addrlen out of bounds (must be in [sizeof(sa_family_t), 128]); flags invalid for family; SOCK_STREAM with `dest != NULL` and addrlen != 0 (returns EISCONN).
- `EAGAIN` / `EWOULDBLOCK` — non-blocking send would block.
- `EMSGSIZE` — datagram too large; UDP > 65507; UNIX_DGRAM > sk_sndbuf.
- `EPIPE` — stream peer closed (+ SIGPIPE unless MSG_NOSIGNAL).
- `EISCONN` — sendto with dest on connected stream.
- `ENOTCONN` — send on unconnected stream.
- `EAFNOSUPPORT` — dest sa_family doesn't match socket family.
- `EHOSTUNREACH` / `ENETUNREACH` / `ECONNREFUSED` / `ECONNRESET` — peer/route errors.
- `ENOMEM` — slab alloc fail.
- `ENOBUFS` — socket buffer full (dgram).
- `EOPNOTSUPP` — flag not supported by family.
- `EACCES` — broadcast send without SO_BROADCAST; LSM denial.
- `EPERM` — cgroup-BPF egress denial; CAP_NET_RAW required for raw header-include.
- `EINTR` — interrupted by signal.

### abi surface

- `dest`/`addrlen`==0: `dest` treated as NULL; sendto behaves like `send`. If socket is unconnected dgram, this yields `-EDESTADDRREQ`.
- `dest`/`addrlen`!=0 on connected stream: `-EISCONN`.
- `MSG_MORE`: hint to coalesce with subsequent send (Linux Nagle assist).
- `MSG_FASTOPEN`: combined connect+send for TCP Fast Open; dest required.
- `MSG_ZEROCOPY`: zerocopy egress (page-pin, completion via MSG_ERRQUEUE).
- internal: `__sys_sendto` constructs a `struct msghdr` with `msg_iov = &[ (buf, len) ]`, `msg_iovlen = 1`, `msg_control = NULL`.

### compatibility contract

REQ-1: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-2: dest copy:
- if udest != NULL ∧ addrlen != 0:
  - if addrlen < sizeof(sa_family_t) ∨ addrlen > 128: return `-EINVAL`.
  - err = move_addr_to_kernel(udest, addrlen, &storage); if err: return `-EFAULT`.
  - msg.msg_name = &storage; msg.msg_namelen = addrlen.
- else: msg.msg_name = NULL; msg.msg_namelen = 0.

REQ-3: iov build:
- iov = [(ubuf, len)].
- err = import_single_range(WRITE, ubuf, len, &iov, &msg.msg_iter).
- if err: return err.

REQ-4: control buffer:
- msg.msg_control = NULL; msg.msg_controllen = 0.
- msg.msg_flags = 0.

REQ-5: LSM:
- err = security_socket_sendmsg(sock, &msg, len); if err: return err.

REQ-6: cgroup-BPF:
- BPF_CGROUP_RUN_PROG_INET_EGRESS for INET; may rewrite or deny.

REQ-7: Dispatch:
- err = sock.ops.sendmsg(sock, &msg, len).

REQ-8: SIGPIPE:
- per-stream peer reset + !(flags & MSG_NOSIGNAL): send_sig(SIGPIPE, current).

REQ-9: Audit:
- audit_log_socketcall(SYS_SENDTO, fd, &storage?, addrlen, ret, flags).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sendto_addrlen_bounds` | INVARIANT | per-sys_sendto: addrlen ∈ [2, 128] when dest != NULL. |
| `sendto_iov_count_eq` | INVARIANT | per-sys_sendto: msg_iter.count() == len pre-dispatch. |
| `sendto_no_control` | INVARIANT | per-sys_sendto: msg_control == NULL ∧ msg_controllen == 0. |
| `sendto_msg_flags_zeroed` | INVARIANT | per-sys_sendto: msg_flags zeroed before dispatch. |
| `sendto_sigpipe_only_without_nosignal` | INVARIANT | per-sys_sendto: SIGPIPE delivered ⟹ !(flags & MSG_NOSIGNAL). |
| `sendto_dest_family_check` | INVARIANT | per-AF sendmsg: msg.msg_name.sa_family == sock.family else EAFNOSUPPORT. |

### Layer 2: TLA+

`uapi/syscalls/sendto.tla`:
- States: validate, addr-copy, iov-build, lsm, bpf, dispatch, sigpipe, return.
- Properties:
  - `safety_addr_bounded` — addrlen > 128 ⟹ EINVAL before any allocation.
  - `safety_connected_dest_rejected` — connected stream + non-NULL dest ⟹ EISCONN.
  - `safety_unconnected_dgram_needs_dest` — unconnected dgram + NULL dest ⟹ EDESTADDRREQ.
  - `safety_partial_send_bounded` — return value ≤ len.
  - `liveness_terminates` — sendto returns within bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: dest != NULL ⟹ addrlen ∈ [sizeof(sa_family_t), 128] | `sys_sendto` |
| pre: msg_iter.count() == len | `sys_sendto` |
| pre: msg_control == NULL | `sys_sendto` |
| post: ret ≥ 0 ⟹ ret ≤ len | `sys_sendto` |
| post: ret == -EPIPE ∧ stream ⟹ MSG_NOSIGNAL set OR SIGPIPE delivered | `sys_sendto` |

### Layer 4: Verus/Creusot functional

Per-`__sys_sendto → sock_sendmsg → sock.ops.sendmsg` semantic equivalence with `net/socket.c`. `move_addr_to_kernel` byte-copy and family check match upstream. `import_single_range` builds an iov_iter equivalent to a single-element `import_iovec` call.

### hardening

- **`addrlen` strict-bounded to `sizeof(struct sockaddr_storage)` (128)** — defense against per-OOB-read into stack storage.
- **`storage` zero-initialized before copy_from_user** — defense against per-info-leak from prior stack into per-AF sendmsg.
- **`msg_control = NULL` enforced — no ancillary on sendto** — defense against per-confused SCM injection.
- **`msg_flags = 0` zeroed pre-dispatch** — defense against per-stack-leak of msg.msg_flags into per-AF code path.
- **`len` is `size_t` (u64); per-AF dispatch downcasts safely** — defense against per-overflow on stream-iter consumption.
- **MSG_NOSIGNAL controls SIGPIPE** — defense against per-uncatchable-pipe.
- **LSM `socket_sendmsg`** — defense against per-MAC bypass.
- **cgroup-BPF `INET_EGRESS`** — defense against per-egress-policy bypass.
- **Per-stream `SO_SNDBUF` accounting** — defense against per-OOM via giant queue.
- **Family-cross-check at per-AF sendmsg** — defense against per-confused-family routing.
- **Broadcast requires `SO_BROADCAST`** — defense against per-amplification attacks via UDP broadcast.

### grsecurity-pax

- **PAX_RANDKSTACK** — kstack base randomized; `sockaddr_storage` + scratch iov[1] placements vary.
- **PaX UDEREF** — userland `ubuf`, `udest` derefs only via `copy_from_user` / `move_addr_to_kernel`. Direct kernel deref of either traps. Particularly important here since both pointers can be NULL in legitimate calls; grsec validates non-null pointers individually.
- **GRKERNSEC_NO_SIMULT_CONNECT** — paired with TCP Fast Open via `MSG_FASTOPEN`: hardened build rate-limits simultaneous TFO opens per task to prevent connection-flooding via combined connect+send.
- **GRKERNSEC_BLACKHOLE** — sendto to a blackholed destination silently completes (no on-wire ICMP feedback).
- **GRKERNSEC_RANDNET** — sendto on raw / ip_id-allocating egress gets extra entropy per packet.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — sendto on `SOCK_RAW` with `IP_HDRINCL` revalidates `CAP_NET_RAW` per-call (not just at socket creation) under hardened policy; defeats post-cap-drop raw-send.
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — UNIX-DGRAM sendto requires the destination path/abstract name to be ACL-readable by the caller; cross-namespace UNIX-DGRAM dest blocked.
- **SOCK_CLOEXEC mandatory under suid** — irrelevant on sendto (no fd creation).
- **SCM_RIGHTS recursion bound** — irrelevant on sendto (no ancillary).
- **sendmmsg compound-bound limit** — sendto is single-call; sendmmsg's per-call vlen bound is handled in `sendmmsg.md`.
- **splice fd-permission boundary** — irrelevant on sendto.
- **getsockopt info-leak prevention** — sendto path zero-inits `msg.msg_flags` and `storage`; no stack residue can be reflected back via cmsg or sk error queue.

