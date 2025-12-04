# Chapter 14. RISC-V Platform Profiles & Embedded Systems

**Part VIII — System Design, Platform Spec & SoC Integration**

---

RISC-V 的 modularity 既是優勢也是挑戰。Base ISA 是 minimal 的，implementation 選擇要包含哪些 extension。這種靈活性使針對特定 use case 的優化成為可能 — microcontroller 可能只包含 RV32IMC，而 server processor 可能實作 RV64IMAFDCV。但這種可變性造成了一個問題：software 如何知道有哪些 feature 可用？

Platform profile 通過為特定 use case 定義 standard extension combination 來解決這個問題。Profile 指定 mandatory extension、optional extension 和 implementation requirement。針對 profile 的 software 可以假設某些 feature 存在，簡化 development 並提高 portability。RVA22 profile 定義了運行 rich operating system 的 application processor 的 requirement。RVA23 profile 添加了更新的 extension 和更嚴格的 requirement。Embedded profile 針對 microcontroller 和 real-time system。

本章探討 RISC-V platform profile 和 embedded system design。我們將檢視 application processor 的 RVA22 和 RVA23 profile、microcontroller 的 embedded profile、MMU 和 no-MMU system 之間的差異，以及 RISC-V 在 embedded space 中與 ARM Cortex-M 的比較。

---

## 14.1 Platform Profiles Overview

**The Fragmentation Problem**

RISC-V 的 modularity 造成潛在的 fragmentation。考慮這些有效的 RISC-V implementation：

- RV32I：Minimal 32-bit core（僅 base ISA）
- RV32IMC：Embedded core（multiply、compressed）
- RV64IMAFDC：Application processor（full general-purpose）
- RV64IMAFDCV：High-performance with vector

為 RV64IMAFDC 撰寫的 software 無法在 RV32IMC 上運行。即使在相同的 base (RV64) 內，不同的 extension combination 也會造成不兼容性。這使得分發 binary software 或開發 portable operating system 變得困難。

**Profiles as a Solution**

Platform profile 定義：

- **Mandatory extension**：必須實作
- **Optional extension**：可以實作
- **Mandatory feature**：特定 implementation requirement（例如，最小 PMP entry 數量）
- **Discovery mechanism**：Software 如何檢測 feature

Profile 為 software development 創建 standard target。OS 可以針對 "RVA22 profile" 並確切知道有哪些 feature 可用。Hardware vendor 可以聲稱 "RVA22 compliant" 並保證與 RVA22 software 的兼容性。

**Profile Naming Convention**

RISC-V profile 使用命名方案：

- **RV**：RISC-V
- **A/M/E**：Application / Microcontroller / Embedded
- **22/23/...**：Ratification 年份（2022、2023 等）
- **S/U**：Supervisor mode / User mode（對於 application profile）

範例：

- **RVA22S**：Application profile，2022，Supervisor mode
- **RVA22U**：Application profile，2022，User mode
- **RVA23S**：Application profile，2023，Supervisor mode
- **RVM23**：Microcontroller profile，2023

**Profile Versioning**

Profile 隨時間演進：

- **Major version**：不兼容的變更（例如，RVA22 → RVA23）
- **Minor version**：向後兼容的添加
- **Errata**：Bug fix，無功能變更

Profile version 保證：

- **Forward compatibility**：RVA22 software 在 RVA23 hardware 上運行
- **Feature stability**：Mandatory extension 在 version 內不會改變

---

## 14.2 RVA22 Profile (Application Processor)

**Target Use Case**

RVA22 針對運行 rich operating system（如 Linux、FreeBSD 或 commercial RTOS）的 application processor。這些系統需要：

- Virtual memory (MMU)
- Privilege level（M、S、U mode）
- General-purpose computing 的 standard extension
- Application workload 的足夠 performance

**RVA22S (Supervisor Mode)**

RVA22S 定義運行 supervisor-mode operating system 的系統的 requirement。

**Mandatory ISA Extensions**：

- **RV64I**：64-bit base integer ISA
- **M**：Integer multiplication and division
- **A**：Atomic instruction
- **F**：Single-precision floating-point
- **D**：Double-precision floating-point
- **C**：Compressed instruction
- **Zicsr**：CSR instruction
- **Zifencei**：Instruction fence
- **Zicntr**：Base counter（cycle、time、instret）
- **Zihpm**：Hardware performance counter
- **Ziccif**：Main memory supports instruction fetch
- **Ziccrse**：Main memory supports misaligned load/store
- **Ziccamoa**：Main memory supports all atomic
- **Zicclsm**：Main memory supports misaligned atomic
- **Za64rs**：Reservation set size（64 byte）
- **Zihintpause**：Pause hint instruction
- **Zba**：Address generation（bit manipulation）
- **Zbb**：Basic bit manipulation
- **Zbs**：Single-bit instruction
- **Zkt**：Data-independent execution time（timing side-channel protection）

**Mandatory Privileged Features**：

- **Sv39**：Page-based virtual memory（39-bit virtual address）
- **Svpbmt**：Page-based memory type
- **Svadu**：Hardware A/D bit update
- **Sstc**：Supervisor-mode timer interrupt
- **Sscofpmf**：Count overflow and privilege mode filtering

**Mandatory Implementation Requirements**：

- 至少 8 個 PMP entry
- 至少 29 個 hardware performance counter
- Main memory 支援 misaligned load/store
- LR/SC reservation set size 為 64 byte

**RVA22U (User Mode)**

RVA22U 定義 user-mode application 的 requirement。它是 RVA22S 的子集：

- 與 RVA22S 相同的 ISA extension
- 沒有 privileged feature（沒有 Sv39，沒有 PMP requirement）
- 針對在 RVA22S system 上運行的 user-space application

**Example: Checking RVA22 Compliance**

```c
// Check if system is RVA22S compliant
bool is_rva22s_compliant(void) {
    // Check ISA extensions via misa
    uint64_t misa = read_csr(misa);
    if ((misa & MISA_I) == 0) return false;  // RV64I
    if ((misa & MISA_M) == 0) return false;  // M extension
    if ((misa & MISA_A) == 0) return false;  // A extension
    if ((misa & MISA_F) == 0) return false;  // F extension
    if ((misa & MISA_D) == 0) return false;  // D extension
    if ((misa & MISA_C) == 0) return false;  // C extension
    
    // Check Sv39 support
    uint64_t satp = read_csr(satp);
    write_csr(satp, SATP_MODE_SV39 << 60);
    if ((read_csr(satp) >> 60) != SATP_MODE_SV39) return false;
    write_csr(satp, satp);  // Restore
    
    // Check PMP entries (at least 8)
    int pmp_count = count_pmp_entries();
    if (pmp_count < 8) return false;
    
    // Check other features via device tree or ACPI
    // ...
    
    return true;
}
```

---

## 14.3 RVA23 Profile

**Improvements Over RVA22**

RVA23 於 2023 年 ratify，在 RVA22 的基礎上添加了額外的 extension 和更嚴格的 requirement：

**New Mandatory Extensions**：

- **Zicond**：Integer conditional operation
- **Zimop**：May-be-operation（為未來 extension 保留）
- **Zcmop**：Compressed may-be-operation
- **Zcb**：Additional compressed instruction
- **Zfa**：Additional floating-point instruction
- **Zawrs**：Wait-on-reservation-set instruction
- **Supm**：Pointer masking（supervisor mode）

**Enhanced Requirements**：

- 最小 16 個 PMP entry（從 8 個增加）
- Sv48 或 Sv57 support（除了 Sv39）
- 改進的 performance counter requirement
- Zkt 的更嚴格 timing guarantee

**Forward Compatibility**

RVA23 與 RVA22 forward-compatible：

- 所有 RVA22 mandatory extension 在 RVA23 中仍然是 mandatory
- RVA22 software 在 RVA23 hardware 上未修改運行
- RVA23 添加 feature 但不刪除或更改現有 feature

**Migration Path**

Hardware vendor 可以支援兩個 profile：

```
RVA22 hardware → RVA23 hardware
  (2022-2024)      (2024+)
       ↓               ↓
   RVA22 software runs on both
   RVA23 software runs only on RVA23+
```

---

## 14.4 Embedded Profiles

**Embedded System Requirements**

Embedded system 與 application processor 有不同的約束：

- **Limited resource**：Small memory、low power、cost-sensitive
- **Real-time requirement**：Deterministic interrupt latency、predictable timing
- **No virtual memory**：許多 embedded system 在沒有 MMU 的情況下運行
- **Simpler software**：Bare-metal 或 RTOS，而不是 full OS

RISC-V embedded profile 針對這些 use case，具有 minimal mandatory extension 和 flexible configuration。

**RV32E Base ISA**

RV32E 是 RV32I 的簡化版本，只有 16 個 register（x0-x15）而不是 32 個。這節省了：

- **Silicon area**：Smaller register file
- **Power**：更少的 register 需要管理
- **Code size**：在某些情況下更短的 register encoding

RV32E 適用於 ultra-low-cost microcontroller，其中每個 gate 都很重要。

```assembly
# RV32E example: Only x0-x15 available
addi x10, x0, 42     # OK: x10 is in range
addi x20, x0, 42     # ERROR: x20 doesn't exist in RV32E
```

**Microcontroller-Oriented Features**

Embedded profile 強調：

- **Compressed instruction (C)**：減少 code size 25-30%
- **Multiply (M)**：對許多 embedded algorithm 至關重要
- **Bit manipulation (B)**：對 embedded control task 高效
- **Fast interrupt**：Low-latency interrupt handling

**Interrupt Handling for Embedded**

Embedded system 需要快速、deterministic interrupt response。RISC-V 提供兩種 interrupt architecture：

1. **CLINT (Core-Local Interruptor)**：Basic timer 和 software interrupt
2. **CLIC (Core-Local Interrupt Controller)**：Advanced interrupt controller for embedded

CLIC feature：

- **Vectored interrupt**：直接跳轉到 handler（無 dispatch overhead）
- **Nested interrupt**：Higher-priority interrupt preempt lower-priority
- **Tail-chaining**：Back-to-back interrupt 無需完整 context save/restore
- **Configurable level**：最多 256 個 priority level

```c
// CLIC interrupt handler (vectored)
void uart_irq_handler(void) {
    // Directly entered, no dispatch needed
    char c = UART->DATA;
    buffer[head++] = c;
    // Tail-chaining: if another interrupt pending, jump directly to it
}
```

**Low-Power Considerations**

Embedded system 優先考慮 power efficiency：

- **Clock gating**：禁用未使用的 module
- **Power domain**：關閉 inactive region
- **Sleep mode**：WFI (Wait For Interrupt) instruction
- **Dynamic voltage/frequency scaling**：調整 performance vs power

```assembly
# Enter low-power mode
wfi              # Wait for interrupt (CPU sleeps)
# CPU wakes on interrupt, resumes here
```

---

## 14.5 MMU vs No-MMU Systems

**MMU-Based Systems**

具有 Memory Management Unit (MMU) 的系統提供：

- **Virtual memory**：每個 process 都有自己的 address space
- **Memory protection**：Process 無法存取彼此的 memory
- **Demand paging**：按需從 disk 加載 page
- **Large address space**：64-bit virtual address

MMU-based system 運行 full operating system，如 Linux：

```
Process A: 0x00000000-0xFFFFFFFF (virtual)
    ↓ MMU translation
Physical: 0x80000000-0x80FFFFFF

Process B: 0x00000000-0xFFFFFFFF (virtual)
    ↓ MMU translation
Physical: 0x81000000-0x81FFFFFF
```

Requirement：

- Sv39/Sv48/Sv57 page table
- TLB for translation caching
- Page fault handling
- 足夠的 memory 用於 page table

**No-MMU Systems**

沒有 MMU 的系統直接使用 physical address：

- **Simpler hardware**：No TLB、no page table walker
- **Lower cost**：更少的 gate、更少的 power
- **Faster context switch**：不需要 TLB flush
- **Deterministic**：No TLB miss latency

No-MMU system 運行：

- **Bare-metal**：Single application、no OS
- **RTOS**：Real-time OS with static memory allocation
- **Embedded Linux**：uClinux（Linux without MMU）

**Memory Protection Without MMU**

No-MMU system 仍然可以使用 PMP (Physical Memory Protection) 提供 memory protection：

```c
// Protect firmware region (0x80000000-0x80010000)
pmpaddr0 = 0x80000000 >> 2;
pmpaddr1 = 0x80010000 >> 2;
pmpcfg0 = 0x89;  // L=1, A=1 (TOR), X=0, W=0, R=1

// Protect peripheral region (0x10000000-0x20000000)
pmpaddr2 = 0x10000000 >> 2;
pmpaddr3 = 0x20000000 >> 2;
pmpcfg0 |= (0x8B << 16);  // L=1, A=1 (TOR), X=0, W=1, R=1
```

PMP 提供：

- Region-based protection（不是 page-based）
- M-mode enforcement
- Locked region（在 reset 之前無法更改）

**Comparison**

| Feature | MMU System | No-MMU System |
|---------|------------|---------------|
| **Address Space** | Virtual (per-process) | Physical (shared) |
| **Memory Protection** | Page-based (4KB granularity) | Region-based (PMP) |
| **OS Support** | Linux, FreeBSD, etc. | RTOS, bare-metal, uClinux |
| **Context Switch** | Slow (TLB flush) | Fast (no TLB) |
| **Memory Overhead** | Page tables (~1% of RAM) | None |
| **Complexity** | High | Low |
| **Determinism** | Lower (TLB misses) | Higher (no TLB) |
| **Use Cases** | Servers, desktops, phones | Microcontrollers, embedded |

---

## 14.6 Comparison with ARM Cortex-M

**ARM Cortex-M Overview**

ARM Cortex-M 是 microcontroller 的主導 architecture。Cortex-M family 包括：

- **Cortex-M0/M0+**：Ultra-low-cost、minimal feature
- **Cortex-M3**：Mainstream、good performance/cost
- **Cortex-M4**：DSP extension、floating-point
- **Cortex-M7**：High performance、cache、double-precision FP
- **Cortex-M33/M55**：ARMv8-M、TrustZone、vector extension

**RISC-V Embedded vs ARM Cortex-M**

| Feature | RISC-V Embedded | ARM Cortex-M |
|---------|-----------------|--------------|
| **ISA** | Open, modular | Proprietary, fixed |
| **Registers** | 32 (or 16 for RV32E) | 16 (13 general-purpose) |
| **Instruction Set** | RISC, load-store | Thumb-2 (mixed 16/32-bit) |
| **Compressed Instructions** | Optional (C extension) | Standard (Thumb) |
| **Multiply/Divide** | Optional (M extension) | Standard (M3+) |
| **Floating-Point** | Optional (F/D extensions) | Optional (M4+) |
| **Vector/SIMD** | Optional (V extension) | Optional (M55 Helium) |
| **Privilege Levels** | M/S/U modes | Handler/Thread modes |
| **Memory Protection** | PMP (region-based) | MPU (region-based) |
| **Interrupt Controller** | CLIC (vectored, nested) | NVIC (vectored, nested) |
| **Interrupt Priorities** | Up to 256 levels | 8-256 levels (implementation) |
| **Licensing** | Open (no fees) | Proprietary (licensing fees) |
| **Ecosystem** | Growing | Mature, extensive |

**Interrupt Model Comparison (CLIC vs NVIC)**

ARM NVIC (Nested Vectored Interrupt Controller)：

- Vectored interrupt（直接跳轉到 handler）
- Nested interrupt with priority level
- Tail-chaining for back-to-back interrupt
- Automatic context save/restore

RISC-V CLIC (Core-Local Interrupt Controller)：

- 與 NVIC 類似的 feature
- 更靈活的 priority level（最多 256）
- Configurable interrupt mode
- 與 RISC-V privilege model 兼容

兩者都為 embedded interrupt handling 提供可比較的功能。

**Ecosystem Comparison**

**ARM Cortex-M Advantages**：

- Mature ecosystem（20+ 年）
- Extensive vendor support（ST、NXP、TI 等）
- Rich middleware（CMSIS、Mbed 等）
- Large developer community
- 在數十億設備中得到驗證

**RISC-V Embedded Advantages**：

- No licensing fee（lower cost）
- Open ISA（customizable）
- Modern design（比 ARM legacy 更乾淨）
- Growing ecosystem（SiFive、Espressif 等）
- Future-proof（community-driven evolution）

**Use Case Recommendations**

**Choose ARM Cortex-M when**：

- Mature ecosystem is critical
- Extensive middleware needed
- Proven reliability required
- Time-to-market is tight

**Choose RISC-V Embedded when**：

- Cost optimization is important
- Customization is needed
- Open-source ecosystem preferred
- Long-term flexibility valued

**Example: ESP32-C3 (RISC-V) vs ESP32 (Xtensa)**

Espressif 的 ESP32-C3 展示了 RISC-V 在 embedded 中的應用：

- RV32IMC core（32-bit、multiply、compressed）
- 160 MHz、single-core
- Wi-Fi + Bluetooth LE
- 400 KB SRAM、4 MB flash
- Arduino、ESP-IDF support

與 ESP32 (Xtensa) 相比：

- 類似的 performance
- 更好的 toolchain（GCC、LLVM）
- Open ISA（vs proprietary Xtensa）
- Growing ecosystem

---

## Summary

Platform profile 和 embedded system design 定義了 RISC-V 如何適應不同的 use case。本章涵蓋了五個關鍵領域，使 RISC-V 能夠服務於 high-performance application processor 和 resource-constrained embedded system。

**Platform profile** 通過定義 standard extension combination 來解決 fragmentation 問題。Profile 指定 mandatory extension、optional feature 和 implementation requirement。這為 software development 創建了 standard target 並保證了兼容性。命名慣例（RVA22S、RVA23U 等）清楚地表明了 profile 的目的和 version。

**RVA22 profile** 針對運行 rich operating system 的 application processor。RVA22S 需要 64-bit base ISA、standard extension（M、A、F、D、C）、bit manipulation（Zba、Zbb、Zbs）、Sv39 virtual memory 和至少 8 個 PMP entry。RVA22U 為 user-mode application 提供相同的 ISA extension。這個 profile 使 Linux、FreeBSD 和其他 full operating system 能夠在 RISC-V implementation 之間 portably 運行。

**RVA23 profile** 在 RVA22 的基礎上添加了額外的 extension 和更嚴格的 requirement。新的 mandatory extension 包括 Zicond（conditional operation）、Zfa（additional floating-point）和 Zawrs（wait-on-reservation-set）。Enhanced requirement 包括 16 個 PMP entry 和 Sv48/Sv57 support。RVA23 保持 forward compatibility — RVA22 software 在 RVA23 hardware 上未修改運行。

**Embedded profile** 針對具有不同約束的 microcontroller 和 real-time system。RV32E 將 register file 減少到 16 個 register，用於 ultra-low-cost application。Embedded system 強調 compressed instruction 以減少 code size、通過 CLIC 進行快速 interrupt handling，以及 low-power feature（如 WFI 和 clock gating）。這些 profile 使 RISC-V 能夠在 microcontroller market 中競爭。

**MMU vs no-MMU system** 代表兩種不同的 memory management 方法。MMU-based system 提供 virtual memory、per-process address space 和 page-based protection，使 full operating system（如 Linux）成為可能。No-MMU system 直接使用 physical address，提供更簡單的 hardware、更低的 cost 和更好的 determinism。PMP 為 no-MMU system 提供 region-based memory protection，在沒有 virtual memory 複雜性的情況下實現 task isolation。

**Comparison with ARM Cortex-M** 顯示了 RISC-V 在 embedded system 中的競爭地位。兩種 architecture 都提供類似的 feature — vectored interrupt、nested interrupt handling、memory protection 和 optional floating-point。RISC-V 在 licensing（open、no fee）、customization（modular ISA）和 modern design 方面具有優勢。ARM Cortex-M 在 ecosystem maturity、vendor support 和 proven deployment 方面領先。選擇取決於優先級：cost 和 flexibility 有利於 RISC-V，而 ecosystem maturity 有利於 ARM。

總之，platform profile 和 embedded system feature 使 RISC-V 能夠服務於從 tiny microcontroller 到 powerful application processor 的全部範圍，並為 software portability 和 hardware compliance 提供清晰的 standard。

---
