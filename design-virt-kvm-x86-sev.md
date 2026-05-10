---
title: "Tier-3: arch/x86/kvm/svm/sev.c — KVM SEV (Secure Encrypted Virtualization) + SEV-ES + SEV-SNP confidential VMs"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

AMD SEV (Secure Encrypted Virtualization) encrypts guest memory with a per-VM key managed by the AMD Platform Secure Processor (PSP); host cannot read guest memory. SEV-ES (Encrypted State) additionally encrypts vCPU register state at vmexit (using a Guest Hypervisor Communication Block / GHCB for vmexit info). SEV-SNP (Secure Nested Paging) adds page-integrity tracking via Reverse Map Table (RMP) preventing host swap-attacks + page-replay attacks. KVM's sev.c drives PSP firmware commands (LAUNCH_START / LAUNCH_UPDATE_DATA / LAUNCH_FINISH / GUEST_STATUS / DEACTIVATE / DECOMMISSION) + sets per-VM ASID + handles SEV-specific vmexits. Used in confidential-computing cloud (AWS Nitro, Azure CVM, GCP Confidential VMs).

This Tier-3 covers `arch/x86/kvm/svm/sev.c` (~5252 lines).

### Acceptance Criteria

- [ ] AC-1: Module init: kvm-amd loaded on Zen2+ host with SEV firmware loaded; sev_hardware_setup succeeds.
- [ ] AC-2: KVM_SEV_INIT: VM allocates SEV ASID; sev_active returns true.
- [ ] AC-3: SEV launch flow: LAUNCH_START + LAUNCH_UPDATE_DATA (encrypt 100MB) + LAUNCH_MEASURE + LAUNCH_FINISH; VM boots Linux guest.
- [ ] AC-4: Host cannot read guest memory: probe via /proc/<qemu-pid>/mem returns encrypted bytes (random-appearing).
- [ ] AC-5: SEV-ES boot: SEV-ES VMs require encrypted VMSA + GHCB; OVMF SEV-ES BIOS boots.
- [ ] AC-6: SEV-SNP boot: PVALIDATE + RMP setup; SNP-attested VM passes attestation.
- [ ] AC-7: KVM_SEV_LAUNCH_MEASURE: returned measurement matches expected hash for fixed firmware.
- [ ] AC-8: SEV migration: VM migrated end-to-end; receiving host can resume.
- [ ] AC-9: PSP error handling: malformed cmd returns error to userspace; no kernel crash.
- [ ] AC-10: kvm-unit-tests `sev` test passes (where supported).

### Architecture

`KvmSevInfo` per-VM:

```
struct KvmSevInfo {
  active: bool,
  es_active: bool,
  snp_active: bool,
  asid: u32,
  handle: u32,                                  // PSP VM handle
  fd: i32,                                       // user fd for SEV resources
  enc_context_owner: KArc<EncContext>,
  enc_pin_lock: Mutex<()>,
  pinned_pages: ListHead,
  policy: u32,
  ...
}

struct EncRegion {
  list: ListNode,
  npages: u64,
  pages: KVec<KArc<Page>>,
  uaddr: u64,
  size: u64,
}
```

Per-vCPU additions for SEV-ES:

```
struct VcpuSvm {
  ...
  vmsa: KBox<Page>,                              // 4KiB encrypted register state
  vmsa_pa: u64,
  ghcb: Option<KArc<Ghcb>>,                      // shared communication
  ghcb_pa: u64,
  ghcb_in_use: bool,
}
```

`KvmSevInfo::launch_start(kvm, argp)`:
1. Read user-supplied policy + dh_pubkey + session_data.
2. Allocate PSP cmd descriptor LAUNCH_START.
3. Populate cmd: handle = sev_info.handle; policy; dh_pubkey; session_data.
4. PSP cmd: send via __sev_do_cmd(SEV_CMD_LAUNCH_START, cmd, &error).
5. If error: cleanup; return.
6. Update sev_info.handle from response.

`SevEs::handle_vmgexit(vcpu)`:
1. Map ghcb shared bytes (may be in user mm).
2. Read ghcb.sw_exit_code.
3. Validate in allow-list (e.g., SVM_EXIT_IO, SVM_EXIT_MSR, SVM_EXIT_NPF).
4. Switch:
   - SVM_EXIT_IO: emulate IO with values from ghcb.
   - SVM_EXIT_MSR: emulate MSR read/write.
   - SVM_EXIT_NPF: page-fault → TDP-MMU.
5. Write result fields to ghcb (e.g., MSR result in ghcb.rax + ghcb.rdx).
6. Mark ghcb.sw_exit_code = 0 (success).
7. Resume guest.

`KvmSevInfo::launch_update_data(kvm, argp)`:
1. Read user uaddr + size.
2. enc_region := alloc EncRegion; pin pages via get_user_pages.
3. PSP cmd: LAUNCH_UPDATE_DATA with handle + dest-pa-list + size.
4. PSP encrypts in-place + updates measurement.
5. Unpin pages.
6. Add region to sev_info.pinned_pages list.

### Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- SVM core (covered in `x86-svm.md` Tier-3)
- TDX (Intel-side confidential VM; covered in `x86-tdx.md` future Tier-3)
- PSP driver (drivers/crypto/ccp; covered in `drivers/crypto/ccp.md` future Tier-3)
- SEV firmware loading (firmware blob; out-of-scope)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_sev_info` | per-VM SEV state | `kernel::kvm::x86::svm::sev::KvmSevInfo` |
| `enum sev_command` | PSP cmd codes (LAUNCH_START etc.) | `SevCommand` |
| `sev_hardware_setup()` | per-host SEV setup at module load | `Sev::hardware_setup` |
| `sev_hardware_teardown()` | per-host teardown | `Sev::hardware_teardown` |
| `sev_vm_init(kvm)` / `sev_vm_destroy(kvm)` | per-VM lifecycle | `KvmSevInfo::vm_init` / `_destroy` |
| `sev_guest_init(kvm, argp)` (KVM_SEV_INIT) | per-VM PSP-side init | `KvmSevInfo::guest_init` |
| `sev_launch_start(kvm, argp)` (KVM_SEV_LAUNCH_START) | PSP LAUNCH_START | `KvmSevInfo::launch_start` |
| `sev_launch_update_data(kvm, argp)` (KVM_SEV_LAUNCH_UPDATE_DATA) | encrypt + measure guest memory | `KvmSevInfo::launch_update_data` |
| `sev_launch_measure(kvm, argp)` (KVM_SEV_LAUNCH_MEASURE) | extract launch measurement | `KvmSevInfo::launch_measure` |
| `sev_launch_finish(kvm, argp)` (KVM_SEV_LAUNCH_FINISH) | finalize + activate | `KvmSevInfo::launch_finish` |
| `sev_es_init_vmcb(svm)` (SEV-ES) | per-vCPU VMCB SEV-ES setup | `SevEs::init_vmcb` |
| `sev_handle_vmgexit(vcpu)` | SEV-ES vmgexit handler | `SevEs::handle_vmgexit` |
| `setup_vmsa_bp_table(...)` (SEV-ES) | encrypted VMSA setup | `SevEs::setup_vmsa` |
| `sev_snp_init(kvm)` (SEV-SNP) | SNP-specific init | `SevSnp::init` |
| `snp_launch_start(...)` (SEV-SNP) | SNP launch flow | `SevSnp::launch_start` |
| `snp_handle_psp_msg_chunk(...)` (SEV-SNP) | per-PSP-message handler | `SevSnp::handle_psp_msg` |
| `sev_pin_memory(kvm, gpa, size)` | pin user pages for encryption | `KvmSevInfo::pin_memory` |
| `sev_unpin_memory(kvm, region)` | unpin | `KvmSevInfo::unpin_memory` |
| `sev_is_active(kvm)` | per-VM SEV active query | `KvmSevInfo::is_active` |
| `sev_es_active(kvm)` | SEV-ES active query | `KvmSevInfo::es_active` |
| `sev_set_msr_protection(...)` | per-MSR access control for SEV-ES guest | `SevEs::set_msr_protection` |

### compatibility contract

REQ-1: Per-VM `kvm_sev_info`:
- `active` (bool: SEV active for this VM).
- `es_active` (bool: SEV-ES).
- `snp_active` (bool: SEV-SNP).
- `asid` (per-VM SEV ASID range; AMD restricts SEV ASIDs to a separate pool).
- `handle` (PSP-assigned VM handle; opaque to KVM).
- `enc_context_owner` (KVM ref to context for migration).
- `enc_pin_lock` (mutex protecting pinned-pages list).
- `pinned_pages` (list of pinned guest-physical regions for encryption operations).
- `psp_data` (per-VM PSP private state).

REQ-2: Per-vCPU SEV-ES VMSA:
- VMSA = VM Save Area, per-vCPU encrypted register state.
- Per-vCPU page allocated at vCPU-create.
- On VMRUN: CPU encrypts/decrypts VMSA atomically.
- On vmexit: VMSA opaque to host; only GHCB exposed.

REQ-3: GHCB (Guest-Hypervisor Communication Block) for SEV-ES:
- 4KiB page; partially shared (decrypted) with host.
- Guest writes vmexit-info (exit_code, exit_info1, exit_info2) before VMGEXIT.
- KVM reads GHCB to determine exit semantics.
- Limited set of exit_codes allowed via GHCB (validated against allow-list).

REQ-4: PSP firmware command dispatch:
- KVM driver writes per-cmd descriptor to PSP MMIO region.
- PSP processes async; KVM polls completion bit.
- Per-cmd error codes returned.
- Commands: INIT, SHUTDOWN, PLATFORM_RESET, PLATFORM_STATUS, GUEST_STATUS, LAUNCH_START / _UPDATE_VMSA / _UPDATE_DATA / _MEASURE / _SECRET / _FINISH, DEACTIVATE, DECOMMISSION, ACTIVATE, GUEST_DECRYPT, etc.

REQ-5: KVM_SEV_INIT ioctl flow:
1. Allocate kvm_sev_info struct.
2. Acquire SEV ASID from sev_asid_bitmap.
3. PSP cmd: GUEST_STATUS (verify slot available).
4. PSP cmd: ACTIVATE (associate ASID with handle).
5. kvm.arch.sev_info = struct.

REQ-6: KVM_SEV_LAUNCH_START flow:
1. Read user-supplied policy + dh_pubkey + session_data.
2. PSP cmd: LAUNCH_START with policy + handle.
3. Receive measurement-context.

REQ-7: KVM_SEV_LAUNCH_UPDATE_DATA flow:
1. Pin guest memory pages.
2. PSP cmd: LAUNCH_UPDATE_DATA with addr + size.
3. PSP encrypts pages in-place + updates measurement.
4. Unpin pages.

REQ-8: KVM_SEV_LAUNCH_FINISH:
1. PSP cmd: LAUNCH_FINISH.
2. VM transitions to running state; encrypted memory permanently sealed under guest key.

REQ-9: SEV-ES VMGEXIT handler:
1. Read GHCB shared bytes; extract guest's reported exit_code.
2. Validate exit_code in allow-list (specific MMIO/IO/MSR ops permitted).
3. Per-exit_code: emulate (e.g., MSR read/write with guest values supplied via GHCB).
4. Write result back to GHCB.
5. Resume guest via VMRUN.

REQ-10: SEV-SNP RMP table:
- AMD-managed Reverse Map Table; one entry per host 4KiB page.
- Per-RMP: assigned (1 = guest-owned), validated (1 = guest-attested), guest-handle, guest-PA.
- Guest accesses unassigned page → RMP-fault → KVM handles (alloc + assign).
- Defense against host swapping out guest page + replacing.

REQ-11: SEV-SNP PVALIDATE / RMPADJUST:
- Guest uses PVALIDATE instruction to declare per-page validation status.
- Guest uses RMPADJUST to change page permissions per-VMSA-restricted-injection.
- KVM intercepts when RMPADJUST attempts privileged action.

REQ-12: ASID management:
- AMD reserves ASID range 1..max_sev_asid for SEV.
- ASIDs 1..max_es_asid additionally reserved for SEV-ES.
- KVM maintains per-host bitmap; allocates per-VM at INIT.

REQ-13: Per-VM enc_context_owner for migration:
- Source VM holds owner ref.
- Migration: PSP cmd GUEST_DECRYPT extracts encrypted memory; transfers via key-exchange.
- Destination VM imports via PSP cmd RECEIVE_START / RECEIVE_UPDATE_DATA / RECEIVE_FINISH.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `asid_unique_per_vm` | INVARIANT | per-VM SEV ASID distinct across all SEV VMs at any time. |
| `psp_cmd_descriptor_aligned` | INVARIANT | per-PSP-cmd descriptor aligned to PSP requirement. |
| `pinned_pages_balanced` | INVARIANT | per-region pin/unpin paired; defense against UAF on user pages. |
| `ghcb_validation` | INVARIANT | GHCB-reported exit_code validated against allow-list; defense against guest spoofing arbitrary exit. |
| `vmsa_aligned` | INVARIANT | per-vCPU VMSA 4KiB aligned. |

### Layer 2: TLA+

`virt/kvm/sev_launch_lifecycle.tla`:
- States: Inactive, Init, LaunchStarted, UpdatingData, Measured, Active, Decommissioned.
- Properties:
  - `safety_active_only_after_finish` — Active state only after LAUNCH_FINISH.
  - `safety_no_data_update_after_finish` — UPDATE_DATA rejected post-FINISH.
  - `liveness_init_eventually_active` — assuming user-supplied valid args, INIT path reaches Active.

`virt/kvm/sev_es_vmgexit.tla`:
- Per-vmgexit dispatch state.
- Properties:
  - `safety_only_allowed_exits` — vmgexit only proceeds for allow-listed exit_codes.
  - `safety_no_register_leak` — non-GHCB-fields not exposed to host.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `KvmSevInfo::launch_start` post: handle obtained from PSP; sev_info populated | `KvmSevInfo::launch_start` |
| `KvmSevInfo::launch_update_data` post: pages pinned; PSP cmd executed; pages unpinned | `KvmSevInfo::launch_update_data` |
| `SevEs::handle_vmgexit` post: ghcb.sw_exit_code result code set; vCPU resumable | `SevEs::handle_vmgexit` |
| Per-VM enc_pin_lock held during pinned_pages mutation | invariants on launch_update |
| Per-VM ASID released on DECOMMISSION | `KvmSevInfo::vm_destroy` |

### Layer 4: Verus/Creusot functional

`KVM_SEV_INIT + LAUNCH_START + LAUNCH_UPDATE_DATA + LAUNCH_MEASURE + LAUNCH_FINISH = encrypted VM with attestable launch measurement` semantic equivalence: per-VM the launch-measurement output represents cryptographic hash of the encrypted-loaded-firmware code matching expected.

### hardening

(Inherits row-1 features from `virt/kvm/x86-svm.md` § Hardening.)

SEV-specific reinforcement:

- **Per-VM SEV ASID distinct** — defense against TLB-leak across confidential VMs.
- **Per-VM PSP handle validated against KVM_SEV_INIT response** — defense against handle-spoofing.
- **Pinned pages refcount tracked** — defense against guest-memory unmapping mid-encryption causing UAF.
- **PSP cmd timeout-bounded** — defense against PSP-hang causing kernel hang.
- **GHCB exit_code allow-list** — defense against guest reaching unsupported handler paths.
- **GHCB only-shared-fields exposed** — defense against private-data leak via GHCB.
- **Per-VM enc_pin_lock** for pinned_pages list — defense against concurrent launch-update-data races.
- **VMSA per-vCPU isolated** — defense against cross-vCPU register leak.
- **SNP RMP enforces page-assignment** — defense against host swap-attack on guest pages.
- **PVALIDATE intercept on privileged ops** — defense against guest mis-using to bypass SNP integrity.
- **DECOMMISSION on VM destroy** — defense against ASID leak.
- **PSP firmware version check** — defense against running KVM with incompatible PSP version.
- **Per-VM enc_context_owner ref-count** — defense against migration source releasing while target still importing.
- **PSP cmd return-status validated** — defense against silent PSP failure leaving inconsistent state.

