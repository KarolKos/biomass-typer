---
project: "Biomass typer"
version: 1
status: draft
created: 2026-05-25
context_type: greenfield
product_type: web-app
target_scale:
  users: small
  qps: low
  data_volume: small
timeline_budget:
  mvp_weeks: 3
  hard_deadline: null
  after_hours_only: true
---

# PRD: Biomass typer

## Vision & Problem Statement

Operator przyjmujący dostawę biomasy w ciepłowni potrzebuje szybkiej, wiarygodnej oceny tego, co właśnie przyjechało — w czasie, w jakim ciężarówka jeszcze stoi przy bramie. Ma dwie ścieżki, obie złe: ocena na oko (szybka, ale obarczona ryzykiem pomyłki) albo laboratorium (kilka dni czekania na wynik pobranej próbki, która z definicji nie reprezentuje całej kilkudziesięciotonowej partii). Cena biomasy (~400 zł/tona) i duża zmienność wartości opałowej między partiami sprawiają, że błąd oceny to bezpośredni koszt firmy — przyjęta partia "dająca mało ciepła" została opłacona pełną stawką jak partia normalna. Skala problemu: każda dostawa.

W ostatnich 2–3 latach pojawiła się zdolność komputerowej identyfikacji typu biomasy ze zdjęcia z dokładnością zbliżoną do oceny doświadczonego operatora — wcześniej nieosiągalna na poziomie pozwalającym budować na niej narzędzie operatora. Drugi insight: z kilku zdjęć z różnych miejsc dostawy aplikacja ocenia **całą partię**, podczas gdy laboratorium z definicji ocenia jedną pobraną próbkę. To nie szybsza wersja istniejącego narzędzia — to inny rodzaj informacji.

## User & Persona

**Primary persona:** operator przyjmujący dostawę biomasy w ciepłowni.

- Fizycznie obecny przy przyjęciu dostawy (brama / waga / plac biomasy).
- Decyzja w czasie rzeczywistym: przyjąć / nie przyjąć / zgłosić wątpliwość mistrzowi zmiany / pobrać próbkę do laboratorium.
- Pracuje w ciepłowni będącej klientem produktu; produkt celowany jako B2B SaaS dla branży ciepłowniczej, nie pod jedną firmę.
- Moment, w którym sięga po aplikację: świeża dostawa stoi na placu, operator musi w ciągu sekund/minut zdecydować, co z nią zrobić.

## Success Criteria

### Primary

- Operator zalogowany w obrębie swojej ciepłowni wrzuca zdjęcia świeżej dostawy i numer dostawy, i w ciągu sekund widzi na ekranie: rozpoznany typ biomasy z confidence score, szacunkowy zakres wartości opałowej [MJ/kg] oraz rekomendowaną decyzję (`accept` / `sample` / `reject`) wyliczoną z reguły opartej o zakres MJ/kg i próg akceptacji ciepłowni. Dostawa trafia do historii ciepłowni, widocznej dla pozostałych operatorów tej samej ciepłowni. Flow działający end-to-end = MVP zaliczony.

### Secondary

- Operator może wyeksportować pojedynczą dostawę do PDF (do teczki dostawy / faktury). Nice-to-have — jeśli zabraknie czasu, MVP może bez tego pojechać.

### Guardrails

- **Izolacja danych między ciepłowniami.** Operator widzi wyłącznie dane swojej ciepłowni — żaden wyciek między ciepłowniami. Wyciek zabija produkt B2B.
- **Czas od submita zdjęć do widocznego wyniku analizy < ~30s p95.** Operator czeka przy bramie z ciężarówką; jeśli aplikacja myśli minutę, wraca do oceny na oko i przestaje używać produktu.

## User Stories

### US-01: Rozpoznanie nowej dostawy biomasy

- **Given** operator jest zalogowany w obrębie swojej ciepłowni
- **And** świeża dostawa biomasy stoi na placu / przy bramie
- **When** operator wybiera "Nowa dostawa" i wpisuje numer dostawy
- **And** załącza 1+ zdjęć (PNG / JPG) tej partii
- **And** zatwierdza analizę
- **Then** w ciągu < ~30s p95 widzi na ekranie: rozpoznany typ biomasy (np. "zrębka iglasta") z confidence score, szacunkowy zakres wartości opałowej (`min_MJ`–`max_MJ` MJ/kg), rekomendowaną decyzję (`accept` / `sample` / `reject`)
- **And** dostawa trafia do historii ciepłowni z czasem wprowadzenia, widoczna dla pozostałych operatorów tej samej ciepłowni

### US-02: Wyszukanie historycznej dostawy

- **Given** operator jest zalogowany w obrębie swojej ciepłowni
- **And** w historii ciepłowni są poprzednie dostawy
- **When** operator otwiera widok historii
- **And** wpisuje numer dostawy w wyszukiwarce
- **Then** widzi pasującą dostawę na liście
- **And** może ją otworzyć, żeby zobaczyć: zdjęcia, rozpoznany typ z confidence, zakres MJ/kg, rekomendowaną decyzję, numer dostawy, czas wprowadzenia

## Functional Requirements

### Auth & dostęp

- FR-001: Operator może zalogować się do aplikacji. Priority: must-have
- FR-002: Operator widzi wyłącznie dane swojej ciepłowni (izolacja między ciepłowniami). Priority: must-have

### Nowa dostawa & rozpoznanie

- FR-003: Operator może utworzyć nową dostawę, podając numer dostawy. Priority: must-have
- FR-004: Operator może załączyć 1+ zdjęć (PNG / JPG) do nowej dostawy. Priority: must-have
- FR-005: Operator otrzymuje rozpoznany typ biomasy oraz confidence score (0–100%) wskazujący pewność rozpoznania, na podstawie wgranych zdjęć. Priority: must-have

  > Socratic: zdjęcie nie ujawnia wilgotności ani frakcji, a różne typy biomasy mogą wyglądać bardzo podobnie. Decyzja: confidence score widoczny dla operatora, który sam ocenia, czy ufać. Automatyczny próg fallbacku do `sample` przy niskim confidence — odłożony, bo bez danych empirycznych byłby strzelany.

- FR-006: Operator widzi szacunkowy zakres wartości opałowej (`min_MJ`–`max_MJ` MJ/kg) dla rozpoznanego typu biomasy. Priority: must-have

  > Socratic: ta sama biomasa potrafi mieć kaloryczność różniącą się 2× w zależności od wilgotności/frakcji — pojedyncza liczba sugerowałaby precyzję, której nie ma. Decyzja: zakres min–max zamiast single value.

- FR-007: Operator widzi rekomendowaną decyzję dla dostawy (`accept` / `sample` / `reject`), wyliczoną deterministycznie z porównania zakresu MJ/kg rozpoznanego typu i progu akceptacji ciepłowni. Priority: must-have

  > Socratic: FR-007 bez jawnej reguły to pusty UI. Decyzja: reguła deterministyczna (pełna treść w `## Business Logic`). Confidence z FR-005 jest sygnałem dla operatora, ale nie wchodzi do reguły — operator sam decyduje, czy ufać.

- FR-009: Wynik analizy każdej dostawy jest trwale dostępny w historii ciepłowni dla wszystkich operatorów tej samej ciepłowni. Priority: must-have

### Historia

- FR-010: Operator może przeglądać listę historycznych dostaw swojej ciepłowni. Priority: must-have
- FR-011: Operator może wyszukać dostawę po numerze dostawy. Priority: must-have

### Export (secondary)

- FR-012: Operator może wyeksportować pojedynczą dostawę do PDF (rozpoznany typ, confidence, zakres MJ/kg, decyzja, zdjęcia, numer dostawy, czas). Priority: nice-to-have

## Non-Functional Requirements

- Pomiędzy ciepłowniami nie ma żadnego wycieku danych — operator nie widzi dostaw, zdjęć ani historii innej ciepłowni.
- Czas od submita zdjęć przez operatora do widocznego wyniku analizy: < ~30s p95.
- # TODO: dodatkowe NFR (kompatybilność przeglądarek, dostępność serwisu, zachowanie przy słabym połączeniu mobilnym operatora przy bramie, zachowanie gdy rozpoznanie nie jest możliwe) — see Open Questions #7

## Business Logic

Aplikacja zamienia zdjęcia dostarczonej biomasy w rekomendację decyzji (`accept` / `sample` / `reject`), porównując zakres wartości opałowej rozpoznanego typu biomasy z progiem akceptacji konkretnej ciepłowni.

Reguła operuje na trzech sygnałach: zdjęciach dostawy (1+, dostarczonych przez operatora przy przyjęciu partii), zakresie wartości opałowej (`min_MJ`–`max_MJ`) dla każdego typu biomasy (informacja stała, taka sama dla wszystkich ciepłowni), oraz progu akceptacji konkretnej ciepłowni (`min_MJ_próg` — minimalna wartość opałowa, którą ciepłownia uznaje za akceptowalną dla swojego kotła; ustawiana per ciepłownia przy onboardingu).

Algorytm decyzyjny jest deterministyczny:

- jeśli `max_MJ` rozpoznanego typu < `min_MJ_próg` ciepłowni → `reject` (nawet w optymistycznym scenariuszu za słaba);
- jeśli `min_MJ` < `min_MJ_próg` ≤ `max_MJ` → `sample` (zakres obejmuje próg — niejednoznaczne, do laboratorium);
- jeśli `min_MJ` rozpoznanego typu ≥ `min_MJ_próg` → `accept`.

Output reguły to: rozpoznany typ, confidence score wskazujący pewność rozpoznania, zakres szacowanej kaloryczności oraz rekomendacja decyzji. Operator widzi te cztery elementy na ekranie po sekundach od submita; decyzja końcowa należy do operatora. Confidence rozpoznania jest sygnałem widocznym, ale **nie wchodzi do reguły** — operator sam ocenia, czy ufać typowi przy niskim confidence. Wilgotność i frakcja partii (które realnie wpływają na faktyczną kaloryczność) nie są mierzone w MVP — szeroki zakres `min_MJ`–`max_MJ` per typ jest sposobem wyrażenia tej niepewności.

## Access Control

Każdy operator loguje się indywidualnym kontem. Mechanizm uwierzytelnienia — decyzja stackowa, poza zakresem PRD. Po zalogowaniu pracuje w kontekście swojej ciepłowni.

**Model ról:** flat. Wszyscy zalogowani operatorzy w obrębie jednej ciepłowni mają te same uprawnienia — mogą wrzucać zdjęcia i widzieć wyniki własne oraz kolegów z tej samej ciepłowni. Brak hierarchii admin/member w MVP.

**Izolacja danych między ciepłowniami (od dnia 1):** każda ciepłownia ma odrębny dostęp do swoich danych — operatorzy ciepłowni A nie widzą żadnych danych ciepłowni B (ani dostaw, ani zdjęć, ani historii). To świadoma decyzja zgodna z pozycjonowaniem B2B SaaS — pojedynczy wyciek między ciepłowniami zabija produkt.

**Onboarding ciepłowni (out of MVP scope):** w MVP nowe ciepłownie + pierwszych operatorów dodaje ręcznie admin produktu poza aplikacją; admin ustawia również próg akceptacji ciepłowni (`min_MJ_próg`) na podstawie informacji od ciepłowni. Self-service signup ciepłowni i edycja progu przez ciepłownię — out of MVP (v2).

## Non-Goals

- **Native mobile app.** MVP w przeglądarce (responsive na telefon dopuszczone); natywna iOS/Android wraca w v2 jako naturalna ewolucja po walidacji rdzenia produktu.
- **Rekomendowana cena dostawy [zł/tona].** Wycięta z MVP — rynkowa cena nie jest funkcją tylko typu biomasy (sezonowość, region, kontrakt, jakość). Stała wartość per typ zestarzeje się i zacznie aktywnie dezinformować operatora. Wraca w v1.1 jako cena z kontraktu dostawca↔ciepłownia.
- **UI edycji słownika typów biomasy.** W MVP słownik typów dostarczany jest jako część aplikacji; jego aktualizacja wymaga nowej wersji aplikacji. UI edycji (dodawanie typów, korekta zakresu MJ/kg) wraca w v1.1.
- **UI edycji progu akceptacji ciepłowni.** Próg ustawiany przez admina produktu przy onboardingu ciepłowni. Edycja przez ciepłownię w aplikacji — v2.
- **Hierarchia ról wewnątrz ciepłowni (admin/member).** Flat model w MVP. Hierarchia gdy klient zgłosi konkretną potrzebę.
- **Self-service signup ciepłowni.** Onboarding ręczny w MVP (admin produktu poza aplikacją). Self-service signup → v2.
- **Osobne progi per kocioł w obrębie ciepłowni.** Założenie: 1 ciepłownia = 1 reprezentatywny próg (najsłabszy kocioł). Osobne progi per kocioł → v1.1 jeśli pojawi się klient z istotnie różnymi kotłami.
- **Automatyczny fallback do `sample` przy niskim confidence rozpoznania.** W MVP confidence widoczne dla operatora, ale nie wchodzi do reguły. Automatyczny próg → v1.1 po kalibracji na pierwszych N dostawach.
- **Dziennik zmian / edycja dostaw / odzyskiwanie usuniętych dostaw.** Wpis dostawy w MVP jest niemodyfikowalny (zachowuje wartość historyczną decyzji). Pełen dziennik zmian → v2 jeśli regulacyjnie wymagany.
- **Wsparcie zarządzania dostawcami i kontraktami w aplikacji.** Wraca razem z ceną w v1.1.

## Open Questions

1. **Walidacja konkurencji.** Czy ktoś z branży lub spoza już ma podobne narzędzie? Sprawdzić przed wyborem stacku i startem MVP. Owner: user.
2. **Próg akceptacji ciepłowni (`min_MJ_próg`) — kto wprowadza i kiedy.** Reguła decyzyjna w `## Business Logic` wymaga progu per ciepłownia. W MVP: admin produktu ustawia przy ręcznym onboardingu na podstawie informacji od ciepłowni. UI edycji przez ciepłownię — out of MVP (v2). Owner: user (operacyjnie admin produktu).
3. **Słownik typów biomasy w v1.1+ — proces aktualizacji.** W MVP słownik dostarczany jest w aplikacji. Otwarte na v1.1: jak zarządzać słownikiem, gdy klient zgłosi nowy typ lub korektę zakresu? Kandydaci: (a) release nowej wersji aplikacji z aktualizacją, (b) tooling admina poza aplikacją, (c) UI admina w aplikacji. Owner: user (w v1.1).
4. **Cena dostawy — odłożona z MVP, walidacja w v1.1/v2.** FR-008 cut: rynkowa cena nie jest funkcją tylko typu biomasy. Realny wariant powrotu: cena z kontraktu dostawca↔ciepłownia. Decyzja po pierwszych użyciach produktu — jeśli operatorzy realnie używają oceny technicznej, cena dorzucana w v1.1. Owner: user.
5. **Próg confidence dla automatycznego fallbacku — kalibracja empiryczna.** Confidence widoczne dla operatora; FR-007 nie używa go jako bramki. Otwarte: czy po pierwszych N dostawach (N≈100) widać próg, poniżej którego operator i tak idzie do `sample` ręcznie? Jeśli tak — kandydat do v1.1 (`confidence < próg → automatyczna rekomendacja sample`). Owner: user (post-MVP).
6. **Acceptance Criteria explicit dla US-01 / US-02.** Schema PRD sugeruje sub-blok `#### Acceptance Criteria` z boundary cases. W shape-notes Given/When/Then pokrywa happy path; brak explicit case'ów: zdjęcie nieczytelne, zdjęcie nie-biomasy, brak rozpoznania z bardzo niskim confidence, błąd uploadu, timeout analizy. Owner: user. By: przed lock PRD.
7. **Pokrycie NFR.** W PRD zapisaliśmy 2 NFR (izolacja danych, czas odpowiedzi). Brak: kompatybilność przeglądarek (operator może używać telefonu albo laptopa przy bramie), dostępność serwisu, zachowanie przy słabym połączeniu mobilnym, zachowanie gdy rozpoznanie nie jest możliwe (output "nie wiem"). Do uzupełnienia przed lock PRD. Owner: user.
8. **MVP scope vs 3-week after-hours budget — świadome ryzyko.** `mvp_weeks: 3`, `after_hours_only: true` (potwierdzone) → realny budżet ~40–65h. Estymata zakresu ~50–80h dev (izolacja danych od dnia 1 + integracja rozpoznawania + historia + manualny onboarding ciepłowni). Soft-gate MVP-too-big wykryty; override zaakceptowany (izolacja danych świadomie pozostawiona od dnia 1). Budżet jest **napięty na granicy lub powyżej**. Kandydaci awaryjnych cięć: FR-012 (PDF), FR-011 (search), historia całość (FR-009/FR-010), pilot z 1 ciepłownią bez pełnej izolacji w pierwszej iteracji. Owner: user.
