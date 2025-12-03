# Appendix F. Memory Model Quick Reference

**RISC-V Weak Memory Ordering (RVWMO) Quick Reference**

---

本附錄提供 RISC-V memory model（RVWMO）的快速參考。理解 memory ordering 對於在 RISC-V 上撰寫正確的 concurrent code 至關重要。

---

## F.1 Memory Ordering Basics

### What Can Be Reordered?

RISC-V Weak Memory Ordering（RVWMO）允許廣泛的 reordering：

| Reordering | Allowed? | Exception |
|------------|----------|-----------|
| **Load → Load** | ✓ Yes | Same address, or FENCE |
| **Load → Store** | ✓ Yes | Same address, or FENCE |
| **Store → Store** | ✓ Yes | Same address, or FENCE |
| **Store → Load** | ✓ Yes | Same address, or FENCE |

**Key Point**: 幾乎所有操作都可以被 reorder，除非：

1. Operation 存取相同 address（overlapping）
2. Operation 被 FENCE instruction 分隔
3. Operation 有 data/control dependency
4. Operation 使用 acquire/release atomic

---

### Preserved Program Order (PPO)

**Preserved Program Order** 是 program order 的子集，必須被遵守：

1. **Overlapping address**: `SW` to X, then `LW` from X → 總是按順序
2. **Explicit fence**: 被 `FENCE` 分隔的 operation → 總是按順序
3. **Acquire/Release**: 帶有 `.aq` 或 `.rl` 的 atomic operation → 強制 ordering
4. **Dependency**: Data dependency（例如 `LW` 然後使用結果）→ 總是按順序
5. **Control dependency**: Branch 然後 dependent operation → 某些 ordering 被保留

---

## F.2 FENCE Instruction Reference

### FENCE Syntax

```assembly
fence pred, succ
```

- **pred**（predecessor）: Fence 之前的 operation（r、w、或 rw）
- **succ**（successor）: Fence 之後的 operation（r、w、或 rw）

---

### Common FENCE Variants

| FENCE | Meaning | Use Case |
|-------|---------|----------|
| `fence rw, rw` | Full fence | 最強的 barrier，order 所有操作 |
| `fence w, w` | Store-store fence | 確保 store 按順序可見 |
| `fence r, r` | Load-load fence | 確保 load 按順序發生 |
| `fence r, rw` | Acquire fence | 在 acquire lock 之後 |
| `fence rw, w` | Release fence | 在 release lock 之前 |
| `fence.i` | Instruction fence | 在 code modification 之後（JIT、self-modifying code）|
| `fence.tso` | TSO fence | x86-compatible ordering |

---

### FENCE Examples

**Full Fence**（最強）：

```assembly
sw a0, 0(s0)      # Store 1
fence rw, rw      # Full fence
lw t0, 0(s1)      # Load 1
```

所有 fence 之前的 operation 在 fence 之後的任何 operation 之前完成。

---

**Store-Store Fence**（publish pattern）：

```assembly
sw a0, 0(s0)      # Write data
fence w, w        # Ensure data written first
sw a1, 0(s1)      # Write flag
```

確保 store 按順序可見。

---

**Load-Load Fence**（consume pattern）：

```assembly
lw t0, 0(s1)      # Read flag
fence r, r        # Ensure flag read first
lw t1, 0(s0)      # Read data
```

確保 load 按順序發生。

---

**Acquire Fence**（在 lock acquisition 之後）：

```assembly
lr.w.aq t0, (a0)  # Acquire lock (with .aq)
# OR
lw t0, 0(a0)      # Read lock
fence r, rw       # Acquire fence
# ... critical section ...
```

防止 critical section 中的 operation 移到 lock acquisition 之前。

---

**Release Fence**（在 lock release 之前）：

```assembly
# ... critical section ...
fence rw, w       # Release fence
sw zero, 0(a0)    # Release lock
```

防止 critical section 中的 operation 移到 lock release 之後。

---

## F.3 Atomic Instructions

### Load-Reserved / Store-Conditional (LR/SC)

**Syntax**：

```assembly
lr.w rd, (rs1)           # Load-reserved word
lr.d rd, (rs1)           # Load-reserved doubleword
sc.w rd, rs2, (rs1)      # Store-conditional word
sc.d rd, rs2, (rs1)      # Store-conditional doubleword
```

**Ordering Annotations**：

- `.aq`（acquire）: 沒有後續 operation 可以移到此之前
- `.rl`（release）: 沒有先前 operation 可以移到此之後
- `.aqrl`（both）: Full ordering

---

**Example: Atomic Increment**

```assembly
retry:
    lr.w t0, (a0)         # Load current value
    addi t0, t0, 1        # Increment
    sc.w t1, t0, (a0)     # Try to store
    bnez t1, retry        # Retry if failed (t1 != 0)
```

---

**Example: Spinlock Acquire**

```assembly
acquire_lock:
    lr.w.aq t0, (a0)      # Load-reserved with acquire
    bnez t0, acquire_lock # If locked, retry
    li t1, 1
    sc.w.aq t2, t1, (a0)  # Try to acquire
    bnez t2, acquire_lock # Retry if failed
```

---

### Atomic Memory Operations (AMO)

**Syntax**：

```assembly
amoswap.w rd, rs2, (rs1)   # Atomic swap
amoadd.w rd, rs2, (rs1)    # Atomic add
amoand.w rd, rs2, (rs1)    # Atomic AND
amoor.w rd, rs2, (rs1)     # Atomic OR
amoxor.w rd, rs2, (rs1)    # Atomic XOR
amomax.w rd, rs2, (rs1)    # Atomic max (signed)
amomaxu.w rd, rs2, (rs1)   # Atomic max (unsigned)
amomin.w rd, rs2, (rs1)    # Atomic min (signed)
amominu.w rd, rs2, (rs1)   # Atomic min (unsigned)
```

**Ordering Annotations**: 與 LR/SC 相同（`.aq`、`.rl`、`.aqrl`）

---

**Example: Spinlock with AMOSWAP**

```assembly
acquire_lock:
    li t0, 1
    amoswap.w.aq t1, t0, (a0)  # Swap 1 into lock, get old value
    bnez t1, acquire_lock       # If old value != 0, retry

release_lock:
    amoswap.w.rl zero, zero, (a0)  # Swap 0 into lock (release)
```

---

## F.4 Common Synchronization Patterns

### Pattern 1: Message Passing

**Problem**: Producer 寫入 data，然後設定 flag。Consumer 等待 flag，然後讀取 data。

**Solution**：

```assembly
# Producer (Hart 0)
    sw a0, 0(s0)      # Write data
    fence w, w        # Ensure data written before flag
    sw a1, 0(s1)      # Write flag = 1

# Consumer (Hart 1)
loop:
    lw t0, 0(s1)      # Read flag
    beqz t0, loop     # Wait for flag
    fence r, r        # Ensure flag read before data
    lw t1, 0(s0)      # Read data
```

**Why fences are needed**：

- 沒有 `fence w, w`: Flag 可能在 data 之前可見
- 沒有 `fence r, r`: Data 可能在 flag 被檢查之前被讀取

---

### Pattern 2: Spinlock (LR/SC)

**Acquire**：

```assembly
acquire_lock:
    lr.w.aq t0, (a0)      # Load-reserved with acquire
    bnez t0, acquire_lock # If locked, retry
    li t1, 1
    sc.w.aq t2, t1, (a0)  # Try to set lock
    bnez t2, acquire_lock # Retry if SC failed
    # Lock acquired, critical section follows
```

**Release**：

```assembly
    # Critical section
    amoswap.w.rl zero, zero, (a0)  # Release lock
    # OR
    fence rw, w
    sw zero, 0(a0)
```

---

### Pattern 3: Spinlock (AMOSWAP)

**Acquire**：

```assembly
acquire_lock:
    li t0, 1
    amoswap.w.aq t1, t0, (a0)  # Atomic swap
    bnez t1, acquire_lock       # If old value != 0, retry
    # Lock acquired
```

**Release**：

```assembly
    amoswap.w.rl zero, zero, (a0)  # Release lock
```

---

### Pattern 4: Dekker's Algorithm (Mutual Exclusion)

**Hart 0**：

```assembly
    li t0, 1
    sw t0, flag0       # flag0 = 1
    fence w, rw        # Ensure flag0 visible before reading flag1
    lw t1, flag1       # Read flag1
    bnez t1, wait      # If flag1 set, wait
    # Critical section
    fence rw, w        # Ensure critical section done
    sw zero, flag0     # flag0 = 0
```

**Hart 1**: （對稱，交換 flag0 和 flag1）

---

### Pattern 5: Producer-Consumer Queue

**Producer**：

```assembly
    # Write data to queue[tail]
    sw a0, 0(s0)

    # Increment tail
    fence w, w         # Ensure data written before tail update
    addi s1, s1, 1
    sw s1, tail_ptr
```

**Consumer**：

```assembly
    # Read tail
    lw t0, tail_ptr
    lw t1, head_ptr
    beq t0, t1, empty  # If tail == head, queue empty

    # Read data from queue[head]
    fence r, r         # Ensure tail read before data
    lw a0, 0(s2)

    # Increment head
    addi s2, s2, 1
    sw s2, head_ptr
```

---

### Pattern 6: Barrier (N threads)

**Barrier Wait**：

```assembly
barrier_wait:
    # Increment counter atomically
    li t0, 1
    amoadd.w.aq t1, t0, (a0)  # counter++, get old value
    addi t1, t1, 1             # t1 = new counter value

    # Check if all threads arrived
    li t2, N                   # N = number of threads
    bne t1, t2, spin           # If not all arrived, spin

    # Reset counter for next barrier
    amoswap.w.rl zero, zero, (a0)
    ret

spin:
    lw t3, 0(a0)
    bne t3, t2, spin
    ret
```

---

## F.5 Memory Model Comparison

### RVWMO vs Other Models

| Model | Strength | Reordering Allowed | Fence Overhead |
|-------|----------|-------------------|----------------|
| **Sequential Consistency** | Strongest | None | N/A (no reordering) |
| **x86 TSO** | Strong | Store→Load only | Low (implicit) |
| **ARM** | Weak | Extensive | Medium |
| **RISC-V RVWMO** | Weak | Extensive | Medium |
| **RISC-V RVTSO** | Strong | Store→Load only | Low |

---

### Ordering Guarantees

| Operation Pair | RISC-V RVWMO | x86 TSO | ARM | SC |
|----------------|--------------|---------|-----|-----|
| Load → Load | ✗ | ✓ | ✗ | ✓ |
| Load → Store | ✗ | ✓ | ✗ | ✓ |
| Store → Store | ✗ | ✓ | ✗ | ✓ |
| Store → Load | ✗ | ✗ | ✗ | ✓ |

✓ = Ordered by default
✗ = Can be reordered (need fence)

---

## F.6 FENCE Equivalents Across Architectures

| RISC-V | x86 | ARM | Purpose |
|--------|-----|-----|---------|
| `fence rw, rw` | `MFENCE` | `DMB SY` | Full barrier |
| `fence w, w` | `SFENCE` | `DMB ST` | Store barrier |
| `fence r, r` | `LFENCE` | `DMB LD` | Load barrier |
| `fence r, rw` | (implicit) | `DMB LD` | Acquire |
| `fence rw, w` | (implicit) | `DMB ST` | Release |
| `fence.i` | (implicit) | `ISB` | Instruction sync |
| `fence.tso` | (implicit) | - | TSO ordering |

---

## F.7 Acquire/Release Semantics

### Acquire Semantics

**Meaning**: 沒有 acquire 之後的 memory operation 可以移到它之前。

**Use Case**: 在 acquire lock 之後、存取 protected data 之前。

**Implementation**：

```assembly
# Option 1: Atomic with .aq
lr.w.aq t0, (a0)

# Option 2: Load + fence
lw t0, 0(a0)
fence r, rw
```

---

### Release Semantics

**Meaning**: 沒有 release 之前的 memory operation 可以移到它之後。

**Use Case**: 在存取 protected data 之後、release lock 之前。

**Implementation**：

```assembly
# Option 1: Atomic with .rl
amoswap.w.rl zero, zero, (a0)

# Option 2: Fence + store
fence rw, w
sw zero, 0(a0)
```

---

### Acquire-Release Pair

**Complete Lock Example**：

```assembly
# Acquire
acquire:
    lr.w.aq t0, (a0)
    bnez t0, acquire
    li t1, 1
    sc.w.aq t2, t1, (a0)
    bnez t2, acquire

# Critical section
    # ... protected operations ...

# Release
    amoswap.w.rl zero, zero, (a0)
```

---

## F.8 Common Pitfalls

### Pitfall 1: Missing Fences

**Wrong**：

```assembly
# Producer
sw a0, 0(s0)      # Write data
sw a1, 0(s1)      # Write flag

# Consumer
lw t0, 0(s1)      # Read flag
lw t1, 0(s0)      # Read data (might be stale!)
```

**Correct**：

```assembly
# Producer
sw a0, 0(s0)
fence w, w        # Add fence!
sw a1, 0(s1)

# Consumer
lw t0, 0(s1)
fence r, r        # Add fence!
lw t1, 0(s0)
```

---

### Pitfall 2: Wrong Fence Type

**Wrong**（使用 load-load fence 進行 release）：

```assembly
# Critical section
fence r, r        # Wrong! Doesn't order stores
sw zero, 0(a0)    # Release lock
```

**Correct**：

```assembly
# Critical section
fence rw, w       # Correct! Orders all ops before stores
sw zero, 0(a0)
```

---

### Pitfall 3: Forgetting .aq/.rl on Atomics

**Wrong**：

```assembly
lr.w t0, (a0)     # Missing .aq!
# Critical section
amoswap.w zero, zero, (a0)  # Missing .rl!
```

**Correct**：

```assembly
lr.w.aq t0, (a0)
# Critical section
amoswap.w.rl zero, zero, (a0)
```

---

### Pitfall 4: Data Race

**Wrong**（沒有 synchronization）：

```assembly
# Hart 0
sw a0, 0(s0)      # Write shared variable

# Hart 1
lw t0, 0(s0)      # Read shared variable (DATA RACE!)
```

**Correct**（使用 lock 或 atomic）：

```assembly
# Hart 0
# ... acquire lock ...
sw a0, 0(s0)
# ... release lock ...

# Hart 1
# ... acquire lock ...
lw t0, 0(s0)
# ... release lock ...
```

---

## F.9 Quick Decision Tree

**Do I need a fence?**

```
Are multiple harts accessing shared memory?
├─ No → No fence needed
└─ Yes → Continue

Are the accesses synchronized (locks, atomics)?
├─ Yes → Fence included in lock/atomic
└─ No → Continue

Are the accesses to the same address?
├─ Yes → No fence needed (hardware preserves order)
└─ No → FENCE REQUIRED!

What type of fence?
├─ Publishing data? → fence w, w
├─ Consuming data? → fence r, r
├─ Acquiring lock? → fence r, rw (or .aq)
├─ Releasing lock? → fence rw, w (or .rl)
└─ Not sure? → fence rw, rw (full fence)
```

---

## F.10 References

- **RISC-V Memory Model Specification**: Chapter 14 of RISC-V ISA Manual
- **RVWMO Formal Specification**: https://github.com/riscv/riscv-isa-manual
- **Memory Model Tools**: herd7, rmem (for verification)
- **Linux Kernel Memory Barriers**: Documentation/memory-barriers.txt

---
