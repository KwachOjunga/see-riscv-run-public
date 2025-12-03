# Appendix F. Memory Model Quick Reference

**RISC-V Weak Memory Ordering (RVWMO) Quick Reference**

---

This appendix provides a quick reference for RISC-V's memory model (RVWMO). Understanding memory ordering is essential for writing correct concurrent code on RISC-V.

---

## F.1 Memory Ordering Basics

### What Can Be Reordered?

RISC-V Weak Memory Ordering (RVWMO) allows extensive reordering:

| Reordering | Allowed? | Exception |
|------------|----------|-----------|
| **Load → Load** | ✓ Yes | Same address, or FENCE |
| **Load → Store** | ✓ Yes | Same address, or FENCE |
| **Store → Store** | ✓ Yes | Same address, or FENCE |
| **Store → Load** | ✓ Yes | Same address, or FENCE |

**Key Point**: Almost everything can be reordered unless:

1. Operations access the same address (overlapping)
2. Operations are separated by a FENCE instruction
3. Operations have data/control dependencies
4. Operations use acquire/release atomics

### Preserved Program Order (PPO)

**Preserved Program Order** is the subset of program order that MUST be respected:

1. **Overlapping addresses**: `SW` to X, then `LW` from X → always in order
2. **Explicit fences**: Operations separated by `FENCE` → always in order
3. **Acquire/Release**: Atomic operations with `.aq` or `.rl` → enforce ordering
4. **Dependencies**: Data dependencies (e.g., `LW` then use result) → always in order
5. **Control dependencies**: Branch then dependent operation → certain orderings preserved

---

## F.2 FENCE Instruction Reference

### FENCE Syntax

```assembly
fence pred, succ
```

- **pred** (predecessor): Operations before fence (r, w, or rw)
- **succ** (successor): Operations after fence (r, w, or rw)

### Common FENCE Variants

| FENCE | Meaning | Use Case |
|-------|---------|----------|
| `fence rw, rw` | Full fence | Strongest barrier, orders everything |
| `fence w, w` | Store-store fence | Ensure stores visible in order |
| `fence r, r` | Load-load fence | Ensure loads happen in order |
| `fence r, rw` | Acquire fence | After acquiring lock |
| `fence rw, w` | Release fence | Before releasing lock |
| `fence.i` | Instruction fence | After code modification (JIT, self-modifying code) |
| `fence.tso` | TSO fence | x86-compatible ordering |

### FENCE Examples

**Full Fence** (strongest):

```assembly
sw a0, 0(s0)      # Store 1
fence rw, rw      # Full fence
lw t0, 0(s1)      # Load 1
```

All operations before fence complete before any operation after fence.

**Store-Store Fence** (publish pattern):

```assembly
sw a0, 0(s0)      # Write data
fence w, w        # Ensure data written first
sw a1, 0(s1)      # Write flag
```

Ensures stores become visible in order.

**Load-Load Fence** (consume pattern):

```assembly
lw t0, 0(s1)      # Read flag
fence r, r        # Ensure flag read first
lw t1, 0(s0)      # Read data
```

Ensures loads happen in order.

**Acquire Fence** (after lock acquisition):

```assembly
lr.w.aq t0, (a0)  # Acquire lock (with .aq)
# OR
lw t0, 0(a0)      # Read lock
fence r, rw       # Acquire fence
# ... critical section ...
```

Prevents operations in critical section from moving before lock acquisition.

**Release Fence** (before lock release):

```assembly
# ... critical section ...
fence rw, w       # Release fence
sw zero, 0(a0)    # Release lock
```

Prevents operations in critical section from moving after lock release.

---

## F.3 Atomic Instructions

### Load-Reserved / Store-Conditional (LR/SC)

**Syntax**:

```assembly
lr.w rd, (rs1)           # Load-reserved word
lr.d rd, (rs1)           # Load-reserved doubleword
sc.w rd, rs2, (rs1)      # Store-conditional word
sc.d rd, rs2, (rs1)      # Store-conditional doubleword
```

**Ordering Annotations**:

- `.aq` (acquire): No later operations can move before this
- `.rl` (release): No earlier operations can move after this
- `.aqrl` (both): Full ordering

**Example: Atomic Increment**

```assembly
retry:
    lr.w t0, (a0)         # Load current value
    addi t0, t0, 1        # Increment
    sc.w t1, t0, (a0)     # Try to store
    bnez t1, retry        # Retry if failed (t1 != 0)
```

**Example: Spinlock Acquire**

```assembly
acquire_lock:
    lr.w.aq t0, (a0)      # Load-reserved with acquire
    bnez t0, acquire_lock # If locked, retry
    li t1, 1
    sc.w.aq t2, t1, (a0)  # Try to acquire
    bnez t2, acquire_lock # Retry if failed
```

### Atomic Memory Operations (AMO)

**Syntax**:

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

**Ordering Annotations**: Same as LR/SC (`.aq`, `.rl`, `.aqrl`)

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

**Problem**: Producer writes data, then sets flag. Consumer waits for flag, then reads data.

**Solution**:

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

**Why fences are needed**:

- Without `fence w, w`: Flag might be visible before data
- Without `fence r, r`: Data might be read before flag is checked

---

### Pattern 2: Spinlock (LR/SC)

**Acquire**:

```assembly
acquire_lock:
    lr.w.aq t0, (a0)      # Load-reserved with acquire
    bnez t0, acquire_lock # If locked, retry
    li t1, 1
    sc.w.aq t2, t1, (a0)  # Try to set lock
    bnez t2, acquire_lock # Retry if SC failed
    # Lock acquired, critical section follows
```

**Release**:

```assembly
    # Critical section
    amoswap.w.rl zero, zero, (a0)  # Release lock
    # OR
    fence rw, w
    sw zero, 0(a0)
```

---

### Pattern 3: Spinlock (AMOSWAP)

**Acquire**:

```assembly
acquire_lock:
    li t0, 1
    amoswap.w.aq t1, t0, (a0)  # Atomic swap
    bnez t1, acquire_lock       # If old value != 0, retry
    # Lock acquired
```

**Release**:

```assembly
    amoswap.w.rl zero, zero, (a0)  # Release lock
```

---

### Pattern 4: Dekker's Algorithm (Mutual Exclusion)

**Hart 0**:

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

**Hart 1**: (symmetric, swap flag0 and flag1)

---

### Pattern 5: Producer-Consumer Queue

**Producer**:

```assembly
    # Write data to queue[tail]
    sw a0, 0(s0)

    # Increment tail
    fence w, w         # Ensure data written before tail update
    addi s1, s1, 1
    sw s1, tail_ptr
```

**Consumer**:

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

**Barrier Wait**:

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

**Meaning**: No memory operations after the acquire can move before it.

**Use Case**: After acquiring a lock, before accessing protected data.

**Implementation**:

```assembly
# Option 1: Atomic with .aq
lr.w.aq t0, (a0)

# Option 2: Load + fence
lw t0, 0(a0)
fence r, rw
```

### Release Semantics

**Meaning**: No memory operations before the release can move after it.

**Use Case**: After accessing protected data, before releasing a lock.

**Implementation**:

```assembly
# Option 1: Atomic with .rl
amoswap.w.rl zero, zero, (a0)

# Option 2: Fence + store
fence rw, w
sw zero, 0(a0)
```

### Acquire-Release Pair

**Complete Lock Example**:

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

**Wrong**:

```assembly
# Producer
sw a0, 0(s0)      # Write data
sw a1, 0(s1)      # Write flag

# Consumer
lw t0, 0(s1)      # Read flag
lw t1, 0(s0)      # Read data (might be stale!)
```

**Correct**:

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

**Wrong** (using load-load fence for release):

```assembly
# Critical section
fence r, r        # Wrong! Doesn't order stores
sw zero, 0(a0)    # Release lock
```

**Correct**:

```assembly
# Critical section
fence rw, w       # Correct! Orders all ops before stores
sw zero, 0(a0)
```

---

### Pitfall 3: Forgetting .aq/.rl on Atomics

**Wrong**:

```assembly
lr.w t0, (a0)     # Missing .aq!
# Critical section
amoswap.w zero, zero, (a0)  # Missing .rl!
```

**Correct**:

```assembly
lr.w.aq t0, (a0)
# Critical section
amoswap.w.rl zero, zero, (a0)
```

---

### Pitfall 4: Data Race

**Wrong** (no synchronization):

```assembly
# Hart 0
sw a0, 0(s0)      # Write shared variable

# Hart 1
lw t0, 0(s0)      # Read shared variable (DATA RACE!)
```

**Correct** (use lock or atomic):

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

<!-- METADATA (for authoring only, remove before publication)

## Appendix Metadata

**Appendix**: F - Memory Model Quick Reference
**Version**: Draft v1p0
**Word Count**: ~1,800 words
**Tables**: 8+
**Code Examples**: 20+

-->
