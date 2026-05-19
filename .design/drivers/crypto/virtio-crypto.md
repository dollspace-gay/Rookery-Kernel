# Tier-3: drivers/crypto/virtio/ — paravirtualized virtio-crypto (AKCipher / skcipher / AEAD / MAC / hash over virtqueues)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/crypto/00-overview.md
upstream-paths:
  - drivers/crypto/virtio/virtio_crypto_core.c
  - drivers/crypto/virtio/virtio_crypto_mgr.c
  - drivers/crypto/virtio/virtio_crypto_common.h
  - drivers/crypto/virtio/virtio_crypto_akcipher_algs.c
  - drivers/crypto/virtio/virtio_crypto_skcipher_algs.c
  - include/uapi/linux/virtio_crypto.h
  - include/uapi/linux/virtio_ids.h
-->

## Summary

`drivers/crypto/virtio/` is the guest-side driver for the **virtio-crypto** paravirtualized cryptography device. The host (QEMU, Cloud-Hypervisor, kvmtool) presents a virtio device exposing crypto offload services; the guest driver registers algorithms with the kernel crypto API that route requests through virtqueues to the host. Backend hosts forward to a real accelerator (QAT, CCP, CAAM, ATA-of-Crypto, software) or terminate in software, transparent to the guest.

The protocol is defined in the virtio-crypto specification (chapter 5.9 of the virtio v1.x spec) and the on-wire structures live in `include/uapi/linux/virtio_crypto.h`. Five service classes are supported:

- **CIPHER** — symmetric block cipher (AES-{ECB,CBC,CTR,XTS}, DES, 3DES, ChaCha20, ...)
- **HASH** — keyless hash (SHA-1/2/3, MD5)
- **MAC** — keyed MAC (HMAC, AES-CMAC, GMAC)
- **AEAD** — authenticated encryption (AES-GCM, AES-CCM, ChaCha20-Poly1305)
- **AKCIPHER** — asymmetric cipher (RSA, ECDSA)

Each service exposes per-session-id state — the guest issues a `CREATE_SESSION` control-vq request, receives a session-id, and uses that session-id for subsequent `OP_ENCRYPT` / `OP_DECRYPT` / `OP_SIGN` / `OP_VERIFY` data-vq requests. `DESTROY_SESSION` releases the session.

This Tier-3 covers the four `.c` files: `_core.c` (probe / virtqueue setup / IRQ workers / suspend-resume), `_mgr.c` (per-device-table manager and lookup), `_akcipher_algs.c` (akcipher class glue), and `_skcipher_algs.c` (skcipher class glue). The hash / MAC / AEAD glue lives under flags-conditional pieces of `_skcipher_algs.c` and the common header.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct virtio_crypto` | per-virtio-crypto-device control block | `drivers::virtio_crypto::VirtioCrypto` |
| `struct data_queue` | per-data-virtqueue context | `drivers::virtio_crypto::DataQueue` |
| `struct virtio_crypto_request` | per-crypto-op in-flight | `drivers::virtio_crypto::Request` |
| `struct virtio_crypto_ctrl_request` | per-control-vq message in-flight | `drivers::virtio_crypto::CtrlRequest` |
| `struct virtio_crypto_op_ctx` | per-session opaque per-op context | `drivers::virtio_crypto::OpCtx` |
| `virtcrypto_probe(vdev)` | virtio device probe callback | `Subsystem::probe` |
| `virtcrypto_remove(vdev)` | virtio device remove callback | `Subsystem::remove` |
| `virtcrypto_init_vqs(vi)` / `virtcrypto_find_vqs(vi)` / `virtcrypto_del_vqs(vi)` | virtqueue lifecycle | `VirtioCrypto::init_vqs` / `_find_vqs` / `_del_vqs` |
| `virtcrypto_alloc_queues(vi)` / `virtcrypto_free_queues(vi)` | per-data-queue allocation | `VirtioCrypto::alloc_queues` / `_free_queues` |
| `virtcrypto_set_affinity(vcrypto)` / `_clean_affinity(vi, hcpu)` | per-CPU NUMA affinity for data queues | `VirtioCrypto::set_affinity` / `_clean_affinity` |
| `virtcrypto_update_status(vcrypto)` | poll device-status during probe | `VirtioCrypto::update_status` |
| `virtcrypto_start_crypto_engines(vcrypto)` / `_clear_crypto_engines(...)` | per-engine crypto-engine init | `VirtioCrypto::start_engines` / `_clear_engines` |
| `virtio_crypto_ctrl_vq_request(vcrypto, sgs[], outs, ins, vc_ctrl_req)` | submit per-session ctrl request | `VirtioCrypto::ctrl_vq_request` |
| `virtio_crypto_ctrlq_callback(vc_ctrl_req)` / `virtcrypto_ctrlq_callback(vq)` | ctrl-vq completion callback | `VirtioCrypto::ctrlq_callback` |
| `virtcrypto_done_work(work)` / `virtcrypto_dataq_callback(vq)` | data-vq completion worker | `DataQueue::done_work` / `_callback` |
| `virtcrypto_devmgr_add_dev(vcrypto)` / `_rm_dev(...)` / `_get_head()` | per-device global manager | `DevMgr::add_dev` / `_rm_dev` / `_get_head` |
| `virtcrypto_get_dev_node(node, service, algo)` | round-robin device lookup by NUMA + service + algo | `DevMgr::get_dev_node` |
| `virtcrypto_dev_get(dev)` / `_dev_put(dev)` | per-device refcount | `DevMgr::dev_get` / `_dev_put` |
| `virtcrypto_dev_start(dev)` / `_dev_stop(dev)` / `_dev_started(...)` | per-device state | `DevMgr::dev_start` / `_dev_stop` / `_dev_started` |
| `virtio_crypto_skcipher_create_session` / `_destroy_session` / `_encrypt` / `_decrypt` | skcipher per-session ops | `SkcipherSession::create` / `_destroy` / `_encrypt` / `_decrypt` |
| `virtio_crypto_rsa_set_priv_key` / `_set_pub_key` / `_encrypt` / `_decrypt` / `_sign` / `_verify` | akcipher (RSA) ops | `RsaSession::set_priv_key` / `_set_pub_key` / `_encrypt` / `_decrypt` / `_sign` / `_verify` |
| `virtio_crypto_alg_validate_key(keylen, alg)` | per-algorithm key-length validator | `Subsystem::validate_key` |

## Compatibility contract

REQ-1: virtio-crypto device class id = 20 (`VIRTIO_ID_CRYPTO`); probe registered via `virtio_driver` against `id_table`.

REQ-2: Per-device feature negotiation: features include `VIRTIO_CRYPTO_F_REVISION_1`, plus service-class supported bits in device-specific config (`virtio_crypto_config.crypto_services`).

REQ-3: Per-device exposes ≥2 virtqueues: 1 control-vq + N data-vqs (one per service or one per CPU per service). Per-CPU data-vq affinity assigned via `virtcrypto_set_affinity`.

REQ-4: Per-device-config reports per-service algorithm masks (`cipher_algo_l/h`, `hash_algo`, `mac_algo_l/h`, `aead_algo`, `akcipher_algo`) — driver only registers algorithms advertised by host.

REQ-5: Per-session lifecycle: `CREATE_SESSION` ctrl-vq submission with key material + algorithm parameters → host returns session-id (64-bit opaque handle) → guest stores in per-tfm context. `DESTROY_SESSION` on `cra_exit` releases host-side state.

REQ-6: Per-data-vq ring submission protocol: per-op header (`virtio_crypto_op_header` with opcode + session-id + flow-id) + service-specific input fields (`virtio_crypto_sym_data_req`, `virtio_crypto_akcipher_data_req`, ...) + source scatter-gather + destination scatter-gather + response status byte.

REQ-7: Asynchronous completion: per-op `crypto_request_complete(req, err)` invoked from `virtcrypto_done_work` workqueue after the data-vq used-ring observes the response status.

REQ-8: NUMA-aware device selection: `virtcrypto_get_dev_node(node, service, algo)` finds the closest device on the requested NUMA node that advertises the requested service+algorithm.

REQ-9: Suspend / resume: `virtcrypto_freeze(vdev)` drains queues + destroys all live sessions; `virtcrypto_restore(vdev)` re-allocates queues + re-registers algorithms (sessions are NOT preserved across S3 — guest userland re-creates as needed).

REQ-10: Hot-config-change: `virtcrypto_config_changed(vdev)` enqueues `vcrypto_config_changed_work` which re-reads device-config and re-applies algorithm registration changes.

REQ-11: Backlog: `crypto_enqueue_request` honored — `CRYPTO_TFM_REQ_MAY_BACKLOG` path queues into per-data-vq engine backlog when ring full.

REQ-12: Per-device strict refcount: `virtcrypto_dev_get`/`_put` reference-counts the device against per-tfm bindings; remove path waits for refcount → 0.

## Acceptance Criteria

- [ ] AC-1: KVM guest with `-device virtio-crypto-pci,cryptodev=<backend>` boots; `dmesg | grep -i 'virtio_crypto\|crypto_engine'` shows probe.
- [ ] AC-2: `cat /proc/crypto | grep virtio` shows guest-side registered algorithms matching host-advertised service classes.
- [ ] AC-3: `tcrypt mode=N` KAT vectors pass for AES-CBC/CTR/XTS, AES-GCM, HMAC-SHA256, RSA via virtio-crypto.
- [ ] AC-4: Throughput test: `cryptsetup benchmark` inside guest shows virtio-crypto AES throughput dependent on host backend.
- [ ] AC-5: Session lifecycle test: 10,000 setkey/encrypt/destroy cycles complete with no session-id leak (host reports session-count → 0 at end).
- [ ] AC-6: Multi-vq scaling test: 8 vCPUs each running stress-ng --crypt; each vCPU drives a different data-vq; observed throughput scales near-linearly.
- [ ] AC-7: Hot-config-change test: host modifies advertised algorithm mask at runtime → guest re-registers (or unregisters) accordingly.
- [ ] AC-8: Freeze/restore (S3) cycle: in-flight ops complete or fail; post-resume, new ops succeed.
- [ ] AC-9: Migration test (live migration of KVM guest with virtio-crypto): in-flight requests fail with `-EAGAIN` and userland retries succeed against destination host.

## Architecture

`VirtioCrypto` is the per-device container:

```
struct VirtioCrypto {
  vdev: Arc<VirtioDevice>,
  ctrl_vq: NonNull<Virtqueue>,
  data_vq: Vec<DataQueue>,         // per-CPU or per-service queues
  ctrl_lock: Mutex,
  ctrl_buf: KBox<CtrlBuffer>,      // staging for ctrl-vq ops
  config: VirtioCryptoConfig,
  max_data_queues: u32,
  curr_queue: AtomicU32,           // round-robin pointer for skcipher
  status: AtomicU32,               // DEV_INITIALIZED / DEV_UP / DEV_DOWN
  flags: u32,                      // FEATURES negotiated bitmap
  affinity_hint_set: bool,
  refcount: Refcount,
  config_work: Work,
  num_engines: u32,
  engines: Vec<CryptoEngine>,
}

struct DataQueue {
  vq: NonNull<Virtqueue>,
  engine: CryptoEngine,
  done_work: Work,
  cb_lock: SpinLock,
  pending: List<Request>,
}

struct Request {
  vc_ctx: Arc<OpCtx>,              // points back to per-tfm session
  req: CryptoRequestHandle,        // skcipher / aead / akcipher async req
  data_vq: Arc<DataQueue>,
  header: VirtioCryptoOpHeader,    // per-op header
  status: u8,                      // filled by host
  sgs_in: SgTable,                 // per-op DMA SGs
  sgs_out: SgTable,
}
```

Probe sequence (`virtcrypto_probe`):
1. `virtcrypto_alloc_queues(vi)` — pre-allocate max_data_queues + 1 ctrl-vq.
2. `virtcrypto_find_vqs(vi)` — request vqs from virtio core; per-data-vq IRQ callback `virtcrypto_dataq_callback`; ctrl-vq IRQ callback `virtcrypto_ctrlq_callback`.
3. Read `virtio_crypto_config` from device-config space.
4. `virtcrypto_start_crypto_engines(vcrypto)` — init per-vq `crypto_engine` for backlog management.
5. `virtcrypto_devmgr_add_dev(vcrypto)` — register in global manager.
6. `virtio_device_ready(vdev)` — signal DRIVER_OK.
7. `virtcrypto_set_affinity(vcrypto)` — per-data-vq affinity to a CPU.
8. `virtcrypto_update_status(vcrypto)` — poll device-status, register per-service algorithms (skcipher, aead, akcipher, hash, mac) honoring config-advertised mask.

Per-tfm setkey flow (e.g., `virtio_crypto_skcipher_setkey`):
1. Allocate `OpCtx` with per-tfm state.
2. Validate keylen via `virtio_crypto_alg_validate_key`.
3. Build `virtio_crypto_sym_create_session_req` ctrl message + key data.
4. `virtio_crypto_ctrl_vq_request(...)` submits + waits on completion.
5. On success, host returns session-id; store in `OpCtx.session_id`.

Per-request encrypt flow (`virtio_crypto_skcipher_encrypt`):
1. Allocate `Request`; populate per-op header with `OPCODE(CIPHER, OP_ENCRYPT)` + session-id.
2. DMA-map source + destination SGLs.
3. Build `virtio_crypto_sym_data_req` with IV + cipher params.
4. `virtqueue_add_sgs(data_vq, sgs, outs, ins, req, GFP_ATOMIC)` + `virtqueue_kick(data_vq)`.
5. Return `-EINPROGRESS`.

Completion flow (`virtcrypto_dataq_callback` → `virtcrypto_done_work`):
1. virtqueue used-ring entry observed.
2. Per-`Request`: read status byte; translate to errno.
3. dma-unmap SGLs.
4. `crypto_request_complete(req->req, err)`.

Ctrl-vq protocol:
1. Caller acquires `ctrl_lock`.
2. Builds `CtrlRequest` with output buffer (request payload) + input buffer (response status + per-op response).
3. `virtqueue_add_sgs` + kick + wait on completion (`vc_ctrl_req->compl`).
4. On callback (`virtio_crypto_ctrlq_callback`): consumer wakes; reads status.

Session destroy flow (`virtio_crypto_skcipher_exit_tfm`):
1. Build `virtio_crypto_sym_destroy_session_req` with session-id.
2. ctrl-vq submit; wait for ack.
3. Zero per-tfm key buffer via `memzero_explicit`.
4. `virtcrypto_dev_put(vcrypto)`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `vq_sg_no_oob` | OOB | virtqueue `add_sgs(out, in)` element counts bounded by per-vq descriptor table size. |
| `session_id_unique` | UNIQUENESS | per-device session-id allocator monotonic; never re-issues active session. |
| `ctrl_buf_no_oob` | OOB | ctrl-vq input/output bounded by `sizeof(union ctrl_request_data)`. |
| `dev_refcount_no_uaf` | UAF | every `dev_get` paired with `dev_put`; remove path waits for count → 0. |
| `affinity_no_oob` | OOB | per-CPU affinity index bounded by `nr_cpu_ids`. |

### Layer 2: TLA+

`models/virtio_crypto/session.tla` (parent-declared): proves CREATE_SESSION → DATA_OP → DESTROY_SESSION lifecycle linearizes correctly under concurrent guest threads sharing one tfm; host-side session table observes no leaked ids; refused-session DATA_OP returns `VIRTIO_CRYPTO_INVSESS`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post-`virtio_crypto_skcipher_setkey`: session-id valid AND `cra_init` increments device refcount | `SkcipherSession::setkey` |
| Per-`virtio_crypto_skcipher_encrypt`: source plaintext + IV unmodified after request return-by-EINPROGRESS; destination written exactly once by host | `SkcipherSession::encrypt` |
| Per-session destroy: host-side session table size decreases by 1 | `SkcipherSession::exit_tfm` |
| Refcount invariant: `vcrypto->refcount == count_active_sessions + 1` (1 = base) | `VirtioCrypto::refcount` |

### Layer 4: Verus/Creusot functional

End-to-end: guest userland TLS handshake via AF_ALG `gcm(aes)` → virtio-crypto-aes-gcm registered → CREATE_SESSION → per-packet ENCRYPT → DESTROY_SESSION → bit-exact ciphertext match against software KAT for same key+IV+plaintext+aad inputs.

## Hardening

(Inherits row-1 features from `drivers/crypto/00-overview.md` § Hardening.)

virtio-crypto-specific reinforcement:

- **Host-side untrust** — every host-returned status byte validated against the known status-code table; unknown codes treated as error.
- **Session-id treated as opaque** — guest never interprets bits; refuse session-id 0 (invalid) and per-host-config max.
- **Per-vq descriptor exhaustion** — `virtqueue_add_sgs` failure returns -ENOSPC; backlog handles via crypto-engine; defense against host stall causing guest soft-lockup.
- **Key buffers `memzero_explicit` after CREATE_SESSION** — guest-side key copy zeroed once host has accepted session.
- **No assumption of in-order completion** — used-ring entries dispatched via opaque `req` pointer match; per-request state tracked independently.
- **Ctrl-vq serialization via `ctrl_lock`** — concurrent CREATE/DESTROY/QUERY sequenced; no torn ctrl-vq state.
- **Per-affinity-cleanup on offline CPU** — `virtcrypto_clean_affinity` handles CPU hotplug; per-vq affinity hints refreshed.
- **Config-change re-validation** — host-driven config change re-validates advertised algorithms; algorithms removed from advertised set are unregistered.
- **Suspend drains in-flight** — `virtcrypto_freeze` waits for pending data-vq ops before declaring frozen.
- **Feature-bit minimum check** — refuse devices not advertising required REVISION_1.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `virtio_crypto`, per-`data_queue`, per-`Request`, per-`OpCtx`, ctrl-vq staging, and per-session key buffers; per-op `virtio_crypto_op_header` + service-specific data structs validated and bounded; scatterlist desc-table writes (`virtqueue_add_sgs`) clamp out-elements + in-elements against virtqueue descriptor count.
- **PAX_KERNEXEC** — `virtio_crypto_driver` ops, per-class algorithm vtables (`virtio_crypto_skcipher_algs`, `virtio_crypto_akcipher_algs`), data-vq callback dispatchers, and pm-ops dispatch live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `virtio_crypto_skcipher_encrypt`, `_setkey`, `_exit_tfm`, akcipher sign/verify, ctrl-vq submit paths, and `virtcrypto_done_work` workqueue entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `virtio_crypto` device, per-`OpCtx`, per-`Request`, per-`CtrlRequest`, and per-session-id consumer counts; overflow trap defeats CREATE/DESTROY race UAFs (a real risk when host CSP misbehaves).
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-session key slabs, per-request payload buffers, ctrl-vq staging, and per-`OpCtx` private state; key material never bleeds across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on every AF_ALG entry reaching virtio-crypto; no kernel-side user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `virtio_crypto_skcipher_alg`, `virtio_crypto_aead_alg`, `virtio_crypto_akcipher_alg`, `virtio_driver` ops, virtqueue callback function pointers, and `crypto_engine` op vtables kCFI-typed; defense against host-induced indirect-call confusion.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `data_queue` virtqueue pointer, per-session-id mapping pointer, and per-`Request` pointer behind CAP_SYSLOG; suppress `%p` in tracepoints.
- **GRKERNSEC_DMESG** — restrict virtio-crypto error (`VIRTIO_CRYPTO_BADMSG`, `_NOTSUPP`, `_INVSESS`, `_NOSPC`, `_KEY_REJECTED`) banners to CAP_SYSLOG so attackers cannot probe host-side state from guest dmesg.
- **CAP_SYS_ADMIN strict on debugfs / config-change knobs** — any guest-side override of advertised algorithm mask requires CAP_SYS_ADMIN in init userns.
- **Virtqueue descriptor PAX_USERCOPY bound** — `virtqueue_add_sgs` out/in counts strictly bounded by per-virtqueue max; refuse oversize SGLs (defense against guest-driven host OOM).
- **Session-id PAX_REFCOUNT** — per-session-id consumer refcount; refuse DESTROY while consumers > 0; refuse DATA_OP on destroyed session.
- **Host status-byte allowlist** — guest treats unknown host status byte values as fatal error; refuse to interpret bits beyond defined `VIRTIO_CRYPTO_*` codes.
- **Per-data-vq affinity audit** — virtcrypto_set_affinity respects cgroup cpuset bindings; refuse pinning outside allowed cpuset.
- **Backlog bound** — per-engine backlog clamped; defense against host-side stall causing guest unbounded queue.
- **PAX_MEMORY_SANITIZE on `OpCtx` private state** — RSA private exponent / EC priv key zeroed on `cra_exit`.

Rationale: virtio-crypto is unusual within `drivers/crypto/` in that the "backend" is a (potentially-hostile or buggy) hypervisor rather than a trusted ASIC. A refcount underflow on a session-id while the host returns a re-used id, a missing key-buffer zeroization, a torn virtqueue descriptor, or an unbounded backlog gives the host a foothold to extract guest key material or DoS the guest. RAP/kCFI on the crypto_alg vtable, PAX_REFCOUNT on session-id consumer counts, PAX_USERCOPY-bounded virtqueue desc-table writes, host-status-byte allowlist, and explicit `memzero_explicit` on per-session key buffers turn the paravirt crypto interface from "trusted because the hypervisor is in the TCB" into a structural guest-side enforcement boundary (especially important for CoCo guests where the hypervisor is NOT in the TCB).

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Host-side virtio-crypto backend (QEMU vhost-user-crypto, vhost-crypto, software fallback) — covered out-of-tree
- Cross-ref `drivers/virtio/00-overview.md` for virtio core (virtqueue allocation, IRQ routing, feature negotiation)
- `crypto_engine` infrastructure (covered in `.design/crypto/algapi.md`)
- Live-migration of virtio-crypto state across hosts (handled by vhost-user-crypto userspace)
- Compression service class (not yet implemented in upstream guest driver)
- 32-bit-only paths
- Implementation code
