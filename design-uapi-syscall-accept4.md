---
title: "Tier-5 syscall: accept4(2) — accept a connection with flags (syscall 288)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`accept4(2)` extends `accept(2)` with an explicit `flags` argument that allows atomically setting `SOCK_CLOEXEC` (close-on-exec) and/or `SOCK_NONBLOCK` (non-blocking) on the newly-returned fd, eliminating the classic accept→fcntl race. It is syscall number **288** on x86-64.

The implementation is the canonical backend: `sys_accept(2)` is implemented as `sys_accept4(... flags=0)`. The kernel removes the first established connection from the listen queue of `sockfd`, creates a new socket fd inheriting the listening socket's family, type, and protocol vector, and installs the fd with the requested CLOEXEC/NONBLOCK bits set atomically.

Critical for: every modern server runtime (libuv, tokio, glommio), every container's safe-FD-passing flow, every `setuid` daemon that must avoid CLOEXEC races.

This Tier-5 covers syscall **288** `accept4(int sockfd, struct sockaddr *addr, socklen_t *addrlen, int flags)`.

### Acceptance Criteria

- [ ] AC-1: accept4(fd, addr, &len, SOCK_CLOEXEC) → returned fd has FD_CLOEXEC set.
- [ ] AC-2: accept4(fd, NULL, NULL, SOCK_NONBLOCK) → fcntl(newfd, F_GETFL) & O_NONBLOCK != 0.
- [ ] AC-3: accept4(fd, addr, &len, SOCK_CLOEXEC | SOCK_NONBLOCK) → both bits set.
- [ ] AC-4: accept4(fd, addr, &len, 0xdeadbeef) → -EINVAL.
- [ ] AC-5: accept4 on non-listening socket → -EINVAL.
- [ ] AC-6: accept4 on SOCK_DGRAM → -EOPNOTSUPP.
- [ ] AC-7: addr NULL: no peer write; *addrlen untouched.
- [ ] AC-8: addrlen < actual: truncate addr; *addrlen = actual length.
- [ ] AC-9: blocking accept4 + SIGINT → -EINTR.
- [ ] AC-10: ECONNABORTED on RST during handshake.
- [ ] AC-11: under `fork() + execve()` race: child never inherits the new socket fd if SOCK_CLOEXEC was passed.
- [ ] AC-12: hitting RLIMIT_NOFILE → -EMFILE.

### Architecture

```
SysSocket::sys_accept4(fd, uaddr, uaddrlen, flags) -> SyscallResult<i32>
  1. if flags & !(SOCK_CLOEXEC | SOCK_NONBLOCK) != 0: return Err(EINVAL).
  2. sock = SocketFd::lookup_light(fd)?;
  3. accept_op = sock.ops.accept.ok_or(EOPNOTSUPP)?;
  4. newsock = Socket::alloc().ok_or(ENFILE)?;
  5. newsock.type = sock.type; newsock.ops = sock.ops;
  6. err = accept_op(&sock, &newsock,
                    sock.file.f_flags | (flags & O_NONBLOCK),
                    /*kern=*/false);
     if err < 0 { Socket::release(newsock); return Err(err); }
  7. if let Some(uaddr_ptr) = uaddr.option() {
        let mut storage = SockaddrStorage::zeroed();
        let len = newsock.ops.getname(&newsock, &mut storage, /*peer=*/true)?;
        UserCopy::move_addr_to_user(&storage, len, uaddr_ptr, uaddrlen)?;
     }
  8. let newfile = Socket::alloc_file(
        newsock,
        flags & (O_CLOEXEC | O_NONBLOCK),
        None,
     )?;
  9. let newfd = FdTable::get_unused_fd_flags(flags & O_CLOEXEC)?;
 10. FdTable::install(newfd, newfile);
 11. Audit::log_socketcall(SYS_ACCEPT4, fd, &storage, len, newfd, flags);
 12. Ok(newfd).
```

`sock_alloc_file` sets `file.f_flags |= (flags & O_NONBLOCK)` and `file.f_mode |= FMODE_NONOTIFY`. CLOEXEC is propagated through `get_unused_fd_flags` into the fd-table entry's `close_on_exec` bitmap atomically with the fd's reservation slot.

### Out of Scope

- `accept(2)` — separate Tier-5 (delegates here).
- `listen(2)` — separate Tier-5.
- TCP three-way handshake — Tier-3 `net/ipv4/tcp.md`.
- Implementation code.

### signature

```c
int accept4(int sockfd, struct sockaddr *addr,
            socklen_t *addrlen, int flags);
```

```rust
pub fn sys_accept4(
    sockfd: i32,
    addr: UserPtr<Sockaddr>,
    addrlen: UserPtr<u32>,
    flags: i32,
) -> SyscallResult<i32>;
```

x86-64 entry: `__x64_sys_accept4` → `__sys_accept4(fd, uaddr, uaddrlen, flags)`.

### parameters

- **`sockfd`** — listening socket fd.
- **`addr`** — optional userland buffer for peer address (NULL discards).
- **`addrlen`** — value-result `socklen_t *`. On entry: caller's buffer length. On return: actual sockaddr length.
- **`flags`** — bitwise OR of:
  - `SOCK_CLOEXEC` (= `O_CLOEXEC`, 0x80000) — atomically set FD_CLOEXEC on the new fd.
  - `SOCK_NONBLOCK` (= `O_NONBLOCK`, 0x800) — atomically set O_NONBLOCK on the new socket.
  - any other bit → `-EINVAL`.

### return

A new non-negative fd referring to the accepted connection, with CLOEXEC/NONBLOCK applied per `flags`.
`-1`/`-errno` on error.

### errors

(Same as `accept(2)` plus:)

- `EINVAL` — `flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK)` ≠ 0.
- `EBADF` — sockfd not a valid open fd.
- `ENOTSOCK` — fd not a socket.
- `EOPNOTSUPP` — socket does not support accept.
- `EFAULT` — addr or addrlen pointer fault.
- `EAGAIN` / `EWOULDBLOCK` — non-blocking listener with empty queue.
- `EMFILE` / `ENFILE` / `ENOMEM` — resource exhaustion.
- `ECONNABORTED` — pending connection aborted in queue.
- `EINTR` — blocking accept interrupted by signal.
- `EPROTO` / `EPROTONOSUPPORT` — protocol error during handshake.
- `EPERM` — netfilter denial.

### abi surface

Atomic setting of CLOEXEC/NONBLOCK is the entire point of `accept4`: the listener's flags are not inherited (no `O_NONBLOCK` leak), and there is no userspace observable window where the new fd exists without CLOEXEC set, defeating the accept-then-fork race that plagued `accept(2) + fcntl(F_SETFD)`.

`SOCK_CLOEXEC` and `O_CLOEXEC` are the same numeric bit; likewise `SOCK_NONBLOCK` and `O_NONBLOCK`. The kernel masks `flags` through `O_CLOEXEC | O_NONBLOCK` and passes the masked result to `sock_alloc_file` and `get_unused_fd_flags`.

### compatibility contract

REQ-1: flags validation:
- if `flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK)` != 0: return `-EINVAL`.

REQ-2: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-3: State + ops validation:
- if !sock.ops.accept: return `-EOPNOTSUPP`.

REQ-4: Newsock allocation:
- newsock = sock_alloc(); if !newsock: return `-ENFILE`.
- newsock.type = sock.type; newsock.ops = sock.ops.

REQ-5: Per-AF accept:
- err = sock.ops.accept(sock, newsock, sock.file.f_flags | (flags & O_NONBLOCK), /*kern=*/0).
- The (`flags & O_NONBLOCK`) bit is folded into the call so that during blocking-wait queueing the newsock's eventual file.f_flags include the requested bit.

REQ-6: Address writeback (if uaddr != NULL):
- len = newsock.ops.getname(newsock, &storage, /*peer=*/1).
- move_addr_to_user(&storage, len, uaddr, uaddrlen).

REQ-7: File wrap (atomic CLOEXEC/NONBLOCK):
- newfile = sock_alloc_file(newsock, flags & (O_CLOEXEC | O_NONBLOCK), NULL).
- fd = get_unused_fd_flags(flags & O_CLOEXEC).
- fd_install(fd, newfile).

REQ-8: Blocking semantics:
- if O_NONBLOCK on listener ∧ queue empty: return `-EAGAIN`.
- else wait_event_interruptible.

REQ-9: LSM:
- security_socket_accept(sock, newsock).

REQ-10: Audit:
- audit_log_socketcall(SYS_ACCEPT4, fd, &storage, len, newfd, flags).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `accept4_flags_strict` | INVARIANT | per-sys_accept4: any bit outside SOCK_CLOEXEC|SOCK_NONBLOCK ⟹ EINVAL. |
| `accept4_cloexec_atomic` | INVARIANT | per-fd_install: close_on_exec bit set before fd is visible. |
| `accept4_nonblock_atomic` | INVARIANT | per-sock_alloc_file: f_flags & O_NONBLOCK set before fd visible. |
| `accept4_no_partial_fd` | INVARIANT | per-error: newsock/newfile/newfd released. |
| `accept4_listener_unchanged` | INVARIANT | per-success: listener's sk_state remains TCP_LISTEN. |

### Layer 2: TLA+

`uapi/syscalls/accept4.tla`:
- States: validate-flags, validate-fd, alloc, dispatch, wait, name-copy, file-wrap, fd-install, return.
- Properties:
  - `safety_cloexec_atomic_with_fd_visibility` — for any interleaving with concurrent fork, child either does not see the fd at all, or sees it with FD_CLOEXEC set if SOCK_CLOEXEC requested.
  - `safety_flags_no_smuggle` — any flag bit outside the documented mask rejected.
  - `liveness_returns` — call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK) == 0 | `sys_accept4` |
| post: ret ≥ 0 ∧ (flags & SOCK_CLOEXEC) ⟹ close_on_exec[ret] == 1 | `sys_accept4` |
| post: ret ≥ 0 ∧ (flags & SOCK_NONBLOCK) ⟹ newfile.f_flags & O_NONBLOCK | `sys_accept4` |
| post: err ⟹ no fd-table mutation | `sys_accept4` |

### Layer 4: Verus/Creusot functional

Per-`__sys_accept4` semantic equivalence with `net/socket.c:__sys_accept4`. CLOEXEC/NONBLOCK atomic guarantee verified against `fs/file.c:__alloc_fd` ordering: `close_on_exec` set before fd-table `fd_array` write becomes RCU-visible.

### hardening

- **flags strict-masked to `SOCK_CLOEXEC | SOCK_NONBLOCK`** — defense against per-future-bit smuggling.
- **CLOEXEC set atomically with fd install** — defense against per-accept-fork-exec race (the canonical reason accept4 exists).
- **O_NONBLOCK set atomically on newfile** — defense against per-blocking-read-on-non-blocking-server race.
- **Newsock released on any failure** — defense against per-leaked-sock.
- **`move_addr_to_user` capped by caller's `*addrlen`** — defense against per-OOB-write.
- **LSM `socket_accept`** — defense against per-MAC bypass.
- **Per-`SOMAXCONN` ceiling** — defense against per-queue-overflow.
- **ECONNABORTED for RST during handshake** — defense against per-uninit-child delivery.
- **Listener's f_flags merged into accept_op call** — defense against per-blocking-listener with non-blocking accepted child (or vice versa) state mismatch.

### grsecurity-pax

- **PAX_RANDKSTACK** — per-syscall kstack base randomization protects on-stack `sockaddr_storage`.
- **PaX UDEREF** — userland `uaddr`/`uaddrlen` only reachable via `copy_{from,to}_user`; direct deref traps.
- **GRKERNSEC_NO_SIMULT_CONNECT** — peer-side limit reduces accept-queue saturation.
- **GRKERNSEC_BLACKHOLE** — peer-side reconnaissance defeat; the local server's `accept4` returns only completed handshakes.
- **GRKERNSEC_RANDNET on accept queue** — accept-queue order randomized; defeats timing side-channels that try to fingerprint which client connected first.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — inherited from listener; accept4 itself does not re-check (listener creation did).
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — UNIX-domain accept4 honors abstract-namespace isolation; `SO_PEERCRED` populated under siglock at accept time.
- **SOCK_CLOEXEC mandatory under suid** — when calling task has `PF_SUID_DUMP`/recent suid transition and `grsec.suid_strict_cloexec=1`, `flags` is force-ORed with `SOCK_CLOEXEC` regardless of caller request. This is the load-bearing grsec rule that motivates accept4 over accept.
- **SCM_RIGHTS recursion bound** — irrelevant on accept path.
- **getsockopt info-leak prevention** — newsock's `SO_PEERCRED`/`SO_PEERSEC` populated atomically with accept4; subsequent `getsockopt` reads are zero-padded.

