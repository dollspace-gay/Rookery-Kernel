---
title: "Tier-3: virt/kvm/dirty_ring.c — Per-vCPU dirty-ring + per-slot dirty bitmap (live migration acq-rel ordering)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

KVM's two complementary dirty-page-tracking mechanisms used for live migration:

- **Per-slot dirty bitmap** (legacy mode, `KVM_MEM_LOG_DIRTY_PAGES`): per-memslot allocated bitmap; each bit = one 4K page; `KVM_GET_DIRTY_LOG` ioctl reads + clears (atomic via `KVM_GET_DIRTY_LOG_PROTECT` mode). Read-then-write-protect for re-tracking. Cross-ref `memslots.md` for per-slot bitmap mgmt.
- **Per-vCPU dirty ring** (modern mode, `KVM_CAP_DIRTY_LOG_RING_ACQ_REL`): per-vCPU lockless ringbuffer of dirty GFN entries with explicit acquire-release semantics; producer is vCPU thread updating SPTE D-bit, consumer is userspace migration thread; eliminates cross-CPU bitmap-cacheline contention that plagues per-slot bitmap mode for large guests.

This Tier-3 covers `virt/kvm/dirty_ring.c` (~270 lines) — the per-vCPU ring implementation + per-vCPU mmap'd shared page handling.

### Acceptance Criteria

- [ ] AC-1: Live migration test with KVM_CAP_DIRTY_LOG_RING_ACQ_REL enabled: 4GB guest memory + sustained vCPU writes + qemu live-migrate; migration completes; destination guest matches source bit-by-bit.
- [ ] AC-2: Ring acq-rel ordering test: producer thread + consumer thread; verify with formal model checker (manual TLA+ run) that no torn entries observed.
- [ ] AC-3: Soft-full handling: configure tiny ring (1024 entries) + write-heavy guest; verify KVM_EXIT_DIRTY_RING_FULL fires; userspace consume + vCPU resumes.
- [ ] AC-4: Reset-then-write-protect cycle: write-protect gfn after RESET; subsequent vCPU write triggers re-fault + new dirty entry.
- [ ] AC-5: Per-VM dirty-ring + per-slot-bitmap mutual exclusion: enabling both rejected with -EINVAL.
- [ ] AC-6: kselftest `tools/testing/selftests/kvm/dirty_log_test` passes for both bitmap + ring modes.

### Architecture

`DirtyRing` lives in `kernel::kvm::DirtyRing`:

```
struct DirtyRing {
  dirty_index: AtomicU32,             // producer-side tail (next write position)
  reset_index: u32,                    // consumer-side head (next read position)
  size: u32,                            // total ring size in entries
  soft_limit: u32,                      // threshold for soft-full vmexit
  dirty_gfns: NonNull<[KvmDirtyGfn]>,   // mmap'd shared page array
  index: u32,                           // vCPU index
}
```

Per-vCPU mmap layout:
- vcpu->run page at offset 0 (cross-ref `kvm-core.md`).
- Coalesced MMIO ring at offset KVM_COALESCED_MMIO_PAGE_OFFSET * PAGE_SIZE.
- Dirty ring at offset KVM_DIRTY_LOG_PAGE_OFFSET * PAGE_SIZE; size = ring_count * 16 bytes.

Producer path (vCPU thread on guest write fault):
1. Per-arch SPTE handler (e.g., `tdp_mmu_set_spte` from `x86-mmu-tdp.md`): when D-bit transitions 0→1, mark dirty.
2. `kvm_dirty_ring_push(vcpu->dirty_ring, slot_id, offset)`:
   - `tail = ring.dirty_index.load(Relaxed)`.
   - `entry = &ring.dirty_gfns[tail % ring.size]`.
   - Write `entry->slot = slot_id`, `entry->offset = offset`.
   - `entry->flags = KVM_DIRTY_GFN_F_DIRTY` (with release semantics: `smp_store_release`).
   - `ring.dirty_index.store(tail + 1, Release)`.
3. `kvm_dirty_ring_softlimit_reached(ring)`: if `tail - reset_index >= ring.soft_limit`, set KVM_REQ_DIRTY_RING_SOFT_FULL on vcpu → causes next vmexit to return KVM_EXIT_DIRTY_RING_FULL to userspace.

Consumer path (userspace migration thread, polling mmap'd ring):
1. Read `entry = &ring.dirty_gfns[head % ring.size]`.
2. `entry.flags` acquired via smp_load_acquire (compiler-generated for shared mmap).
3. If `entry.flags == DIRTY`:
   - Read `entry.slot + entry.offset` → derive gfn → copy associated 4K page to migration destination.
   - Write `entry.flags = RESET` (release).
4. Advance head.

Reset path (userspace calls `KVM_RESET_DIRTY_RINGS` on VM fd after consuming):
1. Walk all vCPUs' rings.
2. For each ring entry from `reset_index` forward:
   - If `flags == RESET`:
     - Per-arch write-protect gfn: `kvm_arch_mmu_enable_log_dirty_pt_masked(kvm, slot, offset, 1)` → forces SPTE write-protected so next vCPU write re-faults + re-marks dirty.
     - Set entry's `flags = INVALID`.
     - Advance `ring.reset_index += 1`.

Hard-full handling: when push tries to wrap onto entry with `flags == DIRTY` (consumer hasn't consumed): vCPU thread blocks via:
1. Set `vcpu->run.exit_reason = KVM_EXIT_DIRTY_RING_FULL`.
2. Return from `__kvm_arch_vcpu_run` to userspace.
3. Userspace consumes entries + calls KVM_RESET_DIRTY_RINGS.
4. Userspace re-invokes KVM_RUN; vcpu retries push, now successful.

Per-VM mode selection: `KVM_ENABLE_CAP(KVM_CAP_DIRTY_LOG_RING_ACQ_REL, ring_size)` enables ring mode; subsequent `KVM_SET_USER_MEMORY_REGION` with `KVM_MEM_LOG_DIRTY_PAGES` flag rejected.

### Out of Scope

- Per-arch SPTE D-bit handling (covered in `x86-mmu-tdp.md` Tier-3 + ARM64 KVM future Tier-3)
- Per-slot bitmap mode details (covered in `memslots.md` Tier-3)
- KVM_RUN exit reason dispatch (covered in `kvm-core.md` Tier-3)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_dirty_ring` | per-vCPU dirty-ring state | `kernel::kvm::DirtyRing` |
| `struct kvm_dirty_gfn` | per-entry GFN+slot+flags | `kernel::kvm::DirtyGfn` |
| `kvm_dirty_ring_alloc(ring, count)` / `_free(ring)` | per-vCPU ring alloc/free | `DirtyRing::alloc` / `_free` |
| `kvm_dirty_ring_get_page(ring, offset)` | per-vCPU page lookup for mmap | `DirtyRing::get_page` |
| `kvm_dirty_ring_push(ring, slot, offset)` | producer: append entry to ring | `DirtyRing::push` |
| `kvm_dirty_ring_check_request(vcpu)` | check if ring full → KVM_REQ_DIRTY_RING_SOFT_FULL set | `Vcpu::check_dirty_ring_request` |
| `kvm_dirty_ring_reset(kvm, ring, &count)` | consumer-side reset (after userspace consumed entries) | `DirtyRing::reset` |
| `kvm_cpu_dirty_log_size()` | per-arch ring entry size | `arch::kvm::dirty_log_size` |
| `kvm_dirty_ring_softlimit_reached(ring)` | check if soft-limit reached (threshold for soft-full vmexit) | `DirtyRing::softlimit_reached` |
| `kvm_dirty_ring_exit(vcpu)` | per-vCPU ring exit on KVM_EXIT_DIRTY_RING_FULL | `Vcpu::dirty_ring_exit` |

### compatibility contract

REQ-1: `KVM_CAP_DIRTY_LOG_RING_ACQ_REL` capability advertised when arch supports it (x86 + ARM64). Userspace enables via `KVM_ENABLE_CAP(KVM_CAP_DIRTY_LOG_RING_ACQ_REL, ring_size)` where ring_size is power-of-2 between 1024 and 131072 entries.

REQ-2: Per-vCPU mmap region: `mmap(vcpu_fd, size, PROT_READ|PROT_WRITE, MAP_SHARED, KVM_DIRTY_LOG_PAGE_OFFSET * PAGE_SIZE)` returns ring shared page; size = `ring_count * sizeof(struct kvm_dirty_gfn)`.

REQ-3: `struct kvm_dirty_gfn` byte-identical layout: `flags` (4 bytes; KVM_DIRTY_GFN_F_DIRTY / _RESET / _INVALID), `slot` (4 bytes), `offset` (8 bytes; in pages).

REQ-4: Producer side (vCPU thread): when SPTE D-bit transitions from 0→1 during page-table walk, push entry to per-vCPU ring (`flags = DIRTY`, `slot = slot_id`, `offset = gfn - base_gfn`). Per-CPU-LOCAL push (no inter-CPU contention since producer is always local vCPU thread).

REQ-5: Acquire-release ordering: producer increments `tail` with release-store (smp_store_release) AFTER writing entry; consumer reads `tail` with acquire-load (smp_load_acquire) BEFORE reading entry. Standard SPSC ringbuffer pattern.

REQ-6: Consumer side (userspace migration thread): poll `head < tail`; for each entry [head, tail): read flags + slot + offset; if `flags == DIRTY`: copy associated guest page to migration destination; mark entry `flags = RESET`; advance head with release-store.

REQ-7: KVM_RESET_DIRTY_RINGS ioctl on VM fd: walks all vCPUs' rings; for each entry with `flags == RESET`: write-protect corresponding gfn (force re-fault on next write to re-mark dirty); set entry `flags = INVALID`.

REQ-8: Soft-full handling: when ring depth exceeds soft-limit (typically 50% capacity), KVM_REQ_DIRTY_RING_SOFT_FULL set on vCPU; on next vmexit, vCPU returns to userspace with `KVM_EXIT_DIRTY_RING_FULL` so userspace can drain.

REQ-9: Hard-full: ring at capacity → vCPU blocks via `kvm_dirty_ring_exit` until userspace consumes some entries.

REQ-10: Per-VM dirty-ring memory accounting: ring pages count against `kvm.mm` for OOM purposes.

REQ-11: Dirty-ring + per-slot-bitmap mutually-exclusive per-VM: cannot enable both modes simultaneously (one or the other).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ring_no_uaf` | UAF | per-vCPU ring memory mmap'd; freed only after vCPU destroy + munmap completed by userspace. |
| `dirty_index_no_overflow` | OVERFLOW | `dirty_index` u32 wraps at 2^32; ring index calculation uses modulo `ring.size`; per-spec reset_index follows so no actual overflow issue. |
| `gfn_no_oob` | OOB | per-entry slot+offset validated against per-VM memslots; out-of-range entries skipped by consumer. |

### Layer 2: TLA+

`models/kvm/dirty_ring.tla` (parent-declared): proves producer-side enqueue from vCPU thread + consumer-side dequeue from userspace migration thread observe acquire-release with no torn or duplicated GFN entries; soft-full + hard-full paths converge to consistent state.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `kvm_dirty_ring_push` post: entry written + flags-DIRTY visible to consumer with release semantics; dirty_index incremented atomically | `DirtyRing::push` |
| Per-ring invariant: `reset_index ≤ dirty_index ≤ reset_index + ring.size` (ring not over-full) | producer + consumer |
| `kvm_dirty_ring_reset` invariant: per-entry flags transition RESET → INVALID; gfn write-protected post-reset | `DirtyRing::reset` |
| Per-VM dirty-ring + bitmap mutual exclusion: at most one mode enabled per-VM | `KVM_ENABLE_CAP` validation |

### Layer 4: Verus/Creusot functional

`Producer push(slot, offset) → consumer reads entry → consumer write RESET → KVM_RESET_DIRTY_RINGS → write-protect gfn → vCPU re-faults on next write` round-trip equivalence: every guest write to dirty-tracked page eventually reflected in consumer's stream; no missed dirty page; no duplicated dirty entry within a single tracking-cycle.

### hardening

(Inherits row-1 features from `virt/kvm/00-overview.md` § Hardening.)

dirty-tracking-specific reinforcement:

- **Per-vCPU ring size bounded** by `KVM_DIRTY_RING_MAX_ENTRIES` (131072); defense against userspace requesting absurdly-large ring causing memory exhaustion.
- **Per-vCPU ring size minimum** by `KVM_DIRTY_RING_MIN_ENTRIES` (1024); defense against pathological tiny ring causing constant soft-full vmexits killing guest performance.
- **Acquire-release ordering enforced** via smp_load_acquire / smp_store_release at boundary instructions; defense against compiler reordering breaking SPSC invariant.
- **Per-vCPU ring producer always local** (no cross-CPU push); defense against cross-CPU cacheline ping-pong.
- **Per-VM mutual exclusion** of ring + bitmap modes; defense against double-tracking causing inconsistent state.
- **gfn validation per-entry** by consumer side — per-arch slot lookup validates gfn before page-copy; defense against malformed entry causing OOB.
- **Soft-full threshold tuned conservatively** (50% capacity by default); defense against ring-full vmexit-storm.
- **Per-VM ring memory accounting** charged to kvm.mm; OOM-kill respects.

