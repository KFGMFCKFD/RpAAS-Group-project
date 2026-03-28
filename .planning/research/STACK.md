# Technology Stack — Flask Refactoring

**Project:** Group-project-AI-ML (USTH GEN14)
**Milestone:** Refactor monolithic app.py into Blueprints + services architecture
**Researched:** 2026-03-28
**Overall confidence:** HIGH for structure choices (well-established Flask patterns);
                       MEDIUM for exact library versions (pip not executable in this env)

---

## Constraint: No Framework Changes

The project requires Flask, SQLite, and Python throughout. This stack document
addresses only the refactoring tooling layer — what libraries to add and what
structural patterns to adopt. It does not propose replacing anything in the
existing requirements.txt.

---

## Recommended Stack for the Refactor

### Core Structural Pattern: Application Factory + Blueprints

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| Flask | >=3.0,<4.0 (already pinned) | Web framework | Already in use; Blueprint support is first-class |
| Flask Blueprints | built-in | Route modularisation | Standard Flask pattern; zero new dependency; exactly what APP-PY-renew.md proposes |
| Application Factory (`create_app`) | built-in pattern | Slim app.py entry point | Enables test client instantiation without side-effects; required for pytest-flask fixture |

**Why application factory matters for this project:**
The existing `app.py` instantiates the Flask app at module level, which means
importing it in a test pulls in all configuration, OAuth setup, and background
threads. Converting to `create_app()` makes the app constructible with test
config (in-memory DB, no real OAuth, no DL subprocess) without touching
production code paths.

**Confidence:** HIGH — Flask 3.x official docs and the existing APP-PY-renew.md
proposal both prescribe this pattern.

---

### Testing Layer (NEW — not currently present)

| Library | Version | Purpose | Why |
|---------|---------|---------|-----|
| pytest | >=8.0 | Test runner | Industry standard; all existing test files already follow `test_*.py` naming, so pytest can discover them with zero changes to file names |
| pytest-flask | >=1.3 | Flask app fixture + test client | Provides `client` and `app` fixtures that wire into `create_app()`; eliminates boilerplate in every test file |
| pytest-mock | >=3.14 | `mocker` fixture wrapping `unittest.mock` | Cleaner than bare `unittest.mock`; critical for mocking the DB, OAuth, and DL service calls inside service functions |
| coverage.py | >=7.0 | Code coverage measurement | Measures whether the service layer is actually tested; drives the "add tests" deliverable |
| pytest-cov | >=5.0 | pytest plugin for coverage.py | Integrates coverage into `pytest` invocation (`pytest --cov=services`) |

**Why pytest over unittest:** The existing test files already use standalone
`def test_*()` functions, which are valid pytest tests. Adopting pytest adds the
discovery runner, fixture system, and plugin ecosystem with no rewrite of
existing tests.

**Why pytest-flask specifically:** It solves the hardest problem in Flask
testing — constructing an isolated test client from `create_app()` — with a
single `@pytest.fixture`. Without it, every test file reimplements app startup.

**Why NOT Flask's built-in test client directly:** You can use it, but
pytest-flask wraps it in a fixture that handles application context push/pop
automatically, preventing the `RuntimeError: Working outside of application
context` failures that trip up new contributors.

**Confidence:** HIGH for pytest (dominant Python test runner as of 2025);
HIGH for pytest-flask (official Flask docs point to it); MEDIUM for version
numbers (based on training data, not live PyPI query).

---

### Service Layer Isolation: No New Libraries

| Technology | Purpose | Why |
|------------|---------|-----|
| Python stdlib only | Service module structure | Services are plain Python classes/functions. No service-locator or DI framework needed at this scale. |
| `unittest.mock.patch` / `pytest-mock` | Isolate DB calls in service tests | The existing `core.database.Database` class is passed as a dependency or imported directly; `patch` intercepts it without touching production code |

**What NOT to add:**
- Flask-SQLAlchemy: The project already uses raw `sqlite3` via `core/database.py`
  and a separate SQLAlchemy path for PostgreSQL. Adding Flask-SQLAlchemy now would
  require migrating all existing queries — out of scope for a 1–2 week retake.
- Flask-Marshmallow / Pydantic: Input validation is already done inline.
  Adding a schema layer is a Phase 2 concern; it would derail the refactor.
- Celery: Background tasks (`background_ai_task`) currently use Python threads.
  Celery would require a broker (Redis/RabbitMQ) and is out of scope. Keep threads.
- Flask-RESTX / APIFlask: The project is not building a REST API product — it is
  cleaning up internal structure. These frameworks add overhead with no jury value.

---

### Branch Strategy Tooling: Git Only

| Technology | Purpose | Why |
|------------|---------|-----|
| Git (already in use) | 3-branch strategy: `main` / `dev` / `demo` | No additional tools needed. Branch protection via GitHub settings, not a CI tool. |

**Why NOT GitHub Actions CI now:** The PROJECT.md explicitly marks CI/CD as out
of scope for the retake. Adding a pipeline would consume time the team does not
have. Manual `pytest` runs before PR merge are sufficient.

---

## Installation

Only new packages needed (everything else already exists in requirements.txt):

```bash
# Add to requirements.txt under a new [testing] section or dev-requirements.txt
pip install pytest>=8.0 pytest-flask>=1.3 pytest-mock>=3.14 pytest-cov>=5.0
```

These are dev-only dependencies. They do not need to be installed on the demo
server. Keep them in a separate `requirements-dev.txt` to avoid jury confusion
about production dependencies.

---

## Structural Decisions

### Blueprint URL Prefix Convention

Use explicit `url_prefix` in every Blueprint registration inside `create_app()`,
not inside the Blueprint definition itself. This makes it easy to see all routes
at a glance in the factory and change prefixes without touching route files.

```python
# In create_app() — preferred
app.register_blueprint(auth_bp, url_prefix='/auth')
app.register_blueprint(billing_bp, url_prefix='/api/billing')

# Not inside routes/auth.py — avoid
auth_bp = Blueprint('auth', __name__, url_prefix='/auth')
```

**Why:** The APP-PY-renew.md proposal does not fix url_prefix in Blueprint
constructors, and Flask allows either approach. Centralising in the factory is
easier to audit.

### Service Functions vs Service Classes

Use **module-level functions** in service files, not classes with `__init__`.
The `process_admin_withdrawal(user, data)` example in APP-PY-renew.md is the
right model. Classes add no benefit here — the DB connection is managed by
`core.database`, not by a service instance.

```python
# Correct pattern (from APP-PY-renew.md)
def process_admin_withdrawal(user, data):
    ...

# Avoid — unnecessary class wrapper
class WalletService:
    def process_admin_withdrawal(self, user, data):
        ...
```

**Why:** Simpler to test (call the function directly), simpler to import, and
matches what the existing `core/services/dl_client.py` and
`core/services/analytics_service.py` already do.

### DB Access in Services

Services should import `db_manager` from `core.database` directly. Do not pass
the DB connection as a parameter — the existing codebase does not do this, and
changing the calling convention everywhere is a refactoring risk.

```python
# services/wallet_service.py
from core.database import db_manager

def get_user_balance(user_id: int) -> float:
    conn = db_manager.get_connection()
    ...
```

For testing: use `pytest-mock`'s `mocker.patch('core.database.db_manager.get_connection')`
to return a mock connection. This is less pure than dependency injection but
requires zero changes to service signatures.

---

## Test File Layout

```
tests/                         # NEW top-level directory (separate from test/)
    conftest.py                # create_app fixture, test DB setup
    test_wallet_service.py     # Unit tests for services/wallet_service.py
    test_subscription_service.py
    test_ai_worker_service.py
    test_workflow_service.py
    test_dl_proxy_service.py
    test_auth_routes.py        # Route-level tests (via test client)
    test_billing_routes.py
```

Keep the existing `test/` directory intact — those scripts test live integrations
and connectivity. The new `tests/` directory (plural) holds pytest unit tests for
the service layer. This avoids confusion and merge conflicts between team members
working on different areas.

**conftest.py skeleton:**

```python
import pytest
from app import create_app  # after app.py is refactored

@pytest.fixture
def app():
    app = create_app({
        'TESTING': True,
        'DATABASE_PATH': ':memory:',   # SQLite in-memory
        'WTF_CSRF_ENABLED': False,     # disable CSRF in tests
        'LOGIN_DISABLED': True,        # disable Flask-Login for unit tests
    })
    yield app

@pytest.fixture
def client(app):
    return app.test_client()
```

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Test runner | pytest | unittest | pytest is a strict superset; existing test files work unchanged |
| App isolation | pytest-flask + create_app | flask test_client() directly | pytest-flask manages app context lifecycle automatically |
| Mocking | pytest-mock | unittest.mock directly | pytest-mock is a thin wrapper — lower risk, same power |
| Service design | Module functions | Flask-Injector (DI) | DI framework adds complexity with no jury-facing benefit |
| Blueprint URL prefix | Centralised in factory | Embedded in Blueprint constructor | Factory approach is easier to audit and modify |

---

## Sources

- Existing `package/requirements.txt`: Flask>=3.0,<4.0 confirmed (HIGH confidence)
- `DOCUMENTS/backend/APP-PY-renew.md`: Refactoring proposal, service function examples (HIGH confidence — project-internal)
- `.planning/codebase/STACK.md`: Existing stack analysis (HIGH confidence — codebase analysis)
- `.planning/codebase/TESTING.md`: Existing test patterns (HIGH confidence — codebase analysis)
- Flask documentation (Blueprints, Application Factory, Testing) — pattern descriptions based on Flask 3.x docs knowledge (MEDIUM confidence — official docs blocked by Cloudflare in this session; patterns are stable and well-established)
- pytest-flask PyPI/GitHub: `app` and `client` fixtures, conftest.py patterns (MEDIUM confidence — training data, not live verified in this session)
