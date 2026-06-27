---
name: story-grooming
description: |
  The `story-grooming` skill refines Tracker stories into development-ready tasks. It assesses sprint fit, breaks oversized stories, decomposes into 2-day tasks, designates work type (agent/human/needs-info), and generates briefs.
---
# story-grooming: Implementation Guide

This guide documents the complete workflow, decision points, and technical details for implementing the `story-grooming` skill.

---

## Overview

The `story-grooming` skill refines Tracker stories into development-ready tasks. It assesses sprint fit, breaks oversized stories, decomposes into 2-day tasks, designates work type (agent/human/needs-info), and generates briefs.

**Process**: Story → (optionally Sub-stories) → 2-day Tasks → Agent/Human designation → Tracker Sub-tasks

---

## Part 1: Story Discovery & Fetching

### Input Methods

The skill supports three invocation patterns:

1. **Single Story**: `"Groom story CARD-101"`
   - User provides Tracker story key
   - Skill fetches story from Tracker
   - Proceeds to grooming

2. **Epic**: `"Groom epic CARD-100"`
   - User provides Tracker epic key
   - Skill queries Tracker for all stories in epic
   - Asks user: "Groom all X stories, or pick one?"
   - Proceeds with selected story(ies)

3. **Discover**: `"Find stories to groom"`
   - Skill queries Tracker for all issues with `backlog` label
   - Lists stories: "Found N stories to groom. Which one?"
   - User selects one
   - Proceeds to grooming

### Fetching Story Data

For each story, fetch from Tracker:
- **Story key** (e.g., "CARD-101")
- **Summary** (e.g., "Challenge Question Validation via Callback")
- **Description** (business context, requirements)
- **Acceptance Criteria** (from story description or custom field)
- **Parent Epic** (if sub-story)
- **Labels** (existing labels)
- **System references** (look in description for system names)

**Validation**:
- ❌ If story has no acceptance criteria, inform user and ask them to add AC or mark story as `needs-info`
- ❌ If story key doesn't exist, inform user and ask to verify key
- ✅ If story exists with AC, proceed

---

## Part 1.5: Glossary Validation

### Check Story Against Glossary

Before grooming, validate that the story uses canonical domain terms from `GLOSSARY.md`.

**Step 1: Load Glossary**
- Check for `GLOSSARY.md` at repo root
- Check for context-specific glossaries (e.g., `src/ordering/GLOSSARY.md`)
- Parse all terms and definitions

**Step 2: Scan Story Text**
- Check story summary, description, and acceptance criteria
- Identify domain terms (nouns that name business concepts)
- Compare against glossary entries

**Step 3: Report Issues**

```
Skill: 📖 **Glossary Check for CARD-101**
       
       ✅ **Terms used correctly:**
       - Cancellation (3 uses) - matches glossary
       - Refund (2 uses) - matches glossary
       
       ⚠️ **Potential conflicts:**
       - "Status" on line 5: glossary defines as "Order state" 
         but context suggests "Payment state"
       
       ❓ **Undefined terms (consider adding to glossary):**
       - "Chargeback" (used 2x) - not in glossary
       - "Void" (used 1x) - not in glossary
       
       Should we resolve these before grooming?
```

### Challenge Against the Glossary

When a term conflicts with the glossary definition, surface it immediately:

```
Skill: "⚠️ Your glossary defines **Cancellation** as 'voiding a paid order.'
       The story uses it to mean 'removing an item from the cart.'
       
       Which is it? Should we:
       1. Use existing definition (update story to say 'Cart Removal')
       2. Add new term 'Cart Removal' to glossary
       3. Update glossary definition of Cancellation"
```

**Don't proceed with grooming until term conflicts are resolved** - they will cause confusion in tasks and agent briefs.

### Adding New Terms

When a new domain term is identified:

```
Skill: "Chargeback" isn't in your glossary.
       
       Suggested definition based on context:
       "When a customer disputes a charge with their bank, resulting in 
       a forced reversal of the transaction."
       
       Accept this definition, or provide your own?

User: Accept

Skill: ✅ Added to GLOSSARY.md:
       
       ## Chargeback
       **Definition:** When a customer disputes a charge with their bank, 
       resulting in a forced reversal of the transaction.
       **Not to be confused with:** Cancellation (voluntary), Refund (merchant-initiated)
```

### Glossary Entry Format

```markdown
## <Term>

**Definition:** <one-sentence definition meaningful to a domain expert>

**Examples:** <concrete example, optional>

**Not to be confused with:** <neighboring term, optional>

**See also:** <related term, optional>
```

**Rules:**
1. Domain terms only - not implementation details (class names, table names)
2. Nouns and noun phrases - concepts, not actions
3. Alphabetical order within the file
4. Track renames in "Not to be confused with"

---

## Part 2: Story Point Validation

### Check for Missing Story Points

Before grooming, verify the story has story points:

```javascript
const storyPoints = await get_story_points({ issueKey: "CARD-101" });

if (storyPoints.storyPoints === null) {
  // Story needs estimation first
  console.log("Story has no story points. Invoking estimate-story...");
}
```

**If story lacks points**:
```
Skill: "Story CARD-101 has no story points estimated.
        Should I estimate it first?"
User: "Yes"
Skill: [invokes estimate-story skill]
```

Or invoke `estimate-story` directly:
```text
Estimate story CARD-101
```

> **Recommended**: Ensure all stories have story points before grooming.
> See `skills/estimate-story/SKILL.md` for estimation guidelines.

---

## Part 3: Sprint Fit Assessment

### Heuristic for Sprint Fit

A story **fits in a 2-week sprint** if ALL of these are true:

| Criteria | Threshold | Flag if |
|----------|-----------|---------|
| **Acceptance Criteria Count** | ≤6 AC | >6 AC |
| **Systems Touched** | ≤3 systems | >3 systems |
| **Complexity Estimate** | S or M | L or XL |

**Example**:
```
Story CARD-101: Challenge Question Validation
- AC count: 8 ❌ (too many)
- Systems: Acquisitions, Coppice, Conversation Platform, Fiserv (4) ❌
- Estimate: Large ❌

Verdict: TOO BIG FOR SPRINT
```

### Determining Systems Touched

Ask user or infer from description:
- "Which systems/services does this story touch?"
- User responds: "acquisitions-platform, coppice, fiserv"
- Count: 3 systems ✓

If user doesn't know, parse story description:
- Look for system names mentioned in AC
- Look for API calls or service references
- Count inferred systems
- Ask user: "Did I miss any systems?"

### User Confirmation

**If story fits**:
```
Skill: "Story CARD-101 fits in a 2-week sprint (4 AC, 2 systems, Medium complexity).
        Ready to break into tasks?"
User: "Yes"
```

**If story doesn't fit**:
```
Skill: "Story CARD-101 is too big for a 2-week sprint.
        Should I split it into sub-stories?"
User: "Yes" or "No"
```

---

## Part 4: Sub-Story Creation (if needed)

### Proposing Sub-Stories

If story doesn't fit sprint, propose a breakdown:

**Strategy**: Group by system or feature area

**Example for CARD-101 (Challenge Question Validation)**:
```
Story is too large. Proposing split:

CARD-101-A: Challenge Q&A Creation & Encryption
  - Systems: acquisitions-platform, postgres
  - AC count: 3
  - Size: Medium (fits sprint)

CARD-101-B: Challenge Question Validation API
  - Systems: acquisitions-platform, coppice
  - AC count: 2
  - Size: Small (fits sprint)

CARD-101-C: SMS Integration & Delivery
  - Systems: communication-platform, salesforce
  - AC count: 2
  - Size: Small (fits sprint)

CARD-101-D: Fiserv Wallet Integration
  - Systems: acquisitions-platform, fiserv
  - AC count: 2
  - Size: Medium (fits sprint)

Does this breakdown make sense?
```

### Creating Sub-Stories in Tracker

For each sub-story:
- **Issue Type**: Story
- **Parent**: Link to parent story (CARD-101)
- **Summary**: `[Parent]-[Letter]: [Description]` (e.g., "CARD-101-A: Challenge Q&A Creation & Encryption")
- **Description**: Include:
  - Parent story link
  - Systems touched
  - Subset of original AC (relevant to this sub-story)
  - Any constraints or notes
- **Labels**: `groomed` (ready for triage and assignment)
- **Status**: "To Do"

### User Decides Next Step

**Option 1**: "Groom these sub-stories now"
- Skill proceeds to groom first sub-story (e.g., CARD-101-A)
- Repeats assessment → task breakdown → output

**Option 2**: "Groom sub-stories later"
- Skill saves metadata (sub-story keys, mapping)
- User can invoke skill later: "Groom story CARD-101-A"
- Skill picks up where it left off

---

## Part 5: Identifying Codebases

### Ask the User

```
Skill: "Which codebases/modules does this story touch?
        Examples: acquisitions-platform, communication-service, transaction-web"
User: "acquisitions-platform (Node.js backend)"
```

### Parse Story Description

Also scan story description for codebase references:
- Look for file paths or module names mentioned in AC
- Look for API endpoints or service calls
- Extract system/codebase names
- Cross-reference with user's response

**Confirm with user**: "Did I capture all codebases?"

### Map to Layers

For each codebase, identify layers:
- **Database/Schema** (migrations, ORM models)
- **API/Backend Logic** (endpoints, business logic)
- **UI/Frontend** (components, forms, pages)
- **Integration/Testing** (integration tests, end-to-end tests)

**Example**:
```
Codebase: acquisitions-platform (Node.js)
Layers:
  - Database: PostgreSQL migrations + Sequelize models
  - API: Express.js endpoints (REST)
  - Tests: Jest (unit + integration)
  (No UI: this is backend-only)
```

---

## Part 6: Breaking Into 2-Day Tasks

### Task Slicing Strategy

Break story into **thin, testable, layer-based tasks**.

**Rule**: Each task = ~2 days of work = 1 Tracker Sub-task

**Approach**: For each codebase, slice by layer:

```
CARD-101-A: Challenge Q&A Creation & Encryption

Task 1: Database Schema
  - Create challenge_qa table
  - Add encryption, indexing, constraints
  - Database migration

Task 2: Generation & Validation API
  - Create POST /challenge-qa endpoint
  - Implement encrypt/decrypt logic
  - Add input validation

Task 3: Unit & Integration Tests
  - Test schema creation
  - Test API endpoints
  - Test encryption/decryption

Task 4: Documentation & Deployment
  - Document schema design & API contract
  - Prepare deployment script
  - Update runbooks
```

### Task Definition Template

For each task, document:

| Field | Description |
|-------|-------------|
| **Title** | `Task N: [Layer] - [Specific Work]` |
| **Parent Story** | Link to parent (e.g., CARD-101-A) |
| **What to build** | 2-3 sentence description (end-to-end) |
| **Systems involved** | Which codebases/services |
| **Acceptance Criteria** | Specific, testable checklist |
| **Codebase references** | Which modules/directories |
| **Definition of Done** | Code review, tests, docs, merge |
| **Estimated size** | ~2 days |

### Example Task: Challenge Q&A Schema

```markdown
## Task 1: Database Schema - Challenge Q&A Table

**Parent Story**: CARD-101-A (Challenge Q&A Creation & Encryption)
**Parent Story Key**: CARD-101-A

### What to build

Create a PostgreSQL table for storing challenge questions and encrypted answers.
Implement encryption at rest (AES-256), add indexes for query performance, and
include validation constraints. Create a migration file with rollback support.

### Acceptance Criteria

- [ ] Create `challenge_qa` table with schema:
  - id (UUID primary key)
  - applicant_id (foreign key to applicants)
  - question_type (enum: SSN_LAST_4, DOB)
  - answer_encrypted (TEXT, AES-256 encrypted)
  - created_at (timestamp, default now)
  - expires_at (timestamp, index for cleanup)
  - attempt_count (int, default 0)
  - locked (boolean, default false)

- [ ] Add constraints:
  - NOT NULL on: id, applicant_id, question_type, answer_encrypted, created_at
  - UNIQUE on: (applicant_id, question_type) per expiry window
  - CHECK: attempt_count >= 0

- [ ] Add indexes:
  - (applicant_id, expires_at) for query optimization
  - (applicant_id, created_at) for audit trails

- [ ] Implement encryption:
  - Use crypto module (Node.js) with AES-256-GCM
  - Store encryption key securely (env var or vault)
  - Include IV (initialization vector) in encrypted blob

- [ ] Create migration file:
  - File: `migrations/YYYYMMDD_create_challenge_qa.js`
  - Include up() and down() functions for rollback

- [ ] Unit tests:
  - Test table creation
  - Test encryption/decryption roundtrip
  - Test constraint enforcement
  - Test index creation

- [ ] Documentation:
  - Schema design doc (why these fields, encryption approach)
  - Migration documentation

- [ ] Definition of Done:
  - Code reviewed and approved
  - All tests passing (>80% coverage)
  - Migration tested in staging
  - Docs updated and verified
  - Ready to merge to main branch

### Notes

**Codebase**: acquisitions-platform (Node.js + Sequelize ORM)
**Directories**: `src/db/migrations/`, `src/models/`, `src/services/encryption/`
**Dependencies**: `crypto` (Node.js built-in), `pg` (PostgreSQL driver)

**Related AC from parent story**:
- Requirement 1.a: "Acquisitions generates challenge Q&A...encrypted"
- Requirement 1.b: "...pass them to Coppice..."
```

---

## Part 7: Task Dependencies & Ordering

### Detecting Dependencies

Tasks have **implicit layer-based dependencies**:

```
Layer 1: Database Schema
    ↓ (must exist before)
Layer 2: API/Backend Logic
    ↓ (must exist before)
Layer 3: UI/Frontend
    ↓ (must exist before)
Layer 4: Integration/Testing
```

**Example for CARD-101-A**:
- Task 1 (Schema) has no dependencies → Can start immediately
- Task 2 (API) blocked by Task 1 → Needs schema to exist
- Task 3 (Tests) blocked by Task 2 → Needs API to test
- Task 4 (Docs) blocked by Task 3 → Needs tests to pass

### Identifying Parallelizable Tasks

Some tasks can be done in parallel if they don't depend on each other:

**Example**:
```
Task 1: Database Schema (Layer 1)
Task 1b: Unit tests for schema (can be written in parallel)
Task 2: API Endpoint (Layer 2, blocked by Task 1)
Task 3: UI Form (Layer 3, blocked by Task 2)
Task 4: Integration tests (Layer 4, blocked by Task 3)

Parallel paths:
- Path A: Task 1 → Task 2 → Task 3 → Task 4 (sequential)
- Path B: Task 1b (unit tests) can run in parallel with Task 1
```

### Proposing Task Order

Ask user:
```
Skill: "I've identified task dependencies:

        Task 1 (Schema) — no blockers
        Task 2 (API) — blocked by Task 1
        Task 3 (Tests) — blocked by Task 2
        Task 4 (Docs) — blocked by Task 3
        
        Parallelizable: Task 1b (unit tests) can run with Task 1
        
        Does this order make sense?"
User: "Yes" or "No, change X because..."
```

---

## Part 8: Designating Agent vs. Human

### Label Progression (State Labels)

State labels are **mutually exclusive** — remove the previous state when transitioning.

| Label | Meaning | Applied When |
|-------|---------|---------------|
| `backlog` | Story is in backlog, ready for grooming | After brd-to-issues creates it |
| `groomed` | Story is groomed, ready for triage | After story-grooming completes |
| `ready` | Ready for implementation | After triage (combine with `agent` or `human`) |
| `needs-info` | Blocked by missing information | Missing info, vague AC, open questions |
| `in-progress` | Work has started | Agent or human picks up the story |
| `ready-for-test` | Implementation complete | All AC implemented, tests passing |

### Work Type Labels (Persistent) — REQUIRED!

**Every task MUST have a work type label.** Set at triage, persists through all states:

| Label | Meaning | Set By |
|-------|---------|--------|
| `agent` | Work suitable for autonomous agent (AFK) | Triage |
| `human` | Work requiring human judgment (HITL) | Triage |

**IMPORTANT:** When creating tasks, ALWAYS apply BOTH:
- State label: `ready` (or `needs-info` if blocked)
- Work type label: `agent` OR `human`

### Heuristic for Work Type Assignment

**`agent`** (autonomous, AFK):
- Routine, well-defined work
- Clear acceptance criteria (testable, unambiguous)
- No architectural decisions
- Examples: CRUD, schema changes, boilerplate tests

**`human`** (needs judgment, HITL):
- Design decisions required
- Complex integrations or error handling
- Security/compliance logic
- Examples: Caching strategy, error recovery, performance optimization

**`needs-info`** (blocked by ambiguity):
- Vague acceptance criteria
- Missing context or requirements
- Open questions from BRD
- Examples: Task with AC "Make it fast" or "Handle edge cases"

### Applying Heuristic

For each task:

```
Task 1: Database Schema
  - Is work routine? YES (table creation, standard schema)
  - Are AC clear? YES (specific fields, constraints, indexes)
  - Design decision? NO
  → Designation: ready + agent ✓

Task 4: Documentation & Deployment
  - Is work routine? NO (requires understanding deployment strategy)
  - Are AC clear? SOMEWHAT (schema doc is clear, deployment is vague)
  - Design decision? YES (how to deploy, what's the rollback plan)
  → Designation: ready + human (needs judgment on deployment)
```

### User Override

Ask user:
```
Skill: "Task designations:
        
        Task 1 (Schema) → ready + agent (routine work, clear AC)
        Task 2 (API) → ready + agent (straightforward endpoint)
        Task 3 (Tests) → ready + agent (standard testing)
        Task 4 (Docs) → ready + human (requires judgment)
        
        Any changes?"
User: "Task 3 should be human because it needs to cover edge cases"
Skill: "Updated. Task 3 is now ready + human"
```

---

## Part 9: Creating Tracker Sub-Tasks

### Sub-Task Creation Process

For each task (in dependency order):

**Create in Tracker**:
- **Issue Type**: Sub-task
- **Parent**: Parent story (e.g., CARD-101-A)
- **Summary**: `Task N: [Layer] - [Description]` (e.g., "Task 1: Database Schema - Challenge Q&A Table")
- **Description**: [Full task description with AC, from Part 5]
- **Labels**: 
  - State label (mutually exclusive): `ready` OR `needs-info`
  - Work type label (persistent): `agent` OR `human`
  - `story:[parent-key]` (e.g., `story:CARD-101-A`)
  - `task-N-[layer]` (e.g., `task-1-schema`)
  - **Note**: Remove any previous state labels (`backlog`, `groomed`, etc.) when setting the new state
- **Blocked by**: [Task keys that block this task, e.g., "CARD-101-A-1"]
- **Status**: "To Do"
- **Story Points**: [Leave blank for team to estimate]

### Example: Creating CARD-101-A-1

```javascript
await createSubtask(
  parentKey: "CARD-101-A",
  summary: "Task 1: Database Schema - Challenge Q&A Table",
  description: `## Task 1: Database Schema - Challenge Q&A Table
  
  **Parent Story**: CARD-101-A (Challenge Q&A Creation & Encryption)
  
  ### What to build
  Create a PostgreSQL table for storing challenge questions...
  
  ### Acceptance Criteria
  - [ ] Create challenge_qa table...
  - [ ] Add constraints...
  ...
  `,
  labels: ["ready", "agent", "story:CARD-101-A", "task-1-schema", "sprint-ready"],
  blockedBy: [] // No blockers for first task
);
```

---

## Part 10: Handling Ambiguities & `needs-info`

### When to Mark `needs-info`

If a task has:
- Vague acceptance criteria (e.g., "Make it fast")
- Missing context (e.g., "What's the expected latency?")
- Open question from BRD (e.g., "Account linking timing")
- Conflicting requirements

**Example**:
```
Task: "Implement caching strategy"

AC is vague: "Cache responses for performance"

Missing info:
  - Which responses?
  - Cache TTL (time-to-live)?
  - Cache invalidation strategy?
  - Cache storage (Redis, in-memory)?

Designation: needs-info
```

### Asking Clarifying Questions

When skill detects ambiguity:

```
Skill: "Task 2 (API) has unclear AC about error handling.
        
        Question: 'If Coppice fails to validate, what should we do?'
        
        Options:
        1. Retry 3 times, then fail
        2. Queue for manual review
        3. Return HTTP 400 immediately
        4. I don't know (mark as needs-info)
        
        What should the behavior be?"
User: "I don't know, need to check with the team"
Skill: "Task marked as needs-info. Team should clarify before implementing."
```

### Marking as `needs-info`

- Label: `needs-info`
- Add comment in Tracker: "Needs clarification: [question]"
- Flag in task: "Blocked by open question: [link to BRD open question]"
- Don't create sub-task until clarified (or create with `needs-info` label)

---

## Part 11: Generating Output

### Summary Table

Output a table showing all tasks:

```markdown
## Story Grooming Summary: CARD-101-A

| Task # | Title | Estimated Size | Designation | Blocked By |
|--------|-------|-----------------|-------------|------------|
| 1 | Database Schema | 2 days | agent | None |
| 2 | Generation & Validation API | 2 days | agent | Task 1 |
| 3 | Unit & Integration Tests | 2 days | agent | Task 2 |
| 4 | Documentation & Deployment | 2 days | human | Task 3 |

**Total**: 4 tasks, 8 days (fits 2-week sprint)
**Agent tasks**: 3
**Human tasks**: 1
**Parallelizable**: Tasks can be done sequentially or with partial overlap
```

### Dependency Graph

```
Task 1 (Schema)
    ↓
Task 2 (API)
    ↓
Task 3 (Tests)
    ↓
Task 4 (Docs)

Timeline: Sequential (8 days)
Parallel opportunities: Task 1 and 1b (unit tests for schema) can overlap
```

### Agent Briefs

For each `agent` task, generate a brief (see AGENT-BRIEF.md):

```markdown
## Agent Brief: Task 1 - Database Schema

**Task Key**: CARD-101-A-1
**Parent Story**: CARD-101-A (Challenge Q&A Creation & Encryption)
**Estimated Time**: 2 days
**Designation**: agent

### What you need to do

Create a PostgreSQL table for storing encrypted challenge questions and answers.
This table will be used by the Challenge Q&A generation and validation API.

### Acceptance Criteria

1. Table created with specified schema (fields, types, constraints)
2. Indexes added for query performance
3. Encryption implemented (AES-256)
4. Migration file created with rollback
5. Unit tests pass (>80% coverage)
6. Documentation updated

### Implementation Steps

1. Create migration file in `src/db/migrations/`
2. Write Sequelize model in `src/models/ChallengeQA.ts`
3. Implement encryption/decryption in `src/services/encryption/`
4. Write unit tests in `src/models/__tests__/`
5. Run migration in staging
6. Update schema docs in `docs/schema/`

### Definition of Done

- [ ] Code reviewed and approved
- [ ] All tests passing
- [ ] Migration tested in staging
- [ ] Docs updated
- [ ] Ready to merge

### If you get stuck

1. Check: Do the acceptance criteria make sense?
2. Ask: Any missing context?
3. Flag: Escalate to team lead if blocked >2 hours
4. Escalate task to `human` if discovery happens
```

### Metadata File

Save `.story-grooming-output.json`:

```json
{
  "story_key": "CARD-101-A",
  "story_title": "Challenge Q&A Creation & Encryption",
  "parent_story": "CARD-101",
  "grooming_date": "2026-05-11T14:30:00Z",
  "sprint_fit": "fits",
  "task_count": 4,
  "tasks": [
    {
      "number": 1,
      "Tracker_key": "CARD-101-A-1",
      "title": "Database Schema - Challenge Q&A Table",
      "designation": "agent",
      "blocked_by": [],
      "codebase": "acquisitions-platform"
    },
    {
      "number": 2,
      "Tracker_key": "CARD-101-A-2",
      "title": "Generation & Validation API",
      "designation": "agent",
      "blocked_by": ["CARD-101-A-1"],
      "codebase": "acquisitions-platform"
    },
    // ... additional tasks
  ],
  "total_days": 8,
  "agent_tasks": 3,
  "human_tasks": 1,
  "needs_info": 0
}
```

### Post to Tracker

Add comment to parent story (CARD-101-A):

```
> *This was generated by story-grooming skill.*

✅ Grooming complete!

- 4 sub-tasks created (CARD-101-A-1 through CARD-101-A-4)
- All tasks fit in 2-week sprint
- Agent tasks: 3
- Human tasks: 1
- Dependency order: Task 1 → Task 2 → Task 3 → Task 4

View sub-tasks in the "Child Issues" section below.
```

---

## Part 12: Error Handling

### Common Errors & Resolution

| Error | Resolution |
|-------|-----------|
| **Story not found** | Ask user to verify Tracker key |
| **Missing acceptance criteria** | Ask user to add AC or mark story `needs-info` |
| **Tracker connection fails** | Show error, retry 3 times, ask user to verify Tracker access |
| **User can't answer question** | Mark task as `needs-info`, flag for team clarification |
| **Circular dependencies detected** | Flag and ask user to break the cycle |
| **Codebase not found** | Ask user to clarify which codebases are involved |
| **Story already has sub-tasks** | Ask: "Should I groom existing sub-tasks, or create new ones?" |

### No Silent Failures

Always inform user of issues:
```
❌ Error: Story CARD-101 has no acceptance criteria.

Options:
1. Add acceptance criteria to the story and try again
2. Mark story as needs-grooming and proceed (risky)
3. Cancel and pick a different story

What would you like to do?
```

---

## Part 13: Testing & QA Checklist

Before deploying the skill, verify:

- [ ] **Story Discovery**: Can find stories by key, epic, or `backlog` label
- [ ] **Sprint Fit Assessment**: Heuristic correctly flags over-large stories
- [ ] **Sub-Story Creation**: Creates child stories in Tracker with correct parent link
- [ ] **Codebase Analysis**: Correctly parses codebase references from story description
- [ ] **Task Slicing**: Creates tasks in dependency order (database → API → UI → tests)
- [ ] **Agent/Human Designation**: Correctly applies heuristic and accepts user overrides
- [ ] **Dependency Detection**: Correctly maps "Blocked by" relationships
- [ ] **Tracker Integration**: 
  - [ ] Creates Sub-tasks with correct parent link
  - [ ] Applies state labels (`ready`, `needs-info`) and work type labels (`agent`, `human`)
  - [ ] Populates "Blocked by" field
  - [ ] Posts comment to parent story
- [ ] **Output Generation**: 
  - [ ] Summary table is accurate
  - [ ] Dependency graph is correct
  - [ ] Agent briefs are generated
  - [ ] Metadata file is saved
- [ ] **Error Handling**: All error scenarios have graceful messages
- [ ] **Edge Cases**: 
  - [ ] Story already fits sprint (no sub-stories)
  - [ ] Story has no systems listed (ask user)
  - [ ] Task has ambiguous AC (mark `needs-info`)
  - [ ] User overrides all designations

---

## Part 14: Configuration & Setup

### Tracker Configuration Checklist

Verify these exist in your Tracker instance before running the skill:

- [ ] **Issue Types**: Epic, Story, Sub-task
- [ ] **Custom Fields**: 
  - [ ] Epic Link (links Story to Epic)
  - [ ] Parent Story (links Sub-story to parent Story)
  - [ ] Blocked By (links Sub-task to blocking Sub-tasks)
- [ ] **Labels**:
  - State labels (mutually exclusive):
    - [ ] `backlog` (initial state after creation)
    - [ ] `groomed` (after story-grooming completes)
    - [ ] `ready` (after triage, ready for implementation)
    - [ ] `needs-info` (blocked by missing information)
    - [ ] `in-progress` (work has started)
    - [ ] `ready-for-test` (implementation complete)
  - Work type labels (persistent, set at triage):
    - [ ] `agent` (AFK - agent can implement)
    - [ ] `human` (HITL - needs human judgment)
  - Other labels:
    - [ ] `story:[key]` (pattern, e.g., `story:CARD-101-A`)
    - [ ] `task-N-[layer]` (pattern, e.g., `task-1-schema`)
- [ ] **Workflow States**: "To Do", "In Progress", "In Review", "Done"

### Pre-Flight Check

Before skill runs, verify:

```
Skill: "Pre-flight check...
        
        ✅ Tracker project configured (CARD)
        ✅ Epic Link field exists
        ✅ Blocked By field exists
        ✅ Required labels exist
        ✅ Sub-task issue type available
        
        Ready to proceed!"
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-05-11 | Initial implementation guide (13 parts) |
