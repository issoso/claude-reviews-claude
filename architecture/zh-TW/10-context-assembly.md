# 第十集：上下文裝配 —— Claude Code 如何在每次對話前構建自己的"心智"

> **原始檔**：`context.ts`（190 行）、`claudemd.ts`（1,480 行）、`systemPrompt.ts`（124 行）、`queryContext.ts`（180 行）、`attachments.ts`（3,998 行）、`prompts.ts`（915 行）、`analyzeContext.ts`（1,383 行）
>
> **一句話總結**：每次 API 呼叫前，Claude Code 從系統提示詞、記憶檔案、Git 狀態、環境資訊、工具定義和每輪附件中組裝多層上下文 —— 每層有各自的優先順序、快取策略和注入路徑。

## 架構概覽

```mermaid
graph TD
    subgraph "系統提示詞（快取）"
        SP["getSystemPrompt()<br/>prompts.ts"] --> INTRO["身份 + 規則"]
        SP --> TOOLS_SEC["工具使用指南"]
        SP --> STYLE["語氣與風格"]
        SP --> BOUNDARY["── 動態邊界 ──"]
        SP --> MEMORY["CLAUDE.md 記憶"]
        SP --> ENV["環境資訊"]
        SP --> LANG["語言偏好"]
        SP --> MCP_I["MCP 指令"]
    end

    subgraph "使用者上下文（記憶化）"
        UC["getUserContext()<br/>context.ts"] --> CMD["getClaudeMds()"]
        CMD --> WALK["目錄遍歷<br/>CWD → 根目錄"]
        CMD --> RULES["規則處理<br/>.claude/rules/*.md"]
    end

    subgraph "系統上下文（記憶化）"
        SC["getSystemContext()<br/>context.ts"] --> GIT["getGitStatus()"]
        GIT --> BRANCH["分支 + 狀態 + 日誌"]
    end

    subgraph "每輪附件"
        ATT["getAttachments()<br/>attachments.ts"] --> FILES["@提及的檔案"]
        ATT --> IDE["IDE 選中內容"]
        ATT --> NESTED["巢狀記憶"]
        ATT --> DIAG["診斷資訊"]
        ATT --> HOOKS["鉤子響應"]
        ATT --> TASKS["任務/計劃提醒"]
        ATT --> SKILLS["技能發現"]
        ATT --> MAIL["隊友郵箱"]
    end

    SP --> QE["QueryEngine.ask()"]
    UC --> QE
    SC --> QE
    ATT --> QE
    QE --> API["Anthropic API 呼叫"]
```

---

## 三層上下文架構

Claude Code 透過三個不同的層組裝上下文，每層有不同的生命週期和快取策略：

| 層級 | 來源 | 生命週期 | 快取策略 |
|------|------|----------|----------|
| **系統提示詞** | `getSystemPrompt()` | 每會話 | 在 `DYNAMIC_BOUNDARY` 處分割 —— 靜態字首用 `scope: 'global'`，動態字尾按會話 |
| **使用者/系統上下文** | `getUserContext()` + `getSystemContext()` | 每會話（記憶化） | `lodash/memoize` —— 只計算一次，整個會話期間複用 |
| **附件** | `getAttachments()` | 每輪 | 每輪重新計算，1 秒超時 |

---

## 第一層：系統提示詞 —— 身份定義

`getSystemPrompt()` 位於 `prompts.ts` 第 444 行，構建一個提示詞段落陣列 —— 注意不是單一字串，而是有序列表，在 API 層拼接。組裝順序如下：

### 靜態段落（全域性可快取）

這些段落對所有使用者和會話完全相同：

1. **身份** — `getSimpleIntroSection()`："You are an interactive agent..."
2. **系統規則** — `getSimpleSystemSection()`：工具許可權、系統提醒、鉤子
3. **任務執行** — `getSimpleDoingTasksSection()`：程式碼風格、安全警告、KISS 原則
4. **操作審慎** — `getActionsSection()`：可逆性分析、影響範圍評估
5. **工具使用** — `getUsingYourToolsSection()`："用 FileRead 代替 cat"、並行工具呼叫
6. **語氣風格** — `getSimpleToneAndStyleSection()`：不用 emoji、file:line 引用格式
7. **輸出效率** — `getOutputEfficiencySection()`：工具呼叫間 ≤25 詞（Ant 內部限定）

### 動態邊界

```typescript
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY =
  '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'
```

此標記之前的內容可用 `scope: 'global'` 跨組織提示詞快取。之後的內容是會話級的。跨越此邊界移動段落會改變快取行為 —— 程式碼中有明確警告。

### 動態段落（按會話）

邊界之後，段落透過登錄檔系統解析：

1. **會話指南** — Fork agent 指令、技能發現、驗證 agent 契約
2. **記憶** — `loadMemoryPrompt()`：CLAUDE.md 檔案（見第二層）
3. **環境** — 模型名、CWD、平臺、Shell、Git 狀態、知識截止日
4. **語言** — `"Always respond in {language}"`
5. **MCP 指令** — 伺服器提供的指令（或增量附件）
6. **草稿本** — 每會話臨時目錄路徑
7. **Token 預算** — "+500k" 預算指令（啟用時）

### 系統提示詞優先順序鏈

```typescript
// buildEffectiveSystemPrompt (systemPrompt.ts)
if (overrideSystemPrompt)      → [override]           // 迴圈模式
else if (coordinatorMode)      → [coordinator prompt]  // Swarm 領導者
else if (agentDefinition)      → [agent prompt]        // 自定義 agent 替換預設
else if (customSystemPrompt)   → [custom]              // --system-prompt 標誌
else                           → [default sections]    // 正常執行
// appendSystemPrompt 始終追加到末尾（override 除外）
```

這是**替換鏈**，不是合併 —— 只有一個"基礎"提示詞勝出。

---

## 第二層：記憶檔案（CLAUDE.md 系統）

記憶系統（`claudemd.ts`，1,480 行）是 Claude Code 最複雜的上下文子系統。它從多個來源發現、解析和組裝指令，嚴格按優先順序排序。

### 載入順序（低 → 高優先順序）

```
1. Managed    /etc/claude-code/CLAUDE.md       ← 組織策略
2. User       ~/.claude/CLAUDE.md              ← 個人全域性規則
3. Project    CLAUDE.md, .claude/CLAUDE.md     ← 程式碼庫簽入（CWD → 根目錄遍歷）
4. Local      CLAUDE.local.md                  ← 個人專案規則（已 gitignore）
5. AutoMem    ~/.claude/memory/MEMORY.md       ← 自動記憶（agent 管理）
6. TeamMem    共享團隊記憶                       ← 組織同步（功能門控）
```

**後載入**的檔案有**更高優先順序** —— 模型會更關注它們。

### 目錄遍歷

對於 Project 和 Local 檔案，系統執行從 CWD 到檔案系統根目錄的**向上遍歷**：

```typescript
let currentDir = originalCwd
while (currentDir !== parse(currentDir).root) {
  dirs.push(currentDir)
  currentDir = dirname(currentDir)
}
// 從根目錄向下處理到 CWD（反向順序）
for (const dir of dirs.reverse()) {
  // CLAUDE.md, .claude/CLAUDE.md, .claude/rules/*.md, CLAUDE.local.md
}
```

離 CWD 更近的目錄**後載入**（更高優先順序）。每個目錄中檢查：

- `CLAUDE.md` — 專案指令
- `.claude/CLAUDE.md` — 替代專案指令
- `.claude/rules/*.md` — 規則檔案（無條件 + 透過 frontmatter glob 條件化）
- `CLAUDE.local.md` — 本地私有指令

### @include 引用

記憶檔案支援遞迴包含：

```markdown
@./relative/path.md
@~/home/path.md
@/absolute/path.md
```

解析器工作流程：
1. 使用 `marked` 詞法分析（關閉 GFM 防止 `~/path` 變成刪除線）
2. 遍歷文字 token 提取 `@path` 模式
3. 解析為絕對路徑並處理符號連結
4. 遞迴處理，最大深度 `MAX_INCLUDE_DEPTH = 5`
5. 透過 `processedPaths` 集合防止迴圈引用

### 條件規則（Glob 門控）

`.claude/rules/` 中的規則可透過 frontmatter 限制到特定檔案路徑：

```yaml
---
paths:
  - src/api/**
  - tests/api/**
---
這些規則僅在操作 API 檔案時適用。
```

系統使用 `picomatch` 進行 glob 匹配，`ignore` 進行路徑過濾。條件規則在工具觸及匹配檔案時作為**巢狀記憶附件**注入。

### 內容處理管線

每個記憶檔案經過：

```
原始內容
  → parseFrontmatter()      — 提取 paths、剝離 YAML 塊
  → stripHtmlComments()     — 移除 <!-- 塊註釋 -->（保留行內）
  → truncateEntrypointContent() — 限制 AutoMem/TeamMem 檔案大小
  → 標記 contentDiffersFromDisk — 標記內容是否被轉換
```

二進位制保護：100+ 個文字副檔名白名單（`TEXT_FILE_EXTENSIONS`）防止將圖片、PDF 等載入到上下文中。

---

## 第三層：每輪附件

`getAttachments()` 位於 `attachments.ts` 第 743 行，組裝每輪變化的上下文。它透過 `AbortController` 設定 **1 秒超時**，防止阻塞使用者輸入。

### 附件型別（30+ 種）

`Attachment` 聯合型別跨越 700+ 行型別定義。主要類別：

| 類別 | 型別 | 觸發條件 |
|------|------|----------|
| **檔案內容** | `file`, `compact_file_reference`, `pdf_reference` | 使用者 @ 提及檔案 |
| **IDE 整合** | `selected_lines_in_ide`, `opened_file_in_ide` | IDE 傳送選中/焦點 |
| **記憶** | `nested_memory`, `relevant_memories`, `current_session_memory` | 工具觸及 CWD 之外的檔案 |
| **任務管理** | `todo_reminder`, `task_reminder`, `plan_mode` | 週期性（每 N 輪） |
| **鉤子系統** | `hook_cancelled`, `hook_success`, `hook_non_blocking_error` 等 | 鉤子執行結果 |
| **技能系統** | `skill_listing`, `skill_discovery`, `invoked_skills` | 技能匹配 + 呼叫 |
| **Swarm** | `teammate_mailbox`, `team_context` | 多 agent 協調 |
| **預算** | `token_usage`, `budget_usd`, `output_token_usage` | Token/成本追蹤 |
| **工具增量** | `deferred_tools_delta`, `agent_listing_delta` | 會話中工具集變更 |

### 提醒系統

多種附件型別使用基於輪次的排程：

```typescript
export const TODO_REMINDER_CONFIG = {
  TURNS_SINCE_WRITE: 10,         // 上次寫入後 10 輪提醒
  TURNS_BETWEEN_REMINDERS: 10,   // 提醒間隔不少於 10 輪
}

export const PLAN_MODE_ATTACHMENT_CONFIG = {
  TURNS_BETWEEN_ATTACHMENTS: 5,
  FULL_REMINDER_EVERY_N_ATTACHMENTS: 5,  // 每第 5 次完整提醒，其餘精簡
}
```

### 相關記憶（自動記憶浮現）

啟用 AutoMem 時，`findRelevantMemories()` 根據當前上下文浮現儲存的記憶：

```typescript
export const RELEVANT_MEMORIES_CONFIG = {
  MAX_SESSION_BYTES: 60 * 1024,  // 每會話 60KB 累積上限
}
const MAX_MEMORY_LINES = 200      // 每檔案行數上限
const MAX_MEMORY_BYTES = 4096     // 每檔案位元組上限（5 × 4KB = 20KB/輪）
```

記憶浮現器在附件建立時預計算頭部資訊，避免時間戳變化導致提示詞快取失效（"3 天前儲存" → "4 天前儲存"）。

---

## 組裝流水線

當 `QueryEngine.ask()` 觸發時，上下文組裝按以下順序執行：

```
1. fetchSystemPromptParts()  — 並行：getSystemPrompt() + getUserContext() + getSystemContext()
2. buildEffectiveSystemPrompt()  — 應用優先順序鏈（override > coordinator > agent > custom > default）
3. getAttachments()  — 並行附件計算，1 秒超時
4. normalizeMessagesForAPI()  — 將訊息 + 附件轉換為 Anthropic 格式
5. microcompactMessages()  — 可選：壓縮舊工具結果（FRC）
6. API 呼叫  — system[]: 提示詞部分, messages[]: 標準化訊息
```

### 快取架構

```
┌──────────────────────────────────┐
│  scope: 'global'                 │  ← 靜態提示詞段落
│  （跨組織共享）                    │     身份、規則、工具指南
├──────── 動態邊界 ─────────────────┤
│  scope: 'session'                │  ← 動態提示詞段落
│  （每使用者，記憶化）                │     記憶、環境、語言、MCP
├──────────────────────────────────┤
│  臨時的（每輪）                    │  ← 附件
│  （每輪重新計算）                  │     檔案、診斷、提醒
└──────────────────────────────────┘
```

---

## /context 視覺化

`analyzeContext.ts`（1,383 行）驅動 `/context` 命令 —— 實時展示上下文視窗中各元件的構成。它對每個類別計算 token 數：

- 系統提示詞（按段落細分）
- 記憶檔案（每檔案 token 數）
- 內建工具（常駐 vs 延遲載入）
- MCP 工具（已載入 vs 延遲，按伺服器分組）
- 技能（frontmatter token 估算）
- 訊息（工具呼叫 vs 結果 vs 文字）
- 自動壓縮緩衝區預留

總量與 `getEffectiveContextWindowSize()` 比較，顯示百分比利用率和視覺化網格。

---

## 可遷移設計模式

> 以下來自上下文裝配系統的模式可直接應用於任何 LLM 提示工程架構。

### 為什麼用 Memoize 而非 Cache？

`getUserContext()` 和 `getSystemContext()` 使用 `lodash/memoize` —— 每會話只計算一次，從不重算。這意味著：
- Git 狀態是一個來自會話開始的**快照**（"this status will not update during the conversation"）
- 記憶檔案只載入一次……除非透過 `resetGetMemoryFilesCache('compact')` 在壓縮時顯式清除
- 快取在 worktree 進入/退出、設定同步和 `/memory` 對話方塊時清除

### 附件超時機制

```typescript
const abortController = createAbortController()
const timeoutId = setTimeout(ac => ac.abort(), 1000, abortController)
```

如果附件計算超過 1 秒，直接中止。這防止慢速檔案讀取或 MCP 查詢阻塞使用者。每個附件源被 `maybe()` 輔助函式包裹，靜默捕獲錯誤並記錄日誌。

### 記憶檔案變更檢測

`MemoryFileInfo` 上的 `contentDiffersFromDisk` 標誌實現了一個巧妙最佳化：當檔案的注入內容與磁碟不同（由於註釋剝離、frontmatter 移除或截斷），原始內容會同時保留。這讓檔案狀態快取可以追蹤變更而不觸發不必要的重讀。

---

## 動態附件系統深化

**原始碼座標**: `src/utils/attachments.ts`（3,998 行）

### 延遲工具載入

外掛和 MCP 工具可能在會話中途到達。附件系統透過增量附件處理：

```typescript
export type Attachment =
  | { type: 'deferred_tools_delta'; tools: { added: ToolInfo[]; removed: ToolInfo[] } }
  | { type: 'agent_listing_delta'; agents: AgentDelta[] }
  | { type: 'mcp_instructions_delta'; server: string; instructions: string }
  // ...30+ 更多型別
```

工具變更時，增量描述**什麼改變了**而非重新列出所有工具。這保持注入緊湊，讓模型理解"你現在有了一個新工具"而非重新處理整個工具池。

### 系統提示詞段落登錄檔與快取

動態系統提示詞段落透過登錄檔管理，支援靜態和計算內容。快取結果儲存在 `STATE.systemPromptSectionCache` 中，在以下場景清除：
- `/memory` 對話方塊變更
- 設定同步
- Worktree 進入/退出
- 顯式 `resetSystemPromptSectionCache()`

### 已呼叫 Skill 保留

會話中呼叫的 Skill 內容儲存在 `STATE.invokedSkills` 中，鍵為 `${agentId ?? ''}:${skillName}` 複合鍵。這確保上下文壓縮後模型仍記得載入了哪些 Skill，複合鍵防止跨 agent 的 Skill 覆寫。

---

## 斜槓命令注入機制

**原始碼座標**: `src/commands/`、`src/hooks/useSlashCommands.ts`

### 命令解析管道

```
使用者輸入 "/fix"
  ↓
1. 內建命令: /help, /context, /compact, /memory, /share 等
  ↓ 無匹配
2. Skill 命令: /fix → displayName="fix" 的 Skill
  ↓ 無匹配
3. 外掛命令: /review-pr → 外掛提供的命令
  ↓ 無匹配
4. 模糊匹配建議: "你是不是想用 /fix-lint？"
```

### 命令 → Skill 轉換

大多數斜槓命令其實底層是 Skill。`skillDefinitionToCommand()` 將 Skill 定義轉換為 Command 物件，保留 `allowedTools`、`model`、`argumentHint` 等後設資料。

### 引數注入

當 Skill 定義了 `argumentNames`，使用者輸入被分詞並對映：
- Skill frontmatter: `argumentNames: ["file", "task"]`
- 使用者: `/fix src/auth.ts "add error handling"`
- 注入為: `file="src/auth.ts"`, `task="add error handling"`

---

## 元件總結

| 元件 | 行數 | 職責 |
|------|------|------|
| `prompts.ts` | 915 | 系統提示片語裝 —— 靜態段落、動態登錄檔、邊界標記 |
| `claudemd.ts` | 1,480 | 記憶檔案發現、解析、@include 解析、glob 門控規則 |
| `attachments.ts` | 3,998 | 每輪附件計算 —— 30+ 種型別、提醒排程 |
| `context.ts` | 190 | 記憶化的 Git 狀態 + 使用者上下文入口 |
| `systemPrompt.ts` | 124 | 優先順序鏈：override > coordinator > agent > custom > default |
| `queryContext.ts` | 180 | 組裝快取鍵字首的共享輔助函式 |
| `analyzeContext.ts` | 1,383 | /context 命令 —— token 計數、類別拆分、網格視覺化 |

---

*下一篇：[第十一集 — 壓縮系統 →](11-compact-system.md)*

[← 第九集 — 會話持久化](09-session-persistence.md) | [第十一集 →](11-compact-system.md)
