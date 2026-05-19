---
title: "Tier-5 syscall: capget(2) — syscall 125"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`capget(2)` reads a task's POSIX.1e capability sets — effective, permitted, inheritable — into userspace. Coupled with its sibling `capset(2)`, it is the raw kernel interface that `libcap` wraps for user-friendly capability inspection. Capabilities are 64-bit on modern kernels, exposed over a versioned ABI: callers first probe with `_LINUX_CAPABILITY_VERSION_3` (the only supported value), receive a two-`__u32`-per-set layout, and reconstruct the 64-bit values themselves.

Critical for: `getcap`, `capsh --print`, container init (verifying delivered caps), Docker/podman bootstrap, security audit tools.

### Acceptance Criteria

- [ ] AC-1: `capget({V3, 0}, data)` on root task: data shows full effective+permitted = (1 << CAP_LAST_CAP+1) - 1 across both halves.
- [ ] AC-2: `capget({V3, 0}, data)` on non-root task: effective/permitted reflect the post-`setuid` cap drop.
- [ ] AC-3: `capget({V3, getpid()}, data)` returns the same data as `header.pid = 0`.
- [ ] AC-4: `capget({0xdeadbeef, 0}, data)` returns `-EINVAL` AND writes `_LINUX_CAPABILITY_VERSION_3` into `header->version`.
- [ ] AC-5: `capget({V3, 99999}, data)` (no such pid) returns `-ESRCH`.
- [ ] AC-6: `capget({V3, -5}, data)` returns `-EINVAL`.
- [ ] AC-7: `capget(NULL, data)` returns `-EFAULT`.
- [ ] AC-8: `capget({V3, 0}, NULL)` returns `-EFAULT`.
- [ ] AC-9: After `prctl(PR_CAP_AMBIENT, PR_CAP_AMBIENT_RAISE, CAP_NET_BIND_SERVICE, 0, 0)`, `capget` shows ambient NOT in any of effective/permitted/inheritable per se but inheritable AND permitted both include CAP_NET_BIND_SERVICE (precondition for ambient raise).
- [ ] AC-10: `capget` of a task in a different user-namespace returns that task's cap sets as represented in *that* user-namespace.
- [ ] AC-11: Race with concurrent capset on target: returned snapshot is wholly pre- or wholly post-change (no torn read).
- [ ] AC-12: `capget({_LINUX_CAPABILITY_VERSION_1, 0}, data)`: kernel accepts (legacy) but returns only low-32 bits; high-32 cleared.

### Architecture

```rust
#[syscall(nr = 125, abi = "sysv")]
pub fn sys_capget(
    header: UserPtr<UserCapHeader>,
    data:   UserPtr<UserCapData>,
) -> isize {
    Capget::do_capget(header, data)
}
```

`struct UserCapHeader { version: u32, pid: i32 }`
`struct UserCapData { effective: u32, permitted: u32, inheritable: u32 }`

`Capget::do_capget(header, data) -> isize`:
1. let mut hdr: UserCapHeader = header.copy_in()?;
2. /* Version probe */
3. let abi_ver = match hdr.version {
4.   _LINUX_CAPABILITY_VERSION_1 => CapAbi::V1,
5.   _LINUX_CAPABILITY_VERSION_2 => CapAbi::V3,    // upgrade silently
6.   _LINUX_CAPABILITY_VERSION_3 => CapAbi::V3,
7.   _ => {
8.     hdr.version = _LINUX_CAPABILITY_VERSION_3;
9.     header.copy_out(&hdr)?;
10.    return Err(EINVAL);
11.  }
12. };
13. /* Pid normalization */
14. let target = match hdr.pid {
15.   0          => current(),
16.   p if p > 0 => Task::find_by_tid(p).ok_or(ESRCH)?,
17.   _          => return Err(EINVAL),
18. };
19. /* Sample target's cred under RCU */
20. let snapshot = rcu::read(|| {
21.   let cred = target.real_cred();
22.   (cred.cap_effective, cred.cap_permitted, cred.cap_inheritable)
23. });
24. /* Marshal */
25. let payload = match abi_ver {
26.   CapAbi::V1 => marshal_v1(snapshot),
27.   CapAbi::V3 => marshal_v3(snapshot),
28. };
29. data.copy_out(&payload)?;
30. Ok(0)

`marshal_v3((eff, prm, inh)) -> [UserCapData; 2]`:
1. [
2.   UserCapData {
3.     effective:   (eff.val() & 0xFFFFFFFF) as u32,
4.     permitted:   (prm.val() & 0xFFFFFFFF) as u32,
5.     inheritable: (inh.val() & 0xFFFFFFFF) as u32,
6.   },
7.   UserCapData {
8.     effective:   (eff.val() >> 32) as u32,
9.     permitted:   (prm.val() >> 32) as u32,
10.    inheritable: (inh.val() >> 32) as u32,
11.  },
12. ]

### Out of Scope

- `capset(2)` (Tier-5 separate doc).
- File capabilities (`security.capability` xattr) (Tier-3 in `security/commoncap.md`).
- Bounding set / ambient set queries via `prctl` (covered in `uapi/syscalls/prctl.md`).
- User-namespace cap mapping (Tier-3 in `kernel/user_namespace.md`).
- Implementation code.

### signature

```c
int capget(cap_user_header_t  header,
           cap_user_data_t    dataptr);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `header` | `cap_user_header_t` | in/out | `{ __u32 version; int pid; }`. `version` MUST be `_LINUX_CAPABILITY_VERSION_3`. `pid` is the target TID (0 = self). On version mismatch the kernel writes the preferred version into `*header` and returns `-EINVAL`. |
| `dataptr` | `cap_user_data_t` | out | Pointer to an **array of two** `struct __user_cap_data_struct` (low 32 bits, high 32 bits) per set. Writes `{ effective, permitted, inheritable }` each as two `__u32` halves. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success. Cap sets written to `*dataptr`. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `header == NULL`; unknown `header.version`; the kernel writes preferred version on this path. |
| `EFAULT` | `header` or `dataptr` unreadable/unwritable. |
| `ESRCH`  | `header.pid > 0` but no such task. |
| `EPERM`  | `header.pid > 0` and target lives in a user-namespace the caller cannot inspect (rare; mostly forbidden combinations). |

### abi surface

```text
__NR_capget (x86_64)  = 125
__NR_capget (i386)    = 184
__NR_capget (generic) = 90

Capability ABI versions
  _LINUX_CAPABILITY_VERSION_1   = 0x19980330   /* 32-bit, single struct; DEPRECATED */
  _LINUX_CAPABILITY_VERSION_2   = 0x20071026   /* 64-bit, two structs; SUPERSEDED */
  _LINUX_CAPABILITY_VERSION_3   = 0x20080522   /* 64-bit, two structs; CURRENT */

typedef struct __user_cap_header_struct {
    __u32 version;
    int   pid;
} *cap_user_header_t;

typedef struct __user_cap_data_struct {
    __u32 effective;
    __u32 permitted;
    __u32 inheritable;
} *cap_user_data_t;

/* V3 layout: dataptr points to TWO of these:
 *   dataptr[0] = low 32 bits of each set (caps 0..31)
 *   dataptr[1] = high 32 bits (caps 32..63)
 * Reassemble: cap_effective_u64 = ((u64)dataptr[1].effective << 32) | dataptr[0].effective
 */

Capability count
  CAP_LAST_CAP = 41           /* CAP_CHECKPOINT_RESTORE is highest in 7.1-rc2 */
  /* The 64-bit set has 22 reserved bits at the top. */
```

### compatibility contract

REQ-1: Syscall number is **125** on x86_64; **90** on generic-syscall archs. ABI-stable.

REQ-2: Caller MUST set `header.version`. Supported value is `_LINUX_CAPABILITY_VERSION_3`. If a caller passes `_LINUX_CAPABILITY_VERSION_2`, the kernel accepts it (back-compat) but treats it identically to V3.

REQ-3: If `header.version` is unrecognized, the kernel writes the preferred version into `*header` and returns `-EINVAL`. Callers use this for probing.

REQ-4: `header.pid == 0` → query the calling task. `header.pid > 0` → query that TID. `header.pid < 0` → `-EINVAL`.

REQ-5: For `header.pid > 0`: target must exist (`ESRCH` if not). NO additional capability is required to *read* another task's capability sets (sets are advisory-readable like `/proc/<pid>/status`).

REQ-6: For `_LINUX_CAPABILITY_VERSION_3`, `dataptr` MUST be writable for `2 * sizeof(struct __user_cap_data_struct) = 24 bytes`. Kernel does NOT pre-check via copy_to_user probe; relies on user pointer faults to return `-EFAULT`.

REQ-7: Writes:
  - `dataptr[0].effective`   = `cred->cap_effective.val & 0xFFFFFFFF`
  - `dataptr[0].permitted`   = `cred->cap_permitted.val & 0xFFFFFFFF`
  - `dataptr[0].inheritable` = `cred->cap_inheritable.val & 0xFFFFFFFF`
  - `dataptr[1].effective`   = `cred->cap_effective.val >> 32`
  - `dataptr[1].permitted`   = `cred->cap_permitted.val >> 32`
  - `dataptr[1].inheritable` = `cred->cap_inheritable.val >> 32`

REQ-8: `capget` does NOT report the bounding set (`cap_bset`) — that is queried via `prctl(PR_CAPBSET_READ, cap)`. Likewise the ambient set is read via `prctl(PR_CAP_AMBIENT, PR_CAP_AMBIENT_IS_SET, cap)`.

REQ-9: Concurrent target-task credential change: `capget` reads under RCU; observes either the pre- or post-change snapshot atomically. Never tears a 64-bit set.

REQ-10: For V1 reads, kernel returns the low 32 bits in a single struct and clears the high bits silently (LIBSEMVER deprecation behavior).

REQ-11: `capget` is unprivileged. There is no LSM hook for the query path (cap sets are considered process-public metadata).

REQ-12: `dataptr == NULL` with `header.version` valid: returns `-EFAULT`.

REQ-13: After successful return, the caller reassembles 64-bit sets per REQ-7 to feed `cap_get_flag` / `cap_to_text` libcap helpers.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `version_probe_writes_back` | INVARIANT | unknown version ⟹ header.version := V3 AND return `-EINVAL`. |
| `pid_zero_self` | INVARIANT | hdr.pid == 0 ⟹ target = current. |
| `pid_negative_einval` | INVARIANT | hdr.pid < 0 ⟹ `-EINVAL`. |
| `snapshot_atomic` | INVARIANT | RCU read sees consistent (eff, prm, inh) triple. |
| `marshal_low_high_split` | INVARIANT | u64 set ↔ (low u32, high u32) bijection. |
| `no_state_mutation` | INVARIANT | capget is read-only; no cred sets mutated. |

### Layer 2: TLA+

`kernel/capget.tla`:
- States: target.cred snapshot.
- Properties:
  - `safety_no_state_change` — capget is purely a query; no cred or task state mutated.
  - `safety_atomic_snapshot` — output never spans two different cred generations.
  - `safety_pid_validated` — negative pid rejected before dereference.
  - `liveness_returns` — every well-formed call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_capget` post: no Task field mutated except header.version on EINVAL path | `Capget::do_capget` |
| `marshal_v3` post: rebuild((low,hi)) == original u64 | `Capget::marshal_v3` |
| `find_by_tid` post: returns Some iff TID alive | `Task::find_by_tid` |

### Layer 4: Verus / Creusot functional

Per-`capget(2)` man-page equivalence. LTP `capget01..capget02` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`capget(2)` reinforcement:

- **Per-version-probe write-back** — defense against per-stale-libcap mis-decode of cap sets.
- **Per-RCU snapshot** — defense against per-torn-read race during concurrent `capset`.
- **Per-negative-pid rejection** — defense against per-kernel-deref via negative-pid arithmetic.
- **Per-readonly-query semantics** — defense against per-mutation-via-capget bug pattern.

### grsecurity / pax surface

- **PaX UDEREF on `header` and `dataptr` copy** — defense against per-cap-buffer kernel-deref bug.
- **PAX_RANDKSTACK at capget entry** — randomizes kernel stack offset per call.
- **GRKERNSEC_PROC restrictions** — `capget` of another task's caps is visible only if that task is visible per grsec proc-pid policy; otherwise `-ESRCH` (hide rather than leak existence).
- **GRKERNSEC_CHROOT_CAPS** — inside chroot, `capget` reports the stripped set (consistent with what the chroot'd task actually has post-strip).
- **Per-grsec audit (optional)** — capget calls on PID != self can be logged via grsec audit policy to detect reconnaissance.
- **CAP_SETPCAP / SETUID / SETGID strict** — grsec policy restricts which tasks can *have* these caps in the first place; `capget` will reflect the stricter set.

