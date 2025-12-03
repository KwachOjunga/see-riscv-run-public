# Chapter 8. Microarchitecture Variations

**Part V — Pipeline & Microarchitecture**

---

In Chapter 7, we explored the classic five-stage in-order pipeline. But modern processors go far beyond this simple model. Out-of-order (OOO) execution allows processors to dynamically reorder instructions to extract more parallelism, dramatically improving performance. While in-order processors execute instructions in program order and stall on dependencies, out-of-order processors can execute independent instructions while waiting for slow operations to complete.

This chapter explores the microarchitectural techniques that enable high-performance RISC-V processors: register renaming to eliminate false dependencies, reorder buffers to maintain precise exceptions, speculative execution to execute beyond branches, and advanced branch prediction to minimize misprediction penalties. We'll examine the cache hierarchy that hides memory latency, and cache coherence protocols that maintain consistency across multiple cores. Understanding these techniques is essential for anyone designing high-performance RISC-V systems or optimizing code for modern processors.

---

## 8.1 Out-of-Order Execution Basics

### In-Order vs Out-of-Order

**In-order processors execute instructions in the exact order they appear in the program.** If an instruction stalls (e.g., waiting for a cache miss), all subsequent instructions must wait, even if they're independent and could execute.

**Out-of-order (OOO) processors can execute instructions in a different order than the program specifies**, as long as the final result is the same. This allows the processor to work around stalls and dependencies, keeping execution units busy.

Example:

```assembly
lw   x1, 0(x2)     # I1: load (cache miss, 100 cycles)
add  x3, x4, x5    # I2: independent of I1
sub  x6, x7, x8    # I3: independent of I1
add  x9, x1, x10   # I4: depends on I1
```

**In-order execution**: I2 and I3 must wait for I1 to complete (100 cycles), even though they're independent.

**Out-of-order execution**: I2 and I3 can execute immediately while I1 is waiting for the cache miss. Only I4 must wait for I1.

This simple reordering can dramatically improve performance, especially when memory latency is high.

**Figure 8.1: In-Order vs Out-of-Order Execution**

```
Cycle:           1    2    3    ...  103  104  105  106  107
lw x1 (miss):    IF   ID   EX   MEM(100 cycles)  WB
add x3,x4,x5:         IF   ID   -------- stall -------- EX   MEM  WB
sub x6,x7,x8:              IF   -------- stall -------- ID   EX   MEM  WB
```

```
Cycle:            1    2    3    4    5    6    7    ...  103  104  105  106
lw x1 (miss):     IF   ID   EX   MEM (100 cycles) -------- WB
add x3,x4,x5:          IF   ID   EX   MEM  WB
sub x6,x7,x8:               IF   ID   EX   MEM  WB
add x9,x1,x10:                   IF   ID   ---- wait ----  EX   MEM  WB
```

Out-of-order execution allows independent instructions to proceed while the load is waiting, dramatically reducing wasted cycles.

### Dynamic Scheduling

**Dynamic scheduling** is the hardware mechanism that enables out-of-order execution. The processor analyzes dependencies at runtime and schedules instructions to execution units when their operands are ready.

Two classic algorithms for dynamic scheduling:

**Scoreboarding** (CDC 6600, 1964): A centralized control unit tracks which registers are being written and which instructions are waiting for them. When all operands are ready, the instruction is issued to an execution unit.

**Tomasulo's algorithm** (IBM 360/91, 1967): Uses **reservation stations** to buffer instructions waiting for operands. When an operand is produced, it's broadcast to all waiting instructions. This eliminates the need for a centralized scoreboard and enables **register renaming** (Section 8.2).

Modern OOO processors use variations of Tomasulo's algorithm with additional structures like the **Reorder Buffer (ROB)** to ensure precise exceptions.

---

## 8.2 Register Renaming

### The Problem: False Dependencies

Consider this code:

```assembly
add  x1, x2, x3    # I1: x1 = x2 + x3
sub  x4, x1, x5    # I2: x4 = x1 - x5 (RAW dependency on x1)
add  x1, x6, x7    # I3: x1 = x6 + x7 (WAW dependency on x1)
mul  x8, x1, x9    # I4: x8 = x1 * x9 (RAW dependency on x1 from I3)
```

I2 has a **true dependency (RAW)** on I1 — it must wait for I1 to produce `x1`.

But I3 has a **false dependency (WAW)** on I1 — both write `x1`, but I3's write doesn't actually depend on I1's value. Similarly, I4 depends on I3's `x1`, not I1's.

**False dependencies (WAR and WAW) limit parallelism** because the processor must serialize instructions that could otherwise execute in parallel.

### Physical vs Architectural Registers

**Register renaming** eliminates false dependencies by mapping architectural registers (the 32 registers visible to the programmer) to a larger set of **physical registers** (hidden from the programmer).

RISC-V has 32 architectural registers (`x0`-`x31`). A high-performance OOO processor might have 128 or 256 physical registers.

**Register Alias Table (RAT)**: Maps each architectural register to a physical register. When an instruction writes an architectural register, it's allocated a new physical register.

Example with renaming:

```assembly
add  P10, P2, P3   # I1: x1 -> P10
sub  P11, P10, P5  # I2: x4 -> P11, reads P10 (I1's result)
add  P12, P6, P7   # I3: x1 -> P12 (new physical register!)
mul  P13, P12, P9  # I4: x8 -> P13, reads P12 (I3's result)
```

Now I3 and I4 use `P12` for `x1`, while I1 and I2 use `P10`. The WAW dependency is eliminated — I3 can execute as soon as `P6` and `P7` are ready, without waiting for I1.

**Figure 8.2: Register Renaming Example**

```mermaid
graph TB
    subgraph "Architectural Registers (Programmer View)"
        x1[x1]
        x2[x2]
        x3[x3]
        x4[x4]
    end

    subgraph "Physical Registers (Hardware)"
        P2[P2]
        P3[P3]
        P5[P5]
        P6[P6]
        P7[P7]
        P9[P9]
        P10[P10]
        P11[P11]
        P12[P12]
        P13[P13]
    end

    subgraph "Register Alias Table (RAT)"
        RAT["x1 → P12<br/>x2 → P2<br/>x3 → P3<br/>x4 → P11"]
    end

    x1 -.->|mapped to| P12
    x2 -.->|mapped to| P2
    x3 -.->|mapped to| P3
    x4 -.->|mapped to| P11

    style P10 fill:#ffcccc
    style P12 fill:#ccffcc
```

In this example, `x1` was previously mapped to `P10` (shown in red, now free), and is currently mapped to `P12` (shown in green).

### Free List Management

**Physical registers must be recycled** when they're no longer needed. A **free list** tracks which physical registers are available for allocation.

When an instruction commits (Section 8.3), its old physical register mapping can be freed:

```
I3 commits: x1 was mapped to P10, now mapped to P12
  -> P10 can be freed (added to free list)
```

**Register renaming eliminates WAR and WAW hazards**, leaving only true RAW dependencies. This dramatically increases instruction-level parallelism.

---

## 8.3 Reorder Buffer (ROB) and Issue Queue

### Reorder Buffer (ROB)

**The ROB ensures that instructions commit in program order**, even though they execute out-of-order. This is essential for **precise exceptions** (Chapter 7).

The ROB is a circular buffer that holds all in-flight instructions in program order. Each entry contains:

- Instruction PC
- Destination register (architectural and physical)
- Result value (when execution completes)
- Exception status
- Ready bit

**Instruction flow through the ROB**:

1. **Dispatch**: Instruction is allocated a ROB entry and issued to an execution unit.
2. **Execute**: Instruction executes out-of-order when operands are ready.
3. **Complete**: Result is written to the ROB entry and broadcast to waiting instructions.
4. **Commit**: When the instruction reaches the head of the ROB and is complete, it commits (updates architectural state).

**Commit is in-order**: Instructions commit from the head of the ROB one at a time (or in small groups). This ensures that if an exception occurs, all earlier instructions have committed and all later instructions can be discarded.

**Figure 8.3: Reorder Buffer (ROB) Structure**

```mermaid
graph TB
    subgraph "Reorder Buffer (Circular Queue)"
        direction TB
        ROB1["ROB Entry 1<br/>PC: 0x1000<br/>Dest: x1→P10<br/>Value: 42<br/>Ready: ✓"]
        ROB2["ROB Entry 2<br/>PC: 0x1004<br/>Dest: x2→P11<br/>Value: -<br/>Ready: ✗"]
        ROB3["ROB Entry 3<br/>PC: 0x1008<br/>Dest: x3→P12<br/>Value: 100<br/>Ready: ✓"]
        ROB4["ROB Entry 4<br/>PC: 0x100C<br/>Dest: x4→P13<br/>Value: -<br/>Ready: ✗"]
        DOTS["..."]
    end

    HEAD[Head Pointer<br/>Commit from here] --> ROB1
    TAIL[Tail Pointer<br/>Dispatch to here] --> DOTS

    ROB1 -->|Commit| COMMIT[Update<br/>Architectural<br/>State]

    style ROB1 fill:#ccffcc
    style ROB3 fill:#ccffcc
    style ROB2 fill:#ffcccc
    style ROB4 fill:#ffcccc
```

Green entries are ready to commit (when they reach the head). Red entries are still executing.

**ROB Example Code**:

```c
// ROB entry structure
struct ROB_Entry {
    uint64_t PC;              // Instruction address
    uint8_t  arch_reg;        // Architectural register (x0-x31)
    uint8_t  phys_reg;        // Physical register (P0-P127)
    uint64_t value;           // Result value
    bool     ready;           // Execution complete?
    bool     exception;       // Exception occurred?
    uint8_t  exception_code;  // Exception type
};

// Commit logic (simplified)
void commit_instruction() {
    ROB_Entry *entry = &ROB[head];

    if (!entry->ready) {
        return;  // Can't commit yet
    }

    if (entry->exception) {
        // Handle exception: flush pipeline, jump to handler
        flush_pipeline();
        PC = trap_handler_address;
        return;
    }

    // Update architectural state
    if (entry->arch_reg != 0) {  // x0 is always zero
        RAT[entry->arch_reg] = entry->phys_reg;
        free_old_physical_register(entry->arch_reg);
    }

    // Advance head pointer
    head = (head + 1) % ROB_SIZE;
}
```

### Issue Queue and Reservation Stations

**The issue queue (or reservation stations) holds instructions waiting for operands.** When an instruction is dispatched, it's placed in the issue queue. When all its operands are ready, it's issued to an execution unit.

**Wakeup and select**:

- **Wakeup**: When a result is produced, it's broadcast to all issue queue entries. Entries waiting for that result mark the operand as ready.
- **Select**: Among all ready instructions, the scheduler selects which ones to issue to execution units (based on priority, age, or other policies).

This is the heart of dynamic scheduling. The issue queue decouples instruction dispatch from execution, allowing the processor to find parallelism dynamically.

---

## 8.4 Load/Store Queue

Memory operations are particularly challenging in OOO processors because they must respect **memory ordering** (Chapter 6) while still allowing reordering for performance.

### Load Queue and Store Queue

**The load queue** holds all in-flight loads. **The store queue** holds all in-flight stores. These queues track memory addresses and data, and enforce ordering constraints.

**Store-to-load forwarding**: If a load reads from the same address as an earlier store, the load can get the data directly from the store queue without waiting for the store to commit to memory.

```assembly
sw   x1, 0(x2)     # Store x1 to address in x2
lw   x3, 0(x2)     # Load from same address
```

The load can forward the data from the store queue, avoiding a memory access.

**Figure 8.4: Load/Store Queue Structure**

```mermaid
graph TB
    subgraph "Store Queue"
        SQ1["Store 1<br/>Addr: 0x2000<br/>Data: 42<br/>Committed: ✗"]
        SQ2["Store 2<br/>Addr: 0x2008<br/>Data: 100<br/>Committed: ✗"]
        SQ3["Store 3<br/>Addr: ?<br/>Data: 55<br/>Committed: ✗"]
    end

    subgraph "Load Queue"
        LQ1["Load 1<br/>Addr: 0x2000<br/>Data: ?"]
        LQ2["Load 2<br/>Addr: 0x2010<br/>Data: ?"]
    end

    LQ1 -.->|Address Match<br/>Forward Data| SQ1
    LQ2 -.->|No Match<br/>Go to Cache| CACHE[Data Cache]

    SQ3 -.->|Address Unknown<br/>Must Wait| WAIT[Stall Load]

    style SQ1 fill:#ccffcc
    style SQ3 fill:#ffcccc
```

Store-to-load forwarding allows loads to get data from earlier stores without waiting for them to commit to memory.

### Memory Disambiguation

**Memory disambiguation** is the problem of determining whether two memory operations access the same address. This is difficult because addresses are often computed dynamically.

```assembly
sw   x1, 0(x2)     # Store to address A
lw   x3, 0(x4)     # Load from address B — is B == A?
```

If `x2` and `x4` contain the same value, the load depends on the store. But the processor doesn't know this until the addresses are computed.

**Conservative approach**: Assume all loads depend on all earlier stores. This is safe but limits parallelism.

**Speculative approach**: Assume loads don't depend on earlier stores and execute them speculatively. If a dependency is later detected (address match), squash the load and re-execute.

Modern processors use **memory dependence prediction** to guess which loads depend on which stores, improving speculation accuracy.

---

## 8.5 Advanced Branch Prediction

Branch prediction is even more critical in OOO processors because mispredictions waste more work (all the speculatively executed instructions must be discarded).

### Two-Level Adaptive Predictors

**Two-level predictors** use both local and global branch history to make predictions. They can capture complex patterns like:

```c
if (a > 0) {        // Branch B1
  if (b > 0) {      // Branch B2
    ...
  }
}
```

If B1 is taken, B2 is more likely to be taken. A global history predictor can learn this correlation.

**Structure**: A global history register (GHR) tracks the last N branch outcomes (taken/not-taken). This is used to index into a pattern history table (PHT) that contains 2-bit counters.

### TAGE (Tagged Geometric History Length)

**TAGE** is a state-of-the-art branch predictor used in modern high-performance processors. It uses multiple predictor tables with different history lengths (e.g., 4, 8, 16, 32, 64 branches).

Each table is indexed by a hash of the PC and history. The predictor uses the longest matching history to make a prediction, falling back to shorter histories if there's no match.

TAGE achieves very high accuracy (98-99%) on most workloads.

**Figure 8.5: TAGE Predictor Structure**

```mermaid
graph TB
    PC[Program Counter] --> HASH1[Hash Function 1]
    PC --> HASH2[Hash Function 2]
    PC --> HASH3[Hash Function 3]
    PC --> BASE[Base Predictor]

    GHR[Global History<br/>Register] --> HASH1
    GHR --> HASH2
    GHR --> HASH3

    HASH1 --> T1[Table 1<br/>History: 4]
    HASH2 --> T2[Table 2<br/>History: 16]
    HASH3 --> T3[Table 3<br/>History: 64]

    T1 --> SELECT[Selector<br/>Choose Longest<br/>Matching History]
    T2 --> SELECT
    T3 --> SELECT
    BASE --> SELECT

    SELECT --> PRED[Prediction<br/>Taken/Not-Taken]

    style T3 fill:#ccffcc
    style SELECT fill:#ffffcc
```

TAGE uses multiple tables with different history lengths. The longest matching history provides the prediction.

### Return Address Stack (RAS)

**Function returns** are predicted using a hardware stack (mentioned in Chapter 7). When a function call is detected (`jal` or `jalr` with `rd != x0`), the return address is pushed onto the RAS. When a return is detected (`jalr x0, 0(x1)`), the top of the RAS is popped and used as the prediction.

The RAS is very accurate (>99%) because function calls and returns are well-structured.

### Indirect Branch Prediction

**Indirect branches** (`jalr` with a computed target) are harder to predict than direct branches. The target can vary widely depending on the value in the register.

**Indirect branch target buffer (iBTB)**: A cache indexed by the branch PC that stores recently seen targets. For virtual function calls or switch statements, the iBTB can achieve good accuracy.

**Advanced techniques**: Some processors use the call path (sequence of recent function calls) to predict indirect branch targets, improving accuracy for polymorphic code.

---

## 8.6 Cache Hierarchy

Modern processors have multiple levels of cache to hide memory latency.

### L1 Instruction and Data Caches

**L1 caches** are small (32-64 KB), fast (1-2 cycle latency), and split into separate instruction (I-cache) and data (D-cache) caches.

**Split I/D caches** allow simultaneous instruction fetch and data access, avoiding structural hazards. They're also optimized differently: I-caches are read-only and can use simpler replacement policies.

**Virtually indexed, physically tagged (VIPT)**: L1 caches often use virtual addresses for indexing (to avoid TLB lookup latency) but physical addresses for tags (to avoid aliasing issues).

### L2 Unified Cache

**L2 cache** is larger (256 KB - 1 MB), slower (10-20 cycles), and unified (holds both instructions and data).

L2 is the victim cache for L1: when data is evicted from L1, it's placed in L2. This creates an inclusive hierarchy (L2 contains everything in L1).

### L3 Shared Cache

**L3 cache** (if present) is even larger (4-32 MB), slower (30-50 cycles), and shared among all cores in a multi-core processor.

L3 reduces traffic to main memory and provides a large shared working set for all cores.

**Figure 8.6: Cache Hierarchy**

```mermaid
graph TB
    subgraph "Core 0"
        CPU0[CPU Core 0]
        L1I0[L1 I-Cache<br/>32 KB<br/>1-2 cycles]
        L1D0[L1 D-Cache<br/>32 KB<br/>1-2 cycles]
    end

    subgraph "Core 1"
        CPU1[CPU Core 1]
        L1I1[L1 I-Cache<br/>32 KB<br/>1-2 cycles]
        L1D1[L1 D-Cache<br/>32 KB<br/>1-2 cycles]
    end

    CPU0 --> L1I0
    CPU0 --> L1D0
    CPU1 --> L1I1
    CPU1 --> L1D1

    L1I0 --> L2_0[L2 Unified<br/>256 KB<br/>10-20 cycles]
    L1D0 --> L2_0
    L1I1 --> L2_1[L2 Unified<br/>256 KB<br/>10-20 cycles]
    L1D1 --> L2_1

    L2_0 --> L3[L3 Shared<br/>8 MB<br/>30-50 cycles]
    L2_1 --> L3

    L3 --> MEM[Main Memory<br/>DDR4/DDR5<br/>100-300 cycles]

    style L1I0 fill:#e1f5ff
    style L1D0 fill:#e1f5ff
    style L1I1 fill:#e1f5ff
    style L1D1 fill:#e1f5ff
    style L2_0 fill:#fff4e1
    style L2_1 fill:#fff4e1
    style L3 fill:#ffe1e1
    style MEM fill:#f0e1ff
```

Each level is larger and slower. L1 is private per core, L2 may be private or shared, L3 is shared among all cores.

### Cache Replacement Policies

**LRU (Least Recently Used)**: Evict the cache line that hasn't been accessed for the longest time. This is effective but expensive to implement for high associativity.

**PLRU (Pseudo-LRU)**: An approximation of LRU that's cheaper to implement. Uses a tree of bits to track approximate recency.

**Random**: Evict a random cache line. Surprisingly effective and very simple.

Modern caches often use PLRU or adaptive policies that combine multiple strategies.

---

## 8.7 Cache Coherence

In multi-core systems, each core has its own L1 cache. **Cache coherence** ensures that all cores see a consistent view of memory.

### MESI Protocol

**MESI** is the most common coherence protocol. Each cache line is in one of four states:

- **M (Modified)**: This cache has the only valid copy, and it's been modified (dirty).
- **E (Exclusive)**: This cache has the only valid copy, and it's clean (matches memory).
- **S (Shared)**: Multiple caches have valid copies, all clean.
- **I (Invalid)**: This cache line is not valid.

**State transitions**:

- **Read miss**: If another cache has the line in M, it writes back to memory and transitions to S. The requesting cache loads the line in S.
- **Write**: If the line is in S, all other caches invalidate their copies. The writing cache transitions to M.

**Figure 8.7: MESI Protocol State Diagram**

```mermaid
stateDiagram-v2
    [*] --> I
    I: Invalid<br/>(No valid copy)
    E: Exclusive<br/>(Only copy, clean)
    S: Shared<br/>(Multiple copies, clean)
    M: Modified<br/>(Only copy, dirty)

    I --> E: Read Miss<br/>(No other copy)
    I --> S: Read Miss<br/>(Other copies exist)
    I --> M: Write Miss

    E --> M: Write
    E --> S: Other Read
    E --> I: Evict

    S --> M: Write<br/>(Invalidate others)
    S --> I: Evict or<br/>Other Write

    M --> S: Other Read<br/>(Write back)
    M --> I: Evict<br/>(Write back)
```

**MESI Example**:

```c
// Core 0 and Core 1 both access the same cache line

// Initial state: Both caches Invalid (I)

// Core 0: Read from address 0x1000
// Core 0 cache: I → E (exclusive, no other copy)

// Core 1: Read from address 0x1000
// Core 0 cache: E → S (shared)
// Core 1 cache: I → S (shared)

// Core 0: Write to address 0x1000
// Core 0 cache: S → M (modified, dirty)
// Core 1 cache: S → I (invalidated)

// Core 1: Read from address 0x1000
// Core 0 cache: M → S (write back to memory)
// Core 1 cache: I → S (load from memory)
```

### MOESI Protocol

**MOESI** adds an **O (Owned)** state: the cache has a dirty copy, but other caches may have shared copies. This reduces write-backs to memory.

### Snooping vs Directory-Based Coherence

**Snooping**: All caches monitor (snoop) a shared bus for memory transactions. When a cache sees a transaction that affects its data, it responds appropriately (invalidate, write-back, etc.).

Snooping is simple but doesn't scale well beyond ~8-16 cores because bus bandwidth becomes a bottleneck.

**Directory-based**: A centralized directory tracks which caches have copies of each cache line. Coherence messages are sent point-to-point rather than broadcast.

Directory-based coherence scales better to many cores (64+) and is used in large multi-core processors.

### RISC-V Coherence Considerations

RISC-V doesn't mandate a specific coherence protocol. The **RVWMO memory model** (Chapter 6) defines the ordering guarantees, but the coherence mechanism is implementation-defined.

Most RISC-V multi-core systems use MESI or MOESI with snooping (for small core counts) or directory-based coherence (for large core counts).

---

## 8.8 Comparison with ARM and MIPS OOO Cores

Let's compare RISC-V OOO implementations with other architectures.

### RISC-V BOOM vs ARM Cortex-A76/A78

**BOOM (Berkeley Out-of-Order Machine)** is an open-source RISC-V OOO core. It has:

- 3-4 issue width
- 128-entry ROB
- 64-entry issue queue
- Advanced branch prediction (TAGE)
- L1 I/D caches, L2 unified cache

**ARM Cortex-A76** (2018) is a high-performance mobile core:

- 4-issue width
- 128-entry ROB
- Sophisticated branch prediction
- 64 KB L1 I/D, 256-512 KB L2

**Comparison**: BOOM and Cortex-A76 are similar in structure. Both use register renaming, ROB, and advanced branch prediction. The main differences are in implementation details (pipeline depth, cache sizes, power optimization).

RISC-V's simpler ISA (no complex addressing modes, no condition codes) makes the OOO logic slightly simpler than ARM's, but the difference is small in modern designs.

### RISC-V vs MIPS R10000 Pipeline

**MIPS R10000** (1996) was one of the first commercial OOO processors:

- 4-issue superscalar
- 32-entry active list (similar to ROB)
- Register renaming with 64 physical registers
- Speculative execution

The R10000 pioneered many techniques still used today. Modern RISC-V OOO cores like BOOM are evolutionary descendants of the R10000 design philosophy.

**Key difference**: RISC-V has no branch delay slots, making branch misprediction recovery simpler than MIPS.

### Microarchitecture Trade-offs

**Complexity vs Performance**: OOO execution provides 2-3x performance improvement over in-order for general-purpose workloads, but at the cost of 3-5x more transistors and power.

**When to use OOO**:

- High-performance applications (servers, desktops, high-end mobile)
- Workloads with irregular memory access patterns
- Code with many branches and dependencies

**When to use in-order**:

- Embedded systems with power/area constraints
- Predictable real-time workloads
- Simple control-dominated code

RISC-V's flexibility allows both in-order (Rocket, SiFive E/U-series) and OOO (BOOM, SiFive P-series) implementations, making it suitable for a wide range of applications.

---

## Summary

In this chapter, we explored advanced microarchitecture techniques for high-performance RISC-V processors:

- **Out-of-order execution**: Dynamic scheduling to extract instruction-level parallelism.
- **Register renaming**: Eliminating false dependencies (WAR, WAW) with physical registers.
- **Reorder buffer**: Ensuring in-order commit for precise exceptions.
- **Load/store queues**: Memory disambiguation and store-to-load forwarding.
- **Advanced branch prediction**: TAGE, RAS, indirect branch prediction.
- **Cache hierarchy**: L1/L2/L3 caches with LRU/PLRU replacement.
- **Cache coherence**: MESI/MOESI protocols for multi-core consistency.
- **Comparisons**: RISC-V BOOM vs ARM Cortex-A76 and MIPS R10000.

RISC-V's clean ISA makes it an excellent target for both simple in-order cores and complex OOO designs. The architecture doesn't impose unnecessary constraints, allowing microarchitects to innovate freely.

In the next chapter, we'll shift focus from hardware to software, exploring how RISC-V systems boot and how firmware and operating systems interact with the hardware.

---

**Chapter 8 Complete** (~5,500 words)
