# DriftGuard — Project Rules & Architecture Guide

**Project:** DriftGuard — Real-Time Fraud Detection API with Automated Data Drift Recovery

## 1. Target Proof Points
- **Imbalance-Aware Fraud Classification:** XGBoost trained with `scale_pos_weight`, evaluated using **PR-AUC**, **Precision**, **Recall**, and **F1** (never raw accuracy alone).
- **Sub-30ms p95 Inference Latency:** FastAPI keeps the promoted `@champion` model and SHAP `TreeExplainer` pre-loaded in RAM at startup. Always benchmark both with and without SHAP.
- **Self-Healing Drift Recovery Loop:** Evidently AI detects data drift against `data/reference/reference.parquet`, triggers challenger retraining, runs a champion-vs-challenger validation gate, promotes `@champion` in MLflow Model Registry, calls FastAPI `/reload-model`, and measures **Automated Recovery Time**.

## 2. Python Runtime & Environment
- **Recommended Python:** `3.12` or `3.13` (`requires-python >=3.12,<3.14`).
  - *Why:* `evidently 0.7.x` internally uses `pydantic.v1` compatibility layers that break on Python `3.14+`, while `shap >= 0.50` targets modern Python (`3.11+`).
- **Virtual Environment:** Always work inside `.venv` (`python -m venv .venv`).
- **Dependencies:**
  - `requirements.txt`: lean runtime packages (used locally and inside Docker images).
  - `requirements-dev.txt`: inherits `requirements.txt` and adds `pytest`, `pytest-asyncio`, `ruff`, and `pyright[nodejs]`.

## 3. Repository Structure & File Ownership
| Path | Purpose | Owner Agent |
| :--- | :--- | :--- |
| `src/__init__.py`, `src/api/__init__.py`, `src/monitoring/__init__.py` | Package markers (and shared config if added) | Lead Orchestrator |
| `src/data_loader.py`, `src/train.py`, `src/explain.py` | Data prep, XGBoost training, SHAP explainer, MLflow logging | `ml-trainer` |
| `src/api/main.py`, `src/api/schemas.py`, `src/api/db.py` | FastAPI `/predict`, `/health`, `/reload-model`, Pydantic schemas, SQLite/Postgres logging | `api-builder` |
| `src/monitoring/simulator.py`, `src/monitoring/drift_check.py`, `src/monitoring/retrain_flow.py` | Stream simulation, Evidently drift detection, challenger gate, auto-recovery | `drift-ops` |
| `Dockerfile`, `docker-compose.yml`, `.dockerignore` | Container build, FastAPI + MLflow orchestration, health checks | `docker-devops` |
| `tests/`, `benchmarks/load_test.py` | `pytest` suites and latency/recovery benchmarks | `qa-benchmarker` |
| Read-only cross-repo audit | Leakage, blocking I/O, SHAP latency, secrets, readability | `mlops-reviewer` |

## 4. Non-Negotiable Engineering Standards
1. **Verify Library APIs Before Coding:** Always check up-to-date docs (via Context7 or official docs) for `evidently` (`0.7.x` `Dataset`, `DataDefinition`, `Report`, `DataDriftPreset`), `mlflow` (`3.x` `MlflowClient.set_registered_model_alias` taking `"champion"` without `@`, loaded via `models:/<name>@champion`), `fastapi`, `xgboost`, and `shap`.
2. **Zero Data Leakage:** Stratified split (`train` / `val` / `test`) must happen *before* any scaling, resampling, or threshold fitting. Save `data/reference/reference.parquet` strictly from training features.
3. **Non-Blocking FastAPI:** Never perform model loading, SHAP tree construction, or synchronous file/DB calls directly inside an `async def` route without offloading or lifespan initialization.
4. **Strict Typing & Validation:** All modules and Pydantic v2 schemas must have full parameter/return annotations and pass `pyright` (`standard` mode with `reportMissingParameterType: true` in `pyrightconfig.json` to avoid noisy missing stubs on ML libraries while enforcing annotations).
5. **Security & Config Hygiene:** Never hardcode secrets, tokens, or connection credentials. Load configuration via environment variables (`.env` / `pydantic-settings`). Use parameterized SQL queries only.
6. **Beginner-Friendly Mentorship:** Explain every new tool, file, and terminal command in plain English, and work strictly phase-by-phase.
7. **Git Hygiene:** Use Conventional Commits (`feat:`, `fix:`, `chore:`, `test:`, `docs:`) after each verified phase. Never commit `.venv/`, `.env`, `data/`, `models/`, `mlruns/`, or `*.db`.

## 5. Strict Phase Workflow
Before writing code for any phase:
1. Confirm the exact file scope with the `Plan` subagent.
2. Delegate code changes to the owning subagent in `.claude/agents/`.
3. Verify tests/checks with `qa-benchmarker`.
4. Run `mlops-reviewer` (read-only) before closing the phase.
5. Provide exact verification terminal commands and wait for confirmation before starting the next phase.
