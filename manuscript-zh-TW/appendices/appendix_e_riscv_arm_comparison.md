# Appendix E. RISC-V vs ARM Instruction Comparison

**Quick Reference for Porting Between RISC-V and ARM**

---

本附錄提供 RISC-V 和 ARM（ARMv8-A AArch64）之間常見 instruction 的並排比較。此參考旨在幫助開發者在兩種架構之間移植程式碼。

---

## E.1 Arithmetic and Logical Instructions

### Integer Arithmetic

| Operation | RISC-V | ARM |
|-----------|--------|-----|
| **Add** | `add rd, rs1, rs2` | `ADD Xd, Xn, Xm` |
| **Add immediate** | `addi rd, rs1, imm` | `ADD Xd, Xn, #imm` |
| **Subtract** | `sub rd, rs1, rs2` | `SUB Xd, Xn, Xm` |
| **Subtract immediate** | `addi rd, rs1, -imm` | `SUB Xd, Xn, #imm` |
| **Negate** | `sub rd, x0, rs` | `NEG Xd, Xm` |
| **Multiply** | `mul rd, rs1, rs2` | `MUL Xd, Xn, Xm` |
| **Multiply high (signed)** | `mulh rd, rs1, rs2` | `SMULH Xd, Xn, Xm` |
| **Multiply high (unsigned)** | `mulhu rd, rs1, rs2` | `UMULH Xd, Xn, Xm` |
| **Divide (signed)** | `div rd, rs1, rs2` | `SDIV Xd, Xn, Xm` |
| **Divide (unsigned)** | `divu rd, rs1, rs2` | `UDIV Xd, Xn, Xm` |
| **Remainder (signed)** | `rem rd, rs1, rs2` | No direct equivalent (use MSUB) |
| **Remainder (unsigned)** | `remu rd, rs1, rs2` | No direct equivalent (use MSUB) |

**Note**: ARM 沒有直接的 remainder instruction。使用：`MSUB Xd, Xn, Xm, Xo`（Xd = Xo - Xn * Xm）

---

### Logical Operations

| Operation | RISC-V | ARM |
|-----------|--------|-----|
| **AND** | `and rd, rs1, rs2` | `AND Xd, Xn, Xm` |
| **AND immediate** | `andi rd, rs1, imm` | `AND Xd, Xn, #imm` |
| **OR** | `or rd, rs1, rs2` | `ORR Xd, Xn, Xm` |
| **OR immediate** | `ori rd, rs1, imm` | `ORR Xd, Xn, #imm` |
| **XOR** | `xor rd, rs1, rs2` | `EOR Xd, Xn, Xm` |
| **XOR immediate** | `xori rd, rs1, imm` | `EOR Xd, Xn, #imm` |
| **NOT** | `xori rd, rs, -1` | `MVN Xd, Xm` |
| **AND NOT** | `andn rd, rs1, rs2` (Zbb) | `BIC Xd, Xn, Xm` |
| **OR NOT** | `orn rd, rs1, rs2` (Zbb) | `ORN Xd, Xn, Xm` |

---

### Shift Operations

| Operation | RISC-V | ARM |
|-----------|--------|-----|
| **Shift left logical** | `sll rd, rs1, rs2` | `LSL Xd, Xn, Xm` |
| **Shift left immediate** | `slli rd, rs1, shamt` | `LSL Xd, Xn, #imm` |
| **Shift right logical** | `srl rd, rs1, rs2` | `LSR Xd, Xn, Xm` |
| **Shift right immediate** | `srli rd, rs1, shamt` | `LSR Xd, Xn, #imm` |
| **Shift right arithmetic** | `sra rd, rs1, rs2` | `ASR Xd, Xn, Xm` |
| **Shift right arith imm** | `srai rd, rs1, shamt` | `ASR Xd, Xn, #imm` |
| **Rotate right** | `ror rd, rs1, rs2` (Zbb) | `ROR Xd, Xn, Xm` |
| **Rotate right immediate** | `rori rd, rs1, shamt` (Zbb) | `ROR Xd, Xn, #imm` |

---

## E.2 Load and Store Instructions

### Basic Loads

| Operation | RISC-V | ARM |
|-----------|--------|-----|
| **Load byte (signed)** | `lb rd, offset(rs1)` | `LDRSB Xd, [Xn, #offset]` |
| **Load byte (unsigned)** | `lbu rd, offset(rs1)` | `LDRB Wd, [Xn, #offset]` |
| **Load halfword (signed)** | `lh rd, offset(rs1)` | `LDRSH Xd, [Xn, #offset]` |
| **Load halfword (unsigned)** | `lhu rd, offset(rs1)` | `LDRH Wd, [Xn, #offset]` |
| **Load word (signed)** | `lw rd, offset(rs1)` | `LDRSW Xd, [Xn, #offset]` |
| **Load word (unsigned)** | `lwu rd, offset(rs1)` | `LDR Wd, [Xn, #offset]` |
| **Load doubleword** | `ld rd, offset(rs1)` | `LDR Xd, [Xn, #offset]` |

---

### Basic Stores

| Operation | RISC-V | ARM |
|-----------|--------|-----|
| **Store byte** | `sb rs2, offset(rs1)` | `STRB Wd, [Xn, #offset]` |
| **Store halfword** | `sh rs2, offset(rs1)` | `STRH Wd, [Xn, #offset]` |
| **Store word** | `sw rs2, offset(rs1)` | `STR Wd, [Xn, #offset]` |
| **Store doubleword** | `sd rs2, offset(rs1)` | `STR Xd, [Xn, #offset]` |

---

### Addressing Modes

**RISC-V**: 僅支援 base+offset

```assembly
lw t0, 8(sp)      # Load from sp + 8
```

**ARM**: 多種模式

```assembly
LDR X0, [SP, #8]       # Base + offset
LDR X0, [SP, #8]!      # Pre-indexed (update SP)
LDR X0, [SP], #8       # Post-indexed (update SP after)
LDR X0, [SP, X1]       # Base + register
LDR X0, [SP, X1, LSL #3]  # Base + shifted register
```

**Porting Note**: RISC-V 需要分別的 add/sub 來實現 pre/post-indexed addressing：

```assembly
# ARM: LDR X0, [SP], #8
# RISC-V equivalent:
ld t0, 0(sp)
addi sp, sp, 8
```

---

## E.3 Branch and Jump Instructions

### Conditional Branches

| Operation | RISC-V | ARM |
|-----------|--------|-----|
| **Branch if equal** | `beq rs1, rs2, label` | `CMP Xn, Xm` + `B.EQ label` |
| **Branch if not equal** | `bne rs1, rs2, label` | `CMP Xn, Xm` + `B.NE label` |
| **Branch if less than** | `blt rs1, rs2, label` | `CMP Xn, Xm` + `B.LT label` |
| **Branch if >= (signed)** | `bge rs1, rs2, label` | `CMP Xn, Xm` + `B.GE label` |
| **Branch if < (unsigned)** | `bltu rs1, rs2, label` | `CMP Xn, Xm` + `B.LO label` |
| **Branch if >= (unsigned)** | `bgeu rs1, rs2, label` | `CMP Xn, Xm` + `B.HS label` |

**Key Difference**: RISC-V 在一個 instruction 中比較和分支。ARM 需要分別的 compare。

---

### Unconditional Jumps

| Operation | RISC-V | ARM |
|-----------|--------|-----|
| **Jump** | `jal x0, label` or `j label` | `B label` |
| **Jump and link** | `jal ra, label` | `BL label` |
| **Jump register** | `jalr x0, 0(rs1)` or `jr rs1` | `BR Xn` |
| **Jump and link register** | `jalr ra, 0(rs1)` | `BLR Xn` |
| **Return** | `jalr x0, 0(ra)` or `ret` | `RET` |

---

## E.4 Compare and Set Instructions

### Comparisons

| Operation | RISC-V | ARM |
|-----------|--------|-----|
| **Set if less than** | `slt rd, rs1, rs2` | `CMP Xn, Xm` + `CSET Xd, LT` |
| **Set if less (unsigned)** | `sltu rd, rs1, rs2` | `CMP Xn, Xm` + `CSET Xd, LO` |
| **Set if less than imm** | `slti rd, rs1, imm` | `CMP Xn, #imm` + `CSET Xd, LT` |
| **Set if less imm (uns)** | `sltiu rd, rs1, imm` | `CMP Xn, #imm` + `CSET Xd, LO` |

---

**ARM Condition Codes**：

| RISC-V | ARM Condition |
|--------|---------------|
| `beq` | `B.EQ` (equal) |
| `bne` | `B.NE` (not equal) |
| `blt` | `B.LT` (less than, signed) |
| `bge` | `B.GE` (greater or equal, signed) |
| `bltu` | `B.LO` (lower, unsigned) |
| `bgeu` | `B.HS` (higher or same, unsigned) |

---

## E.5 Atomic Instructions

### Load-Reserved / Store-Conditional

| Operation | RISC-V | ARM |
|-----------|--------|-----|
| **Load-reserved word** | `lr.w rd, (rs1)` | `LDXR Wd, [Xn]` |
| **Load-reserved dword** | `lr.d rd, (rs1)` | `LDXR Xd, [Xn]` |
| **Store-conditional word** | `sc.w rd, rs2, (rs1)` | `STXR Ws, Wd, [Xn]` |
| **Store-conditional dword** | `sc.d rd, rs2, (rs1)` | `STXR Ws, Xd, [Xn]` |

---

**Example: Atomic Increment**

RISC-V:

```assembly
retry:
    lr.w t0, (a0)
    addi t0, t0, 1
    sc.w t1, t0, (a0)
    bnez t1, retry
```

ARM:

```assembly
retry:
    LDXR W0, [X1]
    ADD W0, W0, #1
    STXR W2, W0, [X1]
    CBNZ W2, retry
```

---

### Atomic Memory Operations (AMO)

| Operation | RISC-V | ARM |
|-----------|--------|-----|
| **Atomic swap** | `amoswap.w rd, rs2, (rs1)` | `SWP Wd, Wm, [Xn]` |
| **Atomic add** | `amoadd.w rd, rs2, (rs1)` | `LDADD Ws, Wt, [Xn]` |
| **Atomic AND** | `amoand.w rd, rs2, (rs1)` | `LDCLR Ws, Wt, [Xn]` (inverted) |
| **Atomic OR** | `amoor.w rd, rs2, (rs1)` | `LDSET Ws, Wt, [Xn]` |
| **Atomic XOR** | `amoxor.w rd, rs2, (rs1)` | `LDEOR Ws, Wt, [Xn]` |
| **Atomic max (signed)** | `amomax.w rd, rs2, (rs1)` | `LDSMAX Ws, Wt, [Xn]` |
| **Atomic max (unsigned)** | `amomaxu.w rd, rs2, (rs1)` | `LDUMAX Ws, Wt, [Xn]` |
| **Atomic min (signed)** | `amomin.w rd, rs2, (rs1)` | `LDSMIN Ws, Wt, [Xn]` |
| **Atomic min (unsigned)** | `amominu.w rd, rs2, (rs1)` | `LDUMIN Ws, Wt, [Xn]` |

**Ordering Annotations**：

- RISC-V: `.aq`（acquire）、`.rl`（release）、`.aqrl`（both）
- ARM: `LDADD` vs `LDADDA` vs `LDADDL` vs `LDADDAL`

---

## E.6 Memory Barriers

| Operation | RISC-V | ARM |
|-----------|--------|-----|
| **Full fence** | `fence rw, rw` | `DMB SY` |
| **Read fence** | `fence r, r` | `DMB LD` |
| **Write fence** | `fence w, w` | `DMB ST` |
| **Acquire fence** | `fence r, rw` | `DMB LD` |
| **Release fence** | `fence rw, w` | `DMB ST` |
| **Instruction fence** | `fence.i` | `ISB` |
| **TLB fence** | `sfence.vma` | `TLBI` + `DSB` + `ISB` |

**RISC-V FENCE Format**: `fence pred, succ`

- `pred`: Predecessor operation（r=read、w=write、rw=both）
- `succ`: Successor operation（r=read、w=write、rw=both）

**ARM Barrier Types**：

- `SY`: Full system
- `ST`: Store only
- `LD`: Load only
- `ISH`: Inner shareable
- `OSH`: Outer shareable

---

## E.7 System Instructions

### CSR / System Register Access

| Operation | RISC-V | ARM |
|-----------|--------|-----|
| **Read CSR/sysreg** | `csrr rd, csr` | `MRS Xd, sysreg` |
| **Write CSR/sysreg** | `csrw csr, rs` | `MSR sysreg, Xn` |
| **Read-modify-write** | `csrrw rd, csr, rs` | `MRS` + modify + `MSR` |
| **Set bits** | `csrrs rd, csr, rs` | `MRS` + `ORR` + `MSR` |
| **Clear bits** | `csrrc rd, csr, rs` | `MRS` + `BIC` + `MSR` |

---

### Exception and Privilege

| Operation | RISC-V | ARM |
|-----------|--------|-----|
| **System call** | `ecall` | `SVC #imm` |
| **Breakpoint** | `ebreak` | `BRK #imm` |
| **Return from exception** | `mret` / `sret` | `ERET` |
| **Wait for interrupt** | `wfi` | `WFI` |
| **Supervisor call** | `ecall` (from U-mode) | `SVC #imm` |
| **Hypervisor call** | `ecall` (from VS-mode) | `HVC #imm` |

---

## E.8 Bit Manipulation (Zbb vs ARM)

| Operation | RISC-V (Zbb) | ARM |
|-----------|--------------|-----|
| **Count leading zeros** | `clz rd, rs` | `CLZ Xd, Xn` |
| **Count trailing zeros** | `ctz rd, rs` | No direct (use RBIT + CLZ) |
| **Count population** | `cpop rd, rs` | No direct (use CNT in NEON) |
| **Byte reverse** | `rev8 rd, rs` | `REV Xd, Xn` |
| **Sign-extend byte** | `sext.b rd, rs` | `SXTB Xd, Wn` |
| **Sign-extend halfword** | `sext.h rd, rs` | `SXTH Xd, Wn` |
| **Zero-extend halfword** | `zext.h rd, rs` | `UXTH Wd, Wn` |
| **Min (signed)** | `min rd, rs1, rs2` | No direct (use CMP + CSEL) |
| **Max (signed)** | `max rd, rs1, rs2` | No direct (use CMP + CSEL) |
| **Rotate right** | `ror rd, rs1, rs2` | `ROR Xd, Xn, Xm` |

---

## E.9 Calling Convention (ABI)

### Register Usage

| Purpose | RISC-V | ARM |
|---------|--------|-----|
| **Arguments** | a0-a7 (x10-x17) | X0-X7 |
| **Return value** | a0-a1 (x10-x11) | X0-X1 |
| **Saved registers** | s0-s11 (x8-x9, x18-x27) | X19-X28 |
| **Temporary registers** | t0-t6 (x5-x7, x28-x31) | X9-X15 |
| **Stack pointer** | sp (x2) | SP |
| **Frame pointer** | fp/s0 (x8) | X29 (FP) |
| **Return address** | ra (x1) | X30 (LR) |
| **Zero register** | x0 (zero) | XZR (X31) |

---

### Function Prologue/Epilogue

**RISC-V**:

```assembly
function:
    addi sp, sp, -16
    sd ra, 8(sp)
    sd s0, 0(sp)
    # ... function body ...
    ld s0, 0(sp)
    ld ra, 8(sp)
    addi sp, sp, 16
    ret
```

**ARM**:

```assembly
function:
    STP X29, X30, [SP, #-16]!
    MOV X29, SP
    # ... function body ...
    LDP X29, X30, [SP], #16
    RET
```

**Key Differences**：

- ARM 有 `STP`/`LDP`（store/load pair）用於高效的 stack operation
- RISC-V 使用分別的 `sd`/`ld` instruction
- ARM 使用 X30（LR）作為 return address；RISC-V 使用 x1（ra）

---

## E.10 Common Code Patterns

### Loop Example

**RISC-V**:

```assembly
    li t0, 0          # i = 0
    li t1, 10         # limit = 10
loop:
    # ... loop body ...
    addi t0, t0, 1    # i++
    blt t0, t1, loop  # if (i < 10) goto loop
```

**ARM**:

```assembly
    MOV X0, #0        # i = 0
    MOV X1, #10       # limit = 10
loop:
    # ... loop body ...
    ADD X0, X0, #1    # i++
    CMP X0, X1
    B.LT loop         # if (i < 10) goto loop
```

---

### Switch Statement

**RISC-V**:

```assembly
    # Assume a0 = switch value
    li t0, 3
    bgtu a0, t0, default
    slli t0, a0, 2    # t0 = a0 * 4
    la t1, jump_table
    add t0, t0, t1
    lw t0, 0(t0)
    jr t0

jump_table:
    .word case0
    .word case1
    .word case2
    .word case3
```

**ARM**:

```assembly
    # Assume X0 = switch value
    CMP X0, #3
    B.HI default
    ADR X1, jump_table
    LDR X2, [X1, X0, LSL #3]
    BR X2

jump_table:
    .quad case0
    .quad case1
    .quad case2
    .quad case3
```

---

## E.11 Porting Checklist

### Syntax Differences

| Aspect | RISC-V | ARM |
|--------|--------|-----|
| **Register prefix** | `x`, `f`, `a`, `t`, `s` | `X`, `W`, `V`, `Q` |
| **Immediate prefix** | None | `#` |
| **Memory syntax** | `offset(base)` | `[base, #offset]` |
| **Comment** | `#` | `//` or `;` |
| **Directive prefix** | `.` | `.` |

---

### Common Pitfalls

1. **Zero Register**: RISC-V `x0` vs ARM `XZR`（different encoding）
2. **Stack Pointer**: RISC-V `sp` is x2；ARM `SP` is separate
3. **Return Address**: RISC-V stores in `ra`；ARM uses `LR`（X30）
4. **Addressing Modes**: ARM has more complex modes（pre/post-indexed）
5. **Conditional Execution**: ARM has conditional instructions；RISC-V uses branches
6. **Remainder**: RISC-V has `rem`/`remu`；ARM requires division + multiply-subtract

---

### Performance Considerations

1. **Code Density**: ARM Thumb-2 vs RISC-V compressed（C extension）
2. **Instruction Fusion**: Both support micro-op fusion（implementation-dependent）
3. **Branch Prediction**: Similar capabilities（implementation-dependent）
4. **Memory Ordering**: RISC-V RVWMO is weaker than ARM（more reordering allowed）

---

## E.12 References

- **RISC-V ISA Manual**: https://riscv.org/technical/specifications/
- **ARM Architecture Reference Manual**: ARMv8-A
- **RISC-V ABI Specification**: https://github.com/riscv-non-isa/riscv-elf-psabi-doc
- **ARM Procedure Call Standard**: AAPCS64

---
