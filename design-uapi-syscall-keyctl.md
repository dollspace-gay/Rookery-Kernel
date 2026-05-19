---
title: "Tier-5 syscall: keyctl(2) — syscall 250"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`keyctl(2)` is the multiplexed entry point for the keyring subsystem's per-key, per-keyring, and per-process operations. A single integer `operation` selects one of ~30 commands (read, update, revoke, invalidate, link, unlink, search, describe, chown, chgrp, setperm, set-timeout, instantiate, reject, assume-authority, get-keyring-id, join-session-keyring, restrict-keyring, dh-compute, pkey-* asymmetric ops, watch-key, move, capabilities, etc.). The remaining four `unsigned long` arguments are per-command interpretation slots; the kernel `copy_from_user`s any pointer-arguments under the per-command schema. Critical for: libkeyutils, kerberos / cifs / nfs ticket lifecycle, fscrypt master-key management, PAM session keyring setup, container key-namespace setup.

This Tier-5 covers the userspace ABI of syscall 250 — the dispatcher and the shared per-arg discipline. Per-`KEYCTL_*` semantics that warrant their own treatment may be promoted to a Tier-4 doc.

### Acceptance Criteria

- [ ] AC-1: `keyctl(KEYCTL_GET_KEYRING_ID, KEY_SPEC_SESSION_KEYRING, 0, 0, 0)` returns positive serial.
- [ ] AC-2: `keyctl(KEYCTL_READ, logon_key, buf, buflen, 0)` ⟹ `-EOPNOTSUPP`.
- [ ] AC-3: `keyctl(KEYCTL_UPDATE, key, NULL, 0, 0)` valid for `user` type ⟹ key cleared.
- [ ] AC-4: `keyctl(KEYCTL_LINK, k, kr, 0, 0)` where `kr` already contains `k` indirectly ⟹ `-EDEADLK`.
- [ ] AC-5: `keyctl(KEYCTL_CHOWN, key, OTHER_UID, -1, 0)` from non-CAP_SYS_ADMIN ⟹ `-EPERM`.
- [ ] AC-6: `keyctl(KEYCTL_DESCRIBE, key, buf, buflen, 0)` returns required-buflen even when buflen < required.
- [ ] AC-7: `keyctl(KEYCTL_INSTANTIATE, stub_key, payload, plen, dest)` without possessing auth-key ⟹ `-EPERM`.
- [ ] AC-8: `keyctl(KEYCTL_REJECT, stub_key, timeout, EKEYREJECTED, dest)`: subsequent `request_key` on same key sees `-EKEYREJECTED` for `timeout` seconds.
- [ ] AC-9: `keyctl(KEYCTL_CLEAR, kr, 0, 0, 0)` empties keyring.
- [ ] AC-10: `keyctl(KEYCTL_INVALIDATE, key, 0, 0, 0)` GCs key immediately; no negative cache.
- [ ] AC-11: `keyctl(KEYCTL_DH_COMPUTE, params, buf, buflen, NULL)` returns DH-shared-secret byte count.
- [ ] AC-12: `keyctl(KEYCTL_RESTRICT_KEYRING, kr, "asymmetric", "builtin_trusted", 0)`: subsequent link of non-built-in cert ⟹ `-EKEYREJECTED`.
- [ ] AC-13: `keyctl(0xDEAD, ...)` ⟹ `-EOPNOTSUPP`.
- [ ] AC-14: `keyctl(KEYCTL_CAPABILITIES, buf, buflen, 0, 0)` writes valid bitmap.
- [ ] AC-15: `keyctl(KEYCTL_MOVE, k, src, dst, KEYCTL_MOVE_EXCL)` where `k ∈ dst` ⟹ `-EEXIST`.
- [ ] AC-16: `keyctl(KEYCTL_WATCH_KEY, key, pipe_fd, 0)`: subsequent key-mutation produces a watch_queue event.

### Architecture

Rookery surface in `kernel/security/keys/keyctl.rs`:

```rust
#[syscall(nr = 250, abi = "sysv")]
pub fn sys_keyctl(operation: i32, a2: usize, a3: usize, a4: usize, a5: usize) -> isize {
    KeyCtl::do_keyctl(operation, a2, a3, a4, a5)
}

pub enum KeyCtlOp { /* KEYCTL_* */ }

impl KeyCtl {
    pub fn do_keyctl(op: i32, a2: usize, a3: usize, a4: usize, a5: usize) -> isize {
        let op = KeyCtlOp::try_from(op).map_err(|_| -EOPNOTSUPP)?;
        match op {
            KeyCtlOp::GetKeyringId        => KeyCtl::get_keyring_id(a2 as i32, a3 != 0),
            KeyCtlOp::JoinSessionKeyring  => KeyCtl::join_session_keyring(a2),
            KeyCtlOp::Update              => KeyCtl::update(a2 as i32, a3, a4),
            KeyCtlOp::Revoke              => KeyCtl::revoke(a2 as i32),
            KeyCtlOp::Chown               => KeyCtl::chown(a2 as i32, a3 as u32, a4 as u32),
            KeyCtlOp::Setperm             => KeyCtl::setperm(a2 as i32, a3 as u32),
            KeyCtlOp::Describe            => KeyCtl::describe(a2 as i32, a3, a4),
            KeyCtlOp::Clear               => KeyCtl::clear(a2 as i32),
            KeyCtlOp::Link                => KeyCtl::link(a2 as i32, a3 as i32),
            KeyCtlOp::Unlink              => KeyCtl::unlink(a2 as i32, a3 as i32),
            KeyCtlOp::Search              => KeyCtl::search(a2 as i32, a3, a4, a5 as i32),
            KeyCtlOp::Read                => KeyCtl::read(a2 as i32, a3, a4),
            KeyCtlOp::Instantiate         => KeyCtl::instantiate(a2 as i32, a3, a4, a5 as i32),
            KeyCtlOp::Negate              => KeyCtl::negate(a2 as i32, a3 as u32, a4 as i32),
            KeyCtlOp::Reject              => KeyCtl::reject(a2 as i32, a3 as u32, a4 as i32, a5 as i32),
            KeyCtlOp::DhCompute           => KeyCtl::dh_compute(a2, a3, a4, a5),
            KeyCtlOp::RestrictKeyring     => KeyCtl::restrict_keyring(a2 as i32, a3, a4),
            KeyCtlOp::Move                => KeyCtl::move_key(a2 as i32, a3 as i32, a4 as i32, a5 as u32),
            KeyCtlOp::Capabilities        => KeyCtl::capabilities(a2, a3),
            KeyCtlOp::WatchKey            => KeyCtl::watch_key(a2 as i32, a3 as i32, a4 as i32),
            /* ... ~30 commands ... */
        }
    }
}
```

`KeyCtl::link(key_serial, keyring_serial) -> isize`:
1. let key = Keyring::lookup_user_key(key_serial, 0, KEY_NEED_LINK)?;
2. let keyring = Keyring::lookup_user_key(keyring_serial, KEY_LOOKUP_CREATE, KEY_NEED_WRITE)?;
3. if !keyring.is_keyring() { return Err(-ENOTDIR); }
4. Keyring::check_for_cycle(&key, &keyring)?;     // EDEADLK
5. Keyring::__key_link(&key, &keyring)?;
6. Ok(0)

### Out of Scope

- `add_key.md` Tier-5 — direct key add
- `request_key.md` Tier-5 — upcall-based lookup
- Per-`KEYCTL_*` Tier-4 docs for `KEYCTL_DH_COMPUTE`, `KEYCTL_PKEY_*`, `KEYCTL_RESTRICT_KEYRING` if/when promoted
- `security/keys/keyctl.md` Tier-3 — dispatcher + per-cmd internals
- `security/keys/persistent.md` Tier-3 — persistent keyring
- Implementation code

### signature

```c
long keyctl(int operation, unsigned long arg2, unsigned long arg3,
            unsigned long arg4, unsigned long arg5);
```

Rust ABI shim:

```rust
pub fn sys_keyctl(operation: i32, a2: usize, a3: usize, a4: usize, a5: usize) -> isize;
```

Syscall number: **250** (x86_64).

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `operation` | `i32` | IN | one of `KEYCTL_*`; selects per-command dispatch arm |
| `arg2..arg5` | `usize` | IN/OUT | per-command interpretation — see ABI surface table |

### return

- **Success**: per-command (often `0`; sometimes a serial / byte-count / boolean).
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EOPNOTSUPP` | unknown `operation`; operation valid but type does not support it (e.g. `keyctl_read` on `logon`) |
| `EINVAL` | per-command parameter shape invalid |
| `EFAULT` | any user-pointer copy-in/out fault |
| `EACCES` | per-command permission denied on the target key/keyring |
| `ENOKEY` | per-command target key not found / not valid |
| `EKEYEXPIRED` / `EKEYREVOKED` / `EKEYREJECTED` | target key in non-usable state |
| `EDQUOT` | per-uid quota exhausted on update/instantiate |
| `EPERM` | per-command capability missing (e.g. `KEYCTL_CHOWN` to a different uid without `CAP_SYS_ADMIN`) |
| `ENOMEM` | allocation failure |
| `EDEADLK` | `KEYCTL_LINK` would create a keyring cycle |
| `ELOOP` | search recursion depth exceeded |

### abi surface

Per-command summary (selected; full set in `include/uapi/linux/keyctl.h`):

| `operation` | Value | Parameters | Effect |
|---|---|---|---|
| `GET_KEYRING_ID / JOIN_SESSION_KEYRING` | `0 / 1` | `serial`,`create` / `name *` | resolve `KEY_SPEC_*` / join named (or anon) session keyring |
| `UPDATE / REVOKE / CHOWN / SETPERM` | `2 / 3 / 4 / 5` | `serial`,`payload *`,`plen` / `serial` / `serial`,`uid`,`gid` / `serial`,`perm` | mutate payload / revoke / ownership / perm bits |
| `DESCRIBE / CLEAR` | `6 / 7` | `serial`,`buf *`,`buflen` / `kr` | write `"type;uid;gid;perm;desc"` / unlink all members |
| `LINK / UNLINK / SEARCH / READ` | `8..11` | per-cmd | add/remove link; search nested kr; read payload (never `logon`) |
| `INSTANTIATE / NEGATE / SET_REQKEY_KEYRING / SET_TIMEOUT` | `12..15` | per-cmd | instantiate via auth-key / negate / default-dest selector / expire-at |
| `ASSUME_AUTHORITY / GET_SECURITY / SESSION_TO_PARENT / REJECT` | `16..19` | per-cmd | auth-key cred switch / LSM label / inherit to parent / negative with errno |
| `INSTANTIATE_IOV / INVALIDATE / GET_PERSISTENT` | `20 / 21 / 22` | per-cmd | iovec instantiate / immediate GC / per-uid persistent kr |
| `DH_COMPUTE` | `23` | `params *`,`buf *`,`buflen`,`kdf *` | Diffie-Hellman + optional KDF |
| `PKEY_QUERY / ENCRYPT / DECRYPT / SIGN / VERIFY` | `24..28` | per-cmd | asymmetric ops |
| `RESTRICT_KEYRING / MOVE / CAPABILITIES / WATCH_KEY` | `29..32` | per-cmd | link-restriction / atomic relink / cap bitmap / watch_queue subscribe |

### compatibility contract

REQ-1: Dispatch:
- Unknown `operation` ⟹ `-EOPNOTSUPP`.
- Per-command dispatch arm responsible for full parameter validation; the dispatcher does **not** validate `arg2..arg5` beyond passing them through.

REQ-2: Capability gates (selected):
- `KEYCTL_CHOWN` to a different uid: `CAP_SYS_ADMIN`.
- `KEYCTL_SETPERM` that grants permission to non-owner: requires `CAP_SYS_ADMIN` or owner.
- `KEYCTL_GET_PERSISTENT` for `uid != current_uid`: `CAP_SETUID`.
- `KEYCTL_RESTRICT_KEYRING` on root-trusted keyring: `CAP_SYS_ADMIN`.

REQ-3: Per-key permission discipline:
- `KEYCTL_READ` requires `KEY_NEED_READ`.
- `KEYCTL_UPDATE` / `INSTANTIATE` / `NEGATE` / `REJECT` require `KEY_NEED_WRITE`.
- `KEYCTL_LINK` / `UNLINK` / `CLEAR` / `MOVE` require `KEY_NEED_WRITE` on the target keyring(s) and `KEY_NEED_LINK` on the linked key.
- `KEYCTL_SEARCH` requires `KEY_NEED_SEARCH` on the search keyring and `KEY_NEED_VIEW` on candidates.
- `KEYCTL_DESCRIBE` / `KEYCTL_GET_SECURITY` require `KEY_NEED_VIEW`.

REQ-4: Type opt-outs:
- `logon` type rejects `KEYCTL_READ` (`-EOPNOTSUPP`).
- `asymmetric` private rejects `KEYCTL_READ` (use `KEYCTL_PKEY_*` instead).
- `big_key` allows read but the buffer must accommodate; otherwise byte count is returned and buffer left untouched.

REQ-5: `KEYCTL_LINK` cycle detection:
- BFS over the prospective edge from `key` to `keyring`; if `key` is itself a keyring transitively reachable from `keyring`, refuse with `-EDEADLK`.

REQ-6: `KEYCTL_INSTANTIATE` / `KEYCTL_NEGATE` / `KEYCTL_REJECT`:
- Valid only on a key referenced by an **auth-key** the caller currently possesses (the `KEY_SPEC_REQKEY_AUTH_KEY` mechanism).
- Calls into `request_key.md` upcall path's other half.

REQ-7: `KEYCTL_GET_PERSISTENT`:
- Returns serial of `_persistent` keyring for `uid` in `current_user_ns()`.
- Lazily creates the persistent keyring if missing.
- Resets the persistent-keyring expiry timer.

REQ-8: `KEYCTL_DH_COMPUTE`:
- `params` struct has prime / base / private-key serials.
- Computes `base^private mod prime`; optionally pipes through KDF (`hashname`, `otherinfo`).
- Requires `KEY_NEED_READ` on prime/base/private keys.

REQ-9: `KEYCTL_PKEY_*`:
- Operate on `asymmetric` type keys.
- `params` contains `info` string (`enc=oaep hash=sha256 ...`) and io descriptors.
- Sign requires private-half key with `KEY_NEED_READ`; verify accepts public-half.

REQ-10: `KEYCTL_CAPABILITIES`:
- Writes a bitmap describing kernel feature support (DH, PKEY, MOVE, WATCH, etc.) to allow libkeyutils to gate.

REQ-11: `KEYCTL_RESTRICT_KEYRING`:
- Attaches a per-keyring restriction callback (`type` + `restriction` string) consulted at every subsequent `KEYCTL_LINK` to that keyring.
- Common restrictions: `"asymmetric:builtin_trusted"`, `"asymmetric:key_or_keyring:<serial>:chain"`.

REQ-12: `KEYCTL_MOVE`:
- Atomic unlink-from-from-ring + link-to-to-ring under one keyring-lock-pair.
- `flags & KEYCTL_MOVE_EXCL` ⟹ fail with `-EEXIST` if already in `to_ring`.

REQ-13: `KEYCTL_WATCH_KEY`:
- Subscribes a `watch_queue` (`pipe2(O_NOTIFICATION_PIPE)`) to key events (instantiate, link, unlink, attribute change, revoke, expire).
- `watch_id == -1` ⟹ unsubscribe.

REQ-14: `KEYCTL_SESSION_TO_PARENT`:
- Caller's session keyring becomes the parent process's session keyring at next sysreturn.
- Requires CAP_SYS_ADMIN or matching creds.

REQ-15: Per-arg copy-in size bounds enforced per-command (e.g. payload ≤ `KEY_PAYLOAD_MAX` for UPDATE; buf ≤ `KEY_DESCRIBE_MAX_BUFLEN` for DESCRIBE).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `op_dispatch_total` | INVARIANT | every `KeyCtlOp` has exactly one dispatch arm |
| `unknown_op_eopnotsupp` | INVARIANT | unknown op ⟹ `-EOPNOTSUPP` |
| `perm_check_before_work` | INVARIANT | permission denied ⟹ no mutation |
| `cycle_detection_pre_link` | INVARIANT | `KEYCTL_LINK` never creates a cycle |
| `logon_read_blocked` | INVARIANT | `KEYCTL_READ` on `logon` ⟹ `-EOPNOTSUPP` |
| `auth_key_required_for_inst` | INVARIANT | `KEYCTL_INSTANTIATE` requires possessing auth-key |

### Layer 2: TLA+

`uapi/keyctl.tla`:
- States: per-op dispatch, per-arg copy-in, per-perm-check, per-mutation, per-fd-or-result-write, returned, failed.
- Properties:
  - `safety_no_mutation_without_perm` — every mutating op gated by per-key permission bits.
  - `safety_no_link_cycle` — keyring graph remains a DAG.
  - `safety_auth_key_scope` — instantiate-class ops scoped to auth-key possession.
  - `safety_quota_on_update` — UPDATE/INSTANTIATE charge quota deltas correctly.
  - `liveness_dispatch_terminates` — every keyctl call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_keyctl` post: returns `-EOPNOTSUPP` for unknown op | `KeyCtl::do_keyctl` |
| `KEYCTL_LINK` post: keyring graph DAG-preserved | `KeyCtl::link` |
| `KEYCTL_READ` post: bytes_copied ≤ min(buflen, payload_len) | `KeyCtl::read` |
| `KEYCTL_INSTANTIATE` post: success ⟹ key.flags has KEY_FLAG_INSTANTIATED | `KeyCtl::instantiate` |
| `KEYCTL_RESTRICT_KEYRING` post: restriction is consulted at every subsequent `LINK` | `KeyCtl::restrict_keyring` |

### Layer 4: Verus/Creusot functional

Per-`keyctl(2)` / `keyctl(3)` / `keyrings(7)` man pages, `Documentation/security/keys/core.rst`, `tools/testing/selftests/keys/`, libkeyutils round-trip semantic equivalence.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

keyctl reinforcement:

- **Per-op capability gate** — defense against per-priv-escalation via unguarded subcommand.
- **Per-op key/keyring permission gate** — defense against unauthorized key mutation.
- **`logon` read-block** — defense against payload exfiltration.
- **Cycle detection** — defense against infinite-search via cyclic keyring graph.
- **Auth-key scope** — defense against instantiate-class ops from arbitrary processes.
- **Per-arg copy bounds per-command** — defense against arg-size-confusion.

### grsecurity/pax-style reinforcement

- **PaX UDEREF + PAX_USERCOPY_HARDEN on every per-command user pointer** — `KEYCTL_DESCRIBE / READ / INSTANTIATE / SEARCH / DH_COMPUTE / PKEY_* / RESTRICT_KEYRING / CAPABILITIES` copies go through SMAP/UDEREF-protected helpers (`copy_from_user`, `strndup_user`); bounded copies use whitelisted slabs respecting `KEY_PAYLOAD_MAX` / `KEY_DESCRIBE_MAX_BUFLEN`. The dispatcher forwards `arg2..arg5` as opaque `usize`; per-command code does `access_ok` + bounded copy.
- **PAX_REFCOUNT on every key + keyring refcount touched by keyctl** — `LINK / UNLINK / MOVE / CHOWN / SETPERM` mutate refcounts and perm bits under spinlock; saturation must trap (critical for long-lived `_USER_KEYRING`).
- **Key-quota per-uid (kernel.keys.maxkeys / maxbytes / RLIMIT_MEMLOCK)** — `UPDATE / INSTANTIATE / INSTANTIATE_IOV` pre-check quota deltas; entire update refused atomically rather than partial. Dual-counted with `RLIMIT_MEMLOCK`; per-userns ceiling honored.
- **CAP_LINUX_IMMUTABLE for protected key types** — `REVOKE / INVALIDATE / CLEAR / SETPERM / CHOWN` on `trusted`, IMA-root `asymmetric`, fscrypt `encrypted` v2 master require `CAP_LINUX_IMMUTABLE` regardless of per-key bits. CAP_SYS_ADMIN in init_userns for `RESTRICT_KEYRING` against root-trusted keyring; `GET_PERSISTENT` for uid ≠ current_uid additionally requires CAP_SETUID in init_userns.
- **GRKERNSEC_HIDESYM on `DESCRIBE / GET_SECURITY / READ` output** — under `kernel.kptr_restrict ≥ 2`, cross-userns identities and pointer-derived fields filtered; fid-style info-leak prevention extends to keyctl's reflective output.
- **GRKERNSEC_CHROOT_FINDTASK fence on `JOIN_SESSION_KEYRING / SESSION_TO_PARENT`** — chrooted task creating or upgrading a session keyring that would be inherited outside the chroot is refused; prevents chroot escape via session-keyring inheritance.
- **`DH_COMPUTE` constant-time; `PKEY_*` private-half protected** — modexp/KDF constant-time per operand bit-length; output buffer cleared on failure. Private-half access to IMA-root / fscrypt-master asymmetric keys requires `CAP_LINUX_IMMUTABLE` in addition to `KEY_NEED_READ`.
- **`WATCH_KEY` watch_queue rate-limit; inotify ucount + max_user_watches awareness; fanotify priority bypass** — > N watches/uid logs WARNING and refuses with `-EBUSY`; libkeyutils helpers watching `/etc/krb5.conf` bounded by `fs.inotify.max_user_watches`; fanotify-perm groups marking `/proc/keys` cannot stall `keyctl` (bypass intentional and audited).
- **Audit log on every cap-gated op; `MOVE` atomicity; `ASSUME_AUTHORITY` scope** — `CHOWN / SETPERM (non-owner) / RESTRICT_KEYRING / SESSION_TO_PARENT / JOIN_SESSION_KEYRING (anon-create) / INSTANTIATE (cross-uid via auth-key)` logged at `LOGLEVEL_INFO`. `MOVE` unlink-from + link-to runs under double keyring lock (Verus post-condition). `ASSUME_AUTHORITY` rejected outside upcall helper context.

