---
phase: 01-foundation-and-branch-safety
plan: "03"
subsystem: app
tags: [app-factory, create-app, flask-factory-pattern, extensions, FOUND-01]
dependency_graph:
  requires: [core/extensions.py, core/models.py]
  provides: [create_app() factory in app.py]
  affects: [all future blueprints, Phase 2 service extraction, Phase 4 app slimming]
tech_stack:
  added: []
  patterns: [app-factory-pattern, init_app-binding, global-keyword-for-backward-compat]
key_files:
  created: []
  modified:
    - app.py
decisions:
  - "D-01 honored: All extensions (login_manager, csrf, limiter) bound via init_app() inside create_app()"
  - "D-04 honored: OAuth config extracted to _configure_oauth(app) private helper called inside create_app()"
  - "D-05 honored: auth_manager, agent_middleware, automation_engine created inside create_app() and stored in app.extensions"
  - "D-07 honored: DL service thread spawning stays in __main__ block, not inside create_app()"
  - "D-08 honored: background workers already instantiate Database() locally, no change needed"
  - "D-15 honored: Incremental migration — routes stay in app.py, only top-section restructured"
  - "Backward compat: Module-level globals (auth_manager, agent_middleware, automation_engine, google) populated via global keyword so existing routes work without modification"
  - "Talisman called directly inside create_app() (does not support init_app pattern)"
  - "User class removed from app.py, imported from core/models.py"
metrics:
  duration: "~3 minutes"
  completed: "2026-03-28"
  tasks_completed: 1
  files_created: 0
  files_modified: 1
---

# Phase 01 Plan 03: App Factory (create_app) Summary

**One-liner:** Wrapped all Flask initialization in a `create_app()` factory function using init_app() binding for extensions, _configure_oauth() helper for OAuth, and global keyword for backward-compatible module-level route access.

## What Was Built

### create_app() Factory Function

The entire top section of app.py (lines 37-260 in the original) was restructured into a `create_app(config_object=None)` function that:

1. Creates the Flask app instance with template folder
2. Applies ProxyFix middleware
3. Sets all app.config values (SECRET_KEY, CSRF, session, rate limiting)
4. Calls `_configure_oauth(app)` to load Google OAuth credentials
5. Initializes Talisman directly (no init_app support)
6. Binds extensions via init_app: `login_manager.init_app()`, `csrf.init_app()`, `limiter.init_app()`
7. Creates auth_manager, agent_middleware, automation_engine and stores them in `app.extensions`
8. Registers unauthorized_handler and user_loader on login_manager
9. Registers context processors (inject_project_config, inject_csrf_token)
10. Registers error handler for CSRFError
11. Registers auth_bp Blueprint
12. Returns the configured Flask app

### Module-Level Structure

```
[imports including from core.extensions and from core.models]
[module-level globals: auth_manager=None, google=None, etc.]
[helper functions: format_plan_dict, parse_db_datetime, etc.]
[_configure_oauth() private helper]
[create_app() factory function]
[auth_bp Blueprint definition + signin/signup routes]
app = create_app()   <-- replaces old app = Flask(...)
[all @app.route(...) decorators -- unchanged]
if __name__ == '__main__':
    db_manager.init_database()
    app.run(...)
```

### Key Design Decisions

- **Global keyword pattern**: `auth_manager`, `agent_middleware`, `automation_engine`, and `google` are declared as `None` at module level, then populated inside `create_app()` via the `global` keyword. This preserves backward compatibility -- all 113 existing routes continue to reference these as module-level names without any changes.
- **Extension imports**: `login_manager`, `csrf`, `limiter`, `db_manager` are imported from `core/extensions.py` (created in Plan 02). No duplicate instantiation in app.py.
- **User class import**: `User` is imported from `core/models.py` (created in Plan 02). The inline `class User(UserMixin)` definition was removed from app.py.
- **auth_bp stays inline**: The Blueprint definition and its two routes (signin, signup) remain at module level in app.py. They will be extracted in Phase 3.

## Verification Results

| Check | Result |
|-------|--------|
| `import app; hasattr(app, 'create_app')` | PASS -- prints `Factory OK` |
| `import app; type(app.app)` | PASS -- prints `<class 'flask.app.Flask'>` |
| `grep -c "def create_app" app.py` | PASS -- returns `1` |
| `from core.extensions import login_manager, csrf, limiter, db_manager` | PASS |
| `from core.models import User` | PASS |
| No module-level `app = Flask(` | PASS |
| No module-level `CSRFProtect(app)` | PASS |
| `login_manager.init_app` inside create_app body | PASS (line 179) |
| `csrf.init_app` inside create_app body | PASS (line 181) |
| `limiter.init_app` inside create_app body | PASS (line 182) |
| `@login_manager.user_loader` inside create_app body | PASS (line 210) |

## Deviations from Plan

None -- plan executed exactly as written.

## Known Stubs

None. All initialization is fully wired with real configuration values and real service instances.

## Commits

| Task | Description | Commit | Files |
|------|-------------|--------|-------|
| 1 | Restructure app.py into create_app() factory | `f9f0cdf` | app.py |

## Self-Check: PASSED

- `01-03-SUMMARY.md` exists: FOUND
- Commit `f9f0cdf` (Task 1): FOUND
