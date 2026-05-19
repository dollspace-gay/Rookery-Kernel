# Tier-3: security/apparmor/lsm.c — AppArmor LSM hooks

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: security/apparmor/00-overview.md
upstream-paths:
  - security/apparmor/lsm.c (~2574 lines)
  - security/apparmor/include/label.h
  - security/apparmor/include/policy.h
  - security/apparmor/include/cred.h
  - security/apparmor/include/file.h
  - security/apparmor/include/path.h
  - security/apparmor/domain.c (change_hat / change_profile)
  - include/linux/lsm_hooks.h
-->

## Summary

AppArmor is a **path-name based** mandatory access control LSM derived from Immunix SubDomain. Where SELinux labels objects, AppArmor labels **subjects** (creds) with a `struct aa_label` — a reference-counted compound of one or more `struct aa_profile`s — and queries each profile's compiled **DFA** (deterministic finite automaton) against the runtime **path** of the object. There is no inode label; permission decisions are made by composing per-component DFA states across a resolved `struct path` plus a `path_cond { uid, mode }`. `lsm.c` registers `apparmor_hooks[]` covering cred / task / ptrace / capable / path-ops (link, unlink, symlink, mkdir, rmdir, mknod, rename, chmod, chown, truncate) / file (open, permission, lock, mmap, mprotect, truncate, receive) / xattr-style procattr (`/proc/<pid>/attr/current`, `/exec`) / sk / unix / inet sockets / io_uring / userns_create / mount / bprm / task_kill. Per-`apparmor_bprm_creds_for_exec`: walks the profile's `x_table` to resolve the exec transition (`profile_transition`), supporting `px` / `cx` / `ix` / `ux` / `pix` / `cix` modifiers. Per-`change_hat`: enters a same-profile "hat" subprofile keyed by a 64-bit `token` (revert-on-mismatch). Per-`change_profile`: switches to a sibling/stacked label. Per-`aa_label.flags & FLAG_UNCONFINED`: bypass fastpath. Critical for: distribution-default confinement (Ubuntu, openSUSE), path-policy enforcement, container default-deny, snap/Docker integration.

This Tier-3 covers `security/apparmor/lsm.c` (~2574 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct aa_label` | per-cred compound label (profiles + flags + secid + mediates-bitmap) | `AaLabel` |
| `struct aa_profile` | per-profile (name, file/cap/rlimits/net DFAs, x_table) | `AaProfile` |
| `struct aa_task_ctx` | per-task hat/onexec context | `AaTaskCtx` |
| `struct aa_file_ctx` | per-file cached label + allow-mask | `AaFileCtx` |
| `struct aa_sk_ctx` | per-sock cached label + peer | `AaSkCtx` |
| `apparmor_blob_sizes` | LSM blob sizing (cred=ptr, file=ctx, task=ctx, sock=ctx) | `Apparmor::BLOB_SIZES` |
| `apparmor_hooks[]` | per-LSM hook table | `Apparmor::HOOKS` |
| `apparmor_init()` | per-init (dfa engine, root ns, sysctl, buffers, init_ctx) | `Apparmor::init` |
| `apparmor_cred_prepare()` | per-cred clone | `Apparmor::cred_prepare` |
| `apparmor_cred_free()` | per-cred drop | `Apparmor::cred_free` |
| `apparmor_task_alloc()` | per-fork ctx | `Apparmor::task_alloc` |
| `apparmor_task_free()` | per-exit ctx drop | `Apparmor::task_free` |
| `apparmor_ptrace_access_check()` | per-ptrace gate | `Apparmor::ptrace_access_check` |
| `apparmor_capable()` | per-capability check | `Apparmor::capable` |
| `apparmor_file_open()` | per-open path check | `Apparmor::file_open` |
| `apparmor_file_permission()` | per-revalidate (rare path) | `Apparmor::file_permission` |
| `apparmor_mmap_file()` / `apparmor_file_mprotect()` | per-mmap / mprotect path check | `Apparmor::mmap_*` |
| `apparmor_bprm_creds_for_exec()` | per-exec profile transition | `Apparmor::bprm_creds_for_exec` |
| `apparmor_bprm_committing_creds()` | per-pre-commit inherit-files / rlimit-transition | `Apparmor::bprm_committing_creds` |
| `apparmor_bprm_committed_creds()` | per-post-commit ctx cleanup | `Apparmor::bprm_committed_creds` |
| `apparmor_task_kill()` | per-signal gate | `Apparmor::task_kill` |
| `apparmor_socket_create()` / `_post_create()` / `_bind()` / `_connect()` / `_sendmsg()` | per-socket-op gates | `Apparmor::socket_*` |
| `apparmor_sk_alloc_security()` | per-sock blob init | `Apparmor::sk_alloc_security` |
| `apparmor_userns_create()` | per-userns(2) gate | `Apparmor::userns_create` |
| `apparmor_setprocattr()` / `apparmor_setselfattr()` | per-`/proc/<pid>/attr/{current,exec}` writer | `Apparmor::set_proc_attr` |
| `aa_change_hat()` | per-hat enter/leave with token | `Apparmor::change_hat` |
| `aa_change_profile()` | per-profile switch / stack | `Apparmor::change_profile` |
| `aa_path_perm()` | per-path DFA lookup | `Aa::path_perm` |
| `aa_file_perm()` | per-file DFA lookup | `Aa::file_perm` |
| `aa_may_ptrace()` | per-ptrace policy | `Aa::may_ptrace` |
| `aa_may_signal()` | per-signal policy | `Aa::may_signal` |
| `aa_capable()` | per-capability policy | `Aa::capable` |
| `cred_label()` / `set_cred_label()` | per-cred label accessor | `Aa::cred_label` |
| `begin_current_label_crit_section()` / `end_current_label_crit_section()` | per-RCU-label critical section | `Aa::label_crit_section` |
| `unconfined(label)` | per-fastpath unconfined check | `Aa::unconfined` |

## Compatibility contract

REQ-1: struct aa_label:
- count: refcount (struct aa_common_ref).
- node: rb_node in `aa_labelset` rbtree.
- rcu: rcu_head for deferred free.
- proxy: aa_proxy * (replaces label-pointer when relabeled).
- hname: text representation (lazy).
- flags: FLAG_PROFILE | FLAG_UNCONFINED | FLAG_STALE | FLAG_INVALID | FLAG_HAT | FLAG_NS_COUNT | FLAG_RENAMED | FLAG_REVOKED | …
- secid: u32 mapped via `aa_secid_*`.
- size: number of profile entries in vec.
- mediates: bitmask of `AA_CLASS_*` this label mediates (file, cap, net, ns, …).
- vec[size]: flex-array of aa_profile * (compound label).
- /* When FLAG_PROFILE: profile[0]=self, profile[1]=poison; rules[]=per-ruleset */.

REQ-2: struct aa_profile (partial; full in `policy.h`):
- base: aa_policy { name, hname }.
- ns: aa_ns *.
- parent / file / xmatch / xattr_count / xattrs.
- x_table[]: exec-transition table (resolved names per `px:`, `cx:`, `Px`, `Cx`, etc.).
- rules: aa_ruleset[ ] — file DFA, cap mask, rlimits, network DFA, mount DFA.
- learning_cache, replaced_by.

REQ-3: struct aa_task_ctx:
- nnp: cached NNP label (snapshot at last exec).
- onexec: aa_label * (pending change_profile onexec target).
- previous: aa_label * (saved for change_hat revert).
- token: u64 (change_hat token; revert requires match).

REQ-4: struct aa_file_ctx:
- lock: spinlock.
- label: aa_label *__rcu (cached at open-time).
- allow: u32 (mask of perms granted at open).

REQ-5: struct aa_sk_ctx:
- label: aa_label *__rcu.
- peer: aa_label *__rcu (NEW connect-time peer).
- … (af_unix peers, etc.)

REQ-6: apparmor_blob_sizes:
- lbs_cred = sizeof(struct aa_label *).  /* cred blob is a POINTER to aa_label */
- lbs_file = sizeof(struct aa_file_ctx).
- lbs_task = sizeof(struct aa_task_ctx).
- lbs_sock = sizeof(struct aa_sk_ctx).

REQ-7: apparmor_cred_alloc_blank(cred, gfp):
- set_cred_label(cred, NULL).
- return 0.

REQ-8: apparmor_cred_prepare(new, old, gfp):
- set_cred_label(new, aa_get_newest_label(cred_label(old))).
- /* aa_get_newest_label follows .proxy chain to dodge stale label after policy replacement */
- return 0.

REQ-9: apparmor_cred_transfer(new, old):
- set_cred_label(new, aa_get_newest_label(cred_label(old))).

REQ-10: apparmor_cred_free(cred):
- aa_put_label(cred_label(cred)).
- set_cred_label(cred, NULL).

REQ-11: apparmor_task_alloc(task, clone_flags):
- aa_dup_task_ctx(task_ctx(task), task_ctx(current)).
- return 0.

REQ-12: apparmor_task_free(task):
- aa_free_task_ctx(task_ctx(task)).

REQ-13: apparmor_ptrace_access_check(child, mode):
- cred = get_task_cred(child); tracee = cred_label(cred).
- tracer = __begin_current_label_crit_section(&needput).
- mode_bits = (mode & PTRACE_MODE_READ) ? AA_PTRACE_READ : AA_PTRACE_TRACE.
- err = aa_may_ptrace(current_cred(), tracer, cred, tracee, mode_bits).
- __end_current_label_crit_section(tracer, needput).
- put_cred(cred); return err.

REQ-14: apparmor_ptrace_traceme(parent): symmetric.

REQ-15: apparmor_capable(cred, ns, cap, opts):
- label = aa_get_newest_cred_label(cred).
- if !unconfined(label): err = aa_capable(cred, label, cap, opts).
- /* Per-profile cap mask: profile->rules[].caps.allow */
- aa_put_label(label); return err.

REQ-16: apparmor_bprm_creds_for_exec(bprm):
- /* Compute new label from x_table and onexec hint */
- label = aa_get_newest_cred_label(bprm->cred).
- new_label = aa_bprm_exec_profile(label, bprm).
- /* x_table resolution: px/cx/ix/ux + modifiers */
- /* Per-NNP: bounded transition or denial */
- if bprm->unsafe & LSM_UNSAFE_NO_NEW_PRIVS:
  - if !aa_label_is_unconfined_subset(new_label, label): return -EPERM.
- set_cred_label(bprm->cred, new_label).

REQ-17: apparmor_bprm_committing_creds(bprm):
- label = aa_current_raw_label(); new_label = cred_label(bprm->cred).
- if new_label->proxy == label->proxy ∨ unconfined(new_label): return.
- aa_inherit_files(bprm->cred, current->files).
- current->pdeath_signal = 0.
- __aa_transition_rlimits(label, new_label).

REQ-18: apparmor_bprm_committed_creds(bprm):
- aa_clear_task_ctx_trans(task_ctx(current)).
- /* Drop onexec, previous, token */

REQ-19: apparmor_file_open(file):
- fctx = file_ctx(file).
- if !path_mediated_fs(file->f_path.dentry): return 0.
- /* Exec-time skip — bprm hook already covered */
- if file->f_flags & __FMODE_EXEC:
  - fctx->allow = MAY_EXEC | MAY_READ | AA_EXEC_MMAP.
  - return 0.
- label = aa_get_newest_cred_label_condref(file->f_cred, &needput).
- if !unconfined(label):
  - cond = { uid = vfsuid_into_kuid(...), mode = inode->i_mode }.
  - err = aa_path_perm(OP_OPEN, file->f_cred, label, &file->f_path, 0, aa_map_file_to_perms(file), &cond).
  - fctx->allow = aa_map_file_to_perms(file).
- aa_put_label_condref(label, needput).
- return err.

REQ-20: apparmor_file_permission(file, mask):
- common_file_perm(OP_FPERM, file, mask).
- /* Uses cached fctx->label; revalidates only if label changed via .proxy */

REQ-21: apparmor_mmap_file(file, reqprot, prot, flags):
- common_mmap(OP_FMMAP, file, prot, flags).
- if !file: return 0.
- mask = (prot & PROT_READ ? MAY_READ : 0) | (prot & PROT_WRITE ? MAY_WRITE : 0 — only for MAP_SHARED) | (prot & PROT_EXEC ? AA_EXEC_MMAP : 0).
- common_file_perm(OP_FMMAP, file, mask).

REQ-22: apparmor_file_mprotect(vma, reqprot, prot):
- /* Enforce on PROT_EXEC additions */.

REQ-23: apparmor_task_kill(target, info, sig, cred):
- tc = get_task_cred(target); tl = aa_get_newest_cred_label(tc).
- if cred: cl = aa_get_newest_cred_label(cred); aa_may_signal(cred, cl, tc, tl, sig); aa_put_label(cl).
- else: cl = __begin_current_label_crit_section(&needput); aa_may_signal(current_cred(), cl, tc, tl, sig).
- aa_put_label(tl); put_cred(tc).

REQ-24: apparmor_socket_create(family, type, protocol, kern):
- if kern: return 0.
- label = begin_current_label_crit_section().
- if !unconfined(label):
  - if family == PF_UNIX: aa_unix_create_perm(label, family, type, protocol).
  - else: aa_af_perm(current_cred(), label, OP_CREATE, AA_MAY_CREATE, family, type, protocol).

REQ-25: apparmor_socket_post_create(sock, family, type, protocol, kern):
- label = kern ? aa_get_label(kernel_t) : aa_get_current_label().
- if sock->sk:
  - ctx = aa_sock(sock->sk).
  - rcu_assign_pointer(ctx->label, aa_get_label(label)).

REQ-26: apparmor_sk_alloc_security(sk, family, gfp):
- ctx = aa_sock(sk); spin_lock_init / RCU init.
- rcu_assign_pointer(ctx->label, NULL) — filled by post_create.

REQ-27: apparmor_sk_clone_security(sk, newsk):
- newctx.label = aa_get_label(rcu_dereference(ctx.label)).
- newctx.peer = aa_get_label(rcu_dereference(ctx.peer)).

REQ-28: apparmor_setprocattr / setselfattr — do_setattr(attr, value, size):
- /* LSM_ATTR_CURRENT */
  - "changehat" args → aa_setprocattr_changehat(args, size, AA_CHANGE_NOFLAGS).
  - "permhat"   args → aa_setprocattr_changehat(args, size, AA_CHANGE_TEST).
  - "changeprofile" args → aa_change_profile(args, AA_CHANGE_NOFLAGS).
  - "permprofile" args → aa_change_profile(args, AA_CHANGE_TEST).
  - "stack"      args → aa_change_profile(args, AA_CHANGE_STACK).
- /* LSM_ATTR_EXEC */
  - "exec"  args → aa_change_profile(args, AA_CHANGE_ONEXEC).
  - "stack" args → aa_change_profile(args, AA_CHANGE_ONEXEC | AA_CHANGE_STACK).
- else: audit AUDIT_APPARMOR_DENIED; return -EINVAL.

REQ-29: aa_change_hat(hats, count, token, flags) (security/apparmor/domain.c):
- Constraints: same-namespace; same profile (subprofile-only); token must match on revert.
- Build new label by `build_change_hat`.
- /* Permissions */: profile must mediate AA_CLASS_HAT and allow change_hat.
- Replace current cred's label via prepare_creds / commit_creds.

REQ-30: aa_change_profile(fqname, flags) (security/apparmor/domain.c):
- Constraints: profile must permit `change_profile` (or `stack`).
- Resolve `fqname` → target aa_label via `aa_label_parse`.
- If AA_CHANGE_ONEXEC: store in task_ctx->onexec (no immediate commit); bprm_creds_for_exec consumes it.
- Else: prepare_creds; set new label; commit_creds.
- AA_CHANGE_TEST: dry-run (perm-check only).
- AA_CHANGE_STACK: compose current + target via `aa_label_merge`.

REQ-31: unconfined(label):
- (label->flags & FLAG_UNCONFINED).
- Used as a hot-path predicate to skip DFA lookup when policy is "unconfined".

REQ-32: apparmor_init():
- aa_setup_dfa_engine() — register DFA matcher.
- aa_alloc_root_ns() — root namespace + unconfined profile.
- apparmor_init_sysctl() — register `kernel.apparmor_restrict_unprivileged_*`.
- alloc_buffers() — per-CPU reusable path buffers (`aa_local_cache`).
- set_init_ctx() — set initial-task label to unconfined.
- security_add_hooks(apparmor_hooks, ARRAY_SIZE, &apparmor_lsmid).
- audit_cfg_lsm(&apparmor_lsmid, AUDIT_CFG_LSM_SECCTX_SUBJECT).
- apparmor_initialized = 1.

REQ-33: DEFINE_LSM(apparmor):
- flags = LSM_FLAG_LEGACY_MAJOR | LSM_FLAG_EXCLUSIVE.
- enabled = &apparmor_enabled.
- blobs = &apparmor_blob_sizes.
- init = apparmor_init.
- initcall_fs = aa_create_aafs.
- initcall_device = apparmor_nf_ip_init (if NETFILTER+SECMARK).
- initcall_late = init_profile_hash (if APPARMOR_HASH).

REQ-34: per-CPU path-buffer cache (`aa_local_cache` + `aa_global_buffers`):
- Reserves `RESERVE_COUNT=2` buffers per-CPU to avoid GFP_ATOMIC in path-derivation.
- `buffer_count` tracks global pool size.

## Acceptance Criteria

- [ ] AC-1: `apparmor_init` registers all `apparmor_hooks[]` and sets `apparmor_initialized = 1`.
- [ ] AC-2: `apparmor_cred_prepare` copies the label pointer via `aa_get_newest_label` (follows `.proxy` after policy replacement).
- [ ] AC-3: `apparmor_file_open` skips DFA lookup when `unconfined(label)` is true and caches `fctx->allow`.
- [ ] AC-4: `apparmor_file_open` with `__FMODE_EXEC` short-circuits — exec is already mediated by bprm hook.
- [ ] AC-5: `apparmor_bprm_creds_for_exec` resolves transition through profile `x_table`; NNP rejects unbounded transition.
- [ ] AC-6: `apparmor_bprm_committing_creds` inherits files (`aa_inherit_files`) and transitions rlimits when proxies differ and not unconfined.
- [ ] AC-7: `apparmor_capable` calls `aa_capable` only when not unconfined.
- [ ] AC-8: `apparmor_ptrace_access_check` calls `aa_may_ptrace` with `AA_PTRACE_READ` or `AA_PTRACE_TRACE` per `PTRACE_MODE_READ`.
- [ ] AC-9: `apparmor_task_kill` calls `aa_may_signal` with both subject and target labels; cred argument overrides current when set.
- [ ] AC-10: `apparmor_socket_create(family=PF_UNIX, …)` routes through `aa_unix_create_perm`; other families via `aa_af_perm`.
- [ ] AC-11: `apparmor_socket_post_create` stamps `sock->sk`'s `aa_sk_ctx.label` to `kernel_t` when `kern==1`, else current label.
- [ ] AC-12: `apparmor_setprocattr("current", "changehat <token> <hat>")` invokes `aa_setprocattr_changehat` with `AA_CHANGE_NOFLAGS`.
- [ ] AC-13: `apparmor_setprocattr("exec", "exec <fqname>")` invokes `aa_change_profile` with `AA_CHANGE_ONEXEC`.
- [ ] AC-14: `aa_change_hat(hats, count, token, flags)` rejects when current profile does not mediate `AA_CLASS_HAT`.
- [ ] AC-15: `aa_change_profile(fqname, AA_CHANGE_STACK)` produces a label whose `size` is the merge of current + target profile counts.

## Architecture

```
struct AaLabel {
  count: AaCommonRef,
  node: RbNode,
  rcu: RcuHead,
  proxy: *AaProxy,
  hname: *u8,                       // lazy text form
  flags: i64,                       // FLAG_PROFILE | FLAG_UNCONFINED | FLAG_STALE | …
  secid: u32,
  size: i32,
  mediates: u64,                    // bit per AA_CLASS_*
  vec: FlexArray<*AaProfile>,       // when !FLAG_PROFILE
  // OR (when FLAG_PROFILE):
  //   profile: [*AaProfile; 2]     // profile[0]=self, profile[1]=poison
  //   rules:   FlexArray<*AaRuleset>
}

struct AaTaskCtx {
  nnp: Option<*AaLabel>,            // cached NNP snapshot
  onexec: Option<*AaLabel>,         // change_profile-onexec pending
  previous: Option<*AaLabel>,       // change_hat revert target
  token: u64,                       // change_hat token
}

struct AaFileCtx {
  lock: SpinLock,
  label: RcuPtr<AaLabel>,
  allow: u32,                       // perms granted at open
}

struct AaSkCtx {
  label: RcuPtr<AaLabel>,
  peer: RcuPtr<AaLabel>,
  // af_unix peers, etc.
}

struct ApparmorBlobSizes {           // lsm_blob_sizes
  lbs_cred: usize = sizeof(*AaLabel),
  lbs_file: usize = sizeof(AaFileCtx),
  lbs_task: usize = sizeof(AaTaskCtx),
  lbs_sock: usize = sizeof(AaSkCtx),
}
```

`Apparmor::file_open(file) -> Result<()>`:
1. fctx = file_ctx(file).
2. if !path_mediated_fs(file.f_path.dentry): return Ok(()).
3. if file.f_flags & __FMODE_EXEC:
   - fctx.allow = MAY_EXEC | MAY_READ | AA_EXEC_MMAP.
   - return Ok(()).
4. label = aa_get_newest_cred_label_condref(file.f_cred, &needput).
5. if !unconfined(label):
   - cond = PathCond { mode: inode.i_mode, uid: vfsuid_into_kuid(file_mnt_idmap(file), inode) }.
   - aa_path_perm(OP_OPEN, file.f_cred, label, &file.f_path, 0, aa_map_file_to_perms(file), &cond)?.
   - fctx.allow = aa_map_file_to_perms(file).
6. aa_put_label_condref(label, needput).
7. Ok(()).

`Apparmor::bprm_creds_for_exec(bprm) -> Result<()>`:
1. old_label = aa_get_newest_cred_label(bprm.cred).
2. profile = aa_label_resolve(old_label, bprm.filename, bprm.file).
3. /* x_table walk: px / cx / ix / ux + modifiers */
4. new_label = aa_x_table_resolve(profile, bprm).
5. if bprm.unsafe & LSM_UNSAFE_NO_NEW_PRIVS:
   - if !aa_label_is_unconfined_subset(new_label, old_label): return Err(-EPERM).
6. set_cred_label(bprm.cred, new_label).
7. Ok(()).

`Apparmor::bprm_committing_creds(bprm)`:
1. cur = aa_current_raw_label().
2. new = cred_label(bprm.cred).
3. if new.proxy == cur.proxy ∨ unconfined(new): return.
4. aa_inherit_files(bprm.cred, current.files).
5. current.pdeath_signal = 0.
6. __aa_transition_rlimits(cur, new).

`Apparmor::bprm_committed_creds(bprm)`:
1. aa_clear_task_ctx_trans(task_ctx(current)).
2. /* Drops onexec / previous / token */.

`Apparmor::capable(cred, ns, cap, opts) -> Result<()>`:
1. label = aa_get_newest_cred_label(cred).
2. err = if !unconfined(label) { aa_capable(cred, label, cap, opts) } else { Ok(()) }.
3. aa_put_label(label).
4. err.

`Apparmor::ptrace_access_check(child, mode) -> Result<()>`:
1. cred = get_task_cred(child); tracee = cred_label(cred).
2. tracer = __begin_current_label_crit_section(&needput).
3. mode_bits = if mode & PTRACE_MODE_READ { AA_PTRACE_READ } else { AA_PTRACE_TRACE }.
4. err = aa_may_ptrace(current_cred(), tracer, cred, tracee, mode_bits).
5. __end_current_label_crit_section(tracer, needput); put_cred(cred).
6. err.

`Apparmor::task_kill(target, info, sig, cred) -> Result<()>`:
1. tc = get_task_cred(target); tl = aa_get_newest_cred_label(tc).
2. err = match cred {
   - Some(cred) => { cl = aa_get_newest_cred_label(cred); r = aa_may_signal(cred, cl, tc, tl, sig); aa_put_label(cl); r },
   - None => { cl = __begin_current_label_crit_section(&needput); r = aa_may_signal(current_cred(), cl, tc, tl, sig); __end_current_label_crit_section(cl, needput); r },
   }.
3. aa_put_label(tl); put_cred(tc).
4. err.

`Apparmor::socket_create(family, type, protocol, kern) -> Result<()>`:
1. if kern: return Ok(()).
2. label = begin_current_label_crit_section().
3. err = if !unconfined(label) {
   - if family == PF_UNIX { aa_unix_create_perm(label, family, type, protocol) }
   - else { aa_af_perm(current_cred(), label, OP_CREATE, AA_MAY_CREATE, family, type, protocol) }
   } else { Ok(()) }.
4. end_current_label_crit_section(label).
5. err.

`Apparmor::sk_alloc_security(sk, family, gfp) -> Result<()>`:
1. ctx = aa_sock(sk).
2. spin_lock_init(&ctx.lock); rcu_assign_pointer(ctx.label, ptr::null_mut()); rcu_assign_pointer(ctx.peer, ptr::null_mut()).
3. Ok(()).
   /* post_create fills ctx.label = current label or kernel_t */

`Apparmor::set_proc_attr(name, value, size) -> Result<usize>`:
1. command = strsep(&args, " ").
2. match (attr, command):
   - (LSM_ATTR_CURRENT, "changehat") => aa_setprocattr_changehat(args, arg_size, AA_CHANGE_NOFLAGS),
   - (LSM_ATTR_CURRENT, "permhat") => aa_setprocattr_changehat(args, arg_size, AA_CHANGE_TEST),
   - (LSM_ATTR_CURRENT, "changeprofile") => aa_change_profile(args, AA_CHANGE_NOFLAGS),
   - (LSM_ATTR_CURRENT, "permprofile") => aa_change_profile(args, AA_CHANGE_TEST),
   - (LSM_ATTR_CURRENT, "stack") => aa_change_profile(args, AA_CHANGE_STACK),
   - (LSM_ATTR_EXEC, "exec") => aa_change_profile(args, AA_CHANGE_ONEXEC),
   - (LSM_ATTR_EXEC, "stack") => aa_change_profile(args, AA_CHANGE_ONEXEC | AA_CHANGE_STACK),
   - _ => fail-and-audit.

`Aa::change_hat(hats, count, token, flags) -> Result<()>`:
1. /* security/apparmor/domain.c:aa_change_hat */.
2. label = begin_current_label_crit_section(); profile = label_profile(label).
3. if unconfined(label): audit "unconfined can not change_hat"; return Err(-EPERM).
4. for each hat in hats: build candidate via `build_change_hat(subj_cred, profile, hat, token)`.
5. /* Token semantics */: nonzero token → save in task_ctx for revert; zero token + previous-set → revert.
6. /* Test flag */: AA_CHANGE_TEST → permission-check without commit.
7. else: prepare_creds; set_cred_label(new, new_label); commit_creds.

`Aa::change_profile(fqname, flags) -> Result<()>`:
1. /* security/apparmor/domain.c:aa_change_profile */.
2. target = aa_label_parse(label, fqname).
3. /* AV: profile must permit change_profile (or stack) to target */.
4. if AA_CHANGE_TEST: return permission-result only.
5. if AA_CHANGE_ONEXEC: store target in task_ctx.onexec; bprm_creds_for_exec consumes.
6. else if AA_CHANGE_STACK: new = aa_label_merge(current_label, target); commit.
7. else: prepare_creds; set_cred_label(new, target); commit_creds.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cred_label_ptr_within_bounds` | INVARIANT | per-cred_label: offset + sizeof(*aa_label) ≤ cred->security blob size. |
| `label_refcount_balanced_per_hook` | INVARIANT | per-hook get/put pairs match (incl. condref). |
| `unconfined_fastpath_skips_dfa` | INVARIANT | per-file_open / capable / ptrace: unconfined(label) ⟹ no DFA lookup. |
| `nnp_unbounded_transition_denied` | INVARIANT | per-bprm_creds_for_exec: NNP ∧ !subset ⟹ Err(-EPERM). |
| `file_open_exec_short_circuit` | INVARIANT | per-file_open: __FMODE_EXEC ⟹ fctx.allow set + return Ok before path_perm. |
| `sk_alloc_then_post_create_labels_sock` | INVARIANT | per-socket(2): sk_alloc_security ⟹ post_create assigns ctx.label. |
| `change_hat_token_revert_strict` | INVARIANT | per-aa_change_hat: revert requires matching saved token. |
| `path_mediated_fs_gates_path_hooks` | INVARIANT | per-common_perm / file_open: !path_mediated_fs ⟹ Ok early. |

### Layer 2: TLA+

`security/apparmor/lsm.tla`:
- Per-cred_prepare + per-bprm_creds + per-committing + per-committed + per-change_hat + per-change_profile + per-policy-replace.
- Properties:
  - `safety_label_proxy_redirects_after_replace` — per-policy replace: cred_prepare resolves to newest via .proxy.
  - `safety_no_change_hat_when_unconfined` — per-aa_change_hat: unconfined ⟹ Err.
  - `safety_nnp_unbounded_denied` — per-bprm_creds: NNP + unbounded ⟹ -EPERM.
  - `safety_change_profile_test_no_commit` — per-AA_CHANGE_TEST: cred not mutated.
  - `safety_onexec_consumed_by_bprm` — per-AA_CHANGE_ONEXEC: task_ctx.onexec drained by bprm_creds_for_exec.
  - `safety_kern_socket_labeled_kernel_t` — per-socket_post_create: kern==1 ⟹ ctx.label == kernel_t.
  - `liveness_per_file_open_terminates` — per-file_open: always returns.
  - `liveness_bprm_committing_terminates` — per-committing_creds: terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Apparmor::file_open` post: !unconfined(label) ⟹ aa_path_perm called with cond.{uid,mode} from inode | `Apparmor::file_open` |
| `Apparmor::bprm_creds_for_exec` post: NNP ⟹ aa_label_is_unconfined_subset(new, old) | `Apparmor::bprm_creds_for_exec` |
| `Apparmor::bprm_committing_creds` post: new.proxy != cur.proxy ∧ !unconfined(new) ⟹ aa_inherit_files called | `Apparmor::bprm_committing_creds` |
| `Apparmor::capable` post: unconfined ⟹ Ok without aa_capable | `Apparmor::capable` |
| `Apparmor::ptrace_access_check` post: AA_PTRACE_READ iff PTRACE_MODE_READ | `Apparmor::ptrace_access_check` |
| `Apparmor::task_kill` post: cred-arg present ⟹ aa_may_signal subject = cred-label | `Apparmor::task_kill` |
| `Apparmor::socket_create` post: PF_UNIX routed to aa_unix_create_perm | `Apparmor::socket_create` |
| `Apparmor::set_proc_attr` post: command → flags mapping bijective per Acceptance table | `Apparmor::set_proc_attr` |
| `Aa::change_hat` post: AA_CHANGE_TEST ⟹ cred unchanged | `Aa::change_hat` |
| `Aa::change_profile` post: AA_CHANGE_STACK ⟹ label.size == current.size + target.size | `Aa::change_profile` |

### Layer 4: Verus/Creusot functional

`Per-execve → bprm_creds_for_exec (x_table resolve + NNP-subset check) → bprm_committing_creds (inherit files, rlimits transition) → bprm_committed_creds → first-file_open under new label (DFA path-perm, cache in fctx.allow) → file_permission fastpath` semantic equivalence: per-`Documentation/admin-guide/LSM/apparmor.rst` + per-AppArmor reference parser (`apparmor_parser`) policy semantics.

## Hardening

(Inherits row-1 features from `security/00-overview.md` § Hardening.)

AppArmor-hooks reinforcement:

- **Per-cred-label pointer dereferenced via `aa_get_newest_label` (proxy chain)** — defense against per-stale-label use after policy replacement.
- **Per-cred-label refcount get/put paired in every hook** — defense against per-label UAF.
- **Per-`unconfined()` fastpath inverted into explicit policy decision** — defense against per-silent-bypass when label flags corrupted.
- **Per-NNP unbounded-transition denial** — defense against per-setuid privilege-escalation via exec.
- **Per-`path_mediated_fs` gate** — defense against per-pseudo-fs (procfs / sysfs / tmpfs) policy spam.
- **Per-`__FMODE_EXEC` short-circuit only after bprm gate** — defense against per-exec re-mediation skip.
- **Per-`AA_CHANGE_TEST` dry-run isolation** — defense against per-test-mode side-effect leakage.
- **Per-`AA_CHANGE_ONEXEC` consumed only by bprm_creds_for_exec** — defense against per-onexec dangling pending state.
- **Per-`change_hat` token-required revert** — defense against per-cross-hat escalation by guessing.
- **Per-per-CPU buffer reservation (`RESERVE_COUNT=2`)** — defense against per-GFP_ATOMIC in path-derivation hot path.
- **Per-`apparmor_initialized` set after `security_add_hooks`** — defense against per-pre-init hook entry returning ambiguous result.
- **Per-`kernel_t` label on `kern==1` sockets** — defense against per-userspace-label leak into kernel sockets.
- **Per-`aa_inherit_files` on transitioning exec** — defense against per-fd-handle-bypass after profile change.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — apparmorfs `replace_profiles`, `remove_profile`, `load_profile` payloads copy-from-user bounded against `aa_replace_profiles` size cap; profile-name buffers `kstrndup_quotable` length-checked before allocation.
- **PAX_KERNEXEC** — `aa_dfa_match`, `unpack_*`, and DFA walker reside in RX `.text`; `apparmor_hooks[]` registered through `security_add_hooks` into kCFI-signed dispatch.
- **PAX_RANDKSTACK** — kstack-offset randomization on every LSM hook entry (`apparmor_bprm_creds_for_exec`, `apparmor_file_open`, `apparmor_socket_*`, mount hooks).
- **PAX_REFCOUNT** — saturating refcount on `struct aa_label`, `aa_profile`, `aa_ns`, `aa_loaddata`; defense against attacker-driven profile thrash wrapping the kref.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `aa_label`, profile policy DFA buffers, and `aa_audit_data` containing path-derivation buffers (per-CPU `aa_buffers` scrubbed before reuse).
- **PAX_UDEREF (SMAP/SMEP)** — ASM_CLAC on apparmorfs write paths; policy-binary parser (`unpack_blob`, `unpack_array`, `unpack_u32`) bounds-checks before deref.
- **PAX_RAP / kCFI** — `apparmor_hooks[]` LSM dispatch and `aa_dfa_ops` (next/perm table indirection) verified through kCFI on every transition.
- **GRKERNSEC_HIDESYM** — `apparmor_hook_heads`, `root_ns`, `aa_dfa_*` symbols stripped from /proc/kallsyms for non-CAP_SYSLOG.
- **GRKERNSEC_DMESG** — policy-load / DFA-corruption / audit-rule-parse diagnostics restricted to CAP_SYSLOG.
- **CAP_MAC_ADMIN strict** — `replace_profiles` / `remove_profile` / namespace `.load` writes gated on CAP_MAC_ADMIN in the policy-owning user-ns; no fallthrough to CAP_SYS_ADMIN.
- **Policy-binary magic + version pinned** — `aa_unpack_header` rejects mismatched `AA_PROFILE_MAGIC` and policy versions outside `[v5, v8]`; unknown opcodes refused, not skipped.
- **LSM audit-rule strict-type-check** — `aa_audit_rule_init` validates audit field type against AUDIT_SUBJ_ROLE/USER/TYPE/SEN/CLR canonical set; rejects mixed-LSM audit predicates.

Per-doc rationale: apparmor_lsm is the policy decision point for the unified label model — every exec, file_open, ptrace_check, and socket hook passes through here. A PAX_REFCOUNT wrap on `aa_label` or a USERCOPY miss on `replace_profiles` would let an unprivileged userspace replace its own profile with `unconfined` and escape MAC. The CAP_MAC_ADMIN strict gate + DFA RX-only + LSM kCFI overlap is what enforces the AppArmor invariant.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `security/apparmor/domain.c` profile transition resolution (covered in `domain.md` Tier-3 if expanded)
- `security/apparmor/policy.c` profile / namespace lifecycle (covered in `policy.md` Tier-3 if expanded)
- `security/apparmor/policy_unpack.c` binary policy loader (covered in `policy-unpack.md` Tier-3 if expanded)
- `security/apparmor/apparmorfs.c` `/sys/kernel/security/apparmor` (covered in `apparmorfs.md` Tier-3 if expanded)
- `security/apparmor/match.c` DFA matcher (covered in `match.md` Tier-3 if expanded)
- `security/apparmor/af_unix.c`, `net.c` socket mediation internals (covered separately if expanded)
- `security/apparmor/mount.c` mount mediation internals (covered separately if expanded)
- Userspace `apparmor_parser` policy compiler (out of kernel)
- Implementation code
