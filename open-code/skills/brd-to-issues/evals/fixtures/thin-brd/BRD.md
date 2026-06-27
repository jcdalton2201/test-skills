# Business Requirements Document: TeamPulse

## Executive Summary
Build an internal tool so engineering managers can see how their teams are doing each week without scheduling 1:1s for status updates.

## Stakeholders
| Role | Name / Group | Concern |
|------|--------------|---------|
| Engineering Managers | EM org | Want visibility without overhead |
| ICs | Engineering | Don't want surveillance |

## Goals & Success Metrics
- Reduce status-update meetings by 30% within one quarter.
- 80% of ICs submit at least 3 of 4 weekly pulses per month.

## Scope
### In Scope
- Weekly pulse survey (mood, blockers, wins).
- Manager dashboard summarizing team-level trends.
### Out of Scope
- Performance review integration.
- Cross-team rollups.

## Functional Requirements
### FR-1: Pulse submission
**Description:** IC fills out a short weekly pulse.
**Priority:** Must

### FR-2: Manager dashboard
**Description:** Manager sees team trends.
**Priority:** Must

### FR-3: Reminders
**Description:** ICs are reminded if they haven't submitted by Friday.
**Priority:** Should

## Open Questions
- What channel for reminders (email, Slack)?
- How is "team" defined — org chart, ad hoc?
