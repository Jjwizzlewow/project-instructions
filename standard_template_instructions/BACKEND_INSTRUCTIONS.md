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
- **CRITICAL**: Services should have clear, single responsibilities
- **CRITICAL**: Avoid services that just pass data through without adding value
- **CRITICAL**: Keep UI/display concerns separate from business logic

## Service Design Principles

### Single Responsibility Principle
- Each service should have ONE clear purpose
- Services should not mix unrelated concerns
- Example: `options_service` handles option calculations, NOT price display data

### Service Boundaries
- Services should be loosely coupled and highly cohesive
- Avoid services that are just "pass-through" layers
- Each service should own its domain logic completely

### Data Flow Patterns
- **Good**: Direct service-to-frontend communication for display data
- **Bad**: Unnecessary data passing through multiple service layers
- **Rule**: If a service doesn't USE the data, it shouldn't PASS the data

### API Design
- Keep API responses focused on their primary purpose
- Avoid mixing unrelated data in single endpoints
- Consider separate endpoints for different concerns
- Example: `/api/resourceA/{id}` vs `/api/resourceB/{id}`
- Mirror routes → services → schemas by domain. When you add a route file for a domain, ensure corresponding service(s) and schema file exist for that domain.
- Avoid placing models for one domain in another domain’s schema file.

### When to Create New Services
- When adding functionality that doesn't fit existing service boundaries
- When a service becomes too large or handles multiple domains
- When you need to avoid circular dependencies

### Anti-Patterns to Avoid
- Services that are just data pass-through layers
- Mixing UI concerns with business logic
- Services that handle multiple unrelated domains
- Tight coupling between services for unrelated functionality

## Schemas (Pydantic)

- Place models in `app/schemas/` and split by bounded context (mirror your routes/services).
  - Example: `schemas/users.py`, `schemas/billing.py`, `schemas/metrics.py`
- Keep each schema file small and focused (<200 lines). Avoid “everything” files.
- Response models should represent one primary concern. If you add a new concern, create a new model/file in its domain.
- Do not return cross-domain data from a single service by default. Compose at the orchestration layer if multiple domains are truly required.
- Optional shared shapes go in `schemas/common.py` (keep minimal).
- If breaking changes are needed, version models explicitly (e.g., `ResponseV2`).

## Configuration

- Define a settings class in `app/config/settings.py` using `pydantic.BaseSettings`.
- Configuration values should be read from environment variables.
- Example values: API prefix, data directory, database URL, API tokens.
- `settings.DATA_DIR` (only if using file-based storage)
- Always load .env via with load_dotenv().

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

## Architecture Decision Guidelines

### Before Adding Data to a Service Response, Ask:
1. Does this service actually USE this data for its core functionality?
2. Is this data related to the service's primary responsibility?
3. Would this data be better served by a separate endpoint?
4. Am I creating unnecessary coupling between services?

### When Refactoring Services:
1. Identify the core responsibility of each service
2. Move unrelated functionality to appropriate services
3. Create new services if needed rather than bloating existing ones
4. Ensure each service has a clear, single purpose

### API Endpoint Design:
- One endpoint = One primary concern
- Avoid mixing option data with price display data
- Consider frontend needs but don't let them drive service architecture
- Prefer multiple focused endpoints over one bloated endpoint

## Rules

- Keep route handlers thin.
- Place all logic in service modules.
- Use pydantic models for request and response validation.
- Use a database only if required by this project.
- Place all configuration in `settings.py`.
- **NEW**: Each service must have a single, clear responsibility.
- **NEW**: Avoid services that are just data pass-through layers.
- **NEW**: Keep UI/display concerns separate from business logic.
- **NEW**: Consider separate API endpoints for different concerns.

