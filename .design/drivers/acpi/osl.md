# Tier-3: drivers/acpi/osl.c — OS Services Layer for ACPICA (memory map, IRQ, mutex/sem, sleep, I/O, override)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/acpi/00-overview.md
upstream-paths:
  - drivers/acpi/osl.c
  - drivers/acpi/osi.c
  - drivers/acpi/tables.c
  - drivers/acpi/internal.h
  - include/acpi/acpiosxf.h
  - include/acpi/platform/acenvex.h
-->

## Summary

`drivers/acpi/osl.c` is the OS Services Layer — the C ABI that ACPICA (third-party BSD-licensed AML core in `drivers/acpi/acpica/`) calls back into. ACPICA expects a host-provided implementation of ~60 `acpi_os_*` callbacks: memory map / unmap, physical I/O (port + memory), virtual-to-physical translation, IRQ install / remove, mutex + semaphore + spinlock primitives, sleep / stall, workqueue, vprintf, OS identification, predefined-name override, table override, RSDP lookup, signal handling. `osl.c` plus the smaller `osi.c` (the `_OSI` allowlist) and `tables.c` (early per-signature table parser registry) form the firewall between ACPICA's internal C world and Linux's kernel services.

This Tier-3 fixes every callback contract, the locking, the lockdown gates on table override / RSDP override / DSDT override, the per-IRQ install path (SCI), the per-memory-map cache that supports interrupt-context lookups, and the workqueue topology (`kacpid_wq`, `kacpi_notify_wq`, `kacpi_hotplug_wq`). The OSL boundary is documented as the **only** surface where Rookery talks to ACPICA — ACPICA does not call into Rust directly.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `acpi_os_get_root_pointer()` | locate the RSDP | `Osl::get_root_pointer` |
| `acpi_os_map_iomem(phys, size)` / `acpi_os_unmap_iomem(virt, size)` | refcounted ioremap cache | `Osl::map_iomem` / `unmap_iomem` |
| `acpi_os_map_memory` / `acpi_os_unmap_memory` (cast wrappers) | non-iomem map alias | `Osl::map_memory` / `unmap_memory` |
| `acpi_os_get_iomem(phys, size)` | per-phys existing-mapping lookup | `Osl::get_iomem` |
| `acpi_map_vaddr_lookup(phys, size)` (RCU) | virt-from-phys cache lookup, RCU-safe (interrupt context) | `Osl::map_vaddr_lookup` |
| `acpi_os_read_port(port, &val, width)` / `acpi_os_write_port(port, val, width)` | I/O port read/write | `Osl::port_*` |
| `acpi_os_read_memory(phys, &val, width)` / `acpi_os_write_memory(phys, val, width)` | physical memory read/write via ioremap | `Osl::mem_*` |
| `acpi_os_read_pci_configuration(pci_id, reg, &val, width)` / `acpi_os_write_pci_configuration(...)` | PCI config-space access | `Osl::pci_*` |
| `acpi_os_install_interrupt_handler(gsi, handler, ctx)` / `acpi_os_remove_interrupt_handler(gsi, handler)` | SCI install / remove | `Osl::irq_*` |
| `acpi_os_signal(function, info)` | ACPICA -> OSPM signal (BREAKPOINT, FATAL) | `Osl::signal` |
| `acpi_os_create_lock(&lock)` / `acpi_os_delete_lock(lock)` / `acpi_os_acquire_lock(lock)` / `acpi_os_release_lock(lock, flags)` | spinlock primitive | `Osl::lock_*` |
| `acpi_os_create_mutex(&mutex)` / `acpi_os_delete_mutex(mutex)` / `acpi_os_acquire_mutex(mutex, timeout)` / `acpi_os_release_mutex(mutex)` | mutex primitive | `Osl::mutex_*` |
| `acpi_os_create_semaphore(max_units, initial_units, &handle)` / `acpi_os_delete_semaphore(...)` / `acpi_os_wait_semaphore(handle, units, timeout)` / `acpi_os_signal_semaphore(...)` | semaphore primitive | `Osl::sem_*` |
| `acpi_os_sleep(ms)` / `acpi_os_stall(us)` / `acpi_os_get_timer()` | timing primitives | `Osl::sleep` / `stall` / `timer` |
| `acpi_os_execute(type, fn, ctx)` / `acpi_os_wait_events_complete()` | workqueue dispatch | `Osl::exec` / `wait_complete` |
| `acpi_os_predefined_override(init_val, &new_val)` | override `_OS_` / `_REV` | `Osl::predefined_override` |
| `acpi_os_table_override(existing_table, &new_table)` | override DSDT/SSDT (built-in) | `Osl::table_override` |
| `acpi_os_physical_table_override(existing_table, &phys, &length)` | override DSDT/SSDT (initramfs / efivar) | `Osl::physical_table_override` |
| `acpi_os_vprintf(fmt, args)` / `acpi_os_printf(fmt, ...)` | trace output | `Osl::printf` |
| `__acpi_os_prepare_sleep(state, pm1a, pm1b)` / `__acpi_os_prepare_extended_sleep(state, a, b)` | per-sleep transition pre-call | `Osl::prepare_sleep` |
| `acpi_os_initialize()` / `acpi_os_initialize1()` / `acpi_os_terminate()` | OSL init phases | `Osl::initialize_*` |

## Compatibility contract

REQ-1: The OSL is the **only** surface ACPICA may invoke; ACPICA never reaches into Rookery-Rust state directly. Every Rust-side state mutation that ACPICA needs to trigger goes through one of the documented `acpi_os_*` callbacks.

REQ-2: `acpi_os_get_root_pointer` returns the RSDP physical address. Sources in order: `acpi_rsdp` cmdline override (gated by `LOCKDOWN_ACPI_TABLES`), per-arch `acpi_arch_get_root_pointer`, `efi.acpi20` / `efi.acpi`, legacy memory scan (x86 only). Returns 0 on failure.

REQ-3: `acpi_os_map_iomem` maintains a refcounted ioremap cache (`acpi_ioremaps`) keyed by `(phys_addr, size)`; multiple callers with overlapping ranges share a single mapping. `acpi_os_unmap_iomem` drops the refcount and queues async RCU unmap via `acpi_os_map_remove` work.

REQ-4: `acpi_map_vaddr_lookup` is RCU-safe — callable from interrupt context — and walks the `acpi_ioremaps` list under RCU. Used by per-OpRegion handlers (`acpi_os_read_memory` / `acpi_os_write_memory`) when the AML interpreter must access SystemMemory from atomic context.

REQ-5: `acpi_os_install_interrupt_handler` accepts only the SCI GSI (`acpi_gbl_FADT.sci_interrupt`); any other GSI returns `AE_BAD_PARAMETER`. Installed with `IRQF_SHARED | IRQF_ONESHOT` via `request_threaded_irq`.

REQ-6: `acpi_os_read_port` / `acpi_os_write_port` use `inb` / `outb` / `inw` / `outw` / `inl` / `outl` per `width`; widths > 32 return `AE_BAD_PARAMETER`. When `CONFIG_HAS_IOPORT=n`, returns `AE_NOT_IMPLEMENTED` and reads return all-1s (mirrors hardware non-existence).

REQ-7: `acpi_os_read_pci_configuration` / `acpi_os_write_pci_configuration` route to `raw_pci_read` / `raw_pci_write` — bypassing the PCI device tree, because PCI device enumeration depends on ACPI being functional first.

REQ-8: `acpi_os_create_mutex` / `..._semaphore` allocate dynamic kernel objects (`struct mutex`, `struct semaphore`) tracked on per-ACPICA lists; deletion is mandatory at namespace teardown.

REQ-9: `acpi_os_execute(type, fn, ctx)` dispatches to one of three workqueues per type: `OSL_GLOBAL_LOCK_HANDLER` / `OSL_NOTIFY_HANDLER` -> `kacpi_notify_wq`, `OSL_GPE_HANDLER` / `OSL_EC_POLL_HANDLER` / `OSL_EC_BURST_HANDLER` -> `kacpid_wq`, `OSL_DEBUGGER_*` -> immediate. `kacpi_hotplug_wq` (cross-ref `bus.md`) is reserved for the bus layer's hotplug.

REQ-10: `acpi_os_predefined_override` may override `_OS_` (returns the per-cmdline `acpi_os_name` string) and `_REV` (returns 5 when `acpi_rev_override` is set). Override is `__init` only.

REQ-11: `acpi_os_table_override` and `acpi_os_physical_table_override` permit DSDT/SSDT override from built-in (`CONFIG_ACPI_CUSTOM_DSDT`) or initramfs (`acpi_table_upgrade`); both paths are gated by `LOCKDOWN_ACPI_TABLES` and audited.

REQ-12: `acpi_os_sleep(ms)` uses `usleep_range` with a 50us / 1% delta for coalescing; `acpi_os_stall(us)` is `udelay` with `touch_nmi_watchdog` on each iteration. `acpi_os_get_timer` returns 100ns-granularity monotonic counter derived from `get_jiffies_64`.

REQ-13: `__acpi_os_prepare_sleep` and `__acpi_os_prepare_extended_sleep` callbacks are settable by the architecture (x86 uses these for the SMM hand-off before S3/S0ix); per-callback default is `NULL` (no-op).

## Acceptance Criteria

- [ ] AC-1: ACPICA boots successfully on UEFI x86 with all OSL callbacks satisfied; `dmesg` shows `ACPI: SCI (IRQ%d) successfully installed`.
- [ ] AC-2: ACPI-ioremap cache test: instrument `acpi_os_map_iomem` and verify that 1000 sequential maps of the same range return the same virt with refcount 1000.
- [ ] AC-3: Interrupt-context map lookup test: an OpRegion handler called from a fixed-event ISR can call `acpi_map_vaddr_lookup` without sleeping.
- [ ] AC-4: SCI install test: `cat /proc/interrupts | grep acpi` shows the SCI line; faking an SCI delivers the registered handler.
- [ ] AC-5: PCI config-space read test: `acpi_os_read_pci_configuration` for a known device returns the same value `lspci -xx` reports.
- [ ] AC-6: Sleep/stall accuracy test: `acpi_os_sleep(100)` blocks for >=100ms; `acpi_os_stall(50)` blocks for >=50us.
- [ ] AC-7: Mutex / semaphore lifecycle test: create + acquire + release + delete works without leaks across 100k iterations.
- [ ] AC-8: Table override test: with `LOCKDOWN_NONE`, a cpio-prepended DSDT override is honoured; with `LOCKDOWN_ACPI_TABLES`, the override is refused with audit log.
- [ ] AC-9: `_OS_` override test: `acpi_os_name=` cmdline propagates to `_OS_` AML lookups.
- [ ] AC-10: Workqueue topology test: `acpi_os_execute` for each type lands on the correct workqueue (`kacpid_wq` vs `kacpi_notify_wq`).
- [ ] AC-11: kselftest + `fwts osl` clean.

## Architecture

`Osl` lives in `drivers::acpi::Osl`:

```
struct Osl {
  ioremaps: List<Arc<AcpiIoremap>>,    // refcounted ioremap cache
  ioremap_lock: Mutex<()>,             // protects list mutate
  permanent_mmap: AtomicBool,
  irq: Mutex<SciState>,
  kacpid_wq: Arc<Workqueue>,
  kacpi_notify_wq: Arc<Workqueue>,
  prepare_sleep_cb: AtomicCell<Option<fn(u8, u32, u32) -> i32>>,
  prepare_extended_sleep_cb: AtomicCell<Option<fn(u8, u32, u32) -> i32>>,
  predefined_override: KCell<PredefinedOverride>,
}

struct AcpiIoremap {
  virt: *mut u8,                       // __iomem
  phys: PhysAddr,
  size: usize,
  track: IoremapTrack,                 // refcount or rcu_work
}

enum IoremapTrack {
  Active { refcount: AtomicUsize },
  Dying { rwork: RcuWork },
}

struct SciState {
  handler: Option<fn(*mut c_void) -> u32>,
  context: *mut c_void,
  gsi: u32,
  irq: u32,
}
```

Refcounted ioremap cache:
1. `acpi_os_map_iomem(phys, size)`:
   - Round phys/size to page boundaries.
   - If `!acpi_permanent_mmap` (very early boot): `__acpi_map_table(phys, size)` (fixmap).
   - Acquire `ioremap_lock`.
   - `acpi_map_lookup(phys, size)`: if found, bump refcount, return existing virt + offset.
   - Else: `kzalloc(map)`, `acpi_map(phys, size)` (`acpi_os_ioremap` on x86, `kmap` on RAM pages for x86, `ioremap` on arm64/riscv), append to `acpi_ioremaps` via `list_add_tail_rcu`.
2. `acpi_os_unmap_iomem(virt, size)`:
   - Acquire `ioremap_lock`.
   - `acpi_map_lookup_virt(virt, size)`.
   - `acpi_os_drop_map_ref(map)`: decrement refcount; on zero, `list_del_rcu` then `queue_rcu_work(system_percpu_wq, &map->track.rwork)` for deferred `acpi_unmap` + `kfree`.
3. RCU-safe lookup: `acpi_map_vaddr_lookup` (caller in `rcu_read_lock` or `ioremap_lock` held) returns `virt + (phys - map->phys)`.

SCI handler:
1. ACPICA calls `acpi_os_install_interrupt_handler(gsi=acpi_gbl_FADT.sci_interrupt, handler, ctx)` at `acpi_enable_subsystem`.
2. Reject if `acpi_irq_handler != NULL` (`AE_ALREADY_ACQUIRED`).
3. `acpi_gsi_to_irq(gsi, &irq)`: per-arch GSI -> Linux IRQ.
4. `request_threaded_irq(irq, NULL, acpi_irq, IRQF_SHARED|IRQF_ONESHOT, "acpi", acpi_irq)`.
5. The `acpi_irq` thunk calls the stored ACPICA handler; result -> `acpi_irq_handled++` (counter) or `acpi_irq_not_handled++`.

Workqueue dispatch:
1. `acpi_os_execute(type, fn, ctx)`:
   - Pick wq per type (notify -> `kacpi_notify_wq`, others -> `kacpid_wq`).
   - `dpc = kzalloc(struct acpi_os_dpc)`; `dpc->function = fn`; `dpc->context = ctx`; `INIT_WORK(&dpc->work, acpi_os_execute_deferred)`.
   - `queue_work(wq, &dpc->work)`.
2. Worker `acpi_os_execute_deferred`:
   - `dpc->function(dpc->context)`.
   - `kfree(dpc)`.

Table override:
1. ACPICA calls `acpi_os_table_override(existing, &new)` or `acpi_os_physical_table_override(existing, &phys, &length)`.
2. Check `security_locked_down(LOCKDOWN_ACPI_TABLES)`; if locked down, return `*new = NULL` (no override).
3. Check built-in override path (`CONFIG_ACPI_CUSTOM_DSDT`): match `existing->signature` against the embedded `AmlCode` symbol.
4. Check initramfs override (`acpi_table_upgrade` populated by `early_initramfs` walk): match against any "kernel/firmware/acpi/*.aml" entry whose signature matches.
5. Return matched override; ACPICA reloads the table.
6. Audit log every override decision.

Predefined override:
1. ACPICA calls `acpi_os_predefined_override(init_val, &new_val)` with `init_val->name` in `_OS_` / `_REV` / etc.
2. If name is `_OS_` and `acpi_os_name[0] != 0`: `*new_val = acpi_os_name` (set by cmdline `acpi_os_name=`).
3. If name is `_REV` and `acpi_rev_override`: `*new_val = (char *)5` (per upstream — int 5 cast).

Sleep preparation:
1. Before S3/S0ix entry, ACPICA invokes the architecture's `__acpi_os_prepare_sleep(state, pm1a_ctrl, pm1b_ctrl)` callback (x86 uses this to hand off to SMM via SMI).
2. Default callback is NULL (no-op); per-arch sets via `acpi_os_set_prepare_sleep`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ioremap_cache_no_uaf` | UAF | every `AcpiIoremap` freed only after refcount drops to zero AND rcu grace expires. |
| `vaddr_lookup_rcu_safe` | UAF | `acpi_map_vaddr_lookup` does not deref a `Dying` entry; list traversal under RCU honours `list_del_rcu` semantics. |
| `sci_handler_singleton` | UNIQUENESS | `acpi_os_install_interrupt_handler` rejects double-install with `AE_ALREADY_ACQUIRED`. |
| `port_width_validated` | OOB | `acpi_os_read_port`/`write_port` width clamped to {8,16,32}; >32 returns `AE_BAD_PARAMETER`. |
| `mutex_sem_balanced` | LIVENESS | every `create_mutex`/`create_semaphore` has a matching `delete_*` at teardown; leak detector catches drift. |

### Layer 2: TLA+

`models/acpi/ioremap_refcount.tla` (this doc): models the refcounted ioremap cache + RCU async unmap and proves that concurrent map / unmap / lookup never observes a freed `acpi_ioremap`.

`models/acpi/workqueue_dispatch.tla` (this doc): models `acpi_os_execute` -> per-wq enqueue -> worker dispatch and proves type-correct workqueue selection across the documented type set.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `acpi_os_map_iomem` post: returned virt is in `acpi_ioremaps` list, refcount > 0, phys range covered | `Osl::map_iomem` |
| `acpi_os_unmap_iomem` post: per-virt entry refcount decremented; if zero, RCU work scheduled | `Osl::unmap_iomem` |
| `acpi_os_install_interrupt_handler` post: SCI line owned by `acpi_irq` thunk; `acpi_sci_irq` reflects Linux IRQ number | `Osl::irq_install` |
| `acpi_os_table_override` post: lockdown-denied path returns `*new = NULL` and audits the attempt | `Osl::table_override` |

### Layer 4: Verus/Creusot functional

ACPICA `AcpiOsMapMemory(phys, size)` -> `acpi_os_map_iomem` -> ioremap cache -> later `acpi_os_unmap_iomem` -> RCU-deferred `acpi_unmap` -> physical mapping torn down. Encoded as a refinement of the per-phys lifetime in the cache.

## Hardening

(Inherits row-1 features from `drivers/acpi/00-overview.md` § Hardening.)

OSL specific reinforcement:

- **RSDP cmdline override lockdown** — `acpi_rsdp=` cmdline only honoured when `security_locked_down(LOCKDOWN_ACPI_TABLES)` permits.
- **Per-ioremap-entry refcount saturating** — defense against refcount overflow forcing premature free.
- **RCU-deferred unmap** — defense against concurrent OpRegion handler dereferencing a torn-down mapping.
- **SCI single-handler invariant** — defense against double-install hijacking the system control interrupt.
- **Port width validation** — defense against widths that exceed hardware register decode.
- **PCI config-space ranges validated** — `pci_id` checked against known buses + devices; defense against AML probing PCI ranges that don't exist.
- **Workqueue topology fixed** — per-type wq mapping immutable; defense against type-confusion enqueueing AML on the wrong wq.
- **Table override audit + lockdown** — every DSDT/SSDT override decision logged via audit; refused under `LOCKDOWN_ACPI_TABLES`.
- **Predefined override `__init` only** — `_OS_` / `_REV` override frozen after `init`; defense against runtime ACPI behaviour swap.
- **Sleep/stall NMI-watchdog touch** — defense against long stall blocking NMI delivery.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `acpi_ioremap`, `acpi_os_dpc`, the per-create mutex / semaphore objects, and table-override staging buffers; every `copy_*_user` from these caches is sized by ACPICA's per-table length.
- **PAX_KERNEXEC** — `osl.c`, `osi.c`, `tables.c` core in `__ro_after_init` text; the per-callback function-pointer table that ACPICA stores during init is `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `acpi_os_map_iomem`, `acpi_os_execute_deferred` (the workqueue entry), the SCI handler thunk, and `acpi_os_install_interrupt_handler`.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-`acpi_ioremap` refcount, on per-SCI install count, on the predefined-override `acpi_os_name` reference; overflow trap kills the offender.
- **PAX_MEMORY_SANITIZE** — zero-on-free for ioremap-cache entries (which carry phys + size — sensitive topology), DPC structs, table-override staging buffers, and predefined-override strings.
- **PAX_UDEREF** — SMAP/PAN enforced on every sysfs / ioctl entry that lands in OSL (`ec_sys` ioctls, debugfs `acpi/`); reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — every ACPICA-callback function pointer (`acpi_irq_handler`, `prepare_sleep_cb`, predefined-override fn) marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms in OSL error paths behind CAP_SYSLOG; suppress `%p` of `acpi_ioremap.virt` and SCI handler in trace.
- **GRKERNSEC_DMESG** — restrict SCI install, ioremap cache pressure, table-override decision, and per-PCI-config audit messages to CAP_SYSLOG.
- **ACPI override-file CAP_SYS_RAWIO (lockdown)** — initramfs DSDT/SSDT override and `acpi_rsdp=` cmdline require CAP_SYS_RAWIO AND a non-locked-down kernel state (`!LOCKDOWN_ACPI_TABLES`).
- **DSDT / SSDT override gated** — both `acpi_os_table_override` (built-in) and `acpi_os_physical_table_override` (initramfs) refuse when lockdown >= integrity; audit logs every refused attempt with the offered table's signature + OEM-ID.
- **Port-io range allowlist** — `acpi_os_read_port` / `acpi_os_write_port` permitted only on ranges declared in the FADT's PM1a/PM1b/GPE0/GPE1 windows or on the per-EC ranges; arbitrary AML port access outside these ranges refused.
- **PCI config write audit** — every `acpi_os_write_pci_configuration` from AML logged via audit subsystem; defense against AML reprogramming device BARs at runtime.
- **`acpi_rsdp=` cmdline single-use** — RSDP override stashed in `acpi_arch_set_root_pointer` and cleared from the cmdline-readable area post-init so kexec inheritance is explicit.
- **Sleep callback registration audited** — `acpi_os_set_prepare_sleep` invocation logged; defense against runtime sleep-callback hijack.
- **OSL `_OSI` allowlist immutability** — `acpi_osi[]` array `__ro_after_init`; defense against runtime injection of a fake OS-identity claim.

Rationale: the OSL is where AML byte interpretation reaches out into the host kernel. A relaxed table-override gate lets a privileged userspace rewrite the AML behind every device driver; a broken ioremap cache UAF gives AML a stale virt that may now be a kernel page; an unbounded port range lets AML poke arbitrary I/O. CAP_SYS_RAWIO + lockdown on table override, audit on every PCI-config write, port-range allowlist, refcount-saturating ioremap cache, and kCFI on the per-callback function-pointer table turn the OSL into a structurally enforced firewall: AML may only reach the kernel through audited, capability-gated, allowlisted callbacks.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- ACPICA core (third-party BSD, imported)
- `_OSI` allowlist contents (covered in `00-overview.md`)
- Per-table early parser registry details (covered in `tables.md` future Tier-3)
- EC OSL callbacks (covered in `ec.md` future Tier-3)
- Per-arch sleep transition (covered in arch-specific sleep docs)
- Implementation code
