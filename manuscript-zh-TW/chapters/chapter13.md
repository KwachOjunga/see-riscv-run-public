# Chapter 13. SoC Integration

**Part VIII — System Design, Platform Spec & SoC Integration**

---

RISC-V processor core 不會單獨運作。要構建完整的 system-on-chip (SoC)，core 必須與 memory controller、interrupt controller、I/O device 和 system interconnect 整合。這種整合決定了 software 如何存取 hardware、device 如何通訊，以及系統如何維護 security 和 performance。

RISC-V 提供 modular approach 來進行 SoC design。與規定特定 peripheral implementation 的 monolithic architecture 不同，RISC-V 定義 standard interface，同時允許 implementation 的靈活性。Physical Memory Protection (PMP) unit 控制 machine mode 中的 memory access。Platform-Level Interrupt Controller (PLIC) 將 interrupt 從 device 路由到 core。Memory-mapped I/O (MMIO) 提供統一的 device access 機制。System interconnect（如 TileLink 和 AXI）將 component 連接在一起。

本章探討 RISC-V core 如何整合到完整的 SoC 中。我們將檢視基本 component — PMP、IOMMU、PLIC、MMIO、memory map、interconnect 和 DMA — 並了解它們如何協同工作以創建功能性系統。理解 SoC integration 對於 system designer、firmware developer 和任何使用 RISC-V hardware platform 的人都至關重要。

---

## 13.1 Physical Memory Protection (PMP)

**The Need for Memory Isolation**

在沒有 virtual memory 的系統中，我們如何防止 untrusted code 存取 sensitive memory region？Bare-metal application 可能需要保護其 firmware 免受 buggy driver 的影響。Embedded RTOS 可能需要將 task 彼此隔離。Machine-mode firmware 必須保護自己免受 supervisor-mode operating system 的影響。

Physical Memory Protection (PMP) 使用 physical address 提供 hardware-enforced memory access control。與在 S-mode 中運作的 virtual memory 的 page table 不同，PMP 在 M-mode 中運作並應用於所有較低的 privilege level。這使 PMP 對於沒有 MMU 的系統至關重要，並且對於保護 M-mode resource（即使在有 MMU 的系統中）也很有用。

**PMP Architecture**

PMP 使用一組 configuration register 來定義受保護的 memory region：

*pmpcfg0-pmpcfg15*：Configuration register（RV32 有 4 個，RV64 有 16 個）

*pmpaddr0-pmpaddr63*：Address register（最多 64 個 region）

每個 PMP entry 包含：

- Address register (pmpaddr) 定義 region
- Configuration byte（在 pmpcfg 中）指定 permission 和 matching mode

**PMP Configuration Format**

```
pmpcfg format (8 bits per entry):
  7     6:5   4:3   2     1     0
  L     0 0   A     X     W     R

L: Lock bit（防止進一步修改）
A: Address matching mode（OFF、TOR、NA4、NAPOT）
X: Execute permission
W: Write permission
R: Read permission
```

**Address Matching Modes**

PMP 支援四種 address matching mode：

*OFF (A=0)*：Region 被禁用，不應用保護

*TOR (A=1)*：Top-of-Range。Region 是 [pmpaddr[i-1], pmpaddr[i])

*NA4 (A=2)*：Naturally Aligned 4-byte region

*NAPOT (A=3)*：Naturally Aligned Power-Of-Two region

最常用的 mode 是 TOR 和 NAPOT：

```c
// Example: Protect 64KB region at 0x80000000 using NAPOT
// NAPOT encoding: address = base | (size/2 - 1)
// For 64KB: 0x80000000 | 0x7FFF = 0x80007FFF
pmpaddr0 = 0x80007FFF >> 2;  // Right shift by 2 (PMP addresses are >> 2)
pmpcfg0 = 0x1F;  // L=0, A=3 (NAPOT), X=1, W=1, R=1

// Example: Protect range [0x80000000, 0x80010000) using TOR
pmpaddr0 = 0x80000000 >> 2;  // Start address
pmpaddr1 = 0x80010000 >> 2;  // End address
pmpcfg0 = 0x09;  // Entry 0: A=1 (TOR), X=0, W=0, R=1
```

**PMP Priority and Matching**

當 access 發生時，PMP 從最低到最高 index 檢查 entry。第一個匹配的 entry 決定 access permission。如果沒有 entry 匹配，則拒絕 access（對於 M-mode，此行為是 implementation-defined）。

```
Access check algorithm:
1. For i = 0 to N-1:
   - If address matches pmpaddr[i] region:
     - Check permissions in pmpcfg[i]
     - If allowed: grant access
     - If denied: raise access fault
2. If no match: deny access (or allow for M-mode)
```

**Lock Bit**

Lock bit (L) 防止進一步修改 PMP entry，直到下一次 reset。這對於保護 M-mode firmware 免受被 compromised 的 S-mode code 禁用至關重要：

```c
// Lock the firmware region
pmpaddr0 = 0x80000000 >> 2;
pmpaddr1 = 0x80010000 >> 2;
pmpcfg0 = 0x89;  // L=1, A=1 (TOR), X=0, W=0, R=1

// Any subsequent write to pmpcfg0 or pmpaddr0/1 is ignored
pmpcfg0 = 0x00;  // This write has no effect!
```

**PMP Use Cases**

1. **Firmware Protection**：M-mode firmware 保護自己免受 S-mode OS 的影響
2. **Device Memory Protection**：防止未經授權存取 MMIO region
3. **Task Isolation**：Embedded RTOS 在沒有 MMU 的情況下隔離 task
4. **Secure Boot**：保護 boot ROM 和 secure storage

---

## 13.2 IOMMU for RISC-V

**The DMA Problem**

Direct Memory Access (DMA) 允許 device 在沒有 CPU 介入的情況下存取 memory，從而提高 I/O-intensive workload 的 performance。但 DMA 造成了 security 問題：device 使用 physical address 並繞過 CPU 的 virtual memory protection。Malicious 或 buggy device 可能讀取 sensitive data 或破壞 kernel memory。

Input-Output Memory Management Unit (IOMMU) 通過為 device 提供 address translation 和 access control 來解決這個問題。就像 MMU 為 CPU 轉換 virtual address 一樣，IOMMU 為 peripheral 轉換 device address。這使得：

- **Device isolation**：每個 device 只能看到自己的 memory
- **Virtualization**：Virtual machine 可以安全地 pass through device
- **Large address space**：32-bit device 可以存取 >4GB memory

**RISC-V IOMMU Architecture**

RISC-V IOMMU specification 定義了 device address translation 的 standard interface。IOMMU 位於 device 和 memory 之間，攔截 device memory request 並通過 device page table 轉換它們。

```
Device → IOMMU → Memory
         ↓
    Device Context
    Device Page Tables
```

Key component：

*Device Context*：Per-device configuration（page table pointer、permission）

*Device Directory Table (DDT)*：將 device ID 映射到 device context

*I/O Page Tables*：類似於 CPU page table，但用於 device address

*Command Queue*：Software 向 IOMMU 發送 command（invalidate TLB 等）

*Fault Queue*：IOMMU 向 software 報告 translation fault

**Device Address Translation**

當 device 發出 memory request 時：

1. IOMMU 從 request 中提取 device ID
2. 在 DDT 中查找 device context
3. Walk device page table 以轉換 address
4. 檢查 permission（read/write/execute）
5. 將轉換後的 request 轉發到 memory 或報告 fault

```c
// Simplified IOMMU translation
struct device_context {
    uint64_t page_table_root;  // Root page table address
    uint32_t permissions;       // Device permissions
    uint32_t address_width;     // Device address width
};

// Device issues read from device address 0x1000
device_addr = 0x1000;
device_id = 0x42;

// IOMMU lookup
ctx = ddt[device_id];
physical_addr = walk_page_table(ctx.page_table_root, device_addr);
if (physical_addr && (ctx.permissions & READ)) {
    forward_to_memory(physical_addr);
} else {
    report_fault(device_id, device_addr);
}
```

**IOMMU Page Table Formats**

RISC-V IOMMU 支援多種 page table format：

- **Sv39/Sv48/Sv57**：與 CPU page table 相同的 format（為了簡單）
- **MSI Page Tables**：Message Signaled Interrupt 的特殊 format

使用與 CPU page table 相同的 format 簡化了 software — OS 可以為 device mapping 重用現有的 page table code。

**IOMMU vs ARM SMMU**

ARM 的 System Memory Management Unit (SMMU) 提供類似的功能：

| Feature | RISC-V IOMMU | ARM SMMU |
|---------|--------------|----------|
| **Page Table Format** | Sv39/Sv48/Sv57 | LPAE (Long descriptor) |
| **Device Identification** | Device ID (configurable) | Stream ID |
| **Command Interface** | Command queue | CMDQ (Command Queue) |
| **Fault Reporting** | Fault queue | Event queue |
| **Virtualization** | Two-stage translation | Stage 1 + Stage 2 |
| **Complexity** | Simpler, modular | More complex, feature-rich |

RISC-V IOMMU 強調簡單性和重用現有 CPU MMU 概念，而 ARM SMMU 經過多代演進，具有廣泛的功能。

---

## 13.3 Platform-Level Interrupt Integration

**Interrupt Routing in SoCs**

典型的 RISC-V SoC 有數十個 interrupt source：UART、timer、GPIO、network controller、storage device。每個 device 需要在需要注意時向 CPU 發出信號。Platform-Level Interrupt Controller (PLIC) 通過以下方式管理這種複雜性：

- 從所有 device 收集 interrupt
- 將 interrupt 路由到適當的 core
- 管理 interrupt priority
- 提供 claim/complete 機制

**PLIC Architecture**

```
Interrupt Sources (1-1023)
    ↓
PLIC Gateway (per source)
    ↓
PLIC Core (priority arbitration)
    ↓
Interrupt Targets (cores × contexts)
```

Key concept：

*Interrupt Source*：可以生成 interrupt 的 device（編號 1-1023，source 0 保留）

*Interrupt Gateway*：將 device interrupt signal 轉換為 PLIC internal format

*Interrupt Target*：CPU context（每個 core 上的 M-mode 或 S-mode）

*Priority*：每個 source 都有 priority（0 = never interrupt，1-7 = increasing priority）

*Threshold*：每個 target 都有 threshold（只有 priority > threshold 的 interrupt 才會被傳遞）

**PLIC Memory Map**

PLIC 通過 memory-mapped register 存取：

```
Base Address: 0x0C000000 (typical)

Interrupt Priorities:
  0x0C000000 + 4*source_id: Priority for source (0-7)

Interrupt Pending:
  0x0C001000: Pending bits (1024 bits, read-only)

Interrupt Enable:
  0x0C002000 + 0x80*context: Enable bits for context

Priority Threshold:
  0x0C200000 + 0x1000*context: Threshold for context

Claim/Complete:
  0x0C200004 + 0x1000*context: Claim and complete register
```

**Interrupt Handling Flow**

1. **Device asserts interrupt**：Device 提升 interrupt line
2. **PLIC gateway captures**：Gateway 設置 pending bit
3. **PLIC arbitration**：PLIC 為每個 target 選擇最高 priority 的 pending interrupt
4. **CPU notification**：PLIC 向 CPU 斷言 external interrupt
5. **Software claim**：Interrupt handler 讀取 claim register（返回 source ID，清除 pending）
6. **Software handling**：Handler 服務 device
7. **Software complete**：Handler 將 source ID 寫入 complete register

```c
// PLIC interrupt handler
void plic_handler(void) {
    uint32_t source = plic_claim();  // Read claim register

    if (source == UART0_IRQ) {
        uart0_interrupt_handler();
    } else if (source == TIMER_IRQ) {
        timer_interrupt_handler();
    }
    // ... handle other sources

    plic_complete(source);  // Write to complete register
}

uint32_t plic_claim(void) {
    volatile uint32_t *claim = (uint32_t*)(PLIC_BASE + 0x200004);
    return *claim;  // Reading claim atomically claims the interrupt
}

void plic_complete(uint32_t source) {
    volatile uint32_t *complete = (uint32_t*)(PLIC_BASE + 0x200004);
    *complete = source;  // Writing complete releases the interrupt
}
```

**Multi-Core Interrupt Routing**

在 multi-core system 中，每個 core 都有單獨的 M-mode 和 S-mode context。PLIC 可以將 interrupt 路由到特定 core：

```c
// Configure UART interrupt to route to core 0 S-mode
#define PLIC_ENABLE_BASE 0x0C002000
#define UART0_IRQ 10
#define CORE0_S_MODE_CONTEXT 1

// Enable UART interrupt for core 0 S-mode
uint32_t *enable = (uint32_t*)(PLIC_ENABLE_BASE + 0x80 * CORE0_S_MODE_CONTEXT);
enable[UART0_IRQ / 32] |= (1 << (UART0_IRQ % 32));

// Set priority
uint32_t *priority = (uint32_t*)(PLIC_BASE + 4 * UART0_IRQ);
*priority = 5;  // Priority 5

// Set threshold
uint32_t *threshold = (uint32_t*)(PLIC_BASE + 0x200000 + 0x1000 * CORE0_S_MODE_CONTEXT);
*threshold = 0;  // Accept all priorities > 0
```

---

## 13.4 Memory-Mapped I/O (MMIO)

**Unified Address Space**

RISC-V 使用 memory-mapped I/O：device 顯示為 memory location。從 device register 讀取使用與從 RAM 讀取相同的 load instruction。寫入 device register 使用相同的 store instruction。這統一了 programming model — 不需要特殊的 I/O instruction。

```assembly
# Read UART status register
li   a0, 0x10000000      # UART base address
lw   a1, 0(a0)           # Read status register

# Write to UART data register
li   a2, 'A'             # Character to send
sw   a2, 4(a0)           # Write to data register
```

**MMIO Address Regions**

典型的 RISC-V SoC memory map 將 address space 劃分為 region：

```
0x00000000 - 0x0FFFFFFF: Debug/Boot ROM
0x10000000 - 0x1FFFFFFF: Peripherals (UART, SPI, GPIO, etc.)
0x20000000 - 0x2FFFFFFF: PLIC
0x30000000 - 0x3FFFFFFF: Reserved
0x40000000 - 0x7FFFFFFF: More peripherals
0x80000000 - 0xFFFFFFFF: DRAM
```

每個 peripheral 獲得一個 address block 用於其 register：

```
UART0: 0x10000000 - 0x10000FFF
  0x10000000: Status register
  0x10000004: Data register
  0x10000008: Control register
  0x1000000C: Baud rate register
```

**MMIO Ordering Requirements**

MMIO access 具有與 normal memory 不同的 ordering requirement：

- **Device register 可能有 side effect**：讀取 status register 可能清除 interrupt flag
- **Write order matters**：以錯誤順序寫入 control register 可能導致 device malfunction
- **Read/write dependency**：Write 必須在後續 read 看到 result 之前完成

RISC-V 提供 fence instruction 來強制執行 ordering：

```assembly
# Ensure MMIO write completes before continuing
li   a0, 0x10000000
li   a1, 0x1
sw   a1, 0(a0)           # Write to control register
fence iorw, iorw         # Ensure write completes
lw   a2, 4(a0)           # Read status register
```

FENCE instruction 採用兩個 operand，指定 predecessor 和 successor operation：

- **i**：Device input（MMIO read）
- **o**：Device output（MMIO write）
- **r**：Memory read
- **w**：Memory write

常見模式：

- `fence iorw, iorw`：Full fence（所有 operation）
- `fence ow, ow`：確保 MMIO write 按順序完成
- `fence ir, ir`：確保 MMIO read 按順序完成

**Uncached vs Cached MMIO**

MMIO region 必須在 page table 中標記為 uncached。Caching device register 會導致：

- **Stale data**：Cache 可能返回舊值而不是當前 device state
- **Lost write**：寫入 cached location 可能不會到達 device
- **Side effect loss**：讀取 cached value 不會觸發 device side effect

MMIO 的 page table entry 使用特殊 attribute：

```c
// Mark MMIO region as uncached and unbuffered
pte = (physical_addr >> 12) << 10;  // PPN
pte |= PTE_V | PTE_R | PTE_W;       // Valid, readable, writable
pte |= PTE_A | PTE_D;                // Accessed, dirty
// Do NOT set PTE_C (cacheable) for MMIO
```

---

## 13.5 SoC Memory Map

**Typical RISC-V SoC Layout**

完整的 RISC-V SoC memory map 包括 ROM、RAM、peripheral 和 reserved region。以下是 32-bit SoC 的典型 layout：

```
Address Range          Size      Description
0x00000000-0x00000FFF  4 KB      Debug ROM
0x00001000-0x00000FFF  60 KB     Reserved
0x00010000-0x0001FFFF  64 KB     Boot ROM (mask ROM)
0x00020000-0x00FFFFFF  ~16 MB    Reserved
0x01000000-0x01FFFFFF  16 MB     CLINT (Core-Local Interruptor)
0x02000000-0x0BFFFFFF  160 MB    Reserved
0x0C000000-0x0FFFFFFF  64 MB     PLIC
0x10000000-0x1000FFFF  64 KB     UART0
0x10010000-0x1001FFFF  64 KB     SPI0
0x10020000-0x1002FFFF  64 KB     GPIO
0x10030000-0x1FFFFFFF  ~256 MB   Other peripherals
0x20000000-0x3FFFFFFF  512 MB    Reserved
0x40000000-0x7FFFFFFF  1 GB      External devices
0x80000000-0xFFFFFFFF  2 GB      DRAM
```

**Address Decode and Routing**

SoC interconnect decode address 並將 request 路由到適當的 component：

```
CPU issues load/store
    ↓
Address decode
    ↓
    ├─ 0x00000000-0x0FFFFFFF → Boot ROM / CLINT / PLIC
    ├─ 0x10000000-0x1FFFFFFF → Peripheral bus
    ├─ 0x20000000-0x7FFFFFFF → Reserved / External
    └─ 0x80000000-0xFFFFFFFF → DRAM controller
```

**Memory Map Examples**

不同的 RISC-V platform 使用不同的 memory map：

**SiFive FU540 (HiFive Unleashed)**:

```
0x00001000: Boot ROM
0x02000000: CLINT
0x0C000000: PLIC
0x10000000: UART0
0x10010000: QSPI0
0x10040000: GPIO
0x80000000: DDR (8 GB)
```

**Kendryte K210**:

```
0x00000000: SRAM (6 MB)
0x40000000: Peripherals
0x50000000: AI accelerator
0x80000000: Flash (16 MB)
```

**QEMU virt machine**:

```
0x00001000: Boot ROM
0x02000000: CLINT
0x0C000000: PLIC
0x10000000: UART0
0x10001000: VirtIO devices
0x80000000: DRAM (configurable)
```

---

## 13.6 System Interconnects

**The Need for Interconnects**

Modern SoC 有多個 master（CPU core、DMA controller、GPU）和多個 slave（memory、peripheral、accelerator）。Interconnect fabric 連接這些 component，處理：

- **Address routing**：將 request 導向正確的 destination
- **Arbitration**：管理 concurrent access
- **Data width conversion**：將 32-bit device 連接到 64-bit bus
- **Clock domain crossing**：橋接不同的 clock frequency

**AXI (Advanced eXtensible Interface)**

ARM 的 AMBA AXI 由於其成熟度和 IP 可用性而廣泛用於 RISC-V SoC。AXI4 提供：

- **Separate read/write channel**：獨立的 read 和 write transaction
- **Burst transfer**：高效的 multi-beat transfer
- **Out-of-order completion**：Transaction 可以以任何順序完成
- **Quality of Service (QoS)**：基於 priority 的 arbitration

AXI signal：

```
Write Address Channel: AWADDR, AWLEN, AWSIZE, AWVALID, AWREADY
Write Data Channel:    WDATA, WSTRB, WLAST, WVALID, WREADY
Write Response:        BRESP, BVALID, BREADY
Read Address Channel:  ARADDR, ARLEN, ARSIZE, ARVALID, ARREADY
Read Data Channel:     RDATA, RRESP, RLAST, RVALID, RREADY
```

**AHB (Advanced High-performance Bus)**

AHB 比 AXI 簡單，適用於 lower-performance peripheral：

- **Single channel**：Address 和 data 共享相同的 channel
- **Pipelined**：Two-stage pipeline（address、data）
- **Simpler protocol**：更容易實作
- **Lower performance**：沒有 out-of-order，有限的 burst

**TileLink**

TileLink 是在 UC Berkeley 開發的 RISC-V-native interconnect：

- **Designed for RISC-V**：匹配 RISC-V memory model
- **Scalable**：從簡單的 embedded 到複雜的 multi-core
- **Cache coherence**：內建對 coherent cache 的支援
- **Three conformance level**：
  - **TL-UL**：Uncached Lightweight（簡單 peripheral）
  - **TL-UH**：Uncached Heavyweight（DMA、accelerator）
  - **TL-C**：Cached（coherent cache）

TileLink 對 RISC-V 的優勢：

- 對 RISC-V atomic（LR/SC、AMO）的 native support
- 高效的 cache coherence protocol
- Open specification（無 licensing）

**Interconnect Comparison**

| Feature | AXI4 | AHB | TileLink |
|---------|------|-----|----------|
| **Channels** | 5 independent | 1 shared | 3 (A, D, optional C/E) |
| **Burst Support** | Yes (up to 256 beats) | Yes (limited) | Yes |
| **Out-of-Order** | Yes | No | Yes |
| **Cache Coherence** | No (needs ACE) | No | Yes (TL-C) |
| **Complexity** | High | Low | Medium |
| **Performance** | High | Medium | High |
| **RISC-V Atomics** | Requires extensions | Requires extensions | Native support |
| **Licensing** | ARM (free for use) | ARM (free for use) | Open (BSD) |
| **Ecosystem** | Mature, extensive IP | Mature, simple IP | Growing, RISC-V focused |

**Choosing an Interconnect**

- **AXI**：Best for high-performance SoC、extensive IP ecosystem、industry standard
- **AHB**：Best for simple embedded system、low-cost peripheral
- **TileLink**：Best for RISC-V-native design、cache coherence、open ecosystem

---

## 13.7 DMA and Coherency

**DMA Controller Integration**

DMA controller 在沒有 CPU 介入的情況下在 memory 和 peripheral 之間傳輸 data。這使 CPU 可以執行其他 task，同時大型 data transfer 在背景中進行。

典型的 DMA use case：

- **Disk I/O**：在 storage 和 memory 之間傳輸 data
- **Network I/O**：在 NIC 和 memory 之間移動 packet
- **Audio/Video**：向/從 media device 串流 data
- **Memory-to-memory**：快速 memory copy operation

DMA controller architecture：

```
CPU configures DMA
    ↓
DMA reads source (memory or device)
    ↓
DMA writes destination (device or memory)
    ↓
DMA signals completion (interrupt)
```

**Cache Coherency Considerations**

當 CPU 有 cache 時，DMA 造成 coherency 問題：

**Problem 1: Stale cache data**

```
1. CPU writes data to memory (data in cache, not yet in RAM)
2. DMA reads from memory (gets old data, not cached data)
3. DMA sends wrong data to device
```

**Problem 2: Stale memory data**

```
1. DMA writes data to memory
2. CPU reads data (gets old cached data, not new DMA data)
3. CPU processes wrong data
```

**Solution**：

1. **Software cache management**（simple、lower performance）：

```c
// Before DMA read (device → memory)
dma_start(device, buffer, size);
dma_wait_complete();
cache_invalidate(buffer, size);  // Discard cached data
// Now CPU can read fresh data
```

```c
// Before DMA write (memory → device)
cache_flush(buffer, size);       // Write cached data to memory
dma_start(buffer, device, size);
dma_wait_complete();
```

1. **Hardware cache coherence**（complex、higher performance）：

- DMA controller 參與 cache coherence protocol
- DMA snoop CPU cache 或使用 coherent interconnect
- 需要 coherent interconnect（ACE for AXI、TL-C for TileLink）

**DMA and Virtual Memory**

DMA controller 通常使用 physical address，但 software 使用 virtual address。這造成挑戰：

**Problem**：Virtual address buffer 可能跨越 non-contiguous physical page

```
Virtual:  [0x1000-0x2FFF] (8 KB contiguous)
Physical: [0x80000000-0x80000FFF] + [0x85000000-0x85000FFF] (non-contiguous!)
```

**Solution**：

1. **Scatter-Gather DMA**：DMA controller 接受 physical address/length pair 的 list

```c
struct sg_entry {
    uint64_t addr;   // Physical address
    uint32_t len;    // Length in bytes
};

struct sg_entry sg_list[] = {
    {0x80000000, 4096},
    {0x85000000, 4096},
};
dma_start_sg(device, sg_list, 2);
```

1. **IOMMU**：將 device address 轉換為 physical address（見 Section 13.2）

2. **Physically contiguous buffer**：從 reserved physical memory 分配 DMA buffer

---

## Summary

SoC integration 將 RISC-V core 與系統的其餘部分連接起來。本章涵蓋了構成完整 RISC-V system-on-chip 的七個基本 component。

**Physical Memory Protection (PMP)** 使用 physical address 提供 hardware-enforced memory access control。PMP 在 M-mode 中運作並保護 memory region 免受 untrusted code 的影響。通過最多 64 個可配置 region、四種 address matching mode（OFF、TOR、NA4、NAPOT）和可鎖定的 entry，PMP 實現了 firmware protection、device memory isolation 以及在沒有 MMU 的系統中的 task separation。

**IOMMU** 通過轉換 device address 和強制執行 access control 將 memory protection 擴展到 device。RISC-V IOMMU 使用與 CPU MMU 相同的 page table format，簡化了 software implementation。這實現了 device isolation、virtualization 的安全 device passthrough，以及對 malicious 或 buggy device 的保護。

**Platform-Level Interrupt Controller (PLIC)** 管理 multi-core system 中的 interrupt routing。PLIC 從最多 1023 個 source 收集 interrupt，按 priority 仲裁，並將它們路由到適當的 CPU context。Claim/complete 機制確保 atomic interrupt handling，而 per-context enable mask 和 threshold 提供靈活的 interrupt management。

**Memory-Mapped I/O (MMIO)** 使用標準 load 和 store instruction 提供統一的 device access 機制。MMIO region 必須在 page table 中標記為 uncached，fence instruction 確保 device access 的正確 ordering。這種統一的 address space 簡化了 programming model，與具有單獨 I/O instruction 的 architecture 相比。

**SoC memory map** 將 address space 組織為 ROM、RAM、peripheral 和 reserved area 的 region。不同的 RISC-V platform 使用不同的 layout，但都遵循通過 interconnect fabric 進行 address decode 和 routing 的原則。理解 memory map 對於 firmware development 和 device driver programming 至關重要。

**System interconnect** 連接 SoC 中的多個 master 和 slave。AXI 提供 high performance 和 extensive IP ecosystem，AHB 為 embedded system 提供簡單性，TileLink 提供 RISC-V-native feature，包括 cache coherence。選擇取決於 performance requirement、IP availability 和 coherence need。

**DMA and coherency** 實現高效的 data transfer，但需要仔細管理 cache coherence。Software 可以使用 cache flush 和 invalidate operation，或 hardware 可以通過 snooping 或 coherent interconnect 提供 coherent DMA。IOMMU 或 scatter-gather DMA 解決了 DMA transfer 的 virtual-to-physical address translation 問題。

這些 component 共同構成了 RISC-V SoC design 的基礎，使從簡單的 microcontroller 到複雜的 multi-core application processor 的一切成為可能。

---
