# Tasks: [FEATURE NAME]

**Ticket**: `<TICKET_ID>` — [Ticket Title]
**Prerequisites**: plan.md (required), prd.md (required)
**Created**: [DATE]

---

## Format: `[ID] [P?] [Story?] Description`

- **Checkbox**: `- [ ]` / `- [x]`
- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2)
- Include exact file paths in descriptions
- **Jira**: Sub-task ID (auto-populated after creation)

### Examples

- ✅ `- [ ] T001 Create project structure per plan`
- ✅ `- [ ] T005 [P] [US1] Create User model in src/models/user.py`
- ✅ `- [ ] T012 [P] [US1] Implement login endpoint — **Jira**: HTGO2-3341`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 [task description with file path]
- [ ] T002 [P] [task description with file path]

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [ ] T003 [task description with file path]
- [ ] T004 [P] [task description with file path]

**Checkpoint**: Foundation ready — user story implementation can begin

---

## Phase 3: User Story 1 — [Title] (Priority: P1) 🎯 MVP

**Goal**: [Brief description from prd.md User Story 1]

**Independent Test**: [How to verify this story works on its own]

### Implementation

- [ ] T005 [P] [US1] [task with file path]
- [ ] T006 [P] [US1] [task with file path]
- [ ] T007 [US1] [task with file path] (depends on T005, T006)

**Checkpoint**: User Story 1 fully functional and testable independently

---

## Phase 4: User Story 2 — [Title] (Priority: P2)

**Goal**: [Brief description from prd.md User Story 2]

**Independent Test**: [How to verify this story works on its own]

### Implementation

- [ ] T008 [P] [US2] [task with file path]
- [ ] T009 [US2] [task with file path]

**Checkpoint**: User Stories 1 AND 2 both work independently

---

[Add more user story phases as needed, following priority order]

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] TXXX [P] Documentation updates
- [ ] TXXX Code cleanup and refactoring
- [ ] TXXX Performance optimization

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup — BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational
  - Can proceed in parallel (if staffed) or sequentially (P1 → P2 → P3)
- **Polish (Final Phase)**: Depends on all desired user stories

### Within Each User Story

- Models before services
- Services before endpoints
- Core implementation before integration
- Story complete before moving to next priority

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Test User Story 1 independently
5. Deploy/demo if ready

### Incremental Delivery

1. Setup + Foundational → Foundation ready
2. Add User Story 1 → Test → Deploy (MVP!)
3. Add User Story 2 → Test → Deploy
4. Each story adds value without breaking previous stories

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- **Jira**: Sub-task IDs are auto-populated by `/specrity.dev.tasks`
