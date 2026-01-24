# 目錄

## 前言

- 封面
- 版權與授權
- 序言
- 目錄

---

## Part I: 簡介

### Chapter 1: 什麼是 RISC-V？

RISC 革命 • 為什麼選擇 RISC-V？開放 ISA 運動 • RISC-V 設計哲學 • RISC-V ISA 概覽 • RISC-V Profiles • RISC-V vs ARM vs MIPS

---

## Part II: Programmer's Model

### Chapter 2: Programmer's Model 與 Register Set

通用暫存器 • 控制與狀態暫存器（CSRs）• 程式狀態與特權層級 • Calling Convention 與 ABI • Stack Frame 結構 • 與 ARM64 和 MIPS 的比較

### Chapter 3: Privilege Level 與 Execution Environment

RISC-V Privilege 架構 • Machine Mode (M-mode) • Supervisor Mode (S-mode) • User Mode (U-mode) • 特權層級轉換 • Execution Environment Interface

---

## Part III: Trap 與 Exception

### Chapter 4: Trap, Exception, Interrupt

Trap 處理概覽 • Exception 類型 • Interrupt 類型 • Trap Delegation • Trap Vector Table • Trap 進入與退出 • 巢狀 Trap • 與 ARM 和 MIPS 的比較

---

## Part IV: 記憶體與定址

### Chapter 5: Virtual Memory 與 Paging (Sv39/Sv48)

Virtual Memory 概覽 • Page Table 結構 • 位址轉換 • TLB 管理 • Sv39 與 Sv48 模式 • 與 ARM 和 x86 的比較

### Chapter 6: Memory Ordering 與 Synchronization

Memory Consistency Model • RISC-V Memory Model (RVWMO) • Fence 指令 • Atomic 操作 • Load-Reserved/Store-Conditional • 與 ARM 和 x86 的比較

---

## Part V: Pipeline 與 Microarchitecture

### Chapter 7: RISC-V Pipeline 基礎

經典五階段 Pipeline • Pipeline Hazard • Hazard 偵測與解決 • Branch Prediction • Pipeline 效能 • 與 ARM 和 MIPS 的比較

### Chapter 8: Microarchitecture 變化

In-Order vs Out-of-Order • Superscalar 執行 • Cache 階層 • 記憶體子系統 • RISC-V Core 範例（Rocket、BOOM）• 效能比較

---

## Part VI: 開機與系統軟體

### Chapter 9: Reset, Boot Flow 與 Firmware

Reset 與初始化 • Boot ROM • Bootloader 階段 • Firmware 元件 • Device Tree • Boot Flow 範例 • 與 ARM 的比較

### Chapter 10: Machine Mode, SBI 與 Supervisor Mode

Machine Mode 概覽 • Supervisor Binary Interface (SBI) • SBI 實作（OpenSBI）• Supervisor Mode 軟體 • OS Kernel 整合 • 與 ARM 的比較

---

## Part VII: ISA Extension

### Chapter 11: RISC-V Standard Extension

Extension 命名慣例 • M Extension（乘除法）• A Extension（Atomics）• F/D Extension（浮點）• C Extension（壓縮）• Zicsr 與 Zifencei • 其他 Standard Extension

### Chapter 12: Vector Processing 與 SIMD 比較

Vector Extension 概覽 • Vector Register File • Vector 指令 • Vector Length Agnostic Programming • Vector Memory 操作 • 與 ARM SVE/SVE2 和 x86 AVX 的比較

---

## Part VIII: Platform 與 SoC

### Chapter 13: SoC 整合

SoC 架構概覽 • Interrupt Controller（PLIC、CLIC、AIA）• Memory-Mapped I/O • DMA 與 Bus Protocol • 電源管理整合

### Chapter 14: RISC-V Platform Profiles 與嵌入式系統

Platform 規格 • RVA Profiles（RVA22、RVA23）• Embedded Profiles • 認證與合規

---

## Part IX: Debug 與效能

### Chapter 15: Debugging 與 Trace

Debug 架構概覽 • Debug Module • Trigger Module • Trace 架構 • JTAG 與 OpenOCD • GDB 整合

### Chapter 16: Performance Counters 與 PMU

效能監控概覽 • 硬體效能計數器 • Event 選擇 • PMU 程式設計 • 效能分析工具

---

## Part X: 比較

### Chapter 17: RISC-V vs ARM vs MIPS — 系統性比較

歷史背景 • ISA 設計哲學 • 指令編碼 • Privilege 架構 • Memory Model • 生態系統與採用 • 未來展望

---

## 附錄

**附錄 A**: CSR 參考
**附錄 B**: Extension 參考
**附錄 C**: Bootloader 參考
**附錄 D**: SBI 參考
**附錄 E**: RISC-V vs ARM 比較
**附錄 F**: Memory Model 參考

---

## 後記

- 關於作者
- 參考文獻
