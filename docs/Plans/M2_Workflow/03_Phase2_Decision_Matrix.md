# Faza 2 — Decision Matrix (Milestone 2)

> **Status:** wynik Fazy 2 workflow M2.
> **Anti-bias ack:** wczytałem `docs/Plans/M2_Workflow/00_Workflow_Plan.md`, `docs/Plans/M2_Workflow/01_Phase0_Problem_Map.md`, `docs/Plans/M2_Workflow/02_Phase1_Capability_Map.md`. **NIE czytałem** żadnych plików z `archive/*`.
> **Cel pliku:** świadomie wybrać poziom ambicji automatyzacji dla każdego z problemów A–F **zanim** zaczniemy projektować *jak* (Fazy 3+). Plus rozstrzygnąć 4 polityki, które przekrojowo wpływają na architekturę. Bez tej fazy ryzykujemy automatyzację rzeczy, która powinna zostać manualna, albo manual rzeczy, którą dało się zautomatyzować tanio.

---

## 1. Skala poziomów ambicji (definicje, których używamy)

| Poziom | Definicja operacyjna | Kiedy wybieramy |
|---|---|---|
| **Full auto** | System wykonuje akcję bez interwencji człowieka, z wysoką pewnością. Pracownik widzi wynik post factum. | Gdy koszt błędu jest niski LUB gdy AI/logika ma confidence > próg auto. |
| **Auto + escalate** | System próbuje sam, ale przy niskiej pewności wrzuca do kolejki "do weryfikacji" / `KLASYFIKACJA_AI = Do weryfikacji`. Pracownik domyka manual. | Gdy koszt błędu jest średni — większość wolumenu chcemy zautomatyzować, ale nie wolno cicho zgadywać. **Domyślny poziom dla decyzji AI w M2** zgodnie z Phase 1 baseline. |
| **Manual + assist** | Pracownik wykonuje akcję sam; system mu pomaga (sugeruje task do dopasowania, draft tytułu, wskazuje kandydatów). | Gdy automatyzacja nie da się zrobić wystarczająco pewnie ALBO gdy ergonomicznie i tak handlowiec musi kliknąć (np. wysyłka maila wychodzącego). |
| **Out of scope** | Świadomie nie ruszamy w M2. | Gdy problem jest poza granicami M2 (telefon, K2) lub gdy ROI w skali EVSO nie uzasadnia kosztu. |

Reguła zespołu (S2 z Fazy 0): **automatyzujemy tylko jeśli prostsze, stabilniejsze albo faktycznie bardziej użyteczne** niż alternatywa. Każda decyzja "Full auto" musi się obronić wobec tej reguły.

---

## 2. Decision Matrix dla problemów A–F

### Problem A — Fragmentacja historii konwersacji

**Decyzja: Full auto.**

**Uzasadnienie.** Sama agregacja maili w jedno miejsce (task ClickUp) nie wymaga decyzji AI — to deterministyczne *doprowadzenie* maila do taska, gdy matching zwrócił trafienie (Problem C). Akcja "dodaj treść maila do activity feed właściwego taska" nie ma sensownej wersji "manual" — pracownik miałby wklejać maila ręcznie, co jest dokładnie tym, co M2 ma wyeliminować (Problem F). Phase 1 baseline pokazuje, że Make → ClickUp Update/Comment jest stabilną ścieżką (CU-9, MK-4).

**Konsekwencja, którą akceptujemy.** Jeśli matching (Problem C) zwróci błędne dopasowanie z wysoką pewnością, treść maila trafi do złego taska — czyli błąd Problemu C kaskaduje się w Problem A. Mityguje to fakt, że przy `Auto + escalate` w Problemie C niska pewność idzie do kolejki, a nie do złego taska. Akceptujemy też, że ekstrakcja treści maila do feedu może gubić formatowanie HTML (Problem A na poziomie *agregacji*, nie *prezentacji*).

---

### Problem B — Identyfikacja klienta ponad kanałami (email-as-identity)

**Decyzja: Full auto** (sama identyfikacja po kluczu `email` jest deterministyczna; *rozstrzyganie który task* przy wielu otwartych = Problem C).

**Uzasadnienie.** Świadome uproszczenie z Fazy 0 (K1, H4): tożsamość klienta = adres email. To jest lookup deterministyczny — `email → list of task_ids` (Make Search Tasks po Custom Field EMAIL, ewentualnie cache w Data Store MK-3). Nie ma tu decyzji AI do podejmowania na poziomie "czy to ten sam człowiek". AI wchodzi dopiero przy *rozstrzyganiu wśród kandydatów* (Problem C) i przy polityce powracającego klienta (sekcja 3, polityka P2).

**Konsekwencja, którą akceptujemy.** Klient, który zmienił adres email (`jan@firma.pl` → `jan@gmail.com`), trafi jako nowy klient (kompromis K1). Manual cleanup po stronie handlowca, jeśli to wychwyci. Dla skali EVSO realna częstotliwość: kilka procent leadów — akceptowalne.

---

### Problem C — Niepełna integracja emaili z CRM (matching email ↔ task)

**Decyzja: Auto + escalate.**

**Uzasadnienie.** To jest *centralna decyzja AI w M2*. Nie można pójść Full auto — confidence Anthropic jest self-reported (AN-1, AN-3), brak kalibracji statystycznej, a koszt błędu jest realny: email "potwierdzam czwartek" wpięty do złego taska to utrata leadu. Manual + assist z kolei zabija ekonomikę M2 (Problem F: 100/tydz × ~2 min ręcznego matchingu = 3.3h tygodniowo). `Auto + escalate` z konkretnym progiem pewności (polityka P1) jest jedynym poziomem, który **i obniża obciążenie**, **i nie tworzy cichych błędów** — dziedziczy wzorzec Phase 1 (`KLASYFIKACJA_AI = Do weryfikacji` przy fallbacku).

**Konsekwencja, którą akceptujemy.** Ok. 10–25% maili (estymacja, do skalibrowania w testowym env) wpadnie do kolejki "Do weryfikacji" i będzie wymagać kliknięcia handlowca. To jest świadomy koszt — w zamian unikamy sytuacji, gdzie AI "wybiera" zły task z wysoką pewnością. Rytuał czyszczenia kolejki "Do weryfikacji" musi być wbudowany w dzień handlowca (design w Fazie 5 / runbook w Fazie 7).

---

### Problem D — Sugerowanie tytułu wątków wychodzących

**Decyzja: Manual + assist.**

**Uzasadnienie.** Akcja "wyślij maila wychodzącego" leży po stronie handlowca — pisze treść, weryfikuje załączniki, klika *Send* w composerze ClickUp (CU-9). Zautomatyzowanie *tytułu* (np. wpisanie z palca przez Make do composera) wymagałoby Chrome extension lub niestandardowej integracji nad ClickUp UI — disqualifikowane przez S2. Realistyczny pattern: AN-4 generuje **sugerowany tytuł** raz, przy tworzeniu/aktualizacji taska, i zapisuje do nowego Custom Field `SUGEROWANY_TYTUŁ_MAILA`. Handlowiec widzi sugestię w tasku i może ją skopiować jednym kliknięciem do composera.

**Konsekwencja, którą akceptujemy.** Handlowiec wciąż wykonuje copy-paste — nie jest to zero-click. Akceptujemy, bo redukuje to "tytuł z palca / niespójność między handlowcami" (cel Problemu D), bez wprowadzania custom UI lub Chrome extension. Sugestia może się "starzeć" względem stanu taska — strategia odświeżania (regeneracja przy zmianie statusu / pól kluczowych) zostaje do Fazy 5.

---

### Problem E — Brak powiadamiania o nowych wiadomościach

**Decyzja: Full auto** (deterministyczne powiadomienie, gdy email trafia do taska).

**Uzasadnienie.** To nie jest decyzja AI — to deterministyczny routing: "email wpadł do taska → @mention assignee w komentarzu / utwórz watcher / push przez ClickUp Inbox". Dwie ścieżki realizacji są dostępne natywnie: (a) Make po dodaniu emaila wykonuje dodatkowy Update/Comment z @mention; (b) ClickUp Automation (CU-6) na trigger "comment added" → notify assignee. Anti-assumption #5 z Fazy 0 (handlowiec nie patrzy w ClickUp częściej niż 2–3× dziennie) wymaga, żeby ten kawałek był aktywny — pasywne czekanie aż ktoś odświeży zabija M2.

**Konsekwencja, którą akceptujemy.** Push mobile bywa cichy (CU-8 limity). Akceptujemy notyfikację natywną ClickUp jako baseline; bridge na Slack/Telegram zostaje jako *opcja w Fazie 5*, nie obowiązek M2. Każdy email trafiający do taska generuje notyfikację — przy spike'u maili od jednego klienta może to wyglądać jak spam, ale to lepszy default niż "cisza". Tłumienie / agregacja powiadomień to sprawa M3+.

---

### Problem F — Skalowalność ręcznego zarządzania

**Decyzja: Full auto** (jako *konsekwencja* automatyzacji Problemów A, B, C, E).

**Uzasadnienie.** Problem F nie ma własnej akcji do wykonania — jest to *miara*, że pozostałe automatyzacje faktycznie ujmują pracy ręcznej. Każda minuta zaoszczędzona w A/B/C/E = bezpośrednia ulga dla F. Jeśli A/B/C/E są na poziomach Full auto / Auto+escalate, to F jest automatycznie spełniony do tego stopnia, w jakim spełnione są tamte. Nie ma sensu projektować osobnej "automatyzacji F".

**Konsekwencja, którą akceptujemy.** F nie ma pojedynczego "fix" — jest pochodną sumarycznej skuteczności A–E. Jeśli np. C ma 25% kolejki escalation, F dostaje 75% redukcji ręcznej pracy w tej części pipeline'u, a nie 100%. Akceptujemy to jako empiryczną metrykę M2, mierzoną *po* uruchomieniu (success criterion #5 z Problem Definition: duplikaty, czas obsługi).

---

### Tabela podsumowująca

| Problem | Poziom ambicji | Kluczowy mechanizm |
|---|---|---|
| A — Fragmentacja historii | **Full auto** | Make doprowadza email do feedu właściwego taska |
| B — Identyfikacja klienta (email-as-identity) | **Full auto** | Lookup deterministyczny po Custom Field EMAIL |
| C — Matching email ↔ task | **Auto + escalate** | AI confidence z progami → auto / kolejka / nowy task |
| D — Sugestia tytułu wychodzącego | **Manual + assist** | AN-4 generuje `SUGEROWANY_TYTUŁ_MAILA`, handlowiec kopiuje |
| E — Powiadamianie pracownika | **Full auto** | @mention assignee + native ClickUp push |
| F — Skalowalność | **Full auto** (pochodna A–E) | Brak własnego mechanizmu — mierzona empirycznie |

**Out of scope w M2** (powtórzenie z Fazy 0, dla porządku): telefon (K2), łączenie tożsamości po telefonie/imieniu (K3), chat / social media / SMS / WhatsApp (K7).

---

## 3. Cztery polityki przekrojowe

Te decyzje przecinają się przez wszystkie problemy i rozstrzygamy je raz, żeby Fazy 3+ miały kotwicę.

### Polityka P1 — Próg pewności AI dla matchingu emaili (Problem C)

**Decyzja:** trzy progi z Phase 1-style enum w `MATCHING_CONFIDENCE`:
- **≥ 0.85** → **auto-match**: email dołączany do wskazanego taska bez interwencji.
- **0.60 – 0.84** → **escalate**: nowy task z `KLASYFIKACJA_AI = Do weryfikacji`, w description AI proponuje "być może ten sam co LEAD-XXXX", handlowiec rozstrzyga jednym kliknięciem (link do kandydata).
- **< 0.60** → **nowy task** standardowo, bez sugestii dopasowania.

**Uzasadnienie.** Self-reported confidence Haiku (AN-1, AN-3) nie jest matematycznie kalibrowane, ale empirycznie u Anthropic na zadaniach klasyfikacyjnych próg ~0.85 daje rozsądny trade-off między precision (mało false positive) a recall (mało eskalacji). 0.85/0.60 jako baseline + walidacja w testowym env (`automatyzacjaevso@gmail.com` + Forminator klon, 50–100 testowych przypadków) — jeśli okaże się, że auto-match przepuszcza błędy, podnosimy do 0.90; jeśli kolejka eskalacji przerasta handlowca, obniżamy 0.85 → 0.80. **Progi to zmienna kalibracyjna, nie ustalenie na zawsze** — decyzja jest na *baseline*, nie na *finalną wartość*.

**Konsekwencja.** Faza 5 musi przewidzieć, że progi są łatwo zmienialne (np. konfigurowalne w Make Data Store / module variables, nie hardcode w prompcie). Faza 6 (Adversarial) sprawdzi case "AI zwraca 0.95 ale myli się" — wtedy mityguje to ścieżka audytu (każda auto-match decyzja loguje się z confidence + reasoning do Custom Field, handlowiec może retroaktywnie zauważyć).

---

### Polityka P2 — Powracający klienci (okno czasu, kiedy "ten sam email" = nowy task)

**Decyzja:** **dwustopniowa, AI-driven z deterministycznym fallbackiem.**

1. **Filtr deterministyczny (pre-AI):** kandydatami do dopasowania są wyłącznie taski z tym samym EMAIL **w dowolnym statusie poza "archiwum/closed"**, w które ostatnia interakcja (`LAST_EMAIL_AT` lub `data_modyfikacji`) miała miejsce **w ciągu ostatnich 90 dni**. Taski starsze niż 90 dni od ostatniej interakcji **lub w statusie zamkniętym** — nie są kandydatami. Nowy email od takiego klienta = nowy task.
2. **Decyzja AI (na kandydatach z punktu 1):** AI dostaje 1–N kandydatów + treść/temat nowego maila i decyduje "kontynuacja sprawy / nowy wyjazd / niepewne". Nawet jeśli kandydat istnieje w oknie 90 dni, AI może powiedzieć "to jest nowy wyjazd" → wtedy tworzymy nowy task (z linkiem "powiązany z LEAD-XXXX" w description, ale jako *osobny task*).

**Uzasadnienie.** Sztywne okno (np. "90 dni i zawsze nowy task") jest zbyt prymitywne — przykład Pani Kasi z Fazy 0 (urlop + urodziny) pokazuje, że nawet w tym samym tygodniu mogą być dwa osobne wyjazdy tego samego klienta. Okno 90 dni jest *prefiltrem*, nie *decyzją końcową*: ogranicza koszt promptu (mała liczba kandydatów, AN-3 limit "3–5 kandydatów"), ale finalne rozstrzygnięcie należy do AI (semantyka: temat, treść, daty wydarzenia). Status "archiwum/closed" automatycznie wyklucza taski domknięte — zgodne z K6 (lead żyje wiecznie, ale closed = nie wracamy).

**Konsekwencja.** Klient, który **w 91. dniu** od ostatniej interakcji pisze kontynuację tej samej sprawy (rzadkie, ale realne), trafi jako nowy task. Manual cleanup. Akceptujemy. Strategia kalibracji okna (90 dni → 60 lub 120) — decyduje empiria po pierwszych miesiącach.

---

### Polityka P3 — Duplikaty (jeden klient, wiele otwartych leadów jednocześnie)

**Decyzja:** **dopuszczamy wiele otwartych tasków per email, AI rozstrzyga do którego trafia każdy nowy email; przy niepewności — escalate.**

Konkretnie:
- Lista kandydatów (po prefiltrze P2) może mieć **>1** task. To jest legitne (Pani Kasia: wieczór panieński czerwiec + urodziny grudzień).
- AN-3 dostaje wszystkich kandydatów z ich `AI_PODSUMOWANIE` (z Phase 1) + treść/temat nowego maila. Wybiera jeden task **z confidence ≥ próg P1** lub eskaluje (zgodnie z P1).
- **Nie scalamy automatycznie duplikatów.** Jeśli powstały dwa taski na tę samą sprawę (np. handlowiec ręcznie utworzył jeden, a Phase 1 utworzył drugi z formularza) — to manual cleanup po stronie handlowca, nie automatyka M2. Power M2 jest w *kierowaniu emaili do właściwego taska*, nie w *czyszczeniu istniejącego stanu*.

**Uzasadnienie.** Auto-merge dwóch tasków to operacja wysokiego ryzyka (utrata historii, błędne łączenie różnych spraw) — łamie regułę S2. Zostawiamy to po stronie człowieka. Jednocześnie, dopuszczanie wielu otwartych leadów per email jest realnością biznesu EVSO (klienci-firmy organizujące cykle imprez) — więc system **musi** umieć po nich nawigować w matcherze.

**Konsekwencja.** Custom Field `EMAIL` w ClickUp **nie jest unique** — mogą istnieć dwa otwarte taski z tą samą wartością. Search Tasks w Make (MK-4) musi to obsłużyć (zwraca listę, iterator po kandydatach przed AI). Faza 5 musi też zaprojektować widok "wszystkie otwarte taski klienta X" dla handlowca (np. ClickUp filtered view / komentarz w nowym tasku z linkami do innych otwartych dla tego samego emaila).

---

### Polityka P4 — Real-time vs batch

**Decyzja:** **Gmail Watch co 5 minut (MK-2 default) jako baseline M2; nie wprowadzamy push przez Pub/Sub w M2.**

**Uzasadnienie.** Anti-assumption #5 z Fazy 0 (handlowiec nie patrzy w ClickUp częściej niż 2–3× dziennie) wskazuje, że *real-time <30s* nie daje proporcjonalnej wartości biznesowej dla EVSO. Phase 1 baseline (Email Intake, scheduled) działa stabilnie. VC-4 (Pub/Sub bridge) wprowadza nową technologię (service account, Pub/Sub, mikroservis) bez jasnego ROI dla skali ~100/tydz — łamie regułę S2. Webhook trigger (MK-1) dla **formularzy** zostaje instant (Phase 1) — to dotyczy Form Intake, nie maili.

**Konsekwencja.** Maksymalna latencja od wysłania maila przez klienta do utworzenia/aktualizacji taska + notyfikacji handlowca: **~5–7 minut** (5 min poll + 1–2 min na AI + ClickUp update). Akceptujemy. Jeśli w przyszłości SLA EVSO wymaga <1 min (np. konkurencja zaczyna odpowiadać szybciej), wracamy do VC-4 jako *upgrade* — architektura w Fazie 5 musi to umożliwić bez przebudowy (warstwa watchera wymienna).

---

## 4. Implikacje dla Faz 3+ (co z tego wynika)

1. **Architektura M2 musi mieć moduł "AI matcher"** — nazwany komponent, który dostaje `(new_email, candidate_tasks)` i zwraca `{decision, target_task_id|null, confidence, reasoning}`. To miejsce, w którym P1+P2+P3 się fizycznie spotykają. Faza 3 zarysuje 3 kierunki, gdzie ten moduł żyje (Make / ClickUp Agent / mikroservis).
2. **Każda decyzja AI musi być audytowalna.** Custom Field `MATCHING_CONFIDENCE` (Number) + `MATCHING_REASONING` (Long Text) + `MATCHING_DECISION` (Dropdown: `auto-match` / `escalated` / `new`) na każdym tasku, którego dotyczy email z M2. Bez tego nie da się skalibrować P1.
3. **Kolejka "Do weryfikacji" rozszerza się** — dziedziczy wzorzec Phase 1 (`KLASYFIKACJA_AI = Do weryfikacji`), ale potrzebuje jasnego sygnału "ten task jest *nowym* taskiem z propozycją dopasowania do LEAD-XXXX" (różne źródło niż "AI nie umie sklasyfikować typu zapytania"). Faza 5: rozważyć osobny status / pole.
4. **Custom Fields do dodania w M2** (lista wstępna, do potwierdzenia w Fazie 5):
   - `LAST_EMAIL_AT` (Date) — do prefiltrów P2.
   - `MATCHING_CONFIDENCE` (Number) + `MATCHING_REASONING` (Long Text) + `MATCHING_DECISION` (Dropdown).
   - `SUGEROWANY_TYTUŁ_MAILA` (Short Text) — Problem D.
   - Wszystkie **additive** zgodnie z H2.
5. **Idempotentność jest Q8 z Fazy 1 — pozostaje otwarta na Fazę 5,** ale P4 (poll co 5 min) sugeruje silnie Data Store z `Message-ID` jako preferowaną strategię (vs. destruktywne "mark as read"), bo poll 5-min jest tolerujący na re-fetchy.

---

## 5. Kryterium gotowości Fazy 2 (self-check)

- [x] 6 wierszy (A–F), każdy z jedną decyzją na skali poziomów ambicji — sekcja 2 + tabela podsumowująca.
- [x] 4 polityki rozstrzygnięte: **P1** (próg pewności AI), **P2** (okno powracających klientów), **P3** (duplikaty), **P4** (real-time vs batch) — sekcja 3.
- [x] Każda decyzja ma uzasadnienie ≥2 zdania + jawnie zapisaną konsekwencję ("przy tym poziomie godzimy się że…").
- [x] Brak referencji do `archive/*`.

**Status:** spełnione.
