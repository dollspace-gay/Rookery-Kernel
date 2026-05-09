---
title: "Tier-3: net/sched/em-meta — extended-match meta predicates (cls_basic ematch suite)"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the tc extended-match (ematch) framework + the per-meta predicate library used by `cls_basic`. Ematches are tree-structured boolean expressions over packet+context predicates, allowing `cls_basic` to compose arbitrary AND/OR/NOT/binary-tree expressions of per-predicate matches. Far less common than `cls_flower` in modern setups but indispensable for legacy QoS scripts, AVT (audio-video) trees, and ipset-driven setups that need fast set-membership matching.

The base ematch types:
- **`em_meta`**: predicate over packet metadata (skb mark, priority, protocol, dev name, sk uid, etc.) — the most-used
- **`em_cmp`**: byte-offset comparison (less expressive than u32; rarely used)
- **`em_nbyte`**: N-byte comparison at offset (similar to em_cmp but for arbitrary lengths)
- **`em_canid`**: CAN-bus message ID matching (specialized for vehicle networks)
- **`em_ipset`**: set-membership query against an ipset (for fast O(1)-or-O(log N) per-set-type lookup)
- **`em_ipt`**: invoke an iptables match module (legacy interop)

Sub-tier-3 of `net/sched/00-overview.md`. Pairs with `net/sched/cls-basic.md` (consumer; cls_basic walks the ematch tree). Used in tight composition with `cls_flower`/`cls_u32` filters when those classifiers' built-in keys are insufficient.

### Requirements

- REQ-1: `tcf_em_register` / `tcf_em_unregister` per-kind ops registration; identical contract.
- REQ-2: `struct tcf_ematch_tree_hdr` + `struct tcf_ematch` byte-identical layout.
- REQ-3: Ematch tree evaluation: per-`relations` AND/OR/XOR composition + per-flag INVERT; identical algorithm.
- REQ-4: `em_meta` recognized fields (per the table) — skb / realm / iif / sk / system; identical lookup.
- REQ-5: `em_meta` comparison operators (EQ / LT / GT + INVERT); identical dispatch.
- REQ-6: `em_cmp` byte-offset comparison; `em_nbyte` N-byte comparison; identical algorithms.
- REQ-7: `em_canid` CAN-bus message-ID match; identical to upstream.
- REQ-8: `em_ipset` set-membership lookup against a named ipset; identical UAPI.
- REQ-9: `em_ipt` iptables-match-module invocation; identical legacy bridge.
- REQ-10: NLA wire format for ematch tree (TCF_EM_*) byte-identical.
- REQ-11: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct tcf_ematch_tree_hdr` + `struct tcf_ematch` byte-identical layout. (covers REQ-2)
- [ ] AC-2: cls_basic + em_meta test: `tc filter add dev eth0 protocol ip parent 1: prio 1 basic match 'meta(nf_mark eq 100)' flowid 1:10` → matching nf_mark=100 packet routed to flowid 1:10. (covers REQ-1, REQ-4)
- [ ] AC-3: AND-tree test: `match 'meta(nf_mark eq 100) and meta(priority eq 5)'` → both must match. (covers REQ-3)
- [ ] AC-4: OR-tree with INVERT: `match 'not meta(nf_mark eq 100) or meta(priority eq 5)'` → standard tree-walk; identical semantics. (covers REQ-3)
- [ ] AC-5: em_cmp test: `match 'cmp(u8 at 0 layer 2 eq 0xaa)'` → first byte of L2 header must be 0xaa. (covers REQ-6)
- [ ] AC-6: em_nbyte test: `match 'nbyte(0a 0b 0c at 4 layer 0)'` → bytes 4..7 must equal 0x0a0b0c. (covers REQ-6)
- [ ] AC-7: em_canid test: SocketCAN setup; `match 'canid(0x123)'` → match if CAN ID is 0x123. (covers REQ-7)
- [ ] AC-8: em_ipset test: `ipset create blacklist hash:ip; ipset add blacklist 1.2.3.4`; `tc filter ... basic match 'ipset(blacklist src)'` → matching packet from 1.2.3.4 → drop. (covers REQ-8)
- [ ] AC-9: em_ipt test: `tc filter ... basic match 'ipt(-m state --state ESTABLISHED)'` → invokes iptables xt_state module. (covers REQ-9)
- [ ] AC-10: All NLA wire encodings byte-identical to upstream. (covers REQ-10)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-11)

### Architecture

### Rust module organization

- `kernel::net::sched::ematch::TcfEmatch` — per-match wrapper
- `kernel::net::sched::ematch::Tree` — ematch tree walker (AND/OR/XOR + INVERT)
- `kernel::net::sched::ematch::Registry` — per-kind ops registry
- `kernel::net::sched::ematch::em_meta::Meta` — `em_meta` predicate
- `kernel::net::sched::ematch::em_meta::Fields` — TCF_META_* field accessor
- `kernel::net::sched::ematch::em_cmp::Cmp` — byte-offset compare
- `kernel::net::sched::ematch::em_nbyte::Nbyte` — N-byte compare
- `kernel::net::sched::ematch::em_canid::Canid` — CAN-bus ID match
- `kernel::net::sched::ematch::em_ipset::Ipset` — ipset bridge
- `kernel::net::sched::ematch::em_ipt::Ipt` — iptables-match bridge

### Locking and concurrency

- **`em_mod_lock`** (mutex): per-system ematch-ops registry mutator
- **RCU**: per-classify hot-path is RCU-side (tree walk + per-predicate match)
- **No new global locks**

### Error handling

- `Err(EINVAL)` — bad NLA / unknown ematch kind / bad field ID
- `Err(ENOENT)` — referenced ipset / iptables-match-module not loaded
- `Err(EOPNOTSUPP)` — em_ipt module not available (CONFIG)
- `Err(EPERM)` — non-CAP_NET_ADMIN

### Out of Scope

- ipset implementation (cross-ref `net/netfilter/ipset/00-overview.md` once added)
- iptables xt_match modules (cross-ref `net/netfilter/00-overview.md` once added)
- cls_basic implementation detail (cross-ref future `net/sched/cls-basic.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| `em_meta` predicate (skb metadata + sock fields) | `net/sched/em_meta.c` |
| `em_cmp` (byte-offset compare) | `net/sched/em_cmp.c` |
| `em_canid` (CAN-bus ID match) | `net/sched/em_canid.c` |
| `em_ipset` (ipset set-membership) | `net/sched/em_ipset.c` |
| `em_ipt` (iptables-match invocation) | `net/sched/em_ipt.c` |
| `em_nbyte` (N-byte at offset) | `net/sched/em_nbyte.c` |
| Public API + UAPI | `include/net/pkt_cls.h`, `include/uapi/linux/pkt_cls.h` |

### compatibility contract

### `struct tcf_ematch_tree_hdr` + ematch tree layout

```c
struct tcf_ematch_tree_hdr {
    __u16 nmatches;     /* number of matches in this tree */
    __u16 progid;       /* tree program ID */
};
```

Ematches form a flat array; tree structure encoded via per-match `relations` field (`TCF_EM_REL_AND | _OR | _XOR`) + `flags` field (`TCF_EM_INVERT`, `TCF_EM_SIMPLE`).

Layout-byte-identical so iproute2's `tc filter ... basic match ...` works unchanged.

### `struct tcf_ematch` (per-match descriptor)

```c
struct tcf_ematch {
    struct tcf_ematch_ops *ops;
    unsigned long  data;
    unsigned int   datalen;
    u16            matchid;
    u16            kind;       /* TCF_EM_*: META / CMP / NBYTE / U32 / CANID / IPSET / IPT */
    u16            flags;
    u16            pad;
    struct net    *net;
};
```

Layout-byte-identical.

### `struct tcf_ematch_ops` vtable

Per-ematch-kind operations:
```c
struct tcf_ematch_ops {
    int          kind;
    int          datalen;
    int          (*change)(struct net *, void *, int, struct tcf_ematch *);
    int          (*match)(struct sk_buff *, struct tcf_ematch *, struct tcf_pkt_info *);
    void         (*destroy)(struct tcf_ematch *);
    int          (*dump)(struct sk_buff *, struct tcf_ematch *);
    struct module *owner;
    struct list_head link;
};
```

Per-kind module exports an `ops` instance via `tcf_em_register`/`unregister`.

### `em_meta` recognized fields

`TCF_META_*` IDs carry per-field semantics:
- skb fields: `MARK`, `PRIORITY`, `TC_INDEX`, `PROTOCOL`, `PKTTYPE`, `PKTLEN`, `DATALEN`, `MACLEN`, `NF_MARK`, `IF_INDEX`, `IF_NAME`, `IF_DEV`, `IF_HWADDR`, `IF_GROUP`
- realm: `REALM`
- iif: `RT_IIF`, `RT_REALM`, `RT_PROTOCOL`, `RT_TOS`
- sk fields: `SK_FAMILY`, `SK_STATE`, `SK_REUSE`, `SK_BOUND_IF`, `SK_REFCNT`, `SK_RCVBUF`, `SK_SNDBUF`, `SK_PROTOCOL`, `SK_TYPE`, `SK_RMEM_ALLOC`, `SK_WMEM_ALLOC`, `SK_OMEM_ALLOC`, `SK_WMEM_QUEUED`, `SK_RCV_QLEN`, `SK_SND_QLEN`, `SK_FORWARD_ALLOCS`, `SK_ALLOCS`, `SK_HASH`, `SK_LINGERTIME`, `SK_PRIO`, `SK_RCVLOWAT`, `SK_RCVTIMEO`, `SK_SNDTIMEO`, `SK_SENDMSG_OFF`, `SK_WRITE_PENDING`
- system: `RANDOM`, `LOAD_AVG`, `VLAN_TAG`
- value: `VALUE` (constant)
- string `IF_NAME` / `IF_DEV` / `IF_HWADDR` matches against named iface

Wire format byte-identical so existing `tc filter ... basic match meta(...)` works unchanged.

### Comparison operators

Each ematch carries a comparison operator: `TCF_EM_OPND_EQ | LT | GT`. Combined with `TCF_EM_INVERT` flag for `!=`, `>=`, `<=` semantics.

### `em_ipset` integration

Per-`em_ipset` ematch: links to a named `ip_set` (cross-ref `net/netfilter/ipset/00-overview.md` once added). Per-classify lookup tests skb's source/dest IP (or other field) for set-membership. Significantly faster than per-prefix walk for large sets (10k+ entries).

### `em_canid` (CAN-bus matching)

For SocketCAN traffic — match against CAN message ID. Used in automotive Linux setups.

### `em_ipt` (iptables-match call-out)

Invoke an iptables match module (e.g., `xt_state`, `xt_addrtype`) from within an ematch tree. Legacy bridge between tc filtering + iptables semantics.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Ematch tree walk (AND/OR/XOR composition; bounded depth) | `kani::proofs::net::sched::ematch::tree_safety` |
| em_meta field-accessor (RCU-side per-field read; no out-of-bounds) | `kani::proofs::net::sched::ematch::meta_safety` |
| em_cmp / em_nbyte byte-offset arithmetic | `kani::proofs::net::sched::ematch::cmp_safety` |
| em_ipset bridge (refcount sound on linked ipset) | `kani::proofs::net::sched::ematch::ipset_safety` |
| em_ipt bridge (xt_match invoke under correct hook context) | `kani::proofs::net::sched::ematch::ipt_safety` |

### Layer 2: TLA+ models

(none new owned)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Ematch tree | depth ≤ MAX_EM_TREE_DEPTH; no cycles | `kani::proofs::net::sched::ematch::tree_invariants` |
| Per-kind ops registry | every entry has unique `kind`; module-load doesn't duplicate-register | `kani::proofs::net::sched::ematch::registry_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `net/sched/00-overview.md`)

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-kind `tcf_ematch_ops` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | tree-walk depth + per-match offset arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB**: per-ematch slabs (cross-ref `net/sched/cls-api.md`); per-kind ops module-refcounted
- **CONSTIFY, SIZE_OVERFLOW**: see above
- **MEMORY_SANITIZE**: freed ematch state cleared (carries match data)
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **KERNEXEC**: per-kind dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook: same as `cls-api.md` (CAP_NET_ADMIN gate).
- Default GR-RBAC policy: empty.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

