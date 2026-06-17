# Story Readiness Scoring Guide

## Overview

Stories are scored 0-100% based on completeness AND quality criteria.
The threshold for ready is 80%.

**Key Principle:** We don't just check that AC/descriptions exist - we check if they have enough detail for an AI agent to implement.

---

## Scoring Summary

| Criteria | Points | Breakdown |
|----------|--------|-----------|
| Acceptance Criteria | 25 | Existence (10) + Quality (15) |
| Description | 20 | Existence (8) + Quality (12) |
| Story Points | 15 | Valid Fibonacci (1-13) |
| Epic Linkage | 10 | Linked to parent |
| Labels | 10 | Has workflow labels |
| Dependencies | 10 | Identified or stated None |
| Assignee | 5 | Has assignee if in sprint |
| Sprint Fit | 5 | Points <=8, AC <=6 |

---

## 1. Acceptance Criteria (25 points)

### Existence Score (10 pts)

- 10 pts: Has 2-6 acceptance criteria
- 7 pts: Has 1 AC or 7-8 AC
- 3 pts: Has AC section but empty or 9+ AC
- 0 pts: No acceptance criteria

### Quality Score (15 pts)

| Check | Points | What We Look For |
|-------|--------|------------------|
| Testable format | +5 | Given/When/Then, should/must/verify |
| Technical specifics | +5 | API endpoints, field names, status codes |
| Edge cases | +3 | Error handling, validation failures |
| Expected values | +2 | Examples, sample data |

### Red Flags (Deduct Points)

These phrases indicate AC too vague for agent work:

| Red Flag | Problem |
|----------|---------|
| handle errors properly | No specific error codes/responses |
| make it fast | No performance target |
| should work correctly | Not testable |
| implement the feature | No details |
| fix the bug | No root cause or expected behavior |
| as per requirements | Requirements not in AC |
| TBD / TODO / TBC | Incomplete |

**Deduction:** -2 pts per red flag (max -5)

### Green Flags (Award Points)

| Green Flag | Example | Points |
|------------|---------|--------|
| Given/When/Then | Given valid input, When submitted, Then return 200 | +5 |
| API endpoint | POST /api/challenge/verify | +5 |
| Status code | return 400 with error | +5 |
| Field name | userId, challengeAnswer | +5 |
| Error case | if invalid, return INVALID_FORMAT | +3 |
| Example value | { verified: true } | +2 |

### AC Examples

**BAD AC (scores ~3/25):**
- Handle errors properly
- Make it fast
- User can submit form

**GOOD AC (scores 25/25):**
- Given a valid challenge answer, When user submits via POST /api/challenge/verify, Then return 200 with { verified: true }
- Given an invalid answer format, When submitted, Then return 400 with error code INVALID_FORMAT
- Response time must be < 200ms for 95th percentile

---

## 2. Description (20 points)

### Existence Score (8 pts)

- 8 pts: 50+ words with some context
- 5 pts: 20-49 words
- 2 pts: Under 20 words
- 0 pts: No description

### Quality Score (12 pts)

| Check | Points | What We Look For |
|-------|--------|------------------|
| User story format | +4 | As a [user], I want [goal], so that [benefit] |
| Technical context | +3 | Systems, APIs, databases mentioned |
| Scope boundaries | +3 | In-scope / out-of-scope stated |
| Examples | +2 | Sample data, example flows |

### Description Examples

**BAD Description (scores ~2/20):**
Add challenge question validation

**GOOD Description (scores 20/20):**

As a cardholder going through instant provisioning,
I want my challenge question answer validated in real-time,
So that I can proceed with card activation without delays.

**Context:** During instant card provisioning, Fiserv sends a callback with the user's challenge answer. We validate against Coppice and return pass/fail.

**Systems involved:**
- Acquisitions Platform (receives callback)
- Coppice (stores challenge Q/A)
- Fiserv (sends callback)

**Scope:**
- IN: Validation logic, API endpoint, error responses
- OUT: Challenge question creation, UI changes

**Example flow:**
1. Fiserv calls POST /api/challenge/verify
2. We lookup answer in Coppice
3. Return { verified: true/false }

---

## 3. Story Points (15 points)

- 15 pts: Valid Fibonacci (1, 2, 3, 5, 8, 13)
- 10 pts: Has points but not Fibonacci (4, 6, 7, etc.)
- 5 pts: Points > 13 (story too large)
- 0 pts: No story points

---

## 4. Epic Linkage (10 points)

- 10 pts: Formally linked to epic
- 5 pts: Epic mentioned but not linked
- 0 pts: No epic linkage (orphan)

---

## 5. Labels (10 points)

- 10 pts: Has workflow label + other labels
- 5 pts: Has some labels, no workflow label
- 0 pts: No labels

State labels (mutually exclusive): backlog, groomed, needs-info, ready, in-progress, ready-for-test
Work type labels (persistent): agent, human

---

## 6. Dependencies (10 points)

- 10 pts: Has blocked-by links OR states No dependencies
- 5 pts: Dependencies mentioned but not linked
- 0 pts: No mention of dependencies

---

## 7. Assignee (5 points)

- 5 pts: Has assignee OR not in active sprint
- 3 pts: In sprint but unassigned
- 0 pts: In sprint, unassigned, blocking work

---

## 8. Sprint Fit (5 points)

- 5 pts: Points <= 8 AND AC count <= 6
- 3 pts: Points <= 13 AND AC <= 8
- 0 pts: Points > 13 OR AC > 8

---

## Quality Patterns Reference

### Red Flag Regex Patterns

handle errors? (properly|correctly)
make it (fast|quick|performant)
should (work|function) (correctly|properly)
implement (the|this) (feature|functionality)
fix (the|this) (bug|issue)
TBD|TODO|TBC
as per (requirements|specs)

### Green Flag Regex Patterns

**Testable:**
given .+ when .+ then
should (return|throw|create|update|delete)
must (return|validate|check|verify)

**Technical:**
(GET|POST|PUT|DELETE) /[path]
(status|code) [1-5][0-9][0-9]
(field|column|table) [name]

**Edge Cases:**
(if|when) (invalid|null|empty|missing)
error (code|message) [value]
(timeout|retry|fallback)

**Examples:**
(example|e.g.|sample)
{ key: value }
returns { ... }
