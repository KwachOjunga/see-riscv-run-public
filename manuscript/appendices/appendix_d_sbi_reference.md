# Appendix D. SBI Call Reference

**Supervisor Binary Interface (SBI) Quick Reference**

---

This appendix provides a comprehensive reference for RISC-V SBI (Supervisor Binary Interface) calls. SBI defines the standard interface between supervisor mode (S-mode) software and machine mode (M-mode) firmware, enabling portable operating systems across different RISC-V platforms.

---

## D.1 SBI Calling Convention

### Register Usage

**Input Registers**:

| Register | Purpose |
|----------|---------|
| **a7** | Extension ID (EID) |
| **a6** | Function ID (FID) |
| **a0** | Parameter 0 / Return value |
| **a1** | Parameter 1 / Return value (optional) |
| **a2** | Parameter 2 |
| **a3** | Parameter 3 |
| **a4** | Parameter 4 |
| **a5** | Parameter 5 |

**Output Registers**:

| Register | Purpose |
|----------|---------|
| **a0** | Error code (0 = success, negative = error) |
| **a1** | Return value (function-specific) |

**Preserved Registers**: All registers except a0 and a1 are preserved across SBI calls.

### Invocation

```c
// S-mode code invokes SBI using ecall instruction
register unsigned long a0 asm("a0") = param0;
register unsigned long a1 asm("a1") = param1;
register unsigned long a6 asm("a6") = function_id;
register unsigned long a7 asm("a7") = extension_id;

asm volatile("ecall"
             : "+r"(a0), "+r"(a1)
             : "r"(a6), "r"(a7)
             : "memory");

// a0 contains error code, a1 contains return value
```

---

## D.2 SBI Error Codes

| Code | Name | Description |
|------|------|-------------|
| 0 | SBI_SUCCESS | Operation completed successfully |
| -1 | SBI_ERR_FAILED | Operation failed |
| -2 | SBI_ERR_NOT_SUPPORTED | Function not supported |
| -3 | SBI_ERR_INVALID_PARAM | Invalid parameter |
| -4 | SBI_ERR_DENIED | Permission denied |
| -5 | SBI_ERR_INVALID_ADDRESS | Invalid address |
| -6 | SBI_ERR_ALREADY_AVAILABLE | Resource already available |
| -7 | SBI_ERR_ALREADY_STARTED | Already started |
| -8 | SBI_ERR_ALREADY_STOPPED | Already stopped |

---

## D.3 Base Extension (EID = 0x10)

**Purpose**: Query SBI implementation details and supported extensions.

### D.3.1 Get SBI Specification Version (FID = 0)

**Returns**: SBI specification version.

- **a1[31:24]**: Major version
- **a1[23:0]**: Minor version

```c
long sbi_get_spec_version(void) {
    register unsigned long a0 asm("a0");
    register unsigned long a1 asm("a1");
    register unsigned long a6 asm("a6") = 0;
    register unsigned long a7 asm("a7") = 0x10;
    
    asm volatile("ecall" : "=r"(a0), "=r"(a1) : "r"(a6), "r"(a7) : "memory");
    return a1;  // Version in a1
}
```

### D.3.2 Get SBI Implementation ID (FID = 1)

**Returns**: Implementation ID.

| ID | Implementation |
|----|----------------|
| 0 | Berkeley Boot Loader (BBL) |
| 1 | OpenSBI |
| 2 | Xvisor |
| 3 | KVM |
| 4 | RustSBI |
| 5 | Diosix |

### D.3.3 Get SBI Implementation Version (FID = 2)

**Returns**: Implementation-specific version number.

### D.3.4 Probe SBI Extension (FID = 3)

**Parameters**:

- **a0**: Extension ID to probe

**Returns**:

- **a1**: 0 = not available, 1 = available

```c
long sbi_probe_extension(long extension_id) {
    register unsigned long a0 asm("a0") = extension_id;
    register unsigned long a1 asm("a1");
    register unsigned long a6 asm("a6") = 3;
    register unsigned long a7 asm("a7") = 0x10;
    
    asm volatile("ecall" : "+r"(a0), "=r"(a1) : "r"(a6), "r"(a7) : "memory");
    return a1;
}
```

### D.3.5 Get Machine Vendor ID (FID = 4)

**Returns**: mvendorid CSR value.

### D.3.6 Get Machine Architecture ID (FID = 5)

**Returns**: marchid CSR value.

### D.3.7 Get Machine Implementation ID (FID = 6)

**Returns**: mimpid CSR value.

---

## D.4 Timer Extension (EID = 0x54494D45 "TIME")

**Purpose**: Program timer interrupts.

### D.4.1 Set Timer (FID = 0)

**Parameters**:

- **a0**: Timer value (absolute time in ticks)

**Description**: Programs the timer to fire at the specified time. Clears pending timer interrupt.

```c
void sbi_set_timer(uint64_t stime_value) {
    register unsigned long a0 asm("a0") = stime_value;
    register unsigned long a6 asm("a6") = 0;
    register unsigned long a7 asm("a7") = 0x54494D45;
    
    asm volatile("ecall" : "+r"(a0) : "r"(a6), "r"(a7) : "memory");
}

// Usage: Set timer to fire in 1 second (assuming 10 MHz timebase)
uint64_t current_time;
asm volatile("rdtime %0" : "=r"(current_time));
sbi_set_timer(current_time + 10000000);
```

---

## D.5 IPI Extension (EID = 0x735049 "sPI")

**Purpose**: Send inter-processor interrupts.

### D.5.1 Send IPI (FID = 0)

**Parameters**:

- **a0**: Hart mask (bitmap of target harts)
- **a1**: Hart mask base (base hart ID for the mask)

**Description**: Sends supervisor software interrupt to specified harts.

```c
long sbi_send_ipi(unsigned long hart_mask, unsigned long hart_mask_base) {
    register unsigned long a0 asm("a0") = hart_mask;
    register unsigned long a1 asm("a1") = hart_mask_base;
    register unsigned long a6 asm("a6") = 0;
    register unsigned long a7 asm("a7") = 0x735049;

    asm volatile("ecall" : "+r"(a0), "+r"(a1) : "r"(a6), "r"(a7) : "memory");
    return a0;
}

// Usage: Send IPI to harts 1, 2, 3
sbi_send_ipi(0b1110, 0);  // Bits 1, 2, 3 set, base = 0

// Send IPI to hart 65 (bit 1 of mask, base = 64)
sbi_send_ipi(0b10, 64);
```

---

## D.6 RFENCE Extension (EID = 0x52464E43 "RFNC")

**Purpose**: Remote fence operations for TLB and instruction cache synchronization.

### D.6.1 Remote FENCE.I (FID = 0)

**Parameters**:

- **a0**: Hart mask
- **a1**: Hart mask base

**Description**: Execute FENCE.I on remote harts.

```c
long sbi_remote_fence_i(unsigned long hart_mask, unsigned long hart_mask_base) {
    register unsigned long a0 asm("a0") = hart_mask;
    register unsigned long a1 asm("a1") = hart_mask_base;
    register unsigned long a6 asm("a6") = 0;
    register unsigned long a7 asm("a7") = 0x52464E43;

    asm volatile("ecall" : "+r"(a0), "+r"(a1) : "r"(a6), "r"(a7) : "memory");
    return a0;
}
```

### D.6.2 Remote SFENCE.VMA (FID = 1)

**Parameters**:

- **a0**: Hart mask
- **a1**: Hart mask base
- **a2**: Start address (virtual address)
- **a3**: Size (number of pages)

**Description**: Execute SFENCE.VMA on remote harts for specified address range.

```c
long sbi_remote_sfence_vma(unsigned long hart_mask, unsigned long hart_mask_base,
                           unsigned long start_addr, unsigned long size) {
    register unsigned long a0 asm("a0") = hart_mask;
    register unsigned long a1 asm("a1") = hart_mask_base;
    register unsigned long a2 asm("a2") = start_addr;
    register unsigned long a3 asm("a3") = size;
    register unsigned long a6 asm("a6") = 1;
    register unsigned long a7 asm("a7") = 0x52464E43;

    asm volatile("ecall" : "+r"(a0), "+r"(a1)
                 : "r"(a2), "r"(a3), "r"(a6), "r"(a7) : "memory");
    return a0;
}

// Usage: Flush TLB for address range on all harts
sbi_remote_sfence_vma(~0UL, 0, 0x80000000, 4096);  // Flush 1 page
```

### D.6.3 Remote SFENCE.VMA with ASID (FID = 2)

**Parameters**:

- **a0**: Hart mask
- **a1**: Hart mask base
- **a2**: Start address
- **a3**: Size
- **a4**: ASID

**Description**: Execute SFENCE.VMA with ASID on remote harts.

```c
long sbi_remote_sfence_vma_asid(unsigned long hart_mask, unsigned long hart_mask_base,
                                unsigned long start_addr, unsigned long size,
                                unsigned long asid) {
    register unsigned long a0 asm("a0") = hart_mask;
    register unsigned long a1 asm("a1") = hart_mask_base;
    register unsigned long a2 asm("a2") = start_addr;
    register unsigned long a3 asm("a3") = size;
    register unsigned long a4 asm("a4") = asid;
    register unsigned long a6 asm("a6") = 2;
    register unsigned long a7 asm("a7") = 0x52464E43;

    asm volatile("ecall" : "+r"(a0), "+r"(a1)
                 : "r"(a2), "r"(a3), "r"(a4), "r"(a6), "r"(a7) : "memory");
    return a0;
}
```

### D.6.4 Remote HFENCE.GVMA (FID = 3)

**Parameters**:

- **a0**: Hart mask
- **a1**: Hart mask base
- **a2**: Guest physical address
- **a3**: Size

**Description**: Execute HFENCE.GVMA on remote harts (hypervisor extension).

### D.6.5 Remote HFENCE.GVMA with VMID (FID = 4)

**Parameters**:

- **a0**: Hart mask
- **a1**: Hart mask base
- **a2**: Guest physical address
- **a3**: Size
- **a4**: VMID

**Description**: Execute HFENCE.GVMA with VMID on remote harts.

### D.6.6 Remote HFENCE.VVMA (FID = 5)

**Parameters**:

- **a0**: Hart mask
- **a1**: Hart mask base
- **a2**: Guest virtual address
- **a3**: Size

**Description**: Execute HFENCE.VVMA on remote harts.

### D.6.7 Remote HFENCE.VVMA with ASID (FID = 6)

**Parameters**:

- **a0**: Hart mask
- **a1**: Hart mask base
- **a2**: Guest virtual address
- **a3**: Size
- **a4**: ASID

**Description**: Execute HFENCE.VVMA with ASID on remote harts.

---

## D.7 Hart State Management Extension (EID = 0x48534D "HSM")

**Purpose**: Manage hart lifecycle (start, stop, suspend).

### D.7.1 Hart Start (FID = 0)

**Parameters**:

- **a0**: Hart ID
- **a1**: Start address (physical address)
- **a2**: Opaque parameter (passed to hart in a1)

**Returns**: SBI_SUCCESS or error code

**Description**: Start the specified hart at the given address.

```c
long sbi_hart_start(unsigned long hartid, unsigned long start_addr,
                    unsigned long opaque) {
    register unsigned long a0 asm("a0") = hartid;
    register unsigned long a1 asm("a1") = start_addr;
    register unsigned long a2 asm("a2") = opaque;
    register unsigned long a6 asm("a6") = 0;
    register unsigned long a7 asm("a7") = 0x48534D;

    asm volatile("ecall" : "+r"(a0), "+r"(a1)
                 : "r"(a2), "r"(a6), "r"(a7) : "memory");
    return a0;
}

// Usage: Start hart 1 at address 0x80200000
sbi_hart_start(1, 0x80200000, 0);
```

### D.7.2 Hart Stop (FID = 1)

**Parameters**: None

**Returns**: Does not return on success

**Description**: Stop the current hart. Hart enters stopped state.

```c
void sbi_hart_stop(void) {
    register unsigned long a6 asm("a6") = 1;
    register unsigned long a7 asm("a7") = 0x48534D;

    asm volatile("ecall" : : "r"(a6), "r"(a7) : "memory");
}
```

### D.7.3 Hart Get Status (FID = 2)

**Parameters**:

- **a0**: Hart ID

**Returns**:

- **a1**: Hart status

**Hart Status Values**:

| Value | Status |
|-------|--------|
| 0 | STARTED |
| 1 | STOPPED |
| 2 | START_PENDING |
| 3 | STOP_PENDING |
| 4 | SUSPENDED |
| 5 | SUSPEND_PENDING |
| 6 | RESUME_PENDING |

```c
long sbi_hart_get_status(unsigned long hartid) {
    register unsigned long a0 asm("a0") = hartid;
    register unsigned long a1 asm("a1");
    register unsigned long a6 asm("a6") = 2;
    register unsigned long a7 asm("a7") = 0x48534D;

    asm volatile("ecall" : "+r"(a0), "=r"(a1) : "r"(a6), "r"(a7) : "memory");
    return a1;
}
```

### D.7.4 Hart Suspend (FID = 3)

**Parameters**:

- **a0**: Suspend type
- **a1**: Resume address
- **a2**: Opaque parameter

**Suspend Types**:

| Value | Type |
|-------|------|
| 0x00000000 | RETENTIVE (retain state, low latency) |
| 0x80000000 | NON_RETENTIVE (lose state, save/restore required) |

---

## D.8 System Reset Extension (EID = 0x53525354 "SRST")

**Purpose**: System-wide reset and shutdown.

### D.8.1 System Reset (FID = 0)

**Parameters**:

- **a0**: Reset type
- **a1**: Reset reason

**Returns**: Does not return on success

**Reset Types**:

| Value | Type |
|-------|------|
| 0x00000000 | SHUTDOWN |
| 0x00000001 | COLD_REBOOT |
| 0x00000002 | WARM_REBOOT |

**Reset Reasons**:

| Value | Reason |
|-------|--------|
| 0x00000000 | NO_REASON |
| 0x00000001 | SYSTEM_FAILURE |

```c
void sbi_system_reset(unsigned long reset_type, unsigned long reset_reason) {
    register unsigned long a0 asm("a0") = reset_type;
    register unsigned long a1 asm("a1") = reset_reason;
    register unsigned long a6 asm("a6") = 0;
    register unsigned long a7 asm("a7") = 0x53525354;

    asm volatile("ecall" : : "r"(a0), "r"(a1), "r"(a6), "r"(a7) : "memory");
    __builtin_unreachable();
}

// Usage: Reboot the system
#define SBI_RESET_TYPE_COLD_REBOOT 1
#define SBI_RESET_REASON_NO_REASON 0
sbi_system_reset(SBI_RESET_TYPE_COLD_REBOOT, SBI_RESET_REASON_NO_REASON);
```

---

## D.9 Performance Monitoring Unit Extension (EID = 0x504D55 "PMU")

**Purpose**: Configure and read performance counters.

### D.9.1 Get Number of Counters (FID = 0)

**Returns**:

- **a1**: Number of counters

### D.9.2 Get Counter Info (FID = 1)

**Parameters**:

- **a0**: Counter index

**Returns**:

- **a1**: Counter info

### D.9.3 Configure Matching Counters (FID = 2)

**Parameters**:

- **a0**: Counter index base
- **a1**: Counter mask
- **a2**: Config flags
- **a3**: Event index
- **a4**: Event data

**Returns**: Number of counters configured

### D.9.4 Start Counters (FID = 3)

**Parameters**:

- **a0**: Counter index base
- **a1**: Counter mask
- **a2**: Start flags
- **a3**: Initial value

### D.9.5 Stop Counters (FID = 4)

**Parameters**:

- **a0**: Counter index base
- **a1**: Counter mask
- **a2**: Stop flags

### D.9.6 Read Firmware Counter (FID = 5)

**Parameters**:

- **a0**: Counter index

**Returns**:

- **a1**: Counter value

---

## D.10 Legacy Extensions (Deprecated)

**Note**: These extensions are deprecated but still widely used for compatibility.

### D.10.1 Console Putchar (EID = 0x01)

**Parameters**:

- **a0**: Character to output

```c
void sbi_console_putchar(int ch) {
    register unsigned long a0 asm("a0") = ch;
    register unsigned long a7 asm("a7") = 0x01;

    asm volatile("ecall" : "+r"(a0) : "r"(a7) : "memory");
}
```

### D.10.2 Console Getchar (EID = 0x02)

**Returns**:

- **a0**: Character read, or -1 if no character available

```c
int sbi_console_getchar(void) {
    register unsigned long a0 asm("a0");
    register unsigned long a7 asm("a7") = 0x02;

    asm volatile("ecall" : "=r"(a0) : "r"(a7) : "memory");
    return a0;
}
```

### D.10.3 Legacy Set Timer (EID = 0x00)

**Parameters**:

- **a0**: Timer value

**Note**: Use Timer Extension (0x54494D45) instead.

### D.10.4 Legacy Clear IPI (EID = 0x03)

**Note**: Deprecated. Clear sip.SSIP bit directly.

### D.10.5 Legacy Send IPI (EID = 0x04)

**Parameters**:

- **a0**: Hart mask pointer

**Note**: Use IPI Extension (0x735049) instead.

### D.10.6 Legacy Remote FENCE.I (EID = 0x05)

**Parameters**:

- **a0**: Hart mask pointer

**Note**: Use RFENCE Extension (0x52464E43) instead.

### D.10.7 Legacy Remote SFENCE.VMA (EID = 0x06)

**Parameters**:

- **a0**: Hart mask pointer
- **a1**: Start address
- **a2**: Size

**Note**: Use RFENCE Extension instead.

### D.10.8 Legacy Remote SFENCE.VMA with ASID (EID = 0x07)

**Parameters**:

- **a0**: Hart mask pointer
- **a1**: Start address
- **a2**: Size
- **a3**: ASID

**Note**: Use RFENCE Extension instead.

### D.10.9 Legacy System Shutdown (EID = 0x08)

**Note**: Use System Reset Extension (0x53525354) instead.

---

## D.11 Extension ID Summary

| EID | Name | Description |
|-----|------|-------------|
| 0x10 | BASE | Base extension (version, probe) |
| 0x54494D45 | TIME | Timer programming |
| 0x735049 | sPI | Inter-processor interrupts |
| 0x52464E43 | RFNC | Remote fence operations |
| 0x48534D | HSM | Hart state management |
| 0x53525354 | SRST | System reset |
| 0x504D55 | PMU | Performance monitoring |
| 0x4442434E | DBCN | Debug console |
| 0x53555350 | SUSP | System suspend |
| 0x43505043 | CPPC | Collaborative Processor Performance Control |
| 0x4E41434C | NACL | Nested Acceleration |

---

## D.12 Common Usage Patterns

### Early Boot Console Output

```c
void early_printk(const char *str) {
    while (*str) {
        if (*str == '\n')
            sbi_console_putchar('\r');
        sbi_console_putchar(*str++);
    }
}
```

### Timer-based Scheduling

```c
void setup_timer_interrupt(uint64_t interval_us) {
    uint64_t current_time;
    asm volatile("rdtime %0" : "=r"(current_time));

    // Assuming 10 MHz timebase (100 ns per tick)
    uint64_t ticks = interval_us * 10;
    sbi_set_timer(current_time + ticks);

    // Enable supervisor timer interrupt
    csr_set(sie, SIE_STIE);
}
```

### Multi-core Synchronization

```c
void flush_tlb_all_harts(void) {
    // Flush TLB on all harts
    sbi_remote_sfence_vma(~0UL, 0, 0, ~0UL);
}

void wake_up_secondary_harts(void) {
    for (int i = 1; i < num_harts; i++) {
        sbi_hart_start(i, (unsigned long)secondary_start, 0);
    }
}
```

### System Shutdown

```c
void system_poweroff(void) {
    sbi_system_reset(0, 0);  // SHUTDOWN, NO_REASON
    while (1);  // Should never reach here
}

void system_reboot(void) {
    sbi_system_reset(1, 0);  // COLD_REBOOT, NO_REASON
    while (1);
}
```

---

## D.13 References

- **SBI Specification**: https://github.com/riscv-non-isa/riscv-sbi-doc
- **OpenSBI Documentation**: https://github.com/riscv-software-src/opensbi
- **Linux RISC-V SBI Implementation**: arch/riscv/kernel/sbi.c

---

<!-- METADATA (for authoring only, remove before publication)

## Appendix Metadata

**Appendix**: D - SBI Call Reference
**Version**: Draft v1p0
**Word Count**: ~2,200 words
**Tables**: 10+
**Code Examples**: 20+

-->
