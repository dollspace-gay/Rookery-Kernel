---
title: "Tier-5 UAPI: include/uapi/linux/landlock.h — Landlock ABI"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

Landlock is an unprivileged-friendly, stackable LSM that lets an arbitrary process build a *ruleset* of explicitly handled access rights, populate it with *allow* rules, and then irrevocably restrict itself (and its descendants) to those rules. Three syscalls form the ABI: `landlock_create_ruleset(attr, size, flags)` creates a ruleset fd from a `struct landlock_ruleset_attr { __u64 handled_access_fs; __u64 handled_access_net; __u64 scoped; }`; `landlock_add_rule(ruleset_fd, rule_type, rule_attr, flags)` populates it via `struct landlock_path_beneath_attr { __u64 allowed_access; __s32 parent_fd; }` (packed) or `struct landlock_net_port_attr { __u64 allowed_access; __u64 port; }`; `landlock_restrict_self(ruleset_fd, flags)` commits the ruleset onto the calling task's credential. The `LANDLOCK_CREATE_RULESET_VERSION` and `LANDLOCK_CREATE_RULESET_ERRATA` flags discover the highest supported ABI version (currently >= 7) and per-version errata bitmask. Filesystem rights (`LANDLOCK_ACCESS_FS_EXECUTE`, `WRITE_FILE`, `READ_FILE`, `READ_DIR`, `REMOVE_DIR`, `REMOVE_FILE`, `MAKE_CHAR`, `MAKE_DIR`, `MAKE_REG`, `MAKE_SOCK`, `MAKE_FIFO`, `MAKE_BLOCK`, `MAKE_SYM`, `REFER`, `TRUNCATE`, `IOCTL_DEV`, `RESOLVE_UNIX`), network rights (`LANDLOCK_ACCESS_NET_BIND_TCP`, `CONNECT_TCP`), and scope flags (`LANDLOCK_SCOPE_ABSTRACT_UNIX_SOCKET`, `SIGNAL`) compose. Critical for: browser sandboxing, container runtime hardening, defensible setuid binaries, unprivileged path-confinement, audit-log-driven security analytics.

This Tier-5 covers `include/uapi/linux/landlock.h` (~407 lines).

### Acceptance Criteria

- [ ] AC-1: `landlock_create_ruleset(NULL, 0, LANDLOCK_CREATE_RULESET_VERSION)` returns ABI version ≥ 7.
- [ ] AC-2: `landlock_create_ruleset(NULL, 0, LANDLOCK_CREATE_RULESET_ERRATA)` returns a non-negative bitmask.
- [ ] AC-3: `landlock_create_ruleset(NULL, 0, 0xDEAD)` returns -EINVAL (unknown flag).
- [ ] AC-4: `landlock_create_ruleset(attr=NULL, size=8, flags=0)` returns -EFAULT.
- [ ] AC-5: `landlock_create_ruleset(attr={handled_access_fs:0, handled_access_net:0, scoped:0}, sizeof, 0)` returns -ENOMSG.
- [ ] AC-6: `landlock_create_ruleset(attr={handled_access_fs: 1<<63, ...}, sizeof, 0)` returns -EINVAL (reserved bit).
- [ ] AC-7: `landlock_create_ruleset(attr={...valid...}, sizeof, 0)` returns fd ≥ 0; fd has `O_CLOEXEC`.
- [ ] AC-8: `landlock_add_rule(fd, LANDLOCK_RULE_PATH_BENEATH, {allowed_access: 1<<63, parent_fd: dirfd}, 0)` returns -EINVAL (not in `handled_access_fs`).
- [ ] AC-9: `landlock_add_rule(fd, 99, ..., 0)` returns -EINVAL (unknown rule type).
- [ ] AC-10: `landlock_add_rule(fd, LANDLOCK_RULE_PATH_BENEATH, {allowed_access:0, parent_fd:dirfd}, 0)` returns -ENOMSG.
- [ ] AC-11: `landlock_add_rule(fd, LANDLOCK_RULE_NET_PORT, {allowed_access: BIND_TCP, port: 70000}, 0)` returns -EINVAL.
- [ ] AC-12: `landlock_restrict_self(fd, 0)` without `no_new_privs` returns -EPERM (unless CAP_SYS_ADMIN).
- [ ] AC-13: `landlock_restrict_self(fd, 0)` with `no_new_privs` succeeds and narrows credential.
- [ ] AC-14: `landlock_restrict_self(fd, 0x10000)` returns -EINVAL (unknown flag).
- [ ] AC-15: `landlock_restrict_self(-1, LANDLOCK_RESTRICT_SELF_LOG_SUBDOMAINS_OFF)` succeeds (log-mute only).
- [ ] AC-16: `landlock_restrict_self(-1, LANDLOCK_RESTRICT_SELF_LOG_SAME_EXEC_OFF)` returns -EINVAL.
- [ ] AC-17: After `landlock_restrict_self(...)`, `open("/etc/shadow", O_RDONLY)` with no FS rule covering /etc returns -EACCES.
- [ ] AC-18: After restrict with `REFER` not handled, `link("/a/x", "/b/x")` across hierarchies returns -EXDEV / -EACCES.
- [ ] AC-19: After restrict with `LANDLOCK_SCOPE_SIGNAL`, `kill(out_of_domain_pid, SIGTERM)` returns -EPERM.
- [ ] AC-20: After restrict with `LANDLOCK_SCOPE_ABSTRACT_UNIX_SOCKET`, `connect(@abstract_outside)` is denied.
- [ ] AC-21: `LANDLOCK_RESTRICT_SELF_TSYNC` propagates the new domain to every thread of the current process.
- [ ] AC-22: Closing the ruleset fd after restrict-self does not undo restrictions.
- [ ] AC-23: `execve(2)` carries the Landlock domain across exec.

### Architecture

```
pub struct LandlockRulesetAttr {
    pub handled_access_fs:  u64,
    pub handled_access_net: u64,
    pub scoped:             u64,
}

pub struct LandlockPathBeneathAttr {
    pub allowed_access: u64,
    pub parent_fd:      i32,
}   /* repr(packed) */

pub struct LandlockNetPortAttr {
    pub allowed_access: u64,
    pub port:           u64,
}

#[repr(u32)]
pub enum LandlockRuleType {
    PathBeneath = 1,
    NetPort     = 2,
}

pub struct Ruleset {
    pub handled_fs:   u64,
    pub handled_net:  u64,
    pub scoped:       u64,
    pub fs_rules:     PathTrie<u64>,        // inode → allowed_access bitmask
    pub net_rules:    PortMap<u16, u64>,    // port  → allowed_access bitmask
    pub sealed:       AtomicBool,           // true once attached to a credential
}

pub struct LandlockDomain {
    pub layers:       Vec<Arc<Ruleset>>,    // stacked, monotonic (newer = narrower)
    pub log_flags:    LogFlags,             // per-domain audit-log policy
}
```

`Landlock::sys_create_ruleset(attr_user, size, flags) -> Result<i32>`:
1. /* Discovery short-circuits */
2. if flags & LANDLOCK_CREATE_RULESET_VERSION:
   - if !attr_user.is_null() || size != 0 || flags != LANDLOCK_CREATE_RULESET_VERSION: return Err(EINVAL).
   - return Ok(LANDLOCK_ABI_VERSION as i32).
3. if flags & LANDLOCK_CREATE_RULESET_ERRATA:
   - if !attr_user.is_null() || size != 0 || flags != LANDLOCK_CREATE_RULESET_ERRATA: return Err(EINVAL).
   - return Ok(landlock_errata_bitmask() as i32).
4. if flags != 0: return Err(EINVAL).
5. let attr: LandlockRulesetAttr = copy_struct_from_user(attr_user, size, sizeof::<LandlockRulesetAttr>())?.
6. if attr.handled_access_fs & !LANDLOCK_MASK_ACCESS_FS_ALL != 0: return Err(EINVAL).
7. if attr.handled_access_net & !LANDLOCK_MASK_ACCESS_NET_ALL != 0: return Err(EINVAL).
8. if attr.scoped & !LANDLOCK_MASK_SCOPE_ALL != 0: return Err(EINVAL).
9. if (attr.handled_access_fs | attr.handled_access_net | attr.scoped) == 0: return Err(ENOMSG).
10. ruleset = Ruleset::new(attr).
11. fd = anon_inode_fd("[landlock-ruleset]", &RULESET_FOPS, ruleset, O_RDWR | O_CLOEXEC)?.
12. return Ok(fd).

`Landlock::sys_add_rule(ruleset_fd, rule_type, rule_attr_user, flags) -> Result<()>`:
1. if flags != 0: return Err(EINVAL).
2. ruleset = ruleset_from_fd(ruleset_fd).ok_or(EBADF)?.
3. if ruleset.sealed.load(Acquire): return Err(EINVAL).
4. match rule_type {
       LandlockRuleType::PathBeneath => {
           let p: LandlockPathBeneathAttr = copy_from_user_packed(rule_attr_user)?.
           if p.allowed_access == 0: return Err(ENOMSG).
           if p.allowed_access & !ruleset.handled_fs != 0: return Err(EINVAL).
           let parent = fd_to_path(p.parent_fd).ok_or(EBADFD)?.
           ruleset.fs_rules.insert(parent.inode(), p.allowed_access).
       },
       LandlockRuleType::NetPort => {
           let n: LandlockNetPortAttr = copy_from_user(rule_attr_user)?.
           if n.allowed_access == 0: return Err(ENOMSG).
           if n.allowed_access & !ruleset.handled_net != 0: return Err(EINVAL).
           if n.port > u16::MAX as u64: return Err(EINVAL).
           ruleset.net_rules.insert(n.port as u16, n.allowed_access).
       },
       _ => return Err(EINVAL),
   }
5. return Ok(()).

`Landlock::sys_restrict_self(ruleset_fd, flags) -> Result<()>`:
1. const VALID = LANDLOCK_RESTRICT_SELF_LOG_SAME_EXEC_OFF | LANDLOCK_RESTRICT_SELF_LOG_NEW_EXEC_ON
              | LANDLOCK_RESTRICT_SELF_LOG_SUBDOMAINS_OFF | LANDLOCK_RESTRICT_SELF_TSYNC.
2. if flags & !VALID != 0: return Err(EINVAL).
3. if !task_no_new_privs(current()) && !capable(CAP_SYS_ADMIN): return Err(EPERM).
4. /* log-mute-only */
5. if ruleset_fd == -1 {
       if flags & !(LANDLOCK_RESTRICT_SELF_LOG_SUBDOMAINS_OFF | LANDLOCK_RESTRICT_SELF_TSYNC) != 0:
           return Err(EINVAL).
       apply_log_flags_only(current(), flags).
       if flags & LANDLOCK_RESTRICT_SELF_TSYNC: tsync_log_flags(current(), flags).
       return Ok(()).
   }
6. ruleset = ruleset_from_fd(ruleset_fd).ok_or(EBADF)?.
7. ruleset.sealed.store(true, Release).
8. domain_old = current_landlock_domain(current()).cloned().unwrap_or_default().
9. domain_new = LandlockDomain { layers: domain_old.layers.push(ruleset.clone()), log_flags: derive_log_flags(domain_old.log_flags, flags) }.
10. if flags & LANDLOCK_RESTRICT_SELF_TSYNC {
        for t in current().process.threads {
            attach_domain(t, domain_new.clone()).
        }
        if task_no_new_privs(current()) { propagate_no_new_privs_to_threads(); }
    } else {
        attach_domain(current(), domain_new).
    }
11. return Ok(()).

`Landlock::check_fs(domain, inode, requested_access) -> Result<()>`:
1. for layer in domain.layers.iter() {
       handled = layer.handled_fs & requested_access.
       if handled == 0: continue.    /* layer doesn't speak about these bits */
       allowed = layer.fs_rules.walk_ancestors(inode).fold(0u64, |acc, m| acc | m).
       missing = handled & !allowed.
       if missing != 0:
           audit_log_if_enabled(domain.log_flags, "fs", inode, missing).
           return Err(EACCES).
   }
2. /* Special: REFER is denied unless explicitly allowed in every layer that handles any FS bit */
3. /* (handled separately by Landlock::check_refer) */
4. return Ok(()).

`Landlock::check_net(domain, port, requested_access) -> Result<()>`:
1. similar to check_fs over net_rules.

`Landlock::check_scope_unix_abstract(domain, server_task) -> Result<()>`:
1. for layer in domain.layers {
       if layer.scoped & LANDLOCK_SCOPE_ABSTRACT_UNIX_SOCKET == 0: continue.
       if !is_in_domain(server_task, layer): return Err(ECONNREFUSED).   /* per current behaviour */
   }
2. return Ok(()).

`Landlock::check_scope_signal(domain, target_task) -> Result<()>`:
1. for layer in domain.layers {
       if layer.scoped & LANDLOCK_SCOPE_SIGNAL == 0: continue.
       if !is_in_domain(target_task, layer): return Err(EPERM).
   }
2. return Ok(()).

### Out of Scope

- inotify (covered in `uapi/headers/inotify.md` Tier-5)
- fanotify (covered in `uapi/headers/fanotify.md` Tier-5)
- LSM framework / stackable LSMs (covered in `security/lsm.md` Tier-3)
- AppArmor / SELinux / Smack (separate LSMs; not Landlock UAPI)
- seccomp-bpf (covered in `kernel/seccomp.md` Tier-3)
- user-namespace credential interaction (covered in `kernel/user-namespace.md` Tier-3)
- man-pages `landlock(7)` / `landlock_create_ruleset(2)` / `landlock_add_rule(2)` / `landlock_restrict_self(2)` prose (upstream copy)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `sys_landlock_create_ruleset` | per-process ruleset fd alloc | `Landlock::sys_create_ruleset` |
| `sys_landlock_add_rule` | per-ruleset rule append | `Landlock::sys_add_rule` |
| `sys_landlock_restrict_self` | per-task irrevocable enforce | `Landlock::sys_restrict_self` |
| `struct landlock_ruleset_attr` | per-ruleset handled-bitmasks | `uapi::landlock::RulesetAttr` |
| `struct landlock_path_beneath_attr` | per-rule: path hierarchy | `uapi::landlock::PathBeneathAttr` |
| `struct landlock_net_port_attr` | per-rule: TCP port | `uapi::landlock::NetPortAttr` |
| `enum landlock_rule_type` | discriminator: PATH_BENEATH / NET_PORT | `uapi::landlock::RuleType` |
| `LANDLOCK_CREATE_RULESET_VERSION` (1<<0) | per-discovery: ABI version | shared |
| `LANDLOCK_CREATE_RULESET_ERRATA` (1<<1) | per-discovery: errata bitmask | shared |
| `LANDLOCK_RESTRICT_SELF_LOG_SAME_EXEC_OFF` (1<<0) | per-restrict: log policy | shared |
| `LANDLOCK_RESTRICT_SELF_LOG_NEW_EXEC_ON` (1<<1) | per-restrict: log policy | shared |
| `LANDLOCK_RESTRICT_SELF_LOG_SUBDOMAINS_OFF` (1<<2) | per-restrict: log policy | shared |
| `LANDLOCK_RESTRICT_SELF_TSYNC` (1<<3) | per-restrict: thread-sync | shared |
| `LANDLOCK_ACCESS_FS_*` | per-FS access bits (17 bits, 1<<0..1<<16) | shared |
| `LANDLOCK_ACCESS_NET_*` | per-NET access bits (1<<0, 1<<1) | shared |
| `LANDLOCK_SCOPE_*` | per-scope bits (1<<0, 1<<1) | shared |

### abi surface (constants + structs)

### Struct `landlock_ruleset_attr`

```
struct landlock_ruleset_attr {
    __u64 handled_access_fs;   /* bitmask of LANDLOCK_ACCESS_FS_* the ruleset will handle */
    __u64 handled_access_net;  /* bitmask of LANDLOCK_ACCESS_NET_* the ruleset will handle */
    __u64 scoped;              /* bitmask of LANDLOCK_SCOPE_* IPC scopes */
};
```

- `LANDLOCK_ACCESS_FS_REFER` is *always* denied by default once any FS access is handled, even when its bit is absent from `handled_access_fs`. To grant `REFER`, callers must set the bit and add per-path-beneath rules that include it.
- The struct can grow in future ABI versions; the size is passed explicitly to `landlock_create_ruleset` and trailing bytes must be zero.

### Filesystem access bits (`__u64`)

| Bit | Constant | ABI version | Applies to |
|---|---|---|---|
| 1ULL << 0  | `LANDLOCK_ACCESS_FS_EXECUTE`     | 1 | file |
| 1ULL << 1  | `LANDLOCK_ACCESS_FS_WRITE_FILE`  | 1 | file (often combined with `TRUNCATE`) |
| 1ULL << 2  | `LANDLOCK_ACCESS_FS_READ_FILE`   | 1 | file |
| 1ULL << 3  | `LANDLOCK_ACCESS_FS_READ_DIR`    | 1 | directory |
| 1ULL << 4  | `LANDLOCK_ACCESS_FS_REMOVE_DIR`  | 1 | directory content (rmdir, rename empty dir) |
| 1ULL << 5  | `LANDLOCK_ACCESS_FS_REMOVE_FILE` | 1 | directory content (unlink, rename) |
| 1ULL << 6  | `LANDLOCK_ACCESS_FS_MAKE_CHAR`   | 1 | directory content (mknod char) |
| 1ULL << 7  | `LANDLOCK_ACCESS_FS_MAKE_DIR`    | 1 | directory content (mkdir, rename in) |
| 1ULL << 8  | `LANDLOCK_ACCESS_FS_MAKE_REG`    | 1 | directory content (creat, open O_CREAT, rename in, link) |
| 1ULL << 9  | `LANDLOCK_ACCESS_FS_MAKE_SOCK`   | 1 | directory content (bind AF_UNIX) |
| 1ULL << 10 | `LANDLOCK_ACCESS_FS_MAKE_FIFO`   | 1 | directory content (mknod FIFO) |
| 1ULL << 11 | `LANDLOCK_ACCESS_FS_MAKE_BLOCK`  | 1 | directory content (mknod block) |
| 1ULL << 12 | `LANDLOCK_ACCESS_FS_MAKE_SYM`    | 1 | directory content (symlink) |
| 1ULL << 13 | `LANDLOCK_ACCESS_FS_REFER`       | 2 | dir source/dest (link, rename across) |
| 1ULL << 14 | `LANDLOCK_ACCESS_FS_TRUNCATE`    | 3 | file (truncate, ftruncate, open O_TRUNC, creat) |
| 1ULL << 15 | `LANDLOCK_ACCESS_FS_IOCTL_DEV`   | 5 | char/block device (ioctl, except generic-safe) |
| 1ULL << 16 | `LANDLOCK_ACCESS_FS_RESOLVE_UNIX`| 9 | AF_UNIX pathname connect/sendmsg target |

### Network access bits (`__u64`)

| Bit | Constant | ABI version | Semantics |
|---|---|---|---|
| 1ULL << 0 | `LANDLOCK_ACCESS_NET_BIND_TCP`    | 4 | bind a TCP socket to the rule's port |
| 1ULL << 1 | `LANDLOCK_ACCESS_NET_CONNECT_TCP` | 4 | connect a TCP socket to the rule's port |

### Scope bits (`__u64`)

| Bit | Constant | ABI version | Semantics |
|---|---|---|---|
| 1ULL << 0 | `LANDLOCK_SCOPE_ABSTRACT_UNIX_SOCKET` | 6 | block connect to abstract AF_UNIX created outside this domain |
| 1ULL << 1 | `LANDLOCK_SCOPE_SIGNAL`               | 6 | block kill/tgkill to tasks outside this domain |

### Rule-type enum and per-rule attribute structs

```
enum landlock_rule_type {
    LANDLOCK_RULE_PATH_BENEATH = 1,
    LANDLOCK_RULE_NET_PORT     = 2,
};

struct landlock_path_beneath_attr {
    __u64 allowed_access;   /* subset of the ruleset's handled_access_fs */
    __s32 parent_fd;        /* fd (preferably O_PATH) anchoring the hierarchy */
} __attribute__((packed));

struct landlock_net_port_attr {
    __u64 allowed_access;   /* subset of the ruleset's handled_access_net */
    __u64 port;             /* host-endian TCP port number; 0 = ephemeral bind */
};
```

The `__attribute__((packed))` on `landlock_path_beneath_attr` is part of the ABI (no trailing pad bytes). `build_check_abi()` in `security/landlock/syscalls.c` static-asserts the layout.

### `landlock_create_ruleset` flags

| Bit | Constant | Semantics |
|---|---|---|
| 1U << 0 | `LANDLOCK_CREATE_RULESET_VERSION` | return the highest supported Landlock ABI version (>=1) instead of creating a ruleset; `attr` MUST be NULL and `size` MUST be 0 |
| 1U << 1 | `LANDLOCK_CREATE_RULESET_ERRATA`  | return a bitmask of fixed-issue indices for the currently running ABI version; `attr` MUST be NULL and `size` MUST be 0 |

### `landlock_restrict_self` flags

| Bit | Constant | Semantics |
|---|---|---|
| 1U << 0 | `LANDLOCK_RESTRICT_SELF_LOG_SAME_EXEC_OFF` | disable audit logging of denials originating from the calling thread (and its no-execve descendants) |
| 1U << 1 | `LANDLOCK_RESTRICT_SELF_LOG_NEW_EXEC_ON`   | enable audit logging of denials after a subsequent execve |
| 1U << 2 | `LANDLOCK_RESTRICT_SELF_LOG_SUBDOMAINS_OFF` | disable audit logging of denials originating from nested Landlock domains created by descendants; combinable with `ruleset_fd == -1` to mute without enforcement |
| 1U << 3 | `LANDLOCK_RESTRICT_SELF_TSYNC`             | atomically apply the new domain (and the log flags) to every thread of the calling process; also propagates `no_new_privs` if set on the caller |

### ABI version table (per `security/landlock/syscalls.c`)

| ABI version | New feature(s) |
|---|---|
| 1 | initial FS access bits (`EXECUTE` .. `MAKE_SYM`) |
| 2 | `LANDLOCK_ACCESS_FS_REFER` |
| 3 | `LANDLOCK_ACCESS_FS_TRUNCATE` |
| 4 | `LANDLOCK_ACCESS_NET_BIND_TCP` / `CONNECT_TCP` (+ `handled_access_net`) |
| 5 | `LANDLOCK_ACCESS_FS_IOCTL_DEV` |
| 6 | `LANDLOCK_SCOPE_ABSTRACT_UNIX_SOCKET`, `LANDLOCK_SCOPE_SIGNAL` (+ `scoped`) |
| 7 | `LANDLOCK_RESTRICT_SELF_*` log-policy flags, `LANDLOCK_CREATE_RULESET_ERRATA` |
| 8 | (placeholder for future minor) |
| 9 | `LANDLOCK_ACCESS_FS_RESOLVE_UNIX` |

(The Rookery baseline targets a minimum of ABI version 7; version queries above 7 return the kernel's highest implemented version per `LANDLOCK_CREATE_RULESET_VERSION`.)

### compatibility contract

REQ-1: `sys_landlock_create_ruleset(attr, size, flags)`:
- /* Discovery short-circuit */
- if flags & LANDLOCK_CREATE_RULESET_VERSION:
  - if attr != NULL ∨ size != 0 ∨ flags != LANDLOCK_CREATE_RULESET_VERSION: return -EINVAL.
  - return current_landlock_abi_version (i32 ≥ 1).
- if flags & LANDLOCK_CREATE_RULESET_ERRATA:
  - if attr != NULL ∨ size != 0 ∨ flags != LANDLOCK_CREATE_RULESET_ERRATA: return -EINVAL.
  - return errata_bitmask_for_current_abi (i32 ≥ 0).
- if flags != 0: return -EINVAL.    /* future-flag-smuggle defense */
- /* Validate attr */
- if !attr ∨ size < offsetofend(struct landlock_ruleset_attr, handled_access_fs): return -EFAULT/-EINVAL.
- if size > sizeof(struct landlock_ruleset_attr) ∧ has_nonzero_trailing_bytes(attr, size): return -E2BIG.
- copy_struct_from_user(&kattr, sizeof(kattr), attr, size).
- /* Per-field validation */
- if kattr.handled_access_fs & ~LANDLOCK_MASK_ACCESS_FS_ALL: return -EINVAL.
- if kattr.handled_access_net & ~LANDLOCK_MASK_ACCESS_NET_ALL: return -EINVAL.
- if kattr.scoped & ~LANDLOCK_MASK_SCOPE_ALL: return -EINVAL.
- if (kattr.handled_access_fs | kattr.handled_access_net | kattr.scoped) == 0: return -ENOMSG.
- /* Allocate ruleset */
- ruleset = landlock_create_ruleset(kattr).
- fd = anon_inode_getfd("[landlock-ruleset]", &ruleset_fops, ruleset, O_RDWR | O_CLOEXEC).
- return fd.

REQ-2: `sys_landlock_add_rule(ruleset_fd, rule_type, rule_attr, flags)`:
- if flags != 0: return -EINVAL.
- ruleset = ruleset_from_fd(ruleset_fd).ok_or(-EBADF)?.
- /* Must not yet be enforced */
- ruleset.must_be_open_for_add().ok_or(-EINVAL)?.
- switch rule_type:
  - case LANDLOCK_RULE_PATH_BENEATH:
    - copy_from_user(&p, rule_attr, sizeof(struct landlock_path_beneath_attr)).
    - if p.allowed_access == 0: return -ENOMSG.
    - if p.allowed_access & ~ruleset.handled_access_fs: return -EINVAL.
    - parent_path = fd_to_path(p.parent_fd).ok_or(-EBADFD)?.
    - landlock_append_fs_rule(ruleset, parent_path, p.allowed_access).
  - case LANDLOCK_RULE_NET_PORT:
    - copy_from_user(&n, rule_attr, sizeof(struct landlock_net_port_attr)).
    - if n.allowed_access == 0: return -ENOMSG.
    - if n.allowed_access & ~ruleset.handled_access_net: return -EINVAL.
    - if n.port > U16_MAX: return -EINVAL.
    - landlock_append_net_rule(ruleset, n.port as u16, n.allowed_access).
  - default: return -EINVAL.
- return 0.

REQ-3: `sys_landlock_restrict_self(ruleset_fd, flags)`:
- /* Validate flags */
- if flags & ~(LANDLOCK_RESTRICT_SELF_LOG_SAME_EXEC_OFF | LANDLOCK_RESTRICT_SELF_LOG_NEW_EXEC_ON | LANDLOCK_RESTRICT_SELF_LOG_SUBDOMAINS_OFF | LANDLOCK_RESTRICT_SELF_TSYNC): return -EINVAL.
- /* Pre-condition: no_new_privs */
- if !task_no_new_privs(current) ∧ !capable(CAP_SYS_ADMIN): return -EPERM.
- /* Special: ruleset_fd == -1 with SUBDOMAINS_OFF: log-mute only */
- if ruleset_fd == -1:
  - if flags & ~(LANDLOCK_RESTRICT_SELF_LOG_SUBDOMAINS_OFF | LANDLOCK_RESTRICT_SELF_TSYNC): return -EINVAL.
  - apply_log_flags_only(current, flags).
  - if flags & LANDLOCK_RESTRICT_SELF_TSYNC: propagate_to_siblings(current, flags).
  - return 0.
- ruleset = ruleset_from_fd(ruleset_fd).ok_or(-EBADF)?.
- new_domain = landlock_merge_or_create_domain(current_landlock_domain(current), ruleset, flags).
- /* TSYNC */
- if flags & LANDLOCK_RESTRICT_SELF_TSYNC:
  - for thread in current.process.threads: landlock_attach_domain(thread, new_domain, flags).
  - if task_no_new_privs(current): for thread: set_no_new_privs(thread).
- else: landlock_attach_domain(current, new_domain, flags).
- return 0.

REQ-4: Per-PATH_BENEATH semantics:
- A rule grants `allowed_access` for the inode `parent_fd` resolves to *and every descendant* of that inode in the directory hierarchy (per `__d_path` walk; bind-mount and overlay-mount aware).
- File-only access bits (e.g. `READ_FILE`, `WRITE_FILE`, `EXECUTE`, `TRUNCATE`, `IOCTL_DEV`, `RESOLVE_UNIX`) on a directory parent grant the right to those operations on files *beneath* the directory.
- Directory-only access bits (`READ_DIR`) apply to the directory itself plus subdirectories.
- Directory-content bits (`REMOVE_*`, `MAKE_*`) apply to the parent's children (subject to the same hierarchy walk).
- `REFER` requires the bit to be set on both source and destination directories of a link/rename across hierarchies, plus the appropriate `MAKE_*` / `REMOVE_*` at the destination/source.

REQ-5: Per-`LANDLOCK_ACCESS_FS_REFER` default-deny:
- Even if `handled_access_fs` does NOT include `REFER`, link/rename across hierarchies is denied. Setting the bit and adding rules is the only way to permit reparenting.

REQ-6: Per-NET_PORT semantics:
- A rule allows `bind` or `connect` to the exact TCP port `port` (host endian, 0..65535).
- `port == 0` with `BIND_TCP` allows binding to the ephemeral range as kernel-assigned.

REQ-7: Per-SCOPE semantics:
- `LANDLOCK_SCOPE_ABSTRACT_UNIX_SOCKET`: `connect(2)` to an abstract `AF_UNIX` socket created outside the calling task's Landlock domain returns `-ECONNREFUSED` (or `-EPERM` per current behaviour); sockets created within the same domain remain accessible.
- `LANDLOCK_SCOPE_SIGNAL`: `kill(2)` / `tgkill(2)` / `pidfd_send_signal(2)` targeting a task outside the domain returns `-EPERM`; in-domain signals proceed.

REQ-8: Per-domain-stacking semantics:
- `landlock_restrict_self` is monotonic: any subsequent restrict can only *narrow* effective access, never widen it.
- A domain is per-credential (`struct cred`); subsequent `execve(2)` carries the domain across (with logging flags re-evaluated per `LOG_SAME_EXEC_OFF` / `LOG_NEW_EXEC_ON`).

REQ-9: Per-`task_no_new_privs` requirement:
- `landlock_restrict_self` requires `PR_SET_NO_NEW_PRIVS == 1` on the caller (or CAP_SYS_ADMIN); without it, returns `-EPERM`.

REQ-10: Per-ruleset-fd lifetime:
- Once `landlock_restrict_self` succeeds, additional `landlock_add_rule` on the same fd is still legal but has no effect on the already-enacted domain (the domain captured a snapshot).
- The ruleset fd is `O_CLOEXEC` by default. Closing it after restrict-self does NOT undo the restriction.

REQ-11: Per-version discovery contract:
- `landlock_create_ruleset(NULL, 0, LANDLOCK_CREATE_RULESET_VERSION)` returns an integer ABI version ≥ 1 monotonically across kernel versions.
- Userspace MUST check this before attempting flags introduced in later versions.

REQ-12: Per-errata bitmask:
- `LANDLOCK_CREATE_RULESET_ERRATA` returns a bitmask of indices (each bit = one specific bug fixed) applicable to the current ABI version; semantics per `Documentation/userspace-api/landlock.rst`.

REQ-13: Per-TSYNC atomicity:
- With `LANDLOCK_RESTRICT_SELF_TSYNC`, all sibling threads' credentials are updated atomically; partial application is not observable (rolled back on failure).
- If any sibling thread already runs a more-restrictive Landlock domain, that thread's domain remains intact (TSYNC is "at least as restrictive").

REQ-14: Per-log-policy flag interaction:
- `LOG_SAME_EXEC_OFF` and `LOG_NEW_EXEC_ON` apply to the newly created domain only.
- `LOG_SUBDOMAINS_OFF` applies only to *future* nested domains.
- Default (no flag set): denials originating from the program creating its own domain are logged; denials in inherited domains created by ancestors are not.

REQ-15: Per-`struct landlock_path_beneath_attr` ABI lock:
- The struct is `__attribute__((packed))`; kernel reads exactly `sizeof(struct landlock_path_beneath_attr)` bytes. Trailing padding is forbidden; trailing junk yields `-E2BIG` by ABI (per `build_check_abi`).

REQ-16: Per-`struct landlock_net_port_attr` port domain:
- `port` is `__u64` on the wire but kernel rejects values > `U16_MAX` (-EINVAL).

REQ-17: Per-rule-type forward-compatibility:
- Unknown `rule_type` values return `-EINVAL`.

REQ-18: Per-resource-cap:
- Per-ruleset rule count is implementation-bounded (allocator-pressure-driven); excessive rule appends may return `-ENOMEM`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `create_ruleset_flag_validation` | INVARIANT | sys_create_ruleset rejects flags outside {VERSION, ERRATA, 0}. |
| `create_ruleset_discovery_no_attr` | INVARIANT | VERSION / ERRATA reject non-zero size or non-null attr. |
| `handled_access_fs_subset_of_known` | INVARIANT | sys_create_ruleset rejects bits outside LANDLOCK_MASK_ACCESS_FS_ALL. |
| `handled_access_net_subset_of_known` | INVARIANT | sys_create_ruleset rejects bits outside LANDLOCK_MASK_ACCESS_NET_ALL. |
| `scoped_subset_of_known` | INVARIANT | sys_create_ruleset rejects bits outside LANDLOCK_MASK_SCOPE_ALL. |
| `empty_ruleset_rejected` | INVARIANT | sys_create_ruleset returns ENOMSG if all three masks are 0. |
| `add_rule_allowed_subset_of_handled` | INVARIANT | per-rule: allowed_access ⊆ handled_access_{fs,net}. |
| `path_beneath_packed_size` | INVARIANT | sizeof(struct landlock_path_beneath_attr) == 12 (4+8 packed). |
| `net_port_u16_only` | INVARIANT | LandlockNetPortAttr.port > u16::MAX ⇒ EINVAL. |
| `restrict_self_no_new_privs` | INVARIANT | restrict_self requires no_new_privs ∨ CAP_SYS_ADMIN. |
| `restrict_self_monotonic` | INVARIANT | post-restrict, effective domain is at-least-as-restrictive as pre-state. |
| `tsync_atomic` | INVARIANT | TSYNC: all threads transition together, no observable partial state. |
| `refer_default_deny` | INVARIANT | cross-hierarchy link/rename denied if REFER not explicitly allowed by every layer. |
| `seal_after_restrict` | INVARIANT | ruleset.sealed=true after restrict; further add_rule has no effect on attached domain. |
| `domain_per_cred_immutable` | INVARIANT | LandlockDomain bound to cred is never widened, only narrowed. |

### Layer 2: TLA+

`uapi/landlock.tla`:
- Per-ruleset creation + per-rule append + per-domain stacking + per-cred attach + per-execve transition.
- Properties:
  - `safety_monotonic_narrowing` — for any task: domain layers can only be appended, never removed.
  - `safety_per_layer_handled_immutable` — once a ruleset is created, handled_access_{fs,net} / scoped are immutable.
  - `safety_seal_then_no_widen` — after sys_restrict_self, additional add_rule on the same fd does not affect any attached domain.
  - `safety_refer_default_deny` — any cross-hierarchy link/rename is denied unless REFER is allowed by every layer.
  - `safety_no_new_privs_required` — restrict_self only succeeds when no_new_privs ∨ CAP_SYS_ADMIN.
  - `safety_tsync_atomic` — TSYNC: all threads observe the new domain at the same logical step.
  - `safety_execve_carries_domain` — execve preserves the credential's Landlock domain.
  - `liveness_audit_log_emitted` — every denial whose log flags permit emits exactly one audit record.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Landlock::sys_create_ruleset` post: discovery ⊥ ruleset (mutually exclusive return modes) | `Landlock::sys_create_ruleset` |
| `Landlock::sys_add_rule` post: allowed_access ⊆ ruleset.handled_access_{fs,net} | `Landlock::sys_add_rule` |
| `Landlock::sys_restrict_self` post: current.cred.landlock_domain narrower-or-equal pre | `Landlock::sys_restrict_self` |
| `Landlock::check_fs` post: returns EACCES iff some handled bit is not granted by ancestor chain | `Landlock::check_fs` |
| `Landlock::check_net` post: returns EACCES iff requested net bit not allowed for port in any layer | `Landlock::check_net` |
| `Landlock::check_scope_signal` post: returns EPERM iff target outside any layer's domain | `Landlock::check_scope_signal` |
| `Landlock::check_scope_unix_abstract` post: returns ECONNREFUSED iff abstract socket created outside any layer's domain | `Landlock::check_scope_unix_abstract` |
| `Ruleset::sealed` monotonicity | `Ruleset` |

### Layer 4: Verus/Creusot functional

`Per-landlock_create_ruleset(attr, size, flags) → per-landlock_add_rule(fd, type, attr, 0) → per-landlock_restrict_self(fd, flags) → per-LSM hooks (path_*, socket_bind/connect, signal)` semantic equivalence: per-`Documentation/userspace-api/landlock.rst` + man-page `landlock(7)` / `landlock_create_ruleset(2)` / `landlock_add_rule(2)` / `landlock_restrict_self(2)`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

Landlock reinforcement:

- **Per-`handled_access_fs` reserved-bit rejection** — defense against per-future-bit smuggle into older kernels.
- **Per-`handled_access_net` reserved-bit rejection** — defense against per-future-bit smuggle.
- **Per-`scoped` reserved-bit rejection** — defense against per-future-bit smuggle.
- **Per-rule `allowed_access ⊆ handled_access` enforcement** — defense against per-rule-grants-unhandled-bit confusion.
- **Per-`LANDLOCK_ACCESS_FS_REFER` default-deny** — defense against per-implicit reparent leak.
- **Per-`task_no_new_privs` gate on `restrict_self`** — defense against per-setuid-privilege-shedding.
- **Per-monotonic domain stacking** — defense against per-domain-widening attack.
- **Per-seal-after-restrict** — defense against per-late-rule injection into attached domain.
- **Per-`O_CLOEXEC` ruleset fd** — defense against per-execve fd inheritance leak.
- **Per-TSYNC atomic across threads** — defense against per-thread-bypass via sibling unrestricted thread.
- **Per-`copy_struct_from_user` size-versioning** — defense against per-attr-grow-mid-syscall confusion.
- **Per-`landlock_path_beneath_attr` packed-layout static-assert** — defense against per-padding infoleak.
- **Per-`landlock_net_port_attr.port > u16::MAX` rejection** — defense against per-port-overflow.
- **Per-Landlock-domain audit logging (per-log-flag policy)** — defense against per-silent-denial.
- **Per-LSM-hook (path_*, socket_bind/connect, signal, file_open) integration** — defense against per-bypass via unhooked path.
- **Per-credential domain (struct cred)** — defense against per-task-domain leak across credential change.

### grsecurity/pax-style reinforcement

- **PaX UDEREF / USERCOPY**: every Landlock syscall ingress (`attr`, `rule_attr`, `parent_fd` dereference into the fdtable) is mediated by `copy_struct_from_user` / `copy_from_user_packed` / `get_unused_fd_flags`. No raw `__user` pointer deref. The packed `landlock_path_beneath_attr` is copied as exactly `sizeof(struct landlock_path_beneath_attr)` bytes; trailing junk yields `-E2BIG` instead of being silently ignored.
- **PAX_RANDKSTACK**: per-syscall entry of `landlock_create_ruleset` / `landlock_add_rule` / `landlock_restrict_self` randomizes the kernel-stack offset. Defends against any pointer-spray reconnaissance against `struct landlock_ruleset` from an unprivileged caller (these syscalls are the *one* set explicitly designed for unprivileged use).
- **GRKERNSEC_PROC restriction on /proc/<pid>/status**: the per-task Landlock domain count (`Landlock: <n>`) and per-credential domain pointer leak should be restricted under `GRKERNSEC_PROC_USER` / `_USERGROUP` to the owning UID — exposing it cross-UID lets a probe enumerate which neighbours are sandboxed.
- **GRKERNSEC_HARDEN_PTRACE**: Landlock domains are inherited across `execve(2)` *and* `fork(2)`; harden so that a `ptrace`-attached child of a Landlock-restricted process cannot escape via `ptrace_may_access` of a less-restricted parent (the LSM hook `security_ptrace_access_check` MUST consult Landlock as well as the standard YAMA/PaX gates).
- **Landlock as a complement to PaX MPROTECT**: PaX MPROTECT prevents creating writable + executable memory; Landlock additionally prevents the sandboxed process from `open(O_WRONLY)` on `/proc/self/mem` (subject to handled FS bits) and from `bind/connect` on arbitrary TCP ports. Together, they raise the bar on JIT-ROP and arbitrary-port C2 channels.
- **GRKERNSEC_NO_FIFO interaction**: a Landlock domain that does not grant `LANDLOCK_ACCESS_FS_MAKE_FIFO` plus the grsec `GRKERNSEC_NO_FIFO` policy double-locks pipe-based confused-deputy vectors in shared `/tmp`; denials are surfaced via Landlock audit (when log flags allow), separately from grsec's own log.
- **fanotify FAN_UNLIMITED_QUEUE / FAN_UNLIMITED_MARKS as security-hostile**: a Landlock-restricted process MUST NOT be able to call `fanotify_init` with those flags even if it somehow holds CAP_SYS_ADMIN inside a user namespace; the LSM hook for `fanotify_init` consults the Landlock domain and rejects perm-class / unlimited-resource modes for sandboxed tasks.
- **inotify IN_ATTRIB on protected files (chattr +i / +a) — covered by PaX UDEREF + Landlock**: a Landlock domain that does not grant any FS rights to `/etc` cannot install an `inotify_add_watch` on `/etc/shadow` and so cannot use IN_ATTRIB as a side-channel; UDEREF guarantees the pathname copy itself is safe.
- **Landlock as namespace-confining — additive to user-namespace gates**: grsec policy treats Landlock domain depth as an additional credential identity for `security_capable` / `security_capset`. A user-namespace `CAP_SYS_ADMIN` granted inside a Landlock-restricted domain is treated as if `no_new_privs` were set even when the unshare flow would otherwise loosen it.
- **Per-fd close-on-exec mandatory**: the ruleset fd is `O_CLOEXEC` by default in upstream; grsec rejects callers that attempt to disable it via `fcntl(F_SETFD, 0)` once `landlock_restrict_self` has been called.
- **Per-`LANDLOCK_RESTRICT_SELF_LOG_*` mandatory minimum logging in hardened builds**: grsec ignores `LOG_SAME_EXEC_OFF` / `LOG_SUBDOMAINS_OFF` when `grsec_enable_audit_landlock_denials == 1` (so attackers cannot mute their own denial trail).
- **PaX-style monotonicity invariant**: `landlock_restrict_self` is provably narrowing; grsec adds a one-way "Landlock-active" `task->flags` bit that, once set, prevents the same task from ever invoking `setresuid` / `setresgid` to a UID outside the original UID set — equivalent to an implicit `PR_SET_KEEPCAPS=0`.
- **Per-real-UID resource accounting**: ruleset memory and rule-count are charged to the real UID's per-user memcg; defense against per-fork-bomb of restrict-self callers.
- **`LANDLOCK_CREATE_RULESET_VERSION` / `_ERRATA` discovery rate-limited**: per-task token bucket prevents tight version-probe loops from being a DoS vector.
- **Audit-log every `EACCES` from Landlock FS hook with full inode path + handled+missing bits** — defense against per-silent-denial debugging dead-end.

