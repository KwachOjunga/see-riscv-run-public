# Appendix B. Extension Reference

**RISC-V ISA Extensions Quick Reference**

---

> 💡 **Usage Guide**: This appendix is your "menu" during project planning. When deciding which extensions your project needs, reference the decision guide here.

---

## 🧩 Extension Selection Guide (Decision Guide)

### Quick Decision Table

| Extension | Full Name | When to Use? | Dependencies | Recommendation |
|-----------|-----------|--------------|--------------|----------------|
| **M** | Multiply/Divide | Almost all projects need it | None | ✅ Strongly recommended |
| **A** | Atomic | Multi-core, OS, Lock-free | None | ✅ Required for OS |
| **F** | Single Float | Floating-point (games, scientific) | None | As needed |
| **D** | Double Float | High-precision floating-point | F | As needed |
| **C** | Compressed | Reduce code size 20-30% | None | ✅ Strongly recommended |
| **V** | Vector | AI/DSP/Matrix operations | D | Required for HPC |
| **Zba** | Address Gen | Heavy array access `a[i*4]` | None | Performance optimization |
| **Zbb** | Bit Manipulation | Bit operations (popcount, clz) | None | Performance optimization |
| **Zbs** | Single-bit | Single-bit operations | None | Performance optimization |
| **Zicsr** | CSR Access | CSR access (separated from I) | None | ✅ Required for system code |
| **Zifencei** | Fence.I | Instruction cache sync (JIT) | None | Required for self-modifying code |

### Common Combinations (Profiles)

```text
Minimal Embedded:     RV32IMC      (Multiply + Compressed)
Standard Embedded:    RV32IMAC     (+ Atomic operations)
Application Proc:     RV64IMAFDC   (= RV64GC, full general-purpose)
High-Performance:     RV64GCV      (+ Vector)
```

### Dependency Graph

```text
        ┌─────┐
        │  I  │ (Base ISA)
        └──┬──┘
           │
    ┌──────┼──────┬──────┬──────┐
    ▼      ▼      ▼      ▼      ▼
  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐
  │ M │  │ A │  │ C │  │ F │  │Zicsr│
  └───┘  └───┘  └───┘  └─┬─┘  └───┘
                         │
                         ▼
                       ┌───┐
                       │ D │
                       └─┬─┘
                         │
                         ▼
                       ┌───┐
                       │ V │
                       └───┘
```

---

## ⚠️ Common Pitfalls

### Pitfall 1: Thinking G Includes C

**Misconception**: `RV64G` includes compressed instructions.

**Truth**: `G = IMAFD`, does NOT include C. For compressed instructions, explicitly write `RV64GC`.

```bash
# ❌ Wrong: Assuming G has compression
riscv64-unknown-elf-gcc -march=rv64g ...

# ✅ Correct: Explicitly add C
riscv64-unknown-elf-gcc -march=rv64gc ...
```

### Pitfall 2: Forgetting Zicsr and Zifencei

**Background**: Starting from RISC-V 2.1 spec, CSR instructions and FENCE.I were separated from I.

**Impact**: Some toolchains require explicit specification.

```bash
# If compiler complains about missing csrr/csrw
riscv64-unknown-elf-gcc -march=rv64gc_zicsr_zifencei ...
```

### Pitfall 3: misa Can Only Be Read in M-mode

**Error Scenario**: Trying to read `misa` to detect extensions in S-mode or U-mode.

**Solution**: Use SBI query, or record during M-mode initialization.

```c
// ❌ In S-mode, this triggers Illegal Instruction
uint64_t misa;
asm volatile ("csrr %0, misa" : "=r" (misa));

// ✅ Query via SBI or Device Tree
// Or save misa to global variable during M-mode boot
```

---

This appendix provides a comprehensive reference for RISC-V ISA extensions. RISC-V's modular design allows implementations to include only the extensions they need, from minimal embedded systems to high-performance application processors.

---

## B.1 Base ISAs

| Base ISA | Description | Register Width | Address Space |
|----------|-------------|----------------|---------------|
| **RV32I** | 32-bit integer base | 32 bits | 32-bit (4 GB) |
| **RV64I** | 64-bit integer base | 64 bits | 64-bit (16 EB) |
| **RV128I** | 128-bit integer base (future) | 128 bits | 128-bit |
| **RV32E** | Embedded variant (16 registers) | 32 bits | 32-bit (4 GB) |

**RV32I**: The base 32-bit integer instruction set. Includes 32 general-purpose registers (x0-x31), integer arithmetic, logical operations, loads/stores, branches, and jumps. Sufficient for simple embedded systems.

**RV64I**: Extends RV32I to 64-bit. Adds 64-bit arithmetic operations (ADDW, SUBW, etc.) and 64-bit loads/stores (LD, SD). Registers are 64 bits wide. Used for application processors and servers.

**RV32E**: Reduced version of RV32I with only 16 registers (x0-x15). Designed for ultra-low-cost embedded systems where area is critical. Reduces register file size by 50%.

---

## B.2 Standard Extensions

### M Extension: Integer Multiplication and Division

**Status**: Ratified  
**Description**: Adds integer multiply, divide, and remainder instructions.

| Instruction | Description |
|-------------|-------------|
| **MUL** | Multiply (lower XLEN bits) |
| **MULH** | Multiply high (signed × signed) |
| **MULHSU** | Multiply high (signed × unsigned) |
| **MULHU** | Multiply high (unsigned × unsigned) |
| **DIV** | Divide (signed) |
| **DIVU** | Divide (unsigned) |
| **REM** | Remainder (signed) |
| **REMU** | Remainder (unsigned) |

**RV64 Additions**: MULW, DIVW, DIVUW, REMW, REMUW (32-bit variants)

**Usage**: Essential for most applications. Division is expensive in hardware, so minimal systems may omit M extension and use software division.

---

### A Extension: Atomic Instructions

**Status**: Ratified
**Description**: Adds atomic memory operations for synchronization.

**Load-Reserved/Store-Conditional**:

| Instruction | Description |
|-------------|-------------|
| **LR.W/D** | Load-Reserved Word/Doubleword |
| **SC.W/D** | Store-Conditional Word/Doubleword |

**Atomic Memory Operations (AMO)**:

| Instruction | Description |
|-------------|-------------|
| **AMOSWAP.W/D** | Atomic swap |
| **AMOADD.W/D** | Atomic add |
| **AMOAND.W/D** | Atomic AND |
| **AMOOR.W/D** | Atomic OR |
| **AMOXOR.W/D** | Atomic XOR |
| **AMOMAX.W/D** | Atomic maximum (signed) |
| **AMOMAXU.W/D** | Atomic maximum (unsigned) |
| **AMOMIN.W/D** | Atomic minimum (signed) |
| **AMOMINU.W/D** | Atomic minimum (unsigned) |

**Ordering Annotations**: .aq (acquire), .rl (release), .aqrl (both)

**Usage**: Required for multi-core systems and lock-free algorithms.

---

### F Extension: Single-Precision Floating-Point

**Status**: Ratified  
**Description**: Adds 32 floating-point registers (f0-f31) and single-precision (32-bit) floating-point operations.

**Registers**: 32 × 32-bit floating-point registers (f0-f31)

**Instructions**:

- Arithmetic: FADD.S, FSUB.S, FMUL.S, FDIV.S, FSQRT.S
- Fused Multiply-Add: FMADD.S, FMSUB.S, FNMADD.S, FNMSUB.S
- Comparison: FEQ.S, FLT.S, FLE.S
- Conversion: FCVT.W.S, FCVT.S.W, FCVT.L.S, FCVT.S.L
- Move: FMV.X.W, FMV.W.X
- Load/Store: FLW, FSW
- Sign Injection: FSGNJ.S, FSGNJN.S, FSGNJX.S
- Min/Max: FMIN.S, FMAX.S
- Classification: FCLASS.S

**CSRs**: fflags, frm, fcsr (floating-point control and status)

---

### D Extension: Double-Precision Floating-Point

**Status**: Ratified  
**Description**: Extends F extension to support double-precision (64-bit) floating-point.

**Requires**: F extension

**Registers**: Extends f0-f31 to 64 bits each

**Instructions**: Same as F extension but with .D suffix (FADD.D, FMUL.D, etc.)

**Additional Conversions**: FCVT.S.D, FCVT.D.S (convert between single and double)

**Load/Store**: FLD, FSD (64-bit loads/stores)

---

### C Extension: Compressed Instructions

**Status**: Ratified  
**Description**: Adds 16-bit compressed instructions to reduce code size.

**Encoding**: 16-bit instructions (bits [1:0] ≠ 11) intermixed with 32-bit instructions

**Instruction Categories**:

- Loads/Stores: C.LW, C.LD, C.SW, C.SD, C.LWSP, C.LDSP, C.SWSP, C.SDSP
- Arithmetic: C.ADDI, C.ADDIW, C.ADDI16SP, C.ADDI4SPN, C.LI, C.LUI
- Logical: C.ANDI, C.SLLI, C.SRLI, C.SRAI
- Branches: C.BEQZ, C.BNEZ
- Jumps: C.J, C.JAL, C.JR, C.JALR
- Register Move: C.MV, C.ADD
- Special: C.NOP, C.EBREAK

**Code Size Reduction**: Typically 25-30% smaller code compared to RV32I/RV64I alone

**Usage**: Highly recommended for all systems. Minimal hardware cost, significant code density improvement.

---

### V Extension: Vector Operations

**Status**: Ratified (v1.0)  
**Description**: Adds vector processing with variable-length vectors.

**Registers**: 32 vector registers (v0-v31), each with configurable element width and length

**CSRs**:

- **vtype**: Vector type (element width, LMUL)
- **vl**: Vector length
- **vstart**: Vector start index (for resuming after exception)
- **vxrm**: Vector fixed-point rounding mode
- **vxsat**: Vector fixed-point saturation flag

**Configuration**: vsetvl, vsetvli (set vector length and type)

**Instruction Categories**:

- Arithmetic: VADD, VSUB, VMUL, VDIV, VREM
- Logical: VAND, VOR, VXOR
- Shift: VSLL, VSRL, VSRA
- Comparison: VMSEQ, VMSNE, VMSLT, VMSLE, VMSGTU
- Load/Store: VLE, VSE (unit-stride), VLSE, VSSE (strided), VLXEI, VSXEI (indexed)
- Reduction: VREDSUM, VREDMAX, VREDMIN
- Mask: VMAND, VMOR, VMXOR, VMNOT
- Permutation: VSLIDEUP, VSLIDEDOWN, VRGATHER
- Floating-Point: VFADD, VFMUL, VFDIV, VFSQRT, VFMADD

**Usage**: High-performance computing, DSP, machine learning

---

## B.3 Bit Manipulation Extensions

### Zba: Address Generation

**Status**: Ratified
**Description**: Instructions for address calculation.

| Instruction | Description |
|-------------|-------------|
| **SH1ADD** | Shift left by 1 and add (rs1 << 1) + rs2 |
| **SH2ADD** | Shift left by 2 and add (rs1 << 2) + rs2 |
| **SH3ADD** | Shift left by 3 and add (rs1 << 3) + rs2 |

**Usage**: Efficient array indexing (e.g., `a[i]` where elements are 2, 4, or 8 bytes)

---

### Zbb: Basic Bit Manipulation

**Status**: Ratified
**Description**: Common bit manipulation operations.

| Instruction | Description |
|-------------|-------------|
| **ANDN** | AND with inverted operand |
| **ORN** | OR with inverted operand |
| **XNOR** | XOR with inverted operand |
| **CLZ** | Count leading zeros |
| **CTZ** | Count trailing zeros |
| **CPOP** | Count population (number of 1 bits) |
| **MAX** | Maximum (signed) |
| **MAXU** | Maximum (unsigned) |
| **MIN** | Minimum (signed) |
| **MINU** | Minimum (unsigned) |
| **SEXT.B** | Sign-extend byte |
| **SEXT.H** | Sign-extend halfword |
| **ZEXT.H** | Zero-extend halfword |
| **ROL** | Rotate left |
| **ROR** | Rotate right |
| **RORI** | Rotate right immediate |
| **ORC.B** | OR-combine bytes |
| **REV8** | Byte-reverse (endian swap) |

**Usage**: Cryptography, compression, bit-field manipulation

---

### Zbc: Carry-Less Multiplication

**Status**: Ratified
**Description**: Carry-less multiplication for cryptography.

| Instruction | Description |
|-------------|-------------|
| **CLMUL** | Carry-less multiply (lower half) |
| **CLMULH** | Carry-less multiply (upper half) |
| **CLMULR** | Carry-less multiply (reversed) |

**Usage**: AES-GCM, CRC calculation

---

### Zbs: Single-Bit Instructions

**Status**: Ratified
**Description**: Single-bit set, clear, invert, extract.

| Instruction | Description |
|-------------|-------------|
| **BCLR** | Bit clear |
| **BCLRI** | Bit clear immediate |
| **BEXT** | Bit extract |
| **BEXTI** | Bit extract immediate |
| **BINV** | Bit invert |
| **BINVI** | Bit invert immediate |
| **BSET** | Bit set |
| **BSETI** | Bit set immediate |

**Usage**: Bit-field manipulation, flag management

---

## B.4 Compressed Extensions (Zc*)

### Zcb: Code Size Reduction (16-bit)

**Status**: Ratified
**Description**: Additional 16-bit instructions for code density.

**Instructions**: C.LBU, C.LHU, C.LH, C.SB, C.SH, C.ZEXT.B, C.SEXT.B, C.SEXT.H, C.ZEXT.H, C.ZEXT.W, C.NOT, C.MUL

**Usage**: Further code size reduction beyond C extension

---

### Zcmp: Push/Pop and Move

**Status**: Ratified
**Description**: Push/pop multiple registers, double move.

**Instructions**:

- CM.PUSH: Push registers to stack
- CM.POP: Pop registers from stack
- CM.POPRET: Pop and return
- CM.POPRETZ: Pop, return, and zero a0
- CM.MVA01S: Move two registers to a0/a1
- CM.MVSA01: Move a0/a1 to two registers

**Usage**: Function prologue/epilogue optimization

---

### Zcmt: Table Jump

**Status**: Ratified
**Description**: Indirect jump via table for switch statements.

**Instructions**: CM.JT, CM.JALT (jump via table)

**Usage**: Efficient switch/case implementation

---

## B.5 Cache Management Extensions

### Zicbom: Cache Block Management

**Status**: Ratified
**Description**: Cache block clean, flush, and invalidate.

**Instructions**:

- CBO.CLEAN: Clean cache block (write back if dirty)
- CBO.FLUSH: Flush cache block (write back and invalidate)
- CBO.INVAL: Invalidate cache block

**Usage**: DMA coherence, cache maintenance

---

### Zicbop: Cache Block Prefetch

**Status**: Ratified
**Description**: Prefetch hints for cache optimization.

**Instructions**:

- PREFETCH.R: Prefetch for read
- PREFETCH.W: Prefetch for write
- PREFETCH.I: Prefetch for instruction

**Usage**: Performance optimization, prefetching

---

### Zicboz: Cache Block Zero

**Status**: Ratified
**Description**: Zero a cache block efficiently.

**Instructions**: CBO.ZERO (zero cache block)

**Usage**: Fast memory initialization

---

## B.6 Privileged Extensions

### Zicsr: CSR Instructions

**Status**: Ratified (part of base)
**Description**: Control and Status Register access instructions.

**Instructions**: CSRRW, CSRRS, CSRRC, CSRRWI, CSRRSI, CSRRCI

**Usage**: Required for privileged software (OS, firmware)

---

### Zifencei: Instruction Fetch Fence

**Status**: Ratified (part of base)
**Description**: Synchronize instruction and data caches.

**Instructions**: FENCE.I

**Usage**: Self-modifying code, JIT compilation, code loading

---

### Zihintpause: Pause Hint

**Status**: Ratified
**Description**: Hint for spin-wait loops.

**Instructions**: PAUSE (encoded as FENCE with specific operands)

**Usage**: Reduce power in spin-locks

---

## B.7 Hypervisor Extension (H)

**Status**: Ratified
**Description**: Support for virtualization.

**Features**:

- Two-stage address translation (VS-stage and G-stage)
- Virtual supervisor mode (VS-mode)
- Hypervisor CSRs (hstatus, hedeleg, hideleg, hgatp, etc.)
- Virtual interrupt management

**Instructions**: HLV, HSV (hypervisor load/store), HFENCE.VVMA, HFENCE.GVMA

**Usage**: Virtual machines, hypervisors (KVM, Xen)

---

## B.8 Cryptography Extensions

### Zk: Scalar Cryptography

**Status**: Ratified
**Description**: Cryptographic instructions for AES, SHA, SM3, SM4.

**Sub-extensions**:

- **Zkn**: NIST algorithms (AES, SHA-256, SHA-512)
- **Zks**: ShangMi algorithms (SM3, SM4)
- **Zkb**: Bit manipulation for crypto
- **Zkr**: Entropy source (seed CSR)

**Instructions**:

- AES: AES32ESI, AES32ESMI, AES32DSI, AES32DSMI, AES64ES, AES64DS, etc.
- SHA-256: SHA256SIG0, SHA256SIG1, SHA256SUM0, SHA256SUM1
- SHA-512: SHA512SIG0, SHA512SIG1, SHA512SUM0, SHA512SUM1
- SM3: SM3P0, SM3P1
- SM4: SM4ED, SM4KS

**Usage**: Secure boot, TLS, disk encryption

---

## B.9 Extension Combinations

### Common Combinations

| Combination | Name | Description |
|-------------|------|-------------|
| **RV32I** | Base | Minimal 32-bit system |
| **RV32IM** | - | Base + multiply/divide |
| **RV32IMC** | - | Base + multiply + compressed |
| **RV32IMAC** | - | Base + multiply + atomic + compressed |
| **RV32IMAFC** | - | Base + M + A + F + C |
| **RV32IMAFDC** | - | Base + M + A + F + D + C |
| **RV32GC** | General | RV32IMAFD_Zicsr_Zifencei + C |
| **RV64GC** | General | RV64IMAFD_Zicsr_Zifencei + C |

**RV32G / RV64G**: "General-purpose" configuration = IMAFD + Zicsr + Zifencei

---

## B.10 Platform Profiles

### RVA22 Profile (Application Processors)

**Base**: RV64I
**Mandatory Extensions**:

- M, A, F, D, C (IMAFD + C)
- Zicsr, Zifencei
- Zba, Zbb, Zbs (bit manipulation)
- Zihintpause
- Zicbom, Zicbop, Zicboz (cache management)
- Sv39 (virtual memory)
- Privileged spec v1.12+

**Optional Extensions**: V, H, Zk

**Usage**: Linux-capable application processors

---

### RVA23 Profile (Next-generation)

**Adds to RVA22**:

- Sv48 or Sv57 (larger virtual address space)
- Zihintntl (non-temporal locality hints)
- Zicond (conditional operations)
- Zawrs (wait-on-reservation-set)
- Zcb, Zcmp, Zcmt (additional compressed)
- Vector extension (V) mandatory

**Usage**: High-performance servers, HPC

---

### RVM23 Profile (Microcontrollers)

**Base**: RV32I or RV64I
**Mandatory Extensions**:

- M, C
- Zicsr, Zifencei
- Zba, Zbb, Zbs
- Zicbop, Zicboz
- PMP (Physical Memory Protection)

**Optional**: A, F, D, V

**Usage**: Embedded microcontrollers

---

## B.11 Extension Naming Convention

**Format**: `RV[32|64|128][I|E][Extensions]`

**Examples**:

- `RV32I`: 32-bit base integer
- `RV64IMAC`: 64-bit with M, A, C extensions
- `RV32GC`: 32-bit general-purpose with compressed
- `RV64GCV`: 64-bit general-purpose with compressed and vector

**Ordering**: Extensions listed in canonical order (IMAFDQCV...)

---

## B.12 Extension Detection

### Runtime Detection (misa CSR)

```assembly
csrr t0, misa
andi t1, t0, (1 << 0)   # Check 'A' extension (bit 0)
bnez t1, has_atomic
```

**misa Bit Assignments**:

- Bit 0: A (Atomic)
- Bit 2: C (Compressed)
- Bit 3: D (Double-precision FP)
- Bit 4: E (Embedded - RV32E)
- Bit 5: F (Single-precision FP)
- Bit 7: H (Hypervisor)
- Bit 8: I (Base integer ISA)
- Bit 12: M (Multiply/Divide)
- Bit 20: U (User mode)
- Bit 21: V (Vector)

---

## B.13 References

- **RISC-V ISA Manual**: Complete extension specifications
- **RISC-V Profiles**: RVA22, RVA23, RVM23 specifications
- **Extension Specifications**: Individual ratified extension documents

---
