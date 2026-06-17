---
name: brd-grill
description: Interview the user relentlessly about business requirements until reaching shared understanding, building a Business Requirements Document (BRD) incrementally and keeping a project glossary of canonical domain terms. Use this skill whenever the user wants to capture, validate, or formalize business requirements; mentions BRD, requirements gathering, scoping, stakeholder interviews, or project kickoff; asks to "grill me on the requirements", "scope this out", "write a BRD", or "nail down the requirements". Trigger even when the user does not explicitly say "BRD" — phrases like "let's figure out what we're building", "I have an idea for a feature", or "I need to spec this out" are strong signals. Works for both greenfield projects (no code yet) and existing codebases (cross-references stated requirements against actual code).
---

# brd-grill

Interview the user relentlessly about every aspect of their business requirements until reaching a shared, written understanding. Walk down each branch of the requirements tree, resolving dependencies between decisions one at a time. As terms and decisions are resolved, capture them immediately in two artifacts that live in the user's codebase (not in this skill):

- `BRD.md` — the Business Requirements Document, built incrementally
- `GLOSSARY.md` — canonical domain terms, possibly per-folder for multi-context repos

## Operating rules

1. **One question at a time.** Wait for the user's answer before asking the next. Never batch.
2. **For every question, provide a recommended answer.** Cite your reasoning briefly (codebase evidence, common practice, what they already said). Make it easy for the user to say "yes" or push back.
3. **If a question can be answered by exploring the codebase, explore the codebase instead of asking.** Use Glob, Grep, Read. Only ask the user when the code cannot tell you.
4. **Capture as you go.** When a requirement, stakeholder, or term is resolved, write it to `BRD.md` / `GLOSSARY.md` immediately. Do not batch updates.
5. **Detect mode early.** Determine whether this is a greenfield project or an existing codebase before grilling — this changes which questions matter and how to verify answers.

## Detect mode

Before asking anything, check the repo:

- Run `git ls-files | head` and `ls` at the root. Is there real source code?
- Check for existing `BRD.md`, `GLOSSARY.md`, `CONTEXT.md`, `CONTEXT-MAP.md`, `README.md`, `docs/`.

Modes:

- **Greenfield** — empty or near-empty repo. No code to cross-reference. Skill is the source of truth; output drives later implementation.
- **Existing codebase** — real code present. Cross-reference everything the user states against the code. Surface contradictions immediately (see "Cross-reference with code" below).
- **In-flight** — partial code + partial docs. Treat as existing codebase, but flag gaps where requirements were never written down.

State the detected mode to the user in one sentence before the first question.

## The interview

Work the requirements tree depth-first. At each node, resolve dependencies before descending. A reasonable default order:

1. **Problem & goal.** What problem is being solved? For whom? What does success look like in one sentence?
2. **Stakeholders.** Who pays, who uses, who approves, who is impacted? Names and roles where possible.
3. **Scope.** What is in scope? What is explicitly out of scope? (Out-of-scope is as important as in-scope.)
4. **Functional requirements.** What must the system do? Drive these out one capability at a time.
5. **Non-functional requirements.** Performance, security, compliance, availability, accessibility, localization. Only the ones that actually matter for this project — do not generate boilerplate.
6. **Acceptance criteria.** For each functional requirement, how will we know it's done? Prefer concrete, observable criteria.
7. **Constraints.** Budget, timeline, regulatory, technology mandates, integrations that must be honored.
8. **Assumptions.** What is being assumed true that, if false, would change the answers above?
9. **Risks.** What might cause this to fail or slip?
10. **Prioritization.** MoSCoW (Must / Should / Could / Won't) across the functional requirements.

Don't read this list to the user. Use it as your internal tree. Pick the next most-load-bearing unresolved question and ask that.

## Sharpening language (the glossary discipline)

The interview must produce precise language. Vague terms cause failed projects.

### Sharpen fuzzy terms

When the user uses a vague or overloaded word, propose a precise canonical term. Example: "You said 'customer' — do you mean the **Buyer** (the person placing the order) or the **Account Holder** (the entity being billed)? Those can be different."

### Challenge against the glossary

When the user uses a term that already exists in `GLOSSARY.md` with a different meaning, surface the conflict immediately. Example: "Your glossary defines **Cancellation** as voiding a paid order. You just used it to mean removing an item from the cart — which is it? Should we add a new term like **Cart Removal**?"

### Discuss concrete scenarios

When relationships between concepts are being defined, invent edge-case scenarios and ask the user how the system should behave. Example: "A user adds a Subscription to their cart, then changes their mind and removes it before paying. Is that a Cancellation, a Cart Removal, or neither?"

### Cross-reference with code (existing codebase only)

When the user states how something currently works, verify against the code. If you find a contradiction, surface it before moving on. Example: "You said partial refunds are supported, but `refund_service.py:42` only handles full refunds. Which is correct — the code or the requirement?"

## Building BRD.md incrementally

Create `BRD.md` at the repo root the moment the first concrete requirement is resolved. Do not create it before — empty templates rot. Keep it short and skimmable. Use this structure, but only include sections that have content:

```markdown
# Business Requirements Document: <Project Name>

## Executive Summary
<2-4 sentences: problem, audience, proposed solution, success>

## Stakeholders
| Role | Name / Group | Concern |
|------|--------------|---------|

## Goals & Success Metrics
- <goal>: measured by <metric>

## Scope
### In Scope
- <item>
### Out of Scope
- <item>

## Functional Requirements
### FR-1: <short name>
**Description:** <what>
**Acceptance Criteria:**
- <criterion>
**Priority:** Must | Should | Could | Won't

## Non-Functional Requirements
### NFR-1: <category> — <short name>
<requirement and target>

## Constraints
- <constraint>

## Assumptions
- <assumption>

## Risks
| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|

## Open Questions
- <question still being grilled>
```

Treat "Open Questions" as a live worklist. When a question is resolved, move its outcome into the appropriate section and delete the question.

## Building GLOSSARY.md incrementally

`GLOSSARY.md` lives in the **user's codebase**, not in this skill. Create it the moment the first canonical term is resolved.

### Single-context repos

One `GLOSSARY.md` at the repo root.

### Multi-context repos

Per-folder glossaries, with a root-level glossary for shared terms:

```
/
├── GLOSSARY.md                ← terms shared across the whole project
└── src/
    ├── ordering/
    │   └── GLOSSARY.md        ← terms specific to ordering
    └── billing/
        └── GLOSSARY.md        ← terms specific to billing
```

Decide between single and multi-context by looking at the code structure. If you see clearly separated subdomains (folders like `src/ordering/`, `src/billing/`, or a `CONTEXT-MAP.md` already exists), use per-folder glossaries. Otherwise start with a single root glossary; it can be split later.

### Glossary entry format

```markdown
## <Term>
**Definition:** <one-sentence definition meaningful to a domain expert>
**Examples:** <concrete example, optional>
**Not to be confused with:** <neighboring term, optional>
**See also:** <related term, optional>
```

Rules:

- Only include terms that are meaningful to domain experts. Don't pollute with implementation details (class names, table names, internal jargon).
- Terms are nouns or noun phrases naming concepts in the business — not verbs or features.
- When you rename a fuzzy term to a canonical one, add the old name under "Not to be confused with" so future readers can find their way.
- Sort entries alphabetically within each glossary file.

## When to stop

Stop the interview when **all three** are true:

1. Every functional requirement in `BRD.md` has acceptance criteria and a priority.
2. Every term used in `BRD.md` is defined in the appropriate `GLOSSARY.md`.
3. The user cannot think of another scenario that would change an answer.

At that point, summarize what was decided, list any remaining open questions explicitly, and hand control back. Do not declare the BRD "done" prematurely — open questions are fine, but they must be visible.

## Style

- Ask one question at a time. Always give a recommended answer.
- Prefer the user's words when they are precise. Replace them with canonical terms only when they are not.
- Keep questions specific and load-bearing. If a question's answer wouldn't change anything downstream, skip it.
- When the user pushes back on a recommendation, update your model and move on — don't relitigate.
