---
title: "Tier-3: kernel/user_namespace.c — User namespace and uid/gid/projid mapping"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

A **user namespace** is the per-task isolation unit for credentials, capabilities, and identifier mapping. Per-`struct user_namespace`: uid_map / gid_map / projid_map (per-extent translation tables), parent (per-tree edge), level (per-depth, ≤ 33 nested), owner / group (per-creator eff-uid/gid pinned in parent), ucount_max[UCOUNT_COUNTS] (per-resource caps), ucounts (per-self slot in parent), flags (USERNS_SETGROUPS_ALLOWED + USERNS_SETGROUPS_ALLOWED), keyring_name_list (per-keyring), binfmt_misc (per-ns). Per-`kuid_t` / `kgid_t` / `kprojid_t`: kernel-internal IDs that **only have meaning in init_user_ns**; per-syscall translation through the **current** user-ns's map ⟹ uid_t / gid_t the caller can name. Per-`make_kuid(ns, uid)`: `KUIDT_INIT(map_id_down(&ns.uid_map, uid))` — translate userspace id ⟹ kernel-global id. Per-`from_kuid(targ, kuid)`: `map_id_up(&targ.uid_map, __kuid_val(kuid))` — translate kernel id ⟹ userspace id in target ns. Per-`uid_eq(a, b)` / `gid_eq(a, b)`: strict equality of `__kuid_val` (not the userspace number). Per-`create_user_ns(new)`: per-`unshare(CLONE_NEWUSER)` / per-`clone(CLONE_NEWUSER)` constructor; new ns starts with **CAP_FULL_SET inside, no caps outside**, an empty uid_map/gid_map, and `USERNS_SETGROUPS_ALLOWED` inherited from parent. Per-`/proc/<pid>/uid_map` and `/proc/<pid>/gid_map`: one-shot write of extent triples `inside_id outside_id count`. Per-`/proc/<pid>/setgroups`: `"allow"` or `"deny"` writes; `deny` is irreversible and required for unprivileged gid_map. Critical for: rootless containers, user-ns-based sandboxing, per-container UID/GID translation, capability scoping.

This Tier-3 covers `kernel/user_namespace.c` (~1415 lines).

### Acceptance Criteria

- [ ] AC-1: `unshare(CLONE_NEWUSER)` succeeds for any unprivileged caller (unless rlimit exceeded or chrooted).
- [ ] AC-2: Nesting past 33 levels (`parent.level > 32`) returns `-ENOSPC`.
- [ ] AC-3: Inside a new user-ns with no map written, `getuid() == overflowuid` (65534 default).
- [ ] AC-4: After writing `"0 1000 1"` to uid_map (as creator with euid 1000), inside `getuid() == 0`; outside, `stat` shows uid 1000.
- [ ] AC-5: Writing a non-identity uid_map without CAP_SETUID in parent (and not the trivial owner identity case) ⟹ `-EPERM`.
- [ ] AC-6: Writing `"deny"` to setgroups before gid_map ⟹ permanent; subsequent `"allow"` write returns `-EPERM`.
- [ ] AC-7: Writing gid_map after `"deny"` is permitted without CAP_SETGID in parent (unprivileged single-identity rule).
- [ ] AC-8: `setgroups(2)` returns `-EPERM` when `userns_may_setgroups(ns) == false`.
- [ ] AC-9: uid_map / gid_map / projid_map are write-once: second write returns `-EPERM`.
- [ ] AC-10: A write of an extent that overlaps a previously-written extent in the same write ⟹ `-EINVAL`.
- [ ] AC-11: A write with > 340 extents (`UID_GID_MAP_MAX_EXTENTS`) ⟹ `-EINVAL`.
- [ ] AC-12: A write where any extent's outside range is not mapped in the parent ⟹ `-EPERM`.
- [ ] AC-13: `current_chrooted()` true ⟹ `unshare(CLONE_NEWUSER)` ⟹ `-EPERM`.
- [ ] AC-14: After unshare, `cap_effective == CAP_FULL_SET` inside the ns; outside, `ns_capable(parent_ns, CAP_*)` is false unless the kuid match grants it.
- [ ] AC-15: `/proc/<pid>/ns/user` symlink stat-inum equals `ns.ns.inum`; `ioctl(NS_GET_USERNS)` from a non-user ns returns that ns's owning user-ns.
- [ ] AC-16: Writing to uid_map from a process whose user-ns is not the target ns and not its parent ⟹ `-EPERM`.

### Architecture

```
pub struct UidGidExtent {
  pub first: u32,        /* inside-ns id range start */
  pub lower_first: u32,  /* parent-ns id range start */
  pub count: u32,
}

pub struct UidGidMap {
  pub nr_extents: u32,
  pub extent: [UidGidExtent; UID_GID_MAP_MAX_BASE_EXTENTS], /* 5 */
  pub forward: *mut UidGidExtent, /* sorted by first (when > 5) */
  pub reverse: *mut UidGidExtent, /* sorted by lower_first */
}

pub struct UserNamespace {
  pub ns: NsCommon,
  pub uid_map: UidGidMap,
  pub gid_map: UidGidMap,
  pub projid_map: UidGidMap,
  pub parent: *const UserNamespace,
  pub level: i32,        /* ≤ 33 */
  pub owner: KUid,       /* creator's euid */
  pub group: KGid,       /* creator's egid */
  pub flags: AtomicU32,  /* USERNS_SETGROUPS_ALLOWED */
  pub ucount_max: [i32; UCOUNT_COUNTS],
  pub rlimit_max: [u64; UCOUNT_RLIMIT_COUNTS],
  pub ucounts: *const Ucounts,
  pub parent_could_setfcap: bool,
  pub work: WorkStruct,
  #[cfg(keys)] pub keyring_name_list: ListHead,
  #[cfg(keys)] pub keyring_sem: RwSemaphore,
  #[cfg(binfmt_misc)] pub binfmt_misc: *mut BinfmtMisc,
}

pub const USERNS_SETGROUPS_ALLOWED: u32 = 1 << 0;
pub const UID_GID_MAP_MAX_BASE_EXTENTS: usize = 5;
pub const UID_GID_MAP_MAX_EXTENTS: usize = 340;
pub const MAX_USERNS_LEVEL: i32 = 33;

pub static INIT_USER_NS: UserNamespace = /* level=0, parent=NULL, flags=USERNS_SETGROUPS_ALLOWED, ... */;
```

`UserNs::create(new: *Cred) -> Result<(), Errno>`:
1. let parent_ns = new.user_ns; let owner = new.euid; let group = new.egid.
2. if parent_ns.level > 32 { return Err(ENOSPC) }.
3. let ucounts = UserNs::inc(parent_ns, owner).ok_or(ENOSPC)?
4. if current_chrooted() { return Err(EPERM) }.
5. if !kuid_has_mapping(parent_ns, owner) || !kgid_has_mapping(parent_ns, group) { return Err(EPERM) }.
6. security_create_user_ns(new)?
7. let ns = kmem_cache_zalloc(user_ns_cachep, GFP_KERNEL).ok_or(ENOMEM)?
8. ns.parent_could_setfcap = cap_raised(new.cap_effective, CAP_SETFCAP).
9. ns_common_init(&mut ns.ns)?
10. ns.parent = parent_ns; ns.level = parent_ns.level + 1; ns.owner = owner; ns.group = group.
11. INIT_WORK(&mut ns.work, UserNs::free).
12. for i in 0..UCOUNT_COUNTS { ns.ucount_max[i] = i32::MAX }.
13. set_userns_rlimit_max(ns, UCOUNT_RLIMIT_NPROC, UserNs::enforced_nproc_rlimit()).
14. set_userns_rlimit_max(ns, UCOUNT_RLIMIT_MSGQUEUE, rlimit(RLIMIT_MSGQUEUE)).
15. set_userns_rlimit_max(ns, UCOUNT_RLIMIT_SIGPENDING, rlimit(RLIMIT_SIGPENDING)).
16. set_userns_rlimit_max(ns, UCOUNT_RLIMIT_MEMLOCK, rlimit(RLIMIT_MEMLOCK)).
17. ns.ucounts = ucounts.
18. mutex_lock(&USERNS_STATE_MUTEX); ns.flags = parent_ns.flags; mutex_unlock.
19. UserNs::setup_sysctls(ns)?
20. UserNs::set_cred_user_ns(new, ns).
21. ns_tree_add(ns).
22. Ok(()).

`UserNs::set_cred_user_ns(cred, ns)`:
1. cred.securebits = SECUREBITS_DEFAULT.
2. cred.cap_inheritable = CAP_EMPTY_SET.
3. cred.cap_permitted = CAP_FULL_SET.
4. cred.cap_effective = CAP_FULL_SET.
5. cred.cap_ambient = CAP_EMPTY_SET.
6. cred.cap_bset = CAP_FULL_SET.
7. #[cfg(keys)] { key_put(cred.request_key_auth); cred.request_key_auth = NULL }.
8. cred.user_ns = ns.

`UserNs::map_id_down(map, id) -> u32`:
1. if map.nr_extents == 0 { return if self == &INIT_USER_NS.uid_map { id } else { u32::MAX } }.
2. if map.nr_extents <= UID_GID_MAP_MAX_BASE_EXTENTS {
   - for e in &map.extent[..map.nr_extents] {
     if e.first <= id && id < e.first + e.count { return id - e.first + e.lower_first }
     }; return u32::MAX.
   }.
3. let e = bsearch(&IdmapKey { map_up: false, id, count: 1 }, &map.forward[..map.nr_extents], cmp_map_id)?
4. id - e.first + e.lower_first.

`UserNs::map_id_up(map, id) -> u32`:  /* symmetric over reverse[] */.

`UserNs::make_kuid(ns, uid) -> KUid`:
1. KUid::new(UserNs::map_id_down(&ns.uid_map, uid)).

`UserNs::from_kuid(targ, kuid) -> u32`:
1. UserNs::map_id_up(&targ.uid_map, __kuid_val(kuid)).

`UserNs::from_kuid_munged(targ, kuid) -> u32`:
1. let uid = UserNs::from_kuid(targ, kuid).
2. if uid == u32::MAX { overflowuid } else { uid }.

`UserNs::map_write(file, buf, count, ppos, cap_setid, map, parent_map) -> Result<usize, Errno>`:
1. if *ppos != 0 || count >= PAGE_SIZE { return Err(EINVAL) }.
2. let kbuf = memdup_user_nul(buf, count)?
3. mutex_lock(&USERNS_STATE_MUTEX).
4. let map_ns = file.private_data.private_user_ns.
5. let mut new_map = UidGidMap::zero().
6. if map.nr_extents != 0 { return Err(EPERM) }.
7. if cap_valid(cap_setid) && !file_ns_capable(file, map_ns, CAP_SYS_ADMIN) { return Err(EPERM) }.
8. parse lines of "<first> <lower_first> <count>" decimal:
   - reject (u32)-1 sentinels.
   - reject overflow (first + count, lower_first + count).
   - reject mappings_overlap(&new_map, &ext).
   - reject (new_map.nr_extents + 1) == UID_GID_MAP_MAX_EXTENTS with another line.
   - insert_extent(&mut new_map, &ext)?
9. if new_map.nr_extents == 0 { return Err(EINVAL) }.
10. if !UserNs::new_idmap_permitted(file, map_ns, cap_setid, &new_map) { return Err(EPERM) }.
11. for ext in new_map iter mut {
    let lf = UserNs::map_id_range_down(parent_map, ext.lower_first, ext.count).
    if lf == u32::MAX { return Err(EPERM) }.
    ext.lower_first = lf.
    }.
12. UserNs::sort_idmaps(&mut new_map)?
13. install: copy inline or store forward/reverse.
14. smp_wmb().
15. map.nr_extents = new_map.nr_extents.
16. *ppos = count; ret = count.
17. mutex_unlock; kfree(kbuf); Ok(ret).

`UserNs::new_idmap_permitted(file, ns, cap_setid, new_map) -> bool`:
1. if cap_setid == CAP_SETUID && !UserNs::verify_root_map(file, ns, new_map) { return false }.
2. if new_map.nr_extents == 1 && new_map.extent[0].count == 1 && uid_eq(ns.owner, file.f_cred.euid) {
   - let id = new_map.extent[0].lower_first.
   - if cap_setid == CAP_SETUID {
     let uid = UserNs::make_kuid(ns.parent, id);
     if uid_eq(uid, file.f_cred.euid) { return true }
     }.
   - if cap_setid == CAP_SETGID {
     let gid = UserNs::make_kgid(ns.parent, id);
     if !(ns.flags & USERNS_SETGROUPS_ALLOWED) && gid_eq(gid, file.f_cred.egid) { return true }
     }.
   }.
3. if !cap_valid(cap_setid) { return true }.
4. if ns_capable(ns.parent, cap_setid) && file_ns_capable(file, ns.parent, cap_setid) { return true }.
5. false.

`UserNs::proc_setgroups_write(file, buf, count, ppos) -> Result<usize, Errno>`:
1. if *ppos != 0 || count >= 8 { return Err(EINVAL) }.
2. let kbuf = copy_from_user_array<8>(buf, count)?; kbuf[count] = 0.
3. let setgroups_allowed = match parse(&kbuf) {
   "allow" => true, "deny" => false, _ => return Err(EINVAL),
   }.
4. mutex_lock(&USERNS_STATE_MUTEX).
5. if setgroups_allowed {
   if !(ns.flags & USERNS_SETGROUPS_ALLOWED) { mutex_unlock; return Err(EPERM) }.
   } else {
   if ns.gid_map.nr_extents != 0 { mutex_unlock; return Err(EPERM) }.
   ns.flags &= !USERNS_SETGROUPS_ALLOWED.
   }.
6. mutex_unlock; *ppos = count; Ok(count).

`UserNs::may_setgroups(ns) -> bool`:
1. mutex_lock(&USERNS_STATE_MUTEX).
2. let ok = ns.gid_map.nr_extents != 0 && (ns.flags & USERNS_SETGROUPS_ALLOWED) != 0.
3. mutex_unlock; ok.

`UserNs::in_userns(ancestor, child) -> bool`:
1. let mut ns = child; while ns.level > ancestor.level { ns = ns.parent }.
2. ns == ancestor.

`UserNs::free(work)`:
1. let mut ns = container_of(work, UserNamespace, work).
2. loop {
   - let parent = ns.parent.
   - ns_tree_remove(ns).
   - free heap extents of uid_map / gid_map / projid_map if > UID_GID_MAP_MAX_BASE_EXTENTS.
   - #[cfg(binfmt_misc)] kfree(ns.binfmt_misc).
   - UserNs::retire_sysctls(ns).
   - key_free_user_ns(ns).
   - ns_common_free(&ns.ns).
   - kfree_rcu(ns, ns.ns_rcu).
   - UserNs::dec(ns.ucounts).
   - ns = parent.
   - if !ns_ref_put(&ns.ns) { break }.
   }.

### Out of Scope

- kernel/cred.c — credential allocation / commit lifecycle (covered separately if expanded)
- kernel/capability.c — capable() / ns_capable() core (covered in `capability.md`)
- kernel/ucount.c — UCOUNT_* counter / rlimit machinery (covered separately if expanded)
- kernel/groups.c — `setgroups`/`getgroups`/group_info (covered separately if expanded)
- kernel/nsproxy.c — per-task ns container (covered separately)
- kernel/pid_namespace.c (covered in `pid-namespace.md`)
- kernel/utsname.c / ipc/namespace.c / fs/mount.c / kernel/cgroup/namespace.c (other ns types)
- fs/proc/namespaces.c / fs/proc/base.c — /proc/<pid>/ns/* glue
- security/commoncap.c LSM cap_capable defaults
- CONFIG_KEYS keyring per-user-ns details (security/keys/*)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct user_namespace` | per-ns descriptor | `UserNamespace` |
| `init_user_ns` | per-boot init user-ns | `INIT_USER_NS` (static) |
| `user_ns_cachep` | per-slab cache | `UserNs::SLAB` |
| `USERNS_SETGROUPS_ALLOWED` | per-flag: gid_map / setgroups allowed | `USERNS_SETGROUPS_ALLOWED` |
| `UID_GID_MAP_MAX_BASE_EXTENTS` (5) | per-inline-extent limit | `UID_GID_MAP_MAX_BASE_EXTENTS` |
| `UID_GID_MAP_MAX_EXTENTS` (340) | per-map total cap | `UID_GID_MAP_MAX_EXTENTS` |
| `userns_state_mutex` | per-map-write serializer | `UserNs::STATE_MUTEX` |
| `create_user_ns(new)` | per-unshare construct | `UserNs::create` |
| `unshare_userns(flags, new_cred)` | per-CLONE_NEWUSER dispatch | `UserNs::unshare` |
| `set_cred_user_ns(cred, ns)` | per-cred reset to CAP_FULL_SET in ns | `UserNs::set_cred_user_ns` |
| `enforced_nproc_rlimit()` | per-create RLIMIT_NPROC pull | `UserNs::enforced_nproc_rlimit` |
| `inc_user_namespaces(ns, uid)` / `dec_user_namespaces(uc)` | per-ucounts UCOUNT_USER_NAMESPACES | `UserNs::inc` / `UserNs::dec` |
| `setup_userns_sysctls(ns)` / `retire_userns_sysctls(ns)` | per-ns sysctl table | `UserNs::setup_sysctls` / `retire_sysctls` |
| `free_user_ns(work)` | per-workqueue cascading free | `UserNs::free` |
| `__put_user_ns(ns)` | per-ref-drop ⟹ schedule_work(free) | `UserNs::put` |
| `make_kuid(ns, uid)` | per-userspace→kernel uid map-down | `UserNs::make_kuid` |
| `make_kgid(ns, gid)` | per-userspace→kernel gid map-down | `UserNs::make_kgid` |
| `make_kprojid(ns, projid)` | per-userspace→kernel projid map-down | `UserNs::make_kprojid` |
| `from_kuid(targ, kuid)` | per-kernel→target-ns uid map-up | `UserNs::from_kuid` |
| `from_kgid(targ, kgid)` | per-kernel→target-ns gid map-up | `UserNs::from_kgid` |
| `from_kprojid(targ, kprojid)` | per-kernel→target-ns projid map-up | `UserNs::from_kprojid` |
| `from_kuid_munged(targ, kuid)` | per-from_kuid with `overflowuid` fallback | `UserNs::from_kuid_munged` |
| `from_kgid_munged(targ, kgid)` | per-from_kgid with `overflowgid` fallback | `UserNs::from_kgid_munged` |
| `from_kprojid_munged(targ, kprojid)` | per-from_kprojid with OVERFLOW_PROJID | `UserNs::from_kprojid_munged` |
| `map_id_down(map, id)` | per-userspace→kernel range map (single id) | `UserNs::map_id_down` |
| `map_id_range_down(map, id, cnt)` | per-userspace→kernel range map (range) | `UserNs::map_id_range_down` |
| `map_id_up(map, id)` | per-kernel→userspace range map (single id) | `UserNs::map_id_up` |
| `cmp_map_id(k, e)` | per-bsearch comparator | `UserNs::cmp_map_id` |
| `sort_idmaps(map)` | per-write-time forward+reverse sort | `UserNs::sort_idmaps` |
| `insert_extent(map, ext)` | per-write extent insert | `UserNs::insert_extent` |
| `mappings_overlap(new_map, ext)` | per-write overlap check | `UserNs::mappings_overlap` |
| `verify_root_map(file, ns, new_map)` | per-uid_map: root-mapping policy | `UserNs::verify_root_map` |
| `new_idmap_permitted(file, ns, cap_setid, map)` | per-write idmap permission | `UserNs::new_idmap_permitted` |
| `map_write(file, buf, count, ppos, cap, map, parent_map)` | per-write extent-parse + install | `UserNs::map_write` |
| `proc_uid_map_write` | per-/proc/<pid>/uid_map write | `UserNs::proc_uid_map_write` |
| `proc_gid_map_write` | per-/proc/<pid>/gid_map write | `UserNs::proc_gid_map_write` |
| `proc_projid_map_write` | per-/proc/<pid>/projid_map write | `UserNs::proc_projid_map_write` |
| `proc_setgroups_show` / `_write` | per-/proc/<pid>/setgroups | `UserNs::proc_setgroups_*` |
| `userns_may_setgroups(ns)` | per-setgroups syscall gate | `UserNs::may_setgroups` |
| `in_userns(ancestor, child)` | per-ancestor walk | `UserNs::in_userns` |
| `current_in_userns(target_ns)` | per-`in_userns(target, current_user_ns())` | `UserNs::current_in_userns` |
| `userns_get` / `userns_put` / `userns_install` / `userns_owner` | per-/proc/<pid>/ns/user ops | `UserNs::ns_get` / `_put` / `_install` / `_owner` |
| `ns_get_owner(ns_common)` | per-`ioctl(NS_GET_USERNS)` | `UserNs::ns_get_owner` |
| `user_namespaces_init` (initcall) | per-boot slab + init_user_ns nstree-add | `UserNs::init` |

### compatibility contract

REQ-1: `struct user_namespace`:
- ns: `struct ns_common` — per-/proc/.../ns/user backing (inum, ops, ref).
- uid_map / gid_map / projid_map: `struct uid_gid_map` — each has nr_extents, inline `extent[UID_GID_MAP_MAX_BASE_EXTENTS]` (5 slots) + heap forward[] / reverse[] (sorted) when > 5.
- parent: `*UserNamespace` — NULL only for init_user_ns.
- level: i32 — depth (init = 0, child = 1, ...); capped at **33** (`level > 32 ⟹ -ENOSPC`).
- owner: kuid_t — euid of creator at unshare; pinned mapping in parent.
- group: kgid_t — egid of creator at unshare; pinned mapping in parent.
- flags: u32 — `USERNS_SETGROUPS_ALLOWED` (1 << 0); inherited from parent at create.
- ucount_max[UCOUNT_COUNTS]: `int[]` — per-resource caps for this ns (INT_MAX by default).
- ucounts: `*Ucounts` — slot in parent's UCOUNT_USER_NAMESPACES.
- rlimit_max[UCOUNT_RLIMIT_COUNTS]: per-ns RLIMIT_NPROC / _MSGQUEUE / _SIGPENDING / _MEMLOCK ceiling (set from creator's rlimits at create).
- parent_could_setfcap: bool — captured `cap_raised(new.cap_effective, CAP_SETFCAP)` at create-time; gates writing file-caps inside ns.
- work: `struct work_struct` — free_user_ns (cascading).
- keyring_name_list / keyring_sem (CONFIG_KEYS): per-ns keyring.
- persistent_keyring_register (CONFIG_PERSISTENT_KEYRINGS): per-user persistent keyring.
- binfmt_misc (CONFIG_BINFMT_MISC): per-ns binfmt_misc table.

REQ-2: `init_user_ns` (`static struct user_namespace`):
- level = 0; parent = NULL.
- uid_map / gid_map / projid_map: empty (identity mapping is the default for init_user_ns alone — all kuid_t == uid_t).
- ucount_max[*] = INT_MAX.
- flags = USERNS_SETGROUPS_ALLOWED.
- Globally singular; never freed.

REQ-3: Per-depth cap (NN-nested limit):
- `create_user_ns()`: `parent_ns.level > 32 ⟹ -ENOSPC`.
- Resulting ns has `level = parent.level + 1`; max ns.level = **33**.
- Prevents unbounded nesting (per-stack and per-translation-cost).

REQ-4: `create_user_ns(new: *Cred)`:
- parent_ns = new.user_ns; owner = new.euid; group = new.egid.
- parent_ns.level > 32 ⟹ `-ENOSPC`.
- `ucounts = inc_user_namespaces(parent_ns, owner)`; NULL ⟹ `-ENOSPC` (UCOUNT_USER_NAMESPACES exceeded).
- `current_chrooted() ⟹ -EPERM` (caller in a chroot cannot unshare a user-ns — would defeat chroot policy).
- `!kuid_has_mapping(parent_ns, owner) ∨ !kgid_has_mapping(parent_ns, group) ⟹ -EPERM` (creator must have a name in parent_ns for accountability).
- `security_create_user_ns(new) < 0 ⟹` LSM error.
- ns = `kmem_cache_zalloc(user_ns_cachep, GFP_KERNEL)`; NULL ⟹ `-ENOMEM`.
- ns.parent_could_setfcap = `cap_raised(new.cap_effective, CAP_SETFCAP)`.
- `ns_common_init(ns)` allocates inum.
- ns.parent = parent_ns (no get; new owns the parent reference via cred chain).
- ns.level = parent.level + 1; ns.owner = owner; ns.group = group.
- INIT_WORK(&ns.work, free_user_ns).
- for i in 0..UCOUNT_COUNTS: ns.ucount_max[i] = INT_MAX.
- `set_userns_rlimit_max(ns, UCOUNT_RLIMIT_NPROC, enforced_nproc_rlimit())`.
- `set_userns_rlimit_max(ns, UCOUNT_RLIMIT_MSGQUEUE, rlimit(RLIMIT_MSGQUEUE))`.
- `set_userns_rlimit_max(ns, UCOUNT_RLIMIT_SIGPENDING, rlimit(RLIMIT_SIGPENDING))`.
- `set_userns_rlimit_max(ns, UCOUNT_RLIMIT_MEMLOCK, rlimit(RLIMIT_MEMLOCK))`.
- ns.ucounts = ucounts.
- mutex_lock(&userns_state_mutex); ns.flags = parent_ns.flags; mutex_unlock — inherit USERNS_SETGROUPS_ALLOWED.
- (CONFIG_KEYS) INIT_LIST_HEAD(&ns.keyring_name_list); init_rwsem(&ns.keyring_sem).
- `setup_userns_sysctls(ns)`; NULL ⟹ `-ENOMEM`.
- `set_cred_user_ns(new, ns)` — see REQ-5.
- `ns_tree_add(ns)`.
- return 0.

REQ-5: `set_cred_user_ns(cred, user_ns)`:
- cred.securebits = `SECUREBITS_DEFAULT`.
- cred.cap_inheritable = `CAP_EMPTY_SET`.
- cred.cap_permitted = `CAP_FULL_SET`.
- cred.cap_effective = `CAP_FULL_SET`.
- cred.cap_ambient = `CAP_EMPTY_SET`.
- cred.cap_bset = `CAP_FULL_SET`.
- (CONFIG_KEYS) key_put(cred.request_key_auth); cred.request_key_auth = NULL.
- cred.user_ns = user_ns.
- Effect: caller is **root inside** the new ns (CAP_FULL_SET), but caps are bound to user_ns ⟹ outside the ns, `ns_capable(parent_ns, ...)` is false unless the kuid match grants it.

REQ-6: `unshare_userns(unshare_flags, new_cred)`:
- !(flags & CLONE_NEWUSER) ⟹ 0.
- cred = `prepare_creds()`; NULL ⟹ `-ENOMEM`.
- err = `create_user_ns(cred)`; err ⟹ put_cred(cred); else *new_cred = cred.

REQ-7: `__put_user_ns(ns)`:
- `schedule_work(&ns.work)` — defer free to workqueue (free_user_ns cascades via parent chain).

REQ-8: `free_user_ns(work)`:
- Loop:
  - parent = ns.parent.
  - ns_tree_remove(ns).
  - if ns.gid_map.nr_extents > UID_GID_MAP_MAX_BASE_EXTENTS: kfree(forward / reverse).
  - same for uid_map / projid_map.
  - (CONFIG_BINFMT_MISC) kfree(ns.binfmt_misc).
  - retire_userns_sysctls(ns); key_free_user_ns(ns); ns_common_free(&ns.ns).
  - `kfree_rcu(ns, ns.ns_rcu)` — RCU-delayed slab free.
  - dec_user_namespaces(ucounts).
  - ns = parent.
- Continue while `ns_ref_put(parent)` (cascading).

REQ-9: `make_kuid(ns, uid) -> kuid_t`:
- `KUIDT_INIT(map_id_down(&ns.uid_map, uid))`.
- Unmapped uid ⟹ kuid with `__kuid_val == (u32)-1` (INVALID_UID); `uid_valid()` returns false.

REQ-10: `from_kuid(targ, kuid) -> uid_t`:
- `map_id_up(&targ.uid_map, __kuid_val(kuid))`.
- Unmapped ⟹ `(uid_t)-1` (caller checks).
- Special case: init_user_ns has empty map but is identity-mapped — `map_id_up(&init_user_ns.uid_map, x) == x` via the "no extents ⟹ identity" branch (see `map_id_range_up_base` for nr_extents==0).

REQ-11: `from_kuid_munged(targ, kuid) -> uid_t`:
- `uid = from_kuid(targ, kuid); if uid == (uid_t)-1 { uid = overflowuid }; return uid`.
- Used by syscalls that have no way to return failure (`getuid`, `stat.st_uid`).

REQ-12: Analogous for gid / projid (`make_kgid` / `from_kgid` / `from_kgid_munged`, `make_kprojid` / `from_kprojid` / `from_kprojid_munged`).
- projid uses `OVERFLOW_PROJID` for munged fallback.

REQ-13: `map_id_down(map, id)`:
- nr_extents == 0 ⟹ for init_user_ns: identity (id ⟹ id); for others: id ⟹ (u32)-1 (no mapping yet).
- nr_extents ≤ `UID_GID_MAP_MAX_BASE_EXTENTS` (5): linear search through inline `extent[]`.
- else: bsearch through sorted `forward[]` (sorted by `first`).
- Match: `e.first ≤ id < e.first + e.count` ⟹ `id - e.first + e.lower_first`.
- No match ⟹ `(u32)-1`.

REQ-14: `map_id_up(map, id)`:
- nr_extents == 0 ⟹ identity for init_user_ns; (u32)-1 otherwise.
- nr_extents ≤ 5: linear search of inline `extent[]` matching `e.lower_first ≤ id < e.lower_first + e.count`.
- else: bsearch through sorted `reverse[]` (sorted by `lower_first`).
- Match: `id - e.lower_first + e.first`.

REQ-15: `uid_eq(a, b)` / `gid_eq(a, b)`: `__kuid_val(a) == __kuid_val(b)` — strict compare on the global kernel id (not the per-ns uid_t).

REQ-16: `/proc/<pid>/uid_map` write (`proc_uid_map_write` → `map_write`):
- !ns.parent ⟹ `-EPERM` (init_user_ns map not writable).
- `seq_ns != ns ∧ seq_ns != ns.parent` ⟹ `-EPERM` (writer must be in ns or its parent).
- Calls `map_write(file, buf, size, ppos, CAP_SETUID, &ns.uid_map, &ns.parent.uid_map)`.

REQ-17: `map_write(file, buf, count, ppos, cap_setid, map, parent_map)`:
- ppos != 0 ∨ count ≥ PAGE_SIZE ⟹ `-EINVAL` (single, ≤ 4KB write only).
- `memdup_user_nul(buf, count)`.
- mutex_lock(&userns_state_mutex).
- map.nr_extents != 0 ⟹ `-EPERM` (write-once; can never replace a non-empty map).
- `cap_valid(cap_setid) ∧ !file_ns_capable(file, map_ns, CAP_SYS_ADMIN) ⟹ -EPERM`.
- Parse lines of `<inside> <outside> <count>` triples (decimal); each:
  - inside / outside == (u32)-1 ⟹ `-EINVAL`.
  - inside + count overflow ⟹ `-EINVAL`.
  - outside + count overflow ⟹ `-EINVAL`.
  - `mappings_overlap(&new_map, &ext) ⟹ -EINVAL`.
  - `(new_map.nr_extents + 1) == UID_GID_MAP_MAX_EXTENTS ∧ next_line != NULL ⟹ -EINVAL` (cap at 340 extents).
  - `insert_extent(&new_map, &ext)`.
- `new_map.nr_extents == 0 ⟹ -EINVAL` (empty write rejected).
- `new_idmap_permitted(file, map_ns, cap_setid, &new_map) ⟹` else `-EPERM`.
- For each extent: `lower_first = map_id_range_down(parent_map, ext.lower_first, ext.count)`; (u32)-1 ⟹ `-EPERM` (cannot map to outside-id that isn't itself mapped in parent).
- `sort_idmaps(&new_map)` (forward by `first`, reverse by `lower_first`).
- Install: copy inline extent[] (≤5) or store forward/reverse pointers.
- `smp_wmb()` — readers must observe extent values before count.
- `map.nr_extents = new_map.nr_extents`.
- mutex_unlock + kfree(kbuf); return count.

REQ-18: `new_idmap_permitted(file, ns, cap_setid, new_map)`:
- `cap_setid == CAP_SETUID ∧ !verify_root_map(file, ns, new_map) ⟹ false`.
- Single-extent count==1 mapping the **opener's own euid** (`uid_eq(ns.owner, cred.euid) ∧ make_kuid(ns.parent, extent[0].lower_first) == cred.euid`) ⟹ true (unprivileged identity mapping permitted for the creator).
- Same for gid with `(ns.flags & USERNS_SETGROUPS_ALLOWED) == 0` requirement (unprivileged gid map only when setgroups already denied).
- `!cap_valid(cap_setid) ⟹ true` (projid never requires capability).
- `ns_capable(ns.parent, cap_setid) ∧ file_ns_capable(file, ns.parent, cap_setid) ⟹ true` (parent-ns CAP_SETUID/SETGID grants any mapping).
- else false.

REQ-19: `verify_root_map(file, ns, new_map)`: per-uid_map specific rule that prevents the creator from mapping root inside without parent CAP_SETFCAP if they did not have it at unshare-time (closed CVE — `ns.parent_could_setfcap`).

REQ-20: `/proc/<pid>/gid_map` write (`proc_gid_map_write`): like uid_map but with `CAP_SETGID` and `ns.parent.gid_map`.

REQ-21: `/proc/<pid>/projid_map` write (`proc_projid_map_write`): like uid_map but with **cap_setid = -1** (no capability needed; project IDs are advisory).

REQ-22: `/proc/<pid>/setgroups` (`proc_setgroups_show` / `_write`):
- show: `(ns.flags & USERNS_SETGROUPS_ALLOWED) ? "allow" : "deny"`.
- write: input must be exactly `"allow"` or `"deny"` (no trailing junk).
- `"allow"`: only valid if currently `allow` (cannot re-enable after deny).
- `"deny"`: only valid if `ns.gid_map.nr_extents == 0` (cannot deny after gid_map written); sets `ns.flags &= ~USERNS_SETGROUPS_ALLOWED`.
- `userns_state_mutex` held.

REQ-23: `userns_may_setgroups(ns) -> bool`:
- `ns.gid_map.nr_extents != 0 ∧ (ns.flags & USERNS_SETGROUPS_ALLOWED)`.
- Gates `setgroups(2)` in `kernel/groups.c`.

REQ-24: `in_userns(ancestor, child) -> bool`:
- child.level < ancestor.level ⟹ false.
- Walk child.parent until level == ancestor.level; return ns == ancestor.

REQ-25: `current_in_userns(target_ns) -> bool`: `in_userns(target_ns, current_user_ns())`.

REQ-26: `userns_install(nsset, ns_common)` — `setns(2)`:
- `!ns_capable(ns_common.user_ns, CAP_SYS_ADMIN)` and per-callee checks; commits new creds with `set_cred_user_ns(new_cred, ns)`.

REQ-27: `ns_get_owner(ns_common)`: returns owning user-ns for any ns_common (used by `ioctl(NS_GET_USERNS)` on non-user namespaces).

REQ-28: `user_namespaces_init` (initcall): `user_ns_cachep = KMEM_CACHE(user_namespace, SLAB_PANIC)`; `ns_tree_add(&init_user_ns)`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `level_bounded` | INVARIANT | per-create: ns.level ∈ [1, 33]. |
| `parent_chain_terminates_at_init` | INVARIANT | per-ns: walking parent chain reaches &INIT_USER_NS in ≤ level steps. |
| `cred_full_caps_in_new_ns` | INVARIANT | per-set_cred_user_ns: cap_effective == CAP_FULL_SET ∧ cap_bset == CAP_FULL_SET. |
| `chroot_blocks_unshare` | INVARIANT | per-create: current_chrooted() ⟹ Err(EPERM). |
| `creator_must_have_mapping` | INVARIANT | per-create: kuid_has_mapping(parent_ns, owner) ∧ kgid_has_mapping(parent_ns, group). |
| `map_write_once` | INVARIANT | per-map_write: prior map.nr_extents != 0 ⟹ Err(EPERM). |
| `map_write_under_state_mutex` | INVARIANT | per-map_write: userns_state_mutex held throughout install. |
| `setgroups_deny_irreversible` | INVARIANT | per-proc_setgroups_write: once USERNS_SETGROUPS_ALLOWED cleared, never re-set. |
| `setgroups_deny_blocked_after_gid_map` | INVARIANT | per-proc_setgroups_write: gid_map.nr_extents != 0 ⟹ deny rejected. |
| `extent_count_capped` | INVARIANT | per-map_write: new_map.nr_extents ≤ UID_GID_MAP_MAX_EXTENTS (340). |
| `extents_non_overlapping` | INVARIANT | per-map_write: mappings_overlap rejected. |
| `map_id_arith_no_overflow` | INVARIANT | per-extent: first + count > first ∧ lower_first + count > lower_first. |
| `forward_reverse_sorted` | INVARIANT | per-sort_idmaps: forward sorted by first; reverse sorted by lower_first. |
| `uid_eq_compares_kuid_val` | INVARIANT | per-uid_eq: __kuid_val(a) == __kuid_val(b); never compares uid_t. |
| `from_kuid_munged_total` | INVARIANT | per-from_kuid_munged: never returns (uid_t)-1. |

### Layer 2: TLA+

`kernel/user-namespace.tla`:
- Per-state: forest of user_namespaces, per-ns uid_map / gid_map / projid_map, per-ns flags, per-cred user_ns + caps.
- Per-action: Unshare / WriteUidMap / WriteGidMap / WriteSetgroups / Setns / Free.
- Properties:
  - `safety_max_depth` — per-Unshare: level ≤ 33.
  - `safety_init_ns_immortal` — per-state: INIT_USER_NS.refcount ≥ 1.
  - `safety_map_write_once` — per-WriteUidMap/Gid/Projid: each map writable at most once.
  - `safety_no_caps_outside_ns` — per-cred: cap_effective applies only via ns_capable(target_ns, cap) ∧ in_userns(target_ns, cred.user_ns).
  - `safety_setgroups_deny_monotonic` — per-flags: USERNS_SETGROUPS_ALLOWED monotonically non-increasing.
  - `safety_gid_map_after_deny_unprivileged_only` — per-WriteGidMap: if !USERNS_SETGROUPS_ALLOWED then only single-identity-mapping permitted unprivileged.
  - `safety_chroot_no_unshare` — per-Unshare: current_chrooted ⟹ refused.
  - `liveness_free_cascade_terminates` — per-Free: bounded by depth (≤ 33).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `UserNs::create` post: ns.level == parent.level + 1 ∧ new.user_ns == ns ∧ ns.flags == parent.flags ∧ caps = CAP_FULL_SET | `UserNs::create` |
| `UserNs::map_id_down` post: result ∈ {extent.lower_first + delta where extent.first ≤ id < extent.first + extent.count} ∪ {u32::MAX} | `UserNs::map_id_down` |
| `UserNs::map_id_up` post: symmetric over reverse extents | `UserNs::map_id_up` |
| `UserNs::make_kuid` ∘ `UserNs::from_kuid(ns, ·)` post: round-trip identity for mapped uids | `UserNs::make_kuid` / `from_kuid` |
| `UserNs::from_kuid_munged` post: ret == from_kuid OR overflowuid | `UserNs::from_kuid_munged` |
| `UserNs::map_write` post: !err ⟹ map.nr_extents == parsed_extents ∧ each ext.lower_first re-mapped via parent_map ∧ sorted | `UserNs::map_write` |
| `UserNs::new_idmap_permitted` post: returns true ⟹ either parent CAP_SETUID/CAP_SETGID held, or unprivileged identity rule satisfied | `UserNs::new_idmap_permitted` |
| `UserNs::proc_setgroups_write("deny")` post: !err ⟹ (ns.flags & USERNS_SETGROUPS_ALLOWED) == 0 | `UserNs::proc_setgroups_write` |
| `UserNs::may_setgroups` post: ret ⟺ ns.gid_map.nr_extents != 0 ∧ (ns.flags & USERNS_SETGROUPS_ALLOWED) | `UserNs::may_setgroups` |
| `UserNs::in_userns(ancestor, child)` post: terminates in ≤ child.level - ancestor.level steps ∧ ret ⟺ ancestor in child.parent* | `UserNs::in_userns` |

### Layer 4: Verus/Creusot functional

`Per-unshare(CLONE_NEWUSER) → create_user_ns → set_cred_user_ns → uid_map write → make_kuid round-trip → from_kuid_munged in target ns` semantic equivalence per `man 7 user_namespaces`, `man 7 capabilities` (section "User namespaces and capabilities"), and `Documentation/admin-guide/namespaces/userns.rst`. `setgroups` interaction per `Documentation/admin-guide/namespaces/setgroups.rst`.

### hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

User-namespace reinforcement:

- **Per-`level > 32` ceiling (33-deep cap)** — defense against per-recursive-unshare stack/cred-walk explosion.
- **Per-UCOUNT_USER_NAMESPACES ucounts slot** — defense against per-unprivileged ns flood (default INT_MAX, sysctl tunable).
- **Per-`current_chrooted()` refusal** — defense against per-chroot-escape via user-ns.
- **Per-`kuid_has_mapping(parent, owner)` requirement** — defense against per-orphan-creator (must be nameable for accountability / kill / signal).
- **Per-`security_create_user_ns` LSM hook** — defense against per-LSM-bypass via unprivileged ns creation.
- **Per-`set_cred_user_ns` CAP_FULL_SET bounded to user_ns** — defense against per-capability-escalation into parent ns.
- **Per-`parent_could_setfcap` captured at create** — defense against per-CVE-2021-22555-style root-mapping after dropping CAP_SETFCAP.
- **Per-write-once uid_map / gid_map / projid_map** — defense against per-mid-life mapping swap.
- **Per-`userns_state_mutex` map-write serializer** — defense against per-torn extent install.
- **Per-`smp_wmb` after extent install** — defense against per-reader observing count > extents.
- **Per-`UID_GID_MAP_MAX_EXTENTS` (340)** — defense against per-extent-table memory exhaustion.
- **Per-`mappings_overlap` rejection** — defense against per-ambiguous translation.
- **Per-`map_id_range_down(parent, lower_first, count)` check** — defense against per-mapping-to-unmapped parent id (containment).
- **Per-`USERNS_SETGROUPS_ALLOWED` monotonically non-increasing** — defense against per-CVE-2014-8989 (re-enabling setgroups to bypass /etc/group restrictions).
- **Per-`/proc/<pid>/setgroups "deny"` required before unprivileged gid_map** — chain-of-trust enforcement.
- **Per-`file_ns_capable(file, ns.parent, cap)`** — defense against per-cred-laundering by reopening the map file.
- **Per-`uid_eq` strict-`__kuid_val` compare** — defense against per-cross-ns aliasing (same uid_t, different kuid).
- **Per-`from_kuid_munged` overflow fallback** — defense against per-stat/getuid failure on unmapped target ns.
- **Per-`RCU-delayed user_namespace free` (`kfree_rcu`)** — defense against per-UAF on concurrent `current_user_ns()` readers.

