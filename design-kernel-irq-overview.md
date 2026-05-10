---
title: "Tier-2: kernel/irq — IRQ subsystem (irqdesc + irqdomain + irq_chip + msi_domain + manage + softirq + tasklet)"
tags: ["tier-2", "kernel", "irq", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for the kernel IRQ subsystem — the layer between an arch-specific interrupt-vector source (LAPIC + IOAPIC + MSI/MSI-X message + GIC + SoC IC + virtio-IRQ + Hyper-V SynIC) and a driver's `request_irq()` callback. Owns the per-irq descriptor (`struct irq_desc`) keyed by Linux IRQ number, the hierarchical irqdomain abstraction translating per-controller HW-IRQ tokens into Linux IRQ numbers, the `struct irq_chip` per-controller vtable (`mask`, `unmask`, `ack`, `eoi`, `set_affinity`, `set_type`, `retrigger`, …), MSI/MSI-X domain hierarchy, threaded IRQs, IRQ migration on CPU hotplug, the Linux IRQ matrix allocator (vector pool reservation), the per-cpu softirq + tasklet bottom-half infrastructure, IRQ affinity sysfs (`/proc/irq/N/smp_affinity`), generic IPI multiplexing, kexec IRQ teardown, and devres helpers.

Heavily cross-referenced from `arch/x86/cpu-init.md` (LAPIC vector init), `arch/x86/idt.md` (per-vector IDT entries dispatching to `do_IRQ` → `handle_irq` → `generic_handle_irq`), `drivers/pci/00-overview.md` (PCI MSI/MSI-X allocation via `msi_domain_alloc_irqs`), `drivers/iommu/00-overview.md` (IRQ-remap msi_parent_ops integration), `virt/kvm/00-overview.md` (KVM in-kernel APIC + KVM_IRQFD integration), `drivers/base/00-overview.md` (devres `devm_request_irq`), `kernel/sched/00-overview.md` (softirq `ksoftirqd/N` per-cpu kthread + tasklet defer), `kernel/time/00-overview.md` (timer-IRQ + clockevents + tick-broadcast), every device driver in tree.

### Out of Scope

- Per-Tier-3 (Phase D)
- ARM-specific irqchip drivers in `drivers/irqchip/` (separate Tier-2 wrapper future — `arm-gic*.c`, `irq-bcm*.c`, `irq-mtk*.c`, etc.)
- Per-x86 vector-domain glue in `arch/x86/kernel/apic/{vector,io_apic}.c` (lives in `arch/x86/idt.md` Tier-3)
- 32-bit-only paths
- Implementation code

### components

- **irqdesc** (`irqdesc.c`): per-Linux-IRQ-number `struct irq_desc` allocation + lookup; either flat `irq_desc[NR_IRQS]` array (CONFIG_SPARSE_IRQ=n) or radix-tree (CONFIG_SPARSE_IRQ=y, default for x86_64); per-IRQ percpu enable count
- **irqdomain** (`irqdomain.c`): hierarchical irq-domain mapping HW-IRQ token → Linux IRQ number; per-domain `struct irq_domain_ops`; legacy / linear / tree / hierarchy modes; IRQ-allocation cascade (e.g., PCI-MSI-domain → IRQ-remap-domain → x86-vector-domain → matrix allocator)
- **chip** (`chip.c`): generic flow handlers (`handle_simple_irq`, `handle_level_irq`, `handle_edge_irq`, `handle_fasteoi_irq`, `handle_percpu_irq`, `handle_percpu_devid_irq`, `handle_nested_irq`, `handle_untracked_irq`); `irq_set_chip_and_handler`, `irq_chip_*` helpers
- **handle** (`handle.c`): `generic_handle_irq` / `__handle_irq_event_percpu` core dispatch; per-IRQ handler list traversal
- **manage** (`manage.c`): `request_irq` / `request_threaded_irq` / `free_irq` / `setup_irq` / `enable_irq` / `disable_irq` / `disable_irq_nosync` / `synchronize_irq` / `irq_set_affinity` / `irq_set_affinity_hint` / threaded-IRQ kthread management (`irq_thread`)
- **affinity** (`affinity.c`): generic IRQ-affinity calculation (cpumask spread per node, per-class), used by drivers via `pci_alloc_irq_vectors_affinity`
- **migration** (`migration.c`): per-IRQ migration on `set_affinity` (handle pending-IRQ + retrigger)
- **cpuhotplug** (`cpuhotplug.c`): IRQ migration off going-down CPU (cpu hotplug callback)
- **matrix** (`matrix.c`): Linux interrupt matrix allocator — per-CPU vector reservation pool used by x86 vector domain + APIC reserved-vector mgmt
- **msi** (`msi.c`): generic MSI domain framework + `msi_domain_alloc_irqs_descs_locked` + per-device msi_desc list management; consumed by PCI-MSI + platform-MSI + IMS (Interrupt Message Store)
- **resend** (`resend.c`): software-resend of edge-IRQs that were pending while masked; per-cpu tasklet-driven resend
- **autoprobe** (`autoprobe.c`): legacy ISA IRQ autoprobing (`probe_irq_on` / `probe_irq_off`); rarely used on modern x86_64 but kept for legacy ISA driver compat
- **dummychip** (`dummychip.c`): `no_irq_chip` + `dummy_irq_chip` (used by perf-IRQs / virtual IRQs / irq_sim)
- **generic-chip** (`generic-chip.c`): per-controller scaffolding for SoC drivers that share register layout (mask/unmask/ack via offset tables) — mostly ARM
- **ipi** (`ipi.c`) + **ipi-mux** (`ipi-mux.c`): inter-processor interrupt + multiplexed IPI infrastructure
- **proc** (`proc.c`): `/proc/interrupts` + `/proc/irq/N/{smp_affinity, smp_affinity_hint, smp_affinity_list, effective_affinity, effective_affinity_list, spurious, node, ...}` sysfs/procfs surface
- **debugfs** (`debugfs.c`): `/sys/kernel/debug/irq/`
- **devres** (`devres.c`): `devm_request_irq` / `devm_request_threaded_irq` / `devm_free_irq` / `devm_request_any_context_irq` / `devm_irq_alloc_descs` (auto-released on driver unbind via drivers/base devres anchor)
- **pm** (`pm.c`): per-IRQ wakeup-source flag; suspend/resume IRQ disable/re-enable
- **kexec** (`kexec.c`): IRQ teardown on kexec
- **spurious** (`spurious.c`): spurious-IRQ detection + per-IRQ disable-after-100k-spurious
- **irq_sim** (`irq_sim.c`): simulated IRQ controller for tests + GPIO chips needing synthetic IRQs
- **irq_test** (`irq_test.c`): KUnit IRQ-allocator + irqdomain tests
- **softirq** (`kernel/softirq.c` — outside `kernel/irq/` directory but tightly coupled): per-CPU softirq dispatch (HI / TIMER / NET_TX / NET_RX / BLOCK / IRQ_POLL / TASKLET / SCHED / HRTIMER / RCU bottom halves), `__do_softirq`, `raise_softirq`, `ksoftirqd/N` per-cpu kthread, `local_bh_disable` / `local_bh_enable`, tasklet `tasklet_struct` queue (deprecated since BH workqueue but still in tree)

### scope

This Tier-2 governs:
- `/home/doll/linux-src/kernel/irq/` (~30 source files)
- `/home/doll/linux-src/kernel/softirq.c` (single file but conceptually part of IRQ subsystem)
- Public headers `include/linux/{interrupt, irq, irqdomain, irqdesc, irqchip, msi}.h`
- The arch-glue is in `arch/x86/kernel/irq*.c` + `arch/x86/kernel/apic/{vector,io_apic,msi}.c` (cross-ref `arch/x86/idt.md` + `arch/x86/cpu-init.md`)

Per-driver irqchip implementations under `drivers/irqchip/` (a separate Tier-2 wrapper future) are consumers of the framework defined here.

### compatibility contract — outline

### `request_irq` / `request_threaded_irq` / `free_irq` API

In-tree drivers consume via `<linux/interrupt.h>`. Source-compatible.

`int request_threaded_irq(unsigned int irq, irq_handler_t handler, irq_handler_t thread_fn, unsigned long flags, const char *name, void *dev)` — flags: `IRQF_TRIGGER_{RISING,FALLING,HIGH,LOW}`, `IRQF_SHARED`, `IRQF_PROBE_SHARED`, `IRQF_TIMER`, `IRQF_PERCPU`, `IRQF_NOBALANCING`, `IRQF_IRQPOLL`, `IRQF_ONESHOT`, `IRQF_NO_SUSPEND`, `IRQF_FORCE_RESUME`, `IRQF_NO_THREAD`, `IRQF_EARLY_RESUME`, `IRQF_COND_SUSPEND`, `IRQF_NO_AUTOEN`, `IRQF_NO_DEBUG`, `IRQF_COND_ONESHOT`. Behavior identical.

### `/proc/interrupts` format

Per-line: `<IRQ_NUM>: <count_cpu0> <count_cpu1> ... <chip_name> <hwirq> [<level/edge>] <action_name1>, <action_name2>, ...` plus header line `         CPU0       CPU1       CPU2       ...`. Trailing `NMI:`, `LOC:`, `SPU:`, `PMI:`, `IWI:`, `RTR:`, `RES:`, `CAL:`, `TLB:`, `TRM:`, `THR:`, `DFR:`, `MCE:`, `MCP:`, `HYP:`, `HRE:`, `ERR:`, `MIS:` per-cpu special counters byte-identical (consumed by `mpstat -I` + `irqbalance` + sosreport + sar).

### `/proc/irq/N/`

Per-IRQ:
- `smp_affinity` (writable hex bitmap) — set affinity via `echo` (e.g., `echo 7 > /proc/irq/24/smp_affinity` for CPUs 0-2)
- `smp_affinity_list` (writable list form, e.g., `0-2`)
- `effective_affinity` / `effective_affinity_list` (read-only — actual delivered affinity after irqchip/msi-domain reduction)
- `smp_affinity_hint` (driver hint for `irqbalance`)
- `node` (NUMA node)
- `spurious` (spurious-IRQ counter)
- `<irq_action_name>/` subdir (per-action affinity for shared IRQs)

Wire format byte-identical so `irqbalance` works unchanged.

### `/proc/irq/default_smp_affinity`

Writable mask defining the default affinity for newly created IRQs. Behavior identical.

### Per-controller `irq_chip` API

In-tree controller drivers (LAPIC, IOAPIC, IR, Intel-iommu IR, AMD-Vi IR, Hyper-V SynIC, etc.) consume via `<linux/irq.h>`. Source-compatible (per-irq_chip vtable layout preserved; `irq_chip_*` helpers source-stable).

### `irqdomain` API

`irq_domain_create_*`, `irq_domain_add_*`, `irq_create_mapping`, `irq_create_fwspec_mapping`, `irq_find_mapping`, `irq_dispose_mapping`. Source-compatible. Hierarchical: `irq_domain_alloc_irqs` cascades through parent domains.

### `msi_domain` framework

Used by PCI (msi_domain_alloc_irqs called from `pci_alloc_irq_vectors`), platform-MSI, IMS (Interrupt Message Store). Source-compatible.

### Threaded IRQs

`request_threaded_irq(.., thread_fn, IRQF_ONESHOT, ..)` creates a per-IRQ kthread `irq/N-<name>` running the deferred work. Identical behavior including realtime scheduling priority promotion (when `irq_set_thread_priority` called).

### Softirq + tasklet

`__do_softirq` invoked from IRQ-exit path (`irq_exit`); also from `ksoftirqd/N` when over-budget. Bottom-half order: HI → TIMER → NET_TX → NET_RX → BLOCK → IRQ_POLL → TASKLET → SCHED → HRTIMER → RCU. `raise_softirq` from any context. `local_bh_disable` / `local_bh_enable` API.

Tasklet API (`tasklet_init`, `tasklet_schedule`, `tasklet_kill`) preserved for in-tree compat (some drivers + net protocols still use; new code prefers workqueue).

### Wakeup IRQs

`irq_set_irq_wake(irq, on)` flags an IRQ as wakeup-source for system suspend/hibernate. Per-IRQ wakeup state preserved across resume. Cross-ref `drivers/base/pm-wakeup.md`.

### IPI

`smp_call_function_*` and direct `arch_send_call_function_*ipi` consume IPI infrastructure from this subsystem. Cross-ref `arch/x86/cpu-init.md` IPI vectors.

### IRQ matrix allocator

Per-CPU vector reservation matrix (256 vectors per cpu on x86_64, minus reserved system vectors). `irq_matrix_alloc` / `_free` / `_assign` / `_reserve_managed`. Wire identical so debugfs vector-dump consumers see the same layout.

### CPU hotplug

`migrate_one_irq` invoked on cpu-down path; pending IRQs re-routed to remaining online CPUs of the affinity-mask intersection. Affinity preserved across hotplug as much as possible (driver-set affinity hint honored after re-up).

### Spurious IRQ handling

After 100K spurious IRQs in 100ms, the IRQ is auto-disabled and a kernel CRIT log emitted. Threshold + behavior identical.

### kexec

On kexec boot, all IRQs are masked at the chip level + irq_chip_disable_*; the new kernel re-discovers via its own irqdomain build-up. Behavior identical.

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `kernel/irq/irqdesc.md` | `irqdesc.c`: per-Linux-IRQ-number desc allocation + radix-tree (sparse) lookup |
| `kernel/irq/irqdomain.md` | `irqdomain.c`: hierarchical irq-domain HW-IRQ → Linux-IRQ mapping |
| `kernel/irq/chip.md` | `chip.c`: generic flow handlers + irq_chip helpers |
| `kernel/irq/handle.md` | `handle.c`: `generic_handle_irq` core dispatch |
| `kernel/irq/manage.md` | `manage.c`: `request_irq` + `free_irq` + threaded IRQs + affinity setting |
| `kernel/irq/affinity.md` | `affinity.c`: generic IRQ-affinity spread calculation |
| `kernel/irq/migration.md` | `migration.c` + `cpuhotplug.c`: per-IRQ migration on set_affinity + cpu-hotplug |
| `kernel/irq/matrix.md` | `matrix.c`: per-CPU vector reservation pool |
| `kernel/irq/msi.md` | `msi.c`: generic MSI domain framework + msi_desc list |
| `kernel/irq/resend.md` | `resend.c`: software-resend of edge-IRQs |
| `kernel/irq/dummychip.md` | `dummychip.c`: no_irq_chip / dummy_irq_chip |
| `kernel/irq/generic-chip.md` | `generic-chip.c`: per-controller scaffolding (mostly ARM) |
| `kernel/irq/ipi.md` | `ipi.c` + `ipi-mux.c`: IPI infrastructure |
| `kernel/irq/proc.md` | `proc.c`: `/proc/interrupts` + `/proc/irq/N/` sysfs/procfs surface |
| `kernel/irq/debugfs.md` | `debugfs.c`: `/sys/kernel/debug/irq/` |
| `kernel/irq/devres.md` | `devres.c`: `devm_request_irq` family |
| `kernel/irq/pm.md` | `pm.c`: per-IRQ wakeup-source + suspend/resume |
| `kernel/irq/kexec.md` | `kexec.c`: IRQ teardown on kexec |
| `kernel/irq/spurious.md` | `spurious.c`: spurious-IRQ detection + auto-disable |
| `kernel/irq/irq-sim.md` | `irq_sim.c`: simulated IRQ controller for GPIO + tests |
| `kernel/irq/autoprobe.md` | `autoprobe.c`: legacy ISA IRQ autoprobing |
| `kernel/irq/softirq.md` | `kernel/softirq.c`: per-CPU softirq dispatch + ksoftirqd + bottom-half order |
| `kernel/irq/tasklet.md` | `kernel/softirq.c` (tasklet portion): legacy tasklet API |

### compatibility outline (top-level)

- REQ-O1: `request_irq` / `request_threaded_irq` / `free_irq` / `setup_irq` / `enable_irq` / `disable_irq` / `synchronize_irq` API source-compatible with in-tree drivers.
- REQ-O2: `/proc/interrupts` byte-identical format + per-cpu special counters (NMI/LOC/SPU/PMI/IWI/RTR/RES/CAL/TLB/TRM/THR/DFR/MCE/MCP/HYP/HRE/ERR/MIS).
- REQ-O3: `/proc/irq/N/{smp_affinity, smp_affinity_list, effective_affinity, effective_affinity_list, smp_affinity_hint, node, spurious}` + `default_smp_affinity` byte-identical (`irqbalance` runs unchanged).
- REQ-O4: All IRQF_* flags + irq trigger types + handler return values byte-identical.
- REQ-O5: `irq_chip` vtable + `irq_chip_*` helpers source-compatible for in-tree controller drivers.
- REQ-O6: `irqdomain` + `msi_domain` API source-compatible (PCI MSI / IR / IMS unchanged).
- REQ-O7: Threaded IRQ kthread named `irq/N-<name>` byte-identical (consumed by perf + ftrace + RT preempt-tools).
- REQ-O8: Softirq + tasklet API source-compatible; bottom-half ordering preserved.
- REQ-O9: IRQ wakeup-source flag preserved across system suspend/hibernate.
- REQ-O10: CPU hotplug IRQ migration preserves driver-set affinity hint as much as possible.
- REQ-O11: Spurious-IRQ auto-disable threshold (100K in 100ms) preserved.
- REQ-O12: TLA+ models declared at this Tier-2 (irq_matrix vector allocation, threaded-IRQ enable/disable race, irqdomain hierarchical-alloc, msi-desc list mgmt, softirq-pending bitmap consistency).
- REQ-O13: Verus/Creusot Layer-4 functional contracts on irq_desc refcount + per-IRQ enable/disable count + affinity-mask validity.
- REQ-O14: Hardening: row-1 features applied per `00-security-principles.md`, with IRQ-specific reinforcement (vector-allocator bounds checking, /proc/irq write CAP_SYS_NICE/CAP_SYS_ADMIN, MSI-X mask-byte enforcement).

### acceptance criteria (top-level)

- [ ] AC-O1: `cat /proc/interrupts` on reference machine matches upstream baseline format byte-identical (every line, every counter format). (covers REQ-O2)
- [ ] AC-O2: `irqbalance --debug` reads `/proc/irq/N/smp_affinity*` + writes new affinity; the IRQ migrates within 100ms. (covers REQ-O3)
- [ ] AC-O3: `request_irq` test driver attaches a handler; HW IRQ delivery counts increment correctly under sustained load. (covers REQ-O1, REQ-O4)
- [ ] AC-O4: Threaded-IRQ test: `request_threaded_irq(.., IRQF_ONESHOT, ..)` spawns `irq/N-<name>` kthread; `pgrep -a irq/` shows it. (covers REQ-O7)
- [ ] AC-O5: PCI NIC `pci_alloc_irq_vectors(pdev, 1, 64, PCI_IRQ_MSIX | PCI_IRQ_AFFINITY)` returns 64 vectors with NUMA-spread affinities. (covers REQ-O6)
- [ ] AC-O6: CPU-hotplug stress test: repeatedly `echo 0 > /sys/devices/system/cpu/cpu1/online` + `echo 1 > ...`; IRQs migrate cleanly, no stuck-IRQ in `/proc/interrupts`. (covers REQ-O10)
- [ ] AC-O7: Spurious-IRQ test (drive a buggy driver claiming a shared IRQ but always returning IRQ_NONE): IRQ auto-disabled after 100K spurious. (covers REQ-O11)
- [ ] AC-O8: NIC throughput under load with NET_RX softirq saturating CPU triggers ksoftirqd/N to run; throughput within 2% of upstream. (covers REQ-O8)
- [ ] AC-O9: System-suspend test: USB keyboard wakeup-IRQ wakes machine from S3. (covers REQ-O9)
- [ ] AC-O10: kselftest `tools/testing/selftests/irqchip/` passes. (covers REQ-O12)
- [ ] AC-O11: KUnit `kernel/irq/irq_test.c` passes (irqdomain alloc/free + matrix alloc/free). (covers REQ-O12)
- [ ] AC-O12: kexec boot: `kexec -e` to a new kernel with all IRQs masked correctly + new kernel boots normally + IRQs re-allocated. (covers REQ-O1, REQ-O5)

### verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/irq/matrix_alloc.tla` | `kernel/irq/matrix.md` (proves: per-CPU vector matrix allocator + concurrent reserve_managed + alloc + free never produces double-allocation of a vector or leaks a slot; managed-vector reservation is honored across CPU hotplug) |
| `models/irq/threaded_disable.tla` | `kernel/irq/manage.md` (proves: `disable_irq` + `synchronize_irq` + concurrent threaded-IRQ kthread execution; on disable_irq return, no in-flight handler is still executing; re-enable_irq + pending HW IRQ replays correctly) |
| `models/irq/irqdomain_hier.tla` | `kernel/irq/irqdomain.md` (proves: hierarchical irqdomain alloc cascade — child-domain alloc requests parent alloc, parent alloc requests grandparent, etc.; partial-failure rolls back cleanly with no leaked allocation in any layer) |
| `models/irq/msi_desc_list.tla` | `kernel/irq/msi.md` (proves: per-device msi_desc list mgmt under concurrent alloc + free + iteration; iteration sees a consistent snapshot; no use-after-free of msi_desc) |
| `models/irq/softirq_pending.tla` | `kernel/irq/softirq.md` (proves: per-CPU `local_softirq_pending` bitmap consistency under raise_softirq from IRQ + raise_softirq from process context + __do_softirq drain; no missed softirq, no over-counted softirq) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `kernel/irq/irqdesc.md` | `irq_to_desc(irq)` invariant: returned desc valid for irq number; per-irq enable count `desc->depth >= 0` always (no underflow); refcount integrity |
| `kernel/irq/manage.md` | `request_irq` post-condition: handler is installed AND irq is enabled (depth==0) on success; on failure, no partial state |
| `kernel/irq/matrix.md` | `irq_matrix_alloc` invariant: returned vector is in [matrix->alloc_start, matrix->alloc_end); no two concurrent allocations on the same CPU return the same vector |

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-irq_desc + per-irqdomain + per-msi_desc refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-controller `irq_chip` + `irq_domain_ops` + `msi_domain_ops` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-IRQ vector arithmetic + matrix bitmap operations + msi-desc count multiplications use checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-controller irq_chip + irq_domain_ops vtables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed irq_desc + msi_desc cleared (carries handler pointer + dev_id used by some shared-IRQ drivers via stale ptr) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | softirq dispatch table + bottom-half handler table read-only-after-init | § Mandatory |

### Row-2 (LSM-stackable) features for IRQ

LSM hooks called: file-LSM hooks on `/proc/irq/N/smp_affinity*` writes, `/proc/irq/default_smp_affinity` writes. SELinux + AppArmor + future GR-RBAC stackable LSM mediate.

GR-RBAC adds:
- Per-role disallow `/proc/irq/N/smp_affinity` write (denies `irqbalance` running unprivileged from steering interrupts).
- Per-role disallow `/proc/irq/default_smp_affinity` write.
- Per-role audit of every IRQ affinity change (which irq, which mask, which uid).

### IRQ-specific reinforcement

- **Vector-allocator bounds checking**: `irq_matrix_alloc` validates target-CPU is online + reserved-managed-vector count below per-cpu cap before allocating.
- **/proc/irq write requires CAP_SYS_NICE** for affinity changes (less restrictive than CAP_SYS_ADMIN — `irqbalance` can be granted just CAP_SYS_NICE) AND CAP_SYS_ADMIN for default_smp_affinity write.
- **MSI-X mask-byte enforcement**: when devices are NOT bound to a userspace driver (vfio-pci), MSI-X table mask-byte writes are mediated by the MSI domain to ensure the kernel-side mask matches the HW-side mask (defense against driver bugs causing mask-state desync that triggers spurious IRQ storms).
- **Spurious-IRQ auto-disable**: per-IRQ counter persists across set_affinity (don't reset on migration, defense against persistent spurious firing being masked by repeated affinity changes).
- **Threaded-IRQ kthread CPU pinning**: when IRQ has set affinity hint to a single CPU, the threaded-IRQ kthread is pinned there too (avoids cross-CPU cache-miss penalty + improves predictability for RT setups).
- **softirq budget**: per-`__do_softirq` invocation budget capped (default 10 iters, ~10ms wallclock); over-budget defers to `ksoftirqd/N` (defense against softirq-storm starving normal scheduler load).
- **kexec IRQ teardown**: every IRQ chip-mask + ack-pending invoked before jump (defense against new-kernel seeing pending unmasked IRQs from old-kernel state).

