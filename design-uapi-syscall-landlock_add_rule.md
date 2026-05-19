---
title: "Tier-5 syscall: landlock_add_rule(2) — syscall 445"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`landlock_add_rule(2)` is the second of three Landlock syscalls (after `landlock_create_ruleset(2)` and before `landlock_restrict_self(2)`). It appends a single per-rule entry to a not-yet-enforced ruleset identified by `ruleset_fd`. Each rule narrows what the future-enforced ruleset will permit: a path-beneath rule grants a specific set of filesystem access bits (`LANDLOCK_ACCESS_FS_*`) under a given directory or file; a network-port rule grants `LANDLOCK_ACCESS_NET_BIND_TCP` / `LANDLOCK_ACCESS_NET_CONNECT_TCP` for a specific TCP port. Rules can only **narrow** the access mask declared at ruleset creation — adding an access bit that was not present in `handled_access_fs` / `handled_access_net` returns `-EINVAL`.

Landlock's design is **monotonic**: rules can be added freely until the ruleset is enforced via `landlock_restrict_self`; once enforced, the ruleset is immutable for that task and (recursively) its descendants. Stacking multiple rulesets always produces the intersection — a child task cannot widen what its parent allowed. This makes Landlock the canonical unprivileged-userspace sandboxing API, complementing seccomp (syscall filtering) and namespaces (resource isolation). Critical for: container runtime hooks, browser sandboxes (Firefox / Chromium content processes), language runtimes (Deno, Bun, Node `--permission`), confined CI workers, and any unprivileged service that wants to drop access to filesystem subtrees and network ports.

### Acceptance Criteria

- [ ] AC-1: Add a `LANDLOCK_RULE_PATH_BENEATH` rule with valid `parent_fd` and `allowed_access ⊆ handled_access_fs`: returns 0.
- [ ] AC-2: Add a path-beneath rule with `allowed_access` including a bit NOT in `handled_access_fs`: returns `-EINVAL`.
- [ ] AC-3: Add a rule to an already-enforced ruleset: returns `-EPERM`.
- [ ] AC-4: Add a rule with `flags != 0`: returns `-EINVAL`.
- [ ] AC-5: Add a `LANDLOCK_RULE_NET_PORT` rule with `port = 8080` and `_NET_BIND_TCP`: returns 0 (when ABI ≥ 4).
- [ ] AC-6: Add a net-port rule with `port = 65536`: returns `-EINVAL`.
- [ ] AC-7: Add a path-beneath rule with `parent_fd` an O_PATH on `/dev/null`: returns 0 (rule binds the inode).
- [ ] AC-8: Add a path-beneath rule with `allowed_access = 0`: returns `-ENOMSG`.
- [ ] AC-9: Add a rule to an invalid `ruleset_fd`: returns `-EBADF`.
- [ ] AC-10: Add a rule to a `ruleset_fd` that refers to a non-Landlock fd: returns `-EBADFD`.
- [ ] AC-11: After successful enforcement, the rule blocks operations outside its grant set (verified by following `landlock_restrict_self` + an access attempt).
- [ ] AC-12: Stacked rulesets always intersect (child cannot widen).
- [ ] AC-13: Landlock disabled at boot (CONFIG_SECURITY_LANDLOCK=n): all `add_rule` calls return `-EOPNOTSUPP`.

### Architecture

```rust
#[syscall(nr = 445, abi = "sysv")]
pub fn sys_landlock_add_rule(
    ruleset_fd: i32,
    rule_type: u32,
    rule_attr: UserPtr<u8>,
    flags: u32,
) -> isize {
    Landlock::add_rule(ruleset_fd, rule_type, rule_attr, flags)
}
```

`Landlock::add_rule(ruleset_fd, rule_type, rule_attr, flags) -> isize`:
1. if flags != 0 { return -EINVAL; }
2. /* Ruleset fd lookup + type check */
3. let ruleset = LandlockFdTable::get_ruleset(current(), ruleset_fd)?;   // EBADF / EBADFD
4. /* Reject if already enforced (immutable) */
5. if ruleset.is_enforced() { return -EPERM; }
6. /* Per-rule-type dispatch */
7. match LandlockRuleType::try_from(rule_type) {
8.   Ok(LandlockRuleType::PathBeneath) => Landlock::add_path_beneath(&ruleset, rule_attr),
9.   Ok(LandlockRuleType::NetPort)     => Landlock::add_net_port(&ruleset, rule_attr),
10.  _ => -EINVAL,
11. }

`Landlock::add_path_beneath(ruleset, attr_ptr) -> isize`:
1. let attr = Landlock::copy_path_beneath_attr(attr_ptr)?;             // EFAULT / E2BIG
2. if attr.allowed_access == 0 { return -ENOMSG; }
3. /* Monotonic narrowing: rule access ⊆ ruleset's handled_access_fs */
4. if (attr.allowed_access & !ruleset.handled_access_fs) != 0 { return -EINVAL; }
5. /* Resolve parent_fd → inode */
6. let parent_file = FdTable::get(current(), attr.parent_fd).ok_or(-EBADF)?;
7. let parent_inode = parent_file.f_inode();
8. /* Bind rule by inode (not path string) */
9. let rule = LandlockRule::PathBeneath {
10.   inode_ref: parent_inode.clone_ref(),
11.   access: attr.allowed_access,
12. };
13. /* Append under ruleset.lock */
14. ruleset.append_rule(rule).map_err(|_| -ENOMEM)?;
15. 0

`Landlock::add_net_port(ruleset, attr_ptr) -> isize`:
1. let attr = Landlock::copy_net_port_attr(attr_ptr)?;
2. if attr.allowed_access == 0 { return -ENOMSG; }
3. if (attr.allowed_access & !ruleset.handled_access_net) != 0 { return -EINVAL; }
4. if attr.port > u16::MAX as u64 { return -EINVAL; }
5. let rule = LandlockRule::NetPort {
6.   port: attr.port as u16,
7.   access: attr.allowed_access,
8. };
9. ruleset.append_rule(rule).map_err(|_| -ENOMEM)?;
10. 0

`Landlock::copy_path_beneath_attr(uptr) -> Result<PathBeneathAttr>`:
1. const K: usize = size_of::<PathBeneathAttr>();
2. let mut attr = PathBeneathAttr::zeroed();
3. unsafe { uptr.copy_in_partial(&mut attr, K)?; }
4. /* Forward-compat probe of trailing bytes */
5. Ok(attr)

### Out of Scope

- `landlock_create_ruleset(2)` (Tier-5 separate doc — ruleset allocation + handled_access_* declaration).
- `landlock_restrict_self(2)` (Tier-5 separate doc — domain stacking + enforcement).
- Landlock LSM hooks (Tier-3 `security/landlock/fs.md`, `security/landlock/net.md`).
- Landlock ruleset internal data structures (Tier-3 `security/landlock/ruleset.md`).
- Domain stacking algorithm (Tier-3 `security/landlock/domain.md`).
- seccomp filtering of Landlock syscalls (Tier-3 `security/seccomp/filters.md`).
- Implementation code.

### signature

```c
int landlock_add_rule(int ruleset_fd,
                      enum landlock_rule_type rule_type,
                      const void *rule_attr,
                      __u32 flags);
```

```c
enum landlock_rule_type {
    LANDLOCK_RULE_PATH_BENEATH = 1,
    LANDLOCK_RULE_NET_PORT     = 2,
};

struct landlock_path_beneath_attr {
    __u64 allowed_access;       /* LANDLOCK_ACCESS_FS_* bitmask */
    __s32 parent_fd;            /* fd referencing a dir or file */
} __attribute__((packed));

struct landlock_net_port_attr {
    __u64 allowed_access;       /* LANDLOCK_ACCESS_NET_* bitmask */
    __u64 port;                 /* TCP port (host byte order) */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `ruleset_fd` | `int` | in | fd from `landlock_create_ruleset(2)`; not yet enforced. |
| `rule_type` | `enum landlock_rule_type` | in | `LANDLOCK_RULE_PATH_BENEATH` or `LANDLOCK_RULE_NET_PORT`. |
| `rule_attr` | `const void *` | in | Per-rule-type struct; tagged-union dispatched by `rule_type`. |
| `flags` | `__u32` | in | Must be 0 (reserved; non-zero returns `-EINVAL`). |

### return value

| Value | Meaning |
|---|---|
| `0` | Rule appended. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EOPNOTSUPP` | Landlock disabled at boot or not built into the kernel. |
| `EINVAL` | `rule_type` unknown; `flags` non-zero; `allowed_access` contains a bit not declared in the ruleset's `handled_access_*`; `port` out of range; reserved struct fields non-zero. |
| `ENOMSG` | `allowed_access == 0` (empty rule has no semantic). |
| `EBADF` | `ruleset_fd` is not a valid fd; or (for path-beneath) `parent_fd` is not a valid fd. |
| `EBADFD` | `ruleset_fd` does not refer to a Landlock ruleset; or `parent_fd` refers to an O_PATH fd that cannot be re-stat'd. |
| `EPERM` | Ruleset has already been enforced (immutable). |
| `EFAULT` | `rule_attr` user pointer faults during `copy_from_user`. |
| `E2BIG` | `rule_attr` size larger than kernel-supported attr layout AND the trailing region is non-zero. |

### abi surface

```text
__NR_landlock_add_rule (x86_64)    = 445
__NR_landlock_add_rule (i386)      = 445
__NR_landlock_add_rule (arm64)     = 445
__NR_landlock_add_rule (riscv)     = 445
__NR_landlock_add_rule (loongarch) = 445

/* Landlock ABI versions */
LANDLOCK_ABI_V1 (5.13) — filesystem rules only
LANDLOCK_ABI_V2 (5.19) — adds LANDLOCK_ACCESS_FS_REFER (cross-hierarchy rename/link)
LANDLOCK_ABI_V3 (6.2)  — adds LANDLOCK_ACCESS_FS_TRUNCATE
LANDLOCK_ABI_V4 (6.7)  — adds LANDLOCK_RULE_NET_PORT
LANDLOCK_ABI_V5 (6.10) — adds LANDLOCK_ACCESS_FS_IOCTL_DEV
LANDLOCK_ABI_V6 (6.12) — adds LANDLOCK_SCOPE_ABSTRACT_UNIX_SOCKET / LANDLOCK_SCOPE_SIGNAL

/* Userspace probes ABI via landlock_create_ruleset(NULL, 0, LANDLOCK_CREATE_RULESET_VERSION) */
```

### compatibility contract

REQ-1: Syscall number is **445** on every architecture. ABI-stable since Linux 5.13.

REQ-2: `ruleset_fd` MUST refer to a Landlock ruleset created via `landlock_create_ruleset(2)` and NOT yet enforced. Enforced rulesets reject all further `add_rule` calls with `-EPERM`.

REQ-3: `rule_type == LANDLOCK_RULE_PATH_BENEATH`:
- `rule_attr` points to `struct landlock_path_beneath_attr`.
- `parent_fd` is opened (typically O_PATH or O_RDONLY) and refers to a directory or regular file.
- `allowed_access` is a non-zero subset of the ruleset's `handled_access_fs`.
- The rule grants access bits when path resolution reaches an inode equal-to-or-beneath `parent_fd`.

REQ-4: `rule_type == LANDLOCK_RULE_NET_PORT` (Linux 6.7+):
- `rule_attr` points to `struct landlock_net_port_attr`.
- `port` is a TCP port number (0..65535).
- `allowed_access` is a non-zero subset of the ruleset's `handled_access_net`.
- Granted operations: `bind(2)` for `_NET_BIND_TCP`, `connect(2)` for `_NET_CONNECT_TCP`.

REQ-5: **Monotonic narrowing only**: `allowed_access & ~handled_access_*` MUST be zero. Adding a bit that wasn't declared at ruleset creation yields `-EINVAL`. This invariant exists so userspace can compute the maximum-possible-grant from `handled_access_*` alone.

REQ-6: **Reserved-field zero check**: extension fields in `landlock_*_attr` MUST be zero on entry. Forward-compat probe: if `rule_attr` is supplied with extra trailing bytes, the kernel verifies they are zero.

REQ-7: **NNP precondition** for enforcement (enforced later in `landlock_restrict_self`): the calling task MUST have either `CAP_SYS_ADMIN` OR `no_new_privs` set. `add_rule` itself does NOT require these — only enforcement does — but practical sandboxes set NNP before any `add_rule` call.

REQ-8: Rule storage: each `add_rule` appends to an internal hash table / tree keyed by inode (path-beneath) or port (net-port). Rule fan-out per inode is bounded; per-ruleset rule count is bounded by `LANDLOCK_MAX_NUM_RULES` (kernel-internal, large).

REQ-9: `parent_fd` is resolved at `add_rule` time, not at enforcement time. The rule binds to the **inode** (not the path string); subsequent renames of the directory do not invalidate the rule.

REQ-10: `parent_fd` may be a path inside a different mount namespace; the rule is recorded by inode regardless. Cross-mountns enforcement uses the inode's path during access checks.

REQ-11: Once `landlock_restrict_self(ruleset_fd)` is called, the ruleset is stacked onto the calling task's domain. From that point: (a) the ruleset is immutable, (b) the task and its descendants can ONLY narrow further by stacking additional rulesets, never widen.

REQ-12: LSM hooks: `add_rule` itself does not call out-of-Landlock LSM hooks; the access-grant accounting happens at filesystem / network operation time via the Landlock LSM.

REQ-13: Per-`seccomp` filters: `landlock_add_rule` can be filtered like any syscall; the rule_type arg is available for filter inspection.

REQ-14: Per-Rookery: rules stored in a Rookery-side `LandlockRuleset` with typed `Rule::PathBeneath { inode_ref, access }` / `Rule::NetPort { port, access }` variants; enforcement state machine prevents post-enforce mutation at the type level.

REQ-15: ABI versioning: userspace probes `landlock_create_ruleset(NULL, 0, LANDLOCK_CREATE_RULESET_VERSION)`; if the returned version is < required, the rule type is unsupported.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_must_be_zero` | INVARIANT | per-add_rule: flags != 0 ⟹ EINVAL. |
| `reject_enforced_ruleset` | INVARIANT | per-add_rule: ruleset.is_enforced ⟹ EPERM. |
| `monotonic_narrowing` | INVARIANT | per-rule: allowed_access ⊆ ruleset.handled_access_*. |
| `inode_binding_not_path` | INVARIANT | per-path-rule: rule stored by inode_ref, not by path string. |
| `attr_copy_bounded` | INVARIANT | per-add_rule: copy_from_user never exceeds sizeof(attr). |
| `reserved_fields_zero` | INVARIANT | per-add_rule: trailing bytes verified zero. |
| `port_in_u16_range` | INVARIANT | per-net-port-rule: port ≤ 65535. |

### Layer 2: TLA+

`security/landlock-add-rule.tla`:
- States: per-ruleset (mutable | enforced), per-rule-append, per-stacking.
- Properties:
  - `safety_no_mutation_after_enforce` — per-ruleset: post-enforce add_rule fails.
  - `safety_monotonic_narrowing` — per-rule: access ⊆ handled_access_*.
  - `safety_inode_binding_stable` — per-rule: rename of parent_fd does not invalidate rule.
  - `safety_stack_intersect` — per-domain: stacked rulesets always intersect (never union).
  - `liveness_add_terminates` — per-add_rule: returns in bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `add_rule` post: ruleset.rules count incremented by exactly 1 on success | `Landlock::add_rule` |
| `add_rule` post: ruleset.rules unchanged on failure | `Landlock::add_rule` |
| `add_path_beneath` post: rule.access ⊆ ruleset.handled_access_fs | `Landlock::add_path_beneath` |
| `add_net_port` post: rule.access ⊆ ruleset.handled_access_net | `Landlock::add_net_port` |
| `add_rule` post: enforced ruleset never mutated | `Landlock::add_rule` |

### Layer 4: Verus / Creusot functional

Per-`landlock_add_rule(2)` man-page equivalence. `tools/testing/selftests/landlock/` selftests pass (fs_test, net_test, ptrace_test). Per-Documentation/userspace-api/landlock.rst semantic equivalence verified.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`landlock_add_rule(2)` reinforcement:

- **Per-monotonic-narrowing invariant** — defense against per-policy-widening; impossible by construction.
- **Per-inode-binding** — defense against per-rename-escape (rules survive rename).
- **Per-enforced-ruleset immutability** — defense against per-post-enforce policy mutation.
- **Per-attr-tail zero-probe** — defense against per-extension-field smuggling.
- **Per-flags-zero check** — defense against per-extension-flag smuggling.
- **Per-reserved-field zero check** — defense against per-union smuggling.
- **Per-allowed-access subset enforced** — defense against per-cap-bit smuggling.
- **Per-rule-count bounded** — defense against per-ruleset DoS (bounded `LANDLOCK_MAX_NUM_RULES`).
- **Per-typed Rule enum in Rookery** — defense against per-raw-pointer type confusion.

### grsecurity / pax-style reinforcement

- **NNP precondition enforced at enforce-time** — `landlock_restrict_self` rejects without `CAP_SYS_ADMIN` OR `PR_SET_NO_NEW_PRIVS`; the add_rule path itself does not require these, but practical sandboxes (and grsec-recommended pattern) set NNP first. Grsec adds an audit warning when a non-NNP process calls `add_rule` (suggests broken sandbox).
- **Monotonic-narrowing enforced unconditionally** — even with `CAP_SYS_ADMIN`, an add_rule that widens beyond `handled_access_*` is rejected. There is NO admin override.
- **PaX UDEREF + SMAP on `rule_attr` copy_from_user** — defense against per-attr-pointer kernel-deref bug.
- **PAX_USERCOPY_HARDEN on `landlock_*_attr` copy** — whitelisted slab, bounded length.
- **Reserved-field zero check is mandatory** — extension-bit smuggling impossible.
- **PAX_REFCOUNT on inode_ref in rules** — defense against per-rule-inode UAF when parent dir is unlinked.
- **GRKERNSEC_LANDLOCK_AUDIT** — every successful add_rule is logged with `pid`, `uid`, `ruleset_fd`, `rule_type`, `allowed_access`, and (for path-beneath) the resolved inode/path. Failed adds also logged at higher rate-limit. Allows postmortem analysis of which sandboxes existed.
- **GRKERNSEC_LANDLOCK_ENFORCE** — optional system-wide policy: certain process groups MUST have a Landlock ruleset enforced before they may exec setuid/setgid binaries. Implements "Landlock-required" zones.
- **Per-ruleset-fd O_CLOEXEC enforced** — ruleset fds are created with O_CLOEXEC; cannot survive execve into a less-trusted binary unless deliberately preserved.
- **GRKERNSEC_HIDESYM on Landlock internals** — `landlock_append_rule`, `landlock_ruleset_*` not exposed in `/proc/kallsyms`.
- **PaX KERNEXEC on Landlock LSM hooks** — hook table read-only after init; per-domain stack-depth bounded at `LANDLOCK_MAX_NUM_LAYERS`.
- **Per-net-port rule restricted to TCP** — UDP not yet supported by Landlock.
- **Audit ring-buffer flood-immune** — per-uid summary at 1 Hz for repeated identical add_rule patterns.

