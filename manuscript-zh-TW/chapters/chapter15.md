# Chapter 15. Debugging & Trace

**Part IX — Performance, Debug & Tools**

---

## 🎯 學習目標

讀完本章後，你將能夠：

1. **掌握 GDB Stub 用法**：使用 QEMU `-s -S` 啟動 GDB Server 進行遠端除錯
2. **熟悉核心除錯指令**：`break`、`si`、`info reg`、`x/` 等 GDB 指令
3. **建立除錯思維**：採用「觀察 → 假設 → 驗證」的系統化除錯流程

---

## 💡 情境引入：讓時間暫停的懷錶

> **場景**：小華指著螢幕上的亂碼結果，快要抓狂了。

**小華**：「我不行了。我只是想把 1 加到 5，為什麼結果會變成 34821？我已經盯著這五行 Assembly 看了一小時了！」

**阿杰**：「用眼睛看是不夠的。小華，程式執行就像火車飛奔，你坐在路邊怎麼可能看清輪子上有沒有裂痕？」

**小華**：「那怎麼辦？加 `printf`？」

**阿杰**：「`printf` 是不錯，但在這種裸機 (Bare-metal) 或者程式崩潰的情況下，你根本印不出東西。你需要一把能讓時間暫停的『懷錶』—— **GDB**。」

**小華**：「暫停時間？」

**阿杰**：「沒錯。透過 **JTAG** 硬體介面（或者我們現在用的 QEMU 模擬器），我們可以強迫 CPU 進入 **Debug Mode**。

| Debug Mode 功能 | 比喻 |
|----------------|------|
| 檢查 Registers | 偷看錢包 |
| 查看 Memory | 翻箱倒櫃 |
| Single Step | 慢動作重播 |
| Breakpoint | 設置陷阱 |

在這個模式下，CPU 就像被按了暫停鍵。我們可以一次只執行一行指令，看看哪一步走歪了。

來，把這個壞掉的程式交給我，我們來抓蟲。」

---

Software development 需要 debugging。程式 crash，我們需要知道原因。Function 返回錯誤的值，我們需要 step through 它的執行。Performance bottleneck 出現，我們需要 trace instruction flow。Debugging 將不透明的 failure 轉化為可理解的問題。

RISC-V 提供全面的 debug architecture，支援 halting debug（停止 processor，檢查 state）和 non-intrusive trace（記錄執行而不停止）。Debug Module 允許 external debugger 通過 JTAG 或其他 interface 控制 processor。Hardware breakpoint 和 trigger 能夠精確控制何時 halt execution。Debug mode 為 debug operation 提供特殊的 execution environment。Trace support 捕獲 instruction 和 data flow 以進行 post-mortem analysis。

本章探討 RISC-V debugging 和 trace capability。我們將檢視 debug architecture、debug interface、hardware breakpoint、debug mode operation、trace support，以及 RISC-V 與 ARM CoreSight debug infrastructure 的比較。

---

## 15.1 RISC-V Debug Architecture

**Debug Requirements**

Debug system 必須提供：

- **Halt and resume**：停止 processor execution，檢查 state，繼續
- **Register access**：讀寫 CPU register
- **Memory access**：讀寫 system memory
- **Breakpoint**：在特定 instruction 或 data access 處停止執行
- **Single-step**：一次執行一條 instruction
- **Reset control**：Reset processor 或 system

RISC-V 的 debug architecture 將關注點分離到不同的 module，允許靈活的 implementation，同時保持 standard interface。

**Debug Components**

```
External Debugger (GDB, OpenOCD)
    ↓
Debug Transport Module (DTM) - JTAG, USB, etc.
    ↓
Debug Module Interface (DMI)
    ↓
Debug Module (DM)
    ↓
RISC-V Core (enters Debug Mode)
```

關鍵組件：

*Debug Transport Module (DTM)*：提供 physical connection（JTAG、USB 等）

*Debug Module (DM)*：控制 core，實作 debug operation

*Debug Module Interface (DMI)*：DTM 和 DM 之間的 standard interface

*Debug Mode*：Debug operation 的特殊 execution mode

**Debug Module (DM)**

Debug Module 是中央組件，負責：

- Halt 和 resume core
- 提供 abstract command 用於 register/memory access
- 管理 hardware breakpoint（trigger）
- 控制 reset

DM 通過 memory-mapped register 訪問：

```
DM Base Address: 0x00000000 (implementation-defined)

Key registers:
  dmcontrol:   Control register (halt, resume, reset)
  dmstatus:    Status register (halted, running, etc.)
  hartinfo:    Hart information
  abstractcs:  Abstract command status
  command:     Abstract command register
  data0-11:    Data transfer registers
  progbuf0-15: Program buffer
```

**Debug Mode vs Machine Mode**

Debug mode 是與 M/S/U mode 不同的特殊 execution mode：

- 比 M-mode 更高的 privilege
- 可以訪問所有 system resource
- 使用獨立的 CSR（dcsr、dpc、dscratch0/1）
- 從 Debug ROM 或 Program Buffer 執行

```
Privilege Hierarchy:
  Debug Mode (highest)
    ↓
  M-mode
    ↓
  S-mode
    ↓
  U-mode (lowest)
```

---

## 15.2 Debug Interface

**JTAG Interface**

JTAG (Joint Test Action Group) 是 RISC-V 的 standard debug interface。它提供：

- 4-wire interface（TDI、TDO、TCK、TMS）
- Boundary scan for testing
- Debug access to the core

JTAG signal：

```
TDI:  Test Data In (serial data input)
TDO:  Test Data Out (serial data output)
TCK:  Test Clock
TMS:  Test Mode Select (state machine control)
TRST: Test Reset (optional)
```

JTAG state machine：

```
Test-Logic-Reset
    ↓
Run-Test/Idle
    ↓
Select-DR-Scan → Capture-DR → Shift-DR → Exit1-DR → Update-DR
    ↓
Select-IR-Scan → Capture-IR → Shift-IR → Exit1-IR → Update-IR
```

**Debug Module Interface (DMI)**

DMI 是 DTM 和 DM 之間的 standard register interface。它提供：

- 32-bit 或 64-bit register access
- DM register 的 address space
- Status 和 error reporting

DMI operation：

```c
// Read DM register
uint32_t dmi_read(uint32_t addr) {
    // DTM shifts address into JTAG
    // DTM reads data from DM
    // Returns data
}

// Write DM register
void dmi_write(uint32_t addr, uint32_t data) {
    // DTM shifts address and data into JTAG
    // DM performs write
}
```

**Abstract Commands**

Abstract command 提供 high-level debug operation，無需進入 debug mode：

*Access Register*：讀寫 CPU register
*Access Memory*：讀寫 system memory
*Quick Access*：Fast register access

```c
// Example: Read register x10 using abstract command
void read_register_x10(uint32_t *value) {
    // Write abstract command
    dm_write(command, 0x00221000);  // regno=10, transfer, size=32
    
    // Wait for completion
    while (dm_read(abstractcs) & ABSTRACTCS_BUSY);
    
    // Read result
    *value = dm_read(data0);
}
```

**System Bus Access**

DM 可以通過 system bus 直接訪問 system memory，繞過 core：

```c
// Read memory at address 0x80000000
uint32_t read_memory(uint64_t addr) {
    dm_write(sbaddress0, addr & 0xFFFFFFFF);
    dm_write(sbaddress1, addr >> 32);
    dm_write(sbcs, SBCS_SBREADONADDR | SBCS_SBACCESS32);

    while (dm_read(sbcs) & SBCS_SBBUSY);

    return dm_read(sbdata0);
}
```

---

## 15.3 Hardware Breakpoints and Triggers

**Trigger Module**

RISC-V 提供靈活的 trigger system 用於 hardware breakpoint 和 watchpoint。Trigger 可以：

- Break on instruction execution（instruction breakpoint）
- Break on data access（data watchpoint）
- Break on exception
- Chain multiple condition

Trigger 通過 CSR 配置：

- `tselect`：Select trigger register
- `tdata1`：Trigger configuration
- `tdata2`：Trigger match value
- `tdata3`：Additional trigger data（optional）

**Trigger Types**

RISC-V 定義了幾種 trigger type：

*Type 2 (mcontrol)*：Address/data match trigger

- Match on instruction fetch、load 或 store
- Configurable match condition（equal、greater、less、mask）
- Action：enter debug mode、raise exception 或 trace

*Type 3 (icount)*：Instruction count trigger

- Break after N instruction
- 對 single-stepping 有用

*Type 4 (itrigger)*：Interrupt trigger

- Break on specific interrupt

*Type 5 (etrigger)*：Exception trigger

- Break on specific exception

**Breakpoint Configuration**

設置 instruction breakpoint：

```c
// Set breakpoint at address 0x80000100
void set_breakpoint(uint64_t addr) {
    // Select trigger 0
    write_csr(tselect, 0);

    // Configure mcontrol trigger
    uint64_t tdata1 = 0;
    tdata1 |= (2ULL << 60);      // type = 2 (mcontrol)
    tdata1 |= (1ULL << 6);       // m = 1 (match in M-mode)
    tdata1 |= (1ULL << 2);       // execute = 1 (match on instruction fetch)
    tdata1 |= (1ULL << 12);      // action = 1 (enter debug mode)

    write_csr(tdata1, tdata1);
    write_csr(tdata2, addr);     // Match address
}
```

設置 data watchpoint：

```c
// Set watchpoint on store to address 0x80001000
void set_watchpoint(uint64_t addr) {
    write_csr(tselect, 1);

    uint64_t tdata1 = 0;
    tdata1 |= (2ULL << 60);      // type = 2 (mcontrol)
    tdata1 |= (1ULL << 6);       // m = 1 (match in M-mode)
    tdata1 |= (1ULL << 1);       // store = 1 (match on store)
    tdata1 |= (1ULL << 12);      // action = 1 (enter debug mode)

    write_csr(tdata1, tdata1);
    write_csr(tdata2, addr);
}
```

**Trigger Chaining**

多個 trigger 可以 chain 以創建複雜的 condition：

```c
// Break when PC = 0x80000100 AND x10 = 42
void set_conditional_breakpoint(void) {
    // Trigger 0: Match PC
    write_csr(tselect, 0);
    uint64_t tdata1_0 = (2ULL << 60) | (1ULL << 6) | (1ULL << 2) | (1ULL << 11);
    // chain = 1 (bit 11)
    write_csr(tdata1, tdata1_0);
    write_csr(tdata2, 0x80000100);

    // Trigger 1: Match x10 value (requires data trigger support)
    write_csr(tselect, 1);
    uint64_t tdata1_1 = (2ULL << 60) | (1ULL << 6) | (1ULL << 12);
    // action = 1 (enter debug mode)
    write_csr(tdata1, tdata1_1);
    // Implementation-specific: match register value
}
```

**Trigger Actions**

當 trigger fire 時，它可以：

- Enter debug mode（action = 1）
- Raise breakpoint exception（action = 0）
- Generate trace event（implementation-specific）

---

## 15.4 Debug Mode

**Entering Debug Mode**

Core 在以下情況下進入 debug mode：

- External debugger request halt（via dmcontrol.haltreq）
- Hardware breakpoint fire（trigger with action = 1）
- Single-step complete（dcsr.step = 1）
- Debug interrupt（haltreq signal）

進入 debug mode 時：

1. PC 保存到 `dpc`（Debug PC）
2. Cause 保存到 `dcsr.cause`
3. Core halt execution
4. PC 跳轉到 Debug ROM 或 Program Buffer

**Debug CSRs**

Debug mode 使用三個特殊的 CSR：

*dcsr (Debug Control and Status)*：

```
Bits:
  [31:28] xdebugver: Debug spec version
  [15]    ebreakm: ebreak enters debug mode in M-mode
  [14]    ebreaks: ebreak enters debug mode in S-mode
  [13]    ebreaku: ebreak enters debug mode in U-mode
  [8:6]   cause: Why debug mode was entered
  [2]     step: Single-step mode
  [1:0]   prv: Privilege mode before debug
```

*dpc (Debug PC)*：進入 debug mode 時保存的 PC

*dscratch0/1*：Debug code 的 scratch register

**Debug ROM and Program Buffer**

進入 debug mode 時，core 從以下位置執行 code：

*Debug ROM*：包含 debug entry code 的小型 ROM

- 保存 context
- 等待 debugger command
- Resume 時恢復 context

*Program Buffer*：Debugger 提供的 code 的 RAM

- Debugger 在此寫入 instruction
- Core 在 debug mode 中執行它們
- 用於複雜 operation（例如，memory copy）

Debug ROM code 範例：

```assembly
# Debug ROM entry point
debug_rom_entry:
    # Save x10 to dscratch0
    csrw dscratch0, x10

    # Load program buffer address
    lui x10, %hi(progbuf)
    addi x10, x10, %lo(progbuf)

    # Jump to program buffer
    jr x10

# Program buffer (written by debugger)
progbuf:
    # Debugger writes instructions here
    # Example: read x5 into data0
    csrw dscratch1, x5
    # ... more instructions ...
    ebreak  # Return to debug ROM
```

**Resuming from Debug Mode**

Resume execution：

1. Debugger 寫入 dmcontrol.resumereq
2. Debug ROM 恢復 context
3. Core 執行 `dret` instruction
4. PC 從 `dpc` 恢復
5. Privilege mode 從 `dcsr.prv` 恢復

```assembly
# Resume from debug mode
debug_resume:
    # Restore x10 from dscratch0
    csrr x10, dscratch0

    # Return from debug mode
    dret  # PC ← dpc, privilege ← dcsr.prv
```

**Single-Stepping**

Single-step mode 執行一條 instruction 然後重新進入 debug mode：

```c
// Enable single-step
void enable_single_step(void) {
    uint64_t dcsr = read_csr(dcsr);
    dcsr |= (1 << 2);  // step = 1
    write_csr(dcsr, dcsr);
}

// Debugger workflow:
// 1. Halt core
// 2. Enable single-step
// 3. Resume (executes one instruction)
// 4. Core re-enters debug mode
// 5. Repeat
```

---

## 15.5 Trace Support

**RISC-V Trace Specification**

Trace 在不 halt core 的情況下捕獲 program execution 以進行分析。RISC-V trace 提供：

- Instruction trace：記錄執行的 instruction
- Data trace：記錄 memory access
- Trace compression：減少 trace bandwidth

Trace 是 non-intrusive 的 — 它不影響 program execution 或 timing。

**Instruction Trace**

Instruction trace 記錄：

- 執行的 instruction（PC value）
- Branch outcome（taken/not taken）
- Exception 和 interrupt
- Context change（privilege mode、ASID）

Trace packet 高效地編碼這些資訊：

```
Trace packet types:
  Format 0: Uncompressed address (full PC)
  Format 1: Differential address (PC delta)
  Format 2: Address with branch map
  Format 3: Synchronization packet
```

**Trace Compression**

Full instruction trace 成本高昂（bandwidth、storage）。RISC-V trace 使用 compression：

*Branch map*：在一個 packet 中編碼多個 branch outcome

```
Example: 8 branches, outcomes = 10110010
  Packet: [type=2, branches=10110010, address=...]
```

*Differential encoding*：編碼 PC delta 而不是 full PC

```
Previous PC: 0x80000100
Current PC:  0x80000104
Packet: [type=1, delta=+4]
```

*Implicit sequences*：不 trace sequential instruction

```
PC sequence: 0x100, 0x104, 0x108, 0x10c
Trace: [0x100, count=4]  # Implicit +4 increments
```

**Data Trace**

Data trace 記錄 memory access：

- Load/store address
- Data value
- Access size（byte、halfword、word、doubleword）

Data trace packet：

```
Data trace packet:
  [type] [address] [data] [size]

Example:
  SW x10, 0(x5)  # Store word
  Packet: [type=store, addr=0x80001000, data=0x12345678, size=4]
```

**Trace Filtering**

Trace 可以 filter 以減少 bandwidth：

- Address range filtering（僅 trace 特定 code region）
- Privilege filtering（僅 trace M-mode、S-mode 等）
- Event filtering（僅 trace branch、exception 等）

```c
// Configure trace filtering (implementation-specific)
void configure_trace_filter(uint64_t start, uint64_t end) {
    // Enable trace for address range [start, end]
    trace_write(TRACE_ADDR_START, start);
    trace_write(TRACE_ADDR_END, end);
    trace_write(TRACE_CONTROL, TRACE_ENABLE | TRACE_FILTER_ADDR);
}
```

---

## 15.6 Comparison with ARM Debug

**RISC-V Debug vs ARM CoreSight**

ARM CoreSight 是全面的 debug 和 trace infrastructure。比較：

| Feature | RISC-V Debug | ARM CoreSight |
|---------|--------------|---------------|
| **Debug Interface** | JTAG, DTM/DMI | JTAG, SWD (Serial Wire Debug) |
| **Debug Module** | Debug Module (DM) | Debug Access Port (DAP) |
| **Halt/Resume** | dmcontrol register | DHCSR register |
| **Breakpoints** | Trigger module (flexible) | FPB (Flash Patch and Breakpoint) |
| **Watchpoints** | Trigger module | DWT (Data Watchpoint and Trace) |
| **Trace** | RISC-V Trace spec | ETM (Embedded Trace Macrocell) |
| **Trace Compression** | Branch map, differential | Branch broadcast, compression |
| **System Access** | System bus access | AHB-AP, APB-AP |
| **Complexity** | Modular, simple | Comprehensive, complex |

**JTAG vs SWD**

RISC-V 使用 JTAG（4-5 wire），ARM 支援 JTAG 和 SWD（2 wire）：

```
JTAG (RISC-V, ARM):
  TDI, TDO, TCK, TMS, (TRST)
  5 pins, standard interface

SWD (ARM only):
  SWDIO (bidirectional data)
  SWCLK (clock)
  2 pins, lower pin count
```

SWD advantage：

- 更少的 pin（對小型 package 很重要）
- 在某些情況下比 JTAG 更快
- ARM-specific optimization

JTAG advantage：

- Industry standard（IEEE 1149.1）
- Widely supported tool
- Boundary scan capability

**Trace Comparison**

RISC-V Trace vs ARM ETM (Embedded Trace Macrocell)：

| Feature | RISC-V Trace | ARM ETM |
|---------|--------------|----------|
| **Instruction Trace** | Yes | Yes |
| **Data Trace** | Yes | Yes (ETMv4+) |
| **Compression** | Branch map, differential | Branch broadcast, Q elements |
| **Bandwidth** | Configurable | Configurable |
| **Filtering** | Address, privilege, event | Address, context ID, VMID |
| **Timestamps** | Optional | Yes |
| **Trace Port** | Implementation-specific | TPIU (Trace Port Interface Unit) |
| **Trace Buffer** | Implementation-specific | ETB (Embedded Trace Buffer) |

ARM ETM 成熟且廣泛部署。RISC-V Trace 較新，但遵循類似的原則，編碼更簡單。

**Debug Tools**

兩種 architecture 都支援 standard debug tool：

RISC-V：

- GDB (GNU Debugger)
- OpenOCD (Open On-Chip Debugger)
- SEGGER J-Link
- Lauterbach TRACE32

ARM：

- GDB
- Keil MDK
- ARM DS-5 / Arm Development Studio
- SEGGER J-Link
- Lauterbach TRACE32

**Practical Differences**

*RISC-V advantages*：

- 更簡單、更模組化的 design
- Open specification（no licensing）
- Flexible trigger system
- 更容易實作

*ARM advantages*：

- Mature ecosystem
- SWD 減少 pin count
- Comprehensive trace infrastructure
- Extensive tool support

對於 embedded system，SWD 的 2-pin interface 很有吸引力。對於複雜的 SoC，兩種 architecture 都提供可比較的 debug capability。

---

## 🛠️ 實作練習：Lab 15.1 — 消失的數值 (Bug Hunting with GDB)

這個 Lab 設計了一個經典的 **Pointer Stride Error**。這是 RISC-V 初學者最常犯的錯：以為 `int` 指針 `+1` 會移動 4 bytes，但在 Assembly 裡 `addi x, x, 1` 真的就是加 1 byte。

### 實驗目標

1. 啟動 QEMU 的 GDB Server 功能
2. 連接 GDB 並載入 Symbol
3. 使用 `layout asm` 檢視組合語言
4. 找出程式中的兩個 Bug

### 有 Bug 的程式碼 (buggy_sum.S)

```assembly
.section .data
# 定義一個陣列：10, 20, 30, 40, 50
# 預期結果：10+20+30+40+50 = 150 (Hex: 0x96)
nums: .word 10, 20, 30, 40, 50

.section .text
.global _start

_start:
    la  t0, nums        # t0 指向陣列開頭
    li  t1, 5           # t1 是迴圈計數器 (Count = 5)
    # BUG 1: 忘記初始化累加器 a0 (Accumulator)
    # 我們假設 a0 是 0，但實際上可能是垃圾值

loop:
    lw  t2, 0(t0)       # 載入當前數字到 t2
    add a0, a0, t2      # 累加：a0 = a0 + t2

    # BUG 2: 指針移動錯誤！
    # 我們讀取的是 word (4 bytes)，但這裡只加了 1
    addi t0, t0, 1      # ❌ 應該是 addi t0, t0, 4

    addi t1, t1, -1     # 計數器減 1
    bnez t1, loop       # 如果還沒數完，繼續跑

stop:
    j stop
```

### 除錯流程

**步驟 A：編譯 (帶 Debug 資訊)**

```bash
# -g 是關鍵！告訴編譯器保留符號表
riscv64-unknown-elf-gcc -g -nostdlib -o buggy_sum.elf buggy_sum.S
```

**步驟 B：啟動 QEMU (作為 Target)**

```bash
# -S: 啟動後立即暫停 CPU
# -s: 開啟 GDB Server，預設 Port 1234
qemu-system-riscv64 -machine virt -nographic \
    -kernel buggy_sum.elf -S -s
```

*(終端機會卡住，請開啟另一個終端機操作 GDB)*

**步驟 C：啟動 GDB (作為 Host)**

```bash
riscv64-unknown-elf-gdb buggy_sum.elf
```

**步驟 D：GDB 互動偵查**

```gdb
(gdb) target remote :1234     # 連接 QEMU
(gdb) layout asm              # 開啟 Assembly 視窗
(gdb) break loop              # 在 loop 標籤設斷點
(gdb) continue                # 執行到斷點

# 進入迴圈後...
(gdb) info reg a0             # 觀察累加器 → 發現不是 0！
(gdb) info reg t0             # 觀察指標
(gdb) si                      # Single Step 一行
(gdb) info reg t0             # 再看指標 → 只移動了 1 byte！

# 發現問題後...
(gdb) x/5xw &nums             # 查看陣列記憶體內容
```

### 預期發現

1. **Bug 1 (a0 未初始化)**：第一次進入 loop 時，`info reg a0` 顯示垃圾值
2. **Bug 2 (指針步長)**：每次 `addi t0, t0, 1` 後，t0 只增加 1，導致讀取到錯位的資料

### 修正後的程式碼

```assembly
_start:
    la  t0, nums
    li  t1, 5
    li  a0, 0           # ✅ FIX 1: 初始化累加器

loop:
    lw  t2, 0(t0)
    add a0, a0, t2
    addi t0, t0, 4      # ✅ FIX 2: word = 4 bytes
    addi t1, t1, -1
    bnez t1, loop

stop:
    j stop
```

---

## ⚠️ 常見陷阱

### 陷阱 1：編譯優化干擾除錯

**錯誤情境**：使用 `-O2` 編譯後，GDB 裡的行號跟原始碼對不上，變數也「看不見」。

**原因**：優化器會重排指令、消除暫存器、內聯函數。

```bash
# ❌ 除錯時不要用高優化等級
riscv64-unknown-elf-gcc -O2 -o program.elf program.c

# ✅ 除錯時使用 -O0 -g
riscv64-unknown-elf-gcc -O0 -g -o program.elf program.c
```

### 陷阱 2：忘記 `-g` 旗標

**錯誤情境**：GDB 顯示「No symbol table is loaded」。

**原因**：沒有加 `-g` 編譯，Symbol 資訊被丟棄。

```bash
# ❌ 沒有 debug info
riscv64-unknown-elf-gcc -o program.elf program.c

# ✅ 保留 debug info
riscv64-unknown-elf-gcc -g -o program.elf program.c
```

### 陷阱 3：QEMU 沒有用 `-S` 暫停

**錯誤情境**：GDB 連接時程式已經跑完或 Crash。

**解決**：一定要加 `-S` 讓 QEMU 啟動後暫停，等待 GDB 連接。

```bash
# ❌ 程式啟動後立刻開始執行
qemu-system-riscv64 -machine virt -kernel program.elf -s

# ✅ 程式啟動後暫停，等待 GDB
qemu-system-riscv64 -machine virt -kernel program.elf -S -s
```

> 💡 **小技巧**：`-s` = GDB Server on port 1234，`-S` = Stop at startup。兩者經常一起使用。

---

## Summary

Debugging 和 trace 對於 software development 和 system analysis 至關重要。本章探討了 RISC-V 的 debug architecture 以及它與 ARM 成熟的 CoreSight infrastructure 的比較。

**Debug architecture** 將關注點分離到模組化組件中。Debug Transport Module 通過 JTAG 或其他 interface 提供 physical connectivity。Debug Module 通過 standard Debug Module Interface 控制 core。Debug mode 為 debug operation 提供 privileged execution environment。這種分離允許靈活的 implementation，同時為 debugger tool 保持 standard interface。

**Debug interface** 使用 JTAG 作為 standard physical layer，為 debug access 提供 four-wire connectivity。Debug Module Interface 定義 register-level operation 以控制 core。Abstract command 能夠進行 high-level operation（如 register 和 memory access），無需進入 debug mode。System bus access 允許 debugger 直接讀寫 memory，完全繞過 core。

**Hardware breakpoint 和 trigger** 提供靈活的 mechanism 來 halt execution。Trigger module 支援多種 trigger type，包括 address match、instruction count、interrupt 和 exception trigger。Trigger 可以 match instruction fetch、data load 或 data store。Trigger chaining 能夠實現複雜的 conditional breakpoint。Action 包括進入 debug mode、raise exception 或 generate trace event。

**Debug mode** 是比 M-mode 更高 privilege 的特殊 execution environment。Core 在 external halt request、breakpoint hit 或 single-step completion 時進入 debug mode。Debug CSR（dcsr、dpc、dscratch）管理 debug state。Debug ROM 提供 entry code，而 Program Buffer 允許 debugger 執行 custom instruction sequence。Single-stepping 執行一條 instruction 然後重新進入 debug mode，實現 step-through debugging。

**Trace support** non-intrusively 捕獲 program execution。Instruction trace 記錄執行的 instruction、branch outcome 和 control flow change。Data trace 記錄 memory access 和 data value。Trace compression 通過 branch map、differential encoding 和 implicit sequence 減少 bandwidth。Trace filtering 將 capture 限制在特定 address range、privilege level 或 event，減少 trace data volume。

**Comparison with ARM** 顯示了相似性和差異性。ARM CoreSight 提供全面的 debug 和 trace infrastructure，具有成熟的 tool support。SWD 提供 2-pin 替代 JTAG，減少 embedded system 的 pin count。ARM ETM 提供廣泛的 trace capability，具有複雜的 compression。RISC-V 的 debug architecture 更簡單、更模組化，具有 open specification 和 flexible trigger system。兩種 architecture 都支援 standard tool（如 GDB 和 OpenOCD），確保實用性。

總之，RISC-V 的 debug 和 trace capability 能夠在從 embedded microcontroller 到 high-performance application processor 的全部範圍內實現有效的 software development、system analysis 和 problem diagnosis。

---
