---
title: "Tier-3: security/selinux/hooks.c — SELinux LSM hooks"
tags: ["tier-3", "security", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

SELinux (Security-Enhanced Linux) is a **type-enforcement / role-based** mandatory access control LSM. Every kernel object (task / cred / inode / file / superblock / sock / msg / shm / sem / key / bpf / perf_event) carries a 32-bit **Security IDentifier (SID)** that resolves through the policy database to a `security_context_t` (`user:role:type:level`). Every operation that crosses an access boundary is gated through `avc_has_perm(ssid, tsid, tclass, requested, audit)` against the **Access Vector Cache (AVC)**; misses fall through to the security server which computes the AV from policy. `hooks.c` registers `selinux_hooks[]` covering binder, ptrace, capable, file/inode/sb, network/socket, IPC, key, bpf, perf, io_uring, xattr/secctx, mmap/mprotect, and exec transitions. Per-`bprm_creds_for_exec`: derives the new SID via `security_transition_sid` and enforces `PROCESS__TRANSITION`, `FILE__ENTRYPOINT`, `NNP`/`nosuid` bounded-transition rules, then `PROCESS__NOATSECURE` sets `bprm->secureexec`. Per-`task_security_struct.avdcache`: 4-entry directional AV cache speeds `inode_permission`. Per-`selinux_state.enforcing`: enforcing vs permissive mode; per-`selinux_state.policy`: RCU-protected current policy. Critical for: type-enforced privilege containment, Android binder mediation, exec-time domain transitions, fine-grained capability-class checks.

This Tier-3 covers `security/selinux/hooks.c` (~7979 lines).

### Acceptance Criteria

- [ ] AC-1: `selinux_init` registers all `selinux_hooks[]` entries via `security_add_hooks` and `selinux_state.initialized` flips to true.
- [ ] AC-2: `selinux_bprm_creds_for_exec` derives `new.sid` via `security_transition_sid` when no explicit `exec_sid` and validates `PROCESS__TRANSITION` + `FILE__ENTRYPOINT`.
- [ ] AC-3: NNP / nosuid execve to non-bounded SID returns `-EPERM` / `-EACCES`.
- [ ] AC-4: `selinux_inode_permission` returns 0 when task AVC cache hit grants `perms` and policy seqno matches.
- [ ] AC-5: `selinux_inode_setxattr` on `security.selinux` requires `FILE__RELABELFROM`, `FILE__RELABELTO`, and `FILESYSTEM__ASSOCIATE`.
- [ ] AC-6: `selinux_socket_create` consults `sockcreate_sid` when set; else `current_sid()`.
- [ ] AC-7: `selinux_capable` maps cap to `SECCLASS_CAPABILITY*` and the correct per-cap permission bit.
- [ ] AC-8: `selinux_ptrace_access_check` denies when `PROCESS__PTRACE` is not in the AV decision.
- [ ] AC-9: `selinux_binder_transaction` checks `BINDER__CALL` and, on SID-mismatch, `BINDER__TRANSFER`.
- [ ] AC-10: Per-permissive mode (`enforcing_enabled() == false`): hooks log but return 0.
- [ ] AC-11: Per-AVD_FLAGS_PERMISSIVE-type: returns 0 even in enforcing.
- [ ] AC-12: `selinux_bprm_committing_creds` closes inherited fds not authorized for `new.sid`.
- [ ] AC-13: `selinux_file_permission` fastpath returns 0 when cred / inode / policy-seqno unchanged since open.
- [ ] AC-14: `selinux_sk_alloc_security` initializes `sid = peer_sid = SECINITSID_UNLABELED`, `sclass = SECCLASS_SOCKET`.
- [ ] AC-15: `selinux_state.policy_mutex` serializes policy reload; readers use RCU.

### Architecture

```
struct SelinuxState {
  enforcing: AtomicBool,            // CONFIG_SECURITY_SELINUX_DEVELOP
  initialized: AtomicBool,
  policycap: [bool; POLICYDB_CAP_MAX],
  status_page: Option<*Page>,
  status_lock: Mutex<()>,
  policy: RcuPtr<SelinuxPolicy>,
  policy_mutex: Mutex<()>,
}

struct CredSec {
  osid: u32,
  sid: u32,
  exec_sid: u32,
  create_sid: u32,
  keycreate_sid: u32,
  sockcreate_sid: u32,
}

struct TaskSec {
  avdcache: TaskAvdCache,           // 4-entry directional AVC
}

struct InodeSec {
  inode: *Inode,
  list: ListNode,
  task_sid: u32,
  sid: u32,
  sclass: u16,
  initialized: LabelState,
  lock: SpinLock,
}

struct FileSec  { sid: u32, fown_sid: u32, isid: u32, pseqno: u32 }
struct SbSec    { sid: u32, def_sid: u32, mntpoint_sid: u32, creator_sid: u32,
                  behavior: u16, flags: u16, lock: Mutex<()>,
                  isec_head: List<InodeSec>, isec_lock: SpinLock }
struct SkSec    { nlbl_state: NlblState, nlbl_secattr: *NetlblSecattr,
                  sid: u32, peer_sid: u32, sclass: u16,
                  sctp_assoc_state: SctpAssocState }
```

`Selinux::bprm_creds_for_exec(bprm) -> Result<()>`:
1. old = selinux_cred(current_cred()); new = selinux_cred(bprm.cred).
2. isec = inode_security(file_inode(bprm.file)).
3. if isec.sclass != SECCLASS_FILE ∧ != SECCLASS_MEMFD_FILE: WARN; return -EACCES.
4. new.sid = old.sid; new.osid = old.sid; create/keycreate/sockcreate = 0.
5. if !selinux_initialized(): new.sid = SECINITSID_INIT; new.exec_sid = 0; return Ok.
6. if old.exec_sid != 0:
   - new.sid = old.exec_sid; new.exec_sid = 0.
   - check_nnp_nosuid(bprm, old, new)?;
7. else:
   - security_transition_sid(old.sid, isec.sid, SECCLASS_PROCESS, None, &new.sid)?
   - if check_nnp_nosuid(bprm, old, new).is_err(): new.sid = old.sid.
8. ad = AuditData::file(bprm.file).
9. if new.sid == old.sid:
   - avc_has_perm(old.sid, isec.sid, isec.sclass, FILE__EXECUTE_NO_TRANS, &ad)?;
10. else:
    - avc_has_perm(old.sid, new.sid, SECCLASS_PROCESS, PROCESS__TRANSITION, &ad)?;
    - avc_has_perm(new.sid, isec.sid, isec.sclass, FILE__ENTRYPOINT, &ad)?;
    - if bprm.unsafe & LSM_UNSAFE_SHARE: avc(PROCESS__SHARE)? -> -EPERM.
    - if bprm.unsafe & LSM_UNSAFE_PTRACE: ptsid = ptrace_parent_sid(); avc(ptsid, new.sid, PROCESS__PTRACE)?.
    - bprm.per_clear |= PER_CLEAR_ON_SETID.
    - bprm.secureexec |= !avc_ok(old.sid, new.sid, SECCLASS_PROCESS, PROCESS__NOATSECURE).
11. Ok(()).

`Selinux::inode_permission(inode, requested) -> Result<()>`:
1. mask = requested & (MAY_READ|MAY_WRITE|MAY_EXEC|MAY_APPEND).
2. if mask == 0: return Ok(()).
3. tsec = selinux_task(current); sid = current_sid().
4. if tsec.avdcache_permnoaudit(sid): return Ok(()).
5. isec = inode_security_rcu(inode, requested & MAY_NOT_BLOCK)?.
6. perms = file_mask_to_av(inode.i_mode, mask).
7. /* Try task-local AVC */
8. match tsec.avdcache_search(isec):
   - Hit(avdc) => denied = perms & !avdc.avd.allowed; rc = if denied != 0 ∧ enforcing ∧ !permissive { -EACCES } else { 0 }.
   - Miss => Avc::has_perm_noaudit(sid, isec.sid, isec.sclass, perms, 0, &avd); tsec.avdcache_update(isec, &avd).
9. audited = avc_audit_required(perms, &avd, rc, FILE__AUDIT_ACCESS-if-MAY_ACCESS, &denied).
10. if audited: audit_inode_permission(inode, perms, audited, denied, rc).
11. rc.

`Selinux::socket_create(family, type, protocol, kern) -> Result<()>`:
1. if kern: return Ok(()).
2. sid = if old_crsec.sockcreate_sid != 0 { old_crsec.sockcreate_sid } else { current_sid() }.
3. sclass = socket_type_to_security_class(family, type, protocol).
4. avc_has_perm(current_sid(), sid, sclass, SOCKET__CREATE, None).

`Selinux::sk_alloc_security(sk, family, gfp) -> Result<()>`:
1. sksec = selinux_sock(sk).
2. sksec.sid = SECINITSID_UNLABELED.
3. sksec.peer_sid = SECINITSID_UNLABELED.
4. sksec.sclass = SECCLASS_SOCKET.
5. selinux_netlbl_sk_security_reset(sksec).
6. Ok(()).

`Selinux::capable(cred, ns, cap, opts) -> Result<()>`:
1. initns = (ns == &init_user_ns).
2. sclass = match (cap < 32, initns) {
   - (true, true)  => SECCLASS_CAPABILITY,
   - (false, true) => SECCLASS_CAPABILITY2,
   - (true, false) => SECCLASS_CAP_USERNS,
   - (false, false) => SECCLASS_CAP2_USERNS,
   }.
3. perm = 1 << (cap & 0x1f).
4. audit = (opts & CAP_OPT_NOAUDIT) ? skip : full.
5. avc_has_perm(cred_sid(cred), cred_sid(cred), sclass, perm, &ad).

`check_nnp_nosuid(bprm, old, new) -> Result<()>`:
1. nnp = bprm.unsafe & LSM_UNSAFE_NO_NEW_PRIVS.
2. nosuid = !mnt_may_suid(bprm.file.f_path.mnt).
3. if !nnp ∧ !nosuid: return Ok.
4. if new.sid == old.sid: return Ok.
5. if selinux_policycap_nnp_nosuid_transition():
   - av = (nnp ? PROCESS2__NNP_TRANSITION : 0) | (nosuid ? PROCESS2__NOSUID_TRANSITION : 0).
   - if avc_has_perm(old.sid, new.sid, SECCLASS_PROCESS2, av, None).is_ok(): return Ok.
6. if security_bounded_transition(old.sid, new.sid).is_ok(): return Ok.
7. Err(if nnp { -EPERM } else { -EACCES }).

`Avc::has_perm(ssid, tsid, tclass, requested, ad) -> Result<()>`:
1. avd = Avc::lookup_or_compute(ssid, tsid, tclass).
2. denied = requested & !avd.allowed.
3. if denied == 0: return Ok(()).
4. if !enforcing_enabled() ∨ avd.flags & AVD_FLAGS_PERMISSIVE:
   - if avd.auditdeny & requested: avc_audit(ad, denied).
   - return Ok(()).
5. if avd.auditdeny & requested: avc_audit(ad, denied).
6. Err(-EACCES).

### Out of Scope

- `security/selinux/avc.c` AVC implementation (covered in `avc.md` Tier-3 if expanded)
- `security/selinux/ss/services.c` security server (covered in `ss-services.md` Tier-3 if expanded)
- `security/selinux/selinuxfs.c` selinuxfs (`/sys/fs/selinux`) interface (covered in `selinuxfs.md` Tier-3 if expanded)
- `security/selinux/netlabel.c` NetLabel integration (covered in `netlabel.md` Tier-3 if expanded)
- `security/selinux/xfrm.c` labeled IPsec (covered in `xfrm.md` Tier-3 if expanded)
- `security/selinux/ibpkey.c` / `ibendport.c` InfiniBand (covered separately if expanded)
- Policy compiler / `checkpolicy` userspace (out of kernel)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct selinux_state` | per-LSM global state (enforcing, policy, policycaps) | `SelinuxState` |
| `struct cred_security_struct` | per-cred SID block | `CredSec` |
| `struct task_security_struct` | per-task AVC cache | `TaskSec` |
| `struct inode_security_struct` | per-inode SID + class | `InodeSec` |
| `struct file_security_struct` | per-file SID + isid + pseqno | `FileSec` |
| `struct superblock_security_struct` | per-sb SID, behavior, mount-opts | `SbSec` |
| `struct sk_security_struct` | per-sock SID, peer SID, sclass, netlabel | `SkSec` |
| `selinux_init()` | per-init | `Selinux::init` |
| `selinux_hooks[]` | per-LSM hook table | `Selinux::HOOKS` |
| `avc_has_perm()` | per-access gate | `Avc::has_perm` |
| `avc_has_perm_noaudit()` | per-cache fastpath | `Avc::has_perm_noaudit` |
| `cred_has_capability()` | per-capable hook helper | `Selinux::cred_capable` |
| `selinux_bprm_creds_for_exec()` | per-exec SID derivation | `Selinux::bprm_creds_for_exec` |
| `selinux_bprm_committing_creds()` | per-pre-commit cleanup | `Selinux::bprm_committing_creds` |
| `selinux_bprm_committed_creds()` | per-post-commit cleanup | `Selinux::bprm_committed_creds` |
| `selinux_inode_permission()` | per-VFS access check | `Selinux::inode_permission` |
| `selinux_inode_setxattr()` | per-xattr write check | `Selinux::inode_setxattr` |
| `selinux_inode_alloc_security()` | per-inode allocation | `Selinux::inode_alloc_security` |
| `selinux_file_permission()` | per-file revalidation | `Selinux::file_permission` |
| `selinux_file_open()` | per-open AV check | `Selinux::file_open` |
| `selinux_socket_create()` | per-socket(2) class check | `Selinux::socket_create` |
| `selinux_sk_alloc_security()` | per-sock blob init | `Selinux::sk_alloc_security` |
| `selinux_task_alloc()` | per-fork task blob | `Selinux::task_alloc` |
| `selinux_cred_prepare()` | per-cred clone | `Selinux::cred_prepare` |
| `selinux_capable()` | per-capability check | `Selinux::capable` |
| `selinux_ptrace_access_check()` | per-ptrace gate | `Selinux::ptrace_access_check` |
| `selinux_binder_set_context_mgr()` | per-binder manager | `Selinux::binder_set_context_mgr` |
| `selinux_binder_transaction()` | per-binder IPC | `Selinux::binder_transaction` |
| `check_nnp_nosuid()` | per-NNP/nosuid bounded transition | `Selinux::check_nnp_nosuid` |
| `security_transition_sid()` | per-policy transition derivation | `SsServer::transition_sid` |
| `inode_security()` / `_rcu()` / `_novalidate()` | per-inode SID accessor | `Selinux::inode_security*` |
| `selinux_cred()` / `selinux_task()` / `selinux_file()` / `selinux_inode()` / `selinux_superblock()` / `selinux_sock()` | per-blob accessor (offset into `cred->security`, `task->security`, …) | `Selinux::*_blob` |

### compatibility contract

REQ-1: struct selinux_state:
- enforcing: bool (CONFIG_SECURITY_SELINUX_DEVELOP gated).
- initialized: bool — set by `selinux_mark_initialized` via `smp_store_release`.
- policycap[__POLICYDB_CAP_MAX]: per-policy capability bitmap.
- status_page: per-userspace status page; status_lock: per-update.
- policy: __rcu pointer to `struct selinux_policy`.
- policy_mutex: per-policy-update.

REQ-2: struct cred_security_struct (per-cred blob):
- osid: SID prior to last execve.
- sid: current SID (subject).
- exec_sid: pending exec-target SID (set by `setexeccon`).
- create_sid: fscreate SID for newly-created files.
- keycreate_sid: keycreate SID for newly-created keys.
- sockcreate_sid: sockcreate SID for newly-created sockets.

REQ-3: struct task_security_struct (per-task blob): avdcache:
- avdcache.sid: cached subject SID.
- avdcache.seqno: AVC sequence at cache fill.
- avdcache.dir[TSEC_AVDC_DIR_SIZE=4]: directional 4-entry AVC.
- avdcache.dir_spot: rotation index.
- avdcache.permissive_neveraudit: short-circuit flag.

REQ-4: struct inode_security_struct:
- inode: backptr.
- list: chained on `sbsec->isec_head` when deferred-init.
- task_sid: SID of creator task (per `inode_alloc_security`).
- sid: SID of this inode.
- sclass: SECCLASS_FILE / _DIR / _CHR_FILE / _SOCK_FILE / _ANON_INODE / _MEMFD_FILE / …
- initialized: LABEL_INVALID / _INITIALIZED / _PENDING.
- lock: spinlock for label transitions.

REQ-5: struct file_security_struct:
- sid: SID of the file description (cred at open-time).
- fown_sid: SID for SIGIO ownership.
- isid: inode SID snapshot at open-time.
- pseqno: policy seqno at open-time (used for revalidation).

REQ-6: struct superblock_security_struct:
- sid: SID of the superblock.
- def_sid: default file SID for this fs.
- mntpoint_sid: SECURITY_FS_USE_MNTPOINT context.
- creator_sid: SID of the mounting task.
- behavior: SECURITY_FS_USE_XATTR / _TRANS / _TASK / _GENFS / _MNTPOINT / _NATIVE / _NONE.
- flags: SE_SBINITIALIZED | SBLABEL_MNT | SE_SBLABELSUPP | …
- isec_head + isec_lock: pending-inode list for `selinux_complete_init`.

REQ-7: struct sk_security_struct:
- sid: subject SID of the socket.
- peer_sid: peer SID (NetLabel / labeled IPSEC / SCTP).
- sclass: SECCLASS_SOCKET / _TCP_SOCKET / _UDP_SOCKET / _UNIX_STREAM_SOCKET / …
- nlbl_state: NetLabel state machine (UNSET / REQUIRE / LABELED / REQSKB / CONNLABELED).
- nlbl_secattr: NetLabel attributes.
- sctp_assoc_state: SCTP association labeling state.

REQ-8: avc_has_perm(ssid, tsid, tclass, requested, ad):
- /* AVC fastpath */
- Look up (ssid, tsid, tclass) in AVC.
- /* Miss → security server */
- if !cached: security_compute_av(ssid, tsid, tclass, &avd).
- /* Apply enforcing */
- denied = requested & ~avd.allowed.
- if !denied: return 0.
- if !enforcing_enabled() ∨ avd.flags & AVD_FLAGS_PERMISSIVE: log + return 0.
- /* Audit */
- if avd.auditdeny & requested: avc_audit(...).
- return -EACCES.

REQ-9: selinux_bprm_creds_for_exec(bprm):
- /* Identify file class */
- isec = inode_security(file_inode(bprm->file)).
- if isec.sclass != SECCLASS_FILE ∧ != SECCLASS_MEMFD_FILE: return -EACCES.
- /* Default-keep current SID */
- new_crsec.sid = old_crsec.sid; new_crsec.osid = old_crsec.sid.
- new_crsec.{create_sid, keycreate_sid, sockcreate_sid} = 0.
- /* Per-pre-policy boot */
- if !selinux_initialized(): new_crsec.sid = SECINITSID_INIT; return 0.
- /* Explicit exec_sid (setexeccon) */
- if old_crsec.exec_sid:
  - new_crsec.sid = old_crsec.exec_sid; exec_sid = 0.
  - check_nnp_nosuid(bprm, old, new) — may fail.
- else:
  - /* Policy transition */
  - security_transition_sid(old.sid, isec.sid, SECCLASS_PROCESS, &new.sid).
  - check_nnp_nosuid; if fail: fall back to old SID.
- /* Per-no-transition */
- if new.sid == old.sid: avc_has_perm(old, isec.sid, FILE__EXECUTE_NO_TRANS).
- /* Per-transition: triple-check */
- else:
  - avc_has_perm(old, new, SECCLASS_PROCESS, PROCESS__TRANSITION).
  - avc_has_perm(new, isec.sid, isec.sclass, FILE__ENTRYPOINT).
  - if unsafe & LSM_UNSAFE_SHARE: PROCESS__SHARE.
  - if unsafe & LSM_UNSAFE_PTRACE: PROCESS__PTRACE from ptsid.
  - bprm->per_clear |= PER_CLEAR_ON_SETID.
  - bprm->secureexec |= !avc(PROCESS__NOATSECURE).

REQ-10: selinux_bprm_committing_creds(bprm):
- if new.sid == new.osid: return (no transition).
- flush_unauthorized_files(cred, current->files) — close fds the new SID cannot retain; replace with /dev/null.
- current->pdeath_signal = 0.
- if !avc(osid, sid, PROCESS__RLIMITINH): reset soft rlimits to min(hard, init.soft).

REQ-11: selinux_bprm_committed_creds(bprm):
- Recompute task-AVC cache for new SID.
- Wake pending signals/notifications under new SID.

REQ-12: selinux_inode_alloc_security(inode):
- isec.inode = inode.
- isec.sid = SECINITSID_UNLABELED.
- isec.sclass = SECCLASS_FILE.
- isec.task_sid = current_sid().
- isec.initialized = LABEL_INVALID.
- spin_lock_init(&isec.lock); INIT_LIST_HEAD(&isec.list).
- return 0.

REQ-13: selinux_inode_permission(inode, requested):
- mask = requested & (MAY_READ|MAY_WRITE|MAY_EXEC|MAY_APPEND).
- if !mask: return 0 (existence test).
- tsec = selinux_task(current).
- /* Permissive-noaudit fastpath */
- if task_avdcache_permnoaudit(tsec, sid): return 0.
- isec = inode_security_rcu(inode, MAY_NOT_BLOCK).
- perms = file_mask_to_av(inode->i_mode, mask).
- /* Try task AVC cache */
- if task_avdcache_search(tsec, isec, &avdc) == 0:
  - denied = perms & ~avdc.avd.allowed.
  - rc = enforcing_enabled() ∧ !permissive ⟹ -EACCES.
- else:
  - avc_has_perm_noaudit(sid, isec.sid, isec.sclass, perms, &avd).
  - task_avdcache_update(tsec, isec, &avd).
- audit_inode_permission(inode, perms, audited, denied, rc).

REQ-14: selinux_inode_setxattr(idmap, dentry, name, value, size, flags):
- /* If not security.selinux → require FILE__SETATTR */
- if !strcmp(name, XATTR_NAME_SELINUX):
  - parse value → newsid via security_context_to_sid.
  - avc_has_perm(current_sid(), isec.sid, isec.sclass, FILE__RELABELFROM).
  - avc_has_perm(current_sid(), newsid, isec.sclass, FILE__RELABELTO).
  - avc_has_perm(newsid, sbsec.sid, SECCLASS_FILESYSTEM, FILESYSTEM__ASSOCIATE).
- else: dentry_has_perm(cred, dentry, FILE__SETATTR).

REQ-15: selinux_file_permission(file, mask):
- /* Per-fast-path: cred unchanged since open */
- if sid == fsec.sid ∧ fsec.isid == isec.sid ∧ fsec.pseqno == avc_policy_seqno():
  - return 0.
- /* Per-revalidate */
- selinux_revalidate_file_permission(file, mask) — full AV recheck.

REQ-16: selinux_socket_create(family, type, protocol, kern):
- if kern: return 0.
- sclass = socket_type_to_security_class(family, type, protocol).
- sid = sockcreate_sid ? sockcreate_sid : current_sid().
- avc_has_perm(current_sid(), sid, sclass, SOCKET__CREATE, NULL).

REQ-17: selinux_sk_alloc_security(sk, family, gfp):
- sksec.peer_sid = SECINITSID_UNLABELED.
- sksec.sid = SECINITSID_UNLABELED.
- sksec.sclass = SECCLASS_SOCKET.
- selinux_netlbl_sk_security_reset(sksec).
- return 0.

REQ-18: selinux_capable(cred, ns, cap, opts):
- cred_has_capability(cred, cap, opts, ns == &init_user_ns).
- avc_has_perm with sclass = SECCLASS_CAPABILITY (or _CAPABILITY2 for cap >= 32, or _CAP_USERNS / _CAP2_USERNS for non-init-userns) and perm = 1 << (cap & 0x1f).

REQ-19: selinux_ptrace_access_check(child, mode):
- if mode & PTRACE_MODE_READ: avc(sid, task_sid_obj(child), SECCLASS_FILE, FILE__READ).
- else: avc(sid, task_sid_obj(child), SECCLASS_PROCESS, PROCESS__PTRACE).

REQ-20: selinux_binder_set_context_mgr(mgr):
- avc_has_perm(cred_sid(current_cred()), cred_sid(mgr), SECCLASS_BINDER, BINDER__SET_CONTEXT_MGR).

REQ-21: selinux_binder_transaction(from, to):
- /* Cross-context binder IPC */
- avc(from_sid, to_sid, SECCLASS_BINDER, BINDER__CALL).
- if from_sid != to_sid: avc(from_sid, to_sid, SECCLASS_BINDER, BINDER__TRANSFER).

REQ-22: check_nnp_nosuid(bprm, old, new):
- nnp = bprm->unsafe & LSM_UNSAFE_NO_NEW_PRIVS.
- nosuid = !mnt_may_suid(bprm->file->f_path.mnt).
- if !nnp ∧ !nosuid: return 0.
- if new.sid == old.sid: return 0.
- /* Policy-capability: nnp_nosuid_transition */
- if selinux_policycap_nnp_nosuid_transition():
  - av = (nnp ? NNP_TRANSITION : 0) | (nosuid ? NOSUID_TRANSITION : 0).
  - if !avc(old.sid, new.sid, SECCLASS_PROCESS2, av): return 0.
- /* Fallback: bounded transition */
- if !security_bounded_transition(old.sid, new.sid): return 0.
- return nnp ? -EPERM : -EACCES.

REQ-23: cred-blob accessors (offset into `cred->security`):
- selinux_cred(cred): cred->security + selinux_blob_sizes.lbs_cred.
- selinux_task(task): task->security + selinux_blob_sizes.lbs_task.
- selinux_file(file): file->f_security + selinux_blob_sizes.lbs_file.
- selinux_inode(inode): inode->i_security + selinux_blob_sizes.lbs_inode.
- selinux_superblock(sb): sb->s_security + selinux_blob_sizes.lbs_superblock.
- selinux_sock(sk): sk->sk_security + selinux_blob_sizes.lbs_sock.

REQ-24: SID classification (sclass):
- File class derived via inode_mode_to_security_class(mode).
- Socket class via socket_type_to_security_class(family, type, protocol).
- Process class always SECCLASS_PROCESS / _PROCESS2.
- Capability class chosen per cap-number / namespace.

REQ-25: selinux_init():
- memset(&selinux_state, 0, ...).
- enforcing_set(selinux_enforcing_boot).
- selinux_avc_init(); mutex_init policy/status.
- cred_init_security() — initial task gets SECINITSID_KERNEL.
- audit_cfg_lsm — register secctx for audit.
- avc_init / avtab_cache_init / ebitmap_cache_init / hashtab_cache_init.
- security_add_hooks(selinux_hooks, ARRAY_SIZE, &selinux_lsmid).
- avc_add_callback for netcache / LSM-notifier / audit-rule.

REQ-26: DEFINE_LSM(selinux):
- flags = LSM_FLAG_LEGACY_MAJOR | LSM_FLAG_EXCLUSIVE.
- enabled = &selinux_enabled_boot.
- blobs = &selinux_blob_sizes.
- init = selinux_init.
- initcall_device = selinux_initcall.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cred_blob_offset_within_bounds` | INVARIANT | per-selinux_cred: offset + sizeof(CredSec) ≤ cred->security blob size. |
| `task_avdcache_seqno_validated` | INVARIANT | per-avdcache hit: avdcache.seqno == avc_policy_seqno(). |
| `bprm_no_new_privs_strict` | INVARIANT | per-bprm_creds_for_exec: NNP + unbounded transition ⟹ Err. |
| `inode_permission_existence_test_zero` | INVARIANT | per-inode_permission: mask==0 ⟹ Ok. |
| `file_permission_fastpath_seqno_match` | INVARIANT | per-file_permission fastpath: fsec.pseqno == avc_policy_seqno() required. |
| `socket_create_kern_bypass` | INVARIANT | per-socket_create: kern==1 ⟹ Ok before any AVC. |
| `bprm_committing_clears_pdeath_signal` | INVARIANT | per-bprm_committing_creds: transition ⟹ pdeath_signal = 0. |
| `selinux_state_initialized_release` | INVARIANT | per-selinux_mark_initialized: smp_store_release ordering. |

### Layer 2: TLA+

`security/selinux/hooks.tla`:
- Per-bprm_creds + per-bprm_committing + per-bprm_committed + per-AVC-cache-miss-then-hit + per-policy-reload.
- Properties:
  - `safety_no_transition_without_entrypoint` — per-exec: new.sid != old.sid ⟹ FILE__ENTRYPOINT granted.
  - `safety_no_transition_without_process_transition` — per-exec: PROCESS__TRANSITION granted.
  - `safety_nnp_unbounded_denied` — per-NNP exec to unbounded SID ⟹ -EPERM.
  - `safety_policy_reload_atomic` — per-policy_mutex: readers see consistent policy.
  - `safety_avc_miss_falls_to_security_server` — per-AVC miss: security_compute_av called.
  - `liveness_per_inode_permission_terminates` — per-inode_permission: always returns.
  - `liveness_bprm_creds_completes` — per-bprm_creds_for_exec: terminates with Ok / Err.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Selinux::bprm_creds_for_exec` post: ret=Ok ⟹ AV(PROCESS__TRANSITION) ∧ AV(FILE__ENTRYPOINT) ∨ new.sid == old.sid | `Selinux::bprm_creds_for_exec` |
| `Selinux::inode_permission` post: ret=Ok ⟹ avd.allowed ⊇ perms ∨ permissive | `Selinux::inode_permission` |
| `Selinux::inode_setxattr` post: name == "security.selinux" ⟹ RELABELFROM ∧ RELABELTO ∧ ASSOCIATE | `Selinux::inode_setxattr` |
| `Avc::has_perm` post: ret=Ok ⟹ (requested & !allowed)==0 ∨ permissive | `Avc::has_perm` |
| `Selinux::socket_create` post: kern ⟹ ret=Ok ∧ no AVC call | `Selinux::socket_create` |
| `Selinux::sk_alloc_security` post: sid==SECINITSID_UNLABELED ∧ peer_sid==SECINITSID_UNLABELED ∧ sclass==SECCLASS_SOCKET | `Selinux::sk_alloc_security` |
| `check_nnp_nosuid` post: ret=Ok ⟹ bounded ∨ nnp_nosuid_transition AV | `Selinux::check_nnp_nosuid` |
| `Selinux::capable` post: sclass selection bijective in (cap≥32, !initns) | `Selinux::capable` |

### Layer 4: Verus/Creusot functional

`Per-execve → bprm_creds_for_exec (transition) → bprm_committing_creds (file-flush, rlimit-inh) → bprm_committed_creds → first-inode_permission post-exec (AVC miss → compute → cache fill) → file_permission fastpath` semantic equivalence: per-`Documentation/admin-guide/LSM/SELinux.rst` + per-NSA SELinux reference policy semantics.

### hardening

(Inherits row-1 features from `security/00-overview.md` § Hardening.)

SELinux-hooks reinforcement:

- **Per-cred-blob offset within `cred->security` bounds** — defense against per-blob-OOB read/write.
- **Per-RCU policy reload** — defense against per-stale-policy decision.
- **Per-policycap reads via `READ_ONCE`** — defense against per-capability-bit tearing.
- **Per-NNP / nosuid bounded-transition enforcement** — defense against per-setuid privilege-escalation.
- **Per-`FILE__ENTRYPOINT` requirement on SID transitions** — defense against per-arbitrary-domain entry.
- **Per-`PROCESS__SHARE` on `LSM_UNSAFE_SHARE`** — defense against per-shared-mm exec hijack.
- **Per-`PROCESS__PTRACE` on `LSM_UNSAFE_PTRACE`** — defense against per-tracer privilege-escalation.
- **Per-`flush_unauthorized_files` post-transition** — defense against per-fd-handle-bypass.
- **Per-task AVC cache invalidation on policy reload (seqno bump)** — defense against per-stale-AV decision.
- **Per-`SECINITSID_UNLABELED` default on new objects** — defense against per-implicit-label trust.
- **Per-`enforcing` flag write via `WRITE_ONCE`** — defense against per-tearing mode-switch.
- **Per-`selinux_state.initialized` `smp_store_release` ordering** — defense against per-pre-policy hook entry.
- **Per-`secureexec` set on no-NOATSECURE transition** — defense against per-LD_PRELOAD style abuse on domain change.
- **Per-binder `BINDER__SET_CONTEXT_MGR` once-only** — defense against per-binder-mgr hijack.

