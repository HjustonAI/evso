# EVSO — Automatyzacja Intake Leadów: Przewodnik Wdrożenia Phase 2

**Wersja:** 1.0
**Data:** 2026-03-31
**Zakres:** Automatyzacja post-intake — przypomnienia o follow-upach, deduplicacja leadów, generowanie roboczych odpowiedzi, automatyzacja obiegu ZLECENIA
**Docelowy czytelnik:** Jeden inżynier mid-level z doświadczeniem w Make.com i ClickUp
**Szacowany nakład:** 12 dni inżynierskich
**Warunek wstępny:** Phase 1 zaimplementowany i stabilny (oba scenariusze Make działają na produkcji, wszystkie 5 nowych Custom Fields jest wypełnianych, automatyzacja ClickUp aktywna)

**Walidacja przez panel ekspertów:** Wszystkie decyzje projektowe w tym dokumencie zostały zwalidowane na Spotkaniu Ekspertów M-002 (Structured Workshop) przez czterech specjalistów — architekturę ClickUp (Marta Kurek), automatyzację Make (Tomasz Sowa), niezawodność AI (Aleksander Bryła) i projektowanie procesów sprzedaży (Karolina Wysocka). Siedem rekordów decyzji, siedem pozycji niepewności i trzy rozwiązane niezgodności są udokumentowane w `EVSO_Phase2_Expert_Meeting_Output.md`.

---

## Sekcja 1 — Analiza stanu obecnego (po Phase 1)

### Co dostarczył Phase 1

Phase 1 stworzył funkcjonalny pipeline intake leadów z dwoma zautomatyzowanymi kanałami:

**Kanał A — Intake formularzy (scenariusz: „EVSO - Form Intake Enhanced"):** Zgłoszenia z WPForms z `partytram.fun`, `partyboat.fun` i `busparty.fun` są przetwarzane przez scenariusz Make.com, który normalizuje wartości miast, wyprowadza typ usługi z domeny, oblicza wynik kompletności, wywołuje Claude Haiku dla klasyfikacji i podsumowania, i tworzy w pełni wypełniony task ClickUp w NOWE ZAPYTANIA.

**Kanał B — Intake emailowy (scenariusz: „EVSO - Email Intake"):** Emaile przychodzące na adresy brandowe są przetwarzane przez scenariusz Make.com, który wykrywa markę, wywołuje Claude Haiku do ekstrakcji strukturalnych pól z nieustrukturyzowanego tekstu emaila, oblicza kompletność i tworzy task ClickUp ze wszystkimi dostępnymi danymi.

**Stan ClickUp po Phase 1:** NOWE ZAPYTANIA ma 16 Custom Fields (11 oryginalnych + 5 nowych: ŹRÓDŁO_KANAŁ, KLASYFIKACJA_AI, AI_PODSUMOWANIE, KOMPLETNOŚĆ, AI_CONFIDENCE). Jedna automatyzacja ClickUp oznacza taski z KLASYFIKACJA_AI = „Do weryfikacji" jako Pilne. Dwa widoki z filtrami (NOWE - PRIORYTET, NOWE - KOMPLETNOŚĆ) pomagają zespołowi sprzedaży ustalać priorytety.

**Stan AI po Phase 1:** Klucz Anthropic API przechowywany w Make.com. Dwa zadania AI operacyjne — Klasyfikacja formularza + Podsumowanie (Scenariusz 1, Moduł 4) i Ekstrakcja emaila + Klasyfikacja (Scenariusz 2, Moduł 3). Oba używają `claude-haiku-4-5-20251001` z error handlerami, które degradują gracefully do „Do weryfikacji".

### Gdzie nadal pozostaje tarcie po Phase 1

1. **Brak mechanizmu follow-upu dla nieobsłużonych leadów.** Task może siedzieć w NOWE ZAPYTANIA bez przypisanego handlowca w nieskończoność. Nie ma limitu czasu, przypomnienia ani eskalacji. Przy ~100 zapytaniach/tydzień, część leadów nieuchronnie wypada przez szczeliny.

2. **Zduplikowane leady między kanałami.** Klient może złożyć formularz I wysłać email w sprawie tego samego eventu. Phase 1 tworzy dwa osobne taski. Zespół sprzedaży odkrywa duplikaty tylko przez ręczne zauważenie tego samego imienia lub emaila — marnując czas i ryzykując sprzeczne odpowiedzi do tego samego klienta.

3. **Powtarzalne pisanie odpowiedzi.** Każdy lead wymaga napisanej przez człowieka odpowiedzi wstępnej — albo follow-upu proszącego o brakujące szczegóły (dla niekompletnych leadów), albo potwierdzenia odbioru (dla gorących leadów). Odpowiedzi te podążają za przewidywalnymi wzorcami. Pisanie ich od zera kosztuje 3-5 minut na lead, czyli 5-8 godzin tygodniowo dla całego zespołu.

4. **Ręczny handoff CRM→ZLECENIA.** Gdy lead konwertuje do potwierdzonego eventu, ktoś musi ręcznie stworzyć nowy task w ZLECENIA, skopiować wszystkie istotne dane z taska CRM, rozszerzyć konwencję nazewnictwa o pakiet i ustawić początkowy status ZLECENIA. Przy 20-30 wygranych eventach miesięcznie, to 2-5 godzin podatnej na błędy pracy kopiuj-wklej.

### Co działa dobrze i musi być zachowane

1. **Pipeline intake Phase 1** — stabilny, przetestowany, z graceful fallback'iem AI. Zmiany w istniejących scenariuszach muszą być addytywne (gałęzie router'a, nowe moduły dołączane na końcu), nigdy destruktywne.
2. **Konwencja nazewnictwa tasków** — `[DATA|BRAK DATY] | [MIASTO|WYBIERZ MIASTO] | [USŁUGA] | [IMIĘ]` — zachowana dokładnie dla tasków CRM.
3. **Custom ID LEAD-####** — stabilne, bez zmian.
4. **NOWE ZAPYTANIA jako jedyny zbiornik intake** — wszystkie leady wchodzą tutaj przed dalszym routingiem.
5. **Separacja pipeline'ów sprzedawców** — SPRZEDAŻ KUBA/WIKTOR/DAWID/KRIS.
6. **AI asystuje, człowiek decyduje** — brak automatycznej komunikacji z klientami, brak automatycznego routingu.
7. **Composer emaila w tasku** — ustalony workflow dla komunikacji wychodzącej.
8. **Separacja CRM ↔ ZLECENIA** — czysta granica między domenami sprzedaży i operacji.

### Dane zebrane vs. brakujące (po Phase 1)

**Aktualnie zbierane (16 Custom Fields):** MIASTO, PAKIET, POZYSKANIE, TELEFON, TYP IMPREZY, USŁUGA, GODZINA EVENTU, DATA EVENTU, EMAIL, ILOŚĆ OSÓB, JEDNOSTKA, ŹRÓDŁO_KANAŁ, KLASYFIKACJA_AI, AI_PODSUMOWANIE, KOMPLETNOŚĆ, AI_CONFIDENCE.

**Brakujące i potrzebne dla Phase 2:**

| Brak | Dlaczego to ważne |
|------|-------------------|
| Śledzenie częstotliwości kontaktu | Brak możliwości sprawdzenia, czy klient zgłaszał się wielokrotnie w sprawie tego samego eventu |
| Status „wygranej sprzedaży" w pipeline'ach sprzedawców | Brak spójnego punktu trigger'a dla automatyzacji CRM→ZLECENIA |
| Due Date na nowych taskach | Brak mechanizmu dla przypomnień opartych na czasie |

---

## Sekcja 2 — Proponowana architektura rozwiązania

### Przepływ end-to-end (ponumerowana sekwencja)

Phase 2 dodaje cztery możliwości do istniejącego pipeline'u Phase 1. Poniższe sekwencje pokazują, jak każda z nich się integruje.

**Możliwość 1 — Przypomnienia o follow-upach:**

1. Scenariusze Make Phase 1 (zarówno Form Intake jak i Email Intake) są aktualizowane: moduł ClickUp Create Task teraz ustawia Due Date = `teraz + 48 godzin` dla każdego taska niebędącego spamem
2. Nowa natywna automatyzacja ClickUp monitoruje nadejście Due Date: jeśli task nie ma Assignee gdy Due Date odpala się, Automatyzacja ustawia Priorytet na Pilny i dodaje komentarz eskalacyjny
3. Zespół sprzedaży widzi oznaczony task w widoku NOWE - PRIORYTET i podejmuje działania

**Możliwość 2 — Deduplicacja leadów:**

1. Nowy Data Store Make (`EVSO_Lead_Dedup`) przechowuje `{email, task_id, created_at}` dla każdego nowego leada
2. Scenariusze Make Phase 1 są modyfikowane: PRZED wywołaniem klasyfikacji AI, lookup Data Store sprawdza, czy dopasowanie emaila istnieje w ciągu ostatnich 7 dni
3. Jeśli BRAK dopasowania → proceed normalnie (wywołanie AI → tworzenie taska → zapis rekordu w Data Store)
4. Jeśli JEST DOPASOWANIE → pomiń wywołanie AI, pomiń tworzenie taska. Zamiast tego: dodaj komentarz do istniejącego taska z nowym zapytaniem, zaktualizuj Custom Fields o wcześniej brakujące dane, przelicz KOMPLETNOŚĆ, zwiększ LICZBA_KONTAKTÓW, dodaj tag WIELOKROTNY KONTAKT
5. Jeśli scalony task nie ma Assignee i LICZBA_KONTAKTÓW > 1, ustaw Priorytet na Pilny
6. Tygodniowy scenariusz czyszczący usuwa rekordy Data Store starsze niż 7 dni

**Możliwość 3 — Generowanie roboczych odpowiedzi:**

1. Po tworzeniu taska w obu scenariuszach Make, nowa gałąź router'a sprawdza KLASYFIKACJA_AI i AI_CONFIDENCE
2. Dla „Niekompletny lead" z confidence ≠ „Niska" → wywołaj Claude Haiku z promptem dla niekompletnego leada → opublikuj wersję roboczą jako komentarz ClickUp
3. Dla „Gorący lead" z confidence ≠ „Niska" → wywołaj Claude Haiku z promptem dla gorącego leada → opublikuj wersję roboczą jako komentarz ClickUp (WYŁĄCZONE przy starcie, włączane przez toggle po walidacji przez zespół)
4. Wersja robocza jest wyraźnie oznaczona jako wygenerowana przez AI. Handlowiec ją czyta, kopiuje przydatne fragmenty do composera emaila, edytuje i wysyła
5. Jeśli wywołanie AI się nie powiedzie → brak komentarza, brak wpływu na task (graceful degradation)

**Możliwość 4 — Automatyzacja obiegu ZLECENIA:**

1. Nowy status `SPRZEDANE` jest dodawany do każdej listy pipeline sprzedawcy (SPRZEDAŻ KUBA/WIKTOR/DAWID/KRIS)
2. Gdy handlowiec przenosi task do statusu SPRZEDANE (decyzja człowieka), webhook ClickUp wyzwala nowy scenariusz Make („EVSO - ZLECENIA Task Creator")
3. Scenariusz pobiera pełne szczegóły taska CRM, tworzy nowy task na liście ZLECENIA ze zmapowanymi polami i rozszerzoną konwencją nazewnictwa (`[DATA] | [MIASTO] | [USŁUGA] | [KLIENT] | [PAKIET]`), ustawia status ZLECENIA na „ZLECENIA BEZ ZALICZEK"
4. Scenariusz dodaje komentarze z wzajemnymi linkami do obu tasków (CRM → ZLECENIA, ZLECENIA → CRM)
5. Status taska CRM nie jest zmieniany poza SPRZEDANE — pozostaje w pipeline sprzedawcy dla śledzenia prowizji i widoczności sprzedaży

### Co się zmienia względem Phase 1

- **Scenariusz Phase 1 nr 1 (Form Intake Enhanced):** Kolejność modułów zmieniona — sprawdzenie dedup wstawione przed wywołaniem AI. Pole Due Date dodane do tworzenia taska. Gałąź router'a dodana po tworzeniu taska dla generowania roboczych odpowiedzi. Zapis Data Store dodany po tworzeniu taska.
- **Scenariusz Phase 1 nr 2 (Email Intake):** Te same modyfikacje co Scenariusz 1.
- **ClickUp NOWE ZAPYTANIA:** Jedno nowe Custom Field (LICZBA_KONTAKTÓW). Jedna nowa Automatyzacja (przypomnienie Due Date).
- **Pipeline'y sprzedawców ClickUp:** Nowy status SPRZEDANE dodany do każdej listy.
- **Make.com:** Jeden nowy Data Store (EVSO_Lead_Dedup). Dwa nowe scenariusze (ZLECENIA Task Creator, Dedup Cleanup).

### Co celowo pozostaje bez zmian

- NOWE ZAPYTANIA pozostaje jedyną listą intake
- Konwencja nazewnictwa tasków CRM jest zachowana dokładnie
- Schemat ID LEAD-#### jest nienaruszony
- Wszystkie prompty AI Phase 1 (klasyfikacja, ekstrakcja) są niezmienione
- Error handlery Phase 1 i zachowanie fallback'u są zachowane
- Ręczne przypisywanie przez sprzedawcę pozostaje (brak automatycznego routingu)
- Cała komunikacja z klientami pozostaje inicjowana przez człowieka

### Jawna granica zakresu

**Ten przewodnik obejmuje:**
- Dodatki do Custom Fields i statusów ClickUp
- Modyfikacje dwóch istniejących scenariuszy Make (Form Intake Enhanced, Email Intake)
- Dwa nowe scenariusze Make (ZLECENIA Task Creator, Dedup Cleanup)
- Jeden nowy Data Store Make
- Dwie nowe automatyzacje ClickUp
- Integrację AI dla generowania roboczych odpowiedzi (dwa prompty)
- Testowanie wszystkich czterech możliwości

**NIE obejmuje (wykluczone zgodnie z instrukcją użytkownika):**
- Wielojęzyczność
- Dashboardy analityczne
- Automatyczny routing leadów według miasta lub typu usługi

---

## Sekcja 3 — Przewodnik konfiguracji ClickUp

### 3.1 Listy / Foldery — bez nowych list

Wszystkie zmiany dotyczą istniejących list. Żadne nowe listy ani foldery nie są tworzone.

### 3.2 Custom Fields do dodania

Dodaj następujące Custom Field do `EVSO / CRM / NOWE ZAPYTANIA`:

**Pole 1: LICZBA_KONTAKTÓW**

| Właściwość | Wartość |
|------------|---------|
| Nazwa | `LICZBA_KONTAKTÓW` |
| Typ | Number (liczba całkowita, bez miejsc dziesiętnych) |
| Wartość domyślna | 1 |
| Lokalizacja | Lista NOWE ZAPYTANIA |
| Cel | Śledzi, ile razy klient skontaktował się w sprawie tego zapytania. Zwiększane przy każdym scaleniu dedup. Wartość > 1 sygnalizuje wielokrotny kontakt — pozytywny sygnał zakupowy wymagający szybkiej uwagi. |

**Uwaga implementacyjna:** Po stworzeniu pola, pobierz jego ClickUp field ID do użycia w modułach Make. Na istniejących taskach pole będzie pokazywać brak wartości (null) do czasu ustawienia. Nowe taski tworzone przez zmodyfikowane scenariusze Make ustawią wartość na `1` przy tworzeniu. Przy scaleniu dedup, wartość na istniejącym tasku jest zwiększana.

### 3.3 Statusy do dodania

Dodaj status `SPRZEDANE` do każdej z czterech list pipeline'ów sprzedawców:

| Lista | Status do dodania | Pozycja | Sugerowany kolor |
|-------|-------------------|---------|-----------------|
| SPRZEDAŻ KUBA | `SPRZEDANE` | Po ostatnim aktywnym statusie sprzedaży (po SZANSA SPRZEDAŻY lub ekwiwalencie) | Zielony |
| SPRZEDAŻ WIKTOR | `SPRZEDANE` | Ta sama pozycja | Zielony |
| SPRZEDAŻ DAWID | `SPRZEDANE` | Ta sama pozycja | Zielony |
| SPRZEDAŻ KRIS | `SPRZEDANE` | Ta sama pozycja | Zielony |

**Wymagana weryfikacja (U-002):** Przed dodaniem, sprawdź każdą listę pipeline'u sprzedawcy, żeby potwierdzić, że mają tę samą strukturę statusów. Obserwowane statusy na SPRZEDAŻ DAWID: NOWE ZAPYTANIE → WYSŁANO OFERTĘ → PRZEDZWONKA → ZASTANAWIA SIĘ → SZANSA SPRZEDAŻY. Zweryfikuj, czy pozostałe trzy listy pasują. Jeśli lista używa innych nazw lub kolejności, dostosuj odpowiednio — kluczowym wymogiem jest, żeby `SPRZEDANE` istniało jako status, do którego handlowiec może przeciągnąć task.

**Cel:** SPRZEDANE jest punktem trigger'a dla automatyzacji ZLECENIA. Jest to akcja człowieka — handlowiec decyduje „ta transakcja jest wygrana" i przenosi task do SPRZEDANE. Automatyzacja następnie tworzy task ZLECENIA.

### 3.4 Automatyzacje ClickUp — dodaj dwie

**Automatyzacja 1: Przypomnienie 48h dla nieprzypisanych tasków**

| Właściwość | Wartość |
|------------|---------|
| Nazwa | `Przypomnienie 48h — brak przypisania` |
| Lokalizacja | Lista NOWE ZAPYTANIA |
| Trigger | Gdy nadejdzie Due Date |
| Warunek | Assignee jest pusty |
| Akcja 1 | Ustaw Priorytet na `Pilny` |
| Akcja 2 | Opublikuj komentarz: `⚠️ To zapytanie czeka na obsługę od 48h. Proszę o przypisanie lub oznaczenie jako nieistotne.` |

**Dlaczego takie podejście:** Due Date jest ustawiany przez Make przy tworzeniu taska (patrz Sekcja 4). Gdy nadchodzi i task wciąż nie ma Assignee, ta Automatyzacja eskaluje widoczność. Używanie Due Date zamiast trigger'ów opartych na czasie spędzonym w statusie jest celowe — Due Date jest widoczny w widokach kalendarza i listy, dając zespołowi termin bez otwierania taska, i odpala się niezależnie od zmian statusu.

**Wymagana weryfikacja (U-004):** Trigger „Gdy nadejdzie Due Date" ClickUp jest dostępny na planach Business i wyższych. Zweryfikuj plan ClickUp EVSO przed konfiguracją. Jeśli plan nie obsługuje tego trigger'a, fallback to scenariusz Make z harmonogramem, który uruchamia się co godzinę i wyszukuje przeterminowane nieprzypisane taski (bardziej złożony, mniej niezawodny, ale niezależny od planu).

**Automatyzacja 2: Eskalacja leadów z wielokrotnym kontaktem (wyzwalana scaleniem dedup)**

| Właściwość | Wartość |
|------------|---------|
| Nazwa | `Eskalacja — wielokrotny kontakt` |
| Lokalizacja | Lista NOWE ZAPYTANIA |
| Trigger | Gdy Custom Field `LICZBA_KONTAKTÓW` się zmienia |
| Warunek | LICZBA_KONTAKTÓW > 1 ORAZ Assignee jest pusty |
| Akcja | Ustaw Priorytet na `Pilny` |

**Cel:** Gdy klient kontaktuje się z EVSO wielokrotnie (scalenie dedup zwiększa licznik), a task nadal nie ma Assignee, jest to silny sygnał zakupowy, który jest ignorowany. Automatyzacja oznacza go do natychmiastowej uwagi.

**Uwaga:** Jeśli warunki Automatyzacji ClickUp nie obsługują natywnie „pole > 1", użyj „pole nie jest puste ORAZ pole nie równa się 1" lub obsłuż tę logikę w scenariuszu Make zamiast tego (ustaw Priorytet na Pilny w gałęzi scalenia dedup, gdy Assignee jest pusty).

### 3.5 Widoki — bez nowych widoków

Istniejące widoki NOWE - PRIORYTET i NOWE - KOMPLETNOŚĆ z Phase 1 już ujawniają informacje potrzebne dla funkcji Phase 2. Pole Priorytet (używane przez przypomnienia o follow-upach i eskalację dedup) jest już widoczne w tych widokach.

**Opcjonalne ulepszenie:** Dodaj `LICZBA_KONTAKTÓW` jako widoczną kolumnę w obu widokach, żeby zespół mógł od razu zobaczyć, które leady mają wiele kontaktów.

---

## Sekcja 4 — Przewodnik scenariuszy Make.com

Phase 2 modyfikuje dwa istniejące scenariusze Phase 1 i dodaje dwa nowe. Modyfikacje są addytywne — nowe moduły są wstawiane lub dołączane, istniejące moduły nie są usuwane.

**Krytyczny warunek wstępny:** Przed modyfikacją jakiegokolwiek scenariusza Phase 1, utwórz kopię zapasową przez sklonowanie scenariusza w Make.com. Nazwij klony „EVSO - Form Intake Enhanced (Phase 1 Backup)" i „EVSO - Email Intake (Phase 1 Backup)". Wyłącz klony. Zapewnia to możliwość rollback'u, jeśli modyfikacje Phase 2 wprowadzą regresje.

### Konfiguracja Data Store (przed modyfikacją scenariuszy)

Utwórz nowy Data Store w Make.com:

```
Data Store: EVSO_Lead_Dedup
Cel: Tabela lookup dla deduplicacji leadów — przechowuje mapowanie email↔task dla ostatnich leadów
Struktura:
  Pole klucza: email (text, klucz główny)
  Dodatkowe pola:
    - task_id (text) — ID taska ClickUp oryginalnego taska
    - task_name (text) — nazwa taska ClickUp (dla odniesienia w komentarzach scalenia)
    - created_at (date) — timestamp kiedy rekord został stworzony
    - phone (text, opcjonalnie) — przechowywany dla potencjalnego przyszłego dopasowania telefonicznego
Maksymalna liczba rekordów: 500 (przy ~100 leadach/tydzień i 7-dniowej retencji, max ~100 aktywnych rekordów — 500 daje odpowiedni bufor)
```

**Kroki konfiguracji:**
1. W Make.com, przejdź do Data Stores
2. Kliknij „Add data store"
3. Nazwa: `EVSO_Lead_Dedup`
4. Dodaj strukturę danych z powyższymi polami
5. Ustaw `email` jako klucz
6. Zapisz

Utwórz też zmienną scenariusza dla okna dedup:

```
Zmienna: dedup_window_days
Wartość: 7
Cel: Konfigurowalne okno dopasowania dedup. Zacznij od 7, dostosuj na podstawie rzeczywistych danych po 30 dniach działania.
```

I zmienną scenariusza dla toggle'a roboczych odpowiedzi:

```
Zmienna: hot_lead_draft_enabled
Wartość: false
Cel: Kontroluje, czy robocze odpowiedzi dla gorących leadów są generowane. Zacznij wyłączone. Włącz po 1-2 tygodniach feedbacku zespołu na temat roboczych odpowiedzi dla niekompletnych leadów.
```

---

### Scenariusz 1: Form Intake Enhanced — Modyfikacje Phase 2

Istniejący scenariusz Phase 1 ma 7 modułów w tej kolejności:

```
Kolejność Phase 1:
  Moduł 1: Webhook (trigger)
  Moduł 2: Set Variables — Normalizacja miasta
  Moduł 3: Set Variables — Wynik kompletności
  Moduł 4: HTTP — Klasyfikacja AI
  Moduł 5: JSON Parse — Odpowiedź AI
  Moduł 6: Set Variables — Mapowanie AI na wartości ClickUp
  Moduł 7: ClickUp — Create Task
```

Phase 2 wstawia nowe moduły i przestawia kolejność. Nowy przepływ:

```
Kolejność Phase 2:
  Moduł 1:  Webhook (trigger) — BEZ ZMIAN
  Moduł 2:  Set Variables — Normalizacja miasta — BEZ ZMIAN
  Moduł 3:  Set Variables — Wynik kompletności — BEZ ZMIAN
  Moduł 4:  Data Store — Search Records (sprawdzenie dedup) — NOWY
  Moduł 5:  Router — NOWY
    Gałąź A (wykryto duplikat):
      Moduł 5A-1: ClickUp — Get Task (pobierz istniejący task)
      Moduł 5A-2: Set Variables — Zbuduj dane scalenia
      Moduł 5A-3: ClickUp — Update Task (zaktualizuj pola, zwiększ LICZBA_KONTAKTÓW)
      Moduł 5A-4: ClickUp — Create Comment (dołącz nowe zapytanie)
      Moduł 5A-5: Data Store — Update Record (odśwież timestamp)
      → Scenariusz kończy się (brak wywołania AI, brak nowego taska, brak wersji roboczej)
    Gałąź B (brak duplikatu — nowy lead):
      Moduł 5B-1: HTTP — Klasyfikacja AI — (PRZENIESIONY z Modułu 4 Phase 1)
      Moduł 5B-2: JSON Parse — Odpowiedź AI — (PRZENIESIONY z Modułu 5 Phase 1)
      Moduł 5B-3: Set Variables — Mapowanie AI na ClickUp — (PRZENIESIONY z Modułu 6 Phase 1)
      Moduł 5B-4: ClickUp — Create Task (z Due Date) — (ZMODYFIKOWANY z Modułu 7 Phase 1)
      Moduł 5B-5: Data Store — Add Record — NOWY
      Moduł 5B-6: Set Variables — Przygotuj kontekst wersji roboczej — NOWY
      Moduł 5B-7: Router — Robocza odpowiedź — NOWY
        Gałąź B1 (Niekompletny lead):
          Moduł 5B-7a: HTTP — Robocza odpowiedź AI (prompt dla niekompletnego leada)
          Moduł 5B-7b: JSON Parse — Odpowiedź z wersją roboczą
          Moduł 5B-7c: ClickUp — Create Comment (robocza odpowiedź)
        Gałąź B2 (Gorący lead — WYŁĄCZONE przy starcie):
          Moduł 5B-7d: HTTP — Robocza odpowiedź AI (prompt dla gorącego leada)
          Moduł 5B-7e: JSON Parse — Odpowiedź z wersją roboczą
          Moduł 5B-7f: ClickUp — Create Comment (robocza odpowiedź)
        Gałąź B3 (wszystkie inne): Brak akcji
```

#### Moduł 4 — Data Store: Search Records (Sprawdzenie dedup)

```
Typ: Data Store > Search Records
Cel: Sprawdź, czy lead z tym emailem już istnieje w oknie dedup
Ustawienia:
  Data store: EVSO_Lead_Dedup
  Filtr:
    Warunek 1: email = {{form_fields.email}}
    Warunek 2: created_at > {{addDays(now; -dedup_window_days)}}
  Limit: 1
Zmienne wyjściowe:
  - dedup_match (boolean — true jeśli zwrócono jakiekolwiek rekordy)
  - existing_task_id (text — task_id z dopasowanego rekordu, lub pusty)
  - existing_task_name (text — task_name z dopasowanego rekordu, lub pusty)
```

**Error handler:**

```
Error handler: Resume
  Przy błędzie (awaria lookup Data Store):
    Ustaw dedup_match = false
    Kontynuuj do Gałęzi B (traktuj jako nowy lead)
  Uzasadnienie: Awaria Data Store powinna degradować do „brak dedup" zamiast „brak intake".
  Zapewnia to, że intake nigdy nie jest blokowany przez system dedup.
```

#### Moduł 5 — Router

```
Typ: Router
Cel: Rozdziel przepływ na podstawie wyniku dedup
Ustawienia:
  Gałąź A:
    Etykieta: "Duplikat — scalenie"
    Warunek: dedup_match = true
  Gałąź B:
    Etykieta: "Nowy lead"
    Warunek: dedup_match = false (fallback / domyślny)
```

#### Gałąź A — Wykryto duplikat (Przepływ scalenia)

**Moduł 5A-1 — ClickUp: Get a Task**

```
Typ: ClickUp > Get a Task
Cel: Pobierz aktualne dane istniejącego taska do scalenia
Ustawienia:
  Task ID: {{existing_task_id}}
Zmienne wyjściowe:
  - existing_task (pełny obiekt taska włącznie z Custom Fields, Assignees)
```

**Moduł 5A-2 — Tools: Set Multiple Variables (Zbuduj dane scalenia)**

```
Typ: Tools > Set Multiple Variables
Cel: Przygotuj dane do aktualizacji scalenia
Ustawienia:
  Zmienna 1:
    Nazwa: new_contact_count
    Wartość: {{if(existing_task.custom_fields.LICZBA_KONTAKTÓW != null; existing_task.custom_fields.LICZBA_KONTAKTÓW + 1; 2)}}

  Zmienna 2:
    Nazwa: merge_comment_body
    Wartość: |
      --- DODATKOWY KONTAKT (Formularz) --- {{formatDate(now; "DD-MM-YYYY HH:mm")}} ---
      Miasto: {{form_fields.miasto}}
      Rodzaj imprezy: {{form_fields.rodzaj_imprezy}}
      Pakiet: {{form_fields.pakiet}}
      Data eventu: {{form_fields.data}}
      Godzina: {{form_fields.godzina}}
      Liczba osób: {{form_fields.liczba_osob}}
      Opis: {{form_fields.opis}}
      ---

  Zmienna 3:
    Nazwa: updated_completeness
    Wartość: Przelicz na podstawie scalonych danych — użyj tej samej formuły co Moduł 3,
      ale zastosuj wyższą wartość między istniejącą a nową dla każdego pola:
      {{round(
        (
          (if(existing_task.custom_fields.MIASTO != "WYBIERZ MIASTO" OR city_normalized != "WYBIERZ MIASTO"; 1; 0)) +
          (if(existing_task.custom_fields.EMAIL != "" OR form_fields.email != ""; 1; 0)) +
          (if(existing_task.custom_fields.TELEFON != "" OR form_fields.telefon != ""; 1; 0)) +
          (if(existing_task.custom_fields.DATA_EVENTU != "" OR form_fields.data != ""; 1; 0)) +
          (if(existing_task.custom_fields.GODZINA_EVENTU != "" OR form_fields.godzina != ""; 1; 0)) +
          (if(existing_task.custom_fields.ILOŚĆ_OSÓB != "" OR form_fields.liczba_osob != ""; 1; 0)) +
          (if(existing_task.custom_fields.TYP_IMPREZY != "" OR form_fields.rodzaj_imprezy != ""; 1; 0)) +
          (if(form_fields.imie_nazwisko != ""; 1; 0))
        ) / 8 * 100
      )}}
```

**Uwaga dotycząca dostępu do Custom Fields:** ClickUp API zwraca Custom Fields jako tablicę obiektów, nie jako płaską mapę. Dokładna ścieżka zależy od struktury wyjściowej modułu ClickUp. Będziesz musiał użyć funkcji Array lub JSON parse do wyodrębnienia konkretnych wartości pól. Przetestuj z prawdziwym taskiem, żeby potwierdzić wzorzec dostępu.

**Moduł 5A-3 — ClickUp: Update a Task**

```
Typ: ClickUp > Update a Task
Cel: Zaktualizuj istniejący task scalonymi danymi
Ustawienia:
  Task ID: {{existing_task_id}}
  Custom Fields:
    - LICZBA_KONTAKTÓW: {{new_contact_count}}
    - KOMPLETNOŚĆ: {{updated_completeness}}
    - MIASTO: {{if(existing_task.custom_fields.MIASTO = "WYBIERZ MIASTO" AND city_normalized != "WYBIERZ MIASTO"; city_normalized; existing_task.custom_fields.MIASTO)}}
    - DATA EVENTU: {{if(existing_task.custom_fields.DATA_EVENTU = "" AND form_fields.data != ""; form_fields.data; existing_task.custom_fields.DATA_EVENTU)}}
    - GODZINA EVENTU: {{if(existing_task.custom_fields.GODZINA_EVENTU = "" AND form_fields.godzina != ""; form_fields.godzina; existing_task.custom_fields.GODZINA_EVENTU)}}
    - ILOŚĆ OSÓB: {{if(existing_task.custom_fields.ILOŚĆ_OSÓB = "" AND form_fields.liczba_osob != ""; form_fields.liczba_osob; existing_task.custom_fields.ILOŚĆ_OSÓB)}}
    - TYP IMPREZY: {{if(existing_task.custom_fields.TYP_IMPREZY = "" AND form_fields.rodzaj_imprezy != ""; form_fields.rodzaj_imprezy; existing_task.custom_fields.TYP_IMPREZY)}}
    - TELEFON: {{if(existing_task.custom_fields.TELEFON = "" AND form_fields.telefon != ""; form_fields.telefon; existing_task.custom_fields.TELEFON)}}
    - PAKIET: {{if(existing_task.custom_fields.PAKIET = "" AND form_fields.pakiet != ""; form_fields.pakiet; existing_task.custom_fields.PAKIET)}}
  Tagi: Dodaj "WIELOKROTNY KONTAKT"
  Priorytet: {{if(new_contact_count > 1 AND length(existing_task.assignees) = 0; "Urgent"; existing_task.priority)}}
```

**Logika:** Nadpisuj Custom Field tylko jeśli istniejąca wartość jest pusta/domyślna ORAZ nowe zgłoszenie dostarcza niepustą wartość. Nigdy nie nadpisuj istniejących danych nowymi danymi — pierwsza wartość jest prawdopodobnie dokładniejsza (pochodzi z bardziej strukturalnego źródła lub wcześniejszego etapu rozmowy).

**Moduł 5A-4 — ClickUp: Create a Comment**

```
Typ: ClickUp > Create a Comment
Cel: Dołącz treść nowego zapytania jako komentarz do istniejącego taska
Ustawienia:
  Task ID: {{existing_task_id}}
  Treść komentarza: {{merge_comment_body}}
```

**Moduł 5A-5 — Data Store: Update a Record**

```
Typ: Data Store > Update a Record
Cel: Odśwież timestamp, żeby okno dedup rozciągało się od ostatniego kontaktu
Ustawienia:
  Data store: EVSO_Lead_Dedup
  Klucz: {{form_fields.email}}
  Pola:
    created_at: {{now}}
```

**Scenariusz kończy się tutaj dla Gałęzi A.** Brak wywołania AI, brak tworzenia nowego taska, brak generowania wersji roboczej. Duplikat jest scalany cicho do istniejącego taska.

---

#### Gałąź B — Nowy lead (Zmodyfikowany przepływ Phase 1 + Generowanie wersji roboczej)

Moduły 5B-1 do 5B-3 to oryginalne Moduły 4, 5 i 6 Phase 1 (Klasyfikacja AI, JSON Parse, Mapowanie wartości) — **przeniesione tutaj ze swojej poprzedniej pozycji, ale poza tym niezmienione.** Patrz Przewodnik Phase 1 Sekcja 4, Moduły 4-6 dla konfiguracji.

**Moduł 5B-4 — ClickUp: Create a Task (Zmodyfikowany)**

To jest Moduł 7 Phase 1 z dwoma uzupełnieniami:

```
Uzupełnienia do istniejącego modułu ClickUp Create Task:

  1. Due Date:
    Wartość: {{addHours(now; 48)}}
    Warunek: Ustaw tylko jeśli clickup_klasyfikacja != "Spam"
    Implementacja: Opakuj w IF:
      {{if(clickup_klasyfikacja != "Spam"; addHours(now; 48); "")}}

  2. Custom Field — LICZBA_KONTAKTÓW:
    Wartość: 1

Wszystkie inne pola pozostają dokładnie tak, jak udokumentowano w Przewodniku Phase 1,
Sekcja 4, Scenariusz 1, Moduł 7.

Wyjście: Zapisz id nowo stworzonego taska do użycia w kolejnych modułach.
```

**Moduł 5B-5 — Data Store: Add a Record**

```
Typ: Data Store > Add a Record
Cel: Zapisz ten lead w tabeli lookup dedup
Ustawienia:
  Data store: EVSO_Lead_Dedup
  Klucz: {{form_fields.email}}
  Pola:
    email: {{form_fields.email}}
    task_id: {{created_task_id}}
    task_name: {{task_title}}
    created_at: {{now}}
    phone: {{form_fields.telefon}}
  Nadpisz: Tak (jeśli ten sam email istnieje, nadpisz — obsługuje przypadki brzegowe, gdy rekord nie został wyczyszczony)
```

**Error handler:**

```
Error handler: Resume
  Przy błędzie: Kontynuuj (awaria zapisu Data Store nie powinna blokować tworzenia taska.
  Task już istnieje w ClickUp — dedup po prostu nie zadziała dla tego leada.)
```

**Moduł 5B-6 — Tools: Set Multiple Variables (Przygotuj kontekst wersji roboczej)**

```
Typ: Tools > Set Multiple Variables
Cel: Przygotuj zmienne kontekstowe dla promptu generowania roboczej odpowiedzi
Ustawienia:
  Zmienna 1:
    Nazwa: missing_fields_list
    Wartość: Zbuduj listę brakujących pól oddzieloną przecinkami:
      {{join(
        filter([
          if(city_normalized = "WYBIERZ MIASTO"; "miasto"; null);
          if(form_fields.data = ""; "data eventu"; null);
          if(form_fields.godzina = ""; "godzina"; null);
          if(form_fields.liczba_osob = ""; "liczba osób"; null);
          if(form_fields.rodzaj_imprezy = ""; "rodzaj imprezy"; null);
          if(form_fields.pakiet = ""; "pakiet"; null)
        ]; item != null)
      ; ", ")}}

  Zmienna 2:
    Nazwa: customer_first_name
    Wartość: Wyodrębnij imię z pełnego imienia i nazwiska:
      {{first(split(form_fields.imie_nazwisko; " "))}}

  Zmienna 3:
    Nazwa: service_name_polish
    Wartość:
      {{if(service_type = "TRAM"; "tramwaj imprezowy";
        if(service_type = "BOAT"; "statek imprezowy";
        if(service_type = "BUS"; "bus imprezowy";
        "event")))}}
```

**Uwaga:** Make.com może nie obsługiwać `filter()` z dokładnie powyższą składnią. Dostosuj używając serii instrukcji IF do budowania ciągu brakujących pól. Kluczowy wymóg: wyprodukuj czytelną po polsku listę braków, żeby prompt AI mógł się do niej odwołać.

**Moduł 5B-7 — Router (Robocza odpowiedź)**

```
Typ: Router
Cel: Przekieruj do odpowiedniego promptu generowania wersji roboczej na podstawie klasyfikacji
Ustawienia:
  Gałąź B1:
    Etykieta: "Wersja robocza — niekompletny lead"
    Warunek: clickup_klasyfikacja = "Niekompletny lead" AND clickup_confidence != "Niska"
  Gałąź B2:
    Etykieta: "Wersja robocza — gorący lead"
    Warunek: clickup_klasyfikacja = "Gorący lead" AND clickup_confidence != "Niska" AND hot_lead_draft_enabled = true
  Gałąź B3:
    Etykieta: "Brak wersji roboczej"
    Warunek: (fallback — wszystkie inne przypadki)
```

**Moduł 5B-7a — HTTP: Make a Request (Wersja robocza dla niekompletnego leada)**

```
Typ: HTTP > Make a Request
Cel: Wygeneruj roboczą odpowiedź follow-up dla niekompletnych leadów
Ustawienia:
  URL: https://api.anthropic.com/v1/messages
  Metoda: POST
  Nagłówki:
    - x-api-key: {{anthropic_api_key}}
    - anthropic-version: 2023-06-01
    - content-type: application/json
  Typ treści: Raw (JSON)
```

```json
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 400,
  "messages": [
    {
      "role": "user",
      "content": "Napisz wersję roboczą odpowiedzi na zapytanie eventowe.\n\nKontekst:\n- Klient: {{form_fields.imie_nazwisko}}\n- Usługa: {{service_name_polish}}\n- Co wiemy: {{ai_podsumowanie}}\n- Brakujące dane: {{missing_fields_list}}\n- Źródło: formularz na {{source_brand}}\n\nZwróć TYLKO tekst odpowiedzi (bez JSON, bez markdown). Odpowiedź powinna:\n1. Podziękować za zapytanie i potwierdzić co zrozumieliśmy\n2. Grzecznie zapytać o brakujące informacje: {{missing_fields_list}}\n3. Zakończyć się zachętą do kontaktu\n\nTon: semi-formalny, ciepły, entuzjastyczny ale profesjonalny. Pisz po polsku. Max 150 słów."
    }
  ],
  "system": "Jesteś asystentem handlowca firmy EVSO, która organizuje eventy na tramwajach, statkach i busach imprezowych w Polsce. Piszesz WERSJE ROBOCZE odpowiedzi do klientów — handlowiec przejrzy i dostosuje tekst przed wysłaniem.\n\nŻelazne zasady:\n- NIGDY nie podawaj cen, rabatów ani warunków finansowych\n- NIGDY nie obiecuj konkretnych dat dostępności ani pojemności\n- NIGDY nie składaj zobowiązań w imieniu firmy\n- NIGDY nie używaj sformułowań typu 'na pewno', 'gwarantujemy', 'bez problemu'\n- Pisz naturalnie po polsku, unikaj korporacyjnego żargonu\n- Używaj imienia klienta (forma 'Panie/Pani + imię' lub sam imię jeśli ton jest luźniejszy)\n\n{{brand_voice_examples}}"
}
```

**Uwaga dotycząca `{{brand_voice_examples}}`:** To placeholder dla prawdziwych przykładów odpowiedzi zebranych od zespołu sprzedaży. Zacznij od pustego ciągu. Gdy zostanie zebranych 5-10 prawdziwych przykładów (warunek wstępny U-003), dodaj je do system promptu jako: `\n\nPrzykłady rzeczywistych odpowiedzi zespołu (naśladuj ten ton):\n---\nPrzykład 1: [tekst prawdziwej odpowiedzi]\n---\nPrzykład 2: [tekst prawdziwej odpowiedzi]\n---`

```
Error handler: Resume
  Przy błędzie: Pomiń tworzenie komentarza. Zaloguj błąd. Kontynuuj scenariusz.
  Task już istnieje — brakująca wersja robocza nie jest awarią.
```

**Moduł 5B-7b — Wyodrębnij tekst odpowiedzi z wersją roboczą**

AI zwraca zwykły tekst (nie JSON) dla roboczych odpowiedzi. Opakuj to w krok wyodrębniania tekstu:

```
Typ: Tools > Set Variable
Cel: Oczyść tekst roboczej odpowiedzi
Ustawienia:
  Zmienna: draft_reply_text
  Wartość: {{5B-7a.response_body.content[].text}}
  Uwaga: Jeśli odpowiedź zawiera znaczniki markdown lub dodatkowe formatowanie, usuń je.
```

**Moduł 5B-7c — ClickUp: Create a Comment (Robocza odpowiedź)**

```
Typ: ClickUp > Create a Comment
Cel: Opublikuj roboczą odpowiedź jako komentarz na nowo stworzonym tasku
Ustawienia:
  Task ID: {{created_task_id}}
  Treść komentarza: |
    🤖 WERSJA ROBOCZA ODPOWIEDZI (wygenerowana przez AI — do edycji przed wysłaniem)
    ---
    {{draft_reply_text}}
    ---
    ⚠️ Przejrzyj i dostosuj przed wysłaniem. Nie zawiera cen ani zobowiązań.
```

---

**Moduł 5B-7d — HTTP: Make a Request (Wersja robocza dla gorącego leada) — WYŁĄCZONE PRZY STARCIE**

```
Typ: HTTP > Make a Request
Cel: Wygeneruj robocze potwierdzenie odbioru dla gorących leadów
Ustawienia:
  URL: https://api.anthropic.com/v1/messages
  Metoda: POST
  Nagłówki: (te same co 5B-7a)
  Typ treści: Raw (JSON)
```

```json
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 400,
  "messages": [
    {
      "role": "user",
      "content": "Napisz wersję roboczą odpowiedzi potwierdzającej otrzymanie zapytania eventowego.\n\nKontekst:\n- Klient: {{form_fields.imie_nazwisko}}\n- Usługa: {{service_name_polish}}\n- Miasto: {{city_normalized}}\n- Data eventu: {{form_fields.data}}\n- Typ imprezy: {{form_fields.rodzaj_imprezy}}\n- Pakiet: {{form_fields.pakiet}}\n- Liczba osób: {{form_fields.liczba_osob}}\n- Co wiemy (podsumowanie): {{ai_podsumowanie}}\n- Źródło: formularz na {{source_brand}}\n\nZwróć TYLKO tekst odpowiedzi (bez JSON, bez markdown). Odpowiedź powinna:\n1. Podziękować za zapytanie\n2. Potwierdzić kluczowe szczegóły (data, miasto, typ, liczba osób)\n3. Poinformować o następnych krokach (np. 'przygotujemy ofertę' lub 'odezwiemy się wkrótce')\n4. Zakończyć pozytywnie\n\nTon: semi-formalny, ciepły, entuzjastyczny. Pisz po polsku. Max 150 słów."
    }
  ],
  "system": "Jesteś asystentem handlowca firmy EVSO, która organizuje eventy na tramwajach, statkach i busach imprezowych w Polsce. Piszesz WERSJE ROBOCZE odpowiedzi do klientów — handlowiec przejrzy i dostosuje tekst przed wysłaniem.\n\nŻelazne zasady:\n- NIGDY nie podawaj cen, rabatów ani warunków finansowych\n- NIGDY nie obiecuj konkretnych dat dostępności ani pojemności\n- NIGDY nie składaj zobowiązań w imieniu firmy (np. 'na pewno się uda', 'gwarantujemy')\n- Możesz powiedzieć 'przygotujemy ofertę' lub 'odezwiemy się' — to są bezpieczne następne kroki\n- Pisz naturalnie po polsku, unikaj korporacyjnego żargonu\n- Potwierdź szczegóły które klient podał, ale nie dodawaj niczego od siebie\n\n{{brand_voice_examples}}"
}
```

Moduły 5B-7e i 5B-7f postępują według tego samego wzorca co 5B-7b i 5B-7c (oczyść odpowiedź, opublikuj jako komentarz ClickUp z tym samym znacznikiem formatu).

```
Error handler: Resume (tak samo jak Gałąź B1 — pomiń komentarz przy awarii)
```

---

### Scenariusz 2: Email Intake — Modyfikacje Phase 2

Modyfikacje Scenariusza 2 (Email Intake) odzwierciedlają Scenariusz 1 dokładnie pod względem struktury. Różnice dotyczą tylko źródeł danych (pola emaila vs. pola formularza):

**Zmiany do zastosowania:**

1. **Wstaw lookup Data Store** (tak samo jak Scenariusz 1, Moduł 4) — ale użyj `{{from_email}}` zamiast `{{form_fields.email}}` jako klucza dedup
2. **Wstaw Router** z Gałęzią A (duplikat) i Gałęzią B (nowy lead) — ta sama logika
3. **Komentarz scalenia Gałęzi A** używa formatu specyficznego dla emaila:
   ```
   --- DODATKOWY KONTAKT (Email) --- {{formatDate(now; "DD-MM-YYYY HH:mm")}} ---
   Od: {{from_name}} <{{from_email}}>
   Temat: {{subject}}
   Treść: {{text_body}}
   ---
   ```
4. **Tworzenie taska Gałęzi B** dodaje Due Date i LICZBA_KONTAKTÓW (tak samo jak Scenariusz 1)
5. **Zapis Data Store Gałęzi B** używa `{{from_email}}` jako klucza
6. **Generowanie roboczej odpowiedzi Gałęzi B** używa kontekstu specyficznego dla emaila w prompcie:
   - Zastąp `{{form_fields.imie_nazwisko}}` przez `{{final_name}}`
   - Zastąp `{{source_brand}}` przez `{{email_brand}}`
   - Zastąp odwołania `{{form_fields.*}}` przez wyekstrahowane przez AI wartości `{{parsed.*}}`
   - missing_fields_list jest budowana z pól parsed (sprawdzenia null zamiast sprawdzeń pustego ciągu)

**Skrót implementacyjny:** Sklonuj nowe moduły zmodyfikowanego Scenariusza 1 (lookup Data Store, Router, przepływ scalenia Gałęzi A, gałęzie wersji roboczej Gałęzi B) i dostosuj ścieżki odwołań do pól. Logika jest identyczna — zmieniają się tylko ścieżki do danych źródłowych.

---

### Scenariusz 3: EVSO - ZLECENIA Task Creator (NOWY)

```
Scenariusz: EVSO - ZLECENIA Task Creator
Cel: Automatycznie tworzy task w ZLECENIA gdy lead CRM jest oznaczony jako SPRZEDANE
Trigger: Webhook ClickUp — zmiana statusu taska
Projekt: B (tworzenie nowego taska) — zgodnie z Decyzją D-006
```

**Wstępna konfiguracja:**
1. Utwórz nowy scenariusz Make.com
2. Skonfiguruj webhook ClickUp w Make, który odpala się na zmiany statusu taska
3. Skonfiguruj webhook do nasłuchiwania folderu CRM (wszystkie listy pipeline'ów sprzedawców)

#### Moduł 1 — Webhooks: Custom Webhook (Zdarzenie ClickUp)

```
Typ: Webhooks > Custom Webhook
Cel: Odbieraj powiadomienia webhook'a ClickUp o zmianach statusu
Ustawienia:
  Nazwa webhook'a: "EVSO ZLECENIA Trigger"
  Uwaga: Zarejestruj ten URL webhook'a w ClickUp (Ustawienia przestrzeni > Integracje > Webhooks)
        lub użyj natywnego modułu trigger'a ClickUp w Make jeśli dostępny:
        ClickUp > Watch Events > Task Status Updated
  Filtr: Przetwarzaj tylko zdarzenia gdzie new_status = "SPRZEDANE"
Zmienne wyjściowe:
  - task_id (task CRM który zmienił status)
  - new_status (powinno być "SPRZEDANE")
  - list_id (która lista pipeline'u sprzedawcy)
```

**Alternatywne podejście:** Jeśli natywny moduł ClickUp Make obsługuje „Watch Tasks" z filtrem statusu, użyj go zamiast custom webhook'a. Przetestuj oba podejścia — natywny moduł jest prostszy w konfiguracji.

#### Moduł 2 — Filter: Weryfikacja statusu

```
Typ: Filter
Cel: Proceed tylko jeśli status to SPRZEDANE (sprawdzenie bezpieczeństwa)
Warunek: new_status = "SPRZEDANE"
Uwaga: Wychwytuje przypadki brzegowe, gdzie webhook wysyła inne zmiany statusu.
```

#### Moduł 3 — ClickUp: Get a Task

```
Typ: ClickUp > Get a Task
Cel: Pobierz pełny task CRM ze wszystkimi Custom Fields
Ustawienia:
  Task ID: {{task_id}}
Zmienne wyjściowe:
  - crm_task (pełny obiekt taska)
  - Wszystkie Custom Fields: MIASTO, USŁUGA, PAKIET, DATA EVENTU, EMAIL, TELEFON,
    ILOŚĆ OSÓB, GODZINA EVENTU, TYP IMPREZY, AI_PODSUMOWANIE, itd.
  - Nazwa taska, opis, assignees
```

#### Moduł 4 — Tools: Set Multiple Variables (Zbuduj dane ZLECENIA)

```
Typ: Tools > Set Multiple Variables
Cel: Przygotuj wszystkie wartości dla taska ZLECENIA
Ustawienia:
  Zmienna 1:
    Nazwa: zlecenie_title
    Wartość: Zbuduj rozszerzoną konwencję nazewnictwa z PAKIETEM:
      {{crm_task.custom_fields.DATA_EVENTU}} | {{crm_task.custom_fields.MIASTO}} | {{crm_task.custom_fields.USŁUGA}} | {{crm_task.name_extracted_client}} | {{crm_task.custom_fields.PAKIET}}
    Uwaga: Wyodrębnij imię klienta z nazwy taska CRM (4. segment po podziale przez " | ")
      {{last(split(crm_task.name; " | "))}} może działać jeśli imię klienta jest zawsze ostatnie.
      Lub wyodrębnij z Custom Fields jeśli dostępne.

  Zmienna 2:
    Nazwa: zlecenie_description
    Wartość: |
      --- DANE ZE ZLECENIA (automatycznie z CRM) ---
      Lead: {{crm_task.custom_id}} ({{crm_task.name}})
      Miasto: {{crm_task.custom_fields.MIASTO}}
      Usługa: {{crm_task.custom_fields.USŁUGA}}
      Pakiet: {{crm_task.custom_fields.PAKIET}}
      Data eventu: {{crm_task.custom_fields.DATA_EVENTU}}
      Godzina: {{crm_task.custom_fields.GODZINA_EVENTU}}
      Liczba osób: {{crm_task.custom_fields.ILOŚĆ_OSÓB}}
      Typ imprezy: {{crm_task.custom_fields.TYP_IMPREZY}}
      Email klienta: {{crm_task.custom_fields.EMAIL}}
      Telefon: {{crm_task.custom_fields.TELEFON}}

      AI Podsumowanie: {{crm_task.custom_fields.AI_PODSUMOWANIE}}

      Opis oryginalnego zapytania:
      {{crm_task.description}}
      ---

  Zmienna 3:
    Nazwa: crm_task_url
    Wartość: https://app.clickup.com/t/{{crm_task.id}}
```

**Uwaga dotycząca wyodrębniania imienia klienta:** Format nazwy taska CRM to `[DATA] | [MIASTO] | [USŁUGA] | [IMIĘ]`. Imię klienta to 4. segment. Użyj `split(crm_task.name; " | ")` i weź indeks [3] (od 0). Jeśli funkcja split Make zwraca tablicę, użyj `get()` lub dostępu tablicowego. Przetestuj z prawdziwą nazwą taska, żeby potwierdzić.

#### Moduł 5 — ClickUp: Create a Task (ZLECENIA)

```
Typ: ClickUp > Create a Task
Ustawienia:
  Workspace: EVSO
  Przestrzeń: EVSO
  Folder: ZLECENIA
  Lista: ZLECENIA (lub konkretna lista ZLECENIA — zweryfikuj dokładną nazwę listy)
  Nazwa taska: {{zlecenie_title}}
  Opis: {{zlecenie_description}}
  Status: ZLECENIA BEZ ZALICZEK
  Due Date: {{crm_task.custom_fields.DATA_EVENTU}} (jeśli dostępne — to jest data eventu, która staje się operacyjnym terminem)
  Custom Fields:
    - MIASTO: {{crm_task.custom_fields.MIASTO}}
    - USŁUGA: {{crm_task.custom_fields.USŁUGA}}
    - PAKIET: {{crm_task.custom_fields.PAKIET}}
    - DATA EVENTU: {{crm_task.custom_fields.DATA_EVENTU}}
    - GODZINA EVENTU: {{crm_task.custom_fields.GODZINA_EVENTU}}
    - EMAIL: {{crm_task.custom_fields.EMAIL}}
    - TELEFON: {{crm_task.custom_fields.TELEFON}}
    - ILOŚĆ OSÓB: {{crm_task.custom_fields.ILOŚĆ_OSÓB}}
Wyjście:
  - zlecenie_task_id (ID nowego taska ZLECENIA)
```

**Wymagana weryfikacja (U-005):** Custom Fields na liście ZLECENIA mogą nie mieć tych samych field ID co w folderze CRM. Sprawdź czy ZLECENIA ma własne definicje Custom Fields. Jeśli nazwy pól pasują, ale ID różnią, zmapuj je ręcznie w konfiguracji modułu. Jeśli ZLECENIA ma dodatkowe pola (np. OBSŁUGA, JEDNOSTKA), pozostaw je puste — zespół operacyjny wypełnia je ręcznie.

#### Moduł 6 — ClickUp: Create a Comment (na tasku ZLECENIA)

```
Typ: ClickUp > Create a Comment
Cel: Linkuj task ZLECENIA z powrotem do taska CRM
Ustawienia:
  Task ID: {{zlecenie_task_id}}
  Treść komentarza: |
    📋 Zlecenie utworzone automatycznie z CRM.
    Lead źródłowy: {{crm_task.custom_id}} — {{crm_task.name}}
    Link do CRM: {{crm_task_url}}
```

#### Moduł 7 — ClickUp: Create a Comment (na tasku CRM)

```
Typ: ClickUp > Create a Comment
Cel: Linkuj task CRM do nowego taska ZLECENIA
Ustawienia:
  Task ID: {{task_id}} (oryginalny task CRM)
  Treść komentarza: |
    ✅ Zlecenie utworzone automatycznie.
    Link do zlecenia: https://app.clickup.com/t/{{zlecenie_task_id}}
    Status: ZLECENIA BEZ ZALICZEK
```

```
Obsługa błędów dla całego scenariusza:
  Przy każdym nieodwracalnym błędzie:
  1. Wyślij powiadomienie emailowe do kontaktu inżynierskiego/ops z detalami błędu
  2. Task CRM pozostaje ze statusem SPRZEDANE — handlowiec może stworzyć task ZLECENIA ręcznie
  3. Zaloguj: task_id, komunikat błędu, timestamp
```

---

### Scenariusz 4: EVSO - Dedup Cleanup (NOWY)

```
Scenariusz: EVSO - Dedup Cleanup
Cel: Tygodniowe czyszczenie wygasłych rekordów w Data Store EVSO_Lead_Dedup
Trigger: Harmonogram — uruchamia się raz w tygodniu (np. niedziela 03:00)
```

#### Moduł 1 — Trigger harmonogramu

```
Typ: Schedule
Ustawienia:
  Uruchom scenariusz: Raz w tygodniu
  Dzień: Niedziela
  Godzina: 03:00 (okres małego ruchu)
```

#### Moduł 2 — Data Store: Search Records

```
Typ: Data Store > Search Records
Cel: Znajdź wszystkie rekordy starsze niż okno dedup
Ustawienia:
  Data store: EVSO_Lead_Dedup
  Filtr: created_at < {{addDays(now; -dedup_window_days)}}
  Limit: 100 (przetwarzaj partiami)
Wyjście: Tablica rekordów do usunięcia
```

#### Moduł 3 — Iterator

```
Typ: Flow Control > Iterator
Cel: Przetwarzaj każdy wygasły rekord indywidualnie
Ustawienia:
  Tablica: {{rekordy wyjściowe Modułu 2}}
```

#### Moduł 4 — Data Store: Delete a Record

```
Typ: Data Store > Delete a Record
Cel: Usuń wygasły rekord
Ustawienia:
  Data store: EVSO_Lead_Dedup
  Klucz: {{iterator.email}}
```

```
Error handler: Resume
  Przy błędzie indywidualnego usunięcia: Pomiń i kontynuuj do następnego rekordu.
  Loguj błędy, ale nie zatrzymuj czyszczenia.
```

**Uwaga:** Jeśli Data Store ma więcej niż 100 wygasłych rekordów (mało prawdopodobne przy wolumenie EVSO), scenariusz będzie musiał uruchomić pętlę szukaj-iteruj-usuwaj wielokrotnie. Przy ~100 leadach/tydzień i 7-dniowej retencji, spodziewaj się maksymalnie ~100 aktywnych rekordów. Czyszczenie to siatka bezpieczeństwa, nie operacja wysokiego wolumenu.

---

## Sekcja 5 — Specyfikacja integracji AI

### Wybór modelu

**Model:** `claude-haiku-4-5-20251001` przez Anthropic API — tak samo jak Phase 1.

**Uzasadnienie dla roboczych odpowiedzi:** Generowanie roboczych odpowiedzi to prosta generacja tekstu prowadzona przez szablon. Prompty są ograniczone (max 150 słów, ścisłe granice dotyczące treści), a output jest recenzowany przez człowieka przed wysłaniem. Haiku wystarczy. Większy model produkowałby marginalnie bardziej dopracowany tekst przy 10-20-krotnie wyższym koszcie, bez mierzalnej poprawy wyniku (handlowiec i tak edytuje wersję roboczą).

**Szacowany dodatkowy koszt API:** Phase 2 dodaje ~1 wywołanie AI na niespamlowany, niezduplikowany lead. Przy ~80 kwalifikujących się leadach/tydzień (100 łącznie minus spam i duplikaty), z ~500 tokenami wejściowymi i ~300 tokenami wyjściowymi: ~1-2 USD/miesiąc dodatkowo. Całkowity system (Phase 1 + Phase 2): ~3-5 USD/miesiąc.

### Zadanie AI 3: Robocza odpowiedź dla niekompletnego leada

**Nazwa zadania:** Robocza odpowiedź follow-up dla niekompletnych leadów

**Kiedy używane:** Scenariusz 1 i 2 (oba intake), Gałąź B1 Router'a Roboczych odpowiedzi

**System prompt (gotowy do kopiowania):**

```
Jesteś asystentem handlowca firmy EVSO, która organizuje eventy na tramwajach, statkach i busach imprezowych w Polsce. Piszesz WERSJE ROBOCZE odpowiedzi do klientów — handlowiec przejrzy i dostosuje tekst przed wysłaniem.

Żelazne zasady:
- NIGDY nie podawaj cen, rabatów ani warunków finansowych
- NIGDY nie obiecuj konkretnych dat dostępności ani pojemności
- NIGDY nie składaj zobowiązań w imieniu firmy
- NIGDY nie używaj sformułowań typu 'na pewno', 'gwarantujemy', 'bez problemu'
- Pisz naturalnie po polsku, unikaj korporacyjnego żargonu
- Używaj imienia klienta (forma 'Panie/Pani + imię' lub sam imię jeśli ton jest luźniejszy)
```

**Szablon wiadomości użytkownika:**

```
Napisz wersję roboczą odpowiedzi na zapytanie eventowe.

Kontekst:
- Klient: {{customer_name}}
- Usługa: {{service_name_polish}}
- Co wiemy: {{ai_podsumowanie}}
- Brakujące dane: {{missing_fields_list}}
- Źródło: {{source_channel}}

Zwróć TYLKO tekst odpowiedzi (bez JSON, bez markdown). Odpowiedź powinna:
1. Podziękować za zapytanie i potwierdzić co zrozumieliśmy
2. Grzecznie zapytać o brakujące informacje: {{missing_fields_list}}
3. Zakończyć się zachętą do kontaktu

Ton: semi-formalny, ciepły, entuzjastyczny ale profesjonalny. Pisz po polsku. Max 150 słów.
```

**Przykładowy output:**

```
Cześć Marek,

Dziękujemy za zapytanie dotyczące statku imprezowego! Widzimy, że planujesz wieczór kawalerski — brzmi świetnie!

Żeby przygotować dla Was najlepszą ofertę, potrzebujemy jeszcze kilku informacji:
- W jakim mieście planujecie event?
- Na kiedy planujecie imprezę (data)?
- Ile osób będzie w grupie?

Jak tylko dostaniemy te dane, od razu zajmiemy się Waszym zapytaniem.

Pozdrawiamy,
Zespół EVSO
```

### Zadanie AI 4: Robocze potwierdzenie odbioru dla gorącego leada

**Nazwa zadania:** Robocze potwierdzenie odbioru dla gorących leadów

**Kiedy używane:** Scenariusz 1 i 2, Gałąź B2 Router'a Roboczych odpowiedzi (WYŁĄCZONE przy starcie)

**System prompt:** Taki sam jak Zadanie AI 3.

**Szablon wiadomości użytkownika:**

```
Napisz wersję roboczą odpowiedzi potwierdzającej otrzymanie zapytania eventowego.

Kontekst:
- Klient: {{customer_name}}
- Usługa: {{service_name_polish}}
- Miasto: {{city}}
- Data eventu: {{event_date}}
- Typ imprezy: {{event_type}}
- Pakiet: {{package}}
- Liczba osób: {{people_count}}
- Co wiemy (podsumowanie): {{ai_podsumowanie}}
- Źródło: {{source_channel}}

Zwróć TYLKO tekst odpowiedzi (bez JSON, bez markdown). Odpowiedź powinna:
1. Podziękować za zapytanie
2. Potwierdzić kluczowe szczegóły (data, miasto, typ, liczba osób)
3. Poinformować o następnych krokach (np. 'przygotujemy ofertę' lub 'odezwiemy się wkrótce')
4. Zakończyć pozytywnie

Ton: semi-formalny, ciepły, entuzjastyczny. Pisz po polsku. Max 150 słów.
```

**Przykładowy output:**

```
Cześć Katarzyna,

Dziękujemy za zapytanie! Z przyjemnością zajmiemy się organizacją Twojego wieczoru panieńskiego.

Potwierdzamy szczegóły:
- Statek imprezowy we Wrocławiu
- Data: 20 czerwca 2026
- Grupa: 12 osób
- Pakiet: LUX

Przygotujemy spersonalizowaną ofertę i odezwiemy się najszybciej jak to możliwe.

Pozdrawiamy serdecznie,
Zespół EVSO
```

### Jawne granice AI (uzupełnienia Phase 2)

Oprócz granic AI Phase 1 (brak cen, brak routingu, brak komunikacji z klientami, brak decyzji kwalifikacyjnych, brak wymyślania danych), Phase 2 dodaje:

5. **Robocze odpowiedzi NIE SĄ NIGDY wysyłane automatycznie.** AI generuje tekst pojawiający się jako komentarz ClickUp. Człowiek musi skopiować go do composera emaila, edytować i wysłać ręcznie. Nie ma automatycznego wysyłania.
6. **Wersje robocze nie mogą zawierać cen.** Nawet jeśli system będzie miał dane cenowe w przyszłości, robocze odpowiedzi nie mogą ich zawierać. Wycena to decyzja biznesowa podejmowana per zapytanie.
7. **Wersje robocze nie mogą składać obietnic.** „Przygotujemy ofertę" jest akceptowalne (to stwierdzenie procesowe). „Możemy pomieścić Waszą grupę" nie jest (to zobowiązanie dotyczące pojemności).
8. **Brak generowania wersji roboczej dla klasyfikacji o niskiej pewności.** Jeśli AI_CONFIDENCE wynosi „Niska", sama klasyfikacja jest niepewna — wersja robocza oparta na niepewnej klasyfikacji mogłaby być myląca.
9. **Brak generowania wersji roboczej dla duplikatów.** Jeśli lead jest identyfikowany jako duplikat i scalany, żadna wersja robocza nie jest generowana. Oryginalny task może już mieć wersję roboczą lub być w aktywnej obsłudze.

### Fallback przy awarii (Phase 2)

Dla generowania roboczych odpowiedzi: Jeśli wywołanie Anthropic API się nie powiedzie, żaden komentarz nie jest publikowany. Task istnieje bez wersji roboczej. Handlowiec pisze ręcznie. Jest to identyczne z workflow sprzed Phase 2 — brak regresji.

Dla lookup Data Store dedup: Jeśli lookup się nie powiedzie, scenariusz proceeds tak, jakby nie było duplikatu. Tworzenie taska kontynuuje normalnie. Dedup jest najlepsiłowy, nie krytyczny.

Dla tworzenia taska ZLECENIA: Jeśli scenariusz się nie powiedzie, task CRM pozostaje ze statusem SPRZEDANE. Handlowiec tworzy task ZLECENIA ręcznie, jak robi to dzisiaj. Powiadomienie o błędzie jest wysyłane do inżynierię.

---

## Sekcja 6 — Sekwencja wdrożenia

### Dzień przed wdrożeniem: Śledztwo ZLECENIA (Decyzja D-007)

```
Dzień 0: Śledztwo procesu ZLECENIA
  Zadania:
    1. Zapytaj zespół sprzedaży: „Kiedy wygrywasz transakcję, co robisz?
       Przenosisz task czy tworzysz nowy w ZLECENIA?" — zaobserwuj lub zapisz obecny proces
    2. Sprawdź: Czy taski ZLECENIA mają ID LEAD-####? (Jeśli tak → prawdopodobnie przeniesienie taska / Projekt A)
    3. Sprawdź: Czy zamknięte taski CRM nadal istnieją w widokach pipeline'u sprzedawców?
       (Jeśli tak → prawdopodobnie nowy task / Projekt B)
    4. Sprawdź: Jakie dane są dodawane podczas przejścia CRM→ZLECENIA?
       (OBSŁUGA, JEDNOSTKA itd.)
    5. Decyzja: Potwierdź Projekt B (tworzenie nowego taska) lub przełącz na Projekt A (przeniesienie taska)
    6. Jeśli Projekt B potwierdzony: Przejdź z Scenariuszem 3 jak udokumentowano
       Jeśli Projekt A: Dostosuj Scenariusz 3 — użyj natywnej automatyzacji ClickUp do przeniesienia taska,
       scenariusz Make obsługuje tylko wzbogacenie po przeniesieniu (patrz output spotkania M-002 dla specyfikacji Projektu A)
    7. Zbierz 5-10 prawdziwych przykładów odpowiedzi od zespołu sprzedaży
       (warunek wstępny U-003 dla generowania wersji roboczych)
  Warunek wstępny: Brak — można uruchomić przed lub równolegle z pracą techniczną Phase 2
  Wynik: Potwierdzona decyzja projektowa ZLECENIA, zebrane przykłady odpowiedzi
```

### Tydzień 1: Deduplicacja + Przypomnienia o follow-upach

```
Dzień 1: Konfiguracja Data Store + zmiany ClickUp
  Zadania:
    1. Utwórz Make Data Store "EVSO_Lead_Dedup" (patrz Sekcja 4, Konfiguracja Data Store)
    2. Dodaj zmienne scenariusza: dedup_window_days = 7, hot_lead_draft_enabled = false
    3. Dodaj Custom Field LICZBA_KONTAKTÓW do NOWE ZAPYTANIA (Sekcja 3.2)
    4. Dodaj status SPRZEDANE do wszystkich 4 list pipeline'ów sprzedawców (Sekcja 3.3)
       — najpierw zweryfikuj strukturę statusów (U-002)
    5. Utwórz automatyzację ClickUp „Przypomnienie 48h" (Sekcja 3.4, Automatyzacja 1)
    6. Utwórz automatyzację ClickUp „Eskalacja — wielokrotny kontakt" (Sekcja 3.4, Automatyzacja 2)
    7. Zweryfikuj, czy plan ClickUp obsługuje trigger'y automatyzacji opartych na czasie (U-004)
    8. Dodaj kolumnę LICZBA_KONTAKTÓW do istniejących widoków (opcjonalnie)
  Warunek wstępny: Phase 1 działający i stabilny
  Wynik: Cała konfiguracja ClickUp + Data Store kompletna

Dzień 2: Modyfikacja scenariusza Form Intake — dedup + Due Date
  Zadania:
    1. Sklonuj „EVSO - Form Intake Enhanced" jako kopię zapasową (wyłącz klon)
    2. Wstaw Moduł 4: Data Store Search (sprawdzenie dedup) — po kompletności, przed AI
    3. Wstaw Moduł 5: Router (dedup vs. nowy)
    4. Zbuduj Gałąź A: przepływ scalenia duplikatu (Moduły 5A-1 do 5A-5)
    5. Przenieś Moduły 4-7 Phase 1 do Gałęzi B
    6. Dodaj Due Date do modułu tworzenia taska Gałęzi B
    7. Dodaj LICZBA_KONTAKTÓW = 1 do modułu tworzenia taska Gałęzi B
    8. Dodaj zapis Data Store (Moduł 5B-5) po tworzeniu taska w Gałęzi B
    9. Dodaj error handlery dla modułów Data Store
    10. NIE dodawaj jeszcze modułów roboczych odpowiedzi (Dzień 4)
  Warunek wstępny: Dzień 1
  Wynik: Scenariusz Form Intake zmodyfikowany z dedup + Due Date

Dzień 3: Modyfikacja scenariusza Email Intake — dedup + Due Date
  Zadania:
    1. Sklonuj „EVSO - Email Intake" jako kopię zapasową (wyłącz klon)
    2. Zastosuj te same modyfikacje co Dzień 2, dostosowane do odwołań pól emaila
    3. Użyj from_email jako klucza dedup zamiast form_fields.email
    4. Komentarz scalenia używa formatu specyficznego dla emaila
    5. Dodaj error handlery
  Warunek wstępny: Dzień 1
  Wynik: Scenariusz Email Intake zmodyfikowany z dedup + Due Date

Dzień 4: Testowanie dedup + przypomnień
  Zadania:
    1. Ponownie uruchom wszystkie smoke testy Phase 1 (Testy 1-7 z Przewodnika Phase 1)
       na zmodyfikowanych scenariuszach — potwierdź brak regresji w podstawowym przepływie intake
    2. Testuj dedup: wyślij ten sam adres email dwukrotnie przez formularz w ciągu 5 minut
       — zweryfikuj, że drugie zgłoszenie scala się do pierwszego taska
       (komentarz dodany, LICZBA_KONTAKTÓW = 2, tag dodany)
    3. Testuj wzbogacenie scalenia dedup: pierwsze zgłoszenie bez daty, drugie z datą
       — zweryfikuj, że pole DATA EVENTU jest aktualizowane na istniejącym tasku
    4. Testuj dedup między kanałami: wyślij formularz, potem email z tym samym adresem email
       — zweryfikuj, że scalenie działa między kanałami
    5. Testuj okno dedup: ręcznie ustaw created_at rekordu Data Store na 8 dni temu,
       wyślij z tym emailem — zweryfikuj, że tworzony jest nowy task (brak dopasowania poza oknem)
    6. Testuj Due Date: utwórz task, poczekaj 48h (lub ręcznie ustaw Due Date na teraz-1h dla testów),
       zweryfikuj że automatyzacja ClickUp odpala się (Priorytet → Pilny, komentarz opublikowany)
    7. Testuj błąd Data Store: tymczasowo zmień nazwę Data Store, wyślij formularz
       — zweryfikuj, że task jest tworzony normalnie (dedup degraduje, intake kontynuuje)
    8. Napraw wszelkie problemy
  Warunek wstępny: Dni 2-3
  Wynik: Funkcje dedup i przypomnień zwalidowane
```

### Tydzień 2: Robocze odpowiedzi + ZLECENIA

```
Dzień 5: Dodaj generowanie roboczych odpowiedzi do scenariusza Form Intake
  Zadania:
    1. Dodaj Moduł 5B-6: Przygotuj kontekst wersji roboczej
       (brakujące pola, imię klienta, nazwa usługi)
    2. Dodaj Moduł 5B-7: Router dla roboczych odpowiedzi
    3. Zbuduj Gałąź B1: Wersja robocza dla niekompletnego leada
       (wywołanie HTTP + parsowanie + komentarz ClickUp)
    4. Zbuduj Gałąź B2: Wersja robocza dla gorącego leada
       (wywołanie HTTP + parsowanie + komentarz ClickUp) — zbuduj ale WYŁĄCZ
    5. Zbuduj Gałąź B3: Brak akcji (fallback)
    6. Dodaj error handlery dla wywołań API wersji roboczych
    7. Jeśli przykłady odpowiedzi są dostępne (z Dnia 0), zintegruj z system promptem
       Jeśli niedostępne, użyj generycznego promptu (oznacz do późniejszej aktualizacji)
  Warunek wstępny: Dzień 4 (dedup przetestowany), Dzień 0 (przykłady jeśli dostępne)
  Wynik: Generowanie roboczych odpowiedzi dodane do Form Intake

Dzień 6: Dodaj generowanie roboczych odpowiedzi do scenariusza Email Intake
  Zadania:
    1. Zastosuj te same moduły wersji roboczych co Dzień 5,
       dostosowane do odwołań pól emaila
    2. Testuj niezależnie
  Warunek wstępny: Dzień 5
  Wynik: Generowanie roboczych odpowiedzi dodane do Email Intake

Dzień 7: Testowanie roboczych odpowiedzi
  Zadania:
    1. Wyślij formularz z niekompletnymi danymi (brakuje daty, miasta)
       — zweryfikuj, że komentarz z wersją roboczą pojawia się z szablonem niekompletnego leada
    2. Wyślij formularz z kompletnymi danymi (gorący lead)
       — zweryfikuj, że komentarz z wersją roboczą NIE pojawia się (gałąź gorącego leada wyłączona)
    3. Tymczasowo włącz hot_lead_draft_enabled, wyślij kompletny formularz
       — zweryfikuj, że komentarz z wersją roboczą pojawia się z szablonem potwierdzenia
       — ponownie wyłącz hot_lead_draft_enabled
    4. Zweryfikuj format komentarza z wersją roboczą (znacznik 🤖, separatory ---, stopka ⚠️)
    5. Wyślij formularz sklasyfikowany jako Spam — zweryfikuj, że nie wygenerowano wersji roboczej
    6. Wyślij formularz z AI_CONFIDENCE = Niska — zweryfikuj, że nie wygenerowano wersji roboczej
    7. Testuj awarię AI wersji roboczej: tymczasowo ustaw zły klucz API tylko dla wywołania wersji roboczej
       — zweryfikuj, że task jest tworzony normalnie, tylko bez komentarza z wersją roboczą
    8. Zweryfikuj interakcję dedup + wersja robocza: wyślij duplikat
       — zweryfikuj, że dla scalenia nie wygenerowano wersji roboczej
    9. Przeczytaj 5 wygenerowanych wersji roboczych z zespołem sprzedaży
       — zbierz wstępny feedback na temat tonu i użyteczności
  Warunek wstępny: Dni 5-6
  Wynik: Generowanie roboczych odpowiedzi zwalidowane

Dzień 8: Buduj scenariusz ZLECENIA Task Creator
  Zadania:
    1. Utwórz nowy scenariusz Make „EVSO - ZLECENIA Task Creator"
    2. Skonfiguruj trigger: webhook ClickUp lub moduł Watch dla zmian statusu
    3. Dodaj filtr: tylko status SPRZEDANE
    4. Dodaj Moduł 3: Get Task (pobierz dane taska CRM)
    5. Dodaj Moduł 4: Budowanie zmiennych (tytuł ZLECENIA, opis, linki)
    6. Dodaj Moduł 5: Create Task na liście ZLECENIA
    7. Dodaj Moduł 6: Komentarz na tasku ZLECENIA (cross-link do CRM)
    8. Dodaj Moduł 7: Komentarz na tasku CRM (cross-link do ZLECENIA)
    9. Dodaj error handler z powiadomieniem emailowym
    10. Przetestuj mapowanie Custom Fields między CRM a ZLECENIA (U-005)
  Warunek wstępny: Dzień 0 (śledztwo potwierdza projekt), Dzień 1 (dodano status SPRZEDANE)
  Wynik: Scenariusz ZLECENIA zbudowany

Dzień 9: Testowanie automatyzacji ZLECENIA
  Zadania:
    1. Utwórz testowy task w pipeline'u sprzedawcy z wypełnionymi Custom Fields
    2. Przenieś task do statusu SPRZEDANE
    3. Zweryfikuj: nowy task pojawia się w ZLECENIA z poprawnym tytułem
       (rozszerzone nazewnictwo z PAKIETEM)
    4. Zweryfikuj: wszystkie zmapowane pola są poprawnie wypełnione
    5. Zweryfikuj: komentarze z wzajemnymi linkami istnieją na obu taskach
    6. Zweryfikuj: task CRM pozostaje w pipeline'u sprzedawcy ze statusem SPRZEDANE
    7. Zweryfikuj: status taska ZLECENIA to ZLECENIA BEZ ZALICZEK
    8. Testuj z brakującymi polami (np. brak PAKIETU, brak GODZINY)
       — zweryfikuj graceful handling
    9. Testuj przypadek błędu: tymczasowo zepsuj scenariusz, przenieś task do SPRZEDANE
       — zweryfikuj, że wysłano powiadomienie o błędzie, task CRM nienaruszony
    10. Napraw wszelkie problemy
  Warunek wstępny: Dzień 8
  Wynik: Automatyzacja ZLECENIA zwalidowana
```

### Tydzień 3: Czyszczenie, stabilizacja, przekazanie

```
Dzień 10: Buduj scenariusz Dedup Cleanup + pełny test integracyjny
  Zadania:
    1. Utwórz nowy scenariusz Make „EVSO - Dedup Cleanup" (Sekcja 4, Scenariusz 4)
    2. Zaplanuj na niedzielę 03:00
    3. Testuj: ręcznie wstaw 5 rekordów w EVSO_Lead_Dedup z created_at = 10 dni temu
       — uruchom scenariusz czyszczący ręcznie — zweryfikuj usunięcie rekordów
    4. Pełny test integracyjny: symuluj cały cykl życia leada:
       a. Wyślij formularz (task stworzony, rekord dedup zapisany,
          komentarz z wersją roboczą opublikowany, Due Date ustawiony)
       b. Wyślij ten sam email przez kanał email
          (scalenie dedup, komentarz dołączony, LICZBA_KONTAKTÓW = 2)
       c. Przypisz task do sprzedawcy
          (przypomnienie o follow-upie nie odpali się — task ma Assignee)
       d. Przenieś task przez pipeline sprzedawcy do SPRZEDANE
          — zweryfikuj automatyczne stworzenie taska ZLECENIA
       e. Poczekaj na cykl czyszczenia — zweryfikuj wyczyszczenie rekordu dedup
    5. Uruchom pełną listę smoke testów Phase 1, żeby potwierdzić brak regresji
  Warunek wstępny: Wszystkie scenariusze zbudowane
  Wynik: Wszystkie 4 funkcje działają end-to-end

Dzień 11: Prezentacja dla zespołu sprzedaży + feedback
  Zadania:
    1. Zaprezentuj nowe funkcje zespołowi sprzedaży:
       - Przypomnienia Due Date i komentarz eskalacyjny ⚠️
       - Scalenia dedup: jak działają tag WIELOKROTNY KONTAKT i LICZBA_KONTAKTÓW
       - Komentarze z roboczymi odpowiedziami: jak je czytać, kopiować do composera emaila,
         edytować i wysyłać
       - Automatyczne tworzenie ZLECENIA: co się dzieje, gdy oznaczą transakcję jako SPRZEDANE
    2. Niech każdy handlowiec przetestuje workflow z roboczą odpowiedzią:
       przeczyta wersję roboczą, skopiuje do emaila, edytuje, wyśle
    3. Zbierz feedback na temat jakości i tonu roboczych odpowiedzi
    4. Jeśli feedback jest pozytywny: rozważ włączenie hot_lead_draft_enabled
    5. Zaktualizuj WIKI PRACOWNIKA o dokumentację wszystkich 4 nowych funkcji
  Warunek wstępny: Dzień 10
  Wynik: Zespół przeszkolony, feedback zebrany

Dzień 12: Poprawki + dokumentacja
  Zadania:
    1. Zaimplementuj pozycje feedbacku o najwyższym priorytecie z Dnia 11
    2. Jeśli zebrano prawdziwe przykłady odpowiedzi: zaktualizuj system prompty
       roboczych odpowiedzi o przykłady głosu marki
    3. Włącz hot_lead_draft_enabled jeśli zespół zatwierdził (lub zaplanuj na kolejny tydzień)
    4. Sprawdź skuteczność dedup: sprawdź Data Store pod kątem nieoczekiwanych wzorców
    5. Udokumentuj wszystkie 4 scenariusze Make: zrób screenshoty konfiguracji modułów,
       zapisz w folderze współdzielonym
    6. Udokumentuj procedury ręcznego fallback'u:
       - „Jeśli dedup się nie powiedzie: leady tworzą osobne taski (tak samo jak zachowanie Phase 1)"
       - „Jeśli generowanie wersji roboczej się nie powiedzie: napisz odpowiedź ręcznie
          (tak samo jak obecne zachowanie)"
       - „Jeśli automatyzacja ZLECENIA się nie powiedzie: utwórz task ZLECENIA ręcznie"
    7. Zaktualizuj smoke testy Phase 1 o uzupełnienia Phase 2 (patrz Sekcja 7)
    8. Napisz krótką notatkę o planowaniu Phase 3 na podstawie pozostałych odłożonych funkcji
  Warunek wstępny: Dzień 11
  Wynik: System udokumentowany, poprawki zastosowane, Phase 2 kompletna
```

**Szacowany całkowity nakład inżynierski: 12 dni roboczych + 1 dzień śledztwa (Dzień 0) = 13 dni roboczych.** W ramach ograniczenia 15 dni z buforem 2 dni.

---

## Sekcja 7 — Lista kontrolna smoke testów

### Testy regresji Phase 1 (uruchom najpierw)

```
Retesty 1-7: Uruchom wszystkie 7 smoke testów Phase 1 z Przewodnika Wdrożenia Phase 1
  Cel: Potwierdzenie, że modyfikacje scenariuszy Phase 2 nie zepsuły istniejącej funkcjonalności intake
  Warunek zaliczenia: Wszystkie 7 testów produkuje te same wyniki co Phase 1
  (z uzupełnieniem Due Date i LICZBA_KONTAKTÓW = 1 na nowych taskach)
```

### Testy Phase 2

```
Test 8: Dedup — Ten sam email przez formularz (w oknie)
  Akcja:
    Krok 1: Wyślij formularz na partyboat.fun z emailem jan@test.pl,
            miasto: w Krakowie, data: 2026-06-15
    Krok 2: Poczekaj 2 minuty
    Krok 3: Wyślij formularz na partytram.fun z emailem jan@test.pl,
            miasto: (puste), data: 2026-06-15
  Oczekiwany wynik:
    - Tylko JEDEN task istnieje w NOWE ZAPYTANIA (z Kroku 1)
    - Task ma komentarz: "--- DODATKOWY KONTAKT (Formularz) ---" z danymi Kroku 3
    - LICZBA_KONTAKTÓW = 2
    - Tag: WIELOKROTNY KONTAKT
    - MIASTO nadal = KRAKÓW (nie nadpisane przez puste)
    - Żaden drugi task nie jest tworzony
  Warunek zaliczenia: Scalenie udane, wzbogacenie danych działa, brak duplikatu taska.
```

```
Test 9: Dedup — Ten sam email między kanałami (formularz, potem email)
  Akcja:
    Krok 1: Wyślij formularz na partyboat.fun z emailem anna@test.pl,
            brakuje daty i miasta
    Krok 2: Wyślij email z anna@test.pl na kontakt@partyboat.fun
            z datą i miastem zawartymi w treści
  Oczekiwany wynik:
    - Tylko JEDEN task istnieje
    - Task ma komentarz ze scalenia emaila z nowymi danymi
    - Pola DATA EVENTU i MIASTO zaktualizowane ze zgłoszenia emailowego
    - KOMPLETNOŚĆ zwiększona
    - LICZBA_KONTAKTÓW = 2
  Warunek zaliczenia: Dedup między kanałami działa, wzbogacenie pól działa.
```

```
Test 10: Dedup — Poza oknem (brak scalenia)
  Akcja:
    Krok 1: Ręcznie wstaw rekord w EVSO_Lead_Dedup z emailem stary@test.pl
            i created_at = 8 dni temu
    Krok 2: Wyślij formularz z emailem stary@test.pl
  Oczekiwany wynik:
    - Nowy task stworzony (brak scalenia)
    - LICZBA_KONTAKTÓW = 1
    - Nowy rekord Data Store stworzony (nadpisując stary)
  Warunek zaliczenia: Okno dedup poprawnie wyklucza stare rekordy.
```

```
Test 11: Dedup — Graceful degradation przy awarii Data Store
  Akcja:
    Krok 1: Tymczasowo zmień nazwę Data Store EVSO_Lead_Dedup
            (lub zepsuj odwołanie do modułu)
    Krok 2: Wyślij formularz z kompletnymi danymi
  Oczekiwany wynik:
    - Task stworzony normalnie w NOWE ZAPYTANIA ze wszystkimi polami
    - Klasyfikacja AI działa
    - Wersja robocza wygenerowana (jeśli ma zastosowanie)
    - Tylko dedup jest pomijany — wszystko inne działa
  Warunek zaliczenia: Awaria Data Store nie blokuje intake.
  Po teście: Przywróć nazwę/odwołanie Data Store.
```

```
Test 12: Przypomnienie o follow-upie — Trigger Due Date
  Akcja:
    Krok 1: Wyślij formularz (niespamowy). Zweryfikuj, że task stworzony z Due Date = teraz + 48h.
    Krok 2: Do testów: ręcznie ustaw Due Date taska na 1 godzinę temu w ClickUp.
    Krok 3: Zweryfikuj, że automatyzacja ClickUp odpala się (może zająć kilka minut).
  Oczekiwany wynik:
    - Priorytet ustawiony na Pilny
    - Komentarz opublikowany: "⚠️ To zapytanie czeka na obsługę od 48h..."
  Warunek zaliczenia: Automatyzacja odpala się przy nadejściu Due Date dla nieprzypisanych tasków.
  Uwaga: Taski spamowe NIE powinny mieć ustawionego Due Date.
          Zweryfikuj, wysyłając zapytanie sklasyfikowane jako spam.
```

```
Test 13: Przypomnienie o follow-upie — Przypisany task (brak trigger'a)
  Akcja:
    Krok 1: Wyślij formularz, zweryfikuj że task stworzony z Due Date
    Krok 2: Przypisz task do dowolnego użytkownika
    Krok 3: Ręcznie ustaw Due Date na przeszłość
  Oczekiwany wynik:
    - Automatyzacja NIE odpala się (task ma Assignee)
    - Priorytet pozostaje niezmieniony
  Warunek zaliczenia: Przypomnienie odpala się tylko dla nieprzypisanych tasków.
```

```
Test 14: Robocza odpowiedź — Niekompletny lead
  Akcja: Wyślij formularz na partyboat.fun z:
    - Imię i Nazwisko: Tomek Testowy
    - Email: tomek@test.pl
    - Typ imprezy: Wieczór kawalerski
    - Data: (puste)
    - Miasto: (puste)
    - Liczba osób: (puste)
    - Opis: "Chcemy zorganizować kawalerskie na statku"
  Oczekiwany wynik:
    - Task stworzony z KLASYFIKACJA_AI = Niekompletny lead
    - Komentarz pojawia się na tasku z formatem:
      🤖 WERSJA ROBOCZA ODPOWIEDZI (wygenerowana przez AI — do edycji przed wysłaniem)
      ---
      [Tekst po polsku proszący o miasto, datę eventu, liczbę osób]
      ---
      ⚠️ Przejrzyj i dostosuj przed wysłaniem. Nie zawiera cen ani zobowiązań.
    - Tekst wersji roboczej jest po polsku
    - Tekst wersji roboczej NIE zawiera żadnych cen ani zobowiązań
    - Tekst wersji roboczej wspomina konkretne brakujące pola
  Warunek zaliczenia: Wersja robocza wygenerowana, poprawnie sformatowana, bezpieczna treściowo.
```

```
Test 15: Robocza odpowiedź — Gorący lead (wyłączony)
  Akcja: Wyślij formularz na partyboat.fun ze wszystkimi polami wypełnionymi
         (tak samo jak Test 1 Phase 1)
  Oczekiwany wynik:
    - Task stworzony z KLASYFIKACJA_AI = Gorący lead
    - BRAK komentarza z roboczą odpowiedzią (hot_lead_draft_enabled = false)
  Warunek zaliczenia: Gałąź gorącego leada poprawnie wyłączona.
```

```
Test 16: Robocza odpowiedź — Gorący lead (włączony)
  Akcja:
    Krok 1: Ustaw hot_lead_draft_enabled = true
    Krok 2: Wyślij formularz ze wszystkimi polami wypełnionymi
  Oczekiwany wynik:
    - Task stworzony z KLASYFIKACJA_AI = Gorący lead
    - Komentarz pojawia się z roboczą odpowiedzią potwierdzającą odbiór
    - Wersja robocza potwierdza szczegóły klientowi
    - Wersja robocza NIE zawiera cen
  Warunek zaliczenia: Wersja robocza dla gorącego leada działa, gdy włączona.
  Po teście: Ponownie ustaw hot_lead_draft_enabled = false.
```

```
Test 17: Robocza odpowiedź — Spam (brak wersji roboczej)
  Akcja: Wyślij formularz lub email, który będzie sklasyfikowany jako Spam
  Oczekiwany wynik:
    - Task stworzony z KLASYFIKACJA_AI = Spam
    - BRAK komentarza z roboczą odpowiedzią
    - BRAK ustawionego Due Date
  Warunek zaliczenia: Taski spamowe nie dostają wersji roboczej ani przypomnienia.
```

```
Test 18: Robocza odpowiedź — Niska pewność (brak wersji roboczej)
  Akcja: Wyślij graniczne/niejednoznaczne zapytanie, które AI klasyfikuje
         z AI_CONFIDENCE = Niska
  Oczekiwany wynik:
    - Task stworzony
    - BRAK komentarza z roboczą odpowiedzią (gating przez confidence blokuje wersję roboczą)
  Warunek zaliczenia: Leady o niskiej pewności nie dostają potencjalnie mylących wersji roboczych.
  Uwaga: Jeśli trudno naturalnie wywołać Niską pewność, zweryfikuj, że logika warunku
         router'a jest poprawna.
```

```
Test 19: Robocza odpowiedź — Awaria AI
  Akcja:
    Krok 1: Tymczasowo ustaw zły klucz API tylko dla modułu HTTP generowania wersji roboczych
            (zachowaj poprawny klucz API dla klasyfikacji)
    Krok 2: Wyślij formularz z niekompletnymi danymi
  Oczekiwany wynik:
    - Task stworzony normalnie z poprawną klasyfikacją i wszystkimi polami
    - BRAK komentarza z wersją roboczą (wywołanie AI wersji roboczej się nie powiodło)
    - Żaden błąd niewidoczny dla użytkownika — cicha degradacja
  Warunek zaliczenia: Awaria wersji roboczej nie wpływa na tworzenie taska ani klasyfikację.
  Po teście: Przywróć poprawny klucz API.
```

```
Test 20: ZLECENIA — Automatyczne tworzenie taska
  Akcja:
    Krok 1: Utwórz lub użyj istniejącego taska w dowolnym pipeline'u sprzedawcy
            (np. SPRZEDAŻ KUBA) z:
      - Tytuł: 15-06-2026 | KRAKÓW | BOAT | Jan Testowy
      - MIASTO: KRAKÓW
      - USŁUGA: BOAT
      - PAKIET: PARTY
      - DATA EVENTU: 2026-06-15
      - EMAIL: jan@test.pl
      - TELEFON: 500100200
      - ILOŚĆ OSÓB: 20
      - TYP IMPREZY: Wieczór kawalerski
    Krok 2: Zmień status taska na SPRZEDANE
  Oczekiwany wynik:
    - Nowy task pojawia się na liście ZLECENIA
    - Tytuł: "15-06-2026 | KRAKÓW | BOAT | Jan Testowy | PARTY" (rozszerzony o PAKIET)
    - Status: ZLECENIA BEZ ZALICZEK
    - Wszystkie zmapowane pola wypełnione (MIASTO, USŁUGA, PAKIET, DATA EVENTU,
      EMAIL, TELEFON, ILOŚĆ OSÓB)
    - Komentarz na tasku ZLECENIA: "📋 Zlecenie utworzone automatycznie z CRM..."
    - Komentarz na tasku CRM: "✅ Zlecenie utworzone automatycznie..."
    - Task CRM pozostaje w pipeline'u sprzedawcy ze statusem SPRZEDANE
  Warunek zaliczenia: Task ZLECENIA stworzony z poprawnymi danymi, wzajemne linki działają,
                      task CRM zachowany.
```

```
Test 21: ZLECENIA — Graceful handling brakujących pól
  Akcja: Przenieś task CRM z niektórymi pustymi polami (np. brak PAKIETU, brak GODZINY)
         do SPRZEDANE
  Oczekiwany wynik:
    - Task ZLECENIA stworzony
    - Tytuł radzi sobie z brakującym PAKIETEM gracefully (pusty segment lub "BRAK")
    - Puste pola są pozostawiane puste w tasku ZLECENIA
      (nie wypełniane "null" ani tekstem błędu)
    - Komentarze z wzajemnymi linkami nadal działają
  Warunek zaliczenia: Automatyzacja obsługuje niekompletne dane bez błędów.
```

```
Test 22: ZLECENIA — Awaria scenariusza
  Akcja:
    Krok 1: Tymczasowo zepsuj scenariusz ZLECENIA (wyłącz lub skonfiguruj błędnie)
    Krok 2: Przenieś task CRM do SPRZEDANE
  Oczekiwany wynik:
    - Wysłano email z powiadomieniem o błędzie do inżynierii
    - Task CRM pozostaje ze statusem SPRZEDANE — brak utraty danych
    - Brak taska ZLECENIA (potrzebny ręczny fallback)
  Warunek zaliczenia: Awaria jest notyfikowana, task CRM jest bezpieczny.
  Po teście: Przywróć scenariusz.
```

```
Test 23: Dedup Cleanup
  Akcja:
    Krok 1: Ręcznie wstaw 3 rekordy w EVSO_Lead_Dedup z created_at = 10 dni temu
    Krok 2: Wstaw 2 rekordy z created_at = dzisiaj
    Krok 3: Uruchom scenariusz EVSO - Dedup Cleanup ręcznie
  Oczekiwany wynik:
    - 3 stare rekordy usunięte
    - 2 nowe rekordy zachowane
  Warunek zaliczenia: Czyszczenie usuwa tylko wygasłe rekordy.
```

```
Test 24: Cykl życia end-to-end
  Akcja:
    Krok 1: Wyślij formularz z emailem lifecycle@test.pl, niekompletne dane (brak miasta)
      → Zweryfikuj: task stworzony, LICZBA_KONTAKTÓW=1, wersja robocza follow-upu opublikowana,
        Due Date ustawiony
    Krok 2: Wyślij email z lifecycle@test.pl z zawartym miastem
      → Zweryfikuj: scalenie do istniejącego taska, MIASTO zaktualizowane,
        LICZBA_KONTAKTÓW=2, tag dodany
    Krok 3: Przypisz task do sprzedawcy
      → Zweryfikuj: przypomnienie Due Date NIE odpali się (task ma teraz Assignee)
    Krok 4: Przenieś task przez pipeline sprzedawcy do SPRZEDANE
      → Zweryfikuj: task ZLECENIA stworzony automatycznie ze wszystkimi danymi
    Krok 5: Sprawdź Data Store — rekord istnieje z lifecycle@test.pl
    Krok 6: Uruchom czyszczenie po wygaśnięciu rekordu
      → Zweryfikuj: rekord usunięty
  Warunek zaliczenia: Kompletny cykl życia leada działa end-to-end ze wszystkimi funkcjami Phase 2.
```

---

## Niepewności i elementy do zbadania

Te pozycje z Eksperckiego Spotkania (M-002) muszą być zweryfikowane podczas wdrożenia:

| ID | Nieznane | Kiedy weryfikować | Wpływ jeśli błędne |
|----|----------|-------------------|---------------------|
| U-001 | Obecny proces handoff CRM→ZLECENIA | Dzień 0 | Błędny projekt automatyzacji |
| U-002 | Spójność statusów pipeline'ów sprzedawców | Dzień 1 | SPRZEDANE może wymagać konfiguracji per lista |
| U-003 | Głos marki EVSO dla odpowiedzi | Dzień 0 (zbierz przykłady) | Wersje robocze brzmią generycznie |
| U-004 | Poziom planu ClickUp dla automatyzacji | Dzień 1 | Może być potrzebna alternatywa oparta na Make dla przypomnień |
| U-005 | Dziedziczenie Custom Fields między listami | Dzień 8 (budowa ZLECENIA) | Mapowanie pól może wymagać ręcznej konfiguracji |
| U-006 | Wydajność Data Store przy wolumenie EVSO | Monitoring po wdrożeniu | Powinno być dobrze (~100 rekordów max) |
| U-007 | Payload webhook'a WPForms zawiera email przed wywołaniem AI | Dzień 2 (modyfikacja scenariusza) | Klucz dedup może wymagać dostosowania |

---

*Koniec przewodnika wdrożenia. Szacowany całkowity nakład: 12 dni inżynierskich + 1 dzień śledztwa w ciągu 3 tygodni.*
