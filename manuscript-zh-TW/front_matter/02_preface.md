# Preface

## Why This Book?

RISC-V represents a fundamental shift in computer architecture. Unlike proprietary instruction set architectures (ISAs) that dominated the industry for decades, RISC-V is open, modular, and designed for the modern era. It's not just another ISA—it's a new way of thinking about processor design, enabling innovation from embedded microcontrollers to high-performance supercomputers.

When I set out to write this book, I wanted to create something that didn't exist: a comprehensive, systematic guide to RISC-V that combines the depth of official specifications with the clarity of a well-written textbook. I was inspired by Dominic Sweetman's classic "See MIPS Run", which masterfully explained MIPS architecture in a way that was both rigorous and accessible. This book aims to do the same for RISC-V, while addressing the unique characteristics and modern features that make RISC-V special.

## Who Should Read This Book?

This book is written for anyone who wants to understand RISC-V deeply:

**System Software Developers**: If you're writing operating systems, bootloaders, firmware, or low-level drivers for RISC-V, this book provides the architectural knowledge you need. You'll learn how exceptions work, how virtual memory is implemented, how to use SBI calls, and how to write correct concurrent code under RISC-V's weak memory model.

**Hardware Engineers**: If you're designing RISC-V processors or SoCs, this book explains the ISA from an implementation perspective. You'll understand pipeline hazards, memory ordering requirements, interrupt controller integration, and the trade-offs between different microarchitectural choices.

**Computer Architecture Students**: If you're learning computer architecture, RISC-V is an excellent teaching vehicle. This book provides a complete picture of a modern ISA, from instruction encoding to system-level features, with comparisons to ARM and MIPS to build your architectural intuition.

**Engineers Transitioning from ARM or MIPS**: If you're experienced with other architectures and moving to RISC-V, this book highlights the similarities and differences. You'll find detailed comparisons of instruction sets, exception models, memory models, and calling conventions to accelerate your learning.

## What Makes This Book Different?

**Comprehensive Coverage**: This book covers the entire RISC-V ecosystem—not just the base ISA, but also extensions (M, A, F, D, C, V, B, H), privileged architecture (M/S/U modes), system software interfaces (SBI, ABI), platform specifications (PLIC, CLIC), and real-world implementation considerations.

**Systematic Organization**: The book is organized into 10 parts that build progressively from fundamentals to advanced topics. Each chapter is self-contained but connects to the broader narrative, making it suitable both for cover-to-cover reading and as a reference.

**Practical Focus**: Every concept is illustrated with code examples, diagrams, and real-world use cases. You'll see how to implement spinlocks, how to handle page faults, how to configure PMPs, and how to debug memory ordering issues—not just abstract theory.

**Comparative Analysis**: Throughout the book, I compare RISC-V with ARM and MIPS. These comparisons help you understand RISC-V's design philosophy and make informed decisions when porting code or designing systems.

**Modern Perspective**: RISC-V is a modern ISA designed with lessons learned from decades of processor evolution. This book emphasizes modern features like weak memory ordering, modular extensions, and formal specifications that distinguish RISC-V from older architectures.

## How to Use This Book

**For Cover-to-Cover Reading**: The chapters are designed to be read in order, building from basic concepts to advanced topics. Start with Part I (RISC-V Overview) and work through to Part X (Comparisons with Other Architectures).

**As a Reference**: Each chapter is self-contained with clear section headings and summaries. The appendices provide quick reference tables for CSRs, extensions, SBI calls, and instruction comparisons. Use the table of contents and index to find specific topics.

**For Hands-On Learning**: The book includes numerous code examples in assembly and C. I encourage you to run these examples on RISC-V simulators (QEMU, Spike) or real hardware. Experimenting with the code will deepen your understanding.

**For Teaching**: This book is suitable for undergraduate or graduate courses in computer architecture or systems programming. Each chapter includes learning objectives and summaries that can guide course structure.

## What's in This Book?

This book contains 17 chapters organized into 10 parts, plus 6 comprehensive appendices:

**Part I: RISC-V Overview** introduces the RISC-V ISA, its history, design philosophy, and ecosystem.

**Part II: The RISC-V Execution Model** covers the fundamental programming model—registers, data types, instruction formats, and your first RISC-V program.

**Part III: Integer Computation** explores arithmetic, logical, and shift operations.

**Part IV: Control Flow** covers branches, jumps, and function calls.

**Part V: Memory Access** explains load/store instructions and addressing modes.

**Part VI: Function Calls** details the calling convention and stack management.

**Part VII: Floating-Point** covers the F and D extensions.

**Part VIII: Vector Operations** explores the V extension for SIMD.

**Part IX: Advanced Features** covers compressed instructions, bit manipulation, and atomic operations.

**Part X: System Programming** explains privileged architecture, virtual memory, interrupts/exceptions, performance optimization, debugging, and real-world applications.

**Appendices** provide quick reference for CSRs, extensions, bootloaders, SBI, RISC-V vs ARM comparison, and memory model.

**Total**: ~146,600 words, ~500 pages

## Acknowledgments

This book would not have been possible without the RISC-V community's commitment to open specifications and transparent development. I'm grateful to RISC-V International and all the engineers who have contributed to the RISC-V specifications.

I also want to thank the authors of the classic architecture books that inspired this work, particularly Dominic Sweetman's "See MIPS Run" and the ARM architecture reference manuals that set the standard for technical documentation.

Finally, thank you to the early readers and reviewers who provided feedback on draft chapters. Your insights have made this book better.

## Feedback and Errata

This is a living book. RISC-V continues to evolve, and I'm committed to keeping this book current. If you find errors, have suggestions, or want to share how you're using the book, please reach out:

**Email**: djiang.tw@gmail.com
**GitHub**: https://github.com/djiangtw
**Errata**: To be announced

**Danny Jiang**
December 2025
