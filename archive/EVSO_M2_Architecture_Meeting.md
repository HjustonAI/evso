# Meeting Output: EVSO M2 — Architektura Ciągłości Konwersacji Emailowej

**Data**: 2026-05-01
**Format**: Structured Workshop (deep-dive)
**Team Charter**: EVSO_M2_Team_Charter.md (v1.0)
**Meeting ID**: M-002 (M-001 było formularz intake redesign)

---

## Meeting Goal

**Jak sformułowany przez użytkownika**: „Opracuj koncepcję rozwiązania tego problemu — kompletną, gotową do wdrożenia. Nie jako listę kroków, ale jako spójną architekturę decyzji."

**Jak rozumie zespół**: Zaprojektować architekturę systemu, w której każda emailowa wiadomość od klienta trafia do dokładnie jednego, właściwego taska w ClickUp, niezależnie od kanału wejścia (in-thread reply, out-of-thread reply, brand-new email z znanego/nieznanego adresu). Operator obsługuje kontakt nie wychodząc z ClickUp. Architektura ma być na tyle konkretna, żeby developer mid-level wiedział *co* budować i *dlaczego*; na tyle ogólna, żeby nie była tutorialem implementacji. Zakres: wyłącznie email; telefon/chat/social poza zakresem.

**Target outcome**: Spójna architektura decyzji — zestaw nazwanych decyzji architektonicznych, każda z uzasadnieniem, jawnym sprzeciwem (jeśli był) i poziomem pewności. Nie checklist implementacji.

---

## Context Summary

### Co wiadomo

- **[FAKT]** EVSO operuje przez 3 strony (partytram.fun, partyboat.fun, busparty.fun) i co najmniej kilka adresów email. Przy formularzach M1 obsługuje 5 różnych form z 4 stron. Źródło: `EVSO_Milestone1_Definition.md`, memory `project_evso.md`.
- **[FAKT]** Milestone 1 jest wdrożony i działa od 2026-04-30. Single-webhook blueprint: webhook → field normalization → AI Haiku (klasyfikacja + ekstrakcja JSON) → mapowanie do ClickUp → `createTaskInList` w liście NOWE FORMULARZE (`list_id 901508747847`). Po stworzeniu taska — natywne automacje ClickUp routują do handlowców per miasto. Źródło: memory + `EVSO_Milestone1_Definition.md`.
- **[FAKT]** ClickUp Business posiada: Email-to-Task (alias adresu na liście, który tworzy taski z przychodzących maili), Email composer w tasku (możliwość wysłania wiadomości z poziomu taska, klient odpowiada na specjalny adres `Reply-To`, odpowiedź wraca do taska), Custom Fields (w tym pole EMAIL na taskach NOWE FORMULARZE), AI Fields, Autopilot Agents. Źródło: dokument problemu sekcja „Zasoby" + memory.
- **[FAKT]** ClickUp ma webhooki na `taskCommentPosted`, `taskUpdated`, oraz event związany z aktywnością emailową w tasku. Źródło: ClickUp API docs (do potwierdzenia w Phase 2 nazwy eventu).
- **[FAKT]** Make.com posiada moduły Gmail (Watch Emails, Send Email), ClickUp (Find Task, Update Task, Create Comment), Data Store (key-value persistence z TTL), Router, Filter, Error Handler. Źródło: dokumentacja Make.
- **[FAKT]** Klient identyfikowany wyłącznie po adresie email (świadomy kompromis z dokumentu problemu, sekcja Problem B).
- **[INFERENCE]** Dwa konta Gmail (po jednym na markę-grupę) — z memory: „Both Gmail accounts need to be connected to ClickUp Email as first step". Zakładamy, że są 2 OAuth-połączone konta Gmail; rzeczywista liczba do potwierdzenia.
- **[INFERENCE]** Skala ~100 zapytań/tydzień to mieszany strumień (formularze + emaile). Z tego email to prawdopodobnie 30-50%, czyli ~30-50 emaili/tydzień, plus kilka odpowiedzi per kierunek = ~100-200 wiadomości/tydzień łącznie. Architekturę projektujemy z 5x rezerwą do ~1000/tydz.

### Co niewiadome (przed pracą)

- **U1**: Czy ClickUp Email-to-Task tworzy task na *każdy* przychodzący email, czy ma konfigurowalne reguły (np. „twórz tylko jeśli sender nie pasuje do istniejącego")? Zakładamy: nie, zawsze tworzy → należy ten mechanizm wyłączyć dla naszego przypadku.
- **U2**: Czy ClickUp injecuje rozpoznawalny nagłówek (np. `X-ClickUp-Task-Id`) lub szczególny `Reply-To` przy wysyłce z taska? To kluczowe dla Make do filtrowania „własnych" wiadomości i unikania pętli. Zakładamy: tak, jest charakterystyczny `Reply-To` typu `reply+<TASKID>@inbound.clickup.com` lub podobny.
- **U3**: Jaka jest dokładna granulacja eventów webhook ClickUp dla wysłanego z taska emaila? Czy dostajemy `Message-ID`?
- **U4**: Czy w pipeline'ie M1 EMAIL custom field jest *zawsze* wypełniony przy nowym tasku z formularza? Zakładamy: tak (formularze mają wymagane pole email, a Haiku ekstrahuje fallback z treści).
- **U5**: Realna częstość scenariusza „klient ma kilka aktywnych tasków" — bez logu nie wiemy. Zakładamy: niska, ale nie zerowa (klient pyta o party tram na maj, potem o party boat na lipiec).

### Ograniczenia stosowane

- Stack zamknięty: ClickUp Business, Make.com, dwa konta Gmail, Claude Haiku.
- Nie zepsuć Milestone 1 (formularze → ClickUp).
- Identyfikacja klienta tylko po adresie email.
- Zgodność z GDPR (minimalizacja, retencja, audit trail).
- Zero nowych narzędzi.

---

## Expert Participation

| Ekspert | Rola | Poziom udziału | Główny obszar wkładu |
|---------|------|----------------|----------------------|
| Marta Kurek | ClickUp Systems Architect | Primary | WI1, WI3, WI4 — gdzie żyje stan w ClickUp; ergonomia operatora w tasku |
| Tomasz Sowa | Make Automation Architect | Primary | WI1, WI2, WI3 — flow Make, idempotencja, integracja Gmail-ClickUp |
| Aleksander Bryła | AI Systems & Reliability | Primary | WI2, WI4 — granica reguła/AI w matchingu, confidence gating |
| Karolina Wysocka | Sales & Service Process Designer | Primary | WI4 — operator UX, powiadomienia, scenariusze ambiguity |
| Igor Mazurek | Email Infrastructure & Deliverability | Primary | WI1, WI2, WI5 — semantyka emaila, threading, Gmail API |
| Hanna Czaplińska | Risk & Adversarial Architect | Primary całość, lead WI5 | Pre-mortem, Uncertainty Register, GDPR |

Wszyscy primary — to dla M2 nie jest spotkanie z obserwatorami.

---

## Work Items — dekompozycja celu

Moderator (ja) podzielił cel na 5 elementów pracy, sekwencjonowanych w kolejności zależności:

1. **WI1 — Topologia ingressu**: skąd email wchodzi do systemu i kto go pierwszy widzi. Wszystkie pozostałe decyzje zależą od tego.
2. **WI2 — Mechanizm matchingu**: jak email znajduje swojego taska. Zależy od WI1.
3. **WI3 — Stan i idempotencja**: gdzie żyje pamięć tego, co już przetworzyliśmy. Zależy od WI1+WI2.
4. **WI4 — Ergonomia operatora i obsługa ambiguity**: co operator widzi, gdzie odpowiada, co system robi gdy nie umie zdecydować. Zależy od WI1-WI3.
5. **WI5 — Pre-mortem**: co musi być prawdą, żeby to nie zadziałało. Wykonujemy *po* projekcie, ale *przed* zatwierdzeniem.

---

## WI1 — Topologia ingressu

**Pytanie centralne**: Gdzie email pierwszy raz styka się z naszym systemem? Czy ClickUp ma przyjmować email natywnie (Email-to-Task), czy ma to robić Make jako warstwa pośrednia?

### Opcje rozważone

- **A. ClickUp natywne Email-to-Task** dla wszystkiego: alias listy odbiera maile, ClickUp tworzy task, automacja ClickUp robi resztę.
- **B. Make Gmail Watcher** dla wszystkiego: ClickUp w ogóle nie odbiera emaili. Make obserwuje Gmail, decyduje, tworzy/aktualizuje task przez API.
- **C. Hybryd**: ClickUp Email composer obsługuje *wychodzące* maile (operator klika w tasku „Send Email", to działa natywnie i zachowuje threading). Make obserwuje Gmaila i obsługuje *wszystkie przychodzące* — w tym te będące odpowiedziami na wiadomości wysłane z ClickUp.

### Stanowiska

**Marta** [REKOMENDACJA]: „Pierwszy odruch — opcja A. ClickUp natywnie ma Reply-To z task ID, klient odpowiada, odpowiedź wpada do taska. To zero kodu, zero Make ops. Ale szybko widzę problem: Email-to-Task tworzy *nowy* task na każdą przychodzącą wiadomość, której Reply-To nie pasuje do żadnego oczekiwanego. To jest dokładnie nasz problem out-of-thread reply. ClickUp nie umie natywnie zrobić: 'sprawdź EMAIL custom field na istniejących taskach i zmatchuj'. Więc opcja A nie pokrywa Channel C z dokumentu problemu."

**Tomasz** [REKOMENDACJA]: „Opcja B byłaby najczystsza — Make ma pełną kontrolę. Ale wtedy *zarówno* in-thread, *jak i* out-of-thread przechodzą przez Make, i my musimy w Make replikować to, co ClickUp już umie (matchować odpowiedź po Reply-To). Niepotrzebna redundancja. Idę za C."

**Igor** [INFERENCE → REKOMENDACJA]: „Mechanizm ClickUp Reply-To opiera się prawdopodobnie na charakterystycznym adresie odpowiedzi, który zawiera task ID. Gdy klient odpowiada w wątku, jego klient pocztowy ustawia `To:` na ten adres. To jest niezawodne dla in-thread odpowiedzi *na konkretną wiadomość ClickUp*. Ale jest nieskuteczne, gdy klient zacznie nowy wątek lub odpowie z urządzenia, które gubi `In-Reply-To`. Zatem: niech ClickUp zajmie się tym, co umie (in-thread po Reply-To), a Make obejmie resztę. Hybryd."

**Aleksander** [REKOMENDACJA]: „Hybryd ma jedno zagrożenie: dwa systemy widzą tę samą wiadomość i obie próbują ją obsłużyć. Trzeba zapewnić, że Make rozpoznaje wiadomości, którymi ClickUp już się zajął, i je pomija. W przeciwnym razie podwoimy taski albo komentarze."

**Karolina** [RECOMMENDATION]: „Z perspektywy operatora hybryd jest okej — nie zauważy różnicy. Dla niego liczy się, że wiadomość jest w tasku. Ale jeśli kiedyś trzeba będzie debugować 'dlaczego ten email nie wpadł do taska', dwa systemy w ścieżce to dwa miejsca do sprawdzenia. To koszt utrzymania."

**Hanna** [PYTANIE]: „Co się stanie, gdy ClickUp ma awarię integracji Gmail (zdarza się, OAuth tokens wygasają)? Jeśli polegamy na ClickUp dla in-thread odpowiedzi, to *cicha* awaria — wiadomości odpadają i nikt tego nie widzi, bo Make ich nie ogląda. Wymaga monitoringu na warstwie Gmail."

### Decyzja D-001 — Topologia ingressu

| Pole | Treść |
|------|-------|
| **ID** | D-001 |
| **Decyzja** | Hybryd (opcja C). ClickUp natywnie obsługuje *tylko* odpowiedzi in-thread na wiadomości wysłane z taska (mechanizm Reply-To). Make obserwuje Gmail dla *wszystkich* przychodzących wiadomości i ma logikę odrzucania tych, którymi ClickUp już się zajął. Email-to-Task w ClickUp jako alias listy jest **wyłączony** — nie chcemy, żeby ClickUp tworzył taski na każdą przychodzącą wiadomość. |
| **Uzasadnienie** | Każda z czystych opcji ma poważną wadę pokrytą przez hybryd: A nie umie matchować po EMAIL custom field, B duplikuje funkcję, którą ClickUp i tak realizuje za darmo. Hybryd minimalizuje pracę Make do tych przypadków, gdzie naprawdę musi (matching po polu, ambiguity), a wykorzystuje natywne ClickUp dla 60-70% ruchu (zwykłe odpowiedzi w wątku). |
| **Sprzeciw** | Karolina i Hanna podnoszą koszt utrzymania (dwa systemy w ścieżce → trudniej debugować, cicha awaria ClickUp integracji Gmail). Akceptujemy ten koszt w zamian za prostotę implementacji. Mitygacja: monitoring Gmail (patrz WI5). |
| **Pewność** | Wysoka — ale **warunkowa** na potwierdzeniu U2 (ClickUp Reply-To/headery). Jeśli okaże się, że ClickUp nie ustawia rozpoznawalnego znacznika „to nasza wiadomość", Make musi mieć inną metodę odróżniania (np. sprawdzanie Message-ID we własnym Data Store outboundów). |
| **Warunki rewizji** | (a) ClickUp wprowadza natywny matching po Custom Field → opcja A staje się żywotna. (b) ClickUp przestaje natywnie zachowywać Reply-To → opcja B się aktywuje. (c) Częstość pętli/podwójnego processingu przekracza 1% wiadomości → architektura idzie do rewizji. |

### Decyzja D-002 — Wyłączenie ClickUp Email-to-Task

| Pole | Treść |
|------|-------|
| **ID** | D-002 |
| **Decyzja** | Email-to-Task aliasy nie są używane jako mechanizm tworzenia tasków z przychodzących emaili. Make jest jedynym mechanizmem tworzącym taski z emaili. |
| **Uzasadnienie** | Email-to-Task ma ślepe punkty: nie umie matchować przychodzącego emaila do istniejącego taska po sender == EMAIL field. Zostawienie go aktywnym tworzy konkurencyjny pipeline produkujący duplikaty (ten sam email tworzy task w Email-to-Task i równolegle Make matchuje go do istniejącego). |
| **Sprzeciw** | Marta (steelman): „ClickUp Email-to-Task ma tę zaletę, że załączniki idą natywnie do taska, a Make musi je explicit wgrywać." Akceptujemy ten koszt — Make wgrywa załączniki przez ClickUp API jako fallback. |
| **Pewność** | Wysoka. |
| **Warunki rewizji** | Jeśli ClickUp doda regułę „nie twórz nowego taska, jeśli sender pasuje do EMAIL field" — Email-to-Task wraca do gry. |

---

## WI2 — Mechanizm matchingu

**Pytanie centralne**: Make ma email od klienta. Skąd wie, do którego taska go skierować?

### Hierarchia matchingu (kolejność prób)

Zespół konwerguje na hierarchii „od najbardziej deterministycznego do najbardziej probabilistycznego". Każdy poziom wykonywany w kolejności, pierwszy który da wynik > threshold wygrywa.

**Poziom 1 — In-Reply-To header → Message-ID lookup**
Make sprawdza nagłówek `In-Reply-To` (i `References` jako fallback) przychodzącej wiadomości. Jeśli któryś `Message-ID` jest w naszym Data Store outboundów (znamy go z poprzedniej wysyłki z taska) → match deterministyczny do task ID powiązanego z tym Message-ID.

**Igor** [FAKT, RFC 5322]: „Każda wiadomość email ma unikalny `Message-ID`. Gdy klient odpowiada, jego klient pocztowy *powinien* (nie zawsze) ustawić `In-Reply-To: <oryginalny-Message-ID>`. To jest najpewniejszy mechanizm. Ale: Gmail App na Androidzie czasem ustawia `In-Reply-To` na inną wiadomość niż ta, na którą faktycznie odpowiadamy (gdy używamy 'Reply All' na zagnieżdżonym wątku). Outlook Mobile potrafi w ogóle zgubić nagłówki przy forwardzie. To jest czemu Poziom 1 to nie jest 100%."

**Aleksander** [INFERENCE]: „Skoro Poziom 1 jest deterministyczny *gdy działa*, to threshold dla niego to po prostu 'header obecny i istnieje w Data Store'. Confidence = 1.0 jeśli match, 0 jeśli nie. Nie ma szarej strefy."

**Poziom 2 — Sender email → EMAIL custom field na *aktywnym* tasku**
Jeśli Poziom 1 nie zwrócił nic, Make pyta ClickUp API: znajdź taski w Spaces EVSO/CRM gdzie EMAIL custom field == sender_email AND status NOT IN (zamknięty, archived, won, lost).

Trzy sub-przypadki:
- **2a — dokładnie 1 task pasuje**: deterministyczne. Confidence = 1.0. Match.
- **2b — 0 tasków pasuje**: nieznany sender → nowy task (Channel B z dokumentu problemu).
- **2c — N > 1 tasków pasuje**: ambiguity. Wchodzimy w Poziom 3.

**Marta** [FAKT, ClickUp]: „Custom Field search w ClickUp API jest dostępny przez `GET /list/{list_id}/task` z filtrem na custom field. Ograniczenie: domyślnie zwraca otwarte taski z listy NOWE FORMULARZE; jeśli klient ma już taska zarchiwizowanego (zamknięty proces) i wraca po pół roku — nie zmatchujemy. Pytanie projektowe: czy 'wraca po pół roku' to ten sam lead czy nowy?"

**Karolina** [REKOMENDACJA]: „Po pół roku to jest praktycznie nowy lead, kontekst się zmienił. Nie próbujemy matchować do zamkniętych tasków — chyba że klient explicit cytuje stary mail. Reaktywacja zamkniętego leada to manualna decyzja handlowca."

**Decyzja zespołu**: matchujemy tylko do tasków o statusie aktywnym (definicja statusu „aktywny" — patrz dec. D-005).

**Poziom 3 — AI disambiguation (tylko dla 2c, multi-task)**
Jeśli sender pasuje do > 1 aktywnego taska, Claude Haiku dostaje:
- Treść przychodzącej wiadomości (subject + body, ucięte do ~2000 znaków)
- Metadane każdego kandydata (typ imprezy, miasto, data, status, ostatnia aktywność)
- Prompt: „Wybierz najlepszy match LUB powiedz UNCERTAIN. JSON: `{task_id: string|null, confidence: 0.0-1.0, reason: string}`"

Threshold:
- `confidence ≥ 0.85` → auto-match.
- `0.5 ≤ confidence < 0.85` → suspected match: dodaj komentarz do top kandydata z linkami do wszystkich kandydatów + @mention właściciela, *nie* dołączaj wiadomości do żadnego taska automatycznie.
- `confidence < 0.5` lub `task_id == null` → fallback: stwórz nowy task w specjalnej liście „AMBIGUITY" z linkami do kandydatów i wiadomością; powiadom Bartka (jako system owner).

**Aleksander** [REKOMENDACJA]: „Najpierw reguła, potem model. Poziom 1 i 2 to reguły — tanie, deterministyczne, audytowalne. Poziom 3 to AI tylko gdy reguła nie wystarczyła. To jest właściwa kolejność. Threshold 0.85 to mój punkt startowy — kalibrujemy po zebraniu logów."

**Karolina** [SPRZECIW, częściowy]: „Threshold 0.85 jest konserwatywny. Operator wolałby auto-match przy 0.7. Każda 'suspected match' = praca dla człowieka. Przy 100/tydz to jeszcze do wytrzymania, ale przy 500 to już druga praca."

**Hanna** [SPRZECIW, kontra Karolina]: „Auto-merge przy 0.7 = 30% szans pomyłki. Pomyłka oznacza, że dwóch *różnych* klientów będzie widziało wzajemnie swoją historię w jednym tasku. To jest wyciek danych w rozumieniu GDPR. Nie schodzimy poniżej 0.85 bez dowodu, że Haiku jest lepszy niż 85% na tym domain."

**Aleksander** [synteza]: „Hanna ma rację co do principle, Karolina co do skali. Propozycja: startujemy z 0.85, mierzymy false positive rate w pierwszych 4 tygodniach (Bartek manualnie weryfikuje sample 50 auto-matchów), kalibrujemy. Confidence threshold to parametr, nie betonowa decyzja."

### Decyzja D-003 — Hierarchia matchingu

| Pole | Treść |
|------|-------|
| **ID** | D-003 |
| **Decyzja** | Trzypoziomowa hierarchia: (1) `In-Reply-To`/`References` header → Data Store outboundów; (2) sender email → ClickUp EMAIL custom field na aktywnych taskach; (3) Claude Haiku disambiguation dla multi-match. Pierwszy poziom z wynikiem wygrywa. |
| **Uzasadnienie** | Reguła deterministyczna jest tańsza, szybsza i audytowalna. AI używamy *tylko* tam, gdzie reguła nie ma odpowiedzi (multi-match). 80%+ ruchu obsłuży Poziom 1+2 bez tokenów. |
| **Sprzeciw** | Karolina by przesunęła próg auto-match niżej dla mniej pracy operatora. Aleksander+Hanna trzymają 0.85 do pierwszej kalibracji. Decyzja: 0.85 startowo, kalibracja po 4 tygodniach. |
| **Pewność** | Wysoka co do struktury hierarchii, średnia co do dokładnych progów (są kalibrowalne). |
| **Warunki rewizji** | (a) False positive rate auto-matchu > 5% w sample manualnym → próg w górę albo Haiku → Sonnet. (b) Operator skarży się na zalew „suspected match" komentarzy → próg w dół + monitoring jakości. |

### Decyzja D-004 — Confidence thresholds

| Pole | Treść |
|------|-------|
| **ID** | D-004 |
| **Decyzja** | Próg auto-match = 0.85. Próg suspected-match = 0.5. Poniżej 0.5 → AMBIGUITY list. |
| **Uzasadnienie** | Konserwatywny start. Cena pomyłki (krzyżowy wyciek danych klientów) > cena dodatkowej pracy operatora. |
| **Sprzeciw** | Karolina (rozwiązany w D-003 — kalibracja po pomiarach). |
| **Pewność** | Średnia — to są progi heurystyczne, nie dowiedzione. Plan: kalibrujemy po 4 tygodniach. |
| **Warunki rewizji** | Po pierwszej kalibracji (po ~4 tyg.); albo wcześniej jeśli incident. |

### Decyzja D-005 — Definicja „aktywnego" taska dla matchingu

| Pole | Treść |
|------|-------|
| **ID** | D-005 |
| **Decyzja** | „Aktywny" = task w Space EVSO/CRM którego status NIE jest jednym z: `closed`, `cancelled`, `lost`, `won`, `archived`. Zarchiwizowane / zamknięte taski są wykluczone z matchingu. |
| **Uzasadnienie** | Klient wracający po długim czasie ma kontekst, który się zmienił. Lepiej stworzyć nowy task niż odgrzebać stary. Reaktywacja zamkniętego leada to świadoma decyzja handlowca. |
| **Sprzeciw** | Hanna: „A jeśli klient cytuje w treści Message-ID zamkniętego maila? Wtedy Poziom 1 zmatchuje, ale Poziom 1 nie sprawdza statusu." Akceptujemy: Poziom 1 ignoruje status (deterministyczny match z dowodem). Poziom 2/3 honoruje status. |
| **Pewność** | Wysoka. |
| **Warunki rewizji** | Jeśli operatorzy zaczynają reklamować „klient wraca i tworzymy mu nowy task zamiast odzyskać stary" — rozważyć rozszerzenie okna na np. „closed w ostatnich 30 dniach". |

---

## WI3 — Stan i idempotencja

**Pytanie centralne**: Gdzie żyje pamięć tego, co system już widział i zrobił? Jak nie przetworzyć tej samej wiadomości dwa razy?

### Mapa stanu

Trzy miejsca trzymania stanu:

**A. Make Data Store `inbound_processed`** (klucz: Message-ID, wartość: `{task_id, processed_at, action}`)
Cel: idempotencja inboundu. Każda przychodząca wiadomość najpierw sprawdzana — jeśli Message-ID już w Data Store, skip. Inaczej procesujemy i zapisujemy.

TTL: 90 dni. Po tym czasie usuwamy (GDPR — minimalizacja).

**B. Make Data Store `outbound_message_id_to_task`** (klucz: Message-ID, wartość: `{task_id, sent_at, sender_account}`)
Cel: matching Poziom 1 (In-Reply-To). Każdy wychodzący email z ClickUp lub z Make ma swój Message-ID zapisany razem z task ID, do którego należy.

TTL: 180 dni (klient może odpowiedzieć z opóźnieniem; po pół roku akceptujemy, że match deterministyczny się nie uda i schodzimy do Poziomu 2).

**C. ClickUp jako źródło prawdy o tasku**
Treść taska, custom fields, komentarze, status, assignee. Make tylko czyta i pisze przez API; nie trzyma kopii.

**Tomasz** [REKOMENDACJA]: „Idempotencja w Make zawsze przez Data Store keyed by Message-ID. Make ma też `Sleep` i retry, ale Data Store check + write w jednej transakcji to jedyna gwarancja, że nie zrobimy dwa razy. Ważne: write do Data Store *po* udanej akcji ClickUp, nie przed — inaczej przy błędzie ClickUp wpadamy w stan 'oznaczone jako przetworzone, ale task nie zaktualizowany'."

**Hanna** [PYTANIE]: „A jeśli ClickUp API odpowie 200, ale faktycznie nie zaktualizuje (rzadkie, ale się zdarza)? Albo write do Data Store padnie po sukcesie ClickUp? Wtedy znowu rozjazd."

**Tomasz** [INFERENCE]: „W praktyce: write Data Store przed pisem ClickUp jest bezpieczniejszy dla idempotencji (nie zduplikujemy), kosztem ryzyka 'zgubionej' wiadomości jeśli ClickUp padnie. Write *po* jest bezpieczniejszy dla danych (jeśli ClickUp padnie, wiadomość przyjdzie znowu i się zaprocesuje). Wybór: write Data Store *po* udanej akcji + retry z exponential backoff dla błędów ClickUp + dead-letter queue (manualne review) po 3 nieudanych próbach."

**Igor** [WNIOSEK]: „Gmail watch może wysłać ten sam event więcej niż raz (Pub/Sub at-least-once semantics). Idempotencja po Message-ID jest *koniecznością*, nie optymalizacją."

### Decyzja D-006 — Strategia idempotencji

| Pole | Treść |
|------|-------|
| **ID** | D-006 |
| **Decyzja** | Trzy Data Store w Make: `inbound_processed` (TTL 90d), `outbound_message_id_to_task` (TTL 180d), `dead_letter` (bez TTL, manualnie czyszczone). Wszystkie kluczowane przez Message-ID. Write do `inbound_processed` *po* udanej akcji ClickUp. Retry z exponential backoff (3 próby), potem dead-letter + alert. |
| **Uzasadnienie** | Gmail Pub/Sub jest at-least-once → bez idempotencji rozjedziemy stan. Write-after-success preferuje brak utraty danych nad brakiem duplikatów; duplikaty wykrywamy i tak po Message-ID. |
| **Sprzeciw** | Hanna podnosi race condition (równoległe wykonanie Make dla tego samego eventu). Mitygacja: Make Scenario z `Sequential Mode = ON` dla webhooka inbound; zapobiega równoległemu wykonaniu. |
| **Pewność** | Wysoka. |
| **Warunki rewizji** | Jeśli skala wzrośnie tak, że Sequential Mode staje się bottleneckiem (>5 zdarzeń/min) — rozważyć distributed locking po Message-ID. Na 100/tydz nie problem. |

### Decyzja D-007 — Co NIE jest w Data Store

| Pole | Treść |
|------|-------|
| **ID** | D-007 |
| **Decyzja** | Pełne treści wiadomości email *nie* są przechowywane w Data Store. Tylko metadane: Message-ID, task_id, timestamp, sender (zhashowany hash zamiast plaintextu jeśli sensowne). Pełne treści żyją tylko w: (a) Gmail (nadal w skrzynce), (b) ClickUp jako komentarz na tasku. |
| **Uzasadnienie** | GDPR — minimalizacja. Data Store to mechanizm idempotencji, nie archiwum. ClickUp ma audit trail i retention policy; Data Store nie. |
| **Sprzeciw** | Brak. |
| **Pewność** | Wysoka. |
| **Warunki rewizji** | Jeśli operatorzy będą potrzebować historycznego widoku „która wiadomość poszła do którego taska" — rozważyć osobne rejestry audytowe, ale i tak nie pełne treści. |

---

## WI4 — Ergonomia operatora i obsługa ambiguity

**Pytanie centralne**: Operator otwiera ClickUp rano. Co widzi? Jak system mu komunikuje, że klient odpisał? Co robi z niejednoznacznymi wiadomościami?

### Sub-pytanie 1 — Reprezentacja wiadomości w tasku

**Marta** [REKOMENDACJA]: „Każda wiadomość email — niezależnie czy in-thread (przez ClickUp) czy out-of-thread (przez Make) — finalnie ląduje *w tym samym miejscu*: jako wpis w sekcji emaili taska w ClickUp. Native ClickUp robi to dla in-thread; Make musi to dorobić dla out-of-thread. ClickUp API ma endpoint `Add Comment` i `Add Attachment`. Najlepiej, jeśli Make dla out-of-thread:
1. Posta wiadomość jako *komentarz* z markerem `[EMAIL OUT-OF-THREAD]` na początku.
2. Załącza HTML body jako plik `.eml` przez Add Attachment.
3. Linkuje do oryginału w Gmail (deep link)."

**Karolina** [SPRZECIW, częściowy]: „Komentarz to nie to samo co email w ClickUp UI. ClickUp ma osobną sekcję 'Email' z wątkiem. Operator nie chce się gubić — wszystko email ma być w 'Email', nie w 'Comments'. Pytanie do Marty: czy ClickUp API pozwala dorzucać do tej sekcji 'Email' programatycznie?"

**Marta** [INFERENCE]: „Sekcja Email w tasku jest wewnętrznie strukturą `task_email_threads` w ClickUp. Z tego co pamiętam, API ma endpoint `Create Email` na tasku, ale to *wysyła* email, nie wgrywa istniejący. Wymaga sprawdzenia (U-NEW). Jeśli API nie pozwala wgrać istniejącej wiadomości do sekcji Email, fallback to komentarz z linkiem do Gmail i `eml` jako załącznik."

### Decyzja D-008 — Reprezentacja out-of-thread emaili

| Pole | Treść |
|------|-------|
| **ID** | D-008 |
| **Decyzja** | Hierarchia preferencji: (1) jeśli ClickUp API pozwala wgrać email do natywnej sekcji Email taska — używamy tego; (2) jeśli nie — komentarz z markerem `📧 EMAIL` w nagłówku + załącznik `.eml` + deep link do Gmail. Wybór finalny zależy od weryfikacji U-NEW (capability ClickUp API). |
| **Uzasadnienie** | Spójność UX (wszystko w sekcji Email) > prostota implementacji (tylko komentarze), ale tylko jeśli API to umożliwia. |
| **Sprzeciw** | Brak — to nie jest sporna decyzja, tylko zależna od faktu. |
| **Pewność** | Średnia — czeka na weryfikację API. |
| **Warunki rewizji** | Po sprawdzeniu ClickUp API. |

### Sub-pytanie 2 — Powiadamianie operatora

**Marta** [FAKT]: „Dodanie komentarza w ClickUp z `@mention` assignee'a generuje natywne powiadomienie (in-app + email + mobile push, zależnie od preferencji usera). Bez @mention — tylko jeśli user jest watcherem taska."

**Karolina** [REKOMENDACJA]: „Każda wiadomość od klienta = `@mention` aktualnego assignee taska. Operator dostaje natywne powiadomienie ClickUp, nie musi sprawdzać skrzynki. To pokrywa kryterium sukcesu #6 z dokumentu problemu."

**Aleksander** [PYTANIE]: „A jeśli task nie ma assignee w danym momencie (np. lead jest w stanie 'nowy', przed routingiem do handlowca)? Komu @mention?"

**Karolina** [REKOMENDACJA]: „Domyślny owner intake (Bartek lub osoba dyżurna). Po routingu automacjami ClickUp do Kuby/Izy/Wiktora — standardowo @mention bieżącego assignee. Jeśli assignee się zmienia w trakcie konwersacji, @mention zawsze leci do *aktualnego* assignee w momencie nowego maila."

### Decyzja D-009 — Powiadamianie

| Pole | Treść |
|------|-------|
| **ID** | D-009 |
| **Decyzja** | Każda nowa wiadomość = komentarz/email z `@mention` aktualnego assignee taska. Jeśli brak assignee → @mention domyślnego ownera intake (parametr konfigurowalny, na start: Bartek). Reszta powiadamiania — natywne ClickUp (in-app, email, push). |
| **Uzasadnienie** | Wykorzystuje istniejący mechanizm ClickUp, zero nowych narzędzi, operator dostaje powiadomienie tym samym kanałem co inne taski. |
| **Sprzeciw** | Brak. |
| **Pewność** | Wysoka. |
| **Warunki rewizji** | Jeśli operatorzy zaczynają wyłączać powiadomienia ClickUp ze względu na overload — rozważyć digest mode (dzienny @mention zamiast natychmiastowego). |

### Sub-pytanie 3 — Obsługa AMBIGUITY (gdy multi-match przy confidence < 0.5)

**Karolina** [REKOMENDACJA]: „Tworzymy taska w specjalnej liście `AMBIGUITY` (lub flagujemy `LEAD-####` z tagiem `ambiguity`). Body taska zawiera: oryginalną wiadomość, listę kandydatów (linki do tasków), powód wątpliwości od Haiku. Operator otwiera, klika 'merge to LEAD-1234' → custom action w Make, który: (a) dodaje wiadomość jako komentarz/email do wybranego taska, (b) usuwa task ambiguity, (c) zapisuje w `outbound_message_id_to_task` dla przyszłej referencji."

**Tomasz** [PYTANIE]: „Custom action — jak operator to wywołuje z ClickUp? ClickUp Custom Field 'Resolve to Task ID' + automacja na change?"

**Marta** [REKOMENDACJA]: „Prościej: w tasku ambiguity dropdown custom field 'Wybierz task docelowy' z opcjami `LEAD-1234`, `LEAD-5678`, `Nowy task`. Change na tym polu triggeruje Make scenario przez ClickUp webhook → Make wykonuje akcję."

**Hanna** [PYTANIE]: „A jeśli operator zignoruje ambiguity task na 7 dni? Klient czeka na odpowiedź, a my nic. Eskalacja?"

**Karolina** [REKOMENDACJA]: „Automacja ClickUp: każdy ambiguity task starszy niż 24h → @mention Bartka. Starszy niż 72h → @mention właściciela firmy."

### Decyzja D-010 — Obsługa AMBIGUITY

| Pole | Treść |
|------|-------|
| **ID** | D-010 |
| **Decyzja** | Lista `AMBIGUITY` w Space EVSO/CRM. Wiadomości poniżej confidence 0.5 (lub Haiku zwraca `task_id=null`) → task w tej liście z dropdown custom field do operator-driven resolution. Zmiana dropdownu triggeruje Make scenario, który wykonuje finalny merge. SLA: 24h → @mention Bartka, 72h → eskalacja. |
| **Uzasadnienie** | Człowiek-in-the-loop dla niepewnych przypadków, automatyczna ścieżka działania po wyborze, audytowalne. Eskalacja zapobiega zapomnieniu. |
| **Sprzeciw** | Brak istotnego. Karolina sugerowała `merge button` — funkcjonalnie tożsame z dropdown change. |
| **Pewność** | Wysoka co do mechanizmu, średnia co do progów SLA (24h/72h heurystyczne). |
| **Warunki rewizji** | Częstość ambiguity tasków > 5% wszystkich emaili → albo próg confidence źle skalibrowany, albo problem identyfikacji klienta wymaga przemyślenia. |

### Sub-pytanie 4 — Jak operator odpowiada

**Marta** [FAKT]: „ClickUp Email composer w tasku już istnieje i działa. Operator klika 'Send Email' w tasku, pisze, wysyła — to leci przez podpięty Gmail OAuth, klient widzi w swojej skrzynce, odpowiedź wraca natywnie do tego taska (mechanizm Reply-To). To jest istniejąca funkcjonalność M0/M1, nie wymaga nowego buildu w M2."

**Igor** [PYTANIE]: „A wychodzące Message-ID — jak Make je dostaje, żeby zapisać w `outbound_message_id_to_task`? Jeśli ClickUp wysyła sam, Make tego eventu nie widzi."

**Marta** [INFERENCE]: „ClickUp ma webhook event prawdopodobnie typu `taskEmailSent` (do potwierdzenia, U-NEW2). Jeśli jest — Make subskrybuje, dostaje task_id i Message-ID, zapisuje w Data Store. Jeśli nie — fallback: Make watchuje Sent folder Gmaila i dla każdego sent emaila parsuje subject/body za znacznikiem `[Task: LEAD-####]` (który ClickUp może injectować) lub szuka task ID po `In-Reply-To` jeśli to odpowiedź."

**Tomasz** [REKOMENDACJA]: „Najpierw weryfikacja webhooka (1 godz. pracy). Jeśli istnieje — używamy. Jeśli nie — Sent folder watcher z parsowaniem Reply-To (`reply+TASKID@inbound.clickup.com`) jako fallback. Oba są realne, oba w stack."

### Decyzja D-011 — Outbound i capture Message-ID

| Pole | Treść |
|------|-------|
| **ID** | D-011 |
| **Decyzja** | Operator wysyła z ClickUp Email composer (istniejące UX). Make capturuje wychodzące Message-ID przez: (a) ClickUp webhook `taskEmailSent` jeśli istnieje, (b) fallback: Gmail Sent folder watcher z parsowaniem charakterystycznego `Reply-To` ClickUp. Capture zapisuje do Data Store `outbound_message_id_to_task`. |
| **Uzasadnienie** | Operator nie zmienia workflowu, infrastruktura email-tracking jest pod spodem. |
| **Sprzeciw** | Brak. |
| **Pewność** | Średnia — zależna od weryfikacji U-NEW2 (czy webhook istnieje). |
| **Warunki rewizji** | Po weryfikacji webhooka. |

---

## WI5 — Pre-mortem: co musi być prawdą, żeby to nie zadziałało

Hanna prowadzi. Każdy ekspert dostaje swoją kategorię ryzyka i zgłasza scenariusze porażki. Zespół ocenia severity × likelihood × cost-of-fix i decyduje, co adresujemy.

### Scenariusze porażki — zgłoszone

**S1 — Auto-responder loop** (Igor)
Klient ma out-of-office. Operator z ClickUp wysyła wiadomość → klient automat odpisuje → Make matchuje (Poziom 2) → komentarz w tasku → operator albo automat ClickUp odpowiada → klient automat odpisuje → … Pętla.
- Severity: Wysoka (zalewa task, zalewa Gmaila quota, marnuje tokeny).
- Likelihood: Średnia (część klientów ma OOO, zwłaszcza w lipcu/sierpniu).
- Mitygacja: Make filtruje po nagłówkach `Auto-Submitted: auto-replied` (RFC 3834), `X-Auto-Response-Suppress`, `Precedence: bulk/auto_reply`. Wiadomości oznaczone jako auto → komentarz „Auto-reply received" z linkiem, *bez* notyfikacji, *bez* dalszych akcji automatycznych.

**S2 — Shared inbox sender** (Igor + Karolina)
Klient pisze z `kontakt@firma.pl` w imieniu firmy. Inny pracownik tej samej firmy też pisze z `kontakt@firma.pl` o czymś innym. Make widzi ten sam sender → matchuje do istniejącego taska → wiadomość trafia do złego klienta.
- Severity: Średnia-Wysoka (krzyżowe wycieki danych w ramach jednej firmy).
- Likelihood: Niska-Średnia (B2B przypadek; mniej dla EVSO event/wesele context, ale nie zerowy).
- Mitygacja: Subject-based heurystyka jako sygnał ostrzegawczy. Jeśli subject całkowicie różny od istniejącego wątku I sender to shared-pattern (`kontakt@`, `info@`, `office@`, `biuro@`, `hr@`, `events@`) → escalate do Poziomu 3 nawet przy 1-task match (nie traktuj jako 2a). Haiku ocenia spójność tematyczną.

**S3 — Spoofed sender / phishing** (Hanna)
Ktoś forguje sender = email znanego klienta i wysyła phishing.
- Severity: Wysoka (treść phishingu trafia do taska, operator może kliknąć).
- Likelihood: Niska (EVSO nie jest atrakcyjnym targetem dla targeted spoofing).
- Mitygacja: SPF/DKIM/DMARC check na każdym mailu. Wiadomości failing DMARC → komentarz z markerem `⚠️ NIEZWERYFIKOWANY NADAWCA`, brak auto-match (idą do AMBIGUITY). To jest natywna funkcja Gmaila — Make czyta header `Authentication-Results`.

**S4 — Operator forwarduje email do firmowej skrzynki** (Karolina)
Operator dostaje na prywatny email zapytanie, forwarduje na biuro@evso.fun. Make widzi: sender = operator (znany w team Gmail), forwarded message. Co robi?
- Severity: Średnia (creates wrong task / wrong attribution).
- Likelihood: Średnia (operatorzy to robią — to częsty pattern).
- Mitygacja: Make rozpoznaje wiadomości od członków teamu (whitelist domain `@evso.fun` + lista emaili operatorów) → traktuje jako forward, parsuje oryginalnego sendera z `From:` w body forwarda (heurystyka), tworzy task z prawdziwym senderem. To jest dodatkowy edge case, mid-priority.

**S5 — Gmail watch token expiration** (Igor)
Gmail Push notifications wymagają odnowienia subskrypcji co 7 dni. Jeśli scenario padnie i nie odnowi → po 7 dniach Gmail przestaje wysyłać eventy. Make nic nie wie. Maile od klientów wpadają do Gmaila, nie do ClickUp.
- Severity: Krytyczna (cicha awaria — wszystko leci w piach).
- Likelihood: Średnia w długim okresie (przy braku monitoringu — niemal pewne, że kiedyś się stanie).
- Mitygacja: (a) Make scheduled scenario co 5 dni, który robi `watch.refresh`. (b) Heartbeat: drugi scenario co godzinę robi `messages.list` z czasem ostatnich 70 min i porównuje liczby z faktycznymi przetworzonymi w `inbound_processed`. Jeśli rozjazd > X% — alert do Bartka.

**S6 — ClickUp API rate limit** (Marta + Tomasz)
ClickUp Business: 100 requests/min. Make scenario robi: lookup → conditional update → comment → notify → update. To 4-5 calls per email. Przy burst 20 emaili w minucie = 100 calls = limit.
- Severity: Średnia (wiadomości stoją w queue, ale nie giną dzięki retry).
- Likelihood: Niska na 100/tydz, średnia na 1000/tydz.
- Mitygacja: Make Sequential Mode + sensible retry policy + monitoring. Realna troska dopiero przy 5x scale.

**S7 — Klient pisze z różnych adresów email do tej samej sprawy** (Aleksander + Karolina)
Klient Janusz złożył formularz z `janusz@gmail.com`, potem napisał z `j.kowalski@firma.pl` o tym samym evencie. Z punktu widzenia systemu to dwie różne osoby → dwa taski.
- Severity: Średnia (duplikat, operator widzi później).
- Likelihood: Średnia (B2B/B2C overlap).
- Mitygacja: ŚWIADOMIE NIE adresujemy w M2. Dokument problemu mówi wprost: „Nie podejmujemy prób łączenia tożsamości klienta ponad różnymi adresami email". Akceptujemy duplikat. Operator może zmergować ręcznie.

**S8 — GDPR: prawo do usunięcia (right to erasure)** (Hanna)
Klient prosi o usunięcie danych. Mamy: task w ClickUp z mailami, Data Store entries z Message-IDs i task_id, kopia w Gmail.
- Severity: Wysoka (compliance).
- Likelihood: Niska-Średnia (rzadkie, ale obowiązkowe).
- Mitygacja: Procedura usunięcia: (a) ClickUp task delete (purges comments + emails), (b) Make `Data Store Search and Delete` keyed by sender email (znajdź wszystkie Message-IDs → usuń), (c) Gmail manual delete (operator ma uprawnienia). Cała procedura jako runbook, nie automacja — wymaga decyzji człowieka.

**S9 — Two emails arriving at exactly the same moment for the same task** (Tomasz)
Race condition: dwie wiadomości od tego samego klienta w odstępie 1 sek. Sequential Mode rozwiązuje to dla *jednego* scenariusza, ale nie chroni przed dwoma niezależnymi scenariuszami (np. z dwóch kont Gmail).
- Severity: Niska (komentarze mogą zostać dodane w odwrotnej kolejności).
- Likelihood: Niska.
- Akceptujemy.

**S10 — Klient odpowiada na bardzo starą wiadomość, której Message-ID już wygasł z Data Store** (Igor)
TTL outbound 180 dni. Klient odpowiada po 200 dniach. Poziom 1 nie matchuje (Message-ID expired). Poziom 2 może zmatchować jeśli task aktywny, ale jeśli zamknięty (D-005) → nowy task.
- Severity: Niska.
- Likelihood: Niska.
- Akceptujemy. Wydłużenie TTL nie pomoże — klient po pół roku to praktycznie nowy lead.

### Issue Register (top-priority po triage)

| ID | Issue | Severity | Likelihood | Akcja w M2 |
|----|-------|----------|-----------|-----------|
| R-1 | Auto-responder loop (S1) | Wysoka | Średnia | **Adresujemy**: filter `Auto-Submitted` headers |
| R-2 | Shared inbox sender (S2) | Średnia-Wysoka | Niska-Średnia | **Adresujemy**: heurystyka subject + shared-pattern → AMBIGUITY |
| R-3 | Spoofed sender (S3) | Wysoka | Niska | **Adresujemy**: DMARC check + flag |
| R-4 | Operator forward (S4) | Średnia | Średnia | **Adresujemy częściowo**: whitelist team domain, parse oryginalnego sendera (best-effort) |
| R-5 | Gmail watch expiration (S5) | Krytyczna | Średnia | **Adresujemy**: scheduled refresh + heartbeat monitoring |
| R-6 | ClickUp API rate limit (S6) | Średnia | Niska | **Monitorujemy**: sequential mode na razie wystarczy, alert jeśli throttle |
| R-7 | Multi-email-per-client (S7) | Średnia | Średnia | **NIE adresujemy** (świadomy kompromis z dokumentu problemu) |
| R-8 | GDPR right to erasure (S8) | Wysoka | Niska | **Adresujemy**: runbook + Data Store search-and-delete capability |
| R-9 | Race condition multi-mailbox (S9) | Niska | Niska | **Akceptujemy** |
| R-10 | Out-of-window reply (S10) | Niska | Niska | **Akceptujemy** |

---

## Disagreements

### Disagreement: Próg auto-match (Karolina vs Aleksander+Hanna)

- **Strony**: Karolina vs Aleksander+Hanna
- **Typ**: Wartościowy (różne priorytety: ergonomia operatora vs integralność danych) z elementem predykcyjnym (jak często Haiku się myli).
- **Pozycja A (Karolina)**: 0.7 jako próg auto-match. Powyżej tego, 30% błędu jest akceptowalne wobec oszczędności pracy operatora przy skali rosnącej.
- **Pozycja B (Aleksander+Hanna)**: 0.85 minimum. False positive = krzyżowy wyciek danych klientów = naruszenie GDPR + uszkodzenie reputacji. Cena pomyłki nieliniowo > cena pracy.
- **Co by zmieniło zdania**: Pomiar — jaki jest *faktyczny* false positive rate Haiku na realnych danych EVSO. Bez tego obie strony argumentują z teorii.
- **Rozwiązanie**: 0.85 startowo + obowiązkowa kalibracja po 4 tygodniach z manualnym audytem 50 auto-matchów. Próg parametryzowany, do dostosowania po danych. Zapisane jako D-004.

### Disagreement: ClickUp natywne vs Make orchestrator (Marta vs Tomasz)

Konstruktywnie rozwiązany przez D-001 (hybryd). Marta dostaje natywne dla in-thread, Tomasz dostaje Make dla reszty. Oboje akceptują.

### Niekonkluzywny zgrzyt: koszt utrzymania hybrydu (Karolina+Hanna vs reszta)

- **Pozycja**: Karolina i Hanna ostrzegają, że hybryd = dwa miejsca debugowania. Reszta zespołu uważa, że to mniejszy koszt niż alternatywa.
- **Status**: Nie zmieniamy decyzji D-001, ale wpisujemy do Uncertainty Register obowiązek monitoringu (S5).

---

## Convergence — synteza architektury

System ma **trzy warstwy fizyczne**:

1. **Gmail (transport + archiwum surowych wiadomości)** — wszystkie maile przechodzą tu fizycznie, w obu kierunkach. Gmail jest jedyną warstwą, w której żyje *pełna* historia surowa.

2. **Make (orchestracja + matching + idempotencja)** — Gmail Watcher dla inboundu, Sent Folder Watcher (lub ClickUp webhook) dla outboundu. Trzy Data Store dla idempotencji i mapowania Message-ID → task_id. Sequential mode dla deterministyczności. Logika hierarchii matchingu (3 poziomy). Wywołuje ClickUp API.

3. **ClickUp (źródło prawdy biznesowej + UX operatora)** — tu mieszka task, jego stan, custom fields, komentarze, sekcja Email, assignee, automacje routujące. Operator pracuje wyłącznie tu. Outbound emaile lecą natywnie z ClickUp Email composer.

**Linie odpowiedzialności** są jednoznaczne: Gmail nie wie nic o taskach. Make nie trzyma stanu biznesowego, tylko techniczny. ClickUp nie watchuje Gmaila, tylko robi natywne reply-tracking. Każda warstwa ma swoją jedyną odpowiedzialność, granice są wyraźne.

**Centralna decyzja architektoniczna**, wokół której kręci się wszystko inne, to **hierarchia matchingu deterministycznego przed AI** (D-003). Reguła zanim model. AI tylko gdy reguła nie ma wyniku, i nawet wtedy z confidence gating + human fallback.

**Drugi filar** to **idempotencja po Message-ID** (D-006). To jest uziemienie systemu — gwarancja, że niezależnie od tego, ile razy event się odpali, akcja zostanie wykonana raz.

**Trzeci filar** to **bezwarunkowa eskalacja ambiguity do człowieka** (D-010). Jeśli system nie jest pewien, decyduje operator, nie model. To jest cena za brak fałszywych pozytywów.

### Trade-Off Map

- **Wybierając hybryd (D-001)** zyskujemy: prostszy match w Make (in-thread idzie out of band), zachowanie istniejącej UX outbound w ClickUp. Tracimy: dwa miejsca debugowania, dependency na ClickUp Gmail integration health.
- **Wybierając próg 0.85 (D-004)** zyskujemy: niski false-positive rate, zgodność z GDPR, niska szansa krzyżowego wycieku. Tracimy: więcej AMBIGUITY tasków = więcej pracy operatora przy skali.
- **Wybierając wykluczenie zamkniętych tasków z matchingu (D-005)** zyskujemy: brak grzebania w „starych grobach", spójność z kontekstem biznesowym. Tracimy: rzadkie ale realne przypadki gdzie klient po 6 miesiącach legalnie wraca i my tworzymy duplikat.
- **Wybierając write-after-success dla Data Store (D-006)** zyskujemy: brak utraty wiadomości przy padzie ClickUp. Tracimy: minimalne ryzyko podwójnego komentarza przy bardzo specyficznym race condition (mitygowane Sequential Mode).
- **Wybierając AI tylko dla disambiguation (D-003)** zyskujemy: niski koszt tokenów, audytowalność większości decyzji, antifragility na awarie API Anthropica. Tracimy: nic istotnego — alternatywa „AI dla wszystkiego" jest droższa i mniej audytowalna.

---

## Uncertainty Register

| ID | Niewiadoma | Wpływ jeśli źle | Rozwiązywalna? | Akcja | Owner |
|----|-----------|-----------------|----------------|-------|-------|
| U-1 | Czy ClickUp Email-to-Task tworzy task na każdy mail (zakładamy: tak) | Średni — nieprawidłowa konfiguracja → konflikt z Make | Tak (test) | Tomasz: 30 min test w sandboxie | Tomasz |
| U-2 | Czy ClickUp używa rozpoznawalnego `Reply-To` z task ID | Wysoki — jeśli nie, Make nie odróżni in-thread od pozostałych | Tak (sprawdzić nagłówki sent emaila) | Igor: 15 min — wysłać testowy email z taska, sprawdzić nagłówki | Igor |
| U-3 | Granulacja webhooka ClickUp `taskEmailSent` (czy zwraca Message-ID) | Wysoki — jeśli nie, fallback to Sent Folder watch | Tak (ClickUp API docs + test) | Marta: 1 godz | Marta |
| U-4 | Czy EMAIL custom field jest *zawsze* wypełniony przy tasku z M1 | Średni — bez tego matching Poziom 2 nie zadziała | Tak (audyt logów M1) | Tomasz + Aleksander: 30 min | Tomasz |
| U-5 | Realna częstość multi-task per email | Niski-Średni — kalibruje progi confidence | Tak (po 4 tyg. obserwacji) | Bartek: monitoring | Bartek |
| U-NEW1 | Czy ClickUp API pozwala wgrać istniejący email do natywnej sekcji Email taska | Średni — wpływa na D-008 | Tak (API docs + test) | Marta: 1 godz | Marta |
| U-NEW2 | Czy istnieje webhook event `taskEmailSent` z Message-ID | Wysoki — wpływa na D-011 | Tak (API docs + test) | Marta+Tomasz: 1 godz wspólnie | Tomasz |
| U-6 | False positive rate Claude Haiku na disambiguation EVSO data | Wysoki — kalibruje D-004 | Tak (manual audit 50 sample po 4 tyg) | Bartek + Aleksander | Aleksander |
| U-7 | Faktyczna lista shared-inbox patternów relevant dla EVSO clientów | Niski-Średni — wpływa na heurystykę S2 | Częściowo (z czasem) | Tomasz: monitoruje, dodaje patterny | Tomasz |
| U-8 | Wymagania GDPR audit/retention specyficzne dla event/sales w PL | Wysoki — compliance | Częściowo (wymaga prawnika) | Bartek: konsultacja prawna *przed* go-live | Bartek |

---

## Open Questions

1. **Czy ClickUp Email composer używa OAuth do Gmaila czy SMTP relayed przez ClickUp?** — Ważne, bo determinuje, gdzie dokładnie żyje Sent folder do watch. Najprawdopodobniej OAuth, sent ląduje w Gmail Sent. Do potwierdzenia. Bez odpowiedzi: zakładamy OAuth, monitorujemy.

2. **Co robimy z załącznikami > 25 MB?** — Gmail hard limit. Nie dostaną się do ClickUp przez API. Najprawdopodobniej operator dostaje komentarz „załącznik za duży, link do Gmail" i obsługuje manualnie. Do uzgodnienia z Karoliną w iteracji wdrożeniowej.

3. **Czy ClickUp natywny mechanizm Reply-To rozpoznaje *wszystkie* nasze konta Gmail, czy tylko jedno połączone konto na workspace?** — Jeśli tylko jedno, hybryd ma lukę dla drugiej marki/konta. Krytyczne do potwierdzenia (U-2 ma podpunkt). Bez odpowiedzi: zakładamy że oba konta mogą być podpięte z osobna, weryfikujemy w pre-build.

4. **Język wiadomości (PL/EN/DE)?** — Haiku radzi sobie z PL i EN. EVSO obsługuje głównie PL. Jeśli pojawi się klient niemiecki/angielski, Haiku nadal poradzi. Akceptujemy bez specjalnej obsługi.

5. **Audit log poza ClickUp** — Hanna pyta: czy Bartek chce historyczny log „która wiadomość trafiła do którego taska kiedy", niezależny od ClickUp? Make Data Store się do tego nie nadaje (TTL 90d). Sugestia: Make zapisuje też do Google Sheets jako audit. Niska priorytet, do decyzji Bartka.

---

## Next Work Package

| Priorytet | Akcja | Uzasadnienie | Zależności |
|-----------|-------|--------------|------------|
| P0 | Weryfikacja U-2, U-3, U-NEW1, U-NEW2 (4 testy ClickUp/Gmail) | Bez tych odpowiedzi nie da się ostatecznie zatwierdzić D-001, D-008, D-011 | Brak — może zacząć Tomasz/Marta od razu |
| P0 | Wyłączyć ClickUp Email-to-Task aliasy dla EVSO/CRM | Konflikt z Make ingressem (D-002) | Brak |
| P1 | Zbudować podstawowy scenario Make: Gmail Watch → idempotency check → matching Poziom 1 i 2 → Add Comment z @mention | MVP architektury; pokrywa 80% przypadków | U-2, U-3, U-4 |
| P1 | Skonfigurować 3 Data Store w Make z TTL i schematem | Idempotencja + outbound tracking (D-006, D-007) | Brak |
| P1 | Heartbeat scenario dla Gmail watch refresh + monitoring | Krytyczne — bez tego cicha awaria (R-5) | Scenario z P1 |
| P2 | Dodać Poziom 3 (Haiku disambiguation) z confidence gating | Edge case dla multi-match (D-003, D-004) | Scenario z P1 działa stabilnie |
| P2 | Zbudować mechanizm AMBIGUITY list + dropdown resolver | Human-in-the-loop dla niskiej confidence (D-010) | Poziom 3 |
| P2 | DMARC check + auto-responder filter (R-1, R-3) | Risk mitigation | Scenario z P1 |
| P3 | Operator forward heuristic (R-4) | Mid-priority edge case | Scenario z P1 |
| P3 | Konsultacja prawna GDPR (U-8) i runbook erasure (S8) | Compliance | Przed go-live |
| P3 | Po 4 tygodniach: kalibracja progów confidence (U-6) | Tuning oparty na danych | 4 tygodnie produkcji |

---

## Artifacts to Update

| Artefakt | Akcja | Konkretna zmiana | Wywołane przez |
|----------|-------|------------------|----------------|
| `EVSO_Milestone2_Specification.md` (istniejący) | Update / Review | Porównać z niniejszą architekturą — gdzie zgodne, scalić; gdzie rozbieżne, podjąć decyzję która wersja wygrywa | Cała sesja M-002 |
| `EVSO_Milestone2_Problem_Definition.md` | Bez zmian | Definicja problemu pozostaje aktualna — wszystkie kryteria sukcesu adresowane przez D-001..D-011 | — |
| `project_evso.md` (memory) | Update | Wpisać że M2 przeszedł architecture review M-002, zapisać główne decyzje (hybrid ingress, 3-tier matching, 0.85 threshold) | D-001, D-003, D-004 |
| Risk Register (do utworzenia jeśli nie istnieje) | Create | R-1..R-10 jako trwały rejestr | WI5 |
| Decision Log (do utworzenia jeśli nie istnieje) | Create | D-001..D-011 jako trwały rejestr | Phase 5 |
| ClickUp workspace (operacyjnie) | Update | (a) Wyłączyć Email-to-Task na liście NOWE FORMULARZE; (b) utworzyć listę AMBIGUITY w Space EVSO/CRM; (c) dodać dropdown custom field „Wybierz task docelowy" na liście AMBIGUITY; (d) automacja eskalacji 24h/72h | D-002, D-010 |
| `EVSO_M2_Team_Charter.md` | Bez zmian | Karta okazała się adekwatna; nie odkryto luk eksperckich w trakcie meetingu | — |

---

## Meeting Metadata

- **Format użyty**: Structured Workshop (deep-dive)
- **Eksperci aktywowani**: 6 z 6
- **Decyzje podjęte**: 11 (D-001 do D-011)
- **Niewiadome zarejestrowane**: 11 (U-1..U-8, U-NEW1, U-NEW2)
- **Otwarte pytania**: 5
- **Szacowana pewność rekomendacji architektonicznej**: **Wysoka** dla struktury (3 warstwy, hierarchia matchingu, idempotencja po Message-ID); **Średnia** dla parametrów (progi confidence, TTL Data Store, dokładny mechanizm capture outbound Message-ID — zależne od weryfikacji U-2, U-3, U-NEW1, U-NEW2).

---

## Pre-final check: co musi być prawdą, żeby to nie zadziałało?

To pytanie zadane sobie samemu przed zatwierdzeniem, jak prosił użytkownik. Odpowiedź szczera:

System może nie zadziałać, jeśli choć jedno z poniższych okaże się prawdą — i wszystkie z nich są w naszym Uncertainty Register, ale warto je tutaj wyłuskać jeszcze raz:

1. **ClickUp nie ma rozpoznawalnego znacznika „to nasza wiadomość"** (U-2 false). Wtedy Make nie odróżni wiadomości ClickUp-bound od pozostałych → albo podwójne processing, albo gubienie. **Mitygacja**: weryfikacja przed buildem; jeśli false → wszystkie inboundy przez Make, wyłączamy ClickUp natywny tracking, opcja B nie hybryd.

2. **EMAIL custom field nie zawsze jest wypełniony na taskach M1** (U-4 false). Wtedy Poziom 2 matchingu często zwróci 0 → fałszywe „nowe taski" dla istniejących klientów → duplikaty. **Mitygacja**: audyt M1 przed M2; ewentualne dorzucenie validacji w M1 zapewniającej zawsze niepuste EMAIL.

3. **Gmail watch padnie cicho** (R-5). Bez heartbeat monitoringu — niewykrywalne aż klient zgłosi „nie odpowiadacie". **Mitygacja**: heartbeat *jest* w architekturze (D-006 implicitly via P1 task „Heartbeat scenario"). Krytyczne, by go zbudować, nie odłożyć.

4. **Operatorzy nie zaadoptują**, bo jednak wolą Gmaila. To jest ryzyko produktowe, nie techniczne. **Mitygacja**: Karolina sprawdza po 2 tygodniach z każdym operatorem osobno, czy używają wyłącznie ClickUp; jeśli nie — diagnostyka friction.

5. **Haiku jest gorszy niż 85% na EVSO disambiguation**. Wtedy auto-match zachowuje się jak loteria. **Mitygacja**: kalibracja po 4 tyg. + threshold parametryzowalny.

6. **ClickUp API nie pozwala programatycznie dodać emaila do natywnej sekcji Email** (U-NEW1 false). Wtedy fallback do komentarzy, co psuje UX założenia D-008. **Mitygacja**: fallback już zaprojektowany; UX akceptowalny, choć nie idealny.

Żaden z tych punktów *nie* jest unaddressowany. Każdy ma w architekturze przypisaną mitygację (proceduralną lub techniczną) lub jawnie zaakceptowaną akceptację. Architektura jest gotowa do walidacji weryfikacjami P0 (U-2, U-3, U-NEW1, U-NEW2) i potem do buildu.

---

*Dokument architektury — finalna wersja sesji M-002. Każda decyzja D-NNN jest do osobnej aktualizacji w przyszłych sesjach na podstawie kalibracji i incident reports.*
