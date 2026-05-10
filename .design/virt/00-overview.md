# Subsystem: virt/ — virtualization shared infrastructure

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: in-v0
upstream-paths:
  - virt/
  - virt/kvm/
  - virt/lib/
  - include/linux/kvm_host.h
  - include/linux/kvm_types.h
  - include/uapi/linux/kvm.h
-->

## Summary
Tier-2 overview for `virt/` — the cross-architecture KVM virtualization core. Owns the KVM userspace ABI (`/dev/kvm`), the per-VM and per-vCPU abstractions, the MMU notifier framework consumed by host MM, and the architecture-agnostic vCPU exit/run loop. Architecture-specific KVM lives in `arch/<arch>/kvm/` (for v0, only `arch/x86/00-overview.md` § kvm.md).

`virt/lib/` holds shared library code used across virtualization paths.

## Upstream references in scope

| Category | Upstream paths | Planned Tier-3 doc |
|---|---|---|
| KVM core (cross-arch) | `virt/kvm/` (kvm_main.c, vfio.c, eventfd.c, irqchip.c, dirty_ring.c, async_pf.c, binary_stats.c, coalesced_mmio.c) | `kvm-core.md` |
| Virt lib | `virt/lib/` | `virt-lib.md` |

## Compatibility contract

### `/dev/kvm` ioctl ABI

The KVM userspace ABI per `include/uapi/linux/kvm.h` is one of the largest ioctl interfaces in the kernel. Key categories preserved byte-identically:

- **System-fd ioctls**: `KVM_GET_API_VERSION`, `KVM_CREATE_VM`, `KVM_GET_VCPU_MMAP_SIZE`, `KVM_CHECK_EXTENSION`, `KVM_GET_SUPPORTED_CPUID` (x86), `KVM_GET_MSR_INDEX_LIST` (x86), `KVM_GET_MSR_FEATURE_INDEX_LIST`.
- **VM-fd ioctls**: `KVM_CREATE_VCPU`, `KVM_GET_DIRTY_LOG`, `KVM_SET_USER_MEMORY_REGION`, `KVM_SET_USER_MEMORY_REGION2`, `KVM_CREATE_IRQCHIP`, `KVM_IRQ_LINE`, `KVM_GET_IRQCHIP`, `KVM_SET_IRQCHIP`, `KVM_SET_GSI_ROUTING`, `KVM_IRQFD`, `KVM_IOEVENTFD`, `KVM_REGISTER_COALESCED_MMIO`, `KVM_UNREGISTER_COALESCED_MMIO`, `KVM_GET_DIRTY_LOG`, `KVM_CLEAR_DIRTY_LOG`, `KVM_RESET_DIRTY_RINGS`, `KVM_PRE_FAULT_MEMORY`, `KVM_GET_STATS_FD`.
- **vCPU-fd ioctls**: `KVM_RUN`, `KVM_GET_REGS`, `KVM_SET_REGS`, `KVM_GET_SREGS`, `KVM_SET_SREGS`, `KVM_GET_FPU`, `KVM_SET_FPU`, `KVM_GET_LAPIC`, `KVM_SET_LAPIC`, `KVM_GET_MSRS`, `KVM_SET_MSRS`, `KVM_TRANSLATE`, `KVM_INTERRUPT`, `KVM_NMI`, `KVM_GET_CPUID2`, `KVM_SET_CPUID2`, `KVM_TPR_ACCESS_REPORTING`, `KVM_X86_SET_VCPU_EVENTS`, `KVM_GET_DEBUGREGS`, `KVM_GET_XSAVE`, `KVM_GET_XCRS`, `KVM_SMI`, `KVM_HYPERV_*`, `KVM_GET_VCPU_EVENTS`, `KVM_GET_TSC_KHZ`.
- **vmexit run-state structure**: `struct kvm_run` mmap'd between userspace QEMU and kernel KVM; layout preserved.

### binary stats

`/sys/kernel/debug/kvm/<n>/*` debug files + the `KVM_GET_STATS_FD` binary stats interface preserve format.

### Tracing

KVM-specific trace events (`/sys/kernel/tracing/events/kvm/*`) preserved.

## Requirements

- REQ-1: Every `/dev/kvm` ioctl preserves number, struct layout, and observable semantics.
- REQ-2: `struct kvm_run` mmap layout is byte-identical so QEMU/kvmtool/firecracker work unmodified.
- REQ-3: Host-side dirty-page tracking (KVM_GET_DIRTY_LOG and dirty-ring) preserves bit semantics.
- REQ-4: Async page fault (APF) ABI preserved.
- REQ-5: Coalesced MMIO ring layout preserved.
- REQ-6: irqfd / ioeventfd semantics preserved.
- REQ-7: KVM stats fd binary format preserved.
- REQ-8: MMU notifier callbacks (consumed by host mm/ to invalidate guest-mapped pages on host PTE changes) preserved.
- REQ-9: All Tier-3 docs declare unsafe-block clusters, TLA+ models (the dirty-ring + irqfd are non-trivial concurrency surfaces), and Kani harnesses.
- REQ-10: x86-specific KVM lives in `arch/x86/00-overview.md` § kvm.md (v0 graduation: `kvm-unit-tests` passes per `00-overview.md` D6 of arch/x86).

## Acceptance Criteria

- [ ] AC-1: `kvm-unit-tests` passes against Rookery KVM with the same set as upstream (delegated to `arch/x86/kvm.md` Tier 3). (covers REQ-1, REQ-10)
- [ ] AC-2: QEMU launches a Linux guest VM under Rookery KVM identically to upstream. (covers REQ-2)
- [ ] AC-3: Live-migration test (KVM_GET_DIRTY_LOG-based) succeeds between an upstream host and a Rookery host. (covers REQ-3)
- [ ] AC-4: Async-PF interaction with a guest paravirtualized for APF preserves the same fault-resolution sequence. (covers REQ-4)
- [ ] AC-5: Coalesced MMIO selftest passes. (covers REQ-5)
- [ ] AC-6: irqfd + ioeventfd selftests pass. (covers REQ-6)
- [ ] AC-7: `KVM_GET_STATS_FD` produces binary-identical layout. (covers REQ-7)
- [ ] AC-8: A host mm change (mprotect, munmap) on a guest-mapped page propagates correctly via MMU notifier. (covers REQ-8)
- [ ] AC-9: `make verify` passes virt/ Kani harnesses. (covers REQ-9)

## Architecture

### Layout map

```
.design/virt/
  00-overview.md        ← this document
  kvm-core.md           ← kvm_main + ioctls + vfio + eventfd + irqchip + dirty_ring + async_pf + coalesced_mmio + binary_stats
  virt-lib.md           ← shared library code in virt/lib/
```

### Cross-references

- `arch/x86/00-overview.md` § `kvm.md` — x86-specific VMX (Intel) + SVM (AMD) implementation; consumes virt/kvm/ as the cross-arch substrate.
- `mm/00-overview.md` — MMU notifier framework integration (host PTE invalidation propagating to guest-mapped pages).
- `kernel/00-overview.md` — `kernel/events/` (perf integration with KVM); `kernel/sched/` (vCPU scheduling).
- `00-glossary.md` — `KVM`.

### Rust module organization (informative)

- `kernel::virt::kvm::core` — Rookery to author
- `kernel::virt::kvm::dirty_ring` — Rookery to author
- `kernel::virt::kvm::irqfd` — Rookery to author
- `kernel::virt::lib` — Rookery to author

### Locking and concurrency

KVM is highly concurrent:
- Per-VM `kvm->lock` (mutex)
- Per-vCPU `vcpu->mutex` for ioctl serialization
- `kvm->slots_lock` (mutex) for memory-slot updates
- `kvm->mmu_lock` (spinlock or rwlock) — page-table updates
- Dirty ring: lock-free SPSC ring per vCPU
- IRQ routing: RCU
- MMU notifier: notifier chain per VM

### Error handling

KVM ioctl errors: ENODEV, EBUSY, EINVAL, ENOMEM, EFAULT, EINTR (vCPU run interrupted by signal), ENOSYS (extension not supported).

## Verification

### Layer 1: Kani SAFETY proofs
- Memory slot lookup
- Dirty ring entry production/consumption
- vCPU run-state pointer lifetime

### Layer 2: TLA+ models
- `models/virt/dirty_ring.tla` — proves dirty-ring SPSC correctness (no entry lost; no double-recording).
- `models/virt/irqfd.tla` — proves irqfd's eventfd-trigger → vCPU-inject ordering.
- `models/virt/mmu_notifier.tla` — proves MMU notifier ordering: host PTE invalidation observable by every vCPU before next entry.

### Layer 3: Kani harnesses
- Memory slot table integrity (no overlap, address-ordered)
- Dirty ring head/tail invariants

### Layer 4: opt-in
- Cross-arch KVM core invariants are good Verus targets but slip-past-v0-feasible.

## Hardening

Placeholder per `00-overview.md` D6. Notable hardening: KVM is itself a security boundary — guests must NOT be able to read/write host memory except via well-defined ABIs. The verification stack here is unusually high-stakes.

## Open Questions

(none — KVM scope is locked per `arch/x86/00-overview.md` D6: in v0 with kvm-unit-tests as graduation gate)

## Out of Scope

- Architecture-specific KVM (owned by `arch/x86/00-overview.md` § kvm.md in v0; other arches deferred)
- KVM userspace (QEMU, kvmtool) — accepted as upstream
- Implementation code
