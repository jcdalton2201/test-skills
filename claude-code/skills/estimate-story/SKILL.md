---
name: estimate-story
description: |
  The `estimate-story` skill analyzes Tracker stories and assigns Fibonacci story points (1, 2, 3, 5, 8, 13).
  Story points loosely equate to days of effort. Can estimate single stories, bulk stories in an epic,
  or find and estimate all unestimated stories in a project. Other skills should reference this skill
  for consistent estimation across workflows.
---

# estimate-story: Implementation Guide

This guide documents the workflow, estimation heuristics, and integration points for the `estimate-story` skill.

---

## Overview

The `estimate-story` skill assigns **Fibonacci story points** to Tracker stories based on intelligent analysis of story content. Story points loosely equate to days of effort.

**Valid Story Points**: 1, 2, 3, 5, 8, 13 (Fibonacci sequence, max 13)

**Key Principle**: If a story estimates > 13 points, it should be split into smaller stories.

**Note**: The skill sets BOTH the "Story Points" and "Story Point Estimate" fields in Tracker to ensure consistency across different Tracker views and reports.

---

## Skill Responsibilities

The skill must:

1. Analyze story content (description, acceptance criteria, technical notes).
2. Apply consistent Fibonacci estimation heuristics.
3. Provide rationale for each estimate.
4. Support single-story, bulk (epic), and project-wide estimation.
5. Show estimates for confirmation before applying.
6. Use `set_story_points` or `estimate_story_points` Tracker tools.
7. Flag stories that exceed 13 points for splitting.

The skill must not:

- Assign non-Fibonacci values (e.g., 4, 6, 7, 10).
- Assign values greater than 13.
- Apply estimates without showing rationale.
- Silently skip stories with errors.

---

## Invocation Modes

### Mode 1: Single Story

```text
Estimate story CARD-101
```

```text
Estimate CARD-101
```

### Mode 2: All Stories in an Epic

```text
Estimate stories in epic CARD-100
```

```text
Estimate all stories under CARD-100
```

### Mode 3: Unestimated Stories in Project

```text
Estimate unestimated stories in project CARD
```

```text
Find and estimate stories without points in CARD
```

### Mode 4: Specific Stories (List)

```text
Estimate stories CARD-101, CARD-102, CARD-103
```

---

## Part 1: Story Point Reference

### Fibonacci Scale

| Points | Effort | Description | Typical Indicators |
|--------|--------|-------------|-------------------|
| **1** | ~1 day | Trivial | Config change, small fix, single file |
| **2** | ~2 days | Small | Well-understood, 1-2 AC, single component |
| **3** | ~3 days | Medium-small | Typical story, 3-4 AC, clear scope |
| **5** | ~5 days | Medium | ~1 week, 5-6 AC, multiple components |
| **8** | ~8 days | Large | >1 week, 7+ AC, cross-system integration |
| **13** | ~13 days | Very large | Complex, many unknowns, consider splitting |

### When to Split (> 13 points)

If analysis suggests > 13 points, the skill should:

1. Flag the story as "too large to estimate".
2. Recommend splitting into smaller stories.
3. Suggest logical split points based on AC groupings.
4. Not assign a point value until split.

---

## Part 2: Estimation Heuristics

### Primary Factors

Analyze these factors to determine story points:

#### 1. Acceptance Criteria Count

| AC Count | Base Points | Notes |
|----------|-------------|-------|
| 1-2 | 1-2 | Trivial to small |
| 3-4 | 3 | Typical story |
| 5-6 | 5 | Medium complexity |
| 7-8 | 8 | Large story |
| 9+ | 13+ | Consider splitting |

#### 2. Description Complexity

| Indicator | Point Modifier |
|-----------|---------------|
| Single component/service | +0 |
| Multiple components | +1-2 |
| Cross-system integration | +2-3 |
| External API/vendor | +2-3 |
| Database schema changes | +1-2 |
| Security/compliance requirements | +1-2 |
| UI + Backend + API | +2-3 |

#### 3. Uncertainty Indicators

| Indicator | Point Modifier |
|-----------|---------------|
| Clear, well-defined scope | +0 |
| "TBD" or "to be determined" | +2 |
| Open questions listed | +1-2 |
| New technology/pattern | +2-3 |
| Missing technical details | +1-2 |
| "Spike" or "research" mentioned | +2 |

#### 4. Complexity Keywords

Scan for keywords that indicate complexity:

**High complexity** (+2-3 points):
- `integration`, `migrate`, `refactor`, `security`, `encryption`
- `compliance`, `audit`, `PCI`, `PII`, `authentication`
- `distributed`, `async`, `event-driven`, `saga`
- `performance`, `optimization`, `caching`

**Medium complexity** (+1-2 points):
- `API`, `endpoint`, `database`, `schema`
- `validation`, `error handling`, `retry`
- `configuration`, `feature flag`

**Low complexity** (+0 points):
- `fix`, `update`, `change`, `add field`
- `logging`, `documentation`, `test`

### Estimation Formula

```
Base Points = AC_Count_Points
Adjusted Points = Base + Complexity_Modifiers + Uncertainty_Modifiers
Final Points = nearest_fibonacci(Adjusted Points)
```

If `Final Points > 13`, flag for splitting.

---

## Part 3: Analysis Workflow

### Step 1: Fetch Story Details

```javascript
const issue = await get_issue({ issueKey: "CARD-101" });
```

Extract:
- Summary (title)
- Description (full text)
- Acceptance criteria (count and content)
- Labels
- Current story points (if any)

### Step 2: Parse Acceptance Criteria

Count AC items by looking for:
- `- [ ]` checkboxes
- `- ` bullet points under "Acceptance Criteria" section
- Numbered lists under AC section

### Step 3: Analyze Complexity

Scan description for:
- Complexity keywords (see Part 2)
- System/component mentions
- Integration points
- Security/compliance mentions
- Uncertainty indicators

### Step 4: Calculate Estimate

```javascript
function estimateStoryPoints(story) {
  let basePoints = getBasePointsFromACCount(story.acCount);
  let modifiers = 0;
  
  // Add complexity modifiers
  modifiers += analyzeComplexityKeywords(story.description);
  modifiers += analyzeIntegrationPoints(story.description);
  modifiers += analyzeUncertainty(story.description);
  
  const rawEstimate = basePoints + modifiers;
  const fibonacciEstimate = nearestFibonacci(rawEstimate);
  
  if (fibonacciEstimate > 13) {
    return { points: null, needsSplit: true, rawEstimate };
  }
  
  return { points: fibonacciEstimate, needsSplit: false, rawEstimate };
}
```

### Step 5: Generate Rationale

For each estimate, generate a rationale:

```markdown
**Estimate**: 5 story points (~5 days)

**Rationale**:
- Acceptance criteria: 5 items (base: 5 points)
- Integration: Involves external API (+2)
- Complexity: Standard CRUD operations (+0)
- Uncertainty: Clear requirements (+0)
- Raw score: 7 → Nearest Fibonacci: 5
```

---

## Part 4: Confirmation Flow

### Single Story Confirmation

```markdown
## Story Point Estimate: CARD-101

**Story**: Add POST redirect support to Coppice
**Current Points**: None

### Analysis

| Factor | Value | Points |
|--------|-------|--------|
| Acceptance Criteria | 4 items | 3 |
| Integration | Single service | +0 |
| Complexity | API + validation | +1 |
| Uncertainty | Clear scope | +0 |
| **Raw Total** | | **4** |
| **Fibonacci** | | **5** |

### Recommendation

**5 story points** (~5 days)

**Rationale**: Medium complexity with 4 AC items, involves API changes and request validation. Well-defined scope with no major unknowns.

---

Apply this estimate? (yes/no/adjust)
```

### Bulk Confirmation (Epic)

```markdown
## Story Point Estimates: Epic CARD-100

**Epic**: Instant Card Provisioning
**Stories**: 6 total, 4 unestimated

### Estimates

| Key | Story | AC | Complexity | Points | Status |
|-----|-------|----|-----------:|-------:|--------|
| CARD-101 | Challenge Question Validation | 4 | Medium | **3** | New |
| CARD-102 | SMS Content Generation | 5 | Medium | **5** | New |
| CARD-103 | Transaction Web Platform | 6 | High | **8** | New |
| CARD-104 | Fiserv Wallet Integration | 8 | High | **13** | ⚠️ Consider split |
| CARD-105 | Edge Cases & Exception Handling | 3 | Low | **3** | Already set |
| CARD-106 | Monitoring & Alerting | 4 | Medium | **5** | New |

### Summary

- **Total stories**: 6
- **To estimate**: 4
- **Already estimated**: 2
- **Needs splitting**: 1 (CARD-104)
- **Total points (new)**: 29

---

Apply these estimates? (yes/no/adjust [KEY] [POINTS])
```

### Adjustment Commands

User can adjust before applying:

```text
adjust CARD-102 3
```

```text
skip CARD-104
```

```text
yes
```

---

## Part 5: Applying Estimates

### Single Story

```javascript
await set_story_points({
  issueKey: "CARD-101",
  points: 5
});

// Add comment with rationale
await add_comment({
  issueKey: "CARD-101",
  comment: `**Story Points Estimate**: 5

Estimated by \`estimate-story\` skill.

**Rationale**:
- Acceptance criteria: 4 items
- Integration: Single service
- Complexity: API + validation
- Uncertainty: Clear scope

~5 days estimated effort.`
});
```

### Bulk Stories

```javascript
const estimates = [
  { key: "CARD-101", points: 3 },
  { key: "CARD-102", points: 5 },
  { key: "CARD-103", points: 8 },
  { key: "CARD-106", points: 5 },
];

for (const { key, points } of estimates) {
  await set_story_points({ issueKey: key, points });
  await add_comment({
    issueKey: key,
    comment: `**Story Points**: ${points}\n\nEstimated by \`estimate-story\` skill. ~${points} days effort.`
  });
}
```

---

## Part 6: Output Summary

### Single Story Output

```markdown
✅ Estimate Applied: CARD-101

- **Story**: Add POST redirect support to Coppice
- **Story Points**: 5 (set in both Story Points and Story Point Estimate fields)
- **Estimated Effort**: ~5 days
- **Rationale**: Medium complexity, 4 AC items, API changes

Comment added to story with rationale.
```

### Bulk Output

```markdown
✅ Estimates Applied: Epic CARD-100

| Key | Story Points | Status |
|-----|-------------:|--------|
| CARD-101 | 3 | ✅ Applied |
| CARD-102 | 5 | ✅ Applied |
| CARD-103 | 8 | ✅ Applied |
| CARD-104 | - | ⚠️ Skipped (needs split) |
| CARD-106 | 5 | ✅ Applied |

**Summary**:
- Applied: 4 stories
- Skipped: 1 story (needs splitting)
- Total points: 21

**Next Steps**:
- CARD-104 needs to be split into smaller stories before estimation.
```

---

## Part 7: Integration with Other Skills

Other skills should reference `estimate-story` for consistent estimation.

### From `brd-to-issues`

After creating stories:

```markdown
## After Story Creation

Once all stories are created, invoke `estimate-story` to set points:

> Estimate stories in epic CARD-100

Or estimate individually as each story is created.
```

### From `create-story`

After creating a single story:

```markdown
## After Story Creation

Invoke `estimate-story` to analyze and set points:

> Estimate story CARD-101

Or ask the user for a days estimate and use `estimate_story_points` tool directly.
```

### From `story-grooming`

When grooming reveals missing estimates:

```markdown
## During Grooming

If the parent story lacks points, invoke:

> Estimate story CARD-101

Then proceed with task breakdown.
```

### Direct Tool Usage (Alternative)

If another skill has already determined the days estimate, it can use the Tracker tool directly:

```javascript
// If you already know the days estimate
await estimate_story_points({
  issueKey: "CARD-101",
  estimatedDays: 4.5,  // Converts to nearest Fibonacci (5)
  comment: "Estimated during story creation"
});

// If you already know the exact Fibonacci points
await set_story_points({
  issueKey: "CARD-101",
  points: 5
});
```

---

## Part 8: Finding Unestimated Stories

### JQL Queries

Find stories without story points:

```javascript
// Stories in epic without points
const jql = `"Epic Link" = CARD-100 AND "Story Points" IS EMPTY AND issuetype = Story`;

// All unestimated stories in project
const jql = `project = CARD AND "Story Points" IS EMPTY AND issuetype = Story`;

// Unestimated stories ready for development (agent work)
const jql = `project = CARD AND "Story Points" IS EMPTY AND labels = "ready" AND labels = "agent"`;
```

### Workflow

```javascript
// 1. Find unestimated stories
const results = await search_issues({
  jql: `"Epic Link" = ${epicKey} AND "Story Points" IS EMPTY AND issuetype = Story`
});

// 2. Analyze each story
const estimates = [];
for (const story of results.issues) {
  const estimate = analyzeStory(story);
  estimates.push({
    key: story.key,
    summary: story.fields.summary,
    ...estimate
  });
}

// 3. Present for confirmation
presentEstimates(estimates);

// 4. Apply confirmed estimates
applyEstimates(confirmedEstimates);
```

---

## Part 9: Error Handling

| Error | Behavior |
|-------|----------|
| Story not found | Report error, continue with other stories in bulk mode |
| Story already estimated | Show current value, ask if should overwrite |
| Invalid story type (Epic, Sub-task) | Skip with message "Story points apply to Stories only" |
| Analysis fails | Use conservative estimate (5), flag for review |
| Tracker API error | Report error, do not mark as estimated |
| Story needs splitting (>13) | Flag but do not assign points |

### No Silent Failures

Always report:
- Which stories were estimated
- Which stories were skipped (and why)
- Which stories need attention

---

## Part 10: Metadata Output

Save `.estimate-story-output.json` after bulk operations:

```json
{
  "skill": "estimate-story",
  "timestamp": "2026-05-14T10:30:00Z",
  "mode": "epic",
  "epic_key": "CARD-100",
  "estimates": [
    {
      "key": "CARD-101",
      "summary": "Challenge Question Validation",
      "ac_count": 4,
      "complexity": "medium",
      "raw_score": 4,
      "story_points": 3,
      "status": "applied",
      "rationale": "4 AC items, single service integration, clear scope"
    },
    {
      "key": "CARD-104",
      "summary": "Fiserv Wallet Integration",
      "ac_count": 8,
      "complexity": "high",
      "raw_score": 15,
      "story_points": null,
      "status": "needs_split",
      "rationale": "Too large (15 raw points). Recommend splitting by integration phase."
    }
  ],
  "summary": {
    "total_stories": 6,
    "estimated": 4,
    "skipped": 1,
    "already_estimated": 1,
    "total_points": 21
  }
}
```

---

## Part 11: Testing Checklist

Before deploying this skill, verify:

- [ ] Skill correctly counts acceptance criteria from various formats.
- [ ] Skill identifies complexity keywords accurately.
- [ ] Skill applies Fibonacci rounding correctly.
- [ ] Skill flags stories > 13 points for splitting.
- [ ] Skill shows estimates for confirmation before applying.
- [ ] Skill handles adjustment commands (adjust, skip).
- [ ] Skill adds rationale comments to stories.
- [ ] Skill handles already-estimated stories appropriately.
- [ ] Skill handles bulk operations (epic, project).
- [ ] Skill handles errors gracefully (story not found, API errors).
- [ ] Skill saves metadata output for bulk operations.
- [ ] Skill can be invoked by other skills (integration test).

---

## Part 12: Example End-to-End Run

### User Input

```text
Estimate stories in epic CARD-100
```

### Skill Output

```markdown
## Analyzing Epic CARD-100: Instant Card Provisioning

Found 6 stories, 4 need estimation.

### Analysis Results

| Key | Story | AC | Raw | Points | Notes |
|-----|-------|---:|----:|-------:|-------|
| CARD-101 | Challenge Question Validation | 4 | 4 | **3** | Clear scope |
| CARD-102 | SMS Content Generation | 5 | 6 | **5** | External SMS API |
| CARD-103 | Transaction Web Platform | 6 | 9 | **8** | UI + Backend + API |
| CARD-104 | Fiserv Wallet Integration | 8 | 15 | **-** | ⚠️ Needs split |
| CARD-105 | Edge Cases | 3 | 3 | **3** | ✓ Already set |
| CARD-106 | Monitoring & Alerting | 4 | 5 | **5** | Metrics + alerts |

### Summary

- **To estimate**: 4 stories
- **Total new points**: 21
- **Needs attention**: CARD-104 (too large)

Apply these estimates? (yes/no/adjust)
```

### User Confirms

```text
yes
```

### Final Output

```markdown
✅ Estimates Applied

| Key | Points | Status |
|-----|-------:|--------|
| CARD-101 | 3 | ✅ Applied |
| CARD-102 | 5 | ✅ Applied |
| CARD-103 | 8 | ✅ Applied |
| CARD-104 | - | ⚠️ Skipped |
| CARD-106 | 5 | ✅ Applied |

Comments added to each story with rationale.

**Next Steps**:
- Review and split CARD-104 into smaller stories.
- Then run: `Estimate story CARD-104-A` (after split)

Metadata saved to `.estimate-story-output.json`
```

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-05-15 | Initial implementation guide |
