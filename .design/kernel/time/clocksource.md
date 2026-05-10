# Tier-3: kernel/time/clocksource.c — Clocksource (per-CPU time source) selection + watchdog

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/time/00-overview.md
upstream-paths:
  - kernel/time/clocksource.c (~1600 lines)
  - include/linux/clocksource.h
-->

## Summary

Clocksource is the per-system monotonic time counter (TSC, HPET, ACPI-PM, jiffies, etc.). Per-driver registers via `clocksource_register_hz` / `_khz` / `_scale` with per-source rate + rating. Per-system picks highest-rated stable source as `curr_clocksource`. Per-watchdog (e.g. HPET) periodically validates against per-CPU TSC: detects drift / TSC-unstable; demotes-and-switches. Per-arch may use VDSO clock_gettime (fast-path) when source is VDSO-capable. Critical for: monotonic time-stamps (CLOCK_MONOTONIC), profilers, scheduling, NTP-disciplined clocks.

This Tier-3 covers `clocksource.c` (~1600 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct clocksource` | per-source state | `Clocksource` |
| `__clocksource_register_scale()` | per-driver register | `Clocksource::register_scale` |
| `clocksource_unregister()` | per-source remove | `Clocksource::unregister` |
| `clocksource_select()` | per-system pick best-rated | `Clocksource::select` |
| `clocksource_watchdog()` | per-iter sanity-check | `Clocksource::watchdog` |
| `clocksource_watchdog_kthread()` | per-detected-fault demote | `Clocksource::watchdog_kthread` |
| `clocksource_change_rating()` | per-source rate-update | `Clocksource::change_rating` |
| `clocksource_resume()` / `_suspend()` | per-PM | `Clocksource::resume` / `suspend` |
| `clocksource_arch_init()` | per-arch hook | `Clocksource::arch_init` |
| `__clocksource_select()` | per-mode (override / boot) | `Clocksource::__select` |
| `read_clocksource_*` (per-source) | per-read fn | shared |
| `clocksource_list` | per-system list | shared |
| `watchdog_list` | per-monitored list | shared |
| `curr_clocksource` | per-system active source | shared |
| `CLOCK_SOURCE_*` flags | per-source attribute | shared |

## Compatibility contract

REQ-1: struct clocksource:
- name: source name ("tsc", "hpet", "acpi_pm", "jiffies", ...).
- read: function pointer returning u64 cycle-counter.
- mask: u64 (bitwidth limit, e.g. (1<<56)-1 for 56-bit counter).
- mult, shift: per-(cycles → ns) precomputed conversion.
- rating: 1..500 (higher = better).
- list: ListLink in clocksource_list.
- flags: CLOCK_SOURCE_*.
- max_idle_ns: max ns before counter-wrap.

REQ-2: Per-rating bands:
- 1..99: unfit (jiffies).
- 100..199: base (sw counters).
- 200..299: usable (low-prec HW).
- 300..399: good (HPET, ACPI-PM).
- 400..499: best HW (TSC if stable).
- 500: must-use (overriding watchdog).

REQ-3: CLOCK_SOURCE_*:
- IS_CONTINUOUS: monotonic.
- VALID_FOR_HRES: high-res timer-capable.
- MUST_VERIFY: per-watchdog validation needed.
- SUSPEND_NONSTOP: continues across S3.
- UNSTABLE: marked unstable by watchdog.
- RESELECT: per-flag for re-select on resume.

REQ-4: __clocksource_register_scale:
- Compute mult+shift from (freq, scale).
- Validate mask correctness.
- Insert into clocksource_list sorted by rating.
- if rating > curr_clocksource.rating: clocksource_select.

REQ-5: clocksource_select:
- Walk clocksource_list (sorted by rating desc).
- For each: check valid + !UNSTABLE.
- Pick top; set curr_clocksource.
- Call timekeeping_notify(new_source) to switch.

REQ-6: clocksource_watchdog (1-second timer):
- For each cs in watchdog_list:
  - Read cs.curr_cycles + watchdog.curr_cycles atomically.
  - Compute delta-ns from each.
  - If |cs_ns - wd_ns| > WATCHDOG_THRESHOLD: cs.flags |= UNSTABLE.
  - Queue watchdog_work to demote.

REQ-7: clocksource_watchdog_kthread:
- For each UNSTABLE cs:
  - clocksource_change_rating(cs, 0).
  - clocksource_select.

REQ-8: Per-VDSO clock_gettime:
- If curr_clocksource.archdata.vclock_mode == VCLOCK_TSC ∨ HVCLOCK ∨ PVCLOCK:
- VDSO bypasses syscall.

REQ-9: clocksource_resume_watchdog:
- After PM-resume: re-arm watchdog timer.

REQ-10: Per-userspace ABI:
- /sys/devices/system/clocksource/clocksource0/{current_clocksource, available_clocksource}.
- Echo to current_clocksource: override.

## Acceptance Criteria

- [ ] AC-1: __clocksource_register_scale(tsc, freq=3GHz, rating=400): tsc in list; curr_clocksource = tsc.
- [ ] AC-2: clocksource_select picks highest-rated stable source.
- [ ] AC-3: clocksource_watchdog detects TSC drift > threshold: TSC → UNSTABLE.
- [ ] AC-4: clocksource_watchdog_kthread demotes UNSTABLE: rating=0, re-select.
- [ ] AC-5: New higher-rated registered: re-select.
- [ ] AC-6: /sys/.../current_clocksource read: returns curr_clocksource.name.
- [ ] AC-7: /sys/.../current_clocksource write "hpet": override.
- [ ] AC-8: PM-suspend / resume: watchdog re-armed.
- [ ] AC-9: VCLOCK_TSC source: VDSO clock_gettime fast-path.
- [ ] AC-10: clocksource_register before timekeeping_init: deferred.

## Architecture

Per-clocksource:

```
struct Clocksource {
  read: fn(cs: &Clocksource) -> u64,
  mask: u64,
  mult: u32,
  shift: u32,
  max_idle_ns: u64,
  maxadj: u32,
  uncertainty_margin: u64,
  archdata: ArchClocksourceData,
  name: &'static str,
  list: ListLink,
  rating: i32,
  enable: fn(cs: &Clocksource) -> i32,
  disable: fn(cs: &Clocksource),
  flags: u32,                                    // CLOCK_SOURCE_*
  suspend: fn(cs: &Clocksource),
  resume: fn(cs: &Clocksource),
  mark_unstable: fn(cs: &Clocksource),
  tick_stable: fn(cs: &Clocksource),
  #[cfg(CONFIG_CLOCKSOURCE_WATCHDOG)]
  wd_list: ListLink,
  cs_last: u64,
  wd_last: u64,
}
```

Globals:

```
static CLOCKSOURCE_LIST: ListHead<Clocksource>;   // sorted by rating desc
static WATCHDOG_LIST: ListHead<Clocksource>;      // CLOCK_SOURCE_MUST_VERIFY
static CURR_CLOCKSOURCE: *Clocksource;
static WATCHDOG: *Clocksource;                    // designated watchdog (e.g. HPET)
static WATCHDOG_TIMER: TimerList;
static CLOCKSOURCE_MUTEX: Mutex<()>;
```

`Clocksource::register_scale(cs, scale, freq) -> Result<()>`:
1. Compute mult + shift = clocks_calc_mult_shift(freq, NSEC_PER_SEC / scale, max_sec=600).
2. cs.mult = mult; cs.shift = shift.
3. cs.max_idle_ns = clocksource_max_deferment(cs).
4. mutex_lock(&clocksource_mutex).
5. /* Insert sorted by rating desc */
6. list_add_priority(&cs.list, &CLOCKSOURCE_LIST).
7. /* Add to watchdog list if needed */
8. if cs.flags & CLOCK_SOURCE_MUST_VERIFY: list_add(&cs.wd_list, &WATCHDOG_LIST).
9. Clocksource::select.
10. mutex_unlock.

`Clocksource::select()`:
1. /* Walk CLOCKSOURCE_LIST (sorted) */
2. best = NULL.
3. for cs in CLOCKSOURCE_LIST:
   - if cs.rating == 0 ∨ cs.flags & UNSTABLE: continue.
   - best = cs; break.
4. if best && best != CURR_CLOCKSOURCE:
   - CURR_CLOCKSOURCE = best.
   - timekeeping_notify(best).

`Clocksource::watchdog(timer)` (1s periodic):
1. for cs in WATCHDOG_LIST:
   - cs_now = cs.read(cs).
   - wd_now = WATCHDOG.read(WATCHDOG).
   - cs_delta = clocksource_cyc2ns(cs_now - cs.cs_last, cs.mult, cs.shift).
   - wd_delta = clocksource_cyc2ns(wd_now - cs.wd_last, WATCHDOG.mult, WATCHDOG.shift).
   - if |cs_delta - wd_delta| > WATCHDOG_THRESHOLD_NSEC:
     - cs.flags |= UNSTABLE.
     - queue_work(&watchdog_work).
   - cs.cs_last = cs_now; cs.wd_last = wd_now.
2. mod_timer(&WATCHDOG_TIMER, jiffies + HZ).

`Clocksource::watchdog_kthread(data)` (deferred):
1. for cs in WATCHDOG_LIST:
   - if cs.flags & UNSTABLE:
     - Clocksource::change_rating(cs, 0).
2. Clocksource::select.

`Clocksource::change_rating(cs, rating)`:
1. mutex_lock(&clocksource_mutex).
2. list_del(&cs.list).
3. cs.rating = rating.
4. list_add_priority(&cs.list, &CLOCKSOURCE_LIST).
5. Clocksource::select.
6. mutex_unlock.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rating_in_range` | INVARIANT | cs.rating ∈ [0, 500]. |
| `list_sorted_by_rating` | INVARIANT | CLOCKSOURCE_LIST[i].rating ≥ [i+1].rating. |
| `curr_clocksource_in_list` | INVARIANT | CURR_CLOCKSOURCE ∈ CLOCKSOURCE_LIST. |
| `watchdog_in_list_or_null` | INVARIANT | WATCHDOG ∈ CLOCKSOURCE_LIST ∨ NULL. |
| `unstable_cs_lower_rating` | INVARIANT | (cs.flags & UNSTABLE) ⟹ cs.rating == 0 (after demotion). |

### Layer 2: TLA+

`kernel/time/clocksource.tla`:
- Per-register + per-select + per-watchdog demote.
- Properties:
  - `safety_curr_is_best` — per-select: curr_clocksource is highest-rated stable.
  - `safety_unstable_eventually_demoted` — per-watchdog detects UNSTABLE ⟹ demoted within bounded time.
  - `liveness_no_starvation` — per-stable source: registered ⟹ eventually selected.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Clocksource::register_scale` post: cs in list; mult/shift computed | `Clocksource::register_scale` |
| `Clocksource::select` post: CURR_CLOCKSOURCE = best-rated stable | `Clocksource::select` |
| `Clocksource::watchdog` post: per-drift > threshold cs.UNSTABLE set | `Clocksource::watchdog` |
| `Clocksource::watchdog_kthread` post: UNSTABLE-cs rating=0; re-selected | `Clocksource::watchdog_kthread` |

### Layer 4: Verus/Creusot functional

`Per-system best-rated stable clocksource selected at-register / at-watchdog; per-VDSO fast-path for clock_gettime` semantic equivalence: per-Documentation/timers/clocksource.rst.

## Hardening

(Inherits row-1 features from `kernel/time/00-overview.md` § Hardening.)

Clocksource reinforcement:

- **Per-mult/shift bounded by max_idle_ns** — defense against per-counter-wrap timekeeping error.
- **Per-watchdog drift detection** — defense against per-TSC-skew silent.
- **Per-mutex serializes list mutation** — defense against per-mutate race.
- **Per-watchdog kthread deferred** — defense against per-watchdog-timer recursion.
- **Per-UNSTABLE flag persists** — defense against per-re-select using broken cs.
- **Per-VDSO mode validated** — defense against per-arch unsupported clock-mode.
- **Per-PM resume re-arms watchdog** — defense against per-resume stale watchdog.
- **Per-/sys override CAP_SYS_ADMIN** — defense against per-unprivileged clocksource-override.
- **Per-source flags validated at register** — defense against per-driver malformed-flags.
- **Per-rating override capped** — defense against per-driver rating-bomb.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/time/timekeeping.c (covered in `00-overview.md` if expanded)
- kernel/time/timer.c (covered in `timer.md` Tier-3)
- kernel/time/hrtimer.c (covered in `hrtimer.md` Tier-3)
- VDSO clock_gettime (covered in `arch/x86/00-overview.md` separately)
- Per-driver clocksource (drivers/clocksource/; covered separately)
- Implementation code
