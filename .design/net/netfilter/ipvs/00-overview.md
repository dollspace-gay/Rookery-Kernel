# Tier-2 (sub): net/netfilter/ipvs — IP Virtual Server (L4 load balancer)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/netfilter/ipvs/
  - include/net/ip_vs.h
  - include/uapi/linux/ip_vs.h
-->

## Summary
Sub-Tier-2 overview for IP Virtual Server (IPVS) — Linux's in-kernel Layer-4 load balancer. Used by:
- Kubernetes kube-proxy in IPVS mode (default for clusters > ~1K services since K8s 1.11)
- LVS (Linux Virtual Server) deployments serving 100K+ rps
- L4-fronted load balancers in front of HTTP/3, gRPC, custom-protocol backends
- Ad-tech / fintech high-rps frontends where xt_set + DNAT can't keep up

Three forwarding methods:
- **NAT (`IP_VS_CONN_F_MASQ`)** — full-NAT both directions; classic forwarding mode
- **Direct Routing / DR (`IP_VS_CONN_F_DROUTE`)** — L2-rewrite-only; only forward path routed; reply directly from real server (requires L2 adjacency)
- **IP Tunneling / TUN (`IP_VS_CONN_F_TUNNEL`)** — IP-in-IP encap; supports cross-subnet pools

11 scheduling algorithms (round-robin / weighted-round-robin / least-connection / weighted-least-connection / source-hashing / destination-hashing / locality-based-LC / LBLCR / shortest-expected-delay / never-queue / Maglev-hash). Persistent affinity via `IP_VS_SVC_F_PERSISTENT` + per-connection persistence template (PE — persistence engine).

Conntrack integration via `ip_vs_nfct.c` for stateful flows. Cluster sync via `ip_vs_sync.c` (UDP-multicast or TCP-unicast) for active-active or active-standby HA.

Sub-tier-2 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/conntrack-core.md` (IPVS conntrack integration), `net/netfilter/nat-core.md` (NAT-mode forwarding consumes NAT mapping infra).

## Scope

This sub-Tier-2 governs **all** of `/home/doll/linux-src/net/netfilter/ipvs/` (~32 source files). Per-component Tier-3 docs would split by:

- `core` (`ip_vs_core.c` — main NF hook + per-skb dispatch)
- `ctl` (`ip_vs_ctl.c` — userspace control via genetlink/setsockopt)
- `conn` (`ip_vs_conn.c` — per-flow state)
- `xmit` (`ip_vs_xmit.c` — per-method forwarding)
- `proto` (`ip_vs_proto.c` + `_tcp.c` + `_udp.c` + `_sctp.c` + `_ah_esp.c`)
- `schedulers` (11 per-algo modules)
- `app + pe` (application-layer helpers + persistence engines like SIP)
- `sync` (cluster sync over UDP/TCP)
- `est` (per-service rate estimator)
- `nfct` (netfilter conntrack integration)

## Compatibility contract — outline

### IPVS UAPI (genetlink + legacy setsockopt)

`include/uapi/linux/ip_vs.h`:
- Modern: GENL_NL_IPVS family + IPVS_CMD_NEW_SERVICE / DEL_SERVICE / SET_SERVICE / GET_SERVICE / NEW_DEST / DEL_DEST / GET_DEST / NEW_DAEMON / SET_INFO / GET_INFO / NEW_TIMEOUT / GET_TIMEOUT / FLUSH / ZERO
- Legacy: SOL_IP setsockopt with IP_VS_SO_SET_*` / IP_VS_SO_GET_*` (still supported for ipvsadm-legacy)

Used by: `ipvsadm` (CLI), `keepalived` (HA + healthcheck daemon), `kube-proxy` (IPVS mode programs IPVS via Netlink).

Wire format byte-identical so all three work unchanged.

### Forwarding modes

| Mode | Constant | Effect |
|---|---|---|
| `IP_VS_CONN_F_MASQ` | full NAT | Bidirectional NAT; reply path NAT'd; backend behind director |
| `IP_VS_CONN_F_DROUTE` | direct routing | L2 MAC-rewrite-only; reply direct from backend (requires L2 path) |
| `IP_VS_CONN_F_TUNNEL` | IP-in-IP | Outer IP wraps original; cross-subnet pools |
| `IP_VS_CONN_F_LOCALNODE` | local | Real server IS the director |

### 11 scheduling algorithms

| Sched | Symbol | Algorithm |
|---|---|---|
| `rr` | `ip_vs_rr_scheduler` | Round-robin |
| `wrr` | `ip_vs_wrr_scheduler` | Weighted round-robin |
| `lc` | `ip_vs_lc_scheduler` | Least-connection |
| `wlc` | `ip_vs_wlc_scheduler` | Weighted least-connection |
| `sh` | `ip_vs_sh_scheduler` | Source-hashing (per-source consistent) |
| `dh` | `ip_vs_dh_scheduler` | Destination-hashing |
| `lblc` | `ip_vs_lblc_scheduler` | Locality-based least-connection |
| `lblcr` | `ip_vs_lblcr_scheduler` | LBLC w/ replication |
| `sed` | `ip_vs_sed_scheduler` | Shortest expected delay |
| `nq` | `ip_vs_nq_scheduler` | Never-queue |
| `mh` | `ip_vs_mh_scheduler` | Maglev hashing (Google's consistent hash) |
| `ovf` | `ip_vs_ovf_scheduler` | Always-overflow (always pick last) |
| `fo` | `ip_vs_fo_scheduler` | Failover (always pick first available) |
| `twos` | `ip_vs_twos_scheduler` | Power-of-two-choices (random load-aware) |

Identical algorithms.

### `struct ip_vs_service` (virtual service)

```c
struct ip_vs_service {
    struct hlist_node s_list;          /* in service hash table */
    struct hlist_node f_list;          /* in firewall mark hash */
    atomic_t refcnt;
    u16 af;
    __u16 protocol;
    union nf_inet_addr addr;            /* virtual IP */
    __be16 port;
    __u32 fwmark;
    unsigned int flags;
    unsigned int timeout;
    __u32 netmask;
    struct list_head destinations;     /* per-real-server list */
    __u32 num_dests;
    struct ip_vs_stats stats;
    char sched_name[IP_VS_SCHEDNAME_MAXLEN];
    struct ip_vs_scheduler __rcu *scheduler;
    spinlock_t sched_lock;
    void *sched_data;
    char pe_name[IP_VS_PENAME_MAXLEN];
    struct ip_vs_pe *pe;
    struct rcu_head rcu_head;
};
```

Layout-equivalent for first cache-line.

### `struct ip_vs_dest` (real server)

Per-real-server: address + port + weight + connection counter + per-server stats.

### `struct ip_vs_conn` (per-flow state)

Per-flow: client tuple + virtual tuple + real tuple + state + per-direction counters + sync sequence.

### Sync daemon (`ip_vs_sync.c`)

Per-direction master/backup sync via UDP multicast or TCP unicast; per-conn state mirrored to standby director for active-active or active-standby HA.

### IPVS netfilter integration

IPVS hooks at NF_INET_LOCAL_IN (DNAT path) + NF_INET_FORWARD (DR/TUN) + NF_INET_POST_ROUTING (NAT mode reply path). Per-conn `nfct` ref linked to conntrack via `ip_vs_nfct.c`.

### Per-service persistence (PE)

Persistence engines like `ip_vs_pe_sip.c` parse application-layer protocols to bind related flows to same real server (e.g., SIP signaling + RTP media to same backend).

## sub-Tier-3 docs governed (deferred to Phase D)

Per-component split — out of scope for this sub-Tier-2 wrapper as a Tier-2 declaration. IPVS is a self-contained subsystem with its own ABI and is least-changed across kernel versions.

## Compatibility outline (top-level)

- REQ-O1: All `IPVS_CMD_*` genetlink + legacy setsockopt commands parsed identically.
- REQ-O2: All 4 forwarding modes (MASQ / DROUTE / TUNNEL / LOCALNODE) supported per upstream.
- REQ-O3: All 14 scheduling algorithms (per the table) implemented identically.
- REQ-O4: `struct ip_vs_service` + `ip_vs_dest` + `ip_vs_conn` first-cache-line layout-equivalent.
- REQ-O5: NF hook integration at LOCAL_IN / FORWARD / POST_ROUTING per upstream priorities.
- REQ-O6: Conntrack integration via `ip_vs_nfct.c`; per-conn `nfct` ref consistent with conntrack.
- REQ-O7: Sync daemon: UDP-multicast + TCP-unicast modes; identical wire format (IP_VS_SYNCD_*).
- REQ-O8: Per-service stats: per-connection + per-byte counters; rate estimator at IP_VS_RATE_* tickrate.
- REQ-O9: Persistence engines: `ip_vs_pe_sip.c` + framework for adding more.
- REQ-O10: Per-service persistence (`IP_VS_SVC_F_PERSISTENT`): per-source-IP affinity bound to real server within timeout window.
- REQ-O11: Hardening: row-1 features applied per `00-security-principles.md`.

## Acceptance Criteria (top-level)

- [ ] AC-O1: `ipvsadm -A -t 10.0.0.1:80 -s rr` test: NLA-encoded IPVS_CMD_NEW_SERVICE byte-identical to upstream's; service visible in `ipvsadm -L`. (covers REQ-O1)
- [ ] AC-O2: All 14 schedulers selectable: install service with each `-s` algorithm; load-distributed across reals. (covers REQ-O3)
- [ ] AC-O3: NAT mode test: client connects to virtual IP; backend sees director-NAT'd traffic; reply NAT'd back. (covers REQ-O2, REQ-O5)
- [ ] AC-O4: DR mode test: backend has VIP on lo aliased; reply direct from backend; client unaware. (covers REQ-O2)
- [ ] AC-O5: TUN mode test: backend across subnet receives IP-in-IP; reply direct. (covers REQ-O2)
- [ ] AC-O6: Maglev hashing consistency test: same source IP always hashes to same backend; backend removal causes minimal rehash. (covers REQ-O3)
- [ ] AC-O7: Persistence test: `ipvsadm -A -t ... -p 300`; same source IP routed to same backend within 300s window. (covers REQ-O10)
- [ ] AC-O8: Sync test: 2-director cluster (master + backup); per-conn state mirrors via sync; failover preserves connections. (covers REQ-O7)
- [ ] AC-O9: SIP PE test: load `ip_vs_pe_sip`; SIP INVITE + RTP media routed to same backend (signaling + media affinity). (covers REQ-O9)
- [ ] AC-O10: Per-service stats: byte/packet/conn counters byte-identical. (covers REQ-O8)
- [ ] AC-O11: Hardening section per Tier-3 child docs is non-empty (deferred to Phase D for IPVS sub-Tier-3 split). (covers REQ-O11)

## Architecture (top-level)

Each future Tier-3 child doc declares its own Rust module organization. Shared abstractions:

- `kernel::net::netfilter::ipvs::core::Core` — main NF hook entry
- `kernel::net::netfilter::ipvs::Service` — `struct ip_vs_service`
- `kernel::net::netfilter::ipvs::Dest` — `struct ip_vs_dest`
- `kernel::net::netfilter::ipvs::Conn` — per-flow state
- `kernel::net::netfilter::ipvs::Xmit` — per-method forwarding
- `kernel::net::netfilter::ipvs::Sched` — pluggable schedulers
- `kernel::net::netfilter::ipvs::Sync` — cluster sync
- `kernel::net::netfilter::ipvs::Pe` — persistence engines
- `kernel::net::netfilter::ipvs::nfct::NfctIntegration` — conntrack bridge
- `kernel::net::netfilter::ipvs::ctl::Genetlink` — GENL_NL_IPVS API

## Verification (top-level)

### Layer 1: Kani SAFETY proofs

Each future Tier-3 child doc declares its own.

### Layer 2: TLA+ models — mandatory list

(none mandatory at this sub-Tier-2; per-component sub-Tier-3 may add)

### Layer 3: invariant harnesses

Per future sub-Tier-3.

### Layer 4: functional correctness (opt-in)

- **Maglev consistent-hash theorem** via Verus — proves: backend addition/removal causes O(1/N) reshuffling; weights respected.

## Hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this sub-Tier-2

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-service + per-dest + per-conn refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-service + per-dest + per-conn slab caches | § Mandatory |
| **MEMORY_SANITIZE** | freed conn state cleared (carries per-flow tuples + sync state) | § Default-on configurable off |
| **LATENT_ENTROPY** | per-service hash perturbation seeded from kernel CSPRNG (defense vs. predictable backend selection attacks) | § Default-on configurable off |

### Row-1 features consumed by this sub-Tier-2

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, LATENT_ENTROPY**: see above
- **CONSTIFY**: per-scheduler `ip_vs_scheduler` instances + per-protocol `ip_vs_proto_data` `static const`
- **USERCOPY**: NLA + setsockopt parsing uses bound-checked accessors
- **SIZE_OVERFLOW**: per-service stats + sync seq arithmetic uses checked operators
- **KERNEXEC**: per-scheduler dispatch via `static const fn-ptr` arrays

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for IPVS_CMD_NEW_SERVICE.
- Default useful GR-RBAC policy: deny IPVS mutations outside gradm-marked `lb_admin` role; IPVS misuse can mask traffic origin.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none at sub-Tier-2 — defer per-component sub-Tier-3 splits to Phase D)

## Out of Scope

- Per-component sub-Tier-3 split (deferred to Phase D)
- 32-bit-only paths
- Implementation code
