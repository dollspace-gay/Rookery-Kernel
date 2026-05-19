# Tier-3: mm/swap — swap subsystem (swapfile, swap_state, page_io for swap, zswap, swap_cgroup)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - mm/swapfile.c
  - mm/swap_state.c
  - mm/swap_cgroup.c
  - mm/page_io.c
  - mm/swap.c
  - mm/zswap.c
  - mm/zsmalloc.c
  - include/linux/swap.h
  - include/linux/swapops.h
-->

## Summary
Tier-3 design for the kernel's swap subsystem: swap-device management (`swapon`/`swapoff` lifecycle), the swap-state cache (a special address_space backing swap pages), page I/O for swap (the read/write path for swapped-out pages), per-memcg swap accounting, and zswap (compressed in-RAM swap cache backed by zsmalloc).

Swap is the eviction destination for anonymous pages (and shmem pages without a backing file). When `mm/reclaim.md`'s vmscan decides to evict an anon page, this Tier-3 owns the path: anon page → swap entry → write to swap device or compress into zswap → free physical page.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Swap device + swapon/swapoff lifecycle | `mm/swapfile.c` |
| Swap-state cache (swap pages indexed by swap entry) | `mm/swap_state.c` |
| Per-memcg swap accounting | `mm/swap_cgroup.c` |
| Page I/O for swap | `mm/page_io.c` |
| Per-CPU LRU mgmt (small set of helpers shared with reclaim) | `mm/swap.c` |
| Compressed in-RAM swap cache | `mm/zswap.c` |
| Compressed page-pool allocator (used by zswap; has its own Tier-3 doc — cross-ref) | `mm/zsmalloc.c` (cross-ref `mm/zsmalloc.md`) |
| Public types | `include/linux/swap.h`, `include/linux/swapops.h` |

## Compatibility contract

### Syscalls

`swapon(2)`, `swapoff(2)` — byte-identical entry/exit ABI per upstream.

### `/proc/swaps`

Format-identical: `Filename Type Size Used Priority` per line.

### `/sys/fs/cgroup/<grp>/memory.swap.{current,max,events,events.local}` (cgroup v2)

Format-identical numeric content.

### Sysctls

- `vm.swappiness` (default 60) — controls aggressiveness of swap-out vs. file-cache eviction
- `vm.swap_only` (boolean) — limits swap to swap devices only (no zswap, no zram)
- `vm.page_cluster` (default 3) — readahead cluster size for swap-in (2^N pages)
- `vm.zswap_*` — zswap-specific knobs (`enabled`, `compressor`, `zpool`, `accept_threshold_percent`, `max_pool_percent`, `same_filled_pages_enabled`)

### Swap-on-disk format

`mm/swapfile.c::swap_setup_header` writes a swap header at swap-area offset 0. Layout matches upstream so existing swap partitions/files mounted via `swapon` work unchanged. The `union swap_header` (`include/linux/swap.h`):

- `info.bootbits[1024]` — boot loader signature area
- `info.version`, `info.last_page`, `info.nr_badpages`, `info.uuid[16]`, `info.volume_name[16]`, `info.padding[117]`, `info.badpages[1]`
- 10 padding bytes
- `magic[10]` — must be "SWAPSPACE2"

Byte-identical so existing-on-disk swap formatted by `mkswap` is consumable.

### Swap entry encoding

`include/linux/swapops.h` defines the encoding of a swap entry into a `swp_entry_t` (a swap-type + offset packed into a single word). Identical encoding so swap entries stored in PTEs (after being swapped out) round-trip correctly.

### zswap shrinker

zswap registers a shrinker that participates in reclaim's general shrinker dispatch (cross-ref `mm/reclaim.md`).

## Requirements

- REQ-1: `swapon(2)` / `swapoff(2)` byte-identical entry/exit. Setup-header parsing is byte-identical.
- REQ-2: Swap entry encoding (swp_entry_t) byte-identical: same per-arch SWP_TYPE_BITS / SWP_OFFSET_MASK constants.
- REQ-3: Per-swap-device priority + striping (multiple equal-priority devices striped) matches upstream.
- REQ-4: swap-cluster allocation (per upstream's `swap_cluster_info`-tracked clusters) matches upstream's algorithm.
- REQ-5: Swap-cache (`swap_state`) is a special `address_space` keyed by swap entry; participates in `mm/page-cache.md`'s xarray-backed lookup with the swap-specific `address_space_operations`.
- REQ-6: Anon-page swap-out: anon page on LRU → mark Dirty + add to swap cache → schedule write via `page_io.c::swap_writepage` → on completion, drop swap-cache reference.
- REQ-7: Anon-page swap-in: page-fault handler (`mm/virtual-memory.md`) → look up swap entry → read from swap (or zswap) → install in PTE.
- REQ-8: zswap policy: under memory pressure, swap-out path tries zswap first (compressed in-RAM); on failure or after `accept_threshold_percent`, falls back to disk swap. Identical decision logic.
- REQ-9: zswap pool management (per upstream's per-pool zsmalloc handle list) preserved.
- REQ-10: Per-memcg swap accounting: charges to `memory.swap.current`; reaches `memory.swap.max` triggers OOM within memcg before global swap exhaustion.
- REQ-11: `/proc/swaps` content format-identical.
- REQ-12: Sysctl knobs (`vm.swappiness`, `vm.page_cluster`, `vm.swap_only`, `vm.zswap_*`) preserve names + value semantics.
- REQ-13: TLA+ model `models/mm/swap_atomic.tla` (per `mm/00-overview.md` Layer 2 mandate) proves: `get_swap_pages` / `put_swap_pages` never produces double-allocation under concurrent CPU access.
- REQ-14: Layer-3 invariant harness for swap slot allocator: every allocated slot is in exactly one of {in-flight read, in-flight write, mapped to swap-cache folio, free} (per `mm/00-overview.md` D4 + REQ-14).
- REQ-15: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `swapon /dev/sdaX` on a kernel-3.x-formatted swap partition succeeds; subsequent `cat /proc/swaps` shows it. (covers REQ-1, REQ-11)
- [ ] AC-2: `pahole struct swap_info_struct` produces byte-identical layout vs. upstream. (covers REQ-2 partly)
- [ ] AC-3: A test sets up two equal-priority swap devices, runs heavy memory pressure; `swap_used` distributes evenly across the two devices. (covers REQ-3, REQ-4)
- [ ] AC-4: A SWAP-IN test (mmap a file, fault pages, force vmscan to evict, then access again): anon pages round-trip via swap correctly; data unchanged. (covers REQ-5, REQ-6, REQ-7)
- [ ] AC-5: A zswap test (`zswap.enabled=1`): under pressure, zswap pool fills; spillover to disk swap when `max_pool_percent` reached. `cat /sys/kernel/debug/zswap/pool_total_size` shows expected growth. (covers REQ-8, REQ-9)
- [ ] AC-6: A cgroup v2 memcg swap-limit test: process exceeds `memory.swap.max`; in-memcg OOM fires before global swap exhaustion. (covers REQ-10)
- [ ] AC-7: `vm.swappiness=0` test: swap-out is suppressed; under same pressure, vm.swappiness=60 evicts anon pages aggressively. Behavior matches upstream. (covers REQ-12)
- [ ] AC-8: `make tla` passes `models/mm/swap_atomic.tla`. (covers REQ-13)
- [ ] AC-9: `make verify` passes `kani::proofs::mm::swap::*` Kani harnesses. (covers REQ-14)
- [ ] AC-10: Hardening section present and follows template. (covers REQ-15)

## Architecture

### Rust module organization

- `kernel::mm::swap::device::SwapDev` — per-swap-device state
- `kernel::mm::swap::cluster` — per-cluster slot allocator
- `kernel::mm::swap::cache` — swap_state address_space
- `kernel::mm::swap::io` — page_io.c analog (swap_writepage, swap_readpage)
- `kernel::mm::swap::cgroup` — per-memcg swap accounting
- `kernel::mm::zswap::Zswap` — compressed in-RAM swap cache
- `kernel::mm::zswap::pool` — zswap pool management
- `kernel::mm::swap::lru_helpers` — small helpers shared with reclaim (cross-ref `mm/reclaim.md`)

### Locking and concurrency

- **swap_lock** (global spinlock; rare; held during swap-device-list mutation) — swapon/swapoff
- **swap_avail_lock** (per-priority-group spinlock) — held during slot allocation
- **swap_info_struct->lock** (per-device spinlock) — held during cluster allocation
- **mem_cgroup_swap_full** check uses RCU
- **zswap_pool_lock** (per-pool spinlock) — held during pool-state mutation

TLA+ model `models/mm/swap_atomic.tla` (mandatory per `mm/00-overview.md` Layer 2) proves the get_swap_pages/put_swap_pages atomicity under concurrent CPU access.

### Error handling

- `Err(ENOSPC)` — swap full
- `Err(EIO)` — swap device I/O error
- `Err(ENOMEM)` — zswap allocation failed (falls back to disk swap)
- `Err(EINVAL)` — bad swap header / version mismatch
- `Err(EBUSY)` — swap device in use during swapoff

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Swap-cluster slot bitmap manipulation | `kani::proofs::mm::swap::cluster_safety` |
| swap_state xarray insert/lookup | `kani::proofs::mm::swap::cache_safety` |
| zswap compressor invocation (raw buffer access) | `kani::proofs::mm::zswap::compress_safety` |
| Swap-entry encoding/decoding | `kani::proofs::mm::swap::entry_codec_safety` |

### Layer 2: TLA+ models

- `models/mm/swap_atomic.tla` (mandatory per `mm/00-overview.md` Layer 2) — proves get_swap_pages + put_swap_pages atomicity. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Swap slot allocator | "Every allocated slot is in exactly one of: in-flight read, in-flight write, mapped to swap-cache folio, free" | `kani::proofs::mm::swap::slot_invariants` |
| Swap-cluster bitmap | "Bits set in cluster bitmap == count_set bits == bound by cluster size" | `kani::proofs::mm::swap::cluster_bitmap_invariants` |
| zswap pool | "Sum of per-pool zsmalloc allocations ≤ max_pool_size" | `kani::proofs::mm::zswap::pool_size_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Swap entry encoding** via Creusot — proves: encode(decode(e)) == e for all valid encoded swap entries.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MEMORY_SANITIZE** (zero on free for swap-out) | When a page is swapped out + freed, the freed page is zeroed (matches `vm.zero_on_free=1`); swap entries themselves are random data + must NOT be zero-on-free (zero would be a valid swap entry collision) | § Default-on configurable off |
| **Swap-page encryption** (when CONFIG_SWAP_ENCRYPTION=y; not currently default per upstream — Rookery should consider escalating to default) | Anon pages encrypted with per-boot random key before write to swap device | (Q1: Rookery may flip default from off to on) |

### Row-1 features consumed by this component

- **REFCOUNT**: per-device refcounts use `Refcount` (saturating)
- **SIZE_OVERFLOW**: swap-entry + offset arithmetic uses checked operators
- **AUTOSLAB**: swap_info_struct allocated via per-type slab cache
- **CONSTIFY**: zswap-compressor vtable is `static const`

### Row-2 / GR-RBAC integration

`swapon`/`swapoff` invoke `security_swap_check` (or equivalent LSM hook). GR-RBAC (per loaded policy) can deny swap operations beyond CAP_SYS_ADMIN.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — `swapon`/`swapoff` syscall argument copies validated; `/proc/swaps` reader output routes through PAX_USERCOPY-checked zones.
- **PAX_KERNEXEC** — swap subsystem .text + rodata RX/RO; zswap compressor vtable CONSTIFY'd.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization on `swapon`/`swapoff` entry; per-page-fault randomization on swap-in path.
- **PAX_REFCOUNT** — `swap_info_struct->users`, swap-cluster refcount, zswap pool refcount saturating.
- **PAX_MEMORY_SANITIZE** — swapped-out anon page contents zeroed from RAM after write completes (the in-RAM copy zeroed; the on-disk copy encrypted per Q1); swap-cache folio freed objects zeroed.
- **PAX_MEMORY_STACKLEAK** — kernel-stack residual zeroing on syscall exit + page-fault exit.
- **PAX_RAP / kCFI** — indirect-call type-signature enforcement on zswap compressor vtable, swap address_space_operations.
- **GRKERNSEC_HIDESYM** — kernel pointers hidden from `/proc/swaps` for non-CAP_SYSLOG; defeats fingerprinting of swap-device layout.
- **GRKERNSEC_DMESG** — swap I/O error dmesg lines (EIO, ENOSPC on swap) restricted to CAP_SYSLOG.
- **Swap-page encryption** (Q1: default-on under consideration) — per-boot random key, AES-XTS-equivalent for every anon page written to swap; defeats cold-boot/forensic recovery of swap contents.
- **`vm.swap_only`** — sysctl restricts swap to swap devices (no zswap, no zram); admin policy knob for swap-discipline systems.
- **swapon CAP_SYS_ADMIN gating** — `swapon`/`swapoff` require CAP_SYS_ADMIN; GR-RBAC can further restrict.
- **GRKERNSEC_KSTACKOVERFLOW** — kernel stack overflow detection on swap-in recursion (alloc → reclaim → swap → alloc).
- **GRKERNSEC_OOM_DENY** — swap-exhausted path's OOM-kill consults grsec OOM-deny policy.
- **Per-memcg swap accounting** — `memory.swap.max` enforced before global OOM; defeats one tenant exhausting swap to OOM another.
- **PAX_USERCOPY on swap-cache reads** — swap-cache folios read into userspace go through `iov_iter` + PAX_USERCOPY validation.

Per-doc rationale: swap is the destination for evicted anonymous pages — every secret a process ever held in heap or stack memory may end up on swap. Without swap-page encryption (Q1 recommends default-on), a cold-boot attack or forensic image of swap device reveals every password, key, and credential the system has ever paged out. PAX_MEMORY_SANITIZE on the in-RAM copy ensures the page is zeroed in RAM after swap-out (so a subsequent slab-spray cannot recover it from RAM). PAX_USERCOPY on swap-in protects against an attacker reading swap-cache folios as user data. The grsec OOM-deny policy prevents an attacker from OOM-killing a privileged process by exhausting swap. Cross-ref Q1 in this doc for the encryption-default escalation.

## Open Questions

<!-- OPEN: Q1 -->
### Q1: Default-enable swap-page encryption
Upstream's CONFIG_SWAP_ENCRYPTION (where present in some downstream patches) is off-by-default. Per the security baseline (`00-security-principles.md`), Rookery considers flipping this to default-on so swapped-out pages are encrypted with a per-boot key. Defeats cold-boot/forensic attacks against swap.

**Recommendation**: Default-on; small CPU cost (per-page AES-XTS-equivalent via the crypto API).

**To resolve**: User confirms or rejects.
<!-- /OPEN -->

## Out of Scope

- zsmalloc internals (cross-ref `mm/zsmalloc.md`)
- swap-on-zRAM (zRAM is a block device; spec'd in `drivers/00-overview.md` § block)
- 32-bit swap (highmem-aware; out per `arch/x86/00-overview.md` D1)
- Implementation code
