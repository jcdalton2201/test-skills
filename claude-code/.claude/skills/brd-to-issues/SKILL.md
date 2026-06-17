---
name: brd-to-issues
description: Convert a BRD, PRD, or technical spec into a prioritized backlog of implementation issues on the project tracker using vertical-slice architecture. Each issue is a thin, complete end-to-end slice sized for ~3 days of work by a mid-level engineer, with Gherkin acceptance criteria, explicit dependencies, and MoSCoW priority. Labels each issue with [backlog], [must]/[should]/[could]/[wont], and a size label. Publishes to GitHub/Linear with proper dependency tracking. Use this skill whenever the user wants to turn a spec into developer-ready tracker issues; mentions "break down this BRD", "create stories", "make tickets from this spec", "slice this into issues", "groom the backlog from the PRD", "plan the work", or hands you a BRD.md / PRD / requirements doc and asks what to build next.
---

# brd-to-issues

Turn a BRD / PRD / technical spec into a prioritized vertical-slice backlog and **publish directly to your project tracker** (GitHub Issues or Linear). Creates one Epic issue + numbered Story issues with proper labels for development tracking.

## Inputs

Look for these, in order:

1. A file path the user named (e.g. "convert `docs/PRD-checkout.md`").
2. `BRD.md` at the repo root (the output of `brd-grill`).
3. `PRD.md`, `SPEC.md`, `docs/requirements*.md`, `docs/prd*.md` — common spec names.
4. If multiple candidates exist, ask the user which one.

Also read `GLOSSARY.md` (root and per-folder) if present. Use its canonical terms in every story — never invent new names for concepts already defined.

If an existing codebase is present, scan it briefly (Glob top-level dirs, read `README.md`, package manifests). This grounds story sizing in real tech stack and surfaces blockers (missing infra, absent test harness, no CI).

## The vertical-slice principle

A vertical slice is a **thin end-to-end implementation of one piece of user-observable value**. It cuts through every layer the change touches — UI, API, domain, persistence, tests, observability — but does the minimum at each layer.

Why this matters: horizontal slicing ("build all the DB schemas first, then all the APIs, then all the UI") delays integration risk and ships nothing until the end. Vertical slicing forces integration on day one and produces demonstrable value every story. It also matches how mid-level engineers actually learn a codebase — by touching every layer of a small thing.

### What counts as a slice

- **Yes:** "User can submit a contact form and see a confirmation" — UI, validation, POST endpoint, persistence, response, one happy-path test.
- **Yes:** "Admin can export a single report as CSV" — button, route, generator, file download, smoke test.
- **No:** "Build the database schema." Not a slice — no user-visible behavior.
- **No:** "Implement authentication." Too large and not a single observable behavior. Slice into "user signs up with email", "user signs in", "user signs out", etc.

### Sizing rule (the 3-day heuristic)

Each story should be completable by a mid-level engineer in ~3 working days (call it 16–24 hours of focused work) including tests and code review. If a story looks bigger, split it. If a story looks much smaller (<1 day), merge it with a neighbor or roll it into a parent story.

Concrete signals a story is too big:
- Touches more than ~5 files in unfamiliar parts of the codebase.
- Has more than ~5 acceptance criteria.
- Needs more than one new external dependency or service.
- The title contains "and" connecting two distinct user outcomes.

Concrete signals a story is too small:
- The only acceptance criterion is "code compiles" or "function exists".
- Cannot be demoed to a non-engineer.
- Would naturally be a single commit inside a larger story.

## Slicing procedure

Work in this order. Each step has a clear output before moving on.

### 1. Identify the Epic

The epic is the umbrella that bundles all stories for this BRD. Usually one BRD = one epic. Exception: if the BRD covers clearly independent capabilities (e.g. "build checkout" *and* "build admin dashboard"), produce one epic per capability and a parent epic that lists them.

The epic captures: the business goal, success metrics, scope boundaries, and a story map (list of stories with priority).

### 2. Walk the BRD's functional requirements

For each functional requirement (FR-N) in the BRD:

1. Identify the smallest user-observable behavior that satisfies it. That is candidate story 1.
2. Ask: can the FR be met by one slice, or does it decompose into several? If several, list them in order of dependency.
3. For each candidate, draft a one-line user-value statement: **"As a `<role>`, I can `<action>` so that `<outcome>`."**
4. If two candidates overlap heavily, merge them. If one candidate is doing two things, split it.

### 3. Sequence by risk and dependency

Order stories so the riskiest, most load-bearing slices come first. A good first story almost always exercises:

- Auth (or whatever access control the system has)
- A round trip through every layer (UI → API → DB → response)
- The CI/CD path (so deploys are unblocked early)

This is the "walking skeleton": the thinnest possible end-to-end slice that proves the system runs. After the walking skeleton, sequence by MoSCoW (Must first) then by dependency.

### 4. Identify blockers and dependencies

For each story, list:

- **Blocked by:** stories or external work that must finish first (other story IDs, infra setup, design assets, third-party API access).
- **Blocks:** stories that depend on this one.

If a story has no blockers and blocks nothing, that's fine — say so explicitly. If everything blocks everything, you have not actually sliced vertically; reconsider.

### 5. Write acceptance criteria in Gherkin

Each story gets 2–5 Gherkin scenarios covering the happy path and the most important edge cases. Format:

```gherkin
Scenario: <short name>
  Given <precondition>
  When <action>
  Then <observable outcome>
```

Rules:
- Every `Then` clause must be observable from outside the code (a returned value, a UI state, a DB row, a logged event). "Then the function returns the right type" is not acceptable.
- Cover at least one negative path per story unless the behavior has no failure mode.
- Use canonical terms from `GLOSSARY.md`.

### 6. Assign priority, sizing, status

- **Priority:** inherit from the parent FR's MoSCoW priority in the BRD. If the FR is a `Must`, every slice required to satisfy the `Must` is also `Must`. Optional embellishments of that capability can be `Should` or `Could`.
- **Size:** estimate in days (1, 2, or 3). If you would write 4+, you have not sliced enough.
- **Status:** every newly-generated story starts at `Backlog`.

### 7. Mint a project key

Issue trackers (Jira, Linear, GitHub Projects) expect a short uppercase project prefix so every issue gets a key like `TPS-1`, `MER-42`, `CYP-17`. Derive one from the BRD:

- Take the project name from the BRD title or Executive Summary.
- Prefer a 3-letter abbreviation that reads naturally: `TeamPulse` → `TPS`, `MercatorPay` → `MER`, `Cyprian` → `CYP`, `Checkout Revamp` → `CKO`.
- For single short words, take initial / mid / final consonants: `Atlas` → `ATL`.
- If a `tracker_prefix` / `project_key` is set in the BRD or a sibling config (`.tracker`, `linear.yml`), honor it instead of deriving.
- Ask the user only when no reasonable 3-letter prefix can be inferred.

Use the prefix as the stem of every key:

- **Epic key:** `<PREFIX>-1` (the epic is itself a tracker issue, conventionally #1).
- **Story keys:** `<PREFIX>-2`, `<PREFIX>-3`, ... — sequential integers, no zero padding.

## Workflow: From Specification to Tracker

### 1. Draft Phase (No Tracker Changes)

**Agent**:
- Reads BRD/PRD
- Identifies vertical slices
- Maps dependencies
- Creates story table
- Shows to user for approval

**User approves or asks changes**:
- "Should we split this story?"
- "Are the dependencies right?"
- "Is this the right first story?"

**Iterate until user approves** (no tracker action yet).

### 2. Publish Phase (Create Tracker Issues with Labels)

**Agent detects tracker**:
- GitHub repo + `gh` CLI → use `gh issue create`
- Linear + MCP tools loaded → use Linear API
- Otherwise ask user how to publish

**Agent publishes in dependency order**:

```
Step 1: Create Epic issue (PREFIX-1)
        Labels: [epic]
        
Step 2: Create Story issues (PREFIX-2, PREFIX-3, ...)
        Each gets labels:
        - [backlog]           (raw issue, needs grooming)
        - [must|should|could] (MoSCoW priority)
        - [s1|s2|s3]          (size: 1-3 days)
        - [epic-<PREFIX>-1]   (link to parent epic)
        
Step 3: Set dependencies via "Blocked by" field
        Each story references blockers in issue body
        
Step 4: Maintain mapping of story# → real issue ID
        So later stories can reference actual IDs
```

**Publishing labels**:

| Label | Meaning | Applied |
|-------|---------|----------|
| `backlog` | Raw issue, needs grooming | All new stories |
| `must` | MoSCoW Must (highest priority) | Must stories |
| `should` | MoSCoW Should (medium priority) | Should stories |
| `could` | MoSCoW Could (nice-to-have) | Could stories |
| `wont` | MoSCoW Won't (explicitly out of scope) | Won't stories |
| `s1` | Sized at 1 day (~4-8 hours) | 1-day stories |
| `s2` | Sized at 2 days (~8-16 hours) | 2-day stories |
| `s3` | Sized at 3 days (~16-24 hours) | 3-day stories |
| `epic` | This is an epic issue (parent) | Epic only |
| `epic-PREFIX-1` | Belongs to epic PREFIX-1 | All story issues |
| `walking-skeleton` | This is the first, load-bearing slice | First story only |

**GitHub command example**:
```bash
gh issue create \
  --title "Walking skeleton: auth endpoint" \
  --body "$(cat <<'EOF'
## Parent Epic
PREFIX-1

## User Value
As a user, I can sign up and sign in so that I can use the app.

## Acceptance Criteria
- [ ] User can sign up with email
- [ ] User receives welcome email
- [ ] User can sign in with credentials

## Blocked by
None - can start immediately

## Blocks
PREFIX-3, PREFIX-4
EOF
)" \
  --label "backlog,must,s2,epic-PREFIX-1,walking-skeleton"
```

**Linear command example**:
```
mcp__linear__save_issue(
  title: "Walking skeleton: auth endpoint",
  description: "<body from template>",
  labelIds: ["backlog", "must", "s2", "epic-PREFIX-1", "walking-skeleton"],
  teamId: "<team>",
  blockingIssueIds: []
)
```

### 3. Output Summary

After publishing, hand back:

```
Published 7 issues to <tracker>:

EPIC:
  PREFIX-1 [epic]                Epic: User Onboarding

STORIES (in dependency order):
  PREFIX-2 [backlog][must][s2][walking-skeleton]     Walking skeleton: auth endpoint
  PREFIX-3 [backlog][must][s2][epic-PREFIX-1]        User signs up with email (blocked by PREFIX-2)
  PREFIX-4 [backlog][must][s2][epic-PREFIX-1]        User signs in with password (blocked by PREFIX-2)
  PREFIX-5 [backlog][should][s2][epic-PREFIX-1]      Send welcome email (blocked by PREFIX-3)
  PREFIX-6 [backlog][should][s1][epic-PREFIX-1]      Add password reset (blocked by PREFIX-2)
  PREFIX-7 [backlog][could][s3][epic-PREFIX-1]       Add OAuth signup (blocked by PREFIX-2)

All issues are in backlog and ready for grooming.
Next step: Use story-grooming to refine stories, then story-readiness or plan-to-tracer-issues to triage.
  - Grooming: `backlog` → `groomed`
  - Triage: `groomed` → `ready` + `agent` or `human`, or `needs-info`
```

## Label Progression (Story Lifecycle)

See [LABEL-PROGRESSION.md](../LABEL-PROGRESSION.md) for the full reference.

### Two Types of Labels

1. **State labels** (mutually exclusive): `backlog` → `groomed` → `ready` / `needs-info` → `in-progress` → `ready-for-test`
2. **Work type labels** (persistent): `agent` or `human` — set at triage, persists through all states

### Label Flow

```
[backlog]                           ← this skill creates raw issues
    ↓ (grooming)
[groomed]                           ← story-grooming refines
    ↓ (triage)
[ready] + [agent] or [human]        ← story-readiness assigns
    ↓ (work starts)
[in-progress] + [agent] or [human]
    ↓ (complete)
[ready-for-test] + [agent] or [human]
    ↓ (merged)
[shipped]
```

### This Skill's Responsibility

**When publishing issues:**
- Apply: `backlog`, priority label (`must`/`should`/`could`), size label (`s1`/`s2`/`s3`), epic label
- Issues are raw and need grooming before work can begin

## When to Stop

Agent is done when:

1. Epic issue created in tracker with story map
2. All story issues created in tracker with proper labels
3. Dependencies linked via "Blocked by" fields
4. Every story has Gherkin acceptance criteria
5. User has approved the backlog (or asked for changes and approved revised version)

Hand control back with:
- Link to epic issue (e.g., "PREFIX-1" or tracker URL)
- List of story IDs with titles
- Note that issues are ready for next phase (planning with plan-to-tracer-issues, or direct implementation with rust-tdd)

## Next Steps After Publishing

**Option A: Use plan-to-tracer-issues (Recommended)**
- For each story, mark as AFK (agent can implement) or HITL (needs human decision)
- Creates child HITL issues for blocking decisions
- Agents know exactly which stories they can pick up

**Option B: Use rust-tdd directly**
- Pick an issue with [ready][agent] labels
- Agent implements via TDD
- Agent creates HITL issues for decisions
- Agent updates story labels when complete

## Integration with Tracker System

Stories published here flow through the label progression:

```
brd-to-issues publishes:  [backlog] [must|should|could] [s1|s2|s3]
                          ↓
story-grooming refines:   - Remove [backlog]
                          - Add [groomed]
                          ↓
story-readiness triages:  - Remove [groomed]
                          - Add [ready] + [agent] OR [ready] + [human] OR [needs-info]
                          ↓
rust-tdd implements:      - Remove [ready]
                          - Add [in-progress]
                          - Keep [agent] or [human]
                          - (if blocked) Remove [in-progress], Add [needs-info]
                          - (when complete) Remove [in-progress], Add [ready-for-test]
```

## Style

- Be terse. Engineers read these in a hurry.
- Use canonical domain terms from `GLOSSARY.md` if present.
- No platitudes. Cut anything that does not change what gets built.
- Acceptance criteria must be externally observable (returned value, UI state, DB row, log event). "Function exists" is not acceptable.
- Cover at least one negative path per story unless the behavior has no failure mode.
- Title is a statement of user-observable behavior, not implementation steps.
