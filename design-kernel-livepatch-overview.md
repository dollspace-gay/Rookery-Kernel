---
title: "Tier-2: kernel/livepatch — kernel live-patching (klp + state machine + transition + shadow + patch)"
tags: ["tier-2", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 wrapper for kernel live-patching — apply security/bug-fix patches to a running kernel without reboot. Backbone of kpatch (RHEL/CentOS) + kGraft (SUSE) + kernel-live-patching commercial offerings. Components: `core.c` (subsystem init + klp_register_patch + per-patch lifecycle), `patch.c` (per-function patch — replace function in ftrace's hash + redirect callers via fentry hook), `state.c` (per-patch state machine + per-task consistency), `transition.c` (per-task transition: gradually mark each task as having reached a safe stack-state then switch over to patched functions; uses ORC unwinder to verify task isn't currently inside any patched function), `shadow.c` (shadow variables — extend existing kernel structs with per-instance fields without changing struct layout, used by patches that need new state per existing object), `core.h`, `patch.h`, `state.h`.

Live patches loaded as kernel modules with klp annotations; klp framework verifies module signature + applies transition.

### Out of Scope

- Implementation code; 32-bit-only paths

### compatibility contract — outline

- `/sys/kernel/livepatch/<patch>/{enabled, transition, force, signal, ...}` byte-identical (kpatch + klp-tools consume).
- Per-patch sysfs subdirs for objects + functions byte-identical.
- klp module annotation API (`KLP_REPLACE`, `klp_func`, `klp_object`, `klp_patch`) source-compat.
- `/proc/<pid>/patch_state` byte-identical (per-task transition state visible).

### tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `kernel/livepatch/core.md` | `core.c`: subsystem init + per-patch lifecycle |
| `kernel/livepatch/patch.md` | `patch.c`: per-function patch via ftrace |
| `kernel/livepatch/state.md` | `state.c`: per-patch state machine |
| `kernel/livepatch/transition.md` | `transition.c`: per-task transition + ORC stack-check |
| `kernel/livepatch/shadow.md` | `shadow.c`: shadow variables |

### compatibility outline / ac / verification / hardening

- REQ-O1: `/sys/kernel/livepatch/*` UAPI byte-identical (kpatch consumes).
- REQ-O2: klp module annotation API source-compat.
- REQ-O3: TLA+ models (per-task transition + ORC unwinder safe-stack-check; concurrent patch-load + task-transition; force-unpatch convergence).
- REQ-O4: AC: kpatch-build + klp-load test patch on reference kernel; transition completes + patched function called.
- Hardening: klp module load requires CAP_SYS_MODULE + module-signature-verification mandatory (CONFIG_LIVEPATCH requires CONFIG_MODULE_SIG_FORCE); per-patch sysfs writes require CAP_SYS_ADMIN.

