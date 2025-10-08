### How to Use This Template

- **Scope**: These are baseline, reusable conventions. Your project-specific instructions override any defaults here.
- **Keep it simple**: Prefer the minimal structure and conventions shown below unless the project doc asks for more.


### Monorepo Layout (Top-Level)

```
/ (repo root)
  backend/              # FastAPI app (see BACKEND_INSTRUCTIONS.md)
  frontend/             # Next.js (App Router) app (see FRONTEND_INSTRUCTIONS.md)
  .env                  # Environment variables (present from start; never commit)
  CLAUDE.md             # Claude-specific workflow guidance
  PROJECT_TEMPLATE.md   # This file
  README.md             # (optional) project-level readme
```

- `frontend/` and `backend/` may be generated or scaffolded per project instructions.
- `data/` is used only if the project chooses local file storage; otherwise, use the DB path from config.

### References (Instruction Files)

- BACKEND_INSTRUCTIONS.md — Backend structure, services, schemas, config
- FRONTEND_INSTRUCTIONS.md — Stack, folders, data fetching, forms, UI
- RUNTIME_AND_TESTING_INSTRUCTIONS.md — Backend runtime and testing policy
- STYLE_GUIDE.md — Frontend style and UI conventions
- WINDOWS_DEVELOPMENT.md — Windows-specific development guidance
- CLAUDE.md — Claude-specific workflow guidance

### Running (Generic, Project-agnostic)

- Backend: `python backend/run.py` (reads config from `backend/app/config/settings.py` and `.env`).
- Frontend: `pnpm dev` inside `frontend/` (Next.js App Router).
- Exact ports, DB/storage choice, and additional flags are defined in the project-specific instructions.

### Configuration & .env

- A `.env` file exists from the beginning. Do not commit it.
- Non-secrets (like host, fixed ports, data dir) are typically defined in `backend/app/config/settings.py` and may be overridden by environment variables if the project requires.
- Secrets (API keys, tokens) live only in `.env` (or your secret manager). Never hardcode secrets.

### CLAUDE.md

- `CLAUDE.md` documents a workflow optimized for Claude. It is not mandatory for other assistants unless your project explicitly says so.
- Follow your project’s instructions if they differ from `CLAUDE.md`.

### Minimal Conventions (Carry Across Projects)

- Backend: thin route handlers in `app/routes/`, business logic in `app/services/`, schemas in `app/schemas/`.
- Frontend: Next.js App Router, Tailwind, shadcn/ui components in `components/ui/`, all HTTP via `lib/api/client.ts`, query hooks in `lib/query/` using React Query.
- Windows: keep console output ASCII-only; avoid background server processes in tests; prefer absolute paths.

### Precedence

1) Project-specific instructions
2) This template (`PROJECT_TEMPLATE.md` and the other instruction files)
3) Tooling defaults

If something is unclear, default to the simplest approach and keep changes minimal.
