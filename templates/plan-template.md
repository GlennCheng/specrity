# Implementation Plan: [FEATURE NAME]

**Ticket**: `<TICKET_ID>` — [Ticket Title]
**Branch**: `feature/<TICKET_ID>-<short-name>`
**Created**: [DATE]
**PRD**: `published/<TICKET_ID>/prd.md`

---

## 需求摘要

[從 PRD 摘要核心需求，2-3 句]

## Technical Context

**Language/Version**: [e.g., PHP 8.1, TypeScript 5.x]
**Primary Dependencies**: [e.g., Laravel, React, Next.js]
**Storage**: [e.g., MySQL, Redis, Elasticsearch]
**Testing**: [e.g., PHPUnit, Jest, Cypress]
**Target Platform**: [e.g., Web (Desktop + Mobile)]
**Project Type**: [e.g., web-service, library, cli, mobile-app]

## 技術分析

### 影響範圍

- [affected file/module 1]
- [affected file/module 2]

### 技術風險

- [risk 1: description + mitigation]
- [risk 2: description + mitigation]

### 依賴項

- [dependency 1]
- [dependency 2]

---

## Research Summary

<!--
  由 /specrity.dev.plan Phase 0 產出。
  記錄技術選型的決策和理由。
  完整內容在 research.md。
-->

| 決策 | 選擇 | 理由 | 替代方案 |
|------|------|------|---------|
| [decision 1] | [chosen] | [why] | [alternatives] |
| [decision 2] | [chosen] | [why] | [alternatives] |

## Data Model

<!--
  由 /specrity.dev.plan Phase 1 產出。
  定義此 feature 涉及的資料實體。
  完整內容在 data-model.md。
-->

### Key Entities

- **[Entity 1]**: [fields, relationships, constraints]
- **[Entity 2]**: [fields, relationships, constraints]

### State Transitions (if applicable)

```
[State A] → [Event] → [State B] → [Event] → [State C]
```

---

## Project Structure

### Documentation (this feature)

```text
specs/<TICKET_ID>/
├── prd.md               # PRD (from /specrity.pm.specify)
├── plan.md              # This file (from /specrity.dev.plan)
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── contracts/           # Phase 1 output (optional)
├── tasks.md             # From /specrity.dev.tasks
└── analyze-log.md       # Analyze loop records
```

### Source Code Changes

```text
[Project-specific structure — fill based on analysis]
```

---

## 實作步驟

### Phase 1: [Phase Name]

- [ ] [task description with file path]
- [ ] [task description with file path]

### Phase 2: [Phase Name]

- [ ] [task description with file path]
- [ ] [task description with file path]

---

## PRD 缺陷紀錄

<!--
  由 Analyze Loop 產出。
  記錄分析過程中發現的 PRD 問題和解決狀態。
-->

| 缺陷 | 狀態 | 處理方式 |
|------|------|---------|
| [defect 1] | ✅ 已解決 | [PM 已修改 PRD] |
| [defect 2] | ⚠️ 標註 | [不阻擋，實作時注意] |

## 驗證計畫

- [verification step 1]
- [verification step 2]

---

## AI Generation Guidelines

When creating this plan from PRD + codebase analysis:

1. **Technical Context**: Fill from actual project analysis, mark unknowns as `NEEDS CLARIFICATION`
2. **Research**: For each unknown, research and document decision rationale
3. **Data Model**: Extract entities from PRD requirements, define fields and relationships
4. **Impact Analysis**: Scan actual codebase to identify affected files
5. **Risk Assessment**: Identify technical risks based on existing architecture
6. **Phase Planning**: Break implementation into logical phases with clear dependencies
