# Tier-5 UAPI: include/uapi/linux/audit.h — Linux audit netlink ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/audit.h (~530 lines)
  - include/linux/audit.h (kernel-internal complement)
  - include/uapi/linux/elf-em.h (EM_* used by AUDIT_ARCH_*)
  - kernel/audit.c (netlink listener + struct audit_status)
  - kernel/auditfilter.c (struct audit_rule_data)
  - kernel/auditsc.c (SYSCALL/PATH/CWD records)
  - Documentation/admin-guide/LSM/index.rst (LSM/MAC record types)
  - Documentation/userspace-api/seccomp_filter.rst (AUDIT_SECCOMP record)
-->

## Summary

The Linux audit subsystem delivers kernel-generated security records over a netlink socket of family `NETLINK_AUDIT` (16). Userspace `auditd` (or any process holding `CAP_AUDIT_CONTROL`) configures the kernel via numbered command messages — `AUDIT_GET` / `AUDIT_SET` set daemon state and policy; `AUDIT_ADD_RULE` / `AUDIT_DEL_RULE` / `AUDIT_LIST_RULES` install per-syscall filtering rules whose body is a `struct audit_rule_data`; `AUDIT_LIST` / `AUDIT_ADD` / `AUDIT_DEL` are the deprecated v1 equivalents. The kernel emits event records in fixed numeric ranges: `AUDIT_FIRST_USER_MSG`..`_LAST_USER_MSG` (1100..1199, userspace trusted apps); `AUDIT_FIRST_USER_MSG2`..`_LAST_USER_MSG2` (2100..2999); kernel event records 1300..1399 (SYSCALL / PATH / IPC / CONFIG_CHANGE / SOCKADDR / CWD / EXECVE / IPC_SET_PERM / MQ_* / KERNEL_OTHER / FD_PAIR / OBJ_PID / TTY / EOE / BPRM_FCAPS / CAPSET / MMAP / NETFILTER_PKT / NETFILTER_CFG / SECCOMP / PROCTITLE / FEATURE_CHANGE / REPLACE / KERN_MODULE / FANOTIFY / TIME_INJOFFSET / TIME_ADJNTPVAL / BPF / EVENT_LISTENER / URINGOP / OPENAT2 / DM_*); access-control records 1400..1499 (AVC / MAC_* / IPE_* / LANDLOCK_*); anomaly records 1700..1799; integrity records 1800..1899; the legacy bulk channel `AUDIT_KERNEL` (2000). Filter rules carry `flags` (filter list: USER / TASK / ENTRY / WATCH / EXIT / EXCLUDE / FS / URING_EXIT), `action` (`AUDIT_NEVER` / `_POSSIBLE` / `_ALWAYS`), `fields[]` (e.g. `AUDIT_PID`, `AUDIT_UID`, `AUDIT_ARCH`, `AUDIT_MSGTYPE`, `AUDIT_PATH`, `AUDIT_EXE`, `AUDIT_OBJ_*`, `AUDIT_FILETYPE`, `AUDIT_FIELD_COMPARE`, …), `values[]`, `fieldflags[]` (comparison operator). Critical for: SELinux / AppArmor / IMA / SECCOMP / IPE / Landlock event delivery, regulatory-compliance logging (FISMA, PCI-DSS, HIPAA), forensic post-incident reconstruction, and any LSM that exposes denials/grants to userspace.

This Tier-5 covers the full UAPI surface of `include/uapi/linux/audit.h` (~530 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `AUDIT_GET` / `AUDIT_SET` | per-status RPC | `audit::msg::get_set_status` |
| `AUDIT_LIST` / `_ADD` / `_DEL` (deprecated) | per-v1-rule RPC | `audit::msg::v1_rules` |
| `AUDIT_USER` (deprecated) | per-user-msg RPC | `audit::msg::user_legacy` |
| `AUDIT_LOGIN` | per-login-uid set | `audit::msg::login` |
| `AUDIT_WATCH_INS/REM/LIST` | per-fs-watch RPC | `audit::msg::watch` |
| `AUDIT_SIGNAL_INFO` | per-auditd signal info | `audit::msg::signal_info` |
| `AUDIT_ADD_RULE/DEL_RULE/LIST_RULES` | per-rule RPC | `audit::msg::rules` |
| `AUDIT_TRIM / _MAKE_EQUIV` | per-watched-tree maintenance | `audit::msg::tree` |
| `AUDIT_TTY_GET/SET` | per-TTY-audit state | `audit::msg::tty` |
| `AUDIT_SET_FEATURE / _GET_FEATURE` | per-feature toggle | `audit::msg::feature` |
| `AUDIT_FIRST_USER_MSG..LAST_USER_MSG` | per-user-space-msg range | `audit::range::user1` |
| `AUDIT_FIRST_USER_MSG2..LAST_USER_MSG2` | per-user-space-msg range 2 | `audit::range::user2` |
| `AUDIT_DAEMON_START/END/ABORT/CONFIG` | per-daemon lifecycle | `audit::rec::daemon` |
| `AUDIT_SYSCALL/PATH/IPC/SOCKETCALL/CONFIG_CHANGE/SOCKADDR/CWD/EXECVE/...` | per-event record class | `audit::rec::kern` |
| `AUDIT_AVC..AUDIT_MAC_OBJ_CONTEXTS` | per-LSM record class | `audit::rec::lsm` |
| `AUDIT_FIRST_KERN_ANOM_MSG..` / anomaly / integrity ranges | per-record range | `audit::range::*` |
| `AUDIT_KERNEL` (2000) | per-async bulk record | `audit::rec::kernel_legacy` |
| `AUDIT_FILTER_USER..AUDIT_FILTER_URING_EXIT` | per-filter list | `audit::FilterList` |
| `AUDIT_NR_FILTERS` | per-filter-count | `audit::NR_FILTERS` |
| `AUDIT_FILTER_PREPEND` | per-rule head-insert flag | `audit::FilterFlags::PREPEND` |
| `AUDIT_NEVER/POSSIBLE/ALWAYS` | per-rule action | `audit::Action` |
| `AUDIT_MAX_FIELDS / _KEY_LEN / BITMASK_SIZE` | per-rule limits | `audit::limits` |
| `AUDIT_SYSCALL_CLASSES + AUDIT_CLASS_*` | per-syscall class | `audit::SyscallClass` |
| `AUDIT_COMPARE_*` | per-field-compare codes | `audit::Compare` |
| `AUDIT_PID..AUDIT_SADDR_FAM` | per-rule field id | `audit::FieldId` |
| `AUDIT_ARG0..ARG3` | per-syscall-arg field | `audit::FieldId::Arg*` |
| `AUDIT_FILTERKEY` | per-rule key string | `audit::FieldId::FilterKey` |
| `AUDIT_NEGATE / _BIT_MASK / _LESS_THAN / _GREATER_THAN / _NOT_EQUAL / _EQUAL / _BIT_TEST / _LESS_THAN_OR_EQUAL / _GREATER_THAN_OR_EQUAL / _OPERATORS` | per-rule op | `audit::Op` |
| `enum { Audit_equal..Audit_bad }` | per-internal-op enum | `audit::OpInternal` |
| `AUDIT_STATUS_*` | per-status mask bit | `audit::StatusMask` |
| `AUDIT_FEATURE_BITMAP_*` | per-feature bit | `audit::FeatureBitmap` |
| `AUDIT_FAIL_SILENT/PRINTK/PANIC` | per-fail-to-log policy | `audit::FailPolicy` |
| `__AUDIT_ARCH_*` / `AUDIT_ARCH_*` | per-syscall-ABI marker | `audit::Arch` |
| `AUDIT_PERM_*` | per-perm-field bit | `audit::Perm` |
| `AUDIT_MESSAGE_TEXT_MAX` | per-record body max | `audit::TEXT_MAX` |
| `enum audit_nlgrps` + `AUDIT_NLGRP_*` | per-multicast group | `audit::NlGrp` |
| `struct audit_status` | per-`AUDIT_GET/SET` payload | `AuditStatus` |
| `struct audit_features` | per-`AUDIT_SET_FEATURE` payload | `AuditFeatures` |
| `AUDIT_FEATURE_VERSION` | per-features-struct version | `audit::FEATURE_VERSION` |
| `AUDIT_FEATURE_ONLY_UNSET_LOGINUID / _LOGINUID_IMMUTABLE` | per-feature id | `audit::Feature` |
| `struct audit_tty_status` | per-`AUDIT_TTY_GET/SET` payload | `AuditTtyStatus` |
| `AUDIT_UID_UNSET / _SID_UNSET` | per-sentinel | `audit::unset` |
| `struct audit_rule_data` | per-`ADD_RULE`/`DEL_RULE`/`LIST_RULES` payload | `AuditRuleData` |

## ABI surface (constants + structs)

### Netlink family

```
#define NETLINK_AUDIT  9   /* defined in <linux/netlink.h> */
```

### Command-and-control messages (1000..1099, bi-directional)

```
AUDIT_GET           = 1000   /* Get status */
AUDIT_SET           = 1001   /* Set status (enable/disable/auditd) */
AUDIT_LIST          = 1002   /* List syscall rules — deprecated v1 */
AUDIT_ADD           = 1003   /* Add syscall rule — deprecated v1 */
AUDIT_DEL           = 1004   /* Delete syscall rule — deprecated v1 */
AUDIT_USER          = 1005   /* Message from userspace — deprecated */
AUDIT_LOGIN         = 1006   /* Define the login id and information */
AUDIT_WATCH_INS     = 1007   /* Insert file/dir watch entry */
AUDIT_WATCH_REM     = 1008   /* Remove file/dir watch entry */
AUDIT_WATCH_LIST    = 1009   /* List all file/dir watches */
AUDIT_SIGNAL_INFO   = 1010   /* Get info about sender of signal to auditd */
AUDIT_ADD_RULE      = 1011   /* Add syscall filtering rule (v2: audit_rule_data) */
AUDIT_DEL_RULE      = 1012   /* Delete syscall filtering rule (v2) */
AUDIT_LIST_RULES    = 1013   /* List syscall filtering rules (v2) */
AUDIT_TRIM          = 1014   /* Trim junk from watched tree */
AUDIT_MAKE_EQUIV    = 1015   /* Append to watched tree */
AUDIT_TTY_GET       = 1016   /* Get TTY auditing status */
AUDIT_TTY_SET       = 1017   /* Set TTY auditing status */
AUDIT_SET_FEATURE   = 1018   /* Turn an audit feature on or off */
AUDIT_GET_FEATURE   = 1019   /* Get which features are enabled */
```

### Userspace messages

```
AUDIT_FIRST_USER_MSG   = 1100
AUDIT_USER_AVC         = 1107   /* filtered differently */
AUDIT_USER_TTY         = 1124   /* non-ICANON TTY input meaning */
AUDIT_LAST_USER_MSG    = 1199

AUDIT_FIRST_USER_MSG2  = 2100
AUDIT_LAST_USER_MSG2   = 2999
```

### Daemon lifecycle records (1200..1299, kernel → user)

```
AUDIT_DAEMON_START   = 1200   /* Daemon startup record */
AUDIT_DAEMON_END     = 1201   /* Daemon normal stop record */
AUDIT_DAEMON_ABORT   = 1202   /* Daemon error stop record */
AUDIT_DAEMON_CONFIG  = 1203   /* Daemon config change */
```

### Kernel event records (1300..1399)

```
AUDIT_SYSCALL          = 1300   /* Syscall event */
/* 1301 deprecated AUDIT_FS_WATCH */
AUDIT_PATH             = 1302   /* Filename / path information */
AUDIT_IPC              = 1303   /* IPC record */
AUDIT_SOCKETCALL       = 1304   /* sys_socketcall arguments */
AUDIT_CONFIG_CHANGE    = 1305   /* Audit system configuration change */
AUDIT_SOCKADDR         = 1306   /* sockaddr copied as syscall arg */
AUDIT_CWD              = 1307   /* Current working directory */
AUDIT_EXECVE           = 1309   /* execve arguments */
AUDIT_IPC_SET_PERM     = 1311   /* IPC new permissions record type */
AUDIT_MQ_OPEN          = 1312   /* POSIX MQ open record type */
AUDIT_MQ_SENDRECV      = 1313   /* POSIX MQ send/receive record type */
AUDIT_MQ_NOTIFY        = 1314   /* POSIX MQ notify record type */
AUDIT_MQ_GETSETATTR    = 1315   /* POSIX MQ get/set attribute record type */
AUDIT_KERNEL_OTHER     = 1316   /* For use by 3rd party modules */
AUDIT_FD_PAIR          = 1317   /* audit record for pipe/socketpair */
AUDIT_OBJ_PID          = 1318   /* ptrace target */
AUDIT_TTY              = 1319   /* Input on an administrative TTY */
AUDIT_EOE              = 1320   /* End of multi-record event */
AUDIT_BPRM_FCAPS       = 1321   /* Information about fcaps increasing perms */
AUDIT_CAPSET           = 1322   /* Record showing argument to sys_capset */
AUDIT_MMAP             = 1323   /* Record showing descriptor and flags in mmap */
AUDIT_NETFILTER_PKT    = 1324   /* Packets traversing netfilter chains */
AUDIT_NETFILTER_CFG    = 1325   /* Netfilter chain modifications */
AUDIT_SECCOMP          = 1326   /* Secure Computing event */
AUDIT_PROCTITLE        = 1327   /* Proctitle emit event */
AUDIT_FEATURE_CHANGE   = 1328   /* audit log listing feature changes */
AUDIT_REPLACE          = 1329   /* Replace auditd if this packet unanswered */
AUDIT_KERN_MODULE      = 1330   /* Kernel Module events */
AUDIT_FANOTIFY         = 1331   /* Fanotify access decision */
AUDIT_TIME_INJOFFSET   = 1332   /* Timekeeping offset injected */
AUDIT_TIME_ADJNTPVAL   = 1333   /* NTP value adjustment */
AUDIT_BPF              = 1334   /* BPF subsystem */
AUDIT_EVENT_LISTENER   = 1335   /* Task joined multicast read socket */
AUDIT_URINGOP          = 1336   /* io_uring operation */
AUDIT_OPENAT2          = 1337   /* Record showing openat2 how args */
AUDIT_DM_CTRL          = 1338   /* Device Mapper target control */
AUDIT_DM_EVENT         = 1339   /* Device Mapper events */
```

### LSM / access-control records (1400..1499)

```
AUDIT_AVC                = 1400   /* SELinux AVC denial or grant */
AUDIT_SELINUX_ERR        = 1401
AUDIT_AVC_PATH           = 1402
AUDIT_MAC_POLICY_LOAD    = 1403
AUDIT_MAC_STATUS         = 1404
AUDIT_MAC_CONFIG_CHANGE  = 1405
AUDIT_MAC_UNLBL_ALLOW    = 1406
AUDIT_MAC_CIPSOV4_ADD    = 1407
AUDIT_MAC_CIPSOV4_DEL    = 1408
AUDIT_MAC_MAP_ADD        = 1409
AUDIT_MAC_MAP_DEL        = 1410
AUDIT_MAC_IPSEC_ADDSA    = 1411
AUDIT_MAC_IPSEC_DELSA    = 1412
AUDIT_MAC_IPSEC_ADDSPD   = 1413
AUDIT_MAC_IPSEC_DELSPD   = 1414
AUDIT_MAC_IPSEC_EVENT    = 1415
AUDIT_MAC_UNLBL_STCADD   = 1416
AUDIT_MAC_UNLBL_STCDEL   = 1417
AUDIT_MAC_CALIPSO_ADD    = 1418
AUDIT_MAC_CALIPSO_DEL    = 1419
AUDIT_IPE_ACCESS         = 1420
AUDIT_IPE_CONFIG_CHANGE  = 1421
AUDIT_IPE_POLICY_LOAD    = 1422
AUDIT_LANDLOCK_ACCESS    = 1423
AUDIT_LANDLOCK_DOMAIN    = 1424
AUDIT_MAC_TASK_CONTEXTS  = 1425
AUDIT_MAC_OBJ_CONTEXTS   = 1426
```

### Anomaly / integrity ranges

```
AUDIT_FIRST_KERN_ANOM_MSG = 1700
AUDIT_LAST_KERN_ANOM_MSG  = 1799
AUDIT_ANOM_PROMISCUOUS    = 1700
AUDIT_ANOM_ABEND          = 1701
AUDIT_ANOM_LINK           = 1702
AUDIT_ANOM_CREAT          = 1703

AUDIT_INTEGRITY_DATA        = 1800
AUDIT_INTEGRITY_METADATA    = 1801
AUDIT_INTEGRITY_STATUS      = 1802
AUDIT_INTEGRITY_HASH        = 1803
AUDIT_INTEGRITY_PCR         = 1804
AUDIT_INTEGRITY_RULE        = 1805
AUDIT_INTEGRITY_EVM_XATTR   = 1806
AUDIT_INTEGRITY_POLICY_RULE = 1807
AUDIT_INTEGRITY_USERSPACE   = 1808

AUDIT_KERNEL = 2000   /* legacy async kernel record */
```

Note: the header does not define a single `AUDIT_FIRST_KERN_MSG` symbol; the kernel-emitted range begins at `AUDIT_DAEMON_START` (1200) and extends through 1999, with `AUDIT_FIRST_KERN_ANOM_MSG`..`AUDIT_LAST_KERN_ANOM_MSG` (1700..1799) being the only explicit "kern-msg" subrange-with-symbols.

### Filter / rule definitions

```
AUDIT_FILTER_USER       = 0x00   /* user-generated messages */
AUDIT_FILTER_TASK       = 0x01   /* at task creation */
AUDIT_FILTER_ENTRY      = 0x02   /* at syscall entry */
AUDIT_FILTER_WATCH      = 0x03   /* file-system watches */
AUDIT_FILTER_EXIT       = 0x04   /* at syscall exit */
AUDIT_FILTER_EXCLUDE    = 0x05   /* before record creation */
AUDIT_FILTER_TYPE       = AUDIT_FILTER_EXCLUDE  /* obsolete alias */
AUDIT_FILTER_FS         = 0x06   /* at __audit_inode_child */
AUDIT_FILTER_URING_EXIT = 0x07   /* at io_uring op exit */
#define AUDIT_NR_FILTERS  8

AUDIT_FILTER_PREPEND    = 0x10   /* prepend to front of list */
```

### Rule actions

```
AUDIT_NEVER     = 0   /* do not build context if rule matches */
AUDIT_POSSIBLE  = 1   /* build context if rule matches */
AUDIT_ALWAYS    = 2   /* generate audit record if rule matches */
```

### Rule limits & syscall classes

```
AUDIT_MAX_FIELDS    = 64
AUDIT_MAX_KEY_LEN   = 256
AUDIT_BITMASK_SIZE  = 64                  /* in u32 words → 2048 bits */
AUDIT_WORD(nr)      = (nr) / 32
AUDIT_BIT(nr)       = 1U << ((nr) - AUDIT_WORD(nr)*32)

AUDIT_SYSCALL_CLASSES = 16
AUDIT_CLASS_DIR_WRITE    = 0
AUDIT_CLASS_DIR_WRITE_32 = 1
AUDIT_CLASS_CHATTR       = 2
AUDIT_CLASS_CHATTR_32    = 3
AUDIT_CLASS_READ         = 4
AUDIT_CLASS_READ_32      = 5
AUDIT_CLASS_WRITE        = 6
AUDIT_CLASS_WRITE_32     = 7
AUDIT_CLASS_SIGNAL       = 8
AUDIT_CLASS_SIGNAL_32    = 9

AUDIT_UNUSED_BITS = 0x07FFFC00   /* validation mask for field ops */
```

### `AUDIT_FIELD_COMPARE` rule list

```
AUDIT_COMPARE_UID_TO_OBJ_UID   = 1
AUDIT_COMPARE_GID_TO_OBJ_GID   = 2
AUDIT_COMPARE_EUID_TO_OBJ_UID  = 3
AUDIT_COMPARE_EGID_TO_OBJ_GID  = 4
AUDIT_COMPARE_AUID_TO_OBJ_UID  = 5
AUDIT_COMPARE_SUID_TO_OBJ_UID  = 6
AUDIT_COMPARE_SGID_TO_OBJ_GID  = 7
AUDIT_COMPARE_FSUID_TO_OBJ_UID = 8
AUDIT_COMPARE_FSGID_TO_OBJ_GID = 9
AUDIT_COMPARE_UID_TO_AUID      = 10
AUDIT_COMPARE_UID_TO_EUID      = 11
AUDIT_COMPARE_UID_TO_FSUID     = 12
AUDIT_COMPARE_UID_TO_SUID      = 13
AUDIT_COMPARE_AUID_TO_FSUID    = 14
AUDIT_COMPARE_AUID_TO_SUID     = 15
AUDIT_COMPARE_AUID_TO_EUID     = 16
AUDIT_COMPARE_EUID_TO_SUID     = 17
AUDIT_COMPARE_EUID_TO_FSUID    = 18
AUDIT_COMPARE_SUID_TO_FSUID    = 19
AUDIT_COMPARE_GID_TO_EGID      = 20
AUDIT_COMPARE_GID_TO_FSGID     = 21
AUDIT_COMPARE_GID_TO_SGID      = 22
AUDIT_COMPARE_EGID_TO_FSGID    = 23
AUDIT_COMPARE_EGID_TO_SGID     = 24
AUDIT_COMPARE_SGID_TO_FSGID    = 25
AUDIT_MAX_FIELD_COMPARE        = AUDIT_COMPARE_SGID_TO_FSGID
```

### Rule fields (per-task and per-exit)

```
/* task-time fields */
AUDIT_PID        =  0
AUDIT_UID        =  1
AUDIT_EUID       =  2
AUDIT_SUID       =  3
AUDIT_FSUID      =  4
AUDIT_GID        =  5
AUDIT_EGID       =  6
AUDIT_SGID       =  7
AUDIT_FSGID      =  8
AUDIT_LOGINUID   =  9
AUDIT_PERS       = 10
AUDIT_ARCH       = 11
AUDIT_MSGTYPE    = 12
AUDIT_SUBJ_USER  = 13
AUDIT_SUBJ_ROLE  = 14
AUDIT_SUBJ_TYPE  = 15
AUDIT_SUBJ_SEN   = 16
AUDIT_SUBJ_CLR   = 17
AUDIT_PPID       = 18
AUDIT_OBJ_USER   = 19
AUDIT_OBJ_ROLE   = 20
AUDIT_OBJ_TYPE   = 21
AUDIT_OBJ_LEV_LOW  = 22
AUDIT_OBJ_LEV_HIGH = 23
AUDIT_LOGINUID_SET = 24
AUDIT_SESSIONID  = 25
AUDIT_FSTYPE     = 26

/* syscall-exit fields */
AUDIT_DEVMAJOR   = 100
AUDIT_DEVMINOR   = 101
AUDIT_INODE      = 102
AUDIT_EXIT       = 103
AUDIT_SUCCESS    = 104   /* exit >= 0; value ignored */
AUDIT_WATCH      = 105
AUDIT_PERM       = 106
AUDIT_DIR        = 107
AUDIT_FILETYPE   = 108
AUDIT_OBJ_UID    = 109
AUDIT_OBJ_GID    = 110
AUDIT_FIELD_COMPARE = 111
AUDIT_EXE        = 112
AUDIT_SADDR_FAM  = 113

AUDIT_ARG0       = 200
AUDIT_ARG1       = AUDIT_ARG0 + 1
AUDIT_ARG2       = AUDIT_ARG0 + 2
AUDIT_ARG3       = AUDIT_ARG0 + 3

AUDIT_FILTERKEY  = 210
```

### Operators (in `fieldflags[i]`)

```
AUDIT_NEGATE                = 0x80000000   /* invert sense (legacy) */
AUDIT_BIT_MASK              = 0x08000000   /* & */
AUDIT_LESS_THAN             = 0x10000000   /* < */
AUDIT_GREATER_THAN          = 0x20000000   /* > */
AUDIT_NOT_EQUAL             = 0x30000000   /* != */
AUDIT_EQUAL                 = 0x40000000   /* == */
AUDIT_BIT_TEST              = (AUDIT_BIT_MASK | AUDIT_EQUAL)         /* &= */
AUDIT_LESS_THAN_OR_EQUAL    = (AUDIT_LESS_THAN | AUDIT_EQUAL)        /* <= */
AUDIT_GREATER_THAN_OR_EQUAL = (AUDIT_GREATER_THAN | AUDIT_EQUAL)     /* >= */
AUDIT_OPERATORS             = (AUDIT_EQUAL | AUDIT_NOT_EQUAL | AUDIT_BIT_MASK)

/* internal lookup enum */
enum { Audit_equal, Audit_not_equal, Audit_bitmask, Audit_bittest,
       Audit_lt, Audit_gt, Audit_le, Audit_ge, Audit_bad };
```

### Status mask bits (`struct audit_status.mask`)

```
AUDIT_STATUS_ENABLED                  = 0x0001
AUDIT_STATUS_FAILURE                  = 0x0002
AUDIT_STATUS_PID                      = 0x0004
AUDIT_STATUS_RATE_LIMIT               = 0x0008
AUDIT_STATUS_BACKLOG_LIMIT            = 0x0010
AUDIT_STATUS_BACKLOG_WAIT_TIME        = 0x0020
AUDIT_STATUS_LOST                     = 0x0040
AUDIT_STATUS_BACKLOG_WAIT_TIME_ACTUAL = 0x0080
```

### Feature bitmap (`audit_status.feature_bitmap`, `audit_features.features`)

```
AUDIT_FEATURE_BITMAP_BACKLOG_LIMIT      = 0x00000001
AUDIT_FEATURE_BITMAP_BACKLOG_WAIT_TIME  = 0x00000002
AUDIT_FEATURE_BITMAP_EXECUTABLE_PATH    = 0x00000004
AUDIT_FEATURE_BITMAP_EXCLUDE_EXTEND     = 0x00000008
AUDIT_FEATURE_BITMAP_SESSIONID_FILTER   = 0x00000010
AUDIT_FEATURE_BITMAP_LOST_RESET         = 0x00000020
AUDIT_FEATURE_BITMAP_FILTER_FS          = 0x00000040
AUDIT_FEATURE_BITMAP_ALL                = (union of all above)

/* deprecated aliases */
AUDIT_VERSION_LATEST               = AUDIT_FEATURE_BITMAP_ALL
AUDIT_VERSION_BACKLOG_LIMIT        = AUDIT_FEATURE_BITMAP_BACKLOG_LIMIT
AUDIT_VERSION_BACKLOG_WAIT_TIME    = AUDIT_FEATURE_BITMAP_BACKLOG_WAIT_TIME

/* Failure-to-log actions */
AUDIT_FAIL_SILENT  = 0
AUDIT_FAIL_PRINTK  = 1
AUDIT_FAIL_PANIC   = 2
```

### Audit-arch markers (`AUDIT_ARCH` field value)

```
__AUDIT_ARCH_CONVENTION_MASK     = 0x30000000
__AUDIT_ARCH_CONVENTION_MIPS64_N32 = 0x20000000
__AUDIT_ARCH_64BIT               = 0x80000000
__AUDIT_ARCH_LE                  = 0x40000000

AUDIT_ARCH_AARCH64    = EM_AARCH64 | __AUDIT_ARCH_64BIT | __AUDIT_ARCH_LE
AUDIT_ARCH_ALPHA      = EM_ALPHA   | __AUDIT_ARCH_64BIT | __AUDIT_ARCH_LE
AUDIT_ARCH_ARCOMPACT  = EM_ARCOMPACT | __AUDIT_ARCH_LE
AUDIT_ARCH_ARCOMPACTBE= EM_ARCOMPACT
AUDIT_ARCH_ARCV2      = EM_ARCV2 | __AUDIT_ARCH_LE
AUDIT_ARCH_ARCV2BE    = EM_ARCV2
AUDIT_ARCH_ARM        = EM_ARM | __AUDIT_ARCH_LE
AUDIT_ARCH_ARMEB      = EM_ARM
AUDIT_ARCH_C6X        = EM_TI_C6000 | __AUDIT_ARCH_LE
AUDIT_ARCH_C6XBE      = EM_TI_C6000
AUDIT_ARCH_CRIS       = EM_CRIS | __AUDIT_ARCH_LE
AUDIT_ARCH_CSKY       = EM_CSKY | __AUDIT_ARCH_LE
AUDIT_ARCH_FRV        = EM_FRV
AUDIT_ARCH_H8300      = EM_H8_300
AUDIT_ARCH_HEXAGON    = EM_HEXAGON
AUDIT_ARCH_I386       = EM_386 | __AUDIT_ARCH_LE
AUDIT_ARCH_IA64       = EM_IA_64 | __AUDIT_ARCH_64BIT | __AUDIT_ARCH_LE
AUDIT_ARCH_M32R       = EM_M32R
AUDIT_ARCH_M68K       = EM_68K
AUDIT_ARCH_MICROBLAZE = EM_MICROBLAZE
AUDIT_ARCH_MIPS       = EM_MIPS
AUDIT_ARCH_MIPSEL     = EM_MIPS | __AUDIT_ARCH_LE
AUDIT_ARCH_MIPS64     = EM_MIPS | __AUDIT_ARCH_64BIT
AUDIT_ARCH_MIPS64N32  = EM_MIPS | __AUDIT_ARCH_64BIT | __AUDIT_ARCH_CONVENTION_MIPS64_N32
AUDIT_ARCH_MIPSEL64   = EM_MIPS | __AUDIT_ARCH_64BIT | __AUDIT_ARCH_LE
AUDIT_ARCH_MIPSEL64N32= EM_MIPS | __AUDIT_ARCH_64BIT | __AUDIT_ARCH_LE | __AUDIT_ARCH_CONVENTION_MIPS64_N32
AUDIT_ARCH_NDS32      = EM_NDS32 | __AUDIT_ARCH_LE
AUDIT_ARCH_NDS32BE    = EM_NDS32
AUDIT_ARCH_NIOS2      = EM_ALTERA_NIOS2 | __AUDIT_ARCH_LE
AUDIT_ARCH_OPENRISC   = EM_OPENRISC
AUDIT_ARCH_PARISC     = EM_PARISC
AUDIT_ARCH_PARISC64   = EM_PARISC | __AUDIT_ARCH_64BIT
AUDIT_ARCH_PPC        = EM_PPC
AUDIT_ARCH_PPC64      = EM_PPC64 | __AUDIT_ARCH_64BIT
AUDIT_ARCH_PPC64LE    = EM_PPC64 | __AUDIT_ARCH_64BIT | __AUDIT_ARCH_LE
AUDIT_ARCH_RISCV32    = EM_RISCV | __AUDIT_ARCH_LE
AUDIT_ARCH_RISCV64    = EM_RISCV | __AUDIT_ARCH_64BIT | __AUDIT_ARCH_LE
AUDIT_ARCH_S390       = EM_S390
AUDIT_ARCH_S390X      = EM_S390 | __AUDIT_ARCH_64BIT
AUDIT_ARCH_SH         = EM_SH
AUDIT_ARCH_SHEL       = EM_SH | __AUDIT_ARCH_LE
AUDIT_ARCH_SH64       = EM_SH | __AUDIT_ARCH_64BIT
AUDIT_ARCH_SHEL64     = EM_SH | __AUDIT_ARCH_64BIT | __AUDIT_ARCH_LE
AUDIT_ARCH_SPARC      = EM_SPARC
AUDIT_ARCH_SPARC64    = EM_SPARCV9 | __AUDIT_ARCH_64BIT
AUDIT_ARCH_TILEGX     = EM_TILEGX | __AUDIT_ARCH_64BIT | __AUDIT_ARCH_LE
AUDIT_ARCH_TILEGX32   = EM_TILEGX | __AUDIT_ARCH_LE
AUDIT_ARCH_TILEPRO    = EM_TILEPRO | __AUDIT_ARCH_LE
AUDIT_ARCH_UNICORE    = EM_UNICORE | __AUDIT_ARCH_LE
AUDIT_ARCH_X86_64     = EM_X86_64 | __AUDIT_ARCH_64BIT | __AUDIT_ARCH_LE
AUDIT_ARCH_XTENSA     = EM_XTENSA
AUDIT_ARCH_LOONGARCH32 = EM_LOONGARCH | __AUDIT_ARCH_LE
AUDIT_ARCH_LOONGARCH64 = EM_LOONGARCH | __AUDIT_ARCH_64BIT | __AUDIT_ARCH_LE
```

### Permission field bits (`AUDIT_PERM` value)

```
AUDIT_PERM_EXEC   = 1
AUDIT_PERM_WRITE  = 2
AUDIT_PERM_READ   = 4
AUDIT_PERM_ATTR   = 8
```

### Message size cap

```
AUDIT_MESSAGE_TEXT_MAX = 8560   /* max body bytes for a single record */
```

### Multicast netlink groups

```
enum audit_nlgrps {
  AUDIT_NLGRP_NONE,       /* group 0, unused */
  AUDIT_NLGRP_READLOG,    /* "best effort" read-only socket */
  __AUDIT_NLGRP_MAX
};
#define AUDIT_NLGRP_MAX  (__AUDIT_NLGRP_MAX - 1)
```

### `struct audit_status`

```
struct audit_status {
  __u32 mask;                       /* which fields are valid */
  __u32 enabled;                    /* 1 = enabled, 0 = disabled, 2 = locked */
  __u32 failure;                    /* AUDIT_FAIL_{SILENT,PRINTK,PANIC} */
  __u32 pid;                        /* pid of auditd process */
  __u32 rate_limit;                 /* messages/sec cap */
  __u32 backlog_limit;              /* queued-messages cap */
  __u32 lost;                       /* messages dropped to date */
  __u32 backlog;                    /* messages currently queued */
  union {
    __u32 version;                  /* deprecated */
    __u32 feature_bitmap;           /* AUDIT_FEATURE_BITMAP_* */
  };
  __u32 backlog_wait_time;          /* queue-full timeout */
  __u32 backlog_wait_time_actual;   /* time blocked because of cap */
};
```

### `struct audit_features`

```
struct audit_features {
  __u32 vers;        /* AUDIT_FEATURE_VERSION = 1 */
  __u32 mask;        /* which feature bits this msg touches */
  __u32 features;    /* desired feature value (0/1) per bit */
  __u32 lock;        /* mark feature immutable for life of kernel */
};

#define AUDIT_FEATURE_VERSION 1
#define AUDIT_FEATURE_ONLY_UNSET_LOGINUID 0
#define AUDIT_FEATURE_LOGINUID_IMMUTABLE  1
#define AUDIT_LAST_FEATURE                AUDIT_FEATURE_LOGINUID_IMMUTABLE
#define audit_feature_valid(x)   ((x) >= 0 && (x) <= AUDIT_LAST_FEATURE)
#define AUDIT_FEATURE_TO_MASK(x) (1 << ((x) & 31))
```

### `struct audit_tty_status`

```
struct audit_tty_status {
  __u32 enabled;     /* 1 = TTY input recorded */
  __u32 log_passwd;  /* 1 = record non-ICANON input too */
};
```

### Sentinels

```
AUDIT_UID_UNSET = (unsigned int)-1
AUDIT_SID_UNSET = (unsigned int)-1
```

### `struct audit_rule_data` (payload of `AUDIT_ADD_RULE` / `_DEL_RULE` / `_LIST_RULES`)

```
struct audit_rule_data {
  __u32 flags;        /* AUDIT_FILTER_* (low 4 bits) | AUDIT_FILTER_PREPEND */
  __u32 action;       /* AUDIT_NEVER / AUDIT_POSSIBLE / AUDIT_ALWAYS */
  __u32 field_count;  /* <= AUDIT_MAX_FIELDS */
  __u32 mask[AUDIT_BITMASK_SIZE];   /* bitmap of syscalls this rule applies to */
  __u32 fields[AUDIT_MAX_FIELDS];   /* AUDIT_PID..AUDIT_FILTERKEY */
  __u32 values[AUDIT_MAX_FIELDS];   /* integer values OR offset into buf[] for strings */
  __u32 fieldflags[AUDIT_MAX_FIELDS]; /* operator: AUDIT_EQUAL / NOT_EQUAL / ... */
  __u32 buflen;       /* total bytes in trailing buf[] */
  char  buf[];        /* concatenated string-field payloads */
};
```

## Compatibility contract

REQ-1 Netlink family + binding:
- Socket: `socket(AF_NETLINK, SOCK_RAW, NETLINK_AUDIT)`.
- Only one process may "claim" auditd via `AUDIT_SET` with `mask & AUDIT_STATUS_PID` and `pid = getpid()`; CAP_AUDIT_CONTROL required.
- Multicast read sockets join group `AUDIT_NLGRP_READLOG` (best-effort, no CAP_AUDIT_CONTROL needed; CAP_AUDIT_READ gates the join under default policy).

REQ-2 `AUDIT_GET` (1000):
- Request body empty (or zeroed `struct audit_status`).
- Response: kernel sends `AUDIT_GET` carrying populated `struct audit_status`.

REQ-3 `AUDIT_SET` (1001):
- Payload: `struct audit_status`.
- `mask` selects which fields to apply (`AUDIT_STATUS_*`).
- Requires `CAP_AUDIT_CONTROL`.
- `enabled = 2` ⇒ "locked": cannot be set back to 0/1 for life of kernel (immutable).

REQ-4 `AUDIT_LIST` / `AUDIT_ADD` / `AUDIT_DEL` (1002..1004) are the deprecated v1 rule API:
- Payload is older `struct audit_rule` (not `audit_rule_data`).
- Kept for backward-compatibility with audit-1.x userspace; modern userspace must use 1011..1013.

REQ-5 `AUDIT_USER` (1005) deprecated:
- Userspace single-line message echoed via netlink; modern userspace uses the 1100..1199 / 2100..2999 ranges.

REQ-6 `AUDIT_LOGIN` (1006):
- Payload: ASCII text — `audit_log_set_loginuid` syscall path; sets `task->loginuid` and `task->sessionid`.
- Requires `CAP_AUDIT_CONTROL` (or, with `AUDIT_FEATURE_ONLY_UNSET_LOGINUID`, also permitted if `loginuid == AUDIT_UID_UNSET`).

REQ-7 `AUDIT_WATCH_INS/REM/LIST` (1007..1009):
- Install/remove/list filesystem watches; correspond to inotify-backed audit watches.

REQ-8 `AUDIT_SIGNAL_INFO` (1010):
- Returns information about the last signal sent to auditd (sender uid, sender pid, SELinux context).

REQ-9 `AUDIT_ADD_RULE` / `_DEL_RULE` / `_LIST_RULES` (1011..1013):
- Payload: `struct audit_rule_data` followed by `buf[buflen]`.
- `flags`: low 4 bits = filter-list (`AUDIT_FILTER_*`); bit `AUDIT_FILTER_PREPEND` = head-insert.
- `action` ∈ `{AUDIT_NEVER, AUDIT_POSSIBLE, AUDIT_ALWAYS}`.
- `field_count` ≤ `AUDIT_MAX_FIELDS = 64`.
- `mask[]`: bitmap of syscalls (`AUDIT_WORD(nr)` / `AUDIT_BIT(nr)`); `AUDIT_BITMASK_SIZE = 64` u32 words = 2048 syscall bits.
- `fields[i]` ∈ `{AUDIT_PID, ..., AUDIT_FILTERKEY, AUDIT_ARG0..3}` + per-class field-ids.
- `values[i]`: integer match value, or offset into `buf[]` for string fields (PATH, EXE, DIR, WATCH, FILTERKEY, OBJ_USER/ROLE/TYPE, etc.).
- `fieldflags[i]`: operator (`AUDIT_EQUAL`, `AUDIT_NOT_EQUAL`, `AUDIT_GREATER_THAN`, etc.).
- `buflen` = total bytes in trailing `buf[]`.
- Requires `CAP_AUDIT_CONTROL`.

REQ-10 `AUDIT_TRIM` (1014) / `AUDIT_MAKE_EQUIV` (1015):
- Watched-tree maintenance ops; auditd-only.

REQ-11 `AUDIT_TTY_GET` / `AUDIT_TTY_SET` (1016..1017):
- Payload: `struct audit_tty_status`.
- `enabled = 1` ⇒ kernel emits `AUDIT_TTY` records for terminal input on admin TTYs.
- `log_passwd = 1` ⇒ even non-ICANON input recorded (defeats password masking).

REQ-12 `AUDIT_SET_FEATURE` / `_GET_FEATURE` (1018..1019):
- Payload: `struct audit_features { vers, mask, features, lock }`.
- `vers` must be `AUDIT_FEATURE_VERSION (1)`.
- `mask` selects which feature bits this message touches.
- `features` provides the desired 0/1 setting for each selected bit.
- `lock` bits ⇒ feature becomes immutable for life of kernel.

REQ-13 User-space message ranges:
- `AUDIT_FIRST_USER_MSG..AUDIT_LAST_USER_MSG` (1100..1199) — trusted userspace messages; CAP_AUDIT_WRITE required to send.
- `AUDIT_USER_AVC = 1107` filtered separately (LSM-AVC userspace correlate).
- `AUDIT_USER_TTY = 1124` — non-ICANON TTY input.
- `AUDIT_FIRST_USER_MSG2..AUDIT_LAST_USER_MSG2` (2100..2999) — additional userspace messages.

REQ-14 Daemon lifecycle records (1200..1299) kernel → user:
- `AUDIT_DAEMON_START / END / ABORT / CONFIG` — auditd lifecycle.

REQ-15 Kernel event records 1300..1399 (kernel → user via unicast to auditd / multicast to READLOG listeners):

- `AUDIT_SYSCALL (1300)`: syscall summary record carrying `arch=`, `syscall=`, `success=`, `exit=`, `a0=..a3=`, `items=`, `ppid=`, `pid=`, `auid=`, `uid=`, `gid=`, `euid=`, `suid=`, `fsuid=`, `egid=`, `sgid=`, `fsgid=`, `tty=`, `ses=`, `comm=`, `exe=`, `subj=`, `key=`.
- `AUDIT_PATH (1302)`: per-pathname record (one per path touched) — `item=`, `name=`, `inode=`, `dev=`, `mode=`, `ouid=`, `ogid=`, `rdev=`, `nametype=`.
- `AUDIT_IPC (1303)`: SysV IPC operation — `ouid=`, `ogid=`, `mode=`, `key=`.
- `AUDIT_IPC_SET_PERM (1311)`: SysV IPC permission change — `qbytes=`, `ouid=`, `ogid=`, `mode=`.
- `AUDIT_SOCKETCALL (1304)`: `sys_socketcall` args (i386 only).
- `AUDIT_CONFIG_CHANGE (1305)`: audit-subsystem state change (rules added/removed, status set).
- `AUDIT_SOCKADDR (1306)`: hex-encoded sockaddr captured at syscall time.
- `AUDIT_CWD (1307)`: caller's CWD at syscall time.
- `AUDIT_EXECVE (1309)`: argv vector — `argc=`, `a0=`, `a1=`, ...
- `AUDIT_MQ_OPEN (1312)` / `AUDIT_MQ_SENDRECV (1313)` / `AUDIT_MQ_NOTIFY (1314)` / `AUDIT_MQ_GETSETATTR (1315)`: POSIX message-queue ops.
- `AUDIT_KERNEL_OTHER (1316)`: catch-all for out-of-tree modules.
- `AUDIT_FD_PAIR (1317)`: pipe/socketpair return — `fd0=`, `fd1=`.
- `AUDIT_OBJ_PID (1318)`: ptrace target — `opid=`, `oauid=`, `ouid=`, `oses=`, `ocomm=`.
- `AUDIT_TTY (1319)`: admin-TTY input echo.
- `AUDIT_EOE (1320)`: zero-payload end-of-event marker for multi-record events.
- `AUDIT_BPRM_FCAPS (1321)`: file-caps usage on execve — `fver=`, `fp=`, `fi=`, `fe=`, `old_pp=`, `old_pi=`, `old_pe=`, `new_pp=`, `new_pi=`, `new_pe=`.
- `AUDIT_CAPSET (1322)`: `sys_capset` arg — `pid=`, `cap_pi=`, `cap_pp=`, `cap_pe=`.
- `AUDIT_MMAP (1323)`: `mmap` flags+fd — `fd=`, `flags=`.
- `AUDIT_NETFILTER_PKT (1324)` / `AUDIT_NETFILTER_CFG (1325)`: netfilter event/config.
- `AUDIT_SECCOMP (1326)`: seccomp event — `auid=`, `uid=`, `gid=`, `ses=`, `pid=`, `comm=`, `exe=`, `sig=`, `arch=`, `syscall=`, `compat=`, `ip=`, `code=`.
- `AUDIT_PROCTITLE (1327)`: opaque hex blob of process title.
- `AUDIT_FEATURE_CHANGE (1328)`: feature bit flipped via `AUDIT_SET_FEATURE`.
- `AUDIT_REPLACE (1329)`: signal to old auditd that a new one is taking over.
- `AUDIT_KERN_MODULE (1330)`: kmod load/unload — `name=`.
- `AUDIT_FANOTIFY (1331)`: fanotify access decision.
- `AUDIT_TIME_INJOFFSET (1332)` / `AUDIT_TIME_ADJNTPVAL (1333)`: clock adjustment events.
- `AUDIT_BPF (1334)`: BPF prog/map create/destroy.
- `AUDIT_EVENT_LISTENER (1335)`: task joined multicast read socket.
- `AUDIT_URINGOP (1336)`: io_uring op completion.
- `AUDIT_OPENAT2 (1337)`: `openat2` how-struct contents — `oflag=`, `mode=`, `resolve=`.
- `AUDIT_DM_CTRL (1338)` / `AUDIT_DM_EVENT (1339)`: Device Mapper.

REQ-16 LSM records (1400..1499):
- `AUDIT_AVC (1400)`: SELinux denial/grant — `denied=` / `granted=` plus `{ scontext, tcontext, tclass, ... }`.
- `AUDIT_SELINUX_ERR / AVC_PATH / MAC_POLICY_LOAD / MAC_STATUS / MAC_CONFIG_CHANGE`: SELinux policy/state.
- `AUDIT_MAC_UNLBL_ALLOW / CIPSOV4_ADD / DEL / MAP_ADD / DEL / IPSEC_* / UNLBL_STCADD / STCDEL / CALIPSO_ADD / DEL`: NetLabel/labelled-networking.
- `AUDIT_IPE_ACCESS / IPE_CONFIG_CHANGE / IPE_POLICY_LOAD`: Integrity Policy Enforcement.
- `AUDIT_LANDLOCK_ACCESS / DOMAIN`: Landlock denial / domain status.
- `AUDIT_MAC_TASK_CONTEXTS / OBJ_CONTEXTS`: multi-LSM stacking.

REQ-17 Anomaly records (1700..1799):
- `AUDIT_ANOM_PROMISCUOUS (1700)`: device entered promisc mode.
- `AUDIT_ANOM_ABEND (1701)`: process killed by signal.
- `AUDIT_ANOM_LINK (1702)`: suspicious symlink/hardlink op.
- `AUDIT_ANOM_CREAT (1703)`: suspicious file creation.

REQ-18 Integrity records (1800..1899):
- `AUDIT_INTEGRITY_DATA (1800)`: IMA data measurement.
- `AUDIT_INTEGRITY_METADATA (1801)`: IMA metadata measurement.
- `AUDIT_INTEGRITY_STATUS (1802)`: IMA enable state.
- `AUDIT_INTEGRITY_HASH (1803)`: hash kind change.
- `AUDIT_INTEGRITY_PCR (1804)`: TPM PCR invalidation.
- `AUDIT_INTEGRITY_RULE (1805)`: IMA policy rule load.
- `AUDIT_INTEGRITY_EVM_XATTR (1806)`: new EVM-covered xattr.
- `AUDIT_INTEGRITY_POLICY_RULE (1807)`: IMA policy rule diagnostic.
- `AUDIT_INTEGRITY_USERSPACE (1808)`: userspace-enforced data integrity.

REQ-19 `AUDIT_KERNEL (2000)` is the legacy async-record channel.

REQ-20 Filter lists (`flags & 0x07`):
- `AUDIT_FILTER_USER (0)`: applied to USER messages (1100..1199 / 2100..2999).
- `AUDIT_FILTER_TASK (1)`: applied at task creation (clone/fork).
- `AUDIT_FILTER_ENTRY (2)`: applied at syscall entry — **deprecated**, kernels return -EINVAL.
- `AUDIT_FILTER_WATCH (3)`: file-system watches.
- `AUDIT_FILTER_EXIT (4)`: applied at syscall exit (the normal case).
- `AUDIT_FILTER_EXCLUDE (5)`: applied **before** record creation; `AUDIT_FILTER_TYPE` is the obsolete alias.
- `AUDIT_FILTER_FS (6)`: applied at `__audit_inode_child`.
- `AUDIT_FILTER_URING_EXIT (7)`: applied at io_uring op exit.
- Total filter-list count `AUDIT_NR_FILTERS = 8`.

REQ-21 `AUDIT_FILTER_PREPEND (0x10)`:
- OR'd into `flags`; rule inserted at head of the filter list (highest priority).

REQ-22 `action`:
- `AUDIT_NEVER (0)`: matching syscall not recorded.
- `AUDIT_POSSIBLE (1)`: build audit context (allows later rules to record).
- `AUDIT_ALWAYS (2)`: matching syscall produces a full record set.

REQ-23 `mask[AUDIT_BITMASK_SIZE = 64]`:
- 64 u32 words = 2048 bits, indexed by syscall number; bit set ⇒ rule applies to that syscall.
- For `AUDIT_FILTER_TASK` / `_USER` / `_EXCLUDE`, `mask` is unused (still 64 words on the wire).

REQ-24 `field_count`:
- 0..`AUDIT_MAX_FIELDS = 64`; out-of-range ⇒ -EINVAL.

REQ-25 `fields[i]` ∈ field-id enum (`AUDIT_PID..AUDIT_FILETYPE..AUDIT_FIELD_COMPARE..AUDIT_EXE..AUDIT_SADDR_FAM` + `AUDIT_ARG0..3` + `AUDIT_FILTERKEY`):
- Task-time fields (0..26) apply to `_TASK`, `_USER`, `_EXCLUDE`.
- Syscall-exit fields (100..113) apply only to `_EXIT` / `_URING_EXIT` / `_FS` lists.
- `AUDIT_FIELD_COMPARE (111)` uses `values[i]` = `AUDIT_COMPARE_*` to specify which two task/object fields to compare.

REQ-26 `values[i]` semantics:
- Numeric for integer fields (PID, UID, ARCH, MSGTYPE, EXIT, FILETYPE, …).
- For string fields (PATH, EXE, DIR, WATCH, FILTERKEY, OBJ_USER/ROLE/TYPE, SUBJ_*): `values[i]` is **length** of the string and the string itself lives at the next offset in `buf[]`; userspace iterates with a running cursor.
- For `AUDIT_FIELD_COMPARE`: `values[i]` ∈ `AUDIT_COMPARE_*` (1..25).
- For `AUDIT_PERM`: `values[i]` ⊆ `{AUDIT_PERM_EXEC | _WRITE | _READ | _ATTR}`.

REQ-27 `fieldflags[i]` operator:
- Bits 28..31 carry op: `AUDIT_EQUAL` (0x40000000), `AUDIT_NOT_EQUAL` (0x30000000), `AUDIT_LESS_THAN` (0x10000000), `AUDIT_GREATER_THAN` (0x20000000), `AUDIT_BIT_MASK` (0x08000000), `AUDIT_BIT_TEST` (mask|equal), `AUDIT_LESS_THAN_OR_EQUAL` (lt|eq), `AUDIT_GREATER_THAN_OR_EQUAL` (gt|eq).
- `AUDIT_NEGATE (0x80000000)`: legacy invert flag (kernel still parses on input for back-compat).
- `AUDIT_OPERATORS = AUDIT_EQUAL | AUDIT_NOT_EQUAL | AUDIT_BIT_MASK`.
- `AUDIT_UNUSED_BITS = 0x07FFFC00` validates that only known bits are set.
- Internal lookup: `enum { Audit_equal, Audit_not_equal, Audit_bitmask, Audit_bittest, Audit_lt, Audit_gt, Audit_le, Audit_ge, Audit_bad }`.

REQ-28 `buflen` + `buf[]`:
- `buflen` = total bytes of trailing buffer.
- Each string-field entry contributes `values[i]` bytes (no NUL terminator).
- Sum of `values[i]` for string fields must equal `buflen`; mismatch ⇒ -EINVAL.

REQ-29 `struct audit_status`:
- `mask` is the bitmap of which fields the caller is setting (kernel ignores all other fields on `AUDIT_SET`).
- `enabled`: `0 = disabled`, `1 = enabled`, `2 = locked` (immutable for life of kernel).
- `failure`: `AUDIT_FAIL_SILENT (0)`, `AUDIT_FAIL_PRINTK (1)`, `AUDIT_FAIL_PANIC (2)` — kernel response when the queue overflows.
- `pid`: pid of the userspace auditd; `0` disconnects.
- `rate_limit`: per-second cap on outgoing messages.
- `backlog_limit`: queued-message cap.
- `lost`: messages dropped since last reset; `AUDIT_FEATURE_BITMAP_LOST_RESET` allows `AUDIT_SET` with `mask & AUDIT_STATUS_LOST` to zero this counter.
- `backlog`: current queue depth.
- `version` / `feature_bitmap`: union — `feature_bitmap` is the newer interpretation.
- `backlog_wait_time`: max time the kernel may block waiting for queue space (ns).
- `backlog_wait_time_actual`: cumulative time actually spent blocked.

REQ-30 `struct audit_features` lock semantics:
- Once `lock` bit set for a feature, `AUDIT_SET_FEATURE` against that feature ⇒ -EPERM.
- `AUDIT_FEATURE_LOGINUID_IMMUTABLE`: loginuid cannot be re-set after initial assignment.

REQ-31 `struct audit_tty_status`:
- Both fields are 0/1; other values ⇒ -EINVAL.
- Requires CAP_AUDIT_CONTROL.

REQ-32 Multicast READLOG group:
- `setsockopt(NETLINK_ADD_MEMBERSHIP, AUDIT_NLGRP_READLOG)`; under default policy requires `CAP_AUDIT_READ`.
- Best-effort: kernel may drop multicast messages without recording to `audit_status.lost`.

REQ-33 `AUDIT_MESSAGE_TEXT_MAX = 8560` bytes:
- Maximum body of a single record; longer records split via multi-record event with `AUDIT_EOE` sentinel.

REQ-34 Multi-record event ordering:
- A single syscall event produces 1× `AUDIT_SYSCALL` + 0..N `AUDIT_PATH` + 0..1 `AUDIT_CWD` + 0..1 `AUDIT_SOCKADDR` + 0..1 `AUDIT_EXECVE` + 0..1 `AUDIT_FD_PAIR` + 0..1 `AUDIT_PROCTITLE` + 0..1 `AUDIT_BPRM_FCAPS` + per-LSM AVC records, all with the same `audit(timestamp:serial)` ID; terminated by `AUDIT_EOE (1320)`.

REQ-35 `AUDIT_ARCH` field in `AUDIT_SYSCALL` record:
- The `AUDIT_ARCH_*` constant for the syscall's ABI; lets userspace cross-decode syscall numbers across 32/64 compat boundaries.
- Matches `seccomp_data.arch` exactly (shared ABI marker space).

REQ-36 Loginuid + sessionid semantics:
- `task->loginuid` is set once via `AUDIT_LOGIN` and (under `AUDIT_FEATURE_LOGINUID_IMMUTABLE`) cannot be re-set.
- `task->sessionid` allocated atomically when loginuid is set.
- Both inherited across fork/exec; preserved across setuid().
- `AUDIT_UID_UNSET = -1` is the initial loginuid value.

REQ-37 Capability gating summary:
- `CAP_AUDIT_CONTROL`: AUDIT_SET, AUDIT_ADD_RULE, AUDIT_DEL_RULE, AUDIT_LIST_RULES, AUDIT_TTY_SET, AUDIT_SET_FEATURE, AUDIT_LOGIN.
- `CAP_AUDIT_WRITE`: send `AUDIT_FIRST_USER_MSG..AUDIT_LAST_USER_MSG` / `AUDIT_FIRST_USER_MSG2..AUDIT_LAST_USER_MSG2`.
- `CAP_AUDIT_READ`: join multicast `AUDIT_NLGRP_READLOG`.

REQ-38 ABI stability:
- Message numbers are frozen; new event types append at higher numbers.
- `struct audit_rule_data`, `audit_status`, `audit_features`, `audit_tty_status` layouts frozen; extension only via `audit_status.feature_bitmap`.
- `AUDIT_FEATURE_BITMAP_*` reports availability of new features negotiated forward-compat.

## Acceptance Criteria

- [ ] AC-1: `socket(AF_NETLINK, SOCK_RAW, NETLINK_AUDIT)` succeeds.
- [ ] AC-2: `AUDIT_GET` from any user returns populated `struct audit_status` with `feature_bitmap` reflecting kernel.
- [ ] AC-3: `AUDIT_SET` without `CAP_AUDIT_CONTROL` ⇒ -EPERM.
- [ ] AC-4: `AUDIT_SET` with `mask=AUDIT_STATUS_PID, pid=getpid()` and CAP_AUDIT_CONTROL: claims auditd; second claimer with different pid kicked via `AUDIT_REPLACE`.
- [ ] AC-5: `AUDIT_SET` with `enabled=2` locks auditing; subsequent `AUDIT_SET enabled=0/1` ⇒ -EPERM.
- [ ] AC-6: `AUDIT_LIST/ADD/DEL` (deprecated v1) ⇒ kernel handles or returns -EOPNOTSUPP under modern config.
- [ ] AC-7: `AUDIT_LOGIN` with CAP_AUDIT_CONTROL sets `task.loginuid`; subsequent re-set with `LOGINUID_IMMUTABLE` ⇒ -EPERM.
- [ ] AC-8: `AUDIT_ADD_RULE` with `struct audit_rule_data { flags=AUDIT_FILTER_EXIT, action=AUDIT_ALWAYS, field_count=2, fields=[AUDIT_PID,AUDIT_EXIT], values=[1234,0], fieldflags=[AUDIT_EQUAL,AUDIT_NOT_EQUAL], mask[…syscall_nr…]=1 }` ⇒ rule installed.
- [ ] AC-9: `AUDIT_ADD_RULE` with `field_count > AUDIT_MAX_FIELDS (64)` ⇒ -EINVAL.
- [ ] AC-10: `AUDIT_ADD_RULE` with `flags=AUDIT_FILTER_ENTRY` ⇒ -EINVAL (deprecated).
- [ ] AC-11: `AUDIT_ADD_RULE` with `flags=AUDIT_FILTER_EXIT | AUDIT_FILTER_PREPEND` ⇒ rule inserted at head.
- [ ] AC-12: `AUDIT_LIST_RULES` returns one netlink message per rule with full `audit_rule_data` payload.
- [ ] AC-13: `AUDIT_DEL_RULE` matching an installed rule removes it.
- [ ] AC-14: `AUDIT_TTY_GET` returns current `audit_tty_status`.
- [ ] AC-15: `AUDIT_TTY_SET` with `enabled=1, log_passwd=0` enables TTY auditing.
- [ ] AC-16: `AUDIT_SET_FEATURE` with `vers != AUDIT_FEATURE_VERSION` ⇒ -EINVAL.
- [ ] AC-17: `AUDIT_SET_FEATURE` setting `lock=1` for `AUDIT_FEATURE_LOGINUID_IMMUTABLE` makes loginuid immutable; subsequent unlock attempts ⇒ -EPERM.
- [ ] AC-18: Sending message in `[AUDIT_FIRST_USER_MSG, AUDIT_LAST_USER_MSG]` without CAP_AUDIT_WRITE ⇒ -EPERM.
- [ ] AC-19: `AUDIT_USER_TTY (1124)` is filterable separately from generic USER messages.
- [ ] AC-20: Joining `AUDIT_NLGRP_READLOG` without CAP_AUDIT_READ ⇒ -EPERM.
- [ ] AC-21: A single syscall event emits `AUDIT_SYSCALL` + N `AUDIT_PATH` + optional `AUDIT_CWD/EXECVE/SOCKADDR/FD_PAIR/PROCTITLE/BPRM_FCAPS/MMAP/IPC/SECCOMP/...` + `AUDIT_EOE`, all sharing a serial.
- [ ] AC-22: `AUDIT_SECCOMP` record on `RET_LOG` seccomp action carries syscall=, arch=, ip=, code=.
- [ ] AC-23: `AUDIT_KERN_MODULE` emitted on `init_module` / `delete_module`.
- [ ] AC-24: `AUDIT_FANOTIFY` emitted on fanotify permission decision.
- [ ] AC-25: `AUDIT_BPF` emitted on bpf prog/map create/destroy.
- [ ] AC-26: `AUDIT_URINGOP` emitted on io_uring op completion when rules match `AUDIT_FILTER_URING_EXIT`.
- [ ] AC-27: `AUDIT_OPENAT2` carries the `struct open_how` fields (`oflag`, `mode`, `resolve`).
- [ ] AC-28: `AUDIT_AVC` emitted on SELinux/AppArmor denial with `denied=` field.
- [ ] AC-29: `AUDIT_LANDLOCK_ACCESS` emitted on Landlock denial.
- [ ] AC-30: `AUDIT_MAC_TASK_CONTEXTS` emitted when multiple LSMs stack and a task carries contexts from each.
- [ ] AC-31: Record body > `AUDIT_MESSAGE_TEXT_MAX (8560)` bytes split across multiple records sharing serial.
- [ ] AC-32: Queue overflow with `failure=AUDIT_FAIL_PANIC` panics kernel; with `_PRINTK` logs warning; with `_SILENT` drops silently and increments `lost`.
- [ ] AC-33: `AUDIT_STATUS_LOST` clears `lost` to zero when `AUDIT_FEATURE_BITMAP_LOST_RESET` is supported.
- [ ] AC-34: `AUDIT_SET` blocks for ≤ `backlog_wait_time` ns when queue full; otherwise drops or panics per `failure`.
- [ ] AC-35: `AUDIT_FIELD_COMPARE` rule with `values[i]=AUDIT_COMPARE_UID_TO_OBJ_UID` and `fieldflags[i]=AUDIT_EQUAL` matches iff `current_uid == file_owner_uid`.
- [ ] AC-36: `AUDIT_ARCH` field in SYSCALL record matches `AUDIT_ARCH_X86_64` on x86_64, `AUDIT_ARCH_AARCH64` on arm64, `AUDIT_ARCH_RISCV64` on riscv64, etc.
- [ ] AC-37: `AUDIT_PERM` with `values=AUDIT_PERM_EXEC|AUDIT_PERM_READ` matches files opened with exec or read intent.
- [ ] AC-38: `AUDIT_EXE` rule with string in `buf[]` matches exec'd binary path.
- [ ] AC-39: `AUDIT_FILTERKEY` populates `key=` field in resulting SYSCALL record.
- [ ] AC-40: `AUDIT_NEGATE` flag on operator inverts match semantics for back-compat.
- [ ] AC-41: `AUDIT_LIST_RULES` reply has one rule per netlink message (NLM_F_MULTI).

## Architecture

```
mod msg {
  pub const GET: u16          = 1000;
  pub const SET: u16          = 1001;
  pub const LIST: u16         = 1002;
  pub const ADD: u16          = 1003;
  pub const DEL: u16          = 1004;
  pub const USER: u16         = 1005;
  pub const LOGIN: u16        = 1006;
  pub const WATCH_INS: u16    = 1007;
  pub const WATCH_REM: u16    = 1008;
  pub const WATCH_LIST: u16   = 1009;
  pub const SIGNAL_INFO: u16  = 1010;
  pub const ADD_RULE: u16     = 1011;
  pub const DEL_RULE: u16     = 1012;
  pub const LIST_RULES: u16   = 1013;
  pub const TRIM: u16         = 1014;
  pub const MAKE_EQUIV: u16   = 1015;
  pub const TTY_GET: u16      = 1016;
  pub const TTY_SET: u16      = 1017;
  pub const SET_FEATURE: u16  = 1018;
  pub const GET_FEATURE: u16  = 1019;
}

#[repr(u8)]
enum FilterList {
  User = 0, Task = 1, Entry = 2, Watch = 3, Exit = 4,
  Exclude = 5, Fs = 6, UringExit = 7,
}
const NR_FILTERS: usize = 8;
const FILTER_PREPEND: u32 = 0x10;

#[repr(u32)]
enum Action { Never = 0, Possible = 1, Always = 2 }

const MAX_FIELDS: usize   = 64;
const MAX_KEY_LEN: usize  = 256;
const BITMASK_SIZE: usize = 64;          // u32 words → 2048 bits
const MESSAGE_TEXT_MAX: usize = 8560;

mod field {
  pub const PID: u32 = 0;  pub const UID: u32 = 1;  pub const EUID: u32 = 2;
  pub const SUID: u32 = 3; pub const FSUID: u32 = 4; pub const GID: u32 = 5;
  pub const EGID: u32 = 6; pub const SGID: u32 = 7; pub const FSGID: u32 = 8;
  pub const LOGINUID: u32 = 9; pub const PERS: u32 = 10; pub const ARCH: u32 = 11;
  pub const MSGTYPE: u32 = 12; /* ...; SUBJ_USER..OBJ_LEV_HIGH..FSTYPE */
  pub const DEVMAJOR: u32 = 100; /* ...DEVMINOR..INODE..EXIT..SUCCESS..WATCH..PERM..DIR
                                    ..FILETYPE..OBJ_UID..OBJ_GID..FIELD_COMPARE..EXE..SADDR_FAM */
  pub const ARG0: u32 = 200;  pub const ARG1: u32 = 201;
  pub const ARG2: u32 = 202;  pub const ARG3: u32 = 203;
  pub const FILTERKEY: u32 = 210;
}

bitflags! Op : u32 {
  NEGATE                = 0x80000000,
  BIT_MASK              = 0x08000000,
  LESS_THAN             = 0x10000000,
  GREATER_THAN          = 0x20000000,
  NOT_EQUAL             = 0x30000000,
  EQUAL                 = 0x40000000,
  BIT_TEST              = 0x48000000,
  LESS_THAN_OR_EQUAL    = 0x50000000,
  GREATER_THAN_OR_EQUAL = 0x60000000,
}

#[repr(C)]
struct AuditRuleData {
  flags: u32,
  action: u32,
  field_count: u32,
  mask: [u32; BITMASK_SIZE],
  fields: [u32; MAX_FIELDS],
  values: [u32; MAX_FIELDS],
  fieldflags: [u32; MAX_FIELDS],
  buflen: u32,
  buf: [u8; 0],                   // trailing flex
}

#[repr(C)]
struct AuditStatus {
  mask: u32,
  enabled: u32,
  failure: u32,
  pid: u32,
  rate_limit: u32,
  backlog_limit: u32,
  lost: u32,
  backlog: u32,
  version_or_feature_bitmap: u32, // union
  backlog_wait_time: u32,
  backlog_wait_time_actual: u32,
}

#[repr(C)]
struct AuditFeatures {
  vers: u32,                      // AUDIT_FEATURE_VERSION = 1
  mask: u32,
  features: u32,
  lock: u32,
}

#[repr(C)]
struct AuditTtyStatus { enabled: u32, log_passwd: u32 }
```

`Audit::netlink_dispatch(nlh)`:
1. Match `nlh.type`:
   - 1000 GET ⇒ reply with `AuditStatus`.
   - 1001 SET ⇒ require CAP_AUDIT_CONTROL; parse `AuditStatus`; apply per `mask`.
   - 1002..1004 v1 rules (deprecated) ⇒ -EOPNOTSUPP or handled via shim.
   - 1005 USER ⇒ require CAP_AUDIT_WRITE; relay as USER record.
   - 1006 LOGIN ⇒ require CAP_AUDIT_CONTROL; set task.loginuid (immutable if locked).
   - 1007..1009 WATCH_* ⇒ require CAP_AUDIT_CONTROL.
   - 1010 SIGNAL_INFO ⇒ return last-signal-to-auditd info.
   - 1011 ADD_RULE / 1012 DEL_RULE / 1013 LIST_RULES ⇒ `Audit::rule_cmd`.
   - 1014 TRIM / 1015 MAKE_EQUIV ⇒ tree maintenance.
   - 1016 TTY_GET / 1017 TTY_SET ⇒ `AuditTtyStatus` apply.
   - 1018 SET_FEATURE / 1019 GET_FEATURE ⇒ `Audit::feature_cmd`.
   - 1100..1199 USER1 / 2100..2999 USER2 ⇒ require CAP_AUDIT_WRITE; emit USER record.
   - else ⇒ -EINVAL.

`Audit::rule_cmd(op, data, buflen, buf)`:
1. Require CAP_AUDIT_CONTROL.
2. Validate `data.flags & 0x07 < NR_FILTERS`.
3. Validate `data.flags & 0x07 != AUDIT_FILTER_ENTRY` (deprecated).
4. Validate `data.action ∈ {NEVER, POSSIBLE, ALWAYS}`.
5. Validate `data.field_count ≤ MAX_FIELDS`.
6. For each i in 0..field_count:
   - Validate `data.fields[i]` is a known field id.
   - Validate `data.fieldflags[i] & ~(OPERATORS | EXTRA_BITS) == 0`; `AUDIT_UNUSED_BITS` must be 0.
   - If string-field: cursor accounting over `buf[buflen]`.
7. Compose `AuditRule`; insert into appropriate filter list (head if `FILTER_PREPEND`).

`Audit::feature_cmd(features)`:
1. Require CAP_AUDIT_CONTROL.
2. Validate `features.vers == AUDIT_FEATURE_VERSION`.
3. For each bit in `features.mask`:
   - If feature locked ⇒ -EPERM.
   - Apply `features.features` for that bit.
   - If `features.lock & bit` ⇒ mark feature immutable for life of kernel.
4. Emit `AUDIT_FEATURE_CHANGE` record.

`Audit::emit_record(type, fields)`:
1. Allocate netlink message; populate header with `audit(timestamp:serial)` ID.
2. Format key=value body.
3. If body > MESSAGE_TEXT_MAX, split across multiple messages sharing serial.
4. Match filter chain against record:
   - For SYSCALL records: walk `FILTER_EXIT` then `FILTER_EXCLUDE`.
   - For TASK records: walk `FILTER_TASK`.
   - For USER records: walk `FILTER_USER`.
   - For FS records: walk `FILTER_FS`.
   - For URING ops: walk `FILTER_URING_EXIT`.
5. If `action == AUDIT_NEVER` matches: drop.
6. If `action == AUDIT_ALWAYS` matches: emit.
7. If queue full: apply `audit_status.failure` policy (SILENT/PRINTK/PANIC); increment `lost`.
8. Else: enqueue + signal auditd; also multicast to READLOG group.

`Audit::user_send(type, msg)`:
1. Require CAP_AUDIT_WRITE if `type ∈ [FIRST_USER_MSG, LAST_USER_MSG2]`.
2. Validate len ≤ MESSAGE_TEXT_MAX.
3. Walk `FILTER_USER` chain; emit if no NEVER match.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_audit_control_required` | INVARIANT | per-SET / ADD_RULE / DEL_RULE / LOGIN / TTY_SET / SET_FEATURE: caller has CAP_AUDIT_CONTROL else -EPERM. |
| `cap_audit_write_required` | INVARIANT | per-USER1 / USER2 send: caller has CAP_AUDIT_WRITE else -EPERM. |
| `cap_audit_read_required` | INVARIANT | per-AUDIT_NLGRP_READLOG join: caller has CAP_AUDIT_READ else -EPERM. |
| `field_count_bound` | INVARIANT | per-ADD_RULE: field_count ≤ AUDIT_MAX_FIELDS. |
| `bitmask_size_fixed` | INVARIANT | per-rule: mask is exactly AUDIT_BITMASK_SIZE u32 words. |
| `filter_list_in_range` | INVARIANT | per-rule: flags & 0x07 < AUDIT_NR_FILTERS. |
| `filter_entry_rejected` | INVARIANT | per-ADD_RULE: AUDIT_FILTER_ENTRY ⇒ -EINVAL. |
| `action_in_range` | INVARIANT | per-rule: action ∈ {NEVER, POSSIBLE, ALWAYS}. |
| `op_mask_legal` | INVARIANT | per-rule: fieldflags[i] & AUDIT_UNUSED_BITS == 0; op bits ⊆ legal operators. |
| `buflen_accounting` | INVARIANT | per-rule: Σ string-field lens == buflen; offsets stay within buf[buflen]. |
| `feature_version_check` | INVARIANT | per-SET_FEATURE: vers == AUDIT_FEATURE_VERSION. |
| `feature_lock_immutable` | INVARIANT | per-SET_FEATURE: bit locked ⇒ subsequent change ⇒ -EPERM. |
| `enabled_locked_immutable` | INVARIANT | per-SET: enabled = 2 ⇒ subsequent enabled change ⇒ -EPERM. |
| `loginuid_immutable_when_locked` | INVARIANT | per-LOGIN: LOGINUID_IMMUTABLE ⇒ re-set ⇒ -EPERM. |
| `message_text_max_bound` | INVARIANT | per-record: body ≤ MESSAGE_TEXT_MAX or split into multi-record event. |
| `eoe_terminates_multi_record` | INVARIANT | per-multi-record event: final record type == AUDIT_EOE. |
| `failure_policy_obeyed` | INVARIANT | per-queue-overflow: behavior matches status.failure (silent/printk/panic). |

### Layer 2: TLA+

`uapi/audit.tla`:
- Models per-task loginuid/sessionid lifecycle, per-filter-list rule sets, queue + backlog, feature locks, auditd lifecycle.
- Properties:
  - `safety_one_auditd` — at most one task is the active auditd at any time.
  - `safety_enabled_locked_monotonic` — once `enabled = 2`, no transition out.
  - `safety_feature_lock_monotonic` — once a feature is locked, it stays.
  - `safety_loginuid_immutable_under_lock` — under `LOGINUID_IMMUTABLE`, task.loginuid set at most once after `AUDIT_UID_UNSET`.
  - `safety_multi_record_serial` — all records of a single syscall event share the same serial, terminated by `AUDIT_EOE`.
  - `safety_filter_list_dispatch_correct` — SYSCALL hits FILTER_EXIT/EXCLUDE; TASK hits FILTER_TASK; USER hits FILTER_USER; URING hits FILTER_URING_EXIT.
  - `safety_queue_full_policy` — overflow → behaviour matches `audit_status.failure`.
  - `liveness_record_eventually_emitted_or_lost` — every triggering event eventually becomes a netlink message or increments `lost`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Audit::netlink_dispatch` post: known message types accepted with proper capability check | `Audit::netlink_dispatch` |
| `Audit::rule_cmd(ADD)` post: rule installed in correct filter list at correct position (head if PREPEND, tail else) | `Audit::rule_cmd` |
| `Audit::rule_cmd(DEL)` post: matching rule removed | `Audit::rule_cmd` |
| `Audit::feature_cmd(SET, lock=1)` post: feature.value applied ∧ feature.locked = true | `Audit::feature_cmd` |
| `Audit::emit_record(type, fields)` post: well-formed key=value body ∧ size ≤ MESSAGE_TEXT_MAX ∨ split | `Audit::emit_record` |
| `Audit::filter_match(rule, ctx)` post: result iff all fields satisfy operators | `Audit::filter_match` |
| `Audit::user_send(type, msg)` post: CAP_AUDIT_WRITE held ∧ msg ≤ MESSAGE_TEXT_MAX ∧ FILTER_USER did not match NEVER | `Audit::user_send` |
| `Audit::loginuid_set` post: task.loginuid = new ∧ task.sessionid allocated; rejected if immutable+set | `Audit::loginuid_set` |
| `Audit::tty_set` post: enabled/log_passwd ∈ {0,1} | `Audit::tty_set` |

### Layer 4: Verus/Creusot functional

Per `Documentation/admin-guide/LSM/index.rst`, `kernel/audit.c`, `kernel/auditfilter.c`, `kernel/auditsc.c`:
- Netlink dispatch semantic equivalence with upstream.
- Filter-list dispatch equivalence with `audit_filter_*` paths.
- Record formatting equivalence with `audit_log_format` family.
- Queue + backlog accounting equivalence with `audit_backlog_*`.
- Multi-record event ordering equivalence with `audit_log_end` and `AUDIT_EOE` emission.
- `AUDIT_FIELD_COMPARE` semantic equivalence with `audit_field_compare`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

audit UAPI reinforcement:

- **Per-`CAP_AUDIT_CONTROL` gate** — defense against per-rule-tampering by unprivileged.
- **Per-`CAP_AUDIT_WRITE` gate** — defense against per-USER-msg spoofing.
- **Per-`CAP_AUDIT_READ` gate** — defense against per-multicast-read leak.
- **Per-`audit_status.enabled = 2` immutable** — defense against per-runtime-disable.
- **Per-`AUDIT_FEATURE_LOGINUID_IMMUTABLE`** — defense against per-loginuid-rewrite.
- **Per-feature lock monotonic** — defense against per-feature-relax.
- **Per-`AUDIT_MAX_FIELDS = 64` bound** — defense against per-quadratic-rule.
- **Per-`AUDIT_BITMASK_SIZE = 64` fixed** — defense against per-mask-overrun.
- **Per-`AUDIT_FILTER_ENTRY` rejected** — defense against per-syscall-entry race.
- **Per-`AUDIT_MESSAGE_TEXT_MAX = 8560` bound** — defense against per-OOM via huge record.
- **Per-multi-record `AUDIT_EOE` terminator** — defense against per-truncated-event interpretation.
- **Per-`audit_status.failure = PANIC`** — defense against per-policy-violation silent-loss.
- **Per-`backlog_limit` + `backlog_wait_time`** — defense against per-syscall-stalls-forever.
- **Per-rule field-id whitelist** — defense against per-future-bit injection.
- **Per-rule operator whitelist (`AUDIT_UNUSED_BITS = 0`)** — defense against per-unknown-op confusion.
- **Per-rule `buflen` strict accounting** — defense against per-overread.
- **Per-`AUDIT_REPLACE` on auditd takeover** — defense against per-silent-handoff.
- **Per-`AUDIT_LOGIN` audited** — defense against per-loginuid-injection.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF/USERCOPY on `audit_buf`** — every `copy_from_user` for `struct audit_status`, `audit_features`, `audit_tty_status`, `audit_rule_data` + trailing `buf[buflen]`, and USER messages uses hardened-usercopy with explicit size bounds (`sizeof(struct …)` for the headers, `buflen` for the trailing buffer); SMAP/SMEP active; no copies issued before all length fields have been parsed and validated against `AUDIT_MAX_FIELDS`, `AUDIT_BITMASK_SIZE`, and `AUDIT_MESSAGE_TEXT_MAX = 8560`.
- **GRKERNSEC_AUDIT_GROUP** — sensitive operations (capability transitions, setuid binaries running, mount/umount, module load, kernel-symbol resolution by unprivileged) generate `AUDIT_ALWAYS` records under a grsec-installed default ruleset that cannot be removed by a non-CAP_SYS_ADMIN userspace; the rules are reinstalled at boot from the grsec policy blob; the audit subsystem refuses `AUDIT_DEL_RULE` against grsec-owned rules with -EPERM.
- **MAX_AUDIT_MESSAGE_LENGTH cap** — record body ≤ `AUDIT_MESSAGE_TEXT_MAX = 8560` strictly enforced; oversized records split into multi-record events with shared serial and `AUDIT_EOE` terminator; under grsec the per-process per-second record cap (rate_limit) cannot be raised past a grsec-policy max regardless of `AUDIT_SET` value.
- **Audit-rate-limit enforced** — `audit_status.rate_limit` cannot be set to 0 (unlimited) by a non-CAP_SYS_ADMIN auditd under grsec; the kernel-internal rate limiter for `printk` warnings about queue overflow uses `printk_ratelimit()` to avoid log flood when `failure = AUDIT_FAIL_PRINTK`.
- **No audit-bypass for non-CAP_AUDIT_CONTROL** — `AUDIT_DEL_RULE` of any rule (grsec-owned or auditd-owned) requires `CAP_AUDIT_CONTROL`; `AUDIT_SET` of `enabled = 0` requires CAP_AUDIT_CONTROL and is blocked under grsec when `enabled = 2` (locked); userspace cannot suppress AVC/SELINUX records by adding `AUDIT_NEVER` rules against them under grsec policy (those records bypass the user-filter chain).
- **GRKERNSEC_HIDESYM** — `AUDIT_SYSCALL.a0..a3` and `AUDIT_EXECVE.aN` arguments are passed through grsec's kernel-pointer-sanitization filter for unprivileged consumers; `AUDIT_KERN_MODULE.name=` is the human module name only (never the kallsyms address); `AUDIT_BPF` records report prog id but not prog tag for unprivileged listeners.
- **PAX_RANDKSTACK** — the netlink recvmsg path that parses `audit_rule_data` re-randomizes the kernel-stack base before allocating the (up-to-4 KB) on-stack scratch for rule validation, so repeated rule-add probes cannot leak stack-layout via partial-failure timing.
- **GRKERNSEC_HARDEN_PTRACE** — cross-uid `AUDIT_LOGIN` (setting another task's loginuid) requires CAP_AUDIT_CONTROL **and** that the caller's user-ns dominate the target's; defense against per-cross-namespace loginuid forge.
- **PAX_KERNEXEC** — the audit-record formatting path uses `vsnprintf`-style format-string assembly only against compile-time-known format strings (the user-supplied USER message body is passed as a single `%s` substituent, never used as format); defense against per-format-string injection.
- **Per-`AUDIT_FILTER_PREPEND` audited under grsec** — head-of-list rule insertion is logged as `AUDIT_CONFIG_CHANGE` so an attacker who briefly held CAP_AUDIT_CONTROL cannot silently inject NEVER rules at the head to suppress later records.
- **Per-`AUDIT_TTY_SET.log_passwd = 1` audited** — non-ICANON TTY input recording (defeats password masking) generates a high-severity `AUDIT_CONFIG_CHANGE` record under grsec; defense against per-silent password-capture rule.
- **Per-`AUDIT_USER_TTY (1124)`** — filtered separately from generic USER messages; under grsec, sending USER_TTY records requires CAP_AUDIT_WRITE + CAP_SYS_TTY_CONFIG to prevent spoofing.
- **Per-`AUDIT_SECCOMP` mandatory** — under grsec policy, `RET_LOG` seccomp actions cannot be suppressed by user-supplied EXCLUDE rules; the audit subsystem bypasses the EXCLUDE filter chain for SECCOMP records.
- **Per-`AUDIT_BPRM_FCAPS`** — fcap-elevated execve always emits a record under grsec (any execve that increases caps beyond the inherited set), regardless of installed rules; defense against per-fcap-stealth-elevation.
- **Per-`AUDIT_CAPSET`** — every `capset(2)` that changes the calling task's caps emits a record under grsec; defense against per-cap-juggling without audit trail.
- **Per-`AUDIT_REPLACE`** — auditd takeover signal is itself audited with the old/new pids and the killing signal; defense against per-stealth-auditd-hijack.
- **Per-`AUDIT_NLGRP_READLOG`** — multicast listeners cannot send (read-only socket); under grsec, joining the group requires CAP_AUDIT_READ and the listener's user-ns must equal the kernel's init user-ns (no cross-userns log peeking).
- **Per-`AUDIT_FEATURE_BITMAP_FILTER_FS = 0x00000040`** — when locked, `AUDIT_FILTER_FS` rules cannot be added/removed; defense against per-runtime-fs-watch-tamper.
- **Per-`AUDIT_FILTER_EXCLUDE` `AUDIT_NEVER` for high-severity types refused** — under grsec policy, EXCLUDE rules cannot suppress `AUDIT_AVC`, `AUDIT_MAC_*`, `AUDIT_ANOM_*`, `AUDIT_INTEGRITY_*`, `AUDIT_LANDLOCK_*`, `AUDIT_IPE_*`, `AUDIT_SECCOMP`, or `AUDIT_BPRM_FCAPS`; defense against per-LSM-event-blindspot creation.
- **Per-syscall `audit_context` lifetime** — `audit_context` allocated at syscall entry and freed at exit; under grsec, the context is allocated from a slab cache with redzones (slub_debug) to catch UAF / overflow on the per-task scratch.
- **Per-`AUDIT_MAKE_EQUIV` / `_TRIM`** — tree-maintenance ops audited; cross-mount-namespace equiv refused.
- **Per-`AUDIT_TIME_INJOFFSET` / `_ADJNTPVAL`** — always emitted (never suppressed) under grsec; clock-jump is a forensic-critical event.
- **Per-`audit_status.lost` reset gating** — `AUDIT_FEATURE_BITMAP_LOST_RESET` honored only for CAP_AUDIT_CONTROL with `lock = 0`; under grsec, the `lost` counter is monotonic-increasing-only.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- Netlink socket family / message framing — covered in `uapi/headers/netlink.md`.
- LSM-specific record content semantics (SELinux AVC, AppArmor, Landlock domain semantics) — covered in `uapi/headers/lsm.md` and per-LSM Tier-3 docs.
- `loginuid` setting via `/proc/self/loginuid` filesystem interface — covered in `uapi/headers/procfs.md`.
- IMA / EVM policy-rule grammar — covered in `uapi/headers/ima.md`.
- Audit ↔ ftrace / perf integration — covered in `uapi/headers/tracefs.md`.
- `AUDIT_ARCH_*` ↔ `EM_*` mapping in detail — covered in `uapi/headers/elf.md`.
- Implementation code (`kernel/audit.c`, `kernel/auditfilter.c`, `kernel/auditsc.c`, `kernel/audit_tree.c`, `kernel/audit_watch.c`).
- Userspace auditd (`auditd(8)`, `auditctl(8)`, `ausearch(8)`) — out of kernel scope.
