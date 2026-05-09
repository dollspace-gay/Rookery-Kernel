---
title: "Tier-3: net/netfilter/conntrack-core — per-flow nf_conn lifecycle, hashtable, expectations"
tags: ["design-doc", "tier-3", "net", "netfilter"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the conntrack core: per-flow `nf_conn` allocation + lifecycle (UNCONFIRMED → CONFIRMED → DYING), per-netns conntrack hashtable, per-tuple lookup (orig + reply directions both indexed), expectation tracking (parent flow expecting related-child flow per helper), conntrack extensions (acct, timestamp, labels, secmark, etc. attached via per-extension slabs), and the per-skb conntrack tagging via `skb->_nfct`.

This Tier-3 owns the lifecycle and storage; per-protocol state machines (TCP/UDP/ICMP/SCTP/GRE) are in `conntrack-proto.md`. The two together implement Linux's stateful firewall foundation that every modern container deployment depends on.

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/conntrack-proto.md` (per-proto state machines), `net/netfilter/nat-core.md` (NAT integration), `net/sched/act-ct.md` (tc-side conntrack action — already documented).

### Requirements

- REQ-1: `struct nf_conn` + `struct nf_conntrack_tuple` byte-identical first cache-line layout.
- REQ-2: Per-netns hashtable keyed on `nf_conntrack_tuple`; both orig + reply directions indexed; lookup via RCU; identical bucket sizing.
- REQ-3: Lifecycle: UNCONFIRMED → CONFIRMED → DYING; transitions identical to upstream; refcount-protected.
- REQ-4: Periodic GC + pressure-driven GC: timer-driven every `nf_conntrack_gc_interval`; reaps DYING + expired.
- REQ-5: Expectation tracking: per-netns expect hashtable; helper-driven create; on-match → child conntrack with IP_CT_RELATED + parent pointer.
- REQ-6: Conntrack extensions framework: `nf_ct_ext_alloc` for per-extension slabs (acct/timestamp/labels/secmark/helper/seqadj/synproxy/nat/ecache); identical ext discrimination.
- REQ-7: IP_CT_* per-skb states (NEW / ESTABLISHED / RELATED + REPLY variants) byte-identical encoding in `skb->_nfct`.
- REQ-8: IPS_* status flags byte-identical bit values.
- REQ-9: Conntrack zones (16-bit + direction flags); per-zone separate state tables.
- REQ-10: `/proc/sys/net/netfilter/nf_conntrack_*` sysctls byte-identical defaults + format.
- REQ-11: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct nf_conn` + `struct nf_conntrack_tuple` byte-identical first cache-line. (covers REQ-1)
- [ ] AC-2: Lifecycle test: TCP SYN → UNCONFIRMED → on first SYN-ACK → CONFIRMED → on FIN+ACK exchange + timeout → DYING. (covers REQ-3)
- [ ] AC-3: Hashtable lookup test: install conntrack via TCP SYN exchange; `conntrack -L` shows both directions; lookup by either direction returns the entry. (covers REQ-2)
- [ ] AC-4: GC test: set short timeout (1s); idle the flow → after gc_interval pass, conntrack reaped; subsequent same-tuple traffic creates fresh entry. (covers REQ-4)
- [ ] AC-5: Expectation test: load `nf_conntrack_ftp` helper; FTP control connection PORT command → expectation registered; subsequent DCC TCP connection from announced port matches → child conntrack with IP_CT_RELATED. (covers REQ-5)
- [ ] AC-6: Extension test: enable `nf_conntrack_acct=1`; conntrack carries per-direction byte/packet counters; `conntrack -L` shows them. (covers REQ-6)
- [ ] AC-7: Zone test: create 2 conntracks with same tuple but zone=1 and zone=2; both visible separately in `conntrack -L --zone 1` and `--zone 2`. (covers REQ-9)
- [ ] AC-8: skb->_nfct test: traffic through conntrack tags `skb->_nfct` with appropriate `IP_CT_*` low-bits + nf_conn pointer. (covers REQ-7)
- [ ] AC-9: IPS_ASSURED test: bidirectional TCP flow → IPS_ASSURED set; high-pressure GC keeps ASSURED ahead of unASSURED. (covers REQ-8)
- [ ] AC-10: Sysctl test: `cat /proc/sys/net/netfilter/nf_conntrack_max` shows default; write new value → max updates. (covers REQ-10)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-11)

### Architecture

### Rust module organization

- `kernel::net::netfilter::conntrack::Hashtable` — per-netns conntrack table
- `kernel::net::netfilter::conntrack::NfConn` — `struct nf_conn` wrapper
- `kernel::net::netfilter::conntrack::Tuple` — `struct nf_conntrack_tuple`
- `kernel::net::netfilter::conntrack::Lifecycle` — UNCONFIRMED → CONFIRMED → DYING transitions
- `kernel::net::netfilter::conntrack::Confirm` — `__nf_conntrack_confirm` (CONFIRMED transition + hashtable insert)
- `kernel::net::netfilter::conntrack::Gc` — periodic + pressure-driven garbage collection
- `kernel::net::netfilter::conntrack::Expect` — expectation tracking
- `kernel::net::netfilter::conntrack::Extensions` — per-extension slab framework
- `kernel::net::netfilter::conntrack::Zone` — per-zone isolation
- `kernel::net::netfilter::conntrack::Skb` — `skb->_nfct` tagging
- `kernel::net::netfilter::conntrack::Sysctl` — per-netns sysctls

### Locking and concurrency

- **Per-bucket spinlocks** on the conntrack hashtable
- **Per-`nf_conn` `lock`** (spinlock): per-flow mutable state
- **Per-netns `nf_conntrack_locks_all_lock`** (spinlock): held during full-table-walk operations (e.g., flush)
- **RCU**: lookup hot-path RCU-side; mutators via RCU-defer free
- **Per-bucket lock acquired in canonical order**: prevents deadlock during multi-bucket operations

### Error handling

- `Err(EINVAL)` — malformed tuple
- `Err(ENOMEM)` — slab alloc fail
- `Err(EBUSY)` — `nf_conntrack_max` reached
- `Err(EEXIST)` — confirm conflicts with existing tuple

### Out of Scope

- Per-protocol state machines (cross-ref `net/netfilter/conntrack-proto.md`)
- Conntrack helpers (cross-ref `net/netfilter/conntrack-helper.md`)
- NAT integration (cross-ref `net/netfilter/nat-core.md`)
- Conntrack netlink (cross-ref `net/netfilter/conntrack-netlink.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Conntrack core: nf_conn alloc, hashtable, lookup, confirmation, GC, lifecycle | `net/netfilter/nf_conntrack_core.c` |
| Conntrack extensions framework | `net/netfilter/nf_conntrack_extend.c` |
| Expectation tracking (helpers register expected related flows) | `net/netfilter/nf_conntrack_expect.c` |
| Public API | `include/net/netfilter/nf_conntrack.h`, `include/net/netfilter/nf_conntrack_tuple.h` |
| UAPI: IP_CT_*, IPS_* status flags | `include/uapi/linux/netfilter/nf_conntrack_common.h` |

### compatibility contract

### `struct nf_conntrack_tuple`

```c
struct nf_conntrack_tuple {
    struct nf_conntrack_man src;     /* manipulable: addr + port (u_src) + l3num */
    struct {
        union nf_inet_addr u3;       /* dst addr */
        union {
            __be16 all;               /* generic */
            struct { __be16 port; }   tcp, udp, sctp, dccp;
            struct { u_int8_t type, code; } icmp, icmpv6;
            struct { __be16 key; }    gre;
        } u;
        u_int8_t protonum;
        u_int8_t dir;
    } dst;
};
```

Layout-byte-identical so existing tooling (conntrack-tools / `conntrack -L`) works unchanged.

### `struct nf_conn` layout

`include/net/netfilter/nf_conntrack.h`:
```c
struct nf_conn {
    struct nf_conntrack ct_general;
    spinlock_t lock;
    u32 timeout;
    struct nf_conntrack_zone zone;
    struct nf_conntrack_tuple_hash tuplehash[IP_CT_DIR_MAX];   /* original + reply */
    unsigned long status;     /* IPS_* flags */
    possible_net_t ct_net;
    /* extensions */
    struct nf_ct_ext *ext;
    union nf_conntrack_proto proto;     /* per-proto state union */
};
```

Plus per-extension extras attached via `ext`:
- `acct` (per-direction byte/packet counters)
- `timestamp` (start/stop time)
- `labels` (128-bit per-flow labels)
- `secmark` (LSM secmark + secctx)
- `helper` (per-helper state)
- `seqadj` (TCP sequence-number adjustment for NAT/connection-track-based modifications)
- `synproxy` (SYN-proxy state)
- `nat` (NAT mapping)
- `ecache` (event cache for conntrack-event userspace listeners)

Layout-equivalent for first cache-line.

### Lifecycle

| State | When |
|---|---|
| **UNCONFIRMED** | Just-allocated; first packet of flow being processed |
| **CONFIRMED** | Lookup succeeded → flow added to hashtable; `IPS_CONFIRMED_BIT` set |
| **DYING** | Marked for deletion; `IPS_DYING_BIT` set; awaiting GC |

Transitions:
- alloc → UNCONFIRMED
- UNCONFIRMED → CONFIRMED on successful `nf_conntrack_confirm` (called at NF_INET_LOCAL_OUT/POST_ROUTING for new flows)
- CONFIRMED → DYING on timeout, FIN, RST, hard expiration, GC pressure
- DYING → freed via `nf_ct_destroy` after RCU sync

### Per-netns hashtable

`net->ct.hash[]` — per-netns hashtable keyed on `nf_conntrack_tuple`. Two entries per flow (orig + reply); both indexed → either-direction lookup finds the flow. Bucket count: `nf_conntrack_htable_size` (default `nf_conntrack_max / 4` ≈ 16K-64K typical).

### Lookup / insert / GC

- `nf_conntrack_find_get` — RCU-side lookup by tuple; refcount-acquire on hit
- `__nf_conntrack_confirm` — CONFIRMED transition; insert both directions into hashtable
- `nf_ct_gc_expired` — GC pass: walk and reap DYING + expired entries
- Periodic GC: timer-driven every `nf_conntrack_gc_interval` (default 1s); also pressure-driven on alloc-fail

### Expectation tracking (`nf_conntrack_expect.c`)

Conntrack helpers (FTP / SIP / IRC / etc.) parse application-layer protocol semantics and register `nf_conntrack_expect` entries — declarations like "expecting a TCP connection from src=X dst=Y dport=Z within next 5min". When matching packet arrives, expectation matches → child conntrack created with `IP_CT_RELATED` state + parent pointer.

`struct nf_conntrack_expect`:
- `tuple` (expected tuple)
- `mask` (mask for partial matching)
- `master` (parent nf_conn)
- `helper` (helper that created expectation)
- `class` (per-helper expectation class)
- `flags`, `timeout`

Per-netns hashtable + LRU; expiration timer per-expect.

### IP_CT_* states (per-skb tagging)

`include/uapi/linux/netfilter/nf_conntrack_common.h`:
- `IP_CT_ESTABLISHED` (0): bidirectional traffic seen
- `IP_CT_RELATED` (1): expected child of an established conntrack (e.g., FTP DCC)
- `IP_CT_NEW` (2): just allocated (UNCONFIRMED)
- `IP_CT_IS_REPLY` (3): bit-flag added to ESTABLISHED/RELATED for reply-direction packets
- `IP_CT_ESTABLISHED_REPLY` = 3
- `IP_CT_RELATED_REPLY` = 4
- `IP_CT_NUMBER` = 5

skb's `_nfct` low-bits encode the IP_CT_* enum; high bits hold pointer.

### IPS_* status flags (per-conntrack `nf_conn.status`)

- `IPS_EXPECTED` (1<<0): created from expectation
- `IPS_SEEN_REPLY` (1<<1): reply direction seen
- `IPS_ASSURED` (1<<2): high-confidence flow (don't easily reap under pressure)
- `IPS_CONFIRMED` (1<<3): in hashtable
- `IPS_SRC_NAT` / `_DST_NAT` (1<<4 / 1<<5): NAT applied per direction
- `IPS_SEQ_ADJUST` (1<<6): TCP seq adjustment active
- `IPS_SRC_NAT_DONE` / `_DST_NAT_DONE` (1<<7 / 1<<8)
- `IPS_DYING` (1<<9): marked for deletion
- `IPS_FIXED_TIMEOUT` (1<<10): timeout not extended on traffic
- `IPS_TEMPLATE` (1<<11): template entry (for cttimeout)
- `IPS_UNTRACKED` (1<<12): NOTRACK target was applied
- `IPS_HELPER` (1<<13): helper attached
- `IPS_OFFLOAD` (1<<14): offloaded to flowtable
- `IPS_HW_OFFLOAD` (1<<15): offloaded to hardware

Identical bit values + semantics.

### Conntrack zones (`nf_conntrack_zone`)

Per-conntrack zone (16-bit) for multi-tenant isolation. Same 5-tuple in different zones are distinct conntracks. Per-zone-direction can be specified independently via `flags` field (`NF_CT_ZONE_DIR_ORIG`, `NF_CT_ZONE_DIR_REPL`).

### `/proc/sys/net/netfilter/nf_conntrack_*` sysctls

- `nf_conntrack_max` (default 16384, scaled by RAM) — total per-system conntrack cap
- `nf_conntrack_buckets` — hashtable size
- `nf_conntrack_acct` (default 0) — enable per-conntrack byte/packet accounting
- `nf_conntrack_timestamp` (default 0)
- `nf_conntrack_events` (default 1) — emit conntrack events to ecache listeners
- `nf_conntrack_checksum` (default 1) — verify checksums on conntracked packets
- `nf_conntrack_log_invalid` — log invalid packets
- `nf_conntrack_gc_interval` (default 1000ms)
- `nf_conntrack_helper` (default 0; new default — disabled, must be explicitly attached via CT_TARGET / nft_ct)
- per-proto timeouts: `nf_conntrack_tcp_timeout_*`, `_udp_timeout`, `_icmp_timeout`, etc.

Format byte-identical so `sysctl -a` output matches.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-bucket insert/erase + RCU-side reader (no use-after-free) | `kani::proofs::net::netfilter::conntrack::hash_safety` |
| Lifecycle transitions (UNCONFIRMED → CONFIRMED → DYING) under per-conn lock | `kani::proofs::net::netfilter::conntrack::lifecycle_safety` |
| Periodic GC walk (no use-after-free of just-freed entries) | `kani::proofs::net::netfilter::conntrack::gc_safety` |
| Expectation match → child create (refcount sound across parent+child) | `kani::proofs::net::netfilter::conntrack::expect_safety` |
| Extension slab alloc/free (per-extension lifecycle aligned with parent nf_conn) | `kani::proofs::net::netfilter::conntrack::ext_safety` |

### Layer 2: TLA+ models

(none new at this granularity; per-proto state machines own `models/net/conntrack_state_machine.tla` — see `conntrack-proto.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-netns hashtable | both directions of a confirmed conntrack appear in their respective buckets; UNCONFIRMED conntracks not in hashtable | `kani::proofs::net::netfilter::conntrack::table_invariants` |
| Per-conn refcount | refcount ≥ 1 between alloc and DYING; freed only after refcount=0 + RCU sync | `kani::proofs::net::netfilter::conntrack::refcount_invariants` |
| Expectation list | every active expect has a non-NULL parent (master); on parent destroy, expects cleaned up | `kani::proofs::net::netfilter::conntrack::expect_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `net/netfilter/00-overview.md` Layer 4)

- **Conntrack lifecycle soundness theorem** via TLA+ refinement — proves: implementation refines the abstract lifecycle state-machine.
- **Expectation match-or-expire theorem** via Verus — proves: every registered expect either matches (creates RELATED conntrack) or expires; never lingers.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-`nf_conn` refcount uses `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-`nf_conn` slab cache; per-extension slab caches | § Mandatory |
| **MEMORY_SANITIZE** | freed nf_conn cleared (carries flow 5-tuple + secctx + labels — sensitive on multi-tenant systems) | § Default-on configurable off |
| **SIZE_OVERFLOW** | hashtable bucket arithmetic + extension-offset arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, SIZE_OVERFLOW**: see above
- **CONSTIFY**: per-extension `nf_ct_ext_type` registrations `static const`
- **USERCOPY**: NLA parsing (consumed via netlink Tier-3) uses bound-checked accessors
- **KERNEXEC**: per-extension dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for conntrack mutations (via netlink + sysctl).
- Default useful GR-RBAC policy: deny conntrack-table flush (`conntrack -F`) outside gradm-marked `firewall_admin` role; flush is a state-clearing primitive that may help malicious processes mask flow correlation.
- DoS-prevention: `nf_conntrack_max` cap + per-zone caps prevent unbounded conntrack-table growth via flow-flood attacks.

### Userspace-visible behavior changes

None beyond upstream defaults. Note: upstream changed `nf_conntrack_helper` default to `0` in v4.7+; Rookery preserves that default.

### Verification

(See § Verification above.)

