# Tier-5 UAPI: include/uapi/asm-generic/ioctl.h — ioctl(2) Encoding ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/asm-generic/ioctl.h (~107 lines)
-->

## Summary

The ioctl encoding UAPI is the canonical scheme used by Linux to pack the four components of an `ioctl(2)` request number into a single 32-bit `unsigned int`: the **direction** (none/read/write/read-write), the **type** ("magic" subsystem byte), the **command number** (per-subsystem sequence), and the **parameter size** (bytes of the userspace argument struct). Every Linux driver writes its ioctl numbers through this scheme so that `_IOC_DIR`/`_IOC_TYPE`/`_IOC_NR`/`_IOC_SIZE` can decode the request and so that the generic core can detect "old binaries calling new kernels" by mismatched sizes.

This Tier-5 covers the asm-generic encoding header. It defines:

- **Bit-field widths**: `_IOC_NRBITS = 8`, `_IOC_TYPEBITS = 8`, `_IOC_SIZEBITS = 14`, `_IOC_DIRBITS = 2`. Total = 32. Arches MAY override `_IOC_SIZEBITS` and `_IOC_DIRBITS` before including (PowerPC, MIPS, SPARC, Alpha do — they swap to `_IOC_SIZEBITS = 13`, `_IOC_DIRBITS = 3`).
- **Bit-position masks**: `_IOC_NRMASK = (1 << NRBITS) - 1 = 0xFF`, `_IOC_TYPEMASK = 0xFF`, `_IOC_SIZEMASK = (1 << SIZEBITS) - 1 = 0x3FFF`, `_IOC_DIRMASK = (1 << DIRBITS) - 1 = 0x3`.
- **Bit shifts**: `_IOC_NRSHIFT = 0`, `_IOC_TYPESHIFT = 8`, `_IOC_SIZESHIFT = 16`, `_IOC_DIRSHIFT = 30`.
- **Direction values**: `_IOC_NONE = 0`, `_IOC_WRITE = 1`, `_IOC_READ = 2`, `_IOC_READ | _IOC_WRITE = 3`. Arches may override (PowerPC: NONE=1, READ=2, WRITE=4 — yes, exclusive bits).
- **Construction macros**: `_IOC(dir, type, nr, size)` packs all four; `_IO(type, nr)` for argument-less; `_IOR(type, nr, T)` for kernel-writes-userspace-reads; `_IOW(type, nr, T)` for userspace-writes-kernel-reads; `_IOWR(type, nr, T)` for bidirectional. The `_BAD` variants (`_IOR_BAD`/`_IOW_BAD`/`_IOWR_BAD`) bypass `_IOC_TYPECHECK` and accept any expression as the type — used for historical drivers whose argument types are not statically known.
- **Decoder macros**: `_IOC_DIR(nr)`, `_IOC_TYPE(nr)`, `_IOC_NR(nr)`, `_IOC_SIZE(nr)`.
- **Driver-compat aliases**: `IOC_IN` (alias for `_IOC_WRITE << _IOC_DIRSHIFT`), `IOC_OUT` (alias for `_IOC_READ << _IOC_DIRSHIFT`), `IOC_INOUT`, `IOCSIZE_MASK` (= `_IOC_SIZEMASK << _IOC_SIZESHIFT`), `IOCSIZE_SHIFT` (= `_IOC_SIZESHIFT`).
- **`_IOC_TYPECHECK(t)`**: a macro that returns `sizeof(t)` and is **only defined for non-kernel builds**; it gives the compiler a chance to syntactically verify `t` is a type expression at macro expansion. Kernel code includes asm-generic with `__KERNEL__` defined and therefore does not see this macro — kernel code calling `_IOR(...)` from within the kernel must encode sizes manually or use `_IOC_*_BAD`.

Critical for: every device driver, every filesystem-private ioctl, every udev/sysfs path, every virtualization control plane (KVM, vfio), every USB/MMC/SCSI/DRM/V4L2/ALSA control flow, every container-runtime that needs to talk to `/dev/*`. The scheme is **frozen ABI** — programs from 1996 still work because the bit positions have never changed.

This Tier-5 covers `include/uapi/asm-generic/ioctl.h` (~107 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `_IOC_NRBITS`/`TYPEBITS`/`SIZEBITS`/`DIRBITS` | per-field width | `Ioc::{NRBITS,TYPEBITS,SIZEBITS,DIRBITS}` |
| `_IOC_NRMASK`/`TYPEMASK`/`SIZEMASK`/`DIRMASK` | per-field mask | `Ioc::{NRMASK,TYPEMASK,SIZEMASK,DIRMASK}` |
| `_IOC_NRSHIFT`/`TYPESHIFT`/`SIZESHIFT`/`DIRSHIFT` | per-field shift | `Ioc::{NRSHIFT,TYPESHIFT,SIZESHIFT,DIRSHIFT}` |
| `_IOC_NONE`/`READ`/`WRITE` | per-direction bit | `IocDir::{NONE,READ,WRITE}` |
| `_IOC` | per-pack macro | `Ioc::pack` |
| `_IO` / `_IOR` / `_IOW` / `_IOWR` | per-construct shorthand | `Ioc::{io,ior,iow,iowr}` |
| `_IOR_BAD` / `_IOW_BAD` / `_IOWR_BAD` | per-non-typechecked construct | `Ioc::{ior_bad,iow_bad,iowr_bad}` |
| `_IOC_DIR` / `_IOC_TYPE` / `_IOC_NR` / `_IOC_SIZE` | per-decode | `Ioc::{dir,type_,nr,size}` |
| `_IOC_TYPECHECK` | per-userspace sizeof check | `Ioc::typecheck` (userspace ABI helper) |
| `IOC_IN` / `IOC_OUT` / `IOC_INOUT` | per-driver-compat alias | `Ioc::{IN,OUT,INOUT}` |
| `IOCSIZE_MASK` / `IOCSIZE_SHIFT` | per-mask helpers | `Ioc::{IOCSIZE_MASK,IOCSIZE_SHIFT}` |
| `do_vfs_ioctl` | per-`ioctl(2)` core | `Vfs::do_ioctl` |
| `vfs_ioctl` | per-file `unlocked_ioctl` dispatch | `Vfs::ioctl_dispatch` |
| `compat_ioctl` | per-32-on-64 ioctl helper | `Vfs::compat_ioctl` |

## ABI surface (constants + structs)

### Bit-field widths (`ioctl.h:23-37`)

```text
_IOC_NRBITS   = 8                /* command-number field */
_IOC_TYPEBITS = 8                /* "magic" subsystem type byte */
_IOC_SIZEBITS = 14               /* userspace arg-struct size (bytes) */
_IOC_DIRBITS  = 2                /* direction code */

/* arch-overridable via <asm/ioctl.h> before including this file */
```

Sum: `8 + 8 + 14 + 2 = 32`. The arches that override (PowerPC, MIPS, Alpha, SPARC) widen `_IOC_DIRBITS` to 3 and narrow `_IOC_SIZEBITS` to 13 so that the direction can be expressed as mutually-exclusive bits.

### Field masks (`ioctl.h:39-42`)

```text
_IOC_NRMASK   = (1 << _IOC_NRBITS)   - 1 = 0xFF
_IOC_TYPEMASK = (1 << _IOC_TYPEBITS) - 1 = 0xFF
_IOC_SIZEMASK = (1 << _IOC_SIZEBITS) - 1 = 0x3FFF
_IOC_DIRMASK  = (1 << _IOC_DIRBITS)  - 1 = 0x03
```

### Field shifts (`ioctl.h:44-47`)

```text
_IOC_NRSHIFT   = 0
_IOC_TYPESHIFT = _IOC_NRSHIFT   + _IOC_NRBITS    =  8
_IOC_SIZESHIFT = _IOC_TYPESHIFT + _IOC_TYPEBITS  = 16
_IOC_DIRSHIFT  = _IOC_SIZESHIFT + _IOC_SIZEBITS  = 30
```

Layout of a packed 32-bit ioctl number:

```
 bit 31         30 29                16 15            8 7             0
+-----+-----------+--------------------+---------------+---------------+
| DIR (2 bits)    |   SIZE (14 bits)   |  TYPE (8b)    |   NR (8b)     |
+-----+-----------+--------------------+---------------+---------------+
```

### Direction values (`ioctl.h:57-67`)

```text
_IOC_NONE  = 0U   /* no argument */
_IOC_WRITE = 1U   /* userspace → kernel: kernel reads user buffer */
_IOC_READ  = 2U   /* kernel → userspace: kernel writes user buffer */
_IOC_READ | _IOC_WRITE = 3U   /* bidirectional */
```

The comment in the header is explicit: `_IOC_WRITE` means userland is **writing** (kernel is reading the buffer); `_IOC_READ` means userland is **reading** (kernel is writing the buffer). This is the opposite of what beginners often expect.

### Construction macros (`ioctl.h:69-91`)

```text
_IOC(dir, type, nr, size) =
    ((dir)  << _IOC_DIRSHIFT)  |
    ((type) << _IOC_TYPESHIFT) |
    ((nr)   << _IOC_NRSHIFT)   |
    ((size) << _IOC_SIZESHIFT)

_IO(type, nr)                = _IOC(_IOC_NONE,  (type), (nr), 0)
_IOR(type, nr, argtype)      = _IOC(_IOC_READ,  (type), (nr), _IOC_TYPECHECK(argtype))
_IOW(type, nr, argtype)      = _IOC(_IOC_WRITE, (type), (nr), _IOC_TYPECHECK(argtype))
_IOWR(type, nr, argtype)     = _IOC(_IOC_READ | _IOC_WRITE, (type), (nr), _IOC_TYPECHECK(argtype))
_IOR_BAD(type, nr, argtype)  = _IOC(_IOC_READ,  (type), (nr), sizeof(argtype))
_IOW_BAD(type, nr, argtype)  = _IOC(_IOC_WRITE, (type), (nr), sizeof(argtype))
_IOWR_BAD(type, nr, argtype) = _IOC(_IOC_READ | _IOC_WRITE, (type), (nr), sizeof(argtype))

/* userspace only: */
_IOC_TYPECHECK(t) = sizeof(t)
```

The `_BAD` variants exist because `_IOC_TYPECHECK` is implemented as a sizeof expression — useful for static analysis but fails when `argtype` is not a type-expression (e.g., is a variable or a parenthesized expression). The `_BAD` variants accept any `sizeof(...)` arg without the typecheck syntactic guard.

### Decoder macros (`ioctl.h:93-97`)

```text
_IOC_DIR(nr)  = ((nr) >> _IOC_DIRSHIFT)  & _IOC_DIRMASK
_IOC_TYPE(nr) = ((nr) >> _IOC_TYPESHIFT) & _IOC_TYPEMASK
_IOC_NR(nr)   = ((nr) >> _IOC_NRSHIFT)   & _IOC_NRMASK
_IOC_SIZE(nr) = ((nr) >> _IOC_SIZESHIFT) & _IOC_SIZEMASK
```

### Driver-compat aliases (`ioctl.h:101-105`)

```text
IOC_IN         = (_IOC_WRITE << _IOC_DIRSHIFT)            = 0x40000000
IOC_OUT        = (_IOC_READ  << _IOC_DIRSHIFT)            = 0x80000000
IOC_INOUT      = ((_IOC_WRITE | _IOC_READ) << _IOC_DIRSHIFT) = 0xC0000000
IOCSIZE_MASK   = (_IOC_SIZEMASK << _IOC_SIZESHIFT)        = 0x3FFF0000
IOCSIZE_SHIFT  = _IOC_SIZESHIFT                           = 16
```

`IOC_IN` / `IOC_OUT` / `IOC_INOUT` are used by sound and SCSI drivers that predate `_IOC` and need to OR the direction bits directly into a precomputed command. `IOCSIZE_MASK`/`SHIFT` lets a driver extract just the size field via `(cmd & IOCSIZE_MASK) >> IOCSIZE_SHIFT`.

## Compatibility contract

REQ-1: `_IOC_NRBITS == 8`, `_IOC_TYPEBITS == 8`, `_IOC_SIZEBITS == 14`, `_IOC_DIRBITS == 2` on x86_64 (Rookery primary arch). These constants MUST NOT be redefined by any in-tree header except the per-arch overrides (PowerPC/MIPS/SPARC/Alpha).

REQ-2: `_IOC_NRMASK = 0xFF`, `_IOC_TYPEMASK = 0xFF`, `_IOC_SIZEMASK = 0x3FFF`, `_IOC_DIRMASK = 0x3`. Derived from the bit widths; MUST hold algebraically (`(1 << N) - 1`).

REQ-3: `_IOC_NRSHIFT = 0`, `_IOC_TYPESHIFT = 8`, `_IOC_SIZESHIFT = 16`, `_IOC_DIRSHIFT = 30`. Sum-of-prefix invariant: each shift equals the previous shift plus the previous bit width.

REQ-4: `_IOC_NONE = 0`, `_IOC_WRITE = 1`, `_IOC_READ = 2`. The combined bidirectional value is `_IOC_READ | _IOC_WRITE = 3`. Userspace and kernel MUST agree on these values; they are part of the cross-binary ABI.

REQ-5: `_IOC(dir, type, nr, size)` MUST pack all four fields into the canonical 32-bit layout — `((dir) << 30) | ((size) << 16) | ((type) << 8) | (nr)` — masked appropriately to fit each field width. The caller is responsible for ensuring `dir < 4`, `type < 256`, `nr < 256`, `size < 16384`. Overflow past field widths produces a malformed cmd (drivers MUST sanity-check on input).

REQ-6: `_IO(type, nr)` MUST encode `dir = _IOC_NONE`, `size = 0`. `_IOR`/`_IOW`/`_IOWR` MUST encode the matching direction and `size = sizeof(argtype)`. The `_BAD` variants bypass the syntactic typecheck but produce identical encodings.

REQ-7: `_IOC_DIR(cmd)` MUST return the 2-bit direction; `_IOC_TYPE(cmd)` the 8-bit magic; `_IOC_NR(cmd)` the 8-bit sequence number; `_IOC_SIZE(cmd)` the 14-bit argument size. Decoded values are inputs to per-driver dispatchers.

REQ-8: `IOC_IN == 0x40000000`, `IOC_OUT == 0x80000000`, `IOC_INOUT == 0xC0000000`. These literal values are baked into sound drivers (ALSA OSS-emulation) and SCSI generic — MUST NOT change.

REQ-9: `IOCSIZE_MASK == 0x3FFF0000`, `IOCSIZE_SHIFT == 16`. A driver computing `(cmd & IOCSIZE_MASK) >> IOCSIZE_SHIFT` MUST get the same value as `_IOC_SIZE(cmd)`.

REQ-10: A kernel-side ioctl dispatcher MUST validate `_IOC_DIR(cmd)` against direction bits expected for the cmd number. If `_IOC_DIR(cmd) & _IOC_WRITE`, the kernel MUST `access_ok` the user buffer for read. If `& _IOC_READ`, MUST `access_ok` for write. The generic `do_vfs_ioctl` performs this check before calling `file->f_op->unlocked_ioctl`.

REQ-11: The generic core MUST reject ioctls with size `> _IOC_SIZEMASK == 16383` bytes (this is the structural maximum; larger arg-structs require a different design — typically passing a pointer-to-pointer in the argument).

REQ-12: An "old binary on new kernel" mismatch — where the driver's expected `_IOC_SIZE` differs from the userspace-supplied `_IOC_SIZE` — MUST be detectable by comparing `_IOC_SIZE(driver_cmd)` against `_IOC_SIZE(user_cmd)`. Drivers often log a warning and either zero-extend the smaller buffer or return `-EINVAL`.

REQ-13: Per-driver "magic" bytes (the `_IOC_TYPE` field) are documented in `Documentation/userspace-api/ioctl/ioctl-number.rst`. Rookery's drivers MUST use distinct magic bytes; assignments require a comment in that file.

REQ-14: `compat_ioctl` MUST decode `_IOC_SIZE` against the **32-bit** struct size for 32-on-64 ioctl translation. Misencoding here is the historical source of CVE-2010-2240-class info-leaks.

REQ-15: `_IOC_TYPECHECK(t)` is **userspace-only**. Kernel code defining new ioctl numbers MUST use the asm-generic header (which omits `_IOC_TYPECHECK` under `__KERNEL__`) and either include `<linux/ioctl.h>` with kernel semantics or use `_IOC_*_BAD` variants for non-type expressions.

REQ-16: `IOC_IN`/`IOC_OUT`/`IOC_INOUT` MUST be precomputed `(_IOC_* << _IOC_DIRSHIFT)` values, NOT recomputed at every call site. The macro expansion MUST collapse to a constant.

## Acceptance Criteria

- [ ] AC-1: `_IOC_NRBITS == 8 ∧ _IOC_TYPEBITS == 8 ∧ _IOC_SIZEBITS == 14 ∧ _IOC_DIRBITS == 2`.
- [ ] AC-2: `_IOC_NRMASK == 0xFF ∧ _IOC_TYPEMASK == 0xFF ∧ _IOC_SIZEMASK == 0x3FFF ∧ _IOC_DIRMASK == 0x3`.
- [ ] AC-3: `_IOC_NRSHIFT == 0 ∧ _IOC_TYPESHIFT == 8 ∧ _IOC_SIZESHIFT == 16 ∧ _IOC_DIRSHIFT == 30`.
- [ ] AC-4: `_IOC_NONE == 0 ∧ _IOC_WRITE == 1 ∧ _IOC_READ == 2`.
- [ ] AC-5: `_IO('K', 1) == ((_IOC_NONE << 30) | (0 << 16) | ('K' << 8) | 1) == 0x00004B01`.
- [ ] AC-6: `_IOR('K', 2, u32) == ((_IOC_READ << 30) | (4 << 16) | ('K' << 8) | 2) == 0x80044B02`.
- [ ] AC-7: `_IOW('K', 3, u64) == ((_IOC_WRITE << 30) | (8 << 16) | ('K' << 8) | 3) == 0x40084B03`.
- [ ] AC-8: `_IOWR('K', 4, u32) == ((_IOC_READ | _IOC_WRITE) << 30) | (4 << 16) | ('K' << 8) | 4) == 0xC0044B04`.
- [ ] AC-9: `_IOR_BAD('K', 5, expr) == _IOR('K', 5, T)` where `T` is the type of `expr` (verified by `sizeof`).
- [ ] AC-10: `_IOC_DIR(_IOR('K', 2, u32)) == _IOC_READ`.
- [ ] AC-11: `_IOC_TYPE(_IOR('K', 2, u32)) == 'K' == 0x4B`.
- [ ] AC-12: `_IOC_NR(_IOR('K', 2, u32)) == 2`.
- [ ] AC-13: `_IOC_SIZE(_IOR('K', 2, u32)) == 4`.
- [ ] AC-14: `IOC_IN == 0x40000000 ∧ IOC_OUT == 0x80000000 ∧ IOC_INOUT == 0xC0000000`.
- [ ] AC-15: `IOCSIZE_MASK == 0x3FFF0000 ∧ IOCSIZE_SHIFT == 16`.
- [ ] AC-16: `_IOC_SIZE(cmd) == (cmd & IOCSIZE_MASK) >> IOCSIZE_SHIFT` for every well-formed cmd.
- [ ] AC-17: `do_vfs_ioctl` rejects ioctls where `_IOC_SIZE > 16383` with `-EINVAL` (or routes them to a driver that has explicitly opted into pointer-indirection).
- [ ] AC-18: `do_vfs_ioctl` calls `access_ok(user_ptr, _IOC_SIZE(cmd))` matching `_IOC_DIR(cmd)` before driver dispatch.

## Architecture

```
pub mod ioc {
    pub const NRBITS:   u32 = 8;
    pub const TYPEBITS: u32 = 8;
    pub const SIZEBITS: u32 = 14;
    pub const DIRBITS:  u32 = 2;

    pub const NRMASK:   u32 = (1 << NRBITS)   - 1;   // 0xFF
    pub const TYPEMASK: u32 = (1 << TYPEBITS) - 1;   // 0xFF
    pub const SIZEMASK: u32 = (1 << SIZEBITS) - 1;   // 0x3FFF
    pub const DIRMASK:  u32 = (1 << DIRBITS)  - 1;   // 0x03

    pub const NRSHIFT:   u32 = 0;
    pub const TYPESHIFT: u32 = NRSHIFT   + NRBITS;    // 8
    pub const SIZESHIFT: u32 = TYPESHIFT + TYPEBITS;  // 16
    pub const DIRSHIFT:  u32 = SIZESHIFT + SIZEBITS;  // 30
}

#[repr(u8)]
pub enum IocDir {
    None  = 0,
    Write = 1,
    Read  = 2,
}

pub const IOC_IN:         u32 = (IocDir::Write as u32) << ioc::DIRSHIFT;  // 0x40000000
pub const IOC_OUT:        u32 = (IocDir::Read  as u32) << ioc::DIRSHIFT;  // 0x80000000
pub const IOC_INOUT:      u32 = ((IocDir::Write as u32) | (IocDir::Read as u32)) << ioc::DIRSHIFT;  // 0xC0000000
pub const IOCSIZE_MASK:   u32 = ioc::SIZEMASK << ioc::SIZESHIFT;  // 0x3FFF0000
pub const IOCSIZE_SHIFT:  u32 = ioc::SIZESHIFT;                   // 16

#[inline]
pub const fn ioc_pack(dir: u32, ty: u32, nr: u32, size: u32) -> u32 {
    (dir  << ioc::DIRSHIFT)  |
    (size << ioc::SIZESHIFT) |
    (ty   << ioc::TYPESHIFT) |
    (nr   << ioc::NRSHIFT)
}

#[inline] pub const fn ioc_dir (cmd: u32) -> u32 { (cmd >> ioc::DIRSHIFT)  & ioc::DIRMASK }
#[inline] pub const fn ioc_type(cmd: u32) -> u32 { (cmd >> ioc::TYPESHIFT) & ioc::TYPEMASK }
#[inline] pub const fn ioc_nr  (cmd: u32) -> u32 { (cmd >> ioc::NRSHIFT)   & ioc::NRMASK }
#[inline] pub const fn ioc_size(cmd: u32) -> u32 { (cmd >> ioc::SIZESHIFT) & ioc::SIZEMASK }

#[macro_export]
macro_rules! _IO   { ($t:expr, $n:expr)             => { ioc_pack(0, $t, $n, 0) }; }
#[macro_export]
macro_rules! _IOR  { ($t:expr, $n:expr, $arg:ty)    => { ioc_pack(2, $t, $n, core::mem::size_of::<$arg>() as u32) }; }
#[macro_export]
macro_rules! _IOW  { ($t:expr, $n:expr, $arg:ty)    => { ioc_pack(1, $t, $n, core::mem::size_of::<$arg>() as u32) }; }
#[macro_export]
macro_rules! _IOWR { ($t:expr, $n:expr, $arg:ty)    => { ioc_pack(3, $t, $n, core::mem::size_of::<$arg>() as u32) }; }
```

`Vfs::do_ioctl(file: &File, cmd: u32, arg: usize) -> Result<i64>`:
1. /* Decode */
2. let dir  = ioc_dir(cmd).
3. let size = ioc_size(cmd).
4. /* Structural bound */
5. if size > ioc::SIZEMASK: return Err(EINVAL).
6. /* Cap-gate per-driver magic (security_file_ioctl) */
7. security_file_ioctl(file, cmd, arg)?.
8. /* Access-check user buffer per direction */
9. if dir != 0 && size != 0:
   - let prot = match dir {
        IocDir::Write as u32 => Prot::Read,    /* kernel reads user */
        IocDir::Read  as u32 => Prot::Write,   /* kernel writes user */
        3u32                 => Prot::ReadWrite,
        _                    => unreachable!(),
     };
   - access_ok(prot, arg as *const u8, size as usize)?.
10. /* Generic fast-paths (FIONREAD, FIONBIO, FIOCLEX, FIONCLEX, FICLONE, FIDEDUPERANGE, ...) */
11. if let Some(rc) = Self::generic_ioctl(file, cmd, arg)? { return Ok(rc); }.
12. /* Per-driver */
13. let op = file.f_op.unlocked_ioctl.ok_or(Err(ENOTTY))?.
14. return op(file, cmd, arg).

`Vfs::compat_ioctl(file: &File, cmd: u32, arg: usize) -> Result<i64>`:
1. /* For 32-on-64: cmd may have been encoded with 32-bit sizeof */
2. /* Per-driver compat_ioctl supplies translation; otherwise fall back to unlocked_ioctl */
3. if let Some(op) = file.f_op.compat_ioctl { return op(file, cmd, arg); }
4. /* Generic compat fallbacks for FIONREAD etc */
5. /* Otherwise -ENOIOCTLCMD */
6. return Err(ENOIOCTLCMD).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `field_widths_sum_to_32` | INVARIANT | `NRBITS + TYPEBITS + SIZEBITS + DIRBITS == 32`. |
| `masks_are_derived` | INVARIANT | per-field: `MASK == (1 << BITS) - 1`. |
| `shifts_are_cumulative` | INVARIANT | `TYPESHIFT == NRSHIFT + NRBITS`, etc. |
| `dir_values_canonical` | INVARIANT | `NONE == 0 ∧ WRITE == 1 ∧ READ == 2`. |
| `pack_decode_roundtrip` | INVARIANT | per-cmd: `ioc_pack(ioc_dir(c), ioc_type(c), ioc_nr(c), ioc_size(c)) == c` when c is well-formed. |
| `io_size_zero` | INVARIANT | `_IO(t, n)` has `ioc_size == 0` and `ioc_dir == _IOC_NONE`. |
| `ior_direction_read` | INVARIANT | `_IOR(t, n, T)` has `ioc_dir == _IOC_READ` and `ioc_size == sizeof::<T>()`. |
| `iow_direction_write` | INVARIANT | `_IOW(t, n, T)` has `ioc_dir == _IOC_WRITE` and `ioc_size == sizeof::<T>()`. |
| `iowr_direction_both` | INVARIANT | `_IOWR(t, n, T)` has `ioc_dir == _IOC_READ | _IOC_WRITE` and `ioc_size == sizeof::<T>()`. |
| `size_capped_at_sizemask` | INVARIANT | per-do_ioctl: `ioc_size(cmd) > SIZEMASK ⟹ EINVAL`. |
| `access_ok_matches_dir` | INVARIANT | per-do_ioctl: dir & WRITE ⟹ access_ok(read); dir & READ ⟹ access_ok(write). |
| `ioc_in_alias` | INVARIANT | `IOC_IN == (_IOC_WRITE << DIRSHIFT) == 0x40000000`. |
| `ioc_out_alias` | INVARIANT | `IOC_OUT == (_IOC_READ << DIRSHIFT) == 0x80000000`. |
| `iocsize_mask_alias` | INVARIANT | `IOCSIZE_MASK == SIZEMASK << SIZESHIFT == 0x3FFF0000`. |

### Layer 2: TLA+

`uapi/headers/ioctl.tla`:
- Per-`_IOC` construction → per-driver decode → access-check → per-driver dispatch lifecycle.
- Properties:
  - `safety_pack_decode_inverse` — per-cmd: `(pack ∘ decode) == id` on the well-formed subset.
  - `safety_size_bounded` — per-decode: `ioc_size(cmd) ≤ 16383`.
  - `safety_access_ok_before_deref` — per-dispatch: every user-pointer deref is preceded by an `access_ok` matching the encoded direction.
  - `safety_magic_disjoint` — per-driver: the kernel-wide set of in-use TYPE bytes has the property that two drivers don't simultaneously register the same (TYPE, NR) pair.
  - `liveness_ioctl_returns` — per-`ioctl(2)`: eventually returns (success, ENOTTY, EINVAL, or driver-specific errno).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ioc_pack` post: result has dir/type/nr/size in correct bit-positions | `ioc_pack` |
| `ioc_dir`/`type`/`nr`/`size` post: returns the corresponding field | `ioc::{dir,type,nr,size}` |
| `_IO`/`_IOR`/`_IOW`/`_IOWR` post: direction and size bits match the variant | `_IO*` macros |
| `Vfs::do_ioctl` post: dir/size validated, access_ok performed, dispatch occurred | `Vfs::do_ioctl` |

### Layer 4: Verus/Creusot functional

`Per ioctl(2) → cmd decode (DIR/TYPE/NR/SIZE) → security_file_ioctl → access_ok(user_buf, size, dir-matching-prot) → file->f_op->unlocked_ioctl(file, cmd, arg) → result` semantic equivalence: per-`Documentation/driver-api/ioctl.rst`, per-`Documentation/userspace-api/ioctl/ioctl-number.rst`, per-`fs/ioctl.c` contract.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

ioctl encoding UAPI reinforcement:

- **`_IOC_SIZEMASK` (14 bits) caps arg-struct at 16383 bytes structurally** — defense against per-oversized-arg DoS and per-stack-allocation overflow in drivers that copy on-stack.
- **`access_ok` matches `_IOC_DIR`** — defense against per-driver direction-confusion (write-marked ioctl reading a user pointer attacker-controlled to point at kernel-space).
- **`security_file_ioctl` LSM hook before dispatch** — defense against per-driver missing-cap escalation.
- **Per-driver magic-byte uniqueness** documented in `ioctl-number.rst` — defense against per-collision dispatch confusion.
- **Generic fast-path `do_vfs_ioctl` handles FIONREAD/FIONBIO/FIOCLEX/FIONCLEX/FICLONE etc** before driver dispatch — defense against per-driver omitting these standard ops.
- **`compat_ioctl` size-translates for 32-on-64** — defense against per-32-bit-clobber stack-disclosure (CVE-2010-2240 class).
- **`_IOC_*_BAD` variants flagged in code review** — defense against per-type-confusion ioctl encoding.
- **Unknown cmd returns `-ENOTTY`, not `-EINVAL`** — defense against per-driver leaking-cmd-existence to unprivileged callers.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF/USERCOPY** on every `copy_from_user`/`copy_to_user` driven by `_IOC_SIZE(cmd)` — defense against per-kernel-pointer dereference where an attacker-controlled cmd encoded an oversized or wrong-direction size. UDEREF specifically catches kernel-pointer-as-user-pointer confusion in `compat_ioctl` translation layers.
- **PAX_RANDKSTACK on syscall entry** — randomize kernel-stack offset on every `ioctl(2)` entry. Many drivers `kmalloc`/`copy_from_user` argument structs whose size is encoded in `_IOC_SIZE`; the per-call randomization defeats heap-layout grooming for adjacent-allocation disclosure.
- **Per-driver CAP gating on every state-changing ioctl** — `_IOC_DIR & _IOC_WRITE` ioctls require a documented capability check inside the driver's `unlocked_ioctl`. Grsec policy is default-deny on drivers that lack an explicit CAP check; an audit framework flags any driver registering an `unlocked_ioctl` without per-cmd cap analysis.
- **Magic-number (`_IOC_TYPE`) validation** at dispatch — drivers are required to switch on `_IOC_TYPE(cmd)` and reject mismatched type bytes (`-ENOTTY`). Defense against per-cross-driver-cmd-confusion where a cmd intended for `/dev/foo` is forwarded to `/dev/bar` via an fd-passing trick.
- **`_IOC_SIZE` cross-checked against driver's expected size** — drivers MUST compare `_IOC_SIZE(cmd)` against the size of their argument struct and reject mismatches (`-EINVAL`), preventing per-stale-binary undefined-behavior (small-buffer reads, partial-buffer-writes). Old binaries get cleanly rejected rather than silently corrupting state.
- **`_IOC_DIR` enforced symmetrically** — `_IOC_READ` (kernel writes user) MUST `copy_to_user` (not just `put_user` of a partial result); `_IOC_WRITE` (kernel reads user) MUST `copy_from_user` upfront, not multiple sub-reads. Defense against per-TOCTOU on user buffer between sub-reads.
- **`compat_ioctl` 32→64 size translation strictly typed** — drivers that don't supply a `compat_ioctl` get the generic translator, which only handles the canonical layout. Defense against per-32-bit-32-bit-padding info-leak (CVE-2010-2240, CVE-2017-7187 classes).
- **`security_file_ioctl` LSM hook** before driver dispatch — SELinux/AppArmor policy decides per-(domain, dev_class, cmd) allow/deny. Defense against per-confined-process privilege escalation via uncommon ioctl numbers.
- **Unknown cmd returns `-ENOTTY`** — never `-EINVAL` and never `-EFAULT`. Defense against per-cmd-existence disclosure (timing-side-channel).
- **GRKERNSEC_DEVICE_SIDEFFECTS / CHROOT_NICE** — chroot'd / namespaced processes have a restricted view of ioctlable devices; even with `CAP_SYS_ADMIN` in a userns, ioctls on host devices (`/dev/sd*`, `/dev/nvme*`, `/dev/kmem`) require initial-userns CAP.
- **Generic `ioctl_number` table audit** — every assigned `_IOC_TYPE` byte is registered with the kernel-build at compile time; collisions are CI-detected. Defense against per-out-of-tree-driver smashing in-tree assignments.
- **`_IOC_TYPECHECK` syntactic typecheck is mandatory in userspace headers** — Rookery's UAPI headers always declare ioctl numbers via `_IOR`/`_IOW`/`_IOWR` (typed) rather than `_BAD` variants except for documented historical-compat exceptions. Defense against per-sizeof-arg-mismatch causing wrong-size kernel-side copy.
- **Per-uid rate limit on `ioctl(2)` failures** — bounded to ~50 failures/s/uid under grsec policy; defense against per-fuzz-burst probing of driver entry points.

## Open Questions

(none at this Tier-5 level — per-driver Tier-3 docs cover per-ioctl semantics for their drivers)

## Out of Scope

- `<asm/ioctl.h>` per-arch overrides (covered in per-arch UAPI docs).
- Per-driver ioctl number assignments (covered in `Documentation/userspace-api/ioctl/ioctl-number.rst` and per-driver Tier-3 docs).
- `include/uapi/linux/fs.h` `FS_IOC_*` generic-filesystem ioctls (`FICLONE`, `FIDEDUPERANGE`, `FIFREEZE`, etc.) (covered in `uapi/headers/fs.md`).
- `include/uapi/linux/kvm.h` (covered in `uapi/headers/kvm.md`).
- `include/uapi/linux/v4l2-*` (covered separately).
- `fs/ioctl.c` implementation (covered in `fs/ioctl.md` Tier-3).
- Implementation code.
