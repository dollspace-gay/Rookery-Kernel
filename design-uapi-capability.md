---
title: "Tier-5 UAPI: include/uapi/linux/capability.h — POSIX capabilities ABI"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

Linux capabilities split the traditional all-or-nothing root privilege into ~41 fine-grained units (`CAP_CHOWN` = 0 .. `CAP_CHECKPOINT_RESTORE` = 40 = `CAP_LAST_CAP`). Each task carries four (since file-caps: five) sets of capability bits: *effective* (what the kernel checks per access), *permitted* (the ceiling of what may be raised into effective), *inheritable* (what survives `execve` into the next program's permitted set), *bounding* (a per-task mask cap-raise can never exceed), and *ambient* (since Linux 4.3: inheritable bits that survive execve into permitted+effective even without a setuid binary). The userspace ABI lives in `capget(2)` and `capset(2)`, which take a `struct __user_cap_header_struct { version, pid }` plus a per-version count of `struct __user_cap_data_struct { effective, permitted, inheritable }` (one u32 triple for v1; two triples for v2 and v3 — encoding up to 64 capability bits). The on-disk xattr format `security.capability` carries `struct vfs_cap_data` (magic_etc, two permitted+inheritable u32 pairs) or `struct vfs_ns_cap_data` (same plus a `rootid` for userns-aware file-caps). Critical for: privilege separation, sandboxing (containers, sudo, setpriv, capsh), setuid-elimination programs (ping, dhclient, the entire systemd capability-juggling), file-caps (`setcap cap_net_raw+ep /usr/bin/ping`), and ambient-caps for unprivileged service binaries.

This Tier-5 covers the full UAPI surface of `include/uapi/linux/capability.h` (~435 lines).

### Acceptance Criteria

- [ ] AC-1: `capget(&hdr, NULL)` with `hdr.version = 0` ⇒ -EINVAL; on return `hdr.version = _LINUX_CAPABILITY_VERSION_3`.
- [ ] AC-2: `capget(&hdr, data)` with `hdr.version = _LINUX_CAPABILITY_VERSION_1, hdr.pid = 0` ⇒ writes 1 triple covering caps 0..31.
- [ ] AC-3: `capget(&hdr, data)` with `hdr.version = _LINUX_CAPABILITY_VERSION_3, hdr.pid = 0` ⇒ writes 2 triples covering caps 0..63.
- [ ] AC-4: `capget(&hdr, data)` with `hdr.version = _LINUX_CAPABILITY_VERSION_2` accepted (back-compat) but deprecation-warning logged once.
- [ ] AC-5: `capset(&hdr, data)` with `hdr.pid != 0` and `hdr.pid != current->pid` ⇒ -EPERM.
- [ ] AC-6: `capset` raising `permitted` beyond previous permitted ⇒ -EPERM.
- [ ] AC-7: `capset` setting `effective` not ⊆ `permitted` ⇒ -EPERM.
- [ ] AC-8: `capset` setting `inheritable` containing bits not in (old.inheritable ∪ old.permitted) ⇒ -EPERM.
- [ ] AC-9: `capset` setting `inheritable` containing bits not in `bounding ∪ old.inheritable` ⇒ -EPERM.
- [ ] AC-10: Capability constants 0 through 40 have the exact numeric values per header.
- [ ] AC-11: `CAP_LAST_CAP == CAP_CHECKPOINT_RESTORE == 40`.
- [ ] AC-12: `cap_valid(0)`, `cap_valid(40)` ⇒ true; `cap_valid(-1)`, `cap_valid(41)` ⇒ false.
- [ ] AC-13: `CAP_TO_INDEX(5) == 0`, `CAP_TO_INDEX(35) == 1`, `CAP_TO_MASK(5) == 0x20`, `CAP_TO_MASK(35) == 0x8`.
- [ ] AC-14: `VFS_CAP_REVISION_1 == 0x01000000`, `_2 == 0x02000000`, `_3 == 0x03000000`.
- [ ] AC-15: `XATTR_CAPS_SZ_1 == 12`, `_2 == 20`, `_3 == 24`.
- [ ] AC-16: `XATTR_CAPS_SZ == XATTR_CAPS_SZ_3 == 24`.
- [ ] AC-17: `VFS_CAP_REVISION == VFS_CAP_REVISION_3`.
- [ ] AC-18: `VFS_CAP_FLAGS_EFFECTIVE == 0x000001`.
- [ ] AC-19: `VFS_CAP_REVISION_MASK == 0xFF000000`, `_SHIFT == 24`.
- [ ] AC-20: `setxattr("security.capability", &vfs_cap_data_rev2, 20)` ⇒ accepted; subsequent `getxattr` returns same buffer.
- [ ] AC-21: `setxattr("security.capability", &vfs_ns_cap_data_rev3, 24)` ⇒ accepted with `rootid` matching writer's userns root.
- [ ] AC-22: `execve` of binary with `cap_net_raw+ep`: new task has cap_net_raw in permitted + effective.
- [ ] AC-23: `execve` of binary with `cap_net_raw+p` (no e bit): new task has cap_net_raw in permitted only; must `capset` to enter effective.
- [ ] AC-24: `execve` after `prctl(PR_SET_NO_NEW_PRIVS, 1)`: no file-cap nor setuid acquisition; new caps ⊆ old caps.
- [ ] AC-25: `prctl(PR_CAPBSET_DROP, CAP_SYS_MODULE)` requires CAP_SETPCAP; drop is irreversible.
- [ ] AC-26: `prctl(PR_CAP_AMBIENT, PR_CAP_AMBIENT_RAISE, CAP_NET_BIND_SERVICE)` requires `CAP_NET_BIND_SERVICE ∈ permitted ∩ inheritable`.
- [ ] AC-27: Ambient cap auto-cleared when permitted or inheritable bit drops.
- [ ] AC-28: User-namespace file-caps (`vfs_ns_cap_data.rootid`): caps activate only when current task's userns maps `rootid` to 0.
- [ ] AC-29: Loading a kernel module ⇒ requires CAP_SYS_MODULE; absent ⇒ -EPERM.
- [ ] AC-30: `bind` to TCP port 80 ⇒ requires `CAP_NET_BIND_SERVICE`; absent ⇒ -EACCES.
- [ ] AC-31: `mount(2)` requires `CAP_SYS_ADMIN`; absent ⇒ -EPERM.
- [ ] AC-32: `ptrace(PTRACE_ATTACH, other_uid_pid)` requires `CAP_SYS_PTRACE`.
- [ ] AC-33: `bpf(BPF_PROG_LOAD, ...)` of `BPF_PROG_TYPE_TRACING` ⇒ requires CAP_PERFMON + CAP_BPF.
- [ ] AC-34: `bpf(BPF_PROG_LOAD, ...)` of `BPF_PROG_TYPE_SCHED_CLS` ⇒ requires CAP_NET_ADMIN + CAP_BPF.
- [ ] AC-35: `bpf_probe_write_user` helper ⇒ requires CAP_SYS_ADMIN (not satisfied by CAP_BPF alone).
- [ ] AC-36: `perf_event_open(2)` requires CAP_PERFMON (or CAP_SYS_ADMIN on pre-5.8 backports) for tracepoints/breakpoints.
- [ ] AC-37: `clone3(...)` with `set_tid` explicit-pid selection requires CAP_CHECKPOINT_RESTORE.
- [ ] AC-38: `prctl(PR_SET_SECUREBITS, SECBIT_NOROOT)` requires CAP_SETPCAP.

### Architecture

```
const VERSION_1: u32 = 0x19980330;      // _LINUX_CAPABILITY_VERSION_1
const VERSION_2: u32 = 0x20071026;      // _LINUX_CAPABILITY_VERSION_2 (deprecated)
const VERSION_3: u32 = 0x20080522;      // _LINUX_CAPABILITY_VERSION_3 (current)
const U32S_1: usize = 1;
const U32S_2: usize = 2;
const U32S_3: usize = 2;

#[repr(C)]
struct CapUserHeader { version: u32, pid: i32 }

#[repr(C)]
struct CapUserData { effective: u32, permitted: u32, inheritable: u32 }

#[repr(u32)]
enum Cap {
  Chown = 0, DacOverride = 1, DacReadSearch = 2, Fowner = 3, Fsetid = 4,
  Kill = 5, Setgid = 6, Setuid = 7,
  Setpcap = 8, LinuxImmutable = 9,
  NetBindService = 10, NetBroadcast = 11, NetAdmin = 12, NetRaw = 13,
  IpcLock = 14, IpcOwner = 15,
  SysModule = 16, SysRawio = 17, SysChroot = 18, SysPtrace = 19,
  SysPacct = 20, SysAdmin = 21, SysBoot = 22, SysNice = 23,
  SysResource = 24, SysTime = 25, SysTtyConfig = 26,
  Mknod = 27, Lease = 28,
  AuditWrite = 29, AuditControl = 30,
  Setfcap = 31, MacOverride = 32, MacAdmin = 33,
  Syslog = 34, WakeAlarm = 35, BlockSuspend = 36,
  AuditRead = 37, Perfmon = 38, Bpf = 39, CheckpointRestore = 40,
}
const LAST_CAP: u32 = Cap::CheckpointRestore as u32;          // 40
fn is_valid(x: i32) -> bool { x >= 0 && (x as u32) <= LAST_CAP }
fn to_index(x: u32) -> usize { (x >> 5) as usize }
fn to_mask(x: u32) -> u32 { 1u32 << (x & 31) }

mod xattr {
  pub const REVISION_MASK: u32     = 0xFF000000;
  pub const REVISION_SHIFT: u32    = 24;
  pub const FLAGS_MASK: u32        = !REVISION_MASK;       // 0x00FFFFFF
  pub const FLAG_EFFECTIVE: u32    = 0x000001;
  pub const REVISION_1: u32        = 0x01000000;
  pub const REVISION_2: u32        = 0x02000000;
  pub const REVISION_3: u32        = 0x03000000;
  pub const U32_1: usize           = 1;
  pub const U32_2: usize           = 2;
  pub const U32_3: usize           = 2;
  pub const CAPS_SZ_1: usize       = 4 * (1 + 2 * U32_1);  // 12
  pub const CAPS_SZ_2: usize       = 4 * (1 + 2 * U32_2);  // 20
  pub const CAPS_SZ_3: usize       = 4 * (2 + 2 * U32_3);  // 24
  pub const CAPS_SZ: usize         = CAPS_SZ_3;
  pub const CURRENT_REVISION: u32  = REVISION_3;
}

#[repr(C)]
struct VfsCapData {                     // revisions 1 and 2
  magic_etc: u32_le,                    // (rev << 0) | flags
  data: [CapEntry; 2],                  // U32_2 = 2 in current build
}
#[repr(C)] struct CapEntry { permitted: u32_le, inheritable: u32_le }

#[repr(C)]
struct VfsNsCapData {                   // revision 3
  magic_etc: u32_le,
  data: [CapEntry; 2],
  rootid: u32_le,                       // uid_t of owning userns root
}
```

`Capability::capget(hdr_user, data_user) -> isize`:
1. Copy `hdr` from user; validate `hdr.version ∈ {VERSION_1, VERSION_2, VERSION_3}`.
2. If unknown ⇒ write `hdr.version = VERSION_3` to user, return -EINVAL.
3. Resolve target task via `hdr.pid` (0 ⇒ current; else lookup).
4. Read `target.cred.cap_effective / _permitted / _inheritable` (each is u64 internally).
5. For VERSION_1: write 1 triple = (low 32 bits each).
6. For VERSION_2 / VERSION_3: write 2 triples (low 32 then high 32).
7. Return 0.

`Capability::capset(hdr_user, data_user) -> isize`:
1. Copy `hdr`; validate version (else preferred-version + -EINVAL).
2. Reject `hdr.pid != 0 && hdr.pid != current.pid` ⇒ -EPERM.
3. Read triples per version into `new_eff`, `new_perm`, `new_inh` (u64 each).
4. Validate against `current.cred`:
   - `new_perm ⊆ old_perm` else -EPERM.
   - `new_eff ⊆ new_perm` else -EPERM.
   - `new_inh ⊆ old_inh ∪ old_perm` else -EPERM.
   - `new_inh ⊆ old_bounding ∪ old_inh` else -EPERM.
5. Allocate new `cred`; apply new sets; auto-drop ambient bits no longer in `new_perm ∩ new_inh`.
6. Commit new cred to current task.
7. Return 0.

`Capability::file_caps_apply(file, task) -> NewCaps`:
1. `xattr = getxattr(file, "security.capability")`.
2. `magic_etc = le32_to_cpu(xattr[0..4])`.
3. `revision = magic_etc >> VFS_CAP_REVISION_SHIFT`.
4. Switch revision:
   - 1: parse `VfsCapData` with `U32_1 = 1`; size must equal `CAPS_SZ_1 = 12`.
   - 2: parse `VfsCapData` with `U32_2 = 2`; size must equal `CAPS_SZ_2 = 20`.
   - 3: parse `VfsNsCapData`; size must equal `CAPS_SZ_3 = 24`; validate `rootid` is mappable in current userns; if not, treat as no file caps.
5. `flag_effective = magic_etc & FLAG_EFFECTIVE`.
6. Compute new cred per execve rule (REQ-16): `new_perm = (file_perm ∩ bounding) ∪ (file_inh ∩ inh)`; `new_eff = flag_effective ? new_perm : ambient_∩_perm`; etc.

`Capability::xattr_validate(buf, size)`:
1. `size ∈ {CAPS_SZ_1, CAPS_SZ_2, CAPS_SZ_3}` else -EINVAL.
2. `magic_etc = le32_to_cpu(buf[0..4])`.
3. `revision = magic_etc >> 24`; cross-check against `size`:
   - rev1 ↔ 12.
   - rev2 ↔ 20.
   - rev3 ↔ 24.
4. `flags = magic_etc & 0x00FFFFFF`; `flags & ~VFS_CAP_FLAGS_EFFECTIVE` must be 0.
5. Accept.

### Out of Scope

- `prctl(PR_CAPBSET_*)` / `prctl(PR_CAP_AMBIENT, ...)` / `prctl(PR_SET_SECUREBITS)` / `prctl(PR_SET_NO_NEW_PRIVS)` interfaces — covered in `uapi/headers/prctl.md`.
- `securebits` bit-by-bit definitions (`SECBIT_*`) — covered in `uapi/headers/securebits.md`.
- `xattr` syscall surface (`setxattr` / `getxattr` / `listxattr` / `removexattr`) — covered in `uapi/headers/xattr.md`.
- User-namespace `uid_map` / `gid_map` semantics — covered in `uapi/headers/user_namespaces.md`.
- LSM-specific cap-override hooks (SELinux, AppArmor) — covered in `uapi/headers/lsm.md` and per-LSM Tier-3 docs.
- File-cap detection by file system (ext4 / xfs / btrfs xattr handlers) — covered in per-FS Tier-3 docs.
- `setuid(2)` / `setresuid(2)` / `setfsuid(2)` cap-transition rules — covered in `uapi/headers/sched_creds.md`.
- BPF capability gating in detail (CAP_BPF vs CAP_PERFMON helper allow-list) — covered in `uapi/headers/bpf.md`.
- Implementation code (`kernel/capability.c`, `security/commoncap.c`, `fs/xattr.c`).

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `_LINUX_CAPABILITY_VERSION_1` (`0x19980330`) | per-v1 magic | `cap::Version::V1` |
| `_LINUX_CAPABILITY_U32S_1` (=1) | per-v1 u32 count | `cap::Version::V1.u32s` |
| `_LINUX_CAPABILITY_VERSION_2` (`0x20071026`) | per-v2 magic (deprecated) | `cap::Version::V2` |
| `_LINUX_CAPABILITY_U32S_2` (=2) | per-v2 u32 count | `cap::Version::V2.u32s` |
| `_LINUX_CAPABILITY_VERSION_3` (`0x20080522`) | per-v3 magic (current) | `cap::Version::V3` |
| `_LINUX_CAPABILITY_U32S_3` (=2) | per-v3 u32 count | `cap::Version::V3.u32s` |
| `_LINUX_CAPABILITY_VERSION` / `_LINUX_CAPABILITY_U32S` | per-legacy non-`__KERNEL__` alias to V1/U32S_1 | `cap::legacy_alias` |
| `struct __user_cap_header_struct` (a.k.a. `cap_user_header_t`) | per-syscall header | `CapUserHeader` |
| `struct __user_cap_data_struct` (a.k.a. `cap_user_data_t`) | per-syscall data triple | `CapUserData` |
| `VFS_CAP_REVISION_MASK` / `_SHIFT` / `_FLAGS_MASK` | per-xattr magic decode | `cap::xattr::revision` |
| `VFS_CAP_FLAGS_EFFECTIVE` | per-xattr effective bit | `cap::xattr::FLAG_EFFECTIVE` |
| `VFS_CAP_REVISION_1/_2/_3` | per-xattr revision | `cap::xattr::Revision` |
| `VFS_CAP_U32_1/_2/_3` | per-revision u32 count | `cap::xattr::U32` |
| `XATTR_CAPS_SZ_1/_2/_3` / `XATTR_CAPS_SZ` | per-revision byte size | `cap::xattr::SZ` |
| `VFS_CAP_U32` / `VFS_CAP_REVISION` | per-current revision alias | `cap::xattr::current` |
| `struct vfs_cap_data` | per-revision-1/2 on-disk | `VfsCapData` |
| `struct vfs_ns_cap_data` | per-revision-3 (ns-aware) on-disk | `VfsNsCapData` |
| `CAP_CHOWN..CAP_CHECKPOINT_RESTORE` (0..40) | per-capability constant | `cap::Cap` |
| `CAP_LAST_CAP` (= `CAP_CHECKPOINT_RESTORE` = 40) | per-highest defined cap | `cap::LAST_CAP` |
| `cap_valid(x)` | per-validity predicate | `cap::is_valid` |
| `CAP_TO_INDEX(x)` (= `x >> 5`) | per-bit-to-u32-index | `cap::to_index` |
| `CAP_TO_MASK(x)` (= `1U << (x & 31)`) | per-bit-to-mask | `cap::to_mask` |

### abi surface (constants + structs)

### Version magics + struct counts

```
#define _LINUX_CAPABILITY_VERSION_1  0x19980330   /* original 32-bit-cap ABI */
#define _LINUX_CAPABILITY_U32S_1     1

#define _LINUX_CAPABILITY_VERSION_2  0x20071026   /* 64-bit cap ABI — deprecated, use v3 */
#define _LINUX_CAPABILITY_U32S_2     2

#define _LINUX_CAPABILITY_VERSION_3  0x20080522   /* 64-bit cap ABI w/ correct semantics */
#define _LINUX_CAPABILITY_U32S_3     2

#ifndef __KERNEL__   /* userspace-only back-compat alias */
#define _LINUX_CAPABILITY_VERSION    _LINUX_CAPABILITY_VERSION_1
#define _LINUX_CAPABILITY_U32S       _LINUX_CAPABILITY_U32S_1
#endif
```

### `struct __user_cap_header_struct` / `cap_user_header_t`

```
typedef struct __user_cap_header_struct {
  __u32 version;     /* one of _LINUX_CAPABILITY_VERSION_{1,2,3} */
  int   pid;         /* target pid (0 = current task); only current/0 allowed for cap_set/get */
} __user *cap_user_header_t;
```

### `struct __user_cap_data_struct` / `cap_user_data_t`

```
struct __user_cap_data_struct {
  __u32 effective;     /* low 32 bits of effective set */
  __u32 permitted;     /* low 32 bits of permitted set */
  __u32 inheritable;   /* low 32 bits of inheritable set */
};
typedef struct __user_cap_data_struct __user *cap_user_data_t;
```

For `_VERSION_1` the syscall reads/writes a single triple (32-bit caps). For `_VERSION_2` and `_VERSION_3` the syscall reads/writes **two** consecutive triples — `data[0]` = low 32 caps, `data[1]` = high 32 caps — encoding up to 64 capability bits.

### VFS-cap xattr (`security.capability`) decoder constants

```
#define VFS_CAP_REVISION_MASK   0xFF000000
#define VFS_CAP_REVISION_SHIFT  24
#define VFS_CAP_FLAGS_MASK      (~VFS_CAP_REVISION_MASK)   /* lower 24 bits */
#define VFS_CAP_FLAGS_EFFECTIVE 0x000001                   /* "ep" effective bit */
```

### VFS-cap xattr revisions

```
#define VFS_CAP_REVISION_1   0x01000000   /* 32-bit caps, no rootid */
#define VFS_CAP_U32_1        1
#define XATTR_CAPS_SZ_1      (sizeof(__le32) * (1 + 2*VFS_CAP_U32_1))   /* 12 bytes */

#define VFS_CAP_REVISION_2   0x02000000   /* 64-bit caps, no rootid */
#define VFS_CAP_U32_2        2
#define XATTR_CAPS_SZ_2      (sizeof(__le32) * (1 + 2*VFS_CAP_U32_2))   /* 20 bytes */

#define VFS_CAP_REVISION_3   0x03000000   /* 64-bit caps + userns rootid */
#define VFS_CAP_U32_3        2
#define XATTR_CAPS_SZ_3      (sizeof(__le32) * (2 + 2*VFS_CAP_U32_3))   /* 24 bytes */

/* current revision aliases */
#define XATTR_CAPS_SZ        XATTR_CAPS_SZ_3
#define VFS_CAP_U32          VFS_CAP_U32_3
#define VFS_CAP_REVISION     VFS_CAP_REVISION_3
```

### `struct vfs_cap_data` (xattr revisions 1 and 2)

```
struct vfs_cap_data {
  __le32 magic_etc;                /* (revision << 24) | flags */
  struct {
    __le32 permitted;              /* little-endian */
    __le32 inheritable;            /* little-endian */
  } data[VFS_CAP_U32];             /* 1 entry for rev1, 2 entries for rev2 */
};
```

### `struct vfs_ns_cap_data` (xattr revision 3 — ns-aware)

```
struct vfs_ns_cap_data {
  __le32 magic_etc;                /* (VFS_CAP_REVISION_3 << 0) | flags */
  struct {
    __le32 permitted;              /* little-endian */
    __le32 inheritable;            /* little-endian */
  } data[VFS_CAP_U32];             /* always 2 entries */
  __le32 rootid;                   /* uid_t of the user-namespace root that owns these caps */
};
```

### Capability constants (POSIX-draft / Linux-specific)

POSIX-draft capabilities (0..7):

```
CAP_CHOWN            = 0    /* override file owner/group restriction on chown */
CAP_DAC_OVERRIDE     = 1    /* override DAC read/write/execute (excl. CAP_LINUX_IMMUTABLE) */
CAP_DAC_READ_SEARCH  = 2    /* override DAC read + search on files/dirs */
CAP_FOWNER           = 3    /* override file-owner-ID match restrictions */
CAP_FSETID           = 4    /* override S_ISUID/S_ISGID clear on chmod / chown */
CAP_KILL             = 5    /* override real/effective UID match for signal sending */
CAP_SETGID           = 6    /* setgid(2) / setgroups(2) / forged-gid creds */
CAP_SETUID           = 7    /* set*uid(2) / forged-pid creds */
```

Linux-specific capabilities (8..40):

```
CAP_SETPCAP            =  8   /* w/o VFS-caps: transfer caps to any pid; w/ VFS-caps: add
                                 caps from bounding into inheritable, drop from bounding,
                                 modify securebits */
CAP_LINUX_IMMUTABLE    =  9   /* modify S_IMMUTABLE and S_APPEND attrs */
CAP_NET_BIND_SERVICE   = 10   /* bind to TCP/UDP ports < 1024 (or ATM VCIs < 32) */
CAP_NET_BROADCAST      = 11   /* broadcast / multicast */
CAP_NET_ADMIN          = 12   /* interface config, firewall, routing, raw sockets-by-uid,
                                 promisc, TOS, multicast, driver stats clear, ATM ctrl */
CAP_NET_RAW            = 13   /* AF_PACKET, raw sockets, transparent proxy bind */
CAP_IPC_LOCK           = 14   /* shmctl(SHM_LOCK), mlock, mlockall */
CAP_IPC_OWNER          = 15   /* override SysV IPC ownership checks */
CAP_SYS_MODULE         = 16   /* init_module/delete_module — kernel-modifying */
CAP_SYS_RAWIO          = 17   /* ioperm/iopl, raw USB messages via /dev/bus/usb */
CAP_SYS_CHROOT         = 18   /* chroot(2) */
CAP_SYS_PTRACE         = 19   /* ptrace() any process */
CAP_SYS_PACCT          = 20   /* process accounting config */
CAP_SYS_ADMIN          = 21   /* secure-attention key, /dev/random admin, quota,
                                 sethostname/setdomainname, mount/umount, smb connections,
                                 autofs root ioctls, nfsservctl, VM86_REQUEST_IRQ, alpha
                                 pci-config, mips/m68k arch ioctls, semaphore removal,
                                 IPC chown-equivalent, shmlock/unlock, swap on/off,
                                 forged socket creds, blockdev readahead/flush, floppy
                                 geometry, xd DMA, md admin, ide tuning, nvram, apm/serial/
                                 bttv admin, capi manufacturer cmds, pci-config debug, sbpcd
                                 DDI ioctl, serial port setup, qic-117 raw, SCSI tagged-
                                 queueing, loopback encryption key, zone reclaim policy,
                                 CAP_BPF+CAP_PERFMON back-compat, hardware-emergency-action */
CAP_SYS_BOOT           = 22   /* reboot(2), kexec_load */
CAP_SYS_NICE           = 23   /* raise priority on others, FIFO/RR realtime sched on self,
                                 set sched on others, cpu-affinity on others, realtime ioprio,
                                 ioprio on others */
CAP_SYS_RESOURCE       = 24   /* override rlimits, quota, ext2 reserved space, ext3 data-
                                 journaling mode, IPC msg-queue size, RTC > 64Hz, console
                                 max, keymap max, memory-reclaim behavior */
CAP_SYS_TIME           = 25   /* clock manipulation (settimeofday, adjtime, RTC) */
CAP_SYS_TTY_CONFIG     = 26   /* TTY device config, vhangup */
CAP_MKNOD              = 27   /* privileged aspects of mknod() (char/block dev creation) */
CAP_LEASE              = 28   /* take leases on files (F_SETLEASE) */
CAP_AUDIT_WRITE        = 29   /* write to audit log via unicast netlink */
CAP_AUDIT_CONTROL      = 30   /* configure audit via unicast netlink */
CAP_SETFCAP            = 31   /* set/remove file capabilities; map uid=0 in child userns */
CAP_MAC_OVERRIDE       = 32   /* override LSM MAC policy */
CAP_MAC_ADMIN          = 33   /* configure LSM MAC policy */
CAP_SYSLOG             = 34   /* configure printk / kmsg consume */
CAP_WAKE_ALARM         = 35   /* arm a wake-from-suspend alarm */
CAP_BLOCK_SUSPEND      = 36   /* prevent system suspend (epoll EPOLLWAKEUP) */
CAP_AUDIT_READ         = 37   /* read audit log via multicast netlink */
CAP_PERFMON            = 38   /* perf_events / i915_perf / privileged observability */
CAP_BPF                = 39   /* create BPF maps, advanced verifier features, load BTF,
                                 retrieve xlated+JITed code, bpf_spin_lock helper */
CAP_CHECKPOINT_RESTORE = 40   /* CRIU operations, clone3() pid selection, write ns_last_pid */

#define CAP_LAST_CAP   CAP_CHECKPOINT_RESTORE   /* current highest = 40 */
#define cap_valid(x)   ((x) >= 0 && (x) <= CAP_LAST_CAP)
```

### Bit-pack helpers

```
#define CAP_TO_INDEX(x)  ((x) >> 5)        /* which u32 word in the data[] array */
#define CAP_TO_MASK(x)   (1U << ((x) & 31)) /* bit mask within that u32 */
```

For 32-bit caps (rev1 / v1): all bits 0..31, single u32.
For 64-bit caps (rev2/rev3 / v2/v3): bits 0..31 in `data[0]`, bits 32..63 in `data[1]`.

### compatibility contract

REQ-1 `capget(header, dataptr)`:
- `header` is `cap_user_header_t`: caller fills `header->version` to indicate the layout they expect; `header->pid` selects target (0 = current; positive = a specific tid; only `pid == 0` or `pid == gettid()` are safe per upstream `capget` policy — others may be permitted by kernel but discouraged).
- `dataptr` is `cap_user_data_t`: kernel fills 1 entry for v1, 2 entries for v2/v3.
- Return: 0 on success.
- Special case: `header->version == 0` or unknown version ⇒ kernel writes the kernel's preferred current version into `header->version` (defaults to `_LINUX_CAPABILITY_VERSION_3` on 7.x) and returns `-EINVAL`. Caller is expected to retry with that version.

REQ-2 `capset(header, dataptr)`:
- Same arguments; `dataptr` is read by kernel.
- Constraint: `header->pid` **must** be 0 or the caller's own tid; setting another task's caps via the syscall is not permitted (use file caps + execve instead). Other values ⇒ -EPERM.
- Rules enforced by kernel on the new caps:
  - new.permitted ⊆ old.permitted (cannot raise permitted).
  - new.effective ⊆ new.permitted.
  - new.inheritable ⊆ old.inheritable ∪ old.permitted.
  - new.inheritable ⊆ old.bounding ∪ old.inheritable (cannot escape bounding set into inheritable).
  - If `securebits & SECBIT_NO_SETUID_FIXUP` not set: setuid transitions may auto-modify the sets.
- Return: 0 on success; -EPERM on rule violation; -EINVAL on malformed header.

REQ-3 `_LINUX_CAPABILITY_VERSION_1` (`0x19980330`):
- Original 32-bit-cap ABI; one `__user_cap_data_struct` triple expected.
- Reading: kernel returns only the low 32 caps; high 32 silently truncated.
- Writing: kernel sets the low 32 caps; high 32 caps left unchanged.
- Userspace-only alias `_LINUX_CAPABILITY_VERSION` resolves to this for back-compat.

REQ-4 `_LINUX_CAPABILITY_VERSION_2` (`0x20071026`):
- 64-bit cap ABI; two `__user_cap_data_struct` triples expected.
- Documented as **deprecated** in upstream comment; v2 had file-cap semantics bugs (no namespacing); kernel still accepts it for back-compat but logs a one-shot warning.
- New userspace must use v3.

REQ-5 `_LINUX_CAPABILITY_VERSION_3` (`0x20080522`):
- Current 64-bit cap ABI; two triples.
- Same wire-format as v2; the version bump signals correct handling of caps across user-namespace transitions (file-caps revision 3 with `rootid`).
- Recommended version for all new code.

REQ-6 `struct __user_cap_header_struct`:
- `version`: one of the three magics; if kernel doesn't support requested version ⇒ kernel writes preferred version into the struct and returns -EINVAL.
- `pid`: target task id; `0` = current; positive = target tid (subject to permission rules); negative ⇒ -EINVAL.

REQ-7 `struct __user_cap_data_struct`:
- `effective`, `permitted`, `inheritable` each carry 32 bits of the corresponding set.
- For v2/v3: `data[0]` carries caps 0..31, `data[1]` carries caps 32..63.
- The bounding set is **not** exposed via this syscall — it's accessed via `prctl(PR_CAPBSET_*)`.
- The ambient set is **not** exposed via this syscall — it's accessed via `prctl(PR_CAP_AMBIENT_*)`.

REQ-8 Capability constants 0..40 — semantic summary:

| Cap | Constant | Semantic role |
|---|---|---|
| 0 | `CAP_CHOWN` | chown override |
| 1 | `CAP_DAC_OVERRIDE` | bypass DAC R/W/X |
| 2 | `CAP_DAC_READ_SEARCH` | bypass DAC R + search |
| 3 | `CAP_FOWNER` | bypass file-owner-ID checks |
| 4 | `CAP_FSETID` | retain S_ISUID/S_ISGID across chown/chmod |
| 5 | `CAP_KILL` | signal other tasks regardless of UID |
| 6 | `CAP_SETGID` | setgid/setgroups + forged gid creds |
| 7 | `CAP_SETUID` | setuid + forged pid creds |
| 8 | `CAP_SETPCAP` | manipulate caps of other / self bounding+securebits |
| 9 | `CAP_LINUX_IMMUTABLE` | toggle S_IMMUTABLE / S_APPEND |
| 10 | `CAP_NET_BIND_SERVICE` | bind privileged port (<1024) |
| 11 | `CAP_NET_BROADCAST` | broadcast / multicast (unused on modern kernels) |
| 12 | `CAP_NET_ADMIN` | network admin (interfaces, firewall, routing) |
| 13 | `CAP_NET_RAW` | AF_PACKET, raw sockets |
| 14 | `CAP_IPC_LOCK` | mlock / SHM_LOCK |
| 15 | `CAP_IPC_OWNER` | bypass SysV IPC perms |
| 16 | `CAP_SYS_MODULE` | load/unload kernel modules |
| 17 | `CAP_SYS_RAWIO` | ioperm / iopl / raw USB |
| 18 | `CAP_SYS_CHROOT` | chroot |
| 19 | `CAP_SYS_PTRACE` | ptrace any task |
| 20 | `CAP_SYS_PACCT` | process-accounting config |
| 21 | `CAP_SYS_ADMIN` | grab-bag (mount, hostname, quotas, secure-attention, swap on/off, …) |
| 22 | `CAP_SYS_BOOT` | reboot / kexec |
| 23 | `CAP_SYS_NICE` | priority / scheduling / cpu-affinity / realtime ioprio |
| 24 | `CAP_SYS_RESOURCE` | override rlimits, quotas, reserved space, msgqueue size |
| 25 | `CAP_SYS_TIME` | clock manipulation |
| 26 | `CAP_SYS_TTY_CONFIG` | TTY config, vhangup |
| 27 | `CAP_MKNOD` | privileged mknod |
| 28 | `CAP_LEASE` | take file leases |
| 29 | `CAP_AUDIT_WRITE` | write audit records (unicast) |
| 30 | `CAP_AUDIT_CONTROL` | configure auditd |
| 31 | `CAP_SETFCAP` | set file caps; map uid=0 into child userns |
| 32 | `CAP_MAC_OVERRIDE` | bypass LSM MAC policy |
| 33 | `CAP_MAC_ADMIN` | configure LSM MAC policy |
| 34 | `CAP_SYSLOG` | configure printk + consume kmsg |
| 35 | `CAP_WAKE_ALARM` | arm wake alarms (RTC_WKALM, timer_create CLOCK_*_ALARM) |
| 36 | `CAP_BLOCK_SUSPEND` | EPOLLWAKEUP, /proc/sys/kernel/print-fatal-signals |
| 37 | `CAP_AUDIT_READ` | read audit log (multicast READLOG) |
| 38 | `CAP_PERFMON` | perf_event_open + i915_perf + privileged observability |
| 39 | `CAP_BPF` | BPF map create, advanced verifier, BTF load, JIT readback |
| 40 | `CAP_CHECKPOINT_RESTORE` | CRIU ops, clone3 pid selection, ns_last_pid write |

`CAP_LAST_CAP = 40`; `cap_valid(x)` ⇔ `0 ≤ x ≤ 40`.

REQ-9 CAP_BPF / CAP_PERFMON / CAP_SYS_ADMIN interaction (per header doc-block):
- `CAP_BPF` permits: creating all BPF map types; advanced verifier features (indirect var access, bounded loops, BPF↔BPF calls, scalar-precision tracking, larger complexity limits, dead-code elimination); BTF load; xlated/JITed prog readback; `bpf_spin_lock()`.
- `CAP_PERFMON` relaxes the verifier further: ptr↔int conversion in BPF; speculation-attack hardening bypass; `bpf_probe_read` arbitrary kernel memory; `bpf_trace_printk` kernel memory.
- `CAP_SYS_ADMIN` is still required to use `bpf_probe_write_user`, to iterate all loaded BPF objects (progs/maps/links/BTFs) and convert IDs ↔ fds.
- Loading tracing programs requires `CAP_PERFMON + CAP_BPF`.
- Loading networking BPF programs requires `CAP_NET_ADMIN + CAP_BPF`.

REQ-10 `CAP_TO_INDEX(x)` / `CAP_TO_MASK(x)`:
- `CAP_TO_INDEX(x) = x >> 5` ⇒ 0 for caps 0..31, 1 for caps 32..63.
- `CAP_TO_MASK(x) = 1U << (x & 31)` ⇒ the bit within the indexed u32.
- Userspace uses these to manipulate the `data[]` array.

REQ-11 `securebits` interaction (not strictly in this header but ABI-relevant):
- `SECBIT_NOROOT`: disables the "uid 0 ⇒ full caps" magic.
- `SECBIT_NO_SETUID_FIXUP`: disables the kernel's auto cap-clear on setuid transition.
- `SECBIT_KEEP_CAPS`: preserve permitted caps across setuid-to-non-zero.
- All gated by `CAP_SETPCAP`; manipulated via `prctl(PR_SET_SECUREBITS)`.

REQ-12 VFS-cap xattr `security.capability` decoding:
- First `__le32` = `magic_etc = (revision << 0) | flags`.
- `revision = magic_etc & VFS_CAP_REVISION_MASK` (0xFF000000) >> 24 → 1, 2, or 3.
- `flags = magic_etc & VFS_CAP_FLAGS_MASK` (0x00FFFFFF) → `VFS_CAP_FLAGS_EFFECTIVE` (0x000001) is the only currently defined bit ("e" in `setcap cap_net_raw+ep file`).
- Total size = `XATTR_CAPS_SZ_{1,2,3}`: 12 / 20 / 24 bytes.

REQ-13 `struct vfs_cap_data` (revisions 1 and 2):
- `magic_etc`: revision + flags packed.
- `data[VFS_CAP_U32]`: 1 entry for rev1 (32-bit caps), 2 entries for rev2 (64-bit caps).
- Each entry = `{ __le32 permitted; __le32 inheritable; }`.
- Effective set is **implicit**: if `VFS_CAP_FLAGS_EFFECTIVE` set ⇒ effective = permitted ∩ (mask of file's effective), else effective = empty.

REQ-14 `struct vfs_ns_cap_data` (revision 3):
- Same layout as `vfs_cap_data` rev2 (magic_etc + 2 data entries) **plus** a trailing `__le32 rootid`.
- `rootid` is the uid_t of the user-namespace root that owns these caps; the file caps only "activate" when the executing task is in a userns whose `uid_map` maps `rootid` to 0.
- Total size = `XATTR_CAPS_SZ_3 = 24 bytes`.

REQ-15 Conversion between `vfs_cap_data` (rev1/2) and `vfs_ns_cap_data` (rev3):
- Read: kernel auto-detects via `magic_etc & VFS_CAP_REVISION_MASK`.
- Write via `setxattr("security.capability", buf, size)`: size selects revision; userspace tools (`libcap`) emit rev3 by default since libcap2 v2.27.

REQ-16 Execve cap-transition (covered here for completeness — implementation in `kernel/cred.c`):
- New_permitted = (file_permitted ∩ bounding) ∪ (file_inheritable ∩ inheritable).
- New_effective = file_effective ? new_permitted : ambient_subset.
- New_inheritable = old_inheritable (preserved).
- New_ambient = old_ambient ∩ new_permitted ∩ new_inheritable (auto-drops on raise).
- If setuid root binary: file_effective = file_inheritable = file_permitted = full-cap-mask (legacy "set-user-ID-root → full caps") **unless** SECBIT_NOROOT set.
- If `PR_SET_NO_NEW_PRIVS`: setuid + file-caps cap-acquisition is suppressed (cannot gain new caps via execve).

REQ-17 `cap_valid(x)`:
- Macro: `(x) >= 0 && (x) <= CAP_LAST_CAP`.
- Validates a capability constant before use in `capable()` or set manipulation.

REQ-18 Backward compatibility:
- Old userspace using `_LINUX_CAPABILITY_VERSION_1` only sees caps 0..31; caps 32..63 are silently invisible.
- Kernel rejects `capset` from v1 userspace if doing so would lose caps 32..63 that are currently set; reply is -EINVAL or the kernel preserves them silently depending on kernel-version policy.

REQ-19 ABI stability:
- Capability constants 0..40 are frozen; new caps append at higher numbers and bump `CAP_LAST_CAP`.
- Version magics frozen; new ABI features require a new version magic.
- `vfs_cap_data` rev1/2 layouts frozen; rev3 added `rootid` for userns.
- `__user_cap_header_struct` / `__user_cap_data_struct` layouts frozen.

REQ-20 Permission to call `capset` on `pid != 0`:
- Kernel returns -EPERM unconditionally on the syscall (file-cap + execve is the only supported cross-task mechanism).
- Some pre-v3-era documentation mentions cross-pid capset; that has been removed.

REQ-21 `header->pid == 0` ⇒ current task; user-supplied pid that equals `current->pid` is also accepted.

REQ-22 File-cap effective-bit "ep" vs "p":
- `setcap cap_net_raw+ep file` ⇒ `magic_etc.flags |= VFS_CAP_FLAGS_EFFECTIVE`.
- `setcap cap_net_raw+p file` ⇒ flag clear; new task must explicitly `capset()` to move permitted into effective.

REQ-23 Ambient-cap subset rule:
- A cap may be in the ambient set only if it is in **both** the inheritable and permitted sets.
- `prctl(PR_CAP_AMBIENT, PR_CAP_AMBIENT_RAISE, cap)` fails with -EPERM if cap ∉ P ∩ I.
- Modifying permitted or inheritable that drops a cap auto-clears it from ambient.

REQ-24 Bounding-set rule:
- A cap may be added to inheritable only if it is in the bounding set.
- `prctl(PR_CAPBSET_DROP, cap)` requires `CAP_SETPCAP`.
- Bounding-set drops are irreversible for the life of the task.

REQ-25 `securebits & SECBIT_NO_CAP_AMBIENT_RAISE`:
- When set, `PR_CAP_AMBIENT_RAISE` permanently fails ⇒ -EPERM.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `capget_version_dispatch` | INVARIANT | per-capget: unknown version ⇒ write VERSION_3 to user + -EINVAL. |
| `capset_pid_check` | INVARIANT | per-capset: hdr.pid != 0 ∧ != current.pid ⇒ -EPERM. |
| `capset_permitted_monotonic_down` | INVARIANT | per-capset: new_perm ⊆ old_perm; else -EPERM. |
| `capset_effective_subset_permitted` | INVARIANT | per-capset: new_eff ⊆ new_perm; else -EPERM. |
| `capset_inheritable_within_old_inh_or_perm` | INVARIANT | per-capset: new_inh ⊆ old_inh ∪ old_perm. |
| `capset_inheritable_within_bounding` | INVARIANT | per-capset: new_inh ⊆ bounding ∪ old_inh. |
| `bounding_monotonic_down` | INVARIANT | per-PR_CAPBSET_DROP: bounding only shrinks. |
| `ambient_subset_perm_inh` | INVARIANT | per-ambient: ambient ⊆ permitted ∩ inheritable always. |
| `cap_in_range` | INVARIANT | per-cap-use: 0 ≤ cap ≤ CAP_LAST_CAP. |
| `xattr_size_revision_match` | INVARIANT | per-xattr-set: size ∈ {12, 20, 24} ∧ matches magic_etc revision. |
| `xattr_flags_legal` | INVARIANT | per-xattr-set: flags & ~VFS_CAP_FLAGS_EFFECTIVE == 0. |
| `xattr_rootid_userns_check` | INVARIANT | per-rev3: rootid mappable in current userns or file-caps treated as absent. |
| `nnp_blocks_cap_acquire` | INVARIANT | per-execve: PR_SET_NO_NEW_PRIVS ⇒ new caps ⊆ old caps. |
| `securebit_secbit_noroot_disables_root_magic` | INVARIANT | per-setuid-root execve: SECBIT_NOROOT ⇒ no auto-full-caps. |

### Layer 2: TLA+

`uapi/capability.tla`:
- Models per-task `{cap_effective, cap_permitted, cap_inheritable, cap_bounding, cap_ambient}`, securebits, file-caps xattr, execve transitions.
- Properties:
  - `safety_permitted_monotonic_within_lifetime` — per-capset path, permitted never grows beyond initial bounding.
  - `safety_bounding_monotonic_down` — bounding only shrinks for life of task.
  - `safety_ambient_subset_invariant` — ambient ⊆ permitted ∩ inheritable always.
  - `safety_execve_no_new_privs` — after `PR_SET_NO_NEW_PRIVS`, new caps ⊆ old caps.
  - `safety_setuid_root_caps_gated_by_secbit_noroot` — with NOROOT, setuid-root binary does not auto-acquire full caps.
  - `safety_file_caps_v3_userns_scoped` — rev3 file caps only effective when rootid maps to 0 in current userns.
  - `safety_capset_cross_pid_forbidden` — capset with foreign pid always -EPERM.
  - `liveness_capget_eventually_returns_caps_or_negotiates_version` — every capget either returns data or instructs a retry with the preferred version.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Capability::capget` post: hdr.version unchanged on success ∧ data triples populated per version | `Capability::capget` |
| `Capability::capset` post: cred swapped iff all rule checks pass | `Capability::capset` |
| `Capability::file_caps_apply` post: new_perm = (file_perm ∩ bounding) ∪ (file_inh ∩ task_inh); new_eff = flag_e ? new_perm : ambient ∩ new_perm | `Capability::file_caps_apply` |
| `Capability::xattr_validate` post: ret == Ok ⇒ size matches revision ∧ flags legal | `Capability::xattr_validate` |
| `Capability::ambient_raise` post: cap was in permitted ∩ inheritable ∧ ambient_subset invariant preserved | `Capability::ambient_raise` |
| `Capability::bounding_drop` post: bounding[cap] = 0 ∧ ambient[cap] = 0 ∧ caller had CAP_SETPCAP | `Capability::bounding_drop` |

### Layer 4: Verus/Creusot functional

Per `Documentation/security/credentials.rst`, `kernel/capability.c`, `security/commoncap.c`:
- `capget` / `capset` op semantic equivalence with upstream.
- File-cap execve transformation equivalence with `cap_bprm_creds_from_file`.
- Ambient-set raise / lower equivalence with `cap_prctl_drop`.
- Bounding-set drop equivalence with `cap_capset` plus `PR_CAPBSET_DROP`.
- `vfs_ns_cap_data.rootid` userns mapping equivalence with `rootid_owns_currentns`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

capability UAPI reinforcement:

- **Per-version negotiation (unknown ⇒ -EINVAL + preferred)** — defense against per-silent-version-drift.
- **Per-`capset` pid-must-be-self** — defense against per-cross-task cap escalation.
- **Per-`new_perm ⊆ old_perm`** — defense against per-permitted-set growth.
- **Per-`new_eff ⊆ new_perm`** — defense against per-effective-without-permitted.
- **Per-`new_inh ⊆ old_inh ∪ old_perm`** — defense against per-inheritable-escape.
- **Per-`new_inh ⊆ bounding ∪ old_inh`** — defense against per-bounding bypass via inheritable.
- **Per-ambient auto-drop on perm/inh shrink** — defense against per-orphan-ambient.
- **Per-bounding monotonic-down** — defense against per-cap-resurrection.
- **Per-`PR_SET_NO_NEW_PRIVS`** — defense against per-setuid + file-caps escalation.
- **Per-`SECBIT_NOROOT`** — defense against per-uid-0-magic-acquisition.
- **Per-xattr size/revision cross-check** — defense against per-truncated-xattr confusion.
- **Per-xattr `flags & ~FLAG_EFFECTIVE == 0`** — defense against per-future-bit injection.
- **Per-`vfs_ns_cap_data.rootid` userns check** — defense against per-cross-userns file-cap activation.
- **Per-`CAP_SETPCAP` required for bounding-set / securebit changes** — defense against per-unprivileged bounding-tamper.
- **Per-`CAP_SETFCAP` required to set file caps** — defense against per-arbitrary file-cap install.
- **Per-`CAP_LAST_CAP` strict** — defense against per-invalid-cap probe.
- **Per-`CAP_TO_INDEX` / `CAP_TO_MASK` arithmetic-bound** — defense against per-out-of-array.
- **Per-`securebits` lock bits** — defense against per-runtime-securebit-relax (paired SECBIT_*_LOCKED bits).
- **Per-`AUDIT_BPRM_FCAPS` record on fcap-elevated execve** — defense against per-fcap-stealth-escalation.
- **Per-`AUDIT_CAPSET` record on every capset** — defense against per-cap-juggle blindspot.

### grsecurity/pax-style reinforcement

- **PaX UDEREF/USERCOPY on capset user-blob** — `copy_from_user` of `struct __user_cap_header_struct` (8 bytes) and `struct __user_cap_data_struct` (12 bytes × U32S_1/2/3 per version) uses hardened-usercopy with explicit per-version size bounds (12 / 24 / 24 bytes for data); SMAP/SMEP active; the kernel reads `hdr.version` first and only then copies the per-version-sized data buffer.
- **GRKERNSEC_FILE_CAPS_DISABLE** — boot-time kernel parameter / config option (`file_caps_enabled = 0`) globally disables file capabilities: `security.capability` xattrs are ignored on execve, all setuid-root binaries follow the legacy "full caps on uid-0 binary" path subject to `SECBIT_NOROOT`; defense against per-fcap-stealth-elevation by an attacker who can write to a binary's xattrs. The toggle is one-way (disable-only) for life of kernel under grsec policy.
- **`file_caps_enabled` boot toggle** — `CONFIG_SECURITY_FILE_CAPABILITIES` controls compile-time enablement; the boot parameter `no_file_caps` (or grsec equivalent) disables at runtime; once disabled, cannot be re-enabled without reboot.
- **No-new-privs blocks suid+caps acquisition** — `prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0)` is a self-imposed one-way switch; once set, `execve` never raises caps via file-caps and never honors setuid/setgid bits on the executed binary; new caps ⊆ old caps strictly. Under grsec, `PR_SET_NO_NEW_PRIVS` is also implicit for any seccomp-filtered task.
- **CAP_OPT_NOAUDIT_ON_NS_TRANSITION** — grsec extension flag (passed via `capable_noaudit` family) suppresses spurious AUDIT_AVC / AUDIT_CAPSET records when a task transitions across user-namespaces while logically retaining its capability profile (e.g., setns into a sibling userns it already has CAP_SYS_ADMIN over); defense against per-audit-flood during normal container init while preserving real privilege-escalation records.
- **Per-`CAP_SYS_ADMIN` audited under grsec** — every `capable(CAP_SYS_ADMIN)` check that succeeds is candidate for audit (grsec policy may sample) since CAP_SYS_ADMIN is the catch-all; defense against per-CAP_SYS_ADMIN abuse going unnoticed.
- **GRKERNSEC_HIDESYM** — `getxattr("security.capability", ...)` for a file the caller does not own returns -ENODATA under grsec for unprivileged callers (file-cap topology is information-disclosure); the `vfs_ns_cap_data.rootid` field is sanitized through the userns-uid map before being read by an unprivileged caller in a child userns.
- **PAX_RANDKSTACK** — `capset(2)` and `capget(2)` entry paths re-randomize kernel-stack base; defense against per-cap-set-probe stack-layout leak.
- **PAX_KERNEXEC** — the `commoncap.c` permission-check path lives in W^X kernel text; `cap_capable` is jump-table-dispatched with KERNEXEC-safe indirection (no user-controllable function pointers).
- **GRKERNSEC_HARDEN_PTRACE** — `CAP_SYS_PTRACE` is necessary but not sufficient to ptrace a target under grsec: the caller must also satisfy `gr_ptrace_allowed(target)` (cross-uid / cross-gid restrictions; uid 0 cannot ptrace non-uid-0 outside its own session under default grsec policy).
- **Per-`CAP_SETFCAP` gated** — setting file capabilities via `setxattr("security.capability", ...)` requires `CAP_SETFCAP`; under grsec, additionally requires that the writer's userns own the file's mount; defense against per-file-cap install on shared-mount filesystems.
- **Per-`CAP_SYS_MODULE` audited and gated** — under grsec, module loading additionally requires `modules_disabled = 0` (a one-way sysctl-lock); a CAP_SYS_MODULE-holder cannot load modules once `modules_disabled = 1`. Defense against per-rootkit-via-module after initial setup.
- **Per-`CAP_SYS_RAWIO`** — under grsec, even CAP_SYS_RAWIO does not allow `ioperm/iopl` (defense against per-userland-direct-port-IO ROP gadget); `iopl` is unconditionally -EPERM under grsec.
- **Per-`CAP_SETUID` + `CAP_SETGID` securebits-locked** — `SECBIT_KEEP_CAPS_LOCKED`, `SECBIT_NO_SETUID_FIXUP_LOCKED`, `SECBIT_NOROOT_LOCKED`, `SECBIT_NO_CAP_AMBIENT_RAISE_LOCKED` may be set once via `PR_SET_SECUREBITS` (with CAP_SETPCAP) and then become immutable for life of task; under grsec, the default for unprivileged-shell-spawned children is to have NOROOT and NO_SETUID_FIXUP locked.
- **Per-`CAP_BPF` + `CAP_PERFMON` separated from `CAP_SYS_ADMIN`** — under grsec, default policy: unprivileged BPF entirely disabled; with `kernel.unprivileged_bpf_disabled = 1`, `CAP_BPF` is required even for self-process map creation; further, `bpf_probe_write_user` is removed from the kernel entirely under grsec (no CAP_SYS_ADMIN escape).
- **Per-`CAP_CHECKPOINT_RESTORE`** — under grsec, CRIU operations gated additionally on `kernel.unprivileged_userns_clone = 0`; defense against per-CRIU-assisted userns escape.
- **Per-`vfs_cap_data` rev1/rev2 deprecated under grsec** — only rev3 (`vfs_ns_cap_data` with `rootid`) is honored when `file_caps_enabled = 1` under grsec policy; defense against per-userns-escape via rev2 file-caps that have no userns binding.
- **Per-xattr `magic_etc` sanity** — `magic_etc & VFS_CAP_REVISION_MASK >> 24 ∈ {1, 2, 3}` strict; `flags & ~VFS_CAP_FLAGS_EFFECTIVE == 0` strict; reserved-bits non-zero ⇒ ENOTSUP (refuse to interpret).
- **Per-`CAP_AUDIT_CONTROL`** — under grsec, additionally requires CAP_SYS_ADMIN (since auditd takeover can suppress security records); defense against per-audit-suppress with only narrow CAP_AUDIT_CONTROL.
- **Per-`CAP_MAC_OVERRIDE`** — under grsec, requires that the task's LSM context dominate the policy being overridden; pure CAP_MAC_OVERRIDE without LSM-context-dominance is insufficient.
- **Per-`CAP_SYSLOG`** — under grsec, `dmesg_restrict = 1` is locked; even CAP_SYSLOG cannot relax it back to 0 for life of kernel.
- **Per-`PR_CAPBSET_DROP`** — drop is irreversible and logged via AUDIT_CAPSET; under grsec, bounding-set drops are propagated to the userns "init" task so a child userns cannot raise dropped caps even via setns to a parent userns.
- **Per-`PR_CAP_AMBIENT_RAISE`** — additionally gated on `securebits & SECBIT_NO_CAP_AMBIENT_RAISE == 0`; under grsec, the SECBIT can be locked at boot via kernel parameter.
- **Per-`CAP_DAC_OVERRIDE` / `CAP_DAC_READ_SEARCH`** — under grsec ChrootRestrictions, these caps are stripped from any task that is the descendant of a `chroot()` call (defense against per-CAP_DAC-bypass of chroot jail).
- **Per-`CAP_SYS_CHROOT`** — under grsec, chroot escape mitigations (PaX chroot pivot, no fchdir-out, no mknod inside chroot) are independent of cap presence; defense against per-classic-chroot-escape.

