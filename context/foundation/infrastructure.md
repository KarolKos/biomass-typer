---
project: biomass-typer
researched_at: 2026-05-29
recommended_platform: Cloudflare Workers
runner_up: Render
context_type: mvp
tech_stack:
  language: TypeScript
  framework: Astro 6 (SSR) + React 19
  runtime: Cloudflare Workers (workerd)
---

## Rekomendacja

**Wdróż na Cloudflare Workers.**

Stack jest już wpięty w Cloudflare (`@astrojs/cloudflare` v13 + `wrangler.jsonc` w trybie Workers), a platforma zdobyła komplet 5×Pass na pięciu kryteriach przyjaznych agentom — w tym najbogatszy ekosystem MCP i najlepszą dokumentację dla agenta (`llms.txt`). Kluczowe dla tego produktu: **~30-sekundowy synchroniczny `fetch()` do zewnętrznego providera AI** (nośnik wymagania `<30s p95`) działa **na darmowym planie** — Workers nie limituje czasu trwania żądania HTTP, dopóki klient jest podłączony, a limit CPU nie liczy czasu czekania na `fetch()`. Wywiad (koszt vs DX neutralnie, jeden region OK, Supabase zewnętrzny OK, brak trwałych połączeń) nie faworyzował żadnego konkurenta na tyle, by przebić te przewagi. Koszt: $0–5/mo.

> **Korekta kontraktu — Workers, nie Pages.** Hint `deployment_target: cloudflare-pages` w `tech-stack.md` jest **nieaktualny**. `@astrojs/cloudflare` v13 wycofał wsparcie dla Cloudflare Pages („migrate to Cloudflare Workers"), a `wrangler.jsonc` projektu jest już skonfigurowany pod Workers (Static Assets). Komenda wdrożeniowa to **`wrangler deploy`**, NIE `wrangler pages deploy` — te dwie ścieżki nie są zamienne. Jeśli `tech-stack.md` ma pozostać spójny, warto poprawić tam hint na `cloudflare-workers`.

## Porównanie platform

Ocena Pass / Partial / Fail na pięciu kryteriach przyjaznych agentom, po nałożeniu twardych ograniczeń stacku (Astro 6 SSR, ~30s synchroniczny fetch do AI, Supabase zewnętrzny) i miękkich wag z wywiadu.

| Platforma | CLI-first | Managed / serverless | Dokumentacja dla agenta | Stabilne API deploy | MCP / integracja | Wynik |
|---|---|---|---|---|---|---|
| **Cloudflare Workers** | Pass | Pass | Pass | Pass | Pass | **5 Pass** |
| **Render** | Pass | Pass | Pass | Pass | Pass | **5 Pass** |
| **Vercel** | Pass | Pass | Pass | Pass | Partial | **4P / 1 Partial** |
| **Netlify** | Partial | Pass | Pass | Pass | Pass | **4P / 1 Partial** |
| **Railway** | Pass | Pass | Partial | Pass | Partial | **3P / 2 Partial** |
| **Fly.io** | Partial | Partial | Partial | Pass | Partial | **1P / 4 Partial** |

**Noty per platforma (uzasadnienie ocen):**

- **Cloudflare Workers** — `wrangler deploy` / `wrangler rollback` / `wrangler tail` to pełna pętla operacyjna z CLI (CLI-first: Pass). Workers to czysty serverless, zero infrastruktury do patchowania (Managed: Pass). Dokumentacja jako `llms.txt` + `llms-full.txt` + markdown przez nagłówek `Accept` (Docs: Pass, best-in-class). Wdrożenie deterministyczne z wersjonowanym rollbackiem do 100 wersji (Deploy API: Pass). ~16 zarządzanych serwerów MCP + API MCP + plugin do Claude Code (MCP: Pass). ~30s fetch działa na Free; D1/R2/KV/Queues dostępne, ale nieużywane (Supabase zewnętrzny).
- **Render** — pełnowartościowe `render` CLI w GA (v2.19): `deploys`, `logs --tail`, `rollback`, `skills` dla agentów (CLI: Pass). Zarządzany Web Service na natywnym Node, bez infrastruktury (Managed: Pass). `llms.txt` + `llms-full.txt` + docs MCP (Docs: Pass). Deploy deterministyczny przez CLI/API/git (Deploy API: Pass). **MCP Server w GA** (sie 2025), 20+ narzędzi, wsparcie Claude Code (MCP: Pass). Limit żądania 100 min — 30s fetch trywialnie się mieści. Free usypia (cold start ~30–60s rozwala budżet `<30s`), więc realnie **$7/mo Starter**. Region Frankfurt (EU) dostępny.
- **Vercel** — `vercel --prod` / `vercel rollback` / `vercel logs` w GA (CLI: Pass; drobny minus: na Hobby rollback tylko do poprzedniego deployu). Czysty serverless (Managed: Pass). `llms.txt` + markdown + agent-resources (Docs: Pass). Deploy/rollback natychmiastowy (Deploy API: Pass). Vercel MCP: **read-only, Public Beta** (MCP: Partial). Fluid Compute (domyślny) daje 300s — 30s fetch OK. Spada na 3. przez: **Hobby tylko niekomercyjny → B2B wymusza Pro $20/mo** oraz aktywny bug Astro 6 SSR na Vercel (esbuild, withastro/astro#16258).
- **Netlify** — **brak CLI rollback** (publikacja poprzedniego deployu przez UI) (CLI: Partial). Czysty serverless (Managed: Pass). `llms.txt` + markdown (Docs: Pass). `netlify deploy --prod` deterministyczny, draft-by-default jako bezpieczna wartość domyślna (Deploy API: Pass). Oficjalny Netlify MCP, rekomendowany obok CLI (MCP: Pass). **Dyskwalifikujące dla tego flow**: synchroniczny timeout funkcji ~26–30s jest sporny i niegwarantowany — rdzeń produktu (synchroniczne „submit → wynik") zagrożony; pewna ścieżka (Background Functions, 15 min) jest asynchroniczna i łamie UX.
- **Railway** — `railway up` / `railway logs` / `railway redeploy` niemal kompletne (CLI: Pass). Zarządzane kontenery, Dockerfile opcjonalny (Managed: Pass). Markdown docs, ale brak `llms.txt` (Docs: Partial). `railway up` deterministyczny (Deploy API: Pass). Oficjalny Railway MCP, ale beta/actively-developed (MCP: Partial). Długo żyjący Node — brak limitu czasu fetch. Brak free tier (~$5/mo Hobby). Atut współlokowanej bazy zneutralizowany (używamy Supabase). Gotcha: trzeba bindować `host: 0.0.0.0` (inaczej 502).
- **Fly.io** — `fly deploy` / `fly logs`, ale rollback = redeploy starszego obrazu, bez pojedynczego verbu (CLI: Partial). Zarządzane VM-ki, ale **własny Dockerfile do utrzymania** (Managed: Partial). Docs HTML, brak potwierdzonego `llms.txt` (Docs: Partial). `fly deploy` deterministyczny (Deploy API: Pass). `fly mcp server` eksperymentalny (MCP: Partial). Długo żyjący Node — brak limitu fetch. Brak free tier (~$2–5/mo always-on). Największy narzut ops dla solo MVP.

### Platformy na krótkiej liście

#### 1. Cloudflare Workers (rekomendacja)

Wygrywa, bo łączy trzy rzeczy naraz: (a) stack już na nią celuje — zero kosztu migracji adaptera; (b) ~30s synchroniczny fetch do AI działa **za darmo**, bez limitu czasu żądania, co bezpośrednio obsługuje nośnik wymagania `<30s p95`; (c) najbogatszy w całej puli ekosystem agentowy — `llms.txt`, ~16 serwerów MCP, plugin do Claude Code, pełna pętla operacyjna z `wrangler`. Dla projektu prowadzonego solo z agentem to maksymalna autonomia operacyjna z terminala.

#### 2. Render (runner-up)

Najbezpieczniejszy fallback i jedyna druga platforma z kompletem 5×Pass. To „nudny Node": zwykły długo żyjący serwer, zero niespodzianek edge, 30s fetch „po prostu działa" (limit żądania 100 min), **MCP w GA**, region Frankfurt (EU). Różnica wobec lidera: kosztuje $7/mo (Free usypia i cold-start rozwaliłby budżet latencji) oraz klucze API są szeroko zakresowane (gorzej z least-privilege niż na Cloudflare). To platforma, na którą się przenosisz, jeśli krawędziowy model Workers (luki `nodejs_compat`, upload przez Workera, churn API sekretów) okaże się w praktyce uciążliwy.

#### 3. Vercel

Najlepszy DX i dokumentacja w klasie, Fluid Compute rozwiązuje problem czasu funkcji (300s na Hobby). Różnica wobec lidera: **Hobby jest tylko niekomercyjny**, a to produkt B2B → wymuszone Pro **$20/mo/seat** (drożej niż Cloudflare i Render), plus aktywny bug Astro 6 SSR na Vercel (esbuild parse error na generowanych chunkach skryptów). Solidna trzecia opcja, gdyby dwie pierwsze odpadły z powodów organizacyjnych.

## Kontrola anty-uprzedzeniowa: Cloudflare Workers

Lider jest częściowo „zadekretowany" przez stack, więc kontrola celowo sprawdza, czy krawędziowy model Workers pasuje do aplikacji, której rdzeniem jest 30-sekundowa **synchroniczna** operacja serwerowa. Użytkownik po przeglądzie zdecydował: **zostajemy przy Cloudflare, ryzyka do rejestru**.

### Adwokat diabła — słabości

1. **API sekretów właśnie się zmieniło i może po cichu wywrócić produkcję.** Astro 6 + adapter v13 **usunęły** `Astro.locals.runtime.env` — wymagane jest `import { env } from 'cloudflare:workers'` albo `astro:env`. Jeśli wiring klienta Supabase SSR odwołuje się do starego wzorca, `SUPABASE_URL`/`SUPABASE_KEY` będą `undefined` **tylko w wdrożonym Workerze**, nie w `astro dev`.
2. **`nodejs_compat` to ruchomy cel.** `@supabase/ssr` i obsługa obrazów (FormData/Blob/crypto) zależą od wbudowanych `node:*` shimowanych przez workerd. Nie każde API jest w pełni spolyfillowane — zależność działająca lokalnie rzuca `not implemented` dopiero w wdrożonym Workerze, dokładnie na ścieżce upload + fetch do AI.
3. **Upload wieloMB zdjęć *przez* Workera.** Przepuszczanie ciężkich body przez Workera dobija do limitów rozmiaru żądania i pamięci/CPU na Free; prawdopodobnie wymusi architekturę signed-upload bezpośrednio do Supabase Storage — ograniczenie, którego zwykły serwer Node by nie narzucił.
4. **Brak prawdziwego rollbacku schematu Supabase.** `wrangler rollback` cofa Workera, ale polityki RLS i migracje żyją w Supabase. Zła migracja, która przecieka dane między tenantami (nośnik FR-002), **nie jest** cofana przez rollback Workera.
5. **Limit 50 subrequestów na Free.** Jeśli jedna analiza rozgałęzia się (upload + provider AI + odczyty/zapisy Supabase) z retry, zbliżasz się do limitu Free; Paid go podnosi — więc „darmowa" historia jest cieńsza, niż wygląda.

### Pre-Mortem — jak to mogło się nie udać

Zespół (solo dev) wdrożył na Workers, bo „stack już był skonfigurowany". Pierwsze tygodnie szły gładko — `astro dev` na workerd dawał wysoką wierność, demo działało. Pierwsza rysa: upload zdjęć od operatorów przy bramie (telefon, słabe łącze, 8-megapikselowe fotki) zaczął losowo zwracać 413/błędy pamięci na Free — body szło przez Workera. Refaktor na signed uploads do Supabase Storage zjadł tydzień z napiętego 3-tygodniowego budżetu. Potem ciche wycofanie: migracja dodająca politykę RLS miała błąd i operatorzy ciepłowni A przez kilka godzin widzieli dostawy ciepłowni B — `wrangler rollback` cofnął kod, ale nie schemat, a nikt nie miał skryptu down-migration. Logi? `wrangler tail` jest live-only; nocny incydent nie zostawił śladu, bo Workers Logs nie były skonfigurowane. Ostatni gwóźdź: sztandarowa zaleta Cloudflare — globalny edge — była nieistotna (jeden region, latencja zdominowana przez 30s providera AI), więc zespół poniósł całą złożoność edge, nie inkasując korzyści. Decyzja nie była zła technicznie — była niedopasowana do kształtu tej aplikacji, a koszt złożoności ujawnił się dopiero pod presją terminu.

### Nieznane niewiadome

- **„Wierność dev" zmieniła się świeżo** — Astro 6 odpala `astro dev` na prawdziwym workerd (plugin Vite, `platformProxy` zbędny). Dobre, ale znaczy, że każde założenie ze startera oparte na starym `wrangler dev`/`platformProxy` jest teraz subtelnie inne — to, czemu ufasz, właśnie się zmieniło.
- **Serwery MCP Cloudflare to endpointy produkcyjne BEZ etykiety GA/beta** — nie da się rozumować o ich SLA; nowe zunifikowane CLI `cf` jest w **technical preview** i może wyprzeć pamięć mięśniową `wrangler` w trakcie projektu.
- **Retencja logów ≠ live tail.** `wrangler tail` to tylko na żywo; trwałe logi wymagają Workers Logs/Logpush — debug sporadycznego timeoutu z zeszłej nocy może nie mieć żadnego śladu.
- **„Jedna platforma" to złudzenie** — Supabase i Cloudflare to dwa konta/dashboardy/rozliczenia; incydent izolacji tenantów wymaga korelowania logów przez oba.
- **Sztandarowa zaleta edge jest dla tej aplikacji w dużej mierze nieistotna** — jeden region + latencja AI-bound oznaczają, że globalny edge prawie nie pomaga w realnym UX. To dlatego Render jest legalnym runner-upem, nie zapchajdziurą.

## Historia operacyjna

Jak Cloudflare Workers działa na co dzień dla tego stacku — jedna konkretna odpowiedź na oś.

- **Wdrożenia podglądowe**: `wrangler versions upload` tworzy wersję preview z unikalnym URL-em `*-<version>.<subdomain>.workers.dev` bez ruszania produkcji; promocja przez `wrangler versions deploy`. Dla preview per-PR — Workers Builds generuje URL podglądu na PR. URL-e podglądu są publicznie routowalne → chroń je przez Cloudflare Access (zero-trust), bo dzielą sekrety Workera z produkcją, dopóki nie rozdzielisz środowisk.
- **Sekrety**: `SUPABASE_URL` / `SUPABASE_KEY` jako Workers Secrets, ustawiane `wrangler secret put <NAZWA>` (szyfrowane w spoczynku, nieodczytywalne z powrotem przez CLI — tylko nadpisywalne). Lokalnie: `.dev.vars` (gitignored). NIE w `wrangler.jsonc` (commitowany) i NIE w `.mcp.json`. Rotacja: ponowne `wrangler secret put` → obowiązuje od następnego deployu. Dostęp w kodzie: `astro:env` lub `import { env } from 'cloudflare:workers'` (NIE `Astro.locals.runtime.env` — usunięte w v13).
- **Wycofywanie**: `wrangler rollback [<version-id>]` natychmiast cofa Workera do wcześniejszej wersji (lista: `wrangler versions list`, do 100 wersji); typowy czas przywrócenia < 1 min. **Krytyczna uwaga**: cofa TYLKO kod Workera — migracje schematu/RLS w Supabase NIE są cofane. Trzymaj testowaną down-migration dla każdej zmiany dotykającej RLS, bo izolacja tenantów (FR-002) jest nośna.
- **Zatwierdzanie**: agent może bez nadzoru: `wrangler deploy`, `wrangler versions upload`, `wrangler tail`, `wrangler rollback` (tylko kod). Tylko człowiek: rotacja service-role key Supabase, usuwanie/zmiana polityk RLS na produkcji, usunięcie Workera lub projektu Supabase, zmiana planu/rozliczeń. Token API Cloudflare zakresowany do **Workers Scripts:Edit dla tego jednego projektu** — bez DNS, bez account-wide, bez billingu.
- **Logi**: live tail — `wrangler tail --format json` (strumień JSON, tylko na żywo). Logi historyczne: Workers Logs (już `observability.enabled: true` w `wrangler.jsonc`) — zapytania w dashboardzie lub przez Observability MCP / API. Dla dłuższej retencji: Logpush.

## Rejestr ryzyka

Każde ryzyko związane z soczewką, która je ujawniła — rejestr jest audytowalny.

| Ryzyko | Źródło | Prawd. | Wpływ | Łagodzenie |
|---|---|---|---|---|
| Wiring sekretów używa usuniętego `Astro.locals.runtime.env` (v13) → `undefined` tylko na produkcji | Adwokat diabła | Ś | W | Audyt każdej ścieżki `SUPABASE_*` pod `astro:env` / `import { env } from 'cloudflare:workers'` przed pierwszym deployem; smoke-test wdrożonego Workera, nie tylko `astro dev` |
| Luki `nodejs_compat` w `@supabase/ssr` / obsłudze obrazów rzucają `not implemented` tylko po wdrożeniu | Adwokat diabła | Ś | W | Smoke-test pełnej ścieżki (login + upload + fetch AI + zapis) na deployonym preview, nie lokalnie; `nodejs_compat` w `compatibility_flags` ✓ |
| Upload wieloMB zdjęć przez Workera dobija limity body/pamięci na Free | Adwokat diabła | W | Ś | Signed-upload bezpośrednio do Supabase Storage; Worker dostaje referencję, nie bajty obrazu |
| Zła migracja RLS przecieka dane między tenantami; `wrangler rollback` nie cofa schematu (FR-002) | Adwokat diabła / Pre-mortem | N | W (krytyczny) | Testowana down-migration dla każdej zmiany RLS; test izolacji tenantów w CI przed deployem; runbook: rollback Workera + ręczne odwrócenie migracji |
| `wrangler tail` jest live-only; nocny incydent bez śladu | Pre-mortem / Nieznane niewiadome | Ś | Ś | Workers Logs włączone (`observability.enabled: true` ✓); Logpush gdy potrzebna dłuższa retencja |
| Globalny edge nieistotny (1 region, latencja AI-bound) — złożoność bez korzyści | Pre-mortem / Nieznane niewiadome | W | N | Świadoma akceptacja; Render jako runner-up gdyby edge-model uwierał; nie optymalizuj pod edge |
| Limit 50 subrequestów/żądanie na Free przy rozgałęzieniu analizy + retry | Adwokat diabła / Research | N | Ś | Monitoruj liczbę subrequestów; Workers Paid ($5/mo) podnosi limit |
| Serwery MCP Cloudflare bez etykiety GA/beta; `cf` CLI w technical preview | Nieznane niewiadome / Research | Ś | N | Trzymaj się `wrangler` (GA) jako podstawy; MCP jako bonus, nie ścieżka krytyczna |
| Vendor split Supabase + Cloudflare: 2 dashboardy/rozliczenia przy incydencie izolacji | Nieznane niewiadome | Ś | Ś | Runbook incydentu izolacji korelujący logi Workers + Supabase; jeden właściciel obu kont |

## Rozpoczęcie pracy

Stack jest **już zescaffoldowany i skonfigurowany pod Workers** (`@astrojs/cloudflare` v13 + `wrangler.jsonc`), więc to nie „init nowego projektu", tylko „wdróż to, co jest, i podłącz sekrety". Komendy zweryfikowane pod Astro 6 / adapter v13 / wrangler (stan 2026-05-29):

1. **Audyt dostępu do sekretów pod Astro 6 / v13** (przed pierwszym deployem — ryzyko #1). Upewnij się, że kod czyta `SUPABASE_URL`/`SUPABASE_KEY` przez `astro:env` lub `import { env } from 'cloudflare:workers'`, NIE przez usunięte `Astro.locals.runtime.env`. Potwierdź `export const prerender = false` na route'ach API.
2. **Zaloguj wrangler i ustaw sekrety produkcyjne**: `npx wrangler login`, potem `npx wrangler secret put SUPABASE_URL` i `npx wrangler secret put SUPABASE_KEY`. Lokalnie te same wartości w `.dev.vars` (gitignored).
3. **Dev lokalny z wiernością runtime**: `npm run dev` — Astro 6 odpala na prawdziwym workerd przez plugin Vite (osobny `wrangler dev` zbędny).
4. **Pierwszy deploy jako preview, nie prod**: `npx wrangler versions upload` → preview URL bez ruszania produkcji. Smoke-test pełnej ścieżki (login + upload + analiza) NA preview — tu wychodzą luki `nodejs_compat`.
5. **Promocja do produkcji**: `npx wrangler deploy` (lub `npx wrangler versions deploy` by promować przetestowaną wersję). Awaryjnie: `npx wrangler rollback`.
6. **Token API zakresowany do tego projektu** (Workers Scripts:Edit, bez DNS/billing/account-wide) do CI i operacji agenta.

## Poza zakresem

W tych badaniach NIE oceniano:
- Konfiguracji obrazu Docker / Dockerfile.
- Konfiguracji potoku CI/CD (obecne `ci.yml` robi tylko `astro sync` + lint + build; auto-deploy-on-merge nie jest jeszcze podłączony — to świadomie poza zakresem badania).
- Architektury na skalę produkcyjną (wieloregionowe HA, DR, SLA).
