---
title: "Tier-3: drivers/mtd/mtdcore.c — MTD core registration + I/O dispatch"
tags: ["tier-3", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **MTD core** is the registration, refcount, and dispatch hub for every flash device exposed to Linux — raw NAND, SPI-NOR, parallel NOR, OneNAND, SPI-NAND, RAM-backed test MTDs, and partitions carved from any of those. Per-device descriptor: `struct mtd_info` (name, type, size, erasesize, writesize, oobsize, flags, ECC layout, function pointers `_read`/`_write`/`_erase`/`_read_oob`/`_write_oob`/`_lock`/`_unlock`/`_block_isbad`/`_block_markbad`/`_get_device`/`_put_device`/`_panic_write`, kref refcount, embedded `struct device`). Per-IDR allocation: `mtd_idr` allocates `mtd<N>` indices under `mtd_table_mutex`. Per-class: `/sys/class/mtd/mtd<N>/{name,type,size,erasesize,writesize,oobsize,subpagesize,oobavail,numeraseregions,flags,ecc_strength,ecc_step_size,bitflip_threshold,corrected_bits,ecc_failures,bad_blocks,bbt_blocks}` (read-only except `bitflip_threshold`). Per-cdev: `mtd_class` + `device_create` for `/dev/mtd<N>` (RW) and `/dev/mtd<N>ro` (RO mirror). Per-OTP: factory + user OTP regions exposed as NVMEM providers (`otp_factory_nvmem`, `otp_user_nvmem`). Per-partition: `mtd_device_parse_register` walks parsers (cmdlineparts, ofparts, redboot, ...) and registers child partition MTDs that inherit master's ops via `mtd_get_master()`. Per-master/partition refcount: child `kref_get` propagates up the parent chain on `__get_mtd_device`. Per-notifier: `register_mtd_user` / `unregister_mtd_user` invokes `add`/`remove` callbacks for filesystem layers (UBI, JFFS2, mtdblock). Per-`MTD_SLC_ON_MLC_EMULATION`: MLC NAND can be exposed as SLC by halving address space (pairing scheme). Per-`ledtrig_mtd_activity`: every erase/write triggers `mtd` LED activity. Per-`ALLOW_ERROR_INJECTION`: `mtd_erase`, `mtd_read`, `mtd_write`, `mtd_block_markbad` are error-injection points. Critical for: every embedded boot ROM, every NAND filesystem, every UBI volume, every persistent storage that isn't a true block device.

This Tier-3 covers `drivers/mtd/mtdcore.c` (~2660 lines).

### Acceptance Criteria

- [ ] AC-1: add_mtd_device: rejects when writesize == 0 (BUG_ON).
- [ ] AC-2: add_mtd_device: rejects when both `_write` and `_write_oob` set (-EINVAL).
- [ ] AC-3: mtd_device_parse_register: cmdline `mtdparts=` parses parts byte-identically to upstream.
- [ ] AC-4: /sys/class/mtd/mtd<N>/ attribute byte-formats match upstream (sysfs_emit "%s\n", "0x%lx\n", "%u\n").
- [ ] AC-5: /dev/mtd<N> + /dev/mtd<N>ro chardev nodes created.
- [ ] AC-6: __get_mtd_device: try_module_get failure → -ENODEV.
- [ ] AC-7: mtd_erase: !MTD_WRITEABLE → -EROFS; addr+len OOB → -EINVAL.
- [ ] AC-8: mtd_panic_write: master.oops_panic_write becomes true.
- [ ] AC-9: mtd_block_markbad: idempotent (already-bad returns 0).
- [ ] AC-10: OTP NVMEM: factory + user providers visible at /sys/bus/nvmem/devices/.
- [ ] AC-11: mtd_notifier add/remove called once per add_mtd_device / del_mtd_device.
- [ ] AC-12: SLC-on-MLC partition: erasesize halved + addresses translated via pairing.
- [ ] AC-13: linux,rootfs DT property → ROOT_DEV = MKDEV(MTD_BLOCK_MAJOR, index) (builtin only).
- [ ] AC-14: mtd_read short-return on EUCLEAN: ecc_stats.corrected incremented + propagated up parent chain.
- [ ] AC-15: del_mtd_device: idr_find mismatch → -ENODEV (no UAF on stale handle).

### Architecture

```
struct MtdInfo {
  type: u8,                                // MTD_NORFLASH | NANDFLASH | MLCNANDFLASH | ...
  flags: u32,                              // MTD_WRITEABLE | MTD_NO_ERASE | ...
  size: u64,
  erasesize: u32,
  writesize: u32,
  writesize_shift: u32, writesize_mask: u32,
  erasesize_shift: u32, erasesize_mask: u32,
  oobsize: u32, oobavail: u32,
  subpagesize: u32,
  bitflip_threshold: u32,
  ecc_strength: u32, ecc_step_size: u32,
  ecc_stats: MtdEccStats,
  numeraseregions: i32,
  eraseregions: *MtdEraseRegionInfo,
  name: *const u8,
  index: i32,
  refcnt: Kref,
  dev: Device,
  owner: *Module,
  parent: Option<*MtdInfo>,
  master: MtdMaster {                      // valid only on master
    partitions_lock: Mutex,
    chrdev_lock: Mutex,
  },
  partitions: ListHead,
  usecount: i32,
  pairing: Option<*const MtdPairingScheme>,
  ooblayout: Option<*const MtdOoblayoutOps>,
  otp_user_nvmem: Option<*NvmemDevice>,
  otp_factory_nvmem: Option<*NvmemDevice>,
  reboot_notifier: NotifierBlock,
  // op fn ptrs (master-only)
  _erase: Option<fn(&MtdInfo, &mut EraseInfo) -> i32>,
  _read: Option<fn(&MtdInfo, u64, usize, &mut usize, &mut [u8]) -> i32>,
  _write: Option<fn(&MtdInfo, u64, usize, &mut usize, &[u8]) -> i32>,
  _read_oob: Option<fn(&MtdInfo, u64, &mut MtdOobOps) -> i32>,
  _write_oob: Option<fn(&MtdInfo, u64, &mut MtdOobOps) -> i32>,
  _panic_write: Option<fn(&MtdInfo, u64, usize, &mut usize, &[u8]) -> i32>,
  _point: Option<fn(&MtdInfo, u64, usize, &mut usize, &mut *const u8, &mut Option<u64>) -> i32>,
  _unpoint: Option<fn(&MtdInfo, u64, usize) -> i32>,
  _lock: Option<fn(&MtdInfo, u64, u64) -> i32>,
  _unlock: Option<fn(&MtdInfo, u64, u64) -> i32>,
  _is_locked: Option<fn(&MtdInfo, u64, u64) -> i32>,
  _block_isbad: Option<fn(&MtdInfo, u64) -> i32>,
  _block_markbad: Option<fn(&MtdInfo, u64) -> i32>,
  _block_isreserved: Option<fn(&MtdInfo, u64) -> i32>,
  _get_device: Option<fn(&MtdInfo) -> i32>,
  _put_device: Option<fn(&MtdInfo)>,
  _reboot: Option<fn(&MtdInfo)>,
  // ...
}
```

`Mtd::add_device(mtd) -> Result<(), Errno>`:
1. /* Idempotence + sanity */
2. if mtd.dev.type.is_some(): WARN_ONCE; return Err(EEXIST).
3. assert mtd.writesize != 0.
4. if mtd._write.is_some() ∧ mtd._write_oob.is_some(): return Err(EINVAL).
5. if mtd._read.is_some() ∧ mtd._read_oob.is_some(): return Err(EINVAL).
6. master = mtd_get_master(mtd).
7. if !master._erase.is_some() ∧ !(mtd.flags & MTD_NO_ERASE): return Err(EINVAL).
8. /* SLC-on-MLC validation */
9. if mtd.flags & MTD_SLC_ON_MLC_EMULATION:
   - if !mtd_is_partition(mtd) ∨ master.type != MTD_MLCNANDFLASH ∨ master.pairing.is_none() ∨ master._writev.is_some(): return Err(EINVAL).
10. let _g = MTD_TABLE_MUTEX.lock().
11. /* IDR alloc */
12. ofidx = mtd_get_of_node(mtd).and_then(|np| of_alias_get_id(np, "mtd")).unwrap_or(-1).
13. i = if ofidx ≥ 0: MTD_IDR.alloc_range(mtd, ofidx, ofidx+1)?; else: MTD_IDR.alloc(mtd)?.
14. mtd.index = i.
15. kref_init(&mtd.refcnt).
16. /* Defaults + shifts */
17. if mtd.bitflip_threshold == 0: mtd.bitflip_threshold = mtd.ecc_strength.
18. if mtd.flags & MTD_SLC_ON_MLC_EMULATION:
    - ngroups = mtd_pairing_groups(master).
    - mtd.erasesize /= ngroups; mtd.size = (mtd_div_by_eb(mtd.size, master) as u64) * mtd.erasesize.
19. mtd.erasesize_shift = if mtd.erasesize.is_power_of_two(): ffs(mtd.erasesize)-1; else 0.
20. mtd.erasesize_mask = (1 << mtd.erasesize_shift) - 1.
21. analogous writesize_shift / writesize_mask.
22. /* Powerup-locked NOR unlock */
23. if (mtd.flags & MTD_WRITEABLE) ∧ (mtd.flags & MTD_POWERUP_LOCK):
    - err = Mtd::unlock(mtd, 0, mtd.size).
    - if err != 0 ∧ err != -EOPNOTSUPP: pr_warning("%s: unlock failed", mtd.name).
    - (failures ignored; writes may not work.)
24. /* Sysfs device */
25. mtd.dev.type = &MTD_DEVTYPE.
26. mtd.dev.class = &MTD_CLASS.
27. mtd.dev.devt = MTD_DEVT(i).
28. dev_set_name(&mtd.dev, "mtd%d", i)?.
29. dev_set_drvdata(&mtd.dev, mtd).
30. mtd_check_of_node(mtd).
31. of_node_get(mtd_get_of_node(mtd)).
32. device_register(&mtd.dev)? — on Err: put_device + jump fail_added.
33. /* NVMEM partition cells */
34. Mtd::nvmem_add(mtd)?.
35. Mtd::debugfs_populate(mtd).
36. /* RO mirror */
37. device_create(&MTD_CLASS, mtd.dev.parent, MTD_DEVT(i)+1, NULL, "mtd%dro", i).
38. /* Notifier fanout */
39. for not in MTD_NOTIFIERS.iter(): (not.add)(mtd).
40. drop(_g).
41. /* DT linux,rootfs */
42. if of_property_read_bool(mtd_get_of_node(mtd), "linux,rootfs"):
    - if IS_BUILTIN(CONFIG_MTD): ROOT_DEV = MKDEV(MTD_BLOCK_MAJOR, mtd.index); pr_info(...).
    - else: pr_warning(...).
43. __module_get(THIS_MODULE).
44. Ok(()).

`Mtd::del_device(mtd) -> Result<(), Errno>`:
1. let _g = MTD_TABLE_MUTEX.lock().
2. if MTD_IDR.find(mtd.index) != Some(mtd): return Err(ENODEV).
3. for not in MTD_NOTIFIERS.iter(): (not.remove)(mtd).
4. kref_put(&mtd.refcnt, Mtd::device_release).
5. Ok(()).
6. /* device_release: MTD_IDR.remove(index); module_put(THIS_MODULE); of_node_put; device_unregister. */

`Mtd::device_parse_register(mtd, types, parser_data, parts, nr_parts) -> Result<(), Errno>`:
1. Mtd::set_dev_defaults(mtd).
2. Mtd::otp_nvmem_add(mtd)?.
3. if CONFIG_MTD_PARTITIONED_MASTER: Mtd::add_device(mtd)?.
4. if CONFIG_MTD_VIRT_CONCAT: mtd_virt_concat_node_create()?.
5. ret = Mtd::parse_partitions(mtd, types, parser_data).
6. match ret:
   - Err(EPROBE_DEFER) → propagate.
   - Ok(n > 0) → ok.
   - Ok(0) ∨ Err(_) ∧ nr_parts > 0: Mtd::add_partitions(mtd, parts, nr_parts)?.
   - Ok(0) ∧ nr_parts == 0 ∧ !device_is_registered(&mtd.dev): Mtd::add_device(mtd)?.
7. if mtd._reboot.is_some() ∧ mtd.reboot_notifier.notifier_call.is_none():
   - mtd.reboot_notifier.notifier_call = Mtd::reboot_notifier.
   - register_reboot_notifier(&mtd.reboot_notifier).
8. On error: nvmem_unregister(mtd.otp_user_nvmem); nvmem_unregister(mtd.otp_factory_nvmem); if device_is_registered: del_mtd_device.

`Mtd::get_device_locked(mtd) -> Result<(), Errno>`:
1. master = mtd_get_master(mtd).
2. if master._get_device.is_some(): master._get_device(mtd)?.
3. if !try_module_get(master.owner):
   - if master._put_device.is_some(): master._put_device(master).
   - return Err(ENODEV).
4. let mut node = Some(mtd).
5. while let Some(m) = node:
   - if m != master: kref_get(&m.refcnt).
   - node = m.parent.
6. if CONFIG_MTD_PARTITIONED_MASTER: kref_get(&master.refcnt).
7. Ok(()).

`Mtd::erase(mtd, instr) -> Result<(), Errno>`:
1. master = mtd_get_master(mtd).
2. mst_ofs = mtd_get_master_ofs(mtd, 0).
3. let mut adjinstr = *instr.
4. instr.fail_addr = MTD_FAIL_ADDR_UNKNOWN.
5. if !mtd.erasesize ∨ !master._erase: return Err(ENOTSUPP).
6. if instr.addr ≥ mtd.size ∨ instr.len > mtd.size - instr.addr: return Err(EINVAL).
7. if !(mtd.flags & MTD_WRITEABLE): return Err(EROFS).
8. if instr.len == 0: return Ok(()).
9. ledtrig_mtd_activity().
10. if mtd.flags & MTD_SLC_ON_MLC_EMULATION:
    - adjinstr.addr = mtd_div_by_eb(instr.addr, mtd) * master.erasesize.
    - adjinstr.len = mtd_div_by_eb(instr.addr+instr.len, mtd)*master.erasesize - adjinstr.addr.
11. adjinstr.addr += mst_ofs.
12. ret = master._erase(master, &adjinstr).
13. if adjinstr.fail_addr != MTD_FAIL_ADDR_UNKNOWN:
    - instr.fail_addr = adjinstr.fail_addr - mst_ofs.
    - if mtd.flags & MTD_SLC_ON_MLC_EMULATION:
      - instr.fail_addr = mtd_div_by_eb(instr.fail_addr, master) * mtd.erasesize.
14. ret.

`Mtd::read_oob(mtd, from, ops) -> Result<(), Errno>`:
1. ret = Mtd::check_oob_ops(mtd, from, ops)?.
2. ops.oobretlen = 0; ops.retlen = 0.
3. master = mtd_get_master(mtd).
4. old_stats = master.ecc_stats.
5. ledtrig_mtd_activity().
6. if mtd.flags & MTD_SLC_ON_MLC_EMULATION:
   - ret = Mtd::io_emulated_slc(mtd, from, /*read=*/true, ops).
7. else:
   - ret = Mtd::read_oob_std(mtd, from, ops).
8. Mtd::update_ecc_stats(mtd, master, &old_stats).
9. // Bubble corrected/failed up parent chain so partitions see same stats as master.
10. ret.

### Out of Scope

- `drivers/mtd/mtdchar.c` /dev/mtd<N> ioctl handlers (covered in `drivers/mtd/chardev-block.md`)
- `drivers/mtd/mtdblock.c` /dev/mtdblock<N> block emulation (covered in `drivers/mtd/chardev-block.md`)
- `drivers/mtd/mtdpart.c` partition arithmetic + parsers (covered in `drivers/mtd/parsers.md`)
- `drivers/mtd/mtdoops.c` + `drivers/mtd/mtdswap.c` (covered in `drivers/mtd/oops-swap.md`)
- `drivers/mtd/nand/raw/` raw NAND bus + per-controller drivers (covered in `drivers/mtd/nand-raw.md`)
- `drivers/mtd/spi-nor/` SPI-NOR chip support (covered in `drivers/mtd/spi-nor.md`)
- `drivers/mtd/ubi/` UBI wear-leveling layer (covered in `drivers/mtd/ubi.md`)
- `fs/jffs2/` + `fs/ubifs/` filesystems on top of MTD (covered in Tier-2 fs/)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mtd_info` | per-device descriptor | `MtdInfo` |
| `struct mtd_oob_ops` | per-OOB I/O request | `MtdOobOps` |
| `struct mtd_notifier` | per-user add/remove cb | `MtdNotifier` |
| `mtd_class` | per-sysfs class | `MTD_CLASS` |
| `mtd_idr` | per-index allocator | `MTD_IDR` |
| `mtd_table_mutex` | per-table lock | `MTD_TABLE_MUTEX` |
| `add_mtd_device()` | per-register | `Mtd::add_device` |
| `del_mtd_device()` | per-unregister | `Mtd::del_device` |
| `mtd_device_parse_register()` | per-register-with-parts | `Mtd::device_parse_register` |
| `mtd_device_unregister()` | per-unregister-master | `Mtd::device_unregister` |
| `register_mtd_user()` / `unregister_mtd_user()` | per-notifier | `Mtd::register_user` / `unregister_user` |
| `get_mtd_device()` / `put_mtd_device()` | per-handle-with-lock | `Mtd::get_device` / `put_device` |
| `__get_mtd_device()` / `__put_mtd_device()` | per-handle-locked | `Mtd::get_device_locked` / `put_device_locked` |
| `get_mtd_device_nm()` | per-lookup-by-name | `Mtd::get_device_by_name` |
| `of_get_mtd_device_by_node()` | per-DT-lookup | `Mtd::get_device_by_of_node` |
| `__mtd_next_device()` | per-iter | `Mtd::next_device` |
| `mtd_erase()` | per-erase | `Mtd::erase` |
| `mtd_read()` / `mtd_write()` | per-data-I/O | `Mtd::read` / `write` |
| `mtd_read_oob()` / `mtd_write_oob()` | per-OOB-I/O | `Mtd::read_oob` / `write_oob` |
| `mtd_panic_write()` | per-panic-context write | `Mtd::panic_write` |
| `mtd_point()` / `mtd_unpoint()` | per-XIP map | `Mtd::point` / `unpoint` |
| `mtd_get_unmapped_area()` | per-NOMMU mmap | `Mtd::get_unmapped_area` |
| `mtd_writev()` / `default_mtd_writev` | per-vectored write | `Mtd::writev` |
| `mtd_lock()` / `mtd_unlock()` / `mtd_is_locked()` | per-NOR-lock | `Mtd::lock` / `unlock` / `is_locked` |
| `mtd_block_isreserved()` / `mtd_block_isbad()` / `mtd_block_markbad()` | per-NAND-bbt | `Mtd::block_isreserved` / `block_isbad` / `block_markbad` |
| `mtd_get_fact_prot_info()` / `mtd_read_fact_prot_reg()` | per-factory-OTP | `Mtd::get_fact_prot_info` / `read_fact_prot_reg` |
| `mtd_get_user_prot_info()` / `mtd_read_user_prot_reg()` / `mtd_write_user_prot_reg()` / `mtd_lock_user_prot_reg()` / `mtd_erase_user_prot_reg()` | per-user-OTP | `Mtd::*_user_prot_reg` |
| `mtd_ooblayout_ecc()` / `mtd_ooblayout_free()` / `mtd_ooblayout_find_eccregion()` | per-OOB-layout | `Mtd::ooblayout_*` |
| `mtd_ooblayout_{get,set}_{ecc,data}bytes()` / `mtd_ooblayout_count_{free,ecc}bytes()` | per-OOB-bytes | `Mtd::ooblayout_bytes_*` |
| `mtd_wunit_to_pairing_info()` / `mtd_pairing_info_to_wunit()` / `mtd_pairing_groups()` | per-MLC-pairing | `Mtd::pairing_*` |
| `mtd_kmalloc_up_to()` | per-best-effort alloc | `Mtd::kmalloc_up_to` |
| `mtd_mmap_capabilities()` | per-NOMMU caps | `Mtd::mmap_capabilities` |
| `mtd_check_expert_analysis_mode()` | per-debug-gate | `Mtd::check_expert_analysis_mode` |
| `mtd_reboot_notifier()` | per-reboot-hook | `Mtd::reboot_notifier` |
| `mtd_otp_nvmem_add()` / `mtd_otp_nvmem_register()` | per-OTP-NVMEM | `Mtd::otp_nvmem_*` |
| `mtd_nvmem_add()` | per-cell-NVMEM | `Mtd::nvmem_add` |
| `init_mtd()` / `cleanup_mtd()` | per-module-init | `Mtd::init` / `exit` |

### compatibility contract

REQ-1: struct mtd_info (ABI-visible subset):
- type: u8 — MTD_ABSENT (0), MTD_RAM (1), MTD_ROM (2), MTD_NORFLASH (3), MTD_NANDFLASH (4), MTD_DATAFLASH (6), MTD_UBIVOLUME (7), MTD_MLCNANDFLASH (8).
- flags: u32 — MTD_WRITEABLE (0x400), MTD_BIT_WRITEABLE (0x800), MTD_NO_ERASE (0x1000), MTD_POWERUP_LOCK (0x2000), MTD_SLC_ON_MLC_EMULATION (0x4000).
- size: u64 — total device size in bytes.
- erasesize: u32 — erase-block size in bytes.
- writesize: u32 — minimum write granularity (NAND page, NOR word, etc.).
- writesize_shift / writesize_mask: u32 — fast div/mod (zero if writesize not pow-of-2).
- erasesize_shift / erasesize_mask: u32 — fast div/mod.
- oobsize: u32 — per-page OOB bytes (NAND).
- oobavail: u32 — OOB bytes available for user (after ECC).
- subpagesize: u32 — sub-page write granularity (= writesize unless NAND supports subpage).
- bitflip_threshold: u32 — corrected-bits count at which mtd_read returns EUCLEAN.
- ecc_strength: u32 — ECC bits correctable per ecc_step_size.
- ecc_step_size: u32 — bytes covered by one ECC codeword.
- ecc_stats: struct mtd_ecc_stats { corrected: u32, failed: u32, badblocks: u32, bbtblocks: u32 }.
- numeraseregions: i32 — count of struct mtd_erase_region_info[] for non-uniform NOR.
- name: *const u8 — display name.
- index: i32 — assigned by idr_alloc; appears as mtd<N>.
- refcnt: kref.
- dev: struct device — embedded.
- owner: *Module — driver owning this MTD.
- parent: *MtdInfo — partition parent (None for master).
- master: nested struct with partitions_lock + chrdev_lock + partitions list.
- partitions: list_head — child partitions.
- usecount: i32 — open count via __get_mtd_device.
- pairing: *const mtd_pairing_scheme — MLC pairing.
- ooblayout: *const mtd_ooblayout_ops — per-controller OOB layout.
- otp_user_nvmem / otp_factory_nvmem: *nvmem_device — OTP regions.
- reboot_notifier: notifier_block — wired if `_reboot` set.
- Per-op function pointers (master-only; partition calls go through mtd_get_master):
  - `_erase`, `_point`, `_unpoint`, `_get_unmapped_area`, `_read`, `_write`, `_panic_write`, `_read_oob`, `_write_oob`, `_get_fact_prot_info`, `_read_fact_prot_reg`, `_get_user_prot_info`, `_read_user_prot_reg`, `_write_user_prot_reg`, `_lock_user_prot_reg`, `_erase_user_prot_reg`, `_writev`, `_sync`, `_lock`, `_unlock`, `_is_locked`, `_block_isreserved`, `_block_isbad`, `_block_markbad`, `_max_bad_blocks`, `_suspend`, `_resume`, `_reboot`, `_get_device`, `_put_device`.

REQ-2: add_mtd_device(mtd):
- /* Pre-conditions */
- WARN_ONCE if mtd.dev.type set (already-registered double-call) → -EEXIST.
- BUG_ON(mtd.writesize == 0).
- WARN_ON if BOTH _write and _write_oob set, or BOTH _read and _read_oob set → -EINVAL (must be one).
- WARN_ON if !master->_erase && !(flags & MTD_NO_ERASE) → -EINVAL.
- MTD_SLC_ON_MLC_EMULATION: only on partitions of MLC master with pairing scheme; reject masters with _writev → -EINVAL.
- /* Lock + allocate index */
- mutex_lock(mtd_table_mutex).
- np = mtd_get_of_node(mtd); ofidx = of_alias_get_id(np, "mtd") or -1.
- if ofidx ≥ 0: i = idr_alloc(mtd_idr, mtd, ofidx, ofidx+1, GFP_KERNEL); else i = idr_alloc(mtd_idr, mtd, 0, 0, GFP_KERNEL).
- mtd.index = i; kref_init(&mtd.refcnt).
- /* Defaults */
- if bitflip_threshold == 0: bitflip_threshold = ecc_strength.
- SLC-on-MLC: erasesize /= ngroups; size /= ngroups.
- erasesize_shift = is_power_of_2(erasesize) ? ffs(erasesize)-1 : 0; mask = (1 << shift) - 1.
- writesize_shift / writesize_mask analogous.
- if MTD_WRITEABLE ∧ MTD_POWERUP_LOCK: mtd_unlock(mtd, 0, mtd.size) — ignore failures except EOPNOTSUPP-log.
- /* Register sysfs device */
- mtd.dev.type = &mtd_devtype; mtd.dev.class = &mtd_class; mtd.dev.devt = MTD_DEVT(i).
- dev_set_name(&mtd.dev, "mtd%d", i).
- dev_set_drvdata(&mtd.dev, mtd).
- of_node_get(np).
- device_register(&mtd.dev) — on error put_device + goto fail_added.
- /* NVMEM provider for partition cells */
- mtd_nvmem_add(mtd).
- mtd_debugfs_populate(mtd).
- /* Read-only mirror chardev */
- device_create(&mtd_class, mtd.dev.parent, MTD_DEVT(i)+1, NULL, "mtd%dro", i).
- /* Notifier fanout */
- list_for_each_entry(not, &mtd_notifiers, list): not.add(mtd).
- mutex_unlock(mtd_table_mutex).
- /* DT linux,rootfs */
- if of_property_read_bool(np, "linux,rootfs") ∧ IS_BUILTIN(CONFIG_MTD): ROOT_DEV = MKDEV(MTD_BLOCK_MAJOR, mtd.index).
- __module_get(THIS_MODULE).
- return 0.

REQ-3: del_mtd_device(mtd):
- mutex_lock(mtd_table_mutex).
- if idr_find(mtd_idr, mtd.index) != mtd → -ENODEV.
- list_for_each_entry(not, &mtd_notifiers, list): not.remove(mtd).
- kref_put(&mtd.refcnt, mtd_device_release).
- mutex_unlock(mtd_table_mutex).
- mtd_device_release: idr_remove, device_unregister, of_node_put, module_put(THIS_MODULE).

REQ-4: mtd_device_parse_register(mtd, types, parser_data, parts, nr_parts):
- mtd_set_dev_defaults(mtd) — inherit owner + name from dev.parent if absent; INIT_LIST_HEAD(partitions); init partitions_lock + chrdev_lock.
- mtd_otp_nvmem_add(mtd) — register factory + user OTP NVMEM providers.
- if CONFIG_MTD_PARTITIONED_MASTER: add_mtd_device(mtd) for master first.
- if CONFIG_MTD_VIRT_CONCAT: mtd_virt_concat_node_create.
- parse_mtd_partitions(mtd, types, parser_data) — try parsers in `types[]` (cmdlinepart, ofpart, redboot, ...). EPROBE_DEFER propagates.
- if parsed > 0: ok; else if nr_parts: add_mtd_partitions(mtd, parts, nr_parts) — driver-provided fallback; else if !device_is_registered(&mtd.dev): add_mtd_device(mtd) — register master with no partitions.
- if mtd._reboot ∧ !mtd.reboot_notifier.notifier_call: mtd.reboot_notifier.notifier_call = mtd_reboot_notifier; register_reboot_notifier(&mtd.reboot_notifier).
- Error path: nvmem_unregister(otp_*_nvmem); if registered: del_mtd_device.

REQ-5: mtd_device_unregister(master):
- if master._reboot: unregister_reboot_notifier; memset reboot_notifier 0.
- nvmem_unregister(master.otp_user_nvmem); nvmem_unregister(master.otp_factory_nvmem).
- if CONFIG_MTD_VIRT_CONCAT: mtd_virt_concat_destroy(master).
- del_mtd_partitions(master).
- if !device_is_registered(&master.dev): return 0.
- return del_mtd_device(master).

REQ-6: get_mtd_device(mtd, num) / __get_mtd_device(mtd):
- get_mtd_device: mutex_lock(table); if num == -1: verify mtd present; else: idr_find(mtd_idr, num). If mtd≠NULL ∧ mtd≠ret: ret=NULL. __get_mtd_device(ret); mutex_unlock.
- __get_mtd_device: master = mtd_get_master(mtd). If master._get_device: master._get_device(mtd) (-EBUSY if driver refuses). try_module_get(master.owner) → -ENODEV if fails. Walk parent chain: each non-master node kref_get(&mtd.refcnt). If CONFIG_MTD_PARTITIONED_MASTER: kref_get(&master.refcnt).

REQ-7: put_mtd_device / __put_mtd_device:
- put_mtd_device: mutex_lock(table); __put_mtd_device; mutex_unlock.
- __put_mtd_device: walk parent chain: each non-master kref_put(&mtd.refcnt, mtd_device_release). If CONFIG_MTD_PARTITIONED_MASTER: kref_put(&master.refcnt, ...). module_put(master.owner). Last: master._put_device(master).

REQ-8: register_mtd_user(new) / unregister_mtd_user(old):
- register_mtd_user: mutex_lock(table); list_add(&new.list, &mtd_notifiers); __module_get(THIS_MODULE); mtd_for_each_device(mtd): new.add(mtd); mutex_unlock.
- unregister_mtd_user: mutex_lock(table); module_put(THIS_MODULE); list_del(&old.list); mtd_for_each_device(mtd): old.remove(mtd); mutex_unlock.

REQ-9: get_mtd_device_nm(name) / of_get_mtd_device_by_node(np):
- get_mtd_device_nm: scan mtd_idr for strcmp(name, mtd.name) match; __get_mtd_device on hit; ERR_PTR(-ENODEV) on miss.
- of_get_mtd_device_by_node: scan for mtd_get_of_node(mtd) == np; ERR_PTR(-EPROBE_DEFER) on miss.

REQ-10: mtd_erase(mtd, instr):
- master = mtd_get_master(mtd); mst_ofs = mtd_get_master_ofs(mtd, 0).
- instr.fail_addr = MTD_FAIL_ADDR_UNKNOWN.
- if !mtd.erasesize ∨ !master._erase → -ENOTSUPP.
- if instr.addr ≥ mtd.size ∨ instr.len > mtd.size - instr.addr → -EINVAL.
- if !(mtd.flags & MTD_WRITEABLE) → -EROFS.
- if !instr.len: return 0.
- ledtrig_mtd_activity().
- SLC-on-MLC: rescale addr/len by erasesize ratio.
- adjinstr.addr += mst_ofs.
- ret = master._erase(master, &adjinstr).
- Translate adjinstr.fail_addr back to partition-local; SLC-on-MLC: divide by master.erasesize, multiply by mtd.erasesize.

REQ-11: mtd_read(mtd, from, len, retlen, buf) / mtd_write(mtd, to, len, retlen, buf):
- Wrap call: build struct mtd_oob_ops { len, datbuf=buf }; call mtd_read_oob / mtd_write_oob; *retlen = ops.retlen.
- mtd_read: WARN_ON_ONCE if *retlen != len ∧ mtd_is_bitflip_or_eccerr(ret) (short-read on non-ECC-error is a driver bug).

REQ-12: mtd_read_oob / mtd_write_oob:
- mtd_check_oob_ops: clamp NULL datbuf len to 0; check offs alignment + oob bounds.
- master = mtd_get_master(mtd).
- ledtrig_mtd_activity().
- SLC-on-MLC: mtd_io_emulated_slc translates page addresses through pairing scheme.
- Otherwise: master._read_oob / _write_oob with translated offset (mtd_read_oob_std / mtd_write_oob_std fall back to plain _read / _write if _read_oob / _write_oob NULL).
- Update ecc_stats via mtd_update_ecc_stats; propagate corrected/failed counters up parent chain.

REQ-13: mtd_panic_write(mtd, to, len, retlen, buf):
- If !master._panic_write → -EOPNOTSUPP.
- Bounds + writeable check.
- master.oops_panic_write = true (sticky flag).
- master._panic_write(master, mtd_get_master_ofs(mtd, to), len, retlen, buf).
- Caller (mtdoops) is allowed to break locks and busy-wait.

REQ-14: mtd_point / mtd_unpoint / mtd_get_unmapped_area:
- mtd_point: -EOPNOTSUPP if !master._point; bounds check; from = mtd_get_master_ofs(mtd, from); master._point(master, from, len, retlen, virt, phys).
- mtd_unpoint: -EOPNOTSUPP if !master._unpoint; bounds check; master._unpoint(...).
- mtd_get_unmapped_area: mtd_point(mtd, offset, len, &retlen, &virt, NULL); if retlen != len: mtd_unpoint + -ENOSYS; else return (unsigned long)virt.

REQ-15: mtd_lock / mtd_unlock / mtd_is_locked (NOR):
- master = mtd_get_master(mtd).
- If !master._lock / _unlock / _is_locked → -EOPNOTSUPP.
- Bounds check on (ofs, len) vs mtd.size.
- Translate ofs via mtd_get_master_ofs.
- master._lock / _unlock / _is_locked(master, ofs, len).

REQ-16: mtd_block_isreserved / _isbad / _markbad (NAND BBT):
- _isreserved: returns 0 if !master._block_isreserved.
- _isbad: returns 0 if !master._block_isbad.
- _markbad: -EOPNOTSUPP if !master._block_markbad; -EROFS if !MTD_WRITEABLE. If already-bad (master._block_isbad returns >0): return 0 (idempotent). Else master._block_markbad, then walk parents incrementing ecc_stats.badblocks.

REQ-17: OTP (One-Time Programmable) ops:
- _get_fact_prot_info / _read_fact_prot_reg: factory-burned OTP (read-only).
- _get_user_prot_info / _read_user_prot_reg / _write_user_prot_reg / _lock_user_prot_reg / _erase_user_prot_reg: user OTP (write-once-then-lock).
- mtd_otp_nvmem_add registers factory + user OTP as NVMEM providers with reg_read = mtd_nvmem_fact_otp_reg_read / mtd_nvmem_user_otp_reg_read.

REQ-18: MLC pairing (mtd_wunit_to_pairing_info / mtd_pairing_info_to_wunit / mtd_pairing_groups):
- For MLC NAND with `pairing` scheme: mtd_wunit_to_pairing_info(mtd, wunit, info): info.group/pair. mtd_pairing_info_to_wunit: inverse. mtd_pairing_groups: count of groups (default 1).

REQ-19: ECC stats:
- mtd_update_ecc_stats(mtd, master, old): diff = master.ecc_stats - old; walk mtd.parent chain accumulating diff.failed + diff.corrected.

REQ-20: sysfs attributes (under /sys/class/mtd/mtd<N>/):
- RO: type, flags, size, erasesize, writesize, subpagesize, oobsize, oobavail, numeraseregions, name, ecc_strength, ecc_step_size, corrected_bits, ecc_failures, bad_blocks, bbt_blocks.
- RW: bitflip_threshold (0..ecc_strength).

REQ-21: chardev:
- /dev/mtd<N> (RW, MTD_DEVT(i) = MKDEV(MTD_CHAR_MAJOR, i*2)).
- /dev/mtd<N>ro (RO mirror, MTD_DEVT(i)+1 = MKDEV(MTD_CHAR_MAJOR, i*2+1)).
- IOCTLs in mtdchar.c (see drivers/mtd/chardev-block.md).

REQ-22: Initialization (init_mtd):
- class_register(&mtd_class).
- mtd_bdi = bdi_alloc + bdi_register("mtd-0"); ra_pages = 0, io_pages = 0 (no readahead on flash).
- proc_create_single("mtd", 0, NULL, mtd_proc_show) — /proc/mtd.
- init_mtdchar().
- debugfs_create_dir("mtd", NULL); debugfs_create_bool("expert_analysis_mode", 0600, &mtd_expert_analysis_mode).

REQ-23: Reboot notifier:
- if mtd._reboot: register_reboot_notifier on registration; called with state ∈ {SYS_RESTART, SYS_HALT, SYS_POWER_OFF}; invokes mtd._reboot(mtd) to quiesce flash (e.g., exit deep-power-down).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mtd_idr_unique_index` | INVARIANT | per-add_mtd_device: idr_alloc returns unique index ∈ [0, INT_MAX]. |
| `mtd_refcnt_balanced` | INVARIANT | per-__get_mtd_device + __put_mtd_device: each kref_get matched by kref_put on every parent. |
| `mtd_table_mutex_held_for_register` | INVARIANT | per-add/del/notifier: mtd_table_mutex held. |
| `master_only_op_fnptrs_called` | INVARIANT | per-mtd_*: partition fwd to mtd_get_master(mtd) before calling op fnptr. |
| `erase_addr_in_bounds` | INVARIANT | per-mtd_erase: instr.addr + instr.len ≤ mtd.size. |
| `write_requires_writeable_flag` | INVARIANT | per-mtd_write / mtd_erase / mtd_block_markbad: !MTD_WRITEABLE ⟹ -EROFS. |
| `read_write_xor_oob_variants` | INVARIANT | per-add_mtd_device: not both _write and _write_oob; not both _read and _read_oob. |
| `markbad_idempotent` | INVARIANT | per-mtd_block_markbad: already-bad returns 0 (no double-write). |
| `panic_write_no_lock` | INVARIANT | per-mtd_panic_write: callable from atomic + may break locks. |
| `notifier_fanout_under_lock` | INVARIANT | per-register_mtd_user: add called for every existing MTD under table_mutex. |

### Layer 2: TLA+

`drivers/mtd/mtdcore.tla`:
- Per-register + per-unregister + per-notifier + per-get/put-handle + per-io.
- Properties:
  - `safety_index_uniqueness` — per-IDR: no two MTDs share an index at any point.
  - `safety_refcount_no_underflow` — per-kref: get always precedes put; final put triggers release exactly once.
  - `safety_master_partition_consistency` — per-partition op: master = mtd_get_master(partition); master.partitions contains partition.
  - `safety_writeable_invariant` — per-erase/write: precondition MTD_WRITEABLE.
  - `safety_notifier_visibility` — per-register_mtd_user: notifier sees exactly the set of MTDs registered between its add and remove.
  - `liveness_register_completes` — per-add_mtd_device: terminates with success or error (no hang).
  - `liveness_unregister_drains_users` — per-del_mtd_device: all notifier.remove run before kref_put.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mtd::add_device` post: success ⟹ device_register'd ∧ MTD_IDR.find(index) == Some(mtd) ∧ kref_init'd | `Mtd::add_device` |
| `Mtd::del_device` post: success ⟹ device_unregister scheduled (via release) ∧ MTD_IDR.find returns None after release | `Mtd::del_device` |
| `Mtd::get_device_locked` post: success ⟹ try_module_get'd ∧ kref_get'd on every parent including master if CONFIG_MTD_PARTITIONED_MASTER | `Mtd::get_device_locked` |
| `Mtd::put_device_locked` post: kref_put on every parent ∧ module_put ∧ if last: _put_device(master) | `Mtd::put_device_locked` |
| `Mtd::erase` post: ok ⟹ master._erase called with translated addr; err ⟹ original instr.fail_addr translated back | `Mtd::erase` |
| `Mtd::read_oob` post: ecc_stats delta propagated up parent chain via update_ecc_stats | `Mtd::read_oob` |
| `Mtd::block_markbad` post: already-bad ⟹ Ok(()); else master._block_markbad called + parent badblocks++ | `Mtd::block_markbad` |

### Layer 4: Verus/Creusot functional

`Per-register → IDR alloc → sysfs device_register → notifier fanout → /dev/mtd<N>{,ro} appear → /sys/class/mtd/mtd<N>/{type,size,erasesize,...} populated → mtdblock + UBI + JFFS2 see add notifier` semantic equivalence: per-Documentation/driver-api/mtdnand.rst + Documentation/devicetree/bindings/mtd/.

`Per-erase / read / write → bounds check → ledtrig_mtd_activity → SLC-on-MLC translation → master fn-ptr → fail_addr translation` semantic equivalence: per-mtd-utils + flashcp + flash_eraseall behavioral traces.

### hardening

(Inherits row-1 features from `drivers/mtd/00-overview.md` § Hardening.)

MTD-core reinforcement:

- **Per-mtd_table_mutex strict for all register/unregister/notifier** — defense against per-add-during-iter races.
- **Per-kref_init in add_mtd_device + kref_put_on release** — defense against per-stale-handle UAF on unregister.
- **Per-mtd_get_master indirection on every op** — defense against per-partition-dispatching-to-itself recursion.
- **Per-MTD_WRITEABLE gate on erase/write/markbad** — defense against per-write-to-RO-NOR brick.
- **Per-MTD_NO_ERASE gate skips _erase requirement** — defense against per-DataFlash/RAM mis-config rejection.
- **Per-bounds checking (addr + len vs mtd.size)** — defense against per-OOB I/O hitting another partition.
- **Per-SLC_ON_MLC_EMULATION partition-only + master-MLC + pairing-required** — defense against per-config-error data corruption.
- **Per-read/write XOR _read_oob/_write_oob hook set** — defense against per-double-dispatch ambiguity.
- **Per-master.oops_panic_write sticky** — defense against per-mtdoops-callback-recursion deadlock.
- **Per-bitflip_threshold ≤ ecc_strength** — defense against per-EUCLEAN spam masking real failures.
- **Per-ledtrig_mtd_activity rate-limited** — defense against per-LED-storm on bulk erase.
- **Per-mtd_block_markbad idempotent** — defense against per-double-mark wearing oob.
- **Per-notifier fanout under table_mutex** — defense against per-UBI/JFFS2-seeing-half-registered MTD.

