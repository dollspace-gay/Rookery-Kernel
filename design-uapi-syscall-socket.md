---
title: "Tier-5 syscall: socket(2) — create a socket endpoint (syscall 41)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`socket(2)` is the entrypoint into the kernel networking stack: it allocates a `struct socket`, binds it to a per-`(family, type, protocol)` `struct proto_ops` dispatch vector, and installs the resulting `struct file` into the caller's fd table. It is syscall number **41** on x86-64.

The triple `(domain, type, protocol)` resolves through `sock_create()` → `__sock_create()` → `net_families[domain].create()` (per-AF factory) → `pf->create(net, sock, protocol, kern=0)`. The new socket is wrapped in a file via `sock_alloc_file()` (which honors `SOCK_CLOEXEC` and `SOCK_NONBLOCK` ORed into `type`) and registered via `fd_install()`.

Critical for: every networked process, every UNIX-domain IPC user (X11, dbus, systemd, container runtimes), every privilege-separation socketpair pattern, every container's network namespace egress.

This Tier-5 covers syscall **41** `socket(int domain, int type, int protocol)`.

### Acceptance Criteria

- [ ] AC-1: `socket(AF_INET, SOCK_STREAM, 0)` returns valid fd; `fstat(fd).st_mode & S_IFMT == S_IFSOCK`.
- [ ] AC-2: `socket(AF_INET, SOCK_STREAM | SOCK_CLOEXEC, 0)`: `fcntl(fd, F_GETFD) & FD_CLOEXEC` set.
- [ ] AC-3: `socket(AF_INET, SOCK_DGRAM | SOCK_NONBLOCK, 0)`: `fcntl(fd, F_GETFL) & O_NONBLOCK` set.
- [ ] AC-4: `socket(AF_FOO_UNKNOWN, SOCK_STREAM, 0)` → `-EAFNOSUPPORT`.
- [ ] AC-5: `socket(AF_INET, SOCK_STREAM | 0x80000000, 0)` → `-EINVAL`.
- [ ] AC-6: `socket(AF_INET, SOCK_RAW, IPPROTO_ICMP)` unprivileged → `-EACCES`; with `CAP_NET_RAW` → success.
- [ ] AC-7: `socket(AF_UNIX, SOCK_STREAM, 0)` succeeds in any user-ns.
- [ ] AC-8: Hitting `RLIMIT_NOFILE` → `-EMFILE`.
- [ ] AC-9: SELinux `socket_create` denial → `-EACCES`.
- [ ] AC-10: AF auto-load: missing module triggers single `request_module` attempt.

### Architecture

```
SysSocket::sys_socket(domain, ty, protocol) -> SyscallResult<i32>
  1. validate ty: mask = SOCK_TYPE_MASK | SOCK_CLOEXEC | SOCK_NONBLOCK
     if ty & !mask: return Err(EINVAL).
  2. flags = ty & (SOCK_CLOEXEC | SOCK_NONBLOCK).
  3. base  = ty & SOCK_TYPE_MASK.
  4. sock = Socket::create(current_net(), domain, base, protocol, kern=false)?;
  5. file = Socket::alloc_file(sock, flags, name=None)?;
  6. fd   = FdTable::get_unused_fd_flags(flags & O_CLOEXEC)?;
  7. FdTable::install(fd, file);
  8. Audit::log_socketcall(SYS_SOCKET, domain, ty, protocol, fd);
  9. return Ok(fd).

Socket::create(net, domain, ty, proto, kern) -> Result<Arc<Socket>>:
  - if domain < 0 || domain as usize >= NPROTO: return Err(EAFNOSUPPORT).
  - Lsm::socket_create(domain, ty, proto, kern)?;
  - pf = net_families::get(domain).or_else(|| {
        ModLoader::request("net-pf-%d", domain);
        net_families::get(domain)
    }).ok_or(EAFNOSUPPORT)?;
  - sock = SocketSlab::alloc().ok_or(ENFILE)?;
  - sock.set_type(ty);
  - pf.create(net, &sock, proto, kern)?;          // fills sock.ops, sk
  - Lsm::socket_post_create(&sock, domain, ty, proto, kern);
  - Ok(sock).
```

Per-domain pf.create populates `sock.ops` with the per-(family, type) `struct proto_ops` vtable (e.g. `inet_stream_ops`, `unix_stream_ops`) and allocates the inner `struct sock` (TCP: `tcp_sock`; UDP: `udp_sock`; UNIX: `unix_sock`).

### Out of Scope

- `socketpair(2)` (syscall 53) — separate Tier-5.
- `bind(2)` / `connect(2)` / `listen(2)` / `accept*(2)` — separate Tier-5s.
- Per-AF `pf.create` factories — Tier-3s per `net/ipv4/af_inet.md`, `net/unix/af_unix.md`, etc.
- `struct proto_ops` vtable internals — Tier-3 per `net/socket.md`.
- Implementation code.

### signature

```c
int socket(int domain, int type, int protocol);
```

```rust
pub fn sys_socket(domain: i32, ty: i32, protocol: i32) -> SyscallResult<i32>;
```

x86-64 entry: `__x64_sys_socket` → `__sys_socket(domain, type, protocol)` → returns positive fd or negative `-errno`.

### parameters

- **`domain`** — protocol family (`AF_UNIX`, `AF_INET`, `AF_INET6`, `AF_PACKET`, `AF_NETLINK`, `AF_VSOCK`, …). Encoded as the `__kernel_sa_family_t` (u16) but passed widened to `int`. Indexes `net_families[]`.
- **`type`** — base socket type (`SOCK_STREAM`, `SOCK_DGRAM`, `SOCK_SEQPACKET`, `SOCK_RAW`, `SOCK_RDM`, `SOCK_DCCP`, `SOCK_PACKET`) ORed with the modifier bits `SOCK_CLOEXEC` (= `O_CLOEXEC`) and `SOCK_NONBLOCK` (= `O_NONBLOCK`). Base = `type & SOCK_TYPE_MASK` (0x0f).
- **`protocol`** — `IPPROTO_*` selector within the family: 0 means "default for (domain,type)" (e.g. `(AF_INET, SOCK_STREAM, 0)` → TCP). For `AF_PACKET` carries the `htons(ETH_P_*)` selector.

### return

On success: a non-negative fd (the smallest free fd in the caller's fd table).
On error: `-1` and `errno` set (in-kernel: `-errno` returned).

### errors

- `EAFNOSUPPORT` — domain unknown or its module not loaded; `request_module("net-pf-%d", domain)` failed and `net_families[domain]` is still NULL.
- `EINVAL` — `type & ~(SOCK_TYPE_MASK | SOCK_CLOEXEC | SOCK_NONBLOCK)` ≠ 0; base type unknown for family; protocol invalid.
- `EPROTONOSUPPORT` — `(family, type, protocol)` combination not supported by the per-AF factory.
- `EACCES` — `SOCK_RAW`/`SOCK_PACKET` without `CAP_NET_RAW` in net-ns owning user-ns; or LSM (SELinux `socket_create`) denial.
- `ENFILE` — system-wide open-file limit reached (`get_empty_filp()` failed).
- `EMFILE` — per-process fd-table limit reached (`__get_unused_fd_flags()` failed).
- `ENOMEM` — slab allocation failure for `struct socket` / `struct sock` / `struct file`.
- `ENOBUFS` — per-AF factory ran out of resources (e.g. AF_NETLINK port table exhausted).

### abi surface

`SOCK_CLOEXEC` = `O_CLOEXEC` (02000000), `SOCK_NONBLOCK` = `O_NONBLOCK` (00004000). `SOCK_TYPE_MASK` = 0x0f. `SOCK_MAX` = 11. `AF_MAX` = 46. Domain range validated in `__sock_create()` against `NPROTO`.

Per-AF `struct net_proto_family` table `net_families[NPROTO]` (RCU-protected, `net_families_lock` rwsem on update). Per-`net_family_register()` slot fills exactly once per pf.

### compatibility contract

REQ-1: Argument validation:
- if `(type & ~(SOCK_TYPE_MASK | SOCK_CLOEXEC | SOCK_NONBLOCK)) != 0`: return `-EINVAL`.
- if `domain < 0 ∨ domain >= NPROTO`: return `-EAFNOSUPPORT`.
- flags = type & (SOCK_CLOEXEC | SOCK_NONBLOCK); base = type & SOCK_TYPE_MASK.

REQ-2: Per-family lookup:
- rcu_read_lock; pf = rcu_dereference(net_families[domain]); rcu_read_unlock.
- if !pf: request_module("net-pf-%d", domain); retry once.
- if still !pf: return `-EAFNOSUPPORT`.

REQ-3: Per-AF factory:
- sock = sock_alloc(); if !sock: return `-ENFILE`.
- sock.type = base.
- err = pf.create(net, sock, protocol, /*kern=*/0); if err: sock_release(sock); return err.

REQ-4: File wrap + fd install:
- file = sock_alloc_file(sock, flags & (O_CLOEXEC | O_NONBLOCK), NULL); if IS_ERR(file): return err.
- fd = get_unused_fd_flags(flags & O_CLOEXEC); if fd < 0: fput(file); return fd.
- fd_install(fd, file); return fd.

REQ-5: SOCK_RAW / SOCK_PACKET:
- pf.create checks `ns_capable(net.user_ns, CAP_NET_RAW)`. If not held: return `-EPERM` (mapped to `EACCES` for legacy contract on AF_PACKET).

REQ-6: LSM hooks:
- security_socket_create(family, type, protocol, kern=0) before pf.create.
- security_socket_post_create(sock, family, type, protocol, kern=0) after.

REQ-7: Audit:
- audit_log(AUDIT_SOCKETCALL, ...) on syscall return per `auditsc.c`.

REQ-8: kern=0 always for syscall path (vs. kern=1 for in-kernel sock_create_kern).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `socket_type_mask_strict` | INVARIANT | per-sys_socket: any bit outside SOCK_TYPE_MASK|CLOEXEC|NONBLOCK ⟹ EINVAL. |
| `socket_domain_bounds` | INVARIANT | per-sys_socket: domain ∉ [0, NPROTO) ⟹ EAFNOSUPPORT. |
| `socket_fd_cleanup_on_error` | INVARIANT | per-sys_socket: any post-alloc-file failure ⟹ fput + sock_release; no fd leak. |
| `socket_cloexec_threading` | INVARIANT | per-fd_install: O_CLOEXEC flag set atomically with fd visibility. |
| `socket_raw_cap_check` | INVARIANT | per-AF_INET/PACKET SOCK_RAW: CAP_NET_RAW checked before pf.create returns success. |

### Layer 2: TLA+

`uapi/syscalls/socket.tla`:
- States: pre-validation, pf-lookup, sock-alloc, pf-create, file-alloc, fd-install, return.
- Properties:
  - `safety_no_partial_fd` — failure on any post-allocation step releases all resources.
  - `safety_module_load_once` — request_module attempted at most once per call.
  - `liveness_returns_or_errors` — every call terminates with fd ≥ 0 or -errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| post: ret ≥ 0 ⟹ fd-table[ret].file.f_op == &socket_file_ops | `sys_socket` |
| post: ret ≥ 0 ⟹ socket.ops != NULL | `Socket::create` |
| post: err ⟹ no new fd, no new file, no new sock | `sys_socket` |
| pre: kern == 0 on syscall path | `Socket::create` |

### Layer 4: Verus/Creusot functional

`socket(AF_INET, SOCK_STREAM, 0)` → installed fd refers to a `struct socket` with `ops == inet_stream_ops`, `sk.sk_family == AF_INET`, `sk.sk_type == SOCK_STREAM`, `sk.sk_protocol == IPPROTO_TCP`, per `net/ipv4/af_inet.c:inet_create`.

### hardening

- **Domain bounds-checked against `NPROTO`** — defense against per-OOB family table read.
- **Per-`SOCK_TYPE_MASK | CLOEXEC | NONBLOCK` strict mask** — defense against per-flag-smuggling future-bit injection.
- **Per-`CAP_NET_RAW` for SOCK_RAW / SOCK_PACKET** — defense against per-unprivileged-sniffer.
- **Per-`CAP_NET_BIND_SERVICE` deferred to bind**, not here.
- **Per-LSM `socket_create` hook** — defense against per-MAC bypass.
- **Per-`fd_install` ordering: file fully initialized before fd visible** — defense against per-TOCTOU on new fd.
- **Per-`SOCK_CLOEXEC` honored atomically** — defense against per-fork/exec FD leak (see grsec note).
- **Per-`request_module` rate-limited / forbidden in unprivileged user-ns** — defense against per-autoload abuse (`modules_autoload` sysctl).
- **Per-`sock_alloc` slab `GFP_KERNEL_ACCOUNT`** — defense against per-cgroup-memory-bypass.

### grsecurity-pax

- **PAX_RANDKSTACK** — per-syscall entry randomizes kstack base before `sys_socket` so probes of return addresses leak less.
- **PaX UDEREF** — not directly applicable here (no userland pointer in `sys_socket` itself) but inherited by downstream `bind`/`connect`/`sendmsg` which dereference sockaddr/msghdr.
- **GRKERNSEC_NO_SIMULT_CONNECT** — relevant transitively: a `socket()` followed by simultaneous `connect()` storm against the same peer is rate-throttled per-task. (Enforcement lives in connect.)
- **GRKERNSEC_BLACKHOLE** — connect-time blackholing of TCP/UDP scan attempts; socket() itself only allocates the fd.
- **GRKERNSEC_RANDNET** — randomizes initial network state used by accept queues and ephemeral ports (transitive).
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — SOCK_RAW, SOCK_PACKET, and several `IPPROTO_*` selectors require these; PaX-hardened kernels require them in the *initial* user-ns (no user-ns shortcut) when `kernel.grsecurity.disable_priv_userns=1`.
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — UNIX-domain sockets get hardened ACL on abstract namespace creation: `AF_UNIX` abstract sockets are restricted to the creating user under `grsec.harden_ipcs`.
- **SOCK_CLOEXEC mandatory under suid** — when the calling task transitioned through suid/sgid recently (`PF_SUID_DUMP`/`current.flags & PF_SUPERPRIV` + grsec policy), `SOCK_CLOEXEC` is force-ORed into `flags` regardless of caller request, preventing FD leak across `execve`.
- **SCM_RIGHTS recursion bound** — deferred to sendmsg(); enforced via `MAX_FD_NESTING=4` across the socket family.
- **getsockopt info-leak prevention** — `SO_PEERCRED` and `SO_PEERSEC` outputs are zero-padded; socket-create path attaches `struct sock_cgroup_data` early so SO_PEERCRED is consistent.

