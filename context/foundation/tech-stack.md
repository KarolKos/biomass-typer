---
starter_id: 10x-astro-starter
package_manager: npm
project_name: biomass-typer
hints:
  language_family: js
  team_size: solo
  deployment_target: cloudflare-workers # corrected 2026-05-29 (was cloudflare-pages); @astrojs/cloudflare v13 dropped Pages support — see infrastructure.md
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
---

## Why this stack

Solo operator (after-hours, ~3-week MVP budget ~40–65h) shipping a multi-tenant B2B SaaS for biomass-recognition with auth, image upload, AI image recognition, and tenant-isolated history. Astro+Supabase+Cloudflare is the recommended default for `(web, js)`: Supabase delivers auth + Postgres + Row-Level Security + storage as a single primitive, directly satisfying the load-bearing FR-002 (tenant isolation from day 1) without custom multi-tenancy plumbing. TypeScript + Zod give the agent explicit contracts on every boundary; the stack clears all four agent-friendly gates (typed, convention-based, popular_in_training, well-documented). AI image recognition runs as a fetch to an external multimodal provider — wall-time only, so Cloudflare Workers' CPU limit is not triggered. Bootstrapper confidence is `first-class` (registered, expected smooth scaffold + occasional manual step). Auth and AI flags on; payments, realtime, background-jobs off per PRD non-goals. CI on GitHub Actions with auto-deploy-on-merge — the starter's standard shape.
