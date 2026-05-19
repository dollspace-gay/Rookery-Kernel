---
title: "Tier-5 syscall: sendmsg(2) — send a message with ancillary data (syscall 46)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`sendmsg(2)` is the most general send primitive: it transmits a scatter/gather message (`iovec` array), optionally to an explicit destination (`msg_name`), with optional ancillary control data (`msg_control`, carrying things like `SCM_RIGHTS` FD passing, `IP_PKTINFO`, `SO_TIMESTAMPING`, `IPV6_HOPLIMIT`). It is syscall number **46** on x86-64.

The path is `sys_sendmsg(fd, umsg, flags)` → `__sys_sendmsg(fd, umsg, flags, /*forbid_cmsg_compat=*/false)` → `___sys_sendmsg` which: (1) copies the `struct user_msghdr` into kernel space, (2) copies the iovec via `import_iovec`, (3) copies the control buffer and feeds it through `scm_send` to materialize ancillary objects (FDs, credentials), (4) dispatches to `sock.ops.sendmsg(sock, &msg, len)`.

Critical for: UNIX-domain FD passing (containers, systemd-socket-activation, privilege separation), IPv6 hop-limit / pktinfo dispatch, raw socket header injection, multicast pktinfo.

This Tier-5 covers syscall **46** `sendmsg(int sockfd, const struct msghdr *msg, int flags)`.

### Acceptance Criteria

- [ ] AC-1: sendmsg on TCP socket: returns bytes sent; partial sends possible.
- [ ] AC-2: sendmsg on UDP socket > MTU + fragmentation off: -EMSGSIZE.
- [ ] AC-3: sendmsg on UNIX-stream with SCM_RIGHTS: receiver's recvmsg gets new fds.
- [ ] AC-4: SCM_RIGHTS with 254 fds: -EINVAL (over SCM_MAX_FD).
- [ ] AC-5: SCM_RIGHTS depth > 4 (FD-passing loop): -ETOOMANYREFS.
- [ ] AC-6: sendmsg on closed stream peer without MSG_NOSIGNAL: -EPIPE + SIGPIPE.
- [ ] AC-7: sendmsg on closed stream peer with MSG_NOSIGNAL: -EPIPE, no SIGPIPE.
- [ ] AC-8: msg.msg_iovlen > UIO_MAXIOV: -EMSGSIZE.
- [ ] AC-9: msg.msg_controllen > INT_MAX: -EINVAL.
- [ ] AC-10: cgroup-BPF denial: -EPERM.
- [ ] AC-11: blocking send interrupted by SIGINT: -EINTR.
- [ ] AC-12: SCM_RIGHTS with EBADF inside fd array: -EBADF, no fds passed.

### Architecture

```
SysSocket::sys_sendmsg(fd, umsg, flags) -> SyscallResult<isize>
  return Self::sys_sendmsg_inner(fd, umsg, flags, /*forbid_cmsg_compat=*/false).

SysSocket::sys_sendmsg_inner(fd, umsg, flags, forbid_compat) -> SyscallResult<isize>
  1. sock = SocketFd::lookup_light(fd)?;
  2. msg = Msghdr::default();
  3. let (uaddr_storage, mut iov) = Self::copy_msghdr_from_user(umsg, &mut msg)?;
  4. if msg.msg_iovlen as usize > UIO_MAXIOV { return Err(EMSGSIZE); }
  5. ImportIovec::write(msg.msg_iov, msg.msg_iovlen, &mut iov, &mut msg.msg_iter)?;
  6. let total_len = msg.msg_iter.count();
  7. let scm = if msg.msg_controllen != 0 {
        let ctl = UserCopy::copy_controllen(msg.msg_control, msg.msg_controllen)?;
        Scm::send(&sock, &msg, ctl)?
     } else { Scm::empty() };
  8. Lsm::socket_sendmsg(&sock, &msg, total_len)?;
  9. CgroupBpf::inet_egress(&sock, &msg)?;
 10. let sent = sock.ops.sendmsg(&sock, &mut msg, total_len)?;
 11. if !(flags & MSG_NOSIGNAL) && sent == -EPIPE { Signal::send(SIGPIPE, current()); }
 12. Audit::log_socketcall(SYS_SENDMSG, fd, ..., sent, flags);
 13. Scm::destroy_unused(&scm);
 14. Ok(sent)
```

`Scm::send(sock, msg, ctl)`:
1. parse cmsg headers using CMSG_NXTHDR.
2. for each cmsg:
   - if (level, type) == (SOL_SOCKET, SCM_RIGHTS):
     - data = cmsg payload; nr = cmsg_len / sizeof(int).
     - if nr > SCM_MAX_FD: return -EINVAL.
     - if current.scm_depth + 1 > MAX_FD_NESTING: return -ETOOMANYREFS.
     - for fd in data: file = fget(fd); if !file: return -EBADF.
     - scm.fp.fp[] = files[].
   - if (SOL_SOCKET, SCM_CREDENTIALS): copy ucred; require root or matching uid/gid/pid.
   - else: sock.ops.recv_ctl or family-specific cmsg handler.
3. return scm.

### Out of Scope

- `recvmsg(2)` — separate Tier-5.
- `sendmmsg(2)` — separate Tier-5.
- `MSG_ZEROCOPY` completion path — Tier-3 `net/core/skbuff.md`.
- Per-AF `sock.ops.sendmsg` implementations — Tier-3 per family.
- Implementation code.

### signature

```c
ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags);
```

```rust
pub fn sys_sendmsg(sockfd: i32, msg: UserPtr<UserMsghdr>, flags: i32) -> SyscallResult<isize>;
```

x86-64 entry: `__x64_sys_sendmsg` → `__sys_sendmsg(fd, umsg, flags, /*forbid_cmsg_compat=*/false)`.

### parameters

- **`sockfd`** — connected or unconnected socket fd.
- **`msg`** — pointer to `struct user_msghdr`:
  ```c
  struct user_msghdr {
      void            *msg_name;       // optional destination sockaddr
      int              msg_namelen;
      struct iovec    *msg_iov;        // scatter/gather buffers
      __kernel_size_t  msg_iovlen;
      void            *msg_control;    // ancillary cmsg buffer
      __kernel_size_t  msg_controllen;
      unsigned         msg_flags;      // returned only by recvmsg
  };
  ```
- **`flags`** — `MSG_*` flags: `MSG_OOB`, `MSG_DONTROUTE`, `MSG_DONTWAIT`, `MSG_NOSIGNAL`, `MSG_CONFIRM`, `MSG_MORE`, `MSG_EOR`, `MSG_FASTOPEN`, `MSG_ZEROCOPY`, `MSG_CMSG_CLOEXEC` (cmsg on RECV side; ignored here), `MSG_SPLICE_PAGES`, `MSG_BATCH`.

### return

Number of bytes sent (≥ 0) on success, `-1`/`-errno` on error. Note: positive return < total iov bytes is possible (partial send for stream sockets in non-blocking mode).

### errors

- `EBADF` — sockfd not open.
- `ENOTSOCK` — fd not a socket.
- `EFAULT` — msg / msg_name / msg_iov / msg_control fault.
- `EINVAL` — msg_iovlen > UIO_MAXIOV (1024); msg_controllen > INT_MAX; msg_namelen invalid; flags invalid for family.
- `EAGAIN` / `EWOULDBLOCK` — non-blocking send would block.
- `EMSGSIZE` — message too large for unframed datagram (UDP > 65507; UNIX_DGRAM > sk_sndbuf-margin).
- `EPIPE` — stream socket peer closed (and `MSG_NOSIGNAL` not set ⟹ also `SIGPIPE` delivered).
- `ENOMEM` — slab alloc fail in skb path.
- `ENOBUFS` — local socket buffer full and SOCK_DGRAM can't queue.
- `EHOSTUNREACH` / `ENETUNREACH` / `ECONNREFUSED` / `ECONNRESET` — peer/route errors.
- `EISCONN` — sendmsg with msg_name on connected stream socket.
- `ENOTCONN` — sendmsg on unconnected stream socket.
- `EOPNOTSUPP` — flag not supported by family.
- `EINTR` — blocking send interrupted by signal.
- `EACCES` — broadcast without SO_BROADCAST; LSM denial.
- `EPERM` — cgroup-BPF denial; scm_send denial.
- `ETOOMANYREFS` — SCM_RIGHTS recursion limit exceeded.
- `EBADF` — SCM_RIGHTS fd inside cmsg is invalid.

### abi surface

- `struct user_msghdr` is the userspace shape; the kernel converts to `struct msghdr` internally (which uses `struct iov_iter` for iov + kernel pointers for cmsg).
- `struct cmsghdr` ancillary header: `{ socklen_t cmsg_len; int cmsg_level; int cmsg_type; }` followed by data, aligned to `__alignof__(struct cmsghdr)`.
- `SCM_RIGHTS` (level=SOL_SOCKET, type=0x01): payload is an array of file descriptors. Recursion limit is `SCM_MAX_FD` (= 253 in 7.1.0-rc2) per message and a per-task "in-flight FD passing depth" limit of `MAX_FD_NESTING` (= 4) to prevent FD-passing loops.
- `MSG_CMSG_CLOEXEC` — receive-side only; ignored here.

### compatibility contract

REQ-1: fd lookup:
- f = sockfd_lookup_light(sockfd); if !f: return `-EBADF`.
- if f.f_op != &socket_file_ops: return `-ENOTSOCK`.

REQ-2: msghdr copy:
- err = copy_msghdr_from_user(&msg, umsg, &uaddr_storage, &iov, /*save_addr=*/true).
- if err: return err.
- if msg.msg_iovlen > UIO_MAXIOV: return `-EMSGSIZE`.

REQ-3: iov import:
- err = import_iovec(WRITE, msg.msg_iov, msg.msg_iovlen, ARRAY_SIZE(iovstack), &iov, &msg.msg_iter).
- if err: return err.
- total_len = iov_iter_count(&msg.msg_iter).

REQ-4: cmsg copy + scm_send:
- if msg.msg_controllen:
  - if msg.msg_controllen > INT_MAX: return `-EINVAL`.
  - ctl_buf = kmalloc(msg.msg_controllen, GFP_KERNEL); copy_from_user.
  - err = scm_send(sock, &msg, &scm); if err: return err.
- else: scm = empty.

REQ-5: LSM:
- err = security_socket_sendmsg(sock, &msg, total_len); if err: return err.

REQ-6: Dispatch:
- err = sock.ops.sendmsg(sock, &msg, total_len).
- if err < 0 and !partial: return err.

REQ-7: scm_send (net/core/scm.c):
- iter over cmsgs; per cmsg:
  - if level == SOL_SOCKET ∧ type == SCM_RIGHTS: scm_fp_copy → arrange to install FDs in receiver.
  - if level == SOL_SOCKET ∧ type == SCM_CREDENTIALS: copy ucred, validate.
  - else: per-AF cmsg handler.

REQ-8: SCM_RIGHTS recursion limit:
- per scm_fp_copy: for each fd, fcheck and get_file, check that current `unix_inflight + 1 ≤ max_files`.
- per-task `current.scm_depth + 1 ≤ MAX_FD_NESTING (4)`; else return `-ETOOMANYREFS`.
- record fds into `scm.fp.fp[]` array (max SCM_MAX_FD = 253).

REQ-9: SIGPIPE:
- per-stream peer reset + !(flags & MSG_NOSIGNAL): send_sig(SIGPIPE, current).

REQ-10: cgroup-BPF:
- BPF_CGROUP_RUN_PROG_INET_EGRESS at sendmsg site for AF_INET/6.

REQ-11: Audit:
- audit_log_socketcall(SYS_SENDMSG, fd, msg_name?, msg_namelen, ret, flags).

REQ-12: msg_iter consumed:
- on success: iov_iter_count(&msg.msg_iter) == 0 or partial-bytes consumed.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sendmsg_iovlen_bounds` | INVARIANT | per-sys_sendmsg: msg_iovlen ≤ UIO_MAXIOV. |
| `sendmsg_controllen_bounds` | INVARIANT | per-sys_sendmsg: msg_controllen ≤ INT_MAX. |
| `sendmsg_scm_rights_max_fd` | INVARIANT | per-scm_send: SCM_RIGHTS fd count ≤ SCM_MAX_FD. |
| `sendmsg_scm_recursion_bound` | INVARIANT | per-scm_send: scm_depth ≤ MAX_FD_NESTING. |
| `sendmsg_sigpipe_only_without_nosignal` | INVARIANT | per-sys_sendmsg: SIGPIPE delivered ⟹ !(flags & MSG_NOSIGNAL). |
| `sendmsg_no_partial_state_on_error` | INVARIANT | per-error: scm allocations freed; iov dropped. |

### Layer 2: TLA+

`uapi/syscalls/sendmsg.tla`:
- States: validate, msghdr-copy, iov-import, ctl-copy, scm-send, lsm, dispatch, return, signal.
- Models SCM_RIGHTS fd-passing graph; tracks scm_depth.
- Properties:
  - `safety_scm_rights_no_loop` — fd-passing graph stays acyclic of depth ≤ 4.
  - `safety_iov_count_eq` — total_len == iov_iter_count pre-dispatch.
  - `safety_partial_send_consistent` — return value bytes ≤ total_len.
  - `liveness_terminates` — sendmsg returns within bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| pre: msg_iovlen ≤ UIO_MAXIOV | `sys_sendmsg` |
| pre: scm.fp.count ≤ SCM_MAX_FD | `scm_send` |
| post: success ⟹ bytes_sent ≤ total_len | `sys_sendmsg` |
| post: error ⟹ scm.fp.count == 0 (all refs dropped) | `sys_sendmsg` |
| post: SIGPIPE delivered ⟹ stream sock peer closed ∧ !MSG_NOSIGNAL | `sys_sendmsg` |

### Layer 4: Verus/Creusot functional

Per-`__sys_sendmsg → ___sys_sendmsg → scm_send → sock.ops.sendmsg` semantic equivalence with `net/socket.c` and `net/core/scm.c`. SCM_RIGHTS fp[] population order and refcount get/put balance match upstream.

### hardening

- **msg_iovlen ≤ UIO_MAXIOV (1024)** — defense against per-iov-array-explosion (kmalloc bomb).
- **msg_controllen ≤ INT_MAX cap** — defense against per-cmsg-buffer kmalloc bomb.
- **SCM_RIGHTS count ≤ SCM_MAX_FD (253)** — defense against per-cmsg fd-blast.
- **MAX_FD_NESTING (= 4) recursion bound on SCM_RIGHTS-in-flight** — defense against per-FD-passing-loop DoS.
- **`fget` per SCM_RIGHTS fd verifies validity before scm.fp.fp[] insertion** — defense against per-EBADF-after-commit.
- **SCM_CREDENTIALS validated against current creds + capabilities** — defense against per-cred-spoof.
- **MSG_NOSIGNAL controls SIGPIPE — both paths handled** — defense against per-uncatchable-pipe.
- **LSM `socket_sendmsg`** — defense against per-MAC bypass.
- **cgroup-BPF `INET_EGRESS`** — defense against per-egress-policy bypass.
- **`scm_destroy` on error path drops all refs** — defense against per-leaked-fd.
- **Per-stream `SO_SNDBUF` accounting** — defense against per-OOM via giant queue.

### grsecurity-pax

- **PAX_RANDKSTACK** — per-syscall kstack base randomization; protects on-stack `iov[]` and `cmsg` parsing scratch.
- **PaX UDEREF** — all userland derefs (msg, msg_name, msg_iov, msg_control) go through `copy_from_user`. Direct deref traps. Especially load-bearing here since `msghdr` aggregates 3 separate userland pointers.
- **GRKERNSEC_NO_SIMULT_CONNECT** — irrelevant on sendmsg path.
- **GRKERNSEC_BLACKHOLE** — irrelevant on sendmsg path (egress).
- **GRKERNSEC_RANDNET** — sendmsg sequencing on raw / netlink sockets gets extra entropy; IP_ID selection on each emitted skb.
- **CAP_NET_RAW / CAP_NET_ADMIN gates** — already enforced at socket creation; sendmsg revalidates `SOCK_RAW` header-include payloads (`IP_HDRINCL`) against CAP_NET_RAW.
- **GRKERNSEC_HARDEN_IPC for SOCK_*UNIX** — UNIX-domain sendmsg with SCM_CREDENTIALS validates pid against current pid (or CAP_SYS_RESOURCE), preventing peer-pid spoof; SCM_RIGHTS to abstract-ns sockets blocked across user-ns under hardened config.
- **SOCK_CLOEXEC mandatory under suid** — irrelevant on sendmsg path (no new fd).
- **SCM_RIGHTS recursion bound** — central grsec rule: hardened build sets `MAX_FD_NESTING = 4` (vs upstream 4) and adds rate-limiting on SCM_RIGHTS passes per sender per second; oversize bursts return `-EMFILE` early.
- **getsockopt info-leak prevention** — irrelevant on sendmsg path.

