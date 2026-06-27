---
name: tdd
description: Test-driven development with red-green-refactor loop. Use when user wants to build features or fix bugs using TDD, mentions "red-green-refactor", wants integration tests, or asks for test-first development.
---

# Test-Driven Development for Rust

## Philosophy

**Core principle**: Tests verify behavior through public interfaces, not implementation. Code can change entirely; tests shouldn't.

**Good tests** are integration-style: they exercise real code paths through public APIs. They describe _what_ the system does, not _how_. A good test reads like a specification - "user can checkout with valid cart" tells you exactly what capability exists. These tests survive refactors because they don't care about internal structure.

**Bad tests** are coupled to implementation. They mock internal collaborators, test private methods, or verify through external means. Warning sign: test breaks when you refactor, but behavior hasn't changed.

See [test.md](test.md) for examples, [mocking.md](mocking.md) for mocking guidelines, and [rust-examples.md](rust-examples.md) for complete Rust code examples.

## Anti-Pattern: Horizontal Slices

**DO NOT write all tests first, then all implementation.** This is "horizontal slicing" - treating RED as "write all tests" and GREEN as "write all code."

This produces bad tests:

- Tests written in bulk test _imagined_ behavior, not _actual_ behavior
- You end up testing the _shape_ of things (data structures, function signatures) rather than user-facing behavior
- Tests become insensitive to real changes - they pass when behavior breaks, fail when behavior is fine
- You outrun your headlights, committing to test structure before understanding the implementation

**Correct approach**: Vertical slices via tracer bullets. One test → one implementation → repeat. Each test responds to what you learned from the previous cycle. Because you just wrote the code, you know exactly what behavior matters and how to verify it.

```
WRONG (horizontal):
  RED:   test1, test2, test3, test4, test5
  GREEN: impl1, impl2, impl3, impl4, impl5

RIGHT (vertical):
  RED→GREEN: test1→impl1
  RED→GREEN: test2→impl2
  RED→GREEN: test3→impl3
  ...
```

## Workflow for Autonomous Code Generation

### 0. Understand the Requirement

**Agent responsibility**:
- Ask clarifying questions: "What's the input? What's the output? What can go wrong?"
- Break requirement into 3-5 key behaviors
- Identify error cases
- Sketch data types needed
- For Rust: "Should this return Result or panic? Should it mutate state?"
- **Check for parent tracker issue** — if user provides one, capture the ID (e.g., `#118`, `MER-42`)

**User provides**:
- Requirement description
- Answers to clarifying questions
- Validation of agent's understanding
- *Optional*: Parent tracker issue ID or URL

### 1. Planning

Before writing any test:

- [ ] Confirm interface design: "public functions, what do they take, what do they return?"
- [ ] List behaviors to test in priority order (not implementation steps)
- [ ] For Rust: Traits needed for mocking, error handling strategy (panic vs Result), ownership model
- [ ] Get user approval on plan

**Focus**: Critical paths, complex logic, error cases. Not every edge case.

**If parent issue exists**: Draft implementation slices using [plan-to-tracer-issues](../plan-to-tracer-issues/SKILL.md) workflow
- One slice per behavior
- Mark as AFK (agent can implement)
- Publish to tracker if user approves
- Reference parent issue under `## Parent`
- Use real issue IDs in subsequent `Blocked by` relationships

### 2. Tracer Bullet (First Test → First Implementation)

**Agent writes ONE test confirming ONE behavior**:

**BEFORE STARTING**:
- If issue exists in tracker: Update status to "In Progress"
- If issue exists in tracker: Add comment with link to branch/PR

**RED phase**:
```
- Write test in #[cfg(test)] module
- Use only public interface
- Test should fail (code doesn't exist yet)
- Agent shows test to user
```

**GREEN phase**:
```
- Write minimal code to pass test
- Stub types, implement just enough
- Don't implement error cases yet
- Compile: cargo test (must pass)
- Agent shows implementation to user
```

See [rust-examples.md](rust-examples.md) Examples 1-2 for simple tracer bullets.

### 3. Incremental Loop (Remaining Behaviors)

**Agent repeats for each remaining behavior**:

```
RED:   Write next test → agent shows to user
GREEN: Write minimal code → cargo test passes → agent shows to user
```

Rules:

- One behavior at a time
- Only code needed for current test
- Don't anticipate future tests
- Tests focus on observable behavior, not implementation
- For Rust: Borrow checker guides design. If test needs `&mut`, implementation needs it.
- Every cycle: tests compile and pass

**Agent shows each cycle to user and explains the design choice.**

See [rust-examples.md](rust-examples.md) Example 3-4 for dependency injection and mutable state patterns.

**Keep tracker updated**:
- If issue exists: Add comments showing progress
- No status changes during loop (stays "In Progress")

### 4. Refactor (When All Tests Pass)

After all tests pass, agent looks for cleanup:

- Extract duplication (helper functions)
- Deepen modules (complexity behind simple interfaces)
- Apply SOLID principles if natural
- For Rust: Consider generics, better error types, trait bounds
- Run `cargo test` after each refactor
- Only refactor code structure, never test structure

**Never refactor while tests RED.** Get all tests GREEN first.

**Keep tracker updated**:
- Still "In Progress" (not done yet)
- Add comment: "Refactoring complete, all tests passing"

### 5. Error Cases (Optional, If Critical)

If requirement mentions error cases:

- Add tests for error paths
- Implement error handling (Result types, custom errors)
- Example: [rust-examples.md](rust-examples.md) Example 5

### 6. Completion (Ready for Testing)

**When all acceptance criteria are met and tests pass**:

1. **Update tracker issue**:
   - Change status from "In Progress" to "Done"
   - Remove label: `[in-progress]`
   - Add label: `[ready-for-test]`
   - Keep label: `[agent]` (work type persists)
   - Add comment with:
     - Link to PR or branch
     - Summary of what was implemented
     - Any decisions that were made
     - Test coverage (e.g., "8 tests covering happy path and 3 error cases")

2. **Example tracker comment**:
   ```
   ## Implementation Complete
   
   Branch: feature/payment-charge
   PR: <link if available>
   
   ### What was implemented
   - Charge endpoint accepts valid/invalid cards
   - Returns Result<Payment, PaymentError>
   - All acceptance criteria met
   
   ### Tests
   - ✓ Valid card charges successfully
   - ✓ Invalid card is declined
   - ✓ Network timeout handled
   - ✓ Edge case: zero amount rejected
   (8 tests total, all passing)
   
   ### Design decisions made
   - Used custom PaymentError enum (per decision #104a)
   - Charge function is pure (no side effects)
   - Uses dependency injection for payment gateway
   
   Ready for code review.
   ```

3. **Clean up labels**:
   - Remove: `[in-progress]`
   - Keep: `[must]`, `[s2]`, `[epic-PREFIX]`, `[agent]`
   - Add: `[ready-for-test]`
   - Final labels: `[must][s2][epic-PREFIX][agent][ready-for-test]`

**When done**: Issue is in "Done" status with `[ready-for-test]` label, ready for security scan.

### 7. Automatic Security Check

**After adding `[ready-for-test]`, run secure-code-review-rust**:

1. **Run security scan**:
   - AST analysis on implemented files
   - Cargo clippy with security lints
   - Cargo audit for dependencies
   - Pattern matching for obvious issues
   - Generate SECURITY-FINDINGS.md

2. **If CRITICAL/HIGH findings**:
   - Create tracker issue: `[security][critical]` or `[security][high]`
      - Update parent issue: Remove `[ready-for-test]`, add `[needs-security-fix]`
   - Status stays: In Progress (not Done)
   - Add comment:
     ```
     ⚠️ **Security Scan Failed**
     
     Found: 2 CRITICAL, 1 HIGH issues
     
     | Severity | Issue | File | Auto-Fix |
     |----------|-------|------|----------|
     | CRITICAL | SQL Injection | payment.rs:42 | ✅ Available |
     | CRITICAL | Hardcoded Secret | config.rs:10 | ✅ Available |
     | HIGH | Path Traversal | upload.rs:88 | ✅ Available |
     
     See SECURITY-FINDINGS.md for details and auto-fix diffs.
     Created issues: SEC-101, SEC-102, SEC-103
     ```

3. **If MEDIUM/LOW only**:
   - Keep `[ready-for-test]` label
   - Add comment: "ℹ️ Security scan passed. N minor findings noted."
   - Continue to code review

4. **Label progression**:
   ```
      Before scan:  [must][s2][epic-PREFIX][ready-for-test]
   
      If issues:    [must][s2][epic-PREFIX][needs-security-fix]
                    + New issue: [security][needs-info]
   
      After fix:    [must][s2][epic-PREFIX][ready-for-test]
                    (re-scan passes)
   
      After merge:  [shipped]
      ```

5. **Fix security issues**:
   - Apply auto-fix from SECURITY-FINDINGS.md
      - Run `cargo test` to verify no regressions
      - Re-run security scan
      - When passed, add `[ready-for-test]` again

See [secure-code-review-rust](../secure-code-review-rust/SKILL.md) for full security scan documentation.

### 8. Integration with Tracker System

When agent encounters a **blocking decision question** (see "When to Ask User" below), the question should be tracked:

**If parent issue exists**:
1. Create HITL child issue on tracker (GitHub/Linear)
2. Title: The decision question (e.g., "Decide: Result vs panic for payment errors")
3. Body:
   ```markdown
   ## Parent
   <reference to parent, e.g., #118>
   
   ## Decision Needed
   <The question the agent is blocked on>
   
   ## Options
   - Option A: <description>
   - Option B: <description>
   
   ## Impact
   <How this decision affects implementation>
   ```
4. Label: `[needs-info]` (this is a blocking question that needs human input)
5. Set blocker: Parent issue is blocked by this decision issue
6. Update parent issue status to "Blocked" (optional, if tracker supports)
Add comment on parent issue: "Blocked waiting for decision on #XYZ"
8. Update parent issue: Remove `[in-progress]`, add `[needs-info]`, keep `[agent]`
9. Wait for user to answer (comment on issue or reply in chat)
10. Update decision issue as resolved when answer is given
11. Update parent issue: Remove `[needs-info]`, add `[in-progress]`, keep `[agent]`
12. Continue implementation

**If no parent issue**: Ask user in chat, track decision in code comments or commit messages.

**Decision questions to track**:
- "Should function return `Result<T, E>` or panic?"
- "Should this take `&self`, `&mut self`, or owned `self`?"
- "Should error be custom type or String error?"
- "Is this behavior critical or can it wait?"
- "Should we create a trait for this or keep concrete?"

This ensures:
- Design decisions are documented and audit-able
- User can review decisions asynchronously
- Other agents can see rationale if revisiting code
- Tracker reflects the actual decision workflow
- Implementation progress is visible in issue status/comments

## Rust-Specific Considerations

### Borrow Checker as Test Guide

The borrow checker is not your enemy; it's test feedback:

- If test needs `&mut Cart`, implementation needs `fn add_item(&mut self, item: Item)`
- If test passes `&immutable_data`, implementation should take `&self`
- Tests that can't compile = design hasn't settled yet (expected for tracer bullet)

### Error Handling

- **Fallible operations**: Always return `Result<T, E>`
- **Tests for Result**: Use `.is_ok()`, `.is_err()`, `.expect()`, pattern matching
- **In tests**: `.expect("msg")` for asserting success, `.unwrap()` acceptable for failures in tracer bullets
- **Custom error types**: Defer to refactor phase unless critical upfront

### Mocking for Testability

1. Define trait for external dependency (database, API, payment gateway, etc.)
2. Create mock struct that implements trait
3. Pass mock via `&dyn Trait` parameter
4. Real implementation: pass concrete struct via same trait object
5. No dependency creation inside functions - always inject

Example: [rust-examples.md](rust-examples.md) Example 3

### Test Structure

All tests live in `#[cfg(test)]` module at bottom of same file:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_name() {
        // test code
    }
}
```

### Common Patterns

```rust
// Equality
assert_eq!(actual, expected);

// Boolean
assert!(condition);
assert!(!condition);

// Error handling
assert!(result.is_err());
assert_eq!(result.unwrap_err(), "expected error");

// Success
assert!(result.is_ok());
assert_eq!(result.unwrap(), expected_value);
```

## Checklist Per Test Cycle

```
[ ] Test describes one behavior
[ ] Test uses public interface only
[ ] Test would survive internal refactor
[ ] Code is minimal for this test
[ ] No speculative features added
[ ] cargo test passes
[ ] For Rust: borrow checker happy, Result types where needed
```

## Tracker Status & Label Management

### Label Progression (State Labels)

State labels are **mutually exclusive** — remove the previous state when transitioning.

| Label | Meaning | Applied When |
|-------|---------|---------------|
| `backlog` | Ungroomed story | Initial creation |
| `groomed` | Story is groomed, ready for triage | After brd-to-issues creates it |
| `ready` | Ready to be picked up | Triaged and ready for work |
| `needs-info` | Blocked by missing information | Missing info, vague AC, open questions |
| `in-progress` | Work has started | Agent or human picks up the story |
| `ready-for-test` | Implementation complete | All AC implemented, tests passing |

### Work Type Labels (Persistent)

Work type labels persist through all state transitions:

| Label | Meaning | Set When |
|-------|---------|----------|
| `agent` | Autonomous agent work (AFK) | At triage — clear AC, no human decisions needed |
| `human` | Needs human judgment (HITL) | At triage — design decisions, security, complex integration |

**Key distinction:**
- State labels (`backlog`, `groomed`, `ready`, `needs-info`, `in-progress`, `ready-for-test`) are mutually exclusive
- Work type labels (`agent`, `human`) persist from triage through completion

### Issue Lifecycle with Label Transitions

```
Published (by brd-to-issues):
  Labels: [groomed][must|should|could][s1|s2|s3][epic-*]
  Comment: "Issue created from BRD"

Triaged (by plan-to-tracer-issues or story-readiness):
  Labels: REMOVE [groomed], ADD [ready] + [agent] or [ready] + [human] or [needs-info]
  Comment: "Triaged: ready for agent work" (or reason for human/needs-info)

Agent Starts Implementation:
  Labels: REMOVE [ready], ADD [in-progress], KEEP [agent]
  Comment: "Starting implementation on branch feature/..."

Agent Blocked by Decision:
  Labels: REMOVE [in-progress], ADD [needs-info], KEEP [agent]
  Comment: "Blocked waiting for decision on #XYZ: <question>"
  (Create HITL issue for the blocking question)

Agent Resumes After Decision:
  Labels: REMOVE [needs-info], ADD [in-progress], KEEP [agent]
  Comment: "Resuming with decision: <answer>"

Agent Completes:
  Labels: REMOVE [in-progress], ADD [ready-for-test], KEEP [agent]
  Comment: Summary of work, test count, decisions made

Code Review (Human):
  Labels: (no change, stays [ready-for-test][agent])
  (human adds approval/requested-changes)

Merged:
  Labels: REMOVE [ready-for-test], remove size labels, optionally ADD [shipped]
  Comment: "Merged as <commit hash>"
```

### Label Changes by Phase

```
Created:    [groomed][must][s2][epic-PREFIX]
Triaged:    [ready][agent][must][s2][epic-PREFIX]        (removed [groomed], added [agent])
Started:    [in-progress][agent][must][s2][epic-PREFIX]  (removed [ready])
Blocked:    [needs-info][agent][must][s2][epic-PREFIX]   (removed [in-progress])
Resumed:    [in-progress][agent][must][s2][epic-PREFIX]  (removed [needs-info])
Complete:   [ready-for-test][agent][must][s2][epic-PREFIX] (removed [in-progress])
Merged:     [shipped] or empty                           (removed [ready-for-test], size, agent)
```

**Key rules:**
- Only one state label at a time. Always remove the previous state label.
- Work type label (`agent` or `human`) persists through all state transitions.

### Required Tracker Comments

**When starting** (REMOVE [ready], ADD [in-progress], KEEP [agent]):
```
## ▶️ Starting Implementation

**Issue:** <title>
**Branch:** feature/...
**Labels:** -[ready] +[in-progress] (keeping [agent])
**Estimated completion:** <estimate based on size>
```

**When blocked** (REMOVE [in-progress], ADD [needs-info], KEEP [agent]):
```
## ⏸️ Blocked - Needs Info

**Waiting for:** Decision on #XYZ
**Question:** <specific question that must be answered>
**Labels:** -[in-progress] +[needs-info] (keeping [agent])
**Created:** HITL issue #XYZ for this decision
```

**When unblocked** (REMOVE [needs-info], ADD [in-progress], KEEP [agent]):
```
## ▶️ Resuming Implementation

**Decision received:** #XYZ resolved with: <answer>
**Labels:** -[needs-info] +[in-progress] (keeping [agent])
```

**When complete** (REMOVE [in-progress], ADD [ready-for-test], KEEP [agent]):
```
## ✅ Implementation Complete

**Branch:** feature/...
**PR:** <link if available>
**Labels:** -[in-progress] +[ready-for-test] (keeping [agent])

### What was implemented
- <behavior 1>
- <behavior 2>
- (list all acceptance criteria that were met)

### Test coverage
- ✓ <test 1>
- ✓ <test 2>
- ✓ <test 3 - edge case>
(<total> tests, all passing)

### Design decisions applied
- Decision #XYZ: <what was decided>
- <other design choices>

### Notes
- <anything else relevant for review>

Ready for code review.
```

## When to Ask User (Blocking Questions)

Agent stops and creates a tracker issue (or asks in chat) when:

**Design decisions**:
- "Should this function return `Result<T, E>` or panic on error?"
- "Should this take `&self`, `&mut self`, or owned `self`?"
- "Should we create a custom error type now or defer?"
- "Should we use a trait for dependency injection here?"
- "Is this a public API or internal helper?"

**Priority/scope**:
- "Is this behavior critical, or can it wait for later cycles?"
- "Should we implement error case X or skip for now?"
- "Should we add validation or trust the caller?"

**Validation**:
- "Does this behavior match what you expected?"
- "Should this function have side effects or be pure?"
- "Is the public interface correct?"

**How to ask**:
1. If parent issue exists → create HITL child issue with `[needs-info]` label
2. Add comment to parent issue: "Blocked waiting for decision on #XYZ"
3. Update parent issue: Remove `[in-progress]`, add `[needs-info]`
4. If no parent issue → ask in chat
5. Either way: agent waits for answer before continuing
6. Agent explains why the decision matters for implementation
7. When user answers: Remove `[needs-info]`, add `[in-progress]`, continue

---

## References

- [test.md](test.md) - What makes good/bad tests
- [mocking.md](mocking.md) - When and how to mock
- [interface-design.md](interface-design.md) - Designing for testability
- [rust-examples.md](rust-examples.md) - Complete Rust TDD examples (read this first!)
- [refactoring.md](refactoring.md) - How to refactor safely

## Integration with Other Skills

- **[plan-to-tracer-issues](../plan-to-tracer-issues/SKILL.md)** — Use this skill to slice a requirement into AFK/HITL issues before starting TDD. Each issue becomes one TDD cycle.
- **[brd-grill](../brd-grill/SKILL.md)** — Use this skill if requirement needs specification/BRD first.

**Workflow**:
1. User provides requirement + parent issue ID
2. TDD skill asks clarifying questions
3. Draft slices (one per behavior)
4. Publish slices as issues to tracker
5. TDD implements each slice as it completes
6. Blocking decisions become HITL child issues
7. User answers decisions, agent continues
