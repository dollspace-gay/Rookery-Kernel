# Tier-3: net/socket.c — BSD socket-API top-level dispatch (sys_socket / sys_bind / sys_connect / sys_sendmsg / sys_recvmsg / sys_select)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/socket-api.md
upstream-paths:
  - net/socket.c
  - include/linux/socket.h
  - include/uapi/linux/socket.h
-->

## Summary

`net/socket.c` is the BSD socket-API entry-point — every userspace `socket()`, `bind()`, `connect()`, `accept()`, `sendmsg()`, `recvmsg()`, `select()`, `epoll_*` call goes through it. Per-syscall: validates args, looks up per-fd struct file → struct socket, dispatches via per-AF struct proto_ops vtable. Manages per-process socket fd table integration with VFS. ~3836 lines covering: socket creation, fd-table integration, msghdr parsing, MSG_* flag handling, security_socket_* hooks, kernel socket helpers (used by NFS, iSCSI, drivers).

This Tier-3 covers `net/socket.c` (~3836 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct socket` | per-process socket | `net::socket::Socket` |
| `__sys_socket(family, type, protocol)` | sys_socket impl | `Sys::socket` |
| `__sys_socketpair(family, type, protocol, &usockvec[2])` | sys_socketpair | `Sys::socketpair` |
| `__sys_bind(fd, &addr, addrlen)` | sys_bind | `Sys::bind` |
| `__sys_connect(fd, &addr, addrlen)` | sys_connect | `Sys::connect` |
| `__sys_listen(fd, backlog)` | sys_listen | `Sys::listen` |
| `__sys_accept4(fd, &addr, &addrlen, flags)` | sys_accept4 | `Sys::accept4` |
| `__sys_getsockname(fd, &addr, &addrlen)` | sys_getsockname | `Sys::getsockname` |
| `__sys_getpeername(fd, &addr, &addrlen)` | sys_getpeername | `Sys::getpeername` |
| `__sys_sendto(fd, buff, size, flags, &addr, addrlen)` | sys_sendto | `Sys::sendto` |
| `__sys_recvfrom(fd, buff, size, flags, &addr, &addrlen)` | sys_recvfrom | `Sys::recvfrom` |
| `__sys_sendmsg(fd, &msghdr, flags, ...)` | sys_sendmsg | `Sys::sendmsg` |
| `__sys_recvmsg(fd, &msghdr, flags, ...)` | sys_recvmsg | `Sys::recvmsg` |
| `__sys_sendmmsg(fd, &mmsg, vlen, flags, &timeout)` | sys_sendmmsg | `Sys::sendmmsg` |
| `__sys_recvmmsg(fd, &mmsg, vlen, flags, &timeout)` | sys_recvmmsg | `Sys::recvmmsg` |
| `__sys_setsockopt(fd, level, optname, optval, optlen)` | sys_setsockopt | `Sys::setsockopt` |
| `__sys_getsockopt(fd, level, optname, optval, &optlen)` | sys_getsockopt | `Sys::getsockopt` |
| `__sys_shutdown(fd, how)` | sys_shutdown | `Sys::shutdown` |
| `kern_sendmsg(...)` / `kern_recvmsg(...)` | kernel-context helpers | `Sys::kern_sendmsg` / `_recvmsg` |
| `sock_alloc_file(sock, flags, dname)` | per-socket fd alloc | `Socket::alloc_file` |
| `sock_from_file(file)` | file → socket reverse-map | `Socket::from_file` |
| `sockfd_lookup(fd, &err)` | per-fd → socket | `Socket::sockfd_lookup` |
| `sockfd_lookup_light(fd, &err, &fput_needed)` | fast-path | `Socket::sockfd_lookup_light` |
| `sock_register(ops)` | per-AF register | `Socket::register` |
| `sock_unregister(family)` | per-AF unregister | `Socket::unregister` |
| `move_addr_to_kernel(uaddr, ulen, &kaddr)` | userspace → kernel addr copy | `Sys::move_addr_to_kernel` |
| `move_addr_to_user(kaddr, klen, uaddr, &ulen)` | kernel → userspace addr copy | `Sys::move_addr_to_user` |
| `verify_iovec(...)` (recvmsg) | per-iovec validation | `Sys::verify_iovec` |
| `___sys_sendmsg(...)` / `___sys_recvmsg(...)` | low-level send/recv | `Sys::___sendmsg` / `___recvmsg` |

## Compatibility contract

REQ-1: Per-process `socket`:
- `state` (SS_FREE / _UNCONNECTED / _CONNECTING / _CONNECTED / _DISCONNECTING).
- `flags` (SOCK_*: NONBLOCK, CLOEXEC, ZEROCOPY).
- `ops` (KArc<ProtoOps>; per-AF socket-API vtable).
- `file` (KWeak<File>; back-ref to fd-table entry).
- `sk` (KArc<Sock>; per-protocol struct sock).
- `wq` (per-socket wait_queue + fasync state).
- `type` (SOCK_STREAM / _DGRAM / _RAW / _PACKET / _SEQPACKET / _DCCP).

REQ-2: __sys_socket(family, type, protocol) flow:
1. Decode flags from type: SOCK_NONBLOCK / SOCK_CLOEXEC.
2. type &= SOCK_TYPE_MASK.
3. sock_create_lite(net, family, type, protocol, &sock):
   - per-AF handler->create(net, sock, protocol, kern=0).
4. sock_alloc_file(sock, flags, dname):
   - alloc anon-inode file with sock as private_data.
   - install in fd-table.
5. Return fd.

REQ-3: Per-AF dispatch:
- Per-family proto_ops registered via sock_register.
- AF_INET → inet_stream_ops / inet_dgram_ops.
- AF_UNIX → unix_stream_ops / unix_dgram_ops.
- AF_NETLINK → netlink_ops.

REQ-4: __sys_sendmsg flow:
1. fd-table lookup → socket.
2. msghdr_from_user_compat OR msghdr_from_user (per-32/64-bit).
3. iov := per-iov vector copied.
4. ___sys_sendmsg(sock, &msg, flags, used_address, allowed_msghdr_flags):
   - security_socket_sendmsg.
   - sock_sendmsg(sock, &msg).
5. Return bytes sent.

REQ-5: __sys_recvmsg flow:
1. fd-table lookup → socket.
2. msghdr_from_user.
3. ___sys_recvmsg(sock, &msg, flags):
   - security_socket_recvmsg.
   - sock_recvmsg(sock, &msg, flags).
   - Update msghdr.msg_namelen + msg_flags.
4. Copy per-msghdr fields back to userspace.

REQ-6: MSG_* flags:
- MSG_OOB (Out-of-band).
- MSG_PEEK (peek without consume).
- MSG_DONTROUTE.
- MSG_TRYHARD.
- MSG_DONTWAIT (non-blocking).
- MSG_NOSIGNAL (suppress SIGPIPE).
- MSG_FASTOPEN (TCP fast-open).
- MSG_ZEROCOPY.
- MSG_CONFIRM.
- MSG_CMSG_CLOEXEC.
- MSG_TRUNC / MSG_EOR / MSG_CTRUNC.

REQ-7: Per-AF address parsing:
- move_addr_to_kernel: userspace sockaddr → kernel sockaddr_storage.
- move_addr_to_user: reverse for getsockname / accept.
- Per-family sa_family validation.

REQ-8: SCM (Socket Control Messages):
- Per-msghdr ancillary data (cmsghdr).
- Used for: SCM_RIGHTS (fd passing), SCM_CREDENTIALS, IP_PKTINFO.

REQ-9: kernel_sendmsg / kernel_recvmsg:
- Used by NFS, iSCSI, drivers for kernel-context socket I/O.
- No userspace copy; iovs reference kernel pages.

REQ-10: Per-socket fasync (SIGIO):
- Per-socket fasync state.
- O_ASYNC + F_SETOWN: SIGIO delivered on data-arrival.

REQ-11: Per-process audit:
- audit_sockaddr per-socket-call records destination addr for logging.

REQ-12: COMPAT layer:
- 32-bit-on-64-bit handling.
- Per-msghdr 32-bit ↔ 64-bit conversion.

## Acceptance Criteria

- [ ] AC-1: socket(AF_INET, SOCK_STREAM, 0): returns fd; subsequent fd-ops valid.
- [ ] AC-2: bind() + listen() + accept(): server-side TCP socket setup.
- [ ] AC-3: sendmsg() with multi-iovec: per-iov coalesced into single skb stream.
- [ ] AC-4: recvmsg() with MSG_PEEK: data not consumed; subsequent recv returns same data.
- [ ] AC-5: SCM_RIGHTS: cmsghdr with file-descriptor; receiver gets fd in own table.
- [ ] AC-6: kernel_sendmsg from NFS-client: data sent without userspace copy.
- [ ] AC-7: getsockname / getpeername: per-socket addresses returned correctly.
- [ ] AC-8: setsockopt(SO_REUSEADDR): subsequent bind to same port allowed.
- [ ] AC-9: COMPAT: 32-bit binary on 64-bit kernel: msghdr fields handled.
- [ ] AC-10: socket() stress: 1M concurrent socket() + close() cycles; no fd leak.

## Architecture

`Socket`:

```
struct Socket {
  state: AtomicI32,                              // SS_*
  flags: u32,                                    // SOCK_*
  ops: KArc<ProtoOps>,
  file: KWeak<File>,
  sk: KArc<Sock>,
  wq: KArc<SocketWq>,
  type_: u16,
}

struct SocketWq {
  wait: WaitQueueHead,
  fasync_list: KAtomicPtr<FasyncStruct>,
}
```

`Sys::socket(family, type, protocol)`:
1. flags := type & ~SOCK_TYPE_MASK.
2. type &= SOCK_TYPE_MASK.
3. retval := sock_create(family, type, protocol, &sock).
4. If retval < 0: return retval.
5. retval := sock_alloc_file(sock, flags, NULL).
6. Return retval (fd).

`Sys::sendmsg(fd, &user_msg, flags)`:
1. sock := sockfd_lookup_light(fd, &err, &fput_needed).
2. msg := msghdr_from_user(&user_msg).
3. ___sys_sendmsg(sock, &msg, flags, NULL, 0):
   - security_socket_sendmsg.
   - sock_sendmsg(sock, &msg).
4. fput_light(file, fput_needed).
5. Return bytes_sent.

`Sys::sockfd_lookup_light(fd, &err, &fput_needed)`:
1. file := fget_light(fd, &fput_needed).
2. If !file: *err = -EBADF; return NULL.
3. sock := sock_from_file(file).
4. If !sock: fput_light; *err = -ENOTSOCK; return NULL.
5. Return sock.

`Socket::alloc_file(sock, flags, dname)`:
1. file := alloc_file_pseudo(SOCK_INODE(sock), sock_mnt, dname, O_RDWR | (flags & O_NONBLOCK), &socket_file_ops).
2. file->private_data = sock.
3. sock->file = file.
4. fd := get_unused_fd_flags(flags).
5. fd_install(fd, file).
6. Return fd.

`Sys::recvmsg(fd, &user_msg, flags)`:
1. sock := sockfd_lookup_light(fd).
2. msg := msghdr_from_user.
3. ___sys_recvmsg(sock, &msg, flags):
   - security_socket_recvmsg.
   - sock_recvmsg(sock, &msg, flags).
4. msg.msg_flags + msg.msg_namelen copied back.
5. Return bytes_received.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `socket_state_valid` | INVARIANT | per-socket state ∈ {SS_FREE, _UNCONNECTED, _CONNECTING, _CONNECTED, _DISCONNECTING}. |
| `fd_install_paired` | INVARIANT | every alloc_file paired with fd_install OR error-path unwound. |
| `sockfd_lookup_no_uaf` | UAF | per-fd lookup via fget_light; reverse via fput_light. |
| `msghdr_user_validation` | INVARIANT | per-msghdr fields (msg_iovlen, msg_controllen) bounded before kernel use. |
| `iov_per_msg_bounded` | INVARIANT | per-msg iovlen ≤ UIO_MAXIOV (1024). |

### Layer 2: TLA+

`net/socket_lifecycle.tla`:
- Per-socket state ∈ {Free, Created, Bound, Listening, Connecting, Connected, Disconnecting, Released}.
- Properties:
  - `safety_op_valid_for_state` — per-AF op valid only in matching state (e.g., listen requires Bound).
  - `safety_release_after_close` — Released → Free after VFS close completes.
  - `liveness_close_eventually_releases` — every close eventually Released.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Sys::socket` post: socket allocated; fd installed; refcount = 1 | `Sys::socket` |
| `Sys::sendmsg` post: per-protocol sendmsg called; security check passed | `Sys::sendmsg` |
| `Sys::recvmsg` post: per-msghdr fields written back; per-protocol recvmsg called | `Sys::recvmsg` |
| `Sys::sockfd_lookup_light` post: returns valid socket OR error code | `Sys::sockfd_lookup_light` |
| Per-AF sock_register / _unregister paired | invariants on AF lifecycle |

### Layer 4: Verus/Creusot functional

`Per-syscall: BSD socket-API call semantics match POSIX/SUS specification` semantic equivalence: per-call the kernel produces userspace-observable outcome matching POSIX (modulo Linux-specific extensions).

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

socket-API-specific reinforcement:

- **Per-msghdr field validation (UIO_MAXIOV cap)** — defense against user-supplied unbounded iovec.
- **Per-cmsghdr length validation** — defense against malformed ancillary data.
- **SCM_RIGHTS fd-passing capability check** — defense against unauthorized fd-leak.
- **security_socket_* hooks** — defense against unauthorized socket-API use.
- **Per-AF sock_register privileged** — defense against malicious AF takeover.
- **Per-fd fget_light reference** — defense against fd-close-during-syscall UAF.
- **Per-syscall audit_sockaddr** — defense against silent privileged-action.
- **MSG_NOSIGNAL gating** — defense against runaway SIGPIPE on closed-pipe-write.
- **COMPAT layer separate from native** — defense against 32-bit-style fields contaminating 64-bit path.
- **Per-socket flags atomic** — defense against torn flag during dup2 race.
- **fd_install delayed until fully-allocated** — defense against partial-init fd visible.
- **kernel_sendmsg validates iov in kernel-mem** — defense against accidental user-mem access.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy on every msghdr/iovec/sockaddr/optval user-buffer; `move_addr_to_kernel` + `move_addr_to_user` rely on PAX_USERCOPY validation slabs.
- **PAX_KERNEXEC** — W^X on JIT'd socket filters (`SO_ATTACH_BPF`) and per-AF `proto_ops` vtables that dispatch from syscall entry.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization at every `__sys_*` entry; the socket-API is the densest syscall cluster in the kernel.
- **PAX_REFCOUNT** — saturating refcount on every `Socket`, `SocketWq`, and file→sock back-reference; `fget_light`/`fput_light` fast-path retains saturation.
- **PAX_MEMORY_SANITIZE** — zero-on-free for SCM message buffers (especially `SCM_RIGHTS` fd arrays and `SCM_CREDENTIALS` cred blobs) and temp sockaddr-from-user storage.
- **PAX_UDEREF / PAX_MEMORY_UDEREF** — strict user-pointer access for every syscall arg; `Sys::sockfd_lookup_light` separates kernel pointer space from user pointer space.
- **GRKERNSEC_HIDESYM** — hide kernel pointers in `/proc/net/*`, `/proc/<pid>/fd`, syslog, and audit records emitted by `audit_sockaddr`.
- **GRKERNSEC_NO_SIMULT_CONNECT** — prevent same-uid parallel connect to identical dst:port (TCP SYN-flood mitigation; rate-limits per-uid).
- **GRKERNSEC_BLACKHOLE** — silent drop of probe packets to closed local ports; suppresses ICMP port-unreachable.
- **GRKERNSEC_RANDNET** — randomize TCP ISN, IP IDs, source ports on every `__sys_connect` / implicit bind.
- **GRKERNSEC_SOCK_PRIV** — socket-bind audit for privileged ports + raw sockets.
- **GRKERNSEC_TPE** — Trusted Path Execution for raw socket bind + AF_PACKET creation.
- **CAP_NET_RAW / CAP_NET_ADMIN / CAP_NET_BIND_SERVICE** strict — enforced in the binding userns, not init_user_ns; SCM_RIGHTS fd-pass cannot launder these.
- **COMPAT-layer isolation** — 32-bit compat msghdr never aliases the 64-bit field layout; PAX_REFCOUNT applies to both paths.

Per-doc rationale: `net/socket.c` is the universal syscall front door — every BSD socket call funnels through `Sys::sendmsg`/`Sys::recvmsg`/`Sys::sockfd_lookup_light`. PaX/Grsecurity reinforcement here is dense because (a) every user pointer in the system that's not a filename lands here, (b) SCM_RIGHTS can launder file-descriptor capabilities across uid boundaries unless gated, and (c) the COMPAT layer is a historical CVE source. Reinforcement focuses on UDEREF on every entry, saturating refs on fd-table interactions, and audit on every privileged bind/connect.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- struct sock (covered in `net/struct-sock.md` Tier-2 + `net/core/sock-core.md` Tier-3)
- Per-protocol sendmsg/recvmsg (covered separately)
- BPF socket filter (covered in `kernel/bpf/bpf-core.md` Tier-3)
- VFS file ops (covered in `fs/vfs/file-table.md`)
- io_uring socket (covered in `io_uring/io_uring-core.md` Tier-3)
- Implementation code
