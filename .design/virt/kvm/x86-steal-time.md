# Tier-3: arch/x86/kvm/x86.c — KVM_FEATURE_STEAL_TIME paravirt (per-vCPU steal-time + preempt-flag + nm-PV-spin)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/x86.c (steal-time related sections; see grep "record_steal_time|MSR_KVM_STEAL_TIME|kvm_steal_time")
  - arch/x86/include/uapi/asm/kvm_para.h (struct kvm_steal_time + KVM_FEATURE_STEAL_TIME + KVM_VCPU_PREEMPTED)
  - include/uapi/linux/kvm_para.h
-->

## Summary

KVM steal-time is a paravirt mechanism that exposes "host-stolen-time" (time the vCPU was runnable but not actually executing because host was running other tasks) to the guest scheduler so it can: (1) account stolen-time as non-user/non-system in /proc/stat, (2) skip stale lock-spin if vCPU holding lock has been preempted (preempted bit), (3) use guest-side scheduling decisions that account for host competition. KVM publishes per-vCPU `struct kvm_steal_time` to a guest-supplied 4KiB page; updates on every vmenter/vmexit boundary with elapsed runnable-but-not-running ns + preempted flag.

This Tier-3 covers steal-time related code in `arch/x86/kvm/x86.c` (~400-500 lines spread across record_steal_time + MSR handler + kvm_steal_time_set_preempted).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_steal_time` | per-vCPU UAPI struct | `kernel::kvm::x86::SteaTime::KvmSteaTimeStruct` |
| `record_steal_time(vcpu)` | per-vmenter update steal-time | `Vcpu::record_steal_time` |
| `kvm_steal_time_set_preempted(vcpu)` | per-vmexit set preempted bit | `Vcpu::steal_time_set_preempted` |
| `MSR_KVM_STEAL_TIME` (0x4b564d03) | per-vCPU enable + control-page-addr | UAPI |
| `KVM_VCPU_PREEMPTED` (1 << 0) | preempted-flag bit | UAPI |
| `KVM_VCPU_FLUSH_TLB` (1 << 1) | guest-requested-flush-TLB bit | UAPI |
| `kvm_pv_clear_preempted(vcpu)` | per-vmenter clear preempted | `Vcpu::pv_clear_preempted` |
| `kvm_arch_vcpu_load(vcpu, cpu)` (vendor hook) | vCPU-load post-clear-preempted | covered in vmx/svm Tier-3s |
| `kvm_arch_vcpu_put(vcpu)` | vCPU-put pre-set-preempted | covered in vmx/svm Tier-3s |
| `kvm_steal_time_init(vcpu)` | per-vCPU init | `Vcpu::steal_time_init` |
| `kvm_steal_time_msr_handler(...)` | WRMSR handler | covered in `Vcpu::handle_msr` |

## Compatibility contract

REQ-1: Per-vCPU `struct kvm_steal_time` (UAPI; 64 bytes):
- `steal` (u64; ns of stolen-time accumulated).
- `version` (u32; even = stable, odd = update-in-progress).
- `flags` (u32; reserved).
- `preempted` (u8; KVM_VCPU_PREEMPTED bit when vCPU not running).
- `u8 pad[47]` (reserved).

REQ-2: Guest enable via WRMSR(MSR_KVM_STEAL_TIME):
- data bit 0: enable.
- data bits[63:6] (4KiB-aligned): control-page guest-PA.
- Disable: data == 0 → clear enable; control-page no longer updated.

REQ-3: Per-vmexit `kvm_steal_time_set_preempted`:
1. If !vcpu.arch.steal_time_enabled: return.
2. Map control-page via memslot.
3. seq := atomic_inc(&vcpu.arch.steal_time_version) | 1; (odd).
4. Write version field.
5. preempted_byte := KVM_VCPU_PREEMPTED.
6. Write `preempted` byte.
7. version even.
8. Write version final.

REQ-4: Per-vmenter `record_steal_time`:
1. now := ktime_get_ns().
2. delta := now - vcpu.arch.steal_time_last_update.
3. If vcpu was preempted (last vmexit set preempted): vcpu.arch.steal_time += delta.
4. Update `steal` field in control-page (with version-counter dance).
5. Clear preempted bit (vCPU now running).

REQ-5: Atomic update via version-counter:
- Same scheme as kvmclock: even = stable, odd = update-in-progress.
- Guest reads atomically: read version; if odd: retry. Read fields. Re-read version; if mismatch: retry.

REQ-6: Per-vCPU last-update tracking:
- vcpu.arch.steal_time_last_update (ktime_t at vmenter).
- vcpu.arch.this_time_out (vmexit-time on vmexit, used to compute next vmenter delta).

REQ-7: KVM_VCPU_FLUSH_TLB hint:
- Guest may set flush-TLB bit; KVM observes + flushes per-vCPU TLB on next vmenter.
- Used by guest pv-tlb-flush optimization.

REQ-8: KVM_FEATURE_STEAL_TIME CPUID bit:
- Advertised in CPUID 0x40000001 EAX bit 5.
- Gates guest's enable WRMSR.

REQ-9: KVM_FEATURE_PV_UNHALT (related):
- Bit 6 of CPUID 0x40000001 EAX.
- Lets guest use PV-spin (kvm_kick_cpu/kvm_wait) which integrates with steal-time preempted-bit.

REQ-10: Per-VM enable gate:
- KVM module param `enable_pv_steal_time` (default true).
- If false: KVM_FEATURE_STEAL_TIME not advertised.

REQ-11: Live migration:
- Per-vCPU MSR_KVM_STEAL_TIME migrated.
- steal accumulator: serialized via `kvm_get_msr` + `kvm_set_msr`.
- Post-migrate: control-page re-mapped on first vmenter.

REQ-12: Guest-PV-spin integration:
- Linux qspinlock guest path checks per-vCPU.preempted bit; if set: skip CAS-spin (lock-holder isn't running anyway) → call PV-yield.

## Acceptance Criteria

- [ ] AC-1: KVM_FEATURE_STEAL_TIME advertised: guest CPUID 0x40000001 EAX bit 5 set.
- [ ] AC-2: Boot Linux guest: dmesg shows "Using KVM steal-time".
- [ ] AC-3: Guest /proc/stat shows steal column nonzero under host CPU contention.
- [ ] AC-4: Preempted-bit visible: guest reading MSR_KVM_STEAL_TIME control-page sees preempted=1 when vCPU not running.
- [ ] AC-5: PV qspinlock optimization: 16-vCPU guest with high lock contention; per-vCPU not-spinning when holder preempted; reduced CPU usage.
- [ ] AC-6: Live migration: steal-time accumulator continues post-migrate (no negative jumps).
- [ ] AC-7: Disable test: guest WRMSR(MSR_KVM_STEAL_TIME, 0); subsequent /proc/stat steal column stops advancing.
- [ ] AC-8: kvm-unit-tests `pv_steal_time` test passes.
- [ ] AC-9: Spec-violation defense: WRMSR(MSR_KVM_STEAL_TIME) with non-page-aligned addr → no enable; KVM logs.

## Architecture

`KvmSteaTimeStruct` per-vCPU UAPI:

```
#[repr(C)]
struct KvmSteaTimeStruct {
  steal: u64,
  version: u32,
  flags: u32,
  preempted: u8,
  pad: [u8; 47],
}
```

Per-vCPU additions to `VcpuArch`:

```
struct VcpuArch {
  ...
  st: VcpuStealTime,
}

struct VcpuStealTime {
  enabled: bool,
  msr_val: u64,                                // MSR_KVM_STEAL_TIME
  cache: GfnToHvaCache,                        // pre-validated HVA mapping
  last_steal: u64,                             // ns at last record_steal_time
  preempted: u8,                               // bitmap (PREEMPTED + FLUSH_TLB)
  version: AtomicU32,
}
```

`Vcpu::record_steal_time(vcpu)` flow:
1. If !vcpu.arch.st.enabled: return.
2. ghc := &vcpu.arch.st.cache.
3. If ghc not valid: kvm_gfn_to_hva_cache_init.
4. now := vcpu.arch.last_running_time.
5. delta := now - vcpu.arch.st.last_steal.
6. vcpu.arch.st.last_steal = now.
7. seq := atomic_inc(&vcpu.arch.st.version) | 1.
8. Write version to control-page at offset version-field.
9. Write `steal += delta` to control-page.
10. Write flags + clear preempted (vCPU now running).
11. seq++ (even).
12. Write version final.

`Vcpu::steal_time_set_preempted(vcpu)`:
1. If !vcpu.arch.st.enabled: return.
2. ghc := &vcpu.arch.st.cache.
3. seq := atomic_inc(&vcpu.arch.st.version) | 1.
4. Write version to control-page.
5. preempted := KVM_VCPU_PREEMPTED.
6. Write preempted-byte.
7. seq++.
8. Write version final.

`Vcpu::handle_msr` MSR_KVM_STEAL_TIME case:
1. addr := data & ~1; enable := data & 1.
2. If !enable: vcpu.arch.st.enabled = false; gfn_to_hva_cache_uninit.
3. If enable:
   - Validate addr 4KiB-aligned + within memslot.
   - kvm_gfn_to_hva_cache_init(vcpu.kvm, &vcpu.arch.st.cache, addr, sizeof(KvmSteaTimeStruct)).
   - vcpu.arch.st.enabled = true.
   - vcpu.arch.st.last_steal = ktime_get_ns().

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `control_page_aligned` | INVARIANT | per-vCPU control-page addr 4KiB-aligned; defense against straddling page boundary write. |
| `version_seq_update_atomic` | INVARIANT | version odd-during-update + even-when-stable per-Intel-KVM-spec. |
| `steal_no_overflow` | INVARIANT | per-vCPU steal accumulator u64; defense against monotonic-only-add wrap (~10^11 years). |
| `gfn_to_hva_cache_revalidated` | INVARIANT | cache invalidated on memslot delete; defense against stale-HVA UAF. |
| `preempted_set_clear_paired` | INVARIANT | set_preempted on vmexit ↔ clear in record_steal_time on vmenter; defense against stuck-preempted. |

### Layer 2: TLA+

`virt/kvm/steal_time_atomic_update.tla`:
- States per per-vCPU steal-time: Stable, UpdateInProgress.
- Properties:
  - `safety_seq_odd_iff_in_progress` — version-bit-0 == 1 iff state == UpdateInProgress.
  - `safety_guest_atomic_read` — guest seq-loop terminates with consistent fields if updates eventually quiesce.
  - `liveness_eventual_quiescence` — assuming bounded host-updates, guest read eventually succeeds.

`virt/kvm/preempted_state.tla`:
- States: NotPreempted, Preempted.
- Transitions: NotPreempted → Preempted on vmexit; Preempted → NotPreempted on vmenter.
- Properties:
  - `safety_preempted_implies_vmexit` — only set by vCPU-put path.
  - `safety_clear_implies_vmenter` — only cleared by vCPU-load path.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::handle_msr(MSR_KVM_STEAL_TIME)` post: addr cached + validated; enabled flag reflects bit-0 | `Vcpu::handle_msr` |
| `Vcpu::record_steal_time` post: control-page steal field == cumulative ns; version even after | `Vcpu::record_steal_time` |
| `Vcpu::steal_time_set_preempted` post: control-page preempted byte == KVM_VCPU_PREEMPTED; version even after | `Vcpu::steal_time_set_preempted` |
| Per-vCPU last_steal monotonically increases | `Vcpu::record_steal_time` |
| `gfn_to_hva_cache_init` post: cache.gpa == addr; cache.hva valid | `kvm_gfn_to_hva_cache_init` |

### Layer 4: Verus/Creusot functional

`Per-vmexit + per-vmenter pair: steal accumulator advances by elapsed-not-running-time; preempted-bit transitions match vCPU-running-state` round-trip equivalence: guest /proc/stat steal-column matches actual stolen-CPU-time for this vCPU.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

steal-time-specific reinforcement:

- **Version sequence-counter atomic update** — defense against guest reading torn fields.
- **Per-vCPU control-page PA validated** at WRMSR — defense against guest pointing to host kernel memory.
- **gfn_to_hva_cache invalidation on memslot delete** — defense against UAF when memslot remapped.
- **PREEMPTED bit cleared by vmenter only** — defense against guest-controlled preempted-bit causing wrong PV-spin decisions.
- **steal accumulator in u64** — defense against pathological pause causing wraparound.
- **Per-vCPU last_steal initialized at first enable** — defense against negative-delta on first record.
- **enable WRMSR validates 4KiB alignment** — defense against straddling-page writes corrupting adjacent guest memory.
- **Live-migrate clears cache + re-validates on first vmenter** — defense against post-migrate stale HVA pointer.
- **MSR_KVM_STEAL_TIME enable-bit** — defense against half-configured pv-steal-time causing incoherent state.
- **enable_pv_steal_time module param** — defense against pre-VM-create disable for security-sensitive VMs.
- **KVM_VCPU_FLUSH_TLB bit honored only if pv-tlb-flush feature enabled** — defense against guest setting unsupported flag bits.
- **steal-time control-page in IOMMU-protected mem** — defense against device-DMA writing steal-time field corrupting guest accounting.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — steal-time control-page write goes through `kvm_write_guest_cached` with bounded `struct kvm_steal_time` size; no direct user-pointer copy.
- **PAX_KERNEXEC** — `kvm_steal_time_set_preempted`, `record_steal_time`, `kvm_set_msr_common` (MSR_KVM_STEAL_TIME branch) resolve through RX-only kernel text.
- **PAX_RANDKSTACK** — preempt-notifier and KVM_RUN paths inherit RANDKSTACK from scheduler / syscall entry.
- **PAX_REFCOUNT** — per-vCPU steal-time gfn-cache pinned through SRCU + vCPU refcount saturating.
- **PAX_MEMORY_SANITIZE** — `vcpu.arch.st` (last_steal, gfn-cache, flags) zeroed on alloc/free; on WRMSR-disable the cached HVA is cleared so a successor enable cannot inherit it.
- **PAX_UDEREF** — steal-time GPA validated against memslots; no raw user pointer dereference.
- **PAX_RAP / kCFI** — `kvm_x86_ops.set_msr` slot RAP-signed.
- **GRKERNSEC_HIDESYM** — steal-time gfn-cache kaddr and per-vCPU last_steal redacted unless CAP_SYSLOG + gr-rbac.
- **GRKERNSEC_DMESG** — invalid-GPA / unaligned-enable WRMSR warnings rate-limited and gated by `dmesg_restrict`.
- **KVM ioctl CAP_SYS_ADMIN strict** — KVM_CAP_PV_STEAL_TIME advertisement gated to CAP_SYS_ADMIN + gr-rbac VM-create role.
- **MSR_KVM_STEAL_TIME passthrough validation** — enable-bit (bit 0), 4KiB-alignment, and GPA-in-memslot validated; reserved-bit writes intercepted and #GP-injected.
- **KVM_VCPU_FLUSH_TLB gating** — preempt_flush bit honored only if pv-tlb-flush CPUID feature enabled; guest setting unsupported flags rejected.
- **IOMMU-protected mem** — steal-time control-page subject to IOMMU domain protection so DMA from a passthrough device cannot corrupt steal-time fields.
- **Nested-virt strict** — L1 steal-time advertised to L2 mediated through L1; L0 steal-time cache never directly written by L2 guest.

Per-doc rationale: steal-time exposes a guest-mapped page that the host updates from preempt-notifier context; the grsec reinforcement here validates the WRMSR enable to defend against unaligned/out-of-memslot GPAs, SANITIZEs the gfn-cache on disable to prevent successor-VM cross-contamination, and applies IOMMU protection so DMA cannot tamper with the same field a preempt notifier is writing.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- VMX vmenter/vmexit details (covered in `x86-vmx.md` Tier-3)
- SVM vmenter/vmexit details (covered in `x86-svm.md` Tier-3)
- kvmclock paravirt (covered in `x86-pvclock.md` Tier-3)
- Async-PF paravirt (covered in `x86-async-pf.md` Tier-3)
- PV-tlb-flush + PV-IPI (covered in `x86-pv-ipi.md` future Tier-3)
- PV-unhalt + PV-spin (covered in `x86-pv-spin.md` future Tier-3)
- Implementation code
