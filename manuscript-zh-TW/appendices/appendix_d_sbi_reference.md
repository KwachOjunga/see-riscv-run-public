# Appendix D. SBI Call Reference

**Supervisor Binary Interface (SBI) Quick Reference**

---

> 💡 **使用指南**：本附錄是 S-mode 呼叫 M-mode 服務的「API 手冊」。當你忘記 a7 是 EID 還是 FID 時，翻到這裡。

---

## 🎯 SBI Calling Convention 圖解

```text
┌─────────────────────────────────────────────────────────────────┐
│                    S-mode (Kernel/OS)                           │
│                                                                 │
│    ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐             │
│    │   a7    │ │   a6    │ │ a0-a5   │ │  ecall  │             │
│    │  EID    │ │  FID    │ │  Args   │ │ ──────► │             │
│    └─────────┘ └─────────┘ └─────────┘ └─────────┘             │
│       │           │           │                                 │
├───────┼───────────┼───────────┼─────────────────────────────────┤
│       ▼           ▼           ▼                                 │
│    ┌─────────────────────────────────────────┐                 │
│    │         Trap to M-mode (OpenSBI)        │                 │
│    └─────────────────────────────────────────┘                 │
│       │                                                         │
│       ▼                                                         │
│    ┌─────────┐ ┌─────────┐                                     │
│    │   a0    │ │   a1    │                                     │
│    │ Error   │ │ Value   │                                     │
│    │  Code   │ │(Return) │                                     │
│    └─────────┘ └─────────┘                                     │
│                                                                 │
│                    M-mode (OpenSBI Firmware)                    │
└─────────────────────────────────────────────────────────────────┘
```

**記憶口訣**：`a7 = EID (哪個 Extension)`, `a6 = FID (哪個 Function)`

---

## 📋 常用 EID 速查表

| EID (Hex) | EID (ASCII) | 名稱 | 用途 | 常用 FID |
|-----------|-------------|------|------|----------|
| `0x10` | — | **Base** | 查詢 SBI 版本/廠商 | 0=版本, 3=探測 Extension |
| `0x54494D45` | "TIME" | **Timer** | 設定 Timer 中斷 | 0=set_timer |
| `0x735049` | "sPI" | **IPI** | 跨核心中斷 | 0=send_ipi |
| `0x52464E43` | "RFNC" | **RFENCE** | 遠端 TLB 刷新 | 0-6 (各種 fence) |
| `0x4442434E` | "DBCN" | **Debug Console** | 除錯輸出 | 0=write, 1=read |
| `0x48534D` | "HSM" | **Hart State Mgmt** | 啟動/停止 Hart | 0=start, 1=stop |

---

## 🛠️ 常用 SBI Wrapper (Copy-Paste Ready)

### sbi_call 通用介面

```c
struct sbiret {
    long error;
    long value;
};

static inline struct sbiret sbi_call(long eid, long fid,
    long a0, long a1, long a2, long a3, long a4, long a5)
{
    struct sbiret ret;
    register long r_a0 asm("a0") = a0;
    register long r_a1 asm("a1") = a1;
    register long r_a2 asm("a2") = a2;
    register long r_a3 asm("a3") = a3;
    register long r_a4 asm("a4") = a4;
    register long r_a5 asm("a5") = a5;
    register long r_a6 asm("a6") = fid;
    register long r_a7 asm("a7") = eid;

    asm volatile("ecall"
        : "+r"(r_a0), "+r"(r_a1)
        : "r"(r_a2), "r"(r_a3), "r"(r_a4), "r"(r_a5), "r"(r_a6), "r"(r_a7)
        : "memory");

    ret.error = r_a0;
    ret.value = r_a1;
    return ret;
}
```

### 常用功能 Wrapper

```c
// 1. 設定 Timer 中斷 (最常用!)
static inline void sbi_set_timer(uint64_t stime_value) {
    sbi_call(0x54494D45, 0, stime_value, 0, 0, 0, 0, 0);
}

// 2. 輸出一個字元 (Debug Console)
static inline void sbi_debug_console_write_byte(char c) {
    sbi_call(0x4442434E, 2, c, 0, 0, 0, 0, 0);
}

// 3. 查詢 SBI 版本
static inline long sbi_get_spec_version(void) {
    struct sbiret ret = sbi_call(0x10, 0, 0, 0, 0, 0, 0, 0);
    return ret.value;
}

// 4. 探測是否支援某 Extension
static inline long sbi_probe_extension(long eid) {
    struct sbiret ret = sbi_call(0x10, 3, eid, 0, 0, 0, 0, 0);
    return ret.value;  // 0 = 不支援, 非 0 = 支援
}
```

---

## ⚠️ 常見陷阱

### 陷阱 1：EID 和 FID 順序搞混

**症狀**：SBI 呼叫返回 `SBI_ERR_NOT_SUPPORTED`。

**原因**：a7 和 a6 放反了。

```c
// ❌ 錯誤：EID 和 FID 放反
register long a6 asm("a6") = 0x10;        // 應該是 FID
register long a7 asm("a7") = 0;           // 應該是 EID

// ✅ 正確：a7=EID, a6=FID
register long a6 asm("a6") = 0;           // FID = 0 (get_spec_version)
register long a7 asm("a7") = 0x10;        // EID = 0x10 (Base Extension)
```

### 陷阱 2：忘記檢查返回的錯誤碼

**症狀**：SBI 呼叫失敗但程式繼續執行，導致後續錯誤難以追蹤。

```c
// ❌ 錯誤：忽略錯誤碼
sbi_set_timer(next_time);

// ✅ 正確：檢查錯誤
struct sbiret ret = sbi_call(0x54494D45, 0, next_time, 0, 0, 0, 0, 0);
if (ret.error != 0) {
    panic("sbi_set_timer failed: %ld", ret.error);
}
```

### 陷阱 3：在 M-mode 呼叫 SBI

**症狀**：程式當機或進入無限迴圈。

**原因**：SBI 是給 S-mode 呼叫的。在 M-mode 呼叫 ecall 會 Trap 回 M-mode 自己。

**解決**：在 M-mode 直接操作 CSR，不要透過 SBI。

---

本附錄提供 RISC-V SBI（Supervisor Binary Interface）call 的完整參考。SBI 定義了 supervisor mode（S-mode）software 和 machine mode（M-mode）firmware 之間的標準介面，使 operating system 能夠在不同的 RISC-V platform 上移植。

---

## D.1 SBI Calling Convention

### Register Usage

**Input Registers**：

| Register | Purpose |
|----------|---------|
| **a7** | Extension ID (EID) |
| **a6** | Function ID (FID) |
| **a0** | Parameter 0 / Return value |
| **a1** | Parameter 1 / Return value (optional) |
| **a2** | Parameter 2 |
| **a3** | Parameter 3 |
| **a4** | Parameter 4 |
| **a5** | Parameter 5 |

**Output Registers**：

| Register | Purpose |
|----------|---------|
| **a0** | Error code (0 = success, negative = error) |
| **a1** | Return value (function-specific) |

**Preserved Registers**: 除了 a0 和 a1 之外的所有 register 在 SBI call 之間都會被保留。

### Invocation

```c
// S-mode code invokes SBI using ecall instruction
register unsigned long a0 asm("a0") = param0;
register unsigned long a1 asm("a1") = param1;
register unsigned long a6 asm("a6") = function_id;
register unsigned long a7 asm("a7") = extension_id;

asm volatile("ecall"
             : "+r"(a0), "+r"(a1)
             : "r"(a6), "r"(a7)
             : "memory");

// a0 contains error code, a1 contains return value
```

---

## D.2 SBI Error Codes

| Code | Name | Description |
|------|------|-------------|
| 0 | SBI_SUCCESS | Operation completed successfully |
| -1 | SBI_ERR_FAILED | Operation failed |
| -2 | SBI_ERR_NOT_SUPPORTED | Function not supported |
| -3 | SBI_ERR_INVALID_PARAM | Invalid parameter |
| -4 | SBI_ERR_DENIED | Permission denied |
| -5 | SBI_ERR_INVALID_ADDRESS | Invalid address |
| -6 | SBI_ERR_ALREADY_AVAILABLE | Resource already available |
| -7 | SBI_ERR_ALREADY_STARTED | Already started |
| -8 | SBI_ERR_ALREADY_STOPPED | Already stopped |

---

## D.3 Base Extension (EID = 0x10)

**Purpose**: 查詢 SBI implementation detail 和 supported extension。

### D.3.1 Get SBI Specification Version (FID = 0)

**Returns**: SBI specification version。

- **a1[31:24]**: Major version
- **a1[23:0]**: Minor version

```c
long sbi_get_spec_version(void) {
    register unsigned long a0 asm("a0");
    register unsigned long a1 asm("a1");
    register unsigned long a6 asm("a6") = 0;
    register unsigned long a7 asm("a7") = 0x10;
    
    asm volatile("ecall" : "=r"(a0), "=r"(a1) : "r"(a6), "r"(a7) : "memory");
    return a1;  // Version in a1
}
```

---

### D.3.2 Get SBI Implementation ID (FID = 1)

**Returns**: Implementation ID。

| ID | Implementation |
|----|----------------|
| 0 | Berkeley Boot Loader (BBL) |
| 1 | OpenSBI |
| 2 | Xvisor |
| 3 | KVM |
| 4 | RustSBI |
| 5 | Diosix |

---

### D.3.3 Get SBI Implementation Version (FID = 2)

**Returns**: Implementation-specific version number。

---

### D.3.4 Probe SBI Extension (FID = 3)

**Parameters**：

- **a0**: Extension ID to probe

**Returns**：

- **a1**: 0 = not available, 1 = available

```c
long sbi_probe_extension(long extension_id) {
    register unsigned long a0 asm("a0") = extension_id;
    register unsigned long a1 asm("a1");
    register unsigned long a6 asm("a6") = 3;
    register unsigned long a7 asm("a7") = 0x10;
    
    asm volatile("ecall" : "+r"(a0), "=r"(a1) : "r"(a6), "r"(a7) : "memory");
    return a1;
}
```

---

### D.3.5 Get Machine Vendor ID (FID = 4)

**Returns**: mvendorid CSR value。

---

### D.3.6 Get Machine Architecture ID (FID = 5)

**Returns**: marchid CSR value。

---

### D.3.7 Get Machine Implementation ID (FID = 6)

**Returns**: mimpid CSR value。

---

## D.4 Timer Extension (EID = 0x54494D45 "TIME")

**Purpose**: 程式化 timer interrupt。

### D.4.1 Set Timer (FID = 0)

**Parameters**：

- **a0**: Timer value（absolute time in ticks）

**Description**: 程式化 timer 在指定時間觸發。清除 pending timer interrupt。

```c
void sbi_set_timer(uint64_t stime_value) {
    register unsigned long a0 asm("a0") = stime_value;
    register unsigned long a6 asm("a6") = 0;
    register unsigned long a7 asm("a7") = 0x54494D45;

    asm volatile("ecall" : "+r"(a0) : "r"(a6), "r"(a7) : "memory");
}

// Usage: Set timer to fire in 1 second (assuming 10 MHz timebase)
uint64_t current_time;
asm volatile("rdtime %0" : "=r"(current_time));
sbi_set_timer(current_time + 10000000);
```

---

## D.5 IPI Extension (EID = 0x735049 "sPI")

**Purpose**: 發送 inter-processor interrupt。

### D.5.1 Send IPI (FID = 0)

**Parameters**：

- **a0**: Hart mask（target hart 的 bitmap）
- **a1**: Hart mask base（mask 的 base hart ID）

**Description**: 發送 supervisor software interrupt 到指定的 hart。

```c
long sbi_send_ipi(unsigned long hart_mask, unsigned long hart_mask_base) {
    register unsigned long a0 asm("a0") = hart_mask;
    register unsigned long a1 asm("a1") = hart_mask_base;
    register unsigned long a6 asm("a6") = 0;
    register unsigned long a7 asm("a7") = 0x735049;

    asm volatile("ecall" : "+r"(a0), "+r"(a1) : "r"(a6), "r"(a7) : "memory");
    return a0;
}

// Usage: Send IPI to harts 1, 2, 3
sbi_send_ipi(0b1110, 0);  // Bits 1, 2, 3 set, base = 0

// Send IPI to hart 65 (bit 1 of mask, base = 64)
sbi_send_ipi(0b10, 64);
```

---

## D.6 RFENCE Extension (EID = 0x52464E43 "RFNC")

**Purpose**: Remote fence operation，用於 TLB 和 instruction cache synchronization。

### D.6.1 Remote FENCE.I (FID = 0)

**Parameters**：

- **a0**: Hart mask
- **a1**: Hart mask base

**Description**: 在 remote hart 上執行 FENCE.I。

```c
long sbi_remote_fence_i(unsigned long hart_mask, unsigned long hart_mask_base) {
    register unsigned long a0 asm("a0") = hart_mask;
    register unsigned long a1 asm("a1") = hart_mask_base;
    register unsigned long a6 asm("a6") = 0;
    register unsigned long a7 asm("a7") = 0x52464E43;

    asm volatile("ecall" : "+r"(a0), "+r"(a1) : "r"(a6), "r"(a7) : "memory");
    return a0;
}
```

---

### D.6.2 Remote SFENCE.VMA (FID = 1)

**Parameters**：

- **a0**: Hart mask
- **a1**: Hart mask base
- **a2**: Start address（virtual address）
- **a3**: Size（number of pages）

**Description**: 在 remote hart 上對指定 address range 執行 SFENCE.VMA。

```c
long sbi_remote_sfence_vma(unsigned long hart_mask, unsigned long hart_mask_base,
                           unsigned long start_addr, unsigned long size) {
    register unsigned long a0 asm("a0") = hart_mask;
    register unsigned long a1 asm("a1") = hart_mask_base;
    register unsigned long a2 asm("a2") = start_addr;
    register unsigned long a3 asm("a3") = size;
    register unsigned long a6 asm("a6") = 1;
    register unsigned long a7 asm("a7") = 0x52464E43;

    asm volatile("ecall" : "+r"(a0), "+r"(a1)
                 : "r"(a2), "r"(a3), "r"(a6), "r"(a7) : "memory");
    return a0;
}

// Usage: Flush TLB for address range on all harts
sbi_remote_sfence_vma(~0UL, 0, 0x80000000, 4096);  // Flush 1 page
```

---

### D.6.3 Remote SFENCE.VMA with ASID (FID = 2)

**Parameters**：

- **a0**: Hart mask
- **a1**: Hart mask base
- **a2**: Start address
- **a3**: Size
- **a4**: ASID

**Description**: 在 remote hart 上執行 SFENCE.VMA with ASID。

```c
long sbi_remote_sfence_vma_asid(unsigned long hart_mask, unsigned long hart_mask_base,
                                unsigned long start_addr, unsigned long size,
                                unsigned long asid) {
    register unsigned long a0 asm("a0") = hart_mask;
    register unsigned long a1 asm("a1") = hart_mask_base;
    register unsigned long a2 asm("a2") = start_addr;
    register unsigned long a3 asm("a3") = size;
    register unsigned long a4 asm("a4") = asid;
    register unsigned long a6 asm("a6") = 2;
    register unsigned long a7 asm("a7") = 0x52464E43;

    asm volatile("ecall" : "+r"(a0), "+r"(a1)
                 : "r"(a2), "r"(a3), "r"(a4), "r"(a6), "r"(a7) : "memory");
    return a0;
}
```

---

### D.6.4 Remote HFENCE.GVMA (FID = 3)

**Parameters**：

- **a0**: Hart mask
- **a1**: Hart mask base
- **a2**: Guest physical address
- **a3**: Size

**Description**: 在 remote hart 上執行 HFENCE.GVMA（hypervisor extension）。

---

### D.6.5 Remote HFENCE.GVMA with VMID (FID = 4)

**Parameters**：

- **a0**: Hart mask
- **a1**: Hart mask base
- **a2**: Guest physical address
- **a3**: Size
- **a4**: VMID

**Description**: 在 remote hart 上執行 HFENCE.GVMA with VMID。

---

### D.6.6 Remote HFENCE.VVMA (FID = 5)

**Parameters**：

- **a0**: Hart mask
- **a1**: Hart mask base
- **a2**: Guest virtual address
- **a3**: Size

**Description**: 在 remote hart 上執行 HFENCE.VVMA。

---

### D.6.7 Remote HFENCE.VVMA with ASID (FID = 6)

**Parameters**：

- **a0**: Hart mask
- **a1**: Hart mask base
- **a2**: Guest virtual address
- **a3**: Size
- **a4**: ASID

**Description**: 在 remote hart 上執行 HFENCE.VVMA with ASID。

---

## D.7 Hart State Management Extension (EID = 0x48534D "HSM")

**Purpose**: 管理 hart lifecycle（start、stop、suspend）。

### D.7.1 Hart Start (FID = 0)

**Parameters**：

- **a0**: Hart ID
- **a1**: Start address（physical address）
- **a2**: Opaque parameter（passed to hart in a1）

**Returns**: SBI_SUCCESS or error code

**Description**: 在指定 address 啟動指定的 hart。

```c
long sbi_hart_start(unsigned long hartid, unsigned long start_addr,
                    unsigned long opaque) {
    register unsigned long a0 asm("a0") = hartid;
    register unsigned long a1 asm("a1") = start_addr;
    register unsigned long a2 asm("a2") = opaque;
    register unsigned long a6 asm("a6") = 0;
    register unsigned long a7 asm("a7") = 0x48534D;

    asm volatile("ecall" : "+r"(a0), "+r"(a1)
                 : "r"(a2), "r"(a6), "r"(a7) : "memory");
    return a0;
}

// Usage: Start hart 1 at address 0x80200000
sbi_hart_start(1, 0x80200000, 0);
```

---

### D.7.2 Hart Stop (FID = 1)

**Parameters**: None

**Returns**: Does not return on success

**Description**: 停止當前 hart。Hart 進入 stopped state。

```c
void sbi_hart_stop(void) {
    register unsigned long a6 asm("a6") = 1;
    register unsigned long a7 asm("a7") = 0x48534D;

    asm volatile("ecall" : : "r"(a6), "r"(a7) : "memory");
}
```

---

### D.7.3 Hart Get Status (FID = 2)

**Parameters**：

- **a0**: Hart ID

**Returns**：

- **a1**: Hart status

**Hart Status Values**：

| Value | Status |
|-------|--------|
| 0 | STARTED |
| 1 | STOPPED |
| 2 | START_PENDING |
| 3 | STOP_PENDING |
| 4 | SUSPENDED |
| 5 | SUSPEND_PENDING |
| 6 | RESUME_PENDING |

```c
long sbi_hart_get_status(unsigned long hartid) {
    register unsigned long a0 asm("a0") = hartid;
    register unsigned long a1 asm("a1");
    register unsigned long a6 asm("a6") = 2;
    register unsigned long a7 asm("a7") = 0x48534D;

    asm volatile("ecall" : "+r"(a0), "=r"(a1) : "r"(a6), "r"(a7) : "memory");
    return a1;
}
```

---

### D.7.4 Hart Suspend (FID = 3)

**Parameters**：

- **a0**: Suspend type
- **a1**: Resume address
- **a2**: Opaque parameter

**Suspend Types**：

| Value | Type |
|-------|------|
| 0x00000000 | RETENTIVE (retain state, low latency) |
| 0x80000000 | NON_RETENTIVE (lose state, save/restore required) |

---

## D.8 System Reset Extension (EID = 0x53525354 "SRST")

**Purpose**: System-wide reset 和 shutdown。

### D.8.1 System Reset (FID = 0)

**Parameters**：

- **a0**: Reset type
- **a1**: Reset reason

**Returns**: Does not return on success

**Reset Types**：

| Value | Type |
|-------|------|
| 0x00000000 | SHUTDOWN |
| 0x00000001 | COLD_REBOOT |
| 0x00000002 | WARM_REBOOT |

**Reset Reasons**：

| Value | Reason |
|-------|--------|
| 0x00000000 | NO_REASON |
| 0x00000001 | SYSTEM_FAILURE |

```c
void sbi_system_reset(unsigned long reset_type, unsigned long reset_reason) {
    register unsigned long a0 asm("a0") = reset_type;
    register unsigned long a1 asm("a1") = reset_reason;
    register unsigned long a6 asm("a6") = 0;
    register unsigned long a7 asm("a7") = 0x53525354;

    asm volatile("ecall" : : "r"(a0), "r"(a1), "r"(a6), "r"(a7) : "memory");
    __builtin_unreachable();
}

// Usage: Reboot the system
#define SBI_RESET_TYPE_COLD_REBOOT 1
#define SBI_RESET_REASON_NO_REASON 0
sbi_system_reset(SBI_RESET_TYPE_COLD_REBOOT, SBI_RESET_REASON_NO_REASON);
```

---

## D.9 Performance Monitoring Unit Extension (EID = 0x504D55 "PMU")

**Purpose**: 配置和讀取 performance counter。

### D.9.1 Get Number of Counters (FID = 0)

**Returns**：

- **a1**: Number of counters

---

### D.9.2 Get Counter Info (FID = 1)

**Parameters**：

- **a0**: Counter index

**Returns**：

- **a1**: Counter info

---

### D.9.3 Configure Matching Counters (FID = 2)

**Parameters**：

- **a0**: Counter index base
- **a1**: Counter mask
- **a2**: Config flags
- **a3**: Event index
- **a4**: Event data

**Returns**: Number of counters configured

---

### D.9.4 Start Counters (FID = 3)

**Parameters**：

- **a0**: Counter index base
- **a1**: Counter mask
- **a2**: Start flags
- **a3**: Initial value

---

### D.9.5 Stop Counters (FID = 4)

**Parameters**：

- **a0**: Counter index base
- **a1**: Counter mask
- **a2**: Stop flags

---

### D.9.6 Read Firmware Counter (FID = 5)

**Parameters**：

- **a0**: Counter index

**Returns**：

- **a1**: Counter value

---

## D.10 Legacy Extensions (Deprecated)

**Note**: 這些 extension 已被棄用，但仍廣泛用於相容性。

### D.10.1 Console Putchar (EID = 0x01)

**Parameters**：

- **a0**: Character to output

```c
void sbi_console_putchar(int ch) {
    register unsigned long a0 asm("a0") = ch;
    register unsigned long a7 asm("a7") = 0x01;

    asm volatile("ecall" : "+r"(a0) : "r"(a7) : "memory");
}
```

---

### D.10.2 Console Getchar (EID = 0x02)

**Returns**：

- **a0**: Character read, or -1 if no character available

```c
int sbi_console_getchar(void) {
    register unsigned long a0 asm("a0");
    register unsigned long a7 asm("a7") = 0x02;

    asm volatile("ecall" : "=r"(a0) : "r"(a7) : "memory");
    return a0;
}
```

---

### D.10.3 Legacy Set Timer (EID = 0x00)

**Parameters**：

- **a0**: Timer value

**Note**: 使用 Timer Extension（0x54494D45）代替。

---

### D.10.4 Legacy Clear IPI (EID = 0x03)

**Note**: 已棄用。直接清除 sip.SSIP bit。

---

### D.10.5 Legacy Send IPI (EID = 0x04)

**Parameters**：

- **a0**: Hart mask pointer

**Note**: 使用 IPI Extension（0x735049）代替。

---

### D.10.6 Legacy Remote FENCE.I (EID = 0x05)

**Parameters**：

- **a0**: Hart mask pointer

**Note**: 使用 RFENCE Extension（0x52464E43）代替。

---

### D.10.7 Legacy Remote SFENCE.VMA (EID = 0x06)

**Parameters**：

- **a0**: Hart mask pointer
- **a1**: Start address
- **a2**: Size

**Note**: 使用 RFENCE Extension 代替。

---

### D.10.8 Legacy Remote SFENCE.VMA with ASID (EID = 0x07)

**Parameters**：

- **a0**: Hart mask pointer
- **a1**: Start address
- **a2**: Size
- **a3**: ASID

**Note**: 使用 RFENCE Extension 代替。

---

### D.10.9 Legacy System Shutdown (EID = 0x08)

**Note**: 使用 System Reset Extension（0x53525354）代替。

---

## D.11 Extension ID Summary

| EID | Name | Description |
|-----|------|-------------|
| 0x10 | BASE | Base extension (version, probe) |
| 0x54494D45 | TIME | Timer programming |
| 0x735049 | sPI | Inter-processor interrupts |
| 0x52464E43 | RFNC | Remote fence operations |
| 0x48534D | HSM | Hart state management |
| 0x53525354 | SRST | System reset |
| 0x504D55 | PMU | Performance monitoring |
| 0x4442434E | DBCN | Debug console |
| 0x53555350 | SUSP | System suspend |
| 0x43505043 | CPPC | Collaborative Processor Performance Control |
| 0x4E41434C | NACL | Nested Acceleration |

---

## D.12 Common Usage Patterns

### Early Boot Console Output

```c
void early_printk(const char *str) {
    while (*str) {
        if (*str == '\n')
            sbi_console_putchar('\r');
        sbi_console_putchar(*str++);
    }
}
```

---

### Timer-based Scheduling

```c
void setup_timer_interrupt(uint64_t interval_us) {
    uint64_t current_time;
    asm volatile("rdtime %0" : "=r"(current_time));

    // Assuming 10 MHz timebase (100 ns per tick)
    uint64_t ticks = interval_us * 10;
    sbi_set_timer(current_time + ticks);

    // Enable supervisor timer interrupt
    csr_set(sie, SIE_STIE);
}
```

---

### Multi-core Synchronization

```c
void flush_tlb_all_harts(void) {
    // Flush TLB on all harts
    sbi_remote_sfence_vma(~0UL, 0, 0, ~0UL);
}

void wake_up_secondary_harts(void) {
    for (int i = 1; i < num_harts; i++) {
        sbi_hart_start(i, (unsigned long)secondary_start, 0);
    }
}
```

---

### System Shutdown

```c
void system_poweroff(void) {
    sbi_system_reset(0, 0);  // SHUTDOWN, NO_REASON
    while (1);  // Should never reach here
}

void system_reboot(void) {
    sbi_system_reset(1, 0);  // COLD_REBOOT, NO_REASON
    while (1);
}
```

---

## D.13 References

- **SBI Specification**: https://github.com/riscv-non-isa/riscv-sbi-doc
- **OpenSBI Documentation**: https://github.com/riscv-software-src/opensbi
- **Linux RISC-V SBI Implementation**: arch/riscv/kernel/sbi.c

---
