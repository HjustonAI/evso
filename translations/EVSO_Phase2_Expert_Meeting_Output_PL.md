# Output spotkania: EVSO Phase 2 — Warsztaty projektowe dla automatyzacji post-intake

**Data**: 2026-03-31
**Format**: Structured Workshop
**Team Charter**: EVSO DevTeam (Kurek, Sowa, Bryła, Wysocka)
**ID spotkania**: M-002

---

## Cel spotkania

**Jak sformułowany**: „Zacznij Phase 2 od spotkania zespołu. Wyklucz wielojęzyczność, dashboard analityczny i automatyczne przypisywanie leadów. Skupcie się wyłącznie na reszcie."

**Jak rozumiany**: Zaprojektowanie architektury wdrożenia czterech funkcji Phase 2 odłożonych z Phase 1: (1) generowanie roboczych odpowiedzi, (2) deduplicacja leadów, (3) przypomnienia o follow-upach, (4) automatyzacja obiegu ZLECENIA. Spotkanie musi wyprodukować konkretne, zbieżne decyzje projektowe dla każdej z nich — nie idee. Trzy pozycje z oryginalnej listy odłożeń Phase 1 są jawnie wykluczone: wielojęzyczność, dashboard analityczny, automatyczne przypisywanie leadów.

**Oczekiwany rezultat**: Plan działania — konkretne decyzje projektowe dla każdej funkcji, wystarczające do napisania przewodnika wdrożenia o tej samej jakości co przewodnik Phase 1.

---

## Podsumowanie kontekstu

### Co jest znane

- [FAKT] Phase 1 jest zaprojektowany i dostarcza: wzbogacony intake formularzy przez Make.com, nowy intake emailowy przez Make.com, klasyfikację/ekstrakcję AI z użyciem Claude Haiku, 5 nowych Custom Fields (ŹRÓDŁO_KANAŁ, KLASYFIKACJA_AI, AI_PODSUMOWANIE, KOMPLETNOŚĆ, AI_CONFIDENCE), 2 nowe widoki ClickUp, 1 automatyzację ClickUp.
- [FAKT] Scenariusze Phase 1 wywołują już Anthropic API dla klasyfikacji/podsumowania. Połączenie API, przechowywanie klucza i wzorce obsługi błędów są ustalone.
- [FAKT] NOWE ZAPYTANIA to centralna lista intake z 11 oryginalnymi + 5 nowymi Custom Fields. Taski używają ID LEAD-#### i konwencji nazewnictwa `[DATA|BRAK DATY] | [MIASTO|WYBIERZ MIASTO] | [USŁUGA] | [IMIĘ]`.
- [FAKT] Istnieją pipelines sprzedawców: SPRZEDAŻ KUBA, SPRZEDAŻ WIKTOR, SPRZEDAŻ DAWID, SPRZEDAŻ KRIS. Obserwowane statusy na tablicy SPRZEDAŻ DAWID: NOWE ZAPYTANIE → WYSŁANO OFERTĘ → PRZEDZWONKA → ZASTANAWIA SIĘ → SZANSA SPRZEDAŻY → [skrócone].
- [FAKT] ZLECENIA to osobny folder/lista ze statusami: ZLECENIA BEZ ZALICZEK, REZERWACJA JEDNOSTKI, ZLECENIA, REZYGNACJA, ANULOWANY, DO ROZLICZENIA, ZREALIZOWANE, 2025, PRZEGRANY. Nazewnictwo tasków rozszerzone o PAKIET.
- [FAKT] ClickUp Email in Task jest już używany — composer emailowy widoczny w taskach leadów.
- [FAKT] Opisy tasków przechowują oryginalne zapytanie klienta.
- [FAKT] Panel aktywności w taskach pokazuje chronologiczną historię zmian, komentarzy i emaili.

### Co jest niepewne

| Nieznane | Dlaczego to ważne | Założenie przy braku danych |
|----------|-------------------|-----------------------------|
| U-001: Obecny proces handoff CRM→ZLECENIA | Decyduje, czy automatyzacja powinna przenosić taski czy tworzyć nowe | Spotkanie dostarcza dwa alternatywne projekty; inżynier musi zweryfikować |
| U-002: Spójny status „wygranego" we wszystkich pipeline'ach sprzedawców | Bez niego nie ma wiarygodnego trigger'a dla automatyzacji ZLECENIA | Zakładamy, że musimy dodać status SPRZEDANE do każdej listy sprzedawców |
| U-003: Ton i głos marki EVSO w odpowiedziach emailowych | Jakość roboczych odpowiedzi zależy od dopasowania do stylu komunikacji firmy | Inżynier musi zebrać 5-10 prawdziwych przykładów odpowiedzi przed pisaniem system promptu |
| U-004: Możliwości planu ClickUp dla automatyzacji czasowych | Niektóre automatyzacje (np. „task był w statusie przez X dni") mogą wymagać określonych planów ClickUp | Zakładamy plan Business; zweryfikuj przed wdrożeniem |
| U-005: Czy pipeline'y sprzedawców współdzielą definicje Custom Fields z NOWE ZAPYTANIA | Wpływa na to, czy dane pól przenoszą się poprawnie przy przenoszeniu tasków między listami | Zakładamy, że Custom Fields są na poziomie folderu CRM, więc współdzielone |

### Zastosowane ograniczenia

1. **Jeden inżynier, ograniczony czas** — całość Phase 2 nie może przekroczyć 15 dni inżynierskich.
2. **No-code/low-code first** — używaj natywnych automatyzacji ClickUp zamiast scenariuszy Make wszędzie tam, gdzie to możliwe.
3. **AI asystuje, człowiek decyduje** — robocze odpowiedzi są sugestiami, nigdy nie są wysyłane automatycznie.
4. **Ręczny fallback** — każda automatyzacja musi działać, gdy Make lub AI są niedostępne.
5. **Bezpieczne do przekazania niestechnicznemu użytkownikowi** — zespół sprzedaży musi móc korzystać ze wszystkich funkcji bez wsparcia inżynierskiego.
6. **Phase 1 musi pozostać stabilny** — zmiany w istniejących scenariuszach Phase 1 muszą być addytywne (gałęzie router'a), nie destruktywne.

---

## Udział ekspertów

| Ekspert | Rola | Poziom uczestnictwa | Główny obszar wkładu |
|---------|------|---------------------|----------------------|
| Marta Kurek | Architekt systemów ClickUp | Podstawowy | Struktura ClickUp dla wszystkich 4 funkcji; projekt przejścia ZLECENIA; natywne możliwości automatyzacji |
| Tomasz Sowa | Architekt automatyzacji Make | Podstawowy | Projekt scenariuszy Make dla dedup i roboczych odpowiedzi; architektura Data Store; strategia modyfikacji scenariuszy |
| Aleksander Bryła | Architekt systemów AI i niezawodności | Podstawowy | Projekt promptów roboczych odpowiedzi; gating przez confidence; egzekwowanie granic AI; logika dopasowania dedup |
| Karolina Wysocka | Projektant procesu sprzedaży i obsługi | Podstawowy | Wpływ na workflow zespołu sprzedaży; ton i użyteczność roboczych odpowiedzi; projektowanie procesu follow-up; wymagania handoff ZLECENIA |

Wszyscy czterej eksperci mają status Podstawowy — każda funkcja dotyka wszystkich czterech dziedzin.

---

## Podsumowanie dyskusji

Warsztaty zostały rozłożone na cztery elementy pracy, ułożone według zależności:

1. **Przypomnienia o follow-upach** (najprostsze, najmniejsze ryzyko — wyznacza wzorzec bazowy)
2. **Deduplicacja leadów** (musi być rozwiązana przed roboczymi odpowiedziami, aby unikać tworzenia odpowiedzi dla duplikatów)
3. **Generowanie roboczych odpowiedzi** (największy wpływ widoczny dla użytkownika, zależy od stabilnej dedup)
4. **Automatyzacja obiegu ZLECENIA** (największa niepewność, wymaga śledztwa)

---

### Element pracy 1: Przypomnienia o follow-upach

#### Marta Kurek — Architekt systemów ClickUp

[FAKT] ClickUp obsługuje natywnie automatyzacje czasowe: „Gdy nadejdzie termin (Due Date)" i „Gdy task był w statusie przez [czas]" to dostępne typy trigger'ów na planach Business i wyższych.

[REKOMENDACJA] Użyj natywnego mechanizmu Due Date ClickUp zamiast budowania scenariusza Make z pollingiem. Podejście: scenariusze Make Phase 1 powinny zostać zaktualizowane, by ustawiać Due Date = `teraz + 48 godzin` dla każdego nowo tworzonego taska, gdzie KLASYFIKACJA_AI NIE jest „Spam". Następnie automatyzacja ClickUp odpala się, gdy nadejdzie Due Date: jeśli task wciąż nie ma przypisanego Assignee, eskaluje go ustawiając Priorytet na Pilny i dodając komentarz z pingiem do zespołu.

Jest to czystsze od trigger'ów opartych na czasie spędzonym w statusie, ponieważ: (a) Due Date jest widoczny w widokach kalendarza i listy, dając zespołowi termin bez otwierania taska, (b) nie zależy od tego, czy task pozostaje w konkretnym statusie — jeśli ktoś zmieni status, ale nie przypisze taska, przypomnienie wciąż się odpali.

[FAKT] Komentarz ClickUp może tagować konkretnych użytkowników lub kanał. Kanał CRM już istnieje.

#### Tomasz Sowa — Architekt automatyzacji Make

[WNIOSKOWANIE] Zgadzam się — automatyzacje ClickUp to właściwe narzędzie. Budowanie tego w Make wymagałoby scenariusza z harmonogramem (np. uruchamianie co godzinę, wyszukiwanie tasków z przekroczonym Due Date, filtrowanie, wysyłanie powiadomienia). To bardziej kruche, zwiększa koszty operacji i wolniej reaguje.

[REKOMENDACJA] Jedyna zmiana po stronie Make: dodanie operacji ustawiania `Due Date` do obu scenariuszy Phase 1 (Form Intake Enhanced i Email Intake). To jedno dodatkowe pole w istniejącym module ClickUp Create Task — około 5 minut konfiguracji na scenariusz.

Przypadek brzegowy: jeśli task jest tworzony podczas fallback'u AI (KLASYFIKACJA_AI = „Do weryfikacji"), POWINIEN nadal otrzymać Due Date, ponieważ są to taski, które najprawdopodobniej wymagają uwagi.

#### Aleksander Bryła — Architekt systemów AI i niezawodności

Nie ma potrzeby angażowania AI. To czysto deterministyczne — oparte na regułach, bez modelu. Poprawny projekt.

#### Karolina Wysocka — Projektant procesu sprzedaży i obsługi

[REKOMENDACJA] Okno 48-godzinne jest właściwe dla NOWE ZAPYTANIA. Nie buduj skomplikowanych wielopoziomowych reguł SLA. Jedna reguła: brak przypisania + termin → oznacz.

Kluczowe wyjaśnienie: przypomnienie NIE powinno odpalać się dla tasków sklasyfikowanych jako Spam. Taski spamowe nie mają pilności i oznaczanie ich marnuje uwagę. Due Date powinno być ustawiane tylko dla leadów niebędących spamem.

Drugi ważny punkt: komentarz automatyzacji ClickUp powinien mówić coś konkretnego i wymagającego działania, nie tylko „zaległy". Sugerowany tekst: `⚠️ To zapytanie czeka na obsługę od 48h. Proszę o przypisanie lub oznaczenie jako nieistotne.`

**Uwaga moderatora:** Brak niezgodności w tym elemencie pracy. Zespół szybko zbiegł do natywnego podejścia ClickUp z Due Date. Zbadano możliwość sprzeciwu: „Co jeśli automatyzacje ClickUp są zawodne?" — Marta potwierdziła, że Automatyzacje są stabilne na planach Business i wyższych. Tomasz potwierdził, że zmiana po stronie Make jest banalna. Zbieżność jest autentyczna.

---

### Element pracy 2: Deduplicacja leadów

#### Tomasz Sowa — Architekt automatyzacji Make

[REKOMENDACJA] Użyj Data Store Make jako tabeli lookup. Oto kompletny mechanizm:

**Przy każdym tworzeniu nowego taska (oba scenariusze):**
1. Po utworzeniu taska w ClickUp, zapisz rekord w Data Store: `{ email: <email_klienta>, task_id: <id_taska_clickup>, created_at: <timestamp> }`
2. PRZED tworzeniem taska, odpytaj Data Store: „Czy jest rekord z tym emailem, gdzie created_at jest w ciągu ostatnich 7 dni?"
3. Jeśli NIE MA dopasowania → proceed normalnie (utwórz nowy task).
4. Jeśli JEST DOPASOWANIE → NIE twórz nowego taska. Zamiast tego: (a) dodaj komentarz do istniejącego taska z treścią nowego zapytania, (b) zaktualizuj KOMPLETNOŚĆ, jeśli nowe zgłoszenie dostarcza wcześniej brakujące pola, (c) dodaj tag do istniejącego taska.

**Struktura Data Store:**
- Nazwa: `EVSO_Lead_Dedup`
- Klucz: `email` (klucz główny)
- Pola: `email` (text), `task_id` (text), `created_at` (date), `phone` (text, opcjonalnie — do przyszłego dopasowania)

Podejście auto-wygasania: Make nie obsługuje natywnie TTL, więc dodaj scenariusz czyszczący uruchamiany co tydzień, który usuwa rekordy starsze niż 7 dni.

Przypadek brzegowy do obsłużenia: Jeśli dedup wykryje dopasowanie, a nowe zgłoszenie ma pola, których brakowało w oryginalnym tasku (np. oryginał nie miał daty, nowe zgłoszenie ją ma), automatyzacja powinna ZAKTUALIZOWAĆ Custom Fields istniejącego taska nowymi danymi. To zamienia duplikat w okazję do wzbogacenia danych.

#### Aleksander Bryła — Architekt systemów AI i niezawodności

[REKOMENDACJA] Dopasowanie emailowe jest deterministyczne — AI niepotrzebne. Poprawnie.

Normalizacja telefonu: Jeśli kiedyś dodamy dopasowanie telefoniczne, normalizuj wszystkie numery telefonów do formatu E.164 przed zapisem i porównaniem. Na razie email-only to właściwy zakres — prostszy, bardziej niezawodny, a email jest najbardziej spójnym identyfikatorem między kanałami.

[FAKT] Ważna kwestia kolejności logicznej: sprawdzenie dedup MUSI nastąpić PRZED wywołaniem klasyfikacji AI. Jeśli wykryjemy duplikat, całkowicie pomijamy wywołanie AI (oszczędność kosztów, unikamy generowania klasyfikacji dla danych, które nie staną się osobnym taskiem). Przepływ scenariusza staje się: Webhook → Normalizacja → **Sprawdzenie dedup** → (jeśli nowy) Wywołanie AI → Utwórz task → Zapisz w Data Store; (jeśli duplikat) → Zaktualizuj istniejący task → Pomiń AI.

To zmienia kolejność modułów w obu scenariuszach Phase 1. Jest to ważna modyfikacja strukturalna, nie tylko addytywna gałąź.

#### Marta Kurek — Architekt systemów ClickUp

[REKOMENDACJA] Gdy duplikat jest łączony z istniejącym taskiem, powinny nastąpić następujące zmiany ClickUp:

1. **Dodany komentarz** z pełną treścią nowego zapytania, wyraźnie oznaczony: `--- DODATKOWY KONTAKT ({{kanał_źródłowy}}) --- {{data}} ---`, a następnie treść nowej wiadomości.
2. **Aktualizacja Custom Field:** Nowe pole `LICZBA_KONTAKTÓW` (Number, domyślnie 1) — zwiększane przy każdym łączeniu.
3. **Tag:** Dodaj tag `WIELOKROTNY KONTAKT` do taska. Jest widoczny w widokach listy i sygnalizuje handlowcowi, że klient zgłosił się wielokrotnie.
4. **Przeliczenie KOMPLETNOŚCI:** Jeśli nowe zgłoszenie wypełnia wcześniej puste pola, zaktualizuj zarówno pola, jak i wynik KOMPLETNOŚCI.

Potrzebne nowe Custom Field: `LICZBA_KONTAKTÓW` (Number, liczba całkowita, wartość domyślna 1, na liście NOWE ZAPYTANIA). Cel: śledzenie, ile razy klient zgłosił się w sprawie tego zapytania.

#### Karolina Wysocka — Projektant procesu sprzedaży i obsługi

[REKOMENDACJA] Wiele kontaktów od tej samej osoby to pozytywny sygnał zakupowy. System dedup powinien uczynić to WIDOCZNYM, nie cichym.

Gdy wykryty jest duplikat, komentarz powinien być wystarczająco widoczny, żeby handlowiec go zauważył. Zwiększanie LICZBA_KONTAKTÓW i dodawanie tagu to dobre rozwiązania — ale najważniejszą akcją jest: jeśli task nie ma Assignee i LICZBA_KONTAKTÓW przekracza 1, ustaw Priorytet na Pilny. Ta osoba aktywnie próbuje się z nami skontaktować — zasługuje na szybką uwagę.

**Moderator — wydobycie niezgodności:**

**Niezgodność: Okno czasowe dla dopasowania dedup**

- **Tomasz**: 7 dni. Praktyczne, Data Store pozostaje mały, obejmuje typowe okno.
- **Karolina**: 14 dni byłoby bezpieczniejsze. Niektórzy klienci czekają tydzień przed wypróbowaniem innego kanału.
- **Aleksander**: 7 dni. Dłuższe okna zwiększają ryzyko fałszywych alarmów — ta sama osoba, autentycznie nowe zapytanie o inną datę.

Moderator zapytał Karolinę: „Co się stanie, jeśli okno 14-dniowe złapie autentycznie nowe zapytanie?" Przyznała: „Połączony komentarz byłby mylący. Handlowiec zobaczyłby starsze zapytanie, które już obsłużył, z nowym wciśniętym do środka. To gorsze niż przeoczony dedup."

**Rozwiązanie:** 7 dni, zapisane jako konfigurowalna zmienna scenariusza, żeby zespół mógł dostosować po obserwacji rzeczywistych danych.

**Niezgodność: Podejście do modyfikacji scenariusza**

- **Aleksander**: Sprawdzenie dedup musi nastąpić PRZED wywołaniem AI, żeby oszczędzać koszty i uniknąć zbędnej klasyfikacji.
- **Tomasz**: Oznacza to przestrukturyzowanie kolejności modułów w obu scenariuszach Phase 1, nie tylko dodanie gałęzi. To bardziej ryzykowne niż czysta zmiana addytywna.

Moderator zapytał Tomasza: „Jakie jest rzeczywiste ryzyko przestawienia modułów?" Tomasz: „Jeśli przeniosę sprawdzenie dedup przed wywołanie AI, a lookup Data Store się nie powiedzie, może zablokować cały scenariusz. Wywołanie AI miało własny error handler — teraz potrzebuję obsługi błędów również na lookup Data Store."

**Rozwiązanie:** Sprawdzenie dedup idzie przed wywołaniem AI (argument Aleksandra o kosztach i logice jest słuszny), ALE lookup Data Store dostaje własny error handler: przy awarii lookup, zakładaj „brak dopasowania" i proceed normalnie. W ten sposób awaria Data Store degraduje do „brak dedup" zamiast „brak intake".

---

### Element pracy 3: Generowanie roboczych odpowiedzi

#### Karolina Wysocka — Projektant procesu sprzedaży i obsługi

[REKOMENDACJA] To jest funkcja, którą zespół sprzedaży poczuje najbardziej. Dwa odrębne przypadki użycia z różnymi szablonami odpowiedzi:

**Przypadek użycia A — Odpowiedź na niekompletny lead:** Klient złożył zapytanie, ale brakuje kluczowych pól (data, miasto, liczba osób). Wersja robocza powinna ciepło potwierdzić zainteresowanie, potwierdzić co zrozumieliśmy i konkretnie zapytać o to, czego brakuje. To NAJWYŻSZA WARTOŚĆ przypadku użycia, ponieważ jest to najbardziej powtarzalne — handlowcy piszą warianty „dziękujemy za kontakt, czy możesz podać datę, miasto i liczebność grupy?" dziesiątki razy w tygodniu. Szablon jest na tyle formulaiczny, że AI radzi sobie z nim dobrze.

**Przypadek użycia B — Potwierdzenie gorącego leada:** Klient złożył kompletne zapytanie. Wersja robocza powinna potwierdzić odbiór, potwierdzić kluczowe szczegóły, wyrazić entuzjazm i określić kolejne kroki (np. „przygotujemy ofertę" lub „ktoś do ciebie zadzwoni"). To wartościowe, ale bardziej wrażliwe — ton musi odpowiadać marce EVSO.

[KRYTYCZNE] Chcę zwrócić uwagę na coś, czego żadne z nas nie może rozwiązać na tym spotkaniu: **nie mamy przykładowych odpowiedzi od EVSO.** Bez prawdziwych przykładów tego, jak zespół aktualnie pisze do klientów, prompt AI wyprodukuje generyczny tekst. Inżynier MUSI zebrać 5-10 prawdziwych przykładów odpowiedzi od zespołu sprzedaży przed napisaniem system promptów. To twardy warunek wstępny.

[REKOMENDACJA] Wdróż etapami:
- Etap 1: Wersje robocze dla niekompletnych leadów (tygodnie 1-2)
- Etap 2: Po walidacji tonu przez zespół i zebraniu feedbacku, dodaj wersje robocze dla gorących leadów (tydzień 3)

Daje to zespołowi bezpieczne wejście do treści generowanych przez AI.

#### Aleksander Bryła — Architekt systemów AI i niezawodności

[REKOMENDACJA] Projekt techniczny systemu roboczych odpowiedzi:

**Model:** Ten sam `claude-haiku-4-5-20251001` — zadanie to prosta generacja tekstu, Haiku wystarczy.

**Kiedy generować:** Na końcu scenariusza Make intake, PO tworzeniu taska. Dodaj gałąź router'a, która odpala się dla KLASYFIKACJA_AI = „Gorący lead" lub „Niekompletny lead" tylko. Nigdy nie generuj wersji roboczych dla Spam lub Zapytanie ogólne.

**Gdzie przechowywać wersję roboczą:** Jako komentarz ClickUp w tasku, poprzedzony wyraźnym znacznikiem, żeby handlowiec wiedział, że to wersja robocza wygenerowana przez AI. NIE w opisie taska (to oryginalne zapytanie) i NIE jako Custom Field (za krótkie, niemodyfikowalne w miejscu).

**Granice AI — ścisłe:**
1. Wersja robocza NIE MOŻE zawierać żadnych cen, rabatów ani zobowiązań finansowych.
2. Wersja robocza NIE MOŻE obiecywać konkretnych dat, dostępności ani pojemności.
3. Wersja robocza NIE MOŻE składać zobowiązań w imieniu EVSO (np. „na pewno pomieścimy").
4. Wersja robocza MUSI być po polsku, dopasowana do semi-casualnego profesjonalnego tonu EVSO.
5. Wersja robocza MUSI być wyraźnie oznaczona jako wygenerowana przez AI.

**Gating przez confidence:** Jeśli AI_CONFIDENCE z klasyfikacji wynosiło „Niska", NIE generuj wersji roboczej. Sama klasyfikacja jest niepewna — wersja robocza oparta na błędnej klasyfikacji byłaby myląca.

**Potrzebne dwa osobne system prompty:**

Dla niekompletnych leadów — prompt otrzymuje: które pola brakują, jakie pola są dostępne, oryginalny tekst zapytania, imię klienta. Generuje ciepłe follow-up pytające o konkretne brakujące szczegóły.

Dla gorących leadów — prompt otrzymuje: wszystkie wyekstrahowane pola, oryginalny tekst zapytania, imię klienta, typ usługi. Generuje potwierdzenie/acknowledgment odbijający z powrotem to, co zrozumieliśmy.

Oba prompty wymagają kalibracji głosu marki EVSO na podstawie prawdziwych przykładów odpowiedzi. Bez tego, flaguję output jako „generyczny — zweryfikuj przed wdrożeniem." [Dotyczy tutaj niepewność U-003.]

**Fallback przy awarii:** Jeśli wywołanie API generowania wersji roboczej się nie powiedzie, żaden komentarz nie jest zamieszczany. Task istnieje bez wersji roboczej — handlowiec pisze ręcznie, jak robi to dzisiaj. To graceful degradation, nie krytyczna awaria.

#### Tomasz Sowa — Architekt automatyzacji Make

[REKOMENDACJA] Implementacja w Make:

Generowanie wersji roboczych to addytywna gałąź do istniejących scenariuszy Phase 1. Po module ClickUp Create Task, dodaj:

1. **Router** — sprawdza wartość KLASYFIKACJA_AI ORAZ AI_CONFIDENCE
2. **Gałąź A (niekompletny lead):** HTTP call do Anthropic API z promptem dla niekompletnego leada → Parsuj odpowiedź → ClickUp: Create Task Comment
3. **Gałąź B (gorący lead):** HTTP call do Anthropic API z promptem dla gorącego leada → Parsuj odpowiedź → ClickUp: Create Task Comment
4. **Gałąź C (wszystkie inne):** Brak akcji — scenariusz kończy się

Komentarz jest tworzony przy użyciu modułu ClickUp `Create a Comment` na właśnie stworzonym tasku. Treść komentarza zawiera tekst wersji roboczej poprzedzony znacznikiem AI.

Ważne: Gałąź generowania wersji roboczej musi mieć własny error handler oddzielny od głównego scenariusza. Jeśli generowanie wersji roboczej się nie powiedzie, task został już z powodzeniem utworzony — po prostu nie dostajemy wersji roboczej. Error handler powinien: zalogować awarię, pominąć komentarz i pozwolić scenariuszowi zakończyć się normalnie.

Dla ścieżki dedup: Jeśli wykryto duplikat i łączymy go z istniejącym taskiem (bez tworzenia nowego taska), NIE generuj wersji roboczej. Oryginalny task może już mieć wersję roboczą lub być w aktywnej obsłudze.

#### Marta Kurek — Architekt systemów ClickUp

[REKOMENDACJA] Format komentarza z wersją roboczą powinien być:

```
🤖 WERSJA ROBOCZA ODPOWIEDZI (wygenerowana przez AI — do edycji przed wysłaniem)
---
[tekst wersji roboczej tutaj]
---
⚠️ Przejrzyj i dostosuj przed wysłaniem. Nie zawiera cen ani zobowiązań.
```

To jest komentarz, nie wersja robocza emaila. Handlowiec go czyta, kopiuje przydatne fragmenty do composera emaila, edytuje według potrzeb i wysyła. Zachowuje pełną kontrolę człowieka.

[FAKT] Composer emaila ClickUp w tasku obsługuje kopiuj-wklej. Workflow to: otwórz task → przeczytaj wersję roboczą AI w komentarzach → otwórz composer emaila → wklej/edytuj → wyślij. Jest to naturalne dla zespołu, ponieważ już używają obszaru komentarzy i composera emaila w tym samym widoku taska.

Żadne nowe Custom Fields nie są potrzebne dla tej funkcji. Wersja robocza żyje w strumieniu komentarzy.

**Moderator — wydobycie niezgodności:**

**Niezgodność: Etapowe wdrożenie vs. oba naraz**

- **Karolina**: Wdrożenie najpierw wersji roboczych dla niekompletnych leadów (Etap 1), wersji roboczych dla gorących leadów później (Etap 2). Powód: ryzyko tonu. Jeśli wersje robocze dla gorących leadów brzmią źle, zespół traci zaufanie do całego systemu.
- **Tomasz**: Techniczny koszt dodania obu naraz jest bliski zeru — to dwie gałęzie tego samego router'a. Opóźnianie Etapu 2 oznacza późniejsze ponowne wdrożenie, co jest zbędnym nakładem inżynierskim.
- **Aleksander**: Oba prompty wymagają kalibracji głosu marki. Jeśli nie mamy przykładów odpowiedzi, OBA będą generyczne. Ale prompt dla niekompletnych leadów jest bardziej formulaiczny („czy możesz podać X, Y, Z?") i mniej wrażliwy tonalnie. Więc etapowanie Karoliny ma sens z perspektywy ryzyka.

Moderator zapytał Tomasza: „Czy jest techniczny koszt budowania obu gałęzi teraz, ale włączenia tylko jednej?" Tomasz: „Mógłbym zbudować obie gałęzie i wyłączyć gałąź gorących leadów zmienną scenariusza. Zero kosztu późniejszego wdrożenia, a kod jest już przetestowany."

**Rozwiązanie:** Zbuduj obie gałęzie w scenariuszu. Wdróż z gałęzią gorących leadów WYŁĄCZONĄ przez zmienną scenariusza. Włącz ją po 1-2 tygodniach feedbacku zespołu na temat wersji roboczych dla niekompletnych leadów. Spełnia to obawy ryzyka Karoliny ORAZ obawy efektywności Tomasza.

**Niezgodność: Warunek wstępny — prawdziwe przykłady odpowiedzi**

- **Karolina**: Twardy warunek wstępny. Bez przykładów odpowiedzi, nie wdrażaj.
- **Tomasz**: To blokuje całą funkcję na zależności niestechnicznej (zbieranie przykładów od zespołu sprzedaży).
- **Aleksander**: Karolina ma rację. Prompt bez kalibracji wyprodukuje generyczny korporacyjny tekst. Ale możemy napisać „kalibrowalny" prompt ze zmienną `{{przykłady_głosu_marki}}`, który zaczyna pusty i jest wypełniany, gdy przykłady są zebrane. Wstępny prompt działa — produkuje po prostu bardziej mdłe wyniki.

**Rozwiązanie:** Zebranie 5-10 prawdziwych przykładów odpowiedzi to zadanie Dnia 1 w sekwencji wdrożenia. System prompt zawiera sekcję głosu marki. Jeśli przykłady nie są dostępne w dniu wdrożenia, wdróż z wersją generyczną i zaktualizuj prompt, gdy przykłady dotrą. Oznaczone jako U-003 w rejestrze niepewności.

---

### Element pracy 4: Automatyzacja obiegu ZLECENIA

#### Marta Kurek — Architekt systemów ClickUp

[KRYTYCZNA NIEPEWNOŚĆ] Nie znamy obecnego procesu handoff CRM→ZLECENIA. Zrzuty ekranu potwierdzają istnienie tasków ZLECENIA, ale nie jak się tam dostają.

**Dwa możliwe scenariusze obecnego stanu:**

**Scenariusz A — Przeniesienie taska:** Task jest przenoszony z listy pipeline sprzedawcy (np. SPRZEDAŻ KUBA) do listy ZLECENIA. Task zachowuje swój ID, historię i pola. Niektóre pola mogą wymagać dodania lub aktualizacji po przeniesieniu.

**Scenariusz B — Tworzenie nowego taska:** Nowy task jest tworzony w ZLECENIA z danymi skopiowanymi z taska CRM. Task CRM pozostaje w pipeline sprzedawcy (być może przeniesiony do statusu „WYGRANA" lub archiwum). Task ZLECENIA to świeży rekord z własną historią.

[FAKT] Nazwy tasków ZLECENIA zawierają PAKIET w konwencji nazewnictwa: `[DATA] | [MIASTO] | [USŁUGA] | [KLIENT] | [PAKIET]` — jest to inne niż nazewnictwo CRM, które nie zawsze zawiera PAKIET. Sugeruje to, że podczas przejścia dochodzi do wzbogacenia danych.

[FAKT] ZLECENIA ma własny przepływ statusów (ZLECENIA BEZ ZALICZEK → REZERWACJA JEDNOSTKI → ZLECENIA → itd.), który jest niezwiązany ze statusami CRM. To odrębna domena workflow.

[REKOMENDACJA] Przed budowaniem jakiejkolwiek automatyzacji, inżynier musi zweryfikować: (a) Czy to przeniesienie czy nowy task? (b) Jakie dane są dodawane podczas przejścia? (c) Czy jest spójny punkt trigger'a (konkretny status lub akcja ręczna)?

[REKOMENDACJA] Niezależnie od tego, który scenariusz jest obecny, potrzebujemy spójnego statusu „wygranej sprzedaży" we wszystkich pipeline'ach sprzedawców. Dodaj `SPRZEDANE` jako status do każdej listy pipeline sprzedawcy. To punkt trigger'a dla automatyzacji.

#### Tomasz Sowa — Architekt automatyzacji Make

[REKOMENDACJA] Zaprojektuję oba scenariusze. Inżynier implementuje ten, który odpowiada rzeczywistości.

**Projekt A — Podejście z przeniesieniem taska:**

Jeśli obecny proces to przenoszenie tasków, automatyzacja powinna być natywną automatyzacją ClickUp:
- Trigger: Gdy status taska zmienia się na `SPRZEDANE` w dowolnej liście CRM sprzedawcy
- Akcja: Przenieś task do listy ZLECENIA + Zmień status na `ZLECENIA BEZ ZALICZEK` (pierwszy status operacyjny)

Następnie scenariusz Make obsługuje wzbogacenie po przeniesieniu:
- Trigger: Webhook ClickUp — task zaktualizowany (zmiana statusu na ZLECENIA BEZ ZALICZEK w liście ZLECENIA)
- Moduł 1: Pobierz szczegóły taska
- Moduł 2: Zaktualizuj nazwę taska, dodając PAKIET: `{{data}} | {{miasto}} | {{usługa}} | {{klient}} | {{pakiet}}`
- Moduł 3: Ustaw pola specyficzne dla ZLECENIA
- Moduł 4: Opublikuj komentarz: „Task przeniesiony z CRM. Zlecenie utworzone automatycznie."

**Projekt B — Podejście z tworzeniem nowego taska:**

Scenariusz Make tworzy task ZLECENIA:
- Trigger: Webhook ClickUp — status taska zmieniony na `SPRZEDANE` w dowolnej liście CRM sprzedawcy
- Moduł 1: Pobierz pełne szczegóły taska CRM (wszystkie Custom Fields)
- Moduł 2: Utwórz nowy task w liście ZLECENIA z zmapowanymi polami i rozszerzoną konwencją nazewnictwa
- Moduł 3: Ustaw status taska CRM na `ARCHIWUM` lub podobny stan końcowy
- Moduł 4: Dodaj komentarz do taska ZLECENIA linkujący do oryginalnego taska CRM
- Moduł 5: Dodaj komentarz do taska CRM linkujący do nowego taska ZLECENIA

Projekt B jest bardziej złożony, ale czystszy architektonicznie — CRM i ZLECENIA mogą ewoluować niezależnie.

[WNIOSKOWANIE] Na podstawie danych widocznych na zrzutach ekranu (taski ZLECENIA mają inny nacisk pól — OBSŁUGA, PAKIET są widoczne; nie ma widocznego LEAD-#### w taskach ZLECENIA), Projekt B (tworzenie nowego taska) jest bardziej prawdopodobnym obecnym wzorcem. Ale to wnioskowanie, nie potwierdzenie.

#### Aleksander Bryła — Architekt systemów AI i niezawodności

Brak zaangażowania AI. Przejście CRM→ZLECENIA jest deterministyczne. Dane są już ustrukturyzowane — nie ma nic do klasyfikowania ani ekstrakcji.

Jedna drobna obserwacja: jeśli task CRM ma AI_PODSUMOWANIE, mogłoby być przydatne przeniesienie go do taska ZLECENIA jako kontekst operacyjny. Ale to nice-to-have, nie wymóg.

#### Karolina Wysocka — Projektant procesu sprzedaży i obsługi

[REKOMENDACJA] Trigger musi być AKCJĄ CZŁOWIEKA (zmiana statusu na SPRZEDANE), nie automatyczną oceną. Decyzja „ten lead jest wygrany" to sąd biznesowy, który musi pozostać przy handlowcu.

Mapowanie pól z CRM do ZLECENIA — minimalny zestaw:

| Pole CRM | Mapowanie ZLECENIA | Uwagi |
|----------|-------------------|-------|
| Nazwa taska | Rozszerzona: dodaj PAKIET | Konwencja nazewnictwa wymaga PAKIETU |
| MIASTO | MIASTO | Bezpośrednia kopia |
| USŁUGA | USŁUGA | Bezpośrednia kopia |
| PAKIET | PAKIET | Bezpośrednia kopia |
| DATA EVENTU | Due date / DATA EVENTU | Staje się operacyjnym terminem |
| EMAIL | EMAIL | Kontakt operacyjny |
| TELEFON | TELEFON | Kontakt operacyjny |
| ILOŚĆ OSÓB | ILOŚĆ OSÓB | Potrzebne do planowania pojemności |
| GODZINA EVENTU | GODZINA EVENTU | Timing operacyjny |
| Opis | Opis | Zachowaj oryginalne zapytanie + wszelką korespondencję |

[FAKT] ZLECENIA ma widoczne pola nieobecne w intake CRM: OBSŁUGA (np. „BEZOBSŁUGOWY"). To pole musi być ustawiane ręcznie podczas lub po przejściu — to decyzja operacyjna, nie przechwytywana przy intake.

„Automatyzacja tworzy task ZLECENIA ze wszystkimi przenoszalnymi danymi. Zespół operacyjny następnie wypełnia OBSŁUGA, JEDNOSTKA i inne pola specyficzne dla realizacji. To jest prawidłowy podział pracy."

**Moderator — wydobycie niezgodności:**

**Niezgodność: Projekt A (przeniesienie) vs. Projekt B (nowy task)**

- **Marta**: Oba są poprawne. Projekt B jest czystszy architektonicznie, ale tworzy duplikację danych. Projekt A zachowuje pełną historię w jednym miejscu.
- **Tomasz**: Projekt B jest bardziej odporny, ponieważ CRM i ZLECENIA mogą ewoluować niezależnie. W Projekcie A, przeniesienie taska do listy z innymi definicjami Custom Fields może powodować problemy z widocznością pól.
- **Karolina**: Zespół sprzedaży musi widzieć swoje wygrane transakcje w pipeline CRM nawet po tym, jak task ZLECENIA istnieje — dla śledzenia prowizji, pokazywania postępów. Jeśli przeniesiemy task, znika z ich widoku.

Moderator zapytał: „Karolina, czy to oznacza, że Projekt B jest wymagany, żeby zachować rekord CRM?" Karolina: „Tak. Task CRM powinien pozostać w pipeline sprzedawcy, oznaczony jako SPRZEDANE. Tworzony jest nowy task ZLECENIA. Oba istnieją."

**Rozwiązanie:** Projekt B (tworzenie nowego taska) jest zalecanym podejściem. Task CRM pozostaje w pipeline sprzedawcy ze statusem SPRZEDANE. Nowy task jest tworzony w ZLECENIA ze zmapowanymi polami. Komentarze z wzajemnymi linkami są dodawane do obu tasków. JEDNAK: inżynier musi zweryfikować, czy to odpowiada obecnemu procesowi ręcznemu, przed implementacją.

**Niezgodność: Inwestowanie 4 dni w automatyzację ZLECENIA vs. redukcja zakresu**

- **Marta**: Automatyzacja ZLECENIA to element o największym ryzyku z powodu U-001. Jest też w osobnej domenie operacyjnej. Jeśli całkowity czas jest ograniczony, ten jest do cięcia lub redukcji.
- **Tomasz**: Zgadzam się, ale inżynier może spędzić Dzień 1 na śledztwie i jeśli proces jest jasny, implementacja jest prosta. Jeśli śledztwo ujawni nieoczekiwaną złożoność, możemy odłożyć części.
- **Karolina**: To oszczędza najwięcej czasu ręcznego na event. Każda wygrana transakcja wymaga 5-10 minut ręcznego tworzenia taska w ZLECENIA. Przy może 20-30 eventach miesięcznie, to 2-5 godzin powtarzalnej pracy.

**Rozwiązanie:** Uwzględnij automatyzację ZLECENIA, ale z jawnym „dniem śledztwa" (Dzień 1 elementu pracy). Jeśli śledztwo ujawni, że proces jest bardziej złożony niż oczekiwano, inżynier ma prawo do zredukowania zakresu do: (a) dodania tylko statusu SPRZEDANE, (b) udokumentowania procesu do implementacji w Phase 3.

---

## Niezgodności

### Niezgodność 1: Okno czasowe dedup

- **Strony**: Tomasz + Aleksander vs. Karolina
- **Typ**: Predykcyjna (jakie zachowanie klientów oczekiwać)
- **Pozycja A (7 dni)**: Krótsze okno redukuje fałszywe alarmy. Utrzymuje Data Store mały. Okno 14-dniowe mogłoby scalać autentycznie różne zapytania tej samej osoby.
- **Pozycja B (14 dni)**: Niektórzy klienci czekają tydzień przed wypróbowaniem innego kanału. Przeoczenie tych duplikatów niweczy cel.
- **Co zmieniłoby zdanie**: Rzeczywiste dane o odstępie między pierwszym a drugim kontaktem tego samego klienta. Po miesiącu działania, analiza Data Store, żeby zobaczyć ile dopasowań dedup pojawia się i w jakim wieku.
- **Rozwiązanie**: 7 dni, zaimplementowane jako zmienna konfigurowalna. Ponowna ocena po 30 dniach działania z rzeczywistymi danymi.

### Niezgodność 2: Etapowanie wdrożenia roboczych odpowiedzi

- **Strony**: Karolina vs. Tomasz
- **Typ**: Wartości (tolerancja ryzyka dla treści skierowanych do klientów)
- **Pozycja A (etapowo)**: Wdrożenie najpierw wersji roboczych dla niekompletnych leadów, dla gorących leadów później. Ryzyko tonu jest realne i szkodliwe.
- **Pozycja B (oba naraz)**: Koszt techniczny jest zerowy; opóźnienie jest zbędnym nakładem.
- **Co zmieniłoby zdanie**: Jeśli przykłady głosu marki są zebrane w Dniu 1 i oba prompty są zatwierdzane przez zespół sprzedaży przed wdrożeniem, jednoczesne wdrożenie staje się bezpieczne.
- **Rozwiązanie**: Zbuduj oba, wdróż etapowo. Gałąź gorących leadów wyłączona przez toggle zmiennej. Włącz po feedbacku zespołu.

### Niezgodność 3: ZLECENIA Projekt A vs. B

- **Strony**: (Brak silnej opozycji — bardziej potrzeba wyjaśnienia)
- **Typ**: Faktyczna (zależy od tego, jaki jest obecny proces)
- **Pozycja A (przeniesienie taska)**: Prostszy, zachowuje historię.
- **Pozycja B (nowy task)**: Czystsza separacja, rekord CRM zachowany dla zespołu sprzedaży.
- **Co zmieniłoby zdanie**: Znajomość obecnego procesu ręcznego.
- **Rozwiązanie**: Zalecany Projekt B. Inżynier weryfikuje przed implementacją. Jeśli obecny proces to faktycznie przeniesienie taska i zespół go preferuje, użyj Projektu A.

---

## Zbieżność

### Synteza

Zespół zbiegł do czterech funkcji z jasnymi projektami technicznymi:

1. **Przypomnienia o follow-upach** — Najlżejsza zmiana. Due Date ustawiane przez Make przy tworzeniu taska; natywna automatyzacja ClickUp oznacza przeterminowane nieprzypisane taski. Żadnych nowych scenariuszy nie potrzeba. ~1,5 dnia.

2. **Deduplicacja leadów** — Umiarkowana złożoność. Lookup Data Store Make przed tworzeniem taska w obu istniejących scenariuszach. Przy dopasowaniu: scalenie jako komentarz + aktualizacja pól + tag. Okno 7-dniowe, konfigurowalne. Wymaga przestawienia modułów w scenariuszach Phase 1 (dedup przed AI). ~3 dni.

3. **Generowanie roboczych odpowiedzi** — Największy wpływ widoczny dla użytkownika. Gałąź router'a dodana do istniejących scenariuszy po tworzeniu taska. Dwa prompty AI (niekompletny lead, gorący lead). Przechowywane jako komentarze ClickUp. Etapowe wdrożenie: najpierw niekompletne, gorące później. Twardy warunek wstępny: zebranie prawdziwych przykładów odpowiedzi. ~3,5 dnia.

4. **Automatyzacja ZLECENIA** — Największa niepewność. Nowy status SPRZEDANE dodany do wszystkich pipeline'ów sprzedawców. Nowy scenariusz Make tworzy task w ZLECENIA z danych CRM. Projekt B (nowy task). Wymaga dnia śledztwa. ~4 dni (włącznie ze śledztwem).

**Łącznie: ~12 dni.** W ramach ograniczenia 15 dni z buforem 3 dni.

Kluczowy trade-off zaakceptowany przez cały zespół: Phase 2 modyfikuje istniejące scenariusze Phase 1 (przestawienie sprawdzenia dedup, gałąź generowania wersji roboczych, pole Due Date) zamiast tworzenia całkowicie osobnych scenariuszy. Jest to bardziej efektywne, ale niesie ryzyko wprowadzenia regresji w stabilnym przepływie intake. Mitygacja: testowanie każdej modyfikacji względem listy kontrolnej smoke testów Phase 1 przed wdrożeniem.

### Rekordy decyzji

| Pole | Treść |
|------|-------|
| **ID** | D-001 |
| **Decyzja** | Przypomnienia o follow-upach używają natywnego Due Date ClickUp + Automatyzacji, a nie scenariuszy Make |
| **Uzasadnienie** | ClickUp obsługuje trigger'y czasowe natywnie i niezawodnie. Budowanie w Make dodałoby kruchość (polling), koszty (operacje) i opóźnienie. Due Date dostarcza też wizualny termin w widokach kalendarza/listy. |
| **Sprzeciw** | Brak |
| **Pewność** | Wysoka — wszyscy czterej eksperci zgadzają się, dokumentacja ClickUp potwierdza możliwości |
| **Ponowna ocena jeśli** | Plan ClickUp nie obsługuje wymaganego typu trigger'a automatyzacji; lub jeśli potrzebne są powiadomienia zewnętrzne (Slack, SMS) — te wymagałyby Make |

| Pole | Treść |
|------|-------|
| **ID** | D-002 |
| **Decyzja** | Deduplicacja leadów używa Data Store Make z dopasowaniem tylko emailowym, 7-dniowym konfigurowalnym oknem |
| **Uzasadnienie** | Email jest najbardziej niezawodnym identyfikatorem między kanałami. Dopasowanie telefoniczne dodaje złożoność normalizacji. 7 dni obejmuje typowe okno zapytanie-follow-up bez ryzyka fałszywych alarmów. Data Store jest prostszy niż wyszukiwanie API ClickUp (które wymagałoby paginowanych zapytań po listach o zmiennej długości). |
| **Sprzeciw** | Karolina wolała 14 dni. Zaakceptowała ryzyko okna 7-dniowego po przyznaniu, że fałszywe alarmy są gorsze. |
| **Pewność** | Średnia — okno 7-dniowe to wykształcone przypuszczenie. Rzeczywiste dane potwierdzą lub wymagają korekty. |
| **Ponowna ocena jeśli** | Po 30 dniach, jeśli analiza pokaże znaczące nieuchwyty duplikatów poza oknem 7-dniowym |

| Pole | Treść |
|------|-------|
| **ID** | D-003 |
| **Decyzja** | Sprawdzenie dedup jest umieszczone PRZED wywołaniem klasyfikacji AI w kolejności modułów scenariusza |
| **Uzasadnienie** | Oszczędność kosztów API na duplikatach. Logicznie poprawne: duplikat nie potrzebuje niezależnej klasyfikacji. Obsługa błędów lookup Data Store domyślnie ustawia „brak dopasowania", żeby awarie dedup nie blokały intake. |
| **Sprzeciw** | Tomasz zauważył ryzyko restrukturyzacji scenariusza, ale zaakceptował z mitygacją obsługi błędów. |
| **Pewność** | Wysoka — logika jest poprawna, obsługa błędów pokrywa ryzyko |
| **Ponowna ocena jeśli** | Data Store ma problemy z niezawodnością na produkcji |

| Pole | Treść |
|------|-------|
| **ID** | D-004 |
| **Decyzja** | Robocze odpowiedzi są przechowywane jako komentarze taska ClickUp ze znacznikiem wygenerowanym przez AI, nie w Custom Fields ani opisie |
| **Uzasadnienie** | Komentarze są widoczne w panelu aktywności, edytowalne w kontekście i nie interferują ze strukturalnymi danymi taska. Opis musi pozostać archiwum oryginalnego zapytania. Custom Fields są za krótkie na tekst odpowiedzi. |
| **Sprzeciw** | Brak |
| **Pewność** | Wysoka |
| **Ponowna ocena jeśli** | ClickUp wprowadzi natywne API „wersja robocza emaila", które mogłoby być użyte zamiast tego |

| Pole | Treść |
|------|-------|
| **ID** | D-005 |
| **Decyzja** | Wdrożenie roboczych odpowiedzi jest etapowe: najpierw wersje robocze dla niekompletnych leadów, wersje robocze dla gorących leadów włączone po 1-2 tygodniach feedbacku |
| **Uzasadnienie** | Odpowiedzi dla niekompletnych leadów są bardziej formulaiczne i niższego ryzyka. Etapowe wdrożenie buduje zaufanie zespołu przed ekspozycją na bardziej wrażliwe tonalnie wersje robocze dla gorących leadów. Techniczny koszt etapowania jest zerowy (toggle zmiennej). |
| **Sprzeciw** | Tomasz: brak technicznego powodu dla etapowania. Zaakceptował argument ryzyka procesu. |
| **Pewność** | Wysoka — to decyzja zarządzania procesem, nie techniczna |
| **Ponowna ocena jeśli** | Przykłady głosu marki są zebrane wcześnie ORAZ zespół jawnie mówi „włącz oba" |

| Pole | Treść |
|------|-------|
| **ID** | D-006 |
| **Decyzja** | Automatyzacja ZLECENIA używa Projektu B (tworzenie nowego taska) ze statusem SPRZEDANE jako trigger'em |
| **Uzasadnienie** | Task CRM musi pozostać w pipeline sprzedawcy dla widoczności zespołu sprzedaży. ZLECENIA to osobna domena operacyjna z innymi potrzebami danych. Tworzenie nowego taska zapewnia czystą separację. Wzajemne linki zachowują identyfikowalność. |
| **Sprzeciw** | Marta zauważyła, że Projekt A (przeniesienie) jest prostszy, ale zaakceptowała Projekt B na podstawie argumentu workflow Karoliny. |
| **Pewność** | Średnia — zależy od U-001 (weryfikacji obecnego procesu). Inżynier ma prawo przełączyć na Projekt A, jeśli śledztwo ujawni, że to jest ustalony wzorzec. |
| **Ponowna ocena jeśli** | Śledztwo pokaże, że obecny proces to przeniesienie taska ORAZ zespół sprzedaży woli utratę rekordu CRM |

| Pole | Treść |
|------|-------|
| **ID** | D-007 |
| **Decyzja** | Element pracy ZLECENIA zawiera jawny dzień śledztwa; zakres może być zredukowany, jeśli proces jest bardziej złożony niż oczekiwano |
| **Uzasadnienie** | U-001 jest nierozwiązywalne na tym spotkaniu. Alokowanie czasu na śledztwo zapobiega budowaniu na błędnych założeniach. Prawo do redukcji zakresu zapobiega pochłonięciu przez element pracy ZLECENIA nieproporcjonalnego czasu. |
| **Sprzeciw** | Brak |
| **Pewność** | Wysoka — to decyzja zarządzania ryzykiem |
| **Ponowna ocena jeśli** | Śledztwo ujawni trywialnie prosty proces (wtedy pomiń czas buforowy) |

### Mapa trade-offów

- **Wybór modyfikacji scenariuszy Phase 1 (gałęzie dedup + wersji roboczych) vs. tworzenia nowych scenariuszy**: Zyski efektywności i ponownego użycia kodu. Koszty ryzyka regresji w stabilnym przepływie intake. Preferowane przez Tomasza (mniej duplikacji) i Aleksandra (dedup musi poprzedzać wywołanie AI). Mitygowane przez ponowne uruchomienie smoke testów Phase 1.

- **Wybór 7-dniowego okna dedup vs. 14-dniowego**: Zyski precyzji (mniej fałszywych alarmów). Koszty recall'u (może przeoczyć niektóre późne duplikaty). Preferowane przez Tomasza + Aleksandra. Karolina zaakceptowała z mitygacją konfigurowalnej zmiennej.

- **Wybór etapowego wdrożenia roboczych odpowiedzi vs. jednoczesnego**: Zyski zaufania i bezpieczeństwa dla zespołu sprzedaży. Koszty 1-2 tygodniowego opóźnienia wersji roboczych gorących leadów. Preferowane przez Karolinę + Aleksandra. Tomasz zaakceptował z mechanizmem toggle.

- **Wybór Projektu B (nowy task ZLECENIA) vs. Projektu A (przeniesienie)**: Zyski czystej separacji domen i zachowania rekordu CRM. Koszty dodatkowej złożoności (dwa taski, wzajemne linki, potencjalny dryft danych). Preferowane przez Karolinę (widoczność sprzedaży) i Tomasza (czystość architektoniczna).

---

## Rejestr niepewności

| ID | Nieznane | Wpływ jeśli błędne | Rozwiązywalne? | Proponowana akcja | Właściciel |
|----|----------|---------------------|----------------|-------------------|------------|
| U-001 | Obecny proces handoff CRM→ZLECENIA (przeniesienie vs. nowy task) | Błędny projekt automatyzacji; stracony czas inżynierski | Tak — zapytaj zespół lub zaobserwuj ich workflow | Inżynier bada w Dniu 1 ZLECENIA | Tomasz Sowa |
| U-002 | Czy wszystkie pipeline'y sprzedawców mają te same statusy | Status SPRZEDANE może wymagać różnych nazw lub pozycji w różnych listach | Tak — sprawdź każdą listę w ClickUp | Inżynier sprawdza wszystkie 4 listy sprzedawców podczas konfiguracji ZLECENIA | Marta Kurek |
| U-003 | Głos marki EVSO dla odpowiedzi klientów | Robocze odpowiedzi mogą brzmieć generycznie lub nie pasować do marki | Częściowo — zbieranie przykładów pomaga, ale kalibracja jest iteracyjna | Zbierz 5-10 prawdziwych przykładów odpowiedzi od zespołu sprzedaży przed pisaniem promptów | Karolina Wysocka |
| U-004 | Poziom planu ClickUp (Business vs. niższy) dla możliwości automatyzacji | Trigger'y automatyzacji czasowych mogą nie być dostępne | Tak — sprawdź subskrypcję ClickUp | Inżynier weryfikuje w Dniu 1 implementacji | Marta Kurek |
| U-005 | Dziedziczenie Custom Fields między listami folderu CRM | Przenoszenie/kopiowanie tasków między listami może stracić wartości Custom Fields | Tak — przetestuj z jednym taskiem | Inżynier testuje podczas dnia śledztwa ZLECENIA | Marta Kurek |
| U-006 | Wydajność Data Store Make przy wolumenie EVSO | Przy ~100 leadach/tydzień i 7-dniowym retencji, Data Store przechowuje max ~100 rekordów. Powinno być dobrze, ale nieprzetestowane. | Tak — monitoruj po wdrożeniu | Monitoruj rozmiar Data Store i latencję lookup przez pierwsze 2 tygodnie | Tomasz Sowa |
| U-007 | Czy payload webhook'a WPForms zawiera wystarczające informacje dla sprawdzenia dedup PRZED wywołaniem AI | Jeśli email nie jest dostępny na etapie dedup, sprawdzenie nie może nastąpić | Tak — przetestuj strukturę payloadu webhook'a | Zweryfikuj podczas modyfikacji scenariusza | Tomasz Sowa |

---

## Otwarte pytania

1. **Jak zespół sprzedaży aktualnie tworzy taski ZLECENIA?** — Ważne, ponieważ decyduje o projekcie automatyzacji. Najlepiej odpowiedzieć: obserwując lub pytając zespół. Obecne założenie: Projekt B (tworzenie nowego taska).

2. **Czy EVSO ma wytyczne głosu marki lub szablony odpowiedzi?** — Ważne, ponieważ jakość roboczych odpowiedzi zależy od kalibracji tonu. Najlepiej odpowiedzieć: Karolina zbiera od zespołu sprzedaży. Obecne założenie: generyczny profesjonalny ton polski, dopracowany po zebraniu przykładów.

3. **Czy wszystkie pipeline'y sprzedawców są na tym samym szablonie listy ClickUp?** — Ważne, ponieważ status SPRZEDANE i trigger'y automatyzacji muszą działać we wszystkich listach. Najlepiej odpowiedzieć: inżynier sprawdza każdą listę. Obecne założenie: podobna struktura, może wymagać konfiguracji per lista.

4. **Na jakim planie ClickUp jest EVSO?** — Ważne dla dostępności funkcji automatyzacji. Najlepiej odpowiedzieć: sprawdź ustawienia konta. Obecne założenie: plan Business lub wyższy.

---

## Następny pakiet pracy

| Priorytet | Akcja | Uzasadnienie | Zależy od |
|-----------|-------|--------------|-----------|
| Wysoki | Zbierz 5-10 prawdziwych przykładów odpowiedzi od zespołu sprzedaży | Twardy warunek wstępny dla kalibracji promptu (U-003) | Dostępność zespołu sprzedaży |
| Wysoki | Zweryfikuj obecny proces handoff CRM→ZLECENIA (U-001) | Decyduje o projekcie automatyzacji ZLECENIA | Dostęp do obserwacji lub wywiadu z zespołem |
| Wysoki | Zweryfikuj poziom planu ClickUp dla możliwości automatyzacji (U-004) | Potwierdza podejście do przypomnień | Dostęp admina ClickUp |
| Wysoki | Napisz Przewodnik Wdrożenia Phase 2 na podstawie decyzji tego spotkania | Produkuje gotowy do budowy dokument | Output tego spotkania |
| Średni | Sprawdź wszystkie listy pipeline'ów sprzedawców pod kątem spójności statusów (U-002) | Potrzebne przed dodaniem statusu SPRZEDANE | Dostęp ClickUp |
| Średni | Przetestuj dziedziczenie Custom Fields przy przenoszeniu tasków między listami CRM (U-005) | Informuje o mapowaniu pól ZLECENIA | Dostęp ClickUp |

---

## Artefakty do aktualizacji

| Artefakt | Akcja | Konkretna zmiana | Wywołane przez |
|----------|-------|-----------------|----------------|
| Scenariusz Make Phase 1: Form Intake Enhanced | Aktualizacja | Dodaj ustawienie pola Due Date; dodaj zapis Data Store; przestaw moduły dla sprawdzenia dedup przed AI; dodaj gałąź router'a dla generowania roboczych odpowiedzi | D-001, D-002, D-003, D-004 |
| Scenariusz Make Phase 1: Email Intake | Aktualizacja | Te same zmiany co Form Intake Enhanced | D-001, D-002, D-003, D-004 |
| ClickUp: Custom Fields NOWE ZAPYTANIA | Aktualizacja | Dodaj pole: LICZBA_KONTAKTÓW (Number, domyślnie 1) | D-002 |
| ClickUp: Automatyzacje NOWE ZAPYTANIA | Utwórz | Nowa automatyzacja: „Gdy nadejdzie Due Date + brak Assignee → Priorytet = Pilny + komentarz" | D-001 |
| ClickUp: Wszystkie listy pipeline'ów sprzedawców | Aktualizacja | Dodaj status: SPRZEDANE | D-006 |
| Make: Nowy Data Store | Utwórz | EVSO_Lead_Dedup z polami email/task_id/created_at/phone | D-002 |
| Make: Nowy scenariusz | Utwórz | EVSO - ZLECENIA Task Creator (wyzwalany statusem SPRZEDANE) | D-006 |
| Make: Nowy scenariusz | Utwórz | EVSO - Dedup Cleanup (tygodniowy, usuwa rekordy > 7 dni) | D-002 |
| WIKI PRACOWNIKA | Aktualizacja | Dodaj dokumentację dla: zachowania dedup, funkcji roboczych odpowiedzi, logiki przypomnienia o follow-upach, automatycznego tworzenia ZLECENIA | Wszystkie decyzje |
| Smoke Testy Phase 1 | Aktualizacja | Ponowne uruchomienie pełnej listy smoke testów Phase 1 po modyfikacjach scenariuszy w celu potwierdzenia braku regresji | D-003 (przestawienie modułów) |

---

## Metadane spotkania

- **Użyty format**: Structured Workshop
- **Aktywowani eksperci**: 4 z 4
- **Podjęte decyzje**: 7
- **Zalogowane niepewności**: 7
- **Otwarte pytania**: 4
- **Szacunkowa pewność w stosunku do głównej rekomendacji**: Średnio-wysoka (wysoka dla funkcji 1-3, średnia dla ZLECENIA ze względu na U-001)
