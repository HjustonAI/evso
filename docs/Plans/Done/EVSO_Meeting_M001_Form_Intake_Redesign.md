# Meeting Output: Redesign scenariusza Form Intake — ekstrakcja AI z wolnego tekstu + migracja na createAMessage

**Date**: 2026-04-02
**Format**: Structured Workshop
**Team Charter**: EVSO_DevTeam v1.0
**Meeting ID**: M-001

---

## Meeting Goal

**As stated**: „Przeanalizuj obecny blueprint obsługi formularza. Klienci czasem wypełniają pola formularza, a czasem olewają je i piszą wszystko w polu 'Opisz swój pomysł' (client_txt). Chcemy, aby LLM wyciągał z treści informacje o nieuzupełnionych polach. Przejdźmy z Simple Text Prompt na moduł API (createAMessage). Zwołaj zebranie EVSO_DevTeam i opracuj idealną wersję scenariusza Make."

**As understood**: Zespół ma zaprojektować docelową wersję scenariusza Make.com „Form Intake", która:
1. Rozpoznaje sytuację, gdy klient pominął pola formularza, ale podał te informacje w wolnym tekście (pole `textarea_1` / `client_txt`)
2. Wysyła dane do Claude Haiku przez moduł `anthropic-claude:createAMessage` (zamiast `simpleTextPrompt`), z system promptem i schematem JSON
3. LLM zwraca zarówno klasyfikację (jak dotychczas), jak i wyekstrahowane dane z wolnego tekstu
4. Scenariusz „mergeuje" dane: pola formularza mają priorytet, ale jeśli pole jest puste, używa wartości wyekstrahowanej przez AI
5. Completeness score jest przeliczany PO merge'u (uwzględniając dane z AI)

**Target outcome**: Action plan — pełna specyfikacja docelowego scenariusza moduł po module, gotowa do ręcznego wdrożenia na koncie klienta.

---

## Context Summary

### What Is Known

- [FACT] Obecny blueprint: 7 modułów — Webhook(1) → SetVars(42: mapowanie pól Forminator) → SetVars(43: brand, service, completeness, ClickUp indeksy) → Anthropic simpleTextPrompt(44) → JSON Parse(45) → SetVars(46: klasyfikacja→ClickUp, task title) → ClickUp createTaskInList(47)
- [FACT] Formularz Forminator wysyła: `select_2` (miasto), `select_3` (rodzaj imprezy), `select_1` (pakiet), `name_1` (imię), `email_1`, `phone_1`, `number_1` (osoby), `textarea_1` (opis/client_txt), `name_2` (data), `name_3` (godzina)
- [FACT] Miasta przychodzą w mianowniku (np. „Bydgoszcz"), nie w narzędniku — blueprint mapuje `upper(trim())` na indeksy ClickUp
- [FACT] Moduł 43 ma błąd referencji do nieistniejącego modułu ID 7 (`clickup_usluga` odwołuje się do `7.service_type` zamiast `43.service_type` — sam siebie referencjonuje)
- [FACT] `simpleTextPrompt` nie obsługuje system promptu ani output schema — wszystko idzie w jednym polu `textPrompt`
- [FACT] `createAMessage` wymaga connection do Anthropic, wspiera `system` prompt, `messages[]`, `max_tokens`, i structured output
- [FACT] ClickUp Custom Fields (UUID → label): MIASTO, PAKIET, POZYSKANIE, TELEFON, TYP IMPREZY, USŁUGA, GODZINA, DATA EVENTU, EMAIL, ILOŚĆ OSÓB, JEDNOSTKA
- [FACT] Brakuje w ClickUp 5 nowych Custom Fields z Phase 1: ŹRÓDŁO_KANAŁ, KLASYFIKACJA_AI, AI_PODSUMOWANIE, KOMPLETNOŚĆ, AI_CONFIDENCE

### What Is Uncertain

| Unknown | Impact | Assumption |
|---------|--------|------------|
| Czy Anthropic connection jest już skonfigurowane na koncie klienta w Make? | Bez connection moduł `createAMessage` nie zadziała | Zakładamy, że klient ma lub skonfiguruje API key Anthropic |
| Czy 5 nowych Custom Fields zostało dodanych do ClickUp? | Brak pól = dane AI nie zostaną zapisane | Zakładamy, że trzeba je dodać przed wdrożeniem |
| Jak często klienci piszą wszystko w wolnym tekście vs. wypełniają pola? | Wpływa na to, jak krytyczna jest ekstrakcja AI | Zakładamy ~30-50% przypadków na podstawie doświadczenia operacyjnego |
| Limit tokenów wejściowych na `createAMessage` w Make | Może obciąć długi opis klienta | Zakładamy 1024 tokens input wystarczy dla formularza |

### Constraints Applied

1. [USER-STATED] Praca ręczna na koncie klienta — nie używamy konektora Make przez API, tylko produkujemy specyfikację do wklejenia
2. [USER-STATED] Forminator jest źródłem formularza, nie WPForms
3. [IMPLICIT] Koszt: Claude Haiku (najtańszy model) — jedno wywołanie na formularz
4. [IMPLICIT] Zachowanie konwencji nazewnictwa tasków: `[DATA] | [MIASTO] | [USŁUGA] | [IMIĘ]`
5. [IMPLICIT] Fallback: jeśli AI nie odpowie, task i tak musi powstać

---

## Expert Participation

| Expert | Role | Participation Level | Primary Contribution Area |
|--------|------|-------------------|--------------------------|
| Tomasz Sowa | Make Automation Architect | Primary | Architektura scenariusza, kolejność modułów, merge logic, error handling |
| Aleksander Bryła | AI Systems & Reliability Architect | Primary | Prompt design, structured output, confidence gating, ekstrakcja z free text |
| Marta Kurek | ClickUp Systems Architect | Primary | Custom Fields mapping, data model, co trafia do ClickUp i w jakim formacie |
| Karolina Wysocka | Sales & Service Process Designer | Secondary | Walidacja: czy wynik jest użyteczny dla handlowców, co z priorytetyzacją |

---

## Discussion Summary

### Work Item 1: Architektura promptu AI — co LLM ma robić?

#### Aleksander Bryła — AI Systems & Reliability Architect

[RECOMMENDATION] Jedno wywołanie AI powinno realizować **trzy zadania naraz**, a nie trzy osobne wywołania:

1. **Klasyfikacja** (goracy_lead / niekompletny_lead / zapytanie_ogolne / spam)
2. **Ekstrakcja brakujących danych** z wolnego tekstu (miasto, data, liczba osób, rodzaj imprezy, godzina)
3. **Podsumowanie** (1-2 zdania po polsku)

[FACT] Rozdzielanie na osobne wywołania API mnożyłoby koszty ×3 i dodawało latencję. Claude Haiku radzi sobie z wielozadaniowymi promptami, o ile output jest strukturalny i jednoznaczny.

[RECOMMENDATION] Kluczowa zasada: **„Najpierw reguła, potem model"**. AI nie powinno wyciągać danych, które JUŻ są w polach formularza. Dlatego prompt musi dostawać informację, KTÓRE pola są puste — a AI wypełnia tylko te.

[RECOMMENDATION] Struktura odpowiedzi JSON:

```json
{
  "klasyfikacja": "goracy_lead|niekompletny_lead|zapytanie_ogolne|spam",
  "confidence": "wysoka|srednia|niska",
  "podsumowanie": "...",
  "ekstrakcja": {
    "miasto": null,
    "data_eventu": null,
    "godzina": null,
    "liczba_osob": null,
    "rodzaj_imprezy": null,
    "pakiet": null,
    "imie_nazwisko": null,
    "telefon": null,
    "email": null
  }
}
```

Pola w `ekstrakcja` powinny być `null` jeśli AI nie znalazło ich w tekście. Nie `""`, nie `"brak"` — dokładnie `null`. To pozwala downstream logic na proste: `if(extracted != null; extracted; "")`.

**Moderator**: Tomasz, jak to wpływa na logikę merge w Make?

#### Tomasz Sowa — Make Automation Architect

[INFERENCE] Null w JSON Parse Make daje pustą wartość, co jest wygodne. Mogę budować merge logic jako:

```
{{if(42.miasto != ""; 42.miasto; if(45.ekstrakcja.miasto != ""; 45.ekstrakcja.miasto; ""))}}
```

Priorytet: pole formularza → ekstrakcja AI → puste.

[RECOMMENDATION] Ale jest subtelność: w Make `null` z JSON Parse staje się pustym stringiem `""`. Nie mogę odróżnić „AI zwróciło null" od „AI zwróciło pusty string". Dlatego zgadzam się z Aleksandrem — **musimy w prompcie wymusić `null` dla braków**, a w Make traktować `""` jako brak.

**Moderator**: Aleksander, czy `createAMessage` w Make wspiera structured output / JSON mode?

#### Aleksander Bryła — AI Systems & Reliability Architect

[FACT] Moduł `createAMessage` w Make pozwala podać `system` prompt osobno od `messages`. To duża przewaga — system prompt definiuje rolę i reguły, user message zawiera dane. [INFERENCE] Make raczej nie wspiera natywnie Anthropic JSON mode (`response_format`), ale możemy wymusić JSON w system prompcie i stripować markdown fences w następnym module — tak jak robi to obecny blueprint w module 45.

[RECOMMENDATION] Lepsze rozwiązanie: w system prompcie dodać twarde reguły:
- „Zwróć wyłącznie czysty JSON"
- „Nie dodawaj żadnego tekstu przed ani po JSON"
- „Nie używaj markdown code fences"

I w JSON Parse module zostawić safety strip: `{{trim(replace(replace(44.result; "` ` `json"; ""); "` ` `"; ""))}}` — na wypadek gdyby model i tak owinął.

### Work Item 2: System prompt + user message — dokładna treść

#### Aleksander Bryła — AI Systems & Reliability Architect

[RECOMMENDATION] **System prompt** (stały, nie zmienia się per request):

```
Jesteś asystentem klasyfikacji i ekstrakcji danych dla firmy eventowej EVSO (tramwaje, statki, busy imprezowe).

ZADANIE:
1. Sklasyfikuj zapytanie klienta.
2. Wyciągnij z wolnego tekstu klienta brakujące dane (tylko te oznaczone jako PUSTE).
3. Napisz krótkie podsumowanie po polsku.

KLASYFIKACJA:
- goracy_lead: podane miasto + data + typ imprezy + kontakt (email lub telefon)
- niekompletny_lead: intencja eventowa jasna, ale brakuje kluczowych danych
- zapytanie_ogolne: pytanie o cennik, dostępność, ogólne info — nie konkretne zapytanie
- spam: nie dotyczy eventów, reklama, niezrozumiałe

CONFIDENCE:
- wysoka: klasyfikacja oczywista
- srednia: pewne niejasności
- niska: dane bardzo niepełne lub sprzeczne

EKSTRAKCJA:
- Wyciągaj TYLKO z pola "opis_klienta"
- Wypełniaj TYLKO pola oznaczone jako "[PUSTE]"
- Jeśli pole ma wartość z formularza, NIE nadpisuj
- Jeśli nie znalazłeś danej informacji w tekście, zwróć null
- miasto: zwróć w mianowniku, wielkimi literami (np. "KRAKÓW", "WROCŁAW")
- rodzaj_imprezy: dopasuj do: Kawalerski, Panieński, Urodziny, Firmówka, Inne
- liczba_osob: tylko liczba
- data_eventu: format YYYY-MM-DD jeśli możliwe
- godzina: format HH:MM jeśli możliwe

PODSUMOWANIE:
- po polsku, 1-2 zdania
- opisz czego klient szuka
- jeśli brakuje danych, zaznacz krótko

WAŻNE:
- Oceń dane całościowo (pola + opis)
- Jeśli klient pisze o imprezie na tramwaju/statku/busie, nie oznaczaj jako spam
- Zwróć WYŁĄCZNIE czysty JSON, bez markdown, bez komentarzy
- Klasyfikację rób na podstawie WSZYSTKICH danych (formularza + wyekstrahowanych)

FORMAT ODPOWIEDZI:
{"klasyfikacja":"...","confidence":"...","podsumowanie":"...","ekstrakcja":{"miasto":null,"data_eventu":null,"godzina":null,"liczba_osob":null,"rodzaj_imprezy":null,"pakiet":null,"imie_nazwisko":null,"telefon":null,"email":null}}
```

[RECOMMENDATION] **User message** (dynamiczny, budowany z danych formularza):

```
Dane z formularza:
- Miasto: {{42.miasto}} {{if(42.miasto = ""; "[PUSTE]"; "")}}
- Rodzaj imprezy: {{42.rodzaj_imprezy}} {{if(42.rodzaj_imprezy = ""; "[PUSTE]"; "")}}
- Pakiet: {{42.pakiet}} {{if(42.pakiet = ""; "[PUSTE]"; "")}}
- Imię i nazwisko: {{42.imie_nazwisko}} {{if(42.imie_nazwisko = ""; "[PUSTE]"; "")}}
- Email: {{42.email}} {{if(42.email = ""; "[PUSTE]"; "")}}
- Telefon: {{42.telefon}} {{if(42.telefon = ""; "[PUSTE]"; "")}}
- Liczba osób: {{42.liczba_osob}} {{if(42.liczba_osob = ""; "[PUSTE]"; "")}}
- Data eventu: {{42.data_eventu}} {{if(42.data_eventu = ""; "[PUSTE]"; "")}}
- Godzina: {{42.godzina}} {{if(42.godzina = ""; "[PUSTE]"; "")}}
- Źródło: {{43.source_brand}}

Opis klienta (wolny tekst):
{{42.client_txt}}
```

**Moderator**: Karolina, czy ta lista pól pokrywa to, co handlowcy potrzebują do obsługi leada?

#### Karolina Wysocka — Sales & Service Process Designer

[FACT] Handlowcy potrzebują minimum: miasta, daty, liczby osób i typu imprezy, żeby przygotować wstępną ofertę. Email lub telefon do kontaktu.

[RECOMMENDATION] Lista jest kompletna. Jedyne co dodam: prompt powinien rozumieć, że klient może pisać kolokwialnie — „jedziemy w 20 osób na kawa" oznacza wieczór kawalerski, 20 osób. „Chcemy w piątek" wymaga kontekstu daty. Jeśli AI nie może ustalić dokładnej daty (np. „w przyszły piątek"), lepiej niech zwróci `null` niż zgaduje.

[RECOMMENDATION] W podsumowaniu AI powinno zaznaczać, co udało się wyciągnąć z tekstu — to da handlowcowi sygnał, żeby zweryfikował te dane. Np. „Klient szuka tramwaju na kawalerski w Krakowie. Miasto i typ imprezy wyciągnięte z opisu — do weryfikacji."

**Moderator**: Aleksander, czy dodanie takiego rozróżnienia w podsumowaniu jest realistyczne bez nadmiernego komplikowania promptu?

#### Aleksander Bryła — AI Systems & Reliability Architect

[RECOMMENDATION] Tak, ale nie komplikujmy formatu JSON. Dodajmy jedną linię do instrukcji podsumowania: „Jeśli dane zostały wyciągnięte z wolnego tekstu, dodaj na końcu: (dane z opisu — do weryfikacji)". To wystarczy. Model Haiku da radę.

### Work Item 3: Architektura modułów Make — docelowy flow

#### Tomasz Sowa — Make Automation Architect

[RECOMMENDATION] Docelowy scenariusz — **8 modułów** (obecnie jest 7, ale przeprojektowujemy):

```
1. Webhook (ID 1) — bez zmian
2. SetVars "Mapowanie pól" (ID 42) — bez zmian (mapuje Forminator → zmienne)
3. SetVars "Brand + Service" (ID 43) — POPRAWKA: naprawić referencję do 7.service_type → użyć własnej zmiennej
4. Anthropic createAMessage (ID 44) — ZMIANA: z simpleTextPrompt na createAMessage
5. JSON Parse (ID 45) — ZMIANA: nowa data structure z polem "ekstrakcja"
6. SetVars "Merge + Completeness" (NOWY / zmieniony 46) — ZMIANA: merge danych, przeliczenie completeness PO ekstrakcji
7. SetVars "ClickUp mapping" (NOWY / zmieniony) — mapowanie na indeksy ClickUp
8. ClickUp createTaskInList (ID 47) — ZMIANA: dodanie 5 nowych Custom Fields
```

[RECOMMENDATION] Kluczowe zmiany moduł po module:

**Moduł 3 (ID 43) — POPRAWKA:**
- Linia `clickup_usluga`: zmienić `{{if(7.service_type = "BOAT"...}}` na `{{if(43.service_type = "BOAT"...}}`  (samoreferencja zamiast nieistniejącego modułu 7)

Ale czekaj — to dalej będzie problem, bo moduł nie może referencjonować sam siebie w Make.

**Moderator**: To istotne. Tomasz, jak to rozwiązać?

#### Tomasz Sowa — Make Automation Architect

[FACT] W Make, SetMultipleVariables nie może odwoływać się do swoich WŁASNYCH zmiennych ustawianych w tym samym kroku — zmienne są dostępne dopiero po zakończeniu modułu.

[RECOMMENDATION] Rozwiązanie: **rozbić moduł 43 na dwa osobne moduły** lub przenieść `clickup_usluga` do późniejszego modułu (np. 46), który już widzi `43.service_type`. Lepsze rozwiązanie:

Moduł 43 ustawia: `source_brand`, `service_type`, `completeness_score` (wstępny), `task_date_display`.

Nowy moduł (po JSON Parse) ustawia: `clickup_miasto`, `clickup_pakiet`, `clickup_pozyskanie`, `clickup_typ_imprezy`, `clickup_usluga` — czyli wszystkie mapowania na indeksy ClickUp, bazując na **zmerge'owanych danych** (formularz + AI).

To jednocześnie rozwiązuje problem z `7.service_type` i pozwala na completeness przeliczony po ekstrakcji.

#### Marta Kurek — ClickUp Systems Architect

[RECOMMENDATION] Zgadzam się. Proponuję następującą strukturę danych w ClickUp Create Task:

**Istniejące Custom Fields (z merge'owanymi danymi):**
- MIASTO → `merged_clickup_miasto` (index)
- PAKIET → `merged_clickup_pakiet` (index)
- TYP IMPREZY → `merged_clickup_typ_imprezy` (index)
- USŁUGA → `merged_clickup_usluga` (index)
- POZYSKANIE → `clickup_pozyskanie` (bez zmian, z source_url)
- TELEFON → `merged_telefon`
- EMAIL → `merged_email`
- DATA EVENTU → `merged_data_eventu`
- GODZINA EVENTU → `merged_godzina`
- ILOŚĆ OSÓB → `merged_liczba_osob`

**Nowe Custom Fields (do dodania w ClickUp):**
- ŹRÓDŁO_KANAŁ (Dropdown: Formularz, Email) → „Formularz"
- KLASYFIKACJA_AI (Dropdown: Gorący lead, Niekompletny lead, Zapytanie ogólne, Spam, Do weryfikacji)
- AI_PODSUMOWANIE (Long Text)
- KOMPLETNOŚĆ (Number, 0-100)
- AI_CONFIDENCE (Dropdown: Wysoka, Średnia, Niska)

[FACT] UUID dla nowych pól nie znamy — trzeba je dodać w ClickUp, a potem uzupełnić w module ClickUp.

**Moderator**: Karolina, jaka wartość ma przeliczanie completeness PO ekstrakcji AI?

#### Karolina Wysocka — Sales & Service Process Designer

[RECOMMENDATION] Ogromna. Dziś jeśli klient wpisze w tekście „Kraków, 15 osób, kawalerski, 20 czerwca" a zostawi pola puste, completeness wyjdzie ~25% mimo że mamy wszystkie potrzebne dane. Po merge z AI completeness powinien wyjść ~88%. To zmienia priorytetyzację tego leada z „niekompletny do uzupełnienia" na „gorący do ofertowania". Handlowiec nie musi tracić czasu na follow-up.

[FACT] To jest **główny problem operacyjny** — handlowcy dzwonią do klientów prosząc o dane, które klient już podał, tyle że w tekście.

### Work Item 4: Merge logic i przeliczanie completeness

#### Tomasz Sowa — Make Automation Architect

[RECOMMENDATION] **Moduł „Merge + Completeness"** (po JSON Parse):

```
Zmienne merge:
  merged_miasto: {{if(42.miasto != ""; upper(trim(42.miasto)); if(45.ekstrakcja.miasto != ""; upper(trim(45.ekstrakcja.miasto)); ""))}}
  merged_rodzaj_imprezy: {{if(42.rodzaj_imprezy != ""; 42.rodzaj_imprezy; if(45.ekstrakcja.rodzaj_imprezy != ""; 45.ekstrakcja.rodzaj_imprezy; ""))}}
  merged_pakiet: {{if(42.pakiet != ""; 42.pakiet; if(45.ekstrakcja.pakiet != ""; 45.ekstrakcja.pakiet; ""))}}
  merged_imie: {{if(42.imie_nazwisko != ""; 42.imie_nazwisko; if(45.ekstrakcja.imie_nazwisko != ""; 45.ekstrakcja.imie_nazwisko; ""))}}
  merged_email: {{if(42.email != ""; 42.email; if(45.ekstrakcja.email != ""; 45.ekstrakcja.email; ""))}}
  merged_telefon: {{if(42.telefon != ""; 42.telefon; if(45.ekstrakcja.telefon != ""; 45.ekstrakcja.telefon; ""))}}
  merged_liczba_osob: {{if(42.liczba_osob != ""; 42.liczba_osob; if(45.ekstrakcja.liczba_osob != ""; 45.ekstrakcja.liczba_osob; ""))}}
  merged_data: {{if(42.data_eventu != ""; 42.data_eventu; if(45.ekstrakcja.data_eventu != ""; 45.ekstrakcja.data_eventu; ""))}}
  merged_godzina: {{if(42.godzina != ""; 42.godzina; if(45.ekstrakcja.godzina != ""; 45.ekstrakcja.godzina; ""))}}
```

```
Completeness (po merge):
  merged_completeness: {{round(((if(merged_imie != ""; 1; 0)) + (if(merged_email != ""; 1; 0)) + (if(merged_telefon != ""; 1; 0)) + (if(merged_miasto != ""; 1; 0)) + (if(merged_data != ""; 1; 0)) + (if(merged_godzina != ""; 1; 0)) + (if(merged_liczba_osob != ""; 1; 0)) + (if(merged_rodzaj_imprezy != ""; 1; 0))) / 8 * 100)}}
```

[FACT] Uwaga: w Make, zmienne ustawione w tym samym module SetMultipleVariables nie mogą referencjonować się nawzajem w wartościach. Dlatego `merged_completeness` nie może być w tym samym module co `merged_miasto` itp. — musi być w osobnym module, albo completeness liczy się bezpośrednio z if-chain bez pośrednich zmiennych.

[RECOMMENDATION] Praktyczne rozwiązanie: **obliczyć completeness inline** bez odwoływania się do zmiennych merge. Zamiast `if(merged_miasto != "")` użyć `if(42.miasto != "" OR 45.ekstrakcja.miasto != ""; 1; 0)`. Wtedy wszystko zmieści się w jednym module.

Albo — prostsze — **rozbić na 2 moduły**: jeden z merge, drugi z completeness + ClickUp indeksy (bo ten drugi potrzebuje wartości merge).

Rekomenduję **2 moduły**:
- Moduł A: merge (9 zmiennych merged_*)
- Moduł B: completeness + clickup_* indeksy (korzysta z merged_*)

#### Aleksander Bryła — AI Systems & Reliability Architect

[RECOMMENDATION] Dodatkowy safety net: **jeśli moduł Anthropic zwróci błąd** (timeout, 500, rate limit), scenariusz powinien kontynuować z danymi formularza BEZ ekstrakcji. Sugeruję Error Handler na module Anthropic z fallback path, który ustawia domyślne puste wartości ekstrakcji i klasyfikację „Do weryfikacji".

[FACT] W Make Error Handler na module można ustawić „Resume" z domyślnymi wartościami lub „Ignore" + Continue. Lepsze jest Resume z jawnym ustawieniem zmiennych fallback.

**Moderator**: Marta, czy „Do weryfikacji" jako klasyfikacja w ClickUp jest operacyjnie sensowne?

#### Marta Kurek — ClickUp Systems Architect

[FACT] Tak. Phase 1 guide przewiduje automation: gdy KLASYFIKACJA_AI = „Do weryfikacji" → Priority = Urgent. Handlowiec widzi czerwoną flagę i wie, że AI nie mogło przetworzyć tego leada.

### Work Item 5: Moduł createAMessage — konfiguracja techniczna

#### Tomasz Sowa — Make Automation Architect

[RECOMMENDATION] Konfiguracja modułu `anthropic-claude:createAMessage` (ID 44):

```
Connection: [Anthropic API — do skonfigurowania na koncie klienta]
Model: claude-haiku-4-5
System Prompt: [pełny system prompt z Work Item 2]
Messages:
  - Role: user
    Content: [dynamiczny user message z Work Item 2]
Max Tokens: 500 (zwiększony z 300 — ekstrakcja wymaga więcej tokenów)
```

[FACT] Moduł `createAMessage` w Make daje output w formacie: `result` (string z odpowiedzią) lub potencjalnie `content[].text`. Trzeba zweryfikować dokładną ścieżkę outputu po podpięciu — może się różnić od `simpleTextPrompt.result`.

[RECOMMENDATION] W JSON Parse (moduł 45) zaktualizować:
- Data structure: dodać pole `ekstrakcja` (object z 9 polami tekstowymi, nullable)
- JSON string: `{{trim(replace(replace(44.result; "` ` `json"; ""); "` ` `"; ""))}}`
  (lub odpowiednia ścieżka jeśli createAMessage daje inny output path)

#### Aleksander Bryła — AI Systems & Reliability Architect

[RECOMMENDATION] Max tokens 500 to OK. Typowa odpowiedź będzie ~200-350 tokenów. Zostawiamy margines na dłuższe podsumowania.

[FACT] Claude Haiku 4.5 kosztuje $0.25/1M input tokens, $1.25/1M output tokens. Przy ~500 tokens input + ~300 tokens output per request, koszt to ~$0.0005 per lead. Przy 100 leads/tydzień = ~$0.05/tydzień. Pomijalny.

---

## Disagreements

### Disagreement: Completeness score — przed czy po merge?

- **Parties**: Tomasz Sowa vs. Aleksander Bryła (minor)
- **Type**: Values
- **Position A** (Tomasz): Warto zachować TAKŻE wstępny completeness (przed AI) jako zmienną wewnętrzną — pozwala porównać, ile AI dołożył
- **Position B** (Aleksander): Jeden score wystarczy — po merge. Dwa score'y komplikują logikę i wymagają dodatkowego Custom Field w ClickUp
- **What would change minds**: Gdyby biznes chciał mierzyć jakość formularzy (ile klienci wypełnia vs. ile AI musi dociągać) — wtedy pre-AI score ma wartość analityczną
- **Resolution**: Kompromis — w ClickUp zapisujemy tylko POST-MERGE completeness. Wstępny completeness z modułu 43 zostawiamy jako zmienną wewnętrzną (nie kasujemy), ale nie mapujemy do ClickUp. W Phase 2 analytics dashboard może to wykorzystać.

### Disagreement: Czy AI powinno ekstrakcją nadpisywać puste select (dropdown) pola?

- **Parties**: Marta Kurek vs. Karolina Wysocka
- **Type**: Values
- **Position A** (Marta): AI wyciąga tekst np. „kawalerski", ale dropdowny w ClickUp mają konkretne wartości. Jeśli AI zwróci „Wieczór kawalerski" a dropdown ma „Kawalerski" — nie zmapuje się. Ryzyko bad data.
- **Position B** (Karolina): Lepiej mieć przybliżoną wartość niż brak. Handlowiec widzi AI_CONFIDENCE i AI_PODSUMOWANIE, może zweryfikować.
- **What would change minds**: Testowanie — jeśli AI zwraca wartości dokładnie w formacie dropdownu, problem nie istnieje
- **Resolution**: Prompt wymusza format wartości dopasowany do ClickUp dropdownów (Aleksander dodał to w instrukcji ekstrakcji). W merge logic, mapowanie AI-wartości na indeksy ClickUp używa tej samej tabeli co mapowanie formularza. Jeśli AI zwróci wartość, która nie pasuje do żadnej opcji — indeks = „" (puste), a completeness nie liczy tego pola jako wypełnione.

---

## Convergence

### Synthesis

Zespół zaprojektował **9-modułowy scenariusz** (zamiast obecnych 7), który rozwiązuje oba problemy: ekstrakcję danych z wolnego tekstu i migrację na `createAMessage`. Kluczowa innowacja: prompt oznacza puste pola tagiem `[PUSTE]`, AI wyciąga tylko brakujące dane, a moduł merge łączy oba źródła z priorytetem formularza. Completeness jest przeliczany po merge, co eliminuje problem fałszywie niskich score'ów dla klientów, którzy piszą w tekście zamiast wypełniać pola.

Trade-off: scenariusz ma 2 moduły więcej, ale zyskuje: lepszą jakość danych, realistyczny completeness, system prompt + structured output, i error handling.

### Decision Records

| Field | Content |
|-------|---------|
| **ID** | D-001 |
| **Decision** | Jedno wywołanie AI realizuje 3 zadania: klasyfikacja + ekstrakcja + podsumowanie |
| **Rationale** | Koszt ×1 vs ×3, latencja ×1 vs ×3. Haiku radzi sobie z multitask prompt. |
| **Dissent** | None |
| **Confidence** | High — testowane podejście w podobnych use case'ach |
| **Revisit if** | Jakość ekstrakcji w testach < 80% accuracy |

| Field | Content |
|-------|---------|
| **ID** | D-002 |
| **Decision** | Prompt oznacza puste pola tagiem [PUSTE], AI wyciąga dane TYLKO z pól oznaczonych jako puste |
| **Rationale** | Zapobiega nadpisywaniu poprawnych danych z formularza. AI nie zgaduje tam gdzie dane istnieją. |
| **Dissent** | None |
| **Confidence** | High — zasada „najpierw reguła, potem model" |
| **Revisit if** | Klienci wpisują sprzeczne dane (inne miasto w polu, inne w tekście) — wtedy potrzebna deduplication logic |

| Field | Content |
|-------|---------|
| **ID** | D-003 |
| **Decision** | Migracja z simpleTextPrompt na createAMessage z osobnym system promptem |
| **Rationale** | System prompt umożliwia stabilną separację instrukcji od danych. Łatwiejsze utrzymanie i debugowanie. |
| **Dissent** | None |
| **Confidence** | High — createAMessage to zalecanego podejście |
| **Revisit if** | Connection Anthropic w Make nie działa na koncie klienta — wtedy fallback na simpleTextPrompt z uproszczonym promptem |

| Field | Content |
|-------|---------|
| **ID** | D-004 |
| **Decision** | Completeness score przeliczany PO merge (formularz + AI), wstępny completeness zachowany jako zmienna wewnętrzna |
| **Rationale** | Post-merge completeness daje realistyczny obraz gotowości leada. Eliminuje fałszywe low scores. |
| **Dissent** | Tomasz chciałby zachować pre-AI score w ClickUp dla analytics — odroczono do Phase 2 |
| **Confidence** | High |
| **Revisit if** | Phase 2 analytics wymaga pre-AI vs post-AI comparison |

| Field | Content |
|-------|---------|
| **ID** | D-005 |
| **Decision** | Scenariusz 9 modułów: Webhook → MapFields → Brand/Service → Anthropic createAMessage → JSON Parse → Merge → ClickUp Mapping + Completeness → ClickUp Labels → ClickUp Create Task |
| **Rationale** | Rozbicie na więcej modułów eliminuje problem samoreferencji, umożliwia merge + rekalkulację completeness |
| **Dissent** | None |
| **Confidence** | High |
| **Revisit if** | Make operations limit wymusza redukcję modułów |

| Field | Content |
|-------|---------|
| **ID** | D-006 |
| **Decision** | Error handler na module Anthropic: Resume z fallback (klasyfikacja = "Do weryfikacji", ekstrakcja = wszystko null) |
| **Rationale** | Task musi powstać nawet jeśli AI nie odpowie. Automation ClickUp flaguje "Do weryfikacji" jako Urgent. |
| **Dissent** | None |
| **Confidence** | High |
| **Revisit if** | AI failure rate > 5% — wtedy potrzebny monitoring |

### Trade-Off Map

- **Choosing createAMessage over simpleTextPrompt**: Gains system prompt separation, structured output, better debugging. Costs Anthropic connection setup on client's Make account. Favored by all experts — net positive.
- **Choosing single AI call (3-in-1) over separate calls**: Gains lower cost, lower latency. Costs slightly more complex prompt. Favored by Aleksander (AI), Tomasz (Make) — operational efficiency.
- **Choosing post-merge completeness only**: Gains simplicity (1 score). Costs loss of pre-AI analytics data. Favored by Aleksander, Marta, Karolina. Tomasz accepts with deferral to Phase 2.

---

## Uncertainty Register

| ID | Unknown | Impact if Wrong | Resolvable? | Proposed Action | Owner |
|----|---------|----------------|-------------|-----------------|-------|
| U-001 | Dokładna ścieżka outputu createAMessage w Make (czy `result` czy `content[0].text`) | JSON Parse nie otrzyma danych | Yes | Testować po podpięciu connection | Tomasz |
| U-002 | Czy 5 nowych Custom Fields jest dodanych w ClickUp | Dane AI nie zapiszą się | Yes | Sprawdzić i dodać przed wdrożeniem | Marta |
| U-003 | Jakość ekstrakcji Haiku z polskich opisów kolokwialnych | Złe dane → gorszy niż brak danych | Partially | Testować na 20 realnych formularzach | Aleksander |
| U-004 | Czy Anthropic connection jest na koncie klienta w Make | Moduł nie zadziała | Yes | Skonfigurować API key | Tomasz |
| U-005 | Czy Make `ifempty()` poprawnie wykrywa null z JSON Parse | Merge logic może nie działać | Yes | Testować z próbnym JSON | Tomasz |

---

## Open Questions

1. **Czy klient ma API key Anthropic?** — Potrzebny do connection w Make. Jeśli nie, trzeba założyć konto na console.anthropic.com. Assumption: klient to zorganizuje.
2. **Jakie UUID dostaną nowe Custom Fields w ClickUp?** — Potrzebne do konfiguracji modułu ClickUp. Rozwiązanie: dodać pola, odczytać UUID z ClickUp API lub z interfejsu Make (widoczne po wyborze listy).
3. **Czy warto dodać tag/label wskazujący które pola są z AI ekstrakcji?** — Karolina sugeruje, że handlowcy chcieliby wiedzieć, które dane są pewne (formularz) a które z AI. Deferred: AI_CONFIDENCE i podsumowanie powinny wystarczyć w Phase 1.

---

## Next Work Package

| Priority | Action | Rationale | Depends On |
|----------|--------|-----------|------------|
| High | Dodać 5 nowych Custom Fields w ClickUp (ŹRÓDŁO_KANAŁ, KLASYFIKACJA_AI, AI_PODSUMOWANIE, KOMPLETNOŚĆ, AI_CONFIDENCE) | Bez tych pól dane AI nie zostaną zapisane | Dostęp do ClickUp workspace klienta |
| High | Skonfigurować Anthropic API connection w Make klienta | Wymagane przez createAMessage | API key Anthropic |
| High | Zbudować scenariusz 9-modułowy wg specyfikacji poniżej | Core deliverable | Custom Fields + Anthropic connection |
| Medium | Przetestować na 10-20 realnych formularzach | Walidacja jakości ekstrakcji | Scenariusz wdrożony |
| Medium | Dodać Error Handler na module Anthropic | Reliability | Scenariusz działający |
| Low | Skonfigurować Automation w ClickUp: KLASYFIKACJA_AI = "Do weryfikacji" → Priority = Urgent | Flaga dla handlowców | Custom Fields dodane |
| Low | Skonfigurować 2 nowe Views w ClickUp (NOWE-PRIORYTET, NOWE-KOMPLETNOŚĆ) | Ergonomia pracy zespołu | Custom Fields + dane testowe |

---

## DOCELOWA SPECYFIKACJA SCENARIUSZA — MODUŁ PO MODULE

### Moduł 1: Webhook (ID 1) — BEZ ZMIAN

```
Module: gateway:CustomWebHook
Hook: 4008814 (WEBHOOK | FORMULARZ LEAD)
Expect: miasto, telefon, data, godzina, uczestnicy, email, imie_nazwisko, uwagi, usluga, zrodlo
```

### Moduł 2: SetVars „Mapowanie Forminator" (ID 42) — BEZ ZMIAN

```
Module: util:SetVariables
Mapowanie:
  miasto ← {{1.select_2}}
  rodzaj_imprezy ← {{1.select_3}}
  pakiet ← {{1.select_1}}
  imie_nazwisko ← {{1.name_1}}
  email ← {{1.email_1}}
  telefon ← {{1.phone_1}}
  liczba_osob ← {{1.number_1}}
  opis ← {{1.textarea_1}}
  data_eventu ← {{1.name_2}}
  godzina ← {{1.name_3}}
  client_txt ← {{1.textarea_1}}
  source_url ← {{if(...)}} [bez zmian]
```

### Moduł 3: SetVars „Brand + Service" (ID 43) — POPRAWKA

```
Module: util:SetVariables
Zmienne:
  source_brand ← [bez zmian, odwołuje się do 42.source_url]
  service_type ← [bez zmian, odwołuje się do 42.source_url]
  task_date_display ← {{if(42.data_eventu != ""; 42.data_eventu; "BRAK DATY")}}

USUNĄĆ z tego modułu:
  - completeness_score (przeniesiony do modułu 7)
  - clickup_miasto (przeniesiony do modułu 7)
  - clickup_pakiet (przeniesiony do modułu 7)
  - clickup_pozyskanie (zostawić tutaj — zależy od source_url, nie od merge)
  - clickup_typ_imprezy (przeniesiony do modułu 7)
  - clickup_usluga (przeniesiony do modułu 7 — naprawia bug z ref do ID 7)
```

Zostają w module 3: `source_brand`, `service_type`, `task_date_display`, `clickup_pozyskanie`.

### Moduł 4: Anthropic createAMessage (ID 44) — ZMIANA

```
Module: anthropic-claude:createAMessage
Connection: [Anthropic API — skonfigurować]
Model: claude-haiku-4-5
System Prompt: [pełny system prompt z sekcji "Work Item 2" powyżej]
Messages:
  Role: user
  Content:
    Dane z formularza:
    - Miasto: {{42.miasto}} {{if(42.miasto = ""; "[PUSTE]"; "")}}
    - Rodzaj imprezy: {{42.rodzaj_imprezy}} {{if(42.rodzaj_imprezy = ""; "[PUSTE]"; "")}}
    - Pakiet: {{42.pakiet}} {{if(42.pakiet = ""; "[PUSTE]"; "")}}
    - Imię i nazwisko: {{42.imie_nazwisko}} {{if(42.imie_nazwisko = ""; "[PUSTE]"; "")}}
    - Email: {{42.email}} {{if(42.email = ""; "[PUSTE]"; "")}}
    - Telefon: {{42.telefon}} {{if(42.telefon = ""; "[PUSTE]"; "")}}
    - Liczba osób: {{42.liczba_osob}} {{if(42.liczba_osob = ""; "[PUSTE]"; "")}}
    - Data eventu: {{42.data_eventu}} {{if(42.data_eventu = ""; "[PUSTE]"; "")}}
    - Godzina: {{42.godzina}} {{if(42.godzina = ""; "[PUSTE]"; "")}}
    - Źródło: {{43.source_brand}}

    Opis klienta (wolny tekst):
    {{42.client_txt}}
Max Tokens: 500

ERROR HANDLER: Resume
  Fallback output: {"klasyfikacja":"do_weryfikacji","confidence":"niska","podsumowanie":"Błąd AI — lead wymaga ręcznej weryfikacji","ekstrakcja":{"miasto":null,"data_eventu":null,"godzina":null,"liczba_osob":null,"rodzaj_imprezy":null,"pakiet":null,"imie_nazwisko":null,"telefon":null,"email":null}}
```

### Moduł 5: JSON Parse (ID 45) — ZMIANA

```
Module: json:ParseJSON
Data structure: NOWA — "EVSO AI Response (Form v2)"
  Pola:
    klasyfikacja (text)
    confidence (text)
    podsumowanie (text)
    ekstrakcja (collection):
      miasto (text)
      data_eventu (text)
      godzina (text)
      liczba_osob (text)
      rodzaj_imprezy (text)
      pakiet (text)
      imie_nazwisko (text)
      telefon (text)
      email (text)
JSON string: {{trim(replace(replace(44.result; "```json"; ""); "```"; ""))}}
  (Zweryfikować ścieżkę output — może być 44.content[1].text lub inna)
```

### Moduł 6: SetVars „Merge" (NOWY)

```
Module: util:SetVariables
Zmienne:
  merged_miasto: {{if(42.miasto != ""; upper(trim(42.miasto)); if(45.ekstrakcja.miasto != ""; upper(trim(45.ekstrakcja.miasto)); ""))}}
  merged_rodzaj_imprezy: {{if(42.rodzaj_imprezy != ""; 42.rodzaj_imprezy; if(45.ekstrakcja.rodzaj_imprezy != ""; 45.ekstrakcja.rodzaj_imprezy; ""))}}
  merged_pakiet: {{if(42.pakiet != ""; 42.pakiet; if(45.ekstrakcja.pakiet != ""; 45.ekstrakcja.pakiet; ""))}}
  merged_imie: {{if(42.imie_nazwisko != ""; 42.imie_nazwisko; if(45.ekstrakcja.imie_nazwisko != ""; 45.ekstrakcja.imie_nazwisko; ""))}}
  merged_email: {{if(42.email != ""; 42.email; if(45.ekstrakcja.email != ""; 45.ekstrakcja.email; ""))}}
  merged_telefon: {{if(42.telefon != ""; 42.telefon; if(45.ekstrakcja.telefon != ""; 45.ekstrakcja.telefon; ""))}}
  merged_liczba_osob: {{if(42.liczba_osob != ""; 42.liczba_osob; if(45.ekstrakcja.liczba_osob != ""; 45.ekstrakcja.liczba_osob; ""))}}
  merged_data: {{if(42.data_eventu != ""; 42.data_eventu; if(45.ekstrakcja.data_eventu != ""; 45.ekstrakcja.data_eventu; ""))}}
  merged_godzina: {{if(42.godzina != ""; 42.godzina; if(45.ekstrakcja.godzina != ""; 45.ekstrakcja.godzina; ""))}}
```

### Moduł 7: SetVars „ClickUp Mapping + Completeness" (ZMIENIONY)

```
Module: util:SetVariables
Zmienne:
  merged_completeness: {{round(((if(PREV.merged_imie != ""; 1; 0)) + (if(PREV.merged_email != ""; 1; 0)) + (if(PREV.merged_telefon != ""; 1; 0)) + (if(PREV.merged_miasto != ""; 1; 0)) + (if(PREV.merged_data != ""; 1; 0)) + (if(PREV.merged_godzina != ""; 1; 0)) + (if(PREV.merged_liczba_osob != ""; 1; 0)) + (if(PREV.merged_rodzaj_imprezy != ""; 1; 0))) / 8 * 100)}}

  clickup_miasto: {{if(PREV.merged_miasto = "WROCŁAW"; 0; if(PREV.merged_miasto = "POZNAŃ"; 1; if(PREV.merged_miasto = "KRAKÓW"; 2; if(PREV.merged_miasto = "GDAŃSK"; 3; if(PREV.merged_miasto = "ŁÓDŹ"; 4; if(PREV.merged_miasto = "BYDGOSZCZ"; 5; if(PREV.merged_miasto = "TORUŃ"; 6; if(PREV.merged_miasto = "KATOWICE"; 7; if(PREV.merged_miasto = "SZCZECIN"; 8; if(PREV.merged_miasto = "WARSZAWA"; 9; if(PREV.merged_miasto = "SOPOT"; 10; "")))))))))))}}

  clickup_pakiet: {{if(upper(trim(PREV.merged_pakiet)) = "FUN"; 0; if(upper(trim(PREV.merged_pakiet)) = "PARTY"; 1; if(upper(trim(PREV.merged_pakiet)) = "LUX"; 2; if(upper(trim(PREV.merged_pakiet)) = "INDYVIDUAL !"; 3; if(upper(trim(PREV.merged_pakiet)) = "INDIVIDUAL !"; 3; if(upper(trim(PREV.merged_pakiet)) = "INVIDUAL II"; 4; if(upper(trim(PREV.merged_pakiet)) = "INDIVIDUAL II"; 4; "")))))))}}

  clickup_typ_imprezy: {{if(upper(trim(PREV.merged_rodzaj_imprezy)) = "KAWALERSKI"; 0; if(upper(trim(PREV.merged_rodzaj_imprezy)) = "PANIEŃSKI"; 1; if(upper(trim(PREV.merged_rodzaj_imprezy)) = "URODZINY"; 2; if(upper(trim(PREV.merged_rodzaj_imprezy)) = "FIRMÓWKA"; 3; if(upper(trim(PREV.merged_rodzaj_imprezy)) = "INNE"; 4; "")))))}}

  clickup_usluga: {{if(43.service_type = "BOAT"; 0; if(43.service_type = "TRAM"; 1; if(43.service_type = "BUS"; 2; "")))}}

Uwaga: PREV = ID modułu 6 (Merge). W Make wpisać konkretny numer ID modułu.
```

### Moduł 8: SetVars „Labels" (ZMIENIONY ID 46)

```
Module: util:SetVariables
Zmienne:
  clickup_klasyfikacja: {{if(45.klasyfikacja = "goracy_lead"; "Gorący lead"; if(45.klasyfikacja = "niekompletny_lead"; "Niekompletny lead"; if(45.klasyfikacja = "zapytanie_ogolne"; "Zapytanie ogólne"; if(45.klasyfikacja = "spam"; "Spam"; "Do weryfikacji"))))}}
  clickup_confidence: {{if(45.confidence = "wysoka"; "Wysoka"; if(45.confidence = "srednia"; "Średnia"; if(45.confidence = "niska"; "Niska"; "Niska")))}}
  task_title: {{if(MERGE.merged_data != ""; MERGE.merged_data; "BRAK DATY")}} | {{if(MERGE.merged_miasto != ""; MERGE.merged_miasto; "BRAK MIASTA")}} | {{if(43.service_type != ""; 43.service_type; "BRAK USŁUGI")}} | {{if(MERGE.merged_imie != ""; MERGE.merged_imie; "BRAK IMIENIA")}}
```

### Moduł 9: ClickUp Create Task (ID 47) — ZMIANA

```
Module: clickup:createTaskInList
Connection: CLICKUP EVSO
Workspace: EVSO (9015890695)
Space: EVSO (90153314249)
Folder: TESTOWY (901513877375)
List: NOWE ZAPYTANIA TESTOWY (901522503871)
Task Name: {{LABELS.task_title}}
Content:
  Zapytanie z formularza:

  Imię i nazwisko: {{MERGE.merged_imie}}
  Email: {{MERGE.merged_email}}
  Telefon: {{MERGE.merged_telefon}}
  Miasto: {{MERGE.merged_miasto}}
  Data: {{MERGE.merged_data}}
  Godzina: {{MERGE.merged_godzina}}
  Liczba uczestników: {{MERGE.merged_liczba_osob}}
  Usługa: {{43.service_type}}
  Pakiet: {{MERGE.merged_pakiet}}
  Źródło: {{43.source_brand}}

  ---
  Oryginalny tekst klienta:
  {{42.client_txt}}

  ---
  AI: {{45.podsumowanie}}

Status: nowe zapytania
Custom Fields:
  ILOŚĆ OSÓB (19536d62): {{MERGE.merged_liczba_osob}}
  TELEFON (234795d0): {{MERGE.merged_telefon}}
  DATA EVENTU (29799a60): {{if(MERGE.merged_data; parseDate(MERGE.merged_data; "YYYY-MM-DD"); "")}}
  MIASTO (35ee7b29): {{MAPPING.clickup_miasto}}
  POZYSKANIE (8546df9d): {{43.clickup_pozyskanie}}
  TYP IMPREZY (b65ab644): {{MAPPING.clickup_typ_imprezy}}
  PAKIET (bec02f93): {{MAPPING.clickup_pakiet}}
  EMAIL (d8265463): {{MERGE.merged_email}}
  GODZINA EVENTU (da316850): {{MERGE.merged_godzina}}
  USŁUGA (ea3c032c): {{MAPPING.clickup_usluga}}
  --- NOWE POLA (UUID do uzupełnienia po dodaniu w ClickUp) ---
  ŹRÓDŁO_KANAŁ: "Formularz"
  KLASYFIKACJA_AI: {{LABELS.clickup_klasyfikacja}}
  AI_PODSUMOWANIE: {{45.podsumowanie}}
  KOMPLETNOŚĆ: {{MAPPING.merged_completeness}}
  AI_CONFIDENCE: {{LABELS.clickup_confidence}}
```

---

## Artifacts to Update

| Artifact | Action | Specific Change | Triggered By |
|----------|--------|----------------|-------------|
| ClickUp Custom Fields | Create | Add 5 new fields: ŹRÓDŁO_KANAŁ (Dropdown), KLASYFIKACJA_AI (Dropdown), AI_PODSUMOWANIE (Long Text), KOMPLETNOŚĆ (Number), AI_CONFIDENCE (Dropdown) | D-005 |
| Make Data Structure | Create | "EVSO AI Response (Form v2)" — JSON schema with klasyfikacja, confidence, podsumowanie, ekstrakcja{9 fields} | D-001 |
| Make Scenario Blueprint | Update | Replace 7-module flow with 9-module flow per specification above | D-005 |
| Make Anthropic Module | Update | Change from simpleTextPrompt to createAMessage, add connection, system prompt, structured user message | D-003 |
| Make Error Handler | Create | Add Resume handler on Anthropic module with fallback JSON | D-006 |
| ClickUp Automation | Create | KLASYFIKACJA_AI = "Do weryfikacji" → Priority = Urgent | D-006 |
| ClickUp Views | Create | 2 new views: NOWE-PRIORYTET, NOWE-KOMPLETNOŚĆ | Phase 1 Guide |
| Phase 1 Implementation Guide | Update | Replace WPForms references with Forminator, update city mapping (nominative → no transformation needed), add ekstrakcja spec | All decisions |

---

## Meeting Metadata

- **Format used**: Structured Workshop
- **Experts activated**: 4 of 4
- **Decisions made**: 6
- **Uncertainties logged**: 5
- **Open questions**: 3
- **Estimated confidence in primary recommendation**: High
