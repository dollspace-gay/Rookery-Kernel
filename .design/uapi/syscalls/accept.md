# Tier-5 syscall: accept(2) — accept a connection on a socket (syscall 43)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - net/socket.c (__sys_accept4, __sys_accept4_file, sys_accept)
  - net/ipv4/inet_connection_sock.c (inet_csk_accept)
  - net/unix/af_unix.c (unix_accept)
  - include/linux/socket.h (struct sockaddr)
-->

## Summary

`accept(2)` is a thin wrapper over `accept4(2)` with `flags == 0`: it removes the first established connection from the listen queue of `sockfd`, creates a new fd referring to that connection, and (optionally) fills `addr`/`addrlen` with the peer's address. It is syscall number **43** on x86-64.

The implementation path is `sys_accept(fd, uaddr, uaddrlen)` → `__sys_accept4(fd, uaddr, uaddrlen, /*flags=*/0)`. All semantics flow through the accept4 backend; see `accept4.md` for the parameterized form.

Critical for: every server (HTTP, SSH, database listeners), every UNIX-domain RPC server, every container's listen-then-dispatch loop.

This Tier-5 covers syscall **43** `accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen)`.

## Signature

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

```rust
pub fn sys_accept(
    sockfd: i32,
    addr: UserPtr<Sockaddr>,
    addrlen: UserPtr<u32>,
) -> SyscallResult<i32>;
```

x86-64 entry: `__x64_sys_accept` → `__sys_accept4(fd, uaddr, uaddrlen, 0)`.

## Parameters

- **`sockfd`** — listening socket fd previously placed in `LISTEN` state via `listen(2)`.
- **`addr`** — optional userland pointer; if NULL the kernel does not write the peer address. If non-NULL, the kernel writes up to `*addrlen` bytes of the peer's `struct sockaddr`.
- **`addrlen`** — value-result pointer. On entry: caller-provided buffer length. On return: actual sockaddr length (may exceed buffer length, in which case `*addr` is truncated but `*addrlen` reports the full length).

## Return

A new non-negative fd (the smallest unused) referring to the accepted connection.
`-1`/`-errno` on error.

## Errors

- `EBADF` — `sockfd` not a valid open fd.
- `ENOTSOCK` — fd not a socket.
- `EINVAL` — socket not in LISTEN state; `*addrlen` negative.
- `EFAULT` — `addr` or `addrlen` not writable.
- `EOPNOTSUPP` — socket type does not support accept (e.g. SOCK_DGRAM, SOCK_RAW).
- `EAGAIN` / `EWOULDBLOCK` — non-blocking socket and no pending connection.
- `EMFILE` / `ENFILE` — fd-table or system-wide file-limit reached.
- `ENOMEM` — slab allocation failure.
- `ECONNABORTED` — pending connection was aborted (RST during SYN_RCVD; `SO_LINGER` zero abort).
- `EPERM` — netfilter denial.
- `EPROTO` / `EPROTONOSUPPORT` — protocol error during handshake (TCP).
- `EINTR` — blocking accept interrupted by signal.
- `ERESTARTSYS` — restartable signal handling.

## ABI surface

- `addrlen` is a `socklen_t` (`u32`), passed by pointer (value-result).
- `addr` must accommodate at minimum the family's expected sockaddr, but the kernel writes only `min(actual_len, *addrlen)` bytes.
- The returned fd inherits `O_NONBLOCK` from the listening socket's `f_flags` only if the historical pre-accept4 contract applies — Linux current: returned fd does **not** inherit O_NONBLOCK from the listener (each newly-accepted socket starts blocking unless `accept4(SOCK_NONBLOCK)` is used).
- `O_CLOEXEC`: never set by `accept(2)`; use `accept4` for atomic CLOEXEC.

## Compatibility contract

REQ-1: Delegation:
- sys_accept(fd, uaddr, uaddrlen) ≡ sys_accept4(fd, uaddr, uaddrlen, /*flags=*/0).

REQ-2: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-3: State validation:
- if !sock.ops.accept: return `-EOPNOTSUPP`.
- if sock.sk.sk_state != TCP_LISTEN (or family equivalent): return `-EINVAL`.

REQ-4: Newsock allocation:
- newsock = sock_alloc(); if !newsock: return `-ENFILE`.
- newsock.type = sock.type.
- newsock.ops = sock.ops.

REQ-5: Per-AF accept:
- err = sock.ops.accept(sock, newsock, sock.file.f_flags, /*kern=*/0).
- For TCP: `inet_csk_accept()` dequeues from `icsk_accept_queue` (waits if blocking).
- For UNIX: `unix_accept()` dequeues from `sk.sk_receive_queue`.

REQ-6: Address writeback:
- if uaddr != NULL:
  - len = newsock.ops.getname(newsock, &storage, /*peer=*/1).
  - move_addr_to_user(&storage, len, uaddr, uaddrlen).

REQ-7: File wrap + fd install:
- newfile = sock_alloc_file(newsock, /*flags=*/0, NULL).
- fd = get_unused_fd_flags(0).
- fd_install(fd, newfile).

REQ-8: Blocking semantics:
- if O_NONBLOCK ∧ queue empty: return `-EAGAIN`.
- else: wait_event_interruptible(sk_sleep, !queue_empty ∨ signal_pending).

REQ-9: LSM:
- security_socket_accept(sock, newsock) before fd_install.

REQ-10: Audit:
- audit_log_socketcall(SYS_ACCEPT, fd, &storage, len, newfd).

## Acceptance Criteria

- [ ] AC-1: accept on listening TCP socket with pending conn: returns fd; getpeername(newfd) matches connector.
- [ ] AC-2: accept with NULL addr: returns fd; no address written.
- [ ] AC-3: accept with addr non-NULL, addrlen < actual: returns fd; `*addrlen` = actual; addr truncated.
- [ ] AC-4: accept on non-listening socket: `-EINVAL`.
- [ ] AC-5: accept on SOCK_DGRAM: `-EOPNOTSUPP`.
- [ ] AC-6: accept on non-blocking listener with empty queue: `-EAGAIN`.
- [ ] AC-7: blocking accept interrupted by SIGINT: `-EINTR`.
- [ ] AC-8: hitting RLIMIT_NOFILE: `-EMFILE`.
- [ ] AC-9: connection aborted in queue (RST before accept): `-ECONNABORTED`.
- [ ] AC-10: returned fd does NOT have FD_CLOEXEC set (vs accept4).
- [ ] AC-11: returned fd starts blocking regardless of listener's O_NONBLOCK.

## Architecture

```
SysSocket::sys_accept(fd, uaddr, uaddrlen) -> SyscallResult<i32>
  return Self::sys_accept4(fd, uaddr, uaddrlen, /*flags=*/0).

SysSocket::sys_accept4(fd, uaddr, uaddrlen, flags) -> SyscallResult<i32>
  1. sock = SocketFd::lookup_light(fd)?;
  2. accept_op = sock.ops.accept.ok_or(EOPNOTSUPP)?;
  3. if sock.sk.sk_state != TCP_LISTEN { return Err(EINVAL); }     // family-aware
  4. newsock = Socket::alloc().ok_or(ENFILE)?;
  5. newsock.type = sock.type; newsock.ops = sock.ops;
  6. err = accept_op(&sock, &newsock, sock.file.f_flags, /*kern=*/false);
     if err < 0 { Socket::release(newsock); return Err(err); }
  7. if let Some(uaddr_ptr) = uaddr.option() {
        let mut storage = SockaddrStorage::zeroed();
        let len = newsock.ops.getname(&newsock, &mut storage, /*peer=*/true)?;
        UserCopy::move_addr_to_user(&storage, len, uaddr_ptr, uaddrlen)?;
     }
  8. let newfile = Socket::alloc_file(newsock, flags & (O_CLOEXEC | O_NONBLOCK), None)?;
  9. let newfd = FdTable::get_unused_fd_flags(flags & O_CLOEXEC)?;
 10. FdTable::install(newfd, newfile);
 11. Audit::log_socketcall(SYS_ACCEPT, fd, &storage, len, newfd);
 12. Ok(newfd).
```

Per-`inet_csk_accept`:
1. lock_sock(sk).
2. if sk_state != TCP_LISTEN: -EINVAL.
3. if reqsk_queue_empty(&icsk.icsk_accept_queue):
   - if O_NONBLOCK: -EAGAIN.
   - else: inet_csk_wait_for_connect(sk, timeout) — wait_event_interruptible.
4. req = reqsk_queue_remove(&icsk.icsk_accept_queue, sk).
5. newsk = req.sk.
6. newsock.sk = newsk.
7. release_sock(sk).
8. return 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `accept_listener_state` | INVARIANT | per-sys_accept: pre-dispatch sk_state == TCP_LISTEN. |
| `accept_no_partial_fd` | INVARIANT | per-error: newsock/newfile/newfd all released. |
| `accept_addrlen_writeback_bounds` | INVARIANT | per-move_addr_to_user: bytes written ≤ caller buffer. |
| `accept_uaddr_optional` | INVARIANT | per-uaddr == NULL: getname not invoked. |
| `accept_cloexec_not_inherited` | INVARIANT | per-flags == 0: returned fd has FD_CLOEXEC == 0. |

### Layer 2: TLA+

`uapi/syscalls/accept.tla`:
- States: validate, alloc-newsock, dispatch, wait, name-copy, file-wrap, fd-install, return.
- Properties:
  - `safety_queue_dequeue_atomic` — accept dequeues exactly one entry per success.
  - `safety_econnaborted_skips` — RST-aborted requests yield -ECONNABORTED, advance to next.
  - `liveness_blocking_returns` — blocking accept eventually returns fd or -EINTR.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| post: ret ≥ 0 ⟹ fd-table[ret].file.f_op == &socket_file_ops | `sys_accept` |
| post: ret ≥ 0 ⟹ newsock.sk.sk_state == TCP_ESTABLISHED | `sys_accept` |
| post: err ⟹ no fd-table mutation | `sys_accept` |
| pre: uaddrlen valid ⟹ uaddrlen-deref ≥ 0 | `sys_accept` |

### Layer 4: Verus/Creusot functional

Per-`inet_csk_accept` semantic equivalence with `net/ipv4/inet_connection_sock.c`: dequeue order preserved (FIFO), `tcp_done` not called on accepted child, refcount transferred correctly.

## Hardening

- **State guard sk_state == TCP_LISTEN** — defense against per-stale-ops invocation on closed listener.
- **newsock fully initialized before fd_install** — defense against per-TOCTOU on new fd.
- **`move_addr_to_user` size capped by caller-provided `*addrlen`** — defense against per-OOB-write in user buffer.
- **Newsock release on any post-accept failure** — defense against per-leaked-sock on error path.
- **Per-`ECONNABORTED` for RST-during-handshake** — defense against per-uninit-conn delivery.
- **LSM `socket_accept`** — defense against per-MAC bypass.
- **Per-blocking accept signal-safe** — defense against per-signal-lost.
- **Per-EAGAIN clean state** — defense against per-spurious-EAGAIN.
- **Per-`SOMAXCONN` ceiling on accept-queue** — defense against per-queue-overflow (set by listen).

## Grsecurity-PaX

- **PAX_RANDKSTACK** — per-syscall kstack base randomization; `struct sockaddr_storage` is on-stack here.
- **PaX UDEREF** — every userland deref of `uaddr` and `uaddrlen` traps if outside the `copy_from/to_user` macros. Defends against per-confused-deputy.
- **GRKERNSEC_NO_SIMULT_CONNECT** — peer-side rate limit; reduces accept-queue pressure from coordinated clients.
- **GRKERNSEC_BLACKHOLE** — incoming SYN to closed ports get no RST, defeating reconnaissance from peers prior to a future accept binding.
- **GRKERNSEC_RANDNET on accept queue** — accept-queue order is randomized (jitter inserted) so that timing-side-channel attacks ("which connection was nth?") are defeated. Implemented in `inet_csk_reqsk_queue_drop_and_put` path with a per-listener salt.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — listener creation requires net-caps for SOCK_RAW; accept just inherits semantic.
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — UNIX-domain accept restricted: if listener path was in abstract namespace, accept'd connection's `SO_PEERCRED` cannot be spoofed via grsec-checked creds.
- **SOCK_CLOEXEC mandatory under suid** — `accept(2)` cannot atomically set CLOEXEC; grsec-policy forces SUID-tainted tasks to use `accept4(SOCK_CLOEXEC)` and rejects `accept(2)` via `-EPERM` when `grsec.suid_strict_cloexec=1`.
- **SCM_RIGHTS recursion bound** — not relevant on accept path.
- **getsockopt info-leak prevention** — newsock's `SO_PEERCRED`/`SO_PEERSEC` populated atomically with accept; no read-before-write hole.

## Open Questions

- Linux historically had a Linux-specific bug where `accept(2)` inherited some of the listener's flags into the new fd. Confirm 7.1.0-rc2 path does NOT inherit O_NONBLOCK; document the deliberate non-inheritance.

## Out of Scope

- `accept4(2)` — separate Tier-5 (covers flags).
- `listen(2)` — separate Tier-5.
- TCP three-way handshake child-sock allocation — Tier-3 `net/ipv4/tcp.md`.
- Implementation code.
