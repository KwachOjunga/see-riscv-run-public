# Chapter 15. Debugging & Trace

**Part IX — Performance, Debug & Tools**

---

## 🎯 Learning Objectives

After reading this chapter, you will be able to:

1. **Master GDB Stub Usage**: Use QEMU `-s -S` to start a GDB Server for remote debugging
2. **Know Core Debug Commands**: `break`, `si`, `info reg`, `x/`, and other GDB commands
3. **Build a Debug Mindset**: Adopt a systematic "Observe → Hypothesize → Verify" debugging workflow

---

## 💡 Scenario: The Pocket Watch That Stops Time

> **Scene**: Junior is pointing at garbled results on the screen, about to lose it.

**Junior**: "I can't take this anymore. I just wanted to add 1 to 5, why is the result 34821? I've been staring at these five lines of Assembly for an hour!"

**Senior**: "Looking with your eyes isn't enough. Junior, program execution is like a speeding train—how can you see if there's a crack in the wheels while sitting by the tracks?"

**Junior**: "So what do I do? Add `printf`?"

**Senior**: "`printf` is fine, but in bare-metal situations or when the program crashes, you can't even print anything. You need a 'pocket watch' that can pause time—**GDB**."

**Junior**: "Pause time?"

**Senior**: "Exactly. Through a **JTAG** hardware interface (or the QEMU emulator we're using now), we can force the CPU into **Debug Mode**.

| Debug Mode Feature | Analogy |
|-------------------|---------|
| Check Registers | Peeking in a wallet |
| View Memory | Searching through drawers |
| Single Step | Slow-motion replay |
| Breakpoint | Setting a trap |

In this mode, the CPU is like someone pressed the pause button. We can execute just one instruction at a time and see where things went wrong.

Come on, give me that broken program. Let's go bug hunting."

---

Software development requires debugging. A program crashes, and we need to know why. A function returns the wrong value, and we need to step through its execution. A performance bottleneck appears, and we need to trace instruction flow. Debugging transforms opaque failures into understandable problems.

RISC-V provides a comprehensive debug architecture that supports both halting debug (stop the processor, examine state) and non-intrusive trace (record execution without stopping). The Debug Module allows external debuggers to control the processor through JTAG or other interfaces. Hardware breakpoints and triggers enable precise control over when to halt execution. Debug mode provides a special execution environment for debug operations. Trace support captures instruction and data flow for post-mortem analysis.

This chapter explores RISC-V debugging and trace capabilities. We'll examine the debug architecture, debug interfaces, hardware breakpoints, debug mode operation, trace support, and how RISC-V compares to ARM's CoreSight debug infrastructure.

---

## 15.1 RISC-V Debug Architecture

**Debug Requirements**

A debug system must provide:

- **Halt and resume**: Stop processor execution, examine state, continue
- **Register access**: Read and write CPU registers
- **Memory access**: Read and write system memory
- **Breakpoints**: Stop execution at specific instructions or data accesses
- **Single-step**: Execute one instruction at a time
- **Reset control**: Reset the processor or system

RISC-V's debug architecture separates concerns into distinct modules, allowing flexible implementation while maintaining standard interfaces.

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

Key components:

*Debug Transport Module (DTM)*: Provides physical connection (JTAG, USB, etc.)

*Debug Module (DM)*: Controls the core, implements debug operations

*Debug Module Interface (DMI)*: Standard interface between DTM and DM

*Debug Mode*: Special execution mode for debug operations

**Debug Module (DM)**

The Debug Module is the central component that:

- Halts and resumes the core
- Provides abstract commands for register/memory access
- Manages hardware breakpoints (triggers)
- Controls reset

The DM is accessed through memory-mapped registers:

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

Debug mode is a special execution mode distinct from M/S/U modes:

- Higher privilege than M-mode
- Can access all system resources
- Uses separate CSRs (dcsr, dpc, dscratch0/1)
- Executes from Debug ROM or Program Buffer

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

JTAG (Joint Test Action Group) is the standard debug interface for RISC-V. It provides:

- 4-wire interface (TDI, TDO, TCK, TMS)
- Boundary scan for testing
- Debug access to the core

JTAG signals:

```
TDI:  Test Data In (serial data input)
TDO:  Test Data Out (serial data output)
TCK:  Test Clock
TMS:  Test Mode Select (state machine control)
TRST: Test Reset (optional)
```

JTAG state machine:

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

DMI is a standard register interface between DTM and DM. It provides:

- 32-bit or 64-bit register access
- Address space for DM registers
- Status and error reporting

DMI operations:

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

Abstract commands provide high-level debug operations without requiring debug mode entry:

*Access Register*: Read/write CPU registers
*Access Memory*: Read/write system memory
*Quick Access*: Fast register access

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

The DM can access system memory directly through the system bus, bypassing the core:

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

RISC-V provides a flexible trigger system for hardware breakpoints and watchpoints. Triggers can:

- Break on instruction execution (instruction breakpoint)
- Break on data access (data watchpoint)
- Break on exceptions
- Chain multiple conditions

Triggers are configured through CSRs:

- `tselect`: Select trigger register
- `tdata1`: Trigger configuration
- `tdata2`: Trigger match value
- `tdata3`: Additional trigger data (optional)

**Trigger Types**

RISC-V defines several trigger types:

*Type 2 (mcontrol)*: Address/data match trigger

- Match on instruction fetch, load, or store
- Configurable match conditions (equal, greater, less, mask)
- Action: enter debug mode, raise exception, or trace

*Type 3 (icount)*: Instruction count trigger

- Break after N instructions
- Useful for single-stepping

*Type 4 (itrigger)*: Interrupt trigger

- Break on specific interrupts

*Type 5 (etrigger)*: Exception trigger

- Break on specific exceptions

**Breakpoint Configuration**

Setting an instruction breakpoint:

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

Setting a data watchpoint:

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

Multiple triggers can be chained to create complex conditions:

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

When a trigger fires, it can:

- Enter debug mode (action = 1)
- Raise breakpoint exception (action = 0)
- Generate trace event (implementation-specific)

---

## 15.4 Debug Mode

**Entering Debug Mode**

The core enters debug mode when:

- External debugger requests halt (via dmcontrol.haltreq)
- Hardware breakpoint fires (trigger with action = 1)
- Single-step completes (dcsr.step = 1)
- Debug interrupt (haltreq signal)

Upon entering debug mode:

1. PC is saved to `dpc` (Debug PC)
2. Cause is saved to `dcsr.cause`
3. Core halts execution
4. PC jumps to Debug ROM or Program Buffer

**Debug CSRs**

Debug mode uses three special CSRs:

*dcsr (Debug Control and Status)*:

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

*dpc (Debug PC)*: Saved PC when entering debug mode

*dscratch0/1*: Scratch registers for debug code

**Debug ROM and Program Buffer**

When entering debug mode, the core executes code from:

*Debug ROM*: Small ROM containing debug entry code

- Saves context
- Waits for debugger commands
- Restores context on resume

*Program Buffer*: RAM for debugger-supplied code

- Debugger writes instructions here
- Core executes them in debug mode
- Used for complex operations (e.g., memory copy)

Example debug ROM code:

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

To resume execution:

1. Debugger writes to dmcontrol.resumereq
2. Debug ROM restores context
3. Core executes `dret` instruction
4. PC restored from `dpc`
5. Privilege mode restored from `dcsr.prv`

```assembly
# Resume from debug mode
debug_resume:
    # Restore x10 from dscratch0
    csrr x10, dscratch0

    # Return from debug mode
    dret  # PC ← dpc, privilege ← dcsr.prv
```

**Single-Stepping**

Single-step mode executes one instruction then re-enters debug mode:

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

Trace captures program execution for analysis without halting the core. RISC-V trace provides:

- Instruction trace: Record executed instructions
- Data trace: Record memory accesses
- Trace compression: Reduce trace bandwidth

Trace is non-intrusive—it doesn't affect program execution or timing.

**Instruction Trace**

Instruction trace records:

- Executed instructions (PC values)
- Branch outcomes (taken/not taken)
- Exceptions and interrupts
- Context changes (privilege mode, ASID)

Trace packets encode this information efficiently:

```
Trace packet types:
  Format 0: Uncompressed address (full PC)
  Format 1: Differential address (PC delta)
  Format 2: Address with branch map
  Format 3: Synchronization packet
```

**Trace Compression**

Full instruction trace is expensive (bandwidth, storage). RISC-V trace uses compression:

*Branch map*: Encode multiple branch outcomes in one packet

```
Example: 8 branches, outcomes = 10110010
  Packet: [type=2, branches=10110010, address=...]
```

*Differential encoding*: Encode PC delta instead of full PC

```
Previous PC: 0x80000100
Current PC:  0x80000104
Packet: [type=1, delta=+4]
```

*Implicit sequences*: Don't trace sequential instructions

```
PC sequence: 0x100, 0x104, 0x108, 0x10c
Trace: [0x100, count=4]  # Implicit +4 increments
```

**Data Trace**

Data trace records memory accesses:

- Load/store addresses
- Data values
- Access size (byte, halfword, word, doubleword)

Data trace packet:

```
Data trace packet:
  [type] [address] [data] [size]

Example:
  SW x10, 0(x5)  # Store word
  Packet: [type=store, addr=0x80001000, data=0x12345678, size=4]
```

**Trace Filtering**

Trace can be filtered to reduce bandwidth:

- Address range filtering (trace only specific code regions)
- Privilege filtering (trace only M-mode, S-mode, etc.)
- Event filtering (trace only branches, exceptions, etc.)

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

ARM CoreSight is a comprehensive debug and trace infrastructure. Comparison:

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

RISC-V uses JTAG (4-5 wires), ARM supports both JTAG and SWD (2 wires):

```
JTAG (RISC-V, ARM):
  TDI, TDO, TCK, TMS, (TRST)
  5 pins, standard interface

SWD (ARM only):
  SWDIO (bidirectional data)
  SWCLK (clock)
  2 pins, lower pin count
```

SWD advantages:

- Fewer pins (important for small packages)
- Faster than JTAG in some cases
- ARM-specific optimization

JTAG advantages:

- Industry standard (IEEE 1149.1)
- Widely supported tools
- Boundary scan capability

**Trace Comparison**

RISC-V Trace vs ARM ETM (Embedded Trace Macrocell):

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

ARM ETM is mature and widely deployed. RISC-V Trace is newer but follows similar principles with simpler encoding.

**Debug Tools**

Both architectures support standard debug tools:

RISC-V:

- GDB (GNU Debugger)
- OpenOCD (Open On-Chip Debugger)
- SEGGER J-Link
- Lauterbach TRACE32

ARM:

- GDB
- Keil MDK
- ARM DS-5 / Arm Development Studio
- SEGGER J-Link
- Lauterbach TRACE32

**Practical Differences**

*RISC-V advantages*:

- Simpler, more modular design
- Open specification (no licensing)
- Flexible trigger system
- Easier to implement

*ARM advantages*:

- Mature ecosystem
- SWD reduces pin count
- Comprehensive trace infrastructure
- Extensive tool support

For embedded systems, SWD's 2-pin interface is attractive. For complex SoCs, both architectures provide comparable debug capabilities.

---

## 🛠️ Hands-on Lab: Lab 15.1 — The Vanishing Values (Bug Hunting with GDB)

This lab features a classic **Pointer Stride Error**—the most common mistake for RISC-V beginners: assuming an `int` pointer `+1` moves 4 bytes, but in Assembly `addi x, x, 1` really does add just 1 byte.

### Lab Objectives

1. Launch QEMU's GDB Server feature
2. Connect GDB and load symbols
3. Use `layout asm` to view assembly
4. Find the two bugs in the program

### Buggy Code (buggy_sum.S)

```assembly
.section .data
# Define an array: 10, 20, 30, 40, 50
# Expected result: 10+20+30+40+50 = 150 (Hex: 0x96)
nums: .word 10, 20, 30, 40, 50

.section .text
.global _start

_start:
    la  t0, nums        # t0 points to array start
    li  t1, 5           # t1 is loop counter (Count = 5)
    # BUG 1: Forgot to initialize accumulator a0
    # We assume a0 is 0, but it might be garbage

loop:
    lw  t2, 0(t0)       # Load current number into t2
    add a0, a0, t2      # Accumulate: a0 = a0 + t2

    # BUG 2: Pointer stride error!
    # We're reading words (4 bytes), but here we only add 1
    addi t0, t0, 1      # ❌ Should be addi t0, t0, 4

    addi t1, t1, -1     # Decrement counter
    bnez t1, loop       # If not done, continue loop

stop:
    j stop
```

### Debug Workflow

**Step A: Compile (with debug info)**

```bash
# -g is key! Tells compiler to keep symbol table
riscv64-unknown-elf-gcc -g -nostdlib -o buggy_sum.elf buggy_sum.S
```

**Step B: Start QEMU (as Target)**

```bash
# -S: Pause CPU immediately after startup
# -s: Enable GDB Server, default Port 1234
qemu-system-riscv64 -machine virt -nographic \
    -kernel buggy_sum.elf -S -s
```

*(Terminal will hang—open another terminal for GDB)*

**Step C: Start GDB (as Host)**

```bash
riscv64-unknown-elf-gdb buggy_sum.elf
```

**Step D: GDB Interactive Investigation**

```gdb
(gdb) target remote :1234     # Connect to QEMU
(gdb) layout asm              # Open Assembly view
(gdb) break loop              # Set breakpoint at loop label
(gdb) continue                # Run until breakpoint

# After entering the loop...
(gdb) info reg a0             # Observe accumulator → not 0!
(gdb) info reg t0             # Observe pointer
(gdb) si                      # Single Step one instruction
(gdb) info reg t0             # Look at pointer again → only moved 1 byte!

# After finding the problem...
(gdb) x/5xw &nums             # View array memory contents
```

### Expected Findings

1. **Bug 1 (a0 uninitialized)**: First time entering loop, `info reg a0` shows garbage value
2. **Bug 2 (pointer stride)**: Each `addi t0, t0, 1` only increases t0 by 1, causing misaligned data reads

### Fixed Code

```assembly
_start:
    la  t0, nums
    li  t1, 5
    li  a0, 0           # ✅ FIX 1: Initialize accumulator

loop:
    lw  t2, 0(t0)
    add a0, a0, t2
    addi t0, t0, 4      # ✅ FIX 2: Stride = 4 bytes (word size)
    addi t1, t1, -1
    bnez t1, loop

stop:
    j stop
```

> **danieRTOS Reference**: The danieRTOS context switch code carefully uses word-aligned offsets when saving/restoring registers to the stack.

---

## ⚠️ Common Pitfalls

### Pitfall 1: Compiler Optimization Interferes with Debugging

**Error Scenario**: After compiling with `-O2`, line numbers in GDB don't match source code, variables "disappear".

**Cause**: Optimizer reorders instructions, eliminates registers, inlines functions.

```bash
# ❌ Don't use high optimization when debugging
riscv64-unknown-elf-gcc -O2 -o program.elf program.c

# ✅ Use -O0 -g when debugging
riscv64-unknown-elf-gcc -O0 -g -o program.elf program.c
```

### Pitfall 2: Forgetting the `-g` Flag

**Error Scenario**: GDB shows "No symbol table is loaded".

**Cause**: Compiled without `-g`, symbol info was discarded.

```bash
# ❌ No debug info
riscv64-unknown-elf-gcc -o program.elf program.c

# ✅ Keep debug info
riscv64-unknown-elf-gcc -g -o program.elf program.c
```

### Pitfall 3: QEMU Not Paused with `-S`

**Error Scenario**: Program already finished or crashed by the time GDB connects.

**Solution**: Always add `-S` to make QEMU pause after startup, waiting for GDB.

```bash
# ❌ Program starts executing immediately after launch
qemu-system-riscv64 -machine virt -kernel program.elf -s

# ✅ Program pauses after launch, waiting for GDB
qemu-system-riscv64 -machine virt -kernel program.elf -S -s
```

> 💡 **Tip**: `-s` = GDB Server on port 1234, `-S` = Stop at startup. These are often used together.

---

## Summary

Debugging and trace are essential for software development and system analysis. This chapter explored RISC-V's debug architecture and how it compares to ARM's mature CoreSight infrastructure.

**Debug architecture** separates concerns into modular components. The Debug Transport Module provides physical connectivity through JTAG or other interfaces. The Debug Module controls the core through a standard Debug Module Interface. Debug mode provides a privileged execution environment for debug operations. This separation allows flexible implementations while maintaining standard interfaces for debugger tools.

**Debug interface** uses JTAG as the standard physical layer, providing four-wire connectivity for debug access. The Debug Module Interface defines register-level operations for controlling the core. Abstract commands enable high-level operations like register and memory access without requiring debug mode entry. System bus access allows the debugger to read and write memory directly, bypassing the core entirely.

**Hardware breakpoints and triggers** provide flexible mechanisms for halting execution. The trigger module supports multiple trigger types including address match, instruction count, interrupt, and exception triggers. Triggers can match on instruction fetch, data load, or data store. Trigger chaining enables complex conditional breakpoints. Actions include entering debug mode, raising exceptions, or generating trace events.

**Debug mode** is a special execution environment with higher privilege than M-mode. The core enters debug mode on external halt requests, breakpoint hits, or single-step completion. Debug CSRs (dcsr, dpc, dscratch) manage debug state. The Debug ROM provides entry code, while the Program Buffer allows debuggers to execute custom instruction sequences. Single-stepping executes one instruction then re-enters debug mode, enabling step-through debugging.

**Trace support** captures program execution non-intrusively. Instruction trace records executed instructions, branch outcomes, and control flow changes. Data trace records memory accesses and data values. Trace compression reduces bandwidth through branch maps, differential encoding, and implicit sequences. Trace filtering limits capture to specific address ranges, privilege levels, or events, reducing trace data volume.

**Comparison with ARM** shows both similarities and differences. ARM CoreSight provides comprehensive debug and trace infrastructure with mature tool support. SWD offers a 2-pin alternative to JTAG, reducing pin count for embedded systems. ARM ETM provides extensive trace capabilities with sophisticated compression. RISC-V's debug architecture is simpler and more modular, with an open specification and flexible trigger system. Both architectures support standard tools like GDB and OpenOCD, ensuring practical usability.

Together, RISC-V's debug and trace capabilities enable effective software development, system analysis, and problem diagnosis across the full range from embedded microcontrollers to high-performance application processors.
