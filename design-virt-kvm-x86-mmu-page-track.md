---
title: "Tier-3: arch/x86/kvm/mmu/page_track.c — KVM page-tracker (per-gfn write-monitor for KVMGT, nested-virt, integrity)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

KVM page-tracker is a per-VM service that lets external clients subscribe to per-gfn write events: when guest writes a tracked gfn, KVM faults via shadow MMU + dispatches to all registered notifiers. Critical for KVMGT (Intel GVT-g) which must see guest GTT-update writes to translate guest-GPU-MMU before forwarding to physical GPU; for nested-virt VMCS shadow which must see L1-VMCS writes; for guest-memory integrity verifiers (e.g., AMD SEV measurement). page_track.c (~374 lines) provides per-VM gfn-tracking-mode counter (per-mode, per-gfn) + register/unregister/notify API.

This Tier-3 covers `arch/x86/kvm/mmu/page_track.c` (~374 lines) + `page_track.h` (~58 lines).

### Acceptance Criteria

- [ ] AC-1: KVMGT plug-in test: register write-tracker on guest GTT page; guest writes to GTT; track_write callback fires with correct (gpa, new_data, bytes).
- [ ] AC-2: Multi-client: 2 clients register; both callbacks fire on guest write.
- [ ] AC-3: Per-gfn ref-count: 2 enable-page calls, 1 disable-page call → SPTE still write-protected; 2 disable-page calls → unprotected on next fault.
- [ ] AC-4: Memslot delete: track_remove_slot fires for each tracked gfn; clients clean up.
- [ ] AC-5: Notifier unregister: in-flight callback completes before unregister returns; verified via SRCU sync.
- [ ] AC-6: Performance: 4KB-tracked-page guest stress test (1M writes); per-write notifier latency < 10us.
- [ ] AC-7: ACCESS-mode tracking (KVMGT): guest access (read or write) faults + notifies.
- [ ] AC-8: Spec-violation defense: page_track_register_notifier with NULL track_write → -EINVAL.

### Architecture

`PageTracker` per-VM:

```
struct PageTracker {
  notifier_head: HlistHeadRcu<KArc<PageTrackNotifierNode>>,
  srcu: SrcuStruct,                          // for notifier list
}

struct PageTrackNotifierNode {
  link: HlistNode,
  track_write: TrackWriteFn,
  track_create_slot: Option<TrackCreateSlotFn>,
  track_remove_slot: Option<TrackRemoveSlotFn>,
  track_flush_slot: Option<TrackFlushSlotFn>,
}

// Embedded in kvm_memory_slot:
struct ArchMemorySlot {
  ...
  gfn_track: [Option<KVec<u16>>; KVM_PAGE_TRACK_MAX],
}
```

`PageTracker::init(kvm)`:
1. Init `notifier_head` empty.
2. Init `srcu`.

`PageTracker::register_notifier(kvm, node)`:
1. Acquire `kvm.lock` (write).
2. hlist_add_head_rcu(node, &kvm.arch.track_notifier_head).
3. Release lock.

`PageTracker::unregister_notifier(kvm, node)`:
1. Acquire kvm.lock.
2. hlist_del_rcu(node).
3. Release lock.
4. synchronize_srcu(kvm.arch.track_srcu).

`PageTracker::slot_track_add_page(kvm, slot, gfn, mode)`:
1. Acquire `kvm.mmu_lock`.
2. offset = gfn - slot.base_gfn.
3. count = slot.arch.gfn_track[mode][offset]++.
4. If count == 0 (was unset): `kvm_mmu_slot_gfn_write_protect(kvm, slot, gfn)`.
5. Release mmu_lock.

`PageTracker::slot_track_remove_page(kvm, slot, gfn, mode)`:
1. Acquire kvm.mmu_lock.
2. offset = gfn - slot.base_gfn.
3. count = --slot.arch.gfn_track[mode][offset].
4. If count == 0: SPTE will be unprotected lazily.
5. Release.

`PageTracker::dispatch_write(vcpu, gpa, new_data, bytes)`:
1. idx = srcu_read_lock(kvm.arch.track_srcu).
2. hlist_for_each_entry_rcu(node, kvm.arch.track_notifier_head):
   - node.track_write(vcpu, gpa, new_data, bytes, node).
3. srcu_read_unlock(idx).

`PageTracker::create_memslot(slot)`:
1. For each mode in {WRITE, ACCESS}:
   - If any client interested in mode: alloc slot.arch.gfn_track[mode] = KVec<u16>(npages).
2. Per-notifier: invoke track_create_slot(kvm, slot, node).

### Out of Scope

- Shadow MMU (covered in `x86-mmu-shadow.md` Tier-3)
- TDP MMU (covered in `x86-mmu-tdp.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- KVMGT GPU plug-in details (covered in `gpu/i915-gvt.md` future Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `enum kvm_page_track_mode` | tracking modes {WRITE, ACCESS} | `kernel::kvm::x86::mmu::PageTrackMode` |
| `struct kvm_page_track_notifier_node` | per-client notifier reg | `PageTrackNotifierNode` |
| `kvm_page_track_register_notifier(kvm, node)` | per-VM client register | `PageTracker::register_notifier` |
| `kvm_page_track_unregister_notifier(kvm, node)` | per-VM client deregister | `PageTracker::unregister_notifier` |
| `kvm_page_track_create_memslot(...)` | per-memslot alloc gfn-track arrays | `PageTracker::create_memslot` |
| `kvm_page_track_free_memslot(...)` | per-memslot free arrays | `PageTracker::free_memslot` |
| `kvm_slot_page_track_add_page(kvm, slot, gfn, mode)` | per-gfn enable tracking | `PageTracker::slot_track_add_page` |
| `kvm_slot_page_track_remove_page(kvm, slot, gfn, mode)` | per-gfn disable | `PageTracker::slot_track_remove_page` |
| `kvm_slot_page_track_is_active(kvm, slot, gfn, mode)` | query | `PageTracker::is_active` |
| `kvm_page_track_write(vcpu, gpa, new_data, bytes)` | dispatch write event | `PageTracker::dispatch_write` |
| `kvm_page_track_flush_slot(kvm, slot)` | per-memslot teardown | `PageTracker::flush_slot` |
| `kvm_page_track_init(kvm)` | per-VM init | `PageTracker::init` |
| `kvm_page_track_cleanup(kvm)` | per-VM teardown | `PageTracker::cleanup` |
| `update_gfn_track(slot, gfn, mode, count)` | inc/dec per-gfn-mode counter | `PageTracker::update_gfn_track` |

### compatibility contract

REQ-1: Per-memslot per-mode gfn-track array:
- `kvm_memory_slot.arch.gfn_track[KVM_PAGE_TRACK_MAX]` array of `unsigned short *`.
- Per-mode array indexed by gfn-offset-in-slot.
- Each entry is ref-count: ≥1 → track-active; 0 → not-tracked.
- Allocated lazily at `create_memslot` per-mode if any client subscribed for mode.

REQ-2: Per-mode tracking semantics:
- WRITE: KVM write-protects SPTE for tracked gfn → guest write → page-fault → KVM emulates write + dispatches to notifiers + restores SPTE (or keeps write-protected per-mode).
- ACCESS (rare; KVMGT only): KVM marks SPTE as not-present → any guest access → fault → notify.

REQ-3: Per-VM notifier list:
- `kvm.arch.track_notifier_head` — head of registered notifier list.
- Each `kvm_page_track_notifier_node` has callbacks: `track_write`, `track_remove_region`, `track_create_slot`, `track_remove_slot`, `track_flush_slot`.
- Callbacks invoked under SRCU read-lock.

REQ-4: Register/Unregister:
- `register_notifier(kvm, node)`: hlist-add-rcu to `kvm.arch.track_notifier_head`.
- `unregister_notifier(kvm, node)`: hlist-del-rcu + synchronize_srcu to ensure no in-flight callback.

REQ-5: Per-gfn enable tracking:
- `slot_track_add_page(kvm, slot, gfn, mode)`:
  - Increment slot.arch.gfn_track[mode][gfn-offset].
  - On 0→1 transition: `kvm_mmu_slot_gfn_write_protect(kvm, slot, gfn)` to write-protect existing SPTEs.
  - Future SPTE installs check `is_active` and apply write-protect.

REQ-6: Per-gfn disable tracking:
- `slot_track_remove_page(kvm, slot, gfn, mode)`:
  - Decrement counter.
  - On 1→0 transition: SPTE may be unprotected (lazily on next page-fault).

REQ-7: Write event dispatch:
- `kvm_page_track_write(vcpu, gpa, new_data, bytes)`:
  - Acquire SRCU read-lock.
  - For each notifier_node in `kvm.arch.track_notifier_head`:
    - Call `node->track_write(vcpu, gpa, new_data, bytes, node)`.
  - Release SRCU.

REQ-8: Memslot lifecycle hooks:
- `track_create_slot`: callback dispatched when new memslot created (for KVMGT to alloc per-slot tracking).
- `track_remove_slot`: pre-delete dispatch.
- `track_flush_slot`: per-slot full-flush.

REQ-9: Mode validity:
- Modes ∈ {WRITE, ACCESS}; `KVM_PAGE_TRACK_MAX` enforced at compile time.
- Per-mode counter capped at u16 max (defense against ref-count overflow on pathological subscribe).

REQ-10: Integration with shadow MMU:
- `kvm_mmu_pte_write` (shadow MMU) calls `kvm_page_track_write` on guest write to shadowed page.
- TDP-MMU integration (less common): write-protected SPTE → fault → emulator + dispatch.

REQ-11: Per-notifier ordering:
- Notifiers called in registration order; clients should be order-independent.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `gfn_track_no_oob` | OOB | per-(slot, gfn, mode) gfn_track array indexed by gfn-offset; bounded by slot.npages. |
| `notifier_list_srcu_safe` | UAF | per-notifier accessed under SRCU read-lock; unregister synchronizes; defense against notifier-free during dispatch. |
| `gfn_track_no_overflow` | INVARIANT | per-counter u16 max enforced at add-page; defense against ref-count wrap. |
| `mmu_lock_held_for_write_protect` | INVARIANT | write-protect operations hold kvm.mmu_lock. |

### Layer 2: TLA+

`virt/kvm/page_track_lifecycle.tla`:
- States: NotTracked, Tracked(n), TrackedWriteProtected(n).
- Transitions:
  - NotTracked → Tracked(1) via add_page (n was 0).
  - Tracked(n) → Tracked(n+1) via add_page.
  - Tracked(n>1) → Tracked(n-1) via remove_page.
  - Tracked(1) → NotTracked via remove_page.
  - Tracked(n) → TrackedWriteProtected(n) via SPTE write-protect.
  - TrackedWriteProtected(n) on guest-write → emulate + notify + back to Tracked(n) or back to TrackedWriteProtected.
- Properties:
  - `safety_no_track_n_zero` — n == 0 implies NotTracked.
  - `safety_notify_only_when_tracked` — track_write notify dispatch only if at least one notifier registered.
  - `liveness_eventual_unprotect` — after final remove_page, SPTE eventually unprotected on next fault.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `register_notifier` post: node in head_rcu list; visible to subsequent dispatch | `PageTracker::register_notifier` |
| `unregister_notifier` post: node not in list; SRCU-sync ensures no in-flight callback | `PageTracker::unregister_notifier` |
| `slot_track_add_page` post: counter incremented; SPTE write-protected on 0→1 | `PageTracker::slot_track_add_page` |
| `dispatch_write` post: every registered notifier called exactly once for this event | `PageTracker::dispatch_write` |
| Per-memslot.arch.gfn_track[mode] allocated iff at least one client interested in mode | `PageTracker::create_memslot` |

### Layer 4: Verus/Creusot functional

`Per-gfn write-protect + guest-write → notifier-dispatch → emulator` round-trip equivalence: per-write event observed by every registered notifier with exact-byte (gpa, new_data, bytes) tuple.

### hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

page-tracker-specific reinforcement:

- **Per-gfn-track counter u16 cap** — defense against ref-count overflow on pathological subscribe.
- **SRCU-protected notifier list** — defense against unregister-during-dispatch UAF.
- **kvm.mmu_lock for write-protect** — defense against concurrent SPTE-update during enable.
- **Per-notifier callbacks called in atomic-safe ctx** — clients must not sleep in track_write; defense against blocking guest-execution path.
- **Per-memslot.arch.gfn_track allocated under kvm.slots_lock** — defense against torn-allocation during memslot setup.
- **Per-memslot delete dispatches track_remove_slot before per-gfn track-counter zero** — defense against dispatch-after-mem-free.
- **Per-mode array size matches slot.npages** — defense against gfn-offset OOB.
- **Per-VM notifier list bounded by KVM_PAGE_TRACK_NOTIFIER_MAX** — defense against unbounded subscribe.
- **Notifier callbacks must not subscribe additional notifiers** (single-write nesting) — defense against re-entrant subscribe causing list mutation during walk.

