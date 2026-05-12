# Faza 0 — Problem Map (Milestone 2)

> **Status:** wynik Fazy 0 workflow M2.
> **Anti-bias ack:** wczytałem `docs/Plans/M2_Workflow/00_Workflow_Plan.md`, `docs/Plans/EVSO_Milestone2_Problem_Definition.md`, `docs/EVSO_Problem_Definition_Context.md`. NIE czytałem żadnych plików z `archive/*`.
> **Cel pliku:** trwała mapa problemu, na której opierają się Fazy 1–7 — bez konieczności wracania do oryginałów.

---

## 1. Problemy A–F — przepisane własnymi słowami

### Problem A — Fragmentacja historii konwersacji

**Istota.** Komunikacja z jednym klientem żyje równolegle w kilku miejscach: task w ClickUp (jeśli był formularz), skrzynka pocztowa (`kontakt@*` w Dhosting, gdzie ląduje większość maili poza wątkiem), pamięć handlowca (telefon), a czasem dwa lub trzy taski-duplikaty utworzone niezależnie. Nie ma jednego widoku "wszystko, co wiemy o tym kliencie".

**Przykład z życia handlowca EVSO.** Klient zgłosił się przez `partytram.fun` w środę — powstał `LEAD-7861`. W piątek zadzwonił, handlowiec zanotował uzgodnienia w komentarzu taska. W poniedziałek klient pisze maila bezpośrednio na `kontakt@partytram.fun` "potwierdzam czwartek" — mail siedzi w Dhosting, do CRM nie wpada. Handlowiec, otwierając `LEAD-7861`, nie wie, że potwierdzenie już przyszło, i sam ponawia kontakt. Klient odbiera to jako brak ogarnięcia.

**Konsekwencja jeśli nierozwiązany.** Handlowcy podejmują decyzje na niepełnych danych, dublują wiadomości, gubią potwierdzenia. Im więcej leadów aktywnych jednocześnie, tym większa szansa na realną stratę zamówienia ("nie odezwali się, to wziąłem inną firmę").

---

### Problem B — Identyfikacja klienta ponad kanałami

**Istota.** Żeby przypiąć nowego maila do istniejącego taska, system musi wiedzieć, że to ta sama osoba. Świadome uproszczenie M2: tożsamość = adres email. Jeden adres = jeden klient. Nie próbujemy łączyć tożsamości po telefonie, imieniu, IP. Dodatkowy wymiar: leady żyją "wiecznie" — klient, który pisał rok temu, może wrócić z **nowym** wyjazdem; system musi rozróżnić "kontynuacja sprawy" od "nowa sprawa od starego klienta".

**Przykład z życia handlowca EVSO.** Pani Kasia w marcu zarezerwowała tramwaj na wieczór panieński. Sprawa zamknięta, task w archiwum. W październiku pisze: "Hej, organizuję urodziny taty na grudzień, czy macie wolny termin?". Z tego samego adresu. System musi utworzyć **nowy** task, nie podpinać do tamtego archiwum sprzed 7 miesięcy.

**Konsekwencja jeśli nierozwiązany.** Albo klienci powracający trafiają do martwych archiwalnych tasków (handlowiec nigdy ich nie zobaczy), albo każdy email tworzy nowy task i tracimy ciągłość bieżących rozmów (nawet w obrębie jednego leadu).

---

### Problem C — Niepełna integracja emaili z CRM

**Istota.** ClickUp ma natywny mechanizm Email-to-Task: każdy task dostaje swój adres `<id>@mg.clickup.com`, a wiadomości wysłane stamtąd wracają w wątek do tego samego taska. To działa **tylko** kiedy klient odpisuje "Reply" na maila wysłanego przez ClickUp. Gdy klient pisze nową wiadomość, odpowiada na innego maila ze swojej skrzynki, pisze z innego klienta pocztowego (Outlook mobile, Gmail web), albo pisze bezpośrednio na publiczny adres `kontakt@partytram.fun` — wątek się urywa i wiadomość ląduje w Dhosting bez powiązania z taskiem.

**Przykład z życia handlowca EVSO.** Handlowiec wysyła z taska ofertę. Klient widzi maila na telefonie, klika "Napisz nową wiadomość" zamiast "Odpowiedz" (bo tak ma odruch), pyta o cenę dodatkowej godziny. Mail trafia na `kontakt@partytram.fun`, w ClickUp cisza. Handlowiec po dwóch dniach dzwoni — klient już zarezerwował u konkurencji.

**Konsekwencja jeśli nierozwiązany.** Mechanizm Email-to-Task jest "papierowo" włączony, ale w praktyce łapie tylko grzeczne odpowiedzi w wątku. Cała reszta wymaga ręcznego przeklejania albo ginie. Skala 100 zapytań/tydzień zamienia to w stałe operacyjne tarcie.

---

### Problem D — Sugerowanie nazwy wątków przy mailach wychodzących

**Istota.** Wątki **przychodzące** w odpowiedzi na maila z ClickUp wracają OK (nawet z "Re:"). Problem jest z **wychodzącymi** mailami inicjowanymi przez handlowca z poziomu taska — gdy zaczyna nowy wątek "od zera". Tytuł takiej wiadomości to ważny sygnał ciągłości zarówno dla klienta, jak i dla późniejszego matchowania (jeśli klient odpowie poza wątkiem, tytuł pomaga zlokalizować właściwy task). Dziś handlowiec wymyśla tytuł sam — bywa generyczny ("Oferta") albo niespójny między handlowcami.

**Przykład z życia handlowca EVSO.** Marek pisze do klienta z taska `LEAD-7912` (Wrocław, autobus, 28.06). Tytuł wpisuje "Oferta wyjazdu" — dwa tygodnie później, przy pięciu otwartych wątkach od tego samego klienta (firma robi cykl wyjazdów integracyjnych), nikt nie wie, który "Oferta wyjazdu" to ten konkretny.

**Konsekwencja jeśli nierozwiązany.** Pogarsza się jakość matchowania (Problem C i F), pogarsza się wewnętrzna nawigacja po skrzynkach, klient też dostaje sygnał "pracują z palca". Nie jest to problem krytyczny, ale jest realnym tarciem ergonomicznym.

---

### Problem E — Brak powiadamiania o nowych wiadomościach

**Istota.** Nawet jeśli mail zostanie poprawnie powiązany z taskiem, handlowiec się o tym nie dowie automatycznie. Musi sam patrzeć w ClickUp i w skrzynkę. Push z ClickUp dla aktywności w tasku jest, ale tylko kiedy task jest na wykonawcy lub watcherze — i nawet wtedy bywa cichy w mobilce.

**Przykład z życia handlowca EVSO.** Klient odpisał o 22:30 z konkretnym pytaniem o dostępność. Mail trafił do taska automatycznie. Handlowiec zaczyna dzień rano, otwiera ClickUp listę "ZLECENIA" → widzi 18 tasków, nie wie, w którym są nowe wiadomości. Otwiera po kolei → trafia na właściwy dopiero w trzecim. W międzyczasie minęło 1.5h od pierwszego loginu.

**Konsekwencja jeśli nierozwiązany.** Nawet idealnie działający routing maili daje 30–50% efektu, bo opóźnienie reakcji handlowca i tak zostaje. Skala (100/tydzień) zamienia to w realne SLA-owe straty.

---

### Problem F — Skalowalność ręcznego zarządzania

**Istota.** Każde ręczne powiązanie maila z taskiem, każde ręczne sprawdzenie "czy ten klient już ma sprawę", każde ręczne wyszukanie nazwiska w starych mailach — to czas. Przy 100/tydzień, nawet 2 minuty na mail = 3.3h tygodniowo na samo administrowanie.

**Przykład z życia handlowca EVSO.** Pani Ola codziennie rano dostaje "wieżę" 12–18 maili w `kontakt@partytram.fun`. Część to potwierdzenia formularzy (już w CRM), część to nowe odzewy klientów, część to zupełnie nowi ludzie. Sortowanie tego ręcznie i tagowanie do tasków zajmuje jej średnio 40 minut, zanim w ogóle zacznie sprzedawać.

**Konsekwencja jeśli nierozwiązany.** Handlowcy są płaceni za sprzedaż, ale spędzają znaczącą część dnia na kuratorstwie skrzynki. Przy wzroście wolumenu (więcej miast, więcej kanałów) ten koszt rośnie liniowo.

---

## 2. Constraints

### 2a. Twarde (narzucone — nie ruszamy bez zmiany założeń projektu)

| # | Constraint | Źródło |
|---|---|---|
| H1 | Stack pozostaje: ClickUp Business + Make.com + Anthropic API (Claude Haiku 4.5). | `CLAUDE.md`, Phase 1 baseline |
| H2 | Phase 1 (Form Intake Enhanced + Email Intake) **musi działać dalej** bez regresji. Wszystkie zmiany M2 są **additive** (router branches, appended modules), nigdy destruktywne. | `CLAUDE.md` ("invariant"), Faza 0 workflow |
| H3 | Custom Field, listy, statusy — **polskie nazwy verbatim** (`NOWE ZAPYTANIA`, `ZLECENIA`, `KLASYFIKACJA_AI`, `KOMPLETNOŚĆ`, `Do weryfikacji`). | `CLAUDE.md` |
| H4 | Tożsamość klienta = adres email (świadome uproszczenie M2). Bez prób łączenia po telefonie/imieniu. | Problem Definition §B |
| H5 | Telefon poza zakresem automatyzacji. Pracownik tworzy task ręcznie. | Problem Definition §"Granice" |
| H6 | Środowisko testowe to Forminator klon + `automatyzacjaevso@gmail.com` + testowy Workspace ClickUp. Nie testujemy na produkcji. | Problem Definition §"Środowisko testowe" |
| H7 | Plan ClickUp: Business (nie Enterprise). Limity AI Agents/AI Fields, Email-to-Task, Automations w ramach tego planu. | Problem Definition §"Narzędzia" |
| H8 | Make blueprinty są **autorytatywnym** stanem — zmiany opisujemy w markdown jako diff, Bartek aplikuje w UI Make i re-eksportuje. | `CLAUDE.md` |

### 2b. Miękkie (preferencje — można negocjować, ale są mocne sygnały)

| # | Preferencja | Sygnał |
|---|---|---|
| S1 | Rozwiązanie ma być proste i utrzymywalne przez **mały zespół** (Bartek + handlowcy bez technicznej wiedzy). Nie buduje się "fragile machine, którą rozumie jedna osoba". | Context §"Constraints and realism" |
| S2 | Reguła zespołu: nie proponujemy czegoś tylko dlatego, że jest możliwe — **tylko jeśli prostsze, stabilniejsze albo faktycznie bardziej użyteczne** niż alternatywa. | `docs/Teams/EVSO_DevTeam.md` (cytowane w workflow) |
| S3 | Koszt operacyjny AI ma znaczenie — Haiku zostało wybrane świadomie pod ~100 zapytań/tydzień. Nie eskalujemy do Sonneta/Opusa "tak na wszelki wypadek". | `CLAUDE.md` |
| S4 | Real-time vs batch: sygnał z Phase 1 (instant Make) sugeruje preferencję real-time, ale to nie jest absolutne — Faza 2 to rozstrzygnie. | Phase 1 baseline |
| S5 | "Out of scope" jest legitnym wynikiem dla M2 — nie wszystko musi być zautomatyzowane teraz. | Problem Definition §"Czego problem NIE zakłada" |

### 2c. Wynegocjowalne (warto wracać do Bartka w trakcie projektowania)

| # | Pytanie | Kiedy zadać |
|---|---|---|
| N1 | Próg pewności AI dla matchingu emaili (≥0.85? 0.9?). Trade-off: niższy próg → więcej automatyki + ryzyko błędu; wyższy → więcej manuala. | Faza 2 — Decision Matrix |
| N2 | Okno czasu po którym "ten sam email" = nowy task (90 dni? 180? po zamknięciu poprzedniego?). | Faza 2 |
| N3 | Polityka duplikatów: jeden klient z dwoma otwartymi leadami **w tym samym czasie** (np. wieczór panieński czerwiec + urodziny grudzień) — to dwa taski czy jeden? | Faza 2 |
| N4 | Czy używamy ClickUp AI Agents jako warstwy matchującej, czy całość matchingu siedzi w Make + Anthropic API? Wpływ na lock-in i koszt. | Faza 3/4 |
| N5 | Vibe-coded mikroservis (Cloudflare Worker / Vercel function) jako miejsce na fuzzy matching + politykę powracających klientów — robić, czy zostać w Make + AI? | Faza 3/4 |
| N6 | Powiadomienia (Problem E): ClickUp push/email natywny vs Slack/Telegram/SMS bridge. Ile handlowców, jakie urządzenia. | Faza 5 |
| N7 | Czy `kontakt@*` adresy z Dhosting forwardujemy do ClickUp Email-to-Task adresu poziomu listy, czy do Gmail watcher w Make, czy obu? | Faza 5 |

---

## 3. Świadome kompromisy — co dokładnie tracimy

| # | Kompromis | Co tracimy |
|---|---|---|
| K1 | **Tożsamość = email.** Klient zmienia adres (`jan@firma.pl` → `jan@gmail.com`) → traktowany jako nowy klient, nowy task. | Tracimy ciągłość historii dla tego (rzadkiego) przypadku. Manual cleanup po stronie handlowca: musi sam zauważyć i scalić. Realna częstotliwość: estymacja kilka procent leadów. |
| K2 | **Telefon = manual.** System nie robi nic z rozmową, dopóki handlowiec sam nie utworzy taska. | Jeśli handlowiec zapomni stworzyć task po rozmowie, a klient potem pisze maila — system traktuje go jak nowego. Tracimy ciągłość telefon→email do momentu, aż task istnieje. Mityguje to Problem Definition: telefon out-of-scope. |
| K3 | **Brak łączenia tożsamości po telefonie/imieniu.** Email od `kasia.nowak@gmail.com` i wcześniejszy lead ze `znajoma_firma@interia.pl` (ten sam człowiek) → dwa osobne taski. | Tracimy "single customer view" w sensie CRM-owym. EVSO świadomie zostaje przy "single email view" — prostsze, mniej fałszywych mergów. |
| K4 | **Pierwsza skrzynka publiczna nie jest wyłączona.** `kontakt@*` w Dhosting nadal istnieje i klienci mogą pisać tam bezpośrednio. M2 musi z tym współpracować, a nie tego eliminować. | Tracimy "clean architecture" gdzie wszystko idzie przez jeden kanał. Zysk: nie odcinamy klientów ani nie ruszamy istniejących stron. Koszt: musimy mieć watchera na te skrzynki. |
| K5 | **AI klasyfikuje z niepewnością — godzimy się na fallback "Do weryfikacji".** Phase 1 baseline: gdy AI nie ma pewności, task ląduje w `KLASYFIKACJA_AI = Do weryfikacji`. M2 dziedziczy tę zasadę dla matchingu maili. | Tracimy iluzję "100% auto". Zysk: nie tworzymy cichych błędów. Koszt: handlowiec musi mieć rytuał czyszczenia kolejki "Do weryfikacji". |
| K6 | **Lead żyje "wiecznie" — nie kasujemy starych tasków.** | Tracimy higienę listy (rośnie). Zysk: powracający klient po 12 miesiącach nie jest "anonimem" — system widzi historię. Koszt operacyjny: filtry/widoki w ClickUp muszą być świadome (statusy → archiwum, ale nie hard delete). |
| K7 | **Nie obsługujemy chat / social media / SMS / WhatsApp w M2.** | Tracimy te kanały świadomie. Mityguje: są dziś niskim wolumenem dla EVSO; jeśli zaczną rosnąć, wracamy w M3+. |

---

## 4. Anti-assumptions — czego rozwiązanie NIE może zakładać

1. **Nie zakładamy, że klient odpowiada w wątku.** Znaczna część odpowiedzi przychodzi poza wątkiem (nowy temat / inna wiadomość / inne urządzenie). Architektura matchująca musi działać **bez** polegania na nagłówkach `In-Reply-To` / `References`.
2. **Nie zakładamy, że każdy email da się jednoznacznie powiązać z jednym taskiem.** Klient z dwoma otwartymi leadami w tym samym tygodniu (wieczór panieński + integracja firmowa) to realny przypadek. System musi mieć politykę dezambiguacji (np. AI rozstrzyga po kontekście tematu/treści; jeśli niepewne — eskalacja, nie zgadywanie).
3. **Nie zakładamy, że klient pisze tylko z jednego adresu.** Mityguje to K3 (świadomie tracimy ten przypadek), **ale** rozwiązanie musi po cichu **nie szkodzić**, kiedy się zdarzy: nie tworzyć błędnych mergów, nie kasować historii.
4. **Nie zakładamy, że Gmail watcher / Make scenariusz nigdy się nie zatrzyma.** Outage'e, rate limity, błędy autoryzacji, retry'e — wszystko jest realne. Architektura musi mieć **idempotentność** (tego samego maila złapanego dwa razy nie powiela jako dwa taski) i **self-healing** (po wznowieniu watcher nadrabia zaległe wiadomości).
5. **Nie zakładamy, że handlowiec patrzy w ClickUp częściej niż 2–3× dziennie.** Powiadomienia muszą być aktywne (push out), nie pasywne (czekanie aż ktoś odświeży).
6. **Nie zakładamy, że ClickUp natywne API/`@mg.clickup.com` nigdy się nie zmieni.** To jest dependency third-party. Architektura powinna mieć ten punkt **wymienny** (warstwa abstrakcji albo udokumentowany failure mode + rollback).
7. **Nie zakładamy, że klient jest "rozsądny" w tytułach maili.** Tytuł "FW: Fwd: zapytanie" / "Re: " / pusty / "(brak tematu)" / emoji — wszystko jest realne. Matching nie może polegać wyłącznie na tytule.
8. **Nie zakładamy, że wolumen pozostanie ~100/tydzień.** Kierunki rozwoju EVSO mogą zwiększyć skalę 3–10×. Architektura nie musi być "enterprise", ale **nie wolno** projektować rzeczy, która łamie się przy 1000/tydzień (Faza 6 to zweryfikuje).

---

## 5. Powiązanie z 7 kryteriami sukcesu (sanity-anchor dla kolejnych faz)

| Kryterium | Adresowane przez problem |
|---|---|
| 1. Każdy email klienta → właściwy task | A, B, C |
| 2. Email od nieznanego → nowy task | B |
| 3. Pełna historia konwersacji w jednym tasku | A, C |
| 4. Pracownik nie wychodzi z ClickUp | C, E |
| 5. Duplikaty zminimalizowane | B, F |
| 6. Powiadamianie pracownika | E |
| 7. Telefon manualnie | (out of scope, K2) |

---

## 6. Kryterium gotowości Fazy 0 (self-check)

- [x] Każdy z 6 problemów (A–F) ma swoją sekcję z 3 częściami (istota / przykład / konsekwencja).
- [x] Każdy świadomy kompromis (K1–K7) ma jasny "koszt" — co dokładnie tracimy.
- [x] Lista anti-assumptions zawiera ≥4 punkty (zawiera 8).
- [x] Brak referencji do `archive/*`.

**Status:** spełnione.
