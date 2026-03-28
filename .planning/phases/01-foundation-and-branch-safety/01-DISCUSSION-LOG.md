# Phase 1: Foundation and Branch Safety - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-28
**Phase:** 01-foundation-and-branch-safety
**Areas discussed:** Extensions strategy, App factory design, Branch workflow, Backwards compat

---

## Extensions Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Lazy init pattern | Create empty instances in extensions.py, call init_app(app) in create_app() | |
| Factory-only pattern | Everything created inside create_app() and passed around | |
| You decide | Claude picks the best approach | ✓ |

**User's choice:** You decide (all 6 extension-related questions)
**Notes:** User trusts Claude's engineering judgment for all technical implementation details. This covers extension init pattern, DB access pattern, User class location, OAuth config location, agent_middleware/automation_engine init, and SUBSCRIPTION_PLANS location.

---

## App Factory Design

| Option | Description | Selected |
|--------|-------------|----------|
| Inside create_app() | Spawn DL service thread at end of factory function | |
| In __main__ block | Only spawn when running directly, not during tests | |
| You decide | Claude picks | ✓ |

**User's choice:** You decide (both questions)
**Notes:** Background worker config access also deferred to Claude's discretion.

---

## Branch Workflow

| Option | Description | Selected |
|--------|-------------|----------|
| Feature -> mixed -> dev -> main | backend/frontend merge into mixed, then dev, then main | ✓ (modified) |
| Feature -> dev -> main | All branches merge directly into dev | |

**User's choice:** Custom flow — frontend + backend -> mixed -> dev -> main. 6 long-lived branches total.
**Notes:** User specified 6 branches: main, dev, demo, backend, frontend, mixed. Backend is for user (app.py work), frontend for friend. Mixed is integration point. Demo forks from dev, freezes 48h before jury. No branch protection — team trusts each other. Multiple Claude Code users — need .planning/ gitignore strategy for session files while keeping architecture docs shared.

---

## Backwards Compatibility

| Option | Description | Selected |
|--------|-------------|----------|
| Incremental | Move one Blueprint at a time | ✓ |
| Big-bang | Move all routes at once | |

**User's choice:** Incremental migration with manual smoke tests
**Notes:** 113 routes must keep working. Verify by starting app and clicking through main features after each Blueprint move.

---

## Claude's Discretion

All technical implementation decisions (D-01 through D-08) deferred to Claude — user wants to focus on team workflow and migration strategy, not implementation details.

## Deferred Ideas

None
