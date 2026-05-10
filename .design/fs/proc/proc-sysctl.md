# Tier-3: fs/proc/proc-sysctl — /proc/sys/* sysctl bridge

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/proc/proc_sysctl.c
  - kernel/sysctl.c
  - include/linux/sysctl.h
  - include/uapi/linux/sysctl.h
-->

## Summary
Tier-3 design for the `/proc/sys/*` sysctl bridge — the procfs-based interface to kernel-tunable parameters. Per-subsystem `register_sysctl()` registrations build a hierarchical tree of `ctl_table` entries; `proc_sysctl.c` exposes them as procfs files where each file's read returns the current value (formatted per-handler) and write parses + applies the new value (with per-handler validation).

Critical compatibility surface: every distro / cloud / container runtime / sysctl.d configuration depends on `/proc/sys/*` paths being byte-identical. A misnamed sysctl breaks `sysctl -p` on every boot. Specific examples that must match:
- `/proc/sys/net/ipv4/ip_forward`, `/tcp_max_syn_backlog`, `/somaxconn`, `/tcp_keepalive_*`
- `/proc/sys/net/core/somaxconn`, `/wmem_max`, `/rmem_max`
- `/proc/sys/kernel/randomize_va_space`, `/sched_*`, `/printk*`, `/sysrq`, `/dmesg_restrict`, `/kptr_restrict`
- `/proc/sys/vm/swappiness`, `/dirty_ratio`, `/dirty_background_ratio`, `/overcommit_memory`
- `/proc/sys/fs/file-max`, `/inotify/max_user_instances`, `/aio-max-nr`
- `/proc/sys/abi/*` per-arch ABI-tunables

Sub-tier-3 of `fs/proc/00-overview.md`. Pairs with `kernel/sysctl.c` (legacy global sysctl table — modern subsys use per-subsys `register_sysctl_table` paths).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| /proc/sys procfs bridge: register_sysctl, ctl_table dispatch, per-handler get/set | `fs/proc/proc_sysctl.c` |
| Legacy global sysctl tables (root-tree, kernel.*, vm.*, fs.*, debug.*) | `kernel/sysctl.c` |
| Public API | `include/linux/sysctl.h` |
| UAPI: legacy `sysctl(2)` syscall constants (deprecated) | `include/uapi/linux/sysctl.h` |

## Compatibility contract

### `struct ctl_table` (per-entry registration)

```c
struct ctl_table {
    const char *procname;               /* path component */
    void *data;                         /* pointer to value */
    int maxlen;                         /* size of *data */
    umode_t mode;                       /* permissions */
    struct ctl_table *child;            /* DEPRECATED — use register_sysctl_paths */
    proc_handler *proc_handler;         /* read/write handler */
    struct ctl_table_poll *poll;
    void *extra1;                       /* per-handler extra (often min/max) */
    void *extra2;
};
```

Layout-byte-identical so existing kernel-module sysctl-table static instances work.

### Standard `proc_handler`s

| Handler | Behavior |
|---|---|
| `proc_dointvec` | Read/write decimal int; no validation |
| `proc_dointvec_minmax` | Read/write decimal int with [min, max] bound |
| `proc_dointvec_jiffies` | Read/write seconds; converts to/from jiffies |
| `proc_dointvec_userhz_jiffies` | Read/write USER_HZ ticks; converts |
| `proc_dointvec_ms_jiffies` | Read/write milliseconds; converts |
| `proc_doulongvec_minmax` | Read/write decimal ulong with [min, max] |
| `proc_doulongvec_ms_jiffies_minmax` | ulong ms↔jiffies w/ bounds |
| `proc_dostring` | Read/write string |
| `proc_dobool` | Read/write 0/1 |
| `proc_douintvec_minmax` | Read/write decimal uint with bounds |
| `proc_dou8vec_minmax` | Read/write u8 with bounds |
| `proc_doupage_minmax` | Read/write `unsigned long` page counts with bounds |

Plus per-subsys custom handlers (e.g., `proc_do_static_key` for static-branches).

Per-handler dispatch byte-identical so existing sysctl-tunable values + min/max behavior matches.

### `register_sysctl()` / `register_sysctl_paths()` / `unregister_sysctl_table()`

Modern API:
```c
struct ctl_table_header *register_sysctl(const char *path, const struct ctl_table *table);
struct ctl_table_header *register_sysctl_init(const char *path, struct ctl_table *table, const char *table_name);
void unregister_sysctl_table(struct ctl_table_header *header);
```

Per-subsys-module register; per-netns variants for net.* sysctls; on unregister, RCU-defer free.

### Per-netns sysctl trees

Per-netns sysctl tables for `net.*` subtrees: each netns has its own copy of writable per-netns sysctls (e.g., `net.ipv4.ip_forward`) — modifying in netns A doesn't affect netns B. Identical per-netns scoping.

### `/proc/sys` directory structure

Top-level subdirs:
```
/proc/sys/
├── abi/         (per-arch ABI tunables)
├── crypto/      (crypto subsys)
├── debug/       (debug-only tunables)
├── dev/         (device-specific)
├── fs/          (filesystem subsys)
├── kernel/      (kernel-core tunables)
├── net/         (per-netns network sysctls)
├── sunrpc/      (NFS/SunRPC)
├── user/        (user-namespace defaults)
└── vm/          (memory-management tunables)
```

Identical layout.

### Permissions per `ctl_table.mode`

Standard modes:
- `0644` — readable by all, writable by root (most read-tunable values)
- `0600` — readable by root only (sensitive — e.g., `kernel.kptr_restrict` source value)
- `0444` — read-only

Identical mode set.

### `extra1` / `extra2` for min/max bounds

Per-handler convention:
- `proc_dointvec_minmax`: `extra1` = `&min`, `extra2` = `&max`
- `proc_dostring`: `extra1` = `&max_len_int`
- Custom handlers: per-handler interpretation

### `sysctl(2)` syscall (deprecated)

The legacy `sysctl(2)` syscall is deprecated and removed for new-arch build (`CONFIG_SYSCTL_SYSCALL=n` default since 2.6.x; explicitly removed in 5.5+). UAPI header retained for ABI compatibility. Procfs is the only supported interface.

Per-Rookery: explicitly DO NOT support `sysctl(2)` syscall; only `/proc/sys/*` interface.

### Per-table sysctl write-validation

Each handler does its own validation:
- Out-of-range writes return -EINVAL (or clamp, per-handler)
- Per-permission writes return -EPERM
- Per-trailing-whitespace handling identical (some handlers strip)

## Requirements

- REQ-1: `struct ctl_table` byte-identical layout.
- REQ-2: All standard proc_handlers (per the table) implement identical semantics + format.
- REQ-3: `register_sysctl()` / `register_sysctl_init()` / `unregister_sysctl_table()` semantics identical; per-netns variants.
- REQ-4: Per-netns sysctl scoping for net.* subtree; modifying in netns A doesn't affect B.
- REQ-5: Top-level subdirectories `abi/crypto/debug/dev/fs/kernel/net/sunrpc/user/vm` registered with full per-subsys subtree.
- REQ-6: Permission modes (0644 / 0600 / 0444) honored per-entry.
- REQ-7: `extra1` / `extra2` min/max bound enforcement per-handler.
- REQ-8: Legacy `sysctl(2)` syscall NOT supported (explicit reject — only procfs interface).
- REQ-9: Per-handler write-validation: out-of-range returns -EINVAL; unauthorized returns -EPERM.
- REQ-10: RCU-defer free on `unregister_sysctl_table` to allow concurrent in-flight readers.
- REQ-11: Per-table `poll` notification: when a sysctl is updated, notify pollers (used by some userspace tools waiting for changes).
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct ctl_table` byte-identical layout. (covers REQ-1)
- [ ] AC-2: `sysctl -a` output byte-identical content (modulo per-arch differences). (covers REQ-2, REQ-3, REQ-5)
- [ ] AC-3: per-handler test: `echo 1 > /proc/sys/net/ipv4/ip_forward`; `cat /proc/sys/net/ipv4/ip_forward` returns 1; out-of-range write (-1 or 99999) returns EINVAL. (covers REQ-2, REQ-9)
- [ ] AC-4: minmax test: `echo 65536 > /proc/sys/net/core/somaxconn` (max); `echo 65537 >` returns EINVAL. (covers REQ-7)
- [ ] AC-5: jiffies-conversion test: `echo 30 > /proc/sys/net/ipv4/tcp_keepalive_time`; subsequent read returns 30 (not jiffies). (covers REQ-2)
- [ ] AC-6: per-netns isolation: in netns A `echo 1 > .../ip_forward`; in netns B `cat .../ip_forward` returns 0 (per-netns independent). (covers REQ-4)
- [ ] AC-7: Permission test: `echo X > /proc/sys/kernel/somekey` as non-root → EACCES (when mode=0600 root-only). (covers REQ-6)
- [ ] AC-8: Legacy syscall rejection: invoking `syscall(__NR__sysctl, ...)` returns ENOSYS. (covers REQ-8)
- [ ] AC-9: RCU-defer test: register sysctl + start concurrent reader + unregister; reader sees consistent value (either old or none); no use-after-free. (covers REQ-10)
- [ ] AC-10: Poll test: open sysctl file with O_NONBLOCK; poll(2); on sysctl-write triggers POLLERR/POLLPRI for the poll-aware sysctls. (covers REQ-11)
- [ ] AC-11: distro test: `sysctl -p /etc/sysctl.conf` from a real Linux distro applies all settings without error. (covers REQ-2, REQ-3, REQ-5)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::fs::proc::sysctl::Bridge` — top-level `/proc/sys/*` registration
- `kernel::fs::proc::sysctl::CtlTable` — `struct ctl_table` wrapper
- `kernel::fs::proc::sysctl::Header` — `struct ctl_table_header` registration handle
- `kernel::fs::proc::sysctl::handlers::*` — per-standard-handler (dointvec/minmax/string/etc.)
- `kernel::fs::proc::sysctl::PerNetnsScope` — per-netns sysctl tree
- `kernel::fs::proc::sysctl::Permissions` — per-entry mode enforcement
- `kernel::fs::proc::sysctl::Poll` — sysctl-update notification
- `kernel::fs::proc::sysctl::Validate` — per-handler write-validation

### Locking and concurrency

- **`sysctl_lock`** (per-system mutex): per-system root-table mutator
- **Per-netns table list mutex**: per-netns subtree mutator
- **RCU**: traversal hot path RCU-side; `unregister` uses RCU-defer free
- **Per-`ctl_table_poll` waitqueue**: poll() consumers

### Error handling

- `Err(EINVAL)` — bad value / out-of-range
- `Err(EACCES)` — permission denied
- `Err(EPERM)` — capability-denied write
- `Err(ENOENT)` — sysctl path not registered
- `Err(EAGAIN)` — sysctl table being unregistered

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-handler read/write buffer bounds (no overrun on long values) | `kani::proofs::fs::proc::sysctl::handler_safety` |
| `register_sysctl` insert under sysctl_lock + RCU-defer-free on unregister | `kani::proofs::fs::proc::sysctl::register_safety` |
| Per-netns sysctl tree isolation | `kani::proofs::fs::proc::sysctl::netns_safety` |
| Min/max bound check (signed/unsigned correctness) | `kani::proofs::fs::proc::sysctl::minmax_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-system sysctl tree | every entry has unique path within parent; no cycles | `kani::proofs::fs::proc::sysctl::tree_invariants` |
| Per-netns sysctl tree | per-netns instance independent from others; no cross-leak | `kani::proofs::fs::proc::sysctl::netns_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Sysctl atomicity theorem** via Verus — proves: read-modify-write of sysctl value is atomic from observer's perspective (no half-updated state visible).
- **Min/max enforcement theorem** via Verus — proves: ∀ write w with handler having extra1/2, post-write value ∈ [min, max] OR write returned -EINVAL.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | per-subsys `ctl_table[]` arrays + per-handler `proc_handler` fn-ptr `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed sysctl-write scratch buffers cleared (carry possibly-sensitive values) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, CONSTIFY**: see above
- **USERCOPY**: read/write uses bound-checked accessors; per-handler explicit length check
- **SIZE_OVERFLOW**: min/max comparison + value-parse arithmetic uses checked operators
- **KERNEXEC**: per-handler dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_SYS_ADMIN)` (or per-domain CAP_NET_ADMIN etc.) for sensitive sysctls.
- LSM hook `security_inode_permission` for per-file access.
- Default useful GR-RBAC policy: deny writes to `/proc/sys/kernel/{sysrq,modules_disabled,perf_event_paranoid,kptr_restrict,dmesg_restrict,unprivileged_bpf_disabled}` outside gradm-marked `kernel_admin` role; these sysctls are direct security-control surfaces.

### Userspace-visible behavior changes

- `sysctl(2)` syscall returns ENOSYS (per REQ-8). Existing code base on `sysctl(2)` is decades-deprecated; modern tooling uses procfs interface only.

### Verification

(See § Verification above.)

## Open Questions

(none — sysctl ABI exhaustively specified by upstream + extensive distro sysctl.d test corpus)

## Out of Scope

- Per-subsys sysctl table contents (cross-ref individual subsystem Tier-3s)
- /proc/sys/net per-netns details (cross-ref `proc-net.md`)
- 32-bit-only paths
- Implementation code
