# Tier-3: drivers/char/mem.c — /dev/{mem,port,zero,null,full,random,urandom,kmsg} core chardev

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/char/mem.c
-->

## Summary

`drivers/char/mem.c` is the core character-device dispatcher for the historical "memory devices" exposed under MEM_MAJOR=1: `/dev/mem` (minor 1), `/dev/null` (3), `/dev/port` (4), `/dev/zero` (5), `/dev/full` (7), `/dev/random` (8), `/dev/urandom` (9), and `/dev/kmsg` (11). The file owns the chrdev registration (`register_chrdev(MEM_MAJOR, "mem", &memory_fops)`), the per-minor `file_operations` switch (`memory_open` → `dev->fops`), the per-device `file_operations` tables (`mem_fops`, `null_fops`, `port_fops`, `zero_fops`, `full_fops`), and the actual read/write/mmap paths for the physical-memory devices (`read_mem`, `write_mem`, `mmap_mem_prepare`, `read_port`, `write_port`). The /dev/{random,urandom,kmsg} entries delegate to file_operations exported by `drivers/char/random.c` and `kernel/printk/printk.c` respectively.

This Tier-3 focuses on the security-sensitive parts: `/dev/mem` and `/dev/port` — the kernel's most powerful diagnostic interfaces. `/dev/mem` reads/writes raw physical memory; `/dev/port` reads/writes raw x86 I/O ports. Both are gated by CAP_SYS_RAWIO, by `LOCKDOWN_DEV_MEM`, and (for /dev/mem) by `CONFIG_STRICT_DEVMEM` + `devmem_is_allowed(pfn)`. The /dev/{null,zero,full} entries are documented for completeness but have negligible attack surface.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `memory_fops` (open=`memory_open`, llseek=`noop_llseek`) | top-level dispatch chrdev | `drivers::mem::DispatcherFops` |
| `memory_open(inode, filp)` | minor → per-device fops swap | `Dispatcher::open` |
| `devlist[]` (per-minor name + fops + fmode + mode) | per-minor table | `Dispatcher::DEV_TABLE` |
| `mem_fops` (read/write/mmap/llseek/open) | /dev/mem | `DevMem::FileOps` |
| `port_fops` (read/write/llseek/open) | /dev/port | `DevPort::FileOps` |
| `null_fops`, `zero_fops`, `full_fops` | /dev/null, /dev/zero, /dev/full | `DevNull` / `DevZero` / `DevFull` |
| `open_port(inode, filp)` | CAP_SYS_RAWIO + LOCKDOWN_DEV_MEM gate | `DevMem::open` / `DevPort::open` |
| `read_mem` / `write_mem` | per-page copy through `xlate_dev_mem_ptr` | `DevMem::read` / `write` |
| `read_port` / `write_port` | byte-at-a-time `inb`/`outb` over `__user *` | `DevPort::read` / `write` |
| `mmap_mem_prepare` / `mmap_mem_ops` | per-VMA mmap of physical pages | `DevMem::mmap_prepare` |
| `valid_phys_addr_range(addr, count)` / `valid_mmap_phys_addr_range(pfn, size)` | arch hook bounds check | `DevMem::valid_phys` / `valid_mmap` |
| `range_is_allowed(pfn, size)` / `page_is_allowed(pfn)` (`CONFIG_STRICT_DEVMEM`) | per-pfn allowlist | `DevMem::range_allowed` |
| `devmem_is_allowed(pfn)` (arch hook in `arch/x86/mm/init.c` et al.) | RAM-page filter | `arch::DevMemPolicy` |
| `phys_mem_access_prot(file, pfn, size, vma_prot)` / `phys_mem_access_prot_allowed(...)` | cache attr arbitration | `DevMem::access_prot` |
| `should_stop_iteration()` | per-page cooperative yield | `DevMem::cooperative_yield` |
| `security_locked_down(LOCKDOWN_DEV_MEM)` | lockdown gate (called from `open_port`) | `DevMem::lockdown_check` |
| `MEM_MAJOR` (1) | chrdev major | `DevMem::MAJOR` |
| `DEVMEM_MINOR` (1), `MEMPORT_MINOR` (4) | per-minor constants | `DevMem::Minors` |

## Compatibility contract

REQ-1: `register_chrdev(MEM_MAJOR=1, "mem", &memory_fops)` claims all 256 minors of major 1 at boot; per-minor dispatch via `memory_open`.

REQ-2: `memory_open` consults `devlist[minor]` and on match copies `dev->fops` into `filp->f_op` + applies `dev->fmode` (e.g. `FMODE_NOWAIT` for null/zero/random/urandom), then chains to `dev->fops->open` if present.

REQ-3: `/dev/mem` open requires CAP_SYS_RAWIO (`if (!capable(CAP_SYS_RAWIO)) return -EPERM`).

REQ-4: `/dev/mem` open also calls `security_locked_down(LOCKDOWN_DEV_MEM)`; nonzero return → `-EPERM`. The same gate guards `/dev/port` since both share `open_port`.

REQ-5: `/dev/mem` read/write/mmap each consult `valid_phys_addr_range(addr, count)` (arch-defined; rejects addresses outside `0..max_pfn << PAGE_SHIFT` and per-arch reserved ranges).

REQ-6: With `CONFIG_STRICT_DEVMEM=y` (default), per-page `page_is_allowed(pfn) → devmem_is_allowed(pfn)` rejects RAM pages that are not in the architecture's "may be exposed" list (typically: ACPI tables, BIOS reserved, PCI MMIO). With `CONFIG_IO_STRICT_DEVMEM=y` additionally, ranges claimed by drivers via `request_mem_region` are refused.

REQ-7: `/dev/port` reads/writes proceed one byte at a time via `inb`/`outb` (or `__inb`/`__outb` per-arch); position is the I/O port number. Lockdown + CAP_SYS_RAWIO gating shared with /dev/mem.

REQ-8: `mmap` of /dev/mem populates the VMA with `remap_pfn_range`; cache attributes derived via `phys_mem_access_prot(file, pfn, size, vma_prot)` with arch hook `phys_mem_access_prot_allowed`.

REQ-9: `/dev/null` write returns `count` and discards; read returns 0 (EOF). Splice supported via `pipe_to_null` and `splice_write_null`. io_uring cmd noop via `uring_cmd_null`.

REQ-10: `/dev/zero` read fills the iter with zero; mmap with `MAP_PRIVATE` succeeds and returns an anonymous COW mapping (`mmap_zero_private_success` → `mmap_zero_prepare`); mmap with `MAP_SHARED` rejected (anonymous shared mapping has no backing).

REQ-11: `/dev/full` write returns `-ENOSPC`; read fills with zero (same as /dev/zero).

REQ-12: `noop_llseek` permits arbitrary seeks on devices where position is meaningful (mem/port); `memory_lseek` clamps within MAX_LFS_FILESIZE.

REQ-13: `should_stop_iteration()` checks `need_resched()`/`signal_pending()` per page; long reads of /dev/mem cooperatively yield.

REQ-14: `mem_devnode(dev, mode)` sets default mode `0666` for null/zero/full/random/urandom; /dev/mem and /dev/port default `0600`.

## Acceptance Criteria

- [ ] AC-1: `open("/dev/mem", O_RDWR)` as root with `CAP_SYS_RAWIO` succeeds when lockdown is off; same call returns `-EPERM` under `kernel_lockdown=integrity` or higher.
- [ ] AC-2: `read("/dev/mem")` at an address backed by RAM (with `CONFIG_STRICT_DEVMEM=y`) returns `-EPERM`; at an ACPI table page it returns data.
- [ ] AC-3: `mmap("/dev/mem", PROT_READ, MAP_SHARED, fd, PCI_MMIO_BAR)` succeeds and maps the BAR; same mmap of a RAM pfn refused.
- [ ] AC-4: `read("/dev/port", buf, 1)` at offset 0x70 (RTC index) returns the RTC index value as root; refused without CAP_SYS_RAWIO.
- [ ] AC-5: `dd if=/dev/zero of=/tmp/x bs=1M count=1` writes 1 MiB of zero.
- [ ] AC-6: `dd if=/dev/null of=/tmp/x bs=1M count=1` produces an empty file.
- [ ] AC-7: `write("/dev/full", buf, n)` returns `-ENOSPC`.
- [ ] AC-8: A long `read("/dev/mem")` cooperatively yields without triggering soft-lockup.
- [ ] AC-9: Lockdown gate: `echo integrity > /sys/kernel/security/lockdown`; subsequent `open("/dev/mem")` returns `-EPERM`.

## Architecture

Per-device dispatch:

```
struct DevEntry {
  name: &'static str,
  fops: &'static FileOperations,
  fmode: FMode,
  mode: u16,
}

const DEV_TABLE: [Option<DevEntry>; 12] = [
  // minor 1
  Some(DevEntry { name: "mem",     fops: &MEM_FOPS,     fmode: FMode::empty(),    mode: 0 }),
  // minor 3
  Some(DevEntry { name: "null",    fops: &NULL_FOPS,    fmode: FMode::NOWAIT,     mode: 0o666 }),
  // minor 4
  Some(DevEntry { name: "port",    fops: &PORT_FOPS,    fmode: FMode::empty(),    mode: 0 }),
  // minor 5
  Some(DevEntry { name: "zero",    fops: &ZERO_FOPS,    fmode: FMode::NOWAIT,     mode: 0o666 }),
  // minor 7
  Some(DevEntry { name: "full",    fops: &FULL_FOPS,    fmode: FMode::empty(),    mode: 0o666 }),
  // minor 8
  Some(DevEntry { name: "random",  fops: &RANDOM_FOPS,  fmode: FMode::NOWAIT,     mode: 0o666 }),
  // minor 9
  Some(DevEntry { name: "urandom", fops: &URANDOM_FOPS, fmode: FMode::NOWAIT,     mode: 0o666 }),
  // minor 11
  Some(DevEntry { name: "kmsg",    fops: &KMSG_FOPS,    fmode: FMode::empty(),    mode: 0o644 }),
  ...
];
```

Dispatcher open path:
1. `memory_open(inode, filp)` reads `iminor(inode)`.
2. Looks up `DEV_TABLE[minor]` → `dev`.
3. `filp->f_op = dev->fops; filp->f_mode |= dev->fmode`.
4. If `dev->fops->open` is non-null, chains into it. For `/dev/mem`, `/dev/port` this is `open_port` which enforces CAP_SYS_RAWIO + LOCKDOWN_DEV_MEM.

`open_port`:
```
static int open_port(struct inode *inode, struct file *filp)
{
    int rc;
    if (!capable(CAP_SYS_RAWIO))
        return -EPERM;
    rc = security_locked_down(LOCKDOWN_DEV_MEM);
    if (rc)
        return rc;
    ...
}
```

`/dev/mem` read path `read_mem(file, buf, count, ppos)`:
1. While `count` > 0:
   - Compute `sz = size_inside_page(*ppos, count)`.
   - `pfn = *ppos >> PAGE_SHIFT`.
   - `valid_phys_addr_range(*ppos, sz)` — refuse if out of range.
   - `page_is_allowed(pfn)` (under CONFIG_STRICT_DEVMEM) — refuse if RAM not on allowlist.
   - `ptr = xlate_dev_mem_ptr(*ppos)` — arch hook (typically `__va` for low memory, `ioremap` for high MMIO).
   - `copy_to_user(buf, ptr, sz)`.
   - `unxlate_dev_mem_ptr` — release ioremap if any.
   - Advance `*ppos += sz; count -= sz`.
   - `should_stop_iteration()` cooperative yield.

`/dev/mem` write path `write_mem` mirrors with `copy_from_user`. Per-arch hook `xlate_dev_mem_ptr` (e.g. x86 `arch/x86/mm/iomap_32.c` for 32-bit; `__va` for 64-bit lowmem) decides whether the page is reachable by direct map or needs `memremap`.

`/dev/mem` mmap path `mmap_mem_prepare(desc)`:
1. `valid_mmap_phys_addr_range(desc->pgoff, size)` — refuse if outside arch policy.
2. `range_is_allowed(desc->pgoff, size)` — per-page allowlist.
3. `desc->vm_page_prot = phys_mem_access_prot(file, desc->pgoff, size, desc->vm_page_prot)`.
4. `mmap_filter_error` translates `EAGAIN`-style retries into `-EBUSY`.
5. `remap_pfn_range_prepare` populates the VMA; `vma->vm_ops = &mmap_mem_ops` so faults beyond initial map are denied.

`/dev/port` read path `read_port(file, buf, count, ppos)`:
1. `i = *ppos`; while `count` > 0 and `i < 65536`: `c = inb(i); put_user(c, buf); i++; buf++; count--`.
2. `*ppos = i`.

`/dev/zero` read iter `read_iter_zero(iocb, iter)`: bulk-copies the static `ZERO_PAGE(0)` into the iter chunked by `PAGE_SIZE`, with `iov_iter_zero` underneath.

`/dev/zero` mmap `mmap_zero_prepare(desc)`: refuses `MAP_SHARED` (returns `-EINVAL`); for `MAP_PRIVATE` it installs an anonymous private VMA backed by `ZERO_PAGE` until first write (COW).

## Hardening

- `open_port` gates `/dev/mem` and `/dev/port` on both CAP_SYS_RAWIO and `security_locked_down(LOCKDOWN_DEV_MEM)` so capability alone is not sufficient when the kernel is in confidentiality/integrity lockdown.
- `valid_phys_addr_range` is an arch hook; x86 enforces `addr < (max_pfn << PAGE_SHIFT)` and rejects addresses inside the kernel image when STRICT_DEVMEM is on.
- `CONFIG_STRICT_DEVMEM=y` is the upstream default; flip it off only on debug builds. `CONFIG_IO_STRICT_DEVMEM=y` adds enforcement that `request_mem_region`-owned ranges are off-limits even to root.
- `/dev/mem` mmap respects `phys_mem_access_prot_allowed`: arch hooks can refuse particular page-attribute combinations (e.g. cacheable mapping of MMIO) to prevent CPU machine checks.
- `/dev/port` operations are 1-byte at a time and cannot violate page-level access controls (no mmap path).
- `should_stop_iteration()` per-page yield prevents an unbounded `read("/dev/mem", buf, GIGABYTES)` from starving the CPU.
- `mem_class.devnode` callback sets default permissions `0600` for mem and port; udev rules cannot loosen without explicit operator change.
- `kmsg_fops` (minor 11) is delegated entirely to `printk` core and inherits its CAP_SYSLOG and lockdown gating.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — every `copy_to_user`/`copy_from_user` in `read_mem`/`write_mem`/`read_port`/`write_port` bounded by `size_inside_page` and explicit `count` decrement; kbuffer slabs (when used for intermediate copy) are whitelisted.
- **PAX_KERNEXEC** — `memory_fops`, `mem_fops`, `port_fops`, `null_fops`, `zero_fops`, `full_fops`, `mmap_mem_ops`, and the `devlist[]` table live in `__ro_after_init` text/data; no runtime patching.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `memory_open`, `read_mem`, `write_mem`, `read_port`, `write_port`, and `mmap_mem_prepare` entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on the per-open `f_count` and on any per-VMA refs taken by `mmap_mem_prepare`; overflow trap defeats UAF on race between close and ongoing read.
- **PAX_MEMORY_SANITIZE** — zero-on-free for any intermediate kmalloc buffer used during /dev/mem chunked transfers so prior contents cannot leak across operations.
- **PAX_UDEREF** — SMAP/PAN enforced on every `copy_to/from_user`; arch SMAP/PAN bracket inside `read_mem`/`write_mem` covers the inner copy.
- **PAX_RAP / kCFI** — `memory_fops`, per-device `file_operations`, and `mmap_mem_ops` are typed indirect calls; substitution refused.
- **GRKERNSEC_HIDESYM** — suppress `%p` for any `xlate_dev_mem_ptr` translation result in trace/debug paths; gate behind CAP_SYSLOG.
- **GRKERNSEC_DMESG** — restrict /dev/mem deny banners (range refused, RAM-page refused) to CAP_SYSLOG so attackers cannot probe the allowlist via dmesg.
- **/dev/mem CAP_SYS_RAWIO + LOCKDOWN_KCORE + LOCKDOWN_DEV_MEM** — open refused unless caller has CAP_SYS_RAWIO and `security_locked_down(LOCKDOWN_DEV_MEM)` returns 0; integrity/confidentiality lockdown removes /dev/mem entirely.
- **Range filter `devmem_is_allowed`** — every read/write/mmap of /dev/mem traverses `page_is_allowed(pfn) → devmem_is_allowed(pfn)`; RAM pages refused, MMIO + ACPI ranges permitted (only when not lockdown-blocked).
- **STRICT_DEVMEM enforced** — `CONFIG_STRICT_DEVMEM=y` and `CONFIG_IO_STRICT_DEVMEM=y` baked in; `/dev/mem` cannot read kernel `.text`, page tables, or driver-claimed MMIO regions.
- **/dev/port range allowlist** — I/O ports below 0x100 (legacy fixed devices) require CAP_SYS_RAWIO; integrity lockdown blocks the entire device. Optional per-port allowlist (CMOS index/data, serial UARTs only) gated by Rookery policy.
- **GRKERNSEC_KMEM equivalent** — /dev/kmem is not built (upstream removed it); `/dev/mem` refuses every RAM pfn under STRICT_DEVMEM, achieving the same posture as the grsec `DENY_USB`/kmem refusal.
- **/dev/port write audit** — every successful `write("/dev/port")` emits a rate-limited audit record (port, byte, caller) so an attacker cannot silently program legacy chipset registers via root.

Rationale: `/dev/mem` and `/dev/port` are the most dangerous chrdevs in the kernel — they are the abstraction over "I am root, give me the raw machine." Two gates protect the kernel: CAP_SYS_RAWIO (a privileged process opens) and LOCKDOWN_DEV_MEM (no process, not even root, opens when integrity/confidentiality lockdown is active). STRICT_DEVMEM enforces a per-page allowlist so even authorized opens cannot read kernel RAM, page tables, or driver MMIO. The grsec posture mandates all three are unconditionally enabled, audit records emit on every /dev/port mutation, and the chardev mode defaults to `0600` so udev rules cannot weaken it.

## Open Questions

- Should the lockdown gate also apply to /dev/zero / /dev/null mmap paths (they do not expose memory but participate in some sandbox escape patterns by being a baseline mmap target)? Current policy: no.
- Whether to ship an additional sysctl `kernel.devmem_allowlist` that lets an operator narrow `devmem_is_allowed` further (e.g. permit only specific PCI BAR ranges).

## Out of Scope

- `/dev/random` and `/dev/urandom` internals (covered in `drivers/char/random.md`)
- `/dev/kmsg` internals (covered in `kernel/printk/printk.md` future Tier-3)
- Arch hook implementations (`xlate_dev_mem_ptr`, `phys_mem_access_prot_allowed`) per arch
- `/dev/kmem` (removed upstream)
- Implementation code
