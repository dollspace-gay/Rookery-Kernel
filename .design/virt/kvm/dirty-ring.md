# Tier-3: virt/kvm/dirty_ring.c — KVM dirty-page ring buffer (per-vCPU)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/dirty-tracking.md
upstream-paths:
  - virt/kvm/dirty_ring.c (~271 lines)
  - include/linux/kvm_host.h (kvm_dirty_ring, kvm_dirty_gfn)
  - include/uapi/linux/kvm.h (KVM_CAP_DIRTY_LOG_RING + KVM_RESET_DIRTY_RINGS)
-->

## Summary

Per-vCPU dirty-page ring is the modern KVM dirty-tracking ABI (KVM_CAP_DIRTY_LOG_RING; ~7.0+) replacing global per-memslot dirty-bitmap with per-vCPU lock-free ring of `kvm_dirty_gfn { slot, offset, flags }` entries. Per-vCPU writes via `kvm_dirty_ring_push` at write-protect-fault; userspace reads via mmap'd ring page. Per-entry transitions: INVALID → DIRTY (kernel) → RESET (userspace acks via KVM_RESET_DIRTY_RINGS ioctl) → INVALID. Per-ring full → vCPU vmexits with KVM_EXIT_DIRTY_RING_FULL forcing flush. Critical for: live migration with reduced lock contention vs traditional bitmap; ARM/x86 without HW DIRTY-bit (uses softdirty).

This Tier-3 covers `dirty_ring.c` (~271 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_dirty_ring` | per-vCPU ring | `KvmDirtyRing` |
| `struct kvm_dirty_gfn` | per-entry slot+offset+flags | `KvmDirtyGfn` |
| `kvm_dirty_ring_alloc()` | per-vCPU alloc | `DirtyRing::alloc` |
| `kvm_dirty_ring_free()` | per-vCPU free | `DirtyRing::free` |
| `kvm_dirty_ring_push()` | per-vCPU push entry on fault | `DirtyRing::push` |
| `kvm_dirty_ring_reset()` | userspace ack-reset entries | `DirtyRing::reset` |
| `kvm_dirty_ring_check_request()` | per-vmenter ring-full check | `DirtyRing::check_request` |
| `kvm_cpu_dirty_log_size()` | per-arch slot+offset entry size | `DirtyRing::cpu_log_size` |
| `KVM_DIRTY_GFN_F_DIRTY` (bit 0) | per-entry dirty | UAPI |
| `KVM_DIRTY_GFN_F_RESET` (bit 1) | per-entry userspace-reset | UAPI |
| `KVM_DIRTY_GFN_F_MASK` | per-entry flag mask | UAPI |
| `KVM_DIRTY_RING_RSVD_ENTRIES` | per-vCPU reserve cap | shared |
| `KVM_EXIT_DIRTY_RING_FULL` | per-vmexit reason | UAPI |

## Compatibility contract

REQ-1: KVM_CAP_DIRTY_LOG_RING capability:
- Userspace enables via KVM_ENABLE_CAP(KVM_CAP_DIRTY_LOG_RING, ring_size).
- ring_size: power-of-2; per-vCPU ring entry count.
- Must be enabled before vCPU creation.

REQ-2: Per-vCPU ring layout:
- mmap'd page region of size = ring_size × sizeof(KvmDirtyGfn) (16 bytes per entry).
- Userspace mmaps via vCPU fd offset KVM_DIRTY_LOG_PAGE_OFFSET.
- Per-entry: slot (u32), offset (u32), flags (u32), pad (u32).

REQ-3: Per-entry flag transitions:
- 00 (INVALID): empty slot.
- 01 (DIRTY): kernel pushed entry (KVM_DIRTY_GFN_F_DIRTY).
- 11 (DIRTY|RESET): userspace ack (after migrating page).
- Kernel transitions 11 → 00 (INVALID) on KVM_RESET_DIRTY_RINGS.

REQ-4: kvm_dirty_ring_push:
- Per-write-fault under mmu_lock: capture (slot, offset).
- Increment ring.dirty_index.
- Get entry at idx % ring.size; verify flags == 0 (INVALID); set slot+offset+DIRTY.
- Per-ring full: ring.dirty_index - ring.reset_index >= ring.size - RSVD.

REQ-5: kvm_dirty_ring_reset:
- Iterate ring entries from ring.reset_index..ring.dirty_index.
- For each entry with F_RESET bit set: clear flags (00); ring.reset_index++.
- Returns count of reset entries.

REQ-6: KVM_EXIT_DIRTY_RING_FULL:
- Per-vmenter: if ring soft-full: vmexit with reason DIRTY_RING_FULL.
- Userspace must call KVM_RESET_DIRTY_RINGS to recover.

REQ-7: Per-vCPU ring alloc:
- alloc_pages_node ring_size × 16 bytes (e.g. 4096 entries = 64KB).
- Each ring page mapped into per-vCPU mmap region.
- Page locked in kernel; not swapped.

REQ-8: Per-RSVD entries:
- KVM_DIRTY_RING_RSVD_ENTRIES = 64 + cpu_dirty_log_size.
- Per-vmenter wakes when avail < RSVD.

REQ-9: Per-vCPU push under mmu_lock:
- Atomically reserved index via memory-barrier.
- Per-push must succeed before mmu_lock unlock.

REQ-10: Per-arch cpu_dirty_log_size:
- x86: 1 (single GFN per fault).
- ARM/RISC-V: may be > 1 (huge-page).

REQ-11: Per-VM kvm.dirty_ring_size:
- All vCPUs share same size.
- Set at first ENABLE_CAP; immutable thereafter.

## Acceptance Criteria

- [ ] AC-1: KVM_ENABLE_CAP(DIRTY_LOG_RING, 4096): per-vCPU ring of 4096 entries.
- [ ] AC-2: vCPU mmap KVM_DIRTY_LOG_PAGE_OFFSET: 64KB region accessible.
- [ ] AC-3: Guest write to write-protected page: kernel pushes entry; F_DIRTY set.
- [ ] AC-4: Userspace ack via F_RESET bit; KVM_RESET_DIRTY_RINGS clears entry.
- [ ] AC-5: Ring near-full: vmexit KVM_EXIT_DIRTY_RING_FULL.
- [ ] AC-6: Per-vCPU rings independent; cross-vCPU push doesn't collide.
- [ ] AC-7: Per-VM dirty_ring_size locked after first cap-enable.
- [ ] AC-8: Per-VM destroy: per-vCPU ring freed.
- [ ] AC-9: Per-vCPU live-migrate: ring state cleared; new dest re-enables.
- [ ] AC-10: kvm-unit-tests `dirty_ring` test passes.

## Architecture

Per-vCPU dirty-ring state:

```
struct KvmDirtyRing {
  dirty_index: u32,                              // kernel write position
  reset_index: u32,                              // userspace ack position
  size: u32,                                     // entry count (power-of-2)
  soft_limit: u32,                               // size - RSVD
  dirty_gfns: Box<[KvmDirtyGfn]>,                // mmap'd ring
  index: u32,                                    // per-vCPU id (for trace)
}

#[repr(C)]
struct KvmDirtyGfn {
  flags: u32,                                    // F_DIRTY | F_RESET
  slot: u32,
  offset: u64,
}
```

Per-vCPU:

```
struct Vcpu {
  ...
  dirty_ring: KvmDirtyRing,
}
```

Per-VM:

```
struct KvmArch {
  ...
  dirty_ring_size: u32,
}
```

`DirtyRing::alloc(kvm, ring, size, index) -> Result<()>`:
1. Validate size is power-of-2.
2. ring.size = size.
3. ring.soft_limit = size - kvm_dirty_ring_get_rsvd_entries().
4. ring.dirty_gfns = alloc_pages(size × sizeof(KvmDirtyGfn)).
5. memset(ring.dirty_gfns, 0, ring_bytes).
6. ring.dirty_index = 0.
7. ring.reset_index = 0.
8. ring.index = index.
9. Ok.

`DirtyRing::push(vcpu, slot, offset)`:
1. ring = &mut vcpu.dirty_ring.
2. cur_idx = ring.dirty_index.
3. entry_idx = cur_idx & (ring.size - 1).
4. entry = &mut ring.dirty_gfns[entry_idx].
5. assert(entry.flags == 0).
6. entry.slot = slot.
7. entry.offset = offset.
8. memory_barrier().
9. entry.flags = KVM_DIRTY_GFN_F_DIRTY.
10. ring.dirty_index = cur_idx + 1.

`DirtyRing::reset(kvm, ring) -> u32`:
1. count = 0.
2. cur = ring.reset_index.
3. while cur < ring.dirty_index:
   - entry_idx = cur & (ring.size - 1).
   - entry = &mut ring.dirty_gfns[entry_idx].
   - if entry.flags & KVM_DIRTY_GFN_F_RESET == 0: break.
   - entry.flags = 0.
   - cur++.
   - count++.
4. ring.reset_index = cur.
5. trace_kvm_dirty_ring_reset(ring).
6. Return count.

`DirtyRing::check_request(vcpu) -> bool`:
1. ring = &vcpu.dirty_ring.
2. avail = ring.size - (ring.dirty_index - ring.reset_index).
3. Return avail < ring.soft_limit.

`DirtyRing::full_vmexit(vcpu)`:
1. vcpu.run.exit_reason = KVM_EXIT_DIRTY_RING_FULL.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `size_power_of_2` | INVARIANT | ring.size is power-of-2 ≥ 1. |
| `entry_idx_lt_size` | INVARIANT | per-push entry_idx < ring.size. |
| `dirty_index_ge_reset_index` | INVARIANT | ring.dirty_index ≥ ring.reset_index. |
| `flags_in_set` | INVARIANT | per-entry.flags ∈ {0, F_DIRTY, F_DIRTY|F_RESET}. |
| `push_only_to_invalid_slot` | INVARIANT | per-push: target entry.flags == 0. |

### Layer 2: TLA+

`virt/kvm/dirty_ring.tla`:
- Per-vCPU push + per-userspace reset + ring-full transition.
- Properties:
  - `safety_no_overwrite` — per-push only into INVALID slot.
  - `safety_reset_only_acked` — per-reset only F_RESET-marked entries.
  - `liveness_eventual_drain` — per-userspace KVM_RESET_DIRTY_RINGS ⟹ ring drains.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `DirtyRing::alloc` post: ring zeroed; size + soft_limit set | `DirtyRing::alloc` |
| `DirtyRing::push` post: entry.flags == F_DIRTY; dirty_index incremented | `DirtyRing::push` |
| `DirtyRing::reset` post: F_RESET-marked entries cleared; reset_index advanced | `DirtyRing::reset` |
| `DirtyRing::check_request` post: returns true ⟺ avail < soft_limit | `DirtyRing::check_request` |

### Layer 4: Verus/Creusot functional

`Per-vCPU dirty-page write → kernel-push entry → userspace mmap-read → ack-flag → kernel-reset clears entry → ring re-usable` semantic equivalence: per-KVM_CAP_DIRTY_LOG_RING ABI matches kvm.h spec.

## Hardening

(Inherits row-1 features from `virt/kvm/dirty-tracking.md` § Hardening.)

Dirty-ring-specific reinforcement:

- **Per-ring entry-flag state-machine** — defense against partial-write entry confusion.
- **Per-push memory-barrier before flags** — defense against userspace seeing slot/offset stale.
- **Per-reset only F_RESET entries cleared** — defense against premature-clear losing dirty info.
- **Per-RSVD margin triggers vmexit** — defense against ring-full silent-drop.
- **Per-vCPU ring page locked** — defense against host swap migrating ring mid-push.
- **Per-VM dirty_ring_size locked-once** — defense against per-resize race.
- **Per-arch RSVD ≥ 64** — defense against arch-specific large-batch push overrun.
- **Per-vCPU ring index unique** — defense against cross-vCPU collision.
- **Per-mmap KVM_DIRTY_LOG_PAGE_OFFSET only after ENABLE_CAP** — defense against per-uninit ring access.
- **Per-vmenter check_request gates dirty work** — defense against per-vCPU silently overrunning.
- **Per-VM noncoherent_dma_count agnostic** — dirty-ring works orthogonally to PAT-quirk.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- Dirty bitmap (legacy; covered in `dirty-tracking.md` Tier-3)
- KVM live migration (covered separately)
- ARM dirty-tracking specifics (covered separately)
- Implementation code
