 Plan: Workflow projektowy dla Milestone 2 (EVSO Email Continuity)
                                                                                                                                         
     Context
                                                                                                                                         
     Bartek ma zdefiniowany problem M2 (docs/Plans/EVSO_Milestone2_Problem_Definition.md) — ciągłość konwersacji email ↔ ClickUp,      
     problemy A–F, 7 kryteriów sukcesu, świadome uproszczenia (np. email = tożsamość klienta). Wcześniejsze próby rozwiązania
     (specyfikacja, meeting notes, ocena agentów ClickUp, "jak rozwiązujemy") zostały przeniesione do archive/ — Bartek świadomie odcina
     się od tych iteracji i chce zaprojektować rozwiązanie od zera, żeby uniknąć biasu i lock-in'u na kierunku, który wcześniej wyszedł.

     Bartek nie prosi o rozwiązanie. Prosi o workflow — strukturalny proces myślowy, który gwarantuje dotarcie do dobrego, spójnego,
     konkretnego rozwiązania. Wynik workflow ma być Implementation Guide (gotowy do ręcznego wdrożenia w Make.com + ClickUp + Anthropic
     API) — nie sama specyfikacja.

     Decyzje sterujące workflow (potwierdzone z Bartkiem):
     1. Punkt końcowy: Implementation Guide (architektura + plan wdrożenia, module-by-module).
     2. Tryb wykonania: Faza-po-fazie, osobne sesje. Każda faza = jeden prompt do Claude → jeden plik markdown → review przez Bartka →
     korekta → przejście do następnej.
     3. Anti-bias: pełne odcięcie od archive/* na zawsze. Nie wolno czytać tych plików nawet w późniejszych fazach.

     ---
     Zasady projektowe workflow

     Każda faza ma być zaprojektowana tak, żeby wymuszała konkretny proces myślowy, którego nie da się skrócić. Bez tych wymuszeń
     pierwsza intuicja wygrywa i tracimy szansę na lepsze rozwiązanie.

     1. Anti-bias guard na starcie każdej sesji. Pierwsze zdanie outputu Claude w każdej fazie: explicit ack listy plików, które wczytał,
      i potwierdzenie "nie czytałem archive/". Brak tego = faza odrzucona.
     2. Divergencja przed konwergencją. Faza 3 wymusza 3 zasadniczo różne kierunki architektoniczne — nie warianty tego samego pomysłu.
     Pierwszy pomysł nie wygrywa automatycznie.
     3. Multi-perspective stress test. Faza 4 używa istniejącego panelu 4 ekspertów (docs/Teams/EVSO_DevTeam.md) przez skill
     expert-meeting-facilitator. Każdy ekspert wypowiada się ze swojej perspektywy, decision record dokumentuje konsensus i
     nie-konsensus.
     4. Adversarial gate przed lock-in. Faza 6 świadomie próbuje zburzyć projekt z Fazy 5. Lepiej znaleźć dziurę teraz niż w produkcji.
     5. Każda faza ma kryterium gotowości. Bez "wydaje mi się że wystarczy" — jest konkretny test, który mówi czy faza skończona.
     6. Outputy są kumulatywne. Każda kolejna faza wczytuje outputy poprzednich. Nie wracamy do problem definition w każdej fazie — Faza
     0 robi z niej trwałą "mapę pracy".
     7. Mapping back na końcu. Faza 7 explicit mapuje rozwiązanie na 7 kryteriów sukcesu z Problem Definition. Loop closure obowiązkowy.

     ---
     Struktura plikowa workflow

     Wszystkie outputy w nowym folderze: docs/Plans/M2_Workflow/

     docs/Plans/M2_Workflow/
     ├── 00_Workflow_Plan.md                     ← KOPIA niniejszego planu (po akceptacji)
     ├── 01_Phase0_Problem_Map.md                ← Faza 0 output
     ├── 02_Phase1_Capability_Map.md             ← Faza 1 output
     ├── 03_Phase2_Decision_Matrix.md            ← Faza 2 output
     ├── 04_Phase3_Divergent_Directions.md       ← Faza 3 output
     ├── 05_Phase4_Expert_Deliberation.md        ← Faza 4 output
     ├── 06_Phase5_Architecture.md               ← Faza 5 output
     ├── 07_Phase6_Adversarial_Review.md         ← Faza 6 output
     └── EVSO_Milestone2_Implementation_Guide.md ← Faza 7 — final deliverable

     Język outputów: PL (zgodny z Problem Definition i kontekstem zespołu). Nazwy techniczne (Custom Fields, scenariusze Make, statusy)
     verbatim zgodnie z CLAUDE.md.

     ---
     Fazy workflow

     Faza 0 — Re-anchoring & Problem Map

     Cel poznawczy: Zinternalizować problem własnymi słowami, bez biasów z poprzednich iteracji. Stworzyć trwałą mapę problemu, która
     będzie wczytywana przez wszystkie kolejne fazy.

     Input:
     - docs/Plans/EVSO_Milestone2_Problem_Definition.md (PL — główne źródło)
     - docs/EVSO_Problem_Definition_Context.md (EN — kontekst biznesowy)

     Zadanie:
     1. Przepisać problemy A–F własnymi słowami — każdy w 3 sekcjach: istota, przykład z życia handlowca EVSO, konsekwencja jeśli
     zostanie nierozwiązany.
     2. Wyodrębnić constraints: (a) twarde — narzucone przez stack/tooling/budżet; (b) miękkie — preferencje organizacyjne; (c)
     wynegocjowalne — gdzie warto wracać do Bartka.
     3. Wylistować świadome kompromisy z Problem Definition (np. email = tożsamość, telefon = manual) i dla każdego — co dokładnie
     tracimy decyzją upraszczającą.
     4. Wylistować anti-assumptions — czego rozwiązanie NIE może zakładać (klient nie zachowuje się przewidywalnie; nie każdy email da
     się jednoznacznie powiązać; itp.).

     Output: 01_Phase0_Problem_Map.md

     Kryterium gotowości: Każdy z 6 problemów (A–F) ma swoją sekcję; każdy świadomy kompromis ma jasny "koszt"; lista anti-assumptions
     zawiera ≥4 punkty.

     ---
     Faza 1 — Capability Mapping

     Cel poznawczy: Wiedzieć dokładnie co MAMY zanim cokolwiek zaprojektujemy. Skok do rozwiązania bez tej wiedzy = zgadywanie.

     Input:
     - 01_Phase0_Problem_Map.md
     - docs/Clickup/EVSO_ClickUp_Context_Snapshot.md
     - docs/Dataflows/EVSO_Lead_Intake_Dataflow_Context.md
     - docs/Make/EVSO_Form_Intake_v2.blueprint.json (jako autorytatywny stan Phase 1)
     - docs/Plans/Done/EVSO_Implementation_Guide_Phase1.md (referencja — co działa dziś, czego nie burzyć)
     - Sekcja "Zasoby dostępne" z EVSO_Milestone2_Problem_Definition.md

     Zadanie:
     1. Inwentaryzacja zdolności tooling'u w 4 obszarach:
       - ClickUp Business: Email-to-Task (per-task @mg.clickup.com), AI Agents, AI Fields, Custom Fields, Automations, Brain, Inbox,
     native email composer, ClickUp Forms.
       - Make.com: data stores, scenariusze, watchery (Email/Gmail), routery, error handlers, Iterators/Aggregators, scheduled vs instant
      triggers.
       - Anthropic API (Claude Haiku 4.5): klasyfikacja, ekstrakcja, semantyczne dopasowanie, agentyczne loopy (multi-turn), tool use,
     prompt caching.
       - Vibe coding (microservice): kiedy ma sens (niestandardowa logika fuzzy matchingu, custom UI, persistent state) i kiedy NIE (gdy
     ClickUp/Make to robi natywnie).
     2. Dla każdej zdolności podać: co potrafi, gdzie ma limity / pułapki, koszt operacyjny (rate limits, ceny).
     3. Zbudować Capability × Problem matrix — tabela 6 problemów × 4 obszary tooling. Każda komórka: "tak, adresuje, jak…" / "częściowo,
      brakuje…" / "nie, wymaga workaround".

     Output: 02_Phase1_Capability_Map.md

     Kryterium gotowości: Każdy z 6 problemów ma ≥1 mapping na konkretną zdolność. Capability matrix wypełniona (24 komórki). Sekcja
     "limity i pułapki" jest niefantazjowana — oparta na rzeczywistych dokumentacjach lub własnym doświadczeniu z Phase 1.

     ---
     Faza 2 — Decision Matrix (linia odcięcia automatyzacji)

     Cel poznawczy: Świadomie wybrać poziom ambicji dla każdego problemu zanim zaczniemy projektować jak. Bez tego ryzyko, że
     automatyzujemy coś co powinno zostać manualne, albo zostawiamy manual coś co dało się zautomatyzować tanio.

     Input: Faza 0 + Faza 1.

     Zadanie: Dla każdego z problemów A–F wybrać 1 z 4 poziomów ambicji:
     - Full auto — system robi to bez interwencji człowieka, z wysoką pewnością.
     - Auto + escalate — system próbuje, ale przy niskiej pewności trafia do kolejki "do weryfikacji" dla pracownika.
     - Manual + assist — pracownik robi sam, ale system mu sugeruje (np. proponowany task do dopasowania, draft tytułu).
     - Out of scope — świadomie nie ruszamy w M2.

     Dla każdej decyzji uzasadnienie + konsekwencja ("przy tym poziomie godzimy się że…"). Dodatkowo rozstrzygnąć:
     - Próg pewności AI dla matchingu emaili (np. ≥0.85 → auto, 0.6–0.85 → escalate, <0.6 → nowy task).
     - Polityka powracających klientów (problem z Problem Definition: lead z zeszłego roku ≠ nowy wyjazd) — okno czasu, po którym nowy
     email = nowy task.
     - Polityka duplikatów (jeden klient, dwa otwarte leady jednocześnie).
     - Real-time vs batch (Make instant vs scheduled).

     Output: 03_Phase2_Decision_Matrix.md

     Kryterium gotowości: 6 wierszy (A–F) × 1 decyzja każdy + 4 polityki rozstrzygnięte. Każda decyzja ma uzasadnienie ≥2 zdania.

     ---
     Faza 3 — Divergent Solution Sketches (3 niezależne kierunki)

     Cel poznawczy: Nie pozwolić pierwszemu pomysłowi wygrać. Wymuszone wygenerowanie 3 zasadniczo różnych kierunków daje widok na space
     rozwiązań i ujawnia ukryte trade-offy.

     Input: Fazy 0, 1, 2.

     Zadanie: Naszkicować 3 kierunki, zasadniczo różne (nie warianty siebie):

     - Kierunek A — "ClickUp-native heavy": ClickUp Agents + Email-to-Task + Automations robią większość pracy. Make jest cienką warstwą
     I/O. AI Fields klasyfikują, Agents matchują.
     - Kierunek B — "Make-watcher heavy": Make jest centralnym mózgiem. Watcher Gmaila → klasyfikator AI w Make → router → ClickUp
     dostaje gotowe instrukcje (utwórz/przypisz/notyfikuj). ClickUp jest prawie pasywnym storage + UI.
     - Kierunek C — "Hybrid + custom service": Krytyczna logika (fuzzy matching email ↔ task, polityka powracających klientów,
     deduplikacja) w mikroservisie (Vibe coding, np. Cloudflare Worker / Vercel function). Make i ClickUp jako I/O i UI.

     Dla każdego kierunku:
     1. Architektura wysokopoziomowa (komponenty + strzałki).
     2. Jak rozwiązuje każdy z 6 problemów (mapping linia po linii).
     3. Kluczowe ryzyka (gdzie kierunek się łamie).
     4. Koszt rozwoju (przyblizone person-days).
     5. Koszt operacyjny (API calls/miesiąc, opłaty).
     6. Operability (jak debugować gdy coś idzie nie tak; ile osób w zespole musi rozumieć).

     Output: 04_Phase3_Divergent_Directions.md

     Kryterium gotowości: 3 kierunki są ZASADNICZO różne (jeśli dwa zaczynają wyglądać podobnie — przeprojektować jeden). Każdy ma 6
     sekcji wyżej. Żaden nie jest "słomianym ludkiem" (każdy musi być realistycznie obroniony).

     ---
     Faza 4 — Expert Panel Stress Test

     Cel poznawczy: Nie dopuścić, żeby kierunek wygrał na bazie estetyki. Wymusić cross-examination z 4 specjalistycznych perspektyw.

     Input: Faza 3 + docs/Teams/EVSO_DevTeam.md (panel ekspertów).

     Skill: expert-meeting-facilitator (już dostępny w środowisku).

     Zadanie: Symulowane spotkanie panelu 4 ekspertów. Dla każdego z 3 kierunków każdy ekspert daje:
     - Marta Kurek (ClickUp Systems Architect) — czy kierunek pasuje do ziarna ClickUp? gdzie automacje się zaczną łamać przy skali? jaka
      jest "ergonomia handlowca" w tym kierunku?
     - Tomasz Sowa (Make Automation Architect) — co jest stabilne/kruche w Make w tym kierunku? jakie są scenariusze błędów? error
     handling?
     - Aleksander Bryła (AI Systems & Reliability) — gdzie AI w tym kierunku można ufać? gdzie zawodzi po cichu? jak wygląda confidence
     gating?
     - Karolina Wysocka (Sales & Service Process Designer) — który kierunek faktycznie ułatwia handlowcowi dzień? gdzie są nowe friction
     points? jak ten kierunek interaguje z istniejącym sales process?

     Zasada zespołu (z EVSO_DevTeam.md): "tylko propozycja, która jest prostsza, stabilniejsza lub faktycznie bardziej użyteczna niż
     alternatywa".

     Output: 05_Phase4_Expert_Deliberation.md
     - Dla każdego eksperta: ranking 3 kierunków + uzasadnienie każdego punktu.
     - Decision Record: wybrany kierunek (lub explicit hybryda — np. "B + element C dla matchingu") + uzasadnienie + lista
     zaakceptowanych kompromisów + lista nie-konsensusu (gdzie eksperci się różnili i jak rozstrzygnięto).

     Kryterium gotowości: 4 rankingi × 3 pozycje. Decision Record podpisany "konsensus" lub "rozstrzygnięty arbitraż" (jasne kto i czemu
     wygrał spór). Lista nie-konsensusu nie jest pusta (jeśli jest — panel pracował zbyt grzecznie).

     ---
     Faza 5 — Architecture Lock-In + Edge Cases

     Cel poznawczy: Wziąć zwycięski kierunek i go skonkretyzować do poziomu, na którym każdy scenariusz brzegowy ma odpowiedź. Nie wolno
     zostawić "to później rozkminimy".

     Input: Faza 4 (zwycięski kierunek) + Faza 0 (powrót do problem definition — sanity check).

     Zadanie:
     1. Architektura final — komponenty, dataflow, integration points, sequence diagrams (markdown text-based).
     2. Pełna lista scenariuszy threading'u (Sx) — od zera, NIE kopiując z archive/. Zaczynamy od 7 kryteriów sukcesu z Problem
     Definition i pytamy: "jakie sytuacje muszą być pokryte żeby spełnić ten warunek". Spodziewane 6–10 scenariuszy. Dla każdego:
     wejście, trigger, kroki systemu, output, fallback przy błędzie.
     3. Data model changes — co dodajemy do ClickUp (nowe Custom Fields, Automations); co dodajemy do Make (data stores, scenariusze);
     ewentualne schemata mikroservisu (jeśli Kierunek C wygrał).
     4. Failure modes — co system robi gdy: AI klasyfikuje błędnie / niżej niż próg, email nie pasuje do żadnego taska, są duplicates,
     klient zmienił adres, klient wraca po roku z nowym wyjazdem, ClickUp API jest down, Gmail watcher gubi wiadomość.
     5. Migration plan — jak włączyć paralelnie z Phase 1 (nie burząc istniejącego form intake) w testowym środowisku (Forminator klon +
     automatyzacjaevso@gmail.com + testowy ClickUp Workspace). Kolejność włączania.

     Output: 06_Phase5_Architecture.md

     Kryterium gotowości: Każdy z 7 kryteriów sukcesu ma ≥1 mapowany scenariusz Sx. Każdy scenariusz Sx ma fallback. Migration plan jest
     etap-po-etapie (nie "włączamy wszystko naraz").

     ---
     Faza 6 — Adversarial Review

     Cel poznawczy: Świadomie zburzyć projekt z Fazy 5. Lepiej znaleźć dziurę teraz niż na żywym kliencie.

     Input: Faza 5.

     Zadanie: Lista ataków na projekt. Dla każdego: opis sytuacji, co się stanie wg projektu, czy to jest OK, decyzja (fix-now /
     accept-risk / fix-later z tagiem). Lista minimalna (każdy musi być rozważony):
     - Skala 10x: 1000 leadów/tydzień zamiast 100. Co się łamie? Rate limits? Koszt API? Manual queue rośnie zbyt szybko?
     - Cicha porażka AI: klasyfikator zwraca wysokie confidence ale błędną decyzję. Jak to wykrywamy? Po jakim czasie?
     - Pracownik ignoruje: notyfikacje są ignorowane przez tydzień. Co system robi? Eskaluje?
     - Zmiana ClickUp API: @mg.clickup.com przestaje działać / format się zmienia. Jak system się degraduje?
     - Zmiana adresu klienta: klient pisze najpierw z jan@firma.pl, potem z jan@gmail.com. Wiemy, że nie obsługujemy tego (świadomy
     kompromis) — ale jaki dokładnie efekt? Duplikat? Nowy task? Manual cleanup?
     - Powrót po roku: klient pisze rok później na nowy wyjazd. System rozpoznaje że to "ten sam email" — czy poprawnie tworzy NOWY task,
      nie wrzuca do archiwalnego?
     - Gmail outage: watcher Make nie odbiera wiadomości przez 6h. Czy system się leczy po przywróceniu? Czy gubimy wiadomości?
     - Operability: Bartek wyjeżdża na 2 tygodnie. Czy zespół (handlowcy bez technicznej wiedzy) może utrzymać system? Co jeśli Make
     scenariusz się zatrzyma — kto i jak diagnozuje?

     Output: 07_Phase6_Adversarial_Review.md

     Kryterium gotowości: ≥8 ataków. Każdy ma decyzję (nie wolno "rozważymy"). Lista fix-now ma być wbudowana z powrotem do Fazy 5
     (update 06_Phase5_Architecture.md) przed przejściem do Fazy 7.

     ---
     Faza 7 — Implementation Guide (Final Deliverable)

     Cel poznawczy: Złożyć wszystko w dokument, który Bartek może wziąć i wdrożyć bez wracania do wcześniejszych dokumentów workflow.

     Input: Faza 5 (po update z Fazy 6) + Faza 6.

     Zadanie: Napisać EVSO_Milestone2_Implementation_Guide.md w strukturze:

     1. Cele — mapping na 7 kryteriów sukcesu z EVSO_Milestone2_Problem_Definition.md. Dla każdego kryterium: jak guide je realizuje +
     jak to przetestować.
     2. Architektura wysokopoziomowa — diagram komponentów + dataflow + integration points (z Fazy 5).
     3. Zmiany w ClickUp — additive, nie destruktywne (zasada z CLAUDE.md):
       - Nowe Custom Fields (z polskimi nazwami).
       - Nowe Automations.
       - Email-to-Task config / @mg.clickup.com setup.
       - AI Agents config (jeśli używamy).
     4. Scenariusze Make — module-by-module, z konkretnymi modułami (Webhook / Gmail Watcher / Anthropic / ClickUp / Iterator / Router /
     Error Handler). Per-scenariusz: trigger, moduły w kolejności, mapping zmiennych, error handling.
     5. Prompty AI (versioned) — pełne prompty (system + user + przykłady), z numerem wersji v1.0. Zachować strukturę KLASYFIKACJA_AI /
     AI_CONFIDENCE etc. z Phase 1.
     6. Plan testów — środowisko testowe (Forminator klon + automatyzacjaevso@gmail.com + testowy Workspace). Test cases mapowane na
     scenariusze Sx z Fazy 5. Dla każdego test case: kroki, expected output, jak weryfikować.
     7. Roll-out plan — etapy (test env → 1 strona pilot → wszystkie strony). Dla każdego etapu: success criteria, rollback plan, kto
     sprawdza.
     8. Operability runbook — jak debugować ("scenariusz Make zatrzymał się — gdzie patrzeć"), gdzie są logi, jak wymienić prompt AI bez
     ruszania scenariusza, jak dodać nową stronę firmową w przyszłości.
     9. Mapping success criteria (powtórzenie sekcji 1 ale jako tabela final) — 7 kryteriów × link do sekcji guide × link do test case.

     Output: EVSO_Milestone2_Implementation_Guide.md

     Kryterium gotowości:
     - Wszystkie 9 sekcji wypełnione.
     - Każde z 7 kryteriów sukcesu ma test case w sekcji 6.
     - Wszystkie zmiany w ClickUp są oznaczone jako additive.
     - Phase 1 (docs/Plans/Done/EVSO_Implementation_Guide_Phase1.md) nie jest modyfikowany — guide tylko go rozszerza.
     - Bartek czytając ten dokument w izolacji (bez Faz 0–6) może go wdrożyć — sprawdź self-contained.

     ---
     Jak rozpocząć każdą fazę (template promptu)

     Każdą sesję zaczynaj jednolinijkowym promptem:

     Wykonaj Fazę N workflow M2 zgodnie z docs/Plans/M2_Workflow/00_Workflow_Plan.md.
     Wczytaj input zgodnie z sekcją "Faza N → Input".
     NIE czytaj plików z archive/ — pełne odcięcie.
     Pierwsze zdanie odpowiedzi: ack listy wczytanych plików + potwierdzenie "nie czytałem archive/".
     Output: docs/Plans/M2_Workflow/<nazwa-pliku-fazy>.md.
     Po zapisaniu pliku: sprawdź "Kryterium gotowości" dla tej fazy i powiedz, czy spełnione, jeśli nie — co poprawić.

     Po wykonaniu fazy: review przez Bartka, korekta jeśli trzeba, dopiero potem następna faza.

     ---
     Krytyczne pliki referencyjne (workflow nie ruszają oryginałów)

     - docs/Plans/EVSO_Milestone2_Problem_Definition.md — kotwica problemu (read-only)
     - docs/EVSO_Problem_Definition_Context.md — kontekst biznesowy
     - docs/Clickup/EVSO_ClickUp_Context_Snapshot.md — stan ClickUp
     - docs/Dataflows/EVSO_Lead_Intake_Dataflow_Context.md — dataflow
     - docs/Make/EVSO_Form_Intake_v2.blueprint.json — Phase 1 baseline (autorytatywny)
     - docs/Plans/Done/EVSO_Implementation_Guide_Phase1.md — co działa dziś (czego nie burzyć)
     - docs/Teams/EVSO_DevTeam.md — panel ekspertów do Fazy 4
     - CLAUDE.md — zasady projektu (additive, polski naming, etc.)

     archive/* — NIE wczytywane w żadnej fazie.

     ---
     Verification — jak wiedzieć, że workflow zadziałał

     Po Fazie 7 sprawdzić punkty końcowe (test integracyjny dla całego workflow):

     1. Self-containment: Implementation Guide czytany w izolacji jest wykonywalny przez Bartka (nie odwołuje się do Faz 0–6).
     2. Coverage: Każde z 7 kryteriów sukcesu z EVSO_Milestone2_Problem_Definition.md ma swoje miejsce w sekcji 9 guide'a.
     3. Backwards-compat: Phase 1 form intake nadal działa po wdrożeniu M2 (zasada additive zachowana).
     4. Test plan executability: Sekcja 6 guide'a (Plan testów) opisuje konkretne kroki w testowym środowisku — Bartek może je wykonać
     krok-po-kroku, używając Forminator klona, automatyzacjaevso@gmail.com i testowego Workspace.
     5. Anti-bias verification: Żaden plik wynikowy w docs/Plans/M2_Workflow/ nie cytuje, nie referuje ani nie kopiuje treści z
     archive/*. Można sprawdzić Grep'em.

     Jeśli któryś punkt nie przechodzi — wracamy do odpowiedniej fazy, nie patchujemy guide'a kosmetycznie.

     ---
     Krytyczne pliki, które ten plan tworzy

     Sam plan workflow:
     - C:\Users\Barte\.claude\plans\docs-plans-evso-milestone2-problem-defi-piped-puffin.md (ten dokument)

     Po akceptacji — kopia do repo:
     - docs/Plans/M2_Workflow/00_Workflow_Plan.md (wykonywalna wersja w repo)

     Outputy faz (utworzone podczas wykonania workflow, nie w tej fazie planowania):
     - docs/Plans/M2_Workflow/01_Phase0_Problem_Map.md ... 07_Phase6_Adversarial_Review.md
     - docs/Plans/M2_Workflow/EVSO_Milestone2_Implementation_Guide.md

● Plan zaakceptowany. Rozpoczynam implementację — pierwszy krok zgodnie z planem to skopiowanie workflow do repo jako
  docs/Plans/M2_Workflow/00_Workflow_Plan.md, żebyś miał bazę do uruchamiania kolejnych faz.

  Listed 1 directory (ctrl+o to expand)                                                                                                  
                                                                                          
● Write(docs\Plans\M2_Workflow\00_Workflow_Plan.md)                                                                                      
  ⎿  Wrote 284 lines to docs\Plans\M2_Workflow\00_Workflow_Plan.md                                                                       
      2 # Workflow projektowy dla Milestone 2 (EVSO Email Continuity)                     
        u kolejnych faz.akceptowany 2026-05-07. Plik wykonawczy — używaj go jako "instrukcji dla siebie i Claude" przy uruchamiani       
      4                                                                                                                                  
      6 ## Context                                                                                                                       
      7 Bartek ma zdefiniowany problem M2 (`docs/Plans/EVSO_Milestone2_Problem_Definition.md`) — ciągłość konwersacji email ↔ Clic       
        nia (specyfikacja, meeting notes, ocena agentów ClickUp, "jak rozwiązujemy") zostały przeniesione do `archive/` — Bartek *       
        nku, który wcześniej wyszedł. iteracji** i chce zaprojektować rozwiązanie od zera, żeby uniknąć biasu i lock-in'u na kieru       
      8                                                                                                                                  
        go, spójnego, konkretnego rozwiązania. Wynik workflow ma być Implementation Guide (gotowy do ręcznego wdrożenia w Make.com
         + ClickUp + Anthropic API) — nie sama specyfikacja.
     … +344 lines (ctrl+o to expand)

● Workflow zainstalowany w repo. Folder docs/Plans/M2_Workflow/ istnieje, 00_Workflow_Plan.md jest w środku jako instrukcja wykonawcza.
                                                                                          
  Co dalej: żeby uruchomić Fazę 0, otwórz nową sesję Claude i wklej:
                                                                                                                                         
  Wykonaj Fazę 0 workflow M2 zgodnie z docs/Plans/M2_Workflow/00_Workflow_Plan.md.
  Wczytaj input zgodnie z sekcją "Faza 0 → Input".                                                                                       
  NIE czytaj plików z archive/ — pełne odcięcie.            
  Pierwsze zdanie odpowiedzi: ack listy wczytanych plików + potwierdzenie "nie czytałem archive/".
  Output: docs/Plans/M2_Workflow/01_Phase0_Problem_Map.md.
  Po zapisaniu pliku: sprawdź "Kryterium gotowości" dla tej fazy i powiedz, czy spełnione, jeśli nie — co poprawić.

  Każda faza ma swój samodzielny prompt — wystarczy zmienić numer fazy i nazwę pliku output.

✻ Brewed for 10m 59s                    
                                                                                          
※ recap: Cel: zaprojektować rozwiązanie M2 (ciągłość emaili w ClickUp) przez 8-fazowy workflow myślowy. Plan workflow zaakceptowany i    
  zapisany w docs/Plans/M2_Workflow/00_Workflow_Plan.md. Następny krok: uruchomić Fazę 0 nowym promptem, który tworzy                    
  01_Phase0_Problem_Map.md. (disable recaps in /config)  