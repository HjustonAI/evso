# EVSO — Automatyzacja Obsługi Zapytań: Przewodnik Wdrożeniowy Phase 1

**Wersja:** 1.0
**Data:** 2026-03-31
**Zakres:** Automatyzacja intake'u przez kanał formularza WWW i e-mail
**Odbiorca:** Jeden inżynier mid-level z doświadczeniem w Make.com i ClickUp
**Szacowany nakład:** 12 dni inżynierskich

**Walidacja przez panel ekspertów:** Każda decyzja projektowa w tym dokumencie została zwalidowana przez cztery perspektywy specjalistyczne — architektura ClickUp (Marta Kurek), automatyzacja Make (Tomasz Sowa), niezawodność AI (Aleksander Bryła) oraz projektowanie procesu sprzedażowego (Karolina Wysocka). Zakres był redukowany wszędzie tam, gdzie którykolwiek ekspert zgłosił zastrzeżenie. Patrz sekcja „Odłożone na Phase 2" na końcu dokumentu.

---

## Sekcja 1 — Analiza Aktualnego Stanu

### Jak zapytania trafiają do systemu

EVSO otrzymuje zapytania eventowe dwoma potwierdzonymi kanałami:

**Kanał A: Formularze na stronach WWW.** Trzy brandowe domeny — `partytram.fun`, `partyboat.fun`, `busparty.fun` — każda hostuje formularz zapytania oparty na WPForms. Formularz zbiera: miasto (dropdown, forma miejscownikowa, np. „w Krakowie"), rodzaj imprezy (dropdown: Wieczór kawalerski, Wieczór panieński, Urodziny, Impreza firmowa), pakiet (dropdown: FUN, PARTY, LUX, INDYWIDUALNY), imię i nazwisko klienta, e-mail, telefon, datę, godzinę, liczbę osób oraz pole tekstowe („Opisz swój pomysł"). Submission formularza wyzwala minimalny scenariusz Make.com: `WPForms submission → Make → ClickUp: Create Task`. Task ląduje w `EVSO / CRM / NOWE ZAPYTANIA` i otrzymuje custom ID w formacie `LEAD-####`.

**Kanał B: Bezpośrednie e-maile.** Klienci piszą też na brandowe adresy e-mail (`kontakt@partyboat.fun` itp.) lub bezpośrednio do konkretnych handlowców (`dawid.cybulkin@evso.pl`, `krzysztof.ksiazek@evso.pl`). Zapytania mailowe są semi-ustrukturyzowane lub w pełni nieustrukturyzowane — mogą zawierać część, całość lub żaden z punktów danych, które zbiera formularz. Nie istnieje potwierdzona automatyzacja importu zapytań mailowych do ClickUp. Obecna obsługa jest ręczna lub semi-ręczna: ktoś czyta maila, tworzy task i kopiuje treść do opisu taska.

### Gdzie tkwi tarcie

1. **Intake e-maili jest ręczny.** Każde zapytanie mailowe wymaga, żeby człowiek je przeczytał, stworzył task w ClickUp, wypełnił pola i skomponował tytuł taska zgodnie z konwencją nazewniczą. Przy ~100 zapytaniach tygodniowo, z których znaczna część przychodzi e-mailem, to największe pojedyncze źródło powtarzalnej pracy ręcznej.

2. **Brak klasyfikacji zapytań.** Gorące leady, niekompletne leady, ogólne pytania informacyjne i spam trafiają do NOWE ZAPYTANIA z tym samym statusem. Zespół sprzedaży musi otwierać i czytać każde zapytanie, żeby ocenić, czym jest. Nie ma szybkiego sposobu sortowania ani filtrowania po jakości zapytania.

3. **Normalizacja miasta — niezweryfikowana.** Formularz wysyła formy miejscownikowe („w Krakowie"), ale ClickUp przechowuje operacyjne formy skrócone („KRAKÓW"). Nie jest potwierdzone, że obecny scenariusz Make wykonuje to mapowanie. Jeśli nie — pola MIASTO mogą być puste lub niespójnie wypełnione.

4. **Niekompletne leady nie są oznaczane.** Zapytanie bez daty, miasta lub liczby osób wygląda identycznie jak kompletne w liście tasków. Luki w kompletności odkrywa się dopiero po otwarciu taska — co marnuje czas na leady wymagające follow-upu zanim można w ogóle przygotować ofertę.

5. **Mapowanie pól formularza do ClickUp jest minimalne.** Nie jest potwierdzone, że obecny scenariusz Make wypełnia wszystkie 11 Custom Fields. Część pól (MIASTO, TYP IMPREZY, PAKIET) może nie być ustawiana automatycznie, co wymusza ręczne uzupełnianie.

6. **Typ usługi jest wnioskowany, nie przechwytywany.** Formularz nie ma widocznego selektora „typ usługi". Domena (partytram = TRAM, partyboat = BOAT, busparty = BUS) sugeruje usługę, ale nie istnieje potwierdzony mechanizm mapujący to na pole USŁUGA w ClickUp.

7. **Brak wsparcia AI na jakimkolwiek etapie.** Każda decyzja o interpretacji, ekstrakcji i klasyfikacji jest podejmowana ręcznie przez człowieka.

### Co działa dobrze i musi pozostać niezmienione

1. **Konwencja nazewnicza tasków** — `[DATE|BRAK DATY] | [CITY|WYBIERZ MIASTO] | [SERVICE] | [NAME]` jest doskonała. Daje natychmiastowy kontekst operacyjny w widokach listy. Musi być zachowana dokładnie w tej formie.

2. **System custom ID `LEAD-####`** — stabilny, unikalny, już zintegrowany z przepływem pracy zespołu.

3. **NOWE ZAPYTANIA jako centralny zbiornik intake'u** — poprawny wybór architektoniczny. Wszystkie surowe leady trafiają w jedno miejsce przed dalszym routingiem.

4. **Listy pipeline'owe poszczególnych handlowców** (SPRZEDAŻ KUBA/WIKTOR/DAWID/KRIS) — rozdzielenie intake'u od pracy sprzedażowej poszczególnych opiekunów jest właściwe.

5. **Composer e-maila w tasku** — możliwość wysyłania wiadomości bezpośrednio z taska, z polem EMAIL sugerującym odbiorcę, to kluczowa zdolność operacyjna.

6. **Opis taska jako archiwum oryginalnej wiadomości** — zachowanie pełnej treści oryginalnego zapytania w opisie taska jest wartościowe pod kątem kontekstu i audytu.

7. **Separacja CRM ↔ ZLECENIA** — czysta granica między sprzedażą a operacjami.

### Dane zbierane vs. brakujące

**Aktualnie zbierane (11 Custom Fields):** MIASTO, PAKIET, POZYSKANIE, TELEFON, TYP IMPREZY, USŁUGA, GODZINA EVENTU, DATA EVENTU, EMAIL, ILOŚĆ OSÓB, JEDNOSTKA.

**Brakujące i potrzebne:**

| Luka | Dlaczego to ważne |
|------|-------------------|
| Klasyfikacja leada (gorący / niekompletny / pytanie ogólne / spam) | Zespół nie może ustalić priorytetów bez otwierania każdego taska |
| Wskaźnik kompletności | Brak sposobu filtrowania leadów gotowych do pracy vs. wymagających follow-upu |
| Kanał źródłowy (formularz vs. e-mail) | Nie można mierzyć efektywności kanałów ani dostosować obsługi |
| Podsumowanie generowane przez AI | Handlowiec musi czytać pełny opis, żeby zrozumieć każde zapytanie |
| Wskaźnik pewności AI | Brak informacji, czy dane wypełnione przez AI wymagają weryfikacji człowieka |

---

## Sekcja 2 — Proponowana Architektura Rozwiązania

### Przepływ end-to-end (sekwencja numerowana)

**Ścieżka A — Zapytanie przez formularz:**

1. Klient wysyła formularz na `partytram.fun`, `partyboat.fun` lub `busparty.fun`
2. WPForms generuje zdarzenie submission
3. Scenariusz Make.com nr 1 („Form Intake") odbiera submission przez webhook
4. Make normalizuje wartość miasta za pomocą statycznej tablicy mapowania (np. „w Krakowie" → „KRAKÓW")
5. Make ustala typ usługi na podstawie domeny źródłowej (partytram → TRAM, partyboat → BOAT, busparty → BUS)
6. Make wywołuje Anthropic API (Claude Haiku) w celu klasyfikacji zapytania i wygenerowania krótkiego podsumowania
7. Make oblicza wskaźnik kompletności na podstawie tego, które pola są wypełnione
8. Make tworzy task w ClickUp w NOWE ZAPYTANIA ze wszystkimi wypełnionymi Custom Fields, tytułem zgodnym z istniejącą konwencją, oryginalną wiadomością w opisie oraz wypełnionymi polami AI
9. Jeśli wywołanie AI nie powiedzie się: Make i tak tworzy task, zostawiając pola AI puste i ustawiając KLASYFIKACJA_AI na „Do weryfikacji"

**Ścieżka B — Zapytanie przez e-mail:**

1. Klient wysyła e-mail na adres brandowy lub bezpośrednio do handlowca
2. Scenariusz Make.com nr 2 („Email Intake") wykrywa nową wiadomość przez moduł Gmail Watch
3. Make wyciąga metadane e-maila (nadawca, temat, treść)
4. Make wywołuje Anthropic API (Claude Haiku) w celu ekstrakcji ustrukturyzowanych pól z treści e-maila (miasto, typ imprezy, data, godzina, liczba osób, pakiet, typ usługi, imię klienta, telefon) oraz klasyfikacji zapytania
5. Make normalizuje wyekstrahowaną wartość miasta za pomocą tej samej statycznej tablicy mapowania
6. Make oblicza wskaźnik kompletności na podstawie wyekstrahowanych pól
7. Make tworzy task w ClickUp w NOWE ZAPYTANIA z wyekstrahowanymi polami zmapowanymi na Custom Fields, treścią e-maila jako opisem oraz wypełnionymi polami AI
8. Jeśli wywołanie AI nie powiedzie się: Make tworzy task zawierający wyłącznie metadane e-maila (nadawca jako EMAIL, temat + treść jako opis) i ustawia KLASYFIKACJA_AI na „Do weryfikacji"

### Co zmienia się względem stanu aktualnego

- Scenariusz intake'u formularza zostaje zastąpiony wersją rozbudowaną, która normalizuje dane, ustala typ usługi, dodaje klasyfikację AI/podsumowanie i zapewnia wypełnienie wszystkich Custom Fields
- Zostaje dodany nowy scenariusz intake'u e-maili, który do tej pory nie istniał
- Do NOWE ZAPYTANIA zostaje dodanych 5 nowych Custom Fields (szczegóły w Sekcji 3)
- Żaden z istniejących statusów, widoków ani dalszych przepływów pracy nie jest modyfikowany

### Co celowo pozostaje bez zmian

- NOWE ZAPYTANIA pozostaje jedyną listą intake'ową
- Konwencja nazewnicza tasków jest zachowana w niezmienionej formie
- Schemat ID `LEAD-####` pozostaje nienaruszony
- Listy pipeline'owe poszczególnych handlowców pozostają niezmienione
- Workflow ZLECENIA pozostaje niezmieniony
- Przypisywanie handlowców pozostaje ręczne (bez automatycznego routingu)
- Cała komunikacja z klientami pozostaje inicjowana przez człowieka

### Wyraźna granica zakresu

**Niniejszy przewodnik obejmuje:**
- Dodanie nowych Custom Fields do NOWE ZAPYTANIA
- Dwa scenariusze Make.com (ulepszony intake formularza, intake e-maili)
- Integrację AI do klasyfikacji, ekstrakcji i podsumowań
- Testowanie powyższego

**NIE jest objęte (Phase 2):**
- Generowanie wersji roboczych odpowiedzi (draft replies)
- Automatyczny routing leadów według miasta lub typu usługi
- Automatyzacje przypomnień o follow-upach
- Automatyzacja workflow ZLECENIA
- Automatyczne odpowiedzi do klientów
- Deduplikacja leadów między kanałami
- Dashboardy analityczne

---

## Sekcja 3 — Przewodnik Konfiguracji ClickUp

### 3.1 Lists / Folders — bez zmian

Nie tworzy się żadnych nowych list ani folderów. Wszystkie zmiany dotyczą istniejącej listy `EVSO / CRM / NOWE ZAPYTANIA`.

### 3.2 Custom Fields do dodania

Dodaj poniższe 5 Custom Fields do listy `EVSO / CRM / NOWE ZAPYTANIA`. Nawigacja: przestrzeń EVSO → folder CRM → lista NOWE ZAPYTANIA → kliknij nagłówek kolumny `+` → „Create New Field".

**Pole 1: ŹRÓDŁO_KANAŁ**

| Właściwość | Wartość |
|-----------|---------|
| Nazwa | `ŹRÓDŁO_KANAŁ` |
| Typ | Dropdown |
| Opcje | `Formularz`, `Email`, `Inny` |
| Lokalizacja | Lista NOWE ZAPYTANIA |
| Cel | Identyfikuje, czy lead pochodzi z formularza WWW czy z e-maila |

**Pole 2: KLASYFIKACJA_AI**

| Właściwość | Wartość |
|-----------|---------|
| Nazwa | `KLASYFIKACJA_AI` |
| Typ | Dropdown |
| Opcje | `Gorący lead`, `Niekompletny lead`, `Zapytanie ogólne`, `Spam`, `Do weryfikacji` |
| Lokalizacja | Lista NOWE ZAPYTANIA |
| Cel | Klasyfikacja jakości zapytania przypisana przez AI. „Do weryfikacji" to wartość fallbackowa, gdy AI jest niedostępne lub poziom pewności jest niski |

**Pole 3: AI_PODSUMOWANIE**

| Właściwość | Wartość |
|-----------|---------|
| Nazwa | `AI_PODSUMOWANIE` |
| Typ | Short Text (lub Long Text, jeśli Short Text ma zbyt restrykcyjny limit znaków) |
| Lokalizacja | Lista NOWE ZAPYTANIA |
| Cel | Wygenerowane przez AI podsumowanie zapytania w 1-2 zdaniach po polsku |

**Pole 4: KOMPLETNOŚĆ**

| Właściwość | Wartość |
|-----------|---------|
| Nazwa | `KOMPLETNOŚĆ` |
| Typ | Number (liczba całkowita, bez miejsc dziesiętnych) |
| Lokalizacja | Lista NOWE ZAPYTANIA |
| Cel | Procent (0-100) informujący, ile kluczowych pól jest wypełnionych. Pomaga zespołowi sprzedaży priorytetyzować kompletne leady |

**Pole 5: AI_CONFIDENCE**

| Właściwość | Wartość |
|-----------|---------|
| Nazwa | `AI_CONFIDENCE` |
| Typ | Dropdown |
| Opcje | `Wysoka`, `Średnia`, `Niska` |
| Lokalizacja | Lista NOWE ZAPYTANIA |
| Cel | Wskazuje pewność klasyfikacji AI. Pomaga zespołowi wiedzieć, kiedy zweryfikować wyniki AI |

### 3.3 Istniejące Custom Fields — weryfikacja i dokumentacja

Przed budowaniem scenariuszy Make otwórz dowolny task w NOWE ZAPYTANIA i zweryfikuj dokładne nazwy i typy poniższych istniejących pól. Zapisz identyfikatory pól ClickUp (widoczne przez ClickUp API lub w strukturze URL taska) — będą potrzebne w modułach Make:

1. `MIASTO` — zweryfikuj, że typ to Dropdown. Zapisz wszystkie aktualne opcje dropdown.
2. `PAKIET` — zweryfikuj typ Dropdown. Oczekiwane opcje: FUN, PARTY, LUX, INDYWIDUALNY.
3. `POZYSKANIE` — zweryfikuj typ (prawdopodobnie Dropdown lub Short Text). Zapisz aktualne wartości.
4. `TELEFON` — zweryfikuj typ (prawdopodobnie Short Text lub Phone).
5. `TYP IMPREZY` — zweryfikuj typ Dropdown. Oczekiwane opcje: Wieczór kawalerski, Wieczór panieński, Urodziny, Impreza firmowa. Sprawdź, czy istnieją dodatkowe opcje.
6. `USŁUGA` — zweryfikuj typ Dropdown. Oczekiwane opcje: TRAM, BOAT. Sprawdź, czy istnieje BUS. Jeśli nie — dodaj.
7. `GODZINA EVENTU` — zweryfikuj typ (prawdopodobnie Short Text lub Time).
8. `DATA EVENTU` — zweryfikuj typ (prawdopodobnie Date).
9. `EMAIL` — zweryfikuj, że typ to Email.
10. `ILOŚĆ OSÓB` — zweryfikuj typ (prawdopodobnie Number lub Short Text).
11. `JEDNOSTKA` — zweryfikuj typ i aktualne opcje.

**Punkt do działania:** Jeśli dropdown `USŁUGA` nie zawiera opcji `BUS` — dodaj ją teraz. Domena `busparty.fun` musi mapować się na prawidłową wartość.

### 3.4 Statusy — bez zmian

Istniejąca struktura statusów w NOWE ZAPYTANIA zostaje zachowana. Nowe pole KLASYFIKACJA_AI realizuje cel klasyfikacyjny bez modyfikowania statusów workflow. To była celowa decyzja projektowa: statusy kontrolują workflow (co zrobić dalej), klasyfikacja to dane o zapytaniu (czym ono jest). Mieszanie tych dwóch warstw tworzyłoby tarcie dla zespołu sprzedaży.

### 3.5 ClickUp Automations — dodaj jedną

Dodaj jedną automatyzację do NOWE ZAPYTANIA, która pomoże zespołowi szybko dostrzec leady wymagające uwagi.

**Automatyzacja 1: Oznacz niską pewność klasyfikacji AI**

| Właściwość | Wartość |
|-----------|---------|
| Nazwa | `Flag AI Do Weryfikacji` |
| Lokalizacja | Lista NOWE ZAPYTANIA |
| Trigger | Gdy Custom Field `KLASYFIKACJA_AI` zostanie ustawione na `Do weryfikacji` |
| Warunek | Brak |
| Akcja | Ustaw Priority na `Urgent` (czerwona flaga) |

Dzięki temu taski, gdzie AI nie mogło przeprowadzić klasyfikacji (z powodu błędu lub niskiej pewności), są wizualnie oznaczone do natychmiastowej obsługi przez człowieka.

**Dlaczego tylko jedna automatyzacja:** Dodanie kolejnych automatyzacji (np. automatyczne przypisanie według miasta) było rozważane i odrzucone przez panel ekspertów. Powód: automatyczne przypisanie zmienia istniejący ręczny proces alokacji w zespole sprzedaży. Ta zmiana wymaga akceptacji zespołu i lepiej ją przeprowadzić w Phase 2, gdy warstwa intake'owa będzie już stabilna.

### 3.6 Views do skonfigurowania

Stwórz dwa nowe zapisane widoki w NOWE ZAPYTANIA, które pomogą zespołowi sprzedaży pracować z nowymi danymi.

**View 1: NOWE - PRIORYTET**

| Właściwość | Wartość |
|-----------|---------|
| Nazwa | `NOWE - PRIORYTET` |
| Typ | List |
| Filtr | KLASYFIKACJA_AI `is` `Gorący lead` LUB KLASYFIKACJA_AI `is` `Do weryfikacji` |
| Sortowanie | Data created, malejąco (najnowsze na górze) |
| Grupowanie | KLASYFIKACJA_AI |
| Widoczne kolumny | Name, KLASYFIKACJA_AI, KOMPLETNOŚĆ, MIASTO, USŁUGA, DATA EVENTU, AI_PODSUMOWANIE, Date created |

**View 2: NOWE - KOMPLETNOŚĆ**

| Właściwość | Wartość |
|-----------|---------|
| Nazwa | `NOWE - KOMPLETNOŚĆ` |
| Typ | List |
| Filtr | KLASYFIKACJA_AI `is not` `Spam` |
| Sortowanie | KOMPLETNOŚĆ, malejąco (najbardziej kompletne na górze) |
| Grupowanie | ŹRÓDŁO_KANAŁ |
| Widoczne kolumny | Name, KOMPLETNOŚĆ, KLASYFIKACJA_AI, MIASTO, USŁUGA, TYP IMPREZY, PAKIET, DATA EVENTU, EMAIL, TELEFON |

---

## Sekcja 4 — Przewodnik po Scenariuszach Make.com

### Scenariusz 1: Form Intake (Rozbudowany)

```
Scenariusz: EVSO - Form Intake Enhanced
Cel: Zastępuje istniejący minimalny scenariusz WPForms→ClickUp wersją normalizującą dane, klasyfikującą przez AI i w pełni wypełniającą wszystkie Custom Fields
Trigger: Webhook — Custom Webhook (odbiera submission WPForms)
```

**Konfiguracja wstępna:** Utwórz nowy scenariusz Make.com. NIE wyłączaj istniejącego scenariusza WPForms, dopóki ten nie zostanie w pełni przetestowany. Po walidacji wyłącz stary.

#### Moduł 1 — Webhooks: Custom Webhook

```
Typ: Webhooks > Custom Webhook
Ustawienia:
  - Webhook name: "EVSO Form Intake"
  - Data structure: zdefiniuj ręcznie lub wykryj automatycznie z pierwszego testowego submission WPForms
Zmienne wyjściowe:
  - form_name (string) — nazwa przesyłanego formularza
  - form_fields (object) — wszystkie wartości pól formularza
  - webhook_url (string) — URL strony, z której przyszło submission (używany do ustalenia domeny/brandu)
Uwagi:
  - Po stworzeniu webhooka skopiuj jego URL
  - W WPForms podmień istniejący URL webhooka Make na nowy
  - Przetestuj przez jednokrotne wysłanie formularza — Make powinno pokazać strukturę przychodzących danych
```

**Zweryfikuj w WPForms:** Sprawdź, jak WPForms wysyła dane. WPForms może używać własnego modułu Make/Integromat lub ogólnego webhooka. Dostosuj moduł trigger'a odpowiednio. Jeśli WPForms korzysta z natywnego modułu Make (`WPForms > Watch Form Submissions`), użyj go zamiast custom webhooka.

#### Moduł 2 — Tools: Set Multiple Variables (Normalizacja miasta)

```
Typ: Tools > Set Multiple Variables
Cel: Mapowanie polskich form miejscownikowych nazw miast na wartości kompatybilne z ClickUp
Ustawienia:
  Zmienna 1:
    Nazwa: city_normalized
    Wartość: Użyj łańcucha IF/SWITCH:
      {{if(form_fields.miasto = "w Katowicach"; "KATOWICE";
        if(form_fields.miasto = "we Wrocławiu"; "WROCŁAW";
        if(form_fields.miasto = "w Poznaniu"; "POZNAŃ";
        if(form_fields.miasto = "w Warszawie"; "WARSZAWA";
        if(form_fields.miasto = "w Krakowie"; "KRAKÓW";
        if(form_fields.miasto = "w Gdańsku / Sopocie"; "GDAŃSK";
        if(form_fields.miasto = "w Toruniu"; "TORUŃ";
        if(form_fields.miasto = "w Bydgoszczy"; "BYDGOSZCZ";
        if(form_fields.miasto = "w Szczecinie"; "SZCZECIN";
        "WYBIERZ MIASTO")))))))))}}

  Zmienna 2:
    Nazwa: service_type
    Wartość: Wywiedź z URL webhooka lub źródła formularza:
      {{if(contains(form_fields.source_url; "partytram"); "TRAM";
        if(contains(form_fields.source_url; "partyboat"); "BOAT";
        if(contains(form_fields.source_url; "busparty"); "BUS";
        "NIEZNANA")))}}

  Zmienna 3:
    Nazwa: task_date_display
    Wartość: Sformatuj datę do tytułu taska:
      {{if(form_fields.data != ""; formatDate(form_fields.data; "DD-MM-YYYY"); "BRAK DATY")}}

  Zmienna 4:
    Nazwa: source_brand
    Wartość:
      {{if(contains(form_fields.source_url; "partytram"); "PARTYTRAM.FUN";
        if(contains(form_fields.source_url; "partyboat"); "PARTYBOAT.FUN";
        if(contains(form_fields.source_url; "busparty"); "BUSPARTY.FUN";
        "NIEZNANE")))}}

Zmienne wyjściowe: city_normalized, service_type, task_date_display, source_brand
```

**Zweryfikuj:** Dokładna nazwa pola URL źródłowego zależy od sposobu, w jaki WPForms wysyła dane. Może znajdować się w nagłówkach webhooka (Referer) lub w ukrytym polu formularza. Potwierdź ścieżkę pola na podstawie rzeczywistego submission.

#### Moduł 3 — Tools: Set Multiple Variables (Wskaźnik kompletności)

```
Typ: Tools > Set Multiple Variables
Cel: Oblicz, ile z 8 kluczowych pól biznesowych jest wypełnionych
Ustawienia:
  Zmienna 1:
    Nazwa: completeness_score
    Wartość:
      {{round(
        (
          (if(form_fields.imie_nazwisko != ""; 1; 0)) +
          (if(form_fields.email != ""; 1; 0)) +
          (if(form_fields.telefon != ""; 1; 0)) +
          (if(city_normalized != "WYBIERZ MIASTO"; 1; 0)) +
          (if(form_fields.data != ""; 1; 0)) +
          (if(form_fields.godzina != ""; 1; 0)) +
          (if(form_fields.liczba_osob != ""; 1; 0)) +
          (if(form_fields.rodzaj_imprezy != ""; 1; 0))
        ) / 8 * 100
      )}}

Zmienne wyjściowe: completeness_score
```

**Uwaga:** Nazwy pól takie jak `form_fields.imie_nazwisko` są placeholderami. Zastąp je rzeczywistymi ścieżkami pól z payload'u webhooka WPForms po przetestowaniu Modułu 1.

#### Moduł 4 — HTTP: Make a Request (Klasyfikacja AI + podsumowanie)

```
Typ: HTTP > Make a Request
Cel: Wywołanie Anthropic API w celu klasyfikacji zapytania i wygenerowania podsumowania
Ustawienia:
  URL: https://api.anthropic.com/v1/messages
  Metoda: POST
  Nagłówki:
    - x-api-key: {{anthropic_api_key}}  (przechowaj w zmiennej scenariusza Make lub Data Store)
    - anthropic-version: 2023-06-01
    - content-type: application/json
  Typ body: Raw (JSON)
  Body:
```

```json
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 300,
  "messages": [
    {
      "role": "user",
      "content": "Przeanalizuj poniższe zapytanie eventowe i zwróć JSON.\n\nDane z formularza:\n- Miasto: {{form_fields.miasto}}\n- Rodzaj imprezy: {{form_fields.rodzaj_imprezy}}\n- Pakiet: {{form_fields.pakiet}}\n- Imię: {{form_fields.imie_nazwisko}}\n- Email: {{form_fields.email}}\n- Telefon: {{form_fields.telefon}}\n- Data: {{form_fields.data}}\n- Godzina: {{form_fields.godzina}}\n- Liczba osób: {{form_fields.liczba_osob}}\n- Opis: {{form_fields.opis}}\n- Źródło: {{source_brand}}\n\nZwróć TYLKO JSON (bez markdown):\n{\"klasyfikacja\": \"goracy_lead|niekompletny_lead|zapytanie_ogolne|spam\", \"confidence\": \"wysoka|srednia|niska\", \"podsumowanie\": \"1-2 zdania po polsku opisujące czego klient szuka\"}"
    }
  ],
  "system": "Jesteś klasyfikatorem zapytań eventowych dla firmy EVSO. Klasyfikujesz zapytania klientów dotyczące eventów (tramwaj imprezowy, statek imprezowy, bus imprezowy). Reguły klasyfikacji:\n- goracy_lead: podane miasto, data, typ imprezy, kontakt — zapytanie gotowe do ofertowania\n- niekompletny_lead: brakuje istotnych danych (data, miasto, lub liczba osób), ale intencja eventowa jest jasna\n- zapytanie_ogolne: pytanie o cennik, dostępność, ogólne informacje — nie jest to jeszcze konkretne zapytanie\n- spam: wiadomość nie dotyczy eventów, jest reklamą, lub jest niezrozumiała\nConfidence:\n- wysoka: klasyfikacja jest oczywista\n- srednia: są pewne niejasności ale klasyfikacja jest prawdopodobna\n- niska: nie jesteś pewien, człowiek powinien zweryfikować\nPodsumowanie pisz zwięźle po polsku, max 2 zdania."
}
```

```
  Parsuj odpowiedź: Tak
  Zmienne wyjściowe:
    - response_body (pełna odpowiedź API w formacie JSON)
    - Wyciągnij z response_body.content[0].text:
      - ai_klasyfikacja
      - ai_confidence
      - ai_podsumowanie
```

**Obsługa błędów:** Dodaj ścieżkę error handler'a po tym module:

```
Error handler: Resume (Wznów)
  W razie błędu: Ustaw zmienne na wartości fallbackowe:
    - ai_klasyfikacja = "do_weryfikacji"
    - ai_confidence = "niska"
    - ai_podsumowanie = ""
  Kontynuuj do następnego modułu (tworzenie taska odbywa się bez danych AI)
```

#### Moduł 5 — JSON: Parse JSON (Ekstrakcja odpowiedzi AI)

```
Typ: JSON > Parse JSON
Cel: Sparsuj tekstową odpowiedź AI na użyteczne zmienne
Ustawienia:
  JSON string: {{response_body.content[].text}}
  Data structure: Utwórz ręcznie:
    - klasyfikacja (text)
    - confidence (text)
    - podsumowanie (text)
Zmienne wyjściowe: klasyfikacja, confidence, podsumowanie
```

**Uwaga:** Claude może okazjonalnie opakowywać odpowiedź w markdown code fences. Jeśli zajdzie taka potrzeba, dodaj przed tym modułem moduł Tools > Text Parser używający wyrażenia regularnego do usunięcia znaczników ```: `{{replace(response_body.content[1].text; "/^```json?\n?|\n?```$/g"; "")}}`.

#### Moduł 6 — Tools: Set Multiple Variables (Mapowanie wyników AI na wartości ClickUp)

```
Typ: Tools > Set Multiple Variables
Cel: Mapuj ciągi wyjściowe AI na dokładne wartości dropdownów ClickUp
Ustawienia:
  Zmienna 1:
    Nazwa: clickup_klasyfikacja
    Wartość:
      {{if(klasyfikacja = "goracy_lead"; "Gorący lead";
        if(klasyfikacja = "niekompletny_lead"; "Niekompletny lead";
        if(klasyfikacja = "zapytanie_ogolne"; "Zapytanie ogólne";
        if(klasyfikacja = "spam"; "Spam";
        "Do weryfikacji"))))}}

  Zmienna 2:
    Nazwa: clickup_confidence
    Wartość:
      {{if(confidence = "wysoka"; "Wysoka";
        if(confidence = "srednia"; "Średnia";
        if(confidence = "niska"; "Niska";
        "Niska")))}}

  Zmienna 3:
    Nazwa: task_title
    Wartość:
      {{task_date_display}} | {{if(city_normalized != "WYBIERZ MIASTO"; city_normalized; "WYBIERZ MIASTO")}} | {{service_type}} | {{form_fields.imie_nazwisko}}
```

#### Moduł 7 — ClickUp: Create a Task

```
Typ: ClickUp > Create a Task
Ustawienia:
  Workspace: EVSO
  Space: EVSO
  Folder: CRM
  List: NOWE ZAPYTANIA
  Task Name: {{task_title}}
  Description: |
    --- ORYGINALNE ZAPYTANIE ---
    Źródło: {{source_brand}} (formularz)
    Data zgłoszenia: {{formatDate(now; "DD-MM-YYYY HH:mm")}}

    Miasto: {{form_fields.miasto}}
    Rodzaj imprezy: {{form_fields.rodzaj_imprezy}}
    Pakiet: {{form_fields.pakiet}}
    Data eventu: {{form_fields.data}}
    Godzina: {{form_fields.godzina}}
    Liczba osób: {{form_fields.liczba_osob}}

    Opis klienta:
    {{form_fields.opis}}
    ---
  Status: NOWE ZAPYTANIA
  Custom Fields:
    - MIASTO: {{city_normalized}}
    - PAKIET: {{form_fields.pakiet}}
    - POZYSKANIE: {{source_brand}}
    - TELEFON: {{form_fields.telefon}}
    - TYP IMPREZY: {{form_fields.rodzaj_imprezy}}
    - USŁUGA: {{service_type}}
    - GODZINA EVENTU: {{form_fields.godzina}}
    - DATA EVENTU: {{form_fields.data}}
    - EMAIL: {{form_fields.email}}
    - ILOŚĆ OSÓB: {{form_fields.liczba_osob}}
    - ŹRÓDŁO_KANAŁ: Formularz
    - KLASYFIKACJA_AI: {{clickup_klasyfikacja}}
    - AI_PODSUMOWANIE: {{podsumowanie}}
    - KOMPLETNOŚĆ: {{completeness_score}}
    - AI_CONFIDENCE: {{clickup_confidence}}
```

**Zweryfikuj w ClickUp:** Custom Fields w module ClickUp w Make są wskazywane przez ID pola, nie przez nazwę. Po stworzeniu 5 nowych pól (Sekcja 3.2) pobierz ich ID. W Make, podczas konfigurowania modułu ClickUp, pola powinny pojawić się w sekcji „Custom Fields", jeśli połączenie ma odpowiednie uprawnienia.

```
Obsługa błędów dla całego scenariusza:
  Dodaj końcową ścieżkę error handler'a, która:
  1. Wysyła e-mail z powiadomieniem o błędzie do kontaktu inżynierskiego/operacyjnego wraz ze szczegółami błędu
  2. Tworzy minimalny task w ClickUp z: tytuł = "BŁĄD INTAKE | {{form_fields.imie_nazwisko}}", opis = surowy payload formularza jako tekst
  Zapewnia to, że żadne zapytanie nie jest utracone nawet podczas awarii systemu.

Test case:
  Akcja: Wyślij formularz na partyboat.fun ze wszystkimi wypełnionymi polami (miasto: w Krakowie, rodzaj imprezy: Wieczór kawalerski, pakiet: PARTY, data: 2026-05-15, godzina: 20:00, liczba osób: 15, imię: Jan Testowy, e-mail: test@test.pl, telefon: 500100200, opis: "Chcemy wynająć statek na wieczór kawalerski")
  Oczekiwany rezultat: Task stworzony w ClickUp z tytułem "15-05-2026 | KRAKÓW | BOAT | Jan Testowy", MIASTO=KRAKÓW, USŁUGA=BOAT, KOMPLETNOŚĆ=100, KLASYFIKACJA_AI=Gorący lead, AI_PODSUMOWANIE wypełnione
```

---

### Scenariusz 2: Email Intake

```
Scenariusz: EVSO - Email Intake
Cel: Automatycznie tworzy taski ClickUp z zapytań e-mailowych wysyłanych na brandowe adresy, używając AI do ekstrakcji ustrukturyzowanych pól z nieustrukturyzowanej treści e-maila
Trigger: Gmail > Watch Emails
```

**Konfiguracja wstępna:** Konto/a Gmail używane do obsługi brandowych e-maili musi/muszą być połączone z Make.com. Jeśli brandowe e-maile są obsługiwane przez Google Workspace, połącz odpowiednie konto. Jeśli trzeba monitorować kilka skrzynek (kontakt@partyboat.fun, kontakt@partytram.fun itp.), utwórz albo jeden scenariusz na każdą skrzynkę, albo użyj filtrowania po etykietach Gmail w jednym Watch dla wszystkich.

#### Moduł 1 — Gmail: Watch Emails

```
Typ: Gmail > Watch Emails
Ustawienia:
  Connection: Połączenie Gmail EVSO (konto odbierające brandowe e-maile)
  Label: INBOX (lub konkretna etykieta, jeśli wiadomości są wstępnie filtrowane)
  Filter:
    - From: (pozostaw puste, żeby łapać wszystkie przychodzące)
    - Subject: (pozostaw puste)
    - Search query: (opcjonalnie — użyj "is:unread" żeby przetwarzać tylko nowe e-maile)
  Mark as read: Tak (po przetworzeniu)
  Maximum number of results: 10 (na cykl wykonania)
  Harmonogram: Co 5 minut (dostosuj w zależności od wolumenu i limitów planu Make)
Zmienne wyjściowe:
  - email_id
  - from_email (adres nadawcy)
  - from_name (wyświetlana nazwa nadawcy)
  - subject (temat)
  - text_body (treść plain text)
  - html_body (treść HTML — użyj jako fallback)
  - date (znacznik czasu otrzymania)
  - attachments (tablica)
```

**Ważne:** Jeśli EVSO ma wiele brandowych adresów e-mail na różnych kontach Gmail, utwórz osobny moduł Gmail Watch dla każdego LUB skonfiguruj reguły przekierowania e-maili tak, aby wszystkie brandowe e-maile trafiały do jednej skrzynki intake'owej. Podejście z jedną skrzynką jest prostsze w utrzymaniu.

#### Moduł 2 — Tools: Set Variable (Ustalenie brandu na podstawie adresu e-mail)

```
Typ: Tools > Set Multiple Variables
Cel: Identyfikuj, który brand/usługa jest powiązana z adresem e-mail
Ustawienia:
  Zmienna 1:
    Nazwa: email_brand
    Wartość:
      {{if(contains(1.to_email; "partytram"); "PARTYTRAM.FUN";
        if(contains(1.to_email; "partyboat"); "PARTYBOAT.FUN";
        if(contains(1.to_email; "busparty"); "BUSPARTY.FUN";
        if(contains(1.to_email; "evso"); "EVSO.PL";
        "NIEZNANE"))))}}

  Zmienna 2:
    Nazwa: email_service_hint
    Wartość:
      {{if(contains(1.to_email; "partytram"); "TRAM";
        if(contains(1.to_email; "partyboat"); "BOAT";
        if(contains(1.to_email; "busparty"); "BUS";
        "")))}}
```

**Uwaga:** Pole `1.to_email` może nie być bezpośrednio dostępne z modułu Gmail Watch. Zweryfikuj dokładną strukturę wyjścia. Może być konieczne użycie `1.headers` i szukanie nagłówka `To:`, lub użycie adresu połączenia Gmail jako proxy.

#### Moduł 3 — HTTP: Make a Request (Ekstrakcja AI + Klasyfikacja)

```
Typ: HTTP > Make a Request
Cel: Wywołanie Anthropic API w celu ekstrakcji ustrukturyzowanych pól z treści e-maila, klasyfikacji i wygenerowania podsumowania
Ustawienia:
  URL: https://api.anthropic.com/v1/messages
  Metoda: POST
  Nagłówki:
    - x-api-key: {{anthropic_api_key}}
    - anthropic-version: 2023-06-01
    - content-type: application/json
  Typ body: Raw (JSON)
```

```json
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 500,
  "messages": [
    {
      "role": "user",
      "content": "Przeanalizuj poniższy email od klienta i wyekstrahuj dane do JSON.\n\nNadawca: {{from_name}} <{{from_email}}>\nTemat: {{subject}}\nTreść:\n{{text_body}}\n\nKontekst: email wysłany na adres {{email_brand}} (firma EVSO — eventy na tramwajach, statkach i busach imprezowych w Polsce).\n\nZwróć TYLKO JSON (bez markdown):\n{\"imie_nazwisko\": \"string lub null\", \"telefon\": \"string lub null\", \"miasto\": \"string lub null — użyj formy mianownikowej np. KRAKÓW, WROCŁAW\", \"typ_imprezy\": \"Wieczór kawalerski|Wieczór panieński|Urodziny|Impreza firmowa|Inny|null\", \"usluga\": \"TRAM|BOAT|BUS|null\", \"pakiet\": \"FUN|PARTY|LUX|INDYWIDUALNY|null\", \"data_eventu\": \"DD-MM-YYYY lub null\", \"godzina\": \"HH:MM lub null\", \"liczba_osob\": \"number lub null\", \"klasyfikacja\": \"goracy_lead|niekompletny_lead|zapytanie_ogolne|spam\", \"confidence\": \"wysoka|srednia|niska\", \"podsumowanie\": \"1-2 zdania po polsku\"}"
    }
  ],
  "system": "Jesteś ekspertem od ekstrakcji danych z zapytań eventowych dla firmy EVSO w Polsce. Firma oferuje eventy na tramwajach imprezowych (TRAM), statkach imprezowych (BOAT) i busach imprezowych (BUS) w miastach: Katowice, Wrocław, Poznań, Warszawa, Kraków, Gdańsk, Toruń, Bydgoszcz, Szczecin.\n\nReguły ekstrakcji:\n- Wyciągaj TYLKO informacje jawnie podane w treści. Nie zgaduj.\n- Jeśli pole nie jest wyraźnie podane, zwróć null.\n- Miasto normalizuj do formy mianownikowej WIELKIMI LITERAMI: KRAKÓW, WROCŁAW, itp.\n- Jeśli klient nie podał usługi ale email przyszedł na adres brandowy, ustaw usluga na podstawie brandu (partytram→TRAM, partyboat→BOAT, busparty→BUS).\n- Datę formatuj jako DD-MM-YYYY.\n- Godzinę formatuj jako HH:MM.\n\nReguły klasyfikacji:\n- goracy_lead: podane miasto + data + typ imprezy + kontakt — gotowe do ofertowania\n- niekompletny_lead: intencja eventowa jest jasna ale brakuje istotnych danych\n- zapytanie_ogolne: pytanie o cennik, dostępność — nie konkretne zapytanie\n- spam: nie dotyczy eventów, reklama, lub niezrozumiałe\n\nConfidence:\n- wysoka: ekstrakcja i klasyfikacja oczywiste\n- srednia: pewne niejasności\n- niska: duża niepewność — człowiek powinien zweryfikować"
}
```

```
Obsługa błędów: Resume (Wznów)
  W razie błędu: Ustaw zmienne fallbackowe:
    - extracted = null (wszystkie pola ekstrakcji)
    - ai_klasyfikacja = "do_weryfikacji"
    - ai_confidence = "niska"
    - ai_podsumowanie = ""
  Kontynuuj tworzenie taska wyłącznie z metadanymi e-maila
```

#### Moduł 4 — JSON: Parse JSON

```
Typ: JSON > Parse JSON
Cel: Sparsuj odpowiedź ekstrakcji AI
Ustawienia:
  JSON string: {{oczyszczony tekst odpowiedzi AI}}
  Data structure:
    - imie_nazwisko (text)
    - telefon (text)
    - miasto (text)
    - typ_imprezy (text)
    - usluga (text)
    - pakiet (text)
    - data_eventu (text)
    - godzina (text)
    - liczba_osob (text)
    - klasyfikacja (text)
    - confidence (text)
    - podsumowanie (text)
```

#### Moduł 5 — Tools: Set Multiple Variables (Budowanie danych taska)

```
Typ: Tools > Set Multiple Variables
Cel: Przygotuj wszystkie wartości do tworzenia taska w ClickUp
Ustawienia:
  Zmienna 1:
    Nazwa: final_name
    Wartość: Użyj wyekstrahowanego imienia, fallback na nazwę nadawcy e-maila:
      {{if(parsed.imie_nazwisko != null; parsed.imie_nazwisko; if(from_name != ""; from_name; from_email))}}

  Zmienna 2:
    Nazwa: final_city
    Wartość:
      {{if(parsed.miasto != null; parsed.miasto; "WYBIERZ MIASTO")}}

  Zmienna 3:
    Nazwa: final_service
    Wartość:
      {{if(parsed.usluga != null; parsed.usluga; email_service_hint)}}

  Zmienna 4:
    Nazwa: final_date_display
    Wartość:
      {{if(parsed.data_eventu != null; parsed.data_eventu; "BRAK DATY")}}

  Zmienna 5:
    Nazwa: task_title
    Wartość:
      {{final_date_display}} | {{final_city}} | {{final_service}} | {{final_name}}

  Zmienna 6:
    Nazwa: completeness_score
    Wartość:
      {{round(
        (
          (if(final_name != "" AND final_name != from_email; 1; 0)) +
          (if(from_email != ""; 1; 0)) +
          (if(parsed.telefon != null; 1; 0)) +
          (if(final_city != "WYBIERZ MIASTO"; 1; 0)) +
          (if(parsed.data_eventu != null; 1; 0)) +
          (if(parsed.godzina != null; 1; 0)) +
          (if(parsed.liczba_osob != null; 1; 0)) +
          (if(parsed.typ_imprezy != null; 1; 0))
        ) / 8 * 100
      )}}

  Zmienna 7:
    Nazwa: clickup_klasyfikacja
    Wartość:
      {{if(parsed.klasyfikacja = "goracy_lead"; "Gorący lead";
        if(parsed.klasyfikacja = "niekompletny_lead"; "Niekompletny lead";
        if(parsed.klasyfikacja = "zapytanie_ogolne"; "Zapytanie ogólne";
        if(parsed.klasyfikacja = "spam"; "Spam";
        "Do weryfikacji"))))}}

  Zmienna 8:
    Nazwa: clickup_confidence
    Wartość:
      {{if(parsed.confidence = "wysoka"; "Wysoka";
        if(parsed.confidence = "srednia"; "Średnia";
        "Niska"))}}
```

#### Moduł 6 — ClickUp: Create a Task

```
Typ: ClickUp > Create a Task
Ustawienia:
  Workspace: EVSO
  Space: EVSO
  Folder: CRM
  List: NOWE ZAPYTANIA
  Task Name: {{task_title}}
  Description: |
    --- ORYGINALNE ZAPYTANIE (EMAIL) ---
    Od: {{from_name}} <{{from_email}}>
    Temat: {{subject}}
    Data otrzymania: {{formatDate(1.date; "DD-MM-YYYY HH:mm")}}
    Źródło: {{email_brand}}

    Treść wiadomości:
    {{text_body}}
    ---
  Status: NOWE ZAPYTANIA
  Custom Fields:
    - MIASTO: {{final_city}}
    - PAKIET: {{if(parsed.pakiet != null; parsed.pakiet; "")}}
    - POZYSKANIE: {{email_brand}}
    - TELEFON: {{if(parsed.telefon != null; parsed.telefon; "")}}
    - TYP IMPREZY: {{if(parsed.typ_imprezy != null AND parsed.typ_imprezy != "Inny"; parsed.typ_imprezy; "")}}
    - USŁUGA: {{final_service}}
    - GODZINA EVENTU: {{if(parsed.godzina != null; parsed.godzina; "")}}
    - DATA EVENTU: {{if(parsed.data_eventu != null; parsed.data_eventu; "")}}
    - EMAIL: {{from_email}}
    - ILOŚĆ OSÓB: {{if(parsed.liczba_osob != null; parsed.liczba_osob; "")}}
    - ŹRÓDŁO_KANAŁ: Email
    - KLASYFIKACJA_AI: {{clickup_klasyfikacja}}
    - AI_PODSUMOWANIE: {{if(parsed.podsumowanie != null; parsed.podsumowanie; "")}}
    - KOMPLETNOŚĆ: {{completeness_score}}
    - AI_CONFIDENCE: {{clickup_confidence}}
```

```
Obsługa błędów dla całego scenariusza:
  W przypadku nieodwracalnego błędu:
  1. Wyślij e-mail z powiadomieniem o błędzie wraz ze szczegółami
  2. Zastosuj etykietę Gmail "INTAKE_ERROR" do oryginalnego e-maila — żeby nie został utracony
  3. E-mail zostaje w skrzynce odbiorczej i może być przetworzony ręcznie

Test case:
  Akcja: Wyślij e-mail na kontakt@partyboat.fun z tematem "Wieczór kawalerski Kraków" i treścią "Cześć, chciałbym zorganizować wieczór kawalerski na statku w Krakowie, 15 maja 2026, ok 20 osób. Dajcie znać ile kosztuje pakiet PARTY. Pozdrawiam, Marek Kowalski, tel 600100200"
  Oczekiwany rezultat: Task stworzony z tytułem "15-05-2026 | KRAKÓW | BOAT | Marek Kowalski", MIASTO=KRAKÓW, USŁUGA=BOAT, TYP IMPREZY=Wieczór kawalerski, ILOŚĆ OSÓB=20, PAKIET=PARTY, KOMPLETNOŚĆ=100, KLASYFIKACJA_AI=Gorący lead
```

---

## Sekcja 5 — Specyfikacja Integracji AI

### Wybór modelu

**Model:** `claude-haiku-4-5-20251001` przez Anthropic API

**Uzasadnienie:**

| Czynnik | Ocena |
|---------|-------|
| Koszt | ~0,25 USD za 1M tokenów wejściowych, ~1,25 USD za 1M tokenów wyjściowych. Przy 100 zapytaniach tygodniowo, ~500 tokenach wejściowych i ~200 tokenach wyjściowych na zapytanie, szacowany koszt miesięczny: ~2-3 USD. Pomijalny. |
| Szybkość | Mediana latencji ~0,5-1s na żądanie. Akceptowalne dla asynchronicznego przetwarzania intake'u. |
| Język polski | Silna kompetencja w rozumieniu i generowaniu języka polskiego. Obsługuje formy miejscownikowe/mianownikowe miast, język potoczny i słownictwo branżowe. |
| Ekstrakcja ustrukturyzowana | Niezawodne wyjście JSON dla prostych schematów. Niski wskaźnik halucynacji przy zadaniach ekstrakcji. |
| Dopasowanie do zadania | Klasyfikacja i ekstrakcja z krótkich tekstów mieści się w możliwościach Haiku. Użycie większego modelu (Sonnet, Opus) zwiększyłoby koszt bez mierzalnej poprawy jakości dla tej złożoności zadań. |

**Alternatywa:** `gpt-4o-mini` przez OpenAI API. Zbliżony profil ceny i szybkości. Wybierz na podstawie tego, do którego API zespół już posiada klucze. Prompty w tym przewodniku używają formatowania w stylu Claude, ale są neutralne modelowo pod względem treści.

### Metoda wywołania API w Make.com

Wszystkie wywołania AI używają modułu `HTTP > Make a Request` z Anthropic Messages API.

**Konfiguracja połączenia:**

1. W Make.com przejdź do scenariusza i dodaj nowe połączenie typu „HTTP"
2. Nie potrzeba prekonfigurowanego uwierzytelniania — klucz API jest przekazywany w nagłówku żądania
3. Przechowuj klucz Anthropic API jako zmienną scenariusza Make.com lub w Data Store dla bezpieczeństwa. Nie wkodowuj go na stałe w module.

**Endpoint API:** `https://api.anthropic.com/v1/messages`

**Wymagane nagłówki dla każdego żądania:**

```
x-api-key: {{anthropic_api_key}}
anthropic-version: 2023-06-01
content-type: application/json
```

### Zadanie AI nr 1: Klasyfikacja zapytań (kanał formularza)

**Nazwa zadania:** Klasyfikacja i podsumowanie zapytania z formularza

**Kiedy używane:** Scenariusz 1 (Form Intake), Moduł 4

**System prompt (gotowy do skopiowania):**

```
Jesteś klasyfikatorem zapytań eventowych dla firmy EVSO. Klasyfikujesz zapytania klientów dotyczące eventów (tramwaj imprezowy, statek imprezowy, bus imprezowy). Reguły klasyfikacji:
- goracy_lead: podane miasto, data, typ imprezy, kontakt — zapytanie gotowe do ofertowania
- niekompletny_lead: brakuje istotnych danych (data, miasto, lub liczba osób), ale intencja eventowa jest jasna
- zapytanie_ogolne: pytanie o cennik, dostępność, ogólne informacje — nie jest to jeszcze konkretne zapytanie
- spam: wiadomość nie dotyczy eventów, jest reklamą, lub jest niezrozumiała
Confidence:
- wysoka: klasyfikacja jest oczywista
- srednia: są pewne niejasności ale klasyfikacja jest prawdopodobna
- niska: nie jesteś pewien, człowiek powinien zweryfikować
Podsumowanie pisz zwięźle po polsku, max 2 zdania.
```

**Template wiadomości użytkownika:**

```
Przeanalizuj poniższe zapytanie eventowe i zwróć JSON.

Dane z formularza:
- Miasto: {{form_fields.miasto}}
- Rodzaj imprezy: {{form_fields.rodzaj_imprezy}}
- Pakiet: {{form_fields.pakiet}}
- Imię: {{form_fields.imie_nazwisko}}
- Email: {{form_fields.email}}
- Telefon: {{form_fields.telefon}}
- Data: {{form_fields.data}}
- Godzina: {{form_fields.godzina}}
- Liczba osób: {{form_fields.liczba_osob}}
- Opis: {{form_fields.opis}}
- Źródło: {{source_brand}}

Zwróć TYLKO JSON (bez markdown):
{"klasyfikacja": "goracy_lead|niekompletny_lead|zapytanie_ogolne|spam", "confidence": "wysoka|srednia|niska", "podsumowanie": "1-2 zdania po polsku opisujące czego klient szuka"}
```

**Oczekiwany format wyjścia:**

```json
{
  "klasyfikacja": "goracy_lead",
  "confidence": "wysoka",
  "podsumowanie": "Klient szuka wieczoru kawalerskiego na statku w Krakowie na 15 maja dla 20 osób, zainteresowany pakietem PARTY."
}
```

**Parsowanie downstream:** JSON jest parsowany w Module 5 (JSON > Parse JSON). Wartość `klasyfikacja` jest mapowana na wartości dropdownów ClickUp w Module 6.

### Zadanie AI nr 2: Ekstrakcja pól z e-maila + klasyfikacja

**Nazwa zadania:** Ekstrakcja, klasyfikacja i podsumowanie zapytania mailowego

**Kiedy używane:** Scenariusz 2 (Email Intake), Moduł 3

**System prompt (gotowy do skopiowania):**

```
Jesteś ekspertem od ekstrakcji danych z zapytań eventowych dla firmy EVSO w Polsce. Firma oferuje eventy na tramwajach imprezowych (TRAM), statkach imprezowych (BOAT) i busach imprezowych (BUS) w miastach: Katowice, Wrocław, Poznań, Warszawa, Kraków, Gdańsk, Toruń, Bydgoszcz, Szczecin.

Reguły ekstrakcji:
- Wyciągaj TYLKO informacje jawnie podane w treści. Nie zgaduj.
- Jeśli pole nie jest wyraźnie podane, zwróć null.
- Miasto normalizuj do formy mianownikowej WIELKIMI LITERAMI: KRAKÓW, WROCŁAW, itp.
- Jeśli klient nie podał usługi ale email przyszedł na adres brandowy, ustaw usluga na podstawie brandu (partytram→TRAM, partyboat→BOAT, busparty→BUS).
- Datę formatuj jako DD-MM-YYYY.
- Godzinę formatuj jako HH:MM.

Reguły klasyfikacji:
- goracy_lead: podane miasto + data + typ imprezy + kontakt — gotowe do ofertowania
- niekompletny_lead: intencja eventowa jest jasna ale brakuje istotnych danych
- zapytanie_ogolne: pytanie o cennik, dostępność — nie konkretne zapytanie
- spam: nie dotyczy eventów, reklama, lub niezrozumiałe

Confidence:
- wysoka: ekstrakcja i klasyfikacja oczywiste
- srednia: pewne niejasności
- niska: duża niepewność — człowiek powinien zweryfikować
```

**Template wiadomości użytkownika:**

```
Przeanalizuj poniższy email od klienta i wyekstrahuj dane do JSON.

Nadawca: {{from_name}} <{{from_email}}>
Temat: {{subject}}
Treść:
{{text_body}}

Kontekst: email wysłany na adres {{email_brand}} (firma EVSO — eventy na tramwajach, statkach i busach imprezowych w Polsce).

Zwróć TYLKO JSON (bez markdown):
{"imie_nazwisko": "string lub null", "telefon": "string lub null", "miasto": "string lub null — użyj formy mianownikowej np. KRAKÓW, WROCŁAW", "typ_imprezy": "Wieczór kawalerski|Wieczór panieński|Urodziny|Impreza firmowa|Inny|null", "usluga": "TRAM|BOAT|BUS|null", "pakiet": "FUN|PARTY|LUX|INDYWIDUALNY|null", "data_eventu": "DD-MM-YYYY lub null", "godzina": "HH:MM lub null", "liczba_osob": "number lub null", "klasyfikacja": "goracy_lead|niekompletny_lead|zapytanie_ogolne|spam", "confidence": "wysoka|srednia|niska", "podsumowanie": "1-2 zdania po polsku"}
```

**Oczekiwany format wyjścia:**

```json
{
  "imie_nazwisko": "Marek Kowalski",
  "telefon": "600100200",
  "miasto": "KRAKÓW",
  "typ_imprezy": "Wieczór kawalerski",
  "usluga": "BOAT",
  "pakiet": "PARTY",
  "data_eventu": "15-05-2026",
  "godzina": null,
  "liczba_osob": 20,
  "klasyfikacja": "goracy_lead",
  "confidence": "wysoka",
  "podsumowanie": "Klient szuka wieczoru kawalerskiego na statku w Krakowie na 15 maja dla 20 osób, zainteresowany pakietem PARTY."
}
```

### Wyraźne granice dla AI

AI **NIE MOŻE**:

1. **Ustalać cen** — ceny zależą od pakietu, miasta, daty, liczby osób i dostępności. AI nie ma tego kontekstu i nie może sugerować cen.
2. **Routować leadów do handlowców** — przypisanie handlowca to decyzja człowieka oparta na obciążeniu, terytorium i dynamice zespołu.
3. **Wysyłać jakichkolwiek wiadomości do klientów** — cała komunikacja wychodząca jest inicjowana przez człowieka z poziomu tasków ClickUp.
4. **Podejmować decyzji kwalifikacyjnych** — AI klasyfikuje do celów priorytetyzacji, ale człowiek decyduje, czy kontynuować pracę z leadem.
5. **Nadpisywać poprawek człowieka** — jeśli handlowiec zmieni pole KLASYFIKACJA_AI lub inne pole wypełnione przez AI, system nie przywraca wartości AI.
6. **Wymyślać danych** — prompty ekstrakcji jawnie nakazują modelowi zwracać `null` dla pól nieobecnych w wejściu. Jeśli AI zwraca wartość, musi być ona możliwa do prześledzenia w oryginalnej treści wiadomości.

### Fallback na wypadek awarii

W obu scenariuszach, jeśli wywołanie Anthropic API nie powiedzie się (timeout, błąd 5xx, rate limit, zniekształcona odpowiedź):

1. Error handler w Make.com przechwytuje awarię.
2. Pola zależne od AI są ustawiane na wartości fallbackowe: KLASYFIKACJA_AI = „Do weryfikacji", AI_CONFIDENCE = „Niska", AI_PODSUMOWANIE = „" (puste).
3. Wszystkie pola nie-AI (z danych formularza lub metadanych e-maila) są nadal normalnie wypełniane.
4. Task ClickUp jest tworzony z wartościami fallbackowymi.
5. Automatyzacja ClickUp (Sekcja 3.5) ustawia Priority na Urgent, oznaczając task do uwagi człowieka.
6. Do kontaktu operacyjnego wysyłane jest powiadomienie e-mail ze szczegółami błędu.

**Rezultat:** System nigdy nie blokuje się na awarii AI. Każde zapytanie dostaje swój task. Taski z nieudanym AI są wizualnie oznaczone i obsługiwane ręcznie — dokładnie tak, jak byłyby obsługiwane bez jakiejkolwiek automatyzacji.

---

## Sekcja 6 — Harmonogram Wdrożenia

### Tydzień 1: Konfiguracja ClickUp + Scenariusz formularza

```
Dzień 1: Konfiguracja pól ClickUp
  Zadania:
    1. Otwórz listę NOWE ZAPYTANIA w ClickUp
    2. Udokumentuj wszystkie istniejące ID Custom Fields (potrzebne do modułów Make)
    3. Zweryfikuj typy istniejących pól i opcje dropdownów zgodnie z oczekiwaniami (Sekcja 3.3)
    4. Dodaj opcję "BUS" do USŁUGA jeśli brakuje
    5. Utwórz 5 nowych Custom Fields (Sekcja 3.2): ŹRÓDŁO_KANAŁ, KLASYFIKACJA_AI, AI_PODSUMOWANIE, KOMPLETNOŚĆ, AI_CONFIDENCE
    6. Utwórz automatyzację ClickUp "Flag AI Do Weryfikacji" (Sekcja 3.5)
    7. Utwórz dwa nowe widoki: NOWE - PRIORYTET, NOWE - KOMPLETNOŚĆ (Sekcja 3.6)
  Wymaganie wstępne: Brak
  Rezultat: Cała konfiguracja ClickUp zakończona

Dzień 2: Konfiguracja Anthropic API + testowanie promptów AI
  Zadania:
    1. Utwórz konto Anthropic API jeśli nie istnieje, wygeneruj klucz API
    2. Ręcznie przetestuj prompt klasyfikacji formularza (Zadanie AI nr 1) używając konsoli Anthropic lub API playground
    3. Wyślij 5 testowych zapytań przez prompt — obejmij: kompletny lead, częściowy lead, ogólne pytanie, spam, zapytanie po angielsku
    4. Zweryfikuj, że wyjście JSON parsuje się poprawnie dla wszystkich 5 przypadków
    5. Dostosuj sformułowanie promptu jeśli któraś klasyfikacja jest niepoprawna
    6. Przechowaj klucz API bezpiecznie w Make.com (zmienna scenariusza lub Data Store)
  Wymaganie wstępne: Brak (można realizować równolegle z Dniem 1)
  Rezultat: Zwalidowane prompty AI, klucz API przechowywany w Make

Dzień 3: Budowa scenariusza Form Intake w Make.com
  Zadania:
    1. Utwórz nowy scenariusz Make "EVSO - Form Intake Enhanced"
    2. Dodaj Moduł 1: trigger Webhook (Sekcja 4, Scenariusz 1, Moduł 1)
    3. Dodaj Moduł 2: normalizacja miasta (Sekcja 4, Scenariusz 1, Moduł 2)
    4. Dodaj Moduł 3: wskaźnik kompletności (Sekcja 4, Scenariusz 1, Moduł 3)
    5. Dodaj Moduł 4: wywołanie HTTP klasyfikacji AI (Sekcja 4, Scenariusz 1, Moduł 4)
    6. Dodaj Moduł 5: parser JSON (Sekcja 4, Scenariusz 1, Moduł 5)
    7. Dodaj Moduł 6: mapowanie wartości (Sekcja 4, Scenariusz 1, Moduł 6)
    8. Dodaj Moduł 7: ClickUp Create Task (Sekcja 4, Scenariusz 1, Moduł 7)
    9. Dodaj error handlery dla modułu AI i całego scenariusza
    10. Skonfiguruj WPForms do wysyłania na nowy URL webhooka (NIE wyłączaj jeszcze starego scenariusza)
  Wymaganie wstępne: Dzień 1 (pola ClickUp istnieją), Dzień 2 (klucz API przechowywany)
  Rezultat: Scenariusz Form Intake zbudowany, jeszcze nie na produkcji

Dzień 4: Testowanie scenariusza Form Intake
  Zadania:
    1. Wyślij testowy formularz na każdej z 3 domen (partytram, partyboat, busparty)
    2. Zweryfikuj tworzenie taska w ClickUp — sprawdź wszystkie 16 Custom Fields
    3. Zweryfikuj konwencję nazewniczą taska zgodną z oczekiwanym wzorcem
    4. Zweryfikuj normalizację miasta dla co najmniej 3 różnych miast
    5. Zweryfikuj wywodzenie typu usługi z każdej domeny
    6. Przetestuj fallback przy awarii AI: tymczasowo ustaw zły klucz API, wyślij formularz, zweryfikuj, że task jest tworzony z "Do weryfikacji"
    7. Napraw wszelkie wykryte problemy
  Wymaganie wstępne: Dzień 3
  Rezultat: Scenariusz Form Intake zwalidowany

Dzień 5: Uruchomienie Form Intake na produkcji
  Zadania:
    1. Wyłącz stary scenariusz WPForms → Make
    2. Włącz nowy scenariusz Form Intake Enhanced
    3. Monitoruj pierwsze 5-10 rzeczywistych submissionów formularza
    4. Zweryfikuj, że taski pojawiają się poprawnie w ClickUp
    5. Sprawdź z zespołem sprzedaży, że nowe pola są widoczne i użyteczne
    6. Udokumentuj wszelkie wykryte problemy do późniejszej naprawy
  Wymaganie wstępne: Dzień 4 (testy zaliczone)
  Rezultat: Form Intake uruchomiony na produkcji
```

### Tydzień 2: Scenariusz e-mail

```
Dzień 6: Konfiguracja kanału e-mail + testowanie promptów AI
  Zadania:
    1. Zidentyfikuj wszystkie konta/skrzynki Gmail odbierające brandowe e-maile
    2. Zdecyduj o podejściu: wiele modułów Gmail Watch czy jedna skrzynka z przekierowaniem
    3. Połącz odpowiednie konto/a Gmail z Make.com
    4. Ręcznie przetestuj prompt ekstrakcji z e-maila (Zadanie AI nr 2) na 5 próbkach e-maili:
       - Kompletne zapytanie po polsku
       - Częściowe zapytanie (brak daty i miasta)
       - Ogólne pytanie o cennik
       - Spam/e-mail marketingowy
       - Zapytanie po angielsku
    5. Zweryfikuj dokładność ekstrakcji JSON dla wszystkich 5 przypadków
    6. Dostosuj prompt jeśli potrzeba
  Wymaganie wstępne: Dzień 2 (klucz API skonfigurowany)
  Rezultat: Gmail połączony, prompt AI dla e-maili zwalidowany

Dzień 7: Budowa scenariusza Email Intake w Make.com
  Zadania:
    1. Utwórz nowy scenariusz Make "EVSO - Email Intake"
    2. Dodaj Moduł 1: Gmail Watch (Sekcja 4, Scenariusz 2, Moduł 1)
    3. Dodaj Moduł 2: wykrywanie brandu (Sekcja 4, Scenariusz 2, Moduł 2)
    4. Dodaj Moduł 3: wywołanie HTTP ekstrakcji AI (Sekcja 4, Scenariusz 2, Moduł 3)
    5. Dodaj Moduł 4: parser JSON (Sekcja 4, Scenariusz 2, Moduł 4)
    6. Dodaj Moduł 5: builder zmiennych (Sekcja 4, Scenariusz 2, Moduł 5)
    7. Dodaj Moduł 6: ClickUp Create Task (Sekcja 4, Scenariusz 2, Moduł 6)
    8. Dodaj error handlery (etykieta Gmail przy błędzie, powiadomienie e-mail)
  Wymaganie wstępne: Dzień 6 (Gmail połączony, prompty przetestowane), Dzień 1 (pola ClickUp istnieją)
  Rezultat: Scenariusz Email Intake zbudowany

Dzień 8: Testowanie scenariusza Email Intake
  Zadania:
    1. Wyślij testowe e-maile na każdy adres brandowy z zewnętrznego konta
    2. Obejmij wszystkie 5 przypadków testowych z Dnia 6
    3. Zweryfikuj tworzenie taska ClickUp — sprawdź wszystkie Custom Fields
    4. Zweryfikuj konwencję nazewniczą taska z danymi wyekstrahowanymi przez AI
    5. Zweryfikuj, że treść e-maila jest zachowana w opisie taska
    6. Przetestuj fallback przy awarii AI: tymczasowo złam klucz API, wyślij e-mail, zweryfikuj fallback task
    7. Przetestuj nakładanie etykiety Gmail przy błędzie
    8. Napraw wszelkie wykryte problemy
  Wymaganie wstępne: Dzień 7
  Rezultat: Scenariusz Email Intake zwalidowany

Dzień 9: Uruchomienie Email Intake na produkcji + monitoring
  Zadania:
    1. Włącz scenariusz Email Intake z polling interval 15 minut (ostrożny start)
    2. Monitoruj pierwsze 10 rzeczywistych zapytań mailowych przetworzonych przez system
    3. Porównaj jakość ekstrakcji AI z ręcznym czytaniem tych samych e-maili
    4. Sprawdź fałszywe alarmy: zweryfikuj, że żadne wewnętrzne/niezapytaniowe e-maile nie tworzą tasków
    5. Dodaj reguły filtrowania Gmail jeśli potrzeba, żeby wykluczyć znanych nadawców nie-zapytań
    6. Dostosuj polling interval w oparciu o wolumen (można skrócić do 5 minut jeśli działa stabilnie)
  Wymaganie wstępne: Dzień 8 (testy zaliczone)
  Rezultat: Email Intake uruchomiony na produkcji
```

### Tydzień 3: Stabilizacja + Przekazanie

```
Dzień 10: Realizacja pełnej listy smoke testów (Sekcja 7)
  Zadania:
    1. Wykonaj wszystkie przypadki smoke testów z Sekcji 7
    2. Udokumentuj wyniki: zaliczone/niezaliczone dla każdego przypadku testowego
    3. Napraw wszelkie niepowodzenia
    4. Powtórz niezaliczone testy po naprawkach
  Wymaganie wstępne: Dzień 5 + Dzień 9 (oba scenariusze na produkcji)
  Rezultat: Wszystkie smoke testy zaliczone

Dzień 11: Omówienie z zespołem sprzedaży + zebranie feedbacku
  Zadania:
    1. Zaprezentuj nowe widoki (NOWE - PRIORYTET, NOWE - KOMPLETNOŚĆ) zespołowi sprzedaży
    2. Wyjaśnij nowe pola: KLASYFIKACJA_AI, KOMPLETNOŚĆ, AI_PODSUMOWANIE, AI_CONFIDENCE
    3. Pokaż, jak filtrować/sortować według tych pól
    4. Wyjaśnij „Do weryfikacji" i co oznacza
    5. Zbierz feedback: co jest użyteczne, co jest mylące, czego brakuje
    6. Stwórz prostą jednostronicową instrukcję w WIKI PRACOWNIKA wyjaśniającą nowe pola
  Wymaganie wstępne: Dzień 10
  Rezultat: Zespół przeszkolony, feedback udokumentowany

Dzień 12: Dopracowanie + dokumentacja
  Zadania:
    1. Wdrożyć poprawki o najwyższym priorytecie na podstawie feedbacku z Dnia 11
    2. Przejrzeć dokładność klasyfikacji AI na 20+ rzeczywistych leadach przetworzonych w ostatnim tygodniu
    3. Dostosować prompty AI jeśli dokładność klasyfikacji jest poniżej 80%
    4. Udokumentować oba scenariusze Make: zrzuty ekranu konfiguracji każdego modułu, zapisz w WIKI PRACOWNIKA lub współdzielonym folderze
    5. Udokumentować procedurę manualnego fallbacku: „Jeśli Make nie działa, jak ręcznie stworzyć task ze wszystkimi wymaganymi polami"
    6. Napisać krótką notatkę planistyczną Phase 2 (na podstawie sekcji Odłożone poniżej)
  Wymaganie wstępne: Dzień 11
  Rezultat: System udokumentowany, poprawki wdrożone
```

**Szacowany łączny czas inżynierski: 12 dni roboczych (2,5 tygodnia)**

---

## Sekcja 7 — Lista Smoke Testów

```
Test 1: Kompletne zapytanie z formularza
  Akcja: Wyślij formularz na partyboat.fun ze wszystkimi wypełnionymi polami:
    - Miasto: w Krakowie
    - Rodzaj imprezy: Wieczór kawalerski
    - Pakiet: PARTY
    - Imię i Nazwisko: Jan Testowy
    - Email: jan.testowy@test.pl
    - Telefon: 500100200
    - Data: 2026-05-15
    - Godzina: 20:00
    - Liczba osób: 15
    - Opis: Wieczór kawalerski na statku, planujemy DJ i catering
  Oczekiwany rezultat:
    - Task stworzony w NOWE ZAPYTANIA
    - Tytuł: "15-05-2026 | KRAKÓW | BOAT | Jan Testowy"
    - MIASTO: KRAKÓW
    - USŁUGA: BOAT
    - TYP IMPREZY: Wieczór kawalerski
    - PAKIET: PARTY
    - POZYSKANIE: PARTYBOAT.FUN
    - ŹRÓDŁO_KANAŁ: Formularz
    - KOMPLETNOŚĆ: 100
    - KLASYFIKACJA_AI: Gorący lead
    - AI_CONFIDENCE: Wysoka
    - AI_PODSUMOWANIE: niepusty tekst po polsku
    - EMAIL: jan.testowy@test.pl
    - TELEFON: 500100200
    - Opis zawiera oryginalne dane formularza
  Warunek zaliczenia: Wszystkie pola zgodne z oczekiwanymi wartościami. Task widoczny w widoku NOWE - PRIORYTET.
```

```
Test 2: Częściowe zapytanie z formularza (brak daty i miasta)
  Akcja: Wyślij formularz na partytram.fun z:
    - Miasto: (pozostaw domyślne / niezaznaczone)
    - Rodzaj imprezy: Urodziny
    - Pakiet: (pozostaw domyślne)
    - Imię i Nazwisko: Anna Częściowa
    - Email: anna@test.pl
    - Telefon: 600200300
    - Data: (pozostaw puste jeśli możliwe, lub wpisz placeholder)
    - Godzina: (pozostaw puste)
    - Liczba osób: 8
    - Opis: Chciałabym urodziny na tramwaju
  Oczekiwany rezultat:
    - Task stworzony z tytułem zawierającym "BRAK DATY | WYBIERZ MIASTO | TRAM | Anna Częściowa"
    - KOMPLETNOŚĆ: między 50-75 (część pól brakuje)
    - KLASYFIKACJA_AI: Niekompletny lead
    - USŁUGA: TRAM (z domeny)
    - ŹRÓDŁO_KANAŁ: Formularz
  Warunek zaliczenia: Task stworzony, klasyfikacja odzwierciedla niekompletność, wskaźnik kompletności jest poniżej 100.
```

```
Test 3: E-mail — kompletne zapytanie
  Akcja: Wyślij e-mail z zewnętrznego konta na kontakt@partyboat.fun z:
    Temat: "Wieczór panieński Wrocław"
    Treść: "Dzień dobry, chciałabym zorganizować wieczór panieński na statku we Wrocławiu dnia 20 czerwca 2026. Będzie nas 12 osób. Interesuje nas pakiet LUX. Proszę o kontakt: 700800900. Pozdrawiam, Katarzyna Nowak"
  Oczekiwany rezultat:
    - Task stworzony w NOWE ZAPYTANIA
    - Tytuł: "20-06-2026 | WROCŁAW | BOAT | Katarzyna Nowak"
    - MIASTO: WROCŁAW
    - USŁUGA: BOAT
    - TYP IMPREZY: Wieczór panieński
    - PAKIET: LUX
    - ILOŚĆ OSÓB: 12
    - TELEFON: 700800900
    - EMAIL: adres e-mail nadawcy
    - ŹRÓDŁO_KANAŁ: Email
    - KOMPLETNOŚĆ: 100
    - KLASYFIKACJA_AI: Gorący lead
    - Opis zawiera pełną treść e-maila
  Warunek zaliczenia: Wszystkie wyekstrahowane pola zgodne z treścią e-maila. Żadne pole nie zostało wymyślone przez AI.
```

```
Test 4: E-mail — spam / nieistotna wiadomość
  Akcja: Wyślij e-mail z zewnętrznego konta na kontakt@partytram.fun z:
    Temat: "Oferta współpracy marketingowej"
    Treść: "Szanowni Państwo, nasza firma oferuje usługi pozycjonowania stron internetowych. Gwarantujemy top 3 w Google w 30 dni. Zapraszam do kontaktu."
  Oczekiwany rezultat:
    - Task stworzony w NOWE ZAPYTANIA
    - KLASYFIKACJA_AI: Spam
    - AI_CONFIDENCE: Wysoka
    - KOMPLETNOŚĆ: niska (wypełniony tylko EMAIL)
    - ŹRÓDŁO_KANAŁ: Email
    - Większość pól eventowych pusta (MIASTO, DATA EVENTU, TYP IMPREZY itp.)
  Warunek zaliczenia: AI poprawnie klasyfikuje jako spam. Żadne pola eventowe nie są wymyślone na podstawie tekstu marketingowego.
```

```
Test 5: E-mail — ogólne/vague zapytanie
  Akcja: Wyślij e-mail z zewnętrznego konta na kontakt@busparty.fun z:
    Temat: "Pytanie"
    Treść: "Hej, ile kosztuje wynajęcie busa? Pozdrawiam Tomek"
  Oczekiwany rezultat:
    - Task stworzony w NOWE ZAPYTANIA
    - Tytuł: "BRAK DATY | WYBIERZ MIASTO | BUS | Tomek"
    - KLASYFIKACJA_AI: Zapytanie ogólne
    - KOMPLETNOŚĆ: niska (25 lub mniej — wypełnione tylko imię i e-mail)
    - USŁUGA: BUS (z brandu domeny)
    - ŹRÓDŁO_KANAŁ: Email
    - Większość pól null/pusta
  Warunek zaliczenia: AI klasyfikuje jako zapytanie ogólne, nie jako lead. USŁUGA poprawnie wywiedzione z domeny brandowej.
```

```
Test 6: Fallback przy awarii AI (formularz)
  Akcja: Tymczasowo ustaw klucz Anthropic API w Make na nieprawidłową wartość. Wyślij formularz na dowolnej domenie z kompletnymi danymi.
  Oczekiwany rezultat:
    - Task stworzony w NOWE ZAPYTANIA (awaria AI nie blokuje tworzenia taska)
    - KLASYFIKACJA_AI: Do weryfikacji
    - AI_CONFIDENCE: Niska
    - AI_PODSUMOWANIE: puste
    - Wszystkie pola nie-AI (MIASTO, USŁUGA, TELEFON, EMAIL itp.) normalnie wypełnione z danych formularza
    - KOMPLETNOŚĆ obliczona normalnie
    - Priority: Urgent (ustawione przez automatyzację ClickUp)
  Warunek zaliczenia: Task istnieje ze wszystkimi polami pochodzącymi z formularza. Awaria AI jest obsłużona elegancko. Flaga priorytetu jest widoczna.
  Po teście: Natychmiast przywróć prawidłowy klucz API.
```

```
Test 7: Fallback przy awarii AI (e-mail)
  Akcja: Tymczasowo ustaw klucz Anthropic API w Make na nieprawidłową wartość. Wyślij testowy e-mail na dowolny adres brandowy.
  Oczekiwany rezultat:
    - Task stworzony w NOWE ZAPYTANIA
    - KLASYFIKACJA_AI: Do weryfikacji
    - AI_CONFIDENCE: Niska
    - EMAIL: adres e-mail nadawcy (z metadanych, nie od AI)
    - Opis: zawiera treść e-maila
    - Tytuł: "BRAK DATY | WYBIERZ MIASTO | [USŁUGA z brandu] | [imię nadawcy lub e-mail]"
    - Priority: Urgent
  Warunek zaliczenia: Task istnieje z metadanymi e-maila. Żadne dane nie utracone mimo awarii AI.
  Po teście: Natychmiast przywróć prawidłowy klucz API.
```

---

## Odłożone na Phase 2

Poniższe funkcje były rozważane podczas projektowania i zostały celowo wykluczone, aby zmieścić się w ograniczeniu 12 dni dla jednego inżyniera. Każdy element został oceniony przez panel ekspertów.

| Funkcja | Dlaczego odłożona | Ekspert, który zgłosił |
|---------|-------------------|----------------------|
| **Generowanie wersji roboczych odpowiedzi (draft replies)** | Dotyczy komunikacji z klientami. Wymaga dopracowania promptów, kalibracji tonu i budowania zaufania zespołu sprzedaży. Zbyt ryzykowne dla Phase 1. | Aleksander (AI): „Drafty trafiające do klientów wymagają tygodni walidacji." Karolina (Sprzedaż): „Zespół musi najpierw zaufać klasyfikacji." |
| **Automatyczny routing leadów do handlowców** | Wymaga uzgodnienia reguł terytorialnych, fallbacku dla zapytań z wielu miast i obsługi nowych handlowców. To zmiana procesowa, nie tylko techniczna. | Karolina (Sprzedaż): „Przypisanie to decyzja zespołu. Nie automatyzuj zanim zespół o to nie poprosi." |
| **Deduplikacja leadów** | Klient może wysłać formularz I e-mail. Deduplikacja wymaga fuzzy matching po e-mailu/telefonie. Dodaje złożoność do obu scenariuszy. | Tomasz (Make): „Dedup wymaga Data Store, logiki lookup i strategii merge. To osobny projekt." |
| **Przypomnienia o follow-upach** | Automatyczne przypomnienia dla niekompletnych leadów. Użyteczne, ale wymaga zdefiniowania reguł SLA, które jeszcze nie istnieją. | Karolina (Sprzedaż): „Najpierw zdefiniuj proces follow-upu ręcznie. Automatyzuj gdy rytm jest już ustalony." |
| **Automatyzacja workflow ZLECENIA** | Operacyjny przepływ po-sprzedażowy. Osobna domena z innymi potrzebami danych. | Marta (ClickUp): „ZLECENIA to inny proces z innymi potrzebami danych. Nie mieszaj z intake'em." |
| **Dashboard analityczny** | Metryki wolumenu leadów, efektywności kanałów, dokładności klasyfikacji. Wartościowe, ale niezbędne dopiero po ustabilizowaniu intake'u. | Wszyscy: „Najpierw uruchom przepływ danych. Dashboard jest łatwy do dodania gdy dane są czyste." |
| **Wsparcie wielojęzyczne** | Zapytania po angielsku i w innych językach istnieją, ale stanowią mniejszość. Obecne prompty obsługują angielski wystarczająco dla klasyfikacji, ale jakość ekstrakcji może być zmienna. | Aleksander (AI): „Prompty i tak działają dla angielskiego. Formalne wsparcie wielojęzyczne może poczekać na dane o wolumenie." |

---

*Koniec przewodnika wdrożeniowego. Łączny szacowany nakład: 12 dni roboczych rozłożonych na 3 tygodnie.*
