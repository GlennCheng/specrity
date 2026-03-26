---
description: "工程師讀取 PRD → 分析程式碼 → 偵測缺陷（迴圈） → 產出 plan.md 實作計畫（整合 analyze 一致性檢查）"
---

# /specrity.dev.plan <TICKET_ID>

## 設計理念

> **此流程專為工程師設計。技術決策在此階段進行。**
> RD 基於 PM 產出的 PRD 做技術分析與規劃。
> 整合原生 Specrity 的 `plan` + `analyze` 兩個指令：
> - **plan**：技術選型、架構設計、實作步驟
> - **analyze**：PRD ↔ plan 一致性檢查、覆蓋率驗證
> 若 PRD 有缺陷，**RD 主動回報 PM**，而非自行假設。

## 概述
此 workflow 協助工程師基於 PRD 產出技術實作計畫。AI 會分析現有程式碼庫，進行技術研究（`research.md`）、資料模型定義（`data-model.md`）、偵測 PRD 缺陷（支援跨 Session 持久化的迴圈），最終參照 `.specrity/templates/plan-template.md` 產出 `plan.md`。

## 前置條件
- PRD 已存在（由 `/specrity.pm.specify` 產出）

---

## 執行步驟

### Step 0: 載入專案設定

#### 0-A: 開場與初步上下文檢查 (Initialization & Context Check)
1. **印出歡迎橫幅**：一旦收到此指令，**第一句話**必須使用 Markdown 的 `text` 區塊，印出以下 ASCII Banner：
   ```text
      ____                  _ __       
     / __/___  ___ ________(_) /___ __ 
    _\ \/ _ \/ -_) __/ __/ / __/ // /  
   /___/ .__/\__/\__/_/ /_/\__/\_, /   
      /_/                     /___/    
   ```
2. **上下文檢測 (Context Sanity Check)**：
   - AI 掃描目前對話視窗 (Session) 的對話記錄。
   - 若發現記憶體中混雜了「其他 Ticket 或不相干任務」的上下文，必須暫停並警告：
     `⚠️ [Specrity 警告] 偵測到當前會話中包含其他任務的上下文，可能會導致生成的內容錯亂。建議另開全新的 Chat。但若確認是相關任務，可回覆繼續。後續取得 PRD 內容後，我會再次進行深度分析。`
   - 等待使用者確認。若上下文乾淨，則印出 `✅ Context check passed.` 並繼續。

#### 0-B: 版本檢查
1. 讀取專案根目錄的 `.specrity-installed`
2. 若不存在 → 提示：`⚠️ 未偵測到 specrity 安裝紀錄。請先執行 ~/specrity/install.sh`
3. 若 `installed_at` 距今超過 30 天 → 提示：`💡 specrity workflow 已安裝 N 天（vX.X.X），建議更新`
4. 不阻擋後續步驟

#### 0-C: 載入設定
1. **強制讀取 `.specrity/.specrity.yml`**（設定優先權：`.specrity/.specrity.yml` → `.env` → 預設值）
2. 決定 `$SPEC_ROOT`（同 pm.specify 邏輯：必須確實解析 config，若為 submodule 需比對 `.gitmodules`，**絕對禁止猜測路徑**）
3. 從 `<TICKET_ID>` 解析 project key，透過 MCP 自動取得 cloudId（若未設定）

#### 0-D: 狀態門檻檢查（Status Gate）

> 確認 Jira Ticket 已到達適合開始 Plan 的狀態，避免 RD 在 PRD 尚未 Review 完時就開始技術分析。

1. 使用 MCP `getJiraIssue` 取得 Ticket 當前狀態
2. 讀取 `jira_transitions` 中的 `plan_start` 設定（如 `"Spec Review -> Planning"`）
3. 若 Ticket 當前狀態**早於** `plan_start` 的來源狀態（例如還在 "Specifying"）：
   ```
   ⚠️ 此 Ticket 目前狀態為「Specifying」，PRD 可能尚未通過 Review。
   建議等 Tech Lead 在 Confluence 審核 PRD 後再開始 Plan。
   (1) 仍然繼續（我確認過了）
   (2) 中止
   ```
4. 若 Ticket 狀態已到達或超過來源狀態 → 通過，印出 `✅ Status gate passed.`
5. 此檢查為「軟擋」：若使用者選擇繼續，照常執行後續步驟

### Step 1: 狀態偵測（State Detection）

檢查 `$SPEC_ROOT/plans/<TICKET_ID>/` 是否存在：

#### 情況 A：不存在 → NEW 模式
1. 尋找 PRD（**Jira Ticket → Confluence → 本地**）：
   > **[CRITICAL INSTRUCTION TO AI]**: RD 可能不在 PM 的同一台機器上，因此**不能依賴本地 `drafts/` 目錄**。必須優先從 Jira Ticket 找到 Confluence PRD 連結。

   **查找順序：**
   - **步驟 1：從 Jira Ticket 取得 Confluence PRD 連結**
     ```
     使用 MCP tool: getJiraIssue
     參數: cloudId = $JIRA_CLOUD_ID, issueIdOrKey = <TICKET_ID>, responseContentFormat = "markdown"
     ```
     掃描 Ticket 的以下位置尋找 Confluence PRD 連結：
     - (a) 自訂欄位（如 "PRD Link" 或 "Confluence Link"）
     - (b) Description 中的 Confluence URL（格式如 `https://*.atlassian.net/wiki/...`）
     - (c) 最近的 Comments 中由 `[Specrity]` 發布的 Confluence 連結
     
     若找到 Confluence URL → 從 URL 解析出 page ID → 使用 MCP 拉取 PRD 內容：
     ```
     使用 MCP tool: getConfluencePage
     參數: cloudId = $JIRA_CLOUD_ID, pageId = <解析出的 page_id>, contentFormat = "markdown"
     ```
   - **步驟 2（備援）：從本地 `state.yml` 取得**
     - 若 Jira 找不到 → 檢查 `$SPEC_ROOT/drafts/<TICKET_ID>/state.yml` 是否有 `confluence_page_id`
     - 若有 → 使用 MCP `getConfluencePage` 拉取
   - **步驟 3（備援）：本地 `prd.md`**
     - 若 Confluence 也拿不到 → 檢查本地 `drafts/<TICKET_ID>/prd.md`
   - **步驟 4（降級）：從 Jira Ticket description 萃取**
     - 若完全找不到 PRD → 提示使用者先執行 `/specrity.pm.specify <TICKET_ID>`

2. 若完全找不到 PRD，提示使用者先執行 `/specrity.pm.specify <TICKET_ID>`
3. 建立 `plans/<TICKET_ID>/` 目錄，初始化：
   ```
   plans/<TICKET_ID>/
   ├── state.yml           # phase: researching, analyze_rounds: 0, prd_source: confluence/local
   │                       # prd_confluence_version: <version>, prd_fetched_at: <timestamp>
   ├── analyze-log.md      # 缺陷偵測紀錄
   ├── research.md         # 技術研究紀錄（Step 2-B 產出）
   └── data-model.md       # 資料模型定義（Step 2-C 產出）
   ```
4. **強制執行 MCP Jira 更新**（在開始研究與分析之前）：
   > **[CRITICAL INSTRUCTION TO AI]**: 透過 MCP 執行 `addCommentToJiraIssue` 時，`commentBody` 參數 **MUST EXACTLY MATCH (必須完全一致，不可改寫)** 雙引號內的內容：
   ```
   使用 MCP tool: addCommentToJiraIssue
   參數: cloudId = $JIRA_CLOUD_ID, issueIdOrKey = <TICKET_ID>, commentBody = "🤖 [Specrity] 🛠️ 技術分析與實作計畫（Plan）階段已開始"
   ```

#### 情況 B：存在 + `phase: analyzing` → RESUME 模式
1. 讀取 `state.yml` → 知道上次分析到哪
2. 讀取 `analyze-log.md` → 載入過去的缺陷紀錄和 PM 回應
3. **偵測 PRD 版本變更**：
   - 若 `state.yml` 記錄了 `prd_confluence_version`：
     - 使用 MCP `getConfluencePage` 取得最新版本號
     - 若版本號不同 → 提示工程師：
       ```
       📄 PRD 在 Confluence 上已被更新（v{old} → v{new}）。
       建議重新讀取最新版 PRD 後繼續分析。
       (1) 重新讀取最新 PRD 並繼續
       (2) 使用原有版本繼續（不推薦）
       ```
   - 若無版本紀錄，比對本地 PRD 是否已更新（與上次快照比對）
4. 顯示恢復摘要：
   ```
   📋 恢復 HTGO2-123 的計畫流程
   - 已進行 N 輪 Analyze
   - 上次發現 M 個缺陷
   - PRD 自上次已更新 ✅ / 未更新 ⏳
   ```

#### 情況 C：存在 + `phase: completed` → 已完成
1. 提示 plan.md 已存在
2. 詢問：(1) 查看現有 plan  (2) 重新分析（覆蓋）

### Step 2: 讀取 PRD + 技術研究

#### 2-A: 讀取 PRD 與程式碼分析
1. 讀取 PRD 完整內容
2. 從 Jira 取得最新 Ticket 資訊（MCP tool: `getJiraIssue`）
3. **深度上下文檢測 (Deep Context Sanity Check)**：
   - AI 必須對比「剛剛載入的 PRD / Ticket 內容」與「先前的對話歷史記錄」。
   - 如果發現歷史對話中的任務需求、架構討論，**與本次 PRD 完全無關甚至衝突**（AI 判斷將導致嚴重的幻覺或資訊混亂），必須中斷流程並拒絕推進：
     `❌ [Specrity 錯誤] 深度分析發現歷史對話與本次 PRD 任務存在明顯衝突。強烈建議您開啟一個「全新乾淨的 Chat」後重新執行指令，以避免實作計畫失真。`
   - 除非使用者強行命令略過，否則停留在此階段。
4. 比對 PRD 與 Jira 一致性，若有差異則提醒
4. 掃描專案結構，識別相關模組
5. 找出會被影響的檔案和函式
6. 識別可能的技術風險和依賴項

#### 2-B: 技術研究（Research）

對於 PRD 涉及的技術選型，進行研究並記錄決策：

1. 識別需要決策的技術問題（框架、API 設計、資料流等）
2. 為每個問題列出選項、優缺點、推薦選擇
3. 產出 `research.md`：
   ```markdown
   # Technical Research: <feature name>

   ## Decision 1: [topic]
   **Context**: [why this decision is needed]
   **Options**:
   - A: [option] — pros / cons
   - B: [option] — pros / cons
   **Decision**: [chosen option]
   **Rationale**: [why]
   ```
4. 更新 `state.yml`：`phase: modeling`

#### 2-C: 資料模型定義（Data Model）

若此 feature 涉及資料變更：

1. 從 PRD requirements 提取實體（entities）
2. 定義欄位、關聯、約束
3. 產出 `data-model.md`：
   ```markdown
   # Data Model: <feature name>

   ## Entities
   ### [Entity Name]
   - field_1: type — description
   - field_2: type — description
   
   ## Relationships
   - [Entity A] 1:N [Entity B]

   ## State Transitions (if applicable)
   [State A] -> [Event] -> [State B]
   ```
4. 更新 `state.yml`：`phase: analyzing`

若此 feature 不涉及資料變更，跳過此步驟，`data-model.md` 可留空或標記 N/A。

### Step 3: 缺陷偵測 + 一致性檢查（Analyze Loop）

整合原生 Specrity 的 `analyze` 指令。類似 `/specrity.pm.specify` 的 Clarify 迴圈，支援跨 Session 持久化。

#### 偵測項目（PRD 缺陷與待評估事項）
- 缺少接受標準（Acceptance Criteria）
- 邏輯矛盾（功能 A 和功能 B 衝突）
- 技術不可行（現有架構不支持）
- 缺少邊界條件（大量資料、併發、錯誤處理）
- 安全性考量缺失（權限、資料保護）
- API 規格不完整（缺少 request/response 定義）
- **[NEEDS RCA] 標記**：若 PRD 有 `[NEEDS RCA]` 標記（如：需評估解法建議是否合適），需進行技術分析並決定最佳解法

#### 一致性檢查（整合自原生 analyze）

| 檢查維度 | 說明 |
|---------|------|
| **Coverage** | 每個 FR 是否都有對應的實作步驟？ |
| **術語一致性** | PRD vs research vs data-model 用詞是否統一？ |
| **矛盾偵測** | PRD 和技術研究有無衝突？ |
| **模糊偵測** | plan 中有無未量化的模糊詞（「快速」、「穩定」）？ |
| **依賴分析** | 外部服務/API 的失敗模式是否考慮？ |

#### 迴圈流程
```
偵測缺陷 → 有缺陷？
             │
             ├── 否 → 跳到 Step 4（產出 plan.md）
             │
             └── 是 → 整理缺陷清單
                        │
                        ▼
                  寫入 analyze-log.md（持久化！）
                        │
                        ▼
                  詢問工程師：
                  (1) 僅標註在 plan 中（不阻擋，繼續產 plan）
                  (2) 回報 PM 並等待修改（進入等待狀態）
                        │
                  ┌─────┴──────┐
                  ▼            ▼
            標註在 plan    Jira Comment 回報 PM
            → Step 4       → 更新 state.yml
                             phase: waiting_pm
                           → 提示工程師：
                             「已回報 PM，PM 修改 PRD 後
                              請重新執行 /specrity.dev.plan HTGO2-123」
                           → 結束（等 PM 改完再 Resume）

（PM 修改 PRD 後，工程師重新執行 /specrity.dev.plan）
  → 進入 RESUME 模式
  → 偵測到 PRD 已更新
  → 重新分析（Analyze Round 2）
  → 確認缺陷已修復 → Step 4
```

#### 持久化檔案
```
plans/<TICKET_ID>/
├── state.yml
│     phase: analyzing | waiting_pm | completed
│     analyze_rounds: 2
│     last_prd_hash: abc123
│     defects_found: 3
│     defects_resolved: 2
│
├── analyze-log.md
│     ## Round 1 (2026-03-03)
│     ### 發現的缺陷
│     1. ⚠️ 缺少並發處理的邊界條件
│     2. ⚠️ API response 格式未定義
│     ### 處理方式：回報 PM
│     ### PM 回應：已更新 PRD（2026-03-04）
│
│     ## Round 2 (2026-03-05)
│     ### 重新分析
│     1. ✅ 並發處理已補充
│     2. ✅ API response 已定義
│     ### 結果：所有缺陷已解決
│
└── plan.md  ← Step 4 完成後才產出
```

### Step 4: 產出 plan.md

1. **必須實際讀取 Template 檔案**：使用檔案讀取工具，去讀取 `$PROJECT_ROOT/.specrity/.specrity/templates/plan-template.md` 的完整內容，作為結構參照
2. 基於 PRD、research.md、data-model.md、程式碼分析、和缺陷偵測結果，填充 template
3. **將 research.md 和 data-model.md 的內容合併進 plan.md**：
   - `research.md` 的技術決策整合為 plan 的「Technical Decisions」區塊
   - `data-model.md` 的資料模型整合為 plan 的「Data Model」區塊
   - 這些內容發布 Confluence 時會一起推上去（見 Step 6）
4. 確保 plan 中的每個實作步驟都能追溯到 PRD 的 FR
5. 儲存 `plan.md` 到 `plans/<TICKET_ID>/plan.md`
6. 更新 `state.yml`：`phase: completed`
7. 展示 plan.md 給工程師確認

### Step 5: 更新 Jira
1. 在 Jira 加上 comment：「Implementation Plan 已建立」
2. **狀態轉移 (Jira Status Transition)**：
   > **[CRITICAL INSTRUCTION TO AI]**: 根據讀取到的 `jira_transitions` 中的 `plan_done` 設定（格式如 `"Planning -> Ready for Dev"`）。若本 Ticket **當前狀態**與設定的**來源狀態**一致，請使用 MCP 找出對應 `id` 並呼叫 `transitionJiraIssue` 完成狀態拖拉。若不吻合或未設定則略過。
3. 告知工程師結果

### Step 6: 發布 Plan 到 Confluence

> 將 plan.md（含 research + data-model 內容）發布到 Confluence，作為 PRD 頁面的子頁面。

#### 6-A: Confluence Review Comment 檢查

> 發布前掃描 PRD 的 Confluence 頁面，確認 Review Comment 都已處理。

1. 若已知 PRD 的 `confluence_prd_page_id`，使用 MCP `getConfluencePageInlineComments`：
   ```
   使用 MCP tool: getConfluencePageInlineComments
   參數: cloudId = $JIRA_CLOUD_ID, pageId = $confluence_prd_page_id, resolutionStatus = "open"
   ```
2. 若有未解決的 comment（`open` 狀態）→ 展示警告：
   ```
   ⚠️ PRD 的 Confluence 頁面上有 N 個未解決的 Review Comment：
   1. @{author}: "{摘要}"
   2. @{author}: "{摘要}"

   建議先解決這些 Comment 再繼續發布 Plan。
   (1) 我已經處理了，繼續
   (2) 中止
   ```
3. 若無未解決 comment → 通過，繼續發布

#### 6-B: 建立或更新 Confluence 頁面

1. **取得 PRD 的 Confluence page ID**：
   - 從 Step 1 取得 PRD 時已知 `confluence_prd_page_id`（從 Jira Ticket 或本地 state.yml）
   - 若找不到 PRD 的 Confluence page ID（PRD 未發布到 Confluence），則使用 `$CONFLUENCE_PARENT_PAGE_ID` 作為父頁面

2. **建立或更新 Confluence 頁面**：
   - 檢查 `plans/<TICKET_ID>/state.yml` 是否已有 `confluence_plan_page_id`
   - **若無（首次發布）** → 使用 MCP `createConfluencePage`：
     ```
     使用 MCP tool: createConfluencePage
     參數:
       cloudId = $JIRA_CLOUD_ID
       spaceId = $CONFLUENCE_SPACE_ID
       parentId = $confluence_prd_page_id（掛在 PRD 子頁面下）
       title = "[<TICKET_ID>] <Ticket Title> — Plan"
       body = plan.md 完整內容（含 research + data-model）
       contentFormat = "markdown"
     ```
   - **若有（更新發布）** → 使用 MCP `updateConfluencePage`：
     ```
     使用 MCP tool: updateConfluencePage
     參數:
       cloudId = $JIRA_CLOUD_ID
       pageId = $confluence_plan_page_id
       body = plan.md 完整內容
       contentFormat = "markdown"
       versionMessage = "Updated via Specrity"
     ```
   > 此更新邏輯確保 research / data-model 在 Analyze 迴圈中變動後，重新產出 plan.md 時 Confluence 也會同步更新。

3. **記錄到 `plans/<TICKET_ID>/state.yml`**：
   ```yaml
   confluence_plan_page_id: <page_id>
   confluence_plan_url: <page_url>
   confluence_plan_last_version: <version.number>
   ```

4. **更新 Jira Ticket**：
   - 使用 MCP `editJiraIssue` 或 `addCommentToJiraIssue`，將 Plan 的 Confluence 連結寫入 Ticket

5. **智慧 Reviewer 通知（Tech Lead @mention）**：
   > 邏輯同 pm.specify Step 4-C：
   - 情況 A：Ticket 有 `review_label` → 自動 @mention `reviewer_account_id`
   - 情況 B：Ticket 無 label → AI 評估 Plan 複雜度（架構變更、跨系統影響、新技術引入等），若建議 Review → 提示工程師是否通知主管
   - 若未設定 `reviewer_account_id` → 跳過

6. **若 Confluence 連線失敗**：不阻擋流程，plan.md 已在本地完成。提示工程師稍後重新執行時會再嘗試發布。

---

## Jira 降級處理

若 Atlassian MCP Server 連線失敗：

```
⚠️ 無法連接 Jira MCP Server。
請手動提供 PRD 內容：

（貼上 PRD markdown 後按 Enter 繼續）
```

跳過 Jira 更新步驟，僅產出 plan.md。

---

## 預期產出

```
plans/<TICKET_ID>/
├── state.yml           # 狀態追蹤（含 Confluence page ID）
├── analyze-log.md      # 缺陷偵測紀錄（跨 Session 持久化，僅本地）
├── research.md         # 技術研究紀錄（內容合併進 plan.md）
├── data-model.md       # 資料模型定義（內容合併進 plan.md）
├── contracts/          # API 規格（選擇性）
└── plan.md             # 技術實作計畫（含 research + data-model，參照 plan-template）
```

- **Confluence 頁面**：`[TICKET_ID] Title — Plan`（掛在 PRD 頁面底下）
- Jira comment + Confluence 連結（若 MCP 可用）
- PRD 缺陷報告（若有發現）

