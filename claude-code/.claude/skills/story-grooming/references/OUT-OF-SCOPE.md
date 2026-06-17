# story-grooming: Out of Scope

This document defines what the `story-grooming` skill does NOT do, what is team responsibility, and key assumptions.

---

## What the Skill Does NOT Do

### ❌ Code Writing
The skill does NOT write code, nor does it generate code templates or boilerplate.

**Why**: Code writing is the developer's responsibility. The skill provides guidance and structure, not implementation.

**Team responsibility**: Developers pick up `ready` + `agent` and `ready` + `human` tasks and implement them.

---

### ❌ Developer Assignment
The skill does NOT assign developers to tasks.

**Why**: Teams have different capacity, expertise, and preferences. Assignment is a team/manager decision.

**Team responsibility**: Team lead or sprint planner assigns tasks based on:
- Developer availability
- Developer skills (e.g., "Alice knows Node.js backend")
- Current workload
- Sprint capacity

**Example**: Skill creates CARD-101-A-1 with `ready` + `agent` labels. Team decides: "Alice will take this task."

---

### ❌ Sprint Planning
The skill does NOT decide which stories go into which sprint.

**Why**: Sprint planning depends on team capacity, priorities, and dependencies—which vary per team.

**Team responsibility**: Product/sprint planning team decides:
- Which stories are in this sprint vs. next sprint
- How many tasks each developer can handle
- Sprint capacity and velocity

**Example**: Skill grooms CARD-101 into 4 tasks (8 days). Team decides: "We can fit 2 tasks this sprint, 2 next sprint."

---

### ❌ Velocity Estimation
The skill does NOT estimate team velocity or capacity.

**Why**: Velocity is historical data unique to each team. The skill can't know how fast your team works.

**Team responsibility**: Track your team's velocity over sprints and use that for capacity planning.

**Example**: Skill identifies task as "~2 days". Your team's experience says "Our team does 2-day tasks in 3 days." Team adjusts accordingly.

---

### ❌ Time Tracking
The skill does NOT track actual time spent on tasks.

**Why**: Time tracking is handled by Jira board workflow and team metrics.

**Team responsibility**: Developers update task status (To Do → In Progress → In Review → Done) and optionally log time.

---

### ❌ Enforcing Strict Dependency Order
The skill detects dependencies but does NOT prevent developers from violating them.

**Why**: Jira can't prevent merges or task changes; it can only suggest order.

**Team responsibility**: Team lead ensures:
- Blockers are respected during implementation
- If a developer wants to reorder, they discuss with the team
- Merge conflicts are handled properly

**Example**: Task 2 is blocked by Task 1. Developer on Task 2 starts anyway and gets merge conflicts. Team resolves during code review.

---

### ❌ Backlog Refinement (Beyond Grooming)
The skill only grooms stories into tasks. It does NOT:
- Refine user stories (that's BRD/requirements workflow)
- Estimate story points (that's team estimation)
- Prioritize stories (that's product management)
- Handle story dependencies at story level (that's epic/roadmap planning)

**Why**: These activities require business/product context outside the skill's scope.

**Team responsibility**: 
- Product team refines requirements (see `grill-me` and `brd-to-issues` skills)
- Team estimates and prioritizes stories
- Product lead manages epic/story dependencies

---

### ❌ Complex Architectures
The skill assumes a typical **layered architecture** (database → API → UI → tests).

The skill does NOT handle:
- Microservices with cross-service dependencies
- Event-driven architectures with pub/sub patterns
- CQRS (Command Query Responsibility Segregation)
- Distributed tracing or observability architecture

**Why**: These require domain-specific knowledge that's project-specific.

**Team responsibility**: If your architecture is non-standard, manually adjust task breakdown or override skill designations.

**Example**: Your story touches 5 microservices with event-driven coordination. Skill creates standard tasks, but team realizes more integration testing is needed. Team adds tasks or changes work type to `human` for architect review.

---

### ❌ Security & Compliance Decisions
The skill does NOT decide security strategy or compliance approach.

**Why**: Security/compliance decisions require expertise and organizational policy that the skill can't have.

**Team responsibility**: 
- Security team reviews tasks that touch auth, encryption, or sensitive data
- Compliance team reviews tasks for regulatory requirements
- Architects make decisions on encryption approach, key management, etc.

**Example**: Task "Encrypt challenge answers" is marked `ready` + `human` because encryption strategy requires security review. Security team provides guidance, then developer implements.

---

### ❌ Performance Optimization Decisions
The skill does NOT decide caching strategy, database optimization, or performance targets.

**Why**: Performance decisions depend on observed metrics, load patterns, and business requirements.

**Team responsibility**: 
- DevOps/SRE team defines performance SLOs
- Team reviews tasks that impact performance
- Performance engineers validate approach

**Example**: Task "Optimize challenge validation query" is marked `ready` + `human` because optimization strategy depends on current metrics. Performance engineer works with developer.

---

### ❌ Handling Architectural Changes Mid-Sprint
The skill creates tasks based on the current story understanding.

The skill does NOT re-groom tasks if:
- Architecture changes mid-sprint
- A critical bug is discovered
- Requirements shift

**Why**: Mid-sprint changes are team decisions, not skill decisions.

**Team responsibility**: If major changes happen:
1. Team discusses impact
2. Decide: proceed with current tasks, pivot, or pause
3. Update tasks in Jira as needed
4. Re-run skill if significant re-grooming needed

**Example**: During implementation of Task 1, team discovers the schema needs major redesign. Team re-grooms Task 1 and creates new tasks.

---

## Team Responsibilities

### Product/Ownership Team
- ✅ Define requirements (BRD)
- ✅ Prioritize stories and features
- ✅ Decide which stories go in which sprint
- ✅ Provide business context and constraints
- ❌ NOT write code or make technical decisions

### Engineering/Tech Team
- ✅ Review task breakdown (groom stories)
- ✅ Make architectural/technical decisions
- ✅ Estimate effort and velocity
- ✅ Implement tasks
- ✅ Review code and ensure quality
- ❌ NOT decide business priorities or sprint assignment

### Development Team (Individual Contributors)
- ✅ Pick up assigned tasks
- ✅ Implement according to acceptance criteria
- ✅ Write tests and documentation
- ✅ Ask for help if blocked
- ✅ Update task status in Jira
- ❌ NOT decide task order or story priority

### Team Lead/Scrum Master
- ✅ Facilitate grooming sessions
- ✅ Assign tasks to developers
- ✅ Enforce dependency order
- ✅ Unblock developers if stuck
- ✅ Track sprint progress
- ❌ NOT implement tasks or make architectural decisions

---

## Key Assumptions

### Assumption 1: Story Has Clear Acceptance Criteria
**Assumed**: Story (from `brd-to-issues` workflow) has well-defined, testable acceptance criteria.

**If not true**: Skill will ask user to add AC or mark story as `needs-info`.

**Your responsibility**: Ensure stories have AC before grooming.

---

### Assumption 2: Codebases Are Known & Documented
**Assumed**: User can name which codebases/systems a story touches, and those systems are documented.

**If not true**: Skill will ask user for clarification. If unclear, task is marked `needs-info`.

**Your responsibility**: Maintain up-to-date architecture documentation so users can quickly identify codebase scope.

---

### Assumption 3: 2-Day Task Size Is Reasonable
**Assumed**: Your team's typical task is ~2 days of work (range: 1-3 days).

**If not true**: Adjust task breakdown accordingly. 2 days is a heuristic, not a hard rule.

**Your responsibility**: Calibrate task size to your team's actual productivity. If tasks consistently take 4+ days, split them further.

---

### Assumption 4: Jira Is Properly Configured
**Assumed**: Your Jira instance has:
- Sub-task issue type
- Epic Link and Blocked By fields
- Required labels (`backlog`, `groomed`, `ready`, `agent`, `human`, `in-progress`, `ready-for-test`, etc.)
- Proper workflow states (To Do, In Progress, etc.)

**If not true**: Skill will provide a pre-flight check. Set up Jira before running skill.

**Your responsibility**: Verify Jira configuration matches requirements (see JIRA-INTEGRATION.md).

---

### Assumption 5: Agent Designation Is Meaningful
**Assumed**: `agent` work type designation is actionable. Either:
- You have AI agents that can implement tasks, OR
- You use it as a signal for "junior developer" or "straightforward work"

**If not true**: Ignore agent designation or use for team discussion.

**Your responsibility**: Decide how your team interprets agent/human designation.

---

### Assumption 6: Dependencies Are Layer-Based
**Assumed**: Story touches codebases with typical layered architecture (database → API → UI → tests).

**If not true**: Skill will create standard tasks, but you may need to adjust dependencies.

**Your responsibility**: If architecture is non-standard, manually adjust task order or override skill's dependency detection.

---

### Assumption 7: No Circular Dependencies Exist
**Assumed**: Story requirements don't have circular dependencies (Slice A depends on Slice B depends on Slice A).

**If not true**: Skill will flag and ask user to break the cycle.

**Your responsibility**: Resolve circular dependencies at story level (before grooming).

---

## Handoff Points

### Story Grooming → Task Implementation

**Skill responsibility**: Create groomed Jira sub-tasks with:
- ✅ Clear acceptance criteria
- ✅ Agent/Human/Needs-Info designation
- ✅ Dependency links
- ✅ Agent briefs (for `agent` work type tasks)

**Team responsibility**: 
- ✅ Review sub-tasks in Jira
- ✅ Assign to developers
- ✅ Developers implement following acceptance criteria
- ✅ Update task status as work progresses

---

### Task Implementation → Code Review

**Developer responsibility**: 
- ✅ Implement task per acceptance criteria
- ✅ Write tests (unit + integration)
- ✅ Ensure tests pass
- ✅ Open PR for code review

**Team responsibility**:
- ✅ Review PR
- ✅ Verify acceptance criteria are met
- ✅ Check code quality and tests
- ✅ Approve and merge

---

### Task Completion → Story Completion

**Team responsibility**:
- ✅ Mark task as "Done" when merged
- ✅ Verify all tasks for a story are done
- ✅ Mark parent story as "Done"
- ✅ Verify story acceptance criteria are met (parent story, not individual tasks)

---

## Common Scenarios: Who Decides?

### Q: Sprint is full. Should we groom more stories for backlog?
**Answer**: Product/sprint planning team decides backlog grooming. Skill just grooms stories you ask it to groom.

---

### Q: A task takes 4 days instead of 2. What do we do?
**Answer**: Your team decides. Options:
- Split task further next time (learn from estimate)
- Accept that this team's "2 days" is actually "4 days" (recalibrate)
- Add developer to task (parallel work)

---

### Q: A developer thinks a task should be `human` instead of `agent`?
**Answer**: Developer can change the work type label in Jira. Skill's designation is a suggestion, not a hard rule.

---

### Q: Story needs to be split differently than skill proposed?
**Answer**: Team can manually adjust. Skill's sub-story breakdown is a proposal. Modify as needed.

---

### Q: What if two tasks have conflicting requirements?
**Answer**: Team resolves during development/code review. Skill can't catch all conflicts; that's why code review exists.

---

### Q: Can we do tasks out of order (skip blockers)?
**Answer**: Technically yes, but:
- You may hit merge conflicts or missing dependencies
- Team lead should understand why order is being violated
- Discuss before violating blockers

---

## Summary: Skill Is a Tool, Not a Manager

The `story-grooming` skill:
- ✅ Analyzes stories and proposes task breakdown
- ✅ Suggests agent/human designation
- ✅ Creates Jira sub-tasks with proper structure
- ✅ Detects dependencies and task order
- ❌ Does NOT make team decisions
- ❌ Does NOT enforce strict rules
- ❌ Does NOT replace team judgment

**Key mindset**: Use the skill to accelerate grooming, but team has final say on all decisions. Trust your team to override the skill when needed.
