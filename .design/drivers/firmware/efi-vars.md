# Tier-3: drivers/firmware/efi/vars.c + efivarfs — EFI variables, secure boot, MOK, capsule

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/firmware/00-overview.md
upstream-paths:
  - drivers/firmware/efi/vars.c
  - drivers/firmware/efi/efivars.c
  - drivers/firmware/efi/efi.c
  - fs/efivarfs/super.c
  - fs/efivarfs/inode.c
  - fs/efivarfs/file.c
  - fs/efivarfs/vars.c
  - fs/efivarfs/internal.h
  - include/linux/efi.h
-->

## Summary

EFI variable plumbing — `vars.c` is the per-backend registration shim (`efivars_register` / `efivars_unregister`) plus the locked/non-blocking variable get/set/query helpers, `efivars.c` is the generic ops backend that wraps the EFI runtime services (`GetVariable`, `SetVariable`, `GetNextVariableName`, `QueryVariableInfo`), `fs/efivarfs/*` is the user-visible filesystem that exposes every variable as `<name>-<vendor-GUID>` under `/sys/firmware/efi/efivars/`. The same backend layer feeds the secure-boot variable namespace (PK, KEK, db, dbx), the MOK variable enrollment path (`MokListRT`, `MokListXRT`, `SbatLevelRT`) parked in pre-ExitBootServices memory by shim, and the EFI capsule update workflow.

This Tier-3 covers the variable runtime contract end-to-end: how `efivars_register` is plumbed, how `efivarfs` mediates user reads/writes, what lockdown / capability gates apply, and how the secure-boot + MOK + capsule write surfaces are gated. The EFI core (memmap, ESRT, runtime-services wrappers) is split into `efi-core.md`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct efivars __efivars` | currently-registered backend | `drivers::efi::vars::Registry::current` |
| `struct efivar_operations` | per-backend ops: `get_variable`, `set_variable`, `set_variable_nonblocking`, `get_next_variable`, `query_variable_store`, `query_variable_info` | `drivers::efi::vars::Operations` |
| `efivars_register(&efivars, &ops)` / `efivars_unregister(...)` | per-backend register / unregister | `Registry::register` / `unregister` |
| `efivar_supports_writes()` | does the current backend implement set_variable | `Registry::supports_writes` |
| `efivar_lock()` / `efivar_trylock()` / `efivar_unlock()` | global efivars semaphore | `Registry::lock` / `try_lock` |
| `efivar_get_variable(name, vendor, &attr, &size, data)` | per-name read | `Registry::get` |
| `efivar_get_next_variable(&name_size, name, vendor)` | per-namespace enumerate | `Registry::next` |
| `efivar_set_variable(name, vendor, attr, size, data)` / `efivar_set_variable_locked(..., nonblocking)` | per-name write | `Registry::set` / `set_locked` |
| `efivar_query_variable_info(attr, &storage_space, &remaining_space, &max_var_size)` | per-attribute quota | `Registry::query_info` |
| `check_var_size(nonblocking, attr, size)` | per-backend quota check before set | `Registry::check_var_size` |
| `generic_ops_register()` / `generic_ops_unregister()` | wire `efi.{get,set,...}_variable` into a backend | `GenericBackend::register` / `unregister` |
| `efivar_supports_writes` notifier chain `efivar_ops_nh` | EFIVAR_OPS_RDONLY / EFIVAR_OPS_RDWR | `Registry::ops_notifier` |
| `efivarfs_fill_super(sb, fc)` | efivarfs mount | `fs::efivarfs::fill_super` |
| `efivarfs_create(dir, dentry, mode, true)` / `efivarfs_unlink(dir, dentry)` | per-variable inode create/destroy | `fs::efivarfs::create` / `unlink` |
| `efivarfs_file_write(file, buf, count, ppos)` / `efivarfs_file_read(...)` | per-variable I/O | `fs::efivarfs::write` / `read` |
| `efivar_init(callback, data, duplicate_check, head)` | walk every variable at mount + invoke callback | `fs::efivarfs::init_walk` |
| `efivar_validate(vendor, name, data, size)` | per-name attribute validation (immutability flags, signed-update attr, secure-boot namespace) | `fs::efivarfs::validate` |
| `efivar_ssdt_load()` | (debug) load SSDT from a UEFI variable named by `efivar_ssdt=` | covered in `efi-core.md` |

## Compatibility contract

REQ-1: Exactly one `struct efivars` may be registered at a time. `efivars_register` rejects a second registration with `-EBUSY` and emits a warning; this preserves the upstream invariant of `__efivars` being a singleton.

REQ-2: The global `efivars_lock` semaphore serializes every read/write/enumerate; `down_interruptible` is used for blocking callers, `down_trylock` for atomic-callable paths (e.g. pstore write from oops context).

REQ-3: `efivar_set_variable_locked(..., nonblocking=true)` must not sleep; backends that lack a `set_variable_nonblocking` fall back to `set_variable` which is allowed to sleep — callers must opt out of nonblocking when they cannot tolerate that fallback.

REQ-4: `check_var_size` rejects per-variable writes larger than 64 KiB by default; backends that implement `query_variable_store` may permit larger writes when firmware quota allows.

REQ-5: `efivar_ops_nh` blocking notifier chain is invoked on register with `EFIVAR_OPS_RDONLY` or `EFIVAR_OPS_RDWR`; consumers (efivarfs, efivars sysfs) flip their write surfaces accordingly.

REQ-6: efivarfs mounts as a kernel-internal filesystem at `/sys/firmware/efi/efivars/`; per-variable filename is `<name>-<vendor-GUID>` with the first 4 bytes of file contents holding the EFI attribute bitmap and the rest being the variable data.

REQ-7: Variable-name length bounded by `EFI_VAR_NAME_LEN` (typically 1024 UCS-2 chars); per-write `ucs2_strsize(name, EFI_VAR_NAME_LEN)` is added to `size` for quota accounting.

REQ-8: The secure-boot variable namespace (PK, KEK, db, dbx, MokListRT, MokListXRT, SbatLevelRT) is treated as runtime-immutable from userspace unless the firmware accepts a signed update (`EFI_VARIABLE_AUTHENTICATED_WRITE_ACCESS` or `EFI_VARIABLE_TIME_BASED_AUTHENTICATED_WRITE_ACCESS` attr); the kernel enforces that this attribute is present for writes targeting the SB namespace.

REQ-9: `efivar_validate` walks a per-vendor allowlist of known-immutable / known-mutable variables (e.g. `LoaderInfo`, `LoaderTimeInitUSec`, `dbxUpdate`, `MokListRT`) and refuses writes that violate the per-name attribute mask.

REQ-10: Capsule update via `/dev/efi_capsule_loader` is independent of the variable namespace but shares the lockdown / capability gating; the loader uses `efi_capsule_supported` and `efi_capsule_update` from the runtime services (cross-ref `efi-core.md`).

REQ-11: MOK variable runtime entries are exposed read-only via `mokvar-table` (cross-ref `efi-core.md`); enrollment of new MOKs is performed via the `MokListRT` / `MokListTrustedRT` variables which use the standard `efivar_set_variable` path with shim's authenticated attribute.

REQ-12: A backend that returns `EFI_UNSUPPORTED` from `set_variable` or `query_variable_store` does not break enumeration / read paths; the registry treats such returns as "read-only backend" and emits `EFIVAR_OPS_RDONLY` on the notifier chain.

## Acceptance Criteria

- [ ] AC-1: `mount -t efivarfs none /sys/firmware/efi/efivars` succeeds on a UEFI x86 boot with runtime services functional.
- [ ] AC-2: `ls /sys/firmware/efi/efivars/` enumerates at least `BootOrder-*`, `BootCurrent-*`, `Boot0000-*`, `SecureBoot-*`, `SetupMode-*`.
- [ ] AC-3: `efibootmgr -v` round-trips `Boot####` entries (create, modify, delete, reorder).
- [ ] AC-4: Secure-boot enrollment test: `mokutil --import key.cer` succeeds via `MokAuthHash`/`MokAuthSize` variables.
- [ ] AC-5: Lockdown-confidentiality test: with `kernel.lockdown=confidentiality`, writes to secure-boot namespace variables (`db`, `dbx`, `KEK`, `PK`) return `-EPERM`.
- [ ] AC-6: Variable-quota test: writing `query_variable_info()->remaining_space + 1` bytes of data returns `EFI_OUT_OF_RESOURCES`.
- [ ] AC-7: Nonblocking-set test: pstore writes from an oops handler complete without sleeping (no schedule-while-atomic warning).
- [ ] AC-8: efivarfs immutability test: `chattr +i /sys/firmware/efi/efivars/<x>` is honoured; runtime overwrite of an immutable variable returns `-EPERM`.
- [ ] AC-9: Backend swap test: `efivars_unregister` followed by `efivars_register` cycles cleanly; userland efivarfs observes the `EFIVAR_OPS_RDONLY`/`EFIVAR_OPS_RDWR` transition.
- [ ] AC-10: kselftest `tools/testing/selftests/firmware/efi/` (where present) plus userland `fwts uefi` clean.

## Architecture

`Registry` lives in `drivers::efi::vars::Registry`:

```
struct Registry {
  current: RcuCell<Option<Arc<Backend>>>,
  lock: Semaphore<1>,                     // efivars_lock
  ops_nh: BlockingNotifierChain<OpsEvent>,
  generic: Mutex<Option<GenericBackend>>,
}

struct Backend {
  efivars: Arc<EfiVars>,
  ops: &'static Operations,
}

struct Operations {
  get_variable: fn(name, vendor, &attr, &size, data) -> efi_status_t,
  get_next_variable: fn(&name_size, name, vendor) -> efi_status_t,
  set_variable: Option<fn(name, vendor, attr, size, data) -> efi_status_t>,
  set_variable_nonblocking: Option<fn(name, vendor, attr, size, data) -> efi_status_t>,
  query_variable_store: Option<fn(attr, size, nonblocking) -> efi_status_t>,
  query_variable_info: Option<fn(attr, &storage_space, &remaining_space, &max_var_size) -> efi_status_t>,
}
```

Registration path:
1. Backend (e.g. `generic_ops`) calls `efivars_register(&efivars, &ops)`.
2. `down_interruptible(&efivars_lock)` — uninterruptible-on-INTR error path drops registration.
3. Reject second registration with `-EBUSY`.
4. Install `current = Arc::new(Backend { efivars, ops })`.
5. Determine `EFIVAR_OPS_RDWR` if `ops.set_variable.is_some()` else `EFIVAR_OPS_RDONLY`.
6. `blocking_notifier_call_chain(&ops_nh, event, NULL)` so efivarfs flips its write surface.
7. Drop semaphore.

Per-variable read (`efivar_get_variable`):
1. Caller has already taken `efivar_lock`.
2. Dispatch through `current.ops.get_variable(name, vendor, &attr, &size, data)`.
3. Status translation: `EFI_BUFFER_TOO_SMALL` → caller retries with larger buffer; `EFI_NOT_FOUND` → `-ENOENT`; `EFI_SUCCESS` → 0.
4. RCU-deref of `current` is safe because `efivar_lock` is held.

Per-variable write (`efivar_set_variable_locked`):
1. If `data_size > 0`: `check_var_size(nonblocking, attr, data_size + ucs2_strsize(name, EFI_VAR_NAME_LEN))`.
2. Select setvar: `set_variable_nonblocking` if nonblocking-path requested and available, else `set_variable`.
3. Dispatch. Status translation per upstream `efi_status_to_err`.

efivarfs mount:
1. `fill_super(sb, fc)` allocates root inode, creates `efivarfs_alloc_dentry`.
2. `efivar_init(walker, &dir_state, duplicate_check=true, &dir_state.list)` walks every variable, invokes `efivarfs_callback` which creates one inode per `<name>-<vendor-GUID>`.
3. Inode `i_op->setattr` allows immutable-flag toggling for known-immutable variables.
4. `file->f_op->{read,write}` are `efivarfs_file_read` / `efivarfs_file_write`.

Per-variable user write (efivarfs):
1. `efivarfs_file_write(file, buf, count, ppos)` — count must be at least `sizeof(u32)` (attribute prefix).
2. `efivar_validate(vendor, name, data, size)` — refuse if per-name immutability is set without the authenticated-write attribute.
3. If write is in the secure-boot namespace, require `EFI_VARIABLE_*AUTHENTICATED_WRITE_ACCESS` attribute.
4. `efivar_lock()`, `efivar_set_variable_locked(..., nonblocking=false)`, `efivar_unlock()`.
5. Translate EFI status to `-errno` for userspace.

Variable enumeration on mount:
1. `efivar_get_next_variable(&name_size, name, vendor)` in a loop until `EFI_NOT_FOUND`.
2. Per-entry: `efivar_validate` decides whether to publish; per-duplicate-check pass refuses duplicates (defense against a backend that returns the same name twice).
3. Bounded by `EFI_VAR_NAME_LEN` and a soft cap on total enumerated entries (defense against runaway backends).

Lockdown gates:
- `LOCKDOWN_NONE`: full read/write surface.
- `LOCKDOWN_INTEGRITY_MAX`: refuse writes to secure-boot namespace and to MOK enrollment variables (`MokListRT`, `MokListTrustedRT`).
- `LOCKDOWN_CONFIDENTIALITY_MAX`: refuse all writes plus refuse reads of selected confidentiality-sensitive variables (e.g. firmware-resource log entries).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `name_strsize_no_oob` | OOB | `ucs2_strsize(name, EFI_VAR_NAME_LEN)` bounded by `EFI_VAR_NAME_LEN * 2` bytes. |
| `register_singleton` | UNIQUENESS | `efivars_register` rejects second registration with `-EBUSY`; never installs two backends concurrently. |
| `nonblocking_no_sleep` | LIVENESS | `efivar_set_variable_locked(.., nonblocking=true)` dispatches only to `set_variable_nonblocking` when available; if not available and `nonblocking==true`, refuse rather than sleep. |
| `enumerate_bounded` | OOB | mount-time `efivar_init` walk bounded by total variable count + soft cap; refuses runaway backend. |

### Layer 2: TLA+

`models/efi/vars_serialization.tla` (this doc): models the global semaphore + per-backend register/unregister cycle and proves that no read/write can observe a partially installed backend, and that the `EFIVAR_OPS_RDWR`/`EFIVAR_OPS_RDONLY` notifier event is delivered before any efivarfs write surface flips.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `efivars_register` post: `current.is_some()` and `ops_nh` fired with correct event | `Registry::register` |
| `efivar_set_variable` post: backend invoked exactly once, lock held throughout | `Registry::set` |
| `efivarfs_file_write` post: validated-attribute mask matches firmware-accepted mask | `fs::efivarfs::write` |
| `efivar_validate` post: refuses writes targeting SB namespace without authenticated-write attribute | `fs::efivarfs::validate` |

### Layer 4: Verus/Creusot functional

Write `BootOrder` via efivarfs -> backend `SetVariable` -> firmware-side persistence -> reread via efivarfs returns identical bytes (modulo attribute prefix). Encoded as a refinement from the efivarfs file contents into the backend variable-store byte view.

## Hardening

(Inherits row-1 features from `drivers/firmware/00-overview.md` § Hardening.)

EFI vars specific reinforcement:

- **Single-backend singleton** — `efivars_register` rejects concurrent backends; defense against driver-stacking confusion.
- **Variable-name length bound** — `EFI_VAR_NAME_LEN` enforced on every read/write; defense against ucs2_strsize integer overflow.
- **Write-size quota** — `check_var_size` rejects writes larger than firmware's stated remaining-space minus per-variable overhead.
- **Nonblocking-path strictness** — `set_variable_nonblocking` is the only setvar permitted from atomic context; defense against schedule-while-atomic from pstore-on-oops.
- **Lockdown integrity check** — every write to secure-boot namespace gated on lockdown mode + authenticated-write attribute.
- **Validate per-name allowlist** — known-immutable variables refused even with the authenticated-write attribute when lockdown is high.
- **Capsule submit gated** — `efi_capsule_loader` write requires CAP_SYS_ADMIN + lockdown integrity-level check + per-capsule GUID against `efi_capsule_supported`.
- **MOK enrollment audit** — every MOK enrollment via `MokListRT` writes an audit record (subject UID, key fingerprint).

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for variable-name UCS-2 buffers, variable-data buffers (`efivar_entry`), capsule staging; every `copy_from_user`/`copy_to_user` is sized with `EFI_VAR_NAME_LEN` or the per-variable max from `query_variable_info`.
- **PAX_KERNEXEC** — `vars.c`, `efivars.c`, and `fs/efivarfs/*` live in `__ro_after_init` text after registration; the `efivar_operations` vtable is `__ro_after_init` once the backend is registered.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `efivarfs_file_write`, `efivar_set_variable`, `efivar_init` enumerate, and `efi_capsule_write`.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-variable `efivar_entry` references (efivarfs inode lifetime), on the `Backend` Arc, and on capsule staging buffers; overflow trap kills the offending task.
- **PAX_MEMORY_SANITIZE** — zero-on-free for variable-data buffers (secret variables like PK/KEK/db key material), capsule staging buffers, and MOK enrollment payloads.
- **PAX_UDEREF** — SMAP/PAN enforced on every `efivarfs_file_write`/`_read` and `efi_capsule_loader_write`; reject user-pointer deref outside canonical helpers.
- **PAX_RAP / kCFI** — `efivar_operations` vtable (`get_variable`, `set_variable`, `set_variable_nonblocking`, `get_next_variable`, `query_variable_store`, `query_variable_info`) marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms in EFI variable error paths behind CAP_SYSLOG; suppress `%p` of variable-data pointers and backend function-pointers.
- **GRKERNSEC_DMESG** — restrict efivarfs mount banners, secure-boot enrollment logs, and variable-quota failure messages to CAP_SYSLOG.
- **SetVariable CAP_SYS_ADMIN** — every write surface in efivarfs and the raw `efivar_set_variable` API requires CAP_SYS_ADMIN in the owning user namespace; userland mount-then-write does not bypass.
- **MOK enrollment capability** — `MokListRT`/`MokListTrustedRT` updates require CAP_MAC_ADMIN in addition to CAP_SYS_ADMIN; defense against compromised-userland MOK injection.
- **Variable name allowlist for shim** — when the shim-published MOK enrollment path is active, only variables matching the shim allowlist (`MokListRT`, `MokListTrustedRT`, `MokListXRT`, `SbatLevelRT`, `MokAuth*`) may be written; defense against shim-impersonation variable abuse.
- **Name length SIZE_OVERFLOW** — `ucs2_strsize(name, EFI_VAR_NAME_LEN)` traps on overflow; defense against integer-overflow on name+size accounting.
- **Secure-boot namespace immutability** — PK, KEK, db, dbx writes refused outright when lockdown is `confidentiality`; refused without authenticated-write attribute under `integrity`.
- **Capsule loader lockdown** — `/dev/efi_capsule_loader` denied when lockdown is `confidentiality`; under `integrity` requires authenticated capsule per `efi_capsule_supported`.

Rationale: EFI variables are the only mutable surface that survives reboot under firmware authority. A relaxed lockdown gate on `SetVariable`, a missing name-length bound, or a backend-stacking confusion bug here will let an attacker promote userland code into the secure-boot trust set or pin a boot-loader of their choice in BootOrder. CAP_SYS_ADMIN + CAP_MAC_ADMIN on the MOK path, lockdown gates on the SB namespace, kCFI on the operations vtable, RCU-singleton backend registration, and authenticated-attribute enforcement turn the EFI variable surface from "anything privileged can write" into "only signed updates may reach the secure-boot trust roots."

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- EFI core runtime services + memmap (covered in `efi-core.md`)
- EFI stub / libstub (covered in `efi-stub.md` future Tier-3)
- pstore EFI variable backend (covered in `pstore.md` future Tier-3)
- shim internals (out-of-tree, but `MokListRT` schema is documented here as a contract)
- Implementation code
