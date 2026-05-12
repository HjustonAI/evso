# Milestone 2 — Definicja Problemu: Ciągłość Konwersacji z Klientem

**Dokument:** Problem Definition  
**Typ:** Koncept — bez specyfikacji rozwiązania  
**Dla kogo:** Architekt systemu, developer, stakeholderzy projektu

---

## Kontekst systemu

Firma operuje przez kilka stron internetowych. Każda strona ma przypisany własny adres email, formularze kontaktowe i jest powiązana z konkretną usługą. Formularze są zintegrowane z systemem CRM (ClickUp) przez warstwę automatyzacji (Make.com), która przetwarza zgłoszenia, wzbogaca je o klasyfikację AI i tworzy ustrukturyzowane taski.

Każdy task w CRM ma unikalny identyfikator (`LEAD-####`), zestaw pól strukturalnych (miasto, data, typ imprezy, dane kontaktowe, etc.) oraz pola wypełniane przez AI (klasyfikacja, podsumowanie). Task jest pojedynczym miejscem prawdy dla danego zapytania.

---

## Co działa — stan wyjściowy

Gdy klient wypełnia formularz na stronie → system automatycznie tworzy task w CRM z wypełnionymi polami i klasyfikacją AI. Klienci nie mają obowiązku wypełniania wszystkich deklarowanych pól, ponieważ większość z nich nie jest na obecnym etapie zaznaczona jako obowiązkowe w formularzach na stronach — taski są więc tworzone, ale często brakuje w nich kompletnych danych. Sama ścieżka systemowa jest deterministyczna, ujednolicona i nie wymaga ręcznej interwencji.

---

## Problem główny

**Klienci nie korzystają wyłącznie z formularzy i mogą mieszać kanały.**

Zapytania sprzedażowe trafiają do firmy wieloma kanałami, a wszystkie powinny prowadzić do jednego, spójnego miejsca w CRM. Rzeczywistość jest inna, klienci dodatkowo korzystają z form kontaktu w sposób bardzo chaotyczny (najpierw zadzwonią, potem napiszą maila, albo wypełnią formularz, a zaraz po tym napiszą dodatkowego maila). Rozwiązanie systemu musi być na to uodpornione i dopasowywać się do tych zachowań, ponieważ obecny mechanizm stron na to pozwala, a ich przebudowa póki co nie wchodzi w grę. 

### Kanał 1: Telefon
Klient dzwoni bezpośrednio. Proces połączenia telefonicznego jest **w pełni manualny** i pozostaje poza zakresem automatyzacji — pracownik po rozmowie ręcznie tworzy task w CRM. Ważne jest jednak to, że jeśli po ręcznym dodaniu takiego taska klient wyśle wiadomość e-mail dotyczącą tej samej sprawy, system musi umieć odnaleźć ten manualnie założony task i przypiąć do niego tego maila. System nie dąży do analizowania głosu.

### Kanał 2: Bezpośredni email
Klient pisze wiadomość na jeden z adresów emailowych firmy, nie korzystając z formularza. Wiadomość trafia do skrzynki pocztowej, ale nie do CRM. Żaden task nie powstaje automatycznie.

### Kanał 3: Odpowiedź emailowa — poza wątkiem
Klient, który wcześniej zgłosił się przez formularz (ma już task w CRM), kontaktuje się ponownie przez email — ale w nowy sposób:
- Pisze nową wiadomość z nowym tematem
- Odpowiada na inną wiadomość ze skrzynki (nie na wiadomość wysłaną z CRM)
- Używa innego urządzenia, przez co email nie jest powiązany z poprzednim wątkiem
- Wiadomość nie zawiera żadnego metadanego łączącego ją z istniejącym taskiem

Efekt: klient ma już task `LEAD-7861`, ale jego nowa wiadomość nie trafia do tego taska. Albo gubi się w skrzynce pocztowej, albo powstaje duplikat taska.

---

## Szczegółowa analiza problemu ciągłości

### Problem A — Fragmentacja historii konwersacji

Historia komunikacji z klientem jest rozproszona między:
- Taskiem w CRM (jeśli formularz został wypełniony)
- Skrzynką email (wiadomości nieprzetworzone przez system)
- Pamięcią pracownika (rozmowy telefoniczne)
- Potencjalnie wieloma taskami (duplikaty przy ponownym kontakcie)

Pracownik przed rozmową z klientem nie ma kompletnego obrazu dotychczasowej komunikacji w jednym miejscu.

### Problem B — Identyfikacja klienta ponad kanałami

Aby powiązać nową wiadomość z istniejącym taskiem, system musi wiedzieć, że nadawca tej wiadomości to ta sama osoba, która złożyła poprzednie zapytanie.

**Przyjęte uproszczenie:** klient jest identyfikowany wyłącznie po adresie email. Wiadomość przychodząca z adresu email, który figuruje w istniejącym tasku → trafia do tego taska. Wiadomość z nieznanego adresu email → traktowana jako nowe zapytanie i skutkuje nowym taskiem. Nie podejmujemy prób łączenia tożsamości klienta ponad różnymi adresami email — jest to świadomy kompromis upraszczający architekturę kosztem rzadkich przypadków, gdy klient zmienia adres kontaktowy.

Dodatkowe utrudnienie pozostające w zakresie problemu:
- Leady w bazie powinny pozostawać "wiecznie". Klienci mogą ponownie odezwać się np. po miesiącu lub po całym roku z nowym wyjazdem. Rozwiązanie musi to obsłużyć i rozpoznawać powrót klienta, aby leady nie stawały się jednorazowe (wtedy tworzymy nowe zapytanie, a nie wrzucamy z opóźnieniem wiadomości z nowego wydarzenia do archiwalnego taska z zeszłego roku).

### Problem C — Niepełna integracja emaili z CRM

CRM posiada wbudowany mechanizm **Email to Task** — możliwość dołączania wiadomości email bezpośrednio do konkretnego taska. Dodatkowo system umożliwia wysyłanie wiadomości do klienta z poziomu taska za pomocą skonfigurowanego adresu firmowego.

Częściowe rozwiązanie już istnieje: jeśli pracownik wyśle wiadomość do klienta z poziomu taska w CRM, a klient odpowie na tę konkretną wiadomość — odpowiedź automatycznie trafia z powrotem do właściwego taska. Ten wątek działa.

Problem polega na tym, że klienci nie zawsze odpowiadają w sposób przewidywalny:
- Piszą nową wiadomość zamiast odpowiadać na wysłaną
- Odpowiadają na inną wiadomość ze skrzynki (nie na wiadomość z CRM)
- Piszą z innego urządzenia lub klienta pocztowego, przez co wątek się urywa
- Piszą bezpośrednio na adres firmy, omijając cały mechanizm powiązania

W tych przypadkach wiadomość trafia do skrzynki pocztowej, a nie do taska — mimo że mechanizm Email to Task w CRM technicznie istnieje i jest dostępny.

### Problem D — Sugerowanie nazwy wątków w ClickUp

Istniejący system umożliwia **wysyłanie** emaili z poziomu taska w CRM do klienta na podany adres z wykorzystaniem automatycznego osadzania wątku. Co ważne, z punktu widzenia systemu odpowiedź klienta na ten email wraca aktualnie prawidłowo do taska (nawet jeśli w temacie emaila pojawi się automatyczne "Re:"). 

Tym, co jest problematyczne na ten moment w kwestii spójności, są wiadomości wychodzące (wysyłane "z palca" do zadania, by zacząć w ogóle wątek email w danym tasku). Konieczny jest mechanizm ułatwiający / sugerujący pracownikom to, jaki podać tytuł dla tworzonej nowej wiadomości ze strony zespołu do klienta.

### Problem E — Brak powiadamiania pracownika o nowych wiadomościach

Nawet jeśli wiadomość od klienta zostanie powiązana z taskiem — przez automatyzację lub ręcznie — pracownik nie jest automatycznie powiadamiany o tym fakcie. Brak powiadomienia oznacza, że pracownik musi aktywnie sprawdzać skrzynkę pocztową i CRM, aby wykryć nowe wiadomości. Przy większej liczbie aktywnych leadów istnieje ryzyko, że wiadomość od klienta zostanie przeoczona lub obsłużona z opóźnieniem.

### Problem F — Skalowalność ręcznego zarządzania

Przy kilkudziesięciu lub kilkuset zapytaniach tygodniowo, każde ręczne powiązanie emaila z taskiem to koszt operacyjny. Pracownicy spędzają czas na administracji zamiast na sprzedaży.

---

## Granice problemu

### Co jest w zakresie problemu:

- Email → CRM (nowe zapytanie od nieznanego nadawcy → nowy task)
- Odpowiedź klienta na email wysłany z CRM → powrót do właściwego taska
- Odpowiedź klienta w nowym wątku lub z nowym tematem → powiązanie z istniejącym taskiem na podstawie adresu email
- Powiadamianie pracownika o nowej wiadomości w tasku
- Duplikaty tasków dla tego samego adresu email → wykrycie i obsługa

### Co jest poza zakresem problemu (na tym etapie):

- **Telefon** — w pełni manualny kanał; pracownik tworzy task ręcznie po rozmowie; brak automatyzacji tego kanału
- Chat na stronie (np. live chat, chatbot)
- Wiadomości z mediów społecznościowych (Instagram, Facebook, LinkedIn)
- SMS / WhatsApp
- Automatyczne generowanie ofert
- Automatyczne przypisywanie leadów do handlowców (jest już realizowane przez ClickUp automations)

---

## Zasoby dostępne do rozwiązania i Architektura

*Ważna uwaga: architektura działania EVSO powstawała historycznie w dużej mierze bez profesjonalnej wiedzy o systemach informatycznych ("jakoś sobie poradzili"). Zespół projektowy ma obecnie wolną rękę, aby podejść do tego od nowa w imię zasad logiki, sensu i tego, jak tego typu systemy powinno projektować się wydajnie, przejrzyście oraz w czysty sposób.*

**Obecna infrastruktura i architektura EVSO:**
- Mamy do dyspozycji kilka stron (na jednych `WP-Forms`, na innych `Forminator`).
- Formularze wysyłają webhooki z submisją do scenariusza w **Make.com**, który przy wykorzystaniu **Claude (Haiku)** obrabia dane i robi z nich Taska w ClickUp.
- Standardowo strony posiadają także skrzynki pocztowe w Dhosting (np. `kontakt@nazwastrony.pl`), a sam adres widnieje na stronach www w sekcji Kontakt, co gmatwa system - bo pozwala klientom na kontaktowanie się na skrzynkę, z pominięciem Make. Formularze dodatkowo (jako zabezpieczenie) wysyłają powiadomienie na tę skrzynkę ze swoim contentem.

**Środowisko testowe:**
Do wdrażania Milestone'a M2 wykorzystujemy gotowe i zintegrowane środowisko testowe:
- **1 wydzielony formularz** (w oparciu o Forminator), który można dowolnie i bez strat przestawiać.
- **Klon scenariusza i osobny webhook w Make**, wysyłający leada za pomocą **ClickUp** w wydzielone i puste miejsce przeznaczone na dewelopment (testowy Workspace w ClickUp).
- **Konto Gmail:** `automatyzacjaevso@gmail.com`. Testowy formularz (poza webhookiem w Make) przesyła powiadomienia właśnie na tę skrzynkę by unikać testowania na skrzynkach sprzedażowych. Mamy tutaj swobodę, ponieważ do ostatecznego rozwiązania możemy przyjąć standard polegający na systematycznym włączeniu Gmaila, po czym podepnie się po prostu właściwe skrzynki ze środowiska Live na wypracowaną procedurę.

**Narzędzia główne:**
- **ClickUp** (plan Business) — istniejące CRM, taski, automatyzacje, pola niestandardowe, historia aktywności, email composer w tasku, mechanizm Email to Task; firmowe adresy email już podpięte do ClickUp
- **ClickUp AI + Agents** — możliwość uruchamiania agentów AI wewnątrz ClickUp, rozumienie kontekstu tasku
- **Vibe Coding** — możliwość tworzenia małych aplikacji, skryptów, serwisów, które uzupełniają gotowe narzędzia
- **Developer (mid-level + silne LLM)** — może budować własne komponenty, promptować AI do złożonych zadań klasyfikacji i routingu

---

## Kryteria sukcesu — jak wygląda rozwiązany problem

Milestone 2 jest rozwiązany, kiedy spełnione są wszystkie poniższe warunki:

1. **Każda wiadomość emailowa od klienta trafia do właściwego taska w CRM** — niezależnie od tego, czy klient odpowiada w wątku, pisze nową wiadomość, zmienia temat czy używa innego urządzenia. Wyjątek: klient piszący z nieznanego adresu email → nowy task (świadomy kompromis).

2. **Klient bez istniejącego taska, który pisze email, automatycznie dostaje nowy task** — analogicznie do zapytania przez formularz.

3. **Historia konwersacji z klientem jest kompletna i widoczna w jednym tasku** — pracownik, otwierając task, widzi całą dotychczasową komunikację: formularz, emaile wysłane i odebrane.

4. **Pracownik może obsłużyć cały kontakt z klientem nie wychodząc z ClickUp** — czyta wiadomości, odpowiada, widzi historię — wszystko w jednym miejscu.

5. **Duplikaty są minimalizowane** — jeśli klient pisze drugi raz z tego samego adresu email, system rozpoznaje istniejący task i nie tworzy nowego.

6. **Pracownik jest powiadamiany o nowej wiadomości w tasku** — system aktywnie informuje pracownika, że klient odpowiedział, bez konieczności samodzielnego sprawdzania skrzynki i CRM.

7. **Rozmowa telefoniczna jest rejestrowana ręcznie** — pracownik tworzy task manualnie po rozmowie; nie jest wymagana żadna automatyzacja tego kanału.

---

## Czego problem NIE zakłada

- Że klient zawsze będzie zachowywał się przewidywalnie
- Że każdy email da się jednoznacznie przypisać do jednego taska
- Że zero ręcznych operacji jest możliwe przy telefonach
- Że wszystkie przypadki brzegowe można zautomatyzować bez kompromisów

Rozwiązanie musi być **odporne na nieoczekiwane zachowanie klienta**, a tam gdzie automatyzacja jest niemożliwa — **ułatwiać pracę ręczną** zamiast ją utrudniać.

---

*Dokument opisuje wyłącznie problem. Nie zawiera sugestii co do architektury rozwiązania, narzędzi ani kolejności implementacji.*
