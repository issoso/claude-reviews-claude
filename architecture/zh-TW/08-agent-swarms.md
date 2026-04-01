# 08 — Swarm 智慧體：多智慧體團隊協調

> **範圍**: `tools/TeamCreateTool/`、`tools/SendMessageTool/`、`tools/shared/spawnMultiAgent.ts`、`utils/swarm/`（~30 個檔案，~6.8 千行）、`utils/teammateMailbox.ts`（1,184 行）
>
> **一句話概括**: Claude Code 如何在 tmux 窗格、iTerm2 分屏或程序內生成並行隊友 —— 全部透過基於檔案的郵箱系統配合鎖檔案併發控制進行協調。

---

## 架構概覽

```mermaid
graph TB
    subgraph Leader["👑 團隊領導"]
        TC["TeamCreate<br/>建立團隊 + config.json"]
        SPAWN["spawnMultiAgent<br/>生成隊友"]
        SEND["SendMessage<br/>私信 / 廣播 / 關閉"]
        INBOX_POLL["useInboxPoller<br/>輪詢訊息"]
    end

    subgraph Backend["🖥️ 後端"]
        TMUX["TmuxBackend<br/>分屏 / 獨立視窗"]
        ITERM["ITermBackend<br/>原生 iTerm2 窗格"]
        INPROC["InProcessBackend<br/>同程序，獨立查詢迴圈"]
    end

    subgraph FS["📁 檔案系統 (~/.claude/teams/)"]
        CONFIG["config.json<br/>團隊清單"]
        MBOX["inboxes/{name}.json<br/>郵箱（加鎖）"]
        TASKS["tasks/{team}/"]
    end

    subgraph Workers["🔧 隊友"]
        W1["teammate-1<br/>researcher"]
        W2["teammate-2<br/>test-runner"]
        W3["teammate-3<br/>implementer"]
    end

    TC --> CONFIG
    SPAWN --> Backend
    TMUX --> W1
    ITERM --> W2
    INPROC --> W3
    SEND --> MBOX
    W1 --> MBOX
    W2 --> MBOX
    W3 --> MBOX
    INBOX_POLL --> MBOX
```

---

## 1. 團隊生命週期

### 建立團隊

`TeamCreateTool` 初始化團隊基礎設施：

1. 生成唯一團隊名（衝突時使用隨機片語合）
2. 建立團隊領導條目，確定性 agent ID：`team-lead@{teamName}`
3. 將 `config.json` 寫入 `~/.claude/teams/{team-name}/`
4. 註冊會話清理（退出時自動刪除，除非已顯式刪除）
5. 重置任務列表目錄，從編號 1 開始

### 生成隊友

`spawnMultiAgent.ts`（1,094 行）處理完整的生成流程：

1. **解析模型**：`inherit` → 領導的模型；`undefined` → 硬編碼回退
2. **生成唯一名稱**：檢查現有成員，追加 `-2`、`-3` 等
3. **檢測後端**：tmux > iTerm2 > 程序內（見 §3）
4. **建立窗格/程序**：後端特定的生成邏輯
5. **構建 CLI 引數**：傳播 `--agent-id`、`--team-name`、`--agent-color`、`--permission-mode`
6. **註冊到團隊檔案**：將成員條目新增到 `config.json`
7. **傳送初始訊息**：將提示詞寫入隊友的郵箱
8. **註冊後臺任務**：用於 UI 任務標識顯示

---

## 2. 郵箱系統

智慧體間通訊的骨幹是**基於檔案的郵箱**，配合鎖檔案併發控制。

### 目錄結構

```
~/.claude/teams/{team-name}/
├── config.json              # 團隊清單
└── inboxes/
    ├── team-lead.json       # 領導的收件箱
    ├── researcher.json      # 隊友收件箱
    └── test-runner.json     # 隊友收件箱
```

### 訊息型別

| 型別 | 方向 | 用途 |
|------|------|------|
| 純文字私信 | 任意 → 任意 | 直接訊息 |
| 廣播（`to: "*"`） | 領導 → 全體 | 團隊公告 |
| `idle_notification` | 工人 → 領導 | "我完成了/被阻塞了/失敗了" |
| `permission_request` | 工人 → 領導 | 工具許可權委託 |
| `permission_response` | 領導 → 工人 | 許可權授予/拒絕 |
| `sandbox_permission_request` | 工人 → 領導 | 網路訪問審批 |
| `plan_approval_request` | 工人 → 領導 | 計劃審查（planModeRequired） |
| `shutdown_request` | 領導 → 工人 | 優雅關閉 |
| `shutdown_approved/rejected` | 工人 → 領導 | 關閉確認 |

### 併發控制

多個 Claude 例項可以併發寫入 —— 鎖檔案透過指數退避重試（10 次重試，5-100ms 超時）序列化訪問。

---

## 3. 後端檢測與執行

三種後端決定隊友的物理執行方式：

| 特性 | Tmux | iTerm2 | 程序內 |
|------|------|--------|--------|
| 隔離性 | 獨立程序 | 獨立程序 | 同程序，獨立查詢迴圈 |
| UI 可見性 | 帶彩色邊框的窗格 | 原生 iTerm2 窗格 | 後臺任務標識 |
| 前置條件 | tmux 已安裝 | `it2` CLI 已安裝 | 無 |
| 非互動模式（`-p`） | ❌ | ❌ | ✅（強制） |
| Socket 隔離 | PID 作用域：`claude-swarm-{pid}` | N/A | N/A |

檢測優先順序：**tmux 內部 → iTerm2 原生 → tmux 外部 → 程序內回退**。

---

## 4. 許可權委託

隊友沒有互動終端 —— 它們將許可權決策委託給領導：

1. 工人需要許可權 → 建立 `permission_request` 訊息
2. 寫入領導的郵箱
3. 領導的 `useInboxPoller` 拾取請求
4. 領導向使用者顯示許可權提示
5. 領導傳送 `permission_response` 回工人的郵箱
6. 工人輪詢收件箱，獲取響應，繼續或中止

### Plan Mode Required

帶 `plan_mode_required: true` 生成的隊友：
- 必須進入 plan 模式並建立計劃
- 計劃作為 `plan_approval_request` 傳送給領導
- 領導稽核後傳送 `plan_approval_response`
- 批准時，領導的許可權模式被繼承（`plan` 對映為 `default`）

---

## 5. 智慧體身份系統

```
格式：{name}@{teamName}
示例：researcher@my-project、team-lead@my-project
```

### CLI 標誌傳播

生成隊友時，領導傳播以下標誌：

| 標誌 | 條件 | 用途 |
|------|------|------|
| `--dangerously-skip-permissions` | bypass 模式 + 非 planModeRequired | 繼承許可權繞過 |
| `--permission-mode auto` | auto 模式 | 繼承分類器 |
| `--model {model}` | 顯式模型覆蓋 | 使用領導的模型 |
| `--settings {path}` | CLI 設定路徑 | 共享設定 |
| `--plugin-dir {dir}` | 內聯外掛 | 共享外掛 |
| `--parent-session-id {id}` | 始終 | 血統追蹤 |

---

## 6. 團隊清理

### 優雅關閉

`SendMessage(type: shutdown_request)` → 隊友回應 `shutdown_approved/rejected`：
- **批准**：程序內隊友中止查詢迴圈；窗格隊友呼叫 `gracefulShutdown(0)`
- **拒絕**：隊友提供原因，繼續工作

### 會話清理

`cleanupSessionTeams()` 在領導退出時執行：
1. 終止孤立的隊友窗格
2. 刪除團隊目錄：`~/.claude/teams/{team-name}/`
3. 刪除任務目錄：`~/.claude/tasks/{team-name}/`
4. 銷燬為隔離隊友建立的 git worktree

---

## 可遷移設計模式

> 以下來自 Swarm 系統的模式可直接應用於任何多智慧體或分散式協調架構。

### 為什麼用檔案郵箱？

郵箱系統使用純 JSON 檔案 + 鎖檔案，而非 IPC、WebSocket 或共享記憶體：
- **跨程序**：tmux 窗格是獨立程序，沒有共享記憶體
- **崩潰安全**：訊息持久化在磁碟上，即使隊友崩潰也不丟失
- **可除錯**：`cat ~/.claude/teams/my-team/inboxes/researcher.json`
- **簡單**：無守護程序，無埠分配，無服務發現

### 一個領導，多個工人

架構強制執行嚴格的領導-工人層級：
- 每個領導會話只能有一個團隊
- 工人不能建立團隊或批准自己的計劃
- 關閉始終由領導發起，工人確認
- 許可權委託始終是 工人 → 領導 → 工人

---

## 8. 協調器模式

**原始碼座標**: `src/coordinator/coordinatorMode.ts`

協調器模式將領導從任務分發者轉變為**綜合引擎** —— 它不僅僅是委派工作，還要理解和整合結果。

### 啟用：雙重門控

構建時特性標誌 AND 執行時環境變數必須同時啟用。恢復會話時，`matchSessionMode()` 自動翻轉變數以匹配恢復會話的模式。

### 協調器工作流

```
研究（工人，並行）→ 綜合（協調器整合發現）→ 實現（工人，按檔案集序列）→ 驗證（工人，並行）
```

核心原則：
- **協調器擁有綜合權** —— 不做"基於你的發現"式委派；協調器必須理解並重述
- **並行是超能力** —— 獨立工人併發執行
- **讀寫隔離** —— 研究任務並行，寫操作按檔案集序列

---

## 9. 任務型別聯合（7 種變體）

**原始碼座標**: `src/tasks/`

每個後臺任務由七種狀態變體之一表示：

```typescript
export type TaskState =
  | LocalShellTaskState         // 本地 shell 命令
  | LocalAgentTaskState         // 透過 AgentTool 的子代理
  | RemoteAgentTaskState        // 遠端 CCR 代理
  | InProcessTeammateTaskState  // 同程序團隊成員
  | LocalWorkflowTaskState      // 本地工作流
  | MonitorMcpTaskState         // MCP 伺服器監控
  | DreamTaskState              // 自動記憶整理
```

### 完成通知

代理完成時注入 `<task-notification>` XML，包含 `task-id`、`status`（completed/failed/killed）、`result`（最終文字響應）和 `usage` 統計。因為作為 `user` 型別訊息注入，LLM 在對話流中自然處理它。

---

## 10. Agent 間通訊協議

**原始碼座標**: `src/tools/SendMessageTool/`

### 結構化訊息型別

除純文字外，代理可以交換帶型別的控制訊息：`shutdown_request`、`shutdown_response`、`plan_approval_response`。

### 訊息路由

```
SendMessage(to="researcher", message="...")
  ↓
程序內隊友？ → 直接 pendingMessages
  ↓ 否
本地代理？ → queuePendingMessage → 在工具輪次邊界消費
  ↓ 否
窗格（tmux/iterm2）？ → 檔案系統郵箱
  ↓ 否
UDS/Bridge？ → socket/bridge 傳輸
  ↓ 否
"*"（廣播）？ → 遍歷所有團隊成員，逐個傳送
```

### 跨會話通訊（UDS）

啟用 `feature('UDS_INBOX')` 時，同一機器上的 Claude Code 會話可透過 Unix Domain Socket 通訊。訊息封裝為 `<cross-session-message>` XML。

---

## 11. DreamTask 與 UltraPlan

### DreamTask：自動記憶整理

DreamTask 執行後臺代理，審查近期會話歷史並將學習成果整理到 `MEMORY.md`。`priorMtime` 欄位充當回滾鎖 —— 如果整理在寫入過程中被終止，系統可以恢復檔案到整理前的狀態。

### UltraPlan：編排式遠端執行

UltraPlan 將代理正規化擴充套件到透過 CCR（Claude Code Runner）的遠端執行，實現"先規劃-後執行"的工作流：遠端生成計劃，必須經過使用者審批後才能開始實施。

---

## 元件總結

| 元件 | 行數 | 角色 |
|------|------|------|
| `spawnMultiAgent.ts` | 1,094 | 統一的隊友生成邏輯 |
| `teammateMailbox.ts` | 1,184 | 基於檔案的郵箱 + 鎖檔案併發 |
| `teamHelpers.ts` | 684 | 團隊檔案 CRUD、清理、worktree 管理 |
| `SendMessageTool.ts` | 918 | 私信、廣播、關閉、計劃審批 |
| `TeamCreateTool.ts` | 241 | 團隊初始化 |
| `backends/registry.ts` | 465 | 後端檢測：tmux > iTerm2 > 程序內 |
| `teammateLayoutManager.ts` | ~400 | 窗格建立、顏色分配、邊框狀態 |

Swarm 系統是 Claude Code 操作最複雜的功能 —— 它將程序管理、基於檔案的 IPC、終端多路複用和分散式許可權委託融合為一個多智慧體框架。檔案郵箱設計優先考慮簡單性和可除錯性而非效能，這在"分散式系統"實際上是共享同一檔案系統的多個 AI 智慧體時是正確的權衡。

---

**上一篇**: [← 07 — 許可權流水線](07-permission-pipeline.md)
**下一篇**: [→ 09 — 會話持久化](09-session-persistence.md)
