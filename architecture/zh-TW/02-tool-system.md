> 🌐 **語言**: [English →](../02-tool-system.md) | 中文

# 工具系統架構：42 個模組，一個介面

> **原始檔**：`Tool.ts` (793 行 —— 介面定義), `tools.ts` (390 行 —— 註冊中心), `tools/` (42+ 個目錄)

## 太長不看，一句話總結

Claude Code 採取的每一項行動 —— 讀取檔案、執行 Bash、搜尋網路、產生子智慧體 —— 都透過一個統一的 `Tool` 介面。42 個以上的工具模組，每個都自包含在自己的目錄中，並在啟動時透過一套分層的過濾系統進行裝配：功能開關 → 許可權規則 → 模式限制 → 拒絕列表。

---

## 1. 工具介面：30+ 個方法，一份契約

Claude Code 中的每個工具都實現相同的 `Tool<Input, Output, Progress>` 型別。這是一個擁有 793 行定義的龐大介面，涵蓋了：
- **標識 (Identity)**：名稱、別名、搜尋提示。
- **模式 (Schema)**：基於 Zod v4 的輸入輸出校驗。
- **核心執行 (Execution)**：包含 `call()` 方法。
- **許可權流水線 (Permission)**：輸入驗證、許可權檢查等。
- **行為標誌 (Behavioral Flags)**：是否只讀、併發安全、破壞性等。
- **UI 渲染 (UI Rendering)**：使用 React + Ink 渲染工具的使用、進度和結果訊息。

### buildTool() 工廠模式
為了避免每個工具都必須手動實現 30 多個方法，Claude Code 使用了一個帶有“失敗即關閉（fail-closed）”預設值的工廠函式：
- `isConcurrencySafe` 預設為 `false`（序列執行）。
- `isReadOnly` 預設為 `false`（寫操作，需要許可權）。
- 這種防禦性設計確保瞭如果開發者忘記宣告某個屬性，系統會選擇最安全的行為。

---

## 2. 工具註冊中心：靜態陣列與動態過濾

所有工具都在 `tools.ts` 中透過 `getAllBaseTools()` 註冊。它返回一個扁平的陣列 —— **不是**外掛登錄檔，也不是複雜的索引，這種設計意在保持極簡。

### 功能門控 (Feature-Gated) 工具
許多工具只有在特定的構建標誌（Feature Flags）開啟時才會出現：
- **ALWAYS**: BashTool, FileReadTool, AgentTool, WebSearchTool 等。
- **GATED**: 
  - `PROACTIVE` → SleepTool 
  - `AGENT_TRIGGERS` → Cron 相關工具
  - `COORDINATOR_MODE` → 協調模式相關工具
  - `WEB_BROWSER_TOOL` → 瀏覽器自動化工具
- **ENV-GATED**: `USER_TYPE=ant` 專屬的 REPLTool, ConfigTool 等。

**編譯時最佳化**：透過 `bun:bundle` 編譯標誌，未開啟的功能程式碼會在構建階段被物理剔除，從而減小二進位制體積。

---

## 3. 工具分類

這 42+ 個工具主要分為 6 大功能類別：

1. **檔案操作 (7 個工具)**：Read, Write, Edit, Glob, Grep, NotebookEdit, Snip。
2. **執行指令 (3-4 個工具)**：Bash, PowerShell, REPL, Sleep。
3. **智慧體管理 (6 個工具)**：AgentTool, SendMessage, TaskStop, TeamCreate 等。
4. **外部整合 (5+ 個工具)**：WebFetch, WebSearch, WebBrowser, MCP 相關資源讀取, LSP。
5. **工作流與計劃 (8+ 個工具)**：PlanMode, Worktree, SkillTool, Task (Todo v2) 等。
6. **通知與監控 (4 個工具)**：MonitorTool, PushNotification 等。

---

## 4. 裝配流水線 (Assembly Pipeline)

工具並不是直接從註冊中心進入 LLM。它們會經過多級過濾流水線：
1. **拒絕規則過濾**：移除被明確禁用的工具。
2. **執行時檢查**：呼叫 `isEnabled()`。
3. **模式過濾**：
   - **Simple 模式**：僅保留核心的 Bash 和檔案讀寫工具。
   - **REPL 模式**：隱藏部分被 VM 封裝的底層工具。
4. **MCP 工具合併**：將來自外部協議的工具與內建工具合併。

**快取穩定性**：內建工具作為字首並按名稱排序，MCP 工具緊隨其後。這種排序方式保證了即使新增或刪除 MCP 工具，內建工具的 Prompt 快取依然保持穩定，節省 API 成本。

---

## 5. 工具搜尋：大工具集的延遲載入

當可用工具過多時（如 MCP 載入了數十個工具），`ToolSearchTool` 開啟了延遲載入機制。模型透過關鍵詞匹配發現工具，而不是在初始 Prompt 中載入全量定義，避免了上下文溢位。

---

## 6. 目錄規範

每個工具都遵循一致的目錄結構：
```
tools/BashTool/
├── BashTool.ts      # 實現邏輯 (buildTool({ ... }))
├── prompt.ts        # 面向 LLM 的描述文字
├── UI.tsx           # React+Ink 渲染元件
├── constants.ts     # 常量（如名稱、限制）
└── utils.ts         # 輔助函式
```
對於像 `AgentTool` 這樣的大型工具，結構會更復雜，包含記憶體管理、Worker 排程等獨立模組。

---

## 可遷移設計模式

> 以下來自工具系統的模式可直接應用於任何外掛或擴充套件架構。

### 模式 1：行為標誌優於能力類
不使用複雜的繼承體系（如 `ReadOnlyTool` 類），而是使用基於輸入的布林方法標誌。例如，`BashTool.isReadOnly()` 會根據命令內容（`ls` vs `rm`）動態返回結果。

### 模式 2：排序順序決定快取穩定性
透過確定的排序演算法（內建工具順序在前），確保了大規模部署下的顯著 API 成本節約。

### 模式 3：完全自包含的模組化
每個工具目錄包含一切：實現、Prompt、UI 和測試。工具之間互不交叉，保證了獨立的可測試性和可維護性。

---

## 8. 工具池組裝：Prompt Cache 與經濟學的交匯

> 為什麼工具排序如此重要？因為一個 MCP 工具插入到錯誤位置，就足以使整個 Prompt Cache 失效 —— 讓每次 API 呼叫多花 12 倍的輸入 Token 成本。

// 原始碼位置: src/tools.ts:345-367

### 分割槽排序策略

`assembleToolPool()` 不只是簡單合併工具 —— 它強制執行嚴格的分割槽排序：

```typescript
export function assembleToolPool(permissionContext, mcpTools) {
  const builtInTools = getTools(permissionContext)
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  // 分割槽排序：內建工具作為連續字首，MCP 工具作為字尾
  // 扁平排序會將 MCP 穿插進內建工具之間，
  // 使所有下游快取鍵全部移位
  const byName = (a, b) => a.name.localeCompare(b.name)
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',  // 名稱衝突時內建工具優先（uniqBy 保留首個）
  )
}
```

Anthropic 服務端的快取策略在最後一個內建工具之後放置全域性快取斷點。如果 MCP 工具穿插到內建區間，所有下游快取鍵都會偏移 —— 把 $0.003 的快取命中變成 $0.036 的全價呼叫。

### 拒絕規則預過濾

// 原始碼位置: src/tools.ts:262-269

工具在**傳送給模型之前**就被過濾掉 —— 而非到呼叫時才攔截。模型永遠看不到被禁用的工具，避免了在註定被拒絕的呼叫上浪費 Token。

---

## 9. 工具搜尋：LLM 時代的延遲載入

> 當 MCP 伺服器新增了數十個工具時，模型的 Prompt 會變得臃腫不堪。ToolSearch 提供了優雅的解決方案：延遲模型當前不需要的工具，讓它按需發現。

// 原始碼位置: src/tools.ts:247-249, Tool.ts shouldDefer/alwaysLoad

### 延遲工具的工作方式

| 欄位 | 作用 |
|-------|---------|
| `shouldDefer` | 工具傳送時標記 `defer_loading: true` —— 模型可以看到名稱但看不到 schema |
| `alwaysLoad` | 永不延遲，即使 ToolSearch 處於啟用狀態 |
| `searchHint` | ToolSearch 匹配關鍵詞（如 NotebookEditTool 的 `'jupyter'`） |

### Schema 未傳送問題

// 原始碼位置: src/services/tools/toolExecution.ts:578-597

當模型在未先發現工具的情況下直接呼叫延遲工具時，型別引數（陣列、數字、布林值）會以字串形式到達 —— 導致 Zod 校驗失敗。系統檢測到這種情況後，注入提示："此工具的 schema 未傳送至 API。請先呼叫 ToolSearchTool 載入它，然後重試。"

---

## 10. 上下文修改：改變世界的工具

> 有些工具不只是產生輸出 —— 它們改變執行上下文字身。`cd` 改變工作目錄。系統如何在不破壞併發執行的前提下處理這種副作用？

// 原始碼位置: src/Tool.ts:321-336

### contextModifier 模式

工具可以返回一個 `contextModifier` 函式，用於轉換後續操作的 `ToolUseContext`。例如 `cd` 命令透過此機制修改當前工作目錄。

### 併發安全守衛

**關鍵約束**：`contextModifier` 僅對 `isConcurrencySafe() === false` 的工具生效。道理很簡單 —— 如果兩個工具並行執行且都嘗試修改上下文（比如都 `cd` 到不同目錄），最終狀態就是不確定的。透過將上下文修改限制在序列工具上，系統在設計層面消除了這個競態條件。

---

## 11. 執行流水線：從模型輸出到副作用

> 工具呼叫不是簡單的函式呼叫。它是一條包含驗證、許可權檢查、鉤子、執行、結果處理和上下文修改的多階段流水線 —— 全部透過 AsyncGenerator 編排。

// 原始碼位置: src/services/tools/toolExecution.ts:337-490, 599-800+

### runToolUse() 入口點

```
模型輸出 tool_use block
    │
    ▼
runToolUse() — AsyncGenerator<MessageUpdateLazy>
    │
    ├── 1. 查詢工具（名稱匹配 → 別名回退 → 報錯）
    ├── 2. 檢查中斷訊號 → 如已中斷：yield 取消訊息，返回
    └── 3. streamedCheckPermissionsAndCallTool()
            ├── 4. Zod schema 驗證（inputSchema.safeParse）
            ├── 5. tool.validateInput() —— 工具特定驗證
            ├── 6. [BashTool] 投機啟動分類器（並行執行）
            ├── 7. PreToolUse hooks（可修改輸入或阻止）
            ├── 8. canUseTool() —— 許可權裁定
            │       ├── allow → 繼續
            │       ├── deny → 返回錯誤
            │       └── ask → 互動提示 / 協調者路由
            ├── 9. tool.call() —— 核心執行
            ├── 10. PostToolUse hooks（可修改輸出）
            ├── 11. mapToolResultToToolResultBlockParam()
            ├── 12. processToolResultBlock() → 大結果持久化
            └── 13. 應用 contextModifier + 注入 newMessages
```

### 大結果持久化

// 原始碼位置: src/utils/toolResultStorage.ts

當工具輸出超過 `maxResultSizeChars` 時，系統將其寫入磁碟並返回預覽 + 檔案路徑。模型可用 `FileReadTool` 讀取完整輸出。各工具上限：BashTool 30K / FileEditTool 100K / GlobTool 100K / GrepTool 100K / FileReadTool 無限自管。

---

## 12. 搜尋工具簡析：GlobTool 與 GrepTool

> 這兩個工具提供模型的"專案內查詢"能力 —— 一個按模式發現檔案，一個按正則搜尋內容。

### GlobTool

// 原始碼位置: src/tools/GlobTool/GlobTool.ts

按模式匹配檔案路徑，結果按修改時間降序排列，預設上限 100 個檔案。標記為 `isConcurrencySafe: true` 且 `isReadOnly: true` —— 無需許可權，可並行執行。

### GrepTool

// 原始碼位置: src/tools/GrepTool/GrepTool.ts

封裝 `ripgrep`，帶安全約束：結果上限 250 條匹配（支援 offset 分頁）、自動排除 `.git`/`node_modules`、支援多種輸出模式（內容/僅檔名/計數）、上下文行支援。同樣標記為併發安全且只讀。

---

## 13. 檔案工具：模型觸及程式碼庫的雙手

> 檔案工具形成三位一體 —— Read、Edit、Write —— 各自具有不同的安全屬性，共享一個 `FileStateCache` 來防止模型覆蓋你的未儲存修改。

### FileReadTool：六種輸出型別，一個介面

// 原始碼位置: src/tools/FileReadTool/FileReadTool.ts:337-718（共 1,184 行）

FileReadTool 是系統中最大的工具。它不只是"讀檔案" —— 它是一個多型的內容消化引擎，支援文字、影象、Jupyter Notebook、PDF、大 PDF 分頁提取、以及檔案未變化存根六種輸出型別。

**去重最佳化**：如果模型對同一檔案/同一範圍讀取兩次且檔案未變化（mtime 匹配），返回 `file_unchanged` 存根而非完整內容。內部遙測顯示約 18% 的 Read 呼叫是同檔案重複 —— 這節省了大量 `cache_creation` Token。

**安全約束**：
- 阻止裝置檔案路徑（`/dev/zero`、`/dev/random`、`/dev/stdin`）—— 會掛起程序
- UNC 路徑檢查 —— 防止 Windows NTLM 憑據透過 SMB 洩露
- `maxResultSizeChars: Infinity` —— 因為持久化到磁碟再讓模型 Read 會形成迴圈依賴

### FileEditTool：字串替換 + 過時寫入檢測

// 原始碼位置: src/tools/FileEditTool/FileEditTool.ts:86-595（共 626 行）

FileEditTool 使用**字串替換**而非 diff/patch。模型提供 `old_string` 和 `new_string`，要求 `old_string` 在檔案中唯一（或使用 `replace_all`）。

**過時寫入守衛**（最關鍵的安全機制）：

```
1. 模型讀取檔案 → FileStateCache 記錄 { content, mtime }
2. 使用者在外部編輯檔案 → mtime 變化
3. 模型嘗試編輯 → mtime > 快取的 mtime → 拒絕
   "檔案自上次讀取後已被修改。請重新讀取。"
```

這防止了模型覆蓋你的手動編輯。Windows 上還有內容對比回退機制，處理雲同步/防毒軟體導致的時間戳誤報。

**驗證流水線**（寫入前 8 項檢查）：
1. 團隊記憶金鑰檢測（防止 API 金鑰寫入共享記憶）
2. `old_string !== new_string`（空操作防護）
3. 拒絕規則檢查
4. 檔案大小限制（1 GiB —— V8 字串長度邊界）
5. 過時寫入檢測（mtime + 內容對比）
6. `old_string` 存在性檢查
7. 唯一性檢查（非 `replace_all` 模式）
8. 設定檔案特殊驗證

### FileWriteTool：建立或完全覆寫

FileWriteTool 故意保持簡單 —— 建立新檔案或完全覆寫已有檔案。與 FileEditTool 共用許可權管道。自動建立父目錄。當 `old_string` 等於整個檔案內容時使用它。

### FileStateCache：三個工具的共享狀態

三個工具共享一個按絕對路徑為鍵的 `readFileState` Map。FileReadTool 在讀取時寫入條目；FileEditTool 在寫入前檢查條目、寫入後更新條目。這個共享快取使得跨 Read→Edit 工作流的過時寫入檢測成為可能。

---

## 總結

| 維度 | 細節 |
|--------|--------|
| **介面定義** | 單一 `Tool` 型別，30+ 方法，高內聚性 |
| **序號產生器制** | 扁平陣列，極簡設計，無複雜容器 |
| **裝配規則** | 分割槽排序（內建字首 + MCP 字尾），保障 Prompt Cache 穩定性 |
| **模式校驗** | 強制 Zod 校驗，執行時安全保證 |
| **預設傾向** | "失敗即關閉"工廠設計，預設最嚴許可權 |
| **ToolSearch** | 延遲工具 → 按需發現 → schema 未傳送檢測 |
| **執行流水線** | 13 步流水線：查詢 → 驗證 → hooks → 許可權 → 呼叫 → hooks → 持久化 |
| **上下文修改** | `contextModifier` —— 僅限非併發工具（競態條件防禦） |
| **結果管理** | 按工具 `maxResultSizeChars`；溢位 → 磁碟持久化 + 預覽 |
| **檔案工具** | Read (6 種輸出 + 去重) / Edit (8 項驗證 + 過時寫入守衛) / Write (簡單覆寫) |
