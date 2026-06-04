# Chapter 14. RISC-V Platform Profiles & Embedded Systems

**Part VIII — System Design, Platform Spec & SoC Integration**

---

## 🎯 Learning Objectives

After reading this chapter, you will be able to:

1. **Understand the Fragmentation Problem**: Grasp how ISA Fragmentation harms software ecosystems
2. **Distinguish Profile Types**: Understand the use-case differences between RVA (Application) and RVM (Microcontroller)
3. **Decode Profile Names**: Interpret the meaning of names like RVA22S, RVM23

---

## 💡 Scenario: À La Carte or Set Menu?

> **Scene**: Junior has multiple chip vendor spec sheets spread across the desk, looking bewildered.

**Junior**: "Senior, I'm going cross-eyed looking at these spec sheets. This vendor says they support `RV64GC`, that one says `RVA22`, and another says `RVM23`. I just want to run Linux—which one do I pick? Can't it be as simple as buying an x86 computer?"

**Senior**: "That's the downside of RISC-V being 'too free'—**Fragmentation**. Before, everyone assembled their own instruction sets: you want M, he wants F, I want C. Then you write software only to find that this CPU is missing an instruction, instant Crash."

**Junior**: "So Profiles are meant to solve this?"

**Senior**: "Exactly. Think of it as **officially certified set menus**:

| Before (À la carte) | Now (Profile) |
|---------------------|---------------|
| Pick your own instruction set | Official pre-configured menu |
| Forgot to order FPU = Linux won't boot | RVA22 label = guaranteed to run |
| Verify compatibility for each product | One ISO runs all compliant hardware |

**RVA (Application Profile)** is the 'deluxe menu' for rich operating systems like Linux/Android. If a vendor dares to slap the RVA22 label on their chip, they guarantee it has MMU, atomic instructions, floating-point, and everything else needed to run Linux."

**Junior**: "What about RVM?"

**Senior**: "**RVM (Microcontroller Profile)** is the 'economy menu' for RTOS or bare-metal. It drops heavy equipment like MMU, focusing on low power and real-time control. If you're just making a smart rice cooker, RVM is enough; but if you're making a smartphone, you definitely need RVA."

**Junior**: "What about the numbers 20, 22, 23? Performance levels?"

**Senior**: "Not performance—**year**. Like a '2022 model year' car. RVA22 represents standards defined in 2022, RVA23 adds some new features (like stronger vector instructions). Newer Linux distributions typically require newer Profile years."

**Junior**: "Got it! So when choosing a CPU, I don't need to check instructions one by one anymore—just confirm it meets the Profile menu I need!"

---

RISC-V's modularity is both a strength and a challenge. The base ISA is minimal, and implementations choose which extensions to include. This flexibility enables optimization for specific use cases—a microcontroller might include only RV32IMC, while a server processor might implement RV64IMAFDCV. But this variability creates a problem: how does software know what features are available?

Platform profiles solve this by defining standard combinations of extensions for specific use cases. A profile specifies mandatory extensions, optional extensions, and implementation requirements. Software targeting a profile can assume certain features exist, simplifying development and improving portability. The RVA22 profile defines requirements for application processors running rich operating systems. The RVA23 profile adds newer extensions and stricter requirements. Embedded profiles target microcontrollers and real-time systems.

This chapter explores RISC-V platform profiles and embedded system design. We'll examine the RVA22 and RVA23 profiles for application processors, embedded profiles for microcontrollers, the differences between MMU and no-MMU systems, and how RISC-V compares to ARM Cortex-M in the embedded space.

---

## 14.1 Platform Profiles Overview

**The Fragmentation Problem**

RISC-V's modularity creates potential fragmentation. Consider these valid RISC-V implementations:

- RV32I: Minimal 32-bit core (base ISA only)
- RV32IMC: Embedded core (multiply, compressed)
- RV64IMAFDC: Application processor (full general-purpose)
- RV64IMAFDCV: High-performance with vectors

Software written for RV64IMAFDC won't run on RV32IMC. Even within the same base (RV64), different extension combinations create incompatibilities. This makes it difficult to distribute binary software or develop portable operating systems.

**Profiles as a Solution**

A platform profile defines:

- **Mandatory extensions**: Must be implemented
- **Optional extensions**: May be implemented
- **Mandatory features**: Specific implementation requirements (e.g., minimum PMP entries)
- **Discovery mechanism**: How software detects features

Profiles create standard targets for software development. An OS can target "RVA22 profile" and know exactly what features are available. Hardware vendors can claim "RVA22 compliant" and guarantee compatibility with RVA22 software.

**Profile Naming Convention**

RISC-V profiles use a naming scheme:

- **RV**: RISC-V
- **A/M/E**: Application / Microcontroller / Embedded
- **22/23/...**: Year of ratification (2022, 2023, etc.)
- **S/U**: Supervisor mode / User mode (for application profiles)

Examples:

- **RVA22S**: Application profile, 2022, Supervisor mode
- **RVA22U**: Application profile, 2022, User mode
- **RVA23S**: Application profile, 2023, Supervisor mode
- **RVM23**: Microcontroller profile, 2023

**Profile Versioning**

Profiles evolve over time:

- **Major version**: Incompatible changes (e.g., RVA22 → RVA23)
- **Minor version**: Backward-compatible additions
- **Errata**: Bug fixes, no functional changes

A profile version guarantees:

- **Forward compatibility**: RVA22 software runs on RVA23 hardware
- **Feature stability**: Mandatory extensions don't change within a version

---

## 14.2 RVA22 Profile (Application Processor)

**Target Use Case**

RVA22 targets application processors running rich operating systems like Linux, FreeBSD, or commercial RTOSes. These systems need:

- Virtual memory (MMU)
- Privilege levels (M, S, U modes)
- Standard extensions for general-purpose computing
- Sufficient performance for application workloads

**RVA22S (Supervisor Mode)**

RVA22S defines requirements for systems running supervisor-mode operating systems.

**Mandatory ISA Extensions**:

- **RV64I**: 64-bit base integer ISA
- **M**: Integer multiplication and division
- **A**: Atomic instructions
- **F**: Single-precision floating-point
- **D**: Double-precision floating-point
- **C**: Compressed instructions
- **Zicsr**: CSR instructions
- **Zifencei**: Instruction fence
- **Zicntr**: Base counters (cycle, time, instret)
- **Zihpm**: Hardware performance counters
- **Ziccif**: Main memory supports instruction fetch
- **Ziccrse**: Main memory supports misaligned loads/stores
- **Ziccamoa**: Main memory supports all atomics
- **Zicclsm**: Main memory supports misaligned atomics
- **Za64rs**: Reservation set size (64 bytes)
- **Zihintpause**: Pause hint instruction
- **Zba**: Address generation (bit manipulation)
- **Zbb**: Basic bit manipulation
- **Zbs**: Single-bit instructions
- **Zkt**: Data-independent execution time (timing side-channel protection)

**Mandatory Privileged Features**:

- **Sv39**: Page-based virtual memory (39-bit virtual address)
- **Svpbmt**: Page-based memory types
- **Svadu**: Hardware A/D bit updates
- **Sstc**: Supervisor-mode timer interrupts
- **Sscofpmf**: Count overflow and privilege mode filtering

**Mandatory Implementation Requirements**:

- At least 8 PMP entries
- At least 29 hardware performance counters
- Misaligned loads/stores supported in main memory
- LR/SC reservation set size of 64 bytes

**RVA22U (User Mode)**

RVA22U defines requirements for user-mode applications. It's a subset of RVA22S:

- Same ISA extensions as RVA22S
- No privileged features (no Sv39, no PMP requirements)
- Targets user-space applications running on RVA22S systems

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

RVA23, ratified in 2023, builds on RVA22 with additional extensions and stricter requirements:

**New Mandatory Extensions**:

- **Zicond**: Integer conditional operations
- **Zimop**: May-be-operations (reserved for future extensions)
- **Zcmop**: Compressed may-be-operations
- **Zcb**: Additional compressed instructions
- **Zfa**: Additional floating-point instructions
- **Zawrs**: Wait-on-reservation-set instructions
- **Supm**: Pointer masking (supervisor mode)

**Enhanced Requirements**:

- Minimum 16 PMP entries (up from 8)
- Sv48 or Sv57 support (in addition to Sv39)
- Improved performance counter requirements
- Stricter timing guarantees for Zkt

**Forward Compatibility**

RVA23 is forward-compatible with RVA22:

- All RVA22 mandatory extensions remain mandatory in RVA23
- RVA22 software runs unmodified on RVA23 hardware
- RVA23 adds features but doesn't remove or change existing ones

**Migration Path**

Hardware vendors can support both profiles:

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

Embedded systems have different constraints than application processors:

- **Limited resources**: Small memory, low power, cost-sensitive
- **Real-time requirements**: Deterministic interrupt latency, predictable timing
- **No virtual memory**: Many embedded systems run without MMU
- **Simpler software**: Bare-metal or RTOS, not full OS

RISC-V embedded profiles target these use cases with minimal mandatory extensions and flexible configurations.

**RV32E Base ISA**

RV32E is a reduced version of RV32I with only 16 registers (x0-x15) instead of 32. This saves:

- **Silicon area**: Smaller register file
- **Power**: Fewer registers to manage
- **Code size**: Shorter register encodings in some cases

RV32E is suitable for ultra-low-cost microcontrollers where every gate counts.

```assembly
# RV32E example: Only x0-x15 available
addi x10, x0, 42     # OK: x10 is in range
addi x20, x0, 42     # ERROR: x20 doesn't exist in RV32E
```

**Microcontroller-Oriented Features**

Embedded profiles emphasize:

- **Compressed instructions (C)**: Reduce code size by 25-30%
- **Multiply (M)**: Essential for many embedded algorithms
- **Bit manipulation (B)**: Efficient for embedded control tasks
- **Fast interrupts**: Low-latency interrupt handling

**Interrupt Handling for Embedded**

Embedded systems need fast, deterministic interrupt response. RISC-V provides two interrupt architectures:

1. **CLINT (Core-Local Interruptor)**: Basic timer and software interrupts
2. **CLIC (Core-Local Interrupt Controller)**: Advanced interrupt controller for embedded

CLIC features:

- **Vectored interrupts**: Jump directly to handler (no dispatch overhead)
- **Nested interrupts**: Higher-priority interrupts preempt lower-priority
- **Tail-chaining**: Back-to-back interrupts without full context save/restore
- **Configurable levels**: Up to 256 priority levels

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

Embedded systems prioritize power efficiency:

- **Clock gating**: Disable unused modules
- **Power domains**: Shut down inactive regions
- **Sleep modes**: WFI (Wait For Interrupt) instruction
- **Dynamic voltage/frequency scaling**: Adjust performance vs power

```assembly
# Enter low-power mode
wfi              # Wait for interrupt (CPU sleeps)
# CPU wakes on interrupt, resumes here
```

---

## 14.5 MMU vs No-MMU Systems

**MMU-Based Systems**

Systems with Memory Management Units (MMUs) provide:

- **Virtual memory**: Each process has its own address space
- **Memory protection**: Processes can't access each other's memory
- **Demand paging**: Load pages from disk on demand
- **Large address spaces**: 64-bit virtual addresses

MMU-based systems run full operating systems like Linux:

```
Process A: 0x00000000-0xFFFFFFFF (virtual)
    ↓ MMU translation
Physical: 0x80000000-0x80FFFFFF

Process B: 0x00000000-0xFFFFFFFF (virtual)
    ↓ MMU translation
Physical: 0x81000000-0x81FFFFFF
```

Requirements:

- Sv39/Sv48/Sv57 page tables
- TLB for translation caching
- Page fault handling
- Sufficient memory for page tables

**No-MMU Systems**

Systems without MMUs use physical addresses directly:

- **Simpler hardware**: No TLB, no page table walker
- **Lower cost**: Fewer gates, less power
- **Faster context switch**: No TLB flush needed
- **Deterministic**: No TLB miss latency

No-MMU systems run:

- **Bare-metal**: Single application, no OS
- **RTOS**: Real-time OS with static memory allocation
- **Embedded Linux**: uClinux (Linux without MMU)

**Memory Protection Without MMU**

No-MMU systems can still provide memory protection using PMP (Physical Memory Protection):

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

PMP provides:

- Region-based protection (not page-based)
- M-mode enforcement
- Locked regions (can't be changed until reset)

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

ARM Cortex-M is the dominant architecture for microcontrollers. The Cortex-M family includes:

- **Cortex-M0/M0+**: Ultra-low-cost, minimal features
- **Cortex-M3**: Mainstream, good performance/cost
- **Cortex-M4**: DSP extensions, floating-point
- **Cortex-M7**: High performance, cache, double-precision FP
- **Cortex-M33/M55**: ARMv8-M, TrustZone, vector extensions

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

ARM NVIC (Nested Vectored Interrupt Controller):

- Vectored interrupts (direct jump to handler)
- Nested interrupts with priority levels
- Tail-chaining for back-to-back interrupts
- Automatic context save/restore

RISC-V CLIC (Core-Local Interrupt Controller):

- Similar features to NVIC
- More flexible priority levels (up to 256)
- Configurable interrupt modes
- Compatible with RISC-V privilege model

Both provide comparable functionality for embedded interrupt handling.

**Ecosystem Comparison**

**ARM Cortex-M Advantages**:

- Mature ecosystem (20+ years)
- Extensive vendor support (ST, NXP, TI, etc.)
- Rich middleware (CMSIS, Mbed, etc.)
- Large developer community
- Proven in billions of devices

**RISC-V Embedded Advantages**:

- No licensing fees (lower cost)
- Open ISA (customizable)
- Modern design (cleaner than ARM legacy)
- Growing ecosystem (SiFive, Espressif, etc.)
- Future-proof (community-driven evolution)

**Use Case Recommendations**

**Choose ARM Cortex-M when**:

- Mature ecosystem is critical
- Extensive middleware needed
- Proven reliability required
- Time-to-market is tight

**Choose RISC-V Embedded when**:

- Cost optimization is important
- Customization is needed
- Open-source ecosystem preferred
- Long-term flexibility valued

**Example: ESP32-C3 (RISC-V) vs ESP32 (Xtensa)**

Espressif's ESP32-C3 demonstrates RISC-V in embedded:

- RV32IMC core (32-bit, multiply, compressed)
- 160 MHz, single-core
- Wi-Fi + Bluetooth LE
- 400 KB SRAM, 4 MB flash
- Arduino, ESP-IDF support

Compared to ESP32 (Xtensa):

- Similar performance
- Better toolchain (GCC, LLVM)
- Open ISA (vs proprietary Xtensa)
- Growing ecosystem

---

## 🛠️ Hands-on Lab: Lab 14.1 — Profile Detector

This lab demonstrates how to detect which Extensions the current hardware supports—key to understanding Profile practical applications.

### Lab Objectives

1. Read the `misa` CSR to see supported Extensions
2. Check if critical CSRs exist
3. Determine which Profile level the current system meets

### Code (profile_detect.c)

```c
#include <stdio.h>
#include <stdint.h>

// Read misa CSR (requires M-mode privilege)
static inline uint64_t read_misa() {
    uint64_t val;
    asm volatile ("csrr %0, misa" : "=r" (val));
    return val;
}

// Extension check (misa bit mapping: A=0, B=1, ..., Z=25)
#define HAS_EXT(misa, letter) ((misa) & (1UL << ((letter) - 'A')))

void check_profile(uint64_t misa) {
    printf("=== RISC-V Profile Detector ===\n\n");

    // Check XLEN (MXL field in misa[63:62])
    int xlen = (misa >> 62) == 2 ? 64 : 32;
    printf("Base: RV%d\n", xlen);

    // List supported Extensions
    printf("Extensions: ");
    for (char c = 'A'; c <= 'Z'; c++) {
        if (HAS_EXT(misa, c)) {
            printf("%c", c);
        }
    }
    printf("\n\n");

    // RVA22 minimum requirements check
    int has_m = HAS_EXT(misa, 'M');  // Integer Multiply
    int has_a = HAS_EXT(misa, 'A');  // Atomics
    int has_f = HAS_EXT(misa, 'F');  // Single-precision Float
    int has_d = HAS_EXT(misa, 'D');  // Double-precision Float
    int has_c = HAS_EXT(misa, 'C');  // Compressed

    printf("--- Profile Compatibility ---\n");

    // RVA22 requires: RV64IMAFDC + more
    if (xlen == 64 && has_m && has_a && has_f && has_d && has_c) {
        printf("[✓] RVA22 basic requirements: PASS\n");
        printf("    (Additional checks needed: Zba, Zbb, Zbs, Sv39, PMP>=8)\n");
    } else {
        printf("[✗] RVA22 basic requirements: FAIL\n");
        if (xlen != 64) printf("    Missing: 64-bit base\n");
        if (!has_m) printf("    Missing: M (Multiply)\n");
        if (!has_a) printf("    Missing: A (Atomics)\n");
        if (!has_f) printf("    Missing: F (Float)\n");
        if (!has_d) printf("    Missing: D (Double)\n");
        if (!has_c) printf("    Missing: C (Compressed)\n");
    }

    // RVM (Microcontroller) compatibility is more relaxed
    if (has_m && has_c) {
        printf("[✓] RVM basic requirements: PASS (RV32/64 + M + C)\n");
    }
}

int main() {
    uint64_t misa = read_misa();

    if (misa == 0) {
        printf("Error: Cannot read misa (not in M-mode?)\n");
        return 1;
    }

    check_profile(misa);
    return 0;
}
```

### Compile and Run

```bash
# Compile (requires M-mode execution environment)
riscv64-unknown-elf-gcc -march=rv64gc -o profile_detect profile_detect.c

# Run on Spike (M-mode simulation)
spike pk profile_detect
```

### Expected Output (QEMU virt machine)

```text
=== RISC-V Profile Detector ===

Base: RV64
Extensions: ACDFIMSU

--- Profile Compatibility ---
[✓] RVA22 basic requirements: PASS
    (Additional checks needed: Zba, Zbb, Zbs, Sv39, PMP>=8)
[✓] RVM basic requirements: PASS (RV32/64 + M + C)
```

### Key Takeaways

1. **misa CSR**: Single read reveals all standard extensions
2. **Profile != Full Compliance**: Even if IMAFDC is present, RVA22 still needs additional features like Zba, Zbb, Zbs, Sv39, and 8+ PMP entries
3. **Runtime Detection**: Production code should check features at boot, not assume

> **danieRTOS Reference**: danieRTOS checks for M extension at startup to decide whether to use hardware or software multiply.

---

## ⚠️ Common Pitfalls

### Pitfall 1: Profile Version ≠ Performance

**Misconception**: "RVA23 is 10% faster than RVA22"

**Truth**: Profile version only represents the **feature set's year**, not clock speed or hardware performance.

```text
RVA22 @ 2.0 GHz could be much faster than RVA23 @ 1.0 GHz!

Profiles define "software compatibility", not "hardware performance".
```

### Pitfall 2: Thinking RV64G = RVA22

**Error Scenario**: Seeing vendor claim `RV64GC` support and assuming latest Fedora will run.

**Truth**: RVA22 requires additional Extensions and hardware features:

| Requirement | RV64GC | RVA22 |
|-------------|--------|-------|
| IMAFDCSU | ✓ | ✓ |
| Zba, Zbb, Zbs | ✗ | ✓ (Mandatory) |
| Sv39 (MMU) | Not specified | ✓ (Mandatory) |
| PMP ≥ 8 entries | Not specified | ✓ (Mandatory) |

### Pitfall 3: Ignoring Profile Backward Compatibility

**Error Scenario**: Programs compiled for RVA23 won't run on RVA22 hardware.

**Correct Understanding**:

```text
RVA22 software → RVA23 hardware  ✓ (Forward compatible)
RVA23 software → RVA22 hardware  ✗ (May use new instructions)
```

> 💡 **Recommendation**: For maximum compatibility, specify an older Profile as your compilation target.

---

## Summary

Platform profiles and embedded system design define how RISC-V adapts to different use cases. This chapter covered five key areas that enable RISC-V to serve both high-performance application processors and resource-constrained embedded systems.

**Platform profiles** solve the fragmentation problem by defining standard combinations of extensions. Profiles specify mandatory extensions, optional features, and implementation requirements. This creates standard targets for software development and guarantees compatibility. The naming convention (RVA22S, RVA23U, etc.) clearly indicates the profile's purpose and version.

**RVA22 profile** targets application processors running rich operating systems. RVA22S requires 64-bit base ISA, standard extensions (M, A, F, D, C), bit manipulation (Zba, Zbb, Zbs), Sv39 virtual memory, and at least 8 PMP entries. RVA22U provides the same ISA extensions for user-mode applications. This profile enables Linux, FreeBSD, and other full operating systems to run portably across RISC-V implementations.

**RVA23 profile** builds on RVA22 with additional extensions and stricter requirements. New mandatory extensions include Zicond (conditional operations), Zfa (additional floating-point), and Zawrs (wait-on-reservation-set). Enhanced requirements include 16 PMP entries and Sv48/Sv57 support. RVA23 maintains forward compatibility—RVA22 software runs unmodified on RVA23 hardware.

**Embedded profiles** target microcontrollers and real-time systems with different constraints. RV32E reduces the register file to 16 registers for ultra-low-cost applications. Embedded systems emphasize compressed instructions for code size, fast interrupt handling through CLIC, and low-power features like WFI and clock gating. These profiles enable RISC-V to compete in the microcontroller market.

**MMU vs no-MMU systems** represent two different approaches to memory management. MMU-based systems provide virtual memory, per-process address spaces, and page-based protection, enabling full operating systems like Linux. No-MMU systems use physical addresses directly, offering simpler hardware, lower cost, and better determinism. PMP provides region-based memory protection for no-MMU systems, enabling task isolation without the complexity of virtual memory.

**Comparison with ARM Cortex-M** shows RISC-V's competitive position in embedded systems. Both architectures provide similar features—vectored interrupts, nested interrupt handling, memory protection, and optional floating-point. RISC-V offers advantages in licensing (open, no fees), customization (modular ISA), and modern design. ARM Cortex-M leads in ecosystem maturity, vendor support, and proven deployment. The choice depends on priorities: cost and flexibility favor RISC-V, while ecosystem maturity favors ARM.

Together, platform profiles and embedded system features enable RISC-V to serve the full spectrum from tiny microcontrollers to powerful application processors, with clear standards for software portability and hardware compliance.
