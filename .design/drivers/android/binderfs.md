# Tier-3: drivers/android/binderfs.c — binderfs pseudo-filesystem for dynamic binder devices

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/android/binderfs.c
  - drivers/android/binder_internal.h
  - include/uapi/linux/android/binderfs.h
-->

## Summary

`binderfs` is a stackable, namespaced pseudo-filesystem that replaces the static `/dev/{binder,hwbinder,vndbinder}` triple with an unlimited set of dynamically-allocated binder devices. AOSP since Android 11 mounts a `binderfs` instance at `/dev/binderfs/` inside the initial mount namespace; container/VM workloads (e.g. anbox, waydroid, Microsoft Android-on-Windows, Chrome OS ARC++) mount their own per-namespace `binderfs` so each Android instance gets an independent binder context space without colliding on global device names.

Each `binderfs` super-block owns: a control device at `binder-control` (handles the `BINDER_CTL_ADD` ioctl that creates new binder character devices on the fly), a `features/` directory exposing kernel-side feature flags (`oneway_spam_detection`, `extended_error`, `freeze_notification`, …), and zero-or-more dynamically created binder devices, each of which is wired into the binder core (cross-ref `binder.md`) as a fresh `binder_context`.

This Tier-3 covers `binderfs.c` (~785 lines: super-block lifecycle, control device, mount options, namespace gating, binder feature surface).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct binderfs_info` | per-super-block state — root user_ns, ipc_ns, device limits | `drivers::android::binderfs::SbInfo` |
| `struct binderfs_mount_opts` | parsed mount options (`max`, `stats`) | `drivers::android::binderfs::MountOpts` |
| `struct binder_features` | kernel feature flag exposure (`features/` subdir) | `drivers::android::binderfs::Features` |
| `binder_fs_type` | `file_system_type` registered for `"binder"` | `BinderFsType` |
| `binderfs_init_fs_context(fc)` | mount path — alloc opts, attach fs_context_ops | `Mount::init_fs_context` |
| `binderfs_fill_super(sb, fc)` | one-time super-block init: alloc root inode, ctl device, features subdir, optional `binder_logs/` | `Mount::fill_super` |
| `binderfs_kill_super(sb)` | unmount path — drain all binder contexts, free `binderfs_info` | `Mount::kill_super` |
| `binderfs_binder_ctl_create(sb)` | create the `binder-control` inode (singleton per sb) | `ControlDev::create` |
| `binder_ctl_ioctl(file, cmd, arg)` | `BINDER_CTL_ADD` handler — alloc minor, register binder device | `ControlDev::ioctl` |
| `binderfs_binder_device_create(ref_inode, dev_userp, opts)` | per-call: ida_alloc minor + register_chrdev + create inode | `Mount::create_device` |
| `binderfs_rename(...)` / `binderfs_unlink(...)` | rename/unlink hooks — refuse modification of `binder-control` | `Mount::rename` / `unlink` |
| `binderfs_evict_inode(inode)` | per-device evict — release binder context + ida_free minor | `Mount::evict_inode` |
| `binderfs_fs_context_parse_param(fc, param)` | parse `max=`, `stats=global` mount options | `MountOpts::parse_param` |
| `init_binder_features(sb)` | create `features/` directory tree | `Features::init` |
| `init_binder_logs(sb)` | optional `binder_logs/` (only when `stats=global` + CAP_SYS_ADMIN) | `Logs::init` |
| `binder_ctl_fops` | `file_operations` for `binder-control` (`unlocked_ioctl`+`compat_ioctl`) | `ControlDev::FileOps` |
| `binderfs_dev` | global `dev_t` reserved for binder devices (per-major + 1M minors) | `BinderfsDev` |
| `binderfs_minors` (IDA) | global minor allocator (`1` … `BINDERFS_MAX_MINOR`) | `MinorIda` |

## Compatibility contract

REQ-1: `binder_fs_type` registered with name `"binder"`; `FS_USERNS_MOUNT` flag set so unprivileged user-namespace owners can mount their own instance.

REQ-2: Mount options parsed via `fs_parameter_spec`:
- `max=<N>`: per-super-block ceiling on dynamically-created binder devices (default `BINDERFS_MAX_MINOR_CAPPED = 1048576`).
- `stats=global`: enable `binder_logs/` directory; reject unless caller is in the initial user_ns AND `CAP_SYS_ADMIN`.

REQ-3: Reconfigure (`mount -o remount`) supported via `fs_context_reconfigure`; only mount-opt changes; refuses to add/remove devices via remount.

REQ-4: `BINDER_CTL_ADD` ioctl ABI (uapi `binderfs.h`):
```
struct binderfs_device {
    char name[BINDERFS_MAX_NAME + 1];   // null-terminated, validated
    __u32 major;                         // output: binderfs_dev MAJOR
    __u32 minor;                         // output: assigned minor
};
#define BINDER_CTL_ADD _IOWR('b', 1, struct binderfs_device)
```
Name length ≤ 255 (`BINDERFS_MAX_NAME`); name must be a valid filesystem name (no `/`, no `..`, not `binder-control`, not `binder_logs`, not `features`).

REQ-5: Per-super-block device count bounded by `MountOpts::max`; exceeded → `-EMFILE` (`-ENOSPC` post-ida_alloc).

REQ-6: `binder-control` inode is mode `0600`, owner = sb's `user_ns` root, ungrouped — only the namespace owner can create devices. The `s_user_ns` of the super-block authorizes operations.

REQ-7: `features/` subdirectory — read-only seq-files; one file per `struct binder_features` boolean.

REQ-8: Per-namespace IPC isolation: the `ipc_namespace` captured at mount time becomes the implicit owner of all binder contexts created via that super-block; binder transactions cannot cross binderfs super-block boundaries.

REQ-9: `binderfs_evict_inode` ordering: drop binder context first (`binder_remove_device`), free `ida_free` minor second, free inode third — defense against ref-after-free.

REQ-10: Refuse rename of `binder-control`, `features`, `binder_logs`; refuse unlink of `binder-control`.

REQ-11: All super-block reads from inside `unshare(CLONE_NEWUSER|CLONE_NEWNS|CLONE_NEWIPC)` yield an empty `binderfs` mount with a fresh control device — container start-up path.

REQ-12: Mount inside a non-init user_ns requires the namespace's owner to have `CAP_SYS_ADMIN` in that user_ns (kernel general policy for `FS_USERNS_MOUNT`).

## Acceptance Criteria

- [ ] AC-1: `mount -t binder binder /dev/binderfs` inside the initial namespace creates `binder-control` + empty device set.
- [ ] AC-2: `ioctl(fd, BINDER_CTL_ADD, &{name="foo"})` produces `/dev/binderfs/foo` as a usable binder character device (cross-ref `binder.md` AC-2).
- [ ] AC-3: `unlink("/dev/binderfs/foo")` releases the binder context, frees the minor, and a subsequent `BINDER_CTL_ADD` of the same name succeeds.
- [ ] AC-4: kselftest `tools/testing/selftests/filesystems/binderfs/binderfs_test` passes.
- [ ] AC-5: Inside `unshare -Umnipfr` an unprivileged user can mount a fresh `binderfs` and `BINDER_CTL_ADD` succeeds, but the device is invisible from the parent namespace.
- [ ] AC-6: Mounting with `stats=global` from inside a non-init user_ns fails `-EPERM`.
- [ ] AC-7: Exceeding `max=N` returns `-EMFILE` and does not leak a minor.
- [ ] AC-8: Rename / unlink of `binder-control` returns `-EPERM`.
- [ ] AC-9: All entries under `features/` parse-able under `getfattr` / `cat` and consistent with `CONFIG_ANDROID_BINDER_*` build flags.

## Architecture

`SbInfo` lives in `drivers::android::binderfs::SbInfo`:

```
struct SbInfo {
  root_uid: KUid,                  // mount-time euid (in s_user_ns)
  root_gid: KGid,
  ipc_ns: Arc<IpcNamespace>,       // captured at fill_super
  control_dentry: Arc<Dentry>,     // binder-control
  device_count: AtomicU32,
  max_devices: u32,                // from MountOpts::max
  log_root_dir: Option<Arc<Dentry>>, // binder_logs/
  proc_log_dir: Option<Arc<Dentry>>,
  features: KBox<Features>,
}

struct MountOpts {
  max: u32,
  stats_mode: BinderfsStatsMode,   // GLOBAL or PROC (default)
}
```

Mount lifecycle `Mount::init_fs_context` → `_get_tree` → `_fill_super`:
1. `init_fs_context` allocates `MountOpts` defaulted to `max = BINDERFS_MAX_MINOR_CAPPED`, `stats = PROC`.
2. `_parse_param` walks `binderfs_fs_parameters` spec table; rejects unknown keys.
3. `_get_tree` invokes `get_tree_nodev(fc, binderfs_fill_super)` — `binderfs` is always a fresh sb per mount call.
4. `_fill_super`:
   a. Validate `stats_mode = GLOBAL` requires `ns_capable(&init_user_ns, CAP_SYS_ADMIN)`.
   b. Allocate `SbInfo`, capture `ipc_ns` from current task.
   c. Build root inode (dir, mode `0755`, sb's user_ns root).
   d. `binderfs_binder_ctl_create(sb)` — alloc `binder-control` inode with `binder_ctl_fops`.
   e. `init_binder_features(sb)` — create `features/` subdir + per-feature seq files.
   f. `init_binder_logs(sb)` — only when `stats_mode == GLOBAL`.

Per-`BINDER_CTL_ADD`:
1. Copy `binderfs_device` from user (bounded by `sizeof(struct)`).
2. Validate name: length 1..255, no `/`, not reserved (`binder-control`, `features`, `binder_logs`).
3. `current_user_ns()` must equal `sb->s_user_ns` (or be a descendant with `CAP_SYS_ADMIN`).
4. Bump `SbInfo::device_count`; if `> max_devices` → `-EMFILE`.
5. `ida_alloc(&binderfs_minors, GFP_KERNEL)` → minor in `1..1048575`.
6. Allocate inode, set i_op/i_fops to `binder_fops` (cross-ref `binder.md`), set i_rdev = `MKDEV(binderfs_dev, minor)`.
7. Create binder context: `binder_alloc_context(name, current_user_ns(), ipc_ns)` — cross-ref `binder.md` `Context`.
8. Link into sb's root directory; write `major`/`minor` back to user.

Per-device eviction `binderfs_evict_inode`:
1. If inode is a binder device: `binder_remove_device(context)` — synchronously drains all `binder_proc` instances bound to the context.
2. `ida_free(&binderfs_minors, minor(inode))`.
3. Decrement `SbInfo::device_count`.

Per-`features/` file:
- One static `binder_features` struct populated at module init. Each field exposed as a 0/1 seq-file; userspace probes feature presence via `read("/dev/binderfs/features/<name>")`.

Unmount `binderfs_kill_super`:
1. Walk root dir; for each remaining binder device inode → `binder_remove_device`.
2. `kfree(SbInfo)`; `kill_litter_super(sb)` for the inode tree.

## Hardening

(Inherits row-1 features from `drivers/android/binder.md` § Hardening.)

binderfs-specific reinforcement:

- **`binder-control` mode 0600** — defense against unprivileged-user device creation.
- **Name allowlist + reserved-name reject** — defense against shadowing `binder-control`/`features`/`binder_logs`.
- **`max=` mount-opt** — defense against unbounded minor consumption DoS.
- **Per-super-block `s_user_ns` enforcement** — control-ioctl checks `current_user_ns()` is sb's owner or descendant.
- **`binderfs_evict_inode` ordering** — context-drain before ida_free; defense against ref-after-free.
- **Refuse rename/unlink of control + features + logs** — defense against confusing the mount root.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab for `binderfs_info` and `binderfs_mount_opts`; the `binderfs_device` ioctl copy is bounded by `sizeof(struct)` with `check_object_size`.
- **PAX_KERNEXEC** — `binder_ctl_fops`, `binderfs_super_ops`, `binderfs_fs_context_ops`, and `binderfs_dir_inode_operations` placed in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset on `binder_ctl_ioctl`, `binderfs_fill_super`, and `binderfs_evict_inode` entries.
- **PAX_REFCOUNT** — saturating refcount on `binderfs_info` and on the binder-context handle held per dynamic device; overflow trap defeats device-create/remove race UAFs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `binderfs_info`, `binderfs_mount_opts`, and per-device inode private data so stale namespace/cred pointers cannot bleed across reuse.
- **PAX_UDEREF** — SMAP/PAN enforced on `binder_ctl_ioctl` `copy_from_user` of `struct binderfs_device`; reject mis-sized payloads.
- **PAX_RAP / kCFI** — `binder_ctl_fops`, `binderfs_super_ops`, and feature seq_file ops marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of per-context pointers in `binder_logs/` behind CAP_SYSLOG; suppress `%p` in stats seq files.
- **GRKERNSEC_DMESG** — restrict `binderfs: ...` super-block init and `BINDER_CTL_ADD` error banners to CAP_SYSLOG.
- **`binder-control` CAP_SYS_ADMIN gate** — control-ioctl requires `CAP_SYS_ADMIN` in the owning user_ns (in addition to mode 0600); defense against unprivileged device-spawn DoS.
- **Per-namespace `max-binder-devices` ulimit** — `MountOpts::max` capped per-user-ns by `RLIMIT_NOFILE`-like accounting; container escapes cannot inflate global minor pool.
- **Namespace-attached binder contexts** — each binder device is attached to `sb->s_user_ns` + captured `ipc_ns`; cross-ns IPC explicitly denied at `binder_transaction` time.
- **Name allowlist** — `BINDERFS_MAX_NAME` length cap, no slashes, no `..`, no reserved names (`binder-control`, `features`, `binder_logs`); defense against parent-traversal and inode shadowing.
- **`stats=global` requires init-userns + CAP_SYS_ADMIN** — debug surface gated; container instances cannot expose global statistics.

Rationale: binderfs is the gate to dynamic binder-context creation — every unprivileged container that wants its own AIDL service-manager goes through it. RAP/kCFI on the file ops table, refcount-overflow trapping on `binderfs_info`, CAP_SYS_ADMIN on the control device, name allowlisting against reserved entries, per-namespace `max` enforcement, and SMAP/PAN on `BINDER_CTL_ADD` convert binderfs from "a tiny pseudo-FS" into a structurally hardened gateway between user-namespace owners and the binder IPC engine.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Binder core IPC (covered in `binder.md`)
- AOSP `servicemanager` userland (out of kernel scope)
- 32-bit binderfs (Rookery is 64-bit only)
- Implementation code
