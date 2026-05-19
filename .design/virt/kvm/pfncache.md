# Tier-3: virt/kvm/pfncache.c — KVM gfn-to-pfn cache (per-vCPU hot-mapped guest pages)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - virt/kvm/pfncache.c (~484 lines)
  - include/linux/kvm_host.h (struct gfn_to_pfn_cache)
-->

## Summary

`gfn_to_pfn_cache` (gpc) caches a per-(GPA, len) → kernel-virtual-address (kva) mapping for KVM hot-paths that need fast access to specific guest pages without per-access GPA-walk + pin-page overhead. Examples: per-vCPU steal-time MSR target, per-vCPU PV-clock page, per-vCPU async-PF reason, Hyper-V SynIC/STIMER pages. Per-gpc tracks generation: when guest memslot mutated → invalidated; per-vCPU re-resolves on next check. Per-gpc protected by per-cache rwlock for fast read + slow refresh. Critical for: per-IRQ-rate / per-vmexit hot-path latency.

This Tier-3 covers `pfncache.c` (~484 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct gfn_to_pfn_cache` | per-cache state | `GfnToPfnCache` |
| `kvm_gpc_init()` | per-cache init | `Gpc::init` |
| `kvm_gpc_activate()` | per-cache bind to GPA | `Gpc::activate` |
| `kvm_gpc_activate_hva()` | per-cache bind to userspace HVA | `Gpc::activate_hva` |
| `kvm_gpc_deactivate()` | per-cache release | `Gpc::deactivate` |
| `kvm_gpc_check()` | per-vCPU validity check | `Gpc::check` |
| `kvm_gpc_refresh()` | per-vCPU re-resolve | `Gpc::refresh` |
| `__kvm_gpc_refresh()` | internal refresh | `Gpc::__refresh` |
| `gpc_invalidate_start()` | per-memslot mutation invalidator | `Gpc::invalidate_start` |
| `kvm_gpc_mark_dirty()` | per-write mark guest page dirty | `Gpc::mark_dirty` |
| `kvm.gpc_lock` | per-VM list of all gpcs | `KvmArch::gpc_lock` |
| `kvm.gpc_list` | per-VM list of all gpcs | `KvmArch::gpc_list` |
| `KVM_HVA_ERR_BAD` | sentinel for invalid HVA | shared |

## Compatibility contract

REQ-1: Per-gpc state:
- `kvm`: parent VM ref.
- `gpa`: guest physical addr.
- `uhva`: userspace virtual addr (if HVA-anchored).
- `khva`: kernel virtual addr (mapped page-window).
- `pfn`: pinned physical frame number.
- `valid`: bool.
- `len`: window length (≤ 2 pages).
- `lock`: rwlock for read/refresh.
- `active`: bool.

REQ-2: `kvm_gpc_init(gpc, kvm)`:
- gpc.kvm = kvm.
- gpc.active = false; valid = false.
- list_add(&gpc.list, &kvm.gpc_list) under gpc_lock.

REQ-3: `kvm_gpc_activate(gpc, gpa, len)`:
- Validate len ≤ 2 × PAGE_SIZE.
- gpc.gpa = gpa; gpc.uhva = KVM_HVA_ERR_BAD; gpc.len = len.
- gpc.active = true.
- Call __kvm_gpc_refresh.

REQ-4: `kvm_gpc_activate_hva(gpc, uhva, len)`:
- Validate len ≤ 2 × PAGE_SIZE.
- gpc.gpa = INVALID_GPA; gpc.uhva = uhva; gpc.len = len.
- gpc.active = true.
- Call __kvm_gpc_refresh.

REQ-5: `kvm_gpc_check(gpc, len)`:
- Read gpc.lock as reader.
- Return gpc.active ∧ gpc.valid ∧ gpc.len ≥ len.
- Per-fast-path: O(1).

REQ-6: `kvm_gpc_refresh(gpc, len)`:
- Take gpc.lock as writer.
- If !gpc.active: return Err.
- Resolve uhva from gpa (per-memslot lookup).
- Pin guest page; map kernel mapping.
- Update gpc.pfn / gpc.khva.
- Set gpc.valid = true.

REQ-7: `gpc_invalidate_start(kvm)`:
- Per-memslot delete/move: walk kvm.gpc_list.
- For each gpc with affected GPA: take writer-lock; gpc.valid = false; release.
- Per-vCPU subsequent check returns false → triggers refresh.

REQ-8: `kvm_gpc_mark_dirty(gpc)`:
- Per-write to khva: kvm_vcpu_mark_page_dirty(gpc.kvm, gpc.gpa >> PAGE_SHIFT).
- For dirty-tracking integration.

REQ-9: Per-len ≤ 2 × PAGE_SIZE:
- Allows page-spanning resolution.
- Beyond 2 pages: not supported (use per-page gpcs).

REQ-10: Per-cache lifetime:
- Active gpcs survive vCPU teardown until kvm_gpc_deactivate.
- Per-VM destroy: iterates kvm.gpc_list and force-deactivates.

REQ-11: Per-MMU-notifier:
- Per-mm-page-mover (e.g. KSM, migration): MMU notifier invalidates gpc.

## Acceptance Criteria

- [ ] AC-1: kvm_gpc_init(&pvclock_gpc, kvm): gpc in kvm.gpc_list.
- [ ] AC-2: kvm_gpc_activate(&pvclock_gpc, gpa, 32): gpc.valid = true; gpc.khva mapped.
- [ ] AC-3: kvm_gpc_check(&pvclock_gpc, 32): returns true.
- [ ] AC-4: Per-memslot delete affecting gpa: gpc.valid = false.
- [ ] AC-5: kvm_gpc_refresh after invalidate: gpc.valid = true; new khva.
- [ ] AC-6: Per-write to gpc.khva + mark_dirty: kvm.dirty_bitmap (or dirty-ring) updated.
- [ ] AC-7: kvm_gpc_deactivate: gpc.active = false; page unpinned.
- [ ] AC-8: Per-VM destroy: all gpcs in kvm.gpc_list deactivated.
- [ ] AC-9: Per-MMU-notifier huge-page split: gpc invalidated.
- [ ] AC-10: 2-page-window: gpc covers spanning correctly.

## Architecture

Per-cache state:

```
struct GfnToPfnCache {
  list: ListLink,                                // kvm.gpc_list
  kvm: &Kvm,
  vcpu: Option<&Vcpu>,                           // owning vCPU (optional)
  gpa: GpaT,
  uhva: HvaT,
  khva: KvaT,                                    // kernel-virtual addr in cache window
  pfn: PfnT,
  active: bool,
  valid: bool,
  len: u64,                                      // ≤ 2 × PAGE_SIZE
  lock: RwLock<()>,
  generation: AtomicU64,                          // per-invalidate
}
```

Per-VM:

```
struct KvmArch {
  ...
  gpc_lock: SpinLock,
  gpc_list: ListHead<GfnToPfnCache>,
}
```

`Gpc::init(gpc, kvm)`:
1. gpc.kvm = kvm.
2. gpc.active = false; gpc.valid = false.
3. spin_lock(&kvm.arch.gpc_lock).
4. list_add(&gpc.list, &kvm.arch.gpc_list).
5. spin_unlock.

`Gpc::activate(gpc, gpa, len) -> Result<()>`:
1. Validate len ≤ 2 × PAGE_SIZE.
2. write_lock(&gpc.lock).
3. gpc.gpa = gpa; gpc.uhva = KVM_HVA_ERR_BAD.
4. gpc.len = len.
5. gpc.active = true.
6. err = Gpc::__refresh(gpc, gpa, KVM_HVA_ERR_BAD).
7. write_unlock.
8. err.

`Gpc::activate_hva(gpc, uhva, len) -> Result<()>`:
1. Same as activate but (uhva, INVALID_GPA, len).

`Gpc::__refresh(gpc, gpa, uhva) -> Result<()>`:
1. If gpa != INVALID_GPA:
   - uhva_resolved = kvm_gfn_to_hva(gpc.kvm, gpa >> PAGE_SHIFT).
   - if uhva_resolved == KVM_HVA_ERR_BAD: gpc.valid = false; return Err.
   - uhva = uhva_resolved.
2. If gpc.uhva == uhva ∧ gpc.valid: return Ok (already valid).
3. Pin guest page: pfn = hva_to_pfn(uhva).
4. khva = kmap(pfn).
5. Per-2-page span: pin second page; vmap to khva contiguously.
6. gpc.pfn = pfn; gpc.khva = khva; gpc.uhva = uhva.
7. gpc.valid = true.
8. gpc.generation++.

`Gpc::check(gpc, len) -> bool`:
1. read_lock(&gpc.lock).
2. ret = gpc.active ∧ gpc.valid ∧ gpc.len ≥ len.
3. read_unlock.
4. Return ret.

`Gpc::refresh(gpc, len) -> Result<()>`:
1. write_lock(&gpc.lock).
2. err = Gpc::__refresh(gpc, gpc.gpa, gpc.uhva).
3. write_unlock.
4. err.

`Gpc::invalidate_start(kvm)`:
1. spin_lock(&kvm.arch.gpc_lock).
2. For gpc in kvm.arch.gpc_list:
   - write_lock(&gpc.lock).
   - gpc.valid = false.
   - write_unlock.
3. spin_unlock.

`Gpc::deactivate(gpc)`:
1. write_lock(&gpc.lock).
2. gpc.active = false.
3. If gpc.valid: kunmap(gpc.khva); kvm_release_pfn(gpc.pfn).
4. gpc.valid = false.
5. write_unlock.
6. spin_lock(&gpc.kvm.arch.gpc_lock).
7. list_del(&gpc.list).
8. spin_unlock.

`Gpc::mark_dirty(gpc)`:
1. read_lock(&gpc.lock).
2. If gpc.valid ∧ gpc.gpa != INVALID_GPA: kvm_vcpu_mark_page_dirty(gpc.kvm, gpc.gpa >> PAGE_SHIFT).
3. read_unlock.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `len_le_2_pages` | INVARIANT | per-activate len ≤ 2 × PAGE_SIZE. |
| `valid_implies_active` | INVARIANT | gpc.valid ⟹ gpc.active. |
| `khva_mapped_iff_valid` | INVARIANT | gpc.valid ⟺ gpc.khva mapped to gpc.pfn. |
| `gpc_in_list_iff_active_or_init` | INVARIANT | gpc.list non-empty ⟺ gpc-init'd. |
| `lock_held_during_refresh` | INVARIANT | per-refresh: write_lock held. |

### Layer 2: TLA+

`virt/kvm/pfncache.tla`:
- Per-gpc activate / refresh / invalidate / deactivate.
- Properties:
  - `safety_no_use_after_invalidate` — per-fast-read: gpc.valid before khva access.
  - `safety_invalidate_propagates` — per-memslot-mutation ⟹ all matching gpc.valid=false.
  - `liveness_refresh_succeeds` — per-active gpc + valid GPA ⟹ refresh restores valid.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Gpc::activate` post: gpc.active=true; valid iff GPA resolved | `Gpc::activate` |
| `Gpc::__refresh` post: gpc.khva maps gpc.pfn corresponding to gpc.uhva | `Gpc::__refresh` |
| `Gpc::invalidate_start` post: per-gpc-in-list valid=false | `Gpc::invalidate_start` |
| `Gpc::deactivate` post: gpc.active=false; pfn released | `Gpc::deactivate` |

### Layer 4: Verus/Creusot functional

`Per-vCPU hot-MSR (steal-time / pvclock) write → Gpc::check fast-path → if valid, write to khva → mark_dirty for live-migration` semantic equivalence: per-gpc matches kernel API contract for hot-page mapping.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

GPC-specific reinforcement:

- **Per-len bounded ≤ 2 pages** — defense against per-window OOB access.
- **Per-rwlock fast read-path** — defense against per-IRQ-rate refresh-stall.
- **Per-invalidate iterates gpc_list** — defense against per-memslot-stale gpc.
- **Per-mark_dirty under read-lock** — defense against per-mark race.
- **Per-deactivate releases pfn** — defense against UAF post-deactivate.
- **Per-list under gpc_lock** — defense against concurrent list mutation.
- **Per-MMU-notifier integrated** — defense against per-mm-page-mover stale khva.
- **Per-gpa = INVALID_GPA HVA path** — defense against per-userspace-managed page semantics.
- **Per-VM destroy iterates gpc_list** — defense against post-VM-destroy lingering gpc.
- **Per-2-page span vmap-contiguous** — defense against per-cross-page khva non-contiguous.

## Grsecurity/PaX-style Reinforcement

Baseline hardening (always applied):

- **PAX_USERCOPY** — gpc khva read/write bounded by `gpc->len` ≤ 2 pages; rejects user-supplied lengths past cache window.
- **PAX_KERNEXEC** — gpc op-vtables read-only after publish.
- **PAX_RANDKSTACK** — randomized kstack on KVM_RUN paths that drive gpc_refresh.
- **PAX_REFCOUNT** — `struct gfn_to_pfn_cache` activate/deactivate ref counted (saturating).
- **PAX_MEMORY_SANITIZE** — pfn/khva fields zeroed on deactivate; vmap'd pages poisoned before free.
- **PAX_UDEREF** — INVALID_GPA HVA path validates user range before copy.
- **PAX_RAP / kCFI** — gpc check/refresh callbacks indirect-call type-checked.
- **GRKERNSEC_HIDESYM** — gpa/khva/pfn redacted in tracepoints.
- **GRKERNSEC_DMESG** — gpc invalidation storms rate-limited.

GPC-specific:

- **CAP_SYS_ADMIN on KVM ioctls that arm a gpc** (kvm_xen, kvm_pv shared pages) — defense against unprivileged guest-shared-page setup.
- **MMU-notifier release path PAX_REFCOUNT-aware** — gpc cannot be re-armed mid-tear-down.
- **2-page span MEMORY_SANITIZE on cross-page rollover** — defense against stale neighbour-page bytes.

Rationale: gpc holds a long-lived kernel mapping of guest memory; UAF here grants host-kernel arbitrary R/W. Sanitize-on-deactivate + saturating refs close the dominant escalation paths.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- KVM memslots (covered in `memslots.md` Tier-3)
- KVM dirty-tracking (covered in `dirty-tracking.md` + `dirty-ring.md` Tier-3)
- KVM PV-clock (covered in `x86-pvclock.md` Tier-3)
- KVM steal-time (covered in `x86-steal-time.md` Tier-3)
- KVM async-PF (covered in `x86-async-pf.md` Tier-3)
- Implementation code
