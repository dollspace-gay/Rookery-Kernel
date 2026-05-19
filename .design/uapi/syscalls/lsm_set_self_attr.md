# Tier-5 syscall: lsm_set_self_attr(2) — syscall 460

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - security/lsm_syscalls.c (SYSCALL_DEFINE4(lsm_set_self_attr), lsm_set_attr)
  - security/security.c (security_setselfattr)
  - include/uapi/linux/lsm.h (struct lsm_ctx, LSM_ATTR_*, LSM_ID_*)
  - include/linux/security.h (security_setselfattr)
  - arch/x86/entry/syscalls/syscall_64.tbl (460  common  lsm_set_self_attr)
-->

## Summary

`lsm_set_self_attr(2)` is the stacking-aware counterpart of `lsm_get_self_attr(2)`: it installs a new value for a per-LSM self attribute (current label, exec-transition label, fs-create label, key-create label, socket-create label; note `LSM_ATTR_PREV` is read-only) on the calling thread.

The caller supplies exactly **one** `struct lsm_ctx` entry that names the target LSM (`ctx[0].id`) and carries the new opaque context bytes. The kernel routes the request to that single LSM's `setselfattr` hook, which performs LSM-specific authorization (e.g. SELinux validates the transition against the policy graph; AppArmor verifies profile change permission; Smack checks `CAP_MAC_ADMIN` for labels above the current).

Critical for: container runtime label transitions, systemd-exec MAC setexec, dbus-broker per-method label switches, init re-exec under a new SELinux domain.

## Signature

```c
int lsm_set_self_attr(
    unsigned int attr,
    struct lsm_ctx *ctx,
    __u32 size,
    __u32 flags
);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `attr` | `unsigned int` | in | One of `LSM_ATTR_*` (writable values only: `CURRENT`, `EXEC`, `FSCREATE`, `KEYCREATE`, `SOCKCREATE`). |
| `ctx` | `struct lsm_ctx *` | in | Pointer to a SINGLE `lsm_ctx` containing the LSM ID and the new context bytes. |
| `size` | `__u32` | in | Total bytes of `ctx` (header + ctx bytes, no trailing pad). Must equal `ctx->len`. |
| `flags` | `__u32` | in | Reserved; MUST be `0`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; new attribute installed. |
| `-1` + `errno` | Failure; no change. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | Bad `attr` (e.g. `LSM_ATTR_PREV` — read-only), `flags != 0`, `size != ctx->len`, `ctx->id == 0`, malformed context bytes. |
| `EFAULT` | `ctx` invalid. |
| `EPERM` | LSM-specific authorization denied (e.g. SELinux policy denies transition; missing `CAP_MAC_ADMIN`). |
| `EOPNOTSUPP` | Target LSM does not implement the requested attribute. |
| `ENOENT` | `ctx->id` does not match any active LSM. |
| `ERANGE` | Context bytes exceed LSM-defined maximum. |
| `ENOSYS` | Kernel built without `CONFIG_SECURITY`. |

## ABI surface

```text
__NR_lsm_set_self_attr  (x86_64) = 460
__NR_lsm_set_self_attr  (arm64)  = 460
__NR_lsm_set_self_attr  (riscv)  = 460
__NR_lsm_set_self_attr  (i386)   = 460

/* Exactly ONE lsm_ctx entry per call. Multi-LSM transitions require
   multiple syscalls (one per LSM). */
```

## Compatibility contract

REQ-1: Syscall number is **460** on x86_64. ABI-stable.

REQ-2: `attr` MUST be one of the writable `LSM_ATTR_*` values. `LSM_ATTR_PREV` is read-only and returns `-EINVAL`.

REQ-3: `flags` MUST be `0`. No flags are currently defined.

REQ-4: `size` MUST equal `ctx->len` — the kernel uses `size` to bound the user copy but validates `ctx->len` for consistency.

REQ-5: `ctx->id` MUST be a non-zero, currently-active LSM ID. Inactive or unknown ID ⟹ `-ENOENT`.

REQ-6: `ctx->flags` field is currently reserved per-entry and MUST be `0`.

REQ-7: `ctx->ctx_len` MUST satisfy `header_size + ctx_len <= size`. Underclaim is a bug; overclaim ⟹ `-EINVAL`.

REQ-8: Target LSM's `setselfattr` hook is the SOLE authorization point. The syscall layer does NOT enforce a generic capability — capability requirements (e.g. `CAP_MAC_ADMIN`) are LSM-internal.

REQ-9: On success, the LSM-owned label storage for the calling task is updated atomically; concurrent `lsm_get_self_attr(2)` reads either see the old or the new label, never a torn intermediate.

REQ-10: Setting an exec-transition label (`LSM_ATTR_EXEC`) does NOT change the current label; it changes the label that will be applied on the next `execve(2)`.

REQ-11: Setting `LSM_ATTR_FSCREATE` / `KEYCREATE` / `SOCKCREATE` to a zero-length context clears the override (subsequent creations fall back to default policy).

REQ-12: A label change to `LSM_ATTR_CURRENT` may require the LSM to retag inheritable resources (open fds, mm). The LSM hook is responsible.

REQ-13: This call IS audited by the audit subsystem when audit-LSM is active; failure records reason code.

REQ-14: Concurrent `lsm_set_self_attr` calls from sibling threads are serialized by the LSM's per-task lock.

REQ-15: After success the new label MUST appear in the next `lsm_get_self_attr` call from the same thread.

## Acceptance Criteria

- [ ] AC-1: SELinux unconfined-to-confined transition allowed by policy: returns 0; `lsm_get_self_attr(CURRENT)` shows new label.
- [ ] AC-2: SELinux transition denied by policy: returns `-EPERM`; label unchanged.
- [ ] AC-3: `attr = LSM_ATTR_PREV` returns `-EINVAL`.
- [ ] AC-4: `flags = 1` returns `-EINVAL`.
- [ ] AC-5: `ctx->id = 0` returns `-EINVAL` or `-ENOENT` per kernel choice (both acceptable).
- [ ] AC-6: `ctx->id = LSM_ID_NOT_LOADED` returns `-ENOENT`.
- [ ] AC-7: `size != ctx->len` returns `-EINVAL`.
- [ ] AC-8: Oversize context bytes return `-ERANGE`.
- [ ] AC-9: Zero-length `ctx_len` on `FSCREATE` clears the override.
- [ ] AC-10: Successful `EXEC` label change does NOT alter `CURRENT` label.
- [ ] AC-11: Audit record emitted on success and on `-EPERM`.

## Architecture

```rust
#[syscall(nr = 460, abi = "sysv")]
pub fn sys_lsm_set_self_attr(
    attr: u32,
    ctx: UserPtr<u8>,
    size: u32,
    flags: u32,
) -> isize {
    Lsm::set_self_attr(attr, ctx, size, flags)
}
```

`Lsm::set_self_attr(attr, ctx_p, size, flags) -> isize`:
1. let attr = LsmAttr::try_from(attr).map_err(|_| EINVAL)?;
2. if attr == LsmAttr::Prev { return Err(EINVAL); }
3. if flags != 0 { return Err(EINVAL); }
4. if size < size_of::<LsmCtxHeader>() as u32 { return Err(EINVAL); }
5. let hdr: LsmCtxHeader = ctx_p.read()?;                       // EFAULT
6. if hdr.len != size as u64 { return Err(EINVAL); }
7. if hdr.flags != 0 { return Err(EINVAL); }
8. let ctx_bytes_len = hdr.ctx_len as usize;
9. if size_of::<LsmCtxHeader>() + ctx_bytes_len > size as usize { return Err(EINVAL); }
10. let lsm = Lsm::lookup_active(hdr.id).ok_or(ENOENT)?;
11. let hook = lsm.attr_hook(attr).ok_or(EOPNOTSUPP)?;
12. let mut ctx_bytes = vec![0u8; ctx_bytes_len];
13. ctx_p.add(size_of::<LsmCtxHeader>()).copy_in(&mut ctx_bytes)?;
14. hook.set_self(current.task(), &ctx_bytes)?;                 // EPERM / ERANGE
15. audit::lsm_label_change(attr, hdr.id, &ctx_bytes, true);
16. Ok(0)

`Lsm::lookup_active(id) -> Option<&'static dyn LsmModule>`:
1. for lsm in Lsm::active_iter() {
2.    if lsm.id() == id { return Some(lsm); }
3. }
4. None

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prev_rejected` | INVARIANT | `attr == LSM_ATTR_PREV` ⟹ `-EINVAL`. |
| `flags_strict_zero` | INVARIANT | `flags != 0` ⟹ `-EINVAL`. |
| `size_eq_len` | INVARIANT | `size != hdr.len` ⟹ `-EINVAL`. |
| `single_lsm_dispatch` | INVARIANT | exactly one LSM hook invoked per call. |
| `failure_no_state_change` | INVARIANT | any error ⟹ label unchanged. |
| `hdr_flags_zero` | INVARIANT | `hdr.flags != 0` ⟹ `-EINVAL`. |

### Layer 2: TLA+

`security/lsm-set-self-attr.tla`:
- Per-call transitions: parse, validate, copy, dispatch, audit.
- Properties:
  - `safety_atomic_label_swap` — old → new transition is atomic w.r.t. concurrent get.
  - `safety_no_lsm_skip` — exactly one LSM hook invoked.
  - `safety_audit_on_deny` — `-EPERM` always emits audit record.
  - `liveness_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Lsm::lookup_active` post: None iff no active LSM has id | `Lsm::lookup_active` |
| `Lsm::set_self_attr` post: success ⟹ subsequent get returns new bytes | `Lsm::set_self_attr` |
| `LsmCtxHeader::validate` post: rejects oversize ctx_len | `LsmCtxHeader::validate` |
| `audit::lsm_label_change` post: emitted on both success and EPERM | `audit::lsm_label_change` |

### Layer 4: Verus / Creusot functional

Per-`lsm_set_self_attr(2)` man page; per-`Documentation/userspace-api/lsm.rst`; SELinux `security_setprocattr` semantics; AppArmor `apparmor_setprocattr` semantics.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`lsm_set_self_attr(2)` reinforcement:

- **Per-`PREV` read-only enforcement** — defense against per-history rewriting attack.
- **Per-`flags == 0` strict** — defense against per-future-flag smuggle.
- **Per-`size == hdr.len` cross-check** — defense against per-length-confusion attack.
- **Per-`hdr.flags == 0` strict** — defense against per-extension-field smuggling.
- **Per-LSM hook is sole authorizer** — defense against per-syscall-layer policy bypass.

## Grsecurity / PaX surface

- **PaX UDEREF on `ctx`** — defense against per-pointer smuggling for the user buffer; SMAP forced.
- **LSM stacking surface restricted** — `GRKERNSEC_LSM_STACK_LOCKDOWN` permits writes only to LSMs the kernel was booted with; late-loaded LSMs cannot register a `setselfattr` hook.
- **PAX_USERCOPY on context bytes** — bounded copy_from_user uses whitelisted slab; rejects oversize.
- **PAX_REFCOUNT on per-task LSM blob during write** — defense against per-blob UAF if task exits during transition.
- **GRKERNSEC_AUDIT_MAC mandatory** — every successful and denied label change is audited (cannot be silenced).
- **Per-CAP_MAC_ADMIN reinforcement** — `GRKERNSEC_MAC_LOCKDOWN` requires capability for any `LSM_ATTR_CURRENT` change once the system has reached multi-user (i.e. early-boot transitions allowed, late transitions denied without explicit cap).
- **Per-`LSM_ATTR_PREV` strict read-only** — defense against per-history-rewrite; redundant with REQ-2 but enforced again at hook-dispatch.
- **Per-`hdr.flags == 0` strict** — defense against per-extension-field smuggling.
- **GRKERNSEC_HIDESYM on LSM-internal label cookies** — internal pointer cookies never echoed into audit.
- **Per-`ENOENT` on inactive LSM** — defense against per-typosquatting of LSM ID to silently no-op.
- **Per-credential refcount honored** — defense against per-cred UAF if `set_creds()` races with task exit.
- **Per-`fork`-time label inheritance constrained** — `GRKERNSEC_LSM_NO_INHERIT_EXEC` prevents pending `EXEC` label from surviving a fork-without-exec.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `lsm_get_self_attr(2)` (see `lsm_get_self_attr.md`).
- `lsm_list_modules(2)` (see `lsm_list_modules.md`).
- Per-LSM policy semantics (Tier-3 `security/selinux.md`, `security/apparmor.md`).
- Implementation code.
