# Tier-5 syscall: getsockname(2) — return the local address of a socket (syscall 51)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - net/socket.c (__sys_getsockname, sys_getsockname, move_addr_to_user)
  - net/ipv4/af_inet.c (inet_getname)
  - net/ipv6/af_inet6.c (inet6_getname)
  - net/unix/af_unix.c (unix_getname)
  - net/netlink/af_netlink.c (netlink_getname)
  - net/packet/af_packet.c (packet_getname)
-->

## Summary

`getsockname(2)` returns the local address bound to a socket. For Internet sockets it is `(addr, port)`; for UNIX-domain sockets it is the bound path (or abstract name, or an empty `sun_path` for unbound autobinds); for netlink sockets it is `(nl_pid, nl_groups)`; for AF_PACKET it is `(ifindex, protocol, hatype, pkttype, halen, addr)`. It is syscall number **51** on x86-64.

The path is `sys_getsockname(fd, uaddr, uaddr_len)` → `__sys_getsockname()` → `sock.ops.getname(sock, &storage, /*peer=*/0)` → `move_addr_to_user(&storage, kernel_namelen, uaddr, uaddr_len)`. The kernel writes both the address bytes and the per-AF `namelen` value back to user.

Critical for: every server that calls `bind(... port=0)` and needs to discover the assigned ephemeral port; every UNIX-socket auto-bind autoselect inspector; every test harness; netstat-like introspection.

This Tier-5 covers syscall **51** `getsockname(int sockfd, struct sockaddr *addr, socklen_t *addrlen)`.

## Signature

```c
int getsockname(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

```rust
pub fn sys_getsockname(sockfd: i32, addr: UserPtr<Sockaddr>, addrlen: UserPtr<u32>)
    -> SyscallResult<i32>;
```

x86-64 entry: `__x64_sys_getsockname` → `__sys_getsockname(fd, uaddr, uaddr_len)`.

## Parameters

- **`sockfd`** — fd from `socket(2)`.
- **`addr`** — userland sockaddr buffer; kernel writes up to `min(kernel_namelen, *addrlen)` bytes.
- **`addrlen`** — IN/OUT: on entry, capacity of `addr` buffer; on exit, the kernel's full address length (may be larger than capacity — truncation indicator).

## Return

0 on success; `-1`/`-errno` on error.

## Errors

- `EBADF` — sockfd not open.
- `ENOTSOCK` — fd not a socket.
- `EFAULT` — addr or addrlen pointer fault.
- `EINVAL` — `*addrlen` is negative on entry.
- `ENOBUFS` — slab/kmem alloc fail copying name (rare).
- `EOPNOTSUPP` — family does not implement `getname` (should never happen for real sockets).

## ABI surface

- `*addrlen` is `socklen_t` (`u32`). The kernel writes back the **actual** kernel-side namelen, even if it exceeds the caller's input capacity (this is how callers discover they need a bigger buffer). The address bytes copied are limited to `min(kernel_namelen, input_addrlen)`.
- `struct sockaddr_in` produced: 16 bytes.
- `struct sockaddr_in6` produced: 28 bytes (sin6_family/sin6_port/sin6_flowinfo/sin6_addr/sin6_scope_id).
- `struct sockaddr_un` produced: `offsetof(sun_path) + path_strlen + 1` for pathname; `offsetof(sun_path) + 1 + abstract_len` for abstract; `sizeof(sa_family_t)` for unnamed.
- `struct sockaddr_nl` produced: 12 bytes (nl_family/nl_pad/nl_pid/nl_groups).
- `struct sockaddr_ll` produced: 20 bytes (sll_family/sll_protocol/sll_ifindex/sll_hatype/sll_pkttype/sll_halen/sll_addr[8]).

## Compatibility contract

REQ-1: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-2: addrlen read-in:
- err = get_user(input_len, uaddr_len); if err: return `-EFAULT`.
- if (int)input_len < 0: return `-EINVAL`.

REQ-3: storage zero:
- storage = struct sockaddr_storage zeroed (128 bytes).
- kernel_namelen = 0.

REQ-4: LSM:
- err = security_socket_getsockname(sock); if err: return err.

REQ-5: Per-AF dispatch:
- err = sock.ops.getname(sock, &storage.addr, /*peer=*/0).
- on success, sock.ops.getname returns kernel_namelen via storage write + return value (kernel_namelen on success in some versions, or via `int *namelen` out-param historically). In our impl, getname returns `Result<usize, Errno>`.

REQ-6: Move to user:
- err = move_addr_to_user(&storage, kernel_namelen, uaddr, uaddr_len).
- copy min(kernel_namelen, input_len) bytes of address.
- write kernel_namelen back to *uaddr_len.

REQ-7: Audit:
- audit_log_socketcall(SYS_GETSOCKNAME, fd, &storage, kernel_namelen, ret).

## Acceptance Criteria

- [ ] AC-1: bind(SOCK_DGRAM, sockaddr_in{0, 0}); getsockname returns assigned ephemeral port; *addrlen == 16.
- [ ] AC-2: getsockname on unbound TCP socket: returns sin_family=AF_INET, sin_port=0, sin_addr=INADDR_ANY; *addrlen == 16.
- [ ] AC-3: getsockname with input_len = 8 (truncated): copies 8 bytes; *addrlen == 16 (full length).
- [ ] AC-4: getsockname with input_len = 0: copies 0 bytes; *addrlen == 16.
- [ ] AC-5: getsockname on UNIX bound to "/tmp/sock": returns path; *addrlen == offsetof(sun_path) + strlen("/tmp/sock") + 1.
- [ ] AC-6: getsockname on UNIX abstract "\\0name": returns abstract name; sun_path[0] == 0; *addrlen == offsetof(sun_path) + 1 + abstract_len.
- [ ] AC-7: getsockname on UNIX unbound: sun_family=AF_UNIX, no path; *addrlen == sizeof(sa_family_t).
- [ ] AC-8: getsockname on closed fd: -EBADF.
- [ ] AC-9: getsockname on pipe fd: -ENOTSOCK.
- [ ] AC-10: getsockname with addr=NULL but addrlen=NULL: -EFAULT.
- [ ] AC-11: getsockname with *addrlen = (u32)-1 (= INT_MIN as signed): -EINVAL.
- [ ] AC-12: getsockname on AF_PACKET: returns sll_ifindex == bound interface.

## Architecture

```
SysSocket::sys_getsockname(fd, uaddr, uaddr_len) -> SyscallResult<i32>
  1. sock = SocketFd::lookup_light(fd)?;
  2. let input_len: u32 = UserCopy::get_user(uaddr_len)?;
  3. if (input_len as i32) < 0 { return Err(EINVAL); }
  4. let mut storage = SockaddrStorage::zeroed();
  5. Lsm::socket_getsockname(&sock)?;
  6. let kernel_namelen = sock.ops.getname(&sock, &mut storage, /*peer=*/false)?;
  7. UserCopy::move_addr_to_user(&storage, kernel_namelen, uaddr, uaddr_len)?;
  8. Audit::log_socketcall(SYS_GETSOCKNAME, fd, &storage, kernel_namelen, 0);
  9. Ok(0)
```

`move_addr_to_user(storage, kernel_namelen, uaddr, uaddr_len)`:
1. let input_len: u32 = UserCopy::get_user(uaddr_len)?.
2. let copy_len = core::cmp::min(kernel_namelen, input_len as usize).
3. UserCopy::to_user(uaddr, &storage.as_bytes()[..copy_len])?.
4. UserCopy::put_user(kernel_namelen as u32, uaddr_len)?.

`inet_getname(sock, storage, peer=0)` (AF_INET):
1. let inet = sock.sk.inet_sock().
2. storage.sin_family = AF_INET.
3. storage.sin_port = inet.inet_sport.
4. storage.sin_addr = inet.inet_rcv_saddr.
5. memset(storage.sin_zero, 0, 8).
6. return Ok(sizeof(struct sockaddr_in)).  // 16

`inet6_getname(sock, storage, peer=0)` (AF_INET6):
1. storage.sin6_family = AF_INET6.
2. storage.sin6_port = sk.sport.
3. storage.sin6_flowinfo = sk6.flowinfo.
4. storage.sin6_addr = sk6.rcv_saddr.
5. storage.sin6_scope_id = sk6.scope_id (when address is link-local).
6. return Ok(sizeof(struct sockaddr_in6)).  // 28

`unix_getname(sock, storage, peer=0)` (AF_UNIX):
1. unix_state_lock(sk).
2. let addr = sk.unix_sock().addr.
3. storage.sun_family = AF_UNIX.
4. if addr.is_none(): return Ok(sizeof(sa_family_t)).
5. let path_or_abstract = addr.unwrap().name.
6. memcpy(storage.sun_path, path_or_abstract, addr.len).
7. unix_state_unlock(sk).
8. return Ok(offsetof(sun_path) + addr.len).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `getsockname_addrlen_signed_check` | INVARIANT | per-sys_getsockname: input_len reinterpreted as i32 is non-negative else EINVAL. |
| `getsockname_storage_zeroed` | INVARIANT | per-sys_getsockname: storage zeroed before getname dispatch. |
| `getsockname_truncation_indicated` | INVARIANT | per-sys_getsockname: *addrlen on return == kernel_namelen (full length, not copy length). |
| `getsockname_copy_bounded` | INVARIANT | per-move_addr_to_user: bytes copied ≤ min(kernel_namelen, input_len). |
| `getsockname_no_info_leak` | INVARIANT | per-sys_getsockname: storage bytes beyond kernel_namelen never written to user. |

### Layer 2: TLA+

`uapi/syscalls/getsockname.tla`:
- States: validate, lsm, dispatch-getname, copy-bytes, copy-namelen, return.
- Properties:
  - `safety_namelen_reported_full` — *addrlen on return == kernel-side namelen, even if input_len < kernel_namelen.
  - `safety_address_bytes_bounded` — bytes copied to user ≤ min(kernel_namelen, input_len).
  - `safety_no_stack_leak` — uninitialized storage bytes never observed in user output.
  - `liveness_terminates` — getsockname returns within bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: input_len reinterpreted as i32 ≥ 0 | `sys_getsockname` |
| pre: storage zeroed | `sys_getsockname` |
| post: *addrlen out = kernel_namelen | `move_addr_to_user` |
| post: bytes copied ≤ min(kernel_namelen, input_len) | `move_addr_to_user` |
| post: copy_len ≤ sizeof(sockaddr_storage) (128) | `move_addr_to_user` |

### Layer 4: Verus/Creusot functional

Per-`inet_getname`, `inet6_getname`, `unix_getname`, `netlink_getname`, `packet_getname` semantic equivalence with upstream files. namelen return values exactly match `sizeof(struct sockaddr_in/in6/un/nl/ll)` (or `offsetof + path_strlen + 1` for UNIX pathname).

## Hardening

- **`storage` zero-initialized** — defense against per-info-leak of prior stack contents to user.
- **`storage` is the full 128-byte `sockaddr_storage`** — defense against per-AF over-write into a smaller stack object.
- **`move_addr_to_user` clamps copy_len** — defense against per-OOB-write into caller's buffer when `kernel_namelen > input_len`.
- **`*addrlen` reports full kernel length** — defense against per-silent-truncation API surprise (caller can detect and resize).
- **Per-AF `getname` holds `unix_state_lock` / inet `lock_sock` as appropriate** — defense against per-race with concurrent bind/connect.
- **LSM `socket_getsockname`** — defense against per-MAC bypass.
- **No fd refcount manipulation beyond `sockfd_lookup_light`** — defense against per-leak on error path.
- **input_len signed-check pre-getname** — defense against per-overflow when input_len > INT_MAX.
- **AF_PACKET getname stops at `halen ≤ 8`** — defense against per-OOB-write of HW address.
- **NETLINK getname pid sourced from `sk.portid`, not user-controlled** — defense against per-pid-spoof readback.

## Grsecurity-PaX

- **PAX_RANDKSTACK** — kstack base randomized per-syscall; `sockaddr_storage` stack placement varies between calls.
- **PaX UDEREF** — `addr` and `addrlen` userland derefs only via `copy_from_user` (for input_len) and `copy_to_user` (for output bytes + output namelen). Direct kernel deref of `uaddr` traps.
- **GRKERNSEC_NO_SIMULT_CONNECT** — irrelevant on getsockname (no connect attempt).
- **GRKERNSEC_BLACKHOLE** — irrelevant on getsockname (no packet emitted).
- **GRKERNSEC_RANDNET** — when paired with `bind(... port=0)`, the ephemeral port returned by getsockname reflects grsec-randomized selection; the readback path itself adds no extra entropy.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — getsockname does not re-check; capability was gated at socket creation.
- **GRKERNSEC_BLOCK_SAME_PROCESS_GETNAME** — hardened policy forbids querying sockname across PID-namespaces for tasks not in the same userns peer-group; returns `-EPERM`.
- **SOCK_CLOEXEC mandatory under suid** — irrelevant on getsockname.
- **SCM_RIGHTS recursion bound** — irrelevant on getsockname.
- **getsockopt info-leak prevention** — `storage` is zeroed before per-AF write; `sin_zero[8]` always zero; sockaddr_un padding zeroed; netlink `nl_pad` zeroed. Stack residue can never appear in returned address. **This is the central grsec invariant for getsockname.**

## Open Questions

- For UNIX autobind (sockets that received a kernel-assigned abstract name via `bind(NULL, 0)`-equivalent autoselect): expose the kernel-assigned name via getsockname? Upstream: yes; we follow.
- For BPF-cgroup-rewritten bind addresses: does getsockname return the rewritten or the user-requested address? Upstream: rewritten; we follow.

## Out of Scope

- `getpeername(2)` — separate Tier-5 (`peer=1` variant of same dispatch).
- `bind(2)` — separate Tier-5.
- Per-AF `sock.ops.getname` implementations — Tier-3 per family.
- Implementation code.
