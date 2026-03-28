# Architecture Patterns — Flask Blueprint + Service Layer Refactoring

**Project:** Group Project AI/ML (USTH GEN14)
**Researched:** 2026-03-28
**Confidence:** HIGH — derived from direct codebase audit of the 3514-line app.py and the existing APP-PY-renew.md proposal

---

## Context: What Exists Right Now

The current `app.py` is a single-file "God Object" with:

- 110 `@app.route` decorators across every domain
- Business logic, raw SQL, background threads, and HTTP response formatting mixed in the same functions
- Global module-level initialization: `db_manager`, `auth_manager`, `automation_engine`, `agent_middleware`, `SUBSCRIPTION_PLANS`
- Two background thread patterns: a fire-and-forget `threading.Thread` for AI tasks, and a subprocess `run_dl_service` for the DL microservice
- 4 already-extracted components in `core/`: `database.py`, `auth.py`, `config.py`, `utils.py`, `workflow_engine.py`, `agent_middleware.py`
- An existing `core/services/` with `dl_client.py` and `analytics_service.py` (partial extraction already done)

The refactoring is NOT a greenfield design — it is a reorganization of existing, working code that must remain backwards-compatible throughout.

---

## Recommended Architecture

### Three-Layer Model

```
HTTP Request
     |
     v
+-------------------+
|  routes/          |  (Flask Blueprints — HTTP layer only)
|  auth.py          |  - Receive request, validate input
|  billing.py       |  - Check auth/permissions
|  workflows.py     |  - Call one service function
|  ai.py            |  - Return jsonify() or render_template()
|  dl_proxy.py      |  - Never contain business logic or SQL
|  admin.py         |
|  inventory.py     |
+-------------------+
     |
     v (plain Python function calls — no HTTP objects)
+-------------------+
|  services/        |  (Business Logic — HTTP-free)
|  wallet_service   |  - Accept plain Python types (int, str, dict)
|  subscription_svc |  - Return (success: bool, result: dict/str)
|  ai_worker_service|  - All SQL lives here
|  workflow_service |  - All external calls (AI API, DL service) here
|  dl_proxy_service |  - Testable without a Flask request context
|  inventory_service|
|  user_service     |
|  analytics_service|  (already exists in core/services/)
+-------------------+
     |
     v (SQL or abstraction)
+-------------------+
|  core/            |  (Infrastructure — shared across all layers)
|  database.py      |  - Database connection + cursor management
|  config.py        |  - All configuration constants
|  auth.py          |  - Auth helpers (already exists)
|  utils.py         |  - Shared utilities (already exists)
|  workflow_engine  |  - Workflow execution (already exists)
|  agent_middleware |  - AI agent context builder (already exists)
+-------------------+
     |
     v
+-------------------+
|  app.py           |  (App Factory — <100 lines)
|                   |  - create_app() function
|                   |  - Register all blueprints
|                   |  - Init Flask-Login, CSRF, Talisman, Limiter
|                   |  - Start DL service thread
+-------------------+
```

---

## Component Boundaries

### What Each Layer May and May Not Do

| Layer | May Import | May NOT Import | Returns |
|-------|------------|----------------|---------|
| `routes/` | `flask`, `flask_login`, `services/*`, `core/config` | `sqlite3`, `requests`, `threading` directly | `jsonify()`, `render_template()`, `redirect()` |
| `services/` | `core/database`, `core/config`, `requests`, `threading` | `flask`, `flask_login`, `request`, `jsonify` | Tuples `(bool, dict/str)` or plain dicts |
| `core/` | stdlib only, no cross-layer imports | `routes/`, `services/` | Instances, primitives |
| `app.py` | Everything for initialization only | Business logic | `Flask` app instance |

The critical boundary: **services must never import from flask or reference `request`/`current_user`**. User identity is passed in as a plain argument.

---

## Detailed Component Map

### routes/ (Blueprints)

| File | Blueprint Name | URL Prefix | Routes Covered (from app.py) |
|------|----------------|------------|------------------------------|
| `routes/auth.py` | `auth_bp` | `/auth`, `/` | `/signin`, `/signup`, `/logout`, `/auth/login/google`, `/auth/google/callback`, `/auth/connect/google` |
| `routes/billing.py` | `billing_bp` | `/api` | `/api/user/wallet`, `/api/user/wallet/topup`, `/api/admin/wallet/*`, `/api/user/subscription/*`, `/api/admin/subscription*`, `/wallet` |
| `routes/workflows.py` | `workflows_bp` | `/api` | `/api/workflows`, `/api/workspaces`, `/api/workspace/*`, `/api/scenarios*`, `/workspace`, `/workspace/builder` |
| `routes/ai.py` | `ai_bp` | `/api/ai` | `/api/ai/chat`, `/api/ai/history`, `/api/ai/status/<job_id>`, `/api/ai/upload` |
| `routes/dl_proxy.py` | `dl_bp` | `/api/dl` | `/api/dl/detect`, `/api/dl/forecast` |
| `routes/admin.py` | `admin_bp` | `/admin`, `/api/admin` | All `/admin/*` pages and `/api/admin/users*`, `/api/admin/analytics*`, `/api/manager/*` |
| `routes/inventory.py` | `inventory_bp` | `/api` | `/api/products*`, `/api/customers*`, `/api/imports*`, `/api/exports*`, `/api/sales*`, `/api/reports*`, `/api/automations*`, `/api/dashboard/stats` |
| `routes/pages.py` | `pages_bp` | `/` | `/dashboard`, `/settings`, `/wallet`, `/sale`, `/products`, etc. (page renders with no logic) |

### services/

| File | Responsibility | Key Functions |
|------|---------------|---------------|
| `services/wallet_service.py` | Topup, withdraw, pending approvals, balance queries | `get_wallet_balance(user_id)`, `process_topup(user_id, amount, evidence_path)`, `process_admin_withdrawal(user, data)`, `get_pending_transactions()`, `approve_transaction(txn_id, admin_user)` |
| `services/subscription_service.py` | Plan upgrades, expiry checks, auto-renew | `upgrade_subscription(user_id, plan_key, wallet_balance)`, `check_expired_subscriptions()`, `set_auto_renew(user_id, enabled)`, `get_subscription_history(filters)` |
| `services/ai_worker_service.py` | Job file I/O, background thread, history CRUD | `submit_ai_job(user_id, message)` -> `job_id`, `get_job_status(job_id)`, `background_ai_task(job_id, user_id, message)`, `get_chat_history(user_id)`, `clear_chat_history(user_id)` |
| `services/workflow_service.py` | Workflow and workspace CRUD, execution trigger | `list_workflows(user_id)`, `save_workflow(user_id, data)`, `delete_workflow(workflow_id, user_id)`, `execute_workflow_for_user(workflow_id, user_id)` |
| `services/dl_proxy_service.py` | HTTP client to DL service on port 5001 | `detect_invoice(file_bytes, filename)`, `forecast_quantity(payload)` |
| `services/inventory_service.py` | Products, customers, imports/exports, sales, reports | One function per endpoint group |
| `services/user_service.py` | Profile updates, settings, session info, permissions | `update_profile(user_id, data)`, `get_user_permissions(user_id)` |

### core/ (unchanged — already extracted)

| File | Role |
|------|------|
| `core/database.py` | `Database` class — connection factory, all raw SQL helpers |
| `core/config.py` | `Config` class — all constants, secrets loading |
| `core/auth.py` | `AuthManager` — login/signup logic |
| `core/agent_middleware.py` | `AgentMiddleware` — builds AI system context |
| `core/workflow_engine.py` | `execute_workflow()` — workflow execution logic |
| `core/services/dl_client.py` | `DLClient` — HTTP client to dl_service (move to `services/dl_proxy_service.py` or keep as internal dep) |
| `core/services/analytics_service.py` | Analytics (already extracted — keep as-is) |

---

## Data Flow

### Standard API Request (e.g., Wallet Topup)

```
Browser POST /api/user/wallet/topup
  -> routes/billing.py: topup()
       - flask_login: current_user check
       - request.get_json() / request.files
       - call: wallet_service.process_topup(current_user.id, amount, evidence_path)
         -> services/wallet_service.py: process_topup()
              - Validate inputs (pure Python)
              - db_manager.get_connection() -> INSERT transaction
              - Return (True, {"message": "..."}) or (False, "error string")
       - jsonify result and return HTTP response
  -> Browser receives JSON
```

### AI Chat (Async Job Pattern)

```
Browser POST /api/ai/chat
  -> routes/ai.py: ai_chat()
       - call: ai_worker_service.submit_ai_job(current_user.id, message)
         -> services/ai_worker_service.py: submit_ai_job()
              - save_job_file(job_id, {"status": "pending"})
              - threading.Thread(target=background_ai_task, ...).start()
              - Return job_id
       - jsonify({"status": "processing", "job_id": job_id})

Browser polls GET /api/ai/status/<job_id>
  -> routes/ai.py: ai_job_status()
       - call: ai_worker_service.get_job_status(job_id)
         -> reads jobs/<job_id>.json
       - jsonify(job_data)
```

### DL Proxy Flow

```
Browser POST /api/dl/detect (multipart file)
  -> routes/dl_proxy.py: api_dl_detect()
       - call: dl_proxy_service.detect_invoice(file_bytes, filename)
         -> services/dl_proxy_service.py
              - HTTP POST to http://localhost:5001/api/ocr/detect
              - (uses core/services/dl_client.py internally)
              - Return result dict or raise
       - jsonify({"success": True, "data": result})
```

---

## App Factory Pattern (Target app.py)

```python
# app.py — target state (<100 lines)

from flask import Flask
from flask_login import LoginManager
from flask_wtf.csrf import CSRFProtect
from flask_talisman import Talisman
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
from authlib.integrations.flask_client import OAuth

from core.config import Config
from core.database import Database


def create_app(config_object=None):
    app = Flask(__name__, template_folder='ui/templates')
    cfg = config_object or Config()

    # --- Configuration ---
    app.config['SECRET_KEY'] = cfg.SECRET_KEY
    # ... other config keys loaded from cfg ...

    # --- Extensions ---
    db_manager = Database()
    app.extensions['database'] = db_manager

    login_manager = LoginManager()
    login_manager.init_app(app)
    login_manager.login_view = 'auth_bp.signin'

    CSRFProtect(app)
    Talisman(app, ...)
    Limiter(get_remote_address, app=app, ...)

    # --- Blueprints ---
    from routes.auth import auth_bp
    from routes.billing import billing_bp
    from routes.workflows import workflows_bp
    from routes.ai import ai_bp
    from routes.dl_proxy import dl_bp
    from routes.admin import admin_bp
    from routes.inventory import inventory_bp
    from routes.pages import pages_bp

    app.register_blueprint(auth_bp)
    app.register_blueprint(billing_bp)
    # ... etc ...

    # --- DL Service Thread ---
    import threading
    from run_dl_service import run_dl_service
    threading.Thread(target=run_dl_service, daemon=True).start()

    return app


if __name__ == '__main__':
    app = create_app()
    app.run(debug=True)
```

Key difference from current state: `db_manager` is NOT a global variable. It is passed into services as a parameter or accessed via `current_app.extensions['database']` inside request contexts. This eliminates the implicit global dependency that currently makes testing hard.

---

## Suggested Build Order

Dependencies determine what must be refactored first. A later step cannot be completed if it depends on an earlier one still being a tangled mess.

### Phase 1 — Foundation (no dependencies, unblock everything)

**Prerequisite: nothing.** These do not depend on any route or business logic.

1. `core/config.py` — already extracted, may need `SUBSCRIPTION_PLANS` constant moved here from `app.py`
2. `core/database.py` — already extracted, no changes needed
3. `app.py` scaffolded as `create_app()` factory — create the shell, temporarily re-register nothing (keep all routes in app.py until Phases 2-3 complete)

**Why first:** Every blueprint and service imports from `core/`. If `core/` is incomplete, downstream extractions fail.

### Phase 2 — Service Layer (can be parallelized across team members)

Extract business logic OUT of `app.py` into `services/`. This is safe because `app.py` still holds all routes — so no user-facing breakage occurs. Each service file is independent.

Team split suggestion:
- Member A: `services/wallet_service.py` + `services/subscription_service.py` (lines ~2121-2892 in app.py)
- Member B: `services/ai_worker_service.py` (lines ~3380-3450 in app.py)
- Member C: `services/inventory_service.py` (lines ~1446-1833 in app.py — products, customers, imports/exports, sales)
- Member D: `services/workflow_service.py` + `services/user_service.py` (lines ~3119-3200 + settings/profile)

**Why second:** Services have no dependency on Blueprint structure. Extracting them first means when Blueprints are wired up (Phase 3), the functions they call already exist.

### Phase 3 — Blueprint Routes (depends on Phase 2)

Create `routes/` files, import from `services/`, migrate `@app.route` decorators one Blueprint at a time. Test after each Blueprint is migrated.

Recommended order within Phase 3:

1. `routes/auth.py` — fewest dependencies, easiest to isolate (auth is self-contained via `core/auth.py`)
2. `routes/dl_proxy.py` — trivial, only 2 routes, delegates entirely to `dl_proxy_service`
3. `routes/ai.py` — 4 routes, depends on `ai_worker_service`
4. `routes/billing.py` — depends on `wallet_service` + `subscription_service`
5. `routes/workflows.py` — depends on `workflow_service`
6. `routes/inventory.py` — largest group (~35 routes), do last to reduce risk
7. `routes/admin.py` — depends on most services, do last

### Phase 4 — Wire App Factory + Delete from app.py

Register all blueprints in `create_app()`. Delete the migrated `@app.route` blocks from `app.py`. Verify `app.py` is under 100 lines.

### Phase 5 — Tests

Write unit tests against `services/` functions — now possible because they have no Flask dependency.

---

## Git Branching Strategy

### Branch Structure

```
main          <- production-stable / post-jury release
  |
  dev         <- integration branch (canary testing)
  |  \
  |   demo    <- jury presentation branch (frozen feature set)
  |
  feature/*   <- individual developer branches (short-lived)
```

### Branch Roles and Rules

| Branch | Purpose | Who Merges To It | Stability Contract |
|--------|---------|------------------|--------------------|
| `main` | Post-retake final state, permanent history | Merge from `dev` after jury | Always deployable, never broken |
| `dev` | Active integration — CI runs here | PRs from `feature/*` | Functional but may have minor bugs |
| `demo` | Frozen jury demo build | One-time branch from `dev` when demo is ready | NEVER receive new commits after freeze; hotfixes only |
| `feature/<name>` | One feature per developer | Developer opens PR to `dev` | Work-in-progress, may be broken |

### Workflow for Each Developer

```
# 1. Always branch from dev
git checkout dev
git pull origin dev
git checkout -b feature/extract-wallet-service

# 2. Work on your single domain (wallet, AI, inventory, etc.)
# 3. Open PR to dev when done
# 4. Another team member reviews and merges to dev
```

### Conflict Prevention Rules

The primary cause of merge conflicts in the current setup is that all 4+ team members edit the same `app.py`. The Blueprint refactor eliminates this because each team member owns a different file:

| Domain | File Owned | Developer |
|--------|-----------|-----------|
| Wallet + Subscription | `services/wallet_service.py`, `services/subscription_service.py`, `routes/billing.py` | Member A |
| AI Chat | `services/ai_worker_service.py`, `routes/ai.py` | Member B |
| Inventory (products/sales/imports/exports) | `services/inventory_service.py`, `routes/inventory.py` | Member C |
| Workflows + User | `services/workflow_service.py`, `services/user_service.py`, `routes/workflows.py` | Member D |
| App factory + Admin + DL proxy | `app.py`, `routes/admin.py`, `routes/dl_proxy.py`, `routes/auth.py` | Team lead |

With this split, each developer touches completely different files. Merge conflicts become nearly impossible during Phase 2 (service extraction) since everyone is creating new files, not editing shared ones.

### Demo Branch Protocol

```
# When dev is stable enough for jury demo (T-2 days before presentation):
git checkout dev
git pull origin dev
git checkout -b demo

# Tag the demo snapshot
git tag v0.1-jury-demo

# If a critical bug is found in demo after freeze:
git checkout demo
git checkout -b hotfix/critical-bug-name
# fix the bug
git checkout demo
git merge hotfix/critical-bug-name
# DO NOT merge hotfix back to dev automatically — evaluate separately
```

The `demo` branch must NEVER receive `dev` merges after the freeze. The jury sees `demo`. Developers continue on `dev` after the freeze without fear of breaking the presentation build.

---

## Scalability Considerations

This refactoring is for a jury demo, not production at scale. These notes are for the jury's architecture questions:

| Concern | Current (God Object) | After Refactor | At Real Scale |
|---------|---------------------|----------------|---------------|
| Test coverage | Near zero (logic entangled with HTTP) | Unit tests possible on services | Same pattern, add integration tests |
| New feature addition | All developers edit app.py, conflicts | Each feature is an isolated Blueprint + service | Same pattern |
| Multiple developers | High merge conflict risk | Each owns different files | Same pattern |
| Replace SQLite | Hard (raw SQL spread across 3514 lines) | Services own all SQL — swap in one place per domain | ORM migration feasible |
| Replace AI backend | Impossible to isolate | `ai_worker_service.py` owns the HTTP call | One file to change |
| Horizontal scaling | Not applicable | Not applicable (SQLite is single-file) | Would need PostgreSQL + session store |

---

## Anti-Patterns to Avoid

### Anti-Pattern 1: Thin Services, Fat Routes

**What goes wrong:** Developers move the route decorator but leave all logic inline in the route function.

```python
# WRONG — this is just app.py code moved to a Blueprint file
@billing_bp.route('/api/user/wallet/topup', methods=['POST'])
@login_required
def topup():
    data = request.get_json()
    conn = db_manager.get_connection()  # raw SQL in route
    c = conn.cursor()
    c.execute("INSERT INTO transactions ...")  # business logic in route
    ...
```

**Prevention:** Route functions should be 10-15 lines max. If a route function contains SQL, it is wrong.

### Anti-Pattern 2: Services That Import Flask

**What goes wrong:** `from flask import request` in a service file.

**Consequences:** The service cannot be tested without a Flask app context. The separation of concerns is broken.

**Prevention:** Services receive all data as function arguments. User identity is `user_id: int`, not `current_user`.

### Anti-Pattern 3: Circular Imports

**What goes wrong:** `routes/billing.py` imports from `services/wallet_service.py`, which imports from `routes/billing.py` for a helper.

**Prevention:** Data flows strictly downward: routes -> services -> core. Services never import from routes.

### Anti-Pattern 4: Global db_manager in Services

**What goes wrong:** `from app import db_manager` inside a service file creates a circular import (app.py imports services, services import app.py).

**Prevention:** Pass `db_manager` into service functions as a parameter, or use `flask.current_app.extensions['database']` inside request-context functions only. For background threads (ai_worker), instantiate `Database()` locally inside the thread function — exactly as the existing `background_ai_task` already does.

### Anti-Pattern 5: Merging to demo After Freeze

**What goes wrong:** A developer pushes a half-finished feature to `demo` the night before the jury presentation.

**Prevention:** The demo branch is protected after the tag. Only hotfixes (one-liner bug fixes) are permitted on `demo`. All new work goes to `dev`.

---

## Sources

- Direct codebase audit: `app.py` (3514 lines), `core/database.py`, `core/config.py`, `core/services/dl_client.py`
- Project proposal: `DOCUMENTS/backend/APP-PY-renew.md`
- Project context: `.planning/PROJECT.md`
- Confidence: HIGH — all component boundaries derived from actual code, not speculation
