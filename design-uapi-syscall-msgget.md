---
title: "Tier-5 syscall: msgget(2) — syscall 68"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`msgget(2)` returns the System V message-queue identifier associated with `key`. If `key == IPC_PRIVATE` or no queue exists for `key` and `IPC_CREAT` is set in `msgflg`, a new queue is created. If both `IPC_CREAT` and `IPC_EXCL` are set and a queue already exists for `key`, the call fails with `EEXIST`. The low nine bits of `msgflg` form the access mode (read/write for owner/group/other) applied to the new queue.

The queue identifier returned is an opaque non-negative integer scoped to the calling IPC namespace; it is reused after deletion. Critical for: System V IPC interop, legacy daemons, container init systems, and SUSv4 conformance.

### Acceptance Criteria

- [ ] AC-1: `msgget(IPC_PRIVATE, 0600)` returns a non-negative id distinct across calls.
- [ ] AC-2: `msgget(K, IPC_CREAT|0600)` followed by `msgget(K, IPC_CREAT|IPC_EXCL|0600)` returns `-EEXIST`.
- [ ] AC-3: `msgget(K, 0)` when no queue exists returns `-ENOENT`.
- [ ] AC-4: Queue created with mode `0600` is not accessible to a uid != cuid without `CAP_IPC_OWNER`.
- [ ] AC-5: `MSGMNI + 1` queues: the last one returns `-ENOSPC`.
- [ ] AC-6: Queue in namespace A is not visible from namespace B (CLONE_NEWIPC).
- [ ] AC-7: ID returned satisfies `(id & 0xFFFF) < MSGMNI` (index portion).
- [ ] AC-8: `key == IPC_PRIVATE` ignores `IPC_EXCL`.

### Architecture

```rust
#[syscall(nr = 68, abi = "sysv")]
pub fn sys_msgget(key: i32, msgflg: i32) -> isize {
    Ipc::msgget(key, msgflg)
}
```

`Ipc::msgget(key, msgflg) -> isize`:
1. let ns = current_ipc_ns();
2. let params = IpcParams { key, flg: msgflg };
3. let ids = &ns.msg_ids;
4. if key == IPC_PRIVATE { return Ipc::newque(ns, &params); }
5. /* Lookup */
6. let r = ipc_findkey(ids, key);
7. match r {
8.   None => {
9.     if msgflg & IPC_CREAT == 0 { return -ENOENT; }
10.    return Ipc::newque(ns, &params);
11.  }
12.  Some(slot) => {
13.    if (msgflg & (IPC_CREAT|IPC_EXCL)) == (IPC_CREAT|IPC_EXCL) { return -EEXIST; }
14.    Ipc::msg_security_associate(slot, msgflg)?;
15.    Ipc::ipcperms(slot, msgflg & 0777)?;
16.    return slot.id as isize;
17.  }
18. }
```

`Ipc::newque(ns, params) -> isize`:
1. if ns.msg_ids.in_use >= ns.msg_ctlmni { return -ENOSPC; }
2. let msq = Box::try_new(MsgQueue::default())?;     // ENOMEM
3. msq.q_perm.mode = (params.flg & 0777) as u16;
4. msq.q_perm.key  = params.key;
5. msq.q_perm.uid  = current_euid();  msq.q_perm.cuid = msq.q_perm.uid;
6. msq.q_perm.gid  = current_egid();  msq.q_perm.cgid = msq.q_perm.gid;
7. msq.q_qbytes    = ns.msg_ctlmnb;
8. Ipc::security_msg_queue_alloc(&mut msq)?;
9. let id = ipc_addid(&ns.msg_ids, &mut msq.q_perm, ns.msg_ctlmni)?;
10. return msq.q_perm.id as isize;

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `private_always_new` | INVARIANT | key==IPC_PRIVATE never collides with existing slot. |
| `excl_rejects_existing` | INVARIANT | IPC_CREAT|IPC_EXCL on existing key ⟹ EEXIST. |
| `nocreate_missing_enoent` | INVARIANT | !IPC_CREAT on missing key ⟹ ENOENT. |
| `id_index_bounded` | INVARIANT | returned id index < MSGMNI. |
| `cap_owner_bypass` | INVARIANT | CAP_IPC_OWNER in user-ns bypasses mode check. |

### Layer 2: TLA+

`ipc/msgget.tla`:
- States: key-lookup, ENOENT/CREAT decision, EEXIST decision, newque allocation, ID-install.
- Properties:
  - `safety_namespace_isolation` — queue created in ns A not visible in ns B.
  - `safety_quota_msgmni` — never exceed MSGMNI per namespace.
  - `safety_creds_capture` — cuid/cgid captured at creation.
  - `liveness_finite_dispatch` — msgget terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `msgget` post: success ⟹ id ∈ ns.msg_ids active set | `Ipc::msgget` |
| `newque` post: q_perm.cuid = euid(current) | `Ipc::newque` |
| `ipc_findkey` post: returns Some iff key matches active slot in ns | `Ipc::ipc_findkey` |

### Layer 4: Verus / Creusot functional

Per-`msgget(2)` man-page; LTP `ipc/msgget*` test set; SUSv4 conformance.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`msgget(2)` reinforcement:

- **Per-namespace MSGMNI quota** — defense against per-queue-exhaustion DoS.
- **Per-cuid capture at creation** — defense against per-ownership-spoofing.
- **Per-LSM alloc/associate hooks** — defense against per-policy-bypass.
- **Per-`(seq << 16) | index` id composition** — defense against per-stale-id reuse.

## Grsecurity / PaX surface

- **PaX UDEREF irrelevant** — msgget takes only scalars; nonetheless SMAP/SMEP active.
- **GRKERNSEC_HARDEN_IPC** — requires that the caller's euid match the queue creator's cuid for any open (msgget on existing key), regardless of mode bits. The traditional Unix "mode 0666 means world-readable IPC" loophole is closed; cross-uid access requires explicit `CAP_IPC_OWNER` in the queue's user namespace.
- **GRKERNSEC_PROC_IPC** — `/proc/sysvipc/msg` listing restricted to root or the queue's cuid; per-row info-leaks (lspid/lrpid) hidden from non-owner.
- **CAP_IPC_OWNER strict in target user-ns** — defense against cross-userns priv-raise; the check uses `ns_capable(msq->ns->user_ns, CAP_IPC_OWNER)`, not the global init_user_ns.
- **MSGMNI per-namespace cap (configurable, default scaled to RAM)** — defense against per-ipc-ns DoS by an unprivileged userns.
- **Sequence number always advances on RMID even on EEXIST race** — defense against per-id-collision attack window during RCU teardown.
- **GRKERNSEC_RAND_IDS** — option to randomize the `seq` component of new ids (instead of monotonic ++), defeating predictability for in-namespace IPC fuzzing.
- **IPC namespace strict** — even root in a child userns cannot reach parent-ns queues; init_ipc_ns is treated as protected.
- **PAX_REFCOUNT on q_perm refcount** — defense against per-refcount-overflow UAF during concurrent RMID.
- **PAX_USERCOPY whitelist** — although msgget itself does no copy, the msq slab is registered with usercopy-deny so accidental copy_to_user against it traps.
- **LSM `security_msg_queue_alloc` mandatory** — defense against per-creation-bypass when SELinux/AppArmor not loaded; built-in stub still records auditable record.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- msgsnd/msgrcv data-path (covered in `msgsnd.md`, `msgrcv.md`).
- msgctl administration (covered in `msgctl.md`).
- Implementation code.
- Pre-2.4 i386 ipc(2) multiplexer trampoline (legacy compat only).

### signature

```c
int msgget(key_t key, int msgflg);
```

```c
#define IPC_PRIVATE   ((__kernel_key_t) 0)
#define IPC_CREAT     00001000
#define IPC_EXCL      00002000
#define IPC_NOWAIT    00004000
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `key` | `key_t` (`int32_t`) | in | IPC key; `IPC_PRIVATE` (0) requests a fresh queue. |
| `msgflg` | `int` | in | `IPC_CREAT`, `IPC_EXCL`, plus low-9-bit mode (rwx ugo). |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Message-queue identifier. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EACCES` | Queue exists but caller lacks permission per `msgflg` mode. |
| `EEXIST` | `IPC_CREAT|IPC_EXCL` requested and queue already exists. |
| `ENOENT` | No queue for `key` and `IPC_CREAT` not set. |
| `ENOMEM` | Out of memory creating the queue control structure. |
| `ENOSPC` | System-wide queue count would exceed `MSGMNI`. |

### abi surface

```text
__NR_msgget  (x86_64)  = 68
__NR_msgget  (arm64)   = generic via __NR_msgget = 186 (SYS_msgget)
__NR_msgget  (riscv)   = 186
__NR_msgget  (i386)    = via ipc(2) multiplexer (call 1)

key_t is int32_t on Linux.
Identifier numeric domain: [0, INT_MAX); sequence-shifted to avoid id reuse races.
```

### compatibility contract

REQ-1: Syscall number is **68** on x86_64. ABI-stable.

REQ-2: `key == IPC_PRIVATE`: always create a new queue regardless of other state; `IPC_EXCL` ignored on `IPC_PRIVATE`.

REQ-3: `key != IPC_PRIVATE`: look up existing queue by key in current `ipc_namespace`. If not found and `IPC_CREAT` set: create. If found and `IPC_CREAT|IPC_EXCL` set: `-EEXIST`. If not found and `IPC_CREAT` not set: `-ENOENT`.

REQ-4: Permission check on existing queue uses `msgflg & 0777` against the queue's stored mode (`ipc_perm.mode`) via `ipcperms()`; `CAP_IPC_OWNER` in the queue's user namespace bypasses the check.

REQ-5: Newly created queue has `msg_perm.uid = msg_perm.cuid = euid(current)`, `msg_perm.gid = msg_perm.cgid = egid(current)`, `msg_perm.mode = msgflg & 0777`, `msg_qbytes = ns->msg_ctlmnb` (default 16384 / MSGMNB).

REQ-6: System-wide limit `MSGMNI` (per-namespace) caps the number of queues; `ENOSPC` on overrun. Default scales with RAM (typically 32000).

REQ-7: Identifier composition: `id = (seq << 16) | index` where `seq` increments per-slot reuse. This makes reuse-races detectable; callers must use the entire returned `int`.

REQ-8: Operation is per-IPC-namespace. Queues in one namespace are invisible to others. CLONE_NEWIPC isolates.

REQ-9: LSM hook `security_msg_queue_alloc(&msq)` called on creation; `security_msg_queue_associate(msq, msgflg)` called on lookup. Either may return `-EACCES`.

REQ-10: No userspace pointers are dereferenced; entirely scalar-in / scalar-out.

