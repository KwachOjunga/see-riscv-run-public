# Chapter 16. Performance Counters & PMU

**Part IX — Performance, Debug & Tools**

---

Performance optimization 需要測量。程式執行緩慢，我們需要知道原因。Cache miss 主導執行時間，或 branch misprediction 導致 pipeline stall，或 memory bandwidth 限制 throughput。Performance counter 將模糊的緩慢轉化為可量化的 bottleneck。

RISC-V 通過一組 hardware performance counter 提供 Performance Monitoring Unit (PMU)。這些 counter 追蹤 event，如執行的 cycle、retired instruction、cache hit 和 miss、branch prediction、TLB access。基本 counter（cycle、instret、time）是強制性的，提供基本 metric。Hardware performance counter（mhpmcounter3-31）是可選的，追蹤 implementation-specific event。這些 counter 共同實現 profiling、bottleneck identification 和 performance analysis。

本章探討 RISC-V performance counter 和 PMU。我們將檢視 counter architecture、基本 counter、hardware performance counter、performance event、profiling technique，以及 RISC-V 與 ARM PMU 的比較。

---

## 16.1 Performance Counter Architecture

**Performance Monitoring Overview**

Performance monitoring 回答以下問題：

- 這個 function 花費了多少 cycle？
- IPC（instructions per cycle）是多少？
- 發生了多少 cache miss？
- 有多少 branch 被 mispredict？
- Performance bottleneck 在哪裡？

RISC-V performance counter 提供基於 hardware 的測量，overhead 最小。Counter 在特定 event 上自動遞增，允許精確測量而無需 software instrumentation。

**Counter CSRs**

RISC-V 在三個 privilege level 定義 performance counter CSR：

*Machine-mode counter*（僅 M-mode）：

- `mcycle`：Machine cycle counter
- `minstret`：Machine instructions-retired counter
- `mhpmcounter3-31`：Machine hardware performance counter（29 個 counter）

*Supervisor/User-mode counter*（可從 S/U-mode 讀取）：

- `cycle`：Cycle counter（mcycle 的 shadow）
- `instret`：Instructions-retired counter（minstret 的 shadow）
- `hpmcounter3-31`：Hardware performance counter（mhpmcounter 的 shadow）

*Time counter*：

- `time`：Real-time counter（wall-clock time）

對於 RV32，每個 counter 都有 high-word CSR（例如 `mcycleh`、`cycleh`）用於 64-bit 值。

**Counter Privilege Level**

Counter 根據 privilege 可訪問：

```
M-mode: 可以讀寫所有 counter（mcycle、minstret、mhpmcounter）
S-mode: 可以讀 cycle、instret、hpmcounter（如果啟用）
U-mode: 可以讀 cycle、instret、hpmcounter（如果啟用）
```

通過 `mcounteren` 和 `scounteren` 進行 access control：

```c
// 為 S-mode 和 U-mode 啟用 cycle 和 instret
uint64_t mcounteren = (1 << 0) | (1 << 2);  // CY, IR
write_csr(mcounteren, mcounteren);

// 為 U-mode 啟用 cycle 和 instret（從 S-mode）
uint64_t scounteren = (1 << 0) | (1 << 2);
write_csr(scounteren, scounteren);
```

**Counter Inhibit**

Counter 可以通過 `mcountinhibit` inhibit（停止）：

```c
// 停止 cycle 和 instret counter
uint64_t mcountinhibit = (1 << 0) | (1 << 2);  // CY, IR
write_csr(mcountinhibit, mcountinhibit);

// 恢復 counter
write_csr(mcountinhibit, 0);
```

這對以下情況有用：

- 測量特定 code region
- 減少 power consumption
- 防止 counter overflow

---

## 16.2 Basic Performance Counters

**mcycle / cycle (Cycle Counter)**

Cycle counter 追蹤 hart 執行的 clock cycle 數量：

```c
// 讀取 cycle counter
uint64_t start = read_csr(cycle);
// ... 要測量的 code ...
uint64_t end = read_csr(cycle);
uint64_t cycles = end - start;

printf("Cycles: %llu\n", cycles);
```

對於 RV32，使用 cycleh 獲取高 32 bit：

```c
// RV32: 讀取 64-bit cycle counter
uint64_t read_cycle_rv32(void) {
    uint32_t hi, lo, hi2;
    do {
        hi = read_csr(cycleh);
        lo = read_csr(cycle);
        hi2 = read_csr(cycleh);
    } while (hi != hi2);  // 如果 high word 改變則重試
    
    return ((uint64_t)hi << 32) | lo;
}
```

**minstret / instret (Instructions Retired Counter)**

Instructions-retired counter 追蹤完成的 instruction 數量：

```c
// 讀取 instret counter
uint64_t start = read_csr(instret);
// ... 要測量的 code ...
uint64_t end = read_csr(instret);
uint64_t instructions = end - start;

printf("Instructions: %llu\n", instructions);
```

**IPC Calculation**

結合 cycle 和 instret 可以得到 IPC（instructions per cycle）：

```c
// 測量 IPC
uint64_t cycles_start = read_csr(cycle);
uint64_t instret_start = read_csr(instret);

// ... 要測量的 code ...

uint64_t cycles_end = read_csr(cycle);
uint64_t instret_end = read_csr(instret);

uint64_t cycles = cycles_end - cycles_start;
uint64_t instructions = instret_end - instret_start;

double ipc = (double)instructions / cycles;
printf("IPC: %.2f\n", ipc);
```

IPC 解釋：

- IPC 接近 1：良好利用（in-order core）
- IPC > 1：Superscalar execution（out-of-order core）
- IPC < 1：Pipeline stall（cache miss、branch mispredict 等）

**time (Real-Time Counter)**

Time counter 提供 wall-clock time：

```c
// 讀取 time counter
uint64_t start_time = read_csr(time);
// ... 要測量的 code ...
uint64_t end_time = read_csr(time);
uint64_t elapsed = end_time - start_time;

// 轉換為 microsecond（假設 1 MHz time counter）
printf("Elapsed time: %llu us\n", elapsed);
```

Time counter frequency 是 platform-specific（通常為 1 MHz 或 10 MHz）。它對以下情況有用：

- Wall-clock timing
- Timeout implementation
- Real-time scheduling

**Difference: cycle vs time**

- `cycle`：計數 CPU cycle（在 sleep 期間停止，隨 frequency scaling 變化）
- `time`：計數 real time（在 sleep 期間繼續，constant frequency）

```c
// 範例：測量 sleep overhead
uint64_t cycles_before = read_csr(cycle);
uint64_t time_before = read_csr(time);

wfi();  // Sleep 直到 interrupt

uint64_t cycles_after = read_csr(cycle);
uint64_t time_after = read_csr(time);

printf("Cycles during sleep: %llu\n", cycles_after - cycles_before);  // ~0
printf("Time during sleep: %llu\n", time_after - time_before);        // > 0
```

---

## 16.3 Hardware Performance Counters

**mhpmcounter3-31 (Hardware Performance Counters)**

RISC-V 提供最多 29 個 hardware performance counter（HPM counter）用於追蹤 implementation-specific event。這些 counter 是可選的 — implementation 可以提供 0 到 29 個 counter。

Counter CSR：

- `mhpmcounter3-31`：M-mode counter（29 個 counter）
- `hpmcounter3-31`：S/U-mode 可讀 counter（mhpmcounter 的 shadow）
- `mhpmevent3-31`：Event selection register

**Event Selection (mhpmevent CSRs)**

每個 HPM counter 都有關聯的 event selector：

```c
// 配置 mhpmcounter3 計數 L1 I-cache miss
write_csr(mhpmevent3, EVENT_L1_ICACHE_MISS);

// Reset counter
write_csr(mhpmcounter3, 0);

// ... 要測量的 code ...

// 讀取 counter
uint64_t icache_misses = read_csr(mhpmcounter3);
printf("L1 I-cache misses: %llu\n", icache_misses);
```

Event code 是 implementation-specific。常見 event 包括：

- Cache event（L1/L2 hit、miss）
- Branch event（taken、not-taken、mispredicted）
- Pipeline event（stall、flush）
- Memory event（load、store、TLB miss）

**Counter Overflow Handling**

Counter 是 64-bit，很少 overflow。如果 overflow 是問題：

```c
// 檢查 overflow（counter wrapped around）
uint64_t start = read_csr(mhpmcounter3);
// ... code ...
uint64_t end = read_csr(mhpmcounter3);

if (end < start) {
    // Overflow 發生
    uint64_t count = (UINT64_MAX - start) + end + 1;
} else {
    uint64_t count = end - start;
}
```

某些 implementation 支援 overflow interrupt（implementation-specific）。

**Example: Multi-Counter Measurement**

同時測量多個 event：

```c
// 配置 counter
write_csr(mhpmevent3, EVENT_L1_DCACHE_MISS);
write_csr(mhpmevent4, EVENT_L2_CACHE_MISS);
write_csr(mhpmevent5, EVENT_BRANCH_MISPREDICT);

// Reset counter
write_csr(mhpmcounter3, 0);
write_csr(mhpmcounter4, 0);
write_csr(mhpmcounter5, 0);

// 測量 code
uint64_t cycles_start = read_csr(cycle);
uint64_t instret_start = read_csr(instret);

// ... 要測量的 code ...

uint64_t cycles_end = read_csr(cycle);
uint64_t instret_end = read_csr(instret);
uint64_t l1_misses = read_csr(mhpmcounter3);
uint64_t l2_misses = read_csr(mhpmcounter4);
uint64_t branch_mispredicts = read_csr(mhpmcounter5);

// 報告
printf("Cycles: %llu\n", cycles_end - cycles_start);
printf("Instructions: %llu\n", instret_end - instret_start);
printf("L1 D-cache misses: %llu\n", l1_misses);
printf("L2 cache misses: %llu\n", l2_misses);
printf("Branch mispredicts: %llu\n", branch_mispredicts);
```

---

## 16.4 Performance Events

**Cache Events**

Cache event 追蹤 memory hierarchy performance：

*L1 Instruction Cache*：

- L1 I-cache access
- L1 I-cache miss
- L1 I-cache hit

*L1 Data Cache*：

- L1 D-cache access
- L1 D-cache miss
- L1 D-cache hit
- L1 D-cache writeback

*L2 Cache*：

- L2 cache access
- L2 cache miss
- L2 cache hit

範例：測量 cache miss rate：

```c
// 配置 counter
write_csr(mhpmevent3, EVENT_L1_DCACHE_ACCESS);
write_csr(mhpmevent4, EVENT_L1_DCACHE_MISS);

// Reset 並測量
write_csr(mhpmcounter3, 0);
write_csr(mhpmcounter4, 0);

// ... code ...

uint64_t accesses = read_csr(mhpmcounter3);
uint64_t misses = read_csr(mhpmcounter4);
double miss_rate = (double)misses / accesses * 100.0;

printf("L1 D-cache miss rate: %.2f%%\n", miss_rate);
```

**Branch Events**

Branch event 追蹤 control flow performance：

*Branch Type*：

- Branch instruction executed
- Branch taken
- Branch not taken

*Branch Prediction*：

- Branch mispredicted
- Branch correctly predicted

範例：測量 branch prediction accuracy：

```c
write_csr(mhpmevent3, EVENT_BRANCH_EXECUTED);
write_csr(mhpmevent4, EVENT_BRANCH_MISPREDICT);

write_csr(mhpmcounter3, 0);
write_csr(mhpmcounter4, 0);

// ... 有 branch 的 code ...

uint64_t branches = read_csr(mhpmcounter3);
uint64_t mispredicts = read_csr(mhpmcounter4);
double accuracy = (1.0 - (double)mispredicts / branches) * 100.0;

printf("Branch prediction accuracy: %.2f%%\n", accuracy);
```

**Pipeline Events**

Pipeline event 追蹤 execution efficiency：

*Stall*：

- Pipeline stall cycle
- Load-use stall
- Store buffer full stall

*Flush*：

- Pipeline flush（branch mispredict、exception）
- I-cache flush
- D-cache flush

範例：識別 stall source：

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

Memory event 追蹤 memory system activity：

*Memory Operation*：

- Load instruction
- Store instruction
- Atomic instruction

*TLB Event*：

- TLB access
- TLB miss（I-TLB、D-TLB）
- Page table walk

範例：測量 TLB performance：

```c
write_csr(mhpmevent3, EVENT_DTLB_ACCESS);
write_csr(mhpmevent4, EVENT_DTLB_MISS);

write_csr(mhpmcounter3, 0);
write_csr(mhpmcounter4, 0);

// ... 有 memory access 的 code ...

uint64_t tlb_accesses = read_csr(mhpmcounter3);
uint64_t tlb_misses = read_csr(mhpmcounter4);
double tlb_miss_rate = (double)tlb_misses / tlb_accesses * 100.0;

printf("D-TLB miss rate: %.2f%%\n", tlb_miss_rate);
```

---

## 16.5 Profiling and Analysis

**perf Tool for RISC-V**

Linux `perf` tool 支援 RISC-V performance counter：

```bash
# 計數 cycle 和 instruction
perf stat -e cycles,instructions ./my_program

# Sample on cycle（profiling）
perf record -e cycles ./my_program
perf report

# 計數 cache miss
perf stat -e L1-dcache-load-misses,L1-dcache-loads ./my_program

# 計數 branch misprediction
perf stat -e branch-misses,branches ./my_program
```

**PMU Programming**

Kernel-level PMU programming：

```c
// Linux kernel: 配置 PMU 用於 profiling
void setup_pmu_profiling(void) {
    // 為 user mode 啟用 cycle 和 instret
    write_csr(mcounteren, 0x7);  // CY, TM, IR

    // 配置 HPM counter 用於 L1 D-cache miss
    write_csr(mhpmevent3, EVENT_L1_DCACHE_MISS);
    write_csr(mhpmcounter3, 0);

    // 為 user mode 啟用 counter
    uint64_t mcounteren = read_csr(mcounteren);
    mcounteren |= (1 << 3);  // HPM3
    write_csr(mcounteren, mcounteren);
}
```

**Event Sampling**

Sampling-based profiling 收集 periodic sample：

```c
// Pseudo-code: Sample-based profiling
void pmu_interrupt_handler(void) {
    // 讀取 interrupt 發生時的 PC
    uint64_t pc = read_csr(mepc);

    // 記錄 sample
    record_sample(pc);

    // Reset counter 用於下一個 sample
    write_csr(mhpmcounter3, -SAMPLE_PERIOD);
}

// Setup sampling
void setup_sampling(void) {
    // 配置 counter 在 SAMPLE_PERIOD event 後 overflow
    write_csr(mhpmevent3, EVENT_CYCLES);
    write_csr(mhpmcounter3, -SAMPLE_PERIOD);

    // 啟用 overflow interrupt（implementation-specific）
    enable_pmu_interrupt();
}
```

**Performance Analysis Techniques**

*Top-down analysis*：

1. 測量整體 IPC
2. 如果 IPC 低，識別 bottleneck：
   - Cache miss？→ 優化 data layout
   - Branch mispredict？→ 改善 branch predictability
   - Pipeline stall？→ 減少 dependency

*Hotspot analysis*：

1. 使用 sampling 找到 hot function
2. 測量 hot function 的 counter
3. 根據 counter data 優化

*Comparative analysis*：

1. 優化前測量
2. 應用優化
3. 優化後測量
4. 比較 counter value

範例 workflow：

```bash
# 優化前
perf stat -e cycles,instructions,L1-dcache-load-misses ./program
# Cycles: 1000000, Instructions: 500000, IPC: 0.5, Misses: 50000

# 優化後（改善 data locality）
perf stat -e cycles,instructions,L1-dcache-load-misses ./program_opt
# Cycles: 600000, Instructions: 500000, IPC: 0.83, Misses: 10000
# 結果：40% speedup，80% cache miss 減少
```

---

## 16.6 Comparison with ARM PMU

**RISC-V Counters vs ARM PMU**

ARM 提供 Performance Monitoring Unit (PMU)，具有類似 capability。比較：

| Feature | RISC-V PMU | ARM PMU |
|---------|------------|---------|
| **Basic Counter** | cycle, instret, time | PMCCNTR (cycle), no instret |
| **HPM Counter** | mhpmcounter3-31 (up to 29) | PMEVCNTRn (typically 6-8) |
| **Event Selection** | mhpmevent3-31 | PMEVTYPER (event type) |
| **Counter Width** | 64-bit | 32-bit or 64-bit (ARMv8) |
| **Overflow** | Implementation-specific | Overflow interrupt (PMOVSCLR) |
| **Access Control** | mcounteren, scounteren | PMUSERENR (user enable) |
| **Counter Inhibit** | mcountinhibit | PMCNTENSET/CLR (enable/disable) |
| **Privilege Level** | M/S/U mode | EL0/EL1/EL2/EL3 |

**Event Mapping**

常見 event 在 architecture 之間的映射：

| Event | RISC-V | ARM |
|-------|--------|-----|
| **Cycle** | cycle CSR | PMCCNTR |
| **Instruction** | instret CSR | No direct equivalent |
| **L1 I-cache miss** | Implementation-specific | 0x01 |
| **L1 D-cache miss** | Implementation-specific | 0x03 |
| **L2 cache miss** | Implementation-specific | 0x17 |
| **Branch mispredict** | Implementation-specific | 0x10 |
| **Branch executed** | Implementation-specific | 0x0C |
| **TLB miss** | Implementation-specific | 0x05 (I-TLB), 0x06 (D-TLB) |

ARM event code 是標準化的（ARM Architecture Reference Manual），而 RISC-V event code 是 implementation-specific。

**Profiling Tool Comparison**

兩種 architecture 都支援 standard profiling tool：

*RISC-V*：

```bash
# perf on RISC-V Linux
perf stat -e cycles,instructions,cache-misses ./program
perf record -e cycles -g ./program
perf report
```

*ARM*：

```bash
# perf on ARM Linux
perf stat -e cycles,instructions,cache-misses ./program
perf record -e cycles -g ./program
perf report
```

`perf` tool 抽象化 architecture 差異，提供一致的 interface。

**Practical Differences**

*RISC-V advantage*：

- 64-bit counter（長時間運行不 overflow）
- 獨立的 instret counter（ARM 缺少這個）
- 最多 29 個 HPM counter（ARM 通常 6-8 個）
- 更簡單的 privilege model

*ARM advantage*：

- 標準化的 event code（跨 implementation 可移植）
- 成熟的 PMU infrastructure
- Overflow interrupt（標準）
- 廣泛的 tool support

*Example: Measuring IPC*

RISC-V：

```c
uint64_t cycles = read_csr(cycle);
uint64_t instret = read_csr(instret);
double ipc = (double)instret / cycles;
```

ARM（需要 software counting）：

```c
uint64_t cycles = read_pmccntr();
// 沒有 instret equivalent — 必須使用 PMU event counter
uint64_t instret = read_pmevcntr(0);  // 配置為 instruction count
double ipc = (double)instret / cycles;
```

RISC-V 的專用 instret counter 簡化了 IPC 測量。

**Implementation Examples**

*RISC-V*：

- SiFive U74：2 個 HPM counter（L1 cache event）
- SiFive P550：6 個 HPM counter（cache、branch、TLB event）
- Alibaba XuanTie C910：4 個 HPM counter

*ARM*：

- Cortex-A53：6 個 PMU counter
- Cortex-A72：6 個 PMU counter
- Cortex-A76：6 個 PMU counter
- Neoverse N1：6 個 PMU counter

RISC-V implementation 在 HPM counter 數量上差異很大。ARM implementation 更一致（通常 6 個 counter）。

---

## Summary

Performance counter 和 Performance Monitoring Unit 實現定量 performance analysis。本章探討了 RISC-V 的 counter architecture 以及它與 ARM 成熟的 PMU infrastructure 的比較。

**Performance counter architecture** 提供基於 hardware 的測量，overhead 最小。Counter CSR 存在於多個 privilege level — machine-mode counter（mcycle、minstret、mhpmcounter）和 supervisor/user-mode 可讀 shadow（cycle、instret、hpmcounter）。通過 mcounteren 和 scounteren 進行 access control，實現選擇性地將 counter 暴露給較低 privilege level。通過 mcountinhibit 進行 counter inhibit，允許停止 counter 以測量特定 code region 或減少 power consumption。

**Basic performance counter** 提供基本 metric。Cycle counter 追蹤 hart 執行的 clock cycle。Instret counter 追蹤 retired（完成）的 instruction。Time counter 以 constant frequency 提供 wall-clock time。結合 cycle 和 instret 可以計算 IPC，這是關鍵的 performance metric。Cycle（在 sleep 期間停止）和 time（在 sleep 期間繼續）之間的差異使得能夠測量 sleep overhead 和 real-time interval。

**Hardware performance counter** 通過 mhpmcounter3-31（最多 29 個 counter）追蹤 implementation-specific event。通過 mhpmevent CSR 進行 event selection，配置每個 counter 追蹤什麼。Counter 是 64-bit，最小化 overflow 問題。多個 counter 可以同時測量不同 event，實現全面的 performance characterization。Counter overflow handling 是 implementation-specific，某些 implementation 支援 overflow interrupt。

**Performance event** 涵蓋 microarchitectural activity 的全部範圍。Cache event 追蹤 L1 instruction cache、L1 data cache 和 L2 cache 的 hit 和 miss，揭示 memory hierarchy performance。Branch event 追蹤 branch execution 和 prediction accuracy，識別 control flow bottleneck。Pipeline event 追蹤 stall 和 flush，顯示 execution efficiency。Memory event 追蹤 load、store 和 TLB performance，揭示 memory system behavior。

**Profiling and analysis** 利用 performance counter 進行優化。Linux perf tool 為 RISC-V counter 提供 standard interface，用於計數 event 和 sampling-based profiling。Kernel 中的 PMU programming 配置 counter 並啟用 user-mode access。Event sampling 收集 periodic sample 以識別 hot code region。Performance analysis technique 包括 top-down analysis（識別 bottleneck category）、hotspot analysis（找到 hot function）和 comparative analysis（測量優化影響）。

**Comparison with ARM** 顯示了相似性和差異性。ARM 的 PMU 提供類似 functionality，具有 cycle counter 和多個 event counter。ARM 跨 implementation 標準化 event code，而 RISC-V 將它們留給 implementation-specific。RISC-V 提供 64-bit counter 和專用 instret counter，簡化 IPC 測量。ARM 提供標準化的 overflow interrupt。兩種 architecture 都支援 perf tool，提供一致的 user experience。RISC-V 允許最多 29 個 HPM counter，而 ARM implementation 通常提供 6-8 個 counter。

總之，RISC-V 的 performance counter 在從 embedded system 到 high-performance processor 的全部範圍內實現有效的 performance measurement、profiling 和 optimization。

---
