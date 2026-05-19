# Tier-3: net/sched/cls-flower — flower classifier (5-tuple + L2/L3/L4 metadata)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/sched/cls_flower.c
  - include/net/pkt_cls.h
  - include/uapi/linux/pkt_cls.h
-->

## Summary
Tier-3 design for `cls_flower` — the modern, multi-key, HW-offloadable classifier that is the workhorse of every container-networking deployment (Cilium, Calico, Antrea), every Open vSwitch flow-table installation, every nftables / firewalld dataplane integration, and most ECN/QoS-aware traffic classification systems. Matches on 30+ distinct keys per filter: L2 (eth_type, vlan, dst/src MAC), L3 (IPv4/IPv6 dst/src, IP DSCP, TTL, fragment flag), L4 (TCP/UDP dst/src port, flags, MPLS labels), tunnel metadata (VxLAN VNI, GRE key, IP-in-IP outer addresses), conntrack state (zone, mark, label), CT_FLAGS, and metadata fields (skb mark, skb prio).

Each filter installs into a flow-mask-and-key-table: the kernel constructs a "mask" (set of keys we care about) and within that mask matches against the per-filter "key". Multiple filters with same mask coexist via a per-mask hashtable; HW offload pre-compiles masks into NIC TCAM entries.

Sub-tier-3 of `net/sched/00-overview.md`. The most-used Tier-3 in `net/sched/cls_*` because it covers the union of legacy `u32` matching plus modern conntrack-aware matching plus tunnel-aware matching.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| flower classifier: per-mask hashtable, per-filter key match, HW offload, NLA parser | `net/sched/cls_flower.c` |
| Public API | `include/net/pkt_cls.h` |
| UAPI: `TCA_FLOWER_*` keys + flags | `include/uapi/linux/pkt_cls.h` |

## Compatibility contract

### Match keys

`TCA_FLOWER_KEY_*` NLAs (each comes paired with a `..._MASK`):

L2:
- `KEY_ETH_DST` / `_SRC` (6 bytes MAC)
- `KEY_ETH_TYPE` (16-bit)
- `KEY_VLAN_ID` / `_PRIO` / `_TPID` / `_ETH_TYPE`
- `KEY_CVLAN_ID` / `_PRIO` / `_TPID` / `_ETH_TYPE` (inner VLAN, dot1ad)
- `KEY_PPPOE_SID`
- `KEY_PPP_PROTO`
- `KEY_NUM_OF_VLANS`

L3:
- `KEY_IP_PROTO` (8-bit)
- `KEY_IPV4_SRC` / `_DST` (32-bit)
- `KEY_IPV6_SRC` / `_DST` (128-bit)
- `KEY_IP_TOS` / `_TTL`
- `KEY_FLAGS` (IP fragment / firstfrag bits)
- `KEY_ARP_SIP` / `_TIP` / `_OP` / `_SHA` / `_THA`

L4 — TCP/UDP/SCTP:
- `KEY_TCP_SRC` / `_DST`
- `KEY_UDP_SRC` / `_DST`
- `KEY_SCTP_SRC` / `_DST`
- `KEY_TCP_FLAGS`
- `KEY_ICMPV4_TYPE` / `_CODE`
- `KEY_ICMPV6_TYPE` / `_CODE`

L4 — MPLS (up to 7 stacked labels):
- `KEY_MPLS_TTL` / `_BOS` / `_TC` / `_LABEL` (per-stack-entry)

Tunnel metadata:
- `KEY_ENC_KEY_ID` (VxLAN VNI etc.)
- `KEY_ENC_IPV4_SRC` / `_DST`
- `KEY_ENC_IPV6_SRC` / `_DST`
- `KEY_ENC_UDP_SRC_PORT` / `_DST_PORT`
- `KEY_ENC_IP_TOS` / `_TTL`
- `KEY_ENC_FLAGS` (DONT_FRAGMENT etc.)
- `KEY_ENC_OPTS` (geneve / gre-options TLV)
- `KEY_ENC_FLAGS`

Conntrack:
- `KEY_CT_STATE` (NEW / ESTABLISHED / RELATED / INVALID / TRACKED / NAT)
- `KEY_CT_ZONE`
- `KEY_CT_MARK`
- `KEY_CT_LABELS`

Metadata:
- `KEY_FLAGS_MARK` (skb->mark)
- `KEY_FLOW_HASH` (4-byte flow hash)
- `KEY_HASH` (32-bit hash)

L2 inner (post-tunnel):
- `KEY_ENC_OPTS_GENEVE` / `_VXLAN` / `_ERSPAN` (per-tunnel TLV)

Each key has a paired `..._MASK` NLA — partial matching is supported.

Wire format byte-identical so iproute2's `tc filter add ... flower ...` works unchanged.

### Per-mask hashtable

Each unique mask installs a `fl_flow_mask` with a hashtable of per-key-tuple filters. On classify:
1. Walk per-block list of `fl_flow_mask` (one per distinct mask)
2. For each: extract from skb the masked key tuple
3. Hash-lookup in mask's hashtable
4. On hit → invoke action chain via `tcf_action_exec`
5. On miss → next mask

Identical algorithm.

### HW offload via `flow_setup_cb_call`

When `TCA_FLOWER_FLAGS` includes `TCA_CLS_FLAGS_SKIP_SW`: filter is HW-only. When `TCA_CLS_FLAGS_SKIP_HW`: filter is software-only. Default: try-HW-fallback-SW. HW offload via per-block `flow_setup_cb_call(TC_CLSFLOWER_REPLACE, &offload_args)` to driver.

Identical contract.

### `TCA_FLOWER_FLAGS`

| Flag | Constant | Semantic |
|---|---|---|
| `SKIP_HW` | 1 | Software-only |
| `SKIP_SW` | 2 | Hardware-only (reject if not offloadable) |
| `IN_HW` | 4 | Filter is in HW (read-only flag in dumps) |
| `NOT_IN_HW` | 8 | Filter not in HW (read-only) |
| `VERBOSE` | 16 | Extack verbose |

Identical bit values.

### Action chain

Each filter has an associated `tcf_exts` (cross-ref `net/sched/cls-api.md` + `act-api.md`). On match, `tcf_action_exec(skb, exts->actions, exts->nr_actions, &result)` runs the actions in sequence.

### Per-filter dump

`RTM_GETTFILTER` dump returns per-filter NLA-encoded keys + masks + flags + stats. Format byte-identical so `tc -s filter show` works.

### Mask sharing across blocks (shared blocks)

When multiple qdiscs share a `tcf_block` (per `cls-api.md` REQ-3), filters with same mask share the per-mask hashtable across all consumers.

## Requirements

- REQ-1: All `TCA_FLOWER_KEY_*` keys (per the table above) parsed identically; each paired with `..._MASK` NLA for partial matching.
- REQ-2: Per-mask hashtable: per-block list of distinct masks; per-mask hashtable of per-key-tuple filters; identical bucket sizing.
- REQ-3: Classify hot path: per-mask walk + per-mask hash lookup + per-filter action exec; identical algorithm.
- REQ-4: HW offload: `TCA_CLS_FLAGS_SKIP_SW` / `SKIP_HW` / `IN_HW` / `NOT_IN_HW` / `VERBOSE` semantics identical; `flow_setup_cb_call(TC_CLSFLOWER_REPLACE/DESTROY)` driver bridge.
- REQ-5: Tunnel metadata matching: `KEY_ENC_*` keys consume skb's per-skb tunnel-metadata-info (set by upstream tunnel decap path).
- REQ-6: Conntrack matching: `KEY_CT_*` keys consume skb's per-skb conntrack tag (set by `act_ct` action; cross-ref future `net/sched/act-ct.md`).
- REQ-7: MPLS-stack matching: up to 7 stacked label entries; per-entry `KEY_MPLS_TTL/BOS/TC/LABEL` semantics identical.
- REQ-8: VLAN / CVLAN matching: outer + inner 802.1Q tags; per-tag `_ID` / `_PRIO` / `_TPID` / `_ETH_TYPE` semantics identical.
- REQ-9: Action chain integration: per-filter `tcf_exts` invoked on match; identical to other classifiers.
- REQ-10: Per-filter dump: NLA-encoded keys + masks + flags byte-identical.
- REQ-11: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: Single-tuple test: `tc filter add dev eth0 ingress flower dst_ip 10.0.0.5 ip_proto tcp dst_port 8080 action drop`; matching SYN → drop + counter; non-matching → pass. (covers REQ-1, REQ-2, REQ-3)
- [ ] AC-2: Mask test: `tc filter add ... flower dst_ip 10.0.0.0/24 ...`; matches 10.0.0.* but not 10.0.1.*. (covers REQ-1)
- [ ] AC-3: Conntrack test: `tc filter add ... flower ct_state +est action pass; tc filter add ... flower ct_state +inv action drop`; established traffic passes; invalid traffic dropped. (covers REQ-6)
- [ ] AC-4: VxLAN tunnel test: install flower filter matching `enc_dst_ip 10.0.1.1 enc_key_id 100` → matches packets with VxLAN VNI=100 from src 10.0.1.1 only. (covers REQ-5)
- [ ] AC-5: VLAN-inner test: 802.1Q-tagged + 802.1ad-tagged frame; flower filter matching `vlan_id 100 cvlan_id 200` matches both tags. (covers REQ-8)
- [ ] AC-6: MPLS-stack test: 3-label MPLS frame; flower filter matching `mpls_label 100 mpls_ttl 64` on outer label matches; on second-label `mpls_label 200` matches second. (covers REQ-7)
- [ ] AC-7: HW-offload test on supporting NIC (Mellanox ConnectX-5+ / Intel ice/i40e): `tc filter add ... flower ... skip_sw`; on success, filter shows `in_hw` flag; on driver-rejection → ENOTSUPP. (covers REQ-4)
- [ ] AC-8: Action chain test: flower match → mirred action redirect to eth1 → gact action pass; tcpdump on eth1 shows redirected packets. (covers REQ-9)
- [ ] AC-9: Per-filter dump test: 100 flower filters with various keys; `tc filter show` returns all with byte-identical NLA encoding. (covers REQ-10)
- [ ] AC-10: Shared block test: 2 qdiscs sharing TCA_INGRESS_BLOCK=42; install one flower filter on block 42 → applies to both qdiscs' ingress. (covers REQ-2)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-11)

## Architecture

### Rust module organization

- `kernel::net::sched::cls_flower::Flower` — classifier instance
- `kernel::net::sched::cls_flower::Mask` — `struct fl_flow_mask`
- `kernel::net::sched::cls_flower::MaskTable` — per-mask hashtable
- `kernel::net::sched::cls_flower::Filter` — per-filter (per-key-tuple)
- `kernel::net::sched::cls_flower::Keys` — per-key extraction from skb
- `kernel::net::sched::cls_flower::Tunnel` — tunnel metadata extraction
- `kernel::net::sched::cls_flower::Conntrack` — CT state extraction
- `kernel::net::sched::cls_flower::Mpls` — MPLS stack extraction
- `kernel::net::sched::cls_flower::Vlan` — VLAN/CVLAN extraction
- `kernel::net::sched::cls_flower::HwOffload` — `flow_setup_cb_call` bridge
- `kernel::net::sched::cls_flower::Dump` — per-filter NLA emit

### Locking and concurrency

- **Per-block `tcf_block_lock`** (mutex, inherited from cls-api): held during per-mask + per-filter mutation
- **RCU**: classify hot path RCU-side; mutator side under tcf_block_lock with RCU-defer
- **Per-mask `ht_lock`**: per-mask hashtable mutation

### Error handling

- `Err(EEXIST)` — duplicate filter at same mask + key tuple
- `Err(ENOENT)` — RTM_DELTFILTER for non-existent
- `Err(EINVAL)` — bad NLA / invalid combination of keys
- `Err(EOPNOTSUPP)` — TCA_CLS_FLAGS_SKIP_SW + driver doesn't support
- `Err(EPERM)` — non-CAP_NET_ADMIN

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-key extraction from skb (no out-of-bounds; bounded protocol headers) | `kani::proofs::net::sched::cls_flower::extract_safety` |
| Per-mask hashtable lookup (no use-after-free during RCU walk) | `kani::proofs::net::sched::cls_flower::lookup_safety` |
| Tunnel metadata extraction (per-skb tunnel-info pointer non-NULL when expected) | `kani::proofs::net::sched::cls_flower::tunnel_safety` |
| MPLS-stack walk (≤ 7 labels) | `kani::proofs::net::sched::cls_flower::mpls_safety` |
| HW-offload callback bridge | `kani::proofs::net::sched::cls_flower::offload_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/net/cls_chain_eval.tla` from `cls-api.md` for chain-eval atomicity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-block masks list | every mask is structurally distinct from others (no duplicate-mask filters share the same mask object) | `kani::proofs::net::sched::cls_flower::mask_dedup_invariants` |
| Per-mask hashtable | every filter's key when masked equals the bucket-key | `kani::proofs::net::sched::cls_flower::filter_invariants` |
| MPLS stack | `BOS=1` exactly on the bottom-of-stack label (last) | `kani::proofs::net::sched::cls_flower::mpls_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Mask-then-match correctness theorem** via Verus — proves: ∀ skb, classify result equals the highest-priority filter whose `(key & mask) == filter_key & mask`.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-mask + per-filter refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-mask + per-filter slab caches | § Mandatory |
| **SIZE_OVERFLOW** | per-key extraction byte-arithmetic uses checked operators | § Mandatory |
| **CONSTIFY** | `tcf_proto_ops` for flower `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY, SIZE_OVERFLOW**: see above
- **MEMORY_SANITIZE**: freed filter state cleared (carries match keys including potentially-sensitive packet selectors like CT_LABELS)
- **USERCOPY**: NLA parsing uses bound-checked accessors throughout
- **KERNEXEC**: per-key extraction dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook: same as `cls-api.md` (CAP_NET_ADMIN gate).
- Default GR-RBAC policy: empty.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

Baseline kernel-wide hardening leveraged by `cls_flower`:

- **PAX_USERCOPY** — `cls_fl_head`, `cls_fl_filter`, `fl_flow_key`, and `fl_flow_mask` allocations are PAX_USERCOPY-whitelisted; netlink dump cannot bleed adjacent slab.
- **PAX_KERNEXEC** — `cls_fl_ops` vtable and rhashtable callback ops live in `__ro_after_init`; corrupted classifier cannot pivot.
- **PAX_RANDKSTACK** — `fl_classify`, `fl_init`, `fl_change`, `fl_walk` stack frames randomized.
- **PAX_REFCOUNT** — `head->refcnt`, `filter->refcnt`, `mask->refcnt` use saturating `refcount_t`.
- **PAX_MEMORY_SANITIZE** — `fl_destroy_filter`/`fl_mask_free` zero per-filter/mask state on RCU free.
- **PAX_UDEREF** — netlink-attr parsing operates on kernel-side copies; mask/key built kernel-side via `fl_set_key`.
- **PAX_RAP/kCFI** — `cls_fl_ops.classify`/`init`/`destroy`/`change`/`walk` and flow-block-cb dispatch are forward-edge-checked.
- **GRKERNSEC_HIDESYM** — `RTM_GETTFILTER` strips `cls_fl_filter`/`mask` kernel pointers.
- **GRKERNSEC_DMESG** — mask-collision and HW-offload-fail `printk_ratelimited` capped.

cls_flower-specific reinforcement:

- **CAP_NET_ADMIN required** for `RTM_NEWTFILTER` with `kind=flower`; strictly enforced.
- **PAX_REFCOUNT on `mask->refcnt`** — shared mask across many filters cannot wrap on rapid add/del.
- **`fl_flow_key` PAX_USERCOPY-whitelisted** — partial-key dump to userspace cannot exfiltrate adjacent state.
- **`flow_block_cb` registration** for HW-offload paired add/del under PAX_KERNEXEC `__ro_after_init` vtable — stale HW-filter cannot survive cls delete.
- **PAX_RAP on `cls_fl_ops.classify`** — softirq classify dispatch forward-edge-checked.

Rationale: flower is the modern hot-path classifier used by Cilium/OVS HW offload; combining USERCOPY whitelisting on the key, saturating mask refs, and PAX_RAP-checked HW-offload-cb dispatch closes the offload-race and partial-disclosure classes.

## Open Questions

(none — flower semantics exhaustively specified by upstream + iproute2 wire-test corpus + extensive Cilium / OVS interop test coverage)

## Out of Scope

- Other classifiers (cross-ref `net/sched/cls-u32.md`, `cls-bpf.md`)
- Action framework (cross-ref `net/sched/act-api.md`)
- Conntrack underlying mechanism (cross-ref `net/netfilter/00-overview.md` once Tier-2 wrapper is added)
- 32-bit-only paths
- Implementation code
