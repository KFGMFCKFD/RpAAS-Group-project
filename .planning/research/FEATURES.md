# Feature Landscape

**Domain:** Flask AI/ML web application — university group project retake (USTH GEN14)
**Researched:** 2026-03-28
**Confidence:** HIGH (derived from first-hand jury feedback, existing codebase, and APP-PY-renew.md proposal)

---

## Context: What the Jury Cares About

This is not a product launch. It is an academic defense before a jury that already failed the first
attempt. The jury's explicit complaints are documented in RETAKENOTES.md:

- They could not understand the architecture (wrong system architecture figure)
- They did not know how the UI was built or what methodology was used
- They did not know how the execution engine works (no description of execution engine)
- The evaluation methodology was wrong (synthetic images, wrong LSTM figure)
- Informal language and missing sections reduced confidence in team's rigour
- Report and presentation were mismatched

The jury is NOT primarily evaluating whether features work. They are evaluating:
1. Whether the team can explain what they built
2. Whether the architecture is defensible and demonstrates engineering judgement
3. Whether the code structure reflects sound software engineering (SoC, testability)
4. Whether the report accurately describes the actual system

Feature decisions should be made with this jury lens in mind.

---

## Table Stakes

Features the jury will mark down if missing or broken. Non-negotiable for the retake.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Flask Blueprint routing | Jury expects "proper" Flask — a 3514-line app.py is a red flag if shown | Medium | APP-PY-renew.md already specifies the 5 blueprints: auth, billing, workflows, ai, dl_proxy |
| App factory pattern (`create_app()`) | Standard Flask pattern; slimming app.py to <100 lines demonstrates the team knows Flask conventions | Low–Medium | Must move middleware/config out of global scope |
| Service layer with no Flask imports | Separation of concerns is the central claim of the refactor; jury will ask to see services/ | Medium | wallet_service, subscription_service, ai_worker_service, workflow_service, dl_proxy_service |
| All existing features still working | Backwards compatibility is a hard constraint — breaking anything during demo is catastrophic | Medium | Auth, wallet, subscription, AI chat, workflows, OCR proxy must all remain functional |
| Correct system architecture diagram | First attempt had wrong figure; the "Brain-Body Split" (3-service architecture) must be accurately shown | Low | This is documentation work, not code — but the jury directly flagged it |
| Description of execution engine | Jury explicitly said "no description of engine" — Kahn's algorithm in workflow_engine.py must be explained | Low | Documented in Q&A sheet; needs to appear in report too |
| Evaluation section redone | Jury called the evaluation methodology wrong; synthetic images must be replaced with real results | Medium | LSTM evaluation, OCR accuracy metrics, confusion matrices all need re-evaluation |
| Unit tests for service layer | The refactor's main stated benefit is testability — having zero tests undermines the claim | Medium | At minimum: wallet_service, subscription_service with pytest |
| Module-level .md documentation | Report requires documentation of HOW app.py was split; each service needs explanation | Low | anser.md says: "write report about how you split app.py" |
| 3-branch git strategy (main/dev/demo) | Protects jury demo from mid-flight merges by 4+ team members | Low | Branch protection on demo branch is critical |

---

## Differentiators

Features not strictly required but that meaningfully impress the jury or reduce risk.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Pytest fixture demonstrating service testability | Showing `def test_process_admin_withdrawal()` running proves the SoC argument — not just claims | Low | Even 3–5 passing tests is compelling evidence; use APP-PY-renew.md's example as the basis |
| Inline docstrings on all service functions | Jury Q&A often triggers "show me that function" — a function with a docstring looks professional vs bare logic | Low | One sentence per function is enough; shows the team treats code as a communication artifact |
| Error handling with explicit HTTP status codes | Routes returning `400`/`403`/`404` with JSON error bodies demonstrates understanding of REST conventions | Low | The APP-PY-renew.md example already shows this pattern; extend consistently |
| Config externalized to `core/config.py` with env var support | Jury may ask "how do you deploy this?" — hardcoded values are a red flag; env vars show production awareness | Low | core/config.py already exists; ensure all secrets/ports/URLs are loaded from there |
| Logging instead of print statements | Production code quality signal; trivial to add but shows the team knows the difference | Low | Replace print() in routes/services with Python's logging module |
| Demo branch with frozen, known-working state | A dedicated demo branch that team has verified working prevents the worst-case scenario: broken demo mid-presentation | Low | Strategy already in PROJECT.md; execute it early and never fast-forward without testing |
| Architecture diagram that matches code | Re-draw the system diagram to reflect actual 3-service split (Body/Brain/DL Service) and reference specific files | Low | High-impact: directly addresses jury complaint #1; requires no coding |

---

## Anti-Features

Features to explicitly NOT build given the 1–2 week timeline. Building these would consume time that
must go to the table stakes above.

| Anti-Feature | Why Avoid | What to Do Instead |
|--------------|-----------|-------------------|
| New UI features or redesign | Jury does not score UI appearance — they score architectural clarity; any new UI work is wasted time | Freeze UI; only fix broken display issues that would embarrass during live demo |
| WebSocket / real-time chat | Out of scope per PROJECT.md; fire-and-poll works, jury accepted the explanation in Q&A docs | Keep existing polling; document why it was the right choice (stateless, proxy-friendly) |
| Database migration (SQLite → PostgreSQL) | Out of scope per PROJECT.md; SQLite is sufficient for demo scale | Leave SQLite; the PGShimCursor already provides migration path — mention this in report |
| CI/CD pipeline or Docker Compose changes | Out of scope per PROJECT.md; manual deployment is fine for retake | Delete or archive docker-compose.yml to reduce confusion (it is already deleted in git status) |
| Mobile responsiveness improvements | Not relevant to jury criteria | |
| New ML models or retraining | The evaluation section needs redone results from existing models, not new models | Re-run existing LSTM and OCR evaluations on proper test data |
| Admin dashboard expansion | Feature set is already complete — extending it risks introducing bugs before the demo | |
| Pagination, search, or filtering on existing lists | Nice-to-have product features with zero jury impact | |
| Async background job queue (Celery, RQ) | Would replace the existing threading approach — far too risky to swap infrastructure in 1–2 weeks | Document the existing background threading pattern; explain its tradeoffs honestly in Q&A |
| API versioning (`/api/v1/`) | Jury will not ask about this; adds complexity for no benefit | |
| Full test coverage (80%+) | Unrealistic in the timeline; targeted tests on service layer are sufficient to make the argument | 5–10 tests demonstrating the pattern beats 0 tests by a wide margin |

---

## Feature Dependencies

The following ordering matters. Building out of order creates rework.

```
1. App factory + config loading (app.py cleanup)
   └─► Required before Blueprints can be registered

2. Service layer files (services/*.py) created empty
   └─► Required before business logic can be moved in

3. Business logic extracted from app.py into services/
   └─► Required before unit tests can be written
   └─► Required before Blueprint routes can call services

4. Blueprint routes (routes/*.py)
   └─► Required before app factory can register_blueprint()
   └─► Imports from services/ so service layer must exist first

5. Unit tests (tests/test_*.py)
   └─► Required after service layer is populated

6. Module documentation (.md files per service)
   └─► Can be written in parallel with any code step
   └─► anser.md content requires the refactor to be done first

7. Evaluation section redone (report)
   └─► Independent of code — can run in parallel from day 1
   └─► Requires running existing models, not new code

8. Demo branch freeze
   └─► Must be the last step before presentation day
```

---

## MVP Recommendation

Given 1–2 weeks and a 4-person team, prioritize strictly:

**Week 1 — Code refactor (2–3 people):**
1. Create `routes/` directory with 5 Blueprint files (thin controllers)
2. Create `services/` directory with 5 service files (business logic extracted from app.py)
3. Rewrite app.py as `create_app()` factory, <100 lines
4. Verify all features still work end-to-end on `dev` branch
5. Write 5–10 pytest tests for service functions

**Week 1 — Report/presentation (1–2 people in parallel):**
1. Fix system architecture diagram (Brain-Body-DL split, accurate file references)
2. Add execution engine description (Kahn's algorithm section)
3. Redo LSTM evaluation with real data
4. Fix informal language and missing captions
5. Align presentation slides with report content

**Week 2 — Polish and freeze:**
1. Write module .md files documenting each service
2. Add docstrings to all service functions
3. Freeze `demo` branch from a verified-working `dev` state
4. Run a dry-run presentation and fill remaining Q&A gaps

**Defer permanently:**
- Any new user-facing feature
- Any infrastructure change (Docker, CI/CD, database migration)
- Test coverage beyond the service layer

---

## Sources

- `RETAKENOTES.md` — Jury feedback from first attempt (primary driver of all table stakes)
- `.planning/PROJECT.md` — Active requirements, constraints, out-of-scope list
- `DOCUMENTS/backend/APP-PY-renew.md` — Refactoring proposal with code examples
- `DOCUMENTS/reports/presentation/q&a/fucntions q&a.md` — Codebase reference sheet for defense
- `DOCUMENTS/reports/presentation/q&a/diagram q&a.md` — Architecture diagram Q&A for defense
- `DOCUMENTS/reports/presentation/THE_FINAL_REPORT.md` — Existing report structure
- Source code inspection: `core/`, `dl_service/`, `ai_agent_service/` directories
- Confidence: HIGH — all findings grounded in primary project documents and codebase, not web research
