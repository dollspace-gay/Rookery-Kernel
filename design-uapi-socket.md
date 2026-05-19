---
title: "Tier-5 UAPI: include/uapi/linux/socket.h + include/linux/socket.h — Socket ABI"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

The socket UAPI is the syscall-edge contract for `socket(2)`, `bind(2)`, `connect(2)`, `listen(2)`, `accept(2)`/`accept4(2)`, `send(2)` family, `recv(2)` family, `socketpair(2)`, `shutdown(2)`, `getsockname(2)`, and ancillary-data (`SCM_*`) delivery. It defines:

- **Socket-type enum** (`SOCK_STREAM`, `SOCK_DGRAM`, `SOCK_RAW`, `SOCK_RDM`, `SOCK_SEQPACKET`, `SOCK_DCCP`, `SOCK_PACKET`) plus the `SOCK_CLOEXEC` / `SOCK_NONBLOCK` modifier bits ORed into the `type` argument.
- **Address-family table** `AF_* / PF_*` (0..45, with `AF_MAX == 46`), the legacy `PF_ROUTE = PF_NETLINK` alias, and the `AF_UNSPEC` wildcard.
- **Address structures**: `struct sockaddr` (legacy 14-byte `sa_data`), `struct sockaddr_unsized` (flexible-array variant for in-kernel callbacks), `struct __kernel_sockaddr_storage` (128-byte aligned superset exposed to userspace as `struct sockaddr_storage`).
- **Message-passing structures**: `struct msghdr` (kernel-side), `struct user_msghdr` (userspace ABI), `struct mmsghdr` (multi-message variant), `struct cmsghdr` (ancillary data), `struct ucred` (peer credentials for `SCM_CREDENTIALS`).
- **Control-message types** `SCM_RIGHTS`, `SCM_CREDENTIALS`, `SCM_SECURITY`, `SCM_PIDFD`.
- **Send/recv flags** `MSG_*` (0x01..0x08000000) including `MSG_OOB`, `MSG_PEEK`, `MSG_DONTWAIT`, `MSG_NOSIGNAL`, `MSG_CMSG_CLOEXEC`, `MSG_ZEROCOPY`, `MSG_FASTOPEN`.
- **`setsockopt(2)` levels** `SOL_*` (0..287) — IP/TCP/UDP plus per-AF protocol levels.
- **`CMSG_*` ancillary-data macros** (`CMSG_FIRSTHDR`, `CMSG_NXTHDR`, `CMSG_DATA`, `CMSG_SPACE`, `CMSG_LEN`, `CMSG_ALIGN`).
- **`SOCK_SNDBUF_LOCK` / `SOCK_RCVBUF_LOCK`** buffer-lock bits and `SOCK_TXREHASH_*` transmit-rehash policy values.

Critical for: every networking syscall, every networked daemon, every container-runtime socket-passing flow (Docker, Kubernetes, systemd-socket-activation), every privilege-separation pattern relying on `SCM_RIGHTS` FD passing.

This Tier-5 covers `include/uapi/linux/socket.h` and consolidates the directly-related kernel header `include/linux/socket.h` which the UAPI client effectively pulls in via libc's `<sys/socket.h>`.

### Acceptance Criteria

- [ ] AC-1: `sizeof(struct __kernel_sockaddr_storage) == 128` and is naturally pointer-aligned.
- [ ] AC-2: `sizeof(struct sockaddr) == 16`; `sizeof(struct ucred) == 12`; `sizeof(struct linger) == 8`.
- [ ] AC-3: `sizeof(struct user_msghdr)` matches glibc/musl on every arch (28 bytes ILP32, 56 bytes LP64).
- [ ] AC-4: `CMSG_SPACE(0)` and `CMSG_LEN(0)` match Linux's values for `sizeof(struct cmsghdr)`.
- [ ] AC-5: `for_each_cmsghdr` iterates a fuzzed control buffer without out-of-bounds read; terminates on truncation.
- [ ] AC-6: `socket(AF_INET, SOCK_STREAM, 0)` returns fd; `socket(AF_INET, SOCK_STREAM | SOCK_CLOEXEC, 0)` returns fd with `FD_CLOEXEC` set.
- [ ] AC-7: `socket(AF_MAX, SOCK_STREAM, 0)` returns `-EAFNOSUPPORT`; `socket(AF_INET, SOCK_STREAM | 0x80, 0)` returns `-EINVAL`.
- [ ] AC-8: AF/PF numeric table (0..45) round-trips: each `AF_X == PF_X`.
- [ ] AC-9: `SCM_RIGHTS` cmsg sent across `AF_UNIX socketpair` with `MSG_CMSG_CLOEXEC`: receiver sees `O_CLOEXEC` on installed fds.
- [ ] AC-10: `SCM_CREDENTIALS` cross-checked: forging `ucred.pid` of another process without `CAP_SYS_ADMIN` returns `-EPERM`.
- [ ] AC-11: `SCM_PIDFD` in `sendmsg(2)` returns `-EINVAL`; in `recvmsg(2)` on a peer-credential-enabled socket returns a valid pidfd.
- [ ] AC-12: `recvmsg(2)` with undersized `msg_controllen` sets `MSG_CTRUNC`.
- [ ] AC-13: `MSG_NOSIGNAL` on `send(2)` over a broken stream returns `-EPIPE` without delivering SIGPIPE.
- [ ] AC-14: `listen(fd, 1 << 30)` clamps to `SOMAXCONN == 4096`.
- [ ] AC-15: `sendmsg(2)` with `MSG_SPLICE_PAGES | MSG_SENDPAGE_NOPOLICY | MSG_SENDPAGE_DECRYPTED` in user flags has those bits masked off before reaching the protocol layer.
- [ ] AC-16: `setsockopt(fd, SOL_SOCKET, SO_SNDBUF, ...)` followed by `SO_SNDBUFFORCE` interaction respects `SOCK_SNDBUF_LOCK`.
- [ ] AC-17: AF_NETLINK `socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE)` returns fd; alias `AF_ROUTE` resolves identically.
- [ ] AC-18: Per arch, `struct cmsghdr` and `CMSG_ALIGN` use `sizeof(long)` alignment matching Linux (8 LP64, 4 ILP32).

### Architecture

```
pub struct Sockaddr {
    pub sa_family: SaFamilyT,   // u16
    pub sa_data: [u8; 14],
}

#[repr(C)]
pub union KernelSockaddrStorage {
    fields: KernelSockaddrStorageFields,
    align: *mut c_void,
}
#[repr(C)]
pub struct KernelSockaddrStorageFields {
    pub ss_family: KernelSaFamilyT, // u16
    pub __data: [u8; 128 - core::mem::size_of::<u16>()],
}

pub struct UserMsghdr {
    pub msg_name: UserPtr<c_void>,
    pub msg_namelen: i32,
    pub msg_iov: UserPtr<Iovec>,
    pub msg_iovlen: KernelSizeT,
    pub msg_control: UserPtr<c_void>,
    pub msg_controllen: KernelSizeT,
    pub msg_flags: u32,
}

pub struct Cmsghdr {
    pub cmsg_len: KernelSizeT,
    pub cmsg_level: i32,
    pub cmsg_type: i32,
    /* payload follows at CMSG_ALIGN(sizeof::<Cmsghdr>()) */
}

pub struct Ucred {
    pub pid: u32,
    pub uid: u32,
    pub gid: u32,
}
```

`Socket::move_addr_to_kernel(uaddr: UserPtr<u8>, ulen: i32, kaddr: &mut KernelSockaddrStorage) -> Result<()>`:
1. if `ulen < 0 || ulen > sizeof(KernelSockaddrStorage)`: return Err(EINVAL).
2. if ulen == 0: kaddr.ss_family = AF_UNSPEC; return Ok.
3. copy_from_user_strict(kaddr, uaddr, ulen) — bounded by ulen ≤ 128.
4. return Ok.

`Cmsg::nxthdr(ctl: *u8, size: usize, cmsg: *Cmsghdr) -> Option<*Cmsghdr>`:
1. /* Bounded step */
2. step = CMSG_ALIGN(cmsg.cmsg_len).
3. nxt = cmsg.cast::<u8>().wrapping_add(step).cast::<Cmsghdr>().
4. /* Bounds check end-of-next */
5. if (nxt.add(1) as *u8).wrapping_sub(ctl) as usize > size: return None.
6. return Some(nxt).

`SysSocket::sys_socket(family: i32, type_: i32, protocol: i32) -> Result<i32>`:
1. /* Validate family */
2. if family < 0 || family >= AF_MAX: return Err(EAFNOSUPPORT).
3. /* Split SOCK_* from CLOEXEC/NONBLOCK */
4. sock_type = type_ & SOCK_TYPE_MASK.
5. flags = type_ & !SOCK_TYPE_MASK.
6. if flags & !(SOCK_CLOEXEC | SOCK_NONBLOCK) != 0: return Err(EINVAL).
7. /* Per LSM */
8. security_socket_create(family, sock_type, protocol, kern=0)?.
9. /* Allocate sk + struct socket */
10. sock = SockAlloc::alloc(family, sock_type)?.
11. /* Protocol-family dispatch */
12. net_families[family].create(sock, protocol, kern=0)?.
13. /* Map to fd */
14. fd = SockAlloc::map_fd(sock, flags & SOCK_CLOEXEC, flags & SOCK_NONBLOCK)?.
15. return Ok(fd).

`SysSocket::sys_accept4(fd, upeer, upeer_len, flags) -> Result<i32>`:
1. if flags & !(SOCK_CLOEXEC | SOCK_NONBLOCK) != 0: return Err(EINVAL).
2. file = fdget(fd)?.
3. listener = file.socket()?.
4. accept_arg = ProtoAcceptArg { flags, kern: 0, .. }.
5. newfile = Socket::do_accept(file, &accept_arg, upeer, upeer_len, flags)?.
6. newfd = SockAlloc::map_fd(newfile, flags & SOCK_CLOEXEC, flags & SOCK_NONBLOCK)?.
7. return Ok(newfd).

`SysSocket::sys_recvmsg(fd, umsg: UserPtr<UserMsghdr>, flags, forbid_compat) -> Result<i64>`:
1. /* Mask out internal-only flags */
2. flags = flags & !MSG_INTERNAL_SENDMSG_FLAGS.
3. /* Copy header */
4. msg = Self::copy_msghdr(umsg)?.
5. if !forbid_compat && (flags & MSG_CMSG_COMPAT) != 0: dispatch compat.
6. sock = sockfd_lookup(fd)?.
7. /* Per protocol recvmsg */
8. ret = sock.proto.recvmsg(sock, &mut msg, flags)?.
9. /* Set MSG_CTRUNC / MSG_TRUNC into msg.msg_flags */
10. /* Copy msg.msg_flags back to user */
11. put_user(msg.msg_flags, &umsg.msg_flags)?.
12. return Ok(ret).

### Out of Scope

- `include/uapi/asm/socket.h` per-arch `SO_*` socket-options (covered in `uapi/headers/asm-socket.md` Tier-5)
- `include/linux/sockios.h` `SIOCxxx` ioctl numbers (covered in `uapi/headers/sockios.md` Tier-5)
- `include/uapi/linux/in.h` / `in6.h` / `un.h` / `netlink.h` per-AF structs (covered separately)
- Protocol-family implementation (`net/ipv4/`, `net/ipv6/`, `net/unix/`, `net/netlink/`) — covered in `net/00-overview.md` Tier-3
- `iovec` UAPI (covered in `uapi/headers/uio.md` Tier-5)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `__kernel_sa_family_t` | per-AF u16 tag | `KernelSaFamilyT` (`u16`) |
| `sa_family_t` | per-AF userspace alias | `SaFamilyT` (`u16`) |
| `struct __kernel_sockaddr_storage` | per-128-byte aligned storage | `KernelSockaddrStorage` |
| `struct sockaddr` | per-legacy 16-byte addr | `Sockaddr` |
| `struct sockaddr_unsized` | per-flexible-array addr | `SockaddrUnsized` |
| `struct linger` | per-SO_LINGER option | `Linger` |
| `struct msghdr` | per-kernel msg control block | `Msghdr` |
| `struct user_msghdr` | per-userspace msg control block | `UserMsghdr` |
| `struct mmsghdr` | per-recvmmsg/sendmmsg | `Mmsghdr` |
| `struct cmsghdr` | per-ancillary-data header | `Cmsghdr` |
| `struct ucred` | per-SCM_CREDENTIALS payload | `Ucred` |
| `__cmsg_nxthdr` | per-CMSG iter step | `Cmsg::nxthdr` |
| `cmsg_nxthdr` | per-msghdr iter step | `Cmsg::nxthdr_msg` |
| `move_addr_to_kernel` | per-copy_from_user addr | `Socket::move_addr_to_kernel` |
| `put_cmsg` | per-emit ancillary | `Socket::put_cmsg` |
| `put_cmsg_notrunc` | per-emit ancillary non-truncating | `Socket::put_cmsg_notrunc` |
| `__sys_socket` | per-socket(2) helper | `SysSocket::sys_socket` |
| `__sys_bind` | per-bind(2) helper | `SysSocket::sys_bind` |
| `__sys_connect` | per-connect(2) helper | `SysSocket::sys_connect` |
| `__sys_listen` | per-listen(2) helper | `SysSocket::sys_listen` |
| `__sys_accept4` | per-accept4(2) helper | `SysSocket::sys_accept4` |
| `__sys_recvmsg` / `__sys_sendmsg` | per-recvmsg/sendmsg helper | `SysSocket::sys_{recv,send}msg` |
| `__sys_recvmmsg` / `__sys_sendmmsg` | per-multi-msg helper | `SysSocket::sys_{recv,send}mmsg` |
| `__sys_socketpair` | per-socketpair(2) helper | `SysSocket::sys_socketpair` |
| `__sys_shutdown` | per-shutdown(2) helper | `SysSocket::sys_shutdown` |

### abi surface (constants + structs)

### Socket types (`include/linux/net.h:80-95`)

```text
SOCK_STREAM     = 1   /* connection-oriented byte-stream (TCP, UNIX-stream) */
SOCK_DGRAM      = 2   /* connectionless datagram (UDP, UNIX-dgram) */
SOCK_RAW        = 3   /* raw protocol access (requires CAP_NET_RAW) */
SOCK_RDM        = 4   /* reliably-delivered message (unordered) */
SOCK_SEQPACKET  = 5   /* sequenced, reliable, connection-oriented */
SOCK_DCCP       = 6   /* Datagram Congestion Control Protocol */
SOCK_PACKET     = 10  /* obsolete: get packets at device level (use AF_PACKET) */
SOCK_MAX        = 11
SOCK_TYPE_MASK  = 0x0f
SOCK_CLOEXEC    = O_CLOEXEC   /* close-on-exec for the new fd */
SOCK_NONBLOCK   = O_NONBLOCK  /* nonblocking I/O on the new socket */
```

The `type` argument to `socket(2)` and `socketpair(2)` is `(SOCK_* | SOCK_CLOEXEC | SOCK_NONBLOCK)`. The base type is extracted via `type & SOCK_TYPE_MASK`.

### Address / protocol families (`include/linux/socket.h:204-310`)

| AF | # | Purpose | PF alias |
|---|---|---|---|
| `AF_UNSPEC` | 0 | unspecified | `PF_UNSPEC` |
| `AF_UNIX` / `AF_LOCAL` | 1 | UNIX-domain socket | `PF_UNIX` / `PF_LOCAL` |
| `AF_INET` | 2 | IPv4 | `PF_INET` |
| `AF_AX25` | 3 | Amateur Radio AX.25 | `PF_AX25` |
| `AF_IPX` | 4 | Novell IPX | `PF_IPX` |
| `AF_APPLETALK` | 5 | AppleTalk DDP | `PF_APPLETALK` |
| `AF_NETROM` | 6 | Amateur Radio NET/ROM | `PF_NETROM` |
| `AF_BRIDGE` | 7 | multiprotocol bridge | `PF_BRIDGE` |
| `AF_ATMPVC` | 8 | ATM PVCs | `PF_ATMPVC` |
| `AF_X25` | 9 | X.25 | `PF_X25` |
| `AF_INET6` | 10 | IPv6 | `PF_INET6` |
| `AF_ROSE` | 11 | Amateur Radio X.25 PLP | `PF_ROSE` |
| `AF_DECnet` | 12 | DECnet | `PF_DECnet` |
| `AF_NETBEUI` | 13 | 802.2 LLC | `PF_NETBEUI` |
| `AF_SECURITY` | 14 | security callback pseudo-AF | `PF_SECURITY` |
| `AF_KEY` | 15 | PF_KEY key-management API | `PF_KEY` |
| `AF_NETLINK` (`AF_ROUTE`) | 16 | kernel/user message bus | `PF_NETLINK` (`PF_ROUTE`) |
| `AF_PACKET` | 17 | raw L2 packets | `PF_PACKET` |
| `AF_ASH` | 18 | Ash | `PF_ASH` |
| `AF_ECONET` | 19 | Acorn Econet | `PF_ECONET` |
| `AF_ATMSVC` | 20 | ATM SVCs | `PF_ATMSVC` |
| `AF_RDS` | 21 | Reliable Datagram Sockets | `PF_RDS` |
| `AF_SNA` | 22 | Linux SNA | `PF_SNA` |
| `AF_IRDA` | 23 | IrDA | `PF_IRDA` |
| `AF_PPPOX` | 24 | PPPoX | `PF_PPPOX` |
| `AF_WANPIPE` | 25 | Wanpipe API | `PF_WANPIPE` |
| `AF_LLC` | 26 | Linux LLC | `PF_LLC` |
| `AF_IB` | 27 | native InfiniBand | `PF_IB` |
| `AF_MPLS` | 28 | MPLS | `PF_MPLS` |
| `AF_CAN` | 29 | Controller Area Network | `PF_CAN` |
| `AF_TIPC` | 30 | TIPC | `PF_TIPC` |
| `AF_BLUETOOTH` | 31 | Bluetooth | `PF_BLUETOOTH` |
| `AF_IUCV` | 32 | IUCV (s390) | `PF_IUCV` |
| `AF_RXRPC` | 33 | RxRPC | `PF_RXRPC` |
| `AF_ISDN` | 34 | mISDN | `PF_ISDN` |
| `AF_PHONET` | 35 | Phonet | `PF_PHONET` |
| `AF_IEEE802154` | 36 | IEEE 802.15.4 | `PF_IEEE802154` |
| `AF_CAIF` | 37 | CAIF | `PF_CAIF` |
| `AF_ALG` | 38 | crypto user-API | `PF_ALG` |
| `AF_NFC` | 39 | NFC | `PF_NFC` |
| `AF_VSOCK` | 40 | host/guest VM sockets | `PF_VSOCK` |
| `AF_KCM` | 41 | Kernel Connection Multiplexor | `PF_KCM` |
| `AF_QIPCRTR` | 42 | Qualcomm IPC Router | `PF_QIPCRTR` |
| `AF_SMC` | 43 | RDMA-accelerated SMC | `PF_SMC` |
| `AF_XDP` | 44 | AF_XDP zero-copy | `PF_XDP` |
| `AF_MCTP` | 45 | Management Component Transport Protocol | `PF_MCTP` |
| `AF_MAX` | 46 | upper bound | `PF_MAX` |

`PF_*` and `AF_*` are numerically identical; `PF_*` is the historical "protocol family" spelling used in `socket(2)`, `AF_*` is the "address family" spelling stored in `sa_family`.

### Address structures (`include/linux/socket.h:36-63`, `include/uapi/linux/socket.h:16-27`)

```text
struct sockaddr {
    sa_family_t  sa_family;   /* AF_* */
    char         sa_data[14]; /* protocol-specific payload */
};

struct sockaddr_unsized {
    __kernel_sa_family_t sa_family;
    char                 sa_data[]; /* flexible: total size known to caller */
};

struct __kernel_sockaddr_storage {  /* exposed as struct sockaddr_storage */
    union {
        struct {
            __kernel_sa_family_t ss_family;        /* AF_* */
            char __data[128 - sizeof(unsigned short)];
        };
        void *__align;                              /* force pointer alignment */
    };
};
#define _K_SS_MAXSIZE 128
```

Family-specific overlays (defined in their own UAPIs but ABI-locked here):

- `struct sockaddr_in` (`AF_INET`, `<linux/in.h>`): `sin_family`, `sin_port` (BE16), `sin_addr.s_addr` (BE32), 8-byte zero pad.
- `struct sockaddr_in6` (`AF_INET6`, `<linux/in6.h>`): `sin6_family`, `sin6_port`, `sin6_flowinfo`, `sin6_addr.s6_addr[16]`, `sin6_scope_id`.
- `struct sockaddr_un` (`AF_UNIX`, `<linux/un.h>`): `sun_family`, `sun_path[108]` (filesystem or abstract-namespace path, leading NUL = abstract).
- `struct sockaddr_nl` (`AF_NETLINK`, `<linux/netlink.h>`): `nl_family`, `nl_pad`, `nl_pid`, `nl_groups`.

### Message-passing structures (`include/linux/socket.h:71-124`)

```text
struct msghdr {                    /* kernel-side */
    void           *msg_name;
    int             msg_namelen;
    int             msg_inq;
    struct iov_iter msg_iter;
    union {
        void       *msg_control;
        void __user *msg_control_user;
    };
    bool            msg_control_is_user : 1;
    bool            msg_get_inq         : 1;
    unsigned int    msg_flags;
    __kernel_size_t msg_controllen;
    struct kiocb   *msg_iocb;
    struct ubuf_info *msg_ubuf;
    int (*sg_from_iter)(struct sk_buff *, struct iov_iter *, size_t);
};

struct user_msghdr {               /* userspace ABI */
    void __user      *msg_name;
    int               msg_namelen;
    struct iovec __user *msg_iov;
    __kernel_size_t   msg_iovlen;
    void __user      *msg_control;
    __kernel_size_t   msg_controllen;
    unsigned int      msg_flags;
};

struct mmsghdr {
    struct user_msghdr msg_hdr;
    unsigned int       msg_len;     /* bytes transferred on this msg */
};

struct cmsghdr {
    __kernel_size_t cmsg_len;       /* including header */
    int             cmsg_level;     /* SOL_* */
    int             cmsg_type;      /* SCM_* or protocol-specific */
    /* followed by CMSG_ALIGN(sizeof(struct cmsghdr)) bytes then payload */
};

struct ucred {                      /* SCM_CREDENTIALS payload */
    __u32 pid;
    __u32 uid;
    __u32 gid;
};

struct linger {                     /* SO_LINGER payload */
    int l_onoff;
    int l_linger;                   /* seconds */
};
```

### Control-message types (`include/linux/socket.h:193-196`)

```text
SCM_RIGHTS       = 0x01   /* array of int file descriptors */
SCM_CREDENTIALS  = 0x02   /* struct ucred peer credentials */
SCM_SECURITY     = 0x03   /* security label string (LSM) */
SCM_PIDFD        = 0x04   /* pidfd of peer (recv-only) */
```

### `MSG_*` send/recv flags (`include/linux/socket.h:319-356`)

```text
MSG_OOB              = 0x00000001   /* out-of-band data */
MSG_PEEK             = 0x00000002   /* leave queued */
MSG_DONTROUTE        = 0x00000004   /* bypass routing */
MSG_TRYHARD          = 0x00000004   /* DECnet alias */
MSG_CTRUNC           = 0x00000008   /* control truncated */
MSG_PROBE            = 0x00000010   /* MTU probe */
MSG_TRUNC            = 0x00000020   /* normal-data truncated */
MSG_DONTWAIT         = 0x00000040   /* nonblocking */
MSG_EOR              = 0x00000080   /* end of record */
MSG_WAITALL          = 0x00000100   /* full request */
MSG_FIN              = 0x00000200
MSG_SYN              = 0x00000400
MSG_CONFIRM          = 0x00000800   /* path-confirm */
MSG_RST              = 0x00001000
MSG_ERRQUEUE         = 0x00002000   /* fetch error queue */
MSG_NOSIGNAL         = 0x00004000   /* no SIGPIPE */
MSG_MORE             = 0x00008000
MSG_WAITFORONE       = 0x00010000   /* recvmmsg: block until 1+ */
MSG_SENDPAGE_NOPOLICY= 0x00010000   /* sendpage internal */
MSG_BATCH            = 0x00040000   /* sendmmsg: more coming */
MSG_NO_SHARED_FRAGS  = 0x00080000   /* sendpage internal */
MSG_SENDPAGE_DECRYPTED=0x00100000  /* sendpage internal */
MSG_SOCK_DEVMEM      = 0x02000000   /* devmem skbs as cmsg */
MSG_ZEROCOPY         = 0x04000000   /* kernel keeps user data ref */
MSG_SPLICE_PAGES     = 0x08000000   /* splice iter pages */
MSG_FASTOPEN         = 0x20000000   /* data in TCP SYN */
MSG_CMSG_CLOEXEC     = 0x40000000   /* O_CLOEXEC on received SCM_RIGHTS fds */
MSG_CMSG_COMPAT      = 0x80000000   /* 32-on-64 compat fixups (CONFIG_COMPAT) */
MSG_EOF              = MSG_FIN
MSG_INTERNAL_SENDMSG_FLAGS = MSG_SPLICE_PAGES | MSG_SENDPAGE_NOPOLICY | MSG_SENDPAGE_DECRYPTED
```

`MSG_INTERNAL_SENDMSG_FLAGS` MUST be masked off on syscall entry — a userspace `sendmsg(2)` MUST NOT pass any of those bits through.

### `setsockopt(2)` levels `SOL_*` (`include/linux/socket.h:363-403`)

```text
SOL_IP        =   0  SOL_TCP       =   6  SOL_UDP       =  17
SOL_IPV6      =  41  SOL_ICMPV6    =  58  SOL_SCTP      = 132
SOL_UDPLITE   = 136  SOL_RAW       = 255  SOL_IPX       = 256
SOL_AX25      = 257  SOL_ATALK     = 258  SOL_NETROM    = 259
SOL_ROSE      = 260  SOL_DECNET    = 261  SOL_X25       = 262
SOL_PACKET    = 263  SOL_ATM       = 264  SOL_AAL       = 265
SOL_IRDA      = 266  SOL_NETBEUI   = 267  SOL_LLC       = 268
SOL_DCCP      = 269  SOL_NETLINK   = 270  SOL_TIPC      = 271
SOL_RXRPC     = 272  SOL_PPPOL2TP  = 273  SOL_BLUETOOTH = 274
SOL_PNPIPE    = 275  SOL_RDS       = 276  SOL_IUCV      = 277
SOL_CAIF      = 278  SOL_ALG       = 279  SOL_NFC       = 280
SOL_KCM       = 281  SOL_TLS       = 282  SOL_XDP       = 283
SOL_MPTCP     = 284  SOL_MCTP      = 285  SOL_SMC       = 286
SOL_VSOCK     = 287
```

`SOL_ICMP = 1` was historically squatted by Linux (collides with `IPPROTO_ICMP`) and is intentionally not defined; userspace uses `SOL_RAW` for raw-ICMP options.

### Ancillary-data macros (`include/linux/socket.h:131-154`)

```text
CMSG_ALIGN(len)       ((len + sizeof(long)-1) & ~(sizeof(long)-1))
CMSG_DATA(cmsg)       ((void *)(cmsg) + sizeof(struct cmsghdr))
CMSG_USER_DATA(cmsg)  ((void __user *)(cmsg) + sizeof(struct cmsghdr))
CMSG_SPACE(len)       (sizeof(struct cmsghdr) + CMSG_ALIGN(len))
CMSG_LEN(len)         (sizeof(struct cmsghdr) + (len))
CMSG_FIRSTHDR(msg)    /* first cmsg or NULL if too small */
CMSG_NXTHDR(msg, c)   /* next cmsg or NULL */
CMSG_OK(msg, c)       /* bounds check */
for_each_cmsghdr(c, msg) /* iterator */
```

### Buffer-lock and rehash bits (`include/uapi/linux/socket.h:29-36`)

```text
SOCK_SNDBUF_LOCK    = 1
SOCK_RCVBUF_LOCK    = 2
SOCK_BUF_LOCK_MASK  = SOCK_SNDBUF_LOCK | SOCK_RCVBUF_LOCK

SOCK_TXREHASH_DEFAULT  = 255
SOCK_TXREHASH_DISABLED = 0
SOCK_TXREHASH_ENABLED  = 1
```

### Other constants

```text
SOMAXCONN = 4096    /* maximum backlog for listen(2) */
IPX_TYPE  = 1
```

### compatibility contract

REQ-1: `_K_SS_MAXSIZE == 128`; `struct __kernel_sockaddr_storage` MUST be 128 bytes; the union with `void *__align` MUST force pointer alignment (8 bytes on LP64, 4 bytes on ILP32).

REQ-2: `sa_family_t` and `__kernel_sa_family_t` MUST be `unsigned short` (u16). Endianness: native (not network byte order).

REQ-3: `struct sockaddr` MUST be 16 bytes total (`sizeof(sa_family) + 14`). `sa_data` MUST be `char[14]`, never wider — userspace code copies `sizeof(struct sockaddr)` into 16-byte buffers.

REQ-4: `BUILD_BUG_ON(sizeof(any_protocol_specific_sockaddr) > sizeof(struct __kernel_sockaddr_storage))` — every per-AF address structure MUST fit in 128 bytes.

REQ-5: `struct user_msghdr` field order, types, and offsets MUST be byte-identical to Linux on every supported arch. Fields: `msg_name` (`void __user *`), `msg_namelen` (`int`), `msg_iov` (`struct iovec __user *`), `msg_iovlen` (`__kernel_size_t`), `msg_control` (`void __user *`), `msg_controllen` (`__kernel_size_t`), `msg_flags` (`unsigned int`).

REQ-6: `struct mmsghdr` MUST be exactly `struct user_msghdr` followed by `unsigned int msg_len` (no padding insertion). On 64-bit it is 56 bytes; on 32-bit it is 32 bytes.

REQ-7: `struct cmsghdr` field order, sizes, and alignment MUST match: `cmsg_len` (`__kernel_size_t`), `cmsg_level` (`int`), `cmsg_type` (`int`). Payload begins at `CMSG_ALIGN(sizeof(struct cmsghdr))` offset from the cmsghdr.

REQ-8: `CMSG_ALIGN(n)` MUST round up to `sizeof(long)` (8 on LP64, 4 on ILP32). `CMSG_SPACE(n)` and `CMSG_LEN(n)` MUST match Linux's macros exactly — userspace iterators (glibc, musl) depend on this.

REQ-9: `__cmsg_nxthdr(ctl, size, cmsg)` MUST return NULL when the computed next header would extend beyond `ctl + size`. The check MUST be `((char *)(__ptr+1) - (char *)__ctl) > __size` (i.e., test the **end** of the next header).

REQ-10: `struct ucred` MUST be `{__u32 pid; __u32 uid; __u32 gid;}`, exactly 12 bytes, naturally aligned.

REQ-11: AF/PF numeric values MUST match the table above. `AF_LOCAL == AF_UNIX == 1`; `AF_ROUTE == AF_NETLINK == 16`. `AF_MAX == PF_MAX == 46`.

REQ-12: `SOCK_*` numeric values MUST match `SOCK_STREAM=1 .. SOCK_PACKET=10`. `SOCK_TYPE_MASK == 0x0f` — the low 4 bits of the `type` argument carry the kind; high bits carry `SOCK_CLOEXEC` / `SOCK_NONBLOCK`.

REQ-13: `SOCK_CLOEXEC == O_CLOEXEC` and `SOCK_NONBLOCK == O_NONBLOCK` (no separate numbering). They are valid bits in the `type` argument to `socket(2)`, `socketpair(2)`, and `accept4(2)`.

REQ-14: `SCM_RIGHTS=1`, `SCM_CREDENTIALS=2`, `SCM_SECURITY=3`, `SCM_PIDFD=4`. All four use `cmsg_level == SOL_SOCKET` (defined in `<asm/socket.h>` as `1`).

REQ-15: `MSG_*` flag values MUST match Linux exactly. Internal-only sendmsg flags (`MSG_SPLICE_PAGES`, `MSG_SENDPAGE_NOPOLICY`, `MSG_SENDPAGE_DECRYPTED`) MUST be masked from any userspace-supplied flags word on syscall entry.

REQ-16: `MSG_CMSG_COMPAT` MUST be `0x80000000` only on `CONFIG_COMPAT` kernels; otherwise it MUST be `0` (constant-fold the mask).

REQ-17: `SOL_*` numeric values MUST match the table. `SOL_SOCKET` itself is defined in `<asm/socket.h>` (typically `1` on most arches, `0xffff` on alpha, sparc, parisc); Rookery follows the same per-arch override.

REQ-18: `SOMAXCONN == 4096`. Userspace `listen(2, backlog)` clamps `backlog` to `SOMAXCONN`.

REQ-19: Buffer-lock bits MUST match: `SOCK_SNDBUF_LOCK=1`, `SOCK_RCVBUF_LOCK=2`, mask = 3.

REQ-20: `SOCK_TXREHASH_DEFAULT=255`, `SOCK_TXREHASH_DISABLED=0`, `SOCK_TXREHASH_ENABLED=1`.

REQ-21: `socket(family, type, protocol)` with `family >= AF_MAX` MUST return `-EAFNOSUPPORT`. With `type & ~(SOCK_TYPE_MASK | SOCK_CLOEXEC | SOCK_NONBLOCK)` non-zero MUST return `-EINVAL`. With unknown `(family, type, protocol)` triple MUST return `-EPROTONOSUPPORT`.

REQ-22: `accept4(fd, NULL, NULL, flags)` with `flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK)` non-zero MUST return `-EINVAL`.

REQ-23: `recvmsg(2)` MUST set `MSG_CTRUNC` in `msg_flags` when ancillary data did not fit in `msg_controllen`. MUST set `MSG_TRUNC` when normal data was truncated and `MSG_PEEK` was not used (or `MSG_TRUNC` was requested with peek).

REQ-24: `recvmsg(2)` with `MSG_CMSG_CLOEXEC` MUST install received SCM_RIGHTS fds with `FD_CLOEXEC` set. This is the only safe way to receive descriptors from an untrusted peer in a multi-threaded process.

REQ-25: `SCM_CREDENTIALS` ancillary data on AF_UNIX MUST validate that the sender either has `CAP_SYS_ADMIN`, or `ucred.pid == sender_pid`, `ucred.uid ∈ {real,eff,saved} uid`, `ucred.gid ∈ {real,eff,saved} gid`.

REQ-26: `SCM_PIDFD` MUST be receive-only — `sendmsg(2)` with `SCM_PIDFD` MUST return `-EINVAL` (the kernel synthesizes the pidfd on `recvmsg`).

REQ-27: `bind(fd, addr, addrlen)` MUST reject `addrlen > sizeof(struct __kernel_sockaddr_storage)` with `-EINVAL` (via `move_addr_to_kernel`).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `kernel_sockaddr_storage_is_128` | INVARIANT | `core::mem::size_of::<KernelSockaddrStorage>() == 128`. |
| `kernel_sockaddr_storage_pointer_aligned` | INVARIANT | `core::mem::align_of::<KernelSockaddrStorage>() == core::mem::align_of::<*const u8>()`. |
| `sockaddr_is_16` | INVARIANT | `sizeof::<Sockaddr>() == 16`. |
| `ucred_is_12` | INVARIANT | `sizeof::<Ucred>() == 12`. |
| `cmsg_align_long` | INVARIANT | per-arch: `CMSG_ALIGN(n) % sizeof::<c_long>() == 0` and `CMSG_ALIGN(n) >= n`. |
| `cmsg_nxthdr_bounded` | INVARIANT | per-call: returned ptr lies within [ctl, ctl+size) or is None. |
| `move_addr_rejects_oversize` | INVARIANT | per-call: ulen > 128 ⟹ EINVAL. |
| `sys_socket_rejects_high_type_bits` | INVARIANT | per-call: type & ~(0xf|CLOEXEC|NONBLOCK) ⟹ EINVAL. |
| `sys_socket_rejects_high_family` | INVARIANT | per-call: family >= AF_MAX ⟹ EAFNOSUPPORT. |
| `msg_internal_flags_masked` | INVARIANT | per-sendmsg: MSG_INTERNAL_SENDMSG_FLAGS cleared in user-flags. |
| `scm_pidfd_send_rejected` | INVARIANT | per-sendmsg: cmsg_type == SCM_PIDFD ⟹ EINVAL. |
| `somaxconn_clamped` | INVARIANT | per-listen: effective backlog ≤ 4096. |

### Layer 2: TLA+

`uapi/headers/socket.tla`:
- Per-`socket(2)` open → `bind(2)` → `listen(2)` → `accept4(2)` → `recvmsg(2)` → `close(2)` lifecycle.
- Per-`SCM_RIGHTS` send/recv: fd table monotonicity.
- Properties:
  - `safety_no_fd_leak_on_cloexec_fork` — per-MSG_CMSG_CLOEXEC: fd not inherited across execve.
  - `safety_ucred_validation` — per-SCM_CREDENTIALS: forged ucred without CAP_SYS_ADMIN ⟹ EPERM.
  - `safety_cmsg_iter_bounded` — per-`for_each_cmsghdr`: never reads past `msg_controllen`.
  - `safety_AF_MAX_enforced` — per-socket(): family ≥ 46 ⟹ EAFNOSUPPORT.
  - `liveness_listen_eventually_accepts` — per-listening socket: incoming SYN accepted (modulo backlog).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `KernelSockaddrStorage` layout post: 128 bytes, pointer aligned | `KernelSockaddrStorage` |
| `UserMsghdr` field offsets per-arch | `UserMsghdr` |
| `Cmsghdr` field offsets and alignment | `Cmsghdr` |
| `Cmsg::nxthdr` post: out-pointer ∈ [ctl, ctl+size) | `Cmsg::nxthdr` |
| `Socket::move_addr_to_kernel` post: ulen ≤ 128 ∧ AF_UNSPEC on ulen==0 | `Socket::move_addr_to_kernel` |
| `SysSocket::sys_socket` post: returned fd has CLOEXEC iff requested | `SysSocket::sys_socket` |
| `SysSocket::sys_recvmsg` post: MSG_CTRUNC set ⟺ ancillary truncated | `SysSocket::sys_recvmsg` |

### Layer 4: Verus/Creusot functional

`Per socket(2) → AF/SOCK validation → net_families[family].create → fd install with CLOEXEC/NONBLOCK` semantic equivalence: per-`Documentation/networking/` and per-glibc `<sys/socket.h>` headers. `Per recvmsg(2) → cmsg iteration → SCM dispatch (RIGHTS / CREDENTIALS / SECURITY / PIDFD) → MSG_CTRUNC/TRUNC flagging` semantic equivalence: per-POSIX 1003.1g §5.5.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

Socket UAPI reinforcement:

- **AF/PF range check at syscall entry** — defense against per-OOB net_families[] indexing.
- **`type` argument: split SOCK_TYPE_MASK + CLOEXEC + NONBLOCK; reject other bits** — defense against per-future-bit ambiguity.
- **`move_addr_to_kernel` bounded by `_K_SS_MAXSIZE`** — defense against per-oversized-addr stack overflow.
- **CMSG iteration uses end-of-next bounds check** — defense against per-iteration OOB read.
- **`MSG_INTERNAL_SENDMSG_FLAGS` masked from userspace flags** — defense against per-flag-injection into sendpage path.
- **`SCM_PIDFD` send-side rejection** — defense against per-PID-spoofing.
- **`SCM_CREDENTIALS` validated against CAP_SYS_ADMIN or sender identity** — defense against per-peer-impersonation.
- **`MSG_CMSG_CLOEXEC` recommended for SCM_RIGHTS recv in threaded processes** — defense against per-fork-race fd leak.
- **`SOMAXCONN` clamp** — defense against per-SYN-flood backlog blow-up.
- **Buffer-lock bits respected** — defense against per-setsockopt budget bypass.

### grsecurity/pax-style reinforcement

- **PaX UDEREF/USERCOPY** on every `copy_from_user(msg_name, sa_data, cmsg payload)` — defense against per-kernel-pointer dereference of user-controlled pointers, and per-stack-buffer-overflow via oversized addrlen.
- **PAX_RANDKSTACK on syscall entry** — randomize kernel-stack offset per `socket()`, `recvmsg()`, `accept4()` entry so heap-grooming for sockaddr-on-stack disclosure becomes statistically infeasible.
- **GRKERNSEC_PROC restrictions** on `/proc/<pid>/net/{tcp,tcp6,udp,udp6,unix,packet,netlink}` — defense against per-unprivileged enumeration of remote endpoints, source-port randomization leaks, and inode-cookie correlation across processes.
- **SOCK_CLOEXEC mandatory on suid binaries** — kernel returns `-EINVAL` for `socket(2)` invocations from suid/sgid processes where `SOCK_CLOEXEC` is missing from `type`. Closes the historic privileged-fd-leak-via-exec(non-suid-child) class of bug.
- **`AF_PACKET` / `SOCK_RAW` require explicit CAP_NET_RAW even with `setuid(0)`-derived euid** — defense against per-suid-helper raw-socket abuse.
- **GRKERNSEC_SOCKET_ALL / _CLIENT / _SERVER** — gid-based gates on `socket(2)` per family (e.g., disallow AF_NETLINK to non-gid-X processes; deny AF_INET client connect to non-gid-Y).
- **GRKERNSEC_RESLOG** on every `socket(2)` and `bind(2)` failure — full audit trail of unauthorized network access attempts.
- **`AF_NETLINK` source-port pinning** — userspace MUST NOT spoof `nl_pid`; the kernel overwrites with the binding process's tgid for non-root.
- **`SCM_RIGHTS` recursion depth bounded** — defense against per-AF_UNIX-cycle DoS where descriptors-passing-descriptors causes infinite mutex chains (Linux CVE-2013-7446 / -2016-2384 family).
- **`SCM_CREDENTIALS` cgroup-scoped** — defense against per-cgroup-namespace-escape via credential forging from a less-privileged container.
- **`MSG_ERRQUEUE` ratelimited per-uid** — defense against per-error-queue flooding causing unbounded memory growth.
- **`AF_ALG` requires `CAP_SYS_ADMIN`** under grsec policy (upstream allows by default) — defense against per-unprivileged crypto-instruction abuse.

