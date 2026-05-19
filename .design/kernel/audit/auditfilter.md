# Tier-3: kernel/auditfilter.c — Audit rule filter engine

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/audit/00-overview.md
upstream-paths:
  - kernel/auditfilter.c (~1463 lines)
  - kernel/audit.h
  - include/linux/audit.h
  - include/uapi/linux/audit.h
-->

## Summary

The audit filter engine is the kernel-side compiler + matcher for `struct audit_rule_data` netlink messages from `auditctl`. Eight per-list filter chains in `audit_filter_list[AUDIT_NR_FILTERS]` (TASK / EXIT / USER / EXCLUDE / TYPE / FS / URING_EXIT / etc.) hold RCU-traversable `audit_entry` records; a parallel `audit_rules_list[]` (mutex-protected) holds the same rules for write-side blocking traversals (e.g. AUDIT_LIST_RULES dump). Per-`audit_data_to_entry` parses a packed `audit_rule_data` (op codes via `audit_to_op` mapping `Audit_equal / _not_equal / _lt / _le / _gt / _ge / _bitmask / _bittest` ↔ wire `AUDIT_EQUAL` etc.; field types `AUDIT_PID / _UID / _GID / _MSGTYPE / _SUBJ_* / _OBJ_* / _WATCH / _DIR / _INODE / _FILTERKEY / _ARCH / _EXE / _FIELD_COMPARE / _ARG[0-3] / _PERM / _FILETYPE / _LOGINUID(_SET) / _SADDR_FAM / _SESSIONID / _FSTYPE / _DEVMINOR / _DEVMAJOR`) into a `struct audit_entry { struct list_head list, list_head rcu, struct audit_krule rule }` via `audit_init_entry` + per-field `audit_field_valid` + `audit_unpack_string` for variable-length string payloads + LSM-rule init via `security_audit_rule_init`. Per-`audit_register_class` populates the `classes[AUDIT_SYSCALL_CLASSES]` bitmap that expands compact AUDIT_SYSCALL_CLASSES bits (read / write / chattr / signal etc.) into per-syscall-number bitmasks. Per-`audit_rule_change` is the AUDIT_ADD_RULE / AUDIT_DEL_RULE netlink dispatch; `audit_list_rules_send` handles AUDIT_LIST_RULES with a `kthread_run(audit_send_list_thread)` background flush to avoid deadlocking against an auditctl-buffer-full socket. Per-`audit_filter(msgtype, listtype)` is the runtime RCU-side matcher used by USER / EXCLUDE / TYPE / FS lists. Critical for: PCI-DSS / FISMA / STIG ruleset compliance, per-arch / per-syscall whitelisting, LSM-policy-reload (`audit_update_lsm_rules`) survival, byte-equivalent `auditctl -l` dump-then-reload.

This Tier-3 covers `kernel/auditfilter.c` (~1463 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `audit_filter_list[AUDIT_NR_FILTERS]` | per-list RCU-traversable filter chain (NR_FILTERS=8) | `AUDIT_FILTER_LIST` |
| `audit_rules_list[AUDIT_NR_FILTERS]` | per-list mutex-protected write-side mirror | `AUDIT_RULES_LIST` |
| `audit_filter_mutex` | per-write-side serialisation | `AUDIT_FILTER_MUTEX` |
| `struct audit_rule_data` (uapi) | per-wire ABI | shared `AuditRuleData` |
| `struct audit_entry` | per-list-element wrapping krule + RCU | `AuditEntry` |
| `struct audit_krule` | per-kernel rule (flags/pflags/listnr/action/field_count/fields/mask/prio/buflen/inode_f/watch/tree/exe/filterkey/rlist/list) | `AuditKrule` |
| `struct audit_field` | per-field (type/val/op/uid/gid/lsm_str/lsm_rule) | `AuditField` |
| `audit_init_entry()` | per-allocate audit_entry + audit_field[] | `AuditEntry::init` |
| `audit_free_rule()` / `audit_free_rule_rcu()` | per-rcu-free path | `AuditEntry::free` / `free_rcu` |
| `audit_free_lsm_field()` | per-LSM field teardown | `AuditField::free_lsm` |
| `audit_unpack_string()` | per-bounded string copy from netlink buf | `AuditEntry::unpack_string` |
| `audit_to_inode()` | per-AUDIT_INODE field check | `AuditKrule::to_inode` |
| `audit_register_class()` (__init) | per-AUDIT_CLASS_* bitmap install | `AuditClass::register` |
| `audit_match_class()` | per-syscall class lookup | `AuditClass::match` |
| `audit_match_class_bits()` (CONFIG_AUDITSYSCALL) | per-mask vs class-bits | `AuditClass::match_bits` |
| `audit_match_signal()` (CONFIG_AUDITSYSCALL) | per-signal-class match | `AuditClass::match_signal` |
| `audit_to_entry_common()` | per-listnr + action + field_count + mask + class-expansion | `AuditEntry::to_entry_common` |
| `audit_ops[]` | per-Audit_op ↔ AUDIT_*_MASK mapping | `AUDIT_OPS` |
| `audit_to_op()` | per-wire op → Audit_op enum | `AuditOp::from_wire` |
| `audit_field_valid()` | per-field type + op + value validation | `AuditField::is_valid` |
| `audit_data_to_entry()` | per-userspace parse | `AuditEntry::from_data` |
| `audit_pack_string()` / `audit_krule_to_data()` | per-kernel-to-wire | `AuditEntry::to_data` |
| `audit_compare_rule()` | per-rule equivalence test | `AuditKrule::cmp` |
| `audit_dupe_lsm_field()` | per-LSM field deep-copy | `AuditField::dupe_lsm` |
| `audit_dupe_rule()` | per-deep-copy on watch update / LSM reload | `AuditEntry::dupe` |
| `audit_find_rule()` | per-listnr/inode-hash duplicate lookup | `AuditEntry::find` |
| `audit_add_rule()` | per-insert (incl. watch/tree side-tables) | `AuditEntry::add` |
| `audit_del_rule()` | per-remove (incl. watch/tree side-tables) | `AuditEntry::del` |
| `audit_list_rules()` | per-AUDIT_LIST_RULES dump | `AuditEntry::list_all` |
| `audit_log_rule_change()` | per-AUDIT_CONFIG_CHANGE record | `AuditEntry::log_change` |
| `audit_rule_change()` | per-AUDIT_ADD_RULE/DEL_RULE netlink dispatch | `AuditEntry::change` |
| `audit_list_rules_send()` | per-AUDIT_LIST_RULES netlink dispatch | `AuditEntry::list_send` |
| `audit_comparator()` | per-u32 comparator | `AuditCmp::u32` |
| `audit_uid_comparator()` | per-kuid_t comparator | `AuditCmp::uid` |
| `audit_gid_comparator()` | per-kgid_t comparator | `AuditCmp::gid` |
| `parent_len()` | per-pathname split helper | `path_parent_len` |
| `audit_compare_dname_path()` | per-dentry-vs-path tail compare | `audit_compare_dname_path` |
| `audit_filter()` | per-runtime USER/EXCLUDE/TYPE/FS list match | `AuditFilter::filter` |
| `update_lsm_rule()` | per-rule LSM re-init | `AuditEntry::update_lsm` |
| `audit_update_lsm_rules()` | per-LSM-policy-reload sweep | `AuditFilter::update_lsm_rules` |
| `prio_low` / `prio_high` | per-AUDIT_FILTER_PREPEND priority counters | shared |
| `audit_n_rules` / `audit_signals` | per-stat counters | shared |
| `audit_inode_hash[AUDIT_INODE_BUCKETS]` | per-inode hash buckets (extern) | shared |

## Compatibility contract

REQ-1: Eight per-list filter chains preserved (compile-time assert `AUDIT_NR_FILTERS == 8`):
- audit_filter_list[0..7]: list_head, RCU-traversable, modified under audit_filter_mutex.
- audit_rules_list[0..7]: list_head, mutex-only (write-side traversal for AUDIT_LIST_RULES).

REQ-2: audit_filter_mutex semantics:
- DEFINE_MUTEX(audit_filter_mutex).
- Held during: audit_add_rule / del_rule / list_rules / update_lsm_rules / audit_add_watch (dropped+reacquired around path lookup) / audit_update_watch / audit_remove_parent_watches.
- Reader-side traversal uses `list_for_each_entry_rcu` and `rcu_read_lock`; writers `list_replace_rcu` and `call_rcu(.., audit_free_rule_rcu)`.

REQ-3: struct audit_entry preserved:
- list: per-RCU-list link into audit_filter_list[].
- rcu: per-call_rcu head.
- rule: embedded struct audit_krule.

REQ-4: struct audit_krule preserved:
- flags: per-AUDIT_FILTER_PREPEND bit (post-translation stripped).
- pflags: per-private bits (AUDIT_LOGINUID_LEGACY).
- listnr: per-AUDIT_FILTER_TASK / _EXIT / _USER / _EXCLUDE / _TYPE / _FS / _URING_EXIT.
- action: per-AUDIT_NEVER / _ALWAYS (AUDIT_POSSIBLE deprecated → EINVAL).
- field_count: per-≤ AUDIT_MAX_FIELDS.
- mask[AUDIT_BITMASK_SIZE]: per-syscall bitmask.
- fields: per-pointer to audit_field[field_count] kzalloc'd block.
- buflen: per-running total of variable-length string field bytes (for dump-pack size).
- prio: per-rule priority (++prio_high for PREPEND, --prio_low otherwise, ~0ULL for non-EXIT lists).
- inode_f: per-pointer-to-AUDIT_INODE-field (for hash dispatch).
- watch / tree / exe: per-side-table pointer.
- filterkey: per-kstrdup'd key (kfree at audit_free_rule).
- arch_f: per-pointer to AUDIT_ARCH field.
- list (struct list_head): per-link into audit_rules_list[].
- rlist (struct list_head): per-link into watch->rules / tree->rules.

REQ-5: struct audit_field preserved:
- type: per-AUDIT_PID / _UID / _GID / _MSGTYPE / _SUBJ_USER / ...
- val: per-u32 (or u32 holding f_val for length of string payload).
- op: per-Audit_equal / _not_equal / _lt / _le / _gt / _ge / _bitmask / _bittest.
- uid: per-kuid_t.
- gid: per-kgid_t.
- lsm_str: per-kstrdup'd LSM string.
- lsm_rule: per-opaque LSM-rule pointer (from security_audit_rule_init).

REQ-6: audit_register_class(class, list) (__init):
- p = kcalloc(AUDIT_BITMASK_SIZE, sizeof(__u32), GFP_KERNEL); ENOMEM-on-fail.
- For each n in list until terminator ~0U:
  - if n >= AUDIT_BITMASK_SIZE * 32 - AUDIT_SYSCALL_CLASSES: kfree(p); return -EINVAL.
  - p[AUDIT_WORD(n)] |= AUDIT_BIT(n).
- if class >= AUDIT_SYSCALL_CLASSES ∨ classes[class] already populated: kfree(p); return -EINVAL.
- classes[class] = p.
- return 0.

REQ-7: audit_match_class(class, syscall):
- if syscall >= AUDIT_BITMASK_SIZE * 32: return 0.
- if class >= AUDIT_SYSCALL_CLASSES ∨ !classes[class]: return 0.
- return classes[class][AUDIT_WORD(syscall)] & AUDIT_BIT(syscall).

REQ-8: audit_to_entry_common(rule):
- listnr = rule->flags & ~AUDIT_FILTER_PREPEND.
- Reject AUDIT_FILTER_ENTRY (deprecated -> pr_err + EINVAL).
- Accept AUDIT_FILTER_EXIT / _URING_EXIT / _TASK (CONFIG_AUDITSYSCALL only); always accept _USER / _EXCLUDE / _FS.
- Reject AUDIT_POSSIBLE (deprecated).
- Reject action ∉ {AUDIT_NEVER, AUDIT_ALWAYS}.
- Reject field_count > AUDIT_MAX_FIELDS.
- entry = audit_init_entry(rule->field_count); ENOMEM-on-fail.
- entry->rule.flags = rule->flags & AUDIT_FILTER_PREPEND.
- entry->rule.listnr = listnr; .action = rule->action; .field_count = rule->field_count.
- Copy rule->mask[0..AUDIT_BITMASK_SIZE].
- /* Expand AUDIT_SYSCALL_CLASSES pseudo-bits: top AUDIT_SYSCALL_CLASSES bits of the mask are class indices */
- For i in 0..AUDIT_SYSCALL_CLASSES:
  - bit = AUDIT_BITMASK_SIZE * 32 - i - 1.
  - p = &entry->rule.mask[AUDIT_WORD(bit)].
  - if !(*p & AUDIT_BIT(bit)): continue.
  - *p &= ~AUDIT_BIT(bit).
  - class = classes[i].
  - if class: OR class[j] into mask[j] for j in 0..AUDIT_BITMASK_SIZE.

REQ-9: audit_field_valid(entry, f):
- AUDIT_MSGTYPE only valid on EXCLUDE/USER lists.
- AUDIT_FSTYPE only valid on FS list.
- AUDIT_PERM not allowed on URING_EXIT.
- AUDIT_FILTER_FS list: only AUDIT_FSTYPE / AUDIT_FILTERKEY field types.
- AUDIT_ARG0..3 / AUDIT_PERS / AUDIT_DEVMINOR: all ops valid.
- AUDIT_UID/EUID/SUID/FSUID/LOGINUID/OBJ_UID, AUDIT_GID/EGID/SGID/FSGID/OBJ_GID, AUDIT_PID/MSGTYPE/PPID/DEVMAJOR/EXIT/SUCCESS/INODE/SESSIONID/SUBJ_SEN/SUBJ_CLR/OBJ_LEV_*/SADDR_FAM: bitmask/bittest forbidden (EINVAL).
- AUDIT_SUBJ_USER/ROLE/TYPE, AUDIT_OBJ_USER/ROLE/TYPE, AUDIT_WATCH/DIR/FILTERKEY/LOGINUID_SET/ARCH/FSTYPE/PERM/FILETYPE/FIELD_COMPARE/EXE: only Audit_equal / _not_equal.
- AUDIT_LOGINUID_SET val ∈ {0,1}; AUDIT_PERM val & ~15 == 0; AUDIT_FILETYPE val & ~S_IFMT == 0; AUDIT_FIELD_COMPARE val ≤ AUDIT_MAX_FIELD_COMPARE; AUDIT_SADDR_FAM val < AF_MAX.

REQ-10: audit_data_to_entry(data, datasz):
- entry = audit_to_entry_common(data); on err return ERR.
- bufp = data->buf; remain = datasz - sizeof(struct audit_rule_data).
- For i in 0..data->field_count:
  - f = &entry->rule.fields[i].
  - f->op = audit_to_op(data->fieldflags[i]); reject Audit_bad.
  - f->type = data->fields[i]; f_val = data->values[i].
  - Legacy: AUDIT_LOGINUID + f_val == AUDIT_UID_UNSET → rewrite to AUDIT_LOGINUID_SET with val=0 and set pflags |= AUDIT_LOGINUID_LEGACY.
  - audit_field_valid; on err goto exit_free.
  - Per-field-type extra parse:
    - AUDIT_LOGINUID / _UID / _EUID / _SUID / _FSUID / _OBJ_UID: f->uid = make_kuid(current_user_ns(), f_val); reject !uid_valid.
    - AUDIT_GID / _EGID / _SGID / _FSGID / _OBJ_GID: f->gid = make_kgid(...); reject !gid_valid.
    - AUDIT_ARCH: f->val = f_val; entry->rule.arch_f = f.
    - AUDIT_SUBJ_USER / _ROLE / _TYPE / _SEN / _CLR / OBJ_USER / _ROLE / _TYPE / OBJ_LEV_LOW / _HIGH: audit_unpack_string into f->lsm_str; security_audit_rule_init(f->type, f->op, str, &f->lsm_rule, GFP_KERNEL); -EINVAL from LSM downgraded to warn + keep (policy may reload later).
    - AUDIT_WATCH: audit_unpack_string → audit_to_watch(&entry->rule, str, f_val, f->op); on err kfree(str) + goto exit_free.
    - AUDIT_DIR: audit_unpack_string → audit_make_tree(&entry->rule, str, f->op); kfree(str); err-prop.
    - AUDIT_INODE: f->val = f_val; audit_to_inode(&entry->rule, f).
    - AUDIT_FILTERKEY: reject duplicate / f_val > AUDIT_MAX_KEY_LEN; audit_unpack_string → entry->rule.filterkey = str.
    - AUDIT_EXE: reject duplicate / f_val > PATH_MAX; audit_unpack_string → audit_alloc_mark(&entry->rule, str, f_val); on err kfree(str) + goto exit_free.
    - default: f->val = f_val.
  - buflen accumulates f_val for variable-length fields.
- if inode_f ∧ inode_f->op == Audit_not_equal: inode_f = NULL. /* != on inode is filtered via hash differently */
- exit_free: if rule.tree: audit_put_tree(rule.tree); if rule.exe: audit_remove_mark(rule.exe); audit_free_rule(entry); return ERR_PTR(err).

REQ-11: audit_krule_to_data(krule):
- data = kzalloc_flex(*data, buf, krule->buflen). /* flexible array sized to buflen */
- data->flags = krule->flags | krule->listnr.
- data->action = krule->action; data->field_count.
- For each field: pack lsm_str / audit_watch_path / audit_tree_path / filterkey / audit_mark_path strings into data->buf and record length in data->values[i]; LOGINUID_LEGACY pflags ↔ AUDIT_LOGINUID + AUDIT_UID_UNSET round-trip preserved.
- Copy mask[].

REQ-12: audit_compare_rule(a, b):
- Field-by-field deep compare (flags/pflags/listnr/action/field_count/fields[].type/op/value-of-correct-flavour/mask[]); LSM strings via strcmp; UID/GID via uid_eq/gid_eq.
- Returns 0 on identical, 1 on different.

REQ-13: audit_dupe_rule(old):
- entry = audit_init_entry(old->field_count).
- Copy flags/pflags/listnr/action/mask/prio/buflen/inode_f/field_count/tree (no refcount on tree, see comment).
- memcpy fields[].
- Per-field deep copy: LSM via audit_dupe_lsm_field (kstrdup lsm_str + fresh security_audit_rule_init); AUDIT_FILTERKEY via kstrdup; AUDIT_EXE via audit_dupe_exe.
- If old->watch: audit_get_watch; new->watch = old->watch.
- On err mid-loop: audit_remove_mark(new->exe) if set, audit_free_rule, ERR_PTR.

REQ-14: audit_find_rule(entry, &list_out):
- if rule.inode_f: h = audit_hash_ino(inode_f->val); list = &audit_inode_hash[h].
- else if rule.watch: walk all AUDIT_INODE_BUCKETS (inode unknown until bind); compare via audit_compare_rule.
- else: list = &audit_filter_list[listnr].
- list_for_each_entry to find match.

REQ-15: audit_add_rule(entry):
- Per-AUDIT_FILTER_USER / _EXCLUDE / _FS: dont_count=1 (these don't bump audit_n_rules).
- mutex_lock(audit_filter_mutex).
- audit_find_rule → -EEXIST + audit_put_tree on temporary tree.
- If watch: audit_add_watch (drops + reacquires audit_filter_mutex internally).
- If tree: audit_add_tree_rule.
- prio: AUDIT_FILTER_EXIT / _URING_EXIT use ++prio_high (PREPEND) or --prio_low; other lists prio = ~0ULL.
- list_add{,_rcu} or list_add_tail{,_rcu} depending on PREPEND flag (stripped after).
- CONFIG_AUDITSYSCALL: bump audit_n_rules / audit_signals.
- mutex_unlock.

REQ-16: audit_del_rule(entry):
- mutex_lock(audit_filter_mutex); audit_find_rule → -ENOENT if missing.
- audit_remove_watch_rule / audit_remove_tree_rule / audit_remove_mark_rule if respective field present.
- Decrement audit_n_rules / audit_signals.
- list_del_rcu(&e->list); list_del(&e->rule.list); call_rcu(audit_free_rule_rcu).
- mutex_unlock; audit_put_tree(temporary).

REQ-17: audit_rule_change(type, seq, data, datasz):
- AUDIT_ADD_RULE: entry = audit_data_to_entry; audit_add_rule; audit_log_rule_change("add_rule", &entry->rule, !err).
- AUDIT_DEL_RULE: entry = audit_data_to_entry; audit_del_rule; audit_log_rule_change("remove_rule", ...).
- default: WARN_ON(1); return -EINVAL.
- Post: if err ∨ DEL_RULE: audit_remove_mark(entry->rule.exe) if set; audit_free_rule(entry).

REQ-18: audit_list_rules_send(request_skb, seq):
- dest = kmalloc_obj(*dest). /* struct audit_netlink_list */
- dest->net = get_net(sock_net(NETLINK_CB(request_skb).sk)).
- dest->portid = NETLINK_CB(request_skb).portid.
- skb_queue_head_init(&dest->q).
- mutex_lock(audit_filter_mutex); audit_list_rules(seq, &dest->q); mutex_unlock.
- tsk = kthread_run(audit_send_list_thread, dest, "audit_send_list"); on err purge skb queue + put_net + kfree + return PTR_ERR.

REQ-19: audit_list_rules(seq, q):
- For i in 0..AUDIT_NR_FILTERS, for each r in audit_rules_list[i]:
  - data = audit_krule_to_data(r); on ENOMEM: break.
  - skb = audit_make_reply(seq, AUDIT_LIST_RULES, 0, 1, data, struct_size(data, buf, data->buflen)).
  - skb_queue_tail(q, skb); kfree(data).
- Final terminator skb: audit_make_reply(seq, AUDIT_LIST_RULES, /*done=*/1, ...).

REQ-20: audit_comparator(left, op, right):
- Audit_equal / _not_equal / _lt / _le / _gt / _ge / _bitmask (left & right) / _bittest ((left & right) == right); default 0.

REQ-21: audit_uid_comparator(left_kuid, op, right_kuid) / audit_gid_comparator:
- Use uid_eq/uid_lt/uid_lte/uid_gt/uid_gte (resp. gid_eq/...); bitmask/bittest return 0.

REQ-22: parent_len(path):
- plen = strlen(path); if 0: return 0.
- Walk back through trailing slashes, then back until next '/' or start; if found '/' include it.

REQ-23: audit_compare_dname_path(dname, path, parentlen):
- If parentlen == AUDIT_NAME_FULL: parentlen = parent_len(path).
- p = path + parentlen; strip trailing slashes from p[..pathlen-parentlen].
- if pathlen != dlen: return 1.
- memcmp(p, dname->name, dlen).

REQ-24: audit_filter(msgtype, listtype):
- ret = 1 (audit by default).
- rcu_read_lock.
- For each e in audit_filter_list[listtype]:
  - For each f in e->rule.fields:
    - AUDIT_PID: audit_comparator(task_tgid_nr(current), f->op, f->val).
    - AUDIT_UID / _LOGINUID: audit_uid_comparator(current_uid() / audit_get_loginuid(current), f->op, f->uid).
    - AUDIT_GID: audit_gid_comparator(current_gid(), f->op, f->gid).
    - AUDIT_LOGINUID_SET: audit_comparator(audit_loginuid_set(current), f->op, f->val).
    - AUDIT_MSGTYPE: audit_comparator(msgtype, f->op, f->val).
    - AUDIT_SUBJ_USER/_ROLE/_TYPE/_SEN/_CLR: security_current_getlsmprop_subj + security_audit_rule_match.
    - AUDIT_EXE: audit_exe_compare(current, e->rule.exe); invert if Audit_not_equal.
    - default: goto unlock_and_return. /* unsupported field on user-side filter */
  - if result < 0: goto unlock_and_return.
  - if !result: break (try next rule).
  - if result > 0: ret = (action == AUDIT_NEVER ∨ listtype == AUDIT_FILTER_EXCLUDE) ? 0 : 1; break.
- unlock_and_return: rcu_read_unlock; return ret.

REQ-25: audit_update_lsm_rules():
- mutex_lock(audit_filter_mutex).
- For i in 0..AUDIT_NR_FILTERS, for each r in audit_rules_list[i]: update_lsm_rule(r); preserve first error.
- mutex_unlock.

REQ-26: update_lsm_rule(r):
- entry = container_of(r, audit_entry, rule).
- if !security_audit_rule_known(r): return 0.
- nentry = audit_dupe_rule(r).
- if entry->rule.exe: audit_remove_mark(entry->rule.exe).
- if IS_ERR(nentry):
  - err = PTR_ERR; audit_panic("error updating LSM filters").
  - if r->watch: list_del(&r->rlist).
  - list_del_rcu(&entry->list); list_del(&r->list).
- else:
  - if r->watch ∨ r->tree: list_replace_init(&r->rlist, &nentry->rule.rlist).
  - list_replace_rcu(&entry->list, &nentry->list); list_replace(&r->list, &nentry->rule.list).
- call_rcu(&entry->rcu, audit_free_rule_rcu).

REQ-27: GFP-NOFS expectations:
- The audit subsystem's *runtime* paths (audit_watch_log_rule_change, audit_log_start from fsnotify callbacks) use GFP_NOFS to avoid recursing into the filesystem while a writeback is in progress.
- Filter additions during audit critical paths (e.g. update_lsm_rule called from audit_update_lsm_rules) likewise budget GFP_KERNEL only when audit_filter_mutex is held in process context with no fs lock above — auditfilter.c itself uses GFP_KERNEL for audit_init_entry / audit_unpack_string / kstrdup, but audit_log_rule_change uses GFP_KERNEL because it is invoked from netlink-control context, not fsnotify.

REQ-28: AUDIT_FILTER_PREPEND handling:
- Stripped from listnr in audit_to_entry_common (listnr = flags & ~AUDIT_FILTER_PREPEND).
- Stored separately in entry->rule.flags.
- audit_add_rule: PREPEND ⟹ list_add at head, prio = ++prio_high (highest); else list_add_tail, prio = --prio_low.
- After insertion the PREPEND bit is cleared from rule.flags.

## Acceptance Criteria

- [ ] AC-1: `auditctl -A exit,always -F arch=b64 -S open -k file_opens` netlink AUDIT_ADD_RULE → audit_data_to_entry parses arch + syscall mask + filterkey; audit_add_rule inserts into audit_filter_list[AUDIT_FILTER_EXIT]; entry->rule.mask bit for `open` set.
- [ ] AC-2: `auditctl -l` → AUDIT_LIST_RULES dispatch spawns audit_send_list kthread; round-trip kernel-rule → wire → user space is byte-identical to original input (modulo PREPEND bit stripping and LOGINUID legacy mapping).
- [ ] AC-3: AUDIT_FILTER_ENTRY (deprecated) rule → audit_to_entry_common pr_err + EINVAL.
- [ ] AC-4: action == AUDIT_POSSIBLE (deprecated) → pr_err + EINVAL.
- [ ] AC-5: AUDIT_MSGTYPE field on filter != USER/EXCLUDE → audit_field_valid -EINVAL.
- [ ] AC-6: AUDIT_FSTYPE field on filter != FS → audit_field_valid -EINVAL.
- [ ] AC-7: Bitmask / bittest op on AUDIT_PID → audit_field_valid -EINVAL.
- [ ] AC-8: AUDIT_WATCH or AUDIT_DIR or AUDIT_EXE field with op != equal/!= → audit_field_valid -EINVAL.
- [ ] AC-9: AUDIT_PERM val with bits outside (4-bit mask) → -EINVAL.
- [ ] AC-10: AUDIT_FILTERKEY duplicated or length > AUDIT_MAX_KEY_LEN → -EINVAL.
- [ ] AC-11: AUDIT_EXE duplicated or length > PATH_MAX → -EINVAL.
- [ ] AC-12: AUDIT_LOGINUID with AUDIT_UID_UNSET → rewritten internally to AUDIT_LOGINUID_SET val=0 + pflags |= AUDIT_LOGINUID_LEGACY; round-trip dump preserves legacy form via audit_krule_to_data.
- [ ] AC-13: audit_register_class with overflow n ≥ AUDIT_BITMASK_SIZE * 32 - AUDIT_SYSCALL_CLASSES → -EINVAL.
- [ ] AC-14: audit_register_class twice for same class → -EINVAL.
- [ ] AC-15: audit_to_entry_common expansion: rule mask with top-class bit set is OR'd with classes[i] into per-syscall bitmask.
- [ ] AC-16: Duplicate rule via AUDIT_ADD_RULE → audit_find_rule returns existing entry; audit_add_rule -EEXIST.
- [ ] AC-17: AUDIT_DEL_RULE non-existent → audit_del_rule -ENOENT.
- [ ] AC-18: Concurrent AUDIT_ADD_RULE and runtime audit_filter: RCU-traversal sees either old or new entry, never a partial one.
- [ ] AC-19: LSM-policy reload calls audit_update_lsm_rules → every rule containing AUDIT_SUBJ_* / OBJ_* has its lsm_rule re-initialised via security_audit_rule_init.
- [ ] AC-20: audit_list_rules_send: kthread_run failure → skb queue purged, get_net is balanced by put_net, dest freed, error returned.
- [ ] AC-21: AUDIT_FILTER_PREPEND on EXIT/URING_EXIT lists prepends and gets the highest priority (prio = ++prio_high).
- [ ] AC-22: audit_filter (runtime): rule with AUDIT_NEVER action matching → returns 0 (skip audit).
- [ ] AC-23: audit_filter on AUDIT_FILTER_EXCLUDE: rule matches → returns 0 even if action is AUDIT_ALWAYS.
- [ ] AC-24: audit_filter encounters unsupported field type → bail (rcu_read_unlock) returning default ret=1.

## Architecture

```
struct AuditEntry {
  list:  RcuListLink,           // -> audit_filter_list[]
  rcu:   RcuHead,
  rule:  AuditKrule,
}

struct AuditKrule {
  flags:        u32,            // AUDIT_FILTER_PREPEND (transient)
  pflags:       u32,            // AUDIT_LOGINUID_LEGACY
  listnr:       u32,            // AUDIT_FILTER_*
  action:       u32,            // AUDIT_NEVER / _ALWAYS
  field_count:  u32,
  mask:         [u32; AUDIT_BITMASK_SIZE],
  buflen:       u32,
  prio:         u64,
  fields:       KBox<[AuditField]>,
  arch_f:       Option<*const AuditField>,
  inode_f:      Option<*const AuditField>,
  watch:        Option<KArc<AuditWatch>>,
  tree:         Option<KArc<AuditTree>>,
  exe:          Option<KArc<AuditFsnotifyMark>>,
  filterkey:    Option<KBox<KStr>>,
  list:         ListLink,       // -> audit_rules_list[]
  rlist:        ListLink,       // -> watch.rules / tree.rules
}

struct AuditField {
  op:       AuditOp,
  type_:    u32,                // AUDIT_*
  val:      u32,
  uid:      KUidT,
  gid:      KGidT,
  lsm_str:  Option<KBox<KStr>>,
  lsm_rule: Option<*mut c_void>,
}
```

`AuditEntry::change(type, seq, data, datasz) -> Result<()>`:
1. match type:
   - AUDIT_ADD_RULE:
     - entry = AuditEntry::from_data(data, datasz)?.
     - err = AuditEntry::add(entry).
     - AuditEntry::log_change("add_rule", &entry.rule, err.is_ok()).
   - AUDIT_DEL_RULE:
     - entry = AuditEntry::from_data(data, datasz)?.
     - err = AuditEntry::del(entry).
     - AuditEntry::log_change("remove_rule", &entry.rule, err.is_ok()).
   - _: WARN_ON(1); return Err(EINVAL).
2. if err.is_err() ∨ type == AUDIT_DEL_RULE:
   - if entry.rule.exe.is_some(): audit_remove_mark(entry.rule.exe).
   - AuditEntry::free(entry).
3. err.

`AuditEntry::from_data(data, datasz) -> Result<KBox<AuditEntry>>`:
1. entry = AuditEntry::to_entry_common(data)?.
2. bufp = &data.buf; remain = datasz - sizeof(AuditRuleData).
3. for i in 0..data.field_count:
   - f = &mut entry.rule.fields[i].
   - f.op = AuditOp::from_wire(data.fieldflags[i]).ok_or(EINVAL)?.
   - f.type_ = data.fields[i]; f_val = data.values[i].
   - legacy_loginuid_rewrite(&mut f, &mut entry.rule.pflags, &mut f_val).
   - AuditField::is_valid(&entry, f)?.
   - per-type:
     - UID family: f.uid = make_kuid(current_user_ns(), f_val); require uid_valid.
     - GID family: f.gid = make_kgid(...); require gid_valid.
     - ARCH: f.val = f_val; entry.rule.arch_f = Some(f).
     - SUBJ_* / OBJ_*: str = AuditEntry::unpack_string(&mut bufp, &mut remain, f_val)?; entry.rule.buflen += f_val; f.lsm_str = Some(str); security_audit_rule_init(f.type_, f.op, str, &mut f.lsm_rule, GFP_KERNEL) (EINVAL downgraded to warn).
     - WATCH: str = unpack_string; AuditWatch::to_watch(&mut entry.rule, str, f_val, f.op)?; entry.rule.buflen += f_val.
     - DIR: str = unpack_string; audit_make_tree(&mut entry.rule, str, f.op); kfree(str); entry.rule.buflen += f_val.
     - INODE: f.val = f_val; AuditKrule::to_inode(&mut entry.rule, f)?.
     - FILTERKEY: reject duplicate / oversize; str = unpack_string; entry.rule.filterkey = Some(str); buflen += f_val.
     - EXE: reject duplicate / oversize; str = unpack_string; entry.rule.exe = audit_alloc_mark(&entry.rule, str, f_val); buflen += f_val.
     - default: f.val = f_val.
4. if entry.rule.inode_f.is_some() ∧ entry.rule.inode_f.op == Audit_not_equal: entry.rule.inode_f = None.
5. Ok(entry).

`AuditEntry::add(entry) -> Result<()>`:
1. dont_count = entry.rule.listnr ∈ {AUDIT_FILTER_USER, _EXCLUDE, _FS}.
2. audit_filter_mutex.lock().
3. e = AuditEntry::find(entry, &mut list).
4. if e.is_some(): mutex.unlock(); if entry.rule.tree.is_some(): audit_put_tree(entry.rule.tree); return Err(EEXIST).
5. if entry.rule.watch.is_some(): AuditWatch::add(&mut entry.rule, &mut list)?. /* drops + reacquires mutex */
6. if entry.rule.tree.is_some(): audit_add_tree_rule(&mut entry.rule)?.
7. entry.rule.prio = ~0u64; if listnr ∈ {EXIT, URING_EXIT}: prio = (flags & PREPEND) ? ++prio_high : --prio_low.
8. if entry.rule.flags & AUDIT_FILTER_PREPEND:
   - list_add(&entry.rule.list, &AUDIT_RULES_LIST[listnr]).
   - list_add_rcu(&entry.list, list).
   - entry.rule.flags &= !AUDIT_FILTER_PREPEND.
9. else: list_add_tail variants.
10. if CONFIG_AUDITSYSCALL: !dont_count ⟹ audit_n_rules += 1; !audit_match_signal(&entry) ⟹ audit_signals += 1.
11. mutex.unlock(); Ok(()).

`AuditEntry::del(entry) -> Result<()>`:
1. audit_filter_mutex.lock().
2. e = AuditEntry::find(entry, &mut list); if None: mutex.unlock(); audit_put_tree(temporary); return Err(ENOENT).
3. if e.rule.watch.is_some(): AuditWatch::remove_rule(&e.rule).
4. if e.rule.tree.is_some(): audit_remove_tree_rule(&e.rule).
5. if e.rule.exe.is_some(): audit_remove_mark_rule(&e.rule).
6. CONFIG_AUDITSYSCALL: decrement counters symmetrically with audit_add_rule.
7. list_del_rcu(&e.list); list_del(&e.rule.list); call_rcu(&e.rcu, audit_free_rule_rcu).
8. mutex.unlock(); audit_put_tree(temporary); Ok(()).

`AuditEntry::find(entry, list_out) -> Option<&AuditEntry>`:
1. if entry.rule.inode_f.is_some(): h = audit_hash_ino(inode_f.val); *list_out = &AUDIT_INODE_HASH[h].
2. else if entry.rule.watch.is_some(): /* unknown ino; sweep all buckets */ for h in 0..AUDIT_INODE_BUCKETS: walk and compare; return on hit.
3. else: *list_out = &AUDIT_FILTER_LIST[listnr].
4. Walk *list_out, return first entry with audit_compare_rule == 0.

`AuditEntry::list_send(request_skb, seq) -> Result<()>`:
1. dest = kmalloc(AuditNetlinkList)?.
2. dest.net = get_net(sock_net(NETLINK_CB(request_skb).sk)).
3. dest.portid = NETLINK_CB(request_skb).portid.
4. skb_queue_head_init(&dest.q).
5. audit_filter_mutex.lock(); AuditEntry::list_all(seq, &dest.q); audit_filter_mutex.unlock().
6. tsk = kthread_run(audit_send_list_thread, dest, "audit_send_list").
7. on Err(tsk): skb_queue_purge(&dest.q); put_net(dest.net); kfree(dest); return Err.
8. Ok(()).

`AuditFilter::filter(msgtype, listtype) -> i32`:
1. ret = 1.
2. rcu_read_lock().
3. for e in AUDIT_FILTER_LIST[listtype].iter_rcu():
   - result = 0.
   - for f in e.rule.fields.iter():
     - result = match f.type_ {
         AUDIT_PID => audit_comparator(task_tgid_nr(current), f.op, f.val),
         AUDIT_UID => audit_uid_comparator(current_uid(), f.op, f.uid),
         AUDIT_GID => audit_gid_comparator(current_gid(), f.op, f.gid),
         AUDIT_LOGINUID => audit_uid_comparator(audit_get_loginuid(current), f.op, f.uid),
         AUDIT_LOGINUID_SET => audit_comparator(audit_loginuid_set(current), f.op, f.val),
         AUDIT_MSGTYPE => audit_comparator(msgtype, f.op, f.val),
         AUDIT_SUBJ_* => { security_current_getlsmprop_subj(&mut prop); security_audit_rule_match(&prop, f.type_, f.op, f.lsm_rule) },
         AUDIT_EXE => { r = audit_exe_compare(current, e.rule.exe); if f.op == Audit_not_equal { !r } else { r } },
         _ => goto unlock_and_return,
       }.
     - if result < 0: goto unlock_and_return.
     - if !result: break.
   - if result > 0:
     - if e.rule.action == AUDIT_NEVER ∨ listtype == AUDIT_FILTER_EXCLUDE: ret = 0.
     - break.
4. unlock_and_return: rcu_read_unlock(); ret.

`AuditFilter::update_lsm_rules() -> Result<()>` (called from LSM-policy-reload):
1. audit_filter_mutex.lock().
2. for i in 0..AUDIT_NR_FILTERS, for r in AUDIT_RULES_LIST[i].iter_safe(): update_lsm_rule(r).
3. audit_filter_mutex.unlock().

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `audit_to_entry_common_rejects_deprecated` | INVARIANT | per-AUDIT_FILTER_ENTRY ∨ AUDIT_POSSIBLE ⟹ EINVAL. |
| `field_count_capped` | INVARIANT | per-audit_to_entry_common: field_count ≤ AUDIT_MAX_FIELDS. |
| `unpack_string_bounds` | INVARIANT | per-audit_unpack_string: len ∈ (0, PATH_MAX] ∧ len ≤ remain. |
| `audit_field_valid_per_listnr` | INVARIANT | per-AUDIT_MSGTYPE in EXCLUDE/USER only; AUDIT_FSTYPE in FS only; AUDIT_PERM forbidden on URING_EXIT. |
| `audit_field_valid_per_op` | INVARIANT | per-Audit_bitmask/_bittest forbidden on uid/gid/inode/etc.; per-only-=/=!= for AUDIT_SUBJ_USER..AUDIT_EXE. |
| `audit_field_valid_per_val_range` | INVARIANT | per-AUDIT_LOGINUID_SET val ∈ {0,1}; PERM ≤ 15; FILETYPE & S_IFMT; FIELD_COMPARE ≤ AUDIT_MAX_FIELD_COMPARE; SADDR_FAM < AF_MAX. |
| `register_class_overflow` | INVARIANT | per-audit_register_class: n < AUDIT_BITMASK_SIZE * 32 - AUDIT_SYSCALL_CLASSES. |
| `register_class_one_shot` | INVARIANT | per-audit_register_class: classes[class] non-NULL ⟹ EINVAL. |
| `dupe_rule_preserves_field_count` | INVARIANT | per-audit_dupe_rule: new.field_count == old.field_count. |
| `find_rule_dispatch` | INVARIANT | per-audit_find_rule: inode_f ⟹ audit_inode_hash[h]; watch ⟹ all buckets; default ⟹ audit_filter_list[listnr]. |
| `add_rule_eexist` | INVARIANT | per-audit_add_rule: duplicate ⟹ EEXIST + tree-cleanup if temporary. |
| `del_rule_enoent` | INVARIANT | per-audit_del_rule: missing ⟹ ENOENT. |
| `list_send_balance_get_put_net` | INVARIANT | per-audit_list_rules_send: every get_net paired with put_net on the failure path. |
| `prepend_priority_strict` | INVARIANT | per-audit_add_rule: PREPEND ⟹ list_add (head) + prio = ++prio_high; else list_add_tail + prio = --prio_low (EXIT/URING_EXIT only). |
| `dont_count_lists_skip_audit_n_rules` | INVARIANT | per-AUDIT_FILTER_USER/_EXCLUDE/_FS: audit_n_rules unchanged. |
| `update_lsm_rule_replace_atomic` | INVARIANT | per-update_lsm_rule: list_replace_rcu(entry, nentry); list_replace(r->list, nentry->rule.list); single observer state. |
| `filter_default_audit_when_no_match` | INVARIANT | per-audit_filter: no rule matches ⟹ ret = 1. |
| `filter_never_or_exclude_zeroes` | INVARIANT | per-audit_filter: match with AUDIT_NEVER ∨ listtype == AUDIT_FILTER_EXCLUDE ⟹ ret = 0. |

### Layer 2: TLA+

`kernel/audit/auditfilter.tla`:
- States: netlink-arrive / parse / validate / find / insert-or-replace / RCU-publish / runtime-match / delete / lsm-reload.
- Properties:
  - `safety_audit_filter_mutex_serialises_writers` — per-add/del/list/update_lsm: at most one writer at a time on the rule lists.
  - `safety_rcu_traversal_sees_consistent_entry` — per-audit_filter / fsnotify-readers: list_replace_rcu + call_rcu ⟹ no torn read.
  - `safety_audit_n_rules_balanced` — per-add/del symmetric increments/decrements; dont_count_lists never bump.
  - `safety_audit_signals_balanced` — per-audit_match_signal == 0 paths.
  - `safety_lsm_reload_preserves_field_count` — per-update_lsm_rule: new.field_count == old.field_count.
  - `safety_audit_filter_never_observes_partial_entry` — per-list_replace_rcu atomicity.
  - `liveness_add_rule_eventually_publishes` — per-AUDIT_ADD_RULE: writers complete in finite time; readers eventually observe.
  - `liveness_list_rules_kthread_drains_queue` — per-AUDIT_LIST_RULES: audit_send_list_thread drains skb queue.
  - `safety_prio_monotone_within_filter_exit` — per-PREPEND vs append on EXIT/URING_EXIT.
  - `safety_inode_hash_dispatch` — per-audit_find_rule: inode_f ⟹ exactly one bucket walked.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `AuditEntry::to_entry_common` post: listnr ∈ valid-set ∧ action ∈ {NEVER, ALWAYS} ∧ field_count ≤ AUDIT_MAX_FIELDS ∧ mask class-expansion applied | `AuditEntry::to_entry_common` |
| `AuditEntry::from_data` post: every field validated; inode_f cleared if op == Audit_not_equal; buflen sum-of-string-lengths | `AuditEntry::from_data` |
| `AuditField::is_valid` post: per-listnr / per-type / per-op / per-val constraints all hold | `AuditField::is_valid` |
| `AuditClass::register` post: classes[class] non-NULL ∧ bitmap correct ∧ no overlap | `AuditClass::register` |
| `AuditClass::match` post: returns 1 iff syscall bit set in classes[class][AUDIT_WORD(syscall)] | `AuditClass::match` |
| `AuditOp::from_wire` post: bijection between wire AUDIT_* and Audit_* enum; unknown ⟹ Audit_bad | `AuditOp::from_wire` |
| `AuditKrule::cmp` post: returns 0 iff structurally equivalent | `AuditKrule::cmp` |
| `AuditEntry::dupe` post: new.field_count == old.field_count ∧ deep-copied LSM/filterkey/exe ∧ shared watch refcount++ ∧ shared tree (no refcount, per-comment) | `AuditEntry::dupe` |
| `AuditEntry::find` post: returns first match (or None) under audit_filter_mutex | `AuditEntry::find` |
| `AuditEntry::add` post: success ⟹ entry on both audit_filter_list[listnr] (RCU) and audit_rules_list[listnr] (mutex) ∧ counters consistent | `AuditEntry::add` |
| `AuditEntry::del` post: success ⟹ entry off both lists ∧ call_rcu scheduled | `AuditEntry::del` |
| `AuditEntry::change` post: AUDIT_DEL_RULE always frees the user-side template entry post-call | `AuditEntry::change` |
| `AuditEntry::list_send` post: failure ⟹ get_net balanced by put_net | `AuditEntry::list_send` |
| `AuditFilter::filter` post: ret ∈ {0, 1} ∧ rcu_read_lock balanced | `AuditFilter::filter` |
| `AuditFilter::update_lsm_rules` post: every rule with known LSM type re-initialised ∨ removed | `AuditFilter::update_lsm_rules` |

### Layer 4: Verus/Creusot functional

`Per-auditctl -w / -a / -A / -D / -l` netlink message → audit_data_to_entry parse → audit_add_rule / audit_del_rule / audit_list_rules_send → RCU-published filter list → runtime audit_filter on USER/EXCLUDE/TYPE/FS lists, or syscall-exit filter via auditsc.c → semantic equivalence with upstream per Documentation/audit/audit.txt + auditctl(8) round-trip byte-identical. LSM-policy reload survives via audit_update_lsm_rules.

## Hardening

(Inherits row-1 features from `kernel/audit/00-overview.md` § Hardening.)

auditfilter reinforcement:

- **Per-AUDIT_FILTER_ENTRY deprecated → EINVAL** — defense against per-legacy-rule installing on a list that no longer evaluates as expected.
- **Per-AUDIT_POSSIBLE deprecated → EINVAL** — defense against per-action that has no defined semantics.
- **Per-action restricted to {AUDIT_NEVER, AUDIT_ALWAYS}** — defense against per-undefined-action-value.
- **Per-field_count ≤ AUDIT_MAX_FIELDS** — defense against per-DoS via huge rule.
- **Per-string fields ≤ PATH_MAX (audit_unpack_string)** — defense against per-huge-string allocation.
- **Per-FILTERKEY ≤ AUDIT_MAX_KEY_LEN + no-duplicate** — defense against per-key bloat.
- **Per-EXE ≤ PATH_MAX + no-duplicate** — defense against per-exe-mark bloat.
- **Per-AUDIT_PERM val ≤ 4-bit + AUDIT_FILETYPE val & S_IFMT + AUDIT_SADDR_FAM < AF_MAX** — defense against per-malformed-field-value.
- **Per-bitmask/bittest forbidden on uid/gid/inode/etc.** — defense against per-nonsensical-comparator that could match unexpectedly.
- **Per-= / != only for AUDIT_SUBJ_* / OBJ_* / WATCH / DIR / FILTERKEY / EXE / ARCH / FILETYPE / FIELD_COMPARE / PERM** — defense against per-string-vs-ordering confusion.
- **Per-make_kuid/_kgid + uid_valid/gid_valid** — defense against per-user-ns escape and per-invalid-mapping.
- **Per-security_audit_rule_init EINVAL downgraded to warn (not free)** — defense against per-LSM-policy-reload-window dropping rules silently; reload re-init can later succeed.
- **Per-audit_filter_mutex serialises all writers** — defense against per-concurrent-rule-add corrupting the list.
- **Per-RCU readers + list_replace_rcu + call_rcu(audit_free_rule_rcu)** — defense against per-runtime-traversal UAF.
- **Per-audit_list_rules_send kthread offload** — defense against per-deadlock when auditctl reads slower than kernel can dump.
- **Per-get_net/put_net balanced in audit_list_rules_send failure path** — defense against per-net-namespace leak.
- **Per-AUDIT_FILTER_USER / _EXCLUDE / _FS excluded from audit_n_rules** — defense against per-stat-counter pollution.
- **Per-update_lsm_rule audit_panic on dupe-failure** — defense against per-stale-LSM-rule retaining after policy reload (visible failure rather than silent stale).
- **Per-list_replace_rcu atomic publish** — defense against per-half-replaced rule visible to filter.
- **Per-AUDIT_FILTER_PREPEND priority via prio_high/_low** — defense against per-prepend-vs-append ordering ambiguity.
- **Per-audit_log_rule_change("add_rule"/"remove_rule") audit-trail** — defense against per-undetected rule change.
- **Per-audit_filter default ret=1 on unsupported field** — defense against per-fail-open silent drop on user/exclude lists (failure-mode is audit-on, not audit-off).

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds-check `copy_from_user(data, ..., struct audit_rule_data)` against the slab whitelist; `audit_rule_data` is variable-length (buf + fields + values) and a classic OOB-read source.
- **PAX_KERNEXEC** — keep `audit_krule_ops`, the per-filter `audit_entry` list heads (`audit_filter_list[]`), and LSM rule-conversion vtables in `__ro_after_init`.
- **PAX_RANDKSTACK** — re-randomize stack on `audit_receive`/`audit_rule_to_entry` to defeat disclosure of the staged rule on stack.
- **PAX_REFCOUNT** — saturating refcount on `struct audit_krule` (via `audit_entry`) and on attached `audit_watch`/`audit_tree`/`audit_fsnotify_mark`; wraparound MUST panic.
- **PAX_MEMORY_SANITIZE** — zero `audit_krule->fields[]` (with embedded LSM rule pointers and string buffers) on free; never recycle a prior policy's `lsm_str` into a new rule.
- **PAX_UDEREF** — strict user/kernel separation when parsing `struct audit_rule_data __user *`; the netlink path MUST NOT deref a smuggled kernel pointer.
- **PAX_RAP / kCFI** — type-check the LSM `security_audit_rule_init`/`_match`/`_free` callbacks and the `f_op` of any fd-based field.
- **GRKERNSEC_HIDESYM** — hide `audit_filter_*`, `audit_add_rule`, `audit_del_rule` from non-root kallsyms.
- **GRKERNSEC_DMESG** — restrict dmesg so per-rule add/del spew (which echoes attacker-supplied field strings) cannot be harvested.
- **`audit_rule_data` PAX_USERCOPY** — explicit usercopy whitelist for the staging buffer because length is attacker-controlled and the kernel does pointer arithmetic into it (`bufp`/`field_count`).
- **LSM-policy reload integrity** — on SELinux/AppArmor policy reload, `update_lsm_rule()` MUST atomically re-init every `audit_krule.lsm_rule`; a half-reloaded rule is a fail-open primitive. PaX-style: if any rule fails re-init under reload, `audit_panic("update_lsm_rule failed")` MUST escalate per `audit_failure` policy, never silently drop.
- **CAP_AUDIT_CONTROL boundary** — every `AUDIT_ADD_RULE`/`AUDIT_DEL_RULE` MUST be re-gated on `CAP_AUDIT_CONTROL` against current creds, not the netlink socket's stale creds.
- **Rationale** — auditfilter is the kernel's own policy DSL parsed from netlink; it is both attacker-reachable (CAP_AUDIT_CONTROL-gated, but the rule-data layout has been the source of multiple OOB-read CVEs) and security-critical (a tampered rule is silent audit blindness). PAX_USERCOPY on `audit_rule_data` and atomic LSM re-init close the two historical bug classes.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `kernel/audit_watch.c` per-watch fsnotify mark management (covered in `audit-watch.md`).
- `kernel/audit_tree.c` recursive directory watches (covered in `audit-tree.md`).
- `kernel/audit_fsnotify.c` AUDIT_EXE mark management (covered in `audit-fsnotify.md`).
- `kernel/auditsc.c` syscall context and per-syscall match (covered in `auditsc.md`).
- `kernel/audit.c` netlink socket and per-record logging (covered in `audit-core.md`).
- LSM hooks (`security/security.c`) implementation.
- Implementation code.
