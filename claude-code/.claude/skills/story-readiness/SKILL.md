---
name: story-readiness
description: |
  The story-readiness skill audits Jira stories and epics for development readiness. It scores tickets (0-100%), identifies gaps, and posts actionable feedback. Stories scoring below 80% are marked needs-info with a detailed readiness report.
---

# story-readiness: Implementation Guide

## Overview

The story-readiness skill audits Jira stories for development readiness by:

1. **Scoring** stories on a 0-100% scale based on completeness criteria
2. **Identifying gaps** with specific, actionable feedback
3. **Posting comments** to Jira for stories that need work (<80%)
4. **Applying labels** to track readiness state
5. **Integrating** with other skills (estimate-story, grill-me, story-grooming)

**Threshold**: Stories scoring >=80% are ready. Stories <80% get needs-info label + readiness report comment.

---

## Label Integration

The system uses two types of labels:

### State Labels (Mutually Exclusive)

Remove the previous state when transitioning.

| Label | Meaning | Applied When |
|-------|---------|---------------|
| `backlog` | Ungroomed story | Initial creation |
| `groomed` | Story has been groomed, ready for readiness check | After initial grooming |
| `ready` | Ready for development | Score >= 80% |
| `needs-info` | Story needs clarification | Score < 80% OR missing info |
| `in-progress` | Work has started | Agent or human picks up the story |
| `ready-for-test` | Implementation complete | All AC implemented, tests passing |

**State flow:** `backlog` → `groomed` → `ready` / `needs-info` → `in-progress` → `ready-for-test`

### Work Type Labels (Persistent)

Set at triage, persists through all state transitions.

| Label | Meaning | Applied When |
|-------|---------|---------------|
| `agent` | Autonomous agent work (AFK) | Clear AC, no human judgment needed |
| `human` | Needs human judgment (HITL) | Requires design decisions or complex context |

### Workflow with Label Transitions

```
[Story arrives with `groomed` label]     ← Input: groomed stories
        │
        ↓
[Story Readiness Check]
        │
   ┌────┴──────────────────────────────────┐
   │                                       │
Score < 80%                           Score >= 80%
   │                               ┌───────┴───────┐
   ↓                               │               │
-[groomed]                     Clear AC?     Design needed?
+[needs-info]                      │               │
   │                               ↓               ↓
   ↓                         -[groomed]      -[groomed]
[Fix issues]                 +[ready]        +[ready]
   │                         +[agent]        +[human]
   └─────→[Re-check]
```

### Transition Rules

**When score < 80%:**
- Remove: `groomed` (if present)
- Remove: `ready` (if present)
- Apply: `needs-info`
- Post readiness report comment
- Note: `agent`/`human` labels are NOT removed (they persist)

**When score >= 80% with clear AC:**
- Remove: `groomed`, `needs-info` (if present)
- Apply: `ready`
- **MUST apply: `agent`** ← This is REQUIRED, not optional!

**When score >= 80% but needs human judgment:**
- Remove: `groomed`, `needs-info` (if present)
- Apply: `ready`
- **MUST apply: `human`** ← This is REQUIRED, not optional!

**IMPORTANT:** Every triaged story MUST have BOTH:
1. A state label (`ready` or `needs-info`)
2. A work type label (`agent` or `human`)

---

## Invocation

Check readiness for CARD-101
Check readiness for epic CARD-100
Audit stories in sprint Sprint 42
Find stories that need readiness checks

---

## Scoring Criteria (100 Points)

| Criteria | Points | What We Check |
|----------|--------|---------------|
| **Acceptance Criteria** | 25 | Existence (10) + Agent-Ready Quality (15) |
| **Description** | 20 | Existence (8) + Agent-Ready Quality (12) |
| Story Points | 15 | Has valid Fibonacci estimate (1-13)? |
| Epic Linkage | 10 | Linked to parent epic? |
| Labels | 10 | Has workflow labels? |
| Dependencies | 10 | Dependencies identified or none stated? |
| Assignee | 5 | Has assignee (if in active sprint)? |
| Sprint Fit | 5 | Points <=8 and AC <=6? |

### Agent-Readiness Quality Checks

The key difference: we don't just check that AC/descriptions *exist*, we check if they have *enough detail for an agent to implement*.

#### AC Quality Scoring (15 pts)

| Check | Points | What We Look For |
|-------|--------|------------------|
| Testable format | +5 | Given/When/Then, "should", "must", "verify" |
| Technical specifics | +5 | API endpoints, field names, tables, status codes |
| Edge cases | +3 | Error handling, validation failures, boundaries |
| Expected values | +2 | Examples, sample data, specific values |

**Red Flags (deduct points):**
- "handle errors properly" → vague
- "make it fast" → no specific target
- "should work correctly" → not testable
- "implement the feature" → no details

**Green Flags (award points):**
- `POST /api/challenge/verify` → specific endpoint
- `return 400 with error code "INVALID_FORMAT"` → specific response
- `Given a valid answer, When submitted, Then return 200` → testable
- `Response time < 200ms` → measurable

#### Description Quality Scoring (12 pts)

| Check | Points | What We Look For |
|-------|--------|------------------|
| User story format | +4 | "As a [user], I want [goal], so that [benefit]" |
| Technical context | +3 | Systems, APIs, databases, services mentioned |
| Scope boundaries | +3 | In-scope / out-of-scope stated |
| Examples | +2 | Sample data, example flows, concrete scenarios |

### Thresholds

| Score | Status | Action |
|-------|--------|--------|
| 90-100% | Ready | Add `ready` + `agent` or `ready` + `human` |
| 80-89% | Ready | Add `ready` + `agent` or `ready` + `human`, note minor gaps |
| 50-79% | Not Ready | Add `needs-info`, post report |
| 0-49% | Incomplete | Add `needs-info`, post report |

---

## Implementation

### Step 1: Fetch Story Data

```javascript
const issue = await jira.get_issue({ issueKey: 'CARD-101' });
const storyPoints = await jira.get_story_points({ issueKey: 'CARD-101' });
```

### Step 2: Calculate Score with Quality Checks

```javascript
// Red flag patterns - vague language that agents can't work with
const RED_FLAGS = [
  /handle\s+(errors?|exceptions?)\s+(properly|correctly|appropriately)/i,
  /make\s+it\s+(fast|quick|performant|efficient)/i,
  /should\s+(work|function|behave)\s+(correctly|properly|as expected)/i,
  /implement\s+(the|this)\s+(feature|functionality|requirement)/i,
  /fix\s+(the|this)\s+(bug|issue|problem)/i,
  /improve\s+(the|this)\s+(performance|speed|efficiency)/i,
  /ensure\s+(it|this)\s+(works|is correct)/i,
  /do\s+(the|this)\s+(needful|necessary)/i,
  /as\s+per\s+(requirements?|specs?|design)/i,
  /TBD|TODO|TBC|to be (determined|decided|confirmed)/i,
];

// Green flag patterns - specific details agents can work with
const GREEN_FLAGS = {
  testable: [
    /given\s+.+,?\s*when\s+.+,?\s*then/i,           // Given/When/Then
    /should\s+(return|throw|emit|log|create|update|delete)/i,
    /must\s+(return|validate|check|verify|ensure)/i,
    /verify\s+(that|the)/i,
    /expect(s|ed)?\s+(to|that)/i,
  ],
  technical: [
    /(GET|POST|PUT|PATCH|DELETE)\s+\/[\w\/\-{}:]+/i, // API endpoints
    /\b(status|code)\s*(:|=)?\s*[1-5]\d{2}\b/i,      // HTTP status codes
    /\b(field|column|property|attribute)\s*[:=]?\s*[\w_]+/i,
    /\b(table|collection|schema)\s*[:=]?\s*[\w_]+/i,
    /\{\s*["']?\w+["']?\s*:/,                         // JSON structure
    /\b(endpoint|api|service|database|queue|cache)\b/i,
  ],
  edgeCases: [
    /\b(if|when)\s+(invalid|null|empty|missing|malformed)/i,
    /\berror\s*(code|message|response)\s*[:=]/i,
    /\b(timeout|retry|fallback|rollback)\b/i,
    /\b(400|401|403|404|500|502|503)\b/,             // Error status codes
    /\b(validation|validates?)\s+(fails?|error)/i,
    /\b(edge\s+case|boundary|limit|max|min)\b/i,
  ],
  examples: [
    /\b(example|e\.g\.|for instance|such as)\s*[:=]?/i,
    /\b(sample|test)\s+(data|input|output|payload)/i,
    /["']\w+["']\s*[:=]\s*["']?[\w\-@.]+["']?/,      // Key-value examples
    /\b(returns?|response)\s*[:=]?\s*\{/i,            // Response examples
  ],
};

function scoreACQuality(acceptanceCriteria) {
  let quality = { testable: 0, technical: 0, edgeCases: 0, examples: 0 };
  let redFlagCount = 0;
  const acText = acceptanceCriteria.join(' ');
  
  // Check for red flags
  for (const pattern of RED_FLAGS) {
    if (pattern.test(acText)) redFlagCount++;
  }
  
  // Check for green flags
  for (const [category, patterns] of Object.entries(GREEN_FLAGS)) {
    for (const pattern of patterns) {
      if (pattern.test(acText)) {
        quality[category]++;
        break; // Only count once per category per AC
      }
    }
  }
  
  // Calculate quality score (max 15)
  let score = 0;
  score += quality.testable > 0 ? 5 : 0;      // Testable format
  score += quality.technical > 0 ? 5 : 0;     // Technical specifics  
  score += quality.edgeCases > 0 ? 3 : 0;     // Edge cases
  score += quality.examples > 0 ? 2 : 0;      // Expected values
  
  // Deduct for red flags (max -5)
  score -= Math.min(redFlagCount * 2, 5);
  
  return {
    score: Math.max(0, score),
    maxScore: 15,
    quality,
    redFlagCount,
    issues: getACQualityIssues(quality, redFlagCount)
  };
}

function scoreDescriptionQuality(description) {
  let score = 0;
  const issues = [];
  
  // User story format (+4)
  const hasUserStory = /as\s+an?\s+[\w\s]+,?\s*i\s+want/i.test(description) ||
                       /so\s+that\s+/i.test(description);
  if (hasUserStory) {
    score += 4;
  } else {
    issues.push('Add user story format: "As a [user], I want [goal], so that [benefit]"');
  }
  
  // Technical context (+3)
  const hasTechnical = /(api|endpoint|service|database|table|queue|system|platform)/i.test(description);
  if (hasTechnical) {
    score += 3;
  } else {
    issues.push('Add technical context: which systems, APIs, or databases are involved');
  }
  
  // Scope boundaries (+3)
  const hasScope = /(in.scope|out.of.scope|scope:|excludes?:|includes?:)/i.test(description) ||
                   /(this\s+(does|will)\s+not|not\s+part\s+of)/i.test(description);
  if (hasScope) {
    score += 3;
  } else {
    issues.push('Add scope boundaries: what is in-scope vs out-of-scope');
  }
  
  // Examples (+2)
  const hasExamples = /(example|e\.g\.|sample|for instance)/i.test(description) ||
                      /```|\{.*:.*\}/s.test(description);
  if (hasExamples) {
    score += 2;
  } else {
    issues.push('Add examples: sample data, example flows, or concrete scenarios');
  }
  
  return { score, maxScore: 12, issues };
}
```

### Step 3: Generate Report Comment

For scores < 80%, generate a markdown comment showing:
- Score percentage and status
- What needs work (gaps table)
- **Quality issues** (vague language, missing specifics)
- Next steps to fix

### Step 4: Update Jira

- Score < 80%: Remove previous state label, add `needs-info`, post comment
- Score >= 80%: Remove previous state label, add `ready` + work type label (`agent` or `human`)

---

## Batch Processing (Epic Audit)

Query all stories in an epic and score each. Show summary table sorted by score (worst first).

Ask for confirmation before posting comments to failing stories.

---

## Skill Integration

| Gap Found | Offer | Command |
|-----------|-------|---------|
| Missing story points | Estimate? | /estimate-story CARD-101 |
| Vague requirements | Grill for details? | /grill-me CARD-101 |
| Story too large | Break it down? | /story-grooming CARD-101 |

---

## Comment Template

When score < 80%, post this comment to Jira:

```markdown
> *This readiness assessment was generated by AI.*

## 📋 Story Readiness Report: {ISSUE_KEY}

**Score: {SCORE}% {EMOJI} {STATUS}**

### ✅ What's Complete
{COMPLETE_ITEMS}

### ❌ What Needs Work
| Gap | Impact | How to Fix |
|-----|--------|------------|
{GAP_ROWS}

### 📝 Recommended Next Steps
{NEXT_STEPS}

---
*Re-run readiness check after updates. Target: 80% or higher.*
```

**Variables:**
- `{EMOJI}`: ✅ (>=80%), ❌ (50-79%), 🚫 (<50%)
- `{STATUS}`: "READY FOR DEVELOPMENT" or "NOT READY FOR DEVELOPMENT"
- `{COMPLETE_ITEMS}`: Bullet list of passing criteria
- `{GAP_ROWS}`: Table rows for each gap
- `{NEXT_STEPS}`: Numbered list of fixes, highest impact first

---

## Example Session

### Example 1: Story with Vague AC

```
User: Check readiness for CARD-101

Skill: 📋 **CARD-101: Challenge Question Validation**
       
       **Score: 47% 🚫 NOT READY FOR DEVELOPMENT**
       
       ✅ What's Complete:
       - Has description (85 words)
       - Linked to epic CARD-100
       - Has labels (slice-1)
       
       ❌ What Needs Work:
       | Gap | Impact | How to Fix |
       |-----|--------|------------|
       | AC Quality | -12% | AC too vague for agent work (see details) |
       | Story Points | -15% | Add estimate (1-13) |
       | Description Quality | -8% | Missing technical context and scope |
       | Dependencies | -10% | Identify blockers or note "None" |
       
       🚩 **Agent-Readiness Issues Found:**
       
       **Acceptance Criteria Problems:**
       - ⚠️ Red flag: "handle errors properly" - too vague, specify error codes/responses
       - ⚠️ Red flag: "make it fast" - add specific target (e.g., "< 200ms")
       - ❌ Missing: No API endpoints or field names specified
       - ❌ Missing: No edge cases (what if validation fails?)
       - ❌ Missing: No expected values or examples
       
       **Description Problems:**
       - ❌ Missing: User story format (As a... I want... So that...)
       - ❌ Missing: Technical context (which systems/APIs involved?)
       - ❌ Missing: Scope boundaries (what's in/out of scope?)
       
       Should I post this report to Jira and mark `needs-info`?

User: Yes

Skill: ✅ Posted readiness report to CARD-101
       ✅ Labels: -groomed, +needs-info
       (work type label `agent` preserved if already set)
       
       Would you like me to run `/grill-me CARD-101` to help extract the missing details?
```

### Example 2: Story with Good Quality (Clear AC)

```
User: Check readiness for CARD-102

Skill: 📋 **CARD-102: SMS Content Generation**
       
       **Score: 92% ✅ READY FOR DEVELOPMENT**
       
       ✅ What's Complete:
       - Acceptance Criteria: 5 testable AC with Given/When/Then format
       - AC Quality: Has API endpoints, error codes, and examples
       - Description: User story format with technical context
       - Story Points: 5 (valid Fibonacci)
       - Epic Link: CARD-100
       - Dependencies: Blocked by CARD-101
       
       ⚠️ Minor suggestions (optional):
       - Could add more edge case coverage for rate limiting
       
       ✅ Labels: -groomed, +ready, +agent
       
       Story is ready for development! Run `/story-grooming CARD-102`?
```

### Example 3: Story Needing Human Judgment

```
User: Check readiness for CARD-103

Skill: 📋 **CARD-103: User Dashboard Redesign**
       
       **Score: 85% ✅ READY FOR DEVELOPMENT**
       
       ✅ What's Complete:
       - Acceptance Criteria: 4 testable AC
       - Description: User story format with context
       - Story Points: 8 (valid Fibonacci)
       - Epic Link: CARD-100
       
       ⚠️ Requires human judgment:
       - UI/UX design decisions needed
       - Stakeholder sign-off on layout
       
       ✅ Labels: -groomed, +ready, +human
       
       Story is ready for human development!
```

---

## JQL Queries

### Find stories needing readiness check (input to this skill)
```jql
project = "PROJECT_KEY" 
AND labels = "groomed" 
ORDER BY created ASC
```

### Find stories marked needs-info (failed readiness)
```jql
project = "PROJECT_KEY" 
AND labels = "needs-info" 
ORDER BY updated DESC
```

### Find ready stories for agent work
```jql
project = "PROJECT_KEY" 
AND labels = "ready" 
AND labels = "agent"
ORDER BY updated DESC
```

### Find ready stories for human work
```jql
project = "PROJECT_KEY" 
AND labels = "ready" 
AND labels = "human"
ORDER BY updated DESC
```

### Find stories in epic for audit
```jql
"Epic Link" = CARD-100 OR parent = CARD-100
ORDER BY created ASC
```

---

## Error Handling

| Error | Response |
|-------|----------|
| Issue not found | "CARD-999 not found. Check the issue key?" |
| No permission | "Cannot access CARD-101. Check Jira permissions." |
| Epic has no stories | "Epic CARD-100 has no linked stories." |
| Already has needs-info | "CARD-101 already marked needs-info. Re-score anyway?" |

---

## Testing Checklist

- [ ] Single story: Correctly calculates score
- [ ] Score < 80%: Posts comment with readiness report
- [ ] Score < 80%: Adds `needs-info` label
- [ ] Score < 80%: Removes `groomed` label (if present)
- [ ] Score < 80%: Removes `ready` label (if present)
- [ ] Score < 80%: Preserves `agent`/`human` work type labels
- [ ] Score >= 80%: Adds `ready` label
- [ ] Score >= 80%: Adds `agent` or `human` work type label (if not already set)
- [ ] Score >= 80%: Removes `groomed` and `needs-info` labels (if present)
- [ ] Epic audit: Shows all stories with scores
- [ ] Epic audit: Asks confirmation before batch posting
- [ ] Offers skill integration (estimate-story, grill-me, story-grooming)
- [ ] Handles missing fields gracefully (partial scores)
