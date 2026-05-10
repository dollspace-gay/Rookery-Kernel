# Tier-3: drivers/spi/spi.c — SPI subsystem core (controller + device + driver + transfer lifecycle)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/spi/00-overview.md
upstream-paths:
  - drivers/spi/spi.c (~5169 lines)
  - drivers/spi/internals.h
  - include/linux/spi/spi.h
  - include/linux/spi/spi_bitbang.h
  - include/uapi/linux/spi/spi.h
-->

## Summary

`drivers/spi/spi.c` is the **bus-agnostic core** of the SPI (Serial Peripheral Interface) subsystem. It implements the `spi` `bus_type` registered with the driver-model core, the `spi_controller_class` / `spi_target_class` classes, and the four owner objects: per-controller `struct spi_controller` (one per HW master/slave engine; allocated by LLD via `__spi_alloc_controller` / `spi_alloc_host` / `spi_alloc_target` and registered via `spi_register_controller`); per-device `struct spi_device` (one per chip-select target, allocated via `spi_alloc_device` and added via `spi_add_device`); per-driver `struct spi_driver` (registered via `__spi_register_driver`, matched by modalias / OF compatible / ACPI HID); per-message `struct spi_message` + per-transfer `struct spi_transfer` (submitted via `spi_async` / `spi_sync` / `spi_sync_locked` / `spi_pipe`-style `spi_write_then_read`). Per-async-submit goes through a kthread message pump (`spi_pump_messages` running on a `kthread_worker` created by `spi_init_queue`) which calls `spi_transfer_one_message` (the default) or driver's `transfer_one_message`. Per-chip-select dispatch is hot-path-optimized: `spi_set_cs` short-circuits when neither CS-line nor mode-bits changed, supports GPIO-CS via `spi_for_each_valid_cs` + `spi_toggle_csgpiod`, and honors per-spi-device `cs_setup` / `cs_hold` / `cs_inactive` `spi_delay`. Per-DMA-safe message: `spi_map_msg` allocates dummy TX/RX buffers when `SPI_CONTROLLER_MUST_TX/RX` is set (krealloc `GFP_KERNEL|GFP_DMA|__GFP_ZERO`), then `__spi_map_msg` builds sg_tables via `spi_map_buf` (supports `vmalloc_addr` / `kmap_to_page` / `virt_addr_valid` buffers; aligned by `dma_get_cache_alignment`). Per-mode (SPI_MODE_0..3 + SPI_LSB_FIRST + SPI_CS_HIGH + SPI_3WIRE + SPI_TX_DUAL/QUAD/OCTAL + SPI_RX_DUAL/QUAD/OCTAL + SPI_NO_TX/RX + SPI_LOOP + SPI_CS_WORD + SPI_MOSI_IDLE_LOW/HIGH) validated in `__spi_setup`. Per-OF / per-ACPI / per-board-info enumeration: `of_register_spi_devices` walks `for_each_available_child_of_node`; `acpi_register_spi_devices` walks via `acpi_walk_namespace`; legacy `spi_register_board_info` populates a `board_list` consumed by `spi_match_controller_to_boardinfo`. Critical for: SPI-NOR boot ROM read (BMC/IPMI firmware), SPI flash chip programming (M25Pxx / W25Q / Macronix / SST), SPI-NAND access, SPI-attached TPM2 / secure-element access, SPI sensors / ADCs / DACs / radios / displays, `/dev/spidev<bus>.<cs>` user-space chardev IOCTLs.

This Tier-3 covers `drivers/spi/spi.c` (~5169 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct spi_controller` | per-master/target controller (host or target) | `SpiController` |
| `struct spi_device` | per-chip-select target on a controller | `SpiDevice` |
| `struct spi_driver` | per-driver registration record | `SpiDriver` |
| `struct spi_message` | per-message (head of a list of transfers) | `SpiMessage` |
| `struct spi_transfer` | per-transfer (tx_buf/rx_buf + len + speed_hz + bits_per_word + cs_change + delays) | `SpiTransfer` |
| `struct spi_delay` | per-delay (value + unit USECS/NSECS/SCK) | `SpiDelay` |
| `struct spi_board_info` | per-legacy (pre-DT/ACPI) board info | `SpiBoardInfo` |
| `spi_bus_type` | per-`bus_type` | `SPI_BUS_TYPE` (static) |
| `spi_controller_class` / `spi_target_class` | per-class registration | `SPI_CONTROLLER_CLASS` / `SPI_TARGET_CLASS` |
| `__spi_alloc_controller()` / `spi_alloc_host()` / `spi_alloc_target()` | per-alloc + zeroed-LLD-private trailer | `SpiCore::alloc_controller` |
| `__devm_spi_alloc_controller()` | per-managed alloc | `SpiCore::devm_alloc_controller` |
| `spi_register_controller()` / `devm_spi_register_controller()` | per-register | `SpiCore::register_controller` / `devm_register_controller` |
| `spi_unregister_controller()` | per-unregister + per-child cleanup | `SpiCore::unregister_controller` |
| `spi_controller_suspend()` / `_resume()` | per-PM | `SpiCore::controller_suspend` / `resume` |
| `__spi_register_driver()` | per-driver register | `SpiCore::register_driver` |
| `spi_alloc_device()` | per-device alloc | `SpiCore::alloc_device` |
| `spi_add_device()` / `__spi_add_device()` | per-device add (CS validation + setup) | `SpiCore::add_device` |
| `spi_new_device()` | per-controller-driver explicit add | `SpiCore::new_device` |
| `spi_unregister_device()` | per-unreg | `SpiCore::unregister_device` |
| `spi_register_board_info()` | per-legacy-board-info register | `SpiCore::register_board_info` |
| `spi_setup()` / `__spi_setup()` | per-mode/bpw/speed validation + driver setup | `SpiCore::setup` |
| `spi_async()` / `__spi_async()` | per-async submit | `SpiCore::async` |
| `spi_sync()` / `__spi_sync()` / `spi_sync_locked()` | per-blocking submit (sync fast-path bypasses kthread) | `SpiCore::sync` / `sync_locked` |
| `spi_bus_lock()` / `spi_bus_unlock()` | per-exclusive-bus | `SpiCore::bus_lock` / `bus_unlock` |
| `spi_write_then_read()` | per-`spi_pipe` helper (uses static `buf` + `SPI_BUFSIZ` or kmalloc DMA buffer) | `SpiCore::write_then_read` |
| `spi_target_abort()` | per-target abort | `SpiCore::target_abort` |
| `spi_optimize_message()` / `spi_unoptimize_message()` / `__spi_optimize_message()` | per-reusable-message pre-optimize | `SpiCore::optimize_message` / `unoptimize_message` |
| `devm_spi_optimize_message()` | per-managed-optimize | `SpiCore::devm_optimize_message` |
| `spi_pump_messages()` / `__spi_pump_messages()` / `__spi_pump_transfer_message()` | per-kthread message-pump loop | `SpiCore::pump_messages` |
| `spi_init_queue()` / `spi_start_queue()` / `spi_stop_queue()` / `spi_destroy_queue()` / `spi_flush_queue()` | per-controller-queue lifecycle | `SpiCore::queue_*` |
| `spi_transfer_one_message()` | per-default per-message dispatch | `SpiCore::transfer_one_message` |
| `spi_queued_transfer()` / `__spi_queued_transfer()` | per-default `controller->transfer` for queued ctlrs | `SpiCore::queued_transfer` |
| `spi_set_cs()` / `spi_is_last_cs()` / `spi_toggle_csgpiod()` | per-chip-select dispatch | `SpiCore::set_cs` / `is_last_cs` / `toggle_csgpiod` |
| `spi_map_msg()` / `__spi_map_msg()` / `spi_map_buf()` / `spi_unmap_msg()` / `spi_unmap_buf()` | per-DMA mapping (sg_table build/teardown) | `SpiCore::map_msg` / `unmap_msg` / `map_buf` / `unmap_buf` |
| `spi_dma_sync_for_device()` / `_for_cpu()` | per-DMA sync | `SpiCore::dma_sync_for_device` / `_cpu` |
| `spi_transfer_wait()` | per-transfer-completion wait (with timeout) | `SpiCore::transfer_wait` |
| `spi_delay_to_ns()` / `spi_delay_exec()` / `spi_transfer_cs_change_delay_exec()` | per-delay execution (USECS/NSECS/SCK) | `SpiCore::delay_to_ns` / `delay_exec` / `transfer_cs_change_delay_exec` |
| `spi_finalize_current_message()` / `spi_finalize_current_transfer()` | per-LLD completion callbacks | `SpiCore::finalize_current_message` / `_transfer` |
| `spi_get_next_queued_message()` | per-peek-next-message | `SpiCore::get_next_queued_message` |
| `spi_split_transfers_maxsize()` / `_maxwords()` / `__spi_split_transfer_maxsize()` | per-transfer splitter | `SpiCore::split_transfers_*` |
| `spi_replace_transfers()` | per-replace-with-alternatives (delay-injection etc.) | `SpiCore::replace_transfers` |
| `__spi_validate()` / `__spi_validate_bits_per_word()` | per-validation | `SpiCore::validate` / `validate_bits_per_word` |
| `spi_take_timestamp_pre()` / `_post()` | per-PTP-timestamp | `SpiCore::take_timestamp_pre` / `_post` |
| `of_register_spi_devices()` / `of_register_spi_device()` / `of_spi_parse_dt()` / `of_spi_notify()` | per-OF enumeration + dynamic | `SpiCore::of_register_spi_devices` / `_device` / `parse_dt` / `notify` |
| `acpi_register_spi_devices()` / `acpi_register_spi_device()` / `acpi_spi_device_alloc()` / `acpi_spi_add_resource()` / `acpi_spi_parse_apple_properties()` / `acpi_spi_notify()` | per-ACPI enumeration + Apple-quirk + dynamic | `SpiCore::acpi_register_spi_*` |
| `of_find_spi_controller_by_node()` / `_device_by_node()` / `acpi_spi_find_controller_by_adev()` / `_device_by_adev()` | per-fwnode lookup | `SpiCore::of_find_*` / `acpi_find_*` |
| `spi_new_ancillary_device()` / `devm_spi_new_ancillary_device()` | per-ancillary CS (e.g. firmware-upload CS) | `SpiCore::new_ancillary_device` |
| `spi_get_gpio_descs()` | per-controller GPIO-CS descriptor table | `SpiCore::get_gpio_descs` |
| `spi_set_thread_rt()` | per-realtime kthread-priority | `SpiCore::set_thread_rt` |
| `spi_dev_get/put` / `spi_controller_get/put` / `spi_controller_set_devdata` | per-refcount + per-LLD-private | `SpiCore::*_get` / `*_put` / `*_set_devdata` |
| `postcore_initcall(spi_init)` | per-init: kmalloc `buf[SPI_BUFSIZ]` + bus_register + class_register + OF/ACPI notifier register | `SpiCore::init` |

## Compatibility contract

REQ-1: struct spi_controller (per-controller):
- `dev`: per-class `Device` (parent = SPI-engine HW device; class = `spi_controller_class` or `spi_target_class`).
- `bus_num`: per-`spi<N>` instance (idr-allocated via `spi_controller_idr`).
- `num_chipselect`: per-max-CS supported by the controller (1..SPI_DEVICE_CS_CNT_MAX).
- `num_data_lanes`: per-data-lanes (1 / 2 / 4 / 8).
- `target` / `slave`: per-target/host flag (mutually exclusive class).
- `mode_bits`: per-supported `SPI_*` mode bits.
- `buswidth_override_bits`: per-DT/ACPI-forced mode bits applied at `spi_alloc_device`.
- `bits_per_word_mask`: per-`SPI_BPW_MASK(n)` bitmap of allowed bpw.
- `min_speed_hz` / `max_speed_hz`: per-Hz bounds.
- `max_message_size` / `max_transfer_size` / `max_dma_len`: per-byte bounds (default `INT_MAX` for max_dma_len if 0).
- `flags`: `SPI_CONTROLLER_HALF_DUPLEX` / `_NO_RX` / `_NO_TX` / `_MUST_RX` / `_MUST_TX` / `_GPIO_SS` / `_SUSPENDED`.
- `dtr_caps`: per-DDR-transfer support (QSPI).
- `cs_gpiods`: per-CS GPIO descriptor table (allocated via `spi_get_gpio_descs` if `use_gpio_descriptors`).
- `last_cs[]` (SPI_DEVICE_CS_CNT_MAX): per-cached CS (SPI_INVALID_CS = -1 initially).
- `last_cs_mode_high` / `last_cs_index_mask`: per-cached CS-mode.
- `unused_native_cs` / `max_native_cs`: per-native-CS budget.
- `setup` / `cleanup` / `prepare_transfer_hardware` / `unprepare_transfer_hardware` / `prepare_message` / `unprepare_message` / `set_cs` / `set_cs_timing` / `transfer` / `transfer_one` / `transfer_one_message` / `handle_err` / `optimize_message` / `unoptimize_message` / `mem_ops`: per-driver-ops.
- `auto_runtime_pm` / `rt` / `must_async`: per-policy.
- `queue` / `queue_lock` / `queue_empty` / `running` / `busy` / `cur_msg` / `cur_msg_incomplete` / `cur_msg_need_completion` / `cur_msg_completion` / `xfer_completion` / `fallback`: per-queue state.
- `kworker` / `pump_messages`: per-kthread + per-kthread-work.
- `bus_lock_spinlock` / `bus_lock_mutex` / `bus_lock_flag` / `io_mutex` / `add_lock`: per-lock.
- `dummy_rx` / `dummy_tx`: per-`SPI_CONTROLLER_MUST_*` filler buffers (krealloc'd).
- `can_dma` / `dma_map_dev` / `dma_tx` / `dma_rx` / `cur_tx_dma_dev` / `cur_rx_dma_dev`: per-DMA.
- `pcpu_statistics`: per-`spi_statistics __percpu` (transfer / byte / message / error counters).
- `list`: per-`spi_controller_list` membership.

REQ-2: struct spi_device (per-CS target):
- `controller`: back-pointer.
- `dev`: per-bus `Device` (parent = controller.dev; bus = spi_bus_type).
- `max_speed_hz`: per-Hz target.
- `chip_select[]` (SPI_DEVICE_CS_CNT_MAX): per-physical-CS-numbers (s8; SPI_INVALID_CS = -1 unused).
- `num_chipselect`: per-logical-CS used (1..SPI_DEVICE_CS_CNT_MAX).
- `cs_index_mask`: per-logical-CS-bitmask (default BIT(0)).
- `mode`: per-`SPI_MODE_X_MASK` + LSB_FIRST + CS_HIGH + 3WIRE + LOOP + NO_CS + READY + TX_DUAL/QUAD/OCTAL + RX_DUAL/QUAD/OCTAL + RX_CPHA_FLIP + CS_WORD + MOSI_IDLE_LOW/HIGH + NO_TX/RX.
- `bits_per_word`: per-bpw (default 8 if zero).
- `cs_gpiod[]` (SPI_DEVICE_CS_CNT_MAX): per-CS GPIO descriptors.
- `cs_setup` / `cs_hold` / `cs_inactive`: per-`spi_delay`.
- `word_delay`: per-word `spi_delay`.
- `irq`: per-IRQ number (from OF / ACPI GPIO IRQ).
- `modalias[SPI_NAME_SIZE]`: per-driver-match string.
- `driver_override`: per-sysfs-override.
- `rt`: per-realtime requirement (boosts `controller.rt` if set).
- `pcpu_statistics`: per-spi_statistics __percpu.

REQ-3: struct spi_message:
- `transfers`: per-list-head of `struct spi_transfer.transfer_list`.
- `spi`: per-target `spi_device` (set on submit).
- `frame_length` / `actual_length`: per-byte tally.
- `status`: per-completion error (negative errno or 0).
- `complete`: per-callback invoked at finalize.
- `context`: per-callback-context.
- `queue`: per-controller-queue linkage.
- `state` / `opt_state`: per-driver scratch.
- `optimized` / `pre_optimized` / `prepared`: per-state flags.
- `is_dma_mapped` (deprecated): per-skip-mapping.

REQ-4: struct spi_transfer:
- `tx_buf` / `rx_buf`: per-DMA-safe pointers (NULL allowed if other set; SPI_CONTROLLER_MUST_* triggers dummy filler).
- `len`: per-byte length (must be multiple of `spi_bpw_to_bytes(bits_per_word)` — partial transfers rejected).
- `speed_hz` / `bits_per_word`: per-override (inherits from `spi_device` if zero).
- `tx_nbits` / `rx_nbits`: SPI_NBITS_SINGLE / _DUAL / _QUAD / _OCTAL.
- `dtr_mode`: per-DDR-mode (QSPI only; requires `controller.dtr_caps`).
- `cs_change`: per-transfer CS-toggle.
- `cs_change_delay`: per-CS-toggle `spi_delay`.
- `cs_off`: per-keep-CS-off-for-this-transfer.
- `delay`: per-end-of-transfer `spi_delay`.
- `word_delay`: per-word `spi_delay`.
- `transfer_list`: per-message linkage.
- `tx_sg` / `rx_sg`: per-DMA sg_table.
- `tx_sg_mapped` / `rx_sg_mapped`: per-DMA-mapped flags.
- `error`: per-driver error flags (e.g. SPI_TRANS_FAIL_NO_START).
- `effective_speed_hz`: per-actual-rate (driver fills).
- `ptp_sts` / `ptp_sts_word_pre` / `ptp_sts_word_post` / `timestamped`: per-PTP-timestamp.

REQ-5: spi_bus_type (per-bus-type):
- `.name = "spi"`.
- `.dev_groups = spi_dev_groups` (`modalias`, `driver_override`, statistics).
- `.match = spi_match_device` — match order: `driver_override` exact → ACPI HID → OF compatible (stripped of vendor prefix) → `id_table` entry → `driver.name` strcmp.
- `.uevent = spi_uevent` — emits `MODALIAS=spi:<name>`.
- `.probe = spi_probe` — calls `of_clk_set_defaults`, then `of_irq_get`/`acpi_dev_gpio_irq_get` for `spi.irq`, then `dev_pm_domain_attach(PD_FLAG_ATTACH_POWER_ON | PD_FLAG_DETACH_POWER_OFF)`, then `sdrv->probe`.
- `.remove = spi_remove` — calls `sdrv->remove`.
- `.shutdown = spi_shutdown` — calls `sdrv->shutdown`.

REQ-6: __spi_alloc_controller(dev, size, target):
- ctlr = kzalloc(size + ALIGN(sizeof(spi_controller), dma_get_cache_alignment), GFP_KERNEL).
- ctlr.pcpu_statistics = spi_alloc_pcpu_stats().
- device_initialize(&ctlr.dev).
- INIT_LIST_HEAD(&ctlr.queue).
- spin_lock_init(&ctlr.queue_lock), &ctlr.bus_lock_spinlock.
- mutex_init(&ctlr.bus_lock_mutex), &ctlr.io_mutex, &ctlr.add_lock.
- ctlr.bus_num = -1; ctlr.num_chipselect = 1; ctlr.num_data_lanes = 1.
- ctlr.target = target.
- ctlr.dev.class = target ? &spi_target_class : &spi_controller_class.
- ctlr.dev.parent = dev.
- device_set_node(&ctlr.dev, dev_fwnode(dev)).
- pm_suspend_ignore_children(&ctlr.dev, true).
- spi_controller_set_devdata(ctlr, (void*)ctlr + ctlr_size) /* LLD-private trailer */.

REQ-7: spi_register_controller(ctlr):
- if !ctlr.dev.parent: return -ENODEV.
- spi_controller_check_ops(ctlr) — require ≥1 of `transfer` / `transfer_one` / `transfer_one_message` (or mem_ops.exec_op).
- bus_num assignment:
  - if bus_num < 0: try `of_alias_get_id(ctlr.dev.of_node, "spi")`.
  - if bus_num ≥ 0: spi_controller_id_alloc(ctlr, bus_num, bus_num+1) — fixed.
  - if bus_num < 0: first_dynamic = of_alias_get_highest_id("spi") + 1 (or 0); spi_controller_id_alloc(ctlr, first_dynamic, 0).
- init_completion(&ctlr.xfer_completion), &ctlr.cur_msg_completion.
- if !ctlr.max_dma_len: ctlr.max_dma_len = INT_MAX.
- dev_set_name(&ctlr.dev, "spi%u", ctlr.bus_num).
- if !is_target ∧ ctlr.use_gpio_descriptors: spi_get_gpio_descs(ctlr); ctlr.mode_bits |= SPI_CS_HIGH.
- if !ctlr.num_chipselect: return -EINVAL.
- for idx in 0..SPI_DEVICE_CS_CNT_MAX: ctlr.last_cs[idx] = SPI_INVALID_CS.
- device_add(&ctlr.dev).
- if ctlr.transfer: pr_info "controller is unqueued, this is deprecated".
- else if ctlr.transfer_one ∨ ctlr.transfer_one_message: spi_controller_initialize_queue(ctlr):
  - ctlr.transfer = spi_queued_transfer.
  - if !ctlr.transfer_one_message: ctlr.transfer_one_message = spi_transfer_one_message.
  - spi_init_queue(ctlr) — kthread_run_worker + kthread_init_work(pump_messages, spi_pump_messages).
  - ctlr.queued = true.
  - spi_start_queue(ctlr) — ctlr.running = true; kthread_queue_work(pump_messages).
- mutex_lock(&board_lock); list_add_tail(&ctlr.list, &spi_controller_list); for each bi in board_list: spi_match_controller_to_boardinfo; unlock.
- of_register_spi_devices(ctlr) /* enumerate OF child nodes */.
- acpi_register_spi_devices(ctlr) /* enumerate ACPI _CRS SpiSerialBus resources */.
- Return 0.

REQ-8: spi_unregister_controller(ctlr):
- if CONFIG_SPI_DYNAMIC: mutex_lock(&ctlr.add_lock).
- device_for_each_child(&ctlr.dev, __unregister) /* drives spi_unregister_device per child */.
- found = idr_find(&spi_controller_idr, id) (under board_lock).
- if ctlr.queued: spi_destroy_queue(ctlr).
- list_del(&ctlr.list).
- device_del(&ctlr.dev).
- if found == ctlr: idr_remove(&spi_controller_idr, id).
- if CONFIG_SPI_DYNAMIC: mutex_unlock(&ctlr.add_lock).
- if !ctlr.devm_allocated: put_device(&ctlr.dev).

REQ-9: spi_controller_suspend / _resume:
- _suspend: if queued: spi_stop_queue(ctlr) (busy-poll 500 × 10ms until queue empty ∧ !busy). __spi_mark_suspended sets SPI_CONTROLLER_SUSPENDED under bus_lock_mutex.
- _resume: __spi_mark_resumed; if queued: spi_start_queue.

REQ-10: __spi_register_driver(owner, sdrv):
- sdrv.driver.owner = owner; sdrv.driver.bus = &spi_bus_type.
- if sdrv.driver.of_match_table: warn for each entry without a matching id_table modalias or driver.name.
- driver_register(&sdrv.driver).

REQ-11: spi_alloc_device(ctlr):
- spi_controller_get(ctlr) — refs the controller.
- spi = kzalloc(sizeof(*spi)).
- spi.pcpu_statistics = spi_alloc_pcpu_stats.
- spi.controller = ctlr.
- spi.dev.parent = &ctlr.dev; spi.dev.bus = &spi_bus_type; spi.dev.release = spidev_release.
- spi.mode = ctlr.buswidth_override_bits.
- spi.num_chipselect = 1.
- device_initialize(&spi.dev).

REQ-12: spi_add_device(spi) / __spi_add_device(spi, parent):
- if spi.num_chipselect > SPI_DEVICE_CS_CNT_MAX: -EOVERFLOW.
- for idx in 0..spi.num_chipselect: validate spi.chip_select[idx] < ctlr.num_chipselect.
- if !target: for idx: spi_dev_check_cs(spi, spi, idx, idx+1) — no duplicate physical CS within this device.
- for idx in spi.num_chipselect..MAX: spi.chip_select[idx] = SPI_INVALID_CS.
- spi_dev_set_name(spi):
  - ACPI: "spi-<acpi_dev_name>".
  - software_node: "spi-<%pfwP>".
  - else: "<dev_name(&ctlr.dev)>.<chip_select[0]>".
- bus_for_each_dev(&spi_bus_type, &check_info, spi_dev_check) — fail if any other device has same physical CS (except `parent` for ancillary).
- if CONFIG_SPI_DYNAMIC ∧ !device_is_registered(&ctlr.dev): return -ENODEV.
- if ctlr.cs_gpiods: spi.cs_gpiod[idx] = ctlr.cs_gpiods[chip_select[idx]].
- __spi_setup(spi, /*initial_setup=*/true) — validates mode bits, calls ctlr.setup, calls spi_set_cs_timing, primes spi_set_cs(spi, false, true).
- device_add(&spi.dev).
- on add error: spi_cleanup(spi).

REQ-13: spi_new_device(ctlr, chip):
- spi = spi_alloc_device(ctlr).
- spi_set_chipselect(spi, 0, chip.chip_select).
- spi.max_speed_hz = chip.max_speed_hz.
- spi.mode = chip.mode.
- spi.bits_per_word = chip.bits_per_word.
- spi.irq = chip.irq.
- strscpy(spi.modalias, chip.modalias, SPI_NAME_SIZE).
- spi.controller_data = chip.controller_data.
- if chip.swnode: device_set_node(&spi.dev, ...) .
- spi.cs_index_mask = BIT(0).
- spi_add_device(spi).

REQ-14: spi_unregister_device(spi):
- if !spi: return.
- device_remove_software_node(&spi.dev).
- device_del(&spi.dev).
- spi_cleanup(spi).
- put_device(&spi.dev).

REQ-15: __spi_setup(spi, initial_setup):
- /* Mode validation */
- reject if hweight(mode & (SPI_TX_DUAL | SPI_TX_QUAD | SPI_NO_TX)) > 1.
- reject if hweight(mode & (SPI_RX_DUAL | SPI_RX_QUAD | SPI_NO_RX)) > 1.
- if (mode & SPI_3WIRE) ∧ (mode & dual/quad/octal): -EINVAL.
- if (mode & SPI_MOSI_IDLE_LOW) ∧ (mode & SPI_MOSI_IDLE_HIGH): -EINVAL.
- bad_bits = mode & ~(ctlr.mode_bits | SPI_CS_WORD | SPI_NO_TX | SPI_NO_RX).
- ugly_bits = bad_bits & (TX/RX dual/quad/octal); if ugly: warn + clear ugly from mode + clear ugly from bad_bits.
- if bad_bits: err "unsupported mode bits" + -EINVAL.
- /* bpw validation */
- if !spi.bits_per_word: spi.bits_per_word = 8.
- else: __spi_validate_bits_per_word(ctlr, spi.bits_per_word) — checks ctlr.bits_per_word_mask if set.
- /* speed clamp */
- if ctlr.max_speed_hz ∧ (!spi.max_speed_hz ∨ spi.max_speed_hz > ctlr.max_speed_hz): spi.max_speed_hz = ctlr.max_speed_hz.
- mutex_lock(&ctlr.io_mutex).
- if ctlr.setup: ctlr.setup(spi) — on err: unlock + err_cleanup.
- spi_set_cs_timing(spi) — calls ctlr.set_cs_timing under pm_runtime if auto_runtime_pm.
- if auto_runtime_pm ∧ set_cs: pm_runtime_resume_and_get; spi_set_cs(spi, false, true); pm_runtime_put_autosuspend.
- else: spi_set_cs(spi, false, true).
- unlock.
- if spi.rt ∧ !ctlr.rt: ctlr.rt = true; spi_set_thread_rt(ctlr).
- on err_cleanup (initial_setup only): spi_cleanup(spi).

REQ-16: spi_set_cs(spi, enable, force):
- /* Hot-path short-circuit */
- if !force ∧ enable == spi_is_last_cs(spi) ∧ last_cs_index_mask matches ∧ last_cs_mode_high matches: return.
- ctlr.last_cs_index_mask = spi.cs_index_mask.
- for idx 0..SPI_DEVICE_CS_CNT_MAX: ctlr.last_cs[idx] = enable ∧ idx < spi.num_chipselect ? spi.chip_select[0] : SPI_INVALID_CS.
- ctlr.last_cs_mode_high = spi.mode & SPI_CS_HIGH.
- if last_cs_mode_high: enable = !enable /* invert for active-high */.
- /* CS-hold delay before deassert */
- if (csgpiod ∨ !ctlr.set_cs_timing) ∧ !activate: spi_delay_exec(&spi.cs_hold, NULL).
- /* Drive CS */
- if csgpiod: spi_for_each_valid_cs(spi, idx): spi_toggle_csgpiod(spi, idx, enable, activate); if SPI_CONTROLLER_GPIO_SS ∧ ctlr.set_cs: ctlr.set_cs(spi, !enable).
- else if ctlr.set_cs: ctlr.set_cs(spi, !enable).
- /* CS-setup or CS-inactive delay */
- if (csgpiod ∨ !ctlr.set_cs_timing): if activate: spi_delay_exec(&spi.cs_setup); else spi_delay_exec(&spi.cs_inactive).

REQ-17: __spi_validate(spi, msg):
- if list_empty(&msg.transfers): -EINVAL.
- msg.spi = spi.
- if (ctlr.flags & SPI_CONTROLLER_HALF_DUPLEX) ∨ (spi.mode & SPI_3WIRE):
  - for each xfer: reject if rx_buf ∧ tx_buf; reject if NO_TX ∧ tx_buf; reject if NO_RX ∧ rx_buf.
- msg.frame_length = 0.
- for each xfer:
  - xfer.effective_speed_hz = 0; msg.frame_length += xfer.len.
  - if !xfer.bits_per_word: = spi.bits_per_word.
  - if !xfer.speed_hz: = spi.max_speed_hz.
  - if ctlr.max_speed_hz ∧ xfer.speed_hz > ctlr.max_speed_hz: clamp.
  - __spi_validate_bits_per_word(ctlr, xfer.bits_per_word).
  - if xfer.dtr_mode ∧ !ctlr.dtr_caps: -EINVAL.
  - w_size = spi_bpw_to_bytes(xfer.bits_per_word).
  - if xfer.len % w_size: -EINVAL /* partial reject */.

REQ-18: spi_map_msg(ctlr, msg) (when SPI_CONTROLLER_MUST_TX/RX set ∧ !3WIRE):
- compute max_tx / max_rx across transfers lacking the matching buf.
- if max_tx: dummy_tx = krealloc(ctlr.dummy_tx, max_tx, GFP_KERNEL|GFP_DMA|__GFP_ZERO).
- if max_rx: dummy_rx = krealloc(ctlr.dummy_rx, max_rx, GFP_KERNEL|GFP_DMA).
- for each xfer with !xfer.len-continue: assign dummy fallback bufs.
- __spi_map_msg(ctlr, msg):
  - if !ctlr.can_dma: 0.
  - tx_dev = ctlr.dma_tx ? dma_tx.device.dev : ctlr.dma_map_dev ?: ctlr.dev.parent.
  - rx_dev = analogous.
  - for each xfer where ctlr.can_dma(ctlr, msg.spi, xfer):
    - if tx_buf: spi_map_buf_attrs(ctlr, tx_dev, &xfer.tx_sg, tx_buf, len, DMA_TO_DEVICE, DMA_ATTR_SKIP_CPU_SYNC); tx_sg_mapped = true.
    - if rx_buf: spi_map_buf_attrs(ctlr, rx_dev, &xfer.rx_sg, rx_buf, len, DMA_FROM_DEVICE, ...); rx_sg_mapped = true.
  - ctlr.cur_tx_dma_dev = tx_dev; ctlr.cur_rx_dma_dev = rx_dev.

REQ-19: spi_map_buf(ctlr, dev, sgt, buf, len, dir):
- max_seg_size = dma_get_max_seg_size(dev).
- if is_vmalloc_addr(buf) ∨ kmap_buf: desc_len = min(max_seg_size, PAGE_SIZE); sgs = DIV_ROUND_UP(len + offset_in_page(buf), desc_len).
- else if virt_addr_valid(buf): desc_len = min(max_seg_size, ctlr.max_dma_len); sgs = DIV_ROUND_UP(len, desc_len).
- else: -EINVAL.
- sg_alloc_table(sgt, sgs, GFP_KERNEL).
- per-sg: min = vmalloc/kmap ? min(desc_len, min(len, PAGE_SIZE - offset_in_page(buf))) : min(len, desc_len).
- vmalloc: vmalloc_to_page → sg_set_page; kmap: kmap_to_page → sg_set_page; else: sg_set_buf.
- dma_map_sgtable(dev, sgt, dir, attrs).

REQ-20: __spi_pump_messages(ctlr, in_kthread):
- mutex_lock(&ctlr.io_mutex).
- spin_lock_irqsave(&ctlr.queue_lock).
- if ctlr.cur_msg: out_unlock.
- if list_empty(&ctlr.queue) ∨ !ctlr.running:
  - if !ctlr.busy: out_unlock.
  - if !in_kthread:
    - if cheap-teardown-allowed (no dummy bufs ∧ no unprepare hw): spi_idle_runtime_pm; busy = false; queue_empty = true.
    - else: kthread_queue_work(ctlr.kworker, &ctlr.pump_messages) /* defer to kthread */.
    - out_unlock.
  - else /* in_kthread */: busy = false; unlock; kfree dummy_tx/rx; if unprepare_transfer_hardware: call it; spi_idle_runtime_pm; lock; queue_empty = true; out_unlock.
- msg = list_first_entry(&ctlr.queue, ...); ctlr.cur_msg = msg.
- list_del_init(&msg.queue).
- was_busy = ctlr.busy; ctlr.busy = true.
- unlock.
- ret = __spi_pump_transfer_message(ctlr, msg, was_busy).
- kthread_queue_work(ctlr.kworker, &ctlr.pump_messages) /* re-arm */.
- ctlr.cur_msg = NULL; ctlr.fallback = false.
- mutex_unlock(&ctlr.io_mutex).
- if !ret: cond_resched().

REQ-21: __spi_pump_transfer_message(ctlr, msg, was_busy):
- if !was_busy ∧ ctlr.auto_runtime_pm: pm_runtime_get_sync(ctlr.dev.parent); on err: msg.status = err; finalize; return.
- if !was_busy ∧ ctlr.prepare_transfer_hardware: call; on err: pm_runtime_put + finalize + return.
- if ctlr.prepare_message: call; on err: finalize + return; else msg.prepared = true.
- spi_map_msg(ctlr, msg); on err: finalize + return.
- WRITE_ONCE(ctlr.cur_msg_incomplete, true); WRITE_ONCE(ctlr.cur_msg_need_completion, false); reinit_completion(&ctlr.cur_msg_completion); smp_wmb.
- ctlr.transfer_one_message(ctlr, msg).
- WRITE_ONCE(ctlr.cur_msg_need_completion, true); smp_mb.
- if cur_msg_incomplete: wait_for_completion(&ctlr.cur_msg_completion).

REQ-22: spi_transfer_one_message(ctlr, msg) /* default */:
- xfer = first; spi_set_cs(msg.spi, !xfer.cs_off, false).
- for each xfer:
  - trace_spi_transfer_start.
  - if (tx_buf ∨ rx_buf) ∧ xfer.len:
    - reinit_completion(&ctlr.xfer_completion).
    - fallback_pio: spi_dma_sync_for_device; ret = ctlr.transfer_one(ctlr, msg.spi, xfer).
    - if ret < 0 ∧ (mapped ∧ SPI_TRANS_FAIL_NO_START): __spi_unmap_msg; ctlr.fallback = true; clear flag; goto fallback_pio.
    - if ret > 0: spi_transfer_wait(ctlr, msg, xfer).
    - spi_dma_sync_for_cpu.
  - else if xfer.len: warn "Bufferless transfer has length".
  - if msg.status != -EINPROGRESS: out.
  - spi_transfer_delay_exec(xfer).
  - if xfer.cs_change:
    - if last xfer: keep_cs = true.
    - else: if !xfer.cs_off: spi_set_cs(msg.spi, false, false); _spi_transfer_cs_change_delay; if !next.cs_off: spi_set_cs(msg.spi, true, false).
  - else if !last ∧ xfer.cs_off != next.cs_off: spi_set_cs(msg.spi, xfer.cs_off, false).
  - msg.actual_length += xfer.len.
- out: if err ∨ !keep_cs: spi_set_cs(msg.spi, false, false).
- if msg.status == -EINPROGRESS: msg.status = ret.
- if msg.status ∧ ctlr.handle_err: ctlr.handle_err(ctlr, msg).
- spi_finalize_current_message(ctlr).

REQ-23: spi_transfer_wait(ctlr, msg, xfer):
- if spi_controller_is_target(ctlr): wait_for_completion_interruptible(&ctlr.xfer_completion) /* target side */.
- else: compute timeout from xfer.speed_hz and xfer.len; wait_for_completion_timeout.
- on timeout/err: log + return err.

REQ-24: spi_async(spi, message):
- spi_maybe_optimize_message(spi, message) — if !pre_optimized ∧ !defer_optimize: __spi_optimize_message → __spi_validate + spi_split_transfers + ctlr.optimize_message (set msg.optimized).
- spin_lock_irqsave(&ctlr.bus_lock_spinlock).
- if ctlr.bus_lock_flag: ret = -EBUSY /* spi_bus_lock active */.
- else: __spi_async(spi, message):
  - if !ctlr.transfer: -ENOTSUPP.
  - SPI_STATISTICS_INCREMENT_FIELD(spi_async) on ctlr + device.
  - trace_spi_message_submit.
  - if !ctlr.ptp_sts_supported: for each xfer: ptp_read_system_prets(xfer.ptp_sts).
  - return ctlr.transfer(spi, message) /* default `spi_queued_transfer` → __spi_queued_transfer → kthread_queue_work */.
- unlock.

REQ-25: spi_sync(spi, message):
- mutex_lock(&spi.controller.bus_lock_mutex).
- __spi_sync(spi, message):
  - if __spi_check_suspended(ctlr): warn-once + -ESHUTDOWN.
  - spi_maybe_optimize_message.
  - SPI_STATISTICS_INCREMENT_FIELD(spi_sync).
  - if queue_empty ∧ !ctlr.must_async: /* fast-path: skip kthread */
    - msg.actual_length = 0; msg.status = -EINPROGRESS.
    - trace_spi_message_submit.
    - SPI_STATISTICS_INCREMENT_FIELD(spi_sync_immediate).
    - __spi_transfer_message_noqueue(ctlr, msg) — same machinery without queueing.
    - return msg.status.
  - /* slow-path: kthread queue + wait completion */
  - msg.complete = spi_complete; msg.context = &done.
  - __spi_async(spi, message).
  - wait_for_completion(&done).
  - return msg.status.
- mutex_unlock.

REQ-26: spi_sync_locked(spi, message):
- bypasses bus_lock_mutex — used by `spi_bus_lock` holders.
- __spi_sync(spi, message).

REQ-27: spi_bus_lock(ctlr):
- mutex_lock(&ctlr.bus_lock_mutex) /* exclusive access for the duration */.
- spin_lock_irqsave(&ctlr.bus_lock_spinlock); ctlr.bus_lock_flag = 1; unlock.

REQ-28: spi_bus_unlock(ctlr):
- spin_lock_irqsave; ctlr.bus_lock_flag = 0; unlock.
- mutex_unlock(&ctlr.bus_lock_mutex).

REQ-29: spi_write_then_read(spi, txbuf, n_tx, rxbuf, n_rx):
- /* Static `buf[SPI_BUFSIZ]` (kmalloc'd in `spi_init`) protected by static mutex */
- if (n_tx + n_rx) > SPI_BUFSIZ ∨ !mutex_trylock(&lock): local_buf = kmalloc(max(SPI_BUFSIZ, n_tx + n_rx), GFP_KERNEL|GFP_DMA).
- else: local_buf = buf.
- spi_message_init; build x[0] (tx, n_tx) + x[1] (rx, n_rx).
- memcpy(local_buf, txbuf, n_tx).
- x[0].tx_buf = local_buf; x[1].rx_buf = local_buf + n_tx.
- spi_sync(spi, &message); on success: memcpy(rxbuf, x[1].rx_buf, n_rx).
- release static buf or kfree local_buf.

REQ-30: of_register_spi_devices(ctlr):
- for_each_available_child_of_node(ctlr.dev.of_node, nc):
  - if OF_POPULATED already set: continue.
  - of_register_spi_device(ctlr, nc):
    - spi = spi_alloc_device(ctlr).
    - of_alias_from_compatible(nc, spi.modalias, SPI_NAME_SIZE).
    - of_spi_parse_dt(ctlr, spi, nc) — parses `reg`, `spi-max-frequency`, `spi-cpha`, `spi-cpol`, `spi-cs-high`, `spi-3wire`, `spi-lsb-first`, `spi-{tx,rx}-bus-width`, `spi-rx-delay-us`, `spi-{cs-{setup,hold,inactive}-delay-ns}`, ancillary properties.
    - of_node_get(nc); device_set_node(&spi.dev, of_fwnode_handle(nc)).
    - spi_add_device(spi).
  - on failure: clear OF_POPULATED.

REQ-31: acpi_register_spi_devices(ctlr) (CONFIG_ACPI):
- acpi_walk_namespace from ctlr.dev's parent ACPI handle, calling acpi_spi_add_device for each child:
  - acpi_register_spi_device(ctlr, adev):
    - if !adev.status.present ∨ enumerated: AE_OK.
    - spi = acpi_spi_device_alloc(ctlr, adev, /*index=*/ -1):
      - acpi_dev_get_resources(adev, &resource_list, acpi_spi_add_resource, &lookup) — parses ACPI_RESOURCE_TYPE_SERIAL_BUS with SerialBusType==SPI: device_selection (CS), connection_speed (max_speed_hz), data_bit_length (bpw), wire_mode, slave_mode (mode), clock_phase, clock_polarity.
      - Apple-quirk: if no max_speed_hz from _CRS ∧ parent matches: acpi_spi_parse_apple_properties → reads `clock-mode`, `spi-cpol`, `spi-cpha`, `spi-cs-high`, `max-clock`, `chip-select` from nested device properties.
      - if !lookup.max_speed_hz: -ENODEV.
      - spi = spi_alloc_device(lookup.ctlr); spi_set_chipselect(spi, 0, lookup.chip_select); ACPI_COMPANION_SET; spi.{max_speed_hz, mode, irq, bits_per_word} = lookup.*.
    - acpi_set_modalias(adev, hid, spi.modalias, SPI_NAME_SIZE).
    - if spi.irq < 0: acpi_dev_gpio_irq_get(adev, 0).
    - acpi_device_set_enumerated(adev).
    - adev.power.flags.ignore_parent = true.
    - spi_add_device(spi); on err: revert ignore_parent; spi_dev_put.

REQ-32: spi_register_board_info(info, n) (legacy):
- bi = kzalloc(n × sizeof(boardinfo)).
- per-entry: memcpy(&bi.board_info, info); list_add_tail(&bi.list, &board_list); for each ctlr in spi_controller_list: spi_match_controller_to_boardinfo.

REQ-33: spi_match_device(dev, drv) match order:
- spi.driver_override (if set): exact match against drv.name.
- ACPI device-id (acpi_driver_match_device).
- OF compatible (strip vendor prefix; match against id_table modalias or drv.name).
- spi_driver.id_table modalias entry.
- driver.name strcmp.

REQ-34: of_spi_notify / acpi_spi_notify (dynamic): handle OF reconfig events and ACPI hot-add — on `OF_RECONFIG_ATTACH_NODE` / ACPI device-attach: find controller via `of_find_spi_controller_by_node` / `acpi_spi_find_controller_by_adev`; register; on detach: lookup device + spi_unregister_device.

REQ-35: spi_target_abort(spi):
- if !ctlr.target_abort: -ENOTSUPP.
- ctlr.target_abort(spi).

REQ-36: spi_take_timestamp_pre / _post:
- per-PTP-timestamp; conditional on xfer.ptp_sts non-NULL; if irqs_off: local_irq_save + preempt_disable; ptp_read_system_prets/postts; restore.

REQ-37: SPI mode bits (per `__spi_setup` + per `__spi_validate`):
- SPI_CPHA (mode 1/3) / SPI_CPOL (mode 2/3) → SPI_MODE_0..3 via SPI_MODE_X_MASK.
- SPI_CS_HIGH: active-high chip-select (default low).
- SPI_LSB_FIRST: shift LSB first (default MSB first).
- SPI_3WIRE: shared MOSI/MISO (half-duplex; bidirectional).
- SPI_LOOP: loopback (controller-side internal MISO↔MOSI).
- SPI_NO_CS: no chip-select (always selected).
- SPI_READY: peripheral pulls SS to pause.
- SPI_TX_DUAL/_QUAD/_OCTAL + SPI_RX_DUAL/_QUAD/_OCTAL: multi-lane.
- SPI_RX_CPHA_FLIP: invert CPHA for RX-only (Renesas RSPI quirk).
- SPI_CS_WORD: CS toggles every word (sw fallback if HW unsupported).
- SPI_MOSI_IDLE_LOW / _HIGH: MOSI idle state (mutually exclusive).
- SPI_NO_TX / _RX: disable a direction.

REQ-38: postcore_initcall(spi_init):
- buf = kmalloc(SPI_BUFSIZ, GFP_KERNEL) /* per-static-`spi_write_then_read` buffer */.
- bus_register(&spi_bus_type).
- class_register(&spi_controller_class).
- if CONFIG_SPI_SLAVE: class_register(&spi_target_class).
- if CONFIG_OF_DYNAMIC: of_reconfig_notifier_register(&spi_of_notifier).
- if CONFIG_ACPI: acpi_reconfig_notifier_register(&spi_acpi_notifier).

## Acceptance Criteria

- [ ] AC-1: spi_register_controller assigns `spi<N>` bus number (idr-allocated); /sys/class/spi_master/spi<N>/ becomes visible.
- [ ] AC-2: of_register_spi_devices enumerates each `available` child of the controller's OF node into a `spi_device` and probes via `spi_match_device`.
- [ ] AC-3: acpi_register_spi_devices enumerates ACPI `SpiSerialBus` `_CRS` resources (including Apple property quirk) into `spi_device`.
- [ ] AC-4: spi_register_board_info: legacy `spi_board_info` matched by controller bus_num + chip_select.
- [ ] AC-5: __spi_setup rejects `SPI_3WIRE | SPI_TX_DUAL` simultaneously (-EINVAL); rejects `SPI_MOSI_IDLE_LOW | SPI_MOSI_IDLE_HIGH` simultaneously.
- [ ] AC-6: __spi_validate rejects partial-transfer (xfer.len % spi_bpw_to_bytes(bpw) != 0) → -EINVAL.
- [ ] AC-7: __spi_validate rejects half-duplex/3WIRE transfer with both tx_buf and rx_buf.
- [ ] AC-8: spi_async submits to kthread message-pump (`spi_pump_messages`) via __spi_queued_transfer; ctlr.queue head removed by __spi_pump_messages.
- [ ] AC-9: spi_sync fast-path (queue_empty ∧ !must_async): bypasses kthread via __spi_transfer_message_noqueue; updates `spi_sync_immediate` stat.
- [ ] AC-10: spi_sync_locked: holds bus_lock_mutex via spi_bus_lock; spi_async returns -EBUSY when bus_lock_flag set.
- [ ] AC-11: spi_transfer_one_message: respects xfer.cs_change with end-of-message keep-CS flag; non-last cs_change toggles CS off+delay+on.
- [ ] AC-12: spi_set_cs short-circuits when CS state unchanged (force=false).
- [ ] AC-13: spi_map_msg krealloc's dummy_tx/_rx GFP_DMA when SPI_CONTROLLER_MUST_TX/_RX set.
- [ ] AC-14: spi_map_buf handles vmalloc-addr, kmap-addr, and direct virt-addr buffers with correct sg_table.
- [ ] AC-15: spi_write_then_read fits within static SPI_BUFSIZ when small; falls back to kmalloc(GFP_KERNEL|GFP_DMA) otherwise.

## Architecture

```
struct SpiController {
  dev: Device,
  bus_num: i32,
  num_chipselect: u32,
  num_data_lanes: u8,
  target: bool,
  mode_bits: u32,
  buswidth_override_bits: u32,
  bits_per_word_mask: u32,
  min_speed_hz: u32, max_speed_hz: u32,
  max_message_size: usize, max_transfer_size: usize, max_dma_len: usize,
  flags: u32,                       // SPI_CONTROLLER_*
  dtr_caps: bool,
  cs_gpiods: Option<Box<[GpioDesc; N]>>,
  use_gpio_descriptors: bool,
  last_cs: [i8; SPI_DEVICE_CS_CNT_MAX],
  last_cs_mode_high: bool,
  last_cs_index_mask: u32,
  max_native_cs: u8, unused_native_cs: i8,
  setup: Option<fn(*mut SpiDevice) -> Errno>,
  cleanup: Option<fn(*mut SpiDevice)>,
  prepare_transfer_hardware: Option<fn(*mut SpiController) -> Errno>,
  unprepare_transfer_hardware: Option<fn(*mut SpiController) -> Errno>,
  prepare_message: Option<fn(*mut SpiController, *mut SpiMessage) -> Errno>,
  unprepare_message: Option<fn(*mut SpiController, *mut SpiMessage) -> Errno>,
  set_cs: Option<fn(*mut SpiDevice, bool)>,
  set_cs_timing: Option<fn(*mut SpiDevice) -> Errno>,
  transfer: Option<fn(*mut SpiDevice, *mut SpiMessage) -> Errno>,
  transfer_one: Option<fn(*mut SpiController, *mut SpiDevice, *mut SpiTransfer) -> i32>,
  transfer_one_message: Option<fn(*mut SpiController, *mut SpiMessage) -> Errno>,
  handle_err: Option<fn(*mut SpiController, *mut SpiMessage)>,
  optimize_message: Option<fn(*mut SpiMessage) -> Errno>,
  unoptimize_message: Option<fn(*mut SpiMessage)>,
  mem_ops: Option<*const SpiControllerMemOps>,
  target_abort: Option<fn(*mut SpiDevice) -> Errno>,
  auto_runtime_pm: bool, rt: bool, must_async: bool, devm_allocated: bool,
  queue: ListHead<SpiMessage>,
  queue_lock: SpinLock,
  queue_empty: AtomicBool,
  running: bool, busy: bool, queued: bool,
  cur_msg: Option<*mut SpiMessage>,
  cur_msg_incomplete: AtomicBool, cur_msg_need_completion: AtomicBool,
  cur_msg_completion: Completion, xfer_completion: Completion,
  fallback: bool,
  kworker: *mut KthreadWorker, pump_messages: KthreadWork,
  bus_lock_spinlock: SpinLock, bus_lock_mutex: Mutex,
  bus_lock_flag: u8, io_mutex: Mutex, add_lock: Mutex,
  dummy_rx: *mut u8, dummy_tx: *mut u8,
  can_dma: Option<fn(*mut SpiController, *mut SpiDevice, *mut SpiTransfer) -> bool>,
  dma_map_dev: *mut Device, dma_tx: *mut DmaChan, dma_rx: *mut DmaChan,
  cur_tx_dma_dev: *mut Device, cur_rx_dma_dev: *mut Device,
  pcpu_statistics: *mut SpiStatistics,           // __percpu
  list: ListHead<SpiController>,
  ptp_sts_supported: bool,
  defer_optimize_message: bool,
  irq_flags: AtomicU64,                          // for take_timestamp_pre PIO path
}

struct SpiDevice {
  controller: *mut SpiController,
  dev: Device,
  max_speed_hz: u32,
  chip_select: [i8; SPI_DEVICE_CS_CNT_MAX],
  num_chipselect: u8, cs_index_mask: u32,
  mode: u32, bits_per_word: u8,
  cs_gpiod: [Option<*mut GpioDesc>; SPI_DEVICE_CS_CNT_MAX],
  cs_setup: SpiDelay, cs_hold: SpiDelay, cs_inactive: SpiDelay,
  word_delay: SpiDelay,
  irq: i32,
  modalias: [u8; SPI_NAME_SIZE],
  driver_override: *mut u8,
  rt: bool,
  pcpu_statistics: *mut SpiStatistics,
  controller_data: *mut (),
}

struct SpiMessage {
  transfers: ListHead<SpiTransfer>,
  spi: *mut SpiDevice,
  frame_length: u32, actual_length: u32,
  status: i32,
  complete: Option<fn(*mut ())>,
  context: *mut (),
  queue: ListHead<SpiMessage>,
  state: *mut (), opt_state: *mut (),
  optimized: bool, pre_optimized: bool, prepared: bool,
}

struct SpiTransfer {
  tx_buf: *const u8, rx_buf: *mut u8,
  len: u32,
  speed_hz: u32, bits_per_word: u8,
  tx_nbits: u8, rx_nbits: u8,
  dtr_mode: bool,
  cs_change: bool, cs_off: bool,
  cs_change_delay: SpiDelay,
  delay: SpiDelay, word_delay: SpiDelay,
  transfer_list: ListHead<SpiTransfer>,
  tx_sg: SgTable, rx_sg: SgTable,
  tx_sg_mapped: bool, rx_sg_mapped: bool,
  error: u32,
  effective_speed_hz: u32,
  ptp_sts: *mut PtpSystemTimestamp,
  ptp_sts_word_pre: u32, ptp_sts_word_post: u32,
  timestamped: bool,
}
```

`SpiCore::register_controller(ctlr) -> Result<(), Errno>`:
1. if !ctlr.dev.parent: return Err(ENODEV).
2. controller_check_ops(ctlr)?.
3. bus_num allocation: fixed vs dynamic via spi_controller_idr.
4. init_completion(xfer_completion, cur_msg_completion).
5. if max_dma_len == 0: max_dma_len = INT_MAX.
6. dev_set_name(spi<N>).
7. if host ∧ use_gpio_descriptors: get_gpio_descs; mode_bits |= SPI_CS_HIGH.
8. if num_chipselect == 0: Err(EINVAL).
9. last_cs[..] = SPI_INVALID_CS.
10. device_add(&ctlr.dev)?.
11. if transfer: pr_info "controller is unqueued, this is deprecated".
12. else if transfer_one | transfer_one_message: controller_initialize_queue(ctlr) — install spi_queued_transfer + default transfer_one_message + spi_init_queue + spi_start_queue.
13. board_lock: list_add_tail(&ctlr.list); match-existing boardinfo.
14. of_register_spi_devices(ctlr).
15. acpi_register_spi_devices(ctlr).

`SpiCore::async(spi, msg) -> Result<(), Errno>`:
1. maybe_optimize_message(spi, msg)?.
2. spin_lock_irqsave(&ctlr.bus_lock_spinlock).
3. if ctlr.bus_lock_flag: ret = Err(EBUSY).
4. else: ret = __async(spi, msg):
   - if !ctlr.transfer: return Err(ENOTSUPP).
   - statistics_increment(spi_async).
   - if !ctlr.ptp_sts_supported: ptp_read_system_prets per xfer.
   - return ctlr.transfer(spi, msg) /* normally spi_queued_transfer */.
5. unlock; return ret.

`SpiCore::sync(spi, msg) -> Result<(), Errno>`:
1. mutex_lock(&ctlr.bus_lock_mutex).
2. __sync(spi, msg):
   - if check_suspended(ctlr): warn_once + Err(ESHUTDOWN).
   - maybe_optimize_message?.
   - statistics_increment(spi_sync).
   - if queue_empty ∧ !must_async:
     - msg.actual_length = 0; msg.status = EINPROGRESS.
     - statistics_increment(spi_sync_immediate).
     - __transfer_message_noqueue(ctlr, msg).
     - return msg.status.
   - msg.complete = spi_complete; msg.context = &done.
   - spin_lock_irqsave(bus_lock_spinlock); __async(spi, msg); unlock.
   - wait_for_completion(&done).
   - msg.complete = NULL; msg.context = NULL.
   - return msg.status.
3. mutex_unlock; return.

`SpiCore::transfer_one_message(ctlr, msg) -> Result<(), Errno>` (default):
1. xfer = first transfer.
2. set_cs(msg.spi, !xfer.cs_off, force=false).
3. for each xfer:
   a. trace_spi_transfer_start.
   b. if (tx_buf ∨ rx_buf) ∧ xfer.len:
      - reinit_completion(xfer_completion).
      - retry_pio: dma_sync_for_device(ctlr, xfer); ret = ctlr.transfer_one(ctlr, spi, xfer).
      - if ret < 0:
        - dma_sync_for_cpu.
        - if (tx_sg_mapped ∨ rx_sg_mapped) ∧ (error & SPI_TRANS_FAIL_NO_START):
          - __unmap_msg; ctlr.fallback = true; error &= ~SPI_TRANS_FAIL_NO_START; goto retry_pio.
        - statistics_increment(errors); err; goto out.
      - if ret > 0: transfer_wait(ctlr, msg, xfer); on err: msg.status = err.
      - dma_sync_for_cpu.
   c. else if xfer.len > 0: warn "Bufferless transfer".
   d. if msg.status != EINPROGRESS: out.
   e. transfer_delay_exec(xfer).
   f. if xfer.cs_change:
      - if last: keep_cs = true.
      - else: !cs_off ⟹ set_cs(false); cs_change_delay_exec; !next.cs_off ⟹ set_cs(true).
   g. else if !last ∧ xfer.cs_off != next.cs_off: set_cs(xfer.cs_off).
   h. msg.actual_length += xfer.len.
4. out: if err ∨ !keep_cs: set_cs(false).
5. if msg.status == EINPROGRESS: msg.status = ret.
6. if msg.status ∧ ctlr.handle_err: ctlr.handle_err.
7. finalize_current_message(ctlr).

`SpiCore::set_cs(spi, enable, force)`:
1. /* Hot-path short-circuit */
2. if !force ∧ enable == is_last_cs(spi) ∧ last_cs_index_mask == spi.cs_index_mask ∧ last_cs_mode_high == (spi.mode & SPI_CS_HIGH): return.
3. update ctlr.last_cs[].
4. last_cs_mode_high = mode & SPI_CS_HIGH.
5. if last_cs_mode_high: enable = !enable /* invert for active-high */.
6. /* Pre-deassert hold-delay */
7. if (csgpiod ∨ !set_cs_timing) ∧ !activate: delay_exec(&spi.cs_hold).
8. if csgpiod:
   - if !NO_CS: for_each_valid_cs: toggle_csgpiod.
   - if SPI_CONTROLLER_GPIO_SS ∧ ctlr.set_cs: ctlr.set_cs(spi, !enable).
9. else if ctlr.set_cs: ctlr.set_cs(spi, !enable).
10. /* Post-assert setup / post-deassert inactive delay */
11. if (csgpiod ∨ !set_cs_timing): activate ? delay_exec(&spi.cs_setup) : delay_exec(&spi.cs_inactive).

`SpiCore::pump_messages(ctlr, in_kthread)` (kthread_work entry):
1. lock io_mutex + queue_lock.
2. if cur_msg: out_unlock.
3. if queue.empty ∨ !running:
   - if !busy: out_unlock.
   - if !in_kthread: defer to kthread (with cheap-teardown short-circuit if no dummy_bufs ∧ no unprepare_hw).
   - else: cheap-teardown — free dummy bufs, call unprepare_transfer_hardware, idle_runtime_pm, mark queue_empty.
4. extract msg = list_first_entry; cur_msg = msg; busy = true.
5. unlock queue_lock.
6. ret = pump_transfer_message(ctlr, msg, was_busy).
7. kthread_queue_work(pump_messages) /* re-arm */.
8. cur_msg = NULL; fallback = false.
9. unlock io_mutex.
10. if !ret: cond_resched().

`SpiCore::map_msg(ctlr, msg)`:
1. /* SPI_CONTROLLER_MUST_TX/RX dummy bufs */
2. if (flags & MUST_TX|MUST_RX) ∧ !3WIRE: compute max_tx, max_rx; krealloc dummy_tx/_rx GFP_KERNEL|GFP_DMA[|__GFP_ZERO].
3. fill dummy_tx into !tx_buf transfers; dummy_rx into !rx_buf transfers.
4. __map_msg: per-xfer ctlr.can_dma ⟹ map_buf_attrs(tx_dev, rx_dev, &sg, ..., DMA_ATTR_SKIP_CPU_SYNC); set tx_sg_mapped / rx_sg_mapped.

`SpiCore::register_driver(owner, sdrv) -> Result<(), Errno>`:
1. sdrv.driver.owner = owner; sdrv.driver.bus = &spi_bus_type.
2. if of_match_table: per-entry warn if no spi_device_id modalias or driver.name match.
3. driver_register(&sdrv.driver).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bus_num_unique` | INVARIANT | per-register_controller: idr_alloc returns unique id; double-alloc fails. |
| `cs_no_duplicate_within_device` | INVARIANT | per-__spi_add_device: logical CS distinct. |
| `cs_no_conflict_across_devices` | INVARIANT | per-spi_dev_check: no two devices on same controller share a physical CS. |
| `mode_bits_validated` | INVARIANT | per-__spi_setup: rejects 3WIRE+dual/quad; rejects MOSI_IDLE_LOW+HIGH; rejects unsupported mode bits. |
| `bpw_validated` | INVARIANT | per-__spi_validate_bits_per_word: bpw matches ctlr.bits_per_word_mask. |
| `partial_transfer_rejected` | INVARIANT | per-__spi_validate: xfer.len % spi_bpw_to_bytes(bpw) == 0. |
| `set_cs_short_circuit_safe` | INVARIANT | per-spi_set_cs: short-circuit only when last_cs/last_cs_mode_high/last_cs_index_mask all match. |
| `bus_lock_blocks_async` | INVARIANT | per-spi_async: bus_lock_flag ⟹ -EBUSY. |
| `sync_check_suspended` | INVARIANT | per-__spi_sync: SPI_CONTROLLER_SUSPENDED ⟹ -ESHUTDOWN. |
| `pump_messages_serialized_by_io_mutex` | INVARIANT | per-__spi_pump_messages: holds io_mutex during transfer; cur_msg never overlaps. |
| `map_buf_dma_safe` | INVARIANT | per-spi_map_buf: rejects non-vmalloc/non-kmap/non-virt-addr buffers. |
| `static_buf_mutex_held` | INVARIANT | per-spi_write_then_read: static buf access guarded by static mutex. |

### Layer 2: TLA+

`drivers/spi/transfer_queue.tla` (cross-ref `models/spi/transfer_queue.tla` in 00-overview):
- Per-controller queue states: STOPPED / RUNNING(busy=true|false) / cur_msg=msg|None / queue=list.
- Per-message states: SUBMITTED / PUMPED / IN_FLIGHT / FINALIZED.
- Properties:
  - `safety_one_cur_msg_at_a_time` — cur_msg non-NULL exclusive of next dequeue.
  - `safety_async_fifo_per_controller` — per-controller, msgs finalize in submit order.
  - `safety_sync_fast_path_iff_queue_empty` — fast-path taken only when queue_empty ∧ !must_async.
  - `safety_bus_lock_excludes_async` — bus_lock_flag ⟹ async returns EBUSY.
  - `liveness_async_eventually_completes` — submitted msg is finalized within bounded retries.
  - `liveness_sync_returns` — spi_sync returns success/error or ESHUTDOWN within bounded time.

`drivers/spi/cs_toggle.tla` (cross-ref `models/spi/cs_toggle.tla`):
- Per-CS line states: ASSERTED / DEASSERTED.
- Per-transfer-in-message events: BEGIN_TRANSFER, END_TRANSFER, CS_TOGGLE (cs_change), CS_OFF.
- Properties:
  - `safety_cs_glitch_free` — no back-to-back ASSERTED→ASSERTED without intervening DEASSERTED if cs_change set; vice versa.
  - `safety_cs_high_inversion` — SPI_CS_HIGH consistently inverts spi_set_cs sense.
  - `safety_cs_off_xfer_keeps_cs_low` — cs_off=true: CS stays deasserted across the transfer.
  - `safety_cs_kept_at_message_end` — if last xfer.cs_change set: CS stays asserted until next message.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `SpiCore::register_controller` post: bus_num assigned ∧ ctlr listed in spi_controller_list | `SpiCore::register_controller` |
| `SpiCore::register_controller` post: queued mode → kworker spawned + queue running | `SpiCore::register_controller` |
| `SpiCore::add_device` post: spi.chip_select validated, listed under controller, __spi_setup succeeded | `SpiCore::add_device` |
| `SpiCore::setup` post: mode masks reduced; bits_per_word ≥ 8 default | `SpiCore::setup` |
| `SpiCore::async` post: msg queued OR -EBUSY OR -ESHUTDOWN OR -ENOTSUPP | `SpiCore::async` |
| `SpiCore::sync` post: msg.status set; bus_lock_mutex released | `SpiCore::sync` |
| `SpiCore::transfer_one_message` post: spi_finalize_current_message called exactly once | `SpiCore::transfer_one_message` |
| `SpiCore::set_cs` post: ctlr.last_cs / last_cs_mode_high / last_cs_index_mask reflect enable state | `SpiCore::set_cs` |
| `SpiCore::map_msg` post: per-xfer sg-mapped flags match buf-NULL-ness; dummy buf inserted iff MUST flag | `SpiCore::map_msg` |
| `SpiCore::map_buf` post: sg_table valid; dma_map_sgtable succeeded; unmap path matches | `SpiCore::map_buf` / `unmap_buf` |
| `SpiCore::validate` post: each xfer has bits_per_word, speed_hz, dtr_mode within controller caps | `SpiCore::validate` |

### Layer 4: Verus/Creusot functional

`Per-controller register → OF/ACPI/board-info enumeration → per-spi_device probe (via spi_match_device modalias) → spi_driver.probe → spi_async/sync submit → kthread pump → spi_transfer_one_message → ctlr.transfer_one per xfer → completion callback → spi_finalize_current_message` semantic equivalence: per-Documentation/spi/spi_summary.rst + per-/dev/spidev<bus>.<cs> IOCTL semantics (`Documentation/spi/spidev.rst`). Per-`spi-mem` consumer (covered by `drivers/spi/spi-mem.md`): consumes `ctlr.mem_ops` populated by per-LLD; this Tier-3 verifies that `spi_controller_check_ops` accepts mem-only controllers (`mem_ops.exec_op` non-NULL with no transfer*).

## Hardening

(Inherits row-1 features from `drivers/spi/00-overview.md` § Hardening.)

SPI-core reinforcement:

- **Per-spi_set_cs short-circuit cached state strict-compare (last_cs, last_cs_mode_high, last_cs_index_mask)** — defense against per-CS-glitch from stale-cache.
- **Per-__spi_validate partial-transfer reject (len % bpw_bytes != 0)** — defense against per-controller alignment-fault.
- **Per-__spi_setup mode-bits whitelist (ctlr.mode_bits | SPI_CS_WORD | SPI_NO_TX | SPI_NO_RX)** — defense against per-driver-requesting-unsupported-mode silent data corruption.
- **Per-mutual-exclusion checks (DUAL/QUAD/NO_TX, DUAL/QUAD/NO_RX, 3WIRE+dual/quad/octal, MOSI_IDLE_LOW+HIGH)** — defense against per-conflicting-mode hardware undefined state.
- **Per-bus_lock_flag in spi_async (-EBUSY)** — defense against per-non-exclusive interleaving during exclusive bus session.
- **Per-spi_controller_check_ops mandate (transfer*/mem_ops.exec_op)** — defense against per-half-baked controller register without I/O path.
- **Per-MUST_TX/_RX dummy buf krealloc GFP_DMA|__GFP_ZERO** — defense against per-MISO sampling uninitialized memory.
- **Per-write_then_read static buf SPI_BUFSIZ + static mutex** — defense against per-concurrent-mutate corruption.
- **Per-cur_msg_completion barrier (smp_wmb + smp_mb pair around transfer_one_message)** — defense against per-finalize-before-wait UAF on cur_msg.
- **Per-spi_unregister_controller children-first __unregister + idr_remove** — defense against per-stale-device after controller-removal.
- **Per-spi_dev_check no-CS-conflict across-bus check** — defense against per-misconfigured-board CS-collision (would short two devices).
- **Per-pm_runtime_resume_and_get balanced with put_noidle on failure** — defense against per-RPM-refcount leak in __spi_setup / __spi_pump_transfer_message.
- **Per-/dev/spidev CAP_SYS_RAWIO + LSM mediation (covered in spidev.md)** — defense against per-userspace SPI-flash-overwrite (boot ROM).

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- /dev/spidev<bus>.<cs> userspace chardev (covered in `drivers/spi/spidev.md` Tier-3).
- spi-mem operations consumed by SPI-NOR / SPI-NAND (covered in `drivers/spi/spi-mem.md` Tier-3).
- Per-controller LLDs (covered in `drivers/spi/controllers-x86.md` and `controllers-misc.md` Tier-3).
- spi-bitbang software-driven controller (covered separately if expanded).
- Implementation code.
