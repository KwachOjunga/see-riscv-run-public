# Chapter 13. SoC Integration

**Part VIII — System Design, Platform Spec & SoC Integration**

---

## 🎯 Learning Objectives

After reading this chapter, you will be able to:

1. **Understand PMP's Role**: Grasp how Physical Memory Protection limits access at the hardware level
2. **Distinguish TOR vs NAPOT**: Understand the configuration differences and use cases for each Address Matching Mode
3. **Configure PMP Entries**: Set up read-only regions and intercept illegal writes
4. **Understand PLIC Architecture**: Grasp how the Platform-Level Interrupt Controller operates
5. **Integrate SoC Components**: Understand how CPU, Memory, and Peripherals connect via Interconnect

---

## 💡 Scenario: The Museum's Red Barrier Poles

> **Scene**: Junior stares at a "Store Access Fault" exception code on the screen, looking confused.

**Junior**: "Architect, this is so strange. I already turned off the MMU (virtual memory) and I'm using physical addresses directly to write to this variable. Why is the CPU still blocking me? Is the board broken?"

**Architect**: "The board is fine. You just hit PMP (Physical Memory Protection)'s 'red barrier poles.'

Imagine memory is a museum:

| Mechanism | Analogy | Function |
|-----------|---------|----------|
| **MMU (Page Table)** | Tour map | Tells you where exhibits are (VA → PA) |
| **PMP** | Red barrier poles + bulletproof glass | Hardware security, limits who can touch what |

Even if you bypass the tour guide (turn off MMU) and rush straight to the Mona Lisa, the hardware security (PMP Checker) will still stop you, because your ID (Privilege Mode) says you're just an ordinary visitor (S-mode/U-mode), and this area is only accessible to the museum director (M-mode)."

**Junior**: "So how do I set up these barrier poles? Do I need start and end addresses?"

**Architect**: "There are two common ways to set up the barriers:

1. **TOR (Top of Range)**: Like stretching a rope. You need two poles (two PMP Entries), and the area between them is the controlled region. Good for arbitrary-sized regions.

2. **NAPOT (Naturally Aligned Power of Two)**: Like placing a fixed-size dome (4KB, 2MB...) over exhibits. You only need to set the center point and dome size—more resource-efficient (uses only one Entry).

Today let's try using NAPOT to cover a 4KB region and make it 'read-only,' and see what happens to your program."

---

A RISC-V processor core doesn't operate in isolation. To build a complete system-on-chip (SoC), the core must integrate with memory controllers, interrupt controllers, I/O devices, and system interconnects. This integration determines how software accesses hardware, how devices communicate, and how the system maintains security and performance.

RISC-V provides a modular approach to SoC design. Unlike monolithic architectures that prescribe specific peripheral implementations, RISC-V defines standard interfaces while allowing flexibility in implementation. The Physical Memory Protection (PMP) unit controls memory access in machine mode. The Platform-Level Interrupt Controller (PLIC) routes interrupts from devices to cores. Memory-mapped I/O (MMIO) provides a uniform mechanism for device access. System interconnects like TileLink and AXI connect components together.

This chapter explores how RISC-V cores integrate into complete SoCs. We'll examine the essential components—PMP, IOMMU, PLIC, MMIO, memory maps, interconnects, and DMA—and see how they work together to create functional systems. Understanding SoC integration is crucial for system designers, firmware developers, and anyone working with RISC-V hardware platforms.

---

## 13.1 Physical Memory Protection (PMP)

**The Need for Memory Isolation**

In systems without virtual memory, how do we prevent untrusted code from accessing sensitive memory regions? A bare-metal application might need to protect its firmware from buggy drivers. An embedded RTOS might need to isolate tasks from each other. Machine-mode firmware must protect itself from supervisor-mode operating systems.

Physical Memory Protection (PMP) provides hardware-enforced memory access control using physical addresses. Unlike virtual memory's page tables (which operate in S-mode), PMP operates in M-mode and applies to all lower privilege levels. This makes PMP essential for systems without MMUs and useful for protecting M-mode resources even in systems with MMUs.

**PMP Architecture**

PMP uses a set of configuration registers to define protected memory regions:

*pmpcfg0-pmpcfg15*: Configuration registers (RV32 has 4, RV64 has 16)

*pmpaddr0-pmpaddr63*: Address registers (up to 64 regions)

Each PMP entry consists of:

- An address register (pmpaddr) defining the region
- A configuration byte (in pmpcfg) specifying permissions and matching mode

**PMP Configuration Format**

```
pmpcfg format (8 bits per entry):
  7     6:5   4:3   2     1     0
  L     0 0   A     X     W     R

L: Lock bit (prevents further modification)
A: Address matching mode (OFF, TOR, NA4, NAPOT)
X: Execute permission
W: Write permission
R: Read permission
```

**Address Matching Modes**

PMP supports four address matching modes:

*OFF (A=0)*: Region is disabled, no protection applied

*TOR (A=1)*: Top-of-Range. Region is [pmpaddr[i-1], pmpaddr[i])

*NA4 (A=2)*: Naturally Aligned 4-byte region

*NAPOT (A=3)*: Naturally Aligned Power-Of-Two region

The most commonly used modes are TOR and NAPOT:

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

When an access occurs, PMP checks entries from lowest to highest index. The first matching entry determines the access permissions. If no entry matches, the access is denied (for M-mode, this behavior is implementation-defined).

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

The lock bit (L) prevents further modification of a PMP entry until the next reset. This is crucial for protecting M-mode firmware from being disabled by compromised S-mode code:

```c
// Lock the firmware region
pmpaddr0 = 0x80000000 >> 2;
pmpaddr1 = 0x80010000 >> 2;
pmpcfg0 = 0x89;  // L=1, A=1 (TOR), X=0, W=0, R=1

// Any subsequent write to pmpcfg0 or pmpaddr0/1 is ignored
pmpcfg0 = 0x00;  // This write has no effect!
```

**PMP Use Cases**

1. **Firmware Protection**: M-mode firmware protects itself from S-mode OS
2. **Device Memory Protection**: Prevent unauthorized access to MMIO regions
3. **Task Isolation**: Embedded RTOS isolates tasks without MMU
4. **Secure Boot**: Protect boot ROM and secure storage

---

## 13.2 IOMMU for RISC-V

**The DMA Problem**

Direct Memory Access (DMA) allows devices to access memory without CPU intervention, improving performance for I/O-intensive workloads. But DMA creates a security problem: devices use physical addresses and bypass the CPU's virtual memory protection. A malicious or buggy device could read sensitive data or corrupt kernel memory.

An Input-Output Memory Management Unit (IOMMU) solves this by providing address translation and access control for devices. Just as the MMU translates virtual addresses for the CPU, the IOMMU translates device addresses for peripherals. This enables:

- **Device isolation**: Each device sees only its own memory
- **Virtualization**: Virtual machines can safely pass through devices
- **Large address spaces**: 32-bit devices can access >4GB memory

**RISC-V IOMMU Architecture**

The RISC-V IOMMU specification defines a standard interface for device address translation. The IOMMU sits between devices and memory, intercepting device memory requests and translating them through device page tables.

```
Device → IOMMU → Memory
         ↓
    Device Context
    Device Page Tables
```

Key components:

*Device Context*: Per-device configuration (page table pointer, permissions)

*Device Directory Table (DDT)*: Maps device IDs to device contexts

*I/O Page Tables*: Similar to CPU page tables, but for device addresses

*Command Queue*: Software sends commands to IOMMU (invalidate TLB, etc.)

*Fault Queue*: IOMMU reports translation faults to software

**Device Address Translation**

When a device issues a memory request:

1. IOMMU extracts device ID from the request
2. Looks up device context in DDT
3. Walks device page tables to translate address
4. Checks permissions (read/write/execute)
5. Forwards translated request to memory or reports fault

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

RISC-V IOMMU supports multiple page table formats:

- **Sv39/Sv48/Sv57**: Same format as CPU page tables (for simplicity)
- **MSI Page Tables**: Special format for Message Signaled Interrupts

Using the same format as CPU page tables simplifies software—the OS can reuse existing page table code for device mappings.

**IOMMU vs ARM SMMU**

ARM's System Memory Management Unit (SMMU) provides similar functionality:

| Feature | RISC-V IOMMU | ARM SMMU |
|---------|--------------|----------|
| **Page Table Format** | Sv39/Sv48/Sv57 | LPAE (Long descriptor) |
| **Device Identification** | Device ID (configurable) | Stream ID |
| **Command Interface** | Command queue | CMDQ (Command Queue) |
| **Fault Reporting** | Fault queue | Event queue |
| **Virtualization** | Two-stage translation | Stage 1 + Stage 2 |
| **Complexity** | Simpler, modular | More complex, feature-rich |

RISC-V IOMMU emphasizes simplicity and reuse of existing CPU MMU concepts, while ARM SMMU has evolved through multiple generations with extensive features.

---

## 13.3 Platform-Level Interrupt Integration

**Interrupt Routing in SoCs**

A typical RISC-V SoC has dozens of interrupt sources: UARTs, timers, GPIOs, network controllers, storage devices. Each device needs to signal the CPU when it requires attention. The Platform-Level Interrupt Controller (PLIC) manages this complexity by:

- Collecting interrupts from all devices
- Routing interrupts to appropriate cores
- Managing interrupt priorities
- Providing claim/complete mechanism

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

Key concepts:

*Interrupt Source*: A device that can generate interrupts (numbered 1-1023, source 0 is reserved)

*Interrupt Gateway*: Converts device interrupt signal to PLIC internal format

*Interrupt Target*: A CPU context (M-mode or S-mode on each core)

*Priority*: Each source has a priority (0 = never interrupt, 1-7 = increasing priority)

*Threshold*: Each target has a threshold (only interrupts with priority > threshold are delivered)

**PLIC Memory Map**

The PLIC is accessed through memory-mapped registers:

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

1. **Device asserts interrupt**: Device raises interrupt line
2. **PLIC gateway captures**: Gateway sets pending bit
3. **PLIC arbitration**: PLIC selects highest priority pending interrupt for each target
4. **CPU notification**: PLIC asserts external interrupt to CPU
5. **Software claim**: Interrupt handler reads claim register (returns source ID, clears pending)
6. **Software handling**: Handler services the device
7. **Software complete**: Handler writes source ID to complete register

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

In a multi-core system, each core has separate M-mode and S-mode contexts. The PLIC can route interrupts to specific cores:

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

RISC-V uses memory-mapped I/O: devices appear as memory locations. Reading from a device register uses the same load instruction as reading from RAM. Writing to a device register uses the same store instruction. This unifies the programming model—no special I/O instructions needed.

```assembly
# Read UART status register
li   a0, 0x10000000      # UART base address
lw   a1, 0(a0)           # Read status register

# Write to UART data register
li   a2, 'A'             # Character to send
sw   a2, 4(a0)           # Write to data register
```

**MMIO Address Regions**

A typical RISC-V SoC memory map divides address space into regions:

```
0x00000000 - 0x0FFFFFFF: Debug/Boot ROM
0x10000000 - 0x1FFFFFFF: Peripherals (UART, SPI, GPIO, etc.)
0x20000000 - 0x2FFFFFFF: PLIC
0x30000000 - 0x3FFFFFFF: Reserved
0x40000000 - 0x7FFFFFFF: More peripherals
0x80000000 - 0xFFFFFFFF: DRAM
```

Each peripheral gets a block of addresses for its registers:

```
UART0: 0x10000000 - 0x10000FFF
  0x10000000: Status register
  0x10000004: Data register
  0x10000008: Control register
  0x1000000C: Baud rate register
```

**MMIO Ordering Requirements**

MMIO accesses have ordering requirements that differ from normal memory:

- **Device registers may have side effects**: Reading a status register might clear an interrupt flag
- **Write order matters**: Writing control registers in wrong order can cause device malfunction
- **Read/write dependencies**: A write must complete before a subsequent read sees the result

RISC-V provides fence instructions to enforce ordering:

```assembly
# Ensure MMIO write completes before continuing
li   a0, 0x10000000
li   a1, 0x1
sw   a1, 0(a0)           # Write to control register
fence iorw, iorw         # Ensure write completes
lw   a2, 4(a0)           # Read status register
```

The FENCE instruction takes two operands specifying predecessor and successor operations:

- **i**: Device input (MMIO read)
- **o**: Device output (MMIO write)
- **r**: Memory read
- **w**: Memory write

Common patterns:

- `fence iorw, iorw`: Full fence (all operations)
- `fence ow, ow`: Ensure MMIO writes complete in order
- `fence ir, ir`: Ensure MMIO reads complete in order

**Uncached vs Cached MMIO**

MMIO regions must be marked as uncached in page tables. Caching device registers would cause:

- **Stale data**: Cache might return old value instead of current device state
- **Lost writes**: Write to cached location might not reach device
- **Side effect loss**: Reading cached value doesn't trigger device side effects

Page table entries for MMIO use special attributes:

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

A complete RISC-V SoC memory map includes ROM, RAM, peripherals, and reserved regions. Here's a typical layout for a 32-bit SoC:

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

The SoC interconnect decodes addresses and routes requests to appropriate components:

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

Different RISC-V platforms use different memory maps:

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

A modern SoC has multiple masters (CPU cores, DMA controllers, GPUs) and multiple slaves (memory, peripherals, accelerators). An interconnect fabric connects these components, handling:

- **Address routing**: Directing requests to correct destination
- **Arbitration**: Managing concurrent accesses
- **Data width conversion**: Connecting 32-bit devices to 64-bit buses
- **Clock domain crossing**: Bridging different clock frequencies

**AXI (Advanced eXtensible Interface)**

ARM's AMBA AXI is widely used in RISC-V SoCs due to its maturity and IP availability. AXI4 provides:

- **Separate read/write channels**: Independent read and write transactions
- **Burst transfers**: Efficient multi-beat transfers
- **Out-of-order completion**: Transactions can complete in any order
- **Quality of Service (QoS)**: Priority-based arbitration

AXI signals:

```
Write Address Channel: AWADDR, AWLEN, AWSIZE, AWVALID, AWREADY
Write Data Channel:    WDATA, WSTRB, WLAST, WVALID, WREADY
Write Response:        BRESP, BVALID, BREADY
Read Address Channel:  ARADDR, ARLEN, ARSIZE, ARVALID, ARREADY
Read Data Channel:     RDATA, RRESP, RLAST, RVALID, RREADY
```

**AHB (Advanced High-performance Bus)**

AHB is simpler than AXI, suitable for lower-performance peripherals:

- **Single channel**: Address and data share the same channel
- **Pipelined**: Two-stage pipeline (address, data)
- **Simpler protocol**: Easier to implement
- **Lower performance**: No out-of-order, limited bursts

**TileLink**

TileLink is a RISC-V-native interconnect developed at UC Berkeley:

- **Designed for RISC-V**: Matches RISC-V memory model
- **Scalable**: From simple embedded to complex multi-core
- **Cache coherence**: Built-in support for coherent caches
- **Three conformance levels**:
  - **TL-UL**: Uncached Lightweight (simple peripherals)
  - **TL-UH**: Uncached Heavyweight (DMA, accelerators)
  - **TL-C**: Cached (coherent caches)

TileLink advantages for RISC-V:

- Native support for RISC-V atomics (LR/SC, AMO)
- Efficient cache coherence protocol
- Open specification (no licensing)

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

- **AXI**: Best for high-performance SoCs, extensive IP ecosystem, industry standard
- **AHB**: Best for simple embedded systems, low-cost peripherals
- **TileLink**: Best for RISC-V-native designs, cache coherence, open ecosystem

---

## 13.7 DMA and Coherency

**DMA Controller Integration**

A DMA controller transfers data between memory and peripherals without CPU intervention. This frees the CPU for other tasks while large data transfers proceed in the background.

Typical DMA use cases:

- **Disk I/O**: Transfer data between storage and memory
- **Network I/O**: Move packets between NIC and memory
- **Audio/Video**: Stream data to/from media devices
- **Memory-to-memory**: Fast memory copy operations

DMA controller architecture:

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

DMA creates coherency problems when the CPU has caches:

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

**Solutions**:

1. **Software cache management** (simple, lower performance):

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

1. **Hardware cache coherence** (complex, higher performance):

- DMA controller participates in cache coherence protocol
- DMA snoops CPU caches or uses coherent interconnect
- Requires coherent interconnect (ACE for AXI, TL-C for TileLink)

**DMA and Virtual Memory**

DMA controllers typically use physical addresses, but software uses virtual addresses. This creates challenges:

**Problem**: Virtual address buffer might span non-contiguous physical pages

```
Virtual:  [0x1000-0x2FFF] (8 KB contiguous)
Physical: [0x80000000-0x80000FFF] + [0x85000000-0x85000FFF] (non-contiguous!)
```

**Solutions**:

1. **Scatter-Gather DMA**: DMA controller accepts list of physical address/length pairs

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

1. **IOMMU**: Translate device addresses to physical addresses (see Section 13.2)

2. **Physically contiguous buffers**: Allocate DMA buffers from reserved physical memory

---

## 🛠️ Hands-on Lab: Lab 13.1 — Memory Firewall (PMP Shield)

This lab demonstrates PMP's core functionality: setting protection rules in M-mode, then switching to S-mode to attempt a violation.

### Lab Objectives

1. Configure PMP Entry 0 as **Read-Only (R=1, W=0)** to protect target variable
2. Configure PMP Entry 1 as **Allow-All (R=1, W=1, X=1)** to let other code run normally
3. Switch to S-mode and attempt a write to trigger Store Access Fault

### NAPOT Encoding Principle

NAPOT encoding can be abstract for beginners. The key formula is:

```text
pmpaddr = (base_addr >> 2) | ((size >> 3) - 1)

Example: Protect a 4KB region starting at 0x80200000
- Base = 0x80200000, Size = 4KB (0x1000)
- 0x80200000 >> 2 = 0x20080000
- (0x1000 >> 3) - 1 = 0x1FF
- pmpaddr = 0x20080000 | 0x1FF = 0x200801FF
```

**Encoding Rules**:

| pmpaddr low bits | Corresponding region size |
|-----------------|--------------------------|
| `...aaaaa0` | 8 bytes |
| `...aaaa01` | 16 bytes |
| `...aaa011` | 32 bytes |
| `...a01111` | 128 bytes |
| `...0111111111` | 4KB (what we use) |

### Code (pmp_lab.S)

```assembly
.section .text
.global _start

_start:
    # ---------------------------------------------------
    # 1. Set up Trap Handler (catch Access Fault later)
    # ---------------------------------------------------
    la t0, trap_handler
    csrw mtvec, t0

    # ---------------------------------------------------
    # 2. Configure PMP (in M-mode)
    # ---------------------------------------------------

    # [Target] Protect a 4KB region at 0x80200000
    # Using NAPOT mode
    li t0, 0x200801FF
    csrw pmpaddr0, t0

    # Entry 0: Enable + NAPOT + Read Only (R=1, W=0, X=0)
    # PMP_R(1) | PMP_A_NAPOT(0x18) = 0x19

    # Entry 1: Open other memory (Allow All)
    # pmpaddr1 set to all 1s (max address), mode set to TOR
    # PMP_R(1) | PMP_W(1) | PMP_X(1) | PMP_A_TOR(0x08) = 0x0F

    # pmpcfg0 = (pmp1cfg << 8) | pmp0cfg = 0x0F19
    li t0, -1
    csrw pmpaddr1, t0

    li t0, 0x0F19
    csrw pmpcfg0, t0        # Firewall activated!

    # ---------------------------------------------------
    # 3. Drop to S-mode
    # ---------------------------------------------------

    # Set mstatus.MPP = 01 (Supervisor)
    li t0, (3 << 11)
    csrc mstatus, t0        # Clear MPP
    li t0, (1 << 11)
    csrs mstatus, t0        # Set MPP to 01 (S-mode)

    la t0, s_mode_entry
    csrw mepc, t0
    mret                    # Jump! Identity becomes Supervisor

s_mode_entry:
    # ---------------------------------------------------
    # 4. Trigger Attack (S-mode Attempt)
    # ---------------------------------------------------
    li a0, 0x80200000       # Address protected by PMP0
    li t1, 0xDEADBEEF

    # Attempt write! Should trigger Exception 7 (Store Access Fault)
    sw t1, 0(a0)

    # If we survive, experiment failed
    li a0, 0
    j stop

stop:
    j stop

trap_handler:
    # Read mcause to check exception type
    csrr t0, mcause

    # Exception 7 = Store Access Fault
    li t1, 7
    bne t0, t1, unexpected

    # SUCCESS: PMP blocked the illegal write!
    li a0, 1                # Return success code
    j stop

unexpected:
    li a0, -1               # Unexpected exception
    j stop
```

### Compile and Run

```bash
# Assemble
riscv64-unknown-elf-as -march=rv64g -o pmp_lab.o pmp_lab.S

# Link (ensure _start is entry point)
riscv64-unknown-elf-ld -T link.ld -o pmp_lab.elf pmp_lab.o

# Run on QEMU
qemu-system-riscv64 -machine virt -nographic -bios pmp_lab.elf
```

### Expected Behavior

1. **M-mode**: PMP entries configured, firewall activated
2. **mret**: Privilege drops to S-mode
3. **sw instruction**: Triggers Store Access Fault (Exception 7)
4. **Trap handler**: Confirms PMP did its job

> **danieRTOS Reference**: A real RTOS would use PMP to isolate kernel data from user tasks, preventing task corruption.

---

## ⚠️ Common Pitfalls

### Pitfall 1: PMP Priority Order Error

**Error Scenario**: Put "deny rule" in pmp15, put "allow rule" in pmp0.

**Consequence**: pmp0 matches first, allowing all access. The deny rule never takes effect.

```c
// ❌ Wrong: Order reversed
pmp0: Allow All (RWX)      // Matches first, permits everything
pmp1: Deny 0x80200000      // Never gets checked

// ✅ Correct: Write specific Deny first, generic Allow last
pmp0: Read-Only 0x80200000 // Check sensitive region first
pmp1: Allow All            // Allow other regions
```

> 💡 **Memory aid**: Like firewall rules, **write exceptions first, default last**.

### Pitfall 2: Forgetting the Default Deny Rule

**Error Scenario**: Only set one PMP Entry to protect the key area, forgot to open other memory.

**Consequence**: Code region not matched by any PMP Entry, S/U-mode can't even fetch the next instruction.

```assembly
# ❌ Wrong: Only one Entry
csrw pmpaddr0, t0       # Protect key
csrw pmpcfg0, 0x19      # Read-Only
mret                    # Jump to S-mode then immediately Crash!

# ✅ Correct: Add Allow All Entry
csrw pmpaddr0, t0       # Protect key
csrw pmpaddr1, t1       # Max address
csrw pmpcfg0, 0x0F19    # pmp0=RO, pmp1=RWX
```

### Pitfall 3: Lock Bit Irreversibility

**Error Scenario**: Setting Lock bit (L=1) during development.

**Consequence**: PMP Entry locked. Only hardware Reset can unlock. Cannot modify rules during debug.

```c
// ❌ Dangerous: Setting Lock during development
pmpcfg0 = 0x99;  // L=1, A=NAPOT, R=1

// ✅ Recommended: Only set Lock in Production
#ifdef PRODUCTION
    pmpcfg0 = 0x99;  // Locked
#else
    pmpcfg0 = 0x19;  // Dev mode, not locked
#endif
```

> 💡 **Reminder**: Lock bit is meant to prevent malicious modification of M-mode Firmware. Don't use it during development.

---

## Summary

SoC integration connects RISC-V cores with the rest of the system. This chapter covered seven essential components that make a complete RISC-V system-on-chip.

**Physical Memory Protection (PMP)** provides hardware-enforced memory access control using physical addresses. PMP operates in M-mode and protects memory regions from untrusted code. With up to 64 configurable regions, four address matching modes (OFF, TOR, NA4, NAPOT), and lockable entries, PMP enables firmware protection, device memory isolation, and task separation in systems without MMUs.

**IOMMU** extends memory protection to devices by translating device addresses and enforcing access control. The RISC-V IOMMU uses the same page table format as the CPU MMU, simplifying software implementation. This enables device isolation, safe device passthrough for virtualization, and protection against malicious or buggy devices.

**Platform-Level Interrupt Controller (PLIC)** manages interrupt routing in multi-core systems. The PLIC collects interrupts from up to 1023 sources, arbitrates by priority, and routes them to appropriate CPU contexts. The claim/complete mechanism ensures atomic interrupt handling, while per-context enable masks and thresholds provide flexible interrupt management.

**Memory-Mapped I/O (MMIO)** provides a uniform mechanism for device access using standard load and store instructions. MMIO regions must be marked uncached in page tables, and fence instructions ensure proper ordering of device accesses. This unified address space simplifies the programming model compared to architectures with separate I/O instructions.

**SoC memory maps** organize address space into regions for ROM, RAM, peripherals, and reserved areas. Different RISC-V platforms use different layouts, but all follow the principle of address decode and routing through the interconnect fabric. Understanding the memory map is essential for firmware development and device driver programming.

**System interconnects** connect multiple masters and slaves in the SoC. AXI provides high performance with extensive IP ecosystem, AHB offers simplicity for embedded systems, and TileLink provides RISC-V-native features including cache coherence. The choice depends on performance requirements, IP availability, and coherence needs.

**DMA and coherency** enable efficient data transfers but require careful management of cache coherence. Software can use cache flush and invalidate operations, or hardware can provide coherent DMA through snooping or coherent interconnects. IOMMU or scatter-gather DMA solves the virtual-to-physical address translation problem for DMA transfers.

Together, these components form the foundation of RISC-V SoC design, enabling everything from simple microcontrollers to complex multi-core application processors.
