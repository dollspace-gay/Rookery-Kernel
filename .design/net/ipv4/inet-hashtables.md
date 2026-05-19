# Tier-3: net/ipv4/inet_hashtables.c — per-protocol bind + established socket hash tables (TCP/UDP/etc. lookup core)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/ipv4/00-overview.md
upstream-paths:
  - net/ipv4/inet_hashtables.c
  - net/ipv4/udp.c (uses similar hashinfo)
  - include/net/inet_hashtables.h
  - include/net/inet_sock.h
-->

## Summary

`net/ipv4/inet_hashtables.c` is the per-AF + per-protocol socket-lookup infrastructure — TCP, UDP, RAW each have their own hashinfo with bind-hash (per-port) + established-hash (per-flow-tuple) + listening-hash (per-port reuseport). Per-skb RX → __inet_lookup_skb returns matching sock via established-tuple-hash O(1) OR per-port listener-walk. Per-bind: per-port bucket records who's bound; SO_REUSEPORT supports multiple listeners on same port via reuseport-group. Critical: every TCP/UDP RX goes through it.

This Tier-3 covers `net/ipv4/inet_hashtables.c` (~1393 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct inet_hashinfo` | per-protocol hash | `net::ipv4::inet_hashtables::InetHashinfo` |
| `struct inet_bind_hashbucket` | per-port bucket | `InetBindHashbucket` |
| `struct inet_bind_bucket` | per-port info | `InetBindBucket` |
| `struct inet_listen_hashbucket` | per-port listener | `InetListenHashbucket` |
| `struct inet_ehash_bucket` | per-tuple bucket | `InetEhashBucket` |
| `inet_bind_hash(sk, tb, port)` | per-sock bind register | `Inet::bind_hash` |
| `inet_put_port(sk)` | per-sock unbind | `Inet::put_port` |
| `inet_csk_get_port(sk, snum)` | per-sock claim port | `Inet::csk_get_port` |
| `__inet_hash(sk, osk)` | per-sock add to estab | `Inet::__inet_hash` |
| `inet_unhash(sk)` | per-sock remove | `Inet::unhash` |
| `__inet_lookup_skb(hashinfo, skb, ...)` | per-skb sock lookup | `Inet::__lookup_skb` |
| `__inet_lookup_established(net, hashinfo, &saddr, sport, &daddr, dport, dif, sdif)` | per-flow estab lookup | `Inet::__lookup_established` |
| `inet_lookup_listener(net, hashinfo, &skb, doff, &saddr, sport, &daddr, dport, dif, sdif)` | per-port listener lookup | `Inet::lookup_listener` |
| `inet_ehashfn(net, &laddr, lport, &faddr, fport)` | per-tuple hash fn | `Inet::ehashfn` |
| `inet_bhash2_*` | per-AF + per-addr-family bind-hash-2 | `Inet::bhash2_*` |
| `inet_check_port_collision_*` | per-port collision detect | `Inet::check_port_collision` |

## Compatibility contract

REQ-1: Per-protocol `inet_hashinfo`:
- `lhash2` (KArc<InetListenHashbucket[N]>; per-port listener hash; N typically 32).
- `ehash` (KArc<InetEhashBucket[N]>; per-tuple established hash; N=N-cores * 16).
- `ehash_locks` (per-bucket spinlocks).
- `ehash_mask` (hash mask).
- `bhash` (KArc<InetBindHashbucket[N]>; per-port bind hash).
- `bhash_size` (count).
- `bhash2` (KArc<InetBindHashbucket[N]>; per-(port, addr) bind hash for IPv6/IPv4-VLAN).
- `bhash2_size` (count).
- `bind_bucket_cachep` (slab for bind_bucket).
- `bsockets` (per-protocol total bind-count).

REQ-2: Per-port bucket `inet_bind_hashbucket`:
- `lock` (per-bucket spinlock).
- `chain` (hlist of inet_bind_bucket).

REQ-3: Per-port info `inet_bind_bucket`:
- `node` (hlist_node in chain).
- `port` (16-bit).
- `fastreuse` (-1 / 0 / 1; speed-up SO_REUSEADDR).
- `fastreuseport` (-1 / 0 / 1; SO_REUSEPORT).
- `fastuid` (per-port UID-share check).
- `family` (AF_INET / AF_INET6).
- `flags` (TB_*: V4_OK, V6_OK, IPV6_USE_SAME, NEW).
- `owners` (chain of bound sks).
- `net` (back-ref to namespace).

REQ-4: Per-tuple bucket `inet_ehash_bucket`:
- `chain` (hlist_nulls).
- `lock` (separate per-bucket lock from chain).

REQ-5: Per-sock estab-hash hashing (`inet_ehashfn`):
- hash = jhash3(daddr, dport | (sport << 16), &ehash_secret) ^ jhash3(saddr, ...).

REQ-6: Per-sock bind flow:
1. inet_csk_get_port(sk, snum):
   - Validate per-namespace ip_local_port_range.
   - Lookup per-port bucket; if collision: check fastreuse / fastreuseport.
   - Allocate inet_bind_bucket if not exists.
   - inet_bind_hash(sk, tb, port).

REQ-7: Per-sock listener-hash flow:
1. After listen(2): __inet_hash adds sk to lhash2 bucket per-port.
2. Per-port reuseport: multiple listeners share via reuseport-group.

REQ-8: Per-sock established-hash flow:
1. After 3-way handshake: __inet_hash adds sk to ehash bucket per-(saddr,sport,daddr,dport).
2. Per-tuple unique; insertion atomic.

REQ-9: Per-skb lookup flow:
1. __inet_lookup_skb:
   - First: __inet_lookup_established (per-tuple O(1)).
   - If miss: inet_lookup_listener (per-port walk).
2. Match: full-tuple match for established; per-port match + reuseport hash for listener.

REQ-10: Per-protocol per-namespace isolation:
- Per-net inet_hashinfo can be per-net OR shared depending on protocol.
- Per-namespace bsockets count tracked.

REQ-11: SO_REUSEPORT:
- Multiple listeners on same port; per-flow hash distributes to one.
- Per-skb hash → reuseport-group selection.

REQ-12: Per-sock SO_BINDTODEVICE:
- Per-sock bound_dev_if; per-skb match must include device.

## Acceptance Criteria

- [ ] AC-1: bind() to port 80: per-port bucket allocated; sk added to owners.
- [ ] AC-2: listen(): sk added to lhash2; subsequent SYN-arrival lookup finds.
- [ ] AC-3: 3-way handshake completes: sk transitions from listener to established hash.
- [ ] AC-4: 100K-sock estab-hash: per-tuple O(1) lookup performance.
- [ ] AC-5: SO_REUSEPORT: 32 workers bind same port; per-flow hash distributes evenly.
- [ ] AC-6: SO_REUSEADDR: per-port collision detection; allow non-conflicting binds.
- [ ] AC-7: Per-namespace: 2 netns with bind to same port without collision.
- [ ] AC-8: SO_BINDTODEVICE: per-skb match includes device.
- [ ] AC-9: ss(8): per-protocol socket list correct.
- [ ] AC-10: linux test project inet-hash tests pass.

## Architecture

`InetHashinfo`:

```
struct InetHashinfo {
  lhash2: KArc<InetListenHashbucket[INET_LHTABLE_SIZE]>,
  ehash: KArc<InetEhashBucket[INET_EHASH_SIZE]>,
  ehash_locks: KArc<RwLock<()>; INET_EHASH_LOCK_SIZE>,
  ehash_mask: u32,
  bhash: KArc<InetBindHashbucket[INET_BHASH_SIZE]>,
  bhash_size: u32,
  bhash2: KArc<InetBindHashbucket[INET_BHASH2_SIZE]>,
  bhash2_size: u32,
  bind_bucket_cachep: KArc<KmemCache>,
  bsockets: AtomicI32,
}

struct InetEhashBucket {
  chain: HlistNullsHead,
}

struct InetBindBucket {
  node: HlistNode,
  port: u16,
  fastreuse: i8,
  fastreuseport: i8,
  fastuid: KuidT,
  family: u16,
  flags: u32,
  owners: HlistHead,
  net: KArc<Net>,
}
```

`Inet::__lookup_established(net, hashinfo, &saddr, sport, &daddr, dport, dif, sdif)`:
1. hash := inet_ehashfn(net, &daddr, dport, &saddr, sport).
2. bucket := &hashinfo.ehash[hash & hashinfo.ehash_mask].
3. lock := bucket-lock.
4. read_lock(lock).
5. For sk in bucket.chain:
   - If INET_TW_MATCH(sk, net, &saddr, sport, &daddr, dport, dif, sdif):
     - read_unlock; return sk.
6. read_unlock.
7. Return NULL.

`Inet::lookup_listener(net, hashinfo, skb, doff, &saddr, sport, &daddr, dport, dif, sdif)`:
1. hash := inet_lhashfn(net, dport).
2. bucket := &hashinfo.lhash2[hash].
3. read_lock.
4. For sk in bucket.head:
   - If matches per-namespace + per-port + per-bound-dev:
     - If reuseport: hash → group-member; return.
     - Else: return sk.
5. read_unlock.
6. Return NULL.

`Inet::bind_hash(sk, tb, port)`:
1. inet_sk(sk)->inet_num = port.
2. sk_add_bind_node(sk, &tb->owners).
3. inet_csk(sk)->icsk_bind_hash = tb.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bhash_idx_bounded` | OOB | per-port-hash idx < hashinfo.bhash_size. |
| `ehash_idx_bounded` | OOB | per-tuple-hash idx < hashinfo.ehash_mask + 1. |
| `bind_bucket_refcount_no_underflow` | INVARIANT | per-port-bucket owner ref-count ≥ 0. |
| `inet_lookup_rcu_protected` | UAF | per-lookup uses rcu_read_lock; defense against close-during-lookup. |
| `port_uniqueness_unless_reuse` | INVARIANT | per-port at most one owner unless SO_REUSEADDR/_PORT. |

### Layer 2: TLA+

`net/ipv4/inet_hash_lifecycle.tla`:
- Per-sock hash-state ∈ {Unhashed, Bound, Listening, Established, Closing}.
- Properties:
  - `safety_per_tuple_unique` — per-(saddr, sport, daddr, dport) at most one Established sock.
  - `safety_per_port_listener_uniqueness_modulo_reuseport` — per-port at most one listener unless reuseport.
  - `liveness_eventual_unhash` — every closing sock eventually Unhashed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Inet::__lookup_established` post: returned sk matches full-tuple OR NULL | `Inet::__lookup_established` |
| `Inet::lookup_listener` post: returned sk matches per-port + per-namespace | `Inet::lookup_listener` |
| `Inet::bind_hash` post: sk in tb->owners; inet_num set | `Inet::bind_hash` |
| `Inet::__inet_hash` post: sk in lhash2 (listener) or ehash (established) | `Inet::__inet_hash` |

### Layer 4: Verus/Creusot functional

`Per-skb: __inet_lookup_skb returns matching sock if exists in established or listener hash` semantic equivalence: per-flow the looked-up sock matches per-spec hashing.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

inet_hashtables-specific reinforcement:

- **Per-bucket lock granularity** — defense against single-lock contention on hot RX path.
- **RCU-protected hash-walk** — defense against close-during-lookup UAF.
- **Per-port bucket ref-counted** — defense against early-free.
- **fastreuse / fastreuseport flags** — defense against race-during-collision-check.
- **SO_BINDTODEVICE per-skb match** — defense against per-NIC bind violation.
- **Per-namespace isolation** — defense against cross-netns sock leak.
- **Reuseport hash per-skb deterministic** — defense against load-balancer-style nondeterminism.
- **inet_ehash_secret per-boot random** — defense against hash-collision DoS.
- **Per-port collision detection** — defense against TIME_WAIT-vs-NEW_SOCK collision.
- **Per-AF distinct lhash2 / bhash** — defense against AF_INET vs AF_INET6 cross-pollination.

## Grsecurity/PaX-style Reinforcement

Rationale: `inet_hashtables` houses `ehash` (established-connections, 4-tuple keyed), `lhash2` (LISTEN-state sockets), and `bhash`/`bhash2` (bind-buckets) — every incoming SYN/ACK/data segment is delivered via these tables, and every `connect(2)` / `bind(2)` mutates them. A hash-collision DoS, secret-leak (predictable ISN), or per-bucket refcount imbalance pivots into connection-hijacking or table-walk UAF.

Baseline (cross-ref `net/00-overview.md` § Hardening):
- **PAX_USERCOPY**: `/proc/net/tcp` enumeration emits opaque ifindex + state via `seq_printf` only; no slab-block dump.
- **PAX_KERNEXEC**: `inet_hashinfo` per-protocol structures (tcp_hashinfo, dccp_hashinfo) live in `__ro_after_init`; per-bucket spinlock array allocated at `inet_hashtables_init` and never resized post-boot.
- **PAX_RANDKSTACK**: every entry into `__inet_lookup_established`, `__inet_lookup_listener`, `inet_bind_bucket_create`, `__inet_hash` re-randomises kernel-stack offset.
- **PAX_REFCOUNT**: `inet_bind_bucket.refcnt`, `inet_listen_hashbucket.count`, `sk_refcnt` saturate.
- **PAX_MEMORY_SANITIZE**: freed `inet_bind_bucket` and `inet_timewait_sock` slab blocks zero-filled before slab-return.
- **PAX_UDEREF**: hashtable lookup paths are kernel-only; no `__user` deref.
- **PAX_RAP / kCFI**: indirect calls through per-`struct proto` hash/unhash methods (tcp_v4_hash, inet_unhash, etc.) are kCFI-tagged.
- **GRKERNSEC_HIDESYM**: per-bucket pointer, `inet_ehash_secret`, and per-socket `&sk` never rendered to `/proc/net/tcp`; only count + opaque hash fingerprint.
- **GRKERNSEC_DMESG**: `port reuse failure` and `bind: cannot assign requested address` warnings ratelimited; CAP_SYSLOG.

inet-hashtables-specific reinforcement:
- **Hash-bucket PAX_REFCOUNT** — `inet_bind_bucket.refcnt` and `inet_bind2_bucket.refcnt` saturate at INT_MAX; defends bind-storm refcount-wrap attack.
- **ISN/sequence-number GRKERNSEC_RANDNET** — `secure_tcp_seq` / `secure_tcp_ts_off` derive the initial sequence number via `siphash_3u32(sport,dport,daddr ^ ehash_secret)` rather than the legacy MD5; per-boot `inet_ehash_secret` rerolled with `get_random_bytes` (grsec policy extends to per-namespace secret tagging).
- **inet_ehash_secret per-boot random** (already in row-2) — grsec audits any kernel-internal symbol read attempting to disclose it.
- **Per-port collision detection** — `inet_csk_get_port` rejects bind to a port already in `bhash` with conflicting cred; defends `SO_REUSEADDR`/`SO_REUSEPORT` policy bypass across credentials.
- **Per-AF distinct lhash2 / bhash isolation** — `AF_INET` and `AF_INET6` use distinct hashbuckets; cross-AF lookup → BUG (defends v4-mapped-v6 confusion).
- **NULL-tail rcu_hlist invariant** — `__hlist_nulls_for_each_entry_rcu` walks under `rcu_read_lock`; mismatched tail-cookie → restart-or-BUG (no UAF window during resize).

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TCP / UDP per-protocol details (covered separately)
- Per-namespace setup (covered in `net/00-overview.md`)
- IPv6 inet6_hashtables (analog; covered separately)
- BPF socket lookup (covered in `kernel/bpf/bpf-core.md` Tier-3)
- Implementation code
