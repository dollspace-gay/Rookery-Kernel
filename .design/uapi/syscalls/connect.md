# Tier-5 syscall: connect(2) — initiate a connection on a socket (syscall 42)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - net/socket.c (__sys_connect, __sys_connect_file, move_addr_to_kernel)
  - net/ipv4/af_inet.c (inet_stream_connect, inet_dgram_connect)
  - net/ipv4/tcp_ipv4.c (tcp_v4_connect)
  - net/unix/af_unix.c (unix_stream_connect, unix_dgram_connect)
  - include/linux/socket.h (struct sockaddr, struct __kernel_sockaddr_storage)
-->

## Summary

`connect(2)` initiates the per-protocol "connect" sequence on a socket: for `SOCK_STREAM` it performs the full handshake (TCP three-way / UNIX stream rendezvous); for `SOCK_DGRAM` it merely sets the default peer for subsequent `send(2)` and filters incoming datagrams. It is syscall number **42** on x86-64.

The path is `sys_connect(fd, uaddr, addrlen)` → `__sys_connect()` → `move_addr_to_kernel()` (copy_from_user with `__kernel_sockaddr_storage` bounds) → `sock.ops.connect(sock, &addr, addrlen, flags)` → per-AF connect handler.

For non-blocking sockets `connect()` returns `-EINPROGRESS` immediately; completion is observed via `poll(POLLOUT)` + `getsockopt(SO_ERROR)`.

Critical for: every client networking flow (HTTP, SSH, DNS), every container-runtime sidecar dial, every privilege-separation pipe.

This Tier-5 covers syscall **42** `connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen)`.

## Signature

```c
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

```rust
pub fn sys_connect(sockfd: i32, addr: UserPtr<Sockaddr>, addrlen: u32) -> SyscallResult<i32>;
```

x86-64 entry: `__x64_sys_connect` → `__sys_connect(sockfd, uaddr, addrlen)`.

## Parameters

- **`sockfd`** — fd previously returned by `socket(2)` (or `accept(2)` though uncommon).
- **`addr`** — userland pointer to a `struct sockaddr` whose first 2 bytes (`sa_family`) select the family interpretation. Layout depends on family: `struct sockaddr_in`, `struct sockaddr_in6`, `struct sockaddr_un`, `struct sockaddr_ll`, `struct sockaddr_nl`, etc.
- **`addrlen`** — total bytes the kernel may read at `addr`. Must be ≤ `sizeof(struct __kernel_sockaddr_storage)` (128).

## Return

0 on success; `-1`/`-errno` on error. For non-blocking stream sockets returns `-EINPROGRESS` once the SYN/connect-attempt has been queued.

## Errors

- `EBADF` — `sockfd` not an open fd.
- `ENOTSOCK` — fd is open but not a socket (`f_op != &socket_file_ops`).
- `EINVAL` — `addrlen` invalid for the family; sock already in inappropriate state.
- `EFAULT` — `addr` not accessible (copy_from_user fault).
- `EAFNOSUPPORT` — `addr.sa_family` not the family the socket was created with (most families).
- `EACCES` / `EPERM` — destination is broadcast and `SO_BROADCAST` not set; LSM `socket_connect` denial; rule-based denial.
- `EADDRINUSE` — local endpoint already in use (ephemeral port exhaustion, IP_BIND_ADDRESS_NO_PORT scenarios).
- `EADDRNOTAVAIL` — local address not bound to any interface; ephemeral pool exhausted.
- `ENETUNREACH` / `EHOSTUNREACH` — no route.
- `ECONNREFUSED` — peer actively refused (TCP RST, UNIX no listener).
- `ETIMEDOUT` — connection attempt timed out (TCP SYN retries exhausted).
- `EISCONN` — socket already connected (stream).
- `ECONNRESET` / `ENOTCONN` — peer reset.
- `EALREADY` — non-blocking connect previously returned `EINPROGRESS` and is still pending.
- `EINPROGRESS` — non-blocking socket, handshake started.
- `EINTR` — signal during blocking wait.
- `EOPNOTSUPP` — connect not supported by ops (e.g. raw socket with no per-AF connect handler).

## ABI surface

`struct sockaddr` is a 16-byte legacy header. Real payloads can be up to 128 bytes (`struct __kernel_sockaddr_storage`). The kernel always validates `addrlen ≤ sizeof(struct sockaddr_storage)` and `≥ sizeof(sa_family_t)` before copy_from_user.

Per-`sock.sk.sk_state` transitions:
- TCP_CLOSE → TCP_SYN_SENT → TCP_ESTABLISHED on success.
- TCP_CLOSE → TCP_SYN_SENT → TCP_CLOSE on failure.
- UNIX: SS_UNCONNECTED → SS_CONNECTING → SS_CONNECTED.

## Compatibility contract

REQ-1: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.
- sock = SOCKET_I(f.f_inode).

REQ-2: addr copy:
- if addrlen < sizeof(sa_family_t) ∨ addrlen > sizeof(__kernel_sockaddr_storage): return `-EINVAL`.
- err = move_addr_to_kernel(uaddr, addrlen, &storage); if err: return err.
- if err == -EFAULT: return `-EFAULT`.

REQ-3: LSM:
- err = security_socket_connect(sock, &storage.addr, addrlen); if err: return err.

REQ-4: Dispatch:
- err = sock.ops.connect(sock, &storage.addr, addrlen, sock.file.f_flags).
- For SOCK_STREAM non-blocking: O_NONBLOCK ⟹ -EINPROGRESS once SYN sent.
- For SOCK_DGRAM: sets sk.sk_peer/sk_daddr; returns 0 immediately.

REQ-5: State guards:
- SOCK_STREAM: if sk_state == TCP_ESTABLISHED: return `-EISCONN`.
- SOCK_STREAM nonblocking + pending: return `-EALREADY`.
- SOCK_STREAM nonblocking start: return `-EINPROGRESS`.

REQ-6: Wait semantics (blocking):
- sock_sndtimeo(sock, flags & O_NONBLOCK).
- wait_event_interruptible_timeout(sk_sleep, sk_state ∈ {ESTABLISHED, CLOSE, ...}).
- if signal_pending: return `-EINTR`.

REQ-7: Audit:
- audit_log socketcall(SYS_CONNECT, fd, addr-truncated, addrlen, ret).

REQ-8: Cgroup BPF:
- BPF_CGROUP_RUN_PROG_INET4/6_CONNECT(sk, &storage.addr); may rewrite address; may deny → `-EPERM`.

REQ-9: SOL_IP options:
- IP_BIND_ADDRESS_NO_PORT defers ephemeral port until connect to avoid reservation.

## Acceptance Criteria

- [ ] AC-1: TCP connect to a listening peer: returns 0; getpeername matches.
- [ ] AC-2: TCP connect to closed port: returns `-ECONNREFUSED`.
- [ ] AC-3: TCP connect with O_NONBLOCK: returns `-EINPROGRESS`; subsequent `poll(POLLOUT)` triggers; `getsockopt(SO_ERROR)` reports 0 or err.
- [ ] AC-4: TCP connect re-issued after EINPROGRESS still pending: `-EALREADY`.
- [ ] AC-5: TCP connect on already-connected socket: `-EISCONN`.
- [ ] AC-6: UDP connect to address: returns 0; subsequent send without sockaddr uses peer.
- [ ] AC-7: UNIX-stream connect: returns 0 after server accept; or `-ECONNREFUSED` if path has no listener.
- [ ] AC-8: addrlen > 128: `-EINVAL`.
- [ ] AC-9: addr pointer in unmapped page: `-EFAULT`.
- [ ] AC-10: LSM denial → `-EACCES`.
- [ ] AC-11: BPF cgroup connect rewrite: kernel uses rewritten address.
- [ ] AC-12: SIGINT during blocking connect: `-EINTR`.

## Architecture

```
SysSocket::sys_connect(fd, uaddr, addrlen) -> SyscallResult<i32>
  1. sock = SocketFd::lookup_light(fd)?;          // -EBADF / -ENOTSOCK
  2. if !(sizeof(SaFamilyT) ..= 128).contains(&addrlen): return Err(EINVAL).
  3. storage = SockaddrStorage::zeroed();
  4. UserCopy::from_user(&mut storage[..addrlen], uaddr)?;   // -EFAULT
  5. Lsm::socket_connect(&sock, &storage, addrlen)?;
  6. CgroupBpf::inet_connect(&sock, &mut storage)?;           // may rewrite or deny
  7. err = sock.ops.connect(&sock, &storage, addrlen, sock.file.f_flags);
  8. Audit::log_socketcall(SYS_CONNECT, fd, &storage, addrlen, err);
  9. err.into()
```

Per-AF connect:
- `inet_stream_connect`: lock_sock, validate family/AF_INET, dispatch to `tcp_v4_connect`, then `inet_wait_for_connect` if blocking.
- `tcp_v4_connect`: route lookup → `ip_route_connect()` → choose source addr → `tcp_set_state(TCP_SYN_SENT)` → `tcp_connect()` builds SYN → `tcp_v4_send_check`, transmit.
- `unix_stream_connect`: lookup target inode/socket, `unix_create_addr()` for client side, queue on peer accept queue, sleep on `sk_sleep` until accepted or refused.
- `inet_dgram_connect`: sets sk_daddr / sk_dport; returns immediately.

`Socket::wait_for_connect(sk, timeout)`:
1. while sk_state == TCP_SYN_SENT ∨ TCP_SYN_RECV:
   - wait_event_interruptible_timeout(sk_sleep, sk_state ∉ {SYN_SENT, SYN_RECV}, &timeout).
   - if signal_pending: return `-EINTR`.
   - if timeout == 0: return `-EINPROGRESS` (or `-EAGAIN` for some families).
2. err = sock_error(sk); return err or 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `connect_addrlen_bounds` | INVARIANT | per-sys_connect: addrlen ∈ [2, 128]; else EINVAL. |
| `connect_storage_zeroed_before_copy` | INVARIANT | per-sys_connect: storage zeroed before copy_from_user. |
| `connect_no_partial_copy_use` | INVARIANT | per-EFAULT: storage not used downstream. |
| `connect_state_progress` | INVARIANT | per-tcp_v4_connect: TCP_CLOSE → TCP_SYN_SENT atomic. |
| `connect_einprogress_only_when_nonblocking` | INVARIANT | per-blocking sock: EINPROGRESS unreachable from sys_connect. |

### Layer 2: TLA+

`uapi/syscalls/connect.tla`:
- States: validate, copy, lsm, bpf, dispatch, wait, return.
- Models TCP_CLOSE → TCP_SYN_SENT → {TCP_ESTABLISHED, TCP_CLOSE} transitions.
- Properties:
  - `safety_state_monotonic_per_attempt` — per-connect: sk_state never regresses except via error path resetting to CLOSE.
  - `safety_einprogress_implies_nonblock` — ret == -EINPROGRESS ⟹ O_NONBLOCK set.
  - `liveness_blocking_terminates` — blocking connect eventually returns (timeout-bounded).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: addrlen ≥ sizeof(sa_family_t) | `sys_connect` |
| post: ret == 0 ⟹ sk_state == ESTABLISHED (stream) ∨ sk_peer set (dgram) | `sys_connect` |
| post: ret < 0 ⟹ no peer/state mutation visible to user | `sys_connect` |
| post: cgroup-bpf may rewrite storage but not exceed 128 B | `CgroupBpf::inet_connect` |

### Layer 4: Verus/Creusot functional

Per-`inet_stream_connect → tcp_v4_connect` semantic equivalence with `net/ipv4/tcp_ipv4.c`. SYN packet header fields (src/dst addr, src/dst port, ISN per RFC 6528 hash) match upstream byte-for-byte.

## Hardening

- **addrlen strictly bounded to `sizeof(struct sockaddr_storage)`** — defense against per-OOB-read of kernel stack via crafted addrlen.
- **storage zero-initialized before copy_from_user** — defense against per-info-leak of prior stack.
- **`move_addr_to_kernel` returns -EFAULT on partial copy; caller must not use storage** — defense against per-uninitialized-byte propagation.
- **LSM `socket_connect` hook** — defense against per-MAC bypass.
- **cgroup-BPF `INET_{4,6}_CONNECT` hook** — defense against per-egress-policy bypass.
- **Per-`SO_BROADCAST` required for connecting to broadcast** — defense against per-amplification.
- **Per-`ip_local_reserved_ports`** — defense against per-reserved-port hijack.
- **Per-`IP_BIND_ADDRESS_NO_PORT` deferred ephemeral allocation** — defense against per-ephemeral-port-starvation.
- **Per-TCP simultaneous-connect rate limit** — defense against per-SYN-flood from local task.

## Grsecurity-PaX

- **PAX_RANDKSTACK** — randomizes kernel stack base per-syscall; defeats stack-relative info-leak from `__kernel_sockaddr_storage` aliasing.
- **PaX UDEREF** — `move_addr_to_kernel` runs with UDEREF enabled: any direct userland deref outside the dedicated `copy_from_user` path traps. Guards against per-confused-deputy reads of `uaddr` after copy.
- **GRKERNSEC_NO_SIMULT_CONNECT** — per-task simultaneous-connect attempts to the same `(daddr, dport)` are rate-throttled (default ≤ 200 inflight). Defeats SYN scanning from compromised processes and bounds half-open connection table growth.
- **GRKERNSEC_BLACKHOLE** — TCP SYNs to closed ports return no RST (silently dropped). connect() returns `-ETIMEDOUT` instead of `-ECONNREFUSED`, defeating port-scan reconnaissance.
- **GRKERNSEC_RANDNET** — randomizes initial sequence numbers (stronger than RFC 6528), source-port selection, and IP ID; perturbs the accept queue ordering on the peer.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — raw-socket connect requires CAP_NET_RAW in the *initial* user-ns under grsec-hardened build.
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — abstract-namespace UNIX connect restricted to creating user when abstract path; pathname UNIX subject to chrooted-restricted policy.
- **SOCK_CLOEXEC mandatory under suid** — not directly relevant to connect (fd already exists) but persists through accept(4) flag inheritance.
- **SCM_RIGHTS recursion bound** — irrelevant here (connect carries no cmsgs) but consistent across socket-family.
- **getsockopt info-leak prevention** — connect populates `sk.sk_err` only on failure; SO_ERROR readout zero-padded.

## Open Questions

- Should `EHOSTUNREACH` vs `ENETUNREACH` discrimination follow upstream exactly, or use a single `EHOSTUNREACH` path? (Per RFC 1122 they are distinct; Linux upstream tracks both.)
- BPF cgroup connect rewrite: do we expose the rewritten address back to userspace via getsockname? Upstream: yes for getpeername after rewrite.

## Out of Scope

- `bind(2)` — separate Tier-5.
- TCP three-way handshake state machine — Tier-3 `net/ipv4/tcp.md`.
- UNIX-domain connect queue — Tier-3 `net/unix/af_unix.md`.
- Implementation code.
