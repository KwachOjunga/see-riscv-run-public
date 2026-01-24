# Bibliography and References

This book is based on publicly available specifications and documentation. All information is derived from open-source materials and official RISC-V specifications.

---

## RISC-V Official Specifications

### ISA Specifications

1. **RISC-V Instruction Set Manual, Volume I: Unprivileged ISA**  
   RISC-V International  
   https://github.com/riscv/riscv-isa-manual  
   Latest version: Ratified 2019, with ongoing updates

2. **RISC-V Instruction Set Manual, Volume II: Privileged Architecture**  
   RISC-V International  
   https://github.com/riscv/riscv-isa-manual  
   Latest version: Ratified 2021, with ongoing updates

### Extension Specifications

1. **RISC-V "V" Vector Extension**  
   RISC-V International  
   https://github.com/riscv/riscv-v-spec  
   Version 1.0, Ratified 2021

2. **RISC-V Bit Manipulation Extension**  
   RISC-V International  
   https://github.com/riscv/riscv-bitmanip  
   Version 1.0, Ratified 2021

3. **RISC-V Cryptography Extensions**  
   RISC-V International  
   https://github.com/riscv/riscv-crypto  
   Version 1.0, Ratified 2021

4. **RISC-V Hypervisor Extension**  
   RISC-V International  
   Included in Privileged Architecture Specification

### Platform Specifications

1. **RISC-V Platform-Level Interrupt Controller (PLIC) Specification**  
   RISC-V International  
   https://github.com/riscv/riscv-plic-spec

2. **RISC-V Core-Local Interrupt Controller (CLIC) Specification**  
   RISC-V International  
   https://github.com/riscv/riscv-fast-interrupt

3. **RISC-V Supervisor Binary Interface (SBI) Specification**  
   RISC-V International  
   https://github.com/riscv-non-isa/riscv-sbi-doc  
   Version 1.0, Ratified 2020

4. **RISC-V ELF psABI Specification**  
    RISC-V International  
    https://github.com/riscv-non-isa/riscv-elf-psabi-doc

---

## RISC-V Software and Tools

1. **RISC-V GNU Compiler Toolchain**
    https://github.com/riscv-collab/riscv-gnu-toolchain

2. **RISC-V LLVM**
    https://github.com/llvm/llvm-project

3. **QEMU RISC-V Emulator**
    https://www.qemu.org/docs/master/system/target-riscv.html

4. **Spike RISC-V ISA Simulator**
    https://github.com/riscv-software-src/riscv-isa-sim

5. **OpenSBI (Open Source Supervisor Binary Interface)**
    https://github.com/riscv-software-src/opensbi

---

## Companion Projects (本書配套專案)

1. **danieRTOS - A Minimal RISC-V RTOS for Learning**  
   Danny Jiang  
   https://github.com/djiangtw/djiang-oss-public/tree/main/daniertos  
   一個專為學習 RISC-V 系統程式設計而設計的最小化 RTOS 實作。本書的 Lab 範例邏輯參考自此專案。

2. **Building danieRTOS - 技術專欄系列**  
   Danny Jiang  
   https://github.com/djiangtw/tech-column-public/tree/main/topics/building-daniertos  
   詳細記錄 danieRTOS 開發過程的技術文章系列，涵蓋 Context Switch、Interrupt Handling、Timer、Scheduler 等核心主題。

---

## Classic Architecture Books

1. **Sweetman, Dominic**. *See MIPS Run, Second Edition*.  
    Morgan Kaufmann, 2006.  
    ISBN: 978-0120884216

2. **Patterson, David A., and John L. Hennessy**. *Computer Organization and Design RISC-V Edition: The Hardware Software Interface*.  
    Morgan Kaufmann, 2017.  
    ISBN: 978-0128122754

3. **Waterman, Andrew, and Krste Asanović (Editors)**. *The RISC-V Reader: An Open Architecture Atlas*.  
    Strawberry Canyon, 2017.  
    ISBN: 978-0999249109

---

## ARM Architecture References

1. **ARM Architecture Reference Manual ARMv8, for ARMv8-A architecture profile**  
    ARM Limited  
    https://developer.arm.com/documentation/

2. **ARM Cortex-A Series Programmer's Guide**  
    ARM Limited  
    https://developer.arm.com/documentation/

---

## Memory Model and Concurrency

1. **RISC-V Memory Consistency Model**  
    Included in RISC-V Unprivileged ISA Specification, Chapter 14  
    https://github.com/riscv/riscv-isa-manual

2. **Alglave, Jade, et al.** "The RISC-V Instruction Set Manual, Volume I: User-Level ISA, Version 2.2 - Memory Model"  
    RISC-V Foundation, 2017

---

## Online Resources

1. **RISC-V International Website**  
    https://riscv.org/

2. **RISC-V Technical Specifications**  
    https://riscv.org/technical/specifications/

3. **RISC-V GitHub Organization**  
    https://github.com/riscv

4. **RISC-V Software Collaboration**  
    https://github.com/riscv-collab

5. **RISC-V Wiki**  
    https://wiki.riscv.org/

---

## Academic Papers

1. **Asanović, Krste, and David A. Patterson**. "Instruction Sets Should Be Free: The Case for RISC-V."  
    EECS Department, University of California, Berkeley, Technical Report No. UCB/EECS-2014-146, 2014.

2. **Waterman, Andrew**. "Design of the RISC-V Instruction Set Architecture."  
    PhD Thesis, University of California, Berkeley, 2016.

---

## Notes

- All RISC-V specifications are available under open licenses (Creative Commons or similar)
- This book does not use any proprietary or confidential information
- All code examples are original or based on publicly available documentation
- Readers should consult the official RISC-V specifications for the most current information

---

**Last Updated**: January 2026 (v0p11 Enhancement)
