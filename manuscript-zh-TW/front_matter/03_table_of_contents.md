# 目錄  
  
## 前言  
  
- 封面  
- 版權與授權  
- 序言  
- 目錄  
  
## Part I — Introduction（簡介）  
  
### Chapter 1: 什麼是 RISC-V？  
  
1.1 RISC-V 的誕生  
1.2 RISC-V 設計哲學  
1.3 RISC-V ISA 概覽  
1.4 RISC-V 生態系統  
1.5 為什麼要學習 RISC-V？  
1.6 如何使用本書  
  
## Part II — Programmer's Model  
  
### Chapter 2: Programmer's Model 與 Register Set  
  
2.1 整數暫存器檔案  
2.2 浮點暫存器檔案  
2.3 控制與狀態暫存器（CSRs）  
2.4 特權模式  
2.5 記憶體模型  
2.6 與 ARM 和 MIPS 的比較  
  
### Chapter 3: Privilege Level 與 Execution Environment  
  
3.1 RISC-V Privilege 架構  
3.2 Machine Mode (M-mode)  
3.3 Supervisor Mode (S-mode)  
3.4 User Mode (U-mode)  
  
## Part III — Traps & Exceptions（陷阱與例外）  
  
### Chapter 4: Trap, Exception, Interrupt  
  
4.1 Trap 處理概覽  
4.2 Exception 類型  
4.3 Interrupt 類型  
4.4 Trap Delegation  
4.5 Trap Vector Table  
4.6 Trap 進入與退出  
4.7 巢狀 Trap  
4.8 與 ARM 和 MIPS 的比較  
  
## Part IV — Memory & Addressing（記憶體與定址）  
  
### Chapter 5: Virtual Memory 與 Paging (Sv39/Sv48)  
  
5.1 Virtual Memory 概覽  
5.2 Page Table 結構  
5.3 位址轉換  
5.4 TLB 管理  
  
### Chapter 6: Memory Ordering 與 Synchronization  
  
6.1 Memory Consistency Model  
6.2 RISC-V Memory Model (RVWMO)  
6.3 Fence 指令  
6.4 Atomic 操作  
6.5 Load-Reserved/Store-Conditional  
6.6 與 ARM 和 x86 的比較  
  
## Part V — Pipeline & Microarchitecture  
  
### Chapter 7: RISC-V Pipeline 基礎  
  
7.1 經典五階段 Pipeline  
7.2 Pipeline Hazard  
7.3 Hazard 偵測與解決  
7.4 Branch Prediction  
7.5 Pipeline 效能  
7.6 與 ARM 和 MIPS 的比較  
  
### Chapter 8: Microarchitecture 變化  
  
8.1 In-Order vs Out-of-Order  
8.2 Superscalar 執行  
8.3 Cache 階層  
8.4 記憶體子系統  
8.5 RISC-V Core 範例  
8.6 Rocket Core  
8.7 BOOM (Berkeley Out-of-Order Machine)  
8.8 效能比較  
  
## Part VI — Booting & System Software（開機與系統軟體）  
  
### Chapter 9: Reset, Boot Flow 與 Firmware  
  
9.1 Reset 與初始化  
9.2 Boot ROM  
9.3 Bootloader 階段  
9.4 Firmware 元件  
9.5 Device Tree  
9.6 Boot Flow 範例  
9.7 與 ARM 的比較  
  
### Chapter 10: Machine Mode, SBI 與 Supervisor Mode  
  
10.1 Machine Mode 概覽  
10.2 Supervisor Binary Interface (SBI)  
10.3 SBI 實作（OpenSBI）  
10.4 Supervisor Mode 軟體  
10.5 OS Kernel 整合  
10.6 與 ARM 的比較  
  
## Part VII — ISA Extensions  
  
### Chapter 11: RISC-V Standard Extensions  
  
11.1 Extension 命名慣例  
11.2 M Extension (Integer Multiply/Divide)  
11.3 A Extension (Atomic Instructions)  
11.4 F Extension (Single-Precision Floating-Point)  
11.5 D Extension (Double-Precision Floating-Point)  
11.6 C Extension (Compressed Instructions)  
11.7 Zicsr 與 Zifencei  
11.8 其他 Standard Extension  
  
### Chapter 12: Vector Processing 與 SIMD 比較  
  
12.1 Vector Extension 概覽  
12.2 Vector Register File  
12.3 Vector 指令  
12.4 Vector Length Agnostic Programming  
12.5 Vector Memory 操作  
12.6 Vector 效能  
12.7 與 ARM SVE/SVE2 的比較  
12.8 與 x86 AVX 的比較  
  
## Part VIII — System Design, Platform Spec & SoC Integration  
  
### Chapter 13: SoC 整合  
  
13.1 SoC 架構概覽  
13.2 Interrupt Controller (PLIC, CLIC, AIA)  
13.3 Memory-Mapped I/O  
13.4 DMA 與 Bus Protocol  
13.5 電源管理整合  
  
### Chapter 14: RISC-V Platform Profiles 與嵌入式系統  
  
14.1 Platform 規格  
14.2 RVA Profiles (RVA22, RVA23)  
14.3 Embedded Profiles  
14.4 認證與合規  
  
## Part IX — Performance, Debug & Tools  
  
### Chapter 15: Debugging 與 Trace  
  
15.1 Debug 架構概覽  
15.2 Debug Module  
15.3 Trigger Module  
15.4 Trace 架構  
15.5 JTAG 與 OpenOCD  
15.6 GDB 整合  
  
### Chapter 16: Performance Counters 與 PMU  
  
16.1 效能監控概覽  
16.2 硬體效能計數器  
16.3 Event 選擇  
16.4 PMU 程式設計  
16.5 效能分析工具  
  
## Part X — RISC-V vs Other Architectures  
  
### Chapter 17: RISC-V vs ARM vs MIPS — 系統性比較  
  
17.1 歷史背景  
17.2 ISA 設計哲學  
17.3 指令編碼  
17.4 Privilege 架構  
17.5 Memory Model  
17.6 生態系統與採用  
17.7 未來展望  
  
## 附錄  
  
**附錄 A**: CSR 參考  
**附錄 B**: Extension 參考  
**附錄 C**: Bootloader 參考  
**附錄 D**: SBI 參考  
**附錄 E**: RISC-V vs ARM 比較  
**附錄 F**: Memory Model 參考  
  
## 後記  
  
- 關於作者  
- 參考文獻  
