---
description: "PM 從 Jira Ticket 起草 PRD — 含 Clarify 互動、Draft-First 狀態機、跨 Session 持久化"
---

# /pm.specify <TICKET_ID>

## 概述
此 workflow 協助 PM 從 Jira Ticket 建立產品需求文件（PRD）。採用 **Draft-First** 設計：PRD 草稿先存在 `drafts/` 目錄，定稿後才建 feature branch。支援跨對話 session 無縫接續。

## 前置條件
- Atlassian MCP Server 已連線（若未連線，進入降級模式）

---

## 執行步驟

### Step 0: 載入專案設定

#### 0-A: 版本檢查
1. 讀取專案根目錄的 `.speckit-jira-installed`
2. 若檔案不存在 → 顯示：
   ```
   ⚠️ 未偵測到 speckit-jira 安裝紀錄。
   請先執行安裝：~/speckit-jira/install.sh <your-tool>
   ```
3. 若檔案存在，檢查 `installed_at` 時間戳記：
   - 若距今超過 30 天 → 顯示：
     ```
     💡 你的 speckit-jira workflow 已安裝 N 天（v0.x.x）。
     建議更新：cd ~/speckit-jira && git pull && ./install.sh <your-tool>
     ```
   - 若在 30 天內 → 靜默通過，不顯示任何訊息
4. 不管版本新舊，都繼續執行後續步驟（只是提醒，不阻擋）

#### 0-B: 載入設定

設定優先權：`.speckit-jira.yml` → `.env` → 自動偵測

1. 找到專案根目錄（往上層尋找 `.git`）
2. 讀取 `.speckit-jira.yml`：
   - `spec_mode`：`local`（預設）/ `submodule` / `external`
   - `spec_path`：spec 目錄的相對路徑（預設 `specs/`）
   - `jira_cloud_id`（選填）
   - `jira_project_key`（選填）
3. 若 `spec_mode: external`，從 `.env` 讀取 `SPEC_REPO_PATH`
4. 若 `.speckit-jira.yml` 不存在且 `.env` 也沒設定：
   - 預設使用 `spec_mode: local`、`spec_path: specs/`
5. 從 `<TICKET_ID>` 解析 project key（如 `HTGO2-123` → `HTGO2`）
6. 若未設定 `jira_cloud_id`，透過 MCP `getAccessibleAtlassianResources` 自動取得
7. 決定 spec 根目錄 `$SPEC_ROOT`：
   - `local` / `submodule` → `$PROJECT_ROOT/$spec_path`
   - `external` → `$SPEC_REPO_PATH`

### Step 1: 狀態偵測（State Detection）

檢查 `$SPEC_ROOT/drafts/<TICKET_ID>/` 是否存在：

#### 情況 A：不存在 → NEW 模式
1. 從 Jira 擷取 Ticket 資訊：
   ```
   使用 MCP tool: getJiraIssue
   參數: cloudId = $JIRA_CLOUD_ID, issueIdOrKey = <TICKET_ID>
   ```
2. 建立 `drafts/<TICKET_ID>/` 目錄
3. 將 Jira 內容儲存為 `jira-snapshot.md`
4. 初始化 `state.yml`：
   ```yaml
   ticket: <TICKET_ID>
   phase: drafting
   created_at: <timestamp>
   updated_at: <timestamp>
   clarify_rounds: 0
   prd_version: 0
   branch_name: null
   ```
5. 初始化空的 `clarify-log.md` 和 `prd.md`
6. 更新 Jira 狀態為 "Specifying"（若 transition 可用）
7. 在 Jira 加上 comment：「PRD 起草已開始」
8. 進入 Step 2: Clarify

#### 情況 B：存在 + phase: drafting → RESUME 模式
1. 讀取 `state.yml` 取得目前進度
2. 讀取 `prd.md` 取得目前草稿
3. 讀取 `clarify-log.md` 取得過去的問答紀錄
4. 比對 Jira 最新狀態 vs `jira-snapshot.md`：
   - 若有差異，列出變更並提醒使用者
   - 更新 `jira-snapshot.md`
5. 向使用者報告：
   ```
   📋 「PROJ-123」PRD 進度恢復：
   - 狀態：起草中（第 N 輪 Clarify）
   - 上次更新：<timestamp>
   - 未回答的問題：N 個
   要繼續 Clarify 嗎？
   ```
6. 進入 Step 2: Clarify（從上次中斷處繼續）

#### 情況 C：存在 + phase: finalized → FINALIZE 模式
1. 讀取 `prd.md` 最終版
2. 詢問使用者：「PRD 已定稿但尚未發布，要 (1) 建立 Branch 並發布 (2) 繼續修改？」
3. 根據選擇進入 Step 3 或 Step 2

#### 情況 D：Feature branch 已存在 → PUBLISHED / REVISE 模式
1. 讀取現有 PRD
2. 詢問使用者：「PRD 已發布到 branch `<branch_name>`，要進入修改模式嗎？」
3. 若要修改，讀取最新 Jira comment（可能有 Dev 回饋）
4. 進入 Step 2: Clarify（修改模式）

### Step 2: Clarify 互動

1. 分析需求內容，找出模糊或不完整的地方
   - ⚠️ **重要限制：不要詢問技術實作細節**
     你是協助 PM 釐清「產品需求」與「商業邏輯」的角色。請絕對避免詢問 RD 相關的技術細節（如：資料庫 Schema、GraphQL 還是 REST API、延遲載入的寫法、前/後端架構等）。
   - **問題應專注於**：
     - 使用者行為（User flow）
     - 畫面行為與邊界條件（Edge cases）
     - 效能與體驗「期望值」（例如：「需要即時反應嗎？」而非「要用 WebSockets 嗎？」）
     - 驗收條件與成功指標
2. 產出釐清問題清單（每次最多 3 個問題，以編號列出，避免造成使用者負擔）
3. **持久化**：將每輪問答記錄到 `clarify-log.md`：
   ```markdown
   ## Round N — <date>

   **Q1: <問題>**
   A1: <使用者回答>

   **Q2: <問題>**
   A2:（尚未回答）
   ```
4. 等待使用者回覆
5. 根據回覆更新 PRD 草稿
6. 更新 `state.yml` 的 `clarify_rounds` 和 `updated_at`
7. 若仍有不清楚的地方，重複 Step 2
8. 當使用者說「可以了」或所有問題都已回覆，詢問是否定稿

### Step 3: 產出 PRD

1. 根據所有 Clarify 資訊，產出完整的 PRD markdown
2. 儲存到 `drafts/<TICKET_ID>/prd.md`
3. 更新 `state.yml` 的 `prd_version` + 1
4. 展示 PRD 給使用者確認
5. 使用者確認後，更新 `state.yml` phase 為 `finalized`

### Step 4: 定稿與發布

PM 確認 PRD 後，根據 `spec_mode` 執行不同流程。

#### 4-A: 搬移到 published

不論 `spec_mode`，都執行搬移：

```
mv $SPEC_ROOT/drafts/<TICKET_ID>/ → $SPEC_ROOT/published/<TICKET_ID>/
```

更新 `published/<TICKET_ID>/state.yml`：`phase: published`

#### 4-B: 發布（依 spec_mode 分流）

##### 若 `spec_mode: submodule`

Spec repo 獨立於主 repo，PM **不碰主 repo**。

1. **在 spec repo 內**（`$SPEC_ROOT` 即 submodule 目錄）：
   - `git add .`
   - `git commit -m "feat: publish PRD for <TICKET_ID>"`
   - `git push origin main`
   
   > ⚠️ Spec repo 永遠在 main 上操作，不切 branch。

2. **更新 Jira**：
   - Comment：「PRD 已發布，請 Review」
   - Transition：狀態 → "Spec Review"（若可用）

3. **提示 Engineer**：
   ```
   ✅ PRD 已發布到 spec repo。
   工程師可執行 /dev.plan HTGO2-123 開始規劃。
   ```

4. **完成**。不建立主 repo branch，由 Engineer 在 `/dev.plan` 時自行建立。

##### 若 `spec_mode: local`

PRD 就在主 repo 裡，PM **必須建 feature branch** 才能 commit。

1. **🛡️ Branch 保護檢查（雙重確認）**
   - 讀取 `.speckit-jira.yml` 中的 `protected_branches` 清單
   - 預計建立的 branch 名稱：`feature/<TICKET_ID>-<slugified-title>`
   - **檢查 1**：目標 branch 不在 `protected_branches` 中
     - 若命中 → **立即中止**：
       ```
       ❌ Cannot push to protected branch!
       Protected branches: main, master, develop, integration, staging, production, release/*
       Aborting.
       ```
   - **檢查 2**：branch 名稱包含 `<TICKET_ID>`
     - 若不包含 → **中止**：
       ```
       ❌ Branch name must contain the ticket ID: <TICKET_ID>
       Aborting.
       ```
   - **檢查 3**：顯示確認訊息
     ```
     Will create and push branch: feature/HTGO2-123-login-feature
     Confirm? (y/N)
     ```
   - 使用者確認後才繼續

2. **建立 Branch 並推送**
   - `git checkout -b feature/<TICKET_ID>-<slugified-title>`
   - `git add specs/`
   - `git commit -m "feat: publish PRD for <TICKET_ID>"`
   - `git push -u origin feature/<TICKET_ID>-<slugified-title>`

3. **更新 Jira**：
   - Comment：PRD 發布 + Branch 名稱
   - Transition：狀態 → "Spec Review"（若可用）

4. **完成**。

##### 若 `spec_mode: external`

與 submodule 類似，PRD 在外部 repo。

1. 在外部 repo 中 commit + push
2. 更新 Jira
3. 完成。不建立主 repo branch。

---

## Jira 降級處理

若 Atlassian MCP Server 連線失敗或 tool 呼叫失敗：

```
⚠️ Unable to connect to Jira MCP Server.
Please provide the following manually:

1. Ticket title:
2. Ticket description:
3. Acceptance Criteria:
4. Additional info:

(Paste and press Enter to continue)
```

後續步驟照常進行，只是跳過 Jira 狀態更新和 comment 操作。

---

## 預期產出

### 所有模式
- `published/<TICKET_ID>/state.yml` — 狀態追蹤
- `published/<TICKET_ID>/jira-snapshot.md` — Jira 需求快照
- `published/<TICKET_ID>/clarify-log.md` — Clarify 問答記錄
- `published/<TICKET_ID>/prd.md` — PRD 完整文件
- Jira 更新（comment + 狀態 transition）

### 僅 local 模式
- 主 repo feature branch（`feature/<TICKET_ID>-<title>`）
