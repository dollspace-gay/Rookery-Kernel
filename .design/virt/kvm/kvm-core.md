# Tier-3: virt/kvm/kvm_main.c — KVM core (vm/vcpu fd lifecycle + ioctl dispatch + vCPU thread loop)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/00-overview.md
upstream-paths:
  - virt/kvm/kvm_main.c
  - include/linux/kvm_host.h
  - include/uapi/linux/kvm.h
-->

## Summary

The arch-agnostic core of KVM — what userspace touches when it opens `/dev/kvm`, calls `KVM_CREATE_VM`, calls `KVM_CREATE_VCPU`, mmaps the per-vcpu shared page, blocks in `KVM_RUN`. This Tier-3 covers `virt/kvm/kvm_main.c` (~6600 lines):

- `/dev/kvm` chardev open + `KVM_*` system ioctl dispatch
- `struct kvm` (per-VM) lifecycle: `kvm_create_vm` → `kvm_destroy_vm`
- vm fd: `KVM_CREATE_VCPU`, `KVM_SET_USER_MEMORY_REGION[2]`, `KVM_GET_DIRTY_LOG`, `KVM_SET_GSI_ROUTING`, `KVM_SIGNAL_MSI`, etc. ioctl dispatch
- `struct kvm_vcpu` (per-vCPU) lifecycle: `kvm_vm_ioctl_create_vcpu` → vcpu fd → `kvm_vcpu_release`
- vcpu fd: `KVM_RUN` (the central blocking ioctl), `KVM_GET/SET_REGS`, `KVM_GET/SET_SREGS`, etc.
- Per-vcpu mmap (`vcpu->run` shared page + per-vcpu coalesced-MMIO ring + per-vcpu dirty-ring)
- vCPU thread main loop (the `kvm_vcpu_compat_ioctl` blocking call): handle signals, immediate_exit, vCPU IO/MMIO completion, then call `kvm_arch_vcpu_ioctl_run` (per-arch entry to VMX/SVM)
- MMU notifier integration (`kvm_mmu_notifier_*`) — host page-table changes invalidate guest SPTEs
- Per-VM RCU memslot table mgmt (`kvm_swap_active_memslots`, `kvm_set_memslot`)
- vcpu preempt notifiers (`kvm_sched_in` / `kvm_sched_out`) for arch-specific context save/restore around context switches
- Per-VM stats + per-vcpu stats (binary-stats + debugfs)

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm` | per-VM control block | `kernel::kvm::Vm` |
| `struct kvm_vcpu` | per-vCPU control block | `kernel::kvm::Vcpu` |
| `kvm_chardev_ops` | `/dev/kvm` file_operations | `kvm::DevKvm::FOPS` |
| `kvm_vm_fops` | vm fd file_operations | `kvm::Vm::FOPS` |
| `kvm_vcpu_fops` | vcpu fd file_operations | `kvm::Vcpu::FOPS` |
| `kvm_dev_ioctl(fp, ioctl, arg)` | `/dev/kvm` ioctl dispatch (KVM_GET_API_VERSION, KVM_CREATE_VM, KVM_GET_VCPU_MMAP_SIZE, KVM_GET_SUPPORTED_CPUID, KVM_CHECK_EXTENSION) | `DevKvm::ioctl` |
| `kvm_vm_ioctl(fp, ioctl, arg)` | vm fd ioctl dispatch | `Vm::ioctl` |
| `kvm_vcpu_ioctl(fp, ioctl, arg)` | vcpu fd ioctl dispatch | `Vcpu::ioctl` |
| `kvm_create_vm(type)` | construct kvm struct + memslots + arch-init | `Vm::create` |
| `kvm_destroy_vm(kvm)` | inverse | `Vm::destroy` (Drop) |
| `kvm_get_kvm(kvm)` / `_put_kvm(kvm)` | per-VM refcount | `Arc<Vm>` get/put |
| `kvm_vm_ioctl_create_vcpu(kvm, n)` | construct vcpu struct + arch-init + alloc vcpu fd | `Vm::create_vcpu` |
| `kvm_vcpu_init(vcpu, kvm, id)` | init shared fields | `Vcpu::init` |
| `kvm_vcpu_destroy(vcpu)` | inverse | `Vcpu::destroy` (Drop) |
| `kvm_get_vcpu(kvm, idx)` | lookup vcpu by index | `Vm::vcpu_at` |
| `kvm_for_each_vcpu(idx, vcpu, kvm)` | iterate vCPUs | `Vm::iter_vcpus` |
| `kvm_arch_vcpu_ioctl_run(vcpu)` | per-arch entry to VMX/SVM run loop (cross-ref `vmx-core.md` / `svm-core.md`) | `arch::kvm::vcpu_run` |
| `kvm_vcpu_block(vcpu)` | block vcpu thread waiting for HALT-wakeup, IPI, runnable | `Vcpu::block` |
| `kvm_vcpu_kick(vcpu)` | wake blocked vcpu (from another CPU) | `Vcpu::kick` |
| `kvm_set_memory_region(kvm, mem)` / `_set_internal_*` | install user memslot | `Vm::set_memory_region` |
| `kvm_get_dirty_log(kvm, log)` / `_clear_dirty_log` | per-slot dirty bitmap | `Vm::get_dirty_log` / `_clear_dirty_log` |
| `kvm_arch_check_processor_compat()` | per-arch boot probe | `arch::kvm::check_processor_compat` |
| `kvm_arch_init(opaque)` / `_exit()` | per-arch global init/exit | `arch::kvm::init` / `_exit` |
| `kvm_init(opaque, vcpu_size, vcpu_align, this_module)` | subsystem init (one-time, called from arch_init) | `Subsystem::init` |
| `kvm_exit()` | subsystem exit | `Subsystem::exit` |
| `kvm_mmu_notifier_*` family (`change_pte`, `invalidate_range_{start,end}`, `clear_flush_young`, `test_young`) | host MMU notifier callbacks | `Vm::mmu_notifier_*` |
| `kvm_sched_in(vcpu, cpu)` / `_sched_out(vcpu, cpu)` | preempt notifier (per-arch context save/restore) | `Vcpu::sched_in` / `_sched_out` |
| `kvm_io_bus_register_dev(kvm, bus, addr, len, &dev)` / `_unregister_dev` | register in-kernel I/O device (e.g., LAPIC, IOAPIC, PIT) | `Vm::io_bus_register` |
| `kvm_io_bus_read(vcpu, bus, addr, len, val)` / `_write` | dispatch in-kernel I/O during VM-exit | `Vm::io_bus_read` / `_write` |
| `kvm_irq_routing_update(kvm)` | update GSI routing table | `Vm::irq_routing_update` |

## Compatibility contract

REQ-1: `/dev/kvm` chardev present, mode 0660, owned by root:kvm group, with all upstream `KVM_*` system ioctls dispatching identically.

REQ-2: `KVM_GET_API_VERSION` returns 12 (frozen).

REQ-3: `KVM_CREATE_VM` returns vm fd; `vm_type` arg accepted set per `KVM_CHECK_EXTENSION(KVM_CAP_VM_TYPES)`.

REQ-4: `KVM_CREATE_VCPU` returns vcpu fd; vcpu fd is mmappable at offset 0 producing the `struct kvm_run` shared page (size = `KVM_GET_VCPU_MMAP_SIZE`).

REQ-5: `KVM_RUN` blocks the calling thread inside the per-arch run loop until: a VM-exit requires userspace handling (KVM_EXIT_IO / _MMIO / _HYPERCALL / _DEBUG / _INTERNAL_ERROR / _SHUTDOWN / _FAIL_ENTRY / _SET_TPR / _NMI / _IOAPIC_EOI / _SYSTEM_EVENT / _XEN / _MEMORY_FAULT / _NOTIFY / _DIRTY_RING_FULL / _AP_RESET_HOLD / _X86_RDMSR / _X86_WRMSR / _X86_BUS_LOCK / _TDX), the calling thread receives a fatal signal (returns -EINTR), or `immediate_exit=1` is set in the shared page.

REQ-6: Per-vCPU `struct kvm_run` mmap'd shared page byte-identical layout (qemu reads `request_interrupt_window`, `immediate_exit`, `cr8`, `apic_base`, `exit_reason`, per-exit union members directly).

REQ-7: `KVM_SET_USER_MEMORY_REGION[2]` installs a memory slot; `kvm_memslots(kvm, as_id)` lookup returns RCU-stable snapshot; concurrent slot install + vCPU access never produce torn slot view.

REQ-8: MMU notifier handlers invoked correctly: every host pte change visible to KVM before the corresponding guest page is reused; `kvm_mmu_notifier_invalidate_range_start` blocks new guest faults on the range; `_invalidate_range_end` releases.

REQ-9: Per-VM + per-vCPU stats counters (binary-stats + debugfs) byte-identical for in-tree consumers (`perf kvm stat`, `kvm-perf-stat-tools`).

REQ-10: vCPU preempt notifier (`kvm_sched_in` / `_sched_out`) called from kernel scheduler exactly once per per-vcpu-thread context-switch event; per-arch context save/restore (VMCS clear / VMCB swap / fpu state) preserved.

## Acceptance Criteria

- [ ] AC-1: `qemu-system-x86_64 -enable-kvm -kernel <vmlinuz> -m 1G -smp 2 -nographic -append console=ttyS0` boots reference Debian guest to login prompt.
- [ ] AC-2: cloud-hypervisor + firecracker boot reference rootfs.
- [ ] AC-3: `KVM_CHECK_EXTENSION` returns matching values for the v7.1 cap set on every supported KVM_CAP_*.
- [ ] AC-4: vCPU mmap of run page works on every vCPU fd; size matches `KVM_GET_VCPU_MMAP_SIZE`.
- [ ] AC-5: Stress: 100x VM create+destroy cycles + 8x vCPU create+destroy per VM; KASAN clean; no fd/memory leak.
- [ ] AC-6: `perf kvm stat live` decodes vmexit reasons correctly.
- [ ] AC-7: MMU notifier test: madvise(MADV_DONTNEED) on a guest-mapped region while guest is reading triggers EPT/NPT invalidation; subsequent guest access faults + re-faults correctly.
- [ ] AC-8: Live-migration with dirty-ring mode transfers correct dirty pages.
- [ ] AC-9: kvm-unit-tests x86 suite passes with same pass/fail manifest as upstream.
- [ ] AC-10: kselftest `tools/testing/selftests/kvm/` passes with same manifest.

## Architecture

`Vm` lives in `kernel::kvm::Vm`:

```
struct Vm {
  refcount: Refcount,
  arch: arch::kvm::ArchVm,         // per-arch state (VMX/SVM specific)
  memslots: [RCUMutex<KBox<KvmMemslots>>; KVM_ADDRESS_SPACE_NUM],
  vcpus: ArrayMutex<MAX_VCPUS, Option<Arc<Vcpu>>>,
  buses: [RCUMutex<KBox<KvmIoBus>>; KVM_NR_BUSES],
  irq_routing: RCUMutex<KBox<KvmIrqRouting>>,
  irqfds: Mutex<Vec<Arc<Irqfd>>>,
  ioeventfds: Mutex<Vec<Arc<Ioeventfd>>>,
  coalesced_mmio_ring: Option<KBox<CoalescedMmioRing>>,
  guest_memfd_files: Mutex<Vec<Arc<GuestMemfdFile>>>,  // CONFIG_KVM_PRIVATE_MEM
  mmu_notifier: KBox<MmuNotifier>,
  stats: RcuMutex<KBox<VmStats>>,
  dirty_ring_size: Option<NonZeroU32>,
  debugfs_dentry: KBox<DebugfsDir>,
  arch_dirty_log_supported: bool,
}
```

`Vcpu` lives in `kernel::kvm::Vcpu`:

```
struct Vcpu {
  refcount: Refcount,
  vm: Arc<Vm>,
  vcpu_idx: u32,
  vcpu_id: u32,
  cpu: AtomicI32,             // currently-running CPU or -1
  arch: arch::kvm::ArchVcpu,   // per-arch state (VMCS, FPU, registers)
  run: NonNull<KvmRun>,        // mmap'd shared page
  pid: AtomicPtr<Pid>,         // owning task
  preempt_notifier: PreemptNotifier,
  blocked: AtomicBool,
  wq: WaitQueue,
  ready: AtomicBool,
  dirty_ring: Option<KBox<DirtyRing>>,
  stats: RcuMutex<KBox<VcpuStats>>,
}
```

`/dev/kvm` open path:
1. `kvm_chardev_ops::open` installs `kvm_chardev_ops::FOPS`.
2. Userspace ioctl(KVM_CREATE_VM, type) → `kvm_dev_ioctl_create_vm(type)`:
   - `Vm::create(type)`: allocate kvm struct, init memslots, init arch (`arch::kvm::vm_init(&mut vm)`), register MMU notifier, create debugfs subdir, return Arc<Vm>.
   - `anon_inode_getfd("kvm-vm", &kvm_vm_fops, vm.into_raw(), O_RDWR | O_CLOEXEC)` → vm fd.
3. Userspace ioctl(KVM_CREATE_VCPU, n) → `Vm::create_vcpu(n)`:
   - Allocate Vcpu struct, init shared run page (`alloc_pages` for the ~16KB region), init preempt notifier.
   - `arch::kvm::vcpu_create(&mut vcpu)` (per-arch VMCS / VMCB init, FPU init, MSR-load list).
   - `Vm::vcpus[n] = Some(Arc::clone(&vcpu))`.
   - `anon_inode_getfd("kvm-vcpu", &kvm_vcpu_fops, vcpu.into_raw(), O_RDWR | O_CLOEXEC)` → vcpu fd.
4. Userspace ioctl(KVM_RUN, 0) → `Vcpu::run()` blocks the thread until exit.

`Vcpu::run()` main loop:
```
loop {
  if signal_pending(current()) { return -EINTR; }
  if vcpu.run.immediate_exit { vcpu.run.immediate_exit = 0; return 0; }
  if !vcpu.ready.load(Acquire) {
    Vcpu::block(&vcpu);  // wait for runnable / IPI / wakeup
    continue;
  }
  let r = arch::kvm::vcpu_run(&mut vcpu);  // → enters guest via VMRUN/VMENTER
  match r {
    EXIT_NEED_USERSPACE => return 0,  // exit_reason set in vcpu.run; userspace handles
    EXIT_RESCHED => continue,         // re-enter guest
    EXIT_FATAL(e) => return e,
  }
}
```

MMU notifier integration: `Vm::mmu_notifier_invalidate_range_start(start, end)` → drops SPTEs for [start, end), bumps `mmu_invalidate_in_progress`; `_end` decrements + wakes vCPUs blocked on `mmu_invalidate_seq` change. Concurrent guest fault during invalidation retries via `kvm_handle_hva_range_no_flush(...)` returning RET_PF_RETRY.

Memslot install (`Vm::set_memory_region`):
1. Validate region (alignment, no overlap, gpa+size doesn't overflow, hva backed by VMA).
2. Allocate new `KvmMemslots` array (RCU copy-on-write).
3. Insert/replace/delete the slot per `KVM_MEM_*` flags.
4. RCU swap pointers atomically.
5. Free old `KvmMemslots` after grace period.

Concurrent vCPU access during memslot install: vCPU uses `kvm_memslots(kvm, as_id)` which is RCU-protected; sees either old or new slot table fully.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `vm_no_uaf` | UAF | `Arc<Vm>` outlives every Arc<Vcpu>; vcpu.vm is always valid. |
| `vcpu_run_mmap_lifetime` | UAF | The `vcpu->run` mmap'd page is freed only after vcpu fd is closed AND no in-flight `KVM_RUN` is using it. |
| `memslot_no_overflow` | OVERFLOW | gpa + size + hva arithmetic checked against u64 overflow. |
| `vcpu_idx_no_oob` | OOB | `Vm::vcpus[idx]` access bounds-checked against MAX_VCPUS. |

### Layer 2: TLA+

`models/kvm/vcpu_runloop.tla` (parent-declared): proves vCPU run-loop signal-pending preempt + immediate_exit handshake; no missed signal/wakeup.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vm::set_memory_region` post: every gpa in old slot's `[gpa, gpa+size)` either still maps (if not deleted) OR returns gfn-not-present on subsequent translation | `Vm::set_memory_region` |
| `Vcpu::run` pre: vcpu->cpu == this CPU, vcpu->ready.load() == true | `Vcpu::run` |
| `Vcpu::block`+`kvm_vcpu_kick` pair invariant: every block matched by exactly one wakeup OR signal | `Vcpu::block` / `_kick` |

### Layer 4: Verus/Creusot functional

`Vm::create` → `Vm::create_vcpu` → `Vcpu::run` → guest-execute → `Vcpu::run returns to userspace` round-trip equivalence: a qemu-driven KVM_RUN call observes the same `vcpu->run` field values as upstream KVM for every supported VM-exit reason. Encoded as Verus model: `forall vm vcpu exit. rookery_run_until_exit(vm, vcpu) == upstream_run_until_exit(vm, vcpu)`.

## Hardening

(Inherits row-1 features from `virt/kvm/00-overview.md` § Hardening.)

kvm-core specific reinforcement:

- **Per-VM + per-vcpu refcount saturating** — overflow saturates at u32::MAX.
- **`/dev/kvm` open requires CAP_SYS_ADMIN** OR membership in `kvm` group with mode 0660 (defense against unprivileged process creating VMs to exploit guest-side bugs that escape).
- **MMU notifier invalidation completeness** — every host page change visible to KVM before page reuse (Layer-2 spte_lifecycle.tla in parent proves this).
- **Memslot validation** — every `KVM_SET_USER_MEMORY_REGION[2]` validated for gpa+size+memory_size+userspace_addr alignment + bounds + no overlap with existing slots + hva backed by valid VMA before commit.
- **vcpu fd close-during-run** — `Vcpu::run` holds Arc<Vcpu>; close from another thread waits for in-flight run to complete via vcpu.mutex + signal_pending check.
- **Stats counters via per-CPU + RCU** — never globally-locked; never overflow (saturating).
- **dirty_ring concurrent producer (vCPU thread) + consumer (userspace migration thread)** with acq-rel fences (Layer-2 dirty_ring.tla in parent proves correctness).
- **Per-VM debugfs dir owned by mode 0700 root** — defense against unprivileged process reading guest-state-derived stats.
- **MAX_VCPUS hard cap = 1024 per VM** in v0 (configurable via Kconfig); defense against per-VM resource exhaustion.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-arch run-loop entry (covered in `virt/kvm/x86-core.md` + `vmx-core.md` + `svm-core.md`)
- Memslot internals (covered in `virt/kvm/memslots.md` future Tier-3)
- Dirty-ring impl (covered in `virt/kvm/dirty-tracking.md` future Tier-3)
- irqfd / ioeventfd (covered in `virt/kvm/eventfd.md` future Tier-3)
- guest_memfd (covered in `virt/kvm/guest-memfd.md` future Tier-3)
- vfio bridge (covered in `virt/kvm/vfio-bridge.md` future Tier-3)
- async page-fault (covered in `virt/kvm/async-pf.md` future Tier-3)
- 32-bit-only paths
- Implementation code
