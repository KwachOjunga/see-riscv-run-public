# Table of Contents  
  
## Front Matter  
  
- Cover  
- Copyright and License  
- Preface  
- Table of Contents  
  
## Part I — Introduction  
  
### Chapter 1: What Is RISC-V?  
  
1.1 The Birth of RISC-V  
1.2 RISC-V Design Philosophy  
1.3 RISC-V ISA Overview  
1.4 RISC-V Ecosystem  
1.5 Why Learn RISC-V?  
1.6 How to Use This Book  
  
## Part II — Programmer's Model  
  
### Chapter 2: Programmer's Model & Register Set  
  
2.1 Integer Register File  
2.2 Floating-Point Register File  
2.3 Control and Status Registers (CSRs)  
2.4 Privilege Modes  
2.5 Memory Model  
2.6 Comparison with ARM and MIPS  
  
### Chapter 3: Privilege Levels & Execution Environment  
  
3.1 RISC-V Privilege Architecture  
3.2 Machine Mode (M-mode)  
3.3 Supervisor Mode (S-mode)  
3.4 User Mode (U-mode)  
  
## Part III — Traps & Exceptions  
  
### Chapter 4: Trap, Exception, Interrupt  
  
4.1 Trap Handling Overview  
4.2 Exception Types  
4.3 Interrupt Types  
4.4 Trap Delegation  
4.5 Trap Vector Table  
4.6 Trap Entry and Exit  
4.7 Nested Traps  
4.8 Comparison with ARM and MIPS  
  
## Part IV — Memory & Addressing  
  
### Chapter 5: Virtual Memory & Paging (Sv39/Sv48)  
  
5.1 Virtual Memory Overview  
5.2 Page Table Structure  
5.3 Address Translation  
5.4 TLB Management  
  
### Chapter 6: Memory Ordering & Synchronization  
  
6.1 Memory Consistency Model  
6.2 RISC-V Memory Model (RVWMO)  
6.3 Fence Instructions  
6.4 Atomic Operations  
6.5 Load-Reserved/Store-Conditional  
6.6 Comparison with ARM and x86  
  
## Part V — Pipeline & Microarchitecture  
  
### Chapter 7: RISC-V Pipeline Fundamentals  
  
7.1 Classic Five-Stage Pipeline  
7.2 Pipeline Hazards  
7.3 Hazard Detection and Resolution  
7.4 Branch Prediction  
7.5 Pipeline Performance  
7.6 Comparison with ARM and MIPS  
  
### Chapter 8: Microarchitecture Variations  
  
8.1 In-Order vs Out-of-Order  
8.2 Superscalar Execution  
8.3 Cache Hierarchy  
8.4 Memory Subsystem  
8.5 RISC-V Core Examples  
8.6 Rocket Core  
8.7 BOOM (Berkeley Out-of-Order Machine)  
8.8 Performance Comparison  
  
## Part VI — Booting & System Software  
  
### Chapter 9: Reset, Boot Flow & Firmware  
  
9.1 Reset and Initialization  
9.2 Boot ROM  
9.3 Bootloader Stages  
9.4 Firmware Components  
9.5 Device Tree  
9.6 Boot Flow Examples  
9.7 Comparison with ARM  
  
### Chapter 10: Machine Mode, SBI & Supervisor Mode  
  
10.1 Machine Mode Overview  
10.2 Supervisor Binary Interface (SBI)  
10.3 SBI Implementation (OpenSBI)  
10.4 Supervisor Mode Software  
10.5 OS Kernel Integration  
10.6 Comparison with ARM  
  
## Part VII — ISA Extensions  
  
### Chapter 11: RISC-V Standard Extensions  
  
11.1 Extension Naming Convention  
11.2 M Extension (Integer Multiply/Divide)  
11.3 A Extension (Atomic Instructions)  
11.4 F Extension (Single-Precision Floating-Point)  
11.5 D Extension (Double-Precision Floating-Point)  
11.6 C Extension (Compressed Instructions)  
11.7 Zicsr and Zifencei  
11.8 Other Standard Extensions  
  
### Chapter 12: Vector Processing & SIMD Comparison  
  
12.1 Vector Extension Overview  
12.2 Vector Register File  
12.3 Vector Instructions  
12.4 Vector Length Agnostic Programming  
12.5 Vector Memory Operations  
12.6 Vector Performance  
12.7 Comparison with ARM SVE/SVE2  
12.8 Comparison with x86 AVX  
  
## Part VIII — System Design, Platform Spec & SoC Integration  
  
### Chapter 13: SoC Integration  
  
13.1 SoC Architecture Overview  
13.2 Interrupt Controllers (PLIC, CLIC, AIA)  
13.3 Memory-Mapped I/O  
13.4 DMA and Bus Protocols  
13.5 Power Management Integration  
  
### Chapter 14: RISC-V Platform Profiles & Embedded Systems  
  
14.1 Platform Specifications  
14.2 RVA Profiles (RVA22, RVA23)  
14.3 Embedded Profiles  
14.4 Certification and Compliance  
  
## Part IX — Performance, Debug & Tools  
  
### Chapter 15: Debugging & Trace  
  
15.1 Debug Architecture Overview  
15.2 Debug Module  
15.3 Trigger Module  
15.4 Trace Architecture  
15.5 JTAG and OpenOCD  
15.6 GDB Integration  
  
### Chapter 16: Performance Counters & PMU  
  
16.1 Performance Monitoring Overview  
16.2 Hardware Performance Counters  
16.3 Event Selection  
16.4 PMU Programming  
16.5 Performance Analysis Tools  
  
## Part X — RISC-V vs Other Architectures  
  
### Chapter 17: RISC-V vs ARM vs MIPS — A Systematic Comparison  
  
17.1 Historical Context  
17.2 ISA Design Philosophy  
17.3 Instruction Encoding  
17.4 Privilege Architecture  
17.5 Memory Model  
17.6 Ecosystem and Adoption  
17.7 Future Outlook  
  
## Appendices  
  
**Appendix A**: CSR Reference  
**Appendix B**: Extension Reference  
**Appendix C**: Bootloader Reference  
**Appendix D**: SBI Reference  
**Appendix E**: RISC-V vs ARM Comparison  
**Appendix F**: Memory Model Reference  
  
## Back Matter  
  
- About the Author  
- Bibliography  
