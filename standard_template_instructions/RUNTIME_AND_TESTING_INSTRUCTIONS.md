# Backend Runtime & Testing Policy

This document tells the coding assistant exactly how to run the backend and write tests for this project. It standardizes ports, configuration, and test behavior so we avoid port conflicts, reload loops, and stale processes.

## 1) Configuration Model

- **Single source of truth:** `backend/app/config/settings.py` (editable by the assistant).
- **What goes in `settings.py`:**
  - `HOST` (e.g., `127.0.0.1`)
  - `BACKEND_PORT` (fixed project port)
  - `FRONTEND_PORT` (fixed project port for CORS)
  - `DATA_DIR` (absolute path for local files/dbs; defaults to `backend/data`)
- **Secrets (API keys only):** kept in environment variables loaded via `dotenv`. The assistant **must not** read, write, or depend on `.env` content beyond standard library `os.getenv` where applicable. Secrets are not committed.

**Rule:** All non-secret runtime values live in `settings.py`. Only secrets live in the environment.

## 2) Ports

- The backend **must** bind to `HOST` and `BACKEND_PORT` from `settings.py`.
- Never auto-increment or pick alternative ports.
- If the port is busy, **exit with a clear single-line error**. Do not continue.
- CORS should allow the frontend origin derived from `FRONTEND_PORT`.

## 3) Server Start/Stop

- A single launcher script (`backend/run.py`) starts the app.
- The launcher must start the server **without reload**. Do not implement any reload toggling logic.
- Do not spawn multiple servers.
- If the port is busy, stop immediately with an actionable message (do not try another port).
- Console output should be concise and ASCII-only (Windows-safe). No status spam.

## 4) Reload Policy

- Reload is **not used** in this project’s standard launcher. Do not add flags, env switches, or printouts about reload state.
- Developers may run uvicorn manually if needed; the assistant does not automate or assume this.

## 5) Testing Standard

- **Unit/Integration tests (default):**
  - **Never** start uvicorn or bind ports.
  - Use FastAPI’s in-process **TestClient** against the imported `app` object.
  - Tests must be deterministic and fast (no polling or sleeps for “server readiness”).
  - Test data must not touch production data.

- **Test data paths:**
  - Derive all paths from `DATA_DIR` and override to a temporary directory inside tests to keep the filesystem clean.

- **E2E tests (only if explicitly requested):**
  - Start exactly one backend instance, **no reload**, on `BACKEND_PORT`.
  - Before starting, verify the port is free; if not, fail fast (no auto-pick).
  - After tests, stop the process explicitly.
  - Keep E2E coverage minimal (smoke checks).

## 6) File Paths

- All file access (JSON, DB files, media) must resolve from `DATA_DIR` in `settings.py`.
- `DATA_DIR` should be absolute and default to `backend/data`.
- Do not use fragile relative path hops (e.g., `../../..`) within route handlers.

## 7) Logging

- Keep logs short and readable.
- On startup, print a single concise line with host and port.
- On shutdown, print a single concise line.
- On port conflict, print a single-line error with the expected port and how to free it.
- No emojis or Unicode-only symbols in console output.

## 8) Assistant “Must / Must Not”

**Must**
- Import `HOST`, `BACKEND_PORT`, `FRONTEND_PORT`, `DATA_DIR` from `settings.py`.
- Fail fast on port conflicts; do not select a different port.
- Use in-process TestClient for tests; never start servers in tests.
- Use temporary directories for test data; never modify production data.

**Must Not**
- Depend on or modify `.env` beyond standard secrets (API keys).
- Toggle reload or spawn background servers in tests.
- Auto-increment, randomize, or silently change ports.
- Introduce environment-driven behavior for non-secrets.

## 9) Minimal Settings Checklist (constants in `settings.py`)

- `HOST`
- `BACKEND_PORT`
- `FRONTEND_PORT`
- `DATA_DIR`

(Secrets such as API keys are provided via environment variables and are not edited by the assistant.)

## 10) Common Failure Scenarios

- **Port in use:** Abort with a single-line error. Do not pick another port.
- **Hanging tests:** Ensure no server processes are started; use TestClient only.
- **Path errors:** Ensure all paths derive from `DATA_DIR`, not relative hops.
- **Stale processes:** The launcher never uses reload; tests never start servers. Kill any stray process manually before starting.

---
This policy keeps configuration editable in code (`settings.py`), isolates secrets to environment variables, fixes ports per project, disables reload complexity, and ensures tests are fast and reliable without binding ports.
