# Tier-2: security/selinux — SELinux LSM (hooks + AVC + AVC cache + policy + xattr + selinuxfs + netlabel + ibpkey)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - security/selinux/
  - include/linux/lsm_hooks.h (SELinux hook implementations)
-->

## Summary

Tier-2 wrapper for SELinux — Type-Enforcement-based MAC LSM, originally NSA-developed, now the dominant Linux MAC enforcing in RHEL/Fedora/Android (since 4.4). Implements full path mediation across every LSM hook including file/inode/socket/IPC/IPC-binder/perf/keys/bpf/userns/RDMA/NETLABEL/INET-conntrack/CIPSO/IPSP. Components: **lsm hooks** (`hooks.c` — every `security_*` LSM hook impl, ~14000 lines), **AVC** (`avc.c` — Access-Vector-Cache for fast policy lookups), **AVC cache + AAS sid table** (`ss/sidtab.c` + `ss/avtab.c` — security-id table + access-vector tables), **policy** (`ss/policydb.c` + `ss/services.c` + `ss/conditional.c` + `ss/constraint.c` + `ss/symtab.c` + `ss/hashtab.c` + `ss/ebitmap.c` + `ss/mls.c` + `ss/mls_types.h`: policy compile + active-policy data structures + MLS Multi-Level-Security), **xattr** (xattr_security_* hooks integrated in `hooks.c`: per-file `security.selinux` xattr stores type label), **selinuxfs** (`selinuxfs.c` — `/sys/fs/selinux/` UAPI), **netlabel** (`netlabel.c` — CIPSO/CALIPSO label-on-IP-packet integration), **netif + netnode + netport + ibpkey** (`netif.c` + `netnode.c` + `netport.c` + `ibpkey.c`: per-interface + per-IP + per-port + per-IB-PKey label cache), **netlink** (`netlink.c` + `nlmsgtab.c`: per-netlink-protocol message-type → permission map), **ima** (`ima.c`: IMA integration glue), **initcalls** (`initcalls.c`: late_initcall to register LSM), **status** (`status.c`: `/sys/fs/selinux/status` polled status page), **genheaders** (`genheaders.c`: build-time header generator).

## Compatibility contract — outline

- `/sys/fs/selinux/` UAPI byte-identical: `enforce`, `disable`, `policy`, `mls`, `policyvers`, `commit_pending_bools`, `context`, `class/<name>/{index,perms/<name>}`, `policy_capabilities/<feature>`, `bools/<name>`, `null`, `avc/{cache_threshold,hash_stats,cache_stats}`, `checkreqprot`, `deny_unknown`, `reject_unknown`, `relabel`, `user`, `member`, `create`, `access`, `validatetrans`, `revalidate_*`. selinuxctl + libselinux + setools + audit2allow + restorecon + chcon + sestatus + semanage all consume unchanged.
- xattr `security.selinux` byte-identical (per-file label as user:role:type:level).
- Policy module wire format byte-identical (semodule + checkmodule + secilc consume unchanged).
- LSM hook impls preserve every behavior contract documented in security/selinux/* (hundreds of hooks).
- `audit:` records prefixed with `type=AVC msg=audit(...)` byte-identical (auditd + ausearch consume).
- Cmdline: `selinux={0,1}`, `enforcing={0,1}`, `selinux_compat_net=`, `audit=`, `audit_backlog_limit=`, `audit_backlog_wait_time=` parsed identically.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `security/selinux/hooks.md` | `hooks.c`: every LSM hook impl (split per-class into sub-Tier-3s if needed: file, inode, socket, ipc, key, bpf, perf, ib, etc.) |
| `security/selinux/avc.md` | `avc.c`: Access-Vector-Cache |
| `security/selinux/sidtab-avtab.md` | `ss/sidtab.c` + `ss/avtab.c`: SID table + access-vector table |
| `security/selinux/policydb.md` | `ss/policydb.c` + `ss/services.c` + related: policy compile + active-policy DS |
| `security/selinux/conditional-constraint.md` | `ss/conditional.c` + `ss/constraint.c`: conditional bools + constraints |
| `security/selinux/mls.md` | `ss/mls.c` + `ss/mls_types.h`: Multi-Level-Security |
| `security/selinux/symtab-hashtab-ebitmap.md` | `ss/symtab.c` + `ss/hashtab.c` + `ss/ebitmap.c`: shared data structures |
| `security/selinux/selinuxfs.md` | `selinuxfs.c` + `status.c`: `/sys/fs/selinux/` UAPI |
| `security/selinux/netlabel.md` | `netlabel.c`: CIPSO/CALIPSO IP-packet labeling |
| `security/selinux/netif-netnode-netport-ibpkey.md` | per-network-object label caches |
| `security/selinux/netlink.md` | `netlink.c` + `nlmsgtab.c`: netlink-message permission map |
| `security/selinux/ima.md` | `ima.c`: IMA glue |

## Compatibility outline

- REQ-O1: `/sys/fs/selinux/` UAPI byte-identical (libselinux + setools + audit2allow + sestatus consume unchanged).
- REQ-O2: xattr `security.selinux` byte-identical.
- REQ-O3: Policy module binary format byte-identical (semodule consumes unchanged).
- REQ-O4: AVC denial messages byte-identical (auditd consumes unchanged).
- REQ-O5: Every LSM hook impl behavior preserved.
- REQ-O6: Stackable LSM coexistence: SELinux + AppArmor + Landlock + GR-RBAC stackable composition works (LSM stacking enabled).
- REQ-O7: TLA+ models (AVC concurrent grant + revoke + cache-flush race; sidtab grow + concurrent SID lookup; policy load atomicity).
- REQ-O8: Hardening: SELinux is the row-2 LSM stackable framework consumer; hardens userspace; row-1 features apply equally to SELinux's own kernel code.

## Acceptance Criteria

- [ ] AC-O1: `sestatus` shows correct enforcement mode + policy version.
- [ ] AC-O2: `getenforce` + `setenforce 0/1` toggle correctly (in permissive mode).
- [ ] AC-O3: Reference policy (RHEL/Fedora targeted policy) loads + boots to multi-user.target with correct AVC behavior.
- [ ] AC-O4: `restorecon -Rv /` relabels every file correctly per policy.
- [ ] AC-O5: AVC denial test: deny rule triggers `type=AVC msg=...` audit record matching upstream format.
- [ ] AC-O6: Stackable test: SELinux + Landlock active simultaneously; both deny path works.
- [ ] AC-O7: kselftest `tools/testing/selftests/security/selinux/` passes.

## Verification

| TLA+ Model | Owner |
|---|---|
| `models/selinux/avc_grant_revoke.tla` | `security/selinux/avc.md` (proves: AVC concurrent grant lookup + cache-flush + revoke; revocation propagates within bounded time; no stale grant after revoke completes) |
| `models/selinux/sidtab_grow.tla` | `security/selinux/sidtab-avtab.md` (proves: sidtab table-grow under concurrent SID alloc + lookup; no SID lost during expansion) |
| `models/selinux/policy_load_atomic.tla` | `security/selinux/policydb.md` (proves: policy reload atomically swaps active policy; concurrent permission checks see either old policy or new policy fully, never half) |

## Hardening

(SELinux is itself a row-2 LSM consumer; row-1 features apply to SELinux's own kernel code too.)

| Feature | Default |
|---|---|
| **REFCOUNT** | per-policy + per-sidtab refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-class `class_perm` tables `static const` | § Mandatory |
| **SIZE_OVERFLOW** | policy entry-count + ebitmap size arithmetic checked | § Mandatory |
| **MEMORY_SANITIZE** | freed sidtab + AVC node cleared (carry per-process labels) | § Default-on configurable |

SELinux-specific reinforcement: policy load gated on CAP_MAC_ADMIN; selinuxfs writes on `enforce` / `disable` / `bools` / `commit_pending_bools` audit-logged; AVC cache stats restricted to CAP_MAC_ADMIN reader; `disable` writable only once at boot (before policy load); `setenforce` denied if policy loaded with `permissive=0` lockdown bit.

## Open Questions
(none at Tier-2)

## Out of Scope
- Reference policy implementation (selinux-policy + reference-policy packages, separate userspace projects)
- Implementation code
- 32-bit-only paths
