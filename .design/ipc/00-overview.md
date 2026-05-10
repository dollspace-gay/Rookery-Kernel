# Subsystem: ipc/ — System V IPC + POSIX message queues

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: in-v0
upstream-paths:
  - ipc/
  - include/linux/ipc.h
  - include/linux/ipc_namespace.h
  - include/linux/msg.h
  - include/linux/sem.h
  - include/linux/shm.h
  - include/uapi/linux/ipc.h
  - include/uapi/linux/msg.h
  - include/uapi/linux/sem.h
  - include/uapi/linux/shm.h
  - include/uapi/linux/mqueue.h
-->

## Summary
Tier-2 overview for `ipc/` — System V IPC (message queues, semaphores, shared memory) plus POSIX message queues. Small subsystem (~13 files), well-defined POSIX semantics, but contains the IPC namespace machinery used by container runtimes.

Note: futex (the modern userspace mutex primitive) is owned by `kernel/00-overview.md` § futex.md, not here. AF_UNIX local sockets are owned by `net/00-overview.md`. eventfd / signalfd / pipe / fanotify (all "IPC-flavored" descriptor mechanisms) are owned by `fs/00-overview.md` § event-fds.md and `fs/00-overview.md` § pipe-splice.md. This subsystem is specifically the System V IPC API (`msgget`, `semget`, `shmget`, …) and POSIX mqueue (`mq_*`).

## Upstream references in scope

| Category | Upstream paths | Planned Tier-3 doc |
|---|---|---|
| System V message queues | `ipc/msg.c`, `ipc/msgutil.c`, `include/linux/msg.h`, `include/uapi/linux/msg.h` | `sysv-msg.md` |
| System V semaphores | `ipc/sem.c`, `include/linux/sem.h`, `include/uapi/linux/sem.h` | `sysv-sem.md` |
| System V shared memory | `ipc/shm.c`, `include/linux/shm.h`, `include/uapi/linux/shm.h` | `sysv-shm.md` |
| POSIX message queues | `ipc/mqueue.c`, `ipc/mq_sysctl.c`, `include/linux/mqueue.h`, `include/uapi/linux/mqueue.h` | `posix-mqueue.md` |
| IPC namespace + util + sysctl | `ipc/namespace.c`, `ipc/util.c`, `ipc/util.h`, `ipc_sysctl.c`, `ipc/syscall.c` (compat-arch dispatcher), `ipc/compat.c` (32-bit-on-64-bit translation) | `ipc-namespace.md` |

## Compatibility contract

### Syscall surface

System V IPC: `msgget`, `msgsnd`, `msgrcv`, `msgctl`, `semget`, `semop`, `semtimedop`, `semctl`, `shmget`, `shmat`, `shmdt`, `shmctl`. POSIX mqueue: `mq_open`, `mq_unlink`, `mq_timedsend`, `mq_timedreceive`, `mq_notify`, `mq_getsetattr`. Compat dispatcher: `ipc(2)` (legacy multiplexer; some arches only).

(Each gets a Tier-5 `uapi/syscalls/<name>.md` in Phase D.)

### `/proc` surfaces

| Path | Owner doc | Compat level |
|---|---|---|
| `/proc/sysvipc/{msg,sem,shm}` | `ipc-namespace.md` | Format-identical |
| `/proc/sys/kernel/{msgmax,msgmni,msgmnb,sem,shmmax,shmmni,shmall}` | `ipc-namespace.md` | Identical |
| `/proc/sys/fs/mqueue/{queues_max,msg_max,msgsize_max,...}` | `posix-mqueue.md` | Identical |
| Mount of mqueuefs at `/dev/mqueue` | `posix-mqueue.md` | Filesystem identical |

## Requirements

- REQ-1: Every System V IPC syscall is byte-identical with upstream — entry/exit, errno semantics, struct layouts (`struct msqid_ds`, `struct semid_ds`, `struct shmid_ds`, `struct ipc_perm`), `IPC_CREAT`/`IPC_EXCL`/`SEM_UNDO` semantics.
- REQ-2: Every POSIX mqueue syscall is byte-identical; mqueuefs filesystem mounts at `/dev/mqueue` and behaves identically.
- REQ-3: IPC namespaces (CLONE_NEWIPC) provide identical isolation semantics — separate `/proc/sysvipc/{msg,sem,shm}` views per namespace.
- REQ-4: Sysctl knobs (`/proc/sys/kernel/{msgmax,msgmni,msgmnb,sem,shmmax,shmmni,shmall}`, `/proc/sys/fs/mqueue/*`) preserve names + value semantics + per-namespace scoping.
- REQ-5: `ipc(2)` multiplexer compat dispatcher (architecture-conditional; required on some arches for 32-bit ABI) preserves entry conventions for ia32 compat layer.
- REQ-6: All Tier-3 docs spawned by this overview each declare their unsafe-block clusters, TLA+ models (the System V semaphore semantics are subtle enough to merit one).

## Acceptance Criteria

- [ ] AC-1: System V IPC selftests under `tools/testing/selftests/sysvipc/` pass with the same set as upstream. (covers REQ-1)
- [ ] AC-2: POSIX mqueue selftests under `tools/testing/selftests/mqueue/` pass. (covers REQ-2)
- [ ] AC-3: An IPC-namespaced container (`unshare -i`) sees an empty `/proc/sysvipc/{msg,sem,shm}` and operations in the namespace don't affect the host. (covers REQ-3)
- [ ] AC-4: A diff between Rookery's and upstream's `/proc/sys/kernel/{msgmax,msgmni,msgmnb,sem,shmmax,shmmni,shmall}` defaults and behavior under writes is empty. (covers REQ-4)
- [ ] AC-5: 32-bit userspace using `ipc(2)` syscalls under Rookery operate identically. (covers REQ-5)
- [ ] AC-6: `make verify` passes ipc/ Kani harnesses; `make tla` passes any ipc/ models. (covers REQ-6)

## Architecture

### Layout map

```
.design/ipc/
  00-overview.md          ← this document
  sysv-msg.md             ← System V message queues
  sysv-sem.md             ← System V semaphores (incl. SEM_UNDO undo log)
  sysv-shm.md             ← System V shared memory
  posix-mqueue.md         ← POSIX mqueue + mqueuefs
  ipc-namespace.md        ← namespaces + util + sysctl + compat dispatcher
```

### Cross-references

- `kernel/00-overview.md` — futex is the modern primitive but this subsystem keeps the SysV API; users who need both consult kernel/futex.md.
- `mm/00-overview.md` — System V shared memory is a special tmpfs-backed mapping; cross-ref to `mm/00-overview.md` and `fs/00-overview.md` § pseudo-fs/ramfs-tmpfs.md.
- `00-glossary.md` — `msg_queue / sem / shm`, `namespace`, `pipe` (related).

### Rust module organization (informative)

- `kernel::ipc::sysv_msg`, `kernel::ipc::sysv_sem`, `kernel::ipc::sysv_shm`, `kernel::ipc::mqueue` — Rookery to author.

### Locking and concurrency

System V IPC uses an `ids->rwsem` for namespace-wide mutator serialization, plus per-object refcount + RCU for lookup. SEM_UNDO has a per-task undo list manipulated under `task->sysvsem.undo_list_lock`.

POSIX mqueue uses a per-mqueue `mq_inode->i_lock` plus the standard wait-queue mechanism.

### Error handling

Standard errno set: ENOENT, EEXIST, EACCES, EIDRM (IPC_RMID destroyed), EINVAL, ENOMEM, EAGAIN (would block), ENOSPC (system limit), EFBIG (msgsize too large).

## Verification

### Layer 1: Kani SAFETY proofs
- SysV-shm shmat/shmdt mapping arithmetic
- SEM_UNDO undo-list manipulation across exec/exit
- mqueuefs inode lifecycle

### Layer 2: TLA+ models
- `models/ipc/sysv_sem_undo.tla` — proves SEM_UNDO atomicity: when a task exits with pending undo, all undo operations apply atomically.
- `models/ipc/mqueue_notify.tla` — proves `mq_notify` registration/delivery is exactly-once.

### Layer 3: Kani harnesses
- SysV ID-table invariants (each numeric ID maps to at most one object; refcounts are consistent)

### Layer 4: opt-in
- POSIX mqueue invariant proofs (FIFO ordering for equal-priority messages; priority ordering otherwise) — Verus tractable.

## Hardening

Placeholder per `00-overview.md` D6. Notable: SHM_HUGETLB cross-references mm/hugetlb.md; shmctl's IPC_RMID-on-attach semantics are a known surface.

## Open Questions

(none — small well-defined surface; per-Tier-3 docs may surface ambiguities)

## Out of Scope

- futex (kernel/-owned)
- AF_UNIX, pipes, eventfd (fs/- and net/-owned)
- 32-bit-only paths (consistent with arch v0)
- Implementation code
