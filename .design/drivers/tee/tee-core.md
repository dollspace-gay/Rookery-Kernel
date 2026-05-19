# Tier-3: drivers/tee/{tee_core,tee_shm}.c — TEE client framework (`/dev/teeN`, GP TEEC, shared memory)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/tee/00-overview.md
upstream-paths:
  - drivers/tee/tee_core.c
  - drivers/tee/tee_shm.c
  - drivers/tee/tee_shm_pool.c
  - drivers/tee/tee_heap.c
  - drivers/tee/tee_private.h
  - include/linux/tee_core.h
  - include/linux/tee_drv.h
  - include/uapi/linux/tee.h
-->

## Summary

The TEE (Trusted Execution Environment) client subsystem provides a kernel-managed conduit between non-secure userspace and a secure-world OS (OP-TEE, AMD-TEE, Trusty, Qualcomm QTEE). It exposes per-driver `/dev/teeN` character devices (and matching privileged `/dev/teeprivN` for supplicants) implementing the GlobalPlatform TEE Client (TEEC) API surface in ioctl form: open/close session, invoke command, cancel, shared-memory alloc/register, object-invoke. Shared memory is bridged via per-driver pool allocators, dma-buf import, and pinned-user-page registration, and is reference-counted across kernel, userspace mappings, and secure-world references.

This Tier-3 covers `tee_core.c` (~1585 lines: cdev mux, ioctl dispatch, client-UUID derivation, context lifecycle, kernel TEE-client API surface) and `tee_shm.c` (~720 lines: `struct tee_shm` lifecycle, dma-buf wiring, user-page pinning, register/unregister with the backend driver).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct tee_device` | per-backend cdev + class + IDR + pool | `drivers::tee::TeeDevice` |
| `struct tee_context` | per-open kernel/user context, supplicant flags | `drivers::tee::Context` |
| `struct tee_shm` | shared-memory descriptor (pool/dyn/dmabuf/usermapped) | `drivers::tee::Shm` |
| `struct tee_shm_pool` / `tee_shm_pool_ops` | per-driver SHM allocator | `drivers::tee::ShmPool` |
| `tee_device_alloc(desc, parent, pool, drvdata)` / `_register()` / `_unregister()` | per-driver TEE-device lifecycle | `TeeDevice::alloc` / `_register` / `_unregister` |
| `teedev_open(teedev)` / `teedev_ctx_get/put` / `teedev_close_context` | per-context lifecycle | `Context::open` / `_get` / `_put` / `_close` |
| `tee_ioctl_version(ctx, uvers)` / `_shm_alloc` / `_shm_register` / `_open_session` / `_invoke` / `_cancel` / `_close_session` / `_supp_recv` / `_supp_send` / `_object_invoke` | ioctl handlers | `Context::ioctl_*` |
| `tee_session_calc_client_uuid(uuid, method, data)` | RFC4122 UUIDv5 client identity | `Context::calc_client_uuid` |
| `tee_shm_alloc_user_buf(ctx, sz)` / `_kernel_buf` / `_priv_buf` / `_register_user_buf` / `_register_kernel_buf` / `_register_fd` | per-SHM allocators | `Shm::alloc_*` / `_register_*` |
| `tee_shm_get_pa(shm, off, *pa)` / `_va(shm, off)` / `_id(shm)` / `_size(shm)` / `_put(shm)` | SHM accessors | `Shm::pa` / `_va` / `_id` / `_size` / `_put` |
| `tee_shm_register_supp_buf(ctx, addr, len)` / `_register_user_buf` | supplicant/user pinned-page SHM | `Shm::register_*` |
| `tee_client_open_session` / `_invoke_func` / `_close_session` / `_cancel_req` / `_get_version` | in-kernel TEE-client API for trusted-key, hwrng, ftpm consumers | `KernelClient::*` |
| `tee_param_is_memref(p)` / GP attr-type ladder | ioctl parameter typing | `Param::is_memref` |
| `tee_bus_match`, `tee_client_driver_register/unregister` | per-TA `auxiliary_bus` for kernel TEE drivers | `TeeBus::*` |

## Compatibility contract

REQ-1: per-backend driver registers via `tee_device_alloc(desc, parent, pool, drvdata)` supplying `struct tee_driver_ops` callbacks (`get_version`, `open`, `close_context`, `release`, `open_session`, `close_session`, `invoke_func`, `cancel_req`, `supp_recv`, `supp_send`, `shm_register`, `shm_unregister`, `object_invoke_func`).

REQ-2: per-`tee_device` IDs allocated from two halves of the 32-slot bitmap: unprivileged `/dev/teeN` in `[0, 16)`, privileged supplicant `/dev/teeprivN` in `[16, 32)`; `TEE_DESC_PRIVILEGED` selects the upper half.

REQ-3: UAPI ioctl set on `TEE_IOC_MAGIC=0xa4`: `TEE_IOC_VERSION`, `_SHM_ALLOC`, `_OPEN_SESSION`, `_INVOKE`, `_CANCEL`, `_CLOSE_SESSION`, `_SUPPL_RECV`, `_SUPPL_SEND`, `_SHM_REGISTER_FD`, `_SHM_REGISTER`, `_OBJECT_INVOKE` — argument layouts in `include/uapi/linux/tee.h`.

REQ-4: per-session client UUID derived via UUIDv5 from `tee_client_uuid_ns` plus the requested login method (`TEE_IOCTL_LOGIN_PUBLIC` / `_USER` / `_GROUP` / `_APPLICATION` / `_REE_KERNEL`); login-group requires CAP-bounded gid mapping check.

REQ-5: per-`tee_shm` flag matrix: `TEE_SHM_DYNAMIC` (registered with backend), `TEE_SHM_POOL` (pool allocator), `TEE_SHM_USER_MAPPED` (pinned user pages via `pin_user_pages_fast`), `TEE_SHM_DMA_BUF` (imported dma-buf), `TEE_SHM_DMA_MEM` (`dma_alloc_pages` for restricted-memory variants), `TEE_SHM_PRIV` (kernel-only, not mmap-able).

REQ-6: per-`tee_shm` refcount is `refcount_t` with separate user-fd ref (anon-inode), per-mapping mmap ref, and per-backend register ref; release tears down in reverse order — release dma-buf / unpin pages / pool-free, then ctx-put, then dev-put.

REQ-7: parameter array bounded `TEE_IOCTL_MAX_PARAMS = (TEE_MAX_ARG_SIZE - sizeof(tee_ioctl_buf_data)) / sizeof(tee_ioctl_param)`; `TEE_MAX_ARG_SIZE = 4096`; per-call param array `kvmalloc_objs`'d with `size_mul` overflow checks.

REQ-8: per-`tee_context` `supp_nowait` defaults false; supplicant `/dev/teeprivN` open is exclusive (per-`tee_device` `supp.ctx` mutex-guarded slot) and starts the TA-enumeration workqueue.

REQ-9: kernel TEE-client API (`tee_client_open_context`/`tee_client_open_session`/`_invoke_func`/`_close_session`/`_close_context`) used by `trusted_keys`, `hwrng-optee`, `ftpm`, `sdm845-pas`, RTC drivers; matches by `match(ver, data)` callback over registered TEE devices.

REQ-10: `tee_bus` `auxiliary_bus` derivative — TAs enumerated by the backend (OP-TEE PTA_CMD_GET_DEVICES) become auxiliary devices on the TEE bus; in-kernel TEE drivers bind via `tee_client_device_id` UUID match.

REQ-11: `set_fs`-free design: all userspace pointers traverse `copy_from_user`/`copy_to_user` with explicit size + `TEE_MAX_ARG_SIZE` clamping; no kernel-pointer pun across the ioctl boundary.

REQ-12: per-`tee_shm` `id` allocated in per-device IDR; `id` is the cookie returned to userspace + secure-world; `id < 0` means private kernel-only shm.

## Acceptance Criteria

- [ ] AC-1: `xtest -l 1` (OP-TEE regression suite, level 1) PASS on a known-good board with OP-TEE.
- [ ] AC-2: `/dev/tee0` openable as ordinary user; `TEE_IOC_VERSION` returns `TEE_IMPL_ID_OPTEE` and `TEE_GEN_CAP_GP`.
- [ ] AC-3: `tee-supplicant` opens `/dev/teepriv0` exclusively; second concurrent open returns `-EBUSY`.
- [ ] AC-4: per-session client-UUID matches GP spec UUIDv5 vectors for `TEE_LOGIN_USER` (uid=N) and `_GROUP` (gid=N) login methods.
- [ ] AC-5: shared-memory life-cycle — allocate via `TEE_IOC_SHM_ALLOC`, mmap, close fd, munmap, ensure no UAF (`KASAN clean`); register via `TEE_IOC_SHM_REGISTER` and `_REGISTER_FD` (dma-buf) likewise.
- [ ] AC-6: kernel TEE-client API consumer (`trusted_keys keytype=trusted`) round-trips a seal/unseal under OP-TEE without `-EAGAIN` storms.
- [ ] AC-7: parameter-array bounds — `TEE_IOC_INVOKE` with `num_params > TEE_IOCTL_MAX_PARAMS` returns `-EINVAL`.
- [ ] AC-8: per-context release — closing the fd while a session is open triggers backend `close_session` cleanup; no leaked sessions visible to OP-TEE introspection.
- [ ] AC-9: KASAN + KMSAN + UBSAN clean across `xtest -l 1` and concurrent supplicant restart loop.

## Architecture

`TeeDevice` lives in `drivers::tee::TeeDevice`:

```
struct TeeDevice {
  desc: &'static TeeDesc,            // backend ops + flags (privileged or not)
  parent: ArcDevice,                 // platform / pci / auxiliary parent device
  id: u8,                            // 0..32, lower 16 unpriv, upper 16 priv
  cdev: CharDevice,
  class_dev: ClassDevice,
  num_users: Refcount,               // contexts holding this teedev
  c_no_user: Completion,             // signaled when num_users==0
  mutex: Mutex<()>,
  idr: Mutex<IdAllocator<Shm>>,      // per-device shm id allocator
  pool: Option<Arc<ShmPool>>,
  drvdata: NonNull<u8>,              // backend private
  dev_attr_groups: Option<&'static [&'static AttributeGroup]>,
}

struct Context {
  refcount: Refcount,
  teedev: Arc<TeeDevice>,
  data: NonNull<u8>,                 // backend-private context state
  list_shm: Mutex<LinkedList<ShmLink>>,
  cap_memref_null: bool,
  supp_nowait: AtomicBool,
  releasing: AtomicBool,
}

struct Shm {
  refcount: Refcount,
  ctx: Arc<Context>,
  paddr: PhysAddr,
  kaddr: Option<NonNull<u8>>,
  size: usize,
  offset: usize,
  flags: ShmFlags,
  id: i32,
  sec_world_id: u64,                 // backend-allocated secure-world cookie
  num_pages: usize,
  pages: Option<KBox<[NonNull<Page>]>>, // pinned-user-page array for USER_MAPPED
}
```

Open `/dev/teeN`:
1. `tee_open(inode, filp)` resolves the owning `tee_device` from the `cdev`.
2. `teedev_open(teedev)` bumps `num_users`, `kzalloc`s `tee_context`, calls backend `ops->open(ctx)` — which sets up per-context state (OP-TEE allocates `optee_context_data` and, if a supplicant fd, claims the `supp.ctx` slot).
3. `ctx->supp_nowait = false` default — userspace must explicitly opt-out of supplicant waits.

Ioctl dispatch (`tee_ioctl`):
- `TEE_IOC_VERSION` → backend `get_version`; sets `cap_memref_null` if `TEE_GEN_CAP_MEMREF_NULL`.
- `TEE_IOC_SHM_ALLOC` → `tee_shm_alloc_user_buf` → `dma_heap`/pool path; returns anon-inode fd + id + size.
- `TEE_IOC_SHM_REGISTER` → `tee_shm_register_user_buf` → `pin_user_pages_fast` + backend `shm_register`.
- `TEE_IOC_SHM_REGISTER_FD` → `dma_buf_get(fd)` + `dma_buf_attach` + `dma_buf_map_attachment` + backend `shm_register`.
- `TEE_IOC_OPEN_SESSION` → copy `tee_ioctl_open_session_arg`, copy params, compute client UUID per `clnt_login`, backend `open_session`, copy back `session` + `ret` + outputs.
- `TEE_IOC_INVOKE` → copy `tee_ioctl_invoke_arg` + params, backend `invoke_func`, copy back outputs.
- `TEE_IOC_CANCEL` → backend `cancel_req(session, cancel_id)`.
- `TEE_IOC_CLOSE_SESSION` → backend `close_session(session)`.
- `TEE_IOC_SUPPL_RECV` / `_SUPPL_SEND` → backend `supp_recv` / `supp_send` (only valid on `/dev/teeprivN`).
- `TEE_IOC_OBJECT_INVOKE` → backend `object_invoke_func` for GP TEE Object API.

Shared-memory release (`tee_shm_release`):
1. `TEE_SHM_DMA_MEM` → `dma_free_pages(parent, size, page, dma_addr, BIDIRECTIONAL)`.
2. `TEE_SHM_DMA_BUF` → `dma_buf_put(ref->dmabuf)`.
3. `TEE_SHM_POOL` → `pool->ops->free(pool, shm)`.
4. `TEE_SHM_DYNAMIC` → backend `shm_unregister(ctx, shm)`; if `TEE_SHM_USER_MAPPED`, `unpin_user_pages(pages, num_pages)`.
5. `teedev_ctx_put(ctx)`; `tee_device_put(teedev)`.

## Hardening

(Inherits row-1 features from `drivers/tee/00-overview.md` — pending parent doc.)

tee-core specific reinforcement:

- **TEE_MAX_ARG_SIZE bound** — single argument payload capped at 4 KiB; `kvmalloc_objs` + `size_mul` overflow-checked, defeats integer-multiply DoS on `num_params`.
- **per-device unprivileged/privileged id-half split** — supplicant ioctl set (`SUPPL_RECV` / `_SEND`) only available on upper-half `/dev/teeprivN`, enforced by `desc->flags & TEE_DESC_PRIVILEGED` plus per-context check.
- **per-shm IDR with `idr_alloc(... 1, 0 ...)`** — never returns id 0 (reserved for "no shm").
- **`unpin_user_pages` on every USER_MAPPED release path** — eliminates pinned-page leak on abnormal close.
- **`tee_context.releasing` interlock** — prevents new `kref_get` once teardown started; defeats use-after-release on concurrent ioctl + close.
- **`teedev->mutex` around backend `register/unregister`** — single backend-visible mutation; defeats double-register and TOCTOU on `pool` swap.
- **GFP_KERNEL_ACCOUNT** — per-ctx and per-shm allocations memcg-accounted; defeats memcg-bypass exhaustion DoS.
- **supplicant slot exclusivity** — `optee->supp.ctx` mutex-guarded; concurrent open returns `-EBUSY`; defeats supplicant impersonation race.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `tee_context`, `tee_shm`, `tee_param[]`, and `tee_ioctl_*` argument structs; every `copy_from_user`/`copy_to_user` carries an explicit byte count clamped to `TEE_MAX_ARG_SIZE` before allocation.
- **PAX_KERNEXEC** — TEE core text in W^X kernel image; `tee_driver_ops` vtables and ioctl-dispatch jump tables live in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across every `tee_ioctl_*` entry and the supplicant `wait_for_completion_killable` path; defeats stack-pivot harvesting on long-blocking ioctls.
- **PAX_REFCOUNT** — saturating `refcount_t` on `tee_context`, `tee_shm`, `tee_device.num_users`, and per-mapping mmap refs; overflow trap defeats fd-dup + concurrent-close race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `tee_context` (carries client-UUID + login data), `tee_shm` descriptors, parameter arrays, and pool slabs; secure-world-visible cookies are scrubbed via `kfree_sensitive`.
- **PAX_UDEREF** — SMAP/PAN enforced on every ioctl entry; user pointer never deref'd outside `copy_from_user`/`copy_to_user`/`pin_user_pages_fast`.
- **PAX_RAP / kCFI** — `struct tee_driver_ops` (open, release, open_session, close_session, invoke_func, cancel_req, supp_recv, supp_send, shm_register, shm_unregister, object_invoke_func) and shm-pool ops marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms + per-TA UUID + per-shm secure-world cookie disclosure behind CAP_SYSLOG; suppress `%p` in TEE tracepoints (`optee_invoke_fn_*`).
- **GRKERNSEC_DMESG** — restrict supplicant-bus-scan, RPC-fail, and TA-enumeration banners to CAP_SYSLOG; attackers cannot probe installed-TA inventory via dmesg.
- **TEEC ioctl PAX_USERCOPY whitelisting** — `tee_ioctl_param`, `tee_ioctl_buf_data`, `tee_ioctl_open_session_arg`, and `tee_ioctl_invoke_arg` are explicitly USERCOPY-whitelisted at the exact `num_params`-scaled byte count.
- **shm dma-buf PAX_REFCOUNT pairing** — `tee_shm_dmabuf_ref::shm.refcount` and the underlying `dma_buf` refcount cross-validated on every release; saturating-overflow trap on double-`dma_buf_put`.
- **`kfree_sensitive` on tee_context** — login data (uid/gid + UUIDv5 name buffer) and per-context backend state cleared before free; client-UUID never bleeds across reuse.
- **session-UUID + supplicant gate CAP_SYS_ADMIN** — opening `/dev/teeprivN` requires CAP_SYS_ADMIN in the owning user namespace; supplicant-driven TA enumeration banned for unprivileged contexts.
- **Login-group gid mapping check** — `TEE_IOCTL_LOGIN_GROUP` / `_GROUP_APPLICATION` requires the calling task to be a member of the requested kgid in its user-namespace; refuse if cross-namespace.
- **User-page pinning bound** — `tee_shm_register_user_buf` clamps `num_pages` against an admin-configured policy ceiling; defeats RLIMIT_MEMLOCK bypass via shared-memory registration storm.

Rationale: TEE-core is the only kernel path that hands buffers + parameter arrays into a secure-world OS that the kernel itself cannot debug or step. A missed `copy_from_user` length, a pinned-page leak, or a refcount underflow propagates straight into the secure world, where the consequences (key disclosure, TA impersonation, RPMB rollback) are unobservable from the REE. RAP/kCFI on the backend vtable, CAP_SYS_ADMIN on `/dev/teeprivN`, login-method gid validation, refcount-overflow trap, and `kfree_sensitive` on client-UUID-bearing structs turn the TEEC API from "trust-the-backend" into a structurally enforced boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- OP-TEE driver internals (covered in `optee.md`)
- AMD-TEE driver (future Tier-3 `amdtee.md`)
- Qualcomm QTEE driver (future Tier-3 `qcomtee.md`)
- Arm TS-TEE FF-A driver (future Tier-3 `tstee.md`)
- GP TEE Object API specifics (future Tier-3 `tee-object.md`)
- Trusted-keys / hwrng / ftpm kernel TEE consumers (covered in their own subsystem docs)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `param_array_no_oob` | OOB | `num_params * sizeof(tee_param) <= TEE_MAX_ARG_SIZE` checked before `kvmalloc_objs` |
| `shm_id_unique` | UNIQUENESS | per-device IDR never re-issues active id |
| `ctx_no_uaf_on_close` | UAF | `ctx.releasing` flag blocks new `kref_get`; release after final put |
| `pinned_pages_balanced` | LEAK | every `pin_user_pages_fast` paired with `unpin_user_pages` on any release path |

### Layer 2: TLA+

`models/tee/supp_slot.tla`: proves the supplicant-slot mutex + completion handshake (`supp.reqs_c` / `req.c`) is deadlock-free under concurrent `SUPPL_RECV` + RPC submission + supplicant `close`.

### Layer 3: Verus invariants

- `Shm::release` post: every flag-branch path frees the matching resource exactly once (dma_buf_put, unpin_user_pages, pool->free, or shm_unregister).
- `Context::ioctl_*` post: every successful path returns with `ctx.refcount` and `shm.refcount` unchanged net.

### Layer 4: Functional

`xtest -l 1` regression suite over OP-TEE QEMU image; concurrent supplicant-restart + xtest stress loop with KASAN/KMSAN/UBSAN enabled.
