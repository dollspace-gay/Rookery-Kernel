# Tier-3: drivers/char/random.c — kernel RNG (entropy pool + ChaCha20 CRNG)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/char/00-overview.md
upstream-paths:
  - drivers/char/random.c (~1712 lines)
  - include/linux/random.h
  - include/uapi/linux/random.h (RND* ioctls, GRND_* flags)
  - Documentation/admin-guide/kernel-parameters.txt (random.trust_cpu, random.trust_bootloader)
-->

## Summary

The **kernel RNG** mixes entropy from many sources (interrupt-timing fast pool, input device timing, disk I/O timing, hardware RNGs, RDSEED/RDRAND/RDTSC, bootloader-provided, latent_entropy plugin) into a Blake2s-keyed input pool, expands it into per-CPU ChaCha20 streams using a "fast key erasure" construction, and exposes user-visible interfaces: `getrandom(2)`, `/dev/random`, `/dev/urandom`. Per-`crng_init` state machine: `CRNG_EMPTY` → `CRNG_EARLY` (POOL_EARLY_BITS == POOL_READY_BITS/2 collected) → `CRNG_READY` (POOL_READY_BITS == BLAKE2S_HASH_SIZE*8 = 256 collected). Per-base_crng: global `{ key[CHACHA_KEY_SIZE], generation, spinlock }` reseeded every CRNG_RESEED_INTERVAL (60s). Per-CPU `struct crng`: cached key + generation + local_lock; bumps generation on global reseed forcing re-key. Per-CPU `struct fast_pool`: `unsigned long pool[4]` SipHash siphash-1-x state mixed from add_interrupt_randomness (irq number, cycle counter, instruction pointer). Per-input pool: `struct blake2s_ctx hash` + spinlock; HKDF-like extract via blake2s. Per-`get_random_bytes(buf, len)`: kernel API → `_get_random_bytes` → `crng_make_state` (per-CPU fast key erasure ChaCha20). Per-`get_random_{u8,u16,u32,u64}`: per-CPU batched 1.5 ChaCha blocks. Per-getrandom(2): GRND_NONBLOCK / GRND_RANDOM / GRND_INSECURE flags; default blocks until crng_ready. Per-`/dev/random`: blocks until ready (or O_NONBLOCK → EAGAIN). Per-`/dev/urandom`: never blocks (rate-limited "uninitialized urandom read" warning). Per-RDSEED/RDRAND: hooked via `arch_get_random_seed_longs` / `arch_get_random_longs`. Per-vDSO: optional vdso_k_rng_data path; bumped on each reseed. Critical for: cryptographic key generation, ASLR, syncookies, TCP ISNs, every kernel module needing randomness.

This Tier-3 covers `drivers/char/random.c` (~1712 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `enum { CRNG_EMPTY, CRNG_EARLY, CRNG_READY }` | per-state-machine | `CrngInit` |
| `crng_init` | per-current-state | `Random::crng_init` |
| `crng_is_ready` (static_branch) | per-fast-path | `Random::crng_is_ready` |
| `crng_init_wait` | per-wait queue | `Random::crng_init_wait` |
| `base_crng { key, generation, lock }` | per-global ChaCha key | `Random::base_crng` |
| `struct crng` (per-CPU) | per-CPU key cache | `Crng` |
| `crng_reseed()` | per-key-rotation | `Random::crng_reseed` |
| `crng_reseed_interval()` | per-interval (CRNG_RESEED_INTERVAL) | `Random::crng_reseed_interval` |
| `crng_fast_key_erasure()` | per-block expand | `Random::fast_key_erasure` |
| `crng_make_state()` | per-CPU ChaCha state | `Random::make_state` |
| `_get_random_bytes()` | per-internal extract | `Random::get_bytes_internal` |
| `get_random_bytes()` | per-kernel API | `Random::get_bytes` |
| `get_random_u8/u16/u32/u64()` | per-batched int | `Random::get_uN` |
| `__get_random_u32_below()` | per-bounded | `Random::get_u32_below` |
| `wait_for_random_bytes()` | per-block until ready | `Random::wait_for_random_bytes` |
| `rng_is_initialized()` | per-state query | `Random::is_initialized` |
| `execute_with_initialized_rng()` | per-callback once-ready | `Random::execute_with_initialized` |
| `input_pool { hash, lock, init_bits }` | per-entropy pool | `Random::input_pool` |
| `_mix_pool_bytes()` / `mix_pool_bytes()` | per-blake2s update | `Random::mix_pool_bytes` |
| `extract_entropy()` | per-HKDF-like seed extract | `Random::extract_entropy` |
| `credit_init_bits()` / `_credit_init_bits()` | per-bit credit | `Random::credit_init_bits` |
| `add_device_randomness()` | per-device unique | `Random::add_device_randomness` |
| `add_hwgenerator_randomness()` | per-hwrng | `Random::add_hwgenerator_randomness` |
| `add_bootloader_randomness()` | per-boot | `Random::add_bootloader_randomness` |
| `add_vmfork_randomness()` | per-VM-fork | `Random::add_vmfork_randomness` |
| `add_interrupt_randomness()` | per-irq mix | `Random::add_interrupt_randomness` |
| `add_input_randomness()` | per-input event | `Random::add_input_randomness` |
| `add_disk_randomness()` | per-disk I/O | `Random::add_disk_randomness` |
| `add_timer_randomness()` | per-event timing | `Random::add_timer_randomness` |
| `struct fast_pool` | per-CPU IRQ pool | `FastPool` |
| `fast_mix()` | per-SipHash mix | `Random::fast_mix` |
| `mix_interrupt_randomness()` | per-CPU timer-deferred drain | `Random::mix_interrupt_randomness` |
| `try_to_generate_entropy()` | per-timer-jitter | `Random::try_to_generate_entropy` |
| `random_pm_notification` | per-suspend/resume reseed | `Random::pm_notify` |
| `random_init_early()` / `random_init()` | per-boot init | `Random::init_early` / `init` |
| `random_prepare_cpu()` / `random_online_cpu()` | per-CPU-hotplug | `Random::cpu_prepare` / `cpu_online` |
| `SYSCALL_DEFINE3(getrandom, ...)` | per-syscall | `Random::sys_getrandom` |
| `random_read_iter()` / `urandom_read_iter()` | per-/dev/{random,urandom} read | `Random::read_iter` |
| `random_write_iter()` | per-write (mix only, no credit) | `Random::write_iter` |
| `random_poll()` | per-poll | `Random::poll` |
| `random_ioctl()` | per-RND* ioctl | `Random::ioctl` |
| `random_fasync()` | per-fasync (SIGIO on ready) | `Random::fasync` |
| `random_fops` / `urandom_fops` | per-cdev fops | `Random::random_fops` / `urandom_fops` |
| `rand_initialize_disk()` | per-disk timer_rand_state alloc | `Random::initialize_disk` |
| `arch_get_random_seed_longs()` / `arch_get_random_longs()` | per-CPU-RDSEED/RDRAND hook | `Random::arch_seed_longs` / `arch_longs` |
| `trust_cpu` / `trust_bootloader` | cmdline parameters | `Random::trust_cpu` / `trust_bootloader` |
| `random_table[]` (sysctl) | /proc/sys/kernel/random/ | `Random::sysctl_table` |

## Compatibility contract

REQ-1: enum crng_init state machine:
- CRNG_EMPTY = 0 — little to no entropy collected; `_get_random_bytes` may return weak values during boot.
- CRNG_EARLY = 1 — POOL_EARLY_BITS == 128 bits credited; base_crng key seeded from extract_entropy.
- CRNG_READY = 2 — POOL_READY_BITS == 256 bits credited; static_branch `crng_is_ready` enabled.
- Monotonic; only increases. Protected by `base_crng->lock`.
- `crng_ready()` macro: `static_branch_likely(&crng_is_ready) || crng_init >= CRNG_READY`.

REQ-2: struct base_crng (singleton):
- key: u8[CHACHA_KEY_SIZE = 32] __aligned(__alignof__(long)).
- generation: unsigned long — bumped on each reseed; ULONG_MAX is reserved invalid.
- lock: spinlock_t.

REQ-3: struct crng (per-CPU `crngs`):
- key: u8[CHACHA_KEY_SIZE = 32].
- generation: unsigned long, init = ULONG_MAX (forces first-use re-key from base).
- lock: local_lock_t.

REQ-4: crng_reseed:
- Optionally re-queued via `queue_delayed_work(system_dfl_wq, &next_reseed, crng_reseed_interval())`.
- extract_entropy(key, CHACHA_KEY_SIZE).
- spin_lock_irqsave(&base_crng.lock).
- memcpy(base_crng.key, key, 32).
- next_gen = base_crng.generation + 1; if next_gen == ULONG_MAX: ++next_gen.
- WRITE_ONCE(base_crng.generation, next_gen).
- if (CONFIG_VDSO_GETRANDOM): smp_store_release(&vdso_k_rng_data->generation, next_gen + 1).
- if (!crng_is_ready): crng_init = CRNG_READY.
- spin_unlock_irqrestore.
- memzero_explicit(key, 32).

REQ-5: crng_fast_key_erasure(key, chacha_state, random_data, len):
- BUG_ON(len > 32).
- chacha_init_consts(chacha_state).
- memcpy(&chacha_state.x[4], key, 32).
- memset(&chacha_state.x[12], 0, 16).
- chacha20_block(chacha_state, first_block).
- memcpy(key, first_block, 32) — key is overwritten in-place; forward secrecy.
- memcpy(random_data, first_block + 32, len).
- memzero_explicit(first_block, 64).

REQ-6: crng_make_state(chacha_state, random_data, len):
- BUG_ON(len > 32).
- If (!crng_ready()):
  - extract_entropy(chacha_state, 48) via base_crng-less fallback (early boot path).
  - Return.
- local_lock_irqsave(&crngs.lock).
- crng = raw_cpu_ptr(&crngs).
- If READ_ONCE(crng.generation) != READ_ONCE(base_crng.generation):
  - spin_lock(&base_crng.lock); memcpy(crng.key, base_crng.key, 32); spin_unlock.
  - crng.generation = base_crng.generation.
- crng_fast_key_erasure(crng.key, chacha_state, random_data, len).
- local_unlock_irqrestore.

REQ-7: _get_random_bytes(buf, len):
- chacha_state on stack.
- crng_make_state(&chacha_state, first_block_random_data, min(len, 32)).
- copy first up-to-32 bytes to buf.
- Loop with chacha20_block(&chacha_state, block) until len exhausted.
- memzero_explicit(chacha_state, sizeof).

REQ-8: get_random_bytes(buf, len): public API → `_get_random_bytes`.

REQ-9: DEFINE_BATCHED_ENTROPY(type) (instantiated for u8, u16, u32, u64):
- struct batch_<type>: `entropy[CHACHA_BLOCK_SIZE * 3 / (2*sizeof(type))]`, local_lock_t, generation, position.
- get_random_<type>():
  - If !crng_ready(): _get_random_bytes(&ret, sizeof(ret)).
  - local_lock_irqsave.
  - batch = this_cpu_ptr.
  - if position >= ARRAY_SIZE(entropy) || base_crng.generation != batch.generation:
    - _get_random_bytes(batch.entropy, sizeof batch.entropy).
    - position = 0; generation = base_crng.generation.
  - ret = batch.entropy[position]; batch.entropy[position] = 0; ++position.
  - local_unlock_irqrestore.
- Cleared on reseed (generation changes) to avoid stale randomness leakage.

REQ-10: input_pool:
- struct: blake2s_ctx hash (initialized with Blake2s IV ⊕ 0x01010000|outlen), spinlock_t lock, unsigned int init_bits.
- POOL_BITS = BLAKE2S_HASH_SIZE * 8 = 256.
- POOL_READY_BITS = POOL_BITS = 256.
- POOL_EARLY_BITS = POOL_READY_BITS / 2 = 128.
- mix_pool_bytes(buf, len): spin_lock_irqsave(&input_pool.lock); blake2s_update(&input_pool.hash, buf, len); spin_unlock_irqrestore.

REQ-11: extract_entropy(buf, len) (HKDF-like):
- block: { rdseed[32/sizeof(long)], counter }.
- Fill rdseed[i] from arch_get_random_seed_longs / arch_get_random_longs / random_get_entropy (cycle counter fallback).
- spin_lock_irqsave(&input_pool.lock).
- blake2s_final(&input_pool.hash, seed).
- block.counter = 0; blake2s(seed, sizeof seed, &block, sizeof block, next_key, BLAKE2S_HASH_SIZE).
- blake2s_init_key(&input_pool.hash, BLAKE2S_HASH_SIZE, next_key, BLAKE2S_HASH_SIZE). // reseed pool with next_key
- spin_unlock_irqrestore.
- While len > 0: ++block.counter; blake2s(seed, ..., buf, i); len -= i; buf += i.
- memzero_explicit(seed); memzero_explicit(&block).

REQ-12: _credit_init_bits(bits):
- add = min(bits, POOL_BITS).
- try_cmpxchg(&input_pool.init_bits, &orig, min(POOL_BITS, orig+add)).
- If orig < POOL_READY_BITS && new >= POOL_READY_BITS:
  - crng_reseed(NULL) → sets crng_init = CRNG_READY.
  - queue_work(system_dfl_wq, &set_ready) → static_branch_enable(&crng_is_ready).
  - atomic_notifier_call_chain(&random_ready_notifier, 0, NULL).
  - if CONFIG_VDSO_GETRANDOM: WRITE_ONCE(vdso_k_rng_data->is_ready, true).
  - wake_up_interruptible(&crng_init_wait).
  - kill_fasync(&fasync, SIGIO, POLL_IN).
  - pr_notice("crng init done\n").
- Else if orig < POOL_EARLY_BITS && new >= POOL_EARLY_BITS:
  - spin_lock(&base_crng.lock); if (crng_init == CRNG_EMPTY): extract_entropy(base_crng.key, 32); crng_init = CRNG_EARLY; spin_unlock.

REQ-13: struct fast_pool (per-CPU `irq_randomness`):
- pool: unsigned long[4] (init = SIPHASH_CONST_{0..3} or HSIPHASH_CONST_{0..3} on 32-bit).
- last: jiffies of last credit.
- count: count|MIX_INFLIGHT bit-31.
- mix: timer_list (callback = mix_interrupt_randomness).

REQ-14: add_interrupt_randomness(irq):
- entropy = random_get_entropy().
- regs = get_irq_regs().
- fast_mix(fast_pool.pool, entropy, (regs ? instruction_pointer(regs) : _RET_IP_) ^ swab(irq)).
- new_count = ++fast_pool.count.
- if (new_count & MIX_INFLIGHT): return.
- if (new_count < 1024 && !time_is_before_jiffies(fast_pool.last + HZ)): return.
- fast_pool.count |= MIX_INFLIGHT.
- if (!timer_pending(&fast_pool.mix)): expires = jiffies; add_timer_on(&fast_pool.mix, raw_smp_processor_id()).

REQ-15: mix_interrupt_randomness (deferred timer cb):
- If (fast_pool != this_cpu_ptr): bail (CPU hotplug raced).
- memcpy(pool[2], fast_pool.pool, 16).
- count = fast_pool.count; fast_pool.count = 0; fast_pool.last = jiffies.
- mix_pool_bytes(pool, 16) → blake2s_update.
- credit_init_bits(clamp((count & U16_MAX) / 64, 1, 128)).
- memzero_explicit(pool).

REQ-16: add_input_randomness(type, code, value):
- if value == last_value: skip autorepeat.
- add_timer_randomness(&input_timer_state, (type<<4) ^ code ^ (code>>4) ^ value).

REQ-17: add_disk_randomness(disk):
- if !disk->random: return.
- add_timer_randomness(disk->random, 0x100 + disk_devt(disk)).

REQ-18: add_timer_randomness(state, num):
- entropy = random_get_entropy(); now = jiffies.
- if (in_hardirq): fast_mix(this_cpu(irq_randomness).pool, entropy, num) [stash for later credit by mix_interrupt_randomness].
- else: spin_lock_irqsave(&input_pool.lock); _mix_pool_bytes(&entropy, …); _mix_pool_bytes(&num, …); unlock.
- if crng_ready: return.
- Compute first-, second-, third-order deltas; pick minimum absolute delta.
- bits = min(fls(delta>>1), 11).
- if in_hardirq: irq_randomness.count += max(1, bits*64) - 1.
- else: _credit_init_bits(bits).

REQ-19: add_device_randomness(buf, len): mix entropy+buf without crediting (initializes pool with device-unique values like MAC, RTC).

REQ-20: add_hwgenerator_randomness(buf, len, entropy, sleep_after):
- mix_pool_bytes(buf, len); credit_init_bits(entropy).
- If sleep_after && !kthread_should_stop && (crng_ready || !entropy): schedule_timeout_interruptible(crng_reseed_interval()).

REQ-21: add_bootloader_randomness(buf, len):
- mix_pool_bytes(buf, len).
- If trust_bootloader (cmdline `random.trust_bootloader=1`, default true): credit_init_bits(len*8).

REQ-22: add_vmfork_randomness(unique_vm_id, len):
- add_device_randomness(unique_vm_id, len) (no credit).
- if crng_ready: crng_reseed(NULL); pr_notice("crng reseeded due to virtual machine fork").
- blocking_notifier_call_chain(&vmfork_chain, 0, NULL).

REQ-23: random_init_early(command_line):
- Mix LATENT_ENTROPY_PLUGIN compile-time seed if enabled.
- Loop: try arch_get_random_seed_longs → arch_get_random_longs; mix_pool_bytes each.
- _mix_pool_bytes(init_utsname(), …); _mix_pool_bytes(command_line, strlen).
- if crng_ready: crng_reseed(NULL) else if trust_cpu: _credit_init_bits(arch_bits).

REQ-24: random_init() (post-time-keeping):
- entropy = random_get_entropy(); now = ktime_get_real(); mix both + latent_entropy.
- If crng_init >= CRNG_READY: crng_set_ready (enables static_branch).
- If crng_ready: crng_reseed(NULL).
- register_pm_notifier(&pm_notifier).
- WARN if !entropy (missing cycle counter).

REQ-25: random_pm_notification(nb, action, data):
- Mix `action`, ktime_get/_boottime/_real timestamps, random_get_entropy.
- If crng_ready && (PM_RESTORE_PREPARE || PM_POST_SUSPEND (no autosleep)): crng_reseed(NULL).

REQ-26: random_prepare_cpu (CPUHP_RANDOM_PREPARE):
- per_cpu(&crngs, cpu)->generation = ULONG_MAX (force re-key on next use).
- per_cpu(&batched_entropy_uN, cpu)->position = UINT_MAX.

REQ-27: random_online_cpu (CPUHP_AP_RANDOM_ONLINE):
- per_cpu(&irq_randomness, cpu)->count = 0 (clear MIX_INFLIGHT).

REQ-28: SYSCALL_DEFINE3(getrandom, ubuf, len, flags):
- Reject unknown flags ⊂ { GRND_NONBLOCK, GRND_RANDOM, GRND_INSECURE }.
- Reject GRND_INSECURE | GRND_RANDOM.
- If !crng_ready && !GRND_INSECURE:
  - If GRND_NONBLOCK: return -EAGAIN.
  - ret = wait_for_random_bytes(); on signal: return -ERESTARTSYS.
- import_ubuf(ITER_DEST, ubuf, len, &iter).
- Return get_random_bytes_user(&iter).

REQ-29: get_random_bytes_user(iter):
- crng_make_state(&chacha_state, &chacha_state.x[4], 32).
- If iov_iter_count <= 32: copy_to_iter(&chacha_state.x[4], 32, iter).
- Else: loop chacha20_block(&chacha_state, block); copy_to_iter; bump counter; cond_resched on PAGE_SIZE boundary.
- memzero_explicit(block); chacha_zeroize_state(&chacha_state).

REQ-30: random_fops / urandom_fops:
- random_fops: read_iter=random_read_iter, write_iter=random_write_iter, poll=random_poll, unlocked_ioctl=random_ioctl, compat_ioctl=compat_ptr_ioctl, fasync=random_fasync, llseek=noop_llseek, splice_read=copy_splice_read, splice_write=iter_file_splice_write.
- urandom_fops: same except read_iter=urandom_read_iter and no poll.

REQ-31: random_read_iter / urandom_read_iter:
- random_read_iter: if !crng_ready && (IOCB_NOWAIT|IOCB_NOIO|O_NONBLOCK): return -EAGAIN; wait_for_random_bytes; get_random_bytes_user.
- urandom_read_iter: try_to_generate_entropy if !crng_ready; rate-limited "uninitialized urandom read" warning; always reads (no blocking).

REQ-32: random_ioctl:
- RNDGETENTCNT: put_user(input_pool.init_bits).
- RNDADDTOENTCNT: CAP_SYS_ADMIN; get_user(ent_count); credit_init_bits.
- RNDADDENTROPY: CAP_SYS_ADMIN; get_user(ent_count, len); write_pool_user; credit_init_bits.
- RNDZAPENTCNT / RNDCLEARPOOL: CAP_SYS_ADMIN; no-op (kept for ABI).
- RNDRESEEDCRNG: CAP_SYS_ADMIN; if !crng_ready return -ENODATA; crng_reseed(NULL).

REQ-33: sysctl `kernel/random/`:
- poolsize (RO): POOL_BITS = 256.
- entropy_avail (RO): input_pool.init_bits.
- write_wakeup_threshold (RW, dummy): POOL_READY_BITS.
- urandom_min_reseed_secs (RW, dummy): CRNG_RESEED_INTERVAL / HZ = 60.
- boot_id (RO): proc_do_uuid (per-boot UUID).
- uuid (RO): proc_do_uuid (fresh UUID each read).

REQ-34: arch_get_random_seed_longs / arch_get_random_longs:
- RDSEED/RDRAND hook (x86); arm/arm64/powerpc/s390 wired via arch headers.
- Trusted if `trust_cpu=1` (default true) for boot-time credit.

## Acceptance Criteria

- [ ] AC-1: get_random_bytes never fails; returns from base_crng/per-CPU crng even pre-CRNG_READY (best-effort).
- [ ] AC-2: getrandom(2) without GRND_INSECURE blocks until crng_ready (or -EAGAIN with GRND_NONBLOCK).
- [ ] AC-3: getrandom(2) with GRND_INSECURE | GRND_RANDOM returns -EINVAL.
- [ ] AC-4: getrandom(2) with unknown flag bit returns -EINVAL.
- [ ] AC-5: /dev/random read blocks until crng_ready unless O_NONBLOCK / IOCB_NOWAIT (returns -EAGAIN).
- [ ] AC-6: /dev/urandom read never blocks; emits rate-limited warning if !crng_ready.
- [ ] AC-7: poll(/dev/random) → EPOLLIN when crng_ready; EPOLLOUT always (writable for entropy submission).
- [ ] AC-8: write to /dev/random or /dev/urandom mixes into pool without crediting (no entropy gain).
- [ ] AC-9: RNDADDENTROPY: requires CAP_SYS_ADMIN; mixes + credits.
- [ ] AC-10: RNDRESEEDCRNG: requires CAP_SYS_ADMIN; returns -ENODATA if !crng_ready.
- [ ] AC-11: crng_init transitions only forward (EMPTY → EARLY → READY).
- [ ] AC-12: crng_reseed bumps base_crng.generation (skipping ULONG_MAX); per-CPU crngs re-key on next use.
- [ ] AC-13: add_interrupt_randomness defers crediting via timer; mix_interrupt_randomness runs on the originating CPU.
- [ ] AC-14: CPU hotplug: random_prepare_cpu invalidates per-CPU crng + batches; random_online_cpu clears fast_pool count.
- [ ] AC-15: System resume (PM_POST_SUSPEND or PM_RESTORE_PREPARE) triggers crng_reseed.
- [ ] AC-16: VM-fork (add_vmfork_randomness) forces immediate crng_reseed.
- [ ] AC-17: fast_key_erasure: original ChaCha key overwritten by first 32 bytes of block before returning random_data.

## Architecture

```
enum CrngInit { Empty = 0, Early = 1, Ready = 2 }

struct BaseCrng {
  key: [u8; CHACHA_KEY_SIZE],          // 32
  generation: AtomicULong,             // ULONG_MAX = invalid
  lock: SpinLock,
}

struct Crng {                          // per-CPU `crngs`
  key: [u8; CHACHA_KEY_SIZE],
  generation: ULong,
  lock: LocalLock,
}

struct InputPool {
  hash: Blake2sCtx,                    // 256-bit output
  lock: SpinLock,
  init_bits: AtomicU32,                // ≤ POOL_BITS = 256
}

struct FastPool {                      // per-CPU `irq_randomness`
  pool: [ULong; 4],                    // SipHash state
  last: jiffies_t,
  count: u32,                          // bit 31 = MIX_INFLIGHT
  mix: TimerList,                      // → mix_interrupt_randomness
}

struct BatchEntropy<T> {               // per-CPU per-type cache
  entropy: [T; CHACHA_BLOCK_SIZE * 3 / (2*size_of::<T>())],
  lock: LocalLock,
  generation: ULong,
  position: u32,                       // UINT_MAX = invalid
}

struct TimerRandState {                // per-source
  last_time: jiffies_t,
  last_delta: i64,
  last_delta2: i64,
}
```

`Random::get_bytes(buf, len)`:
1. → `Random::get_bytes_internal(buf, len)`.
2. ChaChaState on stack.
3. `Random::make_state(&chacha_state, first_random, min(len, 32))`.
4. Copy first up-to-32 bytes.
5. While remaining: `chacha20_block(&chacha_state, block)`; advance counter; copy.
6. memzero_explicit(chacha_state).

`Random::make_state(chacha_state, random_data, len)`:
1. if !crng_ready():
   - extract_entropy(chacha_state, …); return.
2. local_lock_irqsave(&crngs.lock).
3. crng = this_cpu_ptr.
4. cur_gen = READ_ONCE(base_crng.generation).
5. if crng.generation != cur_gen:
   - spin_lock(&base_crng.lock); memcpy(crng.key, base_crng.key, 32); spin_unlock.
   - crng.generation = cur_gen.
6. `Random::fast_key_erasure(&crng.key, chacha_state, random_data, len)`.
7. local_unlock_irqrestore.

`Random::fast_key_erasure(key, chacha_state, random_data, len)`:
1. chacha_init_consts(chacha_state).
2. memcpy(&chacha_state.x[4], key, 32).
3. memset(&chacha_state.x[12], 0, 16).
4. chacha20_block(chacha_state, first_block).
5. memcpy(key, first_block, 32). // forward-secrecy: original key gone
6. memcpy(random_data, first_block + 32, len).
7. memzero_explicit(first_block).

`Random::crng_reseed(work)`:
1. queue_delayed_work(system_dfl_wq, &next_reseed, crng_reseed_interval()).
2. extract_entropy(key_local, 32).
3. spin_lock_irqsave(&base_crng.lock).
4. memcpy(base_crng.key, key_local, 32).
5. next_gen = base_crng.generation + 1; skip ULONG_MAX.
6. WRITE_ONCE(base_crng.generation, next_gen).
7. if CONFIG_VDSO_GETRANDOM: smp_store_release(&vdso_k_rng_data->generation, next_gen + 1).
8. if !crng_is_ready: crng_init = CRNG_READY.
9. spin_unlock_irqrestore.
10. memzero_explicit(key_local).

`Random::extract_entropy(buf, len)`:
1. Fill block.rdseed[] via arch_get_random_seed_longs / arch_get_random_longs / random_get_entropy.
2. spin_lock_irqsave(&input_pool.lock).
3. blake2s_final(&input_pool.hash, seed). // hash output as HKDF "extract" PRK
4. block.counter = 0; blake2s(seed, &block, next_key). // derive next chaining key
5. blake2s_init_key(&input_pool.hash, …, next_key). // re-seed pool for next extract
6. spin_unlock_irqrestore.
7. While len: ++block.counter; blake2s(seed, &block, buf, i); buf += i; len -= i.
8. memzero_explicit(seed); memzero_explicit(&block).

`Random::_credit_init_bits(bits)`:
1. add = min(bits, POOL_BITS); orig = READ_ONCE(input_pool.init_bits).
2. CAS-loop: new = min(POOL_BITS, orig+add).
3. if orig < POOL_READY_BITS && new >= POOL_READY_BITS:
   - crng_reseed(NULL). // sets CRNG_READY
   - queue_work(system_dfl_wq, &set_ready). // → static_branch_enable(crng_is_ready)
   - atomic_notifier_call_chain(&random_ready_notifier, 0, NULL).
   - vdso_k_rng_data->is_ready = true (if vDSO enabled).
   - wake_up_interruptible(&crng_init_wait).
   - kill_fasync(&fasync, SIGIO, POLL_IN).
   - pr_notice("crng init done").
4. else if orig < POOL_EARLY_BITS && new >= POOL_EARLY_BITS:
   - spin_lock(&base_crng.lock); if (crng_init == EMPTY) { extract_entropy(base_crng.key, 32); crng_init = EARLY; }; unlock.

`Random::add_interrupt_randomness(irq)`:
1. entropy = random_get_entropy(); fp = this_cpu(&irq_randomness); regs = get_irq_regs().
2. fast_mix(fp.pool, entropy, (regs ? IP : _RET_IP_) ^ swab(irq)).
3. new_count = ++fp.count.
4. if (new_count & MIX_INFLIGHT): return.
5. if (new_count < 1024 && fp.last + HZ > jiffies): return.
6. fp.count |= MIX_INFLIGHT.
7. if !timer_pending(&fp.mix): fp.mix.expires = jiffies; add_timer_on(&fp.mix, smp_processor_id()).

`Random::mix_interrupt_randomness(work)`:
1. local_irq_disable.
2. if fp != this_cpu_ptr: local_irq_enable; return. // hotplug raced
3. memcpy(pool[2], fp.pool, 16).
4. count = fp.count; fp.count = 0; fp.last = jiffies.
5. local_irq_enable.
6. mix_pool_bytes(pool, 16).
7. credit_init_bits(clamp((count & U16_MAX) / 64, 1, 128)).
8. memzero_explicit(pool).

`Random::sys_getrandom(ubuf, len, flags)`:
1. Validate flags ⊂ { GRND_NONBLOCK, GRND_RANDOM, GRND_INSECURE }: else -EINVAL.
2. Reject GRND_INSECURE | GRND_RANDOM: -EINVAL.
3. if !crng_ready && !GRND_INSECURE:
   - if GRND_NONBLOCK: return -EAGAIN.
   - ret = wait_for_random_bytes(); on signal: return -ERESTARTSYS.
4. import_ubuf(ITER_DEST, ubuf, len, &iter).
5. return get_random_bytes_user(&iter).

`Random::wait_for_random_bytes()`:
1. While !crng_ready:
   - try_to_generate_entropy.
   - wait_event_interruptible_timeout(crng_init_wait, crng_ready, HZ).
2. Return 0 (or -ERESTARTSYS).

`Random::try_to_generate_entropy()`:
1. Sample random_get_entropy NUM_TRIAL_SAMPLES times; count distinct.
2. samples_per_bit = DIV_ROUND_UP(NUM_TRIAL_SAMPLES, num_different + 1). Bail if > HZ/15.
3. Setup `entropy_timer_state` on stack; timer_setup_on_stack(&state.timer, entropy_timer).
4. Loop until crng_ready || signal_pending:
   - Pick next housekeeping CPU round-robin (avoid current).
   - timer.expires = jiffies; add_timer_on(&timer, cpu).
   - mix_pool_bytes(&state.entropy, …); schedule(); state.entropy = random_get_entropy().
5. timer_delete_sync(&timer); timer_destroy_on_stack(&timer).
6. Each `entropy_timer` callback: mix entropy; if atomic_inc_return % samples_per_bit == 0: credit_init_bits(1).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `base_crng_lock_held_during_key_write` | INVARIANT | per-crng_reseed: base_crng.lock held while overwriting key. |
| `per_cpu_crng_local_lock_held_during_make_state` | INVARIANT | per-crng_make_state: local_lock(&crngs.lock) held. |
| `crng_init_monotone` | INVARIANT | per-_credit_init_bits: crng_init only increases. |
| `key_zeroed_after_use` | INVARIANT | per-_get_random_bytes / crng_reseed: memzero_explicit on tmp buffers. |
| `fast_pool_cpu_local` | INVARIANT | per-mix_interrupt_randomness: bails when fp != this_cpu_ptr. |
| `input_pool_lock_held_during_blake2s` | INVARIANT | per-mix_pool_bytes / extract_entropy: input_pool.lock held during state mutation. |
| `getrandom_no_grnd_insecure_random_combo` | INVARIANT | per-sys_getrandom: rejects (GRND_INSECURE | GRND_RANDOM). |
| `getrandom_nonblock_returns_eagain` | INVARIANT | per-sys_getrandom: !crng_ready && GRND_NONBLOCK ⟹ -EAGAIN. |
| `batched_invalidation_on_reseed` | INVARIANT | per-get_random_uN: batch.generation != base_crng.generation forces refill. |
| `mix_inflight_timer_pending` | INVARIANT | per-add_interrupt_randomness: MIX_INFLIGHT ⟺ timer queued. |

### Layer 2: TLA+

`drivers/char/random.tla`:
- States: CRNG_EMPTY, CRNG_EARLY, CRNG_READY; pool.init_bits; per-CPU crng.generation; base_crng.generation.
- Actions: mix_pool_bytes, _credit_init_bits, crng_reseed, crng_make_state, add_interrupt_randomness, mix_interrupt_randomness, sys_getrandom, pm_notify_resume.
- Properties:
  - `safety_crng_init_monotone` — CRNG_EMPTY → CRNG_EARLY → CRNG_READY only.
  - `safety_no_random_pre_early` — _get_random_bytes never returns from base_crng before CRNG_EARLY (uses extract_entropy fallback).
  - `safety_getrandom_blocking_correct` — !crng_ready ∧ default_flags ⟹ syscall blocks until ready.
  - `safety_getrandom_nonblock` — GRND_NONBLOCK ⟹ -EAGAIN if !ready.
  - `safety_generation_bumped_on_reseed` — every reseed strictly increases base_crng.generation (skip ULONG_MAX).
  - `safety_per_cpu_rekey_on_generation_mismatch` — make_state refreshes per-CPU key whenever generation changes.
  - `safety_input_pool_lock_serialization` — concurrent mix_pool_bytes/extract_entropy fully serialized by input_pool.lock.
  - `liveness_eventually_crng_ready` — given sufficient entropy events, _credit_init_bits reaches POOL_READY_BITS.
  - `liveness_getrandom_returns_or_signaled` — wait_for_random_bytes returns 0 or -ERESTARTSYS.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `crng_reseed` post: base_crng.key replaced; generation += 1 (skip ULONG_MAX) | `Random::crng_reseed` |
| `crng_fast_key_erasure` post: original key overwritten; random_data = block[32..32+len] | `Random::fast_key_erasure` |
| `crng_make_state` post: per-CPU key matches base when caller's generation matched | `Random::make_state` |
| `_credit_init_bits` post: init_bits clamped to POOL_BITS; CRNG_EARLY / CRNG_READY transitions atomic | `Random::credit_init_bits` |
| `add_interrupt_randomness` post: fp.count incremented; MIX_INFLIGHT set when timer scheduled | `Random::add_interrupt_randomness` |
| `mix_interrupt_randomness` post: fp.count == 0; fp.last == jiffies; pool zeroed | `Random::mix_interrupt_randomness` |
| `sys_getrandom` post: returns bytes_copied or {-EINVAL, -EAGAIN, -ERESTARTSYS, -EFAULT} | `Random::sys_getrandom` |
| `random_read_iter` post: !crng_ready && NOWAIT ⟹ -EAGAIN | `Random::read_iter` |
| `urandom_read_iter` post: rate-limited warning if !crng_ready; returns bytes regardless | `Random::urandom_read_iter` |
| `random_ioctl(RNDADDENTROPY)` post: requires CAP_SYS_ADMIN; mixes + credits | `Random::ioctl` |
| `random_ioctl(RNDRESEEDCRNG)` post: -ENODATA if !crng_ready; else reseed | `Random::ioctl` |
| `random_pm_notify(PM_POST_SUSPEND)` post: crng_reseed invoked | `Random::pm_notify` |
| `add_vmfork_randomness` post: crng_reseed invoked when crng_ready | `Random::add_vmfork_randomness` |

### Layer 4: Verus/Creusot functional

`Per-boot model: random_init_early → mix arch+utsname+cmdline → trust_cpu credit → random_init → mix entropy+ktime → register_pm_notifier; entropy sources (interrupt fast pool, input timing, disk timing, hwrng, bootloader, vmfork) feed mix_pool_bytes; _credit_init_bits drives EMPTY → EARLY → READY; crng_make_state expands base_crng key via fast-key-erasure; getrandom(2)/dev/random block on crng_init_wait, /dev/urandom never blocks; suspend/resume reseed; CPU hotplug invalidates per-CPU caches.` Semantic equivalence: per-Documentation/admin-guide/kernel-parameters.txt (random.trust_cpu, random.trust_bootloader) + Documentation/admin-guide/sysctl/kernel.rst (kernel.random.*) + man getrandom(2) + man random(4) + man urandom(4).

## Hardening

(Inherits row-1 features from `drivers/char/00-overview.md` § Hardening.)

RNG reinforcement:

- **Per-fast-key-erasure ChaCha20** — defense against per-key-compromise extracting historical output (forward secrecy).
- **Per-CPU crng with generation counter** — defense against per-cross-CPU key leakage and per-reseed-stale-key.
- **Per-base_crng spinlock + per-CPU local_lock** — defense against per-concurrent key mutation.
- **Per-memzero_explicit on temporary keys and blocks** — defense against per-stack-disclosure and per-compiler dead-store-elimination.
- **Per-CRNG_INSECURE | CRNG_RANDOM combination rejected** — defense against per-flag-confusion (would request blocking AND insecure simultaneously).
- **Per-/dev/urandom rate-limited warning on uninitialized read** — defense against per-silent-insecure-boot.
- **Per-wait_for_random_bytes default for /dev/random and getrandom** — defense against per-pre-seed insecure random for crypto.
- **Per-CAP_SYS_ADMIN for RNDADDENTROPY / RNDADDTOENTCNT / RNDRESEEDCRNG / RNDZAPENTCNT / RNDCLEARPOOL** — defense against per-unprivileged-pool-poisoning.
- **Per-CPU hotplug invalidation (random_prepare_cpu zeros generation, random_online_cpu clears fast_pool count)** — defense against per-stale-key-after-offline.
- **Per-PM-notifier reseed on resume** — defense against per-hibernate/suspend key reuse.
- **Per-VM-fork reseed (add_vmfork_randomness)** — defense against per-VM-clone shared CRNG state.
- **Per-arch_get_random_seed_longs precedence over RDRAND** — defense against per-RDRAND-only weakness; RDSEED is preferred when available.
- **Per-trust_cpu / trust_bootloader cmdline gates** — defense against per-malicious-firmware credit injection.
- **Per-credit cap at POOL_BITS** — defense against per-credit-overflow / per-init_bits saturation attack.
- **Per-MIX_INFLIGHT bit on fast_pool.count** — defense against per-double-scheduling mix_interrupt_randomness.
- **Per-fasync SIGIO on CRNG_READY transition** — defense against per-polling-without-notification.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- include/linux/random.h public header (covered in 00-overview)
- crypto/chacha.c ChaCha20 primitive (covered in `crypto/` Tier-3 if expanded)
- crypto/blake2s_generic.c Blake2s primitive (covered in `crypto/` Tier-3 if expanded)
- drivers/char/hw_random/* hwrng framework (covered separately if expanded)
- arch/x86/lib/random.c RDRAND/RDSEED instructions (covered in `arch/x86/` Tier-3 if expanded)
- lib/vdso/getrandom.c vDSO getrandom fast path (covered separately if expanded)
- kernel/cpu.c CPU hotplug state machine (covered in `kernel/` Tier-3 if expanded)
- Implementation code
