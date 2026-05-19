---
title: "Tier-5 syscall: landlock_create_ruleset(2) — syscall 444"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`landlock_create_ruleset(2)` is the first of three syscalls (444 / 445 / 446) that compose the Landlock unprivileged sandboxing facility. It allocates a kernel-resident `landlock_ruleset` — a write-once-then-sealed bundle of (access-right-handled-set, rule-set) — and returns a file descriptor. The caller subsequently uses `landlock_add_rule(2)` (445) to attach individual `(object, access-mask)` rules to the ruleset, then `landlock_restrict_self(2)` (446) to install the ruleset onto the current thread (and inherited by all descendants and exec children). A separate ABI-version-probe mode (`flags & LANDLOCK_CREATE_RULESET_VERSION`) returns the highest ABI version the running kernel implements, so userspace can discover which access-right bits are honored. Critical for: defense-in-depth in browsers, language runtimes, container init, build sandboxes, and any process that wishes to drop ambient authority over the filesystem and network without requiring CAP_SYS_ADMIN.

This Tier-5 covers the userspace ABI of syscall 444. Rule attachment lives in `landlock_add_rule.md` (445) and ruleset installation in `landlock_restrict_self.md` (446).

### Acceptance Criteria

- [ ] AC-1: `landlock_create_ruleset(&attr, sizeof attr, 0)` with `handled_access_fs = LANDLOCK_ACCESS_FS_READ_FILE` returns positive fd.
- [ ] AC-2: Returned fd has `FD_CLOEXEC` set unconditionally; shows as `anon_inode:[landlock-ruleset]` in `/proc/<pid>/fd/`.
- [ ] AC-3: `landlock_create_ruleset(NULL, 0, LANDLOCK_CREATE_RULESET_VERSION)` returns `≥ 5` (baseline 7.1.0-rc2 ABI version).
- [ ] AC-4: `(&attr, sizeof attr, LANDLOCK_CREATE_RULESET_VERSION)` ⟹ `-EINVAL` (attr conflict with version probe).
- [ ] AC-5: `flags = 0x4` (unknown bit) ⟹ `-EINVAL`; `size = 0` (creation path) ⟹ `-EINVAL`.
- [ ] AC-6: `attr.handled_access_fs = 0x80000000` (unknown bit) ⟹ `-EINVAL`.
- [ ] AC-7: `size = sizeof(attr) + 4` with non-zero tail ⟹ `-E2BIG`; zero tail succeeds.
- [ ] AC-8: `size = sizeof(attr) / 2` succeeds with tail zero-extended.
- [ ] AC-9: `attr` NULL with size > 0 ⟹ `-EFAULT`.
- [ ] AC-10: Multiple `create_ruleset` calls in a single process succeed independently; `close(fd)` GCs ruleset when last reference dropped.
- [ ] AC-11: `handled_access_net = LANDLOCK_ACCESS_NET_CONNECT_TCP` on ABI < 4 (simulated) ⟹ `-EINVAL`.

### Architecture

Rookery surface in `kernel/security/landlock/syscall.rs`:

```rust
pub const LANDLOCK_ABI_VERSION: u32 = 5;     // matches baseline 7.1.0-rc2

bitflags! {
    pub struct LandlockCreateFlags: u32 {
        const VERSION = 0x1;
    }
}

#[repr(C)]
#[derive(Clone, Copy, Default)]
pub struct LandlockRulesetAttr {
    pub handled_access_fs:  u64,
    pub handled_access_net: u64,
    pub scoped:             u64,
}

pub struct LandlockRuleset {
    pub abi_version:        u32,
    pub handled_access_fs:  u64,
    pub handled_access_net: u64,
    pub scoped:             u64,
    pub rules_fs:           RwLockedBTree<FsPathKey, u64>,
    pub rules_net:          RwLockedBTree<NetPortKey, u64>,
    pub installed_count:    AtomicU32,
}
```

`Landlock::create_ruleset(attr_uptr, size, flags) -> isize`:
1. let f = LandlockCreateFlags::from_bits(flags).ok_or(-EINVAL)?;
2. if f.contains(LandlockCreateFlags::VERSION) {
     if !attr_uptr.is_null() || size != 0 { return Err(-EINVAL); }
     return Ok(LANDLOCK_ABI_VERSION as isize);
   }
3. if size == 0 { return Err(-EINVAL); }
4. let attr = Landlock::copy_attr_from_user(attr_uptr, size)?;
5. Landlock::validate_handled_access(&attr)?;       // EINVAL on unsupported bits
6. if attr.handled_access_fs == 0 && attr.handled_access_net == 0 && attr.scoped == 0 {
     return Err(-EINVAL);
   }
7. let ruleset = Arc::new(LandlockRuleset {
     abi_version: LANDLOCK_ABI_VERSION,
     handled_access_fs: attr.handled_access_fs,
     handled_access_net: attr.handled_access_net,
     scoped: attr.scoped,
     rules_fs: RwLockedBTree::new(),
     rules_net: RwLockedBTree::new(),
     installed_count: AtomicU32::new(0),
   });
8. anon_inode_getfd("[landlock-ruleset]", &LANDLOCK_RULESET_FOPS, ruleset, O_RDONLY | O_CLOEXEC).map(|fd| fd as isize)

`Landlock::copy_attr_from_user(uptr, size) -> Result<LandlockRulesetAttr>`:
1. const K: usize = size_of::<LandlockRulesetAttr>();
2. let n = min(size, K);
3. let mut kattr = LandlockRulesetAttr::default();
4. unsafe { uptr.copy_in_partial(&mut kattr, n)?; }
5. if size > K {
6.   let tail = size - K;
7.   let mut probe = [0u8; 64];
8.   for off in step_by(tail, 64) {
9.     let n = min(64, tail - off);
10.    uptr.add(K + off).copy_in_partial(&mut probe[..n])?;
11.    if probe[..n].iter().any(|b| *b != 0) { return Err(-E2BIG); }
12.  }
13. }
14. Ok(kattr)

### Out of Scope

- `landlock_add_rule.md` Tier-5 — rule attachment (syscall 445)
- `landlock_restrict_self.md` Tier-5 — ruleset installation onto thread
- `security/landlock/ruleset.md` Tier-3 — ruleset object internals, rule btree
- `security/landlock/fs.md` Tier-3 — per-LSM-hook fs enforcement
- `security/landlock/net.md` Tier-3 — net enforcement
- Implementation code

### signature

```c
int landlock_create_ruleset(const struct landlock_ruleset_attr *attr,
                            size_t size,
                            uint32_t flags);
```

Rust ABI shim:

```rust
pub fn sys_landlock_create_ruleset(
    attr:  UserPtr<LandlockRulesetAttr>,
    size:  usize,
    flags: u32,
) -> isize;
```

Syscall number: **444** (x86_64).

```c
struct landlock_ruleset_attr {
    __u64 handled_access_fs;   /* bitmask of LANDLOCK_ACCESS_FS_* */
    __u64 handled_access_net;  /* bitmask of LANDLOCK_ACCESS_NET_* */
    __u64 scoped;              /* bitmask of LANDLOCK_SCOPE_* (6.7+) */
};
```

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `attr` | `const struct landlock_ruleset_attr *` | IN | per-ruleset handled-access bundle; ignored when `flags & LANDLOCK_CREATE_RULESET_VERSION` |
| `size` | `size_t` | IN | caller-declared `sizeof(*attr)`; kernel uses `min(size, sizeof(landlock_ruleset_attr))` for copy, validates trailing tail is zero |
| `flags` | `u32` | IN | `0` for ruleset-creation; `LANDLOCK_CREATE_RULESET_VERSION` for ABI-version-probe |

### return

- **Success (creation)**: non-negative file descriptor referencing the new ruleset.
- **Success (`LANDLOCK_CREATE_RULESET_VERSION`)**: positive integer = highest supported ABI version (`1`, `2`, `3`, `4`, ...).
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EOPNOTSUPP` | Landlock disabled at boot (`lsm=` lacks `landlock`) or `CONFIG_SECURITY_LANDLOCK=n` |
| `EINVAL` | unknown bit in `flags`; both `LANDLOCK_CREATE_RULESET_VERSION` and a non-NULL `attr` provided; `attr` provides bits in `handled_access_fs` / `handled_access_net` / `scoped` not supported by this ABI version; `size = 0` or `size > sizeof(landlock_ruleset_attr)` with non-zero tail |
| `E2BIG` | `size > sizeof(landlock_ruleset_attr)` AND tail is non-zero |
| `EFAULT` | `attr` user pointer faults during `copy_from_user` |
| `EMFILE` | per-process fd table full (`RLIMIT_NOFILE`) |
| `ENFILE` | system-wide fd table full |
| `ENOMEM` | cannot allocate `landlock_ruleset` |
| `EPERM` | `kernel.unprivileged_userns_clone = 0` semantics applied to Landlock disabled-for-unprivileged mode (hardened policy) |
| `ENOSYS` | `CONFIG_SECURITY_LANDLOCK` disabled |

### abi surface

### Filesystem access rights (`handled_access_fs`)

| Constant | Value | Purpose |
|---|---|---|
| `EXECUTE / WRITE_FILE / READ_FILE / READ_DIR` | `0x1 / 0x2 / 0x4 / 0x8` | mmap PROT_EXEC or execve / open-write / open-read / readdir |
| `REMOVE_DIR / REMOVE_FILE` | `0x10 / 0x20` | rmdir / unlink |
| `MAKE_CHAR / MAKE_DIR / MAKE_REG / MAKE_SOCK` | `0x40..0x200` | mknod char / mkdir / create reg / bind unix |
| `MAKE_FIFO / MAKE_BLOCK / MAKE_SYM` | `0x400 / 0x800 / 0x1000` | mkfifo / mknod block / symlink |
| `REFER (v2+) / TRUNCATE (v3+) / IOCTL_DEV (v5+)` | `0x2000 / 0x4000 / 0x8000` | rename/link across paths / truncate / device ioctl |

### Network access rights (`handled_access_net`, ABI v4+)

`LANDLOCK_ACCESS_NET_BIND_TCP = 0x1`, `LANDLOCK_ACCESS_NET_CONNECT_TCP = 0x2`.

### Scoped rights (`scoped`, ABI v6+)

`LANDLOCK_SCOPE_ABSTRACT_UNIX_SOCKET = 0x1`, `LANDLOCK_SCOPE_SIGNAL = 0x2` (restrict kill / pidfd_send_signal across ruleset boundary).

### Flags

`LANDLOCK_CREATE_RULESET_VERSION = 0x1` (probe-only: return highest supported ABI version).

### compatibility contract

REQ-1: `flags` subset of `{LANDLOCK_CREATE_RULESET_VERSION}`; else `-EINVAL`. If `VERSION` is set: `attr` MUST be NULL and `size` MUST be `0`; else `-EINVAL`. Returns highest supported ABI version, never an fd.

REQ-2: `size` discipline (creation path): `size == 0` ⟹ `-EINVAL`. `size > sizeof(attr)` ⟹ verify trailing bytes are zero (forward-compat probe), else `-E2BIG`. `size < sizeof(attr)` ⟹ tail zero-extended kernel-side (backward-compat).

REQ-3: Each of `handled_access_fs / net / scoped` must be subset of kernel's supported bit-set for negotiated ABI version; bits outside ⟹ `-EINVAL`. At least one bit total MUST be set; all-zero ruleset rejected with `-EINVAL` (Rookery unifies legacy `-ENOMSG`).

REQ-4: Returned fd (creation): `O_RDONLY`-equivalent; only meaningful ops are `landlock_add_rule`, `landlock_restrict_self`, `close`. Implicit `O_CLOEXEC` always (distinct from inotify/fanotify opt-in). Visible as `anon_inode:[landlock-ruleset]`.

REQ-5: Per-process fd accounting: consumes one `RLIMIT_NOFILE` slot.

REQ-6: Lifetime: fd holds the ruleset; `landlock_add_rule` accumulates rules; after `landlock_restrict_self`, the kernel takes an additional reference for the thread cred, so the fd may be closed at any point without affecting the installed restriction. Ruleset freed only when fd and all thread-creds release.

REQ-7: ABI versioning: VERSION probe returns highest supported (baseline 7.1.0-rc2 = ABI v5). Userspace MUST probe before assuming `_REFER / _TRUNCATE / _IOCTL_DEV` / network / scoped bits.

REQ-8: User-namespace:
- Landlock is **unprivileged** by design; no `CAP_SYS_ADMIN` required. Per `current_user_ns()` semantics, however, the kernel may apply hardened-policy gating (see Grsec section).

REQ-9: Sealing:
- After `landlock_restrict_self(fd, ...)` consumes the ruleset, the ruleset is **immutable** for subsequent installations; further `landlock_add_rule(fd, ...)` calls would still mutate the in-fd ruleset object, but a new `landlock_restrict_self(fd, ...)` simply installs the current (extended) ruleset alongside the prior installation — restrictions only ever accumulate, never relax (REQ-10).

REQ-10: Monotonic restriction:
- The thread's effective Landlock rule-set is the **intersection** of all installed rulesets — adding restrictions can never grant rights. This invariant is enforced at `landlock_restrict_self` time, not at create time, but is documented here because it shapes the semantics of `handled_access_*` (a higher-handled-access ruleset is a tighter ruleset).

REQ-11: Compose with seccomp:
- Landlock and seccomp are orthogonal and freely composable. Both are designed for unprivileged use.

REQ-12: No global state mutation at create time:
- `landlock_create_ruleset` allocates only a per-fd object; no thread-cred mutation, no global table, no per-userns change. Pure side-effect-free except for fd-install.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_subset` | INVARIANT | `flags & !LANDLOCK_CREATE_RULESET_VERSION != 0 ⟹ -EINVAL` |
| `version_path_attr_null` | INVARIANT | `VERSION` set ⟹ `attr == NULL ∧ size == 0` |
| `attr_tail_zero` | INVARIANT | `size > K` with non-zero tail ⟹ `-E2BIG` |
| `handled_subset_of_abi` | INVARIANT | `attr.handled_access_*` ⊆ ABI-supported bits |
| `at_least_one_handled_bit` | INVARIANT | total handled-bits-set ≥ 1 for creation path |
| `fd_always_cloexec` | INVARIANT | returned fd has `FD_CLOEXEC` |
| `fd_install_atomic` | INVARIANT | fd visible iff ruleset fully initialized |

### Layer 2: TLA+

`uapi/landlock_create_ruleset.tla`:
- States: `validating`, `version_probe`, `attr_copy`, `attr_validate`, `allocating`, `installing_fd`, `returned`, `failed`.
- Properties:
  - `safety_version_no_fd_install` — VERSION mode never installs an fd.
  - `safety_no_partial_install` — fd visible iff ruleset fully initialized.
  - `safety_attr_tail_zero` — tail validation invariant.
  - `safety_no_side_effects_on_error` — failure ⟹ no fd, no kernel allocation persisted.
  - `liveness_terminate` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `create_ruleset` post(ok, creation): `0 ≤ ret` ∧ ruleset state matches `attr` ∧ rules empty | `Landlock::create_ruleset` |
| `create_ruleset` post(ok, version): `ret == LANDLOCK_ABI_VERSION` | `Landlock::create_ruleset` |
| `create_ruleset` post(err): no kernel-side state change | `Landlock::create_ruleset` |
| `copy_attr_from_user` post: kattr matches user attr (zero-padded) | `Landlock::copy_attr_from_user` |

### Layer 4: Verus/Creusot functional

Per-`landlock_create_ruleset(2)` / `landlock(7)` man pages, `Documentation/userspace-api/landlock.rst`, `tools/testing/selftests/landlock/` round-trip, libpsx / sample-sandboxer semantic equivalence.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

landlock_create_ruleset reinforcement:

- **Forced CLOEXEC on ruleset fd** — defense against fd leaked across exec to a child that could re-extend with looser intent.
- **Forward-compat tail-zero probe; `handled_access_*` ⊆ ABI-supported bits; at-least-one-bit minimum** — defense against extension-field smuggling, silently-honored unknown bits, and meaningless rulesets.
- **No state mutation at create time; per-process fd-table accounting; pure-allocation semantics** — defense against pre-install side effects, fd-flood via `RLIMIT_NOFILE`, and partial-failure cleanup bugs.

### grsecurity/pax-style reinforcement

- **PaX UDEREF + PAX_USERCOPY_HARDEN on `attr` copy_from_user** — `attr` is the only user pointer in this syscall; copy goes through SMAP/UDEREF-protected `copy_from_user` to a stack-allocated `LandlockRulesetAttr` (compile-time bounded). The kernel never directly dereferences `attr`.
- **PaX KERNEXEC + PAX_REFCOUNT on ruleset object** — `landlock_ruleset` allocations are read-mostly post-creation; KERNEXEC keeps the rules btree in a write-then-seal slab so post-`landlock_restrict_self` modification can be audited. `ruleset.refcount` overflow-traps under long-running browsers / runtimes stacking many rulesets across exec.
- **Landlock + GRKERNSEC_CHROOT_FINDTASK as complementary sandbox** — Landlock is *self-confinement* (caller restricts own ambient authority); `GRKERNSEC_CHROOT_FINDTASK` is *namespace-fence* (chrooted task cannot observe/signal outside). They compose: a process that has called `landlock_restrict_self` to drop `LANDLOCK_ACCESS_FS_*` and `LANDLOCK_SCOPE_SIGNAL` cannot subsequently signal outside its scope (Landlock side) and cannot enumerate outside via `/proc` (chroot-findtask side). Together they approximate a no-CAP user sandbox.
- **GRKERNSEC_HARDEN_USERFAULTFD-style policy knob** — `security.landlock.unprivileged_enabled` (default `1`); when `0`, creation path requires `CAP_SYS_ADMIN` (VERSION-probe always succeeds). Analogous to `vm.unprivileged_userfaultfd`; allows administrators to disable unprivileged Landlock where SELinux/AppArmor is the only sanctioned MAC.
- **`handled_access_*` bit-set audit-on-mismatch; per-userns ABI ceiling** — unprivileged uid requesting `_REFER` / `_TRUNCATE` / `_IOCTL_DEV` logged at `LOGLEVEL_INFO`; `security.landlock.abi_ceiling.<userns>` refuses bits above given ABI per userns (pins containers to stable Landlock ABI across migrations).
- **inotify ucount + max_user_watches awareness** — sandboxed processes' per-uid `fs.inotify.max_user_watches` lowered to a fraction (default 1/4); prevents sandbox cover for resource exhaustion.
- **fanotify priority + GRKERNSEC_HIDESYM on fid info-leak prevention** — Landlock-restricted task cannot create fanotify permission-class groups (see `fanotify_init.md`); notification-class groups have `FAN_REPORT_FID` forced off so the `file_handle` inode-number side channel is closed. Pairs with `GRKERNSEC_HIDESYM` to defeat inode-fingerprinting.
- **Forced CLOEXEC strict; PAX_RANDKSTACK on entry; GRKERNSEC_HIDESYM on /proc/<pid>/status Landlock field; audit log on ABI version-probe** — fd unconditionally `O_CLOEXEC` (cannot be cleared via `fcntl(F_SETFD)`); kernel stack randomized at every ruleset creation; per-task Landlock domain ID replaced with sentinel for cross-pidns observers (defeats sandbox enumeration); version-probe logged at `LOGLEVEL_INFO` (rate-limited per uid) to correlate probe with subsequent installation. CAP_LINUX_IMMUTABLE not required but `LANDLOCK_SCOPE_SIGNAL` audit-logged when used to refuse signals from privileged ancestors.

