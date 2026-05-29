# Repository Guidelines

biomass-typer is a multi-tenant B2B SaaS for biomass recognition (PRD at `@context/foundation/prd.md`). Stack: Astro 6 SSR + React 19 + Tailwind 4 + Supabase + Cloudflare Workers.

## Hard rules

- **Tenant isolation = Supabase RLS.** Enable RLS with granular per-operation, per-role policies on every new table before any auth-touching code references it. Load-bearing for FR-002; bypassing it leaks data between tenants.
- **API routes export `const prerender = false`.** Required by `output: "server"` + `@astrojs/cloudflare`. Omitting it fails at runtime, not build time.
- **Never write to `context/archive/`** — entries are immutable. Open a new change folder under `context/changes/` instead.
- **`.env` is gitignored** — holds `SUPABASE_URL` + `SUPABASE_KEY`. Use `.dev.vars` for local Cloudflare workerd dev.
- **`10x-cli/` is a separate tool checkout** — gitignored, not project code.

## Commands

- `npm run dev` — dev server on the Cloudflare workerd runtime
- `npm run build` — production build
- `npm run lint` / `lint:fix` — ESLint, strict type-checked
- `npm run format` — Prettier (Astro + Tailwind plugins)
- `npx supabase start` — local Supabase stack (needs Docker)
- `npx wrangler deploy` — deploy to Cloudflare (alt: push to `main` auto-deploys)

Pre-commit (husky + lint-staged): `eslint --fix` on `*.{ts,tsx,astro}`, `prettier --write` on `*.{json,css,md}`.

## Project structure

- `src/pages/` — Astro routes; `src/pages/api/auth/` — auth endpoints
- `src/components/{auth,ui}/` — React auth forms and shadcn/ui primitives
- `src/lib/supabase.ts` — SSR client (cookie sessions via `@supabase/ssr`)
- `src/middleware.ts` — add protected paths to the `PROTECTED_ROUTES` array
- `supabase/migrations/` — schema and RLS policies
- `context/foundation/` — PRD, shape-notes, tech-stack (build contracts)
- See `@CLAUDE.md.scaffold` for fuller architecture notes from the starter.

## Conventions

- Path alias `@/*` → `./src/*` (see `@tsconfig.json`).
- Astro for static/layout; React only for interactivity. No `"use client"`.
- Tailwind class merging via `cn()` from `@/lib/utils` — never concatenate class strings.
- shadcn/ui: `npx shadcn@latest add <name>` writes to `src/components/ui/`, "new-york" variant.
- API endpoints: uppercase `GET`/`POST` exports, validate input with zod.
- Supabase migrations named `YYYYMMDDHHmmss_short_description.sql`.
- Shared types in `src/types.ts`; business logic in `src/lib/services/`; React hooks in `src/components/hooks/`.

## Testing

No test framework configured yet. Pick one (vitest fits this stack) before merging the first behavioral change to `src/`.

## Commits and PRs

Conventional Commits (`chore:`, `feat:`, `fix:`). CI (`.github/workflows/ci.yml`) runs `astro sync` + lint + build on push/PR to `main`; build needs `SUPABASE_URL` and `SUPABASE_KEY` as GitHub repo secrets. Remote: `KarolKos/biomass-typer`.
