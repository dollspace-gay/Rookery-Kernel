# Tier-5 UAPI: include/uapi/linux/if.h — Network Interface ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/if.h (~298 lines)
-->

## Summary

The `<linux/if.h>` UAPI is the legacy ioctl-edge contract for `SIOCGIFCONF` / `SIOCGIFFLAGS` / `SIOCSIFFLAGS` / `SIOCGIFADDR` / `SIOCSIFADDR` / `SIOCGIFNAME` / `SIOCGIFINDEX` / `SIOCGIFHWADDR` / `SIOCSIFHWADDR` / `SIOCGIFMTU` / `SIOCSIFMTU` / `SIOCSIFNAME` / `SIOCBRADDIF` / etc. It defines:

- **Interface request structure** `struct ifreq` — a 32+16-byte tagged union: `ifrn_name[IFNAMSIZ]` (the interface name, the tag) plus an `ifr_ifru` union covering every per-ioctl payload: `ifru_addr`/`ifru_dstaddr`/`ifru_broadaddr`/`ifru_netmask`/`ifru_hwaddr` (each a `struct sockaddr`), `ifru_flags` (`short`), `ifru_ivalue` (`int` — aliased as `ifr_metric`/`ifr_ifindex`/`ifr_bandwidth`/`ifr_qlen`), `ifru_mtu` (`int`), `ifru_map` (`struct ifmap`), `ifru_slave[IFNAMSIZ]`, `ifru_newname[IFNAMSIZ]`, `ifru_data` (`void __user *`), `ifru_settings` (`struct if_settings`).
- **Convenience field macros**: `ifr_name`, `ifr_addr`, `ifr_dstaddr`, `ifr_broadaddr`, `ifr_netmask`, `ifr_hwaddr`, `ifr_flags`, `ifr_metric`, `ifr_mtu`, `ifr_map`, `ifr_slave`, `ifr_data`, `ifr_ifindex`, `ifr_bandwidth`, `ifr_qlen`, `ifr_newname`, `ifr_settings`.
- **Configuration array** `struct ifconf` — `ifc_len` + union of `ifcu_buf` / `ifcu_req` — used by `SIOCGIFCONF` to dump all interfaces.
- **Device-mapping** `struct ifmap` — legacy ISA/PCMCIA fields (mem_start, mem_end, base_addr, irq, dma, port).
- **Per-port-select identifiers** `IF_PORT_UNKNOWN`/`10BASE2`/`10BASET`/`AUI`/`100BASE_T`/`100BASE_TX`/`100BASE_FX` (from `<linux/netdevice.h>` siblings; exposed via `ifmap.port`).
- **`net_device_flags` enum** — the user-visible flags word read/written via `SIOCGIFFLAGS`/`SIOCSIFFLAGS`: `IFF_UP`, `IFF_BROADCAST`, `IFF_DEBUG`, `IFF_LOOPBACK`, `IFF_POINTOPOINT`, `IFF_NOTRAILERS`, `IFF_RUNNING`, `IFF_NOARP`, `IFF_PROMISC`, `IFF_ALLMULTI`, `IFF_MASTER`, `IFF_SLAVE`, `IFF_MULTICAST`, `IFF_PORTSEL`, `IFF_AUTOMEDIA`, `IFF_DYNAMIC`, `IFF_LOWER_UP`, `IFF_DORMANT`, `IFF_ECHO` (bits 0..18).
- **`IFF_VOLATILE` mask** — the subset of flags the kernel manages and userspace cannot persistently set: `IFF_LOOPBACK|POINTOPOINT|BROADCAST|ECHO|MASTER|SLAVE|RUNNING|LOWER_UP|DORMANT`.
- **Name-size constants** `IFNAMSIZ == 16`, `IFALIASZ == 256`, `ALTIFNAMSIZ == 128`, `IFHWADDRLEN == 6`.
- **RFC 2863 operational status** `IF_OPER_UNKNOWN`/`NOTPRESENT`/`DOWN`/`LOWERLAYERDOWN`/`TESTING`/`DORMANT`/`UP` and **link-mode** `IF_LINK_MODE_DEFAULT`/`DORMANT`/`TESTING`.
- **HDLC interface/protocol selectors** `IF_GET_IFACE`/`IF_GET_PROTO`, `IF_IFACE_V35`/`V24`/`X21`/`T1`/`E1`/`SYNC_SERIAL`/`X21D`, `IF_PROTO_HDLC`/`PPP`/`CISCO`/`FR`/`X25`/`HDLC_ETH`/`RAW` and Frame-Relay PVC variants — exposed via `ifreq.ifr_settings`.

Critical for: `ifconfig(8)`, `ip link` (when it falls back to `SIOCG/SIOC`), every legacy DHCP client (`dhclient`, `dhcpcd`), every L2-bridge configuration tool (`brctl`), every container-runtime that touches interfaces via ioctls (Docker, `systemd-nspawn`), and every WiFi/Bluetooth daemon that calls `SIOCGIFHWADDR`.

This Tier-5 covers `include/uapi/linux/if.h` (~298 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ifreq` | per-ioctl request/response | `Ifreq` |
| `struct ifmap` | per-device-mapping legacy ISA | `Ifmap` |
| `struct ifconf` | per-SIOCGIFCONF dump | `Ifconf` |
| `struct if_settings` | per-ifreq HDLC settings | `IfSettings` |
| `enum net_device_flags` | per-IFF_* bits | `NetDeviceFlags` |
| `IFF_VOLATILE` | per-kernel-managed mask | `Iff::VOLATILE` |
| `IFNAMSIZ` / `IFALIASZ` / `ALTIFNAMSIZ` | per-name buffer sizes | `Iface::{NAMSIZ,ALIASZ,ALTNAMSIZ}` |
| `IFHWADDRLEN` | per-MAC-addr length | `Iface::HWADDRLEN` |
| `IF_OPER_*` | per-RFC2863 oper-state | `IfOper` |
| `IF_LINK_MODE_*` | per-link-mode | `IfLinkMode` |
| `IF_IFACE_*` / `IF_PROTO_*` | per-HDLC selectors | `IfIface` / `IfProto` |
| `dev_ifsioc` | per-SIOC* dispatch | `NetDev::ifsioc` |
| `dev_ioctl` | per-`SIOCGIFCONF`/`SIOCGIFNAME` etc | `NetDev::ioctl` |
| `dev_get_flags` / `dev_change_flags` | per-flags get/set | `NetDev::{get_flags,change_flags}` |
| `dev_change_name` | per-SIOCSIFNAME | `NetDev::change_name` |

## ABI surface (constants + structs)

### Size constants (`if.h:33-36`, `if.h:235`)

```text
IFNAMSIZ      = 16    /* including trailing NUL */
IFALIASZ      = 256   /* alias name buffer */
ALTIFNAMSIZ   = 128   /* alternate names (rtnetlink only) */
IFHWADDRLEN   = 6     /* MAC address length */
```

`IFNAMSIZ == 16` is one of the most ABI-fragile constants in the kernel: every userspace program copies up to 16 bytes (15 chars + NUL) into `ifr_name` and the kernel **must** treat an unterminated name as invalid.

### `struct ifreq` (`if.h:234-256`)

```text
struct ifreq {
    union {
        char ifrn_name[IFNAMSIZ];   /* "eth0", "wlan0", ... */
    } ifr_ifrn;

    union {
        struct sockaddr ifru_addr;       /* SIOCGIFADDR / SIOCSIFADDR */
        struct sockaddr ifru_dstaddr;    /* SIOCGIFDSTADDR / SIOCSIFDSTADDR */
        struct sockaddr ifru_broadaddr;  /* SIOCGIFBRDADDR / SIOCSIFBRDADDR */
        struct sockaddr ifru_netmask;    /* SIOCGIFNETMASK / SIOCSIFNETMASK */
        struct sockaddr ifru_hwaddr;     /* SIOCGIFHWADDR / SIOCSIFHWADDR */
        short           ifru_flags;      /* SIOCGIFFLAGS / SIOCSIFFLAGS */
        int             ifru_ivalue;     /* metric/ifindex/bandwidth/qlen */
        int             ifru_mtu;        /* SIOCGIFMTU / SIOCSIFMTU */
        struct ifmap    ifru_map;        /* SIOCGIFMAP / SIOCSIFMAP */
        char            ifru_slave[IFNAMSIZ];   /* SIOCBONDENSLAVE etc */
        char            ifru_newname[IFNAMSIZ]; /* SIOCSIFNAME */
        void __user    *ifru_data;       /* SIOCDEVPRIVATE / SIOCSIFTXQLEN */
        struct if_settings ifru_settings;/* SIOCWANDEV */
    } ifr_ifru;
};
```

Size: 32 bytes on ILP32 (16 name + 16 union), 40 bytes on LP64 (16 name + 24 union with pointer alignment). The `ifru_data` pointer-bearing variant is the size driver on LP64.

### Field macros (`if.h:259-275`)

```text
ifr_name      = ifr_ifrn.ifrn_name        /* always-valid alias for the tag */
ifr_hwaddr    = ifr_ifru.ifru_hwaddr      /* MAC */
ifr_addr      = ifr_ifru.ifru_addr        /* primary address */
ifr_dstaddr   = ifr_ifru.ifru_dstaddr     /* point-to-point peer */
ifr_broadaddr = ifr_ifru.ifru_broadaddr   /* broadcast */
ifr_netmask   = ifr_ifru.ifru_netmask     /* subnet mask */
ifr_flags     = ifr_ifru.ifru_flags       /* IFF_* */
ifr_metric    = ifr_ifru.ifru_ivalue      /* hop metric */
ifr_mtu       = ifr_ifru.ifru_mtu         /* MTU */
ifr_map       = ifr_ifru.ifru_map         /* ISA/PCMCIA map */
ifr_slave     = ifr_ifru.ifru_slave       /* bonding slave */
ifr_data      = ifr_ifru.ifru_data        /* driver-private blob ptr */
ifr_ifindex   = ifr_ifru.ifru_ivalue      /* RFC 3493 if_index */
ifr_bandwidth = ifr_ifru.ifru_ivalue      /* link speed */
ifr_qlen      = ifr_ifru.ifru_ivalue      /* tx queue length */
ifr_newname   = ifr_ifru.ifru_newname     /* SIOCSIFNAME target */
ifr_settings  = ifr_ifru.ifru_settings    /* HDLC settings */
```

### `struct ifmap` (`if.h:196-204`)

```text
struct ifmap {
    unsigned long  mem_start;   /* device memory range */
    unsigned long  mem_end;
    unsigned short base_addr;   /* I/O port base */
    unsigned char  irq;
    unsigned char  dma;
    unsigned char  port;        /* IF_PORT_* */
    /* 3 bytes implicit padding */
};
```

### `struct ifconf` (`if.h:286-293`)

```text
struct ifconf {
    int ifc_len;            /* in: buffer size; out: bytes written */
    union {
        char        __user *ifcu_buf;
        struct ifreq __user *ifcu_req;
    } ifc_ifcu;
};
#define ifc_buf  ifc_ifcu.ifcu_buf
#define ifc_req  ifc_ifcu.ifcu_req
```

### `enum net_device_flags` (`if.h:82-107`)

```text
IFF_UP          = 1 << 0    /* interface is up (admin) */
IFF_BROADCAST   = 1 << 1    /* broadcast address valid */
IFF_DEBUG       = 1 << 2    /* turn on debugging */
IFF_LOOPBACK    = 1 << 3    /* is a loopback net */
IFF_POINTOPOINT = 1 << 4    /* p-p link */
IFF_NOTRAILERS  = 1 << 5    /* avoid use of trailers */
IFF_RUNNING     = 1 << 6    /* RFC2863 OPER_UP */
IFF_NOARP       = 1 << 7    /* no ARP protocol */
IFF_PROMISC     = 1 << 8    /* receive all packets */
IFF_ALLMULTI    = 1 << 9    /* receive all multicast */
IFF_MASTER      = 1 << 10   /* master of a load balancer */
IFF_SLAVE       = 1 << 11   /* slave of a load balancer */
IFF_MULTICAST   = 1 << 12   /* supports multicast */
IFF_PORTSEL     = 1 << 13   /* can set media type */
IFF_AUTOMEDIA   = 1 << 14   /* auto media select active */
IFF_DYNAMIC     = 1 << 15   /* dialup w/ changing addrs */
IFF_LOWER_UP    = 1 << 16   /* driver signals L1 up */
IFF_DORMANT     = 1 << 17   /* driver signals dormant */
IFF_ECHO        = 1 << 18   /* echo sent packets */

IFF_VOLATILE = (IFF_LOOPBACK | IFF_POINTOPOINT | IFF_BROADCAST | IFF_ECHO |
                IFF_MASTER   | IFF_SLAVE       | IFF_RUNNING   |
                IFF_LOWER_UP | IFF_DORMANT)
```

Bits 16..18 are gated behind `__UAPI_DEF_IF_NET_DEVICE_FLAGS_LOWER_UP_DORMANT_ECHO` for glibc-`<net/if.h>` compatibility. The low 16 bits MUST fit in `short` for `ifr_flags`. `SIOCGIFFLAGS` truncates the kernel's wider internal flag word to the low 16 bits; `SIOCGIFLINK`/`rtnetlink` exposes all 32 bits.

### `IF_GET_IFACE` / `IF_GET_PROTO` and HDLC selectors (`if.h:139-164`)

```text
IF_GET_IFACE = 0x0001
IF_GET_PROTO = 0x0002

IF_IFACE_V35        = 0x1000
IF_IFACE_V24        = 0x1001
IF_IFACE_X21        = 0x1002
IF_IFACE_T1         = 0x1003
IF_IFACE_E1         = 0x1004
IF_IFACE_SYNC_SERIAL= 0x1005   /* read-only */
IF_IFACE_X21D       = 0x1006

IF_PROTO_HDLC       = 0x2000
IF_PROTO_PPP        = 0x2001
IF_PROTO_CISCO      = 0x2002
IF_PROTO_FR         = 0x2003
IF_PROTO_FR_ADD_PVC = 0x2004
IF_PROTO_FR_DEL_PVC = 0x2005
IF_PROTO_X25        = 0x2006
IF_PROTO_HDLC_ETH   = 0x2007
IF_PROTO_FR_ADD_ETH_PVC = 0x2008
IF_PROTO_FR_DEL_ETH_PVC = 0x2009
IF_PROTO_FR_PVC     = 0x200A
IF_PROTO_FR_ETH_PVC = 0x200B
IF_PROTO_RAW        = 0x200C
```

### RFC 2863 operational status (`if.h:167-175`)

```text
IF_OPER_UNKNOWN          = 0
IF_OPER_NOTPRESENT       = 1
IF_OPER_DOWN             = 2
IF_OPER_LOWERLAYERDOWN   = 3
IF_OPER_TESTING          = 4
IF_OPER_DORMANT          = 5
IF_OPER_UP               = 6
```

### Link mode (`if.h:178-182`)

```text
IF_LINK_MODE_DEFAULT = 0
IF_LINK_MODE_DORMANT = 1    /* limit upward transition to DORMANT */
IF_LINK_MODE_TESTING = 2    /* limit upward transition to TESTING */
```

### `IF_PORT_*` (legacy port-select; defined in `<linux/netdevice.h>`, exposed via `ifmap.port`)

```text
IF_PORT_UNKNOWN     = 0
IF_PORT_10BASE2     = 1
IF_PORT_10BASET     = 2
IF_PORT_AUI         = 3
IF_PORT_100BASE_T   = 4
IF_PORT_100BASE_TX  = 5
IF_PORT_100BASE_FX  = 6
```

## Compatibility contract

REQ-1: `IFNAMSIZ == 16` on every supported arch. `ifr_name` MUST be NUL-terminated; the kernel MUST reject `strnlen(ifr_name, IFNAMSIZ) == IFNAMSIZ` (no terminator within bounds) with `-EINVAL`.

REQ-2: `IFALIASZ == 256`, `ALTIFNAMSIZ == 128`, `IFHWADDRLEN == 6`. Cannot change without ABI break.

REQ-3: `struct ifreq` layout — `ifr_ifrn` (16 bytes name union) immediately followed by `ifr_ifru` (union, max-arm-sized). Total `sizeof(struct ifreq)` MUST be 32 bytes ILP32, 40 bytes LP64. Field offset of every `ifru_*` arm MUST be 16.

REQ-4: `struct ifmap` MUST be `{unsigned long mem_start; unsigned long mem_end; unsigned short base_addr; unsigned char irq; unsigned char dma; unsigned char port;}` plus implicit trailing pad. Size: 16 bytes ILP32, 24 bytes LP64.

REQ-5: `struct ifconf` MUST be `{int ifc_len; union {char *ifcu_buf; struct ifreq *ifcu_req;};}`. Size: 8 bytes ILP32, 16 bytes LP64.

REQ-6: `SIOCGIFCONF` — kernel MUST write **as many full `struct ifreq` as fit** into `ifc_buf` up to `ifc_len`, and update `ifc_len` to bytes written. Userspace MUST be able to pass `ifc_buf == NULL` to query the required size (kernel sets `ifc_len` to needed-bytes).

REQ-7: `IFF_*` bit positions MUST match Linux exactly: `IFF_UP=1, IFF_BROADCAST=2, IFF_DEBUG=4, IFF_LOOPBACK=8, IFF_POINTOPOINT=0x10, IFF_NOTRAILERS=0x20, IFF_RUNNING=0x40, IFF_NOARP=0x80, IFF_PROMISC=0x100, IFF_ALLMULTI=0x200, IFF_MASTER=0x400, IFF_SLAVE=0x800, IFF_MULTICAST=0x1000, IFF_PORTSEL=0x2000, IFF_AUTOMEDIA=0x4000, IFF_DYNAMIC=0x8000, IFF_LOWER_UP=0x10000, IFF_DORMANT=0x20000, IFF_ECHO=0x40000`.

REQ-8: `IFF_VOLATILE = IFF_LOOPBACK | IFF_POINTOPOINT | IFF_BROADCAST | IFF_ECHO | IFF_MASTER | IFF_SLAVE | IFF_RUNNING | IFF_LOWER_UP | IFF_DORMANT == 0x3107A`. `SIOCSIFFLAGS` MUST preserve these bits from the kernel-managed state — userspace cannot toggle them via this ioctl.

REQ-9: `SIOCSIFFLAGS` operates on the **low 16 bits** of the flags word. `IFF_LOWER_UP`/`DORMANT`/`ECHO` (bits 16..18) cannot be toggled via this ioctl; they require rtnetlink `RTM_NEWLINK` with `IFLA_OPERSTATE`.

REQ-10: `IFF_PROMISC` and `IFF_ALLMULTI` toggles via `SIOCSIFFLAGS` require `CAP_NET_ADMIN`. The transition increments/decrements a refcount; the actual L2 mode follows the count.

REQ-11: `IFF_UP` toggle (admin up/down) requires `CAP_NET_ADMIN`. Driver `ndo_open`/`ndo_stop` is invoked on the transition.

REQ-12: `SIOCSIFNAME` changes `ifr_name` (current) to `ifr_newname` (target). Requires `CAP_NET_ADMIN` in the netns owning the interface. The new name MUST pass `dev_valid_name` (no NUL except trailing, no '/', no '.', no leading whitespace, no embedded ':', length 1..15).

REQ-13: `SIOCSIFMTU` — new MTU MUST be in `[dev.min_mtu, dev.max_mtu]`. `CAP_NET_ADMIN` required.

REQ-14: `SIOCSIFHWADDR` — new MAC MUST pass `is_valid_ether_addr` (not all-zero, not multicast). `CAP_NET_ADMIN` required. Driver `ndo_set_mac_address` decides whether change is allowed (NIC must be down for many drivers).

REQ-15: `SIOCSIFADDR`/`SIOCSIFNETMASK`/`SIOCSIFBROADCAST`/`SIOCSIFDSTADDR` operate on the **primary** IPv4 address only (in_dev->ifa_list[0]). Require `CAP_NET_ADMIN`. IPv6 must use rtnetlink.

REQ-16: `SIOCGIFINDEX` returns the kernel-assigned `ifindex` (1..) in `ifr_ifindex` (== `ifr_ifru.ifru_ivalue`). `0` is reserved (means "no such interface"). Indices are per-netns and monotonically allocated, recycled after a wraparound (currently 32-bit).

REQ-17: `SIOCGIFNAME` is the inverse of `SIOCGIFINDEX`: input `ifr_ifindex`, output `ifr_name`.

REQ-18: `RFC 2863` operational state `IF_OPER_*` is read via rtnetlink `IFLA_OPERSTATE`, not via `if.h`-style ioctls. `IFF_RUNNING` is the legacy "oper-up" indicator; `IFF_LOWER_UP` is the kernel-managed carrier indicator.

REQ-19: `IF_GET_IFACE` / `IF_GET_PROTO` and the `IF_IFACE_*` / `IF_PROTO_*` selectors are routed via `SIOCWANDEV` and `ifreq.ifr_settings`. These touch HDLC/sync-serial drivers only.

REQ-20: `ifr_data` is a `void __user *` blob pointer carrying driver-private payloads (e.g., `SIOCDEVPRIVATE`). Drivers MUST validate `access_ok` and `copy_from_user` on the pointed-to region — the kernel does NOT do this generically.

## Acceptance Criteria

- [ ] AC-1: `sizeof(struct ifreq) == 40` on LP64, `== 32` on ILP32.
- [ ] AC-2: `sizeof(struct ifmap) == 24` on LP64, `== 16` on ILP32.
- [ ] AC-3: `sizeof(struct ifconf) == 16` on LP64, `== 8` on ILP32.
- [ ] AC-4: `IFNAMSIZ == 16`; `IFALIASZ == 256`; `ALTIFNAMSIZ == 128`; `IFHWADDRLEN == 6`.
- [ ] AC-5: All `IFF_*` numeric values match Linux table above.
- [ ] AC-6: `IFF_VOLATILE == 0x3107A`.
- [ ] AC-7: `SIOCGIFCONF` with `ifc_buf == NULL` returns required `ifc_len` without writing payload.
- [ ] AC-8: `SIOCGIFNAME` followed by `SIOCGIFINDEX` round-trips the same interface.
- [ ] AC-9: `SIOCSIFNAME` with name "eth/0" returns `-EINVAL`; with "eth1" succeeds.
- [ ] AC-10: `SIOCSIFFLAGS` without `CAP_NET_ADMIN` returns `-EPERM`.
- [ ] AC-11: `SIOCSIFFLAGS` toggling `IFF_LOOPBACK` is a silent no-op (volatile preserved).
- [ ] AC-12: `SIOCSIFMTU` with MTU outside `[min_mtu, max_mtu]` returns `-EINVAL`.
- [ ] AC-13: `SIOCSIFHWADDR` with multicast MAC (low bit of byte 0 set) returns `-EADDRNOTAVAIL`.
- [ ] AC-14: `ifr_name` lacking NUL terminator returns `-EINVAL`.
- [ ] AC-15: `SIOCGIFFLAGS` returns only low 16 bits; rtnetlink exposes the full 32-bit flags word.

## Architecture

```
pub const IFNAMSIZ:    usize = 16;
pub const IFALIASZ:    usize = 256;
pub const ALTIFNAMSIZ: usize = 128;
pub const IFHWADDRLEN: usize = 6;

#[repr(C)]
pub union IfrIfrn {
    pub ifrn_name: [u8; IFNAMSIZ],
}

#[repr(C)]
pub union IfrIfru {
    pub ifru_addr:      Sockaddr,
    pub ifru_dstaddr:   Sockaddr,
    pub ifru_broadaddr: Sockaddr,
    pub ifru_netmask:   Sockaddr,
    pub ifru_hwaddr:    Sockaddr,
    pub ifru_flags:     i16,
    pub ifru_ivalue:    i32,
    pub ifru_mtu:       i32,
    pub ifru_map:       Ifmap,
    pub ifru_slave:     [u8; IFNAMSIZ],
    pub ifru_newname:   [u8; IFNAMSIZ],
    pub ifru_data:      UserPtr<c_void>,
    pub ifru_settings:  IfSettings,
}

#[repr(C)]
pub struct Ifreq {
    pub ifr_ifrn: IfrIfrn,
    pub ifr_ifru: IfrIfru,
}

#[repr(C)]
pub struct Ifmap {
    pub mem_start: c_ulong,
    pub mem_end:   c_ulong,
    pub base_addr: u16,
    pub irq:       u8,
    pub dma:       u8,
    pub port:      u8,
    /* 3 bytes implicit pad on LP64 */
}

#[repr(C)]
pub struct Ifconf {
    pub ifc_len:  i32,
    pub ifc_ifcu: IfcIfcu,
}

bitflags! {
    pub struct Iff: u32 {
        const UP          = 1 <<  0;
        const BROADCAST   = 1 <<  1;
        const DEBUG       = 1 <<  2;
        const LOOPBACK    = 1 <<  3;
        const POINTOPOINT = 1 <<  4;
        const NOTRAILERS  = 1 <<  5;
        const RUNNING     = 1 <<  6;
        const NOARP       = 1 <<  7;
        const PROMISC     = 1 <<  8;
        const ALLMULTI    = 1 <<  9;
        const MASTER      = 1 << 10;
        const SLAVE       = 1 << 11;
        const MULTICAST   = 1 << 12;
        const PORTSEL     = 1 << 13;
        const AUTOMEDIA   = 1 << 14;
        const DYNAMIC     = 1 << 15;
        const LOWER_UP    = 1 << 16;
        const DORMANT     = 1 << 17;
        const ECHO        = 1 << 18;
        const VOLATILE    = Self::LOOPBACK.bits | Self::POINTOPOINT.bits |
                            Self::BROADCAST.bits | Self::ECHO.bits |
                            Self::MASTER.bits | Self::SLAVE.bits |
                            Self::RUNNING.bits | Self::LOWER_UP.bits |
                            Self::DORMANT.bits;
    }
}
```

`NetDev::ifsioc(net: &Netns, cmd: u32, ifr: &mut Ifreq) -> Result<()>`:
1. /* Validate interface-name */
2. name = Self::name_from_ifr(ifr)?.    /* NUL-terminate check */
3. /* Look up netdev under RCU */
4. dev = net.dev_get_by_name(&name).ok_or(Err(ENODEV))?.
5. match cmd {
   SIOCGIFFLAGS    => { ifr.ifr_flags = (dev.flags & 0xffff) as i16; }
   SIOCSIFFLAGS    => {
       capable(CAP_NET_ADMIN)?;
       let new = (ifr.ifr_flags as u16) as u32;
       let old = dev.flags;
       /* Preserve VOLATILE bits */
       let target = (old & Iff::VOLATILE.bits) | (new & !Iff::VOLATILE.bits);
       Self::change_flags(&dev, target)?;
   }
   SIOCGIFINDEX    => { ifr.ifr_ifindex = dev.ifindex; }
   SIOCGIFHWADDR   => { ifr.ifr_hwaddr.sa_data[..6].copy_from_slice(&dev.dev_addr); }
   SIOCSIFHWADDR   => {
       capable(CAP_NET_ADMIN)?;
       Self::change_mac(&dev, &ifr.ifr_hwaddr)?;
   }
   SIOCGIFMTU      => { ifr.ifr_mtu = dev.mtu as i32; }
   SIOCSIFMTU      => {
       capable(CAP_NET_ADMIN)?;
       Self::change_mtu(&dev, ifr.ifr_mtu)?;
   }
   SIOCSIFNAME     => {
       capable(CAP_NET_ADMIN)?;
       Self::change_name(&dev, &ifr.ifr_newname)?;
   }
   /* ... per-cmd dispatch ... */
   _ => return Err(EOPNOTSUPP),
}.
6. return Ok(()).

`NetDev::change_flags(dev: &NetDev, target: u32) -> Result<()>`:
1. let old = dev.flags.
2. let changed = old ^ target.
3. if changed & Iff::UP.bits != 0:
   - if target & Iff::UP.bits != 0: dev.ndo_open()?;
   - else: dev.ndo_stop()?.
4. if changed & Iff::PROMISC.bits != 0:
   - dev_set_promiscuity(&dev, sign(target, Iff::PROMISC))?.
5. if changed & Iff::ALLMULTI.bits != 0:
   - dev_set_allmulti(&dev, sign(target, Iff::ALLMULTI))?.
6. dev.flags = target.
7. rtnetlink_event(&dev, RTM_NEWLINK)?.
8. return Ok(()).

`NetDev::change_name(dev: &NetDev, new: &[u8; IFNAMSIZ]) -> Result<()>`:
1. let name = Self::valid_name_or_err(new)?.
2. if name.is_empty() || name == "." || name == ".." || name.contains(b'/'): return Err(EINVAL).
3. if net.dev_get_by_name(name).is_some(): return Err(EEXIST).
4. dev.name = name.into().
5. rtnetlink_event(&dev, RTM_NEWLINK)?.
6. return Ok(()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ifreq_size` | INVARIANT | LP64: `sizeof::<Ifreq>() == 40`; ILP32: 32. |
| `ifmap_size` | INVARIANT | LP64: 24; ILP32: 16. |
| `ifconf_size` | INVARIANT | LP64: 16; ILP32: 8. |
| `iface_constants` | INVARIANT | IFNAMSIZ == 16; IFHWADDRLEN == 6. |
| `iff_bit_table` | INVARIANT | Every IFF_* matches the canonical table. |
| `iff_volatile_mask` | INVARIANT | `Iff::VOLATILE.bits == 0x3107A`. |
| `name_nul_terminated` | INVARIANT | per-ifsioc: name lacking NUL within 16 bytes ⟹ EINVAL. |
| `volatile_preserved_on_set` | INVARIANT | per-SIOCSIFFLAGS: VOLATILE bits in result == VOLATILE bits in old. |
| `cap_net_admin_on_write_ops` | INVARIANT | per-SIOCSIF*: returns EPERM if !CAP_NET_ADMIN. |
| `mtu_bounded` | INVARIANT | per-SIOCSIFMTU: result ∈ [min_mtu, max_mtu]. |

### Layer 2: TLA+

`uapi/headers/if.tla`:
- Per-interface lifecycle: create → SIOCSIFNAME → SIOCSIFADDR → SIOCSIFFLAGS(UP) → SIOCSIFMTU → SIOCSIFFLAGS(~UP) → unregister.
- Properties:
  - `safety_volatile_immutable_via_ioctl` — per-SIOCSIFFLAGS: VOLATILE bits unchanged across ioctl.
  - `safety_name_unique_per_netns` — per-netns: at most one dev with a given name at any time.
  - `safety_ifindex_unique_per_netns` — per-netns: at most one dev with a given ifindex.
  - `safety_cap_net_admin_enforced` — per-write-ioctl: caller has CAP_NET_ADMIN.
  - `safety_promisc_refcount_balanced` — per-SIOCSIFFLAGS PROMISC toggle: refcount get/put balanced.
  - `liveness_iff_up_eventually_opens` — per-SIOCSIFFLAGS(UP): ndo_open called.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Ifreq` layout post: `ifr_ifru` at offset 16 | `Ifreq` |
| `Ifmap` layout post: fields contiguous + trailing pad | `Ifmap` |
| `Ifconf` layout post: pointer-union sized as `void *` | `Ifconf` |
| `Iff` bit table post: values match canonical | `Iff` |
| `NetDev::change_flags` post: VOLATILE preserved; rtnetlink event emitted | `NetDev::change_flags` |
| `NetDev::change_name` post: name passes dev_valid_name; unique in netns | `NetDev::change_name` |
| `NetDev::ifsioc` post: write-ops gated by CAP_NET_ADMIN | `NetDev::ifsioc` |

### Layer 4: Verus/Creusot functional

`Per ioctl(SIOC*, struct ifreq) → name lookup → cmd dispatch → write-cap check → ndo_* invocation → rtnetlink_event` semantic equivalence: per-`Documentation/networking/netdevices.rst`, per-`Documentation/networking/operstates.rst`, per-`include/uapi/linux/sockios.h`, and per-glibc `<net/if.h>` user expectations.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

if.h UAPI reinforcement:

- **`ifr_name` strnlen-validated** — defense against per-non-NUL-terminated name parsing OOB.
- **`IFF_VOLATILE` mask preserved on SIOCSIFFLAGS** — defense against per-user-space-induced bogus state (claiming loopback, claiming master, etc.).
- **`CAP_NET_ADMIN` on every write-ioctl** — defense against per-unprivileged-config-change.
- **`SIOCSIFNAME` rejects '/', '.', '..', empty, embedded NUL** — defense against per-name-confusion attacks (sysfs path injection, /sys/class/net/<name> traversal).
- **MTU bounded by driver `min_mtu`/`max_mtu`** — defense against per-MTU overflow / underflow in IP stack.
- **MAC validated by `is_valid_ether_addr`** — defense against per-multicast-MAC source spoofing.
- **`ifr_data` pointer NOT auto-dereferenced** — drivers MUST validate, no generic kernel-side fast-path.
- **`SIOCGIFCONF` truncates rather than overflows `ifc_len`** — defense against per-userspace-buffer overflow.
- **Per-netns scoping of all SIOC* operations** — defense against per-netns-crossing config leak.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF/USERCOPY** on every `copy_from_user`/`copy_to_user` touching `struct ifreq`, `struct ifmap`, `struct ifconf`, the underlying `struct sockaddr` arms of the union, and the `ifr_data` driver-blob pointer — defense against per-kernel-pointer-dereference of user-controlled bytes during `SIOCGIFCONF` (variable-length dump), `SIOCSIFHWADDR` (MAC copy), and `SIOCDEVPRIVATE` (driver-private blob).
- **PAX_RANDKSTACK on syscall entry** — randomize kernel-stack offset per `ioctl(SIOC*)` entry. `struct ifreq` is frequently allocated on the kernel stack for the duration of the ioctl; randomization defeats heap/stack-grooming for adjacent-field disclosure.
- **CAP_NET_ADMIN gating on every write ioctl** — `SIOCSIFFLAGS`, `SIOCSIFADDR`, `SIOCSIFNETMASK`, `SIOCSIFBRDADDR`, `SIOCSIFDSTADDR`, `SIOCSIFHWADDR`, `SIOCSIFMTU`, `SIOCSIFNAME`, `SIOCSIFTXQLEN`, `SIOCSIFMAP`, `SIOCSIFPFLAGS`, `SIOCSIFLINK`, `SIOCBONDENSLAVE`, `SIOCBRADDIF`. Verified at the **syscall edge** in `dev_ioctl` before family dispatch — defense against per-handler-omission privilege escalation.
- **GRKERNSEC_SOCKET_ALL** — restrict `socket(2)` (the prerequisite for every SIOC* ioctl on a netdev) to a configured group; unprivileged-by-default-deny model. Defense against per-binary-runtime-grant abuse.
- **`SIOCGIFCONF` truncation policy** — when the user buffer is too small, kernel truncates at the last full `struct ifreq` rather than emitting a partial structure; defense against per-uninit-stack-memory leak via straddled buffer.
- **`SIOCSIFNAME` namespace-confined** — name changes are confined to the netns owning the netdev; `dev_change_net_namespace` requires `CAP_NET_ADMIN` in **both** source and target netns. Defense against per-userns-escape via name reuse.
- **`IFF_PROMISC` / `IFF_ALLMULTI` refcount-paranoid** — each toggle increments/decrements a refcount; the actual L2 mode follows the count rather than the last write. Defense against per-toggle race causing persistent eavesdropping (CVE-2014-3122 style).
- **`SIOCSIFHWADDR` requires interface down (most drivers) or CAP_NET_ADMIN + driver-allow** — defense against per-hot-swap MAC spoofing on bonded/bridged interfaces.
- **`ifr_data` driver-private blob requires per-driver CAP gate** — even with `CAP_NET_ADMIN`, certain drivers (e.g., `SIOCETHTOOL`) require additional capability checks for sensitive sub-commands (e.g., firmware flash, NVRAM write). Defense against per-driver-privilege-laundering.
- **GRKERNSEC_PROC restrictions** on `/proc/net/dev`, `/proc/net/wireless`, `/proc/net/route`, `/sys/class/net/<iface>/` — defense against per-unprivileged enumeration of interface presence (which leaks netns membership and MAC addresses).
- **`SIOCGIFFLAGS` truncation to low 16 bits is structural** — `IFF_LOWER_UP`/`DORMANT`/`ECHO` cannot be set via legacy ioctl regardless of capability, forcing modification via rtnetlink (which is rate-limited and audit-logged). Defense against per-legacy-channel state-tampering.
- **`SIOCSIFFLAGS` audit logging** — every `IFF_UP` / `IFF_PROMISC` / `IFF_ALLMULTI` transition is logged via `AUDIT_NETWORK_CHANGE` (mandatory under grsec policy) with caller uid/tgid/cmdline; defense against per-stealth interface-state mutation.

## Open Questions

(none at this Tier-5 level — per-ioctl Tier-3 semantics live in `net/core.md` and `net/ipv4.md`)

## Out of Scope

- `include/uapi/linux/sockios.h` `SIOC*` numeric codes (covered in `uapi/headers/sockios.md` Tier-5).
- `include/uapi/linux/netdevice.h` `IFF_*` mirror constants and per-netdev metadata (covered separately).
- `include/uapi/linux/if_addr.h` `IFA_*` rtnetlink address attributes (covered separately).
- `include/uapi/linux/if_link.h` `IFLA_*` rtnetlink link attributes (covered in `uapi/headers/if_link.md`).
- `include/uapi/linux/ethtool.h` ethtool sub-ioctl ABI (covered separately).
- `net/core/dev_ioctl.c` implementation (covered in `net/core.md` Tier-3).
- Implementation code.
