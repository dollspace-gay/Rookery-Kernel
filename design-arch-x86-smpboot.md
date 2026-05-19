---
title: "Tier-3: arch/x86/kernel/smpboot.c — x86 SMP secondary-CPU bringup"
tags: ["tier-3", "arch", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

On x86 the boot CPU (BSP) wakes each application processor (AP) by sending an **INIT-SIPI-SIPI** APIC IPI sequence whose start-IP encodes the address of a real-mode trampoline (`real_mode_header->trampoline_start`); the trampoline switches the AP from real to long mode and jumps to `secondary_startup_64` in `head_64.S`, which lands in `start_secondary()`. Per-`native_smp_prepare_cpus()` sets up per-CPU sibling masks, `apic_intr_mode`, init_udelay timing, and on 64-bit may enable **parallel bringup** (`smpboot_control = STARTUP_READ_APICID`). Per-`native_kick_ap()` resolves cpu -> apicid then calls `do_boot_cpu()` which picks the wakeup primitive (`wakeup_secondary_cpu_64` / `wakeup_secondary_cpu` / `wakeup_secondary_cpu_via_init`) and waits. Per-AP `start_secondary()` runs `cr4_init`, `cpu_init_exception_handling`, `load_ucode_ap`, `cpuhp_ap_sync_alive`, `cpu_init`, `fpu__init_cpu`, `ap_starting` (APIC, identify, sibling map), `check_tsc_sync_target`, `ap_calibrate_delay`, `set_cpu_online`, finally `cpu_startup_entry(CPUHP_AP_ONLINE_IDLE)`. Per-`native_smp_cpus_done()` rebuilds the scheduler topology, runs NMI selftest, initializes cache-APs. Critical for: bringing every logical CPU online deterministically, hotplug add (`echo 1 > /sys/devices/system/cpu/cpuN/online`), and resume-from-suspend AP restart.

This Tier-3 covers `arch/x86/kernel/smpboot.c` (~1511 lines).

### Acceptance Criteria

- [ ] AC-1: native_smp_prepare_cpus allocates per-CPU sibling/core/die/llc/l2c masks for every possible CPU.
- [ ] AC-2: do_boot_cpu uses real_mode_header->trampoline_start (or trampoline_start64 if apic->wakeup_secondary_cpu_64 present).
- [ ] AC-3: wakeup_secondary_cpu_via_init sends INIT-assert, INIT-deassert, then 2x STARTUP IPI on integrated APIC (0 STARTUP on non-integrated).
- [ ] AC-4: start_secondary loads microcode (load_ucode_ap) BEFORE cpuhp_ap_sync_alive returns.
- [ ] AC-5: start_secondary calls check_tsc_sync_target between ap_starting and ap_calibrate_delay.
- [ ] AC-6: start_secondary sets set_cpu_online with vector_lock held.
- [ ] AC-7: ap_starting calls set_cpu_sibling_map BEFORE notify_cpu_starting.
- [ ] AC-8: native_smp_cpus_done calls build_sched_topology, nmi_selftest, impress_friends, cache_aps_init in order.
- [ ] AC-9: Parallel bringup (X86_64): smpboot_control == STARTUP_READ_APICID after arch_cpuhp_init_parallel_bringup() returns true.
- [ ] AC-10: native_cpu_disable refuses if cpu == 0 (BSP cannot offline itself).
- [ ] AC-11: mwait_play_dead exits when mwait_cpu_dead.status == CPUDEAD_MWAIT_KEXEC_HLT.
- [ ] AC-12: do_boot_cpu fail path calls arch_cpuhp_cleanup_kick_cpu.
- [ ] AC-13: APIC_PIC / APIC_VIRTUAL_WIRE_NO_CONFIG: disable_smp() invoked.
- [ ] AC-14: send_init_sequence clears APIC_ESR on integrated APIC pre-INIT.

### Architecture

```
struct InitialBootSlots {              // per-AP handoff slots written by BSP, read by head_64.S
  initial_code:  u64,                  // start_secondary
  initial_stack: u64,                  // X86_32 only on this path
  smpboot_control: u32,                // STARTUP_PARALLEL_MASK | STARTUP_READ_APICID | cpu_index
}

struct RealModeHeader {                // arch/x86/include/asm/realmode.h
  trampoline_start:   u32,             // 16-bit entry
  trampoline_start64: u32,             // direct-64-bit entry (TDX/SNP/CC variants)
  trampoline_pgd:     u32,
  ...
}

struct MwaitCpuDead {                  // per-CPU offline rendezvous
  control: u32,                        // CPUDEAD_MWAIT_WAIT
  status:  u32,                        // CPUDEAD_MWAIT_KEXEC_HLT
}
```

`Smpboot::prepare_cpus(max_cpus)`:
1. Smpboot::prepare_cpus_common().
2. match apic_intr_mode:
   - PIC | VIRTUAL_WIRE_NO_CONFIG => disable_smp(); return.
   - SYMMETRIC_IO_NO_ROUTING => disable_smp(); setup_percpu_clockev(); return.
   - VIRTUAL_WIRE | SYMMETRIC_IO => continue.
3. setup_percpu_clockev(); print_cpu_info(cpu0); uv_system_init().
4. set_init_udelay().
5. speculative_store_bypass_ht_init().
6. snp_set_wakeup_secondary_cpu().

`Smpboot::kick_ap(cpu, tidle) -> Result<(), i32>`:
1. apicid = apic.cpu_present_to_apicid(cpu).
2. if apicid == BAD_APICID ∨ !apic.id_valid(apicid): return -EINVAL.
3. if num_online_cpus() > 1 ∧ !cpu_check_up_prepare(cpu): return -EIO.
4. common_cpu_up(cpu, tidle)?
5. do_boot_cpu(apicid, cpu, tidle).

`Smpboot::do_boot_cpu(apicid, cpu, idle) -> Result<(), i32>`:
1. start_ip = realmode.trampoline_start.
2. if x86_64 ∧ apic.wakeup_secondary_cpu_64.is_some(): start_ip = realmode.trampoline_start64.
3. idle.thread.sp = task_pt_regs(idle).
4. initial_code.store(start_secondary as u64, Ordering::Release).
5. if x86_32: early_gdt_descr.address = get_cpu_gdt_rw(cpu); initial_stack.store(idle.thread.sp).
6. else if !(smpboot_control.load() & STARTUP_PARALLEL_MASK): smpboot_control.store(cpu).
7. init_espfix_ap(cpu); announce_cpu(cpu, apicid).
8. if x86_platform.legacy.warm_reset: setup_warm_reset_vector(start_ip); APIC ESR clear.
9. smp_mb().
10. let ret = match apic:
    - { wakeup_secondary_cpu_64: Some(f) } => f(apicid, start_ip, cpu),
    - { wakeup_secondary_cpu: Some(f) }    => f(apicid, start_ip, cpu),
    - _ => wakeup_via_init(apicid, start_ip, cpu).
11. if ret != Ok(()): cleanup_kick_cpu(cpu).
12. return ret.

`Smpboot::wakeup_via_init(phys_apicid, start_eip, cpu)`:
1. preempt_disable().
2. maxlvt = lapic_get_maxlvt().
3. send_init_sequence(phys_apicid).
4. mb().
5. let num_starts = if APIC_INTEGRATED(boot_cpu_apic_version) { 2 } else { 0 }.
6. for _ in 0..num_starts:
   - if maxlvt > 3: apic_write(APIC_ESR, 0).
   - apic_read(APIC_ESR).
   - apic_icr_write(APIC_DM_STARTUP | (start_eip >> 12), phys_apicid).
   - udelay(if init_udelay == 0 { 10 } else { 300 }).
   - send_status = safe_apic_wait_icr_idle().
   - udelay(if init_udelay == 0 { 10 } else { 200 }).
   - if maxlvt > 3: apic_write(APIC_ESR, 0).
   - accept_status = apic_read(APIC_ESR) & 0xEF.
   - if send_status != 0 ∨ accept_status != 0: break.
7. preempt_enable().
8. return send_status | accept_status.

`Smpboot::start_secondary()` (called from `head_64.S::secondary_startup_64` after switching to long mode + final page-table):
1. cr4_init().
2. if x86_32: load_cr3(swapper_pg_dir); flush_tlb_all().
3. cpu_init_exception_handling(false).
4. load_ucode_ap().                                  /* microcode BEFORE alive sync */
5. cpuhp_ap_sync_alive().                            /* sync point */
6. cpu_init(); fpu__init_cpu().
7. rcutree_report_cpu_starting(raw_smp_processor_id()).
8. x86_cpuinit.early_percpu_clock_init().
9. ap_starting().
10. check_tsc_sync_target().                         /* TSC vs BSP */
11. ap_calibrate_delay().
12. speculative_store_bypass_ht_init().
13. lock_vector_lock(); set_cpu_online(cpu, true); lapic_online(); unlock_vector_lock().
14. x86_platform.nmi_init().
15. local_irq_enable().
16. x86_platform.setup_percpu_clockev().
17. wmb().
18. cpu_startup_entry(CPUHP_AP_ONLINE_IDLE).         /* never returns */

`Smpboot::ap_starting()`:
1. this_cpu_write(mwait_cpu_dead.status, 0); this_cpu_write(mwait_cpu_dead.control, 0).
2. apic_ap_setup().
3. identify_secondary_cpu(cpuid).
4. set_cpu_sibling_map(cpuid).                       /* topology BEFORE notify */
5. ap_init_aperfmperf().
6. wmb().
7. notify_cpu_starting(cpuid).

`Smpboot::cpu_disable_common()`:
1. remove_siblinginfo(cpu).
2. remove_cpu_from_maps(cpu).
3. lock_vector_lock(); set_cpu_online(cpu, false); unlock_vector_lock().
4. fixup_irqs().
5. lapic_offline().

`Smpboot::mwait_play_dead(eax_hint)` — never returns:
1. md = this_cpu_ptr(&mwait_cpu_dead).
2. md.control = CPUDEAD_MWAIT_WAIT.
3. loop:
   - mb(); clflush(&md.status); mb().
   - if md.status == CPUDEAD_MWAIT_KEXEC_HLT: break.
   - __monitor(&md.status, 0, 0); mb(); __mwait(eax_hint, 0).
4. hlt_play_dead().

### Out of Scope

- arch/x86/realmode/rm/trampoline_64.S — real-mode-to-long-mode trampoline (covered in separate Tier-3)
- arch/x86/kernel/head_64.S — secondary_startup_64 (covered in `boot.md` / `setup.md` companion)
- kernel/cpu.c generic hotplug state machine (covered in `core/cpu-hotplug.md` Tier-3)
- TSC sync test (`arch/x86/kernel/tsc_sync.c`) — covered in `tsc.md` companion + dedicated Tier-3 if expanded
- APIC driver layer (`arch/x86/kernel/apic/`) — covered in `apic.md` Tier-3
- Microcode loader (`arch/x86/kernel/cpu/microcode/`) — covered separately if expanded
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `native_smp_prepare_cpus()` | per-bringup-prep on BSP | `Smpboot::prepare_cpus` |
| `smp_prepare_cpus_common()` | per-CPU mask alloc + sibling0 | `Smpboot::prepare_cpus_common` |
| `native_smp_prepare_boot_cpu()` | per-BSP-init (gdt + percpu) | `Smpboot::prepare_boot_cpu` |
| `native_smp_cpus_done()` | per-post-bringup finalization | `Smpboot::cpus_done` |
| `arch_cpuhp_kick_ap_alive()` | per-hotplug entry | `Smpboot::kick_ap_alive` |
| `native_kick_ap()` | per-cpu wakeup driver | `Smpboot::kick_ap` |
| `do_boot_cpu()` | per-AP wakeup dispatch | `Smpboot::do_boot_cpu` |
| `common_cpu_up()` | per-AP common state init | `Smpboot::common_cpu_up` |
| `wakeup_secondary_cpu_via_init()` | per-INIT-SIPI-SIPI primitive | `Smpboot::wakeup_via_init` |
| `send_init_sequence()` | per-INIT-assert + INIT-deassert | `Smpboot::send_init_sequence` |
| `smpboot_setup_warm_reset_vector()` | per-CMOS warm-reset vector | `Smpboot::setup_warm_reset_vector` |
| `smpboot_restore_warm_reset_vector()` | per-CMOS warm-reset teardown | `Smpboot::restore_warm_reset_vector` |
| `start_secondary()` | per-AP C-entry from head_64.S | `Smpboot::start_secondary` |
| `ap_starting()` | per-AP-APIC+sibling-map | `Smpboot::ap_starting` |
| `ap_calibrate_delay()` | per-AP delay-loop calib | `Smpboot::ap_calibrate_delay` |
| `set_cpu_sibling_map()` | per-CPU sibling/core/die masks | `Smpboot::set_cpu_sibling_map` |
| `smp_set_init_udelay()` | per-INIT delay tuning | `Smpboot::set_init_udelay` |
| `arch_cpuhp_init_parallel_bringup()` | per-parallel bringup feature | `Smpboot::init_parallel_bringup` |
| `arch_cpuhp_sync_state_poll()` | per-state-poll for AP sync | `Smpboot::sync_state_poll` |
| `arch_cpuhp_cleanup_kick_cpu()` | per-fail-cleanup | `Smpboot::cleanup_kick_cpu` |
| `arch_cpuhp_cleanup_dead_cpu()` | per-offline-cleanup | `Smpboot::cleanup_dead_cpu` |
| `native_cpu_disable()` | per-CPU-offline disable | `Smpboot::cpu_disable` |
| `cpu_disable_common()` | per-CPU-offline common | `Smpboot::cpu_disable_common` |
| `remove_siblinginfo()` | per-CPU-offline mask cleanup | `Smpboot::remove_siblinginfo` |
| `native_play_dead()` | per-CPU offline halt | `Smpboot::play_dead` |
| `mwait_play_dead()` | per-CPU MWAIT offline halt | `Smpboot::mwait_play_dead` |
| `hlt_play_dead()` | per-CPU HLT offline halt | `Smpboot::hlt_play_dead` |
| `smp_kick_mwait_play_dead()` | per-CPU MWAIT wakeup-from-dead | `Smpboot::kick_mwait_play_dead` |
| `disable_smp()` | per-bring-up-fallback UP | `Smpboot::disable_smp` |
| `announce_cpu()` | per-kernel-log node print | `Smpboot::announce_cpu` |
| `impress_friends()` | per-summary bogomips print | `Smpboot::impress_friends` |
| `STARTUP_PARALLEL_MASK` / `STARTUP_READ_APICID` | per-parallel-bringup smpboot_control encoding | `STARTUP_*` consts |
| `cpu_sibling_map` / `cpu_core_map` / `cpu_die_map` | per-CPU topology cpumask | `Topology::*_map` |
| `initial_code` / `initial_stack` | per-AP head_64.S handoff slots | `Smpboot::initial_code` / `initial_stack` |
| `real_mode_header->trampoline_start` / `trampoline_start64` | per-AP real-mode entry | `Realmode::trampoline_*` |
| `mwait_cpu_dead` | per-CPU MWAIT-state for offline | `Smpboot::MwaitCpuDead` |

### compatibility contract

REQ-1: native_smp_prepare_cpus(max_cpus):
- /* Common per-CPU mask allocation + sibling-map for CPU0 */
- smp_prepare_cpus_common().
- /* Decide based on apic_intr_mode */
- switch apic_intr_mode:
  - APIC_PIC | APIC_VIRTUAL_WIRE_NO_CONFIG: disable_smp(); return.
  - APIC_SYMMETRIC_IO_NO_ROUTING: disable_smp(); setup_percpu_clockev(); return.
  - APIC_VIRTUAL_WIRE | APIC_SYMMETRIC_IO: fall through.
- /* Per-CPU clockevent */
- x86_init.timers.setup_percpu_clockev().
- print_cpu_info(&cpu_data(0)).
- uv_system_init().
- smp_set_init_udelay().
- speculative_store_bypass_ht_init().
- snp_set_wakeup_secondary_cpu()  /* SEV-SNP path */.

REQ-2: smp_prepare_cpus_common():
- /* Mark all non-boot CPUs hotpluggable (cpu_info.cpu_index = nr_cpu_ids) */
- for cpu in 1..nr_possible: per_cpu(cpu_info.cpu_index, cpu) = nr_cpu_ids.
- /* Allocate per-CPU topology cpumasks */
- for cpu in possible:
  - zalloc_cpumask_var_node(&cpu_sibling_map, cpu).
  - zalloc_cpumask_var_node(&cpu_core_map, cpu).
  - zalloc_cpumask_var_node(&cpu_die_map, cpu).
  - zalloc_cpumask_var_node(&cpu_llc_shared_map, cpu).
  - zalloc_cpumask_var_node(&cpu_l2c_shared_map, cpu).
- set_cpu_sibling_map(0).

REQ-3: native_smp_prepare_boot_cpu():
- if !CONFIG_SMP: switch_gdt_and_percpu_base(me).
- native_pv_lock_init().

REQ-4: native_smp_cpus_done(max_cpus):
- build_sched_topology().
- nmi_selftest().
- impress_friends().
- cache_aps_init().

REQ-5: arch_cpuhp_init_parallel_bringup() — CONFIG_X86_64:
- if !x86_cpuinit.parallel_bringup: return false.
- smpboot_control = STARTUP_READ_APICID.
- return true.

REQ-6: arch_cpuhp_kick_ap_alive(cpu, tidle):
- /* Per-hotplug entry routes to smp_ops.kick_ap_alive (native_kick_ap) */
- return smp_ops.kick_ap_alive(cpu, tidle).

REQ-7: native_kick_ap(cpu, tidle):
- apicid = apic->cpu_present_to_apicid(cpu).
- lockdep_assert_irqs_enabled().
- if apicid == BAD_APICID ∨ !apic_id_valid(apicid): return -EINVAL.
- if num_online_cpus() > 1 ∧ !cpu_check_up_prepare(cpu): return -EIO.
- err = common_cpu_up(cpu, tidle).
- if err: return err.
- err = do_boot_cpu(apicid, cpu, tidle).
- if err: pr_err("do_boot_cpu failed").
- return err.

REQ-8: common_cpu_up(cpu, idle):
- alternatives_enable_smp().
- per_cpu(current_task, cpu) = idle.
- cpu_init_stack_canary(cpu, idle).
- ret = irq_init_percpu_irqstack(cpu).
- if ret: return ret.
- /* X86_32 only */
- if CONFIG_X86_32: per_cpu(cpu_current_top_of_stack, cpu) = task_top_of_stack(idle).
- return 0.

REQ-9: do_boot_cpu(apicid, cpu, idle):
- start_ip = real_mode_header->trampoline_start.
- if CONFIG_X86_64 ∧ apic->wakeup_secondary_cpu_64:
  - start_ip = real_mode_header->trampoline_start64.
- idle.thread.sp = (unsigned long)task_pt_regs(idle).
- initial_code = (unsigned long)start_secondary.
- if CONFIG_X86_32:
  - early_gdt_descr.address = get_cpu_gdt_rw(cpu).
  - initial_stack = idle.thread.sp.
- else if !(smpboot_control & STARTUP_PARALLEL_MASK):
  - smpboot_control = cpu.
- init_espfix_ap(cpu).
- announce_cpu(cpu, apicid).
- if x86_platform.legacy.warm_reset:
  - smpboot_setup_warm_reset_vector(start_ip).
  - APIC ESR clear (if integrated).
- smp_mb().
- /* Dispatch wakeup */
- if apic->wakeup_secondary_cpu_64: ret = apic->wakeup_secondary_cpu_64(apicid, start_ip, cpu).
- else if apic->wakeup_secondary_cpu: ret = apic->wakeup_secondary_cpu(apicid, start_ip, cpu).
- else: ret = wakeup_secondary_cpu_via_init(apicid, start_ip, cpu).
- if ret: arch_cpuhp_cleanup_kick_cpu(cpu).
- return ret.

REQ-10: send_init_sequence(phys_apicid):
- maxlvt = lapic_get_maxlvt().
- if APIC_INTEGRATED(boot_cpu_apic_version) ∧ maxlvt > 3: apic_write(APIC_ESR, 0); apic_read(APIC_ESR).
- /* Assert INIT */
- apic_icr_write(APIC_INT_LEVELTRIG | APIC_INT_ASSERT | APIC_DM_INIT, phys_apicid).
- safe_apic_wait_icr_idle().
- udelay(init_udelay).
- /* Deassert INIT */
- apic_icr_write(APIC_INT_LEVELTRIG | APIC_DM_INIT, phys_apicid).
- safe_apic_wait_icr_idle().

REQ-11: wakeup_secondary_cpu_via_init(phys_apicid, start_eip, cpu):
- preempt_disable().
- maxlvt = lapic_get_maxlvt().
- send_init_sequence(phys_apicid).
- mb().
- num_starts = APIC_INTEGRATED(boot_cpu_apic_version) ? 2 : 0.
- for j in 1..=num_starts:
  - if maxlvt > 3: apic_write(APIC_ESR, 0).
  - apic_read(APIC_ESR).
  - /* STARTUP IPI: vector = start_eip >> 12 */
  - apic_icr_write(APIC_DM_STARTUP | (start_eip >> 12), phys_apicid).
  - udelay(init_udelay == 0 ? 10 : 300).
  - send_status = safe_apic_wait_icr_idle().
  - udelay(init_udelay == 0 ? 10 : 200).
  - if maxlvt > 3: apic_write(APIC_ESR, 0).
  - accept_status = apic_read(APIC_ESR) & 0xEF.
  - if send_status ∨ accept_status: break.
- preempt_enable().
- return send_status | accept_status.

REQ-12: start_secondary(_unused) — AP C-entry from secondary_startup_64:
- cr4_init().
- if CONFIG_X86_32: load_cr3(swapper_pg_dir); __flush_tlb_all().
- cpu_init_exception_handling(false).
- /* Microcode load BEFORE the AP-alive sync point */
- load_ucode_ap().
- /* Sync: ALIVE -> wait for control CPU to release */
- cpuhp_ap_sync_alive().
- cpu_init().
- fpu__init_cpu().
- rcutree_report_cpu_starting(raw_smp_processor_id()).
- x86_cpuinit.early_percpu_clock_init().
- ap_starting().
- /* TSC sync vs BSP */
- check_tsc_sync_target().
- ap_calibrate_delay().
- speculative_store_bypass_ht_init().
- /* Vector allocator online with lock */
- lock_vector_lock().
- set_cpu_online(smp_processor_id(), true).
- lapic_online().
- unlock_vector_lock().
- x86_platform.nmi_init().
- local_irq_enable().
- x86_platform.setup_percpu_clockev().
- wmb().
- cpu_startup_entry(CPUHP_AP_ONLINE_IDLE).

REQ-13: ap_starting():
- this_cpu_write(mwait_cpu_dead.status, 0).
- this_cpu_write(mwait_cpu_dead.control, 0).
- apic_ap_setup().
- identify_secondary_cpu(cpuid).
- /* Topology BEFORE notify_cpu_starting */
- set_cpu_sibling_map(cpuid).
- ap_init_aperfmperf().
- wmb().
- notify_cpu_starting(cpuid).

REQ-14: ap_calibrate_delay():
- calibrate_delay().
- cpu_data(smp_processor_id()).loops_per_jiffy = loops_per_jiffy.

REQ-15: smpboot_setup_warm_reset_vector(start_eip):
- /* CMOS shutdown status + 40:67 vector for legacy warm reset */
- per-arch-specific writes to CMOS_WRITE(0xa, 0xf) and the BIOS warm-reset vector at 0x467.

REQ-16: Parallel bringup (smpboot_control encoding):
- STARTUP_READ_APICID (high bit) — AP reads its own APICID and computes its CPU index from a table.
- STARTUP_PARALLEL_MASK — distinguishes parallel vs sequential mode.
- Single-CPU mode: smpboot_control = cpu (sequential per-AP slot).

REQ-17: CPU-offline path (native_cpu_disable + native_play_dead):
- native_cpu_disable():
  - ret = lapic_offline().
  - if ret: return ret.
  - cpu_disable_common().
  - return 0.
- cpu_disable_common():
  - remove_siblinginfo(cpu).
  - remove_cpu_from_maps(cpu).
  - lock_vector_lock(); set_cpu_online(cpu, false); unlock_vector_lock().
  - fixup_irqs().
  - lapic_offline().
- native_play_dead():
  - play_dead_common().
  - tboot_shutdown(TB_SHUTDOWN_WFS).
  - mwait_play_dead(...) or hlt_play_dead().

REQ-18: mwait_play_dead(eax_hint):
- /* Wait for wakeup via cache-line poll using MONITOR/MWAIT */
- md = this_cpu_ptr(&mwait_cpu_dead).
- md->control = CPUDEAD_MWAIT_WAIT.
- loop:
  - mb(); clflush(&md->status); mb().
  - if md->status == CPUDEAD_MWAIT_KEXEC_HLT: break.
  - __monitor((void *)&md->status, 0, 0).
  - mb().
  - __mwait(eax_hint, 0).
- /* Final HLT (kexec) */
- hlt_play_dead().

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `init_sipi_sipi_ordering` | INVARIANT | per-wakeup_via_init: INIT-assert -> INIT-deassert -> STARTUP IPI x num_starts. |
| `start_eip_aligned_4k` | INVARIANT | per-do_boot_cpu: start_ip in real_mode_header range; STARTUP vector = start_ip >> 12. |
| `apicid_validated` | INVARIANT | per-native_kick_ap: apicid != BAD_APICID ∧ apic_id_valid. |
| `microcode_before_alive_sync` | INVARIANT | per-start_secondary: load_ucode_ap precedes cpuhp_ap_sync_alive. |
| `tsc_sync_before_online` | INVARIANT | per-start_secondary: check_tsc_sync_target precedes set_cpu_online. |
| `vector_lock_around_online` | INVARIANT | per-start_secondary: lock_vector_lock around set_cpu_online + lapic_online. |
| `sibling_map_before_notify` | INVARIANT | per-ap_starting: set_cpu_sibling_map precedes notify_cpu_starting. |
| `parallel_bringup_smpboot_control` | INVARIANT | per-arch_cpuhp_init_parallel_bringup: smpboot_control = STARTUP_READ_APICID. |
| `mwait_kexec_status_break` | INVARIANT | per-mwait_play_dead: status == CPUDEAD_MWAIT_KEXEC_HLT exits loop. |
| `cleanup_on_kick_fail` | INVARIANT | per-do_boot_cpu: ret != 0 ⟹ arch_cpuhp_cleanup_kick_cpu called. |

### Layer 2: TLA+

`arch/x86/smpboot.tla`:
- States: PRESENT, KICKED, INIT_ASSERTED, INIT_DEASSERTED, STARTUP_SENT, TRAMPOLINE, HEAD_64, START_SECONDARY, ALIVE, ONLINE, IDLE, DEAD.
- Properties:
  - `safety_no_double_online` — per-CPU: set_cpu_online(true) issued at most once between BSP-reset and cpu_disable.
  - `safety_init_sipi_sipi_sequence` — per-wakeup_via_init: INIT_DEASSERTED precedes any STARTUP_SENT.
  - `safety_microcode_before_alive` — per-AP: load_ucode_ap state precedes ALIVE.
  - `safety_tsc_sync_before_online` — per-AP: TSC_SYNC_OK precedes ONLINE.
  - `safety_vector_lock_held` — per-AP: vector_lock held during transition to ONLINE.
  - `liveness_kicked_ap_reaches_online_or_fail` — per-do_boot_cpu: KICKED ~> (ONLINE ∨ FAILED) within bounded steps.
  - `liveness_offline_ap_reaches_dead` — per-native_cpu_disable: ONLINE ~> DEAD.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Smpboot::prepare_cpus_common` post: per-CPU masks allocated, set_cpu_sibling_map(0) called | `Smpboot::prepare_cpus_common` |
| `Smpboot::do_boot_cpu` pre: real_mode_header set ∧ initial_code = start_secondary | `Smpboot::do_boot_cpu` |
| `Smpboot::wakeup_via_init` post: send_status / accept_status returned to caller | `Smpboot::wakeup_via_init` |
| `Smpboot::start_secondary` post: cpu_online(cpu) ∧ lapic_online ∧ irqs enabled | `Smpboot::start_secondary` |
| `Smpboot::ap_starting` post: sibling/core/die masks include cpuid | `Smpboot::ap_starting` |
| `Smpboot::cpu_disable_common` post: !cpu_online(cpu) ∧ siblinginfo cleared | `Smpboot::cpu_disable_common` |
| `Smpboot::mwait_play_dead` invariant: progress requires md.status == CPUDEAD_MWAIT_KEXEC_HLT | `Smpboot::mwait_play_dead` |
| `Smpboot::kick_ap` post: ret == 0 ⟹ AP enters head_64.S trampoline | `Smpboot::kick_ap` |

### Layer 4: Verus/Creusot functional

`Per-AP bringup: native_kick_ap -> common_cpu_up -> do_boot_cpu -> {wakeup_via_init | wakeup_secondary_cpu | wakeup_secondary_cpu_64} -> trampoline -> head_64.S secondary_startup_64 -> start_secondary -> ap_starting -> check_tsc_sync_target -> set_cpu_online -> cpu_startup_entry(CPUHP_AP_ONLINE_IDLE)` semantic equivalence: per-Documentation/x86/booting-dt.rst, per-Intel SDM Vol.3A §8.4 (MP Initialization), per-AMD64 APM Vol.2 §16.7.

### hardening

(Inherits row-1 features from `arch/x86/00-overview.md` § Hardening.)

SMP-bringup reinforcement:

- **Per-AP microcode loaded BEFORE cpuhp_ap_sync_alive** — defense against per-stale-ucode running with new MSRs.
- **Per-INIT-assert + INIT-deassert + STARTUP IPI strict ordering** — defense against per-AP-stuck-in-wait-for-SIPI.
- **Per-APIC_ESR cleared pre and post STARTUP** — defense against per-stale APIC error masking real failures.
- **Per-init_udelay tuned (smp_set_init_udelay vs cpu_init_udelay= cmdline)** — defense against per-too-fast-IPI dropping startups.
- **Per-vector_lock held around set_cpu_online + lapic_online** — defense against per-half-valid vector space during concurrent irq teardown.
- **Per-set_cpu_sibling_map called BEFORE notify_cpu_starting** — defense against per-scheduler-seeing-empty-mask.
- **Per-check_tsc_sync_target before set_cpu_online** — defense against per-clock-skew-visible-to-userspace.
- **Per-fixup_irqs in cpu_disable_common** — defense against per-IRQ-stranded on offlined CPU.
- **Per-real_mode_header trampoline pages locked + RX-only after init** — defense against per-trampoline overwrite (attacker primitive on shared real-mode page).
- **Per-arch_cpuhp_cleanup_kick_cpu on kick failure** — defense against per-leaked warm-reset vector + dangling AP state.
- **Per-mwait_cpu_dead.status checked with CLFLUSH + MB fence** — defense against per-cache-stale wakeup detection.
- **Per-snp_set_wakeup_secondary_cpu / TDX wakeup gated** — defense against per-confidential-VM untrusted wakeup vector.

