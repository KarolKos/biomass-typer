---
plan: first-production-deploy
project: biomass-typer
created: 2026-05-29
status: live — E2E verified (signup→dashboard); email confirmation disabled (MVP)
platform: Cloudflare Workers
source: context/foundation/infrastructure.md
---

# Deploy plan — pierwsze wdrożenie produkcyjne (biomass-typer → Cloudflare Workers)

Zatwierdzony plan (Plan Mode, 2026-05-29). Ścieżka audytu „co miało się wydarzyć" dla pierwszego go-live; źródło decyzji: `context/foundation/infrastructure.md`. Planowanie kamieni milowych może traktować ten plik jako prawdę o tym, „co już jest wdrożone".

## Wynik wykonania (2026-05-31)

Worker był **już wdrożony 2026-05-28** (przez użytkownika, 4 deploymenty). Stan po weryfikacji z tej sesji:

- **Live:** `https://biomass-typer.kosy95.workers.dev` (konto kosy95@gmail.com, subdomena `kosy95`). `/` + `/auth/signin` → 200, `/dashboard` → 302 (middleware działa).
- **Sekrety:** `SUPABASE_URL` + `SUPABASE_KEY` ustawione (publishable/anon). ⚠️ **Pitfall:** `$val | wrangler secret put` w PowerShell dokleił trailing CRLF → Supabase „Invalid API key". Naprawione przez **API Cloudflare** (`PUT …/scripts/biomass-typer/secrets`, dokładny JSON, bez newline). **Na przyszłość: nie pipe'ować sekretów do wrangler w PowerShell — użyć API albo trybu interaktywnego.**
- **Ryzyko #2 (`@supabase/ssr` stream) — NIE wystąpiło:** bogus-signin → 302 `error=Invalid login credentials` (klient Supabase działa w runtime Workera). ✅
- **Bindingi:** ASSETS, IMAGES, SESSION (KV) — auto-bindingi adaptera v13 obecne, deploy ich nie wywrócił.

**Wynik E2E (2026-05-31, potwierdzony przez użytkownika):** ✅ signup → signin → `/dashboard` działa. Wybrano szybki wariant MVP: **email confirmation WYŁĄCZONE** w Supabase (zamiast konfiguracji Site/Redirect URL).

**Pozostałe housekeeping:**
1. ⚠️ Przed realnymi użytkownikami: **włącz z powrotem email confirmation** + ustaw Supabase Site URL/Redirect URLs na origin prod (teraz każdy może się zarejestrować dowolnym, niezweryfikowanym mailem).
2. Rotacja tokenu Cloudflare (był w czacie).

## Co wdrażamy

Obecny **szkielet auth** ze startera — strony `/`, `/auth/{signin,signup,confirm-email}`, `/dashboard`; endpointy `POST /api/auth/{signin,signup,signout}`; middleware chroniący `/dashboard` przez `supabase.auth.getUser()`. **Brak** funkcji biomasy (upload / analiza AI / historia), brak migracji i RLS (tylko wbudowane `auth.users`). Wdrażamy fundament; funkcje dobudujemy przeciw żywemu celowi.

## Status wykonania

- ✅ **Faza 2 — build zweryfikowany**: `npm run build` przeszedł czysto (`output: server`, adapter `@astrojs/cloudflare`, ~12.8s). Kod gotowy do deployu **bez zmian** — `src/lib/supabase.ts` używa v13-poprawnego `astro:env/server` (ryzyko #1 już zmitygowane).
- ✅ **Faza 7 (część agenta)**: ten artefakt zapisany; korekta `infrastructure.md` (krok 4/5 + wiersze ryzyka) zastosowana.
- ⏳ **Fazy 0–1, 3–6 — bramki ludzkie**: login, sekrety, go-live, dashboard Supabase, smoke test. Do wykonania przez operatora.

## Nowe ustalenia z buildu (nie było w eksploracji statycznej)

1. **Adapter v13 auto-włącza bindingi `IMAGES` (Cloudflare Images) i `SESSION` (KV)** — niezadeklarowane w `wrangler.jsonc`. Prawdopodobnie nieszkodliwe (auth = cookies Supabase, nie `Astro.session`; brak `<Image>`/processingu obrazów). **Watch-item smoke testu**: jeśli `wrangler tail` pokaże błąd brakującego bindingu przy żądaniu → `npx wrangler kv namespace create SESSION` + dodaj binding, lub wyłącz sesje/`imageService` w opcjach `cloudflare()`. Nie blokuje deployu (bindingi są lazy/runtime).
2. **Sitemap pominięty** — `@astrojs/sitemap` wymaga opcji `site` w `astro.config.mjs`. Benign; jeśli chcesz sitemap, ustaw `site` na produkcyjny URL.

## Runbook (kolejność; bramki ludzkie oznaczone)

**Faza 0 — Cloudflare auth/konto** `[CZŁOWIEK]`
`npx wrangler login` → `npx wrangler whoami` (potwierdź właściwe konto przed utworzeniem czegokolwiek). Token zakresowany (Workers Scripts:Edit) + `CLOUDFLARE_ACCOUNT_ID` to ścieżka pod CI na później.

**Faza 1 — Sekrety** `[CZŁOWIEK, przed deployem]`
```
npx wrangler secret put SUPABASE_URL
npx wrangler secret put SUPABASE_KEY
```
Wartości z lokalnego `.env` (nieśledzony). Nazwy muszą **dokładnie** pasować do schematu `astro:env`. Klucz to **publishable/anon**, NIE service-role. NIE wrzucać do `vars` w `wrangler.jsonc` (bug withastro/astro#16790). Sekret musi istnieć **przed** deployem serwującym daną wersję (`optional: true` nie rzuca na imporcie → brak sekretu = 500 w runtime, nie błąd buildu).

**Faza 2 — Build** `[CLI]` — ✅ WYKONANE
`npm run build` → `./dist`. Build nie potrzebuje sekretów.

**Faza 3 — Pierwszy deploy → go-live** `[CZŁOWIEK — APROBATA]`
`npx wrangler deploy` — tworzy Workera `biomass-typer`, wgrywa `./dist` jako binding `ASSETS`, serwuje na `https://biomass-typer.<subdomena>.workers.dev`. Skopiuj dokładny origin z outputu (potrzebny w Fazie 5). `wrangler deploy` **nie buduje** sam.
> 🔴 `wrangler versions upload` **NIE działa** na Workerze, który nigdy nie był wdrożony — pierwszy push musi być `deploy`. Flow preview→promote dopiero od deployu #2.

**Faza 4 — Włącz/potwierdź URL workers.dev** `[CZŁOWIEK jeśli 404]`
Jeśli URL zwraca 404: dashboard → Worker → Settings → Domains & Routes → włącz **workers.dev** (i Preview URLs).

**Faza 5 — Konfiguracja Supabase Auth pod produkcję** `[CZŁOWIEK — dashboard, APROBATA]`
Dashboard Supabase → Authentication → URL Configuration:
- **Site URL** = `https://biomass-typer.<subdomena>.workers.dev` (dokładny scheme+host).
- **Redirect URLs** = dodaj `…/dashboard` i `…/auth/confirm-email` (lub `…/**`); **zostaw** `http://localhost:4321/**` dla dev.

Bez tego: linki w mailach potwierdzających wskazują `http://localhost:4321/…` → użytkownik produkcyjny nie potwierdzi konta → signin nie działa. Zmiana Site URL działa na cały projekt (bramka aprobaty).

**Faza 6 — Smoke test** `[BRAMKA, nie formalność]`
Z `npx wrangler tail biomass-typer`:
1. Renderują się `/`, `/auth/signin`, `/auth/signup`.
2. `POST /api/auth/signup` → mail przychodzi, link wskazuje origin workers.dev (nie localhost).
3. Po potwierdzeniu + `POST /api/auth/signin` → `/dashboard` ładuje się, middleware `getUser()` bez 500.

**500 na `/dashboard` = ryzyko #2 (`nodejs_compat`)**: bug `@supabase/ssr` „Dynamic require of 'stream'" (supabase/supabase#37592). Może nie wystąpić (bundling Vite Astro różni się od repro; compat_date ≥ 2024-09-23 → `nodejs_compat_v2`). Triage: potwierdź stack przez `wrangler tail`; fix po stronie bundlera (external dla SSR) lub migracja na `@supabase/server`. Sprawdź też brak bindingu SESSION/IMAGES (ustalenie z buildu #1).

**Faza 7 — Utrwal ścieżkę audytu** `[agent]` — ✅ część wykonana
Ten plik + korekta `infrastructure.md`. Po smoke teście: dopisz tu rzeczywisty produkcyjny URL i wynik testu.

## Kolejne wdrożenia (deploy #2+)
`npm run build` → `npx wrangler versions upload` (nieserwujący URL podglądu) → test → `npx wrangler versions deploy` (promocja). Rollback: `npx wrangler rollback` (tylko kod Workera).

## Powiązanie z rejestrem ryzyka (infrastructure.md)
- **#1** (wzorzec env v13) — zmitygowane; kod używa `astro:env/server`. ✅
- **#2** (luki `nodejs_compat`) — konkretnie ryzyko `@supabase/ssr` stream; Faza 6 to bramka.
- **#3** (upload przez Workera), **#4** (split-brain rollbacku RLS) — N/D dla tego deployu (brak uploadu/AI/RLS); wracają z funkcjami biomasy.
- **Dodane do rejestru**: Site/Redirect URL Supabase Auth; auto-bindingi IMAGES/SESSION.

## Weryfikacja (end-to-end)
- `npx wrangler deployments list` pokazuje aktywną wersję.
- Produkcyjny URL serwuje `/` i strony auth.
- Pełny przepływ: signup → mail (link → prod) → potwierdzenie → signin → `/dashboard` bez 500.
- `wrangler tail` czysty w trakcie przepływu.

## Granica człowiek vs agent (oś „zatwierdzanie")
- **Tylko człowiek**: wybór konta Cloudflare, `wrangler login`, wartości sekretów, aprobata `wrangler deploy`, zmiany URL w dashboardzie Supabase.
- **Agent bez nadzoru**: `npm run build`, `wrangler tail`, `deployments list`, zapis/commit artefaktu.

## Poza zakresem
Auto-deploy w CI (obecny `ci.yml` tylko lint+build), własna domena, funkcje biomasy, migracje/RLS (nie istnieją), Cloudflare Access na URL-ach podglądu.
