# Tier-5 UAPI: include/uapi/linux/netlink.h — Netlink ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/netlink.h (~383 lines)
-->

## Summary

The netlink UAPI is the syscall-edge contract for `AF_NETLINK` sockets — the kernel/userspace message bus used by `rtnetlink`, `nfnetlink`, `genetlink`, `audit`, `uevent`, `xfrm`, `selinux`, `inet_diag`, `kobject_uevent`, and roughly two dozen other subsystems. It defines:

- **Per-AF netlink address** `struct sockaddr_nl` (`nl_family`, `nl_pad`, `nl_pid`, `nl_groups`) — 12 bytes, used for `bind(2)` / `connect(2)` / `recvfrom(2)` / `sendto(2)`.
- **Fixed-format message header** `struct nlmsghdr` (`nlmsg_len`, `nlmsg_type`, `nlmsg_flags`, `nlmsg_seq`, `nlmsg_pid`) — 16 bytes, mandatory prefix of every netlink message.
- **Netlink protocol family numbers** `NETLINK_ROUTE`..`NETLINK_SMC` (0..22) carried as the `protocol` argument to `socket(AF_NETLINK, ...)`.
- **Generic message-type values** `NLMSG_NOOP`/`ERROR`/`DONE`/`OVERRUN` (0x1..0x4) and `NLMSG_MIN_TYPE = 0x10` (subsystem-defined types start here).
- **Flags word `nlmsg_flags`**: base bits `NLM_F_REQUEST`/`MULTI`/`ACK`/`ECHO`/`DUMP_INTR`/`DUMP_FILTERED`, GET-request modifiers `NLM_F_ROOT`/`MATCH`/`ATOMIC` (and the convenience `NLM_F_DUMP = ROOT|MATCH`), NEW-request modifiers `NLM_F_REPLACE`/`EXCL`/`CREATE`/`APPEND`, DELETE modifiers `NLM_F_NONREC`/`BULK`, ACK-reply bits `NLM_F_CAPPED`/`ACK_TLVS`.
- **Error / extack** `struct nlmsgerr` (negative `error`, embedded original `struct nlmsghdr msg`, optional `NLMSGERR_ATTR_*` TLVs) — emitted in reply to a request bearing `NLM_F_ACK` or on failure.
- **TLV attributes** `struct nlattr` (`nla_len`, `nla_type`) plus the in-type flag bits `NLA_F_NESTED` (bit 15) and `NLA_F_NET_BYTEORDER` (bit 14), masked off via `NLA_TYPE_MASK`.
- **Alignment macros** `NLMSG_ALIGNTO = 4`, `NLMSG_ALIGN`/`HDRLEN`/`LENGTH`/`SPACE`/`DATA`/`NEXT`/`OK`/`PAYLOAD`, and the parallel `NLA_ALIGNTO`/`NLA_ALIGN`/`NLA_HDRLEN`.
- **Socket-option commands** `NETLINK_ADD_MEMBERSHIP`/`DROP_MEMBERSHIP`/`PKTINFO`/`BROADCAST_ERROR`/`NO_ENOBUFS`/`LISTEN_ALL_NSID`/`LIST_MEMBERSHIPS`/`CAP_ACK`/`EXT_ACK`/`GET_STRICT_CHK` (1..12).
- **Policy descriptors** `enum netlink_attribute_type` (FLAG/U8/U16/U32/U64/S8/S16/S32/S64/BINARY/STRING/NUL_STRING/NESTED/NESTED_ARRAY/BITFIELD32/SINT/UINT) and `enum netlink_policy_type_attr` (MIN/MAX value, MIN/MAX length, POLICY_IDX/MAXTYPE, BITFIELD32_MASK, MASK, PAD) — exposed to userspace via `genetlink` policy dumps.

Critical for: `iproute2` (`ip`, `tc`, `bridge`, `ss`), `iptables-nft`/`nftables`, `wpa_supplicant`, `systemd-udevd`, `audit`, every container-runtime network-namespace setup, every kernel-bypass / DPDK-control-plane userspace driver. Carries CAP_NET_ADMIN-gated configuration and CAP_AUDIT_CONTROL-gated audit messages.

This Tier-5 covers `include/uapi/linux/netlink.h` (~383 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sockaddr_nl` | per-AF_NETLINK socket addr | `SockaddrNl` |
| `struct nlmsghdr` | per-message fixed header | `Nlmsghdr` |
| `struct nlmsgerr` | per-error/ack reply | `Nlmsgerr` |
| `struct nlattr` | per-attribute TLV header | `Nlattr` |
| `struct nl_pktinfo` | per-PKTINFO ancillary group | `NlPktinfo` |
| `struct nl_mmap_req` | per-legacy mmap req | `NlMmapReq` |
| `struct nl_mmap_hdr` | per-legacy mmap frame | `NlMmapHdr` |
| `struct nla_bitfield32` | per-bitfield32 attribute payload | `NlaBitfield32` |
| `enum nl_mmap_status` | per-frame status | `NlMmapStatus` |
| `enum nlmsgerr_attrs` | per-extack TLV ids | `NlmsgerrAttrs` |
| `enum netlink_attribute_type` | per-policy attr-type | `NetlinkAttributeType` |
| `enum netlink_policy_type_attr` | per-policy descriptor | `NetlinkPolicyTypeAttr` |
| `NLMSG_ALIGN`/`HDRLEN`/`LENGTH`/`SPACE`/`DATA`/`NEXT`/`OK`/`PAYLOAD` | per-msg iter macros | `Nlmsg::{align,hdrlen,length,space,data,next,ok,payload}` |
| `NLA_ALIGN`/`NLA_HDRLEN` | per-attr iter macros | `Nla::{align,hdrlen}` |
| `NETLINK_ADD_MEMBERSHIP` ... `NETLINK_GET_STRICT_CHK` | per-sockopt cmd | `NetlinkSockoptCmd` |
| `__sys_socket(AF_NETLINK, ...)` | per-socket(2) | `SysSocket::sys_socket` |
| `netlink_bind` | per-bind(2) AF_NETLINK | `Netlink::bind` |
| `netlink_sendmsg` / `netlink_recvmsg` | per-IO | `Netlink::{sendmsg,recvmsg}` |
| `netlink_autobind` | per-PID allocation | `Netlink::autobind` |
| `netlink_dump` | per-NLM_F_DUMP iter | `Netlink::dump` |
| `netlink_ack` | per-NLM_F_ACK reply | `Netlink::ack` |
| `netlink_extack` | per-NETLINK_EXT_ACK TLV emit | `Netlink::extack` |

## ABI surface (constants + structs)

### Netlink protocol-family numbers (`netlink.h:9-33`)

```text
NETLINK_ROUTE          =  0   /* rtnetlink: link/addr/route/neigh */
NETLINK_UNUSED         =  1   /* reserved-unused */
NETLINK_USERSOCK       =  2   /* userspace-to-userspace bus */
NETLINK_FIREWALL       =  3   /* obsolete ip_queue (kept for ABI) */
NETLINK_SOCK_DIAG      =  4   /* sock-diag (ss(8)) */
NETLINK_NFLOG          =  5   /* netfilter/iptables ULOG */
NETLINK_XFRM           =  6   /* IPsec PFKEY-2 successor */
NETLINK_SELINUX        =  7   /* SELinux events */
NETLINK_ISCSI          =  8   /* Open-iSCSI */
NETLINK_AUDIT          =  9   /* audit subsystem */
NETLINK_FIB_LOOKUP     = 10   /* FIB lookup helper */
NETLINK_CONNECTOR      = 11   /* connector (proc events) */
NETLINK_NETFILTER      = 12   /* nfnetlink: nftables/conntrack/queue */
NETLINK_IP6_FW         = 13   /* obsolete IPv6 firewall */
NETLINK_DNRTMSG        = 14   /* DECnet (obsolete) */
NETLINK_KOBJECT_UEVENT = 15   /* kobject uevents (udev) */
NETLINK_GENERIC        = 16   /* genetlink dispatch family */
/* 17 reserved for NETLINK_DM */
NETLINK_SCSITRANSPORT  = 18   /* SCSI transport events */
NETLINK_ECRYPTFS       = 19   /* eCryptfs */
NETLINK_RDMA           = 20   /* RDMA subsystem */
NETLINK_CRYPTO         = 21   /* AF_ALG / crypto */
NETLINK_SMC            = 22   /* SMC monitoring */
NETLINK_INET_DIAG      = NETLINK_SOCK_DIAG  /* compat alias */
MAX_LINKS              = 32
```

### `struct sockaddr_nl` (`netlink.h:37-42`)

```text
struct sockaddr_nl {
    __kernel_sa_family_t nl_family;  /* AF_NETLINK == 16; u16 */
    unsigned short       nl_pad;     /* MUST be zero */
    __u32                nl_pid;     /* port ID (== tgid for autobind, 0 for kernel) */
    __u32                nl_groups;  /* 32-bit multicast group mask */
};
```

Size: 12 bytes (LP64 / ILP32). `nl_pid == 0` addresses the kernel; positive values address a specific userspace port. `nl_groups` is a bitmask of multicast-group memberships used at send-side broadcast.

### `struct nlmsghdr` (`netlink.h:52-58`)

```text
struct nlmsghdr {
    __u32 nlmsg_len;    /* length including 16-byte header */
    __u16 nlmsg_type;   /* >= 0x10 subsystem-defined; < 0x10 control */
    __u16 nlmsg_flags;  /* see NLM_F_* */
    __u32 nlmsg_seq;    /* sender-chosen sequence (echoed in ack) */
    __u32 nlmsg_pid;    /* sender's nl_pid (kernel sets to source on rx) */
};
```

Size: 16 bytes. Followed by `NLMSG_ALIGN(nlmsg_len) - NLMSG_HDRLEN` bytes of payload (subsystem-specific family-header + `struct nlattr` TLVs).

### Flag bits `nlmsg_flags` (`netlink.h:62-87`)

```text
/* Base */
NLM_F_REQUEST       = 0x01   /* this is a request message */
NLM_F_MULTI         = 0x02   /* multipart; terminate with NLMSG_DONE */
NLM_F_ACK           = 0x04   /* request explicit ack reply */
NLM_F_ECHO          = 0x08   /* echo back to sender */
NLM_F_DUMP_INTR     = 0x10   /* dump was inconsistent (seq changed mid-dump) */
NLM_F_DUMP_FILTERED = 0x20   /* dump was filtered as requested */

/* GET-request modifiers (mutex with NEW/DELETE bits at same offsets) */
NLM_F_ROOT    = 0x100   /* specify tree-root */
NLM_F_MATCH   = 0x200   /* return all matching */
NLM_F_ATOMIC  = 0x400   /* atomic GET (deprecated) */
NLM_F_DUMP    = (NLM_F_ROOT | NLM_F_MATCH)   /* convenience: full dump */

/* NEW-request modifiers */
NLM_F_REPLACE = 0x100   /* override existing */
NLM_F_EXCL    = 0x200   /* do not touch if existing */
NLM_F_CREATE  = 0x400   /* create if not exists */
NLM_F_APPEND  = 0x800   /* append to list */

/* DELETE-request modifiers */
NLM_F_NONREC  = 0x100   /* non-recursive delete */
NLM_F_BULK    = 0x200   /* bulk delete */

/* ACK reply bits */
NLM_F_CAPPED   = 0x100  /* ack was capped (original msg not echoed) */
NLM_F_ACK_TLVS = 0x200  /* NETLINK_EXT_ACK TLVs follow */
```

The high-byte bits (`0x100`/`0x200`/`0x400`/`0x800`) are **context-dependent**: their meaning depends on the operation class (GET vs NEW vs DELETE vs ACK) which the subsystem dispatcher determines from `nlmsg_type`. The low-byte bits (`0x01`/`0x02`/`0x04`/`0x08`/`0x10`/`0x20`) are unambiguous.

Conventional combinations:

```text
4.4BSD ADD     = NLM_F_CREATE | NLM_F_EXCL
4.4BSD CHANGE  = NLM_F_REPLACE
true CHANGE    = NLM_F_CREATE | NLM_F_REPLACE
Append         = NLM_F_CREATE
Check          = NLM_F_EXCL
```

### Generic message types (`netlink.h:112-117`)

```text
NLMSG_NOOP      = 0x1   /* no-op (filler) */
NLMSG_ERROR     = 0x2   /* payload is struct nlmsgerr */
NLMSG_DONE      = 0x3   /* end of NLM_F_MULTI dump */
NLMSG_OVERRUN   = 0x4   /* data was lost */
NLMSG_MIN_TYPE  = 0x10  /* subsystem-defined types start here */
```

### `struct nlmsgerr` (`netlink.h:119-131`)

```text
struct nlmsgerr {
    int             error;   /* negative errno; 0 means ack-only */
    struct nlmsghdr msg;     /* copy of original header (unless NETLINK_CAP_ACK) */
    /* optional NLMSGERR_ATTR_* TLVs follow if NETLINK_EXT_ACK was set */
};
```

### `enum nlmsgerr_attrs` (`netlink.h:150-161`)

```text
NLMSGERR_ATTR_UNUSED    = 0
NLMSGERR_ATTR_MSG       = 1   /* NUL-terminated diagnostic string */
NLMSGERR_ATTR_OFFS      = 2   /* u32 offset of bad attr from header start */
NLMSGERR_ATTR_COOKIE    = 3   /* binary subsystem cookie (on success) */
NLMSGERR_ATTR_POLICY    = 4   /* policy for a rejected attribute */
NLMSGERR_ATTR_MISS_TYPE = 5   /* type of missing required attribute */
NLMSGERR_ATTR_MISS_NEST = 6   /* offset of nest where attr was missing */
NLMSGERR_ATTR_MAX       = __NLMSGERR_ATTR_MAX - 1
```

### Sockopt commands (`netlink.h:163-176`)

```text
NETLINK_ADD_MEMBERSHIP   = 1   /* join multicast group */
NETLINK_DROP_MEMBERSHIP  = 2   /* leave multicast group */
NETLINK_PKTINFO          = 3   /* deliver group id in cmsg */
NETLINK_BROADCAST_ERROR  = 4   /* report broadcast errors */
NETLINK_NO_ENOBUFS       = 5   /* drop quietly on ENOBUFS */
NETLINK_RX_RING          = 6   /* legacy mmap rx ring (userspace only) */
NETLINK_TX_RING          = 7   /* legacy mmap tx ring (userspace only) */
NETLINK_LISTEN_ALL_NSID  = 8   /* receive from all netns */
NETLINK_LIST_MEMBERSHIPS = 9   /* dump joined groups */
NETLINK_CAP_ACK          = 10  /* omit original msg from ack */
NETLINK_EXT_ACK          = 11  /* request extack TLVs in errors */
NETLINK_GET_STRICT_CHK   = 12  /* enable strict input validation */
```

### `struct nlattr` and attribute framing (`netlink.h:229-250`)

```text
struct nlattr {
    __u16 nla_len;    /* including 4-byte header */
    __u16 nla_type;   /* low 14 bits: subsystem-defined */
                       /* bit 14: NLA_F_NET_BYTEORDER */
                       /* bit 15: NLA_F_NESTED */
};

NLA_F_NESTED         = 1 << 15
NLA_F_NET_BYTEORDER  = 1 << 14
NLA_TYPE_MASK        = ~(NLA_F_NESTED | NLA_F_NET_BYTEORDER)   /* 0x3fff */

NLA_ALIGNTO  = 4
NLA_ALIGN(n) = ((n) + 3) & ~3
NLA_HDRLEN   = NLA_ALIGN(sizeof(struct nlattr))   /* 4 */
```

Attribute layout: `[ struct nlattr (4 bytes) | payload | pad-to-4 ]`. Nested attributes (`NLA_F_NESTED`) carry sub-attributes inside the payload; `NLA_F_NET_BYTEORDER` indicates the payload is big-endian (the two flag bits are mutually exclusive).

### Alignment macros (`netlink.h:98-110`)

```text
NLMSG_ALIGNTO          = 4
NLMSG_ALIGN(n)         = ((n) + 3) & ~3
NLMSG_HDRLEN           = NLMSG_ALIGN(sizeof(struct nlmsghdr))   /* 16 */
NLMSG_LENGTH(n)        = (n) + NLMSG_HDRLEN
NLMSG_SPACE(n)         = NLMSG_ALIGN(NLMSG_LENGTH(n))
NLMSG_DATA(nlh)        = (void *)((char *)(nlh) + NLMSG_HDRLEN)
NLMSG_NEXT(nlh, len)   = step to next nlmsg, decrementing len
NLMSG_OK(nlh, len)     = len >= sizeof(struct nlmsghdr) &&
                          nlh->nlmsg_len >= sizeof(struct nlmsghdr) &&
                          nlh->nlmsg_len <= len
NLMSG_PAYLOAD(nlh, n)  = nlh->nlmsg_len - NLMSG_SPACE(n)
```

### `struct nla_bitfield32` (`netlink.h:265-268`)

```text
struct nla_bitfield32 {
    __u32 value;     /* which bits to set/clear */
    __u32 selector;  /* which bits this command targets */
};
```

### `enum netlink_attribute_type` (`netlink.h:304-330`)

```text
NL_ATTR_TYPE_INVALID, FLAG,
U8, U16, U32, U64,
S8, S16, S32, S64,
BINARY, STRING, NUL_STRING,
NESTED, NESTED_ARRAY,
BITFIELD32,
SINT, UINT
```

### `enum netlink_policy_type_attr` (`netlink.h:363-381`)

```text
NL_POLICY_TYPE_ATTR_UNSPEC, TYPE,
MIN_VALUE_S, MAX_VALUE_S, MIN_VALUE_U, MAX_VALUE_U,
MIN_LENGTH, MAX_LENGTH,
POLICY_IDX, POLICY_MAXTYPE,
BITFIELD32_MASK, PAD, MASK
```

### Connection state (`netlink.h:215-218`)

```text
NETLINK_UNCONNECTED = 0
NETLINK_CONNECTED   = 1
```

### Legacy mmap ring frame states (`netlink.h:200-206`, userspace-visible only)

```text
NL_MMAP_STATUS_UNUSED, RESERVED, VALID, COPY, SKIP
```

## Compatibility contract

REQ-1: `struct sockaddr_nl` MUST be exactly 12 bytes: `nl_family` (u16) + `nl_pad` (u16) + `nl_pid` (u32) + `nl_groups` (u32). Field order and offsets MUST match Linux on every supported arch. Endianness is native (not network byte order).

REQ-2: `nl_family` MUST equal `AF_NETLINK == 16`. `bind(2)` MUST reject `nl_family != AF_NETLINK` with `-EINVAL`. `nl_pad` MUST be silently ignored (not validated) for forward compatibility.

REQ-3: `struct nlmsghdr` MUST be exactly 16 bytes: `nlmsg_len` (u32, includes header) + `nlmsg_type` (u16) + `nlmsg_flags` (u16) + `nlmsg_seq` (u32) + `nlmsg_pid` (u32). All numeric fields native byte order.

REQ-4: `NLMSG_ALIGNTO == 4` on every arch. `NLMSG_HDRLEN == 16`. Payload MUST be padded to a 4-byte boundary.

REQ-5: `NLMSG_OK(nlh, len)` MUST verify: `len >= sizeof(struct nlmsghdr)` AND `nlh->nlmsg_len >= sizeof(struct nlmsghdr)` AND `nlh->nlmsg_len <= len`. All three conditions are necessary — dropping any one is a known parsing-OOB class of bug.

REQ-6: `NLMSG_NEXT(nlh, len)` MUST advance by `NLMSG_ALIGN(nlh->nlmsg_len)` and MUST decrement `len` by the same amount. Receiver MUST re-validate `NLMSG_OK` before dereferencing.

REQ-7: `NETLINK_ROUTE..NETLINK_SMC` numeric values MUST match Linux exactly (0,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,18,19,20,21,22). `NETLINK_INET_DIAG` is an alias for `NETLINK_SOCK_DIAG == 4`. `MAX_LINKS == 32` caps `socket(AF_NETLINK, type, protocol)` to `protocol < 32`.

REQ-8: `nlmsg_type < NLMSG_MIN_TYPE (0x10)` is reserved for generic control messages (NOOP/ERROR/DONE/OVERRUN). Subsystem dispatchers MUST treat unknown control types as silent no-ops (or `-EOPNOTSUPP`); they MUST NOT pass them to per-subsystem handlers.

REQ-9: `NLM_F_REQUEST` MUST be set on every userspace-originated request. Kernel MUST drop received messages lacking `NLM_F_REQUEST` (silently or with `-EINVAL` depending on subsystem) — never treat a non-request as authoritative.

REQ-10: `NLM_F_MULTI` MUST be set on every part of a multipart reply, including the terminating `NLMSG_DONE`. The terminating `NLMSG_DONE` has `nlmsg_type == NLMSG_DONE` (0x3), payload = single `int` (overall dump errno; 0 on success).

REQ-11: `NLM_F_ACK` requests MUST receive exactly one `NLMSG_ERROR` reply with `error == 0` on success or `error == -<errno>` on failure. With `NETLINK_CAP_ACK` set, the original `struct nlmsghdr msg` is omitted (zeroed) and `NLM_F_CAPPED` is set in `nlmsg_flags`.

REQ-12: `NLM_F_ECHO` MUST cause the kernel to multicast the resulting state-change notification back to the originating socket (in addition to the regular group). Used by `iproute2` to confirm `RTM_NEW*` operations.

REQ-13: `NLM_F_DUMP_INTR` MUST be set in any dump message emitted after a relevant sequence-counter (e.g., `rtnl_dump_seq`) changed mid-dump. Userspace MUST restart the dump on observing this flag.

REQ-14: `NLM_F_ROOT | NLM_F_MATCH == NLM_F_DUMP` is the canonical full-dump request. `NLM_F_ATOMIC` is honored only with `CAP_NET_ADMIN` and is best-effort.

REQ-15: `NLM_F_REPLACE`, `NLM_F_EXCL`, `NLM_F_CREATE`, `NLM_F_APPEND` MUST be honored per the BSD-derived table above. `EEXIST` MUST be returned when `EXCL` is set and the object exists; `ENOENT` MUST be returned when `REPLACE` is set without `CREATE` and the object does not exist.

REQ-16: `struct nlmsgerr` MUST be `int error; struct nlmsghdr msg;` — 20 bytes minimum on every arch. Trailing extack TLVs (when `NETLINK_EXT_ACK` set) follow `msg` aligned to `NLMSG_ALIGN`.

REQ-17: `struct nlattr` MUST be exactly 4 bytes: `nla_len` (u16) + `nla_type` (u16). Layout `[ nlattr | payload | pad-to-4 ]`. `nla_len` includes the 4-byte header.

REQ-18: `NLA_F_NESTED (bit 15)` and `NLA_F_NET_BYTEORDER (bit 14)` MUST be mutually exclusive. `NLA_TYPE_MASK == 0x3fff` MUST be applied before comparing `nla_type` against subsystem attribute IDs.

REQ-19: `NLA_ALIGNTO == 4`. `NLA_ALIGN(n) == (n + 3) & ~3`. `NLA_HDRLEN == 4`.

REQ-20: `nl_pid` autobind: if userspace `bind(2)` with `nl_pid == 0`, the kernel MUST assign `nl_pid = current->tgid` (or another collision-free value with the same low bits as tgid). Userspace MUST NOT supply `nl_pid != 0 && nl_pid != current->tgid` without `CAP_NET_ADMIN` — the kernel MUST overwrite the supplied value or return `-EPERM`.

REQ-21: `nl_groups != 0` on `bind(2)` requires `CAP_NET_ADMIN` (or per-protocol capability such as `CAP_AUDIT_CONTROL` for `NETLINK_AUDIT`). Membership add via `NETLINK_ADD_MEMBERSHIP` is gated identically.

REQ-22: `NETLINK_EXT_ACK` enables `NLMSGERR_ATTR_*` TLVs in error replies. `NETLINK_CAP_ACK` suppresses the original-message echo. `NETLINK_GET_STRICT_CHK` enables strict validation (reject unknown attributes, enforce attribute policies).

REQ-23: `NETLINK_LISTEN_ALL_NSID` requires `CAP_NET_ADMIN` in the initial netns. Receiver gets `nl_pktinfo` cmsg per message identifying source netns.

REQ-24: `recvmsg(2)` from a netlink socket MUST set `MSG_TRUNC` in `msg_flags` if the message did not fit in `msg.msg_iov` (and `MSG_PEEK` was not used). `NETLINK_NO_ENOBUFS` suppresses `-ENOBUFS` on overrun, replacing it with silent drops.

REQ-25: `nlmsg_pid == 0` on a received message identifies a kernel-originated multicast (never spoofable from userspace — kernel sets this field on send).

## Acceptance Criteria

- [ ] AC-1: `sizeof(struct sockaddr_nl) == 12`; `sizeof(struct nlmsghdr) == 16`; `sizeof(struct nlattr) == 4`; `sizeof(struct nlmsgerr) >= 20`.
- [ ] AC-2: `NLMSG_HDRLEN == 16`; `NLA_HDRLEN == 4`; `NLMSG_ALIGNTO == NLA_ALIGNTO == 4`.
- [ ] AC-3: `NLMSG_ALIGN(13) == 16`; `NLA_ALIGN(5) == 8`.
- [ ] AC-4: `NLMSG_OK` returns false on truncated header, on `nlmsg_len < 16`, and on `nlmsg_len > len`.
- [ ] AC-5: `NLM_F_DUMP == 0x300` (`ROOT | MATCH`); `NLA_F_NESTED == 0x8000`; `NLA_F_NET_BYTEORDER == 0x4000`; `NLA_TYPE_MASK == 0x3fff`.
- [ ] AC-6: `socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE)` returns fd; `socket(AF_NETLINK, SOCK_RAW, MAX_LINKS)` returns `-EPROTONOSUPPORT`.
- [ ] AC-7: `bind` with `nl_pid == 0` autobinds; `nl_pid == 12345` without `CAP_NET_ADMIN` is overwritten with caller tgid (or returns `-EACCES`).
- [ ] AC-8: `bind` with `nl_groups != 0` without `CAP_NET_ADMIN` returns `-EPERM`.
- [ ] AC-9: `sendmsg(NLMSG_NOOP, NLM_F_REQUEST | NLM_F_ACK)` produces one `NLMSG_ERROR` reply with `error == 0`.
- [ ] AC-10: A request with `NLM_F_ACK` but invalid attrs returns `NLMSG_ERROR` with `error < 0` and (if `NETLINK_EXT_ACK` set) `NLMSGERR_ATTR_MSG` + `NLMSGERR_ATTR_OFFS` TLVs.
- [ ] AC-11: Multipart dump terminates with `NLMSG_DONE` carrying `NLM_F_MULTI`.
- [ ] AC-12: Mid-dump rtnl_seq bump causes subsequent parts to bear `NLM_F_DUMP_INTR`.
- [ ] AC-13: `NLM_F_CREATE | NLM_F_EXCL` on existing object returns `-EEXIST`; without `EXCL` it replaces (when `REPLACE` set) or is `-EEXIST` (when neither set, per subsystem).
- [ ] AC-14: `NETLINK_CAP_ACK` suppresses original `nlmsghdr` in ack reply; `NLM_F_CAPPED` set.
- [ ] AC-15: `NETLINK_NO_ENOBUFS` makes overruns silent (no `-ENOBUFS`); without it, `recvmsg` returns `-ENOBUFS` on overrun.
- [ ] AC-16: `NETLINK_LISTEN_ALL_NSID` without `CAP_NET_ADMIN` returns `-EPERM`.
- [ ] AC-17: Attribute with `NLA_F_NESTED | NLA_F_NET_BYTEORDER` set simultaneously is rejected by strict parsers (`NETLINK_GET_STRICT_CHK`).
- [ ] AC-18: Received `nlmsg_pid == 0` is interpreted as kernel-originated; userspace cannot spoof it (kernel rewrites on send).

## Architecture

```
pub struct SockaddrNl {
    pub nl_family: u16,    // AF_NETLINK == 16
    pub nl_pad:    u16,    // zero
    pub nl_pid:    u32,    // port id
    pub nl_groups: u32,    // multicast mask
}

pub struct Nlmsghdr {
    pub nlmsg_len:   u32,
    pub nlmsg_type:  u16,
    pub nlmsg_flags: u16,
    pub nlmsg_seq:   u32,
    pub nlmsg_pid:   u32,
}

pub struct Nlmsgerr {
    pub error: i32,
    pub msg:   Nlmsghdr,
    /* extack TLVs follow when NETLINK_EXT_ACK */
}

pub struct Nlattr {
    pub nla_len:  u16,
    pub nla_type: u16,
}

pub struct NlaBitfield32 {
    pub value:    u32,
    pub selector: u32,
}

pub const NLM_F_REQUEST:       u16 = 0x01;
pub const NLM_F_MULTI:         u16 = 0x02;
pub const NLM_F_ACK:           u16 = 0x04;
pub const NLM_F_ECHO:          u16 = 0x08;
pub const NLM_F_DUMP_INTR:     u16 = 0x10;
pub const NLM_F_DUMP_FILTERED: u16 = 0x20;
pub const NLM_F_ROOT:          u16 = 0x100;
pub const NLM_F_MATCH:         u16 = 0x200;
pub const NLM_F_ATOMIC:        u16 = 0x400;
pub const NLM_F_DUMP:          u16 = NLM_F_ROOT | NLM_F_MATCH;
pub const NLM_F_REPLACE:       u16 = 0x100;
pub const NLM_F_EXCL:          u16 = 0x200;
pub const NLM_F_CREATE:        u16 = 0x400;
pub const NLM_F_APPEND:        u16 = 0x800;
pub const NLM_F_NONREC:        u16 = 0x100;
pub const NLM_F_BULK:          u16 = 0x200;
pub const NLM_F_CAPPED:        u16 = 0x100;
pub const NLM_F_ACK_TLVS:      u16 = 0x200;

pub const NLA_F_NESTED:        u16 = 1 << 15;
pub const NLA_F_NET_BYTEORDER: u16 = 1 << 14;
pub const NLA_TYPE_MASK:       u16 = !(NLA_F_NESTED | NLA_F_NET_BYTEORDER);

pub const NLMSG_ALIGNTO: u32 = 4;
pub const NLA_ALIGNTO:   u32 = 4;
```

`Nlmsg::align(n: u32) -> u32`:
1. return `(n + 3) & !3`.

`Nlmsg::ok(nlh: &Nlmsghdr, len: usize) -> bool`:
1. if len < 16: return false.
2. if nlh.nlmsg_len < 16: return false.
3. if (nlh.nlmsg_len as usize) > len: return false.
4. return true.

`Netlink::bind(sk: &mut Sock, addr: &SockaddrNl) -> Result<()>`:
1. if addr.nl_family != AF_NETLINK: return Err(EINVAL).
2. /* Group-membership capability check */
3. if addr.nl_groups != 0:
   - capable_protocol_admin(sk.protocol)? — typically CAP_NET_ADMIN.
4. /* Port-id assignment */
5. if addr.nl_pid == 0:
   - sk.portid = Netlink::autobind(current.tgid)?.
6. else:
   - if addr.nl_pid != current.tgid && !capable(CAP_NET_ADMIN):
     - return Err(EPERM).
   - if Netlink::portid_in_use(addr.nl_pid): return Err(EADDRINUSE).
   - sk.portid = addr.nl_pid.
7. /* Subscribe multicast groups */
8. for bit in 0..32: if addr.nl_groups & (1 << bit) != 0: sk.groups |= 1 << bit.
9. return Ok(()).

`Netlink::sendmsg(sk: &Sock, msg: &UserMsghdr, len: usize) -> Result<i64>`:
1. /* Bounds */
2. iov = copy_iovec_from_user(msg.msg_iov, msg.msg_iovlen)?.
3. nlh = first nlmsghdr in iov; require Nlmsg::ok(nlh, len).
4. /* Authority: kernel rewrites nlmsg_pid */
5. nlh.nlmsg_pid = sk.portid.
6. /* Dispatch by family */
7. match sk.protocol {
   NETLINK_ROUTE     => rtnl_rcv(sk, nlh)?,
   NETLINK_NETFILTER => nfnetlink_rcv(sk, nlh)?,
   NETLINK_AUDIT     => audit_receive(sk, nlh)?,
   /* ... */
   NETLINK_GENERIC   => genl_rcv(sk, nlh)?,
   _                 => return Err(EPROTONOSUPPORT),
}.
8. /* NLM_F_ACK reply */
9. if nlh.nlmsg_flags & NLM_F_ACK != 0:
   - Netlink::ack(sk, nlh, result_errno)?.
10. return Ok(len as i64).

`Netlink::ack(sk: &Sock, orig: &Nlmsghdr, err: i32) -> Result<()>`:
1. cap = sk.flags & NETLINK_CAP_ACK != 0.
2. extack = sk.flags & NETLINK_EXT_ACK != 0.
3. /* Build NLMSG_ERROR reply */
4. resp.nlmsg_type = NLMSG_ERROR.
5. resp.nlmsg_flags = if cap { NLM_F_CAPPED } else { 0 } | if extack { NLM_F_ACK_TLVS } else { 0 }.
6. payload.error = err.
7. payload.msg = if cap { Nlmsghdr::zero() } else { *orig }.
8. if extack ∧ err < 0: append NLMSGERR_ATTR_MSG, NLMSGERR_ATTR_OFFS TLVs.
9. enqueue resp to sk.receive_queue.
10. return Ok(()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sockaddr_nl_is_12` | INVARIANT | `sizeof::<SockaddrNl>() == 12`. |
| `nlmsghdr_is_16` | INVARIANT | `sizeof::<Nlmsghdr>() == 16`. |
| `nlattr_is_4` | INVARIANT | `sizeof::<Nlattr>() == 4`. |
| `nlmsg_align_idempotent` | INVARIANT | per-call: `align(align(n)) == align(n)`. |
| `nlmsg_ok_implies_safe_deref` | INVARIANT | per-call: `ok(nlh, len) ⟹ nlh.nlmsg_len fits in [16, len]`. |
| `nla_flag_bits_mutex` | INVARIANT | per-attr: NESTED ∧ NET_BYTEORDER ⟹ reject under strict mode. |
| `bind_portid_authority` | INVARIANT | per-bind: `nl_pid != 0 ∧ nl_pid != tgid ∧ !CAP_NET_ADMIN ⟹ EPERM`. |
| `bind_groups_authority` | INVARIANT | per-bind: `nl_groups != 0 ∧ !CAP_NET_ADMIN ⟹ EPERM`. |
| `sendmsg_pid_rewritten` | INVARIANT | per-sendmsg: outbound `nlmsg_pid = sk.portid`. |
| `ack_capped_flag_set` | INVARIANT | per-ack: `NETLINK_CAP_ACK ⟹ NLM_F_CAPPED` in reply, original `msg` zeroed. |
| `multipart_terminated_by_done` | INVARIANT | per-dump: last fragment has `nlmsg_type == NLMSG_DONE`. |
| `protocol_range_check` | INVARIANT | per-socket: `protocol ∈ [0, MAX_LINKS) ∨ EPROTONOSUPPORT`. |

### Layer 2: TLA+

`uapi/headers/netlink.tla`:
- Per-socket(AF_NETLINK) → bind → sendmsg(request) → kernel dispatch → ack/multicast → recvmsg lifecycle.
- Properties:
  - `safety_portid_unique` — per-bind: two sockets cannot share a nonzero portid simultaneously.
  - `safety_nlmsg_pid_not_spoofable` — per-sendmsg: outbound `nlmsg_pid` always == sk.portid.
  - `safety_dump_terminates_with_done` — every NLM_F_MULTI dump emits exactly one NLMSG_DONE.
  - `safety_ack_at_most_once_per_request` — per-NLM_F_ACK: ≤ 1 NLMSG_ERROR reply.
  - `safety_capped_ack_zeroes_msg` — per-NETLINK_CAP_ACK: original nlmsghdr in reply is zero-filled.
  - `liveness_request_eventually_ack` — per-NLM_F_ACK request: eventually receives NLMSG_ERROR.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `SockaddrNl` layout post: 12 bytes, fields contiguous | `SockaddrNl` |
| `Nlmsghdr` layout post: 16 bytes, fields contiguous | `Nlmsghdr` |
| `Nlattr` layout post: 4 bytes | `Nlattr` |
| `Nlmsg::ok` post: ret ⟹ safe to deref nlmsg_len | `Nlmsg::ok` |
| `Nlmsg::align` post: ret = ((n + 3) & ~3) | `Nlmsg::align` |
| `Netlink::bind` post: portid ∈ {tgid, autobind result}; groups ⊆ {grant-set} | `Netlink::bind` |
| `Netlink::sendmsg` post: nlmsg_pid == sk.portid in outbound copy | `Netlink::sendmsg` |
| `Netlink::ack` post: exactly one NLMSG_ERROR enqueued | `Netlink::ack` |

### Layer 4: Verus/Creusot functional

`Per socket(AF_NETLINK, type, protocol) → bind(sockaddr_nl) → sendmsg(nlmsg/...) → per-protocol-dispatch (rtnl_rcv / nfnetlink_rcv / genl_rcv / ...) → recvmsg(ack-or-multicast)` semantic equivalence: per-`Documentation/userspace-api/netlink/intro.rst`, per-`Documentation/userspace-api/netlink/specs.rst`, per-RFC 3549 (rtnetlink), and per-libnl3 ABI tests.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

Netlink UAPI reinforcement:

- **`NLMSG_OK` enforced before every dereference** — defense against per-truncated-header parse OOB (CVE-2016-3134 class).
- **`nlmsg_pid` rewritten by kernel on send** — defense against per-message source-spoofing (CVE-2009-3613 lineage).
- **`nl_pid` bind requires `CAP_NET_ADMIN` for cross-tgid values** — defense against per-portid-squatting / impersonation.
- **`nl_groups` bind requires per-protocol capability** — defense against per-multicast eavesdropping (audit, uevent, xfrm).
- **Protocol-number range check `< MAX_LINKS`** — defense against per-OOB `nl_table[]` indexing.
- **`NETLINK_GET_STRICT_CHK` rejects unknown attributes** — defense against per-policy-bypass via padded/unknown TLVs.
- **`NLA_F_NESTED ∧ NLA_F_NET_BYTEORDER` reject** — defense against per-attribute-flag confusion.
- **Dump consistency via `NLM_F_DUMP_INTR`** — defense against per-dump-during-mutation TOCTOU.
- **`NETLINK_NO_ENOBUFS` bounded retry / silent drop** — defense against per-overrun amplification.
- **Per-uid send-rate limiting** — defense against per-flood DoS on kernel netlink queues.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF/USERCOPY** on every `copy_from_user`/`copy_to_user` touching `struct sockaddr_nl`, `struct nlmsghdr`, `struct nlattr`, and `struct iovec` payloads — defense against per-kernel-pointer-dereference of user-controlled bytes during `bind(2)` / `sendmsg(2)` / `recvmsg(2)` and against per-TLV-walk OOB read where `nla_len` is attacker-chosen.
- **PAX_RANDKSTACK on syscall entry** — randomize kernel-stack offset on every `socket(AF_NETLINK,...)`, `bind(2)`, `sendmsg(2)`, `recvmsg(2)`, `setsockopt(SOL_NETLINK,...)` entry so heap/stack grooming of the netlink parsing context (per-family `cb_running` and dump callbacks) is statistically infeasible.
- **GRKERNSEC_NETLINK_AUDIT** — every `NETLINK_AUDIT` socket open, every `audit_set_*` request, every group-1024 subscription is logged via the audit subsystem itself; defense against per-stealth audit-tampering and against per-CAP_AUDIT_CONTROL abuse.
- **NETLINK_PORT_PRIVS** — `nl_pid` values < 1024 are reserved to processes with `CAP_NET_ADMIN`. Unprivileged autobind is restricted to `nl_pid >= 1024`, matching the privileged-port convention from BSD; defense against per-portid-confusion with well-known service IDs.
- **CAP_NET_ADMIN gating on all writer operations** — every `RTM_NEWLINK/SETLINK/DELLINK/NEWROUTE/DELROUTE`, every `nftables` table/chain/rule mutation, every `NETLINK_XFRM` SA/SP modification, and every `NETLINK_ADD_MEMBERSHIP` for non-default groups requires `CAP_NET_ADMIN`. The check is at the syscall edge in `sendmsg`, not deep in the family handler — defense against per-handler-omission privilege escalation.
- **CAP_AUDIT_CONTROL / CAP_AUDIT_WRITE split for NETLINK_AUDIT** — `AUDIT_*_CONFIG` (rules, watch lists) requires `CAP_AUDIT_CONTROL`; `AUDIT_USER` writes require `CAP_AUDIT_WRITE`. Default-deny under grsec policy.
- **CAP_SYS_ADMIN for NETLINK_KOBJECT_UEVENT multicast** — userspace cannot inject synthetic uevents (which can trigger `modprobe` autoload) without `CAP_SYS_ADMIN`; defense against per-fake-uevent kernel-module-loading-as-unprivileged (CVE-2017-6074 class on protocol module autoloading).
- **`nlmsg_pid` rewrite on every outbound frame** — kernel forcibly sets `nlmsg_pid = sk.portid` (or `0` for kernel-origin multicasts), making source-spoofing structurally impossible regardless of attacker control over the payload.
- **`NLM_F_REPLACE` / `_EXCL` / `_CREATE` / `_APPEND` race-narrowed under rtnl_lock** — every state-changing request is serialized on `rtnl_lock` (or per-subsystem mutex); defense against per-TOCTOU races between flag check and state mutation.
- **`NETLINK_LISTEN_ALL_NSID` denied in unprivileged user-namespaces** — even with `CAP_NET_ADMIN` inside a user-namespace, the cross-netns multicast listening capability requires `CAP_NET_ADMIN` in the **initial** netns; defense against per-userns-escape via cross-namespace netlink eavesdropping.
- **Attribute-policy enforcement via `NETLINK_GET_STRICT_CHK`** under grsec policy is the default-on mode — every subsystem must declare a `struct nla_policy[]` and unknown/oversize attributes are rejected with extack. Defense against per-policy-bypass and per-uninitialized-kernel-stack disclosure via padded attributes.
- **`NETLINK_EXT_ACK` extack strings sanitized** — `NLMSGERR_ATTR_MSG` strings emitted by the kernel are bounded (`NL_MAX_EXTACK_MSG`) and stripped of non-printable bytes; defense against per-extack-channel kernel-memory-leak (similar to CVE-2017-15265 class).
- **Per-uid rate-limiting on `NETLINK_AUDIT` `AUDIT_USER` writes** — bounded to ~5 msg/s per uid under grsec; defense against per-audit-log-DoS that masks concurrent attacker activity.

## Open Questions

(none at this Tier-5 level — per-family Tier-3 docs cover rtnetlink / nfnetlink / genetlink / audit specifics)

## Out of Scope

- `include/uapi/linux/rtnetlink.h` — rtnetlink message types (`RTM_*`) and family-header structs (covered in `uapi/headers/rtnetlink.md` Tier-5).
- `include/uapi/linux/genetlink.h` — generic-netlink dispatch ABI (covered separately).
- `include/uapi/linux/netfilter/nfnetlink.h` — nfnetlink subsystem dispatch (covered separately).
- `include/uapi/linux/audit.h` — audit message types (covered in `uapi/headers/audit.md` Tier-5).
- `include/uapi/linux/inet_diag.h` — sock_diag dump ABI (covered separately).
- `net/netlink/af_netlink.c` implementation (covered in `net/netlink.md` Tier-3).
- Implementation code.
