---
title: "Tier-2: net/netfilter — netfilter framework (conntrack, NAT, nftables, flowtable, x_tables)"
tags: ["design-doc", "tier-2", "net", "netfilter"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-2 overview for the netfilter framework — Linux's all-encompassing packet filtering / mangling / NAT / connection-tracking infrastructure. Spans:
- **Hook framework** (`net/netfilter/core.c`): per-NF_INET_*-hook callback chains; per-AF (IPv4/IPv6/Bridge/ARP/DECnet) hook tables registered at `NF_INET_PRE_ROUTING / LOCAL_IN / FORWARD / LOCAL_OUT / POST_ROUTING`
- **Connection tracking** (`net/netfilter/nf_conntrack_*`): per-flow `nf_conn` state with per-protocol state machines (TCP/UDP/ICMP/ICMPv6/SCTP/GRE), conntrack helpers (FTP/SIP/IRC/PPTP/H.323/SANE/TFTP/Amanda/etc.), expectation tracking, conntrack labels, conntrack timestamps, conntrack accounting, conntrack BPF integration, conntrack OVS integration
- **NAT** (`net/netfilter/nf_nat_*`): SNAT/DNAT/MASQUERADE/REDIRECT mapping installation + per-helper NAT (FTP/IRC/SIP/...)
- **nftables** (`net/netfilter/nf_tables_*`, `nft_*`): modern packet-filtering ruleset with VM-style expression evaluation, sets/maps/concats/intervals, dynamic-set timing, flowtable integration; replaces legacy iptables/ip6tables/ebtables/arptables
- **Flowtable** (`net/netfilter/nf_flow_table_*`): per-tuple fast-path that bypasses the full netfilter rule chain for established flows; HW-offloadable
- **Legacy x_tables** (`net/netfilter/x_tables.c`, `net/ipv4/netfilter/ip_tables.c`, etc.): iptables/ip6tables/ebtables/arptables compatibility layer; many distros still ship iptables-nft (iptables-cli backed by nftables in kernel)
- **IPVS** (`net/netfilter/ipvs/`): IP virtual server load balancer (separate Tier-3 family)
- **ipset** (`net/netfilter/ipset/`): named sets of IPs/ports/MACs for fast lookup (consumed by `cls_flower KEY_CT_*`, `act_ct`, nftables, ip_tables)
- **NFLOG / nflog** (`net/netfilter/nf_log_syslog.c`): packet logging
- **netfilter queue** (`net/netfilter/nfnetlink_queue.c`): per-packet handoff to userspace via `nfnetlink_queue` (NFQUEUE)

Sub-tier-2 of `net/00-overview.md`. Many already-written Tier-3s cross-reference netfilter (`net/sched/act-ct.md`, `net/sched/cls-flower.md` `KEY_CT_*` keys, `net/xfrm/state.md` LSM hooks).

### Out of Scope

- Implementation code

### scope

This Tier-2 governs **all** of `/home/doll/linux-src/net/netfilter/` (~150 source files) plus per-AF netfilter directories (`net/ipv4/netfilter/`, `net/ipv6/netfilter/`, `net/bridge/netfilter/`) plus the public API + UAPI headers. Per-file Tier-3 docs live under `.design/net/netfilter/`.

### compatibility contract — outline

### NF hook framework (5 standard hooks per AF)

| Hook | NF_INET_* | When |
|---|---|---|
| `PRE_ROUTING` | 0 | After RX validation, before route lookup |
| `LOCAL_IN` | 1 | After route resolves to local |
| `FORWARD` | 2 | After route resolves to remote |
| `LOCAL_OUT` | 3 | TX from local socket, before route |
| `POST_ROUTING` | 4 | Just before driver xmit |

Each AF (IPv4/IPv6/Bridge/ARP/DECnet/Netdev) registers its own per-hook chain. Per-hook callbacks return one of:
- `NF_ACCEPT` (continue)
- `NF_DROP` (discard)
- `NF_QUEUE` (handoff via NFQUEUE)
- `NF_STOLEN` (callback owns it now)
- `NF_REPEAT` (re-run callback chain)

Wire format byte-identical so existing rule chains work unchanged.

### Connection tracking

`struct nf_conn` per-flow state (per-tuple):
- `tuplehash[2]` (original + reply tuple)
- `status` (CONFIRMED / SEEN_REPLY / EXPECTED / SRC_NAT_DONE / DST_NAT_DONE / etc.)
- `proto` (per-proto state — TCP state, UDP timeout, ICMP id, etc.)
- `helper` (per-helper state)
- `master` (parent for related flows)
- `mark`, `labels`, `secmark`, `secctx`, `zone`
- `acct[]` (per-direction byte/packet counters)
- `timestamp[2]` (start/stop)
- `timeout`

Identical layout for the first cache-line.

### NAT

Per-`nf_conn` NAT mapping (when SNAT/DNAT applied): rewrites src/dst per `nf_nat_setup_info`. Bidirectional translation maintained via tuple-pair lookup; reverse-direction packets de-NAT'd transparently.

### nftables (modern rule engine)

Per-table → per-chain → per-rule structure:
- Tables: family ∈ {ip, ip6, inet, arp, bridge, netdev}; named
- Chains: per-table; type ∈ {filter, nat, route}; hook + priority; per-AF auto-registration
- Rules: per-chain; per-rule expressions form a VM-style program
- Sets: per-table; typed (IP, MAC, port, concatenation, interval); fast lookup
- Maps: per-table set with associated values (verdict, mark, etc.)
- Flowtables: per-table; for fast-path offload

Wire format byte-identical (NETLINK_NETFILTER family, subsys NFNL_SUBSYS_NFTABLES) so `nft` userspace works unchanged.

### Legacy x_tables (iptables/ip6tables/ebtables/arptables)

Per-table → per-chain → per-rule structure (older model):
- Tables: filter/nat/mangle/raw/security
- Chains: built-in (per-hook) or user-defined
- Rules: header match (struct ipt_entry / ip6t_entry / ebt_entry) + per-match xt_match modules + per-target xt_target

Wire format byte-identical so `iptables-legacy`/`ip6tables-legacy` (vs. `iptables-nft`) work identically; modern distros run iptables-nft which translates to nftables in-kernel.

### Flowtable (fast-path)

Per-tuple `nf_flow_offload` entry — when conntrack sees an established TCP/UDP flow, optionally insert into per-table flowtable. Subsequent packets of that flow bypass conntrack-lookup + nftables/iptables rule chain entirely; routed via `nf_flow_offload_xmit`. HW offload via per-driver flowtable callbacks.

Identical contract.

### IPVS (load balancer)

`net/netfilter/ipvs/` — Layer-4 load balancing: per-virtual-service, per-real-server pool, persistence, scheduling algorithms (rr/wrr/lc/wlc/sh/dh/lblc/lblcr/sed/nq/...). Separate Tier-3 family (`net/netfilter/ipvs/00-overview.md` once added).

### ipset (named sets)

`net/netfilter/ipset/` — named IP/port/MAC sets with multiple set-types (hash:ip / hash:net / hash:ip,port / list:set / etc.) and per-set lookup speed up to O(1). Consumed by tc `act_ct`, `em_ipset`, `cls_flower`, nftables `lookup` expression, iptables `xt_set` match.

### NFLOG + nflog

Per-rule logging via NFLOG target → `nfnetlink_log` per-AF; userspace daemon (ulogd) consumes via NFNL_SUBSYS_ULOG socket.

### NFQUEUE

Per-rule packet handoff to userspace via `nfnetlink_queue` for verdicts.

### tier-3 docs governed by this tier-2

(Phase C will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `net/netfilter/core.md` | Hook framework (`core.c`) |
| `net/netfilter/conntrack-core.md` | Conntrack hashtable + per-flow lifecycle |
| `net/netfilter/conntrack-proto.md` | Per-proto state machines (TCP/UDP/ICMP/SCTP/GRE) |
| `net/netfilter/conntrack-helper.md` | Conntrack helper modules (FTP/SIP/IRC/H.323/PPTP/...) |
| `net/netfilter/conntrack-expect.md` | Expectation tracking |
| `net/netfilter/conntrack-netlink.md` | NFNL_SUBSYS_CTNETLINK conntrack control |
| `net/netfilter/conntrack-bpf.md` | BPF kfunc set for conntrack |
| `net/netfilter/nat-core.md` | NAT mapping installation + per-direction translation |
| `net/netfilter/nat-helpers.md` | Per-helper NAT (FTP/IRC/SIP/...) |
| `net/netfilter/nft-api.md` | nftables NETLINK API + table/chain/rule mutation |
| `net/netfilter/nft-core.md` | nftables VM evaluation engine |
| `net/netfilter/nft-expressions.md` | nft_payload, nft_meta, nft_immediate, nft_lookup, etc. |
| `net/netfilter/nft-sets.md` | nft_set_hash + nft_set_rbtree set implementations |
| `net/netfilter/flowtable.md` | nf_flow_table fast-path + HW offload |
| `net/netfilter/nf-queue.md` | NFQUEUE userspace handoff |
| `net/netfilter/nf-log.md` | NFLOG packet logging |
| `net/netfilter/x-tables.md` | x_tables compatibility layer |
| `net/netfilter/iptables.md` | iptables/ip6tables legacy iptables core |
| `net/netfilter/ebtables.md` | ebtables Ethernet bridge filtering |
| `net/netfilter/arptables.md` | arptables ARP filtering |
| `net/netfilter/ipset/00-overview.md` | ipset named-set framework (sub-Tier-2) |
| `net/netfilter/ipvs/00-overview.md` | IPVS load balancer (sub-Tier-2) |

### compatibility outline (top-level)

- REQ-O1: NF hook framework: 5 standard hooks per AF; verdict constants `NF_ACCEPT/DROP/QUEUE/STOLEN/REPEAT` byte-identical.
- REQ-O2: Conntrack hashtable + per-flow lifecycle: per-tuple insert/lookup/expire; identical algorithm.
- REQ-O3: Per-proto state machines (TCP per RFC 793/6298, UDP, ICMP, SCTP, GRE) byte-identical to upstream.
- REQ-O4: NAT: SNAT / DNAT / MASQUERADE / REDIRECT mapping installation + bidirectional translation; identical.
- REQ-O5: nftables NETLINK API (NFNL_SUBSYS_NFTABLES) wire format byte-identical so `nft` userspace works.
- REQ-O6: nftables VM evaluation: per-rule expression chain; identical opcode set + dispatch.
- REQ-O7: Flowtable fast-path: per-tuple offload + HW offload bridge identical.
- REQ-O8: x_tables / iptables / ip6tables / ebtables / arptables wire format byte-identical for legacy ABI.
- REQ-O9: ipset NETLINK API (NFNL_SUBSYS_IPSET) wire format byte-identical.
- REQ-O10: IPVS NETLINK API (NFNL_SUBSYS_IPVS / GENL_NL_*) wire format byte-identical.
- REQ-O11: Per-Tier-3 child documents define per-component requirements — total netfilter ABI fidelity covered cumulatively.
- REQ-O12: TLA+ models declared at this Tier-2 (see § Verification — Layer 2 below).
- REQ-O13: Hardening: row-1 features applied; default-on configurable per `00-security-principles.md`.

### acceptance criteria (top-level)

- [ ] AC-O1: `nft list ruleset` byte-identical content after equivalent setup. (covers REQ-O5)
- [ ] AC-O2: `iptables-legacy -L` byte-identical after equivalent setup (legacy ABI). (covers REQ-O8)
- [ ] AC-O3: `conntrack -L` byte-identical content. (covers REQ-O2)
- [ ] AC-O4: NAT test: setup masquerade rule via nftables; outbound connection NAT'd; reverse path correctly de-NAT'd. (covers REQ-O4)
- [ ] AC-O5: Flowtable test: install flowtable; established flow → hits fast path on subsequent packets; tcpdump verifies skipped chain. (covers REQ-O7)
- [ ] AC-O6: ipset test: `ipset create blacklist hash:ip; ipset add blacklist 1.2.3.4; nft ... ip saddr @blacklist drop` → matching packets dropped. (covers REQ-O9)
- [ ] AC-O7: IPVS test: `ipvsadm -A -t 10.0.0.1:80 -s rr; ipvsadm -a -t 10.0.0.1:80 -r 10.0.0.10:80`; load balanced across real servers. (covers REQ-O10)
- [ ] AC-O8: Per-Tier-3 ACs cumulatively cover the netfilter ABI surface. (meta-AC)
- [ ] AC-O9: TLA+ models per § Verification all pass `make tla`. (covers REQ-O12)
- [ ] AC-O10: Hardening section per Tier-3 child docs is non-empty. (covers REQ-O13)

### architecture (top-level)

Each Tier-3 child doc declares its own Rust module organization. The shared abstractions:

- `kernel::net::netfilter::core::Hook` — per-AF/per-hook callback chain
- `kernel::net::netfilter::conntrack::NfConn` — per-flow state
- `kernel::net::netfilter::conntrack::Hashtable` — per-netns conntrack table
- `kernel::net::netfilter::nat::NfNat` — NAT mapping
- `kernel::net::netfilter::nft::Table`, `Chain`, `Rule`, `Set`, `Flowtable`
- `kernel::net::netfilter::nft::Vm` — VM evaluation engine
- `kernel::net::netfilter::flowtable::Offload` — flowtable fast-path
- `kernel::net::netfilter::xtables::Compat` — legacy x_tables bridge

### verification (top-level)

### Layer 1: Kani SAFETY proofs

Each Tier-3 child doc declares its own.

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/net/conntrack_state_machine.tla` | `net/netfilter/conntrack-proto.md` (per-proto state machines — TCP per RFC 6298 follows the spec; UDP/ICMP/SCTP/GRE per their RFCs; transitions are sound under concurrent RX/TX) |
| `models/net/nat_translation.tla` | `net/netfilter/nat-core.md` (NAT-translation invariant: bidirectional mapping is consistent; reverse path de-NAT correctly inverts; concurrent flows don't share state) |
| `models/net/flowtable_offload.tla` | `net/netfilter/flowtable.md` (flowtable fast-path correctness: a packet matching a flowtable entry produces output equivalent to the full conntrack+rule-chain path; HW-offloaded entries match SW path) |
| `models/net/nft_vm_eval.tla` | `net/netfilter/nft-core.md` (nftables VM evaluation atomicity: per-rule expression chain executes deterministically; concurrent ruleset transactions are atomic via NFT_MSG_NEWRULE/COMMIT) |

### Layer 3: invariant harnesses

Per Tier-3.

### Layer 4: functional correctness (opt-in)

- **TCP-conntrack RFC compliance theorem** via TLA+ refinement — proves: kernel TCP-state-machine refines RFC 793/6298 + 5681 + 7414 reordering rules.
- **NAT bidirectional-mapping theorem** via Verus — proves: ∀ NAT'd flow, `(orig.src, orig.dst, orig.sport, orig.dport)` maps to consistent `(nat.src, nat.dst, nat.sport, nat.dport)` and reverse direction inverts correctly.
- **nftables transaction atomicity theorem** via TLA+ refinement — proves: NFT_MSG_BEGIN..COMMIT is atomic (concurrent classify never sees partial-rule state).

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | nf_conn, nft_table/chain/rule/set/expr, nf_flow_offload entries use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-type slab caches | § Mandatory |
| **MEMORY_SANITIZE** | freed nf_conn cleared (carries flow 5-tuple + labels + secctx); freed nft state cleared (carries policy decisions) | § Default-on configurable off |
| **SIZE_OVERFLOW** | conntrack hashtable + flowtable + nftables expression-eval arithmetic uses checked operators (CVE class: CVE-2022-class nft-set-overflow + ip_tables comparison-overflow) | § Mandatory |

### Row-1 features consumed by this Tier-2

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, SIZE_OVERFLOW**: see above
- **CONSTIFY**: per-AF nf_hooks ops, nft expression ops, x_tables match/target ops `static const`
- **USERCOPY**: NLA/NETLINK parsing uses bound-checked accessors
- **KERNEXEC**: per-expression dispatch via `static const fn-ptr` arrays

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for all nft/iptables/conntrack mutations.
- LSM hook `security_capable(CAP_BPF)` for nf_conntrack_bpf, nf_flow_table_bpf, nf_nat_bpf integration.
- LSM hook `security_skb_classify_flow` (already standard) — per-flow LSM tagging.
- Default useful GR-RBAC policy: deny netfilter mutations outside gradm-marked `firewall_admin` role; netfilter is the highest-privilege firewall surface.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

