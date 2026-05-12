# Faza 1 — Capability Map (Milestone 2)

> **Status:** wynik Fazy 1 workflow M2.
> **Anti-bias ack:** wczytałem `docs/Plans/M2_Workflow/00_Workflow_Plan.md`, `docs/Plans/M2_Workflow/01_Phase0_Problem_Map.md`, `docs/Clickup/EVSO_ClickUp_Context_Snapshot.md`, `docs/Dataflows/EVSO_Lead_Intake_Dataflow_Context.md`, `docs/Plans/Done/EVSO_Implementation_Guide_Phase1.md` (jako referencja Phase 1 — co działa i czego nie burzymy; `docs/Make/EVSO_Form_Intake_v2.blueprint.json` jest autorytatywnym stanem opisanym module-by-module właśnie w tym Implementation Guide), sekcję "Zasoby dostępne i Architektura" z `docs/Plans/EVSO_Milestone2_Problem_Definition.md`. **NIE czytałem** żadnych plików z `archive/*`.
> **Cel pliku:** trwała inwentaryzacja zdolności tooling'u w 4 obszarach (ClickUp Business / Make.com / Anthropic API / Vibe coding) + Capability × Problem matrix. Faza 2 wybiera linię odcięcia automatyzacji **opierając się na tym, co tu udokumentowano**.

---

## 1. Zasada obowiązująca w tej fazie

Opisujemy **co MAMY**, nie co byśmy chcieli mieć. Dla każdej zdolności:
- *Co potrafi* — konkretna funkcja, w którym narzędziu, na jakim planie.
- *Limity / pułapki* — gdzie się łamie (z dokumentacji, z Phase 1 baseline, lub jasno oznaczone "do potwierdzenia").
- *Koszt operacyjny* — ceny, rate limits, ops cost (nie czas dewelopera; ten przyjdzie w Fazie 3).

Nie fantazjujemy: jeśli czegoś nie wiemy z 100% pewnością — flagujemy `[do potwierdzenia]`.

---

## 2. ClickUp Business — inwentaryzacja zdolności

### CU-1. Email-to-Task per task (`<id>@mg.clickup.com`)

- **Co potrafi.** Każdy task w ClickUp ma własny adres `<task-hash>@mg.clickup.com`. Wiadomości wysłane na ten adres trafiają do activity feed taska jako wiadomość email. Odpowiedź klienta na maila wysłanego z poziomu taska wraca do tego samego taska (in-thread reply przez nagłówki `In-Reply-To` / `References`).
- **Limity / pułapki.**
  - Działa dobrze tylko dla **odpowiedzi w wątku**. Nowy wątek od klienta → nie trafia (Problem C, Anti-assumption #1).
  - Adres `mg.clickup.com` jest po stronie ClickUp — to dependency third-party (Anti-assumption #6). Zmiana formatu/usługi po stronie ClickUp = degradacja całej architektury.
  - Zmiana subjectu przez klienta zwykle nie zrywa powrotu (`Re:` jest OK — działają headers), ale dokumentacja ostrzega o edge case'ach.
  - Adres `mg.clickup.com` jest "wewnętrzny" — klient go nie zna i nie pisze tam celowo.
- **Koszt operacyjny.** Brak dodatkowego — w cenie planu Business.

### CU-2. List-level email intake (mail-to-task na poziomie listy)

- **Co potrafi.** ClickUp dokumentuje natywne "Tasks via email" — adres na poziomie listy, na który mail tworzy nowy task. Subject = nazwa, body = description.
- **Limity / pułapki.**
  - Tworzy NOWY task za każdym razem — **nie matchuje** do istniejącego (czyli nie rozwiązuje Problem B/C samodzielnie).
  - Mapowanie do Custom Fields ograniczone (subject/body → tylko podstawowe pola; bez normalizacji miasta, bez klasyfikacji AI).
  - `[do potwierdzenia]` aktualny stan użycia w EVSO Workspace — Phase 1 nie polega na tym mechanizmie.
- **Koszt operacyjny.** W cenie planu.

### CU-3. AI Agents (Custom Agents w ClickUp)

- **Co potrafi.** ClickUp udostępnia natywne AI Agents — agenty mogą czytać konteksty z Workspace (Docs, Tasks), reagować na trigger (np. nowy komentarz, nowa wiadomość), wykonywać akcje (zmiana statusu, dodanie komentarza, przypisanie). Mogą "rozumieć" treść taska.
- **Limity / pułapki.**
  - Kluczowy limit: agent ClickUp działa **w domenie ClickUp** — operuje na taskach, listach, polach. Aby zrobić matching emaila ↔ task, musi mieć email *już w ClickUp* lub mieć narzędzie do query'owania tasków po polu EMAIL.
  - Agent nie pełni funkcji watchera Gmaila — to musi przyjść z zewnątrz (Make).
  - Operability: Bartek nie ma jeszcze produkcyjnego doświadczenia z ClickUp Agents w Phase 1 (Phase 1 idzie przez Make + Anthropic API). Wprowadzenie Agent jako warstwy decyzyjnej M2 oznacza nową technologię w stacku.
  - `[do potwierdzenia]` które feature'y są dostępne na planie Business vs. Enterprise (limity AI uses, rodzaje akcji, dostęp do API).
  - Zachowanie agenta jest mniej deterministyczne niż prompt w Make → trudniej debugować i wersjonować.
- **Koszt operacyjny.** Plan Business obejmuje pewną liczbę "AI uses" / miesiąc — `[do potwierdzenia]` dokładny pakiet i over-cap.

### CU-4. AI Fields

- **Co potrafi.** Custom Field typu AI — wartość wyliczana przez ClickUp AI na podstawie reszty taska (description + inne pola). Możliwe użycie do auto-podsumowań, klasyfikacji, ekstrakcji.
- **Limity / pułapki.**
  - Phase 1 świadomie **nie używa** AI Fields — wszystkie wyniki AI (`KLASYFIKACJA_AI`, `AI_PODSUMOWANIE`, `AI_CONFIDENCE`, `KOMPLETNOŚĆ`) są zapisywane przez Make po wywołaniu Anthropic API. Powód: pełna kontrola nad promptem + wersjonowanie + tańszy Haiku zamiast ClickUp credits.
  - Refresh AI Field nie jest typowo "live" przy zmianie pola źródłowego — `[do potwierdzenia]` częstotliwość.
  - Mniejsza kontrola nad promptem (UI ClickUp, brak versioningu).
- **Koszt operacyjny.** Wlicza się w pakiet AI uses planu.

### CU-5. Custom Fields (klasyczne)

- **Co potrafi.** Pełny zestaw używany w Phase 1: Dropdown, Short Text, Long Text, Number, Date, Email, Phone. Mapping Make → ClickUp Custom Field via field ID — dobrze przetestowane (Phase 1 ma 16 pól wypełnionych z Make).
- **Limity / pułapki.**
  - Custom Fields żyją na poziomie listy/foldera/space — przy dodawaniu nowych pól dla M2 (np. `EMAIL_THREAD_ID`, `LAST_EMAIL_AT`, `MATCHING_CONFIDENCE`) musimy zdecydować świadomie, gdzie je przyczepiamy. Domyślnie: na liście `NOWE ZAPYTANIA` (additive zgodnie z `CLAUDE.md`).
  - Pole typu **Email** zasila automatycznie sugestię w composerze maila → kluczowa zdolność dla M2.
- **Koszt operacyjny.** Brak.

### CU-6. Automations (natywne, no-code)

- **Co potrafi.** Trigger (status change / field change / created / etc.) → Action (set field / move / assign / notify). Phase 1 używa jednej automation: "Flag AI Do Weryfikacji" (priority Urgent przy `KLASYFIKACJA_AI = Do weryfikacji`).
- **Limity / pułapki.**
  - Trigger "field changed" jest synchroniczny ze zmianą pola, ale logika rozgałęzień jest płaska — bez warunków zagnieżdżonych w jednym recipe (musisz tworzyć wiele recipes).
  - Brak natywnego "wait X minutes" i brak "callback do external API" — do tego idziemy w Make.
  - Trudno zrobić "wyślij push tylko do tego pracownika z pola Y" w jednej automation bez kombinacji.
- **Koszt operacyjny.** Limity automation runs/mc na poziomie planu — `[do potwierdzenia]` Business cap.

### CU-7. ClickUp Brain

- **Co potrafi.** Q&A nad Workspace ("co wiemy o kliencie X"), generowanie podsumowań tasków, summarize threadu komentarzy.
- **Limity / pułapki.**
  - Z perspektywy M2 — to jest pomocnicze (handlowiec może użyć, jeśli historia konwersacji żyje już w tasku). Brain **nie pomaga w samym matchingu** — to nie jest narzędzie agentyczne wpinane w pipeline.
  - Skuteczność rośnie gdy task ma zebrane wszystko w jednym miejscu — czyli M2 zwiększa wartość Brain, nie odwrotnie.
- **Koszt operacyjny.** AI uses pool planu.

### CU-8. Inbox + push notifications

- **Co potrafi.** Inbox per użytkownik agreguje powiadomienia (mentions, assignments, watcher updates, comment replies). Push do mobile + desktop + email digest.
- **Limity / pułapki.**
  - Notyfikacja triggeruje się tylko jeśli user jest **assignee** lub **watcher** lub jest @mention. Sam fakt "do taska wpadł nowy email z zewnątrz" sam z siebie nie powiadamia, jeśli nikt nie jest watcherem — `[do potwierdzenia]` na 100%, ale to jest mocna intuicja z Phase 1.
  - Push mobile czasem cichy (Problem E).
- **Koszt operacyjny.** Brak.

### CU-9. Native email composer w tasku

- **Co potrafi.** Z poziomu taska wysyłka maila z firmowego adresu (już podpięty w EVSO Workspace zgodnie z Problem Definition §"Narzędzia"). Pole EMAIL taska zasila sugestię odbiorcy. Wątek wychodzący ma osadzony reply-to do `<id>@mg.clickup.com` (powrót w wątku — patrz CU-1).
- **Limity / pułapki.**
  - Tytuł maila wychodzącego jest pisany "z palca" przez handlowca → Problem D.
  - Brak natywnej "sugestii tytułu" — to miejsce, gdzie warto wstrzyknąć AI off-line (przez Custom Field z propozycją).
- **Koszt operacyjny.** Brak.

### CU-10. ClickUp Forms

- **Co potrafi.** Wbudowane formularze tworzące taski.
- **Limity / pułapki.** EVSO **nie używa** ich w produkcji — formularze to WPForms / Forminator na stronach brandowych. Z perspektywy M2 to ślepa zdolność (do M3+).
- **Koszt operacyjny.** N/A.

---

## 3. Make.com — inwentaryzacja zdolności

### MK-1. Webhook trigger (Custom Webhook)

- **Co potrafi.** Phase 1 baseline. WPForms / Forminator → POST → instant scenario start. Niska latencja (<1s widoczny task). Auto-detect struktury z pierwszej testowej submisji.
- **Limity / pułapki.**
  - "Instant" tylko do limitu jednoczesnych operations — przy spike'u 1000/tydz (Adversarial w Fazie 6) trzeba sprawdzić queue.
  - Webhook URL jest wrażliwy — rotacja wymaga update'u w WPForms / Forminator.
- **Koszt operacyjny.** Każda submisja = 1 ops + n modułów dalej. Dla M2 **dodajemy** webhook lub Gmail watcher, nie zastępujemy.

### MK-2. Gmail Watch Emails

- **Co potrafi.** Phase 1 ma to w "EVSO - Email Intake": polling co 5 min (configurable, można zejść do 1 min na droższych planach), zwraca metadata + body + attachments + headers. Filtrowanie po label / search query (`is:unread`, `from:`, etc.).
- **Limity / pułapki.**
  - **Polling, nie push** — minimalne opóźnienie 1–5 min (Anti-assumption #5: handlowiec też nie patrzy częściej, więc to jest OK na ogół, ale ważne dla "real-time vs batch" w Fazie 2).
  - "Mark as read: Yes" jest sposobem idempotentności (nie chwytamy tej samej wiadomości dwa razy), ale wprowadza **destruktywne side effect** na skrzynce. Alternatywa: deduplication po `Message-ID` przez Data Store.
  - Outage Gmaila / OAuth refresh — scenariusz może "leżeć" — Make wysyła notyfikację o błędzie (jeśli włączone). Self-healing po przywróceniu **jest** (poll znajdzie zaległe, jeśli filter nie został w międzyczasie ich zmodyfikował).
  - `to_email` / `to_header` — Phase 1 guide notuje, że nie zawsze jest bezpośrednio dostępne (czasem trzeba parsować headers). Dla wielu skrzynek brand: forward na jeden inbox + label, albo wiele Watch modules.
- **Koszt operacyjny.** Każdy poll cycle z **znalezionymi** itemami = ops; bez itemów nie powinien naliczać (`[do potwierdzenia]` na obecnym planie EVSO).

### MK-3. Data Stores

- **Co potrafi.** Persistent KV-store wewnątrz Make. Phase 1 **nie używa** — ale dla M2 to kluczowy komponent: trzymanie `(email → list of LEAD-IDs)`, deduplication po `Message-ID` Gmaila, polityka "ostatnia interakcja > N dni → nowy task".
- **Limity / pułapki.**
  - Limit rozmiaru per record + limit rekordów per store na planie — `[do potwierdzenia]`.
  - Brak query'owania innym niż klucz + simple filters. Nie jest to baza relacyjna. Brak transakcji ani CAS — dwie równoległe execucje mogą sobie przydeptywać.
  - Bardzo dobre na "lookup po emailu". Słabe na "lookup po fuzzy similarity" (do tego ClickUp search albo external search).
- **Koszt operacyjny.** Wliczone w plan, ale każda operacja na store = ops.

### MK-4. ClickUp module (search / get / create / update / comment / attachment)

- **Co potrafi.** Phase 1 używa "Create a Task" + Custom Fields mapping. Dostępne także: Search Tasks (filter by Custom Field value), Get Task, Update Task, Add Comment, Upload Attachment, List Tasks. Search Tasks po wartości pola EMAIL jest **kluczową zdolnością dla M2** (matching: czy istnieje task z tym emailem).
- **Limity / pułapki.**
  - Search Tasks zwraca paginowane wyniki, max ~100 per page. Przy 100/tydz to luźne, ale skalowalność do 10k aktywnych leadów wymaga dobrej filtracji (np. tylko nie-archive + ostatnie X dni).
  - Update Task z Custom Fields — Phase 1 baseline, działa stabilnie. Field IDs trzeba retrievować raz i hardcode'ować (lub trzymać w data store).
  - "Add Comment as email-in-task" — `[do potwierdzenia]` czy Make module wprost wystawia opcję "post jako wiadomość email do thread'u taska", czy musimy iść przez SMTP do `<id>@mg.clickup.com` z poziomu Make (HTTP/Email module). To krytyczne dla Problem A (jedna historia w jednym miejscu, w sekcji email a nie zwykłych komentarzy).
- **Koszt operacyjny.** 1 ops per call. Search + Update + Comment = 3 ops na pojedynczy email do istniejącego taska. Przy 100/tydz × ~4 wezwania średnio = ~1600 ops/mc tylko na M2 ścieżce dopasowania.

### MK-5. Routery + Filters + Aggregators / Iterators

- **Co potrafi.** Phase 1 używa router'a (form vs error fallback). Dla M2: router na 3+ ścieżki ("nowy email → match found → comment do taska" / "match not found → create new task" / "ambiguous → escalate"). Iterator do listy znalezionych tasków przy disambiguacji.
- **Limity / pułapki.**
  - Router + filter to deklaratywne if/else, nie pełna logika — przy złożonej polityce dezambiguacji szybko robi się "spaghetti scenariusz", trudny w utrzymaniu.
  - To miejsce, gdzie Faza 4 ekspertów (Tomasz Sowa) będzie pytać "kiedy to jest stabilne, kiedy się rozjeżdża".
- **Koszt operacyjny.** Same routery — 0 ops. Każda gałąź — operations swoje moduły.

### MK-6. HTTP module (custom API call)

- **Co potrafi.** Phase 1 używa go do `https://api.anthropic.com/v1/messages` (bez natywnego Anthropic modułu). Pełna kontrola nad headerami, body, parsing odpowiedzi. Idealny do mikroservisu (Vibe coding) — Make woła nasz endpoint.
- **Limity / pułapki.**
  - Timeout default ~40s — przy dłuższym przetwarzaniu trzeba split na "submit" + "callback".
  - Brak retry logiki natywnie (poza error handler "Resume" / "Break"). Phase 1 wzorzec: error handler → fallback `KLASYFIKACJA_AI = Do weryfikacji`. Ten wzorzec **musi się powtórzyć** w M2.
- **Koszt operacyjny.** 1 ops per call.

### MK-7. Error handlers ("Resume", "Rollback", "Commit", "Break", "Ignore")

- **Co potrafi.** Phase 1 baseline: HTTP Anthropic ma error handler typu `Resume` z fallbackiem `do_weryfikacji`. Make pozwala na error handling per-module. Notyfikacja email do operatora przy fatal error.
- **Limity / pułapki.**
  - Error handler **Resume** wymaga ustawienia wartości fallback dla zmiennych — jeśli się o to nie zadba, dalsze moduły dostają `null`.
  - "Break + retry" istnieje, ale liczba retries jest płaska — przy chronicznym 429 z Anthropic nie radzi sobie z exponential backoff bez ręcznego zegara.
- **Koszt operacyjny.** Każdy retry = ops.

### MK-8. Scheduled vs instant scenarios

- **Co potrafi.** Form intake — instant (webhook). Email intake — scheduled (Gmail watcher co 5 min). Obie ścieżki współistnieją w Phase 1.
- **Limity / pułapki.** Scheduled = bound to plan: minimalna częstotliwość zależy od planu (1 min, 5 min, 15 min). Instant nie ma tego limitu, ale jest płatny per ops.
- **Koszt operacyjny.** Scheduled liczy ops nawet bez nowych itemów (zależy od modułu — Gmail Watch z `is:unread` zwykle ops dopiero przy znalezieniu).

### MK-9. Blueprint export/import

- **Co potrafi.** Eksport scenariusza do JSON (autorytatywny stan zgodnie z `CLAUDE.md`). Import do innego Workspace.
- **Limity / pułapki.** Connections (OAuth, API keys) **nie są** częścią eksportu — trzeba odtwarzać po stronie target. Mapping do Custom Fields ID też jest powiązany z konkretnym ClickUp connection.
- **Koszt operacyjny.** Brak.

---

## 4. Anthropic API (Claude Haiku 4.5) — inwentaryzacja zdolności

Model produkcyjny: `claude-haiku-4-5-20251001` (per `CLAUDE.md`). Ceny poniżej to ujęcie rzędu wielkości — przy weryfikacji w Fazie 5 trzeba potwierdzić aktualną tabelę cenową Anthropic.

### AN-1. Klasyfikacja (deterministyczna, JSON output)

- **Co potrafi.** Phase 1 baseline: prompt zwraca JSON `{klasyfikacja, confidence, podsumowanie}`. Latencja ~1–3s dla 300 tokenów output. Spójność wysoka przy dobrze zdefiniowanym enum.
- **Limity / pułapki.**
  - Czasem owijka w markdown code fence — Phase 1 ma defensywny regex strip.
  - "Confidence" jest **self-reported** przez model — nie jest matematycznie kalibrowane. Trzeba ostrożnie z progami (Faza 2 to rozstrzygnie).
  - Bez few-shot examples dla edge cases jakość spada na nietypowych zapytaniach (Phase 1 prompt jest minimalistyczny — w M2 możliwe rozszerzenie).
- **Koszt operacyjny.** Haiku 4.5 — koszt rzędu kilku dolarów za milion tokenów input, kilkunastu za output (`[do potwierdzenia]` aktualne ceny przy konfiguracji). W praktyce: pojedyncza klasyfikacja 500 input + 200 output = ułamek centa. 100/tydz → poniżej dolara/mc. Klasyfikacja **kosztowo nieistotna** w skali EVSO.

### AN-2. Ekstrakcja strukturalna z nieustrukturyzowanego tekstu

- **Co potrafi.** Phase 1 "Email Intake": z body maila wyciągnąć miasto, datę, godzinę, liczbę osób, telefon, typ imprezy, pakiet. Działa solidnie na większości typowych zapytań (Phase 1 doświadczenie produkcyjne; jakość w edge cases wymaga benchmarku).
- **Limity / pułapki.**
  - Halucynacje pól, których nie było w mailu — trzeba mieć "jeśli niepewny, zostaw null". Dobrze sformułowany prompt to mityguje.
  - Polski język + slang + skróty regionalne — Haiku radzi, ale jakość rośnie z few-shot.
- **Koszt operacyjny.** Jak w AN-1, kilkukrotnie wyższy przy dłuższych mailach.

### AN-3. Semantyczne dopasowanie (matching email ↔ task)

- **Co potrafi.** Najistotniejsza dla M2 zdolność: dla nowego maila i N kandydatów (taski z tym samym emailem, ewentualnie wynik wstępnej filtracji), oceń który (jeśli którykolwiek) jest "this is the right task" + zwróć confidence + uzasadnienie. Działa to dobrze gdy każdy task ma sensowne podsumowanie / kontekst — Phase 1 daje `AI_PODSUMOWANIE`, które wpina się jako wejście dla M2.
- **Limity / pułapki.**
  - Im więcej kandydatów, tym dłuższy prompt → wyższy koszt + niższa precyzja. Praktyczny limit: 3–5 kandydatów. Powyżej — pre-filtracja po kluczach (np. ostatnie 90 dni, ten sam status, to samo miasto/usługa).
  - Self-reported confidence — patrz AN-1. Faza 2 wybierze próg.
  - "Ten sam klient, dwa osobne wyjazdy" (przykład Pani Kasia z Phase 0) — tu AI musi rozumieć kontekst tematu i treści, nie tylko emaila. To jest realne zadanie dla Haiku, ale **wymaga benchmarku** — Phase 1 nie ma tego use case'u w produkcji.
- **Koszt operacyjny.** Większy niż prosta klasyfikacja (prompt zawiera N podsumowań kandydatów). Przy 100/tydz wciąż rzędu kilku dolarów/mc; przy 1000/tydz — kilkanaście do kilkudziesięciu. Realny, ale nie blokujący.

### AN-4. Sugerowanie tytułu maila wychodzącego (Problem D)

- **Co potrafi.** Krótki call: "dla taska z polami X i historii Y, zaproponuj tytuł maila wychodzącego w polskim, w spójnej konwencji". 50 tokenów output, latencja <1s.
- **Limity / pułapki.** Bardziej UX integration question (gdzie i kiedy się odpalić — ofline w trakcie tworzenia taska i wpisać do Custom Field, czy live w composerze) niż AI question. Patrz problem mapping w sekcji 6.
- **Koszt operacyjny.** Trywialny.

### AN-5. Multi-turn / agentic loops + Tool use

- **Co potrafi.** Anthropic API obsługuje tool use (function calling) i wieloetapowe rozmowy. Można zbudować agenta który: (a) szuka tasków po emailu (tool: ClickUp Search), (b) ocenia kandydatów, (c) loguje decyzję.
- **Limity / pułapki.**
  - W Make trudno orkiestrować multi-turn natywnie — typowo robisz albo "single-shot z dużym promptem" (Phase 1 wzorzec), albo wynosisz orkiestrację do mikroservisu (Vibe). Make HTTP module nie jest najwygodniejszym miejscem na 5-step agent loop.
  - Tool use zwiększa ilość roundtripów → koszt + latencja rośnie.
- **Koszt operacyjny.** 2–5× single-shot zależnie od liczby kroków.

### AN-6. Prompt caching

- **Co potrafi.** Anthropic obsługuje cache dla long static prefixes (system prompt, few-shot examples). Daje ~90% redukcji input cost dla powtarzalnych prefiksów + redukcję latencji.
- **Limity / pułapki.**
  - 5-min TTL — przy 100/tydz potok jest sporadyczny, więc cache hit będzie niski. Przy 1000/tydz (skala Fazy 6) staje się znaczący.
  - Wymaga jawnego oznaczenia prefiksu jako `cache_control` — Phase 1 tego nie używa, M2 powinien rozważyć dla system promptu matchera (jeśli jest długi z few-shot).
- **Koszt operacyjny.** Ma sens powyżej ~5 calls/min average.

### AN-7. Rate limits

- **Co potrafi.** Plan EVSO `[do potwierdzenia]` — typowo Tier 1/2 daje kilkadziesiąt RPM dla Haiku. Dla 100/tydz to absurdalnie luźne, dla 1000/tydz nadal OK.
- **Limity / pułapki.** 429 = exponential backoff. Make natywnie tego nie ma — fallback to "Do weryfikacji" lub queue do retry przez Data Store.

---

## 5. Vibe coding (mikroservis) — inwentaryzacja zdolności

### VC-1. Niestandardowa logika fuzzy matchingu / disambiguacji

- **Co potrafi.** Cloudflare Worker / Vercel function / mała Express app: dostaje `(new_email_payload, candidate_tasks)`, wraca `{matched_task_id, confidence, reasoning, action: append|create|escalate}`. Wewnątrz: deterministyczne heurystyki (czas od ostatniej interakcji, status taska, miasto pasuje) + opcjonalnie call do Anthropic API. Pełna kontrola nad logiką.
- **Limity / pułapki.**
  - Wprowadza nową technologię w stack EVSO (Soft constraint S1: utrzymywalność dla małego zespołu). Bartek + handlowcy bez wiedzy technicznej — jeśli mikroservis padnie, kto debug'uje?
  - Wymaga deploy pipeline + monitoring + secrets management (Anthropic key) — to jest realny koszt operacyjny *i* poznawczy.
  - Ale: można go zbudować jako **single-file deployment** (Cloudflare Worker, ~100 LOC), co radykalnie zmniejsza koszt utrzymania. To jest realne dla EVSO — jeden kierunek architektoniczny w Fazie 3.
- **Koszt operacyjny.** Cloudflare Worker free tier (~100k req/dzień) ≫ EVSO need. Vercel hobby — analogicznie. Realny koszt: 0 PLN/mc dla skali EVSO.

### VC-2. Persistent state poza Make Data Store

- **Co potrafi.** Małe SQLite (Cloudflare D1) / KV (Cloudflare KV) / Upstash Redis. Lepsza ergonomia niż Make Data Store dla logik typu "lista wszystkich threadId per email + last_seen_at + flags".
- **Limity / pułapki.**
  - Duplikuje to, co Make Data Store potrafi. Jeśli Make wystarcza (a dla 100/tydz wystarcza) — mikroservis jest overkill.
  - Operability: dwa miejsca prawdy = dwa miejsca, gdzie można zgubić dane.
- **Koszt operacyjny.** Free tier wystarczy.

### VC-3. Custom UI (lekki dashboard, custom inbox, etc.)

- **Co potrafi.** React/Next.js page, która łączy się z ClickUp API + Gmail API i wyświetla "kolejka do weryfikacji" w wygodniejszej formie niż View w ClickUp.
- **Limity / pułapki.**
  - **Najsłabsze uzasadnienie dla M2.** ClickUp Views + Inbox + AI Fields powinny rozwiązać UX. Custom UI = nowa powierzchnia, nowy auth, nowy training dla handlowców.
  - Reguła zespołu (S2): "tylko jeśli prostsze, stabilniejsze, bardziej użyteczne niż alternatywa". UI custom przegrywa z natywnym ClickUp dla M2 chyba że alternatywa jest udokumentowanie zła. Faza 4 to rozstrzygnie.
- **Koszt operacyjny.** Realny: deployment + maintenance + occasional rewrite.

### VC-4. Webhook pośrednik / proxy (między Gmail a Make)

- **Co potrafi.** Mały serwis, który odbiera push z Gmail (Gmail Push Notifications via Pub/Sub) i wpina to do Make jako webhook — daje **prawdziwy real-time** zamiast 5-min poll.
- **Limity / pułapki.**
  - Dla 100/tydz różnica 5 min vs <30s nie jest critical — ale dla SLA ofertowego może być (klient odbiera w 1.5h vs 5 min).
  - Wprowadza Pub/Sub auth + service account. Operability cost.
- **Koszt operacyjny.** Free tier GCP wystarczy.

### VC-5. Kiedy NIE warto Vibe coding

- ClickUp Automations / Make natywnie robi to **prosto** (CU-6, MK-5).
- Logika decyzyjna mieści się w pojedynczym promptcie do Haiku z parsowaniem JSON (AN-1..3).
- Odpowiedzialny za to jest Bartek solo i nie ma czasu na pielęgnację mikrousługi.
- Nie ma jasnej historyjki "co się psuje, gdy serwis padnie" + "kto wtedy klika ratunek".

---

## 6. Capability × Problem matrix

> Każda komórka: ✅ adresuje + sposób / 🟡 częściowo + brak / ❌ nie + workaround.
> Problemy A–F z `01_Phase0_Problem_Map.md`.

| Problem | ClickUp Business | Make.com | Anthropic API | Vibe coding (microservice) |
|---|---|---|---|---|
| **A — Fragmentacja historii** | ✅ Jeśli email trafi do taska (CU-1 + CU-9) — task staje się single source of truth (description + activity + comment-as-email + composer). Brain (CU-7) agreguje treść do Q&A. | ✅ Make jako warstwa, która **doprowadza** maile do taska (search po EMAIL + add comment / SMTP do `<id>@mg.clickup.com`). Bez Make → CU-1 łapie tylko in-thread. | 🟡 AI nie jest **przyczyną** rozwiązania, ale poprawia użyteczność: AN-1..2 daje `AI_PODSUMOWANIE`, dzięki czemu otwarcie taska natychmiast pokazuje kontekst. | 🟡 Niepotrzebne dla samego "fragmentacja". Może być potrzebne dla custom logiki agregacji (rzadkie przypadki: kilka tasków tego samego klienta → merge view), ale to M3+. |
| **B — Identyfikacja klienta ponad kanałami** | 🟡 Custom Field EMAIL (CU-5) jest kluczem. Search Tasks po EMAIL przez Make (MK-4) — działa, ale to ClickUp jako *target*, nie jako *matcher*. CU-3 (AI Agents) mogłyby teoretycznie matchować, ale plan Business + brak doświadczenia = wysokie ryzyko. | ✅ MK-4 Search Tasks po EMAIL + MK-3 Data Store (cache `email → [task_ids]` + `last_seen_at`) + MK-5 router do "match / new / escalate". Naturalne miejsce dla matchingu. | ✅ AN-3 — semantyczne rozstrzyganie wśród kandydatów (powracający klient po roku vs kontynuacja sprawy: AI patrzy na temat + treść + datę ostatniej interakcji). Konieczne dla Anti-assumption #2. | 🟡 Tylko jeśli logika polityki "powracający klient po N dniach = nowy task" robi się złożona (np. kalendarzowo: "tydzień przed eventem nie tworzymy nowego taska"). Dla prostej logiki Make + AN wystarcza. |
| **C — Niepełna integracja emaili z CRM** | 🟡 CU-1 (per-task `mg.clickup.com`) **łapie tylko in-thread**. CU-2 (list-level) tworzy nowy task — bez matchingu. Native komponenty same z siebie nie domykają problemu. | ✅ MK-2 (Gmail Watch) + MK-4 (Search Tasks po EMAIL + Add Comment-as-email lub SMTP do `<id>@mg.clickup.com`) zamyka pętlę: każdy email → albo do istniejącego taska, albo nowy. **To miejsce, gdzie M2 się dzieje.** | ✅ AN-3 dla disambiguacji + AN-2 dla wzbogacenia treści. Bez AI matching jest tylko po emailu — wtedy "powracający klient" trafia do złego (archiwalnego) taska. | 🟡 Opcja: VC-4 dla real-time push zamiast polling. Tylko jeśli SLA wymaga <1 min. Dla EVSO 5 min poll OK na start. |
| **D — Sugerowanie tytułu wątków wychodzących** | 🟡 Composer w tasku (CU-9) jest miejscem akcji, ale **nie ma natywnej sugestii tytułu**. AI Fields (CU-4) mogą "wyliczyć" sugerowany tytuł na poziomie taska — handlowiec kopiuje. Brain (CU-7) — ad hoc Q&A. | 🟡 Make może wstępnie wygenerować "sugerowany tytuł" przy tworzeniu taska (już przy intake) i wpisać do Custom Field `SUGEROWANY_TYTUŁ_MAILA`. To nie jest "live sugestia w composerze", ale jest *good enough*. | ✅ AN-4 — krótki call, deterministyczny output. Trywialny koszt. | ❌ Vibe coding tutaj to overkill — chyba że budujemy custom Chrome extension nad ClickUp composerem. Zdecydowanie disqualified by S2. |
| **E — Brak powiadomienia o nowych wiadomościach** | 🟡 CU-8 (Inbox + push) działa, ale **tylko gdy user jest assignee/watcher/@mention**. Wpadnięcie maila do taska samo nie triggeruje notyfikacji jeśli nikt nie jest watcherem. CU-6 (Automations) na trigger "comment added" / "email received" może zrobić @mention assignee — to zamyka ścieżkę. | ✅ Po dodaniu emaila do taska Make może równolegle: (a) trigger ClickUp comment z @mention assignee, (b) opcjonalnie ping na Slack/Telegram bridge. | ❌ AI nie jest właściwym narzędziem — to deterministyczny routing. | 🟡 Tylko jeśli zespół chce powiadomień przez kanał spoza ClickUp (np. SMS, WhatsApp) i ta integracja nie jest dostępna w Make natywnie. Dla Telegram/Slack: Make ma natywne moduły. |
| **F — Skalowalność ręcznego zarządzania** | 🟡 Każde zaautomatyzowane przypisanie maila do taska = oszczędność 1–3 min handlowca. Same ClickUp natywne mechanizmy (CU-1, CU-9) już dają częściową ulgę dla in-thread case. | ✅ Make jest miejscem orkiestracji — im więcej decyzji deleguje się tu (z AI), tym mniej ręcznego klikania. | ✅ AN-1..3 to **operator wykonujący** te ręczne decyzje. Per call ułamek centa × 100/tydz ≈ kilka dolarów/mc — ekonomicznie znacząco tańsze niż minuty handlowca. | 🟡 Dla bardzo specyficznej polityki (np. "klient z firmy X zawsze do handlowca Y") — ale to do M3+. |

**Sanity-check pokrycia:**
- Każdy z 6 problemów ma ≥1 mapping na konkretną zdolność → ✅
- Wszystkie 24 komórki wypełnione → ✅

---

## 7. Najistotniejsze obserwacje wpadające do Fazy 2

Pojawiają się jako bezpośrednie konsekwencje sekcji 2–6 i będą rozstrzygane w Decision Matrix:

1. **Make jest naturalnym miejscem orkiestracji M2**, bo Phase 1 już tam żyje, error handling jest sprawdzony, a koszty (ops + Anthropic) są niskie nawet przy skali 10×. Ciężar argumentacji spada na "dlaczego mielibyśmy *nie* używać Make jako mózgu".
2. **ClickUp Agents (CU-3) jest realną alternatywą**, ale wprowadza nową, słabo przetestowaną w EVSO warstwę i lock-in na ClickUp. Faza 4 (Marta) to zbada.
3. **Mikroservis (VC-1..4) ma sens tylko jeśli logika polityki "ten sam klient / nowy wyjazd" jest na tyle złożona, że nie mieści się w jednym Anthropic call'u + Make routerze.** Dla 100/tydz raczej nie. Faza 3 zarysuje wariant C jako kontrolę.
4. **Email-to-Task per task (CU-1) zostaje jako "darmowa baseline" dla in-thread.** Pytanie nie brzmi "czy go używamy", tylko "czy oprócz niego mamy warstwę matchującą poza-wątek".
5. **AI confidence (AN-1, AN-3) jest self-reported.** Próg w Fazie 2 musi być wybrany ostrożnie — nie ma kalibracji statystycznej. Mocno kandyduje "0.85 → auto, 0.6–0.85 → escalate, <0.6 → nowy task" jako baseline, ale wymaga walidacji empirycznej w testowym środowisku.
6. **Real-time vs batch.** Dla M2 baseline = Gmail Watch co 5 min (MK-2). Push przez VC-4 jest opcją, ale SLA EVSO nie wymaga tego — Bartek to zdefiniuje w Fazie 2.
7. **Idempotentność** musi przyjść z Data Store (MK-3) trzymającego `Message-ID` ostatnio przetworzonych, niezależnie od strategii "mark as read" — bo "mark as read" to destruktywny side effect, którego boimy się dla pierwszych iteracji.
8. **Notyfikacje (Problem E)** = ClickUp comment z @mention przez Automation (CU-6) lub Make. Slack/Telegram to nice-to-have, nie must-have.
9. **Phase 1 baseline error handling pattern** ("HTTP fail → fallback Do weryfikacji + ewentualny powiadom email") jest wzorcem, który **kopiujemy** dla każdego nowego AI calla M2.

---

## 8. Otwarte pytania (`[do potwierdzenia]` zebrane)

Te punkty wymagają potwierdzenia lub eksperymentu zanim Faza 5 (architektura final) zamknie ich miejsce. Nie blokują Fazy 2/3/4 — można je sparkować i jeśli Faza 2 wybierze poziom ambicji wymagający odpowiedzi, Bartek je dośledzi.

| # | Pytanie | Krytyczność |
|---|---|---|
| Q1 | Czy ClickUp Business obejmuje produkcyjne użycie AI Agents (CU-3) w zakresie potrzebnym dla matchera, czy to feature Enterprise? | High (jeśli Faza 4 wybierze Kierunek A) |
| Q2 | Czy Make ClickUp module wprost obsługuje "Add Comment as email-in-task" tak, że trafia do thread'u maili (a nie zwykłych komentarzy)? Jeśli nie — czy potrzebny jest call do `<id>@mg.clickup.com` z poziomu Make HTTP / Email module? | High (każdy kierunek) |
| Q3 | Limity AI uses i automation runs na planie ClickUp Business EVSO — dokładne kwoty/mc + over-cap. | Medium |
| Q4 | Limit ops/mc na obecnym planie Make EVSO + jakie są koszty up-tier'u jeśli M2 + Phase 1 razem to przekraczają. | Medium |
| Q5 | Tier Anthropic API EVSO (rate limits per minute) + aktualna tabela cen Haiku 4.5. | Low (skala niska) |
| Q6 | Czy Gmail Watch z `is:unread` zlicza ops tylko przy znalezieniu maila, czy każdy poll cycle? | Low (porządkowe) |
| Q7 | Czy publiczne `kontakt@*` w Dhosting da się forwardować na `automatyzacjaevso@gmail.com` (test env) lub bezpośrednio na list-level email ClickUp w prod env, bez ruszania samych skrzynek Dhosting? | Medium (Faza 5 — migration plan) |
| Q8 | Strategia idempotentności: Gmail "mark as read" vs Make Data Store po `Message-ID` — które wybieramy w M2 baseline? | Medium (Faza 5) |

---

## 9. Kryterium gotowości Fazy 1 (self-check)

- [x] Inwentaryzacja w 4 obszarach (ClickUp Business / Make.com / Anthropic API / Vibe coding) — sekcje 2–5, każda ze zdolnościami opisanymi pod kątem *co potrafi / limity / koszt*.
- [x] Capability × Problem matrix wypełniona (24 komórki = 6 problemów × 4 obszary) — sekcja 6.
- [x] Każdy z 6 problemów ma ≥1 mapping na konkretną zdolność (sanity-check w sekcji 6).
- [x] Sekcja "limity i pułapki" jest niefantazjowana — oparta na Phase 1 baseline (`docs/Plans/Done/EVSO_Implementation_Guide_Phase1.md` + autorytatywny `docs/Make/EVSO_Form_Intake_v2.blueprint.json` opisany module-by-module w guide), dokumentacji ClickUp/Make/Anthropic odzwierciedlonej w Phase 1, oraz jasno oznaczonych `[do potwierdzenia]` tam, gdzie pewności nie ma. Otwarte pytania zebrane w sekcji 8.
- [x] Brak referencji do `archive/*`.

**Status: spełnione.**
