# Rust TDD Skill - Complete Guide

This skill automates Rust code generation via Test-Driven Development (TDD). Use it to write production-quality Rust code from requirements with minimal human intervention.

## Quick Start

1. **Have a requirement?** → Use `brd-grill` skill to write `BRD.md`
2. **Have a BRD?** → Use `brd-to-issues` skill to write `EPIC.md` + `STORIES.md`
3. **Have stories?** → Use `plan-to-tracer-issues` to publish `[ready][agent]` issues
4. **Have issues?** → **Use this skill** to implement each `[ready][agent]` issue

Or jump straight to step 4 if you already have a clear issue with acceptance criteria.

## Files in This Skill

### Core Documentation

- **[SKILL.md](SKILL.md)** - Main workflow for TDD in Rust. Read this first.
  - Philosophy: what makes good tests
  - Workflow phases: understand → plan → tracer bullet → loop → refactor
  - Rust-specific considerations
  - When to ask the user (blocking decisions)
  - Integration with tracker system (GitHub/Linear)

- **[TRACKER-WORKFLOW.md](TRACKER-WORKFLOW.md)** - Tracker management during implementation ⭐ **READ THIS**
  - Issue status lifecycle with label progression
  - When and how to update tracker (comments, status, labels)
  - Label management: state labels are mutually exclusive, work type labels persist
  - Label flow: `backlog` → `groomed` → `ready` / `needs-info` → `in-progress` → `ready-for-test`
  - Work type labels: `agent` or `human` — set at triage, persist through all states
  - Comment templates for starting, blocking, resuming, completing
  - Complete example workflow

- **[rust-examples.md](rust-examples.md)** - Five complete Rust TDD examples (caveman mode)
  - Example 1: Simple validation (tracer bullet)
  - Example 2: Result types and error handling
  - Example 3: Dependency injection with mocking
  - Example 4: Mutable state and borrow checker
  - Example 5: Error handling strategies
  - Quick tips and patterns

### Test & Design Guidance

- **[test.md](test.md)** - What makes good tests vs bad tests
  - Integration-style tests (observable behavior)
  - Anti-patterns (implementation-detail tests)
  - Rust examples with `#[test]` attributes

- **[mocking.md](mocking.md)** - When and how to mock
  - Mock at system boundaries only
  - Dependency injection patterns
  - Rust trait-based mocking

- **[interface-design.md](interface-design.md)** - Design for testability
  - Accept dependencies, don't create them
  - Return results, don't mutate state
  - Small surface area

- **[refactoring.md](refactoring.md)** - How to refactor safely
  - Only refactor when all tests pass
  - Incremental cleanup
  - Maintain test integrity

- **[deep-modules.md](deep-modules.md)** - Small interface, deep implementation
  - Hide complexity behind simple APIs
  - Make modules easy to understand and test

### Integration & Workflow

- **[DEVELOPMENT-FLOW.md](DEVELOPMENT-FLOW.md)** - Label flow through all 5 phases
    - Complete pipeline: brd-grill → brd-to-issues → plan-to-tracer-issues → rust-tdd → closure
    - What happens in each phase
    - How state labels flow: `backlog` → `groomed` → `ready` / `needs-info` → `in-progress` → `ready-for-test`
    - Work type labels (`agent` or `human`) persist from triage through completion
    - Example: payment processor full flow
    - When to use each skill

  - **[../LABEL-PROGRESSION.md](../LABEL-PROGRESSION.md)** - Label progression reference
    - State labels: `backlog`, `groomed`, `ready`, `needs-info`, `in-progress`, `ready-for-test` (mutually exclusive)
    - Work type labels: `agent`, `human` (persistent from triage)
    - Transition rules by skill
    - Common workflows and scenarios

## How This Skill Works

### Input
You provide:
- An issue with `[ready][agent]` labels and acceptance criteria (Gherkin scenarios)
- Optional parent issue ID (e.g., `#118`)
- Clarifying answers if agent asks

### Process
Agent follows TDD red-green-refactor:
1. **RED**: Write test (fails because code doesn't exist)
2. **GREEN**: Write minimal code to pass
3. **REPEAT**: One behavior at a time
4. **REFACTOR**: Clean up when all tests pass

### Decision Handling
When agent hits a decision point:
- Creates `[needs-info]` child issue (if parent exists)
- Explains the decision and its impact
- Waits for your answer
- Continues with decision made

### Output
- ✅ Rust code with comprehensive tests
- ✅ All tests passing (`cargo test`)
- ✅ Clean, refactored implementation
- ✅ Decision issues resolved or documented
- ✅ Ready to review and merge

## Integration with Your Skills Ecosystem

### Before TDD (Upstream)

**brd-grill** → write requirements → `BRD.md`
```
Agent grills you on:
- Problem and goal
- Stakeholders
- Scope (in/out)
- Functional requirements
- Non-functional requirements
- Acceptance criteria
- Constraints, assumptions, risks
- Priorities (MoSCoW)
```

**brd-to-issues** → break requirements → `EPIC.md`, `STORIES.md`
```
Agent slices into vertical-slice stories:
- Each story is end-to-end (UI, API, DB, tests)
- Sized for ~3 days of work
- Gherkin acceptance criteria
- Dependencies mapped
```

**plan-to-tracer-issues** → publish to tracker → Real issues
```
Agent publishes stories as issues:
- Labels: [groomed] (initial state)
- Then triaged to: [ready] + [agent] (agent can implement)
-              or: [ready] + [human] (needs human judgment)
-              or: [needs-info] (blocked by missing info)
- Work type label ([agent] or [human]) persists through all states
- Marked in dependency order
```

### During TDD (This Skill)

**rust-tdd** → implement issue → Rust code
```
Agent implements one [ready][agent] issue:
- Remove [ready], add [in-progress], keep [agent]
- RED: write test for behavior
- GREEN: minimal code to pass
- Loop for each behavior
- REFACTOR: clean when all tests pass
- If blocked: [needs-info] + create blocking issue (keep [agent])
- When done: remove [in-progress], add [ready-for-test] (keep [agent])
```

## Typical Workflow

```
Day 1:  Use brd-grill
        → Output: BRD.md with requirements

Day 2:  Use brd-to-issues
        → Output: EPIC.md + STORIES.md with vertical slices

Day 3:  Use plan-to-tracer-issues or story-readiness
        → Output: GitHub/Linear issues [ready][agent] / [ready][human] / [needs-info]

Days 4-N: Use rust-tdd (this skill)
        → Input: One [ready][agent] issue
        → Output: Implemented, tested Rust code
        → Repeat for each issue
```

## Rules of Engagement

✅ **Agent will**:
- Write tests first (RED)
- Implement minimal code (GREEN)
- Show work at each step
- Ask for decisions (create HITL issues)
- Create comprehensive tests
- Refactor when all tests pass

❌ **Agent won't**:
- Write code before tests
- Anticipate future requirements
- Refactor while tests are failing
- Implement error cases that weren't in acceptance criteria
- Guess at design decisions

## Key Rust Patterns Used

### Dependency Injection (Testability)
```rust
// Testable: accepts dependency
fn process_order(order: &Order, gateway: &dyn PaymentGateway) -> Result<()>

// Hard to test: creates dependency internally
fn process_order(order: &Order) -> Result<()> {
    let gateway = StripeGateway::new()?;
    // ...
}
```

### Error Handling (Result Types)
```rust
// Good: returns Result for fallible operations
fn charge(amount: u64) -> Result<Payment, PaymentError>

// Tests can check: assert!(result.is_err())
// Real code can handle: charge().map_err(handle_error)?
```

### Mutable State (Borrow Checker Guides Design)
```rust
// Tests write: let mut cart = Cart::new()
// This signals: implementation needs fn add_item(&mut self, item: Item)
// Borrow checker prevents sharing mistakes automatically
```

### Test Organization
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn behavior_being_tested() {
        // Arrange
        let input = ...;
        // Act
        let result = function_under_test(input);
        // Assert
        assert_eq!(result, expected);
    }
}
```

## Decision Questions You'll See

Agent may ask:

**Design decisions**:
- "Should this return `Result<T, E>` or panic on error?"
- "Should this take `&self`, `&mut self`, or owned `self`?"
- "Should we create a trait here or keep it concrete?"

**Scope decisions**:
- "Is this behavior critical, or can it wait for next cycle?"
- "Should we implement error case X or skip for now?"

**Validation**:
- "Does this behavior match what you expected?"
- "Should this function be pure or have side effects?"

For each question, agent will:
1. Create HITL issue (if parent issue exists)
2. Explain the options and tradeoffs
3. Recommend an answer
4. Wait for your decision
5. Continue with decision made

## When NOT to Use This Skill

This skill is for **implementation from spec**. Use other skills for:

- **Requirements gathering** → Use `brd-grill`
- **Breaking down requirements** → Use `brd-to-issues`
- **Publishing to tracker** → Use `plan-to-tracer-issues`
- **Code review** → Use human review, this skill provides tests
- **Exploratory work** → This needs clear acceptance criteria first

## Success Criteria

You know this skill is working when:

✅ Issues go from `[ready][agent]` → `[in-progress][agent]` → `[ready-for-test][agent]` → merged  
✅ Work type label `[agent]` persists through all state transitions  
✅ All behaviors match acceptance criteria  
✅ Code is tested (tests in `#[cfg(test)]` modules)  
✅ Tests compile and pass (`cargo test`)  
✅ Design decisions are documented (blocking issues with `[needs-info]`)  
✅ Blocking questions create tracker issues (not lost in chat)  
✅ One issue per TDD cycle (vertical slices)  
✅ Code survives refactors without test changes

## Getting Help

1. **Read [SKILL.md](SKILL.md)** - Full workflow walkthrough
2. **Read [rust-examples.md](rust-examples.md)** - See concrete examples
3. **Check [DEVELOPMENT-FLOW.md](DEVELOPMENT-FLOW.md)** - Understand label flow
4. **Reference [LABEL-CHEATSHEET.md](LABEL-CHEATSHEET.md)** - Quick answers on labels

## Example: Using This Skill

**You provide**:
```
Issue #42: Implement email validation
Acceptance criteria (Gherkin):
  Scenario: Valid email passes
    When I validate "user@example.com"
    Then it returns valid
  
  Scenario: Invalid format fails
    When I validate "not-an-email"
    Then it returns invalid

Parent: #10 (User onboarding epic)
Labels: [ready][agent] [must] [s2] [epic-USR-1]
```

**Agent executes**:
```
1. Ask clarifications (if any)
2. RED: write test for valid email
3. GREEN: implement minimal validation
4. RED: write test for invalid email
5. GREEN: enhance validation
6. REFACTOR: clean up
7. Show code, all tests pass
8. Ready for review
```

**You get**:
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_email_passes() {
        assert!(is_valid_email("user@example.com"));
    }

    #[test]
    fn invalid_email_fails() {
        assert!(!is_valid_email("not-an-email"));
    }
}

fn is_valid_email(email: &str) -> bool {
    // implementation
}
```

Done. Ready to merge. 🚀

---

## File Organization

```
rust-tdd/
├── README.md                    ← You are here
├── SKILL.md                     ← Main workflow (read next)
├── rust-examples.md             ← 5 complete examples (read after SKILL.md)
├── DEVELOPMENT-FLOW.md          ← How labels flow through phases
├── LABEL-CHEATSHEET.md          ← Quick reference on labels
├── test.md                      ← What makes good/bad tests
├── mocking.md                   ← When and how to mock
├── interface-design.md          ← Design for testability
├── refactoring.md               ← How to refactor safely
└── deep-modules.md              ← Hide complexity behind APIs
```

Read in this order:
1. **README.md** (this file) - Overview
2. **SKILL.md** - Complete workflow
3. **rust-examples.md** - See it in action
4. **DEVELOPMENT-FLOW.md** - Understand label pipeline
5. Others - Reference as needed
