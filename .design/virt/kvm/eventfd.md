# Tier-3: virt/kvm/eventfd.c — KVM_IRQFD + KVM_IOEVENTFD (eventfd-driven IRQ injection + MMIO/PIO bypass; vhost integration)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/00-overview.md
upstream-paths:
  - virt/kvm/eventfd.c
  - include/linux/kvm_host.h (struct kvm_kernel_irqfd, struct _ioeventfd)
  - include/uapi/linux/kvm.h (KVM_IRQFD + KVM_IOEVENTFD UAPI)
-->

## Summary

Two complementary eventfd-driven optimizations critical to cloud VM performance:

- **KVM_IRQFD**: `eventfd_signal(eventfd)` from another kernel subsystem (vhost-net/scsi/vsock, vfio-pci virqfd, virtio-vdpa) → triggers IRQ injection into a vCPU via the in-kernel APIC (cross-ref `x86-lapic.md`). Bypass vs userspace round-trip: virtio-net packet RX → vhost-net writes to eventfd → KVM_IRQFD signals vCPU LAPIC → IPI delivers interrupt → guest driver sees IRQ — all in kernel, no qemu involvement on hot path.
- **KVM_IOEVENTFD**: when guest does MMIO/PIO write to specific `gpa+length+match` (e.g., virtio doorbell) → KVM signals a registered eventfd → consumer (vhost) wakes up to process. Bypass vs userspace round-trip for guest virtio doorbell rings.

Plus **resampler**: for level-triggered interrupts (legacy emulated INTx + some virtio devices), `KVM_IRQFD` with `resamplefd` automatically asserts/de-asserts based on guest EOI; userspace doesn't see EOI events.

This Tier-3 covers `virt/kvm/eventfd.c` (~1050 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_kernel_irqfd` | per-irqfd kernel state | `kernel::kvm::Irqfd` |
| `struct kvm_kernel_irqfd_resampler` | per-resamplefd state for level-triggered IRQs | `kernel::kvm::IrqfdResampler` |
| `struct _ioeventfd` | per-ioeventfd state | `kernel::kvm::Ioeventfd` |
| `kvm_irqfd(kvm, args)` | KVM_IRQFD ioctl handler | `Vm::handle_irqfd` |
| `kvm_irqfd_assign(kvm, args)` | install an irqfd | `Vm::irqfd_assign` |
| `kvm_irqfd_deassign(kvm, args)` | uninstall an irqfd | `Vm::irqfd_deassign` |
| `kvm_irqfd_release(kvm)` | per-VM destroy: release all irqfds | `Vm::irqfd_release` |
| `kvm_irq_routing_update(kvm)` | per-VM IRQ routing table update notifier | `Vm::irq_routing_update` |
| `irqfd_wakeup(wait, mode, sync, key)` | eventfd POLLIN callback | `Irqfd::wakeup` |
| `irqfd_inject(work)` | workqueue worker for actual IRQ inject | `Irqfd::inject_work` |
| `irqfd_resampler_ack(eoi)` | per-resampler EOI ack notifier | `IrqfdResampler::ack` |
| `irqfd_resampler_notify(resampler)` | dispatch ack to all irqfd's of resampler | `IrqfdResampler::notify` |
| `kvm_ioeventfd(kvm, args)` | KVM_IOEVENTFD ioctl handler | `Vm::handle_ioeventfd` |
| `kvm_assign_ioeventfd(kvm, args)` / `_deassign_ioeventfd(...)` | install/uninstall ioeventfd | `Vm::assign_ioeventfd` / `_deassign_ioeventfd` |
| `kvm_assign_ioeventfd_idx(kvm, bus_idx, args)` | per-bus install (KVM_PIO_BUS / _MMIO_BUS / _VIRTIO_CCW_NOTIFY_BUS) | `Vm::assign_ioeventfd_idx` |
| `kvm_io_bus_register_dev(kvm, bus_idx, addr, len, &dev)` | per-bus register MMIO/PIO range | `Vm::io_bus_register_dev` |

## Compatibility contract

REQ-1: `KVM_IRQFD(struct kvm_irqfd)` UAPI byte-identical:
- `fd`: eventfd fd (input).
- `gsi`: target GSI (Global System Interrupt) — translates via per-VM IRQ routing table to vector + delivery mode.
- `flags`: `KVM_IRQFD_FLAG_DEASSIGN` / `KVM_IRQFD_FLAG_RESAMPLE`.
- `resamplefd`: resampler fd (when RESAMPLE flag set).
- Returns 0 on success / -errno on failure.

REQ-2: When `eventfd_signal(eventfd)` fires from another subsystem:
- Per-eventfd POLLIN callback `irqfd_wakeup` invoked.
- For edge-triggered: directly call `kvm_set_irq` to inject + advance.
- For level-triggered: queue `irqfd_inject` workqueue work; on guest EOI, `irqfd_resampler_ack` signals resamplefd to userspace.

REQ-3: Per-VM IRQ routing: `KVM_SET_GSI_ROUTING` ioctl installs per-GSI → (vector, delivery_mode, dest_id) mapping. KVM_IRQFD looks up per-GSI route at signal time + dispatches to LAPIC.

REQ-4: `KVM_IOEVENTFD(struct kvm_ioeventfd)` UAPI byte-identical:
- `addr`: MMIO/PIO base address (gpa).
- `len`: 0 / 1 / 2 / 4 / 8 (specific-length match; len=0 matches any length).
- `data`: data value to match against guest write (or 0 for length-only match if `KVM_IOEVENTFD_FLAG_DATAMATCH` clear).
- `fd`: eventfd to signal on match.
- `flags`: `_DATAMATCH` / `_PIO` (else MMIO) / `_DEASSIGN` / `_VIRTIO_CCW_NOTIFY` (s390-only).
- Returns 0 on success.

REQ-5: KVM_IOEVENTFD match on guest write: when guest does I/O at `addr` with matching length + data, KVM signals `eventfd_signal(fd, 1)` instead of returning to userspace via KVM_EXIT_IO/MMIO.

REQ-6: vhost integration: vhost-net + vhost-scsi + vhost-vsock + vhost-vdpa each use KVM_IOEVENTFD pair (one per virtqueue: TX-doorbell + RX-call) + KVM_IRQFD pair (one per virtqueue: TX-completion-notify + RX-data-available-notify).

REQ-7: vfio-pci integration: per-MSI-X-vector eventfd registered via VFIO_DEVICE_SET_IRQS DATA_EVENTFD; KVM_IRQFD pairs with this eventfd → HW MSI-X delivered directly to guest vCPU (no vmexit on common case with posted-interrupts).

REQ-8: Per-irqfd lifetime: registered via KVM_IRQFD, removed via KVM_IRQFD_DEASSIGN OR per-fd close (kvm or eventfd file).

REQ-9: Per-VM all-irqfds released on VM destroy: `kvm_irqfd_release` walks all irqfds + decrements refcounts.

REQ-10: Resampler per-GSI shared: multiple irqfds with same GSI share one resampler; ack from any irqfd's de-assert callback signals all resamplefds.

## Acceptance Criteria

- [ ] AC-1: qemu vhost-net test: virtio-net guest at line-rate; per-VQ doorbell + IRQ both bypassed via KVM_IRQFD + KVM_IOEVENTFD; verified via `perf kvm stat` showing 0 EXIT_REASON_IO_INSTRUCTION for virtio doorbell + 0 EXIT_REASON_EXTERNAL_INTERRUPT for virtio IRQ.
- [ ] AC-2: vfio-pci passthrough test: NIC MSI-X delivered to guest via virqfd → KVM_IRQFD → in-kernel LAPIC; latency within 5% of bare-metal.
- [ ] AC-3: Resampler test: legacy INTx-emulated device with KVM_IRQFD_FLAG_RESAMPLE; guest EOI triggers resamplefd; userspace re-asserts on persistent IRQ source.
- [ ] AC-4: KVM_IRQFD_DEASSIGN: deassign during in-flight IRQ inject — clean shutdown, no UAF.
- [ ] AC-5: KVM_IOEVENTFD with DATAMATCH: only matching data triggers; non-matching writes fall through to userspace.
- [ ] AC-6: KVM_IOEVENTFD len=0 (any-length match): any-size write at addr triggers signal.
- [ ] AC-7: kselftest `tools/testing/selftests/kvm/x86_64/irqfd_test` + `ioeventfd_test` pass.

## Architecture

`Irqfd` lives in `kernel::kvm::Irqfd`:

```
struct Irqfd {
  refcount: Refcount,
  kvm: Arc<Vm>,
  eventfd: Arc<EventFdCtx>,
  gsi: u32,
  irq_entry: KvmKernelIrqRoutingEntry,
  irq_entry_lock: SeqLock,
  list: ListEntry,                         // links into kvm.irqfds list
  pt: PollTable,
  wait: WaitQueueEntry,
  inject: WorkStruct,
  shutdown: WorkStruct,
  resampler: Option<Arc<IrqfdResampler>>,
  resamplefd: Option<Arc<EventFdCtx>>,
  resampler_link: ListEntry,
  irqsource_id: i32,                        // for resampler ack tracking
}

struct IrqfdResampler {
  refcount: Refcount,
  kvm: Arc<Vm>,
  list: Mutex<Vec<Arc<Irqfd>>>,             // irqfds sharing this resampler (same GSI)
  notifier: Mutex<KvmIrqAckNotifier>,
  link: ListEntry,
}

struct Ioeventfd {
  list: ListEntry,
  eventfd: Arc<EventFdCtx>,
  addr: u64,
  length: u32,
  datamatch: u64,
  flags: u32,
  bus_idx: u32,
  io_dev: KvmIoDevice,                      // pseudo-device on KVM IO bus
}
```

`KVM_IRQFD` ioctl flow `Vm::handle_irqfd(args)`:
1. If args.flags & KVM_IRQFD_FLAG_DEASSIGN: `Vm::irqfd_deassign(args)`.
2. Else: `Vm::irqfd_assign(args)`:
   - Allocate `Irqfd` struct.
   - `eventfd = eventfd_ctx_fdget(args.fd)`.
   - If args.flags & KVM_IRQFD_FLAG_RESAMPLE:
     - Look up or alloc IrqfdResampler for this GSI.
     - `resamplefd = eventfd_ctx_fdget(args.resamplefd)`.
   - `init_waitqueue_func_entry(&irqfd.wait, irqfd_wakeup)`.
   - Fetch routing entry for GSI.
   - Add to `kvm.irqfds` list.
   - `eventfd_ctx_add_wait(&irqfd.wait)` — register POLLIN callback.

POLLIN callback `irqfd_wakeup(wait, ...)`:
1. Read seqlock of irq_entry under read lock.
2. Per-entry's delivery_mode + vector + dest:
   - Fast path (no resampler): inject via `kvm_arch_set_irq_inatomic(...)` if possible (no sleeping context).
   - Slow path: queue `irqfd.inject` work + return; workqueue runs `kvm_set_irq(...)`.

`irqfd_inject(work)` workqueue worker:
1. `kvm_set_irq(irqfd.kvm, irqfd.irqsource_id, irqfd.gsi, 1, false)` (assert level=1 for level-triggered, edge-trigger for edge).

Resampler ack `irqfd_resampler_ack(notifier)`:
1. Called from APIC EOI handler when guest acks IRQ.
2. For each irqfd in resampler.list: `kvm_set_irq(..., level=0, ...)` (de-assert).
3. `eventfd_signal(irqfd.resamplefd, 1)` — userspace can now check IRQ source state + re-assert if persistent.

`KVM_IOEVENTFD` ioctl flow `Vm::handle_ioeventfd(args)`:
1. Validate addr+len combination + flags.
2. If args.flags & KVM_IOEVENTFD_FLAG_DEASSIGN: `Vm::deassign_ioeventfd(args)`.
3. Else: `Vm::assign_ioeventfd(args)`:
   - Allocate `Ioeventfd` struct.
   - `eventfd = eventfd_ctx_fdget(args.fd)`.
   - Determine bus_idx: KVM_PIO_BUS if args.flags & KVM_IOEVENTFD_FLAG_PIO else KVM_MMIO_BUS.
   - `kvm_io_bus_register_dev(kvm, bus_idx, args.addr, args.len, &io_dev)` — register pseudo-device on KVM IO bus.
   - When guest does I/O at addr+len with matching data: per-pseudo-device `write` callback `ioeventfd_write(...)`:
     - If args.flags & KVM_IOEVENTFD_FLAG_DATAMATCH: compare write-data vs args.datamatch; mismatch returns -EOPNOTSUPP (fall through to userspace).
     - Else (no datamatch): always match.
     - `eventfd_signal(eventfd, 1)`.
     - Return 0 (handled — no vmexit).

KVM IO bus dispatch (cross-ref `kvm-core.md`):
- Per-bus list of registered devices sorted by addr.
- Guest I/O exit (KVM_EXIT_IO / _MMIO during VMRUN): `kvm_io_bus_write(...)` or `_read(...)` walks list looking for match.
- If found: in-kernel handle (no userspace round-trip).
- Else: return EXIT to userspace (qemu handles).

Per-VM destroy `Vm::irqfd_release`:
1. Walk kvm.irqfds list under mutex.
2. For each irqfd: schedule shutdown work + remove from list.
3. Walk kvm.ioeventfds list: deregister from IO bus + drop eventfd refs.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `irqfd_no_uaf` | UAF | `Arc<Irqfd>` outlives any in-flight wakeup callback + workqueue work; shutdown waits for in-flight to complete. |
| `eventfd_no_uaf` | UAF | `Arc<EventFdCtx>` held while irqfd registered; close-during-signal safe. |
| `routing_seqlock_consistent` | INVARIANT | per-irqfd routing entry read under seqlock; concurrent KVM_SET_GSI_ROUTING update never produces torn read. |
| `resampler_no_double_ack` | INVARIANT | per-irqfd resampler ack invoked exactly once per assert→EOI cycle. |

### Layer 2: TLA+

`models/kvm/irqfd_lifetime.tla` (parent-declared): proves irqfd add/remove + eventfd close + irq-injection race — no use-after-free of the eventfd_ctx.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vm::irqfd_assign` post: irqfd in `kvm.irqfds` list; eventfd POLLIN callback registered | `Vm::irqfd_assign` |
| `Vm::irqfd_deassign` post: irqfd removed; eventfd callback unregistered; in-flight inject work canceled OR completed | `Vm::irqfd_deassign` |
| `irqfd_wakeup` invariant: per-eventfd-signal triggers exactly one IRQ inject (edge) or one assert+resample-cycle (level) | `irqfd_wakeup` |
| `kvm_io_bus_write` invariant: matching ioeventfd's eventfd signaled exactly once per matching guest write | `kvm_io_bus_write` |

### Layer 4: Verus/Creusot functional

`vhost-net writes to eventfd → irqfd_wakeup → kvm_set_irq → LAPIC IRR set → guest VMENTRY → guest sees IRQ` round-trip equivalence: any eventfd_signal eventually delivers IRQ to target vCPU within bounded time (modulo vCPU scheduled-out + KVM_RUN re-entry).

## Hardening

(Inherits row-1 features from `virt/kvm/00-overview.md` § Hardening.)

eventfd-specific reinforcement:

- **Per-irqfd refcount + RCU-safe access** — defense against UAF on close-race + concurrent injection.
- **Per-eventfd close-during-irqfd-signal safe** — eventfd refcount held while irqfd registered; eventfd close blocked until irqfd deassigned.
- **Per-VM irqfd count cap** — bounded (default 1024 per VM); defense against per-VM eventfd-flood causing memory exhaustion.
- **Per-VM ioeventfd count cap** — bounded (default 256 per IO bus per VM).
- **Resampler ack delivered exactly once** — per-IRQ-cycle de-assert serialized via per-IRQ ack notifier.
- **GSI routing seqlock-protected** — concurrent KVM_SET_GSI_ROUTING + KVM_IRQFD signal race-free.
- **Per-irqfd shutdown work serialized** with eventfd signal — defense against in-flight signal + concurrent deassign causing UAF.
- **KVM_IOEVENTFD datamatch validation strict** — len=0 with DATAMATCH rejected (semantic mismatch).
- **vfio-pci virqfd integration LSM-mediated** — cross-ref `drivers/vfio/00-overview.md` § Hardening.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- LAPIC IRQ injection (covered in `x86-lapic.md` Tier-3)
- IOAPIC + PIC (covered in `x86-ioapic.md` future Tier-3)
- vfio-pci virqfd (covered in `drivers/vfio/virqfd.md` future Tier-3)
- vhost integration details (covered in `drivers/vhost/00-overview.md` Tier-2)
- 32-bit-only paths
- Implementation code
