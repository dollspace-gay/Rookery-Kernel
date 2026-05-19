# Tier-3: drivers/mmc/core/core.c — MMC/SD/SDIO core (host + card + request lifecycle)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/mmc/00-overview.md
upstream-paths:
  - drivers/mmc/core/core.c (~2429 lines)
  - drivers/mmc/core/core.h
  - include/linux/mmc/host.h
  - include/linux/mmc/card.h
  - include/linux/mmc/core.h
  - include/linux/mmc/mmc.h
  - include/linux/mmc/sd.h
  - include/linux/mmc/sdio.h
-->

## Summary

`drivers/mmc/core/core.c` is the **bus-agnostic core** of the MMC/SD/SDIO subsystem. It owns three lifetimes: per-`struct mmc_host` (a controller/slot, allocated by per-controller LLD via `mmc_alloc_host`, registered via `mmc_add_host`, removed via `mmc_remove_host`), per-`struct mmc_card` (a physical card detected on a host, owned by the per-card-type bus handler MMC / SD / SDIO via `mmc_attach_bus`), and per-`struct mmc_request` (a command + optional data phase + optional stop, submitted via `mmc_start_request` / `mmc_wait_for_req`, completed by the LLD via `mmc_request_done`). Per-rescan-`mmc_rescan`-workqueue probes attached card type in the strict order **SDIO → SD → MMC** at decreasing frequencies (`freqs[]`). Per-UHS-I (Ultra High Speed) cards: signal-voltage switch via `mmc_set_uhs_voltage` (CMD11 + voltage-switch + clock-gate dance) and per-CRC-error retune via `mmc_retune` (lazily armed by `mmc_retune_needed`, executed at `__mmc_start_request`). Per-claim model: a host has one claimer at a time (`__mmc_claim_host` / `mmc_release_host`, nestable for same claimer, with `pm_runtime_get_sync` on first claim). Per-data-timeout: per-card-type heuristic (`mmc_set_data_timeout`) — MMC uses CSD's `taac_ns/taac_clks * (10 << r2w_factor)`, SD uses `* 100` capped at 250/3000ms, SDIO fixed 1s, SPI-mode at least 100ms read / 1s write. Per-CQE (Command Queueing Engine, eMMC 5.1+): separate `mmc_cqe_start_req` / `mmc_cqe_request_done` / `mmc_cqe_recovery` path. Per-`mmc_blk` (block driver): consumes `mmc_get_card` / `mmc_put_card` to gate every `request_queue` request. Critical for: SD card boot, eMMC main storage, SDIO Wi-Fi/BT combo cards, RPMB-secure-storage gate.

This Tier-3 covers `drivers/mmc/core/core.c` (~2429 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mmc_host` | per-controller / per-slot | `MmcHost` (in `host.c`) |
| `struct mmc_card` | per-card | `MmcCard` (in `bus.c`) |
| `struct mmc_request` | per-command-or-transfer | `MmcRequest` |
| `struct mmc_command` | per-CMD | `MmcCommand` |
| `struct mmc_data` | per-data-phase | `MmcData` |
| `struct mmc_ios` | per-controller-state (clock, bus_width, signal_voltage, timing) | `MmcIos` |
| `struct mmc_bus_ops` | per-card-type ops vtable (remove/detect/suspend/resume/runtime/alive/shutdown/cache_ctrl/hw_reset/sw_reset/handle_undervoltage) | `MmcBusOps` |
| `mmc_alloc_host()` / `mmc_add_host()` / `mmc_remove_host()` / `mmc_free_host()` | per-host lifecycle (in `host.c`, called by per-controller LLD) | `MmcCore::alloc_host` / `add_host` / `remove_host` / `free_host` |
| `mmc_attach_bus()` / `mmc_detach_bus()` | per-card-type binding | `MmcCore::attach_bus` / `detach_bus` |
| `mmc_attach_sdio()` / `mmc_attach_sd()` / `mmc_attach_mmc()` / `mmc_attach_sd_uhs2()` | per-card-type probe | (in `sdio.c` / `sd.c` / `mmc.c` / `sd_uhs2.c`) |
| `mmc_request_done()` | per-LLD completion callback | `MmcCore::request_done` |
| `mmc_command_done()` | per-LLD command-phase done (for `cap_cmd_during_tfr`) | `MmcCore::command_done` |
| `mmc_start_request()` | per-request submit (claim held) | `MmcCore::start_request` |
| `mmc_wait_for_req()` / `_done()` / `mmc_wait_for_cmd()` | per-request sync wait | `MmcCore::wait_for_req` / `wait_for_req_done` / `wait_for_cmd` |
| `mmc_cqe_start_req()` / `_request_done()` / `_post_req()` / `mmc_cqe_recovery()` | per-CQE async submit + completion + recovery | `MmcCore::cqe_*` |
| `mmc_is_req_done()` | per-`cap_cmd_during_tfr` poll | `MmcCore::is_req_done` |
| `mmc_set_data_timeout()` | per-data-phase timeout calculation | `MmcCore::set_data_timeout` |
| `__mmc_claim_host()` / `mmc_release_host()` / `mmc_get_card()` / `mmc_put_card()` | per-host exclusive claim (nestable, PM-gating) | `MmcCore::claim_host` / `release_host` / `get_card` / `put_card` |
| `mmc_set_chip_select()` / `mmc_set_clock()` / `mmc_set_bus_mode()` / `mmc_set_bus_width()` / `mmc_set_timing()` / `mmc_set_driver_type()` | per-IOS mutator (calls `host->ops->set_ios`) | `MmcCore::set_chip_select` / `set_clock` / etc. |
| `mmc_set_initial_state()` / `mmc_set_initial_signal_voltage()` | per-power-up reset | `MmcCore::set_initial_state` / `set_initial_signal_voltage` |
| `mmc_set_signal_voltage()` / `mmc_set_uhs_voltage()` / `mmc_host_set_uhs_voltage()` | per-signal-voltage switch (CMD11 dance) | `MmcCore::set_signal_voltage` / `set_uhs_voltage` / `host_set_uhs_voltage` |
| `mmc_select_voltage()` / `mmc_vddrange_to_ocrmask()` | per-OCR-mask negotiation | `MmcCore::select_voltage` / `vddrange_to_ocrmask` |
| `mmc_power_up()` / `mmc_power_off()` / `mmc_power_cycle()` | per-host power sequencing | `MmcCore::power_up` / `power_off` / `power_cycle` |
| `mmc_execute_tuning()` | per-UHS-I HS200/HS400 tuning kick-off (calls `host->ops->execute_tuning`) | `MmcCore::execute_tuning` |
| `mmc_retune_needed()` / `mmc_retune_hold()` / `_release()` / `_recheck()` / `mmc_retune()` | per-CRC-arm + per-call-site-gate retune state machine | `MmcCore::retune_*` |
| `mmc_hw_reset()` / `mmc_sw_reset()` / `mmc_hw_reset_for_init()` | per-card reset | `MmcCore::hw_reset` / `sw_reset` / `hw_reset_for_init` |
| `mmc_handle_undervoltage()` | per-regulator-undervoltage emergency stop | `MmcCore::handle_undervoltage` |
| `mmc_detect_change()` / `_mmc_detect_change()` / `mmc_rescan()` | per-card-insert/remove kick + rescan workqueue | `MmcCore::detect_change` / `_detect_change` / `rescan` |
| `mmc_detect_card_removed()` / `_mmc_detect_card_removed()` | per-card-alive poll | `MmcCore::detect_card_removed` |
| `mmc_start_host()` / `mmc_stop_host()` / `__mmc_stop_host()` | per-host start + stop | `MmcCore::start_host` / `stop_host` / `__stop_host` |
| `mmc_erase()` / `mmc_calc_max_discard()` / `mmc_init_erase()` / `mmc_erase_group_aligned()` | per-erase / trim / discard | `MmcCore::erase` / `calc_max_discard` / `init_erase` / `erase_group_aligned` |
| `mmc_card_can_erase()` / `_can_trim()` / `_can_discard()` / `_can_sanitize()` / `_can_secure_erase_trim()` / `_can_cmd23()` / `_is_blockaddr()` | per-capability probe | `MmcCore::card_can_*` / `card_is_blockaddr` |
| `mmc_set_blocklen()` | per-CMD16 block-length set | `MmcCore::set_blocklen` |
| `mmc_card_alternative_gpt_sector()` | per-NVIDIA-Tegra alt-GPT offset | `MmcCore::card_alternative_gpt_sector` |
| `freqs[]` array | per-rescan-frequency-ladder (400 kHz → max) | shared const |
| `use_spi_crc` module-param | per-SPI-mode CRC enable | shared |
| `subsys_initcall(mmc_init)` | per-init: `mmc_register_bus` + `mmc_register_host_class` + `sdio_register_bus` | `MmcCore::init` |

## Compatibility contract

REQ-1: struct mmc_host (per-controller, allocated by LLD):
- `class_dev`: per-`/sys/class/mmc_host/mmc<N>/` kobject.
- `index`: per-`mmcN` instance number (idr-allocated in `host.c`).
- `ops`: per-`struct mmc_host_ops` vtable (request, set_ios, get_ro, get_cd, execute_tuning, prepare_hs400_tuning, hs400_complete, hs400_enhanced_strobe, card_event, card_busy, init_card, multi_io_quirk, start_signal_voltage_switch, prepare_sd_clock_switch, ...).
- `ios`: per-current `struct mmc_ios` (clock, vdd, bus_mode, chip_select, power_mode, bus_width, timing, signal_voltage, drv_type, enhanced_strobe).
- `f_min` / `f_max` / `f_init`: per-Hz bounds.
- `ocr_avail`: per-OCR-mask of host-supported voltages.
- `caps` / `caps2`: per-`MMC_CAP_*` / `MMC_CAP2_*` flags (4-bit / 8-bit data, SDR104, HS200, HS400, HS400ES, UHS-II, SD-Express, NO_PRESCAN_POWERUP, NEEDS_POLL, SYNC_RUNTIME_PM, NO_SD, NO_MMC, NO_SDIO, ALT_GPT_TEGRA, ...).
- `max_seg_size` / `max_segs` / `max_blk_size` / `max_blk_count` / `max_req_size` / `max_busy_timeout`: per-DMA-bound.
- `card`: per-attached `struct mmc_card` (NULL when no card).
- `bus_ops`: per-card-type vtable (set by `mmc_attach_bus`).
- `detect`: per-`struct delayed_work` rescan handle.
- `detect_change`: per-flag set by `_mmc_detect_change`.
- `rescan_disable` / `rescan_entered`: per-rescan gates.
- `ongoing_mrq`: per-`cap_cmd_during_tfr` active request.
- `claimed` / `claim_cnt` / `claimer` / `default_ctx` / `wq` / `lock`: per-host exclusive-claim state.
- `cqe_on` / `cqe_ops` / `cqe_private`: per-CQE state.
- `retune_period` / `retune_paused` / `retune_now` / `retune_crc_disable` / `need_retune`: per-retune state.
- `pm_flags` / `pm_caps` / `ws`: per-PM + per-wakesource.
- `led`: per-`/sys/class/leds/mmc<N>::` activity trigger.
- `slot`: per-CD-IRQ / per-WP-GPIO descriptor.
- `uhs2_sd_tran`: per-UHS-II transmission state.

REQ-2: struct mmc_card (per-card):
- `host`: back-pointer.
- `type`: `MMC_TYPE_MMC` / `MMC_TYPE_SD` / `MMC_TYPE_SDIO` / `MMC_TYPE_SD_COMBO`.
- `state`: `MMC_STATE_PRESENT` / `_READONLY` / `_BLOCKADDR` / `_HIGHSPEED` / `_REMOVED` / `_DOING_BKOPS` / ... bit-flags.
- `rca`: per-Relative Card Address (RCA).
- `ocr` / `raw_cid[4]` / `raw_csd[4]` / `raw_scr[2]` / `raw_ssr[16]` / `ext_csd`: per-register-cache.
- `cid` / `csd` / `scr` / `ssr` / `ext_csd`: per-decoded fields.
- `erase_size` / `erase_shift` / `pref_erase`: per-erase-granularity.
- `dev`: per-`/sys/bus/mmc/devices/mmc<N>:<RCA>/` kobject.

REQ-3: struct mmc_request:
- `sbc`: per-SET_BLOCK_COUNT (CMD23) prefix command (or NULL).
- `cmd`: per-`struct mmc_command` (opcode, arg, flags `MMC_RSP_*` / `MMC_CMD_*`, resp[4], retries, error, busy_timeout, has_ext_addr, ext_addr).
- `data`: per-`struct mmc_data` (timeout_ns, timeout_clks, blksz, blocks, blk_addr, error, flags `MMC_DATA_READ`/`_WRITE`/`_STREAM`, sg, sg_len, sg_count, bytes_xfered).
- `stop`: per-STOP_TRANSMISSION (CMD12) command (or NULL — used for streaming reads/writes).
- `completion`: per-`struct completion` (signaled by `mmc_wait_done`).
- `cmd_completion`: per-`struct completion` for `cap_cmd_during_tfr`.
- `done`: per-callback invoked by `mmc_request_done` (e.g. `mmc_blk` queue completion).
- `host`: back-pointer.
- `tag`: per-CQE-tag (0..31).
- `cap_cmd_during_tfr`: per-flag (CQE-direct CMDs interleaved with data).
- `recovery_notifier`: per-CQE-recovery hook.

REQ-4: mmc_alloc_host(extra, dev) (defined in `host.c`):
- /* Allocate `struct mmc_host` + LLD-private trailer */
- Initialize `class_dev` per-class.
- Initialize `idr`-allocated `index`.
- INIT_DELAYED_WORK(&host.detect, mmc_rescan).
- init_waitqueue_head(&host.wq).
- spin_lock_init(&host.lock).
- mutex_init for the rescan path.
- Initialize default-ctx + retune-timer.
- Return `*mmc_host` or NULL.

REQ-5: mmc_add_host(host) (defined in `host.c`):
- /* Register class device */
- device_add(&host.class_dev).
- mmc_add_host_debugfs(host).
- mmc_start_host(host).
- Return 0 or -errno.

REQ-6: mmc_remove_host(host):
- mmc_stop_host(host).
- device_del(&host.class_dev).

REQ-7: mmc_attach_bus(host, ops):
- host.bus_ops = ops.

REQ-8: mmc_detach_bus(host):
- host.bus_ops = NULL.

REQ-9: _mmc_detect_change(host, delay, cd_irq):
- if cd_irq ∧ !(host.caps & MMC_CAP_NEEDS_POLL):
  - __pm_wakeup_event(host.ws, 5000) /* 5s wake-budget for user-space uevent */.
- host.detect_change = 1.
- mmc_schedule_delayed_work(&host.detect, delay).

REQ-10: mmc_detect_change(host, delay):
- _mmc_detect_change(host, delay, /*cd_irq*/ true).

REQ-11: mmc_rescan(work):
- /* container_of(work, struct mmc_host, detect.work) */
- if host.rescan_disable: return.
- if !mmc_card_is_removable(host) ∧ host.rescan_entered: return.
- host.rescan_entered = 1.
- if host.trigger_card_event ∧ host.ops.card_event:
  - mmc_claim_host(host).
  - host.ops.card_event(host).
  - mmc_release_host(host).
  - host.trigger_card_event = false.
- if host.bus_ops: host.bus_ops.detect(host).
- host.detect_change = 0.
- if host.bus_ops != NULL: goto out (card still attached).
- mmc_claim_host(host).
- if mmc_card_is_removable(host) ∧ host.ops.get_cd ∧ host.ops.get_cd(host) == 0:
  - mmc_power_off(host).
  - mmc_release_host(host).
  - goto out.
- if mmc_card_sd_express(host): mmc_release_host(host); goto out.
- if !mmc_attach_sd_uhs2(host): mmc_release_host(host); goto out.
- for freq in freqs[]:
  - if freq > host.f_max ∧ has-next: continue; else freq = host.f_max.
  - if !mmc_rescan_try_freq(host, max(freq, host.f_min)): break.
  - if freqs[i] <= host.f_min: break.
- mmc_release_host(host).
- out:
  - if host.caps & MMC_CAP_NEEDS_POLL: mmc_schedule_delayed_work(&host.detect, HZ) /* 1-Hz poll */.

REQ-12: mmc_rescan_try_freq(host, freq):
- host.f_init = freq.
- mmc_power_up(host, host.ocr_avail).
- mmc_hw_reset_for_init(host).
- if !(host.caps2 & MMC_CAP2_NO_SDIO): sdio_reset(host) /* CMD52 ignored by SD/eMMC */.
- mmc_go_idle(host) /* CMD0 */.
- if !(host.caps2 & MMC_CAP2_NO_SD):
  - if mmc_send_if_cond_pcie(host, host.ocr_avail): goto out.
  - if mmc_card_sd_express(host): return 0.
- /* Strict probe order: SDIO → SD → MMC */
- if !(host.caps2 & MMC_CAP2_NO_SDIO) ∧ !mmc_attach_sdio(host): return 0.
- if !(host.caps2 & MMC_CAP2_NO_SD)  ∧ !mmc_attach_sd(host):   return 0.
- if !(host.caps2 & MMC_CAP2_NO_MMC) ∧ !mmc_attach_mmc(host):  return 0.
- out:
  - mmc_power_off(host).
  - return -EIO.

REQ-13: mmc_start_host(host):
- power_up = !(host.caps2 & (MMC_CAP2_NO_PRESCAN_POWERUP | MMC_CAP2_SD_UHS2)).
- host.f_init = max(min(freqs[0], host.f_max), host.f_min).
- host.rescan_disable = 0.
- if power_up: claim → mmc_power_up(host, host.ocr_avail) → release.
- mmc_gpiod_request_cd_irq(host).
- _mmc_detect_change(host, /*delay=*/ 0, /*cd_irq=*/ false).

REQ-14: __mmc_stop_host(host):
- if host.rescan_disable: return.
- if host.slot.cd_irq >= 0:
  - mmc_gpio_set_cd_wake(host, false).
  - disable_irq(host.slot.cd_irq).
- host.rescan_disable = 1.
- cancel_delayed_work_sync(&host.detect).

REQ-15: mmc_stop_host(host):
- __mmc_stop_host(host).
- host.pm_flags = 0.
- if host.bus_ops:
  - /* Bus-handler remove must be called WITHOUT host claimed (deadlock) */
  - host.bus_ops.remove(host).
  - mmc_claim_host(host).
  - mmc_detach_bus(host).
  - mmc_power_off(host).
  - mmc_release_host(host).
  - return.
- mmc_claim_host(host); mmc_power_off(host); mmc_release_host(host).

REQ-16: __mmc_claim_host(host, ctx, abort) (nestable for same context):
- might_sleep().
- add_wait_queue(&host.wq, &wait).
- spin_lock_irqsave(&host.lock).
- loop:
  - set_current_state(TASK_UNINTERRUPTIBLE).
  - stop = abort ? atomic_read(abort) : 0.
  - if stop ∨ !host.claimed ∨ mmc_ctx_matches(host, ctx, task): break.
  - unlock; schedule(); lock.
- set_current_state(TASK_RUNNING).
- if !stop:
  - host.claimed = 1.
  - mmc_ctx_set_claimer(host, ctx, task).
  - host.claim_cnt += 1.
  - if first claim: pm = true.
- else: wake_up(&host.wq).
- unlock + remove_wait_queue.
- if pm: pm_runtime_get_sync(mmc_dev(host)).
- return stop.

REQ-17: mmc_release_host(host):
- WARN_ON(!host.claimed).
- spin_lock_irqsave(&host.lock).
- if --host.claim_cnt:
  - unlock /* nested release */.
- else:
  - host.claimed = 0.
  - host.claimer.task = NULL.
  - host.claimer = NULL.
  - unlock.
  - wake_up(&host.wq).
  - pm_runtime_mark_last_busy(mmc_dev(host)).
  - if host.caps & MMC_CAP_SYNC_RUNTIME_PM: pm_runtime_put_sync_suspend(mmc_dev(host)).
  - else: pm_runtime_put_autosuspend(mmc_dev(host)).

REQ-18: mmc_start_request(host, mrq):
- if mrq.cmd.has_ext_addr: mmc_send_ext_addr(host, mrq.cmd.ext_addr).
- init_completion(&mrq.cmd_completion).
- mmc_retune_hold(host).
- if mmc_card_removed(host.card): return -ENOMEDIUM.
- WARN_ON(!host.claimed).
- err = mmc_mrq_prep(host, mrq):
  - cmd.error = 0; cmd.mrq = mrq; cmd.data = mrq.data.
  - if data: validate `blksz ≤ host.max_blk_size`, `blocks ≤ host.max_blk_count`, `blocks * blksz ≤ host.max_req_size`, ∑sg.length == blocks * blksz; return -EINVAL on fail.
  - if stop: data.stop = stop; stop.mrq = mrq.
- if host.uhs2_sd_tran: mmc_uhs2_prepare_cmd(host, mrq).
- led_trigger_event(host.led, LED_FULL).
- __mmc_start_request(host, mrq):
  - mmc_retune(host) — if err: mrq.cmd.error = err; mmc_request_done(host, mrq); return.
  - if sdio_is_io_busy(mrq.cmd.opcode, mrq.cmd.arg) ∧ host.ops.card_busy:
    - poll card_busy up to ~500ms; on timeout: -EBUSY → mmc_request_done.
  - if mrq.cap_cmd_during_tfr: host.ongoing_mrq = mrq; reinit_completion(&mrq.cmd_completion).
  - if host.cqe_on: host.cqe_ops.cqe_off(host).
  - host.ops.request(host, mrq).
- return 0.

REQ-19: mmc_request_done(host, mrq) /* called by LLD */:
- /* CRC error arms retune (unless tuning op or `retune_crc_disable`) */
- if !mmc_op_tuning(cmd.opcode) ∧ !host.retune_crc_disable ∧ (cmd.error == -EILSEQ ∨ sbc.error == -EILSEQ ∨ data.error == -EILSEQ ∨ stop.error == -EILSEQ): mmc_retune_needed(host).
- if cmd.error ∧ cmd.retries ∧ mmc_host_is_spi(host) ∧ (cmd.resp[0] & R1_SPI_ILLEGAL_COMMAND): cmd.retries = 0.
- if host.ongoing_mrq == mrq: host.ongoing_mrq = NULL.
- mmc_complete_cmd(mrq) /* completes cmd_completion */.
- trace_mmc_request_done.
- if !err ∨ !cmd.retries ∨ mmc_card_removed(host.card):
  - mmc_should_fail_request(host, mrq) /* fault-injection */.
  - if !host.ongoing_mrq: led_trigger_event(host.led, LED_OFF).
  - pr_debug per cmd / sbc / data / stop.
- if mrq.done: mrq.done(mrq) /* Note: retries handled in `mmc_wait_for_req_done` via re-call of `__mmc_start_request` */.

REQ-20: mmc_command_done(host, mrq) /* called by LLD for `cap_cmd_during_tfr` after cmd phase */:
- complete_all(&mrq.cmd_completion).

REQ-21: mmc_wait_for_req(host, mrq):
- __mmc_start_req(host, mrq):
  - mmc_wait_ongoing_tfr_cmd(host) — if a `cap_cmd_during_tfr` mrq is in flight, wait for its cmd-completion.
  - init_completion(&mrq.completion); mrq.done = mmc_wait_done.
  - err = mmc_start_request; on err: mark cmd error + complete + complete cmd_completion.
- if !mrq.cap_cmd_during_tfr: mmc_wait_for_req_done(host, mrq).

REQ-22: mmc_wait_for_req_done(host, mrq):
- loop:
  - wait_for_completion(&mrq.completion).
  - if !cmd.error ∨ !cmd.retries ∨ mmc_card_removed(host.card): break.
  - mmc_retune_recheck(host).
  - cmd.retries -= 1; cmd.error = 0; __mmc_start_request(host, mrq).
- mmc_retune_release(host).

REQ-23: mmc_wait_for_cmd(host, cmd, retries):
- WARN_ON(!host.claimed).
- zero cmd.resp; cmd.retries = retries; mrq.cmd = cmd; cmd.data = NULL.
- mmc_wait_for_req(host, &mrq).
- return cmd.error.

REQ-24: CQE path (`mmc_cqe_start_req` / `_request_done` / `_post_req` / `_recovery`):
- `_start_req`: mmc_retune (CQE can't retune mid-queue); mrq.host = host; mmc_mrq_prep; if host.uhs2_sd_tran: mmc_uhs2_prepare_cmd; host.cqe_ops.cqe_request.
- `_request_done`: per-LLD callback. CRC error arms retune. Calls `mrq.done(mrq)` directly (no retry loop — caller-handled).
- `_post_req`: optional host.cqe_ops.cqe_post_req.
- `_recovery`: mmc_retune_hold_now; cqe_recovery_start; CMD12 STOP_TRANSMISSION (`MMC_RSP_R1B_NO_CRC | MMC_CMD_AC`, 1000ms busy timeout); poll busy; CMDQ_TASK_MGMT arg=1 (discard entire queue); cqe_recovery_finish; on err retry CMDQ_TASK_MGMT once; mmc_retune_release.

REQ-25: mmc_set_data_timeout(data, card):
- if mmc_card_sdio(card): data.timeout_ns = 1_000_000_000; data.timeout_clks = 0; return.
- mult = mmc_card_sd(card) ? 100 : 10.
- if data.flags & MMC_DATA_WRITE: mult <<= card.csd.r2w_factor.
- data.timeout_ns = card.csd.taac_ns * mult.
- data.timeout_clks = card.csd.taac_clks * mult.
- if SD:
  - timeout_us = data.timeout_ns / 1000.
  - if card.host.ios.clock: timeout_us += data.timeout_clks * 1000 / (host.ios.clock / 1000).
  - limit_us = write ? 3_000_000 : 100_000.
  - if timeout_us > limit_us: data.timeout_ns = limit_us * 1000; data.timeout_clks = 0.
  - if timeout_us == 0: data.timeout_ns = limit_us * 1000.
- if mmc_card_long_read_time(card) ∧ MMC_DATA_READ: data.timeout_ns = 600_000_000; data.timeout_clks = 0.
- if mmc_host_is_spi(card.host):
  - WRITE: data.timeout_ns = max(data.timeout_ns, 1_000_000_000) /* 1s */.
  - READ:  data.timeout_ns = max(data.timeout_ns,   100_000_000) /* 100ms */.

REQ-26: IOS mutators all call `mmc_set_ios(host)` which dispatches to `host->ops->set_ios(host, &host->ios)`:
- mmc_set_chip_select(host, mode): host.ios.chip_select = mode.
- mmc_set_clock(host, hz): WARN if hz < f_min; clamp ≤ f_max; host.ios.clock = hz.
- mmc_set_bus_mode(host, mode): MMC_BUSMODE_OPENDRAIN / _PUSHPULL.
- mmc_set_bus_width(host, width): MMC_BUS_WIDTH_1 / _4 / _8.
- mmc_set_timing(host, timing): MMC_TIMING_LEGACY / _MMC_HS / _SD_HS / _UHS_SDR12 / _SDR25 / _SDR50 / _SDR104 / _DDR50 / _MMC_HS200 / _MMC_HS400 / _MMC_DDR52.
- mmc_set_driver_type(host, drv_type): 0..3 (SD-spec drive-strength).

REQ-27: mmc_set_initial_state(host) (called from `mmc_power_up` and after `hw_reset`):
- if host.cqe_on: host.cqe_ops.cqe_off(host).
- mmc_retune_disable(host).
- chip_select: SPI=CS_HIGH, else CS_DONTCARE.
- bus_mode = PUSHPULL; bus_width = 1; timing = LEGACY; drv_type = 0; enhanced_strobe = false.
- if (caps2 & MMC_CAP2_HS400_ES) ∧ host.ops.hs400_enhanced_strobe: host.ops.hs400_enhanced_strobe(host, &host.ios).
- mmc_set_ios(host).
- mmc_crypto_set_initial_state(host).

REQ-28: mmc_power_up(host, ocr):
- if host.ios.power_mode == MMC_POWER_ON: return.
- mmc_pwrseq_pre_power_on(host).
- host.ios.vdd = fls(ocr) - 1.
- host.ios.power_mode = MMC_POWER_UP.
- mmc_set_initial_state(host).
- mmc_set_initial_signal_voltage(host).
- mmc_delay(host.ios.power_delay_ms).
- mmc_pwrseq_post_power_on(host).
- host.ios.clock = host.f_init.
- host.ios.power_mode = MMC_POWER_ON.
- mmc_set_ios(host).
- mmc_delay(host.ios.power_delay_ms) /* ≥74 clocks + ≥1ms */.

REQ-29: mmc_power_off(host):
- if host.ios.power_mode == MMC_POWER_OFF: return.
- mmc_pwrseq_power_off(host).
- host.ios.clock = 0; host.ios.vdd = 0; power_mode = MMC_POWER_OFF.
- mmc_set_initial_state(host).
- mmc_delay(1).

REQ-30: mmc_power_cycle(host, ocr):
- mmc_power_off; mmc_delay(1); mmc_power_up(host, ocr).

REQ-31: mmc_set_initial_signal_voltage(host):
- Try 3.3V → fallback 1.8V → fallback 1.2V (via mmc_set_signal_voltage).

REQ-32: mmc_set_signal_voltage(host, signal_voltage):
- old = host.ios.signal_voltage.
- host.ios.signal_voltage = signal_voltage.
- if host.ops.start_signal_voltage_switch: err = ops.start_signal_voltage_switch(host, &host.ios).
- if err: host.ios.signal_voltage = old.
- return err.

REQ-33: mmc_host_set_uhs_voltage(host) /* 5-ms-gated clock-pause for SD spec */:
- clock = host.ios.clock.
- host.ios.clock = 0; mmc_set_ios(host).
- if mmc_set_signal_voltage(host, MMC_SIGNAL_VOLTAGE_180): return -EAGAIN.
- mmc_delay(10) /* spec says 5ms */.
- host.ios.clock = clock; mmc_set_ios(host).
- return 0.

REQ-34: mmc_set_uhs_voltage(host, ocr) — full SD CMD11 dance:
- if !host.ops.start_signal_voltage_switch: return -EPERM.
- if !host.ops.card_busy: pr_warn (cannot verify).
- send CMD11 (SD_SWITCH_VOLTAGE), R1, MMC_CMD_AC.
- on err: power_cycle.
- if !SPI ∧ (resp[0] & R1_ERROR): return -EIO.
- mmc_delay(1); verify card holds CMD+DAT[0:3] low (card_busy must return true).
- mmc_host_set_uhs_voltage; if fail: power_cycle.
- mmc_delay(1); verify card released busy (card_busy must return false).
- on any failure: mmc_power_cycle(host, ocr).
- return err.

REQ-35: mmc_execute_tuning(card) /* HS200/HS400 only */:
- host = card.host.
- if !host.ops.execute_tuning: return 0.
- if host.cqe_on: host.cqe_ops.cqe_off(host).
- opcode = mmc_card_mmc(card) ? MMC_SEND_TUNING_BLOCK_HS200 : MMC_SEND_TUNING_BLOCK.
- err = host.ops.execute_tuning(host, opcode).
- if !err: mmc_retune_clear(host); mmc_retune_enable(host).
- else if !host.detect_change: pr_err + mmc_debugfs_err_stats_inc(host, MMC_ERR_TUNING).
- return err.

REQ-36: mmc_retune (UHS-I CRC-driven re-tune):
- `mmc_retune_needed(host)`: per-CRC-error: host.need_retune = 1 (cheap; armed from `mmc_request_done`).
- `mmc_retune_hold(host)` / `mmc_retune_hold_now(host)`: per-call-site-gate (e.g. during sub-step that must not get interrupted by tuning).
- `mmc_retune_release(host)`: per-counterpart.
- `mmc_retune_recheck(host)`: per-`mmc_wait_for_req_done`: arm retune for next retry.
- `mmc_retune(host)`: per-`__mmc_start_request`/`mmc_cqe_start_req` entry: if need_retune ∧ !retune_paused ∧ host.ops.execute_tuning → execute_tuning + retune-clear.
- `mmc_retune_clear(host)`: per-success: need_retune = 0.
- `mmc_retune_enable(host)` / `_disable(host)`: per-arm timer (`retune_period` seconds).

REQ-37: mmc_hw_reset(card) / mmc_sw_reset(card) / mmc_hw_reset_for_init(host):
- `mmc_hw_reset`: call host.ops.card_hw_reset; on success rebind via `mmc_attach_bus`+rescan.
- `mmc_sw_reset`: card-type-specific (eMMC: GO_PRE_IDLE; SDIO: CMD52 reset).
- `mmc_hw_reset_for_init`: called from `mmc_rescan_try_freq` if MMC_CAP_HW_RESET — emergency reset before probe.

REQ-38: mmc_handle_undervoltage(host):
- __mmc_stop_host(host) /* race-free vs card-removal */.
- if !host.bus_ops ∨ !host.bus_ops.handle_undervoltage: return 0.
- dev_warn "Undervoltage detected, initiating emergency stop".
- return host.bus_ops.handle_undervoltage(host).

REQ-39: mmc_detect_card_removed(host) (called by callers holding host claim):
- WARN_ON(!host.claimed).
- if !host.card: return 1.
- if !mmc_card_is_removable(host): return 0.
- ret = mmc_card_removed(card).
- if !host.detect_change ∧ !(host.caps & MMC_CAP_NEEDS_POLL): return ret.
- host.detect_change = 0.
- if !ret: ret = _mmc_detect_card_removed(host).
- if ret ∧ (host.caps & MMC_CAP_NEEDS_POLL): cancel_delayed_work(&host.detect); _mmc_detect_change(host, 0, false).
- return ret.

REQ-40: _mmc_detect_card_removed(host):
- if !host.card ∨ mmc_card_removed(host.card): return 1.
- ret = host.bus_ops.alive(host).
- /* Per-SD-card-mechanical-addendum: slow remove may show alive but get_cd=0; re-arm 200ms */
- if !ret ∧ host.ops.get_cd ∧ !host.ops.get_cd(host): mmc_detect_change(host, msecs_to_jiffies(200)).
- if ret: mmc_card_set_removed(host.card).
- return ret.

REQ-41: mmc_erase(card, from, nr, arg):
- per-card-type validation (can_erase / can_trim / can_discard).
- per-alignment (`mmc_align_erase_size` ↔ `erase_size`).
- CMD35/36 (ERASE_GROUP_START/END) + CMD38 (ERASE) with `arg` ∈ {MMC_ERASE_ARG, MMC_TRIM_ARG, MMC_DISCARD_ARG, MMC_SECURE_ERASE_ARG, MMC_SECURE_TRIM1_ARG, MMC_SECURE_TRIM2_ARG, SD_ERASE_ARG}.
- per-`mmc_erase_timeout` busy-poll.

REQ-42: mmc_calc_max_discard(card):
- per-card max discard size keeping erase timeout ≤ host.max_busy_timeout (default heuristic else card-specific max).

REQ-43: mmc_set_blocklen(card, blocklen):
- CMD16 (SET_BLOCKLEN) with arg=blocklen, MMC_RSP_R1 | MMC_CMD_AC.

REQ-44: mmc_select_voltage(host, ocr):
- mask off bits 0..6 (warn if claimed) → mmc_vddrange_to_ocrmask cross-reference → pick lowest supported.

REQ-45: subsys_initcall(mmc_init):
- mmc_register_bus() /* /sys/bus/mmc/ */.
- mmc_register_host_class() /* /sys/class/mmc_host/ */.
- sdio_register_bus() /* /sys/bus/sdio/ */.
- Unwinds on failure.

REQ-46: module_exit(mmc_exit):
- sdio_unregister_bus.
- mmc_unregister_host_class.
- mmc_unregister_bus.

## Acceptance Criteria

- [ ] AC-1: mmc_alloc_host returns a valid `mmc_host` and `mmc_add_host` makes `/sys/class/mmc_host/mmc<N>/` visible.
- [ ] AC-2: SD card insert: `mmc_rescan` succeeds via `mmc_attach_sd` and `host.bus_ops` non-NULL; `mmcblk<N>` block device appears.
- [ ] AC-3: SDIO Wi-Fi card insert: `mmc_attach_sdio` succeeds before `mmc_attach_sd` is attempted (probe-order respect).
- [ ] AC-4: eMMC (non-removable): `mmc_attach_mmc` succeeds at `host.f_init = freqs[0]` after MMC_CAP2_NO_SD / NO_SDIO masking.
- [ ] AC-5: `mmc_start_request` rejects mrq when `cmd.data.blocks * blksz > host.max_req_size` → -EINVAL.
- [ ] AC-6: `mmc_wait_for_req_done` retries on cmd.error and exhausts `cmd.retries`.
- [ ] AC-7: `mmc_request_done` arms retune on `-EILSEQ` (CRC).
- [ ] AC-8: UHS-I SD: `mmc_set_uhs_voltage` performs CMD11 → 1.8V switch → clock-gate-10ms → CMD-busy verify; on failure mmc_power_cycle.
- [ ] AC-9: HS200/HS400 eMMC: `mmc_execute_tuning` calls `host.ops.execute_tuning` with MMC_SEND_TUNING_BLOCK_HS200.
- [ ] AC-10: `mmc_set_data_timeout`: SDIO = 1 s; SD = capped 250/3000 ms; SPI ≥ 1 s write / 100 ms read.
- [ ] AC-11: `__mmc_claim_host` is reentrant for the same ctx; `claim_cnt` increments without sleeping.
- [ ] AC-12: First claim takes pm_runtime ref; last release drops it.
- [ ] AC-13: CQE recovery: STOP_TRANSMISSION + CMDQ_TASK_MGMT (arg=1) discards the entire queue.
- [ ] AC-14: `mmc_detect_change` schedules `mmc_rescan` after `delay` jiffies and grants 5s pm_wakeup budget for cd_irq path.
- [ ] AC-15: `mmc_handle_undervoltage` stops the host before invoking `bus_ops.handle_undervoltage` (race-free with card-removal).
- [ ] AC-16: `mmc_card_alternative_gpt_sector` only applies to MMC_CAP2_ALT_GPT_TEGRA + eMMC + ext_csd.rev≥3 + blockaddr + non-removable.

## Architecture

```
struct MmcHost {
  class_dev: Device,
  index: u32,
  ops: *const MmcHostOps,
  ios: MmcIos,
  f_min: u32, f_max: u32, f_init: u32,
  ocr_avail: u32,
  caps: u32, caps2: u32,
  max_seg_size: u32, max_segs: u16, max_blk_size: u32,
  max_blk_count: u32, max_req_size: u32, max_busy_timeout: u32,
  card: Option<*mut MmcCard>,
  bus_ops: Option<*const MmcBusOps>,
  detect: DelayedWork,
  detect_change: bool, rescan_disable: bool, rescan_entered: bool,
  trigger_card_event: bool,
  ongoing_mrq: Option<*mut MmcRequest>,
  claimed: bool, claim_cnt: u32,
  claimer: Option<*mut MmcCtx>, default_ctx: MmcCtx,
  wq: WaitQueueHead, lock: SpinLock,
  cqe_on: bool, cqe_ops: Option<*const MmcCqeOps>, cqe_private: *mut (),
  retune_period: u32, retune_paused: bool, retune_now: bool,
  retune_crc_disable: bool, need_retune: bool,
  pm_flags: u32, pm_caps: u32, ws: *mut WakeupSource,
  led: *mut LedTrigger,
  slot: MmcSlot,
  uhs2_sd_tran: bool,
  ws_sd_uhs2: SdUhs2Tran,
  err_stats: [u32; MMC_ERR_MAX],
  ...
}

struct MmcIos {
  clock: u32, vdd: u8, bus_mode: u8, chip_select: u8,
  power_mode: u8, bus_width: u8, timing: u8, signal_voltage: u8,
  drv_type: u8, enhanced_strobe: bool, power_delay_ms: u32,
}

struct MmcCard {
  host: *mut MmcHost,
  ty: u8, state: u32, rca: u32,
  raw_cid: [u32; 4], raw_csd: [u32; 4],
  raw_scr: [u32; 2], raw_ssr: [u32; 16],
  cid: MmcCid, csd: MmcCsd, scr: SdScr, ssr: SdSsr, ext_csd: EmmcExtCsd,
  ocr: u32, erase_size: u32, erase_shift: u32, pref_erase: u32,
  dev: Device,
}

struct MmcRequest {
  sbc: Option<*mut MmcCommand>,
  cmd: *mut MmcCommand,
  data: Option<*mut MmcData>,
  stop: Option<*mut MmcCommand>,
  completion: Completion,
  cmd_completion: Completion,
  done: Option<fn(*mut MmcRequest)>,
  host: *mut MmcHost,
  tag: i32,
  cap_cmd_during_tfr: bool,
  recovery_notifier: Option<fn(*mut MmcRequest)>,
}
```

`MmcCore::start_request(host, mrq) -> Result<(), Errno>`:
1. if cmd.has_ext_addr: send_ext_addr(host, cmd.ext_addr).
2. init_completion(&mrq.cmd_completion).
3. retune_hold(host).
4. if card_removed(host.card): return Err(ENOMEDIUM).
5. WARN_ON(!host.claimed).
6. mrq_prep(host, mrq)? — validates blksz/blocks/sg-sum.
7. if host.uhs2_sd_tran: uhs2_prepare_cmd(host, mrq).
8. led_trigger_event(host.led, LED_FULL).
9. __start_request(host, mrq):
   - retune(host)? — on err: cmd.error = err; request_done(host, mrq); return.
   - if sdio_is_io_busy(cmd.opcode, cmd.arg) ∧ host.ops.card_busy:
     - tries = 500; while card_busy ∧ --tries: delay(1ms). On timeout: cmd.error = EBUSY; request_done.
   - if mrq.cap_cmd_during_tfr: host.ongoing_mrq = mrq; reinit_completion.
   - if host.cqe_on: cqe_off(host).
   - host.ops.request(host, mrq).
10. Ok.

`MmcCore::request_done(host, mrq)`:
1. /* Arm retune on CRC anywhere */
2. if !op_tuning(cmd.opcode) ∧ !host.retune_crc_disable ∧ any-EILSEQ: retune_needed(host).
3. /* SPI illegal-cmd: kill retries */
4. if cmd.error ∧ cmd.retries ∧ host_is_spi(host) ∧ (cmd.resp[0] & R1_SPI_ILLEGAL_COMMAND): cmd.retries = 0.
5. if host.ongoing_mrq == mrq: host.ongoing_mrq = NULL.
6. complete_cmd(mrq).
7. trace_request_done.
8. if !cmd.error ∨ !cmd.retries ∨ card_removed(host.card):
   - should_fail_request(host, mrq).
   - if !host.ongoing_mrq: led_trigger_event(host.led, LED_OFF).
   - pr_debug per sbc / cmd / data / stop.
9. if mrq.done: mrq.done(mrq).

`MmcCore::wait_for_req_done(host, mrq)`:
1. loop:
   - wait_for_completion(&mrq.completion).
   - cmd = mrq.cmd.
   - if !cmd.error ∨ !cmd.retries ∨ card_removed(host.card): break.
   - retune_recheck(host).
   - cmd.retries -= 1; cmd.error = 0; __start_request(host, mrq).
2. retune_release(host).

`MmcCore::rescan(work)`:
1. host = container_of(work, MmcHost, detect.work).
2. if host.rescan_disable: return.
3. if !card_is_removable(host) ∧ host.rescan_entered: return.
4. host.rescan_entered = true.
5. if host.trigger_card_event ∧ host.ops.card_event:
   - claim → ops.card_event(host) → release; trigger_card_event = false.
6. if host.bus_ops: host.bus_ops.detect(host).
7. host.detect_change = false.
8. if host.bus_ops != NULL: goto out.
9. claim_host(host).
10. if card_is_removable(host) ∧ host.ops.get_cd ∧ host.ops.get_cd(host) == 0: power_off; release; goto out.
11. if card_sd_express(host): release; goto out.
12. if !attach_sd_uhs2(host): release; goto out.
13. for freq in freqs[]: try mmc_rescan_try_freq.
14. release.
15. out: if host.caps & MMC_CAP_NEEDS_POLL: schedule_delayed_work(&host.detect, HZ).

`MmcCore::rescan_try_freq(host, freq)`:
1. host.f_init = freq.
2. power_up(host, host.ocr_avail).
3. hw_reset_for_init(host).
4. if !(caps2 & NO_SDIO): sdio_reset(host).
5. go_idle(host).
6. if !(caps2 & NO_SD):
   - send_if_cond_pcie? → out.
   - if card_sd_express(host): return Ok.
7. /* Strict order */
8. if !(caps2 & NO_SDIO) ∧ attach_sdio(host)? : return Ok.
9. if !(caps2 & NO_SD)   ∧ attach_sd(host)?   : return Ok.
10. if !(caps2 & NO_MMC) ∧ attach_mmc(host)?  : return Ok.
11. out: power_off; return Err(EIO).

`MmcCore::claim_host(host, ctx, abort) -> Result<(), Aborted>`:
1. might_sleep.
2. add_wait_queue(host.wq, wait).
3. spin_lock_irqsave(host.lock).
4. loop:
   - TASK_UNINTERRUPTIBLE.
   - stop = abort ? atomic_read(abort) : 0.
   - if stop ∨ !host.claimed ∨ ctx_matches(host, ctx, task): break.
   - unlock; schedule; lock.
5. TASK_RUNNING.
6. if !stop: claimed = true; ctx_set_claimer(host, ctx, task); claim_cnt += 1; pm = (claim_cnt == 1).
7. else: wake_up(host.wq).
8. unlock; remove_wait_queue.
9. if pm: pm_runtime_get_sync(mmc_dev(host)).
10. return.

`MmcCore::release_host(host)`:
1. WARN_ON(!host.claimed).
2. spin_lock_irqsave(host.lock).
3. claim_cnt -= 1.
4. if claim_cnt > 0: unlock /* nested */; return.
5. claimed = false; claimer.task = NULL; claimer = NULL.
6. unlock; wake_up(host.wq).
7. pm_runtime_mark_last_busy(mmc_dev(host)).
8. if host.caps & MMC_CAP_SYNC_RUNTIME_PM: pm_runtime_put_sync_suspend.
9. else: pm_runtime_put_autosuspend.

`MmcCore::set_data_timeout(data, card)`:
1. SDIO: 1s flat; return.
2. mult = SD ? 100 : 10.
3. WRITE: mult <<= card.csd.r2w_factor.
4. data.timeout_ns = csd.taac_ns * mult; data.timeout_clks = csd.taac_clks * mult.
5. SD: cap at 250/3000ms via timeout_us computation.
6. long_read_time + READ: 600ms.
7. SPI: WRITE ≥ 1s; READ ≥ 100ms.

`MmcCore::set_uhs_voltage(host, ocr) -> Result<(), Errno>`:
1. if !host.ops.start_signal_voltage_switch: return Err(EPERM).
2. CMD11; on err: power_cycle.
3. if !SPI ∧ resp[0] & R1_ERROR: return Err(EIO).
4. delay(1ms); verify card_busy (must be busy).
5. host_set_uhs_voltage(host): clock=0 → mmc_set_ios → set_signal_voltage(1.8V) → delay(10ms) → restore clock → mmc_set_ios.
6. delay(1ms); verify !card_busy.
7. on failure: power_cycle(host, ocr).

`MmcCore::cqe_recovery(host) -> Result<(), Errno>`:
1. retune_hold_now.
2. cqe_ops.cqe_recovery_start.
3. CMD12 (R1B_NO_CRC, MMC_CMD_AC, 1000ms timeout); MMC_CMD_RETRIES retries.
4. mmc_poll_for_busy(card, 1000ms, MMC_BUSY_IO).
5. CMDQ_TASK_MGMT arg=1 (discard queue); on err retry once.
6. cqe_ops.cqe_recovery_finish.
7. retune_release.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `host_claim_balanced` | INVARIANT | per-claim/release: claim_cnt monotone non-negative; release matches claim. |
| `claim_pm_runtime_balanced` | INVARIANT | first claim pm_runtime_get_sync; last release pm_runtime_put_*. |
| `start_request_requires_claim` | INVARIANT | mmc_start_request asserts host.claimed. |
| `wait_for_cmd_requires_claim` | INVARIANT | mmc_wait_for_cmd asserts host.claimed. |
| `detect_card_removed_requires_claim` | INVARIANT | mmc_detect_card_removed asserts host.claimed. |
| `rescan_only_one_at_a_time` | INVARIANT | delayed_work serialization: rescan never re-entered (per-host work). |
| `retune_paired` | INVARIANT | retune_hold/release strictly paired per start_request / wait_for_req_done. |
| `cqe_off_before_non_cqe_request` | INVARIANT | __start_request: cqe_on ⟹ cqe_off pre-dispatch. |
| `mrq_prep_bounds_checked` | INVARIANT | data.blksz ≤ max_blk_size, blocks ≤ max_blk_count, blocks*blksz ≤ max_req_size, ∑sg.length == blocks*blksz. |
| `crc_arms_retune` | INVARIANT | EILSEQ in cmd/sbc/data/stop ∧ !retune_crc_disable ∧ !op_tuning ⟹ need_retune=1. |
| `set_clock_clamped` | INVARIANT | hz=0 OR f_min ≤ hz ≤ f_max post-clamp. |

### Layer 2: TLA+

`drivers/mmc/core.tla`:
- Per-host states: UNCLAIMED / CLAIMED(ctx, depth) / RESCANNING / SUSPENDED.
- Per-card states: ABSENT / PROBED / ATTACHED(type) / REMOVED.
- Per-request states: IDLE / PREPARED / IN_FLIGHT / COMPLETED / RETRY.
- Properties:
  - `safety_claim_exclusive` — at most one ctx claims host at a time (recursive same-ctx allowed).
  - `safety_probe_order_sdio_first` — per-rescan_try_freq: SDIO attached before SD before MMC (when none gated).
  - `safety_data_timeout_within_bound` — set_data_timeout result ≤ host.max_busy_timeout (when configured).
  - `safety_retune_paired` — every retune_hold matched by retune_release.
  - `safety_cqe_off_pre_legacy` — non-CQE request never dispatched while cqe_on.
  - `liveness_rescan_eventually_completes` — rescan returns within finite freqs[] iteration.
  - `liveness_wait_for_req_terminates` — bounded by initial cmd.retries.
  - `liveness_uhs_voltage_terminates` — CMD11 dance reaches success or power_cycle.

`drivers/mmc/uhs_retune.tla`:
- Models retune_needed/hold/release/recheck/execute interleaved with request submit.
- Property `safety_retune_never_mid_cqe` — retune never executes while cqe_on without explicit cqe_off.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `MmcCore::start_request` pre: host.claimed | `MmcCore::start_request` |
| `MmcCore::start_request` post: ok ⟹ ops.request called exactly once | `MmcCore::start_request` |
| `MmcCore::mrq_prep` post: bounds-checked or -EINVAL returned | `MmcCore::mrq_prep` |
| `MmcCore::request_done` post: mrq.done called iff non-null | `MmcCore::request_done` |
| `MmcCore::wait_for_req_done` post: cmd.error == 0 ∨ cmd.retries == 0 ∨ card_removed | `MmcCore::wait_for_req_done` |
| `MmcCore::rescan_try_freq` post: bus_ops set (success) ∨ powered off (failure) | `MmcCore::rescan_try_freq` |
| `MmcCore::claim_host` post: claimed ∧ claim_cnt ≥ 1 ∧ first-claim ⟹ pm_runtime_get | `MmcCore::claim_host` |
| `MmcCore::release_host` post: last release ⟹ pm_runtime_put | `MmcCore::release_host` |
| `MmcCore::set_data_timeout` post: SDIO=1s ∧ SD≤cap ∧ SPI≥{100ms/1s} | `MmcCore::set_data_timeout` |
| `MmcCore::set_uhs_voltage` post: ok ⟹ ios.signal_voltage=1.8V ∧ verified card_busy | `MmcCore::set_uhs_voltage` |
| `MmcCore::cqe_recovery` post: queue discarded (CMDQ_TASK_MGMT arg=1) | `MmcCore::cqe_recovery` |

### Layer 4: Verus/Creusot functional

`Per-card-insert → mmc_detect_change(host, delay=0, cd_irq=true) → mmc_rescan workqueue → mmc_rescan_try_freq across freqs[] → attach_{sdio,sd,mmc} → bus_ops set → mmc_blk_probe creates /dev/mmcblkN` semantic equivalence: per-Documentation/driver-api/mmc/ + per-mmc-utils-consumes-byte-identical-sysfs / IOCTL behavior. Per-`mmc_blk` integration: `mmc_blk_probe` claims card lazily on each request via `mmc_get_card` and releases via `mmc_put_card`; this Tier-3 verifies the claim/release surface; the block-side queue logic is owned by `drivers/mmc/block.md`.

## Hardening

(Inherits row-1 features from `drivers/mmc/00-overview.md` § Hardening.)

MMC-core reinforcement:

- **Per-host claim strict reentrancy** — defense against per-claim-leak / per-double-release.
- **Per-CRC-error retune-armed** — defense against per-UHS-I silent corruption.
- **Per-mrq_prep bounds checked (blksz/blocks/sg-sum vs host.max_*)** — defense against per-LLD DMA-OOB.
- **Per-card-removed early-return in start_request** — defense against per-stale-card UAF.
- **Per-detect_change pm_wakeup 5s bound** — defense against per-rescan-storm wake-source-leak.
- **Per-rescan strict probe order (SDIO→SD→MMC)** — defense against per-card-type-misdetect (SDIO CMD52 destructive on misrouted SD).
- **Per-handle_undervoltage stops host before bus_ops** — defense against per-race-with-card-removal during emergency stop.
- **Per-CQE recovery discards entire queue** — defense against per-in-flight-CQE-tag-reuse after error.
- **Per-set_uhs_voltage power-cycle on failure** — defense against per-CMD11 partial-switch leaving line in undefined voltage.
- **Per-cqe_off before legacy request** — defense against per-CQE-vs-legacy concurrent submission.
- **Per-mmc_set_clock clamp to f_max** — defense against per-overclock controller damage.
- **Per-SPI-mode CMD CRC-disable rejection of R1_SPI_ILLEGAL_COMMAND** — defense against per-SPI-card retry-storm.
- **Per-MMC_CAP2_ALT_GPT_TEGRA narrow predicate** — defense against per-non-Tegra wrong GPT offset.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `mmc_blk_ioctl` (MMC_IOC_CMD / MMC_IOC_MULTI_CMD) reads `mmc_ioc_cmd` from user into a kernel-side `mmc_blk_ioc_data` slab whitelisted to exactly `sizeof(struct mmc_ioc_cmd) + data_len`; the response buffer is bounded by `MMC_IOC_MAX_BYTES` and copy_to_user'd against an exact-sized object.
- **PAX_KERNEXEC** — `mmc_host_ops`, `mmc_bus_ops`, `mmc_driver`'s probe/remove/suspend/resume table, and the per-card-type ops (`mmc_ops`, `sd_ops`, `sdio_ops`) all live in `__ro_after_init` rodata.
- **PAX_RANDKSTACK** — every `mmc_blk_issue_rq`, `mmc_wait_for_req`, and SDIO `sdio_io_rw_*` entry crosses the per-syscall random kstack offset before reaching the host controller's `request` callback.
- **PAX_REFCOUNT** — `mmc_card->dev.kobj.kref`, `mmc_host->ref`, `mmc_blk_data->usage`, and `sdio_func->use_count` are refcount_t with saturation; a forged hot-remove race cannot wrap the card ref and trigger UAF on the block-disk minor.
- **PAX_MEMORY_SANITIZE** — `mmc_blk_ioc_data` slabs are zeroed on free; the per-request `mmc_request_done` path zeros bounce-buffer pages before they are returned to the page allocator so prior LBA contents do not leak.
- **PAX_UDEREF** — `mmc_blk_ioctl_cmd` copies the user request into kernel space via `copy_from_user`, validates `opcode` against an allowlist (CMD53/CMD56/etc), and only then dispatches; no `__user` deref under KERNEL_DS.
- **PAX_RAP / kCFI** — `mmc_host_ops` (`.request`, `.set_ios`, `.get_ro`, `.get_cd`, `.enable_sdio_irq`, `.start_signal_voltage_switch`, etc.), `mmc_bus_ops`, and `sdio_driver` callbacks are typed indirect-call edges.
- **mmc_card REFCOUNT discipline** — `get_device(&card->dev)` / `put_device(&card->dev)` brackets every external user (`mmc_blk_probe`, `sdio_register_driver`), so a card eject during `mmc_blk_issue_rq` cannot free the card until the in-flight bio drains.
- **mmc_blk CAP_SYS_RAWIO** — `MMC_IOC_CMD` and `MMC_IOC_MULTI_CMD` are gated by `capable(CAP_SYS_RAWIO)` (and `bdev_read_only` for write opcodes); RPMB partition access additionally requires the userspace caller to hold `CAP_SYS_RAWIO` to issue CMD23/CMD18/CMD25 sequences.
- **SDIO bind allowlist** — `sdio_register_driver` matches against the device's CIS-derived `vendor` / `device` IDs; combined with `CONFIG_SDIO_UART_BIND_ALLOWLIST`-style policies, unprivileged userland cannot rebind an SDIO function to a malicious in-tree driver via sysfs `bind`/`unbind` (those nodes require `CAP_SYS_ADMIN`).
- **GRKERNSEC_HIDESYM / GRKERNSEC_DMESG** — `mmc_host`, `mmc_card`, and `mmc_blk_data` pointers are scrubbed under `kptr_restrict ≥ 2`; card-init verbose tracing is at `KERN_DEBUG` with `__ratelimit`.
- **RPMB partition CAP_SYS_RAWIO** — accessing `/dev/mmcblkNrpmb` requires `CAP_SYS_RAWIO` and a write-counter authentication round-trip; an unprivileged process cannot forge keyed boot-image rollback.
- **Boot partition ro_lock** — `mmc_blk_ioctl` rejects writes to a boot partition once `boot_ro_lock` is set in EXT_CSD; the lock survives reboot and prevents firmware/U-Boot replay.
- **eMMC GP partition userns isolation** — general-purpose partition access requires the requesting userns to hold `CAP_SYS_RAWIO` against the host bus; container roots in a child userns are denied.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-card-type probe internals (covered by `drivers/mmc/00-overview.md` § Tier-3 `core.md` and the per-card-type files `mmc.c` / `sd.c` / `sdio.c` separately).
- Block layer integration / `mmc_blk` (covered in `drivers/mmc/block.md` Tier-3).
- SDIO function-driver framework (covered in `drivers/mmc/sdio-func.md` Tier-3).
- Inline Crypto Engine (covered in `drivers/mmc/crypto.md` Tier-3).
- Per-controller LLDs / SDHCI (covered in `drivers/mmc/host-sdhci.md` Tier-3).
- UHS-II transmission state machine (delegated to `core/sd_uhs2.c` Tier-3, not in v0 maintenance set).
- Implementation code.
