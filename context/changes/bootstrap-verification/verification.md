---
bootstrapped_at: 2026-05-25T13:15:00Z
starter_id: 10x-astro-starter
starter_name: "10x Astro Starter (Astro + Supabase + Cloudflare)"
project_name: biomass-typer
language_family: js
package_manager: npm
cwd_strategy: git-clone
bootstrapper_confidence: first-class
phase_3_status: ok
audit_command: "npm audit --json"
---

## Hand-off

Verbatim hand-off from `context/foundation/tech-stack.md`:

```yaml
starter_id: 10x-astro-starter
package_manager: npm
project_name: biomass-typer
hints:
  language_family: js
  team_size: solo
  deployment_target: cloudflare-pages
  ci_provider: github-actions
  ci_default_flow: auto-deploy-on-merge
  bootstrapper_confidence: first-class
  path_taken: standard
  quality_override: false
  self_check_answers: null
  has_auth: true
  has_payments: false
  has_realtime: false
  has_ai: true
  has_background_jobs: false
```

### Why this stack

Solo operator (after-hours, ~3-week MVP budget ~40–65h) shipping a multi-tenant B2B SaaS for biomass-recognition with auth, image upload, AI image recognition, and tenant-isolated history. Astro+Supabase+Cloudflare is the recommended default for `(web, js)`: Supabase delivers auth + Postgres + Row-Level Security + storage as a single primitive, directly satisfying the load-bearing FR-002 (tenant isolation from day 1) without custom multi-tenancy plumbing. TypeScript + Zod give the agent explicit contracts on every boundary; the stack clears all four agent-friendly gates (typed, convention-based, popular_in_training, well-documented). AI image recognition runs as a fetch to an external multimodal provider — wall-time only, so Cloudflare Workers' CPU limit is not triggered. Bootstrapper confidence is `first-class` (registered, expected smooth scaffold + occasional manual step). Auth and AI flags on; payments, realtime, background-jobs off per PRD non-goals. CI on GitHub Actions with auto-deploy-on-merge — the starter's standard shape.

## Pre-scaffold verification

| Signal       | Value                                         | Severity | Notes                                                                |
| ------------ | --------------------------------------------- | -------- | -------------------------------------------------------------------- |
| npm package  | not run                                       | n/a      | `cmd_template` starts with `git clone` — npm CLI step skipped per spec |
| GitHub repo  | przeprogramowani/10x-astro-starter pushed 2026-05-17 | fresh    | from `card.docs_url`; 8 days before scaffold                         |

No staleness warning fired. Both `gh` CLI not installed — fell back to `curl https://api.github.com/repos/przeprogramowani/10x-astro-starter` (public endpoint, no auth needed).

## Scaffold log

**Resolved invocation**: `git clone https://github.com/przeprogramowani/10x-astro-starter .bootstrap-scaffold && cd .bootstrap-scaffold && npm install`
**Strategy**: git-clone
**Exit code**: 0
**Files moved**: 18 top-level entries (15 from scaffold + `.gitignore` moved silently + node_modules + package-lock.json)
**Conflicts (.scaffold siblings)**: `CLAUDE.md.scaffold` (cwd `CLAUDE.md` from 10x-toolkit preserved verbatim; starter's CLAUDE.md landed as sibling)
**`.gitignore` handling**: moved silently (cwd had no `.gitignore`)
**`.bootstrap-scaffold/.git/` cleanup**: deleted before move-up (per git-clone strategy — no upstream history leakage)
**`.bootstrap-scaffold/` cleanup**: deleted after move-up

Preserved verbatim (cwd contents existing pre-scaffold):
- `.claude/` — 10x-toolkit skill installations
- `10x-cli/` — local 10x CLI files
- `context/` — bootstrap chain metadata (shape-notes, PRD, tech-stack)
- `CLAUDE.md` — 10x-toolkit instructions (7775B)
- `skills-lock.json` — 10x-cli skill registry

Notable details:
- `npm install` reported `added 772 packages, audited 773 packages in 34s`.
- 2 deprecation warnings emitted (`@babel/plugin-proposal-private-methods`, `node-domexception`) — informational, not bootstrapper concerns.
- `308 packages are looking for funding` — informational.

## Post-scaffold audit

**Tool**: `npm audit --json`
**Summary**: 0 CRITICAL, 1 HIGH, 9 MODERATE, 0 LOW (10 total)
**Direct vs transitive**: 0 direct CRITICAL, 0 direct HIGH, 2 direct MODERATE (of 9 total MODERATE), 0 direct LOW. The single HIGH finding is **transitive**, reached via Cloudflare's Vite plugin chain.

#### CRITICAL findings

None.

#### HIGH findings

- **devalue** (5.6.3 – 5.8.0, transitive)
  - Advisory: [GHSA-77vg-94rm-hx3p](https://github.com/advisories/GHSA-77vg-94rm-hx3p) — "Svelte devalue: DoS via sparse array deserialization"
  - CVSS: 7.5 (AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H)
  - CWE: CWE-770 (Allocation of Resources Without Limits)
  - Reached via: `@cloudflare/vite-plugin` chain
  - Fix available: yes — `npm audit fix` should resolve it within the transitive tree.
  - Impact for this project: low operational concern in MVP — the DoS vector requires hostile input through a devalue deserializer, which isn't on the request path of the biomass-recognition flow. Still worth `npm audit fix` early.

#### MODERATE findings

- **@astrojs/check** (>=0.9.3, **direct**) — via `@astrojs/language-server`. Fix available (semver-major: downgrade to 0.9.2).
- **@astrojs/language-server** (>=2.14.0, transitive) — via `volar-service-yaml`.
- **@cloudflare/vite-plugin** (transitive) — via `miniflare` / `wrangler` / `ws`.
- **miniflare** (transitive) — via `ws`.
- **volar-service-yaml** (<=0.0.70, transitive) — via `yaml-language-server`.
- **wrangler** (**direct**) — via `miniflare`.
- **ws** (8.0.0 – 8.20.0, transitive) — [GHSA-58qx-3vcg-4xpx](https://github.com/advisories/GHSA-58qx-3vcg-4xpx) "Uninitialized memory disclosure", CVSS 4.4.
- **yaml** (>=2.0.0 <2.8.3, transitive) — [GHSA-48c2-rrv3-qjmp](https://github.com/advisories/GHSA-48c2-rrv3-qjmp) "Stack overflow via deeply nested YAML collections", CVSS 4.3.
- **yaml-language-server** (transitive) — via `yaml`.

#### LOW / INFO findings

None.

**Suggested next move on findings**: try `npm audit fix` first (non-breaking auto-fixes only). If `devalue` and `ws` don't resolve via fix, they sit downstream of `@cloudflare/vite-plugin` and `wrangler` — wait for those packages to ship updated versions or use `npm audit fix --force` knowing it may downgrade `@astrojs/check` (semver-major). For an MVP that's still in scaffold state, defer-and-monitor is acceptable.

## Hints recorded but not acted on

| Hint                       | Value                          |
| -------------------------- | ------------------------------ |
| bootstrapper_confidence    | first-class                    |
| quality_override           | false                          |
| path_taken                 | standard                       |
| self_check_answers         | null (standard path)           |
| team_size                  | solo                           |
| deployment_target          | cloudflare-pages               |
| ci_provider                | github-actions                 |
| ci_default_flow            | auto-deploy-on-merge           |
| has_auth                   | true                           |
| has_payments               | false                          |
| has_realtime               | false                          |
| has_ai                     | true                           |
| has_background_jobs        | false                          |

v1 surfaces these but does not compensate. A future M1L4 skill ("Memory Architecture") will read `bootstrapper_confidence`, `quality_override`, `path_taken`, and the `has_*` feature flags to generate ecosystem-specific agent context (`AGENTS.md` / `CLAUDE.md`).

## Next steps

Next: a future skill will set up agent context (CLAUDE.md, AGENTS.md). For now, your project is scaffolded and verified — happy hacking.

Useful manual steps in the meantime:

- `git init` (if you have not already) to start your own repo history. The starter's upstream `.git/` was deleted before move-up.
- Review `CLAUDE.md.scaffold` and decide whether to merge starter-shipped guidance into your existing `CLAUDE.md` (10x-toolkit instructions) or keep them separate.
- Configure Supabase **Row-Level Security** policies before writing any auth-touching code — this is the load-bearing primitive for FR-002 (tenant isolation between heating plants). The starter's gotcha note flagged this; the PRD's guardrail makes it non-negotiable.
- Address audit findings per project risk tolerance — `npm audit fix` is a safe first move; the single HIGH (`devalue`) is transitive and not on the biomass-recognition request path.
- Reproject the runtime envelope: `wrangler.jsonc` is preconfigured for Cloudflare Pages/Workers; `supabase/` directory carries migration scaffolding for the auth + RLS schema.
