---
title: "Tier-3: net/netfilter/ipvs/ip_vs_core.c — IPVS (IP Virtual Server / LVS) L4 load-balancer"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

IPVS (IP Virtual Server) is in-kernel L4 (TCP/UDP/SCTP) load-balancer (LVS project). Per-`ip_vs_service` defines per-(VIP:vport, protocol) service; per-`ip_vs_dest` adds real-server (RS) backends per-service. Per-skb matching service VIP: scheduler picks RS per-policy (RR, WRR, LC, WLC, LBLC, SH, DH, FO, OVF, MH, SED, NQ); creates per-(client:cport, VIP:vport, RS:rport) connection-entry; subsequent skbs route to same RS. Per-forwarding-mode: NAT (DNAT), DR (Direct Route MAC-rewrite), TUN (IP-in-IP encap). Per-`ip_vs_sync` master/backup state replication. Critical for: per-DC L4 LB, OpenStack-LBaaS, container-ingress.

This Tier-3 covers `ip_vs_core.c` (~2632 lines).

### Acceptance Criteria

- [ ] AC-1: ipvsadm -A -t VIP:vport -s wlc: ip_vs_service created.
- [ ] AC-2: ipvsadm -a -t VIP:vport -r RS:rport -m: RS added; NAT-mode.
- [ ] AC-3: Per-incoming SYN to VIP:vport: ip_vs_schedule picks RS; conn-entry created.
- [ ] AC-4: Subsequent skbs from same client: lookup conn-entry → same RS.
- [ ] AC-5: Per-DR mode: skb.dst-MAC rewritten to RS.
- [ ] AC-6: Per-TUN mode: IP-in-IP encap to RS.
- [ ] AC-7: Per-WLC: pick RS with min(active/weight).
- [ ] AC-8: Per-Maglev (MH): per-source-IP-hash assigns RS.
- [ ] AC-9: ip_vs_sync master → backup: per-conn-entry replicated.
- [ ] AC-10: ipvsadm -lc -t VIP:vport: per-conn list dumped.

### Architecture

Per-service:

```
struct IpVsService {
  list: ListLink,
  af: u16,
  protocol: u16,
  addr: IpVsAddr,
  port: __be16,
  fwmark: u32,
  flags: u32,
  timeout: u32,
  netmask: u32,
  destinations: ListHead<IpVsDest>,
  num_dests: u32,
  scheduler: *IpVsScheduler,
  sched_data: *void,
  ...
}

struct IpVsDest {
  list: ListLink,
  af: u16,
  addr: IpVsAddr,
  port: __be16,
  weight: AtomicI32,
  conn_flags: AtomicU32,                          // IP_VS_CONN_F_*
  u_threshold: u32,
  l_threshold: u32,
  activeconns: AtomicI32,
  inactconns: AtomicI32,
  persistconns: AtomicI32,
  ...
}

struct IpVsConn {
  hlist: HListLink,                                // in conn-table
  caddr: IpVsAddr,                                 // client
  cport: __be16,
  vaddr: IpVsAddr,                                 // virtual
  vport: __be16,
  daddr: IpVsAddr,                                 // destination (RS)
  dport: __be16,
  af: u16,
  protocol: u16,
  flags: AtomicU32,                                // IP_VS_CONN_F_*
  dest: *IpVsDest,
  state: u16,
  refcnt: AtomicI32,
  timeout: u32,
  timer: TimerList,
  app: *IpVsApp,                                   // per-helper (FTP, etc.)
  ...
}

struct IpVsScheduler {
  list: ListLink,
  name: &'static str,
  refcnt: AtomicI32,
  init_service: fn(svc) -> i32,
  done_service: fn(svc) -> i32,
  add_dest: fn(svc, dest),
  del_dest: fn(svc, dest),
  upd_dest: fn(svc, dest),
  schedule: fn(svc, skb, iph) -> *IpVsDest,
}
```

`IpVs::in(state, skb, ipvsh)` (NF_INET_LOCAL_IN hook):
1. /* Per-skb: lookup conn-entry */
2. cp = ip_vs_conn_in_get(net, ipvsh.proto, &ipvsh.saddr, ipvsh.sport, &ipvsh.daddr, ipvsh.dport).
3. if !cp:
   - svc = ip_vs_service_find(net, ipvsh.proto, &ipvsh.daddr, ipvsh.dport).
   - if !svc: return NF_ACCEPT.
   - cp = ip_vs_schedule(svc, skb, ipvsh, conn_flags).
   - if !cp: return NF_DROP.
4. /* Per-conn dispatch */
5. return ip_vs_xmit(cp, skb, ipvsh).

`IpVs::schedule(svc, skb, iph, ...) -> *IpVsConn`:
1. dest = svc.scheduler.schedule(svc, skb, iph).
2. if !dest: return NULL.
3. cp = ip_vs_conn_new(svc, ..., dest, ...).
4. return cp.

`IpVs::xmit(cp, skb, ipvsh)`:
1. switch cp.flags & IP_VS_CONN_F_FWD_MASK:
   - IP_VS_CONN_F_MASQ: ip_vs_nat_xmit(skb, cp, ipvsh).
   - IP_VS_CONN_F_DROUTE: ip_vs_dr_xmit(skb, cp, ipvsh).
   - IP_VS_CONN_F_TUNNEL: ip_vs_tunnel_xmit(skb, cp, ipvsh).
   - IP_VS_CONN_F_LOCALNODE: ip_vs_null_xmit(skb, cp).

`IpVs::nat_xmit(skb, cp, ipvsh)`:
1. /* Modify skb.iph.daddr = cp.daddr; skb.tcph.dest = cp.dport */
2. ip_route_me_harder(skb).
3. dst_output(skb).

`IpVs::dr_xmit(skb, cp, ipvsh)`:
1. /* Look up dest's MAC via ARP */
2. neigh = neigh_lookup(cp.daddr).
3. dev_queue_xmit_with_dst_mac(skb, neigh.ha).

`IpVs::tunnel_xmit(skb, cp, ipvsh)`:
1. /* Encapsulate IP-in-IP: outer-IP src = local + dst = cp.daddr */
2. skb_push(skb, sizeof(iphdr)).
3. fill outer iph.
4. ip_local_out(skb).

`IpVsConn::new(svc, ..., dest, ...) -> *IpVsConn`:
1. cp = kmem_cache_alloc.
2. cp.{caddr, cport, vaddr, vport, daddr, dport, proto, af, dest, flags, state}.
3. hlist_add(&cp.hlist, &conn_tab[hashkey]).
4. timer_setup(&cp.timer, ip_vs_conn_expire, 0).
5. return cp.

### Out of Scope

- net/netfilter/ipvs/ip_vs_{ctl, sync, conn, app}.c (covered separately if expanded)
- Per-scheduler module (rr, wrr, lc, wlc, lblc, sh, dh, fo, ovf, mh, sed, nq; covered separately)
- nf_conntrack interaction (covered in `nf_conntrack.md` separately)
- ipvsadm userspace (out-of-tree)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ip_vs_service` | per-(VIP:vport, proto) virtual service | `IpVsService` |
| `struct ip_vs_dest` | per-RS backend | `IpVsDest` |
| `struct ip_vs_conn` | per-active conn-entry | `IpVsConn` |
| `struct ip_vs_scheduler` | per-policy scheduler | `IpVsScheduler` |
| `struct ip_vs_protocol` | per-(TCP/UDP/SCTP/AH/ESP) handler | `IpVsProtocol` |
| `ip_vs_in()` / `ip_vs_out()` | per-skb NF-hook in/out | `IpVs::in` / `out` |
| `ip_vs_schedule()` | per-skb pick RS | `IpVs::schedule` |
| `ip_vs_conn_new()` | per-conn-entry create | `IpVsConn::new` |
| `ip_vs_conn_hashkey()` | per-(c, c, v, v, r, r) hash | `IpVsConn::hashkey` |
| `ip_vs_conn_in_get()` / `_out_get()` | per-(direction) conn-lookup | `IpVsConn::in_get` / `out_get` |
| `ip_vs_xmit()` | per-RS xmit (NAT/DR/TUN) | `IpVs::xmit` |
| `ip_vs_nat_xmit()` / `_dr_xmit()` / `_tunnel_xmit()` | per-mode xmit | `IpVs::*_xmit` |
| `ip_vs_sync_conn()` | per-conn-entry sync to backup | `IpVsSync::conn` |
| `ip_vs_genl_*` | per-netlink ABI | `IpVs::genl_*` |
| `IP_VS_FWD_*` enum | NAT / LOCAL / TUNNEL / ROUTE | UAPI |
| `IP_VS_CMD_*` | netlink commands | UAPI |

### compatibility contract

REQ-1: Per-service (ip_vs_service):
- af: AF_INET / AF_INET6.
- proto: IPPROTO_TCP / UDP / SCTP / AH / ESP.
- addr: VIP.
- port: virtual port.
- flags: IP_VS_SVC_F_PERSISTENT / HASHED / ONEPACKET / SCHED1.
- timeout: persistent timeout.
- scheduler: ip_vs_scheduler.
- destinations: ListHead of ip_vs_dest.

REQ-2: Per-destination (ip_vs_dest):
- af, addr, port: RS endpoint.
- weight: per-RS weight (for WRR/WLC).
- conn_flags: IP_VS_CONN_F_FWD_MASK / MASQ / TUNNEL / DROUTE / etc.
- u_threshold, l_threshold: upper/lower conn-threshold.
- atime, ctime: timing stats.
- activeconns: AtomicI32.
- inactconns: AtomicI32.

REQ-3: Per-scheduler algorithms:
- RR: Round-Robin.
- WRR: Weighted Round-Robin.
- LC: Least-Connections.
- WLC: Weighted Least-Connections (default).
- LBLC: Locality-Based Least-Connections.
- LBLCR: Locality-Based with Replication.
- SH: Source-Hashing.
- DH: Destination-Hashing.
- FO: Forwarding-Only / weighted-failover.
- OVF: Overflow-connection.
- MH: Maglev-Hashing.
- SED: Shortest-Expected-Delay.
- NQ: Never-Queue.

REQ-4: Per-forwarding mode:
- IP_VS_CONN_F_MASQ (NAT): DNAT skb to RS; SNAT reply.
- IP_VS_CONN_F_DROUTE (DR): rewrite skb.dst-MAC to RS; ARP-only redirect.
- IP_VS_CONN_F_TUNNEL: IP-in-IP encap to RS.
- IP_VS_CONN_F_LOCALNODE: deliver locally.

REQ-5: Per-skb in-path (ip_vs_in):
- NF_HOOK at NF_INET_LOCAL_IN.
- Lookup conn-entry by (client, VIP, vport).
- If miss: ip_vs_schedule → pick RS → ip_vs_conn_new.
- If hit: per-conn dispatch.
- ip_vs_xmit per-conn-mode.

REQ-6: ip_vs_schedule(svc, skb, hdr):
- Per-svc.scheduler.schedule(svc, skb).
- Returns ip_vs_dest selection.

REQ-7: Per-WLC formula:
- WLC picks dest with min(activeconns / weight).

REQ-8: Per-conn-table:
- Hash by (client_af + client_addr + cport + vaddr + vport + proto).
- 2^N-bucket rhashtable.

REQ-9: Per-persistent timeout:
- Per-(client) sticky-to-RS for IP_VS_SVC_F_PERSISTENT.

REQ-10: Per-sync (master/backup):
- ip_vs_sync_conn over UDP-multicast.
- Backup replicates per-conn-entry from master.

REQ-11: Per-userspace ABI:
- ip_vs_genl netlink: IP_VS_CMD_NEW/SET/DEL/GET_SERVICE/DEST.
- ipvsadm tool.
- /proc/net/ip_vs (legacy).

REQ-12: Per-namespace:
- Per-net IPVS instance.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `service_unique_per_proto_addr_port` | INVARIANT | per-(af, proto, addr, port): at most one service. |
| `conn_in_table_iff_in_hash` | INVARIANT | per-cp: in conn_tab[hashkey] iff hashed-in. |
| `dest_in_service` | INVARIANT | per-cp.dest ∈ svc.destinations. |
| `fwd_mode_exclusive` | INVARIANT | per-cp.flags has exactly one IP_VS_CONN_F_FWD_* bit. |
| `weight_positive` | INVARIANT | per-dest.weight > 0 (else dest inactive). |

### Layer 2: TLA+

`net/netfilter/ipvs/ipvs.tla`:
- Per-skb in-path + schedule + xmit + sync.
- Properties:
  - `safety_per_conn_same_RS` — per-active conn: subsequent skbs route to same RS.
  - `safety_per_service_destinations_consistent` — per-svc destinations ⊆ ip_vs registered.
  - `liveness_per_sync_eventually_propagates` — per-conn-entry on master ⟹ eventually on backup.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IpVs::in` post: per-skb dispatched via xmit or accepted | `IpVs::in` |
| `IpVs::schedule` post: per-svc-policy returns dest; conn_new called | `IpVs::schedule` |
| `IpVs::nat_xmit` post: skb.daddr = cp.daddr; ip_route called | `IpVs::nat_xmit` |
| `IpVs::dr_xmit` post: skb.dst-MAC = neigh.ha | `IpVs::dr_xmit` |

### Layer 4: Verus/Creusot functional

`Per-(VIP, vport, proto) load-balanced to per-RS-list with chosen scheduler + per-conn-entry sticks` semantic equivalence: per-LVS/IPVS architecture.

### hardening

(Inherits row-1 features from `net/netfilter/00-overview.md` § Hardening.)

IPVS reinforcement:

- **Per-conn_table sized to limit** — defense against per-DDoS conn-bomb.
- **Per-conn timeout-expire** — defense against per-stale conn-entry leak.
- **Per-CAP_NET_ADMIN for genl** — defense against unprivileged config.
- **Per-sync HMAC (CONFIG_IP_VS_SYNC_AUTH)** — defense against per-sync-spoof.
- **Per-scheduler null-check** — defense against per-driver-error scheduler-NULL.
- **Per-namespace IPVS scoped** — defense against cross-ns leak.
- **Per-NF-hook RCU-protected** — defense against per-svc-list mutate UAF.
- **Per-FWD_MASK exclusive** — defense against per-cp ambiguous mode.
- **Per-dest weight bounded** — defense against per-config insane weight.
- **Per-conn-rate-limit per service** — defense against per-service flood.
- **Per-MH Maglev seed per-svc** — defense against per-deterministic spread.

