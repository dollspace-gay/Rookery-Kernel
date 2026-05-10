# Tier-3: net/xfrm/state — Security Association Database (SADB)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/xfrm/xfrm_state.c
  - net/xfrm/xfrm_hash.c
  - net/xfrm/xfrm_hash.h
  - net/xfrm/xfrm_state_bpf.c
  - include/net/xfrm.h
  - include/uapi/linux/xfrm.h
-->

## Summary
Tier-3 design for the Security Association Database (SADB): per-netns hashtables of `struct xfrm_state` keyed by (daddr, spi, proto), (saddr, daddr, proto), and (sequence number) to support all SA-lookup paths used in RX/TX, the SA lifecycle (VALID → EXPIRING via soft/hard lifetimes → DEAD), the SPI allocator (`xfrm_alloc_spi` with cryptographically-strong randomness), the `xfrm_state_walk` iterator for dump operations, BPF state introspection (`xfrm_state_bpf.c`), and the SA mutation primitives (`xfrm_state_add`, `xfrm_state_update`, `xfrm_state_delete`).

Sub-tier-3 of `net/xfrm/00-overview.md`. The "S" in IPSec; lookups happen on every encrypted packet's RX and TX path. Pairs with `net/xfrm/policy.md` (the SPD), `net/xfrm/input.md` (RX consumer), `net/xfrm/output.md` (TX consumer), `net/xfrm/replay.md` (per-SA replay window).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| SADB core: `xfrm_state_*` mutation, hashtables, lookup, walk, lifetime mgmt, SPI alloc | `net/xfrm/xfrm_state.c` |
| Hash table primitive (shared with policy) | `net/xfrm/xfrm_hash.c`, `net/xfrm/xfrm_hash.h` |
| BPF SA introspection | `net/xfrm/xfrm_state_bpf.c` |
| Public API | `include/net/xfrm.h` |
| UAPI | `include/uapi/linux/xfrm.h` (struct xfrm_id, struct xfrm_selector, struct xfrm_state_walk) |

## Compatibility contract

### Per-netns hashtables

Three independent per-netns hashtables for fast lookup:
1. `state_bydst[]` — keyed on (daddr, family, proto)
2. `state_bysrc[]` — keyed on (saddr, daddr, family, proto)
3. `state_byspi[]` — keyed on (spi, daddr, family)

Plus a `state_byseq[]` for ACQUIRE-driven lookup. Bucket sizing dynamic per-netns (`xfrm_hash_alloc`/`xfrm_hash_grow`); resize on threshold crossing.

Identical bucket sizing + dynamic resize so existing perf characteristics match.

### `struct xfrm_state` layout

`include/net/xfrm.h` (referenced from `xfrm.h`):
- `bydst`, `bysrc`, `byspi` (per-table list_head linkage)
- `refcnt`, `lock`
- `id` (struct xfrm_id: `daddr`, `spi`, `proto`)
- `sel` (struct xfrm_selector)
- `props` (struct xfrm_state_walk: per-walk list_head, lifetime, replay-state, policy-class, family, type, mode, …)
- `lft` (struct xfrm_lifetime_cfg: hard/soft byte/packet/time limits)
- `curlft` (struct xfrm_lifetime_cur: byte/packet/time-current)
- `replay` + `replay_esn` (replay window state)
- `mark`, `if_id`
- `tfcpad` (Traffic Flow Confidentiality padding)
- `xfrag` (frag-after-encrypt mode)
- `aead`, `aalg`, `ealg`, `calg`, `encap`, `tfc`, `coaddr`
- `km` (per-key-manager state)
- `xso` (offload state if HW-offloaded)
- `rcache` (route cache for fast TX)

Layout-equivalent for first cache-line.

### SA lifecycle

| Transition | Trigger |
|---|---|
| INIT → VALID | `xfrm_state_add` after key install |
| VALID → EXPIRING | soft byte/packet/time limit reached → `XFRM_MSG_EXPIRE` to userspace daemon (rekey hint) |
| EXPIRING → DEAD | hard byte/packet/time limit reached → SA torn down |
| any → DEAD | `xfrm_state_delete` from userspace |

Identical timers + thresholds.

### `xfrm_alloc_spi` — RFC 4303 § 2.1 cryptographically-random SPI

For ESP/AH: 32-bit SPI must be unpredictable across SAs. Algorithm:
1. Try N times: `get_random_bytes(&spi, sizeof(spi))`; reject reserved range `[0..255]` (RFC 4303 § 2.1) and IKE-userspace-reserved range; reject if collision in `state_byspi[]`
2. After N iterations or `htonl(low)..htonl(high)` range exhaustion → return `-EAGAIN`

Identical algorithm so SPI distribution is bit-identical across builds.

### `XFRM_MSG_NEWSA / UPDSA / DELSA / GETSA / FLUSHSA / EXPIRE / ACQUIRE`

NETLINK_XFRM messages parsed by `xfrm_user.c` (cross-ref `net/xfrm/user.md`); semantics here:
- NEWSA: install fresh; conflict (same key) → EEXIST
- UPDSA: replace key in place (atomic swap)
- DELSA: tear down; refcount drains
- GETSA: lookup; return state-walk-formatted attrs
- FLUSHSA: bulk drop matching filter
- EXPIRE: kernel-emitted message indicating soft-limit hit
- ACQUIRE: kernel-emitted message indicating policy-required SA missing → daemon should IKE up

Wire format byte-identical (cross-ref `net/xfrm/user.md`).

### `xfrm_state_walk_*` (dump iterator)

Walk-based iteration for `XFRM_MSG_GETSA` dumps; tracks per-walker cursor across multiple netlink replies. Identical algorithm.

### BPF state introspection

`xfrm_state_bpf.c` exposes per-SA fields to BPF programs via the `bpf_xdp_metadata_*` style API. Read-only. Identical UAPI.

## Requirements

- REQ-1: Per-netns hashtables: `state_bydst[]`, `state_bysrc[]`, `state_byspi[]`, `state_byseq[]`; identical bucket sizing + dynamic resize.
- REQ-2: `struct xfrm_state` first-cache-line layout-equivalent.
- REQ-3: SA lifecycle transitions: VALID → EXPIRING → DEAD driven by lifetime limits + user/daemon mutation.
- REQ-4: `xfrm_alloc_spi`: cryptographically-strong SPI allocation per RFC 4303 § 2.1; reserved range `[0..255]` rejected; collision-rejection.
- REQ-5: `xfrm_state_add`, `xfrm_state_update`, `xfrm_state_delete`, `xfrm_state_lookup_byspi`, `xfrm_state_lookup_byaddr`, `xfrm_state_lookup` semantics identical.
- REQ-6: Per-SA refcount via `Refcount` (saturating); destroy via per-state RCU-deferred `xfrm_state_destroy`.
- REQ-7: Soft-limit handling: byte/packet/time lifetime; on cross → emit `XFRM_MSG_EXPIRE` (soft) to per-netns notify socket; identical formatting.
- REQ-8: Hard-limit handling: byte/packet/time lifetime; on cross → mark DEAD + emit `XFRM_MSG_EXPIRE` (hard) + cleanup.
- REQ-9: ACQUIRE path: when policy-driven `xfrm_lookup` finds policy but no SA → emit `XFRM_MSG_ACQUIRE` to daemon + queue policy template waiters; identical.
- REQ-10: Per-netns `xfrm_state_walk_init` / `xfrm_state_walk` / `xfrm_state_walk_done` iterator semantics identical.
- REQ-11: BPF state introspection via `xfrm_state_bpf.c` exposes documented fields read-only.
- REQ-12: TLA+ model `models/net/xfrm_state_machine.tla` (mandatory per `net/xfrm/00-overview.md` Layer 2) — proves SA lifecycle transitions are sound under concurrent xfrm_user mutations + RX/TX consumers; refcount never goes negative; DEAD transitions are total.
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct xfrm_state` byte-identical first cache-line. (covers REQ-2)
- [ ] AC-2: `ip xfrm state add ...` test: state visible via `ip xfrm state list`; lookup-by-spi returns it. (covers REQ-1, REQ-5)
- [ ] AC-3: `ip xfrm state allocspi` test: SPI is in valid range (256..0xFFFFFFFF except IKE-reserved); no collision in single-netns over 1000 calls; entropy ≥ 30 bits. (covers REQ-4)
- [ ] AC-4: Soft-lifetime test: SA with `lifetime soft byte 1024` → after 1024-byte traffic → daemon receives `XFRM_MSG_EXPIRE` with `hard=0`; SA still valid. (covers REQ-7)
- [ ] AC-5: Hard-lifetime test: SA with `lifetime hard byte 2048` → after 2048-byte traffic → SA torn down + daemon receives `XFRM_MSG_EXPIRE` with `hard=1`; subsequent lookups return ENOENT. (covers REQ-8)
- [ ] AC-6: ACQUIRE test: install policy without SA → send packet → daemon receives `XFRM_MSG_ACQUIRE`; installation of matching SA via `XFRM_MSG_NEWSA` drains queued packet. (covers REQ-9)
- [ ] AC-7: Walk test: 100 SAs installed; `XFRM_MSG_GETSA` dump → returns all 100 across multiple netlink replies. (covers REQ-10)
- [ ] AC-8: BPF test: BPF program reads SA fields via `xfrm_state_bpf` API; values match `ip xfrm state list`. (covers REQ-11)
- [ ] AC-9: TLA+ `models/net/xfrm_state_machine.tla` proves: refcount ≥ 0 invariant; DEAD-final invariant; no concurrent NEWSA + DELSA leaves orphan. (covers REQ-12)
- [ ] AC-10: Refcount test: 1000 SAs installed + dropped → no leak (verify via /proc/slabinfo). (covers REQ-6)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::net::xfrm::state::Sadb` — per-netns SADB
- `kernel::net::xfrm::state::XfrmState` — `struct xfrm_state` wrapper
- `kernel::net::xfrm::state::tables::HashTables` — `state_bydst`/`bysrc`/`byspi`/`byseq`
- `kernel::net::xfrm::state::lifecycle::Lifecycle` — VALID/EXPIRING/DEAD transitions
- `kernel::net::xfrm::state::lifetime::LifetimeTimer` — soft+hard byte/packet/time limits
- `kernel::net::xfrm::state::spi::SpiAllocator` — RFC 4303 § 2.1 SPI allocation
- `kernel::net::xfrm::state::walk::StateWalk` — dump iterator
- `kernel::net::xfrm::state::acquire::AcquireQueue` — policy-template waiters
- `kernel::net::xfrm::state::bpf::BpfStateAccess` — read-only BPF API

### Locking and concurrency

- **Per-netns `xfrm_state_lock`** (rwlock): protects per-netns hashtables + walk iterators
- **Per-`xfrm_state` `lock`** (spinlock): protects per-SA mutable state (curlft, replay)
- **`xfrm_state_gc_lock`** (per-netns spinlock): GC list mutator
- **RCU**: lookup hot path is RCU-side; mutator side via `xfrm_state_lock`
- **Per-netns `xfrm_acq_seq`** (atomic): ACQUIRE sequence counter

### Error handling

- `Err(EEXIST)` — NEWSA for duplicate SA (same xfrm_id)
- `Err(ENOENT)` — DELSA / GETSA for non-existent
- `Err(EAGAIN)` — SPI-allocation retry exhausted
- `Err(EINVAL)` — bad SA params (e.g., proto unsupported)
- `Err(ENOMEM)` — slab alloc fail
- `Err(EPERM)` — non-CAP_NET_ADMIN

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-netns hashtable insert/erase under xfrm_state_lock | `kani::proofs::net::xfrm::state::table_safety` |
| `xfrm_state_lookup_byspi/byaddr/by` lookup | `kani::proofs::net::xfrm::state::lookup_safety` |
| Lifetime-timer arithmetic (no overflow on byte/packet 64-bit accumulators) | `kani::proofs::net::xfrm::state::lifetime_safety` |
| SPI allocation rejection-loop (terminates within bound) | `kani::proofs::net::xfrm::state::spi_safety` |
| State-walk cursor across multiple replies | `kani::proofs::net::xfrm::state::walk_safety` |
| ACQUIRE queue insert / drain on NEWSA | `kani::proofs::net::xfrm::state::acquire_safety` |

### Layer 2: TLA+ models

- `models/net/xfrm_state_machine.tla` (mandatory per `net/xfrm/00-overview.md`) — proves SA lifecycle. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-netns SADB | every state appears in all 3 of `state_bydst[]` + `state_bysrc[]` + `state_byspi[]` (or `byseq[]` if pre-acquire) at every observable state | `kani::proofs::net::xfrm::state::table_invariants` |
| Per-SA refcount | refcount ≥ 0; equals walk-side+RX-side+TX-side+xfrm_user holders count | `kani::proofs::net::xfrm::state::refcount_invariants` |
| SPI uniqueness | per-(daddr, family) SPI is unique across all alive states | `kani::proofs::net::xfrm::state::spi_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `net/xfrm/00-overview.md` Layer 4)

- **SA lifecycle refinement theorem** via TLA+ refinement — proves: implementation refines the abstract state-machine model.
- **SPI distribution theorem** via Verus — proves: under the rejection-loop algorithm and a sound CSPRNG, SPI distribution is uniform on the valid range modulo collision-rejection bias of < 2^-30.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-`xfrm_state` refcount uses `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-`xfrm_state` slab cache | § Mandatory |
| **MEMORY_SANITIZE** | freed `xfrm_state` cleared (cipher/auth keys + replay window) | § Default-on configurable off |
| **LATENT_ENTROPY** | `xfrm_alloc_spi` seeded from kernel CSPRNG | § Default-on configurable off |
| **SIZE_OVERFLOW** | byte/packet 64-bit accumulator arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, LATENT_ENTROPY, SIZE_OVERFLOW**: see above
- **CONSTIFY**: hash-function pointers `static const`
- **USERCOPY**: NLA/SADB-message parsing uses bound-checked accessors
- **KERNEXEC**: per-protocol type-handler dispatch via `static const fn-ptr` arrays

### Row-2 / GR-RBAC integration

- LSM hooks (per upstream): `security_xfrm_state_alloc`, `security_xfrm_state_alloc_acquire`, `security_xfrm_state_delete`, `security_xfrm_state_free`, `security_xfrm_state_pol_flow_match`. GR-RBAC policy can deny per-subject (default empty).
- Useful default GR-RBAC policy: deny SA mutation outside gradm-marked `ipsec_admin` role; SA install gives kernel-level encryption-key control.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — SADB semantics are exhaustively specified by upstream + RFC 4302 + 4303)

## Out of Scope

- Per-protocol AH/ESP/IPCOMP transforms (cross-ref `net/xfrm/ah-esp-ipv4.md`, `net/xfrm/ah-esp-ipv6.md`)
- SPD (cross-ref `net/xfrm/policy.md`)
- RX/TX pipelines (cross-ref `net/xfrm/input.md`, `net/xfrm/output.md`)
- Replay window (cross-ref `net/xfrm/replay.md`)
- 32-bit-only paths
- Implementation code
