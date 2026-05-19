# Tier-3: drivers/crypto/ — kernel crypto offload driver class (CCP, QAT, CAAM, virtio-crypto, Chelsio, et al.)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/00-overview.md
upstream-paths:
  - drivers/crypto/Kconfig
  - drivers/crypto/Makefile
  - drivers/crypto/ccp/
  - drivers/crypto/intel/qat/
  - drivers/crypto/caam/
  - drivers/crypto/virtio/
  - drivers/crypto/chelsio/
  - drivers/crypto/cavium/
  - drivers/crypto/qce/
  - drivers/crypto/stm32/
  - drivers/crypto/atmel-aes.c
  - drivers/crypto/atmel-sha.c
  - drivers/crypto/atmel-tdes.c
  - drivers/crypto/padlock-aes.c
  - drivers/crypto/padlock-sha.c
  - drivers/crypto/hifn_795x.c
  - drivers/crypto/geode-aes.c
  - include/linux/crypto.h
  - include/crypto/internal/skcipher.h
  - include/crypto/internal/aead.h
  - include/crypto/internal/hash.h
  - include/crypto/internal/akcipher.h
  - include/crypto/internal/rng.h
-->

## Summary

`drivers/crypto/` is the kernel's HARDWARE crypto offload driver class. Each subdirectory or top-level `.c` file binds a specific hardware crypto accelerator to one or more crypto-API algorithm types (skcipher, AEAD, ahash, akcipher, rng) registered with the generic crypto framework at `crypto/` (cross-ref `.design/crypto/00-overview.md`, `.design/crypto/algapi.md`, `.design/crypto/aead.md`, `.design/crypto/skcipher.md`, `.design/crypto/af-alg.md`).

The kernel crypto API performs algorithm lookup by name+priority; hardware drivers register algorithms with **higher priority** than software fallbacks (`crypto/aes_generic.c`, `crypto/sha256_generic.c`). The crypto core picks the highest-priority registered algorithm that satisfies the consumer mask, automatically using hardware offload when present and falling back to software (or a CPU-instruction accelerator like AES-NI / SHA-NI / ARM CE) when not.

This Tier-3 overview enumerates the major device families, the shared driver-class patterns (registration, request lifecycle, fallback, RNG plumbing, FIPS gating), and the policy posture grsec applies across the entire class. Per-driver Tier-3 docs follow as siblings (`ccp.md`, `qat.md`, `caam.md`, `virtio-crypto.md`, `chelsio-crypto.md`) covering the four most-deployed silicon families plus the paravirtualized backend.

## Upstream references

| Upstream subtree | Vendor / silicon | Algorithms registered | Doc |
|---|---|---|---|
| `drivers/crypto/ccp/` | AMD CCP / PSP (Cryptographic Coprocessor + SEV/SEV-ES/SEV-SNP firmware host interface, TEE, SFS, DBC) | AES, SHA, RSA, AES-GCM, AES-CMAC, AES-XTS, DES3, RNG | `ccp.md` |
| `drivers/crypto/intel/qat/` | Intel QuickAssist (4xxx, 420xx, c3xxx, c62x, dh895xcc + VF) | skcipher (AES-CBC/XTS/CTR), AEAD (AES-GCM/CCM, ChaCha20-Poly1305), ahash (SHA-1/2/3), akcipher (RSA, DH, ECDH), compression (deflate, lz4, zstd) | `qat.md` |
| `drivers/crypto/caam/` | NXP / Freescale CAAM (Cryptographic Acceleration and Assurance Module, i.MX 6/7/8, Layerscape) | skcipher, AEAD, ahash, akcipher (RSA), RNG4, blob-key wrap/unwrap | `caam.md` |
| `drivers/crypto/virtio/` | Paravirtualized virtio-crypto (QEMU/Cloud-Hypervisor backend) | skcipher (AES-CBC/CTR/XTS), AEAD, MAC, hash, akcipher (RSA, ECDSA) | `virtio-crypto.md` |
| `drivers/crypto/chelsio/` | Chelsio Terminator T5/T6 (cxgb4 ULD bound) | AES-CBC/XTS/CTR/CCM/GCM, SHA1/2, IPsec ESP, TLS record offload | `chelsio-crypto.md` |
| `drivers/crypto/cavium/` | Marvell/Cavium Nitrox + ZIP | skcipher, AEAD, ahash, akcipher | future |
| `drivers/crypto/qce/` | Qualcomm Crypto Engine (MSM) | skcipher, ahash, AEAD | future |
| `drivers/crypto/stm32/` | STMicroelectronics STM32 AES/HASH/CRYP | skcipher, ahash | future |
| `drivers/crypto/marvell/` | Marvell OcteonTX2 / CESA | skcipher, AEAD, ahash | future |
| `drivers/crypto/hisilicon/` | HiSilicon SEC / HPRE / TRNG / ZIP | skcipher, AEAD, akcipher | future |
| `drivers/crypto/inside-secure/` | Inside-Secure SafeXcel EIP-197 | skcipher, AEAD, ahash | future |
| `drivers/crypto/allwinner/` | Allwinner sun4i / sun8i SS | skcipher, ahash, RNG | future |
| `drivers/crypto/amlogic/` | Amlogic Meson GXL crypto | skcipher | future |
| `drivers/crypto/atmel-{aes,sha,tdes,ecc,sha204a}.c` | Microchip ATMEL AES / SHA / TDES / ECC | skcipher, ahash, akcipher | future |
| `drivers/crypto/padlock-{aes,sha}.c` | VIA PadLock | skcipher, ahash | future |
| `drivers/crypto/geode-aes.c` | AMD Geode LX | skcipher | future |
| `drivers/crypto/hifn_795x.c` | HIFN 7951 | skcipher, ahash | future |

## Compatibility contract

REQ-1: Each hardware driver registers algorithms via `crypto_register_skciphers`, `crypto_register_aeads`, `crypto_register_ahashes`, `crypto_register_akciphers`, `crypto_register_rng` (cross-ref `.design/crypto/algapi.md`). Registration order is observed: probe + HW-readiness check + register on success, unregister + free on remove.

REQ-2: Per-algorithm priority field MUST be set greater than software fallback (typical: 100=software generic, 300-500=hardware offload, 800-1000=heavily-optimized HW). Crypto API resolves consumer requests by highest priority.

REQ-3: Each request enters the driver via the registered `setkey`, `encrypt`, `decrypt`, `init/update/final` (hash), `sign/verify` (akcipher), `generate/seed` (rng) callbacks. The driver MAY return `-EINPROGRESS` for async HW, completing later via `crypto_request_complete(req, err)`.

REQ-4: HW driver MUST handle CPU-side scatterlists; either DMA-map per-request, use coherent DMA pools, or copy into bounce buffers. Per-request DMA addresses freed at completion.

REQ-5: HW driver MUST observe the crypto-API `CRYPTO_TFM_REQ_MAY_SLEEP` / `CRYPTO_TFM_REQ_MAY_BACKLOG` flags — backlog handling via `crypto_enqueue_request` + per-driver worker thread or hardware queue.

REQ-6: HW driver MUST honour FIPS mode: when `fips_enabled` is set at boot, algorithms not approved (e.g., legacy DES, MD4, MD5, weak ARC4) MUST NOT be registered (or MUST be flagged `CRYPTO_ALG_NEED_FALLBACK` and pass through to a FIPS-approved fallback).

REQ-7: HW driver registering an RNG (CCP, CAAM, QAT, virtio-crypto, hisilicon TRNG) MUST register via `hwrng_register` for entropy contribution (cross-ref `drivers/char/random.md`) and/or `crypto_register_rng` for DRBG-class RNGs.

REQ-8: Module load + algorithm-priority-bump MUST require CAP_SYS_MODULE; runtime debugfs/sysfs configuration knobs MUST require CAP_SYS_ADMIN.

REQ-9: Driver-class shutdown ordering: on `rmmod` or device-unplug, every in-flight request MUST be drained or aborted, algorithms unregistered, and DMA buffers freed before the device disappears.

REQ-10: Per-driver firmware blobs (CCP/PSP firmware, QAT MMP/AE firmware, CAAM RTA descriptors) loaded via `request_firmware()` MUST validate the firmware signature when CONFIG_FW_LOADER_SIG=y.

REQ-11: Hardware key material — wrapped keys, key-handles, key slots — MUST be zeroed on free. Cleartext key buffers MUST live in `kzalloc(GFP_KERNEL | __GFP_ZERO)` slabs and be wiped via `memzero_explicit` after the last use.

REQ-12: SR-IOV-capable accelerators (QAT 4xxx/420xx, CCP-via-PSP, CAAM job-rings) MUST gate VF passthrough to userspace via vfio-pci + iommufd (cross-ref `.design/drivers/iommu/iommufd-device.md`).

## Acceptance Criteria

- [ ] AC-1: `cat /proc/crypto` lists hardware-registered algorithms with `driver: <hw-driver>-<algo>` and `priority` greater than software fallback.
- [ ] AC-2: `cryptsetup benchmark` shows AES-XTS throughput improvement vs software-only when CCP/QAT/CAAM driver loaded.
- [ ] AC-3: `tcrypt` mode tests pass for each algorithm registered by each loaded driver.
- [ ] AC-4: `dmesg | grep -i 'crypto\|sev\|psp\|qat\|caam'` shows correct probe, firmware load, and algorithm register sequence.
- [ ] AC-5: `rngtest -c 1000 < /dev/hwrng` passes statistical health for each driver registering an RNG.
- [ ] AC-6: FIPS gate test: with `fips=1` cmdline, weak algorithms (MD5 software AND any hardware MD5) MUST NOT appear in `/proc/crypto`.
- [ ] AC-7: SR-IOV passthrough test (QAT 4xxx, CCP-PSP): VF attaches to vfio-pci, guest sees device, crypto operations succeed end-to-end.
- [ ] AC-8: Stress under `stress-ng --crypt 64 --timeout 300s`: no oops, no leaks reported by `kmemleak`, no DMA-mapping leaks reported by `dma-debug`.
- [ ] AC-9: `rmmod` of each crypto driver while operations in flight completes cleanly with no UAF on the request path.

## Architecture

The driver class shares a five-stage lifecycle, instantiated per silicon family:

```
struct CryptoOffload {
  device: Arc<HwDevice>,            // PCI / platform / OF device handle
  algs: AlgRegistry,                // ALG / aead / ahash / akcipher / rng vectors
  queues: Vec<HwQueue>,             // per-engine request queue (one per HW ring)
  backlog: BacklogQueue,            // CRYPTO_TFM_REQ_MAY_BACKLOG path
  fw: Option<Firmware>,             // request_firmware blob (PSP, QAT MMP, CAAM RTA)
  rng: Option<HwRng>,               // hwrng_register entry (TRNG / DRBG)
  fips_ok: bool,                    // computed at probe from boot fips_enabled
  cap_mask: CapMask,                // CAP_SYS_ADMIN-gated knobs
}
```

Stage 1: **probe** — bus-match callback (PCI/of/platform) instantiates `HwDevice`, reads silicon ID + revision, maps MMIO via `pci_iomap`/`devm_ioremap_resource`, alloc DMA pools.

Stage 2: **firmware load + HW init** — `request_firmware("xxx.bin")`, validate signature (signed FW only when `CONFIG_FW_LOADER_SIG=y`), DMA to HW load region, kick HW reset + boot, poll status. CCP: PSP firmware via SEV mailbox. QAT: MMP + AE firmware via UCLO loader. CAAM: RTIC + RNG4 instantiation. virtio-crypto: VIRTIO_CONFIG_S_DRIVER_OK.

Stage 3: **algorithm registration** — populate `struct skcipher_alg[] / aead_alg[] / ahash_alg[] / akcipher_alg[] / rng_alg[]` with priority + cra_flags + setkey + en/de + walks; `crypto_register_*` returns the `cra_list` insertion.

Stage 4: **request handling** — per-request `setkey` validates key length and stages key into HW (or slab); `encrypt`/`decrypt` allocates a per-request HW descriptor, scatter-walks the SG list, builds the descriptor, submits to a HW ring, returns `-EINPROGRESS`. IRQ / poll completes the request and calls `crypto_request_complete`.

Stage 5: **remove/unbind** — `crypto_unregister_*` releases the algorithm table; HW queue drained; firmware-loaded regions freed; per-device state freed.

Inter-driver patterns:

- **Fallback discipline** — drivers that cannot handle every input shape (small IV, unaligned SG, very-large key) declare `CRYPTO_ALG_NEED_FALLBACK` and at `cra_init` allocate a tfm of the same name with `CRYPTO_ALG_NEED_FALLBACK` cleared; the driver routes unsupported inputs to the fallback via `crypto_skcipher_encrypt(fallback_req)`. Required so that an offload can advertise an algorithm without claiming complete coverage.
- **Backlog** — `crypto_enqueue_request` + per-driver kthread or workqueue drains backlog when HW ring free.
- **RNG plumbing** — hardware RNGs register with `hwrng_register` (entropy contribution to `random.c`) AND optionally with `crypto_register_rng` (CTR-DRBG / Hash-DRBG implementations). Two-path topology lets the crypto API treat the RNG as a primitive while the entropy pool consumes it as a high-quality source.
- **FIPS gate** — at probe, each driver checks `fips_enabled` and conditionally suppresses non-FIPS-approved algs (`crypto_register_skciphers` table mutated or skipped).
- **debugfs/sysfs surface** — every driver exposes a debugfs root under `/sys/kernel/debug/<driver>/` for queue depth + IRQ counts + telemetry; sysfs nodes under `/sys/class/crypto/`.

## Hardening

(Inherits row-1 features from `drivers/00-overview.md` § Hardening.)

drivers/crypto class-specific reinforcement:

- **Firmware signature validation enforced** — drivers that load HW firmware honour `CONFIG_FW_LOADER_SIG_FORCE=y`; unsigned firmware refused.
- **Key buffers `memzero_explicit` on free** — keys never reach kfree/free_page without explicit zeroization.
- **Per-request DMA mapping released on every completion path** — including the error path; dma-debug tracks leaks.
- **Per-algorithm fallback path enforced for unsupported shapes** — offload never silently corrupts; fallback engaged or `-EINVAL` returned.
- **HW ring index update via WRITE_ONCE + IRQ-safe head/tail pointers** — defense against concurrent submit/complete races corrupting ring metadata.
- **Reset-on-error path** — accelerator reset path (FW timeout, parity error, ring stall) tears down algorithms, resets HW, and re-registers cleanly; no half-state.
- **CONFIG_CRYPTO_USER_API gated** — userland alg lookup via AF_ALG (cross-ref `.design/crypto/af-alg.md`) does not let user enumerate or pin driver-internal context.
- **RNG health check gates entropy contribution** — TRNG/DRBG output passes continuous statistical test before mixing into `/dev/random` pool.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for per-tfm context (`tfm->__crt_ctx`), per-request descriptors, key buffers, scatterlist arrays, and FW load buffers; AF_ALG copy_to/from_user paths bounded by `crypto_skcipher_blocksize`.
- **PAX_KERNEXEC** — every crypto driver's vtables (`skcipher_alg`, `aead_alg`, `ahash_alg`, `akcipher_alg`, `rng_alg`, `cra_init`/`cra_exit`) live in `__ro_after_init` text; W^X enforced on offload driver text segments.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `crypto_skcipher_encrypt`, `crypto_aead_decrypt`, `crypto_ahash_update`, akcipher sign/verify entry points.
- **PAX_REFCOUNT** — saturating `refcount_t` on every per-driver device handle, per-engine queue, per-session handle, and per-tfm context; overflow trap defeats register/unregister race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for tfm context slabs, per-request descriptor slabs, FW staging regions, key handle tables, and DMA pool buffers; key material never bleeds across slab reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every AF_ALG / setkey / ioctl entry into a crypto driver; no user-pointer deref outside canonical `copy_from_user`.
- **PAX_RAP / kCFI** — all `crypto_alg` vtable dispatch (`setkey`, `encrypt`, `decrypt`, `init`/`update`/`final`, `sign`/`verify`, `generate`/`seed`) kCFI-typed; indirect-call validation refuses cross-type jumps.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of HW driver internals (PSP mailbox base, QAT BAR, CAAM RTA pointer) behind CAP_SYSLOG; `%p` suppressed in driver tracepoints.
- **GRKERNSEC_DMESG** — restrict crypto-driver fault, FW-load-fail, and HW-error banners to CAP_SYSLOG so attackers cannot probe HW state via dmesg.
- **CAP_SYS_MODULE** — strict gate on module load for every crypto driver (cross-ref `kernel/module/00-overview.md`); refuse unsigned modules when CONFIG_MODULE_SIG_FORCE=y.
- **CAP_SYS_ADMIN** strict on debugfs / sysfs / ioctl control surfaces for every crypto driver; per-driver `/sys/kernel/debug/<driver>/` requires CAP_SYS_ADMIN open.
- **Firmware signature required** — every `request_firmware` consumer in this class validates the FW image cryptographically; refuse unsigned/expired FW.
- **FIPS lockstep** — with `fips=1`, the driver registration table refuses non-FIPS-approved algs at probe; no runtime promotion.
- **Fallback discipline** — drivers MUST honour `CRYPTO_ALG_NEED_FALLBACK` and route unsupported inputs to a non-offload sibling rather than silently failing or corrupting.
- **RNG output gated** — TRNG / DRBG continuous-health-test failure shuts down the RNG immediately and propagates an error to `hwrng_register`'s consumer (`random.c`).

Rationale: hardware crypto offload is one of the highest-value targets in the kernel — a single sidechannel, key-leak, or DMA bounce out of an HW driver compromises every disk-encryption, TLS termination, and VPN it serves. RAP/kCFI on the crypto_alg vtable surface, FW signature enforcement, FIPS lockstep, fallback discipline, refcount-overflow trap, and explicit key zeroization turn the offload class from "trusted because it's signed by the vendor" into a structural boundary the rest of the kernel can rely on.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Software-only crypto algorithms (covered in `.design/crypto/` Tier-3 subtree)
- AF_ALG userspace interface (covered in `.design/crypto/af-alg.md`)
- The crypto-API algorithm registration framework (covered in `.design/crypto/algapi.md`)
- SoC-specific TRNG drivers under `drivers/char/hw_random/` (covered in `drivers/char/00-overview.md`)
- Per-cipher software implementations (e.g., `arch/x86/crypto/aesni-intel_*`)
- Per-driver Tier-4 implementation breakdown (deferred until v0 port)
- 32-bit-only paths
- Implementation code
