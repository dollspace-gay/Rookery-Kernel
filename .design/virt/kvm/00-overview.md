# Tier-2: virt/kvm — KVM virtualization (core + x86 + vCPU + MMU + irqchip + SEV/TDX)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - virt/kvm/
  - arch/x86/kvm/
  - include/linux/kvm_host.h
  - include/uapi/linux/kvm.h
-->

## Summary

Tier-2 overview for the Kernel-based Virtual Machine. KVM exposes hardware virtualization (Intel VMX / AMD SVM) to userspace VMMs (qemu-system-x86_64 / cloud-hypervisor / firecracker / crosvm) through the `/dev/kvm` ioctl interface. Three concentric scopes:

- **Generic core** (`virt/kvm/`): arch-independent vCPU dispatch, memory-slots, dirty-ring, async pf, eventfd (irqfd / ioeventfd), guest_memfd, coalesced MMIO, vfio bridge, pfn-cache, binary-stats.
- **x86 arch** (`arch/x86/kvm/`): CPUID virtualization, instruction emulation, in-kernel APIC / IOAPIC / PIC / PIT, MTRR / SMM / Hyper-V / Xen hyperv-emulation, MMU (TDP + shadow), VMX vendor backend, SVM vendor backend, PMU, x86 ioctl glue.
- **Vendor/feature shards**: `vmx/` (Intel: VMX root, nested, SGX, TDX), `svm/` (AMD: AVIC, nested, SEV / SEV-ES / SEV-SNP), `mmu/` (TDP MMU + shadow + page-track + sptes).

Heavily cross-referenced by `arch/x86/cpu-init.md` (VMXE/SMXE bits + SVM MSRs), `arch/x86/idt.md` (#VE / #VC handlers), `arch/x86/mm-pgtable.md` (EPT/NPT walk semantics mirror x86 paging), `mm/00-overview.md` (MMU notifiers — KVM uses the only in-tree consumer chain), `kernel/sched/00-overview.md` (preempt-notifier-driven vCPU thread context-switch), `fs/00-overview.md` (`anon_inode` for vCPU/vm fds; `guest_memfd` extends shmem semantics), `drivers/vfio/` (KVM↔VFIO IRQ bypass + IOMMU group binding), `kernel/cgroup/cpuset.md` (vCPU thread cpuset confinement), `kernel/perf/` (PMU passthrough), `net/00-overview.md` (vhost-net consumer of irqfd/ioeventfd).

## Scope

This Tier-2 governs:
- `/home/doll/linux-src/virt/kvm/` (~13 source files)
- `/home/doll/linux-src/arch/x86/kvm/` (top level ~25 files)
- `/home/doll/linux-src/arch/x86/kvm/mmu/` (TDP MMU + shadow MMU + page-track + sptes)
- `/home/doll/linux-src/arch/x86/kvm/vmx/` (VMX root + nested-vmx + posted-intr + SGX + TDX)
- `/home/doll/linux-src/arch/x86/kvm/svm/` (SVM root + nested-svm + AVIC + SEV/SEV-ES/SEV-SNP)
- `include/linux/kvm_host.h` (in-kernel API)
- `include/uapi/linux/kvm.h` (UAPI struct + ioctl numbers — frozen wire format)

## Compatibility contract — outline

### `/dev/kvm` ioctl wire format

`KVM_*` ioctl numbers + struct layouts byte-identical so qemu / cloud-hypervisor / firecracker binary-compatible without recompile. Key categories:

**System ioctls** (on `/dev/kvm` fd):
- `KVM_GET_API_VERSION` → 12 (frozen)
- `KVM_CREATE_VM` → vm fd
- `KVM_GET_MSR_INDEX_LIST`, `KVM_GET_MSR_FEATURE_INDEX_LIST`
- `KVM_GET_SUPPORTED_CPUID`, `KVM_GET_EMULATED_CPUID`
- `KVM_CHECK_EXTENSION` → introspection of optional features (KVM_CAP_*)
- `KVM_GET_VCPU_MMAP_SIZE`

**VM ioctls** (on vm fd):
- `KVM_CREATE_VCPU` → vcpu fd
- `KVM_GET_DIRTY_LOG` / `KVM_CLEAR_DIRTY_LOG`
- `KVM_SET_USER_MEMORY_REGION` / `KVM_SET_USER_MEMORY_REGION2` (memory slots)
- `KVM_CREATE_GUEST_MEMFD` (CONFIG_KVM_PRIVATE_MEM)
- `KVM_SET_TSS_ADDR`, `KVM_SET_IDENTITY_MAP_ADDR` (legacy real-mode 32-bit guest support)
- `KVM_CREATE_IRQCHIP` → in-kernel APIC/IOAPIC/PIC; `KVM_IRQ_LINE`, `KVM_GET_IRQCHIP` / `KVM_SET_IRQCHIP`
- `KVM_CREATE_PIT2`, `KVM_GET_PIT2` / `KVM_SET_PIT2`
- `KVM_REGISTER_COALESCED_MMIO` / `KVM_UNREGISTER_COALESCED_MMIO`
- `KVM_IRQFD` (eventfd → irq), `KVM_IOEVENTFD` (mmio/pio → eventfd)
- `KVM_SET_GSI_ROUTING`
- `KVM_SIGNAL_MSI`
- `KVM_GET_CLOCK` / `KVM_SET_CLOCK`
- `KVM_RESET_DIRTY_RINGS`
- `KVM_X86_SET_MSR_FILTER`
- `KVM_MEMORY_ENCRYPT_OP` (SEV / SEV-ES / SEV-SNP / TDX feature multiplexer)

**vCPU ioctls** (on vcpu fd):
- `KVM_RUN` (the central one — userspace blocks here while CPU is in non-root mode; returns at every VM-exit)
- `KVM_GET_REGS` / `KVM_SET_REGS`, `KVM_GET_SREGS` / `KVM_SET_SREGS`, `KVM_GET_FPU` / `KVM_SET_FPU`, `KVM_GET_XSAVE` / `KVM_SET_XSAVE`, `KVM_GET_XCRS` / `KVM_SET_XCRS`
- `KVM_GET_MSRS` / `KVM_SET_MSRS`
- `KVM_GET_CPUID2` / `KVM_SET_CPUID2`
- `KVM_GET_LAPIC` / `KVM_SET_LAPIC`
- `KVM_INTERRUPT`, `KVM_NMI`, `KVM_SMI`
- `KVM_GET_VCPU_EVENTS` / `KVM_SET_VCPU_EVENTS` (pending exception/interrupt/NMI/SMI/SIPI state)
- `KVM_GET_MP_STATE` / `KVM_SET_MP_STATE`
- `KVM_TRANSLATE` (gva → gpa via guest CR3)
- `KVM_GET_TSC_KHZ` / `KVM_SET_TSC_KHZ`
- `KVM_KVMCLOCK_CTRL`
- `KVM_ENABLE_CAP`
- `KVM_X86_SET_MCE`, `KVM_X86_GET_MCE_CAP_SUPPORTED`
- `KVM_GET_NESTED_STATE` / `KVM_SET_NESTED_STATE`

**Wire format** for `struct kvm_run` (mmap'd on vcpu fd) MUST match upstream byte-for-byte — every reason field, every union member, every padding byte. qemu reads this struct directly after every `KVM_RUN`.

### `KVM_CAP_*` capability bits

Userspace probes optional features via `KVM_CHECK_EXTENSION(KVM_CAP_X)`. v0 set MUST include at minimum the Linux 7.1 stable subset (~250 capabilities). Specific ones with strong qemu-side dependency: `KVM_CAP_USER_MEMORY`, `KVM_CAP_USER_MEMORY2`, `KVM_CAP_DIRTY_LOG_RING`, `KVM_CAP_MEMORY_FAULT_INFO`, `KVM_CAP_SET_GUEST_DEBUG`, `KVM_CAP_X86_SMM`, `KVM_CAP_SYNC_REGS`, `KVM_CAP_HLT`, `KVM_CAP_HYPERV*`, `KVM_CAP_XEN_HVM`, `KVM_CAP_X86_APIC_BUS_CYCLES_NS`, `KVM_CAP_SEV_*`, `KVM_CAP_VM_TYPES` (TDX/SEV-SNP).

### KVM_EXIT_* reason codes (in `struct kvm_run.exit_reason`)

Semantic meaning byte-identical:
- `KVM_EXIT_UNKNOWN`, `KVM_EXIT_EXCEPTION`, `KVM_EXIT_IO`, `KVM_EXIT_HYPERCALL`, `KVM_EXIT_DEBUG`, `KVM_EXIT_HLT`, `KVM_EXIT_MMIO`, `KVM_EXIT_IRQ_WINDOW_OPEN`, `KVM_EXIT_SHUTDOWN`, `KVM_EXIT_FAIL_ENTRY`, `KVM_EXIT_INTR`, `KVM_EXIT_SET_TPR`, `KVM_EXIT_TPR_ACCESS`, `KVM_EXIT_NMI`, `KVM_EXIT_INTERNAL_ERROR`, `KVM_EXIT_OSI`, `KVM_EXIT_PAPR_HCALL`, `KVM_EXIT_S390_*` (no-op stubs on x86), `KVM_EXIT_WATCHDOG`, `KVM_EXIT_EPR`, `KVM_EXIT_SYSTEM_EVENT`, `KVM_EXIT_IOAPIC_EOI`, `KVM_EXIT_HYPERV`, `KVM_EXIT_ARM_NISV`, `KVM_EXIT_X86_RDMSR` / `KVM_EXIT_X86_WRMSR` (filter trap), `KVM_EXIT_DIRTY_RING_FULL`, `KVM_EXIT_AP_RESET_HOLD`, `KVM_EXIT_X86_BUS_LOCK`, `KVM_EXIT_XEN`, `KVM_EXIT_RISCV_SBI` (no-op stub), `KVM_EXIT_NOTIFY`, `KVM_EXIT_LOONGARCH_IOCSR` (no-op), `KVM_EXIT_MEMORY_FAULT`, `KVM_EXIT_TDX`.

### Per-vCPU `kvm_run` shared page

Mmap'd on vcpu fd at offset 0; size returned by `KVM_GET_VCPU_MMAP_SIZE`. Userspace reads `request_interrupt_window` / `immediate_exit` / `cr8` / `apic_base` and writes responses to MMIO/IO union after handling an exit. **Layout byte-identical**.

### Coalesced MMIO ring

On vm fd via `KVM_REGISTER_COALESCED_MMIO`. Mmap'd page contains a ring of deferred MMIO writes (typical use: VGA framebuffer batching by qemu). Format byte-identical.

### Dirty-page tracking

Two modes:
1. Per-slot bitmap retrieved by `KVM_GET_DIRTY_LOG` (legacy).
2. Per-vCPU dirty-ring (`KVM_CAP_DIRTY_LOG_RING_ACQ_REL`), mmap'd on vcpu fd at offset `KVM_DIRTY_LOG_PAGE_OFFSET * PAGE_SIZE`. Acq-rel ordering required; ring enqueued during page-table-walk in vCPU thread, dequeued by userspace migration thread. Format byte-identical.

### irqfd / ioeventfd integration

Userspace creates `eventfd(2)`, hands it to KVM via `KVM_IRQFD` (input: eventfd → guest IRQ line) or `KVM_IOEVENTFD` (output: guest MMIO/IO write → eventfd). Used by **vhost-net** + **vhost-vsock** + **vhost-user** to bypass userspace VMM round-trip. Lifetime + EOI semantics identical to upstream.

### `guest_memfd` (CONFIG_KVM_PRIVATE_MEM)

`KVM_CREATE_GUEST_MEMFD` returns an fd backing private guest memory not mappable into host userspace. Used by SEV-SNP / TDX confidential guests. Behaves like an inode-anon fd (cross-ref `fs/00-overview.md` anon_inode); ftruncate-ext/punch-hole semantics + KVM-side fault-handling identical to upstream.

### `/sys/module/kvm/`, `/sys/module/kvm_intel/`, `/sys/module/kvm_amd/` parameters

All module params byte-identical:
- `kvm.halt_poll_ns`, `kvm.halt_poll_ns_grow`, `kvm.halt_poll_ns_shrink`, `kvm.eager_page_split`
- `kvm.nx_huge_pages`, `kvm.nx_huge_pages_recovery_ratio`, `kvm.nx_huge_pages_recovery_period_ms`
- `kvm.flush_on_reuse`, `kvm.tdp_mmu`, `kvm.allow_write_only_address`
- `kvm_intel.{ept, vpid, flexpriority, vmm_exclusive, fasteoi, vnmi, ple_gap, ple_window, nested, unrestricted_guest, emulate_invalid_guest_state, enable_shadow_vmcs, enable_evmcs_version, dump_invalid_vmcs, sgx, allow_smaller_maxphyaddr, pml, error_on_inconsistent_vmcs_config, nested_early_check, preemption_timer, posted_intr_wakeup_count}`
- `kvm_amd.{npt, nested, avic, sev, sev_es, sev_snp, lbrv, tsc_scaling, pause_filter_thresh, pause_filter_count, pause_filter_count_grow, pause_filter_count_shrink, pause_filter_count_max, dump_invalid_vmcb}`

### `/proc/cpuinfo` flags

`vmx` (Intel) / `svm` (AMD) feature bits MUST be advertised when KVM module probes successfully. Cross-ref `arch/x86/cpu-init.md` REQ-X (CPUID feature mirroring).

### Tracepoints

`/sys/kernel/tracing/events/kvm/` byte-identical event names + format (used by `perf kvm stat`):
- `kvm_entry`, `kvm_exit`, `kvm_inj_virq`, `kvm_inj_exception`, `kvm_apic`, `kvm_apic_ipi`, `kvm_apic_accept_irq`, `kvm_msr`, `kvm_cr`, `kvm_pio`, `kvm_emulate_insn`, `kvm_mmio`, `kvm_fast_mmio`, `kvm_eoi`, `kvm_pi_irte_update`, `kvm_pml_full`, `kvm_track_creq`, `kvm_avic_*`, `kvm_amd_*`, `kvm_vmgexit_*` (SEV-ES) etc.

### debugfs

`/sys/kernel/debug/kvm/` byte-identical:
- Per-VM dirs (named by vm-fd inode): pf_fixed, pf_guest, tlb_flush, invlpg, exits, mmio_exits, signal_exits, irq_window_exits, halt_exits, halt_attempted_poll, halt_successful_poll, halt_poll_invalid, halt_wakeup, hypercalls, irq_injections, etc.
- `vm_stat` and `vcpu_stat` aggregate dumps.

### MMU notifier integration

KVM is the canonical (and ~only) in-tree consumer of MMU notifiers. Host page-migration / KSM / NUMA-balancing / mprotect / unmap fires the notifier; KVM tears down the matching SPTEs. The host↔guest pgtable-coherence invariant. Cross-ref `mm/00-overview.md`.

### vCPU pinning + cpuset interaction

Userspace VMM pins each vCPU thread with `pthread_setaffinity_np`; cpuset cgroup confinement applies. Cross-ref `kernel/cgroup/cpuset.md`. KVM_RUN halt-polling honors `nohz_full` cpu masks.

### KVM ↔ VFIO IRQ-bypass

`virt/kvm/vfio.c` group registration + `KVM_DEV_TYPE_VFIO`. Used for posted-interrupt direct-IRQ delivery from passed-through PCIe devices. Cross-ref future `drivers/vfio/00-overview.md`.

## Tier-3 docs governed by this Tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `virt/kvm/kvm-core.md` | `kvm_main.c`: vm/vcpu fd lifecycle, ioctl dispatch, vCPU thread loop |
| `virt/kvm/memslots.md` | per-VM memory slots: `kvm_set_memory_region`, gfn→hva translation, mmu_notifier hookup |
| `virt/kvm/dirty-tracking.md` | `dirty_ring.c` + per-slot bitmap; acq-rel ring semantics |
| `virt/kvm/eventfd.md` | irqfd + ioeventfd implementation (`virt/kvm/eventfd.c`) |
| `virt/kvm/coalesced-mmio.md` | `virt/kvm/coalesced_mmio.c` ring |
| `virt/kvm/async-pf.md` | async page fault: host-side major-fault notification (`async_pf.c`) |
| `virt/kvm/guest-memfd.md` | private guest memory inode (`guest_memfd.c`) |
| `virt/kvm/binary-stats.md` | binary-stats fd interface (`binary_stats.c`) |
| `virt/kvm/pfncache.md` | gpa→pfn cache for hot paths (`pfncache.c`) |
| `virt/kvm/vfio-bridge.md` | KVM↔VFIO IRQ bypass (`vfio.c`) |
| `virt/kvm/x86-core.md` | `arch/x86/kvm/x86.c`: x86-arch ioctl handlers, vCPU init/destroy, run loop entry |
| `virt/kvm/x86-cpuid.md` | `cpuid.c`: CPUID virtualization tables + KVM_GET_SUPPORTED_CPUID |
| `virt/kvm/x86-emulate.md` | `emulate.c` + `kvm_emulate.h`: instruction emulator (real-mode + 32-bit + invalid-guest-state recovery) |
| `virt/kvm/x86-lapic.md` | `lapic.c`: in-kernel local APIC, x2APIC, posted-interrupts wiring |
| `virt/kvm/x86-ioapic.md` | `ioapic.c` + `i8259.c` + `i8254.c`: in-kernel IOAPIC, PIC, PIT |
| `virt/kvm/x86-mmu-tdp.md` | `mmu/tdp_mmu.c` + `mmu/tdp_iter.c`: TDP MMU (default for EPT/NPT) |
| `virt/kvm/x86-mmu-shadow.md` | `mmu/mmu.c` + `mmu/paging_tmpl.h` + `mmu/spte.c`: shadow MMU (legacy / nested) |
| `virt/kvm/x86-mmu-notifier.md` | host MMU notifier handlers (`mmu/page_track.c`) |
| `virt/kvm/x86-pmu.md` | `pmu.c` + `vmx/pmu_intel.c` + `svm/pmu.c`: vPMU passthrough/emulation |
| `virt/kvm/x86-smm.md` | `smm.c`: System Management Mode emulation |
| `virt/kvm/x86-mtrr.md` | `mtrr.c`: MTRR virtualization |
| `virt/kvm/x86-hyperv.md` | `hyperv.c`: Hyper-V hypercall surface for Windows guests + nested Hyper-V |
| `virt/kvm/x86-xen.md` | `xen.c`: Xen PV hypercall surface |
| `virt/kvm/vmx-core.md` | `vmx/vmx.c` + `vmx/main.c`: VMX root + VMCS lifecycle + vmenter/vmexit handlers |
| `virt/kvm/vmx-nested.md` | `vmx/nested.c`: nested VMX (L1 hypervisor in L2 guest) |
| `virt/kvm/vmx-posted-intr.md` | `vmx/posted_intr.c`: posted-interrupt descriptors + IPI shortcut |
| `virt/kvm/vmx-sgx.md` | `vmx/sgx.c`: SGX virtualization |
| `virt/kvm/vmx-tdx.md` | `vmx/tdx.c` + `vmx/tdx_arch.h`: Intel TDX confidential guests |
| `virt/kvm/svm-core.md` | `svm/svm.c`: SVM root + VMCB lifecycle + vmrun/vmexit |
| `virt/kvm/svm-nested.md` | `svm/nested.c`: nested SVM |
| `virt/kvm/svm-avic.md` | `svm/avic.c`: AMD AVIC posted-interrupt-equivalent |
| `virt/kvm/svm-sev.md` | `svm/sev.c`: SEV / SEV-ES / SEV-SNP confidential guests |

## Compatibility outline (top-level)

- REQ-O1: All `KVM_*` ioctl numbers + struct layouts byte-identical to upstream UAPI (qemu / cloud-hypervisor / firecracker run unmodified).
- REQ-O2: `struct kvm_run` mmap'd shared page byte-identical layout.
- REQ-O3: `KVM_CAP_*` capability set: at minimum the v7.1 stable set for x86_64.
- REQ-O4: `KVM_EXIT_*` reason codes byte-identical.
- REQ-O5: `/dev/kvm` chardev present; module params under `/sys/module/kvm{,_intel,_amd}/` byte-identical.
- REQ-O6: `events/kvm/` tracepoints byte-identical names + format.
- REQ-O7: debugfs `/sys/kernel/debug/kvm/` byte-identical layout + counters.
- REQ-O8: `guest_memfd` UAPI + behavior identical (CONFIG_KVM_PRIVATE_MEM enabled).
- REQ-O9: SEV / SEV-ES / SEV-SNP / TDX feature ioctls + KVM_MEMORY_ENCRYPT_OP multiplexer wire format identical.
- REQ-O10: irqfd + ioeventfd identical (vhost-net runs unmodified).
- REQ-O11: TLA+ models declared at this Tier-2 (vCPU run-loop liveness, dirty-ring acq-rel, MMU spte invariants, irqchip injection ordering).
- REQ-O12: Verus/Creusot Layer-4 functional contracts on the KVM_RUN kernel-side state machine (no UAF on vcpu fd close while in-flight; correct VMCS/VMCB load/clear pairing).
- REQ-O13: Hardening: row-1 features applied per `00-security-principles.md`, with KVM-specific reinforcement (nested-guest-controlled VMCS field validation, SEV/TDX boundary auditing).

## Acceptance Criteria (top-level)

- [ ] AC-O1: qemu-system-x86_64 boots an unmodified Debian guest to login prompt under Rookery KVM. (covers REQ-O1, REQ-O2, REQ-O4)
- [ ] AC-O2: cloud-hypervisor + firecracker boot their reference rootfs unmodified. (covers REQ-O1, REQ-O2)
- [ ] AC-O3: `KVM_CHECK_EXTENSION` returns matching values for the v7.1 cap set. (covers REQ-O3)
- [ ] AC-O4: vhost-net throughput within 2% of upstream baseline (irqfd/ioeventfd correct). (covers REQ-O10)
- [ ] AC-O5: Dirty-ring-mode live-migration of a 4 GB guest succeeds with `qemu --machine kvm-dirty-ring-size=4096`. (covers REQ-O2, REQ-O11)
- [ ] AC-O6: `perf kvm stat live` decodes vmexit reasons correctly (tracepoint format identical). (covers REQ-O6)
- [ ] AC-O7: `cat /sys/module/kvm/parameters/halt_poll_ns` returns the same string as upstream. (covers REQ-O5)
- [ ] AC-O8: SEV-SNP guest launch via `KVM_SEV_SNP_LAUNCH_START` succeeds on AMD EPYC reference HW (when CONFIG_KVM_AMD_SEV=y and capable HW present). (covers REQ-O9)
- [ ] AC-O9: TDX guest launch via `KVM_TDX_INIT_VM` succeeds on Intel reference HW (when CONFIG_INTEL_TDX_HOST=y and capable HW present). (covers REQ-O9)
- [ ] AC-O10: kvm-unit-tests x86 suite passes with same pass/fail manifest as upstream. (covers REQ-O1, REQ-O11, REQ-O12)
- [ ] AC-O11: kselftest `tools/testing/selftests/kvm/` passes with same manifest. (covers REQ-O1, REQ-O11, REQ-O12)
- [ ] AC-O12: Nested guest (KVM-on-KVM) boots Linux guest under L1 KVM running under L0 Rookery KVM. (covers REQ-O1)

## Verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/kvm/vcpu_runloop.tla` | `virt/kvm/kvm-core.md` (proves: vCPU `KVM_RUN` blocking + signal-pending preempt + immediate_exit handshake; no missed signal/wakeup) |
| `models/kvm/dirty_ring.tla` | `virt/kvm/dirty-tracking.md` (proves: producer-side enqueue from vCPU thread + consumer-side dequeue from userspace migration thread observe acquire-release with no torn or duplicated GFN entries) |
| `models/kvm/spte_lifecycle.tla` | `virt/kvm/x86-mmu-tdp.md` (proves: SPTE state machine — non-present → present → R/W → dirty → flushed → unmap; concurrent host MMU notifier invalidation + guest fault never produce a stale SPTE pointing to a freed pfn) |
| `models/kvm/lapic_injection.tla` | `virt/kvm/x86-lapic.md` (proves: in-kernel APIC interrupt injection + EOI ordering matches `KVM_GET_LAPIC` snapshot semantics; lowest-priority delivery picks one and only one destination) |
| `models/kvm/posted_intr.tla` | `virt/kvm/vmx-posted-intr.md` (proves: PID descriptor PIR/SN/ON bits race-free under concurrent IPI from host + remote vCPU) |
| `models/kvm/sev_snp_psp.tla` | `virt/kvm/svm-sev.md` (proves: SEV-SNP PSP command serialization + RMP table updates atomic; no firmware command interleave that violates SNP spec) |
| `models/kvm/irqfd_lifetime.tla` | `virt/kvm/eventfd.md` (proves: irqfd add/remove + eventfd close + irq-injection race — no use-after-free of the eventfd_ctx) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `virt/kvm/kvm-core.md` | vcpu fd refcount + close-during-run safety; vcpu->run page mapping lifetime tied to vcpu fd |
| `virt/kvm/memslots.md` | Memory slot RCU-lookup invariant: every slot reachable from `kvm_memslots(kvm, as_id)` has a non-zero refcount and a valid hva range |
| `virt/kvm/x86-mmu-tdp.md` | TDP page-table walk pre/postcondition: walk-result GPA always corresponds to a present SPTE at time of read, or returns RET_PF_RETRY; no infinite walk |

## Hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-vm + per-vcpu + per-memslot + per-irqfd refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-arch `kvm_x86_ops` vtable + `kvm_irq_chip` ops `static const` | § Mandatory |
| **SIZE_OVERFLOW** | gpa/hva arithmetic (slot offset + length + alignment) uses checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | KVM hypercall + emulator dispatch tables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed vCPU + VMCS/VMCB state cleared (carries guest secrets) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | KVM JIT-emitted vmenter trampolines (if any vendor uses) follow W^X | § Mandatory |

### Row-2 (LSM-stackable) features for KVM

KVM-side LSM hooks called: `security_kvm_*` (e.g., `security_kvm_dev_ioctl`). Per-vm LSM label inherited from creating process's LSM context. SELinux + AppArmor + future GR-RBAC stackable LSM each get a chance to mediate `KVM_CREATE_VM`, `KVM_CREATE_VCPU`, `KVM_RUN`, `KVM_MEMORY_ENCRYPT_OP`. GR-RBAC adds: per-role `/dev/kvm` access cap; per-role disallow of nested-virt; per-role disallow of TDX/SEV-SNP launch.

### KVM-specific reinforcement

- **Nested-guest VMCS/VMCB field validation**: all `KVM_SET_NESTED_STATE` paths must validate every guest-controlled field (consistency-check attacks like CVE-2018-12126 / CVE-2020-2732 / etc.). Tier-3 `vmx-nested.md` + `svm-nested.md` enumerate every check; Layer-4 contracts pin the validation list.
- **SEV/TDX firmware boundary**: PSP + TDX-module commands serialized via dedicated mutex per `00-security-principles.md` § "trust-boundary explicit"; failure modes audited (cross-ref `svm/sev.c` + `vmx/tdx.c`).
- **MMU-notifier invalidation completeness**: every host pte change visible to KVM before reuse; failure → guest reads freed page. Layer-2 `spte_lifecycle.tla` is the proof.
- **CPUID filtering**: KVM_SET_CPUID2 entries cross-checked against host-supported features (no exposing CPUID bits that lack a matching VMCS/VMCB control or emulator path → guest #UD weirdness or worse).
- **MSR filter**: KVM_X86_SET_MSR_FILTER honored; default filter conservative.

## Open Questions

(none at Tier-2 — defer to per-Tier-3 once Phase D begins; expected hot questions: TDX module ABI version pin, SEV-SNP firmware version pin, Hyper-V emulation surface scope, Xen PV emulation scope, `enable_apicv` default on vs off)

## Out of Scope

- Per-Tier-3 (Phase D)
- ARM / RISC-V / s390 / PPC / MIPS / LoongArch KVM (`arch/arm64/kvm/` etc. — out of v0 since x86_64-only)
- KVM PV-MMU (long-deprecated — not implemented)
- 32-bit guest support beyond the unrestricted-guest fallback (real-mode + protected-mode w/o paging via emulator + EPT identity-map)
- Implementation code
