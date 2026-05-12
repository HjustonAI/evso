# EVSO — Lead Intake Automation: Phase 1 Implementation Guide

**Version:** 1.0
**Date:** 2026-03-31
**Scope:** Intake automation for website form and email channels
**Target reader:** Single mid-level engineer with Make.com and ClickUp experience
**Estimated effort:** 12 engineer-days

**Expert panel validation:** All design decisions in this document were validated against four specialist perspectives — ClickUp architecture (Marta Kurek), Make automation (Tomasz Sowa), AI reliability (Aleksander Bryła), and sales process design (Karolina Wysocka). Scope was reduced where any expert raised an objection. See "Deferred to Phase 2" at the end.

---

## Section 1 — Current State Analysis

### How inquiries currently enter the system

EVSO receives event inquiries through two confirmed channels:

**Channel A: Website forms.** Three branded domains — `partytram.fun`, `partyboat.fun`, `busparty.fun` — each host a WPForms-powered inquiry form. The form collects: city (dropdown, locative Polish forms like "w Krakowie"), event type (dropdown: Wieczór kawalerski, Wieczór panieński, Urodziny, Impreza firmowa), package (dropdown: FUN, PARTY, LUX, INDYWIDUALNY), customer name, email, phone, date, time, number of people, and a free-text field ("Opisz swój pomysł"). Form submissions trigger a minimal Make.com scenario: `WPForms submission → Make → ClickUp: Create Task`. The task lands in `EVSO / CRM / NOWE ZAPYTANIA` and receives a `LEAD-####` custom ID.

**Channel B: Direct emails.** Customers also email brand addresses (`kontakt@partyboat.fun`, etc.) or individual salespeople directly (`dawid.cybulkin@evso.pl`, `krzysztof.ksiazek@evso.pl`). Email inquiries are semi-structured or fully unstructured — they may contain some, all, or none of the data points that the form captures. There is no confirmed automation for importing email inquiries into ClickUp. The current handling appears to be manual or semi-manual: someone reads the email, creates a task, and copies the content into the task description.

### Where friction exists today

1. **Email intake is manual.** Every email inquiry requires a human to read it, create a ClickUp task, populate fields, and compose the task title in the naming convention. At ~100 inquiries/week with a meaningful portion arriving via email, this is the largest single source of repetitive manual work.

2. **No classification of inquiry type.** Hot leads, incomplete leads, general information requests, and spam all land in NOWE ZAPYTANIA with the same status. The sales team must open and read each one to determine what it is. There is no quick way to sort or filter by inquiry quality.

3. **City normalization is unverified.** The form sends locative Polish forms ("w Krakowie") but ClickUp stores operational short forms ("KRAKÓW"). It is not confirmed that the current Make scenario performs this mapping. If it doesn't, MIASTO fields may be empty or inconsistently populated.

4. **Incomplete leads are not flagged.** An inquiry missing date, city, or number of people looks identical to a complete one in the task list. The sales team discovers completeness gaps only after opening the task — wasting time on leads that need follow-up before any offer can be prepared.

5. **Form-to-ClickUp field mapping is minimal.** It is not confirmed that the current Make scenario populates all 11 Custom Fields. Some fields (MIASTO, TYP IMPREZY, PAKIET) may not be set automatically, forcing manual population.

6. **Service type is inferred, not captured.** The form does not have a visible "service type" selector. The domain (partytram = TRAM, partyboat = BOAT, busparty = BUS) implies the service, but there is no confirmed mechanism that maps this to the USŁUGA field in ClickUp.

7. **No AI assistance on any step.** Every interpretation, extraction, and classification decision is made manually by a human.

### What works well and must be preserved

1. **Task naming convention** — `[DATE|BRAK DATY] | [CITY|WYBIERZ MIASTO] | [SERVICE] | [NAME]` is excellent. It gives instant operational context in list views. Must be preserved exactly.

2. **LEAD-#### custom ID system** — stable, unique, and already integrated into the team's workflow.

3. **NOWE ZAPYTANIA as single intake reservoir** — correct architectural choice. All raw leads enter one place before downstream routing.

4. **Seller-specific pipeline lists** (SPRZEDAŻ KUBA/WIKTOR/DAWID/KRIS) — the separation between intake and owner-specific sales work is sound.

5. **Email composer in task** — the ability to send emails from within a task, with the EMAIL field suggesting the recipient, is a critical operational capability.

6. **Description as original message archive** — preserving the full original inquiry text in the task description is valuable for context and auditing.

7. **CRM ↔ ZLECENIA separation** — clean boundary between sales and operations.

### Data captured vs. missing

**Currently captured (11 Custom Fields):** MIASTO, PAKIET, POZYSKANIE, TELEFON, TYP IMPREZY, USŁUGA, GODZINA EVENTU, DATA EVENTU, EMAIL, ILOŚĆ OSÓB, JEDNOSTKA.

**Missing and needed:**

| Gap | Why it matters |
|-----|---------------|
| Lead classification (hot / incomplete / info request / spam) | Sales team cannot prioritize without opening every task |
| Completeness score | No way to filter leads that are ready to work vs. need follow-up |
| Source channel (form vs. email) | Cannot measure channel performance or tailor handling |
| AI-generated summary | Sales team must read full description to understand each inquiry |
| AI confidence indicator | No way to know if AI-populated data needs human verification |

---

## Section 2 — Proposed Solution Architecture

### End-to-end flow (numbered sequence)

**Path A — Form inquiry:**

1. Customer submits form on `partytram.fun`, `partyboat.fun`, or `busparty.fun`
2. WPForms generates a submission event
3. Make.com Scenario 1 ("Form Intake") receives the submission via webhook
4. Make normalizes city value using a static mapping table (e.g., "w Krakowie" → "KRAKÓW")
5. Make determines service type from the source domain (partytram → TRAM, partyboat → BOAT, busparty → BUS)
6. Make calls Anthropic API (Claude Haiku) to classify the inquiry and generate a short summary
7. Make calculates a completeness score based on which fields are filled
8. Make creates a ClickUp task in NOWE ZAPYTANIA with all Custom Fields populated, task name following the existing convention, description containing the original message, and new AI fields filled
9. If the AI call fails: Make creates the task anyway with AI fields left blank and KLASYFIKACJA_AI set to "Do weryfikacji"

**Path B — Email inquiry:**

1. Customer sends email to a brand address or salesperson address
2. Make.com Scenario 2 ("Email Intake") detects the new email via Gmail Watch module
3. Make extracts email metadata (sender, subject, body text)
4. Make calls Anthropic API (Claude Haiku) to extract structured fields from the email body (city, event type, date, time, people count, package, service type, customer name, phone) and classify the inquiry
5. Make normalizes any extracted city value using the same static mapping table
6. Make calculates completeness score based on extracted fields
7. Make creates a ClickUp task in NOWE ZAPYTANIA with extracted fields mapped to Custom Fields, email body as description, and AI fields populated
8. If the AI call fails: Make creates the task with only email metadata (sender as EMAIL, subject + body as description) and KLASYFIKACJA_AI set to "Do weryfikacji"

### What changes relative to current state

- Form intake scenario is replaced with a richer version that normalizes data, determines service type, adds AI classification/summary, and ensures all Custom Fields are populated
- A new email intake scenario is added where none existed
- Five new Custom Fields are added to NOWE ZAPYTANIA (details in Section 3)
- No existing statuses, views, or downstream workflows are modified

### What deliberately stays the same

- NOWE ZAPYTANIA remains the sole intake list
- Task naming convention is preserved exactly
- LEAD-#### ID scheme is untouched
- Seller-specific pipeline lists are untouched
- ZLECENIA workflow is untouched
- Manual seller assignment remains (no automatic routing)
- All customer-facing communication remains human-initiated

### Explicit scope boundary

**This guide covers:**
- ClickUp field additions to NOWE ZAPYTANIA
- Two Make.com scenarios (form intake improvement, email intake)
- AI integration for classification, extraction, and summarization
- Testing of the above

**NOT covered (Phase 2):**
- Draft reply generation
- Automated seller routing by city or service type
- Follow-up reminder automations
- ZLECENIA workflow automation
- Customer-facing auto-responses
- Deduplication of leads across channels
- Analytics dashboards

---

## Section 3 — ClickUp Setup Guide

### 3.1 Lists / Folders — no changes

No new lists or folders are created. All changes apply to the existing list `EVSO / CRM / NOWE ZAPYTANIA`.

### 3.2 Custom Fields to add

Add the following 5 Custom Fields to the list `EVSO / CRM / NOWE ZAPYTANIA`. Navigate to: EVSO space → CRM folder → NOWE ZAPYTANIA list → click the `+` column header → "Create New Field."

**Field 1: ŹRÓDŁO_KANAŁ**

| Property | Value |
|----------|-------|
| Name | `ŹRÓDŁO_KANAŁ` |
| Type | Dropdown |
| Options | `Formularz`, `Email`, `Inny` |
| Location | NOWE ZAPYTANIA list |
| Purpose | Identifies whether the lead came from a website form or email |

**Field 2: KLASYFIKACJA_AI**

| Property | Value |
|----------|-------|
| Name | `KLASYFIKACJA_AI` |
| Type | Dropdown |
| Options | `Gorący lead`, `Niekompletny lead`, `Zapytanie ogólne`, `Spam`, `Do weryfikacji` |
| Location | NOWE ZAPYTANIA list |
| Purpose | AI-assigned classification of inquiry quality. "Do weryfikacji" is the fallback when AI is unavailable or confidence is low |

**Field 3: AI_PODSUMOWANIE**

| Property | Value |
|----------|-------|
| Name | `AI_PODSUMOWANIE` |
| Type | Short Text (or Long Text if Short Text has character limits that are too restrictive) |
| Location | NOWE ZAPYTANIA list |
| Purpose | 1-2 sentence AI-generated summary of the inquiry in Polish |

**Field 4: KOMPLETNOŚĆ**

| Property | Value |
|----------|-------|
| Name | `KOMPLETNOŚĆ` |
| Type | Number (integer, no decimals) |
| Location | NOWE ZAPYTANIA list |
| Purpose | Percentage (0-100) indicating how many key fields are populated. Helps sales team prioritize complete leads |

**Field 5: AI_CONFIDENCE**

| Property | Value |
|----------|-------|
| Name | `AI_CONFIDENCE` |
| Type | Dropdown |
| Options | `Wysoka`, `Średnia`, `Niska` |
| Location | NOWE ZAPYTANIA list |
| Purpose | Indicates how confident the AI classification is. Helps sales team know when to double-check AI results |

### 3.3 Existing Custom Fields — verify and document

Before building Make scenarios, open any task in NOWE ZAPYTANIA and verify the exact field names and types for these existing fields. Record the ClickUp field IDs (visible via ClickUp API or in the task URL structure) for use in Make modules:

1. `MIASTO` — verify type is Dropdown. Record all current dropdown options.
2. `PAKIET` — verify type is Dropdown. Expected options: FUN, PARTY, LUX, INDYWIDUALNY.
3. `POZYSKANIE` — verify type (likely Dropdown or Short Text). Record current values.
4. `TELEFON` — verify type (likely Short Text or Phone).
5. `TYP IMPREZY` — verify type is Dropdown. Expected options: Wieczór kawalerski, Wieczór panieński, Urodziny, Impreza firmowa. Check for additional options.
6. `USŁUGA` — verify type is Dropdown. Expected options: TRAM, BOAT. Check if BUS exists. If not, add it.
7. `GODZINA EVENTU` — verify type (likely Short Text or Time).
8. `DATA EVENTU` — verify type (likely Date).
9. `EMAIL` — verify type is Email.
10. `ILOŚĆ OSÓB` — verify type (likely Number or Short Text).
11. `JEDNOSTKA` — verify type and current options.

**Action item:** If `USŁUGA` dropdown does not contain `BUS` as an option, add it now. The `busparty.fun` domain needs to map to a valid value.

### 3.4 Statuses — no changes

The existing status structure in NOWE ZAPYTANIA is preserved. The new KLASYFIKACJA_AI field serves the classification purpose without modifying workflow statuses. This was a deliberate design decision: statuses control workflow (what to do next), while classification is data about the inquiry (what it is). Mixing them would create friction for the sales team.

### 3.5 ClickUp Automations — add one

Add one automation to NOWE ZAPYTANIA to help the sales team quickly spot leads that need attention.

**Automation 1: Flag low-confidence AI classifications**

| Property | Value |
|----------|-------|
| Name | `Flag AI Do Weryfikacji` |
| Location | NOWE ZAPYTANIA list |
| Trigger | When Custom Field `KLASYFIKACJA_AI` is set to `Do weryfikacji` |
| Condition | None |
| Action | Set Priority to `Urgent` (red flag) |

This ensures that tasks where AI could not classify (due to error or low confidence) are visually flagged for immediate human attention.

**Why only one automation:** Adding more automations (e.g., auto-assign based on city) was considered and rejected by the expert panel. Reason: automatic assignment changes the sales team's current manual allocation process. That change requires team buy-in and is better handled in Phase 2 after the intake layer is stable.

### 3.6 Views to configure

Create two new saved views in NOWE ZAPYTANIA to help the sales team work with the new data.

**View 1: NOWE - PRIORYTET**

| Property | Value |
|----------|-------|
| Name | `NOWE - PRIORYTET` |
| Type | List |
| Filter | KLASYFIKACJA_AI `is` `Gorący lead` OR KLASYFIKACJA_AI `is` `Do weryfikacji` |
| Sort | Date created, descending (newest first) |
| Group by | KLASYFIKACJA_AI |
| Visible columns | Name, KLASYFIKACJA_AI, KOMPLETNOŚĆ, MIASTO, USŁUGA, DATA EVENTU, AI_PODSUMOWANIE, Date created |

**View 2: NOWE - KOMPLETNOŚĆ**

| Property | Value |
|----------|-------|
| Name | `NOWE - KOMPLETNOŚĆ` |
| Type | List |
| Filter | KLASYFIKACJA_AI `is not` `Spam` |
| Sort | KOMPLETNOŚĆ, descending (most complete first) |
| Group by | ŹRÓDŁO_KANAŁ |
| Visible columns | Name, KOMPLETNOŚĆ, KLASYFIKACJA_AI, MIASTO, USŁUGA, TYP IMPREZY, PAKIET, DATA EVENTU, EMAIL, TELEFON |

---

## Section 4 — Make.com Scenario Guide

### Scenario 1: Form Intake (Enhanced)

```
Scenario: EVSO - Form Intake Enhanced
Purpose: Replaces the existing minimal WPForms→ClickUp scenario with a version that normalizes data, classifies via AI, and fully populates all Custom Fields
Trigger: Webhook — Custom Webhook (receives WPForms submission)
```

**Preliminary setup:** Create a new Make.com scenario. Do NOT disable the existing WPForms scenario until this one is fully tested. Once validated, disable the old one.

#### Module 1 — Webhooks: Custom Webhook

```
Type: Webhooks > Custom Webhook
Settings:
  - Webhook name: "EVSO Form Intake"
  - Data structure: Define manually or auto-detect from first WPForms test submission
Output variables:
  - form_name (string) — name of submitting form
  - form_fields (object) — all form field values
  - webhook_url (string) — the URL of the submitting page (used to determine domain/brand)
Notes:
  - After creating the webhook, copy its URL
  - In WPForms, replace the existing Make webhook URL with this new one
  - Test by submitting the form once — Make should show the incoming data structure
```

**Verify in WPForms:** Check how form data is sent. WPForms may use its own Make/Integromat module or a generic webhook. Adapt the trigger module accordingly. If WPForms uses the native Make module (`WPForms > Watch Form Submissions`), use that instead of a custom webhook.

#### Module 2 — Tools: Set Multiple Variables (Normalize City)

```
Type: Tools > Set Multiple Variables
Purpose: Map locative Polish city names to ClickUp-compatible values
Settings:
  Variable 1:
    Name: city_normalized
    Value: Use an IF/SWITCH chain:
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

  Variable 2:
    Name: service_type
    Value: Derive from webhook URL or form source:
      {{if(contains(form_fields.source_url; "partytram"); "TRAM";
        if(contains(form_fields.source_url; "partyboat"); "BOAT";
        if(contains(form_fields.source_url; "busparty"); "BUS";
        "NIEZNANA")))}}

  Variable 3:
    Name: task_date_display
    Value: Format the date for the task title:
      {{if(form_fields.data != ""; formatDate(form_fields.data; "DD-MM-YYYY"); "BRAK DATY")}}

  Variable 4:
    Name: source_brand
    Value:
      {{if(contains(form_fields.source_url; "partytram"); "PARTYTRAM.FUN";
        if(contains(form_fields.source_url; "partyboat"); "PARTYBOAT.FUN";
        if(contains(form_fields.source_url; "busparty"); "BUSPARTY.FUN";
        "NIEZNANE")))}}

Output variables: city_normalized, service_type, task_date_display, source_brand
```

**Verify:** The exact field name for the source URL depends on how WPForms sends data. It may be in the webhook headers (Referer) or in a hidden form field. Test with a real submission to confirm the field path.

#### Module 3 — Tools: Set Multiple Variables (Completeness Score)

```
Type: Tools > Set Multiple Variables
Purpose: Calculate how many of the 8 key business fields are populated
Settings:
  Variable 1:
    Name: completeness_score
    Value:
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

Output variables: completeness_score
```

**Note:** Field names like `form_fields.imie_nazwisko` are placeholders. Replace with actual field paths from the WPForms webhook payload after testing Module 1.

#### Module 4 — HTTP: Make a Request (AI Classification + Summary)

```
Type: HTTP > Make a Request
Purpose: Call Anthropic API to classify the inquiry and generate a summary
Settings:
  URL: https://api.anthropic.com/v1/messages
  Method: POST
  Headers:
    - x-api-key: {{anthropic_api_key}}  (store in Make connection or scenario variable)
    - anthropic-version: 2023-06-01
    - content-type: application/json
  Body type: Raw (JSON)
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
  Parse response: Yes
  Output variables:
    - response_body (the full API response JSON)
    - Extract from response_body.content[0].text:
      - ai_klasyfikacja
      - ai_confidence
      - ai_podsumowanie
```

**Error handling:** Add an error handler route after this module:

```
Error handler: Resume
  On error: Set variables to fallback values:
    - ai_klasyfikacja = "do_weryfikacji"
    - ai_confidence = "niska"
    - ai_podsumowanie = ""
  Continue to next module (task creation proceeds without AI data)
```

#### Module 5 — JSON: Parse JSON (Extract AI Response)

```
Type: JSON > Parse JSON
Purpose: Parse the AI text response into usable variables
Settings:
  JSON string: {{response_body.content[].text}}
  Data structure: Create manually:
    - klasyfikacja (text)
    - confidence (text)
    - podsumowanie (text)
Output variables: klasyfikacja, confidence, podsumowanie
```

**Note:** Claude may occasionally wrap the response in markdown code fences. Add a Tools > Text Parser module before this one if needed, using regex to strip ``` markers: `{{replace(response_body.content[1].text; "/^```json?\n?|\n?```$/g"; "")}}`.

#### Module 6 — Tools: Set Multiple Variables (Map AI Output to ClickUp Values)

```
Type: Tools > Set Multiple Variables
Purpose: Map AI output strings to exact ClickUp dropdown values
Settings:
  Variable 1:
    Name: clickup_klasyfikacja
    Value:
      {{if(klasyfikacja = "goracy_lead"; "Gorący lead";
        if(klasyfikacja = "niekompletny_lead"; "Niekompletny lead";
        if(klasyfikacja = "zapytanie_ogolne"; "Zapytanie ogólne";
        if(klasyfikacja = "spam"; "Spam";
        "Do weryfikacji"))))}}

  Variable 2:
    Name: clickup_confidence
    Value:
      {{if(confidence = "wysoka"; "Wysoka";
        if(confidence = "srednia"; "Średnia";
        if(confidence = "niska"; "Niska";
        "Niska")))}}

  Variable 3:
    Name: task_title
    Value:
      {{task_date_display}} | {{if(city_normalized != "WYBIERZ MIASTO"; city_normalized; "WYBIERZ MIASTO")}} | {{service_type}} | {{form_fields.imie_nazwisko}}
```

#### Module 7 — ClickUp: Create a Task

```
Type: ClickUp > Create a Task
Settings:
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

**Verify in ClickUp:** Custom Fields in the Make ClickUp module are referenced by their field ID, not by name. After creating the 5 new fields (Section 3.2), retrieve their field IDs. In Make, when configuring the ClickUp module, the fields should appear in the "Custom Fields" section if the connection has proper permissions.

```
Error handling for entire scenario:
  Add a final error handler route that:
  1. Sends an email notification to the engineering/ops contact with the error details
  2. Creates a minimal ClickUp task with only: task title = "BŁĄD INTAKE | {{form_fields.imie_nazwisko}}", description = raw form payload as text
  This ensures no inquiry is lost even during system failures.

Test case:
  Action: Submit form on partyboat.fun with all fields filled (city: w Krakowie, event type: Wieczór kawalerski, package: PARTY, date: 2026-05-15, time: 20:00, people: 15, name: Jan Testowy, email: test@test.pl, phone: 500100200, description: "Chcemy wynająć statek na wieczór kawalerski")
  Expected: ClickUp task created with title "15-05-2026 | KRAKÓW | BOAT | Jan Testowy", MIASTO=KRAKÓW, USŁUGA=BOAT, KOMPLETNOŚĆ=100, KLASYFIKACJA_AI=Gorący lead, AI_PODSUMOWANIE filled
```

---

### Scenario 2: Email Intake

```
Scenario: EVSO - Email Intake
Purpose: Automatically creates ClickUp tasks from email inquiries sent to brand addresses, using AI to extract structured fields from unstructured email text
Trigger: Gmail > Watch Emails
```

**Preliminary setup:** The Gmail account(s) used for brand email must be connected to Make.com. If brand emails are handled through Google Workspace, connect the relevant account. If multiple inboxes need monitoring (kontakt@partyboat.fun, kontakt@partytram.fun, etc.), either create one scenario per inbox or use Gmail label-based filtering with a single watch.

#### Module 1 — Gmail: Watch Emails

```
Type: Gmail > Watch Emails
Settings:
  Connection: EVSO Gmail connection (the account receiving brand emails)
  Label: INBOX (or a specific label if pre-filtered)
  Filter:
    - From: (leave empty to catch all incoming)
    - Subject: (leave empty)
    - Search query: (optional — use "is:unread" to process only new emails)
  Mark as read: Yes (after processing)
  Maximum number of results: 10 (per execution cycle)
  Scheduling: Every 5 minutes (adjust based on volume and Make plan limits)
Output variables:
  - email_id
  - from_email (sender address)
  - from_name (sender display name)
  - subject
  - text_body (plain text content)
  - html_body (HTML content — use as fallback)
  - date (received timestamp)
  - attachments (array)
```

**Important:** If EVSO uses multiple brand email addresses on different Gmail accounts, create a separate Gmail Watch module for each OR set up email forwarding rules so all brand emails forward to a single intake inbox. The single-inbox approach is simpler to maintain.

#### Module 2 — Tools: Set Variable (Determine Brand from Email Address)

```
Type: Tools > Set Multiple Variables
Purpose: Identify which brand/service the email was sent to
Settings:
  Variable 1:
    Name: email_brand
    Value:
      {{if(contains(1.to_email; "partytram"); "PARTYTRAM.FUN";
        if(contains(1.to_email; "partyboat"); "PARTYBOAT.FUN";
        if(contains(1.to_email; "busparty"); "BUSPARTY.FUN";
        if(contains(1.to_email; "evso"); "EVSO.PL";
        "NIEZNANE"))))}}

  Variable 2:
    Name: email_service_hint
    Value:
      {{if(contains(1.to_email; "partytram"); "TRAM";
        if(contains(1.to_email; "partyboat"); "BOAT";
        if(contains(1.to_email; "busparty"); "BUS";
        "")))}}
```

**Note:** The `1.to_email` field may not be directly available from Gmail Watch. Verify the exact output structure. You may need to use `1.headers` and look for the `To:` header, or use the Gmail connection address as a proxy.

#### Module 3 — HTTP: Make a Request (AI Extraction + Classification)

```
Type: HTTP > Make a Request
Purpose: Call Anthropic API to extract structured fields from email text, classify, and summarize
Settings:
  URL: https://api.anthropic.com/v1/messages
  Method: POST
  Headers:
    - x-api-key: {{anthropic_api_key}}
    - anthropic-version: 2023-06-01
    - content-type: application/json
  Body type: Raw (JSON)
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
Error handling: Resume
  On error: Set fallback variables:
    - extracted = null (all extraction fields)
    - ai_klasyfikacja = "do_weryfikacji"
    - ai_confidence = "niska"
    - ai_podsumowanie = ""
  Continue to task creation with email metadata only
```

#### Module 4 — JSON: Parse JSON

```
Type: JSON > Parse JSON
Purpose: Parse the AI extraction response
Settings:
  JSON string: {{cleaned AI response text}}
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

#### Module 5 — Tools: Set Multiple Variables (Build Task Data)

```
Type: Tools > Set Multiple Variables
Purpose: Prepare all values for ClickUp task creation
Settings:
  Variable 1:
    Name: final_name
    Value: Use extracted name, fall back to email sender name:
      {{if(parsed.imie_nazwisko != null; parsed.imie_nazwisko; if(from_name != ""; from_name; from_email))}}

  Variable 2:
    Name: final_city
    Value:
      {{if(parsed.miasto != null; parsed.miasto; "WYBIERZ MIASTO")}}

  Variable 3:
    Name: final_service
    Value:
      {{if(parsed.usluga != null; parsed.usluga; email_service_hint)}}

  Variable 4:
    Name: final_date_display
    Value:
      {{if(parsed.data_eventu != null; parsed.data_eventu; "BRAK DATY")}}

  Variable 5:
    Name: task_title
    Value:
      {{final_date_display}} | {{final_city}} | {{final_service}} | {{final_name}}

  Variable 6:
    Name: completeness_score
    Value:
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

  Variable 7:
    Name: clickup_klasyfikacja
    Value:
      {{if(parsed.klasyfikacja = "goracy_lead"; "Gorący lead";
        if(parsed.klasyfikacja = "niekompletny_lead"; "Niekompletny lead";
        if(parsed.klasyfikacja = "zapytanie_ogolne"; "Zapytanie ogólne";
        if(parsed.klasyfikacja = "spam"; "Spam";
        "Do weryfikacji"))))}}

  Variable 8:
    Name: clickup_confidence
    Value:
      {{if(parsed.confidence = "wysoka"; "Wysoka";
        if(parsed.confidence = "srednia"; "Średnia";
        "Niska"))}}
```

#### Module 6 — ClickUp: Create a Task

```
Type: ClickUp > Create a Task
Settings:
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
Error handling for entire scenario:
  On any unrecoverable error:
  1. Send email notification with error details
  2. Apply Gmail label "INTAKE_ERROR" to the original email so it is not lost
  3. The email stays in the inbox and can be processed manually

Test case:
  Action: Send an email to kontakt@partyboat.fun with subject "Wieczór kawalerski Kraków" and body "Cześć, chciałbym zorganizować wieczór kawalerski na statku w Krakowie, 15 maja 2026, ok 20 osób. Dajcie znać ile kosztuje pakiet PARTY. Pozdrawiam, Marek Kowalski, tel 600100200"
  Expected: ClickUp task created with title "15-05-2026 | KRAKÓW | BOAT | Marek Kowalski", MIASTO=KRAKÓW, USŁUGA=BOAT, TYP IMPREZY=Wieczór kawalerski, ILOŚĆ OSÓB=20, PAKIET=PARTY, KOMPLETNOŚĆ=100, KLASYFIKACJA_AI=Gorący lead
```

---

## Section 5 — AI Integration Specification

### Model selection

**Model:** `claude-haiku-4-5-20251001` via Anthropic API

**Justification:**

| Factor | Assessment |
|--------|-----------|
| Cost | ~$0.25 per 1M input tokens, ~$1.25 per 1M output tokens. At 100 inquiries/week with ~500 tokens input and ~200 tokens output each, estimated monthly cost: ~$2-3. Negligible. |
| Speed | Median latency ~0.5-1s per request. Acceptable for async intake processing. |
| Polish language | Strong Polish comprehension and generation. Handles locative/nominative city forms, colloquial language, and domain-specific vocabulary. |
| Structured extraction | Reliable JSON output for simple schemas. Low hallucination rate on extraction tasks. |
| Task fit | Classification and extraction from short texts is well within Haiku's capability ceiling. Using a larger model (Sonnet, Opus) would add cost without measurably improving accuracy for this task complexity. |

**Alternative:** `gpt-4o-mini` via OpenAI API. Similar price and speed profile. Choose based on which API the team already has credentials for. The prompts in this guide use Claude-style formatting but are model-agnostic in content.

### API call method in Make.com

All AI calls use the `HTTP > Make a Request` module with the Anthropic Messages API.

**Connection setup:**

1. In Make.com, go to the scenario and add a new connection of type "HTTP"
2. No pre-configured authentication is needed — the API key is passed in the request header
3. Store the Anthropic API key as a Make.com scenario variable or in a Data Store for security. Do not hard-code it in the module.

**API endpoint:** `https://api.anthropic.com/v1/messages`

**Required headers for every request:**

```
x-api-key: {{anthropic_api_key}}
anthropic-version: 2023-06-01
content-type: application/json
```

### AI Task 1: Inquiry Classification (Form Channel)

**Task name:** Form Inquiry Classification + Summary

**When used:** Scenario 1 (Form Intake), Module 4

**System prompt (copy-paste ready):**

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

**User message template:**

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

**Expected output format:**

```json
{
  "klasyfikacja": "goracy_lead",
  "confidence": "wysoka",
  "podsumowanie": "Klient szuka wieczoru kawalerskiego na statku w Krakowie na 15 maja dla 20 osób, zainteresowany pakietem PARTY."
}
```

**Downstream parsing:** JSON is parsed in Module 5 (JSON > Parse JSON). The `klasyfikacja` value is mapped to ClickUp dropdown values in Module 6.

### AI Task 2: Email Field Extraction + Classification

**Task name:** Email Inquiry Extraction, Classification + Summary

**When used:** Scenario 2 (Email Intake), Module 3

**System prompt (copy-paste ready):**

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

**User message template:**

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

**Expected output format:**

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

### Explicit AI boundaries

The AI is **NOT** permitted to:

1. **Determine pricing** — pricing depends on package, city, date, group size, and availability. AI does not have this context and must not suggest prices.
2. **Route leads to sellers** — seller assignment is a human decision based on workload, city territory, and team dynamics.
3. **Send any message to a customer** — all outbound communication is human-initiated from within ClickUp tasks.
4. **Make qualification decisions** — AI classifies for prioritization, but the human decides whether to pursue a lead.
5. **Override human corrections** — if a sales team member changes KLASYFIKACJA_AI or any other AI-populated field, the AI does not re-set it.
6. **Invent data** — the extraction prompts explicitly instruct the model to return `null` for fields not present in the input. If AI returns a value, it must be traceable to text in the original message.

### Failure fallback

For both scenarios, if the Anthropic API call fails (timeout, 5xx error, rate limit, malformed response):

1. The Make.com error handler catches the failure.
2. AI-dependent fields are set to fallback values: KLASYFIKACJA_AI = "Do weryfikacji", AI_CONFIDENCE = "Niska", AI_PODSUMOWANIE = "" (empty).
3. All non-AI fields (from form data or email metadata) are still populated normally.
4. The ClickUp task is created with the fallback values.
5. The ClickUp automation (Section 3.5) sets Priority to Urgent, flagging the task for human attention.
6. An email notification is sent to the ops contact with the error details.

**Result:** The system never blocks on AI failure. Every inquiry gets a task. AI-failed tasks are visually marked and handled manually — the same way they would be handled without any automation at all.

---

## Section 6 — Implementation Sequence

### Week 1: ClickUp Setup + Form Scenario

```
Day 1: ClickUp field setup
  Tasks:
    1. Open NOWE ZAPYTANIA list in ClickUp
    2. Document all existing Custom Field IDs (needed for Make modules)
    3. Verify existing field types and dropdown options match expectations (Section 3.3)
    4. Add USŁUGA option "BUS" if missing
    5. Create 5 new Custom Fields (Section 3.2): ŹRÓDŁO_KANAŁ, KLASYFIKACJA_AI, AI_PODSUMOWANIE, KOMPLETNOŚĆ, AI_CONFIDENCE
    6. Create ClickUp automation "Flag AI Do Weryfikacji" (Section 3.5)
    7. Create two new views: NOWE - PRIORYTET, NOWE - KOMPLETNOŚĆ (Section 3.6)
  Prerequisite: None
  Deliverable: All ClickUp configuration complete

Day 2: Anthropic API setup + AI prompt testing
  Tasks:
    1. Create Anthropic API account if not existing, generate API key
    2. Test Form Classification prompt (AI Task 1) manually using the Anthropic Console or API playground
    3. Send 5 test inquiries through the prompt — cover: complete lead, partial lead, general question, spam, English-language inquiry
    4. Verify JSON output parses correctly for all 5 cases
    5. Adjust prompt wording if any classification is incorrect
    6. Store API key securely in Make.com (scenario variable or Data Store)
  Prerequisite: None (can run in parallel with Day 1)
  Deliverable: Validated AI prompts, API key stored in Make

Day 3: Build Form Intake scenario in Make.com
  Tasks:
    1. Create new Make scenario "EVSO - Form Intake Enhanced"
    2. Add Module 1: Webhook trigger (Section 4, Scenario 1, Module 1)
    3. Add Module 2: City normalization (Section 4, Scenario 1, Module 2)
    4. Add Module 3: Completeness score (Section 4, Scenario 1, Module 3)
    5. Add Module 4: AI classification HTTP call (Section 4, Scenario 1, Module 4)
    6. Add Module 5: JSON parser (Section 4, Scenario 1, Module 5)
    7. Add Module 6: Value mapping (Section 4, Scenario 1, Module 6)
    8. Add Module 7: ClickUp Create Task (Section 4, Scenario 1, Module 7)
    9. Add error handlers for AI module and entire scenario
    10. Configure WPForms to send to new webhook URL (do NOT disable old scenario yet)
  Prerequisite: Day 1 (ClickUp fields exist), Day 2 (API key stored)
  Deliverable: Form Intake scenario built, not yet live

Day 4: Test Form Intake scenario
  Tasks:
    1. Submit test form on each of the 3 domains (partytram, partyboat, busparty)
    2. Verify task creation in ClickUp — check all 16 Custom Fields
    3. Verify task naming convention matches expected pattern
    4. Verify city normalization for at least 3 different cities
    5. Verify service type derivation from each domain
    6. Test AI failure fallback: temporarily set wrong API key, submit form, verify task created with "Do weryfikacji"
    7. Fix any issues found
  Prerequisite: Day 3
  Deliverable: Form Intake scenario validated

Day 5: Go live with Form Intake
  Tasks:
    1. Disable old WPForms → Make scenario
    2. Enable new Form Intake Enhanced scenario
    3. Monitor first 5-10 real form submissions
    4. Verify tasks appear correctly in ClickUp
    5. Check with sales team that new fields are visible and useful
    6. Document any issues for later fix
  Prerequisite: Day 4 (tests pass)
  Deliverable: Form Intake live in production
```

### Week 2: Email Scenario

```
Day 6: Email channel setup + AI prompt testing
  Tasks:
    1. Identify all Gmail accounts/inboxes that receive brand emails
    2. Decide approach: multiple Gmail Watch modules or single forwarding inbox
    3. Connect relevant Gmail account(s) to Make.com
    4. Test Email Extraction prompt (AI Task 2) manually with 5 sample emails:
       - Complete inquiry in Polish
       - Partial inquiry (missing date and city)
       - General question about pricing
       - Spam/marketing email
       - English-language inquiry
    5. Verify JSON extraction accuracy for all 5 cases
    6. Adjust prompt if needed
  Prerequisite: Day 2 (API key set up)
  Deliverable: Gmail connected, email AI prompt validated

Day 7: Build Email Intake scenario in Make.com
  Tasks:
    1. Create new Make scenario "EVSO - Email Intake"
    2. Add Module 1: Gmail Watch (Section 4, Scenario 2, Module 1)
    3. Add Module 2: Brand detection (Section 4, Scenario 2, Module 2)
    4. Add Module 3: AI extraction HTTP call (Section 4, Scenario 2, Module 3)
    5. Add Module 4: JSON parser (Section 4, Scenario 2, Module 4)
    6. Add Module 5: Variable builder (Section 4, Scenario 2, Module 5)
    7. Add Module 6: ClickUp Create Task (Section 4, Scenario 2, Module 6)
    8. Add error handlers (Gmail label on failure, email notification)
  Prerequisite: Day 6 (Gmail connected, prompts tested), Day 1 (ClickUp fields exist)
  Deliverable: Email Intake scenario built

Day 8: Test Email Intake scenario
  Tasks:
    1. Send test emails to each brand address from an external email account
    2. Cover all 5 test cases from Day 6
    3. Verify ClickUp task creation — check all Custom Fields
    4. Verify task naming convention with AI-extracted data
    5. Verify email body is preserved in task description
    6. Test AI failure: temporarily break API key, send email, verify fallback task
    7. Test Gmail label application on error
    8. Fix any issues found
  Prerequisite: Day 7
  Deliverable: Email Intake scenario validated

Day 9: Email Intake go-live + monitoring
  Tasks:
    1. Enable Email Intake scenario with 15-minute polling interval (conservative start)
    2. Monitor first 10 real email inquiries processed
    3. Compare AI extraction quality against manual reading of the same emails
    4. Check for false positives: verify no internal/non-inquiry emails are creating tasks
    5. Add Gmail filter rules if needed to exclude known non-inquiry senders
    6. Adjust polling interval based on volume (can reduce to 5 minutes if stable)
  Prerequisite: Day 8 (tests pass)
  Deliverable: Email Intake live in production
```

### Week 3: Stabilization + Handover

```
Day 10: Run full Smoke Test Checklist (Section 7)
  Tasks:
    1. Execute all smoke test cases from Section 7
    2. Document results: pass/fail per test case
    3. Fix any failures
    4. Re-run failed tests after fixes
  Prerequisite: Day 5 + Day 9 (both scenarios live)
  Deliverable: All smoke tests passing

Day 11: Sales team walkthrough + feedback collection
  Tasks:
    1. Demonstrate new views (NOWE - PRIORYTET, NOWE - KOMPLETNOŚĆ) to sales team
    2. Explain new fields: KLASYFIKACJA_AI, KOMPLETNOŚĆ, AI_PODSUMOWANIE, AI_CONFIDENCE
    3. Show them how to filter/sort by these fields
    4. Explain "Do weryfikacji" and what it means
    5. Collect feedback: what is useful, what is confusing, what is missing
    6. Create a simple 1-page internal reference in WIKI PRACOWNIKA explaining the new fields
  Prerequisite: Day 10
  Deliverable: Team trained, feedback documented

Day 12: Refinements + documentation
  Tasks:
    1. Implement highest-priority feedback items from Day 11
    2. Review AI classification accuracy on 20+ real leads processed in the past week
    3. Adjust AI prompts if classification accuracy is below 80%
    4. Document both Make scenarios: screenshot each module configuration, save in WIKI PRACOWNIKA or shared folder
    5. Document manual fallback procedure: "If Make is down, how to create a task manually with all required fields"
    6. Write a brief Phase 2 planning note (based on Deferred section below)
  Prerequisite: Day 11
  Deliverable: System documented, refinements applied
```

**Estimated total engineering time: 12 working days (2.5 weeks)**

---

## Section 7 — Smoke Test Checklist

```
Test 1: Complete Form Inquiry
  Action: Submit form on partyboat.fun with all fields filled:
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
  Expected result:
    - ClickUp task created in NOWE ZAPYTANIA
    - Title: "15-05-2026 | KRAKÓW | BOAT | Jan Testowy"
    - MIASTO: KRAKÓW
    - USŁUGA: BOAT
    - TYP IMPREZY: Wieczór kawalerski
    - PAKIET: PARTY
    - POZYSKANIE: PARTYBOAT.FUN
    - ŹRÓDŁO_KANAŁ: Formularz
    - KOMPLETNOŚĆ: 100
    - KLASYFIKACJA_AI: Gorący lead
    - AI_CONFIDENCE: Wysoka
    - AI_PODSUMOWANIE: non-empty Polish text
    - EMAIL: jan.testowy@test.pl
    - TELEFON: 500100200
    - Description contains original form data
  Pass condition: All fields match expected values. Task visible in NOWE - PRIORYTET view.
```

```
Test 2: Partial Form Inquiry (missing date and city)
  Action: Submit form on partytram.fun with:
    - Miasto: (leave default / unselected)
    - Rodzaj imprezy: Urodziny
    - Pakiet: (leave default)
    - Imię i Nazwisko: Anna Częściowa
    - Email: anna@test.pl
    - Telefon: 600200300
    - Data: (leave empty if possible, or enter placeholder)
    - Godzina: (leave empty)
    - Liczba osób: 8
    - Opis: Chciałabym urodziny na tramwaju
  Expected result:
    - Task created with title containing "BRAK DATY | WYBIERZ MIASTO | TRAM | Anna Częściowa"
    - KOMPLETNOŚĆ: between 50-75 (some fields missing)
    - KLASYFIKACJA_AI: Niekompletny lead
    - USŁUGA: TRAM (from domain)
    - ŹRÓDŁO_KANAŁ: Formularz
  Pass condition: Task created, classification reflects incompleteness, completeness score is less than 100.
```

```
Test 3: Email — Complete Inquiry
  Action: Send email from external account to kontakt@partyboat.fun with:
    Subject: "Wieczór panieński Wrocław"
    Body: "Dzień dobry, chciałabym zorganizować wieczór panieński na statku we Wrocławiu dnia 20 czerwca 2026. Będzie nas 12 osób. Interesuje nas pakiet LUX. Proszę o kontakt: 700800900. Pozdrawiam, Katarzyna Nowak"
  Expected result:
    - Task created in NOWE ZAPYTANIA
    - Title: "20-06-2026 | WROCŁAW | BOAT | Katarzyna Nowak"
    - MIASTO: WROCŁAW
    - USŁUGA: BOAT
    - TYP IMPREZY: Wieczór panieński
    - PAKIET: LUX
    - ILOŚĆ OSÓB: 12
    - TELEFON: 700800900
    - EMAIL: (sender's email address)
    - ŹRÓDŁO_KANAŁ: Email
    - KOMPLETNOŚĆ: 100
    - KLASYFIKACJA_AI: Gorący lead
    - Description contains full email body
  Pass condition: All extracted fields match email content. No fields are hallucinated.
```

```
Test 4: Email — Spam / Irrelevant Message
  Action: Send email from external account to kontakt@partytram.fun with:
    Subject: "Oferta współpracy marketingowej"
    Body: "Szanowni Państwo, nasza firma oferuje usługi pozycjonowania stron internetowych. Gwarantujemy top 3 w Google w 30 dni. Zapraszam do kontaktu."
  Expected result:
    - Task created in NOWE ZAPYTANIA
    - KLASYFIKACJA_AI: Spam
    - AI_CONFIDENCE: Wysoka
    - KOMPLETNOŚĆ: low (only email populated)
    - ŹRÓDŁO_KANAŁ: Email
    - Most event fields empty (MIASTO, DATA EVENTU, TYP IMPREZY, etc.)
  Pass condition: AI correctly classifies as spam. No event fields are fabricated from the marketing text.
```

```
Test 5: Email — Vague / General Inquiry
  Action: Send email from external account to kontakt@busparty.fun with:
    Subject: "Pytanie"
    Body: "Hej, ile kosztuje wynajęcie busa? Pozdrawiam Tomek"
  Expected result:
    - Task created in NOWE ZAPYTANIA
    - Title: "BRAK DATY | WYBIERZ MIASTO | BUS | Tomek"
    - KLASYFIKACJA_AI: Zapytanie ogólne
    - KOMPLETNOŚĆ: low (25 or less — only name and email populated)
    - USŁUGA: BUS (from brand)
    - ŹRÓDŁO_KANAŁ: Email
    - Most fields null/empty
  Pass condition: AI classifies as general inquiry, not as lead. USŁUGA correctly derived from brand domain.
```

```
Test 6: AI Failure Fallback (Form)
  Action: Temporarily set the Anthropic API key in Make to an invalid value. Submit a form on any domain with complete data.
  Expected result:
    - Task created in NOWE ZAPYTANIA (AI failure does not block task creation)
    - KLASYFIKACJA_AI: Do weryfikacji
    - AI_CONFIDENCE: Niska
    - AI_PODSUMOWANIE: empty
    - All non-AI fields (MIASTO, USŁUGA, TELEFON, EMAIL, etc.) populated normally from form data
    - KOMPLETNOŚĆ calculated normally
    - Priority: Urgent (set by ClickUp automation)
  Pass condition: Task exists with all form-derived fields. AI failure is gracefully handled. Priority flag is visible.
  Post-test: Restore the correct API key immediately.
```

```
Test 7: AI Failure Fallback (Email)
  Action: Temporarily set the Anthropic API key in Make to an invalid value. Send a test email to any brand address.
  Expected result:
    - Task created in NOWE ZAPYTANIA
    - KLASYFIKACJA_AI: Do weryfikacji
    - AI_CONFIDENCE: Niska
    - EMAIL: sender's email (from metadata, not AI)
    - Description: contains email body
    - Title: "BRAK DATY | WYBIERZ MIASTO | [SERVICE from brand] | [sender name or email]"
    - Priority: Urgent
  Pass condition: Task exists with email metadata. No data lost despite AI failure.
  Post-test: Restore the correct API key immediately.
```

---

## Deferred to Phase 2

The following features were considered during design and deliberately excluded to stay within the 12-day, single-engineer constraint. Each item was evaluated against the expert panel.

| Feature | Why deferred | Expert who flagged |
|---------|-------------|-------------------|
| **Draft reply generation** | Touches customer-facing communication. Requires prompt tuning, tone calibration, and sales team trust-building. Too risky for Phase 1. | Aleksander (AI): "AI drafts reaching customers need weeks of validation." Karolina (Sales): "Sales team needs to trust the classification first." |
| **Automated seller routing** | Requires agreement on territory rules, fallback for cross-city inquiries, and handling of new sellers. Process change, not just technical change. | Karolina (Sales): "Assignment is a team decision. Don't automate it before the team asks for it." |
| **Lead deduplication** | A customer might submit a form AND send an email. Deduplication needs fuzzy matching on email/phone. Adds complexity to both scenarios. | Tomasz (Make): "Dedup requires a Data Store, lookup logic, and merge strategy. That's a project in itself." |
| **Follow-up reminders** | Automated reminders for incomplete leads. Useful but requires defining SLA rules that don't exist yet. | Karolina (Sales): "Define the follow-up process manually first. Automate it once the rhythm is established." |
| **ZLECENIA workflow automation** | Post-sales operational flow. Completely separate domain from intake. | Marta (ClickUp): "ZLECENIA is a different process with different data needs. Don't mix it with intake." |
| **Analytics dashboard** | Metrics on lead volume, channel performance, classification accuracy. Valuable but not essential for Phase 1 operations. | All: "Get the data flowing first. Dashboard is easy to add once data is clean." |
| **Multi-language support** | English and other language inquiries exist but are a minority. The current prompts handle English inputs adequately for classification, but extraction quality may vary. | Aleksander (AI): "The prompts work for English already. Explicit multi-language support can wait for volume data." |

---

*End of implementation guide. Total estimated effort: 12 engineer-days across 3 weeks.*
