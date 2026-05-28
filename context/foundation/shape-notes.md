---
project: "Biomass typer"
context_type: greenfield
created: 2026-05-22
updated: 2026-05-25
product_type: web-app
tech_preferences:
  language_family: open
hard_constraints: []
checkpoint:
  current_phase: complete
  phases_completed: [1, 2, 3, 4, 5, 6]
  frs_drafted: 11
  quality_check_status: passed-with-override
timeline_budget:
  mvp_weeks: 3
---

# Shape Notes

> Seed idea (verbatim): "Chce zbudowac aplikacje ktora bedzie rozpoznawala biomase i podpowiadala co z nia mozna zrobic. Pomoc dla pracownikow cieplowni gdy przyjedzie do nich jakas biomasa to mozna wrzucic zdjecie i aplikacja szybko analizuje i identyfikuje co to jest i co z tym zrobic. Docelowo chcialbym to zrobic jako aplikacje mobilna ale jako mvp zrobmy to jako aplikacje webowa gdzie mozna wrzucac po prostu zdjecia i inne formaty png itp"

## Vision & Problem Statement

Operator przyjmujący dostawę biomasy w ciepłowni potrzebuje szybkiej, wiarygodnej oceny tego, co właśnie przyjechało — w czasie, w jakim ciężarówka jeszcze stoi przy bramie. Dziś ma dwie ścieżki, obie złe:

- **Ocena na oko** — szybka, ale obarczona ryzykiem pomyłki. Operator może zaakceptować partię, która okaże się mało kaloryczna albo niezgodna z deklaracją dostawcy.
- **Laboratorium** — kilka dni czekania na wynik, dodatkowo próbka pobrana w jednym miejscu nie reprezentuje całej partii (kilkadziesiąt ton). Wynik dociera za późno, żeby cokolwiek z dostawą zrobić.

Cena biomasy (~400 zł/tona) i duża zmienność wartości opałowej między partiami sprawiają, że błąd oceny to bezpośredni koszt firmy — przyjęta partia "dająca mało ciepła" została opłacona pełną stawką jak partia normalna. Skala problemu: każda dostawa.

**Kategoria bólu:** paraliż decyzyjny + brakująca zdolność. Operator nie ma narzędzia, na którym mógłby oprzeć decyzję w czasie rzeczywistym; istniejące narzędzie (laboratorium) działa za wolno i nie ocenia całej partii.

**Insight stojący za pomysłem:**
1. Modele wizyjne (CV / multimodalne LLM) w ostatnich 2–3 latach osiągnęły poziom, który pozwala rozpoznać typ biomasy ze zdjęcia z dokładnością zbliżoną do oceny doświadczonego operatora — to nowa możliwość, której nie było kilka lat temu.
2. Aplikacja może ocenić **całą partię** z kilku zdjęć (z różnych miejsc dostawy), gdy lab z definicji ocenia tylko pobraną próbkę. To inny rodzaj informacji, nie szybsza wersja tej samej.

## User & Persona

**Primary persona:** operator przyjmujący dostawę biomasy w ciepłowni.

- Fizycznie obecny przy przyjęciu dostawy (brama / waga / plac biomasy).
- Decyzja w czasie rzeczywistym: przyjąć / nie przyjąć / zgłosić wątpliwość mistrzowi zmiany / pobrać próbkę do lab.
- Pracuje w wielu ciepłowniach — produkt celowany jako B2B SaaS dla branży ciepłowniczej, nie pod jedną firmę.

## Access Control

Operator loguje się indywidualnym kontem (email + hasło lub OAuth — wybór mechanizmu to decyzja stackowa, poza zakresem PRD). Po zalogowaniu trafia do workspace swojej ciepłowni.

- **Model ról:** flat. Wszyscy zalogowani operatorzy w obrębie jednej ciepłowni mają te same uprawnienia (mogą wrzucać zdjęcia i widzieć wyniki własne oraz kolegów z tej samej ciepłowni). Brak hierarchii admin/member w MVP — model ról można rozbudować później, gdy klient zgłosi taką potrzebę.
- **Multi-tenancy:** od dnia 1. Każda ciepłownia jest odrębnym tenantem; operatorzy ciepłowni A nie widzą dostaw ciepłowni B. To zwiększa zakres MVP (workspace, izolacja danych, model rejestracji ciepłowni), ale jest świadomą decyzją zgodną z produktem B2B SaaS dla branży.

## Success Criteria

### Primary

Operator (zalogowany w workspace swojej ciepłowni) wrzuca 3 zdjęcia świeżej dostawy biomasy + numer dostawy. W ciągu sekund dostaje na ekranie: rozpoznany typ biomasy z confidence score, szacunkowy zakres wartości opałowej [MJ/kg], rekomendowaną decyzję (`accept` / `sample` / `reject`) wyliczoną z reguły opartej o zakres MJ/kg i próg kotła ciepłowni. Dostawa trafia do historii ciepłowni, dostępnej dla pozostałych operatorów w tej samej ciepłowni. Flow działający end-to-end = MVP udany.

### Secondary

Operator może wyeksportować wynik analizy dostawy do PDF (do teczki dostawy / faktury). Nice-to-have — jeśli zabraknie czasu, MVP może bez tego pojechać; dodawany pod koniec sprintu lub zostawiany do v2.

### Guardrails

- **Izolacja danych między ciepłowniami.** Tenant A nie widzi żadnych danych tenanta B — ani dostaw, ani zdjęć, ani historii. Wyciek między tenantami zabija produkt B2B. Wymaga rygorystycznego row-level security / tenant isolation w warstwie danych.
- **Czas odpowiedzi modelu < ~30s p95.** Operator czeka przy bramie z ciężarówką; jeśli aplikacja myśli minutę, wraca do oceny na oko i nie używa produktu. 30s p95 to próg user-perceived; konkretna implementacja (streaming, cache, model wybór) — downstream.

## Functional Requirements

### Auth & Workspace
- FR-001: Operator może zalogować się do aplikacji. Priority: must-have
- FR-002: Operator widzi wyłącznie dane swojej ciepłowni (tenant isolation). Priority: must-have

### Nowa dostawa & rozpoznanie
- FR-003: Operator może utworzyć nową dostawę, podając numer dostawy. Priority: must-have
- FR-004: Operator może załączyć 1+ zdjęć (formaty obrazów: PNG / JPG) do nowej dostawy. Priority: must-have
- FR-005: System rozpoznaje typ biomasy ze zdjęć (model AI / multimodalny) i zwraca rozpoznany typ wraz z confidence score (np. 0–100%). Operator widzi confidence i sam decyduje, czy ufać. Automatyczny próg fallbacku do "skieruj do lab" — out of MVP scope (brak danych empirycznych do strzelania progu). Priority: must-have
- FR-006: System pokazuje szacunkową wartość opałową jako zakres min–max [MJ/kg] — lookup z bazy typów biomasy po rozpoznanym typie. Baza typów trzyma dwie wartości (`min_MJ`, `max_MJ`) per typ, żeby nie udawać precyzji, której nie ma (ten sam typ potrafi się różnić nawet 2× w zależności od wilgotności/frakcji). Priority: must-have
- FR-007: System pokazuje rekomendowaną decyzję dla dostawy: `accept` / `sample` / `reject`. Reguła deterministyczna oparta o zakres MJ/kg (FR-006) i próg kotła ciepłowni (`min_MJ_próg`, konfiguracja per tenant):
  - `max_MJ < min_MJ_próg` → `reject` (nawet w optymistycznym scenariuszu za słaba)
  - `min_MJ < min_MJ_próg ≤ max_MJ` → `sample` (rozrzut obejmuje próg — niejednoznaczne)
  - `min_MJ ≥ min_MJ_próg` → `accept`

  Confidence AI z FR-005 jest sygnałem **widocznym dla operatora**, ale **nie wpływa na rekomendację** — operator sam decyduje, czy ufać typowi przy niskim confidence. Priority: must-have
- FR-009: Wynik analizy dostawy (typ, confidence, `min_MJ`, `max_MJ`, decyzja, zdjęcia, numer dostawy, timestamp, operator_id) zapisuje się trwale do historii ciepłowni. Priority: must-have

### Historia
- FR-010: Operator może przeglądać listę historycznych dostaw swojej ciepłowni. Priority: must-have
- FR-011: Operator może wyszukać dostawę po numerze dostawy. Priority: must-have

### Export (secondary)
- FR-012: Operator może wyeksportować pojedynczą dostawę (typ, confidence, zakres MJ/kg, decyzja, zdjęcia, numer, timestamp) do PDF. Priority: nice-to-have

## User Stories

### US-01: Rozpoznanie nowej dostawy biomasy

- **Given** operator jest zalogowany w workspace swojej ciepłowni
- **And** świeża dostawa biomasy stoi na placu / przy bramie
- **When** operator wybiera "Nowa dostawa", wpisuje numer dostawy
- **And** załącza 1+ zdjęć (PNG/JPG) tej partii
- **And** zatwierdza analizę
- **Then** w ciągu < ~30s p95 widzi na ekranie: rozpoznany typ biomasy (np. "zrębka iglasta") z confidence score, szacunkowy zakres wartości opałowej [`min_MJ` – `max_MJ` MJ/kg] z bazy typów, rekomendowaną decyzję (`accept` / `sample` / `reject`) wyliczoną z reguły FR-007
- **And** dostawa trafia do historii ciepłowni z timestampem, widoczna dla pozostałych operatorów tej samej ciepłowni

### US-02: Wyszukanie historycznej dostawy

- **Given** operator jest zalogowany w workspace swojej ciepłowni
- **And** w historii ciepłowni są poprzednie dostawy
- **When** operator otwiera widok historii
- **And** wpisuje numer dostawy w wyszukiwarce
- **Then** widzi pasującą dostawę na liście
- **And** może ją otworzyć, żeby zobaczyć: zdjęcia, rozpoznany typ z confidence, zakres MJ/kg, rekomendowaną decyzję, numer dostawy, timestamp

## Business Logic

### Reguła domenowa (one-sentence)

> Aplikacja zamienia zdjęcia dostarczonej biomasy w rekomendację decyzji (`accept` / `sample` / `reject`), porównując zakres wartości opałowej rozpoznanego typu biomasy z progiem kotła konkretnej ciepłowni.

### Sygnały na wejściu

1. Zdjęcia partii (1+, PNG/JPG) — FR-004.
2. `min_MJ_próg` ciepłowni — konfiguracja per tenant, snapshotowana na moment analizy.
3. Słownik `BiomassType` z `min_MJ` / `max_MJ` per typ — globalny seed.

### Krok analizy (deterministyczny pipeline)

1. Model AI/multimodalny rozpoznaje typ biomasy + zwraca confidence score → FR-005.
2. Lookup w `BiomassType` po `recognized_type_id` → `(min_MJ, max_MJ)` → FR-006.
3. Reguła FR-007 (porównanie zakresu z progiem):
   - `max_MJ < min_MJ_próg` → `reject`
   - `min_MJ < min_MJ_próg ≤ max_MJ` → `sample`
   - `min_MJ ≥ min_MJ_próg` → `accept`
4. Wynik (typ, confidence, zakres, decyzja, snapshot progu) trwale zapisuje się w `Delivery` → FR-009.

### Co reguła ŚWIADOMIE pomija (w MVP)

- **Confidence AI nie wchodzi do reguły.** Operator widzi confidence i sam ocenia, czy ufać. Próg confidence — strzelany bez danych empirycznych, kalibracja w v1.1 (Open Q).
- **Wilgotność / frakcja partii** — niemierzalne ze zdjęcia z akceptowalną dokładnością. Zakres `min_MJ`–`max_MJ` jest sposobem wyrażenia tej niepewności.
- **Cena dostawy.** FR-008 cut z MVP (Open Q).
- **Kontekst dostawcy / kontraktu.** Encje `Supplier` / `Contract` odłożone razem z ceną.

## Data Model

### Encje (szkic — bez wyboru bazy danych)

```
Tenant (Ciepłownia)
  id
  nazwa
  min_MJ_próg                   # parametr reguły FR-007, ustawiany ręcznie przy onboardingu
  created_at

Operator (User)
  id
  email
  tenant_id           → FK Tenant
  # mechanizm auth — decyzja stackowa, poza PRD

BiomassType
  id (slug)                     # np. "wood_chips_softwood"
  nazwa wyświetlana             # "zrębka iglasta"
  min_MJ
  max_MJ
  # seed globalny (nie per tenant); hardcoded fixture/migration w MVP

Delivery
  id
  tenant_id           → FK Tenant
  operator_id         → FK Operator           # audyt: kto wprowadził
  numer_dostawy
  recognized_type_id  → FK BiomassType
  recognized_type_name                        # snapshot nazwy na moment analizy
  confidence_score                            # 0–100
  min_MJ_snapshot                             # snapshot z BiomassType
  max_MJ_snapshot                             # snapshot z BiomassType
  min_MJ_próg_snapshot                        # snapshot z Tenant
  decyzja                                     # accept / sample / reject
  created_at

DeliveryPhoto
  id
  delivery_id         → FK Delivery
  blob / blob_url
```

### Decyzje projektowe (potwierdzone)

1. **Snapshot zamiast referencji.** `min_MJ`, `max_MJ`, `min_MJ_próg`, `recognized_type_name` zapisują się jako wartości historyczne na moment analizy. Późniejsze zmiany w `BiomassType` ani w `Tenant.min_MJ_próg` **nie wpływają wstecznie** na historię. Bezpieczeństwo audytu > normalizacja.
2. **`operator_id` w `Delivery`.** Każda dostawa wie, kto ją wprowadził — wartość audytowa dla wewnętrznego użycia ciepłowni.
3. **`BiomassType` globalny.** Słownik typów biomasy jest cechą branży, nie ciepłowni — wszyscy tenanci dzielą ten sam słownik. W MVP: hardcoded seed/migration; UI do edycji słownika — out of scope.
4. **Jeden próg kotła per tenant.** Założenie: jedna ciepłownia = jeden reprezentatywny próg `min_MJ_próg` (najsłabszy kocioł). Encja `Boiler` z osobnymi progami — odłożona na v1.1, jeśli pojawi się klient z istotnie różnymi kotłami.

### Tenant isolation (kluczowe dla Guardrails)

- Każda encja zawierająca dane użytkownika (`Operator`, `Delivery`, `DeliveryPhoto`) trzyma `tenant_id` jako klucz izolacji.
- Wszystkie zapytania w warstwie aplikacji **muszą** filtrować po `tenant_id` z sesji zalogowanego operatora.
- Mechanizm wymuszenia (row-level security w DB vs filtrowanie w warstwie aplikacji vs ORM middleware) — decyzja stackowa, downstream w 10x-tech-stack-selector.

## Stack Openness

- **`product_type`:** `web-app`. MVP w przeglądarce, responsive na telefon dopuszczone; native mobile out of scope MVP. Wizja mobilnej aplikacji żyje w `Vision & Problem Statement` i wraca w v2.
- **`tech_preferences.language_family`:** `open`. Wybór deleguje do `10x-tech-stack-selector` na podstawie profilu aplikacji (multimodal AI integration + multi-tenant web app + niewielki 3-tygodniowy MVP).
- **Twarde constraints wobec stacku:** brak. Tech-stack-selector ma pełną swobodę rekomendacji w warstwie frameworka, bazy danych, auth providera, AI providera, deployment platform.

## Quality Gate

- **Empty-CRUD:** ✓ NIE wykryto. Reguła domenowa jest realna i opiera się na trzech ortogonalnych sygnałach (typ z AI, zakres MJ/kg z słownika, próg kotła z tenanta). Deterministyczny algorytm w FR-007.
- **MVP-too-big:** ⚠ WYKRYTO (soft-gate), override zaakceptowany.
  - Sygnały: multi-tenancy od dnia 1 + integracja AI + historia + onboarding manualny + auth → estymata ~15–20 dni dev = ~50–80h after-hours przy `mvp_weeks: 3`.
  - Override źródło: user świadomie zaakceptował multi-tenancy w `Access Control` jako spójną z B2B SaaS positioning. FR-012 (PDF) już oznaczone jako nice-to-have, naturalny pierwszy cut jeśli budżet się nie zgodzi.
  - Kandydaci awaryjnych cięć (jeśli MVP się nie wyrabia): (1) FR-012 cut, (2) FR-011 (search) cut, (3) historia (FR-009/FR-010) cut → MVP single-shot bez persistence, (4) single-tenant launch z jedną ciepłownią pilotażową, multi-tenant w v1.1.

## Open Questions

- **Czy konkurencja już istnieje?** User flagował niepewność: być może ktoś z branży lub spoza już ma podobne narzędzie. Walidacja konkurencji jest do zrobienia przed wyborem stacku / startem MVP.
- **Próg kotła `min_MJ_próg` per tenant — kto go wprowadza i kiedy?** FR-007 wymaga, żeby każda ciepłownia miała zdefiniowany próg minimalnej wartości opałowej akceptowalnej dla swojego kotła. W MVP onboarding ciepłowni jest manualny (Open Q poniżej) — próg wprowadza admin produktu przy zakładaniu tenanta na podstawie informacji od ciepłowni. UI do edycji progu przez ciepłownię — out of MVP scope (v2).
- **Onboarding ciepłowni — out of MVP scope.** W MVP nowe ciepłownie + pierwszych operatorów dodaje ręcznie admin produktu (poza aplikacją). Self-service signup do v2.
- **Baza typów biomasy w v1.1+ — proces aktualizacji słownika.** W MVP `BiomassType` jest hardcoded seed/migration (Data Model). Otwarte pytanie na v1.1: jak zarządzać słownikiem, gdy klient zgłosi nowy typ biomasy lub trzeba poprawić `min_MJ`/`max_MJ`? Kandydaci: (a) release aplikacji z nową migracją, (b) tooling admina produktu poza aplikacją, (c) UI admina w aplikacji (out of MVP).
- **Cena dostawy [zł/tona] — odłożona z MVP, do walidacji na v1.1/v2.** FR-008 cut: rynkowa cena biomasy nie jest funkcją tylko typu (zmienia się sezonowo, regionalnie, per kontrakt), flat lookup zestarzeje się w tygodnie i zacznie aktywnie dezinformować operatora. Realny wariant powrotu: cena z kontraktu dostawca↔tenant (encje `Dostawca` + `Kontrakt` + input dostawcy przy nowej dostawie). Wraca po walidacji rdzenia produktu (oceny technicznej dostawy). Decyzja cenowa w MVP zostaje w gestii człowieka z dostępem do kontraktu, nie operatora na bramie.
- **Próg confidence dla operatora — strzelany, do kalibracji empirycznej.** FR-005 zwraca confidence score; FR-007 nie używa go jako bramki (operator widzi i sam ocenia). Pytanie: czy po pierwszych N dostawach (np. N=100) widać próg, poniżej którego operator i tak idzie do `sample` ręcznie? Jeśli tak — kandydat do v1.1 (`confidence < próg → automatyczna rekomendacja sample`).
