# Tier-2: security/apparmor — AppArmor LSM (path-based MAC: profiles + policy + apparmorfs + DFA matching)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - security/apparmor/
-->

## Summary

Tier-2 wrapper for AppArmor — path-based MAC LSM (vs SELinux's type-enforcement). Default LSM in Ubuntu/Debian + SUSE since 12.04 / 12. Components: **lsm hook impls** (`lsm.c` — every `security_*` LSM hook impl), **policy + profile** (`policy.c` + `policy_unpack.c` + `policy_unpack_test.c` + `policy_ns.c` + `domain.c`: profile + namespace + cross-profile transitions), **DFA match engine** (`match.c`: Aho-Corasick / table-based DFA for path matching), **af_unix + ipc + file + capability** (`af_unix.c` + `ipc.c` + `file.c` + `capability.c` + `mount.c` + `net.c` + `task.c`: per-class hook impls), **label + secid** (`label.c` + `secid.c`: per-task label + sec-id), **apparmorfs** (`apparmorfs.c`: `/sys/kernel/security/apparmor/` UAPI), **audit** (`audit.c`: per-decision audit record), **crypto** (`crypto.c`: SHA1/256 hash for profile-load integrity), **resource + rlimit** (`resource.c`: per-profile rlimit), **lib + match-bin** (`lib.c` + `match.c`).

## Compatibility contract — outline

- `/sys/kernel/security/apparmor/` UAPI byte-identical: `profiles` (list active profiles), `.load`/`.replace`/`.remove` (profile mgmt), `.access` (per-profile access query), `policy/` subdir, `features/` capability-bits, `policy/` profile-state inspection. apparmor-utils + aa-* tools + apparmor_parser consume unchanged.
- Profile binary blob format byte-identical (apparmor_parser produces blobs that load).
- Cmdline `apparmor={0,1}` parsed identically.
- Audit messages prefixed `audit: type=APPARMOR ...` byte-identical.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `security/apparmor/lsm.md` | `lsm.c`: LSM hook impl |
| `security/apparmor/policy.md` | `policy.c` + `policy_unpack.c` + `policy_ns.c`: profile + namespace mgmt |
| `security/apparmor/domain.md` | `domain.c`: cross-profile transitions |
| `security/apparmor/match.md` | `match.c`: DFA match engine |
| `security/apparmor/file-mount-net-ipc-cap.md` | `file.c` + `mount.c` + `net.c` + `ipc.c` + `capability.c` + `task.c` + `af_unix.c`: per-class hook impls |
| `security/apparmor/label-secid.md` | `label.c` + `secid.c` |
| `security/apparmor/apparmorfs.md` | `apparmorfs.c`: UAPI |
| `security/apparmor/audit.md` | `audit.c` |
| `security/apparmor/crypto.md` | `crypto.c`: profile hash |
| `security/apparmor/resource.md` | `resource.c`: per-profile rlimit |

## Compatibility outline

- REQ-O1: `/sys/kernel/security/apparmor/` UAPI byte-identical.
- REQ-O2: Profile binary blob format byte-identical.
- REQ-O3: Audit message format byte-identical.
- REQ-O4: Stackable LSM coexistence works (AppArmor + SELinux + Landlock simultaneously).
- REQ-O5: Per-namespace AppArmor profile (apparmor inside container) works.
- REQ-O6: TLA+ models (DFA match termination, profile-load atomicity, per-task label transition).
- REQ-O7: Hardening: profile load CAP_MAC_ADMIN; per-profile audit.

## Acceptance Criteria

- [ ] AC-O1: `aa-status` shows correct # of profiles loaded + their modes.
- [ ] AC-O2: Default Ubuntu profile set loads + enforces correctly.
- [ ] AC-O3: `aa-genprof <bin>` interactive profile generation works.
- [ ] AC-O4: Per-namespace test: AppArmor inside container with own profile-set.
- [ ] AC-O5: kselftest `tools/testing/selftests/security/apparmor/` passes.

## Verification

| TLA+ Model | Owner |
|---|---|
| `models/apparmor/dfa_match.tla` | `security/apparmor/match.md` (proves: DFA Aho-Corasick match termination + correctness; pathological-glob input bounded time) |
| `models/apparmor/profile_load.tla` | `security/apparmor/policy.md` (proves: profile load + replace atomically swaps; concurrent enforcement check sees either old or new fully) |
| `models/apparmor/label_transition.tla` | `security/apparmor/label-secid.md` (proves: per-task label transition during exec safe under concurrent ptrace) |

## Hardening

(AppArmor is row-2 LSM consumer; row-1 applies to its kernel code.)

| Feature | Default |
|---|---|
| **REFCOUNT** | per-profile + per-label refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-class hook tables `static const` | § Mandatory |
| **SIZE_OVERFLOW** | profile blob length + DFA table size arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed profile blob cleared | § Default-on configurable |

AppArmor-specific reinforcement: profile load CAP_MAC_ADMIN + LSM hook (`security_capable`); profile blob hash-verified before activate (defense against profile-tampering on disk); per-profile audit log.

## Open Questions
(none at Tier-2)

## Out of Scope
- apparmor-utils userspace
- Implementation code
- 32-bit-only paths
