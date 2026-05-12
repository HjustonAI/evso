# Faza 4 — Expert Panel Stress Test (Milestone 2)

> **Status:** wynik Fazy 4 workflow M2.
> **Anti-bias ack:** wczytałem `00_Workflow_Plan.md`, `04_Phase3_Divergent_Directions.md`, `docs/Teams/EVSO_DevTeam.md`. Korzystam pośrednio z ustaleń Faz 0–2 (constraints, polityki P1–P4) cytowanych w Fazie 3. **NIE czytałem** żadnych plików z `archive/*`.
> **Format spotkania:** Decision Session (3 zdefiniowane opcje — kierunki A/B/C z Fazy 3; wynik = wybór + decision record).
> **Reguła zespołu (z Charter):** propozycja przechodzi tylko jeśli jest **prostsza, stabilniejsza albo realnie bardziej użyteczna** niż alternatywa.

---

## 0. Meeting Brief

| | |
|---|---|
| **Goal as stated** | "Stress test 3 kierunków M2 (A/B/C) panelem 4 ekspertów. Wybór kierunku + decision record + lista nie-konsensusu." |
| **Goal as understood** | Wybrać kierunek (lub jawnie zdefiniowaną hybrydę) realizujący 6 problemów A–F, polityki P1–P4 i constraints H1–H8 z minimalnym ryzykiem regresji wobec Phase 1. |
| **What "done" looks like** | Ranking każdego z 4 ekspertów × 3 kierunki + uzasadnienie pozycji + Decision Record (decision, rationale, dissent, conditions for revisiting, confidence) + lista nie-konsensusu (≥3 pkt) + lista zaakceptowanych kompromisów. |
| **Key dimensions → eksperci** | Lokalizacja mózgu/lock-in → Marta + Tomasz; tryb decyzji AI / audytowalność → Aleksander; ergonomia handlowca / kolejka "Do weryfikacji" → Karolina. |
| **Known context** | Faza 0 (problem map, K1–K7, anti-assumptions), Faza 1 (capability map, otwarte pytania Q1–Q8), Faza 2 (decyzje per problem A–F + polityki P1–P4), Faza 3 (3 kierunki z risk + cost + operability). |
| **Missing context** | Q1 (ClickUp Agents w planie Business — limity), Q2 (Make ClickUp module: comment-as-email vs SMTP), Q3/Q4 (twarde limity AI uses i Make ops na obecnym planie), Q7 (forward Dhosting → inbox), Q8 (idempotentność: mark-as-read vs Data Store). Q1 jest blocker tylko dla Kierunku A — w panelu pracujemy z założeniem "Q1 nieznane" i traktujemy je jako risk. |

**Tryb:** direct (bez gate'u clarification — Faza 0–3 dostarczyły wystarczająco kontekstu na working depth; otwarte pytania trafiają do uncertainty register).

---

## 1. Aktywacja ekspertów

| Ekspert | Rola w spotkaniu | Uzasadnienie |
|---|---|---|
| **Marta Kurek** (ClickUp Systems Architect) | Primary | Kierunek A żyje w jej domenie; w B/C ClickUp jest storage + UI — ergonomia handlowca i model danych są jej. |
| **Tomasz Sowa** (Make Automation Architect) | Primary | Make jest centralny w B i okablowaniem w C; on ocenia kruchość/stabilność scenariuszy i error handling. |
| **Aleksander Bryła** (AI Systems & Reliability) | Primary | Polityka P1 (progi pewności), audyt decyzji, single-shot vs agentic loop — jego rdzeń. |
| **Karolina Wysocka** (Sales & Service Process Designer) | Primary | Kryterium "który kierunek faktycznie ułatwia handlowcowi dzień" + design rytuału kolejki "Do weryfikacji". |

Wszyscy primary — agenda dotyka każdego rdzenia. Brak observerów.

---

## 2. Discussion — Runda 1: pierwsze reakcje na 3 kierunki

### Marta Kurek

[FACT] Z Fazy 1 (CU-3): "Bartek nie ma jeszcze produkcyjnego doświadczenia z ClickUp Agents w Phase 1. Wprowadzenie Agent jako warstwy decyzyjnej M2 oznacza nową technologię w stacku." [FACT] Q1 z Fazy 1 jest oznaczone jako "High criticality" dla Kierunku A.

[INFERENCE] Kierunek A ma estetycznie najlepsze "wszystko-w-jednym", ale opiera centralną zdolność matchera na technologii, której nie używamy nigdzie indziej w EVSO. To jest dokładnie scenariusz, przed którym charter ostrzega: nowa technologia przyjmowana, *zanim* udowodni, że jest stabilniejsza lub bardziej użyteczna od alternatywy. [RECOMMENDATION] A jako baseline odrzucam — można wrócić do *elementów* A (np. AI Field na `SUGEROWANY_TYTUŁ_MAILA` — to jest tania, lokalna integracja).

[FACT] W B i C ClickUp jest pasywnym storage + UI. Z perspektywy ergonomii handlowca to jest ten sam UX — handlowiec widzi task z activity feed, "Do weryfikacji" jako filtered view, push z @mention. **Różnica między B i C dla handlowca jest niewidoczna.** [INFERENCE] Decyzja B vs C jest decyzją Bartka jako operatora systemu, nie handlowca jako użytkownika.

[RECOMMENDATION] Custom Fields nowe (P1: `MATCHING_CONFIDENCE`, `MATCHING_DECISION`, `MATCHING_REASONING`, `LAST_EMAIL_AT`, `SUGEROWANY_TYTUŁ_MAILA`) — wszystkie additive na liście `NOWE ZAPYTANIA` (zgodnie z H2/H3). Dodatkowo postuluję **dropdown `MATCHING_DECISION` z konkretnymi enum** (`auto-match` / `escalated-need-confirm` / `new-task` / `linked-related`) — żeby filtered view "Do weryfikacji" miał jednoznaczne źródło prawdy.

### Tomasz Sowa

[FACT] Phase 1 baseline w Make działa stabilnie z wzorcem: HTTP → Anthropic → parse JSON → router → ClickUp + error handler "Resume" z fallbackiem `Do weryfikacji`. To jest **udowodniony wzorzec produkcyjny** w EVSO.

[INFERENCE] Kierunek B jest bezpośrednim rozszerzeniem tego wzorca o trzy elementy: (1) Gmail Watch zamiast Webhook, (2) Search Tasks po EMAIL przed call'em do Anthropic, (3) router 3-gałęziowy zamiast 2 (auto / escalate / new). To jest **inkrementalny krok** od Phase 1, nie jakościowy skok. Ryzyko wykonawcze jest najniższe ze wszystkich trzech.

[FACT] Z Fazy 3 (B.3 #1): "spaghetti scenariusz" jest wymieniony jako risk Kierunku B. [INFERENCE] To jest realny, ale **dyscyplinowalny** risk — ja kontroluję go na trzech poziomach: (a) każda gałąź routera jako osobny "blok" z komentarzami inline, (b) izolacja error handling per moduł krytyczny (HTTP Anthropic, ClickUp Update, Data Store), (c) blueprint export do gita po każdej istotnej zmianie (CLAUDE.md mówi explicit: blueprint = autorytatywny stan). Przy 100/tydz to są ~3–5 modułów dodanych względem Phase 1, nie 30.

[INFERENCE] Kierunek C wprowadza w Make **mniej** modułów (bo logika jest w Workerze), ale dodaje **nową powierzchnię** poza Make: deploy pipeline, secrets, monitoring Workera, reconciliation między KV a ClickUp. Z mojej perspektywy operatora Make to jest *przeniesienie złożoności*, nie jej redukcja. Złożoność tylko zmienia adres.

[RECOMMENDATION] Niezależnie od wyboru kierunku — **idempotentność po `Message-ID` w Make Data Store** jako baseline (Q8 z Fazy 1, P4 z Fazy 2 sugeruje to silnie). "Mark as read" w Gmailu jako destruktywny side effect odrzucamy.

### Aleksander Bryła

[FACT] Z Fazy 1 (AN-1): "Confidence jest **self-reported** przez model — nie jest matematycznie kalibrowane." Z Fazy 2 (P1): progi 0.85/0.60 są "baseline, nie finalna wartość" — wymagają walidacji empirycznej.

[INFERENCE] Pytanie kierunkowe nie brzmi "Make vs Worker", tylko: **czy single-shot prompt z 0–5 kandydatami da decyzję na poziomie produkcyjnym dla zadania matchera EVSO?** Jeśli tak — Kierunek B jest wystarczający. Jeśli nie — potrzebujemy refinement (Kierunek C z agentic loop). Bez danych empirycznych odpowiadam: **tak, wystarcza**, z dwoma warunkami.

[INFERENCE] Warunek 1 — *prompt*. System prompt musi zawierać: definicję zadania, format JSON output (`{decision, target_task_id, confidence, reasoning}`), 2–3 few-shot przykłady (kontynuacja sprawy / nowy wyjazd od starego klienta / ambiguous → escalate), oraz explicit instrukcję "jeśli niepewny, escalate, nie zgaduj". Phase 1 prompt jest minimalistyczny — dla matchera M2 musi być rozbudowany. Najpierw reguła, potem model.

[INFERENCE] Warunek 2 — *audyt*. Każda decyzja AI musi mieć zapisane: input (treść maila + kandydaci z ich `AI_PODSUMOWANIE` z Phase 1), output (full JSON), wersję promptu (`v1.0`), timestamp. W Kierunku B zapisuję to jako Custom Fields na tasku (`MATCHING_REASONING`, `MATCHING_CONFIDENCE`) plus Make execution log. W Kierunku C miałbym `decision_log` w D1/KV — czystsze. **Ale to nie jest wystarczająca przewaga, żeby uzasadnić Workera.** Make execution history z 30-dniowym retentionem + Custom Fields per task = wystarczający audyt dla 100/tydz.

[FLAG / non-consensus] Mam **wątpliwości co do progu 0.85** z Fazy 2. Self-reported confidence Haiku jest miękki — empirycznie z innych projektów widać, że model deklaruje 0.9+ na decyzjach, które po review są błędne. **Postuluję start od 0.92 dla auto-match**, kalibrację w dół po 50 testowych przypadkach. Faza 2 tego nie rozstrzygnęła ostatecznie ("baseline, nie finalna") — chcę to wpisać jawnie.

[RECOMMENDATION] Niezależnie od kierunku: **first 100 produkcyjnych decyzji to "shadow mode"** — system robi auto-match, ale handlowiec dostaje notyfikację z linkiem "potwierdź / zmień" *do każdej* decyzji (nie tylko escalate). To jest pojedyncza Custom Automation w ClickUp na 2 tygodnie. Po 100 sprawdzonych decyzjach — wyłączamy shadow mode dla auto-match ≥próg.

### Karolina Wysocka

[FACT] Phase 0 anti-assumption #5: "handlowiec nie patrzy w ClickUp częściej niż 2–3× dziennie". [FACT] Phase 0 Problem F: średnio 40 minut rano na sortowanie skrzynki przez Panią Olę.

[INFERENCE] Z perspektywy handlowca **żaden z trzech kierunków nie pokazuje swojej różnicy w UI**. We wszystkich trzech: nowy email → wpada do taska, "Do weryfikacji" jako kolejka, push z @mention. Marta to potwierdziła. **Decyzja techniczna nie jest decyzją UX**. To upraszcza moją rolę — ja oceniam niezależnie od kierunku, czy *cały rytuał* działa.

[INFERENCE] Najtrudniejszy moment dnia handlowca = **rano, kolejka "Do weryfikacji" + nowe taski naraz**. Jeśli system w przeciągu nocy podejmie 30 auto-match decyzji + 5 escalate, handlowiec rano powinien mieć JEDEN widok pokazujący: (a) co AI zrobił automatycznie (do retroaktywnego sprawdzenia, ale z presumpcją OK), (b) co czeka na decyzję (escalate). Bez tego rytuał się rozjeżdża i kolejka rośnie.

[RECOMMENDATION] **Dwa filtered views w `NOWE ZAPYTANIA`**: "Do weryfikacji teraz" (`MATCHING_DECISION = escalated-need-confirm`) + "Auto-match z ostatnich 24h" (`MATCHING_DECISION = auto-match` AND `created < 24h`). Pierwszy to obowiązek przed sprzedażą; drugi to opcjonalny audyt przy kawie.

[FLAG] Aleksander postuluje shadow mode. [INFERENCE] Z perspektywy handlowca **shadow mode na 100 pierwszych decyzji = 100 dodatkowych klików** — jeśli system robi to dobrze, znielubi rytuał i zacznie klikać "potwierdź" bez czytania. Wtedy shadow mode nie waliduje, tylko trenuje na ignorowanie. **Postuluję shadow mode tylko na pierwsze 30 decyzji + losowe 10% z kolejnych 70**, nie na wszystkie 100. To zachowa uważność.

[INFERENCE] Problem D (sugestia tytułu) jest dla mnie **drugorzędny**. Handlowiec docenia, ale to nie jest moment, w którym sprzedaż się wygrywa lub przegrywa. Manual + assist z Fazy 2 jest właściwy. Nie chcę widzieć kierunku, który dorzuca custom UI tylko dla D.

---

## 3. Discussion — Runda 2: kontrowersje wymuszone przez moderatora

### Kontrowersja 1: Czy Kierunek A jest faktycznie martwy, czy tylko nieaktualny?

> **Moderator:** Marta odrzuciła A; nikt inny nie obronił. Przed lock-in'em chcę usłyszeć, czy ktoś widzi scenariusz, w którym A wygrywa.

**Tomasz:** [INFERENCE] Tylko jeden — gdyby ClickUp Agents okazały się dramatycznie tańsze i bardziej deterministyczne niż Anthropic + Make. Q1 z Fazy 1 jest niezbadane, ale **prawdopodobieństwo a priori jest niskie**: ClickUp Agents to młoda funkcja, planowo wycenianа premium, plan Business prawdopodobnie ma twarde limity AI uses. Bez zbadania Q1 nie podejmę go nawet jako pilot.

**Aleksander:** [INFERENCE] Druga sytuacja: gdyby zadanie matchera mieściło się w *kompletnie* deklaratywnej polityce (np. "ten sam email + ostatnia interakcja <30 dni → ten task"). Wtedy AI nie jest potrzebne — Automation ClickUp by wystarczyła. **Ale nasza polityka P2 to wyklucza** (90 dni window jest tylko prefiltrem, AI rozstrzyga semantykę). Więc A nie ma swojej niszy. Martwy.

**Karolina:** [INFERENCE] Z mojej strony: gdyby A radykalnie podnosił UX handlowca względem B/C. Nie podnosi — UI jest taki sam. Martwy.

**[KONSENSUS]** Kierunek A odrzucony. Pozostaje kandydatem na elementy lokalne (AI Field na `SUGEROWANY_TYTUŁ_MAILA`) ale nie jako kierunek matchera.

### Kontrowersja 2: B vs C — czy single-shot wystarcza?

> **Moderator:** Aleksander mówi "tak, z warunkami". Tomasz wskazał, że C "przesuwa złożoność". Czy ktokolwiek widzi konkretną decyzję, w której single-shot zawodzi, a agentic loop ratuje?

**Aleksander:** [INFERENCE] Konkretny przypadek: klient pisze "potwierdzam termin" z 5 otwartymi taskami (firma robi cykl wyjazdów integracyjnych). 5 kandydatów w prompcie + krótki mail bez kontekstu = AI zwraca confidence 0.4–0.6 dla każdego. Single-shot daje "escalate". Agentic loop *mógłby* w drugiej iteracji "doczytać" pełen description każdego kandydata i zawęzić. **Ale** tu pomaga lepszy prefiltr (status taska, planowana data), nie agentic loop. Heurystyka deterministyczna > AI loop.

**Tomasz:** [INFERENCE] Potwierdzam. Każdy przypadek, w którym agentic loop wygrywa nad single-shot, można rozwiązać lepszym prefiltrem w Make + lepszym promptem. Prefiltry są tanie i deterministyczne — agentic loop jest drogi i probabilistyczny.

**Marta:** [INFERENCE] Z mojej strony jeszcze: agentic loop **nie ma dostępu do kontekstu Workspace ClickUp** chyba że dorzucimy tool `query_clickup`. To jest ten sam koszt operacyjny co Search Tasks w Make — tylko w Workerze. Net zero przewaga.

**Karolina:** Jednoznaczny case dla agentic loop nie pojawia się w realiach EVSO 100/tydz. Może w 1000/tydz, ale wtedy mamy dane i wracamy do tej decyzji.

**[KONSENSUS częściowy → arbitraż wymagany ze względu na non-consensus #2]** Single-shot (Kierunek B) wybrany jako baseline. Agentic loop (Kierunek C) **nie jest wybrany dziś**, ale Faza 5 ma zaprojektować "matcher jako wymienny komponent" — interfejs HTTP `(new_email, candidates) → decision`, tak żeby przejście B → C było fizycznie wykonalne bez przeprojektowania architektury, jeśli skala lub jakość tego wymusi.

### Kontrowersja 3: Próg P1 (0.85 vs 0.92) i kalibracja

> **Moderator:** Aleksander postawił mocny argument przeciw 0.85 baseline. Faza 2 ten próg ustaliła. Czy zmieniamy?

**Aleksander:** [RECOMMENDATION] Start 0.92 dla auto-match, 0.65 dla escalate (zamiast 0.60). Po 50–100 testowych przypadkach kalibrujemy w dół, jeśli kolejka eskalacji jest za duża. **Ryzyko asymetryczne**: błędny auto-match = utracony lead; nadmiar eskalacji = handlowiec klika dwa razy dziennie. Asymetria mówi "ostrożny start".

**Tomasz:** [INFERENCE] Z perspektywy operatora 0.92 vs 0.85 to jedna liczba w prompt'cie / jedna zmienna w routerze Make. Operacyjnie różnica zerowa. Argument cenowy: jeśli 30% maili idzie do escalate przy 0.92 zamiast 15% przy 0.85, to handlowiec klika 100/tydz × 30% = 30/tydz × 30s = 15 min/tydz. To jest mało.

**Karolina:** [FLAG] **Nie zgadzam się** z "klika 30s × 30/tydz = 15 min". Klik nie jest kliknem — escalate wymaga przeczytania maila + przejrzenia kandydatów + decyzji. Realnie 1.5–3 min per escalate. 30 × 2 min = **60 min/tydz** — godzina ekstra. Przy 0.85 to 30 min/tydz. Różnica realna. **Postuluję 0.85, ale z obowiązkową kalibracją po 2 tygodniach na żywych danych.**

**Aleksander:** [INFERENCE] Karolina ma rację co do realnego kosztu eskalacji. **Ale** błędny auto-match przy 0.85 też kosztuje — 1 utracony lead = setki/tysiące PLN. Asymetria. Pozostaję przy 0.92.

**Marta:** [RECOMMENDATION] Kompromis pragmatyczny: **0.85 baseline + shadow mode pierwsze 2 tygodnie** (po Karolinie zmodyfikowane: shadow tylko na 30 + 10% reszty). Empirycznie sprawdzimy czy 0.85 daje błędy. Jeśli daje >2% błędu w shadow → podnosimy do 0.90.

**[ARBITRAŻ]** Decyzja moderatora: wybrana wersja Marty — 0.85 baseline z empiryczną walidacją w shadow mode. Aleksander pozostaje przy dissent (preferowałby 0.92). To trafia do non-consensus log.

### Kontrowersja 4: Shadow mode — wszystko czy próbka?

> **Moderator:** Aleksander chce shadow na pierwsze 100. Karolina mówi "zignorują, klikając OK". Konflikt UX vs reliability.

**Aleksander:** [INFERENCE] Karolina ma rację co do "trening na ignorowanie" — to jest realny pattern. Akceptuję jej propozycję: shadow na pierwsze 30 + losowe 10% kolejnych 70. To wciąż daje statystycznie mocny sygnał (37 obserwacji), bez wypalania uważności.

**Karolina:** Akceptuję.

**[KONSENSUS]** Shadow mode: pierwsze 30 decyzji + losowe 10% kolejnych 70. Po 100 decyzjach review wyników i decyzja czy próg podnieść/zostawić.

### Kontrowersja 5: Gdzie żyje "Do weryfikacji" — task czy osobny status?

> **Moderator:** Faza 2 (sekcja 4 #3) sygnalizuje: "kolejka Do weryfikacji rozszerza się — dziedziczy wzorzec Phase 1, ale potrzebuje sygnału że to *nowy* task z propozycją dopasowania". Marta i Karolina, jak to rozwiązujecie?

**Marta:** [RECOMMENDATION] Phase 1 używa `KLASYFIKACJA_AI = "Do weryfikacji"`. Dla M2 dorzucamy **odrębny enum** `MATCHING_DECISION = "escalated-need-confirm"`. Filtered view "Do weryfikacji" w `NOWE ZAPYTANIA` używa OR: `KLASYFIKACJA_AI = "Do weryfikacji" OR MATCHING_DECISION = "escalated-need-confirm"`. Dwa sygnały, jeden widok dla handlowca.

**Karolina:** [FLAG] **Nie zgadzam się.** Z perspektywy handlowca to są dwa różne typy decyzji do podjęcia: (a) "AI nie wie jakiego typu jest to zapytanie" (Phase 1 escalate) — handlowiec klasyfikuje, (b) "AI nie wie do którego taska to przyporządkować" (M2 escalate) — handlowiec dopasowuje lub tworzy nowy. **Inne narzędzia decyzyjne.** Postuluję **dwa osobne views** ze świadomym splitem.

**Marta:** [INFERENCE] Karolina ma rację. Dwa views to lepszy UX, jednolity sygnał operacyjny (jedno pole `MATCHING_DECISION`) — to nie są wykluczające się. Akceptuję.

**[KONSENSUS]** Dwa filtered views w `NOWE ZAPYTANIA`: (1) "Do weryfikacji — typ zapytania" (Phase 1 escalate), (2) "Do weryfikacji — dopasowanie maila" (M2 escalate). Plus trzeci read-only "Auto-match audit (24h)".

### Kontrowersja 6: `Add Comment as email` vs SMTP do `<id>@mg.clickup.com`

> **Moderator:** Q2 z Fazy 1 jest oznaczone "high criticality każdy kierunek". Tomasz, czy to blocker?

**Tomasz:** [FACT] Make ClickUp module ma "Add Comment" z opcjonalnym checkboxem "as email" — ale różne źródła sugerują, że to dodaje komentarz oznaczony jako email-w-tasku, niekoniecznie wpływający na thread `mg.clickup.com`. [INFERENCE] **Nie jestem pewien dziś bez testu.** Plan B: SMTP do `<id>@mg.clickup.com` z poziomu Make Email module — to jest udokumentowana ścieżka, ale wymaga poprawnego From (musi być @firmowa domena, inaczej ClickUp może odrzucić). [RECOMMENDATION] **Test w środowisku testowym jako pierwszy task Fazy 5.** To jest 1h pracy, blokuje Faza 5/7 design dataflow.

**Aleksander:** Jeśli SMTP ścieżka — uwaga na rate limity Make Email module + poprawność From header. To jest miejsce, gdzie "działa w teście, pęka w produkcji" jest realne.

**[KONSENSUS]** Q2 zostaje **otwarte** do empirycznego testu w Fazie 5. Faza 5 musi zaprojektować dataflow z założeniem fallbacku: preferowana ścieżka = ClickUp module "Add Comment as email", awaryjna = SMTP do `<id>@mg.clickup.com`. Decyzja po teście.

---

## 4. Rankingi ekspertów (3 kierunki)

### Marta Kurek

| Pozycja | Kierunek | Uzasadnienie |
|---|---|---|
| 1 | **B** | Pasywny ClickUp = czysty model danych. Custom Fields additive, filtered views native, brak nowej technologii w stacku. Pasuje do ziarna ClickUp jako "warstwa danych", co zespół rozumie. |
| 2 | **C** | UX handlowca taki sam jak B. Tracimy: nowa powierzchnia operacyjna (Worker), bus factor jeszcze gorszy. Zyskujemy: lepszy audit log. Nie warte. |
| 3 | **A** | Obciążenie ClickUp Agents niezbadane (Q1). Nowa technologia jako rdzeń = ryzyko niespójne z planem Business. Estetycznie najlepszy "all-in-one", ale to nie jest wystarczające uzasadnienie. |

### Tomasz Sowa

| Pozycja | Kierunek | Uzasadnienie |
|---|---|---|
| 1 | **B** | Inkrementalne rozszerzenie Phase 1. Wszystkie wzorce (HTTP → Anthropic, error handler → Do weryfikacji, Watch + Search + Update) sprawdzone produkcyjnie. Risk wykonawczy najniższy. |
| 2 | **C** | Czystsza architektura na papierze, ale "przesuwa złożoność" do Workera. Dla 100/tydz nie warte dodatkowej powierzchni operacyjnej. |
| 3 | **A** | Ścieżka emaili **omija Make** — tracę kontrolę nad error handling i obserwability. Dla mnie to oznacza, że jeśli coś się rozsypie, nie mam mojej standardowej toolbox. |

### Aleksander Bryła

| Pozycja | Kierunek | Uzasadnienie |
|---|---|---|
| 1 | **B** | Single-shot z dobrze zbudowanym promptem (few-shot, JSON schema, "if uncertain → escalate") wystarcza dla 100/tydz. Audit przez Make execution log + Custom Fields per task = wystarczający dla skali. **Pod warunkiem** że P1 baseline 0.85 jest empirycznie zwalidowany. |
| 2 | **C** | Agentic loop daje teoretyczną przewagę precyzji + lepszy decision_log, ale praktycznych przypadków, w których wygrywa nad single-shot + dobry prefiltr, nie znaleźliśmy. Realne przy 1000/tydz, nie przy 100. |
| 3 | **A** | ClickUp Agents = czarna skrzynka z perspektywy AI reliability. Brak wersjonowania promptu, brak dostępu do raw outputu modelu, "self-reported confidence" Agenta jest jeszcze bardziej miękki niż Haiku przez API. Kierunek bez audytowalności = kierunek nie do produkcji w zadaniu o realnym koszcie błędu. |

### Karolina Wysocka

| Pozycja | Kierunek | Uzasadnienie |
|---|---|---|
| 1 | **B** | UX handlowca = wszystkie 3 kierunki to samo. Wybieram po operatorze: B = Bartek może debug'ować w UI Make (znanym narzędziu), więc gdy coś się zepsuje, naprawa jest szybka. To jest service continuity. |
| 2 | **A** (równo z C) | Estetyka "wszystko w ClickUp" jest atrakcyjna, ale ryzyko Q1 jest niezbadane. Nie wybiorę produkcyjnego rozwiązania na nadziei. |
| 2 | **C** (równo z A) | Bartek na 2 tygodnie urlopu + Worker padnie = zespół nie radzi sobie. Bus factor handlowca = 0. Nie wybiorę. |

---

## 5. Decision Record

### Decyzja

**Wybrany kierunek: B — "Make-watcher heavy" (Faza 3 §B), z dwoma wbudowanymi elementami z innych kierunków:**

1. **Element z A:** AI Field na ClickUp dla `SUGEROWANY_TYTUŁ_MAILA` (Problem D, Manual + assist z Fazy 2). Lokalne, tanie, korzysta z natywnej zdolności CU-4 bez wprowadzania ClickUp Agents jako warstwy decyzyjnej. *Lub* — alternatywnie — call Anthropic z Make przy tworzeniu taska, do tego samego pola. Wybór konkretnej ścieżki w Fazie 5 po sprawdzeniu kosztu AI uses ClickUp.
2. **Element z C:** Architektura Fazy 5 musi zaprojektować "AI matcher jako wymienny komponent" z czystym kontraktem `(new_email, candidates) → {decision, target_task_id, confidence, reasoning}`. W M2 implementacja = HTTP module Make do Anthropic + parsing. W przyszłości (jeśli skala / jakość wymusi) = endpoint Cloudflare Worker. **Bez przebudowy reszty pipeline'u.**

### Rationale

- **Konsensus 4/4 ekspertów** na pozycji #1 dla B (wszystkie cztery rankingi).
- **Odrzucenie A jednogłośne** w Rundzie 2: ClickUp Agents nie udowodnione (Q1 nieznane), brak audytowalności (Aleksander), nowa technologia bez korzyści UX (Marta + Karolina), tracimy kontrolę error handling (Tomasz).
- **Odrzucenie C jako baseline jednogłośne**: brak realnego przypadku przewagi nad B + dobry prefiltr; przesunięcie złożoności bez jej redukcji; bus factor; over-engineering dla 100/tydz. **Ale** zachowujemy go jako "upgrade path" przez wymóg architektoniczny (matcher jako wymienny komponent).
- **Reguła zespołu** ("prostsze, stabilniejsze, faktycznie bardziej użyteczne"): B jest **prostszy** (inkrement Phase 1) i **stabilniejszy** (znane wzorce, znana platforma) niż A i C. Bardziej użyteczny niż A (kontrola + audyt). Tak samo użyteczny jak C dla skali EVSO.

### Dissent (zarejestrowany)

- **Aleksander Bryła** dissent na próg P1: preferuje 0.92 baseline zamiast 0.85. Decyzja arbitrażowa moderatora: 0.85 + obowiązkowy shadow mode + review po 100 decyzjach. Aleksander wpisany jako "zaakceptował arbitraż, pozostaje przy preferencji".

### Conditions for revisiting

| # | Warunek | Akcja przy spełnieniu |
|---|---|---|
| R1 | Empiryczna walidacja po 100 decyzjach: błąd auto-match >2% | Podnieść próg P1 z 0.85 do 0.90 lub 0.92 (dissent Aleksandra zostaje wówczas uznany retroaktywnie). |
| R2 | Empiryczna walidacja: kolejka escalate >40% maili | Obniżyć próg z 0.85 do 0.80 lub poszerzyć prefiltr P2 (np. 90 → 60 dni). |
| R3 | Skala >500 emaili/tydz utrzymana 4 tygodnie | Re-evaluacja Kierunku C (Worker) jako upgrade — interfejs matchera ułatwia migrację. |
| R4 | Q1 (ClickUp Agents na planie Business) zostanie potwierdzone jako produkcyjnie wykonalne i tańsze | Re-evaluacja użycia ClickUp Agents dla *podzadań* (np. summarize threadu w tasku) — **nie** dla matchingu. |
| R5 | Q2 (Make Add Comment as email vs SMTP) test pokazuje, że żadna ścieżka nie wpina maila do thread'u `mg.clickup.com` poprawnie | Eskalacja do Bartka — zmiana strategii dataflow w Fazie 5. |

### Confidence level

**Medium-High.** Konsensus 4/4 na #1, kierunek jest inkrementem znanego baseline'u (Phase 1), 5 otwartych pytań Q1/Q2/Q3/Q4/Q7 adresowanych w Fazie 5 jako weryfikacja empiryczna. Niskie confidence pojedyncze: próg P1 (do empirycznej walidacji) i ścieżka comment-as-email vs SMTP (do testu).

### Status decyzji

**Konsensus** na wybór kierunku B + elementy z A/C.
**Rozstrzygnięty arbitraż** moderatora na próg P1 = 0.85 (Aleksander dissent zarejestrowany, ale zaakceptował proces).

---

## 6. Lista nie-konsensusu (kompletna)

| # | Punkt sporny | Strony | Rozstrzygnięcie | Status |
|---|---|---|---|---|
| NK1 | Próg P1 baseline: 0.85 vs 0.92 | Aleksander (0.92) vs Tomasz/Marta/Karolina (0.85) | Arbitraż moderatora: 0.85 + shadow mode + review po 100 | Zamknięte (Aleksander zaakceptował arbitraż, dissent zarejestrowany) |
| NK2 | Single-shot wystarcza vs agentic loop teoretycznie lepszy | Aleksander częściowo (uznał, że dla EVSO wystarcza, ale teoria mówi co innego) vs reszta | Konsensus: B baseline + matcher jako wymienny komponent (R3 jako warunek revisit) | Zamknięte |
| NK3 | Shadow mode 100% pierwszych 100 vs próbka 30+10% reszty | Aleksander (100%) → Karolina (próbka, ryzyko "klikania OK") | Aleksander zaakceptował argument Karoliny | Zamknięte przez konsensus |
| NK4 | Jeden filtered view "Do weryfikacji" (OR-gate) vs dwa osobne views | Marta (jeden view, OR pól) → Karolina (dwa views, różne narzędzia decyzyjne) | Marta zaakceptowała argument Karoliny | Zamknięte przez konsensus |
| NK5 | Cost ekspert kalkulacja eskalacji: 30s vs 1.5–3 min | Tomasz (30s) vs Karolina (1.5–3 min) | Karolina ma rację — escalate to nie klik, to decyzja. Wpływa na próg P1 i waliduje argument Aleksandra częściowo. | Zamknięte (Karolina) |
| NK6 | Q2 — preferowana ścieżka comment-as-email | Tomasz: nie pewny — Add Comment vs SMTP | Otwarte do testu w Fazie 5 z fallback-design | **Otwarte** (zaadresowane w Fazie 5) |
| NK7 | AI Field ClickUp vs Make+Anthropic dla `SUGEROWANY_TYTUŁ_MAILA` | Marta sugeruje AI Field (lokalne); Aleksander preferuje Make+Anthropic (kontrola promptu, wersjonowanie) | Otwarte do decyzji w Fazie 5 (zależy od kosztu AI uses ClickUp — Q3) | **Otwarte** (zaadresowane w Fazie 5) |

**Lista nie-konsensusu nie jest pusta** (7 pozycji, 2 świadomie pozostawione otwarte do Fazy 5 — to jest intencjonalna delegacja, nie pominięcie).

---

## 7. Zaakceptowane kompromisy

Kompromisy przyjęte świadomie wraz z wyborem Kierunku B:

| # | Kompromis | Co tracimy | Mitygacja |
|---|---|---|---|
| KK1 | "Spaghetti scenariusz" Make jako realny risk | Czytelność scenariusza degraduje się przy >10 modułach | Dyscyplina Tomasza: izolacja gałęzi, blueprint do gita, dokumentacja inline |
| KK2 | Single-shot prompt zamiast agentic loop | Brak refinement w trudnych przypadkach | Lepszy prefiltr (P2) + lepszy prompt (few-shot, "if uncertain → escalate") |
| KK3 | Próg 0.85 zamiast 0.92 (asymetria błędu) | Wyższe ryzyko błędnego auto-match niż Aleksander preferuje | Shadow mode pierwsze 30 + 10% reszty + review po 100 (R1) |
| KK4 | Audit decyzji w Make execution log + Custom Fields, nie w dedicated DB | 30-dniowy retention Make ogranicza retroaktywny audit | Custom Field `MATCHING_REASONING` na tasku zostaje wiecznie |
| KK5 | Brak push real-time (5-min poll) | Maks. opóźnienie ~5–7 min (P4 z Fazy 2) | Akceptowalne dla SLA EVSO; matcher wymienny umożliwia upgrade |
| KK6 | Bus factor = Bartek | Krytyczny single point of operability | Operability runbook (Faza 7 sekcja 8); fallback-design "Do weryfikacji" gdy AI niedostępne (Phase 1 wzorzec) |

---

## 8. Uncertainty register (zaktualizowany po Fazie 4)

| # | Niewiadoma | Krytyczność | Adresowane w |
|---|---|---|---|
| U1 | Q1: ClickUp Agents na planie Business — limity i koszty | Niska (po wyborze B) | Re-evaluacja w R4 |
| U2 | Q2: Make Add Comment as email vs SMTP `<id>@mg.clickup.com` | **Wysoka** | Pierwszy task Fazy 5 — empiryczny test |
| U3 | Q3: ClickUp AI uses pakiet Business — relevant tylko dla `SUGEROWANY_TYTUŁ_MAILA` | Średnia | NK7 — Faza 5 decyduje na tej podstawie |
| U4 | Q4: Make ops cap obecnego planu EVSO | Średnia | Faza 5 — kalkulacja po finalnym dataflow |
| U5 | Q7: Forward `kontakt@*` Dhosting → inbox | Średnia | Faza 5 — migration plan |
| U6 | Empiryczna jakość Haiku na zadaniu matchera EVSO | **Wysoka** | Shadow mode → review po 100 (R1, R2) |
| U7 | Realny rozkład eskalacji (estymacja Faza 2: 10–25%) vs rzeczywistość | Średnia | Pomiar w shadow mode |

---

## 9. Action items (przekazane do Fazy 5)

1. **A1.** Zaprojektować dataflow scenariusza Make "Email Intake M2" — moduł po module, z idempotentnością po `Message-ID` w Data Store.
2. **A2.** Przeprowadzić test empiryczny Q2 (Add Comment as email vs SMTP) w środowisku testowym **przed** finalizacją dataflow.
3. **A3.** Napisać prompt v1.0 matchera (system + few-shot 2–3 przykłady + JSON schema) — Aleksander.
4. **A4.** Zaprojektować shadow mode jako "Make notification + ClickUp Custom Field `SHADOW_REVIEW_NEEDED`" + automatyczne losowanie 10% — Tomasz + Marta.
5. **A5.** Zaprojektować dwa filtered views w `NOWE ZAPYTANIA` ("Do weryfikacji — typ" i "Do weryfikacji — dopasowanie") + read-only "Auto-match audit 24h" — Marta.
6. **A6.** Zdefiniować interfejs HTTP "AI matcher" jako wymienny komponent (kontrakt request/response) — przygotowanie pod ewentualne przejście na Worker w przyszłości.
7. **A7.** Zaprojektować Custom Fields M2: `LAST_EMAIL_AT`, `MATCHING_CONFIDENCE`, `MATCHING_REASONING`, `MATCHING_DECISION` (enum), `SUGEROWANY_TYTUŁ_MAILA` — Marta.
8. **A8.** Zdecydować ścieżkę dla `SUGEROWANY_TYTUŁ_MAILA` (NK7) na bazie Q3 — Faza 5.

---

## 10. Trade-off map (synteza)

> Wybór B daje:
> - Spójność z Phase 1 (znany idiom Make + Anthropic + error handler), kosztem braku potencjalnej precyzji agentic loop.
> - Audytowalność przez Make execution log + Custom Fields, kosztem 30-dniowego retentionu execution log.
> - Niskie ryzyko wykonawcze, kosztem ryzyka "spaghetti scenariusz" przy rozroście.
> - Operability dla Bartka (znane narzędzie), kosztem bus factor = 1.
>
> Wybór A dałby: lepszy "all-in-one" UX, kosztem niezbadanego ryzyka ClickUp Agents + brak audytowalności + nowa technologia.
> Wybór C dałby: lepszy decision_log + cleaner kod, kosztem 2× czasu rozwoju + dodatkowej powierzchni operacyjnej + jeszcze gorszego bus factor.

Wybór B + element A (AI Field/Make dla `SUGEROWANY_TYTUŁ_MAILA`) + element C (matcher jako wymienny komponent) realizuje regułę "prostsze + stabilniejsze + bardziej użyteczne" wzdłuż wszystkich trzech wymiarów.

---

## 11. Kryterium gotowości Fazy 4 (self-check)

- [x] **4 rankingi × 3 pozycje** — sekcja 4, każdy ekspert podpisany, każda pozycja z uzasadnieniem.
- [x] **Decision Record** podpisany — sekcja 5: konsensus na wybór kierunku B; rozstrzygnięty arbitraż na próg P1 (0.85). Rationale, dissent, conditions for revisiting (R1–R5), confidence level (Medium-High), status decyzji.
- [x] **Lista nie-konsensusu nie pusta** — sekcja 6 (7 pozycji NK1–NK7, w tym 5 zamkniętych przez konsensus/arbitraż i 2 świadomie pozostawione otwarte do Fazy 5).
- [x] **Lista zaakceptowanych kompromisów** — sekcja 7 (KK1–KK6).
- [x] Brak referencji do `archive/*`.

**Status: spełnione.**
