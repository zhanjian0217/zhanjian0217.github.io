---
pubDatetime: 2026-04-02
title: 拆解 Claude Code 的 API Request：System Prompt 與 Message 結構全解析
postSlug: claude-code-api-request-analysis
featured: false
draft: false
tags:
  - ai
  - claude-code
description: 從一份實際的 API request body 拆解 Claude Code 的完整結構，理解 system prompt、system reminder、tools 與 messages 的設計邏輯
---

# 拆解 Claude Code 的 API Request：System Prompt 與 Message 結構全解析

## 前言

使用 Claude Code 時，底層實際上是透過 Anthropic Messages API 與 Claude 模型溝通
但 Claude Code 在發送 request 之前，會注入大量的 system prompt、工具定義、動態上下文等資訊
這些都不是使用者直接寫的——而是 Claude Code 自動組裝的

本文從一份實際發送給 Anthropic Claude API 的 request body —— 拆解其完整結構
理解 Claude Code 是怎麼把「使用者的一句話」包裝成一個完整的 API 請求

---

## 一、Request 的整體結構

一個 Claude Code 發出的 API request 大致長這樣：

```json
{
  "model": "claude-opus-4-6",
  "system": [ ... ],      // System Prompt（核心行為指令）
  "messages": [ ... ],    // 對話歷史（包含動態注入的 system-reminder）
  "tools": [ ... ]        // 工具定義（JSON Schema）
}
```

四大區塊各司其職：

| 區塊       | 角色                   | 特性         |
| ---------- | ---------------------- | ------------ |
| `model`    | 指定使用的模型         | 固定值       |
| `system`   | 核心 system prompt     | 穩定、可快取 |
| `messages` | 對話歷史 + 動態注入    | 每輪變動     |
| `tools`    | 可用工具的 schema 定義 | 預載入的工具 |

---

## 二、System Prompt — 核心行為指令

`system` 欄位是 Anthropic Messages API 原生支援的頂層參數，模型會將其視為最高優先級的指令。`system` 包含 3 個 text block

### 2.1 三個 Text Block

**Block 1：Billing Metadata**

```json
{
  "type": "text",
  "text": "x-anthropic-billing-header: cc_version=2.1.83.c50; cc_entrypoint=cli; cch=4abe5;"
}
```

不是給模型看的指令，而是用於計費與版本追蹤的 metadata。記錄了 Claude Code 的版本號、入口點等資訊

**Block 2：角色定義（一句話）**

```json
{
  "type": "text",
  "text": "You are Claude Code, Anthropic's official CLI for Claude.",
  "cache_control": { "type": "ephemeral", "ttl": "1h" }
}
```

簡短的角色宣告，注意這裡帶了 `cache_control`

**Block 3：完整行為規範（約 400+ 行）**

這是整個 system prompt 的主體，內容非常龐大。以下列出主要章節：

```
# System                       — 基本行為規則（輸出格式、工具權限、hook 機制）
# Doing tasks                  — 任務執行原則（先讀再改、不過度工程）
# Executing actions with care  — 謹慎執行（不可逆操作要確認）
# Using your tools             — 工具使用規則（用 Read 不用 cat、用 Grep 不用 grep）
# Tone and style               — 風格（簡潔、不用 emoji、用 file:line 引用）
# Output efficiency            — 輸出效率（直奔重點、不廢話）
# auto memory                  — Memory 持久化系統（四種記憶類型的讀寫規則）
# Environment                  — 環境資訊（工作目錄、平台、shell、模型版本）
```

幾個值得注意的設計：

- **不過度工程**：明確指示「不要加 docstring、不要加不必要的 error handling、不要為未來需求設計」
- **工具優先於 Bash**：寧可用內建的 Read / Edit / Grep 工具，也不要用 `cat` / `sed` / `grep` 指令
- **Memory 系統**：Claude Code 有一套基於檔案的持久記憶機制，分為 user / feedback / project / reference 四種類型

### 2.2 cache_control 的作用

Block 2 和 Block 3 都帶了 `cache_control`：

```json
"cache_control": { "type": "ephemeral", "ttl": "1h" }
```

這是 Anthropic API 的 [Prompt Caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) 功能
由於 system prompt 在多輪對話中幾乎不變，加上 cache 可以避免重複處理相同的 token，降低延遲與成本

_`ephemeral` 表示快取存活 1 小時，過期後重新計算_

---

## 三、System Reminder — 動態注入的上下文

### 3.1 什麼是 System Reminder？

除了正式的 `system` 欄位，Claude Code 還會在 `messages` 中注入 `<system-reminder>` 標籤。它的形式是普通的 user message text，但被 `<system-reminder>` 標籤包裹：

```json
{
  "role": "user",
  "content": [
    { "type": "text", "text": "<system-reminder>...</system-reminder>" },
    { "type": "text", "text": "今天 MLB 有比賽嗎" } // 真正的使用者輸入
  ]
}
```

**為什麼不直接放在 `system` 裡？**

`system` 欄位的內容在整個對話中是固定的（且已加了 cache），但有些資訊需要**每一輪動態更新**——例如可用工具清單可能隨著對話進展而變化、當前日期每天不同
把這些動態資訊以 `<system-reminder>` 的形式注入 messages，既不影響 system prompt 的快取，又能確保模型拿到最新的上下文

### 3.2 三種 System Reminder 的用途

第一次 user message 會發送包含 3 個 system-reminder：

**Reminder 1：Deferred Tools 清單**

```xml
<system-reminder>
The following deferred tools are now available via ToolSearch:
AskUserQuestion, CronCreate, CronDelete, CronList,
EnterPlanMode, ExitPlanMode, NotebookEdit, WebFetch, WebSearch...
</system-reminder>
```

告訴 Claude：「這些工具存在，但 schema 還沒載入。需要時先用 `ToolSearch` 取得完整定義。」這是一種 lazy loading 機制

**Reminder 2：Skills 清單**

```xml
<system-reminder>
The following skills are available for use with the Skill tool:
- update-config: ...
- keybindings-help: ...
- simplify: ...
- loop: ...
- schedule: ...
- claude-api: ...
</system-reminder>
```

告訴 Claude 有哪些 slash command（如 `/commit`、`/simplify`）可以呼叫

**Reminder 3：當前日期**

```xml
<system-reminder>
Today's date is 2026-03-26.
</system-reminder>
```

LLM 本身沒有即時時間概念，需要外部注入

### 3.3 System vs System Reminder 對照

|              | `system`（API 欄位）    | `<system-reminder>`（messages 內） |
| ------------ | ----------------------- | ---------------------------------- |
| API 層級     | 原生 system prompt      | 普通 user message text             |
| 模型如何看待 | 最高優先級指令          | 被訓練為系統資訊對待               |
| 內容特性     | 穩定、長期不變          | 動態、每輪可能更新                 |
| 快取         | 有 `cache_control`      | 無（不需要，因為會變動）           |
| 典型內容     | 角色定義、行為規範、SOP | 可用工具、skills、日期             |

---

## 四、Messages — 一次對話的完整流程

### 4.1 對話背景

使用者問了：「今天 MLB 有比賽嗎」& 「轉換成 +8 時區顯示」。整個對話產生了 8 個 messages

### 4.2 逐步拆解

#### Message 1 — User 提問

```
role: user
content: [system-reminder x3] + "今天 MLB 有比賽嗎"
```

Claude Code 在使用者輸入前面注入了 3 個 system-reminder，然後才是真正的使用者問題

#### Message 2 — Assistant 呼叫 ToolSearch

```
role: assistant
content: [thinking] + ToolSearch({ query: "WebSearch", max_results: 1 })
```

Claude 判斷需要搜尋網路，但 `WebSearch` 是 deferred tool——只知道名字，沒有完整的 JSON Schema，無法直接呼叫。所以先呼叫 `ToolSearch` 來取得 `WebSearch` 的完整定義

#### Message 3 — ToolSearch 結果回傳

```
role: user
content: tool_result(tool_reference: WebSearch) + "Tool loaded."
```

Harness 回傳 WebSearch 的 schema 已載入
_注意這裡的 `role: user` 不是真正的使用者——**所有 tool result 在 Anthropic API 中都必須以 `role: user` 回傳**，這是 [API 的設計規範](https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use)_

> Continue the conversation by sending a new message with the `role` of `user`, and a `content` block containing the `tool_result` type and the following information

#### Message 4 — Assistant 呼叫 WebSearch

```
role: assistant
content: WebSearch({ query: "MLB games today March 26 2026" })
```

現在 WebSearch 的 schema 已載入，Claude 正式發出搜尋請求

#### Message 5 — WebSearch 結果回傳

```
role: user
content: tool_result（搜尋結果：11 場比賽、連結清單、轉播資訊）
```

Harness 執行實際的網路搜尋，把結果以 `tool_result` 的形式回傳

#### Message 6 — Assistant 整理回覆

```
role: assistant
content: 用中文整理的比賽表格（含隊伍、時間、轉播平台）
```

Claude 將搜尋結果整理成結構化的中文回答，不需要呼叫任何工具

#### Message 7 — User 追問

```
role: user
content: "轉換成 +8 時區顯示"
```

使用者追問，要求時區轉換

#### Message 8 — Assistant 直接回覆

```
role: assistant
content: [thinking] + 時區轉換後的表格
```

這次不需要工具，Claude 直接計算 ET → UTC+8（+12 小時）並回覆。`thinking` block 中的推理過程被加密簽名（`signature` 欄位），外部無法查看具體內容

### 4.3 Deferred Tools 與 Lazy Loading 機制

整個 Message 2-3 的 ToolSearch 流程，體現了 Claude Code 的 **lazy loading 設計**：

```
預載入的工具（在 tools 欄位中有完整 schema）：
  Agent, Bash, Edit, Glob, Grep, Read, Write, Skill, ToolSearch

Deferred 工具（只有名字，需要時才載入 schema）：
  WebSearch, WebFetch, TaskCreate, NotebookEdit, CronCreate...
```

**為什麼要這樣設計？**

每個工具的 JSON Schema 定義都會佔用 token。如果把所有工具的完整 schema 都放進 `tools` 欄位，會大幅增加每次 request 的 token 數量（和成本）
Deferred tools 機制讓 Claude 只在需要時才載入特定工具的 schema，節省不必要的 token 開銷

**流程圖：**

```
使用者提問
  → Claude 判斷需要 WebSearch
  → WebSearch 是 deferred tool，schema 未載入
  → 呼叫 ToolSearch 取得 schema
  → schema 載入完成
  → 正式呼叫 WebSearch
  → 取得結果、回覆使用者
```

---

## 五、Tools — 工具定義

### 5.1 工具的結構

每個工具在 `tools` 陣列中以標準格式定義：

```json
{
  "name": "Bash",
  "description": "Executes a given bash command...",
  "input_schema": {
    "type": "object",
    "properties": {
      "command": { "type": "string", "description": "The command to execute" },
      "timeout": { "type": "number" },
      "description": { "type": "string" }
    },
    "required": ["command"]
  }
}
```

模型根據 `description` 判斷何時使用該工具，根據 `input_schema` 生成合法的參數

### 5.2 預載入 vs Deferred

| 類型     | 範例                                                          | 特性                                                  |
| -------- | ------------------------------------------------------------- | ----------------------------------------------------- |
| 預載入   | Agent, Bash, Edit, Glob, Grep, Read, Write, Skill, ToolSearch | 完整 schema 在 `tools` 中，可直接呼叫                 |
| Deferred | WebSearch, WebFetch, TaskCreate, NotebookEdit, CronCreate...  | 只有名字在 system-reminder 中，需先用 ToolSearch 載入 |

預載入的都是高頻工具（讀檔、寫檔、搜尋、執行指令），幾乎每次對話都會用到。Deferred 的則是低頻或情境性工具，按需載入以節省 token

### 5.3 工具呼叫的 Message 格式

工具呼叫在 messages 中表現為一組 request-response：

```json
// Assistant 發起工具呼叫
{ "role": "assistant", "content": [
    { "type": "tool_use", "id": "toolu_xxx", "name": "WebSearch", "input": { "query": "..." } }
]}

// Harness 回傳結果（注意 role 是 user）
{ "role": "user", "content": [
    { "type": "tool_result", "tool_use_id": "toolu_xxx", "content": "..." }
]}
```

每次工具呼叫都會消耗兩個 message（一去一回），這也是為什麼一個簡單的問題最終產生了 8 個 messages

---

## 附錄：完整 Request 結構示意

```json
{
  "model": "claude-opus-4-6",

  "system": [
    { "text": "billing metadata" },
    { "text": "You are Claude Code...", "cache_control": {...} },
    { "text": "完整行為規範（400+ 行）...", "cache_control": {...} }
  ],

  "messages": [
    { "role": "user",      "content": ["<system-reminder>...deferred tools", "<system-reminder>...skills", "<system-reminder>...date", "今天 MLB 有比賽嗎"] },
    { "role": "assistant",  "content": ["thinking", "ToolSearch(WebSearch)"] },
    { "role": "user",       "content": ["tool_result: WebSearch loaded"] },
    { "role": "assistant",  "content": ["WebSearch(MLB games today)"] },
    { "role": "user",       "content": ["tool_result: 搜尋結果"] },
    { "role": "assistant",  "content": ["整理後的比賽表格"] },
    { "role": "user",       "content": ["轉換成 +8 時區顯示"] },
    { "role": "assistant",  "content": ["thinking", "時區轉換後的表格"] },
  ],

  "tools": [
    { "name": "Agent",     "description": "...", "input_schema": {...} },
    { "name": "Bash",      "description": "...", "input_schema": {...} },
    { "name": "Edit",      "description": "...", "input_schema": {...} },
    { "name": "Glob",      "description": "...", "input_schema": {...} },
    { "name": "Grep",      "description": "...", "input_schema": {...} },
    { "name": "Read",      "description": "...", "input_schema": {...} },
    { "name": "Write",     "description": "...", "input_schema": {...} },
    { "name": "Skill",     "description": "...", "input_schema": {...} },
    { "name": "ToolSearch","description": "...", "input_schema": {...} }
  ]
}
```
