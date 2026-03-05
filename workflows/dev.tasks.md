---
description: "讀取 plan + spec → 拆解為具體任務 → 產出 tasks.md → 建立 Jira Sub-tasks"
---

# /dev.tasks <TICKET_ID>

## 概述
此 workflow 協助工程師將 `plan.md` 拆解為具體、可執行的任務清單，產出 `tasks.md`，並為每個任務在 Jira 建立 Sub-task。

## 前置條件
- `plan.md` 已存在（由 `/dev.plan` 產出）
- PRD 已存在

---

## 執行步驟

### Step 0: 載入專案設定

#### 0-A: 版本檢查
1. 讀取專案根目錄的 `.speckit-jira-installed`
2. 若不存在 → 提示：`⚠️ 未偵測到 speckit-jira 安裝紀錄。請先執行 ~/speckit-jira/install.sh`
3. 若 `installed_at` 距今超過 30 天 → 提示：`💡 speckit-jira workflow 已安裝 N 天（vX.X.X），建議更新`
4. 不阻擋後續步驟

#### 0-B: 載入設定
1. 讀取 `.speckit-jira.yml`（設定優先權：`.speckit-jira.yml` → `.env` → 預設值）
2. 決定 `$SPEC_ROOT`（同 pm.specify 邏輯）
3. 從 `<TICKET_ID>` 解析 project key，透過 MCP 自動取得 cloudId（若未設定）
4. 讀取 `plan.md` 和 PRD
2. 讀取 `plan.md`
3. 讀取 PRD（作為需求參考）
4. 若 `plan.md` 不存在，提示使用者先執行 `/dev.plan <TICKET_ID>`

### Step 1: 拆解任務
1. 分析 `plan.md` 中的實作步驟
2. 將每個 Phase/步驟拆解為具體的開發任務
3. 每個任務應包含：
   - 清晰的標題
   - 完成條件
   - 預估複雜度（S/M/L）
   - 依賴關係（哪些任務要先完成）

### Step 2: 產出 tasks.md
1. 產出 `tasks.md` 格式如下：
   ```markdown
   # Tasks: <feature title>

   ## 參考
   - PRD: <link>
   - Plan: <link>
   - Jira: <TICKET_ID>

   ## 任務清單

   ### Phase 1: <phase name>

   #### Task 1: <task title>
   - **複雜度**: S
   - **依賴**: 無
   - **完成條件**: <acceptance criteria>
   - **Jira Sub-task**: <SUB-TASK-ID>（自動建立後填入）
   - [ ] 完成

   #### Task 2: <task title>
   - **複雜度**: M
   - **依賴**: Task 1
   - **完成條件**: <acceptance criteria>
   - **Jira Sub-task**: <SUB-TASK-ID>
   - [ ] 完成

   ### Phase 2: <phase name>
   ...
   ```
2. 展示給工程師確認

### Step 3: 建立 Jira Sub-tasks（若 MCP 可用）
1. 取得 Jira 可用的 issue types：
   ```
   使用 MCP tool: getJiraProjectIssueTypesMetadata
   參數: projectIdOrKey = $JIRA_PROJECT_KEY
   ```
2. 對每個 Task 建立 Sub-task：
   ```
   使用 MCP tool: createJiraIssue
   參數:
     projectKey = $JIRA_PROJECT_KEY
     issueTypeName = "Sub-task"
     parent = <TICKET_ID>
     summary = <task title>
     description = <completion criteria>
   ```
3. 將建立的 Sub-task ID 回填到 `tasks.md` 中
4. 儲存更新後的 `tasks.md`

### Step 4: 更新 Jira 主 Ticket
1. 在主 Ticket 加上 comment：
   ```
   📋 Tasks breakdown 已建立：
   - 共 N 個任務
   - Phase 1: N 個任務
   - Phase 2: N 個任務
   - Sub-tasks 已建立：<SUB-TASK-IDs>
   ```
2. 若可用，更新 Jira 狀態為 "Task Review"
3. 告知工程師結果

---

## Jira 降級處理

若 Atlassian MCP Server 連線失敗：

```
⚠️ 無法連接 Jira MCP Server。
Sub-tasks 將不會自動建立。
tasks.md 已產出，你可以手動在 Jira 建立 Sub-tasks。
```

僅產出 `tasks.md`，跳過 Jira Sub-task 建立。

---

## 預期產出
- `tasks.md` — 具體任務清單（含完成條件、複雜度、依賴）
- Jira Sub-tasks（若 MCP 可用）
- Jira comment 摘要
