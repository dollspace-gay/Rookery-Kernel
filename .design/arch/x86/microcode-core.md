# Tier-3: arch/x86/kernel/cpu/microcode/core.c — x86 microcode loader core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: arch/x86/00-overview.md
upstream-paths:
  - arch/x86/kernel/cpu/microcode/core.c (~934 lines)
  - arch/x86/kernel/cpu/microcode/internal.h
  - arch/x86/include/asm/microcode.h
  - arch/x86/kernel/cpu/common.c (microcode_check)
-->

## Summary

The x86 microcode loader updates per-CPU microcode revisions either **early** (from initrd, before SMP), at **resume** (BSP, via syscore), or **late** (post-boot, via `/sys/devices/system/cpu/microcode/reload`). Per-CPU state lives in `struct ucode_cpu_info` (cpu_sig {sig, pf, rev} + opaque `*mc` blob). Vendor dispatch is via `struct microcode_ops`: Intel (`init_intel_microcode`) or AMD (`init_amd_microcode`) installs `request_microcode_fw`, `apply_microcode`, `collect_cpu_info`, optional `stage_microcode` / `finalize_late_load`. Per-late-load orchestration uses `stop_machine_cpuslocked` with a primary/secondary SMT rendezvous: only the primary thread of each core invokes `apply_microcode`; siblings spin or follow via SCTRL_APPLY. Per-`microcode_check` walks `cpuinfo_x86.x86_capability` before vs after and warns when CPUID features changed. Per-security boundary: AMD `final_levels` lock out specific corrupt revisions; Intel `min_req_ver` (in `microcode_header_intel`) enforces minimum revision before applying; `force_minrev` cmdline overrides via `CONFIG_MICROCODE_LATE_FORCE_MINREV`. Critical for: speculative-execution mitigations (Spectre / MDS / RetBleed / Downfall / SRSO), errata fixes, secure-firmware-update lifecycle.

This Tier-3 covers `arch/x86/kernel/cpu/microcode/core.c` (~934 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ucode_cpu_info` | per-CPU sig + mc blob | `UcodeCpuInfo` |
| `struct cpu_signature` | per-CPU {sig, pf, rev} | `CpuSignature` |
| `struct microcode_ops` | per-vendor dispatch table | `MicrocodeOps` |
| `enum ucode_state` | per-load result | `UcodeState` |
| `struct microcode_ctrl` | per-CPU rendezvous slot | `MicrocodeCtrl` |
| `struct early_load_data` | per-boot old/new rev | `EarlyLoadData` |
| `load_ucode_bsp()` | per-BSP early load | `Microcode::load_bsp` |
| `load_ucode_ap()` | per-AP early load | `Microcode::load_ap` |
| `microcode_loader_disabled()` | per-policy gate | `Microcode::is_disabled` |
| `amd_check_current_patch_level()` | per-AMD final-levels check | `Microcode::amd_check_final_levels` |
| `early_parse_cmdline()` | per-`microcode=` parse | `Microcode::parse_cmdline` |
| `find_microcode_in_initrd()` | per-initrd cpio scan | `Microcode::find_in_initrd` |
| `reload_early_microcode()` | per-resume reload | `Microcode::reload_early` |
| `microcode_bsp_resume()` / `microcode_bsp_syscore_resume()` | per-resume callback | `Microcode::bsp_resume` |
| `load_late_locked()` | per-/sys reload driver | `Microcode::load_late_locked` |
| `load_late_stop_cpus(is_safe)` | per-stop_machine wrapper | `Microcode::load_late_stop_cpus` |
| `setup_cpus()` | per-online/offline mask setup | `Microcode::setup_cpus` |
| `load_cpus_stopped()` | per-stop_machine fn | `Microcode::load_cpus_stopped` |
| `microcode_update_handler()` | per-NMI/direct dispatch | `Microcode::update_handler` |
| `microcode_nmi_handler()` | per-sibling NMI safe entry | `Microcode::nmi_handler` |
| `microcode_offline_nmi_handler()` | per-soft-offline stub | `Microcode::offline_nmi_handler` |
| `load_primary()` / `__load_primary()` | per-primary-thread update | `Microcode::load_primary` |
| `load_secondary()` | per-sibling apply or copy | `Microcode::load_secondary` |
| `wait_for_cpus()` / `wait_for_ctrl()` | per-rendezvous spinwait | `Microcode::wait_for_cpus` / `_ctrl` |
| `kick_offline_cpus()` / `release_offline_cpus()` | per-NMI to soft-offline siblings | `Microcode::kick_offline` / `release_offline` |
| `reload_store()` | per-sysfs WO `reload` attr | `Microcode::reload_store` |
| `version_show()` / `processor_flags_show()` | per-sysfs RO attrs | `Microcode::version_show` / `flags_show` |
| `mc_cpu_online()` / `mc_cpu_down_prep()` | per-cpuhp callbacks | `Microcode::cpu_online` / `cpu_down_prep` |
| `microcode_init()` (late_initcall) | per-bootstrap | `Microcode::init` |
| `microcode_check()` (in common.c) | per-post-update CPUID validation | `Microcode::check` |
| `force_minrev` | per-cmdline override | shared |
| `final_levels[]` | per-AMD bricked-revision list | shared |

## Compatibility contract

REQ-1: struct ucode_cpu_info:
- cpu_sig: per-CPU signature `{ sig: u32, pf: u32, rev: u32 }`.
  - sig: per-Family/Model/Stepping (Intel CPUID 0x1.eax; AMD equivalent).
  - pf: per-processor flags (Intel platform-id from MSR 0x17; AMD = 0).
  - rev: per-current microcode revision (Intel MSR 0x8B; AMD MSR_AMD64_PATCH_LEVEL = 0x8B).
- mc: per-CPU pointer to chosen microcode blob (vendor-specific layout); NULL ⇒ no patch selected.
- Array: `struct ucode_cpu_info ucode_cpu_info[NR_CPUS]`.

REQ-2: struct microcode_ops:
- request_microcode_fw(cpu, dev): per-vendor firmware request (`/lib/firmware/intel-ucode/...` or `/lib/firmware/amd-ucode/...`). Returns `enum ucode_state`.
- microcode_fini_cpu(cpu): per-vendor cleanup at cpu_down_prep.
- apply_microcode(cpu): per-vendor MSR write (Intel: MSR_IA32_UCODE_WRITE; AMD: MSR_AMD64_PATCH_LOADER). Runs on the target CPU.
- stage_microcode(): per-vendor optional pre-stage to staging device (Intel only).
- collect_cpu_info(cpu, csig): per-vendor signature collection (Intel: CPUID 1 + MSR 0x17 + MSR 0x8B; AMD: family-based).
- finalize_late_load(result): per-vendor optional post-load hook.
- Bitfields: nmi_safe : 1 — apply path is NMI-safe; use_nmi : 1 — drive rendezvous via NMI; use_staging : 1 — call stage_microcode.

REQ-3: enum ucode_state:
- UCODE_OK = 0  // no change required.
- UCODE_NEW       // new revision available, no minrev check.
- UCODE_NEW_SAFE  // new revision available, minrev check passed.
- UCODE_UPDATED   // apply_microcode succeeded (revision incremented).
- UCODE_NFOUND    // no matching patch in firmware.
- UCODE_ERROR     // generic failure.
- UCODE_TIMEOUT   // rendezvous timeout.
- UCODE_OFFLINE   // soft-offline sibling participated trivially.

REQ-4: microcode_loader_disabled():
- if dis_ucode_ldr: return true (sticky).
- if !cpuid_feature(): dis_ucode_ldr = true; return true  // pre-CPUID CPU.
- hypervisor_present = native_cpuid_ecx(1) & BIT(31)  // hypervisor reserved bit.
- if (hypervisor_present && !CONFIG_MICROCODE_DBG) || amd_check_current_patch_level(): dis_ucode_ldr = true.
- return dis_ucode_ldr.

REQ-5: amd_check_current_patch_level():
- if vendor != AMD: false.
- native_rdmsr(MSR_AMD64_PATCH_LEVEL=0x8B, lvl, dummy).
- final_levels[] = { 0x01000098, 0x0100009f, 0x010000af, 0 }.
- for lvl_i in final_levels: if lvl == lvl_i: return true (bricked / locked-final; updates forbidden).
- return false.

REQ-6: early_parse_cmdline():
- Parses boot_command_line option `microcode=`:
  - `force_minrev` ⇒ `force_minrev = true` (allow updates that lower below min_req_ver — debug only).
  - `dis_ucode_ldr` ⇒ disable loader.
  - CONFIG_MICROCODE_DBG: `base_rev=<hex>` ⇒ override boot-time rev for testing.
- Compat: `dis_ucode_ldr` as bool option also disables.

REQ-7: load_ucode_bsp() (early):
- early_parse_cmdline().
- if microcode_loader_disabled(): return.
- cpuid_1_eax = native_cpuid_eax(1).
- switch vendor:
  - INTEL: family < 6 → return; else load_ucode_intel_bsp(&early_data).
  - AMD: family < 0x10 → return; else load_ucode_amd_bsp(&early_data, cpuid_1_eax).
  - else: return.

REQ-8: load_ucode_ap() (early):
- if dis_ucode_ldr: return  // no microcode_loader_disabled() — .init section concern; BSP already parsed.
- cpuid_1_eax = native_cpuid_eax(1).
- switch vendor: INTEL ≥ 6 → load_ucode_intel_ap(); AMD ≥ 0x10 → load_ucode_amd_ap(cpuid_1_eax).

REQ-9: find_microcode_in_initrd(path) -> struct cpio_data:
- if !CONFIG_BLK_DEV_INITRD: return empty.
- start = initrd_start (after reserve_initrd) OR boot_params.ramdisk_image + PAGE_OFFSET (early BSP).
- size = boot_params.ramdisk_size (extended on x86_64 via ext_ramdisk_size).
- return find_cpio_data(path, (void *)start, size, NULL).

REQ-10: microcode_init() (late_initcall):
- if microcode_loader_disabled(): -EINVAL.
- vendor dispatch: INTEL → init_intel_microcode; AMD → init_amd_microcode; else log "no support".
- if !microcode_ops: -ENODEV.
- pr_info_once revision strings.
- microcode_fdev = faux_device_create("microcode")  // for firmware_request.
- dev_root = bus_get_dev_root(&cpu_subsys); sysfs_create_group(&dev_root.kobj, &cpu_root_microcode_group)  // /sys/devices/system/cpu/microcode/.
- register_syscore(&mc_syscore)  // bsp_resume hook.
- cpuhp_setup_state(CPUHP_AP_ONLINE_DYN, "x86/microcode:online", mc_cpu_online, mc_cpu_down_prep).

REQ-11: mc_cpu_online(cpu):
- uci = ucode_cpu_info + cpu.
- memset(uci, 0).
- microcode_ops->collect_cpu_info(cpu, &uci->cpu_sig).
- cpu_data(cpu).microcode = uci->cpu_sig.rev.
- if cpu == 0: boot_cpu_data.microcode = uci->cpu_sig.rev.
- sysfs_create_group(&dev->kobj, &mc_attr_group)  // /sys/devices/system/cpu/cpuN/microcode/{version,processor_flags}.

REQ-12: mc_cpu_down_prep(cpu):
- microcode_fini_cpu(cpu)  // dispatches ops->microcode_fini_cpu.
- sysfs_remove_group(&dev->kobj, &mc_attr_group).

REQ-13: reload_store(dev, attr, buf, size):
- val = kstrtoul(buf); if err || val != 1: -EINVAL.
- cpus_read_lock().
- ret = load_late_locked().
- cpus_read_unlock().
- return ret ? ret : size.
- Gated by CONFIG_MICROCODE_LATE_LOADING.
- Permissions: DEVICE_ATTR_WO → 0200 root-only.

REQ-14: load_late_locked():
- if !setup_cpus(): -EBUSY  // primary threads offline.
- switch microcode_ops->request_microcode_fw(0, &microcode_fdev->dev):
  - UCODE_NEW: load_late_stop_cpus(is_safe=false).
  - UCODE_NEW_SAFE: load_late_stop_cpus(is_safe=true).
  - UCODE_NFOUND: -ENOENT.
  - UCODE_OK: 0.
  - default: -EBADFD.

REQ-15: setup_cpus():
- allow_smt_offline = ops->nmi_safe || (ops->use_nmi && apic->nmi_to_offline_cpu).
- cpumask_clear(&cpu_offline_mask).
- for each cpu in cpu_present_mask AND cpus_booted_once_mask:
  - if !cpu_online(cpu):
    - if primary_thread(cpu) || !allow_smt_offline: pr_err "CPU %u not online, loading aborted"; return false.
    - else: cpumask_set_cpu(cpu, &cpu_offline_mask); per_cpu(ucode_ctrl, cpu) = {SCTRL_WAIT, -1, ...}.
  - else:
    - ctrl.ctrl_cpu = first sibling thread; per_cpu(ucode_ctrl, cpu) = ctrl.
- return true.

REQ-16: load_late_stop_cpus(is_safe):
- if !is_safe: pr_err "Late microcode loading without minimal revision check."
- if ops->use_staging: ops->stage_microcode().
- atomic_set(&late_cpus_in, num_online_cpus()).
- atomic_set(&offline_in_nmi, 0).
- loops_per_usec = loops_per_jiffy / (TICK_NSEC / 1000).
- store_cpu_caps(&prev_info).
- if ops->use_nmi: static_branch_enable_cpuslocked(&microcode_nmi_handler_enable).
- stop_machine_cpuslocked(load_cpus_stopped, NULL, cpu_online_mask).
- if ops->use_nmi: static_branch_disable.
- Analyze per_cpu(ucode_ctrl.result) — count updated/timedout/siblings/offline/failed.
- if ops->finalize_late_load: finalize_late_load(!updated).
- if !updated && !failed && !timedout: return 0.
- if !is_safe || failed || timedout: add_taint(TAINT_CPU_OUT_OF_SPEC, LOCKDEP_STILL_OK).
- pr_info revision: 0x%x -> 0x%x.
- microcode_check(&prev_info).
- return (updated + siblings == num_online_cpus()) ? 0 : -EIO.

REQ-17: load_cpus_stopped(unused) (stop_machine fn):
- if ops->use_nmi:
  - this_cpu_write(ucode_ctrl.nmi_enabled, true).
  - apic->send_IPI(smp_processor_id(), NMI_VECTOR)  // self-NMI; rendezvous happens in NMI context.
- else:
  - microcode_update_handler()  // direct invocation.

REQ-18: microcode_update_handler():
- cpu = raw_smp_processor_id().
- if ucode_ctrl.ctrl_cpu == cpu: load_primary(cpu)  // I am the primary.
- else: load_secondary(cpu).
- touch_nmi_watchdog().

REQ-19: load_primary(cpu):
- nr_offl = cpumask_weight(&cpu_offline_mask).
- if cpu == 0 && nr_offl: proceed = kick_offline_cpus(nr_offl)  // self+offline siblings via NMI.
- if proceed: __load_primary(cpu).
- if cpu == 0 && nr_offl: release_offline_cpus().

REQ-20: __load_primary(cpu):
- if !wait_for_cpus(&late_cpus_in):
  - this_cpu_write(ucode_ctrl.result, UCODE_TIMEOUT); pr_err; return.
- ret = ops->apply_microcode(cpu).
- this_cpu_write(ucode_ctrl.result, ret); this_cpu_write(ucode_ctrl.ctrl, SCTRL_DONE).
- ctrl = (ret == UCODE_UPDATED || ret == UCODE_OK) ? SCTRL_APPLY : SCTRL_DONE.
- for each sibling in topology_sibling_cpumask(cpu) (excluding self): per_cpu(ucode_ctrl.ctrl, sibling) = ctrl.

REQ-21: load_secondary(cpu):
- ctrl_cpu = raw_cpu_read(ucode_ctrl.ctrl_cpu).
- if !load_secondary_wait(ctrl_cpu): pr_err_once; return.
- if ucode_ctrl.ctrl == SCTRL_APPLY: ret = ops->apply_microcode(cpu).
- else: ret = per_cpu(ucode_ctrl.result, ctrl_cpu)  // copy primary's result.
- this_cpu_write(ucode_ctrl.result, ret).
- this_cpu_write(ucode_ctrl.ctrl, SCTRL_DONE).

REQ-22: load_secondary_wait(ctrl_cpu):
- wait_for_cpus(&late_cpus_in): if false → UCODE_TIMEOUT.
- wait_for_ctrl(): if true → return true.
- panic("Microcode load: Primary CPU %d timed out").

REQ-23: wait_for_cpus(cnt):
- raw_atomic_dec_return(cnt).
- spin up to USEC_PER_SEC; touch_nmi_watchdog every millisecond when !use_nmi.
- if cnt reaches 0: return true.
- on timeout: raw_atomic_inc(cnt) to gate latecomers; return false.

REQ-24: wait_for_ctrl():
- spin up to USEC_PER_SEC; return true when ucode_ctrl.ctrl != SCTRL_WAIT.
- on timeout: return false.

REQ-25: kick_offline_cpus(nr_offl):
- for each cpu in cpu_offline_mask: per_cpu(ucode_ctrl.nmi_enabled, cpu) = true; apic_send_nmi_to_offline_cpu(cpu).
- spin up to USEC_PER_SEC/2; return atomic_read(&offline_in_nmi) == nr_offl.

REQ-26: release_offline_cpus():
- for each cpu in cpu_offline_mask: per_cpu(ucode_ctrl.ctrl, cpu) = SCTRL_DONE.

REQ-27: microcode_nmi_handler() (NMI entry):
- if !raw_cpu_read(ucode_ctrl.nmi_enabled): return false.
- raw_cpu_write(ucode_ctrl.nmi_enabled, false).
- return microcode_update_handler().
- noinstr: instrumentation forbidden until primary completes — IRET from #INT3/#DB/#PF re-enables NMI, breaking rendezvous.

REQ-28: microcode_offline_nmi_handler() (soft-offline stub):
- if !ucode_ctrl.nmi_enabled: return.
- ucode_ctrl.nmi_enabled = false.
- ucode_ctrl.result = UCODE_OFFLINE.
- raw_atomic_inc(&offline_in_nmi).
- wait_for_ctrl()  // park.

REQ-29: microcode_check(prev_info) (security boundary):
- perf_check_microcode(); amd_check_microcode().
- store_cpu_caps(&curr_info).
- if memcmp(prev_info->x86_capability, curr_info.x86_capability) == 0: return.
- pr_warn "CPU features have changed after loading microcode, but might not take effect."
- pr_warn "Please consider either early loading through initrd/built-in or a potential BIOS update."

REQ-30: microcode_bsp_resume() (syscore):
- cpu = smp_processor_id(); uci = ucode_cpu_info + cpu.
- if uci->mc: ops->apply_microcode(cpu).
- else: reload_early_microcode(cpu)  // vendor-specific re-application from saved blob.

REQ-31: Security boundary — sig + revision check:
- Intel patch header (`struct microcode_header_intel`): `min_req_ver` field; loader refuses to downgrade or sidegrade below it unless `force_minrev` is set (debug-only).
- AMD `final_levels[]`: any current rev ∈ {0x01000098, 0x0100009f, 0x010000af} disables loader entirely.
- `request_microcode_fw` selects patch by matching `cpu_sig.sig` (family/model/stepping) AND `cpu_sig.pf` (platform-id mask) AND patch.rev > current.rev (minrev path) OR per `force_minrev` policy.
- `apply_microcode` is the only path that writes the vendor MSR; runs on the target CPU only.

## Acceptance Criteria

- [ ] AC-1: microcode_loader_disabled returns true after dis_ucode_ldr cmdline; subsequent load paths no-op.
- [ ] AC-2: amd_check_current_patch_level returns true for any rev in final_levels; loader disabled.
- [ ] AC-3: load_ucode_bsp on Intel family < 6 returns without loading; on AMD family < 0x10 returns without loading.
- [ ] AC-4: microcode_init: late_initcall runs; sysfs `/sys/devices/system/cpu/microcode/{reload,version,processor_flags}` paths exist with correct permissions (reload=0200, version=0444, processor_flags=0444).
- [ ] AC-5: reload_store: writing "0" returns -EINVAL; writing "1" triggers load_late_locked; root-only.
- [ ] AC-6: load_late_locked: request_microcode_fw returning UCODE_NEW_SAFE drives load_late_stop_cpus(is_safe=true); UCODE_NEW drives (is_safe=false) with TAINT_CPU_OUT_OF_SPEC.
- [ ] AC-7: setup_cpus refuses load (-EBUSY) when any primary thread is offline.
- [ ] AC-8: setup_cpus allows soft-offline SMT siblings when ops->nmi_safe OR (ops->use_nmi && apic->nmi_to_offline_cpu).
- [ ] AC-9: load_late_stop_cpus: stop_machine_cpuslocked drives all online CPUs to load_cpus_stopped exactly once; primary applies; siblings copy result or apply per SCTRL_APPLY.
- [ ] AC-10: load_secondary_wait timeout on primary triggers panic("Microcode load: Primary CPU %d timed out").
- [ ] AC-11: microcode_check: prev vs curr x86_capability bitmap differs → pr_warn lines emitted.
- [ ] AC-12: ops->apply_microcode invoked exclusively on the target CPU (per ucode_ctrl.ctrl_cpu == self for primaries; this_cpu_read on secondaries).
- [ ] AC-13: microcode_bsp_resume: uci->mc != NULL → ops->apply_microcode; NULL → reload_early_microcode.
- [ ] AC-14: cpuhp_setup_state(CPUHP_AP_ONLINE_DYN): mc_cpu_online populates ucode_cpu_info[cpu].cpu_sig; mc_cpu_down_prep invokes ops->microcode_fini_cpu.
- [ ] AC-15: force_minrev cmdline override allows below-min_req_ver patches; absent → such patches return UCODE_ERROR / UCODE_NFOUND.
- [ ] AC-16: NMI path: ops->use_nmi=true enables microcode_nmi_handler_enable static branch; disables after stop_machine completes.

## Architecture

```
struct CpuSignature {
  sig: u32,
  pf: u32,
  rev: u32,
}

struct UcodeCpuInfo {
  cpu_sig: CpuSignature,
  mc: Option<*const ()>,                 // vendor-opaque microcode blob
}

enum UcodeState {
  Ok       = 0,
  New      = 1,
  NewSafe  = 2,
  Updated  = 3,
  NFound   = 4,
  Error    = 5,
  Timeout  = 6,
  Offline  = 7,
}

struct MicrocodeOps {
  request_microcode_fw: fn(cpu: i32, dev: *Device) -> UcodeState,
  microcode_fini_cpu: Option<fn(cpu: i32)>,
  apply_microcode: fn(cpu: i32) -> UcodeState,
  stage_microcode: Option<fn()>,
  collect_cpu_info: fn(cpu: i32, csig: *CpuSignature) -> i32,
  finalize_late_load: Option<fn(result: i32)>,
  nmi_safe: bool,
  use_nmi: bool,
  use_staging: bool,
}

enum SiblingCtrl {
  Wait,
  Apply,
  Done,
}

struct MicrocodeCtrl {
  ctrl: SiblingCtrl,
  result: UcodeState,
  ctrl_cpu: u32,
  nmi_enabled: bool,
}

struct EarlyLoadData {
  old_rev: u32,
  new_rev: u32,
}

const FINAL_LEVELS_AMD: [u32; 3] = [0x01000098, 0x0100009f, 0x010000af];

static MICROCODE_OPS: AtomicPtr<MicrocodeOps>;
static DIS_UCODE_LDR: AtomicBool;
static FORCE_MINREV: AtomicBool;                  // CONFIG_MICROCODE_LATE_FORCE_MINREV
static HYPERVISOR_PRESENT: AtomicBool;
static UCODE_CPU_INFO: [UcodeCpuInfo; NR_CPUS];
static UCODE_CTRL: PerCpu<MicrocodeCtrl>;
static LATE_CPUS_IN: AtomicI32;
static OFFLINE_IN_NMI: AtomicI32;
static CPU_OFFLINE_MASK: CpuMask;
static LOOPS_PER_USEC: AtomicU32;
static EARLY_DATA: EarlyLoadData;
```

`Microcode::load_bsp()` (early, BSP):
1. parse_cmdline().
2. if is_disabled(): return.
3. cpuid_1_eax = native_cpuid_eax(1).
4. match vendor:
   - Intel: if family(cpuid_1_eax) < 6 { return }; load_ucode_intel_bsp(&EARLY_DATA).
   - Amd: if family(cpuid_1_eax) < 0x10 { return }; load_ucode_amd_bsp(&EARLY_DATA, cpuid_1_eax).
   - else: return.

`Microcode::load_ap()` (early, AP):
1. if DIS_UCODE_LDR: return.
2. cpuid_1_eax = native_cpuid_eax(1).
3. match vendor:
   - Intel: family >= 6 ⇒ load_ucode_intel_ap().
   - Amd: family >= 0x10 ⇒ load_ucode_amd_ap(cpuid_1_eax).

`Microcode::is_disabled() -> bool`:
1. if DIS_UCODE_LDR.load(): return true.
2. if !cpuid_feature(): DIS_UCODE_LDR.store(true); return true.
3. HYPERVISOR_PRESENT.store(native_cpuid_ecx(1) & BIT(31) != 0).
4. if (HYPERVISOR_PRESENT && !cfg!(MICROCODE_DBG)) || amd_check_final_levels(): DIS_UCODE_LDR.store(true).
5. DIS_UCODE_LDR.load().

`Microcode::amd_check_final_levels() -> bool`:
1. if vendor != Amd: return false.
2. native_rdmsr(MSR_AMD64_PATCH_LEVEL=0x8B, &lvl, &dummy).
3. for f in FINAL_LEVELS_AMD: if lvl == f: return true.
4. false.

`Microcode::init()` (late_initcall):
1. if is_disabled(): return Err(-EINVAL).
2. ops = match boot_cpu_vendor() { Intel => init_intel_microcode(), Amd => init_amd_microcode(), _ => { pr_err; None } }.
3. if ops.is_none(): return Err(-ENODEV).
4. MICROCODE_OPS.store(ops).
5. pr_info_once!("Current revision: 0x{:08x}", EARLY_DATA.new_rev.or(EARLY_DATA.old_rev)).
6. if EARLY_DATA.new_rev != 0: pr_info_once!("Updated early from: 0x{:08x}", EARLY_DATA.old_rev).
7. microcode_fdev = faux_device_create("microcode")?.
8. dev_root = bus_get_dev_root(&cpu_subsys); sysfs_create_group(&dev_root.kobj, &cpu_root_microcode_group)?.
9. register_syscore(&MC_SYSCORE).
10. cpuhp_setup_state(CPUHP_AP_ONLINE_DYN, "x86/microcode:online", mc_cpu_online, mc_cpu_down_prep).

`Microcode::cpu_online(cpu) -> i32`:
1. uci = &mut UCODE_CPU_INFO[cpu].
2. *uci = UcodeCpuInfo::default().
3. (ops.collect_cpu_info)(cpu, &mut uci.cpu_sig).
4. cpu_data(cpu).microcode = uci.cpu_sig.rev.
5. if cpu == 0: boot_cpu_data.microcode = uci.cpu_sig.rev.
6. sysfs_create_group(&dev.kobj, &MC_ATTR_GROUP).

`Microcode::reload_store(buf) -> Result<()>`:
1. val = kstrtoul(buf)?; if val != 1: return Err(-EINVAL).
2. cpus_read_lock().
3. ret = load_late_locked().
4. cpus_read_unlock().
5. ret.

`Microcode::load_late_locked() -> Result<()>`:
1. if !setup_cpus(): return Err(-EBUSY).
2. state = (ops.request_microcode_fw)(0, &microcode_fdev.dev).
3. match state:
   - UcodeState::New: load_late_stop_cpus(is_safe=false).
   - UcodeState::NewSafe: load_late_stop_cpus(is_safe=true).
   - UcodeState::NFound: Err(-ENOENT).
   - UcodeState::Ok: Ok(()).
   - _ : Err(-EBADFD).

`Microcode::setup_cpus() -> bool`:
1. allow_smt_offline = ops.nmi_safe || (ops.use_nmi && apic.nmi_to_offline_cpu.is_some()).
2. CPU_OFFLINE_MASK.clear().
3. for cpu in cpu_present_mask AND cpus_booted_once_mask:
   - if !cpu_online(cpu):
     - if topology_is_primary_thread(cpu) || !allow_smt_offline: pr_err!("CPU {} not online, loading aborted", cpu); return false.
     - else: CPU_OFFLINE_MASK.set(cpu); UCODE_CTRL.set(cpu, { ctrl: Wait, result: -1, ctrl_cpu: 0, nmi_enabled: false }).
   - else:
     - ctrl_cpu = topology_sibling_cpumask(cpu).first().
     - UCODE_CTRL.set(cpu, { ctrl: Wait, result: -1, ctrl_cpu, nmi_enabled: false }).
4. true.

`Microcode::load_late_stop_cpus(is_safe) -> Result<()>`:
1. if !is_safe: pr_err!("Late microcode loading without minimal revision check.").
2. if ops.use_staging: (ops.stage_microcode)().
3. LATE_CPUS_IN.store(num_online_cpus()).
4. OFFLINE_IN_NMI.store(0).
5. LOOPS_PER_USEC.store(loops_per_jiffy / (TICK_NSEC / 1000)).
6. old_rev = boot_cpu_data.microcode.
7. store_cpu_caps(&mut prev_info).
8. if ops.use_nmi: static_branch_enable_cpuslocked(&microcode_nmi_handler_enable).
9. stop_machine_cpuslocked(load_cpus_stopped, None, cpu_online_mask).
10. if ops.use_nmi: static_branch_disable_cpuslocked.
11. Analyze counters from UCODE_CTRL: updated / siblings / timedout / offline / failed.
12. if ops.finalize_late_load.is_some(): (ops.finalize_late_load)(if updated == 0 { 1 } else { 0 }).
13. if updated == 0:
    - if failed == 0 && timedout == 0: return Ok(()).
    - if offline < CPU_OFFLINE_MASK.weight(): pr_warn!("{} offline siblings did not respond.", ...); return Err(-EIO).
    - pr_err!("update failed: {} CPUs failed {} CPUs timed out", failed, timedout); return Err(-EIO).
14. if !is_safe || failed > 0 || timedout > 0: add_taint(TAINT_CPU_OUT_OF_SPEC, LOCKDEP_STILL_OK).
15. pr_info!("load: updated on {} primary CPUs with {} siblings", updated, siblings).
16. pr_info!("revision: 0x{:x} -> 0x{:x}", old_rev, boot_cpu_data.microcode).
17. microcode_check(&prev_info).
18. if updated + siblings == num_online_cpus(): Ok(()) else Err(-EIO).

`Microcode::load_cpus_stopped(_unused) -> i32` (stop_machine fn):
1. if ops.use_nmi:
   - this_cpu_write(UCODE_CTRL.nmi_enabled, true).
   - apic.send_IPI(smp_processor_id(), NMI_VECTOR).
2. else: update_handler().
3. 0.

`Microcode::update_handler() -> bool`:
1. cpu = raw_smp_processor_id().
2. if raw_cpu_read(UCODE_CTRL.ctrl_cpu) == cpu: load_primary(cpu).
3. else: load_secondary(cpu).
4. touch_nmi_watchdog().
5. true.

`Microcode::nmi_handler() -> bool` (noinstr):
1. if !raw_cpu_read(UCODE_CTRL.nmi_enabled): return false.
2. raw_cpu_write(UCODE_CTRL.nmi_enabled, false).
3. update_handler().

`Microcode::offline_nmi_handler()` (noinstr):
1. if !raw_cpu_read(UCODE_CTRL.nmi_enabled): return.
2. raw_cpu_write(UCODE_CTRL.nmi_enabled, false).
3. raw_cpu_write(UCODE_CTRL.result, UcodeState::Offline).
4. raw_atomic_inc(&OFFLINE_IN_NMI).
5. wait_for_ctrl().

`Microcode::load_primary(cpu)`:
1. nr_offl = CPU_OFFLINE_MASK.weight().
2. proceed = if cpu == 0 && nr_offl > 0 { kick_offline_cpus(nr_offl) } else { true }.
3. if proceed: __load_primary(cpu).
4. if cpu == 0 && nr_offl > 0: release_offline_cpus().

`Microcode::__load_primary(cpu)`:
1. if !wait_for_cpus(&LATE_CPUS_IN):
   - this_cpu_write(UCODE_CTRL.result, UcodeState::Timeout).
   - pr_err_once!("load: {} CPUs timed out", LATE_CPUS_IN.load() - 1).
   - return.
2. ret = (ops.apply_microcode)(cpu).
3. this_cpu_write(UCODE_CTRL.result, ret).
4. this_cpu_write(UCODE_CTRL.ctrl, SiblingCtrl::Done).
5. ctrl = if ret == UcodeState::Updated || ret == UcodeState::Ok { Apply } else { Done }.
6. for sibling in topology_sibling_cpumask(cpu) excluding self:
   - per_cpu(UCODE_CTRL.ctrl, sibling) = ctrl.

`Microcode::load_secondary(cpu)` (noinstr):
1. ctrl_cpu = raw_cpu_read(UCODE_CTRL.ctrl_cpu).
2. if !load_secondary_wait(ctrl_cpu):
   - pr_err_once!("load: {} CPUs timed out", LATE_CPUS_IN.load() - 1).
   - return.
3. /* primary done; instrumentation allowed */
4. ret = if this_cpu_read(UCODE_CTRL.ctrl) == SiblingCtrl::Apply { (ops.apply_microcode)(cpu) }
        else { per_cpu(UCODE_CTRL.result, ctrl_cpu) }.
5. this_cpu_write(UCODE_CTRL.result, ret).
6. this_cpu_write(UCODE_CTRL.ctrl, SiblingCtrl::Done).

`Microcode::wait_for_cpus(cnt) -> bool` (noinstr):
1. raw_atomic_dec_return(cnt).
2. for timeout in 0..USEC_PER_SEC:
   - if raw_atomic_read(cnt) == 0: return true.
   - for _ in 0..LOOPS_PER_USEC.load(): cpu_relax().
   - if !ops.use_nmi && (timeout % USEC_PER_MSEC == 0): touch_nmi_watchdog().
3. raw_atomic_inc(cnt).
4. false.

`Microcode::wait_for_ctrl() -> bool` (noinstr):
1. for timeout in 0..USEC_PER_SEC:
   - if raw_cpu_read(UCODE_CTRL.ctrl) != SiblingCtrl::Wait: return true.
   - for _ in 0..LOOPS_PER_USEC.load(): cpu_relax().
   - if !ops.use_nmi && (timeout % USEC_PER_MSEC == 0): touch_nmi_watchdog().
2. false.

`Microcode::kick_offline_cpus(nr_offl) -> bool`:
1. for cpu in CPU_OFFLINE_MASK:
   - per_cpu(UCODE_CTRL.nmi_enabled, cpu) = true.
   - apic_send_nmi_to_offline_cpu(cpu).
2. for _ in 0..(USEC_PER_SEC/2):
   - if OFFLINE_IN_NMI.load() == nr_offl: return true.
   - udelay(1).
3. false.

`Microcode::bsp_resume()` (syscore):
1. cpu = smp_processor_id(); uci = &UCODE_CPU_INFO[cpu].
2. if uci.mc.is_some(): (ops.apply_microcode)(cpu).
3. else: reload_early_microcode(cpu).

`Microcode::check(prev_info)`:
1. perf_check_microcode(); amd_check_microcode().
2. store_cpu_caps(&mut curr_info).
3. if prev_info.x86_capability == curr_info.x86_capability: return.
4. pr_warn!("x86/CPU: CPU features have changed after loading microcode, but might not take effect.").
5. pr_warn!("x86/CPU: Please consider either early loading through initrd/built-in or a potential BIOS update.").

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dis_ucode_ldr_sticky` | INVARIANT | per-is_disabled: once true, never false again until reboot. |
| `amd_final_levels_disable_loader` | INVARIANT | per-is_disabled: amd rev ∈ FINAL_LEVELS_AMD ⇒ disabled. |
| `apply_runs_on_target_cpu` | INVARIANT | per-apply_microcode: invoked iff smp_processor_id() == cpu argument. |
| `late_cpus_in_balanced` | INVARIANT | per-wait_for_cpus: cnt monotonic toward zero or restored on timeout. |
| `ucode_ctrl_transitions` | INVARIANT | per-MicrocodeCtrl.ctrl: Wait → (Apply|Done); Apply → Done; Done terminal. |
| `primary_threads_required_online` | INVARIANT | per-setup_cpus: any offline primary ⇒ -EBUSY. |
| `nmi_handler_gated_by_per_cpu_enable` | INVARIANT | per-nmi_handler: ucode_ctrl.nmi_enabled false ⇒ no-op. |
| `static_branch_paired` | INVARIANT | per-load_late_stop_cpus: nmi static branch enabled before stop_machine, disabled after. |
| `microcode_check_warns_on_capability_change` | INVARIANT | per-check: bitmap differs ⇒ pr_warn lines. |
| `force_minrev_required_for_downgrade` | INVARIANT | per-request_microcode_fw: below-min_req_ver patch ⇒ rejected unless force_minrev. |
| `secondary_panic_on_primary_timeout` | INVARIANT | per-load_secondary_wait: wait_for_ctrl timeout ⇒ panic. |

### Layer 2: TLA+

`arch/x86/microcode-core.tla`:
- Per-CPU-state {Idle, Waiting, Applying, Done, Timeout, Offline}.
- Per-MicrocodeCtrl rendezvous protocol.
- Properties:
  - `safety_apply_only_on_self` — per-apply: ctrl_cpu == self for primaries; per-CPU for secondaries.
  - `safety_no_apply_while_waiting` — per-secondary: SiblingCtrl::Wait ⇒ no apply.
  - `safety_dis_ucode_ldr_sticky` — per-policy: never re-enabled.
  - `safety_offline_only_when_allowed` — per-setup_cpus: offline-SMT iff allow_smt_offline.
  - `safety_no_double_load` — per-stop_machine: each CPU traverses load_cpus_stopped exactly once.
  - `safety_amd_final_level_disables` — per-amd_check: in FINAL_LEVELS ⇒ loader disabled.
  - `liveness_primary_eventually_done` — per-__load_primary: SCTRL_DONE reached or panic.
  - `liveness_secondary_eventually_done` — per-load_secondary: SCTRL_DONE reached or panic.
  - `safety_microcode_check_warns_on_diff` — per-CPUID-bitmap diff ⇒ pr_warn.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Microcode::is_disabled` post: returns true ⇒ DIS_UCODE_LDR sticky | `Microcode::is_disabled` |
| `Microcode::load_bsp` post: only loads on Intel >= 6 / AMD >= 0x10 | `Microcode::load_bsp` |
| `Microcode::setup_cpus` post: !true ⇒ at least one primary offline OR !allow_smt_offline | `Microcode::setup_cpus` |
| `Microcode::load_late_stop_cpus` post: revision updated XOR (-EIO OR Ok-no-change) | `Microcode::load_late_stop_cpus` |
| `Microcode::__load_primary` post: result ∈ {Updated, Ok, NFound, Error, Timeout}; ctrl == Done | `Microcode::__load_primary` |
| `Microcode::load_secondary` post: result == primary's OR own apply; ctrl == Done | `Microcode::load_secondary` |
| `Microcode::nmi_handler` post: nmi_enabled cleared on entry | `Microcode::nmi_handler` |
| `Microcode::check` post: cap bitmap differs ⇒ both pr_warn emitted | `Microcode::check` |
| `Microcode::bsp_resume` post: uci.mc.is_some ⇒ apply_microcode; else reload_early_microcode | `Microcode::bsp_resume` |

### Layer 4: Verus/Creusot functional

`Per-CPU rendezvous → apply_microcode → ctrl_done → result analysis → microcode_check` semantic equivalence: per-Documentation/x86/microcode.rst (late-loading rendezvous spec) AND per-Intel SDM Vol 3A § "Microcode Update Facility" AND per-AMD APM Vol 2 § "Patch Loader".

## Hardening

(Inherits row-1 features from `arch/x86/00-overview.md` § Hardening.)

microcode-loader reinforcement:

- **Per-AMD final_levels[] hard-disable** — defense against per-write-to-bricked-CPU.
- **Per-DIS_UCODE_LDR sticky** — defense against per-toggle-after-failure.
- **Per-min_req_ver minimum-revision gate** — defense against per-downgrade attack.
- **Per-force_minrev requires CONFIG_MICROCODE_LATE_FORCE_MINREV + cmdline** — defense against per-silent debug-override.
- **Per-apply runs on target CPU only** — defense against per-cross-CPU MSR write.
- **Per-primary/secondary SMT rendezvous via stop_machine** — defense against per-mid-update sibling interference.
- **Per-NMI rendezvous gated by per-CPU ucode_ctrl.nmi_enabled** — defense against per-spurious-NMI participation.
- **Per-noinstr on NMI path** — defense against per-IRET-from-#INT3/#DB/#PF re-enabling NMI.
- **Per-static_branch enable/disable paired** — defense against per-NMI handler leak after load.
- **Per-primary timeout ⇒ panic** — defense against per-CPU-stuck-with-locks corruption.
- **Per-microcode_check warns on CPUID change** — defense against per-silent-feature-mismatch.
- **Per-reload sysfs WO 0200 root-only** — defense against per-unprivileged trigger.
- **Per-CPU_HOTPLUG cpus_read_lock around load_late_locked** — defense against per-online-during-update.
- **Per-hypervisor_present disables loader by default** — defense against per-hypervisor-microcode collision.
- **Per-soft-offline siblings allowed only with ops->nmi_safe OR (use_nmi && nmi_to_offline_cpu)** — defense against per-NMI-in-play_dead corruption.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- arch/x86/kernel/cpu/microcode/intel.c Intel-specific apply path (covered separately if expanded)
- arch/x86/kernel/cpu/microcode/amd.c AMD-specific apply path (covered separately if expanded)
- arch/x86/kernel/cpu/microcode/intel-ucode-defs.h header layout (header-only)
- kernel/stop_machine.c stop_machine engine (covered separately if expanded)
- drivers/base/firmware_loader/* request_firmware (covered separately if expanded)
- arch/x86/kernel/cpu/bugs.c speculative-mitigation feature gating (covered in `cpu-mitigations.md` Tier-3)
- Implementation code
