# Chapter 1. 什麼是 RISC-V？

**Part I — 導論**

---

RISC-V 代表了我們思考處理器架構方式的根本性轉變。不同於需要授權和權利金的專有 instruction set，RISC-V 完全開放且免費。不同於將所有功能綁在一起的單體式架構，RISC-V 採用模組化設計，可以實現從微型 microcontroller 到高效能 server 的各種應用。不同於背負數十年歷史包袱的架構，RISC-V 於 2010 年從零開始設計，汲取了 30 年 RISC 演進的經驗。

本章將透過探討 RISC-V 的歷史背景、設計哲學，以及在架構領域中的定位來介紹 RISC-V。我們將追溯從 1980 年代的 RISC 革命到 UC Berkeley 創建 RISC-V 的歷程。我們將探討為什麼開放的 ISA 如此重要，以及 RISC-V 的模組化設計如何實現前所未有的靈活性。我們將比較 RISC-V 與 ARM 和 MIPS，以了解其競爭地位。讀完本章後，你不僅會理解 RISC-V 是什麼，還會明白它為何重要，以及為何在業界快速獲得採用。

---

## 1.1 RISC 革命

**RISC 的起源**

RISC-V 的故事並非始於 2010 年，而是要追溯到 1980 年代初期，當時計算機架構師開始質疑複雜 instruction set 的主流思維。幾乎同時出現了兩個具有里程碑意義的專案：由 David Patterson 和 Carlo Séquin 領導的 Berkeley RISC 專案，以及由 John Hennessy 領導的 Stanford MIPS 專案。兩個團隊都得出了一個激進的結論：越簡單越好。

Berkeley RISC 專案始於 1980 年，挑戰了複雜 instruction 對於高效能是必要的傳統觀念。Patterson 的團隊證明，具有少量精心選擇的簡單 instruction 的處理器，可以超越當時的 CISC (Complex Instruction Set Computer) 處理器。關鍵洞察是：應該由 compiler 而非硬體來處理複雜性。

與此同時，在 Stanford，John Hennessy 的 MIPS (Microprocessor without Interlocked Pipeline Stages) 專案以略微不同的方法追求類似的目標。MIPS 設計強調 pipeline 效率和 compiler 最佳化，創造出一個最終為 Silicon Graphics workstation 和無數 embedded system 提供動力的架構。

IBM 的 801 專案雖然較少被公開討論，但也為 RISC 哲學貢獻了關鍵想法。始於 1970 年代末期，它證明了具有大型 register file 的 load-store 架構可以達到優異的效能。

**RISC 設計哲學**

RISC 的核心是一個看似簡單的想法：讓常見情況快速執行。RISC 架構透過幾個關鍵原則來實現這一點：

*Load-Store 架構*：只有 load 和 store instruction 可以存取 memory。所有運算都在 register 中進行。這種清晰的分離簡化了 pipeline 設計，並能實現更高的 clock frequency。

*固定長度 Instruction*：所有 instruction 都是相同大小（通常是 32 bits）。這使得處理器可以並行 fetch 和 decode instruction，簡化了 instruction fetch stage，並實現高效的 pipelining。

*簡單的 Addressing Mode*：RISC 架構通常只支援少數幾種 addressing mode，對於 memory access 通常只有 base+offset。複雜的位址計算使用明確的算術 instruction 來執行。

*大型 Register File*：擁有 32 個或更多 general-purpose register，RISC 處理器可以將經常使用的資料保存在快速的 register 中，而不是緩慢的 memory 中。這減少了 memory traffic 並提高了效能。

**RISC vs CISC**

RISC vs CISC 的辯論主導了整個 1980 和 1990 年代的計算機架構討論。CISC 架構如 Intel x86 和 Motorola 68000 具有數百個複雜的 instruction、可變長度的 instruction encoding，以及眾多的 addressing mode。其哲學是提供豐富的 instruction set，緊密匹配高階語言的結構。

RISC 採取了相反的方法。透過保持 instruction 簡單且統一，RISC 處理器可以達到更高的 clock frequency 和更高效的 pipelining。產生高效程式碼的負擔從硬體轉移到 compiler，但隨著 compiler 技術的成熟，這被證明是一個成功的策略。

效能比較最初有利於 RISC。RISC 處理器可能需要執行更多 instruction 來完成相同的任務，但它可以更快地執行每個 instruction，並具有更好的 pipeline 效率。結果往往是整體效能更優，特別是對於計算密集型的 workload。

**歷史影響**

RISC 革命催生了幾個影響計算領域的重要架構：

*MIPS*：1985 年由 MIPS Computer Systems 商業化，MIPS 架構為 Silicon Graphics workstation 提供動力，並在 embedded system 中無處不在。其簡潔的設計使其成為教授計算機架構的首選。即使在今天，MIPS 處理器仍然存在於 router、set-top box 和其他 embedded device 中。

*SPARC*：Sun Microsystems 的 Scalable Processor Architecture (1987) 在 1990 年代主導了 workstation 和 server 市場。SPARC 的 register window 和簡潔架構使其在 Unix 系統和科學計算中很受歡迎。

*ARM*：也許是最成功的 RISC 架構，ARM (Acorn RISC Machine，後來改為 Advanced RISC Machine) 始於 1985 年，作為 Acorn Archimedes 電腦的處理器。其對功耗效率的關注使其成為行動裝置的主導架構。今天，ARM 處理器為幾乎所有 smartphone 和 tablet 提供動力。

*PowerPC*：Apple、IBM 和 Motorola 之間的聯盟在 1991 年產生了 PowerPC，結合了 IBM POWER 架構的想法和 RISC 原則。PowerPC 在 1994 年至 2006 年間為 Apple Macintosh 電腦提供動力，並在 embedded 和高效能計算中仍然很重要。

這些架構證明了 RISC 原則在實踐中是有效的。它們證明了簡單、規則的 instruction set 可以提供優異的效能，同時簡化處理器設計。這一遺產直接影響了 RISC-V 的設計哲學。

**Figure 1.1: RISC 架構演進時間軸**

**RISC 架構演進時間軸 (1980-2010)**

| 年份 | 架構 | 重要性 |
|------|------|--------|
| 1980 | Berkeley RISC-I | 第一個 RISC 處理器，展示 RISC 原則 |
| 1980 | IBM 801 | 早期 RISC 設計，影響後續架構 |
| 1981 | Berkeley RISC-II | 改進的 RISC 設計，register window |
| 1983 | Stanford MIPS | 強調 pipeline 效率 |
| 1985 | MIPS R2000 | 第一個商業化 RISC 處理器 |
| 1985 | ARM1 (Acorn) | 低功耗 RISC，用於個人電腦 |
| 1987 | Sun SPARC | Workstation 和 server 市場 |
| 1991 | PowerPC | Apple/IBM/Motorola 聯盟 |
| 1994 | ARMv4 (ARM7TDMI) | Thumb instruction set，embedded 主導地位 |
| 2001 | ARMv6 (ARM11) | 進階功能，多媒體支援 |
| 2010 | RISC-V | 開放、模組化、可擴展的 ISA |

這個時間軸顯示了 RISC-V 如何建立在 30 年 RISC 架構演進的基礎上，從前輩的成功和錯誤中學習。

---

## 1.2 為什麼選擇 RISC-V？開放 ISA 運動

**對開放 ISA 的需求**

到了 2010 年，計算機架構領域已經整合為少數幾個專有的 instruction set architecture。Intel 的 x86 主導 PC 和 server。ARM 主導行動和 embedded system。MIPS 和 PowerPC 服務於利基市場。它們都有一個共同特徵：都是專有的，需要授權和權利金支付。

這為研究人員、教育工作者和創新者帶來了幾個問題：

*授權成本*：即使是學術研究，獲得架構授權也可能既昂貴又耗時。公司每製造一顆晶片都要支付大量權利金。

*修改限制*：專有 ISA 通常禁止未經明確許可的修改或擴展。這抑制了創新，使探索新的架構想法變得困難。

*碎片化*：每個供應商的擴展和修改都創造了不相容的變體。僅 ARM 就有眾多 profile 和 extension，使得編寫可移植的軟體變得具有挑戰性。

*長期不確定性*：圍繞專有 ISA 建立產品的公司面臨未來授權條款、支援和架構壽命的不確定性。

開源軟體運動已經證明了協作開發和不受限制存取的力量。為什麼不將相同的原則應用於硬體 instruction set？

**RISC-V 的誕生**

2010 年，UC Berkeley 的一個團隊由 Krste Asanović、David Patterson、Yunsup Lee 和 Andrew Waterman 領導，著手創建一個新的 instruction set architecture，用於研究和教育。他們需要一個 ISA：

- 自由且開放，沒有授權費用或限制
- 簡潔且簡單，適合教學
- 實用且完整，能夠執行真實的 operating system
- 可擴展，允許為專門應用添加 custom instruction
- 穩定，具有永不改變的 frozen base

團隊最初考慮使用現有的開放 ISA，但發現沒有一個能滿足所有要求。MIPS 正在變得開放，但帶有歷史包袱。OpenRISC 存在但缺乏業界動力。SPARC V8 可用但過於複雜。

因此，他們從零開始設計 RISC-V，從 30 年的 RISC 架構演進中學習。名稱 "RISC-V"（發音為 "risk-five"）代表 Berkeley 開發的第五代 RISC 架構，繼 RISC-I 到 RISC-IV 之後。

最初的 RISC-V 規範於 2011 年發布，定義了一個最小的 32-bit integer instruction set (RV32I)，只有 47 個 instruction。這個 base ISA 被刻意保持小型且 frozen，確保為 RISC-V 編寫的軟體將永遠可以執行。

**RISC-V International**

隨著 RISC-V 獲得關注，顯然需要一個正式組織來管理規範並確保一致性。2015 年，RISC-V Foundation 成立為非營利組織，以標準化和推廣該架構。

2020 年，該基金會遷至瑞士並成為 RISC-V International，反映其全球性質，並確保不受任何單一國家的出口管制或政治考量影響。

RISC-V International 透過協作治理模式運作：

- Technical committee 開發和批准規範
- 成員包括公司、大學和個人
- 所有規範都免費提供
- 任何人都可以實現 RISC-V，無需費用或授權
- 成員貢獻開發但不控制 ISA

這種開放治理確保 RISC-V 保持真正開放和供應商中立，不同於由單一公司控制的專有 ISA。

**產業生態系統**

從學術專案開始的 RISC-V 已經發展成為一個蓬勃發展的產業生態系統。到 2025 年，RISC-V International 擁有來自 70 多個國家的 3,000 多個成員，包括主要科技公司、新創公司、大學和政府組織。

硬體實現範圍從微型 microcontroller 到高效能 application processor：

- SiFive 為 embedded 和 application processor 生產商業 RISC-V core
- Western Digital 已在 storage controller 中出貨數十億個 RISC-V core
- NVIDIA 在 GPU microcontroller 中使用 RISC-V
- Alibaba 的 T-Head 開發高效能 RISC-V 處理器
- 眾多新創公司正在建立基於 RISC-V 的產品

軟體生態系統也快速成熟。GNU toolchain (GCC, binutils) 和 LLVM 支援 RISC-V。主要 operating system 包括 Linux、FreeBSD 和 real-time operating system 都在 RISC-V 上執行。Java、Python、JavaScript 等語言 runtime 已被移植。

學術界的採用特別強勁。RISC-V 的簡潔設計和開放性質使其成為教授計算機架構的理想選擇。全球大學在課程中使用 RISC-V，研究人員使用它來探索新的架構想法，而無需授權障礙。

**與專有 ISA 的比較**

RISC-V 的開放模式與專有 ISA 之間的對比鮮明：

*ARM 授權*：ARM Holdings 授權其架構和 core 設計。被授權方支付預付費用和每顆晶片的權利金。Architecture license（允許 custom core 設計）昂貴且受限。這種模式對 ARM 來說是有利可圖的，但為創新和教育創造了障礙。

*x86 雙頭壟斷*：Intel 和 AMD 透過專利和交叉授權協議控制 x86 架構。沒有其他公司可以合法實現 x86 處理器。這種雙頭壟斷限制了 PC 和 server 市場的競爭和創新。

*RISC-V 開放模式*：任何人都可以實現 RISC-V，無需費用、授權或權利金。規範免費提供。允許 custom extension。這種開放性促進創新、降低成本，並消除 vendor lock-in。

開放模式並不意味著 RISC-V 是「免費」的，即零成本。設計和製造處理器仍然需要大量投資。但它消除了授權費用和限制的人為障礙，允許基於實現品質而非 ISA 存取的競爭。

---

## 1.3 RISC-V 設計哲學

**簡潔性**

RISC-V 將簡潔性作為核心原則。Base integer instruction set (RV32I) 只包含 47 個 instruction——足以執行完整的 operating system 和 application，但又小到可以在幾個小時內完全理解。

這種簡潔性體現在幾個方面：

*正交的 Instruction Set*：Instruction 沒有特殊情況或例外。Load 和 store instruction 無論資料類型如何都以相同方式工作。算術 instruction 統一地在 register 上操作。

*規則的 Encoding*：Instruction format 一致且可預測。Opcode 總是在相同位置。Source 和 destination register 佔據固定欄位。這種規則性簡化了 decoding，並實現高效的實現。

*沒有隱式操作*：RISC-V instruction 只做它們所說的，不多不少。沒有隱藏的副作用、隱式 register 更新或 condition code 修改（除了明確的 compare instruction）。

簡潔性延伸到 privilege architecture。RISC-V 定義了三個 privilege level (Machine, Supervisor, User)，責任分離清晰。Base specification 中沒有複雜的 security state 或 trust zone——如果需要，可以作為 extension 添加。

**模組化**

也許 RISC-V 最獨特的特徵是其模組化設計。RISC-V 不是定義一個單體式 ISA，而是將功能分離為一個小型 base ISA 加上可選的標準 extension。

Base ISA (RV32I, RV64I, 或 RV128I) 是 frozen 的，永遠不會改變。它提供：

- Integer 算術和邏輯運算
- Load 和 store instruction
- Control flow (branch 和 jump)
- System instruction (environment call, fence)

標準 extension 添加功能：

- **M**: Integer multiplication 和 division
- **A**: Atomic instruction，用於同步
- **F**: Single-precision floating-point
- **D**: Double-precision floating-point
- **C**: Compressed 16-bit instruction，提高 code density
- **V**: Vector operation，用於資料並行
- **B**: Bit manipulation

處理器只實現它需要的 extension。Embedded microcontroller 可能只實現 RV32I。能執行 Linux 的 application processor 會實現 RV64IMAC（通常縮寫為 RV64GC，其中 G = IMAFD）。高效能處理器可能添加 V 用於 vector processing。

這種模組化提供了幾個好處：

- 實現可以針對特定應用進行調整
- 可以添加新 extension 而不影響現有程式碼
- ISA 可以演進而不破壞相容性
- 教育和研究專案可以從簡單開始，根據需要添加複雜性

**Figure 1.2: RISC-V 模組化架構**

**RISC-V ISA 模組化結構**

| 類別 | 組件 | 描述 | 狀態 |
|------|------|------|------|
| **Base ISA** | RV32I | 32-bit integer base | FROZEN |
| | RV64I | 64-bit integer base | FROZEN |
| | RV128I | 128-bit integer base | FROZEN |
| **標準 Extension** | M | Multiply/Divide | Standard |
| | A | Atomic | Standard |
| | F | Single-precision floating-point | Standard |
| | D | Double-precision floating-point | Standard |
| | C | Compressed instruction (16-bit) | Standard |
| | V | Vector operation | Standard |
| | B | Bit manipulation | Standard |
| **Custom Extension** | Custom | Domain-specific instruction | Vendor-defined |

模組化設計允許實現只包含它們需要的 extension，從最小的 embedded system (僅 RV32I) 到高效能處理器 (RV64GCV)。

**可擴展性**

除了標準 extension，RISC-V 明確支援 custom extension。Instruction encoding 為 custom opcode 保留空間，允許供應商添加專門的 instruction，而不會與標準 instruction 衝突。

Custom extension 實現：

- Domain-specific accelerator (cryptography, AI, signal processing)
- 競爭優勢的專有功能
- 新 instruction type 的研究
- 架構想法的快速原型

關鍵是 custom extension 不會破壞相容性。不使用 custom instruction 的軟體可以不變地執行。Compiler 可以在可用時生成使用 custom instruction 的程式碼，否則回退到標準 instruction。

這種可擴展性在實踐中已被證明是有價值的。公司已經為加密、機器學習和其他專門任務添加了 custom instruction。研究人員探索了新的架構概念，而無需 fork ISA。

**穩定性**

RISC-V 對穩定性做出了強有力的承諾。Base ISA 是 frozen 的——它永遠不會改變。2011 年為 RV32I 編寫的軟體將在 2050 年及以後的 RISC-V 處理器上執行。

標準 extension 遵循嚴格的批准流程。一旦批准，extension 就被 frozen。新版本可能添加功能，但必須保持向後相容性。

這種穩定性為長期投資提供了信心。公司可以建立產品，知道 ISA 不會在它們下面改變。軟體開發人員可以編寫將在未來處理器上執行的程式碼。

穩定性承諾將 RISC-V 與一些經歷了不相容變更的其他開放 ISA 區分開來。它也與專有 ISA 形成對比，在專有 ISA 中，供應商可以在新版本中棄用功能或改變行為。

---

## 1.4 RISC-V ISA 概覽

**Base Integer ISA**

RISC-V 定義了三個 base integer ISA，僅在 register 寬度上有所不同：

*RV32I*：32-bit register 和 address。適用於 embedded system、microcontroller 和 32-bit application。Base RV32I instruction set 是 frozen 的，包含 47 個 instruction。

*RV64I*：64-bit register 和 address。設計用於 application processor、server 和需要大 address space 的系統。RV64I 擴展了 RV32I，添加了用於 64-bit 操作的額外 instruction（如 ADDW 用於帶 sign extension 的 32-bit addition）。

*RV128I*：128-bit register 和 address。為需要非常大 address space 的未來系統保留。規範是初步的，尚未 frozen。

所有三個 ISA 共享相同的基本 instruction format 和哲學。為 RV32I 編寫的程式碼通常可以用最少的更改重新編譯為 RV64I。從 32-bit 到 64-bit 的轉換比某些其他架構更簡潔。

還有 RV32E，一個只有 16 個 register 而不是 32 個的簡化變體，設計用於晶片面積至關重要的極小型 embedded system。

**標準 Extension**

標準 extension 為 base ISA 添加功能：

*M Extension - Multiplication 和 Division*：添加 integer multiply、divide 和 remainder instruction。對大多數 application 來說是必需的，但對最簡單的 embedded system 是可選的。M extension 在 RV32 中添加 8 個 instruction，在 RV64 中添加 13 個。

*A Extension - Atomic Instruction*：為 multi-processor system 中的同步提供 atomic memory operation。包括用於 lock-free algorithm 的 load-reserved/store-conditional (LR/SC) 和 atomic memory operation (AMO)，如 atomic add、swap 和 compare-and-swap。

*F 和 D Extension - Floating-Point*：F 添加 single-precision (32-bit) floating-point，而 D 添加 double-precision (64-bit)。這些 extension 包括算術運算、比較、轉換，以及用於 floating-point value 的獨立 register file。它們遵循 IEEE 754 標準。

*C Extension - Compressed Instruction*：為常見操作添加 16-bit instruction encoding，將 code density 提高 25-30%。C extension 對於 memory 有限的 embedded system 特別有價值。Compressed instruction 可以與標準 32-bit instruction 自由混合。

*V Extension - Vector Processing*：提供用於資料並行的 vector operation。與固定寬度的 SIMD（如 ARM NEON 或 x86 AVX）不同，RISC-V vector 是 length-agnostic 的，允許相同的程式碼在不同的 vector length 上高效執行。這類似於 ARM 的 SVE，但設計更簡潔。

*B Extension - Bit Manipulation*：為常見的 bit manipulation operation 添加 instruction，如 count leading zero、rotate 和 bit field extraction。這些操作在 cryptography、compression 和其他 algorithm 中很常見。

Base ISA 加 extension 的組合用字串表示，如 "RV64IMAFD" 或 "RV32IMC"。字母 "G" 是 IMAFD (general-purpose) 的簡寫，所以 "RV64GC" 表示 RV64IMAFD 加 compressed instruction。

**ISA 命名慣例**

RISC-V 使用系統化的命名慣例來指定處理器實現的功能：

- **RV32** 或 **RV64** 或 **RV128**：Base integer ISA 寬度
- **I**：Base integer instruction set（總是存在）
- **M**：Multiplication 和 division
- **A**：Atomic instruction
- **F**：Single-precision floating-point
- **D**：Double-precision floating-point
- **C**：Compressed instruction
- **V**：Vector extension
- **G**：IMAFD (general-purpose) 的簡寫

額外的字母表示其他 extension。字母的順序遵循規範中定義的標準序列。

範例：

- **RV32I**：最小的 32-bit 處理器，僅 base ISA
- **RV32IMC**：32-bit，帶 multiply/divide 和 compressed instruction（microcontroller 常見）
- **RV64GC**：64-bit general-purpose 處理器，帶 compressed instruction（application processor 常見）
- **RV64GCV**：64-bit general-purpose，帶 compressed 和 vector extension

這種命名慣例使處理器具有的能力一目了然。

**Figure 1.3: ISA 命名慣例範例**

**常見 ISA 字串分解**

| ISA 字串 | 組件 | 意義 |
|----------|------|------|
| **RV64GC** | RV64 | 64-bit base integer ISA |
| | G | General-purpose = IMAFD |
| | - I | Base integer instruction |
| | - M | Multiply/Divide |
| | - A | Atomic |
| | - F | Single-precision float |
| | - D | Double-precision float |
| | C | Compressed instruction |
| **RV32IMC** | RV32 | 32-bit base integer ISA |
| | I | Base integer instruction |
| | M | Multiply/Divide |
| | C | Compressed instruction |

注意："G" 是 IMAFD 的簡寫，代表 general-purpose 處理器配置。

---

## 1.5 RISC-V Profile

**Profile 概念**

隨著 RISC-V 的成熟，可選 extension 的靈活性帶來了一個挑戰：軟體開發人員如何知道他們可以依賴哪些功能？只實現 RV64I 的處理器與實現 RV64GCV 的處理器非常不同。

RISC-V Profile 透過為特定用例定義標準的 extension 組合來解決這個問題。Profile 指定：

- 必須存在的強制 extension
- 可能存在的可選 extension
- 每個 extension 的特定版本
- 額外要求（如 privilege mode）

針對 profile 的軟體可以假設所有強制功能都可用，簡化開發並確保可移植性。

**RVA22 Profile**

RVA22 profile（2022 年批准）針對能夠執行 Linux 等豐富 operating system 的 application processor。它有兩個變體：

*RVA22U (Unprivileged)*：指定 user-mode ISA。強制 extension 包括：

- RV64I base ISA
- M, A, F, D, C extension（即 RV64GC）
- Zicsr (CSR instruction)
- Zifencei (instruction fence)
- 各種其他 Z extension，用於特定功能

*RVA22S (Supervisor)*：為 OS 支援添加 supervisor-mode 要求：

- Sv39 virtual memory (39-bit virtual address)
- Supervisor mode 和所需的 CSR
- SBI (Supervisor Binary Interface) 支援
- 額外的 privilege 相關 extension

聲稱符合 RVA22S 的處理器保證可以執行標準 Linux distribution 和其他類 Unix operating system。

**RVA23 Profile**

RVA23（2023 年批准）在 RVA22 的基礎上添加了額外功能：

- Vector extension (V) 是強制的
- 額外的 bit manipulation instruction
- 用於虛擬化的 Hypervisor extension
- 增強的 memory ordering 功能

RVA23 代表下一代 application processor，將 vector processing 作為標準功能而非選項。

**Embedded Profile**

雖然 RVA profile 針對 application processor，但 embedded system 有單獨的 profile：

*Microcontroller Profile*：為資源受限的裝置指定最小功能集。這些可能只需要 RV32IMC 和 machine mode，省略 supervisor mode 和 virtual memory。

*Real-Time Profile*：為 real-time operating system 添加確定性 interrupt handling 和 timing 的要求。

Profile 系統允許 RISC-V 服務於不同的市場，同時保持清晰的相容性邊界。軟體開發人員可以針對 profile，而不是試圖支援每種可能的 extension 組合。

---

## 1.6 RISC-V vs ARM vs MIPS

**歷史背景**

要理解 RISC-V 在架構領域中的地位，將其與 RISC 前輩進行比較會很有幫助：

*MIPS (1985)*：商業 RISC 的先驅，MIPS 建立了許多 RISC-V 遵循的原則。其簡潔的設計使其在教育和 embedded system 中很受歡迎。然而，MIPS 分裂成多個不相容的變體（MIPS I 到 MIPS V，加上 MIPS32/MIPS64），其專有性質限制了採用。2019 年，MIPS 成為開源，但那時 RISC-V 已經獲得了動力。

*ARM (1985)*：從簡單的 RISC 設計開始，ARM 演變成一個具有眾多 extension 和 profile 的複雜架構。其對功耗效率的關注使其在行動裝置中佔主導地位。然而，ARM 仍然是專有的，需要授權和權利金。該架構在 35 年以上的演進中積累了相當大的複雜性。

*RISC-V (2010)*：從兩個前輩中學習，RISC-V 結合了 MIPS 的簡潔設計哲學和 ARM 對實際應用的實用關注，同時添加了開放性這一關鍵要素。它避免了困擾 MIPS 的碎片化和 ARM 中積累的複雜性。

**ISA 複雜度比較**

*Instruction 數量*：RISC-V 的 base RV32I 有 47 個 instruction。MIPS32 有大約 60 個 base instruction。ARM 的 instruction 數量由於眾多變體而難以確定，但 ARMv8-A 在包括所有 extension 時有數百個 instruction。

*Encoding Format*：RISC-V 使用 6 種基本 instruction format，全部 32 bit（加上 16-bit compressed format）。MIPS 使用 3 種 format，全部 32 bit。ARM 在 Thumb mode 中使用可變長度 encoding，在 ARM mode 中使用固定 32-bit，具有複雜的 encoding 規則。

*Addressing Mode*：RISC-V 支援 base+offset 用於 memory access，複雜 addressing 的 offset 明確計算。MIPS 類似。ARM 支援更複雜的 addressing mode，包括 auto-increment 和 scaled indexing。

趨勢很明顯：RISC-V 最簡單，MIPS 中等複雜，ARM 最複雜。這種簡潔性使 RISC-V 更容易實現、驗證和教學。

**授權和生態系統**

*RISC-V*：完全開放且免費。不需要授權，沒有權利金，沒有限制。任何人都可以實現、修改或擴展 RISC-V。規範公開可用。

*ARM*：專有且需要授權。Architecture license 花費數百萬美元。適用每顆晶片的權利金。修改需要許可。然而，ARM 提供廣泛的 IP、工具和生態系統支援。

*MIPS*：歷史上是專有的，2019 年在 Wave Computing 下成為開源。然而，轉型艱難，MIPS 缺乏 RISC-V 的動力和生態系統。

授權差異是根本性的。RISC-V 實現了專有 ISA 無法匹敵的創新和競爭。

**使用案例和採用**

*Embedded System*：RISC-V 在 microcontroller 和 embedded processor 中迅速獲得市場份額。其模組化允許針對特定需求量身定制的實現。Western Digital 已在 storage controller 中出貨數十億個 RISC-V core。

*Application Processor*：ARM 主導行動裝置，但 RISC-V 正在這個領域嶄露頭角。SiFive 和其他公司正在開發高效能 RISC-V core。開放性質吸引了想要避免 ARM 授權的公司。

*High-Performance Computing*：RISC-V 正在被探索用於 HPC accelerator 和專門處理器。Vector extension 使其在資料並行 workload 中具有競爭力。

*教育和研究*：RISC-V 已成為教學和研究的首選架構，在許多大學中取代了 MIPS。

**未來展望**

RISC-V 的軌跡很明確：由開放性、簡潔性和業界支援驅動的快速增長。它不會一夜之間在 smartphone 中取代 ARM，但它正在成為授權成本和靈活性重要的新設計的預設選擇。

與 MIPS 的比較特別有啟發性。MIPS 具有技術優勢，但保持專有時間太長。當它開放時，RISC-V 已經佔據了開放 ISA 的思想份額。ARM 的技術卓越性和生態系統令人敬畏，但其專有性質為 RISC-V 創造了機會。

RISC-V 不僅代表一個新的 ISA，而且代表處理器架構的新模式：開放、協作，並且不受 vendor lock-in 的限制。這種模式對下一代計算來說是令人信服的。

---

## 總結

RISC-V 建立在 30 年 RISC 架構演進的基礎上，從 MIPS、SPARC、ARM 和 PowerPC 的成功和錯誤中學習。1980 年代的 RISC 革命證明了簡單、規則的 instruction set 可以超越複雜的 instruction set，建立了 RISC-V 今天遵循的原則：load-store 架構、固定長度 instruction、簡單的 addressing mode 和大型 register file。

開放 ISA 運動解決了專有架構的根本問題：授權成本、修改限制、碎片化和 vendor lock-in。RISC-V 提供了一個完全開放且免費的 instruction set architecture，由 RISC-V International 透過協作模式治理，確保供應商中立。任何人都可以實現、修改或擴展 RISC-V，無需費用或授權。

RISC-V 的設計哲學強調**簡潔性**（base ISA 中只有 47 個 instruction）、**模組化**（針對特定需求的可選 extension）、**可擴展性**（custom instruction 不破壞相容性）和**穩定性**（永不改變的 frozen base ISA）。這種模組化方法允許針對特定應用量身定制的實現，從僅 RV32I 的 microcontroller 到 RV64GCV 高效能處理器。

標準 extension 根據需要添加功能：M 用於 multiplication 和 division，A 用於 atomic operation，F 和 D 用於 floating-point，C 用於 compressed instruction，V 用於 vector processing，B 用於 bit manipulation。Profile 如 RVA22 和 RVA23 為特定用例定義標準的 extension 組合，確保軟體可移植性，同時保持靈活性。

與 ARM 和 MIPS 相比，RISC-V 更簡單（更少的 instruction，更簡潔的 encoding）、更現代（沒有歷史包袱）且完全開放（沒有授權費用或限制）。雖然 ARM 主導行動裝置，MIPS 已經衰落，但 RISC-V 在 embedded system 中迅速獲得採用，在 application processor 中嶄露頭角，並成為教育和研究的首選架構。

RISC-V 不僅代表一個新的 ISA，而且代表處理器架構的新模式：開放、協作，並且不受 vendor lock-in 的限制。這種開放性，結合技術卓越性和業界支援，使 RISC-V 成為下一代計算的架構。

---

## Chapter Metadata

**Chapter**: 1 - 什麼是 RISC-V？
**Part**: I - 導論
**版本**: Draft v0p1
**字數**: ~5,500 字
**Section**: 6
**圖表**: 3

**撰寫重點**：

- **目標讀者**：完全初學者、學生、任何 RISC-V 新手
- **深度層級**：高階概覽、歷史背景、動機
- **關鍵概念**：RISC 歷史、開放 ISA 運動、模組化、profile
- **比較**：ARM vs MIPS vs RISC-V（授權、複雜度、生態系統）

**讀者關鍵收穫**：

1. RISC-V 建立在 30 年以上的 RISC 架構演進基礎上
2. 開放 ISA 模式消除了授權費用和限制
3. 模組化設計允許量身定制的實現（RV32I 到 RV64GCV）
4. Profile 確保軟體可移植性（RVA22, RVA23）
5. 比 ARM 簡單，比 MIPS 現代，完全開放

---

**作者**: Danny Jiang
**最後更新**: 2025-12-02
