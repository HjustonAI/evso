# EVSO ClickUp Context Snapshot

**Purpose:** reusable context file for LLMs that need to understand how EVSO's ClickUp is structured and used operationally.  
**Source basis:** this file is derived **only from the supplied screenshots**.  
**Reliability note:** everything below is limited to what is visibly confirmed on the screenshots. Where a label is cut off or ambiguous, it is marked as **uncertain**.

---

## 1) High-level interpretation

EVSO uses ClickUp as a multi-layer operating system with at least four distinct domains:

1. **CRM / sales intake** — incoming leads and sales pipelines.
2. **Zlecenia / delivery** — confirmed or operationally processed events/orders.
3. **Zadania / personal execution** — person-specific internal task lists.
4. **Centrum dowodzenia / knowledge** — shared documentation and operating playbooks.

There is also a separate **Home / Inbox / Channels** layer used for notifications, communication, and workspace navigation.

---

## 2) Workspace / space structure

## Confirmed top-level spaces
- **All Tasks - EVSO** — aggregate/all-tasks space.
- **EVSO** — main operational space.
- **TESTOWE** — separate testing space.

## Primary space: EVSO

### Folder: CRM
Observed lists / lists-like objects:
- **NOWE ZAPYTANIA** — central raw lead intake list. Visible count on screenshot: **274**
- **SPRZEDAŻ KUBA** — sales pipeline for Kuba. Visible count: **303**
- **SPRZEDAŻ WIKTOR** — sales pipeline for Wiktor. Visible count: **227**
- **SPRZEDAŻ DAWID** — sales pipeline for Dawid. Visible count: **1** on one sidebar screenshot, but another screen suggests many more tasks in the list view. Treat the sidebar count as a snapshot, not a stable truth.
- **SPRZEDAŻ KRIS** — sales pipeline for Kris. Visible count: **58**
- **COLD CALL** — cold-call related list/process. Visible count: **3**
- **Callback 2025/2026** — callback list/checklist style object. Visible count: **76**
- **Callback 2026 KUBA** — callback list/checklist style object. Visible count: **41**
- **Sprzedaż Wiktor v2** — additional Wiktor sales workflow. Count not clearly visible.

### Folder: CENTRUM DOWODZENIA
Observed content:
- **WIKI PRACOWNIKA** — document/wiki hub (blue doc icon, verified badge visible)

### Folder: ZLECENIA
Observed list:
- **ZLECENIA** — operational orders/events list. Visible count: **294**
- Lock icon is visible next to the list name on at least one screenshot.

### Folder: ZADANIA
Observed lists:
- **KUBA ZADANIA** — visible count: **8**
- **IZA ZADANIA** — visible count: **14**

### Folder / list area: TESTOWY
Observed:
- **TESTOWY** — green folder
- **Pozostałe do przypilnowania** — checklist/list-like object under TESTOWY

---

## 3) Home / Inbox / communication layer

## Home panel
Observed items:
- **Inbox**
- **Replies**
- **Assigned Comments**
- **My Tasks**
- **More**

## Inbox classification tabs
Visible tabs inside Inbox:
- **All**
- **Primary**
- **Other**
- **Later**
- **Cleared**

## Favorites
Observed favorites:
- **SPRZEDAŻ DAWID**
- **ZLECENIA**

## Realizacje area
Observed:
- **ROZLICZENIA**

## Channels
Observed channels:
- **SPRZEDAŻ KUBA**
- **EVSO**
- **ZLECENIA**
- **CRM**
- **General - EVSO**
- **Welcome**

## Direct messages
Observed:
- **Dawid**
- **EVSO — You**

**Interpretation:** ClickUp Home/Inbox is used as a personal/internal notification and communication layer, separate from the actual CRM task architecture.

---

## 4) EVSO space overview / dashboard layer

A screenshot of the **EVSO** space shows multiple overview widgets.

## Top navigation tabs seen on EVSO space
- **Channel**
- **Overview**
- **Board**
- **List**
- **Calendar**
- **Gantt**
- **Table**
- **+ View**

## Overview widgets observed
### Workflow by Status
Pie chart showing cross-workspace or cross-space status distribution. Readable labels include:
- **NOWE ZAPYTANIA 274**
- **NOWE ZAPYTANIE 85**
- **WYSŁANO OFERTE 1K 55**
- **WYSŁANO OFERTE KP 61**
- **NOWY KLIENT 39**
- **ZLECENIA 20**
- **NIE ODBIERAŁ 19**
- **PRÓBA KONTAKTU 1 31**
- **PRZEGRANY 195**
- **ANULOWANY 32**
- **SPRZEDAŻ 48H 68**
- **2025 211**

Some labels may be truncated slightly by the screenshot, but the status names above are visibly legible enough to be useful.

### Recent
Recent objects shown include:
- **SPRZEDAŻ KUBA** in CRM
- **CRM** in EVSO
- **ZLECENIA** in EVSO
- **ZLECENIA** in ZLECENIA
- **NOWE ZAPYTANIA** in CRM
- **List** in TESTOWY
- **TESTOWY** in EVSO
- **SPRZEDAŻ WIKTOR** in CRM

### Docs widget
Observed docs from **WIKI PRACOWNIKA**:
- **GDZIE JAK SIĘ ROZLICZAMY**
- **COLD CALL INSTRUKCJA**
- **STAWKI POZYCJE**
- **Wzór Tabelki Rozliczenia**
- **SAMOUCZEK EVENT CLICKUP**
- **CHECK LISTA REALIZATORZY**
- **ZADANIA IZKA WALIZKA**
- **ZADANIA EVSIAKI SELL**

### Folders widget
Visible folder shortcuts:
- **ZLECENIA**
- **CRM**
- **CENTRUM DOWODZENIA**
- **ZADANIA**
- **TESTOWY**

---

## 5) CRM architecture

## 5.1 NOWE ZAPYTANIA — central intake list

This appears to be the **main raw inbound lead reservoir** before leads are distributed into seller-specific pipelines.

### Breadcrumb / location
- **EVSO / CRM / NOWE ZAPYTANIA**

### Views visible
- **Channel**
- **List**
- **Board**
- **+ View**

### List group / status header
- **NOWE ZAPYTANIA** — visible item count: **274**

### Columns visible in the list view
- **Name**
- **DATA EVENTU**
- **Assignee**
- **Due date**
- **Date created**
- **Custom Task ID**

### Naming convention of tasks in NOWE ZAPYTANIA
The screenshot shows a very consistent pattern:

`[DATE or BRAK DATY] | [CITY or WYBIERZ MIASTO] | [SERVICE] | [CUSTOMER NAME]`

Examples:
- `BRAK DATY | KRAKOW | TRAM | domingo fernandez`
- `BRAK DATY | WROCLAW | BOAT | Ola`
- `24-01-2026 | WYBIERZ MIASTO | TRAM | Arleta Szymańska`
- `BRAK DATY | WYBIERZ MIASTO | BOAT | Ewa Łoboda`

### Task ID convention
The list shows a **Custom Task ID** column using the pattern:
- `LEAD-4750`
- `LEAD-4885`
- `LEAD-5101`
- etc.

**Interpretation:** leads are explicitly tracked with a dedicated custom ID namespace `LEAD-####`.

---

## 5.2 Structure of an individual lead task in NOWE ZAPYTANIA

One lead task is shown open in detail view.

### Header / task identity
- Title example: `BRAK DATY | KRAKOW | TRAM | domingo fernandez`
- Custom task ID visible next to the task type: **LEAD-4750**

### Standard task metadata observed
Left/center panel:
- **Status**
- **Dates**
- **Time estimate**
- **Tags**
- **Assignees**
- **Priority**
- **Track time**

At the moment of the screenshot many of those core fields are empty except status.

### Status
- Visible value: **NOWE ZAPYTANIA**

### Description usage
The task description contains the **original customer inquiry** in natural language.  
In the example, it is a long English message describing a bachelor party enquiry, with missing exact date but including constraints and requested details.

**Interpretation:** the original inbound message is preserved in the task body/description.

### Custom fields visible on the task
Observed field names:
- **MIASTO**
- **PAKIET**
- **POZYSKANIE**
- **TELEFON**
- **TYP IMPREZY**
- **USŁUGA**
- **GODZINA EVENTU**
- **DATA EVENTU**
- **EMAIL**
- **ILOŚĆ OSÓB**
- **JEDNOSTKA**

### Example populated values visible in the open task
- **POZYSKANIE:** `TRAMPARTY.PL`
- **TELEFON:** `628057720`
- **USŁUGA:** `TRAM`
- **GODZINA EVENTU:** `20:00–22:00`
- **EMAIL:** `domfermun@gmail.com`
- **ILOŚĆ OSÓB:** `10`

Fields visibly empty in that example:
- **MIASTO**
- **PAKIET**
- **TYP IMPREZY**
- **DATA EVENTU**
- **JEDNOSTKA**

### Right-side Activity panel
The right panel is clearly labeled **Activity**.

Observed characteristics:
- chronological event history is shown there
- example log lines include:
  - task creation
  - "Show more"
  - setting the EMAIL field
- this panel is also used as the point of entry for email composition

### Email composer inside task
At the bottom-right, the task includes an email composer with:
- **From** dropdown
- **To**
- **Cc / Bcc**
- **Suggested emails** derived from the task (in the screenshot the suggestion is the same as the EMAIL field)
- **Subject**
- signature option
- send controls
- tab indicator showing **Email**

**Interpretation:** the CRM lead task is already set up as a place where outbound client emails can be sent directly from ClickUp, using data from task fields.

---

## 5.3 SPRZEDAŻ KUBA — sales pipeline list

### Breadcrumb / location
- **EVSO / CRM / SPRZEDAŻ KUBA**

### Top tabs / views visible
- **NOWE LEADY**
- **DZISIAJ DO ZROBIENIA**
- **ZALEGŁE**
- **FIRMÓWKI**
- **TABLICA**
- **+ View**

This suggests that a single seller list has multiple operational views layered on top of the same underlying dataset.

### Columns visible in the list/table view
- **Name**
- **WARTOŚĆ**
- **DATA**
- **Total tim...** (truncated; likely total time)
- **STATUS OBSŁUGI**
- **Due date**
- **MIASTO**
- **USŁUGA**

### Group/status sections visible
- **NOWE ZAPYTANIE** — visible count: **12**
- **WYSŁANO OFERTE 1K** — visible count: **55**

Other groups likely exist further down, but only these are clearly visible on the screenshot.

### Conventions visible in rows
Task names still follow a date/city/service/customer pattern, for example:
- `01-04-2026 | WROCŁAW | TRAM | Barbara Stąpor`
- `25-04-2026 | SZCZECIN | BOAT | Kinga Jeż | Oferta do wygenerowania`
- `11-07-2026 | Wrocław | BOAT | Nadia Motriuk`
- `BRAK DATY | WROCLAW | TRAM | integracyjne.pl`

### Field/value patterns visible
- **WARTOŚĆ** is represented with star icons
- **STATUS OBSŁUGI** contains values such as **PRZEPALONY**
- **MIASTO** is a colored label/tag column
- **USŁUGA** is a colored label/tag column, with values such as **TRAM** or **BOAT**

**Interpretation:** after raw intake, leads appear to be moved or copied into owner-specific sales pipelines where sales work is managed with additional handling status, due dates, and prioritization/value scoring.

---

## 5.4 SPRZEDAŻ DAWID — board-style sales pipeline

### Breadcrumb / location
- **EVSO / CRM / SPRZEDAŻ DAWID**

### Views visible
- **Channel**
- **TABLICA**
- **List**
- one additional view label is partially visible/uncertain
- **+ View**

### Board columns / statuses visible
- **NOWE ZAPYTANIE**
- **WYSŁANO OFERTĘ**
- **PRZEDZWONKA**
- **ZASTANAWIA SIĘ**
- **SZANSA SPRZEDAŻY**
- rightmost column label is cut off and therefore **uncertain** (starts with `POCZEK...`)

### Example card anatomy
One card in `NOWE ZAPYTANIE` shows:
- title format with date, city, service, name:
  - `11-07-2026 | Wrocław | BOAT | Nadia Motriuk`
- status label displayed inside card: **NOWE ZAPYTANIE**
- colored field chips:
  - **BOAT**
  - **WROCŁAW**
  - **FIRMÓWKA**
  - **PARTYBOAT.FUN**
- numeric/text entries:
  - phone number `+48884625113`
  - value/quantity `40`

**Interpretation:** seller-specific pipelines can also be managed in Kanban form with rich card metadata shown directly on the board.

---

## 6) ZLECENIA architecture

## 6.1 ZLECENIA list / delivery-operational workflow

### Breadcrumb / location
- **EVSO / ZLECENIA / ZLECENIA**

### Tabs / views visible
- **Channel**
- **ZLECENIA**
- **ROZLICZENIA JEDNOSTKI**
- **Calendar**
- **Board**
- **Dashboard**
- **Feedback Form**
- **+ View**

### Status groups visible in the left-side grouped list
- **ZLECENIA BEZ ZALICZEK** — count: **4**
- **REZERWACJA JEDNOSTKI** — count: **8**
- **ZLECENIA** — count: **20**
- **REZYGNACJA** — count: **13**
- **ANULOWANY** — count: **32**
- **DO ROZLICZENIA** — count: **5**
- **ZREALIZOWANE** — count: **10**
- **2025** — count: **0**
- **PRZEGRANY** — count: **1**

### Columns visible
- **Name**
- **Due date**
- **Assignee**
- **OBSŁUGA**
- **MIASTO**
- **PAKIET**
- one date-related column on the far right with truncated label `DATA SPR...` — **uncertain exact full name**

### Row/task naming convention
Still follows a structured pattern:
`[DATE] | [CITY] | [SERVICE] | [PERSON/CLIENT] | [PACKAGE]`

Examples:
- `11-04-26 | WROCŁAW | TRAM | Aleks Meredyk | PARTY`
- `02-05-26 | BYDGOSZCZ | BOAT | Maria Musiał | FUN BEZOBSŁUGOWY`
- `10-04-2026 | KRAKÓW | GEAORGE | NATALIA HOREZGA | PARTY`  
  (exact spelling preserved from screenshot even if it may reflect a typo in the live system)

There is also a task called:
- **TABELKA KOSZTY**

### Field/value patterns visible
- **OBSŁUGA** values include: `BEZOBSŁUGOWY`
- **MIASTO** values shown as colored labels: `WROCŁAW`, `BYDGOSZCZ`, `KRAKÓW`
- **PAKIET** values shown as colored labels: `PARTY`, `FUN`

**Interpretation:** ZLECENIA is the post-sales / operational execution domain for concrete booked events and their downstream handling, accounting, and completion.

---

## 7) Documentation / procedural artifacts inside ClickUp

The screenshots reveal that EVSO uses ClickUp not only for tasks but also for internal documentation and process artifacts.

## Knowledge base / wiki
- Main hub: **WIKI PRACOWNIKA**

## Confirmed documents / artifacts
- `GDZIE JAK SIĘ ROZLICZAMY`
- `COLD CALL INSTRUKCJA`
- `STAWKI POZYCJE`
- `Wzór Tabelki Rozliczenia`
- `SAMOUCZEK EVENT CLICKUP`
- `CHECK LISTA REALIZATORZY`
- `ZADANIA IZKA WALIZKA`
- `ZADANIA EVSIAKI SELL`

## Non-ClickUp visual process artifact included in screenshots
One screenshot shows a process graphic titled **CHECK EVENTÓW**, with day-by-day steps for operational checking.  
This indicates the company also stores/uses visual operational process maps alongside task structures.

---

## 8) Structural conventions inferred from screenshots

The following conventions are visible strongly enough to be treated as real architectural patterns:

### A. Centralized raw intake
`CRM / NOWE ZAPYTANIA` acts as the central intake reservoir for new leads.

### B. Seller-specific downstream pipelines
There are separate sales lists such as:
- SPRZEDAŻ KUBA
- SPRZEDAŻ WIKTOR
- SPRZEDAŻ DAWID
- SPRZEDAŻ KRIS

This suggests ownership-based sales processing after intake.

### C. Strong naming convention
Task names embed operationally important metadata in the title:
- date or missing-date marker
- city
- service type
- client/customer name
- sometimes package or extra note

### D. Dedicated lead ID namespace
Leads have a custom ID format:
- `LEAD-####`

### E. Lead task as single source of truth
An open lead task combines:
- original inquiry text
- structured custom fields
- activity log
- email sending interface

### F. Separation between sales and operations
CRM and ZLECENIA are separate folders/process domains.

### G. Internal comms are native to ClickUp
Inbox, channels, comments, activity, and direct messages are clearly used as part of daily operations.

---

## 9) Machine-friendly snapshot

```yaml
workspace:
  name: EVSO
  top_level_spaces:
    - All Tasks - EVSO
    - EVSO
    - TESTOWE

main_space_EVSO:
  folders:
    CRM:
      lists:
        - NOWE ZAPYTANIA
        - SPRZEDAŻ KUBA
        - SPRZEDAŻ WIKTOR
        - SPRZEDAŻ DAWID
        - SPRZEDAŻ KRIS
        - COLD CALL
        - Callback 2025/2026
        - Callback 2026 KUBA
        - Sprzedaż Wiktor v2
    CENTRUM_DOWODZENIA:
      docs:
        - WIKI PRACOWNIKA
    ZLECENIA:
      lists:
        - ZLECENIA
    ZADANIA:
      lists:
        - KUBA ZADANIA
        - IZA ZADANIA
    TESTOWY:
      contents:
        - Pozostałe do przypilnowania

crm:
  intake_list:
    name: NOWE ZAPYTANIA
    purpose: raw inbound leads
    id_pattern: LEAD-####
    title_pattern: "[DATE|BRAK DATY] | [CITY|WYBIERZ MIASTO] | [SERVICE] | [NAME]"
    key_fields:
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
      - original inquiry text
      - activity log
      - outbound email composer

  seller_lists:
    - SPRZEDAŻ KUBA
    - SPRZEDAŻ WIKTOR
    - SPRZEDAŻ DAWID
    - SPRZEDAŻ KRIS

operations:
  folder: ZLECENIA
  main_list: ZLECENIA
  status_groups:
    - ZLECENIA BEZ ZALICZEK
    - REZERWACJA JEDNOSTKI
    - ZLECENIA
    - REZYGNACJA
    - ANULOWANY
    - DO ROZLICZENIA
    - ZREALIZOWANE
    - 2025
    - PRZEGRANY

communication:
  home_items:
    - Inbox
    - Replies
    - Assigned Comments
    - My Tasks
  inbox_tabs:
    - All
    - Primary
    - Other
    - Later
    - Cleared
  channels:
    - SPRZEDAŻ KUBA
    - EVSO
    - ZLECENIA
    - CRM
    - General - EVSO
    - Welcome
```

---

## 10) Safe usage notes for future LLM context

When this file is provided to an LLM, it should assume:

1. **NOWE ZAPYTANIA** is the raw lead intake area.
2. **LEAD-####** is the core lead identifier format.
3. Lead tasks likely combine:
   - inbound message
   - structured sales fields
   - outbound email handling
4. Sales work is segmented by owner/person (`SPRZEDAŻ KUBA`, `SPRZEDAŻ WIKTOR`, etc.).
5. Operational delivery lives in **ZLECENIA**, not in raw intake.
6. **WIKI PRACOWNIKA** is the main documentation hub.
7. The Inbox is an internal notification/communication surface, not the authoritative data store for lead structure.

---

## 11) What is confirmed vs inferred

### Directly confirmed by screenshots
- names of main spaces/folders/lists/channels/docs shown above
- visible counts shown near many lists/status groups
- `LEAD-####` custom task ID format
- custom fields inside the open lead task
- activity panel + email composer inside the task
- separate CRM and ZLECENIA domains
- seller-specific lists
- presence of wiki/docs inside ClickUp

### Reasonable but still inferred
- `NOWE ZAPYTANIA` is the central intake reservoir
- seller lists represent owner-specific handling stages after intake
- ZLECENIA is the post-sales operational delivery space
- task title structure is intentionally standardized and meaningful

These inferences are strong and practical, but they are still interpretations layered on top of the visible screenshots.

---

End of context snapshot.
