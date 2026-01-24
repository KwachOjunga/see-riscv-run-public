# Table of Contents

## Front Matter

- Cover
- Copyright and License
- Preface
- Table of Contents

---

## Part I: Introduction

### Chapter 1: What Is RISC-V?

The RISC Revolution • Why RISC-V? The Open ISA Movement • RISC-V Design Philosophy • RISC-V ISA Overview • RISC-V Profiles • RISC-V vs ARM vs MIPS

---

## Part II: Programmer's Model

### Chapter 2: Programmer's Model & Register Set

General-Purpose Registers • Control and Status Registers (CSRs) • Program State and Privilege Levels • Calling Convention and ABI • Stack Frame Structure • Comparison with ARM64 and MIPS

### Chapter 3: Privilege Levels & Execution Environment

RISC-V Privilege Architecture • Machine Mode (M-mode) • Supervisor Mode (S-mode) • User Mode (U-mode) • Privilege Level Transitions • Execution Environment Interface

---

## Part III: Traps & Exceptions

### Chapter 4: Trap, Exception, Interrupt

Trap Handling Overview • Exception Types • Interrupt Types • Trap Delegation • Trap Vector Table • Trap Entry and Exit • Nested Traps • Comparison with ARM and MIPS

---

## Part IV: Memory & Addressing

### Chapter 5: Virtual Memory & Paging (Sv39/Sv48)

Virtual Memory Overview • Page Table Structure • Address Translation • TLB Management • Sv39 and Sv48 Modes • Comparison with ARM and x86

### Chapter 6: Memory Ordering & Synchronization

Memory Consistency Model • RISC-V Memory Model (RVWMO) • Fence Instructions • Atomic Operations • Load-Reserved/Store-Conditional • Comparison with ARM and x86

---

## Part V: Pipeline & Microarchitecture

### Chapter 7: RISC-V Pipeline Fundamentals

Classic Five-Stage Pipeline • Pipeline Hazards • Hazard Detection and Resolution • Branch Prediction • Pipeline Performance • Comparison with ARM and MIPS

### Chapter 8: Microarchitecture Variations

In-Order vs Out-of-Order • Superscalar Execution • Cache Hierarchy • Memory Subsystem • RISC-V Core Examples (Rocket, BOOM) • Performance Comparison

---

## Part VI: Booting & System Software

### Chapter 9: Reset, Boot Flow & Firmware

Reset and Initialization • Boot ROM • Bootloader Stages • Firmware Components • Device Tree • Boot Flow Examples • Comparison with ARM

### Chapter 10: Machine Mode, SBI & Supervisor Mode

Machine Mode Overview • Supervisor Binary Interface (SBI) • SBI Implementation (OpenSBI) • Supervisor Mode Software • OS Kernel Integration • Comparison with ARM

---

## Part VII: ISA Extensions

### Chapter 11: RISC-V Standard Extensions

Extension Naming Convention • M Extension (Multiply/Divide) • A Extension (Atomics) • F/D Extensions (Floating-Point) • C Extension (Compressed) • Zicsr and Zifencei • Other Standard Extensions

### Chapter 12: Vector Processing & SIMD Comparison

Vector Extension Overview • Vector Register File • Vector Instructions • Vector Length Agnostic Programming • Vector Memory Operations • Comparison with ARM SVE/SVE2 and x86 AVX

---

## Part VIII: Platform & SoC

### Chapter 13: SoC Integration

SoC Architecture Overview • Interrupt Controllers (PLIC, CLIC, AIA) • Memory-Mapped I/O • DMA and Bus Protocols • Power Management Integration

### Chapter 14: RISC-V Platform Profiles & Embedded Systems

Platform Specifications • RVA Profiles (RVA22, RVA23) • Embedded Profiles • Certification and Compliance

---

## Part IX: Debug & Performance

### Chapter 15: Debugging & Trace

Debug Architecture Overview • Debug Module • Trigger Module • Trace Architecture • JTAG and OpenOCD • GDB Integration

### Chapter 16: Performance Counters & PMU

Performance Monitoring Overview • Hardware Performance Counters • Event Selection • PMU Programming • Performance Analysis Tools

---

## Part X: Comparison

### Chapter 17: RISC-V vs ARM vs MIPS — A Systematic Comparison

Historical Context • ISA Design Philosophy • Instruction Encoding • Privilege Architecture • Memory Model • Ecosystem and Adoption • Future Outlook

---

## Appendices

**Appendix A**: CSR Reference
**Appendix B**: Extension Reference
**Appendix C**: Bootloader Reference
**Appendix D**: SBI Reference
**Appendix E**: RISC-V vs ARM Comparison
**Appendix F**: Memory Model Reference

---

## Back Matter

- About the Author
- Bibliography
