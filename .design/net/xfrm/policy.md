# Tier-3: net/xfrm/policy — Security Policy Database (SPD) + xfrm_lookup

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/xfrm/xfrm_policy.c
  - net/xfrm/xfrm_hash.c
  - net/xfrm/xfrm_hash.h
  - net/ipv4/xfrm4_policy.c
  - net/ipv6/xfrm6_policy.c
  - include/net/xfrm.h
  - include/uapi/linux/xfrm.h
-->

## Summary
Tier-3 design for the Security Policy Database (SPD): per-netns policy storage indexed by direction (`XFRM_POLICY_IN/OUT/FWD`) and selector, the per-hashtable bydst+byidx indexes, the policy lookup hot-path (`__xfrm_policy_lookup`/`xfrm_policy_lookup`/`__xfrm_policy_check`), the bundle cache (compiled `dst_entry` chains for repeated TX with the same flow + policy combination), and the per-AF policy adapters (`xfrm4_policy.c`, `xfrm6_policy.c`).

Policy templates inside a policy specify the SAs to apply (proto/mode/algo) and how to acquire them. On TX, `xfrm_lookup` matches policy → resolves templates → SAs → builds bundle (chain of `dst_entry`s) → attaches to skb. On RX, `__xfrm_policy_check` validates that received SA-stack matches a configured IN policy.

Sub-tier-3 of `net/xfrm/00-overview.md`. Pairs with `net/xfrm/state.md` (SADB consumer), `net/xfrm/input.md` (RX policy check), `net/xfrm/output.md` (TX bundle build).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| SPD core: per-netns hashtables, lookup, mutation, walk, bundle cache | `net/xfrm/xfrm_policy.c` |
| Hash table primitive (shared with state) | `net/xfrm/xfrm_hash.c`, `net/xfrm/xfrm_hash.h` |
| Per-AF v4 adapter (xfrm4_dst_ops, xfrm4_policy_afinfo) | `net/ipv4/xfrm4_policy.c` |
| Per-AF v6 adapter (xfrm6_dst_ops, xfrm6_policy_afinfo) | `net/ipv6/xfrm6_policy.c` |
| Public API | `include/net/xfrm.h` |
| UAPI | `include/uapi/linux/xfrm.h` (struct xfrm_userpolicy_id, struct xfrm_user_tmpl) |

## Compatibility contract

### Policy direction tags

| Direction | Constant | Applied to |
|---|---|---|
| `XFRM_POLICY_IN` | 0 | RX path; validates received SA stack |
| `XFRM_POLICY_OUT` | 1 | TX path; resolves SAs to apply |
| `XFRM_POLICY_FWD` | 2 | Forwarded packets (gateway role) |

(`XFRM_POLICY_MAX = 3`.)

### Per-netns hashtables

For each direction, three hashtables:
1. `policy_bydst[]` — keyed on (selector.dst, family, dir)
2. `policy_byidx[]` — keyed on (index, dir)
3. `policy_inexact_list[dir]` — fallback list for non-exact-match selectors

Plus a per-AF inexact rb-tree for prefix-based lookups (used when destination-only-match isn't sufficient). Bucket sizing dynamic; resize on threshold.

Identical structure + sizing.

### `struct xfrm_policy` layout

`include/net/xfrm.h`:
- `bydst`, `byidx` (per-table list_head linkage)
- `lock` (rwlock)
- `refcnt`
- `bydst_reinsert` flag
- `selector` (`struct xfrm_selector`: daddr/saddr/dport/sport/proto/family/prefixlen_d/prefixlen_s/ifindex/user)
- `priority`
- `index`
- `pos`
- `mark`
- `if_id`
- `walk` (struct xfrm_policy_walk_entry)
- `inexact_list`, `inexact_bin` (inexact-match support)
- `bundles` (per-policy compiled dst_entry chain cache list)
- `xdo` (offload state if HW-offloaded)
- `lft` (lifetime), `curlft` (current)
- `xfrm_nr` (template count)
- `family`
- `action` (XFRM_POLICY_ALLOW / BLOCK)
- `flags`
- `xfrm_vec[XFRM_MAX_DEPTH]` (template array)

Layout-equivalent for first cache-line.

### Templates: `struct xfrm_tmpl`

Each policy carries up to `XFRM_MAX_DEPTH=6` templates, each specifying:
- `id` (xfrm_id: daddr/spi/proto)
- `saddr`
- `reqid`
- `mode` (TRANSPORT/TUNNEL/RO/IN_TRIGGER/BEET)
- `share` (XFRM_SHARE_*)
- `optional`
- `family`
- `ealgos`/`aalgos`/`calgos` algorithm masks

### Bundle cache

Per-policy `bundles` list of compiled `dst_entry` chains for fast TX. Each bundle:
- Is keyed on (template-array, flow-class)
- Holds a chain `dst_entry → xfrm_dst → … → (final dst)`
- Attaches to skb as `skb_dst_set_noref` for fast XFRM-aware TX
- Refcounted; evicted on policy mutation or template SA expiry

### `xfrm_lookup_*` hot path (TX)

`xfrm_lookup` / `xfrm_lookup_route` / `__xfrm_lookup` — called from per-AF route lookup. For each TX flow:
1. Lookup policy via `xfrm_policy_lookup_bytype(net, dir=OUT, fl, family)`
2. If policy found:
   a. Lookup templates → resolve SAs (via `xfrm_state_find` → `xfrm_state_lookup_byaddr`)
   b. If any template missing → emit ACQUIRE; return EAGAIN
   c. Build bundle (`xfrm_bundle_create`)
   d. Cache + return
3. If no policy → return original dst (plaintext)

Identical sequence + result.

### `__xfrm_policy_check` (RX)

Called from `xfrm_input` end-of-pipe to validate incoming SA stack matches IN policy. For each RX flow:
1. Lookup IN policy via `xfrm_policy_lookup_bytype(net, dir=IN, fl, family)`
2. If policy + skb has SA stack:
   a. For each template in policy: verify a corresponding SA was applied to skb in correct order
   b. Mismatch → drop + counter `XfrmInTmplMismatch++`
   c. Match → ACCEPT
3. If no policy + skb has SA stack: drop + counter `XfrmInNoPols++`
4. If policy XFRM_POLICY_BLOCK: drop + counter `XfrmInPolBlock++`
5. If no policy + skb plain: ACCEPT

Identical decision tree.

### `XFRM_MSG_NEWPOLICY/UPDPOLICY/DELPOLICY/GETPOLICY/FLUSHPOLICY`

NETLINK_XFRM messages parsed by `xfrm_user.c` (cross-ref `net/xfrm/user.md`); semantics here:
- NEWPOLICY: install fresh; conflict on (selector, dir, priority, index) → EEXIST
- UPDPOLICY: replace in place
- DELPOLICY: tear down; refcount drains; bundle cache flushed
- GETPOLICY: lookup; return walk-formatted attrs
- FLUSHPOLICY: bulk drop matching filter

Wire format byte-identical (cross-ref `net/xfrm/user.md`).

### Policy walk iterator

`xfrm_policy_walk_init` / `xfrm_policy_walk` / `xfrm_policy_walk_done` for `XFRM_MSG_GETPOLICY` dumps; per-walker cursor; identical.

## Requirements

- REQ-1: Per-netns hashtables `policy_bydst[]`, `policy_byidx[]`, `policy_inexact_list[]` + inexact rb-tree per direction; identical bucket sizing + dynamic resize.
- REQ-2: `struct xfrm_policy` first-cache-line layout-equivalent.
- REQ-3: Three direction tags `IN=0`, `OUT=1`, `FWD=2` (`XFRM_POLICY_MAX=3`); per-direction independent storage.
- REQ-4: Templates: per-policy `xfrm_vec[XFRM_MAX_DEPTH=6]`; per-template fields per layout above.
- REQ-5: `xfrm_policy_lookup_bytype` priority-ordered selector match; first-match-wins; identical algorithm.
- REQ-6: `xfrm_lookup` TX path: policy → template-resolve → bundle-build; missing template → emit ACQUIRE.
- REQ-7: `__xfrm_policy_check` RX path: per-template SA-stack validation; XfrmInTmplMismatch/XfrmInNoPols/XfrmInPolBlock counters bumped at decision points.
- REQ-8: Bundle cache: per-policy `bundles` list; eviction on policy mutation or template SA expiry.
- REQ-9: NEWPOLICY/UPDPOLICY/DELPOLICY/GETPOLICY/FLUSHPOLICY semantics identical.
- REQ-10: Policy walk iterator semantics identical.
- REQ-11: Per-AF adapters: xfrm4_dst_ops + xfrm4_policy_afinfo (v4) + xfrm6_dst_ops + xfrm6_policy_afinfo (v6); identical.
- REQ-12: TLA+ model `models/net/xfrm_policy_eval.tla` (mandatory per `net/xfrm/00-overview.md` Layer 2) — proves policy evaluation atomicity: a TX flow either matches the most-recent committed policy snapshot, or returns EAGAIN; never matches a partial mutation.
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct xfrm_policy` byte-identical first cache-line. (covers REQ-2)
- [ ] AC-2: `ip xfrm policy add ...` test: policy visible via `ip xfrm policy list`; lookup-by-direction returns it. (covers REQ-1, REQ-9)
- [ ] AC-3: TX policy-match test: install policy with selector matching dst=`10.0.0.0/24`; TX to `10.0.0.5` → bundle built (verifiable via tcpdump showing ESP); TX to `192.168.0.5` → plaintext. (covers REQ-5, REQ-6)
- [ ] AC-4: RX policy-validation test: install IN policy requiring ESP from peer; receive plaintext → drop + `XfrmInNoPols++`; receive correctly ESP'd → ACCEPT. (covers REQ-7)
- [ ] AC-5: Template-mismatch test: policy expects ESP+AH stack; receive ESP-only → drop + `XfrmInTmplMismatch++`. (covers REQ-7)
- [ ] AC-6: BLOCK policy test: install OUT policy with action=BLOCK matching dst=`10.0.1.0/24`; TX to that range → drop + `XfrmOutPolBlock++`. (covers REQ-7)
- [ ] AC-7: Bundle cache test: 1000 TX to same dst → first invocation builds bundle; subsequent reuses cached (verifiable via /proc/net/xfrm_stat or perf counters); policy mutation evicts cache. (covers REQ-8)
- [ ] AC-8: ACQUIRE-on-missing-template test: install policy with template but no matching SA; TX → EAGAIN to caller + ACQUIRE message to daemon. (covers REQ-6)
- [ ] AC-9: Walk test: 100 policies installed; `XFRM_MSG_GETPOLICY` dump → returns all 100. (covers REQ-10)
- [ ] AC-10: Per-AF adapter test: install v4 + v6 policies in same netns; RX/TX paths use the right per-AF adapter (verify via tcpdump on each AF). (covers REQ-11)
- [ ] AC-11: TLA+ `models/net/xfrm_policy_eval.tla` proves: TX never matches a mid-mutation policy; concurrent UPDPOLICY + xfrm_lookup → either old or new, never partial. (covers REQ-12)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::net::xfrm::policy::Spd` — per-netns SPD
- `kernel::net::xfrm::policy::XfrmPolicy` — `struct xfrm_policy` wrapper
- `kernel::net::xfrm::policy::tables::HashTables` — per-direction hashtables + inexact rb-tree
- `kernel::net::xfrm::policy::lookup::PolicyLookup` — `__xfrm_policy_lookup` / `bytype`
- `kernel::net::xfrm::policy::tx::XfrmLookup` — TX path
- `kernel::net::xfrm::policy::rx::PolicyCheck` — `__xfrm_policy_check`
- `kernel::net::xfrm::policy::bundle::BundleCache` — per-policy compiled dst_entry chain cache
- `kernel::net::xfrm::policy::bundle::Builder` — `xfrm_bundle_create`
- `kernel::net::xfrm::policy::walk::PolicyWalk` — dump iterator
- `kernel::net::xfrm::policy::v4::Xfrm4PolicyAfinfo` — v4 adapter
- `kernel::net::xfrm::policy::v6::Xfrm6PolicyAfinfo` — v6 adapter

### Locking and concurrency

- **Per-netns `xfrm_policy_lock`** (rwlock): protects per-netns hashtables + walk iterators
- **Per-`xfrm_policy` `lock`** (rwlock): protects per-policy mutable state (bundles list)
- **Per-policy `bundles` list**: linked-list of bundles cached for this policy
- **RCU**: lookup hot path is RCU-side; mutator side via `xfrm_policy_lock`

### Error handling

- `Err(EEXIST)` — NEWPOLICY for duplicate
- `Err(ENOENT)` — DELPOLICY / GETPOLICY for non-existent
- `Err(EAGAIN)` — `xfrm_lookup` returns when ACQUIRE is pending (template not yet resolved)
- `Err(EINVAL)` — bad policy params (e.g., template missing required fields)
- `Err(ENOMEM)` — slab alloc fail
- `Err(EPERM)` — non-CAP_NET_ADMIN

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-netns hashtable insert/erase under xfrm_policy_lock | `kani::proofs::net::xfrm::policy::table_safety` |
| `xfrm_policy_lookup_bytype` priority-ordered walk | `kani::proofs::net::xfrm::policy::lookup_safety` |
| Bundle build chain (refcount sound across resolve) | `kani::proofs::net::xfrm::policy::bundle_build_safety` |
| Bundle cache eviction on policy mutation | `kani::proofs::net::xfrm::policy::bundle_evict_safety` |
| `__xfrm_policy_check` SA-stack walk (no out-of-bounds) | `kani::proofs::net::xfrm::policy::check_safety` |

### Layer 2: TLA+ models

- `models/net/xfrm_policy_eval.tla` (mandatory per `net/xfrm/00-overview.md`) — proves policy-evaluation atomicity. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-netns SPD | every policy appears in exactly the bucket determined by `policy_hash_bucket(selector.daddr, family, dir)` (or in inexact list/tree) | `kani::proofs::net::xfrm::policy::table_invariants` |
| Per-policy bundles list | every cached bundle's templates match policy's `xfrm_vec`; refcount > 0 | `kani::proofs::net::xfrm::policy::bundle_invariants` |
| Per-policy refcount | refcount = walk-side+lookup-side+xfrm_user holders + per-bundle holders | `kani::proofs::net::xfrm::policy::refcount_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Policy-evaluation refinement theorem** via TLA+ refinement — proves: implementation refines the abstract policy-eval model; concurrent UPDPOLICY ↔ lookup is sound.
- **First-match-wins theorem** via Verus — proves: `xfrm_policy_lookup_bytype` returns the highest-priority policy whose selector matches `fl`; lower-priority matches are skipped.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-`xfrm_policy` refcount uses `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-`xfrm_policy` slab cache | § Mandatory |
| **MEMORY_SANITIZE** | freed `xfrm_policy` cleared (selectors carry source/dest IPs that may be policy-relevant) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **CONSTIFY**: `xfrm4_policy_afinfo`, `xfrm6_policy_afinfo`, `xfrm_dst_ops` `static const`
- **USERCOPY**: NLA parsing uses bound-checked accessors
- **SIZE_OVERFLOW**: priority + index arithmetic uses checked operators

### Row-2 / GR-RBAC integration

- LSM hooks: `security_xfrm_policy_alloc`, `security_xfrm_policy_clone`, `security_xfrm_policy_free`, `security_xfrm_policy_delete`, `security_xfrm_policy_lookup`. GR-RBAC policy can deny per-subject (default empty).
- Useful default GR-RBAC policy: deny policy mutation outside gradm-marked `ipsec_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — SPD + xfrm_lookup semantics are exhaustively specified by upstream + RFC 4301)

## Out of Scope

- SADB (cross-ref `net/xfrm/state.md`)
- RX/TX pipelines (cross-ref `net/xfrm/input.md`, `net/xfrm/output.md`)
- 32-bit-only paths
- Implementation code
