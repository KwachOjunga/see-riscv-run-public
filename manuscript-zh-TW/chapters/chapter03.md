# Chapter 3. Privilege Level 與 Execution Environment

**Part II — RISC-V Execution Model**

---

現代處理器必須平衡兩個相互競爭的需求：application 需要隔離和保護，而 operating system 需要對硬體的受控存取。RISC-V 透過簡潔的 privilege 架構解決這個問題，該架構具有三個 level——Machine、Supervisor 和 User——每個都有明確定義的責任和能力。

本章探討 RISC-V 如何實現 privilege 分離，從控制所有硬體的強制性 Machine mode，到實現 operating system 和 application 的可選 Supervisor 和 User mode。我們將檢視抽象化平台差異的 Supervisor Binary Interface (SBI)、定義程式如何與其環境互動的 execution environment interface，以及 RISC-V 的 privilege model 如何與 ARM 的 exception level 比較。理解這些概念對於任何使用 RISC-V system software 的人都至關重要，從 firmware 開發者到 OS kernel 工程師。

---

## 3.1 RISC-V Privilege 架構

**Privilege Model**

與其他現代處理器相比，RISC-V 的 privilege 架構優雅地簡潔。ARM 定義了四個 exception level，x86 有 ring 0-3（儘管通常只使用 0 和 3），RISC-V 只定義了三個 privilege mode：Machine (M)、Supervisor (S) 和 User (U)。

這種簡潔性是有意為之的。RISC-V 設計者觀察到，大多數系統只需要兩到三個 privilege level：一個用於 application，一個用於 operating system，一個用於 firmware。額外的 level 增加了複雜性，但對大多數使用情況沒有相應的好處。

Privilege mode 形成層次結構：

- **Machine mode** 是最高 privilege level，對所有硬體具有不受限制的存取
- **Supervisor mode** 用於 operating system，對 privileged operation 具有受控存取
- **User mode** 是最低 privilege level，用於具有最小 privilege 的 application

重要的是，並非所有 mode 都是強制性的。簡單的 embedded system 可能只實現 M-mode。具有基本 memory protection 的 microcontroller 可能實現 M-mode 和 U-mode。運行 Linux 的完整 application processor 實現所有三個 mode。

**Machine Mode (M-mode)**

Machine mode 是 RISC-V 中唯一強制性的 privilege level。每個 RISC-V 處理器都必須實現 M-mode，即使它不實現其他 privilege level。

M-mode 對整個系統具有完全且不受限制的存取。它可以：

- 存取所有 memory 和 I/O device
- 讀取和寫入所有 CSR
- 執行所有 instruction
- 配置並將 trap 委派給較低的 privilege level
- 控制 physical memory protection (PMP)

當 RISC-V 處理器 reset 時，它開始在 M-mode 中執行。第一個運行的程式碼——通常是 bootloader 或 firmware——在 M-mode 中執行。此程式碼初始化硬體，設置 memory protection，並可能最終將控制權轉移到 supervisor-mode software（如 operating system）或直接轉移到 user-mode application。

在沒有 operating system 的 embedded system 中，所有程式碼都可能在 M-mode 中運行。如果不需要，則不需要使用較低的 privilege level。這種靈活性使 RISC-V 能夠從簡單的 microcontroller 擴展到複雜的 application processor。

M-mode software 通常實現：

- **Bootloader**：reset 後運行的初始程式碼
- **Firmware**：Low-level 硬體初始化和 runtime service
- **SBI (Supervisor Binary Interface)**：為 supervisor-mode software 提供的 service
- **Trap handler**：用於未委派給較低 privilege level 的 trap

**Supervisor Mode (S-mode)**

Supervisor mode 是可選的，但在 application processor 中幾乎是通用的。它專為管理多個 user process 的 operating system kernel 而設計。

S-mode 的 privilege 比 U-mode 多，但比 M-mode 少。它可以：

- 控制 virtual memory（page table、TLB）
- 處理從 M-mode 委派的 trap
- 存取 supervisor-level CSR
- 執行 privileged instruction（如用於 TLB 管理的 SFENCE.VMA）

S-mode 不能：

- 存取 machine-level CSR
- 直接控制 physical memory protection
- 處理未委派給它的 trap
- 存取未映射到其 address space 的 I/O device

這種受限存取是有意為之的。M-mode firmware 保留對硬體的最終控制，而 S-mode software (OS kernel) 管理 virtual memory 和 user process。這種分離允許 firmware 提供 platform-specific service，同時 OS 保持 platform-independent。

Operating system 如 Linux、FreeBSD 和 real-time operating system 在 RISC-V 上的 S-mode 中運行。它們使用 S-mode privilege 來：

- 管理 virtual memory 以實現 process 隔離
- 處理來自 user application 的 system call
- 管理 interrupt 和 exception
- 調度 process 並管理資源

**User Mode (U-mode)**

User mode 是最低 privilege level，用於 application code。與 S-mode 一樣，U-mode 是可選的，但它在任何需要將 application 彼此隔離以及與 OS 隔離的系統中實現。

U-mode 具有最小的 privilege。它可以：

- 執行 unprivileged instruction
- 存取映射到其 virtual address space 的 memory
- 讀取一些 user-accessible CSR（如 performance counter）
- 透過 ECALL 向更高 privilege level 請求 service

U-mode 不能：

- 存取 privileged CSR
- 執行 privileged instruction
- 直接存取 I/O device
- 修改 page table 或 TLB
- 禁用 interrupt

當 U-mode code 需要 privileged operation（如 file I/O 或 memory allocation）時，它使用 ECALL instruction trap 到 S-mode。OS kernel 檢查 trap，如果允許則執行請求的操作，並返回到 U-mode。

這種隔離是現代 operating system 的基礎。每個 user process 在 U-mode 中運行，具有自己的 virtual address space。Process 不能相互干擾或干擾 kernel。如果 process 崩潰，它不會影響其他 process 或系統。

**Hypervisor Extension (H-mode)**

Hypervisor extension 增加了對虛擬化的支援，允許多個 operating system 同時在同一硬體上運行。與 M/S/U mode 不同，hypervisor extension 是真正可選的，僅在虛擬化使用情況下需要。

H extension 不添加新的 privilege level。相反，它添加：

- **VS-mode** (Virtual Supervisor)：Guest OS kernel mode
- **VU-mode** (Virtual User)：Guest OS user mode
- **HS-mode** (Hypervisor-extended Supervisor)：Hypervisor mode

這允許 hypervisor 在 HS-mode 中運行，管理在 VS-mode 和 VU-mode 中運行的多個 guest OS。每個 guest OS 認為它在 S-mode 和 U-mode 中運行，但實際上在虛擬化的 VS-mode 和 VU-mode 中運行。

Hypervisor 在 HS-mode (Hypervisor-extended Supervisor mode) 中運行並管理多個 guest operating system。每個 guest OS 在 VS-mode 中運行，認為它在 S-mode 中。Hypervisor 攔截某些操作並為每個 guest 提供虛擬化硬體。

這個 extension 對於 cloud computing 和 server 虛擬化很重要，但大多數 embedded system 甚至許多 application processor 都不實現它。

**Figure 3.1a: RISC-V Privilege 層次結構**

```
Privilege Level 層次結構：

┌─────────────────────────────────────────┐
│   Machine Mode (M-mode)                 │
│   Firmware, Bootloader, SBI             │
│   完全硬體存取                           │
└─────────────────────────────────────────┘
         ↑                    ↓
    Exception/          MRET (return)
    Interrupt
         ↑                    ↓
┌─────────────────────────────────────────┐
│   Supervisor Mode (S-mode)              │
│   OS Kernel                             │
│   Virtual memory, Delegated trap        │
└─────────────────────────────────────────┘
         ↑                    ↓
    ECALL/              SRET (return)
    Exception
         ↑                    ↓
┌─────────────────────────────────────────┐
│   User Mode (U-mode)                    │
│   Applications                          │
│   最小 privilege                         │
└─────────────────────────────────────────┘

轉換機制：
- 向上（到更高 privilege）：Exception, Interrupt, ECALL
- 向下（到較低 privilege）：MRET, SRET
- 未委派的 exception 直接 trap 到 M-mode
```

**Figure 3.1b: Hypervisor Extension（可選）**

```
Hypervisor Extension 架構：

┌─────────────────────────────────────────┐
│   HS-mode (Hypervisor)                  │
│   管理 guest VM                          │
└─────────────────────────────────────────┘
         ↑                    ↓
      Trap              Return
         ↑                    ↓
┌─────────────────────────────────────────┐
│   VS-mode (Guest OS Kernel)             │
│   虛擬化的 supervisor                    │
└─────────────────────────────────────────┘
         ↑                    ↓
      Trap              Return
         ↑                    ↓
┌─────────────────────────────────────────┐
│   VU-mode (Guest Applications)          │
│   虛擬化的 user mode                     │
└─────────────────────────────────────────┘

特性：
- Two-stage address translation
- 額外的 CSR 用於虛擬化控制
- Guest OS 認為它在 S-mode/U-mode 中運行
```

---

## 3.2 Privilege Level vs ARM Exception Level

**RISC-V 的三層 Model**

RISC-V 的 M/S/U privilege model 是刻意最小化的。三個 level 足以滿足大多數系統：

- M-mode 用於 firmware 和 platform-specific code
- S-mode 用於 OS kernel
- U-mode 用於 application

這種簡潔性有優勢：

- 更容易理解和實現
- 更少的 privilege 轉換意味著更少的 overhead
- 清晰的關注點分離

Model 也很靈活。不需要所有三個 level 的系統可以省略 S-mode 或 U-mode。Bare-metal embedded system 可能只使用 M-mode。簡單的 RTOS 可能使用 M-mode 和 U-mode 而不使用 S-mode。

**ARM 的四層 Model**

ARM 採用不同的方法，具有四個 exception level (EL)：

- **EL0**：Application（類似 RISC-V U-mode）
- **EL1**：OS kernel（類似 RISC-V S-mode）
- **EL2**：Hypervisor（用於虛擬化）
- **EL3**：Secure monitor（用於 TrustZone）

此外，ARM 的 TrustZone 創建了兩個平行世界——Secure 和 Non-secure——每個都有自己的 exception level 集。這造成了相當大的複雜性：

- EL3 管理 Secure 和 Non-secure world 之間的轉換
- 每個 world 都有自己的 EL0、EL1 和 EL2
- 不同的 exception level 具有不同的能力

ARM model 解決了真實需求。EL3 和 TrustZone 提供硬體強制的安全隔離，對於處理敏感資料（如支付憑證）的行動裝置很重要。EL2 為 server 和 cloud computing 實現高效虛擬化。

但這種複雜性是有代價的：

- 更複雜的 privilege 轉換
- 更多的 CSR 和 state 需要管理
- 更陡峭的學習曲線
- 更多的實現複雜性

**Privilege 轉換比較**

RISC-V 和 ARM 之間改變 privilege level 的機制不同，儘管概念相似。

*RISC-V 轉換*：

- **向上**（到更高 privilege）：Exception、interrupt 或 ECALL instruction
- **向下**（到較低 privilege）：MRET 或 SRET instruction
- Trap 原因存儲在 xcause CSR 中
- Return address 存儲在 xepc CSR 中
- Trap handler address 來自 xtvec CSR

*ARM 轉換*：

- **向上**：Exception 或 SVC (supervisor call) instruction
- **向下**：ERET (exception return) instruction
- Exception syndrome 存儲在 ESR_ELx 中
- Return address 存儲在 ELR_ELx 中
- Exception vector 來自 VBAR_ELx

概念幾乎相同——兩種架構都保存 state、跳轉到 handler，並提供 return instruction。主要區別在於命名和涉及的 level 數量。

RISC-V 的更簡單 model 意味著需要處理的情況更少。U-mode 中的 exception 可能會進入 S-mode（如果委派）或 M-mode（如果未委派）。在 ARM 中，EL0 中的 exception 可能會進入 EL1、EL2 或 EL3，具體取決於配置，如果涉及 TrustZone，可能涉及 world switch。

**安全 Model 比較**

安全性是架構差異最大的地方。

*ARM TrustZone*：
ARM 的 TrustZone 創建了兩個平行的 execution environment——Secure 和 Non-secure world。每個 world 都有自己的：

- Memory region（某些 memory 僅限 Secure）
- Peripheral（某些 device 僅限 Secure）
- Exception level（每個 world 中的 EL0-EL2）

Secure world 可以存取 Non-secure 資源，但反之則不行。EL3 (Secure monitor) 管理 world 之間的轉換。這為安全關鍵程式碼（如加密操作、DRM 和支付處理）提供了強大的隔離。

TrustZone 在 ARM Cortex-A 處理器中是強制性的，並廣泛用於行動裝置。它已被證明可以有效保護敏感操作免受受損 OS kernel 的影響。

*RISC-V PMP 和 ePMP*：
RISC-V 採用不同的方法，使用 Physical Memory Protection (PMP)。PMP 允許 M-mode 為較低 privilege level 定義具有特定存取權限的 memory region。

PMP 提供：

- 最多 16 個（或更多）memory region
- Per-region permission（read、write、execute）
- 保護 S-mode 和 U-mode
- 靈活的 region 大小和對齊

Enhanced PMP (ePMP) extension 添加：

- 即使 M-mode 也無法修改的 locked region
- 更靈活的 permission model
- 更好地支援安全使用情況

PMP 比 TrustZone 簡單，但不太全面。它不提供獨立的 execution environment 或 world switching。對於許多使用情況，PMP 就足夠了。對於需要強隔離的高安全性 application，可能需要額外的機制。

RISC-V 的方法是保持 base 架構簡單，並允許針對特定安全需求的 extension。如果需要，custom extension 可以添加類似 TrustZone 的功能。這種靈活性允許實現為其使用情況選擇正確的安全 model，而不會為不需要它的系統強制增加複雜性。

**Figure 3.2: RISC-V vs ARM Privilege Model 比較**

```
RISC-V Privilege Level:          ARM Exception Level:

┌─────────────────┐              ┌─────────────────┐
│    M-mode       │              │      EL3        │
│   Firmware      │              │ Secure Monitor  │
│      SBI        │              │   TrustZone     │
└─────────────────┘              └─────────────────┘
        ↕                                ↕
┌─────────────────┐              ┌─────────────────┐
│    S-mode       │              │      EL2        │
│   OS Kernel     │              │   Hypervisor    │
└─────────────────┘              └─────────────────┘
        ↕                                ↕
┌─────────────────┐              ┌─────────────────┐
│    U-mode       │              │      EL1        │
│  Applications   │              │    OS Kernel    │
└─────────────────┘              └─────────────────┘
                                         ↕
                                 ┌─────────────────┐
                                 │      EL0        │
                                 │  Applications   │
                                 └─────────────────┘

RISC-V: 3 level (簡潔)          ARM: 4 level (全面)
可選 S/U mode                    TrustZone (Secure/Non-secure world)
Hypervisor extension (可選)      EL2 hypervisor (標準)
```

**Trade-off 和使用情況**

兩種 model 都不是普遍優越的——每種都做出不同的 trade-off。

*RISC-V 優勢*：

- 更容易理解和實現
- Privilege 轉換的 overhead 更低
- 靈活——只實現你需要的
- 更容易驗證和確認

*ARM 優勢*：

- TrustZone 提供強大的安全隔離
- 四個 exception level 實現更細粒度的 privilege 分離
- 成熟的生態系統，具有經過驗證的安全解決方案
- Hypervisor 支援是標準的（EL2）

對於 embedded system 和 microcontroller，RISC-V 的簡潔性通常更可取。對於處理敏感資料的行動裝置，ARM 的 TrustZone 已被證明很有價值。對於 server 和 cloud computing，兩種架構都可以提供足夠的虛擬化支援（RISC-V 透過 H extension，ARM 透過 EL2）。

關鍵見解是，RISC-V 的模組化方法允許在需要時添加複雜性（透過 extension，如用於虛擬化的 H），同時保持 base 簡單。ARM 的方法是在 base 架構中提供全面的功能，這確保了一致性，但即使對於不需要所有功能的系統也強制增加複雜性。

---

## 3.3 Execution Environment Interface (EEI)

**什麼是 EEI？**

Execution Environment Interface (EEI) 定義了程式與其 execution environment 之間的介面。它指定：

- 哪些 instruction 可用
- 如何進行 system call
- 程式如何與 I/O 互動
- Memory layout 和 addressing
- Interrupt 和 exception handling

不同的 privilege level 有不同的 EEI。User-mode 程式的 EEI 與 supervisor-mode kernel 不同，後者又與 machine-mode firmware 不同。

EEI 概念很重要，因為它將 ISA（存在哪些 instruction）與 execution environment（這些 instruction 如何與系統互動）分開。相同的 RISC-V ISA 可以支援不同使用情況的不同 EEI。

**Application Execution Environment (AEE)**

Application Execution Environment 是 user-mode 程式的 EEI。它定義了 application 可以做什麼以及它們如何與 operating system 互動。

典型的 AEE 提供：

- 具有 process 隔離的 virtual memory
- 透過 ECALL instruction 的 system call
- Standard library function
- File I/O、networking 和其他 OS service
- 用於 asynchronous event 的 signal handling

AEE 通常由 operating system 和 ABI (Application Binary Interface) 定義。例如，RISC-V 上的 Linux 定義了特定的 AEE，包括：

- System call number 和 calling convention
- Signal delivery mechanism
- Virtual memory layout
- Thread-local storage access

為此 AEE 編寫的 application 可以在任何 RISC-V Linux 系統上運行，無論底層硬體如何。

**Supervisor Execution Environment (SEE)**

Supervisor Execution Environment 是在 S-mode 中運行的 OS kernel 的 EEI。它定義了 kernel 如何與底層 firmware 和硬體互動。

SEE 通常由 M-mode firmware 透過 Supervisor Binary Interface (SBI) 提供。SBI 定義了 M-mode 向 S-mode 提供的 service，例如：

- Timer 管理
- Inter-processor interrupt (IPI)
- Remote fence operation（TLB shootdown）
- System reset 和 shutdown
- Console I/O（用於 debugging）

透過 SBI 提供這些 service，firmware 抽象化了 platform-specific 細節。OS kernel 可以是 platform-independent 的，呼叫 SBI function 而不是直接存取硬體。這類似於 x86 上的 BIOS/UEFI 或 ARM 的 PSCI (Power State Coordination Interface)。

**SBI 範例**

以下是 OS kernel 如何使用 SBI 設置 timer 的範例：

```c
// SBI timer extension
#define SBI_EXT_TIME 0x54494D45

// 設置下一個 timer interrupt
void sbi_set_timer(uint64_t stime_value) {
    register unsigned long a0 asm("a0") = stime_value;
    register unsigned long a6 asm("a6") = 0;  // fid = 0 (set_timer)
    register unsigned long a7 asm("a7") = SBI_EXT_TIME;

    asm volatile (
        "ecall"
        : "+r" (a0)
        : "r" (a6), "r" (a7)
        : "memory"
    );
}
```

Kernel 呼叫 `sbi_set_timer()`，它使用 ECALL trap 到 M-mode。M-mode firmware 處理請求，配置硬體 timer，並返回到 S-mode。Kernel 不需要知道 timer 硬體的細節——SBI 抽象化了這些。

**Bare-Metal Execution Environment**

並非所有 RISC-V 系統都運行 operating system。Embedded system 通常直接在硬體上運行「bare-metal」程式碼，而沒有 OS。

在 bare-metal environment 中：

- 程式碼在 M-mode 中運行，具有完全硬體存取權限
- 沒有 virtual memory 或 process 隔離
- 直接存取所有 peripheral
- Custom interrupt handler
- Application-specific memory layout

Bare-metal EEI 由硬體平台和使用的任何 runtime library 定義。例如，microcontroller 可能提供：

- 初始化硬體的 startup code
- Interrupt vector table
- 基本 I/O function
- Memory map 文檔

Bare-metal 程式設計在 embedded system、IoT device 和 real-time application 中很常見，在這些應用中，OS 的 overhead 是不可接受的或不必要的。

**Bare-Metal 範例**

以下是簡單的 bare-metal startup code：

```assembly
# Bare-metal startup code (M-mode)
.section .text.init
.global _start

_start:
    # 設置 stack pointer
    la sp, _stack_top

    # 清除 BSS section
    la t0, _bss_start
    la t1, _bss_end
1:
    bge t0, t1, 2f
    sd zero, 0(t0)
    addi t0, t0, 8
    j 1b
2:
    # 設置 trap handler
    la t0, trap_handler
    csrw mtvec, t0

    # 啟用 interrupt
    li t0, 0x888        # MIE, MTIE, MSIE
    csrw mie, t0
    li t0, 0x8          # MIE bit
    csrw mstatus, t0

    # 跳轉到 main
    call main

    # 如果 main 返回，halt
halt:
    wfi
    j halt
```

這段程式碼在 M-mode 中運行，直接配置硬體。沒有 OS，沒有 SBI——只有程式碼和硬體。

---

## Summary

RISC-V 的 privilege 架構為分離 system software 責任提供了簡潔、靈活的 model。三個 privilege level——Machine (M-mode)、Supervisor (S-mode) 和 User (U-mode)——形成層次結構，每個 level 都有明確定義的能力和限制。M-mode 是強制性的，具有不受限制的硬體存取權限，適合 firmware 和 bootloader。S-mode 是可選的，專為 operating system 設計，對 privileged operation 和 virtual memory 具有受控存取。U-mode 是可選的，用於 application，具有最小 privilege 和強隔離。

Privilege model 比最初看起來更靈活。簡單的 embedded system 只實現 M-mode。具有基本保護的 microcontroller 實現 M-mode 和 U-mode。運行 Linux 的完整 application processor 實現所有三個 mode。Hypervisor extension 為虛擬化添加了 VS-mode 和 VU-mode，使多個 guest operating system 能夠在單個處理器上運行。

Supervisor Binary Interface (SBI) 在 M-mode firmware 和 S-mode operating system 之間提供標準化介面。SBI 抽象化 platform-specific 細節，允許 OS kernel 在不同的 RISC-V 實現之間可移植。關鍵 SBI service 包括 timer 管理、inter-processor interrupt、remote fence operation 和 system reset。OpenSBI 提供支援眾多平台的參考實現。

Execution Environment Interface (EEI) 定義了程式如何與其 execution environment 互動。Application Execution Environment (AEE) 是 user 程式看到的——system call、standard library function 和 OS service。Supervisor Execution Environment (SEE) 是 OS kernel 看到的——SBI call、硬體存取和平台 service。Bare-metal environment 提供直接硬體存取，沒有 OS layer。

與 ARM 的四個 exception level (EL0-EL3) 相比，RISC-V 的三個 privilege level 更簡單、更靈活。ARM 的 EL3 (Secure Monitor) 和 EL2 (Hypervisor) 在 ARMv8-A 中始終存在，即使未使用。RISC-V 使 S-mode 和 U-mode 可選，並將 hypervisor 支援作為 extension 添加。這種模組化允許 RISC-V 從微型 microcontroller 擴展到高性能 server，而不會帶來不必要的複雜性。

Privilege 架構反映了 RISC-V 的設計哲學：提供最少的強制性功能，使其他一切都可選，並保持關注點的清晰分離。這種方法使跨廣泛應用的高效實現成為可能，同時保留在需要時添加高級功能的靈活性。
