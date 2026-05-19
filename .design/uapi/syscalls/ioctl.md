# Tier-5 syscall: ioctl(2) — syscall 16

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/ioctl.c (SYSCALL_DEFINE3(ioctl), do_vfs_ioctl, vfs_ioctl, file_ioctl)
  - include/linux/fs.h (struct file_operations { .unlocked_ioctl, .compat_ioctl })
  - include/uapi/linux/ioctl.h (_IO, _IOR, _IOW, _IOWR, _IOC_TYPE, _IOC_SIZE)
  - include/uapi/asm-generic/ioctls.h (TCGETS, TIOCGWINSZ, FIONREAD, ...)
  - security/security.c (security_file_ioctl LSM hook)
  - arch/x86/entry/syscalls/syscall_64.tbl (16  common  ioctl)
-->

## Summary

`ioctl(2)` is the **everything-else** multiplexed syscall. It accepts an open file descriptor and a driver-defined `request` number, plus an arbitrary third argument (treated by the kernel as `unsigned long`, usually a user pointer), and dispatches into the underlying file's `file_operations.unlocked_ioctl` (or `.compat_ioctl` for 32-on-64 callers). Every character driver, every block driver, every socket family, every pseudo-filesystem (`/proc`, `/sys`, `/dev/null`, `/dev/loop*`, `/dev/dm-*`, `/dev/kvm`, `/dev/fuse`, `/dev/dri/*`, `/dev/snd/*`, `/dev/tty*`, ...) carries its own ioctl table. The request space is partitioned by the `_IOC_TYPE()` "magic" byte (per-subsystem) and the `_IOC_NR()` sequence number; the upper bits encode direction (`_IOC_NONE`/`_IOC_READ`/`_IOC_WRITE`/`_IOC_READ|_IOC_WRITE`) and payload size.

ioctl is, by surface area, the **largest** stable userspace interface in the kernel: thousands of distinct request numbers, each with its own per-driver semantics, capability gating, copy_from/to_user contract, and verification needs. It is also the most-exploited interface historically: ioctl-driven CVEs dominate Linux-kernel security advisories. Critical for: terminal control (TCGETS / TIOCSWINSZ), socket control (SIOCGIFCONF, SIOCETHTOOL), block device geometry (BLKGETSIZE64, BLKDISCARD), DRM/KMS rendering (DRM_IOCTL_MODE_*), KVM guest control (KVM_RUN, KVM_SET_REGS), FUSE (FUSE_DEV_IOC_CLONE), nbd, dm, loop, tun/tap, evdev, snd-pcm, watchdog, perf, BPF link triggering, container runtime hooks. ioctl is NOT deprecated — it is the canonical entry point for driver-specific operations, alongside the much narrower `unlocked_ioctl_compat` selectors.

## Signature

```c
int ioctl(int fd, unsigned long request, ...);   /* glibc */
```

```c
/* Kernel-side: */
SYSCALL_DEFINE3(ioctl, unsigned int, fd, unsigned int, cmd, unsigned long, arg)
```

```c
/* Request encoding (include/uapi/linux/ioctl.h) */
#define _IOC_NRBITS    8
#define _IOC_TYPEBITS  8
#define _IOC_SIZEBITS  14
#define _IOC_DIRBITS   2

#define _IOC_NONE   0U
#define _IOC_WRITE  1U
#define _IOC_READ   2U

#define _IOC(dir,type,nr,size) (((dir)<<_IOC_DIRSHIFT)|((type)<<_IOC_TYPESHIFT)|((nr)<<_IOC_NRSHIFT)|((size)<<_IOC_SIZESHIFT))
#define _IO(type,nr)            _IOC(_IOC_NONE,(type),(nr),0)
#define _IOR(type,nr,size)      _IOC(_IOC_READ,(type),(nr),sizeof(size))
#define _IOW(type,nr,size)      _IOC(_IOC_WRITE,(type),(nr),sizeof(size))
#define _IOWR(type,nr,size)     _IOC(_IOC_READ|_IOC_WRITE,(type),(nr),sizeof(size))
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor; refers to a character device, socket, block device, pipe, eventfd, or any inode whose `f_op->unlocked_ioctl` is non-NULL. |
| `request` | `unsigned long` | in | Driver-defined request code. Upper bits encode direction + size; lower 16 bits identify subsystem (`_IOC_TYPE`, 8 bits) and operation number (`_IOC_NR`, 8 bits). |
| `arg` | `unsigned long` | varies | Third arg, often a user pointer cast to `unsigned long`. Sometimes a value (integer command). Driver-defined. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Driver-defined success. Many ioctls return 0; some return counts, descriptors, or data sizes. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` is not an open file descriptor. |
| `EFAULT` | `arg` is a user pointer and references inaccessible memory. |
| `EINVAL` | Request not valid for the device or arguments out of range. |
| `ENOTTY` | The fd's file object does NOT implement the requested ioctl (the "not a typewriter" historical name). Most-common ioctl error. |
| `ENOIOCTLCMD` | Per-driver internal not-supported; rewritten to `ENOTTY` at the syscall boundary. |
| `EPERM` | Required capability (CAP_NET_ADMIN, CAP_SYS_RAWIO, CAP_SYS_ADMIN, ...) missing. |
| `EACCES` | LSM (SELinux/AppArmor) denied the request via `security_file_ioctl`. |
| `EBUSY` | Device busy or operation conflicts with concurrent state. |
| `ENODEV` | Device removed or not present. |
| `EOPNOTSUPP` | Operation not supported by this kernel build. |
| `E2BIG` | Argument structure size exceeds driver's maximum. |
| `ENOTSUP` | Per-driver alias for EOPNOTSUPP. |

## ABI surface

```text
__NR_ioctl (x86_64)    = 16
__NR_ioctl (i386)      = 54
__NR_ioctl (arm64)     = 29
__NR_ioctl (riscv)     = 29
__NR_ioctl (loongarch) = 29

/* request bit layout (asm-generic) */
[31:30] direction  (NONE=0, WRITE=1, READ=2, READ|WRITE=3)
[29:16] size       (sizeof of argument struct)
[15: 8] type       (subsystem magic; see Documentation/userspace-api/ioctl/ioctl-number.rst)
[ 7: 0] nr         (per-subsystem operation number)
```

## Compatibility contract

REQ-1: Syscall number is **16** on x86_64; **29** on arm64/riscv/loongarch (generic). ABI-stable since Linux 1.0.

REQ-2: ioctl dispatches into `file->f_op->unlocked_ioctl(file, cmd, arg)`. If `unlocked_ioctl` is NULL, returns `-ENOTTY`.

REQ-3: For 32-bit-on-64-bit callers, dispatch goes through `compat_ioctl` (if non-NULL) which translates 32-bit pointers / sizes to 64-bit. Drivers MUST provide `compat_ioctl` for any 64-bit-only `unlocked_ioctl` that takes user-pointer-bearing structures.

REQ-4: A small set of generic ioctls are handled in `fs/ioctl.c` before dispatch to the driver: FIBMAP, FICLONE, FIDEDUPERANGE, FIFREEZE, FIGETBSZ, FIONBIO, FIONCLEX, FIOCLEX, FIONREAD, FS_IOC_GETFLAGS / FS_IOC_SETFLAGS, FS_IOC_FSGETXATTR / FS_IOC_FSSETXATTR, FS_IOC_GETVERSION / FS_IOC_SETVERSION, FS_IOC_GETFSLABEL / FS_IOC_SETFSLABEL.

REQ-5: LSM hook `security_file_ioctl(file, cmd, arg)` is called BEFORE the driver's `unlocked_ioctl`. SELinux maps cmd to a permission via `selinux_file_ioctl`; AppArmor checks via profile policy. Hook denial yields `-EACCES`.

REQ-6: Capability gating is per-driver:
- Networking ioctls (SIOCSIFADDR, SIOCSIFFLAGS, SIOCETHTOOL) require `CAP_NET_ADMIN`.
- Block-device raw access (BLKDISCARD, BLKZEROOUT) requires `CAP_SYS_ADMIN`.
- /dev/mem-style access (KDSKBSENT, VT_*) varies per-tty.
- KVM_* requires the kvm-fd ownership; KVM_CREATE_VM requires `CAP_SYS_ADMIN` in init_user_ns (or per-userns delegation).
- DRM master ioctls require DRM master state.

REQ-7: Magic-number validation (drivers SHOULD):
- Verify `_IOC_TYPE(cmd) == DRIVER_MAGIC` at top of handler.
- Verify `_IOC_DIR(cmd)` matches the user-access direction.
- Verify `_IOC_SIZE(cmd) == sizeof(expected_struct)` for structured ioctls.
- `access_ok(uptr, _IOC_SIZE(cmd))` if direction includes READ or WRITE.

REQ-8: For `_IOC_DIR(cmd) & _IOC_WRITE`: driver copies FROM userspace via `copy_from_user`. For `_IOC_DIR(cmd) & _IOC_READ`: driver copies TO userspace via `copy_to_user`. Mismatched directions are an ABI violation.

REQ-9: ioctl is async-signal-unsafe in general. Blocking ioctls (TIOCMIWAIT, RNDADDENTROPY, KVM_RUN) MAY be interrupted; restart-with-`-ERESTARTSYS` policy is driver-specific.

REQ-10: `unlocked_ioctl` is invoked WITHOUT the BKL; the driver is responsible for its own locking. The "_unlocked" prefix is historical and refers to the BKL removal in 2010.

REQ-11: Per-`seccomp` filters: requests can be filtered by `cmd` value via `BPF_LD | BPF_W | BPF_ABS, offsetof(seccomp_data, args[1])`. Modern seccomp filters frequently denylist dangerous ioctls (e.g. mount-related).

REQ-12: Per-`audit`: ioctls under audit-watch (audit_filter_ioctl_*) emit AUDIT_PATH + AUDIT_SYSCALL records with cmd + arg.

REQ-13: Tracing: `ftrace -e syscalls:sys_enter_ioctl` exposes cmd + arg. `bpftrace -e tracepoint:syscalls:sys_enter_ioctl` is the canonical observability path.

REQ-14: Per-Rookery: a centralized `IoctlRouter` maps `(file_type, _IOC_TYPE, _IOC_NR)` to typed handler closures with size + direction verification at the dispatch site; driver code receives a typed `IoctlArg` enum, not a raw `unsigned long`.

REQ-15: ioctl numbers are documented in `Documentation/userspace-api/ioctl/ioctl-number.rst`; new drivers MUST claim a magic byte to avoid collision.

## Acceptance Criteria

- [ ] AC-1: `ioctl(fd, FIONREAD, &bytes)` on a socket returns 0 and writes the bytes-available count.
- [ ] AC-2: `ioctl(badfd, ANY, 0)` returns `-1, EBADF`.
- [ ] AC-3: `ioctl(devnull_fd, TIOCGWINSZ, &ws)` returns `-1, ENOTTY` (no tty).
- [ ] AC-4: `ioctl(socket_fd, SIOCSIFFLAGS, &ifr)` without `CAP_NET_ADMIN` returns `-1, EPERM`.
- [ ] AC-5: `ioctl(blkdev_fd, BLKGETSIZE64, &sz)` returns 0 and the device size in bytes.
- [ ] AC-6: SELinux deny via `security_file_ioctl` returns `-1, EACCES`.
- [ ] AC-7: 32-bit caller's ioctl with pointer args routes through `compat_ioctl`.
- [ ] AC-8: ioctl with `_IOC_SIZE(cmd)` mismatching driver expectation: driver returns `-EINVAL`.
- [ ] AC-9: ioctl with `_IOC_TYPE(cmd)` not matching driver magic: driver returns `-ENOTTY`.
- [ ] AC-10: ioctl with bad user pointer: returns `-1, EFAULT`.
- [ ] AC-11: ioctl on a /dev/kvm fd without `CAP_SYS_ADMIN` (init_userns) for KVM_CREATE_VM: returns `-1, EPERM`.
- [ ] AC-12: ioctl is restartable when interrupted with a signal and `SA_RESTART` is set.

## Architecture

```rust
#[syscall(nr = 16, abi = "sysv")]
pub fn sys_ioctl(fd: i32, cmd: u32, arg: usize) -> isize {
    Ioctl::do_ioctl(fd, cmd, arg)
}
```

`Ioctl::do_ioctl(fd, cmd, arg) -> isize`:
1. let file = FdTable::get(current(), fd).ok_or(-EBADF)?;
2. /* Generic FS-level ioctls intercepted first */
3. if let Some(ret) = Ioctl::handle_generic(&file, cmd, arg) {
4.   return ret;
5. }
6. /* LSM hook BEFORE driver dispatch */
7. security_file_ioctl(&file, cmd, arg).map_err(|_| -EACCES)?;
8. /* Per-fop dispatch */
9. let fop = file.f_op();
10. let handler = if current().is_compat_task() {
11.   fop.compat_ioctl.ok_or(-ENOTTY)?
12. } else {
13.   fop.unlocked_ioctl.ok_or(-ENOTTY)?
14. };
15. /* Driver handler runs without BKL; its own locking applies */
16. let ret = handler(&file, cmd, arg);
17. /* Normalize ENOIOCTLCMD → ENOTTY */
18. if ret == -ENOIOCTLCMD { -ENOTTY } else { ret }

`Ioctl::handle_generic(file, cmd, arg) -> Option<isize>`:
1. match cmd {
2.   FIONBIO     => Some(Ioctl::set_nonblocking(file, arg as *const i32)),
3.   FIOCLEX     => Some(Ioctl::set_cloexec(file, true)),
4.   FIONCLEX    => Some(Ioctl::set_cloexec(file, false)),
5.   FIONREAD    => Some(Ioctl::get_readable_bytes(file, arg as *mut i32)),
6.   FIGETBSZ    => Some(Ioctl::get_block_size(file, arg as *mut i32)),
7.   FS_IOC_GETFLAGS => Some(Ioctl::get_inode_flags(file, arg as *mut u32)),
8.   FS_IOC_SETFLAGS => Some(Ioctl::set_inode_flags(file, arg as *const u32)),
9.   FIBMAP / FIFREEZE / FITHAW / FICLONE / FIDEDUPERANGE => Some(Ioctl::fs_op(file, cmd, arg)),
10.  _ => None,
11. }

`Ioctl::validate_magic(cmd, expected_type, expected_size) -> Result<()>`:
1. if _IOC_TYPE(cmd) != expected_type { return Err(-ENOTTY); }
2. if _IOC_SIZE(cmd) != expected_size as u32 { return Err(-EINVAL); }
3. if _IOC_DIR(cmd) & _IOC_READ && !access_ok::<u8>(arg, _IOC_SIZE(cmd)) { return Err(-EFAULT); }
4. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_validation_before_dispatch` | INVARIANT | per-ioctl: file lookup precedes any handler call. |
| `lsm_hook_before_driver` | INVARIANT | per-ioctl: security_file_ioctl runs before unlocked_ioctl. |
| `compat_path_for_compat_task` | INVARIANT | per-ioctl: compat task → compat_ioctl handler. |
| `enoioctlcmd_rewrite` | INVARIANT | per-driver-return: ENOIOCTLCMD ⟹ ENOTTY at syscall boundary. |
| `enotty_when_no_handler` | INVARIANT | per-ioctl: unlocked_ioctl == NULL ⟹ ENOTTY. |
| `arg_pointer_access_ok` | INVARIANT | per-driver: arg as user pointer checked via access_ok before copy. |
| `magic_validation` | INVARIANT | per-driver: _IOC_TYPE match before structured copy. |

### Layer 2: TLA+

`fs/ioctl.tla`:
- States: per-fd-lookup, per-LSM-decision, per-handler-dispatch, per-return-normalization.
- Properties:
  - `safety_lsm_pre_handler` — every successful handler call has prior LSM accept.
  - `safety_compat_routing` — compat tasks never reach native handler.
  - `safety_enotty_no_handler` — null handler ⟹ ENOTTY.
  - `safety_enoioctlcmd_hidden` — userspace never sees ENOIOCTLCMD.
  - `liveness_dispatch_terminates` — every ioctl returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_ioctl` post: returns -EBADF for invalid fd | `Ioctl::do_ioctl` |
| `do_ioctl` post: never invokes driver after LSM denial | `Ioctl::do_ioctl` |
| `do_ioctl` post: never returns ENOIOCTLCMD to userspace | `Ioctl::do_ioctl` |
| `validate_magic` post: rejects wrong magic / wrong size | `Ioctl::validate_magic` |
| `handle_generic` post: handles all FS-level cmds before driver dispatch | `Ioctl::handle_generic` |

### Layer 4: Verus / Creusot functional

Per-`ioctl(2)` man-page equivalence. LTP `ioctl0*` tests pass. Driver-specific selftests (KVM, DRM, net, block) cover per-magic semantics.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`ioctl(2)` reinforcement:

- **Per-LSM-hook mandatory pre-dispatch** — defense against per-driver-permission-skip.
- **Per-magic-byte validation by driver** — defense against per-cross-driver request smuggling.
- **Per-size validation** — defense against per-short-copy / per-overlong-copy bugs.
- **Per-direction validation** — defense against per-WRITE-as-READ access pattern.
- **Per-compat-ioctl path** — defense against per-32-on-64 pointer-truncation bugs.
- **Per-cap-gate at handler entry** — defense against per-priv-escalation via unguarded driver.
- **Per-typed-IoctlArg in Rookery** — defense against per-raw-unsigned-long type confusion.
- **Per-access_ok before copy_from/to_user** — defense against per-kernel-deref via crafted pointer.
- **Per-ENOIOCTLCMD normalization** — defense against per-information-leak via internal errno.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF + SMAP forced on all copy_from/to_user** — defense against per-arg-pointer kernel-deref bug; any direct kernel access to userspace addresses outside `copy_*_user` faults.
- **GRKERNSEC_IOCTL_RESTRICT** — global ioctl gate: in hardened mode, ioctls with non-standard magic bytes (outside an allowlist) require `CAP_SYS_ADMIN`. The allowlist covers TTY (`'t'`, `'T'`), socket (`'s'`), block (`0x12`), evdev (`'E'`), and a small set of others.
- **Per-driver CAP_* gating audited** — every driver `unlocked_ioctl` MUST contain at least one `capable(...)` call OR explicitly mark itself "unprivileged-safe"; grsec scans driver init for the annotation.
- **Magic-number validation enforced** — every driver MUST validate `_IOC_TYPE(cmd) == DRIVER_MAGIC` at handler entry. Grsec config `GRKERNSEC_IOCTL_MAGIC_STRICT` rejects ioctls where the driver did not validate magic.
- **security_file_ioctl LSM hook is mandatory** — cannot be skipped; SELinux + AppArmor + Yama profiles enforce per-fd ioctl allow-lists for confined processes.
- **PAX_REFCOUNT on file object refcount during ioctl** — defense against per-driver UAF via early file close.
- **GRKERNSEC_CHROOT_FSIOCTL** — inside a grsec-hardened chroot, FIFREEZE, FITHAW, FS_IOC_FSGETXATTR-with-immutable-bit are blocked.
- **GRKERNSEC_HIDESYM on driver ioctl handlers** — driver function pointers not exposed in `/proc/kallsyms`.
- **PaX KERNEXEC on f_op tables** — `struct file_operations` instances are read-only after init; rootkit hooking of `unlocked_ioctl` is blocked.
- **PAX_USERCOPY_HARDEN on ioctl argument copy** — bounded copy uses whitelisted slabs.
- **GRKERNSEC_BPF_HARDEN** — BPF-link ioctls (BPF_LINK_*) require `CAP_BPF`; unprivileged BPF disabled.
- **Per-tty hardening** — TIOCSTI is gated by `kernel.protected_ttys=1` in grsec defaults; CVE-2017-2636-class vulnerabilities are fenced.
- **Per-KVM hardening** — KVM_RUN under grsec is restricted to init_userns + `CAP_SYS_ADMIN`; KVM_HC_CLOCK_PAIRING-style attack surface gated.
- **Per-DRM hardening** — DRM master ioctls require active foreground session; backgrounded sessions cannot re-acquire master.
- **Kexec preservation** — ioctl LSM hooks rebuilt identically on post-kexec kernel.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-driver ioctl tables (covered in Tier-3 / Tier-4 docs per subsystem).
- KVM ioctl semantics (Tier-3 `virt/kvm/api.md`).
- DRM ioctl semantics (Tier-3 `drivers/gpu/drm/ioctl.md`).
- TTY ioctl semantics (Tier-3 `drivers/tty/n_tty.md`).
- Block / scsi ioctl semantics (Tier-3 `block/ioctl.md`).
- Socket ioctl semantics (Tier-3 `net/socket-ioctl.md`).
- LSM `security_file_ioctl` hook internals (Tier-3 `security/lsm-hooks.md`).
- Implementation code.
