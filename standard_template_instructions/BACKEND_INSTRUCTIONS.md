# Backend Structure (FastAPI)

This backend uses FastAPI. Follow this structure when generating code. The project stores data using either local files or a database, but not both. Business logic should be separated from HTTP routes, and configuration should be centralized.

## Stack

- Python
- FastAPI
- pydantic for request/response schemas
- Optional: SQLAlchemy for database access (if this project uses a database)
- Optional: JSON or binary file access using the `data/` directory
- Configuration via pydantic BaseSettings

## Structure

```
backend/
  run.py                  # Entry point that starts the FastAPI app
  app/
    main.py               # FastAPI app setup (routers, middleware)
    config/
      settings.py         # Configuration loaded from environment variables
    routes/               # HTTP route handlers (FastAPI endpoints)
      ...
    services/             # Application logic used by route handlers
      ...
    schemas/              # Pydantic models for request and response data
      ...
    db/                   # Only included if this project uses a database
      database.py         # SQLAlchemy engine and session setup
      models.py           # SQLAlchemy models
      crud.py             # DB read/write functions
  data/                   # Local data files (json, db files, audio, etc.)
  tests/                  # Test suite using pytest and FastAPI TestClient
```

## Routing

- All route files go in `app/routes/`.
- Route handlers should only handle request/response logic.
- Business logic must be delegated to service functions.

## Services

- Service modules go in `app/services/`.
- Each service file handles logic for a specific feature or domain.
- Services may read from/write to local files or a database, depending on the project.

## Schemas

- All request and response models go in `app/schemas/`.
- Use pydantic models to define the structure of API payloads.

## Configuration

- Define a settings class in `app/config/settings.py` using `pydantic.BaseSettings`.
- Configuration values should be read from environment variables.
- Example values: API prefix, data directory, database URL, API tokens.
- All local data file paths must resolve from `settings.DATA_DIR`.
- `settings.DATA_DIR` should be an absolute path, defaulting to `backend/data`.

## Data Access

- This project uses either:
  - Local files resolved from `settings.DATA_DIR`
  - A database with models and CRUD logic in `app/db/`
- Only one approach should be implemented in the project.

## Running the App

- Launch the app using `python backend/run.py`.
- `run.py` starts uvicorn and points to the FastAPI app in `app/main.py`.

## Testing

- All tests go in `backend/tests/`.
- Use `pytest` and FastAPI’s `TestClient`.
- Do not run uvicorn or use server reload in test code.

## Linting and Formatting (Backend)

- Prefer Ruff for linting (and optional formatting) to keep tooling simple.
- If using Black for formatting, configure Ruff for lint-only and let Black handle formatting.
- Example commands: `ruff check .` and either `ruff format .` or `black .` (pick one formatter).

## Rules

- Keep route handlers thin.
- Place all logic in service modules.
- Use pydantic models for request and response validation.
- Use a database only if required by this project.
- Use `settings.DATA_DIR` for all local file storage.
- Place all configuration in `settings.py`.

