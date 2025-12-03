# Chapter 2. Programmer's Model 與 Register Set

**Part II — Programmer's Model**

---

理解處理器架構始於理解其 programmer's model：軟體用來與硬體互動的 register、instruction 和慣例。RISC-V 的 programmer's model 簡潔、規則，並且設計兼顧簡潔性和效率。

本章探討每個 RISC-V 程式設計師必須知道的基本元素。我們將檢視 32 個 general-purpose register 及其慣用用途、管理處理器狀態的 Control and Status Register (CSR)、將 user code 與 operating system code 分離的 privilege level，以及使 function 能夠協同工作的 calling convention。我們將看到 RISC-V 的設計選擇——如 zero register、獨立的 CSR address space 和簡潔的 privilege model——如何簡化硬體實現和軟體開發。

---

## 2.1 General-Purpose Register

**Register File**

RISC-V 提供 32 個 general-purpose register，編號為 x0 到 x31。每個 register 的寬度為 XLEN bit，其中 XLEN 在 RV32 中為 32，在 RV64 中為 64，在 RV128 中為 128。本討論專注於 RV64，這是 application processor 最常見的變體。

32 個 register 的設計遵循 RISC 傳統。它足夠大，可以將經常使用的值保存在快速的 register 中而不是緩慢的 memory 中，但又足夠小，可以在硬體中高效實現。Register file 需要多個 read 和 write port 來支援 instruction 執行，大小直接影響晶片面積和存取時間。

與某些架構不同，RISC-V 的 register 是真正的 general-purpose。大多數 register 沒有特殊限制——任何 register 都可以用作大多數 instruction 的 source 或 destination。這種正交性簡化了硬體實現和 compiler 設計。

**Register x0: Zero Register**

Register x0 是特殊的：它總是讀取為零，對它的寫入會被丟棄。這可能看起來很浪費，但它非常有用。

Zero register 使幾個常見操作無需專用 instruction：

- **NOP (no operation)**：`ADDI x0, x0, 0` 將零加到零並存儲在 x0 中（丟棄結果）
- **Move**：`ADDI x1, x2, 0` 將零加到 x2 並存儲在 x1 中，有效地將 x2 複製到 x1
- **Load immediate**：`ADDI x1, x0, 42` 將 42 加到零，將常數 42 載入 x1
- **Unconditional branch**：`BEQ x0, x0, target` 如果 x0 等於 x0 則 branch（總是 true）

Zero register 也簡化了硬體。許多操作自然產生零（如 register 與自身的 XOR），擁有專用的 zero register 使這些操作明確且高效。

**標準 Register 名稱（ABI）**

雖然硬體將 register 稱為 x0-x31，但軟體使用 Application Binary Interface (ABI) 定義的符號名稱。這些名稱指示每個 register 的慣用用途：

| Register | ABI 名稱 | 描述 | 保存者 |
|----------|---------|------|--------|
| x0 | zero | 硬連線為零 | — |
| x1 | ra | Return address | Caller |
| x2 | sp | Stack pointer | Callee |
| x3 | gp | Global pointer | — |
| x4 | tp | Thread pointer | — |
| x5-x7 | t0-t2 | Temporary | Caller |
| x8 | s0/fp | Saved register / Frame pointer | Callee |
| x9 | s1 | Saved register | Callee |
| x10-x11 | a0-a1 | Function argument / Return value | Caller |
| x12-x17 | a2-a7 | Function argument | Caller |
| x18-x27 | s2-s11 | Saved register | Callee |
| x28-x31 | t3-t6 | Temporary | Caller |

這些名稱是慣例，不是硬體要求。處理器不強制執行它們——你可以將 `sp` 用於算術運算（儘管你的程式可能會崩潰）。但遵循 ABI 確保來自不同 compiler 和 library 的程式碼可以互操作。

32 個 general-purpose register 按其 ABI 名稱和慣用用途組織。Register 分為以下類別：zero register (x0)、special-purpose register (ra, sp, gp, tp)、caller-saved temporary 和 argument (t0-t6, a0-a7)，以及 callee-saved register (s0-s11)。完整的 register file 組織與 ABI 名稱顯示在上表中。

**Caller-Saved vs Callee-Saved**

ABI 根據誰在 function call 之間保留其值，將 register 分為兩類：

*Caller-Saved Register* (t0-t6, a0-a7)：如果 calling function 在 call 之後需要這些值，則必須保存它們。Called function 可以自由修改它們。這些用於 temporary value 和 function argument。

*Callee-Saved Register* (s0-s11, sp)：Called function 必須保留這些。如果使用它們，必須在進入時保存其值，並在返回前恢復它們。這些用於必須在 function call 之間存活的值。

這種劃分最佳化了常見情況。如果 temporary value 在 call 之後不使用，則不需要保存。Long-lived value 會自動在 call 之間保留。

**Special-Purpose Register**

幾個 register 有特殊的慣用用途：

*ra (x1) - Return Address*：存儲 function call 的 return address。JAL 和 JALR instruction (jump-and-link) 自動將 return address 寫入 ra。Function 透過跳轉到 ra 中的 address 來返回。

*sp (x2) - Stack Pointer*：指向 stack 的頂部。按照慣例，stack 向下增長（朝向較低的 address）。Function 透過從 sp 減去來分配 stack space，透過加到 sp 來釋放。

*gp (x3) - Global Pointer*：指向 4KB global variable 區域的中間。這允許使用 12-bit signed offset（從 gp ±2KB）的單個 load/store instruction 存取 global。Linker 設置 gp，它在執行期間保持不變。

*tp (x4) - Thread Pointer*：在 multi-threaded program 中指向 thread-local storage。每個 thread 都有自己的 tp 值，允許高效存取 thread-specific data。

*fp (x8) - Frame Pointer*：s0 的別名，用於指向當前 stack frame。某些程式碼使用 fp 來存取 local variable 和 function argument，而 sp 可能在 function 執行期間改變。其他程式碼省略 frame pointer 以釋放另一個 register。

**實際中的 Register 使用**

理解 register 使用對於閱讀 assembly code 和理解 compiler output 至關重要。這是一個典型的 function call 序列：

```assembly
# Caller 準備 argument
li a0, 10          # 第一個 argument 在 a0
li a1, 20          # 第二個 argument 在 a1

# Caller 保存任何需要的 temporary
sd t0, 0(sp)       # 如果 call 後需要，保存 t0

# Call function
jal ra, my_func    # 跳轉到 my_func，將 return address 保存在 ra

# Function 返回後，結果在 a0
mv t0, a0          # 將結果移到 t0

# 恢復保存的 temporary
ld t0, 0(sp)       # 恢復 t0
```

這個序列展示了 ABI 慣例的實際應用：argument 在 a0-a7 中傳遞，return value 在 a0 中返回，return address 在 ra 中，caller 負責保存 temporary register。

在 called function 內部：

```assembly
my_func:
    # Prologue: 分配 stack frame
    addi sp, sp, -32   # 分配 32 byte
    sd ra, 24(sp)      # 保存 return address
    sd s0, 16(sp)      # 如果要使用 s0，保存它

    # Function body 使用 a0, a1 (argument)
    add s0, a0, a1     # 使用 s0 進行計算

    # 準備 return value
    mv a0, s0          # Return value 在 a0

    # Epilogue: 恢復並返回
    ld s0, 16(sp)      # 恢復 s0
    ld ra, 24(sp)      # 恢復 return address
    addi sp, sp, 32    # 釋放 stack frame
    ret                # 返回（ret 是 jalr x0, 0(ra) 的 pseudo-instruction）
```

這個模式——prologue、body、epilogue——是 RISC-V function 的標準。Prologue 保存 register 並分配 stack space。Body 執行計算。Epilogue 恢復 register 並返回。

---

## 2.2 Control and Status Register (CSR)

**CSR 概覽**

除了 32 個 general-purpose register，RISC-V 為 Control and Status Register (CSR) 定義了一個獨立的 address space。這些 register 控制處理器行為、報告狀態，並提供對 privileged functionality 的存取。

CSR 使用專用 instruction（CSRRW、CSRRS、CSRRC 及其 immediate 變體）存取，而不是普通的 load/store instruction。每個 CSR 都有一個 12-bit address，允許最多 4,096 個 CSR，儘管目前只定義了一小部分。

CSR address space 按 privilege level 和 read/write access 分區：

- Bit [11:10] 編碼存取 CSR 所需的 privilege level
- Bit [9:8] 指示 read/write vs read-only
- Bit [7:0] 識別特定 register

這種編碼允許硬體快速檢查存取權限。嘗試從不足的 privilege level 存取 CSR 或寫入 read-only CSR 會導致 illegal instruction exception。

**Figure 2.1: CSR Address Space 組織**

```
CSR Address Space (12-bit)
├── Bit [11:10]: Privilege Level
│   ├── 00: User
│   ├── 01: Supervisor
│   ├── 10: Reserved
│   └── 11: Machine
├── Bit [9:8]: Read/Write
│   ├── 00-10: Read/Write
│   └── 11: Read-Only
└── Bit [7:0]: Register ID
    └── 每個 privilege level 256 個可能的 register

範例 CSR：
- mstatus: 0x300 (Machine Status)
- sstatus: 0x100 (Supervisor Status)
- cycle: 0xC00 (Cycle Counter, Read-Only)
```

**Machine-Level CSR**

Machine mode 是 RISC-V 中最高的 privilege level，可以存取所有 CSR。關鍵的 machine-level CSR 包括：

*mstatus* (Machine Status)：控制和報告處理器狀態的各個方面：

- MIE: Machine Interrupt Enable（M-mode 的 global interrupt enable）
- MPIE: 先前的 MIE 值（在 trap 時保存）
- MPP: 先前的 privilege mode（在 trap 時保存）
- MPRV: Modify Privilege（影響 memory access privilege）
- 各種 extension enable bit（FS 用於 floating-point，VS 用於 vector）

*misa* (Machine ISA)：指示實現了哪些 extension。每個 bit 對應一個 extension（bit 0 = A extension，bit 12 = M extension，等等）。這個 register 允許軟體檢測可用功能。在某些實現中，misa 是 read-only；在其他實現中，可以寫入以動態啟用/禁用 extension。

*mie* (Machine Interrupt Enable)：控制啟用哪些 interrupt。每個 bit 對應一個 interrupt source（software interrupt、timer interrupt、external interrupt）。即使在 mie 中設置了 bit，只有當 mstatus 中的 MIE 也設置時，才會接受 interrupt。

*mip* (Machine Interrupt Pending)：指示哪些 interrupt 正在等待。當 interrupt 到達時，硬體設置 bit；軟體可以讀取 mip 以確定哪些 interrupt 正在等待。

*mtvec* (Machine Trap Vector)：指定 trap handler 的 address。低 2 bit 選擇模式：

- 0 (Direct)：所有 trap 跳轉到相同 address (mtvec & ~0x3)
- 1 (Vectored)：Interrupt 跳轉到 (mtvec & ~0x3) + 4×cause，exception 跳轉到 (mtvec & ~0x3)

*mepc* (Machine Exception PC)：存儲導致 trap 的 instruction 的 program counter（對於 exception）或在處理 interrupt 後要恢復的 instruction。Trap handler 透過將 mepc 寫入 PC 來返回。

*mcause* (Machine Cause)：指示導致 trap 的原因。高 bit 區分 interrupt (1) 和 exception (0)。低 bit 編碼特定原因（例如，2 = illegal instruction，11 = environment call from M-mode）。

*mtval* (Machine Trap Value)：提供有關 trap 的額外資訊。對於與 address 相關的 exception（如 page fault），mtval 包含 faulting address。對於 illegal instruction exception，它可能包含 instruction 本身。

*mscratch* (Machine Scratch)：machine-mode 軟體的 general-purpose register。通常用於在進入 trap handler 時暫時保存 register，在 handler 設置其 stack 之前。

**Supervisor-Level CSR**

Supervisor mode 用於 operating system。它有自己的 CSR 集，類似於 machine-level CSR：

*sstatus*：mstatus 的受限視圖，僅顯示與 supervisor mode 相關的欄位（SIE、SPIE、SPP 等）。寫入 sstatus 實際上修改 mstatus 中的相應欄位。

*sie, sip*：Supervisor interrupt enable 和 pending register，類似於 mie/mip，但用於 supervisor-level interrupt。

*stvec, sepc, scause, stval, sscratch*：Trap-handling CSR 的 supervisor 版本，當 trap 被委派給 supervisor mode 時使用。

*satp* (Supervisor Address Translation and Protection)：控制 virtual memory：

- MODE: 選擇 address translation scheme（Bare、Sv39、Sv48 等）
- ASID: Address Space Identifier，用於 TLB tagging
- PPN: Root page table 的 physical page number

satp register 對 virtual memory 至關重要。寫入 satp 可以改變 address translation mode 或切換到不同的 page table，實現 process 之間的 context switch。

**User-Level CSR**

User mode 可以存取有限的 CSR 集，主要用於 performance monitoring 和 floating-point control：

*fflags, frm, fcsr*：Floating-point exception flag、rounding mode 和組合的 control/status register。這些允許 user code 控制 floating-point 行為並檢測 exception。

*cycle, time, instret*：從 user mode 可存取的 performance counter（如果未被 supervisor/machine mode 禁用）。這些提供經過的 cycle 數、當前時間和已退休的 instruction，對 profiling 和 timing 很有用。

**CSR Instruction**

RISC-V 提供六個 CSR 操作 instruction：

*CSRRW rd, csr, rs1* (CSR Read-Write)：原子地將 csr 中的值與 rs1 中的值交換，將舊的 CSR 值寫入 rd。如果 rd 是 x0，則抑制讀取（對 write-only access 有用）。

*CSRRS rd, csr, rs1* (CSR Read-Set)：將 csr 讀入 rd，然後設置 csr 中對應於 rs1 中 1 bit 的 bit。如果 rs1 是 x0，這是 read-only operation。

*CSRRC rd, csr, rs1* (CSR Read-Clear)：將 csr 讀入 rd，然後清除 csr 中對應於 rs1 中 1 bit 的 bit。

*CSRRWI, CSRRSI, CSRRCI*：使用 5-bit immediate value 而不是 register 的 immediate 變體。

這些 instruction 是 atomic 的，確保 CSR 修改不會被中斷。Read-set 和 read-clear operation 對於操作單個 bit 而不影響其他 bit 特別有用。

範例：啟用 machine-mode interrupt：

```assembly
# 讀取 mstatus，設置 MIE bit (bit 3)
li t0, 0x8         # MIE = bit 3
csrrs x0, mstatus, t0  # 設置 MIE，不讀取舊值（rd = x0）
```

範例：保存和修改 CSR：

```assembly
# 保存當前 mstatus 並禁用 interrupt
csrrci t0, mstatus, 0x8  # 清除 MIE，將舊值保存在 t0
# ... critical section ...
csrw mstatus, t0         # 恢復原始 mstatus
```

CSR instruction 提供對 privileged state 的受控存取，使 operating system 和 firmware 能夠管理處理器，同時防止 user code 干擾系統操作。

---

## 2.3 Program State 與 Privilege Level

**Privilege Mode**

RISC-V 定義了三個 privilege level，從低到高：

*User Mode (U-mode)*：最低 privilege level，用於 application code。User mode 無法存取大多數 CSR 或執行 privileged instruction。它通常在啟用 virtual memory 的情況下運行，將 process 彼此隔離並與 OS 隔離。

*Supervisor Mode (S-mode)*：用於 operating system。Supervisor mode 可以管理 virtual memory、處理從 machine mode 委派的 trap，並存取 supervisor-level CSR。它無法存取 machine-level CSR 或為 firmware 保留的某些 privileged operation。

*Machine Mode (M-mode)*：最高 privilege level，用於 firmware 和 bootloader。Machine mode 對所有硬體資源具有不受限制的存取。它可以存取所有 CSR、執行所有 instruction，並將 trap 委派給較低的 privilege level。

並非所有實現都支援所有 mode。簡單的 embedded system 可能只實現 M-mode。Microcontroller 可能實現 M-mode 和 U-mode。完整的 application processor 實現所有三個 mode。

當前 privilege level 不存儲在專用 register 中。相反，它隱含在處理器狀態中，可以從 CSR（如 trap 後的 mstatus.MPP（先前 privilege））推斷。

**Privilege Level 轉換**

Privilege level 之間的轉換透過明確定義的機制發生：

*Trap to Higher Privilege*：當發生 exception 或 interrupt 到達時，處理器 trap 到更高的 privilege level（或保持在相同 level）。Trap handler 由 xtvec CSR 確定（M-mode 的 mtvec，S-mode 的 stvec）。處理器將當前 PC 保存在 xepc 中，將原因保存在 xcause 中，將額外資訊保存在 xtval 中。

*Return from Trap*：MRET、SRET 和 URET instruction 從 trap 返回，從 xstatus.xPP 恢復 privilege level，從 xepc 恢復 PC。這些 instruction 是 privileged 的——MRET 只能在 M-mode 中執行，SRET 在 S-mode 或更高 mode 中執行。

*Environment Call*：ECALL instruction 明確請求 trap 到更高的 privilege level。User code 使用 ECALL 調用 OS service（system call）。OS code 使用 ECALL 調用 firmware service（SBI call）。Trap handler 檢查 calling context 以確定請求了哪個 service。

這種受控轉換機制確保 privilege escalation 只透過定義的 entry point 發生，維護系統安全。

**Figure 2.2: Privilege Level 轉換**

```
Privilege Level 狀態轉換：

Machine Mode (M-mode) ←→ Supervisor Mode (S-mode) ←→ User Mode (U-mode)
    ↑                         ↑                           ↑
    │                         │                           │
    │ Reset/Boot              │ MRET                      │ SRET
    │                         │                           │
    │                         │ Exception/Interrupt       │ ECALL (system call)
    │                         │ (not delegated)           │ Exception/Interrupt
    │                         │                           │ (delegated to S-mode)
    │                         │                           │
    └─────────────────────────┴───────────────────────────┘

M-mode: 最高 privilege，完全硬體存取，Firmware/Bootloader
S-mode: OS kernel，Virtual memory control，Delegated trap
U-mode: Application code，受限存取，Virtual memory enabled
```

狀態圖顯示 privilege level 如何透過 trap（向上）和 return instruction（向下）轉換。ECALL 明確請求更高 privilege，而 exception 和 interrupt 導致自動轉換。

---

## 2.4 Calling Convention 與 ABI

**RISC-V Calling Convention**

Calling convention 定義 function 如何相互呼叫：如何傳遞 argument、如何傳達 return value，以及必須保留哪些 register。RISC-V 遵循 System V ABI (Application Binary Interface)，許多其他架構也使用它。

Calling convention 是*軟體*慣例，不由硬體強制執行。處理器不關心你使用哪些 register 作為 argument。但遵循慣例確保來自不同 compiler 和 library 的程式碼可以互操作。

**Argument 傳遞**

Function argument 在 register `a0` 到 `a7` (x10-x17) 中傳遞。第一個 argument 在 `a0` 中，第二個在 `a1` 中，依此類推：

```c
int add(int x, int y, int z) {
    return x + y + z;
}
```

編譯為：

```assembly
add:
    add a0, a0, a1    # a0 = x + y
    add a0, a0, a2    # a0 = (x + y) + z
    ret
```

Argument `x`、`y` 和 `z` 到達 `a0`、`a1` 和 `a2`。結果在 `a0` 中返回。

如果 function 有超過 8 個 argument，額外的 argument 在 stack 上傳遞：

```c
int sum9(int a, int b, int c, int d, int e, int f, int g, int h, int i) {
    return a + b + c + d + e + f + g + h + i;
}
```

Argument `a` 到 `h` 在 `a0`-`a7` 中。Argument `i` 在 stack 上的 `sp+0`。

**Return Value**

Return value 在 `a0` 和 `a1` 中傳遞：

- 單個 return value（最多 XLEN bit）：`a0`
- 兩個 return value 或 RV64 上的 128-bit value：`a0`（低）和 `a1`（高）

例如，在 RV64 上返回 128-bit integer 的 function：

```c
__int128 multiply(__int128 x, __int128 y);
```

在 `a0` 中返回低 64 bit，在 `a1` 中返回高 64 bit。

Structure 和 union 的處理特殊：

- 小 struct（≤ 2×XLEN bit）在 `a0` 和 `a1` 中返回
- 較大的 struct 透過 pointer 返回：caller 分配 space 並在 `a0` 中傳遞 pointer；function 在那裡寫入結果並在 `a0` 中返回 pointer

**Caller-Saved vs Callee-Saved Register**

Calling convention 將 register 分為兩類：

*Caller-saved* (temporary register)：如果 caller 需要在 function call 之間保留這些值，則必須保存它們。Called function 可以自由修改它們。

- `t0`-`t6` (x5-x7, x28-x31): Temporary
- `a0`-`a7` (x10-x17): Argument/return value
- `ra` (x1): Return address（被 `call` 修改）

*Callee-saved* (saved register)：Called function 必須保留這些。如果使用它們，必須在進入時保存它們，並在返回前恢復它們。

- `s0`-`s11` (x8-x9, x18-x27): Saved register
- `sp` (x2): Stack pointer

範例：

```assembly
function:
    # Prologue: 保存 callee-saved register
    addi sp, sp, -16
    sd s0, 0(sp)
    sd s1, 8(sp)

    # Function body: 可以自由使用 s0, s1
    mv s0, a0
    mv s1, a1
    # ... 計算 ...

    # Epilogue: 恢復 callee-saved register
    ld s0, 0(sp)
    ld s1, 8(sp)
    addi sp, sp, 16
    ret
```

**Special Register**

幾個 register 有特殊角色：

*Stack Pointer (sp)*：指向 stack 的頂部。必須由 callee 保留。Stack 向下增長（朝向較低的 address）。

*Return Address (ra)*：保存當前 function 的 return address。由 `call`（或 `jal`）設置，由 `ret`（即 `jalr zero, 0(ra)`）使用。

*Frame Pointer (fp/s0)*：可選地指向當前 stack frame 的 base。這與 `s0` 是同一個 register。使用 frame pointer 簡化 debugging 和 stack unwinding，但會佔用一個 register。

*Global Pointer (gp)*：指向 global data。用於 relaxation optimization——linker 可以用 gp-relative address 替換 absolute address，節省 instruction。通常在程式啟動時設置一次，永不改變。

*Thread Pointer (tp)*：指向 thread-local storage (TLS)。每個 thread 都有自己的 TLS 區域。OS 在創建 thread 時設置 `tp`。

---

## 2.5 Stack Frame 結構

**Stack Layout**

Stack 是用於以下目的的 memory 區域：

- Local variable
- Saved register
- Function argument（超過前 8 個）
- Return address（用於 nested call）

Stack 向下增長。Stack pointer (`sp`) 指向 stack 的頂部（最低 address）。分配 stack space 意味著從 `sp` 減去；釋放意味著加到 `sp`。

典型的 stack frame 如下所示：

```
較高 address
+------------------+
| Caller's frame   |
+------------------+
| Argument 9+      | ← 在 stack 上傳遞
+------------------+
| Return address   | ← 由 caller 保存（如果需要）
+------------------+
| Saved register   | ← Callee-saved (s0-s11)
+------------------+
| Local variable   |
+------------------+
| Outgoing arg     | ← 用於此 function 呼叫的 function
+------------------+ ← sp (stack pointer)
較低 address
```

**Function Prologue 和 Epilogue**

*Prologue* 是 function 開始時設置 stack frame 的程式碼：

```assembly
function:
    # Prologue
    addi sp, sp, -32      # 分配 32 byte
    sd ra, 24(sp)         # 保存 return address
    sd s0, 16(sp)         # 保存 s0
    sd s1, 8(sp)          # 保存 s1
    # (Local variable 使用 sp+0 到 sp+7)

    # Function body
    # ...

    # Epilogue
    ld ra, 24(sp)         # 恢復 return address
    ld s0, 16(sp)         # 恢復 s0
    ld s1, 8(sp)          # 恢復 s1
    addi sp, sp, 32       # 釋放 stack frame
    ret
```

*Epilogue* 是結尾處拆除 stack frame 並返回的程式碼。

**Frame Pointer**

某些 function 使用 frame pointer (`fp`，即 `s0`)。Frame pointer 指向 stack frame 中的固定位置，使存取 local variable 和 argument 更容易：

```assembly
function:
    # 帶 frame pointer 的 prologue
    addi sp, sp, -32
    sd ra, 24(sp)
    sd s0, 16(sp)         # 保存舊 frame pointer
    addi s0, sp, 32       # 將 frame pointer 設置為舊 sp

    # 現在可以相對於 fp 存取 local：
    # Local var 在 fp-8, fp-16, 等等

    # Epilogue
    ld ra, -8(s0)
    ld s0, -16(s0)
    addi sp, s0, -32
    ret
```

Frame pointer 是可選的。它們簡化 debugging（debugger 可以遍歷 stack）和 exception handling，但會佔用一個 register。

**Leaf Function**

*Leaf function* 是不呼叫任何其他 function 的 function。Leaf function 通常可以避免保存 `ra` 和分配 stack frame：

```assembly
leaf_function:
    # 不需要 prologue
    add a0, a0, a1
    ret
    # 不需要 epilogue
```

這更有效率，但僅在 function 不呼叫任何東西且不需要保存 register 時才有效。

---

## 2.6 與 ARM64 和 MIPS 的比較

**Register 數量和使用**

所有三個架構都有 32 個 general-purpose register，但它們的使用方式不同：

*RISC-V*:

- 32 個 register (x0-x31)
- x0 硬連線為零
- 31 個可用 register
- 清晰的 caller/callee-saved 區分

*ARM64*:

- 31 個 general-purpose register (x0-x30)
- x31 是特殊的：根據 context 是 zero register 或 stack pointer
- 30 個完全通用的 register
- Link register (x30) 保存 return address

*MIPS*:

- 32 個 register ($0-$31)
- $0 硬連線為零
- $31 是 return address (ra)
- 30 個可用 general register

**Calling Convention**

*RISC-V*:

- Argument: a0-a7（8 個 register）
- Return: a0-a1
- Caller-saved: t0-t6, a0-a7
- Callee-saved: s0-s11

*ARM64*:

- Argument: x0-x7（8 個 register）
- Return: x0-x1
- Caller-saved: x0-x18
- Callee-saved: x19-x28

*MIPS*:

- Argument: $a0-$a3（4 個 register，少於 RISC-V/ARM）
- Return: $v0-$v1
- Caller-saved: $t0-$t9
- Callee-saved: $s0-$s7

RISC-V 和 ARM64 相似，都提供 8 個 argument register。MIPS 較舊，只提供 4 個，這意味著對於有許多 argument 的 function，需要更多 stack 使用。

**Special Register**

*RISC-V*:

- sp (x2): Stack pointer
- ra (x1): Return address
- gp (x3): Global pointer
- tp (x4): Thread pointer

*ARM64*:

- sp (在某些 context 中為 x31): Stack pointer
- lr (x30): Link register (return address)
- 沒有 global pointer 等效物
- Platform register 用於 TLS

*MIPS*:

- sp ($29): Stack pointer
- ra ($31): Return address
- gp ($28): Global pointer
- 沒有標準 thread pointer

**Zero Register**

RISC-V 和 MIPS 都有硬連線的 zero register (x0 / $0)。ARM64 的 x31 在某些 context 中可以充當 zero，但也用作 stack pointer，這更複雜。

Zero register 非常有用：

- `mv rd, rs` 是 `addi rd, rs, 0` 或 `add rd, rs, zero`
- `li rd, imm` 是 `addi rd, zero, imm`
- 丟棄結果：`add zero, a0, a1`（計算但丟棄）

---

## 總結

RISC-V 的 programmer's model 在軟體和硬體之間提供了簡潔、規則的介面。32 個 general-purpose register (x0-x31) 遵循 RISC 傳統，x0 硬連線為零——這個簡單功能使許多常見操作無需專用 instruction。Application Binary Interface (ABI) 為 register 分配慣用角色：a0-a7 用於 argument，t0-t6 用於 temporary，s0-s11 用於 saved register，ra 用於 return address。

Calling convention 平衡了效率和簡潔性。Caller-saved register（temporary 和 argument）允許 callee 自由使用它們而無需保存。Callee-saved register (s0-s11) 在 call 之間保留值，實現 long-lived variable。Stack pointer (sp) 和 frame pointer (s0/fp) 支援 local variable 和 nested call 的 stack frame。這個慣例實現了獨立編譯和高效的 function call。

Control and Status Register (CSR) 管理處理器狀態和配置。與 general-purpose register 不同，CSR 使用獨立的 12-bit address space 和專用 instruction（CSRRW、CSRRS、CSRRC）。CSR 按 privilege level 分區：machine mode CSR (0x300-0x3FF) 控制硬體，supervisor mode CSR (0x100-0x1FF) 支援 operating system，user mode CSR (0x000-0x0FF) 提供 performance counter 和其他 user-accessible state。

RISC-V 定義了三個 privilege level：Machine mode (M-mode) 具有完全硬體存取權限，處理初始化和 low-level exception。Supervisor mode (S-mode) 運行具有 virtual memory 和受控硬體存取的 operating system。User mode (U-mode) 以受限 privilege 運行 application。這種清晰的分離實現了從 embedded microcontroller（僅 M-mode）到完整 operating system (M+S+U) 的安全、高效系統。

與 ARM64 和 MIPS 相比，RISC-V 的 programmer's model 更簡潔、更一致。ARM64 有類似的 register 慣例，但有 x31 作為 zero register 和 stack pointer 的雙重角色等怪癖。MIPS 顯示其年齡，argument register 較少，命名不太一致。RISC-V 的獨立 CSR address space 比 ARM 的 system register encoding 或 MIPS 的 coprocessor 0 model 更簡潔。

Programmer's model 反映了 RISC-V 的設計哲學：簡潔性、規則性和關注點的清晰分離。這些原則使 RISC-V 比更複雜的架構更容易學習、實現和最佳化。

---

## Chapter Metadata

**Chapter**: 2 - Programmer's Model 與 Register Set
**Part**: II - Programmer's Model
**版本**: Draft v0p1（繁體中文版）
**字數**: ~6,500 字
**Section**: 6
**圖表**: 3
**程式碼範例**: 10+

**關鍵要點**：

1. 32 個 general-purpose register (x0-x31)，x0 硬連線為零
2. CSR 提供對 privileged state 的受控存取
3. 三個 privilege level (M/S/U) 及受控轉換
4. 標準 calling convention（8 個 argument register，caller/callee-saved）
5. Stack frame 遵循標準結構（prologue/epilogue 模式）

**技術亮點**：

- 完整的 register file 組織
- CSR address space 組織
- Privilege level 狀態轉換
- Function call 範例（prologue/epilogue）
- 比較表：RISC-V vs ARM64 vs MIPS
