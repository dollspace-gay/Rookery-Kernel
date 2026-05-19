# Tier-3: init/initramfs.c — initramfs cpio unpack + decompressor

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: init/00-overview.md
upstream-paths:
  - init/initramfs.c (~784 lines)
  - init/initramfs_internal.h
  - init/noinitramfs.c
  - include/linux/initrd.h
  - include/linux/decompress/generic.h
  - lib/decompress_inflate.c / _bunzip2.c / _unlzma.c / _unxz.c / _unlzo.c / _unlz4.c / _unzstd.c
-->

## Summary

`init/initramfs.c` consumes the kernel image's built-in initramfs (`__initramfs_start`/`__initramfs_size`, populated by `usr/`) and the optional bootloader-supplied initrd (`initrd_start`/`initrd_end`) and unpacks both into the in-memory rootfs (a tmpfs-backed `ramfs` instance registered by `vfs_caches_init`). Per-`populate_rootfs` is a `rootfs_initcall` that schedules an async-domain job (`do_populate_rootfs`) which calls `unpack_to_rootfs` first on the built-in then on the bootloader's blob. Per-`unpack_to_rootfs` detects each segment's compression (`decompress_method`), invokes the matching decompressor (gz / bz2 / lzma / xz / lzo / lz4 / zstd / none) with a `flush_buffer` callback that drives an 8-state cpio FSM (`Start/Collect/GotHeader/SkipIt/GotName/CopyFile/GotSymlink/Reset`). Per-cpio newc (`070701`) or newc-crc (`070702`) header (110 bytes) is parsed by `parse_header` into per-entry `ino/mode/uid/gid/nlink/mtime/body_len/major/minor/rdev/name_len/hdr_csum`. Per-hardlink hash (`find_link`/`free_hash`) detects shared inodes across the same `(major,minor,ino,S_IFMT)`. Per-`TRAILER!!!` marks end-of-archive. Per-`free_initrd_mem` releases bootloader-allocated initrd RAM (optionally retained via `retain_initrd` cmdline, exposed at `/sys/firmware/initrd`). Critical for: surviving every distro's initramfs encoding, byte-equivalent FS state to upstream.

This Tier-3 covers `init/initramfs.c` (~784 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `populate_rootfs()` | per-`rootfs_initcall` entry | `Initramfs::populate_rootfs` |
| `do_populate_rootfs(unused, cookie)` | per-async-domain worker | `Initramfs::do_populate_rootfs` |
| `wait_for_initramfs()` | per-PID-1 sync | `Initramfs::wait_for_initramfs` |
| `unpack_to_rootfs(buf, len)` | per-blob driver | `Initramfs::unpack_to_rootfs` |
| `parse_header(s)` | per-cpio newc parse | `Initramfs::parse_header` |
| `write_buffer(buf, len)` | per-FSM-tick | `Initramfs::write_buffer` |
| `flush_buffer(bufv, len)` | per-decompressor callback | `Initramfs::flush_buffer` |
| `read_into(buf, size, next)` | per-FSM-stage | `Initramfs::read_into` |
| `eat(n)` | per-byte-advance | `Initramfs::eat` |
| `do_start` / `do_collect` / `do_header` / `do_skip` / `do_name` / `do_copy` / `do_symlink` / `do_reset` | per-state-handler | `Initramfs::do_*` |
| `actions[]` | per-state-dispatch table | `Initramfs::actions` |
| `find_link(major, minor, ino, mode, name)` | per-hardlink-lookup | `Initramfs::find_link` |
| `free_hash()` | per-hardlink-table-free | `Initramfs::free_hash` |
| `maybe_link()` | per-hardlink-emit | `Initramfs::maybe_link` |
| `clean_path(path, fmode)` | per-overwrite-strip | `Initramfs::clean_path` |
| `xwrite(file, p, count, pos)` | per-`kernel_write` w/ EINTR loop + checksum | `Initramfs::xwrite` |
| `do_utime(filename, mtime)` / `do_utime_path(path, mtime)` | per-mtime-restore | `Initramfs::do_utime` |
| `dir_add(name, nlen, mtime)` / `dir_utime()` | per-dir-mtime-defer | `Initramfs::dir_add` / `dir_utime` |
| `reserve_initrd_mem()` | per-arch-stage memblock reservation | `Initramfs::reserve_initrd_mem` |
| `free_initrd_mem(start, end)` | per-initrd-release | `Initramfs::free_initrd_mem` |
| `kexec_free_initrd()` | per-kexec-aware free | `Initramfs::kexec_free_initrd` |
| `populate_initrd_image(err)` | per-CONFIG_BLK_DEV_RAM fallback to `/initrd.image` | `Initramfs::populate_initrd_image` |
| `retain_initrd_param(str)` | per-`retain_initrd` cmdline | shared |
| `keepinitrd_setup(str)` | per-`keepinitrd` arch-alias | shared |
| `initramfs_async_setup(str)` | per-`initramfs_async=` cmdline | shared |
| `decompress_method(buf, len, &name)` | per-magic-detect | shared (lib/decompress*) |
| `bin_attr_initrd` | per-`/sys/firmware/initrd` sysfs file | shared |
| `__initramfs_start[] / __initramfs_size` | per-built-in archive | linker-provided |
| `initrd_start / initrd_end / initrd_below_start_ok` | per-bootloader-blob | shared globals |
| `phys_initrd_start / phys_initrd_size` | per-arch-handoff | shared globals |
| `csum_present` / `io_csum` / `hdr_csum` | per-newc-crc state | static-globals |
| `state` / `next_state` / `victim` / `byte_count` / `this_header` / `next_header` / `collected` / `remains` / `collect` | per-FSM-state | static-globals |
| `header_buf` / `symlink_buf` / `name_buf` | per-unpack staging buffers | per-call alloc |
| `wfile` / `wfile_pos` | per-write-target file | static-globals |
| `initramfs_domain` | per-`ASYNC_DOMAIN_EXCLUSIVE` | `Initramfs::initramfs_domain` |
| `initramfs_cookie` | per-async-cookie | `Initramfs::initramfs_cookie` |
| `do_retain_initrd` | per-`retain_initrd` flag | shared |
| `initramfs_async` | per-`initramfs_async=` flag | shared |
| `message` | per-error-string | static-global |

## Compatibility contract

REQ-1: cpio newc header (110 bytes total, all fields 8-hex-ASCII except magic):
- Bytes 0..6: magic `"070701"` (newc) or `"070702"` (newc-crc); else error.
- Bytes 6..14: ino (`parsed[0]`).
- Bytes 14..22: mode (`parsed[1]`).
- Bytes 22..30: uid (`parsed[2]`).
- Bytes 30..38: gid (`parsed[3]`).
- Bytes 38..46: nlink (`parsed[4]`).
- Bytes 46..54: mtime (`parsed[5]`) — y2106 cutoff.
- Bytes 54..62: body_len (`parsed[6]`).
- Bytes 62..70: major (`parsed[7]`).
- Bytes 70..78: minor (`parsed[8]`).
- Bytes 78..86: rdev-major (`parsed[9]`), 86..94: rdev-minor (`parsed[10]`) → `rdev = new_encode_dev(MKDEV(...))`.
- Bytes 94..102: name_len (`parsed[11]`).
- Bytes 102..110: hdr_csum (`parsed[12]`) — only valid when newc-crc.

REQ-2: `parse_header(s)`:
- `for (i = 0, s += 6; i < 13; i++, s += 8) parsed[i] = simple_strntoul(s, NULL, 16, 8)`.
- Field assignments per REQ-1.

REQ-3: `do_header()`:
- magic check: `070701` → `csum_present = false`; `070702` → `csum_present = true`; else error (special-case `"070707"` → `"incorrect cpio method used: use -H newc option"`).
- `parse_header(collected)`.
- `next_header = this_header + N_ALIGN(name_len) + body_len`; `next_header = (next_header + 3) & ~3` — 4-byte alignment.
- `state = SkipIt`.
- if `name_len <= 0 || name_len > PATH_MAX`: return 0 (skip).
- if `S_ISLNK(mode)`:
  - if `body_len > PATH_MAX`: return 0.
  - `collect = collected = symlink_buf; remains = N_ALIGN(name_len) + body_len; next_state = GotSymlink; state = Collect`.
- if `S_ISREG(mode) || !body_len`: `read_into(name_buf, N_ALIGN(name_len), GotName)`.

REQ-4: `N_ALIGN(len) = (((len) + 1) & ~3) + 2` — cpio name padding (header + name to 4-byte boundary).

REQ-5: FSM states (`enum state`):
- `Start` — `do_start`: `read_into(header_buf, CPIO_HDRLEN, GotHeader)`.
- `Collect` — `do_collect`: copy `min(remains, byte_count)` into `collect`; if `remains == 0` then `state = next_state`.
- `GotHeader` — `do_header`: parse + decide next state.
- `SkipIt` — `do_skip`: advance to `next_header`.
- `GotName` — `do_name`: open regular file / mkdir / mknod / link.
- `CopyFile` — `do_copy`: `xwrite(wfile, victim, min(byte_count, body_len), &wfile_pos)`; on completion `do_utime_path(&wfile->f_path, mtime); fput(wfile)`; if `csum_present && io_csum != hdr_csum`: `error("bad data checksum")`.
- `GotSymlink` — `do_symlink`: split at `N_ALIGN(name_len)`; `init_symlink(target, linkpath)`; `init_chown` AT_SYMLINK_NOFOLLOW; `do_utime`.
- `Reset` — `do_reset`: eat zero padding; if `byte_count && (this_header & 3)`: error `"broken padding"`.

REQ-6: `do_name()`:
- `state = SkipIt; next_state = Reset`.
- Require `collected[name_len - 1] == '\0'`; else `pr_err("initramfs name without nulterm: %.*s")` + `error("malformed archive")` + return 1.
- if `strcmp(collected, "TRAILER!!!") == 0`: `free_hash()`; return 0 (end of cpio).
- `clean_path(collected, mode)`.
- if `S_ISREG(mode)`:
  - `ml = maybe_link()`.
  - if `ml >= 0`:
    - `openflags = O_WRONLY|O_CREAT|O_LARGEFILE; if (ml != 1) openflags |= O_TRUNC`.
    - `wfile = filp_open(collected, openflags, mode)`; if IS_ERR return 0.
    - `wfile_pos = 0; io_csum = 0`.
    - `vfs_fchown(wfile, uid, gid); vfs_fchmod(wfile, mode)`.
    - if `body_len`: `vfs_truncate(&wfile->f_path, body_len)`.
    - `state = CopyFile`.
- else-if `S_ISDIR(mode)`: `init_mkdir(collected, mode); init_chown(collected, uid, gid, 0); init_chmod(collected, mode); dir_add(collected, name_len, mtime)`.
- else-if `S_ISBLK || S_ISCHR || S_ISFIFO || S_ISSOCK`: if `maybe_link() == 0`: `init_mknod(collected, mode, rdev); init_chown(collected, uid, gid, 0); init_chmod(collected, mode); do_utime(collected, mtime)`.

REQ-7: Hardlinks:
- `hash(major, minor, ino) = ((ino + minor + (major << 3)) + ((ino + minor + (major << 3)) >> 5)) & 31` — 32-bucket table `head[32]`.
- `find_link(major, minor, ino, mode, name)`: walk bucket; match on `(ino, minor, major, mode & S_IFMT)`; return existing name or insert new entry and return NULL.
- `maybe_link()`: if `nlink >= 2`: `old = find_link(major, minor, ino, mode, collected); if (old) { clean_path(collected, 0); return (init_link(old, collected) < 0) ? -1 : 1; } return 0`.
- `free_hash()`: free all entries; `hardlink_seen = false`.

REQ-8: `clean_path(path, fmode)`:
- `init_stat(path, &st, AT_SYMLINK_NOFOLLOW)`.
- if exists && `(st.mode ^ fmode) & S_IFMT`: type changed → `init_rmdir(path)` (was dir) or `init_unlink(path)` (was file).

REQ-9: `xwrite(file, p, count, pos)`:
- Loop while `count`:
  - `rv = kernel_write(file, p, count, pos)`.
  - if `rv < 0`: if `EINTR` or `EAGAIN` continue; else return `out ? out : rv`.
  - else if `rv == 0`: break.
  - if `csum_present`: `for i in 0..rv: io_csum += p[i]` — byte-sum checksum (matches newc-crc spec).
  - `p += rv; out += rv; count -= rv`.
- return `out`.

REQ-10: `write_buffer(buf, len)`:
- `byte_count = len; victim = buf`.
- `while (!actions[state]())` loop — handler returns 0 to continue, 1 to yield.
- return `len - byte_count`.

REQ-11: `flush_buffer(bufv, len)` (called by decompressor):
- if `message`: return -1.
- Loop `written = write_buffer(buf, len)`:
  - if `written == len`: done.
  - else inspect `buf[written]`: `'0'` → restart at `state = Start` (new cpio segment); `'\0'` → `state = Reset` (zero padding); else `error("junk within compressed archive")`.
- return `origLen` (total bytes accepted from decompressor).

REQ-12: `unpack_to_rootfs(buf, len)`:
- Allocate `bufs = kmalloc({header[CPIO_HDRLEN], symlink[PATH_MAX + N_ALIGN(PATH_MAX) + 1], name[N_ALIGN(PATH_MAX)]})`; panic on failure (`can't allocate buffers`).
- `header_buf = bufs->header; symlink_buf = bufs->symlink; name_buf = bufs->name`.
- `state = Start; this_header = 0; message = NULL`.
- While `!message && len`:
  - `saved_offset = this_header`.
  - if `*buf == '0' && !(this_header & 3)`: uncompressed cpio segment → `state = Start; written = write_buffer(buf, len); buf += written; len -= written; continue`.
  - if `!*buf`: zero byte; `buf++; len--; this_header++; continue`.
  - `this_header = 0`.
  - `decompress = decompress_method(buf, len, &compress_name)`.
  - `pr_debug("Detected %s compressed data", compress_name)`.
  - if `decompress`: `res = decompress(buf, len, NULL, flush_buffer, NULL, &my_inptr, error)`; if `res`: `error("decompressor failed")`.
  - else if `compress_name`: `pr_err("compression method %s not configured", compress_name); error("decompressor failed")`.
  - else: `error("invalid magic at start of compressed archive")`.
  - if `state != Reset`: `error("junk at the end of compressed archive")`.
  - `this_header = saved_offset + my_inptr; buf += my_inptr; len -= my_inptr`.
- `dir_utime(); free_hash(); kfree(bufs)`.
- return `message`.

REQ-13: Supported decompressors (per `lib/decompress_*` and `decompress_method`):
| Magic prefix | Format | Decompressor | Kconfig |
|---|---|---|---|
| `1f 8b` | gzip | `gunzip` | CONFIG_DECOMPRESS_GZIP |
| `42 5a 68` | bzip2 | `bunzip2` | CONFIG_DECOMPRESS_BZIP2 |
| `5d 00 00` | lzma (legacy) | `unlzma` | CONFIG_DECOMPRESS_LZMA |
| `fd 37 7a 58 5a 00` | xz | `unxz` | CONFIG_DECOMPRESS_XZ |
| `89 4c 5a 4f` | lzo | `unlzo` | CONFIG_DECOMPRESS_LZO |
| `02 21 4c 18` | lz4 | `unlz4` | CONFIG_DECOMPRESS_LZ4 |
| `28 b5 2f fd` | zstd | `unzstd` | CONFIG_DECOMPRESS_ZSTD |
| `'0' '7' '0' '7' '0' '1'/'2'` | uncompressed cpio newc | (none) | always |

REQ-14: `populate_rootfs()` (registered via `rootfs_initcall`):
- `initramfs_cookie = async_schedule_domain(do_populate_rootfs, NULL, &initramfs_domain)`.
- `usermodehelper_enable()`.
- if `!initramfs_async`: `wait_for_initramfs()` synchronously.
- return 0.

REQ-15: `do_populate_rootfs(unused, cookie)`:
- `err = unpack_to_rootfs(__initramfs_start, __initramfs_size)`.
- if `err`: `panic_show_mem("%s", err)` (built-in initramfs must be valid).
- if `!initrd_start || IS_ENABLED(CONFIG_INITRAMFS_FORCE)`: goto done.
- if `CONFIG_BLK_DEV_RAM`: `printk("Trying to unpack rootfs image as initramfs...")`; else `printk("Unpacking initramfs...")`.
- `err = unpack_to_rootfs((char *)initrd_start, initrd_end - initrd_start)`.
- if `err`:
  - CONFIG_BLK_DEV_RAM: `populate_initrd_image(err)` — write blob to `/initrd.image`.
  - else: `printk(KERN_EMERG "Initramfs unpacking failed: %s", err)`.
- `done:` `security_initramfs_populated()`.
- if `!do_retain_initrd && initrd_start && !kexec_free_initrd()`: `free_initrd_mem(initrd_start, initrd_end)`.
- else if `do_retain_initrd && initrd_start`: `bin_attr_initrd.size = end - start; bin_attr_initrd.private = (void *)initrd_start; sysfs_create_bin_file(firmware_kobj, &bin_attr_initrd)`.
- `initrd_start = 0; initrd_end = 0; init_flush_fput()`.

REQ-16: `wait_for_initramfs()`:
- if `!initramfs_cookie`: `pr_warn_once("wait_for_initramfs() called before rootfs_initcalls")`; return (caller's access will fail as if no rootfs).
- else: `async_synchronize_cookie_domain(initramfs_cookie + 1, &initramfs_domain)`.

REQ-17: `reserve_initrd_mem()`:
- `initrd_start = initrd_end = 0` (ignore device-tree-computed virt addr).
- if `!phys_initrd_size`: return.
- `start = round_down(phys_initrd_start, PAGE_SIZE); size = phys_initrd_size + (phys_initrd_start - start); size = round_up(size, PAGE_SIZE)`.
- if `!memblock_is_region_memory(start, size)`: `pr_err("INITRD: 0x%llx+0x%lx is not a memory region")` → goto disable.
- if `memblock_is_region_reserved(start, size)`: `pr_err("INITRD: 0x%llx+0x%lx overlaps in-use memory region")` → goto disable.
- `memblock_reserve(start, size); initrd_start = (unsigned long)__va(phys_initrd_start); initrd_end = initrd_start + phys_initrd_size; initrd_below_start_ok = 1`.
- `disable:` `pr_cont(" - disabling initrd"); initrd_start = 0; initrd_end = 0`.

REQ-18: `free_initrd_mem(start, end)` (weak; arch may override):
- `free_reserved_area((void *)start, (void *)end, POISON_FREE_INITMEM, "initrd")`.

REQ-19: `kexec_free_initrd()` (CONFIG_CRASH_RESERVE):
- `crashk_start/end = __va(crashk_res.{start,end})`.
- if `initrd_start >= crashk_end || initrd_end <= crashk_start`: return false (no overlap; caller does normal free).
- `memset((void *)initrd_start, 0, end - start)` — clear leaked memory.
- if `initrd_start < crashk_start`: `free_initrd_mem(initrd_start, crashk_start)`.
- if `initrd_end > crashk_end`: `free_initrd_mem(crashk_end, initrd_end)`.
- return true.

REQ-20: `populate_initrd_image(err)` (CONFIG_BLK_DEV_RAM fallback):
- `printk("rootfs image is not initramfs (%s); looks like an initrd", err)`.
- `file = filp_open("/initrd.image", O_WRONLY|O_CREAT|O_LARGEFILE, 0700)`; if IS_ERR return.
- `xwrite(file, (char *)initrd_start, initrd_end - initrd_start, &pos)`.
- if short write: `pr_err("/initrd.image: incomplete write")`.
- `fput(file)`.

REQ-21: Cmdline parameters owned here:
- `retain_initrd` (no value) → `do_retain_initrd = 1`.
- `keepinitrd` (arch alias, CONFIG_ARCH_HAS_KEEPINITRD) → same.
- `initramfs_async=BOOL` → `initramfs_async` flag.

## Acceptance Criteria

- [ ] AC-1: A `cpio -o -H newc` archive of an arbitrary tree appended (concatenated) to vmlinuz unpacks to a byte-identical `/` tree on Rookery vs. upstream — verified for files, symlinks, dirs, FIFOs, char devs, block devs, sockets, hardlinks.
- [ ] AC-2: Each of gzip / bzip2 / xz / lzma / lzo / lz4 / zstd compressed initramfs blobs unpacks identically (covers REQ-13).
- [ ] AC-3: A cpio newc-crc (`070702`) archive with a deliberately wrong `hdr_csum` produces error `"bad data checksum"` and aborts that file (but does not panic).
- [ ] AC-4: A cpio archive with `name_len > PATH_MAX` skips the offending entry (does not crash).
- [ ] AC-5: A cpio archive containing `TRAILER!!!` cleanly ends parsing and frees the hardlink hash.
- [ ] AC-6: Concatenated (multi-segment) initramfs: `boot.cpio.gz + 0-padding + overlay.cpio.gz` unpacks both segments, overlay overlays boot.
- [ ] AC-7: Hardlinks across nlink≥2 entries with matching `(major,minor,ino,S_IFMT)` produce `init_link` calls; only first instance allocates content.
- [ ] AC-8: When an entry's existing type differs from the cpio mode (e.g., upstream had a dir at path, new archive has a file), `clean_path` calls `init_rmdir`/`init_unlink` first.
- [ ] AC-9: `populate_rootfs` runs at `rootfs_initcall` time, async by default; `initramfs_async=0` forces sync via `wait_for_initramfs`.
- [ ] AC-10: `wait_for_initramfs()` called before `rootfs_initcalls` emits `pr_warn_once("wait_for_initramfs() called before rootfs_initcalls")` and returns.
- [ ] AC-11: `retain_initrd` cmdline → initrd memory retained, exposed at `/sys/firmware/initrd` (size = blob size).
- [ ] AC-12: Without `retain_initrd` and with no kexec overlap, `free_initrd_mem(initrd_start, initrd_end)` is called after unpack; with kexec overlap, only non-crashk regions are freed.
- [ ] AC-13: CONFIG_BLK_DEV_RAM build: if `unpack_to_rootfs(initrd, ...)` fails on the bootloader blob, `populate_initrd_image` writes the blob to `/initrd.image` for legacy initrd handling.
- [ ] AC-14: Built-in initramfs failure (`unpack_to_rootfs(__initramfs_start, __initramfs_size)` returns non-NULL) panics with `panic_show_mem`.
- [ ] AC-15: CONFIG_INITRAMFS_FORCE: bootloader-supplied initrd is ignored even if `initrd_start != 0`.

## Architecture

```
struct CpioHash {
  ino: i32,
  minor: i32,
  major: i32,
  mode: u16,         // umode_t
  name: [u8; N_ALIGN(PATH_MAX)],
}

struct UnpackState {
  // FSM
  state: State,            // Start | Collect | GotHeader | SkipIt | GotName | CopyFile | GotSymlink | Reset
  next_state: State,
  victim: *u8,             // input cursor
  byte_count: usize,       // input bytes remaining in current write_buffer call
  this_header: i64,        // logical position of current header
  next_header: i64,        // logical position of next header
  collected: *u8,          // pointer to current collected datum
  remains: i64,            // bytes still to collect into 'collect'
  collect: *u8,            // collect destination

  // current entry (cpio newc)
  ino: u64, major: u64, minor: u64, nlink: u64,
  mode: u16, body_len: u64, name_len: u64,
  uid: u32, gid: u32, rdev: u32, mtime: i64,
  hdr_csum: u32,
  csum_present: bool, io_csum: u32,

  // write target (for CopyFile)
  wfile: Option<*File>,
  wfile_pos: i64,

  // staging buffers
  header_buf: *u8,
  symlink_buf: *u8,
  name_buf: *u8,

  // hardlink hash
  hash_head: [Option<Box<CpioHash>>; 32],
  hardlink_seen: bool,

  // dir mtime defer
  dir_list: List<DirEntry>,

  // error sticky
  message: Option<&'static str>,
}

const N_ALIGN(len): usize = (((len) + 1) & !3) + 2;
const CPIO_HDRLEN: usize = 110;
```

`Initramfs::populate_rootfs() -> i32`:
1. `initramfs_cookie = async_schedule_domain(do_populate_rootfs, None, &initramfs_domain)`.
2. `usermodehelper_enable()`.
3. if `!initramfs_async`: `wait_for_initramfs()`.
4. return 0.

`Initramfs::do_populate_rootfs(_, _cookie)`:
1. `err = unpack_to_rootfs(__initramfs_start, __initramfs_size)`.
2. if `err.is_some()`: `panic_show_mem!("{}", err)`.
3. if `initrd_start == 0 || CONFIG_INITRAMFS_FORCE`: goto step 7.
4. `pr_info` per CONFIG_BLK_DEV_RAM branch.
5. `err = unpack_to_rootfs(initrd_start as *u8, (initrd_end - initrd_start) as usize)`.
6. if `err.is_some()`:
   - CONFIG_BLK_DEV_RAM → `populate_initrd_image(err)`.
   - else → `pr_emerg!("Initramfs unpacking failed: {}", err)`.
7. `security_initramfs_populated()`.
8. if `!do_retain_initrd && initrd_start && !kexec_free_initrd()`: `free_initrd_mem(initrd_start, initrd_end)`.
9. else-if `do_retain_initrd && initrd_start`: configure `bin_attr_initrd`; `sysfs_create_bin_file(firmware_kobj, &bin_attr_initrd)`.
10. `initrd_start = 0; initrd_end = 0`.
11. `init_flush_fput()`.

`Initramfs::wait_for_initramfs()`:
1. if `!initramfs_cookie`: `pr_warn_once!("wait_for_initramfs() called before rootfs_initcalls")`; return.
2. else: `async_synchronize_cookie_domain(initramfs_cookie + 1, &initramfs_domain)`.

`Initramfs::unpack_to_rootfs(buf, len) -> Option<&'static str>`:
1. `bufs = kmalloc({header_buf, symlink_buf, name_buf}).or(panic_show_mem!("can't allocate buffers"))`.
2. wire `header_buf/symlink_buf/name_buf` to `bufs`.
3. `state = Start; this_header = 0; message = None`.
4. while `message.is_none() && len > 0`:
   - `saved_offset = this_header`.
   - if `*buf == b'0' && (this_header & 3) == 0`: uncompressed cpio inline:
     - `state = Start; written = write_buffer(buf, len); buf += written; len -= written; continue`.
   - if `*buf == 0`: `buf++; len--; this_header++; continue`.
   - `this_header = 0`.
   - `(decompress, compress_name) = decompress_method(buf, len)`.
   - `pr_debug!("Detected {} compressed data", compress_name)`.
   - if `decompress.is_some()`: `res = decompress(buf, len, None, flush_buffer, None, &mut my_inptr, error)`; if `res != 0`: `error("decompressor failed")`.
   - else if `compress_name.is_some()`: `pr_err!("compression method {} not configured", compress_name); error("decompressor failed")`.
   - else: `error("invalid magic at start of compressed archive")`.
   - if `state != Reset`: `error("junk at the end of compressed archive")`.
   - `this_header = saved_offset + my_inptr; buf += my_inptr; len -= my_inptr`.
5. `dir_utime(); free_hash(); kfree(bufs)`.
6. return `message`.

`Initramfs::write_buffer(buf, len) -> usize`:
1. `byte_count = len; victim = buf`.
2. while `actions[state]() == 0`: continue.
3. return `len - byte_count`.

`Initramfs::flush_buffer(bufv, len) -> i64`:
1. if `message.is_some()`: return -1.
2. `buf = bufv; orig_len = len`.
3. while `(written = write_buffer(buf, len)) < len && message.is_none()`:
   - `c = buf[written]`.
   - if `c == '0'`: `buf += written; len -= written; state = Start`.
   - else if `c == 0`: `buf += written; len -= written; state = Reset`.
   - else: `error("junk within compressed archive")`.
4. return `orig_len`.

`Initramfs::parse_header(s)`:
1. `for i in 0..13: parsed[i] = simple_strntoul(&s[6 + i * 8], 16, 8)`.
2. assign per REQ-1.

`Initramfs::do_header() -> i32`:
- See REQ-3 step-by-step.

`Initramfs::do_name() -> i32`:
- See REQ-6 step-by-step.

`Initramfs::do_copy() -> i32`:
1. if `byte_count >= body_len`:
   - if `xwrite(wfile, victim, body_len, &wfile_pos) != body_len`: `error("write error")`.
   - `do_utime_path(&wfile.f_path, mtime); fput(wfile)`.
   - if `csum_present && io_csum != hdr_csum`: `error("bad data checksum")`.
   - `eat(body_len); state = SkipIt`; return 0.
2. else:
   - if `xwrite(wfile, victim, byte_count, &wfile_pos) != byte_count`: `error("write error")`.
   - `body_len -= byte_count; eat(byte_count)`; return 1.

`Initramfs::do_symlink() -> i32`:
1. if `collected[name_len - 1] != b'\0'`: `pr_err`/`error("malformed archive")`; return 1.
2. `collected[N_ALIGN(name_len) + body_len] = 0`.
3. `clean_path(collected, 0)`.
4. `init_symlink(&collected[N_ALIGN(name_len)..], collected)`.
5. `init_chown(collected, uid, gid, AT_SYMLINK_NOFOLLOW)`.
6. `do_utime(collected, mtime)`.
7. `state = SkipIt; next_state = Reset`; return 0.

`Initramfs::do_reset() -> i32`:
1. while `byte_count > 0 && *victim == b'\0'`: `eat(1)`.
2. if `byte_count > 0 && (this_header & 3) != 0`: `error("broken padding")`.
3. return 1.

`Initramfs::find_link(major, minor, ino, mode, name) -> Option<*u8>`:
1. `bucket = hash(major, minor, ino) % 32`.
2. for `q` in `head[bucket]`: if `q.ino == ino && q.minor == minor && q.major == major && ((q.mode ^ mode) & S_IFMT) == 0`: return `Some(q.name)`.
3. `q = kmalloc(CpioHash).or_else(|| panic_show_mem!("can't allocate link hash entry"))`.
4. `q = { major, minor, ino, mode, name: strscpy(name), next: None }`.
5. push to `head[bucket]`.
6. `hardlink_seen = true`.
7. return None.

`Initramfs::free_hash()`:
1. if `!hardlink_seen`: return.
2. for `bucket in 0..32`: while `head[bucket].is_some()`: `pop and kfree`.
3. `hardlink_seen = false`.

`Initramfs::maybe_link() -> i32`:
1. if `nlink < 2`: return 0.
2. `old = find_link(major, minor, ino, mode, collected)`.
3. if `old.is_none()`: return 0.
4. `clean_path(collected, 0)`.
5. return if `init_link(old, collected) < 0 { -1 } else { 1 }`.

`Initramfs::xwrite(file, p, count, pos) -> isize`:
1. `out = 0`.
2. while `count > 0`:
   - `rv = kernel_write(file, p, count, pos)`.
   - if `rv < 0`: if `rv == -EINTR || rv == -EAGAIN`: continue; else return if `out > 0 { out } else { rv }`.
   - if `rv == 0`: break.
   - if `csum_present`: for `i in 0..rv`: `io_csum += p[i] as u32`.
   - `p += rv; out += rv; count -= rv`.
3. return `out`.

`Initramfs::reserve_initrd_mem()`:
- See REQ-17 step-by-step.

`Initramfs::free_initrd_mem(start, end)` (weak):
- `free_reserved_area(start, end, POISON_FREE_INITMEM, "initrd")`.

`Initramfs::kexec_free_initrd() -> bool`:
- See REQ-19 step-by-step.

`Initramfs::populate_initrd_image(err)`:
- See REQ-20 step-by-step.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cpio_header_total_size` | INVARIANT | per-`parse_header`: consumes exactly 110 bytes (CPIO_HDRLEN) from `s`. |
| `cpio_magic_validated` | INVARIANT | per-`do_header`: returns error if magic ∉ `{"070701", "070702"}`. |
| `name_len_bound` | INVARIANT | per-`do_header`: `name_len > PATH_MAX` ⟹ skip (no buffer overflow). |
| `symlink_body_len_bound` | INVARIANT | per-`do_header` S_ISLNK: `body_len > PATH_MAX` ⟹ skip. |
| `next_header_alignment` | INVARIANT | per-`do_header`: `next_header & 3 == 0` (4-byte aligned). |
| `hardlink_hash_collision_safe` | INVARIANT | per-`find_link`: bucket walks 32 slots; insertion never overflows. |
| `eat_no_underflow` | INVARIANT | per-`eat(n)`: `n <= byte_count` (FSM guarantee). |
| `unpack_input_consumed` | INVARIANT | per-`unpack_to_rootfs`: advances `buf` by at least 1 per loop iteration when `*buf != 0`. |
| `csum_when_present_verified` | INVARIANT | per-newc-crc: `csum_present == true` ⟹ `io_csum` compared against `hdr_csum` in `do_copy`. |
| `trailer_terminates` | INVARIANT | per-`do_name`: `collected == "TRAILER!!!"` ⟹ `free_hash` + return 0 (no further state changes). |
| `wfile_paired_fput` | INVARIANT | per-`do_copy`: `fput(wfile)` happens iff `wfile = filp_open(...)` succeeded earlier. |

### Layer 2: TLA+

`init/initramfs.tla`:
- Per-FSM transitions modeled per `enum state`.
- Per-`flush_buffer` invoked once per decompressor with full output.
- Properties:
  - `safety_state_transitions_legal` — per-FSM: only the documented transitions occur (Start→GotHeader→{SkipIt | GotName | Collect→GotSymlink}, GotName→{CopyFile | SkipIt}, CopyFile→SkipIt, GotSymlink→SkipIt→Reset).
  - `safety_input_byte_count_monotone` — per-`write_buffer`: `byte_count` only decreases.
  - `safety_this_header_monotone_per_segment` — per-segment: `this_header` only advances.
  - `safety_trailer_idempotent` — per-`do_name` TRAILER: subsequent input produces no FS changes.
  - `liveness_unpack_terminates` — per-`unpack_to_rootfs`: terminates for any finite input.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Initramfs::parse_header` post: parsed fields == big-endian-hex of input bytes | `Initramfs::parse_header` |
| `Initramfs::do_header` post: state ∈ {SkipIt, Collect, error} | `Initramfs::do_header` |
| `Initramfs::do_name` post: state ∈ {SkipIt, CopyFile} | `Initramfs::do_name` |
| `Initramfs::do_copy` post: `body_len` decreases by `min(byte_count, body_len)` | `Initramfs::do_copy` |
| `Initramfs::do_symlink` post: `init_symlink(target, link)` called; state == SkipIt | `Initramfs::do_symlink` |
| `Initramfs::find_link` post: returns Some(existing.name) iff hash match; else None and new entry inserted | `Initramfs::find_link` |
| `Initramfs::xwrite` post: returns total bytes written; csum updated iff `csum_present` | `Initramfs::xwrite` |
| `Initramfs::unpack_to_rootfs` post: all `bufs` freed; `free_hash` called | `Initramfs::unpack_to_rootfs` |
| `Initramfs::wait_for_initramfs` post: blocks until `initramfs_cookie + 1` synchronized | `Initramfs::wait_for_initramfs` |
| `Initramfs::reserve_initrd_mem` post: `(initrd_start, initrd_end)` set iff `memblock_reserve` succeeded | `Initramfs::reserve_initrd_mem` |

### Layer 4: Verus/Creusot functional

`Per-cpio-newc segment: parse_header → do_header → (S_ISREG: do_name → do_copy | S_ISDIR: do_name | S_ISLNK: do_symlink | S_ISBLK/CHR/FIFO/SOCK: do_name+init_mknod) → SkipIt → Reset` semantic equivalence: per-`cpio(5)` man page, per-`Documentation/driver-api/early-userspace/early_userspace_support.rst`, per-`Documentation/filesystems/ramfs-rootfs-initramfs.rst`.

`Per-multi-segment: concatenated `0xx`-prefixed cpio segments with optional decompressors between` semantic equivalence: each segment overlays the previous.

## Hardening

(Inherits row-1 features from `init/00-overview.md` § Hardening.)

initramfs reinforcement:

- **Per-`name_len > PATH_MAX` skip** — defense against per-malicious-long-name buffer overflow.
- **Per-`body_len > PATH_MAX` for symlinks skip** — defense against per-malicious-symlink overflow.
- **Per-`collected[name_len - 1] == '\0'` check** — defense against per-non-null-terminated path → `init_*` UAF.
- **Per-newc-crc `io_csum == hdr_csum` verification** — defense against per-corrupted-payload silent acceptance.
- **Per-`clean_path` type-change unlink** — defense against per-type-confusion when overlay archives swap dir↔file.
- **Per-`csum_present` only when magic == `070702`** — defense against per-spoofed-crc unchecked archive.
- **Per-`free_hash` on TRAILER and on every `unpack_to_rootfs` return** — defense against per-hash-leak across builds.
- **Per-`security_initramfs_populated` LSM hook** — defense against per-policy-bypass on initramfs contents.
- **Per-`memblock_is_region_memory` + `memblock_is_region_reserved` checks** — defense against per-initrd overlap with reserved memory.
- **Per-`free_initrd_mem` uses `POISON_FREE_INITMEM`** — defense against per-UAF on stale initrd pages.
- **Per-`kexec_free_initrd` memsets crashk-overlap** — defense against per-secret-leak into kexec region.
- **Per-`panic_show_mem` on built-in initramfs failure** — defense against per-boot-with-corrupted-image proceeding silently.
- **Per-`pr_warn_once` from `wait_for_initramfs` pre-rootfs_initcalls** — defense against per-silent-deadlock-or-stale-FS access.

## Open Questions

- Per-`CONFIG_INITRAMFS_PRESERVE_MTIME=n` builds, `do_utime` / `do_utime_path` / `dir_add` / `dir_utime` are no-ops; Rookery follows the same Kconfig contract. Decide whether to make mtime preservation mandatory in Rookery to reduce variance — TBD with maintainer.
- Per-`CONFIG_ARCH_HAS_KEEPINITRD` arch alias: should Rookery accept `keepinitrd` on x86 too (as a no-op alias for `retain_initrd`)?

## Out of Scope

- `decompress_method` magic detection — covered by `lib/decompress.md` (or each per-format Tier-3).
- Individual decompressor implementations (`gunzip`, `bunzip2`, `unlzma`, `unxz`, `unlzo`, `unlz4`, `unzstd`) — covered in `lib/decompress-*.md`.
- `ramfs` / tmpfs filesystem itself — covered in `fs/ramfs.md`.
- `init_mkdir` / `init_chown` / `init_chmod` / `init_link` / `init_symlink` / `init_unlink` / `init_rmdir` / `init_mknod` / `init_utimes` / `init_stat` / `init_eaccess` / `init_dup` / `init_flush_fput` syscall-equivalent helpers — covered in `init/init_syscalls.md` (or folded into `fs/exec.md`).
- `prepare_namespace` / `do_mounts*` — covered in `init/rootfs-mount.md`.
- `usermodehelper_enable` — covered in `kernel/umh.md`.
- `async_schedule_domain` / `async_synchronize_cookie_domain` — covered in `kernel/async.md`.
- `sysfs_bin_attr_simple_read` / `sysfs_create_bin_file` — covered in `fs/sysfs.md`.
- Bootconfig parser (`xbc_*`) — covered in `init/main.md` § Bootconfig.
- Implementation code.
