---
name: plan-to-tracer-issues
description: Take an existing issue, plan, spec, or PRD and groom it into independently-grabbable issues on the project issue tracker using tracer-bullet vertical slices. Each slice is end-to-end (schema, API, UI, tests) and marked with a work type label (agent or human). Quiz the user on granularity, dependencies, and work type labels until they approve, then publish issues in dependency order with the ready label and appropriate work type. Use when user wants to convert a plan into issues, create implementation tickets, groom a story, break down work into tracer-bullet slices, or mentions "slice this", "groom this", "turn this into tickets", "make agent issues", or "break down this issue". Pairs with brd-grill (for upstream BRDs) but works on any plan, spec, PRD, or parent issue.
---

# plan-to-tracer-issues

Groom a plan, spec, PRD, or existing parent issue into a backlog of tracer-bullet vertical-slice issues, published to the project issue tracker in dependency order. Each slice gets state label `ready` (or `needs-info` if blocked) plus a work type label: `human` (HITL, human-in-the-loop) or `agent` (AFK, agent can finish unsupervised). Prefer `agent`.

## Workflow

1. **Gather context** — locate the source (issue / plan / spec / PRD).
2. **Explore codebase** (optional) — ground slices in real structure.
3. **Draft vertical slices** — assign work type (agent or human), with rationale.
4. **Quiz the user** — iterate until they approve.
5. **Publish to tracker** — dependency order, `ready` + work type labels, real IDs in `Blocked by`.

Do **not** close or modify any parent issue.

## 1. Gather context

Source priority:

1. The user named an issue (`MER-42`, `#118`, URL) → fetch it.
2. The user named a file (`docs/PRD-checkout.md`) → read it.
3. `BRD.md` / `PRD.md` / `SPEC.md` / `docs/requirements*.md` at repo root.
4. Multiple candidates → ask which one.

Read `GLOSSARY.md` (root and per-folder) if present and reuse canonical terms.

If source is an existing tracker issue, capture its identifier — every child slice will reference it under `## Parent`.

## 2. Explore codebase (optional)

If a codebase is present and slicing benefits from it (sizing, naming, layer count), do a light scan:

- Glob top-level dirs.
- Read `README.md`, package manifests, `Makefile` / `justfile`.
- Note test framework, CI path, deploy path.

Skip if the plan is greenfield or the user says "don't explore".

## 3. Draft vertical slices

### Vertical-slice rules

- Each slice delivers a narrow but **complete** path through every layer it touches (schema, API, UI, tests).
- A completed slice is **demoable or verifiable on its own**.
- Prefer **many thin slices** over few thick ones.
- Each slice has a single user-observable outcome. Title containing "and" is a smell.

Not a slice:
- "Build the database schema." → no observable behavior.
- "Implement authentication." → too broad; split into "user signs up", "user signs in", etc.

### Sizing

Aim for ~1–3 days of focused work per slice for a mid-level engineer (or an AFK agent run). Split if larger. Merge if trivially smaller.

### Label Assignment

Each issue gets two types of labels:

**State label** (mutually exclusive, exactly one):
- **`ready`** — triaged and ready to work. All information present, AC fully specified.
- **`needs-info`** — blocked by missing information. Examples: ambiguous acceptance criteria, open questions from BRD, conflicting requirements, missing context.

**Work type label** (persistent, set at triage, exactly one):
- **`agent`** (AFK) — agent can implement, test, and merge without human input. No open architectural choices, no design review needed, AC fully specified, no new external accounts/secrets, no destructive migrations.
- **`human`** (HITL) — requires human interaction. Examples: architectural decision still open, UX/design review needed, new infra/account provisioning, security-sensitive change, schema migration with data backfill, anything touching prod credentials.

**Default to `agent`.** Only mark `human` when a concrete human-only step is named. Only mark `needs-info` when specific information is missing. If a slice is blocked because of one open question, prefer splitting: a small `needs-info` "clarify X" slice, then a `ready` + `agent` "implement X" follow-up.

### Drafting

For each slice, draft:

- Title (imperative, single outcome).
- One-line user value: "As a `<role>`, I can `<action>` so that `<outcome>`."
- State label (`ready` or `needs-info`) + work type (`agent` or `human`) + one-line rationale.
- Acceptance criteria as a checklist (3–6 items, all externally observable).
- Blockers (other slices, or "None").

Sequence slices so the walking skeleton (auth, end-to-end round trip, CI/CD path) comes first, then dependencies, then nice-to-haves.

## 4. Quiz the user

Present the draft backlog as a compact table:

| # | Title | State | Work Type | Blocked by | Why this slice |
|---|-------|-------|-----------|------------|----------------|
| 1 | ... | ready | agent | — | walking skeleton |
| 2 | ... | ready | human | 1 | needs design review on X |
| 3 | ... | ready | agent | 2 | ... |
| 4 | ... | needs-info | agent | — | missing latency requirements |

Then ask, explicitly:

- Does the granularity feel right? (too coarse / too fine)
- Are the dependency relationships correct?
- Should any slices be merged or split further?
- Are the labels correct? (state: `ready` vs `needs-info`; work type: `agent` vs `human`)

Iterate. Revise the table after each round. Do **not** publish until the user approves.

## 5. Publish issues

After approval:

1. **Detect tracker.**
   - GitHub repo + `gh` CLI available → use `gh issue create`.
   - Linear MCP tools loaded (`mcp__*__save_issue`) → use Linear.
   - Otherwise ask the user which tracker and how to publish (or write to `STORIES.md` as fallback).
2. **Publish in dependency order** — blockers first, so each child slice can reference real IDs in `## Blocked by`. Maintain a map of `slice# → real issue ID` as you go.
3. **Labels — MUST apply BOTH state AND work type labels.**
   - Remove `groomed` label (this skill triages groomed issues).
   - Apply state label: `ready` or `needs-info`.
   - **ALWAYS apply work type label: `agent` or `human`.** This is required!
   - Apply any project-mandated labels the user names.
   
   **IMPORTANT:** Every triaged issue MUST have:
   - Exactly ONE state label (`ready` or `needs-info`)
   - Exactly ONE work type label (`agent` or `human`)
   
   Example label sets:
   - `ready, agent` — ready for autonomous agent work
   - `ready, human` — ready but needs human judgment
   - `needs-info, agent` — blocked, but will be agent work once unblocked
4. **Body** — use the template below verbatim.
5. **Do not** close, edit, or comment on the parent issue. Just reference it under `## Parent`.

### Issue body template

```markdown
## Parent

<reference to parent issue on the tracker, e.g. `#118` or `MER-42` or URL. Omit this section entirely if the source was a file/plan, not an existing issue.>

## What to build

<Concise description of this vertical slice. Describe end-to-end behavior, not layer-by-layer implementation.

Avoid specific file paths or code snippets — they go stale fast. Exception: if a prototype produced a snippet that encodes a decision more precisely than prose can (state machine, reducer, schema, type shape), inline it here and note briefly that it came from a prototype. Trim to the decision-rich parts — not a working demo, just the important bits.>

## Acceptance criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Blocked by

- <reference to blocking ticket, e.g. `#119`>

<or "None - can start immediately" if no blockers>
```

### Publishing commands

GitHub:

```bash
gh issue create \
  --title "<slice title>" \
  --body "$(cat <<'EOF'
<body using template above>
EOF
)" \
  --label "ready,agent"   # ALWAYS include BOTH state (ready) AND work type (agent/human)
```

Linear: call `mcp__*__save_issue` with `title`, `description`, `labelIds` (resolve names → IDs first via `mcp__*__list_issue_labels`), and `teamId`.

After publishing, hand back a summary:

```
Published 7 issues to <tracker>:
  #201 [ready] [agent]   Walking skeleton: list endpoint
  #202 [ready] [agent]   Add filtering by status (blocked by #201)
  #203 [ready] [human]   Decide pagination strategy (blocked by #201)
  #204 [needs-info] [agent]  Caching strategy (missing: TTL requirements)
  ...

Label progression: see ../LABEL-PROGRESSION.md
  - Work type label (agent/human) persists through all state changes
  - When work starts: ready → in-progress
  - When blocked: in-progress → needs-info
  - When complete: in-progress → ready-for-test
```

## Style

- Be terse. Engineers read these tickets in a hurry.
- Use canonical domain terms from `GLOSSARY.md` if present.
- No platitudes. Cut anything that does not change what gets built.
- Acceptance criteria must be externally observable (returned value, UI state, DB row, log event). "Function exists" is not acceptable.
- Cover at least one negative path per slice unless the behavior has no failure mode.

## Label Progression Reference

See [../LABEL-PROGRESSION.md](../LABEL-PROGRESSION.md) for the full label system.

This skill handles the `groomed` → `ready` / `needs-info` transition and assigns work type labels:

```
[groomed]                           ← upstream skill creates issues
    ↓ (this skill triages)
[ready] + [agent] or [human]        ← work type set here, persists forever
    or
[needs-info] + [agent] or [human]   ← if blocked by missing info
    ↓ (work starts)
[in-progress] + [agent/human]       ← agent or human picks up work
    ↓ (if blocked by question)
[needs-info] + [agent/human]        ← create HITL issue for question
    ↓ (when unblocked)
[in-progress] + [agent/human]
    ↓ (implementation complete)
[ready-for-test] + [agent/human]    ← all AC met, tests passing
    ↓ (merged)
[shipped] or closed
```

**Key rules:**
- Remove the previous state label when transitioning. Only one state label at a time.
- Work type label (`agent` or `human`) is set at triage and never changes.

## When to stop

You are done when:

1. Every slice in the approved table has a published issue.
2. Every issue has the parent reference (if applicable), AC, and real `Blocked by` IDs.
3. Every issue carries exactly one state label (`ready` or `needs-info`) and exactly one work type label (`agent` or `human`).
4. The `groomed` label has been removed from all triaged issues.
5. The parent issue is untouched.
