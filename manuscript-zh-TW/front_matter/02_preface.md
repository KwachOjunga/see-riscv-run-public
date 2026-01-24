# 序言

## 為什麼寫這本書？

RISC-V 代表了電腦架構的根本性轉變。與主導產業數十年的專有指令集架構（ISA）不同，RISC-V 是開放的、模組化的，專為現代時代設計。它不僅僅是另一個 ISA——它是一種思考處理器設計的新方式，從嵌入式微控制器到高效能超級電腦都能實現創新。

當我開始寫這本書時，我想創造一些不存在的東西：一本全面、系統性的 RISC-V 指南，結合官方規格的深度與優秀教科書的清晰度。我受到 Dominic Sweetman 經典著作《See MIPS Run》的啟發，該書以嚴謹且易懂的方式精彩地解釋了 MIPS 架構。本書旨在為 RISC-V 做同樣的事情，同時處理使 RISC-V 與眾不同的獨特特性和現代功能。

## 誰應該閱讀這本書？

這本書是為任何想深入了解 RISC-V 的人而寫的：

**系統軟體開發者**：如果你正在為 RISC-V 編寫作業系統、bootloader、韌體或低階驅動程式，這本書提供你所需的架構知識。你將學習 exception 如何運作、virtual memory 如何實作、如何使用 SBI 呼叫，以及如何在 RISC-V 的弱記憶體模型下編寫正確的並行程式碼。

**硬體工程師**：如果你正在設計 RISC-V 處理器或 SoC，這本書從實作角度解釋 ISA。你將了解 pipeline hazard、memory ordering 要求、interrupt controller 整合，以及不同 microarchitecture 選擇之間的權衡。

**電腦架構學生**：如果你正在學習電腦架構，RISC-V 是一個優秀的教學工具。這本書提供現代 ISA 的完整圖像，從指令編碼到系統級功能，並與 ARM 和 MIPS 進行比較以建立你的架構直覺。

**從 ARM 或 MIPS 轉型的工程師**：如果你有其他架構的經驗並正在轉向 RISC-V，這本書突出了相似之處和差異。你將找到指令集、exception 模型、memory model 和 calling convention 的詳細比較，以加速你的學習。

## 這本書有什麼不同？

**全面涵蓋**：這本書涵蓋整個 RISC-V 生態系統——不僅是基本 ISA，還包括 extension（M、A、F、D、C、V、B、H）、privileged architecture（M/S/U 模式）、系統軟體介面（SBI、ABI）、platform specification（PLIC、CLIC），以及真實世界的實作考量。

**系統性組織**：本書分為 10 個部分，從基礎到進階主題逐步建構。每章都是獨立的，但與更廣泛的敘事相連，使其既適合從頭到尾閱讀，也適合作為參考。

**實務導向**：每個概念都用程式碼範例、圖表和真實世界用例來說明。你將看到如何實作 spinlock、如何處理 page fault、如何配置 PMP，以及如何除錯 memory ordering 問題——不僅僅是抽象理論。

**比較分析**：在整本書中，我將 RISC-V 與 ARM 和 MIPS 進行比較。這些比較幫助你理解 RISC-V 的設計哲學，並在移植程式碼或設計系統時做出明智的決定。

**現代視角**：RISC-V 是一個現代 ISA，設計時吸取了數十年處理器演進的經驗教訓。這本書強調現代功能，如弱記憶體排序、模組化 extension 和形式規格，這些使 RISC-V 與較舊的架構區分開來。

## 如何使用這本書

**從頭到尾閱讀**：章節設計為按順序閱讀，從基本概念建構到進階主題。從 Part I（RISC-V 概覽）開始，一直到 Part X（與其他架構的比較）。

**作為參考**：每章都是獨立的，有清晰的章節標題和摘要。附錄提供 CSR、extension、SBI 呼叫和指令比較的快速參考表。使用目錄和索引來找到特定主題。

**動手學習**：本書包含大量 assembly 和 C 的程式碼範例。我鼓勵你在 RISC-V 模擬器（QEMU、Spike）或真實硬體上執行這些範例。實驗程式碼將加深你的理解。

**用於教學**：這本書適合電腦架構或系統程式設計的大學或研究所課程。每章都包含學習目標和摘要，可以指導課程結構。

## 這本書包含什麼？

這本書包含 17 個章節，分為 10 個部分，加上 6 個完整的附錄：

**Part I: RISC-V 概覽**介紹 RISC-V ISA、其歷史、設計哲學和生態系統。

**Part II: RISC-V 執行模型**涵蓋基本程式設計模型——暫存器、資料類型、指令格式，以及你的第一個 RISC-V 程式。

**Part III: 整數運算**探討算術、邏輯和位移操作。

**Part IV: 控制流程**涵蓋分支、跳躍和函式呼叫。

**Part V: 記憶體存取**解釋 load/store 指令和定址模式。

**Part VI: 函式呼叫**詳述 calling convention 和堆疊管理。

**Part VII: 浮點運算**涵蓋 F 和 D extension。

**Part VIII: 向量操作**探討用於 SIMD 的 V extension。

**Part IX: 進階功能**涵蓋壓縮指令、位元操作和原子操作。

**Part X: 系統程式設計**解釋 privileged architecture、virtual memory、interrupt/exception、效能最佳化、除錯和真實世界應用。

**附錄**提供 CSR、extension、bootloader、SBI、RISC-V vs ARM 比較和 memory model 的快速參考。

**總計**：約 160,000+ 字，約 550 頁

**v0p11 增強功能**：每章現在都包含學習目標（🎯）、角色對話情境引入（💡）、Hands-on Labs（🛠️）完整 assembly 程式碼和 QEMU/GDB 觀察步驟，以及常見陷阱（⚠️）標註常見錯誤和解決方案。

## 致謝

沒有 RISC-V 社群對開放規格和透明開發的承諾，這本書是不可能完成的。我感謝 RISC-V International 和所有為 RISC-V 規格做出貢獻的工程師。

我也要感謝啟發這項工作的經典架構書籍的作者，特別是 Dominic Sweetman 的《See MIPS Run》和設定技術文件標準的 ARM 架構參考手冊。

最後，感謝早期讀者和審閱者對草稿章節提供的反饋。你們的見解使這本書更好。

## 反饋與勘誤

這是一本持續更新的書。RISC-V 持續演進，我致力於保持這本書的時效性。如果你發現錯誤、有建議，或想分享你如何使用這本書，請聯繫：

**Email**: djiang.tw@gmail.com
**GitHub**: https://github.com/djiangtw
**勘誤**: 待公布

**Danny Jiang**
2026 年 1 月
