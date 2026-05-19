# Tier-5 syscall: landlock_restrict_self(2) — syscall 446

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - security/landlock/syscalls.c (SYSCALL_DEFINE2(landlock_restrict_self))
  - security/landlock/cred.c (landlock_init_ruleset_domain, landlock_cred)
  - security/landlock/task.c (task domain inheritance on fork/exec)
  - include/uapi/linux/landlock.h
  - arch/x86/entry/syscalls/syscall_64.tbl (446  common  landlock_restrict_self)
  - Documentation/userspace-api/landlock.rst, man landlock(7), landlock_restrict_self(2)
-->

## Summary

`landlock_restrict_self(2)` installs a previously-created Landlock ruleset (referenced by fd from `landlock_create_ruleset(2)`) onto the **calling thread**, merging it with any pre-existing Landlock domain held by the thread. The merge semantics are strictly monotonic: the thread's effective access set is the **intersection** of all installed rulesets, so each call can only ever tighten restrictions. The installed ruleset is inherited by all descendants (fork / clone / clone3 children always inherit the parent's Landlock domain) and survives execve(2) — there is no Landlock "drop" syscall by design. This is the third and final syscall of the Landlock trio (444 / 445 / 446). Critical for: unprivileged self-sandboxing in browsers, language runtimes, build sandboxes, container init, and any process that wishes to drop ambient filesystem/network authority before executing untrusted code.

This Tier-5 covers the userspace ABI of syscall 446. Ruleset creation lives in `landlock_create_ruleset.md` (444); rule attachment in `landlock_add_rule.md` (445).

## Signature

```c
int landlock_restrict_self(int ruleset_fd, uint32_t flags);
```

Rust ABI shim:

```rust
pub fn sys_landlock_restrict_self(ruleset_fd: i32, flags: u32) -> isize;
```

Syscall number: **446** (x86_64).

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `ruleset_fd` | `i32` | IN | fd returned by `landlock_create_ruleset(2)`; must reference a `landlock-ruleset` object |
| `flags` | `u32` | IN | reserved; currently `0`. ABI v6+ adds `LANDLOCK_RESTRICT_SELF_LOG_*` selectors for per-task audit policy |

## Return

- **Success**: `0`.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EOPNOTSUPP` | Landlock disabled at boot or `CONFIG_SECURITY_LANDLOCK=n` |
| `EINVAL` | unknown bit in `flags`; `ruleset_fd` references a non-Landlock anon_inode |
| `EBADF` | `ruleset_fd` is not a valid fd |
| `EBADFD` | `ruleset_fd` is a Landlock fd but refers to an inconsistent / corrupt ruleset (defensive; not expected in normal use) |
| `EPERM` | task is set-uid (`AT_SECURE`) and current `no_new_privs` is not set — Landlock requires `PR_SET_NO_NEW_PRIVS == 1` (or `CAP_SYS_ADMIN`); refusing prevents privilege boundary confusion |
| `E2BIG` | combined ruleset would exceed per-task rule-count cap |
| `ENOMEM` | cannot allocate the new merged domain |

## ABI surface

Flags bits (ABI v6+):

| Constant | Value | Purpose |
|---|---|---|
| `LANDLOCK_RESTRICT_SELF_LOG_SAME_EXEC_OFF` | `0x1` | suppress audit for same-exec denials |
| `LANDLOCK_RESTRICT_SELF_LOG_NEW_EXEC_ON` | `0x2` | enable audit only for post-exec denials |
| `LANDLOCK_RESTRICT_SELF_LOG_SUBDOMAINS_OFF` | `0x4` | suppress audit for sub-domain-inherited denials |

(For baseline 7.1.0-rc2 with `LANDLOCK_ABI_VERSION = 5`, `flags` MUST be `0`.)

Required preconditions:
- `PR_SET_NO_NEW_PRIVS == 1` (set via `prctl(PR_SET_NO_NEW_PRIVS, 1, ...)`), OR
- The task holds `CAP_SYS_ADMIN` in `init_user_ns` (rare; mainly for system-management agents).

Both conditions ensure that subsequent execve cannot regain unrestricted authority via a setuid binary — the Landlock domain survives exec, and `no_new_privs` neutralizes setuid bits, so the two together close the privilege-escalation gap.

## Compatibility contract

REQ-1: `flags` validation:
- For ABI v5: `flags` MUST be `0`; non-zero ⟹ `-EINVAL`.
- For ABI v6+: `flags` is a subset of `LANDLOCK_RESTRICT_SELF_LOG_*`; unknown bit ⟹ `-EINVAL`.

REQ-2: `ruleset_fd` validation:
- `lookup_fd(ruleset_fd)` ⟹ if no fd, `-EBADF`.
- `fd.f_op != &LANDLOCK_RULESET_FOPS` ⟹ `-EINVAL`.
- `fd.private_data` cast to `LandlockRuleset` must yield a consistent object (cross-check `abi_version ≤ KERNEL_ABI_VERSION`); else `-EBADFD`.

REQ-3: `no_new_privs` precondition:
- `current.no_new_privs == false && !capable(CAP_SYS_ADMIN)` ⟹ `-EPERM`.
- The check happens *before* any state mutation.

REQ-4: Domain merge:
- The thread's current `landlock_cred->domain` (may be NULL for a never-restricted thread) is composed with the incoming ruleset.
- The merge is the **set-union of handled-access bits** and the **set-intersection of per-path/per-port rule sets** — i.e. the resulting thread can perform only what BOTH the prior domain and the new ruleset would have independently allowed, on the union of access dimensions that either covers.
- The composition preserves the "tighter is the new ground truth" invariant: a process can never relax a previously-installed Landlock domain.

REQ-5: Per-task rule-count cap:
- Cumulative number of rules across all installed rulesets ≤ `LANDLOCK_MAX_NUM_RULES` (default `4096`).
- Exceeded ⟹ `-E2BIG`.

REQ-6: Atomicity:
- Domain merge happens under `task.cred_lock` (or RCU-update of `task.cred`); pre-merge and post-merge are observable but no torn-state.
- On failure (e.g. `-ENOMEM` mid-merge), the original domain is preserved unchanged.

REQ-7: Inheritance:
- fork/clone/clone3 children inherit the parent's `landlock_cred->domain` (refcounted shared).
- execve preserves the domain across the new mm + new credentials (the new task picks up the domain as part of credential transition).
- The domain is **persistent** for the task's lifetime; no syscall removes it.

REQ-8: Thread vs. process scope:
- Landlock restrictions are **per-thread** (per-`task_struct`), not per-process. A multithreaded process must call `landlock_restrict_self` from each thread that needs the restriction, OR have a single thread call it followed by all-thread synchronization.
- This differs from seccomp's `SECCOMP_FILTER_FLAG_TSYNC` semantics; Landlock has no thread-sync flag in baseline.

REQ-9: Composition with seccomp:
- Both restrictions stack independently and orthogonally; seccomp filters syscall numbers, Landlock filters per-object access. Order of installation does not matter.

REQ-10: Empty merge:
- If `ruleset_fd` refers to a ruleset with `handled_access_fs = 0 && handled_access_net = 0 && scoped = 0`, the merge is rejected at create time (see `landlock_create_ruleset.md` REQ-3). `landlock_restrict_self` thus never sees an empty ruleset.

REQ-11: fd reference:
- `landlock_restrict_self` takes an additional reference on the ruleset (via the merged domain); the caller may safely `close(ruleset_fd)` afterwards.

REQ-12: Logging policy (ABI v6+):
- The `LANDLOCK_RESTRICT_SELF_LOG_*` bits select which subsequent denial events produce audit messages; default (no flag) logs all denials at `LOGLEVEL_INFO`, rate-limited.

REQ-13: Subdomain semantics (ABI v6+):
- Stacked rulesets form a **subdomain chain**; each `landlock_restrict_self` call pushes a new subdomain onto the chain. Denials may be attributed to a specific subdomain for audit purposes.

## Acceptance Criteria

- [ ] AC-1: Task with `no_new_privs = 1`: `landlock_restrict_self(fd, 0)` returns `0`; subsequent disallowed access denied with `-EACCES`.
- [ ] AC-2: Task with `no_new_privs = 0` and no CAP_SYS_ADMIN: `landlock_restrict_self(fd, 0)` ⟹ `-EPERM`.
- [ ] AC-3: `ruleset_fd = -1`: `-EBADF`.
- [ ] AC-4: `ruleset_fd` is a regular file fd: `-EINVAL`.
- [ ] AC-5: `flags = 0x1` on ABI v5 kernel: `-EINVAL`.
- [ ] AC-6: After `restrict_self`, fork child inherits the domain: child denied same accesses as parent.
- [ ] AC-7: After `restrict_self`, execve preserves the domain: the new process is still restricted.
- [ ] AC-8: Two successive `restrict_self` calls with different rulesets: final domain is the intersection.
- [ ] AC-9: Restriction is monotonic: a later `restrict_self` cannot grant access denied by an earlier one.
- [ ] AC-10: `close(ruleset_fd)` after restrict_self: domain still in effect.
- [ ] AC-11: Cumulative rule count exceeding `LANDLOCK_MAX_NUM_RULES`: `-E2BIG`.
- [ ] AC-12: Thread scope: thread A calls restrict_self; thread B in same process is unaffected.
- [ ] AC-13: ENOMEM mid-merge: pre-existing domain unchanged.
- [ ] AC-14: setuid binary executed by restricted task: setuid bit suppressed (combination of no_new_privs + Landlock).
- [ ] AC-15: Composable with seccomp: filter installed before or after Landlock; both effects apply.

## Architecture

Rookery surface in `kernel/security/landlock/syscall.rs`:

```rust
bitflags! {
    pub struct LandlockRestrictFlags: u32 {
        const LOG_SAME_EXEC_OFF    = 0x1;  // ABI v6+
        const LOG_NEW_EXEC_ON      = 0x2;  // ABI v6+
        const LOG_SUBDOMAINS_OFF   = 0x4;  // ABI v6+
    }
}

pub struct LandlockDomain {
    pub layers:           Vec<Arc<LandlockRuleset>>,
    pub handled_access_fs:  u64,        // union
    pub handled_access_net: u64,        // union
    pub scoped:             u64,        // union
    pub rule_count:         u32,        // cumulative
    pub log_policy:         LandlockLogPolicy,
}

pub struct LandlockCred {
    pub domain: Option<Arc<LandlockDomain>>,
}
```

`Landlock::restrict_self(ruleset_fd, flags) -> isize`:
1. let f = LandlockRestrictFlags::from_bits(flags).ok_or(-EINVAL)?;
2. if LANDLOCK_ABI_VERSION < 6 && !f.is_empty() { return Err(-EINVAL); }
3. let file = current_files().lookup(ruleset_fd).ok_or(-EBADF)?;
4. if file.f_op as *const _ != &LANDLOCK_RULESET_FOPS as *const _ { return Err(-EINVAL); }
5. let ruleset: Arc<LandlockRuleset> = file.private_data_as::<LandlockRuleset>().ok_or(-EBADFD)?;
6. if !current_task().no_new_privs && !capable(CAP_SYS_ADMIN) { return Err(-EPERM); }
7. /* Build new domain by composing with current. */
8. let new_domain = Landlock::compose_domain(
     current_cred().landlock.domain.clone(),
     ruleset.clone(),
     f,
   )?;
9. if new_domain.rule_count > LANDLOCK_MAX_NUM_RULES { return Err(-E2BIG); }
10. /* Install: cred-copy + replace. */
11. let new_cred = prepare_creds().ok_or(-ENOMEM)?;
12. new_cred.landlock.domain = Some(Arc::new(new_domain));
13. commit_creds(new_cred);
14. ruleset.installed_count.fetch_add(1);
15. Ok(0)

`Landlock::compose_domain(prev, ruleset, flags) -> Result<LandlockDomain>`:
1. let mut layers = prev.as_ref().map(|d| d.layers.clone()).unwrap_or_default();
2. layers.push(ruleset.clone());
3. /* Union of handled bits */
4. let handled_access_fs = prev.as_ref().map_or(0, |d| d.handled_access_fs) | ruleset.handled_access_fs;
5. let handled_access_net = prev.as_ref().map_or(0, |d| d.handled_access_net) | ruleset.handled_access_net;
6. let scoped = prev.as_ref().map_or(0, |d| d.scoped) | ruleset.scoped;
7. /* Cumulative rule count */
8. let rule_count = prev.as_ref().map_or(0, |d| d.rule_count)
                  + ruleset.rules_fs.len() as u32
                  + ruleset.rules_net.len() as u32;
9. let log_policy = LandlockLogPolicy::from_flags(prev, flags);
10. Ok(LandlockDomain { layers, handled_access_fs, handled_access_net, scoped, rule_count, log_policy })

`LANDLOCK_RULESET_FOPS::release`:
1. fput drops `Arc<LandlockRuleset>`; the ruleset itself is freed only when `installed_count + open_fd_count == 0`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nnp_required` | INVARIANT | `no_new_privs == 0 && !CAP_SYS_ADMIN ⟹ -EPERM` |
| `fd_is_landlock` | INVARIANT | `f_op == LANDLOCK_RULESET_FOPS` else `-EINVAL` |
| `flags_subset` | INVARIANT | `flags & !ALL_LOG_FLAGS != 0 ⟹ -EINVAL` (ABI v5: any non-zero ⟹ `-EINVAL`) |
| `monotonic_restriction` | INVARIANT | post-state's permitted set ⊆ pre-state's permitted set |
| `rule_count_bounded` | INVARIANT | cumulative ≤ `LANDLOCK_MAX_NUM_RULES` |
| `failure_no_domain_mutation` | INVARIANT | any error path leaves `current.cred.landlock` unchanged |
| `domain_inherited_on_fork` | INVARIANT | child task's domain == parent's at fork time |
| `domain_persists_across_exec` | INVARIANT | exec preserves the domain |

### Layer 2: TLA+

`uapi/landlock_restrict_self.tla`:
- States: `validating`, `fd_lookup`, `nnp_check`, `composing`, `installing_cred`, `done`, `failed`.
- Properties:
  - `safety_monotonic_restriction` — installed domain only ever tightens; never grants new access.
  - `safety_nnp_or_cap` — restrict-self requires `no_new_privs` or CAP_SYS_ADMIN.
  - `safety_no_partial_install` — failure ⟹ original cred preserved.
  - `safety_inheritance` — fork/exec preserve domain.
  - `liveness_terminate` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `restrict_self` post(ok): `cred.landlock.domain = compose(prev, ruleset)` ∧ `rule_count ≤ MAX` | `Landlock::restrict_self` |
| `restrict_self` post(err): `cred.landlock.domain` unchanged | `Landlock::restrict_self` |
| `compose_domain` post: `handled_fs | net | scoped` is union; permitted-set is intersection | `Landlock::compose_domain` |
| `restrict_self` post(fork): child.cred.landlock == parent.cred.landlock | (fork hook) |
| `restrict_self` post(exec): post-exec.cred.landlock == pre-exec.cred.landlock | (exec hook) |

### Layer 4: Verus/Creusot functional

Per-`landlock_restrict_self(2)` / `landlock(7)` man pages, `Documentation/userspace-api/landlock.rst`, `tools/testing/selftests/landlock/` round-trip, sample-sandboxer semantic equivalence.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

landlock_restrict_self reinforcement:

- **no_new_privs precondition** — defense against setuid-binary privilege regain post-restriction.
- **Monotonic restriction** — defense against ruleset-stacking to widen authority.
- **Per-task rule-count cap** — defense against domain inflation DoS.
- **Atomic cred replacement** — defense against torn domain state observed by concurrent threads.
- **Inheritance across fork/exec** — defense against escape via child / new-mm.
- **No "drop Landlock" syscall** — defense against domain-rescind in a compromised task.
- **fd type check** — defense against opaque fd confusion.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF inherited** — the syscall takes only scalars; no user pointers. UDEREF discipline applies at the generic syscall entry path; the kernel never dereferences a user pointer in `landlock_restrict_self` body.
- **PAX_REFCOUNT on `LandlockDomain.refcount` and `LandlockRuleset.installed_count`** — defense against per-refcount-overflow UAF. Long-lived browser / runtime processes may stack many restrict_self calls; saturation must trap.
- **PaX KERNEXEC on domain chain** — the domain's `layers` vector and merged bit-masks are write-once-after-commit; subsequent inheritance treats them as read-only. KERNEXEC enforces that the in-place domain object is not mutated after `commit_creds`.
- **PAX_RANDKSTACK on landlock_restrict_self entry** — sandbox installation is a security-sensitive operation often performed at process startup or pre-exec; randomize kernel stack at every restrict_self to deny ROP-target stability.
- **Landlock + GRKERNSEC_CHROOT_FINDTASK as complementary sandbox** — Landlock confines the issuer's *access to objects*; `GRKERNSEC_CHROOT_FINDTASK` confines the issuer's *observation of other tasks*. Together they provide a no-CAP user sandbox approximating namespace + MAC. Under hardened policy, a chrooted task that has not installed a Landlock domain emits an audit `LOGLEVEL_INFO` hint at chroot-time; a Landlock-restricted task that is not chrooted likewise emits a hint. Both are advisory, never blocking.
- **GRKERNSEC_HARDEN_USERFAULTFD-style policy knob** — `security.landlock.unprivileged_enabled = 0` mode (set by administrator) forces `landlock_restrict_self` to additionally require that the ruleset have been created by a process holding `CAP_SYS_ADMIN`. This lets administrators disable unprivileged Landlock when SELinux/AppArmor is the only sanctioned MAC.
- **Per-userns ABI ceiling** — `security.landlock.abi_ceiling.<userns>` is consulted at `restrict_self` time; if the ruleset's `abi_version > userns_ceiling`, refuse with `-EINVAL`. Defends against container migration into a kernel with newer Landlock semantics than the container was tested against.
- **inotify ucount + max_user_watches downgrade** — when a task installs a Landlock domain, its effective `fs.inotify.max_user_watches` is lowered to a fraction of the per-uid cap (default `1/4`, configurable via sysctl) to bound resource exhaustion from a sandboxed task that retains inotify capability. The original cap is restored only on exit; not on fork/exec.
- **fanotify priority + GRKERNSEC_HIDESYM on fid info-leak prevention** — a Landlock-restricted task cannot create fanotify **permission-class** groups (rejected with `-EPERM` at `fanotify_init` time when the caller has a Landlock domain). Notification-class groups created by such a task have `FAN_REPORT_FID` forced off so the file-handle inode-number side channel is closed. This pairs with `GRKERNSEC_HIDESYM` to defeat inode-fingerprinting attacks from sandboxed code that retains audit-style observability.
- **key-quota per-uid (RLIMIT_MEMLOCK / kernel.keys.maxkeys) downgrade for Landlock-restricted tasks** — under hardened policy, a Landlock-restricted task's effective per-uid `kernel.keys.maxkeys` is lowered (default `1/2`). Prevents a sandboxed task from inflating the keystore as a covert channel back to its issuer.
- **CAP_LINUX_IMMUTABLE not required, but observed on `LANDLOCK_SCOPE_SIGNAL`** — restricting cross-domain signal delivery is logged when the restricted task could previously have signalled privileged ancestors (its launcher); audit-trail enables forensic correlation with subsequent ancestor anomalies.
- **GRKERNSEC_HIDESYM on `/proc/<pid>/status` Landlock domain field** — the Landlock domain ID exposed in `/proc/<pid>/status` is replaced with sentinel `0` for cross-pidns / cross-uid observers; defeats sandbox-enumeration.
- **Stacking depth cap with audit** — under hardened policy, more than `N = 16` stacked rulesets per task triggers an audit log entry; defense against domain-stacking DoS via rule-count-bypass attacks that craft many small rulesets.
- **Audit log on every restrict_self** — at `LOGLEVEL_INFO`, with creator uid + creds + pidns + cumulative rule count + handled-access bit-mask. Rate-limited per uid to prevent log flood from sandbox-init loops.
- **Reject restrict_self in `ptrace`-attached state** — defense against ptrace-supervisor coercing the tracee into a domain favorable to escape via subsequent ptrace operations. Under hardened policy, refuse with `-EPERM` if `current.ptrace != 0`; the tracer must detach first.
- **`PR_SET_NO_NEW_PRIVS` is irreversible** — already true upstream; hardened policy additionally audit-logs any attempt to clear it (which fails with `-EPERM`), correlating with prior `landlock_restrict_self`.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `landlock_create_ruleset.md` Tier-5 — ruleset creation (syscall 444)
- `landlock_add_rule.md` Tier-5 — rule attachment (syscall 445)
- `security/landlock/cred.md` Tier-3 — cred-side domain object, fork/exec inheritance
- `security/landlock/task.md` Tier-3 — per-task LSM hook dispatch
- `security/landlock/fs.md` Tier-3 — per-LSM-hook fs enforcement
- `security/landlock/net.md` Tier-3 — net enforcement
- Implementation code
