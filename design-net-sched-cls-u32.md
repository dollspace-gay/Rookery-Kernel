---
title: "Tier-3: net/sched/cls-u32 — u32 classifier (32-bit-key match-tree)"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for `cls_u32` — the original Linux classifier (since kernel 2.4) that matches on 32-bit-aligned values within the packet at user-specified byte offsets. Filters are organized into a tree of `tc_u_hnode` (hashtables) and `tc_u_knode` (filter nodes); each `knode` carries a list of `(offset, mask, value)` selector tuples that must all match. Less expressive than `cls_flower` (no built-in semantic key model — userspace must specify byte offsets for each header field) but unmatched in raw expressiveness — can match on arbitrary packet bytes.

Still heavily used by:
- Distros' default QoS scripts (e.g., `tc-meter` examples, `tc qdisc add ... ingress` + `tc filter add ... u32 match ip src ...` for legacy script libraries)
- Telco / ISP traffic-classification setups predating flower (often vendor-shipped configs)
- Some test-infrastructure for tc that exercises the per-byte matching

Sub-tier-3 of `net/sched/00-overview.md`. Pairs with `net/sched/cls-flower.md` (modern equivalent). Both implement the `tcf_proto_ops` contract from `net/sched/cls-api.md`.

### Requirements

- REQ-1: `struct tc_u32_sel` + `struct tc_u32_key` byte-identical layout.
- REQ-2: All `TCA_U32_*` NLAs (per the table) parsed identically.
- REQ-3: Per-knode classify: walk `sel->keys[]`; per-key `(packet[off] & mask) == val` matching algorithm identical.
- REQ-4: Hash table linking via `TCA_U32_LINK`: per-knode chain into next hnode; identical recursive walk.
- REQ-5: Hash table divisor: power-of-2 ≤ 256; bucket selection per `(packet_bytes_hashed mod divisor)`.
- REQ-6: HW offload: `TCA_CLS_FLAGS_SKIP_SW / SKIP_HW / IN_HW / NOT_IN_HW` semantics identical.
- REQ-7: Per-knode pcnt: per-CPU hit + miss counters; aggregated for `tc -s filter show`.
- REQ-8: Indev match: per-knode optional ingress-iface-name match.
- REQ-9: Action chain integration: per-knode `tcf_exts` invoked on match (cross-ref `cls-api.md`).
- REQ-10: Per-filter dump: NLA-encoded sel + keys + hash + divisor + classid byte-identical.
- REQ-11: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct tc_u32_sel` + `struct tc_u32_key` byte-identical layout. (covers REQ-1)
- [ ] AC-2: Single-key match test: `tc filter add dev eth0 protocol ip parent 1: prio 1 u32 match ip src 1.2.3.4/32 flowid 1:10` → matching packet routed to flowid 1:10. (covers REQ-2, REQ-3)
- [ ] AC-3: Multi-key match test: `tc filter ... u32 match ip src 1.2.3.4 match ip dst 5.6.7.8 match ip dport 8080 0xffff` → all 3 must match. (covers REQ-3)
- [ ] AC-4: Hash table link test: install hnode A with knode linking to hnode B; matching packet through A → recurses into B; B's knode matches → final action fires. (covers REQ-4, REQ-5)
- [ ] AC-5: Divisor test: `tc filter add ... u32 ht 800: link 800: divisor 16` → 16-bucket hnode created; lookup distributes per `(hash mod 16)`. (covers REQ-5)
- [ ] AC-6: HW-offload test on supporting NIC: `... u32 ... skip_sw` → driver-side accept; `IN_HW` flag set; HW-counter increments on traffic. (covers REQ-6)
- [ ] AC-7: pcnt test: 1000 matching packets through filter → `tc -s filter show` reports `hit_count = 1000`. (covers REQ-7)
- [ ] AC-8: Indev test: install filter with `indev eth0`; packet arriving on eth1 → no match. (covers REQ-8)
- [ ] AC-9: Action chain test: `... u32 match ip src 1.2.3.4 action mirred egress redirect dev eth1` → matching packet redirected to eth1. (covers REQ-9)
- [ ] AC-10: Per-filter dump test: `tc filter show dev eth0` byte-identical content for u32 filters. (covers REQ-10)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-11)

### Architecture

### Rust module organization

- `kernel::net::sched::cls_u32::U32Classifier` — classifier instance
- `kernel::net::sched::cls_u32::Hnode` — `struct tc_u_hnode` (hashtable)
- `kernel::net::sched::cls_u32::Knode` — `struct tc_u_knode` (filter node)
- `kernel::net::sched::cls_u32::Sel` — selector + key array
- `kernel::net::sched::cls_u32::ClassifyWalker` — recursive hnode/knode walk
- `kernel::net::sched::cls_u32::Linker` — `TCA_U32_LINK` chain
- `kernel::net::sched::cls_u32::HwOffload` — `flow_setup_cb_call` bridge
- `kernel::net::sched::cls_u32::Pcnt` — per-CPU per-knode counters
- `kernel::net::sched::cls_u32::Indev` — ingress-iface match
- `kernel::net::sched::cls_u32::Dump` — NLA emit

### Locking and concurrency

- **Per-block `tcf_block_lock`** (mutex, inherited): held during per-hnode + per-knode mutation
- **RCU**: classify hot path RCU-side; mutator side under tcf_block_lock with RCU-defer free
- **Per-CPU `pcnt`**: lockless per-CPU counters; aggregated via `READ_ONCE` for dump

### Error handling

- `Err(EEXIST)` — duplicate filter at same priority
- `Err(ENOENT)` — RTM_DELTFILTER for non-existent
- `Err(EINVAL)` — bad NLA / bad sel keys
- `Err(EOPNOTSUPP)` — TCA_CLS_FLAGS_SKIP_SW + driver doesn't support
- `Err(EPERM)` — non-CAP_NET_ADMIN

### Out of Scope

- Modern alternative `cls_flower` (cross-ref `net/sched/cls-flower.md`)
- BPF classifier (cross-ref `net/sched/cls-bpf.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| u32 classifier: tree of hnode + knode, per-knode selector list, classify hot path, NLA parser, dump | `net/sched/cls_u32.c` |
| Public API | `include/net/pkt_cls.h` |
| UAPI: `TCA_U32_*` NLAs + struct tc_u32_sel + struct tc_u32_key | `include/uapi/linux/pkt_cls.h` |

### compatibility contract

### `struct tc_u32_sel` (selector — header of selector list)

```c
struct tc_u32_sel {
    unsigned char         flags;
    unsigned char         offshift;
    unsigned char         nkeys;
    __be16                offmask;
    __u16                 off;
    short                 offoff;
    short                 hoff;
    __be32                hmask;
    struct tc_u32_key     keys[0];
};
```

### `struct tc_u32_key` (per-selector match)

```c
struct tc_u32_key {
    __be32 mask;
    __be32 val;
    int    off;
    int    offmask;
};
```

Each key matches: `(packet[off] & ntohl(mask)) == ntohl(val)`.

Layout-byte-identical so iproute2's `tc filter add ... u32 match` works unchanged.

### NLA configuration

`TCA_OPTIONS` nested NLAs:
- `TCA_U32_CLASSID` (target classid for parent qdisc)
- `TCA_U32_HASH` (handle of containing hash table)
- `TCA_U32_LINK` (link to next hash table — chained matching)
- `TCA_U32_DIVISOR` (hash table size for new tables; power of 2, max 256)
- `TCA_U32_SEL` (struct tc_u32_sel + array of keys)
- `TCA_U32_POLICE` (legacy embedded police action)
- `TCA_U32_ACT` (action chain — replaces POLICE for new code)
- `TCA_U32_INDEV` (interface-name match for ingress)
- `TCA_U32_PCNT` (per-knode hit-counter; read-only)
- `TCA_U32_MARK` (skb-mark match)
- `TCA_U32_FLAGS` (TCA_CLS_FLAGS_SKIP_SW / SKIP_HW / IN_HW / NOT_IN_HW)
- `TCA_U32_PAD`

Wire format byte-identical so iproute2 + tc-test-suite work unchanged.

### Classify hot path

For each skb:
1. Start at `tp->root` (root `tc_u_hnode`)
2. Compute hash from skb's bytes per current hnode's `divisor` + `offmask`/`offoff`
3. Walk knode list at hash bucket
4. For each knode: walk `sel->keys[]` → if all keys match → action / classid → done
5. If knode has `link` → recurse into linked hnode
6. If no match in this hash → terminate with TC_ACT_UNSPEC

Identical recursive walk.

### Hash table linking (`TCA_U32_LINK`)

Optional second-level hashing for performance: a knode can link to another hnode that further classifies after the parent's match. Forms a tree of hash tables — useful for hierarchical match (e.g., outer-IP-mask first, then inner-port within matched outer-IP).

### HW offload (`TCA_CLS_FLAGS_SKIP_SW` / `SKIP_HW`)

Per-knode HW offload via `flow_setup_cb_call(TC_CLSU32_REPLACE/DESTROY, &offload)`. Driver-side optionally accepts the filter; on success → `IN_HW` flag set in dumps.

### Per-knode `tc_u_pcnt` statistics

Per-knode hit + miss counters (per-CPU); aggregated for `tc -s filter show`. Identical format.

### Indev match (`TCA_U32_INDEV`)

Per-knode optional ingress interface match: skb must arrive on the named interface. Used in shared-block setups to scope filters by ingress port.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-knode key array walk (≤ `nkeys`; no out-of-bounds) | `kani::proofs::net::sched::cls_u32::keys_safety` |
| Hnode hash bucket walk | `kani::proofs::net::sched::cls_u32::hash_safety` |
| Linker chain depth bound (no infinite recursion) | `kani::proofs::net::sched::cls_u32::link_safety` |
| Per-key offset arithmetic (skb_pull bounds) | `kani::proofs::net::sched::cls_u32::offset_safety` |
| HW-offload bridge | `kani::proofs::net::sched::cls_u32::offload_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/net/cls_chain_eval.tla` from `cls-api.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-block hnode list | every hnode has unique `handle` | `kani::proofs::net::sched::cls_u32::hnode_invariants` |
| Per-hnode knode list | every knode in hash bucket B has hash matching B | `kani::proofs::net::sched::cls_u32::knode_invariants` |
| Linker chain | per-knode `link` chain has no cycles; total depth ≤ MAX_U32_LINK_DEPTH | `kani::proofs::net::sched::cls_u32::link_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `net/sched/00-overview.md`)

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-hnode + per-knode refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-hnode + per-knode slab caches | § Mandatory |
| **SIZE_OVERFLOW** | per-key offset arithmetic uses checked operators (CVE class: malformed offset → out-of-bounds skb-read) | § Mandatory |
| **CONSTIFY** | `tcf_proto_ops` for u32 `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY, SIZE_OVERFLOW**: see above
- **MEMORY_SANITIZE**: freed knode state cleared (carries match keys)
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **KERNEXEC**: classify dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook: same as `cls-api.md` (CAP_NET_ADMIN gate).
- Default GR-RBAC policy: empty.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

