# Tier-3: virt/kvm/irqchip.c — KVM cross-arch IRQ routing infrastructure (GSI → MSI/IOAPIC/PIC dispatch)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - virt/kvm/irqchip.c (~261 lines)
  - include/linux/kvm_host.h (kvm_irq_routing_table, kvm_kernel_irq_routing_entry)
  - arch/x86/kvm/irq.c (set_routing_entry impl)
-->

## Summary

Per-VM IRQ routing table maps **GSI** (Global System Interrupt; userspace-allocated) → **kernel routing entry** (KVM_IRQ_ROUTING_IRQCHIP for IOAPIC/PIC pin OR KVM_IRQ_ROUTING_MSI for MSI message OR KVM_IRQ_ROUTING_HV_SINT for Hyper-V SynIC OR KVM_IRQ_ROUTING_XEN for Xen evtchn). Per-eventfd / per-VFIO-IRQ resolves GSI → entry → arch-specific deliver-handler. Per-VM userspace ABI: KVM_SET_GSI_ROUTING ioctl publishes new routing-table; old-rt freed via RCU. Per-VM kvm.irq_routing pointer RCU-protected for fast read-side. Critical for: irqfd, MSI-X, VFIO assigned-device IRQ, Hyper-V SynIC, Xen evtchn.

This Tier-3 covers `irqchip.c` (~261 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_irq_routing_table` | per-VM routing-table | `KvmIrqRoutingTable` |
| `struct kvm_kernel_irq_routing_entry` | per-GSI entry | `KvmKernelIrqRoutingEntry` |
| `kvm.irq_routing` | per-VM RCU-pointer | `KvmArch::irq_routing` |
| `kvm_set_irq_routing()` | per-VM publish new rt | `Kvm::set_irq_routing` |
| `kvm_set_routing_entry()` | per-arch parse user-entry | `Kvm::set_routing_entry` |
| `setup_routing_entry()` | per-rt assemble | `Kvm::setup_routing_entry` |
| `kvm_set_irq()` | per-IRQ delivery dispatch | `Kvm::set_irq` |
| `kvm_send_userspace_msi()` | userspace-direct MSI | `Kvm::send_userspace_msi` |
| `kvm_irq_routing_update()` | per-arch post-update hook | `Kvm::irq_routing_update` |
| `KVM_IRQ_ROUTING_IRQCHIP` | per-entry type | UAPI |
| `KVM_IRQ_ROUTING_MSI` | per-entry type | UAPI |
| `KVM_IRQ_ROUTING_HV_SINT` | per-entry type | UAPI |
| `KVM_IRQ_ROUTING_XEN_EVTCHN` | per-entry type | UAPI |
| `KVM_IRQCHIP_NUM_PINS` | per-VM max GSI for IRQCHIP routing | shared |

## Compatibility contract

REQ-1: Per-VM kvm_irq_routing_table:
- `chip[KVM_NR_IRQCHIPS][KVM_IRQCHIP_NUM_PINS]`: per-irqchip-pin → GSI lookup.
- `nr_rt_entries`: number of GSIs.
- `map`: per-GSI head of list of routing-entries (multi-target broadcast).

REQ-2: Per-routing-entry:
- `gsi`: u32.
- `type`: KVM_IRQ_ROUTING_*.
- `flags`: KVM_MSI_VALID_DEVID / etc.
- `set`: per-arch dispatch fn pointer.
- Per-type union:
  - irqchip: { irqchip, pin }.
  - msi: { address_lo, address_hi, data, devid }.
  - hv_sint: { vcpu, sint }.
  - xen: { type, port_or_pirq, vcpu_idx }.

REQ-3: Per-VM kvm_set_irq_routing(kvm, ue, nr, flags):
- Allocate new kvm_irq_routing_table sized for `nr` entries.
- For i in 0..nr:
  - setup_routing_entry(kvm, new_rt, e, &ue[i])?
- Atomic-swap kvm.irq_routing → new_rt.
- synchronize_srcu (kvm.irq_srcu) before freeing old_rt.

REQ-4: Per-arch kvm_set_routing_entry(kvm, e, ue):
- Per-x86: set e.set = kvm_set_msi (msi) | kvm_set_pic_irq (irqchip pic) | kvm_set_ioapic_irq (irqchip ioapic) | kvm_xen_set_evtchn (xen) | kvm_hv_set_sint (hv_sint).

REQ-5: kvm_set_irq(kvm, irq_source_id, irq, level, line_status):
- srcu_read_lock(&kvm.irq_srcu).
- rt = rcu_dereference(kvm.irq_routing).
- For each entry in rt.map[irq]:
  - entry.set(entry, kvm, irq_source_id, level, line_status).
- srcu_read_unlock.
- Return ret.

REQ-6: kvm_send_userspace_msi(kvm, msi):
- Build kvm_kernel_irq_routing_entry locally with msg.address/data.
- entry.set = kvm_set_msi.
- kvm_set_msi(entry, kvm, KVM_USERSPACE_IRQ_SOURCE_ID, 1, false).

REQ-7: Per-arch irq_routing_update:
- After kvm.irq_routing swap: invoke kvm_arch_irq_routing_update(kvm).
- Per-x86: update IOAPIC RTEs from new rt; update VFIO-PI IRTEs.

REQ-8: Per-flag KVM_MSI_VALID_DEVID:
- For interrupt remapping: msi.devid identifies source-PCI BDF.

REQ-9: Per-rt update userspace ABI:
- KVM_SET_GSI_ROUTING ioctl: copy `struct kvm_irq_routing` array of entries.

REQ-10: Per-multi-route per-GSI:
- Per-GSI may have multiple kvm_kernel_irq_routing_entry on map list (e.g. fan-out).

## Acceptance Criteria

- [ ] AC-1: KVM_CREATE_IRQCHIP: default routing-table established (PIC + IOAPIC GSIs).
- [ ] AC-2: KVM_SET_GSI_ROUTING with MSI entry GSI=24 + addr=0xFEE00000 + data=0x4030: msg-based delivery on kvm_set_irq(24).
- [ ] AC-3: kvm_set_irq(0, 0, 1, 0): IOAPIC pin 0 raised; kvm_lapic_set_irq invoked.
- [ ] AC-4: kvm_send_userspace_msi: builds entry on stack; calls kvm_set_msi.
- [ ] AC-5: KVM_SET_GSI_ROUTING twice in succession: SRCU synchronizes; old rt freed.
- [ ] AC-6: kvm_set_irq under irq_srcu read-lock: no UAF.
- [ ] AC-7: HV_SINT entry: kvm_hv_set_sint dispatches to per-vCPU SynIC.
- [ ] AC-8: Xen evtchn entry: kvm_xen_set_evtchn dispatches.
- [ ] AC-9: Per-multi-GSI: same GSI dispatches all entries on map list.
- [ ] AC-10: KVM_MSI_VALID_DEVID: msi.devid passed to IOMMU IRTE.

## Architecture

Per-VM routing-table:

```
struct KvmIrqRoutingTable {
  chip: [[i32; KVM_IRQCHIP_NUM_PINS]; KVM_NR_IRQCHIPS],   // per-irqchip-pin → GSI
  nr_rt_entries: u32,
  map: VarLenArr<HListHead<KvmKernelIrqRoutingEntry>>,    // per-GSI list
}

struct KvmKernelIrqRoutingEntry {
  gsi: u32,
  type_: u32,                                              // KVM_IRQ_ROUTING_*
  flags: u32,
  set: fn(&Self, &Kvm, irq_source_id, level, line_status) -> i32,
  link: HListLink,
  data: union {
    irqchip: { irqchip: u32, pin: u32 },
    msi: { address_lo: u32, address_hi: u32, data: u32, devid: u32 },
    hv_sint: { vcpu: u32, sint: u32 },
    xen: { type_: u32, port_or_pirq: u32, vcpu_idx: u32 },
  },
}
```

Per-VM:

```
struct KvmArch {
  ...
  irq_routing: SrcuPointer<KvmIrqRoutingTable>,
  irq_srcu: SrcuStruct,
}
```

`Kvm::set_irq_routing(kvm, ue, nr, flags) -> Result<()>`:
1. If nr > KVM_MAX_IRQ_ROUTES: return Err(EINVAL).
2. nr_rt_entries = max(KVM_IRQCHIP_NUM_PINS, max-gsi(ue)+1).
3. new_rt = vmalloc(KvmIrqRoutingTable + nr_rt_entries * sizeof(HListHead)).
4. memset(chip, -1, sizeof(chip)).
5. For i in 0..nr:
   - setup_routing_entry(kvm, new_rt, &ue[i])?
6. mutex_lock(&kvm.irq_lock).
7. old_rt = rcu_dereference_protected(kvm.irq_routing, kvm.irq_lock).
8. rcu_assign_pointer(kvm.irq_routing, new_rt).
9. kvm_arch_irq_routing_update(kvm).
10. mutex_unlock(&kvm.irq_lock).
11. synchronize_srcu_expedited(&kvm.irq_srcu).
12. free_irq_routing_table(old_rt).
13. Ok.

`Kvm::setup_routing_entry(kvm, rt, e, ue) -> Result<()>`:
1. r = kvm_set_routing_entry(kvm, e, ue)? .
2. If e.type == KVM_IRQ_ROUTING_IRQCHIP: rt.chip[e.data.irqchip][e.data.pin] = e.gsi.
3. hlist_add_head(&e.link, &rt.map[e.gsi]).

`Kvm::set_irq(kvm, irq_source_id, irq, level, line_status) -> i32`:
1. idx = srcu_read_lock(&kvm.irq_srcu).
2. rt = rcu_dereference(kvm.irq_routing).
3. ret = -1.
4. If irq < rt.nr_rt_entries:
   - For each e in rt.map[irq]:
     - r = e.set(e, kvm, irq_source_id, level, line_status).
     - If r > ret: ret = r.
5. srcu_read_unlock(&kvm.irq_srcu, idx).
6. Return ret.

`Kvm::send_userspace_msi(kvm, msi) -> Result<()>`:
1. e = KvmKernelIrqRoutingEntry::new_msi(msi.address_lo, msi.address_hi, msi.data, msi.devid).
2. e.set = kvm_set_msi.
3. e.set(&e, kvm, KVM_USERSPACE_IRQ_SOURCE_ID, 1, false).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nr_routes_le_max` | INVARIANT | per-set: nr ≤ KVM_MAX_IRQ_ROUTES. |
| `gsi_lt_nr_rt_entries` | INVARIANT | per-rt: e.gsi < rt.nr_rt_entries. |
| `chip_pin_lt_pins` | INVARIANT | per-IRQCHIP entry: pin < KVM_IRQCHIP_NUM_PINS. |
| `set_fn_non_null` | INVARIANT | per-rt entry: e.set != null. |
| `srcu_read_held_during_set_irq` | INVARIANT | per-set_irq: rt-deref under srcu_read_lock. |

### Layer 2: TLA+

`virt/kvm/irq_routing.tla`:
- Per-VM RT-update + per-set_irq dispatch.
- Properties:
  - `safety_no_stale_rt_after_swap` — post-set_irq_routing: subsequent set_irq uses new rt.
  - `safety_old_rt_freed_after_srcu` — old_rt freed only after srcu sync.
  - `liveness_set_irq_eventually_completes` — per-set_irq returns within bounded entries.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Kvm::set_irq_routing` post: kvm.irq_routing == new_rt; old_rt freed | `Kvm::set_irq_routing` |
| `Kvm::set_irq` post: per-entry on map list invoked | `Kvm::set_irq` |
| `Kvm::send_userspace_msi` post: kvm_set_msi invoked with msi fields | `Kvm::send_userspace_msi` |
| `Kvm::setup_routing_entry` post: e linked into rt.map[e.gsi] | `Kvm::setup_routing_entry` |

### Layer 4: Verus/Creusot functional

`Per-userspace KVM_SET_GSI_ROUTING → kvm.irq_routing reflects new rt; per-irqfd / per-VFIO IRQ delivers via routed entry` semantic equivalence: per-routing matches KVM API doc on GSI semantics.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

IRQ-routing-specific reinforcement:

- **Per-rt SRCU read-lock for fast-path** — defense against per-update reader-blocking.
- **Per-rt update under kvm.irq_lock** — defense against concurrent updates causing torn-state.
- **Per-rt old freed only after srcu_synchronize** — defense against UAF mid-fast-path.
- **Per-entry e.set non-null enforced** — defense against per-arch handler missing.
- **Per-entry type validated against KVM_IRQ_ROUTING_*** — defense against type confusion.
- **Per-MSI valid_devid required for IRTE-posting** — defense against incomplete MSI for IOMMU.
- **Per-GSI bounded by KVM_MAX_IRQ_ROUTES** — defense against unbounded rt-table alloc.
- **Per-irqchip pin bounded by KVM_IRQCHIP_NUM_PINS** — defense against per-pin OOB access.
- **Per-rt-table vmalloc'd (large)** — defense against alloc-failure on small kmalloc cache.
- **Per-update kvm_arch_irq_routing_update propagates to PI/IRTE** — defense against per-VFIO posted-IRQ stale routing.
- **Per-userspace MSI uses KVM_USERSPACE_IRQ_SOURCE_ID** — defense against per-source confusion.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- IOAPIC (covered in `x86-ioapic.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- 8259 PIC (covered in `x86-pic.md` Tier-3)
- Posted Interrupts (covered in `x86-posted-intr.md` Tier-3)
- Hyper-V SynIC (covered in `x86-hyperv.md` Tier-3)
- Xen evtchn (covered in `x86-xen.md` Tier-3)
- irqfd (covered in `eventfd.md` Tier-3)
- Implementation code
