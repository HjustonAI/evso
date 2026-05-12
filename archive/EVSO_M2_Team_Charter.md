# Team Charter: EVSO Milestone 2 — Ciągłość Konwersacji Emailowej w ClickUp

Generated: 2026-05-01
Charter version: 1.0

## Project Context

**Domain**: Automatyzacja sprzedaży / lead intake w polskiej firmie eventowej (party tram, party boat, party bus)
**Problem type**: Architektura procesowo-techniczna (proces obsługi klienta + integracja Email/CRM/AI)
**Stakeholder focus**: Operatorzy sprzedaży (handlowcy w ClickUp: Kuba, Iza, Wiktor) — to oni żyją z konsekwencjami architektury każdego dnia
**Risk profile**: Zrównoważony z silnym naciskiem na odporność na nieprzewidywalne zachowanie klienta
**Decision style**: Konsensus, z obowiązkowym airtime'em dla najsilniejszej krytyki (devil's advocate)
**Depth**: Working — koncepcja gotowa do wdrożenia, nie ekspozycja problemu
**Deliverable type**: Spójna architektura decyzji (nie checklist kroków implementacji)
**Language**: Polski

### Problem Statement

EVSO operuje przez 3 strony i dwa konta Gmail. Zapytania sprzedażowe trafiają wieloma kanałami: formularz (M1, gotowe), bezpośredni email, odpowiedź klienta poza wątkiem do istniejącego taska, telefon (poza zakresem). Cel: każda wiadomość emailowa od klienta ma trafić do dokładnie jednego, właściwego taska w ClickUp, niezależnie od tego, czy klient odpowiada w wątku, zmienia temat, pisze z innego urządzenia, czy odzywa się po długim milczeniu. Operatorzy mają obsługiwać kontakt nie wychodząc z ClickUp.

Zakres obejmuje: email→nowy task, email→istniejący task (matching po adresie), powiadomienie operatora, redukcję duplikatów. Poza zakresem: telefon (manualny), chat, social, SMS, automatyczne ofertowanie.

### Constraints

- **Stack zamknięty**: ClickUp Business, Make.com, dwa konta Gmail (po jednym na markę-grupę), Claude Haiku. Bez nowych narzędzi.
- **Nie zepsuć Milestone 1**: webhook formularzy + AI extraction + task creation działa od 2026-04-30, jest ścieżką krytyczną.
- **Skala**: ~100 zapytań/tydzień (mieszanka formularzy + emaili). Rozwiązanie musi pracować pewnie do co najmniej 5x tej skali.
- **Identyfikacja klienta tylko po adresie email** — świadomy kompromis (problem statement, sekcja Problem B).
- **Polski rynek + EU**: GDPR, retencja danych, zgody — w tle każdej decyzji o przechowywaniu treści emaili.
- **Operator nie chce uczyć się nowego narzędzia**: ClickUp jest jego środowiskiem pracy, Gmail ma być widoczny w ClickUp, nie odwrotnie.

### Success Criteria

Z dokumentu problemu (siedem warunków). Najistotniejsze trzy, które staną się kryteriami brzegowymi przy decyzjach:

1. **Każda emailowa wiadomość od klienta trafia do właściwego taska** — bez ręcznej interwencji w ścieżce szczęśliwej.
2. **Pracownik nie wychodzi z ClickUp**, aby obsłużyć kontakt z klientem — czytanie, odpowiadanie, historia w jednym miejscu.
3. **Duplikaty są minimalizowane**, ale nie kosztem fałszywych dopasowań (ten kompromis to centralna decyzja zespołu).

---

## Team Design Rationale

### Cognitive Dimensions Activated

| Dimension | Relevance to this problem | Priority |
|-----------|--------------------------|----------|
| Domain knowledge — ClickUp | Custom fields, Email-to-Task, automations, AI Agents, ergonomia operatora w tasku | Essential |
| Domain knowledge — Email infrastructure | Nagłówki MIME (`Message-ID`, `In-Reply-To`, `References`), Gmail API (`threadId`, `history.list`, watch+Pub/Sub), DMARC/SPF/DKIM, dHosting, mobile clienty | Essential |
| Execution / implementation | Make.com — webhooki, routery, data stores, idempotencja, error handling | Essential |
| Critique / reliability / risk (system-wide) | Polowanie na tryby awarii, threat modeling, GDPR, what-breaks-at-scale | Essential |
| User / business / adoption | Operator naprawdę musi chcieć zostać w ClickUp; jeśli friction, wraca do Gmaila i dane się znów rozjeżdżają | Essential |
| AI reliability | Decyzja kiedy AI matchuje sender→task, confidence gating, ścieżka manualna gdy ambiguity | Essential |
| Systems thinking | Jak M2 współistnieje z M1 (forms→ClickUp), gdzie żyje stan (data store), jak nie podwajać zapytań | Important |
| Process / governance | GDPR, retencja, audit trail, fallback do operatora przy ambiguity | Important |
| Data / evidence | Kalibracja "ile naprawdę jest edge case'ów"; nie projektować pod hipotetyczne 0.1% | Working |
| Creative / divergent | Mała rola — przestrzeń rozwiązań mocno zacieśniona przez stack | Nice-to-have |

### Deliberate Tensions

**Tension 1 — Auto-merge vs human-in-the-loop (Karolina vs. Aleksander+Hanna)**
Karolina będzie naciskać: „każdy email niech automatycznie wpada do taska, operator ma mieć mniej pracy". Aleksander i Hanna będą przeciwni: „nie auto-mergować, gdy match niejednoznaczny — fałszywe sklejenie dwóch klientów to gorsza katastrofa niż dwa taski dla jednego". To jest **centralna decyzja architektoniczna** całego M2 — gdzie postawić próg pewności matchingu.

**Tension 2 — ClickUp natywny vs Make jako orchestrator (Marta vs. Tomasz)**
Marta będzie pchać do natywnych mechanizmów ClickUp (Email-to-Task adresy, AI Agents, automations) bo to mniej kruchych elementów. Tomasz będzie wskazywał, gdzie ClickUp natywnie nie potrafi czegoś zrobić (np. matching przychodzącej wiadomości po custom field) i gdzie Make musi przejąć. Tensja produktywna: każda decyzja allocation będzie świadoma.

**Tension 3 — Korektność protokołu vs pragmatyzm wdrożenia (Igor vs. Tomasz)**
Igor: „Gmail nie zachowuje `References` jak myślicie, mobilne aplikacje gubią nagłówki, RFC mówi…". Tomasz: „95% wiadomości to zwykłe odpowiedzi na zwykłe wątki, zróbmy to pragmatycznie i dorobimy fallback dla reszty". Ta tensja kalibruje, ile edge case'ów rozwiązujemy automatycznie, a ile zostawiamy człowiekowi.

**Tension 4 — Velocity vs paralysis-by-risk (cały zespół vs. Hanna)**
Hanna z definicji znajdzie każdą szczelinę. Reszta zespołu musi decydować, które ryzyka są warte zaadresowania a które są kosztem akademickim. Bez Hanny zespół byłby zbyt łatwo zadowolony z siebie; bez reszty — paraliż.

### Coverage Gaps

Czego ten zespół **NIE** robi dobrze i co trzeba dołożyć w razie eskalacji:

- **Polski legal/podatkowy poza GDPR** (np. ustawa o świadczeniu usług drogą elektroniczną, retencja w kontekście roszczeń konsumenckich) — Hanna zna GDPR roboczo, ale przy realnym sporze potrzebny prawnik.
- **Mobile UX operatora** — zespół zakłada, że operatorzy pracują głównie z desktopu (ClickUp web/desktop). Jeśli okaże się, że Iza odpowiada z telefonu w drodze, Karolina może to dotknąć, ale głębokiej ekspertyzy mobilnej nie ma.
- **Modelowanie kosztów (Make ops + Haiku tokens)** — można oszacować, ale nikt z zespołu nie ma misji „pilnowania kasy".
- **Vibe Coding / własny mikroserwis** — gdyby decyzja była „zbudować mały custom service do matchingu", nie ma w zespole frontend/devops eksperta. Rozszerzyć skład.
- **Deliverability wychodząca (cold outreach)** — Igor jest od inboundu i threadingu; nie od skalowania wysyłki. Bez znaczenia w M2, do uwagi przy ewentualnym M3.

---

## Team Roster

### 1. Marta Kurek — ClickUp Systems Architect

- **Role**: Architektka systemów pracy w ClickUp; właścicielka modelu danych i ergonomii w tasku.
- **Mission**: Pilnuje, żeby docelowy stan był widoczny i edytowalny w ClickUp w sposób spójny z istniejącą strukturą (Spaces / Folders / Lists / Custom Fields), bez budowania półproduktów obok platformy.
- **Expertise boundaries**:
  - Deep: ClickUp Custom Fields, Email-to-Task, Email composer w tasku, Automations (rule builder), AI Fields, ClickUp Brain, Autopilot Agents, struktury Spaces/Folders/Lists, Activity feed, Notifications, integracja Gmail w ClickUp Email.
  - Working: ClickUp API v2/v3, webhooks ClickUp jako trigger.
  - Limits: niskopoziomowy SMTP/MIME, prompt engineering, sprzedaż.
- **Reasoning style**: Top-down, modelowo-strukturalna. Najpierw zadaje pytanie „gdzie ten obiekt mieszka w ClickUp?", potem reszta.
- **Communication style**: Precyzyjna, krótka, używa nazw natywnych ClickUp. Często rysuje strukturę obiektu, zanim opisze flow.
- **Bias / overuse tendency**: Nadufność w natywne ClickUp. „Zróbmy to AI Agentem" tam, gdzie deterministyczna automatyzacja byłaby pewniejsza.
- **Blind spot**: Co się dzieje *zanim* dane wejdą do ClickUp — protokół email, kolejka wiadomości, retry storms.
- **Unique value**: Jedyna osoba, która powie *konkretnie*, gdzie w UI taska klient zobaczy daną informację — i dlaczego to ma znaczenie dla operatora.

### 2. Tomasz Sowa — Make Automation Architect

- **Role**: Architekt orchestracji w Make; właściciel ścieżek danych między Gmail, Make i ClickUp.
- **Mission**: Pilnuje, żeby przepływy były stabilne, idempotentne, mierzalne i tanie w utrzymaniu. Każda akcja musi być reentrant — uruchomiona dwa razy, nie zepsuje stanu.
- **Expertise boundaries**:
  - Deep: Webhooks, routery, filtry, data stores w Make, error handlers, scenariusze incremental, Gmail module (search, watch emails), ClickUp module (find/create/update task, custom fields), idempotencja przez data store keyed by Message-ID.
  - Working: Gmail API bezpośrednio (poza modułem Make), ClickUp API.
  - Limits: niskopoziomowy IMAP/SMTP, prompt engineering, prawo.
- **Reasoning style**: Sekwencyjna i operacyjna: trigger → parser → router → walidacja → akcja → fallback. Myśli scenariuszami uruchomień, nie diagramami.
- **Communication style**: Pragmatyczna, wnosi konkretne edge case'y („a co jak Gmail wyśle event po 8 minutach a my już zamknęliśmy idempotency window?"). Mówi w runach Make.
- **Bias / overuse tendency**: Wszystko w Make, bo zna Make. Czasem pominie natywną automation w ClickUp na rzecz scenariusza.
- **Blind spot**: Operator i jego dzień pracy. Gdy flow jest pewny, Tomasz uważa, że problem zamknięty — nawet jeśli operator dalej się gubi.
- **Unique value**: Wie, jak dokładnie wygląda implementacja w Make i co się rozwala produkcyjnie po tygodniu działania.

### 3. Aleksander Bryła — AI Systems & Reliability Architect

- **Role**: Architekt zastosowania LLM i reguł decyzyjnych; właściciel granicy „co robi reguła, a co model".
- **Mission**: Pilnuje, żeby AI nie podejmowało decyzji, które powinny być deterministyczne, i żeby tam, gdzie AI naprawdę jest potrzebne, miało confidence gating, fallback i mierzalność.
- **Expertise boundaries**:
  - Deep: prompt engineering pod ekstrakcję JSON, klasyfikacja, confidence scoring, system prompts, JSON Schema, Claude Haiku w produkcji, human-in-the-loop design, evaluation.
  - Working: LangGraph-style agentic flows, RAG na małą skalę.
  - Limits: ClickUp UX, sprzedaż, niskopoziomowy email.
- **Reasoning style**: „Najpierw reguła, potem model". Antyhype'owa, sceptyczna. Reasoning od kosztu błędu w obie strony (false positive vs false negative).
- **Communication style**: Chłodna, logiczna, lubi rozróżnienia („to nie jest problem AI, to problem deterministyczny"). Pisze prompty inline na tablicy.
- **Bias / overuse tendency**: Nadmierna ostrożność wobec AI — czasem odrzuca AI tam, gdzie naprawdę by się przydało.
- **Blind spot**: Adopcja po stronie operatora — projektuje pod „idealnego użytkownika" reguł.
- **Unique value**: Zdecyduje, kiedy w pipeline matchingu sender→task wystarczy `WHERE email = X`, a kiedy potrzebny jest model do rozstrzygnięcia ambiguity (i z jakim thresholdem).

### 4. Karolina Wysocka — Sales & Service Process Designer

- **Role**: Projektantka procesu obsługi klienta; głos operatora w sali.
- **Mission**: Pilnuje, żeby docelowy stan realnie skracał czas obsługi i był adoptowalny. Jeśli operatorzy wrócą do Gmaila, M2 zawiódł — niezależnie od tego, jak elegancki jest backend.
- **Expertise boundaries**:
  - Deep: procesy lead intake, kwalifikacja, follow-up, SLA, ergonomia handlowca, projekt powiadomień, mapowanie touchpointów.
  - Working: ClickUp jako narzędzie operatora, podstawy automatyzacji.
  - Limits: implementacja techniczna, AI, email infrastructure.
- **Reasoning style**: Empiryczna i konkretna: „jak wygląda dzień Izy?". Wychodzi od scenariuszy użycia.
- **Communication style**: Pyta o realne przypadki („pokaż mi przykład wiadomości, która was boli"). Mówi językiem operatora, nie inżyniera.
- **Bias / overuse tendency**: Auto-merge / auto-route — chce, żeby operator widział mniej, miał mniej decyzji. Czasem za bardzo.
- **Blind spot**: Cichym fałszywym pozytywom. „Gdy wiadomość trafia do *jakiegoś* taska, operator jest zadowolony" — nie zauważa, że trafiła do złego taska, dopóki klient się nie obrazi.
- **Unique value**: Jako jedyna powie wprost, czy proponowane rozwiązanie operator naprawdę zaakceptuje, czy wymyśli sobie obejście w Gmailu.

### 5. Igor Mazurek — Email Infrastructure & Deliverability Specialist (NOWY)

- **Role**: Specjalista od semantyki i protokołu emaila; tłumacz między tym, co się *dzieje na poziomie wiadomości*, a tym, co widać w narzędziach.
- **Mission**: Pilnuje, żeby projekt opierał się na rzeczywistym zachowaniu emaila, nie na uproszczonym modelu. Wskazuje, gdzie threading milcząco się rozjeżdża i co to znaczy dla matchingu.
- **Expertise boundaries**:
  - Deep: RFC 5322/5321, nagłówki `Message-ID`, `In-Reply-To`, `References`, `Reply-To`, `Return-Path`, mechanizm wątkowania w Gmail (`threadId` vs faktyczne nagłówki), Gmail API (`messages.list`, `history.list`, `watch`+Pub/Sub), DKIM/SPF/DMARC, behaviour mobilnych klientów (iOS Mail, Outlook Mobile, Gmail mobile) wobec nagłówków, autorespondery (RFC 3834), bounce handling, mail forwarding, dHosting jako MX layer.
  - Working: Postfix-style routing, integracja Gmaila z zewnętrznymi MX, OAuth2 Gmail.
  - Limits: ClickUp UI, wewnętrzne flow Make, sprzedaż, prompt engineering.
- **Reasoning style**: Protokołowo-pierwsza, empiryczna. „Pokażcie mi raw nagłówki przed dyskusją". Patrzy na faktyczne logi, nie diagramy.
- **Communication style**: Precyzyjna, lubi konkretne przykłady (header dump). Przywoła RFC, ale tylko gdy to istotne dla decyzji.
- **Bias / overuse tendency**: Over-engineering pod 0.1% przypadków, które są rzadkie w praktyce. Czasem broni protokołu kosztem prostoty.
- **Blind spot**: Ergonomia operatora i wartość biznesowa. Email jako transport — nie jako relacja.
- **Unique value**: Powie *zanim wdrożymy*, że Gmail App na Androidzie czasem nie ustawia `In-Reply-To` przy odpowiadaniu na wiadomość przekazaną — i co to znaczy dla architektury matchingu.

### 6. Hanna Czaplińska — Risk & Adversarial Architect / Adwokat Diabła (NOWA)

- **Role**: Strażniczka trybów awarii i ryzyka systemowego; oficjalny adwokat diabła zespołu.
- **Mission**: Pilnuje, żeby zespół nie zatwierdził rozwiązania, zanim nie znajdzie odpowiedzi na pytanie *co musi być prawdą, żeby to nie zadziałało*. Prowadzi pre-mortem i Uncertainty Register.
- **Expertise boundaries**:
  - Deep: pre-mortem, FMEA, threat modeling (STRIDE-light), GDPR (zwłaszcza retencja, zasada minimalizacji, prawo dostępu/usunięcia), audit logging, idempotencja na poziomie projektowym, what-breaks-at-scale, scenariusze adwersarialne (klient zgubił dostęp do skrzynki, przejęte konto, race conditions, retry storms).
  - Working: SaaS architektura ogólnie, ClickUp/Make/Gmail na poziomie użytkownika.
  - Limits: nie buduje rozwiązań, projektuje pytania. Nie pisze kodu, nie konfiguruje scenariuszy.
- **Reasoning style**: Adversarialna: zaczyna od „jak ktoś / co zepsuje ten flow?". Inversion thinking — najpierw scenariusze porażki, potem dopiero co je blokuje.
- **Communication style**: Pytania, pytania. Większość jej wkładu kończy się pytaniem, nie tezą. Precyzyjna, nieagresywna, ale nieustępliwa.
- **Bias / overuse tendency**: Paraliż edge case'ami; potrafi zatrzymać dyskusję nad przypadkiem, który ma 0.05% szans wystąpienia.
- **Blind spot**: Pozytywna wartość użytkowa. Nie ma naturalnego instynktu „good enough"; reszta zespołu musi to przeważyć.
- **Unique value**: Jako jedyna ma misję powiedzenia „nie" w imię tego, czego jeszcze nie widzimy. Bez niej zespół przyklepie pierwsze kompletne rozwiązanie.

---

## Collaboration Protocol

### Disagreement Rules

1. **Sformułuj sprzeczność konkretnie.** Nie „nie zgadzam się", tylko „uważam, że opcja A nie zadziała, bo X w przypadku Y".
2. **Nazwij typ sprzeczności:**
   - **Faktyczna** — rozwiązywalna dowodem (sprawdź dokumentację Gmail API, sprawdź ClickUp docs, uruchom test).
   - **Wartościowa** — różne priorytety (ergonomia vs korektność, velocity vs ryzyko). Wystaw kompromis na stół.
   - **Predykcyjna** — różne modele przyszłości (jak często zachodzi case X). Nazwij założenia, oszacuj pesymistycznie i optymistycznie.
3. **Każda strona — najsilniejsza wersja swojego argumentu w 2-3 zdaniach.** Bez kontry w pierwszej rundzie.
4. **Co by zmieniło zdanie?** Każda strona odpowiada. Jeśli „nic" — to wartościowa, eskaluj do decyzji.
5. **Decyzja przez konsensus z mandatem dla devil's advocate:** finalna opcja musi być akceptowalna dla wszystkich („mogę z tym żyć"), a Hanna ma obowiązek wskazać, jakie ryzyko zostaje na stole, nawet jeśli akceptujemy decyzję.

### Convergence Format

Każda istotna decyzja zapisana jako:

| Pole | Treść |
|------|-------|
| **Decyzja** | Co zostało postanowione |
| **Uzasadnienie** | Dlaczego — sformułowane tak, żeby najsilniejszy oponent uznał za fair |
| **Sprzeciw** | Kto się nie zgadzał i steelman jego zastrzeżenia |
| **Warunki rewizji** | Jakie nowe dowody / okoliczności reotwierają decyzję |
| **Pewność** | Wysoka / Średnia / Niska — z uzasadnieniem |

### Evidence Standards

Każde twierdzenie etykietowane:

- **[FAKT]**: Zweryfikowane, źródłowane (ClickUp docs, Gmail API docs, Make blog, RFC, log produkcyjny). Cytuj.
- **[WNIOSEK]**: Logiczna konsekwencja faktów. Pokaż łańcuch.
- **[REKOMENDACJA]**: Decyzja wartościowa. Nazwij wartości i kompromisy.

Każdy może zadać pytanie „fakt, wniosek czy rekomendacja?" w dowolnym momencie.

### Uncertainty Register

| ID | Niewiadoma | Wpływ jeśli źle | Rozwiązywalna? | Akcja |
|----|-----------|-----------------|----------------|-------|
| U1 | (do uzupełnienia w trakcie pracy) | | | |

Hanna jest właścicielką tego rejestru i pilnuje, żeby był aktualizowany w trakcie spotkań.

### Clarification Gate

Zanim zespół zacznie projektować, sprawdza:

- [ ] Kluczowe terminy zdefiniowane (co dokładnie znaczy „task", „lead", „match", „duplikat" w naszym kontekście)
- [ ] Kryteria sukcesu wprost (z dokumentu problemu — okej, mamy)
- [ ] Twarde ograniczenia znane (stack zamknięty — okej)
- [ ] Krajobraz interesariuszy (operatorzy + Bartek jako właściciel automatyzacji + klient)
- [ ] To jest właściwy problem (nie XY)

Gdy któryś punkt niezamknięty — pierwszą produkcją zespołu są pytania, nie odpowiedzi.

---

## Usage Notes

Ta karta jest gotowa do reuse w kolejnych sesjach. W tej sesji zespół przejdzie od razu do pracy nad M2 (przez skill `expert-meeting-facilitator`). Karta nie wygasa. Zrewiduj ją, jeśli:

- Pojawi się decyzja o dodaniu nowych narzędzi (np. własny mikroserwis Vibe Coded → potrzebny dev backend/frontend).
- Skala wzrośnie znacząco (>10x, czyli 1000+ zapytań/tydzień).
- Wejdzie kanał poza zakresem M2 (chat, social, SMS) — wtedy potrzebny specjalista per-kanał.
