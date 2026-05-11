# Tier-3: drivers/pci/pcie/aer.c — PCIe Advanced Error Reporting

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/pci/00-overview.md
upstream-paths:
  - drivers/pci/pcie/aer.c (~1740 lines)
  - drivers/pci/pcie/err.c (pcie_do_recovery)
  - include/linux/aer.h
  - include/uapi/linux/pci_regs.h (PCI_ERR_*, PCI_EXP_AER_FLAGS, PCI_EXP_RTCTL_*)
-->

## Summary

The **AER (Advanced Error Reporting)** driver is the PCIe root-port / RCEC service that consumes ERR_COR / ERR_NONFATAL / ERR_FATAL messages reported by the PCIe error-source hierarchy. The Root Port raises an MSI / MSI-X interrupt whose top-half `aer_irq` reads `PCI_ERR_ROOT_STATUS` + `PCI_ERR_ROOT_ERR_SRC`, drains them into a per-RP `kfifo`, and wakes the threaded handler `aer_isr`. The thread, per `aer_err_source`, splits correctable vs uncorrectable, builds `struct aer_err_info`, walks the hierarchy via `find_source_device` / `pci_walk_bus` to identify the error-source device(s), then per-severity:
- Correctable: write-1-to-clear `PCI_ERR_COR_STATUS`, invoke driver `cor_error_detected` callback, log+statistics.
- Uncorrectable Non-Fatal (ERR_NONFATAL): `pcie_do_recovery(dev, pci_channel_io_normal, aer_root_reset)` — invokes per-driver `pci_error_handlers.error_detected`, optionally `mmio_enabled`, then `resume`.
- Uncorrectable Fatal (ERR_FATAL): `pcie_do_recovery(dev, pci_channel_io_frozen, aer_root_reset)` — drives link / hierarchy reset, then `slot_reset` + `resume`.

Out-of-band paths: `aer_recover_queue` (called from APEI/GHES NMI context) drains AER capability registers from a `ghes_estatus_pool`-backed entry and schedules `aer_recover_work_func` to invoke the same recovery path. EDR (Error Disconnect Recover) is ACPI-mediated and overlaps DPC; this driver coordinates via `pcie_aer_is_native` / `host->native_aer`. Critical for: Recovering production endpoints from transient PCIe errors, surfacing root-cause hardware faults to userspace via sysfs+tracepoints, gating ECRC policy.

This Tier-3 covers `drivers/pci/pcie/aer.c` (~1740 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct aer_err_source` | per-RP FIFO entry (root status + err src ID) | `AerErrSource` |
| `struct aer_err_info` | per-error transient record | `AerErrInfo` |
| `struct aer_info` | per-device counters + ratelimits | `AerInfo` |
| `struct aer_rpc` | per-root-port + kfifo | `AerRpc` |
| `struct aer_recover_entry` | per-APEI deferred record | `AerRecoverEntry` |
| `aer_irq()` | per-RP top-half | `Aer::irq` |
| `aer_isr()` / `aer_isr_one_error()` / `aer_isr_one_error_type()` | per-threaded bottom-half | `Aer::isr` / `Aer::isr_one_error` / `Aer::isr_one_error_type` |
| `aer_process_err_devices()` | per-hierarchy dispatch | `Aer::process_err_devices` |
| `find_source_device()` / `find_device_iter()` | per-source-walk | `Aer::find_source_device` |
| `is_error_source()` / `add_error_device()` | per-iter | `Aer::is_error_source` / `Aer::add_error_device` |
| `aer_get_device_error_info()` | per-cap-read | `Aer::get_device_error_info` |
| `aer_print_error()` / `aer_print_source()` / `__aer_print_error()` | per-emit | `Aer::print_error` / `print_source` / `print_error_inner` |
| `pci_aer_handle_error()` / `handle_error_source()` | per-severity dispatch | `Aer::handle_error` / `handle_error_source` |
| `pcie_do_recovery()` (from `err.c`) | per-recovery state-machine | `PciePortErr::do_recovery` |
| `aer_root_reset()` | per-reset callback | `Aer::root_reset` |
| `aer_recover_queue()` / `aer_recover_work_func()` | per-APEI/GHES path | `Aer::recover_queue` / `recover_work` |
| `aer_enable_rootport()` / `aer_disable_rootport()` | per-probe/remove | `Aer::enable_rootport` / `disable_rootport` |
| `aer_enable_irq()` / `aer_disable_irq()` | per-RP-CMD bit | `Aer::enable_irq` / `disable_irq` |
| `aer_probe()` / `aer_remove()` / `aer_suspend()` / `aer_resume()` | per-service-driver hooks | `Aer::probe` / `remove` / `suspend` / `resume` |
| `pci_aer_init()` / `pci_aer_exit()` | per-device init/exit | `Aer::dev_init` / `dev_exit` |
| `pci_save_aer_state()` / `pci_restore_aer_state()` | per-PM save/restore | `Aer::save_state` / `restore_state` |
| `pci_aer_clear_status()` / `pci_aer_raw_clear_status()` | per-clear-all | `Aer::clear_status` / `raw_clear_status` |
| `pci_aer_clear_nonfatal_status()` / `pci_aer_clear_fatal_status()` | per-severity-clear | `Aer::clear_nonfatal_status` / `clear_fatal_status` |
| `pci_aer_unmask_internal_errors()` | per-internal-unmask (CXL) | `Aer::unmask_internal_errors` |
| `pcie_set_ecrc_checking()` / `pcie_ecrc_get_policy()` | per-ECRC policy | `Aer::set_ecrc_checking` / `ecrc_get_policy` |
| `pcie_aer_is_native()` | per-native-vs-firmware | `Aer::is_native` |
| `pci_print_aer()` | per-AER-cap dump (APEI) | `Aer::print_aer` |
| `cper_severity_to_aer()` | per-CPER mapping | `Aer::cper_severity_to_aer` |
| `aer_ratelimit()` | per-severity-ratelimit | `Aer::ratelimit` |
| `pci_dev_aer_stats_incr()` / `pci_rootport_aer_stats_incr()` | per-counter inc | `Aer::dev_stats_incr` / `rp_stats_incr` |
| `aer_stats_attr_group` / `aer_attr_group` | per-sysfs | `Aer::sysfs_groups` |
| `aerdriver` (`struct pcie_port_service_driver`) | per-service registration | `Aer::driver` |
| `pcie_aer_init()` (init-call) | per-service-register | `Aer::module_init` |
| `pci_no_aer()` / `pci_aer_available()` | per-cmdline disable | `Aer::no_aer` / `available` |

## Compatibility contract

REQ-1: struct aer_err_source:
- status: per-`PCI_ERR_ROOT_STATUS` snapshot (`PCI_ERR_ROOT_COR_RCV` | `_MULTI_COR_RCV` | `_UNCOR_RCV` | `_MULTI_UNCOR_RCV` | `_FATAL_RCV` | `_FIRST_FATAL_RCV` | `_NONFATAL_MSG_RCV` | `_FATAL_MSG_RCV`).
- id: per-`PCI_ERR_ROOT_ERR_SRC` — packed `(ERR_FATAL_NONFATAL_ID << 16) | ERR_COR_ID`.

REQ-2: struct aer_err_info:
- dev[AER_MAX_MULTI_ERR_DEVICES]: per-error-source(s) (capped at MULTI_ERR_DEVICES).
- error_dev_num: per-populated dev count.
- id: per-source requester-ID (BDF).
- severity: AER_CORRECTABLE (=2) | AER_NONFATAL (=0) | AER_FATAL (=1).
- multi_error_valid: per-multi-bit-in-root-status.
- first_error: per-`PCI_ERR_CAP_FEP` (PCI_ERR_CAP.first-error-pointer).
- status: per-`PCI_ERR_{COR,UNCOR}_STATUS`.
- mask: per-`PCI_ERR_{COR,UNCOR}_MASK`.
- tlp_header_valid: per-TLP-header-log validity (after `tlp_header_logged`).
- tlp: per-`pcie_tlp_log` snapshot of `PCI_ERR_HEADER_LOG` (+ prefix log if FLIT).
- level: per-printk level ("KERN_WARNING" cor / "KERN_ERR" uncor).
- is_cxl: per-`pcie_is_cxl(dev)`.
- ratelimit_print[]: per-dev ratelimit gate.
- root_ratelimit_print: per-RP ratelimit gate.

REQ-3: struct aer_rpc:
- rpd: per-root-port `pci_dev *`.
- aer_fifo: per-RP kfifo of `struct aer_err_source` sized AER_ERROR_SOURCES_MAX (128).

REQ-4: pci_aer_init(dev):
- dev.aer_cap = pci_find_ext_capability(dev, PCI_EXT_CAP_ID_ERR).
- if !dev.aer_cap: return.
- dev.aer_info = kzalloc(struct aer_info).
- if !dev.aer_info: dev.aer_cap = 0; return.
- ratelimit_state_init(&aer_info.correctable_ratelimit, DEFAULT_RATELIMIT_INTERVAL, DEFAULT_RATELIMIT_BURST).
- ratelimit_state_init(&aer_info.nonfatal_ratelimit, DEFAULT_RATELIMIT_INTERVAL, DEFAULT_RATELIMIT_BURST).
- /* PM save buffer */
- n = pcie_cap_has_rtctl(dev) ? 5 : 4.
- pci_add_ext_cap_save_buffer(dev, PCI_EXT_CAP_ID_ERR, sizeof(u32) * n).
- pci_aer_clear_status(dev).
- if pci_aer_available(): pci_enable_pcie_error_reporting(dev) — sets `PCI_EXP_AER_FLAGS` in `PCI_EXP_DEVCTL`.
- pcie_set_ecrc_checking(dev).

REQ-5: pcie_aer_is_native(dev):
- host = pci_find_host_bridge(dev.bus).
- if !dev.aer_cap: return 0.
- return pcie_ports_native ∨ host.native_aer.

REQ-6: aer_irq(irq, context):
- pdev = (pcie_device *)context; rpc = get_service_data(pdev); rp = rpc.rpd; aer = rp.aer_cap.
- e_src = {}.
- pci_read_config_dword(rp, aer + PCI_ERR_ROOT_STATUS, &e_src.status).
- if !(e_src.status & AER_ERR_STATUS_MASK): return IRQ_NONE.
- pci_read_config_dword(rp, aer + PCI_ERR_ROOT_ERR_SRC, &e_src.id).
- pci_write_config_dword(rp, aer + PCI_ERR_ROOT_STATUS, e_src.status). /* W1C */
- if !kfifo_put(&rpc.aer_fifo, e_src): return IRQ_HANDLED.
- return IRQ_WAKE_THREAD.

REQ-7: aer_isr(irq, context) — threaded bottom-half:
- if kfifo_is_empty(&rpc.aer_fifo): return IRQ_NONE.
- while kfifo_get(&rpc.aer_fifo, &e_src):
  - aer_isr_one_error(rpc.rpd, &e_src).
- return IRQ_HANDLED.

REQ-8: aer_isr_one_error(root, e_src):
- pci_rootport_aer_stats_incr(root, e_src).
- /* Per-PCIe, correctable reported first */
- if e_src.status & PCI_ERR_ROOT_COR_RCV:
  - multi = e_src.status & PCI_ERR_ROOT_MULTI_COR_RCV.
  - e_info = { id: ERR_COR_ID(e_src.id), severity: AER_CORRECTABLE, level: KERN_WARNING, multi_error_valid: !!multi }.
  - aer_isr_one_error_type(root, &e_info).
- if e_src.status & PCI_ERR_ROOT_UNCOR_RCV:
  - fatal = e_src.status & PCI_ERR_ROOT_FATAL_RCV.
  - multi = e_src.status & PCI_ERR_ROOT_MULTI_UNCOR_RCV.
  - e_info = { id: ERR_UNCOR_ID(e_src.id), severity: fatal ? AER_FATAL : AER_NONFATAL, level: KERN_ERR, multi_error_valid: !!multi }.
  - aer_isr_one_error_type(root, &e_info).

REQ-9: aer_isr_one_error_type(root, info):
- found = find_source_device(root, info).
- if info.root_ratelimit_print ∨ (!found ∧ aer_ratelimit(root, info.severity)):
  - aer_print_source(root, info, found).
- if found: aer_process_err_devices(info).

REQ-10: find_source_device(parent, e_info):
- e_info.error_dev_num = 0.
- /* RP itself may be the agent */
- if find_device_iter(parent, e_info): return true.
- if pci_pcie_type(parent) == PCI_EXP_TYPE_RC_EC: pcie_walk_rcec(parent, find_device_iter, e_info).
- else: pci_walk_bus(parent.subordinate, find_device_iter, e_info).
- return e_info.error_dev_num > 0.

REQ-11: is_error_source(dev, e_info):
- /* BDF match attempt */
- if PCI_BUS_NUM(e_info.id) != 0 ∧ !(dev.bus.bus_flags & PCI_BUS_FLAGS_NO_AERSID):
  - if e_info.id == pci_dev_id(dev): return true.
  - if !e_info.multi_error_valid: return false.
- /* Fallback: probe AER capability */
- pcie_capability_read_word(dev, PCI_EXP_DEVCTL, &reg16).
- if !(reg16 & PCI_EXP_AER_FLAGS): return false.
- if !dev.aer_cap: return false.
- /* Severity-specific */
- if e_info.severity == AER_CORRECTABLE: read COR_STATUS / COR_MASK.
- else: read UNCOR_STATUS / UNCOR_MASK.
- return (status & ~mask) != 0.

REQ-12: add_error_device(e_info, dev):
- i = e_info.error_dev_num.
- if i ≥ AER_MAX_MULTI_ERR_DEVICES: return -ENOSPC.
- e_info.dev[i] = pci_dev_get(dev).
- e_info.error_dev_num++.
- if aer_ratelimit(dev, e_info.severity): e_info.ratelimit_print[i] = 1; e_info.root_ratelimit_print = 1.
- return 0.

REQ-13: aer_get_device_error_info(info, i):
- dev = info.dev[i]; aer = dev.aer_cap; type = pci_pcie_type(dev).
- info.status = 0; info.tlp_header_valid = 0; info.is_cxl = pcie_is_cxl(dev).
- if !aer: return 0.
- if info.severity == AER_CORRECTABLE:
  - read COR_STATUS into info.status, COR_MASK into info.mask.
  - if !(status & ~mask): return 0.
- else if type ∈ {ROOT_PORT, RC_EC, DOWNSTREAM} ∨ info.severity == AER_NONFATAL:
  - read UNCOR_STATUS into info.status, UNCOR_MASK into info.mask.
  - if !(status & ~mask): return 0.
  - read PCI_ERR_CAP into aercc; info.first_error = PCI_ERR_CAP_FEP(aercc).
  - if tlp_header_logged(info.status, aercc):
    - info.tlp_header_valid = 1.
    - pcie_read_tlp_log(dev, aer + PCI_ERR_HEADER_LOG, aer + PCI_ERR_PREFIX_LOG, aer_tlp_log_len(dev, aercc), aercc & PCI_ERR_CAP_TLP_LOG_FLIT, &info.tlp).
- return 1.

REQ-14: aer_process_err_devices(e_info):
- /* Two-pass: report-all first, then handle — so reset doesn't lose records */
- for i in 0..e_info.error_dev_num:
  - if aer_get_device_error_info(e_info, i): aer_print_error(e_info, i).
- for i in 0..e_info.error_dev_num:
  - if aer_get_device_error_info(e_info, i): handle_error_source(e_info.dev[i], e_info).

REQ-15: handle_error_source(dev, info):
- cxl_rch_handle_error(dev, info).
- pci_aer_handle_error(dev, info).
- pci_dev_put(dev).

REQ-16: pci_aer_handle_error(dev, info):
- if info.severity == AER_CORRECTABLE:
  - if dev.aer_cap: write status to COR_STATUS (W1C).
  - if pcie_aer_is_native(dev):
    - if dev.driver.err_handler.cor_error_detected: invoke.
    - pcie_clear_device_status(dev).
- else if info.severity == AER_NONFATAL: pcie_do_recovery(dev, pci_channel_io_normal, aer_root_reset).
- else if info.severity == AER_FATAL: pcie_do_recovery(dev, pci_channel_io_frozen, aer_root_reset).

REQ-17: struct pci_error_handlers callbacks invoked by pcie_do_recovery (drivers/pci/pcie/err.c):
- error_detected(dev, state) — state is `pci_channel_io_normal` (NONFATAL) or `pci_channel_io_frozen` (FATAL); returns `pci_ers_result_t`.
- mmio_enabled(dev) — invoked when all drivers returned `PCI_ERS_RESULT_CAN_RECOVER` and link is up.
- slot_reset(dev) — invoked after `aer_root_reset` for FATAL or after `PCI_ERS_RESULT_NEED_RESET`.
- resume(dev) — invoked on successful recovery.
- cor_error_detected(dev) — invoked directly from `pci_aer_handle_error` for correctable.
- Aggregated to per-tree `pci_ers_result_t` — DISCONNECT terminates.

REQ-18: aer_root_reset(dev):
- type = pci_pcie_type(dev).
- root = (type == RC_END) ? dev.rcec : pcie_find_root_port(dev).
- aer = root ? root.aer_cap : 0.
- if (host.native_aer ∨ pcie_ports_native) ∧ aer: aer_disable_irq(root).
- if type ∈ {RC_EC, RC_END}: rc = pcie_reset_flr(dev, PCI_RESET_DO_RESET).
- else: rc = pci_bus_error_reset(dev).
- if (host.native_aer ∨ pcie_ports_native) ∧ aer:
  - read+write PCI_ERR_ROOT_STATUS (clear).
  - aer_enable_irq(root).
- return rc ? PCI_ERS_RESULT_DISCONNECT : PCI_ERS_RESULT_RECOVERED.

REQ-19: aer_enable_rootport(rpc):
- /* Per-RP setup */
- clear PCI_EXP_DEVSTA.
- pcie_capability_clear_word(pdev, PCI_EXP_RTCTL, SYSTEM_ERROR_INTR_ON_MESG_MASK) /* disable SERR */.
- read PCI_ERR_ROOT_STATUS, write back (clear).
- if status & AER_ERR_STATUS_MASK:
  - if RC_EC: pcie_walk_rcec(pdev, clear_status_iter, NULL).
  - else if pdev.subordinate: pci_walk_bus(pdev.subordinate, clear_status_iter, NULL).
- clear COR_STATUS, UNCOR_STATUS.
- aer_enable_irq(pdev) — sets `ROOT_PORT_INTR_ON_MESG_MASK` (COR_EN | NONFATAL_EN | FATAL_EN) in `PCI_ERR_ROOT_COMMAND`.

REQ-20: aer_disable_rootport(rpc):
- aer_disable_irq(pdev) — clears `ROOT_PORT_INTR_ON_MESG_MASK` in `PCI_ERR_ROOT_COMMAND`.
- read+write PCI_ERR_ROOT_STATUS to clear pending bits.

REQ-21: aer_probe(dev) /* per-pcie_port_service_driver */:
- BUILD_BUG_ON if string arrays under-sized.
- if pci_pcie_type(port) ∉ {RC_EC, ROOT_PORT}: return -ENODEV.
- rpc = devm_kzalloc(struct aer_rpc); rpc.rpd = port; INIT_KFIFO(rpc.aer_fifo).
- set_service_data(dev, rpc).
- devm_request_threaded_irq(device, dev.irq, aer_irq, aer_isr, IRQF_SHARED, "aerdrv", dev).
- cxl_rch_enable_rcec(port).
- aer_enable_rootport(rpc).

REQ-22: ECRC policy (CONFIG_PCIE_ECRC):
- ECRC_POLICY_DEFAULT (BIOS-controlled), ECRC_POLICY_OFF, ECRC_POLICY_ON.
- pcie_set_ecrc_checking(dev): if !pcie_aer_is_native(dev): return. Per-policy enable/disable PCI_ERR_CAP_ECRC_{GENE,CHKE}.
- pcie_ecrc_get_policy(str): match against `ecrc_policy_str[]`.

REQ-23: APEI/GHES path (CONFIG_ACPI_APEI_PCIEAER):
- aer_recover_queue(domain, bus, devfn, severity, aer_regs):
  - entry = {bus, devfn, domain, severity, regs: aer_regs}.
  - kfifo_in_spinlocked(&aer_recover_ring, &entry, 1, &aer_recover_ring_lock).
  - schedule_work(&aer_recover_work) — runs `aer_recover_work_func`.
- aer_recover_work_func: drain ring; per-entry pci_get_domain_bus_and_slot; pci_print_aer; ghes_estatus_pool_region_free; pcie_do_recovery per-severity.

REQ-24: PM save/restore:
- pci_save_aer_state(dev): save UNCOR_MASK, UNCOR_SEVER, COR_MASK, PCI_ERR_CAP (+ ROOT_COMMAND if pcie_cap_has_rtctl).
- pci_restore_aer_state(dev): symmetric write-back.

REQ-25: AER capability registers (PCI Express r6.0 §7.8.4):
- PCI_ERR_UNCOR_STATUS / UNCOR_MASK / UNCOR_SEVER — uncorrectable error logs.
- PCI_ERR_COR_STATUS / COR_MASK — correctable error logs.
- PCI_ERR_CAP — capability + first error pointer + ECRC + TLP-prefix-log control + Comp-Time-Out-Log.
- PCI_ERR_HEADER_LOG (16 bytes) — TLP header at error capture.
- PCI_ERR_PREFIX_LOG — optional TLP prefix log (E2E / FLIT).
- PCI_ERR_ROOT_COMMAND — interrupt enables for ERR_COR / NONFATAL / FATAL.
- PCI_ERR_ROOT_STATUS — per-source bits (W1C).
- PCI_ERR_ROOT_ERR_SRC — packed COR / UNCOR Requester-IDs.

REQ-26: Sysfs:
- /sys/bus/pci/devices/<BDF>/aer_dev_correctable — per-COR-bit counters + TOTAL_ERR_COR.
- /sys/bus/pci/devices/<BDF>/aer_dev_fatal — per-UNCOR-bit counters + TOTAL_ERR_FATAL.
- /sys/bus/pci/devices/<BDF>/aer_dev_nonfatal — per-UNCOR-bit counters + TOTAL_ERR_NONFATAL.
- /sys/bus/pci/devices/<BDF>/aer_rootport_total_err_cor / _fatal / _nonfatal (RP+RC_EC only).
- /sys/bus/pci/devices/<BDF>/aer/correctable_ratelimit_interval_ms / burst (RW, CAP_SYS_ADMIN).
- /sys/bus/pci/devices/<BDF>/aer/nonfatal_ratelimit_interval_ms / burst (RW, CAP_SYS_ADMIN).

REQ-27: /proc/bus/pci visibility:
- AER capability bytes exposed through generic PCI config-space `/proc/bus/pci/<bus>/<devfn>` read (offset = `dev.aer_cap`); writes gated by CAP_SYS_ADMIN. The AER driver does not export a dedicated /proc node — `aer_attr_group` / `aer_stats_attr_group` are sysfs-only. Userspace tools (e.g. `setpci`) drive the cap directly.

REQ-28: Tracepoint:
- trace_aer_event(pci_name(dev), (status & ~mask), severity, tlp_header_valid, &tlp, bus_type) — emitted by `aer_print_error` and `pci_print_aer`.

REQ-29: EDR (Error Disconnect Recover) — ACPI:
- Defined in `drivers/pci/pcie/edr.c` (separate file but shares the AER `pci_error_handlers` contract).
- AER coordinates by honoring `pcie_aer_is_native`: if firmware owns AER, native ISR returns early at `pci_aer_handle_error` (does not invoke `cor_error_detected` / `pcie_clear_device_status`).
- EDR uses ACPI `_OST` to acknowledge; the `pci_error_handlers` callbacks (error_detected → slot_reset → resume) are the same.

REQ-30: pci_aer_unmask_internal_errors:
- read UNCOR_MASK, clear PCI_ERR_UNC_INTN, write back.
- read COR_MASK, clear PCI_ERR_COR_INTERNAL, write back.
- EXPORT_SYMBOL_FOR_MODULES — cxl_core only.

REQ-31: cper_severity_to_aer (CONFIG_ACPI_APEI_PCIEAER):
- CPER_SEV_RECOVERABLE → AER_NONFATAL.
- CPER_SEV_FATAL → AER_FATAL.
- default → AER_CORRECTABLE.

REQ-32: Ratelimit (per-severity):
- aer_ratelimit(dev, AER_NONFATAL) → __ratelimit(&aer_info.nonfatal_ratelimit).
- aer_ratelimit(dev, AER_CORRECTABLE) → __ratelimit(&aer_info.correctable_ratelimit).
- AER_FATAL → 1 (never ratelimited).

REQ-33: CONFIG_PCIE_AER kconfig boundary; pci_no_aer() / pci_aer_available() gates initialization.

## Acceptance Criteria

- [ ] AC-1: Root Port receives ERR_COR → `aer_irq` reads ROOT_STATUS, ROOT_ERR_SRC; W1C; kfifo_put; IRQ_WAKE_THREAD.
- [ ] AC-2: `aer_isr` drains kfifo; per `e_src` → `aer_isr_one_error` → correctable then uncorrectable branch.
- [ ] AC-3: `find_source_device` for RP with valid BDF id locates source via `pci_dev_id` match.
- [ ] AC-4: `find_source_device` for `PCI_BUS_FLAGS_NO_AERSID` falls back to scanning per-device AER_STATUS.
- [ ] AC-5: Correctable: COR_STATUS cleared (W1C); `cor_error_detected` invoked when present.
- [ ] AC-6: Uncorrectable Non-Fatal: `pcie_do_recovery(dev, pci_channel_io_normal, ...)` → `error_detected` returns `CAN_RECOVER` → `mmio_enabled` → `resume`.
- [ ] AC-7: Uncorrectable Fatal: `pcie_do_recovery(dev, pci_channel_io_frozen, ...)` → `error_detected` → `aer_root_reset` → `slot_reset` → `resume`.
- [ ] AC-8: `aer_root_reset` for RC_EC / RC_END uses `pcie_reset_flr`; else `pci_bus_error_reset`.
- [ ] AC-9: TLP header valid → `pcie_read_tlp_log` populates `info.tlp`; emitted by `pcie_print_tlp_log`.
- [ ] AC-10: `aer_recover_queue` (APEI path) enqueues entry; `aer_recover_work` invokes same recovery dispatch.
- [ ] AC-11: PM suspend (aer_suspend / pci_save_aer_state) saves UNCOR_MASK/SEVER + COR_MASK + ERR_CAP; resume restores; subsequent error correctly handled.
- [ ] AC-12: ratelimit: bursts of correctable above threshold log "ratelimited" not duplicate prints.
- [ ] AC-13: sysfs aer_dev_correctable shows per-bit + TOTAL counter; matches `aer_info.dev_cor_errs[]` + `dev_total_cor_errs`.
- [ ] AC-14: ECRC policy `on`: `PCI_ERR_CAP_ECRC_GENE | _CHKE` set when `_GENC | _CHKC` capability bits supported.
- [ ] AC-15: pci_aer_unmask_internal_errors: clears INTN/INTERNAL in masks.
- [ ] AC-16: pcie_aer_is_native(dev) returns false when firmware owns AER; native handler does not clear status or invoke recovery.

## Architecture

```
struct AerErrSource {
  status: u32,                             // PCI_ERR_ROOT_STATUS snapshot
  id: u32,                                 // PCI_ERR_ROOT_ERR_SRC packed
}

struct AerErrInfo {
  dev: [Option<*PciDev>; AER_MAX_MULTI_ERR_DEVICES],
  error_dev_num: u8,
  id: u16,
  severity: AerSeverity,                   // CORRECTABLE | NONFATAL | FATAL
  multi_error_valid: bool,
  first_error: u8,
  status: u32,
  mask: u32,
  tlp_header_valid: bool,
  tlp: PcieTlpLog,
  level: &'static str,                     // KERN_WARNING | KERN_ERR
  is_cxl: bool,
  ratelimit_print: [bool; AER_MAX_MULTI_ERR_DEVICES],
  root_ratelimit_print: bool,
}

struct AerInfo {                            // per-pci_dev->aer_info
  dev_cor_errs: [u64; AER_MAX_TYPEOF_COR_ERRS],
  dev_fatal_errs: [u64; AER_MAX_TYPEOF_UNCOR_ERRS],
  dev_nonfatal_errs: [u64; AER_MAX_TYPEOF_UNCOR_ERRS],
  dev_total_cor_errs: u64,
  dev_total_fatal_errs: u64,
  dev_total_nonfatal_errs: u64,
  rootport_total_cor_errs: u64,
  rootport_total_fatal_errs: u64,
  rootport_total_nonfatal_errs: u64,
  correctable_ratelimit: RatelimitState,
  nonfatal_ratelimit: RatelimitState,
}

struct AerRpc {                             // per-root-port pcie_device service-data
  rpd: *PciDev,
  aer_fifo: Kfifo<AerErrSource, AER_ERROR_SOURCES_MAX>,
}
```

`Aer::irq(irq, ctx) -> IrqReturn`:
1. rpc = get_service_data(ctx); rp = rpc.rpd; aer = rp.aer_cap.
2. pci_config_read32(rp, aer + PCI_ERR_ROOT_STATUS, &e_src.status).
3. if (e_src.status & AER_ERR_STATUS_MASK) == 0: return IRQ_NONE.
4. pci_config_read32(rp, aer + PCI_ERR_ROOT_ERR_SRC, &e_src.id).
5. pci_config_write32(rp, aer + PCI_ERR_ROOT_STATUS, e_src.status). /* W1C */
6. if !kfifo_put(&rpc.aer_fifo, e_src): return IRQ_HANDLED. /* dropped */
7. return IRQ_WAKE_THREAD.

`Aer::isr(irq, ctx) -> IrqReturn` (threaded):
1. if kfifo_empty(&rpc.aer_fifo): return IRQ_NONE.
2. while kfifo_get(&rpc.aer_fifo, &e_src): Aer::isr_one_error(rpc.rpd, &e_src).
3. return IRQ_HANDLED.

`Aer::isr_one_error(root, e_src)`:
1. Aer::rp_stats_incr(root, e_src).
2. /* Correctable first */
3. if e_src.status & PCI_ERR_ROOT_COR_RCV:
   - multi = e_src.status & PCI_ERR_ROOT_MULTI_COR_RCV.
   - info = AerErrInfo { id: ERR_COR_ID(e_src.id), severity: CORRECTABLE, level: KERN_WARNING, multi_error_valid: !!multi, ..default }.
   - Aer::isr_one_error_type(root, &info).
4. if e_src.status & PCI_ERR_ROOT_UNCOR_RCV:
   - fatal = e_src.status & PCI_ERR_ROOT_FATAL_RCV.
   - multi = e_src.status & PCI_ERR_ROOT_MULTI_UNCOR_RCV.
   - info = AerErrInfo { id: ERR_UNCOR_ID(e_src.id), severity: fatal ? FATAL : NONFATAL, level: KERN_ERR, multi_error_valid: !!multi, ..default }.
   - Aer::isr_one_error_type(root, &info).

`Aer::isr_one_error_type(root, info)`:
1. found = Aer::find_source_device(root, info).
2. if info.root_ratelimit_print ∨ (!found ∧ Aer::ratelimit(root, info.severity)):
   - Aer::print_source(root, info, found).
3. if found: Aer::process_err_devices(info).

`Aer::find_source_device(parent, info) -> bool`:
1. info.error_dev_num = 0.
2. if Aer::find_device_iter(parent, info): return true.
3. if pci_pcie_type(parent) == PCI_EXP_TYPE_RC_EC: pcie_walk_rcec(parent, find_device_iter, info).
4. else: pci_walk_bus(parent.subordinate, find_device_iter, info).
5. return info.error_dev_num > 0.

`Aer::is_error_source(dev, info) -> bool`:
1. if PCI_BUS_NUM(info.id) != 0 ∧ !(dev.bus.bus_flags & PCI_BUS_FLAGS_NO_AERSID):
   - if info.id == pci_dev_id(dev): return true.
   - if !info.multi_error_valid: return false.
2. pcie_capability_read_word(dev, PCI_EXP_DEVCTL, &reg16).
3. if !(reg16 & PCI_EXP_AER_FLAGS): return false.
4. if !dev.aer_cap: return false.
5. /* Read severity-specific status/mask */
6. read STATUS, MASK (COR or UNCOR by severity).
7. return (status & ~mask) != 0.

`Aer::get_device_error_info(info, i) -> bool`:
1. dev = info.dev[i]; aer = dev.aer_cap; type = pci_pcie_type(dev).
2. info.status = 0; info.tlp_header_valid = false; info.is_cxl = pcie_is_cxl(dev).
3. if !aer: return false.
4. if info.severity == CORRECTABLE: read COR_STATUS / COR_MASK; if !(status & ~mask): return false.
5. else if type ∈ {ROOT_PORT, RC_EC, DOWNSTREAM} ∨ info.severity == NONFATAL:
   - read UNCOR_STATUS / UNCOR_MASK; if !(status & ~mask): return false.
   - read PCI_ERR_CAP into aercc; info.first_error = PCI_ERR_CAP_FEP(aercc).
   - if tlp_header_logged(info.status, aercc):
     - info.tlp_header_valid = true.
     - pcie_read_tlp_log(dev, aer + PCI_ERR_HEADER_LOG, aer + PCI_ERR_PREFIX_LOG, aer_tlp_log_len(dev, aercc), aercc & PCI_ERR_CAP_TLP_LOG_FLIT, &info.tlp).
6. return true.

`Aer::process_err_devices(info)`:
1. /* Pass-1: log all */
2. for i in 0..info.error_dev_num:
   - if Aer::get_device_error_info(info, i): Aer::print_error(info, i).
3. /* Pass-2: handle / recover */
4. for i in 0..info.error_dev_num:
   - if Aer::get_device_error_info(info, i): Aer::handle_error_source(info.dev[i], info).

`Aer::handle_error(dev, info)`:
1. if info.severity == CORRECTABLE:
   - if dev.aer_cap: pci_config_write32(dev, aer + PCI_ERR_COR_STATUS, info.status).
   - if pcie_aer_is_native(dev):
     - if dev.driver?.err_handler?.cor_error_detected: invoke.
     - pcie_clear_device_status(dev).
2. else if info.severity == NONFATAL: PciePortErr::do_recovery(dev, pci_channel_io_normal, Aer::root_reset).
3. else /* FATAL */: PciePortErr::do_recovery(dev, pci_channel_io_frozen, Aer::root_reset).

`Aer::root_reset(dev) -> PciErsResult`:
1. type = pci_pcie_type(dev).
2. root = (type == RC_END) ? dev.rcec : pcie_find_root_port(dev).
3. aer = root ? root.aer_cap : 0.
4. host = pci_find_host_bridge(dev.bus).
5. if (host.native_aer ∨ pcie_ports_native) ∧ aer: Aer::disable_irq(root).
6. if type ∈ {RC_EC, RC_END}: rc = pcie_reset_flr(dev, PCI_RESET_DO_RESET).
7. else: rc = pci_bus_error_reset(dev).
8. if (host.native_aer ∨ pcie_ports_native) ∧ aer:
   - read+write PCI_ERR_ROOT_STATUS (clear).
   - Aer::enable_irq(root).
9. return rc ? DISCONNECT : RECOVERED.

`Aer::recover_queue(domain, bus, devfn, severity, regs)` (APEI):
1. entry = AerRecoverEntry { bus, devfn, domain, severity, regs }.
2. if kfifo_in_spinlocked(&aer_recover_ring, &entry, 1, &aer_recover_ring_lock):
   - schedule_work(&aer_recover_work).
3. else: pr_err("buffer overflow ...").

`Aer::recover_work_func()`:
1. while kfifo_get(&aer_recover_ring, &entry):
   - pdev = pci_get_domain_bus_and_slot(entry.domain, entry.bus, entry.devfn).
   - if !pdev: pr_err_ratelimited; continue.
   - Aer::print_aer(pdev, entry.severity, entry.regs).
   - ghes_estatus_pool_region_free(entry.regs, sizeof(aer_capability_regs)).
   - if NONFATAL: PciePortErr::do_recovery(pdev, pci_channel_io_normal, Aer::root_reset).
   - else if FATAL: PciePortErr::do_recovery(pdev, pci_channel_io_frozen, Aer::root_reset).
   - pci_dev_put(pdev).

`Aer::enable_rootport(rpc)`:
1. pcie_capability_read_word(pdev, PCI_EXP_DEVSTA, &reg16).
2. pcie_capability_write_word(pdev, PCI_EXP_DEVSTA, reg16). /* W1C */
3. pcie_capability_clear_word(pdev, PCI_EXP_RTCTL, SYSTEM_ERROR_INTR_ON_MESG_MASK).
4. read+write PCI_ERR_ROOT_STATUS (clear).
5. if reg32 & AER_ERR_STATUS_MASK:
   - if RC_EC: pcie_walk_rcec(pdev, clear_status_iter, NULL).
   - else if pdev.subordinate: pci_walk_bus(pdev.subordinate, clear_status_iter, NULL).
6. clear COR_STATUS, UNCOR_STATUS.
7. Aer::enable_irq(pdev). /* PCI_ERR_ROOT_COMMAND |= COR_EN | NONFATAL_EN | FATAL_EN */

`Aer::probe(dev) -> Result<()>`:
1. if pci_pcie_type(port) ∉ {RC_EC, ROOT_PORT}: return -ENODEV.
2. rpc = devm_kzalloc(struct AerRpc).
3. rpc.rpd = port; INIT_KFIFO(rpc.aer_fifo); set_service_data(dev, rpc).
4. devm_request_threaded_irq(device, dev.irq, Aer::irq, Aer::isr, IRQF_SHARED, "aerdrv", dev).
5. cxl_rch_enable_rcec(port).
6. Aer::enable_rootport(rpc).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `irq_w1c_root_status` | INVARIANT | per-aer_irq: PCI_ERR_ROOT_STATUS write equals snapshot status. |
| `aer_fifo_bound` | INVARIANT | per-aer_irq: kfifo at most AER_ERROR_SOURCES_MAX; on full, drop with IRQ_HANDLED. |
| `correctable_first` | INVARIANT | per-aer_isr_one_error: PCI_ERR_ROOT_COR_RCV processed before _UNCOR_RCV. |
| `error_dev_num_bound` | INVARIANT | per-add_error_device: error_dev_num ≤ AER_MAX_MULTI_ERR_DEVICES. |
| `pci_dev_get_put_balanced` | INVARIANT | per-add_error_device + handle_error_source: every get has a put. |
| `severity_dispatch` | INVARIANT | per-pci_aer_handle_error: COR→W1C+callback, NONFATAL→io_normal, FATAL→io_frozen. |
| `fatal_not_ratelimited` | INVARIANT | per-aer_ratelimit(FATAL) returns 1. |
| `native_gates_recovery` | INVARIANT | per-pcie_aer_is_native==false: no cor callback, no clear_device_status. |
| `tlp_header_only_when_logged` | INVARIANT | per-get_device_error_info: tlp_header_valid only set when tlp_header_logged true. |

### Layer 2: TLA+

`drivers/pci/aer.tla`:
- States: IRQ_FIRE → FIFO_PUT → THREAD_DRAIN → FIND_SOURCE → PROCESS → HANDLE → (COR_CLEAR | DO_RECOVERY).
- Properties:
  - `safety_no_lost_root_status` — per-aer_irq: W1C happens before kfifo_put returns IRQ_WAKE_THREAD.
  - `safety_correctable_then_uncorrectable` — per-aer_isr_one_error: ordering preserved.
  - `safety_dev_refcount_balanced` — per-add_error_device → pci_dev_put in handle_error_source.
  - `safety_two_pass_log_before_handle` — per-aer_process_err_devices: all aer_print_error before any handle_error_source.
  - `safety_fatal_drives_recovery` — per-FATAL: pcie_do_recovery with pci_channel_io_frozen.
  - `liveness_per_fifo_entry_eventually_handled` — per-kfifo_put: aer_isr eventually consumes it.
  - `liveness_recovery_terminates` — per-pcie_do_recovery: returns within bounded steps (driver callbacks).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Aer::irq` post: returns WAKE_THREAD ⟺ root status had ERR_STATUS_MASK ∧ kfifo had space | `Aer::irq` |
| `Aer::isr` post: kfifo drained at exit | `Aer::isr` |
| `Aer::find_source_device` post: error_dev_num ∈ [0, AER_MAX_MULTI_ERR_DEVICES] | `Aer::find_source_device` |
| `Aer::get_device_error_info` post: ret ⟹ (status & ~mask) != 0 | `Aer::get_device_error_info` |
| `Aer::handle_error` post: COR ⟹ COR_STATUS cleared | `Aer::handle_error` |
| `Aer::handle_error` post: FATAL ⟹ pcie_do_recovery(io_frozen) invoked | `Aer::handle_error` |
| `Aer::root_reset` post: returns DISCONNECT ⟺ pci_bus_error_reset failed | `Aer::root_reset` |
| `Aer::recover_queue` post: schedule_work iff kfifo_in succeeded | `Aer::recover_queue` |
| `Aer::enable_rootport` post: ROOT_COMMAND has COR_EN|NONFATAL_EN|FATAL_EN | `Aer::enable_rootport` |

### Layer 4: Verus/Creusot functional

`Per-AER-interrupt → root-status drain → kfifo bottom-half → find_source → process_err_devices (two-pass) → handle_error_source → severity dispatch (COR / NONFATAL / FATAL) → pcie_do_recovery + per-driver pci_error_handlers callbacks → aer_root_reset on FATAL → resume` semantic equivalence with `pcie_do_recovery` in `drivers/pci/pcie/err.c` and PCIe r6.0 §6.2.7. Spec source: Documentation/PCI/pci-error-recovery.rst, Documentation/PCI/pcieaer-howto.rst.

## Hardening

(Inherits row-1 features from `drivers/pci/00-overview.md` § Hardening.)

AER reinforcement:

- **Per-PCI_ERR_ROOT_STATUS W1C-before-kfifo_put** — defense against per-stale-bit refire after kfifo overflow.
- **Per-`aer_fifo` bounded (AER_ERROR_SOURCES_MAX = 128) with drop-on-full** — defense against per-storm DoS.
- **Per-AER_MAX_MULTI_ERR_DEVICES cap** — defense against per-walk allocating unbounded `dev[]` references.
- **Per-`pci_dev_get` paired with `pci_dev_put`** — defense against per-source UAF on driver-unbind during recovery.
- **Per-severity ratelimit (cor / nonfatal); FATAL never ratelimited** — defense against per-log-flood on cor-storm; per-loss of fatal signal.
- **Per-two-pass log-before-handle in `aer_process_err_devices`** — defense against per-record-loss when handler triggers reset before logging.
- **Per-`pcie_aer_is_native` gate on COR callback + `pcie_clear_device_status`** — defense against per-firmware-owned-AER trampling.
- **Per-`PCI_BUS_FLAGS_NO_AERSID` fallback to per-device AER_STATUS scan** — defense against per-buggy-RP that loses Requester-ID.
- **Per-RP irq-disable around `aer_root_reset`** — defense against per-re-entrant ISR during reset.
- **Per-APEI ring (`aer_recover_ring`) writer-spinlock; single-reader work** — defense against per-NMI/GHES races.
- **Per-`ghes_estatus_pool_region_free` after consumption** — defense against per-pool exhaustion.
- **Per-sysfs ratelimit-set CAP_SYS_ADMIN** — defense against per-unprivileged interval=0 (log-flood enable).
- **Per-PM save/restore of AER caps** — defense against per-D3-resume loss of mask/severity/ECRC config.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- drivers/pci/pcie/err.c `pcie_do_recovery` state-machine (covered separately if expanded)
- drivers/pci/pcie/dpc.c Downstream Port Containment (covered separately if expanded)
- drivers/pci/pcie/edr.c EDR ACPI handshake (covered separately; AER coordinates via `pcie_aer_is_native`)
- drivers/pci/pcie/portdrv.c port-bus service infrastructure (covered separately)
- drivers/cxl/core/* CXL RAS hooks (covered separately)
- drivers/acpi/apei/ghes.c CPER decoding (covered separately)
- Implementation code
