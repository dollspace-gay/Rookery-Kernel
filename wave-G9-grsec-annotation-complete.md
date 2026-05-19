---
title: "Wave-G9 grsec annotation complete (594 docs across G1-G9)"
tags: ["grsec", "phase-d"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---

Wave-G9 commit 1c234d2: final grsec annotation pass across 124 Tier-3 docs covering wireless (wireguard, cfg80211, xsk), IPsec (xfrm 16 docs), security (apparmor, lsm-audit, security-core, selinux), sound (control, init, pcm, rawmidi, seq, timer), mount syscalls (9), and virt/kvm (87 docs: core + x86 features for PMU, PV, SVM, TDX/SEV, VMX, MMU). Campaign totals (waves G1-G9): 594 Tier-3 docs received standardized 9-baseline + 5-10-specific + rationale section. Only excluded: references/grsec-pax-notes.md (the source ref). Template: PAX_USERCOPY/KERNEXEC/RANDKSTACK/REFCOUNT/MEMORY_SANITIZE/UDEREF/RAP/kCFI + GRKERNSEC_HIDESYM/DMESG. Next: long-tail driver-by-driver enumeration phase.
