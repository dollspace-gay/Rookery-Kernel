# Tier-3: drivers/crypto/caam/ — NXP CAAM (Cryptographic Acceleration and Assurance Module, i.MX 6/7/8 + Layerscape)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/crypto/00-overview.md
upstream-paths:
  - drivers/crypto/caam/ctrl.c
  - drivers/crypto/caam/ctrl.h
  - drivers/crypto/caam/jr.c
  - drivers/crypto/caam/jr.h
  - drivers/crypto/caam/caamalg.c
  - drivers/crypto/caam/caamalg_desc.c
  - drivers/crypto/caam/caamalg_desc.h
  - drivers/crypto/caam/caamalg_qi.c
  - drivers/crypto/caam/caamalg_qi2.c
  - drivers/crypto/caam/caamalg_qi2.h
  - drivers/crypto/caam/caamhash.c
  - drivers/crypto/caam/caamhash_desc.c
  - drivers/crypto/caam/caamhash_desc.h
  - drivers/crypto/caam/caampkc.c
  - drivers/crypto/caam/caampkc.h
  - drivers/crypto/caam/caamprng.c
  - drivers/crypto/caam/caamrng.c
  - drivers/crypto/caam/blob_gen.c
  - drivers/crypto/caam/key_gen.c
  - drivers/crypto/caam/key_gen.h
  - drivers/crypto/caam/qi.c
  - drivers/crypto/caam/qi.h
  - drivers/crypto/caam/dpseci.c
  - drivers/crypto/caam/dpseci.h
  - drivers/crypto/caam/dpseci-debugfs.c
  - drivers/crypto/caam/desc.h
  - drivers/crypto/caam/desc_constr.h
  - drivers/crypto/caam/regs.h
  - drivers/crypto/caam/error.c
  - drivers/crypto/caam/intern.h
  - drivers/crypto/caam/pdb.h
  - drivers/crypto/caam/pkc_desc.c
  - drivers/crypto/caam/sg_sw_sec4.h
  - drivers/crypto/caam/sg_sw_qm.h
  - drivers/crypto/caam/sg_sw_qm2.h
  - include/soc/fsl/caam-blob.h
  - include/keys/trusted_caam.h
-->

## Summary

`drivers/crypto/caam/` is the driver for NXP/Freescale's CAAM ("Cryptographic Acceleration and Assurance Module"), the in-SoC crypto block on i.MX 6/7/8 family, QorIQ, and Layerscape parts. CAAM is more than a crypto offload — it is a programmable descriptor engine with secure-memory partitions, hardware key blob wrap/unwrap, RNG4 (a SP800-90 deterministic + non-deterministic random number generator), trusted-key support (via the kernel `trusted_keys` framework), and FIPS-140-2 boundary semantics.

CAAM expresses every operation as a microcode-like **descriptor** assembled from CAAM-instruction-set words (LOAD, STORE, FIFO_LOAD, FIFO_STORE, ALGORITHM, JUMP, MOVE, ...). The driver constructs these descriptors per crypto API request, submits to a hardware ring (one of three transport backends), and reads results.

Three transport variants:
- **Job Rings (JR)** — classical i.MX 6/7/8 transport; per-ring producer/consumer dequeue (`drivers/crypto/caam/jr.c` + `caamalg.c` / `caamhash.c` / `caampkc.c`).
- **QI (Queue Interface)** — Layerscape QMan/BMan hardware portal transport (`drivers/crypto/caam/qi.c` + `caamalg_qi.c`).
- **QI2 / DPSECI** — Layerscape Gen2 DPAA2 transport via DPSECI MC object (`drivers/crypto/caam/dpseci.c` + `caamalg_qi2.c`).

This Tier-3 covers `ctrl.c` (per-SoC init), `jr.c` (job-ring transport), the algorithm registration glue, blob/key wrap, RNG4 init + health, and secure-memory partition usage. The two Layerscape transports are noted but not exhaustively dissected (each is comparable in complexity to a Tier-3 of its own).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct caam_drv_private` | per-CAAM control block | `drivers::caam::CaamPrivate` |
| `struct caam_drv_private_jr` | per-job-ring control block | `drivers::caam::JrPrivate` |
| `struct caam_jrentry_info` | per-job-ring queued descriptor | `drivers::caam::JrEntry` |
| `struct caam_ctrl __iomem` | CAAM controller MMIO layout | `drivers::caam::CtrlReg` |
| `struct caam_perfmon __iomem` | CAAM perfmon registers (CHA version + era) | `drivers::caam::Perfmon` |
| `caam_probe(pdev)` | per-platform-device probe | `Subsystem::probe` |
| `caam_jr_probe(pdev)` | per-job-ring child platform-device probe | `JrDevice::probe` |
| `caam_jr_alloc()` / `caam_jr_free(rdev)` | pick a job-ring (round-robin) / release | `Subsystem::jr_alloc` / `_jr_free` |
| `caam_jr_enqueue(dev, desc, cb, areq)` | submit a descriptor to a JR | `JrDevice::enqueue` |
| `caam_jr_dequeue(devarg)` | per-JR completion tasklet | `JrDevice::dequeue` |
| `caam_jr_interrupt(irq, st_dev)` | per-JR IRQ handler | `JrDevice::irq_handler` |
| `caam_jr_init_hw(...)` / `caam_reset_hw_jr(...)` / `caam_jr_shutdown(...)` | per-JR lifecycle | `JrDevice::init_hw` / `_reset_hw` / `_shutdown` |
| `instantiate_rng(ctrldev, state_handle_mask, gen_sk)` | RNG4 instantiate / re-seed handles | `CaamPrivate::instantiate_rng` |
| `deinstantiate_rng(ctrldev, state_handle_mask)` | RNG4 deinstantiate handles | `CaamPrivate::deinstantiate_rng` |
| `kick_trng(dev, ent_delay)` | TRNG entropy source kick (LFSR) | `CaamPrivate::kick_trng` |
| `caam_get_era(perfmon)` / `caam_get_era_from_hw(perfmon)` | per-CAAM era (revision) | `CaamPrivate::era` |
| `caam_ctrl_suspend(dev)` / `caam_ctrl_resume(dev)` / `caam_state_save(dev)` / `caam_state_restore(dev)` | per-CAAM suspend/resume | `CaamPrivate::suspend` / `_resume` / `_state_save` / `_state_restore` |
| `caam_alg_register` / `caam_aead_register` / `caam_hash_register` / `caam_pkc_register` / `caam_rng_register` | per-class algorithm registration | `Subsystem::register_*` |
| `caam_blob_gen_init(dev)` / `caam_encap_blob(dev, info)` / `caam_decap_blob(dev, info)` | blob-key wrap / unwrap | `BlobGen::encap` / `_decap` |
| `caam_qi_init(pdev)` / `caam_qi_enqueue(dev, fd)` | QI transport (Layerscape) | `QiTransport::init` / `_enqueue` |
| `dpaa2_caam_probe(ls_dev)` / `dpaa2_caam_enqueue(...)` | DPSECI transport (Layerscape Gen2) | `Dpseci::probe` / `_enqueue` |
| `gen_split_key(dev, key_out, alg, key_in, keylen, mdsize)` | split-key generation for AES-HMAC inner+outer | `KeyGen::gen_split_key` |

## Compatibility contract

REQ-1: Per-SoC probe via DT match (`fsl,sec-v4.0`, `fsl,imx6-caam`, `fsl,imx7-caam`, `fsl,imx8-caam`, ...) or fsl-mc match (Layerscape Gen2 DPSECI). `caam_probe` initializes the controller; per-job-ring children (`fsl,sec-v4.0-job-ring`) instantiated as separate platform devices probed by `caam_jr_probe`.

REQ-2: Per-CAAM-era detection via `caam_perfmon.ccbvid.era` (or DT fallback); per-era cap discovery (e.g., RNG4 vs RNG/no-RNG, CHA versions for AES, DES, HASH, PK).

REQ-3: RNG4 instantiation: per-state-handle (typically 0..3) `RNG4_HANDLE_INSTANTIATE` descriptor submitted; on failure `kick_trng` adjusts the LFSR entropy delay parameter and retries (`needs_entropy_delay_adjustment` per-SoC). All state handles instantiated before any crypto class registered.

REQ-4: Per-job-ring lifecycle: alloc + `caam_jr_init_hw` initializes JR input/output ring base addresses; ring sizes per `JOBR_DEPTH`; IRQ registered; tasklet pinned to ring.

REQ-5: Per-request descriptor construction: shared descriptor (per-tfm, contains expanded key) + per-request job-descriptor (references shared descriptor + per-request scatterlists + IV).

REQ-6: Algorithm registration: `caamalg_register_skcipher` + `caamalg_register_aead` + `caamhash_register` + `caampkc_register` + `caamrng_register`. Active devs counted; first device of a kind registers, last unregisters.

REQ-7: Blob-key wrap/unwrap via `caam_encap_blob` / `caam_decap_blob`: opaque key blob (encrypted with CAAM's per-SoC OTPMK or NVRAM-derived key) provides offline persistence. Used by kernel `trusted_keys` framework (CONFIG_TRUSTED_KEYS=y, type "caam") for trusted-key support equivalent to TPM2 trusted keys (cross-ref `security/keys/00-overview.md`).

REQ-8: Secure-Memory (SM) partition: per-CAAM has 4 secure-memory partitions (per-i.MX 6 era; varies by SoC). Used for in-CAAM key storage (key material never leaves SM in plaintext over the MMIO bus). Allocation via per-partition group + page descriptors; CAP_SYS_RAWIO gated.

REQ-9: Three transport backends: JR (default), QI (Layerscape Gen1), QI2/DPSECI (Layerscape Gen2). Per-platform DT decides which transport is registered. Algorithm registration tables shared but transport-glue per-backend.

REQ-10: Per-i.MX errata workarounds: `handle_imx6_err005766` (RNG init), `needs_entropy_delay_adjustment` (entropy-delay sweep), per-SoC clock-bulk init (`caam_imx6_clks`, `caam_imx7_clks`, `caam_imx6ul_clks`, `caam_vf610_clks`).

REQ-11: Suspend/resume: `caam_state_save` snapshots CAAM controller register state to a per-CAAM blob; `caam_state_restore` restores on resume; refused if `caam_off_during_pm` (some i.MX SoCs power CAAM off in S3).

REQ-12: RNG4 continuous health-check: HW provides health-test failure interrupt; failure triggers RNG deinstantiate + re-instantiate; persistent failure logged + RNG unregistered.

## Acceptance Criteria

- [ ] AC-1: i.MX 8M Mini boot: `dmesg | grep -i caam` shows CAAM controller probe, RNG4 instantiation success, 4 job-rings probed.
- [ ] AC-2: `cat /proc/crypto | grep caam` shows registered skcipher (`xts-aes-caam`, `cbc-aes-caam`, ...), aead (`gcm-aes-caam`, `authenc-hmac-sha256-cbc-aes-caam`, ...), ahash (`sha256-caam`, ...), akcipher (`rsa-caam`), and rng (`stdrng-caam`).
- [ ] AC-3: `cryptsetup benchmark` shows CAAM-AES-XTS speed-up vs software on i.MX 8.
- [ ] AC-4: `rngtest -c 1000 < /dev/hwrng` passes (CAAM TRNG / RNG4 contributing).
- [ ] AC-5: Trusted-key test: `keyctl add trusted kmk "new 32 keyhandle=caam" @s` creates a CAAM-blob-wrapped key; reboot + restore round-trips.
- [ ] AC-6: Blob round-trip test: `caam_encap_blob(plain) → blob → caam_decap_blob(blob) → plain` byte-exact recovery.
- [ ] AC-7: tcrypt KAT vectors pass for all CAAM-registered algorithms.
- [ ] AC-8: Suspend/resume cycle: S3 → resume → algorithms still registered with same priority; no crypto requests fail across the cycle.
- [ ] AC-9: RNG4 health-check fault injected (via clock manipulation): driver detects, deinstantiates, re-instantiates; no entropy contributed during failure window.

## Architecture

`CaamPrivate` is the per-controller root:

```
struct CaamPrivate {
  ctrl: NonNull<CtrlReg>,         // CAAM controller MMIO
  jrpdev: Vec<Arc<JrDevice>>,     // per-job-ring child platform devices
  total_jobrs: u32,
  qi_present: bool,
  qi: Option<KBox<QiTransport>>,
  dpseci: Option<KBox<Dpseci>>,
  era: u8,                        // CAAM era (e.g., 8, 9, 10)
  caam_state: Option<KBox<CaamStateBlob>>,
  jrindex: AtomicUsize,           // round-robin pointer
  rng_handles_instantiated: u8,   // bitmask of instantiated RNG4 state handles
  clks: Vec<ClkBulk>,             // per-SoC clock-bulk handles
  blob_priv: Option<KBox<BlobGen>>,
  debugfs_root: Option<DebugfsRoot>,
}

struct JrDevice {
  jrregs: NonNull<JrRegs>,        // per-JR MMIO
  inpring: DmaCoherent<[u64; JOBR_DEPTH]>,   // input ring
  outring: DmaCoherent<[OutRingEntry; JOBR_DEPTH]>,   // output ring
  entinfo: [JrEntry; JOBR_DEPTH], // per-slot per-cmd info
  inp_ring_write_index: SpinLockU32,
  out_ring_read_index: AtomicU32,
  head: u32,
  tail: u32,
  irq: IrqLine,
  tasklet: Tasklet,
  engine: CryptoEngine,
  hwrng: Option<HwrngHandle>,
  refcount: Refcount,
}

struct CaamStateBlob { /* opaque per-CAAM register dump for suspend/resume */ }
```

Probe sequence (`caam_probe`):
1. DT match → identify SoC family (i.MX6/7/8, Layerscape, vf610, ULP).
2. `init_clocks(dev, data)` — enable per-SoC clocks via `clk_bulk_*`.
3. ioremap controller; read `era` from perfmon.
4. Apply per-SoC errata (`handle_imx6_err005766`).
5. `caam_ctrl_rng_init(dev)`:
   - For each RNG4 state handle (typically 0..3): try `instantiate_rng(...)`.
   - On failure: `kick_trng(dev, ent_delay)` increases entropy-delay; retry.
   - Once instantiated: register `devm_add_action_or_reset(devm_deinstantiate_rng)` to teardown.
6. For each `fsl,sec-v4.0-job-ring` child node: `of_platform_populate` instantiates per-JR platform device → `caam_jr_probe` runs.
7. Per-JR: ioremap JR MMIO, alloc input + output rings (DMA-coherent, JOBR_DEPTH=1024 typical), register IRQ + tasklet, init crypto-engine, register self in `jr_driver_data.jr_list`.
8. Once first JR ready: register algorithms (`caam_algapi_init`, `caam_algapi_hash_init`, `caam_pkc_init`, `caam_rng_init`, `caam_blob_gen_init`).

Per-request descriptor flow (`caam_jr_enqueue`):
1. Caller builds per-cmd job descriptor referencing per-tfm shared descriptor + per-request SG addresses.
2. Pick free `entinfo[head]` slot.
3. Record callback + areq + DMA-mapped descriptor pointer.
4. Write descriptor DMA address into `inpring[head]`.
5. DMA-flush input ring; bump `head` (wrap on `JOBR_DEPTH`).
6. Write JR input-ring jobs-added register.
7. Return `-EINPROGRESS`.

Completion (`caam_jr_interrupt` → `caam_jr_dequeue`):
1. IRQ raises on output-ring jobs-completed.
2. Tasklet runs `caam_jr_dequeue`.
3. For each entry in output ring: read descriptor address + status word.
4. Match against `entinfo[tail]`; call `callback(dev, desc, status, areq)`.
5. Advance `tail`; write JR output-ring removed-jobs register.

RNG4 instantiation (`instantiate_rng`):
1. Build instantiation descriptor (`build_instantiation_desc`) targeting each requested state-handle bit.
2. `run_descriptor_deco0` — execute on DECO0 (descriptor controller 0).
3. Poll status; if failure, return -EAGAIN.
4. `kick_trng(dev, ent_delay+1)` widen entropy delay; retry from `caam_ctrl_rng_init`.

Blob encap (`caam_encap_blob`):
1. Build CAAM blob-encap descriptor referencing input key + blob-key class + output blob buffer.
2. Submit via JR; wait completion.
3. Output = 48 bytes of metadata + ciphertext + MAC (per CAAM "Black Blob" format).
4. Blob can be persisted to non-secure storage; decap only succeeds on same CAAM with same OTPMK.

Suspend (`caam_ctrl_suspend`):
1. Quiesce JRs: `caam_jr_stop_processing` / `caam_jr_flush`.
2. `caam_state_save(dev)` — snapshot controller registers to `caam_state` blob.
3. Disable CAAM clocks.

Resume (`caam_ctrl_resume`):
1. Re-enable clocks.
2. `caam_state_restore(dev)` — restore controller registers from `caam_state` blob.
3. Restart each JR via `caam_jr_restart_processing`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `jr_ring_no_oob` | OOB | per-JR input/output ring indices wrap correctly within `JOBR_DEPTH`. |
| `desc_no_oob` | OOB | per-request descriptor word count < `MAX_CAAM_DESCSIZE` (64 words). |
| `rng_state_handle_valid` | BOUNDS | `state_handle_mask` only addresses bits 0..3 (valid handle count). |
| `blob_size_no_oob` | OOB | blob input/output sizes clamped against CAAM "Black Blob" max. |
| `sm_partition_valid` | BOUNDS | secure-memory partition id strictly bounded by per-SoC partition count. |

### Layer 2: TLA+

`models/caam/job_ring.tla` (parent-declared): proves per-JR input/output ring producer/consumer protocol — concurrent enqueue from N CPUs + IRQ-driven dequeue — observes head/tail FIFO ordering required by spec; no descriptor reordering across slots.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post-`instantiate_rng`: every requested state-handle bit reflected in `rng_handles_instantiated` AND HW state register agrees | `CaamPrivate::instantiate_rng` |
| Per-`caam_jr_enqueue`: per-slot `entinfo` filled before `inp_jobs_added` write; cleared after callback invocation | `JrDevice::enqueue` |
| Per-`caam_encap_blob` → `_decap_blob` round-trip: bit-exact recovery of original plaintext | `BlobGen::encap` / `_decap` |
| Per-`caam_state_save` → `_state_restore` round-trip: every saved register matches restored register (modulo per-volatile clear-on-read) | `CaamPrivate::suspend` / `_resume` |

### Layer 4: Verus/Creusot functional

End-to-end: trusted-key creation: `keyctl add trusted kmk "new 32"` → caam-trusted-key handler → `caam_encap_blob` produces 80-byte blob → stored in keyring → reboot → `keyctl read` → `caam_decap_blob` recovers original 32-byte key material → bit-exact match. Encoded as Verus invariant chained from `security/keys/trusted.md`.

## Hardening

(Inherits row-1 features from `drivers/crypto/00-overview.md` § Hardening.)

CAAM-specific reinforcement:

- **RNG4 health-check failure observed** — HW health-test failure IRQ triggers RNG deinstantiate; no entropy contributed during failure; persistent failure logs + unregisters.
- **Blob-key OTPMK never reachable from kernel** — wrap/unwrap descriptors reference OTPMK by class only; OTPMK bits never read into a CPU register.
- **Secure-memory partition allocation CAP_SYS_RAWIO-gated** — explicit capability check on SM partition assign + page-descriptor write.
- **Per-JR ring DMA-coherent + dma-fenced** — input/output rings allocated from coherent pool; writes ordered with DMA fence before tail-register update.
- **Per-i.MX clock-disable on suspend** — CAAM clocks gated when not in active state; defense against side-channel via leaked clock domain.
- **Per-descriptor SG count clamped** — `MAX_SG_TABLE_ENTRIES` enforced; refuse oversize SGLs.
- **Per-shared-descriptor refcount on tfm lifetime** — shared descriptor freed only after last per-request job done.
- **DT match validates SoC compatibility** — refuse unsupported SoC string; defense against forced probe on incompatible HW.
- **RNG4 entropy-delay sweep bounded** — `kick_trng` increments `ent_delay` up to per-SoC max; failure beyond max aborts init.
- **Per-RNG4-handle deinstantiate on driver remove** — `devm_deinstantiate_rng` registered at instantiation; teardown deterministic.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `caam_drv_private`, per-JR ring slots, per-request descriptor buffers, blob-encap/decap staging, and shared-descriptor caches; trusted-key blob copies bounded by CAAM "Black Blob" max.
- **PAX_KERNEXEC** — per-class algorithm vtables (`caamalg_skcipher_template`, `caamalg_aead_template`, `caamhash_template`, `caampkc_template`, `caamrng_template`), `caam_ctrl_pm_ops`, `caam_jr_pm_ops`, per-transport (`jr`, `qi`, `dpseci`) dispatch tables live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `caam_jr_enqueue`, `instantiate_rng`, `caam_encap_blob`/`_decap_blob`, RNG IRQ tasklet, and JR completion tasklet.
- **PAX_REFCOUNT** — saturating `refcount_t` on `caam_drv_private`, per-JR `caam_drv_private_jr`, per-tfm shared-descriptor handles, trusted-key references, and blob-gen lifetime; overflow trap defeats rmmod-during-IO race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-tfm shared descriptors (which contain expanded key schedule), per-request job descriptors, blob staging buffers, RNG4 instantiation buffers, suspend/resume state blobs; key material never bleeds across slab reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every AF_ALG / trusted-key / keyctl entry into CAAM; no user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `caam_alg`, `caam_aead_alg`, `caam_hash_template`, `caam_akcipher_alg`, `caam_rng_alg` vtable indirect calls, per-transport (`jr`/`qi`/`dpseci`) enqueue dispatchers, and pm-ops dispatch kCFI-typed.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of CAAM controller MMIO base, per-JR ring DMA addresses, secure-memory partition pointers, and OTPMK class identifiers behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict CAAM error (`caam_status` desc-status decode), RNG4 health-fault, secure-memory access-violation, and blob-decap failure banners to CAP_SYSLOG.
- **CAP_SYS_RAWIO on secure-memory partition operations** — SM partition assignment + page-descriptor writes require CAP_SYS_RAWIO in init userns; defense against userland claiming SM partitions for arbitrary use.
- **CAP_SYS_ADMIN on blob-gen operations from userspace** — keyctl trusted-key creation via CAAM goes through `trusted_keys` framework which enforces CAP_SYS_ADMIN on policy ops.
- **RNG4 health-test kill switch** — continuous-health-test failure deinstantiates RNG immediately; no entropy contributed until re-instantiation succeeds and passes acceptance.
- **Blob-key wrap class enforced** — `caam_encap_blob` MUST request a blob-key class (OTPMK-backed); refuse class 0 (zero key) or out-of-range class.
- **Per-JR irq affinity audited** — JR IRQ pinning controlled via sysfs; CAP_SYS_NICE / CAP_SYS_ADMIN guarded.
- **Per-tfm shared-descriptor key material** — expanded key schedule lives in shared descriptor; freed via `memzero_explicit` + `kfree`.

Rationale: CAAM is the trust root for every i.MX / Layerscape device — boot-image-signature verification, trusted-key wrap, secure-memory key storage, and the only hardware RNG on most NXP SoCs. A refcount underflow on a JR while a trusted-key operation is in flight, a missed RNG health-check failure, a relaxed secure-memory permission, or an unzeroed shared-descriptor slab silently demotes blob-key confidentiality. RAP/kCFI on the per-class vtables, CAP_SYS_RAWIO on SM partition writes, RNG4 continuous-health-test kill switch, refcount-overflow trapping, and explicit `memzero_explicit` on shared descriptors turn CAAM from "trusted because NXP signed the SoC" into a structural CoCo / trusted-key boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- QI / QI2 transport detail (covered in `drivers/crypto/caam/qi.md` future Tier-3)
- DPSECI / fsl-mc bus integration (covered in `drivers/bus/fsl-mc.md` future Tier-3)
- Trusted-keys framework proper (covered in `security/keys/trusted.md` future Tier-3)
- CAAM signed-boot use (HABv4) — out of kernel proper
- Per-SoC clock topology (covered in `drivers/clk/imx/` future)
- 32-bit-only paths (where i.MX6 32-bit drops out of v0 port set)
- Implementation code
