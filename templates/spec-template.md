# Feature Specification: [FEATURE NAME]

**Ticket**: `<TICKET_ID>` — [Ticket Title]
**Created**: [DATE]
**Status**: Draft
**Author**: [PM Name]

---

## Story Background *(mandatory)*

<!--
  WHY are we doing this? What problem are we solving?
  Write for business stakeholders, not developers.
-->

### Problem Statement

[Describe the current pain point or opportunity. Be specific about who is affected and how.]

### Business Goal

[What does success look like from a business perspective?]

### Current State

[How do users currently handle this? What is the workaround?]

---

## User Scenarios & Testing *(mandatory)*

<!--
  User stories should be PRIORITIZED as user journeys ordered by importance.
  Each user story must be INDEPENDENTLY TESTABLE.
  Assign priorities (P1, P2, P3) where P1 is the most critical.
-->

### User Story 1 — [Brief Title] (Priority: P1) 🎯 MVP

[Describe this user journey in plain language]

**Why this priority**: [Explain the value and why it has this priority level]

**Independent Test**: [How to verify this story works on its own]

**Acceptance Scenarios**:

1. **Given** [initial state], **When** [action], **Then** [expected outcome]
2. **Given** [initial state], **When** [action], **Then** [expected outcome]

---

### User Story 2 — [Brief Title] (Priority: P2)

[Describe this user journey in plain language]

**Why this priority**: [Explain the value and why it has this priority level]

**Independent Test**: [How to verify this story works on its own]

**Acceptance Scenarios**:

1. **Given** [initial state], **When** [action], **Then** [expected outcome]

---

### User Story 3 — [Brief Title] (Priority: P3)

[Describe this user journey in plain language]

**Why this priority**: [Explain the value and why it has this priority level]

**Independent Test**: [How to verify this story works on its own]

**Acceptance Scenarios**:

1. **Given** [initial state], **When** [action], **Then** [expected outcome]

---

[Add more user stories as needed, each with an assigned priority]

### Edge Cases

<!--
  Boundary conditions, error scenarios, unexpected user behavior.
  Focus on WHAT should happen, not HOW to implement it.
-->

- What happens when [boundary condition]?
- How should the system respond when [error scenario]?
- What does the user see if [unexpected situation]?

---

## Scope *(mandatory)*

### In Scope

- [Feature/behavior that IS included]
- [Feature/behavior that IS included]

### Out of Scope

- [Feature/behavior explicitly NOT included in this ticket]
- [Feature/behavior explicitly NOT included in this ticket]

### Phase Boundaries

- **Phase 1 (MVP)**: [Minimum viable scope]
- **Phase 2**: [Next iteration scope]

---

## Requirements *(mandatory)*

<!--
  Focus on WHAT, not HOW. No tech stack, APIs, or code structure.
  Written for business stakeholders, not developers.
  Each requirement must be testable.
-->

### Functional Requirements

- **FR-001**: System MUST [specific capability from user perspective]
- **FR-002**: Users MUST be able to [key interaction]
- **FR-003**: System MUST [specific behavior]

*For unclear requirements, use the [NEEDS CLARIFICATION] marker:*

- **FR-004**: System MUST [capability] [NEEDS CLARIFICATION: specific question about what's unclear]

### Key Entities *(include if feature involves data)*

- **[Entity 1]**: [What it represents, key attributes — without implementation details]
- **[Entity 2]**: [What it represents, relationships to other entities]

---

## Experience Expectations *(optional — include when relevant)*

<!--
  Define EXPECTATIONS, not technical solutions.
  Example: "Must feel instant" NOT "Use WebSockets"
-->

- **Loading experience**: [e.g., "Must feel instant" / "Can show loading indicator"]
- **Responsiveness**: [e.g., "Must work on mobile and desktop"]
- **Accessibility**: [e.g., "Must support keyboard navigation"]
- **Offline behavior**: [e.g., "Should show cached content when offline"]

---

## Success Criteria *(mandatory)*

<!--
  Technology-agnostic, measurable, user-focused.
  No mention of frameworks, databases, or tools.
-->

### Measurable Outcomes

- **SC-001**: [User-facing metric, e.g., "Users can complete the task in under 2 minutes"]
- **SC-002**: [Performance metric, e.g., "Page loads in under 3 seconds on 3G"]
- **SC-003**: [Business metric, e.g., "Reduce support tickets about X by 50%"]
- **SC-004**: [Adoption metric, e.g., "90% of users complete the flow on first attempt"]

---

## Clarifications *(auto-populated by /pm.specify)*

<!--
  This section is automatically populated during the Clarify loop.
  Each session records questions asked and answers received.
-->

### Session [DATE]

- Q: [question] → A: [answer]
- Q: [question] → A: [answer]

---

## Assumptions *(optional)*

<!--
  Reasonable defaults that were assumed during spec writing.
  Document them here so engineers know what was assumed vs. confirmed.
-->

- [Assumption 1, e.g., "Existing login flow remains unchanged"]
- [Assumption 2, e.g., "Data volume is under 10,000 records"]

---

## AI Generation Guidelines

When creating this spec from a user prompt:

1. **Make informed guesses**: Use context, industry standards, and common patterns to fill gaps
2. **Document assumptions**: Record reasonable defaults in the Assumptions section
3. **Limit clarifications**: Maximum 3 `[NEEDS CLARIFICATION]` markers — only for critical decisions that:
   - Significantly impact feature scope or user experience
   - Have multiple reasonable interpretations with different implications
   - Lack any reasonable default
4. **Prioritize clarifications**: scope > user experience > success metrics > edge cases
5. **Think like a PM**: Every requirement should answer "what value does this deliver to the user?"
6. **No technical jargon**: Avoid mentioning frameworks, APIs, databases, or implementation patterns

**Examples of reasonable defaults** (don't ask about these):

- Data retention: Industry-standard practices for the domain
- Performance targets: Standard web/mobile app expectations unless specified
- Error handling: User-friendly messages with appropriate fallbacks
- Authentication: Follow existing project patterns
