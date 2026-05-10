# Tier-3: arch/x86/kvm/i8254.c — KVM in-kernel 8254 PIT (Programmable Interval Timer) emulation

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/i8254.c (~816 lines)
  - arch/x86/kvm/i8254.h
-->

## Summary

Per-VM in-kernel emulated Intel-8254 PIT (Programmable Interval Timer) with 3 channels at I/O 0x40-0x43. Channel 0 → IRQ0 (system clock; pre-HPET legacy timer). Channel 1 → DRAM refresh (legacy; not used). Channel 2 → PC speaker output. Per-channel: counter (16-bit) + count_load_time + mode (0..5) + bcd + gate + read/write state + status. Per-channel HRTimer fires at programmed rate; per-fire injects IRQ0 (channel 0) or toggles speaker bit (channel 2). Per-VM PIT instance allocated on KVM_CREATE_PIT2 ioctl. Reinject mode handles missed ticks for catch-up. Critical for: pre-HPET legacy guest software (DOS / Windows 2000 / older boot loaders).

This Tier-3 covers `i8254.c` (~816 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_pit` | per-VM PIT state | `KvmPit` |
| `struct kvm_kpit_state` | per-VM PIT 3 channels | `KvmKpitState` |
| `struct kvm_kpit_channel_state` | per-channel state | `KvmKpitChannelState` |
| `kvm_create_pit()` | per-VM PIT alloc | `Pit::create` |
| `kvm_free_pit()` | per-VM PIT free | `Pit::free` |
| `pit_load_count()` | per-channel load counter | `Pit::load_count` |
| `pit_get_count()` | per-channel read counter | `Pit::get_count` |
| `pit_get_out()` | per-channel out-line state | `Pit::get_out` |
| `pit_latch_count()` | per-channel latch counter for read | `Pit::latch_count` |
| `pit_latch_status()` | per-channel latch status | `Pit::latch_status` |
| `pit_ioport_read()` / `_write()` | per-port dispatch | `Pit::ioport_read` / `ioport_write` |
| `speaker_ioport_read()` / `_write()` | per-port 0x61 (speaker) | `Pit::speaker_read` / `speaker_write` |
| `create_pit_timer()` | per-channel HRTimer arm | `Pit::create_timer` |
| `pit_do_work()` | per-fire workqueue handler | `Pit::do_work` |
| `kvm_pit_ack_irq()` | per-IRQ-ack reinject handler | `Pit::ack_irq` |
| `kvm_pit_set_reinject()` | per-VM reinject mode toggle | `Pit::set_reinject` |
| `KVM_PIT_FREQ` (1193182 Hz) | per-PIT input clock | shared |

## Compatibility contract

REQ-1: I/O port layout:
- 0x40-0x43: PIT channels 0/1/2 + control word.
- 0x61: PC speaker port (bit 0 = ch2 gate; bit 1 = speaker enable; bit 4 = refresh; bit 5 = ch2 out).

REQ-2: Per-channel state:
- count: 16-bit current counter.
- count_load_time: ktime when counter loaded.
- mode: 0..5 (interrupt-on-terminal-count / hardware-retriggerable-one-shot / rate-generator / square-wave / software-triggered-strobe / hardware-triggered-strobe).
- bcd: 1 = BCD count; 0 = binary.
- gate: 1 = enabled.
- rw_state: WRITE_LSB / WRITE_MSB / WRITE_WORD0 / WRITE_WORD1 / READ_LSB / READ_MSB.
- read_state / write_state: independent from rw_state.
- latched_count: latched value for read.
- count_latched: 0=not / 1=lsb-latched / 2=msb-latched / 3=word-latched.
- status_latched: 1 if status latched.
- status: latched status byte.

REQ-3: Per-control-word at 0x43:
- Bits[7:6] = channel select (00=ch0, 01=ch1, 10=ch2, 11=read-back).
- Bits[5:4] = access mode (00=latch, 01=lsb, 10=msb, 11=word).
- Bits[3:1] = mode (000=mode0, 001=mode1, 010=mode2, 011=mode3, 100=mode4, 101=mode5).
- Bit[0] = BCD.

REQ-4: Mode 2 (Rate Generator):
- Most common for ch0: per-counter-decrement-to-1 → output goes low; counter reloaded; output back high.
- Triggers IRQ0 at rate KVM_PIT_FREQ / count Hz.

REQ-5: Mode 3 (Square Wave):
- ch2 (speaker) typical: per-(count/2) cycles toggle output.

REQ-6: Per-channel HRTimer:
- create_pit_timer(pit, ch, val, mode):
  - period = (val * NSEC_PER_SEC) / KVM_PIT_FREQ.
  - hrtimer_start(&pit.timer, ktime_add_ns(now, period), HRTIMER_MODE_ABS).
- Per-fire: pit_do_work invoked via workqueue.

REQ-7: pit_do_work flow:
- For ch0: kvm_set_irq(kvm, irq_source_id, 0, 1); + later kvm_set_irq(0, 0).
- For ch2: toggle speaker bit (latched in 0x61).

REQ-8: Per-IRQ ack notifier:
- Per-VM kvm_irq_ack_notifier registered.
- Per-IRQ0-ack: pit.pending--; if reinject ∧ pending > 0: re-fire.

REQ-9: Per-reinject mode:
- KVM_PIT_REINJECT_DISABLE / ENABLE.
- ENABLE: missed-ticks tracked via pit.pending; per-EOI re-injects.
- DISABLE: pit.pending always 1; lost ticks dropped.

REQ-10: Per-VM userspace ABI:
- KVM_CREATE_PIT2: per-VM PIT alloc with config.
- KVM_GET_PIT2 / KVM_SET_PIT2: per-channel migration.
- KVM_PIT_FLAGS: KVM_PIT_FLAGS_HPET_LEGACY (suppress when HPET legacy-replacement-route active).

REQ-11: Per-port-bus integration:
- Registered on KVM_PIO_BUS at ports 0x40..0x43 + 0x61.

## Acceptance Criteria

- [ ] AC-1: KVM_CREATE_PIT2: per-VM PIT allocated; ports 0x40-0x43 + 0x61 registered.
- [ ] AC-2: Guest writes ctrl 0x36 to 0x43: ch0, mode 3, lsb+msb access, binary.
- [ ] AC-3: Guest writes counter 0x18FF (4863 → ~245.4 Hz) to 0x40 via lsb+msb: timer armed at 245 Hz.
- [ ] AC-4: HRTimer fires at programmed rate; IRQ0 delivered.
- [ ] AC-5: Guest reads ch0 counter via 0x40: latch_count returns elapsed-aware count.
- [ ] AC-6: Reinject mode ENABLE: per-vCPU paused → after resume, missed ticks delivered.
- [ ] AC-7: ch2 + speaker bit set (0x61.bit1=1): periodic toggle at programmed rate.
- [ ] AC-8: KVM_PIT_FLAGS_HPET_LEGACY: ch0 IRQ0 suppressed.
- [ ] AC-9: KVM_GET_PIT2 returns 3 channels' state.
- [ ] AC-10: KVM_SET_PIT2 restores per-VM PIT state.
- [ ] AC-11: kvm-unit-tests `pit` test passes.

## Architecture

Per-channel state:

```
struct KvmKpitChannelState {
  count: u32,                                    // 16-bit (0 = 65536)
  latched_count: u16,
  count_latched: u8,                             // 0 / LSB / MSB / WORD
  status_latched: u8,
  status: u8,
  read_state: u8,
  write_state: u8,
  write_latch: u8,
  rw_mode: u8,
  mode: u8,                                      // 0..5
  bcd: u8,
  gate: u8,
  count_load_time: KtimeT,
}
```

Per-VM PIT state:

```
struct KvmKpitState {
  channels: [KvmKpitChannelState; 3],
  flags: u32,                                    // KVM_PIT_FLAGS_*
  is_periodic: bool,
  inject_lock: SpinLock,
  pit_timer: HrTimer,
  pit: &mut KvmPit,
}

struct KvmPit {
  pit_state: KvmKpitState,
  base_address: u32,                             // 0x40
  speaker_dummy: u32,
  irq_source_id: i32,
  irq_ack_notifier: KvmIrqAckNotifier,
  reinject: bool,
  pending: AtomicU32,                             // missed ticks
  irq_ack: AtomicU32,
  expired: WorkStruct,                            // workqueue
  worker: KthreadWorker,
  worker_task: KThreadHandle,
  kvm: &Kvm,
  speaker_dev: KvmIoDevice,
  pit_dev: KvmIoDevice,
}
```

`Pit::create(kvm, flags) -> Result<()>`:
1. Allocate KvmPit; init each channel with mode 0; gate = 1 for ch0.
2. Register kvm_io_bus(KVM_PIO_BUS, 0x40, 4, &pit_dev).
3. Register kvm_io_bus(KVM_PIO_BUS, 0x61, 1, &speaker_dev).
4. Register IRQ ack notifier.
5. Create per-VM kthread worker.
6. kvm.arch.vpit = pit.

`Pit::load_count(pit, ch_idx, val)`:
1. ch = pit.pit_state.channels[ch_idx].
2. ch.count = val.
3. ch.count_load_time = ktime_get().
4. If ch_idx == 0: per-mode arm hrtimer.
   - period = (val * NSEC_PER_SEC) / KVM_PIT_FREQ.
   - is_periodic = (mode == 2 ∨ mode == 3).
   - hrtimer_start(&pit.pit_timer, period, HRTIMER_MODE_ABS).

`Pit::get_count(pit, ch) -> u32`:
1. elapsed_ns = ktime_get() - ch.count_load_time.
2. elapsed_ticks = elapsed_ns * KVM_PIT_FREQ / NSEC_PER_SEC.
3. switch ch.mode:
   - 0 / 1: count -= elapsed_ticks (clamp 0).
   - 2: (count - elapsed_ticks) % count.
   - 3: ((count - 2*elapsed_ticks) % count + count) % count.
   - 4 / 5: count - elapsed_ticks (clamp 0).
4. Return result.

`Pit::ioport_write(pit, addr, val)`:
1. switch (addr - 0x40):
   - 0/1/2 (channel data): write per-rw_state (LSB/MSB/WORD).
   - 3 (control): decode ch + access + mode; init per-channel state.
4. Per-counter write complete: pit_load_count(pit, ch).

`Pit::do_work(work)`:
1. pit.pending++.
2. if !pit.reinject ∧ pit.pending > 1: pit.pending = 1.
3. kvm_set_irq(kvm, irq_source_id, 0, 1).
4. kvm_set_irq(kvm, irq_source_id, 0, 0).

`Pit::ack_irq(notifier)`:
1. pit.pending--.
2. if pit.pending > 0 ∧ pit.reinject: schedule_work(&pit.expired).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `channel_idx_lt_3` | INVARIANT | per-channel-access ch_idx < 3. |
| `count_le_65536` | INVARIANT | per-channel.count ≤ 65536. |
| `mode_le_5` | INVARIANT | per-channel.mode ∈ [0, 5]. |
| `pending_zero_in_no_reinject` | INVARIANT | !reinject ⟹ pending ≤ 1. |
| `hrtimer_period_positive` | INVARIANT | per-armed hrtimer period > 0. |

### Layer 2: TLA+

`virt/kvm/pit.tla`:
- Per-channel load + timer fire + IRQ0 delivery + ack.
- Properties:
  - `safety_irq_ack_decrements_pending` — per-IRQ ack: pending--.
  - `safety_periodic_timer_repeats` — per-mode 2/3: timer rearms after fire.
  - `liveness_pending_eventually_drained` — per-reinject + ack ⟹ pending → 0.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Pit::load_count` post: ch.count == val; hrtimer armed iff ch0 | `Pit::load_count` |
| `Pit::get_count` post: returned count consistent with elapsed-ns + mode | `Pit::get_count` |
| `Pit::do_work` post: pit.pending incremented; IRQ0 toggled | `Pit::do_work` |
| `Pit::ack_irq` post: pit.pending--; rearm if reinject ∧ pending > 0 | `Pit::ack_irq` |

### Layer 4: Verus/Creusot functional

`Per-channel-load with mode 2 + count N → IRQ0 fires at KVM_PIT_FREQ/N Hz` semantic equivalence: per-PIT matches Intel 8254 datasheet.

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

PIT-specific reinforcement:

- **Per-VM PIT inject_lock serializes state mutation** — defense against concurrent fire/ack corruption.
- **Per-channel mode validated** — defense against guest setting mode > 5.
- **Per-counter load 0 = 65536** — defense against per-zero-divisor in period calculation.
- **Per-hrtimer period bounded** — defense against integer-overflow in period.
- **Per-VM PIT restricted to KVM_CREATE_PIT2** — defense against accidental allocation.
- **Per-non-reinject pending capped at 1** — defense against pending unbounded growth.
- **Per-reinject pending bounded by `KVM_PIT_REINJECT_MAX`** — defense against malicious workload causing IRQ-storm.
- **Per-HPET-legacy flag suppresses ch0 IRQ** — defense against double-IRQ collision when guest uses HPET as legacy timer.
- **Per-VM PIT state migrated via KVM_GET_PIT2** — defense against post-migrate guest seeing wrong count/mode.
- **Per-channel speaker port 0x61 bit-2 ignored** — defense against guest-spec divergence.
- **Per-VM kthread worker bound to KVM module lifetime** — defense against UAF post-VM-destroy.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- IOAPIC (covered in `x86-ioapic.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- 8259 PIC (covered in `x86-pic.md` Tier-3)
- HPET (covered separately)
- Implementation code
