# EVSO — Lead Intake Automation: Phase 2 Implementation Guide

**Version:** 1.0
**Date:** 2026-03-31
**Scope:** Post-intake automation — follow-up reminders, lead deduplication, draft reply generation, ZLECENIA workflow automation
**Target reader:** Single mid-level engineer with Make.com and ClickUp experience
**Estimated effort:** 12 engineer-days
**Prerequisite:** Phase 1 implementation complete and stable (both Make scenarios live, all 5 new Custom Fields populated, ClickUp Automation active)

**Expert panel validation:** All design decisions in this document were validated in Expert Meeting M-002 (Structured Workshop) by four specialists — ClickUp architecture (Marta Kurek), Make automation (Tomasz Sowa), AI reliability (Aleksander Bryła), and sales process design (Karolina Wysocka). Seven decision records, seven uncertainty items, and three resolved disagreements are documented in `EVSO_Phase2_Expert_Meeting_Output.md`.

---

## Section 1 — Current State Analysis (Post-Phase 1)

### What Phase 1 delivered

Phase 1 created a functional lead intake pipeline with two automated channels:

**Channel A — Form Intake (Scenario: "EVSO - Form Intake Enhanced"):** WPForms submissions from `partytram.fun`, `partyboat.fun`, and `busparty.fun` are processed through a Make.com scenario that normalizes city values, derives service type from the domain, calculates a completeness score, calls Claude Haiku for classification and summary, and creates a fully-populated ClickUp task in NOWE ZAPYTANIA.

**Channel B — Email Intake (Scenario: "EVSO - Email Intake"):** Emails arriving at brand addresses are processed through a Make.com scenario that detects the brand, calls Claude Haiku to extract structured fields from unstructured email text, calculates completeness, and creates a ClickUp task with all available data.

**ClickUp state after Phase 1:** NOWE ZAPYTANIA has 16 Custom Fields (11 original + 5 new: ŹRÓDŁO_KANAŁ, KLASYFIKACJA_AI, AI_PODSUMOWANIE, KOMPLETNOŚĆ, AI_CONFIDENCE). One ClickUp Automation flags tasks with KLASYFIKACJA_AI = "Do weryfikacji" as Urgent. Two filtered views (NOWE - PRIORYTET, NOWE - KOMPLETNOŚĆ) help the sales team prioritize.

**AI state after Phase 1:** Anthropic API key stored in Make.com. Two AI tasks operational — Form Classification + Summary (Scenario 1, Module 4) and Email Extraction + Classification (Scenario 2, Module 3). Both use `claude-haiku-4-5-20251001` with error handlers that degrade gracefully to "Do weryfikacji."

### Where friction remains after Phase 1

1. **No follow-up mechanism for unhandled leads.** A task can sit in NOWE ZAPYTANIA without an assignee indefinitely. There is no timeout, reminder, or escalation. At ~100 inquiries/week, some leads inevitably slip through the cracks.

2. **Duplicate leads across channels.** A customer may submit a form AND send an email about the same event. Phase 1 creates two separate tasks. The sales team discovers duplicates only by manually noticing the same name or email — wasting time and risking contradictory responses to the same customer.

3. **Repetitive reply drafting.** Every lead requires a human-written initial response — either a follow-up asking for missing details (for incomplete leads) or an acknowledgment confirming receipt (for hot leads). These replies follow predictable patterns. Writing them from scratch each time costs 3-5 minutes per lead, or 5-8 hours per week across the team.

4. **Manual CRM-to-ZLECENIA handoff.** When a lead converts to a confirmed event, someone must manually create a new task in ZLECENIA, copy all relevant data from the CRM task, extend the naming convention to include the package, and set the initial ZLECENIA status. At 20-30 won events per month, this is 2-5 hours of error-prone copy-paste work.

### What works well and must be preserved

1. **Phase 1 intake pipeline** — stable, tested, with graceful AI fallback. Phase 2 changes to existing scenarios must be additive (router branches, new modules appended), never destructive.
2. **Task naming convention** — `[DATE|BRAK DATY] | [CITY|WYBIERZ MIASTO] | [SERVICE] | [NAME]` — preserved exactly for CRM tasks.
3. **LEAD-#### custom IDs** — stable, no changes.
4. **NOWE ZAPYTANIA as single intake reservoir** — all leads enter here before downstream routing.
5. **Seller-specific pipeline separation** — SPRZEDAŻ KUBA/WIKTOR/DAWID/KRIS.
6. **AI assists, humans decide** — no automated customer communication, no automated routing.
7. **Email composer in task** — the established workflow for outbound communication.
8. **CRM ↔ ZLECENIA separation** — clean boundary between sales and operations domains.

### Data captured vs. missing (post-Phase 1)

**Currently captured (16 Custom Fields):** MIASTO, PAKIET, POZYSKANIE, TELEFON, TYP IMPREZY, USŁUGA, GODZINA EVENTU, DATA EVENTU, EMAIL, ILOŚĆ OSÓB, JEDNOSTKA, ŹRÓDŁO_KANAŁ, KLASYFIKACJA_AI, AI_PODSUMOWANIE, KOMPLETNOŚĆ, AI_CONFIDENCE.

**Missing and needed for Phase 2:**

| Gap | Why it matters |
|-----|---------------|
| Contact frequency tracking | No way to know if a customer has reached out multiple times about the same event |
| "Sale won" status in seller pipelines | No consistent trigger point for CRM→ZLECENIA automation |
| Due Date on new tasks | No mechanism for time-based follow-up reminders |

---

## Section 2 — Proposed Solution Architecture

### End-to-end flow (numbered sequence)

Phase 2 adds four capabilities to the existing Phase 1 pipeline. The numbered sequences below show how each integrates.

**Capability 1 — Follow-up Reminders:**

1. Phase 1 Make scenarios (both Form Intake and Email Intake) are updated: the ClickUp Create Task module now sets Due Date = `now + 48 hours` for every non-spam task
2. A new ClickUp native Automation monitors Due Date arrival: if the task has no Assignee when Due Date fires, the Automation sets Priority to Urgent and posts an escalation comment
3. The sales team sees the flagged task in the NOWE - PRIORYTET view and acts on it

**Capability 2 — Lead Deduplication:**

1. A new Make Data Store (`EVSO_Lead_Dedup`) stores `{email, task_id, created_at}` for every new lead
2. Phase 1 Make scenarios are modified: BEFORE the AI classification call, a Data Store lookup checks if an email match exists within the last 7 days
3. If NO match → proceed normally (AI call → task creation → store record in Data Store)
4. If MATCH found → skip AI call, skip task creation. Instead: add a comment to the existing task with the new inquiry, update Custom Fields with any previously-missing data, recalculate KOMPLETNOŚĆ, increment LICZBA_KONTAKTÓW, add tag WIELOKROTNY KONTAKT
5. If the merged task has no Assignee and LICZBA_KONTAKTÓW > 1, set Priority to Urgent
6. A weekly cleanup scenario deletes Data Store records older than 7 days

**Capability 3 — Draft Reply Generation:**

1. After task creation in both Make scenarios, a new router branch checks KLASYFIKACJA_AI and AI_CONFIDENCE
2. For "Niekompletny lead" with confidence ≠ "Niska" → call Claude Haiku with the incomplete-lead prompt → post draft as ClickUp comment
3. For "Gorący lead" with confidence ≠ "Niska" → call Claude Haiku with the hot-lead prompt → post draft as ClickUp comment (DISABLED at launch, enabled via toggle after team validation)
4. Draft is clearly marked as AI-generated. The handlowiec reads it, copies useful parts into the email composer, edits, and sends
5. If AI call fails → no comment posted, no impact on task (graceful degradation)

**Capability 4 — ZLECENIA Workflow Automation:**

1. A new status `SPRZEDANE` is added to every seller pipeline list (SPRZEDAŻ KUBA/WIKTOR/DAWID/KRIS)
2. When a handlowiec moves a task to SPRZEDANE status (human decision), a ClickUp webhook triggers a new Make scenario ("EVSO - ZLECENIA Task Creator")
3. The scenario fetches full CRM task details, creates a new task in the ZLECENIA list with mapped fields and extended naming convention (`[DATE] | [CITY] | [SERVICE] | [CLIENT] | [PACKAGE]`), sets ZLECENIA status to "ZLECENIA BEZ ZALICZEK"
4. The scenario posts cross-link comments on both tasks (CRM → ZLECENIA, ZLECENIA → CRM)
5. The CRM task status is not changed beyond SPRZEDANE — it stays in the seller pipeline for commission tracking and sales visibility

### What changes relative to Phase 1

- **Phase 1 Scenario 1 (Form Intake Enhanced):** Module order changed — dedup check inserted before AI call. Due Date field added to task creation. Router branch added after task creation for draft reply generation. Data Store write added after task creation.
- **Phase 1 Scenario 2 (Email Intake):** Same modifications as Scenario 1.
- **ClickUp NOWE ZAPYTANIA:** One new Custom Field (LICZBA_KONTAKTÓW). One new Automation (Due Date reminder).
- **ClickUp seller pipelines:** New status SPRZEDANE added to each list.
- **Make.com:** One new Data Store (EVSO_Lead_Dedup). Two new scenarios (ZLECENIA Task Creator, Dedup Cleanup).

### What deliberately stays the same

- NOWE ZAPYTANIA remains the sole intake list
- Task naming convention for CRM tasks is preserved exactly
- LEAD-#### ID scheme is untouched
- All Phase 1 AI prompts (classification, extraction) are unchanged
- Phase 1 error handlers and fallback behavior are preserved
- Manual seller assignment remains (no automatic routing)
- All customer-facing communication remains human-initiated

### Explicit scope boundary

**This guide covers:**
- ClickUp field and status additions
- Modifications to two existing Make scenarios (Form Intake Enhanced, Email Intake)
- Two new Make scenarios (ZLECENIA Task Creator, Dedup Cleanup)
- One new Make Data Store
- Two new ClickUp Automations
- AI integration for draft reply generation (two prompts)
- Testing of all four capabilities

**NOT covered (excluded per user direction):**
- Multi-language support
- Analytics dashboards
- Automatic lead routing by city or service type

---

## Section 3 — ClickUp Setup Guide

### 3.1 Lists / Folders — no new lists created

All changes apply to existing lists. No new lists or folders are created.

### 3.2 Custom Fields to add

Add the following Custom Field to `EVSO / CRM / NOWE ZAPYTANIA`:

**Field 1: LICZBA_KONTAKTÓW**

| Property | Value |
|----------|-------|
| Name | `LICZBA_KONTAKTÓW` |
| Type | Number (integer, no decimals) |
| Default value | 1 |
| Location | NOWE ZAPYTANIA list |
| Purpose | Tracks how many times a customer has reached out about this inquiry. Incremented on each dedup merge. A value > 1 signals repeated contact — a positive buying signal that deserves fast attention. |

**Implementation note:** After creating the field, retrieve its ClickUp field ID for use in Make modules. On existing tasks, the field will show no value (null) until set. New tasks created by the modified Make scenarios will set this to `1` at creation. On dedup merge, the existing task's value is incremented.

### 3.3 Statuses to add

Add the status `SPRZEDANE` to each of the four seller pipeline lists:

| List | Status to add | Position | Color suggestion |
|------|---------------|----------|-----------------|
| SPRZEDAŻ KUBA | `SPRZEDANE` | After the last active sales status (after SZANSA SPRZEDAŻY or equivalent) | Green |
| SPRZEDAŻ WIKTOR | `SPRZEDANE` | Same position | Green |
| SPRZEDAŻ DAWID | `SPRZEDANE` | Same position | Green |
| SPRZEDAŻ KRIS | `SPRZEDANE` | Same position | Green |

**Verification required (U-002):** Before adding, check each seller pipeline list to confirm they share the same status structure. The observed statuses on SPRZEDAŻ DAWID are: NOWE ZAPYTANIE → WYSŁANO OFERTĘ → PRZEDZWONKA → ZASTANAWIA SIĘ → SZANSA SPRZEDAŻY. Verify whether the other three lists match. If a list uses different names or ordering, adapt accordingly — the key requirement is that `SPRZEDANE` exists as a status that the handlowiec can drag a task into.

**Purpose:** SPRZEDANE is the trigger point for the ZLECENIA automation. It is a human action — the handlowiec decides "this deal is won" and moves the task to SPRZEDANE. The automation then creates the ZLECENIA task.

### 3.4 ClickUp Automations — add two

**Automation 1: Follow-up reminder for unassigned tasks**

| Property | Value |
|----------|-------|
| Name | `Przypomnienie 48h — brak przypisania` |
| Location | NOWE ZAPYTANIA list |
| Trigger | When Due Date arrives |
| Condition | Assignee is empty |
| Action 1 | Set Priority to `Urgent` |
| Action 2 | Post comment: `⚠️ To zapytanie czeka na obsługę od 48h. Proszę o przypisanie lub oznaczenie jako nieistotne.` |

**Why this approach:** Due Date is set by Make at task creation (see Section 4). When it arrives and the task still has no Assignee, this Automation escalates visibility. Using Due Date rather than "status duration" triggers is intentional — Due Date is visible in Calendar and List views, giving the team a deadline without opening the task, and it fires regardless of status changes.

**Verification required (U-004):** ClickUp's "When Due Date arrives" trigger is available on Business plans and above. Verify EVSO's ClickUp plan before configuring. If the plan doesn't support this trigger, the fallback is a Make scheduled scenario that runs hourly and searches for overdue unassigned tasks (more complex, less reliable, but plan-independent).

**Automation 2: Escalate multi-contact leads (triggered by dedup merge)**

| Property | Value |
|----------|-------|
| Name | `Eskalacja — wielokrotny kontakt` |
| Location | NOWE ZAPYTANIA list |
| Trigger | When Custom Field `LICZBA_KONTAKTÓW` changes |
| Condition | LICZBA_KONTAKTÓW > 1 AND Assignee is empty |
| Action | Set Priority to `Urgent` |

**Purpose:** When a customer contacts EVSO multiple times (dedup merge increments the counter), and the task still has no Assignee, this is a strong buying signal being ignored. The Automation flags it for immediate attention.

**Note:** If ClickUp's Automation conditions don't support "field > 1" natively, use "field is not empty AND field is not 1" or handle this logic in the Make scenario instead (set Priority to Urgent in the dedup-merge branch when Assignee is empty).

### 3.5 Views — no new views needed

The existing NOWE - PRIORYTET and NOWE - KOMPLETNOŚĆ views from Phase 1 already surface the information needed for Phase 2 features. The Priority field (used by follow-up reminders and dedup escalation) is already visible in these views.

**Optional enhancement:** Add `LICZBA_KONTAKTÓW` as a visible column in both views so the team can see which leads have multiple contacts at a glance.

---

## Section 4 — Make.com Scenario Guide

Phase 2 modifies the two existing Phase 1 scenarios and adds two new scenarios. The modifications are additive — new modules are inserted or appended, existing modules are not removed.

**Critical prerequisite:** Before modifying any Phase 1 scenario, create a backup by cloning the scenario in Make.com. Name the clones "EVSO - Form Intake Enhanced (Phase 1 Backup)" and "EVSO - Email Intake (Phase 1 Backup)." Disable the clones. This ensures rollback capability if Phase 2 modifications introduce regressions.

### Data Store Setup (before modifying scenarios)

Create a new Data Store in Make.com:

```
Data Store: EVSO_Lead_Dedup
Purpose: Lookup table for lead deduplication — stores email↔task mapping for recent leads
Structure:
  Key field: email (text, primary key)
  Additional fields:
    - task_id (text) — ClickUp task ID of the original task
    - task_name (text) — ClickUp task name (for reference in merge comments)
    - created_at (date) — timestamp when the record was created
    - phone (text, optional) — stored for potential future phone-based matching
Max records: 500 (at ~100 leads/week and 7-day retention, max ~100 active records — 500 provides ample buffer)
```

**Setup steps:**
1. In Make.com, navigate to Data Stores
2. Click "Add data store"
3. Name: `EVSO_Lead_Dedup`
4. Add data structure with the fields above
5. Set `email` as the key
6. Save

Also create a scenario variable for the dedup window:

```
Variable: dedup_window_days
Value: 7
Purpose: Configurable dedup matching window. Start at 7, adjust based on real data after 30 days of operation.
```

And a scenario variable for the draft reply toggle:

```
Variable: hot_lead_draft_enabled
Value: false
Purpose: Controls whether hot-lead draft replies are generated. Start disabled. Enable after 1-2 weeks of team feedback on incomplete-lead drafts.
```

---

### Scenario 1: Form Intake Enhanced — Phase 2 Modifications

The existing Phase 1 scenario has 7 modules in this order:

```
Phase 1 order:
  Module 1: Webhook (trigger)
  Module 2: Set Variables — City normalization
  Module 3: Set Variables — Completeness score
  Module 4: HTTP — AI Classification
  Module 5: JSON Parse — AI response
  Module 6: Set Variables — Map AI to ClickUp values
  Module 7: ClickUp — Create Task
```

Phase 2 inserts new modules and reorders. The new flow:

```
Phase 2 order:
  Module 1:  Webhook (trigger) — UNCHANGED
  Module 2:  Set Variables — City normalization — UNCHANGED
  Module 3:  Set Variables — Completeness score — UNCHANGED
  Module 4:  Data Store — Search Records (dedup check) — NEW
  Module 5:  Router — NEW
    Branch A (duplicate detected):
      Module 5A-1: ClickUp — Get Task (fetch existing task)
      Module 5A-2: Set Variables — Build merge data
      Module 5A-3: ClickUp — Update Task (update fields, increment LICZBA_KONTAKTÓW)
      Module 5A-4: ClickUp — Create Comment (append new inquiry)
      Module 5A-5: Data Store — Update Record (refresh timestamp)
      → Scenario ends (no AI call, no new task, no draft)
    Branch B (no duplicate — new lead):
      Module 5B-1: HTTP — AI Classification — (MOVED from Phase 1 Module 4)
      Module 5B-2: JSON Parse — AI response — (MOVED from Phase 1 Module 5)
      Module 5B-3: Set Variables — Map AI to ClickUp values — (MOVED from Phase 1 Module 6)
      Module 5B-4: ClickUp — Create Task (with Due Date) — (MODIFIED from Phase 1 Module 7)
      Module 5B-5: Data Store — Add Record — NEW
      Module 5B-6: Set Variables — Prepare draft context — NEW
      Module 5B-7: Router — Draft Reply — NEW
        Branch B1 (Niekompletny lead):
          Module 5B-7a: HTTP — AI Draft Reply (incomplete lead prompt)
          Module 5B-7b: JSON Parse — Draft response
          Module 5B-7c: ClickUp — Create Comment (draft reply)
        Branch B2 (Gorący lead — DISABLED at launch):
          Module 5B-7d: HTTP — AI Draft Reply (hot lead prompt)
          Module 5B-7e: JSON Parse — Draft response
          Module 5B-7f: ClickUp — Create Comment (draft reply)
        Branch B3 (all others): No action
```

#### Module 4 — Data Store: Search Records (Dedup Check)

```
Type: Data Store > Search Records
Purpose: Check if a lead with this email already exists within the dedup window
Settings:
  Data store: EVSO_Lead_Dedup
  Filter:
    Condition 1: email = {{form_fields.email}}
    Condition 2: created_at > {{addDays(now; -dedup_window_days)}}
  Limit: 1
Output variables:
  - dedup_match (boolean — true if any records returned)
  - existing_task_id (text — task_id from matched record, or empty)
  - existing_task_name (text — task_name from matched record, or empty)
```

**Error handler:**

```
Error handler: Resume
  On error (Data Store lookup failure):
    Set dedup_match = false
    Continue to Branch B (treat as new lead)
  Rationale: Data Store downtime should degrade to "no dedup" rather than "no intake."
  This ensures intake is never blocked by the dedup system.
```

#### Module 5 — Router

```
Type: Router
Purpose: Split flow based on dedup result
Settings:
  Branch A:
    Label: "Duplikat — merge"
    Condition: dedup_match = true
  Branch B:
    Label: "Nowy lead"
    Condition: dedup_match = false (fallback / default)
```

#### Branch A — Duplicate Detected (Merge Flow)

**Module 5A-1 — ClickUp: Get a Task**

```
Type: ClickUp > Get a Task
Purpose: Fetch the existing task's current data for merge
Settings:
  Task ID: {{existing_task_id}}
Output variables:
  - existing_task (full task object including Custom Fields, Assignees)
```

**Module 5A-2 — Tools: Set Multiple Variables (Build Merge Data)**

```
Type: Tools > Set Multiple Variables
Purpose: Prepare data for the merge update
Settings:
  Variable 1:
    Name: new_contact_count
    Value: {{if(existing_task.custom_fields.LICZBA_KONTAKTÓW != null; existing_task.custom_fields.LICZBA_KONTAKTÓW + 1; 2)}}

  Variable 2:
    Name: merge_comment_body
    Value: |
      --- DODATKOWY KONTAKT (Formularz) --- {{formatDate(now; "DD-MM-YYYY HH:mm")}} ---
      Miasto: {{form_fields.miasto}}
      Rodzaj imprezy: {{form_fields.rodzaj_imprezy}}
      Pakiet: {{form_fields.pakiet}}
      Data eventu: {{form_fields.data}}
      Godzina: {{form_fields.godzina}}
      Liczba osób: {{form_fields.liczba_osob}}
      Opis: {{form_fields.opis}}
      ---

  Variable 3:
    Name: updated_completeness
    Value: Recalculate based on merged data — use the same formula as Module 3
      but apply the higher value between existing and new for each field:
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

**Note on Custom Field access:** ClickUp API returns Custom Fields as an array of objects, not a flat map. The exact path depends on the ClickUp module output structure. You'll need to use Array functions or a JSON parse to extract specific field values. Test with a real task to confirm the access pattern.

**Module 5A-3 — ClickUp: Update a Task**

```
Type: ClickUp > Update a Task
Purpose: Update the existing task with merged data
Settings:
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
  Tags: Add "WIELOKROTNY KONTAKT"
  Priority: {{if(new_contact_count > 1 AND length(existing_task.assignees) = 0; "Urgent"; existing_task.priority)}}
```

**Logic:** Only overwrite a Custom Field if the existing value is empty/default AND the new submission provides a non-empty value. Never overwrite existing data with new data — the first value is likely more accurate (from a more structured source or earlier in the conversation).

**Module 5A-4 — ClickUp: Create a Comment**

```
Type: ClickUp > Create a Comment
Purpose: Append the new inquiry content as a comment on the existing task
Settings:
  Task ID: {{existing_task_id}}
  Comment body: {{merge_comment_body}}
```

**Module 5A-5 — Data Store: Update a Record**

```
Type: Data Store > Update a Record
Purpose: Refresh the timestamp so the dedup window extends from the latest contact
Settings:
  Data store: EVSO_Lead_Dedup
  Key: {{form_fields.email}}
  Fields:
    created_at: {{now}}
```

**Scenario ends here for Branch A.** No AI call, no new task creation, no draft generation. The duplicate is merged silently into the existing task.

---

#### Branch B — New Lead (Modified Phase 1 Flow + Draft Generation)

Modules 5B-1 through 5B-3 are the original Phase 1 Modules 4, 5, and 6 (AI Classification, JSON Parse, Value Mapping) — **moved here from their previous position but otherwise unchanged.** See Phase 1 guide Section 4, Modules 4-6 for configuration.

**Module 5B-4 — ClickUp: Create a Task (Modified)**

This is the Phase 1 Module 7 with two additions:

```
Additions to the existing ClickUp Create Task module:

  1. Due Date:
    Value: {{addHours(now; 48)}}
    Condition: Only set if clickup_klasyfikacja != "Spam"
    Implementation: Wrap in IF:
      {{if(clickup_klasyfikacja != "Spam"; addHours(now; 48); "")}}

  2. Custom Field — LICZBA_KONTAKTÓW:
    Value: 1

All other fields remain exactly as documented in Phase 1 guide, Section 4, Scenario 1, Module 7.

Output: Save the newly created task_id for use in subsequent modules.
```

**Module 5B-5 — Data Store: Add a Record**

```
Type: Data Store > Add a Record
Purpose: Store this lead in the dedup lookup table
Settings:
  Data store: EVSO_Lead_Dedup
  Key: {{form_fields.email}}
  Fields:
    email: {{form_fields.email}}
    task_id: {{created_task_id}}
    task_name: {{task_title}}
    created_at: {{now}}
    phone: {{form_fields.telefon}}
  Overwrite: Yes (if same email exists, overwrite — this handles edge cases where a record wasn't cleaned up)
```

**Error handler:**

```
Error handler: Resume
  On error: Continue (Data Store write failure should never block task creation.
  The task already exists in ClickUp — dedup just won't work for this lead.)
```

**Module 5B-6 — Tools: Set Multiple Variables (Prepare Draft Context)**

```
Type: Tools > Set Multiple Variables
Purpose: Prepare context variables for the draft reply generation prompt
Settings:
  Variable 1:
    Name: missing_fields_list
    Value: Build a comma-separated list of missing fields:
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

  Variable 2:
    Name: customer_first_name
    Value: Extract first name from full name:
      {{first(split(form_fields.imie_nazwisko; " "))}}

  Variable 3:
    Name: service_name_polish
    Value:
      {{if(service_type = "TRAM"; "tramwaj imprezowy";
        if(service_type = "BOAT"; "statek imprezowy";
        if(service_type = "BUS"; "bus imprezowy";
        "event")))}}
```

**Note:** Make.com may not support `filter()` with the exact syntax above. Adapt using a series of IF statements to build the missing fields string. The key requirement is: produce a human-readable Polish list of what's missing so the AI prompt can reference it.

**Module 5B-7 — Router (Draft Reply)**

```
Type: Router
Purpose: Route to appropriate draft generation prompt based on classification
Settings:
  Branch B1:
    Label: "Draft — niekompletny lead"
    Condition: clickup_klasyfikacja = "Niekompletny lead" AND clickup_confidence != "Niska"
  Branch B2:
    Label: "Draft — gorący lead"
    Condition: clickup_klasyfikacja = "Gorący lead" AND clickup_confidence != "Niska" AND hot_lead_draft_enabled = true
  Branch B3:
    Label: "Brak draftu"
    Condition: (fallback — all other cases)
```

**Module 5B-7a — HTTP: Make a Request (Incomplete Lead Draft)**

```
Type: HTTP > Make a Request
Purpose: Generate a draft follow-up reply for incomplete leads
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

**Note on `{{brand_voice_examples}}`:** This is a placeholder for real example replies collected from the sales team. Start with an empty string. Once 5-10 real reply examples are collected (prerequisite U-003), add them to the system prompt as: `\n\nPrzykłady rzeczywistych odpowiedzi zespołu (naśladuj ten ton):\n---\nPrzykład 1: [real reply text]\n---\nPrzykład 2: [real reply text]\n---`

```
Error handler: Resume
  On error: Skip comment creation. Log error. Continue scenario.
  The task already exists — missing a draft is not a failure.
```

**Module 5B-7b — JSON Parse (Draft Response)**

The AI returns plain text (not JSON) for draft replies. However, wrap this in a text extraction step:

```
Type: Tools > Set Variable
Purpose: Clean the draft reply text
Settings:
  Variable: draft_reply_text
  Value: {{5B-7a.response_body.content[].text}}
  Note: If the response contains markdown fences or extra formatting, strip them.
```

**Module 5B-7c — ClickUp: Create a Comment (Draft Reply)**

```
Type: ClickUp > Create a Comment
Purpose: Post the draft reply as a comment on the newly created task
Settings:
  Task ID: {{created_task_id}}
  Comment body: |
    🤖 WERSJA ROBOCZA ODPOWIEDZI (wygenerowana przez AI — do edycji przed wysłaniem)
    ---
    {{draft_reply_text}}
    ---
    ⚠️ Przejrzyj i dostosuj przed wysłaniem. Nie zawiera cen ani zobowiązań.
```

---

**Module 5B-7d — HTTP: Make a Request (Hot Lead Draft) — DISABLED AT LAUNCH**

```
Type: HTTP > Make a Request
Purpose: Generate a draft acknowledgment reply for hot leads
Settings:
  URL: https://api.anthropic.com/v1/messages
  Method: POST
  Headers: (same as 5B-7a)
  Body type: Raw (JSON)
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

**Module 5B-7e** and **5B-7f** follow the same pattern as 5B-7b and 5B-7c (clean response, post as ClickUp comment with the same format marker).

```
Error handler: Resume (same as Branch B1 — skip comment on failure)
```

---

### Scenario 2: Email Intake — Phase 2 Modifications

The modifications to Scenario 2 (Email Intake) mirror Scenario 1 exactly in structure. The differences are only in the data sources (email fields vs. form fields):

**Changes to apply:**

1. **Insert Data Store lookup** (same as Scenario 1, Module 4) — but use `{{from_email}}` instead of `{{form_fields.email}}` as the dedup key
2. **Insert Router** with Branch A (duplicate) and Branch B (new lead) — same logic
3. **Branch A merge comment** uses email-specific format:
   ```
   --- DODATKOWY KONTAKT (Email) --- {{formatDate(now; "DD-MM-YYYY HH:mm")}} ---
   Od: {{from_name}} <{{from_email}}>
   Temat: {{subject}}
   Treść: {{text_body}}
   ---
   ```
4. **Branch B task creation** adds Due Date and LICZBA_KONTAKTÓW (same as Scenario 1)
5. **Branch B Data Store write** uses `{{from_email}}` as the key
6. **Branch B draft reply generation** uses email-specific context in the prompt:
   - Replace `{{form_fields.imie_nazwisko}}` with `{{final_name}}`
   - Replace `{{source_brand}}` with `{{email_brand}}`
   - Replace `{{form_fields.*}}` references with the AI-extracted `{{parsed.*}}` values
   - missing_fields_list is built from parsed fields (null checks instead of empty-string checks)

**Implementation shortcut:** Clone the modified Scenario 1's new modules (Data Store lookup, Router, Branch A merge flow, Branch B draft branches) and adapt the field references. The logic is identical — only the data source paths change.

---

### Scenario 3: EVSO - ZLECENIA Task Creator (NEW)

```
Scenario: EVSO - ZLECENIA Task Creator
Purpose: Automatically creates a task in ZLECENIA when a CRM lead is marked as SPRZEDANE
Trigger: ClickUp webhook — task status changed
Design: B (new task creation) — per Decision D-006
```

**Preliminary setup:**
1. Create a new Make.com scenario
2. Set up a ClickUp webhook in Make that triggers on task status changes
3. Configure the webhook to listen to the CRM folder (all seller pipeline lists)

#### Module 1 — Webhooks: Custom Webhook (ClickUp Event)

```
Type: Webhooks > Custom Webhook
Purpose: Receive ClickUp webhook notifications for status changes
Settings:
  Webhook name: "EVSO ZLECENIA Trigger"
  Note: Register this webhook URL in ClickUp (Space settings > Integrations > Webhooks)
        or use Make's native ClickUp trigger module if available:
        ClickUp > Watch Events > Task Status Updated
  Filter: Process only events where new_status = "SPRZEDANE"
Output variables:
  - task_id (the CRM task that changed status)
  - new_status (should be "SPRZEDANE")
  - list_id (which seller pipeline list)
```

**Alternative approach:** If Make's native ClickUp module supports "Watch Tasks" with status filter, use that instead of a custom webhook. Test both approaches — the native module is simpler to set up.

#### Module 2 — Filter: Verify Status

```
Type: Filter
Purpose: Only proceed if the status is SPRZEDANE (safety check)
Condition: new_status = "SPRZEDANE"
Note: This catches edge cases where the webhook sends other status changes.
```

#### Module 3 — ClickUp: Get a Task

```
Type: ClickUp > Get a Task
Purpose: Fetch the full CRM task with all Custom Fields
Settings:
  Task ID: {{task_id}}
Output variables:
  - crm_task (full task object)
  - All Custom Fields: MIASTO, USŁUGA, PAKIET, DATA EVENTU, EMAIL, TELEFON,
    ILOŚĆ OSÓB, GODZINA EVENTU, TYP IMPREZY, AI_PODSUMOWANIE, etc.
  - Task name, description, assignees
```

#### Module 4 — Tools: Set Multiple Variables (Build ZLECENIA Data)

```
Type: Tools > Set Multiple Variables
Purpose: Prepare all values for the ZLECENIA task
Settings:
  Variable 1:
    Name: zlecenie_title
    Value: Build extended naming convention with PACKAGE:
      {{crm_task.custom_fields.DATA_EVENTU}} | {{crm_task.custom_fields.MIASTO}} | {{crm_task.custom_fields.USŁUGA}} | {{crm_task.name_extracted_client}} | {{crm_task.custom_fields.PAKIET}}
    Note: Extract the client name from the CRM task title (4th segment after splitting by " | ")
      {{last(split(crm_task.name; " | "))}} may work if the client name is always last.
      Or extract from Custom Fields if available.

  Variable 2:
    Name: zlecenie_description
    Value: |
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

  Variable 3:
    Name: crm_task_url
    Value: https://app.clickup.com/t/{{crm_task.id}}
```

**Note on client name extraction:** The CRM task title format is `[DATE] | [CITY] | [SERVICE] | [NAME]`. The client name is the 4th segment. Use `split(crm_task.name; " | ")` and take index [3] (0-based). If Make's split function returns an array, use `get()` or array access. Test with a real task name to confirm.

#### Module 5 — ClickUp: Create a Task (ZLECENIA)

```
Type: ClickUp > Create a Task
Settings:
  Workspace: EVSO
  Space: EVSO
  Folder: ZLECENIA
  List: ZLECENIA (or the specific ZLECENIA list — verify the exact list name)
  Task Name: {{zlecenie_title}}
  Description: {{zlecenie_description}}
  Status: ZLECENIA BEZ ZALICZEK
  Due Date: {{crm_task.custom_fields.DATA_EVENTU}} (if available — this is the event date, which becomes the operational deadline)
  Custom Fields:
    - MIASTO: {{crm_task.custom_fields.MIASTO}}
    - USŁUGA: {{crm_task.custom_fields.USŁUGA}}
    - PAKIET: {{crm_task.custom_fields.PAKIET}}
    - DATA EVENTU: {{crm_task.custom_fields.DATA_EVENTU}}
    - GODZINA EVENTU: {{crm_task.custom_fields.GODZINA_EVENTU}}
    - EMAIL: {{crm_task.custom_fields.EMAIL}}
    - TELEFON: {{crm_task.custom_fields.TELEFON}}
    - ILOŚĆ OSÓB: {{crm_task.custom_fields.ILOŚĆ_OSÓB}}
Output:
  - zlecenie_task_id (the new ZLECENIA task ID)
```

**Verification required (U-005):** Custom Fields in the ZLECENIA list may not have the same field IDs as in the CRM folder. Check if ZLECENIA has its own Custom Field definitions. If field names match but IDs differ, map them manually in the module configuration. If ZLECENIA has additional fields (e.g., OBSŁUGA, JEDNOSTKA), leave them empty — the operations team fills these manually.

#### Module 6 — ClickUp: Create a Comment (on ZLECENIA task)

```
Type: ClickUp > Create a Comment
Purpose: Link the ZLECENIA task back to the CRM task
Settings:
  Task ID: {{zlecenie_task_id}}
  Comment body: |
    📋 Zlecenie utworzone automatycznie z CRM.
    Lead źródłowy: {{crm_task.custom_id}} — {{crm_task.name}}
    Link do CRM: {{crm_task_url}}
```

#### Module 7 — ClickUp: Create a Comment (on CRM task)

```
Type: ClickUp > Create a Comment
Purpose: Link the CRM task to the new ZLECENIA task
Settings:
  Task ID: {{task_id}} (the original CRM task)
  Comment body: |
    ✅ Zlecenie utworzone automatycznie.
    Link do zlecenia: https://app.clickup.com/t/{{zlecenie_task_id}}
    Status: ZLECENIA BEZ ZALICZEK
```

```
Error handling for entire scenario:
  On any unrecoverable error:
  1. Send email notification to engineering/ops contact with error details
  2. The CRM task remains at SPRZEDANE status — the handlowiec can create the ZLECENIA task manually
  3. Log: task_id, error message, timestamp
```

---

### Scenario 4: EVSO - Dedup Cleanup (NEW)

```
Scenario: EVSO - Dedup Cleanup
Purpose: Weekly cleanup of expired records in the EVSO_Lead_Dedup Data Store
Trigger: Schedule — runs once per week (e.g., Sunday 03:00)
```

#### Module 1 — Schedule Trigger

```
Type: Schedule
Settings:
  Run scenario: Once a week
  Day: Sunday
  Time: 03:00 (low-traffic period)
```

#### Module 2 — Data Store: Search Records

```
Type: Data Store > Search Records
Purpose: Find all records older than the dedup window
Settings:
  Data store: EVSO_Lead_Dedup
  Filter: created_at < {{addDays(now; -dedup_window_days)}}
  Limit: 100 (process in batches)
Output: Array of records to delete
```

#### Module 3 — Iterator

```
Type: Flow Control > Iterator
Purpose: Process each expired record individually
Settings:
  Array: {{Module 2 output records}}
```

#### Module 4 — Data Store: Delete a Record

```
Type: Data Store > Delete a Record
Purpose: Remove the expired record
Settings:
  Data store: EVSO_Lead_Dedup
  Key: {{iterator.email}}
```

```
Error handling: Resume
  On individual delete error: Skip and continue to next record.
  Log errors but don't stop the cleanup.
```

**Note:** If the Data Store has more than 100 expired records (unlikely at EVSO's volume), the scenario will need to run the search-iterate-delete loop multiple times. At ~100 leads/week and 7-day retention, expect ~100 active records maximum. The cleanup is a safety net, not a high-volume operation.

---

## Section 5 — AI Integration Specification

### Model selection

**Model:** `claude-haiku-4-5-20251001` via Anthropic API — same as Phase 1.

**Justification for draft replies:** Draft reply generation is simple template-guided text generation. The prompts are constrained (max 150 words, strict boundaries on content), and the output is reviewed by a human before sending. Haiku is sufficient. A larger model would produce marginally more polished text but at 10-20x the cost, with no measurable improvement in the outcome (the handlowiec edits the draft anyway).

**Estimated additional API cost:** Phase 2 adds ~1 AI call per non-spam, non-duplicate lead. At ~80 qualifying leads/week (100 total minus spam and duplicates), with ~500 tokens input and ~300 tokens output: ~$1-2/month additional. Total system (Phase 1 + Phase 2): ~$3-5/month.

### AI Task 3: Incomplete Lead Draft Reply

**Task name:** Draft Follow-up for Incomplete Leads

**When used:** Scenario 1 and 2 (both intakes), Branch B1 of the Draft Reply Router

**System prompt (copy-paste ready):**

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

**User message template:**

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

**Expected output (example):**

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

### AI Task 4: Hot Lead Acknowledgment Draft

**Task name:** Draft Acknowledgment for Hot Leads

**When used:** Scenario 1 and 2, Branch B2 of the Draft Reply Router (DISABLED at launch)

**System prompt:** Same as AI Task 3.

**User message template:**

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

**Expected output (example):**

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

### Explicit AI boundaries (Phase 2 additions)

In addition to the Phase 1 AI boundaries (no pricing, no routing, no customer messaging, no qualification decisions, no data invention), Phase 2 adds:

5. **Draft replies are NEVER sent automatically.** The AI generates text that appears as a ClickUp comment. A human must copy it to the email composer, edit it, and send it manually. There is no automated send.
6. **Drafts must not contain pricing.** Even if the system has pricing data in the future, draft replies must not include it. Pricing is a business decision made per-inquiry.
7. **Drafts must not make promises.** "We'll prepare an offer" is acceptable (it's a process statement). "We can accommodate your group" is not (it's a commitment about capacity).
8. **No draft generation for low-confidence classifications.** If AI_CONFIDENCE is "Niska," the classification itself is uncertain — a draft based on an uncertain classification could be misleading.
9. **No draft generation for duplicates.** If a lead is identified as a duplicate and merged, no draft is generated. The original task may already have a draft or be in active handling.

### Failure fallback (Phase 2)

For draft reply generation: If the Anthropic API call fails, no comment is posted. The task exists without a draft. The handlowiec writes manually. This is identical to the pre-Phase-2 workflow — no regression.

For dedup Data Store lookup: If the lookup fails, the scenario proceeds as if no duplicate exists. Task creation continues normally. Dedup is best-effort, not mission-critical.

For ZLECENIA task creation: If the scenario fails, the CRM task remains at SPRZEDANE status. The handlowiec creates the ZLECENIA task manually, as they do today. An error notification is sent to engineering.

---

## Section 6 — Implementation Sequence

### Pre-implementation day: ZLECENIA Investigation (Decision D-007)

```
Day 0: ZLECENIA process investigation
  Tasks:
    1. Ask the sales team: "When you win a deal, what do you do? Do you move the task
       or create a new one in ZLECENIA?" — observe or record the current process
    2. Check: Do ZLECENIA tasks have LEAD-#### IDs? (If yes → likely task move / Design A)
    3. Check: Do closed CRM tasks still exist in seller pipeline views? (If yes → likely new task / Design B)
    4. Check: What data is added during the CRM→ZLECENIA transition? (OBSŁUGA, JEDNOSTKA, etc.)
    5. Decision: Confirm Design B (new task creation) or switch to Design A (task move)
    6. If Design B is confirmed: Proceed with Scenario 3 as documented
       If Design A: Adapt Scenario 3 — use ClickUp native Automation to move task,
       Make scenario handles post-move enrichment only (see meeting output M-002 for Design A spec)
    7. Collect 5-10 real reply examples from the sales team (prerequisite U-003 for draft generation)
  Prerequisite: None — can run before or in parallel with Phase 2 technical work
  Deliverable: Confirmed ZLECENIA design decision, collected reply examples
```

### Week 1: Deduplication + Follow-up Reminders

```
Day 1: Data Store setup + ClickUp changes
  Tasks:
    1. Create Make Data Store "EVSO_Lead_Dedup" (see Section 4, Data Store Setup)
    2. Add scenario variables: dedup_window_days = 7, hot_lead_draft_enabled = false
    3. Add Custom Field LICZBA_KONTAKTÓW to NOWE ZAPYTANIA (Section 3.2)
    4. Add SPRZEDANE status to all 4 seller pipeline lists (Section 3.3) — verify status structure first (U-002)
    5. Create ClickUp Automation "Przypomnienie 48h" (Section 3.4, Automation 1)
    6. Create ClickUp Automation "Eskalacja — wielokrotny kontakt" (Section 3.4, Automation 2)
    7. Verify ClickUp plan supports time-based Automation triggers (U-004)
    8. Add LICZBA_KONTAKTÓW column to existing views (optional)
  Prerequisite: Phase 1 live and stable
  Deliverable: All ClickUp + Data Store configuration complete

Day 2: Modify Form Intake scenario — dedup + Due Date
  Tasks:
    1. Clone "EVSO - Form Intake Enhanced" as backup (disable clone)
    2. Insert Module 4: Data Store Search (dedup check) — after completeness, before AI
    3. Insert Module 5: Router (dedup vs. new)
    4. Build Branch A: duplicate merge flow (Modules 5A-1 through 5A-5)
    5. Move Phase 1 Modules 4-7 into Branch B
    6. Add Due Date to Branch B task creation module
    7. Add LICZBA_KONTAKTÓW = 1 to Branch B task creation module
    8. Add Data Store write (Module 5B-5) after task creation in Branch B
    9. Add error handlers for Data Store modules
    10. Do NOT add draft reply modules yet (Day 4)
  Prerequisite: Day 1
  Deliverable: Form Intake scenario modified with dedup + Due Date

Day 3: Modify Email Intake scenario — dedup + Due Date
  Tasks:
    1. Clone "EVSO - Email Intake" as backup (disable clone)
    2. Apply same modifications as Day 2, adapted for email field references
    3. Use from_email as dedup key instead of form_fields.email
    4. Merge comment uses email-specific format
    5. Add error handlers
  Prerequisite: Day 1
  Deliverable: Email Intake scenario modified with dedup + Due Date

Day 4: Test dedup + reminders
  Tasks:
    1. Re-run all Phase 1 smoke tests (Tests 1-7 from Phase 1 guide) on modified scenarios
       — confirm no regressions in basic intake flow
    2. Test dedup: submit same email address twice via form within 5 minutes
       — verify second submission merges into first task (comment added, LICZBA_KONTAKTÓW = 2, tag added)
    3. Test dedup merge enrichment: first submission missing date, second submission has date
       — verify DATA EVENTU field is updated on the existing task
    4. Test dedup across channels: submit form, then send email with same email address
       — verify merge works cross-channel
    5. Test dedup window: manually set a Data Store record's created_at to 8 days ago,
       submit with that email — verify new task is created (no match outside window)
    6. Test Due Date: create task, wait 48h (or manually set Due Date to now-1h for testing),
       verify ClickUp Automation fires (Priority → Urgent, comment posted)
    7. Test Data Store error: temporarily rename the Data Store, submit form
       — verify task is created normally (dedup degrades, intake continues)
    8. Fix any issues
  Prerequisite: Days 2-3
  Deliverable: Dedup and reminder features validated
```

### Week 2: Draft Replies + ZLECENIA

```
Day 5: Add draft reply generation to Form Intake scenario
  Tasks:
    1. Add Module 5B-6: Prepare draft context (missing fields, customer name, service name)
    2. Add Module 5B-7: Router for draft reply
    3. Build Branch B1: Incomplete lead draft (HTTP call + parse + ClickUp comment)
    4. Build Branch B2: Hot lead draft (HTTP call + parse + ClickUp comment) — build but DISABLE
    5. Build Branch B3: No action (fallback)
    6. Add error handlers for draft API calls
    7. If reply examples are available (from Day 0), integrate into system prompt
       If not available, use generic prompt (mark for update later)
  Prerequisite: Day 4 (dedup tested), Day 0 (examples if available)
  Deliverable: Draft reply generation added to Form Intake

Day 6: Add draft reply generation to Email Intake scenario
  Tasks:
    1. Apply same draft modules as Day 5, adapted for email field references
    2. Test independently
  Prerequisite: Day 5
  Deliverable: Draft reply generation added to Email Intake

Day 7: Test draft replies
  Tasks:
    1. Submit form with incomplete data (missing date, city)
       — verify draft comment appears on task with incomplete-lead template
    2. Submit form with complete data (hot lead)
       — verify NO draft comment appears (hot-lead branch disabled)
    3. Enable hot_lead_draft_enabled temporarily, submit complete form
       — verify draft comment appears with acknowledgment template
       — disable hot_lead_draft_enabled again
    4. Verify draft comment format matches spec (🤖 marker, --- separators, ⚠️ footer)
    5. Submit form classified as Spam — verify no draft generated
    6. Submit form with AI_CONFIDENCE = Niska — verify no draft generated
    7. Test draft AI failure: temporarily break API key for draft call only
       — verify task is created normally, just no draft comment
    8. Verify dedup + draft interaction: submit duplicate — verify no draft generated for the merge
    9. Read 5 generated drafts with the sales team — collect initial feedback on tone and usefulness
  Prerequisite: Days 5-6
  Deliverable: Draft reply generation validated

Day 8: Build ZLECENIA Task Creator scenario
  Tasks:
    1. Create new Make scenario "EVSO - ZLECENIA Task Creator"
    2. Set up trigger: ClickUp webhook or Watch module for status changes
    3. Add filter: only SPRZEDANE status
    4. Add Module 3: Get Task (fetch CRM task data)
    5. Add Module 4: Variable builder (ZLECENIA title, description, links)
    6. Add Module 5: Create Task in ZLECENIA list
    7. Add Module 6: Comment on ZLECENIA task (cross-link to CRM)
    8. Add Module 7: Comment on CRM task (cross-link to ZLECENIA)
    9. Add error handler with email notification
    10. Test Custom Field mapping between CRM and ZLECENIA (U-005)
  Prerequisite: Day 0 (investigation confirms design), Day 1 (SPRZEDANE status added)
  Deliverable: ZLECENIA scenario built

Day 9: Test ZLECENIA automation
  Tasks:
    1. Create a test task in a seller pipeline with full Custom Fields populated
    2. Move task to SPRZEDANE status
    3. Verify: new task appears in ZLECENIA with correct title (extended naming with PAKIET)
    4. Verify: all mapped fields are populated correctly
    5. Verify: cross-link comments exist on both tasks
    6. Verify: CRM task stays in seller pipeline at SPRZEDANE status
    7. Verify: ZLECENIA task status is ZLECENIA BEZ ZALICZEK
    8. Test with missing fields (e.g., no PAKIET) — verify graceful handling
    9. Test error case: temporarily break the scenario, move task to SPRZEDANE
       — verify error notification sent, CRM task unaffected
    10. Fix any issues
  Prerequisite: Day 8
  Deliverable: ZLECENIA automation validated
```

### Week 3: Cleanup, Stabilization, Handover

```
Day 10: Build Dedup Cleanup scenario + full integration test
  Tasks:
    1. Create new Make scenario "EVSO - Dedup Cleanup" (Section 4, Scenario 4)
    2. Schedule for Sunday 03:00
    3. Test: manually insert 5 records with created_at = 10 days ago in Data Store
       — run cleanup scenario — verify records are deleted
    4. Full integration test: simulate a lead's entire lifecycle:
       a. Submit form (task created, dedup record stored, draft comment posted, Due Date set)
       b. Submit same email again via email channel (dedup merge, comment appended, LICZBA_KONTAKTÓW = 2)
       c. Assign the task to a seller (stops follow-up reminder from escalating)
       d. Move task through seller pipeline to SPRZEDANE
       e. Verify ZLECENIA task created automatically
       f. Wait for cleanup cycle — verify dedup record cleaned up
    5. Run full Phase 1 smoke test suite to confirm no regressions
  Prerequisite: All scenarios built
  Deliverable: All 4 features working end-to-end

Day 11: Sales team walkthrough + feedback
  Tasks:
    1. Demonstrate new features to sales team:
       - Due Date reminders and the ⚠️ escalation comment
       - Dedup merges: how WIELOKROTNY KONTAKT tag and LICZBA_KONTAKTÓW work
       - Draft reply comments: how to read them, copy to email composer, edit, send
       - ZLECENIA auto-creation: what happens when they mark a deal as SPRZEDANE
    2. Let each handlowiec test the draft reply workflow: read a draft, copy to email, edit, send
    3. Collect feedback on draft reply quality and tone
    4. If feedback is positive: consider enabling hot_lead_draft_enabled
    5. Update WIKI PRACOWNIKA with documentation for all 4 new features
  Prerequisite: Day 10
  Deliverable: Team trained, feedback collected

Day 12: Refinements + documentation
  Tasks:
    1. Implement highest-priority feedback items from Day 11
    2. If real reply examples have been collected: update draft reply system prompts with brand voice examples
    3. Enable hot_lead_draft_enabled if team approves (or schedule for next week)
    4. Review dedup effectiveness: check Data Store for unexpected patterns
    5. Document all 4 Make scenarios: screenshot module configurations, save in shared folder
    6. Document manual fallback procedures:
       - "If dedup fails: leads will create separate tasks (same as Phase 1 behavior)"
       - "If draft generation fails: write reply manually (same as current behavior)"
       - "If ZLECENIA automation fails: create ZLECENIA task manually"
    7. Update Phase 1 smoke tests with Phase 2 additions (see Section 7)
    8. Write brief Phase 3 planning note based on remaining deferred features
  Prerequisite: Day 11
  Deliverable: System documented, refinements applied, Phase 2 complete
```

**Estimated total engineering time: 12 working days + 1 investigation day (Day 0) = 13 working days.** Within the 15-day constraint with 2 days of buffer.

---

## Section 7 — Smoke Test Checklist

### Phase 1 Regression Tests (run first)

```
Retest 1-7: Run all 7 Phase 1 smoke tests from the Phase 1 Implementation Guide
  Purpose: Confirm that Phase 2 scenario modifications did not break existing intake functionality
  Pass condition: All 7 tests produce the same results as Phase 1 (with the addition of Due Date and LICZBA_KONTAKTÓW = 1 on new tasks)
```

### Phase 2 Tests

```
Test 8: Dedup — Same Email via Form (within window)
  Action:
    Step 1: Submit form on partyboat.fun with email jan@test.pl, city: w Krakowie, date: 2026-06-15
    Step 2: Wait 2 minutes
    Step 3: Submit form on partytram.fun with email jan@test.pl, city: (empty), date: 2026-06-15
  Expected result:
    - Only ONE task exists in NOWE ZAPYTANIA (from Step 1)
    - Task has comment: "--- DODATKOWY KONTAKT (Formularz) ---" with Step 3 data
    - LICZBA_KONTAKTÓW = 2
    - Tag: WIELOKROTNY KONTAKT
    - MIASTO still = KRAKÓW (not overwritten by empty)
    - No second task created
  Pass condition: Merge successful, data enrichment works, no duplicate task.
```

```
Test 9: Dedup — Same Email Cross-Channel (form then email)
  Action:
    Step 1: Submit form on partyboat.fun with email anna@test.pl, missing date and city
    Step 2: Send email from anna@test.pl to kontakt@partyboat.fun with date and city included
  Expected result:
    - Only ONE task exists
    - Task has comment from email merge with new data
    - DATA EVENTU and MIASTO fields updated from the email submission
    - KOMPLETNOŚĆ increased
    - LICZBA_KONTAKTÓW = 2
  Pass condition: Cross-channel dedup works, field enrichment works.
```

```
Test 10: Dedup — Outside Window (no merge)
  Action:
    Step 1: Manually insert a record in EVSO_Lead_Dedup with email stary@test.pl and created_at = 8 days ago
    Step 2: Submit form with email stary@test.pl
  Expected result:
    - New task created (no merge)
    - LICZBA_KONTAKTÓW = 1
    - New Data Store record created (overwriting the old one)
  Pass condition: Dedup window correctly excludes old records.
```

```
Test 11: Dedup — Data Store Failure Graceful Degradation
  Action:
    Step 1: Temporarily rename EVSO_Lead_Dedup Data Store (or break the module reference)
    Step 2: Submit form with complete data
  Expected result:
    - Task created normally in NOWE ZAPYTANIA with all fields
    - AI classification works
    - Draft reply generated (if applicable)
    - Only dedup is skipped — everything else functions
  Pass condition: Data Store failure does not block intake.
  Post-test: Restore Data Store name/reference.
```

```
Test 12: Follow-up Reminder — Due Date Trigger
  Action:
    Step 1: Submit form (non-spam). Verify task created with Due Date = now + 48h.
    Step 2: For testing: manually set the task's Due Date to 1 hour ago in ClickUp.
    Step 3: Verify the ClickUp Automation fires (may take a few minutes).
  Expected result:
    - Priority set to Urgent
    - Comment posted: "⚠️ To zapytanie czeka na obsługę od 48h..."
  Pass condition: Automation triggers on Due Date arrival for unassigned tasks.
  Note: Spam tasks should NOT have a Due Date set. Verify by submitting a spam-classified inquiry.
```

```
Test 13: Follow-up Reminder — Assigned Task (no trigger)
  Action:
    Step 1: Submit form, verify task created with Due Date
    Step 2: Assign the task to any user
    Step 3: Manually set Due Date to past
  Expected result:
    - Automation does NOT fire (task has an Assignee)
    - Priority remains unchanged
  Pass condition: Reminder only triggers for unassigned tasks.
```

```
Test 14: Draft Reply — Incomplete Lead
  Action: Submit form on partyboat.fun with:
    - Name: Tomek Draft
    - Email: tomek@test.pl
    - Typ imprezy: Wieczór kawalerski
    - Date: (empty)
    - City: (empty)
    - Liczba osób: (empty)
    - Opis: "Chcemy zorganizować kawalerskie na statku"
  Expected result:
    - Task created with KLASYFIKACJA_AI = Niekompletny lead
    - Comment appears on task with format:
      🤖 WERSJA ROBOCZA ODPOWIEDZI (wygenerowana przez AI — do edycji przed wysłaniem)
      ---
      [Polish text asking for miasto, data eventu, liczba osób]
      ---
      ⚠️ Przejrzyj i dostosuj przed wysłaniem. Nie zawiera cen ani zobowiązań.
    - Draft text is in Polish
    - Draft text does NOT contain any prices or commitments
    - Draft text mentions the specific missing fields
  Pass condition: Draft generated, correctly formatted, content-safe.
```

```
Test 15: Draft Reply — Hot Lead (disabled)
  Action: Submit form on partyboat.fun with all fields filled (same as Phase 1 Test 1)
  Expected result:
    - Task created with KLASYFIKACJA_AI = Gorący lead
    - NO draft reply comment (hot_lead_draft_enabled = false)
  Pass condition: Hot lead branch correctly disabled.
```

```
Test 16: Draft Reply — Hot Lead (enabled)
  Action:
    Step 1: Set hot_lead_draft_enabled = true
    Step 2: Submit form with all fields filled
  Expected result:
    - Task created with KLASYFIKACJA_AI = Gorący lead
    - Comment appears with draft acknowledgment reply
    - Draft confirms details back to customer
    - Draft does NOT contain prices
  Pass condition: Hot lead draft works when enabled.
  Post-test: Set hot_lead_draft_enabled = false again.
```

```
Test 17: Draft Reply — Spam (no draft)
  Action: Submit form or send email that will be classified as Spam
  Expected result:
    - Task created with KLASYFIKACJA_AI = Spam
    - NO draft reply comment
    - NO Due Date set
  Pass condition: Spam tasks get no draft and no reminder.
```

```
Test 18: Draft Reply — Low Confidence (no draft)
  Action: Submit a borderline/ambiguous inquiry that AI classifies with AI_CONFIDENCE = Niska
  Expected result:
    - Task created
    - NO draft reply comment (confidence gating blocks draft)
  Pass condition: Low-confidence leads don't get potentially misleading drafts.
  Note: If hard to trigger Niska confidence naturally, verify the router condition logic is correct.
```

```
Test 19: Draft Reply — AI Failure
  Action:
    Step 1: Temporarily set wrong API key for the draft generation HTTP module only
           (keep the classification API key correct)
    Step 2: Submit form with incomplete data
  Expected result:
    - Task created normally with correct classification and all fields
    - NO draft comment (draft AI call failed)
    - No error visible to user — silent degradation
  Pass condition: Draft failure doesn't affect task creation or classification.
  Post-test: Restore correct API key.
```

```
Test 20: ZLECENIA — Automatic Task Creation
  Action:
    Step 1: Create or use an existing task in any seller pipeline (e.g., SPRZEDAŻ KUBA) with:
      - Title: 15-06-2026 | KRAKÓW | BOAT | Jan Testowy
      - MIASTO: KRAKÓW
      - USŁUGA: BOAT
      - PAKIET: PARTY
      - DATA EVENTU: 2026-06-15
      - EMAIL: jan@test.pl
      - TELEFON: 500100200
      - ILOŚĆ OSÓB: 20
      - TYP IMPREZY: Wieczór kawalerski
    Step 2: Change task status to SPRZEDANE
  Expected result:
    - New task appears in ZLECENIA list
    - Title: "15-06-2026 | KRAKÓW | BOAT | Jan Testowy | PARTY" (extended with PAKIET)
    - Status: ZLECENIA BEZ ZALICZEK
    - All mapped fields populated (MIASTO, USŁUGA, PAKIET, DATA EVENTU, EMAIL, TELEFON, ILOŚĆ OSÓB)
    - Comment on ZLECENIA task: "📋 Zlecenie utworzone automatycznie z CRM..."
    - Comment on CRM task: "✅ Zlecenie utworzone automatycznie..."
    - CRM task remains in seller pipeline at SPRZEDANE status
  Pass condition: ZLECENIA task created with correct data, cross-links work, CRM task preserved.
```

```
Test 21: ZLECENIA — Missing Fields Graceful Handling
  Action: Move a CRM task with some empty fields (e.g., no PAKIET, no GODZINA) to SPRZEDANE
  Expected result:
    - ZLECENIA task created
    - Title handles missing PAKIET gracefully (empty segment or "BRAK")
    - Empty fields are left empty in ZLECENIA task (not filled with "null" or error text)
    - Cross-link comments still work
  Pass condition: Automation handles incomplete data without errors.
```

```
Test 22: ZLECENIA — Scenario Failure
  Action:
    Step 1: Temporarily break the ZLECENIA scenario (disable or misconfigure)
    Step 2: Move CRM task to SPRZEDANE
  Expected result:
    - Error notification email sent to engineering
    - CRM task remains at SPRZEDANE — no data loss
    - No ZLECENIA task created (manual fallback needed)
  Pass condition: Failure is notified, CRM task is safe.
  Post-test: Restore scenario.
```

```
Test 23: Dedup Cleanup
  Action:
    Step 1: Manually insert 3 records in EVSO_Lead_Dedup with created_at = 10 days ago
    Step 2: Insert 2 records with created_at = today
    Step 3: Run the EVSO - Dedup Cleanup scenario manually
  Expected result:
    - 3 old records deleted
    - 2 recent records preserved
  Pass condition: Cleanup deletes only expired records.
```

```
Test 24: End-to-End Lifecycle
  Action:
    Step 1: Submit form with email lifecycle@test.pl, incomplete data (missing city)
      → Verify: task created, LICZBA_KONTAKTÓW=1, draft follow-up posted, Due Date set
    Step 2: Send email from lifecycle@test.pl with city included
      → Verify: merge into existing task, MIASTO updated, LICZBA_KONTAKTÓW=2, tag added
    Step 3: Assign task to a seller
      → Verify: Due Date reminder will NOT fire (task now has Assignee)
    Step 4: Move task through seller pipeline to SPRZEDANE
      → Verify: ZLECENIA task created with all data
    Step 5: Check Data Store — record exists with lifecycle@test.pl
    Step 6: Run cleanup after record expires
      → Verify: record deleted
  Pass condition: Complete lead lifecycle works end-to-end with all Phase 2 features.
```

---

## Uncertainties and Investigation Items

These items from the Expert Meeting (M-002) must be verified during implementation:

| ID | Unknown | When to verify | Impact if wrong |
|----|---------|---------------|-----------------|
| U-001 | Current CRM→ZLECENIA handoff process | Day 0 | Wrong automation design |
| U-002 | Seller pipeline status consistency | Day 1 | SPRZEDANE may need per-list configuration |
| U-003 | EVSO brand voice for replies | Day 0 (collect examples) | Drafts sound generic |
| U-004 | ClickUp plan tier for Automations | Day 1 | May need Make-based alternative for reminders |
| U-005 | Custom Field inheritance across lists | Day 8 (ZLECENIA build) | Field mapping may need manual configuration |
| U-006 | Data Store performance at EVSO volume | Post-deploy monitoring | Should be fine (~100 records max) |
| U-007 | WPForms webhook payload includes email before AI call | Day 2 (scenario modification) | Dedup key may need adjustment |

---

*End of implementation guide. Total estimated effort: 12 engineer-days + 1 investigation day across 3 weeks.*
