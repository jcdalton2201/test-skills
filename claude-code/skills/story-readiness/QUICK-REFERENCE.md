# Story Readiness - Quick Reference

## Input

This skill takes `groomed` stories as input (they've been groomed from `backlog`).

## Check Single Story

1. Fetch story: get_issue + get_story_points
2. Calculate score with quality checks
3. If score < 80%: post comment, remove `groomed`, add `needs-info`
4. If score >= 80% with clear AC: remove `groomed`, add `ready` + `agent`
5. If score >= 80% but needs judgment: remove `groomed`, add `ready` + `human`

## Output

- `ready` + `agent` — ready for autonomous agent work
- `ready` + `human` — ready but needs human judgment
- `needs-info` — not ready, needs clarification

## Scoring (100 Points)

| Criteria | Points | Breakdown |
|----------|--------|-----------|
| Acceptance Criteria | 25 | Existence (10) + Quality (15) |
| Description | 20 | Existence (8) + Quality (12) |
| Story Points | 15 | Fibonacci 1-13 |
| Epic Link | 10 | Linked to parent |
| Labels | 10 | Workflow labels |
| Dependencies | 10 | Identified or None |
| Assignee | 5 | If in sprint |
| Sprint Fit | 5 | pts<=8, AC<=6 |

## Quality Checks (NEW)

### AC Quality (15 pts)
- +5: Testable format (Given/When/Then)
- +5: Technical specifics (API, fields, codes)
- +3: Edge cases (errors, validation)
- +2: Expected values (examples)

### Description Quality (12 pts)
- +4: User story format
- +3: Technical context (systems, APIs)
- +3: Scope boundaries (in/out)
- +2: Examples

## Red Flags (Deduct Points)

Vague language agents cannot work with:

- handle errors properly -> specify error codes
- make it fast -> add target (< 200ms)
- should work correctly -> not testable
- implement the feature -> no details
- TBD / TODO -> incomplete

## Green Flags (Award Points)

Specific details agents can implement:

- POST /api/challenge/verify -> endpoint
- return 400 with INVALID_FORMAT -> error code
- Given X, When Y, Then Z -> testable
- { verified: true } -> example value

## Thresholds

- >= 80%: Ready (`ready` + `agent` or `ready` + `human`)
- < 80%: Not Ready (`needs-info` + comment)

## Label System

### State Labels (Mutually Exclusive)

`backlog` → `groomed` → `ready` / `needs-info` → `in-progress` → `ready-for-test`

**Rule:** Remove previous state label, then add new one.

### Work Type Labels (Persistent)

`agent` or `human` — set at triage, persists through all state transitions.

## Label Actions — MUST APPLY WORK TYPE LABEL!

**Every triaged story MUST have BOTH a state label AND a work type label.**

Score < 80%:
- REMOVE: `groomed`, `ready`
- ADD: `needs-info`
- KEEP: `agent`/`human` (work type persists if already set)

Score >= 80% (clear AC):
- REMOVE: `groomed`, `needs-info`
- ADD: `ready`
- **MUST ADD: `agent`** ← Required!

Score >= 80% (needs judgment):
- REMOVE: `groomed`, `needs-info`
- ADD: `ready`
- **MUST ADD: `human`** ← Required!

## Comment Template (< 80%)

Includes:
- Score and status
- What is complete
- What needs work (gaps table)
- Agent-readiness issues (red flags, missing specifics)
- Next steps

## Skill Integration

| Gap | Command |
|-----|---------|
| No estimate | /estimate-story |
| Vague AC/desc | /grill-me |
| Too large | /story-grooming |
