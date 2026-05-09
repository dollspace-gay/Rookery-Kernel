---
title: "Tier-3: net/socket-api — BSD socket layer"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the BSD/POSIX socket layer — the syscall-side userspace interface to networking. Owns `socket(2)` syscall + family (`bind`, `connect`, `accept`, `accept4`, `listen`, `socketpair`, `send`, `sendto`, `sendmsg`, `sendmmsg`, `recv`, `recvfrom`, `recvmsg`, `recvmmsg`, `shutdown`, `setsockopt`, `getsockopt`, `getsockname`, `getpeername`), the protocol-family registry (`AF_*`), and SCM (Socket Control Messages — for sendmsg/recvmsg ancillary data including SCM_RIGHTS for fd-passing).

Sub-tier-3 of `net/00-overview.md`. Every networked userspace process touches these syscalls; ABI byte-identity is foundational for drop-in compat.

### Requirements

- REQ-1: Every BSD socket syscall byte-identical entry/exit ABI per upstream.
- REQ-2: `AF_*` numeric values byte-identical; `SOCK_*` types + modifier flags byte-identical.
- REQ-3: Generic SOL_SOCKET sockopts: numeric values + struct layouts (struct linger, struct timeval/timespec) byte-identical.
- REQ-4: SCM control messages: SCM_RIGHTS fd-passing semantics identical (target task receives the dup of each fd; refcount + close-on-exec preserved); SCM_CREDENTIALS gives sender pid/uid/gid; SCM_PIDFD with SO_PASSPIDFD (newer) gives pidfd to sender.
- REQ-5: `struct msghdr`, `struct cmsghdr`, `struct iovec` UAPI byte-identical.
- REQ-6: `struct sockaddr_*` UAPI byte-identical for every protocol family.
- REQ-7: `socket(2)` dispatches to per-family `proto_ops` table; family registration via `sock_register` matches upstream.
- REQ-8: `accept(2)` / `accept4(2)` returns a new fd with SOCK_CLOEXEC + SOCK_NONBLOCK propagated per accept4 flags.
- REQ-9: `socketpair(AF_UNIX, SOCK_*, ...)` returns two connected sockets in [fd1, fd2].
- REQ-10: `sendmmsg` / `recvmmsg` (multi-message variants) preserve atomicity semantics: if N messages out of N fail, the syscall returns N (count of successfully transmitted).
- REQ-11: `shutdown(2)` semantics: SHUT_RD half-closes read; SHUT_WR half-closes write; SHUT_RDWR closes both.
- REQ-12: ZEROCOPY (`MSG_ZEROCOPY`) + `SCM_DEVMEM_DMABUF` ABIs preserved for high-perf userspace.
- REQ-13: BPF socket filter (`SO_ATTACH_FILTER` / `SO_ATTACH_BPF`) attachment ABIs preserved (cross-ref `kernel/00-overview.md` § bpf).
- REQ-14: AF_ALG (kernel-crypto socket): connect to a kernel-side algorithm; sendmsg with data; recvmsg gets transformed result. Cross-ref `crypto/00-overview.md` § af-alg.md.
- REQ-15: LSM hooks: `security_socket_create` / `security_socket_post_create` / `security_socket_bind` / `security_socket_connect` / `security_socket_listen` / `security_socket_accept` / `security_socket_sendmsg` / `security_socket_recvmsg` / `security_socket_setsockopt` / `security_socket_getsockopt` / `security_socket_shutdown` / `security_socket_sock_rcv_skb` / `security_socket_getpeersec_*` / `security_sk_alloc` / `security_sk_free` / `security_sk_clone` / `security_sk_classify_flow` — every operation has an LSM hook.
- REQ-16: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: A `strace` golden trace of a curated network test (HTTP client + server) byte-identical vs. upstream. (covers REQ-1)
- [ ] AC-2: Every `AF_*` value compile-time const-asserted to match upstream's value. (covers REQ-2)
- [ ] AC-3: `pahole struct msghdr`, `struct cmsghdr`, `struct sockaddr_in`, `struct sockaddr_in6`, `struct sockaddr_un`, `struct iovec` byte-identical layouts vs. upstream. (covers REQ-5, REQ-6)
- [ ] AC-4: An SCM_RIGHTS test passes 100 fds across an AF_UNIX SOCK_DGRAM socket; receiver gets all 100 with refcounts incremented identically. (covers REQ-4)
- [ ] AC-5: `setsockopt(SOL_SOCKET, SO_RCVBUF, &val, sizeof(val))` round-trips on every documented opt. (covers REQ-3)
- [ ] AC-6: A test exercising AF_INET, AF_INET6, AF_UNIX, AF_NETLINK, AF_PACKET each create + bind + listen + accept correctly. (covers REQ-7)
- [ ] AC-7: An `accept4(SOCK_CLOEXEC)` test confirms the returned fd has FD_CLOEXEC set. (covers REQ-8)
- [ ] AC-8: `socketpair(AF_UNIX, SOCK_STREAM, 0, sv)` returns two fds; data sent via sv[0] arrives on sv[1]. (covers REQ-9)
- [ ] AC-9: `sendmmsg(fd, msgvec, 5, 0)` returns 5 on success; if 3 succeed before EAGAIN, returns 3. (covers REQ-10)
- [ ] AC-10: `shutdown(fd, SHUT_RD)` test causes subsequent recv to return 0; SHUT_WR causes send to return EPIPE. (covers REQ-11)
- [ ] AC-11: `MSG_ZEROCOPY` test on TCP send returns expected SCM_TIMESTAMPING-on-completion notifications. (covers REQ-12)
- [ ] AC-12: An `SO_ATTACH_BPF` test attaches a BPF prog to a socket; subsequent recv'd packets are filtered per the program. (covers REQ-13)
- [ ] AC-13: An AF_ALG test creates a socket, binds to "skcipher/cbc(aes)", sends key + data; recvmsg returns ciphertext. (covers REQ-14)
- [ ] AC-14: An LSM-policy denying `security_socket_create` translates to EACCES from socket(2). (covers REQ-15)
- [ ] AC-15: Hardening section present and follows template. (covers REQ-16)

### Architecture

### Rust module organization

- `kernel::net::socket::Socket` — `struct socket` + `struct sock` wrapper
- `kernel::net::socket::syscalls` — socket / bind / connect / accept / send / recv / etc.
- `kernel::net::socket::scm` — SCM ancillary-data marshaling
- `kernel::net::socket::registry` — per-family `proto_ops` registration
- `kernel::net::socket::sock` — `struct sock` lifecycle (cross-ref `net/struct-sock.md`)
- `kernel::net::socket::sockopts` — SOL_SOCKET sockopt dispatch

### Locking and concurrency

- **Per-`sock` lock** (`sk->sk_lock`): composite of socket-bottom-half lock + slock + per-listening-queue locks. Held during recv/send.
- **Receive-queue lock** (`sk->sk_receive_queue.lock`): protects per-sock receive queue.
- **Wait queues** (`sk->sk_wq`): epoll/poll integration.

### Error handling

Standard errno set per syscall; key items: ENOTSOCK, EBADF, EINVAL, EOPNOTSUPP, EPROTONOSUPPORT, EAFNOSUPPORT, EINTR, EAGAIN/EWOULDBLOCK, EMSGSIZE, ENOMEM, EACCES, EPERM, EPIPE, ECONNRESET, ECONNABORTED, ECONNREFUSED, EHOSTUNREACH, ENETUNREACH, ETIMEDOUT.

### Out of Scope

- Per-protocol implementations (cross-ref individual net/* docs)
- struct sock detail (cross-ref `net/struct-sock.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Socket syscall family + dispatch | `net/socket.c` |
| SCM ancillary-data (SCM_RIGHTS, SCM_CREDS, SCM_TIMESTAMP, etc.) | `net/core/scm.c` |
| `struct sock` lifecycle + reference accounting | `net/core/sock.c` (cross-ref `net/struct-sock.md` Tier-3 to be authored) |
| Datagram-socket helpers | `net/core/datagram.c` |
| Public API | `include/linux/net.h`, `include/linux/socket.h`, `include/net/sock.h` |
| UAPI | `include/uapi/linux/net.h`, `include/uapi/linux/socket.h` |

### compatibility contract

### Syscalls

`socket`, `socketpair`, `bind`, `connect`, `listen`, `accept`, `accept4`, `getsockname`, `getpeername`, `send`, `sendto`, `sendmsg`, `sendmmsg`, `recv`, `recvfrom`, `recvmsg`, `recvmmsg`, `shutdown`, `setsockopt`, `getsockopt`, `socketcall` (legacy compat).

(Each gets a Tier-5 `uapi/syscalls/<name>.md` in Phase D.)

### Address families (`AF_*`)

`include/linux/socket.h` defines `enum AF_*`. Numeric values byte-identical:
- `AF_UNSPEC=0`, `AF_UNIX=1`, `AF_LOCAL=AF_UNIX`, `AF_INET=2`, `AF_AX25=3`, `AF_IPX=4`, `AF_APPLETALK=5`, `AF_NETROM=6`, `AF_BRIDGE=7`, `AF_ATMPVC=8`, `AF_X25=9`, `AF_INET6=10`, `AF_ROSE=11`, `AF_DECnet=12`, `AF_NETBEUI=13`, `AF_SECURITY=14`, `AF_KEY=15`, `AF_NETLINK=16`, `AF_PACKET=17`, `AF_ASH=18`, `AF_ECONET=19`, `AF_ATMSVC=20`, `AF_RDS=21`, `AF_SNA=22`, `AF_IRDA=23`, `AF_PPPOX=24`, `AF_WANPIPE=25`, `AF_LLC=26`, `AF_IB=27`, `AF_MPLS=28`, `AF_CAN=29`, `AF_TIPC=30`, `AF_BLUETOOTH=31`, `AF_IUCV=32`, `AF_RXRPC=33`, `AF_ISDN=34`, `AF_PHONET=35`, `AF_IEEE802154=36`, `AF_CAIF=37`, `AF_ALG=38`, `AF_NFC=39`, `AF_VSOCK=40`, `AF_KCM=41`, `AF_QIPCRTR=42`, `AF_SMC=43`, `AF_XDP=44`, `AF_MCTP=45`, `AF_MAX`.

### Socket type (`SOCK_*`)

`SOCK_STREAM=1`, `SOCK_DGRAM=2`, `SOCK_RAW=3`, `SOCK_RDM=4`, `SOCK_SEQPACKET=5`, `SOCK_DCCP=6`, `SOCK_PACKET=10`, plus modifier flags `SOCK_CLOEXEC=O_CLOEXEC`, `SOCK_NONBLOCK=O_NONBLOCK`. Numeric byte-identical.

### Generic sockopts (`SOL_SOCKET`)

`SO_DEBUG`, `SO_REUSEADDR`, `SO_TYPE`, `SO_ERROR`, `SO_DONTROUTE`, `SO_BROADCAST`, `SO_SNDBUF`, `SO_RCVBUF`, `SO_KEEPALIVE`, `SO_OOBINLINE`, `SO_LINGER`, `SO_RCVLOWAT`, `SO_RCVTIMEO`, `SO_SNDTIMEO`, `SO_BINDTODEVICE`, `SO_ATTACH_FILTER`, `SO_DETACH_FILTER`, `SO_PEERNAME`, `SO_TIMESTAMP*`, `SO_PEERCRED`, `SO_PASSCRED`, `SO_PEERSEC`, `SO_PASSSEC`, `SO_LOCK_FILTER`, `SO_BUSY_POLL`, `SO_MAX_PACING_RATE`, `SO_BPF_EXTENSIONS`, `SO_INCOMING_CPU`, `SO_ATTACH_BPF`, `SO_DETACH_BPF`, `SO_REUSEPORT`, `SO_INCOMING_NAPI_ID`, `SO_COOKIE`, `SO_PEERGROUPS`, `SO_ZEROCOPY`, `SO_TXTIME`, `SO_BINDTOIFINDEX`, `SO_DETACH_REUSEPORT_BPF`, `SO_PREFER_BUSY_POLL`, `SO_BUSY_POLL_BUDGET`, `SO_NETNS_COOKIE`, `SO_BUF_LOCK`, `SO_RESERVE_MEM`, `SO_TXREHASH`, `SO_RCVMARK`, `SO_PASSPIDFD`, `SO_PEERPIDFD`, `SO_DEVMEM_LINEAR`, `SO_DEVMEM_DMABUF`, `SO_DEVMEM_DONTNEED`, plus more recent additions.

Numeric byte-identical.

### SCM message types

`SCM_RIGHTS` (fd-passing), `SCM_CREDENTIALS`, `SCM_TIMESTAMP`, `SCM_TIMESTAMPNS`, `SCM_TIMESTAMPING`, `SCM_TIMESTAMPING_PKTINFO`, `SCM_PIDFD` (with `SO_PASSPIDFD`), `SCM_TXTIME`, `SCM_DEVMEM_DMABUF`. Numeric byte-identical.

### `struct msghdr` + `struct cmsghdr`

`include/linux/socket.h` (struct msghdr) and `include/uapi/linux/socket.h` (struct __kernel_msghdr, struct cmsghdr). UAPI byte-identical layout — every userspace using sendmsg/recvmsg depends.

### `struct sockaddr_*`

Per protocol family: `sockaddr_in` (AF_INET), `sockaddr_in6` (AF_INET6), `sockaddr_un` (AF_UNIX), `sockaddr_nl` (AF_NETLINK), `sockaddr_ll` (AF_PACKET), etc. UAPI byte-identical.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Socket alloc + sock_register dispatch | `kani::proofs::net::socket::register_safety` |
| SCM message walk + fd-pass | `kani::proofs::net::scm::walk_safety` |
| sockaddr len validation | `kani::proofs::net::socket::sockaddr_len_safety` |
| msghdr iovec walk | `kani::proofs::net::socket::msghdr_walk_safety` |

### Layer 2: TLA+ models

(none mandatory at this sub-tier — concurrency models live with the per-protocol Tier-3 docs)

### Layer 3: Kani harnesses for data-structure invariants

- `proto_ops` registry: each AF has at most one registered handler; AF_MAX bound enforced
- SCM message bounds: cmsghdr length ≥ size of data
- msghdr->iovec pointer validation under UDEREF

### Layer 4: Functional correctness (opt-in)

- **SCM_RIGHTS fd-pass correctness** via Verus — proves: every passed fd lands on the receiver with the same access semantics + refcount-incremented identically.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | sock + socket refcounts use `Refcount` (saturating); cross-ref `net/struct-sock.md` | § Mandatory |
| **AUTOSLAB** | sock + socket allocated via per-protocol `KmemCache::<Sock<P>>::new()` | § Mandatory |
| **PASSCRED / PEERCRED** (per-socket credential-passing toggle) | preserved per upstream; LSM-mediated via `security_socket_*` hooks | (existing) |

### Row-1 features consumed by this component

- **UDEREF**: every userspace addr (sockaddr, msghdr, optval) passed via `UserPtr<...>`
- **SIZE_OVERFLOW**: msg-iov size + sockopt-len arithmetic uses checked operators
- **CONSTIFY**: per-AF `proto_ops` vtables provided by protocol modules are `static const`
- **MEMORY_SANITIZE**: SCM message buffers + temp sockaddr-from-userspace zeroed on free

### Row-2 / GR-RBAC integration

This is a major LSM-hook nexus. Every documented hook (per REQ-15) is dispatched. GR-RBAC's network-restriction policies (deny socket creation by class; deny bind to privileged port for non-CAP_NET_BIND_SERVICE; deny SCM_RIGHTS pass to specific peers) all consume these hooks.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

