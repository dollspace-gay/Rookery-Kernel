---
title: "Tier-5 syscall: getpeername(2) — return the peer's address of a connected socket (syscall 52)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`getpeername(2)` returns the remote address of a **connected** socket. For Internet sockets it is the peer's `(addr, port)`; for UNIX-stream/SEQPACKET it is the peer's bound name (or empty if the peer is unbound); for connected UDP it is the destination set by `connect(2)`; for netlink it is the destination portid. It is syscall number **52** on x86-64.

The path is `sys_getpeername(fd, uaddr, uaddr_len)` → `__sys_getpeername()` → `sock.ops.getname(sock, &storage, /*peer=*/1)` → `move_addr_to_user(&storage, kernel_namelen, uaddr, uaddr_len)`. It shares all post-dispatch handling with `getsockname(2)`; only the `peer=1` flag and the unconnected-socket error code differ.

Critical for: every accepted server connection that wants to log the client's address; every connected UDP client that wants to recover the destination; every UNIX-socket service that wants to inspect the peer pid via credentials side-channel (although peer-creds use `SO_PEERCRED`, not getpeername).

This Tier-5 covers syscall **52** `getpeername(int sockfd, struct sockaddr *addr, socklen_t *addrlen)`.

### Acceptance Criteria

- [ ] AC-1: getpeername on accepted TCP socket: returns client's (addr, port); *addrlen == 16.
- [ ] AC-2: getpeername on listening TCP socket (LISTEN state): -ENOTCONN.
- [ ] AC-3: getpeername on unbound TCP socket: -ENOTCONN.
- [ ] AC-4: getpeername on connect()ed UDP: returns destination set by connect; *addrlen == 16.
- [ ] AC-5: getpeername on unconnected UDP: -ENOTCONN.
- [ ] AC-6: getpeername on UNIX-stream peer-bound: returns peer path or empty (sa_family_t only for autobind/unbound peer).
- [ ] AC-7: getpeername on UNIX-stream not-connected: -ENOTCONN.
- [ ] AC-8: getpeername with input_len = 8 (truncated): copies 8 bytes; *addrlen == 16 (full length).
- [ ] AC-9: getpeername with *addrlen = (u32)-1: -EINVAL.
- [ ] AC-10: getpeername on closed fd: -EBADF.
- [ ] AC-11: getpeername on pipe fd: -ENOTSOCK.
- [ ] AC-12: getpeername on raw socket: -ENOTCONN (raw is never "connected" in the strict TCP sense unless explicitly connect()ed).

### Architecture

```
SysSocket::sys_getpeername(fd, uaddr, uaddr_len) -> SyscallResult<i32>
  1. sock = SocketFd::lookup_light(fd)?;
  2. let input_len: u32 = UserCopy::get_user(uaddr_len)?;
  3. if (input_len as i32) < 0 { return Err(EINVAL); }
  4. let mut storage = SockaddrStorage::zeroed();
  5. Lsm::socket_getpeername(&sock)?;
  6. let kernel_namelen = sock.ops.getname(&sock, &mut storage, /*peer=*/true)?;
  7. UserCopy::move_addr_to_user(&storage, kernel_namelen, uaddr, uaddr_len)?;
  8. Audit::log_socketcall(SYS_GETPEERNAME, fd, &storage, kernel_namelen, 0);
  9. Ok(0)
```

`inet_getname(sock, storage, peer=1)` (AF_INET):
1. let inet = sock.sk.inet_sock().
2. if !inet.inet_dport ∨ sk.sk_state == TCP_CLOSE/TCP_LISTEN: return Err(ENOTCONN).
3. storage.sin_family = AF_INET.
4. storage.sin_port = inet.inet_dport.
5. storage.sin_addr = inet.inet_daddr.
6. memset(storage.sin_zero, 0, 8).
7. return Ok(sizeof(struct sockaddr_in)).

`inet6_getname(sock, storage, peer=1)` (AF_INET6):
1. if sk.sk_state ∈ {TCP_CLOSE, TCP_LISTEN}: return Err(ENOTCONN).
2. storage.sin6_family = AF_INET6.
3. storage.sin6_port = sk.dport.
4. storage.sin6_flowinfo = sk6.flowinfo.
5. storage.sin6_addr = sk6.daddr.
6. storage.sin6_scope_id = sk6.scope_id if link-local.
7. return Ok(sizeof(struct sockaddr_in6)).

`unix_getname(sock, storage, peer=1)` (AF_UNIX):
1. unix_state_lock(sk).
2. let peer = unix_peer_get(sk); if !peer: unix_state_unlock(sk); return Err(ENOTCONN).
3. let addr = peer.unix_sock().addr.
4. storage.sun_family = AF_UNIX.
5. if addr.is_none(): unix_state_unlock(sk); return Ok(sizeof(sa_family_t)).
6. memcpy(storage.sun_path, addr.name, addr.len).
7. unix_state_unlock(sk).
8. return Ok(offsetof(sun_path) + addr.len).

### Out of Scope

- `getsockname(2)` — separate Tier-5 (`peer=0` variant of same dispatch).
- `accept(2)` / `accept4(2)` — separate Tier-5 (these also return peer address inline).
- Per-AF `sock.ops.getname` implementations — Tier-3 per family.
- `SO_PEERCRED`, `SO_PEERSEC` — separate Tier-3 sockopt docs.
- Implementation code.

### signature

```c
int getpeername(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

```rust
pub fn sys_getpeername(sockfd: i32, addr: UserPtr<Sockaddr>, addrlen: UserPtr<u32>)
    -> SyscallResult<i32>;
```

x86-64 entry: `__x64_sys_getpeername` → `__sys_getpeername(fd, uaddr, uaddr_len)`.

### parameters

- **`sockfd`** — fd from `socket(2)` / `accept(2)` / `accept4(2)`.
- **`addr`** — userland sockaddr buffer; kernel writes up to `min(kernel_namelen, *addrlen)` bytes.
- **`addrlen`** — IN/OUT: on entry, capacity of `addr`; on exit, the kernel's full address length.

### return

0 on success; `-1`/`-errno` on error.

### errors

- `EBADF` — sockfd not open.
- `ENOTSOCK` — fd not a socket.
- `EFAULT` — addr or addrlen pointer fault.
- `EINVAL` — `*addrlen` negative on entry.
- `ENOTCONN` — socket not connected (the central error vs getsockname).
- `ENOBUFS` — kmem alloc fail (rare).
- `EOPNOTSUPP` — family does not implement peer dispatch.

### abi surface

Identical to `getsockname(2)` on the writeback side. The differences:
- per-AF dispatcher reads from peer-side fields (`inet.inet_dport`/`inet.inet_daddr` for AF_INET; `sk6.daddr`/`sk6.dport` for AF_INET6; `unix_sk.peer` for AF_UNIX; `nlk.dst_pid`/`nlk.dst_group` for AF_NETLINK).
- if not connected: returns `-ENOTCONN`.
- AF_PACKET has no peer concept: returns sll_ifindex=0 / `-EOPNOTSUPP` depending on family. Linux upstream: AF_PACKET returns 0 with the bound interface (i.e. behaves as getsockname). We follow.

### compatibility contract

REQ-1: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-2: addrlen read-in:
- err = get_user(input_len, uaddr_len); if err: return `-EFAULT`.
- if (int)input_len < 0: return `-EINVAL`.

REQ-3: storage zero:
- storage = struct sockaddr_storage zeroed (128 bytes).

REQ-4: LSM:
- err = security_socket_getpeername(sock); if err: return err.

REQ-5: Per-AF dispatch with peer=1:
- err = sock.ops.getname(sock, &storage.addr, /*peer=*/1).
- if !connected (sk.sk_state ≠ TCP_ESTABLISHED for TCP; sk.peer == NULL for UNIX-stream; sk.daddr == 0 for UDP without prior connect): return `-ENOTCONN`.

REQ-6: Move to user:
- err = move_addr_to_user(&storage, kernel_namelen, uaddr, uaddr_len).
- copy min(kernel_namelen, input_len) bytes.
- write kernel_namelen back to *uaddr_len.

REQ-7: Audit:
- audit_log_socketcall(SYS_GETPEERNAME, fd, &storage, kernel_namelen, ret).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `getpeername_unconnected_rejected` | INVARIANT | per-AF getname(peer=1): not-connected ⟹ ENOTCONN. |
| `getpeername_storage_zeroed` | INVARIANT | per-sys_getpeername: storage zeroed before dispatch. |
| `getpeername_truncation_indicated` | INVARIANT | per-sys_getpeername: *addrlen on return == kernel_namelen. |
| `getpeername_copy_bounded` | INVARIANT | per-move_addr_to_user: bytes copied ≤ min(kernel_namelen, input_len). |
| `getpeername_no_info_leak` | INVARIANT | per-sys_getpeername: storage bytes beyond kernel_namelen never written to user. |
| `getpeername_peer_refcount_balanced` | INVARIANT | per-unix_getname(peer=1): every unix_peer_get balanced by sock_put on the local path. |

### Layer 2: TLA+

`uapi/syscalls/getpeername.tla`:
- States: validate, lsm, dispatch-getname-peer, check-connected, copy-bytes, copy-namelen, return.
- Properties:
  - `safety_only_connected_returns_peer` — return success ⟹ socket was connected at moment of dispatch.
  - `safety_namelen_reported_full` — *addrlen on return == kernel_namelen.
  - `safety_address_bytes_bounded` — bytes copied ≤ min(kernel_namelen, input_len).
  - `safety_no_stack_leak` — uninitialized storage bytes never observed.
  - `liveness_terminates` — getpeername returns within bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: input_len reinterpreted as i32 ≥ 0 | `sys_getpeername` |
| pre: storage zeroed | `sys_getpeername` |
| post: success ⟹ socket connected at dispatch | per-AF getname(peer=1) |
| post: *addrlen out = kernel_namelen | `move_addr_to_user` |
| post: bytes copied ≤ min(kernel_namelen, input_len) | `move_addr_to_user` |
| post: unix-stream peer refcount unchanged across call | `unix_getname` |

### Layer 4: Verus/Creusot functional

Per-`inet_getname(peer=1)`, `inet6_getname(peer=1)`, `unix_getname(peer=1)`, `netlink_getname(peer=1)` semantic equivalence with upstream files. `unix_peer_get` / `sock_put` balance preserved. ENOTCONN reported for TCP_CLOSE/TCP_LISTEN/TCP_NEW_SYN_RECV states matching upstream.

### hardening

- **`storage` zero-initialized** — defense against per-info-leak of prior stack contents.
- **`storage` is the full 128-byte `sockaddr_storage`** — defense against per-AF over-write.
- **`move_addr_to_user` clamps copy_len** — defense against per-OOB-write into caller's buffer.
- **`*addrlen` reports full kernel length** — defense against per-silent-truncation API surprise.
- **`unix_state_lock` + balanced `unix_peer_get`/`sock_put`** — defense against per-race with concurrent close and per-refleak.
- **TCP state-machine check before peer copy** — defense against per-stale-peer-data after RST.
- **LSM `socket_getpeername`** — defense against per-MAC bypass.
- **input_len signed-check pre-getname** — defense against per-overflow when input_len > INT_MAX.
- **NETLINK peer pid sourced from connected `dst_pid`** — defense against per-pid-spoof readback.

### grsecurity-pax

- **PAX_RANDKSTACK** — kstack base randomized; `sockaddr_storage` stack placement varies.
- **PaX UDEREF** — `addr` / `addrlen` derefs only via `copy_from_user` (input_len) and `copy_to_user` (output bytes + namelen).
- **GRKERNSEC_NO_SIMULT_CONNECT** — paired hardening at connect-time means peer-address visible to getpeername is reliably the one chosen by a non-spoofable handshake.
- **GRKERNSEC_BLACKHOLE** — if peer sent RST and blackhole drops it, getpeername may still return last-known peer until state-transition observed; consistent with upstream.
- **GRKERNSEC_RANDNET** — irrelevant on getpeername (no port selection).
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — getpeername does not re-check; gated at socket creation.
- **GRKERNSEC_BLOCK_PEERNAME_ACROSS_NS** — hardened policy: getpeername returns `-ENOTCONN` when the connected peer is in a different user-ns and the caller lacks `CAP_NET_ADMIN` in the peer ns; defeats peer-address discovery via cross-ns socketpair-passing.
- **SOCK_CLOEXEC mandatory under suid** — irrelevant on getpeername.
- **SCM_RIGHTS recursion bound** — irrelevant on getpeername.
- **getsockopt info-leak prevention** — `storage` zeroed; `sin_zero[8]` always zero; sockaddr_un padding zeroed. **This is the central grsec invariant for getpeername** — peer addresses must never leak unrelated stack state.

