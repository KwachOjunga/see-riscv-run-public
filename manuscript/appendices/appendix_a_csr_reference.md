# Appendix A. CSR Reference

**Control and Status Register Quick Reference**

---

This appendix provides a comprehensive reference for RISC-V Control and Status Registers (CSRs). CSRs control processor behavior, report status, and provide access to privileged functionality. Each CSR has a 12-bit address and is accessed using dedicated CSR instructions (CSRRW, CSRRS, CSRRC, and their immediate variants).

---

## A.1 CSR Address Space Organization

CSR addresses are 12 bits, organized as follows:

```
Bits [11:10]: Privilege Level
  00 = User/Unprivileged
  01 = Supervisor
  10 = Hypervisor (reserved in base spec)
  11 = Machine

Bits [9:8]: Read/Write Access
  00 = Read/Write
  01 = Read/Write
  10 = Read/Write
  11 = Read-Only

Bits [7:0]: Register Number
```

**Access Rules**:

- Accessing a CSR from insufficient privilege level causes an illegal instruction exception
- Writing to a read-only CSR (bits [11:10] = 11) causes an illegal instruction exception
- Unimplemented CSRs may read as zero or cause an exception (implementation-defined)

---

## A.2 Machine-Level CSRs (M-mode)

### Machine Information Registers

| CSR | Address | R/W | Description |
|-----|---------|-----|-------------|
| **mvendorid** | 0xF11 | RO | Vendor ID (JEDEC manufacturer ID) |
| **marchid** | 0xF12 | RO | Architecture ID (implementation-specific) |
| **mimpid** | 0xF13 | RO | Implementation ID (version number) |
| **mhartid** | 0xF14 | RO | Hardware thread ID (unique per hart) |
| **mconfigptr** | 0xF15 | RO | Pointer to configuration data structure |

**Usage**: These read-only CSRs identify the processor implementation. Software can use them to detect features, apply workarounds, or report system information.

**Example**:

```assembly
csrr t0, mhartid        # Read hart ID
csrr t1, mvendorid      # Read vendor ID
```

---

### Machine Trap Setup

| CSR | Address | R/W | Description |
|-----|---------|-----|-------------|
| **mstatus** | 0x300 | RW | Machine status register |
| **misa** | 0x301 | RW | ISA and extensions (may be read-only) |
| **medeleg** | 0x302 | RW | Exception delegation to S-mode |
| **mideleg** | 0x303 | RW | Interrupt delegation to S-mode |
| **mie** | 0x304 | RW | Machine interrupt enable |
| **mtvec** | 0x305 | RW | Machine trap-handler base address |
| **mcounteren** | 0x306 | RW | Counter enable for S-mode |
| **mstatush** | 0x310 | RW | Additional machine status (RV32 only) |

---

### Machine Trap Handling

| CSR | Address | R/W | Description |
|-----|---------|-----|-------------|
| **mscratch** | 0x340 | RW | Scratch register for M-mode trap handlers |
| **mepc** | 0x341 | RW | Machine exception program counter |
| **mcause** | 0x342 | RW | Machine trap cause |
| **mtval** | 0x343 | RW | Machine bad address or instruction |
| **mip** | 0x344 | RW | Machine interrupt pending |
| **mtinst** | 0x34A | RW | Machine trap instruction (transformed) |
| **mtval2** | 0x34B | RW | Machine bad guest physical address |

---

### Machine Memory Protection

| CSR | Address | R/W | Description |
|-----|---------|-----|-------------|
| **pmpcfg0** | 0x3A0 | RW | PMP configuration register 0 |
| **pmpcfg1** | 0x3A1 | RW | PMP configuration register 1 (RV32 only) |
| **pmpcfg2** | 0x3A2 | RW | PMP configuration register 2 |
| **pmpcfg3** | 0x3A3 | RW | PMP configuration register 3 (RV32 only) |
| **pmpcfg4-15** | 0x3A4-0x3AF | RW | PMP configuration registers 4-15 |
| **pmpaddr0-15** | 0x3B0-0x3BF | RW | PMP address registers 0-15 |
| **pmpaddr16-63** | 0x3C0-0x3EF | RW | PMP address registers 16-63 |

**Note**: RV32 uses pmpcfg0, pmpcfg2, pmpcfg4, etc. (even-numbered only). RV64 uses pmpcfg0, pmpcfg2, pmpcfg4, etc., with each holding 8 configuration bytes.

---

### Machine Counters and Timers

| CSR | Address | R/W | Description |
|-----|---------|-----|-------------|
| **mcycle** | 0xB00 | RW | Machine cycle counter (lower 32/64 bits) |
| **minstret** | 0xB02 | RW | Machine instructions retired counter |
| **mhpmcounter3-31** | 0xB03-0xB1F | RW | Machine performance monitoring counters |
| **mcycleh** | 0xB80 | RW | Upper 32 bits of mcycle (RV32 only) |
| **minstreth** | 0xB82 | RW | Upper 32 bits of minstret (RV32 only) |
| **mhpmcounter3h-31h** | 0xB83-0xB9F | RW | Upper 32 bits of mhpmcounter (RV32 only) |

---

### Machine Counter Setup

| CSR | Address | R/W | Description |
|-----|---------|-----|-------------|
| **mcountinhibit** | 0x320 | RW | Machine counter-inhibit register |
| **mhpmevent3-31** | 0x323-0x33F | RW | Machine performance monitoring event selectors |

**Usage**: mcountinhibit controls which counters are active. Setting bit N stops counter N from incrementing, saving power.

---

## A.3 Supervisor-Level CSRs (S-mode)

### Supervisor Trap Setup

| CSR | Address | R/W | Description |
|-----|---------|-----|-------------|
| **sstatus** | 0x100 | RW | Supervisor status register (subset of mstatus) |
| **sie** | 0x104 | RW | Supervisor interrupt enable |
| **stvec** | 0x105 | RW | Supervisor trap-handler base address |
| **scounteren** | 0x106 | RW | Counter enable for U-mode |

---

### Supervisor Trap Handling

| CSR | Address | R/W | Description |
|-----|---------|-----|-------------|
| **sscratch** | 0x140 | RW | Scratch register for S-mode trap handlers |
| **sepc** | 0x141 | RW | Supervisor exception program counter |
| **scause** | 0x142 | RW | Supervisor trap cause |
| **stval** | 0x143 | RW | Supervisor bad address or instruction |
| **sip** | 0x144 | RW | Supervisor interrupt pending |

---

### Supervisor Address Translation and Protection

| CSR | Address | R/W | Description |
|-----|---------|-----|-------------|
| **satp** | 0x180 | RW | Supervisor address translation and protection |

**satp Format (RV64)**:

```
Bits [63:60]: Mode (0=Bare, 8=Sv39, 9=Sv48, 10=Sv57)
Bits [59:44]: ASID (Address Space Identifier)
Bits [43:0]:  PPN (Physical Page Number of root page table)
```

---

## A.4 User-Level CSRs (U-mode)

### Floating-Point Control and Status

| CSR | Address | R/W | Description |
|-----|---------|-----|-------------|
| **fflags** | 0x001 | RW | Floating-point accrued exceptions |
| **frm** | 0x002 | RW | Floating-point rounding mode |
| **fcsr** | 0x003 | RW | Floating-point control and status (fflags + frm) |

**fflags Bits**:

- Bit 0: NV (Invalid Operation)
- Bit 1: DZ (Divide by Zero)
- Bit 2: OF (Overflow)
- Bit 3: UF (Underflow)
- Bit 4: NX (Inexact)

**frm Values**:

- 0: RNE (Round to Nearest, ties to Even)
- 1: RTZ (Round towards Zero)
- 2: RDN (Round Down, towards -∞)
- 3: RUP (Round Up, towards +∞)
- 4: RMM (Round to Nearest, ties to Max Magnitude)

---

### User Counters and Timers

| CSR | Address | R/W | Description |
|-----|---------|-----|-------------|
| **cycle** | 0xC00 | RO | Cycle counter (lower 32/64 bits) |
| **time** | 0xC01 | RO | Timer (lower 32/64 bits) |
| **instret** | 0xC02 | RO | Instructions retired counter |
| **hpmcounter3-31** | 0xC03-0xC1F | RO | Performance monitoring counters |
| **cycleh** | 0xC80 | RO | Upper 32 bits of cycle (RV32 only) |
| **timeh** | 0xC81 | RO | Upper 32 bits of time (RV32 only) |
| **instreth** | 0xC82 | RO | Upper 32 bits of instret (RV32 only) |
| **hpmcounter3h-31h** | 0xC83-0xC9F | RO | Upper 32 bits of hpmcounter (RV32 only) |

**Note**: These are read-only shadows of the machine-level counters. Access can be disabled by mcounteren (for S-mode) or scounteren (for U-mode).

---

## A.5 Debug CSRs

Debug CSRs are accessible only in Debug Mode (entered via debugger or trigger).

| CSR | Address | R/W | Description |
|-----|---------|-----|-------------|
| **dcsr** | 0x7B0 | RW | Debug control and status register |
| **dpc** | 0x7B1 | RW | Debug program counter |
| **dscratch0** | 0x7B2 | RW | Debug scratch register 0 |
| **dscratch1** | 0x7B3 | RW | Debug scratch register 1 |

**dcsr Bit Fields**:

- Bits [31:28]: xdebugver (Debug specification version)
- Bits [8:6]: cause (Reason for entering debug mode)
  - 1: ebreak instruction
  - 2: Trigger module
  - 3: Debugger halt request
  - 4: Single step
  - 5: Reset halt
- Bit [2]: step (Single-step mode enable)
- Bits [1:0]: prv (Privilege level before entering debug mode)

---

## A.6 Trigger/Debug Module CSRs

| CSR | Address | R/W | Description |
|-----|---------|-----|-------------|
| **tselect** | 0x7A0 | RW | Trigger select register |
| **tdata1** | 0x7A1 | RW | Trigger data register 1 (type and config) |
| **tdata2** | 0x7A2 | RW | Trigger data register 2 (match value) |
| **tdata3** | 0x7A3 | RW | Trigger data register 3 (additional data) |
| **tinfo** | 0x7A4 | RO | Trigger info (supported types) |
| **tcontrol** | 0x7A5 | RW | Trigger control |
| **mcontext** | 0x7A8 | RW | Machine context register |
| **scontext** | 0x7AA | RW | Supervisor context register |

**Usage**: Triggers enable hardware breakpoints and watchpoints. tselect chooses which trigger to configure, tdata1-3 configure the selected trigger.

---

## A.7 Key CSR Bit Fields

### mstatus (Machine Status Register)

**RV64 Format**:

```
Bit  63: SD (State Dirty - summary of FS/XS)
Bits 36-37: SXL (S-mode XLEN)
Bits 34-35: UXL (U-mode XLEN)
Bit  22: TSR (Trap SRET)
Bit  21: TW (Timeout Wait - trap WFI)
Bit  20: TVM (Trap Virtual Memory - trap SATP writes)
Bit  19: MXR (Make eXecutable Readable)
Bit  18: SUM (permit Supervisor User Memory access)
Bit  17: MPRV (Modify PRiVilege)
Bits 15-16: XS (user eXtension State)
Bits 13-14: FS (Floating-point State)
Bits 11-12: MPP (Machine Previous Privilege)
Bit  8: SPP (Supervisor Previous Privilege)
Bit  7: MPIE (Machine Previous Interrupt Enable)
Bit  5: SPIE (Supervisor Previous Interrupt Enable)
Bit  3: MIE (Machine Interrupt Enable)
Bit  1: SIE (Supervisor Interrupt Enable)
```

**FS/XS Values**:

- 0: Off (all off)
- 1: Initial (none dirty, some on)
- 2: Clean (none dirty, some on)
- 3: Dirty (some dirty)

**MPP/SPP Values**:

- 0: User mode
- 1: Supervisor mode
- 3: Machine mode

---

### mtvec (Machine Trap Vector)

**Format**:

```
Bits [XLEN-1:2]: BASE (trap handler base address, 4-byte aligned)
Bits [1:0]: MODE
  0 = Direct (all traps to BASE)
  1 = Vectored (interrupts to BASE + 4*cause, exceptions to BASE)
```

**Example**:

```assembly
la t0, trap_handler
csrw mtvec, t0          # Direct mode (MODE=0)

la t0, trap_handler
ori t0, t0, 1           # Set MODE=1
csrw mtvec, t0          # Vectored mode
```

---

### mcause (Machine Cause Register)

**Format**:

- Bit [XLEN-1]: Interrupt (1=interrupt, 0=exception)
- Bits [XLEN-2:0]: Exception Code

**Exception Codes** (Interrupt=0):

| Code | Exception |
|------|-----------|
| 0 | Instruction address misaligned |
| 1 | Instruction access fault |
| 2 | Illegal instruction |
| 3 | Breakpoint |
| 4 | Load address misaligned |
| 5 | Load access fault |
| 6 | Store/AMO address misaligned |
| 7 | Store/AMO access fault |
| 8 | Environment call from U-mode |
| 9 | Environment call from S-mode |
| 11 | Environment call from M-mode |
| 12 | Instruction page fault |
| 13 | Load page fault |
| 15 | Store/AMO page fault |

**Interrupt Codes** (Interrupt=1):

| Code | Interrupt |
|------|-----------|
| 0 | User software interrupt |
| 1 | Supervisor software interrupt |
| 3 | Machine software interrupt |
| 4 | User timer interrupt |
| 5 | Supervisor timer interrupt |
| 7 | Machine timer interrupt |
| 8 | User external interrupt |
| 9 | Supervisor external interrupt |
| 11 | Machine external interrupt |

---

### satp (Supervisor Address Translation and Protection)

**RV64 Format**:

```
Bits [63:60]: MODE
  0 = Bare (no translation)
  8 = Sv39 (39-bit virtual address)
  9 = Sv48 (48-bit virtual address)
  10 = Sv57 (57-bit virtual address)
Bits [59:44]: ASID (Address Space Identifier, 16 bits)
Bits [43:0]: PPN (Physical Page Number of root page table, 44 bits)
```

**RV32 Format**:

```
Bit [31]: MODE (0=Bare, 1=Sv32)
Bits [30:22]: ASID (9 bits)
Bits [21:0]: PPN (22 bits)
```

**Example**:

```assembly
# Switch to Sv39 mode with ASID=1, root page table at 0x80200000
li t0, 0x8000000000080200  # MODE=8, ASID=0, PPN=0x80200
csrw satp, t0
sfence.vma                 # Flush TLB
```

---

## A.8 CSR Instructions Quick Reference

| Instruction | Format | Operation |
|-------------|--------|-----------|
| **CSRRW** | csrrw rd, csr, rs1 | t = CSR; CSR = rs1; rd = t |
| **CSRRS** | csrrs rd, csr, rs1 | t = CSR; CSR = t \| rs1; rd = t |
| **CSRRC** | csrrc rd, csr, rs1 | t = CSR; CSR = t & ~rs1; rd = t |
| **CSRRWI** | csrrwi rd, csr, imm | t = CSR; CSR = imm; rd = t |
| **CSRRSI** | csrrsi rd, csr, imm | t = CSR; CSR = t \| imm; rd = t |
| **CSRRCI** | csrrci rd, csr, imm | t = CSR; CSR = t & ~imm; rd = t |

**Pseudo-instructions**:

```assembly
csrr rd, csr        # Read CSR (csrrs rd, csr, x0)
csrw csr, rs1       # Write CSR (csrrw x0, csr, rs1)
csrs csr, rs1       # Set bits (csrrs x0, csr, rs1)
csrc csr, rs1       # Clear bits (csrrc x0, csr, rs1)
csrwi csr, imm      # Write immediate (csrrwi x0, csr, imm)
csrsi csr, imm      # Set bits immediate (csrrsi x0, csr, imm)
csrci csr, imm      # Clear bits immediate (csrrci x0, csr, imm)
```

---

## A.9 Common CSR Usage Patterns

### Enable Machine-Mode Interrupts

```assembly
# Enable machine timer and external interrupts
li t0, 0x88             # MTIE (bit 7) + MEIE (bit 11)
csrs mie, t0            # Set bits in mie

# Enable global interrupts
li t0, 0x8              # MIE (bit 3)
csrs mstatus, t0        # Set MIE in mstatus
```

### Trap Handler Entry

```assembly
trap_handler:
    # Save context
    csrrw sp, mscratch, sp  # Swap sp with mscratch

    # Save registers on stack
    addi sp, sp, -32*8
    sd x1, 0(sp)
    sd x2, 8(sp)
    # ... save all registers ...

    # Read trap cause
    csrr t0, mcause
    csrr t1, mepc
    csrr t2, mtval

    # Handle trap...
```

### Context Switch (Change satp)

```assembly
# Switch to new process page table
# a0 = new satp value
csrw satp, a0
sfence.vma              # Flush TLB
```

### Disable Interrupts for Critical Section

```assembly
# Save and disable interrupts
csrrci t0, mstatus, 0x8  # Clear MIE, save old mstatus

# Critical section...

# Restore interrupts
csrw mstatus, t0         # Restore original mstatus
```

---

## A.10 CSR Access Permissions

**Privilege Level Check**:

- CSR address bits [11:10] encode minimum privilege level
- Accessing CSR from lower privilege → illegal instruction exception

**Read-Only Check**:

- CSR address bits [11:10] = 11 → read-only
- Writing to read-only CSR → illegal instruction exception

**Implementation-Defined Behavior**:

- Unimplemented CSRs may:
  - Read as zero, writes ignored (WARL - Write Any, Read Legal)
  - Cause illegal instruction exception
  - Implementation must document behavior

---

## A.11 References

- **RISC-V Privileged Specification**: Complete CSR definitions and bit fields
- **RISC-V Debug Specification**: Debug CSRs (dcsr, dpc, tselect, tdata)
- **RISC-V ISA Manual**: CSR instructions and access rules

---
