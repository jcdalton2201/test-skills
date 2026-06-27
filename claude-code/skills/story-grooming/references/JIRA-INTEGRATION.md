# Jira Integration Guide

This guide documents how the `story-grooming` skill and triage workflows integrate with your Jira instance via the `js/jira/index.js` client.

---

## Label Mapping

### Triage Workflow Labels

The triage workflow uses Jira labels to track state and category. Standardize on these label names in your Jira project:

#### Category Labels
- `bug` — Issue is a bug (something broken)
- `enhancement` — Issue is a feature request or improvement

#### State Labels (Mutually Exclusive)

State labels represent the current workflow state. **Only one state label should be active at a time.** Remove the previous state label when transitioning.

| Label | Meaning | Set By |
|-------|---------|--------|
| `backlog` | Issue is in backlog, ready for grooming | `brd-to-issues` |
| `groomed` | Issue is groomed, ready for triage | `story-grooming` |
| `ready` | Ready for implementation | Triage (combine with `agent` or `human`) |
| `needs-info` | Blocked by missing information | `story-readiness`, triage, `rust-tdd` |
| `in-progress` | Work has started | `rust-tdd` or human |
| `ready-for-test` | Implementation complete | `rust-tdd` or human |

#### Work Type Labels (Persistent)

Work type labels are set at triage and **persist through all states**:

| Label | Meaning | Set By |
|-------|---------|--------|
| `agent` | Work suitable for autonomous agent (AFK) | Triage |
| `human` | Work requiring human judgment (HITL) | Triage |

Other labels (not state labels):
- `wontfix` — Issue will not be actioned

**Important:** If multiple state labels are present, the skill will flag it and ask for clarification before proceeding.

### BRD-to-Issues Workflow Labels

The `brd-to-issues` skill uses labels to track BRD conversion and implementation status:

#### Epic Labels
- `brd-epic` — This is an Epic created from a BRD
- `brd:[brd-name]` — Tags the Epic with its source BRD name (e.g., `brd:instant-provisioning`)

#### Story Labels (Vertical Slices)
- `backlog` — Story is in backlog, ready for grooming
- `brd:[brd-name]` — Links the Story to its source BRD (same as parent Epic)
- `slice-N` — Tags the Story with its slice number (e.g., `slice-1`, `slice-2`)

**Example**:
```
Epic CARD-100 (Instant Card Provisioning)
  Labels: brd-epic, brd:instant-provisioning
  
Story CARD-101 (Slice 1: Challenge Question Validation)
  Labels: backlog, brd:instant-provisioning, slice-1
  
Story CARD-102 (Slice 2: SMS Content Generation)
  Labels: backlog, brd:instant-provisioning, slice-2
```

---

## Example JQL Queries

Use these JQL patterns to query your Jira project. Replace `PROJECT_KEY` with your project key.

### Find unlabeled issues (never triaged)
```jql
project = "PROJECT_KEY" 
AND labels is EMPTY 
ORDER BY created ASC
```

### Find backlog issues ready for grooming
```jql
project = "PROJECT_KEY" 
AND labels = "backlog" 
ORDER BY updated ASC
```

### Find groomed issues awaiting triage
```jql
project = "PROJECT_KEY" 
AND labels = "groomed" 
ORDER BY updated ASC
```

### Find issues needing info with recent reporter activity
```jql
project = "PROJECT_KEY" 
AND labels = "needs-info" 
AND comment >= -7d 
ORDER BY updated ASC
```

### Find all ready agent issues
```jql
project = "PROJECT_KEY" 
AND labels = "ready" 
AND labels = "agent" 
ORDER BY created ASC
```

### Find all ready human issues
```jql
project = "PROJECT_KEY" 
AND labels = "ready" 
AND labels = "human" 
ORDER BY created ASC
```

### Find all BRD-based Stories (slices)
```jql
project = "PROJECT_KEY" 
AND labels ~ "slice-" 
ORDER BY created ASC
```

### Find all issues from a specific BRD
```jql
project = "PROJECT_KEY" 
AND labels = "brd:instant-provisioning" 
ORDER BY created ASC
```

### Find Epics created from BRDs
```jql
project = "PROJECT_KEY" 
AND labels = "brd-epic" 
ORDER BY created DESC
```

---

## Jira Client Functions

All functions below come from `js/jira/index.js`. Import them in your skill scripts or integrations.

### searchIssues(jql, options)

Query issues using JQL.

**Signature:**
```javascript
searchIssues(jql, options = {})
```

**Parameters:**
- `jql` (string) — JQL query string
- `options` (object, optional):
  - `maxResults` (number, default 50) — Max issues to return
  - `fields` (string[], optional) — Fields to include (defaults to common triage fields)
  - `expand` (string[], optional) — Fields to expand (e.g., `changelog`)

**Returns:** `{issues: object[], total: number}`

**Example:**
```javascript
const { issues, total } = await searchIssues(
  'project = "PROJ" AND labels = "groomed"',
  { maxResults: 20 }
);
console.log(`Found ${total} issues`);
```

### getIssue(issueKey, options)

Get full details for a single issue.

**Signature:**
```javascript
getIssue(issueKey, options = {})
```

**Parameters:**
- `issueKey` (string) — Issue key (e.g., "PROJ-42")
- `options` (object, optional):
  - `fields` (string[], optional) — Fields to include
  - `expand` (string[], optional) — Fields to expand

**Returns:** Full issue object with fields

**Example:**
```javascript
const issue = await getIssue("PROJ-42");
console.log(issue.fields.summary);
console.log(issue.fields.labels);
```

### addComment(issueKey, commentText)

Post a comment to an issue.

**Signature:**
```javascript
addComment(issueKey, commentText)
```

**Parameters:**
- `issueKey` (string) — Issue key
- `commentText` (string) — Comment body (plain text, will be formatted by Jira)

**Returns:** Created comment object

**Example:**
```javascript
await addComment("PROJ-42", "This is a test comment");
```

**Note:** Always start generated comments with:
```
> *This was generated by AI during triage.*
```

### updateLabels(issueKey, addLabels, removeLabels)

Add and/or remove labels on an issue.

**Signature:**
```javascript
updateLabels(issueKey, addLabels = [], removeLabels = [])
```

**Parameters:**
- `issueKey` (string) — Issue key
- `addLabels` (string[], optional) — Labels to add
- `removeLabels` (string[], optional) — Labels to remove

**Returns:** void

**Example:**
```javascript
// Transition from backlog to groomed (after story-grooming)
await updateLabels("PROJ-42", ["groomed"], ["backlog"]);

// Transition from groomed to ready + agent (after triage)
await updateLabels("PROJ-42", ["ready", "agent"], ["groomed"]);

// Transition from groomed to ready + human (after triage)
await updateLabels("PROJ-42", ["ready", "human"], ["groomed"]);

// Transition from ready to in-progress (work type label persists)
await updateLabels("PROJ-42", ["in-progress"], ["ready"]);

// Mark as blocked (needs-info)
await updateLabels("PROJ-42", ["needs-info"], ["in-progress"]);

// Unblock and resume
await updateLabels("PROJ-42", ["in-progress"], ["needs-info"]);

// Complete implementation
await updateLabels("PROJ-42", ["ready-for-test"], ["in-progress"]);
```

### transitionIssue(issueKey, transitionId, fields)

Move an issue to a new workflow status.

**Signature:**
```javascript
transitionIssue(issueKey, transitionId, fields = {})
```

**Parameters:**
- `issueKey` (string) — Issue key
- `transitionId` (string) — Transition ID (get from `getTransitions()`)
- `fields` (object, optional) — Fields to update during transition

**Returns:** void

**Example:**
```javascript
const transitions = await getTransitions("PROJ-42");
const closeTransition = transitions.find(t => t.name === "Close");
if (closeTransition) {
  await transitionIssue("PROJ-42", closeTransition.id);
}
```

### getTransitions(issueKey)

Get available workflow transitions for an issue.

**Signature:**
```javascript
getTransitions(issueKey)
```

**Parameters:**
- `issueKey` (string) — Issue key

**Returns:** `object[]` — Array of transition objects with `id`, `name`, `to`

**Example:**
```javascript
const transitions = await getTransitions("PROJ-42");
transitions.forEach(t => {
  console.log(`${t.id}: ${t.name} → ${t.to.name}`);
});
```

---

## Epic & Story Creation (BRD-to-Issues Workflow)

The `brd-to-issues` skill uses these functions to create Epics and Stories from a BRD.

### createEpic(epicName, description, labels)

Create a new Epic issue.

**Signature:**
```javascript
createEpic(epicName, description, labels = [])
```

**Parameters:**
- `epicName` (string) — Epic summary/title (e.g., "Instant Card Provisioning")
- `description` (string) — Epic description (markdown supported, include BRD summary + link)
- `labels` (string[], optional) — Labels to apply (should include `brd-epic` and `brd:[name]`)

**Returns:** `{key: string, id: string}` — Epic issue key and ID

**Example:**
```javascript
const epic = await createEpic(
  "Instant Card Provisioning",
  `## BRD: Instant Card Provisioning\n\nSummary: Enable customers to instantly receive and activate their card...\n\n[Link to BRD](./docs/brd/brd-instant-provisioning.md)`,
  ["brd-epic", "brd:instant-provisioning"]
);
console.log(`Created Epic: ${epic.key}`);
```

### createStory(storyTitle, description, epicKey, labels, blockedBy)

Create a new Story issue linked to an Epic.

**Signature:**
```javascript
createStory(storyTitle, description, epicKey, labels = [], blockedBy = [])
```

**Parameters:**
- `storyTitle` (string) — Story summary/title (e.g., "Slice 1: Challenge Question Validation")
- `description` (string) — Story description (markdown; include "What to build" + acceptance criteria + architecture notes)
- `epicKey` (string) — Parent Epic key (e.g., "CARD-100")
- `labels` (string[], optional) — Labels to apply (should include `backlog`, `brd:[name]`, `slice-N`)
- `blockedBy` (string[], optional) — Array of issue keys that block this story (e.g., `["CARD-101"]`)

**Returns:** `{key: string, id: string}` — Story issue key and ID

**Example:**
```javascript
const story = await createStory(
  "Slice 1: Challenge Question Validation via Callback",
  `## What to build\n\nAcquisitions Platform creates and encrypts challenge questions...\n\n### Acceptance Criteria\n\n- [ ] Acquisitions generates challenge Q&A...`,
  "CARD-100",
  ["backlog", "brd:instant-provisioning", "slice-1"],
  [] // No blockers for first slice
);
console.log(`Created Story: ${story.key}`);

// For dependent slices:
const story2 = await createStory(
  "Slice 2: SMS Content Generation & Delivery",
  `## What to build\n\n...`,
  "CARD-100",
  ["backlog", "brd:instant-provisioning", "slice-2"],
  ["CARD-101"] // Blocked by Slice 1
);
console.log(`Created Story: ${story2.key}`);
```

### updateBlockedBy(issueKey, blockingIssueKeys)

Update the "Blocked by" field for an issue (links to blocking issues).

**Signature:**
```javascript
updateBlockedBy(issueKey, blockingIssueKeys = [])
```

**Parameters:**
- `issueKey` (string) — Issue key to update
- `blockingIssueKeys` (string[], optional) — Array of issue keys that block this issue

**Returns:** void

**Example:**
```javascript
// Mark CARD-103 as blocked by CARD-102
await updateBlockedBy("CARD-103", ["CARD-102"]);

// Clear blockers
await updateBlockedBy("CARD-104", []);
```

### linkIssuesToEpic(epicKey, storyKeys)

Batch link multiple Stories to an Epic (useful for adding slices to existing Epic after updates).

**Signature:**
```javascript
linkIssuesToEpic(epicKey, storyKeys = [])
```

**Parameters:**
- `epicKey` (string) — Epic key (e.g., "CARD-100")
- `storyKeys` (string[], optional) — Array of Story keys to link

**Returns:** void

**Example:**
```javascript
// Link existing stories to epic
await linkIssuesToEpic("CARD-100", ["CARD-101", "CARD-102", "CARD-103"]);
```

---

## Sub-Task Creation (story-grooming Workflow)

The `story-grooming` skill uses these functions to create Sub-tasks from stories.

### createSubtask(parentKey, summary, description, labels, blockedBy)

Create a new Sub-task issue linked to a parent Story.

**Signature:**
```javascript
createSubtask(parentKey, summary, description, labels = [], blockedBy = [])
```

**Parameters:**
- `parentKey` (string) — Parent Story key (e.g., "CARD-101-A")
- `summary` (string) — Sub-task summary/title (e.g., "Task 1: Database Schema")
- `description` (string) — Sub-task description (markdown; include AC, implementation steps)
- `labels` (string[], optional) — Labels to apply (should include `ready` + `agent` or `human`, `task-N-[layer]`)
- `blockedBy` (string[], optional) — Array of Sub-task keys that block this task

**Returns:** `{key: string, id: string}` — Sub-task issue key and ID

**Example:**
```javascript
const subtask1 = await createSubtask(
  "CARD-101-A",
  "Task 1: Database Schema - Challenge Q&A Table",
  `## What to build\n\nCreate a PostgreSQL table for storing encrypted challenge Q&A...\n\n### Acceptance Criteria\n\n- [ ] Table created with fields...`,
  ["ready", "agent", "story:CARD-101-A", "task-1-schema", "sprint-ready"],
  [] // No blockers for first task
);
console.log(`Created Sub-task: ${subtask1.key}`);

// Second sub-task (blocked by first)
const subtask2 = await createSubtask(
  "CARD-101-A",
  "Task 2: Challenge Q&A API Endpoint",
  `## What to build\n\nCreate API endpoint for generating and validating challenge Q&A...\n\n### Acceptance Criteria\n\n- [ ] POST /challenge-qa endpoint created...`,
  ["ready", "agent", "story:CARD-101-A", "task-2-api", "sprint-ready"],
  [subtask1.key] // Blocked by Task 1
);
console.log(`Created Sub-task: ${subtask2.key}`);
```

### updateSubtaskLabels(subtaskKey, addLabels, removeLabels)

Update labels on a Sub-task (e.g., change work type from `agent` to `human`).

**Signature:**
```javascript
updateSubtaskLabels(subtaskKey, addLabels = [], removeLabels = [])
```

**Parameters:**
- `subtaskKey` (string) — Sub-task key (e.g., "CARD-101-A-1")
- `addLabels` (string[], optional) — Labels to add
- `removeLabels` (string[], optional) — Labels to remove

**Returns:** void

**Example:**
```javascript
// Change work type from agent to human (state label unchanged)
await updateSubtaskLabels("CARD-101-A-1", ["human"], ["agent"]);

// Mark as needs-info (state change, work type persists)
await updateSubtaskLabels("CARD-101-A-2", ["needs-info"], ["ready"]);
```

### queryReadyForGroomingStories(projectKey)

Query Jira for all stories with `backlog` label (stories ready for grooming).

**Signature:**
```javascript
queryReadyForGroomingStories(projectKey)
```

**Parameters:**
- `projectKey` (string) — Jira project key (e.g., "CARD")

**Returns:** `{issues: object[], total: number}` — Array of stories ready to groom

**Example:**
```javascript
const { issues, total } = await queryReadyForGroomingStories("CARD");
console.log(`Found ${total} stories in backlog ready to groom:`);
issues.forEach(story => {
  console.log(`- ${story.key}: ${story.fields.summary}`);
});
```

---

## BRD-to-Issues Workflow: Creating a Full Breakdown

Here's the complete flow for converting a BRD into Jira Epics and Stories:

1. **Create the Epic** → `createEpic()` with BRD title, summary, and `brd-epic` label
2. **Create Stories in order** → `createStory()` for each slice, referencing the Epic key
3. **Update dependencies** → `updateBlockedBy()` to link dependent slices (uses real Jira keys)
4. **Add labels** → Each Story gets `backlog`, `brd:[name]`, `slice-N`
5. **Output metadata** → Save `.brd-to-issues-output.json` with Epic/Story keys and slice mapping

**Example workflow:**
```javascript
// 1. Create Epic
const epic = await createEpic(
  "Instant Card Provisioning",
  "BRD summary and link...",
  ["brd-epic", "brd:instant-provisioning"]
);
console.log(`Created Epic: ${epic.key}`); // CARD-100

// 2. Create Slice 1 (no blockers)
const slice1 = await createStory(
  "Slice 1: Challenge Question Validation",
  "Slice 1 description...",
  epic.key, // CARD-100
  ["backlog", "brd:instant-provisioning", "slice-1"],
  []
);
console.log(`Created Story: ${slice1.key}`); // CARD-101

// 3. Create Slice 2 (blocked by Slice 1)
const slice2 = await createStory(
  "Slice 2: SMS Content Generation & Delivery",
  "Slice 2 description...",
  epic.key,
  ["backlog", "brd:instant-provisioning", "slice-2"],
  [slice1.key] // CARD-101
);
console.log(`Created Story: ${slice2.key}`); // CARD-102

// 4. Create Slice 3 (blocked by Slice 2)
const slice3 = await createStory(
  "Slice 3: Transaction Web Platform Review",
  "Slice 3 description...",
  epic.key,
  ["backlog", "brd:instant-provisioning", "slice-3"],
  [slice2.key] // CARD-102
);
console.log(`Created Story: ${slice3.key}`); // CARD-103

// 5. Output metadata
const metadata = {
  brd_file: "./docs/brd/brd-instant-provisioning.md",
  jira_project_key: "CARD",
  epic_key: epic.key,
  slices: [
    { number: 1, title: "Challenge Question Validation", jira_key: slice1.key, blocked_by: [] },
    { number: 2, title: "SMS Content Generation & Delivery", jira_key: slice2.key, blocked_by: [slice1.key] },
    { number: 3, title: "Transaction Web Platform Review", jira_key: slice3.key, blocked_by: [slice2.key] }
  ]
};
// Save metadata to .brd-to-issues-output.json
```

---

## Workflow: Triaging an Issue

Here's the typical flow when the skill triages a single issue:

1. **Query:** User provides issue key or search criteria
2. **Fetch:** Call `getIssue(issueKey)` to get full context (body, comments, current labels)
3. **Analyze:** Review comments for prior triage notes (start with `> *This was generated by AI during triage.*`)
4. **Recommend:** Suggest category + state labels
5. **Wait:** Maintainer confirms or overrides
6. **Update:** 
   - Call `addComment()` with triage notes or agent brief
   - Call `updateLabels()` to apply new state + category labels
   - Call `transitionIssue()` if closing (for `wontfix`)

---

## Handling Label Conflicts

State labels are mutually exclusive. If an issue has multiple state labels (e.g., both `groomed` and `ready`), flag it and ask:

```
⚠️ Conflict detected: This issue has both "groomed" and "ready" labels.
State labels should be mutually exclusive. Which state is correct? I'll remove the other.
```

Before proceeding, confirm the maintainer's choice and remove the conflicting label.

### Valid State Transitions
- `backlog` → `groomed` | `needs-info`
- `groomed` → `ready` | `needs-info`
- `ready` → `in-progress` | `needs-info`
- `needs-info` → `ready` | `in-progress`
- `in-progress` → `needs-info` | `ready-for-test`
- `ready-for-test` → `shipped` or closed

**Note:** Work type labels (`agent` or `human`) are set when transitioning to `ready` and persist through all subsequent states.

---

## Error Handling

Your Jira client throws custom errors:

- `JiraAuthError` — Invalid credentials (check token/email config)
- `JiraNotFoundError` — Issue key doesn't exist
- `JiraConfigError` — Jira not configured
- `JiraError` — General API error (rate limit, server error, etc.)

When the skill encounters these, it should log the error, inform the user, and suggest remediation (e.g., "Check your Jira token in config").

### Epic/Story Creation Errors

- **Epic creation fails**: Check that `brd-epic` and `brd:[name]` labels are valid for your Jira instance
- **Story creation fails**: Verify Epic key exists; check that `backlog`, `brd:[name]`, `slice-N` labels are valid
- **Blocked by update fails**: Verify blocking issue keys exist before linking
- **Batch linking fails**: Check all Story keys belong to the same project as the Epic

---

## Example: Full BRD-to-Issues Conversion

```javascript
const fs = require("fs");

async function convertBrdToIssues(brdFile, epicName, projectKey) {
  try {
    // 1. Parse BRD (extract requirements + slices)
    const brdContent = fs.readFileSync(brdFile, "utf-8");
    const slices = parseBrdToSlices(brdContent); // Custom parser
    
    // 2. Create Epic
    const epic = await createEpic(
      epicName,
      `## BRD: ${epicName}\n\n${brdContent.split("\n").slice(0, 20).join("\n")}\n\nSee full BRD: ${brdFile}`,
      ["brd-epic", `brd:${epicName.toLowerCase().replace(/ /g, "-")}`]
    );
    
    console.log(`✅ Created Epic: ${epic.key}`);
    
    // 3. Create Stories in dependency order
    const sliceKeys = {};
    for (const slice of slices) {
      const blockingKeys = slice.blockedBy
        .map(depNum => sliceKeys[`slice-${depNum}`])
        .filter(Boolean);
      
      const story = await createStory(
        `Slice ${slice.number}: ${slice.title}`,
        slice.description,
        epic.key,
        ["backlog", `brd:${epicName.toLowerCase().replace(/ /g, "-")}`, `slice-${slice.number}`],
        blockingKeys
      );
      
      sliceKeys[`slice-${slice.number}`] = story.key;
      console.log(`✅ Created Story: ${story.key}`);
    }
    
    // 4. Save metadata
    const metadata = {
      brd_file: brdFile,
      epic_key: epic.key,
      slices: Object.entries(sliceKeys).map(([name, key]) => ({ name, key }))
    };
    fs.writeFileSync(".brd-to-issues-output.json", JSON.stringify(metadata, null, 2));
    
    console.log(`✅ Conversion complete. Metadata saved to .brd-to-issues-output.json`);
    return metadata;
  } catch (error) {
    console.error(`❌ Conversion failed: ${error.message}`);
    throw error;
  }
}
```

---

## Notes

- All Jira API calls are authenticated via the config in `config/index.js` (email + token)
- The client supports both standard REST API and Agile API (board-based queries)
- Comment bodies use Atlassian Document Format (ADF); the client wraps plain text automatically
- Label names are case-sensitive and whitespace matters
- Epic and Story creation return issue keys (e.g., `CARD-100`), which can be used immediately for linking and blocking
- Batch operations (e.g., creating many stories at once) may hit rate limits; consider adding delay logic if needed
