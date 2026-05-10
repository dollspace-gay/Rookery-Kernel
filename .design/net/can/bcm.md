# Tier-3: net/can/bcm.c — CAN BCM (Broadcast Manager) protocol

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/can/00-overview.md
upstream-paths:
  - net/can/bcm.c (~1875 lines)
  - include/uapi/linux/can/bcm.h
-->

## Summary

CAN BCM (Broadcast Manager) provides per-CAN-ID **periodic + content-triggered** transmission and **filter-with-throttling** reception over SocketCAN. Per-socket op-code messages: `TX_SETUP` (per-(can_id) periodic TX with optional intervals + counter), `TX_DELETE`, `TX_READ`, `TX_SEND` (one-shot), `RX_SETUP` (per-(can_id, can_mask) RX-filter with optional content-change-only delivery + max-rate throttle), `RX_DELETE`, `RX_READ`. Per-(can_id, ifindex) op tracked. Replaces userspace polling for many automotive use cases. Critical for: automotive ECU monitoring, J1939 stacks, periodic-telemetry, signal-multiplexer apps.

This Tier-3 covers `bcm.c` (~1875 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bcm_sock` | per-socket state | `BcmSock` |
| `struct bcm_op` | per-op (per-(can_id, mode)) | `BcmOp` |
| `struct bcm_msg_head` | per-msg header (opcode + flags) | `BcmMsgHead` |
| `bcm_init()` | per-module init | `Bcm::init` |
| `bcm_create()` | socket(AF_CAN, SOCK_DGRAM, CAN_BCM) | `Bcm::create` |
| `bcm_release()` | close() | `Bcm::release` |
| `bcm_connect()` | bind to ifindex | `Bcm::connect` |
| `bcm_sendmsg()` | per-op-msg send | `Bcm::sendmsg` |
| `bcm_recvmsg()` | per-event recv | `Bcm::recvmsg` |
| `bcm_tx_setup()` | TX_SETUP handler | `Bcm::tx_setup` |
| `bcm_rx_setup()` | RX_SETUP handler | `Bcm::rx_setup` |
| `bcm_tx_send()` | TX_SEND one-shot | `Bcm::tx_send` |
| `bcm_can_tx()` | per-tick periodic-tx | `Bcm::can_tx` |
| `bcm_rx_handler()` | per-skb RX-filter callback | `Bcm::rx_handler` |
| `bcm_rx_changed()` | per-content-change check | `Bcm::rx_changed` |
| `bcm_rx_thr_handler()` | per-throttle timer | `Bcm::rx_thr_handler` |
| `bcm_tx_timeout_handler()` | per-tx interval timer | `Bcm::tx_timeout_handler` |
| `TX_SETUP` / `TX_DELETE` / `TX_READ` / `TX_SEND` / `TX_STATUS` / `TX_EXPIRED` | TX opcodes | UAPI |
| `RX_SETUP` / `RX_DELETE` / `RX_READ` / `RX_STATUS` / `RX_TIMEOUT` / `RX_CHANGED` | RX opcodes | UAPI |
| `SETTIMER` / `STARTTIMER` / `TX_COUNTEVT` / `TX_ANNOUNCE` / `TX_CP_CAN_ID` / `RX_FILTER_ID` / `RX_CHECK_DLC` / `RX_NO_AUTOTIMER` / `RX_ANNOUNCE_RESUME` / `TX_RESET_MULTI_IDX` | flags | UAPI |

## Compatibility contract

REQ-1: socket(AF_CAN, SOCK_DGRAM, CAN_BCM):
- bcm_create allocates BcmSock; bo.bound = false.

REQ-2: connect(sockaddr_can):
- bo.ifindex = addr.can_ifindex.
- ifindex == 0: per-any-iface.

REQ-3: TX_SETUP (per-tx periodic):
- struct bcm_msg_head + N × struct can_frame.
- bcm_op.can_id = msg.can_id.
- bcm_op.nframes = N.
- bcm_op.frames = N × can_frame copy.
- bcm_op.ival1 = initial interval (ktime).
- bcm_op.ival2 = subsequent interval.
- bcm_op.count = #counter for ival1, 0 = unlimited.
- Per-flag SETTIMER + STARTTIMER: arm hrtimer.
- Per-flag TX_COUNTEVT: send TX_EXPIRED when counter hits 0.

REQ-4: bcm_tx_timeout_handler (per-fire):
- For each frame in bcm_op.frames: bcm_can_tx(op).
- Decrement counter; if 0 ∧ TX_COUNTEVT: send TX_EXPIRED to userspace.
- Re-arm at ival1 (if count > 0) or ival2.

REQ-5: bcm_can_tx (per-tx):
- skb_clone of cached frame.
- skb.dev = bo.ifindex-dev.
- can_send(skb, ro.loopback).

REQ-6: TX_SEND (one-shot):
- bcm_tx_send: build skb from msg + 1 frame; can_send.

REQ-7: RX_SETUP (per-rx filter):
- bcm_op.can_id = msg.can_id; can_mask = msg.frames[0].can_mask (per-frame[0]).
- N frames = expected per-can_id frames (for content-change check).
- Per-flag RX_FILTER_ID: filter only by can_id (no per-byte content compare).
- Per-flag RX_CHECK_DLC: check DLC matches.
- Per-ival1: timeout (no-RX → RX_TIMEOUT).
- Per-ival2: throttle (max-rate of RX_CHANGED).
- can_rx_register on per-iface.

REQ-8: bcm_rx_handler (per-skb):
- bcm_op = data.
- /* Per-content-compare: */
- if RX_FILTER_ID: emit RX_CHANGED.
- else: per-can_frame compare bytes; if changed: emit RX_CHANGED.
- Per-throttle: throttle to ival2-rate.

REQ-9: TX_DELETE / RX_DELETE:
- Find bcm_op by (can_id, ifindex); release.

REQ-10: TX_READ / RX_READ:
- Reply with current state.

REQ-11: Per-userspace events emit:
- TX_EXPIRED: counter hit 0.
- RX_CHANGED: content changed (or first-rx).
- RX_TIMEOUT: no-rx within ival1.
- TX_STATUS / RX_STATUS: from TX_READ / RX_READ.

REQ-12: Per-AF_CAN integration:
- can_rx_register / can_rx_unregister for RX_SETUP.

## Acceptance Criteria

- [ ] AC-1: socket(AF_CAN, SOCK_DGRAM, CAN_BCM): succeeds.
- [ ] AC-2: connect to vcan0: bo.bound = true.
- [ ] AC-3: TX_SETUP with ival1=100ms, count=10, frames=[A]: 10 × A sent at 100ms intervals.
- [ ] AC-4: TX_COUNTEVT after 10th: TX_EXPIRED queued for recv.
- [ ] AC-5: TX_SEND: 1 frame sent immediately.
- [ ] AC-6: RX_SETUP can_id=A, frames=[A_template]: per-RX-A delivered when content changes.
- [ ] AC-7: RX_TIMEOUT with ival1=1s + no-RX: RX_TIMEOUT delivered.
- [ ] AC-8: RX_CHECK_DLC + DLC mismatch: per-skb dropped.
- [ ] AC-9: TX_DELETE: per-op released; hrtimer stopped.
- [ ] AC-10: bcm_release: all per-op cleaned up; can_rx_unregister called.

## Architecture

Per-socket state:

```
struct BcmSock {
  sk: Sock,
  bound: bool,
  ifindex: i32,
  notifier: NotifierBlock,
  tx_ops: ListHead<BcmOp>,
  rx_ops: ListHead<BcmOp>,
  dropped_usr_msgs: AtomicU32,
  bcm_proc_read: *ProcDirEntry,
}

struct BcmOp {
  list: ListLink,
  can_id: u32,
  flags: u32,                                    // SETTIMER / STARTTIMER / TX_COUNTEVT / RX_FILTER_ID / ...
  count: u32,
  nframes: u32,
  currframe: u32,
  frames: Vec<CanFrame>,
  last_frames: Vec<CanFrame>,                    // for content-compare
  sk: *Sock,
  rx_reg_dev: *NetDev,
  ival1: KtimeT,                                 // initial interval / RX-timeout
  ival2: KtimeT,                                 // subsequent / RX-throttle
  timer: HrTimer,                                // tx-timer / rx-timeout-timer
  thrtimer: HrTimer,                             // rx-throttle timer
  kt_ival1: KtimeT,
  kt_ival2: KtimeT,
  kt_lastmsg: KtimeT,
  fixed_can_dlc: u8,
}
```

`Bcm::create(net, sock, proto, kern) -> Result<()>`:
1. Validate sock.type == SOCK_DGRAM; proto == CAN_BCM.
2. Allocate BcmSock; init.
3. bo.notifier = bcm_notifier (per-iface event).
4. Register with net.can.bcm_proc.

`Bcm::connect(sock, addr, alen) -> Result<()>`:
1. addr_can = (sockaddr_can*)addr.
2. bo.ifindex = addr_can.can_ifindex.
3. bo.bound = true.

`Bcm::sendmsg(sock, msg, size) -> Result<usize>`:
1. msg_head = (BcmMsgHead*)msg.iov_base.
2. nframes = msg_head.nframes.
3. switch msg_head.opcode:
   - TX_SETUP: Bcm::tx_setup(msg_head, msg, sk, ifindex).
   - TX_DELETE: Bcm::tx_delete(msg_head, sk, ifindex).
   - TX_READ: Bcm::tx_read(msg_head, sk, ifindex).
   - TX_SEND: Bcm::tx_send(msg_head, msg, sk, ifindex).
   - RX_SETUP: Bcm::rx_setup(msg_head, msg, sk, ifindex).
   - RX_DELETE: Bcm::rx_delete(msg_head, sk, ifindex).
   - RX_READ: Bcm::rx_read(msg_head, sk, ifindex).
4. Return size.

`Bcm::tx_setup(msg_head, msg, sk, ifindex) -> Result<()>`:
1. op = lookup(bo.tx_ops, msg_head.can_id, ifindex) ∨ alloc.
2. /* Copy frames from msg */
3. for i in 0..msg_head.nframes:
   - op.frames[i] = msg.iov_buf[i + 1].
4. op.count = msg_head.count.
5. op.kt_ival1 = ktime_set(msg_head.ival1.tv_sec, msg_head.ival1.tv_usec*1000).
6. op.kt_ival2 = ktime_set(msg_head.ival2.tv_sec, msg_head.ival2.tv_usec*1000).
7. op.flags = msg_head.flags.
8. if op.flags & SETTIMER ∨ STARTTIMER:
   - hrtimer_start(&op.timer, op.kt_ival1, HRTIMER_MODE_REL).

`Bcm::tx_timeout_handler(timer) -> HrTimerRestart`:
1. op = container_of(timer, BcmOp, timer).
2. for i in 0..op.nframes: Bcm::can_tx(op, &op.frames[i]).
3. if op.count > 0: op.count--.
4. if op.count == 0 ∧ op.flags & TX_COUNTEVT:
   - bcm_send_to_user(op, TX_EXPIRED).
5. /* Re-arm */
6. if op.count > 0: hrtimer_forward_now(&op.timer, op.kt_ival1); return HRTIMER_RESTART.
7. else if op.kt_ival2 > 0: hrtimer_forward_now(&op.timer, op.kt_ival2); return HRTIMER_RESTART.
8. return HRTIMER_NORESTART.

`Bcm::can_tx(op, frame)`:
1. skb = sock_alloc_send_skb.
2. skb.dev = dev_get_by_index(net, op.ifindex).
3. memcpy(skb_put(skb, sizeof(can_frame)), frame, sizeof(can_frame)).
4. can_send(skb, 1 /* loopback */).

`Bcm::rx_setup(msg_head, msg, sk, ifindex) -> Result<()>`:
1. op = lookup ∨ alloc.
2. op.can_id = msg_head.can_id.
3. /* Per-content-compare frames */
4. for i in 0..nframes: op.frames[i] = msg.frames[i]; op.last_frames[i] = zeroed.
5. op.flags = msg_head.flags.
6. if !(op.flags & RX_NO_AUTOTIMER ∧ op.kt_ival1):
   - hrtimer_start(&op.timer, op.kt_ival1, REL).  // RX-timeout.
7. can_rx_register(net, ifindex, op.can_id, msg.frames[0].can_mask, Bcm::rx_handler, op, "bcm", sk).

`Bcm::rx_handler(skb, data)`:
1. op = data.
2. /* Per-content compare */
3. for i in 0..op.nframes:
   - if op.flags & RX_FILTER_ID ∨ memcmp(&skb_can_frame.data, &op.last_frames[i].data, sizeof) != 0:
     - op.last_frames[i] = skb_can_frame.
     - changed = true.
4. if changed:
   - if op.flags & RX_THROTTLE ∧ within ival2:
     - schedule_thr_timer.
   - else:
     - bcm_send_to_user(op, RX_CHANGED, &skb_can_frame).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tx_op_unique_per_can_id_iface` | INVARIANT | per-bo.tx_ops: at most one op per (can_id, ifindex). |
| `rx_op_unique_per_can_id_iface` | INVARIANT | per-bo.rx_ops: at most one op per (can_id, ifindex). |
| `count_decrements_to_zero` | INVARIANT | per-tx_timeout: count >= 0. |
| `frames_count_consistent` | INVARIANT | op.frames.len() == op.nframes. |
| `flags_in_set` | INVARIANT | op.flags is union of valid SETTIMER/STARTTIMER/TX_COUNTEVT/RX_FILTER_ID/etc. |

### Layer 2: TLA+

`net/can/bcm.tla`:
- Per-tx_setup periodic-tx + count-decrement + per-rx_setup content-compare + timeout.
- Properties:
  - `safety_periodic_tx_count_bounded` — per-TX_SETUP with count=N: TX fires exactly N times (then expired).
  - `safety_rx_changed_only_on_change` — per-rx-skb: emit RX_CHANGED iff content differs.
  - `liveness_eventual_rx_timeout` — per-RX_SETUP + no-rx for ival1: RX_TIMEOUT delivered.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Bcm::tx_setup` post: op in bo.tx_ops; timer armed at kt_ival1 | `Bcm::tx_setup` |
| `Bcm::tx_timeout_handler` post: per-fire frames sent; count-- if applicable | `Bcm::tx_timeout_handler` |
| `Bcm::rx_setup` post: op in bo.rx_ops; can_rx_register called | `Bcm::rx_setup` |
| `Bcm::rx_handler` post: per-changed-byte: RX_CHANGED queued | `Bcm::rx_handler` |

### Layer 4: Verus/Creusot functional

`Per-CAN_BCM userspace ABI: TX_SETUP / RX_SETUP per-(can_id) op + periodic-tx + content-change RX → fewer userspace polls` semantic equivalence: per-Documentation/networking/can.rst BCM.

## Hardening

(Inherits row-1 features from `net/can/raw.md` § Hardening.)

CAN_BCM-specific reinforcement:

- **Per-op count cap** — defense against per-op flood DoS.
- **Per-ival1/ival2 minimum bound** — defense against per-tick CPU storm.
- **Per-op_lifetime tied to socket** — defense against per-op leak after close.
- **Per-ifindex re-validation on netif-event** — defense against per-iface-down stale op.
- **Per-RX_FILTER_ID without content-bytes** — defense against per-skb-buffer-OOR.
- **Per-RX_CHECK_DLC** — defense against per-malformed DLC OOB.
- **Per-skb_clone bounded** — defense against per-flood OOM.
- **Per-rcu free of BcmOp** — defense against per-callback UAF.
- **Per-can_rx_register/unregister paired** — defense against per-stale callback.
- **Per-namespace per-net.can.bcm_proc scoped** — defense against cross-ns leak.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/can/raw (covered in `raw.md` Tier-3)
- net/can/af_can.c family core (covered separately)
- net/can/isotp.c (ISO-TP; covered separately)
- net/can/j1939/ (covered separately)
- Implementation code
