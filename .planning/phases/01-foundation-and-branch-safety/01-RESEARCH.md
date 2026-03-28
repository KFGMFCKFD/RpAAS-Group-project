# Phase 1: Foundation and Branch Safety - Research

**Researched:** 2026-03-28
**Domain:** Flask app factory pattern, circular import prevention, Git branching strategy
**Confidence:** HIGH (all findings grounded in direct codebase inspection of app.py at 3514 lines)

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Branch Workflow**
- D-09: 6 long-lived branches: `main`, `dev`, `demo`, `backend`, `frontend`, `mixed`
- D-10: Merge flow: frontend + backend -> mixed (integration) -> dev (canary) -> main (production)
- D-11: Demo = fork from dev at stable point, freeze 48 hours before jury presentation
- D-12: No branch protection rules — team trusts each other, anyone pushes anywhere
- D-13: All branches are long-lived (persist throughout the project)
- D-14: Multiple team members use Claude Code — `.planning/` files need careful merge strategy:
  - Shared files (ARCHITECTURE.md, codebase maps): merge for combined knowledge
  - Session-specific files (STATE.md, research stages): add to .gitignore to avoid conflicts

**Backwards Compatibility**
- D-15: Incremental migration — move one Blueprint at a time, app.py shrinks gradually
- D-16: Manual smoke test after each Blueprint move (start app, click through main features)

### Claude's Discretion

Claude has full discretion on all technical implementation details (D-01 through D-08):
- D-01: Choose best init pattern for extensions (lazy init vs factory-only)
- D-02: Choose how services access db_manager (shared instance, app.extensions, or parameter passing)
- D-03: Choose where User class (UserMixin) should live (core/models.py, core/auth.py, or core/extensions.py)
- D-04: Choose where OAuth configuration moves (inside create_app or core/oauth.py)
- D-05: Choose how agent_middleware and automation_engine are initialized (extensions vs on-demand)
- D-06: Choose where SUBSCRIPTION_PLANS dict lives (core/config.py or services/subscription_service.py)
- D-07: Choose where DL service thread spawning happens (inside create_app or __main__ block)
- D-08: Choose how background workers access config in factory pattern (app context or param passing)

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| FOUND-01 | Application uses factory pattern (`create_app()`) instead of module-level Flask instance | App factory pattern section below; existing app.py line 37 creates module-level `app` — must be wrapped in `create_app()` |
| FOUND-02 | Shared extensions (db_manager, login_manager, csrf, limiter) live in `core/extensions.py` to prevent circular imports | Extensions pattern section below; current globals at app.py lines 96-134 must move |
| FOUND-03 | Git branching strategy established with main, dev, and demo branches (plus backend, frontend, mixed per D-09) | Branch setup section below; no branch strategy exists yet |
| FOUND-04 | Demo branch protection — frozen 24-48h before jury presentation (no hard protection rules per D-12) | Demo freeze protocol section below |
</phase_requirements>

---

## Summary

Phase 1 is a structural preparation phase — no functional changes, only reorganization. The codebase currently has a 3514-line `app.py` acting as a God Object with all Flask extensions, route logic, and business logic interleaved at module level. Three things must happen in this phase: (1) the 6-branch Git strategy must be established so team members can work independently, (2) all shared singleton objects must be extracted to `core/extensions.py` so that future Blueprint files can import them without creating circular import cycles, and (3) the module-level `app` creation must be wrapped in a `create_app()` factory function while keeping all 113 existing routes working.

The critical constraint is ordering: extensions.py must be created and verified working BEFORE any other changes. The circular import problem is the most dangerous failure mode in this phase — if extensions are not in place, any future Blueprint import will crash the app on startup. The branch strategy must be established first (before any code changes) so team members have the correct base branches from which to work.

The app already has partial infrastructure in place: `core/` has `database.py`, `auth.py`, `config.py`; `routes/` directory exists but is empty (only `__pycache__`); `services/` directory exists but is empty (only `__pycache__`). The auth Blueprint (`auth_bp`) is already defined inline in `app.py` at line 262 and registered at line 333 — it is a Blueprint in name but not yet in a separate file. Flask-Login's `login_view` is already correctly set to `'auth.signin'` (line 107).

**Primary recommendation:** Create branches first, then `core/extensions.py` with all singletons extracted, then wrap `app.py` in `create_app()` — in that exact order, verifying `python app.py` starts clean after each step.

---

## Standard Stack

### Core (already installed — from existing app.py imports)

| Library | Version | Purpose | Notes |
|---------|---------|---------|-------|
| Flask | 3.0-3.x | Web framework, factory pattern | Already used; `create_app()` is a Flask idiom |
| Flask-Login | 0.6+ | User session management | `login_manager.init_app(app)` pattern needed |
| Flask-WTF | 1.2+ | CSRF protection | `csrf.init_app(app)` pattern needed |
| Flask-Talisman | 1.1+ | Security headers | Instantiated directly in `Talisman(app, ...)` |
| Flask-Limiter | 3.5+ | Rate limiting | Currently uses `app=app` — must switch to `init_app` |
| authlib | 1.x | OAuth2 client | `OAuth(app)` — must move inside factory |

No new packages need to be installed for Phase 1. All required libraries are already present.

### Git (branching)

| Tool | Purpose | Notes |
|------|---------|-------|
| Git | Version control, branch management | Already in use on `main` branch |

No new tooling required for the branch setup task.

---

## Architecture Patterns

### Recommended Project Structure (after Phase 1)

```
app.py                   # <-- becomes create_app() factory only (~60 lines)
core/
  extensions.py          # NEW: all shared singletons (db_manager, login_manager, csrf, limiter)
  models.py              # NEW: User(UserMixin) class, moved from app.py line 230
  database.py            # unchanged
  auth.py                # unchanged
  config.py              # unchanged
  utils.py               # unchanged
routes/                  # empty now — populated in Phase 3
services/                # empty now — populated in Phase 2
```

### Pattern 1: Extensions Module (core/extensions.py)

**What:** All Flask extension objects and shared singletons initialized WITHOUT an app instance, then bound to an app in `create_app()` using `init_app()`.

**When to use:** Mandatory when Blueprints in separate files need to import shared objects (limiter, csrf, login_manager, db_manager) — which is every Blueprint.

**The init_app() contract:** Flask extensions that support it accept `init_app(app)` as a deferred binding. The extension object is created once (at import time of `extensions.py`), bound to a specific app instance only inside `create_app()`. Blueprint files import the object from `extensions.py` — they never import from `app.py`.

```python
# core/extensions.py
from flask_login import LoginManager
from flask_wtf.csrf import CSRFProtect
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
from core.database import Database

# Created without app — bound later via init_app()
login_manager = LoginManager()
csrf = CSRFProtect()
limiter = Limiter(get_remote_address)
db_manager = Database()
```

```python
# app.py — create_app() factory
from core.extensions import login_manager, csrf, limiter, db_manager

def create_app(config_object=None):
    app = Flask(__name__, template_folder='ui/templates')
    cfg = config_object or Config()

    # Configuration
    app.config['SECRET_KEY'] = cfg.SECRET_KEY
    app.config['WTF_CSRF_ENABLED'] = True
    app.config['RATELIMIT_ENABLED'] = False
    app.config['PERMANENT_SESSION_LIFETIME'] = timedelta(days=7)
    app.config['SESSION_REFRESH_EACH_REQUEST'] = True
    app.config['JSON_AS_ASCII'] = False
    app.config['SESSION_COOKIE_SECURE'] = False
    app.config['PREFERRED_URL_SCHEME'] = 'http'

    # Security middleware
    app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_proto=1, x_host=1, x_prefix=1)

    # Extensions bound to this app
    db_manager.init_app(app) if hasattr(db_manager, 'init_app') else None
    app.extensions['database'] = db_manager

    login_manager.init_app(app)
    login_manager.login_view = 'auth.signin'

    csrf.init_app(app)
    limiter.init_app(app)

    Talisman(app, force_https=False, content_security_policy={...}, ...)

    # OAuth — configured inside factory
    _configure_oauth(app)

    # Non-route registrations (context processors, error handlers)
    _register_template_helpers(app)

    # Blueprints — imported INSIDE the function to avoid circular imports at module load time
    from routes.auth import auth_bp
    app.register_blueprint(auth_bp, url_prefix='/auth')
    # (other blueprints registered here in Phase 3)

    # DL service thread — inside factory or in __main__ block (see D-07 recommendation below)
    return app


if __name__ == '__main__':
    app = create_app()
    app.run(debug=True, port=5000)
```

**Critical:** Blueprint imports must be INSIDE `create_app()`, not at the top of `app.py`. This is the mechanism that breaks circular imports — by the time a Blueprint file is imported, `core/extensions.py` already has the singleton objects available without any reference back to `app.py`.

### Pattern 2: User Class in core/models.py

**What:** Move the `User(UserMixin)` class (currently at app.py line 230) to `core/models.py`.

**Why:** Every Blueprint that handles auth (auth routes, any route that constructs a User object) would otherwise import from `app.py`, recreating the circular dependency.

```python
# core/models.py
from flask_login import UserMixin

class User(UserMixin):
    def __init__(self, id, email, first_name, last_name, role='user', google_token=None, avatar=None):
        self.id = id
        self.email = email
        self.first_name = first_name
        self.last_name = last_name
        self.role = role
        self.google_token = google_token
        self.avatar = avatar

    @property
    def name(self):
        return f"{self.first_name} {self.last_name}"
```

```python
# app.py — user_loader registered inside create_app()
from core.models import User
from core.extensions import login_manager, db_manager
from core.auth import AuthManager

def create_app(...):
    ...
    auth_manager = AuthManager(db_manager)

    @login_manager.user_loader
    def load_user(user_id):
        try:
            user_data = auth_manager.get_user_by_id(int(user_id))
            if user_data:
                return User(
                    user_data['id'], user_data['email'],
                    user_data.get('first_name', ''), user_data.get('last_name', ''),
                    user_data.get('role', 'user'), user_data.get('google_token'),
                    user_data.get('avatar')
                )
        except Exception as e:
            print(f"Error loading user {user_id}: {e}")
        return None
    ...
```

### Pattern 3: Context Processors and Error Handlers Preserved in Factory

**What:** The non-route app.py registrations (`@app.context_processor`, `@app.errorhandler`, `@login_manager.unauthorized_handler`) must be explicitly carried into the factory. They are easy to lose.

**Checklist of registrations that must survive the refactor:**
- `@app.context_processor inject_project_config` (app.py line 163)
- `@app.context_processor inject_csrf_token` (app.py line 215)
- `@app.errorhandler(CSRFError) handle_csrf_error` (app.py line 221)
- `@login_manager.unauthorized_handler login_unauthorized` (app.py line 110)
- Helper functions: `parse_db_datetime`, `format_display_datetime`, `parse_metadata`, `format_plan_dict` — keep in `core/utils.py` or `app.py` (not lost)

**Recommended approach:** Extract these into a `_register_template_helpers(app)` private function called inside `create_app()`. Keeps factory clean while preserving all registrations.

### Pattern 4: Git Branch Strategy

**What:** 6 long-lived branches with a layered merge flow (locked as D-09 through D-13).

```
main       <- production-stable, post-jury release
  |
  dev      <- integration, canary testing — all final integrations land here
  |  \
  |   demo <- jury presentation branch (frozen T-48h before presentation)
  |
  mixed    <- integration of frontend + backend work before promoting to dev
  |  \
  |   backend   <- owned by backend developer (the user)
  |   frontend  <- owned by frontend developer (teammate)
```

**Branch creation commands:**
```bash
# From current main branch
git checkout main
git checkout -b dev
git checkout -b demo
git checkout -b mixed
git checkout -b backend
git checkout -b frontend
git push origin dev demo mixed backend frontend
```

Per D-12: No branch protection rules. Per D-11: Demo frozen 48h before jury — enforced by team agreement, not GitHub rules.

### Pattern 5: .planning/ Gitignore Strategy

**What:** Some `.planning/` files must be shared across team members; session-specific files must be ignored to avoid conflicts.

**Files to keep tracked (shared knowledge):**
- `.planning/codebase/ARCHITECTURE.md`
- `.planning/codebase/STRUCTURE.md`
- `.planning/codebase/CONVENTIONS.md`
- `.planning/REQUIREMENTS.md`
- `.planning/PROJECT.md`
- `.planning/phases/*/` (planning artifacts)
- `.planning/research/`

**Files to add to .gitignore (session-specific, conflict-prone):**
```gitignore
# GSD session-specific planning files (each team member has their own)
.planning/STATE.md
.planning/phases/*/*-CONTEXT.md
```

**Reasoning:** STATE.md tracks per-session position (current plan, resume point). Two team members simultaneously at different plan positions will always conflict. CONTEXT.md files are generated per discussion session and are not consumed by the running app.

### Anti-Patterns to Avoid

- **Importing from app.py in any Blueprint or service:** Blueprint files must import shared objects only from `core/extensions.py` and `core/models.py`. Never `from app import db_manager`.
- **Top-level Blueprint imports in app.py:** Blueprint imports must be inside the `create_app()` body — if they are at the top of the file, the circular import occurs at module load time before `create_app()` runs.
- **Limiter with `app=app` in extensions.py:** `Limiter(get_remote_address, app=app, ...)` at module level will fail because there is no `app` yet when `extensions.py` is imported. Use `Limiter(get_remote_address)` and call `limiter.init_app(app)` inside the factory.
- **Talisman doesn't support init_app:** Flask-Talisman is initialized with `Talisman(app, ...)` directly inside `create_app()`, not in `extensions.py`. This is the correct usage pattern.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Deferred extension initialization | Custom bind/registry system | Flask's `init_app()` pattern | Built-in to Flask extensions ecosystem; zero additional code |
| Branch protection without GitHub Pro | Shell hooks, scripts | Team agreement + demo freeze protocol | Team trusts each other (D-12); hooks add complexity with no payoff |
| Circular import detection | Custom import hooks | Run `python app.py` after each step | Python's import error message is immediate and clear |
| User session management | Custom session middleware | Flask-Login (already installed) | Already configured; `login_manager.init_app(app)` is one line |

**Key insight:** This phase is reorganization, not new functionality. No new libraries, no new systems. Every tool needed is already installed and working in app.py — the task is moving the wires, not adding new ones.

---

## Claude's Discretion Recommendations

These are the D-01 through D-08 decisions left to Claude. Based on the codebase audit:

**D-01 (Extension init pattern):** Use **factory-only init** — extensions created in `core/extensions.py` without app argument, bound in `create_app()` via `init_app()`. This is the standard Flask pattern and the only approach that supports testing with multiple app instances.

**D-02 (How services access db_manager):** Use **module-level import from `core/extensions.py`** — `from core.extensions import db_manager`. This works cleanly because `db_manager = Database()` in extensions.py creates the instance at import time (Database class doesn't need the Flask app — it connects to SQLite directly). For background threads, instantiate `Database()` locally inside the thread function, as the existing `background_ai_task` already does.

**D-03 (Where User class lives):** `core/models.py` — dedicated models file following Python conventions. Avoids bloating `core/auth.py` (which is already a full AuthManager class) and avoids importing Flask-Login into `core/extensions.py` (which would create an extensions.py that is harder to read).

**D-04 (Where OAuth configuration moves):** Inside `create_app()` directly — a private `_configure_oauth(app)` helper function within `app.py`. OAuth config depends on `app.config` values, so it belongs inside the factory. No need for a separate `core/oauth.py` file for Phase 1 scope.

**D-05 (agent_middleware and automation_engine initialization):** **On-demand inside create_app()**, stored in `app.extensions`. These are heavy objects (each takes `db_manager` as constructor arg). They don't need to be in `extensions.py` — they belong to specific route domains and can be initialized inside the factory and stored in `app.extensions['agent_middleware']` and `app.extensions['automation_engine']`.

**D-06 (Where SUBSCRIPTION_PLANS lives):** `core/config.py` for Phase 1. It's a constant dict with no Flask dependency. Move it into `Config` as a class attribute. In Phase 2, it migrates to `services/subscription_service.py` when that service is extracted.

**D-07 (Where DL service thread spawning happens):** In the `if __name__ == '__main__':` block, NOT inside `create_app()`. Reason: the thread spawns a subprocess that starts a separate server process. Running it inside `create_app()` would spawn the DL process even when running tests or when a WSGI server imports the app for workers. The `__main__` block is intentional — it only runs when the file is executed directly.

**D-08 (How background workers access config in factory pattern):** Background threads (`background_ai_task`) instantiate `Database()` locally at thread start time — this is what the current code already does and it works correctly. Config values (`JOBS_DIR`, etc.) should be module-level constants in `services/ai_worker_service.py` (Phase 2), not passed as parameters.

---

## Common Pitfalls

### Pitfall 1: Limiter Created With app= in extensions.py

**What goes wrong:** `limiter = Limiter(get_remote_address, app=app, default_limits=[...])` in extensions.py fails with NameError because `app` doesn't exist at that point.

**Why it happens:** The current app.py creates `limiter` on line 96 in the same file as the `app` object. When this is split, the `app` reference is no longer available.

**How to avoid:** In `core/extensions.py`, use `limiter = Limiter(get_remote_address)` with NO `app` argument and NO `default_limits`. Set `default_limits` inside `create_app()` via `app.config['RATELIMIT_DEFAULT_LIMITS'] = ["20000 per day", "5000 per hour"]` BEFORE calling `limiter.init_app(app)`.

**Warning signs:** `NameError: name 'app' is not defined` on startup.

### Pitfall 2: Circular Import at Module Level

**What goes wrong:** A Blueprint file has `from app import db_manager` at the top level, and `app.py` has `from routes.auth import auth_bp` at the top level. Python sees the cycle and fails with `ImportError: cannot import name 'db_manager' from partially initialized module 'app'`.

**Why it happens:** Blueprint files need shared objects. The instinct is to import them from where they currently live — `app.py`.

**How to avoid:** Blueprint imports in `app.py` must be inside `create_app()`. Blueprint files must import from `core/extensions.py`, never from `app.py`.

**Warning signs:** `ImportError: cannot import name X from partially initialized module` at startup.

### Pitfall 3: Talisman Applied Before Config is Set

**What goes wrong:** `Talisman(app, ...)` applied too early in `create_app()`, before `app.config['SESSION_COOKIE_SECURE']` and other security config is set. Talisman reads config at instantiation time.

**How to avoid:** Apply `Talisman(app, ...)` AFTER all `app.config[...]` assignments, not before.

### Pitfall 4: User Loader Registered Outside App Context

**What goes wrong:** `@login_manager.user_loader` callback registered at module level in `app.py` while `login_manager` is in `extensions.py`. The callback is registered before `login_manager.init_app(app)` is called, so it is bound to the wrong (no) app context.

**How to avoid:** Register `@login_manager.user_loader` INSIDE `create_app()`, after `login_manager.init_app(app)`.

### Pitfall 5: auth_bp Registration Breaks After Extraction

**What goes wrong:** The current `auth_bp` (inline at app.py line 262) uses `@limiter.limit(...)` decorators directly. When moved to `routes/auth.py`, this imports `limiter` from `core/extensions.py` — but `limiter` is not yet initialized with an app. The decorator call fails.

**Important:** Flask-Limiter decorators are evaluated at decoration time (when the function is defined). However, with the `init_app()` pattern, Flask-Limiter 3.x evaluates limits lazily at request time, not at decoration time — so the import of `limiter` from `extensions.py` is safe even before `limiter.init_app(app)` has been called, as long as no requests are processed before that.

**Verification:** After creating `core/extensions.py` and before moving `auth_bp` to a separate file, confirm `python app.py` starts cleanly and `/auth/signin` returns 200.

### Pitfall 6: .planning/ Merge Conflicts Between Team Members

**What goes wrong:** Two team members both have active Claude Code sessions. Both write to `.planning/STATE.md` with different session positions. Git merge fails or incorrectly resolves.

**How to avoid:** Add session-specific files to `.gitignore` immediately when setting up branches. Specifically: `.planning/STATE.md` and optionally `.planning/phases/*/*-CONTEXT.md`.

**Warning signs:** Merge conflict markers in STATE.md after `git pull`.

---

## Code Examples

Verified patterns from direct codebase inspection:

### extensions.py (target state)

```python
# core/extensions.py
# Source: derived from app.py lines 96-134 + Flask init_app pattern

from flask_login import LoginManager
from flask_wtf.csrf import CSRFProtect
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address
from core.database import Database

# Singletons created without app — bound in create_app() via init_app()
db_manager = Database()
login_manager = LoginManager()
csrf = CSRFProtect()
limiter = Limiter(get_remote_address)
# Note: default_limits set via app.config in create_app(), not here
```

### create_app() skeleton (target app.py)

```python
# app.py — target state after Phase 1
# Source: derived from app.py lines 37-334

import os
import json
from datetime import timedelta

from flask import Flask, redirect, url_for, flash, session, jsonify, request
from werkzeug.middleware.proxy_fix import ProxyFix
from flask_talisman import Talisman
from flask_wtf.csrf import CSRFError, generate_csrf
from authlib.integrations.flask_client import OAuth

from core.config import Config
from core.models import User
from core.auth import AuthManager
from core.automation_engine import AutomationEngine
from core.agent_middleware import AgentMiddleware
from core.extensions import db_manager, login_manager, csrf, limiter


def create_app(config_object=None):
    app = Flask(__name__, template_folder='ui/templates')
    cfg = config_object or Config()

    # --- Configuration ---
    app.config['SECRET_KEY'] = os.environ.get('SECRET_KEY', 'change_me_to_a_secure_random_value')
    app.config['WTF_CSRF_ENABLED'] = True
    app.secret_key = app.config['SECRET_KEY']
    app.config['SESSION_COOKIE_SECURE'] = False
    app.config['PREFERRED_URL_SCHEME'] = 'http'
    app.config['PERMANENT_SESSION_LIFETIME'] = timedelta(days=7)
    app.config['SESSION_REFRESH_EACH_REQUEST'] = True
    app.config['JSON_AS_ASCII'] = False
    app.config['RATELIMIT_ENABLED'] = False
    app.config['RATELIMIT_DEFAULT_LIMITS'] = ["20000 per day", "5000 per hour"]

    # --- Security Middleware ---
    app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_proto=1, x_host=1, x_prefix=1)
    Talisman(app, force_https=False, content_security_policy={...}, ...)

    # --- Extensions ---
    app.extensions['database'] = db_manager
    login_manager.init_app(app)
    login_manager.login_view = 'auth.signin'
    csrf.init_app(app)
    limiter.init_app(app)

    # --- OAuth ---
    _configure_oauth(app)

    # --- Dependent services (need db_manager) ---
    auth_manager = AuthManager(db_manager)
    app.extensions['auth_manager'] = auth_manager
    app.extensions['automation_engine'] = AutomationEngine(db_manager)
    app.extensions['agent_middleware'] = AgentMiddleware(db_manager)

    # --- User Loader ---
    @login_manager.user_loader
    def load_user(user_id):
        try:
            user_data = auth_manager.get_user_by_id(int(user_id))
            if user_data:
                return User(
                    user_data['id'], user_data['email'],
                    user_data.get('first_name', ''), user_data.get('last_name', ''),
                    user_data.get('role', 'user'), user_data.get('google_token'),
                    user_data.get('avatar')
                )
        except Exception as e:
            print(f"Error loading user {user_id}: {e}")
        return None

    # --- Template Helpers (context processors, error handlers) ---
    _register_template_helpers(app, cfg)

    # --- Blueprints (imported INSIDE factory to avoid circular imports) ---
    from routes.auth import auth_bp
    app.register_blueprint(auth_bp, url_prefix='/auth')

    # Main routes (still inline until Phase 3 extracts them)
    _register_inline_routes(app)

    return app


def _configure_oauth(app):
    try:
        with open('secrets/google_oauth.json') as f:
            oauth_secrets = json.load(f)
        app.config['GOOGLE_CLIENT_ID'] = oauth_secrets.get('GOOGLE_CLIENT_ID')
        app.config['GOOGLE_CLIENT_SECRET'] = oauth_secrets.get('GOOGLE_CLIENT_SECRET')
    except FileNotFoundError:
        print("Warning: secrets/google_oauth.json not found. OAuth will not work.")
        app.config['GOOGLE_CLIENT_ID'] = os.environ.get('GOOGLE_CLIENT_ID')
        app.config['GOOGLE_CLIENT_SECRET'] = os.environ.get('GOOGLE_CLIENT_SECRET')

    oauth = OAuth(app)
    google = oauth.register(
        name='google',
        client_id=app.config['GOOGLE_CLIENT_ID'],
        client_secret=app.config['GOOGLE_CLIENT_SECRET'],
        server_metadata_url='https://accounts.google.com/.well-known/openid-configuration',
        client_kwargs={'scope': 'openid email profile ...'}
    )
    app.extensions['oauth'] = oauth
    app.extensions['google_oauth'] = google


def _register_template_helpers(app, cfg):
    from flask_wtf.csrf import generate_csrf

    @app.context_processor
    def inject_project_config():
        project_name = getattr(cfg, 'PROJECT_NAME', 'Group Project AI-ML')
        site_domain = getattr(cfg, 'SITE_DOMAIN', 'localhost:5000')
        base_url = getattr(cfg, 'BASE_URL', f"http://{site_domain}")
        return {'project_name': project_name, 'project_config': cfg,
                'SITE_DOMAIN': site_domain, 'BASE_URL': base_url}

    @app.context_processor
    def inject_csrf_token():
        return dict(csrf_token=lambda: generate_csrf())

    @app.errorhandler(CSRFError)
    def handle_csrf_error(error):
        if request.path.startswith('/api/'):
            return jsonify({'success': False, 'message': 'CSRF token missing or invalid'}), 400
        flash('Security check failed. Please refresh the page and try again.', 'error')
        referer = request.referrer or url_for('auth.signin')
        return redirect(referer)

    @login_manager.unauthorized_handler
    def login_unauthorized():
        try:
            path = request.path or ''
            accepts_json = 'application/json' in (request.headers.get('Accept') or '')
        except Exception:
            path = ''
            accepts_json = False
        if path.startswith('/api/') or accepts_json or request.headers.get('X-Requested-With') == 'XMLHttpRequest':
            return jsonify({'success': False, 'message': 'Unauthorized - please login'}), 401
        flash('Please log in to continue', 'error')
        return redirect(url_for('auth.signin'))


def _register_inline_routes(app):
    """Placeholder: all @app.route decorators stay here until Phase 3 extracts them."""
    # Current app.py routes (lines 335-3514) are pasted here intact.
    # This function is intentionally large in Phase 1 — it shrinks in Phase 3.
    pass


if __name__ == '__main__':
    # DL service thread spawned here (not in create_app) — D-07
    import threading
    from run_dl_service import run_dl_service
    threading.Thread(target=run_dl_service, daemon=True).start()

    app = create_app()
    app.run(debug=True, port=5000)
```

### Git Branch Setup

```bash
# Source: D-09, D-10, D-11, D-12 decisions

git checkout main
git checkout -b dev
git push origin dev

git checkout dev
git checkout -b demo
git push origin demo

git checkout dev
git checkout -b mixed
git push origin mixed

git checkout dev
git checkout -b backend
git push origin backend

git checkout dev
git checkout -b frontend
git push origin frontend
```

### .gitignore additions for .planning/ merge safety

```gitignore
# GSD session-specific planning files (conflict-prone in multi-developer setup)
.planning/STATE.md
```

---

## State of the Art

| Old Approach | Current Approach | Notes |
|--------------|------------------|-------|
| Module-level `app = Flask(...)` | `create_app()` factory | Factory required for testability and multiple configs |
| `Limiter(get_remote_address, app=app, ...)` | `Limiter(get_remote_address)` + `limiter.init_app(app)` | Flask-Limiter 3.x supports init_app pattern |
| All singletons in app.py | `core/extensions.py` singleton module | Standard Flask pattern since ~2012 |
| CSRFProtect(app) inline | `csrf = CSRFProtect()` + `csrf.init_app(app)` | Same deferred init pattern |

**Current state (Phase 1 entry):** app.py is 3514 lines. `routes/` and `services/` directories exist but are empty. `auth_bp` Blueprint is defined inline in app.py lines 262-333 and registered at line 333.

---

## Open Questions

1. **Does `Database()` in `core/database.py` accept `init_app(app)` or does it connect directly on `__init__`?**
   - What we know: `Database()` is instantiated at app.py line 125 with no app argument. It works currently.
   - What's unclear: Whether it has an `init_app` method or simply connects to SQLite on construction.
   - Recommendation: Check `core/database.py` before writing extensions.py. If `Database()` has no `init_app`, just assign to `app.extensions['database']` after calling `Database()` — no init_app call needed. This is the current pattern at app.py line 134.

2. **Does the `auth_bp` Blueprint (inline at app.py line 264) need to stay inline during Phase 1 or can it be moved to `routes/auth.py`?**
   - What we know: Phase 1 scope is factory + extensions + branches. Blueprint extraction is Phase 3.
   - What's unclear: The CONTEXT.md says "all 113 existing routes must keep working" — the auth_bp is already a Blueprint, so keeping it inline is valid for Phase 1.
   - Recommendation: Keep `auth_bp` inline in app.py for Phase 1. Do not move it to `routes/auth.py` until Phase 3. Premature extraction adds scope to Phase 1 without meeting Phase 3 requirements.

3. **What is the exact `auth_bp` URL prefix impact?**
   - What we know: `auth_bp` is registered with `url_prefix='/auth'` at line 333. `login_manager.login_view = 'auth.signin'` is already correctly qualified.
   - What's unclear: Are there any JavaScript `fetch('/signin')` calls that break on the `/auth/` prefix? This was present before Phase 1, so it's a pre-existing issue, not introduced by Phase 1.
   - Recommendation: Document as a known risk. Phase 1 does not change this behavior — the auth_bp registration line 333 is preserved as-is.

---

## Environment Availability

All required tools are confirmed available (existing git repo, Python already running the Flask app).

| Dependency | Required By | Available | Notes |
|------------|------------|-----------|-------|
| Git | Branch setup (FOUND-03, FOUND-04) | Yes | Already in use; main branch active |
| Python 3.x | App factory refactor | Yes | app.py already runs |
| Flask + all extensions | core/extensions.py | Yes | All already installed in current environment |
| `routes/` directory | Future Blueprint files | Yes | Directory exists, only `__pycache__` inside |
| `services/` directory | Future service files | Yes | Directory exists, only `__pycache__` inside |
| `core/` directory | extensions.py, models.py | Yes | `core/__init__.py` exists |

**No missing dependencies.** Phase 1 requires no new package installations.

---

## Validation Architecture

`nyquist_validation` is enabled in `.planning/config.json`.

### Test Framework

No pytest infrastructure exists in the project yet (tests/ and test/ directories are in .gitignore). Phase 1 does not require automated tests — TEST-01 through TEST-03 are Phase 4 requirements. Validation for Phase 1 is manual smoke testing.

| Property | Value |
|----------|-------|
| Framework | Manual smoke test (no pytest until Phase 4) |
| Config file | None — Wave 0 for Phase 4 |
| Quick run command | `python app.py` (startup check) |
| Full suite command | Manual: start app, navigate all main routes |

### Phase Requirements to Test Map

| Req ID | Behavior | Test Type | Automated Command | Notes |
|--------|----------|-----------|-------------------|-------|
| FOUND-01 | `create_app()` function exists and returns Flask app | smoke | `python -c "from app import create_app; app = create_app(); print('OK')"` | Manual after implementation |
| FOUND-02 | `core/extensions.py` exists with db_manager, login_manager, csrf, limiter | smoke | `python -c "from core.extensions import db_manager, login_manager, csrf, limiter; print('OK')"` | Manual after implementation |
| FOUND-02 | App starts without ImportError | smoke | `python app.py` exits cleanly (no traceback on import) | Run immediately after each file change |
| FOUND-03 | 6 branches exist | manual | `git branch -a` | Visual check |
| FOUND-04 | Demo freeze protocol documented | manual | Team agreement + check date | No automated check |

### Sampling Rate

- **After each file created/modified:** Run `python app.py` — startup must succeed
- **After all 4 requirements implemented:** Run full manual smoke test (start app, visit /auth/signin, /dashboard, /admin)
- **Phase gate:** All 4 requirements verified manually before advancing to Phase 2

### Wave 0 Gaps

No test file infrastructure needed for Phase 1. TEST-01 (pytest setup) is Phase 4 scope.

*(No gaps for Phase 1 — validation is startup smoke test + manual navigation, not automated tests)*

---

## Project Constraints (from CLAUDE.md)

These directives apply to all work in this phase:

1. **PRD.md is read-only** — never modify it
2. **TODO.md is the living backlog** — may add items, mark complete, preserve section order
3. **All commits use Conventional Commits format** — `feat`, `fix`, `refactor`, `chore`, etc.
4. **Update docs/ after every significant change** before marking task complete
5. **Run tests before marking any implementation task complete** — for Phase 1 this means `python app.py` smoke test
6. **Never hardcode secrets** — OAuth secrets remain in `secrets/google_oauth.json` or env vars
7. **Consult `docs/technical/DECISIONS.md`** before changes that may conflict with prior architectural decisions
8. **Tech stack locked: Flask, SQLite, Python** — no framework changes
9. **Backwards compatibility required** — all existing routes must keep working after Phase 1
10. **Code style: 4-space indent, snake_case functions/variables, PascalCase classes**
11. **Use project logger** — `from utils.logger import get_logger; logger = get_logger(__name__)` for new code (existing print() in app.py is acceptable to keep during Phase 1)
12. **GSD workflow enforcement** — file changes must go through `/gsd:execute-phase`, not direct edits

---

## Sources

### Primary (HIGH confidence)

- Direct inspection of `app.py` (3514 lines, 2026-03-28) — all line number references verified
- `.planning/research/PITFALLS.md` — pitfalls verified against actual code
- `.planning/research/ARCHITECTURE.md` — component boundaries derived from code audit
- `DOCUMENTS/backend/APP-PY-renew.md` — team's own refactoring proposal
- `.planning/phases/01-foundation-and-branch-safety/01-CONTEXT.md` — locked decisions D-01 through D-16

### Secondary (MEDIUM confidence)

- `.planning/codebase/STRUCTURE.md` — confirms routes/ and services/ exist but are empty
- `.planning/codebase/ARCHITECTURE.md` — confirms current layer structure

### No external sources required

All findings are grounded in direct code inspection. No web searches needed — the patterns (Flask app factory, init_app, extensions module) are well-established Flask idioms that have been stable since Flask 0.8 (circa 2011). The implementation details are driven entirely by what already exists in this specific codebase.

---

## Metadata

**Confidence breakdown:**
- Branch setup (FOUND-03, FOUND-04): HIGH — pure Git operations, no code risk
- Extensions module (FOUND-02): HIGH — init_app pattern verified against current imports in app.py
- App factory (FOUND-01): HIGH — create_app() is standard Flask; all current globals mapped to correct factory locations
- Pitfalls: HIGH — all verified against specific line numbers in app.py

**Research date:** 2026-03-28
**Valid until:** 2026-04-28 (30 days — stable Flask patterns, slow-moving domain)
