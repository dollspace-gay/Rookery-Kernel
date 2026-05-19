---
title: "Tier-5 syscall: add_key(2) — syscall 248"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`add_key(2)` instantiates a kernel-resident key of the requested `type` with the requested `description`, copies the user-supplied `payload` of `plen` bytes into a kernel-owned, type-specific representation, and links the resulting `struct key` into the destination `keyring`. The kernel maintains a typed, per-namespace, ref-counted keystore with per-key possession / view / read / write / search / link / set-attr permission bits and per-uid quotas. Key types supported in baseline include `user` (opaque blob), `logon` (write-only user blob — never readable post-instantiate), `keyring` (a key whose payload is a set of links to other keys), `dns_resolver`, `cifs.idmap`, `rxrpc`, `asymmetric`, `encrypted`, `trusted`, `big_key`. The five default keyrings — `KEY_SPEC_THREAD_KEYRING`, `_PROCESS_KEYRING`, `_SESSION_KEYRING`, `_USER_KEYRING`, `_USER_SESSION_KEYRING` — are addressed by negative `key_serial_t` sentinels and are lazily created on first reference. Critical for: kerberos / cifs credential storage, network filesystem mount-time secret handoff, dm-crypt and fscrypt master keys, TPM-backed sealed keys (`trusted`), userspace-encrypted persistence (`encrypted`), `keyctl(2)` chained operations.

This Tier-5 covers the userspace ABI of syscall 248. Lookup and lifecycle ops live in `request_key.md` and `keyctl.md`.

### Acceptance Criteria

- [ ] AC-1: `add_key("user", "k1", "hello", 5, KEY_SPEC_SESSION_KEYRING)` returns positive serial; `keyctl_read` returns `"hello"`.
- [ ] AC-2: `add_key("logon", "svc:k1", "secret", 6, ...)` returns serial; `keyctl_read` ⟹ `-EOPNOTSUPP`.
- [ ] AC-3: `add_key("logon", "noprefix", ...)` ⟹ `-EINVAL` (no `:` separator).
- [ ] AC-4: `add_key("bogus", ...)` ⟹ `-EINVAL` (unknown type).
- [ ] AC-5: `add_key("user", "", payload, 0, ...)` ⟹ `-EINVAL` (empty desc).
- [ ] AC-6: `plen = 2 MiB` ⟹ `-EINVAL`.
- [ ] AC-7: `keyring = 0xDEAD` (non-existent) ⟹ `-ENOKEY`.
- [ ] AC-8: Caller lacks `KEY_NEED_WRITE` on dest ⟹ `-EACCES`.
- [ ] AC-9: Per-uid `kernel.keys.maxkeys` reached ⟹ `-EDQUOT`.
- [ ] AC-10: Per-uid `kernel.keys.maxbytes` reached ⟹ `-EDQUOT`.
- [ ] AC-11: Bad `type` user pointer ⟹ `-EFAULT`.
- [ ] AC-12: `add_key("keyring", "kr1", NULL, 0, ...)`: returns serial; `keyctl_describe` shows `keyring; kr1`.
- [ ] AC-13: `add_key("keyring", "kr1", "x", 1, ...)` ⟹ `-EINVAL`.
- [ ] AC-14: Existing key replaced: same `(type, description)` re-add returns same serial with updated payload (if `KEY_USR_WRITE`).
- [ ] AC-15: `add_key("asymmetric", "x", garbage, 100, ...)` ⟹ `-EKEYREJECTED`.

### Architecture

Rookery surface in `kernel/security/keys/syscall.rs`:

```rust
pub type KeySerial = i32;

pub struct Key {
    pub serial:        KeySerial,
    pub type_:         &'static KeyType,
    pub description:   KString,
    pub payload:       KeyPayload,
    pub perm:          KeyPerm,
    pub uid:           Uid,
    pub gid:           Gid,
    pub flags:         AtomicU32,        // KEY_FLAG_*: INSTANTIATED, REVOKED, INVALIDATED, NEGATIVE
    pub expiry:        Option<Ktime>,
    pub user_keys:     Arc<KeyUser>,     // per-uid quota anchor
    pub refcount:      RefCount,
}

pub struct KeyUser {
    pub uid:           Uid,
    pub qnkeys:        AtomicU32,
    pub qnbytes:       AtomicU32,
    pub maxkeys:       u32,
    pub maxbytes:      u32,
}
```

`Keyring::add_key(type_uptr, desc_uptr, payload_uptr, plen, ring) -> isize`:
1. let type_name = getname_kernel(type_uptr, 32)?;            // EFAULT/EINVAL
2. let type_ = KeyType::lookup(&type_name).ok_or(-EINVAL)?;
3. type_.allowed_in(current_user_ns())?;                      // EPERM
4. let desc = getname_kernel(desc_uptr, 4096)?; type_.validate_description(&desc)?;  // EINVAL / ENODEV logon
5. if plen > KEY_PAYLOAD_MAX { return -EINVAL; }
6. let payload = if plen > 0 { Some(vmemdup_user(payload_uptr, plen)?) } else { None };
7. let dest = Keyring::lookup_user_key(ring, KEY_LOOKUP_CREATE, KEY_NEED_WRITE)?;
8. Keyring::check_quota(&current_user_keys(), plen)?;         // EDQUOT
9. let mut prep = KeyPreparsePayload::new(payload.as_deref(), plen);
10. type_.preparse(&mut prep)?;                               // EKEYREJECTED/EINVAL
11. let key = Key::alloc(type_, &desc, current_creds(), DEFAULT_PERM, KEY_ALLOC_QUOTA, current_user_keys())?;
12. match Keyring::instantiate_and_link(&key, &prep, &dest) {
13.   Ok(()) => Ok(key.serial as isize),
14.   Err(e) => { Keyring::free_uninstantiated(&key); Err(e) }
15. }

`Keyring::check_quota(ku, plen) -> Result`: atomically checks `qnkeys + 1 ≤ maxkeys` and `qnbytes + plen ≤ maxbytes` under user-keys lock, then increments both counters; failure ⟹ `-EDQUOT` with no delta.

### Out of Scope

- `request_key.md` Tier-5 — upcall-based key lookup
- `keyctl.md` Tier-5 — multiplexed key operations
- `security/keys/keyring.md` Tier-3 — keyring search, restriction, GC
- `security/keys/key-types.md` Tier-3 — per-type instantiation, preparse, destroy callbacks
- Implementation code

### signature

```c
key_serial_t add_key(const char *type, const char *description,
                     const void *payload, size_t plen,
                     key_serial_t keyring);
```

Rust ABI shim:

```rust
pub fn sys_add_key(
    type_:        UserPtr<u8>,
    description:  UserPtr<u8>,
    payload:      UserPtr<u8>,
    plen:         usize,
    keyring:      i32,
) -> isize;
```

Syscall number: **248** (x86_64).

`key_serial_t` is a signed 32-bit identifier; positive = real key, negative = `KEY_SPEC_*` sentinel.

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `type` | `const char *` | IN | NUL-terminated type name (`"user"`, `"logon"`, `"keyring"`, `"asymmetric"`, ...); resolved via the registered `key_type` table |
| `description` | `const char *` | IN | NUL-terminated key description; used both for search-keyring lookup and as the canonical display name (`/proc/keys`) |
| `payload` | `const void *` | IN | type-specific blob; may be `NULL` for some types (e.g. `keyring` ignores payload, must be NULL) |
| `plen` | `size_t` | IN | payload byte length |
| `keyring` | `i32` (`key_serial_t`) | IN | destination keyring; positive serial of an existing keyring or one of `KEY_SPEC_THREAD_KEYRING` / `_PROCESS_KEYRING` / `_SESSION_KEYRING` / `_USER_KEYRING` / `_USER_SESSION_KEYRING` |

### return

- **Success**: positive `key_serial_t` (the new key's serial).
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `ENOKEY` | destination keyring does not exist; or a default keyring sentinel referred to a not-yet-allocated keyring that could not be allocated |
| `EKEYEXPIRED` | destination keyring has expired |
| `EKEYREVOKED` | destination keyring revoked |
| `EINVAL` | unknown `type`; payload format invalid for the type; `description` empty; `plen > KEY_PAYLOAD_MAX` (1 MiB) |
| `ENODEV` | `type` is `"logon"` and `description` lacks the mandatory `"prefix:"` form |
| `EACCES` | caller lacks `KEY_NEED_WRITE` on the destination keyring; or type forbidden in current userns; or attempt to add a non-instantiable type |
| `EDQUOT` | per-uid `key_user.qnkeys` / `qnbytes` quota exhausted (`kernel.keys.maxkeys`, `kernel.keys.maxbytes`) |
| `ENOMEM` | kernel allocation failure (key, description copy, payload preparse) |
| `EFAULT` | any user pointer faults during copy-in |
| `EPERM` | `type` requires capability (`CAP_SYS_ADMIN` for `"asymmetric"` cert import in some modes) |
| `EKEYREJECTED` | type's `instantiate` callback rejected the payload (e.g. malformed asymmetric key) |

### abi surface

`KEY_SPEC_*` keyring sentinels (negative `key_serial_t`):

| Constant | Value | Purpose |
|---|---|---|
| `KEY_SPEC_THREAD_KEYRING` | `-1` | per-thread keyring (cleared on thread exit) |
| `KEY_SPEC_PROCESS_KEYRING` | `-2` | per-process keyring (cleared on exec unless inherited) |
| `KEY_SPEC_SESSION_KEYRING` | `-3` | per-session keyring (inherited across fork, optionally cleared on setresuid) |
| `KEY_SPEC_USER_KEYRING` | `-4` | per-uid persistent keyring |
| `KEY_SPEC_USER_SESSION_KEYRING` | `-5` | per-uid session keyring |
| `KEY_SPEC_GROUP_KEYRING` | `-6` | per-gid keyring (reserved; not used) |
| `KEY_SPEC_REQKEY_AUTH_KEY` | `-7` | request_key auth key for upcall |
| `KEY_SPEC_REQUESTOR_KEYRING` | `-8` | requestor's keyring during upcall |

Per-uid limits (sysctl, in `/proc/sys/kernel/keys/`):
- `maxkeys` — per-uid number-of-keys cap (default `200`).
- `maxbytes` — per-uid total payload-bytes cap (default `20000`).
- `root_maxkeys`, `root_maxbytes` — root's cap.
- `gc_delay` — seconds before a revoked key is GC'd.
- `persistent_keyring_expiry` — auto-expiry for `_USER_KEYRING`.

Permission bits (`key_perm_t`, 32 bits, four groups of 8: possessor / user / group / other):
- `KEY_POS_VIEW` / `READ` / `WRITE` / `SEARCH` / `LINK` / `SETATTR` / `ALL`.
- Same set for `KEY_USR_*`, `KEY_GRP_*`, `KEY_OTH_*`.

### compatibility contract

REQ-1: `type` lookup:
- Resolved against the in-kernel `key_types_list`.
- Unknown ⟹ `-EINVAL`.
- Forbidden types in current userns (`KEY_TYPE_NS_KEYRING_USER` policy) ⟹ `-EPERM`.
- `type == ".keyring"` (internal-only) ⟹ `-EPERM`.

REQ-2: `description`:
- Non-empty, `< 4096` bytes including NUL.
- For `"logon"` type: must contain a `:` separator (`prefix:suffix`).
- Empty / too-long ⟹ `-EINVAL`.

REQ-3: `payload` / `plen`:
- `plen > 1 MiB` (`KEY_PAYLOAD_MAX`) ⟹ `-EINVAL`.
- `payload` may be `NULL` iff `plen == 0` and the type permits (`keyring` requires it).
- Copy-in via `kmemdup_nul` / `vmemdup_user` depending on size; userspace pointer faults ⟹ `-EFAULT`.

REQ-4: Destination keyring resolution:
- Positive serial: `lookup_user_key(keyring, KEY_LOOKUP_CREATE, KEY_NEED_WRITE)`.
- Negative serial: resolve via `KEY_SPEC_*` table; may lazily create thread/process keyrings.
- Not a keyring (wrong type) ⟹ `-ENOTDIR`.

REQ-5: Per-uid quota:
- Charge `1` to `qnkeys`, `plen` to `qnbytes` against `current_user()->user_keys`.
- Exhaust ⟹ `-EDQUOT`.
- Root may bypass via `kernel.keys.root_maxkeys` separate counters.

REQ-6: Key allocation:
- `key_alloc(type, description, current_uid, gid, current_cred, perm, flags, restrict)`.
- `flags`: `KEY_ALLOC_QUOTA_OVERRUN` (root-only), `KEY_ALLOC_NOT_IN_QUOTA`, `KEY_ALLOC_BUILT_IN`, `KEY_ALLOC_BYPASS_RESTRICTION`.
- Default permission: `KEY_POS_ALL | KEY_USR_ALL` (possessor + owner full).

REQ-7: Instantiation + linking: `type->preparse(prep)` validates payload format → kernel prepared representation; `key_instantiate_and_link(key, prep.payload, prep.datalen, dest_keyring, NULL)` performs atomic insert via `__key_link` (may evict expired keys). Existing `(type, description)` in dest: replace if `KEY_USR_WRITE` per `__key_create_or_update`. On failure (`-EKEYREJECTED`), key freed without linking.

REQ-9: Returned serial:
- `key->serial`, a positive 32-bit integer assigned at `key_alloc`.

REQ-10: `logon` write-only:
- After instantiation, `payload` is never readable via `keyctl_read` (`-EOPNOTSUPP`); only consumable by kernel-side consumers.

REQ-11: `keyring` payload:
- `payload` must be `NULL`, `plen == 0`; the new keyring is empty.
- Otherwise `-EINVAL`.

REQ-12: User-namespace scoping:
- `_USER_KEYRING` / `_USER_SESSION_KEYRING` live in `current_user_ns()`.
- Cross-userns key access via possession is denied unless the userns root explicitly grants it.

REQ-13: Persistent keyring:
- `KEY_SPEC_USER_KEYRING` is auto-expiring (default 7 days); each successful add resets the timer.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `type_lookup_total` | INVARIANT | unknown type ⟹ `-EINVAL` |
| `plen_bounded` | INVARIANT | `plen ≤ KEY_PAYLOAD_MAX` |
| `quota_atomic` | INVARIANT | quota check + bump under user_keys lock |
| `logon_unreadable` | INVARIANT | logon-type payload never copied back via keyctl_read |
| `instantiate_or_free` | INVARIANT | failure path frees key without linking |
| `userns_scope` | INVARIANT | `_USER_KEYRING` resolved in `current_user_ns()` |
| `serial_positive` | INVARIANT | returned serial > 0 |

### Layer 2: TLA+

`uapi/add_key.tla`:
- States: `type_resolve`, `desc_validate`, `payload_copy`, `dest_lookup`, `quota`, `alloc`, `preparse`, `instantiate`, `link`, `done`, `failed`.
- Properties:
  - `safety_quota_paired_with_free` — every successful add increments quota; every key drop decrements.
  - `safety_no_partial_link` — failure after `key_alloc` ⟹ key freed before return.
  - `safety_logon_writeonly` — logon-type key never has read-back path.
  - `liveness_terminate` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `add_key` post(ok): `ret > 0` ∧ key is in dest keyring ∧ quota incremented | `Keyring::add_key` |
| `add_key` post(err): no quota delta, no key visible | `Keyring::add_key` |
| `add_key` post(replace): same serial, updated payload, quota delta = `new_plen - old_plen` | `Keyring::add_key` |
| `add_key` post(logon): key flagged write-only, payload not in any read-path | `Keyring::add_key` |

### Layer 4: Verus/Creusot functional

Per-`add_key(2)` / `keyrings(7)` / `keyutils(7)` man pages, `Documentation/security/keys/core.rst`, `tools/testing/selftests/keys/` round-trip, libkeyutils semantic equivalence.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

add_key reinforcement:

- **logon write-only enforcement** — defense against payload exfiltration via `keyctl_read`.
- **Per-uid quota (qnkeys/qnbytes) + KEY_PAYLOAD_MAX (1 MiB); `type` closed-set; description cap (4096)** — defense against keystore exhaustion, key-bomb, type-injection, description-table DoS.
- **`KEY_USR_WRITE` on dest; userns scoping of `_USER_KEYRING`** — defense against unauthorized keyring mutation and cross-userns key leak.

### grsecurity/pax-style reinforcement

- **PaX UDEREF / PAX_USERCOPY_HARDEN on `type`, `description`, `payload` copy-in** — every user pointer goes through `getname` / `vmemdup_user` under SMAP/UDEREF; bounded `vmemdup_user` with `plen ≤ KEY_PAYLOAD_MAX` pre-check; payload buffers come from a whitelisted slab (`kvmalloc` with `__GFP_USERCOPY` for large).
- **PAX_REFCOUNT on `key.refcount` and `keyring.refcount`** — defense against per-refcount-overflow UAF on long-lived shared objects retained across billions of `keyctl_search` calls.
- **Key-quota strict per-uid (RLIMIT_MEMLOCK / kernel.keys.maxkeys / maxbytes)** — payloads count against `RLIMIT_MEMLOCK` in addition to `kernel.keys.maxbytes` (mlocked-equivalent, never swapped). Per-uid kernel-resident objects dual-counted (per-uid sysctl + per-task rlimit) for container-cgroup isolation on either path; lower of (`maxkeys`, `RLIMIT_NPROC`-derived ceiling) applies; per-userns ceiling tunable.
- **CAP_LINUX_IMMUTABLE for protected keys** — `trusted` (TPM-sealed), IMA root-ca `asymmetric`, fscrypt `encrypted` master keys must be created by a uid holding `CAP_LINUX_IMMUTABLE`; subsequent `keyctl_invalidate` / `keyctl_revoke` likewise requires the cap. Protects long-lived trust roots from accidental removal by a compromised uid.
- **GRKERNSEC_HIDESYM on `/proc/keys` and `/proc/key-users`** — under `kernel.kptr_restrict ≥ 2`, restrict to `CAP_SYS_ADMIN` in init_userns; cross-uid entries sanitized; defeats per-uid quota / serial-number leak across pidns boundaries.
- **Payload zeroed on key destruction (`kfree_sensitive` / `kvfree_sensitive`)** — mandatory for `logon`, `trusted`, `encrypted`, `asymmetric` (private), `user`; defeats UAF read of remnant secret data.
- **GRKERNSEC_CHROOT_FINDTASK-style fence on `_SESSION_KEYRING` upgrade** — a chrooted process invoking `add_key(... KEY_SPEC_SESSION_KEYRING)` that would lazily create a new session keyring inheriting outside refs is refused when the chroot policy is set.
- **`logon` type description namespace check** — the `prefix:` portion must match a per-userns allow-list; prevents unprivileged squatting on `cifs.spnego:*` / `rxrpc:*` used by kernel upcalls.
- **Audit log on every `add_key` with sensitive type** — `trusted`, `encrypted`, `asymmetric` (private), `logon`: logged at `LOGLEVEL_INFO` with creator uid + creds + pidns + dest keyring serial + key serial + payload byte count (never payload contents).
- **Per-userns root capped at init-userns root_maxkeys; persistent keyring expiry strictly enforced** — child-userns uid 0 capped at lower of (child `maxkeys`, init `root_maxkeys`); `KEY_SPEC_USER_KEYRING` expiry cannot be extended past userns boot lifetime. `KEY_ALLOC_QUOTA_OVERRUN` rejected from non-init-userns root even with CAP_SYS_RESOURCE.

