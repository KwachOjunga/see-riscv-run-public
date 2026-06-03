# Appendix C. Boot Loader Reference Implementation

**Minimal RISC-V Bootloader Example**

---

> 💡 **Usage Guide**: This appendix is your "boot disk" for starting projects. When you need to write bare-metal code from scratch, copy templates directly from here.

---

## 🚀 Minimal Viable Boot Template (Copy-Paste Ready)

### Minimal Linker Script (`link.ld`)

This is the most frequently copy-pasted file in bare-metal projects:

```ld
/* link.ld - For QEMU virt machine */
OUTPUT_ARCH(riscv)
ENTRY(_start)

MEMORY {
    RAM (rwx) : ORIGIN = 0x80000000, LENGTH = 128M
}

SECTIONS {
    . = 0x80000000;

    .text : {
        *(.text.boot)       /* Ensure boot code comes first */
        *(.text .text.*)
    } > RAM

    .rodata : {
        *(.rodata .rodata.*)
    } > RAM

    .data : {
        *(.data .data.*)
    } > RAM

    .bss : {
        _bss_start = .;
        *(.bss .bss.*)
        *(COMMON)
        _bss_end = .;
    } > RAM

    . = ALIGN(16);
    . = . + 0x4000;         /* Reserve 16KB Stack */
    _stack_top = .;
}
```

### Minimal Entry Point (`entry.S`)

```assembly
# entry.S - Minimal boot code
.section .text.boot
.global _start

_start:
    # 1. Set Stack Pointer
    la sp, _stack_top

    # 2. Clear BSS section
    la t0, _bss_start
    la t1, _bss_end
clear_bss:
    bge t0, t1, bss_done
    sd zero, 0(t0)
    addi t0, t0, 8
    j clear_bss
bss_done:

    # 3. Jump to C main
    call main

    # 4. Halt after main returns
halt:
    wfi
    j halt
```

### Minimal Main (`main.c`)

```c
// main.c - Minimal Hello World (UART)
#define UART_BASE 0x10000000  // QEMU virt UART address

void uart_putc(char c) {
    volatile char *uart = (volatile char *)UART_BASE;
    *uart = c;
}

void uart_puts(const char *s) {
    while (*s) uart_putc(*s++);
}

int main(void) {
    uart_puts("Hello, RISC-V!\n");
    return 0;
}
```

### Compile and Run

```bash
# Compile
riscv64-unknown-elf-gcc -nostdlib -T link.ld \
    -o hello.elf entry.S main.c

# Run
qemu-system-riscv64 -machine virt -nographic \
    -kernel hello.elf
```

---

## 📊 Typical Boot Flow Diagram

```text
┌─────────────────────────────────────────────────────────────┐
│                     Power-On Reset                          │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  ZSBL (Zeroth-Stage Bootloader) - ROM                       │
│  • PC = Reset Vector (0x1000 or implementation-defined)     │
│  • Initialize clock, DRAM Controller                        │
│  • Jump to FSBL                                             │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  FSBL (First-Stage Bootloader) - Flash/ROM                  │
│  • Initialize SPI/SD storage device                         │
│  • Load OpenSBI to DRAM                                     │
│  • Jump to OpenSBI                                          │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  OpenSBI (M-mode Firmware)                                  │
│  • Set up PMP to protect M-mode memory                      │
│  • Initialize SBI services                                  │
│  • Set medeleg/mideleg to delegate traps                    │
│  • Jump to S-mode Kernel                                    │
└─────────────────────────┬───────────────────────────────────┘
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  Linux Kernel (S-mode)                                      │
│  • Initialize virtual memory                                │
│  • Start init process                                       │
└─────────────────────────────────────────────────────────────┘
```

---

This appendix provides a reference implementation of a minimal RISC-V bootloader. This code demonstrates the essential steps required to boot a RISC-V system from reset to loading an operating system. While production bootloaders like U-Boot are much more complex, this example illustrates the core concepts.

---

## C.1 Boot Sequence Overview

```
Power-On Reset
    ↓
Reset Vector (0x1000 or implementation-defined)
    ↓
ZSBL (Zeroth-Stage Bootloader) - ROM code
    ├─ Initialize clocks
    ├─ Initialize DRAM
    └─ Jump to FSBL
        ↓
FSBL (First-Stage Bootloader) - Flash/ROM
    ├─ Initialize storage (SPI/SD)
    ├─ Load SSBL to DRAM
    ├─ Verify SSBL (optional)
    └─ Jump to SSBL
        ↓
SSBL (Second-Stage Bootloader) - U-Boot/GRUB
    ├─ Initialize devices
    ├─ Load kernel and device tree
    ├─ Set up boot arguments
    └─ Jump to kernel
        ↓
Operating System (Linux/FreeBSD)
```

---

## C.2 ZSBL: Zeroth-Stage Bootloader

**Purpose**: Minimal ROM code to initialize DRAM and load FSBL.

**Constraints**:

- Must fit in small on-chip ROM (typically 16-64 KB)
- No DRAM available initially
- Must run from ROM or tightly-coupled memory (TCM)

### ZSBL Entry Point

```assembly
# zsbl_start.S - ZSBL entry point
# Runs in M-mode immediately after reset

.section .text.init
.global _start

_start:
    # Disable interrupts
    csrw mie, zero
    csrw mip, zero
    
    # Set up trap vector (point to error handler)
    la t0, trap_handler
    csrw mtvec, t0
    
    # Initialize stack pointer
    # Use on-chip SRAM (e.g., 0x08000000 + 16KB)
    la sp, _stack_top
    
    # Clear BSS section
    la t0, _bss_start
    la t1, _bss_end
1:
    bge t0, t1, 2f
    sd zero, 0(t0)
    addi t0, t0, 8
    j 1b
2:
    
    # Jump to C code
    call zsbl_main
    
    # Should never return
    j .

trap_handler:
    # Minimal trap handler - just hang
    j .
```

### ZSBL Main Function

```c
// zsbl_main.c - ZSBL main logic

#include <stdint.h>

// Hardware addresses (example for SiFive FU540)
#define DRAM_BASE       0x80000000
#define DRAM_SIZE       (8 * 1024 * 1024 * 1024UL)  // 8 GB
#define FSBL_LOAD_ADDR  0x80000000
#define FSBL_SIZE       (128 * 1024)  // 128 KB
#define SPI_FLASH_BASE  0x20000000

// DRAM controller registers (simplified)
#define DRAM_CTRL_BASE  0x10000000
#define DRAM_INIT_REG   (DRAM_CTRL_BASE + 0x00)
#define DRAM_STATUS_REG (DRAM_CTRL_BASE + 0x04)

void dram_init(void) {
    volatile uint32_t *init_reg = (uint32_t *)DRAM_INIT_REG;
    volatile uint32_t *status_reg = (uint32_t *)DRAM_STATUS_REG;
    
    // Trigger DRAM initialization
    *init_reg = 0x1;
    
    // Wait for DRAM ready
    while ((*status_reg & 0x1) == 0) {
        // Busy wait
    }
}

void load_fsbl(void) {
    uint8_t *src = (uint8_t *)SPI_FLASH_BASE;
    uint8_t *dst = (uint8_t *)FSBL_LOAD_ADDR;
    
    // Simple memcpy from SPI flash to DRAM
    for (size_t i = 0; i < FSBL_SIZE; i++) {
        dst[i] = src[i];
    }
}

void zsbl_main(void) {
    // 1. Initialize DRAM
    dram_init();
    
    // 2. Load FSBL from SPI flash to DRAM
    load_fsbl();
    
    // 3. Jump to FSBL
    void (*fsbl_entry)(void) = (void (*)(void))FSBL_LOAD_ADDR;
    fsbl_entry();
    
    // Should never reach here
    while (1);
}
```

---

## C.3 FSBL: First-Stage Bootloader

**Purpose**: Load second-stage bootloader (U-Boot) from storage.

**Features**:

- Initialize storage controller (SPI, SD, eMMC)
- Load SSBL image from storage
- Verify SSBL (checksum or signature)
- Jump to SSBL

### FSBL Main Function

```c
// fsbl_main.c - FSBL main logic

#include <stdint.h>

#define SSBL_LOAD_ADDR  0x80200000  // Load U-Boot at 2MB offset
#define SSBL_SIZE       (512 * 1024)  // 512 KB
#define SSBL_FLASH_OFFSET 0x40000     // Offset in SPI flash

// SPI controller registers (simplified)
#define SPI_BASE        0x10040000
#define SPI_CTRL_REG    (SPI_BASE + 0x00)
#define SPI_DATA_REG    (SPI_BASE + 0x04)
#define SPI_STATUS_REG  (SPI_BASE + 0x08)

void spi_init(void) {
    volatile uint32_t *ctrl_reg = (uint32_t *)SPI_CTRL_REG;
    
    // Configure SPI: 8-bit mode, clock divider = 4
    *ctrl_reg = 0x04;
}

void spi_read(uint32_t offset, uint8_t *buf, size_t len) {
    volatile uint32_t *data_reg = (uint32_t *)SPI_DATA_REG;
    volatile uint32_t *status_reg = (uint32_t *)SPI_STATUS_REG;
    
    // Send read command (0x03) + 24-bit address
    *data_reg = 0x03;
    *data_reg = (offset >> 16) & 0xFF;
    *data_reg = (offset >> 8) & 0xFF;
    *data_reg = offset & 0xFF;
    
    // Read data
    for (size_t i = 0; i < len; i++) {
        // Wait for data ready
        while ((*status_reg & 0x1) == 0);
        buf[i] = *data_reg & 0xFF;
    }
}

uint32_t calculate_checksum(uint8_t *data, size_t len) {
    uint32_t sum = 0;
    for (size_t i = 0; i < len; i++) {
        sum += data[i];
    }
    return sum;
}

void fsbl_main(void) {
    uint8_t *ssbl_addr = (uint8_t *)SSBL_LOAD_ADDR;
    
    // 1. Initialize SPI controller
    spi_init();
    
    // 2. Load SSBL from SPI flash
    spi_read(SSBL_FLASH_OFFSET, ssbl_addr, SSBL_SIZE);
    
    // 3. Verify SSBL (simple checksum)
    uint32_t *checksum_ptr = (uint32_t *)(ssbl_addr + SSBL_SIZE - 4);
    uint32_t expected_checksum = *checksum_ptr;
    uint32_t actual_checksum = calculate_checksum(ssbl_addr, SSBL_SIZE - 4);
    
    if (actual_checksum != expected_checksum) {
        // Checksum failed - hang
        while (1);
    }
    
    // 4. Jump to SSBL
    void (*ssbl_entry)(void) = (void (*)(void))SSBL_LOAD_ADDR;
    ssbl_entry();
    
    // Should never reach here
    while (1);
}
```

---

## C.4 Minimal SSBL: Second-Stage Bootloader

**Purpose**: Load kernel and device tree, set up boot environment.

**Features**:

- Parse device tree
- Load kernel image
- Set up boot arguments
- Jump to kernel in S-mode

### SSBL Main Function

```c
// ssbl_main.c - Minimal second-stage bootloader

#include <stdint.h>

#define KERNEL_LOAD_ADDR  0x80400000  // Load kernel at 4MB offset
#define DTB_LOAD_ADDR     0x82000000  // Load device tree at 32MB
#define KERNEL_SIZE       (8 * 1024 * 1024)  // 8 MB
#define DTB_SIZE          (64 * 1024)  // 64 KB

// Boot arguments for kernel
struct boot_args {
    uint64_t hartid;
    uint64_t dtb_addr;
};

void uart_putc(char c) {
    volatile uint32_t *uart_tx = (uint32_t *)0x10010000;
    *uart_tx = c;
}

void uart_puts(const char *s) {
    while (*s) {
        uart_putc(*s++);
    }
}

void load_kernel_and_dtb(void) {
    // In real bootloader, this would load from storage
    // For this example, assume kernel and DTB are already in memory
    uart_puts("Loading kernel...\n");
    // ... load kernel to KERNEL_LOAD_ADDR ...

    uart_puts("Loading device tree...\n");
    // ... load DTB to DTB_LOAD_ADDR ...
}

void jump_to_kernel(uint64_t hartid, uint64_t dtb_addr, uint64_t kernel_addr) {
    // Set up registers for kernel entry
    // a0 = hartid
    // a1 = dtb_addr

    __asm__ volatile (
        "mv a0, %0\n"
        "mv a1, %1\n"
        "jr %2\n"
        :
        : "r"(hartid), "r"(dtb_addr), "r"(kernel_addr)
        : "a0", "a1"
    );
}

void ssbl_main(void) {
    uint64_t hartid;

    // Read hart ID
    __asm__ volatile ("csrr %0, mhartid" : "=r"(hartid));

    uart_puts("SSBL: Second-Stage Bootloader\n");

    // 1. Load kernel and device tree
    load_kernel_and_dtb();

    // 2. Set up boot arguments
    uart_puts("Booting kernel...\n");

    // 3. Jump to kernel (in S-mode)
    // Note: In real bootloader, would delegate to S-mode first
    jump_to_kernel(hartid, DTB_LOAD_ADDR, KERNEL_LOAD_ADDR);

    // Should never reach here
    while (1);
}
```

---

## C.5 Linker Script

**Purpose**: Define memory layout for bootloader.

```ld
/* bootloader.ld - Linker script for RISC-V bootloader */

OUTPUT_ARCH("riscv")
ENTRY(_start)

MEMORY
{
    ROM   (rx)  : ORIGIN = 0x00001000, LENGTH = 64K
    SRAM  (rwx) : ORIGIN = 0x08000000, LENGTH = 16K
    DRAM  (rwx) : ORIGIN = 0x80000000, LENGTH = 8G
}

SECTIONS
{
    /* Code section in ROM */
    .text : {
        *(.text.init)
        *(.text*)
    } > ROM

    /* Read-only data in ROM */
    .rodata : {
        *(.rodata*)
    } > ROM

    /* Data section in SRAM */
    .data : {
        _data_start = .;
        *(.data*)
        _data_end = .;
    } > SRAM AT> ROM

    /* BSS section in SRAM */
    .bss : {
        _bss_start = .;
        *(.bss*)
        *(COMMON)
        _bss_end = .;
    } > SRAM

    /* Stack in SRAM */
    .stack : {
        . = ALIGN(16);
        . += 8K;
        _stack_top = .;
    } > SRAM
}
```

---

## C.6 Makefile

```makefile
# Makefile for RISC-V bootloader

CROSS_COMPILE = riscv64-unknown-elf-
CC = $(CROSS_COMPILE)gcc
AS = $(CROSS_COMPILE)as
LD = $(CROSS_COMPILE)ld
OBJCOPY = $(CROSS_COMPILE)objcopy

CFLAGS = -march=rv64imac -mabi=lp64 -mcmodel=medany \
         -O2 -Wall -Wextra -nostdlib -nostartfiles \
         -fno-builtin -fno-common

LDFLAGS = -T bootloader.ld -nostdlib

ZSBL_OBJS = zsbl_start.o zsbl_main.o
FSBL_OBJS = fsbl_start.o fsbl_main.o
SSBL_OBJS = ssbl_start.o ssbl_main.o

all: zsbl.bin fsbl.bin ssbl.bin

zsbl.elf: $(ZSBL_OBJS)
 $(LD) $(LDFLAGS) -o $@ $^

fsbl.elf: $(FSBL_OBJS)
 $(LD) $(LDFLAGS) -o $@ $^

ssbl.elf: $(SSBL_OBJS)
 $(LD) $(LDFLAGS) -o $@ $^

%.bin: %.elf
 $(OBJCOPY) -O binary $< $@

%.o: %.S
 $(CC) $(CFLAGS) -c -o $@ $<

%.o: %.c
 $(CC) $(CFLAGS) -c -o $@ $<

clean:
 rm -f *.o *.elf *.bin

.PHONY: all clean
```

---

## C.7 Common Boot Issues and Solutions

### Issue 1: Hart Hangs at Reset

**Symptoms**: System doesn't boot, no output.

**Possible Causes**:

- Reset vector pointing to invalid address
- ROM not mapped correctly
- Clock not initialized

**Debug Steps**:

```assembly
# Add debug output at very first instruction
_start:
    li t0, 0x10010000  # UART base
    li t1, 'A'
    sw t1, 0(t0)       # Write 'A' to UART
    # ... rest of code ...
```

---

### Issue 2: DRAM Initialization Fails

**Symptoms**: System hangs after DRAM init, or data corruption.

**Possible Causes**:

- Incorrect DRAM controller configuration
- Clock frequency mismatch
- Timing parameters wrong

**Debug Steps**:

```c
void dram_test(void) {
    volatile uint32_t *test_addr = (uint32_t *)0x80000000;

    // Write test pattern
    *test_addr = 0xDEADBEEF;

    // Read back
    if (*test_addr != 0xDEADBEEF) {
        uart_puts("DRAM test failed!\n");
        while (1);
    }
}
```

---

### Issue 3: Bootloader Doesn't Load

**Symptoms**: ZSBL runs but FSBL doesn't start.

**Possible Causes**:

- SPI flash not initialized
- Wrong flash offset
- Corrupted image

**Debug Steps**:

```c
void fsbl_main(void) {
    uart_puts("FSBL starting...\n");

    // Verify first few bytes of SSBL
    uint8_t *ssbl = (uint8_t *)SSBL_LOAD_ADDR;
    uart_puts("First bytes: ");
    for (int i = 0; i < 16; i++) {
        uart_puthex(ssbl[i]);
        uart_putc(' ');
    }
    uart_putc('\n');
}
```

---

## C.8 References

- **U-Boot Documentation**: https://u-boot.readthedocs.io/
- **OpenSBI Documentation**: https://github.com/riscv-software-src/opensbi
- **RISC-V Boot Flow**: RISC-V Platform Specification
- **Device Tree Specification**: https://devicetree.org/

---
