# Label Progression Reference

This document defines the **label system** used across all skills. There are two types of labels:

1. **State labels** — Mutually exclusive, represent workflow position
2. **Work type labels** — Persistent, represent who should do the work

## State Labels (Mutually Exclusive)

Only one state label should be active at any time. Remove the previous state label when transitioning.

| Label | Meaning | Who Sets It |
|-------|---------|-------------|
| `backlog` | Raw issue, needs grooming | `brd-to-issues` |
| `groomed` | Refined and ready for triage | `story-grooming`, `story-readiness` |
| `needs-info` | Blocked by missing information | `story-readiness`, `plan-to-tracer-issues`, `rust-tdd` |
| `ready` | Triaged and ready for work | `story-readiness`, `plan-to-tracer-issues` |
| `in-progress` | Work has started | `rust-tdd` or human |
| `ready-for-test` | Implementation complete | `rust-tdd` or human |
| `shipped` | Merged and deployed (optional) | Human or CI |

## Work Type Labels (Persistent) — REQUIRED!

**Every triaged issue MUST have a work type label.** Set once at triage time. Persists through all state transitions.

| Label | Meaning | Who Sets It |
|-------|---------|-------------|
| `agent` | Autonomous agent can implement (AFK) | `story-readiness`, `plan-to-tracer-issues` |
| `human` | Needs human judgment (HITL) | `story-readiness`, `plan-to-tracer-issues` |

**IMPORTANT:** When triaging from `groomed` → `ready`, you MUST apply BOTH:
- State label: `ready`
- Work type label: `agent` OR `human`

The work type label is NOT optional. It enables filtering like "all agent work in progress."

## Label Flow Diagram

```
[backlog]                           ← brd-to-issues creates raw issues
    │
    ↓ (grooming: refine AC, size, break down)
    │
[groomed]                           ← story-grooming refines it
    │
    ↓ (triage: who should work on this?)
    │
┌───┴───────────────────────────┐
│                             │
↓                             ↓
[needs-info]              [ready] + [agent] or [human]
    │                         │
    │                         ↓ (work starts)
    │                    [in-progress] + [agent] or [human]
    │                         │
    │      ┌──────────────────┤
    │      │                  │
    │      │ (blocked)        │ (implementation complete)
    │      ↓                  ↓
    └──────┤             [ready-for-test] + [agent] or [human]
           │                  │
           │                  ↓ (merged)
           │             [shipped] or closed
           │
           └──→ (clarification received) ──→ [ready] + [agent] or [human]
```

**Note:** The `[agent]` or `[human]` work type label is set at triage and persists through all subsequent states.

## Transition Rules

### Rule 1: Always Remove Previous State Label

When transitioning to a new state, **remove the previous state label first**, then add the new one.

```
WRONG: Issue has [groomed] [ready]  ← Two state labels!
RIGHT: Issue has [ready]            ← Previous removed
```

### Rule 2: Work Type Labels Persist

Once set, `[agent]` or `[human]` stays on the issue through all state transitions:

```
Triaged:    [ready][agent][must][s2]
Started:    [in-progress][agent][must][s2]    ← [agent] persists
Complete:   [ready-for-test][agent][must][s2] ← [agent] still there
```

### Rule 3: Other Labels Are Additive

These labels are **not state labels** and don't need to be removed during transitions:
- Priority: `must`, `should`, `could`, `wont`
- Size: `s1`, `s2`, `s3`
- Epic: `epic`, `epic-PREFIX-1`
- Work type: `agent`, `human`
- Special: `walking-skeleton`

## Transitions by Skill

### brd-to-issues
- **Creates issues with:** `backlog` + priority + size + epic labels
- Issues are raw, need grooming before work can begin

### story-grooming
- **Input:** Issues with `backlog`
- **Output:** 
  - Remove `backlog`, add `groomed`
  - Refines AC, sizes story, breaks down if needed

### story-readiness / plan-to-tracer-issues
- **Input:** Issues with `groomed`
- **Output:**
  - Score >= 80%, clear AC → Remove `groomed`, add `ready` + `agent`
  - Score >= 80%, needs judgment → Remove `groomed`, add `ready` + `human`
  - Score < 80% → Remove `groomed`, add `needs-info`

### rust-tdd (or human implementation)
- **When starting:**
  - Remove `ready`
  - Add `in-progress`
  - Keep `agent` or `human` label
  
- **When blocked by question:**
  - Remove `in-progress`
  - Add `needs-info`
  - Create blocking issue for the question
  - Keep `agent` or `human` label
  
- **When unblocked:**
  - Remove `needs-info`
  - Add `in-progress`
  - Keep `agent` or `human` label
  
- **When implementation complete:**
  - Remove `in-progress`
  - Add `ready-for-test`
  - Keep `agent` or `human` label

### After merge
- Remove `ready-for-test`, size labels
- Optionally add `shipped`
- `agent` or `human` can stay for historical tracking

## Example Issue Lifecycle

```
Created:    [backlog][must][s2][epic-MER-1]
Groomed:    [groomed][must][s2][epic-MER-1]            (removed backlog)
Triaged:    [ready][agent][must][s2][epic-MER-1]       (removed groomed, added agent)
Started:    [in-progress][agent][must][s2][epic-MER-1] (removed ready, kept agent)
Blocked:    [needs-info][agent][must][s2][epic-MER-1]  (removed in-progress, kept agent)
Resumed:    [in-progress][agent][must][s2][epic-MER-1] (removed needs-info, kept agent)
Complete:   [ready-for-test][agent][must][s2][epic-MER-1] (removed in-progress, kept agent)
Merged:     [shipped][agent][must]                     (removed ready-for-test, s2)
```

## JQL Queries

```sql
-- Find raw issues in backlog (need grooming)
labels = "backlog"

-- Find groomed issues awaiting triage
labels = "groomed"

-- Find issues ready for agent work
labels = "ready" AND labels = "agent"

-- Find issues ready for human work
labels = "ready" AND labels = "human"

-- Find all agent work in progress
labels = "in-progress" AND labels = "agent"

-- Find all human work in progress  
labels = "in-progress" AND labels = "human"

-- Find issues needing information
labels = "needs-info"

-- Find issues ready for testing (agent work)
labels = "ready-for-test" AND labels = "agent"

-- Find all agent work (any state)
labels = "agent"

-- Find all human work (any state)
labels = "human"
```
