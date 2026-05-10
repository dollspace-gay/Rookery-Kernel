# Tier-2: security/keys — kernel keyring (key + keyring + keyctl + per-keytype + encrypted-keys + trusted-keys + DH/ECDH/RSA/ECDSA)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - security/keys/
  - include/linux/key.h
  - include/uapi/linux/keyctl.h
-->

## Summary

Tier-2 wrapper for the kernel keyring subsystem — central key/secret store underneath kerberos credentials, NFSv4 idmapper, fs-crypt master keys, dm-crypt secrets, IMA appraisal keys, system trust-store, hardware-TPM-backed keys, asymmetric DH/ECDH/RSA/ECDSA keys for IPsec PKI + module-signing + IMA-signing. Components: **core** (`key.c` + `keyring.c` + `keyctl.c` + `keyctl_pkey.c` + `permission.c` + `compat.c` + `compat_dh.c` + `process_keys.c` + `request_key.c` + `request_key_auth.c` + `gc.c` + `proc.c` + `sysctl.c` + `internal.h`: keyring lifecycle + per-key permissions + lookup + GC), **per-keytype**: `user.c` (user-keytype), `keyring.c` (keyring-keytype), `big_key.c` (large-payload key with TMPFS-backing), `encrypted-keys/` (encrypted-key-type backed by encryption-key for sealed storage), `trusted-keys/` + `trusted_*.c` (TPM/CAAM/TEE-sealed keys), `dh.c` (Diffie-Hellman keys), `persistent.c` (persistent across exec/uid-change). **Crypto integration**: `keyctl_pkey.c` provides ASYMMETRIC keytype `KEYCTL_PKEY_*` ops (sign/verify/encrypt/decrypt) backed by the asymmetric crypto subsystem.

## Compatibility contract — outline

- `keyctl(2)` syscall #250 + `add_key(2)` #248 + `request_key(2)` #249 byte-identical UAPI.
- `KEYCTL_*` operations (~30): `_GET_KEYRING_ID`, `_JOIN_SESSION_KEYRING`, `_UPDATE`, `_REVOKE`, `_CHOWN`, `_SETPERM`, `_DESCRIBE`, `_CLEAR`, `_LINK`, `_UNLINK`, `_SEARCH`, `_READ`, `_INSTANTIATE`, `_NEGATE`, `_SET_REQKEY_KEYRING`, `_SET_TIMEOUT`, `_ASSUME_AUTHORITY`, `_GET_SECURITY`, `_SESSION_TO_PARENT`, `_REJECT`, `_INSTANTIATE_IOV`, `_INVALIDATE`, `_GET_PERSISTENT`, `_DH_COMPUTE`, `_PKEY_QUERY`, `_PKEY_ENCRYPT`, `_PKEY_DECRYPT`, `_PKEY_SIGN`, `_PKEY_VERIFY`, `_RESTRICT_KEYRING`, `_MOVE`, `_CAPABILITIES`, `_WATCH_KEY`. Wire format byte-identical (keyctl + libkeyutils consume unchanged).
- Per-keytype payload formats byte-identical: user-key arbitrary bytes, encrypted-key sealed format with master+key-format, trusted-key TPM-sealed format, asymmetric-key X.509-DER or PGP.
- `/proc/keys` byte-identical format (per-line: id flags timeout perm uid gid type:desc).
- `/proc/key-users` byte-identical (per-uid usage stats).
- `/proc/sys/kernel/keys/{root_maxkeys,root_maxbytes,maxkeys,maxbytes,gc_delay,persistent_keyring_expiry}` byte-identical.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `security/keys/key-core.md` | `key.c` + `internal.h`: per-key lifecycle |
| `security/keys/keyring.md` | `keyring.c`: keyring-as-key-type + keyring search |
| `security/keys/keyctl.md` | `keyctl.c` + `compat.c` + `keyctl_pkey.c`: keyctl(2) syscall + asymmetric ops |
| `security/keys/process-request.md` | `process_keys.c` + `request_key.c` + `request_key_auth.c`: per-task keyrings + request-key upcall |
| `security/keys/permission.md` | `permission.c`: per-key permissions |
| `security/keys/gc.md` | `gc.c`: per-key timeout + collection |
| `security/keys/sysctl-proc.md` | `sysctl.c` + `proc.c`: `/proc/keys` + sysctls |
| `security/keys/user-keytype.md` | `user.c`: user-keytype |
| `security/keys/big-key.md` | `big_key.c`: large-payload key |
| `security/keys/encrypted-keys.md` | `encrypted-keys/`: encrypted-key-type |
| `security/keys/trusted-keys.md` | `trusted-keys/` + `trusted_*.c`: TPM/CAAM/TEE-sealed keys |
| `security/keys/dh.md` | `dh.c` + `compat_dh.c`: Diffie-Hellman key compute |
| `security/keys/persistent.md` | `persistent.c`: persistent keyring across exec/uid |

## Compatibility outline

- REQ-O1: All key syscalls + KEYCTL_* ops + UAPI byte-identical (libkeyutils + keyutils + krb5-libs + nfs-utils consume unchanged).
- REQ-O2: Per-keytype payload formats byte-identical.
- REQ-O3: `/proc/keys` + `/proc/key-users` + sysctl knobs byte-identical.
- REQ-O4: TPM-trusted keys + encrypted-keys workflows identical.
- REQ-O5: ASYMMETRIC keytype + KEYCTL_PKEY_* ops integrate with asymmetric crypto.
- REQ-O6: TLA+ models (per-key refcount + concurrent timeout + invalidate; keyring search + concurrent link/unlink; trust-key TPM-seal/unseal serialization).
- REQ-O7: Hardening: per-process keyring isolation; sealed-key never exposed to userspace post-install; KEYCTL_RESTRICT_KEYRING enforced.

## Acceptance Criteria

- [ ] AC-O1: `keyctl list @s` + `keyctl new_session` + `keyctl add user foo bar @s` round-trip works.
- [ ] AC-O2: Kerberos `kinit` populates `@s` keyring with TGT.
- [ ] AC-O3: `keyctl add encrypted enc1 "new user:masterkey 32" @u` creates encrypted-key.
- [ ] AC-O4: TPM-trusted-key: `keyctl add trusted ktrust "new 32" @u` creates TPM-sealed key.
- [ ] AC-O5: Asymmetric key: `cat ca.x509 | keyctl padd asymmetric "" @u` loads X.509.
- [ ] AC-O6: kselftest `tools/testing/selftests/keys/` passes.

## Verification

| TLA+ Model | Owner |
|---|---|
| `models/keys/refcount_timeout.tla` | `security/keys/key-core.md` (proves: per-key refcount + concurrent timeout-fire + invalidate; never freed while still ref'd; never double-collected) |
| `models/keys/keyring_search.tla` | `security/keys/keyring.md` (proves: keyring search + concurrent link/unlink; search returns either pre- or post-link state, never partial; cycle detection prevents infinite recursion) |
| `models/keys/trust_seal.tla` | `security/keys/trusted-keys.md` (proves: TPM-trusted-key seal/unseal serialized via TPM session lock; concurrent seal+unseal never produce wrong key on unseal) |

## Hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-key + per-keyring refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-keytype `key_type` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-key payload-len + per-keyring entry-count arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed key payload zeroed (carries cryptographic secrets) | § Default-on configurable |

Keys-specific reinforcement: every freed key payload zeroed before return-to-allocator (mandatory, not configurable for cryptographic keys); KEYCTL_RESTRICT_KEYRING enforced (defense against rogue cert injection); persistent keyring expiry capped (defense against stale credentials); TPM/TEE seal-unseal serialized via TPM-LSM hook; per-uid keyring quota enforced (defense against keyring-flood DoS).

## Open Questions
(none at Tier-2)

## Out of Scope
- Implementation code
- 32-bit-only paths
- userspace keyutils (separate)
