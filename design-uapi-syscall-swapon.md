---
title: "Tier-5 syscall: swapon(2) — syscall 167"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`swapon(2)` activates a swap area — either a block device or a regular file — adding it to the kernel's swap-allocation pool. The kernel reads the swap header (page-0 magic `SWAPSPACE2`), validates the layout, builds the per-slot bitmap and extent table, then inserts the new `swap_info_struct` into `swap_avail_heads` (per-node priority list). After `swapon`, kswapd / direct-reclaim may write anonymous pages here.

Critical for: system memory expansion, hibernation-resume, container memory-overcommit policies, zswap backing.

### Acceptance Criteria

- [ ] AC-1: `swapon("/dev/sdb1", 0)` on mkswap'd partition: returns 0; `/proc/swaps` lists it.
- [ ] AC-2: Non-root caller: `-EPERM`.
- [ ] AC-3: `path` invalid: `-ENOENT`.
- [ ] AC-4: Path to regular file on ext4 with header: success.
- [ ] AC-5: Path to file on tmpfs: `-EOPNOTSUPP`.
- [ ] AC-6: Header signature mismatch: `-EINVAL`.
- [ ] AC-7: Re-swapon same path: `-EBUSY`.
- [ ] AC-8: 33rd swapon when max=32: `-ENFILE`.
- [ ] AC-9: `SWAP_FLAG_PREFER | 100`: priority 100 inserted.
- [ ] AC-10: `SWAP_FLAG_DISCARD`: TRIM observed on slot release.
- [ ] AC-11: Block device read-only: `-EROFS`.
- [ ] AC-12: Concurrent `swapon` of two distinct paths: both succeed.

### Architecture

```rust
#[syscall(nr = 167, abi = "sysv")]
pub fn sys_swapon(path: UserPtr<u8>, swapflags: i32) -> isize {
    Swap::do_swapon(path, swapflags)
}
```

`Swap::do_swapon(path_uptr, swapflags) -> isize`:
1. if !cap(CAP_SYS_ADMIN, init_user_ns()): return -EPERM;
2. let path = Fs::getname(path_uptr)?;
3. /* Validate flags */
4. if swapflags & !SWAP_FLAG_ALL_VALID != 0: return -EINVAL;
5. /* Reserve a swap_info slot */
6. let p = Swap::alloc_swap_info().ok_or(-ENFILE)?;
7. /* Open the path */
8. let file = Fs::filp_open(&path, O_RDWR | O_LARGEFILE | O_EXCL_BDEV)?;
9. /* Claim file as swap */
10. Swap::claim_swapfile(p, file)?;                  // EBUSY / EROFS / EINVAL
11. /* Read header */
12. let hdr = Swap::read_swap_header(p)?;            // EIO / EINVAL
13. /* Build extent table */
14. Swap::setup_swap_extents(p, &hdr)?;
15. /* Optional discard */
16. if swapflags & SWAP_FLAG_DISCARD != 0 {
17.   p.flags |= SWP_DISCARDABLE;
18.   if swapflags & SWAP_FLAG_DISCARD_ONCE != 0 {
19.     Swap::discard_swap(p)?;
20.   }
21. }
22. /* Set priority */
23. p.priority = if swapflags & SWAP_FLAG_PREFER != 0 {
24.   (swapflags & SWAP_FLAG_PRIO_MASK) as i16
25. } else {
26.   --next_auto_prio
27. };
28. /* Activate */
29. Swap::enable_swap_info(p);
30. Ok(0)

`Swap::read_swap_header(p) -> Result<SwapHeader>`:
1. let page = read_mapping_page(p.swap_file.mapping, 0)?;
2. let hdr: &SwapHeader = page.as_ref();
3. if &hdr.magic.sig != b"SWAPSPACE2": return Err(EINVAL);
4. if hdr.info.version != 1: return Err(EINVAL);
5. if hdr.info.last_page >= p.max_pages: return Err(EINVAL);
6. Ok(hdr.clone())

### Out of Scope

- Swap-slot allocation logic (Tier-3 `mm/swap_state.md`).
- Reclaim & writeback to swap (Tier-3 `mm/swap.md`, `mm/vmscan.md`).
- zswap interaction (Tier-3 `mm/zswap.md`).
- Implementation code.

### signature

```c
int swapon(const char *path, int swapflags);
```

```c
#define SWAP_FLAG_PREFER        0x8000   /* priority bits valid */
#define SWAP_FLAG_PRIO_MASK     0x7fff   /* 0..32767 */
#define SWAP_FLAG_PRIO_SHIFT    0
#define SWAP_FLAG_DISCARD       0x10000  /* enable TRIM/discard */
#define SWAP_FLAG_DISCARD_ONCE  0x20000  /* discard once at swapon */
#define SWAP_FLAG_DISCARD_PAGES 0x40000  /* discard per freed cluster */
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | Path to swap area (block device or regular file). NUL-terminated. |
| `swapflags` | `int` | in | `SWAP_FLAG_*` ORed flags; lower 15 bits hold priority if `SWAP_FLAG_PREFER` set. |

### return value

| Value | Meaning |
|---|---|
| `0` | Swap area online. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EPERM` | Caller lacks `CAP_SYS_ADMIN`. |
| `EFAULT` | `path` user pointer faults. |
| `ENOENT` | `path` does not exist. |
| `EINVAL` | Not a regular file or block device; header invalid; bad flags. |
| `EBUSY` | Path already in use as swap area. |
| `ENOMEM` | Per-slot allocation failed. |
| `EIO` | I/O error reading header. |
| `ENFILE` | Reached `MAX_SWAPFILES` (typically 32). |
| `EOPNOTSUPP` | File on filesystem that lacks `swap_activate`. |
| `EROFS` | Swap area would be read-only. |

### abi surface

```text
__NR_swapon  (x86_64)  = 167
__NR_swapon  (arm64)   = 224
__NR_swapon  (riscv)   = 224
__NR_swapon  (i386)    = 87

/* Swap header (first page): union swap_header { info: { ..., version=1, sig="SWAPSPACE2" }} */
```

### compatibility contract

REQ-1: Syscall number is **167** on x86_64. ABI-stable.

REQ-2: Capability: `CAP_SYS_ADMIN` in init userns (root namespace); user-namespaces cannot enable swap.

REQ-3: `path` resolved via standard `namei` (follows symlinks, honors mount namespace).

REQ-4: Path must point to:
- block device with `mkswap` header; or
- regular file with `mkswap` header AND the fs supplies `f_op->swap_activate`.

REQ-5: Swap header layout: 1 page; final 10 bytes are signature `"SWAPSPACE2"`; preceding fields = version (must be 1), last page index, bad-page list count, sw_volume UUID, label.

REQ-6: Per-`swap_extent` table: built from FIBMAP / `swap_activate` (`bmap` for ext-style, direct for btrfs); each extent = (start_page, nr_pages, start_block).

REQ-7: Per-`swap_info_struct`: contains bitmap of slots, priority, flags, max, file, bdev, swap_extent list. Inserted into `swap_avail_heads[nid]` sorted by priority desc.

REQ-8: Priority: when `SWAP_FLAG_PREFER` set, uses `swapflags & SWAP_FLAG_PRIO_MASK`; else auto-decremented (newest gets next-lower).

REQ-9: `SWAP_FLAG_DISCARD`: enables ongoing per-cluster `blkdev_issue_discard` as slots are freed.

REQ-10: `SWAP_FLAG_DISCARD_ONCE`: discards entire device once at swapon (long blocking operation).

REQ-11: `SWAP_FLAG_DISCARD_PAGES`: discards on free even without device-discard advisory.

REQ-12: For block device: `bd_claim` taken — exclusive. For regular file: holds `S_SWAPFILE` flag on inode.

REQ-13: Hibernation interaction: `MAX_SWAPFILES_SHIFT` reserves type-id bits for hibernation/zswap; type id assignment fixed at swapon time.

REQ-14: After swapon, anonymous pages are eligible to be swapped here when reclaim scans LRU.

REQ-15: `swapon` blocks until extent build complete.

REQ-16: Per-file swap requires fs `swap_activate`; otherwise `-EOPNOTSUPP` (e.g., tmpfs cannot back swap).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_sys_admin_init_ns` | INVARIANT | non-init-userns root rejected. |
| `extent_table_total` | INVARIANT | sum of extent nr_pages == swap_info.pages. |
| `priority_inserted_sorted` | INVARIANT | swap_avail_heads sorted desc-by-priority. |
| `header_signature_validated` | INVARIANT | bad sig ⟹ EINVAL, no swap_info exposed. |

### Layer 2: TLA+

`mm/swapon.tla`:
- States: per-cap, per-getname, per-claim, per-header, per-extents, per-activate.
- Properties:
  - `safety_no_activate_on_bad_header`.
  - `safety_no_double_claim`.
  - `safety_priority_total_order`.
  - `liveness_swapon_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_swapon` post: success ⟹ p inserted in swap_avail_heads | `Swap::do_swapon` |
| `read_swap_header` post: returns header with valid sig | `Swap::read_swap_header` |
| `setup_swap_extents` post: extent_table covers [0, max) | `Swap::setup_swap_extents` |

### Layer 4: Verus / Creusot functional

Per-`swapon(2)` man-page semantic equivalence. util-linux `swapon` parity.

### hardening

(Inherits row-1 from `uapi/00-overview.md` § Hardening.)

`swapon(2)` reinforcement:

- **Per-CAP_SYS_ADMIN init-userns gate** — defense against per-userns swap-activation escape.
- **Per-header signature validation** — defense against per-malformed-swap header attack.
- **Per-extent table total check** — defense against per-overlap allocation corruption.
- **Per-file-system `swap_activate` mandatory** — defense against per-broken-fs slot mapping.
- **Per-bd_claim exclusive lock** — defense against per-concurrent-mount UAF.
- **Per-MAX_SWAPFILES enforced** — defense against per-resource-exhaustion DoS.

### grsecurity / pax surface

- **PaX UDEREF on `path` getname** — defense against per-user-pointer kernel-deref bug; SMAP forced.
- **CAP_SYS_ADMIN in init_user_ns mandatory** — grsec strictly forbids swap activation from non-init user namespaces, even with `CAP_SYS_ADMIN` in the child namespace. Closes the userns + swap = persistent-pivot attack surface.
- **GRKERNSEC_MEM block on /dev/mem-style raw swap** — direct activation of `/dev/mem`, `/dev/kmem`, or `/dev/port` as swap is refused independently of file_operations checks.
- **GRKERNSEC_CHROOT_NO_SWAP** — processes in a chroot cannot call `swapon`, even if they hold `CAP_SYS_ADMIN`.
- **PAX_USERCOPY_HARDEN on swap header copy** — header read into a whitelisted slab buffer.
- **PAX_REFCOUNT on swap_info_struct refcount** — defense against per-double-swapon UAF.
- **PaX KERNSEAL on `swap_info` array** — array sealed after init; runtime registration uses a separate path with PaX RAP checks.
- **GRKERNSEC_TPE on swap file paths** — TPE restricts which paths a non-root subject could ever name, blocking dangling-swap-handoff exploits.
- **Per-encrypted-swap requirement** — grsec config can mandate that any swap activation must be on a dm-crypt device; non-encrypted paths refused.
- **GRKERNSEC_AUDIT_SWAPON** — every swapon/swapoff emits an audit record (path, prio, flags).

