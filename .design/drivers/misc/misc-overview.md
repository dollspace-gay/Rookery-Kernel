# Tier-3: drivers/misc/ — misc-char framework overview + per-device chardev pattern (dynamic minor allocation, /dev/<name>, miscdevice file-ops dispatch)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/char/misc.c
  - include/linux/miscdevice.h
  - drivers/misc/*
-->

## Summary

The `drivers/misc/` tree is the kernel's catch-all parking lot for character-device drivers that do not belong to any first-class subsystem — small EEPROMs (`eeprom/`, `ds1682.c`), I/O-board MCUs (`atmel-ssc.c`, `c2port`), confidential-computing helpers (`nsm.c`, `open-dice.c`, `tpm` adjuncts), debug taps (`lkdtm`, `dummy-irq.c`, `kgdbts.c`), vendor-specific accelerator front-ends (`fastrpc.c`, `mei`, `bcm-vk`, `mchp_pci1xxxx`, `genwqe`, `lis3lv02d`), and a long tail of dual-purpose helpers. The unifying mechanism is `drivers/char/misc.c` — the misc-char core. Every misc driver populates a `struct miscdevice` and calls `misc_register(misc)`. The core (a) installs a chardev for major `MISC_MAJOR (10)` on first init, (b) allocates a minor number (either fixed from the curated table in `<linux/miscdevice.h>` or dynamically from the pool above `MISC_DYNAMIC_MINOR (255)`), (c) creates a `device_create` node so udev materializes `/dev/<name>`, and (d) routes the chardev `open()` to the per-instance `miscdevice->fops` (replacing `file->f_op` for the file's remaining lifetime). This Tier-3 covers the framework contract, the per-instance pattern that every misc driver follows, the security implications of "anyone can be /dev/foo", and the audit-surface concentration that a single `misc_register` call represents.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct miscdevice` | per-instance descriptor (minor, name, fops, parent, mode, groups) | `drivers::misc::MiscDevice` |
| `misc_register(misc)` | install miscdevice into core, allocate minor, create `/dev/<name>` | `MiscCore::register_device` |
| `misc_deregister(misc)` | unregister, free minor, destroy device | `MiscCore::deregister_device` |
| `misc_fops.open(inode, file)` | core open dispatcher: look up minor, install `misc->fops` into `file->f_op` | `MiscCore::dispatch_open` |
| `misc_seq_ops` (`/proc/misc`) | enumerate registered minors | `MiscCore::proc_misc` |
| `MISC_MAJOR (10)` | fixed major number for all misc devices | `MiscCore::MAJOR` |
| `MISC_DYNAMIC_MINOR (255)` sentinel + `ida_alloc_range(misc_minors_ida, 256, MINORMASK, ...)` | dynamic minor pool above 255 | `MiscCore::dynamic_minor_pool` |
| `MODULE_ALIAS_MISCDEV(minor)` | per-minor module alias for autoload | `Macro::module_alias_miscdev` |
| `module_misc_device(misc)` / `builtin_misc_device(misc)` | helper init macros | `Macro::module_misc_device` |
| `misc_minors_ida` | bitmap allocator for dynamic minors | `MiscCore::minors_ida` |
| Fixed minor table in `<linux/miscdevice.h>` (e.g. `KVM_MINOR=232`, `FUSE_MINOR=229`, `HPET_MINOR=228`, `RFKILL_MINOR=242`, `LOOP_CTRL_MINOR=237`, …) | curated reserved minors for legacy/well-known nodes | `Uapi::ReservedMinors` |

## Compatibility contract

REQ-1: All miscdevices share `MISC_MAJOR (10)`; the misc core registers one chardev region of `MINORMASK + 1` minors on its `misc_init()`.

REQ-2: Driver-supplied minor numbers in range `[0, 254]` are treated as fixed (must match the curated table); a collision returns `-EBUSY` from `misc_register`.

REQ-3: Driver-supplied minor equal to `MISC_DYNAMIC_MINOR (255)` requests an allocation from the dynamic pool `[256, MINORMASK]`; the assigned minor is written back to `misc->minor`.

REQ-4: Driver-supplied minor greater than `MISC_DYNAMIC_MINOR` is allowed for callers that pre-allocate from the dynamic IDA — the core validates and registers as-is.

REQ-5: On `misc_register` the core sets up `misc->this_device = device_create(misc_class, parent, MKDEV(MISC_MAJOR, minor), misc, "%s", nodename ?: name)`; udev hot-plug fires.

REQ-6: `misc->fops` is installed into `file->f_op` only on first `open()`; until then the chardev defaults to the core's `misc_fops`. After install, all subsequent `read/write/ioctl/mmap/poll/release` go straight to the per-instance fops.

REQ-7: `misc->groups` (optional) attaches sysfs attribute groups under `/sys/class/misc/<name>/`.

REQ-8: `misc->mode` (optional, default 0600) sets the udev default for `/dev/<name>`.

REQ-9: `misc->nodename` (optional) overrides the device-node basename; supports subdirectory layouts like `infiniband/uverbs0` via udev rules.

REQ-10: `misc_deregister` waits for in-flight chardev opens (via cdev refcount), frees the dynamic minor, and destroys the sysfs node; a driver must not free `misc` until `misc_deregister` returns.

REQ-11: Per-driver fops typically expose only the subset of file-operations the device supports; missing ops return `-ENOTTY`/`-EINVAL` via VFS defaults.

## Acceptance Criteria

- [ ] AC-1: `cat /proc/misc` enumerates every loaded misc driver with its allocated minor.
- [ ] AC-2: Loading a driver that calls `misc_register` with `MISC_DYNAMIC_MINOR` results in `ls /dev/<name>` appearing immediately (udev-mediated), with mode taken from `misc->mode` or 0600 default.
- [ ] AC-3: Two drivers requesting the same fixed minor: second `misc_register` returns `-EBUSY`; only the first appears in `/proc/misc`.
- [ ] AC-4: Unloading a misc driver with an open file descriptor: `misc_deregister` blocks (or refuses, depending on driver) until the fd is closed; `/dev/<name>` is removed after the last close.
- [ ] AC-5: `MODULE_ALIAS_MISCDEV(KVM_MINOR)` triggers autoload of `kvm.ko` when `/dev/kvm` is opened with `kvm.ko` absent (`request_module("char-major-10-232")`).
- [ ] AC-6: A misc driver that does not provide `read` returns `-EINVAL` on read syscall via VFS default `no_llseek`/`bad_*` fallbacks.
- [ ] AC-7: `kmemleak scan` after `modprobe foo_misc; rmmod foo_misc` reports zero leaks.
- [ ] AC-8: Concurrent `misc_register`/`misc_deregister` from parallel insmods serialize on `misc_mtx`; no minor double-allocation.

## Architecture

`MiscCore` in `drivers::misc_core::MiscCore` (single subsystem object, one instance per running kernel):

```
struct MiscCore {
  major: u32,                       // == MISC_MAJOR == 10
  class: KBox<DeviceClass>,         // /sys/class/misc/
  registered: Mutex<List<Arc<MiscDevice>>>,  // all registered misc devices
  minors_ida: Ida,                  // dynamic pool [256, MINORMASK]
  proc_node: Option<ProcDirEntry>,  // /proc/misc
}

struct MiscDevice {
  minor: u32,                       // fixed (< 255) or dynamic (>= 256)
  name: KStr,
  fops: &'static FileOperations,
  parent: Option<Arc<Device>>,
  this_device: Arc<Device>,         // populated on misc_register
  groups: &'static [&'static AttributeGroup],
  nodename: Option<KStr>,
  mode: Umode,
  list_link: ListLink,              // anchored in MiscCore::registered
}
```

Core init `MiscCore::init`:
1. `class_create("misc")` → `misc_class`.
2. `__register_chrdev(MISC_MAJOR, 0, MINORMASK + 1, "misc", &misc_fops)` — claim the entire major-10 minor space.
3. `ida_init(&misc_minors_ida)`.
4. `proc_create_seq("misc", 0, NULL, &misc_seq_ops)` — `/proc/misc` lister.

Per-device register `MiscCore::register_device(misc)`:
1. Validate `misc->minor`:
   - If `> MISC_DYNAMIC_MINOR`: caller pre-allocated, accept as-is.
   - If `== MISC_DYNAMIC_MINOR`: `ida_alloc_range(misc_minors_ida, MISC_DYNAMIC_MINOR + 1, MINORMASK, GFP_KERNEL)`.
   - Else fixed: validate not already in `registered` list.
2. Mark `misc->minor` to the assigned value.
3. `device_create_with_groups(misc_class, parent, dev_t, misc, groups, nodename ?: name)`.
4. Append to `registered` under `misc_mtx`.

Per-device deregister `MiscCore::deregister_device(misc)`:
1. Remove from `registered` under `misc_mtx`.
2. `device_destroy(misc_class, MKDEV(MISC_MAJOR, misc->minor))`.
3. If dynamic minor (`> MISC_DYNAMIC_MINOR`): `ida_free(misc_minors_ida, misc->minor)`, then reset `misc->minor = MISC_DYNAMIC_MINOR` so a subsequent `misc_register` of the same struct re-allocates.

Open dispatcher `MiscCore::dispatch_open(inode, file)`:
1. `minor = iminor(inode)`.
2. Walk `registered` under `misc_mtx` (or RCU) to find the `miscdevice` with that minor.
3. If found: capture `new_fops = misc->fops`, install `file->f_op = new_fops`, then call `new_fops->open(inode, file)` if non-NULL.
4. If not found AND minor `< MISC_DYNAMIC_MINOR`: `request_module("char-major-%d-%d", MISC_MAJOR, minor)` and retry once. Otherwise `-ENODEV`.

`/proc/misc` lister: `seq_printf("%3i %s\n", misc->minor, misc->name)` for each registered device, sorted by minor.

Per-driver pattern (typical):
```c
static const struct file_operations foo_fops = {
  .owner = THIS_MODULE,
  .open  = foo_open,
  .release = foo_release,
  .unlocked_ioctl = foo_ioctl,
};
static struct miscdevice foo_misc = {
  .minor = MISC_DYNAMIC_MINOR,
  .name  = "foo",
  .fops  = &foo_fops,
  .mode  = 0600,
};
module_misc_device(foo_misc);
```

## Hardening

- `misc_register` is called with the misc-core mutex held; no two parallel registrations can race on the same minor.
- Dynamic-minor allocation is bounded by `MINORMASK - MISC_DYNAMIC_MINOR`; once exhausted, `ida_alloc_range` returns `-ENOSPC` rather than wrapping.
- `dispatch_open` looks up minor under the misc-core lock and captures `misc->fops` into a local before installing into `file->f_op` — no torn read across deregistration races.
- `misc_deregister` removes the device from the lookup list *before* it can be re-opened, so a slow `open()` already in progress completes with the captured fops (no UAF), but a new `open()` after deregister sees `-ENODEV`.
- `device_create` is invoked with the misc-supplied `parent` — udev rules can filter on `SUBSYSTEM=="misc"` plus parent device for per-instance permissions.
- `MODULE_ALIAS_MISCDEV(minor)` is only useful for fixed minors; dynamic-minor devices cannot autoload via open(minor) — autoload must be filename-based via udev.
- `/proc/misc` exposes only `(minor, name)` pairs — no kernel pointers; the proc file is world-readable by default.
- The misc-class `/sys/class/misc/` directory inherits sysfs-default permissions; per-instance `mode` controls only the `/dev/<name>` node, not the sysfs class entry.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelist slab caches for `miscdevice` and per-driver private state (each driver responsible for its own slab whitelist); `/proc/misc` output cross only `seq_file` helpers.
- **PAX_KERNEXEC** — `misc_fops`, `misc_seq_ops`, `misc_class`, and the misc-core init path live in W^X regions; per-driver fops live in `__ro_after_init` once `misc_register` returns.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `misc_open`, `misc_register`, and per-driver `open`/`ioctl` entries to break leaks of `miscdevice` and per-instance state pointers.
- **PAX_REFCOUNT** — saturating `refcount_t` on the `device` underlying `misc->this_device`; overflow trap defeats register/deregister UAF under driver-fault injection.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `miscdevice` (driver-owned, but documented as required) and per-driver private state; dynamic minor IDs zeroed on `ida_free` so a stale `misc->minor` cannot leak into a sibling driver.
- **PAX_UDEREF** — SMAP/PAN enforced on every per-driver `ioctl`/`read`/`write` user-pointer dereference; the misc core never touches user buffers itself.
- **PAX_RAP / kCFI** — `misc_fops`, per-driver `file_operations`, and per-driver `attribute_group` callbacks marked `__ro_after_init` with kCFI-typed indirect dispatch; the open-time `file->f_op = misc->fops` install is itself a kCFI-typed assignment.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `miscdevice` pointers, `misc_class` base, and per-driver private structures behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict `misc_register`-fail, dynamic-minor-exhausted, and per-driver init banners to CAP_SYSLOG so attackers cannot probe which misc drivers are loaded via dmesg.
- **misc_register minor allocator** — `ida_alloc_range` strictly bounded to `[MISC_DYNAMIC_MINOR + 1, MINORMASK]`; on exhaustion, `-ENOSPC` and a CAP_SYSLOG-gated banner. Per-fixed-minor collisions return `-EBUSY` without revealing the conflicting driver name to non-root.
- **/dev/<name> permission gates** — `miscdevice->mode` defaults enforced to 0600 root:root unless the driver explicitly opts into a more permissive mode; udev rules MUST be reviewed for misc-class devices since `SUBSYSTEM=="misc"` rules can over-broad-grant.
- **file_operations kCFI** — both `misc_fops.open` (the core dispatcher) and the per-driver `fops` installed at `file->f_op` time are kCFI-typed; an attacker who corrupts `miscdevice->fops` is caught at the indirect call rather than after pivot.
- **MISC_DYNAMIC_MINOR bounded** — dynamic minor allocation never returns a value below `MISC_DYNAMIC_MINOR + 1`; the IDA is bounded above by `MINORMASK` so an exhausted pool is observable, not silently wrapped.
- **Open-time module autoload** — `request_module("char-major-10-%d", minor)` only fires for `minor < MISC_DYNAMIC_MINOR` (curated fixed minors); dynamic minors never trigger autoload, so an attacker cannot probe arbitrary `char-major-10-N` module names via /dev opens.

Rationale: `drivers/misc/` is the single largest concentration of "small driver, big audit surface" in the kernel — dozens of unrelated subsystems share `/dev/<name>` semantics, default permission policy, and the same `misc_register` allocator. A torn `file->f_op` swap, a stale dynamic minor, or a permissive `mode` default would silently downgrade every misc driver simultaneously. RAP/kCFI on `misc_fops` and per-driver fops at install time, refcount-overflow trapping on `this_device`, default-0600 mode policy, CAP_SYSLOG-gated registration banners, and a strictly-bounded dynamic minor IDA turn the misc framework from "a /dev/foo node showed up, hope it's safe" into a uniformly-gated, leak-clean character-device base layer that the entire `drivers/misc/` tree (and dozens of other subsystems) builds on.

## Open Questions

- The misc-class default `mode` semantics rely on per-driver opt-in; consider centralizing a CAP_SYS_ADMIN-required policy for "newly registered misc with mode > 0600" so future drivers cannot silently broaden access. Out of scope for this Tier-3.
- `/proc/misc` exposes the loaded-driver list to any user; consider GRKERNSEC_PROC-style gating of `/proc/misc` to root-only to deny driver-fingerprinting from sandboxes. Out of scope here.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `minor_unique` | UNIQUENESS | concurrent `misc_register` calls never assign overlapping minors under `misc_mtx`. |
| `fops_install_atomic` | ATOMICITY | `file->f_op = misc->fops` install observed atomically by all CPUs that hold the file's reference. |
| `ida_bounds` | OOB | dynamic minor strictly in `[MISC_DYNAMIC_MINOR + 1, MINORMASK]`. |
| `deregister_no_uaf` | UAF | `misc_deregister` removes from list before freeing; in-flight `dispatch_open` either completes against the captured fops or sees `-ENODEV`. |

### Layer 2: TLA+

`models/misc/register_open.tla`: prove that for any interleaving of `misc_register(A)`, `misc_deregister(A)`, `misc_register(B)` (where A and B request the same minor), and concurrent `open(/dev/<minor>)`, the open either reaches A's fops, reaches B's fops, or returns `-ENODEV` — never a torn/mixed fops install.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `MiscCore::register_device` post: `misc->this_device != NULL && misc->minor != 0 && misc ∈ MiscCore.registered` | `MiscCore::register_device` |
| `MiscCore::deregister_device` post: `misc ∉ MiscCore.registered && /dev/<name> removed` | `MiscCore::deregister_device` |
| `MiscCore::dispatch_open` post: `file->f_op == misc->fops` for the resolved minor, or `open` returned `-ENODEV` | `MiscCore::dispatch_open` |

### Layer 4: Verus/Creusot functional

Userspace `open("/dev/foo", O_RDWR)` → VFS chardev path → `misc_fops.open` → minor lookup → `file->f_op = foo_misc.fops` → `foo_fops.open(file)` → `foo_misc->this_device` reference held until close. Encoded as a Verus invariant proving the file's fops are exactly the registered misc instance's fops for the entire lifetime of the open file.
