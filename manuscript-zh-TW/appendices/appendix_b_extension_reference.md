# Appendix B. Extension Reference

**RISC-V ISA Extension 快速參考**

---

本附錄提供 RISC-V ISA extension 的全面參考。RISC-V 的模組化設計允許 implementation 僅包含所需的 extension，從最小的 embedded system 到高性能 application processor。

---

## B.1 Base ISAs

| Base ISA | Description | Register Width | Address Space |
|----------|-------------|----------------|---------------|
| **RV32I** | 32-bit integer base | 32 bits | 32-bit (4 GB) |
| **RV64I** | 64-bit integer base | 64 bits | 64-bit (16 EB) |
| **RV128I** | 128-bit integer base (future) | 128 bits | 128-bit |
| **RV32E** | Embedded variant (16 registers) | 32 bits | 32-bit (4 GB) |

**RV32I**：32-bit integer instruction set 的 base。包含 32 個 general-purpose register（x0-x31）、integer arithmetic、logical operation、load/store、branch 和 jump。足以支援簡單的 embedded system。

**RV64I**：將 RV32I 擴展到 64-bit。新增 64-bit arithmetic operation（ADDW、SUBW 等）和 64-bit load/store（LD、SD）。Register 為 64 bit 寬。用於 application processor 和 server。

**RV32E**：RV32I 的精簡版本，僅有 16 個 register（x0-x15）。設計用於超低成本 embedded system，其中 area 至關重要。Register file size 減少 50%。

---

## B.2 Standard Extensions

### M Extension: Integer Multiplication and Division

**Status**: Ratified  
**Description**: 新增 integer multiply、divide 和 remainder instruction。

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

**RV64 Additions**: MULW, DIVW, DIVUW, REMW, REMUW (32-bit variant)

**Usage**: 大多數 application 必需。Division 在 hardware 中成本高，因此最小系統可能省略 M extension 並使用 software division。

---

### A Extension: Atomic Instructions

**Status**: Ratified
**Description**: 新增 atomic memory operation 用於 synchronization。

**Load-Reserved/Store-Conditional**：

| Instruction | Description |
|-------------|-------------|
| **LR.W/D** | Load-Reserved Word/Doubleword |
| **SC.W/D** | Store-Conditional Word/Doubleword |

**Atomic Memory Operations (AMO)**：

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

**Usage**: Multi-core system 和 lock-free algorithm 必需。

---

### F Extension: Single-Precision Floating-Point

**Status**: Ratified  
**Description**: 新增 32 個 floating-point register（f0-f31）和 single-precision（32-bit）floating-point operation。

**Registers**: 32 × 32-bit floating-point register（f0-f31）

**Instructions**：

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
**Description**: 擴展 F extension 以支援 double-precision（64-bit）floating-point。

**Requires**: F extension

**Registers**: 將 f0-f31 擴展到每個 64 bit

**Instructions**: 與 F extension 相同，但使用 .D suffix（FADD.D、FMUL.D 等）

**Additional Conversions**: FCVT.S.D, FCVT.D.S (convert between single and double)

**Load/Store**: FLD, FSD (64-bit load/store)

---

### C Extension: Compressed Instructions

**Status**: Ratified  
**Description**: 新增 16-bit compressed instruction 以減少 code size。

**Encoding**: 16-bit instruction（bits [1:0] ≠ 11）與 32-bit instruction 混合

**Instruction Categories**：

- Loads/Stores: C.LW, C.LD, C.SW, C.SD, C.LWSP, C.LDSP, C.SWSP, C.SDSP
- Arithmetic: C.ADDI, C.ADDIW, C.ADDI16SP, C.ADDI4SPN, C.LI, C.LUI
- Logical: C.ANDI, C.SLLI, C.SRLI, C.SRAI
- Branches: C.BEQZ, C.BNEZ
- Jumps: C.J, C.JAL, C.JR, C.JALR
- Register Move: C.MV, C.ADD
- Special: C.NOP, C.EBREAK

**Code Size Reduction**: 與單獨的 RV32I/RV64I 相比，code 通常小 25-30%

**Usage**: 強烈建議所有系統使用。Hardware 成本最小，code density 顯著改善。

---

### V Extension: Vector Operations

**Status**: Ratified (v1.0)
**Description**: 新增 vector processing，支援 variable-length vector。

**Registers**: 32 個 vector register（v0-v31），每個具有可配置的 element width 和 length

**CSRs**：

- **vtype**: Vector type（element width、LMUL）
- **vl**: Vector length
- **vstart**: Vector start index（用於 exception 後恢復）
- **vxrm**: Vector fixed-point rounding mode
- **vxsat**: Vector fixed-point saturation flag

**Configuration**: vsetvl, vsetvli (set vector length and type)

**Instruction Categories**：

- Arithmetic: VADD, VSUB, VMUL, VDIV, VREM
- Logical: VAND, VOR, VXOR
- Shift: VSLL, VSRL, VSRA
- Comparison: VMSEQ, VMSNE, VMSLT, VMSLE, VMSGTU
- Load/Store: VLE, VSE (unit-stride), VLSE, VSSE (strided), VLXEI, VSXEI (indexed)
- Reduction: VREDSUM, VREDMAX, VREDMIN
- Mask: VMAND, VMOR, VMXOR, VMNOT
- Permutation: VSLIDEUP, VSLIDEDOWN, VRGATHER
- Floating-Point: VFADD, VFMUL, VFDIV, VFSQRT, VFMADD

**Usage**: High-performance computing、DSP、machine learning

---

## B.3 Bit Manipulation Extensions

### Zba: Address Generation

**Status**: Ratified
**Description**: Address calculation 的 instruction。

| Instruction | Description |
|-------------|-------------|
| **SH1ADD** | Shift left by 1 and add (rs1 << 1) + rs2 |
| **SH2ADD** | Shift left by 2 and add (rs1 << 2) + rs2 |
| **SH3ADD** | Shift left by 3 and add (rs1 << 3) + rs2 |

**Usage**: 高效的 array indexing（例如 `a[i]`，其中 element 為 2、4 或 8 byte）

---

### Zbb: Basic Bit Manipulation

**Status**: Ratified
**Description**: 常見的 bit manipulation operation。

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

**Usage**: Cryptography、compression、bit-field manipulation

---

### Zbc: Carry-Less Multiplication

**Status**: Ratified
**Description**: Cryptography 的 carry-less multiplication。

| Instruction | Description |
|-------------|-------------|
| **CLMUL** | Carry-less multiply (lower half) |
| **CLMULH** | Carry-less multiply (upper half) |
| **CLMULR** | Carry-less multiply (reversed) |

**Usage**: AES-GCM、CRC calculation

---

### Zbs: Single-Bit Instructions

**Status**: Ratified
**Description**: Single-bit set、clear、invert、extract。

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

**Usage**: Bit-field manipulation、flag management

---

## B.4 Compressed Extensions (Zc*)

### Zcb: Code Size Reduction (16-bit)

**Status**: Ratified
**Description**: 額外的 16-bit instruction 用於 code density。

**Instructions**: C.LBU, C.LHU, C.LH, C.SB, C.SH, C.ZEXT.B, C.SEXT.B, C.SEXT.H, C.ZEXT.H, C.ZEXT.W, C.NOT, C.MUL

**Usage**: 在 C extension 之外進一步減少 code size

---

### Zcmp: Push/Pop and Move

**Status**: Ratified
**Description**: Push/pop multiple register、double move。

**Instructions**：

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
**Description**: 通過 table 的 indirect jump，用於 switch statement。

**Instructions**: CM.JT, CM.JALT (jump via table)

**Usage**: 高效的 switch/case implementation

---

## B.5 Cache Management Extensions

### Zicbom: Cache Block Management

**Status**: Ratified
**Description**: Cache block clean、flush 和 invalidate。

**Instructions**：

- CBO.CLEAN: Clean cache block (write back if dirty)
- CBO.FLUSH: Flush cache block (write back and invalidate)
- CBO.INVAL: Invalidate cache block

**Usage**: DMA coherence、cache maintenance

---

### Zicbop: Cache Block Prefetch

**Status**: Ratified
**Description**: Cache optimization 的 prefetch hint。

**Instructions**：

- PREFETCH.R: Prefetch for read
- PREFETCH.W: Prefetch for write
- PREFETCH.I: Prefetch for instruction

**Usage**: Performance optimization、prefetching

---

### Zicboz: Cache Block Zero

**Status**: Ratified
**Description**: 高效地 zero cache block。

**Instructions**: CBO.ZERO (zero cache block)

**Usage**: Fast memory initialization

---

## B.6 Privileged Extensions

### Zicsr: CSR Instructions

**Status**: Ratified (part of base)
**Description**: Control and Status Register access instruction。

**Instructions**: CSRRW, CSRRS, CSRRC, CSRRWI, CSRRSI, CSRRCI

**Usage**: Privileged software（OS、firmware）必需

---

### Zifencei: Instruction Fetch Fence

**Status**: Ratified (part of base)
**Description**: Synchronize instruction 和 data cache。

**Instructions**: FENCE.I

**Usage**: Self-modifying code、JIT compilation、code loading

---

### Zihintpause: Pause Hint

**Status**: Ratified
**Description**: Spin-wait loop 的 hint。

**Instructions**: PAUSE (encoded as FENCE with specific operands)

**Usage**: Spin-lock 中減少 power

---

## B.7 Hypervisor Extension (H)

**Status**: Ratified
**Description**: Virtualization 支援。

**Features**：

- Two-stage address translation（VS-stage 和 G-stage）
- Virtual supervisor mode（VS-mode）
- Hypervisor CSR（hstatus、hedeleg、hideleg、hgatp 等）
- Virtual interrupt management

**Instructions**: HLV, HSV (hypervisor load/store), HFENCE.VVMA, HFENCE.GVMA

**Usage**: Virtual machine、hypervisor（KVM、Xen）

---

## B.8 Cryptography Extensions

### Zk: Scalar Cryptography

**Status**: Ratified
**Description**: AES、SHA、SM3、SM4 的 cryptographic instruction。

**Sub-extensions**：

- **Zkn**: NIST algorithm（AES、SHA-256、SHA-512）
- **Zks**: ShangMi algorithm（SM3、SM4）
- **Zkb**: Bit manipulation for crypto
- **Zkr**: Entropy source（seed CSR）

**Instructions**：

- AES: AES32ESI, AES32ESMI, AES32DSI, AES32DSMI, AES64ES, AES64DS, etc.
- SHA-256: SHA256SIG0, SHA256SIG1, SHA256SUM0, SHA256SUM1
- SHA-512: SHA512SIG0, SHA512SIG1, SHA512SUM0, SHA512SUM1
- SM3: SM3P0, SM3P1
- SM4: SM4ED, SM4KS

**Usage**: Secure boot、TLS、disk encryption

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

**Mandatory Extensions**：

- M, A, F, D, C (IMAFD + C)
- Zicsr, Zifencei
- Zba, Zbb, Zbs (bit manipulation)
- Zihintpause
- Zicbom, Zicbop, Zicboz (cache management)
- Sv39 (virtual memory)
- Privileged spec v1.12+

**Optional Extensions**: V, H, Zk

**Usage**: Linux-capable application processor

---

### RVA23 Profile (Next-generation)

**Adds to RVA22**：

- Sv48 or Sv57 (larger virtual address space)
- Zihintntl (non-temporal locality hints)
- Zicond (conditional operations)
- Zawrs (wait-on-reservation-set)
- Zcb, Zcmp, Zcmt (additional compressed)
- Vector extension (V) mandatory

**Usage**: High-performance server、HPC

---

### RVM23 Profile (Microcontrollers)

**Base**: RV32I or RV64I

**Mandatory Extensions**：

- M, C
- Zicsr, Zifencei
- Zba, Zbb, Zbs
- Zicbop, Zicboz
- PMP (Physical Memory Protection)

**Optional**: A, F, D, V

**Usage**: Embedded microcontroller

---

## B.11 Extension Naming Convention

**Format**: `RV[32|64|128][I|E][Extensions]`

**Examples**：

- `RV32I`: 32-bit base integer
- `RV64IMAC`: 64-bit with M, A, C extensions
- `RV32GC`: 32-bit general-purpose with compressed
- `RV64GCV`: 64-bit general-purpose with compressed and vector

**Ordering**: Extension 按 canonical order 列出（IMAFDQCV...）

---

## B.12 Extension Detection

### Runtime Detection (misa CSR)

```assembly
csrr t0, misa
andi t1, t0, (1 << 0)   # Check 'A' extension (bit 0)
bnez t1, has_atomic
```

**misa Bit Assignments**：

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

- **RISC-V ISA Manual**: Complete extension specification
- **RISC-V Profiles**: RVA22、RVA23、RVM23 specification
- **Extension Specifications**: Individual ratified extension document

---
