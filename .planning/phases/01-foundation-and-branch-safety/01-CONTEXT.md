# Phase 1: Foundation and Branch Safety - Context

**Gathered:** 2026-03-28
**Status:** Ready for planning

<domain>
## Phase Boundary

Establish the structural foundation for the app.py refactor: create app factory pattern, extract shared extensions to prevent circular imports, set up multi-branch strategy for 4+ team members. All 113 existing routes must keep working after this phase — no functional changes, only structural preparation.

</domain>

<decisions>
## Implementation Decisions

### Extensions Strategy
- **D-01:** Claude's Discretion — choose best init pattern for extensions (lazy init vs factory-only)
- **D-02:** Claude's Discretion — choose how services access db_manager (shared instance, app.extensions, or parameter passing)
- **D-03:** Claude's Discretion — choose where User class (UserMixin) should live (core/models.py, core/auth.py, or core/extensions.py)
- **D-04:** Claude's Discretion — choose where OAuth configuration moves (inside create_app or core/oauth.py)
- **D-05:** Claude's Discretion — choose how agent_middleware and automation_engine are initialized (extensions vs on-demand)
- **D-06:** Claude's Discretion — choose where SUBSCRIPTION_PLANS dict lives (core/config.py or services/subscription_service.py)

### App Factory Design
- **D-07:** Claude's Discretion — choose where DL service thread spawning happens (inside create_app or __main__ block)
- **D-08:** Claude's Discretion — choose how background workers access config in factory pattern (app context or param passing)

### Branch Workflow
- **D-09:** 6 long-lived branches: `main`, `dev`, `demo`, `backend`, `frontend`, `mixed`
- **D-10:** Merge flow: frontend + backend -> mixed (integration) -> dev (canary) -> main (production)
- **D-11:** Demo = fork from dev at stable point, freeze 48 hours before jury presentation
- **D-12:** No branch protection rules — team trusts each other, anyone pushes anywhere
- **D-13:** All branches are long-lived (persist throughout the project)
- **D-14:** Multiple team members use Claude Code — .planning/ files need careful merge strategy:
  - Shared files (ARCHITECTURE.md, codebase maps): merge for combined knowledge
  - Session-specific files (STATE.md, research stages): add to .gitignore to avoid conflicts

### Backwards Compatibility
- **D-15:** Incremental migration — move one Blueprint at a time, app.py shrinks gradually
- **D-16:** Manual smoke test after each Blueprint move (start app, click through main features)

### Claude's Discretion
Claude has full discretion on all technical implementation details (D-01 through D-08). The user trusts Claude's engineering judgment for:
- Extension initialization patterns
- Database access patterns in services
- File organization for User class, OAuth, config data
- Thread spawning and background worker patterns

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Refactoring Proposal
- `DOCUMENTS/backend/APP-PY-renew.md` — Detailed refactoring plan with code examples for services/routes split

### Current Architecture
- `.planning/codebase/ARCHITECTURE.md` — Current system architecture and layer analysis
- `.planning/codebase/STRUCTURE.md` — Directory layout and key file locations
- `.planning/codebase/CONVENTIONS.md` — Code style and naming patterns to follow

### Project Context
- `.planning/PROJECT.md` — Project goals, constraints, key decisions
- `.planning/REQUIREMENTS.md` — Phase 1 requirements: FOUND-01 through FOUND-04

### Research
- `.planning/research/ARCHITECTURE.md` — Recommended Blueprint + service layer architecture
- `.planning/research/PITFALLS.md` — Circular import risks, url_for breakage, partial refactor traps

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `core/database.py` — Database class already exists, used as db_manager in app.py
- `core/auth.py` — AuthManager class already handles auth logic
- `core/config.py` — Config class with environment variable loading
- `core/services/dl_client.py` — DLClient pattern shows how services already access DL service
- `routes/` directory — exists but empty, ready for Blueprint files
- `services/` directory — exists but empty, ready for service modules

### Established Patterns
- Module-level globals in app.py (lines 96-128): limiter, csrf, login_manager, db_manager, Talisman
- AuthManager, AutomationEngine, AgentMiddleware all take db_manager as constructor arg
- `auth_bp` Blueprint already defined inline at app.py line 262 (partial refactor — needs completing)
- Service functions pattern in `core/services/dl_client.py` — classes with methods, not standalone functions

### Integration Points
- app.py line 134: `app.extensions['database'] = db_manager` — existing extension registration pattern
- app.py line 107: `login_manager.login_view = 'auth.signin'` — will need Blueprint-qualified name
- Lines 289, 291: `url_for()` calls use unqualified endpoint names — will break when routes move
- Jinja2 templates in `ui/templates/` — may contain `url_for()` calls needing Blueprint qualification

</code_context>

<specifics>
## Specific Ideas

- User wants branches created for team members immediately so they can start working independently
- The branch strategy is designed for team separation: user owns backend, friend owns frontend
- `.planning/` merge conflicts are a known concern — need gitignore rules for session-specific files while keeping architecture docs shared
- APP-PY-renew.md is the team's existing proposal and should be followed closely

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-foundation-and-branch-safety*
*Context gathered: 2026-03-28*
