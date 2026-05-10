# Tier-3: kernel/irq/irqdesc.c — per-Linux-IRQ-number descriptor allocation + sparse-IRQ radix-tree lookup

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/irq/00-overview.md
upstream-paths:
  - kernel/irq/irqdesc.c
  - kernel/irq/internals.h
  - include/linux/irqdesc.h
-->

## Summary

`struct irq_desc` is the per-Linux-IRQ-number control block — one per active Linux IRQ. This Tier-3 covers `kernel/irq/irqdesc.c` (~1100 lines): per-IRQ desc allocation/free, lookup via either flat `irq_desc[NR_IRQS]` array (CONFIG_SPARSE_IRQ=n, legacy) or radix-tree (CONFIG_SPARSE_IRQ=y, default for x86_64), per-CPU per-IRQ statistics arrays (`kstat_irqs_cpu`), `irq_to_desc(irq)` fast-path. Owned operations: `irq_alloc_descs`, `irq_free_descs`, `irq_to_desc`, `__irq_get_desc_lock` / `_unlock` (per-irq lock helpers), per-irq affinity-mask + per-irq pending-mask + per-irq effective-affinity-mask, kobject embedding for `/proc/irq/N/`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct irq_desc` | per-IRQ control block | `kernel::irq::IrqDesc` |
| `irq_to_desc(irq)` | lookup desc by Linux IRQ number | `IrqDesc::lookup` |
| `irq_alloc_descs(irq, from, cnt, node)` | alloc N consecutive IRQ numbers + descs | `IrqDesc::alloc_range` |
| `irq_alloc_desc(node)` | alloc one IRQ number | `IrqDesc::alloc` |
| `irq_alloc_desc_at(at, node)` | alloc specific IRQ number | `IrqDesc::alloc_at` |
| `irq_alloc_desc_from(from, node)` | alloc IRQ ≥ `from` | `IrqDesc::alloc_from` |
| `irq_free_descs(irq, cnt)` | free N consecutive IRQ descs | `IrqDesc::free_range` |
| `irq_free_desc(irq)` | free one IRQ desc | `IrqDesc::free` |
| `__irq_get_desc_lock(irq, &flags, ...)` / `__irq_put_desc_unlock(...)` | per-IRQ lock acquire/release | `IrqDesc::lock` (`MutexGuard`) |
| `irq_lock_sparse()` / `irq_unlock_sparse()` | sparse-IRQ tree lock | `SparseIrqTree::lock` |
| `irq_get_next_irq(offset)` | iterate active IRQs from offset | `IrqDesc::iter_from` |
| `for_each_active_irq(irq)` | macro: iterate every active IRQ | `IrqDesc::iter_active` |
| `kstat_incr_irq_this_cpu(irq)` | per-CPU stat counter increment | `IrqDesc::incr_stat_this_cpu` |
| `kstat_irqs_cpu(irq, cpu)` | per-CPU stat counter read | `IrqDesc::stat_cpu` |
| `kstat_irqs(irq)` | sum across all CPUs | `IrqDesc::stat_total` |
| `irq_set_chip_data(irq, data)` / `irq_get_chip_data` | per-irq_chip private storage | `IrqDesc::chip_data*` |
| `irq_set_handler_data(irq, data)` / `irq_get_handler_data` | per-handler private storage | `IrqDesc::handler_data*` |
| `arch_show_interrupts(p, prec)` | per-arch /proc/interrupts trailing section | `arch::show_interrupts` |
| `show_interrupts(p, v)` | /proc/interrupts seq_file render | `IrqDesc::show_interrupts` |

## Compatibility contract

REQ-1: `irq_to_desc(N)` returns the same descriptor pointer for the same Linux-IRQ-number across calls (until the irq is freed).

REQ-2: `for_each_active_irq` iteration order matches upstream (ascending IRQ number).

REQ-3: Per-CPU per-IRQ stat counters (`kstat_irqs_cpu`) byte-identical semantics — incremented exactly once per IRQ delivery on the handling CPU.

REQ-4: `/proc/interrupts` formatted output byte-identical (per-IRQ row + trailing per-CPU NMI/LOC/SPU/PMI/IWI/RTR/RES/CAL/TLB/TRM/THR/DFR/MCE/MCP/HYP/HRE/ERR/MIS counters).

REQ-5: Sparse-IRQ default-on (CONFIG_SPARSE_IRQ=y) — IRQ allocation lazily expands the radix-tree, supports very large IRQ-number spaces (millions of IRQs for SR-IOV-heavy systems with many MSI-X vectors).

REQ-6: Per-IRQ desc embedded kobject in `/proc/irq/N/` registered + cleaned up correctly across alloc/free.

REQ-7: Per-IRQ affinity_mask + effective_affinity_mask + pending_mask preserve their semantics across CPU hotplug (cross-ref `kernel/irq/cpuhotplug.md`).

## Acceptance Criteria

- [ ] AC-1: `cat /proc/interrupts` on reference x86_64 machine produces output byte-identical to upstream baseline (every row, every counter format).
- [ ] AC-2: `irq_alloc_descs(N=64)` returns 64 consecutive IRQ numbers; `irq_free_descs` releases all 64; subsequent re-alloc may reuse them.
- [ ] AC-3: SR-IOV stress: enable 256 VFs each consuming 16 MSI-X vectors → 4096 active IRQs; `for_each_active_irq` traverses correctly.
- [ ] AC-4: kstat_irqs_cpu matches the per-CPU IRQ delivery counts shown in `/proc/interrupts`.
- [ ] AC-5: Sparse-IRQ stress: alloc IRQ at numbers 1, 1000, 1_000_000 → all three present in `for_each_active_irq` iteration in ascending order.
- [ ] AC-6: kobject lifecycle test — alloc + use + free 1000 IRQs → no `/proc/irq/N` directory leaks.
- [ ] AC-7: KUnit `kernel/irq/irq_test.c` passes.

## Architecture

`IrqDesc` lives in `kernel::irq::IrqDesc` with:
- `desc.lock: SpinLock<()>` — per-IRQ lock
- `desc.chip: AtomicPtr<dyn IrqChip>` — current chip
- `desc.handler: AtomicPtr<IrqFlowHandler>` — flow handler (handle_simple_irq / handle_level_irq / handle_edge_irq / handle_fasteoi_irq / handle_percpu_irq / handle_percpu_devid_irq / handle_nested_irq)
- `desc.action: Mutex<Vec<Arc<IrqAction>>>` — handler list (multiple for IRQF_SHARED)
- `desc.kobj: KObject` — `/proc/irq/N/`
- `desc.affinity_mask: CpuMask` + `desc.effective_affinity_mask: CpuMask` + `desc.pending_mask: CpuMask`
- `desc.kstat_irqs: PerCpu<u64>` — per-CPU stat counter
- `desc.depth: i32` — disable-depth (incremented by `disable_irq`)
- `desc.istate: AtomicU32` — IRQS_PENDING / IRQS_REPLAY / IRQS_AUTODETECT / IRQS_POLL_INPROGRESS / etc. flags
- `desc.threads_active: AtomicU32` + `desc.threads_oneshot: AtomicU32` — for threaded-IRQ tracking
- `desc.parent_irq: u32` — for hierarchical irq-domain
- `desc.dev_id: AtomicPtr<()>` — per-handler dev_id

Storage backend:
- **Sparse-IRQ mode (default)**: `irq_descs: Mutex<RadixTree<u32, Arc<IrqDesc>>>`. Lookup is O(log N) but typically RCU-protected fast-path. Allocation via `irq_alloc_descs` reserves a range from a per-system bitmap allocator (`allocated_irqs: SpinLock<Bitmap>`) + populates the radix tree.
- **Flat mode (CONFIG_SPARSE_IRQ=n)**: `irq_descs: [Option<IrqDesc>; NR_IRQS]` — fast O(1) lookup but bounded.

`IrqDesc::lookup(irq)`:
1. RCU read-side critical section.
2. If sparse: `radix_tree.lookup(irq)`.
3. If flat: bounds-check + `irq_descs[irq].as_ref()`.
4. Returns `Option<Arc<IrqDesc>>` (refcount-bumped).

`IrqDesc::alloc_range(from, cnt, node)`:
1. Take `allocated_irqs` bitmap lock.
2. `bitmap.find_next_zero_area(from, cnt)` → first hole big enough.
3. Reserve range in bitmap.
4. For each new IRQ, allocate `IrqDesc` (per-NUMA node), embed kobject, register `/proc/irq/N/`.
5. Insert into radix-tree (sparse) or array slot (flat).
6. Return base IRQ number.

`IrqDesc::free_range(base, cnt)`:
1. For each IRQ in [base, base+cnt): unregister `/proc/irq/N/`, drop `IrqDesc` (refcount 1→0 → release).
2. Clear bitmap range.

Per-CPU stat counters: `desc.kstat_irqs[cpu].fetch_add(1, Relaxed)` from interrupt handler on the local CPU. Sum read via per-cpu loop in `kstat_irqs(irq)`.

`/proc/interrupts` rendering: seq_file iterator walks `for_each_active_irq`; per-row shows IRQ number, per-CPU stat counter columns, chip name, hwirq, flow type, action name(s). Trailing per-CPU summary section delegates to per-arch `arch_show_interrupts` (x86 prints LOC/SPU/PMI/IWI/RTR/RES/CAL/TLB/TRM/THR/DFR/MCE/MCP/HYP/HRE/ERR/MIS).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `irqdesc_no_uaf` | UAF | `IrqDesc::lookup` always returns either `Some(Arc)` (refcount-bumped) or `None`; never returns dangling reference. |
| `kstat_no_overflow` | OVERFLOW | per-CPU u64 stat counters; never observed wraparound in practice (2^64 / freq); saturating add as defense-in-depth. |
| `bitmap_no_oob` | OOB | `allocated_irqs` bitmap operations bounds-checked against MAX_IRQ_DESCS (sparse) or NR_IRQS (flat). |

### Layer 2: TLA+

(none for this Tier-3 specifically — covered by parent's `models/irq/threaded_disable.tla` + `models/irq/matrix_alloc.tla`.)

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `irq_to_desc(irq)` returns Some iff irq is in allocated range AND not freed since last alloc | `IrqDesc::lookup` |
| `for_each_active_irq` visits each active IRQ exactly once per iteration | `IrqDesc::iter_active` |
| per-IRQ `desc.depth >= 0` always (no underflow from disable/enable mismatch) | `IrqDesc::enable` / `IrqDesc::disable` |

### Layer 4: Verus/Creusot functional

`IrqDesc::alloc_range(from, cnt)` post-condition (Verus): returned base satisfies `base >= from` AND `[base, base+cnt)` all unallocated before, all allocated after; bitmap updated atomically; radix-tree contains all `cnt` new entries.

## Hardening

(Inherits row-1 features from `kernel/irq/00-overview.md` § Hardening.)

irqdesc-specific reinforcement:

- **Bitmap allocator bounds-checked** — every `bitmap.find_next_zero_area` + `bitmap.set` validated against MAX_IRQ_DESCS.
- **Radix-tree node alloc per-NUMA** — IRQ desc allocated on the requesting NUMA node for cache locality.
- **kobject lifetime** matched to `Arc<IrqDesc>` refcount — kobject_put on Arc drop guarantees `/proc/irq/N/` cleanup.
- **Per-IRQ chip_data + handler_data** opaque pointer ownership tracked via `PhantomData<dyn Any>` in Rust wrapper to prevent type confusion.
- **Per-IRQ depth saturation** — `disable_irq` on already-disabled IRQ saturates depth at u32::MAX (no overflow); reciprocal `enable_irq` on depth==0 logs WARN (defense against driver enable/disable mismatch).

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- IRQ flow handlers (covered in `kernel/irq/chip.md`)
- IRQ allocation/dispatch logic for `request_irq` (covered in `kernel/irq/manage.md`)
- Per-architecture `arch_show_interrupts` (x86 in `arch/x86/idt.md`)
- 32-bit-only paths
- Implementation code
