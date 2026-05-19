---
title: "Tier-5 syscall: getsockopt(2) — get socket option (syscall 55)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`getsockopt(2)` reads a socket option at a given protocol level into a userland buffer, writing back the actual length consumed. The `(level, optname)` tuple dispatches to per-level handlers analogous to `setsockopt(2)`. It is syscall number **55** on x86-64.

The path is `sys_getsockopt(fd, level, optname, optval, optlen)` → `__sys_getsockopt` → `sock.ops.getsockopt(sock, level, optname, optval, optlen)` (or `sock_getsockopt` for SOL_SOCKET). Each handler validates the user-provided `*optlen`, computes the option value, copies it out, and updates `*optlen`.

`optlen` is value-result: caller provides buffer size, kernel returns actual length written. This is the canonical surface for the eBPF `BPF_PROG_TYPE_CGROUP_SOCKOPT` getsockopt hook, which can synthesize entirely new option values.

Critical for: every server's connection introspection (`SO_ERROR`, `SO_PEERCRED`, `TCP_INFO`), every TLS handshake observability path, every container's network debugging.

This Tier-5 covers syscall **55** `getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen)`.

### Acceptance Criteria

- [ ] AC-1: getsockopt(SOL_SOCKET, SO_TYPE, &out, &len): out == SOCK_STREAM (or socket's type); len == 4.
- [ ] AC-2: getsockopt(SOL_SOCKET, SO_ERROR, ...): returns pending error; subsequent read returns 0.
- [ ] AC-3: getsockopt(SOL_SOCKET, SO_RCVBUF, ...): returns kernel value (Linux convention = 2× requested).
- [ ] AC-4: getsockopt(SOL_SOCKET, SO_PEERCRED, ...) on UNIX-stream connected: returns peer ucred.
- [ ] AC-5: getsockopt(SOL_SOCKET, SO_PEERCRED, ...) on TCP socket: -ENOPROTOOPT.
- [ ] AC-6: getsockopt with caller's *optlen smaller than option size: truncated; *optlen reflects bytes written (truncated).
- [ ] AC-7: getsockopt with unknown optname: -ENOPROTOOPT.
- [ ] AC-8: getsockopt optval fault: -EFAULT.
- [ ] AC-9: getsockopt optlen pointer fault: -EFAULT.
- [ ] AC-10: getsockopt(IPPROTO_TCP, TCP_INFO, &tcp_info, &len): all fields populated; len == sizeof(struct tcp_info).
- [ ] AC-11: getsockopt(SOL_SOCKET, SO_BINDTODEVICE, ...): null-terminated string ≤ IFNAMSIZ.
- [ ] AC-12: BPF cgroup getsockopt hook returns synthesized value; kernel handler skipped.

### Architecture

```
SysSocket::sys_getsockopt(fd, level, optname, optval, optlen) -> SyscallResult<i32>
  1. sock = SocketFd::lookup_light(fd)?;
  2. let max_len: i32 = UserCopy::get_user(optlen)? as i32;
  3. if max_len < 0 { return Err(EINVAL); }
  4. Lsm::socket_getsockopt(&sock, level, optname)?;
  5. let mut ctx = BpfGetsockoptCtx::new(level, optname, optval, max_len as u32);
  6. let pre_handled = CgroupBpf::getsockopt_pre(&sock, &mut ctx)?;
  7. if !pre_handled {
        let err = if level == SOL_SOCKET {
            Sock::getsockopt(&sock, level, optname, optval, optlen)
        } else {
            let op = sock.ops.getsockopt.ok_or(EOPNOTSUPP)?;
            op(&sock, level, optname, optval, optlen)
        };
        err?;
     }
  8. CgroupBpf::getsockopt_post(&sock, &mut ctx)?;
  9. ctx.write_back(optval, optlen)?;            // truncates to user's max_len
 10. Audit::log_socketcall(SYS_GETSOCKOPT, fd, None, 0, 0, level, optname);
 11. Ok(0)
```

Per-`Sock::getsockopt(sock, level, optname, optval, optlen)`:
1. let max_len: i32 = get_user(optlen)?; if max_len < 0: return EINVAL.
2. let mut buf = SmallScratch::zeroed();    // zero-init for info-leak prevention
3. lock_sock(sk).
4. switch optname:
   - SO_ERROR: val = xchg(&sk.sk_err, 0); buf.write_int(val); len = 4.
   - SO_TYPE: buf.write_int(sk.sk_type); len = 4.
   - SO_DOMAIN: buf.write_int(sk.sk_family); len = 4.
   - SO_LINGER: ling.l_onoff = sock_flag(sk, SOCK_LINGER); ling.l_linger = sk.sk_lingertime/HZ; copy len = 8.
   - SO_RCVBUF: buf.write_int(sk.sk_rcvbuf); len = 4.
   - SO_PEERCRED:
     - if sk.sk_family != AF_UNIX: return ENOPROTOOPT.
     - if sk.sk_state != TCP_ESTABLISHED ∧ !u.peercred_set: return ENOTCONN.
     - copy ucred (zero-padded).
   - SO_BINDTODEVICE: read iface name; null-terminate.
   - default: return ENOPROTOOPT.
5. release_sock(sk).
6. let copy_len = min(len, max_len).
7. copy_to_user(optval, &buf, copy_len).
8. put_user(copy_len, optlen).
9. return 0.

### Out of Scope

- `setsockopt(2)` — separate Tier-5.
- Per-option implementation details — Tier-3 per family.
- BPF cgroup getsockopt program type — Tier-3 `kernel/bpf/cgroup.md`.
- TCP_INFO struct definition — Tier-5 UAPI header `uapi/headers/tcp.md`.
- Implementation code.

### signature

```c
int getsockopt(int sockfd, int level, int optname,
               void *optval, socklen_t *optlen);
```

```rust
pub fn sys_getsockopt(
    sockfd: i32, level: i32, optname: i32,
    optval: UserPtr<u8>, optlen: UserPtr<u32>,
) -> SyscallResult<i32>;
```

x86-64 entry: `__x64_sys_getsockopt` → `__sys_getsockopt(fd, level, optname, optval, optlen)`.

### parameters

- **`sockfd`** — socket fd.
- **`level`** — protocol level (`SOL_SOCKET`, `IPPROTO_IP`, `IPPROTO_TCP`, `IPPROTO_IPV6`, `IPPROTO_ICMPV6`, `SOL_NETLINK`, `SOL_PACKET`, `SOL_XDP`, …).
- **`optname`** — option number within level.
- **`optval`** — userland output buffer.
- **`optlen`** — value-result `socklen_t *`. On entry: buffer length. On return: actual length written.

### return

0 on success; `-1`/`-errno` on error.

### errors

- `EBADF` — sockfd not open.
- `ENOTSOCK` — fd not a socket.
- `EFAULT` — optval or optlen fault.
- `EINVAL` — `*optlen` negative; optname unknown for level.
- `ENOPROTOOPT` — option not supported at this level.
- `ENOTCONN` — option requires connected sock (`SO_PEERCRED`, `SO_PEERSEC`, `TCP_INFO` on closed).
- `EACCES` / `EPERM` — LSM denial; CAP gate on rare introspection (`SO_PEERSEC` with non-self peer ns).
- `ENOMEM` — slab alloc fail.

### abi surface

- `optlen` is value-result. On entry caller-provided; on return kernel-written. Kernel writes back the actual number of bytes placed in `optval`. If actual ≤ caller-provided: writes actual. Most options return exact size; some (`TCP_CONGESTION`, `SO_BINDTODEVICE`) return a string of variable length truncated to `*optlen`.
- For BPF-getsockopt rewrites: BPF program may write up to `PAGE_SIZE` of synthesized data; kernel still truncates to caller's `*optlen`.
- Zero-padding: short reads (kernel value smaller than caller's buffer) must NOT leak adjacent kernel memory. Modern path uses `copy_to_user` with the exact computed length and writes the new `*optlen` reflecting that.

### Common readback options

| Option | Type | Notes |
|---|---|---|
| `SO_ERROR` | int | clears sk_err on read |
| `SO_TYPE` / `SO_DOMAIN` / `SO_PROTOCOL` | int | read-only |
| `SO_ACCEPTCONN` | int (bool) | LISTEN state |
| `SO_REUSEADDR` / `SO_REUSEPORT` | int (bool) | |
| `SO_KEEPALIVE` | int (bool) | |
| `SO_LINGER` | struct linger | |
| `SO_RCVBUF` / `SO_SNDBUF` | int | kernel-doubled value (per Linux convention) |
| `SO_RCVTIMEO_OLD` / `_NEW` | struct timeval / __kernel_sock_timeval | |
| `SO_PEERCRED` | struct ucred | UNIX-only |
| `SO_PEERSEC` | char[] | LSM context |
| `SO_PEERGROUPS` | gid_t[] | UNIX-only |
| `SO_BINDTODEVICE` | char[IFNAMSIZ] | name string |
| `SO_COOKIE` | u64 | sk_cookie |
| `SO_NETNS_COOKIE` | u64 | net-ns cookie |
| `SO_PEEK_OFF` | int | recv MSG_PEEK offset |
| `TCP_INFO` | struct tcp_info | TCP connection stats |
| `TCP_CONGESTION` | char[] | algorithm name |
| `IP_PKTOPTIONS` | struct ip_options[] | reconstructed IP options |

### compatibility contract

REQ-1: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-2: optlen read:
- get_user(max_len, optlen).
- if max_len < 0: return `-EINVAL`.

REQ-3: LSM:
- err = security_socket_getsockopt(sock, level, optname); if err: return err.

REQ-4: BPF cgroup hook (pre + post):
- BPF_CGROUP_RUN_PROG_GETSOCKOPT may pre-empt (synthesize value) and/or post-process (rewrite kernel value).
- If pre-empt: kernel handler skipped; BPF result used.
- If post-process: BPF runs after kernel handler.

REQ-5: Level dispatch:
- if level == SOL_SOCKET:
  - err = sock_getsockopt(sock, level, optname, optval, optlen).
- else:
  - if !sock.ops.getsockopt: return `-EOPNOTSUPP`.
  - err = sock.ops.getsockopt(sock, level, optname, optval, optlen).

REQ-6: SOL_SOCKET handler (sock_getsockopt):
- if max_len < expected for optname (read-side may shrink): write min and adjust *optlen.
- switch optname:
  - SO_ERROR: val = sock_error(sk); copy_int.
  - SO_TYPE: copy_int(sk.sk_type).
  - SO_DOMAIN: copy_int(sk.sk_family).
  - SO_PROTOCOL: copy_int(sk.sk_protocol).
  - SO_ACCEPTCONN: copy_int(sk.sk_state == TCP_LISTEN).
  - SO_REUSEADDR: copy_int(sk.sk_reuse).
  - SO_KEEPALIVE: copy_int(sock_flag(sk, SOCK_KEEPOPEN)).
  - SO_LINGER: struct linger filled; copy_struct.
  - SO_RCVBUF: copy_int(sk.sk_rcvbuf).
  - SO_SNDBUF: copy_int(sk.sk_sndbuf).
  - SO_PEERCRED: requires UNIX + connected; copy struct ucred.
  - SO_PEERSEC: requires UNIX + connected; copy LSM context string.
  - SO_BINDTODEVICE: copy iface name; null-terminated; truncate to max_len.
  - SO_COOKIE: copy u64(sk.sk_cookie).
  - SO_NETNS_COOKIE: copy u64(sk.sk_net.cookie).
  - default: ENOPROTOOPT.

REQ-7: Info-leak prevention:
- Per-option: kernel computes exact size; copy_to_user copies exactly that many bytes; *optlen updated to exact size.
- Never copy more than computed size, never less without zero-fill.
- For struct readbacks (linger, ucred, tcp_info): scratch struct zero-initialized before population.

REQ-8: SO_ERROR side effect:
- sock_error(sk) reads sk.sk_err AND clears it atomically.

REQ-9: SO_PEERCRED state requirement:
- UNIX-only; if sk_state != ESTABLISHED ∧ not listen-passed-cred path: return `-ENOTCONN`.

REQ-10: BPF rewrite bounds:
- BPF prog can write up to PAGE_SIZE; kernel truncates to user's max_len; updates *optlen accordingly.

REQ-11: Audit:
- audit_log_socketcall(SYS_GETSOCKOPT, fd, NULL, 0, ret, level, optname).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `getsockopt_optlen_signed` | INVARIANT | per-sys_getsockopt: *optlen < 0 ⟹ EINVAL. |
| `getsockopt_scratch_zeroed` | INVARIANT | per-handler: scratch struct zeroed before population. |
| `getsockopt_copy_bytes_eq_optlen` | INVARIANT | per-handler: copy_to_user bytes == *optlen writeback. |
| `getsockopt_no_overread` | INVARIANT | per-handler: bytes written ≤ caller's max_len. |
| `getsockopt_so_error_clears` | INVARIANT | per-SO_ERROR: post-read sk.sk_err == 0. |
| `getsockopt_peercred_unix_only` | INVARIANT | per-SO_PEERCRED: sk_family == AF_UNIX else ENOPROTOOPT. |

### Layer 2: TLA+

`uapi/syscalls/getsockopt.tla`:
- States: validate, lsm, bpf-pre, dispatch, handler, bpf-post, writeback, return.
- Properties:
  - `safety_optlen_writeback_le_caller` — kernel *optlen ≤ caller *optlen pre.
  - `safety_no_kernel_leak` — bytes copied ≤ computed-size; remainder of caller buffer unmodified.
  - `safety_so_error_atomic_clear` — sk_err read and cleared atomically.
  - `liveness_terminates` — call returns within bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: *optlen ≥ 0 | `sys_getsockopt` |
| post: success ⟹ bytes_copied == *optlen (writeback) | `sys_getsockopt` |
| post: success ⟹ bytes_copied ≤ caller's max_len | `sys_getsockopt` |
| post: SO_ERROR success ⟹ sk.sk_err == 0 | `Sock::getsockopt` |
| post: success ⟹ no uninitialized scratch byte copied | `Sock::getsockopt` |
| post: SO_PEERCRED success ⟹ sk_family == AF_UNIX | `Sock::getsockopt` |

### Layer 4: Verus/Creusot functional

Per-`__sys_getsockopt → sock_getsockopt / sock.ops.getsockopt` semantic equivalence with `net/socket.c` and `net/core/sock.c`. Each option's value computation matches upstream byte-for-byte for ucred, linger, tcp_info, etc.

### hardening

- **`*optlen` read with `get_user`, signed-checked** — defense against per-negative-len OOB-write into user buffer.
- **Scratch struct zero-init before per-handler population** — defense against per-info-leak of kernel stack into `optval`.
- **`copy_to_user` bytes == writeback `*optlen`** — defense against per-overread / leak-of-adjacent-kernel-mem.
- **Per-option exact-size computation** — defense against per-padded-struct uninitialized bytes (`struct linger` reserved field, `struct tcp_info` padding).
- **`SO_ERROR` atomic read-and-clear** — defense against per-double-error-delivery.
- **`SO_PEERCRED` / `SO_PEERSEC` require AF_UNIX + connected** — defense against per-cred-fishing on unrelated sockets.
- **`SO_BINDTODEVICE` returns null-terminated string** — defense against per-iface-name overflow into adjacent bytes.
- **LSM `socket_getsockopt`** — defense against per-MAC bypass.
- **cgroup-BPF getsockopt hook** — defense against per-introspection bypass; can synthesize or rewrite.
- **`SO_DOMAIN/_TYPE/_PROTOCOL` read-only constants** — defense against per-spoof; consistent value derived from sk at creation.

### grsecurity-pax

- **PAX_RANDKSTACK** — per-syscall kstack randomization; on-stack scratch displaced.
- **PaX UDEREF** — every `copy_to_user` / `put_user` of optval/optlen goes through UDEREF macros. Direct deref of `optval` outside that path traps. Important here because getsockopt is the **write side** to user memory.
- **GRKERNSEC_NO_SIMULT_CONNECT** — irrelevant on getsockopt path.
- **GRKERNSEC_BLACKHOLE** — irrelevant on getsockopt path.
- **GRKERNSEC_RANDNET on accept queue** — `SO_ACCEPTCONN` reflects listening state; queue ordering randomization opaque to getsockopt readers.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — hardened build: `SO_PEERSEC` for cross-user-ns peers requires CAP_AUDIT_READ; `SO_PEERPIDFD` requires CAP_SYS_ADMIN to expose pidfd of UNIX peer.
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — `SO_PEERCRED` populated only with credentials that grsec validated at accept-time; cross-ns abstract-ns peers return zeroed ucred + `-EPERM` if `grsec.harden_ipcs` set.
- **SOCK_CLOEXEC mandatory under suid** — getsockopt does not return fds; the corresponding cmsg-installed FDs use `MSG_CMSG_CLOEXEC` (covered in recvmsg).
- **SCM_RIGHTS recursion bound** — irrelevant on getsockopt path.
- **getsockopt info-leak prevention** — THE central grsec rule for this syscall:
  - every per-option handler must zero-initialize scratch before population;
  - every `copy_to_user` byte-count must equal the writeback `*optlen`;
  - every struct readback (`struct linger`, `struct ucred`, `struct tcp_info`, `struct icmp6_filter`) explicitly zeroes padding/reserved fields;
  - `SO_PEERCRED.pid` is the namespace-translated pid (no init-ns pid leak);
  - `SO_PEERSEC` truncation never leaves a non-NUL terminator into uninitialized bytes;
  - `SO_BINDTODEVICE` writes a null-terminated string with explicit memset.

