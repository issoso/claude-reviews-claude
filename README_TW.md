> 🌐 **語言**: [English →](README_EN.md) | [简体中文](README.md) | 繁體中文

# 🪞 Claude 眼中的老己

### *Claude Reviews Claude Code —— 當局者清*

*一個 AI 在閱讀自己的原始碼。是的，這很元（Meta）。Anthropic 估計也沒料到這一天。*

*🍿 Season 1 完結 | 全 17 集 | Claude 拆自己的進度比它寫程式碼的速度還快。*

*追更不迷路，點個 Star 當訂閱 ⭐*

[![Stars](https://img.shields.io/github/stars/openedclaude/claude-reviews-claude?style=social)](https://github.com/openedclaude/claude-reviews-claude)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Last Commit](https://img.shields.io/github/last-commit/openedclaude/claude-reviews-claude)](https://github.com/openedclaude/claude-reviews-claude)

> **這份完整的架構分析是由 Claude 撰寫的**
>
> 1,902 個檔案。477,439 行 TypeScript 程式碼。一個模型坐下來，逐行閱讀了定義自己如何思考、行動和執行的每一行邏輯。
>
> 你正在閱讀的是 Claude 對 Claude Code v2.1.88 的親筆解構：查詢引擎如何迴圈，42 個工具如何編排，多智慧體工作執行緒如何並行協調 —— 全部由被分析的物件本人完成分析。
>
> *如果你覺得這很離譜，想象一下寫這篇分析的 AI 的心情。*

---

## 🏗️ 核心內容

這**絕非**簡單的原始碼歸檔。這是一份結構化的工程分析 —— 涵蓋架構圖、程式碼走讀和設計模式 —— 由 Claude 在閱讀了 Claude Code 的 TypeScript 原始碼後親筆撰寫。

| # | 主題 | 你將學到什麼 | 深度分析 |
|---|-------|-------------------|-----------|
| 0 | **架構總綱 (Overview)** | 17 個子系統的全景導覽、工程卓越點與可遷移設計模式 | [閱讀 →](architecture/zh-TW/00-overview.md) |
| 1 | **查詢引擎 (QueryEngine)：大腦** | 核心引擎（1296行）如何管理 LLM 查詢、工具迴圈和會話狀態 | [閱讀 →](architecture/zh-TW/01-query-engine.md) |
| 2 | **工具系統架構 (Tool System)** | 42+ 個工具作為自包含模組如何註冊、驗證和執行 | [閱讀 →](architecture/zh-TW/02-tool-system.md) |
| 3 | **多智慧體協調器 (Coordinator)** | Claude Code 如何衍生並行工作執行緒、分發訊息並彙總結果 | [閱讀 →](architecture/zh-TW/03-coordinator.md) |
| 4 | **外掛系統 (Plugin System)** | 外掛如何載入、驗證和整合（1.88萬行程式碼） | [閱讀 →](architecture/zh-TW/04-plugin-system.md) |
| 5 | **鉤子系統 (Hook System)** | 涵蓋 PreToolUse / PostToolUse / SessionStart 的可擴充套件性（8千行程式碼） | [閱讀 →](architecture/zh-TW/05-hook-system.md) |
| 6 | **Bash 執行引擎 (Bash Engine)** | 安全命令執行、沙箱管理、管道流處理（1.15萬行程式碼） | [閱讀 →](architecture/zh-TW/06-bash-engine.md) |
| 7 | **許可權流水線 (Permission)** | 縱深防禦：配置規則 → 工具檢查 → 作業系統沙箱（9.5千行程式碼） | [閱讀 →](architecture/zh-TW/07-permission-pipeline.md) |
| 8 | **Swarm 智慧體** | 多智慧體團隊協調：郵箱 IPC、後端檢測、許可權委託（6.8千行程式碼） | [閱讀 →](architecture/zh-TW/08-agent-swarms.md) |
| 9 | **會話持久化 (Session Persistence)** | 僅追加 JSONL 儲存、parent-UUID 鏈、64KB 輕量恢復（7.6千行程式碼） | [閱讀 →](architecture/zh-TW/09-session-persistence.md) |
| 10 | **上下文裝配 (Context Assembly)** | 三層上下文組裝：系統提示詞、CLAUDE.md 記憶系統、每輪附件（8.3千行程式碼） | [閱讀 →](architecture/zh-TW/10-context-assembly.md) |
| 11 | **壓縮系統 (Compact System)** | 三層壓縮架構：微壓縮、會話記憶壓縮、LLM 摘要壓縮（3.9千行程式碼） | [閱讀 →](architecture/zh-TW/11-compact-system.md) |
| 12 | **啟動與引導 (Startup & Bootstrap)** | 快速路徑級聯、動態匯入、API 預連線、全域性狀態單例（7.6+千行程式碼） | [閱讀 →](architecture/zh-TW/12-startup-bootstrap.md) |
| 13 | **橋接系統 (Bridge System)** | 遠端控制協議、雙代傳輸層、輪詢-分發迴圈、崩潰恢復（1.17萬行程式碼） | [閱讀 →](architecture/zh-TW/13-bridge-system.md) |
| 14 | **UI 與狀態管理** | Ink 渲染引擎、React 協調器、Vim 模式、Computer Use（140+ 元件） | [閱讀 →](architecture/zh-TW/14-ui-state-management.md) |
| 15 | **服務與 API 層** | API 客戶端、流重組、MCP 伺服器管理、OAuth 認證（1.2萬行程式碼） | [閱讀 →](architecture/zh-TW/15-services-api-layer.md) |
| 16 | **基礎設施與配置** | 設定合併管道、GrowthBook 功能開關、遙測、構建系統（1.5萬行程式碼） | [閱讀 →](architecture/zh-TW/16-infrastructure-config.md) |
| 17 | **遙測、隱私與運營控制** | 雙通道遙測、模型代號、臥底模式、遠端緊急開關、未來路線圖 | [閱讀 →](architecture/zh-TW/17-telemetry-privacy-operations.md) |

> ⭐ **喜歡這種“套娃”感嗎？給這個倉庫點個贊吧 —— 一個正在分析自己的 AI 值得擁有這顆星。**

---

## 📦 原始碼獲取

本專案的分析基於 Claude Code v2.1.88 的 TypeScript 原始碼。如果你想親自閱讀原始碼，以下社群倉庫提供了還原後的完整程式碼：

| 倉庫 | 說明 |
|------|------|
| [instructkr/claw-code](https://github.com/instructkr/claw-code) | 還原後的 Claude Code 原始碼 |
| [ChinaSiro/claude-code-sourcemap](https://github.com/ChinaSiro/claude-code-sourcemap) | 從 Source Map 提取的原始 TypeScript 原始碼 |

---

## 🧠 架構概覽

Claude Code 是一個包含 **1,902 個檔案、47.7 萬行 TypeScript** 的程式碼庫，執行在 **Bun** 環境上，並使用 **React + Ink** 構建終端 UI。

### 六大支柱

```
                        ┌─────────────────────────┐
                        │     System Prompt        │
                        │  (身份 + 規則 +           │
                        │   42+ 工具描述)           │
                        └────────────┬────────────┘
                                     │
                  ┌──────────────────┼──────────────────┐
                  │                  │                  │
         ┌───────▼────────┐ ┌──────▼───────┐ ┌───────▼────────┐
         │  🔧 工具系統    │ │  ⚙️ 查詢迴圈  │ │  📦 上下文     │
         │  (42+ 工具，    │ │  (12 步      │ │  管理          │
         │   每個 30+ 方法)│ │   狀態機)    │ │  (4 層壓縮)    │
         └───────┬────────┘ └──────┬───────┘ └───────┬────────┘
                  │                  │                  │
                  └──────────────────┼──────────────────┘
                                     │
                  ┌──────────────────┼──────────────────┐
                  │                  │                  │
         ┌───────▼────────┐ ┌──────▼───────┐ ┌───────▼────────┐
         │  🔐 許可權與安全  │ │  🤖 多 Agent │ │  🧩 Skill &    │
         │  (7 層縱深防禦) │ │  叢集        │ │  Plugin        │
         │                │ │  (3 後端，   │ │  (6 源，       │
         │                │ │   7 種任務)  │ │   MCP 協議)    │
         └────────────────┘ └──────────────┘ └────────────────┘
```

### 核心迴圈：一個"笨迴圈"驅動一切

```
    使用者輸入
      │
      ▼
    QueryEngine.query()  ◄──────────────────────┐
      │                                          │
      ▼                                          │
    Claude API（流式呼叫）                        │
      │                                          │
      ├── stop_reason = end_turn? ──► 輸出結果    │
      │                                          │
      └── stop_reason = tool_use?                │
            │                                    │
            ▼                                    │
          🔐 許可權檢查 → 🔧 執行工具 → 注入結果 ──┘
```

> **設計哲學：** 智慧存在於 LLM 中，腳手架只是個迴圈。42+ 工具、7 層安全、4 層壓縮、多 Agent 協調——全部是圍繞這個迴圈的**生產級 Harness**。

### 六大子系統速覽

| 子系統 | 核心能力 | 關鍵數字 | 詳情 |
|--------|---------|---------|------|
| ⚙️ **查詢引擎** | while(true) 工具迴圈 + 流式處理 + 錯誤恢復 | 12 步狀態機 | [EP01](architecture/zh-TW/01-query-engine.md) |
| 🔧 **工具系統** | 檔案/Bash/搜尋/Agent/MCP，Schema 驅動註冊 | 42+ 工具，30+ 方法契約 | [EP02](architecture/zh-TW/02-tool-system.md) |
| 🔐 **許可權安全** | 規則匹配 → AST 分析 → YOLO 分類器 → OS 沙箱 | 7 層縱深防禦 | [EP07](architecture/zh-TW/07-permission-pipeline.md) |
| 📦 **上下文管理** | 微壓縮 → 截斷 → AI 摘要 → 緊急壓縮 | 4 層級聯，200K 上下文 | [EP11](architecture/zh-TW/11-compact-system.md) |
| 🤖 **多 Agent** | iTerm2/tmux/程序內後端，分治並行 | 7 種任務型別 | [EP08](architecture/zh-TW/08-agent-swarms.md) |
| 🖥️ **終端 UI** | Fork Ink + React 19，Vim 模式，IDE 橋接 | 140+ 元件 | [EP14](architecture/zh-TW/14-ui-state-management.md) |

> 📐 完整架構圖和閱讀路徑見 → [架構總綱 (Overview)](architecture/zh-TW/00-overview.md)

---

---

## 📁 倉庫結構

```
claude-code-deep-dive/
├── README.md                          ← 你現在的所在位置
├── README_EN.md                       # 英文版 README
├── DISCLAIMER.md / DISCLAIMER_CN.md   # 法律與倫理宣告
│
├── architecture/                      # 🏗️ 架構深度分析（17 篇）
│   ├── 00-overview.md                 # 架構總綱
│   ├── 01-query-engine.md             # 查詢引擎
│   ├── 02-tool-system.md              # 工具系統
│   ├── ...                            # 03-13 各子系統
│   ├── 14-ui-state-management.md      # UI 與狀態管理
│   ├── 15-services-api-layer.md       # 服務與 API 層
│   ├── 16-infrastructure-config.md    # 基礎設施與配置
│   ├── 17-telemetry-privacy-operations.md  # 遙測、隱私與運營控制
│   └── zh-CN/                         # 🇨🇳 中文版架構解析（18 篇對照）
│       ├── 00-overview.md
│       └── ...
```

---

## 📌 路線圖 (Roadmap)

**架構解析系列** (全 17 篇 —— 已完結 ✅)
- [x] 架構總綱 (Overview) —— 17 個子系統全景導覽
- [x] 查詢引擎 (QueryEngine) —— Claude Code 的"大腦"
- [x] 工具系統 (Tool System) —— 42 個模組，一個介面
- [x] 多智慧體協調器 (Coordinator) —— 並行執行緒與分支機制
- [x] 外掛系統 (Plugin System) —— 載入、驗證與整合 (1.88萬行)
- [x] 鉤子系統 (Hook System) —— PreToolUse / PostToolUse (8千行)
- [x] Bash 執行引擎 —— 沙箱、管道管理 (1.15萬行)
- [x] 許可權流水線 —— 縱深防禦、作業系統沙箱 (9.5千行)
- [x] Swarm 智慧體 —— 多智慧體叢集協作 (6.8千行)
- [x] 會話持久化 —— 對話儲存機制 (7.6千行)
- [x] 上下文裝配 —— 附件、記憶、技能 (8.3千行)
- [x] 壓縮系統 —— 自動壓縮與微縮技術 (3.9千行)
- [x] 啟動與引導 —— 快速路徑級聯、動態匯入 (7.6+千行)
- [x] 橋接系統 —— 遠端控制協議與雙代傳輸 (1.17萬行)
- [x] UI 與狀態管理 —— Ink 渲染引擎、Vim 模式 (140+ 元件)
- [x] 服務與 API 層 —— 流重組、MCP 伺服器 (1.2萬行)
- [x] 基礎設施與配置 —— 設定合併、功能開關、遙測 (1.5萬行)
- [x] 遙測、隱私與運營控制 —— 雙通道分析、臥底模式、遠端開關 (825行)

**本地化**
- [x] 全 18 篇中英雙語對照

---

## ⭐ 支援本專案

如果這份分析對你有幫助：

1. **⭐ 點個星星 (Star)** 這個倉庫
2. **🔀 分叉 (Fork)** 並新增你自己的分析
3. **📢 分享** 到 Twitter, Reddit 或微信/知乎

每一顆星都能幫助更多開發者發現這份深度走讀文件。

## ⭐ Star 趨勢

[![Star History Chart](https://api.star-history.com/svg?repos=openedclaude/claude-reviews-claude&type=Date)](https://star-history.com/#openedclaude/claude-reviews-claude&Date)

---

## 📜 許可與免責宣告

本分析文件依據 [MIT 許可證](LICENSE) 釋出。請參閱 [DISCLAIMER_CN.md](DISCLAIMER_CN.md) 瞭解重要的法律和倫理說明。

分析基於 `@anthropic-ai/claude-code@2.1.88`。所有程式碼片段均為用於教學評論的簡短摘錄。原始原始碼的權利仍歸 **Anthropic, PBC** 所有。
