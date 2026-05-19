---
title: "Tier-5 syscall: lsm_get_self_attr(2) — syscall 459"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`lsm_get_self_attr(2)` retrieves the calling thread's per-LSM security attribute(s) of a given kind (current label, exec-transition label, fs-create label, key-create label, previous label, socket-create label) and writes the results into a caller-supplied buffer of `struct lsm_ctx` entries. The kernel iterates across all active LSMs that publish a hook for the requested attribute and emits one `lsm_ctx` per LSM, with the LSM ID, attribute ID, contextual flags, and the variable-length context bytes.

This is the post-LSM-stacking replacement for the old single-LSM `/proc/self/attr/{current,exec,prev,...}` reads. It enables a process to enumerate the labels assigned to it by every stacked LSM (SELinux, AppArmor, Smack, BPF-LSM, Landlock, IMA, ...) in one syscall.

Critical for: stacking-aware userspace (systemd, container runtimes, dbus-broker, audit), label introspection without relying on the legacy single-major-LSM `/proc/self/attr/current` file interface.

### Acceptance Criteria

- [ ] AC-1: Single-active-LSM system: returns exactly 1 entry for `LSM_ATTR_CURRENT`.
- [ ] AC-2: Stacked (SELinux+BPF-LSM) system: returns 2 entries for `LSM_ATTR_CURRENT`.
- [ ] AC-3: `LSM_FLAG_SINGLE` with `ctx[0].id = LSM_ID_SELINUX` returns only the SELinux entry.
- [ ] AC-4: `attr = 9999` returns `-EINVAL`.
- [ ] AC-5: `flags = 0xdead` returns `-EINVAL`.
- [ ] AC-6: `*size = 0` returns `-E2BIG` with `*size` set to required.
- [ ] AC-7: `ctx == NULL && *size != 0` returns `-EINVAL`.
- [ ] AC-8: Attribute not implemented by any active LSM returns `-EOPNOTSUPP`.
- [ ] AC-9: Buffer too small for one of several entries returns `-E2BIG` (atomic: nothing partial).
- [ ] AC-10: Entries are 8-byte aligned and iterable by `len`.

### Architecture

```rust
#[syscall(nr = 459, abi = "sysv")]
pub fn sys_lsm_get_self_attr(
    attr: u32,
    ctx: UserPtr<u8>,
    size: UserPtr<u32>,
    flags: u32,
) -> isize {
    Lsm::get_self_attr(attr, ctx, size, flags)
}
```

`Lsm::get_self_attr(attr, ctx, size_p, flags) -> isize`:
1. let attr = LsmAttr::try_from(attr).map_err(|_| EINVAL)?;
2. if (flags & !LSM_FLAG_SINGLE) != 0 { return Err(EINVAL); }
3. let mut size = size_p.read()?;                              // EFAULT
4. let single_id = if flags & LSM_FLAG_SINGLE != 0 {
5.     let first: LsmCtxHeader = ctx.read()?;
6.     if first.id == 0 { return Err(EINVAL); }
7.     Some(first.id)
8. } else { None };
9. let entries = Lsm::collect_self(attr, single_id)?;          // EOPNOTSUPP if empty
10. let required = entries.iter().map(|e| e.packed_size()).sum::<u32>();
11. if required > size {
12.    size_p.write(required)?;
13.    return Err(E2BIG);
14. }
15. let mut off = 0u32;
16. for e in &entries {
17.    e.write_to_user(ctx.add(off as usize))?;
18.    off += e.packed_size();
19. }
20. size_p.write(off)?;
21. Ok(entries.len() as isize)

`Lsm::collect_self(attr, single) -> Result<Vec<LsmCtxOwned>>`:
1. let mut out = Vec::new();
2. for lsm in Lsm::active_iter() {
3.    if let Some(id) = single { if lsm.id() != id { continue; } }
4.    let Some(hook) = lsm.attr_hook(attr) else { continue; };
5.    out.push(hook.get_self(current.task())?);
6. }
7. if out.is_empty() { return Err(EOPNOTSUPP); }
8. Ok(out)

### Out of Scope

- `lsm_set_self_attr(2)` (see `lsm_set_self_attr.md`).
- `lsm_list_modules(2)` (see `lsm_list_modules.md`).
- Per-LSM label semantics (Tier-3 `security/selinux.md`, `security/apparmor.md`, etc.).
- Implementation code.

### signature

```c
int lsm_get_self_attr(
    unsigned int attr,
    struct lsm_ctx *ctx,
    __u32 *size,
    __u32 flags
);
```

```c
/* attr values */
#define LSM_ATTR_CURRENT       100
#define LSM_ATTR_EXEC          101
#define LSM_ATTR_FSCREATE      102
#define LSM_ATTR_KEYCREATE     103
#define LSM_ATTR_PREV          104
#define LSM_ATTR_SOCKCREATE    105

/* flags */
#define LSM_FLAG_SINGLE        0x0001  /* return only the LSM whose ID == ctx[0].id */

struct lsm_ctx {
    __u64 id;
    __u64 flags;
    __u64 len;        /* total bytes consumed by this entry */
    __u64 ctx_len;    /* bytes of variable-length context data following */
    __u8  ctx[];      /* opaque, LSM-defined */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `attr` | `unsigned int` | in | One of `LSM_ATTR_*`; selects which attribute kind to retrieve. |
| `ctx` | `struct lsm_ctx *` | out | User buffer to receive packed `lsm_ctx` entries. |
| `size` | `__u32 *` | in/out | In: total byte capacity of `ctx`. Out: total bytes written (or required-size on `-E2BIG`). |
| `flags` | `__u32` | in | `LSM_FLAG_*`. Currently only `LSM_FLAG_SINGLE` is defined. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of `lsm_ctx` entries written. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | Bad `attr`, unknown `flags` bits, `ctx == NULL && *size != 0`, `LSM_FLAG_SINGLE` with `ctx[0].id == 0`. |
| `EFAULT` | `ctx` or `size` user pointer invalid. |
| `E2BIG` | `*size` too small to hold the requested data; `*size` updated to the required size, nothing written into `ctx`. |
| `EOPNOTSUPP` | No active LSM publishes the requested attribute. |
| `ENOSYS` | Kernel built without `CONFIG_SECURITY`. |

### abi surface

```text
__NR_lsm_get_self_attr  (x86_64) = 459
__NR_lsm_get_self_attr  (arm64)  = 459
__NR_lsm_get_self_attr  (riscv)  = 459
__NR_lsm_get_self_attr  (i386)   = 459

/* Each lsm_ctx entry is variable-length: header + ctx bytes + alignment pad.
   Per-entry `len` field is authoritative for iteration. */
```

### compatibility contract

REQ-1: Syscall number is **459** on x86_64. ABI-stable.

REQ-2: `attr` MUST be one of the defined `LSM_ATTR_*` constants; unknown ⟹ `-EINVAL`.

REQ-3: `flags` MUST have only known bits set; currently `LSM_FLAG_SINGLE` is the only legal bit.

REQ-4: With `LSM_FLAG_SINGLE`, the caller pre-populates `ctx[0].id` with the LSM ID they want. The kernel returns only that LSM's entry (or none, if that LSM does not implement the attribute).

REQ-5: Without `LSM_FLAG_SINGLE`, all active LSMs that publish the attribute emit an entry.

REQ-6: Entries are packed with no padding beyond the per-entry `len` rounded up to 8-byte alignment.

REQ-7: `*size` is read first; if zero is passed, the kernel writes the required-size back and returns `-E2BIG` without dereferencing `ctx`.

REQ-8: Per-entry `ctx_len` is the number of context bytes that follow the header.

REQ-9: The variable-length `ctx[]` is OPAQUE — the LSM defines its format (SELinux: NUL-terminated `user:role:type:level` string; AppArmor: profile name; Smack: label; BPF-LSM: caller-defined; Landlock: ruleset ID).

REQ-10: Entries are emitted in stable LSM-ID order so userspace can index reliably.

REQ-11: `ctx == NULL` is legal only when `*size == 0`; combination yields `-E2BIG` with `*size` set to required-size.

REQ-12: Per-`LSM_ATTR_PREV`: returns the label held before the most recent transition (set by `setexeccon` / exec); only meaningful for LSMs that track previous labels.

REQ-13: Per-`LSM_ATTR_KEYCREATE` / `SOCKCREATE`: returns the override label that will be applied to the next key / socket creation, NULL if no override is set.

REQ-14: The returned context bytes are a snapshot at the moment of the syscall; concurrent label changes by other threads in the same task are racy and the kernel makes no atomicity guarantee across multiple attrs.

REQ-15: This syscall does NOT require any capability; a process can always read its own labels.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `attr_validated` | INVARIANT | unknown `attr` ⟹ `-EINVAL` before any LSM iteration. |
| `flags_validated` | INVARIANT | reserved bits ⟹ `-EINVAL`. |
| `size_required_path` | INVARIANT | undersized buffer ⟹ `-E2BIG`, `*size` updated, no bytes written. |
| `entry_alignment` | INVARIANT | every entry's `len` rounded up to 8. |
| `entries_atomic` | INVARIANT | success ⟹ all entries written; failure ⟹ none. |
| `single_id_zero_rejected` | INVARIANT | `LSM_FLAG_SINGLE` with `id == 0` ⟹ `-EINVAL`. |

### Layer 2: TLA+

`security/lsm-get-self-attr.tla`:
- Per-call transitions: parse, size-probe, collect, pack, write, count.
- Properties:
  - `safety_atomic_buffer_write` — buffer never partially populated.
  - `safety_entries_in_stable_lsm_order` — output order matches `Lsm::active_iter`.
  - `safety_e2big_updates_size` — `-E2BIG` always sets `*size` to required.
  - `liveness_finite_lsm_iteration` — bounded LSM set ⟹ syscall terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Lsm::collect_self` post: returns Err(EOPNOTSUPP) iff no active LSM publishes attr | `Lsm::collect_self` |
| `Lsm::get_self_attr` post: ret == entries.len() | `Lsm::get_self_attr` |
| `LsmCtxOwned::packed_size` post: multiple of 8 | `LsmCtxOwned::packed_size` |
| `LsmCtxOwned::write_to_user` post: header.len == packed_size | `LsmCtxOwned::write_to_user` |

### Layer 4: Verus / Creusot functional

Per-`lsm_get_self_attr(2)` man page; per-`Documentation/userspace-api/lsm.rst`; SELinux / AppArmor / Smack hook semantics for current / exec / prev.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`lsm_get_self_attr(2)` reinforcement:

- **Per-`attr` allow-list** — defense against per-future-attr smuggling.
- **Per-`flags` allow-list** — defense against per-flag-bit smuggling.
- **Per-`-E2BIG` size-update before write** — defense against per-info-leak via unsized buffer.
- **Per-LSM stable ID ordering** — defense against per-non-determinism for caller parsers.
- **Per-entry length rounding** — defense against per-misaligned iteration UB in userspace consumers.

### grsecurity / pax surface

- **PaX UDEREF on `ctx` and `size`** — defense against per-pointer smuggling; SMAP forced.
- **LSM stacking surface restricted** — `GRKERNSEC_LSM_STACK_LOCKDOWN` ensures only the kernel-config-pinned LSM set can be enumerated; runtime LSM additions disabled past boot.
- **Per-`ctx_len` zeroed pad** — defense against per-info-leak: the alignment pad between entries is zeroed in-kernel before `copy_to_user`.
- **PAX_USERCOPY on the variable-length context bytes** — bounded write uses whitelisted slab; rejects oversize.
- **GRKERNSEC_HIDESYM on LSM-private label pointers** — internal label cookie addresses never leak via `ctx[]`.
- **Per-`-E2BIG` size-leak guard** — required-size field is the **only** information returned on undersize; no entry contents leaked. `GRKERNSEC_PROC_GETPID` style identity check is applied (must match `current`).
- **PAX_REFCOUNT on per-LSM module refcount during iteration** — defense against per-LSM-rmmod race.
- **Per-attr capability gate (optional GRKERNSEC_LSM_RESTRICT)** — `LSM_ATTR_PREV` exposure can be conditioned on `CAP_AUDIT_READ` to prevent leak of pre-transition label.
- **Per-`LSM_FLAG_SINGLE` id != 0** — defense against per-uninitialized-struct query that would otherwise return all LSMs unintentionally.
- **Per-`active_iter` snapshotted** — defense against per-mid-iteration LSM rmmod via stale pointer.

