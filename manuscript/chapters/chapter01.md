# Chapter 1. What Is RISC-V?

**Part I — Introduction**

---

RISC-V represents a fundamental shift in how we think about processor architectures. Unlike proprietary instruction sets that require licenses and royalties, RISC-V is completely open and free. Unlike monolithic architectures that bundle everything together, RISC-V is modular, allowing implementations from tiny microcontrollers to high-performance servers. Unlike architectures burdened by decades of legacy decisions, RISC-V was designed from scratch in 2010, learning from 30 years of RISC evolution.

This chapter introduces RISC-V by exploring its historical context, design philosophy, and place in the architecture landscape. We'll trace the RISC revolution from the 1980s through RISC-V's creation at UC Berkeley. We'll examine why an open ISA matters and how RISC-V's modular design enables unprecedented flexibility. We'll compare RISC-V with ARM and MIPS to understand its competitive position. By the end, you'll understand not just what RISC-V is, but why it matters and why it's rapidly gaining adoption across the industry.

---

## 1.1 The RISC Revolution

**Origins of RISC**

The story of RISC-V begins not in 2010, but in the early 1980s, when computer architects began questioning the prevailing wisdom of complex instruction sets. Two landmark projects emerged almost simultaneously: the Berkeley RISC project led by David Patterson and Carlo Séquin, and the Stanford MIPS project led by John Hennessy. Both teams arrived at a radical conclusion: simpler is better.

The Berkeley RISC project, starting in 1980, challenged the conventional wisdom that complex instructions were necessary for high performance. Patterson's team demonstrated that a processor with a small, carefully chosen set of simple instructions could outperform contemporary CISC (Complex Instruction Set Computer) processors. The key insight was that compilers, not hardware, should handle complexity.

Meanwhile, at Stanford, John Hennessy's MIPS (Microprocessor without Interlocked Pipeline Stages) project pursued similar goals with a slightly different approach. The MIPS design emphasized pipeline efficiency and compiler optimization, creating an architecture that would eventually power Silicon Graphics workstations and countless embedded systems.

IBM's 801 project, though less publicized, also contributed crucial ideas to the RISC philosophy. Started in the late 1970s, it demonstrated that a load-store architecture with a large register file could achieve excellent performance.

**RISC Design Philosophy**

At the heart of RISC is a deceptively simple idea: make the common case fast. RISC architectures achieve this through several key principles:

*Load-Store Architecture*: Only load and store instructions access memory. All computation happens in registers. This clean separation simplifies pipeline design and enables higher clock frequencies.

*Fixed-Length Instructions*: All instructions are the same size (typically 32 bits). This allows the processor to fetch and decode instructions in parallel, simplifying the instruction fetch stage and enabling efficient pipelining.

*Simple Addressing Modes*: RISC architectures typically support only a few addressing modes, often just base+offset for memory access. Complex address calculations are performed using explicit arithmetic instructions.

*Large Register File*: With 32 or more general-purpose registers, RISC processors can keep frequently used data in fast registers rather than slow memory. This reduces memory traffic and improves performance.

**RISC vs CISC**

The RISC vs CISC debate dominated computer architecture discussions throughout the 1980s and 1990s. CISC architectures like the Intel x86 and Motorola 68000 featured hundreds of complex instructions, variable-length instruction encoding, and numerous addressing modes. The philosophy was to provide rich instruction sets that closely matched high-level language constructs.

RISC took the opposite approach. By keeping instructions simple and uniform, RISC processors could achieve higher clock frequencies and more efficient pipelining. The burden of generating efficient code shifted from hardware to compilers, but this proved to be a winning strategy as compiler technology matured.

Performance comparisons initially favored RISC. A RISC processor might execute more instructions to accomplish the same task, but it could execute each instruction faster and with better pipeline efficiency. The result was often superior overall performance, especially for compute-intensive workloads.

**Historical Impact**

The RISC revolution spawned several influential architectures that shaped the computing landscape:

*MIPS*: Commercialized by MIPS Computer Systems in 1985, the MIPS architecture powered Silicon Graphics workstations and became ubiquitous in embedded systems. Its clean design made it a favorite for teaching computer architecture. Even today, MIPS processors are found in routers, set-top boxes, and other embedded devices.

*SPARC*: Sun Microsystems' Scalable Processor Architecture (1987) dominated the workstation and server market in the 1990s. SPARC's register windows and clean architecture made it popular for Unix systems and scientific computing.

*ARM*: Perhaps the most successful RISC architecture, ARM (Acorn RISC Machine, later Advanced RISC Machine) started in 1985 as a processor for the Acorn Archimedes computer. Its focus on power efficiency made it the dominant architecture for mobile devices. Today, ARM processors power virtually every smartphone and tablet.

*PowerPC*: The alliance between Apple, IBM, and Motorola produced PowerPC in 1991, combining ideas from IBM's POWER architecture with RISC principles. PowerPC powered Apple Macintosh computers from 1994 to 2006 and remains important in embedded and high-performance computing.

These architectures proved that RISC principles worked in practice. They demonstrated that simple, regular instruction sets could deliver excellent performance while simplifying processor design. This legacy directly influenced RISC-V's design philosophy.

**Figure 1.1: RISC Architecture Evolution Timeline**

**RISC Architecture Evolution Timeline (1980-2010)**

| Year | Architecture | Significance |
|------|--------------|--------------|
| 1980 | Berkeley RISC-I | First RISC processor, demonstrated RISC principles |
| 1980 | IBM 801 | Early RISC design, influenced later architectures |
| 1981 | Berkeley RISC-II | Refined RISC design, register windows |
| 1983 | Stanford MIPS | Emphasized pipeline efficiency |
| 1985 | MIPS R2000 | First commercial RISC processor |
| 1985 | ARM1 (Acorn) | Low-power RISC for personal computers |
| 1987 | Sun SPARC | Workstation and server market |
| 1991 | PowerPC | Apple/IBM/Motorola alliance |
| 1994 | ARMv4 (ARM7TDMI) | Thumb instruction set, embedded dominance |
| 2001 | ARMv6 (ARM11) | Advanced features, multimedia support |
| 2010 | RISC-V | Open, modular, extensible ISA |

This timeline shows how RISC-V builds on 30 years of RISC architecture evolution, learning from both successes and mistakes of its predecessors.

---

## 1.2 Why RISC-V? The Open ISA Movement

**The Need for Open ISA**

By 2010, the computer architecture landscape had consolidated around a few proprietary instruction set architectures. Intel's x86 dominated PCs and servers. ARM dominated mobile and embedded systems. MIPS and PowerPC served niche markets. All shared one characteristic: they were proprietary, requiring licenses and royalty payments.

This created several problems for researchers, educators, and innovators:

*Licensing Costs*: Even for academic research, obtaining architecture licenses could be expensive and time-consuming. Companies faced significant royalty payments for each chip manufactured.

*Restrictions on Modification*: Proprietary ISAs typically prohibited modifications or extensions without explicit permission. This stifled innovation and made it difficult to explore new architectural ideas.

*Fragmentation*: Each vendor's extensions and modifications created incompatible variants. ARM alone had numerous profiles and extensions, making it challenging to write portable software.

*Long-Term Uncertainty*: Companies building products around a proprietary ISA faced uncertainty about future licensing terms, support, and the architecture's longevity.

The open-source software movement had demonstrated the power of collaborative development and unrestricted access. Why not apply the same principles to hardware instruction sets?

**Birth of RISC-V**

In 2010, a team at UC Berkeley led by Krste Asanović, David Patterson, Yunsup Lee, and Andrew Waterman set out to create a new instruction set architecture for research and education. They needed an ISA that was:

- Free and open, with no licensing fees or restrictions
- Clean and simple, suitable for teaching
- Practical and complete, capable of running real operating systems
- Extensible, allowing custom instructions for specialized applications
- Stable, with a frozen base that would never change

The team initially considered using an existing open ISA but found none that met all their requirements. MIPS was becoming open but carried legacy baggage. OpenRISC existed but lacked industry momentum. SPARC V8 was available but complex.

So they designed RISC-V from scratch, learning from 30 years of RISC architecture evolution. The name "RISC-V" (pronounced "risk-five") represents the fifth generation of RISC architectures developed at Berkeley, following RISC-I through RISC-IV.

The initial RISC-V specification, released in 2011, defined a minimal 32-bit integer instruction set (RV32I) with just 47 instructions. This base ISA was deliberately kept small and frozen, ensuring that software written for RISC-V would run forever.

**RISC-V International**

As RISC-V gained traction, it became clear that a formal organization was needed to manage the specification and ensure consistency. In 2015, the RISC-V Foundation was established as a non-profit organization to standardize and promote the architecture.

In 2020, the foundation relocated to Switzerland and became RISC-V International, reflecting its global nature and ensuring neutrality from any single country's export controls or political considerations.

RISC-V International operates through a collaborative governance model:

- Technical committees develop and ratify specifications
- Members include companies, universities, and individuals
- All specifications are freely available
- Anyone can implement RISC-V without fees or licenses
- Members contribute to development but don't control the ISA

This open governance ensures that RISC-V remains truly open and vendor-neutral, unlike proprietary ISAs controlled by single companies.

**Industry Ecosystem**

What started as an academic project has grown into a thriving industry ecosystem. By 2025, RISC-V International has over 3,000 members from more than 70 countries, including major technology companies, startups, universities, and government organizations.

Hardware implementations range from tiny microcontrollers to high-performance application processors:

- SiFive produces commercial RISC-V cores for embedded and application processors
- Western Digital has shipped billions of RISC-V cores in storage controllers
- NVIDIA uses RISC-V for GPU microcontrollers
- Alibaba's T-Head develops high-performance RISC-V processors
- Numerous startups are building RISC-V-based products

The software ecosystem has matured rapidly. The GNU toolchain (GCC, binutils) and LLVM support RISC-V. Major operating systems including Linux, FreeBSD, and real-time operating systems run on RISC-V. Language runtimes for Java, Python, JavaScript, and others have been ported.

Academic adoption has been particularly strong. RISC-V's clean design and open nature make it ideal for teaching computer architecture. Universities worldwide use RISC-V in courses, and researchers use it to explore new architectural ideas without licensing barriers.

**Comparison with Proprietary ISAs**

The contrast between RISC-V's open model and proprietary ISAs is stark:

*ARM Licensing*: ARM Holdings licenses its architecture and core designs. Licensees pay upfront fees and per-chip royalties. Architecture licenses (allowing custom core design) are expensive and restricted. This model has been profitable for ARM but creates barriers for innovation and education.

*x86 Duopoly*: Intel and AMD control the x86 architecture through patents and cross-licensing agreements. No other company can legally implement x86 processors. This duopoly limits competition and innovation in the PC and server markets.

*RISC-V Open Model*: Anyone can implement RISC-V without fees, licenses, or royalties. The specification is freely available. Custom extensions are permitted. This openness enables innovation, reduces costs, and eliminates vendor lock-in.

The open model doesn't mean RISC-V is "free" in the sense of zero cost. Designing and manufacturing processors still requires significant investment. But it removes the artificial barriers of licensing fees and restrictions, allowing competition based on implementation quality rather than ISA access.

---

## 1.3 RISC-V Design Philosophy

**Simplicity**

RISC-V embraces simplicity as a core principle. The base integer instruction set (RV32I) contains just 47 instructions—enough to run a complete operating system and applications, but small enough to understand completely in a few hours.

This simplicity manifests in several ways:

*Orthogonal Instruction Set*: Instructions don't have special cases or exceptions. Load and store instructions work the same way regardless of data type. Arithmetic instructions operate uniformly on registers.

*Regular Encoding*: Instruction formats are consistent and predictable. The opcode is always in the same position. Source and destination registers occupy fixed fields. This regularity simplifies decoding and enables efficient implementation.

*No Implicit Operations*: RISC-V instructions do exactly what they say, nothing more. There are no hidden side effects, implicit register updates, or condition code modifications (except for explicit compare instructions).

The simplicity extends to the privilege architecture. RISC-V defines three privilege levels (Machine, Supervisor, User) with clean separation of responsibilities. There are no complex security states or trust zones in the base specification—those can be added as extensions if needed.

**Modularity**

Perhaps RISC-V's most distinctive feature is its modular design. Rather than defining one monolithic ISA, RISC-V separates functionality into a small base ISA plus optional standard extensions.

The base ISA (RV32I, RV64I, or RV128I) is frozen and will never change. It provides:

- Integer arithmetic and logical operations
- Load and store instructions
- Control flow (branches and jumps)
- System instructions (environment calls, fences)

Standard extensions add functionality:

- **M**: Integer multiplication and division
- **A**: Atomic instructions for synchronization
- **F**: Single-precision floating-point
- **D**: Double-precision floating-point
- **C**: Compressed 16-bit instructions for code density
- **V**: Vector operations for data parallelism
- **B**: Bit manipulation

A processor implements only the extensions it needs. An embedded microcontroller might implement just RV32I. A Linux-capable application processor would implement RV64IMAC (often abbreviated as RV64GC, where G = IMAFD). A high-performance processor might add V for vector processing.

This modularity provides several benefits:

- Implementations can be tailored to specific applications
- New extensions can be added without affecting existing code
- The ISA can evolve without breaking compatibility
- Educational and research projects can start simple and add complexity as needed

**Figure 1.2: RISC-V Modular Architecture**

**RISC-V ISA Modular Structure**

| Category | Component | Description | Status |
|----------|-----------|-------------|--------|
| **Base ISA** | RV32I | 32-bit integer base | FROZEN |
| | RV64I | 64-bit integer base | FROZEN |
| | RV128I | 128-bit integer base | FROZEN |
| **Standard Extensions** | M | Multiply/Divide | Standard |
| | A | Atomics | Standard |
| | F | Single-precision floating-point | Standard |
| | D | Double-precision floating-point | Standard |
| | C | Compressed instructions (16-bit) | Standard |
| | V | Vector operations | Standard |
| | B | Bit manipulation | Standard |
| **Custom Extensions** | Custom | Domain-specific instructions | Vendor-defined |

The modular design allows implementations to include only the extensions they need, from minimal embedded systems (RV32I only) to high-performance processors (RV64GCV).

**Extensibility**

Beyond standard extensions, RISC-V explicitly supports custom extensions. The instruction encoding reserves space for custom opcodes, allowing vendors to add specialized instructions without conflicting with standard ones.

Custom extensions enable:

- Domain-specific accelerators (cryptography, AI, signal processing)
- Proprietary features for competitive advantage
- Research into new instruction types
- Rapid prototyping of architectural ideas

The key is that custom extensions don't break compatibility. Software that doesn't use custom instructions runs unchanged. Compilers can generate code that uses custom instructions when available and falls back to standard instructions otherwise.

This extensibility has proven valuable in practice. Companies have added custom instructions for encryption, machine learning, and other specialized tasks. Researchers have explored new architectural concepts without forking the ISA.

**Stability**

RISC-V makes a strong commitment to stability. The base ISA is frozen—it will never change. Software written for RV32I in 2011 will run on RISC-V processors in 2050 and beyond.

Standard extensions follow a rigorous ratification process. Once ratified, an extension is frozen. New versions may add features but must maintain backward compatibility.

This stability provides confidence for long-term investments. Companies can build products knowing the ISA won't change underneath them. Software developers can write code that will run on future processors.

The stability commitment distinguishes RISC-V from some other open ISAs that have undergone incompatible changes. It also contrasts with proprietary ISAs where vendors can deprecate features or change behavior in new versions.

---

## 1.4 RISC-V ISA Overview

**Base Integer ISAs**

RISC-V defines three base integer ISAs, differing only in register width:

*RV32I*: 32-bit registers and addresses. Suitable for embedded systems, microcontrollers, and 32-bit applications. The base RV32I instruction set is frozen and contains 47 instructions.

*RV64I*: 64-bit registers and addresses. Designed for application processors, servers, and systems requiring large address spaces. RV64I extends RV32I with additional instructions for 64-bit operations (like ADDW for 32-bit addition with sign extension).

*RV128I*: 128-bit registers and addresses. Reserved for future systems requiring very large address spaces. The specification is preliminary and not yet frozen.

All three ISAs share the same basic instruction formats and philosophy. Code written for RV32I can often be recompiled for RV64I with minimal changes. The transition from 32-bit to 64-bit is cleaner than in some other architectures.

There's also RV32E, a reduced variant with only 16 registers instead of 32, designed for extremely small embedded systems where chip area is critical.

**Standard Extensions**

The standard extensions add functionality to the base ISA:

*M Extension - Multiplication and Division*: Adds integer multiply, divide, and remainder instructions. Essential for most applications but optional for the simplest embedded systems. The M extension adds 8 instructions in RV32 and 13 in RV64.

*A Extension - Atomic Instructions*: Provides atomic memory operations for synchronization in multi-processor systems. Includes load-reserved/store-conditional (LR/SC) for lock-free algorithms and atomic memory operations (AMO) like atomic add, swap, and compare-and-swap.

*F and D Extensions - Floating-Point*: F adds single-precision (32-bit) floating-point, while D adds double-precision (64-bit). These extensions include arithmetic operations, comparisons, conversions, and a separate register file for floating-point values. They follow the IEEE 754 standard.

*C Extension - Compressed Instructions*: Adds 16-bit instruction encodings for common operations, improving code density by 25-30%. The C extension is particularly valuable for embedded systems with limited memory. Compressed instructions can be freely mixed with standard 32-bit instructions.

*V Extension - Vector Processing*: Provides vector operations for data parallelism. Unlike fixed-width SIMD (like ARM NEON or x86 AVX), RISC-V vectors are length-agnostic, allowing the same code to run efficiently on different vector lengths. This is similar to ARM's SVE but with a cleaner design.

*B Extension - Bit Manipulation*: Adds instructions for common bit manipulation operations like count leading zeros, rotate, and bit field extraction. These operations are common in cryptography, compression, and other algorithms.

The combination of base ISA plus extensions is denoted by a string like "RV64IMAFD" or "RV32IMC". The letter "G" is shorthand for IMAFD (general-purpose), so "RV64GC" means RV64IMAFD plus compressed instructions.

**ISA Naming Convention**

RISC-V uses a systematic naming convention to specify which features a processor implements:

- **RV32** or **RV64** or **RV128**: Base integer ISA width
- **I**: Base integer instruction set (always present)
- **M**: Multiplication and division
- **A**: Atomic instructions
- **F**: Single-precision floating-point
- **D**: Double-precision floating-point
- **C**: Compressed instructions
- **V**: Vector extension
- **G**: Shorthand for IMAFD (general-purpose)

Additional letters indicate other extensions. The order of letters follows a standard sequence defined in the specification.

Examples:

- **RV32I**: Minimal 32-bit processor, base ISA only
- **RV32IMC**: 32-bit with multiply/divide and compressed instructions (common for microcontrollers)
- **RV64GC**: 64-bit general-purpose processor with compressed instructions (common for application processors)
- **RV64GCV**: 64-bit general-purpose with compressed and vector extensions

This naming convention makes it immediately clear what capabilities a processor has.

**Figure 1.3: ISA Naming Convention Examples**

**Common ISA String Breakdown**

| ISA String | Components | Meaning |
|------------|------------|---------|
| **RV64GC** | RV64 | 64-bit base integer ISA |
| | G | General-purpose = IMAFD |
| | - I | Base integer instructions |
| | - M | Multiply/Divide |
| | - A | Atomics |
| | - F | Single-precision float |
| | - D | Double-precision float |
| | C | Compressed instructions |
| **RV32IMC** | RV32 | 32-bit base integer ISA |
| | I | Base integer instructions |
| | M | Multiply/Divide |
| | C | Compressed instructions |

Note: "G" is a shorthand for IMAFD, representing a general-purpose processor configuration.

---

## 1.5 RISC-V Profiles

**The Profile Concept**

As RISC-V matured, the flexibility of optional extensions created a challenge: how do software developers know what features they can rely on? A processor implementing just RV64I is very different from one implementing RV64GCV.

RISC-V Profiles solve this problem by defining standard combinations of extensions for specific use cases. A profile specifies:

- Mandatory extensions that must be present
- Optional extensions that may be present
- Specific versions of each extension
- Additional requirements (like privilege modes)

Software targeting a profile can assume all mandatory features are available, simplifying development and ensuring portability.

**RVA22 Profile**

The RVA22 profile (ratified in 2022) targets application processors capable of running rich operating systems like Linux. It comes in two variants:

*RVA22U (Unprivileged)*: Specifies the user-mode ISA. Mandatory extensions include:

- RV64I base ISA
- M, A, F, D, C extensions (i.e., RV64GC)
- Zicsr (CSR instructions)
- Zifencei (instruction fence)
- Various other Zextensions for specific functionality

*RVA22S (Supervisor)*: Adds supervisor-mode requirements for OS support:

- Sv39 virtual memory (39-bit virtual addresses)
- Supervisor mode and required CSRs
- SBI (Supervisor Binary Interface) support
- Additional privilege-related extensions

A processor claiming RVA22S compliance guarantees it can run standard Linux distributions and other Unix-like operating systems.

**RVA23 Profile**

RVA23 (ratified in 2023) builds on RVA22 with additional features:

- Vector extension (V) is mandatory
- Additional bit manipulation instructions
- Hypervisor extension for virtualization
- Enhanced memory ordering features

RVA23 represents the next generation of application processors, with vector processing as a standard feature rather than an option.

**Embedded Profiles**

While RVA profiles target application processors, separate profiles exist for embedded systems:

*Microcontroller Profiles*: Specify minimal feature sets for resource-constrained devices. These might require only RV32IMC with machine mode, omitting supervisor mode and virtual memory.

*Real-Time Profiles*: Add requirements for deterministic interrupt handling and timing, important for real-time operating systems.

The profile system allows RISC-V to serve diverse markets while maintaining clear compatibility boundaries. Software developers can target a profile rather than trying to support every possible combination of extensions.

---

## 1.6 RISC-V vs ARM vs MIPS

**Historical Context**

To understand RISC-V's place in the architecture landscape, it's helpful to compare it with its RISC predecessors:

*MIPS (1985)*: The pioneer of commercial RISC, MIPS established many principles that RISC-V follows. Its clean design made it popular for education and embedded systems. However, MIPS fragmented into multiple incompatible variants (MIPS I through MIPS V, plus MIPS32/MIPS64), and its proprietary nature limited adoption. In 2019, MIPS became open-source, but by then RISC-V had captured the momentum.

*ARM (1985)*: Starting as a simple RISC design, ARM evolved into a complex architecture with numerous extensions and profiles. Its focus on power efficiency made it dominant in mobile devices. However, ARM remains proprietary, requiring licenses and royalties. The architecture has accumulated considerable complexity over 35+ years of evolution.

*RISC-V (2010)*: Learning from both predecessors, RISC-V combines MIPS's clean design philosophy with ARM's practical focus on real-world applications, while adding the crucial element of openness. It avoids the fragmentation that plagued MIPS and the complexity that accumulated in ARM.

**ISA Complexity Comparison**

*Instruction Count*: RISC-V's base RV32I has 47 instructions. MIPS32 has about 60 base instructions. ARM's instruction count is harder to pin down due to numerous variants, but ARMv8-A has hundreds of instructions when including all extensions.

*Encoding Formats*: RISC-V uses 6 basic instruction formats, all 32 bits (plus 16-bit compressed formats). MIPS uses 3 formats, all 32 bits. ARM uses variable-length encoding in Thumb mode and fixed 32-bit in ARM mode, with complex encoding rules.

*Addressing Modes*: RISC-V supports base+offset for memory access, with offsets computed explicitly for complex addressing. MIPS is similar. ARM supports more complex addressing modes including auto-increment and scaled indexing.

The trend is clear: RISC-V is the simplest, MIPS is moderately complex, and ARM is the most complex. This simplicity makes RISC-V easier to implement, verify, and teach.

**Licensing and Ecosystem**

*RISC-V*: Completely open and free. No licenses required, no royalties, no restrictions. Anyone can implement, modify, or extend RISC-V. The specification is publicly available.

*ARM*: Proprietary and licensed. Architecture licenses cost millions of dollars. Per-chip royalties apply. Modifications require permission. However, ARM offers extensive IP, tools, and ecosystem support.

*MIPS*: Historically proprietary, became open-source in 2019 under Wave Computing. However, the transition was rocky, and MIPS lacks the momentum and ecosystem of RISC-V.

The licensing difference is fundamental. RISC-V enables innovation and competition that proprietary ISAs cannot match.

**Use Cases and Adoption**

*Embedded Systems*: RISC-V is rapidly gaining share in microcontrollers and embedded processors. Its modularity allows implementations tailored to specific needs. Western Digital has shipped billions of RISC-V cores in storage controllers.

*Application Processors*: ARM dominates mobile devices, but RISC-V is emerging in this space. SiFive and others are developing high-performance RISC-V cores. The open nature appeals to companies wanting to avoid ARM licensing.

*High-Performance Computing*: RISC-V is being explored for HPC accelerators and specialized processors. The vector extension makes it competitive for data-parallel workloads.

*Education and Research*: RISC-V has become the architecture of choice for teaching and research, displacing MIPS in many universities.

**Future Outlook**

RISC-V's trajectory is clear: rapid growth driven by openness, simplicity, and industry support. It won't displace ARM in smartphones overnight, but it's becoming the default choice for new designs where licensing costs and flexibility matter.

The comparison with MIPS is particularly instructive. MIPS had technical merit but remained proprietary too long. By the time it opened, RISC-V had captured the open ISA mindshare. ARM's technical excellence and ecosystem are formidable, but its proprietary nature creates opportunities for RISC-V.

RISC-V represents not just a new ISA, but a new model for processor architecture: open, collaborative, and free from vendor lock-in. This model is proving compelling for the next generation of computing.

---

## Summary

RISC-V builds on 30 years of RISC architecture evolution, learning from the successes and mistakes of MIPS, SPARC, ARM, and PowerPC. The RISC revolution of the 1980s demonstrated that simple, regular instruction sets could outperform complex ones, establishing principles that RISC-V follows today: load-store architecture, fixed-length instructions, simple addressing modes, and large register files.

The open ISA movement addresses fundamental problems with proprietary architectures: licensing costs, restrictions on modification, fragmentation, and vendor lock-in. RISC-V provides a completely open and free instruction set architecture, governed by RISC-V International through a collaborative model that ensures vendor neutrality. Anyone can implement, modify, or extend RISC-V without fees or licenses.

RISC-V's design philosophy emphasizes **simplicity** (just 47 instructions in the base ISA), **modularity** (optional extensions for specific needs), **extensibility** (custom instructions without breaking compatibility), and **stability** (frozen base ISA that will never change). This modular approach allows implementations tailored to specific applications, from RV32I-only microcontrollers to RV64GCV high-performance processors.

The standard extensions add functionality as needed: M for multiplication and division, A for atomic operations, F and D for floating-point, C for compressed instructions, V for vector processing, and B for bit manipulation. Profiles like RVA22 and RVA23 define standard combinations of extensions for specific use cases, ensuring software portability while preserving flexibility.

Compared to ARM and MIPS, RISC-V is simpler (fewer instructions, cleaner encoding), more modern (no legacy baggage), and completely open (no licensing fees or restrictions). While ARM dominates mobile devices and MIPS has declined, RISC-V is rapidly gaining adoption in embedded systems, emerging in application processors, and becoming the architecture of choice for education and research.

RISC-V represents not just a new ISA, but a new model for processor architecture: open, collaborative, and free from vendor lock-in. This openness, combined with technical excellence and industry support, positions RISC-V as the architecture for the next generation of computing.
