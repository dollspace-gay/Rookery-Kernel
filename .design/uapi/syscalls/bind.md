# Tier-5 syscall: bind(2) — bind a name to a socket (syscall 49)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - net/socket.c (__sys_bind, sys_bind, move_addr_to_kernel)
  - net/ipv4/af_inet.c (inet_bind, __inet_bind)
  - net/ipv6/af_inet6.c (inet6_bind)
  - net/unix/af_unix.c (unix_bind)
  - net/netlink/af_netlink.c (netlink_bind)
-->

## Summary

`bind(2)` assigns a local address (the "name") to a socket. For Internet sockets this is `(addr, port)`; for UNIX-domain sockets it is a filesystem path or an abstract-namespace name; for netlink sockets it is a `(pid, groups)` pair; for AF_PACKET it is `(ifindex, protocol)`. It is syscall number **49** on x86-64.

The path is `sys_bind(fd, uaddr, addrlen)` → `__sys_bind()` → `move_addr_to_kernel()` → `sock.ops.bind(sock, &addr, addrlen)`. Per-AF bind enforces port-reuse semantics, capability checks (`CAP_NET_BIND_SERVICE` for ports < 1024), and family-specific naming rules.

Critical for: every server (must `bind` before `listen`), every UNIX-socket service file (`/run/.../socket`), every netlink subscriber.

This Tier-5 covers syscall **49** `bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen)`.

## Signature

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

```rust
pub fn sys_bind(sockfd: i32, addr: UserPtr<Sockaddr>, addrlen: u32) -> SyscallResult<i32>;
```

x86-64 entry: `__x64_sys_bind` → `__sys_bind(fd, uaddr, addrlen)`.

## Parameters

- **`sockfd`** — fd from `socket(2)`.
- **`addr`** — userland pointer to a `struct sockaddr` whose `sa_family` field selects family-specific decode.
- **`addrlen`** — bytes available at `addr`. Must satisfy `sizeof(sa_family_t) ≤ addrlen ≤ sizeof(struct __kernel_sockaddr_storage) == 128`.

## Return

0 on success; `-1`/`-errno` on error.

## Errors

- `EBADF` — sockfd not open.
- `ENOTSOCK` — fd not a socket.
- `EINVAL` — addrlen out of bounds; socket already bound; family mismatch.
- `EFAULT` — addr pointer fault.
- `EAFNOSUPPORT` — `sa_family` doesn't match socket's family.
- `EADDRINUSE` — local address+port already in use and `SO_REUSEADDR`/`SO_REUSEPORT` rules deny.
- `EADDRNOTAVAIL` — requested address not assigned to any local interface.
- `EACCES` / `EPERM` — port < 1024 without `CAP_NET_BIND_SERVICE`; LSM denial; chroot path denial; abstract-ns grsec denial.
- `ELOOP` — UNIX path traversal too deep.
- `ENAMETOOLONG` — UNIX path > 108 bytes.
- `ENOENT` / `ENOTDIR` — UNIX path parent missing / not dir.
- `EROFS` — UNIX bind on read-only fs.
- `ENOMEM` — slab/kmem alloc fail.
- `ENOBUFS` — netlink port table exhausted.

## ABI surface

- `addrlen` is `socklen_t` (`u32`). Negative values (signed reinterpretation) bounce off the bounds check.
- `struct sockaddr_in` is 16 bytes; `struct sockaddr_in6` is 28 bytes; `struct sockaddr_un` up to 110 bytes (108 path + sa_family); `struct sockaddr_ll` 20 bytes; `struct sockaddr_nl` 12 bytes.
- The 128-byte upper bound is `sizeof(struct __kernel_sockaddr_storage)`.

## Compatibility contract

REQ-1: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-2: addr copy:
- if addrlen < sizeof(sa_family_t) ∨ addrlen > 128: return `-EINVAL`.
- err = move_addr_to_kernel(uaddr, addrlen, &storage); if err: return `-EFAULT`.

REQ-3: LSM:
- err = security_socket_bind(sock, &storage, addrlen); if err: return err.

REQ-4: Cgroup BPF:
- err = BPF_CGROUP_RUN_PROG_INET4/6_BIND(sk, &storage); may rewrite or deny.

REQ-5: Per-AF dispatch:
- err = sock.ops.bind(sock, &storage.addr, addrlen).

REQ-6: AF_INET semantics (inet_bind):
- if storage.family != AF_INET: return `-EAFNOSUPPORT`.
- if port < 1024 ∧ port != 0 ∧ !ns_capable(net.user_ns, CAP_NET_BIND_SERVICE):
  - return `-EACCES`.
- if !inet_addr_valid_or_nonlocal(net, inet, addr, chk_addr_ret):
  - return `-EADDRNOTAVAIL`.
- inet.inet_rcv_saddr = inet.inet_saddr = storage.sin_addr.s_addr.
- snum = ntohs(storage.sin_port).
- if !snum: snum = sk.sk_prot.get_port(sk, 0).
- else if sk_prot.get_port(sk, snum): return `-EADDRINUSE`.
- sk.sk_state = TCP_CLOSE.

REQ-7: AF_INET6 semantics (inet6_bind):
- Same with sin6_addr; if storage.sin6_addr is IPv4-mapped, fall through to inet_bind logic on a v4-mapped basis.
- Handle IPV6_V6ONLY flag.

REQ-8: AF_UNIX semantics (unix_bind):
- if abstract (sun_path[0] == '\0'): hash on abstract namespace, no FS.
- else: create directory entry via `kern_path_create` + mknod(S_IFSOCK | 0666 & ~umask).
- if already-bound: return `-EINVAL`.

REQ-9: AF_NETLINK semantics (netlink_bind):
- portid = storage.nl_pid; groups = storage.nl_groups.
- if groups ∧ !ns_capable(net.user_ns, CAP_NET_ADMIN): return `-EPERM`.
- insert into nl_table per protocol.

REQ-10: AF_PACKET semantics (packet_bind):
- requires CAP_NET_RAW already established at socket().

REQ-11: SO_REUSEADDR/SO_REUSEPORT interactions:
- per-`inet_csk_get_port`: same-port allowed when SO_REUSEADDR+TIME_WAIT or SO_REUSEPORT with matching uid.

REQ-12: Audit:
- audit_log_socketcall(SYS_BIND, fd, &storage, addrlen, ret).

## Acceptance Criteria

- [ ] AC-1: bind(SOCK_DGRAM, sockaddr_in{0, 0}) → ephemeral port assigned; getsockname returns it.
- [ ] AC-2: bind to specific port already in use → -EADDRINUSE.
- [ ] AC-3: bind to port 80 unprivileged → -EACCES; with CAP_NET_BIND_SERVICE → success.
- [ ] AC-4: bind to address not on any interface → -EADDRNOTAVAIL.
- [ ] AC-5: bind on already-bound socket → -EINVAL.
- [ ] AC-6: bind UNIX path "/tmp/sock": creates inode; second bind to same path → -EADDRINUSE.
- [ ] AC-7: bind UNIX abstract: no FS entry; second bind to same name → -EADDRINUSE.
- [ ] AC-8: bind addrlen > 128 → -EINVAL.
- [ ] AC-9: bind addrlen < sizeof(sa_family_t) → -EINVAL.
- [ ] AC-10: bind addr pointer faults → -EFAULT.
- [ ] AC-11: bind AF_INET6 with IPV6_V6ONLY set, then AF_INET bind to same port → success.
- [ ] AC-12: BPF cgroup INET_BIND rewrite: kernel uses rewritten address.
- [ ] AC-13: bind on RAW socket without CAP_NET_RAW: -EPERM (caught at socket() but re-validated here for AF_PACKET).

## Architecture

```
SysSocket::sys_bind(fd, uaddr, addrlen) -> SyscallResult<i32>
  1. sock = SocketFd::lookup_light(fd)?;
  2. if !(size_of::<SaFamilyT>() ..= 128).contains(&(addrlen as usize)):
        return Err(EINVAL).
  3. storage = SockaddrStorage::zeroed();
  4. UserCopy::from_user(&mut storage.bytes[..addrlen as usize], uaddr)?;
  5. Lsm::socket_bind(&sock, &storage, addrlen)?;
  6. CgroupBpf::inet_bind(&sock, &mut storage)?;
  7. err = sock.ops.bind(&sock, &storage, addrlen);
  8. Audit::log_socketcall(SYS_BIND, fd, &storage, addrlen, err);
  9. err.into()
```

Per-`inet_bind` (AF_INET):
1. if storage.sin_family != AF_INET: return -EAFNOSUPPORT.
2. chk_addr_ret = inet_addr_type_table(net, addr, sk.sk_bound_dev_if).
3. if !sysctl_ip_nonlocal_bind ∧ chk_addr_ret ∉ {LOCAL, MULTICAST, BROADCAST, ANYCAST, RTN_BROADCAST}:
   - return -EADDRNOTAVAIL.
4. snum = ntohs(addr.sin_port).
5. if snum ∧ snum < PROT_SOCK ∧ !ns_capable(net.user_ns, CAP_NET_BIND_SERVICE):
   - return -EACCES.
6. lock_sock(sk).
7. if sk.sk_state != TCP_CLOSE ∨ inet.inet_num: return -EINVAL.
8. inet.inet_rcv_saddr = inet.inet_saddr = addr.sin_addr.s_addr.
9. if sk.sk_prot.get_port(sk, snum):
   - inet.inet_saddr = inet.inet_rcv_saddr = 0.
   - return -EADDRINUSE.
10. if inet.inet_rcv_saddr: sk.sk_userlocks |= SOCK_BINDADDR_LOCK.
11. if snum: sk.sk_userlocks |= SOCK_BINDPORT_LOCK.
12. inet.inet_sport = htons(inet.inet_num).
13. release_sock(sk).
14. return 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bind_addrlen_bounds` | INVARIANT | per-sys_bind: addrlen ∈ [2, 128] else EINVAL. |
| `bind_storage_zeroed` | INVARIANT | per-sys_bind: storage zeroed before copy_from_user. |
| `bind_family_match` | INVARIANT | per-inet_bind: storage.sin_family == AF_INET else EAFNOSUPPORT. |
| `bind_low_port_cap` | INVARIANT | per-inet_bind: port < 1024 ⟹ CAP_NET_BIND_SERVICE. |
| `bind_no_double_bind` | INVARIANT | per-inet_bind: already-bound ⟹ EINVAL. |

### Layer 2: TLA+

`uapi/syscalls/bind.tla`:
- States: validate, copy, lsm, bpf, dispatch, port-alloc, return.
- Properties:
  - `safety_port_uniqueness` — two binds to same (family, addr, port) without SO_REUSEPORT-compat ⟹ second returns EADDRINUSE.
  - `safety_low_port_cap` — port < 1024 without CAP_NET_BIND_SERVICE ⟹ EACCES.
  - `safety_no_partial_bind_on_error` — failure leaves sk.inet_num == 0.
  - `liveness_terminates` — bind returns within bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: addrlen ≥ sizeof(sa_family_t) ∧ ≤ 128 | `sys_bind` |
| post: ret == 0 ⟹ sk.sk_userlocks & (SOCK_BINDADDR_LOCK | SOCK_BINDPORT_LOCK) set per request | `inet_bind` |
| post: ret < 0 ⟹ sk.inet_rcv_saddr unchanged | `inet_bind` |
| post: UNIX-path bind success ⟹ FS entry of type S_IFSOCK exists | `unix_bind` |

### Layer 4: Verus/Creusot functional

Per-`inet_bind`, `inet6_bind`, `unix_bind` semantic equivalence with upstream files. Hash-table insertion order in port hash (`inet_bind_bucket`) matches upstream; SO_REUSEPORT bucket logic (`inet_csk_get_port` with `bind_conflict`) preserved bit-for-bit.

## Hardening

- **addrlen strict-bounded to `sizeof(struct sockaddr_storage)`** — defense against per-OOB-read of stack storage.
- **storage zero-initialized** — defense against per-info-leak of prior stack contents into per-AF bind handler.
- **`move_addr_to_kernel` checks family pre-dispatch** — defense against per-confused-family bind.
- **`CAP_NET_BIND_SERVICE` for ports < 1024** — defense against per-privileged-port hijack.
- **`ip_nonlocal_bind` sysctl gate** — defense against per-nonlocal-bind footgun (default off).
- **LSM `socket_bind`** — defense against per-MAC bypass.
- **cgroup-BPF `INET_BIND` hook** — defense against per-egress/ingress policy bypass.
- **Per-`inet_bind_bucket` lock held during get_port** — defense against per-race-bind-conflict.
- **UNIX abstract bind hashed under namespace** — defense against per-cross-ns name collision.
- **UNIX path bind uses `kern_path_create`** — defense against per-symlink-traversal during bind.

## Grsecurity-PaX

- **PAX_RANDKSTACK** — kstack base randomized per-syscall; `sockaddr_storage` placement varies between calls.
- **PaX UDEREF** — userland `uaddr` reachable only via `copy_from_user`; direct deref of `uaddr` outside that path traps.
- **GRKERNSEC_NO_SIMULT_CONNECT** — not directly relevant to bind, but bind followed by simultaneous-connect storm is throttled at the connect side.
- **GRKERNSEC_BLACKHOLE** — bound sockets that subsequently receive scan attempts to unbound ports on the same address are silently dropped at netfilter.
- **GRKERNSEC_RANDNET** — ephemeral port selection during `bind(... port=0)` uses extra randomness (jitter on top of `ip_local_port_range` selection) defeating predictable port allocation.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — netlink bind with non-zero `nl_groups` requires CAP_NET_ADMIN in the initial user-ns (no user-ns shortcut under grsec).
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — abstract-ns UNIX bind requires `grsec.harden_ipcs` policy match (creator-only acl); pathname-UNIX bind validates parent-directory ACL strictly. Chrooted processes are forbidden from binding abstract-ns socks (config `grsec.chroot_deny_abstract`).
- **SOCK_CLOEXEC mandatory under suid** — bind is not fd-creating, but suid-tainted tasks that bind to privileged ports are audited.
- **SCM_RIGHTS recursion bound** — irrelevant on bind.
- **getsockopt info-leak prevention** — `SO_BINDTODEVICE` readback is bounded; ephemeral port assignment never reveals foreign sockets' selection state.

## Open Questions

- Should `ip_nonlocal_bind` be replaced by a per-netns sysctl + per-cgroup BPF rather than a global toggle? (Upstream: still global.)
- For abstract-namespace UNIX, do we expose grsec ACL via getsockopt to allow daemons to introspect? Upstream grsec: no.

## Out of Scope

- `listen(2)` — separate Tier-5.
- `setsockopt(SO_REUSEPORT)` group hashing — Tier-3 `net/core/sock_reuseport.md`.
- TCP port-hash table — Tier-3 `net/ipv4/inet_hashtables.md`.
- Implementation code.
