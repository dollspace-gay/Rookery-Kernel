# Tier-3: drivers/tee/optee/* — OP-TEE driver (SMC/FF-A ABI, RPC, supplicant, notif)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/tee/tee-core.md
upstream-paths:
  - drivers/tee/optee/core.c
  - drivers/tee/optee/call.c
  - drivers/tee/optee/smc_abi.c
  - drivers/tee/optee/ffa_abi.c
  - drivers/tee/optee/rpc.c
  - drivers/tee/optee/supp.c
  - drivers/tee/optee/notif.c
  - drivers/tee/optee/device.c
  - drivers/tee/optee/protmem.c
  - drivers/tee/optee/optee_private.h
  - drivers/tee/optee/optee_smc.h
  - drivers/tee/optee/optee_ffa.h
  - drivers/tee/optee/optee_msg.h
  - drivers/tee/optee/optee_rpc_cmd.h
-->

## Summary

The OP-TEE driver is the Linaro-maintained reference REE-side driver for the OP-TEE open-source secure-world OS running in Arm TrustZone (Armv7 / Armv8) or as a partition under an FF-A SPM (Armv8.4+, Arm SystemReady). It implements the `tee_driver_ops` interface (see `tee-core.md`) for two transports: the legacy SMC ABI (`smc_abi.c` — raw `arm_smccc_smc` calls into TF-A which routes to OP-TEE) and the FF-A ABI (`ffa_abi.c` — partition-message-send via the Arm Firmware Framework). Above the transport, the driver carries shared `optee_msg` argument structs, manages a dynamic + static shared-memory pool, services RPC callbacks from secure world (sleep, get-time, RPMB, I2C, notif wait), drives a `tee-supplicant` userspace agent for filesystem/RPMB I/O, and enumerates pseudo-TAs (PTAs) onto the TEE bus.

This Tier-3 covers `core.c` (~280 lines: shared init, sysfs, rpmb wiring), `call.c` (~670 lines: msg-arg lifecycle + cookie tracking), `smc_abi.c` (~1980 lines: SMC transport, dynamic shm pool, async notif IRQ, driver probe), `rpc.c` (~460 lines: RPC handlers for get-time, i2c, notif-wait), and `supp.c` (~360 lines: supplicant request queue).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct optee` | per-instance driver state (transport ops, supp, notif, pools) | `drivers::optee::Optee` |
| `struct optee_ops` | transport-specific ops (`do_call_with_arg`, `to_msg_param`, `from_msg_param`) | `drivers::optee::Ops` |
| `struct optee_smc` / `struct optee_ffa` | per-transport substate (sec_caps, memrefs, notif domain) | `drivers::optee::SmcState` / `FfaState` |
| `struct optee_msg_arg` (`OPTEE_MSG_ATTR_TYPE_*`) | shared-memory cmd descriptor crossing SMC boundary | `drivers::optee::MsgArg` |
| `struct optee_session` / `struct optee_context_data` | per-context session list | `drivers::optee::Session` / `ContextData` |
| `optee_open(ctx, cap_memref_null)` / `_release` / `_close_context` | `tee_driver_ops` open/close | `Optee::open` / `_release` / `_close_context` |
| `optee_smc_do_call_with_arg(ctx, shm, offs, sys)` | issue SMC, drain RPC, wait completion | `SmcState::do_call_with_arg` |
| `optee_ffa_do_call_with_arg(ctx, shm, offs, sys)` | issue FF-A direct-msg, drain RPC | `FfaState::do_call_with_arg` |
| `optee_open_session(ctx, arg, param)` / `_close_session` / `_invoke_func` / `_cancel_req` | TEE-client ops | `Optee::open_session` / `_close_session` / `_invoke_func` / `_cancel_req` |
| `optee_get_msg_arg(ctx, num_params, *msg_arg_shm, **msg_arg, *msg_parg)` / `optee_free_msg_arg` | per-call msg-arg allocator | `Optee::get_msg_arg` / `_free_msg_arg` |
| `optee_handle_rpc(ctx, call, &param)` / `handle_rpc_func_cmd_get_time` / `_i2c_transfer` / `_shm_alloc` / `_shm_free` / `_notif_wait` / `_supp_send` / `_supp_recv` | per-RPC-cmd handlers | `Optee::handle_rpc_*` |
| `optee_supp_thrd_req(ctx, func, num_params, param)` / `optee_supp_recv` / `optee_supp_send` | supplicant rendezvous | `Supp::thrd_req` / `_recv` / `_send` |
| `optee_notif_send(optee, key)` / `optee_notif_wait(optee, key, timeout_ms)` / `optee_notif_async_alloc(optee, max_key)` | async-notif primitive | `Notif::send` / `_wait` / `_async_alloc` |
| `optee_enumerate_devices(get_cmd)` | scan PTAs onto TEE bus (`PTA_CMD_GET_DEVICES{,_SUPP,_RPMB}`) | `Optee::enumerate_devices` |
| `optee_shm_register(ctx, shm, pages, num_pages, start)` / `_unregister` | secure-world shm-register fastcall | `Optee::shm_register` / `_unregister` |
| `optee_pool_op_alloc` / `_free` (static + dynamic + protmem variants) | shm-pool implementations | `Pool::alloc` / `_free` |

## Compatibility contract

REQ-1: per-transport probe: SMC driver probes platform device with DT compatible `linaro,optee-tz`; FF-A driver probes via FF-A partition discovery and matches OP-TEE UUID `486178e0-e7f8-11e3-bc5e-0002a5d5c51b`.

REQ-2: SMC handshake: `OPTEE_SMC_CALLS_UID` returns OP-TEE UUID, `_CALLS_REVISION` returns API rev, `_CALL_GET_OS_REVISION` returns OS rev+build-id, `_EXCHANGE_CAPABILITIES` returns `sec_caps` (dynamic shm, RPC arg, async notif, RPMB probe).

REQ-3: per-`do_call_with_arg`: allocates msg-arg in static priv-pool or dynamic pool; passes physical address of msg-arg in `a1:a2`; SMC returns either `RETURN_OK` / `_ETHREAD_LIMIT` (retry) / `_RPC_PREFIX` (RPC needed, secure world parked).

REQ-4: per-RPC: when SMC returns `OPTEE_SMC_RPC_FUNC_*`, driver inspects RPC cmd code, dispatches to handler (`RPC_FUNC_ALLOC` / `_FREE` / `_FOREIGN_INTR` / `_CMD` / `_SHM_ALLOC` / `_SHM_FREE`), then issues `OPTEE_SMC_CALL_RETURN_FROM_RPC` to resume.

REQ-5: per-RPC-CMD: `OPTEE_RPC_CMD_GET_TIME`, `_LOAD_TA`, `_RPMB`, `_FS`, `_NOTIFICATION`, `_I2C_TRANSFER`, `_SHM_ALLOC`, `_SHM_FREE`, `_SUSPEND`, `_PLUGIN`; each parameter-typed and bounds-checked.

REQ-6: per-supplicant `tee-supplicant`: opens `/dev/teepriv0`, blocks in `TEE_IOC_SUPPL_RECV`, services file-IO / RPMB / TEE-FS, posts result via `TEE_IOC_SUPPL_SEND`.

REQ-7: dynamic shm pool: backed by `alloc_pages` + `dma_map_sg`; registered into secure world via `OPTEE_MSG_CMD_REGISTER_SHM` so secure world maps it in its own page-table.

REQ-8: static shm pool: a reserved-memory region (DT `reserved-memory`) `memremap`'d at probe; used for the priv-arg pool when dynamic shm unavailable; per-page tagged `KMEMLEAK_NOT_LEAK`.

REQ-9: async notif (sec_caps `OPTEE_SMC_SEC_CAP_ASYNC_NOTIF`): per-IRQ from secure-world Arm GICv3+ doorbell wakes per-key `wait_for_completion_interruptible_timeout` on `notif.waiters`; per-cpu IRQ enabled via cpuhotplug callback.

REQ-10: kernel-RPMB routing: when `OPTEE_SMC_SEC_CAP_RPMB_PROBE` available, in-kernel `rpmb` framework consumers respond to RPMB RPCs without supplicant; `/sys/class/tee/teeN/rpmb_routing_model` exposes "kernel" vs "user".

REQ-11: protmem (sec_caps `_PROTECTED_MEMORY`): per-region `optee_protmem_alloc` carves out `CMA`-backed pages, registers in secure world as TEE-restricted memory, exposes as dma-buf to userspace for restricted-display / DRM-content protection.

REQ-12: PTA enumeration: `optee_enumerate_devices(PTA_CMD_GET_DEVICES_SUPP)` scans non-RPMB PTAs and registers `tee_client_device` on the TEE bus with UUID drivers; `_RPMB` variant called after RPMB-class device appears.

## Acceptance Criteria

- [ ] AC-1: `xtest` full regression PASS on QEMU Armv8 OP-TEE image (regression + benchmark + GP TEEC compliance).
- [ ] AC-2: `dmesg | grep optee` shows OS-UUID + revision + sec-caps at probe; `ls /sys/class/tee/tee0/` contains `rpmb_routing_model` + `revision`.
- [ ] AC-3: FF-A transport: probe OP-TEE-SPMC under Hafnium + FF-A core driver; xtest level-1 PASS.
- [ ] AC-4: async-notif: latency-sensitive secure-world IRQ wakes REE waiter within configured deadline on RZ/G3S, i.MX8MP, and QEMU virt-targets.
- [ ] AC-5: kernel-RPMB: `optee_rpmb_routing_model == kernel` on a board with eMMC RPMB + sec_cap `RPMB_PROBE`; secure-storage TAs survive supplicant restart.
- [ ] AC-6: dynamic shm: stress register/unregister 1M cycles, no leak (`grep -i optee /proc/slabinfo` stable; KMEMLEAK clean).
- [ ] AC-7: supplicant kill/restart while sessions in-flight: in-flight invocations return `TEEC_ERROR_COMMUNICATION`, no kernel hang.
- [ ] AC-8: PTA enumeration: `ls /sys/bus/tee/devices/` shows expected per-UUID auxiliary devices on the platform.
- [ ] AC-9: protmem: dma-buf import into `drm/v4l2` round-trips with secure-world clearing the region; KASAN clean under register/unregister storm.

## Architecture

`Optee` lives in `drivers::optee::Optee`:

```
struct Optee {
  teedev: Arc<TeeDevice>,            // /dev/teeN
  supp_teedev: Arc<TeeDevice>,       // /dev/teeprivN
  ops: &'static Ops,                 // transport vtable
  smc: Option<KBox<SmcState>>,
  ffa: Option<KBox<FfaState>>,
  pool: Arc<TeeShmPool>,             // dynamic shm pool
  scan_bus_done: AtomicBool,
  scan_bus_work: WorkStruct,
  rpmb_scan_bus_done: AtomicBool,
  rpmb_scan_bus_work: WorkStruct,
  rpmb_intf: NotifierBlock,
  in_kernel_rpmb_routing: bool,
  supp: Supp,
  notif: Notif,
  revision: OsRevision,
  protmem: Option<KBox<ProtMem>>,
}

struct Supp {
  ctx: Mutex<Option<Arc<Context>>>,  // owning supplicant context
  reqs_c: Completion,
  reqs: Mutex<LinkedList<SuppReq>>,
  idr: Mutex<IdAllocator<SuppReq>>,
  req_id: Mutex<i32>,                // currently-recv'd, awaiting send
}

struct Notif {
  lock: SpinLock,
  waiters: BTreeMap<u32, Completion>, // key -> waiter
  active: BTreeMap<u32, bool>,        // key -> already-signaled flag
}
```

SMC do_call_with_arg (`optee_smc_do_call_with_arg`):
1. Build `arm_smccc_res` args: `a0 = OPTEE_SMC_CALL_WITH_ARG` (or `_RPC_ARG`), `a1 = upper(parg)`, `a2 = lower(parg)`, `a3 = cookie`.
2. Loop:
   - `arm_smccc_smc(a0,a1,a2,a3,a4,a5,a6,a7,&res)`.
   - If `res.a0 == OPTEE_SMC_RETURN_OK` → break.
   - If `res.a0 == OPTEE_SMC_RETURN_ETHREAD_LIMIT` → `wait_for_completion(call_queue)` then retry.
   - If `(res.a0 & OPTEE_SMC_RETURN_RPC_PREFIX_MASK) == OPTEE_SMC_RETURN_RPC_PREFIX` → call `handle_rpc(...)` then `a0 = OPTEE_SMC_CALL_RETURN_FROM_RPC` and continue.
3. Return msg-arg `ret` + `ret_origin` to caller.

RPC handler `optee_handle_rpc`:
- `OPTEE_SMC_RPC_FUNC_ALLOC` → secure world wants RPC-arg buffer; `tee_shm_alloc_kernel_buf`; pass back physical address.
- `_FREE` → `tee_shm_free` matching cookie.
- `_FOREIGN_INTR` → schedule via `schedule()` for kthread preemption, loop continues.
- `_CMD` → dispatch on `arg->cmd`: `GET_TIME` / `WAIT_QUEUE` / `SUSPEND` / `SHM_ALLOC` / `SHM_FREE` / `I2C_TRANSFER` / `NOTIFICATION` handled in-driver; remainder forwarded to supplicant via `optee_supp_thrd_req`.

Supplicant rendezvous (`optee_supp_thrd_req`):
1. Allocate `optee_supp_req` with kernel-side params copy.
2. `list_add_tail(&req->link, &supp->reqs)`; `complete(&supp->reqs_c)`.
3. `wait_for_completion_killable(&req->c)` — if killed, unlink from queue (or IDR if already received) and return `TEEC_ERROR_COMMUNICATION`.
4. On supplicant `SUPPL_SEND`, `req->ret` copied back; supplicant-mutated params marshalled back to secure world.

Async notif IRQ (`optee_smc_pcpu_irq_handler`):
1. SMC `OPTEE_SMC_GET_ASYNC_NOTIF_VALUE` until empty.
2. For each returned key: if `key == OPTEE_SMC_ASYNC_NOTIF_VALUE_DO_BOTTOM_HALF` → schedule bottom-half work; else `optee_notif_send(optee, key)` → `complete(&notif.waiters[key])`.

PTA enumeration (`optee_enumerate_devices`):
1. Open kernel TEE context, open session to PTA `pta_device_enum` (UUID `7011a688-...`).
2. Invoke `PTA_CMD_GET_DEVICES{,_SUPP,_RPMB}` to fetch UUID array.
3. For each UUID, allocate `tee_client_device`, set `id.uuid`, `device_register` on `tee_bus`.

## Hardening

(Inherits from `tee-core.md` + `drivers/tee/00-overview.md`.)

OP-TEE specific:

- **SMC return-value validation** — every `res.a0` strictly classified into OK / RPC / ETHREAD_LIMIT / ERROR; unrecognized values → `-EINVAL` + dev_err + abandon call.
- **RPC param-attr ladder validation** — every RPC handler checks `arg->num_params` and per-param `attr` against an explicit attr-array (e.g. I2C transfer requires exactly `VALUE_IN, VALUE_IN, MEMREF_INOUT, VALUE_OUT`); mismatched RPCs return `TEEC_ERROR_BAD_PARAMETERS`.
- **per-call msg-arg via priv-pool** — small SMC arg structs come from a 512-byte-aligned private pool, never from user-controlled shm; defeats secure-world-side argument tampering via user-shm aliasing.
- **dynamic-shm register PA range validation** — every page array passed to `OPTEE_MSG_CMD_REGISTER_SHM` walked + checked vs `pfn_valid + !PageReserved`; refuse non-kernel-RAM PAs.
- **supplicant request killable-wait** — `wait_for_completion_killable` avoids unbounded D-state on hung supplicant; on kill, request cleanly unlinked.
- **per-cpu notif IRQ disabled on cpu-offline** — `optee_cpuhp_disable_pcpu_irq` registered so a CPU shedding doesn't lose IRQs into the void.
- **Workqueue scan_bus_done** — PTA enumeration runs once per probe; cannot be re-armed by supplicant restart spam.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `optee_supp_req`, `optee_context_data`, RPC `tee_param[]` arrays, and the priv-pool msg-arg slab.
- **PAX_KERNEXEC** — OP-TEE driver text W^X; `optee_ops`, RPC-dispatch jump table, and RPMB-class interface notifier in `__ro_after_init`.
- **PAX_RANDKSTACK** — randomize kstack offset across `do_call_with_arg`, `handle_rpc`, `optee_supp_recv`/`_send`, and the per-cpu notif IRQ handler.
- **PAX_REFCOUNT** — saturating `refcount_t` on `optee` itself, on `optee_session` per-context list entries, and on every per-RPC cookie issued to secure world.
- **PAX_MEMORY_SANITIZE** — zero-on-free for msg-arg slabs, supplicant param buffers, secure-page (`protmem`) regions on detach, and PTA-enumeration UUID arrays; secure-page reclaim path scrubs before re-registration.
- **PAX_UDEREF** — SMAP/PAN across every transition from `tee_supp_send` / `_recv` ioctl into driver; user pointers stay funneled through `tee-core` `copy_from_user`/`copy_to_user`.
- **PAX_RAP / kCFI** — `struct optee_ops` (`do_call_with_arg`, `to_msg_param`, `from_msg_param`), per-RPC handler table, and per-pool `tee_shm_pool_ops` marked `__ro_after_init` with kCFI dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms + per-OP-TEE-build-id + per-PTA UUID disclosure behind CAP_SYSLOG; suppress `%p` in `optee_trace.h` events.
- **GRKERNSEC_DMESG** — restrict RPC-error, supplicant-restart, async-notif-storm, and RPMB-probe banners to CAP_SYSLOG.
- **SMC arg PAX_USERCOPY** — every word marshalled into `arm_smccc_res` is a kernel-derived integer; no user pointer is ever passed in `a0..a7`; verified by audit at SMC-call sites.
- **Supplicant RPC allowlist** — RPC commands forwarded to supplicant restricted to an explicit policy allowlist (FS, RPMB, LOAD_TA); kernel-implementable RPCs (GET_TIME, I2C, NOTIFICATION) never reach supplicant.
- **secure-page MEMORY_SANITIZE** — `optee_protmem` regions zeroed before re-issue to secure world; defeats stale-content disclosure across restricted-memory tenants.
- **RPC PA→VA validation** — every physical address received from secure world via RPC (`SHM_FREE` cookies, RPMB block pointers) re-validated against the per-context registered-shm list; reject unknown PAs.
- **CAP_SYS_ADMIN on supplicant fd** — `/dev/teepriv0` open gated CAP_SYS_ADMIN in the owning user namespace; supplicant can request kernel I2C / RPMB / FS, so it is a privileged surface.
- **FF-A partition-id pinning** — the OP-TEE partition id matched at probe is cached in `__ro_after_init` and re-validated on every `ffa_msg_send_direct_req`; reject if remapped.

Rationale: OP-TEE is the kernel's most privileged downstream peer — it can read all platform memory (when TZASC isn't enforced for kernel pages), and it provides services (RPMB sealing, attestation, FDE keys) whose integrity becomes worthless if the REE driver can be coerced into mis-routing RPCs. RAP/kCFI on transport ops, SMC-return classification, RPC allowlisting, per-RPC PA→VA validation, supplicant CAP_SYS_ADMIN, and refcount-overflow trapping on per-session cookies turn the REE driver from a thin SMC shim into a structurally enforced policy gate.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- OP-TEE OS internals (out of kernel tree)
- Arm SMCCC core (covered in arm-smccc Tier-3)
- FF-A core driver (future Tier-3 `arm-ffa.md`)
- TEE-supplicant userspace details (covered in `Documentation/staging/tee.rst`)
- TZASC / TrustZone Address Space Controller config (firmware concern)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rpc_attr_check` | TYPE | every RPC handler validates `num_params` + per-param `attr` against expected pattern before deref |
| `cookie_unique` | UNIQUENESS | per-call cookies allocated from refcounted IDR; never re-issued before `OPTEE_SMC_RETURN_OK` |
| `supp_req_no_uaf` | UAF | killable-wait unlink under `supp.mutex` is exclusive with `idr_remove` |
| `dyn_shm_pfn_valid` | RANGE | every page passed to `REGISTER_SHM` satisfies `pfn_valid(page_to_pfn(page))` |

### Layer 2: TLA+

`models/tee/optee_rpc.tla`: proves SMC ↔ RPC handshake is bounded — every secure-world resume (`CALL_RETURN_FROM_RPC`) eventually returns to a terminal status; no livelock under repeated `ETHREAD_LIMIT` retry.

### Layer 3: Verus invariants

- `Optee::do_call_with_arg` post: returns `RETURN_OK | RPC_RESUMED | ERROR(_)`, never a mid-RPC state.
- `Supp::thrd_req` post: returned cookie either appears in `idr` exactly once or has been killed-and-removed.

### Layer 4: Functional

`xtest` full + xtest stress under KASAN/KMSAN/KCSAN; `xtest -t regression` cross-tested over SMC and FF-A transports; supplicant fault-injection (kill -9 during in-flight RPC).
