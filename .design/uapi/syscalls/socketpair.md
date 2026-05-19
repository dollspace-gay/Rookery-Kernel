# Tier-5 syscall: socketpair(2) — create a pair of connected sockets (syscall 53)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - net/socket.c (__sys_socketpair, sys_socketpair, sock_create_lite, sock_alloc_file)
  - net/unix/af_unix.c (unix_socketpair)
  - net/socket.c (sock_create)
  - include/uapi/linux/net.h (SOCK_CLOEXEC, SOCK_NONBLOCK)
-->

## Summary

`socketpair(2)` creates two connected unnamed sockets of the same family and type, and returns the pair as a 2-element `int sv[2]` array of file descriptors. It is conceptually a `pipe(2)` that yields bidirectional sockets instead of half-duplex pipes. Only `AF_UNIX` is supported (AF_INET / AF_INET6 cannot produce a connected pair atomically because that requires bind+listen+accept+connect; socketpair has no addressing). It is syscall number **53** on x86-64.

The path is `sys_socketpair(domain, type, protocol, usv)` → `__sys_socketpair()` → two `sock_create_lite` calls → `sock1.ops.socketpair(sock1, sock2)` (i.e. `unix_socketpair`) → two `sock_alloc_file` calls → `put_user` of the fd pair into `usv[]`. The pair lives in the calling task's fd table.

Critical for: every pipe-like full-duplex IPC, every `fork`-based parent/child control channel, every privilege-separation handshake (the bridge process keeps one fd, hands the other across an `execve`), every `socketpair` test of cmsg passing (SCM_RIGHTS without going through a path-bound listener).

This Tier-5 covers syscall **53** `socketpair(int domain, int type, int protocol, int sv[2])`.

## Signature

```c
int socketpair(int domain, int type, int protocol, int sv[2]);
```

```rust
pub fn sys_socketpair(domain: i32, type_: i32, protocol: i32, sv: UserPtr<[i32; 2]>)
    -> SyscallResult<i32>;
```

x86-64 entry: `__x64_sys_socketpair` → `__sys_socketpair(family, type, protocol, usockvec)`.

## Parameters

- **`domain`** — must be `AF_UNIX` (a.k.a. `AF_LOCAL`, value 1). Other families return `-EOPNOTSUPP`/`-EAFNOSUPPORT`.
- **`type`** — `SOCK_STREAM` (1) | `SOCK_DGRAM` (2) | `SOCK_SEQPACKET` (5) | `SOCK_RAW` (3, root only) — OR'd with `SOCK_CLOEXEC` (0o02000000) and/or `SOCK_NONBLOCK` (0o00004000).
- **`protocol`** — 0 (default UNIX protocol) is the only valid value; non-zero returns `-EPROTONOSUPPORT`.
- **`sv`** — userland pointer to a 2-element `int` array; on success, populated with `(fd1, fd2)`.

## Return

0 on success; `-1`/`-errno` on error. On success, `sv[0]` and `sv[1]` are valid fds; on failure, `sv` is unchanged.

## Errors

- `EAFNOSUPPORT` — domain not supported (only `AF_UNIX` accepted).
- `EOPNOTSUPP` — type not supported by AF_UNIX (`SOCK_RAW` requires `CAP_NET_RAW`).
- `EPROTONOSUPPORT` — protocol != 0.
- `EINVAL` — type contains bits other than the type ∪ `SOCK_CLOEXEC` ∪ `SOCK_NONBLOCK`.
- `EFAULT` — `sv` pointer fault.
- `EMFILE` — per-process fd-table full (would need two fds and only ≤1 slot remains).
- `ENFILE` — system-wide fd limit reached.
- `ENOMEM` — slab alloc fail for `struct socket` / `struct sock` / file.
- `EACCES` / `EPERM` — LSM denial; chroot denial under grsec.

## ABI surface

- `SOCK_CLOEXEC` — if set in `type`, both new fds get `O_CLOEXEC`.
- `SOCK_NONBLOCK` — if set in `type`, both new sockets get `O_NONBLOCK`.
- `type & SOCK_TYPE_MASK` (0xF) extracts the actual type; upper bits are flags.
- `sv[]` ordering: `sv[0]` and `sv[1]` are symmetric (no "client/server" distinction).
- AF_UNIX socketpair is always "connected" semantically (`sk.sk_state == TCP_ESTABLISHED`-equivalent; peer points to the other socket).

## Compatibility contract

REQ-1: domain check:
- if domain != AF_UNIX: return `-EAFNOSUPPORT`.

REQ-2: type flags check:
- flags = type & ~SOCK_TYPE_MASK.
- if flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK): return `-EINVAL`.
- raw_type = type & SOCK_TYPE_MASK.
- if raw_type not in {SOCK_STREAM, SOCK_DGRAM, SOCK_SEQPACKET, SOCK_RAW}: return `-EOPNOTSUPP`/`-ESOCKTNOSUPPORT`.
- if raw_type == SOCK_RAW ∧ !ns_capable(net.user_ns, CAP_NET_RAW): return `-EPERM`.

REQ-3: protocol check:
- if protocol != 0: return `-EPROTONOSUPPORT`.

REQ-4: alloc two struct socket (sock1, sock2) via `sock_create_lite(family, type, protocol)` x2.
- on fail of either: free the first; return `-ENOMEM` / `-EMFILE`/`-ENFILE`.

REQ-5: LSM:
- err = security_socket_post_create(sock1, ...); security_socket_post_create(sock2, ...).
- err = security_socketpair(sock1, sock2); if err: free both; return err.

REQ-6: per-AF socketpair:
- err = sock1.ops.socketpair(sock1, sock2) — i.e. `unix_socketpair`.
- on fail: free both sockets; return err.

REQ-7: file alloc + fd alloc:
- file1 = sock_alloc_file(sock1, flags); file2 = sock_alloc_file(sock2, flags).
- fd1 = get_unused_fd_flags(flags); fd2 = get_unused_fd_flags(flags).
- on any partial failure: put_unused_fd / fput as appropriate; return err.

REQ-8: copy fd-pair to user FIRST, then install:
- err = put_user(fd1, &usv[0]); err |= put_user(fd2, &usv[1]).
- if err: put_unused_fd(fd1); put_unused_fd(fd2); fput(file1); fput(file2); return `-EFAULT`.

REQ-9: install both atomically:
- fd_install(fd1, file1).
- fd_install(fd2, file2).

REQ-10: Audit:
- audit_log_socketcall(SYS_SOCKETPAIR, fd1, fd2, type, protocol).

## Acceptance Criteria

- [ ] AC-1: socketpair(AF_UNIX, SOCK_STREAM, 0, sv): success; sv[0] and sv[1] connected.
- [ ] AC-2: socketpair(AF_UNIX, SOCK_DGRAM, 0, sv): success; sendto/recvfrom round-trip works without bind.
- [ ] AC-3: socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sv): success; record boundaries preserved.
- [ ] AC-4: socketpair(AF_INET, SOCK_STREAM, 0, sv): -EOPNOTSUPP.
- [ ] AC-5: socketpair(AF_UNIX, SOCK_STREAM, 6, sv): -EPROTONOSUPPORT (TCP protocol invalid for UNIX).
- [ ] AC-6: socketpair(AF_UNIX, SOCK_STREAM | 0x80000000, 0, sv): -EINVAL (unknown flag).
- [ ] AC-7: socketpair(AF_UNIX, SOCK_STREAM | SOCK_CLOEXEC, 0, sv): both fds have FD_CLOEXEC.
- [ ] AC-8: socketpair(AF_UNIX, SOCK_STREAM | SOCK_NONBLOCK, 0, sv): both sockets non-blocking.
- [ ] AC-9: socketpair with sv pointing to fault: -EFAULT; no fds leaked.
- [ ] AC-10: socketpair under RLIMIT_NOFILE with only 1 slot: -EMFILE; no fds leaked.
- [ ] AC-11: getsockname on socketpair fds: returns sun_family=AF_UNIX, no path (autobind not used).
- [ ] AC-12: SCM_RIGHTS passing across the pair: receiver gets installed fd.
- [ ] AC-13: socketpair(AF_UNIX, SOCK_RAW, 0, sv) unprivileged: -EPERM.
- [ ] AC-14: close(sv[0]); read(sv[1]): returns 0 (EOF) for stream; recv returns 0.

## Architecture

```
SysSocket::sys_socketpair(domain, type_, protocol, usv) -> SyscallResult<i32>
  1. if domain != AF_UNIX { return Err(EOPNOTSUPP); }
  2. let flags = type_ & !SOCK_TYPE_MASK;
  3. if flags & !(SOCK_CLOEXEC | SOCK_NONBLOCK) != 0 { return Err(EINVAL); }
  4. let raw_type = type_ & SOCK_TYPE_MASK;
  5. if raw_type == SOCK_RAW && !UserNs::current().net_ns.capable(CAP_NET_RAW) {
        return Err(EPERM);
     }
  6. if protocol != 0 { return Err(EPROTONOSUPPORT); }
  7. let sock1 = SockCreate::lite(AF_UNIX, raw_type, 0)?;
  8. let sock2 = SockCreate::lite(AF_UNIX, raw_type, 0).map_err(|e| {
        SockCreate::drop(sock1); e
     })?;
  9. Lsm::socket_post_create(&sock1, AF_UNIX, raw_type, 0)?;
 10. Lsm::socket_post_create(&sock2, AF_UNIX, raw_type, 0)?;
 11. Lsm::socketpair(&sock1, &sock2)?;
 12. UnixOps::socketpair(&sock1, &sock2)?;
 13. let file1 = SockAllocFile::alloc(&sock1, flags)?;
 14. let file2 = SockAllocFile::alloc(&sock2, flags)?;
 15. let fd1 = FdTable::get_unused_fd_flags(flags)?;
 16. let fd2 = FdTable::get_unused_fd_flags(flags).map_err(|e| {
        FdTable::put_unused_fd(fd1); e
     })?;
 17. UserCopy::put_user(fd1, &usv[0]).and_then(|_| UserCopy::put_user(fd2, &usv[1]))
        .map_err(|e| {
            FdTable::put_unused_fd(fd1);
            FdTable::put_unused_fd(fd2);
            FileRef::fput(file1);
            FileRef::fput(file2);
            EFAULT
        })?;
 18. FdTable::install(fd1, file1);
 19. FdTable::install(fd2, file2);
 20. Audit::log_socketpair(fd1, fd2, type_, protocol);
 21. Ok(0)
```

`unix_socketpair(sock1, sock2)`:
1. let sk1 = sock1.sk; let sk2 = sock2.sk.
2. unix_state_lock_double(sk1, sk2).
3. unix_peer(sk1).set(sk2); sock_hold(sk2).
4. unix_peer(sk2).set(sk1); sock_hold(sk1).
5. sk1.sk_state = TCP_ESTABLISHED; sk2.sk_state = TCP_ESTABLISHED. (for STREAM/SEQPACKET)
6. (for DGRAM: sk_state remains TCP_CLOSE; peer pointer is the "connection")
7. unix_state_unlock_double(sk1, sk2).
8. return Ok(()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `socketpair_unix_only` | INVARIANT | per-sys_socketpair: domain == AF_UNIX else error before any alloc. |
| `socketpair_atomic_fd_install` | INVARIANT | per-sys_socketpair: success ⟹ both fds visible at the same scheduling point. |
| `socketpair_no_partial_state` | INVARIANT | per-failure: zero fds leaked; both struct socket freed; both struct file freed. |
| `socketpair_flags_propagated` | INVARIANT | per-SOCK_CLOEXEC: both fds have FD_CLOEXEC; per-SOCK_NONBLOCK: both socks have O_NONBLOCK. |
| `socketpair_peer_bidi` | INVARIANT | per-unix_socketpair: peer(sk1) == sk2 ∧ peer(sk2) == sk1 ∧ refcount on each held by the other. |
| `socketpair_raw_capable` | INVARIANT | per-SOCK_RAW: ns_capable(CAP_NET_RAW) checked before alloc. |

### Layer 2: TLA+

`uapi/syscalls/socketpair.tla`:
- States: validate, alloc-sock1, alloc-sock2, lsm, dispatch-unix-pair, alloc-file1, alloc-file2, alloc-fd1, alloc-fd2, put-user, install-fd1, install-fd2, return.
- Properties:
  - `safety_no_fd_leak_on_error` — any error state ⟹ rollback: both struct socket freed, both files freed, both unused-fds released.
  - `safety_atomic_visibility` — both fds become visible to userspace only after put_user succeeds.
  - `safety_peer_bidirectional` — success ⟹ sk1.peer == sk2 ∧ sk2.peer == sk1.
  - `safety_flags_consistent` — both fds carry identical CLOEXEC/NONBLOCK bits.
  - `liveness_terminates` — socketpair returns within bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: domain == AF_UNIX | `sys_socketpair` |
| pre: protocol == 0 | `sys_socketpair` |
| pre: (type & ~SOCK_TYPE_MASK) & ~(SOCK_CLOEXEC \| SOCK_NONBLOCK) == 0 | `sys_socketpair` |
| post: success ⟹ both fds installed ∧ both refcount == 1 ∧ peer pointers bidirectional | `sys_socketpair` |
| post: failure ⟹ zero new fds in caller's fd table | `sys_socketpair` |
| post: success ⟹ caller-visible put_user(fd1) ∧ put_user(fd2) completed before either fd_install | `sys_socketpair` |

### Layer 4: Verus/Creusot functional

Per-`unix_socketpair` semantic equivalence with `net/unix/af_unix.c`: peer-pointer assignment order, `sock_hold` refcount bumps, `sk_state` transitions for STREAM/SEQPACKET vs DGRAM exactly match. `sock_alloc_file` + `fd_install` pairing matches `net/socket.c`.

## Hardening

- **fd-pair install happens AFTER `put_user` succeeds** — defense against per-leaked-fd when user buffer faults.
- **`get_unused_fd_flags` reserved before file alloc, then `fd_install` after put_user** — defense against per-TOCTOU-fd race.
- **Both `struct socket` allocated before any `sock_alloc_file`** — defense against per-half-failure with one file alive.
- **`unix_state_lock_double` orders locks deterministically (ptr-compare)** — defense against per-lock-order inversion.
- **Per-error rollback path** — defense against per-leaked-socket / per-leaked-file / per-leaked-fd.
- **`SOCK_CLOEXEC` and `SOCK_NONBLOCK` propagated to both files atomically** — defense against per-flag-skew between the pair.
- **AF_UNIX-only restriction** — defense against per-AF unsupported family causing kernel WARN.
- **Protocol-0 restriction** — defense against per-protocol-0-only assumption later in `unix_socketpair`.
- **CAP_NET_RAW for SOCK_RAW** — defense against per-unprivileged raw socketpair (UNIX-RAW does exist).
- **LSM `socketpair`** — defense against per-MAC bypass.

## Grsecurity-PaX

- **PAX_RANDKSTACK** — kstack base randomized; `sv[]` staging buffer randomized.
- **PaX UDEREF** — `usv` deref only via `put_user`; direct kernel deref traps.
- **GRKERNSEC_NO_SIMULT_CONNECT** — irrelevant on socketpair (no connect race).
- **GRKERNSEC_BLACKHOLE** — irrelevant on socketpair (no packet emitted).
- **GRKERNSEC_RANDNET** — irrelevant on socketpair (no port allocation).
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — `SOCK_RAW` socketpair gated to `CAP_NET_RAW` in the network namespace.
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — socketpair always creates *unnamed* UNIX sockets, so they bypass abstract-namespace ACLs. Hardened policy logs each socketpair under `grsec.audit.ipc_unnamed` to allow operator to spot heavy socketpair use (e.g., a process attempting massive parallel fd-passing).
- **SOCK_CLOEXEC mandatory under suid** — grsec `suid_strict_cloexec=1` forces `SOCK_CLOEXEC` to be implied on both fds when the caller has recently transitioned through SUID. Defeats classic socketpair-leak-across-execve attack vector.
- **SCM_RIGHTS recursion bound** — `current.scm_depth` bound applies to subsequent sendmsg on the pair, not to socketpair itself.
- **getsockopt info-leak prevention** — irrelevant on socketpair (no sockopt read).
- **sendmmsg compound-bound limit** — irrelevant on socketpair.
- **splice fd-permission boundary** — irrelevant on socketpair.

## Open Questions

- Should grsec ever permit `AF_INET` socketpair (some BSD systems do via implicit loopback connect)? Upstream Linux + grsec: no.
- For `SOCK_DGRAM` socketpair, do we autobind both ends to abstract names so they appear in `unix_diag`? Upstream: no — they remain unnamed. We follow.

## Out of Scope

- `socket(2)` — separate Tier-5.
- `pipe2(2)` — separate Tier-5 (half-duplex analog).
- `unix_socketpair` internal details — Tier-3 `net/unix/af_unix.md`.
- `SCM_RIGHTS` mechanics over the pair — covered in `sendmsg.md` Tier-5.
- Implementation code.
