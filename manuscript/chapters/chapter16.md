# Chapter 16. Performance Counters & PMU

**Part IX — Performance, Debug & Tools**

---

Performance optimization requires measurement. A program runs slowly, and we need to know why. Cache misses dominate execution time, or branch mispredictions cause pipeline stalls, or memory bandwidth limits throughput. Performance counters transform vague slowness into quantifiable bottlenecks.

RISC-V provides a Performance Monitoring Unit (PMU) through a set of hardware performance counters. These counters track events like cycles executed, instructions retired, cache hits and misses, branch predictions, and TLB accesses. The basic counters (cycle, instret, time) are mandatory and provide fundamental metrics. Hardware performance counters (mhpmcounter3-31) are optional and track implementation-specific events. Together, these counters enable profiling, bottleneck identification, and performance analysis.

This chapter explores RISC-V performance counters and the PMU. We'll examine the counter architecture, basic counters, hardware performance counters, performance events, profiling techniques, and how RISC-V compares to ARM's PMU.

---

## 16.1 Performance Counter Architecture

**Performance Monitoring Overview**

Performance monitoring answers questions like:

- How many cycles did this function take?
- What is the IPC (instructions per cycle)?
- How many cache misses occurred?
- How many branches were mispredicted?
- Where is the performance bottleneck?

RISC-V performance counters provide hardware-based measurement with minimal overhead. Counters increment automatically on specific events, allowing precise measurement without software instrumentation.

**Counter CSRs**

RISC-V defines performance counter CSRs in three privilege levels:

*Machine-mode counters* (M-mode only):

- `mcycle`: Machine cycle counter
- `minstret`: Machine instructions-retired counter
- `mhpmcounter3-31`: Machine hardware performance counters (29 counters)

*Supervisor/User-mode counters* (readable from S/U-mode):

- `cycle`: Cycle counter (shadow of mcycle)
- `instret`: Instructions-retired counter (shadow of minstret)
- `hpmcounter3-31`: Hardware performance counters (shadow of mhpmcounter)

*Time counter*:

- `time`: Real-time counter (wall-clock time)

For RV32, each counter has a high-word CSR (e.g., `mcycleh`, `cycleh`) for 64-bit values.

**Counter Privilege Levels**

Counters are accessible based on privilege:

```
M-mode: Can read/write all counters (mcycle, minstret, mhpmcounter)
S-mode: Can read cycle, instret, hpmcounter (if enabled)
U-mode: Can read cycle, instret, hpmcounter (if enabled)
```

Access control via `mcounteren` and `scounteren`:

```c
// Enable cycle and instret for S-mode and U-mode
uint64_t mcounteren = (1 << 0) | (1 << 2);  // CY, IR
write_csr(mcounteren, mcounteren);

// Enable cycle and instret for U-mode (from S-mode)
uint64_t scounteren = (1 << 0) | (1 << 2);
write_csr(scounteren, scounteren);
```

**Counter Inhibit**

Counters can be inhibited (stopped) via `mcountinhibit`:

```c
// Stop cycle and instret counters
uint64_t mcountinhibit = (1 << 0) | (1 << 2);  // CY, IR
write_csr(mcountinhibit, mcountinhibit);

// Resume counters
write_csr(mcountinhibit, 0);
```

This is useful for:

- Measuring specific code regions
- Reducing power consumption
- Preventing counter overflow

---

## 16.2 Basic Performance Counters

**mcycle / cycle (Cycle Counter)**

The cycle counter tracks the number of clock cycles executed by the hart:

```c
// Read cycle counter
uint64_t start = read_csr(cycle);
// ... code to measure ...
uint64_t end = read_csr(cycle);
uint64_t cycles = end - start;

printf("Cycles: %llu\n", cycles);
```

For RV32, use cycleh for the high 32 bits:

```c
// RV32: Read 64-bit cycle counter
uint64_t read_cycle_rv32(void) {
    uint32_t hi, lo, hi2;
    do {
        hi = read_csr(cycleh);
        lo = read_csr(cycle);
        hi2 = read_csr(cycleh);
    } while (hi != hi2);  // Retry if high word changed
    
    return ((uint64_t)hi << 32) | lo;
}
```

**minstret / instret (Instructions Retired Counter)**

The instructions-retired counter tracks the number of instructions completed:

```c
// Read instret counter
uint64_t start = read_csr(instret);
// ... code to measure ...
uint64_t end = read_csr(instret);
uint64_t instructions = end - start;

printf("Instructions: %llu\n", instructions);
```

**IPC Calculation**

Combining cycle and instret gives IPC (instructions per cycle):

```c
// Measure IPC
uint64_t cycles_start = read_csr(cycle);
uint64_t instret_start = read_csr(instret);

// ... code to measure ...

uint64_t cycles_end = read_csr(cycle);
uint64_t instret_end = read_csr(instret);

uint64_t cycles = cycles_end - cycles_start;
uint64_t instructions = instret_end - instret_start;

double ipc = (double)instructions / cycles;
printf("IPC: %.2f\n", ipc);
```

IPC interpretation:

- IPC close to 1: Good utilization (in-order core)
- IPC > 1: Superscalar execution (out-of-order core)
- IPC < 1: Pipeline stalls (cache misses, branch mispredicts, etc.)

**time (Real-Time Counter)**

The time counter provides wall-clock time:

```c
// Read time counter
uint64_t start_time = read_csr(time);
// ... code to measure ...
uint64_t end_time = read_csr(time);
uint64_t elapsed = end_time - start_time;

// Convert to microseconds (assuming 1 MHz time counter)
printf("Elapsed time: %llu us\n", elapsed);
```

The time counter frequency is platform-specific (typically 1 MHz or 10 MHz). It's useful for:

- Wall-clock timing
- Timeout implementation
- Real-time scheduling

**Difference: cycle vs time**

- `cycle`: Counts CPU cycles (stops during sleep, varies with frequency scaling)
- `time`: Counts real time (continues during sleep, constant frequency)

```c
// Example: Measure sleep overhead
uint64_t cycles_before = read_csr(cycle);
uint64_t time_before = read_csr(time);

wfi();  // Sleep until interrupt

uint64_t cycles_after = read_csr(cycle);
uint64_t time_after = read_csr(time);

printf("Cycles during sleep: %llu\n", cycles_after - cycles_before);  // ~0
printf("Time during sleep: %llu\n", time_after - time_before);        // > 0
```

---

## 16.3 Hardware Performance Counters

**mhpmcounter3-31 (Hardware Performance Counters)**

RISC-V provides up to 29 hardware performance counters (HPM counters) for tracking implementation-specific events. These counters are optional—implementations may provide 0 to 29 counters.

Counter CSRs:

- `mhpmcounter3-31`: M-mode counters (29 counters)
- `hpmcounter3-31`: S/U-mode readable counters (shadows of mhpmcounter)
- `mhpmevent3-31`: Event selection registers

**Event Selection (mhpmevent CSRs)**

Each HPM counter has an associated event selector:

```c
// Configure mhpmcounter3 to count L1 I-cache misses
write_csr(mhpmevent3, EVENT_L1_ICACHE_MISS);

// Reset counter
write_csr(mhpmcounter3, 0);

// ... code to measure ...

// Read counter
uint64_t icache_misses = read_csr(mhpmcounter3);
printf("L1 I-cache misses: %llu\n", icache_misses);
```

Event codes are implementation-specific. Common events include:

- Cache events (L1/L2 hits, misses)
- Branch events (taken, not-taken, mispredicted)
- Pipeline events (stalls, flushes)
- Memory events (loads, stores, TLB misses)

**Counter Overflow Handling**

Counters are 64-bit and rarely overflow. If overflow is a concern:

```c
// Check for overflow (counter wrapped around)
uint64_t start = read_csr(mhpmcounter3);
// ... code ...
uint64_t end = read_csr(mhpmcounter3);

if (end < start) {
    // Overflow occurred
    uint64_t count = (UINT64_MAX - start) + end + 1;
} else {
    uint64_t count = end - start;
}
```

Some implementations support overflow interrupts (implementation-specific).

**Example: Multi-Counter Measurement**

Measuring multiple events simultaneously:

```c
// Configure counters
write_csr(mhpmevent3, EVENT_L1_DCACHE_MISS);
write_csr(mhpmevent4, EVENT_L2_CACHE_MISS);
write_csr(mhpmevent5, EVENT_BRANCH_MISPREDICT);

// Reset counters
write_csr(mhpmcounter3, 0);
write_csr(mhpmcounter4, 0);
write_csr(mhpmcounter5, 0);

// Measure code
uint64_t cycles_start = read_csr(cycle);
uint64_t instret_start = read_csr(instret);

// ... code to measure ...

uint64_t cycles_end = read_csr(cycle);
uint64_t instret_end = read_csr(instret);
uint64_t l1_misses = read_csr(mhpmcounter3);
uint64_t l2_misses = read_csr(mhpmcounter4);
uint64_t branch_mispredicts = read_csr(mhpmcounter5);

// Report
printf("Cycles: %llu\n", cycles_end - cycles_start);
printf("Instructions: %llu\n", instret_end - instret_start);
printf("L1 D-cache misses: %llu\n", l1_misses);
printf("L2 cache misses: %llu\n", l2_misses);
printf("Branch mispredicts: %llu\n", branch_mispredicts);
```

---

## 16.4 Performance Events

**Cache Events**

Cache events track memory hierarchy performance:

*L1 Instruction Cache*:

- L1 I-cache access
- L1 I-cache miss
- L1 I-cache hit

*L1 Data Cache*:

- L1 D-cache access
- L1 D-cache miss
- L1 D-cache hit
- L1 D-cache writeback

*L2 Cache*:

- L2 cache access
- L2 cache miss
- L2 cache hit

Example: Measure cache miss rate:

```c
// Configure counters
write_csr(mhpmevent3, EVENT_L1_DCACHE_ACCESS);
write_csr(mhpmevent4, EVENT_L1_DCACHE_MISS);

// Reset and measure
write_csr(mhpmcounter3, 0);
write_csr(mhpmcounter4, 0);

// ... code ...

uint64_t accesses = read_csr(mhpmcounter3);
uint64_t misses = read_csr(mhpmcounter4);
double miss_rate = (double)misses / accesses * 100.0;

printf("L1 D-cache miss rate: %.2f%%\n", miss_rate);
```

**Branch Events**

Branch events track control flow performance:

*Branch Types*:

- Branch instructions executed
- Branch taken
- Branch not taken

*Branch Prediction*:

- Branch mispredicted
- Branch correctly predicted

Example: Measure branch prediction accuracy:

```c
write_csr(mhpmevent3, EVENT_BRANCH_EXECUTED);
write_csr(mhpmevent4, EVENT_BRANCH_MISPREDICT);

write_csr(mhpmcounter3, 0);
write_csr(mhpmcounter4, 0);

// ... code with branches ...

uint64_t branches = read_csr(mhpmcounter3);
uint64_t mispredicts = read_csr(mhpmcounter4);
double accuracy = (1.0 - (double)mispredicts / branches) * 100.0;

printf("Branch prediction accuracy: %.2f%%\n", accuracy);
```

**Pipeline Events**

Pipeline events track execution efficiency:

*Stalls*:

- Pipeline stall cycles
- Load-use stall
- Store buffer full stall

*Flushes*:

- Pipeline flush (branch mispredict, exception)
- I-cache flush
- D-cache flush

Example: Identify stall sources:

```c
write_csr(mhpmevent3, EVENT_PIPELINE_STALL);
write_csr(mhpmevent4, EVENT_LOAD_USE_STALL);

write_csr(mhpmcounter3, 0);
write_csr(mhpmcounter4, 0);

// ... code ...

uint64_t total_stalls = read_csr(mhpmcounter3);
uint64_t load_use_stalls = read_csr(mhpmcounter4);

printf("Total stall cycles: %llu\n", total_stalls);
printf("Load-use stalls: %llu (%.1f%%)\n",
       load_use_stalls,
       (double)load_use_stalls / total_stalls * 100.0);
```

**Memory Events**

Memory events track memory system activity:

*Memory Operations*:

- Load instructions
- Store instructions
- Atomic instructions

*TLB Events*:

- TLB access
- TLB miss (I-TLB, D-TLB)
- Page table walk

Example: Measure TLB performance:

```c
write_csr(mhpmevent3, EVENT_DTLB_ACCESS);
write_csr(mhpmevent4, EVENT_DTLB_MISS);

write_csr(mhpmcounter3, 0);
write_csr(mhpmcounter4, 0);

// ... code with memory accesses ...

uint64_t tlb_accesses = read_csr(mhpmcounter3);
uint64_t tlb_misses = read_csr(mhpmcounter4);
double tlb_miss_rate = (double)tlb_misses / tlb_accesses * 100.0;

printf("D-TLB miss rate: %.2f%%\n", tlb_miss_rate);
```

---

## 16.5 Profiling and Analysis

**perf Tool for RISC-V**

The Linux `perf` tool supports RISC-V performance counters:

```bash
# Count cycles and instructions
perf stat -e cycles,instructions ./my_program

# Sample on cycles (profiling)
perf record -e cycles ./my_program
perf report

# Count cache misses
perf stat -e L1-dcache-load-misses,L1-dcache-loads ./my_program

# Count branch mispredictions
perf stat -e branch-misses,branches ./my_program
```

**PMU Programming**

Kernel-level PMU programming:

```c
// Linux kernel: Configure PMU for profiling
void setup_pmu_profiling(void) {
    // Enable cycle and instret for user mode
    write_csr(mcounteren, 0x7);  // CY, TM, IR

    // Configure HPM counter for L1 D-cache misses
    write_csr(mhpmevent3, EVENT_L1_DCACHE_MISS);
    write_csr(mhpmcounter3, 0);

    // Enable counter for user mode
    uint64_t mcounteren = read_csr(mcounteren);
    mcounteren |= (1 << 3);  // HPM3
    write_csr(mcounteren, mcounteren);
}
```

**Event Sampling**

Sampling-based profiling collects periodic samples:

```c
// Pseudo-code: Sample-based profiling
void pmu_interrupt_handler(void) {
    // Read PC where interrupt occurred
    uint64_t pc = read_csr(mepc);

    // Record sample
    record_sample(pc);

    // Reset counter for next sample
    write_csr(mhpmcounter3, -SAMPLE_PERIOD);
}

// Setup sampling
void setup_sampling(void) {
    // Configure counter to overflow after SAMPLE_PERIOD events
    write_csr(mhpmevent3, EVENT_CYCLES);
    write_csr(mhpmcounter3, -SAMPLE_PERIOD);

    // Enable overflow interrupt (implementation-specific)
    enable_pmu_interrupt();
}
```

**Performance Analysis Techniques**

*Top-down analysis*:

1. Measure overall IPC
2. If IPC is low, identify bottleneck:
   - Cache misses? → Optimize data layout
   - Branch mispredicts? → Improve branch predictability
   - Pipeline stalls? → Reduce dependencies

*Hotspot analysis*:

1. Use sampling to find hot functions
2. Measure counters for hot functions
3. Optimize based on counter data

*Comparative analysis*:

1. Measure before optimization
2. Apply optimization
3. Measure after optimization
4. Compare counter values

Example workflow:

```bash
# Before optimization
perf stat -e cycles,instructions,L1-dcache-load-misses ./program
# Cycles: 1000000, Instructions: 500000, IPC: 0.5, Misses: 50000

# After optimization (improved data locality)
perf stat -e cycles,instructions,L1-dcache-load-misses ./program_opt
# Cycles: 600000, Instructions: 500000, IPC: 0.83, Misses: 10000
# Result: 40% speedup, 80% reduction in cache misses
```

---

## 16.6 Comparison with ARM PMU

**RISC-V Counters vs ARM PMU**

ARM provides a Performance Monitoring Unit (PMU) with similar capabilities. Comparison:

| Feature | RISC-V PMU | ARM PMU |
|---------|------------|---------|
| **Basic Counters** | cycle, instret, time | PMCCNTR (cycle), no instret |
| **HPM Counters** | mhpmcounter3-31 (up to 29) | PMEVCNTRn (typically 6-8) |
| **Event Selection** | mhpmevent3-31 | PMEVTYPER (event type) |
| **Counter Width** | 64-bit | 32-bit or 64-bit (ARMv8) |
| **Overflow** | Implementation-specific | Overflow interrupt (PMOVSCLR) |
| **Access Control** | mcounteren, scounteren | PMUSERENR (user enable) |
| **Counter Inhibit** | mcountinhibit | PMCNTENSET/CLR (enable/disable) |
| **Privilege Levels** | M/S/U modes | EL0/EL1/EL2/EL3 |

**Event Mapping**

Common events mapped between architectures:

| Event | RISC-V | ARM |
|-------|--------|-----|
| **Cycles** | cycle CSR | PMCCNTR |
| **Instructions** | instret CSR | No direct equivalent |
| **L1 I-cache miss** | Implementation-specific | 0x01 |
| **L1 D-cache miss** | Implementation-specific | 0x03 |
| **L2 cache miss** | Implementation-specific | 0x17 |
| **Branch mispredict** | Implementation-specific | 0x10 |
| **Branch executed** | Implementation-specific | 0x0C |
| **TLB miss** | Implementation-specific | 0x05 (I-TLB), 0x06 (D-TLB) |

ARM event codes are standardized (ARM Architecture Reference Manual), while RISC-V event codes are implementation-specific.

**Profiling Tool Comparison**

Both architectures support standard profiling tools:

*RISC-V*:

```bash
# perf on RISC-V Linux
perf stat -e cycles,instructions,cache-misses ./program
perf record -e cycles -g ./program
perf report
```

*ARM*:

```bash
# perf on ARM Linux
perf stat -e cycles,instructions,cache-misses ./program
perf record -e cycles -g ./program
perf report
```

The `perf` tool abstracts architecture differences, providing a consistent interface.

**Practical Differences**

*RISC-V advantages*:

- 64-bit counters (no overflow on long runs)
- Separate instret counter (ARM lacks this)
- Up to 29 HPM counters (ARM typically 6-8)
- Simpler privilege model

*ARM advantages*:

- Standardized event codes (portable across implementations)
- Mature PMU infrastructure
- Overflow interrupts (standard)
- Extensive tool support

*Example: Measuring IPC*

RISC-V:

```c
uint64_t cycles = read_csr(cycle);
uint64_t instret = read_csr(instret);
double ipc = (double)instret / cycles;
```

ARM (requires software counting):

```c
uint64_t cycles = read_pmccntr();
// No instret equivalent—must use PMU event counter
uint64_t instret = read_pmevcntr(0);  // Configured for instruction count
double ipc = (double)instret / cycles;
```

RISC-V's dedicated instret counter simplifies IPC measurement.

**Implementation Examples**

*RISC-V*:

- SiFive U74: 2 HPM counters (L1 cache events)
- SiFive P550: 6 HPM counters (cache, branch, TLB events)
- Alibaba XuanTie C910: 4 HPM counters

*ARM*:

- Cortex-A53: 6 PMU counters
- Cortex-A72: 6 PMU counters
- Cortex-A76: 6 PMU counters
- Neoverse N1: 6 PMU counters

RISC-V implementations vary widely in HPM counter count. ARM implementations are more consistent (typically 6 counters).

---

## Summary

Performance counters and the Performance Monitoring Unit enable quantitative performance analysis. This chapter explored RISC-V's counter architecture and how it compares to ARM's mature PMU infrastructure.

**Performance counter architecture** provides hardware-based measurement with minimal overhead. Counter CSRs exist at multiple privilege levels—machine-mode counters (mcycle, minstret, mhpmcounter) and supervisor/user-mode readable shadows (cycle, instret, hpmcounter). Access control through mcounteren and scounteren enables selective counter exposure to lower privilege levels. Counter inhibit via mcountinhibit allows stopping counters to measure specific code regions or reduce power consumption.

**Basic performance counters** provide fundamental metrics. The cycle counter tracks clock cycles executed by the hart. The instret counter tracks instructions retired (completed). The time counter provides wall-clock time at a constant frequency. Together, cycle and instret enable IPC calculation, a key performance metric. The difference between cycle (stops during sleep) and time (continues during sleep) enables measuring sleep overhead and real-time intervals.

**Hardware performance counters** track implementation-specific events through mhpmcounter3-31 (up to 29 counters). Event selection via mhpmevent CSRs configures what each counter tracks. Counters are 64-bit, minimizing overflow concerns. Multiple counters can measure different events simultaneously, enabling comprehensive performance characterization. Counter overflow handling is implementation-specific, with some implementations supporting overflow interrupts.

**Performance events** cover the full spectrum of microarchitectural activity. Cache events track L1 instruction cache, L1 data cache, and L2 cache hits and misses, revealing memory hierarchy performance. Branch events track branch execution and prediction accuracy, identifying control flow bottlenecks. Pipeline events track stalls and flushes, showing execution efficiency. Memory events track loads, stores, and TLB performance, revealing memory system behavior.

**Profiling and analysis** leverage performance counters for optimization. The Linux perf tool provides a standard interface to RISC-V counters for counting events and sampling-based profiling. PMU programming in the kernel configures counters and enables user-mode access. Event sampling collects periodic samples to identify hot code regions. Performance analysis techniques include top-down analysis (identify bottleneck category), hotspot analysis (find hot functions), and comparative analysis (measure optimization impact).

**Comparison with ARM** shows both similarities and differences. ARM's PMU provides similar functionality with a cycle counter and multiple event counters. ARM standardizes event codes across implementations, while RISC-V leaves them implementation-specific. RISC-V provides 64-bit counters and a dedicated instret counter, simplifying IPC measurement. ARM provides standardized overflow interrupts. Both architectures support the perf tool, providing a consistent user experience. RISC-V allows up to 29 HPM counters, while ARM implementations typically provide 6-8 counters.

Together, RISC-V's performance counters enable effective performance measurement, profiling, and optimization across the full range from embedded systems to high-performance processors.
