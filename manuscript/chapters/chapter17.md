# Chapter 17. RISC-V vs ARM vs MIPS — A Systematic Comparison

**Part X — RISC-V vs Other Architectures**

---

Architecture choice shapes everything. The instruction set determines how software expresses computation, how hardware implements execution, and how ecosystems develop around the platform. RISC-V enters a landscape dominated by ARM in mobile and embedded systems, and historically influenced by MIPS in education and networking. Understanding how these architectures compare reveals RISC-V's design decisions, trade-offs, and competitive position.

This chapter provides a systematic comparison of RISC-V, ARM, and MIPS across eleven dimensions: ISA design philosophy, instruction set complexity, register architecture, exception and interrupt models, memory models, virtual memory, interrupt architecture, calling conventions, pipeline and microarchitecture, ecosystem and licensing, and future directions. Each section examines how the architectures approach the same problem, highlighting similarities, differences, and the implications for software and hardware.

RISC-V represents a modern, modular, open approach. ARM represents a comprehensive, evolving, commercial approach. MIPS represents classic RISC simplicity with historical commercial roots. Together, they illustrate the spectrum of architectural design choices.

---

## 17.1 ISA Design Philosophy

**RISC-V: Modular and Extensible**

RISC-V's philosophy emphasizes modularity and extensibility:

- **Base ISA**: Minimal, frozen foundation (RV32I, RV64I, RV128I)
- **Standard extensions**: Optional, composable modules (M, A, F, D, C, V, etc.)
- **Custom extensions**: Vendor-specific additions without ISA fragmentation
- **Clean slate**: No legacy baggage, designed from first principles

This modular approach allows:

- Tiny microcontrollers (RV32I only)
- Application processors (RV64IMAFDCV)
- Custom accelerators (base + custom extensions)

**ARM: Comprehensive and Evolving**

ARM's philosophy emphasizes comprehensiveness and evolution:

- **Comprehensive ISA**: Rich instruction set covering many use cases
- **Profiles**: Different ISA subsets (A-profile, R-profile, M-profile)
- **Backward compatibility**: New versions extend, rarely remove features
- **Market-driven**: Features added based on market needs

ARM profiles:

- **A-profile**: Application processors (ARMv8-A, ARMv9-A)
- **R-profile**: Real-time processors (ARMv8-R)
- **M-profile**: Microcontrollers (ARMv8-M, ARMv9-M)

**MIPS: Classic RISC Simplicity**

MIPS's philosophy emphasizes classic RISC principles:

- **Simple, regular ISA**: Load-store architecture, fixed-length instructions
- **Delayed branches**: Expose pipeline to software (MIPS I-IV)
- **Coprocessor model**: Floating-point and other functions as coprocessors
- **Minimal complexity**: Keep hardware simple, let software handle complexity

MIPS influenced RISC-V's design but RISC-V modernized many aspects (no delayed branches, cleaner privilege model).

**Design Trade-offs**

| Aspect | RISC-V | ARM | MIPS |
|--------|--------|-----|------|
| **Modularity** | High (base + extensions) | Medium (profiles) | Low (monolithic) |
| **Extensibility** | High (custom extensions) | Low (vendor-specific) | Low (proprietary) |
| **Backward Compatibility** | High (frozen base) | High (evolutionary) | Medium (versions) |
| **Complexity** | Low to medium | Medium to high | Low |
| **Flexibility** | High | Medium | Low |

---

## 17.2 Instruction Set Complexity

**Instruction Count Comparison**

Approximate instruction counts:

| Architecture | Base Instructions | With Extensions | Total (typical) |
|--------------|-------------------|-----------------|-----------------|
| **RISC-V RV32I** | 47 | +M(8), +A(11), +F(26), +D(26), +C(~40) | ~150-200 |
| **RISC-V RV64I** | 59 | +M(8), +A(11), +F(26), +D(26), +C(~40) | ~170-220 |
| **ARM ARMv8-A** | ~500 base | +NEON, +SVE, +crypto | ~1000+ |
| **MIPS32** | ~100 base | +FPU, +DSP | ~200-300 |

RISC-V has the smallest base ISA. ARM has the largest instruction set. MIPS falls in between.

**Encoding Formats**

*RISC-V*: 6 base formats (R, I, S, B, U, J) + compressed (CR, CI, CSS, CIW, CL, CS, CA, CB, CJ)

```
R-type: [funct7|rs2|rs1|funct3|rd|opcode]
I-type: [imm[11:0]|rs1|funct3|rd|opcode]
S-type: [imm[11:5]|rs2|rs1|funct3|imm[4:0]|opcode]
```

*ARM*: Multiple formats (data processing, load/store, branch, etc.)

```
Data processing: [cond|00|I|opcode|S|Rn|Rd|operand2]
Load/store:      [cond|01|I|P|U|B|W|L|Rn|Rd|offset]
```

*MIPS*: 3 formats (R, I, J)

```
R-type: [opcode|rs|rt|rd|shamt|funct]
I-type: [opcode|rs|rt|immediate]
J-type: [opcode|address]
```

RISC-V and MIPS have simpler, more regular encodings than ARM.

**Addressing Modes**

*RISC-V*:

- Register: `add rd, rs1, rs2`
- Immediate: `addi rd, rs1, imm`
- Base+offset: `lw rd, offset(rs1)`

*ARM*:

- Register: `ADD Rd, Rn, Rm`
- Immediate: `ADD Rd, Rn, #imm`
- Base+offset: `LDR Rd, [Rn, #offset]`
- Base+register: `LDR Rd, [Rn, Rm]`
- Pre/post-indexed: `LDR Rd, [Rn, #offset]!` or `LDR Rd, [Rn], #offset`

*MIPS*:

- Register: `add $rd, $rs, $rt`
- Immediate: `addi $rt, $rs, imm`
- Base+offset: `lw $rt, offset($rs)`

ARM has the most addressing modes. RISC-V and MIPS are simpler.

**ISA Complexity Metrics**

| Metric | RISC-V | ARM | MIPS |
|--------|--------|-----|------|
| **Instruction formats** | 6 base + 9 compressed | 10+ | 3 |
| **Addressing modes** | 3 | 10+ | 3 |
| **Conditional execution** | Branch only | Most instructions (ARMv7), limited (ARMv8) | Branch only |
| **Instruction length** | 32-bit (16-bit compressed) | 32-bit (16-bit Thumb) | 32-bit |
| **Regularity** | High | Medium | High |

RISC-V and MIPS prioritize regularity. ARM prioritizes expressiveness.

---

## 17.3 Register Architecture

**RISC-V: x0-x31 (x0 = zero)**

RISC-V provides 32 general-purpose registers:

- `x0`: Hardwired zero (reads as 0, writes ignored)
- `x1-x31`: General-purpose registers
- `f0-f31`: Floating-point registers (F/D extensions)

Special conventions:

- `x1` (ra): Return address
- `x2` (sp): Stack pointer
- `x8` (s0/fp): Frame pointer

**ARM: X0-X30 + XZR + SP**

ARM ARMv8-A provides 31 general-purpose registers + special registers:

- `X0-X30`: General-purpose registers (64-bit)
- `W0-W30`: 32-bit views of X0-X30
- `XZR` (X31): Zero register (reads as 0, writes ignored)
- `SP`: Stack pointer (separate from general registers)
- `PC`: Program counter (not directly accessible in ARMv8)

ARM has 31 general registers vs RISC-V's 32 (including zero).

**MIPS: $0-$31 ($0 = zero)**

MIPS provides 32 general-purpose registers:

- `$0` ($zero): Hardwired zero
- `$1-$31`: General-purpose registers

Special conventions:

- `$31` ($ra): Return address
- `$29` ($sp): Stack pointer
- `$30` ($fp): Frame pointer

MIPS and RISC-V have identical register counts and zero register concept.

**Special-Purpose Registers**

*RISC-V*: CSRs (Control and Status Registers)

- Accessed via `csrr`, `csrw`, `csrrw`, etc.
- Examples: `mstatus`, `mtvec`, `mepc`, `mcause`

*ARM*: System registers

- Accessed via `MRS`, `MSR` instructions
- Examples: `SCTLR_EL1`, `VBAR_EL1`, `ELR_EL1`, `ESR_EL1`

*MIPS*: Coprocessor 0 registers

- Accessed via `mfc0`, `mtc0` instructions
- Examples: `Status`, `Cause`, `EPC`, `BadVAddr`

All three use separate namespaces for system registers.

---

## 17.4 Exception and Interrupt Models

**RISC-V: Trap Model (M/S/U Modes)**

RISC-V uses a unified trap model:

- **Privilege modes**: M (Machine), S (Supervisor), U (User)
- **Trap types**: Exceptions (synchronous), Interrupts (asynchronous)
- **Trap vector**: `mtvec` (M-mode), `stvec` (S-mode)
- **Trap cause**: `mcause` (M-mode), `scause` (S-mode)
- **Trap PC**: `mepc` (M-mode), `sepc` (S-mode)

Trap handling:

```assembly
# M-mode trap handler
trap_handler:
    csrr t0, mcause      # Read cause
    csrr t1, mepc        # Read PC
    # Handle trap
    mret                 # Return from trap
```

**ARM: Exception Levels (EL0-EL3)**

ARM uses exception levels:

- **Exception levels**: EL0 (User), EL1 (OS), EL2 (Hypervisor), EL3 (Secure Monitor)
- **Exception types**: Synchronous, IRQ, FIQ, SError
- **Exception vector**: `VBAR_EL1`, `VBAR_EL2`, `VBAR_EL3`
- **Exception syndrome**: `ESR_EL1`, `ESR_EL2`, `ESR_EL3`
- **Exception link**: `ELR_EL1`, `ELR_EL2`, `ELR_EL3`

Exception handling:

```assembly
# EL1 exception handler
exception_handler:
    mrs x0, ESR_EL1      # Read syndrome
    mrs x1, ELR_EL1      # Read PC
    # Handle exception
    eret                 # Return from exception
```

**MIPS: Exception Handling**

MIPS uses a simpler exception model:

- **Modes**: User, Kernel
- **Exception vector**: Fixed at 0x80000180 (general) or 0x80000000 (reset/NMI)
- **Cause register**: Encodes exception type
- **EPC**: Exception PC

Exception handling:

```assembly
# MIPS exception handler
exception_handler:
    mfc0 k0, Cause       # Read cause
    mfc0 k1, EPC         # Read PC
    # Handle exception
    eret                 # Return from exception
```

**Comparison Table**

| Feature | RISC-V | ARM | MIPS |
|---------|--------|-----|------|
| **Privilege Levels** | 3 (M/S/U) + optional H | 4 (EL0-EL3) | 2 (User/Kernel) |
| **Trap/Exception Types** | Unified (trap) | 4 types (Sync, IRQ, FIQ, SError) | Unified (exception) |
| **Vector Table** | mtvec, stvec | VBAR_ELn | Fixed address |
| **Cause Register** | mcause, scause | ESR_ELn | Cause |
| **Return Instruction** | mret, sret | eret | eret |
| **Nested Interrupts** | Software-managed | Hardware-managed | Software-managed |

ARM has the most sophisticated exception model. RISC-V is modular. MIPS is simplest.

---

## 17.5 Memory Models

**RISC-V: RVWMO (Weak Memory Ordering)**

RISC-V uses a weak memory model (RVWMO):

- **Ordering**: Relaxed by default
- **Fences**: `fence` instruction for ordering
- **Atomics**: LR/SC and AMO instructions
- **TSO extension**: Optional total store ordering (Ztso)

Memory ordering:

```assembly
# RISC-V: Ensure store visible before load
sw x10, 0(x5)
fence w, r           # Write-to-read fence
lw x11, 0(x6)
```

**ARM: Weak Memory Model**

ARM uses a weak memory model:

- **Ordering**: Relaxed by default
- **Barriers**: DMB (data memory barrier), DSB (data synchronization barrier), ISB (instruction synchronization barrier)
- **Atomics**: LDXR/STXR (exclusive) and atomic instructions (ARMv8.1+)

Memory ordering:

```assembly
# ARM: Ensure store visible before load
STR X0, [X1]
DMB SY               # Data memory barrier (system)
LDR X2, [X3]
```

**MIPS: Sequential Consistency Variants**

MIPS traditionally used sequential consistency, but modern MIPS supports weak ordering:

- **Ordering**: Sequential consistency (MIPS I-III), weak ordering (MIPS IV+)
- **Barriers**: `sync` instruction
- **Atomics**: LL/SC (load-linked/store-conditional)

Memory ordering:

```assembly
# MIPS: Ensure store visible before load
sw $t0, 0($a0)
sync                 # Synchronization barrier
lw $t1, 0($a1)
```

**Memory Barrier Instructions**

| Architecture | Barrier Instruction | Purpose |
|--------------|---------------------|---------|
| **RISC-V** | `fence r, w` | Order reads before writes |
| **RISC-V** | `fence w, r` | Order writes before reads |
| **RISC-V** | `fence rw, rw` | Full fence |
| **ARM** | `DMB` | Data memory barrier |
| **ARM** | `DSB` | Data synchronization barrier |
| **ARM** | `ISB` | Instruction synchronization barrier |
| **MIPS** | `sync` | Synchronization barrier |

RISC-V's fence is more fine-grained (specify predecessor/successor). ARM has multiple barrier types. MIPS has a single sync instruction.

---

## 17.6 Virtual Memory

**RISC-V: Sv39/Sv48 Page Tables**

RISC-V virtual memory:

- **Sv39**: 39-bit virtual address, 3-level page table
- **Sv48**: 48-bit virtual address, 4-level page table
- **Sv57**: 57-bit virtual address, 5-level page table (future)
- **Page sizes**: 4 KB, 2 MB (megapage), 1 GB (gigapage)
- **TLB**: Implementation-specific

Page table entry (Sv39):

```
[63:54] Reserved
[53:28] PPN[2] (26 bits)
[27:19] PPN[1] (9 bits)
[18:10] PPN[0] (9 bits)
[9:0]   Flags (V, R, W, X, U, G, A, D)
```

**ARM: 48-bit VA with TTBR0/1**

ARM virtual memory (ARMv8-A):

- **VA size**: 48-bit (configurable 36-48 bits)
- **Page table levels**: 4 levels (configurable)
- **Page sizes**: 4 KB, 2 MB, 1 GB
- **TTBR0/TTBR1**: Separate page tables for user/kernel

Page table entry (4 KB granule):

```
[63:48] Ignored/SW use
[47:12] Output address
[11:2]  Attributes (AF, SH, AP, NS, etc.)
[1]     Table/Block
[0]     Valid
```

**MIPS: TLB-Based MMU**

MIPS virtual memory:

- **TLB**: Software-managed TLB (no hardware page table walk)
- **VA size**: 32-bit (MIPS32), 64-bit (MIPS64)
- **Page sizes**: 4 KB (typical), configurable
- **TLB entries**: 16-64 entries (implementation-specific)

TLB entry:

```
EntryHi:  [VPN | ASID]
EntryLo0: [PFN | C | D | V | G]
EntryLo1: [PFN | C | D | V | G]
PageMask: Page size
```

**Page Table Walk Comparison**

| Feature | RISC-V | ARM | MIPS |
|---------|--------|-----|------|
| **Hardware Walk** | Yes | Yes | No (software TLB refill) |
| **Page Table Levels** | 3-5 (Sv39-Sv57) | 4 (configurable) | N/A (TLB only) |
| **Page Sizes** | 4 KB, 2 MB, 1 GB | 4 KB, 2 MB, 1 GB | 4 KB (configurable) |
| **TLB Refill** | Hardware | Hardware | Software (exception) |
| **ASID** | Yes (satp.ASID) | Yes (TTBR.ASID) | Yes (EntryHi.ASID) |

RISC-V and ARM use hardware page table walks. MIPS uses software TLB refill, giving more flexibility but requiring software overhead.

---

## 17.7 Interrupt Architecture

**RISC-V: PLIC/CLIC/AIA**

RISC-V interrupt architecture has evolved:

- **CLINT**: Core-Local Interruptor (timer, software interrupts)
- **PLIC**: Platform-Level Interrupt Controller (external interrupts)
- **CLIC**: Core-Local Interrupt Controller (vectored, nested interrupts for embedded)
- **AIA**: Advanced Interrupt Architecture (MSI, IMSIC for servers)

PLIC example:

```c
// PLIC interrupt handler
void plic_handler(void) {
    uint32_t source = plic_claim();  // Claim interrupt

    if (source == UART_IRQ) {
        uart_handler();
    }

    plic_complete(source);  // Complete interrupt
}
```

**ARM: GIC (GICv3/GICv4)**

ARM uses the Generic Interrupt Controller (GIC):

- **GICv2**: Legacy, supports up to 8 cores
- **GICv3**: Modern, supports many cores, message-based
- **GICv4**: Adds virtualization support
- **Interrupt types**: SGI (software), PPI (private peripheral), SPI (shared peripheral)

GIC example:

```c
// GIC interrupt handler
void gic_handler(void) {
    uint32_t intid = gic_acknowledge();  // Acknowledge interrupt

    if (intid == UART_INTID) {
        uart_handler();
    }

    gic_end_of_interrupt(intid);  // End of interrupt
}
```

**MIPS: Simple IRQ Model**

MIPS uses a simple interrupt model:

- **Interrupt lines**: 8 hardware interrupt lines (IP0-IP7)
- **Interrupt mask**: Controlled by Status register
- **Interrupt pending**: Indicated by Cause register
- **External controller**: Optional external interrupt controller

MIPS example:

```c
// MIPS interrupt handler
void mips_interrupt_handler(void) {
    uint32_t cause = read_c0_cause();
    uint32_t pending = (cause >> 8) & 0xFF;  // IP bits

    if (pending & (1 << 2)) {  // IP2
        uart_handler();
    }
}
```

**Interrupt Routing and Priority**

| Feature | RISC-V (PLIC) | ARM (GIC) | MIPS |
|---------|---------------|-----------|------|
| **Interrupt Sources** | 1-1023 | 32-1020 | 8 lines |
| **Priority Levels** | 0-255 | 0-255 | None (software) |
| **Routing** | Per-hart enable | Affinity routing | Fixed |
| **Vectoring** | Optional (CLIC) | Yes | No |
| **Nesting** | Software-managed | Hardware-managed | Software-managed |

ARM GIC is the most sophisticated. RISC-V PLIC is flexible. MIPS is simplest.

---

## 17.8 Calling Conventions

**RISC-V: RV64 SysV ABI**

RISC-V calling convention (RV64):

- **Arguments**: a0-a7 (x10-x17)
- **Return values**: a0-a1 (x10-x11)
- **Saved registers**: s0-s11 (x8-x9, x18-x27)
- **Temporary registers**: t0-t6 (x5-x7, x28-x31)
- **Stack pointer**: sp (x2)
- **Return address**: ra (x1)

Function call:

```assembly
# Caller
addi sp, sp, -16
sd ra, 8(sp)
call function        # ra ← PC+4, PC ← function
ld ra, 8(sp)
addi sp, sp, 16

# Callee
function:
    addi sp, sp, -16
    sd s0, 8(sp)
    # ... function body ...
    ld s0, 8(sp)
    addi sp, sp, 16
    ret              # PC ← ra
```

**ARM: AAPCS64**

ARM calling convention (AAPCS64):

- **Arguments**: X0-X7
- **Return values**: X0-X1
- **Saved registers**: X19-X28
- **Temporary registers**: X9-X15
- **Stack pointer**: SP
- **Return address**: X30 (LR)
- **Frame pointer**: X29 (FP)

Function call:

```assembly
# Caller
STP X29, X30, [SP, #-16]!
BL function          # LR ← PC+4, PC ← function
LDP X29, X30, [SP], #16

# Callee
function:
    STP X29, X30, [SP, #-16]!
    MOV X29, SP
    # ... function body ...
    LDP X29, X30, [SP], #16
    RET              # PC ← LR
```

**MIPS: O32/N32/N64 ABIs**

MIPS has multiple ABIs:

- **O32**: 32-bit, 4 argument registers
- **N32**: 32-bit pointers, 64-bit registers
- **N64**: 64-bit

MIPS O32 calling convention:

- **Arguments**: $a0-$a3 ($4-$7)
- **Return values**: $v0-$v1 ($2-$3)
- **Saved registers**: $s0-$s7 ($16-$23)
- **Temporary registers**: $t0-$t9 ($8-$15, $24-$25)
- **Stack pointer**: $sp ($29)
- **Return address**: $ra ($31)

Function call:

```assembly
# Caller
addiu $sp, $sp, -8
sw $ra, 4($sp)
jal function         # $ra ← PC+8, PC ← function
lw $ra, 4($sp)
addiu $sp, $sp, 8

# Callee
function:
    addiu $sp, $sp, -8
    sw $s0, 4($sp)
    # ... function body ...
    lw $s0, 4($sp)
    addiu $sp, $sp, 8
    jr $ra           # PC ← $ra
```

**ABI Comparison**

| Feature | RISC-V | ARM | MIPS (O32) |
|---------|--------|-----|------------|
| **Argument Registers** | 8 (a0-a7) | 8 (X0-X7) | 4 ($a0-$a3) |
| **Return Registers** | 2 (a0-a1) | 2 (X0-X1) | 2 ($v0-$v1) |
| **Saved Registers** | 12 (s0-s11) | 10 (X19-X28) | 8 ($s0-$s7) |
| **Temporary Registers** | 7 (t0-t6) | 7 (X9-X15) | 10 ($t0-$t9) |
| **Stack Growth** | Downward | Downward | Downward |
| **Alignment** | 16 bytes | 16 bytes | 8 bytes (O32) |

RISC-V and ARM have more argument registers than MIPS O32, reducing stack usage.

---

## 17.9 Pipeline and Microarchitecture

**In-Order vs Out-of-Order**

All three architectures support both in-order and out-of-order implementations:

*RISC-V*:

- In-order: SiFive E-series, Rocket
- Out-of-order: SiFive P-series, BOOM (Berkeley Out-of-Order Machine)

*ARM*:

- In-order: Cortex-A5, Cortex-A7, Cortex-A53
- Out-of-order: Cortex-A72, Cortex-A76, Neoverse N1/V1

*MIPS*:

- In-order: MIPS 24K, 34K
- Out-of-order: MIPS 74K, R10000

**Branch Prediction Strategies**

Modern implementations use sophisticated branch prediction:

| Implementation | Branch Predictor | Accuracy |
|----------------|------------------|----------|
| **SiFive U74** | 2-level adaptive | ~90% |
| **SiFive P550** | TAGE predictor | ~95% |
| **ARM Cortex-A76** | Multi-level predictor | ~95%+ |
| **MIPS 74K** | 2-level adaptive | ~90% |

Out-of-order cores require better branch prediction to maintain performance.

**Cache Hierarchies**

Typical cache configurations:

*RISC-V (SiFive P550)*:

- L1 I-cache: 32 KB, 4-way
- L1 D-cache: 32 KB, 8-way
- L2 cache: 512 KB - 2 MB, 8-way

*ARM (Cortex-A76)*:

- L1 I-cache: 64 KB, 4-way
- L1 D-cache: 64 KB, 4-way
- L2 cache: 256 KB - 512 KB, 8-way

*MIPS (74K)*:

- L1 I-cache: 32 KB, 4-way
- L1 D-cache: 32 KB, 4-way
- L2 cache: Optional, implementation-specific

**Implementation Examples**

| Implementation | Type | Pipeline Stages | Issue Width | IPC (peak) |
|----------------|------|-----------------|-------------|------------|
| **SiFive E76** | In-order | 8 | 1 | 1.0 |
| **SiFive P550** | Out-of-order | 13 | 3 | 3.0 |
| **ARM Cortex-A53** | In-order | 8 | 2 | 2.0 |
| **ARM Cortex-A76** | Out-of-order | 13 | 4 | 4.0 |
| **MIPS 24K** | In-order | 8 | 1 | 1.0 |
| **MIPS 74K** | Out-of-order | 15 | 2 | 2.0 |

ARM has the most aggressive out-of-order implementations. RISC-V is catching up. MIPS development has slowed.

---

## 17.10 Ecosystem and Licensing

**RISC-V: Open and Free ISA**

RISC-V ecosystem:

- **ISA**: Open, royalty-free
- **Licensing**: No licensing fees
- **Governance**: RISC-V International (non-profit)
- **Implementations**: Open-source (Rocket, BOOM) and commercial (SiFive, Andes, etc.)
- **Software**: GCC, LLVM, Linux, FreeBSD, Zephyr, FreeRTOS

Advantages:

- No licensing costs
- Customizable (add custom extensions)
- Growing ecosystem
- Academic and research-friendly

Challenges:

- Younger ecosystem (less mature than ARM)
- Fewer commercial implementations
- Software ecosystem still developing

**ARM: Commercial Licensing**

ARM ecosystem:

- **ISA**: Proprietary
- **Licensing**: Architecture license (design own core) or implementation license (use ARM core)
- **Governance**: ARM Holdings (commercial company)
- **Implementations**: ARM Cortex series, vendor designs (Apple, Qualcomm, Samsung)
- **Software**: Mature ecosystem (GCC, LLVM, Linux, Android, iOS)

Advantages:

- Mature, proven ecosystem
- Extensive software support
- Wide industry adoption
- Strong performance

Challenges:

- Licensing fees
- Less customizable
- Vendor lock-in

**MIPS: Historical Commercial, Now Open**

MIPS ecosystem:

- **ISA**: Now open (MIPS Open initiative, 2018)
- **Licensing**: Historically commercial, now open
- **Governance**: MIPS Open (Wave Computing)
- **Implementations**: Historical (MIPS Technologies), now limited
- **Software**: GCC, LLVM, Linux (legacy support)

Status:

- Historical importance (education, networking)
- Declining commercial adoption
- Open initiative came too late
- Largely superseded by ARM and RISC-V

**Industry Adoption and Ecosystem**

| Aspect | RISC-V | ARM | MIPS |
|--------|--------|-----|------|
| **Market Share** | Growing (embedded, IoT) | Dominant (mobile, embedded) | Declining |
| **Vendors** | SiFive, Andes, Alibaba, etc. | ARM, Apple, Qualcomm, etc. | Limited |
| **Software Ecosystem** | Growing | Mature | Legacy |
| **Tool Support** | Good (GCC, LLVM) | Excellent | Good (legacy) |
| **Community** | Active, growing | Mature, large | Small |
| **Education** | Increasing | Common | Historical |

ARM dominates commercially. RISC-V is growing rapidly. MIPS is declining.

---

## 17.11 Future Directions

**RISC-V Roadmap**

RISC-V is actively evolving:

- **Ratified extensions**: V (vector), B (bit manipulation), Zicond (conditional ops)
- **In progress**: J (JIT support), P (packed SIMD), Zc (code size reduction)
- **Proposed**: Crypto extensions, memory tagging, capabilities
- **Platform profiles**: RVA22, RVA23 (standardize extension combinations)

Focus areas:

- Performance (vector, out-of-order)
- Security (crypto, memory tagging)
- Code density (compressed, Zc)
- Ecosystem maturity (tools, software)

**ARM Roadmap**

ARM continues to evolve:

- **ARMv9-A**: SVE2 (scalable vector), TME (transactional memory), MTE (memory tagging)
- **Confidential Compute**: Realm Management Extension (RME)
- **AI/ML**: Matrix extensions, neural processing
- **Automotive**: ASIL-D safety, real-time

Focus areas:

- AI/ML acceleration
- Security (confidential computing)
- Automotive and safety
- Performance scaling

**Emerging Extensions**

*RISC-V*:

- **Crypto**: AES, SHA, scalar crypto
- **Vector**: V extension (ratified), improvements ongoing
- **Hypervisor**: H extension (ratified)
- **Bit manipulation**: B extension (ratified)

*ARM*:

- **SVE2**: Scalable vector (successor to NEON)
- **MTE**: Memory Tagging Extension (security)
- **TME**: Transactional Memory Extension
- **SME**: Scalable Matrix Extension (AI/ML)

**Security Features**

Both architectures are adding security features:

*RISC-V*:

- PMP (Physical Memory Protection)
- sPMP (Supervisor PMP)
- Crypto extensions
- Proposed: Memory tagging, capabilities (CHERI-like)

*ARM*:

- TrustZone (secure world)
- MTE (Memory Tagging Extension)
- PAC (Pointer Authentication Codes)
- BTI (Branch Target Identification)

**Industry Trends**

*RISC-V trends*:

- Rapid adoption in China (Alibaba, Huawei)
- Growing in IoT and embedded
- Increasing in data center (SiFive, Ventana)
- Strong in research and education

*ARM trends*:

- Continued dominance in mobile
- Growing in data center (AWS Graviton, Ampere)
- Strong in automotive
- Expanding in AI/ML

*MIPS trends*:

- Declining market share
- Legacy support only
- Some niche applications (networking)

The future favors RISC-V (open, growing) and ARM (mature, dominant). MIPS is largely historical.

---

## Summary

Comparing RISC-V, ARM, and MIPS reveals different architectural philosophies and trade-offs. This chapter examined eleven dimensions of comparison, showing how each architecture approaches the same problems.

**ISA design philosophy** distinguishes the three architectures fundamentally. RISC-V emphasizes modularity and extensibility with a frozen base ISA and composable extensions. ARM emphasizes comprehensiveness and evolution with rich instruction sets and market-driven features. MIPS emphasizes classic RISC simplicity with regular encodings and minimal complexity. These philosophies shape all subsequent design decisions.

**Instruction set complexity** varies significantly. RISC-V has the smallest base ISA (47 instructions for RV32I) with optional extensions. ARM has the largest instruction set (1000+ instructions) covering many use cases. MIPS falls in between with classic RISC simplicity. Encoding formats reflect this—RISC-V and MIPS use regular formats, while ARM uses more complex encodings for expressiveness.

**Register architecture** shows convergence. All three provide 31-32 general-purpose registers with a hardwired zero register (RISC-V x0, ARM XZR, MIPS $0). ARM separates the stack pointer from general registers. All three use separate namespaces for system registers accessed through special instructions.

**Exception and interrupt models** reflect different privilege architectures. RISC-V uses a unified trap model with M/S/U modes. ARM uses exception levels (EL0-EL3) with four exception types. MIPS uses a simpler two-mode model. ARM provides the most sophisticated exception handling, while MIPS is simplest.

**Memory models** all use weak ordering for performance. RISC-V uses RVWMO with fine-grained fence instructions. ARM uses a weak model with multiple barrier types (DMB, DSB, ISB). MIPS evolved from sequential consistency to weak ordering with sync barriers. All three provide atomic instructions for synchronization.

**Virtual memory** shows architectural maturity. RISC-V and ARM use hardware page table walks with multi-level page tables (Sv39/Sv48 for RISC-V, 4-level for ARM). MIPS uses software-managed TLB refill, providing flexibility at the cost of software overhead. All three support multiple page sizes and ASID for efficient context switching.

**Interrupt architecture** ranges from simple to sophisticated. RISC-V uses PLIC for platform interrupts with CLIC for embedded and AIA for servers. ARM uses GIC (Generic Interrupt Controller) with mature multi-core support. MIPS uses a simple 8-line interrupt model. ARM GIC is most mature, RISC-V is evolving, MIPS is simplest.

**Calling conventions** show practical similarities. All three use similar register allocation strategies with 4-8 argument registers, 2 return registers, and saved/temporary register sets. RISC-V and ARM provide 8 argument registers, while MIPS O32 provides only 4. All use downward-growing stacks with 8-16 byte alignment.

**Pipeline and microarchitecture** demonstrate implementation diversity. All three architectures support both in-order and out-of-order implementations. ARM has the most aggressive out-of-order cores (Cortex-A76, Neoverse). RISC-V is catching up with competitive designs (SiFive P550, BOOM). MIPS development has slowed. Branch prediction and cache hierarchies are similar across modern implementations.

**Ecosystem and licensing** represent the fundamental business difference. RISC-V is open and royalty-free, enabling customization and eliminating licensing costs. ARM is proprietary with licensing fees but offers a mature, proven ecosystem. MIPS was commercial but opened too late, now declining. ARM dominates commercially, RISC-V is growing rapidly, MIPS is historical.

**Future directions** show active evolution. RISC-V is adding vector, crypto, and security extensions while standardizing platform profiles. ARM is advancing AI/ML, security (MTE), and confidential computing. Both are adding memory tagging and enhanced security features. Industry trends favor RISC-V (open, growing) and ARM (mature, dominant), while MIPS remains largely historical.

Together, these comparisons show RISC-V as a modern, modular, open alternative to ARM's mature, comprehensive, commercial approach, with MIPS representing classic RISC principles now largely superseded. The choice depends on priorities: openness and customization favor RISC-V, ecosystem maturity favors ARM.
