# See RISC-V Run: Fundamentals  
  
**A Comprehensive Guide to RISC-V Architecture**  
  
[![License](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey)](https://creativecommons.org/licenses/by/4.0/)
[![Version](https://img.shields.io/badge/Version-Draft%20v0p11-blue)]()
[![Author](https://img.shields.io/badge/Author-Danny%20Jiang-orange)]()
[![Updated](https://img.shields.io/badge/Updated-Jan%202026-green)]()

**Language**: English | [繁體中文](README-zh-TW.md)
  
---  
  
## 📖 About This Book

*See RISC-V Run: Fundamentals* is a comprehensive guide to the RISC-V architecture, inspired by Dominic Sweetman's classic *See MIPS Run*. Rather than presenting dry specifications, this book teaches you *how to think* like a system designer through dialogues between "Junior" and "Senior" engineers, using real-world metaphors like "The Gourmet Kitchen" (ISA modularity), "The Museum's Red Barrier Poles" (PMP), and "The Building's Privilege Hierarchy" (M/S/U modes) to explain complex concepts.

Each chapter begins with clear learning objectives, followed by hands-on labs where you can run the code—from "Hello World" to implementing a PMP memory firewall and enabling paging—with complete assembly code and QEMU/GDB observation steps. Common pitfalls sections help you avoid real bugs like pointer stride errors, CSR access violations, and trap handler crashes.

### What You'll Learn

- **RISC-V ISA Fundamentals**: Base integer instructions, standard extensions, and instruction encoding
- **Programmer's Model**: Register sets, privilege levels, and execution environment
- **Memory System**: Virtual memory, paging, memory ordering, and synchronization
- **Exception Handling**: Traps, exceptions, interrupts, and delegation mechanisms
- **Pipeline & Microarchitecture**: Classic pipeline design, hazards, and performance optimization
- **System Software**: Boot flow, firmware, SBI (Supervisor Binary Interface), and OS integration
- **Platform Integration**: Interrupt controllers, SoC design, and system specifications
- **Comparative Analysis**: Detailed comparisons with ARM and MIPS architectures

### Target Audience

- RISC-V engineers and developers
- Computer architecture students and researchers
- System software developers
- Engineers transitioning from ARM/MIPS/x86 to RISC-V
  
---  
  
## 📚 Book Structure  
  
**17 Chapters** organized into **10 Parts**:  
  
- **Part I** — Introduction  
- **Part II** — Programmer's Model  
- **Part III** — Traps & Exceptions  
- **Part IV** — Memory & Addressing  
- **Part V** — Pipeline & Microarchitecture  
- **Part VI** — Booting & System Software  
- **Part VII** — ISA Extensions  
- **Part VIII** — System Design, Platform Spec & SoC Integration  
- **Part IX** — Performance, Debug & Tools  
- **Part X** — RISC-V vs Other Architectures  
  
**6 Appendices**: CSR Reference, Extension Reference, Bootloader Reference, SBI Reference, RISC-V vs ARM Comparison, Memory Model Reference  
  
**Total**: ~100,000+ words (~400 pages)
  
---  
  
## 📥 Source Files  
  
All Markdown source files are available in this repository:  
  
- **English Version**: `manuscript/`  
- **Traditional Chinese Version**: `manuscript-zh-TW/`  
  
**Current Version**: Draft v0p11 - January 2026  
  
---  
  
## 📄 License  
  
**Copyright © 2025 Danny Jiang**  
  
This work is licensed under the **Creative Commons Attribution 4.0 International License (CC BY 4.0)**.  
  
**You are free to:**  
  
- **Share** — copy and redistribute the material in any medium or format  
- **Adapt** — remix, transform, and build upon the material for any purpose, even commercially  
  
**Under the following terms:**  
  
- **Attribution** — You must give appropriate credit, provide a link to the license, and indicate if changes were made.  
  
**License**: https://creativecommons.org/licenses/by/4.0/  
  
---  
  
## 🌟 Features  
  
- ✅ **Open Source**: All content freely available under CC BY 4.0  
- ✅ **Comprehensive**: Complete coverage from ISA basics to system design  
- ✅ **Practical**: Real-world examples and implementation insights  
- ✅ **Comparative**: Detailed comparisons with ARM and MIPS  
- ✅ **Up-to-date**: Based on latest RISC-V specifications  
- ✅ **Bilingual**: Available in English and Traditional Chinese  
  
---  
  
## 🔗 Links  
  
- **Author**: [Danny Jiang](https://github.com/djiangtw)  
- **Email**: djiang.tw@gmail.com  
- **LinkedIn**: [linkedin.com/in/danny-jiang-26359644](https://www.linkedin.com/in/danny-jiang-26359644/)  
- **RISC-V International**: [riscv.org](https://riscv.org)  
  
---  
  
## 📝 Citation  
  
If you use this book in your research or teaching, please cite:  
  
```  
Danny Jiang. (2025). See RISC-V Run: Fundamentals - A Comprehensive Guide to RISC-V Architecture.  
Licensed under CC BY 4.0. https://github.com/djiangtw/see-riscv-run-public  
```  
  
---  
  
## 🙏 Acknowledgments  
  
This book is made possible by:  
  
- **RISC-V International** and all contributors to the RISC-V specifications  
- **The RISC-V community** for their collaborative spirit and commitment to open standards  
- **Early reviewers** who provided valuable feedback  
- **Family and friends** for their unwavering support  
  
---  
  
**Last Updated**: January 2026  
  
