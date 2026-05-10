---
title: "Tier-2: drivers/target — LIO target framework (TCM core + iSCSI target + FC + iSER + SRP target + tcm_user + tcm_qla2xxx + vhost-scsi-target + sbp + loop)"
tags: ["tier-2", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for the LIO (Linux-IO) target framework — kernel-side SCSI/iSCSI/FC/SRP/iSER target backbone used by every NAS/SAN appliance running Linux (TrueNAS Core/Scale, openSUSE-target, ESOS), every cloud-storage gateway, every kernel-side iSCSI/FC/iSER target, every SCSI passthrough setup. Components: **TCM core** (`target_core_*.c`: target_core_alua, target_core_configfs, target_core_device, target_core_fabric_configfs, target_core_fabric_lib, target_core_file, target_core_hba, target_core_iblock, target_core_internal.h, target_core_mod, target_core_pr, target_core_pscsi, target_core_rd, target_core_sbc, target_core_spc, target_core_stat, target_core_tmr, target_core_tpg, target_core_transport, target_core_ua, target_core_user, target_core_xcopy: SCSI command processing + ALUA + per-target-portal-group + persistent-reservations + SBC/SPC + transport + user-mode), **per-fabric drivers**: `iscsi/` (iSCSI target — `iscsi_target_*.c`), `loopback/` (kernel-side loopback transport), `tcm_fc/` (NVMe-style FC — wait, this is FCP, not NVMe — pairs with libfc), `tcm_remote/`, `sbp/` (FireWire SBP-2 target — legacy). Plus iSER target (`drivers/infiniband/ulp/isert/` — separate Tier-2), vhost-scsi target (`drivers/vhost/scsi.c` — separate Tier-2), TCM-Qla2xxx (`drivers/scsi/qla2xxx/tcm_qla2xxx.c` — pairs with FC LLD).

### compatibility contract — outline

- Configfs at `/sys/kernel/config/target/{core/<backend>/<name>, iscsi/<iqn>, loopback/<n>, fc/<wwn>, vhost/<dev>, sbp/<n>}/{lun/, acls/, ...}` byte-identical (targetcli + tcm-pr-tools + rtsadmin consume unchanged).
- TCM backends: iblock (block-device-backed), file (sparse-file-backed), pscsi (passthrough to a host-side SCSI device), rd (ramdisk), user (TCM user — userspace plugin via `/dev/tcmu-N`).
- Per-fabric LUN export via configfs identical.
- Persistent-reservation handling per SPC-3/4 (cross-ref `block/00-overview.md` pr_ops).

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `drivers/target/tcm-core.md` | `target_core_{configfs,device,hba,internal,mod,transport}.c`: framework |
| `drivers/target/tcm-spc-sbc.md` | `target_core_{spc,sbc}.c`: SPC + SBC command sets |
| `drivers/target/tcm-pr-alua.md` | `target_core_{pr,alua,ua,tmr}.c`: persistent-reservations + ALUA + unit-attentions + task-mgmt |
| `drivers/target/tcm-tpg.md` | `target_core_{tpg,fabric_configfs,fabric_lib}.c`: per-target-portal-group |
| `drivers/target/tcm-backends.md` | `target_core_{iblock,file,pscsi,rd,user,xcopy}.c`: backends |
| `drivers/target/tcm-stat.md` | `target_core_stat.c` |
| `drivers/target/iscsi-target.md` | `iscsi/`: iSCSI target |
| `drivers/target/loopback.md` | `loopback/`: kernel-side loop transport |
| `drivers/target/tcm-fc.md` | `tcm_fc/`: FC target via libfc |
| `drivers/target/tcm-remote-sbp.md` | `tcm_remote/` + `sbp/`: remote + FireWire SBP-2 |

### compatibility outline / ac / verification / hardening

- REQ-O1: Configfs byte-identical (targetcli consumes unchanged).
- REQ-O2: All TCM backends + per-fabric drivers source-compat.
- REQ-O3: SPC/SBC command processing semantics identical.
- REQ-O4: TLA+ models (per-target ALUA path-state convergence; persistent-reservation key-handling under concurrent acquire/release; per-fabric session lifecycle).
- REQ-O5: AC: targetcli configures iSCSI LUN; remote initiator (open-iscsi) connects + IO works; tcmu userspace backend plugin works.
- Hardening: configfs writes require CAP_SYS_ADMIN; per-target LSM mediation; iSCSI CHAP secrets keyring-shielded.

