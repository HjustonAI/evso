# Jak rozwiązujemy ciągłość konwersacji emailowej w EVSO

**Cel dokumentu**: czytelne wytłumaczenie, jak działa zaprojektowane rozwiązanie. Nie zastępuje dokumentu architektury (`EVSO_M2_Architecture_Meeting.md`) — uzupełnia go w formie narracyjnej, bez decyzji w tabelach.

---

## W jednym akapicie

Każdy email od klienta — niezależnie czy odpowiada w wątku, pisze nową wiadomość, zmienia temat czy odzywa się po miesiącach — będzie automatycznie trafiał do właściwego taska w ClickUp. Operator nie wychodzi z ClickUp: dostaje powiadomienie, czyta, odpowiada w sekcji Email taska, a system pod spodem dba o to, żeby cała korespondencja żyła w jednym miejscu. Tam, gdzie system nie potrafi jednoznacznie rozstrzygnąć, do którego taska należy wiadomość — pyta operatora, zamiast zgadywać. To świadomy wybór: lepiej, żeby człowiek raz na jakiś czas kliknął, niż żeby dwa różne zapytania klienta posklejały się w jeden task.

---

## Problem w skrócie

EVSO traci spójność rozmów z klientami, bo komunikacja idzie różnymi drogami. Formularz na stronie tworzy task automatycznie (to działa od M1). Ale klient potem może odpowiedzieć emailem, który nie nawiązuje do żadnej wcześniejszej wiadomości z ClickUp. Może napisać z innego urządzenia. Może odezwać się po dwóch miesiącach z nowym tematem. W każdym z tych przypadków — bez M2 — wiadomość ląduje w skrzynce Gmail, ale nigdy w tasku ClickUp. Operator albo musi sam ją odszukać i przekleić, albo gubi ją zupełnie.

M2 zamyka tę lukę.

---

## Trzy warstwy systemu

Rozwiązanie ma trzy fizyczne warstwy, z których każda ma jedną odpowiedzialność.

**Gmail jest transportem.** Tu fizycznie żyją surowe wiadomości — przychodzące i wychodzące. Gmail nie wie nic o taskach. Jego rola to skrzynka i archiwum.

**Make.com jest orchestratorem.** To on obserwuje Gmaila, czyta nagłówki, decyduje, do którego taska należy wiadomość, sprawdza czy już jej nie przetwarzał, woła API ClickUp, dba o idempotencję. Make trzyma stan techniczny (jakie wiadomości już widzieliśmy, jakie wysłaliśmy z którego taska), ale nie trzyma stanu biznesowego.

**ClickUp jest źródłem prawdy biznesowej i interfejsem operatora.** Tu mieszka task — jego status, custom fieldy, komentarze, sekcja Email z wątkiem korespondencji, assignee. Operator pracuje wyłącznie w ClickUp. Wychodzące maile leci się natywnym Email composerem z taska — to działa już dziś.

Każda warstwa robi to, w czym jest najlepsza, i nic poza tym. Gmail nie próbuje zrozumieć, co to jest task. ClickUp nie watchuje przychodzących maili (świadomie wyłączamy mu Email-to-Task, żeby nie konkurował z Make). Make nie jest UI dla operatora — operator nigdy nie loguje się do Make.

---

## Co się dzieje, gdy klient pisze maila

Cztery scenariusze, które pokrywają cały zakres problemu.

### Scenariusz 1 — klient odpowiada w wątku na maila wysłanego z ClickUp

Operator wysłał wiadomość z taska `LEAD-7861`. ClickUp wysłał ją przez Gmail z charakterystycznym `Reply-To` zawierającym task ID. Klient klika „Odpowiedz", jego klient pocztowy ustawia `To:` na ten adres. Wiadomość trafia z powrotem do Gmaila z nagłówkiem, który ClickUp natywnie rozpoznaje — i ClickUp sam dorzuca odpowiedź do sekcji Email taska `LEAD-7861`. Make widzi tę samą wiadomość w Gmailu, ale rozpoznaje ją jako „już obsłużoną przez ClickUp" (po sygnaturze `Reply-To`) i ją pomija.

To jest najczęstszy przypadek. Niemal nic nie kosztuje — ClickUp i tak by to zrobił sam.

### Scenariusz 2 — klient pisze nowego maila bez wcześniejszej historii

Klient wchodzi na stronę partytram.fun, znajduje adres kontakt@partytram.fun, pisze nowego maila. Make widzi nową wiadomość w Gmailu. Sprawdza:

- Czy nagłówek `In-Reply-To` wskazuje na jakąś wiadomość, którą wcześniej wysłaliśmy? Nie.
- Czy adres nadawcy (`klient@gmail.com`) pasuje do pola EMAIL na jakimś aktywnym tasku w ClickUp? Nie.

Skoro klient jest dla systemu nieznany — Make tworzy nowy task w liście NOWE FORMULARZE, dokładnie tak samo jak robi to pipeline formularzy. Treść maila idzie do AI Haiku, która ekstrahuje ustrukturyzowane pola (miasto, data, typ imprezy, dane kontaktowe). Powstały task wpada w istniejące automacje routujące do Kuby, Izy lub Wiktora. Operator dostaje powiadomienie ClickUp.

### Scenariusz 3 — klient odpowiada poza wątkiem

To jest kluczowy przypadek M2. Klient ma już taska `LEAD-7861` (np. zapytał miesiąc temu o party tram). Pisze nowego maila — z nowym tematem, bez `In-Reply-To`, może z innego urządzenia. Make sprawdza:

- `In-Reply-To` → nic nie pasuje.
- Sender → pole EMAIL → pasuje dokładnie do jednego aktywnego taska, `LEAD-7861`.

Match. Wiadomość trafia do `LEAD-7861` jako wpis w sekcji Email (jeśli ClickUp API to umożliwia) lub jako komentarz z załączonym `.eml` (fallback). Aktualny assignee taska dostaje `@mention`, czyli natywne powiadomienie ClickUp. Cała historia jest w jednym miejscu.

### Scenariusz 4 — klient ma kilka aktywnych tasków

Klient `klient@gmail.com` ma dwa otwarte taski — `LEAD-7861` o party tram i `LEAD-8003` o party boat. Pisze maila. Make matchuje sendera, ale znajduje dwóch kandydatów. Tu dopiero wchodzi AI: Claude Haiku dostaje treść wiadomości i metadane obu tasków (typ imprezy, miasto, data, status) i odpowiada strukturalnym JSON-em z wybranym `task_id` i poziomem pewności.

- Pewność ≥ 0.85 → automatyczny match do wybranego taska.
- Pewność 0.5–0.85 → komentarz „prawdopodobny match" w jednym tasku z linkami do drugiego, operator decyduje finalnie.
- Pewność < 0.5 lub Haiku nie umie zdecydować → wiadomość ląduje w specjalnej liście AMBIGUITY z dropdownem „Wybierz task docelowy"; operator klika, system finalizuje merge.

Próg 0.85 jest startowy — po czterech tygodniach produkcji robimy manualny audyt 50 auto-matchów i kalibrujemy. To jest świadoma decyzja: lepiej dać operatorowi trochę więcej pracy na początku niż ryzykować, że dwa różne zapytania posklejają się w jeden task. Krzyżowy wyciek danych klientów to gorsza katastrofa niż dodatkowa praca.

---

## Logika dopasowania — w trzech warstwach od najmocniejszej do najsłabszej

Hierarchia matchingu jest najważniejszą decyzją całego rozwiązania, więc zasługuje na osobny akapit. Idziemy od najbardziej deterministycznego do najbardziej probabilistycznego — i pierwsza warstwa, która zwróci wynik, wygrywa.

**Warstwa 1 — `In-Reply-To` w nagłówku wiadomości.** Każdy email ma unikalny `Message-ID`. Gdy klient odpowiada, jego klient pocztowy *powinien* (nie zawsze, ale często) ustawić `In-Reply-To` na `Message-ID` wiadomości oryginalnej. Make trzyma w swoim Data Store mapowanie „każdy `Message-ID`, który wysłaliśmy z ClickUp, → ID taska, z którego poszedł". Jeśli przychodząca wiadomość ma `In-Reply-To` pasujący do tego Data Store — match jest deterministyczny, pewność = 1.0, koniec dyskusji.

**Warstwa 2 — sender email kontra pole EMAIL na aktywnym tasku.** Jeśli warstwa 1 nie zadziałała, Make pyta ClickUp API: „znajdź taski w Space CRM, gdzie EMAIL = `nadawca@klient.pl` AND status jest aktywny". Status „aktywny" wyklucza `closed`, `cancelled`, `lost`, `won`, `archived` — bo klient wracający po zamkniętym procesie to praktycznie nowy lead, kontekst się zmienił. Jeśli wynik to dokładnie jeden task, match. Jeśli zero tasków, klient jest dla nas nowy, tworzymy nowy task. Jeśli więcej niż jeden — idziemy na warstwę 3.

**Warstwa 3 — Claude Haiku rozstrzyga niejednoznaczność.** Haiku dostaje treść wiadomości i metadane wszystkich kandydatów, i odpowiada JSON-em z wybranym `task_id` i pewnością 0–1. Tu zaczyna się gradient pewności i progi opisane wyżej.

Filozofia: **najpierw reguła, potem model.** AI używamy tylko tam, gdzie reguła nie ma odpowiedzi, i nawet wtedy z dolnym progiem pewności, poniżej którego pytamy człowieka. Większość ruchu — szacujemy 80% — obsłuży warstwa 1 i 2, bez wydawania ani jednego tokena.

---

## Co się dzieje, gdy system nie jest pewien

Architektura nie udaje, że zawsze potrafi zdecydować. Gdy AI zwraca niską pewność albo system widzi sygnał ostrzegawczy (np. shared-inbox-pattern jak `kontakt@firma.pl`, gdzie pod jednym adresem mogą siedzieć różne osoby), wiadomość trafia do specjalnej listy AMBIGUITY w ClickUp. Każdy wpis tam ma:

- pełną treść oryginalnej wiadomości,
- listę linków do tasków-kandydatów,
- powód wątpliwości od Haiku,
- dropdown custom field „Wybierz task docelowy" z opcjami: każdy z kandydatów albo „nowy task".

Gdy operator wybierze wartość w dropdownie, ClickUp emituje webhook, który wywołuje Make, a Make wykonuje finalny merge: dorzuca wiadomość do wybranego taska, kasuje task ambiguity, zapisuje mapowanie w Data Store dla przyszłej referencji. Jeśli ambiguity task wisi 24 godziny — automacja ClickUp wysyła `@mention` do Bartka. Po 72 godzinach — eskalacja wyżej. Klient nie powinien czekać.

---

## Co może to zabić — i jak się chronimy

Pre-mortem zespołu zwrócił dziesięć scenariuszy porażki. Najważniejsze pięć z mitygacjami:

**Auto-responder loop.** Klient ma OOO. Operator wysyła maila → automat odpowiada → my traktujemy jako nową wiadomość → operator/automat odpowiada → klient automat odpowiada → pętla. Ochrona: Make filtruje po nagłówkach `Auto-Submitted: auto-replied` i `Precedence: bulk` (RFC 3834). Auto-odpowiedzi trafiają do taska jako informacja, ale bez powiadomienia i bez dalszych akcji.

**Shared inbox sender.** Z `kontakt@firma.pl` pisze raz osoba A o jednej sprawie, raz osoba B o innej. Sender ten sam, wiadomości niepowiązane. Ochrona: Make rozpoznaje shared-inbox patterny (`kontakt@`, `info@`, `office@`, `biuro@`, `hr@`, `events@`) i nawet przy 1-task match wymusza eskalację do warstwy 3 — Haiku porównuje spójność tematyczną z istniejącym wątkiem.

**Sender spoofing / phishing.** Ktoś forguje sender = znany klient i wysyła phishing. Ochrona: Make czyta nagłówek `Authentication-Results` (Gmail go dorzuca). Wiadomości fail DMARC dostają marker `⚠️ NIEZWERYFIKOWANY NADAWCA`, nie idą w auto-match, lądują w AMBIGUITY.

**Cicha awaria Gmail Watch.** Gmail Push wymaga odnowienia subskrypcji co 7 dni. Bez tego — po tygodniu maile przestają przychodzić do Make i nikt o tym nie wie. Ochrona: scheduled scenario co 5 dni odnawiający `watch`, plus heartbeat scenario co godzinę, który porównuje liczbę wiadomości w Gmailu (z ostatnich 70 minut) z liczbą faktycznie przetworzoną w Data Store. Rozjazd > X% → alert.

**GDPR — prawo do usunięcia.** Klient prosi o usunięcie. Mamy: task w ClickUp, wpisy w Data Store, kopię w Gmail. Ochrona: runbook (procedura krok po kroku, nie automat — wymaga decyzji człowieka): kasujemy task w ClickUp (purge komentarzy + maili w taska), kasujemy wpisy w Data Store keyed by sender email, manualnie kasujemy w Gmail.

Reszta scenariuszy (race conditions, rate limits, klient z różnych adresów, klient odpowiadający po 200 dniach) jest w pełnym pre-mortem w dokumencie architektury — albo z mitygacją, albo świadomie zaakceptowana jako niska likelihood × niski impact.

---

## Co świadomie NIE rozwiązujemy

Kilka rzeczy zostawiamy na boku, świadomie. To nie są przeoczenia, to są decyzje.

Klient piszący z różnych adresów email do tej samej sprawy nie zostanie automatycznie złączony. Dokument problemu mówi to wprost — identyfikacja po adresie email jest naszym single source of truth. Jeśli `janusz@gmail.com` i `j.kowalski@firma.pl` to ta sama osoba, zobaczymy dwa taski. Operator może zmergować ręcznie.

Telefon nie jest zautomatyzowany. Pracownik tworzy taska ręcznie po rozmowie. Nie próbujemy przechwytywać rozmów ani transkrypcji.

Reaktywacja zamkniętego leada — gdy klient wraca po pół roku — tworzy nowy task, nie ożywia starego. Świadomie: kontekst po pół roku jest inny. Operator może oznaczyć w komentarzu jako reopened i ręcznie dolinkować stary task.

Czat na stronie, social media, SMS, WhatsApp — poza zakresem M2. Nie próbujemy.

---

## Co zostało do potwierdzenia, zanim ruszy build

Kilka decyzji wisi na założeniach, które trzeba zweryfikować empirycznie — to są godziny pracy, nie tygodnie. Przed pierwszym scenariuszem produkcyjnym musimy odpowiedzieć:

Czy ClickUp używa rozpoznawalnego znacznika (np. `Reply-To` z task ID lub własnego nagłówka) przy wysyłce z taska? To kluczowe — dzięki temu Make odróżnia wiadomości obsługiwane natywnie przez ClickUp od pozostałych. Test: 15 minut — wysłać testowego maila z taska, zajrzeć w nagłówki w Gmailu.

Czy ClickUp emituje webhook event z `Message-ID` przy wysłaniu maila z taska? Jeśli tak — Make subskrybuje i ma od ręki mapowanie do Data Store. Jeśli nie — fallback, Make watchuje folder Sent i parsuje. Oba realne, oba tanie.

Czy ClickUp API pozwala programatycznie wgrać istniejący email do natywnej sekcji Email taska? Jeśli tak — perfekcyjne UX (wszystko w jednej sekcji, niezależnie czy ClickUp dorzucił natywnie czy Make). Jeśli nie — fallback, komentarz z markerem 📧 i załącznikiem `.eml`. UX trochę gorsze, ale akceptowalne.

Czy pole EMAIL na taskach z M1 jest *zawsze* niepuste? Jeśli nie — warstwa 2 matchingu ma luki. Krótki audyt logów M1 to wyjaśni; w razie czego dorzuca się walidację po stronie M1.

---

## Krótkie podsumowanie filozofii

Trzy zasady, na których opiera się całe rozwiązanie.

Po pierwsze, **najpierw reguła, potem model**. AI używamy tylko tam, gdzie reguła naprawdę nie ma odpowiedzi. Większość ruchu nie kosztuje ani jednego tokena.

Po drugie, **idempotencja przed wszystkim**. Każda wiadomość ma `Message-ID`. Make wpierw sprawdza, czy już jej nie widział. Pisze do Data Store dopiero po udanej akcji w ClickUp. Sequential mode zapobiega race conditions. Bez tego — stan się rozjeżdża i system staje się niewykrywalnie zepsuty.

Po trzecie, **niepewność jest informacją, nie błędem**. Gdy system nie wie, do którego taska należy wiadomość, nie zgaduje — pyta operatora. Lepiej zrobić mniej, ale dobrze, niż auto-mergować dwóch różnych klientów w jeden task.

---

*Dokument towarzyszący architekturze. Pełna lista decyzji, ryzyk i niewiadomych z uzasadnieniami: `EVSO_M2_Architecture_Meeting.md`. Skład zespołu, który tę architekturę wypracował: `EVSO_M2_Team_Charter.md`. Definicja problemu: `EVSO_Milestone2_Problem_Definition.md`.*
