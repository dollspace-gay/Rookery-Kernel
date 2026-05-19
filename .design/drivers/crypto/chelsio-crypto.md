# Tier-3: drivers/crypto/chelsio/ — Chelsio Terminator T5/T6 crypto offload via cxgb4 ULD

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/crypto/00-overview.md
upstream-paths:
  - drivers/crypto/chelsio/chcr_core.c
  - drivers/crypto/chelsio/chcr_core.h
  - drivers/crypto/chelsio/chcr_algo.c
  - drivers/crypto/chelsio/chcr_algo.h
  - drivers/crypto/chelsio/chcr_crypto.h
  - drivers/net/ethernet/chelsio/cxgb4/cxgb4_uld.h
  - include/linux/skbuff.h
-->

## Summary

`drivers/crypto/chelsio/` is the crypto-offload personality of Chelsio Terminator T5/T6 NICs (cxgb4 family). It does NOT bind to a separate PCI ID — instead, it registers as a **ULD** (Upper-Layer Driver) on top of the cxgb4 networking driver, sharing the same physical device, ring/queue infrastructure, and firmware. The crypto subsystem exposes the NIC's on-die crypto block via the kernel crypto API while preserving the NIC's normal packet-processing role.

Supported algorithms:
- **skcipher**: AES-CBC, AES-CTR, AES-XTS, AES-RFC3686-CTR
- **AEAD**: AES-GCM, AES-CCM, RFC4106 AES-GCM-ESP, RFC4309 AES-CCM-ESP, AUTHENC(HMAC-SHA*, AES-CBC), AUTHENC(HMAC-SHA*, AES-CTR), AUTHENC(NULL, ...) IPsec variants
- **ahash**: SHA-1, SHA-224, SHA-256, SHA-384, SHA-512 + HMAC variants

Beyond the crypto-API interface, the same hardware backs:
- **IPsec inline offload** (handled in `drivers/net/ethernet/chelsio/cxgb4/cxgb4_ipsec.c`)
- **TLS record offload (kTLS-Tx + kTLS-Rx)** (handled in `drivers/net/ethernet/chelsio/cxgb4/cxgb4_tls.c`)

This Tier-3 doc focuses on the crypto-API surface — the ULD registration with cxgb4, the per-request Work-Request (WR) encoding, the queue indexing model, and the response handler path. The IPsec / TLS paths are noted as out-of-scope cross-references.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct chcr_driver_data` | global driver state — per-cxgb4-device list + selection pointer | `drivers::chcr::DriverData` |
| `struct uld_ctx` | per-cxgb4-device ULD context (1 per Chelsio PF) | `drivers::chcr::UldCtx` |
| `struct chcr_dev` | per-ULD-context crypto-device state | `drivers::chcr::ChcrDev` |
| `struct chcr_context` | per-tfm context (skcipher/aead/ahash) | `drivers::chcr::ChcrContext` |
| `struct chcr_aead_ctx` / `chcr_gcm_ctx` / `chcr_authenc_ctx` / `ablk_ctx` / `hmac_ctx` | per-tfm per-class context | `drivers::chcr::AeadCtx` / `GcmCtx` / `AuthencCtx` / `AblkCtx` / `HmacCtx` |
| `struct chcr_wr` | on-wire Work-Request descriptor (fw_crypto_lookaside_wr + sec_cpl + key_ctx) | `drivers::chcr::Wr` |
| `chcr_uld_add(lld)` | ULD-add callback from cxgb4: instantiate per-device chcr_dev | `Subsystem::uld_add` |
| `chcr_uld_state_change(handle, state)` | cxgb4 state-change callback (UP / DOWN / DETACH) | `UldCtx::state_change` |
| `chcr_uld_rx_handler(handle, rsp, pkt)` | rx response handler from cxgb4 | `UldCtx::rx_handler` |
| `chcr_send_wr(skb)` | submit Work-Request skb to a cxgb4 control queue | `Subsystem::send_wr` |
| `chcr_dev_add(u_ctx)` / `chcr_dev_init(u_ctx)` / `chcr_dev_move(u_ctx)` | per-device state machine | `UldCtx::dev_add` / `_dev_init` / `_dev_move` |
| `chcr_detach_device(u_ctx)` | per-device detach: wait for outstanding WRs, unregister algorithms | `UldCtx::detach_device` |
| `chcr_inc_wrcount(dev)` / `chcr_dec_wrcount(dev)` | per-device outstanding-WR refcount | `ChcrDev::inc_wrcount` / `_dec_wrcount` |
| `assign_chcr_device()` | round-robin pick of an active Chelsio device | `Subsystem::assign_chcr_device` |
| `chcr_register_alg()` / `chcr_unregister_alg()` | per-template registration loop | `Subsystem::register_alg` / `_unregister_alg` |
| `chcr_aes_encrypt` / `_decrypt` / `chcr_ahash_*` / `chcr_aead_encrypt` / `_decrypt` | per-class request entry | `*Session::encrypt` / `_decrypt` / `_update` / `_final` |
| `create_cipher_wr(wrparam)` / `create_aead_wr(req, ...)` / `create_hash_wr(req, ...)` | per-request WR builder | `Wr::build_cipher` / `_build_aead` / `_build_hash` |
| `chcr_cipher_fallback(cipher, req)` / `chcr_aead_fallback(req, ...)` | software fallback path | `*Session::fallback` |

## Compatibility contract

REQ-1: Driver registers with cxgb4 via `cxgb4_register_uld(CXGB4_ULD_CRYPTO, &chcr_uld_info)`; cxgb4 calls back with `add` per discovered Chelsio device, providing an `lld_info` describing per-device queues + capabilities.

REQ-2: Per-cxgb4-device, `chcr_uld_add(lld)` allocates a `uld_ctx`, registers in the global `chcr_driver_data` list, and instantiates a `chcr_dev` with state `CHCR_INIT`. State advances to `CHCR_ATTACH` on first `state_change(UP)`.

REQ-3: Algorithm registration deferred until at least one device is `CHCR_ATTACH`; `chcr_register_alg` runs the per-template registration loop iterating `driver_algs[]` (~30 entries covering skcipher + aead + ahash variants).

REQ-4: Per-request Work-Request format: `fw_crypto_lookaside_wr` (firmware lookaside header) + `ulp_txpkt` + `ulptx_idata` + `_fw_ofld_resp_wr` + `cpl_tx_sec_pdu` (Security CPL command) + cipher key context + IV + scatterlist/double-scatterlist (ULPTX / DSGL).

REQ-5: Submission via cxgb4 control queue: build skb containing the WR + place inline data; `chcr_send_wr(skb)` enqueues into cxgb4's offload queue index (`qid` returned by `get_qidxs`); cxgb4 DMAs to NIC.

REQ-6: Response via cxgb4 rx path: cxgb4 demuxes by opcode (`CPL_FW6_PLD`) and invokes `chcr_uld_rx_handler`; rx handler runs `work_handlers[CPL_FW6_PLD]` → `cpl_fw6_pld_handler` → completion matched against pending request by per-request cookie.

REQ-7: Per-tfm context carries cra_flags `CRYPTO_ALG_NEED_FALLBACK`; on `cra_init` allocates a sibling sw tfm; oversize / unaligned / unsupported inputs routed to fallback via `chcr_cipher_fallback`.

REQ-8: Per-device outstanding-WR refcount via `chcr_inc_wrcount`/`_dec_wrcount`; detach (`chcr_detach_device`) blocks on `wait_event` for the refcount to hit zero before tearing down.

REQ-9: Per-CPU queue selection: `get_qidxs(req, &txqidx, &rxqidx)` returns a per-CPU-affined Tx/Rx queue pair for the request; affines to current CPU's offload queue index modulo per-device queue count.

REQ-10: Multi-device load balancing: `assign_chcr_device` returns the next `uld_ctx` in `last_dev` rotation; per-device state must be `CHCR_ATTACH` to be eligible.

REQ-11: Per-device PM: cxgb4 owns power and the chcr ULD does not implement separate suspend/resume — it follows the cxgb4 state-machine via `chcr_uld_state_change`.

REQ-12: Per-WR FIPS compliance: with `fips=1`, non-FIPS-approved variants (e.g., RFC3686 with non-approved IV form) absent from registration table.

## Acceptance Criteria

- [ ] AC-1: Chelsio T5/T6 hosting `modprobe cxgb4 chcr` shows `dmesg | grep -i 'chcr\|chelsio'` confirming ULD registration and per-device attach.
- [ ] AC-2: `cat /proc/crypto | grep chcr` lists registered algorithms with priority > software fallback.
- [ ] AC-3: `tcrypt mode=N` KAT vectors pass for every Chelsio-registered algorithm.
- [ ] AC-4: `cryptsetup benchmark` shows Chelsio AES-XTS speedup vs software.
- [ ] AC-5: NIC-still-functional test: while crypto offload running under stress, normal `iperf3` over the same Chelsio NIC achieves expected throughput (no functional regression).
- [ ] AC-6: Detach test: with crypto operations in flight, hot-unbind cxgb4 device; observed: chcr blocks on `chcr_detach_device` until pending WRs drain, no oops, algorithms unregistered cleanly.
- [ ] AC-7: Multi-device load balance: with two Chelsio cards plugged in, AF_ALG stress shows traffic balanced across both via `assign_chcr_device`.
- [ ] AC-8: Fallback test: unsupported input shape (small IV, oversize key) routes through `chcr_cipher_fallback`; ciphertext correct.
- [ ] AC-9: Stress under `stress-ng --crypt 32 --timeout 300s`: no DMA-debug, no kmemleak, no soft-lockup; `chcr_wrcount` returns to 0 after stress completes.

## Architecture

`UldCtx` is the per-cxgb4-device container:

```
struct UldCtx {
  entry: ListEntry,                // chcr_driver_data.act_dev / inact_dev
  lldi: Arc<Cxgb4LldInfo>,         // cxgb4 LLD info (queues, ports, BAR)
  dev: ChcrDev,
}

struct ChcrDev {
  state: ChcrState,                // CHCR_INIT / CHCR_ATTACH / CHCR_DETACH
  refcount: Refcount,              // outstanding-WR count
  wrcount_lock: SpinLock,
  detach_in_progress: AtomicBool,
  detach_work: Work,
  fallback_count: AtomicU64,
  errs: AtomicU64,                 // per-class error counters
}

struct ChcrContext {
  dev: Arc<ChcrDev>,
  txqidx: u16,
  rxqidx: u16,
  ntxq: u16,
  nrxq: u16,
  crypto_ctx: KBox<union {
    AblkCtx ablkctx,
    AeadCtx aeadctx,
    HmacCtx hmacctx,
  }>,
}
```

ULD attach (`chcr_uld_add`):
1. Allocate `uld_ctx`; copy `lld_info` (per-cxgb4-device queues, BAR pointers, port mask).
2. Insert into `chcr_driver_data.inact_dev` list.
3. `chcr_dev_init(u_ctx)` — initialize ChcrDev state machine in CHCR_INIT.
4. Return handle to cxgb4.

ULD state change (`chcr_uld_state_change`):
1. On `CXGB4_STATE_UP`:
   - `chcr_dev_add(u_ctx)` — move from inact_dev to act_dev list.
   - If first attached device: `chcr_register_alg()` registers algorithms with crypto API.
   - state → CHCR_ATTACH.
2. On `CXGB4_STATE_DETACH` / `CXGB4_STATE_DOWN`:
   - `chcr_detach_device(u_ctx)` — set detach flag, wake any backlogged WRs, block on refcount → 0.
   - If last attached device: `chcr_unregister_alg()` unregisters from crypto API.

Per-request encrypt flow (`chcr_aes_encrypt`):
1. `c_ctx(tfm)` retrieves `ChcrContext` from tfm.
2. `process_cipher(req, qid, fallback_skb, op_type)` — validate inputs, route oversize/unaligned to fallback.
3. `create_cipher_wr(&wrparam)` — build per-request WR:
   - `chcr_wr` header.
   - `sec_cpl` Security CPL command (algorithm, key length, IV mode).
   - `_key_ctx` referencing per-tfm key schedule.
   - IV inline.
   - ULPTX walk over source SG; DSGL walk over destination SG.
4. `chcr_inc_wrcount(dev)`.
5. `chcr_send_wr(skb)` — enqueue into cxgb4 offload queue at `txqidx`.
6. Return `-EINPROGRESS`.

Per-response flow (`chcr_uld_rx_handler` → `cpl_fw6_pld_handler`):
1. cxgb4 demuxes by `opcode` field of rx CPL → invokes `chcr_uld_rx_handler`.
2. Opcode `CPL_FW6_PLD`: handler matches WR cookie to pending request via per-context request pointer.
3. Per-class response handler (`chcr_handle_cipher_resp`, `chcr_handle_aead_resp`, `chcr_handle_ahash_resp`) runs.
4. dma-unmap (if any), update IV (chained mode), `chcr_dec_wrcount(dev)`.
5. `crypto_request_complete(req, err)`.

Round-robin device pick (`assign_chcr_device`):
1. Walk `chcr_driver_data.act_dev` from `last_dev`'s next entry.
2. Find first device with state CHCR_ATTACH AND outstanding queue not full (`cxgb4_is_crypto_q_full`).
3. Update `last_dev`; return.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `wr_sg_no_oob` | OOB | per-WR ULPTX + DSGL walks bounded by `MAX_NUM_SG` (cxgb4 firmware limit); refuse oversize SGLs. |
| `key_schedule_no_oob` | OOB | `_key_ctx` size bounded by per-algorithm key-context-size constants. |
| `wrcount_no_uaf` | UAF | every `inc_wrcount` paired with `dec_wrcount`; detach blocks until count → 0. |
| `qid_valid` | BOUNDS | per-request `txqidx`/`rxqidx` strictly bounded by `ntxq`/`nrxq` from cxgb4. |
| `state_machine` | STATE | ChcrDev state transitions strict: INIT → ATTACH → DETACH; no skip. |

### Layer 2: TLA+

`models/chelsio_crypto/uld.tla` (parent-declared): proves ULD attach/detach handshake with cxgb4 — concurrent ULD detach + WR submit serializes via wrcount; no WR submitted after detach signal observed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post-`chcr_dev_add`: at least one device in `act_dev` AND algorithms registered with crypto API | `UldCtx::dev_add` |
| Per-`chcr_send_wr`: WR cookie unique within outstanding window; matched exactly once by response handler | `Subsystem::send_wr` |
| Per-`chcr_detach_device` post: outstanding WR count == 0 AND any further `chcr_send_wr` returns -ENODEV | `UldCtx::detach_device` |
| Per-`chcr_aes_encrypt`: source plaintext+IV unmodified after EINPROGRESS; destination filled exactly once at response | `*Session::encrypt` |

### Layer 4: Verus/Creusot functional

End-to-end: `cryptsetup luksOpen` over a chcr-offloaded AES-XTS volume → guest crypto API picks chcr-aes-xts → per-block WR built → cxgb4 control queue → NIC firmware → response → per-block ciphertext bit-exact against software KAT.

## Hardening

(Inherits row-1 features from `drivers/crypto/00-overview.md` § Hardening.)

Chelsio-specific reinforcement:

- **MMIO bounded to cxgb4-owned BAR** — chcr never directly accesses Chelsio MMIO; all hardware reach goes through cxgb4 LLD callbacks.
- **Per-WR cookie verified at response time** — defense against malformed CPL or replay.
- **WR-size clamp** — per-WR total size bounded against cxgb4 control-queue descriptor size; refuse oversize WRs.
- **Per-tfm fallback enforced** — `CRYPTO_ALG_NEED_FALLBACK` set on every Chelsio algorithm; software sibling allocated at `cra_init`.
- **Detach interlock** — `chcr_detach_device` sets `detach_in_progress` THEN waits for `wrcount → 0`; new requests refused immediately on detach signal.
- **Per-CPU queue selection** — `get_qidxs` clamps qid modulo per-device queue count; defense against per-CPU mismatch causing out-of-range queue write.
- **Fallback observed via counter** — `dev->fallback_count` increments; userland may monitor; sudden spike signals HW unhappiness.
- **AER recovery** — cxgb4 owns PCIe error recovery; chcr observes via `state_change(DETACH/UP)` and re-syncs algorithm registration accordingly.
- **Per-class error counters** — bumped on every response with non-success status; CAP_SYSLOG-gated dmesg banner for sustained error rate.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `uld_ctx`, `chcr_context`, per-class `chcr_aead_ctx`/`ablk_ctx`/`hmac_ctx`/`chcr_gcm_ctx`/`chcr_authenc_ctx`, per-request reqctx, and Work-Request skb headroom; ULPTX/DSGL walk lengths clamped by `MAX_NUM_SG`.
- **PAX_KERNEXEC** — `chcr_uld_info` (ULD callbacks), `driver_algs[]` template vtables, `work_handlers[]` per-opcode dispatch table, and per-class request entry points live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `chcr_aes_encrypt`/`_decrypt`, `chcr_aead_encrypt`/`_decrypt`, `chcr_ahash_*`, `chcr_send_wr`, and `chcr_uld_rx_handler`.
- **PAX_REFCOUNT** — saturating `refcount_t` on `chcr_dev` outstanding-WR count, per-tfm `chcr_context` references, and per-`uld_ctx` lifetime; overflow trap defeats detach-during-stress races and double-completion UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-tfm key-context slabs (which contain expanded AES key schedule + HMAC inner/outer keys), per-request reqctx, Work-Request skb headroom, IV scratch buffers, and per-class context unions; key material never bleeds across slab reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every AF_ALG entry reaching chcr; no kernel-side user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `chcr_alg_template` per-class vtables (`skcipher_alg`, `aead_alg`, `ahash_alg`), `work_handlers[CPL_*]` dispatch, `chcr_uld_info` ULD callbacks, and per-WR build dispatchers (`create_cipher_wr`/`create_aead_wr`/`create_hash_wr`) kCFI-typed.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of cxgb4 LLD info pointer, per-WR cookie addresses, and per-device offload queue base behind CAP_SYSLOG; tracepoint `%p` suppressed.
- **GRKERNSEC_DMESG** — restrict chcr WR-failure, response-status-error, detach-stuck, fallback-spike banners to CAP_SYSLOG so attackers cannot probe NIC offload state via dmesg.
- **CAP_SYS_ADMIN strict on fallback-rate / queue-depth knobs** — debugfs writes to fallback counters or queue thresholds require CAP_SYS_ADMIN in init userns.
- **WR descriptor PAX_USERCOPY** — per-WR `chcr_wr` + key-context + ULPTX/DSGL strictly bounded; refuse SGL with `MAX_NUM_SG` exceeded; defense against oversize-SG corrupting cxgb4 control queue.
- **MMIO accesses go through cxgb4 LLD only** — chcr forbidden from direct readl/writel on Chelsio BAR; defense against accidental BAR mismap or driver-class violation.
- **FIPS-test gate** — with `fips=1`, non-FIPS-approved templates (e.g., legacy RFC3686 form, MD5-based HMAC) absent from `driver_algs[]`; refuse runtime promotion.
- **Detach interlock audited** — `chcr_detach_device` MUST observe `wrcount == 0` before unregistering algorithms; any short-circuit refused.
- **Fallback availability invariant** — every chcr alg requires a registered software sibling; refuse alg registration when fallback unavailable.
- **Per-WR cookie unique within outstanding window** — cookie space exhaustion handled by waiting for response; never reused while outstanding.

Rationale: Chelsio crypto is a NIC sharing a single piece of silicon between datapath (with normal packet processing) and crypto offload, which makes it uniquely sensitive to detach/teardown races — a missed wrcount, an oversize ULPTX walk, or a torn per-WR cookie can corrupt the offload queue used by both the crypto path AND the NIC's normal traffic. RAP/kCFI on the per-class vtables and per-opcode dispatch, PAX_USERCOPY on WR descriptors, FIPS-test gating, detach interlock auditing, and explicit `memzero_explicit` on per-tfm key context turn the chcr ULD from "trusted because the NIC firmware is signed" into a structural multi-tenant offload boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- IPsec inline offload over the same hardware (covered in `drivers/net/ethernet/chelsio/cxgb4/cxgb4_ipsec.md` future Tier-3)
- kTLS-Tx / kTLS-Rx record offload over the same hardware (covered in `drivers/net/ethernet/chelsio/cxgb4/cxgb4_tls.md` future Tier-3)
- cxgb4 ULD core (covered in `drivers/net/ethernet/chelsio/cxgb4/00-overview.md` future Tier-3)
- cxgb4 firmware loading + crypto firmware blob (covered in `drivers/net/ethernet/chelsio/cxgb4/00-overview.md` future)
- Per-WR ULPTX / DSGL on-wire format detail (Chelsio T5/T6 hardware spec)
- 32-bit-only paths
- Implementation code
