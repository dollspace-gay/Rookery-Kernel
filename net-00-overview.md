---
title: "Subsystem: net/ — networking stack"
tags: ["design-doc", "subsystem", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-2 overview for `net/` — the kernel networking stack. Owns the socket layer (BSD sockets API), the cross-protocol substrate (sk_buff, net_device, struct sock), every IP version (IPv4 + IPv6 + MPLS), every transport protocol (TCP, UDP, SCTP, MPTCP, DCCP, RDS, RxRPC, TIPC, SMC), the firewall/filter framework (netfilter + iptables/nftables), QoS and traffic control (`net/sched/`), IPsec (`net/xfrm/`), the L2 bridge + VLAN + DSA + switchdev abstractions, ethernet helpers + ethtool ABI, devlink (vendor-driver-management ABI), wireless (cfg80211 + mac80211 + ieee802154), bluetooth, eXpress Data Path (XDP), TLS offload, sunrpc (NFS transport), and ~25 less-common protocol families.

`net/` is the second-largest subsystem after `fs/` by surface area: 67 subdirectories, ~400K LoC. Per-protocol coverage strategy mirrors `fs/`: class-tiered with a named v0 port set; the long tail keeps upstream C via FFI or is dropped (CONFIG=n).

### Requirements

- REQ-1: Every socket-level syscall is byte-identical with upstream — entry/exit conventions, address-family dispatch, errno returns.
- REQ-2: TCP, UDP, ICMP, IPv4, IPv6 wire formats are byte-identical (RFC compliant + Linux's specific extensions).
- REQ-3: AF_UNIX (local domain) and AF_PACKET (raw + cooked) preserve their semantics.
- REQ-4: AF_NETLINK + generic netlink (genl) preserve wire format and protocol-family registration semantics; existing `iproute2`, `ethtool`, `nftables` userspace tools work unmodified.
- REQ-5: Netfilter framework (nftables, iptables-legacy, ipset, conntrack) preserves its hook points, the nftables wire ABI over NFNL netlink, and conntrack's per-flow accounting.
- REQ-6: TC (Traffic Control / QoS) preserves all 40+ schedulers, classifiers, and actions; `tc` userspace works unmodified.
- REQ-7: IPsec (xfrm) preserves the SAD/SPD models, AH/ESP wire formats, IKE_SA_INIT/IKE_AUTH negotiation interface to userspace (XFRM_MSG_* netlink), tunneling and transport modes.
- REQ-8: Bridge + VLAN + DSA + switchdev preserve their L2 abstractions; existing tools (`bridge`, `tc`-based switchdev offload) work unmodified.
- REQ-9: ethtool ABI preserved (every ETHTOOL_* command, register dump format, statistics name list).
- REQ-10: devlink ABI preserved (DEVLINK_CMD_*, region snapshots, fmsg, traps).
- REQ-11: TLS offload (kTLS) preserves the SOL_TLS setsockopt API.
- REQ-12: MPTCP preserves its sockopt suite, path manager, and observed wire format.
- REQ-13: cfg80211 + mac80211 (wireless) preserve the cfg80211 netlink (NL80211) interface; existing `iw`, `wpa_supplicant`, `hostapd` work unmodified.
- REQ-14: Bluetooth preserves the BT-MGMT and HCI ABIs; `bluetoothctl` and `BlueZ` work unmodified.
- REQ-15: XDP program attachment ABI (BPF_PROG_TYPE_XDP_*, XDP_REDIRECT, AF_XDP sockets) preserved; XDP-using userspace works unmodified.
- REQ-16: All Tier-3 docs spawned by this overview each declare their unsafe-block clusters, TLA+ models (where novel concurrency primitives are introduced), and Kani harnesses for protocol-state-machine invariants.
- REQ-17: The implementation reuses upstream rust-for-linux's existing net abstractions (`rust/kernel/net.rs`, `rust/kernel/net/`).
- REQ-18: For each protocol-family in the v0 port set (decided per Q1 below): full Rust implementation with verification artifacts. For each protocol-family NOT in v0: upstream-C-via-FFI or CONFIG=n.

### Acceptance Criteria

- [ ] AC-1: Socket-syscall `strace` golden tests pass byte-identically. (covers REQ-1)
- [ ] AC-2: Wireshark/tcpdump capture of TCP/UDP/ICMP/v4/v6 traffic on Rookery vs. upstream is byte-identical (modulo timestamps). (covers REQ-2)
- [ ] AC-3: AF_UNIX selftests (`tools/testing/selftests/net/af_unix/`) pass. (covers REQ-3)
- [ ] AC-4: `iproute2` / `ethtool` / `nftables` / `tc` / `bridge` / `ip` userspace tools, unmodified, produce identical output for equivalent kernel state. (covers REQ-4, REQ-5, REQ-6, REQ-8, REQ-9)
- [ ] AC-5: nftables selftests (`tools/testing/selftests/net/netfilter/`) pass. (covers REQ-5)
- [ ] AC-6: TC selftests (`tools/testing/selftests/tc-testing/`) pass. (covers REQ-6)
- [ ] AC-7: xfrm selftests (`tools/testing/selftests/net/xfrm_*`) pass. (covers REQ-7)
- [ ] AC-8: ethtool selftests pass + a curated `ethtool -i / -k / -g / -S / -d` golden test produces identical output. (covers REQ-9)
- [ ] AC-9: devlink selftests pass. (covers REQ-10)
- [ ] AC-10: ktls selftest mounts an OpenSSL-driven TLS offload session. (covers REQ-11)
- [ ] AC-11: MPTCP selftests pass. (covers REQ-12)
- [ ] AC-12: `iw` / `wpa_supplicant` / `hostapd` smoke tests pass on a virtual mac80211 device. (covers REQ-13)
- [ ] AC-13: BlueZ smoke test (HCI virtual device + `bluetoothctl`) succeeds. (covers REQ-14)
- [ ] AC-14: XDP selftests (`tools/testing/selftests/bpf/test_xdp_*`) pass. (covers REQ-15)
- [ ] AC-15: `make verify` passes net/ Kani harnesses; `make tla` passes net/ models. (covers REQ-16)
- [ ] AC-16: A grep over Rookery for `kernel::net::*` shows reuse of upstream rust-for-linux abstractions. (covers REQ-17)
- [ ] AC-17: For each protocol in v0 port set: a curated reference workload (e.g., `iperf3` for TCP, `dnsperf` for DNS-over-UDP, `ssh` for TCP/keepalive) runs successfully. (covers REQ-18)

### Architecture

### Layout map (Tier-3 docs spawned)

```
.design/net/
  00-overview.md              ← this document
  socket-api.md
  skbuff.md
  netdev.md
  struct-sock.md
  bpf-net.md                  ← cross-references kernel/bpf/00-overview.md
  fib.md                      ← FIB / routing tables / dst cache
  netlink.md
  packet.md                   ← AF_PACKET (raw, cooked, fanout)
  unix-socket.md
  ipv4/
    00-overview.md
    tcp.md
    udp.md
    icmp.md
    igmp.md
    fib4.md
    fragment.md
    multicast.md
    raw.md
    arp.md
    gre-tunnels.md
    mroute.md
  ipv6/
    00-overview.md            ← analogous structure to ipv4
    tcp6.md, udp6.md, icmp6.md, mld.md, fib6.md, ...
  netfilter.md                ← may grow into nested subdir during Tier-3 work
  traffic-control.md          ← TC (40+ schedulers and classifiers)
  xfrm.md                     ← IPsec
  l2/
    00-overview.md
    bridge.md
    vlan.md
    dsa.md
    switchdev.md
  ethtool.md
  devlink.md
  tls.md
  mptcp.md
  sctp.md
  wireless/
    00-overview.md
    cfg80211.md
    mac80211.md
    ieee802154.md
  bluetooth.md
  sunrpc.md
  vsock.md
  9p-transport.md
  niche-transports.md         ← KCM + RDS + RxRPC
```

### Cross-references

- `kernel/00-overview.md` — `kernel/bpf/` provides BPF program substrate; `bpf-net.md` integrates.
- `mm/00-overview.md` — sk_buff allocations are on slab; mempool helpers from mm/.
- `lib/00-overview.md` — checksum helpers (`include/net/checksum.h` consumes `lib/checksum.c`).
- `fs/00-overview.md` — netlink sockets are file descriptors; sunrpc serves NFS in fs/network-fs/nfs-client.md.
- `crypto/00-overview.md` (Phase B) — TLS, IPsec consume the kernel crypto API.
- `block/00-overview.md` — none direct.
- `drivers/00-overview.md` (Phase B) — every NIC driver consumes netdev API; every wireless driver consumes mac80211; every Bluetooth driver consumes BlueZ HCI.
- `00-glossary.md` — `sk_buff`, `net_device`, `struct sock`, `NAPI`, `netfilter`.

### Rust module organization (informative)

- `kernel::net` — exists upstream (`rust/kernel/net.rs` + `rust/kernel/net/`); extend
- `kernel::net::skb` — Rookery to author / extend
- `kernel::net::netdev` — Rookery to author
- `kernel::net::sock` — Rookery to author
- `kernel::net::ip{4,6}` — Rookery to author (large)
- `kernel::net::netlink` — Rookery to author
- `kernel::net::netfilter` — Rookery to author
- `kernel::net::tc` — Rookery to author (heavy, ~80 modules)

### Locking and concurrency

Net is heavy concurrency, second only to mm/ + kernel/sched:
- Per-`sk` lock (`sk->sk_lock` — composite of socket-bottom-half lock + slock + per-listening-queue locks).
- RCU for routing table reads (FIB lookup is RCU-hot).
- `rtnl_lock` (mutex) — global netdev mutator; held for ifup/ifdown/ifaddr/route changes.
- `nfnl_lock` — netfilter mutator.
- Per-queue `tx_queue->_xmit_lock` — TX path serialization.
- Per-NAPI `napi->state` — atomic state machine.
- Connection tracking: per-bucket spinlock + RCU for read.

Many of these primitives have nontrivial concurrency contracts. TLA+ models in Tier-3 docs.

### Error handling

Net-specific common returns: ENETDOWN, ENETUNREACH, EHOSTUNREACH, ENETRESET, ECONNRESET, ECONNREFUSED, ECONNABORTED, ETIMEDOUT, EMSGSIZE, EPROTOTYPE, EPROTONOSUPPORT, EAFNOSUPPORT, EOPNOTSUPP, EADDRINUSE, EADDRNOTAVAIL.

### Out of Scope

- Niche protocols listed in Q1 bucket C.
- Wireless full Rust port (per Q2 — defer to v1+).
- Bluetooth full Rust port (per Q3 — defer to v1+).
- 32-bit-only network paths.
- Test fixtures.
- Implementation code — `.design/` contains specs only.

### upstream references in scope

| Category | Upstream paths | Planned Tier-3 doc |
|---|---|---|
| Socket layer + protocol-family registration | `net/socket.c`, `include/linux/{socket,net}.h`, `include/uapi/linux/socket.h` | `socket-api.md` |
| sk_buff (canonical packet container) | `net/core/skbuff.c`, `net/core/datagram.c`, `net/core/gro.c`, `net/core/gso.c`, `include/linux/skbuff.h` | `skbuff.md` |
| net_device + dev rx/tx + GRO/GSO/LRO | `net/core/dev.c`, `net/core/dev_*.c`, `net/core/dev_ioctl.c`, `net/core/dev_api.c`, `net/core/devmem.c`, `net/core/link_watch.c`, `net/core/hotdata.c`, `net/core/gro_cells.c`, `net/core/hwbm.c`, `include/linux/netdevice.h` | `netdev.md` |
| struct sock + protocol-agnostic socket state | `net/core/sock.c` (probably), `include/net/sock.h` | `struct-sock.md` |
| BPF on the network path | `net/core/filter.c`, `net/core/lwt_bpf.c`, `net/bpf/`, `net/xdp/`, `net/core/sock_bpf.c` (etc.), `net/core/bpf_sk_storage.c` | `bpf-net.md` (cross-references `kernel/bpf/00-overview.md`) |
| Routing infrastructure (FIB) | `net/core/fib_notifier.c`, `net/core/fib_rules.c`, `net/core/dst.c`, `net/core/dst_cache.c`, `net/core/flow_*.c` | `fib.md` |
| netlink (cross-protocol control plane) | `net/netlink/` (af_netlink, genetlink, policy) | `netlink.md` |
| Packet socket + raw sockets | `net/packet/` | `packet.md` |
| AF_UNIX (local domain sockets) | `net/unix/` | `unix-socket.md` |
| IPv4 (TCP/UDP/ICMP/IGMP, FIB, TLS-SOL_IP, TC, …) | `net/ipv4/` | `ipv4/00-overview.md` (Tier 3 hub) spawning `ipv4/{tcp,udp,icmp,igmp,fib,frag,multicast,raw,tunnel,gre,mroute,arp,...}.md` |
| IPv6 | `net/ipv6/` | `ipv6/00-overview.md` (analogous structure to ipv4) |
| Netfilter (iptables/nftables/conntrack) | `net/netfilter/` | `netfilter.md` (substantial — spawns sub-docs for nf_tables, ipset, conntrack) |
| TC / QoS | `net/sched/` (40+ schedulers and classifiers) | `traffic-control.md` |
| IPsec | `net/xfrm/` | `xfrm.md` |
| L2 bridging | `net/bridge/`, `net/8021q/` (VLAN), `net/dsa/` (Distributed Switch Architecture), `net/switchdev/` | `l2/00-overview.md` (Tier 3 hub) spawning `l2/{bridge,vlan,dsa,switchdev}.md` |
| Ethernet helpers + ethtool ABI | `net/ethernet/`, `net/ethtool/` | `ethtool.md` |
| Devlink (vendor-driver management ABI) | `net/devlink/` | `devlink.md` |
| TLS in kernel | `net/tls/`, `net/psp/` (Per-packet Security Protocol) | `tls.md` |
| MPTCP (Multipath TCP) | `net/mptcp/` | `mptcp.md` |
| SCTP | `net/sctp/` | `sctp.md` |
| Wireless (cfg80211 + mac80211 + ieee802154 + 6lowpan) | `net/wireless/`, `net/mac80211/`, `net/ieee802154/`, `net/mac802154/`, `net/6lowpan/` | `wireless/00-overview.md` (Tier 3 hub) |
| Bluetooth | `net/bluetooth/` | `bluetooth.md` |
| RPC (used by NFS) | `net/sunrpc/`, `net/handshake/` (TLS handshake offload for sunrpc) | `sunrpc.md` (cross-ref `fs/network-fs/nfs-client.md`) |
| Vsock (host-VM communication) | `net/vmw_vsock/` | `vsock.md` |
| 9P transport | `net/9p/` | `9p-transport.md` (cross-ref `fs/network-fs/9p.md`) |
| Generic infra: drop_monitor, link_watch, devmem, hotdata, etc. | listed under the netdev/skb docs above | folded into `netdev.md` |
| Ceph transport (used by fs/ceph) | `net/ceph/` | folded into `fs/network-fs/ceph.md` (network/transport split documented there) |
| KCM, RDS, RxRPC | `net/kcm/`, `net/rds/`, `net/rxrpc/` | `niche-transports.md` (covers all three) |
| Strparser | `net/strparser/` | folded into `tls.md` and `bpf-net.md` |
| **Niche / dead protocols** (CONFIG=n by default for v0): net/appletalk, net/atm, net/x25, net/lapb, net/llc, net/iucv (s390-only), net/phonet (Nokia), net/dccp (deprecated, may be removed), net/key (PF_KEY), net/qrtr (Qualcomm IPC), net/ife, net/nfc, net/nsh, net/openvswitch, net/batman-adv, net/hsr, net/can, net/dcb, net/dns_resolver, net/dsa shouldn't be there (it's L2), net/ncsi, net/netlabel, net/psample, net/rfkill, net/shaper, net/smc, net/tipc, net/l2tp, net/l3mdev, net/mac802154 listed twice — let's clean: niche set is appletalk, atm, x25, lapb, llc, iucv, phonet, dccp, key, qrtr, ife, can, mctp (?), nfc, nsh, openvswitch, batman-adv, hsr, dcb, dns_resolver, ncsi, netlabel, psample, rfkill, smc, tipc, l2tp | per-protocol Tier-3 docs are deferred; some kept as upstream-C-via-FFI |

### compatibility contract

`net/` owns the third-largest userspace-visible compat slice (after `mm/` and `kernel/`). The compat target is unusually well-defined because most network protocols are RFC-specified.

### Syscall surface

`socket`, `bind`, `connect`, `accept`, `accept4`, `listen`, `socketpair`, `getsockname`, `getpeername`, `send`, `sendto`, `sendmsg`, `sendmmsg`, `recv`, `recvfrom`, `recvmsg`, `recvmmsg`, `shutdown`, `setsockopt`, `getsockopt`, `socketcall` (compat).

(Each gets a Tier-5 `uapi/syscalls/<name>.md` in Phase D. The set is small but every option to setsockopt/getsockopt is a separate compat surface.)

### Address families (UAPI)

Every value in `enum AF_*` has a documented protocol-family handler. Compat target: every AF_* value upstream supports continues to work (or returns ENOSYS-equivalent identically). UAPI: `include/linux/socket.h` enum `AF_*`.

### `/proc/net/*` surfaces

| Path | Owner doc | Compat level |
|---|---|---|
| `/proc/net/{tcp,tcp6,udp,udp6,raw,raw6,unix,packet,netlink}` | per-protocol Tier-3 doc | Format-identical |
| `/proc/net/{route,fib_trie,fib_triestat,arp,igmp,if_inet6,ipv6_route,sockstat,sockstat6}` | `fib.md`, `ipv4/00-overview.md`, `ipv6/00-overview.md` | Format-identical |
| `/proc/net/dev`, `/proc/net/dev_mcast`, `/proc/net/wireless` | `netdev.md`, `wireless/00-overview.md` | Format-identical |
| `/proc/net/netfilter/*` | `netfilter.md` | Format-identical |
| `/proc/net/{stat/<x>}` (nf_conntrack stats etc.) | `netfilter.md` | Format-identical |
| `/proc/net/xfrm_stat` | `xfrm.md` | Format-identical |
| `/proc/net/<proto-specific>` | per-protocol Tier-3 docs | Format-identical |

### `/sys` surfaces

| Path | Owner doc | Compat level |
|---|---|---|
| `/sys/class/net/<dev>/{addr_*,carrier,dev_id,duplex,...}` | `netdev.md` | Identical |
| `/sys/class/net/<dev>/queues/{rx,tx}-<n>/*` | `netdev.md` | Identical |
| `/sys/class/net/<dev>/statistics/*` | `netdev.md` | Field-identical |
| `/sys/class/net/<dev>/wireless/*` | `wireless/00-overview.md` | Identical |
| `/sys/devices/virtual/net/lo/*` | `netdev.md` | Identical |

### Wire-level

- **TCP wire format**: byte-identical (RFC 793 + RFC 7323 + RFC 9293 + IETF-track extensions). Window-scale, SACK, ECN, Timestamps, Fast Open.
- **UDP wire format**: byte-identical (RFC 768).
- **IPv4 + IPv6 packet headers**: byte-identical.
- **ICMP / ICMPv6 message types**: byte-identical.
- **Netfilter NFNL netlink protocol** (used by nftables): wire-identical.
- **TC netlink protocol** (used by `tc`): wire-identical.
- **NFS RPC wire format** (NFSv3 / v4 / v4.1 / pNFS): wire-identical.
- **TLS wire protocol** (handshake + record layer): RFC-identical (but TLS handshake is userspace; only record-layer offload is in kernel).

### ioctls (mostly socket / netdev)

- **SIOC* family**: SIOCGIFNAME, SIOCSIFFLAGS, SIOCGIFADDR, SIOCETHTOOL (passes through to `ethtool.md`), SIOCBOND* (bonding), SIOCBR* (bridge), SIOCWAN* (WAN), SIOCSI* (interface mgmt)
- **SO_* setsockopt names + values**: identical
- **TCP_* options**: identical (TCP_NODELAY, TCP_CORK, TCP_KEEPIDLE, TCP_USER_TIMEOUT, TCP_FASTOPEN, TCP_QUICKACK, TCP_DEFER_ACCEPT, TCP_MD5SIG, …)

### verification

### Layer 1: Kani SAFETY proofs

Anticipated `unsafe` clusters:
- skb construction + frag-list + cloning
- netdev RX path: NAPI poll, GRO accumulation, GSO segmentation
- TX path: enqueue, hardware-queue selection, fragment-by-MSS
- Netlink message parsing
- Netfilter hook chain traversal under RCU
- xfrm template/state lookup

### Layer 2: TLA+ models

- `models/net/skb_clone.tla` — proves shared-skb refcounting under concurrent clone/free is correct.
- `models/net/napi_state.tla` — proves NAPI's `STATE_SCHED` / `STATE_NPSVC` / `STATE_DISABLE` state machine never deadlocks.
- `models/net/conntrack.tla` — proves connection-tracking entry lifecycle (NEW → ESTABLISHED → CLOSED) under concurrent traversal.
- `models/net/tcp_state.tla` — proves TCP's RFC 9293 state machine implementation matches the spec under concurrent recv-from-network + app-write.
- `models/net/fib_rcu.tla` — proves FIB lookup readers always observe coherent route entries despite concurrent mutators.
- `models/net/rtnl_lock.tla` — proves rtnl_lock's big-mutex semantics serialize all netdev mutations correctly.

### Layer 3: Kani harnesses

Per `00-overview.md` D4, net is NOT one of the four mandatory Layer-3 areas (those were mm, arch, sched, fs/dcache). However many net data structures benefit from invariant harnesses:

| Data structure | Invariant | Harness |
|---|---|---|
| skb frag list | "Shared-info refcount is consistent with frag list length" | `kani::proofs::net::skb::frag_invariants` |
| FIB trie | "Lookup converges; longest-prefix-match invariant" | `kani::proofs::net::fib::trie_invariants` |
| conntrack hash | "Each entry exists in exactly one bucket" | `kani::proofs::net::conntrack::hash_invariants` |
| TCP send queue | "skb chain by sequence number; no overlap; no gap until rcv_nxt" | `kani::proofs::net::tcp::sendq_invariants` |

### Layer 4: Functional correctness via Creusot / Verus / Prusti (opt-in)

Strong opt-in candidates:
- `tcp.md` — TCP-state-machine functional correctness via Verus (state machine matches RFC 9293). Strong candidate.
- `netlink.md` — netlink-message parser correctness via Creusot.
- `xfrm.md` — IPsec replay-window correctness.
- `bpf-net.md` — XDP verifier soundness (cross-references kernel/bpf/verifier.md).

### hardening

Placeholder per `00-overview.md` D6. net/ owns implementation of:
- Netfilter (defense in depth — userspace-controllable firewall)
- IPsec (cryptographic transport)
- TLS offload
- Connection tracking with size limits
- TCP SYN cookies (defense against SYN flood)
- Reverse-path filtering (rp_filter)
- Sysctl knobs hardening tcp_*/tcp_synack_retries etc.
- BPF on the network path (XDP-based DDoS filtering)

Per D6, binding policy in deferred `00-security-principles.md`.

