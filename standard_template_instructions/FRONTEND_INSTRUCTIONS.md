# Frontend Stack Reference for New Projects

This is the standard frontend architecture used across all projects. It is designed to be clean, minimal, and scalable. All projects follow the same conventions so they are easy to reason about and easy to work with using AI code assistants for generating components, routes, and UI logic.

## Overview

The frontend is built using the modern React framework (Next.js with the App Router). Styling is handled with Tailwind CSS. Data fetching and server state management is handled via React Query (TanStack Query). Forms are built using React Hook Form and validated with Zod. For basic UI components like buttons and inputs, the shadcn/ui generator is used to install local, accessible, Tailwind-styled components.

All code is written in TypeScript. No CSS-in-JS, no external component libraries like MUI or Ant. The stack is intentionally kept small and consistent.

## Core Stack

- Framework: Next.js (App Router)
- Language: TypeScript
- UI Library: React
- Styling: Tailwind CSS
- UI Components: shadcn/ui (locally generated components)
- Data Fetching: TanStack Query (React Query)
- Forms: React Hook Form
- Validation: Zod
- Icons: lucide-react
- Dates: date-fns

## Folder Structure

```
/src
  /app
    layout.tsx              # Shared layout
    globals.css             # Tailwind base styles
    (routes)/...            # Route groups (e.g. /episodes, /lessons)
  /components
    /ui                     # shadcn/ui components (Button, Input, Dialog, etc.)
    [Feature].tsx           # Domain-specific UI components
  /lib
    /api
      client.ts             # Centralized fetch wrapper
    /query
      hooks.ts              # React Query hooks
      keys.ts               # Query keys
    /types                  # Shared TypeScript types (optional)
    /utils                  # Utility functions (e.g. cn, formatDate)
```

## Data Fetching

All HTTP requests go through `lib/api/client.ts`. This is a simple wrapper around `fetch()` that handles headers, errors, and typing.

React Query is used for:
- Caching and deduplication
- Background refetching
- Retry logic
- Built-in loading and error states

Query hooks are defined in `lib/query/hooks.ts`.

Example:

```ts
// lib/query/hooks.ts
export function useEpisodes() {
  return useQuery({
    queryKey: ['episodes'],
    queryFn: () => api.getEpisodes(),
  });
}
```

To refresh data manually:

```ts
const queryClient = useQueryClient();
queryClient.invalidateQueries(['episodes']);
```

## Forms and Validation

Use React Hook Form with Zod schemas for consistent form handling and validation.

```ts
const schema = z.object({
  email: z.string().email(),
});

type FormValues = z.infer<typeof schema>;
```

Form UI elements are composed using shadcn/ui components such as `Input` and `Button`.

## UI and Styling

- Tailwind is used for all styling. No CSS-in-JS, no external UI libraries.
- Reusable UI primitives (e.g. Button, Dialog) are generated with `shadcn-ui` and stored in `components/ui/`.
- Domain-specific UI components live in `components/`.

All component styling should be done with Tailwind utility classes, and optionally wrapped in `cn()` from `lib/utils/cn.ts` if class merging is needed.

## Conventions and Rules

- All API calls go through `lib/api/client.ts`.
- All React Query hooks live in `lib/query/hooks.ts`.
- All fetches use typed return values.
- All UI elements are composed using Tailwind + shadcn/ui.
- No external component libraries (e.g., MUI, Chakra, Ant).
- No Zustand, Redux, or other state libraries unless explicitly needed.
- No Server Actions. All business logic lives in the backend (FastAPI).
- Use `pnpm` for package management across all projects.

## Notes

- Only install shadcn/ui components that are needed per project.
- If a charting library is used, wrap it in a chart adapter layer (`components/charts/`) to isolate dependencies.
- For testing (optional), Playwright can be used for E2E coverage, but it is not required for most projects.

## Summary

This stack is:
- Simple and modern
- Easy for AI agents to reason about
- Consistent across projects
- Lightweight but powerful
- Built to scale cleanly

This structure should be followed in all new frontend apps unless there's a compelling reason to diverge.

## Additional Rules

- Do not use Bootstrap. Use Tailwind CSS and shadcn/ui for all UI.

## API Client Contract

```ts
// lib/api/client.ts
export const API_BASE = process.env.NEXT_PUBLIC_API_BASE!;
type Json = Record<string, unknown> | unknown[];

async function request<T>(path: string, init?: RequestInit): Promise<T> {
  const res = await fetch(`${API_BASE}${path}`, {
    ...init,
    headers: { 'Content-Type': 'application/json', ...(init?.headers || {}) },
  });
  if (!res.ok) {
    const text = await res.text().catch(() => '');
    throw { status: res.status, message: res.statusText, detail: text || undefined };
  }
  return res.json() as Promise<T>;
}

export const api = {
  get: <T>(p: string) => request<T>(p),
  post: <T>(p: string, body: Json) => request<T>(p, { method: 'POST', body: JSON.stringify(body) }),
};
```

- All HTTP calls must go through `lib/api/client.ts`.
- If using React Query, hooks live in `lib/query`. Never fetch directly in components.

## Linting and Formatting (Frontend)

- Use ESLint with Next.js default config.
- Use Prettier for formatting.
- Scripts (example):

```json
{
  "scripts": {
    "lint": "next lint",
    "format": "prettier --write ."
  }
}
```