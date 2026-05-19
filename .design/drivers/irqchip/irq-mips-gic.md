# Tier-3: drivers/irqchip/irq-mips-gic.c — MIPS Global Interrupt Controller (shared + per-VPE local, EIC mode, cluster-aware)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/irqchip/00-overview.md
upstream-paths:
  - drivers/irqchip/irq-mips-gic.c
  - arch/mips/include/asm/mips-cps.h
  - arch/mips/include/asm/mips-gic.h
-->

## Summary

MIPS Global Interrupt Controller — the per-cluster MIPS-CPS (Coherent Processing System) interrupt aggregator used by MIPS Boston, Malta, Cavium Octeon III legacy, and Ingenic JZ4780-class SoCs. The hardware partitions interrupts into shared (system-wide, routed to a selectable VPE) and local (per-VPE: timer, perfcount, FDC, watchdog, SWInt0/1). Modern MIPS-CPS supports multiple clusters; each cluster has its own GIC instance accessible via redirect-block (CMGCR) MMIO indirection.

Each VPE (Virtual Processing Element — a hardware thread on a multi-threaded MIPS core) appears as a CPU to Linux. The GIC drives a single CPU pin (selected from EI0..EI5) when in compatibility mode, or generates 64 vectorized EIC (External Interrupt Controller) vectors when CPU is in EIC mode (`cpu_has_veic`).

This Tier-3 covers `irq-mips-gic.c` (~1000 lines: per-VPE init, shared-IRQ dispatch, local-IRQ dispatch, IPI domain, EIC binding, polarity/trigger config, cluster-redirect locking).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `gic_of_init(node, parent)` | DT init: discover base from `reg` or CM GCR_GIC_BASE, ioremap, create domain | `drivers::irqchip::mipsgic::Gic::of_init` |
| `gic_irq_domain` | single irq_domain spanning local (0..GIC_NUM_LOCAL_INTRS-1) + shared (GIC_NUM_LOCAL_INTRS..end) | `Gic::Domain` |
| `gic_irq_domain_map(d, virq, hwirq)` | per-hwirq install: choose local vs shared chip, set per-CPU vs system handler | `Gic::domain_map` |
| `gic_irq_domain_alloc(d, virq, nr_irqs, arg)` | hierarchical alloc path | `Gic::domain_alloc` |
| `gic_irq_domain_xlate(d, ctrlr, intspec, intsize, &hwirq, &type)` | DT 3-cell `<type hwirq trigger>` decode | `Gic::xlate` |
| `gic_handle_shared_int(chained)` | read GIC_SH_PEND mask, AND with pcpu_masks, dispatch via `generic_handle_domain_irq` | `Gic::handle_shared_int` |
| `gic_handle_local_int(chained)` | read GIC_VL_PEND, dispatch local IRQs | `Gic::handle_local_int` |
| `gic_irq_dispatch(desc) / __gic_irq_dispatch()` | chained vs EIC entry | `Gic::dispatch` |
| `gic_set_type(d, type)` | program GIC_SH_POL + GIC_SH_TRIG for level/edge | `Gic::set_type` |
| `gic_set_affinity(d, mask, force)` | rewrite per-shared-IRQ VPE map + pcpu_masks bit | `Gic::set_affinity` |
| `gic_local_irq_controller` | per-VPE local irq_chip | `Gic::LocalChip` |
| `gic_level_irq_controller` / `gic_edge_irq_controller` | per-shared-IRQ chips | `Gic::SharedChip` |
| `gic_send_ipi(d, cpu)` | write GIC_SH_WEDGE (or redir_wedge for remote cluster) | `Gic::send_ipi` |
| `gic_register_ipi_domain(node)` | hierarchical IPI domain on top of gic_irq_domain | `Gic::register_ipi_domain` |
| `gic_bind_eic_interrupt(irq, set)` | EIC mode: bind interrupt to a shadow register set | `Gic::bind_eic_interrupt` |
| `gic_cpu_startup` / `cpuhp_setup_state(CPUHP_AP_IRQ_MIPS_GIC_STARTING)` | per-VPE bringup callback | `Gic::cpu_startup` |
| `gic_get_c0_compare_int() / gic_get_c0_perfcount_int() / gic_get_c0_fdc_int()` | resolve C0_Compare/PerfCnt/FDC IRQs to gic-mapped virqs | `Gic::get_c0_*_int` |
| `gic_irq_lock_cluster(d) / mips_cm_unlock_other()` | cluster redirect-block lock for cross-cluster MMIO | `Gic::cluster_lock` |
| `IRQCHIP_DECLARE(mips_gic, "mti,gic", gic_of_init)` | DT registration | `Gic::compat` |

## Compatibility contract

REQ-1: Single DT node `compatible = "mti,gic"`; MMIO base either declared in `reg` or inherited from CM GCR_GIC_BASE (probed via `read_gcr_gic_base()`); length defaults to 0x20000.

REQ-2: GIC_CONFIG.NUMINTERRUPTS field encodes `(N+1)*8 - 1` shared interrupts; driver reads + computes `gic_shared_intrs`.

REQ-3: GIC_NUM_LOCAL_INTRS fixed at 7: Watchdog, Compare, GIC, FDC, PerfCount, SWInt0, SWInt1. Each routable iff GIC_VX_CTL bits indicate (compatibility mode); always routable in EIC mode.

REQ-4: Per-CPU vector selection — DT property `mti,reserved-cpu-vectors = <...>` lists C0 vectors NOT usable by GIC; SW0+SW1 always reserved; first free vector chosen as `cpu_vec`.

REQ-5: EIC mode (`cpu_has_veic`): GIC drives `__gic_irq_dispatch` via `set_vi_handler(cpu_vec, __gic_irq_dispatch)`; up to 64 EIC vectors with per-vector shadow register set assignable via `write_gic_vl_eic_shadow_set`.

REQ-6: Compatibility mode (no VEIC): GIC drives a single CPU C0 vector; `irq_set_chained_handler(MIPS_CPU_IRQ_BASE + cpu_vec, gic_irq_dispatch)` cascades into MIPS CPU IRQ.

REQ-7: Shared IRQ per-CPU steering: each shared IRQ has a target-VPE bitmap programmed via GIC_SH_MAP_*. Driver also maintains an SW `pcpu_masks[NR_CPUS][GIC_MAX_LONGS]` bitmap that AND's the pending vector before dispatch to avoid cross-VPE delivery races.

REQ-8: Multi-cluster support: when `mips_cps_multicluster_cpus()`, non-local cluster GIC accesses go through the CM redirect-block (`mips_cm_lock_other()` + `write_gic_redir_*` + `mips_cm_unlock_other()`); driver helpers `gic_irq_lock_cluster(d)` and `for_each_online_cpu_gic(cpu, gic_lock)` encapsulate the indirection.

REQ-9: Polarity (`GIC_POL_ACTIVE_HIGH`/`_LOW`) + trigger (`GIC_TRIG_LEVEL`/`_EDGE`) programmed per shared IRQ via `change_gic_pol(i, ...)` + `change_gic_trig(i, ...)`; default at init = active-high, level-triggered, masked.

REQ-10: IPIs allocated from a hierarchical IPI domain layered on top of `gic_irq_domain`; per-CPU pair of shared IRQs reserved via `ipi_resrv` bitmap; `gic_send_ipi` writes GIC_SH_WEDGE to fire.

REQ-11: C0_Compare timer (`mips_cpu_irq`) typically routed via GIC local IRQ 1; `gic_get_c0_compare_int()` returns the virq for `arch/mips/kernel/cevt-r4k.c` to request.

REQ-12: `cpuhp_setup_state(CPUHP_AP_IRQ_MIPS_GIC_STARTING, "irqchip/mips/gic:starting", gic_cpu_startup, NULL)` arms per-VPE init on hotplug.

## Acceptance Criteria

- [ ] AC-1: Boot MIPS Malta + GIC (qemu-system-mips64 -M malta) — dmesg shows `mti,gic` init with shared + local IRQ counts; `cat /proc/interrupts` shows per-CPU timer (GIC_LOCAL_INT_TIMER) + virtio peripherals.
- [ ] AC-2: SMP boot on 4-VPE Boston board — each VPE comes online; IPIs (`IPI_RESCHEDULE`, `IPI_CALL_FUNCTION`) deliver across VPEs.
- [ ] AC-3: EIC mode (CPU with VEIC): 64 EIC vectors available; perfcount IRQ uses dedicated vector with shadow register set 1.
- [ ] AC-4: Multi-cluster (2-cluster Boston): IRQ from a device on cluster 1 routed to a VPE on cluster 0 — `gic_irq_lock_cluster` engages redirect-block + write succeeds.
- [ ] AC-5: `/proc/irq/N/smp_affinity 0x08` reroutes a shared IRQ to VPE 3; observable in GIC_SH_MAP_* + `pcpu_masks` update.
- [ ] AC-6: CPU hotplug stress (offline/online VPE 1 100 cycles) — per-VPE local IRQ re-armed without leakage; no lockdep splat on `gic_lock`.
- [ ] AC-7: DT polarity write rejected post-init at runtime (e.g. via debugfs absence); only the init-fn programs polarity.

## Architecture

`Gic` (Rust analogue):

```
struct Gic {
  base:               NonNull<u8>,          // ioremap'd MMIO
  shared_intrs:       u32,                  // discovered from GIC_CONFIG
  cpu_pin:            u32,                  // 0 in EIC, else cpu_vec - GIC_CPU_PIN_OFFSET
  irq_domain:         Arc<IrqDomain>,
  ipi_domain:         Option<Arc<IrqDomain>>,
  per_cpu_masks:      PerCpu<[u64; GIC_MAX_LONGS / 64]>,
  ipi_resrv:          Bitmap<{GIC_MAX_INTRS}>,
  ipi_available:      Bitmap<{GIC_MAX_INTRS}>,
  all_vpes_chip_data: [AllVpesChipData; GIC_NUM_LOCAL_INTRS],
  lock:               RawSpinLock,
}
```

Init flow `gic_of_init`:
1. Walk `mti,reserved-cpu-vectors`; OR into `reserved` mask.
2. Find first free C0 vector via `find_first_zero_bit`.
3. Resolve MMIO base from `reg` cell or CM GCR_GIC_BASE; ioremap.
4. Read `GIC_CONFIG.NUMINTERRUPTS`; compute `gic_shared_intrs = (N+1)*8`.
5. If EIC: install `set_vi_handler(cpu_vec + GIC_PIN_TO_VEC_OFFSET, __gic_irq_dispatch)`; else `irq_set_chained_handler(MIPS_CPU_IRQ_BASE + cpu_vec, gic_irq_dispatch)`.
6. `irq_domain_create_simple(fwnode, GIC_NUM_LOCAL_INTRS + shared_intrs, 0, &gic_irq_domain_ops, NULL)`.
7. `gic_register_ipi_domain(node)` to layer IPI domain on top.
8. `board_bind_eic_interrupt = &gic_bind_eic_interrupt`.
9. For each cluster: initialise shared IRQs to ACTIVE_HIGH, LEVEL, masked.
10. `cpuhp_setup_state(CPUHP_AP_IRQ_MIPS_GIC_STARTING, ..., gic_cpu_startup, NULL)`.

Hot-path shared IRQ dispatch `gic_handle_shared_int(chained)`:
1. `pcpu_mask = this_cpu_ptr(pcpu_masks)`.
2. Read GIC_SH_PEND into local bitmap via `ioread64_copy` (64-bit) or `ioread32_copy` (32-bit).
3. AND pending with `pcpu_mask` → only this VPE's claimed IRQs.
4. For each set bit `intr`: `generic_handle_domain_irq(gic_irq_domain, GIC_SHARED_TO_HWIRQ(intr))`.

Hot-path local IRQ dispatch `gic_handle_local_int(chained)`:
1. Read `GIC_VL_PEND` (per-VPE register).
2. AND with `GIC_VL_MASK` to find unmasked pending.
3. For each set bit: `generic_handle_domain_irq(gic_irq_domain, GIC_LOCAL_TO_HWIRQ(intr))`.

`gic_set_affinity`:
1. Determine target VPE from `cpumask`.
2. Update GIC_SH_MAP_x register to point to the target VPE.
3. Update SW `pcpu_masks`: clear bit in old VPE, set in new VPE; transition window protected by gic_lock.
4. Update effective-affinity in `irq_data`.

EIC vector binding `gic_bind_eic_interrupt(irq, set)`: each EIC vector can use one of 4 shadow register sets — useful for fast-handler perf counters; `write_gic_vl_eic_shadow_set(irq, set)` programs the per-vector shadow-set index.

Cluster redirect: `gic_irq_lock_cluster(d)` checks `cpu_cluster(irq_target)` vs current cluster; if different, calls `mips_cm_lock_other(cl, 0, 0, CM_GCR_Cx_OTHER_BLOCK_GLOBAL)` and returns true so the caller uses `write_gic_redir_*` accessors; otherwise returns false for local accesses.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `shared_intr_in_range` | OOB | `intr < gic_shared_intrs` before GIC_SH_* register offset. |
| `pcpu_mask_consistent` | INVARIANT | `pcpu_masks[cpu]` bit set ↔ GIC_SH_MAP target-bit matches `cpu`. |
| `cluster_unlock_paired` | RAII | every `mips_cm_lock_other` paired with `mips_cm_unlock_other` on all return paths. |
| `eic_vec_in_range` | OOB | EIC vector index ≤ 63; refused otherwise. |

### Layer 2: TLA+

`models/irqchip/mips_gic_affinity.tla` (future): models concurrent `set_affinity` from CPU A while CPU B handles a pending interrupt; proves no double-dispatch and no lost-IRQ.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Gic::set_type` post: GIC_SH_POL + GIC_SH_TRIG bits match requested IRQ_TYPE_* | `Gic::set_type` |
| `Gic::send_ipi` post: GIC_SH_WEDGE written exactly once with hwirq | `Gic::send_ipi` |
| `Gic::cpu_startup` post: per-VPE GIC_VL_MASK = `gic_all_vpes_chip_data[i].mask` for each enabled local | `Gic::cpu_startup` |

### Layer 4: Verus/Creusot functional

Encoded: writing `pcpu_masks[cpu]` bit for shared IRQ `intr` after `gic_set_affinity` ensures the next `gic_handle_shared_int` on `cpu` will (and only will) dispatch `intr`. Relates affinity transition to delivered-on-cpu.

## Hardening

- All GIC_SH_* and GIC_VL_* offsets bounded by `gic_shared_intrs` / `GIC_NUM_LOCAL_INTRS`.
- Cluster-redirect MMIO accessed only via `mips_cm_lock_other`/`mips_cm_unlock_other` pairs; `for_each_online_cpu_gic` uses a `guard(raw_spinlock_irqsave)` to enforce.
- IRQ polarity + trigger programmed only at init (per-cluster loop in `gic_of_init`); no runtime API exposed to flip polarity.
- `ipi_resrv` + `ipi_available` bitmaps protect IPI reservation from over-allocation.
- DT `mti,reserved-cpu-vectors` parsed defensively; out-of-range vectors clamped to ST0_IM bit-count.
- Per-VPE `gic_cpu_startup` cpuhp callback re-arms local IRQs on hotplug to defeat stale-VPE-state.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `Gic` instance, per-CPU `pcpu_masks`, and IPI bitmap pages whitelisted; `/proc/interrupts` row copies strictly bounded.
- **PAX_KERNEXEC** — `gic_handle_shared_int`, `gic_handle_local_int`, `__gic_irq_dispatch`, and `gic_*_irq_controller` ops in W^X `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `gic_handle_*_int` and chained-handler entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `irq_domain` and IPI-allocation refs; overflow trap defeats double-free on cpu hot-unplug.
- **PAX_MEMORY_SANITIZE** — zero-on-free for the per-CPU mask area on CPU offline; defense against stale-mask-after-hotplug reuse.
- **PAX_UDEREF** — SMAP/PAN on every `/proc/irq/*/smp_affinity` write that flows into `gic_set_affinity`.
- **PAX_RAP / kCFI** — `gic_level_irq_controller`, `gic_edge_irq_controller`, `gic_local_irq_controller` callbacks (`.irq_mask`, `.irq_unmask`, `.irq_ack`, `.irq_eoi`, `.irq_set_type`, `.irq_set_affinity`), domain ops, and chained-handler trampolines marked kCFI-typed.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `mips_gic_base`, `gic_irq_domain`, `gic_all_vpes_chip_data[]`, and per-VPE register helpers behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict GIC init banners, cluster-skip warnings, and reserved-vector clamp messages to CAP_SYSLOG so attackers cannot enumerate the cluster/VPE topology.
- **Per-CPU local routes RO after init** — `gic_all_vpes_chip_data[]` populated during `gic_of_init`; locked `__ro_after_init` so per-local-IRQ routing/mask defaults cannot be silently changed.
- **EIC mode lockdown** — `board_bind_eic_interrupt` set once at init; refuse any post-init swap of the EIC binding hook; shadow-set assignments validated against vector-ID range.
- **Polarity register CAP_SYS_ADMIN** — no userland surface programs polarity post-boot; if a future debugfs entry is added it MUST be CAP_SYS_ADMIN gated and locked behind a build-time `EXPERT` toggle.
- **Cluster-redirect lock strict** — refuse `gic_irq_lock_cluster` followed by missing `mips_cm_unlock_other` via lockdep; ensure no GIC redirect block left enabled across IRQ-entry boundary (would deadlock NMIs).
- **IPI bitmap bounded** — IPI alloc bounded by `GIC_MAX_INTRS`; reject IPI alloc beyond `ipi_available`; defense against runaway alloc loops on misconfigured DT.

Rationale: the MIPS GIC drives every device IRQ on its target VPE and is the only IPI mechanism on MIPS-CPS — a corrupted per-VPE mask, a mis-released cluster-redirect lock, or a runtime polarity flip silently corrupts interrupt delivery topology-wide. kCFI on chip-ops, RO local-route table, strict cluster-lock RAII, and bounded IPI bitmaps turn the MIPS GIC into a structurally enforced delivery path on multi-cluster CPS systems.

## Open Questions

- (none at this Tier-3 level)

## Out of Scope

- MIPS R4K legacy CPU IRQ (covered in `arch/mips/` future Tier-3)
- MIPS CM (Coherence Manager) internals (covered in `arch/mips/cps.md` future Tier-3)
- Cavium Octeon legacy interrupt controller (separate driver)
- 32-bit-only PIC drivers (Atheros AR7240, etc.) used on older MIPS SoCs
- Implementation code
