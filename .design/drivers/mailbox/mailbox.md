# Tier-3: drivers/mailbox/mailbox.c — Mailbox framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/mailbox/00-overview.md
upstream-paths:
  - drivers/mailbox/mailbox.c (~634 lines)
  - include/linux/mailbox_controller.h
  - include/linux/mailbox_client.h
-->

## Summary

The **mailbox framework** is a generic inter-processor / co-processor messaging abstraction. A *controller* (e.g. IPCC, ARM-MHU, OMAP-MBOX, MediaTek-CMDQ, Qualcomm-APCS) registers `struct mbox_controller` with `num_chans` `struct mbox_chan` slots; a *client* (PCC, SCMI, remoteproc, thermal) calls `mbox_request_channel()` to bind, then `mbox_send_message()` to queue a message. The controller's `struct mbox_chan_ops::send_data()` actually pushes hardware; TX-completion is signalled via one of three methods — **txdone-IRQ** (controller has interrupt: `mbox_chan_txdone(chan, r)`), **txdone-poll** (no IRQ: hrtimer at `txpoll_period` ms calls `last_tx_done(chan)`), or **txdone-by-ack** (controller blind: client calls `mbox_client_txdone(chan, r)` after protocol-ACK). RX is async: controller calls `mbox_chan_received_data(chan, mssg)` which invokes the client's `rx_callback`. Per-chan ring `MBOX_TX_QUEUE_LEN` slot ring buffer; blocking send waits on `chan->tx_complete` up to `cl->tx_tout`. Critical for: ARM PSCI/SCMI, ACPI PCC, AMP coprocessor IPC, modem-bridge.

This Tier-3 covers `drivers/mailbox/mailbox.c` (~634 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mbox_controller` | per-controller | `MboxController` |
| `struct mbox_chan` | per-channel | `MboxChan` |
| `struct mbox_chan_ops` | per-driver vtable | `MboxChanOps` |
| `struct mbox_client` | per-client | `MboxClient` |
| `mbox_controller_register()` | per-register | `Mailbox::controller_register` |
| `mbox_controller_unregister()` | per-deregister | `Mailbox::controller_unregister` |
| `devm_mbox_controller_register()` | per-devm | `Mailbox::devm_controller_register` |
| `mbox_request_channel()` | per-bind | `Mailbox::request_channel` |
| `mbox_request_channel_byname()` | per-name-bind | `Mailbox::request_channel_byname` |
| `mbox_bind_client()` | per-direct-bind | `Mailbox::bind_client` |
| `mbox_free_channel()` | per-release | `Mailbox::free_channel` |
| `mbox_send_message()` | per-tx-submit | `Mailbox::send_message` |
| `mbox_flush()` | per-tx-flush | `Mailbox::flush` |
| `mbox_chan_txdone()` | per-IRQ-tx-complete | `Mailbox::chan_txdone` |
| `mbox_client_txdone()` | per-ACK-tx-complete | `Mailbox::client_txdone` |
| `mbox_chan_received_data()` | per-rx-push | `Mailbox::chan_received_data` |
| `mbox_client_peek_data()` | per-rx-poke | `Mailbox::client_peek_data` |
| `mbox_chan_tx_slots_available()` | per-tx-queue-query | `Mailbox::tx_slots_available` |
| `add_to_rbuf()` (static) | per-enqueue | `MboxChan::add_to_rbuf` |
| `msg_submit()` (static) | per-dequeue+ship | `MboxChan::msg_submit` |
| `tx_tick()` (static) | per-tx-fsm-tick | `MboxChan::tx_tick` |
| `txdone_hrtimer()` (static) | per-poll-hrtimer | `MboxController::txdone_hrtimer` |
| `MBOX_TXDONE_BY_IRQ` / `_POLL` / `_ACK` | per-tx-mode | `TxDoneMethod` |
| `MBOX_TX_QUEUE_LEN` | per-ring-size | `MBOX_TX_QUEUE_LEN` |
| `MBOX_NO_MSG` | per-sentinel | `MBOX_NO_MSG` |
| `fw_mbox_index_xlate()` (static) | per-default-xlate | `Mailbox::fw_index_xlate` |
| `mbox_cons` / `con_mutex` | per-global-list | `MAILBOX_REGISTRY` |

## Compatibility contract

REQ-1: struct mbox_controller:
- dev: backing struct device.
- ops: per-driver vtable (`struct mbox_chan_ops *`).
- chans: array of `mbox_chan` (length `num_chans`).
- num_chans: per-controller channel count.
- txdone_irq: bool, controller-IRQ-completes-TX.
- txdone_poll: bool, controller-poll-completes-TX (mutually exclusive with txdone_irq; otherwise ACK).
- txpoll_period: per-millisecond poll period for `MBOX_TXDONE_BY_POLL`.
- of_xlate / fw_xlate: per-DT / per-fwnode index translator (default = `fw_mbox_index_xlate` if both NULL).
- poll_hrt / poll_hrt_lock: per-controller hrtimer + spinlock for poll mode.
- node: list head in global `mbox_cons` list.

REQ-2: struct mbox_chan:
- mbox: back-pointer to `mbox_controller`.
- cl: bound client (NULL = free).
- txdone_method: one of `MBOX_TXDONE_BY_IRQ` / `_POLL` / `_ACK` (or `_POLL | _ACK` after binding a knows_txdone client to a poll controller).
- msg_data: ring buffer of `void *` slots (size `MBOX_TX_QUEUE_LEN`).
- msg_count: per-pending count in ring.
- msg_free: per-next-free idx in ring.
- active_req: pointer of in-flight request, `MBOX_NO_MSG` when idle.
- tx_complete: completion for `cl->tx_block`.
- lock: per-channel spinlock (irqsave).
- con_priv: per-driver private cookie.

REQ-3: struct mbox_chan_ops:
- send_data(chan, data) -> int: per-driver TX submit. Atomic, may return -EBUSY.
- startup(chan) -> int: per-bind init (optional). May sleep.
- shutdown(chan): per-unbind teardown (optional). May sleep.
- last_tx_done(chan) -> bool: per-poll completion check. **Required** for `MBOX_TXDONE_BY_POLL`. Atomic.
- peek_data(chan) -> bool: per-RX-poke from client; True iff data was pulled and pushed via `mbox_chan_received_data`. Atomic.
- flush(chan, timeout_ms) -> int: per-busy-loop-flush for atomic-context drivers; must call `mbox_chan_txdone` on success.

REQ-4: struct mbox_client:
- dev: per-consumer device.
- tx_block: per-blocking-send.
- tx_tout: per-block-timeout (ms); 0 = infinite (3600000 ms cap).
- knows_txdone: per-protocol-completion (forces TXDONE_BY_ACK when controller was POLL).
- tx_prepare(cl, mssg): per-pre-submit shim (e.g. byte-swap, header).
- tx_done(cl, mssg, r): per-TX-completion callback. Called from atomic context.
- rx_callback(cl, mssg): per-RX delivery. Called from atomic context.

REQ-5: mbox_controller_register(mbox):
- /* Per-validation */
- if !mbox ∨ !mbox.dev ∨ !mbox.ops ∨ !mbox.chans ∨ !mbox.num_chans: return -EINVAL.
- /* Per-txdone mode selection */
- if mbox.txdone_irq: txdone = MBOX_TXDONE_BY_IRQ.
- else if mbox.txdone_poll: txdone = MBOX_TXDONE_BY_POLL.
- else: txdone = MBOX_TXDONE_BY_ACK.
- /* Poll mode requires last_tx_done */
- if txdone == MBOX_TXDONE_BY_POLL:
  - if !mbox.ops.last_tx_done: return -EINVAL.
  - hrtimer_setup(&mbox.poll_hrt, txdone_hrtimer, CLOCK_MONOTONIC, HRTIMER_MODE_REL).
  - spin_lock_init(&mbox.poll_hrt_lock).
- /* Per-channel init */
- for i in 0..mbox.num_chans:
  - chan = &mbox.chans[i].
  - chan.cl = NULL.
  - chan.mbox = mbox.
  - chan.active_req = MBOX_NO_MSG.
  - chan.txdone_method = txdone.
  - spin_lock_init(&chan.lock).
- /* Default xlate */
- if !mbox.fw_xlate ∧ !mbox.of_xlate: mbox.fw_xlate = fw_mbox_index_xlate.
- /* Per-list */
- guard(mutex)(&con_mutex).
- list_add_tail(&mbox.node, &mbox_cons).
- return 0.

REQ-6: mbox_controller_unregister(mbox):
- if !mbox: return.
- guard(mutex)(&con_mutex):
  - list_del(&mbox.node).
  - for i in 0..mbox.num_chans: mbox_free_channel(&mbox.chans[i]).
  - if mbox.txdone_poll: hrtimer_cancel(&mbox.poll_hrt).

REQ-7: devm_mbox_controller_register(dev, mbox):
- /* devres-managed; auto-unregister on driver detach */
- ptr = devres_alloc(__devm_mbox_controller_unregister, sizeof(*ptr), GFP_KERNEL).
- if !ptr: return -ENOMEM.
- err = mbox_controller_register(mbox).
- if err < 0: devres_free(ptr); return err.
- devres_add(dev, ptr); *ptr = mbox.
- return 0.

REQ-8: mbox_request_channel(cl, index):
- /* Lookup via DT/fwnode property "mboxes" + "#mbox-cells" */
- dev = cl.dev; if !dev: return ERR_PTR(-ENODEV).
- fwnode = dev_fwnode(dev); if !fwnode: return ERR_PTR(-ENODEV).
- fwnode_property_get_reference_args(fwnode, "mboxes", "#mbox-cells", 0, index, &fwspec).
- /* Per-walk mbox_cons under con_mutex */
- guard(mutex)(&con_mutex):
  - chan = ERR_PTR(-EPROBE_DEFER).
  - for mbox in mbox_cons:
    - if device_match_fwnode(mbox.dev, fwspec.fwnode):
      - if mbox.fw_xlate: chan = mbox.fw_xlate(mbox, &fwspec); if !IS_ERR(chan): break.
      - else if mbox.of_xlate: chan = mbox.of_xlate(mbox, &spec); if !IS_ERR(chan): break.
  - fwnode_handle_put(fwspec.fwnode).
  - if IS_ERR(chan): return chan.
  - ret = __mbox_bind_client(chan, cl).
  - if ret: chan = ERR_PTR(ret).
- return chan.

REQ-9: __mbox_bind_client(chan, cl):
- /* Exclusivity + module ref */
- if chan.cl ∨ !try_module_get(chan.mbox.dev.driver.owner): return -EBUSY.
- /* Initialise ring under chan.lock */
- scoped_guard(spinlock_irqsave, &chan.lock):
  - chan.msg_free = 0.
  - chan.msg_count = 0.
  - chan.active_req = MBOX_NO_MSG.
  - chan.cl = cl.
  - init_completion(&chan.tx_complete).
  - /* Per-knows_txdone override */
  - if chan.txdone_method == MBOX_TXDONE_BY_POLL ∧ cl.knows_txdone:
    - chan.txdone_method = MBOX_TXDONE_BY_ACK.
- /* Driver hook */
- if chan.mbox.ops.startup:
  - ret = chan.mbox.ops.startup(chan).
  - if ret: mbox_free_channel(chan); return ret.
- return 0.

REQ-10: mbox_free_channel(chan):
- if !chan ∨ !chan.cl: return.
- if chan.mbox.ops.shutdown: chan.mbox.ops.shutdown(chan).
- /* Queued TX requests abandoned silently (no callbacks) */
- scoped_guard(spinlock_irqsave, &chan.lock):
  - chan.cl = NULL.
  - chan.active_req = MBOX_NO_MSG.
  - if chan.txdone_method == MBOX_TXDONE_BY_ACK:
    - chan.txdone_method = MBOX_TXDONE_BY_POLL. /* restore controller-default */
- module_put(chan.mbox.dev.driver.owner).

REQ-11: mbox_send_message(chan, mssg):
- /* Per-input validation */
- if !chan ∨ !chan.cl ∨ mssg == MBOX_NO_MSG: return -EINVAL.
- /* Enqueue */
- t = add_to_rbuf(chan, mssg).
- if t < 0: dev_err "Try increasing MBOX_TX_QUEUE_LEN"; return t.
- /* Try-submit */
- msg_submit(chan).
- /* Blocking wait */
- if chan.cl.tx_block:
  - wait = (chan.cl.tx_tout == 0) ? msecs_to_jiffies(3600000) : msecs_to_jiffies(chan.cl.tx_tout).
  - ret = wait_for_completion_timeout(&chan.tx_complete, wait).
  - if ret == 0: t = -ETIME; tx_tick(chan, t).
- return t. /* Non-negative token in non-block mode */

REQ-12: add_to_rbuf(chan, mssg):
- guard(spinlock_irqsave)(&chan.lock).
- if chan.msg_count == MBOX_TX_QUEUE_LEN: return -ENOBUFS.
- idx = chan.msg_free.
- chan.msg_data[idx] = mssg.
- chan.msg_count++.
- chan.msg_free = (idx == MBOX_TX_QUEUE_LEN - 1) ? 0 : idx + 1.
- return idx.

REQ-13: msg_submit(chan):
- scoped_guard(spinlock_irqsave, &chan.lock):
  - /* Only one in-flight request per channel */
  - if !chan.msg_count ∨ chan.active_req != MBOX_NO_MSG: break.
  - count = chan.msg_count; idx = chan.msg_free.
  - /* Per-FIFO order from msg_free */
  - idx = (idx >= count) ? idx - count : idx + MBOX_TX_QUEUE_LEN - count.
  - data = chan.msg_data[idx].
  - if chan.cl.tx_prepare: chan.cl.tx_prepare(chan.cl, data).
  - err = chan.mbox.ops.send_data(chan, data).
  - if !err: chan.active_req = data; chan.msg_count--.
- /* Per-POLL kickoff */
- if !err ∧ (chan.txdone_method & MBOX_TXDONE_BY_POLL):
  - scoped_guard(spinlock_irqsave, &chan.mbox.poll_hrt_lock):
    - hrtimer_start(&chan.mbox.poll_hrt, 0, HRTIMER_MODE_REL).

REQ-14: tx_tick(chan, r):
- /* Advance FSM */
- scoped_guard(spinlock_irqsave, &chan.lock):
  - mssg = chan.active_req.
  - chan.active_req = MBOX_NO_MSG.
- msg_submit(chan). /* Next */
- if mssg == MBOX_NO_MSG: return.
- if chan.cl.tx_done: chan.cl.tx_done(chan.cl, mssg, r).
- if r != -ETIME ∧ chan.cl.tx_block: complete(&chan.tx_complete).

REQ-15: txdone_hrtimer(hrtimer):
- mbox = container_of(hrtimer, mbox_controller, poll_hrt).
- resched = false.
- for i in 0..mbox.num_chans:
  - chan = &mbox.chans[i].
  - if chan.active_req != MBOX_NO_MSG ∧ chan.cl:
    - txdone = chan.mbox.ops.last_tx_done(chan).
    - if txdone: tx_tick(chan, 0). /* OK */
    - else: resched = true.
- if resched:
  - scoped_guard(spinlock_irqsave, &mbox.poll_hrt_lock):
    - if !hrtimer_is_queued(hrtimer): hrtimer_forward_now(hrtimer, ms_to_ktime(mbox.txpoll_period)).
  - return HRTIMER_RESTART.
- return HRTIMER_NORESTART.

REQ-16: mbox_chan_txdone(chan, r) (IRQ path):
- if !(chan.txdone_method & MBOX_TXDONE_BY_IRQ):
  - dev_err "Controller can't run the TX ticker"; return.
- tx_tick(chan, r).

REQ-17: mbox_client_txdone(chan, r) (ACK path):
- if !(chan.txdone_method & MBOX_TXDONE_BY_ACK):
  - dev_err "Client can't run the TX ticker"; return.
- tx_tick(chan, r).

REQ-18: mbox_chan_received_data(chan, mssg):
- /* No buffering — direct dispatch; atomic context */
- if chan.cl.rx_callback: chan.cl.rx_callback(chan.cl, mssg).

REQ-19: mbox_client_peek_data(chan) -> bool:
- if chan.mbox.ops.peek_data: return chan.mbox.ops.peek_data(chan).
- return false.

REQ-20: mbox_flush(chan, timeout_ms):
- if !chan.mbox.ops.flush: return -ENOTSUPP.
- ret = chan.mbox.ops.flush(chan, timeout_ms).
- if ret < 0: tx_tick(chan, ret).
- return ret.

REQ-21: fw_mbox_index_xlate(mbox, sp) (default):
- if sp.nargs < 1 ∨ sp.args[0] >= mbox.num_chans: return ERR_PTR(-EINVAL).
- return &mbox.chans[sp.args[0]].

REQ-22: Per-MBOX_TX_QUEUE_LEN ring full ⟹ -ENOBUFS (caller must throttle or use `mbox_chan_tx_slots_available`).

## Acceptance Criteria

- [ ] AC-1: mbox_controller_register with NULL ops / num_chans=0 / no dev: -EINVAL.
- [ ] AC-2: mbox_controller_register with txdone_poll && !last_tx_done: -EINVAL.
- [ ] AC-3: mbox_controller_register with txdone_irq=false && txdone_poll=false: txdone_method = MBOX_TXDONE_BY_ACK on all chans.
- [ ] AC-4: mbox_request_channel honours "mboxes" / "#mbox-cells" and dispatches to fw_xlate / of_xlate; bind is exclusive (second request -> -EBUSY).
- [ ] AC-5: mbox_request_channel before controller register: -EPROBE_DEFER.
- [ ] AC-6: mbox_send_message with mssg == MBOX_NO_MSG: -EINVAL.
- [ ] AC-7: mbox_send_message fills ring; MBOX_TX_QUEUE_LEN+1 returns -ENOBUFS.
- [ ] AC-8: mbox_send_message with cl.tx_block && tx_tout: returns 0 on completion, -ETIME on timeout (tx_tick(-ETIME) invoked).
- [ ] AC-9: mbox_chan_txdone in BY_IRQ mode advances FSM and calls cl.tx_done; in BY_POLL/ACK mode emits dev_err.
- [ ] AC-10: mbox_client_txdone in BY_ACK mode advances FSM; otherwise dev_err.
- [ ] AC-11: txdone_hrtimer reschedules itself with txpoll_period until last_tx_done() returns true.
- [ ] AC-12: knows_txdone client bound to a POLL controller -> txdone_method becomes BY_ACK for that chan.
- [ ] AC-13: mbox_free_channel calls shutdown, clears cl, restores BY_POLL if BY_ACK was set via knows_txdone, drops module ref.
- [ ] AC-14: mbox_controller_unregister calls mbox_free_channel for every chan and cancels poll_hrt.
- [ ] AC-15: devm_mbox_controller_register releases controller on driver detach automatically.

## Architecture

```
struct MboxController {
  dev: *Device,
  ops: &'static MboxChanOps,
  chans: NonNull<[MboxChan]>,        // length num_chans
  num_chans: u32,
  txdone_irq: bool,
  txdone_poll: bool,
  txpoll_period: u32,                 // ms
  of_xlate: Option<OfXlateFn>,
  fw_xlate: Option<FwXlateFn>,
  poll_hrt: HrTimer,
  poll_hrt_lock: SpinLock,
  node: ListLink,
}

struct MboxChan {
  mbox: NonNull<MboxController>,
  cl: Option<NonNull<MboxClient>>,
  txdone_method: u8,                  // bitmask of TxDoneMethod
  msg_data: [*mut c_void; MBOX_TX_QUEUE_LEN],
  msg_count: u32,
  msg_free: u32,
  active_req: *mut c_void,            // MBOX_NO_MSG = idle sentinel
  tx_complete: Completion,
  lock: SpinLock,                     // irqsave
  con_priv: *mut c_void,
}

bitflags! TxDoneMethod {
  BY_IRQ  = 0x1,
  BY_POLL = 0x2,
  BY_ACK  = 0x4,
}
```

`Mailbox::controller_register(mbox) -> Result<()>`:
1. Validate: dev / ops / chans / num_chans non-zero.
2. Select txdone:
   - mbox.txdone_irq  -> BY_IRQ.
   - else mbox.txdone_poll -> BY_POLL (require ops.last_tx_done; init poll_hrt + poll_hrt_lock).
   - else -> BY_ACK.
3. For each chan in chans: zero cl, set mbox, active_req = MBOX_NO_MSG, txdone_method, spin_lock_init.
4. If both xlate hooks NULL: fw_xlate = fw_mbox_index_xlate.
5. con_mutex.lock(); list_add_tail(node, &mbox_cons); unlock.

`Mailbox::request_channel(cl, index) -> Result<&MboxChan>`:
1. cl.dev or -ENODEV; fwnode or -ENODEV.
2. fwnode_property_get_reference_args("mboxes", "#mbox-cells", 0, index, &fwspec).
3. con_mutex.lock():
   - chan = -EPROBE_DEFER.
   - Walk mbox_cons; on dev/fwnode match, dispatch fw_xlate or of_xlate.
   - fwnode_handle_put(fwspec.fwnode).
   - if Ok(chan): __bind_client(chan, cl).

`MboxChan::add_to_rbuf(mssg) -> Result<u32>` (spinlock_irqsave):
1. If msg_count == MBOX_TX_QUEUE_LEN: -ENOBUFS.
2. idx = msg_free; msg_data[idx] = mssg; msg_count += 1; msg_free = (idx+1) % MBOX_TX_QUEUE_LEN.
3. Ok(idx).

`MboxChan::msg_submit(&self)`:
1. spinlock_irqsave:
   - if msg_count == 0 ∨ active_req != MBOX_NO_MSG: return (out-of-band err preserved).
   - dequeue head (FIFO order using msg_free - msg_count modulo MBOX_TX_QUEUE_LEN).
   - if cl.tx_prepare: cl.tx_prepare(cl, data).
   - err = ops.send_data(chan, data).
   - if Ok: active_req = data; msg_count -= 1.
2. If Ok ∧ (txdone_method & BY_POLL): poll_hrt_lock; hrtimer_start(0).

`MboxChan::tx_tick(&self, r: i32)`:
1. spinlock_irqsave: mssg = active_req; active_req = MBOX_NO_MSG.
2. msg_submit(self) /* drain next */.
3. if mssg == MBOX_NO_MSG: return.
4. if cl.tx_done: cl.tx_done(cl, mssg, r).
5. if r != -ETIME ∧ cl.tx_block: tx_complete.complete().

`Mailbox::send_message(chan, mssg) -> Result<i32>`:
1. Validate chan / cl / mssg != MBOX_NO_MSG.
2. add_to_rbuf(chan, mssg) -> token.
3. msg_submit(chan).
4. if cl.tx_block:
   - wait = if cl.tx_tout == 0 { 3_600_000 ms } else { cl.tx_tout ms }.
   - if wait_for_completion_timeout(&chan.tx_complete, wait) == 0:
     - token = -ETIME; tx_tick(chan, -ETIME).
5. Ok(token).

`Mailbox::txdone_hrtimer(hrt) -> HrtimerRestart`:
1. mbox = container_of(hrt, MboxController, poll_hrt).
2. resched = false.
3. for chan in &mbox.chans:
   - if chan.active_req != MBOX_NO_MSG ∧ chan.cl:
     - if ops.last_tx_done(chan): tx_tick(chan, 0).
     - else: resched = true.
4. if resched: poll_hrt_lock; if !hrtimer_is_queued(hrt): hrtimer_forward_now(hrt, ms_to_ktime(mbox.txpoll_period)); Restart.
5. else: NoRestart.

`Mailbox::chan_received_data(chan, mssg)`:
- Atomic context. cl.rx_callback(cl, mssg) if present. Controller ACKs hardware only after return.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `controller_register_validates_inputs` | INVARIANT | per-controller_register: NULL dev/ops/chans/num_chans=0 rejected. |
| `poll_requires_last_tx_done` | INVARIANT | per-controller_register: BY_POLL ⟹ ops.last_tx_done non-null. |
| `chan_bind_exclusive` | INVARIANT | per-__mbox_bind_client: chan.cl != NULL ⟹ -EBUSY. |
| `tx_ring_bounded` | INVARIANT | per-add_to_rbuf: msg_count ≤ MBOX_TX_QUEUE_LEN. |
| `active_req_single_in_flight` | INVARIANT | per-msg_submit: send_data invoked iff active_req == MBOX_NO_MSG. |
| `txdone_method_dispatch_correct` | INVARIANT | per-mbox_chan_txdone: invoked only when BY_IRQ set; per-mbox_client_txdone: only when BY_ACK set. |
| `txdone_hrtimer_only_self_resched` | INVARIANT | per-txdone_hrtimer: rearmed only when active_req still pending. |
| `module_ref_balanced` | INVARIANT | per-bind/free: try_module_get / module_put per channel. |

### Layer 2: TLA+

`drivers/mailbox/mailbox.tla`:
- Per-bind + per-enqueue + per-submit + per-completion + per-unbind.
- Properties:
  - `safety_at_most_one_in_flight` — per-chan: |{idx : msg_data[idx] == active_req}| ≤ 1.
  - `safety_ring_capacity` — per-chan: msg_count ∈ 0..MBOX_TX_QUEUE_LEN.
  - `safety_no_callback_after_free` — per-mbox_free_channel: no tx_done / rx_callback fires after cl = NULL.
  - `safety_txdone_path_exclusive` — per-mode: BY_IRQ ⟹ only mbox_chan_txdone advances; BY_POLL ⟹ only hrtimer; BY_ACK ⟹ only mbox_client_txdone (or knows_txdone-override).
  - `liveness_tx_block_eventually_resolves` — per-cl.tx_block: completes within tx_tout or returns -ETIME.
  - `liveness_poll_eventually_drains` — per-BY_POLL: hrtimer eventually observes last_tx_done = true OR unregister cancels timer.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mailbox::controller_register` post: chans[i].mbox == mbox ∧ chans[i].cl == None ∧ active_req == MBOX_NO_MSG | `Mailbox::controller_register` |
| `MboxChan::add_to_rbuf` post: Ok(idx) ⟹ idx < MBOX_TX_QUEUE_LEN ∧ msg_data[idx] == mssg | `MboxChan::add_to_rbuf` |
| `MboxChan::msg_submit` post: ¬err ⟹ active_req != MBOX_NO_MSG | `MboxChan::msg_submit` |
| `MboxChan::tx_tick` post: active_req == MBOX_NO_MSG ∧ tx_done called once per dequeued mssg | `MboxChan::tx_tick` |
| `Mailbox::send_message` post: !tx_block ⟹ returns non-negative token; tx_block ⟹ returns 0 on success, -ETIME on timeout, ≤0 on send_data error | `Mailbox::send_message` |
| `Mailbox::request_channel` post: Ok(chan) ⟹ chan.cl == Some(cl) ∧ try_module_get succeeded | `Mailbox::request_channel` |
| `Mailbox::free_channel` post: chan.cl == None ∧ active_req == MBOX_NO_MSG ∧ module_put dispatched | `Mailbox::free_channel` |

### Layer 4: Verus/Creusot functional

`Per-bind → per-enqueue → per-submit (ops.send_data) → per-completion (IRQ ∨ POLL.last_tx_done ∨ ACK) → tx_tick → cl.tx_done → next msg_submit → release` semantic equivalence: per-Documentation/driver-api/mailbox.rst + per-controller-driver shims (PCC, SCMI, OMAP, MHU).

## Hardening

(Inherits row-1 features from `drivers/mailbox/00-overview.md` § Hardening.)

Mailbox-framework reinforcement:

- **Per-chan.lock spinlock-irqsave around ring** — defense against per-IRQ-vs-task ring corruption.
- **Per-MBOX_TX_QUEUE_LEN ring bounded** — defense against per-unbounded-enqueue OOM by misbehaving client.
- **Per-MBOX_NO_MSG sentinel != any valid pointer** — defense against per-NULL-as-message ambiguity.
- **Per-active_req single in-flight invariant** — defense against per-double-send_data race.
- **Per-try_module_get on bind / module_put on free** — defense against per-controller-module-unload while bound.
- **Per-txdone_method exact-mask check in chan_txdone / client_txdone** — defense against per-wrong-path TX-ack.
- **Per-BY_POLL requires last_tx_done callback** — defense against per-stuck-poll due to missing driver hook.
- **Per-hrtimer cancelled in controller_unregister** — defense against per-UAF on freed mbox in timer.
- **Per-mbox_request_channel returns -EPROBE_DEFER, not panic, on missing controller** — defense against per-init-order race.
- **Per-cl.tx_block timeout capped at 3,600,000 ms** — defense against per-forever wait when tx_tout=0 + driver wedge.
- **Per-shutdown invoked from free_channel under con_mutex protection** — defense against per-shutdown-after-unregister race.
- **Per-knows_txdone client coerces method without re-validating last_tx_done absence** — assumes ACK path strictly; defense against silent-poll-failure with knows_txdone.
- **Per-flush() fallback path tx_tick(ret) on failure** — defense against per-flush-failure stuck active_req.

## Grsecurity/PaX-style Reinforcement

Rationale: mailbox channels carry control-plane messages between the host kernel and security-relevant co-processors (PSCI, SCMI, TF-A, modem, NPU). A corrupted `mbox_chan_ops` vtable, mis-dispatched txdone callback, or message pointer aliasing here yields direct control of remote-firmware ABI requests — including reset, power-domain, and SCMI clock/voltage commands. The reinforcement below restates baseline PaX/grsec coverage applied to `drivers/mailbox/mailbox.c` plus mailbox-framework-specific reinforcement.

Baseline (cross-ref `drivers/mailbox/00-overview.md` § Hardening):
- **PAX_USERCOPY**: mailbox framework has no direct user surface in-kernel; debugfs/sysfs counters (if exposed by controller drivers) go through whitelisted slab caches.
- **PAX_KERNEXEC**: per-`mbox_chan_ops` and per-`mbox_controller.of_xlate`/`fw_xlate` vtables installed at register time and placed `__ro_after_init` per controller; never repointed.
- **PAX_RANDKSTACK**: client/controller entry points (`mbox_send_message`, `mbox_chan_txdone`, `mbox_chan_received_data`) re-randomise kernel stack on entry — especially important since `rx_callback` is invoked from atomic / IRQ context.
- **PAX_REFCOUNT**: per-`mbox_chan` `try_module_get` / `module_put` count saturating; per-controller `node` list usage refcount-checked.
- **PAX_MEMORY_SANITIZE**: `msg_data[idx]` slot zeroed on dequeue (after `send_data` succeeds) so a freed message pointer cannot be re-dispatched; per-`MboxChan` `con_priv` zeroed on `mbox_free_channel`.
- **PAX_UDEREF**: no direct user pointers in mailbox.c.
- **PAX_RAP / kCFI**: `send_data`, `startup`, `shutdown`, `last_tx_done`, `peek_data`, `flush`, `tx_prepare`, `tx_done`, `rx_callback`, `of_xlate`, `fw_xlate` all dispatched via kCFI-tagged indirect calls; mismatched signature traps to `BUG()`.
- **GRKERNSEC_HIDESYM**: `dev_err`/`dev_warn` from mailbox.c emits only device name; `&chan` / `&mbox` rendered with `%pK`.
- **GRKERNSEC_DMESG**: queue-overflow / "Try increasing MBOX_TX_QUEUE_LEN" warnings ratelimited; `dmesg` access requires CAP_SYSLOG.

mailbox-framework-specific reinforcement:
- **`mbox_chan` PAX_REFCOUNT on cl-binding** — `chan.cl` set/clear uses saturating counter under `chan.lock`; defends double-bind races during driver bringup.
- **RAP on `chan_ops`** — `mbox_chan_ops` signatures kCFI-tagged with arg/ret type fingerprint; controller drivers cannot publish ABI-mismatched ops (e.g. `send_data` returning pointer instead of int).
- **`MBOX_NO_MSG` sentinel disallows NULL message and disallows the kernel-text address range** — defends against a controller driver passing `mssg = (void*)mbox_chan_txdone` to confuse the FSM.
- **`msg_data[]` ring bounded + GFP_ATOMIC-only path** — never reallocs; defends against atomic-context allocation failure cascade.
- **`mbox_request_channel` enforces fwnode same-namespace** — fwspec.fwnode must share root with consumer's `cl.dev` fwnode; defends against cross-namespace channel hijack on multi-tenant SoCs.
- **`txdone_method` mask-tested with strict `&` (not `==`)** — already in row-2 hardening; extended with grsec audit entry when a controller calls into the wrong path.
- **Poll-mode hrtimer interval clamped** — `txpoll_period` floor 1 ms, ceiling 1000 ms; defends against SoC firmware quirks driving 0-period busy-loop.
- **`tx_tout` 3,600,000 ms hard cap** — already in row-2; extended with audit log when client passes `tx_tout > 60_000` to flag misbehaving consumers.
- **`mbox_controller_unregister` con_mutex barrier** — guarantees in-flight `mbox_request_channel` walkers finish before controller free; defends against UAF on iterating list while node freed.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Individual controller drivers (omap, mhu, pcc, sti, ti-msgmgr, qcom-apcs-ipc, mtk-cmdq, …) — each gets its own Tier-3 if/when expanded.
- PCC sub-spec (ACPI Platform Communication Channel) — covered by ACPI subsystem docs.
- SCMI / SCPI protocol layer atop mailbox — covered separately.
- DT bindings format (#mbox-cells semantics) — Documentation/devicetree/bindings/mailbox/.
- Implementation code.
