# Tier-3: net/netfilter — packet-filter framework (nftables, iptables-legacy, conntrack, NAT)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/netfilter/
  - net/netfilter/core.c
  - net/netfilter/nf_tables_api.c
  - net/netfilter/nf_tables_core.c
  - net/netfilter/nf_conntrack_core.c
  - net/netfilter/nf_log.c
  - net/netfilter/nf_queue.c
  - net/netfilter/x_tables.c
  - net/netfilter/ipset/
  - net/ipv4/netfilter/
  - net/ipv6/netfilter/
  - include/linux/netfilter.h
  - include/uapi/linux/netfilter.h
  - include/net/netfilter/nf_tables.h
  - include/net/netfilter/nf_conntrack.h
-->

## Summary
Tier-3 design for the netfilter packet-filter framework: the kernel's hookable packet-mediation engine, its modern (`nf_tables` / nftables) and legacy (`x_tables` / iptables) policy backends, the connection-tracking subsystem (`conntrack`; tracks IP connection state — required for stateful firewalling + NAT), the per-namespace ipset hash machinery, the `NFNL` netlink protocol family, packet logging (`NFLOG`), and queueing (`nfqueue` — packet-to-userspace handoff).

Sub-tier-3 of `net/00-overview.md`. **Mandatory in v0** per `00-overview.md` D3 — every distro ships `nftables` and many run `iptables-legacy`. Most middlebox + firewall infrastructure on the planet runs through this code.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Hook framework + dispatch | `net/netfilter/core.c` |
| nftables (modern) | `net/netfilter/nf_tables_api.c`, `net/netfilter/nf_tables_core.c`, plus `nft_*.c` (per-expression) |
| iptables-legacy (`x_tables`) | `net/netfilter/x_tables.c` plus per-protocol `arp_tables.c`, `ip_tables.c`, `ip6_tables.c` |
| Connection tracking | `net/netfilter/nf_conntrack_core.c`, plus `nf_conntrack_*.c` per-protocol (`tcp`, `udp`, `icmp`, `dccp`, `sctp`, `gre`, `helpers`, `expect`, `acct`, `seqadj`, `extend`, `labels`, `proto_*`, ...) |
| NAT | `net/netfilter/nf_nat_core.c`, plus `nf_nat_*.c` per-protocol |
| Per-arch netfilter | `net/ipv4/netfilter/` + `net/ipv6/netfilter/` |
| ipset | `net/netfilter/ipset/ip_set_*.c` |
| Logging | `net/netfilter/nf_log.c`, `nf_log_*.c` |
| Queueing (packet → userspace) | `net/netfilter/nf_queue.c`, `nfnetlink_queue.c` |
| Public types | `include/linux/netfilter.h`, `include/net/netfilter/nf_tables.h`, `include/net/netfilter/nf_conntrack.h` |
| UAPI | `include/uapi/linux/netfilter.h`, `include/uapi/linux/netfilter/*` |

## Compatibility contract

### Hook points

`include/linux/netfilter.h` defines `enum nf_inet_hooks` per family (NFPROTO_IPV4, IPV6, ARP, BRIDGE, NETDEV, DECNET, INET):
- `NF_INET_PRE_ROUTING=0`, `NF_INET_LOCAL_IN=1`, `NF_INET_FORWARD=2`, `NF_INET_LOCAL_OUT=3`, `NF_INET_POST_ROUTING=4`. Numeric byte-identical.

### nftables ABI

`include/uapi/linux/netfilter/nf_tables.h` defines the netlink messages. Wire format (over NFNL netlink) byte-identical so existing `nft` userspace works. nftables expressions: counter, ct (conntrack-match), exthdr, fib, flow_offload, hash, lookup, masq, match, meta, nat, numgen, objref, payload, queue, range, redir, reject, rt, set, socket, synproxy, tunnel, xfrm, dynset, last. Per-expression byte-identical.

### iptables-legacy ABI

`include/uapi/linux/netfilter_ipv4/ip_tables.h` (and IPv6 / ARP / EBT variants) — `setsockopt`-based control plane (IP_SET_IPT_REPLACE etc.). Byte-identical.

### Conntrack

`/proc/net/nf_conntrack` (per-namespace conntrack table dump) format-identical. Per-flow tuple: `<protocol>     <protonum> <timeout> <state> src=<src_addr> dst=<dst_addr> sport=<sport> dport=<dport> [SEEN_REPLY] mark=<mark> use=<refcount>`.

### NAT

NAT modes: SNAT, DNAT, MASQUERADE, REDIRECT, NETMAP. Each via netfilter target (legacy) or nft `nat` expression (modern). Identical.

### NFNL netlink subsystems

- `NFNL_SUBSYS_NFTABLES` (modern policy)
- `NFNL_SUBSYS_QUEUE` (packet-to-userspace via netlink)
- `NFNL_SUBSYS_ULOG` (packet logging via netlink)
- `NFNL_SUBSYS_OSF` (OS fingerprinting)
- `NFNL_SUBSYS_IPSET` (ipset)
- `NFNL_SUBSYS_ACCT` (accounting helper)
- `NFNL_SUBSYS_CTNETLINK` (conntrack via netlink)
- `NFNL_SUBSYS_HOOK` (hook registration via netlink)

Numeric subsys IDs byte-identical.

## Requirements

- REQ-1: Hook dispatch: `nf_hook` invokes registered hooks at every NF_INET_* point per family; identical priority ordering (NF_IP_PRI_*).
- REQ-2: nftables (`nf_tables_api.c`) wire ABI byte-identical so `nft list ruleset` / `nft -f` work unmodified.
- REQ-3: iptables-legacy (`x_tables`) ABI byte-identical so `iptables-save` / `iptables-restore` work unmodified.
- REQ-4: Conntrack: per-flow state machine (NEW, ESTABLISHED, RELATED, INVALID, UNTRACKED) per upstream's `nf_conntrack_*` per-protocol modules. Hash table + LRU + timeout-driven eviction match upstream.
- REQ-5: NAT: SNAT/DNAT/MASQUERADE/REDIRECT/NETMAP semantics byte-identical.
- REQ-6: ipset: every set type (hash:ip, hash:net, hash:mac, list:set, bitmap:ip, bitmap:port, bitmap:ipmac, …) byte-identical wire-format + behavior.
- REQ-7: NFLOG: packet logging via netlink to userspace (`ulogd`); wire format byte-identical.
- REQ-8: NFQUEUE: packet handoff to userspace queue; identical netlink protocol.
- REQ-9: Per-namespace netfilter state: each network namespace has its own ruleset + conntrack table; CLONE_NEWNET creates fresh netfilter state.
- REQ-10: BPF integration: BPF programs can be invoked at netfilter hooks via flow_offload; flow_offload-driven hardware acceleration preserved.
- REQ-11: Hardware offload (flowtables): conntrack flows offloaded to hardware (when supported); identical decision logic.
- REQ-12: TLA+ model `models/net/conntrack.tla` (per `net/00-overview.md` Layer 2 list) proves connection-tracking entry lifecycle (NEW → ESTABLISHED → CLOSED) under concurrent traversal.
- REQ-13: Existing `nftables`, `iptables`, `ipset`, `conntrack`, `ulogd`, `nfqd` userspace tools work unmodified.
- REQ-14: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `nft list ruleset` byte-identical (modulo dynamic counts) on identical-config Rookery vs. upstream. (covers REQ-2)
- [ ] AC-2: `iptables-save` byte-identical. (covers REQ-3)
- [ ] AC-3: A canonical firewall + NAT setup (e.g., systemd-networkd's NAT for a guest VM) functions identically. (covers REQ-1, REQ-2, REQ-5)
- [ ] AC-4: `cat /proc/net/nf_conntrack` for an active session byte-identical content. (covers REQ-4)
- [ ] AC-5: A SNAT/MASQ test from a private subnet through a NAT box: outbound packets translated; replies map back; identical to upstream behavior. (covers REQ-5)
- [ ] AC-6: `ipset save` / `ipset restore` round-trip byte-identical. (covers REQ-6)
- [ ] AC-7: An NFLOG-tapping `ulogd` test produces identical log entries on Rookery vs. upstream. (covers REQ-7)
- [ ] AC-8: An NFQUEUE userspace verdict test routes packets through userspace + back; identical semantics. (covers REQ-8)
- [ ] AC-9: A `CLONE_NEWNET`-spawned namespace has empty initial ruleset; configuration there doesn't leak to host namespace. (covers REQ-9)
- [ ] AC-10: A BPF-on-netfilter-hook test attaches + filters at a hook. (covers REQ-10)
- [ ] AC-11: A flowtable-offload test on hardware (or an emulator that supports it) shows packets hitting hardware fast-path. (covers REQ-11)
- [ ] AC-12: `make tla` passes `models/net/conntrack.tla`. (covers REQ-12)
- [ ] AC-13: nftables selftests under `tools/testing/selftests/net/netfilter/` pass with the same set as upstream. (covers REQ-1 through REQ-11, REQ-13)
- [ ] AC-14: Hardening section present and follows template. (covers REQ-14)

## Architecture

### Rust module organization

- `kernel::net::netfilter::core::HookDispatch` — nf_hook dispatch
- `kernel::net::netfilter::nft::NfTables` — nftables core
- `kernel::net::netfilter::nft::expressions` — per-expression modules (counter, ct, lookup, meta, nat, payload, set, ...)
- `kernel::net::netfilter::xt::XTables` — iptables-legacy
- `kernel::net::netfilter::conntrack::ConntrackCore` — conntrack core
- `kernel::net::netfilter::conntrack::proto::Tcp` / `Udp` / `Icmp` / `Sctp` / `Gre` / `Dccp` — per-protocol conntrack
- `kernel::net::netfilter::nat::NatCore` — NAT core
- `kernel::net::netfilter::ipset::IpSet` — ipset hash machinery
- `kernel::net::netfilter::log::NfLog` — NFLOG
- `kernel::net::netfilter::queue::NfQueue` — NFQUEUE
- `kernel::net::netfilter::nfnl::Nfnl` — NFNL netlink subsystem dispatch
- `kernel::net::netfilter::flowtable::FlowOffload` — hardware flow offload

### Locking and concurrency

- **`nf_hook_mutex`** — netfilter hook list mutator
- **Per-hook RCU**: hook-list traversal under rcu_read_lock; mutator uses synchronize_rcu
- **`nf_conntrack_lock`** (hashed per-bucket): protects conntrack hash buckets
- **Conntrack-RCU**: lookups RCU-only; mutator under bucket lock
- **`nfnl_lock`**: netfilter-netlink writer lock

TLA+ model `models/net/conntrack.tla` covers the per-flow state machine under racing traversal.

### Error handling

- `Err(ENOENT)` — table/chain/rule not found
- `Err(EEXIST)` — duplicate
- `Err(EBUSY)` — table in use
- `Err(EINVAL)` — bad arg / bad expression
- `Err(EOPNOTSUPP)` — unsupported (e.g., HW offload on non-supporting NIC)
- `Err(ENOMEM)` — alloc failed

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Hook list traversal under RCU | `kani::proofs::net::netfilter::hook_traverse_safety` |
| nftables expression evaluation (per-expression) | `kani::proofs::net::netfilter::nft_eval_safety` |
| Conntrack hash insertion + RCU read | `kani::proofs::net::netfilter::ct_hash_safety` |
| NAT translation (SNAT/DNAT mangle) | `kani::proofs::net::netfilter::nat_xform_safety` |
| ipset hash bucket manipulation | `kani::proofs::net::netfilter::ipset_safety` |

### Layer 2: TLA+ models

- `models/net/conntrack.tla` (mandatory per `net/00-overview.md` Layer 2) — proves per-flow state machine (NEW → ESTABLISHED → CLOSED) under concurrent traversal. Owned here.
- `models/net/nft_lookup.tla` (NEW) — proves nftables lookup correctness under concurrent ruleset replacement (atomic-rule-update).

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Conntrack hash | Each entry exists in exactly one bucket (per `net/00-overview.md` mandatory L3) | `kani::proofs::net::netfilter::ct_hash_invariants` |
| nftables table tree | Per-namespace tree of (family, table, chain, rule); each rule appears at most once in a chain | `kani::proofs::net::netfilter::nft_tree_invariants` |
| ipset hash | Sets indexed by (family, name); each set's hash bucket consistent | `kani::proofs::net::netfilter::ipset_invariants` |

### Layer 4: Functional correctness (opt-in)

- **nftables expression evaluation** via Verus — proves: each expression evaluates per its specification (e.g., `meta mark` reads the right field, `ct state` reads conntrack state correctly).
- **Conntrack state machine** vs. RFC 5382 (TCP) + RFC 7857 (UDP) — Verus opt-in for per-protocol state machines.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | conntrack-entry refcount, nft-table refcount, ipset refcount use `Refcount` | § Mandatory |
| **AUTOSLAB** | Per-protocol-conntrack + nft-table + ipset slab caches each per-type-tagged | § Mandatory |
| **CONFIG_NETFILTER_DEFAULT_ON_BOOT** (default-on hardening — mostly userspace policy) | Distros configure default policy via systemd-networkd / nftables.service | (existing; preserved) |
| **conntrack hash-spray defense** | conntrack hash uses SipHash24 (rather than crc32) per upstream's recent hardening; resists hash-collision DoS | § Default-on (matches upstream) |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-conntrack-entry + per-nft-rule + per-ipset
- **UDEREF**: netlink message parse via `netlink_*` helpers (cross-ref `lib/usercopy.md`)
- **SIZE_OVERFLOW**: rule-byte + offset arithmetic uses checked operators
- **CONSTIFY**: per-expression vtables provided by nft modules are `static const`

### Row-2 / GR-RBAC integration

netfilter is itself a kernel-side firewall — defense-in-depth above LSM. LSM hooks: `security_socket_*` (cross-ref `net/socket-api.md`). GR-RBAC's policy can deny `nfnl` netlink ops (preventing rule modification by less-privileged tasks).

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy on every netlink-message buffer (nftables, ctnetlink, ipset, nfqueue) and every setsockopt-based iptables-legacy ruleset blob.
- **PAX_KERNEXEC** — W^X on nftables JIT and `cls_bpf` programs invoked from netfilter hooks; per-expression evaluators are `static const` arrays.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization on every nfnetlink + setsockopt-based rule mutation.
- **PAX_REFCOUNT** — saturating refcount on every conntrack-entry, nft-table, nft-chain, nft-set, ipset, and flowtable; overflow trips audit + halts the offending writer.
- **PAX_MEMORY_SANITIZE** — zero-on-free for conntrack expectations + helpers (carries NAT translation state), ipset bucket entries, and nft `meta` blobs that may store user data.
- **PAX_UDEREF / PAX_MEMORY_UDEREF** — netlink message parse via `nla_*` accessors that bound-check; rule blobs from iptables-legacy never directly deref user pointers.
- **GRKERNSEC_HIDESYM** — hide kernel pointers in `/proc/net/nf_conntrack`, `/proc/net/ip_tables_*`, nft netlink dumps, audit records, and packet traces.
- **GRKERNSEC_NO_SIMULT_CONNECT** — feeds conntrack: throttles per-uid simultaneous NEW connections to identical 5-tuples (mitigates SYN-flood + conntrack-table-exhaustion DoS).
- **GRKERNSEC_BLACKHOLE** — netfilter DROP target audited at default-rate; silent black-hole mode suppresses audit when target chain is the explicit blackhole sink.
- **GRKERNSEC_RANDNET** — conntrack hash uses SipHash24 with per-boot random key (already upstream); reinforced by gr-random seeding.
- **GRKERNSEC_NETFILTER** — `nfnetlink` writer ops + iptables setsockopt restricted to CAP_NET_ADMIN-in-init-userns (default-on); per-namespace netfilter mutation gated by GR-RBAC subject role.
- **GRKERNSEC_SOCK_PRIV** — AF_NETLINK NFNL bind audited; per-uid rate-limited.
- **PAX_SIZE_OVERFLOW** — rule-byte + offset + per-bucket count arithmetic uses checked operators; ipset hash chain length saturating.
- **CAP_NET_ADMIN** strict — every nfnetlink writer + setsockopt-based mutation enforces CAP_NET_ADMIN in init_user_ns (not just current userns), preventing unprivileged-userns rule manipulation that escapes via parent-ns visible state.

Per-doc rationale: netfilter is itself a kernel-side firewall — it is the defense-in-depth above LSM and below TCP. PaX/Grsecurity reinforcement is especially critical because (a) iptables-legacy still passes raw rule blobs via setsockopt (historical CVE source), (b) conntrack tables are a known DoS amplification surface and require per-bucket saturation, (c) the nfnetlink + nftables JIT pair are JIT engines that PAX_KERNEXEC must keep W^X, and (d) per-namespace netfilter state can be mutated by unprivileged-userns processes unless `CAP_NET_ADMIN` is enforced in init_user_ns.

## Open Questions

(none — netfilter semantics are exhaustively specified by upstream + RFC + IETF documents)

## Out of Scope

- Per-NIC hardware-offload driver (cross-ref `drivers/net/00-overview.md`)
- Specific firewall policies (those are userspace concerns)
- 32-bit-only paths
- Implementation code
