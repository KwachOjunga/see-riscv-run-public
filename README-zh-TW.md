# See RISC-V Run: Fundamentals（繁體中文版）  
  
**RISC-V 架構完整指南**  
  
[![License](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey)](https://creativecommons.org/licenses/by/4.0/)
[![Version](https://img.shields.io/badge/Version-Draft%20v0p11-blue)]()
[![Author](https://img.shields.io/badge/Author-Danny%20Jiang-orange)]()
[![Updated](https://img.shields.io/badge/Updated-Jan%202026-green)]()

**語言**：[English](README.md) | 繁體中文
  
---  
  
## 📖 關於本書

《See RISC-V Run: Fundamentals》是一本全面介紹 RISC-V 架構的指南，受到 Dominic Sweetman 經典著作《See MIPS Run》的啟發。本書不只是乾澀的規格說明，而是透過「小華」與「阿杰」工程師的對話，用「美食廚房」（ISA 模組化）、「博物館的紅色欄杆」（PMP）、「建築物的權限階層」（M/S/U 模式）等生活化比喻，教你*像系統設計師一樣思考*。

每章開頭都有明確的學習目標，搭配 Hands-on Labs 讓你實際執行程式碼——從「Hello World」到實作 PMP 記憶體防火牆、啟用分頁——完整的 assembly 程式碼搭配 QEMU/GDB 觀察步驟。常見陷阱區塊幫助你避開前人踩過的坑，如指標 stride 錯誤、CSR 存取違規、trap handler 崩潰等。

### 你將學到

- **RISC-V ISA 基礎**：基本整數指令、標準擴展和指令編碼
- **Programmer's Model**：暫存器組、特權層級和執行環境
- **記憶體系統**：虛擬記憶體、分頁、記憶體排序和同步
- **例外處理**：Trap、Exception、Interrupt 和委派機制
- **Pipeline 與 Microarchitecture**：經典 pipeline 設計、hazard 和效能最佳化
- **系統軟體**：開機流程、韌體、SBI（Supervisor Binary Interface）和 OS 整合
- **平台整合**：中斷控制器、SoC 設計和系統規格
- **比較分析**：與 ARM 和 MIPS 架構的詳細比較

### 目標讀者

- RISC-V 工程師和開發者
- 計算機架構學生和研究者
- 系統軟體開發者
- 從 ARM/MIPS/x86 轉向 RISC-V 的工程師
  
---  
  
## 📚 書籍結構  
  
**17 個章節**，分為 **10 個部分**：  
  
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
  
**6 個附錄**：CSR Reference、Extension Reference、Bootloader Reference、SBI Reference、RISC-V vs ARM Comparison、Memory Model Reference  
  
**總計**：約 100,000+ 字（約 400 頁）
  
---  
  
## 📥 原始檔案  
  
所有 Markdown 原始檔案都在此 repository 中：  
  
- **英文版**：`manuscript/`  
- **繁體中文版**：`manuscript-zh-TW/`  
  
**目前版本**：Draft v0p11 - 2026 年 1 月  
  
---  
  
## 📄 授權  
  
**版權所有 © 2025 Danny Jiang**  
  
本著作採用 **Creative Commons Attribution 4.0 International License (CC BY 4.0)** 授權。  
  
**您可以自由地：**  
  
- **分享** — 以任何媒介或格式複製及散布本素材  
- **修改** — 重混、轉換本素材，及依本素材建立新素材，且為任何目的，包含商業性質之使用  
  
**惟需遵守下列條件：**  
  
- **姓名標示** — 您必須給予適當表彰、提供指向本授權條款的連結，以及指出（本作品的原始版本）是否已被變更。  
  
**授權條款**：https://creativecommons.org/licenses/by/4.0/  
  
---  
  
## 🌟 特色  
  
- ✅ **開源**：所有內容在 CC BY 4.0 下免費提供  
- ✅ **全面**：從 ISA 基礎到系統設計的完整涵蓋  
- ✅ **實用**：真實世界範例和實作見解  
- ✅ **比較**：與 ARM 和 MIPS 的詳細比較  
- ✅ **最新**：基於最新的 RISC-V 規格  
- ✅ **雙語**：提供英文和繁體中文版本  
  
---  
  
## 🔗 連結  
  
- **作者**：[Danny Jiang](https://github.com/djiangtw)  
- **Email**：djiang.tw@gmail.com  
- **LinkedIn**：[linkedin.com/in/danny-jiang-26359644](https://www.linkedin.com/in/danny-jiang-26359644/)  
- **RISC-V International**：[riscv.org](https://riscv.org)  
  
---  
  
## 📝 引用  
  
如果您在研究或教學中使用本書，請引用：  
  
```  
Danny Jiang. (2025). See RISC-V Run: Fundamentals - RISC-V 架構完整指南.  
採用 CC BY 4.0 授權. https://github.com/djiangtw/see-riscv-run-public  
```  
  
---  
  
## 🙏 致謝  
  
本書得以完成，感謝：  
  
- **RISC-V International** 以及所有 RISC-V 規格的貢獻者  
- **RISC-V 社群**，他們的協作精神和對開放標準的承諾  
- **早期審閱者**，他們提供了寶貴的反饋  
- **家人和朋友**，在寫作過程中給予堅定不移的支持  
  
---  
  
**最後更新**：2026 年 1 月  
  
