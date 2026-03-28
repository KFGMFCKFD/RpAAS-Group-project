# Project Guidelines

> **Current Phase**: Final Defense & Q&A Juries.
> **Status**: The project is mostly complete. The codebase is locked for new features; only critical bug fixes are permitted.

## Report & Presentation
- **Main Report**: `DOCUMENTS/reports/THE_FINAL_REPORT.md`. *Already submitted to reviewers. Do not modify.*
- **Presentation Materials**: Located in `DOCUMENTS/reports/presentation/`.
  - `main.tex`: Final presentation slides.
  - `q&a/emergency file traveller.md`: Quick navigation guide for codebase review during Q&A (e.g., "Where is the OCR part?").
  - `q&a/report q&a.md`: Q&A regarding the report content.
  - `q&a/diagram q&a.md`: Q&A about architecture diagrams and component interactions.
  - `THE_FINAL_REPORT.md`: Frozen report copy.

## Q&A Focus
- Be prepared to explain architecture, design choices, Data preprocessing, and how to run the project.
- **Critical Components**: Ensure the Workflow sections run smoothly, especially the **OCR** and **LSTM** nodes. Reviewers will want to test these heavily.
- **Key Concepts**: Algorithms (e.g., Kahn's algorithm for workflow execution), component interaction, and methods from the report.

## Code Style
- **Languages**: Python (primary), HTML/CSS/JS (UI).
- **Formatting**: Python uses 4 spaces indentation. Follow PEP 8 generally (see `app.py` and `core/database.py` for examples). No strict linter.
- **Encoding**: Ensure UTF-8 encoding is used, especially for Windows compatibility.
- **Docstrings**: Required for utility functions and classes.

## Architecture
- **Management System** (`app.py`): Main Flask app handling UI, Auth, and Google Integration. Server-side rendered with Jinja2 templates in `ui/templates/`.
- **AI Agent Service** (`ai_agent_service/`): Autonomous agent logic using 8-bit quantized Qwen models and RAG.
- **DL Service** (`dl_service/`): Separate Flask microservice for Deep Learning tasks (OCR using TensorFlow/Keras, LSTM forecasting).
- **Database**: Dual support for SQLite and PostgreSQL. Configured in `core/config.py` with an abstraction layer in `core/database.py` (`PGShimCursor`).

## Build and Test
- **Run Commands**:
  - Main App: `python app.py`
  - AI Agent: `python ai_agent_service/main.py`
  - DL Service: `python dl_service/model_app.py`
  - Database: `docker-compose up db`
- **Dependencies**: No root `requirements.txt` (common libs: flask, requests, pandas, numpy, torch). AI Agent uses `ai_agent_service/requirements.txt`.
- **Tests**: Standalone scripts in the `test/` directory. Run manually (e.g., `python test/test_ai_service.py`). No `pytest` runner is configured.

## Project Conventions
- **Core Logic**: Shared utilities reside in `core/` (e.g., `core/auth.py`, `core/workflow_engine.py`).
- **Integration**:
  - Google: `core/google_integration.py` (Drive, Sheets, Gmail, Analytics).
  - Make.com: `core/make_integration.py` (`trigger_webhook`).
  - Deep Learning: Main app consumes APIs defined in `dl_service/model_app.py`.
- **Secrets & Security**: Stored in JSON files in `secrets/` (e.g., `google_oauth.json`, `database.json`) or environment variables. Managed by `core/auth.py`. Do not commit secrets.

