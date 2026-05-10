---
title: "Tier-2: security/integrity — IMA + EVM + digsig + EFI Secure Boot integration"
tags: ["tier-2", "security", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for the kernel integrity subsystem — measure + appraise + protect file content + xattrs at runtime. Components: **IMA** (`ima/` — Integrity Measurement Architecture: per-file content hash measured at access; appended to TPM PCR for remote attestation; appraisal-mode rejects modified files; per-file `security.ima` xattr stores signed digest; per-template measurement record), **EVM** (`evm/` — Extended Verification Module: per-file `security.evm` xattr HMAC-protects all other security.* xattrs against offline tampering), **digsig** (`digsig.c` + `digsig_asymmetric.c`: asymmetric signature verification used by IMA and module-signing), **EFI Secure Boot** (`efi_secureboot.c`: detect SecureBoot mode at boot, lock down kernel modes per `kernel_lockdown` LSM), **iint** (`iint.c`: per-inode integrity-info cache), **integrity_audit** (`integrity_audit.c`: audit messages for IMA/EVM events).

### Acceptance Criteria

- [ ] AC-O1: IMA policy applied at boot; `cat /sys/kernel/security/ima/ascii_runtime_measurements` shows measurements for all execve'd binaries.
- [ ] AC-O2: `tpm2_pcrread sha1:10` shows PCR 10 extended with measurement digest.
- [ ] AC-O3: IMA appraise mode rejects modified binary: modify `/usr/bin/ls`, exec returns -EACCES.
- [ ] AC-O4: EVM xattr verify: tampering with `security.selinux` xattr detected via `security.evm` mismatch on next access.
- [ ] AC-O5: SecureBoot test: boot with SB=on, `/sys/kernel/security/lockdown` shows confidentiality mode.
- [ ] AC-O6: kselftest `tools/testing/selftests/integrity/` passes.

### Out of Scope

- Implementation code
- 32-bit-only paths
- ima-evm-utils userspace

### compatibility contract — outline

- IMA policy file `/sys/kernel/security/ima/policy` (write-once at boot, then re-load via `ima_policy_update=1`) byte-identical syntax: `measure`, `dont_measure`, `appraise`, `dont_appraise`, `audit`, `hash`, `dont_hash` rules with `func=`, `mask=`, `fsmagic=`, `fsuuid=`, `fsname=`, `obj_user=`, `obj_role=`, `obj_type=`, `subj_user=`, `subj_role=`, `subj_type=`, `appraise_type=`, `appraise_flag=`, `keyrings=`, `template=`, `permit_directio` qualifiers.
- `/sys/kernel/security/ima/{ascii_runtime_measurements, binary_runtime_measurements, runtime_measurements_count, violations}` byte-identical (consumed by ima-evm-utils + tpm2-tools).
- `/sys/kernel/security/integrity/evm` byte-identical.
- xattr `security.ima` (signed file digest) + `security.evm` (HMAC over security.* xattrs) byte-identical.
- IMA appraisal modes (none / fix / log / enforce) cmdline `ima_appraise=` byte-identical.
- IMA template (`ima-ng`, `ima-sig`, `ima-modsig`, `ima-buf`, custom) byte-identical wire format for measurement records.
- TPM PCR extension (default PCR 10, configurable) byte-identical.

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `security/integrity/iint.md` | `iint.c`: per-inode integrity-info cache |
| `security/integrity/digsig.md` | `digsig.c` + `digsig_asymmetric.c`: signature verification |
| `security/integrity/efi-secureboot.md` | `efi_secureboot.c`: EFI SecureBoot detection + kernel_lockdown |
| `security/integrity/integrity-audit.md` | `integrity_audit.c`: audit messages |
| `security/integrity/ima-main.md` | `ima/ima_main.c` + `ima/ima_init.c` + `ima/ima_policy.c`: IMA core + policy |
| `security/integrity/ima-template.md` | `ima/ima_template.c` + `ima/ima_template_lib.c`: measurement template |
| `security/integrity/ima-appraise.md` | `ima/ima_appraise.c`: appraisal mode |
| `security/integrity/ima-modsig.md` | `ima/ima_modsig.c`: appended signature for module-style files |
| `security/integrity/ima-asymmetric.md` | `ima/ima_asymmetric_keys.c`: keyring backing IMA appraisal |
| `security/integrity/ima-fs.md` | `ima/ima_fs.c`: `/sys/kernel/security/ima/` UAPI |
| `security/integrity/ima-queue.md` | `ima/ima_queue.c` + `ima/ima_queue_keys.c`: deferred measurement queue |
| `security/integrity/ima-tpm.md` | `ima/ima_crypto.c` + TPM PCR extension |
| `security/integrity/evm-main.md` | `evm/evm_main.c` + `evm/evm_crypto.c` + `evm/evm_secfs.c`: EVM core |

### compatibility outline

- REQ-O1: IMA policy file syntax + `/sys/kernel/security/ima/*` UAPI byte-identical (ima-evm-utils + ima-policy-tools consume unchanged).
- REQ-O2: xattr `security.ima` + `security.evm` byte-identical.
- REQ-O3: IMA template wire format byte-identical (remote-attestation tools consume unchanged).
- REQ-O4: IMA appraisal modes parse + behave identically.
- REQ-O5: TPM PCR 10 (default) extension byte-identical.
- REQ-O6: EFI SecureBoot detection + kernel_lockdown auto-engage in `confidentiality` mode if SecureBoot=on.
- REQ-O7: TLA+ models (per-inode iint cache concurrency, IMA queue deferred-measurement order, EVM HMAC-update under concurrent xattr-set).
- REQ-O8: Hardening: signed-only mode default-on for distros with SecureBoot.

### verification

| TLA+ Model | Owner |
|---|---|
| `models/integrity/iint_cache.tla` | `security/integrity/iint.md` (proves: per-inode iint allocate + concurrent file-modify + iint-flush; cache stays consistent with on-disk state) |
| `models/integrity/ima_queue.tla` | `security/integrity/ima-queue.md` (proves: deferred-measurement queue ordering; PCR extend in measurement-record-creation order; concurrent extend never reorders) |
| `models/integrity/evm_hmac_update.tla` | `security/integrity/evm-main.md` (proves: EVM HMAC over security.* xattrs updated atomically on xattr-set; concurrent xattr-set + read never sees torn HMAC) |

### hardening

(integrity is itself the hardening subsystem; row-1 features apply equally.)

| Feature | Default |
|---|---|
| **REFCOUNT** | per-iint refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-template `ima_template_field` tables `static const` | § Mandatory |
| **SIZE_OVERFLOW** | template buffer + measurement-list arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed iint + EVM HMAC state cleared (carries cryptographic state) | § Default-on configurable |

Integrity-specific reinforcement: IMA policy load CAP_MAC_ADMIN; built-in IMA + EVM keys (Sforshee + per-distro CA) loaded into `.ima` and `.evm` keyrings at boot from MOKList + DB; runtime keyring add gated on SecureBoot; appraisal in `enforce` mode default-on for distros with SecureBoot+IMA setup; EVM-protected xattrs include `security.{ima,selinux,smack,apparmor,capability}`; rotation of master keys logged + audited.

