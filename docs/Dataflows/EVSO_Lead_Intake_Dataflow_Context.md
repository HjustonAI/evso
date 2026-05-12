# EVSO — Context plik: lejek wejścia zgłoszeń i obieg danych do ClickUp

**Cel pliku:** dostarczyć LLM lub człowiekowi precyzyjnego kontekstu o tym, **jak zgłoszenia wpadają do systemu**, jakie dane są zbierane na wejściu, jak są normalizowane i gdzie lądują w ClickUp.

**Zakres:** ten plik dotyczy głównie **warstwy wejścia danych / intake**, a nie pełnej architektury całego workspace.

**Źródła użyte do przygotowania:**
- screenshoty stron internetowych i formularzy dostarczone przez użytkownika,
- screenshoty ClickUp dostarczone wcześniej,
- wcześniejszy snapshot architektury ClickUp EVSO,
- oficjalna dokumentacja ClickUp, ale **wyłącznie w zakresie potrzebnym do zrozumienia danych i przepływu**.

**Ważna zasada interpretacyjna:**
- wszystko, co wynika bezpośrednio ze screenshotów lub z deklaracji użytkownika, jest oznaczane jako **potwierdzone**,
- wszystko, co jest logiczną interpretacją albo bezpiecznym wnioskiem roboczym, jest oznaczane jako **inferencja / interpretacja**,
- wszystko, co byłoby już zmianą projektową lub rekomendacją, jest opisane osobno jako **rekomendacja**.

---

## 1) Executive summary

EVSO zbiera zgłoszenia eventowe co najmniej dwoma głównymi kanałami:

1. **formularze na stronach brandowych**:
   - `partytram.fun`
   - `partyboat.fun`
   - `busparty.fun`

2. **bezpośrednie wiadomości e-mail** kierowane na adresy kontaktowe.

Na dziś potwierdzony stan wejścia wygląda tak:
- formularz zbiera dane częściowo ustrukturyzowane,
- istnieją bardzo proste scenariusze Make typu **WPForms -> Create Task in ClickUp**,
- część klientów **nie używa formularza**, tylko pisze bezpośrednio maila,
- przez to do systemu trafiają dwa typy leadów:
  - **lead strukturalny** (z formularza),
  - **lead półstrukturalny / nieustrukturyzowany** (z maila).

KlikUpowym miejscem prawdy dla surowego intake'u jest lista **`EVSO / CRM / NOWE ZAPYTANIA`**, gdzie leady mają custom ID w formacie **`LEAD-####`** i są reprezentowane jako taski. W tasku przechowywane są jednocześnie:
- opis oryginalnego zapytania,
- zestaw Custom Fields,
- historia aktywności,
- możliwość wysyłania maili z poziomu taska.

---

## 2) Zakres tego dokumentu

Ten plik opisuje dokładnie:

- **skąd** wpływają zgłoszenia,
- **jakie dane** użytkownik może wprowadzić na wejściu,
- **jakie dane są widoczne w ClickUp** po stronie taska-leada,
- **jakie mechanizmy ClickUp są istotne dla intake'u**,
- **jaka jest kanoniczna mapa danych wejściowych -> danych w ClickUp**,
- **gdzie są ryzyka utraty jakości danych**.

Ten plik **nie opisuje szczegółowo**:
- całego delivery flow po wygraniu sprzedaży,
- szczegółów rozliczeń,
- szczegółów wszystkich widoków i dashboardów,
- pełnej przyszłej architektury automatyzacji.

---

## 3) Kanały wejścia zgłoszeń

## 3.1 Formularze na stronach brandowych

### Potwierdzone domeny
- `partytram.fun`
- `partyboat.fun`
- `busparty.fun`

### Potwierdzona informacja od użytkownika
Użytkownik wskazał, że **wszystkie te strony wyglądają praktycznie tak samo** i stanowią różne wejścia do podobnego lejka.

### Co widać na screenach
Na screenach pokazany został przede wszystkim `partyboat.fun`, ale układ należy traktować jako wzorzec dla całej rodziny stron.

Widoczny jest:
- hero z CTA,
- filtr/quick selector na stronie głównej,
- modal formularza po kliknięciu CTA `Opisz swój pomysł`,
- kontakt page z adresami mailowymi i telefonami.

### Charakter danych z formularza
Formularz jest **częściowo ustrukturyzowany**. Użytkownik wybiera część danych z predefiniowanych opcji, a część wpisuje ręcznie.

To oznacza, że formularz daje znacznie lepszy materiał dla automatyzacji niż zwykły mail.

---

## 3.2 Bezpośrednie maile

### Potwierdzony stan
Użytkownik wskazał, że:
- część klientów **nie korzysta z formularza**,
- zamiast tego **pisze maile bezpośrednio**.

### Co wynika z tego dla jakości danych
Lead mailowy jest zwykle:
- mniej ustrukturyzowany,
- bardziej różnorodny językowo,
- częściej niekompletny,
- czasem bardzo ogólny,
- czasem spamowy,
- czasem nie dotyczy prostego pakietowego eventu.

### Charakter danych z maila
W wiadomości mogą, ale nie muszą się pojawić:
- miasto,
- typ imprezy,
- pakiet,
- data,
- godzina,
- liczba osób,
- telefon,
- pełny opis potrzeby.

W praktyce mail jest więc źródłem **semi-structured / unstructured input**.

---

## 4) Publiczna warstwa wejścia — formularz WWW

## 4.1 Widoczne pola formularza

Na podstawie screenshotów formularza widać następujące pola:

### Selektory / pola wyboru
- `Wybierz miejscowość`
- `Wybierz rodzaj imprezy`
- `Wybierz pakiet`

### Pola tekstowe / kontaktowe
- `Imię i Nazwisko *`
- `Adres e-mail *`
- `Numer telefonu *`
- `Data *`
- `Godzina *`
- `Liczba osób *`
- `Opisz swój pomysł`

### Walidacja / antyspam
- reCAPTCHA (`I'm not a robot`)

### Akcja końcowa
- przycisk `Wyślij`

### Ważna obserwacja
Na screenie gwiazdką oznaczone są na pewno:
- imię i nazwisko,
- email,
- telefon,
- data,
- godzina,
- liczba osób.

Nie należy natomiast na 100% zakładać tylko na podstawie screena, że:
- miasto jest technicznie wymagane,
- rodzaj imprezy jest technicznie wymagany,
- pakiet jest technicznie wymagany,
- opis pomysłu jest wymagany.

Z punktu widzenia kontekstu LLM należy więc traktować je jako:
- **widoczne i logicznie ważne**,
- ale niekoniecznie zawsze formalnie required na poziomie walidacji frontu.

---

## 4.2 Widoczne słowniki wyboru na stronie

## Miejscowość
Na screenie dropdown `Miejscowość` pokazuje opcje:
- `w Katowicach`
- `we Wrocławiu`
- `w Poznaniu`
- `w Warszawie`
- `w Krakowie`
- `w Gdańsku / Sopocie`
- `w Toruniu`
- `w Bydgoszczy`
- `w Szczecinie`

### Wniosek praktyczny
W warstwie WWW miasta są prezentowane w formie językowej przyjaznej użytkownikowi (`w Krakowie`, `we Wrocławiu`), ale w ClickUp wartości są przechowywane w bardziej technicznej, skróconej formie typu:
- `KRAKOW` / `KRAKÓW`
- `WROCLAW` / `WROCŁAW`
- `BYDGOSZCZ`
- `SZCZECIN`

To oznacza, że **normalizacja miasta jest ważnym elementem lejka danych**.

## Rodzaj imprezy
Na screenie widać opcje:
- `Wieczór kawalerski`
- `Wieczór panieński`
- `Urodziny`
- `Impreza firmowa`

To jest naturalny kandydat do mapowania na ClickUp field typu **`TYP IMPREZY`**.

## Pakiet
Na screenie widać opcje:
- `FUN`
- `PARTY`
- `LUX`
- `INDYWIDUALNY`

To jest naturalny kandydat do mapowania na ClickUp field typu **`PAKIET`**.

---

## 4.3 Czego formularz nie mówi wprost, ale silnie sugeruje

Na podstawie screenshotów i brandingu można bezpiecznie przyjąć następujący model roboczy:

- domena `partytram.fun` oznacza źródło związane z ofertą typu **tram / partytram**,
- domena `partyboat.fun` oznacza źródło związane z ofertą typu **boat / statek**,
- domena `busparty.fun` oznacza źródło związane z ofertą typu **bus / partybus**.

### Ważne zastrzeżenie
To jest **sensowna interpretacja biznesowa**, ale w tym pliku należy ją traktować jako:
- **bardzo mocną inferencję**,
- nie jako technicznie potwierdzony mapping aktualnej automatyzacji.

Czyli: domena najpewniej dostarcza kontekstu usługi, ale screenshoty nie pokazują bezpośrednio hidden fielda ani scenariusza Make, który to ustawia.

---

## 4.4 Kontakt direct z poziomu strony

Na screenie kontakt page `partyboat.fun/kontakt/` widać co najmniej:

### Adresy i kontakty ogólne
- `kontakt@partyboat.fun`
- numer telefonu dla wydarzeń prywatnych: `505 194 995`

### Kontakty firmowe / handlowe
- `kontakt@partyboat.fun`
- `Dawid 505 255 100`
- `dawid.cybulkin@evso.pl`
- miasta przypisane: `Bydgoszcz, Gdańsk, Toruń, Wrocław`
- `Krzysztof 795 333 020`
- `krzysztof.ksiazek@evso.pl`
- miasta przypisane: `Katowice, Kraków, Poznań, Szczecin, Warszawa`

### Znaczenie dla lejka
To oznacza, że część leadów może trafiać:
- na ogólny adres brandowy,
- bezpośrednio do konkretnego handlowca,
- potencjalnie z przypisaniem per miasto.

To jest ważne, bo **kanał mailowy nie musi być pojedynczy**.

---

## 5) Aktualny stan automatyzacji (AS-IS)

### Potwierdzone przez użytkownika
Użytkownik wskazał, że istnieją:
- **najbardziej podstawowe automatyzacje Make**,
- typu **jeden moduł WPForm i przekazanie -> utwórz task w ClickUp**.

### Jak należy to rozumieć
Najbezpieczniejszy opis obecnego stanu to:

**AS-IS automation pattern:**
`WPForms submission -> Make -> ClickUp: Create Task`

### Czego nie potwierdzono
Nie zostało potwierdzone, że scenariusz:
- uzupełnia wszystkie Custom Fields,
- waliduje lub normalizuje wartości,
- deduplikuje leady,
- routuje leady po mieście lub typie imprezy,
- tworzy komentarze zamiast tasków,
- dopina załączniki,
- aktualizuje istniejące taski.

Dlatego obecny stan należy traktować jako **minimalny intake bridge**, a nie jako pełny system routingu.

---

## 6) ClickUp jako miejsce docelowe dla danych wejściowych

Ta sekcja opiera się na wcześniejszym snapshotcie architektury ClickUp EVSO.

## 6.1 Główna lista intake

### Potwierdzone miejsce w hierarchii
- `EVSO / CRM / NOWE ZAPYTANIA`

### Rola listy
Ta lista pełni rolę **centralnego zbiornika nowych leadów** przed ich dalszym opracowaniem sprzedażowym.

### Widoczne kolumny listy
- `Name`
- `DATA EVENTU`
- `Assignee`
- `Due date`
- `Date created`
- `Custom Task ID`

### Identyfikator leada
Leady używają custom ID w schemacie:
- `LEAD-####`

Przykłady:
- `LEAD-4750`
- `LEAD-4885`
- `LEAD-5101`

---

## 6.2 Konwencja nazwy taska

W `NOWE ZAPYTANIA` widoczny jest wzorzec nazewniczy:

`[DATE or BRAK DATY] | [CITY or WYBIERZ MIASTO] | [SERVICE] | [NAME]`

Przykłady:
- `BRAK DATY | KRAKOW | TRAM | domingo fernandez`
- `BRAK DATY | WROCLAW | BOAT | Ola`
- `24-01-2026 | WYBIERZ MIASTO | TRAM | Arleta Szymańska`
- `BRAK DATY | WYBIERZ MIASTO | BOAT | Ewa Łoboda`

### Znaczenie tej konwencji
Nazwa taska pełni jednocześnie funkcję:
- szybkiego podglądu danych,
- tymczasowego nośnika stanu kompletności,
- uproszczonego nagłówka operacyjnego.

### Ważna obserwacja
W nazwie taska bywają używane formy znormalizowane lub uproszczone, np.:
- `KRAKOW` zamiast `w Krakowie`,
- `BRAK DATY` gdy pole daty nie jest znane,
- `WYBIERZ MIASTO` gdy miasto nie zostało jeszcze określone.

---

## 6.3 Potwierdzone Custom Fields na tasku-leadzie

Na otwartym tasku leada widać następujące pola:

- `MIASTO`
- `PAKIET`
- `POZYSKANIE`
- `TELEFON`
- `TYP IMPREZY`
- `USŁUGA`
- `GODZINA EVENTU`
- `DATA EVENTU`
- `EMAIL`
- `ILOŚĆ OSÓB`
- `JEDNOSTKA`

### Przykładowe wypełnione wartości widoczne na screenie
- `POZYSKANIE: TRAMPARTY.PL`
- `TELEFON: 628057720`
- `USŁUGA: TRAM`
- `GODZINA EVENTU: 20:00–22:00`
- `EMAIL: domfermun@gmail.com`
- `ILOŚĆ OSÓB: 10`

### Co to oznacza dla modelu danych
ClickUp task-lead przechowuje jednocześnie:
- dane kontaktowe,
- dane eventowe,
- źródło pozyskania,
- klasyfikację oferty / typu usługi,
- w razie potrzeby dalej description z pełnym tekstem zapytania.

---

## 6.4 Description jako nośnik oryginalnego zapytania

Na screenshotach widać, że opis taska zawiera pełną treść oryginalnego zapytania klienta w języku naturalnym.

To oznacza, że description pełni rolę:
- **surowego inputu**,
- archiwum pierwszego kontaktu,
- kontekstu dla dalszej pracy handlowca lub AI.

---

## 6.5 Activity + email composer

Na tasku widać prawy panel `Activity` oraz email composer na dole.

To oznacza, że task może agregować:
- historię zmian,
- komentarze / aktywność,
- wysyłkę maili z poziomu taska.

W praktyce task jest więc nie tylko rekordem danych, ale też **miejscem prowadzenia konwersacji**.

---

## 6.6 Downstream handoff po intake

Na podstawie wcześniejszych screenshotów ClickUp można bezpiecznie przyjąć, że po etapie intake lead trafia dalej do owner-specific sales pipelines, np.:
- `SPRZEDAŻ KUBA`
- `SPRZEDAŻ WIKTOR`
- `SPRZEDAŻ DAWID`
- `SPRZEDAŻ KRIS`

A po sprzedaży / potwierdzeniu przechodzi do obszaru:
- `ZLECENIA`

Dla tego pliku ważne jest jednak tylko to, że:
- **`NOWE ZAPYTANIA` to wejście / intake**,
- późniejsze listy są już downstream processing.

---

## 7) Tylko potrzebne mechaniki z dokumentacji ClickUp

Ta sekcja zawiera wyłącznie te fakty z dokumentacji ClickUp, które są istotne dla zrozumienia obiegu danych.

## 7.1 Custom Fields są zależne od lokalizacji

ClickUp pozwala dodawać Custom Fields do:
- List,
- Folderów,
- Space,
- całego Workspace.

Jeżeli Custom Field zostanie dodane do konkretnej listy, będzie widoczne na wszystkich taskach w tej liście.

### Znaczenie dla EVSO
To wspiera model, w którym `NOWE ZAPYTANIA` ma własny zestaw pól intake'owych, a inne listy mogą mieć dalsze pola operacyjne.

---

## 7.2 Custom Fields są widoczne pod opisem taska i jako kolumny w widokach

ClickUp pokazuje Custom Fields:
- pod description taska,
- jako kolumny w widokach List/Table,
- częściowo także w innych widokach.

### Znaczenie dla EVSO
Dzięki temu lead jest jednocześnie:
- czytelny w tabeli,
- operacyjny po otwarciu taska,
- gotowy do dalszej obróbki.

---

## 7.3 Pole typu Email może zasilać composer maila

W dokumentacji ClickUp opisano, że jeśli task ma **Email Custom Field**, jego wartość pojawia się jako sugestia przy komponowaniu wiadomości z taska.

### Znaczenie dla EVSO
Pole `EMAIL` w leadzie nie jest tylko polem informacyjnym — ono może służyć bezpośrednio do rozpoczęcia lub kontynuacji korespondencji z klientem z poziomu taska.

---

## 7.4 Odpowiedzi na maile wysłane z taska wracają do taska, ale temat ma znaczenie

ClickUp opisuje, że odpowiedzi na mail wysłany z taska pojawiają się w activity feedzie taska. Jednocześnie dokumentacja zaznacza, że jeżeli odbiorca zmieni subject wiadomości, odpowiedź nie wróci do ClickUp w tym samym flow.

### Znaczenie dla EVSO
To jest bardzo ważna właściwość przy projektowaniu lejka:
- zwykły `reply` może wracać do taska,
- ale „nowy mail” lub zmieniony temat może już wymagać osobnego routingu.

---

## 7.5 Inbox w ClickUp nie jest systemem rekordów dla leadów

ClickUp dokumentuje, że Inbox jest:
- osobisty dla użytkownika,
- związany z danym Workspace,
- służy do powiadomień, update'ów, komentarzy i przypomnień.

### Znaczenie dla EVSO
Inbox **nie powinien być traktowany jako główne repozytorium leadów**. Systemem rekordów dla intake'u pozostają taski w CRM.

---

## 7.6 ClickUp ma natywne "tasks/comments via email"

ClickUp posiada funkcję pozwalającą tworzyć taski i komentarze z maila przez unikalny URL dla taska lub listy.

### Znaczenie dla EVSO
To jest funkcja platformy, ale **nie ma dowodu**, że jest dziś główną osią Waszego intake'u. W obecnym pliku należy ją traktować jako:
- istotną zdolność systemu,
- potencjalną opcję pomocniczą,
- nie jako potwierdzony główny kanał produkcyjny.

---

## 7.7 Make ma natywne akcje potrzebne do obsługi intake'u

Z dokumentacji ClickUp dotyczącej Make wynika, że integracja wspiera m.in.:
- tworzenie tasków,
- edycję tasków,
- edycję tasków z custom fields,
- dodawanie komentarzy do tasków,
- upload attachmentów,
- wyszukiwania/listowania.

### Znaczenie dla EVSO
To potwierdza, że nawet prosty intake może być z czasem rozbudowany bez zmiany narzędzia docelowego.

---

## 8) Kanoniczny model danych wejściowych

Ta sekcja nie mówi jeszcze, jak system powinien działać idealnie, tylko **jak najlepiej opisać dane wejściowe w sposób kanoniczny**.

## 8.1 Typy rekordów wejściowych

W kontekście EVSO należy rozróżnić dwa podstawowe typy inputu:

### A. `form_submission`
Lead z formularza WWW.
Charakterystyka:
- bardziej kompletny,
- częściowo znormalizowany,
- zawiera pola wyboru,
- łatwiejszy do mapowania.

### B. `direct_email`
Lead z bezpośredniego maila.
Charakterystyka:
- mniej kompletny,
- bardziej swobodny językowo,
- może być bardzo ogólny,
- wymaga większej interpretacji.

---

## 8.2 Kanoniczne pola biznesowe

Poniżej znajduje się najlepszy roboczy model tego, jakie biznesowe informacje faktycznie krążą w lejku.

| Kanoniczne pole | Znaczenie | Typ | Źródło formularz | Źródło e-mail | ClickUp field / nośnik |
|---|---|---|---|---|---|
| `lead_id` | identyfikator leada | string | generowany po utworzeniu taska | generowany po utworzeniu taska | `Custom Task ID` (`LEAD-####`) |
| `source_channel` | kanał wejścia | enum/string | form | email | częściowo `POZYSKANIE` lub metadane integracji |
| `source_brand` | brand / domena wejściowa | enum/string | partytram / partyboat / busparty | e-mail brandowy lub handlowy | najpewniej `POZYSKANIE` lub pole pomocnicze |
| `customer_name` | imię i nazwisko klienta | string | pole formularza | z treści / podpisu maila | część nazwy taska + ewentualnie description |
| `email` | mail klienta | string | pole formularza | nadawca lub treść | `EMAIL` |
| `phone` | telefon klienta | string | pole formularza | czasem w treści | `TELEFON` |
| `city_raw` | forma miasta z wejścia | string | np. `w Krakowie` | np. z treści maila | surowy input / description |
| `city_normalized` | znormalizowane miasto | enum/string | po mapowaniu | po interpretacji | `MIASTO` |
| `event_type` | rodzaj imprezy | enum | dropdown | z treści / interpretacji | `TYP IMPREZY` |
| `service_type` | medium/usługa typu tram/boat/bus | enum | po domenie lub selekcji pośredniej | z treści / brandu | `USŁUGA` |
| `package` | pakiet oferty | enum | dropdown | z treści jeśli występuje | `PAKIET` |
| `event_date` | dokładna data wydarzenia | date/string | pole formularza | czasem w treści | `DATA EVENTU` |
| `event_time` | godzina wydarzenia | time/string | pole formularza | czasem w treści | `GODZINA EVENTU` |
| `people_count` | liczba uczestników | int/string | pole formularza | czasem w treści | `ILOŚĆ OSÓB` |
| `free_text_request` | opis potrzeby | long text | `Opisz swój pomysł` | treść maila | description taska |
| `unit_or_asset` | konkretna jednostka / przypisanie | enum/string | zwykle brak na wejściu | zwykle brak | `JEDNOSTKA` |

---

## 8.3 Kluczowe reguły normalizacji

### Miasto
Dane wejściowe ze strony są w formie odmiennej językowo, np.:
- `w Krakowie`
- `we Wrocławiu`
- `w Bydgoszczy`

Natomiast w ClickUp i nazwach tasków funkcjonują formy uproszczone/operacyjne, np.:
- `KRAKOW` / `KRAKÓW`
- `WROCLAW` / `WROCŁAW`
- `BYDGOSZCZ`

**Reguła robocza:**
- zachować `city_raw` jeśli dostępne,
- ustawić `MIASTO` jako formę znormalizowaną.

### Usługa
`USŁUGA` w ClickUp opisuje raczej **medium / typ jednostki** (`TRAM`, `BOAT`) niż rodzaj imprezy.

Dlatego nie należy mieszać:
- `TYP IMPREZY` = np. `Wieczór kawalerski`
- `USŁUGA` = np. `TRAM` / `BOAT` / potencjalnie `BUS`

### Źródło pozyskania
`POZYSKANIE` jest polem źródłowym, ale na podstawie screenshotów nie da się jeszcze na 100% stwierdzić, czy przechowuje:
- nazwę domeny,
- kanał marketingowy,
- nazwę brandu,
- czy inny identyfikator źródła.

Na screenie widać wartość `TRAMPARTY.PL`, więc pole należy traktować jako **źródłowo-marketingowe**, nie stricte jako pole eventowe.

---

## 9) AS-IS data flow — opis krok po kroku

## 9.1 Ścieżka A: formularz WWW

```text
Użytkownik odwiedza stronę brandową
-> otwiera formularz "Opisz swój pomysł"
-> wybiera / wpisuje dane
-> przechodzi reCAPTCHA
-> wysyła formularz
-> WPForms generuje submission
-> prosty scenariusz Make odbiera submission
-> Make tworzy task w ClickUp
-> task trafia do CRM / NOWE ZAPYTANIA
-> lead otrzymuje Custom Task ID typu LEAD-####
-> dane są widoczne w tasku i/lub w nazwie taska
```

### Co jest potwierdzone
- formularz istnieje,
- pola formularza istnieją,
- prosty scenariusz WPForms -> Make -> Create Task istnieje,
- ClickUp jest miejscem docelowym.

### Czego nie potwierdzono
- dokładnego mapowania każdego pola,
- czy description zawsze zawiera pełen payload,
- czy wszystkie Custom Fields są ustawiane automatycznie,
- czy naming jest generowany automatycznie czy częściowo ręcznie.

---

## 9.2 Ścieżka B: bezpośredni mail

```text
Klient wysyła e-mail bezpośrednio
-> wiadomość trafia na adres brandowy lub handlowy
-> dane są mniej ustrukturyzowane
-> człowiek lub dodatkowy proces musi wprowadzić dane do ClickUp
-> powstaje task-lead w CRM / NOWE ZAPYTANIA
-> description przechowuje treść wiadomości
-> brakujące pola mogą pozostać puste
```

### Co jest potwierdzone
- kanał e-mail istnieje,
- klienci realnie z niego korzystają,
- część leadów przychodzi poza formularzem.

### Czego nie potwierdzono
- czy obecnie jest zautomatyzowany import maila,
- czy mail tworzy task automatycznie,
- czy e-mail jest kopiowany ręcznie do description,
- czy obecnie direct-email flow korzysta z Email to Task / Email in ClickUp / Make / ręcznej obsługi.

W tym pliku należy więc opisywać go jako **istniejący kanał wejścia, ale niejednoznacznie zautomatyzowany**.

---

## 9.3 Ścieżka C: odpowiedź z poziomu taska

```text
Lead istnieje już jako task w ClickUp
-> handlowiec otwiera task
-> pole EMAIL sugeruje odbiorcę w composerze
-> mail wychodzi z taska
-> zwykły reply klienta może wrócić do activity taska
-> zmiana subjectu przez klienta może przerwać ten powrót do taska
```

To nie jest pierwotny intake, ale jest ważne dla pełnego obrazu obiegu danych, bo od tego momentu task staje się także nośnikiem korespondencji.

---

## 10) Krytyczne miejsca utraty jakości danych

Ta sekcja jest bardzo ważna dla przyszłego użycia pliku przez LLM lub architekta automatyzacji.

## 10.1 Formularz i ClickUp nie mówią tym samym językiem danych

Przykład:
- formularz: `w Krakowie`
- task: `KRAKOW` / `KRAKÓW`

To oznacza konieczność normalizacji.

## 10.2 `TYP IMPREZY` i `USŁUGA` to nie to samo

Przykład:
- `Wieczór kawalerski` != `BOAT`
- `Impreza firmowa` != `TRAM`

Jeśli te dwie warstwy będą mylone, powstanie chaos w danych.

## 10.3 Mail ma niższą kompletność niż formularz

Mail może nie zawierać:
- daty,
- miasta,
- telefonu,
- liczby osób,
- pakietu.

Dlatego direct email intake wymaga bardziej ostrożnego opisu niż form intake.

## 10.4 `POZYSKANIE` jest ważne, ale nie do końca jednoznaczne semantycznie

Wartość `TRAMPARTY.PL` sugeruje source attribution, ale nie rozwiązuje samodzielnie wszystkich pytań o:
- brand,
- kampanię,
- kanał,
- domenę,
- medium wydarzenia.

## 10.5 Nazwa taska zawiera dane biznesowe, ale nie powinna być jedynym źródłem prawdy

Nazwa taska jest bardzo użyteczna operacyjnie, ale nadal najpewniejszym miejscem dla systemowej pracy powinny być:
- Custom Fields,
- description,
- ID taska.

---

## 11) Dokumentacyjny model obiegu danych — wersja idealna do kontekstu LLM

Poniższy model jest najlepszym skrótem semantycznym dla LLM.

### Krótkie ujęcie
EVSO ma wieloźródłowy intake leadów eventowych. Najważniejsze wejścia to brandowe formularze WWW i bezpośrednie maile. Formularz daje dane częściowo ustrukturyzowane: miasto, typ imprezy, pakiet, dane kontaktowe, data, godzina, liczba osób i opis pomysłu. Mail daje dane bardziej swobodne i często niekompletne. Oba typy zgłoszeń kończą jako taski w `EVSO / CRM / NOWE ZAPYTANIA`, gdzie lead dostaje `LEAD-####`, zachowuje oryginalny tekst zapytania w description i ma zestaw pól takich jak `MIASTO`, `PAKIET`, `POZYSKANIE`, `TELEFON`, `TYP IMPREZY`, `USŁUGA`, `GODZINA EVENTU`, `DATA EVENTU`, `EMAIL`, `ILOŚĆ OSÓB`, `JEDNOSTKA`. Następnie lead może być dalej obrabiany przez handlowca i korespondencję mailową z poziomu taska.

### Bardzo ważne skróty myślowe dla LLM
- `NOWE ZAPYTANIA` = surowe wejście / intake.
- `LEAD-####` = główny identyfikator leada.
- `TYP IMPREZY` = typ wydarzenia.
- `USŁUGA` = medium/event vehicle (tram/boat/bus), nie rodzaj imprezy.
- `POZYSKANIE` = źródło lub brand / kanał, ale semantyka pola nie jest jeszcze w 100% rozstrzygnięta.
- `EMAIL` = pole kontaktowe oraz źródło sugestii dla composerów email w tasku.
- ClickUp Inbox != baza leadów; taski są systemem rekordów.

---

## 12) Machine-friendly snapshot

```yaml
context_type: EVSO_lead_intake_dataflow
version: 1
scope: intake_only

channels:
  website_forms:
    domains:
      - partytram.fun
      - partyboat.fun
      - busparty.fun
    similarity_note: "Użytkownik wskazał, że strony wyglądają praktycznie tak samo"
    visible_fields:
      selects:
        - miejscowosc
        - rodzaj_imprezy
        - pakiet
      text_fields:
        - imie_i_nazwisko
        - adres_email
        - numer_telefonu
        - data
        - godzina
        - liczba_osob
        - opis_swojego_pomyslu
      validation:
        - recaptcha
    city_options_visible:
      - w Katowicach
      - we Wrocławiu
      - w Poznaniu
      - w Warszawie
      - w Krakowie
      - w Gdańsku / Sopocie
      - w Toruniu
      - w Bydgoszczy
      - w Szczecinie
    event_type_options_visible:
      - Wieczór kawalerski
      - Wieczór panieński
      - Urodziny
      - Impreza firmowa
    package_options_visible:
      - FUN
      - PARTY
      - LUX
      - INDYWIDUALNY

  direct_email:
    exists: true
    structure: semi_structured_or_unstructured
    notes:
      - klient czasem omija formularz
      - wiadomości bywają niekompletne
      - możliwe wejście przez adres ogólny lub handlowców

current_automation:
  confirmed_pattern: "WPForms -> Make -> ClickUp Create Task"
  sophistication: minimal
  unknowns:
    - exact field mapping
    - deduplication
    - enrichment
    - normalization depth
    - email ingestion automation

clickup_target:
  list: "EVSO / CRM / NOWE ZAPYTANIA"
  task_id_pattern: "LEAD-####"
  task_title_pattern: "[DATE|BRAK DATY] | [CITY|WYBIERZ MIASTO] | [SERVICE] | [NAME]"
  custom_fields_visible:
    - MIASTO
    - PAKIET
    - POZYSKANIE
    - TELEFON
    - TYP IMPREZY
    - USŁUGA
    - GODZINA EVENTU
    - DATA EVENTU
    - EMAIL
    - ILOŚĆ OSÓB
    - JEDNOSTKA
  task_contains:
    - original inquiry in description
    - activity log
    - email composer

canonical_data_contract:
  structured_input_type: form_submission
  semi_structured_input_type: direct_email
  key_business_fields:
    - lead_id
    - source_channel
    - source_brand
    - customer_name
    - email
    - phone
    - city_raw
    - city_normalized
    - event_type
    - service_type
    - package
    - event_date
    - event_time
    - people_count
    - free_text_request
    - unit_or_asset

critical_normalization_rules:
  - city should be normalized from locative website form to canonical operational city value
  - event_type must stay separate from service_type
  - source attribution should not be confused with service_type
  - task title is useful but should not be treated as sole source of truth
```

---

## 13) Potwierdzone vs inferowane

## Bezpośrednio potwierdzone
- istnieją trzy domeny brandowe: `partytram.fun`, `partyboat.fun`, `busparty.fun`,
- strony są wg użytkownika bardzo podobne,
- istnieje formularz z widocznymi polami opisanymi wyżej,
- istnieją direct e-maile jako alternatywny kanał wejścia,
- istnieje prosty scenariusz `WPForms -> Make -> Create Task in ClickUp`,
- intake trafia do ClickUp,
- w ClickUp istnieje lista `EVSO / CRM / NOWE ZAPYTANIA`,
- leady mają custom task ID `LEAD-####`,
- task-lead ma pola: `MIASTO`, `PAKIET`, `POZYSKANIE`, `TELEFON`, `TYP IMPREZY`, `USŁUGA`, `GODZINA EVENTU`, `DATA EVENTU`, `EMAIL`, `ILOŚĆ OSÓB`, `JEDNOSTKA`,
- task przechowuje description i ma email composer oraz activity.

## Mocne inferencje robocze
- formularz na każdej z trzech domen pełni analogiczną rolę wejściową,
- domena brandowa najpewniej niesie kontekst typu usługi,
- `POZYSKANIE` jest polem źródłowym / source attribution,
- `NOWE ZAPYTANIA` jest centralnym zbiornikiem intake'u,
- direct email flow nie jest dziś tak dobrze ustrukturyzowany jak form flow.

## Rzeczy niepotwierdzone i nie wolno ich traktować jako fakt
- dokładny mapping każdego pola WPForms do konkretnego Custom Field,
- dokładna semantyka pola `POZYSKANIE`,
- czy `BUS` już istnieje jako wartość w polu `USŁUGA`,
- czy direct mail intake jest dziś częściowo zautomatyzowany,
- czy routing po mieście działa automatycznie,
- czy tytuł taska jest generowany całkowicie automatycznie.

---

## 14) Jak używać tego pliku jako kontekstu dla LLM

LLM dostający ten plik powinien rozumieć, że:

1. EVSO ma **multi-source intake** leadów eventowych.
2. Formularz WWW daje dane częściowo ustrukturyzowane.
3. Mail daje dane bardziej swobodne i mniej kompletne.
4. ClickUp task w `NOWE ZAPYTANIA` jest rekordem intake'owym.
5. `LEAD-####` to referencyjny numer leada.
6. `TYP IMPREZY` i `USŁUGA` to dwa różne wymiary danych.
7. `MIASTO` wymaga normalizacji względem form wejściowych z WWW.
8. `EMAIL` jest polem kontaktowym i kanałem do dalszej korespondencji z taska.
9. Inbox ClickUp nie jest głównym repozytorium leadów.
10. jeśli LLM ma projektować automatyzację lub data mapping, powinien bazować na **Custom Fields i description**, a nie wyłącznie na tytule taska.

---

## 15) Relacja do wcześniejszego pliku kontekstowego

Ten dokument **uzupełnia** wcześniejszy snapshot architektury ClickUp EVSO.

Podział ról między plikami:
- `EVSO_ClickUp_Context_Snapshot.md` = szeroki opis struktury workspace i operacyjnej architektury ClickUp,
- `EVSO_Lead_Intake_Dataflow_Context.md` = szczegółowy opis warstwy wejścia danych, lejka zgłoszeń i tego, jak dane trafiają do systemu.

---

Koniec pliku.
