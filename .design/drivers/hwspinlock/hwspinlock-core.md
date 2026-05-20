# Tier-3: drivers/hwspinlock/hwspinlock_core.c — Hardware spinlock framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/hwspinlock/hwspinlock_core.c
  - drivers/hwspinlock/hwspinlock_internal.h
  - include/linux/hwspinlock.h
-->

## Summary

The HW spinlock framework — generic abstraction over per-platform hardware-arbitrated locks used to serialize a shared resource (typically shared SRAM, IPC scratch registers, or shared peripherals) between heterogeneous cores (AP + DSP/M-class/modem) where neither side can use Linux-only software synchronization. Per-platform implementations (TI OMAP, Qualcomm tcsr/MSM, SPRD, STM32, Allwinner sun6i) supply `hwspinlock_ops::{trylock, unlock, relax, bust}`, and the core wraps them with per-lock software spinlock (to disable preemption + serialize across local CPUs), memory barriers, and timeout handling.

This Tier-3 covers `hwspinlock_core.c` (~892 lines: per-bank register, per-lock request/free, `__hwspin_trylock`/`__hwspin_lock_timeout`/`__hwspin_unlock` in HWLOCK_IRQSTATE/IRQ/RAW/IN_ATOMIC/default modes, `hwspin_lock_bust`, radix-tree indexed by global lock id, devm wrappers, OF-by-name lookup).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct hwspinlock_device` | per-bank descriptor (dev, ops, base_id, num_locks, lock[]) | `drivers::hwspinlock::Bank` |
| `struct hwspinlock` | per-lock instance (software lock + bank backref + priv) | `drivers::hwspinlock::HwSpinlock` |
| `struct hwspinlock_ops` | per-bank ops vtable (trylock, unlock, relax, notify, bust) | `drivers::hwspinlock::Ops` |
| `hwspin_lock_register(bank, dev, ops, base_id, num_locks)` | per-bank register | `Bank::register` |
| `hwspin_lock_unregister(bank)` | per-bank unregister (refuses while any lock in use) | `Bank::unregister` |
| `devm_hwspin_lock_register(dev, bank, ops, base_id, num_locks)` | devm wrapper | `Bank::devm_register` |
| `hwspin_lock_request_specific(id)` | acquire a specific lock id (radix-tree lookup) | `HwSpinlock::request_specific` |
| `hwspin_lock_free(hwlock)` | release a previously-requested lock | `HwSpinlock::free` |
| `devm_hwspin_lock_request_specific(dev, id)` / `_free` | devm wrappers | `HwSpinlock::devm_*` |
| `__hwspin_trylock(hwlock, mode, *flags)` | non-blocking acquire | `HwSpinlock::trylock` |
| `__hwspin_lock_timeout(hwlock, to, mode, *flags)` | acquire with msec timeout | `HwSpinlock::lock_timeout` |
| `__hwspin_unlock(hwlock, mode, *flags)` | release | `HwSpinlock::unlock` |
| `hwspin_lock_bust(hwlock, id)` | break stale lock held by remote id (best-effort) | `HwSpinlock::bust` |
| `of_hwspin_lock_get_id(np, index)` / `_byname(np, name)` | DT phandle-based id resolution | `Bank::of_get_id` / `_of_get_id_byname` |
| `hwspinlock_tree` (RADIX_TREE) | global lock-id → `hwspinlock*` map | `Subsystem::tree` |
| `HWSPINLOCK_UNUSED` tag | radix-tree tag marking unused locks | `Subsystem::unused_tag` |
| `HWSPINLOCK_RETRY_DELAY_US` (=100) | atomic-context retry delay | `consts::HWSPINLOCK_RETRY_DELAY_US` |

## Compatibility contract

REQ-1: Each lock indexed by global integer id = `bank->base_id + intra_bank_index`; ids are continuous within a bank and unique across all registered banks.

REQ-2: Per-lock mode set:
- `HWLOCK_IRQSTATE` — disable IRQs + save state on lock, restore on unlock.
- `HWLOCK_IRQ` — disable IRQs unconditionally, re-enable on unlock.
- `HWLOCK_RAW` — caller manages preemption (e.g. caller already holds a sleeping lock).
- `HWLOCK_IN_ATOMIC` — caller already in atomic context; timeout enforced via udelay.
- default — disable preemption only (spin_trylock).

REQ-3: `__hwspin_trylock` semantics — first take per-lock software spinlock (preempt-disable / IRQ-state as per mode), then call `ops->trylock`; if hardware lock fails, undo software lock and return -EBUSY; on success, emit `mb()` after acquire.

REQ-4: `__hwspin_lock_timeout` — loop on -EBUSY with `ops->relax` (if supplied) between attempts; expire either by jiffies (process context) or by cumulative `udelay(HWSPINLOCK_RETRY_DELAY_US)` (HWLOCK_IN_ATOMIC).

REQ-5: `__hwspin_unlock` — emit `mb()` before unlock, then `ops->unlock`, then undo per-mode software lock (irqrestore / irq-enable / preempt-enable).

REQ-6: `hwspin_lock_request_specific` is the only request entry; the framework intentionally has no "any free" anonymous request — every caller is expected to know the platform-specific lock id (via DT or platform code).

REQ-7: `hwspin_lock_unregister` refuses (-EBUSY) per-lock if the radix-tree HWSPINLOCK_UNUSED tag is clear (lock still requested).

REQ-8: `ops->trylock` and `ops->unlock` are mandatory at register time; `ops->relax`, `ops->bust`, `ops->notify` optional.

REQ-9: Per-bank module-pin: `try_module_get(bank->dev->driver->owner)` on request, `module_put` on free — prevents unloading bank driver while a lock is held.

REQ-10: Per-bank pm_runtime: `pm_runtime_get_sync` on request (best-effort, -EACCES tolerated), `pm_runtime_put` on free.

REQ-11: DT helpers — `of_hwspin_lock_get_id(np, index)` parses `hwlocks` property + `#hwlock-cells`; `of_hwspin_lock_get_id_byname(np, name)` looks up by `hwlock-names` index then dispatches.

REQ-12: `hwspin_lock_bust(hwlock, remote_id)` is opt-in (-EOPNOTSUPP if `ops->bust` absent); when supplied, breaks a stuck lock owned by `remote_id` (out-of-band recovery, e.g. dead remote-core scenario).

## Acceptance Criteria

- [ ] AC-1: TI OMAP4/5 boot probes `omap_hwspinlock`; `/sys/kernel/debug/hwspinlock/` (if debugfs enabled) lists 32 locks.
- [ ] AC-2: Qualcomm boot probes `qcom_hwspinlock`; remoteproc/ipc subsystems acquire/release locks across AP↔SMD/RPM IPC.
- [ ] AC-3: `__hwspin_trylock` in HWLOCK_IRQSTATE returns 0 on first acquire and -EBUSY on second concurrent attempt without unlock.
- [ ] AC-4: `__hwspin_lock_timeout(hwlock, 100, HWLOCK_IN_ATOMIC, NULL)` returns -ETIMEDOUT after ~100ms when lock is held.
- [ ] AC-5: Module containing the bank driver cannot be unloaded while a per-lock request is outstanding; succeeds after `hwspin_lock_free`.
- [ ] AC-6: `hwspin_lock_unregister` returns -EBUSY when any lock is still requested.
- [ ] AC-7: `of_hwspin_lock_get_id_byname(np, "ipc")` resolves the named entry via `hwlock-names`.

## Architecture

```
struct Bank {
  dev: &Device,
  ops: &'static Ops,
  base_id: i32,
  num_locks: i32,
  lock: [HwSpinlock; num_locks],       // flexible-array tail of bank descriptor
}

struct HwSpinlock {
  lock: SpinLock<()>,                   // software lock disables preempt/IRQ
  bank: &Bank,
  priv: *mut (),                        // per-impl private (e.g. MMIO offset)
}

struct Ops {
  trylock: fn(&HwSpinlock) -> bool,
  unlock:  fn(&HwSpinlock),
  relax:   Option<fn(&HwSpinlock)>,    // cpu_relax-style backoff
  notify:  Option<fn(&HwSpinlock, int)>,
  bust:    Option<fn(&HwSpinlock, unsigned int /* remote_id */) -> i32>,
}

// Global registry
static hwspinlock_tree: RadixTree<i32, &HwSpinlock>;
static hwspinlock_tree_lock: Mutex<()>;
const HWSPINLOCK_UNUSED: Tag = 0;
const HWSPINLOCK_RETRY_DELAY_US: u32 = 100;
```

Per-bank register `hwspin_lock_register(bank, dev, ops, base_id, num_locks)`:
1. Validate non-NULL: bank/ops/dev/ops->trylock/ops->unlock; num_locks>0.
2. Per-lock: spin_lock_init(&hwlock->lock); hwlock->bank=bank; insert into radix tree under id = base_id+i; tag HWSPINLOCK_UNUSED.
3. On any insert failure, rollback by removing already-inserted entries.

Per-lock request `hwspin_lock_request_specific(id)`:
1. Take `hwspinlock_tree_lock`.
2. radix_tree_lookup(id); -EINVAL on miss.
3. Check HWSPINLOCK_UNUSED tag present; if cleared, -EBUSY.
4. `__hwspin_lock_request(hwlock)` — try_module_get owner, pm_runtime_get_sync, clear HWSPINLOCK_UNUSED.
5. Return `hwlock`.

Per-lock trylock `__hwspin_trylock(hwlock, mode, *flags)`:
1. Per mode: `spin_trylock{_irq, _irqsave}` (sets *flags for IRQSTATE) or noop for RAW/IN_ATOMIC.
2. If software lock fails, -EBUSY.
3. `ret = ops->trylock(hwlock)`; on false, undo software lock + return -EBUSY.
4. `mb()` after acquire.
5. Return 0.

Per-lock timeout `__hwspin_lock_timeout(hwlock, to, mode, *flags)`:
1. `expire = msecs_to_jiffies(to) + jiffies`; `atomic_delay = 0`.
2. Loop: trylock; if not -EBUSY, return.
3. If HWLOCK_IN_ATOMIC: `udelay(HWSPINLOCK_RETRY_DELAY_US)`; `atomic_delay += 100`; if `atomic_delay > to*1000`, -ETIMEDOUT.
4. Else: if jiffies past expire, -ETIMEDOUT.
5. Call `ops->relax` if present.

Per-lock unlock `__hwspin_unlock(hwlock, mode, *flags)`:
1. `mb()` before release.
2. `ops->unlock(hwlock)`.
3. Per mode: spin_unlock_{irqrestore/irq/}/noop.

DT lookup `of_hwspin_lock_get_id(np, index)`:
1. `of_parse_phandle_with_args(np, "hwlocks", "#hwlock-cells", index, &args)`.
2. RCU-walk hwspinlock_tree to find bank whose `dev` matches `args.np`.
3. Translate via `of_hwspin_lock_simple_xlate` (args_count == 1 required).
4. Validate `id ∈ [0, bank->num_locks)`; return `base_id + id`.

## Hardening

(Inherits row-1 features from `drivers/bus/00-overview.md` § Hardening.)

hwspinlock-core specific reinforcement:

- **Mandatory `trylock` + `unlock` ops** at register — defeats partial-vtable null-deref.
- **Per-mode flags-NULL guard** — `WARN_ON(!flags && mode==HWLOCK_IRQSTATE)` rejects invalid mode/flags pairing in both trylock and unlock.
- **Per-lock `bank` backref** is set at register and never mutated — defeats per-lock bank-pointer-confusion.
- **Module pin via `try_module_get(owner)`** — defeats unload-while-locked use-after-free.
- **Pre-acquire/pre-release `mb()`** — required because software spinlock barriers are too early/late relative to the hardware lock observation window.
- **HWSPINLOCK_UNUSED tag guards unregister** — bank can never be released while a lock is still requested.
- **Atomic-mode timeout converted to cumulative udelay** — refuses unbounded busy-loop in IRQ-disabled contexts.
- **Radix-tree tree-lock as mutex (process-context only)** — request/free/register paths are not callable from atomic context, separating fast path from setup path.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab cache for `hwspinlock_device` (bank, allocated with flexible array of `hwspinlock` tail) and per-bank private regions; no userspace surface in framework.
- **PAX_KERNEXEC** — hwspinlock core in W^X kernel text; per-bank `hwspinlock_ops` and DT-lookup helpers live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `__hwspin_trylock`, `__hwspin_lock_timeout`, `__hwspin_unlock`, and `hwspin_lock_bust`.
- **PAX_REFCOUNT** — saturating `refcount_t` on per-bank `dev` and per-driver-owner module refcount via `try_module_get`; overflow trap defeats request/free imbalance UAFs across bank teardown.
- **PAX_MEMORY_SANITIZE** — zero-on-free for bank descriptors (per-bank flexible tail of `hwspinlock`) so stale `priv` pointers and bank backrefs cannot bleed across re-register.
- **PAX_UDEREF** — SMAP/PAN enforced on every DT-side helper entry; framework helpers themselves take only kernel-owned `hwspinlock` pointers.
- **PAX_RAP / kCFI** — `hwspinlock_ops` (`trylock`/`unlock`/`relax`/`notify`/`bust`) marked `__ro_after_init` with kCFI-typed indirect dispatch; opcode dispatch in `__hwspin_trylock` cannot be redirected to a wrong-signature callback.
- **GRKERNSEC_HIDESYM** — gate kallsyms and per-bank register/base disclosure behind CAP_SYSLOG; suppress `%p` in any debugfs printing of `hwlock->priv`.
- **GRKERNSEC_DMESG** — restrict ETIMEDOUT, EBUSY-on-unregister, and request-after-free banners to CAP_SYSLOG so attackers cannot probe shared-memory lock contention via dmesg.
- **Lock-id PAX_USERCOPY** — `id` in `hwspin_lock_request_specific(id)` validated against radix-tree presence + per-bank `num_locks` range; -EINVAL on miss; no caller-supplied id reaches hardware op without bound check.
- **Timeout bound** — `to` in `__hwspin_lock_timeout` is an `unsigned int` msec; HWLOCK_IN_ATOMIC path enforces cumulative `atomic_delay` not exceeding `to*1000` µs; framework refuses unbounded spinning in atomic context.
- **`hwspinlock_ops` kCFI** — each platform driver registers a typed ops struct; mismatched signature (e.g. wrong arg count on `bust`) rejected by kCFI at register-time link.
- **IRQ-disable counted** — HWLOCK_IRQSTATE/HWLOCK_IRQ correctly pair `spin_{trylock,unlock}_irq{,save,restore}`; on hardware-trylock failure the software-side IRQ state is fully unwound before -EBUSY return, defeating IRQ-leak corruption.
- **Bank-unregister gate** — `hwspin_lock_unregister` checks HWSPINLOCK_UNUSED tag per lock; refuses release while any lock is still requested, defeating "unregister-while-locked" remote-core deadlock.
- **CAP_SYS_ADMIN** on any future debugfs `bust` surface — `bust` is a destructive recovery op that should never be triggered from unprivileged callers.

Rationale: HW spinlocks arbitrate shared resources across heterogeneous cores; an underflow on the module refcount, a missed `mb()`, an IRQ-state leak on the -EBUSY rollback path, or an unbounded atomic-context retry can wedge the IPC channel between AP and remote-cores (modem, DSP), which is often a security-critical boundary. RAP/kCFI on `ops`, refcount-overflow trapping on bank teardown, lock-id range enforcement, and cumulative-udelay bound turn the framework from "best-effort" into a structural enforcement boundary.

## Open Questions

(none at this Tier-3 level)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `lock_id_radix_present` | OOB | `hwspin_lock_request_specific(id)` succeeds iff radix-tree contains `id`. |
| `trylock_unwind` | INVARIANT | every -EBUSY return from `__hwspin_trylock` has fully unwound the per-mode software lock. |
| `unlock_mb_order` | ORDER | `mb()` issued before `ops->unlock`; `ops->trylock` success followed by `mb()` before return. |
| `atomic_timeout_bounded` | LIVENESS | HWLOCK_IN_ATOMIC path returns within `to+ε` ms even when the hardware lock is permanently held. |

### Layer 2: TLA+

`models/hwspinlock/cross_core.tla` (parent-declared): proves correctness of cross-core mutual exclusion between AP and a remote core when both observe `mb()` ordering around `ops->trylock`/`ops->unlock`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `hwspin_lock_unregister` post: no lock with HWSPINLOCK_UNUSED cleared is removed; returns -EBUSY otherwise | `Bank::unregister` |
| `__hwspin_lock_timeout(_, _, HWLOCK_IN_ATOMIC, _)` post: never sleeps; cumulative delay ≤ to*1000 µs + one retry granularity | `HwSpinlock::lock_timeout` |
| `__hwspin_lock_request` post: `try_module_get` succeeded iff function returned 0 | `Subsystem::lock_request` |

### Layer 4: Verus/Creusot functional

Acquire-release-acquire pattern: `lock(A) → unlock(A) → lock(A)` succeeds in HWLOCK_IRQSTATE mode with IRQ state correctly saved/restored across the cycle; encoded as Verus invariant chained with the per-bank `ops` postconditions.

## Out of Scope

- Per-platform implementations (OMAP, Qualcomm, SPRD, STM32, sun6i) — separate future Tier-3
- userland exposure (hwspinlock is kernel-only)
- 32-bit-only paths
- Implementation code
