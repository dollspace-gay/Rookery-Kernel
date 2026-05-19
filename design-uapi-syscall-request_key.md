---
title: "Tier-5 syscall: request_key(2) — syscall 249"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`request_key(2)` searches the caller's keyring hierarchy (thread → process → session → user-session → user) for a key matching `(type, description)`. If found, the matching key's serial is returned and the key is optionally linked into `dest_keyring`. If not found, the kernel performs an upcall: it forks a `/sbin/request-key` (or per-`type` configured helper) under a synthetic auth-key process, hands the helper the `callout_info` blob, waits for the helper to invoke `keyctl_instantiate` to populate the answer, then returns. The upcall mechanism is what wires kerberos / dns / cifs userspace daemons into the kernel's mount-time and rpc-time secret-acquisition paths. Critical for: NFSv4/CIFS auth, AFS rxrpc, dns resolver, fscrypt v2 master-key fetch, mount-time on-demand secret materialization.

This Tier-5 covers the userspace ABI of syscall 249. Mutation lives in `add_key.md`, multiplex ops in `keyctl.md`.

### Acceptance Criteria

- [ ] AC-1: Pre-existing matching key in session keyring ⟹ returned serial; no upcall.
- [ ] AC-2: No match, `callout_info == NULL` ⟹ `-ENOKEY`; no upcall.
- [ ] AC-3: No match, `callout_info != NULL`, helper succeeds ⟹ instantiated key serial.
- [ ] AC-4: Helper calls `keyctl_reject(... -EKEYREJECTED)` ⟹ caller sees `-EKEYREJECTED`.
- [ ] AC-5: Helper exits without instantiating ⟹ `-ENOKEY`.
- [ ] AC-6: Helper timeout ⟹ negative instantiation, caller sees `-ENOKEY`.
- [ ] AC-7: Signal during wait ⟹ `-EINTR`.
- [ ] AC-8: Recursion depth exceeded ⟹ `-ENOKEY` (baseline) / `-ELOOP` (hardened).
- [ ] AC-9: `dest_keyring` not writable ⟹ `-EACCES`.
- [ ] AC-10: `callout_info` longer than `PAGE_SIZE-1` ⟹ `-EINVAL`.
- [ ] AC-11: Unknown `type` ⟹ `-EINVAL`.
- [ ] AC-12: Caller quota exhausted during upcall ⟹ `-EDQUOT` on completion.
- [ ] AC-13: Match found in thread keyring takes precedence over session keyring.
- [ ] AC-14: Helper missing ⟹ `-EAGAIN`.
- [ ] AC-15: Negatively-cached key respected on second call within negative_timeout window.

### Architecture

Rookery surface in `kernel/security/keys/request.rs`:

```rust
pub struct AuthKey {
    pub stub_key:     KeyRef,        // the uninstantiated key being satisfied
    pub caller_creds: Cred,          // identity helper acts under
    pub dest_keyring: KeyRef,
    pub callout_info: KVec<u8>,
}

pub struct UpcallCtx {
    pub auth_key:     KeyRef,
    pub helper_path:  &'static str,  // typically "/sbin/request-key"
    pub timeout:      Ktime,
    pub recursion:    u32,
}
```

`Keyring::request_key(type_uptr, desc_uptr, callout_uptr, dest_serial) -> isize`:
1. let type_name = getname_kernel(type_uptr, 32)?;
2. let type_ = KeyType::lookup(&type_name).ok_or(-EINVAL)?;
3. let desc = getname_kernel(desc_uptr, 4096)?;
4. type_.validate_description(&desc)?;
5. let callout = if !callout_uptr.is_null() {
     Some(strndup_user(callout_uptr, PAGE_SIZE - 1)?)
   } else { None };
6. let dest = if dest_serial == 0 {
     type_.default_dest_keyring(current_creds())?
   } else {
     Keyring::lookup_user_key(dest_serial, KEY_LOOKUP_CREATE, KEY_NEED_WRITE)?
   };
7. /* Search */
8. if let Some(key) = Keyring::search_process_keyrings(&type_, &desc, current_creds())? {
9.   Keyring::link_into(&key, &dest)?;
10.  return Ok(key.serial as isize);
11. }
12. /* No match. If no callout requested, fail. */
13. let callout = callout.ok_or(-ENOKEY)?;
14. /* Upcall path */
15. let stub = Key::alloc_uninstantiated(&type_, &desc, current_creds(), KEY_ALLOC_QUOTA)?;
16. let auth = AuthKey::create_for(&stub, &dest, callout)?;
17. let ctx = UpcallCtx { auth_key: auth.into_key(), helper_path: "/sbin/request-key", timeout: type_.request_timeout, recursion: current_request_depth() + 1 };
18. if ctx.recursion > MAX_REQUEST_DEPTH { return Err(-ENOKEY); }
19. Keyring::call_request_helper(&ctx)?;                  // EAGAIN if helper missing
20. let outcome = Keyring::wait_for_instantiation(&stub, ctx.timeout)?;  // EINTR
21. match outcome {
22.   Instantiated => Ok(stub.serial as isize),
23.   NegativelyInstantiated(errno) => Err(errno),         // EKEYREJECTED / ENOKEY
24. }

### Out of Scope

- `add_key.md` Tier-5 — direct key add
- `keyctl.md` Tier-5 — multiplex key ops (`keyctl_instantiate`, `keyctl_reject`, ...)
- `security/keys/request-key.md` Tier-3 — full upcall infrastructure, auth-key allocation, helper lifecycle
- `security/keys/process_keys.md` Tier-3 — process-keyring search
- Implementation code

### signature

```c
key_serial_t request_key(const char *type, const char *description,
                         const char *callout_info,
                         key_serial_t dest_keyring);
```

Rust ABI shim:

```rust
pub fn sys_request_key(
    type_:         UserPtr<u8>,
    description:   UserPtr<u8>,
    callout_info:  UserPtr<u8>,           // NULL = search only, no upcall
    dest_keyring:  i32,
) -> isize;
```

Syscall number: **249** (x86_64).

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `type` | `const char *` | IN | NUL-terminated type name; resolved against `key_types_list` |
| `description` | `const char *` | IN | NUL-terminated description; for `"logon"` must contain `:` |
| `callout_info` | `const char *` | IN | NUL-terminated blob passed to `/sbin/request-key` on upcall; `NULL` ⟹ no upcall, fail with `-ENOKEY` if not found |
| `dest_keyring` | `i32` (`key_serial_t`) | IN | keyring into which the resolved key is linked on success; `0` ⟹ default (typically the caller's session keyring) |

### return

- **Success**: positive `key_serial_t` of the (found or instantiated) key.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `ENOKEY` | not found in any searched keyring AND `callout_info == NULL`; or upcall ran but helper called `keyctl_reject` |
| `EKEYEXPIRED` | found key has expired (and not re-instantiable via upcall) |
| `EKEYREVOKED` | found key revoked |
| `EKEYREJECTED` | helper rejected the request (`keyctl_reject`) |
| `EINVAL` | unknown `type`; empty / too-long description |
| `EACCES` | caller lacks `KEY_NEED_SEARCH` on a keyring that would otherwise satisfy; or `dest_keyring` not writable |
| `EDQUOT` | per-uid quota exceeded during upcall-driven instantiation |
| `EFAULT` | any user-pointer copy-in fault |
| `ENOMEM` | allocation failure (key, auth-key, upcall infrastructure) |
| `EAGAIN` | upcall helper not configured (`/sbin/request-key` missing) or fork failure |
| `EINTR` | wait for upcall interrupted by signal |

### abi surface

The syscall composes with three subsystems:
- the **search** path (`search_process_keyrings` → `keyring_search_rcu`),
- the **upcall** path (`call_usermodehelper(/sbin/request-key, ...)`),
- the **auth-key** (`KEY_SPEC_REQKEY_AUTH_KEY`) construct that lets the helper instantiate the answer with the caller's identity.

`callout_info` shape (passed to helper via env / argv):
- `argv[0] = "/sbin/request-key"`
- `argv[1] = "create"`
- `argv[2] = key_serial of waiting key`
- `argv[3] = caller uid`
- `argv[4] = caller gid`
- `argv[5] = caller threadring serial`
- `argv[6] = caller processring serial`
- `argv[7] = caller sessionring serial`
- env: `KEY_TYPE`, `KEY_DESC`, `KEY_CALLOUT_INFO`

### compatibility contract

REQ-1: `type` resolved against `key_types_list`; unknown ⟹ `-EINVAL`.

REQ-2: `description` validation per `add_key` REQ-2.

REQ-3: Search order (RCU, per-keyring permission check):
- Thread keyring (`KEY_SPEC_THREAD_KEYRING`).
- Process keyring (`KEY_SPEC_PROCESS_KEYRING`).
- Session keyring (`KEY_SPEC_SESSION_KEYRING`).
- Per-uid user-session keyring.
- Per-uid user keyring.
- Per-keyring nested search (transitive via `KEY_TYPE_KEYRING` links) with cycle detection.

REQ-4: For each candidate key, check `KEY_NEED_SEARCH` at every keyring in the chain and `KEY_NEED_VIEW` on the candidate itself. Failures cause the candidate to be skipped silently (not error-returned).

REQ-5: First matching key (by `(type, description)` exact match):
- If non-expired, non-revoked, instantiated ⟹ return.
- If negatively instantiated (a prior `keyctl_reject`): treated as no-match; continue search; if no further match and `callout_info != NULL`, upcall.

REQ-6: `dest_keyring`:
- `0` ⟹ default destination per `type->default_dest_keyring` (typically session keyring of caller).
- Positive serial ⟹ `lookup_user_key(dest, KEY_LOOKUP_CREATE, KEY_NEED_WRITE)`; not writable ⟹ `-EACCES`.
- Resolved key linked into dest only if it wasn't already a link reachable from dest.

REQ-7: Upcall:
- Triggered iff search returns no instantiated match AND `callout_info != NULL`.
- Allocate an uninstantiated stub key.
- Create an **auth-key**: `request_key_auth` type, bound to the stub key, possessable only by the helper process.
- The helper's session-keyring contains the auth-key; that lets the helper call `keyctl_instantiate(stub_key, payload, dest)` using the caller's identity.
- `call_usermodehelper("/sbin/request-key", argv, env)` with `UMH_WAIT_PROC`.
- On helper exit: poll `stub_key.flags` for `KEY_FLAG_INSTANTIATED` or `KEY_FLAG_NEGATIVE`.
- Timeout (per `key_type->request_key_timeout` or default `300s`) ⟹ negative instantiation.

REQ-8: `callout_info` length capped at `PAGE_SIZE - 1`; longer ⟹ `-EINVAL`.

REQ-9: Auth-key lifecycle:
- Created with `KEY_ALLOC_NOT_IN_QUOTA` (does not count against caller quota).
- Revoked on helper completion (success or failure).
- Helper cannot escape the auth-key's scope (its session-keyring is reset to contain only the auth-key + the caller's session as anchor).

REQ-10: Recursive request:
- If the helper itself calls `request_key`, the recursion depth is bounded (default `8`); exceeded ⟹ `-ELOOP` (returned as `-ENOKEY` to caller in baseline).

REQ-11: Per-uid quota during upcall:
- The newly-instantiated key is charged against the *caller's* uid, not the helper's.

REQ-12: User-namespace scope:
- Search path honors `current_user_ns()` boundaries.
- Helper runs in init userns by default (configurable per type).

REQ-13: Cancellation:
- Caller waits in `TASK_INTERRUPTIBLE`; signal delivery ⟹ `-EINTR`.
- The upcall continues independently; a later `request_key` may reap the result via search.

REQ-14: Negative cache:
- A negatively-instantiated key persists for `negative_timeout` (default `60s`) to avoid upcall storms.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `search_before_upcall` | INVARIANT | upcall only after search returns no instantiated match |
| `authkey_scope` | INVARIANT | helper's keyring scope ⊆ caller's at upcall start |
| `callout_size_bounded` | INVARIANT | `callout_info` size ≤ PAGE_SIZE - 1 |
| `recursion_bounded` | INVARIANT | request depth ≤ MAX_REQUEST_DEPTH |
| `quota_against_caller` | INVARIANT | instantiated-key quota charged to caller, not helper |
| `negative_cache_respected` | INVARIANT | negative-cached key returned (`-ENOKEY`) without re-upcall |

### Layer 2: TLA+

`uapi/request_key.tla`:
- States: `validating`, `searching`, `found`, `linking`, `upcall_alloc_stub`, `upcall_alloc_auth`, `helper_running`, `wait_inst`, `done`, `failed`.
- Properties:
  - `safety_helper_uses_caller_creds` — helper's `keyctl_instantiate` runs as caller's uid.
  - `safety_authkey_revoked_post_helper` — auth-key revoked on helper exit (success or failure).
  - `safety_no_partial_link` — failure ⟹ no link to dest_keyring.
  - `liveness_terminate_or_signal` — either bounded by helper timeout or caller signal.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `request_key` post(ok): `ret > 0` ∧ key found-or-instantiated ∧ linked into dest | `Keyring::request_key` |
| `request_key` post(err): no caller-side quota delta on search-only fail | `Keyring::request_key` |
| `request_key` post(upcall-fail): stub key negatively-instantiated, cached for negative_timeout | `Keyring::request_key` |
| `request_key` post(no-callout, no-match): `-ENOKEY`, no allocation | `Keyring::request_key` |

### Layer 4: Verus/Creusot functional

Per-`request_key(2)` / `request-key(8)` / `request-key.conf(5)` man pages, `Documentation/security/keys/request-key.rst`, `tools/testing/selftests/keys/` round-trip, libkeyutils + keyutils `request-key` daemon semantic equivalence.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

request_key reinforcement:

- **Search-then-upcall ordering** — defense against denying cached availability and forcing helper churn.
- **Auth-key scope minimization** — defense against helper privilege accumulation.
- **`callout_info` size cap** — defense against helper argv inflation.
- **Recursion depth cap** — defense against helper-of-helper loops.
- **Quota charged to caller** — defense against helper-uid quota abuse.
- **Negative cache** — defense against upcall-storm DoS.
- **Helper timeout** — defense against indefinite waiter blockage.

### grsecurity/pax-style reinforcement

- **PaX UDEREF on `type`, `description`, `callout_info`** — every user pointer goes through `getname` / `strndup_user` under SMAP/UDEREF; the kernel never directly dereferences a user pointer in the `request_key` body. `callout_info` copy uses bounded `strndup_user(PAGE_SIZE - 1)`.
- **PAX_USERCOPY_HARDEN on callout_info slab** — copy goes through a whitelisted slab; refuse copies that would cross slab object boundaries.
- **PAX_REFCOUNT on auth-key and stub-key refcounts** — defense against per-refcount-overflow UAF; the auth-key in particular is short-lived but carries caller creds, so a saturation would let a malicious helper retain caller identity past revocation.
- **Key-quota per-uid (RLIMIT_MEMLOCK / kernel.keys.maxkeys / maxbytes)** — upcall-instantiated keys count against the caller's quota; under hardened policy, quota check happens **twice** — once at stub-alloc to refuse early, and once at `keyctl_instantiate` time in case the helper returned a much larger payload than callout suggested. Per-uid quota is dual-counted with `RLIMIT_MEMLOCK` so payloads cannot be silently swapped, and per-userns ceiling must be tunable.
- **CAP_LINUX_IMMUTABLE for protected key types via upcall** — if the requested type is one of the protected-key types (`trusted`, `asymmetric` private half used as IMA root, `encrypted` for fscrypt v2 master), the **helper** must run as a uid carrying `CAP_LINUX_IMMUTABLE`; the upcall configuration in `/etc/request-key.conf` is validated against this constraint at registration.
- **GRKERNSEC_HIDESYM on `/proc/keys` during upcall window** — between stub-key creation and instantiation/negative-cache, the stub key is visible in `/proc/keys` as `(uninstantiated)`; under hardened policy, restrict that view to `CAP_SYS_ADMIN` to avoid leaking the existence of pending secret-acquisition operations to unprivileged observers.
- **GRKERNSEC_CHROOT_FINDTASK-style fence** — when caller is chrooted and `grsec_chroot_findtask` is set, refuse upcall; the helper would otherwise run in init-userns context outside the chroot, expanding the chroot's effective filesystem reach via the helper's behavior.
- **Helper auth-key session-keyring strict minimization** — the helper's session keyring contains **only** the auth-key and the caller's session-keyring as a single anchored read-only reference; the helper cannot link arbitrary keys back into the caller's keyring beyond the stub. Defense against helper-escape via keyring-substitution attacks.
- **Helper timeout bounded per-type with audit** — every helper invocation is logged at `LOGLEVEL_INFO` with type, description hash, caller uid, helper pid, exit status, and elapsed time. Helpers exceeding `2× nominal timeout` are SIGKILL'd and the event is logged at `LOGLEVEL_WARNING`.
- **inotify ucount + max_user_watches awareness** — the request-key helper traditionally watches `/etc/request-key.conf` and per-type configs via inotify; under hardened policy the helper's inotify-watch count is bounded by `fs.inotify.max_user_watches` for the helper's uid (usually root, but per-namespace ceiling applies). Prevents a malicious config-injection from causing the helper to allocate watches on every PATH component.
- **Recursion depth strict (8 → 4 under hardened policy)** — `MAX_REQUEST_DEPTH` lowered; transitive upcalls beyond depth 4 are refused with explicit `-ELOOP` (not silently `-ENOKEY`); every refused recursion is audit-logged.
- **fanotify priority distinction** — if a fanotify-permission group has marks on `/etc/request-key.conf` or `/sbin/request-key`, its priority does **not** allow it to gate the upcall (the upcall path bypasses fanotify-perm to avoid deadlock on credential-acquisition for the very mount the listener may also be watching). `GRKERNSEC_HIDESYM` on fid info-leak prevents the listener from learning the configured helper path beyond what is exported by sysctl.
- **Negative cache audit** — every negative instantiation logged at `LOGLEVEL_INFO`; defends against silent privilege denials in production.
- **Helper-side reject reason audit** — `keyctl_reject` errno values logged with reason; defends against helper masquerading benign denials as transient.
- **Audit log on every cross-userns upcall** — when caller userns ≠ helper userns, log creator uid + caller userns ID + helper userns ID + key type.

