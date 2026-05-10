# Tier-3: virt/kvm/coalesced_mmio.c — Per-VM coalesced-MMIO ring (batch MMIO writes without per-write vmexit)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/00-overview.md
upstream-paths:
  - virt/kvm/coalesced_mmio.c
  - virt/kvm/coalesced_mmio.h
  - include/uapi/linux/kvm.h (KVM_REGISTER_COALESCED_MMIO + KVM_UNREGISTER_COALESCED_MMIO)
-->

## Summary

Coalesced MMIO is a KVM optimization that lets userspace VMM register specific MMIO regions where guest writes are buffered into a per-VM ring (mmap'd to userspace) instead of triggering per-write vmexit + KVM_EXIT_MMIO. Used by qemu for VGA framebuffer (where guest does many sequential writes that VGA driver doesn't need acked individually), some virtio-mmio doorbells (where guest write doesn't need synchronous response), legacy SVGA emulation. Saves ~5000 cycles per write by batching ~100 writes into one userspace dispatch.

Per-VM ring shared via mmap on vCPU-fd at `KVM_COALESCED_MMIO_PAGE_OFFSET * PAGE_SIZE`; producer (in-kernel KVM IO bus dispatch) appends `kvm_coalesced_mmio` entries; consumer (userspace VMM) drains.

This Tier-3 covers `virt/kvm/coalesced_mmio.c` (~190 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_coalesced_mmio_dev` | per-region pseudo-device on KVM IO bus | `kernel::kvm::CoalescedMmioDev` |
| `struct kvm_coalesced_mmio_ring` | shared ring header + entries | `kernel::kvm::CoalescedMmioRing` |
| `struct kvm_coalesced_mmio` | per-entry record (phys_addr + len + data + pio) | `kernel::kvm::CoalescedMmioEntry` |
| `kvm_coalesced_mmio_init(kvm)` / `_free(kvm)` | per-VM init/free | `Vm::coalesced_mmio_init` / `_free` |
| `kvm_vm_ioctl_register_coalesced_mmio(kvm, &zone)` | KVM_REGISTER_COALESCED_MMIO ioctl handler | `Vm::register_coalesced_mmio` |
| `kvm_vm_ioctl_unregister_coalesced_mmio(kvm, &zone)` | KVM_UNREGISTER_COALESCED_MMIO ioctl handler | `Vm::unregister_coalesced_mmio` |
| `coalesced_mmio_in_range(dev, addr, len)` | per-dev range-match check | `CoalescedMmioDev::in_range` |
| `coalesced_mmio_write(vcpu, dev, addr, len, val)` | per-dev write handler (called from kvm_io_bus_write) | `CoalescedMmioDev::write` |
| `coalesced_mmio_destructor(dev)` | per-dev release | `CoalescedMmioDev::destroy` |

## Compatibility contract

REQ-1: `KVM_REGISTER_COALESCED_MMIO(struct kvm_coalesced_mmio_zone)` UAPI byte-identical:
- `addr` (gpa), `size` (bytes), `pio` (0=MMIO / 1=PIO).
- Returns 0 on success, -ENOMEM on full ring-list, -EINVAL on overlap with existing zone.

REQ-2: Per-VM ring layout: shared 4KB page mmap'd via `mmap(vcpu_fd, PAGE_SIZE, PROT_READ|PROT_WRITE, MAP_SHARED, KVM_COALESCED_MMIO_PAGE_OFFSET * PAGE_SIZE)`.
- Header (start of page): `first` (consumer head, written by userspace), `last` (producer tail, written by kernel).
- Entry array (sized to fill page): `kvm_coalesced_mmio` records, 32 bytes each → ~127 entries per ring.

REQ-3: Per-entry layout: `phys_addr` (8 bytes; gpa) + `len` (4 bytes; 1/2/4/8) + `data[8]` (8 bytes; little-endian aligned to len) + `pio` (4 bytes; 0/1) + `pad[3]` (12 bytes padding to 32 bytes).

REQ-4: Per-region registration: `kvm_io_bus_register_dev(kvm, KVM_MMIO_BUS, addr, size, &coalesced_dev)` (or KVM_PIO_BUS for pio); per-region pseudo-device installed on KVM IO bus (cross-ref `kvm-eventfd.md` § ioeventfd).

REQ-5: Per-write dispatch: when guest does MMIO/PIO write at registered range, KVM IO bus walks devices; coalesced_mmio_dev's `write` callback:
1. Append entry to per-VM ring tail (under per-VM ring_lock).
2. Increment `last` pointer (modulo ring size).
3. Return 0 (handled — no vmexit to userspace).

REQ-6: Ring full handling: when next-tail would equal `first` (consumer head), `coalesced_mmio_write` returns 1 indicating "fall through to userspace KVM_EXIT_MMIO" — vCPU vmexits, userspace processes pending entries + the current write.

REQ-7: Userspace consumption: on every vmexit (or periodically), userspace VMM reads ring header `last`; for each entry from `first` to `last`, dispatches to per-region driver model (e.g., qemu cirrus VGA driver); advance `first` after consuming each entry.

REQ-8: Per-VM ring shared across all vCPUs — single producer is the per-vCPU IO bus dispatch; per-VM ring_lock serializes producers.

REQ-9: KVM_UNREGISTER_COALESCED_MMIO removes per-region pseudo-device; subsequent guest writes at that addr fall through to KVM_EXIT_MMIO.

REQ-10: VM destroy releases all coalesced_mmio_dev's via `coalesced_mmio_destructor`.

## Acceptance Criteria

- [ ] AC-1: qemu cirrus VGA test: guest VGA driver writes pixel row → kernel coalesces ~100 writes into ring → userspace drains in one batch; per-frame vmexit count near 1 instead of ~10000.
- [ ] AC-2: Ring full test: rapid writes overflowing 127-entry ring → vmexit on overflow → userspace drains + resumes.
- [ ] AC-3: KVM_REGISTER_COALESCED_MMIO + UNREGISTER round-trip + verify subsequent writes return to userspace.
- [ ] AC-4: PIO variant (pio=1): guest PIO write at registered port → coalesced into ring same as MMIO.
- [ ] AC-5: kselftest `tools/testing/selftests/kvm/coalesced_io_test` passes.
- [ ] AC-6: Per-VM ring overflow stress: sustained writes with userspace not draining → per-write fallback to vmexit; no UAF; ring resyncs cleanly.

## Architecture

`CoalescedMmioDev` lives in `kernel::kvm::CoalescedMmioDev`:

```
struct CoalescedMmioDev {
  dev: KvmIoDevice,                    // base IO bus device
  kvm: Arc<Vm>,
  zone: KvmCoalescedMmioZone,           // (addr, size, pio)
}

struct CoalescedMmioRing {
  first: u32,                           // consumer head (userspace writes)
  last: u32,                            // producer tail (kernel writes)
  ring: VarLenArray<KvmCoalescedMmio>,  // entry array
}
```

Per-VM init `Vm::coalesced_mmio_init(kvm)`:
1. Allocate per-VM ring page via `alloc_page(GFP_KERNEL)`.
2. `kvm.coalesced_mmio_ring = page_address(page)`.
3. Initialize header: `first = 0, last = 0`.
4. Allocate `kvm.coalesced_zones` list (linked list of registered zones).

Per-region register `Vm::register_coalesced_mmio(zone)`:
1. Validate zone.size > 0 + zone.addr non-zero.
2. Allocate `CoalescedMmioDev`.
3. Initialize `dev.zone = zone`.
4. `dev.dev.ops = &coalesced_mmio_ops` (write callback).
5. `kvm_io_bus_register_dev(kvm, zone.pio ? KVM_PIO_BUS : KVM_MMIO_BUS, zone.addr, zone.size, &dev.dev)`.
6. Add to `kvm.coalesced_zones` list.

Per-write dispatch `coalesced_mmio_write(vcpu, dev, addr, len, val)`:
1. Take `kvm.ring_lock`.
2. Read ring `first` (acquire-load).
3. Compute `next_last = (last + 1) % ring_size`.
4. If `next_last == first`: ring full → drop ring_lock + return 1 (fall through to userspace).
5. Else:
   - Build entry: `entry = ring.ring[last]; entry.phys_addr = addr; entry.len = len; memcpy(entry.data, val, len); entry.pio = dev.zone.pio`.
   - `ring.last = next_last` (release-store).
6. Drop ring_lock.
7. Return 0 (handled).

Userspace consumer (qemu KVM_RUN return):
1. After each KVM_RUN, before re-enter: read `ring->last` (acquire-load).
2. For each entry `[first, last)`:
   - Dispatch to per-driver-model device based on `entry.phys_addr` (e.g., cirrus VGA driver writes pixel data).
   - Advance `first = (first + 1) % ring_size` (release-store).

Per-region unregister `Vm::unregister_coalesced_mmio(zone)`:
1. Find matching CoalescedMmioDev in coalesced_zones list.
2. `kvm_io_bus_unregister_dev(kvm, bus_idx, &dev.dev)`.
3. Remove from list.
4. Drop refcount → release.

VM destroy `Vm::coalesced_mmio_free(kvm)`:
1. Walk coalesced_zones list; for each: unregister + free.
2. Free coalesced_mmio_ring page.

Per-VM ring_lock vs single-vCPU producer: each vCPU's IO-bus dispatch acquires ring_lock; concurrent writes from N vCPUs serialize via spinlock.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dev_no_uaf` | UAF | `CoalescedMmioDev` outlives all in-flight per-write dispatches; KVM IO bus unregister waits for per-write callback to complete. |
| `ring_idx_no_oob` | OOB | `last` + `first` indices wrap modulo ring_size; bounds-checked at every access. |
| `entry_data_size_correct` | INVARIANT | per-entry `data[8]` array filled per `len` (1/2/4/8 bytes); higher bytes zeroed. |
| `producer_serialized` | ATOMICITY | per-VM ring_lock serializes concurrent producer writes; no two vCPUs claim same ring slot. |

### Layer 2: TLA+

(No specific TLA+ — ring SPSC semantics covered by parent's eventfd/dirty-ring TLA+ patterns.)

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vm::register_coalesced_mmio` post: `kvm.coalesced_zones` contains new zone; KVM IO bus has new pseudo-device | `Vm::register_coalesced_mmio` |
| `coalesced_mmio_write` post: returns 0 + ring entry appended OR returns 1 + ring full (fall-through) | `coalesced_mmio_write` |
| Per-ring invariant: `(last - first) mod ring_size <= ring_size - 1` (never overflow consumer) | producer + consumer |
| Per-entry invariant: `entry.len ∈ {1, 2, 4, 8}`; `entry.phys_addr` in registered zone range | per-entry build |

### Layer 4: Verus/Creusot functional

`Producer write(addr, len, val) → ring entry appended → consumer drain → userspace dispatch` round-trip equivalence: every coalesced write eventually visible to userspace consumer in producer order; no missed write modulo ring-overflow path.

## Hardening

(Inherits row-1 features from `virt/kvm/00-overview.md` § Hardening.)

coalesced-mmio specific reinforcement:

- **Per-VM ring page CAP_SYS_ADMIN-mediated** at mmap — defense against unprivileged process getting access to in-kernel-emitted MMIO data stream.
- **Per-region registration validated against existing zones** — overlap rejected (-EINVAL); defense against double-zone-confusion.
- **Per-VM ring_lock acquired in canonical order** — defense against deadlock with KVM IO bus mutex.
- **Per-entry data[8] zeroed beyond `len` bytes** — defense against information leak from prior entry's stale data.
- **Ring full → fall-through to userspace** — never silently drops writes; defense against guest-state-loss from coalescing.
- **Per-region unregister synchronized with in-flight per-write** via KVM IO bus RCU semantics; defense against UAF.
- **Per-VM coalesced-zones count cap** — bounded (default 64 zones per VM); defense against per-VM zone-flood.
- **Per-VM ring page bounded to single 4 KB** — fixed size; defense against userspace-controlled ring size attacks.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM IO bus dispatch (covered in `kvm-core.md` Tier-3)
- KVM_IRQFD + KVM_IOEVENTFD (covered in `kvm-eventfd.md` Tier-3)
- vCPU run loop (covered in `kvm-core.md` Tier-3)
- 32-bit-only paths
- Implementation code
