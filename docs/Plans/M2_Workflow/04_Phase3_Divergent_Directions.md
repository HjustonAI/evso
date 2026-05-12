# Faza 3 — Divergent Solution Sketches (Milestone 2)

> **Status:** wynik Fazy 3 workflow M2.
> **Anti-bias ack:** wczytałem `docs/Plans/M2_Workflow/00_Workflow_Plan.md`, `01_Phase0_Problem_Map.md`, `02_Phase1_Capability_Map.md`, `03_Phase2_Decision_Matrix.md`. **NIE czytałem** żadnych plików z `archive/*`.
> **Cel pliku:** naszkicować 3 **zasadniczo różne** kierunki architektoniczne, każdy realistycznie obroniony, żeby Faza 4 (panel ekspertów) miała materiał do stress testu.

---

## 0. Co czyni te kierunki "zasadniczo różnymi"

Aby uniknąć trzech wariantów tego samego pomysłu, kierunki różnią się jednocześnie po **trzech osiach**:

| Oś | Kierunek A | Kierunek B | Kierunek C |
|---|---|---|---|
| **Lokalizacja "mózgu" (gdzie żyje decyzja matchingu)** | Wewnątrz ClickUp (AI Agent + AI Fields + Automations) | Wewnątrz Make (router + jeden HTTP call do Anthropic) | Wewnątrz mikroservisu (Cloudflare Worker z agentyczną pętlą + tool use) |
| **Model stanu (gdzie żyje "wiedza o relacji email↔task")** | Native ClickUp: pole EMAIL na tasku + AI Agent konteksty Workspace | Make Data Store (`email → task_ids`, `Message-ID` ledger) + ClickUp jako ostateczny storage | Własna baza (Cloudflare KV / D1) jako event ledger; ClickUp i Make czerpią z niej |
| **Tryb decyzji (jak zapada wybór)** | Probabilistyczny agent ClickUp w "swoim" sandboxie — mała kontrola promptu, ale rozumie kontekst Workspace | Deterministyczne pre-filtry w Make + **jednorazowy** Anthropic call na 1–N kandydatach — pełna kontrola promptu | **Agentyczna pętla** (multi-step, tool use): search → evaluate → ewentualny refine → decision; logika polityki jako kod (P1–P3 explicit) |
| **Operability (kto debuguje gdy się psuje)** | Bartek + ClickUp Support; trudne wersjonowanie agenta | Bartek solo w UI Make; wzorce z Phase 1 | Bartek solo w kodzie + deploy pipeline; obcy idiom dla zespołu |
| **Lock-in vector** | ClickUp (jeśli zmienimy CRM, mózg trzeba przepisać) | Make (Make jest okablowaniem; relatywnie cienka warstwa AI) | Własny kod (najmniejszy zewnętrzny lock-in, ale największy własny ciężar) |

Te trzy osie są skorelowane (kierunek = spójna konfiguracja), ale różnice są **kategorialne**, nie ilościowe. Żaden nie jest słomianym ludkiem — każdy ma realny scenariusz, w którym wygrywa.

---

## Kierunek A — "ClickUp-native heavy"

### A.1. Architektura wysokopoziomowa

```
[Strony brand: Forminator/WPForms]
          │  (Phase 1 — bez zmian)
          ▼
   [Make: Form Intake]  ──────────►  [ClickUp: NOWE ZAPYTANIA]
                                          ▲   ▲
                                          │   │
[Skrzynki kontakt@*  ── forward ──►  list-level mail-to-task
 Dhosting]                            (ClickUp natywne, CU-2)
                                          │
                                          ▼
                                  [ClickUp AI Agent
                                   "Email Matcher"]
                                          │
                                          │ używa: AI Fields,
                                          │ Custom Fields, Search Tasks
                                          ▼
                                  decyzja: dołącz do istniejącego
                                  taska / zostaw jako nowy / oznacz
                                  "Do weryfikacji"
                                          │
                                          ▼
                              [ClickUp Automations]
                                @mention assignee, push notif
```

Make pozostaje cienką warstwą I/O dla formularzy (Phase 1 bez zmian). **Cała ścieżka emaili idzie obok Make** — wprost do list-level email ClickUp (CU-2). Nowy task tworzony jest natywnie przez ClickUp; ClickUp AI Agent (CU-3) reaguje na trigger "task created from email", używa AI Fields (CU-4) do podsumowania, Search Tasks po `EMAIL` do znalezienia kandydatów, i decyduje, czy ten task jest *naprawdę nowy*, czy powinien zostać zmergowany / oznaczony jako duplikat / zlinkowany do istniejącego.

Make wraca dopiero w jednym miejscu: **outbound** (Phase 1 Form Intake, którego nie ruszamy) i ewentualnie jako "AI sugestia tytułu maila wychodzącego" (Problem D) — pojedynczy webhook trigger ClickUp → Make → Anthropic → update Custom Field `SUGEROWANY_TYTUŁ_MAILA`.

### A.2. Mapping na 6 problemów

- **A — fragmentacja historii:** ✅ list-level mail (CU-2) wpina każdy nowy email jako task, in-thread reply (CU-1) wraca w wątek. Po decyzji Agent o dołączeniu — Agent wykonuje "merge / link / move comment" w obrębie ClickUp.
- **B — identyfikacja:** ✅ Custom Field EMAIL + Agent z dostępem do Search Tasks. Lookup deterministyczny + Agent sprawdza politykę P2.
- **C — matching:** ✅ AI Agent jest matcherem. Realizuje politykę P1 progu pewności i P3 duplikatów wewnątrz ClickUp. **To jest serce tego kierunku.**
- **D — sugestia tytułu:** 🟡 możliwe przez AI Field na tasku (`SUGEROWANY_TYTUŁ_MAILA = AI Field z formuły`). Refresh nie jest live, ale "good enough".
- **E — powiadomienia:** ✅ ClickUp Automations + Inbox + push (CU-6, CU-8). Najnaturalniej z całych trzech kierunków — wszystko dzieje się w ClickUp, więc notyfikacje są "na miejscu".
- **F — skalowalność:** ✅ jako pochodna A–E.

### A.3. Kluczowe ryzyka

1. **Lock-in na ClickUp Agents.** Jeśli plan Business **nie obejmuje** produkcyjnego użycia AI Agents w potrzebnym zakresie (Q1 z Fazy 1), kierunek upada lub wymaga upgrade'u na Enterprise. Realne ryzyko cenowe.
2. **Deterministyczność matchera.** AI Agent w ClickUp ma mniej deterministyczne zachowanie niż prompt w Make: trudniej wersjonować prompt, trudniej testować, trudniej audytować "dlaczego Agent wybrał ten task". `MATCHING_REASONING` Custom Field musi być wypełniany przez Agenta — wymaga kreatywnej konfiguracji.
3. **Brak doświadczenia produkcyjnego z ClickUp Agents w EVSO** — Phase 1 nie używa ich; wprowadzenie w M2 to **nowa technologia** z perspektywy Bartka (S1).
4. **Polityka powracających klientów (P2)** opiera się na 90-dniowym oknie — w ClickUp Automations to wymaga workaroundu (np. filtr po `LAST_EMAIL_AT`), a Agent musi to respektować w prompt'cie. Łatwiejsze deklaratywnie w Make.
5. **`kontakt@*` w Dhosting → list-level email ClickUp** wymaga forward (Q7 z Fazy 1). Działa, ale to nowa konfiguracja, nie pokryta Phase 1.

### A.4. Koszt rozwoju (przybliżony)

- Konfiguracja list-level email + forward Dhosting: **0.5 PD**.
- Custom Fields nowe (M2 set z Fazy 2): **0.5 PD**.
- Konfiguracja AI Agent (prompt + dostęp do Search Tasks + reguły): **2–3 PD** (większość to iteracje na nieprzewidywalnym zachowaniu).
- AI Fields dla `AI_PODSUMOWANIE` + `SUGEROWANY_TYTUŁ_MAILA`: **0.5 PD**.
- Automations powiadomień: **0.5 PD**.
- Test na `automatyzacjaevso@gmail.com` + kalibracja progów: **2 PD**.
- **Suma: ~6–7 PD.**

### A.5. Koszt operacyjny

- ClickUp AI uses (Agents + AI Fields) — pakiet planu Business + ewentualne over-cap. **Realne ryzyko, że 100 emaili/tydz × kilka uses na każdy (Agent decision + AI Field refresh + summarize) wyczerpie pakiet, wymuszając upgrade.** `[do potwierdzenia]` Q3 z Fazy 1 to *blocker* dla tego kierunku.
- Make: minimalne (tylko outbound dla formularzy + sugestia tytułu).
- Anthropic API bezpośrednio: minimalne (Anthropic używany tylko przez Make w drobnych miejscach).
- **Total est. (przy założeniu mieścenia się w pakiecie ClickUp):** kilkanaście PLN/mc. **Przy over-cap:** może wzrosnąć do kilkuset PLN/mc.

### A.6. Operability

- **Plus:** wszystko w jednym narzędziu — handlowiec widzi historię, decyzję AI, notyfikację w ClickUp. Nie ma "drugiego ekranu".
- **Minus:** debug "dlaczego Agent zdecydował X" jest trudny — ClickUp nie eksponuje pełnych logów decyzji Agent. Wersjonowanie promptu Agenta jest słabe (UI ClickUp, brak gita).
- **Liczba osób, które muszą rozumieć system:** 1 (Bartek) — handlowcy są tylko użytkownikami. Plus operability.
- **Co się dzieje, gdy ClickUp Agents zmieni API/limity:** musimy przeprojektować mózg matchera. To samo dla `mg.clickup.com`. Wektor pojedynczego punktu awarii jest skupiony.

---

## Kierunek B — "Make-watcher heavy"

### B.1. Architektura wysokopoziomowa

```
[Strony brand]                     [Skrzynki kontakt@* ── forward ──►
      │                              automatyzacja@gmail.com (single inbox)]
      ▼                                       │
[Make: Form Intake]                   [Make: Email Intake M2]
   (Phase 1, bez zmian)                       │
      │                                       │ Gmail Watch (5 min)
      │                                       ▼
      │                          ┌────────────────────────────┐
      │                          │ 1. Idempotency check       │
      │                          │    (Data Store: Message-ID)│
      │                          │ 2. Search ClickUp by EMAIL │
      │                          │    (filter: 90-day window, │
      │                          │     not closed)            │
      │                          │ 3. Iterator po kandydatach │
      │                          │ 4. HTTP → Anthropic Haiku  │
      │                          │    (single shot, JSON out: │
      │                          │     decision+confidence    │
      │                          │     +reasoning+target_id)  │
      │                          │ 5. Router:                 │
      │                          │    ≥0.85 → comment to task │
      │                          │    0.6–0.85 → new task +   │
      │                          │       Do weryfikacji + link│
      │                          │    <0.6 → new task         │
      │                          │ 6. Update Data Store       │
      │                          │ 7. Trigger ClickUp comment │
      │                          │    @mention assignee       │
      │                          │ 8. Error handler:          │
      │                          │    fallback Do weryfikacji │
      │                          └─────────────┬──────────────┘
      │                                        │
      ▼                                        ▼
              [ClickUp: NOWE ZAPYTANIA / ZLECENIA]
                       (storage + UI)
```

ClickUp jest **prawie pasywnym storage + UI**. Make jest centralnym mózgiem: jego router materializuje politykę P1, jego Data Store materializuje P2 prefiltr (90 dni) i idempotentność, jego iterator obsługuje P3 (lista kandydatów). Anthropic dostaje jedno wywołanie HTTP per email, z 0–5 kandydatami w prompt'cie, i zwraca strukturalny JSON. Logika decyzji ma trzy źródła prawdy — **wszystkie w Make**.

### B.2. Mapping na 6 problemów

- **A — fragmentacja:** ✅ Make po decyzji wpina email do taska (HTTP do `<id>@mg.clickup.com` lub Add Comment via ClickUp module — Q2 z Fazy 1). Treść trafia na activity feed.
- **B — identyfikacja:** ✅ Search Tasks po Custom Field EMAIL + cache w Data Store (`email → task_ids`).
- **C — matching:** ✅ pełna kontrola: prefiltr deterministyczny (90 dni, nie-closed) + AN-3 z 0–5 kandydatami + router progowy (P1).
- **D — sugestia tytułu:** ✅ osobna krótka HTTP do Anthropic (AN-4) przy tworzeniu/aktualizacji taska, wynik do `SUGEROWANY_TYTUŁ_MAILA`.
- **E — powiadomienia:** ✅ Make po dodaniu emaila wykonuje dodatkowy ClickUp Update / Comment z @mention assignee. ClickUp Inbox + push robi resztę.
- **F — skalowalność:** ✅ Make ops + Anthropic koszty rosną liniowo, ale w skali 100/tydz to <kilkanaście PLN/mc.

### B.3. Kluczowe ryzyka

1. **"Spaghetti scenariusz".** Router + iterator + error handler + nested filtry rosną szybko — przy złożonej polityce dezambiguacji scenariusz Make staje się trudny do utrzymania (MK-5 risk z Fazy 1). Mityguje dyscyplina: każda gałąź izolowana, dokumentacja inline.
2. **Single-shot AI call** ma ograniczoną zdolność do refinement. Jeśli AI mówi "niepewne — daj mi treść taska X szczegółowo", brak mechanizmu drugiego callu. Mityguje: Phase 1 pokazuje, że Haiku radzi sobie w jednym shot'cie, jeśli prompt jest dobrze zbudowany.
3. **Search Tasks po EMAIL** zwraca paginowane wyniki. Przy >100 aktywnych leadów per email (ekstremum) może mieć błędy. Realnie nie spotkamy w EVSO, ale Faza 6 sprawdzi.
4. **Make Data Store nie jest transakcyjny** — przy spike'u równoległych poll'ów Gmail (po outage) idempotentność po `Message-ID` może mieć race conditions. Mityguje: 5-min poll cycle naturalnie serializuje.
5. **`mg.clickup.com` dependency** — gdy się zmieni, ścieżka "Make doprowadza email do taska" wymaga przepisania. Tak samo w kierunku A, ale w B mamy go *jawnie* w jednym module — łatwiej wymienić.

### B.4. Koszt rozwoju

- Forward `kontakt@*` → `automatyzacjaevso@gmail.com` (test) / produkcyjny inbox: **0.5 PD** (Q7).
- Custom Fields M2: **0.5 PD**.
- Make scenariusz "Email Intake M2" (8 modułów wyżej + error handling): **3 PD**.
- Prompty Anthropic (matcher v1.0 + tytuł v1.0): **1 PD**.
- Data Store struktura + idempotency logic: **0.5 PD**.
- Notyfikacje (ClickUp Comment + @mention): **0.5 PD**.
- Test na env testowym + kalibracja P1: **2 PD**.
- **Suma: ~7–8 PD.**

### B.5. Koszt operacyjny

- Make: ~100 emaili/tydz × ~8 ops (Watch+Search+Iterator+HTTP+Router+Update+Notify+DataStore) ≈ **~3200 ops/mc** dla M2 + Phase 1 baseline. Na dzisiejszym planie EVSO `[do potwierdzenia]` mieści się; przy skali 1000/tydz może wymagać up-tier (Q4).
- Anthropic Haiku: ~100/tydz × ~ułamek centa = **kilka USD/mc**. Trywialny.
- ClickUp: w cenie planu (nie używamy AI Agents/AI Fields — minimalne credits).
- **Total est.:** ~30–80 PLN/mc dla 100/tydz; **~100–250 PLN/mc** dla 1000/tydz.

### B.6. Operability

- **Plus:** Bartek już zna Make z Phase 1. Wzorce error handling kopiowalne 1:1. Wersjonowanie promptu Anthropic w prostym tekście (można trzymać w gicie). Audyt decyzji jest trywialny — każdy execution Make ma pełny log inputu i outputu.
- **Minus:** debug "dlaczego AI tak zdecydował" wymaga zaglądania do Make execution history — to nie jest UX dla handlowców. Bartek jest single point of knowledge.
- **Liczba osób, które muszą rozumieć:** 1 (Bartek) operacyjnie, 0 (handlowcy) świadomie. Identycznie jak A.
- **Co się dzieje, gdy Make się zatrzyma:** maile się piętrzą w Gmailu, ale po wznowieniu watcher nadrabia (Phase 1 self-healing). Idempotency by Message-ID chroni przed duplikatami.

---

## Kierunek C — "Hybrid + custom microservice (agentic)"

### C.1. Architektura wysokopoziomowa

```
[Strony brand]                  [Skrzynki kontakt@* ── forward ──►
      │                           automatyzacja@gmail.com]
      ▼                                   │
[Make: Form Intake]              [Make: thin Email Intake]
   (Phase 1)                             │
      │                                  │ Gmail Watch (5 min)
      │                                  │ → POST /emails/inbound
      │                                  ▼
      │              ┌──────────────────────────────────────────┐
      │              │  Mikroservis "EVSO Email Router"         │
      │              │  (Cloudflare Worker, ~300–500 LOC)       │
      │              │                                          │
      │              │  Endpoints:                              │
      │              │   POST /emails/inbound  (od Make)        │
      │              │   GET  /tasks/:email/state               │
      │              │   POST /tasks/:id/title-suggestion       │
      │              │                                          │
      │              │  Stan: Cloudflare KV / D1                │
      │              │   - email_index (email → task_ids)       │
      │              │   - message_ledger (Message-ID seen)     │
      │              │   - decision_log (audit trail)           │
      │              │                                          │
      │              │  Logika (jako kod, nie router Make):     │
      │              │   1. Idempotency check                   │
      │              │   2. Pobierz kandydatów (ClickUp API)    │
      │              │      z prefiltrem P2 (90d, not closed)   │
      │              │   3. Agentyczna pętla z tool use:        │
      │              │      - tool: get_task_summary(id)        │
      │              │      - tool: get_email_history(email)    │
      │              │      - max 3 iteracje                    │
      │              │      - decyzja + confidence + reasoning  │
      │              │   4. Wykonaj akcję (ClickUp API):        │
      │              │      append / create / escalate          │
      │              │   5. Zaloguj do decision_log             │
      │              │   6. Zwróć rezultat → Make → Notify      │
      │              └──────────────────────┬───────────────────┘
      │                                     │
      ▼                                     ▼
              [ClickUp: NOWE ZAPYTANIA / ZLECENIA]
                       (storage + UI)
```

Mikroservis hostuje **politykę** jako kod (TypeScript/JS), a nie deklarację Make. Polityka P1 (progi pewności), P2 (90 dni + AI override), P3 (lista kandydatów) są w jawnych funkcjach z testami jednostkowymi. Anthropic używany przez **agentic loop z tool use** (AN-5): agent może w trakcie decyzji "doczytać" szczegóły taska, jeśli streszczenie nie wystarcza. Make jest okablowaniem (Gmail Watch → POST → Notify), ClickUp jest storage.

### C.2. Mapping na 6 problemów

- **A — fragmentacja:** ✅ identycznie jak B (mikroservis wpina email do taska przez ClickUp API).
- **B — identyfikacja:** ✅ `email_index` w KV — wydajny lookup, bez paginacji ClickUp Search.
- **C — matching:** ✅ agentic loop daje wyższą precyzję na trudnych przypadkach (powracający klient z ambiguous treścią). **Tu kierunek C wygrywa, jeśli realnie potrzebujemy multi-step refine.**
- **D — sugestia tytułu:** ✅ endpoint `/title-suggestion` callable z Make / ClickUp Automation.
- **E — powiadomienia:** 🟡 mikroservis nie powiadamia bezpośrednio — Make po otrzymaniu rezultatu wykonuje ClickUp Comment z @mention. Dodatkowy hop, ale czysta separacja.
- **F — skalowalność:** ✅ Cloudflare Worker free tier obsługuje 10k+ żądań/dzień; agentic loop koszt rośnie 2–3× względem single-shot, ale wciąż w USD/mc.

### C.3. Kluczowe ryzyka

1. **Operability dla małego zespołu (S1).** Mikroservis = nowa technologia w stacku EVSO. Jeśli Bartek wyjeżdża na 2 tygodnie i Worker padnie (deploy expired token, KV quota, Anthropic outage), **żaden handlowiec nie ma kompetencji do diagnozy**. Mityguje: dobry monitoring + runbook + auto-fallback w Make ("jeśli Worker timeout → utwórz task z `KLASYFIKACJA_AI = Do weryfikacji`, bypass matchera"). Ale to dodatkowa złożoność.
2. **Nadmiarowość względem 100/tydz.** Agentic loop z tool use ma sens przy złożoności decyzji, której Faza 2 nie zidentyfikowała jako blokera. Kierunek B z single-shot prompt'em prawdopodobnie wystarczy. Ryzyko **over-engineering**.
3. **Dwa miejsca prawdy** dla state'u: Cloudflare KV i ClickUp. Drift jest realny (np. Worker zaktualizuje KV, ale ClickUp API call padnie — KV mówi "matched", ClickUp nie). Mityguje: write-through pattern + reconciliation job.
4. **Deploy + secrets management.** Wrangler / Vercel CLI, sekrety Anthropic + ClickUp API key, environment variables — nowa powierzchnia operacyjna. Dla Bartka to znajome (wykonalne), dla zespołu nie.
5. **Lock-in na własny kod.** Najmniejszy zewnętrzny lock-in (Cloudflare Worker = open standard, łatwo migrować), ale każda zmiana logiki wymaga release cycle (kod + deploy), nie kliku w UI Make.

### C.4. Koszt rozwoju

- Setup Cloudflare Worker + KV + secrets + CI deploy: **1 PD**.
- Endpoint `/emails/inbound` + idempotency + ClickUp client: **2 PD**.
- Agentic loop (Anthropic tool use, max 3 iteracje, retry, error handling): **3 PD**.
- Endpoint `/title-suggestion`: **0.5 PD**.
- Make scenariusz "thin email intake" (Watch + POST + Notify): **1 PD**.
- Custom Fields ClickUp: **0.5 PD**.
- Testy jednostkowe polityki + integracyjne (env testowe): **3 PD**.
- Runbook + monitoring + fallback w Make: **1.5 PD**.
- **Suma: ~12–13 PD** (~2× Kierunek B).

### C.5. Koszt operacyjny

- Cloudflare Worker + KV: **0 PLN/mc** dla skali EVSO (free tier).
- Anthropic Haiku: ~100/tydz × ~3 calls per email (agentic loop) × ułamek centa = **kilkanaście USD/mc** (3× względem B). Wciąż tania.
- Make: niższe ops niż B (cieńszy scenariusz) — **~1500 ops/mc**.
- ClickUp: bez AI Agents/Fields — minimalne credits.
- **Total est.:** ~50–100 PLN/mc dla 100/tydz; **~150–400 PLN/mc** dla 1000/tydz (głównie Anthropic + Make).

### C.6. Operability

- **Plus:** logika polityki w kodzie z testami — najlepsza precyzja i powtarzalność. Audyt decyzji w `decision_log` — łatwy do query. Wersjonowanie przez gita.
- **Minus:** **najgorszy single-bus-factor**. Bartek = single point of expertise. Handlowcy nie zdiagnozują niczego głębiej niż "system nie działa, dzwoń do Bartka". Bez ról on-call EVSO to realny problem.
- **Liczba osób, które muszą rozumieć:** 1 (Bartek) głęboko, plus opcjonalnie zewnętrzny dev na fallback. To jest jakościowo gorsze niż A/B, gdzie "rozumieć" znaczy "umieć kliknąć w UI Make/ClickUp".
- **Co się dzieje, gdy Worker padnie:** Make fallback tworzy taski z `Do weryfikacji` — system **degraduje do manualnego matchingu**, ale nie traci maili. Self-healing po przywróceniu Workera (Make retry queue + ledger).

---

## Porównanie syntetyczne (jednym spojrzeniem)

| Wymiar | A — ClickUp-native | B — Make-watcher | C — Microservice agentic |
|---|---|---|---|
| Lokalizacja decyzji | ClickUp AI Agent | Make router + Anthropic single-shot | Cloudflare Worker + Anthropic agentic loop |
| Główny lock-in | ClickUp (Business→Enterprise ryzyko) | Make (znajome z Phase 1) | Własny kod (najmniejszy ext.) |
| Koszt rozwoju (PD) | ~6–7 | ~7–8 | ~12–13 |
| Koszt op. (100/tydz) | kilkanaście PLN+ ryzyko over-cap AI uses | ~30–80 PLN/mc | ~50–100 PLN/mc |
| Koszt op. (1000/tydz) | ryzyko Enterprise upgrade | ~100–250 PLN/mc | ~150–400 PLN/mc |
| Audytowalność decyzji AI | słaba (UI Agent ClickUp) | dobra (Make execution log) | bardzo dobra (decision_log + git) |
| Operability dla zespołu | dobra UX, debug trudny | znajomy idiom (Phase 1) | wymaga dewelopera |
| Spójność z Phase 1 | nowa ścieżka obok Phase 1 | bezpośrednie rozszerzenie wzorca Phase 1 | dodatkowa warstwa nad Phase 1 |
| Ryzyko over-engineeringu | niskie | niskie | **realne** (jeśli single-shot wystarczy) |
| Ryzyko under-engineeringu | średnie (Agent może nie utrzymać polityki P2 spójnie) | niskie | bardzo niskie |
| Najlepszy w… | gdy chcemy "all-in-one" UX i ClickUp Agents są stabilną technologią z planu Business | gdy P1–P3 mieszczą się w jednym promptcie + routerze (najbardziej prawdopodobne) | gdy realnie potrzebujemy multi-step refine i mamy budżet operacyjny na utrzymanie kodu |

---

## Co Faza 4 (panel ekspertów) ma rozstrzygnąć

Panel ma cross-examinować po 4 perspektywach (ClickUp / Make / AI / Sales process) i odpowiedzieć:

1. **Czy ClickUp Agents są wystarczająco dojrzałe** dla produkcyjnego użycia w planie Business? (Marta) → wpływ na A.
2. **Czy single-shot prompt z 0–5 kandydatami realnie pokrywa P3** (powracający klient z ambiguous treścią)? (Aleksander) → wpływ B vs C.
3. **Gdzie scenariusz Make w B się łamie przy skali / błędach** i jaki jest plan degradacji? (Tomasz) → wpływ na B.
4. **Który kierunek faktycznie ułatwia handlowcowi dzień**, biorąc pod uwagę gdzie żyje "Do weryfikacji" i jak działa rytuał czyszczenia kolejki? (Karolina) → wpływ na wszystkie trzy, ale szczególnie na A (UX wewnątrz ClickUp) vs B/C (UX z dwóch ekranów).

Kierunki są jawnie napięte: A wygrywa na *operability dla zespołu* i UX, B wygrywa na *spójności z Phase 1 i pragmatyce*, C wygrywa na *audytowalności i precyzji*. Żaden nie dominuje pozostałych po wszystkich osiach — to celowe, panel ma pracować, nie klepnąć oczywistość.

---

## Kryterium gotowości Fazy 3 (self-check)

- [x] 3 kierunki zaszkicowane.
- [x] Kierunki są **zasadniczo różne** (nie warianty siebie) — udokumentowane w sekcji 0 po trzech osiach: lokalizacja mózgu, model stanu, tryb decyzji + dwie pomocnicze (operability, lock-in).
- [x] Każdy kierunek ma 6 wymaganych sekcji: architektura, mapping na 6 problemów, ryzyka, koszt rozwoju, koszt operacyjny, operability.
- [x] Żaden kierunek nie jest "słomianym ludkiem" — każdy ma scenariusz, w którym wygrywa (sekcja porównania syntetycznego, ostatni wiersz "Najlepszy w…").
- [x] Brak referencji do `archive/*`.

**Status: spełnione.**
