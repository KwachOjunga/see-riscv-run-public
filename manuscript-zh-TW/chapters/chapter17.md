# Chapter 17. RISC-V vs ARM vs MIPS — A Systematic Comparison

**Part X — RISC-V vs Other Architectures**

---

Architecture 選擇塑造一切。Instruction set 決定 software 如何表達計算、hardware 如何實現執行，以及 ecosystem 如何圍繞 platform 發展。RISC-V 進入一個由 ARM 主導 mobile 和 embedded system、歷史上受 MIPS 影響的 education 和 networking 的環境。理解這些 architecture 如何比較，揭示了 RISC-V 的設計決策、trade-off 和競爭地位。

本章提供 RISC-V、ARM 和 MIPS 在十一個維度的系統性比較：ISA 設計哲學、instruction set complexity、register architecture、exception 和 interrupt model、memory model、virtual memory、interrupt architecture、calling convention、pipeline 和 microarchitecture、ecosystem 和 licensing，以及 future direction。每個章節檢視 architecture 如何處理相同問題，突出相似性、差異性，以及對 software 和 hardware 的影響。

RISC-V 代表現代、模組化、開放的方法。ARM 代表全面、演進、商業的方法。MIPS 代表經典 RISC 簡潔性和歷史商業根源。它們共同說明了 architectural design choice 的範圍。

---

## 17.1 ISA Design Philosophy

**RISC-V: Modular and Extensible**

RISC-V 的哲學強調 modularity 和 extensibility：

- **Base ISA**：Minimal、frozen foundation（RV32I、RV64I、RV128I）
- **Standard extension**：Optional、composable module（M、A、F、D、C、V 等）
- **Custom extension**：Vendor-specific addition，不造成 ISA fragmentation
- **Clean slate**：沒有 legacy baggage，從 first principle 設計

這種 modular approach 允許：

- Tiny microcontroller（僅 RV32I）
- Application processor（RV64IMAFDCV）
- Custom accelerator（base + custom extension）

**ARM: Comprehensive and Evolving**

ARM 的哲學強調 comprehensiveness 和 evolution：

- **Comprehensive ISA**：Rich instruction set，涵蓋許多 use case
- **Profile**：不同 ISA subset（A-profile、R-profile、M-profile）
- **Backward compatibility**：新版本擴展，很少移除 feature
- **Market-driven**：根據 market need 添加 feature

ARM profile：

- **A-profile**：Application processor（ARMv8-A、ARMv9-A）
- **R-profile**：Real-time processor（ARMv8-R）
- **M-profile**：Microcontroller（ARMv8-M、ARMv9-M）

**MIPS: Classic RISC Simplicity**

MIPS 的哲學強調經典 RISC principle：

- **Simple, regular ISA**：Load-store architecture、fixed-length instruction
- **Delayed branch**：將 pipeline 暴露給 software（MIPS I-IV）
- **Coprocessor model**：Floating-point 和其他 function 作為 coprocessor
- **Minimal complexity**：保持 hardware 簡單，讓 software 處理 complexity

MIPS 影響了 RISC-V 的設計，但 RISC-V 現代化了許多方面（沒有 delayed branch、更乾淨的 privilege model）。

**Design Trade-offs**

| Aspect | RISC-V | ARM | MIPS |
|--------|--------|-----|------|
| **Modularity** | High（base + extension） | Medium（profile） | Low（monolithic） |
| **Extensibility** | High（custom extension） | Low（vendor-specific） | Low（proprietary） |
| **Backward Compatibility** | High（frozen base） | High（evolutionary） | Medium（version） |
| **Complexity** | Low to medium | Medium to high | Low |
| **Flexibility** | High | Medium | Low |

---

## 17.2 Instruction Set Complexity

**Instruction Count Comparison**

近似 instruction count：

| Architecture | Base Instructions | With Extensions | Total (typical) |
|--------------|-------------------|-----------------|-----------------|
| **RISC-V RV32I** | 47 | +M(8), +A(11), +F(26), +D(26), +C(~40) | ~150-200 |
| **RISC-V RV64I** | 59 | +M(8), +A(11), +F(26), +D(26), +C(~40) | ~170-220 |
| **ARM ARMv8-A** | ~500 base | +NEON, +SVE, +crypto | ~1000+ |
| **MIPS32** | ~100 base | +FPU, +DSP | ~200-300 |

RISC-V 擁有最小的 base ISA。ARM 擁有最大的 instruction set。MIPS 介於兩者之間。

**Encoding Formats**

*RISC-V*：6 個 base format（R、I、S、B、U、J）+ compressed（CR、CI、CSS、CIW、CL、CS、CA、CB、CJ）

```
R-type: [funct7|rs2|rs1|funct3|rd|opcode]
I-type: [imm[11:0]|rs1|funct3|rd|opcode]
S-type: [imm[11:5]|rs2|rs1|funct3|imm[4:0]|opcode]
```

*ARM*：Multiple format（data processing、load/store、branch 等）

```
Data processing: [cond|00|I|opcode|S|Rn|Rd|operand2]
Load/store:      [cond|01|I|P|U|B|W|L|Rn|Rd|offset]
```

*MIPS*：3 個 format（R、I、J）

```
R-type: [opcode|rs|rt|rd|shamt|funct]
I-type: [opcode|rs|rt|immediate]
J-type: [opcode|address]
```

RISC-V 和 MIPS 擁有比 ARM 更簡單、更規則的 encoding。

**Addressing Modes**

*RISC-V*：

- Register：`add rd, rs1, rs2`
- Immediate：`addi rd, rs1, imm`
- Base+offset：`lw rd, offset(rs1)`

*ARM*：

- Register：`ADD Rd, Rn, Rm`
- Immediate：`ADD Rd, Rn, #imm`
- Base+offset：`LDR Rd, [Rn, #offset]`
- Base+register：`LDR Rd, [Rn, Rm]`
- Pre/post-indexed：`LDR Rd, [Rn, #offset]!` 或 `LDR Rd, [Rn], #offset`

*MIPS*：

- Register：`add $rd, $rs, $rt`
- Immediate：`addi $rt, $rs, imm`
- Base+offset：`lw $rt, offset($rs)`

ARM 擁有最多 addressing mode。RISC-V 和 MIPS 更簡單。

**ISA Complexity Metrics**

| Metric | RISC-V | ARM | MIPS |
|--------|--------|-----|------|
| **Instruction format** | 6 base + 9 compressed | 10+ | 3 |
| **Addressing mode** | 3 | 10+ | 3 |
| **Conditional execution** | Branch only | Most instruction（ARMv7）、limited（ARMv8） | Branch only |
| **Instruction length** | 32-bit（16-bit compressed） | 32-bit（16-bit Thumb） | 32-bit |
| **Regularity** | High | Medium | High |

RISC-V 和 MIPS 優先考慮 regularity。ARM 優先考慮 expressiveness。

---

## 17.3 Register Architecture

**RISC-V: x0-x31 (x0 = zero)**

RISC-V 提供 32 個 general-purpose register：

- `x0`：Hardwired zero（讀取為 0，寫入被忽略）
- `x1-x31`：General-purpose register
- `f0-f31`：Floating-point register（F/D extension）

Special convention：

- `x1`（ra）：Return address
- `x2`（sp）：Stack pointer
- `x8`（s0/fp）：Frame pointer

**ARM: X0-X30 + XZR + SP**

ARM ARMv8-A 提供 31 個 general-purpose register + special register：

- `X0-X30`：General-purpose register（64-bit）
- `W0-W30`：X0-X30 的 32-bit view
- `XZR`（X31）：Zero register（讀取為 0，寫入被忽略）
- `SP`：Stack pointer（與 general register 分離）
- `PC`：Program counter（ARMv8 中不可直接訪問）

ARM 有 31 個 general register，而 RISC-V 有 32 個（包括 zero）。

**MIPS: $0-$31 ($0 = zero)**

MIPS 提供 32 個 general-purpose register：

- `$0`（$zero）：Hardwired zero
- `$1-$31`：General-purpose register

Special convention：

- `$31`（$ra）：Return address
- `$29`（$sp）：Stack pointer
- `$30`（$fp）：Frame pointer

MIPS 和 RISC-V 擁有相同的 register count 和 zero register concept。

**Special-Purpose Registers**

*RISC-V*：CSR（Control and Status Register）

- 通過 `csrr`、`csrw`、`csrrw` 等訪問
- 範例：`mstatus`、`mtvec`、`mepc`、`mcause`

*ARM*：System register

- 通過 `MRS`、`MSR` instruction 訪問
- 範例：`SCTLR_EL1`、`VBAR_EL1`、`ELR_EL1`、`ESR_EL1`

*MIPS*：Coprocessor 0 register

- 通過 `mfc0`、`mtc0` instruction 訪問
- 範例：`Status`、`Cause`、`EPC`、`BadVAddr`

所有三種 architecture 都為 system register 使用獨立的 namespace。

---

## 17.4 Exception and Interrupt Models

**RISC-V: Trap Model (M/S/U Modes)**

RISC-V 使用統一的 trap model：

- **Privilege mode**：M（Machine）、S（Supervisor）、U（User）
- **Trap type**：Exception（synchronous）、Interrupt（asynchronous）
- **Trap vector**：`mtvec`（M-mode）、`stvec`（S-mode）
- **Trap cause**：`mcause`（M-mode）、`scause`（S-mode）
- **Trap PC**：`mepc`（M-mode）、`sepc`（S-mode）

Trap handling：

```assembly
# M-mode trap handler
trap_handler:
    csrr t0, mcause      # 讀取 cause
    csrr t1, mepc        # 讀取 PC
    # 處理 trap
    mret                 # 從 trap 返回
```

**ARM: Exception Levels (EL0-EL3)**

ARM 使用 exception level：

- **Exception level**：EL0（User）、EL1（OS）、EL2（Hypervisor）、EL3（Secure Monitor）
- **Exception type**：Synchronous、IRQ、FIQ、SError
- **Exception vector**：`VBAR_EL1`、`VBAR_EL2`、`VBAR_EL3`
- **Exception syndrome**：`ESR_EL1`、`ESR_EL2`、`ESR_EL3`
- **Exception link**：`ELR_EL1`、`ELR_EL2`、`ELR_EL3`

Exception handling：

```assembly
# EL1 exception handler
exception_handler:
    mrs x0, ESR_EL1      # 讀取 syndrome
    mrs x1, ELR_EL1      # 讀取 PC
    # 處理 exception
    eret                 # 從 exception 返回
```

**MIPS: Exception Handling**

MIPS 使用更簡單的 exception model：

- **Mode**：User、Kernel
- **Exception vector**：固定在 0x80000180（general）或 0x80000000（reset/NMI）
- **Cause register**：編碼 exception type
- **EPC**：Exception PC

Exception handling：

```assembly
# MIPS exception handler
exception_handler:
    mfc0 k0, Cause       # 讀取 cause
    mfc0 k1, EPC         # 讀取 PC
    # 處理 exception
    eret                 # 從 exception 返回
```

**Comparison Table**

| Feature | RISC-V | ARM | MIPS |
|---------|--------|-----|------|
| **Privilege Level** | 3（M/S/U）+ optional H | 4（EL0-EL3） | 2（User/Kernel） |
| **Trap/Exception Type** | Unified（trap） | 4 type（Sync、IRQ、FIQ、SError） | Unified（exception） |
| **Vector Table** | mtvec、stvec | VBAR_ELn | Fixed address |
| **Cause Register** | mcause、scause | ESR_ELn | Cause |
| **Return Instruction** | mret、sret | eret | eret |
| **Nested Interrupt** | Software-managed | Hardware-managed | Software-managed |

ARM 擁有最複雜的 exception model。RISC-V 是模組化的。MIPS 最簡單。

---

## 17.5 Memory Models

**RISC-V: RVWMO (Weak Memory Ordering)**

RISC-V 使用 weak memory model（RVWMO）：

- **Ordering**：預設 relaxed
- **Fence**：`fence` instruction 用於 ordering
- **Atomic**：LR/SC 和 AMO instruction
- **TSO extension**：Optional total store ordering（Ztso）

Memory ordering：

```assembly
# RISC-V: 確保 store 在 load 之前可見
sw x10, 0(x5)
fence w, r           # Write-to-read fence
lw x11, 0(x6)
```

**ARM: Weak Memory Model**

ARM 使用 weak memory model：

- **Ordering**：預設 relaxed
- **Barrier**：DMB（data memory barrier）、DSB（data synchronization barrier）、ISB（instruction synchronization barrier）
- **Atomic**：LDXR/STXR（exclusive）和 atomic instruction（ARMv8.1+）

Memory ordering：

```assembly
# ARM: 確保 store 在 load 之前可見
STR X0, [X1]
DMB SY               # Data memory barrier（system）
LDR X2, [X3]
```

**MIPS: Sequential Consistency Variants**

MIPS 傳統上使用 sequential consistency，但現代 MIPS 支援 weak ordering：

- **Ordering**：Sequential consistency（MIPS I-III）、weak ordering（MIPS IV+）
- **Barrier**：`sync` instruction
- **Atomic**：LL/SC（load-linked/store-conditional）

Memory ordering：

```assembly
# MIPS: 確保 store 在 load 之前可見
sw $t0, 0($a0)
sync                 # Synchronization barrier
lw $t1, 0($a1)
```

**Memory Barrier Instructions**

| Architecture | Barrier Instruction | Purpose |
|--------------|---------------------|---------|
| **RISC-V** | `fence r, w` | Order read before write |
| **RISC-V** | `fence w, r` | Order write before read |
| **RISC-V** | `fence rw, rw` | Full fence |
| **ARM** | `DMB` | Data memory barrier |
| **ARM** | `DSB` | Data synchronization barrier |
| **ARM** | `ISB` | Instruction synchronization barrier |
| **MIPS** | `sync` | Synchronization barrier |

RISC-V 的 fence 更細粒度（指定 predecessor/successor）。ARM 有多種 barrier type。MIPS 有單一 sync instruction。

---

## 17.6 Virtual Memory

**RISC-V: Sv39/Sv48 Page Tables**

RISC-V virtual memory：

- **Sv39**：39-bit virtual address、3-level page table
- **Sv48**：48-bit virtual address、4-level page table
- **Sv57**：57-bit virtual address、5-level page table（future）
- **Page size**：4 KB、2 MB（megapage）、1 GB（gigapage）
- **TLB**：Implementation-specific

Page table entry（Sv39）：

```
[63:54] Reserved
[53:28] PPN[2]（26 bit）
[27:19] PPN[1]（9 bit）
[18:10] PPN[0]（9 bit）
[9:0]   Flag（V、R、W、X、U、G、A、D）
```

**ARM: 48-bit VA with TTBR0/1**

ARM virtual memory（ARMv8-A）：

- **VA size**：48-bit（可配置 36-48 bit）
- **Page table level**：4 level（可配置）
- **Page size**：4 KB、2 MB、1 GB
- **TTBR0/TTBR1**：User/kernel 的獨立 page table

Page table entry（4 KB granule）：

```
[63:48] Ignored/SW use
[47:12] Output address
[11:2]  Attribute（AF、SH、AP、NS 等）
[1]     Table/Block
[0]     Valid
```

**MIPS: TLB-Based MMU**

MIPS virtual memory：

- **TLB**：Software-managed TLB（沒有 hardware page table walk）
- **VA size**：32-bit（MIPS32）、64-bit（MIPS64）
- **Page size**：4 KB（typical）、可配置
- **TLB entry**：16-64 entry（implementation-specific）

TLB entry：

```
EntryHi:  [VPN | ASID]
EntryLo0: [PFN | C | D | V | G]
EntryLo1: [PFN | C | D | V | G]
PageMask: Page size
```

**Page Table Walk Comparison**

| Feature | RISC-V | ARM | MIPS |
|---------|--------|-----|------|
| **Hardware Walk** | Yes | Yes | No（software TLB refill） |
| **Page Table Level** | 3-5（Sv39-Sv57） | 4（可配置） | N/A（TLB only） |
| **Page Size** | 4 KB、2 MB、1 GB | 4 KB、2 MB、1 GB | 4 KB（可配置） |
| **TLB Refill** | Hardware | Hardware | Software（exception） |
| **ASID** | Yes（satp.ASID） | Yes（TTBR.ASID） | Yes（EntryHi.ASID） |

RISC-V 和 ARM 使用 hardware page table walk。MIPS 使用 software TLB refill，提供更多靈活性但需要 software overhead。

---

## 17.7 Interrupt Architecture

**RISC-V: PLIC/CLIC/AIA**

RISC-V interrupt architecture 已經演進：

- **CLINT**：Core-Local Interruptor（timer、software interrupt）
- **PLIC**：Platform-Level Interrupt Controller（external interrupt）
- **CLIC**：Core-Local Interrupt Controller（vectored、nested interrupt for embedded）
- **AIA**：Advanced Interrupt Architecture（MSI、IMSIC for server）

PLIC 範例：

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

ARM 使用 Generic Interrupt Controller（GIC）：

- **GICv2**：Legacy，支援最多 8 個 core
- **GICv3**：Modern，支援許多 core，message-based
- **GICv4**：添加 virtualization support
- **Interrupt type**：SGI（software）、PPI（private peripheral）、SPI（shared peripheral）

GIC 範例：

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

MIPS 使用簡單的 interrupt model：

- **Interrupt line**：8 個 hardware interrupt line（IP0-IP7）
- **Interrupt mask**：由 Status register 控制
- **Interrupt pending**：由 Cause register 指示
- **External controller**：Optional external interrupt controller

MIPS 範例：

```c
// MIPS interrupt handler
void mips_interrupt_handler(void) {
    uint32_t cause = read_c0_cause();
    uint32_t pending = (cause >> 8) & 0xFF;  // IP bit

    if (pending & (1 << 2)) {  // IP2
        uart_handler();
    }
}
```

**Interrupt Routing and Priority**

| Feature | RISC-V（PLIC） | ARM（GIC） | MIPS |
|---------|---------------|-----------|------|
| **Interrupt Source** | 1-1023 | 32-1020 | 8 line |
| **Priority Level** | 0-255 | 0-255 | None（software） |
| **Routing** | Per-hart enable | Affinity routing | Fixed |
| **Vectoring** | Optional（CLIC） | Yes | No |
| **Nesting** | Software-managed | Hardware-managed | Software-managed |

ARM GIC 最複雜。RISC-V PLIC 靈活。MIPS 最簡單。

---

## 17.8 Calling Conventions

**RISC-V: RV64 SysV ABI**

RISC-V calling convention（RV64）：

- **Argument**：a0-a7（x10-x17）
- **Return value**：a0-a1（x10-x11）
- **Saved register**：s0-s11（x8-x9、x18-x27）
- **Temporary register**：t0-t6（x5-x7、x28-x31）
- **Stack pointer**：sp（x2）
- **Return address**：ra（x1）

Function call：

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

ARM calling convention（AAPCS64）：

- **Argument**：X0-X7
- **Return value**：X0-X1
- **Saved register**：X19-X28
- **Temporary register**：X9-X15
- **Stack pointer**：SP
- **Return address**：X30（LR）
- **Frame pointer**：X29（FP）

Function call：

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

MIPS 有多個 ABI：

- **O32**：32-bit、4 個 argument register
- **N32**：32-bit pointer、64-bit register
- **N64**：64-bit

MIPS O32 calling convention：

- **Argument**：$a0-$a3（$4-$7）
- **Return value**：$v0-$v1（$2-$3）
- **Saved register**：$s0-$s7（$16-$23）
- **Temporary register**：$t0-$t9（$8-$15、$24-$25）
- **Stack pointer**：$sp（$29）
- **Return address**：$ra（$31）

Function call：

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

| Feature | RISC-V | ARM | MIPS（O32） |
|---------|--------|-----|------------|
| **Argument Register** | 8（a0-a7） | 8（X0-X7） | 4（$a0-$a3） |
| **Return Register** | 2（a0-a1） | 2（X0-X1） | 2（$v0-$v1） |
| **Saved Register** | 12（s0-s11） | 10（X19-X28） | 8（$s0-$s7） |
| **Temporary Register** | 7（t0-t6） | 7（X9-X15） | 10（$t0-$t9） |
| **Stack Growth** | Downward | Downward | Downward |
| **Alignment** | 16 byte | 16 byte | 8 byte（O32） |

RISC-V 和 ARM 比 MIPS O32 有更多 argument register，減少 stack usage。

---

## 17.9 Pipeline and Microarchitecture

**In-Order vs Out-of-Order**

所有三種 architecture 都支援 in-order 和 out-of-order implementation：

*RISC-V*：

- In-order：SiFive E-series、Rocket
- Out-of-order：SiFive P-series、BOOM（Berkeley Out-of-Order Machine）

*ARM*：

- In-order：Cortex-A5、Cortex-A7、Cortex-A53
- Out-of-order：Cortex-A72、Cortex-A76、Neoverse N1/V1

*MIPS*：

- In-order：MIPS 24K、34K
- Out-of-order：MIPS 74K、R10000

**Branch Prediction Strategies**

Modern implementation 使用複雜的 branch prediction：

| Implementation | Branch Predictor | Accuracy |
|----------------|------------------|----------|
| **SiFive U74** | 2-level adaptive | ~90% |
| **SiFive P550** | TAGE predictor | ~95% |
| **ARM Cortex-A76** | Multi-level predictor | ~95%+ |
| **MIPS 74K** | 2-level adaptive | ~90% |

Out-of-order core 需要更好的 branch prediction 來維持 performance。

**Cache Hierarchies**

Typical cache configuration：

*RISC-V（SiFive P550）*：

- L1 I-cache：32 KB、4-way
- L1 D-cache：32 KB、8-way
- L2 cache：512 KB - 2 MB、8-way

*ARM（Cortex-A76）*：

- L1 I-cache：64 KB、4-way
- L1 D-cache：64 KB、4-way
- L2 cache：256 KB - 512 KB、8-way

*MIPS（74K）*：

- L1 I-cache：32 KB、4-way
- L1 D-cache：32 KB、4-way
- L2 cache：Optional、implementation-specific

**Implementation Examples**

| Implementation | Type | Pipeline Stage | Issue Width | IPC（peak） |
|----------------|------|-----------------|-------------|------------|
| **SiFive E76** | In-order | 8 | 1 | 1.0 |
| **SiFive P550** | Out-of-order | 13 | 3 | 3.0 |
| **ARM Cortex-A53** | In-order | 8 | 2 | 2.0 |
| **ARM Cortex-A76** | Out-of-order | 13 | 4 | 4.0 |
| **MIPS 24K** | In-order | 8 | 1 | 1.0 |
| **MIPS 74K** | Out-of-order | 15 | 2 | 2.0 |

ARM 擁有最積極的 out-of-order implementation。RISC-V 正在趕上。MIPS 開發已經放緩。

---

## 17.10 Ecosystem and Licensing

**RISC-V: Open and Free ISA**

RISC-V ecosystem：

- **ISA**：Open、royalty-free
- **Licensing**：沒有 licensing fee
- **Governance**：RISC-V International（non-profit）
- **Implementation**：Open-source（Rocket、BOOM）和 commercial（SiFive、Andes 等）
- **Software**：GCC、LLVM、Linux、FreeBSD、Zephyr、FreeRTOS

Advantage：

- 沒有 licensing cost
- Customizable（添加 custom extension）
- Growing ecosystem
- Academic 和 research-friendly

Challenge：

- Younger ecosystem（比 ARM 不成熟）
- 較少 commercial implementation
- Software ecosystem 仍在發展

**ARM: Commercial Licensing**

ARM ecosystem：

- **ISA**：Proprietary
- **Licensing**：Architecture license（設計自己的 core）或 implementation license（使用 ARM core）
- **Governance**：ARM Holdings（commercial company）
- **Implementation**：ARM Cortex series、vendor design（Apple、Qualcomm、Samsung）
- **Software**：Mature ecosystem（GCC、LLVM、Linux、Android、iOS）

Advantage：

- Mature、proven ecosystem
- Extensive software support
- Wide industry adoption
- Strong performance

Challenge：

- Licensing fee
- 較少 customizable
- Vendor lock-in

**MIPS: Historical Commercial, Now Open**

MIPS ecosystem：

- **ISA**：現在 open（MIPS Open initiative、2018）
- **Licensing**：歷史上 commercial，現在 open
- **Governance**：MIPS Open（Wave Computing）
- **Implementation**：Historical（MIPS Technologies），現在有限
- **Software**：GCC、LLVM、Linux（legacy support）

Status：

- Historical importance（education、networking）
- Declining commercial adoption
- Open initiative 來得太晚
- 主要被 ARM 和 RISC-V 取代

**Industry Adoption and Ecosystem**

| Aspect | RISC-V | ARM | MIPS |
|--------|--------|-----|------|
| **Market Share** | Growing（embedded、IoT） | Dominant（mobile、embedded） | Declining |
| **Vendor** | SiFive、Andes、Alibaba 等 | ARM、Apple、Qualcomm 等 | Limited |
| **Software Ecosystem** | Growing | Mature | Legacy |
| **Tool Support** | Good（GCC、LLVM） | Excellent | Good（legacy） |
| **Community** | Active、growing | Mature、large | Small |
| **Education** | Increasing | Common | Historical |

ARM 在商業上占主導地位。RISC-V 快速增長。MIPS 正在衰退。

---

## 17.11 Future Directions

**RISC-V Roadmap**

RISC-V 正在積極演進：

- **Ratified extension**：V（vector）、B（bit manipulation）、Zicond（conditional op）
- **In progress**：J（JIT support）、P（packed SIMD）、Zc（code size reduction）
- **Proposed**：Crypto extension、memory tagging、capability
- **Platform profile**：RVA22、RVA23（標準化 extension combination）

Focus area：

- Performance（vector、out-of-order）
- Security（crypto、memory tagging）
- Code density（compressed、Zc）
- Ecosystem maturity（tool、software）

**ARM Roadmap**

ARM 繼續演進：

- **ARMv9-A**：SVE2（scalable vector）、TME（transactional memory）、MTE（memory tagging）
- **Confidential Compute**：Realm Management Extension（RME）
- **AI/ML**：Matrix extension、neural processing
- **Automotive**：ASIL-D safety、real-time

Focus area：

- AI/ML acceleration
- Security（confidential computing）
- Automotive 和 safety
- Performance scaling

**Emerging Extensions**

*RISC-V*：

- **Crypto**：AES、SHA、scalar crypto
- **Vector**：V extension（ratified）、improvement ongoing
- **Hypervisor**：H extension（ratified）
- **Bit manipulation**：B extension（ratified）

*ARM*：

- **SVE2**：Scalable vector（NEON 的 successor）
- **MTE**：Memory Tagging Extension（security）
- **TME**：Transactional Memory Extension
- **SME**：Scalable Matrix Extension（AI/ML）

**Security Features**

兩種 architecture 都在添加 security feature：

*RISC-V*：

- PMP（Physical Memory Protection）
- sPMP（Supervisor PMP）
- Crypto extension
- Proposed：Memory tagging、capability（CHERI-like）

*ARM*：

- TrustZone（secure world）
- MTE（Memory Tagging Extension）
- PAC（Pointer Authentication Code）
- BTI（Branch Target Identification）

**Industry Trends**

*RISC-V trend*：

- 在中國快速採用（Alibaba、Huawei）
- 在 IoT 和 embedded 增長
- 在 data center 增加（SiFive、Ventana）
- 在 research 和 education 強勁

*ARM trend*：

- 在 mobile 繼續主導
- 在 data center 增長（AWS Graviton、Ampere）
- 在 automotive 強勁
- 在 AI/ML 擴展

*MIPS trend*：

- Market share 下降
- 僅 legacy support
- 一些 niche application（networking）

未來有利於 RISC-V（open、growing）和 ARM（mature、dominant）。MIPS 主要是歷史性的。

---

## Summary

比較 RISC-V、ARM 和 MIPS 揭示了不同的 architectural philosophy 和 trade-off。本章檢視了十一個比較維度，展示每種 architecture 如何處理相同問題。

**ISA design philosophy** 從根本上區分了三種 architecture。RISC-V 強調 modularity 和 extensibility，具有 frozen base ISA 和 composable extension。ARM 強調 comprehensiveness 和 evolution，具有 rich instruction set 和 market-driven feature。MIPS 強調經典 RISC simplicity，具有 regular encoding 和 minimal complexity。這些哲學塑造了所有後續設計決策。

**Instruction set complexity** 差異顯著。RISC-V 擁有最小的 base ISA（RV32I 47 個 instruction），具有 optional extension。ARM 擁有最大的 instruction set（1000+ instruction），涵蓋許多 use case。MIPS 介於兩者之間，具有經典 RISC simplicity。Encoding format 反映了這一點 — RISC-V 和 MIPS 使用 regular format，而 ARM 使用更複雜的 encoding 以獲得 expressiveness。

**Register architecture** 顯示趨同。所有三種都提供 31-32 個 general-purpose register，具有 hardwired zero register（RISC-V x0、ARM XZR、MIPS $0）。ARM 將 stack pointer 與 general register 分離。所有三種都為 system register 使用獨立的 namespace，通過 special instruction 訪問。

**Exception and interrupt model** 反映了不同的 privilege architecture。RISC-V 使用統一的 trap model，具有 M/S/U mode。ARM 使用 exception level（EL0-EL3），具有四種 exception type。MIPS 使用更簡單的 two-mode model。ARM 提供最複雜的 exception handling，而 MIPS 最簡單。

**Memory model** 都使用 weak ordering 以獲得 performance。RISC-V 使用 RVWMO，具有細粒度 fence instruction。ARM 使用 weak model，具有多種 barrier type（DMB、DSB、ISB）。MIPS 從 sequential consistency 演進到 weak ordering，具有 sync barrier。所有三種都提供 atomic instruction 用於 synchronization。

**Virtual memory** 顯示 architectural maturity。RISC-V 和 ARM 使用 hardware page table walk，具有 multi-level page table（RISC-V 的 Sv39/Sv48、ARM 的 4-level）。MIPS 使用 software-managed TLB refill，以 software overhead 為代價提供靈活性。所有三種都支援多種 page size 和 ASID，用於高效 context switching。

**Interrupt architecture** 從簡單到複雜。RISC-V 使用 PLIC 用於 platform interrupt，CLIC 用於 embedded，AIA 用於 server。ARM 使用 GIC（Generic Interrupt Controller），具有成熟的 multi-core support。MIPS 使用簡單的 8-line interrupt model。ARM GIC 最成熟，RISC-V 正在演進，MIPS 最簡單。

**Calling convention** 顯示實際相似性。所有三種都使用類似的 register allocation strategy，具有 4-8 個 argument register、2 個 return register，以及 saved/temporary register set。RISC-V 和 ARM 提供 8 個 argument register，而 MIPS O32 僅提供 4 個。所有都使用 downward-growing stack，具有 8-16 byte alignment。

**Pipeline and microarchitecture** 展示 implementation diversity。所有三種 architecture 都支援 in-order 和 out-of-order implementation。ARM 擁有最積極的 out-of-order core（Cortex-A76、Neoverse）。RISC-V 正在趕上，具有競爭性設計（SiFive P550、BOOM）。MIPS 開發已經放緩。Branch prediction 和 cache hierarchy 在 modern implementation 中相似。

**Ecosystem and licensing** 代表根本的 business difference。RISC-V 是 open 和 royalty-free，實現 customization 並消除 licensing cost。ARM 是 proprietary，具有 licensing fee，但提供 mature、proven ecosystem。MIPS 曾經是 commercial，但 open 得太晚，現在正在衰退。ARM 在商業上占主導地位，RISC-V 快速增長，MIPS 是歷史性的。

**Future direction** 顯示積極演進。RISC-V 正在添加 vector、crypto 和 security extension，同時標準化 platform profile。ARM 正在推進 AI/ML、security（MTE）和 confidential computing。兩者都在添加 memory tagging 和增強的 security feature。Industry trend 有利於 RISC-V（open、growing）和 ARM（mature、dominant），而 MIPS 仍然主要是歷史性的。

總之，這些比較顯示 RISC-V 作為 ARM 成熟、全面、商業方法的現代、模組化、開放替代方案，MIPS 代表經典 RISC principle，現在主要被取代。選擇取決於優先級：openness 和 customization 有利於 RISC-V，ecosystem maturity 有利於 ARM。

---
