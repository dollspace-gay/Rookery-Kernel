# Tier-5 syscall: capset(2) — syscall 126

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/capability.c (SYSCALL_DEFINE2(capset), cap_validate_magic)
  - security/commoncap.c (cap_capset)
  - include/uapi/linux/capability.h (cap_user_header_t, cap_user_data_t)
  - include/linux/cred.h (struct cred.cap_effective/permitted/inheritable, cap_bset)
  - arch/x86/entry/syscalls/syscall_64.tbl (126  common  capset)
-->

## Summary

`capset(2)` writes the calling task's capability sets — effective, permitted, inheritable — under the constraints of `capabilities(7)`. The kernel enforces three classes of invariant on every call: (1) permitted′ ⊆ permitted (cannot raise permitted), (2) effective′ ⊆ permitted′ (effective bounded by new permitted), (3) inheritable′ ⊆ (permitted ∪ inheritable) ∩ cap_bset (inheritable bounded by bounding set and the previous permitted/inheritable union). It is the foundation of capability-aware setuid binaries, container init, and any program that "drops privileges except for what it actually needs."

Like `capget`, the data layout is versioned and 64-bit-split. The bounding set is NOT modifiable via `capset`; use `prctl(PR_CAPBSET_DROP, ...)`. The ambient set is NOT modifiable via `capset`; use `prctl(PR_CAP_AMBIENT, ...)`.

## Signature

```c
int capset(cap_user_header_t       header,
           const cap_user_data_t   dataptr);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `header` | `cap_user_header_t` | in/out | `{ __u32 version; int pid; }`. `version` MUST be V2/V3. `pid` MUST be 0 (self) or the caller's TID; cross-task capset is forbidden. |
| `dataptr` | `const cap_user_data_t` | in | Pointer to two `__user_cap_data_struct` (low + high) of the new {effective, permitted, inheritable}. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. Calling task's capability sets updated atomically. |
| `-1` + `errno` | Failure. Cap sets unchanged. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM`  | Requested permitted ⊄ current permitted (cannot raise); requested inheritable not within (permitted ∪ inheritable) ∩ cap_bset; requested effective ⊄ requested permitted; `CAP_SETPCAP` requirement violated. |
| `EINVAL` | Unknown `header.version` (kernel writes preferred V3 back); `pid` not 0 and not caller's own tid; data has bits beyond CAP_LAST_CAP set. |
| `EFAULT` | `header` or `dataptr` unreadable/unwritable. |
| `ESRCH`  | `pid > 0` and not self / not in tgid (legacy paths). |

## ABI surface

```text
__NR_capset (x86_64)  = 126
__NR_capset (i386)    = 185
__NR_capset (generic) = 91

Identical struct types to capget. Identical V1/V2/V3 versioning.

Reserved-bit mask (V3, post-CAP_LAST_CAP)
  CAP_LAST_CAP = 41
  CAP_VALID_MASK = (1ULL << (CAP_LAST_CAP + 1)) - 1
  /* Bits above must be zero; nonzero ⟹ -EINVAL */

Securebits that interact with capset
  SECBIT_KEEP_CAPS                     (1 << 4)
  SECBIT_KEEP_CAPS_LOCKED              (1 << 5)
  SECBIT_NO_SETUID_FIXUP               (1 << 2)
  SECBIT_NO_SETUID_FIXUP_LOCKED        (1 << 3)
  SECBIT_NOROOT                        (1 << 0)
  SECBIT_NOROOT_LOCKED                 (1 << 1)
  SECBIT_NO_CAP_AMBIENT_RAISE          (1 << 6)
  SECBIT_NO_CAP_AMBIENT_RAISE_LOCKED   (1 << 7)
```

## Compatibility contract

REQ-1: Syscall number is **126** on x86_64; **91** on generic-syscall archs. ABI-stable.

REQ-2: `header.pid` MUST be `0` or equal to `current.tid`. Linux historically also allowed setting the caller's *own* tgid threads, but kernel ≥ 2.6.26 restricts capset to self only. Cross-task capset → `-EINVAL` or `-EPERM`.

REQ-3: `header.version` MUST be V2 or V3. Unknown → write V3 into `*header` and return `-EINVAL` (probe protocol identical to `capget`).

REQ-4: Reserved bits (above `CAP_LAST_CAP`) MUST be zero in every set. Else `-EINVAL`. This rejects mis-coded libcap V1 callers writing junk to high bits.

REQ-5: Capability arithmetic — let `P`, `I`, `E` be current permitted, inheritable, effective; let `P'`, `I'`, `E'` be requested. Let `B` = bounding set, `A` = ambient set. Then:
  - `P' ⊆ P`                       (permitted is monotone-shrinking)
  - `E' ⊆ P'`                      (effective is a subset of new permitted)
  - `I' ⊆ (I ∪ P) ∩ B`             (inheritable bounded by union and bset)
Each violation → `-EPERM`.

REQ-6: If the caller wants to ADD to inheritable, it must have `CAP_SETPCAP` in its current *effective* set. (Adding to inheritable was historically a unique privilege; legacy gating retained as a CAP_SETPCAP requirement.)

REQ-7: Securebits interaction:
  - `SECBIT_NOROOT` prevents `setuid(nonzero) -> setuid(0)` from regaining caps; capset must respect.
  - `SECBIT_KEEP_CAPS` is independent of capset (relevant only at setuid).
  - `SECBIT_NO_CAP_AMBIENT_RAISE` blocks ambient raises via `prctl`, not capset.

REQ-8: After capset, the kernel reduces `A` (ambient) to `A ∩ P' ∩ I'` (ambient never exceeds permitted ∩ inheritable). This is a side effect, not directly visible until `prctl(PR_CAP_AMBIENT, IS_SET, cap)`.

REQ-9: After capset, `cap_bset` is unchanged (capset never modifies bset; `PR_CAPBSET_DROP` is the only path).

REQ-10: Atomicity: the new {P', I', E'} are committed under `cred->lock` write; no concurrent reader sees a partially-applied set. RCU-published new `cred`.

REQ-11: V1 calls receive 32-bit data; high-32 bits of the resulting sets are forced to zero. V1 is deprecated; modern callers MUST use V3.

REQ-12: LSM hook `security_capset` invoked AFTER the upstream `capabilities(7)` checks; LSMs (selinux/apparmor) MAY further deny. Hook is advisory-only on capabilities (not used to grant) by upstream convention.

REQ-13: A process with all-zero {P, I, E} cannot regain any capability via capset — capability gain requires either an execve of a binary with file caps, or ambient (raised under permitted+inheritable precondition).

REQ-14: capset is NOT a syscall that runs filter-stops in seccomp-`PTRACE_O_TRACESECCOMP` mode by default; treated as a normal syscall for filtering.

REQ-15: If `pid > 0 && pid == current.tid`, accepted (equivalent to 0). If `pid` is a sibling thread in the same tgid, legacy behavior allowed in 2.4 era; modern kernels reject → `-EPERM`.

REQ-16: Successful capset that drops a previously held capability is logged by audit subsystem (AUDIT_CAPSET).

## Acceptance Criteria

- [ ] AC-1: `capset({V3, 0}, {E=0, P=0, I=0})` succeeds; `capget` reports all sets empty.
- [ ] AC-2: `capset({V3, 0}, {E, P=full, I=0})` from a task whose current permitted is `full` succeeds.
- [ ] AC-3: `capset({V3, 0}, {E, P′})` with `P′ ⊃ P` returns `-EPERM`.
- [ ] AC-4: `capset({V3, 0}, {E′, P′})` with `E′ ⊄ P′` returns `-EPERM`.
- [ ] AC-5: `capset({V3, 0}, {*, *, I′})` with `I′ ⊄ (I ∪ P) ∩ B` returns `-EPERM`.
- [ ] AC-6: Adding caps to inheritable without `CAP_SETPCAP` in effective returns `-EPERM`.
- [ ] AC-7: `capset({V3, other_tid}, data)` (different task) returns `-EPERM` (or `-EINVAL`).
- [ ] AC-8: `capset({V3, getpid()}, data)` (own tid) succeeds.
- [ ] AC-9: `capset({0xdeadbeef, 0}, data)` returns `-EINVAL` AND writes V3 into `header.version`.
- [ ] AC-10: `capset` with bit > CAP_LAST_CAP set in any field returns `-EINVAL`.
- [ ] AC-11: `capset` succeeds; ambient set is reduced to `A ∩ P′ ∩ I′` automatically.
- [ ] AC-12: After `capset` dropping CAP_NET_BIND_SERVICE, `bind(socket, port < 1024)` returns `-EACCES`.
- [ ] AC-13: Failed capset (any reason): caps unchanged; `capget` reports pre-call values.
- [ ] AC-14: Concurrent `capget` from another task observes either pre or post snapshot, never torn.
- [ ] AC-15: AUDIT_CAPSET record emitted on every successful call.
- [ ] AC-16: V1 capset call only affects low-32 bits; high-32 forced to 0.
- [ ] AC-17: capset cannot grow bset (no path); `PR_CAPBSET_READ` after capset unchanged.

## Architecture

```rust
#[syscall(nr = 126, abi = "sysv")]
pub fn sys_capset(
    header: UserPtr<UserCapHeader>,
    data:   UserPtr<UserCapData>,
) -> isize {
    Capset::do_capset(header, data)
}
```

`Capset::do_capset(header, data) -> isize`:
1. let mut hdr: UserCapHeader = header.copy_in()?;
2. let abi_ver = match hdr.version {
3.   _LINUX_CAPABILITY_VERSION_2 |
4.   _LINUX_CAPABILITY_VERSION_3 => CapAbi::V3,
5.   _LINUX_CAPABILITY_VERSION_1 => CapAbi::V1,
6.   _ => { hdr.version = _LINUX_CAPABILITY_VERSION_3;
7.          header.copy_out(&hdr)?;
8.          return Err(EINVAL); }
9. };
10. /* pid must be self */
11. if hdr.pid != 0 && hdr.pid != current().tid { return Err(EPERM); }
12. /* Copy + reassemble new sets */
13. let new = parse_caps(&data, abi_ver)?;
14. if !valid_mask(new) { return Err(EINVAL); }    // bits ≤ CAP_LAST_CAP
15. /* Capability arithmetic */
16. Capset::check(&new)?;                          // EPERM if any rule violated
17. /* LSM */
18. security::capset(&new)?;
19. /* Commit under cred lock; RCU-publish new cred */
20. Capset::commit(new)?;
21. /* Reduce ambient to A ∩ P' ∩ I' */
22. Capset::reduce_ambient();
23. /* Audit */
24. audit::log_capset(&new);
25. Ok(0)

`Capset::check(new: CapTriple) -> Result`:
1. let (cur_p, cur_i, cur_e) = current_caps();
2. let bset = current().cred.cap_bset;
3. if !subset(new.permitted, cur_p) { return Err(EPERM); }
4. if !subset(new.effective, new.permitted) { return Err(EPERM); }
5. let allowed_inh = bitop_and(bitop_or(cur_i, cur_p), bset);
6. if !subset(new.inheritable, allowed_inh) { return Err(EPERM); }
7. /* Adding to inheritable requires CAP_SETPCAP */
8. if !subset(new.inheritable, cur_i) && !has_cap_effective(CAP_SETPCAP) {
9.   return Err(EPERM);
10. }
11. Ok(())

`Capset::commit(new)`:
1. let mut new_cred = prepare_creds()?;
2. new_cred.cap_permitted   = new.permitted;
3. new_cred.cap_inheritable = new.inheritable;
4. new_cred.cap_effective   = new.effective;
5. commit_creds(new_cred);

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `permitted_monotone_shrink` | INVARIANT | post: P' ⊆ P. |
| `effective_subset_permitted` | INVARIANT | post: E' ⊆ P'. |
| `inheritable_bounded` | INVARIANT | post: I' ⊆ (I ∪ P) ∩ B. |
| `cap_setpcap_for_inh_raise` | INVARIANT | I' ⊃ I ⟹ CAP_SETPCAP ∈ E. |
| `bset_immutable_via_capset` | INVARIANT | post: B' == B. |
| `ambient_reduced` | INVARIANT | post: A' ⊆ A ∩ P' ∩ I'. |
| `self_only` | INVARIANT | header.pid ∈ {0, current.tid}. |
| `reserved_bits_zero` | INVARIANT | any bit > CAP_LAST_CAP set ⟹ `-EINVAL`. |
| `atomic_on_error` | INVARIANT | any check failure ⟹ caps unchanged. |

### Layer 2: TLA+

`kernel/capset.tla`:
- States: task.cred.{cap_effective, cap_permitted, cap_inheritable, cap_bset, cap_ambient, securebits}.
- Properties:
  - `safety_capabilities7_arithmetic` — every commit obeys capabilities(7) rules.
  - `safety_bset_unmodified` — invariant.
  - `safety_atomic` — commit is all-or-nothing under cred lock.
  - `safety_self_only` — never modifies another task's caps.
  - `safety_ambient_constraint` — A ⊆ P ∩ I post-commit.
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `check` post: returns Ok ⟹ new triple obeys all capabilities(7) rules | `Capset::check` |
| `commit` post: cred fields == new triple; bset unchanged | `Capset::commit` |
| `reduce_ambient` post: A' == A ∩ P' ∩ I' | `Capset::reduce_ambient` |
| `parse_caps` post: high bits of u64 == 0 for caps > CAP_LAST_CAP | `Capset::parse_caps` |

### Layer 4: Verus / Creusot functional

Per-`capset(2)` + `capabilities(7)` semantic equivalence. LTP `capset01..capset04` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`capset(2)` reinforcement:

- **Per-permitted monotone** — defense against per-self-elevation via capset.
- **Per-inheritable bounded by bset** — defense against per-bypass-of-PR_CAPBSET_DROP via inh-raise.
- **Per-CAP_SETPCAP gate on inh-raise** — defense against per-unprivileged-inheritance-injection.
- **Per-self-only target** — defense against per-cross-task cap injection.
- **Per-ambient auto-reduce** — defense against per-stale-ambient surviving permitted/inheritable drop.
- **Per-atomic-or-fail** — defense against per-partial-cap-state corruption.
- **Per-version-probe write-back** — defense against per-mis-coded-libcap mis-write.
- **Per-reserved-bit check** — defense against per-future-cap accidental allocation.
- **Per-AUDIT_CAPSET** — defense against per-cap-drop log elision.

## Grsecurity / PaX surface

- **PaX UDEREF on `header` and `dataptr`** — defense against per-cap-buffer kernel-deref.
- **PAX_RANDKSTACK at capset entry** — randomizes kernel stack offset per call.
- **GRKERNSEC_CHROOT_CAPS** — chroot strips `CAP_SETPCAP` (among others) on entry; capset inside chroot cannot raise inheritable.
- **CAP_SETPCAP / SETUID / SETGID strict** — grsec policy may further refuse capset operations that target these caps unless the task is on an RBAC whitelist.
- **Per-grsec capset audit** — every successful capset logged with delta (which caps gained/lost in P, I, E).
- **Per-grsec inheritable freeze** — grsec policy can lock inheritable to current value; further capset attempts to change inheritable return `-EPERM`.
- **PaX KERNEXEC on cred-update path** — `commit_creds` runs in KERNEXEC-mapped code; no R/W cred-update gadget.
- **GRKERNSEC_BRUTE** — repeated `-EPERM` from capset triggers per-uid rate-limit (capability-probing detector).
- **No `CAP_SYS_MODULE` reraise** — grsec ensures that once a task drops `CAP_SYS_MODULE`, capset cannot get it back even under nominal capabilities(7) rules (policy override).

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- File capabilities (`security.capability` xattr) (Tier-3 in `security/commoncap.md`).
- Bounding-set drop via `prctl(PR_CAPBSET_DROP)` (covered in `uapi/syscalls/prctl.md`).
- Ambient raise via `prctl(PR_CAP_AMBIENT)` (covered in `uapi/syscalls/prctl.md`).
- User-namespace cap-map projection (Tier-3 in `kernel/user_namespace.md`).
- LSM-specific cap policies (selinux/apparmor) (Tier-3 in respective LSM docs).
- Implementation code.
