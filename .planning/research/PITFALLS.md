# Domain Pitfalls: Flask Monolith Refactoring

**Domain:** Flask Blueprint refactoring + university jury retake
**Researched:** 2026-03-28
**Confidence:** HIGH (grounded in direct codebase inspection)

---

## Critical Pitfalls

Mistakes that cause broken demos, cascade failures, or jury failure.

---

### Pitfall 1: Circular Import Death Loop

**What goes wrong:** When Blueprints import from `app.py` (or each other) while `app.py` imports those same Blueprints, Python raises `ImportError: cannot import name 'X' from partially initialized module`. The app refuses to start entirely — demo-killing on presentation day.

**Why it happens:** The current `app.py` defines global objects (`db_manager`, `auth_manager`, `limiter`, `csrf`, `login_manager`) at module level, then the existing `auth_bp` Blueprint is also defined in the same file (lines 262-333). When routes are extracted into separate files and those files need `db_manager` or `limiter`, they import from `app.py`. But `app.py` imports those Blueprint files. Python sees a cycle and fails.

**Evidence in this codebase:** `auth_bp` is already partially extracted but still lives inside `app.py`. The `limiter` decorator (`@limiter.limit(...)`) is applied directly on Blueprint routes (lines 267, 303). Moving those routes to `routes/auth.py` requires importing `limiter` — and `limiter` lives in `app.py`.

**Consequences:** Import error at startup. App does not start. Total demo failure.

**Prevention:**
1. Move all shared singletons (`db_manager`, `auth_manager`, `limiter`, `csrf`, `login_manager`) into `core/extensions.py`. Initialize them with no arguments (`limiter = Limiter(get_remote_address)`) in that file.
2. In `app.py` (the factory), call `limiter.init_app(app)`, `csrf.init_app(app)`, etc.
3. Blueprint files import from `core/extensions.py`, never from `app.py`.
4. `app.py` imports Blueprint files only inside the factory function body, not at the top level.

**Detection:** Run `python app.py` immediately after each file is extracted. A circular import crashes on startup with a clear traceback. Test this before touching the next Blueprint.

**Phase:** Addresses in Blueprint extraction phase — must be solved before any route files are created.

---

### Pitfall 2: The "Partial Refactor" — App That Looks Refactored But Isn't

**What goes wrong:** The team creates `routes/auth.py`, `routes/billing.py`, etc. but still leaves 80% of the logic in `app.py` registered directly on the `app` object. The jury opens `app.py`, sees it's still 2000+ lines, and the "refactoring" claim falls apart.

**Why it happens:** Extracting the easy routes first creates an illusion of progress. The complex, intertwined routes (wallet, subscription, AI chat) are left in `app.py` because they share too many locals (`SUBSCRIPTION_PLANS`, `AI_JOBS`, `JOBS_DIR`, inline thread functions). Time pressure causes the team to stop before it's actually done.

**Evidence in this codebase:** `auth_bp` is already defined inline in `app.py` — it's a Blueprint in name only, not in a separate file. `SUBSCRIPTION_PLANS` (line 143) and `AI_JOBS` / `JOBS_DIR` (lines 138-139) are module-level globals that multiple future routes would need. `background_ai_task` and `save_job_file`/`load_job_file` are defined as bare functions in `app.py` and referenced by the AI chat routes at the bottom.

**Consequences:** Jury is unconvinced. The report claims separation of concerns but the code does not demonstrate it. `anser.md` requires documenting HOW app.py was split — if it isn't truly split, this deliverable cannot be completed honestly.

**Prevention:**
- Set a strict completion criterion: `app.py` must be under 100 lines before calling the refactor "done."
- Define the migration order upfront: `core/extensions.py` first, then services, then route files, then `app.py` cleanup.
- Do not commit a "refactored" version to the `demo` branch until `app.py` is genuinely slim.
- Assign one team member per Blueprint file to ensure complete extraction, not shared partial work.

**Detection:** Run `wc -l app.py` (or count lines in editor) after each sprint. If still above 200 lines when routes are supposedly extracted, something is still stranded in `app.py`.

**Phase:** Completion gate for the entire refactoring phase.

---

### Pitfall 3: Breaking Existing Functionality During Extraction

**What goes wrong:** A route is moved to a Blueprint file and registered with a `url_prefix`. The frontend JavaScript or Jinja2 `url_for()` calls that referenced the old route name now 404. This is silent — no Python error, just broken UI on demo day.

**Why it happens:** `url_for('signin')` becomes `url_for('auth.signin')` after registration as a Blueprint. Jinja2 templates and client-side `fetch('/auth/signin')` hardcoded paths stop matching. Flask-Login's `login_view = 'auth.signin'` must also be updated (it is currently set correctly at line 107, but `redirect(url_for('admin_workspace'))` on line 289 and `redirect(url_for('workspace'))` on line 291 are unqualified — these break if `admin_workspace` moves to a Blueprint).

**Evidence in this codebase:**
- Line 107: `login_manager.login_view = 'auth.signin'` — already qualified, correct.
- Line 289: `redirect(url_for('admin_workspace'))` — unqualified. Will break if route moves.
- Line 291: `redirect(url_for('workspace'))` — unqualified. Will break if route moves.
- Line 327: `redirect(url_for('auth.signin'))` — qualified, correct.
- Templates use `url_for()` extensively — each extracted Blueprint changes endpoint names.

**Consequences:** Pages that work before refactor 404 after refactor. Demo fails on features the jury specifically checks.

**Prevention:**
- Before extracting any Blueprint, search all `.html` templates and `.py` files for `url_for('routename')` and update to `url_for('blueprint.routename')`.
- Keep a mapping document: old endpoint name → new `blueprint.endpoint` name.
- Run the app and manually navigate all main pages (auth, workspace, dashboard, billing) after each Blueprint is extracted, before merging to `dev`.
- Never merge a Blueprint extraction PR to `dev` without a smoke test checklist sign-off.

**Detection:** After extracting each Blueprint, run `python app.py` and open every page in the browser. Any `BuildError: Could not build url for endpoint` is the warning sign.

**Phase:** Each individual Blueprint extraction PR.

---

### Pitfall 4: Services Layer That Still Contains Flask Imports

**What goes wrong:** Business logic is moved to `services/wallet_service.py` etc., but because it was copy-pasted from route handlers, it still contains `from flask import request`, `jsonify`, `current_user`, or `session`. The service layer is not actually decoupled from HTTP. Unit tests cannot be written for it without a Flask application context.

**Why it happens:** Copy-paste from inline route code preserves Flask imports automatically. Under time pressure, the team does not audit the paste. The service technically "runs" because the Flask app context is active during web requests, so the bug hides until someone writes a test.

**Evidence in this codebase:** The existing `core/` services (`core/auth.py`, `core/database.py`) are correctly decoupled — they take plain Python arguments, not Flask request objects. The refactoring proposal (APP-PY-renew.md) explicitly warns: "Tuyệt đối không chứa logic của HTTP (như import `request`, `jsonify`)". This constraint must be enforced during extraction.

**Consequences:** Services cannot be unit tested in isolation. The jury may ask "can you unit test this service?" — the answer becomes no. The entire point of the refactoring (testability, separation of concerns) is undermined.

**Prevention:**
- After pasting code into a service file, immediately grep the file for `from flask import` and `import flask`. Any hit is a bug.
- Services must receive all data as plain function arguments (dicts, strings, ints). They return plain Python values (tuples, dicts, booleans).
- The route layer is the only place `request`, `jsonify`, `current_user`, and `session` appear.

**Detection:** `grep -r "from flask import" services/` — any result is a violation.

**Phase:** Service extraction phase. Check before each service file is committed.

---

### Pitfall 5: Merge Conflicts Destroying the `demo` Branch

**What goes wrong:** Two team members both modify `app.py` (or the same Blueprint file) on different branches. The merge conflict is resolved incorrectly — a route is accidentally deleted, a registration line is duplicated, or a dependency injection is dropped. The app starts but silently misses an endpoint. Discovered on presentation day.

**Why it happens:** `app.py` is a 3514-line single file — every feature is in it. With 4+ members, any work that touches `app.py` simultaneously creates conflicts. The `demo` branch is supposed to be stable, but if someone merges a half-working `dev` into `demo` under deadline pressure, broken code reaches the jury.

**Evidence in this codebase:** Git history shows rapid emergency commits ("emergency fix the ocr system", multiple force-style fixes). The branching strategy (main/dev/demo) is still pending — it has not been set up yet, meaning the team is currently working without a protected demo branch.

**Consequences:** The demo branch contains broken code on presentation day. Features that worked before stop working. The jury sees errors.

**Prevention:**
- Set up the three-branch strategy before any refactoring begins. Protect `demo` with a branch rule (require PR, no direct push).
- Assign exclusive ownership: one team member = one Blueprint file. Never two people in the same file simultaneously.
- Merge to `dev` only via PRs, with a working-app verification step.
- Merge to `demo` only 24 hours before presentation, with a full smoke test.
- Never merge to `demo` while a refactoring PR is still in progress on `dev`.

**Detection:** Git conflicts in `app.py` or Blueprint files are the warning sign. If a merge requires manual conflict resolution in route registration code, treat it as high risk and retest everything.

**Phase:** Must be set up before any other work begins. This is the first task in the project.

---

## Moderate Pitfalls

---

### Pitfall 6: Global State Stranded in `app.py` Blocks Service Extraction

**What goes wrong:** `AI_JOBS`, `JOBS_DIR`, and `SUBSCRIPTION_PLANS` are module-level globals in `app.py`. When `services/ai_worker_service.py` is created, these either stay in `app.py` (creating an import dependency back into `app.py`) or are duplicated (creating drift). Neither is acceptable.

**Evidence in this codebase:** Lines 138-147 in `app.py` define these globals. The `background_ai_task` function on line ~3420 references `save_job_file` and `JOBS_DIR`. The AI chat routes (lines 3454-3509) reference `AI_JOBS` and `JOBS_DIR` directly.

**Prevention:** Move `AI_JOBS`, `JOBS_DIR`, and `SUBSCRIPTION_PLANS` into `services/ai_worker_service.py` and `services/subscription_service.py` respectively before extracting the route files that use them. Never import these from `app.py` in a service — that recreates the coupling.

**Phase:** Service extraction phase, before route extraction.

---

### Pitfall 7: `User` Class Must Be Accessible to All Blueprints

**What goes wrong:** The `User(UserMixin)` class (line 230 in `app.py`) is used by `load_user`, auth routes, and by `current_user` references everywhere. When `routes/auth.py` is created, it needs to construct `User` objects. If `User` stays in `app.py`, every Blueprint imports from `app.py` — circular imports return.

**Prevention:** Move the `User` class to `core/models.py` or `core/auth.py` immediately as step one of the refactoring. Everything imports `User` from there, never from `app.py`.

**Phase:** First step of app.py cleanup, before any Blueprint is created.

---

### Pitfall 8: Report Documents the Plan, Not the Reality

**What goes wrong:** `anser.md` requires a report section describing how `app.py` was split. The team writes this section based on the plan (APP-PY-renew.md) before the code is finished, or writes it about what they intended to do rather than what the code actually shows. The jury asks a question that exposes the gap — "show me the wallet service" — and it doesn't exist or doesn't match the description.

**Why it happens:** Report deadline and code deadline coincide. Writing the report early to save time means writing about a future state. The jury cross-references report against running code.

**Prevention:** Write the `anser.md` report section and module documentation only after the code is finished and working. Document what IS, not what the plan says. Use real file paths, real function names, real line counts.

**Phase:** Documentation phase — must happen after refactoring is complete on `dev`.

---

### Pitfall 9: `context_processor` and `errorhandler` Lost During Extraction

**What goes wrong:** `@app.context_processor` decorators (lines 163-176, 215-218) and `@app.errorhandler(CSRFError)` (line 221) are registered on the `app` object. They are easy to miss when extracting Blueprints because they are not routes — they look like supporting code. If they are not preserved in the factory, templates break (missing `csrf_token`, missing `project_name`) and CSRF errors return 500 instead of a graceful redirect.

**Prevention:** Keep a checklist of non-route registrations in `app.py`:
- `@app.context_processor inject_project_config` (line 163)
- `@app.context_processor inject_csrf_token` (line 215)
- `@app.errorhandler(CSRFError)` (line 221)
- `@login_manager.unauthorized_handler` (line 110)

These stay in `app.py` (the factory) or move to a dedicated `core/template_context.py` registered at factory time. They must not be lost.

**Phase:** app.py factory cleanup phase.

---

## Minor Pitfalls

---

### Pitfall 10: `url_prefix` Changes Break Direct URL Bookmarks and Fetch Calls

**What goes wrong:** Registering `auth_bp` with `url_prefix='/auth'` means `/signin` becomes `/auth/signin`. JavaScript `fetch('/signin', ...)` calls and any hardcoded URLs in templates break silently.

**Evidence in this codebase:** `auth_bp` is already registered with `url_prefix='/auth'` (line 333). Check all templates for hardcoded `/signin` or `/signup` strings.

**Prevention:** Search templates for hardcoded URL strings. Replace all with `url_for()`. If JavaScript needs URLs, pass them via data attributes from Jinja2: `data-url="{{ url_for('auth.signin') }}"`.

**Phase:** Template audit during Blueprint extraction.

---

### Pitfall 11: `limiter` Decorators on Blueprint Routes Require `init_app` Pattern

**What goes wrong:** The `Limiter` object on line 96 is created with `Limiter(get_remote_address, app=app, ...)`. When routes are extracted, the Blueprint's `@limiter.limit(...)` decorator will not work unless `limiter` is created without `app=app` first (`Limiter(get_remote_address)`) and initialized with `limiter.init_app(app)` in the factory.

**Evidence in this codebase:** Lines 267 and 303 apply `@limiter.limit()` directly to Blueprint routes inside `app.py`. Extracting these routes requires the extensions pattern.

**Prevention:** Refactor `limiter` to the `core/extensions.py` extensions pattern before extracting any Blueprint that uses rate limiting.

**Phase:** `core/extensions.py` creation, before any Blueprint extraction.

---

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|---------------|------------|
| Branching setup | Skipped under time pressure | Set up main/dev/demo branches on day 1 before any code changes |
| `core/extensions.py` creation | Circular imports if this step is skipped | Create extensions.py first, migrate all singletons, verify `python app.py` still works |
| `User` class extraction | Blueprint auth.py needs User, User is in app.py | Move User to `core/models.py` before creating any route file |
| Service extraction (wallet, subscription) | Globals `SUBSCRIPTION_PLANS` stranded in app.py | Move globals into respective service files first |
| Route extraction (billing, ai, workflows) | `url_for()` references break | Audit all templates for endpoint name changes before merging |
| `app.py` factory cleanup | context_processor and errorhandler lost | Explicit checklist of non-route registrations to preserve |
| Documentation (anser.md) | Written about plan not reality | Write after code is done; use real paths and function names |
| Demo branch merge | Merging broken dev into demo | 24-hour freeze on demo before presentation; full smoke test required |

---

## Jury-Specific Warnings

These pitfalls are specific to the retake context.

### Pitfall J1: Code and Report Describe Different Architectures

The jury failed the team partly because the system architecture figure was wrong. The same mistake can happen with the refactoring: the report describes the proposed `services/` layout, but the actual code still has logic in `app.py`. The jury will ask to see a specific service file — if it is empty, stub-only, or missing, the inconsistency undermines credibility.

**Prevention:** Do not write report sections about architecture until that architecture exists in the running code. The `anser.md` deliverable should be the last thing written, after the refactor is confirmed working on `dev`.

### Pitfall J2: Demo Branch Is Behind `dev`

The team fixes a bug in `dev` after merging to `demo`. Presentation uses `demo`. The jury sees the old bug. This happens when `dev` and `demo` diverge in the final hours.

**Prevention:** Final merge from `dev` to `demo` happens no later than the evening before presentation. No emergency commits to `dev` after that merge unless they are also cherry-picked to `demo`.

### Pitfall J3: Refactor Breaks the One Feature the Jury Demos

Wallet, OCR, and AI chat are the highest-visibility features. A refactoring error that breaks any of them is maximum impact on jury impression.

**Prevention:** Write a manual smoke test checklist covering: login, OCR upload, AI chat, wallet topup flow. Run this checklist on `demo` branch the morning of presentation.

---

## Sources

- Direct inspection of `app.py` (3514 lines, 2026-03-28)
- `DOCUMENTS/backend/APP-PY-renew.md` (refactoring proposal)
- `RETAKENOTES.md` (jury failure notes)
- `.planning/PROJECT.md` (project constraints and context)
- `.planning/codebase/CONCERNS.md` (tech debt and fragile areas analysis)
- `.planning/codebase/ARCHITECTURE.md` (architecture analysis)
- Confidence: HIGH — all pitfalls verified against actual code line numbers
