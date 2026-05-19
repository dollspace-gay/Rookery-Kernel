# Tier-5 UAPI: include/uapi/linux/kvm.h — KVM userspace ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/kvm.h (~1672 lines)
  - arch/x86/include/uapi/asm/kvm.h (KVM_IRQCHIP_*, struct kvm_msrs, struct kvm_msr_list, struct kvm_cpuid2, KVM_X86_QUIRK_*, KVM_GUESTDBG_*, KVM_X86_DEFAULT_VM / _SW_PROTECTED_VM / _SEV_VM / _SEV_ES_VM / _TDX_VM)
  - arch/x86/include/uapi/asm/kvm_para.h (KVM_FEATURE_*)
-->

## Summary

`/dev/kvm` is the host-userspace handle to the kernel-based virtual machine. Per `KVMIO` ioctl class (0xAE), userspace opens `/dev/kvm`, calls `KVM_GET_API_VERSION` (must equal `KVM_API_VERSION = 12`), `KVM_CHECK_EXTENSION(cap_id)` to probe capabilities, then `KVM_CREATE_VM` to receive a VM-fd. Per-VM ioctls (`KVM_SET_USER_MEMORY_REGION`, `KVM_CREATE_VCPU`, `KVM_CREATE_IRQCHIP`, ...) configure guest memory and emulated devices. Per-vCPU ioctls (`KVM_RUN`, `KVM_GET_REGS`, `KVM_SET_SREGS`, `KVM_GET_MSRS`, ...) drive guest execution. `KVM_RUN` enters the guest until a trap — exit reason and per-exit-reason payload are reported in the `struct kvm_run` page mmap'd at `vcpu_fd` offset 0 (size `KVM_GET_VCPU_MMAP_SIZE`). Critical for: virtualization, container runtimes (Firecracker/Cloud Hypervisor/QEMU), and the entire guest-machine ABI; **any** drift breaks every hypervisor running on the kernel.

This Tier-5 covers the full UAPI surface of `include/uapi/linux/kvm.h` (~1672 lines) plus the x86-arch UAPI definitions referenced by the generic header.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `KVM_API_VERSION` | per-kabi version constant = 12 | `kvm::api::API_VERSION` |
| `struct kvm_userspace_memory_region` | per-memslot descriptor (slot, flags, gpa, size, hva) | `KvmUserspaceMemoryRegion` |
| `struct kvm_userspace_memory_region2` | per-memslot ext (guest_memfd, offset) | `KvmUserspaceMemoryRegion2` |
| `struct kvm_run` | per-vCPU shared page (exit_reason + union) | `KvmRun` |
| `struct kvm_hyperv_exit` | per-Hyper-V exit (synic / hcall / syndbg) | `KvmHypervExit` |
| `struct kvm_xen_exit` | per-Xen hypercall exit | `KvmXenExit` |
| `struct kvm_irq_level` | per-IRQ level (irq, level) | `KvmIrqLevel` |
| `struct kvm_irqchip` | per-PIC / IOAPIC state | `KvmIrqchip` |
| `struct kvm_pit_config` | per-PIT2 config | `KvmPitConfig` |
| `struct kvm_dirty_log` | per-slot dirty bitmap | `KvmDirtyLog` |
| `struct kvm_clear_dirty_log` | per-slot clear range | `KvmClearDirtyLog` |
| `struct kvm_signal_mask` | per-vCPU sigmask | `KvmSignalMask` |
| `struct kvm_mp_state` | per-vCPU multiprocessor state | `KvmMpState` |
| `struct kvm_guest_debug` | per-vCPU debug control | `KvmGuestDebug` |
| `struct kvm_ioeventfd` | per-VM ioeventfd registration | `KvmIoeventfd` |
| `struct kvm_irqfd` | per-VM irqfd registration | `KvmIrqfd` |
| `struct kvm_enable_cap` | per-cap enable | `KvmEnableCap` |
| `struct kvm_irq_routing` / `_entry` | per-GSI routing table | `KvmIrqRouting` |
| `struct kvm_msi` | per-MSI signal | `KvmMsi` |
| `struct kvm_one_reg` / `_reg_list` | per-arch register access | `KvmOneReg` / `KvmRegList` |
| `struct kvm_clock_data` | per-VM clock | `KvmClockData` |
| `struct kvm_create_device` / `_device_attr` | per-device fd CRUD | `KvmCreateDevice` / `KvmDeviceAttr` |
| `struct kvm_dirty_gfn` | per-ring entry (flags, slot, offset) | `KvmDirtyGfn` |
| `struct kvm_stats_header` / `_stats_desc` | per-binary-stats descriptor | `KvmStatsHeader` / `KvmStatsDesc` |
| `struct kvm_memory_attributes` | per-range attribute set (private) | `KvmMemoryAttributes` |
| `struct kvm_create_guest_memfd` | per-guest_memfd create | `KvmCreateGuestMemfd` |
| `struct kvm_pre_fault_memory` | per-pre-fault range | `KvmPreFaultMemory` |
| `struct kvm_coalesced_mmio_ring` | per-ring of coalesced MMIO | `KvmCoalescedMmioRing` |
| `struct kvm_translation` | per-`KVM_TRANSLATE` (linear -> phys) | `KvmTranslation` |
| `struct kvm_interrupt` | per-`KVM_INTERRUPT` (irq) | `KvmInterrupt` |
| `struct kvm_enc_region` | per-encrypt range | `KvmEncRegion` |
| `struct kvm_msr_list` (arch/x86) | per-`KVM_GET_MSR_INDEX_LIST` | `KvmMsrList` |
| `struct kvm_msrs` (arch/x86) | per-`KVM_GET_MSRS` / `_SET_MSRS` | `KvmMsrs` |
| `struct kvm_cpuid2` (arch/x86) | per-`KVM_SET_CPUID2` / `_GET_CPUID2` | `KvmCpuid2` |

## ABI surface (constants + structs)

### Top-level constants

```
KVMIO                                 = 0xAE          /* ioctl class */
KVM_API_VERSION                       = 12            /* MUST match: KVM_GET_API_VERSION */
__KVM_HAVE_GUEST_DEBUG                                /* backward-compat marker */
SYNC_REGS_SIZE_BYTES                  = 2048          /* kvm_run.s.padding upper bound */
KVM_S390_SIE_PAGE_OFFSET              = 1
KVM_S390_GET_SKEYS_NONE               = 1
KVM_S390_SKEYS_MAX                    = 1048576
```

### Memslot flags (`struct kvm_userspace_memory_region.flags`)

```
KVM_MEM_LOG_DIRTY_PAGES               = 1 << 0
KVM_MEM_READONLY                      = 1 << 1
KVM_MEM_GUEST_MEMFD                   = 1 << 2
```

### `struct kvm_userspace_memory_region`

```
struct kvm_userspace_memory_region {
  u32 slot;
  u32 flags;
  u64 guest_phys_addr;
  u64 memory_size;     /* bytes */
  u64 userspace_addr;  /* hva */
};
```

### `struct kvm_userspace_memory_region2` (KVM_SET_USER_MEMORY_REGION2)

```
struct kvm_userspace_memory_region2 {
  u32 slot;
  u32 flags;
  u64 guest_phys_addr;
  u64 memory_size;
  u64 userspace_addr;
  u64 guest_memfd_offset;
  u32 guest_memfd;
  u32 pad1;
  u64 pad2[14];
};
```

### Top-level ioctls on `/dev/kvm` fd

```
KVM_GET_API_VERSION         = _IO(KVMIO,   0x00)                                 -> int (must == 12)
KVM_CREATE_VM               = _IO(KVMIO,   0x01)                                 -> vm-fd
KVM_GET_MSR_INDEX_LIST      = _IOWR(KVMIO, 0x02, struct kvm_msr_list)
KVM_CHECK_EXTENSION         = _IO(KVMIO,   0x03)                                 -> 0 (no) or n (yes)
KVM_GET_VCPU_MMAP_SIZE      = _IO(KVMIO,   0x04)                                 -> bytes
KVM_GET_SUPPORTED_CPUID     = _IOWR(KVMIO, 0x05, struct kvm_cpuid2)
KVM_S390_ENABLE_SIE         = _IO(KVMIO,   0x06)
KVM_GET_EMULATED_CPUID      = _IOWR(KVMIO, 0x09, struct kvm_cpuid2)
KVM_GET_MSR_FEATURE_INDEX_LIST = _IOWR(KVMIO, 0x0a, struct kvm_msr_list)
```

### VM-fd ioctls

```
KVM_CREATE_VCPU             = _IO(KVMIO,   0x41)                                 -> vcpu-fd
KVM_GET_DIRTY_LOG           = _IOW(KVMIO,  0x42, struct kvm_dirty_log)
KVM_SET_NR_MMU_PAGES        = _IO(KVMIO,   0x44)
KVM_GET_NR_MMU_PAGES        = _IO(KVMIO,   0x45)  /* deprecated */
KVM_SET_USER_MEMORY_REGION  = _IOW(KVMIO,  0x46, struct kvm_userspace_memory_region)
KVM_SET_TSS_ADDR            = _IO(KVMIO,   0x47)
KVM_SET_IDENTITY_MAP_ADDR   = _IOW(KVMIO,  0x48, __u64)
KVM_SET_USER_MEMORY_REGION2 = _IOW(KVMIO,  0x49, struct kvm_userspace_memory_region2)

KVM_CREATE_IRQCHIP          = _IO(KVMIO,   0x60)
KVM_IRQ_LINE                = _IOW(KVMIO,  0x61, struct kvm_irq_level)
KVM_GET_IRQCHIP             = _IOWR(KVMIO, 0x62, struct kvm_irqchip)
KVM_SET_IRQCHIP             = _IOR(KVMIO,  0x63, struct kvm_irqchip)
KVM_CREATE_PIT              = _IO(KVMIO,   0x64)
KVM_IRQ_LINE_STATUS         = _IOWR(KVMIO, 0x67, struct kvm_irq_level)
KVM_REGISTER_COALESCED_MMIO   = _IOW(KVMIO, 0x67, struct kvm_coalesced_mmio_zone)
KVM_UNREGISTER_COALESCED_MMIO = _IOW(KVMIO, 0x68, struct kvm_coalesced_mmio_zone)
KVM_SET_GSI_ROUTING         = _IOW(KVMIO,  0x6a, struct kvm_irq_routing)
KVM_REINJECT_CONTROL        = _IO(KVMIO,   0x71)
KVM_IRQFD                   = _IOW(KVMIO,  0x76, struct kvm_irqfd)
KVM_CREATE_PIT2             = _IOW(KVMIO,  0x77, struct kvm_pit_config)
KVM_SET_BOOT_CPU_ID         = _IO(KVMIO,   0x78)
KVM_IOEVENTFD               = _IOW(KVMIO,  0x79, struct kvm_ioeventfd)
KVM_SET_CLOCK               = _IOW(KVMIO,  0x7b, struct kvm_clock_data)
KVM_GET_CLOCK               = _IOR(KVMIO,  0x7c, struct kvm_clock_data)
KVM_SIGNAL_MSI              = _IOW(KVMIO,  0xa5, struct kvm_msi)
KVM_CREATE_DEVICE           = _IOWR(KVMIO, 0xe0, struct kvm_create_device)
KVM_GET_STATS_FD            = _IO(KVMIO,   0xce)
KVM_MEMORY_ENCRYPT_OP       = _IOWR(KVMIO, 0xba, unsigned long)
KVM_MEMORY_ENCRYPT_REG_REGION   = _IOR(KVMIO, 0xbb, struct kvm_enc_region)
KVM_MEMORY_ENCRYPT_UNREG_REGION = _IOR(KVMIO, 0xbc, struct kvm_enc_region)
KVM_RESET_DIRTY_RINGS       = _IO(KVMIO,  0xc7)
KVM_CLEAR_DIRTY_LOG         = _IOWR(KVMIO, 0xc0, struct kvm_clear_dirty_log)
KVM_SET_MEMORY_ATTRIBUTES   = _IOW(KVMIO, 0xd2, struct kvm_memory_attributes)
KVM_CREATE_GUEST_MEMFD      = _IOWR(KVMIO, 0xd4, struct kvm_create_guest_memfd)
KVM_PRE_FAULT_MEMORY        = _IOWR(KVMIO, 0xd5, struct kvm_pre_fault_memory)
KVM_X86_SET_MSR_FILTER      = _IOW(KVMIO,  0xc6, struct kvm_msr_filter)
KVM_ENABLE_CAP              = _IOW(KVMIO,  0xa3, struct kvm_enable_cap)   /* VM or vCPU */
KVM_HAS_DEVICE_ATTR         = _IOW(KVMIO,  0xe3, struct kvm_device_attr)
KVM_SET_DEVICE_ATTR         = _IOW(KVMIO,  0xe1, struct kvm_device_attr)
KVM_GET_DEVICE_ATTR         = _IOW(KVMIO,  0xe2, struct kvm_device_attr)
```

### vCPU-fd ioctls

```
KVM_RUN                     = _IO(KVMIO,   0x80)
KVM_GET_REGS                = _IOR(KVMIO,  0x81, struct kvm_regs)
KVM_SET_REGS                = _IOW(KVMIO,  0x82, struct kvm_regs)
KVM_GET_SREGS               = _IOR(KVMIO,  0x83, struct kvm_sregs)
KVM_SET_SREGS               = _IOW(KVMIO,  0x84, struct kvm_sregs)
KVM_TRANSLATE               = _IOWR(KVMIO, 0x85, struct kvm_translation)
KVM_INTERRUPT               = _IOW(KVMIO,  0x86, struct kvm_interrupt)
KVM_GET_MSRS                = _IOWR(KVMIO, 0x88, struct kvm_msrs)
KVM_SET_MSRS                = _IOW(KVMIO,  0x89, struct kvm_msrs)
KVM_SET_CPUID               = _IOW(KVMIO,  0x8a, struct kvm_cpuid)
KVM_SET_SIGNAL_MASK         = _IOW(KVMIO,  0x8b, struct kvm_signal_mask)
KVM_GET_FPU                 = _IOR(KVMIO,  0x8c, struct kvm_fpu)
KVM_SET_FPU                 = _IOW(KVMIO,  0x8d, struct kvm_fpu)
KVM_GET_LAPIC               = _IOR(KVMIO,  0x8e, struct kvm_lapic_state)
KVM_SET_LAPIC               = _IOW(KVMIO,  0x8f, struct kvm_lapic_state)
KVM_SET_CPUID2              = _IOW(KVMIO,  0x90, struct kvm_cpuid2)
KVM_GET_CPUID2              = _IOWR(KVMIO, 0x91, struct kvm_cpuid2)
KVM_TPR_ACCESS_REPORTING    = _IOWR(KVMIO, 0x92, struct kvm_tpr_access_ctl)
KVM_SET_VAPIC_ADDR          = _IOW(KVMIO,  0x93, struct kvm_vapic_addr)
KVM_GET_MP_STATE            = _IOR(KVMIO,  0x98, struct kvm_mp_state)
KVM_SET_MP_STATE            = _IOW(KVMIO,  0x99, struct kvm_mp_state)
KVM_NMI                     = _IO(KVMIO,   0x9a)
KVM_SET_GUEST_DEBUG         = _IOW(KVMIO,  0x9b, struct kvm_guest_debug)
KVM_X86_SETUP_MCE           = _IOW(KVMIO,  0x9c, __u64)
KVM_X86_GET_MCE_CAP_SUPPORTED = _IOR(KVMIO,  0x9d, __u64)
KVM_X86_SET_MCE             = _IOW(KVMIO,  0x9e, struct kvm_x86_mce)
KVM_GET_VCPU_EVENTS         = _IOR(KVMIO,  0x9f, struct kvm_vcpu_events)
KVM_SET_VCPU_EVENTS         = _IOW(KVMIO,  0xa0, struct kvm_vcpu_events)
KVM_GET_DEBUGREGS           = _IOR(KVMIO,  0xa1, struct kvm_debugregs)
KVM_SET_DEBUGREGS           = _IOW(KVMIO,  0xa2, struct kvm_debugregs)
KVM_GET_XSAVE               = _IOR(KVMIO,  0xa4, struct kvm_xsave)
KVM_SET_XSAVE               = _IOW(KVMIO,  0xa5, struct kvm_xsave)
KVM_GET_XCRS                = _IOR(KVMIO,  0xa6, struct kvm_xcrs)
KVM_SET_XCRS                = _IOW(KVMIO,  0xa7, struct kvm_xcrs)
KVM_DIRTY_TLB               = _IOW(KVMIO,  0xaa, struct kvm_dirty_tlb)
KVM_GET_ONE_REG             = _IOW(KVMIO,  0xab, struct kvm_one_reg)
KVM_SET_ONE_REG             = _IOW(KVMIO,  0xac, struct kvm_one_reg)
KVM_KVMCLOCK_CTRL           = _IO(KVMIO,   0xad)
KVM_GET_REG_LIST            = _IOWR(KVMIO, 0xb0, struct kvm_reg_list)
KVM_SMI                     = _IO(KVMIO,   0xb7)
KVM_GET_SREGS2              = _IOR(KVMIO,  0xcc, struct kvm_sregs2)
KVM_SET_SREGS2              = _IOW(KVMIO,  0xcd, struct kvm_sregs2)
KVM_GET_SUPPORTED_HV_CPUID  = _IOWR(KVMIO, 0xc1, struct kvm_cpuid2)
KVM_GET_NESTED_STATE        = _IOWR(KVMIO, 0xbe, struct kvm_nested_state)
KVM_SET_NESTED_STATE        = _IOW(KVMIO,  0xbf, struct kvm_nested_state)
KVM_GET_XSAVE2              = _IOR(KVMIO,  0xcf, struct kvm_xsave)
```

### `struct kvm_run` — vCPU shared page

```
struct kvm_run {
  /* in */
  u8  request_interrupt_window;
  u8  immediate_exit;            /* HINT_UNSAFE_IN_KVM */
  u8  padding1[6];
  /* out */
  u32 exit_reason;
  u8  ready_for_interrupt_injection;
  u8  if_flag;
  u16 flags;
  /* in (pre) / out (post) */
  u64 cr8;
  u64 apic_base;
#ifdef __KVM_S390
  u64 psw_mask;
  u64 psw_addr;
#endif
  union {
    struct { u64 hardware_exit_reason; } hw;                                /* EXIT_UNKNOWN */
    struct { u64 hardware_entry_failure_reason; u32 cpu; } fail_entry;      /* EXIT_FAIL_ENTRY */
    struct { u32 exception; u32 error_code; } ex;                           /* EXIT_EXCEPTION */
    struct { u8 direction; u8 size; u16 port; u32 count; u64 data_offset;
           } io;                                                            /* EXIT_IO */
    struct { struct kvm_debug_exit_arch arch; } debug;                      /* EXIT_DEBUG */
    struct { u64 phys_addr; u8 data[8]; u32 len; u8 is_write; } mmio;       /* EXIT_MMIO */
    struct { u64 phys_addr; u8 data[8]; u32 len; u8 is_write; } iocsr_io;   /* EXIT_LOONGARCH_IOCSR */
    struct { u64 nr; u64 args[6]; u64 ret;
             union { u32 longmode; u64 flags; }; } hypercall;               /* EXIT_HYPERCALL */
    struct { u64 rip; u32 is_write; u32 pad; } tpr_access;                  /* EXIT_TPR_ACCESS */
    struct { u8 icptcode; u16 ipa; u32 ipb; } s390_sieic;                   /* EXIT_S390_SIEIC */
    u64 s390_reset_flags;                                                   /* EXIT_S390_RESET */
    struct { u64 trans_exc_code; u32 pgm_code; } s390_ucontrol;             /* EXIT_S390_UCONTROL */
    struct { u32 dcrn; u32 data; u8 is_write; } dcr;                        /* EXIT_DCR (deprecated) */
    struct { u32 suberror; u32 ndata; u64 data[16]; } internal;             /* EXIT_INTERNAL_ERROR */
    struct { u32 suberror; u32 ndata; u64 flags;
             union { struct { u8 insn_size; u8 insn_bytes[15]; }; };
           } emulation_failure;                                             /* EXIT_INTERNAL_ERROR/EMULATION */
    struct { u64 gprs[32]; } osi;                                           /* EXIT_OSI */
    struct { u64 nr; u64 ret; u64 args[9]; } papr_hcall;                    /* EXIT_PAPR_HCALL */
    struct { u16 subchannel_id; u16 subchannel_nr;
             u32 io_int_parm; u32 io_int_word;
             u32 ipb; u8 dequeued; } s390_tsch;                             /* EXIT_S390_TSCH */
    struct { u32 epr; } epr;                                                /* EXIT_EPR */
    struct { u32 type; u32 ndata;
             union { u64 flags; u64 data[16]; }; } system_event;            /* EXIT_SYSTEM_EVENT */
    struct { u64 addr; u8 ar; u8 reserved; u8 fc; u8 sel1; u16 sel2;
           } s390_stsi;                                                     /* EXIT_S390_STSI */
    struct { u8 vector; } eoi;                                              /* EXIT_IOAPIC_EOI */
    struct kvm_hyperv_exit hyperv;                                          /* EXIT_HYPERV */
    struct { u64 esr_iss; u64 fault_ipa; } arm_nisv;                        /* EXIT_ARM_NISV / EXIT_ARM_LDST64B */
    struct { u8 error; u8 pad[7]; u32 reason; u32 index; u64 data; } msr;   /* EXIT_X86_RDMSR / EXIT_X86_WRMSR */
    struct kvm_xen_exit xen;                                                /* EXIT_XEN */
    struct { ulong extension_id; ulong function_id;
             ulong args[6]; ulong ret[2]; } riscv_sbi;                      /* EXIT_RISCV_SBI */
    struct { ulong csr_num; ulong new_value;
             ulong write_mask; ulong ret_value; } riscv_csr;                /* EXIT_RISCV_CSR */
    struct { u32 flags; } notify;                                           /* EXIT_NOTIFY */
    struct { u64 flags; u64 gpa; u64 size; } memory_fault;                  /* EXIT_MEMORY_FAULT */
    struct { u64 flags; u64 nr;
             union { struct { u64 ret; u64 data[5]; } unknown;
                     struct { u64 ret; u64 gpa; u64 size; } get_quote;
                     struct { u64 ret; u64 leaf;
                              u64 r11, r12, r13, r14; } get_tdvmcall_info;
                     struct { u64 ret; u64 vector; } setup_event_notify; };
           } tdx;                                                           /* EXIT_TDX */
    struct { u64 flags; u64 esr; u64 gva; u64 gpa; } arm_sea;               /* EXIT_ARM_SEA */
    struct kvm_exit_snp_req_certs snp_req_certs;                            /* EXIT_SNP_REQ_CERTS */
    char padding[256];
  };
  u64 kvm_valid_regs;
  u64 kvm_dirty_regs;
  union { struct kvm_sync_regs regs; char padding[SYNC_REGS_SIZE_BYTES]; } s;
};
```

### `KVM_EXIT_*` reasons

```
KVM_EXIT_UNKNOWN          = 0
KVM_EXIT_EXCEPTION        = 1
KVM_EXIT_IO               = 2
KVM_EXIT_HYPERCALL        = 3
KVM_EXIT_DEBUG            = 4
KVM_EXIT_HLT              = 5
KVM_EXIT_MMIO             = 6
KVM_EXIT_IRQ_WINDOW_OPEN  = 7
KVM_EXIT_SHUTDOWN         = 8
KVM_EXIT_FAIL_ENTRY       = 9
KVM_EXIT_INTR             = 10
KVM_EXIT_SET_TPR          = 11
KVM_EXIT_TPR_ACCESS       = 12
KVM_EXIT_S390_SIEIC       = 13
KVM_EXIT_S390_RESET       = 14
KVM_EXIT_DCR              = 15  /* deprecated */
KVM_EXIT_NMI              = 16
KVM_EXIT_INTERNAL_ERROR   = 17
KVM_EXIT_OSI              = 18
KVM_EXIT_PAPR_HCALL       = 19
KVM_EXIT_S390_UCONTROL    = 20
KVM_EXIT_WATCHDOG         = 21
KVM_EXIT_S390_TSCH        = 22
KVM_EXIT_EPR              = 23
KVM_EXIT_SYSTEM_EVENT     = 24
KVM_EXIT_S390_STSI        = 25
KVM_EXIT_IOAPIC_EOI       = 26
KVM_EXIT_HYPERV           = 27
KVM_EXIT_ARM_NISV         = 28
KVM_EXIT_X86_RDMSR        = 29
KVM_EXIT_X86_WRMSR        = 30
KVM_EXIT_DIRTY_RING_FULL  = 31
KVM_EXIT_AP_RESET_HOLD    = 32
KVM_EXIT_X86_BUS_LOCK     = 33
KVM_EXIT_XEN              = 34
KVM_EXIT_RISCV_SBI        = 35
KVM_EXIT_RISCV_CSR        = 36
KVM_EXIT_NOTIFY           = 37
KVM_EXIT_LOONGARCH_IOCSR  = 38
KVM_EXIT_MEMORY_FAULT     = 39
KVM_EXIT_TDX              = 40
KVM_EXIT_ARM_SEA          = 41
KVM_EXIT_ARM_LDST64B      = 42
KVM_EXIT_SNP_REQ_CERTS    = 43

/* Sub-fields */
KVM_EXIT_IO_IN            = 0
KVM_EXIT_IO_OUT           = 1
KVM_EXIT_HYPERV_SYNIC     = 1
KVM_EXIT_HYPERV_HCALL     = 2
KVM_EXIT_HYPERV_SYNDBG    = 3
KVM_EXIT_XEN_HCALL        = 1
KVM_INTERNAL_ERROR_EMULATION                  = 1
KVM_INTERNAL_ERROR_SIMUL_EX                   = 2
KVM_INTERNAL_ERROR_DELIVERY_EV                = 3
KVM_INTERNAL_ERROR_UNEXPECTED_EXIT_REASON     = 4
KVM_INTERNAL_ERROR_EMULATION_FLAG_INSTRUCTION_BYTES = 1ULL << 0
KVM_MSR_EXIT_REASON_INVAL                     = 1 << 0
KVM_MSR_EXIT_REASON_UNKNOWN                   = 1 << 1
KVM_MSR_EXIT_REASON_FILTER                    = 1 << 2
KVM_MEMORY_EXIT_FLAG_PRIVATE                  = 1ULL << 3
KVM_NOTIFY_CONTEXT_INVALID                    = 1 << 0
KVM_EXIT_ARM_SEA_FLAG_GPA_VALID               = 1ULL << 0
```

### `KVM_SYSTEM_EVENT_*` (kvm_run.system_event.type)

```
KVM_SYSTEM_EVENT_SHUTDOWN        = 1
KVM_SYSTEM_EVENT_RESET           = 2
KVM_SYSTEM_EVENT_CRASH           = 3
KVM_SYSTEM_EVENT_WAKEUP          = 4
KVM_SYSTEM_EVENT_SUSPEND         = 5
KVM_SYSTEM_EVENT_SEV_TERM        = 6
KVM_SYSTEM_EVENT_TDX_FATAL       = 7
```

### MP-state (`struct kvm_mp_state.mp_state`)

```
KVM_MP_STATE_RUNNABLE           = 0
KVM_MP_STATE_UNINITIALIZED      = 1
KVM_MP_STATE_INIT_RECEIVED      = 2
KVM_MP_STATE_HALTED             = 3
KVM_MP_STATE_SIPI_RECEIVED      = 4
KVM_MP_STATE_STOPPED            = 5
KVM_MP_STATE_CHECK_STOP         = 6
KVM_MP_STATE_OPERATING          = 7
KVM_MP_STATE_LOAD               = 8
KVM_MP_STATE_AP_RESET_HOLD      = 9
KVM_MP_STATE_SUSPENDED          = 10
```

### `KVM_GUESTDBG_*` (guest_debug.control)

```
KVM_GUESTDBG_ENABLE            = 0x00000001    /* common */
KVM_GUESTDBG_SINGLESTEP        = 0x00000002    /* common */
KVM_GUESTDBG_USE_SW_BP         = 0x00010000    /* x86 */
KVM_GUESTDBG_USE_HW_BP         = 0x00020000    /* x86 */
KVM_GUESTDBG_INJECT_DB         = 0x00040000    /* x86 */
KVM_GUESTDBG_INJECT_BP         = 0x00080000    /* x86 */
KVM_GUESTDBG_BLOCKIRQ          = 0x00100000    /* x86 */
```

### `KVM_IRQCHIP_*` (x86 chip_id)

```
KVM_IRQCHIP_PIC_MASTER         = 0
KVM_IRQCHIP_PIC_SLAVE          = 1
KVM_IRQCHIP_IOAPIC             = 2
KVM_NR_IRQCHIPS                = 3
```

### IRQ-routing entry types (`struct kvm_irq_routing_entry.type`)

```
KVM_IRQ_ROUTING_IRQCHIP        = 1
KVM_IRQ_ROUTING_MSI            = 2
KVM_IRQ_ROUTING_S390_ADAPTER   = 3
KVM_IRQ_ROUTING_HV_SINT        = 4
KVM_IRQ_ROUTING_XEN_EVTCHN     = 5
KVM_IRQ_ROUTING_XEN_EVTCHN_PRIO_2LEVEL = (u32)(-1)
KVM_MSI_VALID_DEVID            = 1U << 0
```

### Ioeventfd flags

```
KVM_IOEVENTFD_FLAG_DATAMATCH         = 1 << 0
KVM_IOEVENTFD_FLAG_PIO               = 1 << 1
KVM_IOEVENTFD_FLAG_DEASSIGN          = 1 << 2
KVM_IOEVENTFD_FLAG_VIRTIO_CCW_NOTIFY = 1 << 3
KVM_IOEVENTFD_VALID_FLAG_MASK        = (1 << kvm_ioeventfd_flag_nr_max) - 1
```

### Irqfd flags

```
KVM_IRQFD_FLAG_DEASSIGN              = 1 << 0
KVM_IRQFD_FLAG_RESAMPLE              = 1 << 1
```

### Clock flags (`struct kvm_clock_data.flags`)

```
KVM_CLOCK_TSC_STABLE                 = 2
KVM_CLOCK_REALTIME                   = 1 << 2
KVM_CLOCK_HOST_TSC                   = 1 << 3
```

### KVM_X86_DISABLE_EXITS_*

```
KVM_X86_DISABLE_EXITS_MWAIT          = 1 << 0
KVM_X86_DISABLE_EXITS_HLT            = 1 << 1
KVM_X86_DISABLE_EXITS_PAUSE          = 1 << 2
KVM_X86_DISABLE_EXITS_CSTATE         = 1 << 3
KVM_X86_DISABLE_EXITS_APERFMPERF     = 1 << 4
```

### KVM_X86_QUIRK_*

```
KVM_X86_QUIRK_LINT0_REENABLED                   = 1 << 0
KVM_X86_QUIRK_CD_NW_CLEARED                     = 1 << 1
KVM_X86_QUIRK_LAPIC_MMIO_HOLE                   = 1 << 2
KVM_X86_QUIRK_OUT_7E_INC_RIP                    = 1 << 3
KVM_X86_QUIRK_MISC_ENABLE_NO_MWAIT              = 1 << 4
KVM_X86_QUIRK_FIX_HYPERCALL_INSN                = 1 << 5
KVM_X86_QUIRK_MWAIT_NEVER_UD_FAULTS             = 1 << 6
KVM_X86_QUIRK_SLOT_ZAP_ALL                      = 1 << 7
KVM_X86_QUIRK_STUFF_FEATURE_MSRS                = 1 << 8
KVM_X86_QUIRK_IGNORE_GUEST_PAT                  = 1 << 9
KVM_X86_QUIRK_VMCS12_ALLOW_FREEZE_IN_SMM        = 1 << 10
```

### KVM_X86 VM-type (`KVM_CREATE_VM` arg on x86)

```
KVM_X86_DEFAULT_VM         = 0
KVM_X86_SW_PROTECTED_VM    = 1
KVM_X86_SEV_VM             = 2
KVM_X86_SEV_ES_VM          = 3
KVM_X86_SNP_VM             = 4
KVM_X86_TDX_VM             = 5
```

### KVM_VM_TYPE (architectural)

```
KVM_VM_S390_UCONTROL       = 1
KVM_VM_PPC_HV              = 1
KVM_VM_PPC_PR              = 2
KVM_VM_MIPS_AUTO           = 0
KVM_VM_MIPS_VZ             = 1
KVM_VM_MIPS_TE             = 2
KVM_VM_TYPE_ARM_IPA_SIZE_MASK   = 0xffULL
KVM_VM_TYPE_ARM_PROTECTED       = 1UL << 31
```

### KVM_FEATURE_* (CPUID 0x40000001 EAX bits — `arch/x86/include/uapi/asm/kvm_para.h`)

```
KVM_FEATURE_CLOCKSOURCE              = 0
KVM_FEATURE_NOP_IO_DELAY             = 1
KVM_FEATURE_MMU_OP                   = 2
KVM_FEATURE_CLOCKSOURCE2             = 3
KVM_FEATURE_ASYNC_PF                 = 4
KVM_FEATURE_STEAL_TIME               = 5
KVM_FEATURE_PV_EOI                   = 6
KVM_FEATURE_PV_UNHALT                = 7
KVM_FEATURE_PV_TLB_FLUSH             = 9
KVM_FEATURE_ASYNC_PF_VMEXIT          = 10
KVM_FEATURE_PV_SEND_IPI              = 11
KVM_FEATURE_POLL_CONTROL             = 12
KVM_FEATURE_PV_SCHED_YIELD           = 13
KVM_FEATURE_ASYNC_PF_INT             = 14
KVM_FEATURE_MSI_EXT_DEST_ID          = 15
KVM_FEATURE_HC_MAP_GPA_RANGE         = 16
KVM_FEATURE_MIGRATION_CONTROL        = 17
KVM_FEATURE_CLOCKSOURCE_STABLE_BIT   = 24
```

### `struct kvm_msr_list` (arch/x86)

```
struct kvm_msr_list {
  u32 nmsrs;
  u32 indices[];   /* flex array, count = nmsrs */
};
```

### `struct kvm_msrs` (arch/x86)

```
struct kvm_msr_entry { u32 index; u32 reserved; u64 data; };
struct kvm_msrs {
  u32 nmsrs;
  u32 pad;
  struct kvm_msr_entry entries[];   /* count = nmsrs */
};
```

### `struct kvm_cpuid2` (arch/x86)

```
struct kvm_cpuid_entry2 {
  u32 function;
  u32 index;
  u32 flags;
  u32 eax, ebx, ecx, edx;
  u32 padding[3];
};
struct kvm_cpuid2 {
  u32 nent;
  u32 padding;
  struct kvm_cpuid_entry2 entries[];   /* count = nent */
};

KVM_CPUID_FLAG_SIGNIFCANT_INDEX     = 1 << 0
KVM_CPUID_FLAG_STATEFUL_FUNC        = 1 << 1
KVM_CPUID_FLAG_STATE_READ_NEXT      = 1 << 2
```

### `struct kvm_dirty_log`

```
struct kvm_dirty_log {
  u32 slot;
  u32 padding1;
  union { void __user *dirty_bitmap; u64 padding2; };  /* one bit per page */
};
struct kvm_clear_dirty_log {
  u32 slot;
  u32 num_pages;
  u64 first_page;
  union { void __user *dirty_bitmap; u64 padding2; };
};
```

### `struct kvm_dirty_gfn` (dirty ring entry)

```
struct kvm_dirty_gfn { u32 flags; u32 slot; u64 offset; };

KVM_DIRTY_GFN_F_DIRTY            = _BITUL(0)
KVM_DIRTY_GFN_F_RESET            = _BITUL(1)
KVM_DIRTY_GFN_F_MASK             = 0x3
KVM_DIRTY_LOG_PAGE_OFFSET        = 0 (default; arch-overrideable)
KVM_DIRTY_LOG_MANUAL_PROTECT_ENABLE = 1 << 0
KVM_DIRTY_LOG_INITIALLY_SET         = 1 << 1
```

### `KVM_CAP_*` capability list (probed by `KVM_CHECK_EXTENSION`)

```
KVM_CAP_IRQCHIP                              = 0
KVM_CAP_HLT                                  = 1
KVM_CAP_MMU_SHADOW_CACHE_CONTROL             = 2
KVM_CAP_USER_MEMORY                          = 3
KVM_CAP_SET_TSS_ADDR                         = 4
KVM_CAP_VAPIC                                = 6
KVM_CAP_EXT_CPUID                            = 7
KVM_CAP_CLOCKSOURCE                          = 8
KVM_CAP_NR_VCPUS                             = 9
KVM_CAP_NR_MEMSLOTS                          = 10
KVM_CAP_PIT                                  = 11
KVM_CAP_NOP_IO_DELAY                         = 12
KVM_CAP_PV_MMU                               = 13
KVM_CAP_MP_STATE                             = 14
KVM_CAP_COALESCED_MMIO                       = 15
KVM_CAP_SYNC_MMU                             = 16
KVM_CAP_IOMMU                                = 18
KVM_CAP_DESTROY_MEMORY_REGION_WORKS          = 21
KVM_CAP_USER_NMI                             = 22
KVM_CAP_SET_GUEST_DEBUG                      = 23
KVM_CAP_REINJECT_CONTROL                     = 24
KVM_CAP_IRQ_ROUTING                          = 25
KVM_CAP_IRQ_INJECT_STATUS                    = 26
KVM_CAP_ASSIGN_DEV_IRQ                       = 29
KVM_CAP_JOIN_MEMORY_REGIONS_WORKS            = 30
KVM_CAP_MCE                                  = 31
KVM_CAP_IRQFD                                = 32
KVM_CAP_PIT2                                 = 33
KVM_CAP_SET_BOOT_CPU_ID                      = 34
KVM_CAP_PIT_STATE2                           = 35
KVM_CAP_IOEVENTFD                            = 36
KVM_CAP_SET_IDENTITY_MAP_ADDR                = 37
KVM_CAP_XEN_HVM                              = 38
KVM_CAP_ADJUST_CLOCK                         = 39
KVM_CAP_INTERNAL_ERROR_DATA                  = 40
KVM_CAP_VCPU_EVENTS                          = 41
KVM_CAP_S390_PSW                             = 42
KVM_CAP_HYPERV                               = 44
KVM_CAP_HYPERV_VAPIC                         = 45
KVM_CAP_HYPERV_SPIN                          = 46
KVM_CAP_INTR_SHADOW                          = 49
KVM_CAP_DEBUGREGS                            = 50
KVM_CAP_X86_ROBUST_SINGLESTEP                = 51
KVM_CAP_ENABLE_CAP                           = 54
KVM_CAP_XSAVE                                = 55
KVM_CAP_XCRS                                 = 56
KVM_CAP_ASYNC_PF                             = 59
KVM_CAP_TSC_CONTROL                          = 60
KVM_CAP_GET_TSC_KHZ                          = 61
KVM_CAP_MAX_VCPUS                            = 66
KVM_CAP_ONE_REG                              = 70
KVM_CAP_TSC_DEADLINE_TIMER                   = 72
KVM_CAP_SYNC_REGS                            = 74
KVM_CAP_KVMCLOCK_CTRL                        = 76
KVM_CAP_SIGNAL_MSI                           = 77
KVM_CAP_READONLY_MEM                         = 81
KVM_CAP_IRQFD_RESAMPLE                       = 82
KVM_CAP_DEVICE_CTRL                          = 89
KVM_CAP_EXT_EMUL_CPUID                       = 95
KVM_CAP_HYPERV_TIME                          = 96
KVM_CAP_ENABLE_CAP_VM                        = 98
KVM_CAP_IOEVENTFD_NO_LENGTH                  = 100
KVM_CAP_VM_ATTRIBUTES                        = 101
KVM_CAP_CHECK_EXTENSION_VM                   = 105
KVM_CAP_X86_SMM                              = 117
KVM_CAP_MULTI_ADDRESS_SPACE                  = 118
KVM_CAP_SPLIT_IRQCHIP                        = 121
KVM_CAP_IOEVENTFD_ANY_LENGTH                 = 122
KVM_CAP_HYPERV_SYNIC                         = 123
KVM_CAP_VCPU_ATTRIBUTES                      = 127
KVM_CAP_MAX_VCPU_ID                          = 128
KVM_CAP_X2APIC_API                           = 129
KVM_CAP_MSI_DEVID                            = 131
KVM_CAP_IMMEDIATE_EXIT                       = 136
KVM_CAP_X86_DISABLE_EXITS                    = 143
KVM_CAP_HYPERV_SYNIC2                        = 148
KVM_CAP_HYPERV_VP_INDEX                      = 149
KVM_CAP_GET_MSR_FEATURES                     = 153
KVM_CAP_HYPERV_EVENTFD                       = 154
KVM_CAP_HYPERV_TLBFLUSH                      = 155
KVM_CAP_NESTED_STATE                         = 157
KVM_CAP_MSR_PLATFORM_INFO                    = 159
KVM_CAP_HYPERV_SEND_IPI                      = 161
KVM_CAP_COALESCED_PIO                        = 162
KVM_CAP_HYPERV_ENLIGHTENED_VMCS              = 163
KVM_CAP_EXCEPTION_PAYLOAD                    = 164
KVM_CAP_MANUAL_DIRTY_LOG_PROTECT             = 166  /* Obsolete */
KVM_CAP_HYPERV_CPUID                         = 167
KVM_CAP_MANUAL_DIRTY_LOG_PROTECT2            = 168
KVM_CAP_PMU_EVENT_FILTER                     = 173
KVM_CAP_HYPERV_DIRECT_TLBFLUSH               = 175
KVM_CAP_HALT_POLL                            = 182
KVM_CAP_ASYNC_PF_INT                         = 183
KVM_CAP_LAST_CPU                             = 184
KVM_CAP_SMALLER_MAXPHYADDR                   = 185
KVM_CAP_STEAL_TIME                           = 187
KVM_CAP_X86_USER_SPACE_MSR                   = 188
KVM_CAP_X86_MSR_FILTER                       = 189
KVM_CAP_ENFORCE_PV_FEATURE_CPUID             = 190
KVM_CAP_SYS_HYPERV_CPUID                     = 191
KVM_CAP_DIRTY_LOG_RING                       = 192
KVM_CAP_X86_BUS_LOCK_EXIT                    = 193
KVM_CAP_SET_GUEST_DEBUG2                     = 195
KVM_CAP_SGX_ATTRIBUTE                        = 196
KVM_CAP_VM_COPY_ENC_CONTEXT_FROM             = 197
KVM_CAP_PTP_KVM                              = 198
KVM_CAP_HYPERV_ENFORCE_CPUID                 = 199
KVM_CAP_SREGS2                               = 200
KVM_CAP_EXIT_HYPERCALL                       = 201
KVM_CAP_BINARY_STATS_FD                      = 203
KVM_CAP_EXIT_ON_EMULATION_FAILURE            = 204
KVM_CAP_VM_MOVE_ENC_CONTEXT_FROM             = 206
KVM_CAP_VM_GPA_BITS                          = 207
KVM_CAP_XSAVE2                               = 208
KVM_CAP_SYS_ATTRIBUTES                       = 209
KVM_CAP_PMU_CAPABILITY                       = 212
KVM_CAP_DISABLE_QUIRKS2                      = 213
KVM_CAP_VM_TSC_CONTROL                       = 214
KVM_CAP_SYSTEM_EVENT_DATA                    = 215
KVM_CAP_X86_TRIPLE_FAULT_EVENT               = 218
KVM_CAP_X86_NOTIFY_VMEXIT                    = 219
KVM_CAP_VM_DISABLE_NX_HUGE_PAGES             = 220
KVM_CAP_DIRTY_LOG_RING_ACQ_REL               = 223
KVM_CAP_DIRTY_LOG_RING_WITH_BITMAP           = 225
KVM_CAP_PMU_EVENT_MASKED_EVENTS              = 226
KVM_CAP_USER_MEMORY2                         = 231
KVM_CAP_MEMORY_FAULT_INFO                    = 232
KVM_CAP_MEMORY_ATTRIBUTES                    = 233
KVM_CAP_GUEST_MEMFD                          = 234
KVM_CAP_VM_TYPES                             = 235
KVM_CAP_PRE_FAULT_MEMORY                     = 236
KVM_CAP_X86_APIC_BUS_CYCLES_NS               = 237
KVM_CAP_X86_GUEST_MODE                       = 238
KVM_CAP_GUEST_MEMFD_FLAGS                    = 244
```
(arch-specific KVM_CAP_PPC_*, _ARM_*, _S390_*, _MIPS_*, _RISCV_*, _LOONGARCH_* are listed in the header but elided here; their numeric IDs are part of the ABI.)

### Memory-encryption / guest-memfd / attributes

```
struct kvm_enc_region          { u64 addr; u64 size; };
struct kvm_memory_attributes   { u64 address; u64 size; u64 attributes; u64 flags; };
KVM_MEMORY_ATTRIBUTE_PRIVATE   = 1ULL << 3
struct kvm_create_guest_memfd  { u64 size; u64 flags; u64 reserved[6]; };
GUEST_MEMFD_FLAG_MMAP          = 1ULL << 0
GUEST_MEMFD_FLAG_INIT_SHARED   = 1ULL << 1
struct kvm_pre_fault_memory    { u64 gpa; u64 size; u64 flags; u64 padding[5]; };
```

### Bus-lock / PMU / notify-vmexit

```
KVM_BUS_LOCK_DETECTION_OFF             = 1 << 0
KVM_BUS_LOCK_DETECTION_EXIT            = 1 << 1
KVM_PMU_CAP_DISABLE                    = 1 << 0
KVM_X86_NOTIFY_VMEXIT_ENABLED          = 1ULL << 0
KVM_X86_NOTIFY_VMEXIT_USER             = 1ULL << 1
```

### Stats (binary)

```
struct kvm_stats_header { u32 flags; u32 name_size; u32 num_desc;
                          u32 id_offset; u32 desc_offset; u32 data_offset; };
struct kvm_stats_desc   { u32 flags; s16 exponent; u16 size;
                          u32 offset; u32 bucket_size; char name[]; };
KVM_STATS_TYPE_{CUMULATIVE|INSTANT|PEAK|LINEAR_HIST|LOG_HIST} (shift 0, mask 0xF)
KVM_STATS_UNIT_{NONE|BYTES|SECONDS|CYCLES|BOOLEAN} (shift 4, mask 0xF)
KVM_STATS_BASE_{POW10|POW2} (shift 8, mask 0xF)
```

### `enum kvm_device_type` (KVM_CREATE_DEVICE)

```
KVM_DEV_TYPE_FSL_MPIC_20         = 1
KVM_DEV_TYPE_FSL_MPIC_42
KVM_DEV_TYPE_XICS
KVM_DEV_TYPE_VFIO
KVM_DEV_TYPE_ARM_VGIC_V2
KVM_DEV_TYPE_FLIC
KVM_DEV_TYPE_ARM_VGIC_V3
KVM_DEV_TYPE_ARM_VGIC_ITS
KVM_DEV_TYPE_XIVE
KVM_DEV_TYPE_ARM_PV_TIME
KVM_DEV_TYPE_RISCV_AIA
KVM_DEV_TYPE_LOONGARCH_IPI
KVM_DEV_TYPE_LOONGARCH_EIOINTC
KVM_DEV_TYPE_LOONGARCH_PCHPIC
KVM_DEV_TYPE_LOONGARCH_DMSINTC
KVM_DEV_TYPE_ARM_VGIC_V5
KVM_DEV_TYPE_MAX
KVM_CREATE_DEVICE_TEST                = 1
KVM_DEV_VFIO_FILE                     = 1
KVM_DEV_VFIO_FILE_ADD                 = 1
KVM_DEV_VFIO_FILE_DEL                 = 2
KVM_DEV_VFIO_GROUP                    = KVM_DEV_VFIO_FILE
KVM_DEV_VFIO_GROUP_ADD                = KVM_DEV_VFIO_FILE_ADD
KVM_DEV_VFIO_GROUP_DEL                = KVM_DEV_VFIO_FILE_DEL
KVM_DEV_VFIO_GROUP_SET_SPAPR_TCE      = 3
```

### KVM_REG_* (KVM_GET_ONE_REG / KVM_SET_ONE_REG arch identifiers)

```
KVM_REG_ARCH_MASK              = 0xff00000000000000ULL
KVM_REG_GENERIC                = 0x0000000000000000ULL
KVM_REG_PPC                    = 0x1000000000000000ULL
KVM_REG_X86                    = 0x2000000000000000ULL
KVM_REG_IA64                   = 0x3000000000000000ULL
KVM_REG_ARM                    = 0x4000000000000000ULL
KVM_REG_S390                   = 0x5000000000000000ULL
KVM_REG_ARM64                  = 0x6000000000000000ULL
KVM_REG_MIPS                   = 0x7000000000000000ULL
KVM_REG_RISCV                  = 0x8000000000000000ULL
KVM_REG_LOONGARCH              = 0x9000000000000000ULL
KVM_REG_SIZE_{U8..U2048}       (shift 52, mask 0x00f0000000000000)
```

### Note on KVM_REQ_* (kernel-internal)

`KVM_REQ_*` request bits live in `include/linux/kvm_host.h` and `arch/<arch>/include/asm/kvm_host.h`. They are **not** UAPI — they govern `vcpu->requests` per-iteration. Listed here for cross-reference only:
- `KVM_REQ_TLB_FLUSH`, `KVM_REQ_MMU_RELOAD`, `KVM_REQ_PENDING_TIMER`, `KVM_REQ_UNHALT`,
  `KVM_REQ_VM_DEAD`, `KVM_REQ_UNBLOCK`, `KVM_REQ_DIRTY_RING_SOFT_FULL`, plus arch-specific
  bits (x86: `KVM_REQ_EVENT`, `KVM_REQ_TRIPLE_FAULT`, `KVM_REQ_NMI`, `KVM_REQ_SMI`, ...).
Rookery must preserve their **semantics** but not their numeric identity (no UAPI dependency).

## Compatibility contract

REQ-1 `KVM_GET_API_VERSION` (top-level, `_IO(KVMIO, 0x00)`):
- Argument: none.
- Returns: integer. **Must == 12**.
- If kernel returns any other value, userspace aborts (compatibility check).

REQ-2 `KVM_CREATE_VM` (top-level, `_IO(KVMIO, 0x01)`):
- Argument: machine-type. 0 = arch default; x86 may set `KVM_X86_*_VM`; arm64 may set `KVM_VM_TYPE_ARM_IPA_SIZE(n) | KVM_VM_TYPE_ARM_PROTECTED`; s390 may set `KVM_VM_S390_UCONTROL`; ppc may set `KVM_VM_PPC_HV` / `KVM_VM_PPC_PR`; mips may set `KVM_VM_MIPS_*`.
- Returns: new VM fd, or -EINVAL on unsupported machine-type, -ENOMEM on alloc-fail.
- Per-VM lifetime: fd-tied; close releases all vCPUs, memslots, devices.

REQ-3 `KVM_GET_MSR_INDEX_LIST` / `KVM_GET_MSR_FEATURE_INDEX_LIST`:
- Argument: `struct kvm_msr_list *` (in: `nmsrs` = buffer cap; out: `nmsrs` = actual count).
- If buffer too small: returns -E2BIG, leaves `nmsrs` set to required count.

REQ-4 `KVM_CHECK_EXTENSION`:
- Argument: `KVM_CAP_*` cap id (int).
- Returns: 0 = unsupported; ≥1 = supported (some caps return a value, e.g., `KVM_CAP_MAX_VCPUS`).
- Per-VM-or-top-level: top-level fd allows global probe; VM-fd allows VM-scope check (`KVM_CAP_CHECK_EXTENSION_VM`).

REQ-5 `KVM_GET_VCPU_MMAP_SIZE`:
- Argument: none.
- Returns: size in bytes (multiple of PAGE_SIZE) — the mmap length for vcpu_fd at offset 0 (= `struct kvm_run` + sync_regs).

REQ-6 `KVM_SET_USER_MEMORY_REGION` (VM-fd):
- Argument: `struct kvm_userspace_memory_region`.
- `flags` ⊆ {`KVM_MEM_LOG_DIRTY_PAGES`, `KVM_MEM_READONLY`, `KVM_MEM_GUEST_MEMFD`}.
- `memory_size == 0` deletes the slot.
- `guest_phys_addr` + `memory_size` must be PAGE_SIZE aligned and not wrap.
- `userspace_addr` must be PAGE_SIZE aligned and reference user-accessible memory of the calling mm.
- `KVM_MEM_GUEST_MEMFD` not valid with `KVM_SET_USER_MEMORY_REGION` (only `_REGION2`).
- Returns 0 / -EEXIST / -EINVAL / -ENOMEM.

REQ-7 `KVM_SET_USER_MEMORY_REGION2`:
- As REQ-6 + `guest_memfd_offset`/`guest_memfd`. Requires `KVM_CAP_GUEST_MEMFD`.
- `flags & KVM_MEM_GUEST_MEMFD` ⇒ `guest_memfd` is a valid fd from `KVM_CREATE_GUEST_MEMFD`.

REQ-8 `KVM_CREATE_VCPU` (VM-fd):
- Argument: vcpu slot id (≥0, < `KVM_CAP_MAX_VCPU_ID`).
- Returns: vcpu fd.

REQ-9 `KVM_RUN` (vCPU-fd):
- Argument: none.
- Behaviour:
  1. Inspect `kvm_run.immediate_exit`: if non-zero, return -EINTR without entering guest.
  2. Read `kvm_run.request_interrupt_window`: if 1, exit when interrupt-window is open.
  3. Enter guest until a trap or pending signal.
  4. On exit: set `kvm_run.exit_reason` and fill matching union arm. Returns 0.
  5. Signals: -EINTR with `EXIT_INTR`.
- The `kvm_run` page is shared mmap; userspace **must not** assume any field is stable across exits.

REQ-10 `struct kvm_run` exit-reason dispatch (per-arm contract):

REQ-10.1 `EXIT_IO`:
- `io.direction` ∈ {`KVM_EXIT_IO_IN`, `KVM_EXIT_IO_OUT`}.
- `io.size` ∈ {1,2,4} bytes.
- `io.port` = I/O port.
- `io.count` = number of transfers (string ops).
- `io.data_offset` = byte offset into kvm_run page where data buffer lives.
- Userspace performs I/O then re-enters `KVM_RUN`.

REQ-10.2 `EXIT_MMIO`:
- `mmio.phys_addr` = guest physical address.
- `mmio.len` ∈ {1..8}.
- `mmio.is_write`: 1 = guest store; data in `mmio.data[0..len]`. 0 = guest load; userspace fills `mmio.data[0..len]` before re-enter.

REQ-10.3 `EXIT_FAIL_ENTRY`:
- `fail_entry.hardware_entry_failure_reason` = arch-specific entry-fail code.
- `fail_entry.cpu` = vcpu id.

REQ-10.4 `EXIT_EXCEPTION`:
- `ex.exception` = vector. `ex.error_code` = arch error.

REQ-10.5 `EXIT_HYPERCALL`:
- `hypercall.nr`, `hypercall.args[6]`, `hypercall.ret`, `hypercall.longmode` / `hypercall.flags`.

REQ-10.6 `EXIT_DEBUG`:
- `debug.arch` = `struct kvm_debug_exit_arch` (arch-specific).

REQ-10.7 `EXIT_HYPERV`:
- `hyperv.type` ∈ {`HYPERV_SYNIC`=1, `HYPERV_HCALL`=2, `HYPERV_SYNDBG`=3}.
- Per-type union: `synic` {msr, control, evt_page, msg_page} / `hcall` {input, result, params[2]} / `syndbg` {msr, control, status, send_page, recv_page, pending_page}.

REQ-10.8 `EXIT_XEN`:
- `xen.type` ∈ {`XEN_HCALL`=1}.
- `xen.u.hcall` {longmode, cpl, input, result, params[6]}.

REQ-10.9 `EXIT_X86_RDMSR` / `EXIT_X86_WRMSR`:
- `msr.error` (user→kernel response: 0/1), `msr.reason` (kernel→user; INVAL|UNKNOWN|FILTER), `msr.index`, `msr.data` (kernel↔user).

REQ-10.10 `EXIT_S390_SIEIC`:
- `s390_sieic.icptcode`, `.ipa`, `.ipb`.

REQ-10.11 `EXIT_ARM_NISV` / `EXIT_ARM_LDST64B`:
- `arm_nisv.esr_iss`, `.fault_ipa`.

REQ-10.12 `EXIT_SYSTEM_EVENT`:
- `system_event.type` ∈ `KVM_SYSTEM_EVENT_*`.
- `system_event.ndata` then either `flags` (legacy union) or `data[16]` (when `KVM_CAP_SYSTEM_EVENT_DATA`).

REQ-10.13 `EXIT_DIRTY_RING_FULL`:
- No per-arm fields; userspace must drain the dirty ring and call `KVM_RESET_DIRTY_RINGS`.

REQ-10.14 `EXIT_RISCV_SBI`:
- `riscv_sbi.extension_id`, `.function_id`, `.args[6]`, `.ret[2]`.

REQ-10.15 `EXIT_RISCV_CSR`:
- `riscv_csr.csr_num`, `.new_value`, `.write_mask`, `.ret_value`.

REQ-10.16 `EXIT_NOTIFY`:
- `notify.flags` (`KVM_NOTIFY_CONTEXT_INVALID = 1 << 0`).

REQ-10.17 `EXIT_MEMORY_FAULT`:
- `memory_fault.flags` (`KVM_MEMORY_EXIT_FLAG_PRIVATE = 1ULL << 3`), `.gpa`, `.size`.

REQ-10.18 `EXIT_INTERNAL_ERROR`:
- `internal.suberror` ∈ {`EMULATION`=1, `SIMUL_EX`=2, `DELIVERY_EV`=3, `UNEXPECTED_EXIT_REASON`=4}.
- If `EMULATION`: payload reinterpreted as `emulation_failure { suberror, ndata, flags, { insn_size, insn_bytes[15] } }`.

REQ-10.19 `EXIT_TDX`:
- `tdx.flags`, `.nr`, plus per-leaf union: `unknown`, `get_quote`, `get_tdvmcall_info`, `setup_event_notify`.

REQ-10.20 `EXIT_ARM_SEA`:
- `arm_sea.flags` (`ARM_SEA_FLAG_GPA_VALID = 1ULL << 0`), `.esr`, `.gva`, `.gpa`.

REQ-10.21 `EXIT_SNP_REQ_CERTS`:
- `snp_req_certs.gpa`, `.npages`, `.ret`.

REQ-10.22 `EXIT_LOONGARCH_IOCSR`:
- `iocsr_io.phys_addr`, `.data[8]`, `.len`, `.is_write` (same shape as MMIO).

REQ-10.23 `EXIT_TPR_ACCESS`:
- `tpr_access.rip`, `.is_write`.

REQ-10.24 `EXIT_S390_TSCH`:
- `s390_tsch.subchannel_id`, `.subchannel_nr`, `.io_int_parm`, `.io_int_word`, `.ipb`, `.dequeued`.

REQ-10.25 `EXIT_S390_STSI`:
- `s390_stsi.addr`, `.ar`, `.fc`, `.sel1`, `.sel2`.

REQ-10.26 `EXIT_IOAPIC_EOI`:
- `eoi.vector`.

REQ-10.27 `EXIT_EPR`:
- `epr.epr`.

REQ-10.28 `EXIT_OSI`:
- `osi.gprs[32]`.

REQ-10.29 `EXIT_PAPR_HCALL`:
- `papr_hcall.nr`, `.ret`, `.args[9]`.

REQ-11 `KVM_GET_REGS` / `KVM_SET_REGS` (arch-specific layout via `struct kvm_regs`).

REQ-12 `KVM_GET_MSRS` / `KVM_SET_MSRS` (x86):
- Argument: `struct kvm_msrs { u32 nmsrs; u32 pad; struct kvm_msr_entry entries[nmsrs]; }`.
- Returns: number of MSRs successfully read/written (may be < nmsrs).

REQ-13 `KVM_SET_CPUID2` / `KVM_GET_CPUID2` / `KVM_GET_SUPPORTED_CPUID` / `KVM_GET_EMULATED_CPUID` / `KVM_GET_SUPPORTED_HV_CPUID`:
- Argument: `struct kvm_cpuid2 { u32 nent; u32 pad; struct kvm_cpuid_entry2 entries[nent]; }`.
- -E2BIG when buffer too small; kernel sets `nent` to required size.

REQ-14 `KVM_GET_DIRTY_LOG`:
- Argument: `struct kvm_dirty_log { u32 slot; u32 padding1; void __user *dirty_bitmap; }`.
- Bitmap: one bit per page; size = ceil(memory_size / PAGE_SIZE / 8).
- Returns 0 / -EINVAL / -ENOENT.

REQ-15 `KVM_CLEAR_DIRTY_LOG` (with `KVM_CAP_MANUAL_DIRTY_LOG_PROTECT2`):
- Argument: `struct kvm_clear_dirty_log { slot, num_pages, first_page, dirty_bitmap }`.

REQ-16 `KVM_IRQ_LINE` / `KVM_IRQ_LINE_STATUS`:
- Argument: `struct kvm_irq_level { union { u32 irq; s32 status; }; u32 level; }`.

REQ-17 `KVM_SET_GSI_ROUTING`:
- Argument: `struct kvm_irq_routing { u32 nr; u32 flags; struct kvm_irq_routing_entry entries[nr]; }`.
- Each entry: gsi, type ∈ `KVM_IRQ_ROUTING_*`, flags, per-type union (irqchip / msi / s390_adapter / hv_sint / xen_evtchn).

REQ-18 `KVM_IRQFD`:
- `kvm_irqfd { fd, gsi, flags, resamplefd, pad[16] }`.
- `flags & KVM_IRQFD_FLAG_DEASSIGN` = unregister; `flags & KVM_IRQFD_FLAG_RESAMPLE` = level mode.

REQ-19 `KVM_IOEVENTFD`:
- `kvm_ioeventfd { datamatch, addr, len, fd, flags, pad[36] }`.
- `flags`: `DATAMATCH | PIO | DEASSIGN | VIRTIO_CCW_NOTIFY`.

REQ-20 `KVM_SIGNAL_MSI`:
- `kvm_msi { address_lo, address_hi, data, flags (MSI_VALID_DEVID), devid, pad[12] }`.

REQ-21 `KVM_ENABLE_CAP`:
- `kvm_enable_cap { cap, flags, args[4], pad[64] }`.
- Some caps are VM-scope (require `KVM_CAP_ENABLE_CAP_VM`), some vCPU-scope.

REQ-22 `KVM_SET_GUEST_DEBUG`:
- `kvm_guest_debug { control, pad, arch }`.
- `control` bits: `KVM_GUESTDBG_ENABLE | _SINGLESTEP` plus arch-specific (x86: `USE_SW_BP | USE_HW_BP | INJECT_DB | INJECT_BP | BLOCKIRQ`).

REQ-23 `KVM_GET_/SET_MP_STATE`: `kvm_mp_state { u32 mp_state }` ∈ `KVM_MP_STATE_*`.

REQ-24 `KVM_TRANSLATE`:
- `kvm_translation { linear_address; out: physical_address, valid, writeable, usermode }`.

REQ-25 `KVM_INTERRUPT`:
- `kvm_interrupt { irq }` — inject IRQ vector.

REQ-26 `KVM_GET_CLOCK` / `KVM_SET_CLOCK`:
- `kvm_clock_data { clock, flags ∈ {TSC_STABLE | REALTIME | HOST_TSC}, realtime, host_tsc, pad[4] }`.

REQ-27 `KVM_REGISTER_COALESCED_MMIO` / `_UNREGISTER_*`:
- `kvm_coalesced_mmio_zone { addr, size, union {pad, pio} }`.
- Ring at `kvm_run` offset (per-VM): `kvm_coalesced_mmio_ring { first, last; kvm_coalesced_mmio entries[] }`.

REQ-28 `KVM_CREATE_DEVICE`:
- `kvm_create_device { type ∈ KVM_DEV_TYPE_*; out fd; flags ∈ {0, KVM_CREATE_DEVICE_TEST} }`.

REQ-29 `KVM_*_DEVICE_ATTR`:
- `kvm_device_attr { flags, group, attr, addr }`.

REQ-30 `KVM_ENABLE_CAP(DIRTY_LOG_RING)` + per-vCPU mmap at `KVM_DIRTY_LOG_PAGE_OFFSET`:
- Ring of `struct kvm_dirty_gfn { u32 flags; u32 slot; u64 offset }`.
- Lifecycle: 00 (Invalid) → 01 (Dirty) → 1X (Reset). Userspace transitions 01→1X after harvest; `KVM_RESET_DIRTY_RINGS` consumes the 1X→00 transition.

REQ-31 `KVM_CREATE_GUEST_MEMFD`:
- `kvm_create_guest_memfd { size, flags ∈ {MMAP, INIT_SHARED}, reserved[6] }`.
- Returns: new fd.

REQ-32 `KVM_SET_MEMORY_ATTRIBUTES`:
- `kvm_memory_attributes { address, size, attributes, flags }`.
- `attributes & KVM_MEMORY_ATTRIBUTE_PRIVATE` ⇒ range backed by guest_memfd.

REQ-33 `KVM_PRE_FAULT_MEMORY`:
- `kvm_pre_fault_memory { gpa, size, flags, padding[5] }`.
- Pre-faults guest-physical range to reduce per-vmexit latency.

REQ-34 `KVM_X86_SET_MSR_FILTER`:
- `kvm_msr_filter { flags (DEFAULT_DENY), ranges[16] }`.
- Each range: `flags (READ | WRITE), nmsrs, base, bitmap *`.

REQ-35 `KVM_MEMORY_ENCRYPT_OP` / `_REG_REGION` / `_UNREG_REGION`:
- Encrypt-API entry; per-`KVM_X86_SEV_VM` / `KVM_X86_SEV_ES_VM` / `KVM_X86_SNP_VM` / `KVM_X86_TDX_VM`.
- `kvm_enc_region { addr, size }`.

REQ-36 `KVM_GET_STATS_FD`:
- Returns: fd whose contents follow `struct kvm_stats_header` + descriptor block + data block.

REQ-37 ABI compatibility:
- Layout, size, padding, and field order of every UAPI struct are frozen.
- Adding a new `KVM_EXIT_*`, `KVM_CAP_*`, or ioctl number requires no change to existing structs.
- Reusing a deprecated number (e.g., `KVM_EXIT_DCR = 15`) is forbidden.

REQ-38 KVM_API_VERSION = 12 — bumping is reserved for fully ABI-breaking redesign.

REQ-39 `HINT_UNSAFE_IN_KVM(immediate_exit)`:
- Field is renamed `immediate_exit__unsafe` in kernel build; UAPI name `immediate_exit` remains stable for userspace.

REQ-40 Coalesced-MMIO ring count:
- `KVM_COALESCED_MMIO_MAX = (PAGE_SIZE - sizeof(struct kvm_coalesced_mmio_ring)) / sizeof(struct kvm_coalesced_mmio)`.

## Acceptance Criteria

- [ ] AC-1: `KVM_GET_API_VERSION` returns exactly 12.
- [ ] AC-2: `KVM_CREATE_VM(0)` returns a VM fd that supports `KVM_CREATE_VCPU`.
- [ ] AC-3: `KVM_GET_VCPU_MMAP_SIZE` returns size ≥ `sizeof(struct kvm_run)`.
- [ ] AC-4: `mmap(vcpu_fd, offset=0, size=KVM_GET_VCPU_MMAP_SIZE)` yields a writable `struct kvm_run`.
- [ ] AC-5: `KVM_SET_USER_MEMORY_REGION(slot=0, gpa=0, size=N, hva=H)` accepts page-aligned ranges, -EINVAL otherwise.
- [ ] AC-6: `KVM_SET_USER_MEMORY_REGION(memory_size=0)` deletes the slot.
- [ ] AC-7: `KVM_RUN` with no pending signal enters guest and returns 0 on exit with `exit_reason` set.
- [ ] AC-8: `EXIT_IO` populates `io.direction/size/port/count/data_offset` correctly.
- [ ] AC-9: `EXIT_MMIO` populates `mmio.phys_addr/data/len/is_write`.
- [ ] AC-10: `EXIT_FAIL_ENTRY.hardware_entry_failure_reason` reports VMX/SVM entry failure code unchanged.
- [ ] AC-11: `EXIT_EXCEPTION.exception` carries vector; `.error_code` matches hardware semantics.
- [ ] AC-12: `EXIT_HYPERCALL.nr` and `.args[6]` round-trip without modification.
- [ ] AC-13: `EXIT_HYPERV.type` ∈ {1,2,3} corresponds to synic/hcall/syndbg union arm.
- [ ] AC-14: `EXIT_XEN.type == 1` populates `xen.u.hcall`.
- [ ] AC-15: `EXIT_X86_{RD,WR}MSR.msr.reason` is one of `INVAL/UNKNOWN/FILTER`.
- [ ] AC-16: `EXIT_S390_SIEIC.icptcode/ipa/ipb` matches SIE interception layout.
- [ ] AC-17: `EXIT_ARM_NISV.esr_iss/fault_ipa` matches ESR_EL2 / HPFAR_EL2 emulation expectations.
- [ ] AC-18: `EXIT_SYSTEM_EVENT.type` ∈ `KVM_SYSTEM_EVENT_*`.
- [ ] AC-19: `EXIT_DIRTY_RING_FULL` requires userspace to drain ring before re-entry.
- [ ] AC-20: `EXIT_RISCV_SBI.extension_id/function_id/args/ret` matches SBI calling convention.
- [ ] AC-21: `EXIT_RISCV_CSR.csr_num/new_value/write_mask/ret_value` matches CSR-emul semantics.
- [ ] AC-22: `EXIT_NOTIFY.flags & KVM_NOTIFY_CONTEXT_INVALID` indicates VMCS corruption.
- [ ] AC-23: `EXIT_INTERNAL_ERROR.suberror == EMULATION` reinterprets union as `emulation_failure`.
- [ ] AC-24: `EXIT_MEMORY_FAULT.flags & PRIVATE` for guest_memfd faults; `.gpa/.size` populated.
- [ ] AC-25: `EXIT_TDX.flags/nr` plus matching union arm by leaf.
- [ ] AC-26: `KVM_CHECK_EXTENSION(KVM_CAP_USER_MEMORY)` returns ≥ 1.
- [ ] AC-27: `KVM_GET_MSR_INDEX_LIST` with insufficient buffer returns -E2BIG and writes required size.
- [ ] AC-28: `KVM_GET_/SET_MSRS` round-trip preserves index/data per entry.
- [ ] AC-29: `KVM_SET_CPUID2`/`_GET_CPUID2` round-trip preserves all entries.
- [ ] AC-30: `KVM_GET_DIRTY_LOG` writes a bitmap whose set bits match pages written by the guest since last call.
- [ ] AC-31: `KVM_SET_GUEST_DEBUG.control` accepts `ENABLE | SINGLESTEP | USE_{SW,HW}_BP | INJECT_{DB,BP} | BLOCKIRQ` (x86).
- [ ] AC-32: `KVM_IRQFD(FLAG_RESAMPLE)` valid only when `KVM_CAP_IRQFD_RESAMPLE` ≥ 1.
- [ ] AC-33: `KVM_CREATE_DEVICE(type=KVM_DEV_TYPE_VFIO)` returns a device fd usable with `_DEVICE_ATTR`.
- [ ] AC-34: `KVM_CREATE_GUEST_MEMFD(size, flags)` returns a writable fd usable as memslot backing.
- [ ] AC-35: `KVM_SET_MEMORY_ATTRIBUTES(attributes & PRIVATE)` flips per-page private status.
- [ ] AC-36: `KVM_PRE_FAULT_MEMORY(gpa, size)` returns 0 on success; faulted pages do not vmexit on first access.
- [ ] AC-37: KVMIO ioctl numbers (0x00–0xd5) used in this header are not aliased to a different cmd.

## Architecture

```
struct KvmUserspaceMemoryRegion {
  slot: u32,
  flags: u32,                        // KVM_MEM_{LOG_DIRTY_PAGES|READONLY|GUEST_MEMFD}
  guest_phys_addr: u64,
  memory_size: u64,
  userspace_addr: u64,
}

struct KvmRun {
  request_interrupt_window: u8,
  immediate_exit: u8,
  padding1: [u8; 6],
  exit_reason: u32,                  // KVM_EXIT_*
  ready_for_interrupt_injection: u8,
  if_flag: u8,
  flags: u16,
  cr8: u64,
  apic_base: u64,
  // s390-only: psw_mask, psw_addr
  exit: ExitPayload,                 // tagged union of all exit-reason arms
  kvm_valid_regs: u64,
  kvm_dirty_regs: u64,
  sync_regs: SyncRegs,               // arch-specific; padded to SYNC_REGS_SIZE_BYTES = 2048
}

enum ExitPayload {
  Hw { hardware_exit_reason: u64 },
  FailEntry { hardware_entry_failure_reason: u64, cpu: u32 },
  Ex { exception: u32, error_code: u32 },
  Io { direction: IoDir, size: u8, port: u16, count: u32, data_offset: u64 },
  Debug { arch: KvmDebugExitArch },
  Mmio { phys_addr: u64, data: [u8; 8], len: u32, is_write: u8 },
  IocsrIo { phys_addr: u64, data: [u8; 8], len: u32, is_write: u8 },
  Hypercall { nr: u64, args: [u64; 6], ret: u64, flags: u64 },
  TprAccess { rip: u64, is_write: u32 },
  S390Sieic { icptcode: u8, ipa: u16, ipb: u32 },
  S390ResetFlags(u64),
  S390UControl { trans_exc_code: u64, pgm_code: u32 },
  Dcr { dcrn: u32, data: u32, is_write: u8 },                /* deprecated */
  Internal { suberror: u32, ndata: u32, data: [u64; 16] },
  EmulationFailure { suberror: u32, ndata: u32, flags: u64, insn_size: u8, insn_bytes: [u8; 15] },
  Osi { gprs: [u64; 32] },
  PaprHcall { nr: u64, ret: u64, args: [u64; 9] },
  S390Tsch { subchannel_id: u16, subchannel_nr: u16, io_int_parm: u32, io_int_word: u32, ipb: u32, dequeued: u8 },
  Epr { epr: u32 },
  SystemEvent { ty: u32, ndata: u32, data: [u64; 16] },
  S390Stsi { addr: u64, ar: u8, fc: u8, sel1: u8, sel2: u16 },
  Eoi { vector: u8 },
  Hyperv(KvmHypervExit),
  ArmNisv { esr_iss: u64, fault_ipa: u64 },
  Msr { error: u8, reason: u32, index: u32, data: u64 },
  Xen(KvmXenExit),
  RiscvSbi { extension_id: ulong, function_id: ulong, args: [ulong; 6], ret: [ulong; 2] },
  RiscvCsr { csr_num: ulong, new_value: ulong, write_mask: ulong, ret_value: ulong },
  Notify { flags: u32 },
  MemoryFault { flags: u64, gpa: u64, size: u64 },
  Tdx { flags: u64, nr: u64, payload: TdxPayload },
  ArmSea { flags: u64, esr: u64, gva: u64, gpa: u64 },
  SnpReqCerts { gpa: u64, npages: u64, ret: u64 },
  Padding([u8; 256]),
}
```

`Kvm::ioctl_dispatch(fd, cmd, arg)`:
1. Look up fd-class (kvm-top, vm, vcpu, device, stats, memfd).
2. Switch on `_IOC_NR(cmd)`; route to `Kvm::top::*`, `Kvm::vm::*`, `Kvm::vcpu::*`, `Kvm::device::*`.
3. Top-level: `0x00` API_VERSION → return 12; `0x01` CREATE_VM → alloc Vm, return fd; `0x02` GET_MSR_INDEX_LIST → copy; `0x03` CHECK_EXTENSION → cap probe; `0x04` GET_VCPU_MMAP_SIZE; `0x05` GET_SUPPORTED_CPUID; `0x09` GET_EMULATED_CPUID; `0x0a` GET_MSR_FEATURE_INDEX_LIST.
4. VM-fd: 0x41 CREATE_VCPU, 0x42 GET_DIRTY_LOG, 0x46/0x49 SET_USER_MEMORY_REGION{,2}, 0x60–0x6a IRQCHIP/PIT, 0x71/0x76/0x79 reinject/irqfd/ioeventfd, 0x7b/0x7c clock, 0xa5 SIGNAL_MSI, 0xba–0xbd encrypt, 0xc0 CLEAR_DIRTY_LOG, 0xc7 RESET_DIRTY_RINGS, 0xd2 SET_MEMORY_ATTRIBUTES, 0xd4 CREATE_GUEST_MEMFD, 0xd5 PRE_FAULT_MEMORY, 0xe0/0xe1/0xe2/0xe3 device.
5. vCPU-fd: 0x80 RUN; 0x81–0xcd registers (regs, sregs, msrs, cpuid2, lapic, fpu, xsave, xcrs, debugregs, vcpu_events, mp_state, sregs2, nested_state, one_reg, signal_mask, ...); 0x9b SET_GUEST_DEBUG; 0xb7 SMI; 0xc2 ARM_VCPU_FINALIZE; 0xcc/0xcd SREGS2; 0xcf XSAVE2.
6. Validate copy_from_user / copy_to_user with explicit sized checks (no implicit object-size relaxation).

`Kvm::run(vcpu) -> KvmRun`:
1. `if vcpu.run.immediate_exit != 0 { return -EINTR }`.
2. Restore guest state from `vcpu.arch_state`.
3. Enter guest until trap.
4. Translate vmexit to `ExitPayload` arm; write to `vcpu.run.exit_reason` + matching union.
5. Save guest state to `vcpu.arch_state`.
6. Return 0.

`Kvm::set_user_memory_region(vm, region) -> Result`:
1. Validate flags ⊆ `{LOG_DIRTY_PAGES, READONLY, GUEST_MEMFD}`.
2. If `flags & GUEST_MEMFD` and ioctl != REGION2: -EINVAL.
3. If memory_size == 0: delete slot.
4. Else: page-align checks (gpa, size, hva).
5. Insert/update slot via `vm.memslots`.
6. If `LOG_DIRTY_PAGES` set: allocate dirty bitmap.

`Kvm::check_extension(cap) -> i32`:
- Switch on cap; return 0 (unsupported), 1, or n (per-cap value).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `api_version_const` | INVARIANT | per-`KVM_GET_API_VERSION`: returns exactly 12. |
| `kvm_run_size_within_mmap` | INVARIANT | per-`mmap(vcpu_fd)`: sizeof(kvm_run) + sync_regs ≤ `KVM_GET_VCPU_MMAP_SIZE`. |
| `exit_union_arm_consistent` | INVARIANT | per-`KVM_RUN`: written union arm matches `exit_reason`. |
| `memslot_alignment` | INVARIANT | per-`KVM_SET_USER_MEMORY_REGION`: gpa, size, hva are PAGE_SIZE-aligned. |
| `memslot_flags_valid` | INVARIANT | per-`KVM_SET_USER_MEMORY_REGION`: flags ⊆ legal mask. |
| `cpuid2_nent_bound` | INVARIANT | per-`KVM_*_CPUID2`: nent ≤ buffer capacity; else -E2BIG. |
| `msr_list_nmsrs_bound` | INVARIANT | per-`KVM_GET_MSR_INDEX_LIST`: nmsrs ≤ buffer. |
| `dirty_ring_state_transitions` | INVARIANT | per-`kvm_dirty_gfn.flags`: 00→01→1X→00 only. |
| `irq_routing_entry_type_known` | INVARIANT | per-`KVM_IRQ_ROUTING_*`: type ∈ {1,2,3,4,5}. |
| `guestdbg_control_mask` | INVARIANT | per-`KVM_SET_GUEST_DEBUG`: control bits ⊆ legal arch mask. |
| `ioctl_class_KVMIO` | INVARIANT | per-ioctl: `_IOC_TYPE(cmd) == 0xAE`. |
| `immediate_exit_obeyed` | INVARIANT | per-`KVM_RUN`: `kvm_run.immediate_exit == 1` ⇒ returns -EINTR before guest entry. |

### Layer 2: TLA+

`uapi/kvm-ioctl.tla`:
- Models vm/vcpu fd lifetime + ioctl dispatch + KVM_RUN state machine.
- Properties:
  - `safety_api_version_stable` — per-`KVM_GET_API_VERSION` always returns 12.
  - `safety_memslot_no_overlap` — per-`KVM_SET_USER_MEMORY_REGION`: new slot does not overlap existing slot's gpa range unless replacing same slot id.
  - `safety_dirty_ring_no_reorder` — entries harvested in sequence.
  - `safety_exit_reason_implies_arm` — per `KVM_RUN`: exit_reason value uniquely determines which union arm is meaningful.
  - `liveness_run_terminates` — every `KVM_RUN` eventually returns (signal, vmexit, fault, or shutdown).
  - `safety_close_releases_children` — close(vm_fd) releases all vcpu/device/memslot resources.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `KvmRun::exit_reason ∈ KVM_EXIT_*` post-RUN | `Kvm::vcpu::run` |
| `KvmUserspaceMemoryRegion::flags & ~MASK == 0` | `Kvm::vm::set_user_memory_region` |
| `KvmCpuid2::nent ≤ buf_cap` ∨ `errno == E2BIG` | `Kvm::vcpu::set_cpuid2` |
| `KvmDirtyGfn::flags ⊆ {DIRTY, RESET}` | dirty-ring producer |
| `KvmIrqRoutingEntry::type ∈ {1..5}` | `Kvm::vm::set_gsi_routing` |
| `KvmGuestDebug::control ⊆ ENABLE|SINGLESTEP|x86mask` | `Kvm::vcpu::set_guest_debug` |
| `KvmMsi::flags & ~MSI_VALID_DEVID == 0` | `Kvm::vm::signal_msi` |
| `KvmIoeventfd::flags ⊆ VALID_MASK` | `Kvm::vm::ioeventfd` |
| `KvmIrqfd::flags ⊆ {DEASSIGN, RESAMPLE}` | `Kvm::vm::irqfd` |
| `KvmMemoryAttributes::attributes ⊆ {PRIVATE}` | `Kvm::vm::set_memory_attributes` |

### Layer 4: Verus/Creusot functional

Per `Documentation/virt/kvm/api.rst`:
- ioctl dispatch table (number → handler) matches upstream.
- struct layouts byte-equivalent: offset and size of every UAPI field equal to upstream `gcc -E` expansion on the same arch.
- KVM_RUN semantic equivalence with upstream vmx / svm / arm64 trap-and-emulate path for at least: IO, MMIO, EXCEPTION, HLT, HYPERCALL, FAIL_ENTRY, INTERNAL_ERROR/EMULATION, SYSTEM_EVENT, MEMORY_FAULT, DIRTY_RING_FULL.
- KVM_CHECK_EXTENSION numeric IDs of all listed `KVM_CAP_*` are preserved.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

KVM-UAPI reinforcement:

- **Per-ioctl class strict** — `_IOC_TYPE(cmd) == KVMIO (0xAE)` enforced; non-class ioctls rejected.
- **Per-cmd-number switch exhaustive** — unknown numbers return -ENOTTY (never silently succeed).
- **Per-fd-class routing** — top/vm/vcpu/device/stats/memfd ioctls only valid on matching fd-class.
- **Per-copy_from_user with explicit length** — every UAPI struct copied with `sizeof(struct)` (no truncation, no over-read).
- **Per-kvm_run union arm written exactly one** — defense against per-confused-deputy reads.
- **Per-`immediate_exit` honored early** — defense against per-late-cancel guest-resource leak.
- **Per-memslot alignment + non-overlap + sanity-checked hva** — defense against per-malicious-mapping.
- **Per-`KVM_MEM_GUEST_MEMFD` gated on _REGION2** — defense against per-feature smuggle.
- **Per-dirty-ring index acq/rel ordering** — defense against per-out-of-order-harvest.
- **Per-coalesced-MMIO ring count derived (not user-supplied)** — defense against per-over-allocation.
- **Per-deprecated EXIT_DCR rejected post-arch-cutover** — defense against per-stale-userspace.
- **Per-`HINT_UNSAFE_IN_KVM` renamed-in-kernel** — defense against per-TOCTOU on `immediate_exit`.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF/USERCOPY** — every `kvm_userspace_memory_region.userspace_addr` dereference and every `copy_from_user`/`copy_to_user` on `kvm_run`, `kvm_msrs`, `kvm_cpuid2`, `kvm_dirty_log`, `kvm_irq_routing` payloads goes through hardened-usercopy with explicit sized bounds; no kernel-pointer-sized reads of arbitrary user buffers; SMAP/SMEP must be on for `KVM_RUN` to enter.
- **GRKERNSEC_HIDESYM** — KVM error klog messages (`fail_entry.hardware_entry_failure_reason`, internal errors, emulation_failure.insn_bytes) never leak host kernel addresses; per-vcpu printk-rate-limited.
- **GRKERNSEC_PERF_HARDEN** — `KVM_GET_STATS_FD` + `kvm_stats_desc` exposed only to `CAP_PERFMON` or owner; `perf_event_open` on KVM events restricted; binary-stats name strings sanitized.
- **PAX_RANDKSTACK** — every `KVM_RUN` re-randomizes the per-task kernel-stack base before guest entry; ensures vmexit handler return-address cannot be probed via repeated `EXIT_DEBUG` / `EXIT_INTERNAL_ERROR`.
- **PAX_KERNEXEC** — KVM guest-state save/restore preserves W^X for host: VMCS / VMCB / arch-state buffers live in NX kernel pages; never patched at runtime; `KVM_SET_NESTED_STATE` validates that loaded state cannot make any host page W+X.
- **GRKERNSEC_HARDEN_PTRACE** — `KVM_GET_REGS` / `KVM_SET_REGS` / `KVM_GET_SREGS` / `KVM_SET_SREGS` / `KVM_GET_MSRS` / `KVM_GET_LAPIC` permitted only by the process that owns the vcpu fd; cross-process access via `/proc/<pid>/fd/<vcpu>` blocked unless `gr_ptrace_allowed(target)` returns true.
- **CAP_SYS_ADMIN gate** — opening `/dev/kvm` requires per-policy capability (default: `CAP_SYS_ADMIN` or membership in `kvm` group with grsec policy allow); `KVM_CREATE_VM` denied to non-priv processes by default; per-cgroup quota on number of VMs and total guest memory.
- **GRKERNSEC_KMEM** — although KVM does not expose `/dev/mem` / `/dev/kmem` itself, the same principle applies: guest_memfd backing pages are unmappable from `/proc/<host>/mem`; `KVM_SET_MEMORY_ATTRIBUTES(PRIVATE)` removes pages from the host direct-map; `KVM_PRE_FAULT_MEMORY` cannot probe outside the calling vm's memslot universe.
- **Seccomp landlock-style restrictions** — recommended user-side filter: allow only `ioctl(kvm_fd, KVM_*, ...)` with cmd ∈ {KVM_GET_API_VERSION, KVM_CREATE_VM, KVM_CHECK_EXTENSION, KVM_GET_VCPU_MMAP_SIZE} on the top-level fd; per-fd-class restrict-set enforced at policy layer so a compromised vmm cannot escalate to unrelated ioctl numbers.
- **Per-`KVM_MEMORY_ENCRYPT_OP` opaque-blob bounded** — defense against per-SEV/SNP command smuggle; subcommand allow-list enforced.
- **Per-`KVM_X86_SET_MSR_FILTER` immutable-after-first-set** — defense against per-tocttou MSR-policy bypass.
- **Per-`KVM_SET_GSI_ROUTING` entry-count bounded** — defense against per-OOM via huge routing table.
- **Per-`KVM_CREATE_DEVICE_TEST` returns no fd** — defense against per-feature-probe escalation.
- **Per-`KVM_DEV_VFIO_FILE` requires DAC ownership of vfio fd** — defense against per-cross-process device hijack.
- **Per-`immediate_exit` bit cleared on `KVM_SET_NESTED_STATE`** — defense against per-stale-bit guest-leak.
- **Per-`KVM_RUN` `flags` validated** — defense against per-userspace-corruption of host-only field.
- **Per-`request_interrupt_window` rejected when LAPIC in-kernel disabled** — defense against per-confused-irq-injection.
- **Per-dirty-ring page-offset = `KVM_DIRTY_LOG_PAGE_OFFSET`** — defense against per-overlay attack on `kvm_run`.
- **Per-`KVM_MEM_LOG_DIRTY_PAGES` + `KVM_CAP_DIRTY_LOG_RING` mutual-exclusion enforced per VM** — defense against per-double-bookkeeping disagreement.
- **Per-`KVM_X86_QUIRK_*` opt-out via `KVM_CAP_DISABLE_QUIRKS2`** — defense against per-legacy-behavior fingerprint.
- **Per-`KVM_GET_SUPPORTED_HV_CPUID` requires KVM_CAP_HYPERV_CPUID** — defense against per-feature-leak to non-Hyper-V guests.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- Arch-specific UAPI: `arch/x86/include/uapi/asm/kvm.h` (covered partially here for `KVM_IRQCHIP_*`, `KVM_GUESTDBG_*`, `KVM_X86_QUIRK_*`, `KVM_X86_*_VM`, `kvm_msrs`, `kvm_msr_list`, `kvm_cpuid2`) — full coverage deferred to `uapi/headers/asm-x86-kvm.md`.
- Arch-specific UAPI: arm64 (`arch/arm64/include/uapi/asm/kvm.h`), s390 (`arch/s390/include/uapi/asm/kvm.h`), ppc, riscv, mips, loongarch — separate Tier-5 docs.
- `KVM_REQ_*` request-bit semantics (kernel-internal, not UAPI).
- `Documentation/virt/kvm/api.rst` text — referenced but not embedded.
- KVM device-attr namespaces per device type — deferred to per-device-type Tier-5 docs.
- Implementation code (`virt/kvm/kvm_main.c`, `arch/x86/kvm/*.c`).
