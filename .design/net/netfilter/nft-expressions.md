# Tier-3: net/netfilter/nft-expressions — per-expression types (payload/meta/cmp/...)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/netfilter/nft_payload.c
  - net/netfilter/nft_meta.c
  - net/netfilter/nft_immediate.c
  - net/netfilter/nft_lookup.c
  - net/netfilter/nft_cmp.c
  - net/netfilter/nft_bitwise.c
  - net/netfilter/nft_byteorder.c
  - net/netfilter/nft_numgen.c
  - net/netfilter/nft_ct.c
  - net/netfilter/nft_counter.c
  - net/netfilter/nft_nat.c
  - net/netfilter/nft_log.c
  - net/netfilter/nft_dynset.c
  - net/netfilter/nft_objref.c
  - net/netfilter/nft_quota.c
  - net/netfilter/nft_limit.c
  - net/netfilter/nft_redir.c
  - net/netfilter/nft_masq.c
  - net/netfilter/nft_socket.c
  - net/netfilter/nft_xfrm.c
  - net/netfilter/nft_exthdr.c
  - net/netfilter/nft_hash.c
  - net/netfilter/nft_range.c
  - net/netfilter/nft_reject.c
  - net/netfilter/nft_fib.c
  - net/netfilter/nft_flow_offload.c
  - net/netfilter/nft_fwd_netdev.c
  - net/netfilter/nft_dup_netdev.c
  - net/netfilter/nft_queue.c
  - net/netfilter/nft_compat.c
  - net/netfilter/nft_inner.c
  - net/netfilter/nft_last.c
  - net/netfilter/nft_osf.c
  - net/netfilter/nft_connlimit.c
-->

## Summary
Tier-3 design for the catalog of nftables VM expression types — every per-rule expression that the VM eval engine (cross-ref `net/netfilter/nft-core.md`) can dispatch. Each is a small per-file module exporting a `nft_expr_ops` (cross-ref nft-core.md REQ-2) plus its `nft_expr_type` registration. The expression types form the user-visible language exposed to userspace via NFTA_EXPR_NAME / DATA NLAs (parsed by `nft-api.md`).

This Tier-3 covers the full catalog rather than per-expression sub-Tier-3s — each expression is small, self-contained, and follows a uniform pattern, so consolidation is reasonable. A future Tier-4 could split per-expression if any specific expression accumulates enough complexity.

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/nft-core.md` (consumer; VM eval engine), `net/netfilter/nft-api.md` (NLA parser), `net/netfilter/nft-sets.md` (consumed by `lookup` + `dynset` expressions).

## Upstream references in scope

| Expression | Upstream path | Purpose |
|---|---|---|
| `payload` | `nft_payload.c` | Read/write packet bytes at L2/L3/L4 offset into a register |
| `meta` | `nft_meta.c` | Read skb/sk metadata fields (mark, priority, iif, oif, protocol, etc.) into a register |
| `immediate` | `nft_immediate.c` | Load constant value into register; or set verdict (final action) |
| `cmp` | `nft_cmp.c` | Compare register against constant: == / != / < / <= / > / >= |
| `bitwise` | `nft_bitwise.c` | Bit-level operations (AND, OR, XOR, LSHIFT, RSHIFT) into a register |
| `byteorder` | `nft_byteorder.c` | NTOH / HTON conversions on register |
| `lookup` | `nft_lookup.c` | Look up register value in a set; on hit, store associated value (for maps) and/or apply verdict |
| `dynset` | `nft_dynset.c` | Dynamically add element to set (rate-limit, connection-track table, dynamic block-list) |
| `objref` | `nft_objref.c` | Reference a stateful object (counter / quota / limit / synproxy / ct-helper) by name or set-element |
| `range` | `nft_range.c` | Compare register against [low, high] range |
| `numgen` | `nft_numgen.c` | Generate sequential or random number for stateful loadbalance (`numgen inc mod 5`) |
| `hash` | `nft_hash.c` | Compute jhash or symhash of register; useful for stateless loadbalance |
| `ct` | `nft_ct.c` | Read or set conntrack metadata (state, mark, label, helper, secctx) |
| `ct_fast` | `nft_ct_fast.c` | Optimized fast-path for common ct-state checks |
| `counter` | `nft_counter.c` | Per-rule per-CPU bytes/packets counter |
| `quota` | `nft_quota.c` | Per-rule lifetime byte cap + verdict on overflow |
| `limit` | `nft_limit.c` | Per-rule rate-limit (token bucket — packets-per-time or bytes-per-time) |
| `connlimit` | `nft_connlimit.c` | Per-source-IP / per-saddr connection-count cap (stateful) |
| `nat` | `nft_nat.c` | SNAT/DNAT verdict (consumes from register chosen target) |
| `redir` | `nft_redir.c` | REDIRECT verdict (DNAT to local) |
| `masq` | `nft_masq.c` | MASQUERADE verdict (SNAT from outbound iface) |
| `log` | `nft_log.c` | Emit log via NFLOG / nflog with prefix + level |
| `reject` | `nft_reject.c` | Send ICMP unreachable / TCP RST / explicit reject reply |
| `reject_inet` / `reject_netdev` | `nft_reject_inet.c` / `nft_reject_netdev.c` | Per-AF reject variants |
| `fib` | `nft_fib.c` | Lookup FIB (route table) for skb tuple; verdicts based on result |
| `fib_inet` / `fib_netdev` | `nft_fib_inet.c` / `nft_fib_netdev.c` | Per-AF FIB variants |
| `flow_offload` | `nft_flow_offload.c` | Add flow to flowtable (`flow add @ft`) |
| `fwd_netdev` | `nft_fwd_netdev.c` | Forward to specified netdev (egress redirect) |
| `dup_netdev` | `nft_dup_netdev.c` | Mirror copy to netdev (egress mirror) |
| `queue` | `nft_queue.c` | Queue packet to NFQUEUE for userspace verdict |
| `socket` | `nft_socket.c` | Look up local socket for skb tuple; verdicts based on socket fields |
| `xfrm` | `nft_xfrm.c` | Read xfrm SA fields (SPI, REQID, etc.) into register |
| `exthdr` | `nft_exthdr.c` | Read IPv6 extension header / TCP option fields into register |
| `osf` | `nft_osf.c` | Operating-system fingerprint match (TCP-SYN-based identification) |
| `last` | `nft_last.c` | Per-rule last-match-timestamp (for nft monitor / unused-rule detection) |
| `inner` | `nft_inner.c` | Match against inner packet of tunneled flows (VxLAN / GENEVE / GRE inner header) |
| `compat` | `nft_compat.c` | xt_match / xt_target compatibility wrapper for legacy iptables-style modules |

Sub-tier-3 of `net/netfilter/00-overview.md`.

## Compatibility contract

Each expression type registers via `nft_register_expr(&type)` per upstream module init. The `nft_expr_type` carries:

```c
struct nft_expr_type {
    const struct nft_expr_ops *(*select_ops)(const struct nft_ctx *, const struct nlattr * const tb[]);
    const struct nft_expr_ops *ops;
    const struct nft_expr_ops *inner_ops;       /* for nested-context types like nft_inner */
    struct list_head list;
    const char *name;
    struct module *owner;
    const struct nla_policy *policy;
    unsigned int maxattr;
    u8 family;
    u8 flags;
};
```

Each expression's NLA schema (per `policy[]`) defines its userspace-visible parameters. Wire format byte-identical so `nft` userspace works.

### High-frequency expressions (covered in detail)

#### `payload` (most-used: every rule that examines packet headers)

```
NFTA_PAYLOAD_DREG    - destination register
NFTA_PAYLOAD_BASE    - NFT_PAYLOAD_LL_HEADER / NETWORK_HEADER / TRANSPORT_HEADER / INNER_HEADER
NFTA_PAYLOAD_OFFSET  - byte offset within the chosen header
NFTA_PAYLOAD_LEN     - byte length to copy
NFTA_PAYLOAD_SREG    - source register (for write-mode)
NFTA_PAYLOAD_CSUM_TYPE - checksum-update type (NFT_PAYLOAD_CSUM_NONE / INET / SCTP)
NFTA_PAYLOAD_CSUM_OFFSET - checksum-byte-offset for incremental update
NFTA_PAYLOAD_CSUM_FLAGS  - checksum-update flags
```

Read mode: copies `len` bytes from skb at `base+offset` into register `dreg`. Write mode: copies from `sreg` to skb (with checksum incremental update if applicable).

Identical wire format.

#### `meta` (skb + sk metadata)

`NFTA_META_KEY` enum:
- `NFT_META_LEN / NFPROTO / L4PROTO / PRIORITY / MARK / IIF / OIF / IIFNAME / OIFNAME / IIFTYPE / OIFTYPE / SKUID / SKGID / NFTRACE / RTCLASSID / SECMARK / NFPROTO / PROTOCOL / HOUR / DAY / CGROUP / PRANDOM / SECPATH / IIFKIND / OIFKIND / DEVGROUP / IIF_GROUP / OIF_GROUP / CGROUPV2 / TIME_NS / TIME_DAY / TIME_HOUR / SDIF / SDIFNAME / BRI_IIFNAME / BRI_OIFNAME / BRI_IIFPVID / BRI_IIFVPROTO / IBRPVID / IBRVPROTO / IPSEC`

(35+ keys.) Each reads the corresponding skb / sk / cgroup / ipsec field into register `dreg`.

Identical key set + accessors.

#### `cmp` (compare)

```
NFTA_CMP_SREG  - register to compare
NFTA_CMP_OP    - NFT_CMP_EQ / NEQ / LT / LTE / GT / GTE
NFTA_CMP_DATA  - constant to compare against
```

Identical comparison set.

#### `lookup` (set membership query — used by all set-based filters)

```
NFTA_LOOKUP_SET     - target set name
NFTA_LOOKUP_SREG    - register holding key
NFTA_LOOKUP_DREG    - destination register for value (maps only)
NFTA_LOOKUP_FLAGS   - NFT_LOOKUP_F_INV (invert result)
NFTA_LOOKUP_SET_ID  - per-batch set ID
```

Hit → continue; miss → NFT_BREAK.

#### `dynset` (dynamic set update)

```
NFTA_DYNSET_SET_NAME - target set
NFTA_DYNSET_SREG_KEY - register for key
NFTA_DYNSET_SREG_DATA - register for value (maps)
NFTA_DYNSET_OP - NFT_DYNSET_OP_ADD / DELETE / UPDATE
NFTA_DYNSET_TIMEOUT - per-element timeout
NFTA_DYNSET_EXPR - nested expressions (e.g., per-element counter)
```

Used for stateful per-flow tracking (flow rate-limit, dynamic block list).

### Other expressions

Per-expression NLA schemas + per-expression `eval` callbacks per upstream. All wire-format byte-identical.

## Requirements

- REQ-1: All listed expression types (per the table above) registered via `nft_register_expr`; per-type `nft_expr_type` layout-identical to upstream.
- REQ-2: Per-type NLA schemas (per `policy[]`) byte-identical wire format.
- REQ-3: `payload`: read/write modes; per-base header positioning (LL/NETWORK/TRANSPORT/INNER); checksum-incremental update on write.
- REQ-4: `meta`: 35+ key types (per the list); each reads the corresponding field into register identically.
- REQ-5: `cmp`: 6 comparison ops (EQ/NEQ/LT/LTE/GT/GTE); identical comparison semantics.
- REQ-6: `lookup`: set-membership query; hit → continue + (for maps) load value; miss → NFT_BREAK; INV flag inverts.
- REQ-7: `dynset`: dynamic set add/delete/update with optional per-element timeout + nested expressions; identical semantics.
- REQ-8: `ct` + `ct_fast`: 30+ ct-key types (state/mark/label/helper/secctx/zone/protoinfo/avg-pkt/byte/packet etc.); identical accessors.
- REQ-9: `nat` / `redir` / `masq`: integrate with `nf_nat_setup_info` (cross-ref `net/netfilter/nat-core.md`); identical NAT-mode semantics.
- REQ-10: `flow_offload`: integrate with `nf_flow_offload_add` (cross-ref `net/netfilter/flowtable.md`).
- REQ-11: `compat`: xt_match / xt_target legacy bridge so iptables-nft can install legacy match modules through nftables; identical semantics to xt_action_param eval.
- REQ-12: `socket`: look up local socket for skb 5-tuple; expose sk fields (mark, uid, gid, transparent, cgroup) into register.
- REQ-13: `fib` / `fib_inet`: FIB (route table) lookup; verdicts based on RTN_LOCAL / UNICAST / BROADCAST / MULTICAST etc.
- REQ-14: All expression `dump` callbacks emit per-NLA byte-identical wire format.
- REQ-15: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `payload` test: `nft add rule ... ip daddr 1.2.3.4 drop` → matching packet → drop; tcpdump on rule shows internal payload+cmp evaluation. (covers REQ-3, REQ-5)
- [ ] AC-2: `payload` write test: `nft add rule ... ip ttl set 64` → outbound packet's TTL rewritten to 64; checksum incrementally updated. (covers REQ-3)
- [ ] AC-3: `meta` test: `nft add rule ... meta mark 0x100 accept` → matching mark accepted. (covers REQ-4)
- [ ] AC-4: `meta` test: `nft add rule ... meta iifname "eth0" jump custom` → packets from eth0 jump to custom chain. (covers REQ-4)
- [ ] AC-5: `lookup` test: `nft add set ... allowed_ips { type ipv4_addr; }; add element ... allowed_ips { 1.2.3.4 }; add rule ... ip saddr @allowed_ips accept` → matching saddr accepted. (covers REQ-6)
- [ ] AC-6: `dynset` test: `nft add rule ... ip saddr add @blocklist { ip saddr timeout 60s }` → saddr added to blocklist with 60s timeout; subsequent traffic checks. (covers REQ-7)
- [ ] AC-7: `ct` test: `nft add rule ... ct state established accept` → established connections accepted. (covers REQ-8)
- [ ] AC-8: `nat` test: `nft add rule ip nat post oifname "eth0" snat to 192.0.2.1` → NAT applied. (covers REQ-9)
- [ ] AC-9: `flow_offload` test: `nft add rule ... ip protocol tcp flow add @ft` → established TCP flow → added to flowtable. (covers REQ-10)
- [ ] AC-10: `compat` test: `iptables-nft -A FORWARD -m conntrack --ctstate ESTABLISHED -j ACCEPT` → translated to nftables rule using nft_compat with xt_conntrack match; behavior identical to iptables-legacy. (covers REQ-11)
- [ ] AC-11: `socket` test: `nft add rule ... socket cgroupv2 level 1 "system.slice" accept` → accept packets owned by sockets in cgroup. (covers REQ-12)
- [ ] AC-12: `fib` test: `nft add rule ... fib saddr type local accept` → packets with saddr=local IP accepted. (covers REQ-13)
- [ ] AC-13: All NLA dumps byte-identical for all expressions. (covers REQ-14)
- [ ] AC-14: Hardening section present and follows template. (covers REQ-15)

## Architecture

### Rust module organization

- `kernel::net::netfilter::nft::expr::Registry` — per-system nft_expr_type list
- `kernel::net::netfilter::nft::expr::payload::Payload` — read/write skb header bytes
- `kernel::net::netfilter::nft::expr::meta::Meta` — skb/sk/cgroup/secpath/ipsec metadata
- `kernel::net::netfilter::nft::expr::immediate::Immediate` — load constant / set verdict
- `kernel::net::netfilter::nft::expr::cmp::Cmp` — register comparison
- `kernel::net::netfilter::nft::expr::bitwise::Bitwise`
- `kernel::net::netfilter::nft::expr::byteorder::ByteOrder`
- `kernel::net::netfilter::nft::expr::lookup::Lookup` — set-membership
- `kernel::net::netfilter::nft::expr::dynset::Dynset` — dynamic set add
- `kernel::net::netfilter::nft::expr::objref::Objref` — stateful object reference
- `kernel::net::netfilter::nft::expr::range::Range` — range comparison
- `kernel::net::netfilter::nft::expr::numgen::Numgen`
- `kernel::net::netfilter::nft::expr::hash::Hash`
- `kernel::net::netfilter::nft::expr::ct::Ct`, `CtFast`
- `kernel::net::netfilter::nft::expr::counter::Counter`
- `kernel::net::netfilter::nft::expr::quota::Quota`
- `kernel::net::netfilter::nft::expr::limit::Limit`
- `kernel::net::netfilter::nft::expr::connlimit::Connlimit`
- `kernel::net::netfilter::nft::expr::nat::Nat`, `Redir`, `Masq`
- `kernel::net::netfilter::nft::expr::log::Log`
- `kernel::net::netfilter::nft::expr::reject::Reject`, `RejectInet`, `RejectNetdev`
- `kernel::net::netfilter::nft::expr::fib::Fib`, `FibInet`, `FibNetdev`
- `kernel::net::netfilter::nft::expr::flow_offload::FlowOffload`
- `kernel::net::netfilter::nft::expr::fwd_netdev::FwdNetdev`, `DupNetdev`
- `kernel::net::netfilter::nft::expr::queue::Queue`
- `kernel::net::netfilter::nft::expr::socket::Socket`
- `kernel::net::netfilter::nft::expr::xfrm::Xfrm`
- `kernel::net::netfilter::nft::expr::exthdr::Exthdr`
- `kernel::net::netfilter::nft::expr::osf::Osf`
- `kernel::net::netfilter::nft::expr::last::Last`
- `kernel::net::netfilter::nft::expr::inner::Inner`
- `kernel::net::netfilter::nft::expr::compat::Compat` — xt_match / xt_target bridge

### Locking and concurrency

- **`nft_expr_type_mutex`** (per-system mutex): expression-type registry mutator
- **Per-expression internal state**: per-`nft_expr` private data; init / activate / deactivate / destroy lifecycle (cross-ref `nft-core.md` REQ-2)
- **RCU**: dispatch hot path is RCU-side

### Error handling

- `Err(EINVAL)` — bad NLA / invalid combination
- `Err(EOPNOTSUPP)` — expression not supported by HW offload (per `offload` callback)
- `Err(EEXIST)` — duplicate expression-type registration
- `Err(ENOMEM)` — alloc fail
- `Err(ENOENT)` — referenced object/set not found

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `payload` skb-header read/write bounds (per-base offset arithmetic) | `kani::proofs::net::netfilter::nft::expr::payload_safety` |
| `meta` per-key field accessor (no out-of-bounds; type-correct register write) | `kani::proofs::net::netfilter::nft::expr::meta_safety` |
| `cmp` per-op comparison (correct sign semantics; bounded compare length) | `kani::proofs::net::netfilter::nft::expr::cmp_safety` |
| `lookup` set-side reader (RCU-side; no use-after-free) | `kani::proofs::net::netfilter::nft::expr::lookup_safety` |
| `dynset` set-side mutator (under set-bucket lock; per-element timeout sound) | `kani::proofs::net::netfilter::nft::expr::dynset_safety` |
| `compat` xt_match dispatch (legacy module dispatch via xt_action_param) | `kani::proofs::net::netfilter::nft::expr::compat_safety` |

### Layer 2: TLA+ models

(none new; consumed via `models/net/nft_vm_eval.tla` from `nft-core.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-system nft_expr_type list | every entry has unique `name`; registry consistent under mutator lock | `kani::proofs::net::netfilter::nft::expr::registry_invariants` |
| Per-expression private data size | matches `nft_expr_type.ops->size` always | `kani::proofs::net::netfilter::nft::expr::size_invariants` |

### Layer 4: Functional correctness (opt-in)

- **payload checksum-incremental theorem** via Verus — proves: post-`payload write` checksum equals RFC 1624 incremental update of pre-write checksum.
- **lookup set-membership soundness theorem** via Verus — proves: ∀ set element `e`, `lookup` returns hit iff `e ∈ set` (under concurrent set mutation, RCU-side reads are sound).

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-expression `nft_expr_ops` + `nft_expr_type` instances `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-expression skb-bounds + register-size arithmetic uses checked operators | § Mandatory |
| **MEMORY_SANITIZE** | freed expression private state cleared (carries match keys + selectors) | § Default-on configurable off |

### Row-1 features consumed by this component

- **CONSTIFY, SIZE_OVERFLOW, MEMORY_SANITIZE**: see above
- **REFCOUNT, AUTOSLAB**: per-expression slabs (cross-ref `nft-api.md`)
- **USERCOPY**: NLA parsing in init/dump uses bound-checked accessors
- **KERNEXEC**: per-expression dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook: same as `nft-api.md` (CAP_NET_ADMIN gate; mutations only via API).
- Default useful GR-RBAC policy: deny `nft_compat` xt_match / xt_target installation outside gradm-marked `firewall_admin` role; legacy xt_modules are a frequent CVE source (compat layer reduces attack-surface defenses).

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

- PAX_USERCOPY: NFTA_DATA payload for `nft_immediate`, `nft_cmp`, `nft_bitwise`, and `nft_payload` validated against `NFT_DATA_VALUE_MAXLEN` before any copy_from_user / nla_memcpy.
- PAX_KERNEXEC: expression `eval` callbacks reside in .text; `nft_expr_type` table immutable post-init.
- PAX_RANDKSTACK: chain evaluation entry randomizes stack base, defeating known offsets used by ROP into expression evaluators.
- PAX_REFCOUNT: per-expression private-area references (`nft_object->use` for `nft_objref`, `nft_set->use` for `nft_lookup`) saturating refcount_t.
- PAX_MEMORY_SANITIZE: per-expr private data zeroed on `expr->ops->destroy()`; freed register backing scrubbed.
- PAX_UDEREF: control-path `nft_parse_register_load()`/`nft_parse_register_store()` validates register indices and source ranges under uderef.
- PAX_RAP / kCFI: `nft_expr_ops` (eval/init/destroy/dump/clone/select_ops) kCFI-typed; substituting an ops vector with mismatched prototypes traps.
- GRKERNSEC_HIDESYM: expression dumps redact kernel pointers; `nft_expr_type` addresses hidden.
- GRKERNSEC_DMESG: expression init failures rate-limited; CAP_SYSLOG gates verbose error context.
- nftables CAP_NET_ADMIN: expression registration via `NFT_MSG_NEWRULE` gated by CAP_NET_ADMIN in net_ns.
- Transaction commit-abort: failed expression init in a rule rolls back the entire transaction; partial expressions are destroyed in reverse order.
- nft_chain PAX_REFCOUNT: `nft_jump` / `nft_goto` chain target refs use `nft_use_inc()` saturating; cannot wrap to free a referenced chain.
- JIT W^X (PAX_KERNEXEC): expressions that compile lookup tables (`nft_set_pipapo` AVX2) emit data-only pages; no executable codegen at expression level.

Rationale: per-expression OOB-read/write (e.g., CVE-2022-2588 nft_object UAF, CVE-2023-3390 chain binding UAF) are recurring; register-index validation plus kCFI-typed ops dispatch contains the class.

## Open Questions

(none — per-expression types exhaustively specified by upstream + nft regression-test suite + iproute2/iptables-nft compat coverage)

## Out of Scope

- Per-expression Tier-4 docs (deferred until any expression accumulates significant per-expression complexity)
- VM eval engine (cross-ref `nft-core.md`)
- Set storage backends (cross-ref `nft-sets.md`)
- API + transactions (cross-ref `nft-api.md`)
- 32-bit-only paths
- Implementation code
