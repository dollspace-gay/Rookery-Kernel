# Tier-3: drivers/crypto/ccp/ — AMD CCP / PSP (Cryptographic Coprocessor + SEV-firmware-host-interface + TEE + RNG)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/crypto/00-overview.md
upstream-paths:
  - drivers/crypto/ccp/ccp-dev.c
  - drivers/crypto/ccp/ccp-dev.h
  - drivers/crypto/ccp/ccp-dev-v3.c
  - drivers/crypto/ccp/ccp-dev-v5.c
  - drivers/crypto/ccp/ccp-ops.c
  - drivers/crypto/ccp/ccp-crypto-main.c
  - drivers/crypto/ccp/ccp-crypto-aes.c
  - drivers/crypto/ccp/ccp-crypto-aes-xts.c
  - drivers/crypto/ccp/ccp-crypto-aes-galois.c
  - drivers/crypto/ccp/ccp-crypto-aes-cmac.c
  - drivers/crypto/ccp/ccp-crypto-des3.c
  - drivers/crypto/ccp/ccp-crypto-rsa.c
  - drivers/crypto/ccp/ccp-crypto-sha.c
  - drivers/crypto/ccp/psp-dev.c
  - drivers/crypto/ccp/sev-dev.c
  - drivers/crypto/ccp/tee-dev.c
  - drivers/crypto/ccp/sfs.c
  - drivers/crypto/ccp/dbc.c
  - drivers/crypto/ccp/hsti.c
  - drivers/crypto/ccp/sp-dev.c
  - drivers/crypto/ccp/sp-pci.c
  - drivers/crypto/ccp/sp-platform.c
  - drivers/crypto/ccp/ccp-dmaengine.c
  - drivers/crypto/ccp/ccp-debugfs.c
  - include/linux/ccp.h
  - include/linux/psp.h
  - include/linux/psp-sev.h
  - include/linux/psp-platform-access.h
  - include/uapi/linux/psp-sev.h
-->

## Summary

`drivers/crypto/ccp/` is the AMD Secure Processor (ASP, formerly "PSP") driver tree. It binds three distinct logical functions that share one PCI device per AMD package (CPU socket or APU):

1. **CCP** — Cryptographic Coprocessor: symmetric/asymmetric crypto offload (AES, SHA, RSA, DES3) + TRNG, exposed via the kernel crypto API.
2. **PSP** — Platform Security Processor mailbox + sub-mailboxes for **SEV** (Secure Encrypted Virtualization, including SEV-ES + SEV-SNP), **TEE** (trusted application interface used by Microsoft fTPM and AMD-SP TAs), **DBC** (Dynamic Boost Control firmware-signed throttling), **SFS** (Seamless Firmware Servicing), **HSTI** (Hardware Security Test Interface).
3. **DMA-engine** — CCP doubles as a `dmaengine` provider (memcpy offload) on platforms where the CCP exposes data-movement queues.

The driver is the host-side counterpart of AMD's PSP firmware. Every SEV/SEV-ES/SEV-SNP guest-launch, attestation report, migration handshake, and platform-init sequence funnels through this driver via the `psp-sev.h` UAPI (`/dev/sev` character device) — making `drivers/crypto/ccp/` simultaneously a crypto offload driver and the kernel's primary CoCo (Confidential Computing) attestation entry point.

This Tier-3 covers all five sub-modules with focus on the PSP mailbox protocol, SEV-firmware command path, key-handle lifetime, and TRNG plumbing.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sp_device` | per-AMD-package Secure-Processor PCI device (root container) | `drivers::ccp::SpDevice` |
| `struct ccp_device` | CCP sub-device (crypto + TRNG + dmaengine) | `drivers::ccp::CcpDevice` |
| `struct psp_device` | PSP sub-device (mailbox + SEV + TEE + SFS + DBC + HSTI) | `drivers::ccp::PspDevice` |
| `struct sev_device` | SEV-firmware host interface state (under PSP) | `drivers::ccp::SevDevice` |
| `struct tee_device` | TEE TA mailbox interface (under PSP) | `drivers::ccp::TeeDevice` |
| `struct ccp_cmd` | per-request CCP command descriptor | `drivers::ccp::CcpCmd` |
| `struct ccp_cmd_queue` | per-engine HW ring | `drivers::ccp::CcpQueue` |
| `ccp_register_rng(ccp)` / `ccp_unregister_rng(ccp)` | hwrng register/unregister for CCP TRNG | `CcpDevice::register_rng` / `_unregister_rng` |
| `ccp_enqueue_cmd(cmd)` | submit crypto cmd to round-robin selected CCP | `Subsystem::enqueue_cmd` |
| `ccp_dequeue_cmd(q)` / `ccp_do_cmd_complete(data)` | per-queue dispatch + completion | `CcpQueue::dequeue` / `_do_complete` |
| `ccp_trng_read(rng, data, max, wait)` | hwrng read callback | `CcpDevice::trng_read` |
| `sev_do_cmd(cmd, data, psp_ret)` | execute a SEV PSP command (LAUNCH_START, LAUNCH_UPDATE_DATA, ATTESTATION_REPORT, …) | `SevDevice::do_cmd` |
| `__sev_do_cmd_locked(cmd, data, psp_ret)` | locked variant; caller holds `sev_cmd_mutex` | `SevDevice::do_cmd_locked` |
| `__sev_platform_init_locked` / `__sev_firmware_shutdown` | platform init / shutdown sequence | `SevDevice::platform_init` / `_firmware_shutdown` |
| `snp_alloc_firmware_page` / `snp_free_firmware_page` | SNP-firmware-only page alloc (RMP firmware state) | `SevDevice::snp_alloc_fw_page` / `_free_fw_page` |
| `snp_reclaim_pages(paddr, npages, locked)` | reclaim SNP-firmware-owned page back to hypervisor | `SevDevice::snp_reclaim_pages` |
| `psp_send_platform_access_msg(...)` | PSP platform-access generic submission | `PspDevice::send_platform_access_msg` |
| `tee_init` / `tee_destroy` / `tee_send_cmd_buffer` | TEE TA interface | `TeeDevice::init` / `_destroy` / `_send_cmd` |
| `sfs_dev_init` / `dbc_dev_init` / `hsti_dev_init` | sub-feature init | `PspDevice::sfs_init` / `_dbc_init` / `_hsti_init` |
| `ccp5_offset_init` / `ccp5_init` | per-CCP v5 register layout + engine init | `CcpDevice::v5_init` |
| `ccp_dev_init` / `ccp_dev_destroy` | overall CCP lifecycle | `CcpDevice::init` / `_destroy` |
| `sev_irq_handler(irq, data, status)` | SEV-cmd-completion IRQ | `SevDevice::irq_handler` |

## Compatibility contract

REQ-1: Per-AMD-package PCI device binding — `sp-pci.c` registers PCI IDs covering AMD desktop/server CPUs and APUs (Ryzen, EPYC, Threadripper). One `struct sp_device` per package.

REQ-2: Per-package CCP probed against per-silicon version table: v3 (Carrizo / older), v5 (Ryzen+ / EPYC). Per-version op-table dispatch (`ccp_actions`) chosen at probe.

REQ-3: CCP exposes algorithms via crypto API: AES-{ECB,CBC,CTR,XTS,GCM,CCM,CMAC}, DES3, SHA-1/224/256/384/512, RSA, ECC-CDH, plus a TRNG.

REQ-4: CCP TRNG registered via `hwrng_register` at probe and exposed as `/dev/hwrng` consumer for `random.c` entropy mixing (cross-ref `drivers/char/random.md`).

REQ-5: PSP mailbox interface: per-package single mailbox with command + status doorbell registers; per-cmd buffer DMA-mapped from coherent pool; per-cmd serialized via `sev_cmd_mutex`.

REQ-6: SEV firmware command set (`psp-sev.h` UAPI) reachable via `/dev/sev` ioctl: `SEV_FACTORY_RESET`, `SEV_PLATFORM_STATUS`, `SEV_PEK_GEN`, `SEV_PEK_CSR`, `SEV_PEK_CERT_IMPORT`, `SEV_PDH_CERT_EXPORT`, `SEV_PDH_GEN`, `SEV_GET_ID`, `SEV_GET_ID2`, plus SNP commands (`SNP_PLATFORM_STATUS`, `SNP_COMMIT`, `SNP_SET_CONFIG`, `SNP_VLEK_LOAD`).

REQ-7: SEV-ES / SEV-SNP guest launch protocol: KVM-SEV uses `sev_do_cmd` from inside the kernel (no UAPI ioctl) — `LAUNCH_START`, `LAUNCH_UPDATE_DATA/VMSA/CPUID`, `LAUNCH_MEASURE`, `LAUNCH_SECRET`, `LAUNCH_FINISH`, `SEND_START/UPDATE_DATA/FINISH` (migration), `RECEIVE_START/UPDATE_DATA/FINISH`, `GUEST_STATUS`, `ATTESTATION_REPORT`.

REQ-8: SNP firmware ownership: per-`snp_alloc_firmware_page` allocates a page transitioning into firmware-owned state via RMP table update; `snp_reclaim_pages` reverses; sweep on shutdown.

REQ-9: TEE sub-device exposes mailbox for fTPM (Microsoft) and AMD TAs via the `tee` framework or `psp-platform-access` UAPI.

REQ-10: DBC (Dynamic Boost Control), SFS (Seamless Firmware Service), HSTI (Hardware Security Test Interface) expose `/dev/dbc`, `/sys/devices/.../sfs/`, `/sys/.../hsti/` surfaces — all CAP_SYS_ADMIN gated.

REQ-11: `init_ex` path: SEV firmware optional `INIT_EX` boot loading a 32 KB persistent buffer from a file (path via `init_ex_path` module param); enables "persisted PEK" across cold-boot.

REQ-12: Round-robin CCP selection — multi-socket EPYC has multiple CCPs; `ccp_get_device` cycles via `ccp_rr` spinlock-protected pointer balancing load across packages.

## Acceptance Criteria

- [ ] AC-1: `lspci -d 1022:` lists each AMD-package's Secure Processor; `dmesg | grep -i 'ccp\|psp\|sev'` shows probe of CCP + PSP + SEV sub-devices.
- [ ] AC-2: `cat /proc/crypto | grep ccp-` lists CCP-registered algorithms with priority > software fallback.
- [ ] AC-3: `cryptsetup benchmark --cipher aes-xts` shows CCP-AES-XTS throughput improvement vs software AES-NI fallback on EPYC.
- [ ] AC-4: `rngtest -c 1000 < /dev/hwrng` passes for CCP TRNG.
- [ ] AC-5: `/dev/sev` exists with mode 0600 (root-only); `SEV_PLATFORM_STATUS` ioctl returns valid version + state.
- [ ] AC-6: KVM SEV guest launch end-to-end: `qemu-system-x86_64 -object sev-guest,...` boots an Ubuntu CoCo image; `LAUNCH_*` commands logged in dmesg with `psp_ret=0`.
- [ ] AC-7: KVM SEV-SNP guest launch + attestation report: in-guest `snpguest report` returns a signed report; signature verifies against the platform PDH chain.
- [ ] AC-8: SNP page-state-change exercise: `kvmtool` reclaim of guest-private pages succeeds without `RMP #PF`.
- [ ] AC-9: `cryptsetup` + LUKS volume mount under `stress-ng --crypt 32 --timeout 300s` — no kmemleak, no DMA-debug leaks, no dmesg WARN from `ccp_*`.

## Architecture

`SpDevice` is the root per-package container:

```
struct SpDevice {
  pdev: Arc<PciDev>,           // sp-pci device handle
  io_map: NonNull<u8>,         // BAR2 (CCP + PSP MMIO)
  ccp: Option<KBox<CcpDevice>>,
  psp: Option<KBox<PspDevice>>,
  lock: SpinLock,
  irq: IrqLine,
}

struct CcpDevice {
  ord: u32,                    // global ordinal
  cmd_q: [CcpQueue; MAX_HW_QUEUES],
  cmd_count: AtomicUsize,
  backlog: WaitQueueHead,
  hwrng: HwrngHandle,
  dmaengine: DmaDevice,
  ksb_count: u32,              // key storage block count
  ksb_bitmap: SpinLockBitmap,  // KSB allocation bitmap
}

struct PspDevice {
  io_regs: NonNull<u8>,        // PSP mailbox MMIO offset within sp BAR
  sev: Option<KBox<SevDevice>>,
  tee: Option<KBox<TeeDevice>>,
  sfs: Option<KBox<SfsDevice>>,
  dbc: Option<KBox<DbcDevice>>,
  hsti: Option<KBox<HstiDevice>>,
  irq_status: AtomicU32,
  platform_access_mutex: Mutex,
}

struct SevDevice {
  cmd_mutex: Mutex,             // sev_cmd_mutex
  state: SevState,              // UNINIT / INIT / WORKING / SHUTDOWN
  api_major: u8,
  api_minor: u8,
  build: u8,
  misc: MiscDeviceHandle,       // /dev/sev
  cmd_buf: DmaCoherent<[u8; 2*PAGE_SIZE]>,
  init_ex_buffer: Option<DmaCoherent<[u8; SEV_INIT_EX_BUF_SIZE]>>,
  es_tmr: Option<DmaCoherent<[u8; SEV_TMR_SIZE]>>,
  snp_hv_fixed_pages: List<HvFixedEntry>,
  irq_complete: Completion,
  refcount: Refcount,
}
```

Probe sequence (`sp_pci_probe`):
1. PCI BAR2 ioremap → split into CCP region (offset 0) and PSP region (offset 0x10000).
2. Allocate `SpDevice` + per-sub-feature `CcpDevice` + `PspDevice`.
3. If CCP version supported: `ccp_dev_init(sp)` → reset CCP, init each hardware queue, register algorithms via `ccp_crypto_register`, register TRNG via `hwrng_register`, expose dmaengine channels.
4. If PSP present: `psp_dev_init(sp)` → reset PSP, init mailbox, request firmware via `request_firmware("amd/amd_sev_fam17h_modelXxh.sbin")` for SEV.
5. Sub-device init: `sev_dev_init`, `tee_init`, `sfs_dev_init`, `dbc_dev_init`, `hsti_dev_init`. Each is independently feature-gated by Kconfig + by what the PSP firmware advertises.
6. SEV `__sev_platform_init_locked` if `psp_init_on_probe=true`: `INIT` (or `INIT_EX` with persistent buf) → state → WORKING.

Crypto request flow (`ccp_enqueue_cmd`):
1. Round-robin select CCP via `ccp_get_device`.
2. Allocate per-cmd descriptor; DMA-map per-SG buffers; reserve Key Storage Block slot.
3. Append to per-queue ring + ring doorbell.
4. HW completes → IRQ → `ccp_do_cmd_complete` → tasklet runs `cmd->callback(err, data)`.
5. crypto API `crypto_request_complete(req, err)` resumes consumer.

SEV command flow (`sev_do_cmd`):
1. Acquire `sev_cmd_mutex`.
2. If SNP-legacy-handling-needed: `snp_prep_cmd_buf` builds + maps per-buffer descriptors (`cmd_buf_desc`).
3. Copy command payload into `sev_dev->cmd_buf` (DMA-coherent staging buffer).
4. Write CMD register + ring PSP doorbell.
5. Sleep on `irq_complete` until IRQ handler signals completion (timeout `psp_cmd_timeout`, default 100s).
6. Read CMD status — `psp_ret` filled.
7. If write-back required (status reports): copy response from `cmd_buf` to caller payload.
8. If SNP: `snp_reclaim_cmd_buf` reverts pages from firmware-owned to hypervisor-owned via RMP update.
9. Release mutex.

SNP firmware page lifecycle (`snp_alloc_firmware_page` → `snp_free_firmware_page`):
1. `alloc_pages(GFP_KERNEL, order)` from kernel pool.
2. `rmp_mark_pages_firmware(paddr, npages, locked)` — RMP entry flipped to firmware-owned; CPU MUST NOT touch page until reclaimed.
3. Use via SEV/SNP commands.
4. On free: `snp_reclaim_pages(paddr, npages, locked)` flips RMP entry back; `__free_pages`.

TRNG read (`ccp_trng_read`):
1. Read TRNG status; if not ready and `wait`, sleep + retry up to `max` size.
2. Stream `min(max, FIFO_DEPTH)` bytes from TRNG FIFO into `data`.
3. Return bytes read.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ccp_cmd_no_oob` | OOB | per-cmd SG walk + KSB indexing strictly bounded by `ccp->ksb_count`. |
| `sev_cmd_buf_no_oob` | OOB | `cmd_buf` writes bounded by `sev_cmd_buffer_len(cmd)`; refuse oversize payload. |
| `snp_page_no_uaf` | UAF | every `snp_alloc_firmware_page` paired with `snp_reclaim_pages` before `__free_pages`. |
| `sev_state_machine` | STATE | `state` transitions strictly UNINIT→INIT→WORKING→SHUTDOWN; no skip. |
| `sev_cmd_mutex_acq` | LOCK | every `__sev_do_cmd_locked` precondition asserts `mutex_is_locked(&sev_cmd_mutex)`. |

### Layer 2: TLA+

`models/ccp/psp_mailbox.tla` (parent-declared): proves PSP mailbox doorbell + completion-IRQ + command-buffer-coherent-DMA observe ordering required by AMD PSP spec; concurrent submit from N CPUs serializes via `sev_cmd_mutex`; SNP page-state transitions linearize against RMP table.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post-`sev_do_cmd(LAUNCH_FINISH)`: guest `state == RUNNING`; measurement stable; no further `LAUNCH_*` accepted | `SevDevice::do_cmd` |
| Post-`sev_do_cmd(ATTESTATION_REPORT)`: report signature verifies under platform VCEK / VLEK chain | `SevDevice::do_cmd` |
| Per-`ccp_enqueue_cmd`: per-cmd descriptor freed exactly once after callback invocation; no double-free | `CcpQueue::dequeue` |
| Per-`ccp_trng_read`: bytes returned to consumer pass continuous-bit-pattern health check | `CcpDevice::trng_read` |

### Layer 4: Verus/Creusot functional

End-to-end: KVM userland `KVM_SEV_LAUNCH_START` → kernel `sev_issue_cmd(LAUNCH_START)` → PSP firmware → handle assigned → subsequent `LAUNCH_UPDATE_DATA` measurement extends consistent with VMSA contents → `LAUNCH_FINISH` measurement matches local SHA-384 over the union of inputs. Encoded as Verus invariant chained from `arch/x86/kvm/svm/sev.md`.

## Hardening

(Inherits row-1 features from `drivers/crypto/00-overview.md` § Hardening.)

ccp/psp-specific reinforcement:

- **SEV firmware signature validation** — `request_firmware` for `amd_sev_*.sbin` enforces `CONFIG_FW_LOADER_SIG_FORCE=y`; unsigned/expired firmware refused; firmware version compared against `MIN_FW_VERSION` table per silicon.
- **`/dev/sev` mode 0600 (root-only)** — misc device created with restrictive default; no group-write.
- **`sev_cmd_mutex` enforced** — all `_locked` variants assert mutex held; no path to PSP doorbell bypasses serialization.
- **PSP doorbell timeout** — `psp_cmd_timeout` defaults to 100s; on timeout, `psp_dead` flag set; further commands return `-ENODEV` until reset.
- **SNP RMP discipline** — `snp_reclaim_pages` MUST succeed before `__free_pages`; failure path WARN + leak rather than UAF.
- **`init_ex_path` validated** — buffer path string sanity-checked; file opened as root via `open_file_as_root`; 32 KB cap on file size.
- **PSP IRQ status read in one MMIO transaction** — defense against torn-read attacks against the mailbox.
- **CCP Key Storage Block (KSB) per-allocation zeroed on free** — `ksb_bitmap` releases mark slot, then KSB content cleared.
- **Per-cmd backlog bound** — `ccp_cmd_queue->backlog` capped to defeat per-context flood DoS.
- **Suspend/resume invalidates SEV state** — `ccp_dev_suspend` drains in-flight; resume re-INITs from clean state, no resume-with-stale-handles.
- **DMA-coherent staging buffers** — `cmd_buf`, `init_ex_buffer`, `es_tmr` allocated from coherent pool with no cross-VM bleed.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `ccp_cmd`, `ccp_cmd_queue`, `sev_device`, `psp_device`, and `tee_device` allocations; SEV ioctl userland copies bounded to `sev_cmd_buffer_len(cmd_id)`; PSP mailbox `cmd_buf` carved out of a dedicated coherent slab with PAX_USERCOPY-strict bounds.
- **PAX_KERNEXEC** — `sev_misc_ops`, `psp_master_ops`, `ccp_actions` v3/v5 vtables, IRQ handlers, and TEE TA dispatch tables in `__ro_after_init` text; PSP firmware staging region permanent-W-NX.
- **PAX_RANDKSTACK** — randomize stack across `sev_do_cmd`, `ccp_enqueue_cmd`, `__sev_do_cmd_locked`, `snp_alloc_firmware_page`, and SEV ioctl entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `sev_device`, `psp_device`, per-CCP `ccp_device`, per-cmd `ccp_cmd`, SEV launch-handle, and TEE TA session refcounts; overflow trap defeats unplug/free races (e.g., concurrent `/dev/sev` open + driver remove).
- **PAX_MEMORY_SANITIZE** — zero-on-free for `cmd_buf` staging, KSB slots, TEE session buffers, SEV `init_ex_buffer`, ES TMR region, and per-VM `LAUNCH_SECRET` payload slab; ensures SEV key/secret material does not bleed across consumers.
- **PAX_UDEREF** — SMAP/PAN enforced on every `/dev/sev` ioctl, TEE submission, DBC IOCTL, and platform-access mailbox entry; no kernel-side user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — every PSP/SEV/TEE/SFS/DBC/HSTI vtable indirect call (cmd dispatch, IRQ handler chain, SEV-misc `file_operations`) kCFI-typed; mismatched call-site trapped.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of PSP mailbox MMIO base, SEV firmware load address, RMP page-table pointer, and CCP BAR addresses behind CAP_SYSLOG; `%p` suppressed in tracepoints.
- **GRKERNSEC_DMESG** — restrict SEV-firmware-error, RMP-violation, PSP-timeout, and CCP-parity banners to CAP_SYSLOG so SEV-related state machines cannot be probed from dmesg.
- **CAP_SYS_ADMIN strict** — `/dev/sev`, `/dev/tee`, `/dev/dbc`, debugfs entries (`ccp_debugfs.c`), and HSTI sysfs nodes require CAP_SYS_ADMIN in init userns; refuse user-ns delegation by default.
- **SEV firmware signature mandatory** — `request_firmware_direct` paths configured so unsigned `amd_sev_*.sbin` is refused; firmware version floor enforced per silicon family.
- **SNP RMP-page integrity gate** — `snp_alloc_firmware_page` failure (RMP update fail) returns `NULL`; `snp_reclaim_pages` failure WARN + leak + refuse-free rather than UAF.
- **PSP-dead flag latched** — once `psp_dead=true`, every entry point returns `-ENODEV`; recovery requires explicit module reload (audited).
- **SEV launch-handle PAX_REFCOUNT** — KVM-SEV launch contexts hold refcounted handles; defeat double-`LAUNCH_FINISH` / use-after-`SEV_GUEST_DEACTIVATE` races.
- **DBC signed-firmware enforcement** — Dynamic-Boost mailbox refuses unsigned policy blobs; signature chain validated against per-platform CA.

Rationale: the AMD-SP is the root of trust for every SEV / SEV-ES / SEV-SNP guest in the system, the host's source of high-quality entropy via TRNG, and the in-kernel mediator for fTPM secrets. A refcount underflow on a SEV launch-handle, an unsigned firmware acceptance, a missed RMP reclaim, or a PSP-mailbox race silently demotes confidential-computing isolation. RAP/kCFI on the PSP/SEV/TEE dispatch tables, CAP_SYS_ADMIN gating on `/dev/sev` + `/dev/dbc`, FW signature enforcement, RMP-integrity gates, refcount-overflow trapping, and explicit `memzero_explicit` on launch-secret slabs turn the AMD-SP driver from "trusted because it's the firmware" into a structural CoCo enforcement boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM-side SEV/SNP control plane (covered in `arch/x86/kvm/svm/sev.md` future Tier-3)
- RMP table management on the page-allocator side (covered in `arch/x86/mm/snp.md` future Tier-3)
- TEE userspace TA loader (covered in `drivers/tee/00-overview.md` future Tier-3)
- DMA-engine subsystem proper (covered in `drivers/dma/00-overview.md` future Tier-3)
- 32-bit-only paths (CCP exists on AMD64 only)
- Implementation code
