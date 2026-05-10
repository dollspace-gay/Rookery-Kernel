---
title: "Tier-2: net/netlink — Netlink (af_netlink + GENERIC + diag + policy validation)"
tags: ["tier-2", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for AF_NETLINK — the kernel⇄userspace control-plane channel that every modern Linux configuration tool consumes: `iproute2` (NETLINK_ROUTE, NETLINK_GENERIC for nftables/devlink/nl80211/etc.), `iptables-nft` + `nftables` (NETLINK_NETFILTER), `audit` (NETLINK_AUDIT), `udev`/`systemd-journald` (NETLINK_KOBJECT_UEVENT, NETLINK_AUDIT), `selinuxctl` (NETLINK_SELINUX), `xfrm` ipsec userspace (NETLINK_XFRM), `genl` consumers (~80+ Generic-Netlink families). Components: **af_netlink** (`af_netlink.c` + `af_netlink.h`: AF_NETLINK socket impl — per-protocol-family dispatch, multicast group mgmt, peer-to-peer + broadcast, netlink dump support with NLM_F_DUMP), **genetlink** (`genetlink.c` + `genetlink.h` + `policy.c`: Generic Netlink — protocol-family-on-NETLINK_GENERIC with auto-assigned id; consumed by ~80 subsystems including nl80211, devlink, ethtool-netlink, taskstats, ipvs, l2tp, tipc, fou, ipset, ila, nbd, smc, mptcp, mctp), **diag** (`diag.c`: NETLINK_NETLINK_DIAG — introspect netlink sockets, used by `ss -A netlink`).

NETLINK protocol families (defined elsewhere but consumed via this Tier-2): `NETLINK_ROUTE` (rtnetlink — net/core/rtnetlink.c), `NETLINK_USERSOCK` (userspace-userspace), `NETLINK_FIREWALL` (deprecated), `NETLINK_SOCK_DIAG` (sock-diag), `NETLINK_NFLOG` (legacy netfilter log), `NETLINK_XFRM` (IPsec), `NETLINK_SELINUX`, `NETLINK_ISCSI` (iSCSI userspace), `NETLINK_AUDIT`, `NETLINK_FIB_LOOKUP` (FIB lookup), `NETLINK_CONNECTOR` (connector), `NETLINK_NETFILTER` (nftables / nfnetlink), `NETLINK_IP6_FW` (legacy), `NETLINK_DNRTMSG` (DECnet), `NETLINK_KOBJECT_UEVENT` (uevent multicast), `NETLINK_GENERIC` (genl), `NETLINK_SCSITRANSPORT`, `NETLINK_ECRYPTFS`, `NETLINK_RDMA`, `NETLINK_CRYPTO`, `NETLINK_SMC`.

### Acceptance Criteria

- [ ] AC-O1: `ip route show` + `ip link show` (NETLINK_ROUTE) work.
- [ ] AC-O2: `nft list ruleset` (NETLINK_NETFILTER) works.
- [ ] AC-O3: `iw dev wlan0 info` (NETLINK_GENERIC family nl80211) works.
- [ ] AC-O4: `ss -A netlink` (NETLINK_NETLINK_DIAG) lists netlink sockets.
- [ ] AC-O5: `auditctl -l` (NETLINK_AUDIT) reads audit rules.
- [ ] AC-O6: `udevadm monitor` (NETLINK_KOBJECT_UEVENT) shows uevents.
- [ ] AC-O7: kselftest `tools/testing/selftests/net/netlink/` passes.

### Out of Scope

- Per-protocol-family handlers (rtnetlink in net/core, sock_diag in net/core, audit in kernel/audit, etc.)
- Implementation code
- 32-bit-only paths

### compatibility contract — outline

- AF_NETLINK socket protocol family numbers + per-family multicast group numbers byte-identical so iproute2, iptables, nftables, audit, udev, libnl, libmnl consume unchanged.
- Netlink message header `struct nlmsghdr` (16 bytes) byte-identical layout.
- Netlink attribute header `struct nlattr` byte-identical with NLA_HDRLEN / NLA_ALIGNTO=4.
- NLM_F_* flags (`_REQUEST`, `_MULTI`, `_ACK`, `_ECHO`, `_DUMP_INTR`, `_DUMP_FILTERED`, `_ROOT`, `_MATCH`, `_ATOMIC`, `_DUMP`, `_REPLACE`, `_EXCL`, `_CREATE`, `_APPEND`) byte-identical.
- NLMSG_* constants byte-identical.
- Per-genl-family numeric IDs are dynamically assigned but stable per-build (consumers query via `CTRL_CMD_GETFAMILY` lookup at runtime); multicast group names are stable.
- NLA_POLICY validation framework (NL_POLICY_*, NLA_REJECT_TYPE_*, etc.) preserved.
- NETLINK_NETLINK_DIAG (`SOCK_DIAG_BY_FAMILY` w/ AF_NETLINK + SOCK_RAW) byte-identical so `ss -A netlink` consumes unchanged.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `net/netlink/af-netlink.md` | `af_netlink.c` + `af_netlink.h`: AF_NETLINK socket + per-family dispatch + multicast |
| `net/netlink/dump.md` | `af_netlink.c` (dump portion): NLM_F_DUMP semantics |
| `net/netlink/genetlink.md` | `genetlink.c` + `genetlink.h`: Generic Netlink framework |
| `net/netlink/policy.md` | `policy.c`: NLA_POLICY validation framework |
| `net/netlink/diag.md` | `diag.c`: NETLINK_NETLINK_DIAG introspection |

### compatibility outline

- REQ-O1: AF_NETLINK family + per-protocol IDs + multicast groups byte-identical (libnl/libmnl/libnftables/libnl-route/libnl-genl consume unchanged).
- REQ-O2: nlmsghdr + nlattr layouts + NLM_F_* flags byte-identical.
- REQ-O3: Generic Netlink CTRL_CMD_* + multicast group registration byte-identical.
- REQ-O4: NLA_POLICY validation framework source-compat.
- REQ-O5: NETLINK_NETLINK_DIAG byte-identical (ss consumes unchanged).
- REQ-O6: TLA+ models (per-socket recv-skb queue + concurrent send-from-N-clients; netlink-dump cursor stability; broadcast multicast delivery to N subscribers).
- REQ-O7: Hardening: per-namespace netlink scoping, per-uid netlink message rate-limit, NETLINK_FIREWALL deprecated.

### verification

| TLA+ Model | Owner |
|---|---|
| `models/netlink/recv_queue.tla` | `net/netlink/af-netlink.md` (proves: per-socket recv skb-queue under concurrent senders + dump producer; queue overflow returns -ENOBUFS, no message dropped silently; back-pressure correctly applied) |
| `models/netlink/dump_cursor.tla` | `net/netlink/dump.md` (proves: NLM_F_DUMP cursor stability — concurrent rb-tree modifications during multi-skb dump; cursor either restarts cleanly with NLM_F_DUMP_INTR set or skips affected entries deterministically) |
| `models/netlink/multicast_broadcast.tla` | `net/netlink/af-netlink.md` (proves: multicast broadcast to N subscribers — every subscriber receives message exactly once or gets -ENOBUFS; ordering preserved per-subscriber) |

### hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-netlink_sock refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-family `genl_family` + `nla_policy` tables `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-msg payload + nla payload arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed netlink_sock priv data cleared | § Default-on configurable |

Netlink-specific reinforcement: NLA_POLICY validation default-on (defense against undersized/oversized attribute attacks); per-socket recv-buffer cap enforced; NETLINK_FIREWALL family disabled (deprecated); per-namespace scoping for non-root namespaces; CAP_NET_ADMIN required for most kernel-side write commands; per-genl-family LSM hook (`security_genlmsg_send`).

