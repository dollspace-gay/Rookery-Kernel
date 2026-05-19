---
title: "Tier-5 syscall: setsockopt(2) — set socket option (syscall 54)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`setsockopt(2)` writes a socket option at a given protocol level. The `(level, optname)` tuple dispatches to per-level handlers: `SOL_SOCKET` → `sock_setsockopt`, `IPPROTO_IP` → `do_ip_setsockopt`, `IPPROTO_TCP` → `do_tcp_setsockopt`, `IPPROTO_IPV6` → `do_ipv6_setsockopt`, `IPPROTO_ICMPV6` → `do_icmpv6_setsockopt`, plus per-AF custom levels (`SOL_UDP`, `SOL_NETLINK`, `SOL_PACKET`, `SOL_XDP`). It is syscall number **54** on x86-64.

The path is `sys_setsockopt(fd, level, optname, optval, optlen)` → `__sys_setsockopt` → `sock.ops.setsockopt(sock, level, optname, optval, optlen)` (or the level-specific shortcut for SOL_SOCKET). Each handler validates `optlen`, copies in `optval`, and applies the option.

Critical for: every server (`SO_REUSEADDR`, `SO_REUSEPORT`, `TCP_NODELAY`, `SO_KEEPALIVE`), every container's network shaping (`SO_MARK`, `SO_BINDTODEVICE`), every TLS handshake (`SO_RCVTIMEO`, `TCP_FASTOPEN`).

This Tier-5 covers syscall **54** `setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen)`.

### Acceptance Criteria

- [ ] AC-1: setsockopt(SOL_SOCKET, SO_REUSEADDR, &one, 4): subsequent bind to TIME_WAIT addr succeeds.
- [ ] AC-2: setsockopt(SOL_SOCKET, SO_PRIORITY, &7, 4) unprivileged: -EPERM.
- [ ] AC-3: setsockopt(SOL_SOCKET, SO_MARK, ...) unprivileged: -EPERM.
- [ ] AC-4: setsockopt(SOL_SOCKET, SO_RCVBUF, &huge, 4): clamped to 2 * sysctl_rmem_max.
- [ ] AC-5: setsockopt(SOL_SOCKET, SO_RCVBUFFORCE, &huge, 4) with CAP_NET_ADMIN: not clamped.
- [ ] AC-6: setsockopt(IPPROTO_TCP, TCP_NODELAY, &one, 4): TCP segments not coalesced.
- [ ] AC-7: setsockopt(IPPROTO_IPV6, IPV6_V6ONLY, &one, 4) after bind: -EINVAL (must be set pre-bind).
- [ ] AC-8: setsockopt(SOL_SOCKET, SO_DOMAIN, ...) on set: -ENOPROTOOPT.
- [ ] AC-9: setsockopt with unknown optname at known level: -ENOPROTOOPT.
- [ ] AC-10: setsockopt with optlen=0 where int expected: -EINVAL.
- [ ] AC-11: BPF cgroup setsockopt denial: -EPERM.
- [ ] AC-12: LSM denial: -EACCES.
- [ ] AC-13: optval fault: -EFAULT.

### Architecture

```
SysSocket::sys_setsockopt(fd, level, optname, optval, optlen) -> SyscallResult<i32>
  1. sock = SocketFd::lookup_light(fd)?;
  2. let mut ctx = BpfSetsockoptCtx::new(level, optname, optval, optlen);
  3. CgroupBpf::setsockopt(&sock, &mut ctx)?;          // may rewrite or deny -EPERM
  4. let (level, optname, optval, optlen) = ctx.finalize();
  5. Lsm::socket_setsockopt(&sock, level, optname)?;
  6. let err = if level == SOL_SOCKET {
        Sock::setsockopt(&sock, level, optname, optval, optlen)
     } else {
        let op = sock.ops.setsockopt.ok_or(EOPNOTSUPP)?;
        op(&sock, level, optname, optval, optlen)
     };
  7. Audit::log_socketcall(SYS_SETSOCKOPT, fd, None, 0, err, level, optname);
  8. err.into()
```

Per-level handler `Sock::setsockopt(sock, level, optname, optval, optlen)`:
1. lock_sock(sk).
2. switch optname:
   - SO_REUSEADDR: copy_int(); sk.sk_reuse = val ? SK_CAN_REUSE : SK_NO_REUSE.
   - SO_KEEPALIVE: copy_int(); sock_valbool_flag(sk, SOCK_KEEPOPEN, val).
   - SO_LINGER: if optlen < sizeof(linger): EINVAL; copy_struct; sk.sk_lingertime = ling.l_linger * HZ.
   - SO_RCVBUF: copy_int(); if !cap_net_admin: clamp ≤ sysctl_rmem_max; sk.sk_rcvbuf = max(2*val, SOCK_MIN_RCVBUF).
   - SO_RCVBUFFORCE: require CAP_NET_ADMIN; no clamp.
   - SO_PRIORITY: if val > 6 ∧ !cap_net_admin: EPERM; sk.sk_priority = val.
   - SO_MARK: require CAP_NET_ADMIN; sk.sk_mark = val.
   - SO_BINDTODEVICE: require CAP_NET_RAW/CAP_NET_ADMIN; ifindex lookup; sk.sk_bound_dev_if = ifindex.
   - SO_ATTACH_FILTER: require !unprivileged_bpf_disabled or CAP_NET_ADMIN; sk_attach_filter.
   - SO_DOMAIN / _TYPE / _PROTOCOL: return ENOPROTOOPT.
   - default: ENOPROTOOPT.
3. release_sock(sk).

### Out of Scope

- `getsockopt(2)` — separate Tier-5.
- Per-option implementation details — Tier-3 per family (e.g. `net/ipv4/tcp.md` for TCP options).
- BPF cgroup setsockopt program type — Tier-3 `kernel/bpf/cgroup.md`.
- Implementation code.

### signature

```c
int setsockopt(int sockfd, int level, int optname,
               const void *optval, socklen_t optlen);
```

```rust
pub fn sys_setsockopt(
    sockfd: i32, level: i32, optname: i32,
    optval: UserPtr<u8>, optlen: u32,
) -> SyscallResult<i32>;
```

x86-64 entry: `__x64_sys_setsockopt` → `__sys_setsockopt(fd, level, optname, optval, optlen)`.

### parameters

- **`sockfd`** — socket fd.
- **`level`** — protocol level: `SOL_SOCKET` (1), `IPPROTO_IP` (0), `IPPROTO_TCP` (6), `IPPROTO_UDP` (17), `IPPROTO_IPV6` (41), `IPPROTO_ICMPV6` (58), `SOL_NETLINK` (270), `SOL_PACKET` (263), `SOL_XDP` (283), and many others.
- **`optname`** — option number within the level (`SO_REUSEADDR`, `SO_KEEPALIVE`, `TCP_NODELAY`, `IP_TTL`, `IPV6_HOPLIMIT`, …).
- **`optval`** — userland pointer to option value (typically an `int`, sometimes a struct).
- **`optlen`** — size of `*optval` in bytes.

### return

0 on success; `-1`/`-errno` on error.

### errors

- `EBADF` — sockfd not open.
- `ENOTSOCK` — fd not a socket.
- `EFAULT` — optval fault.
- `EINVAL` — optname unknown for level; optlen wrong size; value out of range.
- `ENOPROTOOPT` — option not supported at this level.
- `ENOTCONN` — option requires connected sock.
- `EPERM` / `EACCES` — option requires capability (CAP_NET_ADMIN for `SO_PRIORITY > 6`, `SO_RCVBUFFORCE`, `SO_SNDBUFFORCE`, `SO_BINDTODEVICE`, etc.); LSM denial.
- `EISCONN` — option cannot be modified post-connect.
- `EBUSY` — option being concurrently mutated.
- `ENOMEM` — slab alloc fail.
- `EAGAIN` — transient unavailability.
- `EDOM` — value outside protocol-defined domain.

### abi surface

- `optlen` is `socklen_t` (`u32`). Most options accept an `int` (4 bytes); some accept structs (`struct linger` 8 bytes, `struct ucred` 12 bytes, `struct sock_fprog` 16 bytes, `struct timeval` 16 bytes, `struct sock_filter[]` variable).
- `level == SOL_SOCKET` handlers historically duplicated across `sock_setsockopt`; modern path is unified through `do_sock_setsockopt`.
- BPF cgroup hook `BPF_CGROUP_RUN_PROG_SETSOCKOPT` runs before the in-kernel handler and may rewrite or deny.

### Common SOL_SOCKET options

| Option | Type | Notes |
|---|---|---|
| `SO_REUSEADDR` | int (bool) | reuse local addr |
| `SO_REUSEPORT` | int (bool) | reuse port across processes |
| `SO_KEEPALIVE` | int (bool) | enable TCP keepalive |
| `SO_LINGER` | struct linger | close-time linger |
| `SO_RCVBUF` / `SO_SNDBUF` | int | buffer cap |
| `SO_RCVBUFFORCE` / `SO_SNDBUFFORCE` | int | CAP_NET_ADMIN bypass |
| `SO_BROADCAST` | int (bool) | permit broadcast send |
| `SO_BINDTODEVICE` | char[IFNAMSIZ] | bind to interface |
| `SO_PRIORITY` | int (0..6, >6 CAP_NET_ADMIN) | skb priority |
| `SO_MARK` | int (CAP_NET_ADMIN) | netfilter mark |
| `SO_TIMESTAMP_OLD` / `_NEW` | int (bool) | RX timestamping |
| `SO_TIMESTAMPNS` | int (bool) | RX ns-resolution |
| `SO_RCVTIMEO_OLD` / `_NEW` | struct timeval | recv timeout |
| `SO_SNDTIMEO_OLD` / `_NEW` | struct timeval | send timeout |
| `SO_ATTACH_FILTER` / `SO_DETACH_FILTER` | struct sock_fprog | cBPF filter |
| `SO_ATTACH_BPF` | int (bpf fd) | eBPF filter |
| `SO_PEEK_OFF` | int | recv MSG_PEEK offset |
| `SO_PASSCRED` / `SO_PEERCRED` | int (bool) | UNIX cred passing |
| `SO_PASSSEC` / `SO_PEERSEC` | int (bool) | UNIX SELinux ctx |
| `SO_DOMAIN` / `SO_TYPE` / `SO_PROTOCOL` | int (read-only on get) | -ENOPROTOOPT on set |

### compatibility contract

REQ-1: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-2: BPF cgroup hook (pre):
- err = BPF_CGROUP_RUN_PROG_SETSOCKOPT(sk, &level, &optname, optval, &optlen, &kctx).
- if err == -EPERM: return `-EPERM`.
- bpf may rewrite level/optname/optval/optlen.

REQ-3: Level dispatch:
- if level == SOL_SOCKET:
  - err = sock_setsockopt(sock, level, optname, optval, optlen).
- else:
  - if !sock.ops.setsockopt: return `-EOPNOTSUPP`.
  - err = sock.ops.setsockopt(sock, level, optname, optval, optlen).

REQ-4: SOL_SOCKET handler (sock_setsockopt):
- if optlen < expected for optname: return `-EINVAL`.
- copy_from_user(&val, optval, sizeof(val)).
- switch (optname):
  - SO_REUSEADDR: sk.sk_reuse = val ? SK_CAN_REUSE : SK_NO_REUSE.
  - SO_KEEPALIVE: sock_valbool_flag(sk, SOCK_KEEPOPEN, val).
  - SO_LINGER: copy struct linger; set sk.sk_lingertime.
  - SO_RCVBUF: clamp val ≤ sysctl_rmem_max; sk.sk_rcvbuf = max(2 * val, SOCK_MIN_RCVBUF).
  - SO_RCVBUFFORCE: CAP_NET_ADMIN required; bypass sysctl_rmem_max.
  - SO_PRIORITY: val > 6 ⟹ CAP_NET_ADMIN.
  - SO_MARK: CAP_NET_ADMIN.
  - SO_BINDTODEVICE: CAP_NET_RAW or CAP_NET_ADMIN; lookup ifindex.
  - SO_ATTACH_FILTER: install cBPF; CAP_NET_ADMIN for unprivileged_bpf_disabled.
  - SO_ATTACH_BPF: install eBPF via prog fd.
  - SO_DOMAIN / SO_TYPE / SO_PROTOCOL: `-ENOPROTOOPT` on set (read-only).

REQ-5: IPPROTO_IP handler (do_ip_setsockopt):
- IP_TTL, IP_TOS, IP_HDRINCL (CAP_NET_RAW), IP_MULTICAST_TTL, IP_ADD_MEMBERSHIP, IP_PKTINFO, IP_FREEBIND (CAP_NET_ADMIN), IP_TRANSPARENT (CAP_NET_ADMIN), IP_BIND_ADDRESS_NO_PORT.

REQ-6: IPPROTO_TCP handler (do_tcp_setsockopt):
- TCP_NODELAY, TCP_CORK, TCP_KEEPIDLE, TCP_KEEPINTVL, TCP_KEEPCNT, TCP_SYNCNT, TCP_LINGER2, TCP_DEFER_ACCEPT, TCP_QUICKACK, TCP_CONGESTION (string optname; lookup registered module), TCP_FASTOPEN (CAP_NET_BIND_SERVICE), TCP_FASTOPEN_CONNECT, TCP_INQ, TCP_TX_DELAY, TCP_MD5SIG (CAP_NET_ADMIN).

REQ-7: IPPROTO_IPV6 handler (do_ipv6_setsockopt):
- IPV6_V6ONLY, IPV6_UNICAST_HOPS, IPV6_MULTICAST_HOPS, IPV6_PKTINFO, IPV6_HOPLIMIT, IPV6_ADDRFORM, IPV6_JOIN_GROUP, IPV6_TCLASS, IPV6_RECVPKTINFO, IPV6_TRANSPARENT (CAP_NET_ADMIN), IPV6_FREEBIND (CAP_NET_ADMIN).

REQ-8: IPPROTO_ICMPV6 handler (do_icmpv6_setsockopt):
- ICMPV6_FILTER: copy struct icmp6_filter; validate optlen == 32.

REQ-9: LSM:
- err = security_socket_setsockopt(sock, level, optname); if err: return err.

REQ-10: optlen validation per option:
- Every option enforces its expected size. Reading beyond optlen is forbidden.

REQ-11: Audit:
- audit_log_socketcall(SYS_SETSOCKOPT, fd, NULL, 0, ret, level, optname).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `setsockopt_optlen_validated` | INVARIANT | per-handler: copy_from_user bytes ≤ optlen. |
| `setsockopt_cap_gated` | INVARIANT | per-CAP-protected option: capable() checked before mutation. |
| `setsockopt_lock_sock_held` | INVARIANT | per-handler: lock_sock held during state mutation. |
| `setsockopt_so_domain_readonly` | INVARIANT | per-SO_DOMAIN/_TYPE/_PROTOCOL set: returns ENOPROTOOPT. |
| `setsockopt_bpf_pre_hook_runs` | INVARIANT | per-syscall: BPF cgroup hook invoked before kernel handler. |

### Layer 2: TLA+

`uapi/syscalls/setsockopt.tla`:
- States: validate, bpf, lsm, dispatch, per-handler, return.
- Properties:
  - `safety_cap_gates_enforced` — every CAP-protected option ⟹ check before mutation.
  - `safety_bpf_pre_lsm_pre_kernel` — ordering BPF → LSM → kernel handler.
  - `safety_no_partial_mutation` — handler error ⟹ no sk state mutation.
  - `liveness_terminates` — call returns within bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: optlen ≥ expected for optname | `Sock::setsockopt` |
| post: err ⟹ sk unchanged | `Sock::setsockopt` |
| post: SO_RCVBUF success ⟹ sk_rcvbuf ≤ 2 * sysctl_rmem_max (unless FORCE) | `Sock::setsockopt` |
| post: SO_MARK success ⟹ CAP_NET_ADMIN held | `Sock::setsockopt` |
| post: BPF hook may rewrite ⟹ rewritten values fed to kernel | `sys_setsockopt` |

### Layer 4: Verus/Creusot functional

Per-`__sys_setsockopt → sock_setsockopt / sock.ops.setsockopt` semantic equivalence with `net/socket.c` and `net/core/sock.c`. Per-option behavior matches Documentation/networking/.

### hardening

- **Per-option optlen strict-checked** — defense against per-OOB-read of user buffer.
- **CAP_NET_ADMIN gate on RCVBUFFORCE/SNDBUFFORCE/MARK/PRIORITY>6/TRANSPARENT/FREEBIND** — defense against per-unprivileged-bypass.
- **CAP_NET_RAW/CAP_NET_ADMIN gate on BINDTODEVICE** — defense against per-unprivileged-interface-bind.
- **`unprivileged_bpf_disabled` for SO_ATTACH_FILTER/SO_ATTACH_BPF** — defense against per-cBPF-JIT-spray.
- **LSM `socket_setsockopt`** — defense against per-MAC bypass.
- **cgroup-BPF setsockopt hook** — defense against per-policy bypass; can deny or rewrite.
- **`SO_DOMAIN/_TYPE/_PROTOCOL` rejected on set** — defense against per-identity-spoof.
- **`SO_RCVBUF` clamped to `sysctl_rmem_max`** — defense against per-buffer-bloat OOM.
- **`SO_LINGER` linger time bounded** — defense against per-DoS-via-huge-linger.
- **`SO_ATTACH_FILTER` `sock_fprog` size bounded to BPF_MAXINSNS** — defense against per-filter-explosion.

### grsecurity-pax

- **PAX_RANDKSTACK** — per-syscall kstack randomization; on-stack option scratch displaced.
- **PaX UDEREF** — every userland deref of `optval` goes through `copy_from_user`. Direct deref outside this macro traps. Especially important for variable-size options (linger, sock_fprog).
- **GRKERNSEC_NO_SIMULT_CONNECT** — irrelevant on setsockopt path.
- **GRKERNSEC_BLACKHOLE** — irrelevant on setsockopt path.
- **GRKERNSEC_RANDNET** — `SO_RANDOMIZE_PORT`-style ephemeral selection inherits randnet.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — hardened build requires capabilities in the *initial* user-ns (no user-ns shortcut) for SO_MARK, SO_PRIORITY > 6, SO_RCVBUFFORCE, SO_BINDTODEVICE, IP_TRANSPARENT, IP_FREEBIND, IPV6_TRANSPARENT, IPV6_FREEBIND, SO_ATTACH_FILTER (when `kernel.unprivileged_bpf_disabled=1`).
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — `SO_PASSCRED`/`SO_PASSSEC` for UNIX-domain sockets validated against connected peer's credentials; cross-ns abstract-ns options blocked.
- **SOCK_CLOEXEC mandatory under suid** — `SO_ATTACH_BPF` carrying a prog fd: under SUID-tainted task with `grsec.suid_strict_cloexec=1`, the prog fd must have CLOEXEC set; otherwise `-EPERM`.
- **SCM_RIGHTS recursion bound** — irrelevant on setsockopt path.
- **getsockopt info-leak prevention** — setsockopt is the write side; the corresponding read-side hardening lives in getsockopt.md, but `SO_BINDTODEVICE` write here ensures the read-back name is null-terminated and not raw kernel memory.

