# Project Research Summary

**Project:** Group-project-AI-ML (USTH GEN14 — Flask Monolith Refactoring)
**Domain:** Academic defense — Flask AI/ML web app refactoring for jury retake
**Researched:** 2026-03-28
**Confidence:** HIGH

## Executive Summary

This project is a university retake defense for a Python/Flask AI/ML web application. The primary objective is NOT to ship new features — it is to refactor a 3514-line monolithic `app.py` into a defensible Blueprint + service layer architecture, and to fix the specific weaknesses the jury documented in `RETAKENOTES.md`. The jury failed the first attempt because of a wrong architecture diagram, missing explanation of the execution engine, wrong evaluation methodology, and informal documentation. All work must be viewed through that lens: the jury evaluates engineering judgment and the ability to explain what was built, not feature completeness.

The recommended approach is the Application Factory + Blueprints + Service Layer pattern described in `DOCUMENTS/backend/APP-PY-renew.md`. This is a well-established Flask pattern with no new major dependencies — the stack remains Flask + SQLite + Python throughout. The refactor is organized into 5 phases with clear dependencies: foundation (extensions + factory shell), service layer extraction (parallelizable across 4 team members), Blueprint route extraction, app factory wiring, and finally tests + documentation. The `demo` branch must be frozen from a verified-working `dev` state no later than 24 hours before presentation.

The key risk is attempting too much. Anti-features — new UI, Docker changes, database migration, Celery, CI/CD — must be hard-refused. The second major risk is a partial refactor: `app.py` must reach under 100 lines before the refactor can be claimed "done." The jury will ask to see specific service files; if those files are stub-only or missing, credibility collapses. A third risk is circular imports, which crash the app on startup and are demo-killing. These three risks dominate all others and must be addressed in the earliest phases.

---

## Key Findings

### Recommended Stack

The stack is constrained — no framework changes are permitted. The refactoring introduces only dev-tooling additions. The core structural change is adopting the Application Factory pattern (`create_app()`) and Flask Blueprints, both of which are built into Flask 3.x and require zero new production dependencies.

For testing — currently entirely absent — the recommended additions are `pytest`, `pytest-flask`, `pytest-mock`, and `pytest-cov`. These are dev-only dependencies and should live in a separate `requirements-dev.txt`. The existing test files already follow `test_*.py` naming conventions so pytest discovers them with no modifications. Service layer functions should be module-level functions (not classes), accepting plain Python types and returning plain Python tuples/dicts — this matches the existing `core/services/` pattern and is the only design that makes unit testing practical.

**Core technologies:**
- Flask Blueprints (built-in): Route modularization — standard Flask pattern, zero new dependency
- Application Factory `create_app()` (built-in): Slim entry point — enables test client instantiation without side-effects
- `core/extensions.py` (new file, no library): Shared singletons (`limiter`, `csrf`, `login_manager`) — prevents circular imports when Blueprints are extracted
- pytest >=8.0: Test runner — already compatible with existing `test_*.py` naming, no rewrites needed
- pytest-flask >=1.3: Flask fixture system — manages app context lifecycle, prevents `RuntimeError: Working outside of application context`
- pytest-mock >=3.14: Mocking — critical for isolating DB and OAuth calls in service unit tests

**What NOT to add:** Flask-SQLAlchemy, Pydantic, Celery, Flask-RESTX, or any infrastructure change. These are out of scope and consume time the team does not have.

### Expected Features

The jury is not evaluating whether features work — they are evaluating whether the architecture is defensible. Feature decisions must be made with that lens.

**Must have (table stakes):**
- Flask Blueprint routing (5+ Blueprint files) — jury expects "proper" Flask; a 3514-line app.py is a red flag
- App factory pattern with `app.py` under 100 lines — demonstrates knowledge of Flask conventions
- Service layer with zero Flask imports (`services/*.py`) — separation of concerns is the central claim
- All existing features still functional — breaking anything during demo is catastrophic
- Correct system architecture diagram (Brain-Body-DL split) — jury directly flagged the wrong figure
- Description of execution engine (Kahn's algorithm in `workflow_engine.py`) — jury explicitly said it was missing
- Evaluation section redone with real data — jury called the synthetic-image methodology wrong
- Unit tests for service layer (5-10 tests minimum) — having zero tests undermines the testability claim
- 3-branch git strategy (main/dev/demo) set up before any code changes — protects demo from mid-flight breakage

**Should have (differentiators that reduce jury risk):**
- Pytest fixture demonstrating service testability with a passing test run
- Inline docstrings on all service functions (one sentence each)
- Error handling with explicit HTTP status codes (400/403/404 with JSON bodies)
- Config externalized to `core/config.py` with env var support
- Logging instead of print statements
- Architecture diagram that references specific file paths

**Defer permanently:**
- Any new user-facing feature
- WebSocket / real-time updates (existing polling is acceptable and defensible)
- Database migration (SQLite to PostgreSQL)
- CI/CD pipeline or Docker changes
- Mobile responsiveness
- New ML models or retraining
- Test coverage beyond the service layer

### Architecture Approach

The target architecture is a strict three-layer model: `routes/` (Blueprints — HTTP only, 10-15 lines per function, no SQL), `services/` (business logic — no Flask imports, plain Python in/out), and `core/` (infrastructure — database, config, auth, workflow engine, already largely extracted). A new `core/extensions.py` file is needed to hold shared singletons and break the circular import risk. The app factory wires all Blueprints together and stays under 100 lines. The existing `core/` layer is already largely correct — the main work is extracting 110 route handlers and their embedded business logic into the new `services/` and `routes/` directories.

**Major components:**
1. `app.py` (App Factory) — `create_app()` only; registers Blueprints, initializes extensions, starts DL thread; target: <100 lines
2. `routes/` (8 Blueprint files) — `auth`, `billing`, `workflows`, `ai`, `dl_proxy`, `admin`, `inventory`, `pages`; HTTP layer only
3. `services/` (7 service files) — `wallet_service`, `subscription_service`, `ai_worker_service`, `workflow_service`, `dl_proxy_service`, `inventory_service`, `user_service`; all SQL and business logic lives here
4. `core/` (unchanged infrastructure) — `database.py`, `config.py`, `auth.py`, `workflow_engine.py`, `agent_middleware.py`; plus new `extensions.py` and `models.py`
5. `tests/` (new pytest directory) — `conftest.py` + one test file per service; separate from existing `test/` (live integration scripts)

**Critical boundary rule:** Services must never import from `flask` or reference `request`, `jsonify`, or `current_user`. User identity is passed as `user_id: int`. This is the enforcement mechanism for separation of concerns.

### Critical Pitfalls

1. **Circular import death loop** — When Blueprint files import `limiter`/`db_manager` from `app.py` and `app.py` imports those Blueprint files, Python crashes with `ImportError` at startup. Prevention: create `core/extensions.py` first, move all singletons there, then extract Blueprints.

2. **Partial refactor — app.py still over 200 lines** — Creating Blueprint files but leaving 80% of logic in `app.py` means the refactoring claim is not credible when the jury asks to see `app.py`. Prevention: enforce the under-100-line completion criterion before calling the phase done.

3. **Breaking existing functionality via `url_for()` renames** — Moving a route to a Blueprint changes `url_for('signin')` to `url_for('auth.signin')`. Jinja2 templates and JS fetch calls break silently with 404s. Prevention: audit all templates for unqualified `url_for()` calls before each Blueprint merge.

4. **Services containing Flask imports** — Copy-pasting from route handlers brings `from flask import request` into service files, making unit tests impossible. Prevention: `grep -r "from flask import" services/` before committing each service file.

5. **Merging broken code to `demo` branch under deadline pressure** — The git history shows emergency commits without protected branches. Prevention: set up main/dev/demo branches on day 1 with branch protection on `demo`; freeze `demo` 24 hours before presentation with a full smoke test.

---

## Implications for Roadmap

Based on combined research, the following 5-phase structure is recommended. The ordering is determined by hard dependencies: Blueprints cannot work without the extensions layer; service tests cannot be written until services exist; documentation cannot honestly describe code that hasn't been written yet.

### Phase 1: Foundation and Branch Safety
**Rationale:** Circular imports (Pitfall 1) and merge conflicts (Pitfall 5) are demo-killing risks that cost zero features to fix but destroy everything if skipped. This phase has no dependencies and unblocks all other phases.
**Delivers:** Protected branch structure, `core/extensions.py` with all singletons migrated, `User` class moved to `core/models.py`, `SUBSCRIPTION_PLANS`/`AI_JOBS`/`JOBS_DIR` moved to their respective service files, `app.py` scaffolded as `create_app()` shell (routes still in `app.py` temporarily)
**Addresses:** Table stakes: 3-branch git strategy, app factory pattern
**Avoids:** Pitfall 1 (circular imports), Pitfall 5 (demo branch corruption), Pitfall 7 (User class accessibility)
**Research flag:** Standard Flask patterns — no additional research needed

### Phase 2: Service Layer Extraction (Parallelizable)
**Rationale:** Services have no dependency on Blueprint structure. Extracting them first means when Blueprints are wired up in Phase 3, the functions they call already exist. This phase can be split across all 4 team members working on completely different files simultaneously — zero merge conflicts.
**Delivers:** `services/wallet_service.py`, `services/subscription_service.py`, `services/ai_worker_service.py`, `services/workflow_service.py`, `services/dl_proxy_service.py`, `services/inventory_service.py`, `services/user_service.py` — all with zero Flask imports
**Team split:** Member A: wallet + subscription; Member B: ai_worker; Member C: inventory; Member D: workflow + user; Team lead: dl_proxy
**Addresses:** Table stakes: service layer with no Flask imports; differentiator: inline docstrings
**Avoids:** Pitfall 2 (partial refactor), Pitfall 4 (Flask imports in services), Pitfall 6 (global state stranded in app.py)
**Research flag:** Standard patterns — no additional research needed

### Phase 3: Blueprint Route Extraction
**Rationale:** Blueprint files depend on services (Phase 2) being populated. Extract one Blueprint at a time and smoke-test the app after each one before moving to the next. Recommended order: `auth` -> `dl_proxy` -> `ai` -> `billing` -> `workflows` -> `inventory` -> `admin`.
**Delivers:** `routes/auth.py`, `routes/billing.py`, `routes/workflows.py`, `routes/ai.py`, `routes/dl_proxy.py`, `routes/admin.py`, `routes/inventory.py`, `routes/pages.py`
**Addresses:** Table stakes: Flask Blueprint routing, all existing features still working
**Avoids:** Pitfall 3 (url_for breakage — audit templates before each merge), Pitfall 9 (context_processor and errorhandler lost), Pitfall 10 (hardcoded URL strings), Pitfall 11 (limiter init_app pattern)
**Research flag:** Standard Flask patterns — no additional research needed

### Phase 4: App Factory Wiring + Tests
**Rationale:** After all routes are extracted, `app.py` can be cleaned to under 100 lines by registering all Blueprints in `create_app()` and deleting the original route blocks. Tests can now be written against the service layer (no Flask context needed).
**Delivers:** Final `app.py` under 100 lines, `tests/conftest.py`, 5-10 passing pytest tests covering `wallet_service` and `subscription_service` at minimum, `pytest --cov=services` baseline coverage report
**Addresses:** Table stakes: app factory <100 lines, unit tests for service layer; differentiator: error handling with HTTP status codes
**Avoids:** Pitfall 2 completion gate (wc -l app.py < 100)
**Stack:** pytest, pytest-flask, pytest-mock, pytest-cov (from `requirements-dev.txt`)
**Research flag:** Standard patterns — no additional research needed

### Phase 5: Documentation, Evaluation, and Demo Freeze
**Rationale:** Documentation must describe the actual running system, not the plan. `anser.md` report sections, module .md files, and the architecture diagram must be written after Phase 4 is confirmed working on `dev`. The evaluation section (LSTM metrics, OCR accuracy) can run in parallel from day 1 as it is independent of code. Demo branch is frozen from verified `dev` no later than 24 hours before presentation.
**Delivers:** Fixed system architecture diagram (Brain-Body-DL split with real file references), execution engine section (Kahn's algorithm), redone LSTM/OCR evaluation with real data, module .md files per service, `demo` branch tagged and frozen, dry-run presentation completed
**Addresses:** All jury-specific table stakes: correct architecture diagram, execution engine description, evaluation section redone
**Avoids:** Pitfall 8 (report documents plan not reality), Pitfall J1 (code/report mismatch), Pitfall J2 (demo branch behind dev), Pitfall J3 (refactor breaks high-visibility features)
**Research flag:** No research needed — this is documentation and evaluation work

### Phase Ordering Rationale

- Foundation first because circular imports crash the app on startup — no later phase can proceed if the app doesn't start
- Service extraction before route extraction because Blueprints call services; creating an empty call target would require rewriting imports in Phase 3
- Tests after services exist because the entire value of the refactor is that services are testable in isolation — tests before services are impossible
- Documentation last because the jury cross-references report claims against running code; writing documentation before code is done creates the exact mismatch that caused the first failure
- Evaluation section (report) is the only workstream that can start on day 1 in parallel, since it is independent of code changes

### Research Flags

Phases with well-established patterns (skip research-phase):
- **Phase 1:** Application Factory and extensions.py are canonical Flask 3.x patterns with official documentation
- **Phase 2:** Module-level service functions are plain Python; no framework-specific research needed
- **Phase 3:** Blueprint registration is well-documented; the `url_for()` audit is procedural
- **Phase 4:** pytest-flask fixture patterns are well-documented in official pytest-flask docs
- **Phase 5:** Documentation and evaluation — no research needed, judgment call on content

No phases require `/gsd:research-phase` — all patterns are well-established and grounded in primary project documents.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Flask 3.x confirmed in requirements.txt; testing library recommendations based on stable patterns; version numbers are MEDIUM (not live-verified against PyPI) |
| Features | HIGH | Derived entirely from primary sources: RETAKENOTES.md (jury feedback), PROJECT.md (constraints), APP-PY-renew.md (proposal), and direct codebase inspection |
| Architecture | HIGH | All component boundaries derived from direct audit of app.py (3514 lines); all line number references verified in codebase |
| Pitfalls | HIGH | All pitfalls verified with specific line numbers from the actual codebase; not speculative |

**Overall confidence:** HIGH

### Gaps to Address

- **Exact pytest library versions:** pip was not executable during research; versions listed (pytest >=8.0, pytest-flask >=1.3, etc.) are based on training data and should be verified against live PyPI before adding to `requirements-dev.txt`. This is low-risk — the patterns are stable across minor versions.
- **Template `url_for()` audit scope:** The full extent of unqualified `url_for()` calls across all Jinja2 templates was not enumerated. This audit must be performed during Phase 3 as each Blueprint is extracted. A single missed reference creates a silent 404. Recommend running `grep -r "url_for('" ui/templates/` before each Blueprint merge.
- **SUBSCRIPTION_PLANS exact location:** The research identifies `SUBSCRIPTION_PLANS` at line 143 of `app.py` as a global that must be moved to `services/subscription_service.py`. Verify no other file imports it directly from `app.py` before moving — a missed import would cause an `ImportError`.
- **Evaluation data availability:** The report's evaluation section needs real OCR accuracy metrics and LSTM confusion matrices from non-synthetic test data. Whether this data is available or needs to be generated was not confirmed. This should be scoped on day 1 of Phase 5 work.

---

## Sources

### Primary (HIGH confidence)
- `RETAKENOTES.md` — Jury failure feedback, direct basis for all table stakes
- `DOCUMENTS/backend/APP-PY-renew.md` — Refactoring proposal with code examples and explicit constraints
- `app.py` (direct inspection, 3514 lines, 2026-03-28) — All architectural decisions verified against actual code
- `.planning/PROJECT.md` — Active requirements, out-of-scope constraints, team context
- `.planning/codebase/STACK.md`, `TESTING.md`, `ARCHITECTURE.md`, `CONCERNS.md` — Codebase analysis

### Secondary (MEDIUM confidence)
- Flask 3.x documentation (Application Factory, Blueprints, Testing) — Patterns are stable and well-established; live docs not fetched during research session
- pytest-flask PyPI / GitHub — `conftest.py` patterns and fixture documentation; based on training data, not live session

### Tertiary (LOW confidence)
- Library version numbers (pytest >=8.0, pytest-flask >=1.3, etc.) — Should be verified against live PyPI before installation

---
*Research completed: 2026-03-28*
*Ready for roadmap: yes*
