# EVSO — Milestone 1: AI-Powered Form Intake System

**Status:** ✅ COMPLETED  
**Completion date:** 2026-04-30  
**Blueprint name:** `WEBHOOK -> AI -> CLICKUP`  
**Webhook name:** `Forms_ai_clickup`

---

## What this milestone delivers in one sentence

Every inquiry submitted through any EVSO web form is automatically classified by AI, enriched with data extracted from the client's free-text description, and lands as a fully structured task in ClickUp — without any manual human intervention.

---

## Scope: what the system processes

**Input: 5 contact forms across 4 EVSO websites**

| Website | Service | Form type |
|---------|---------|-----------|
| partytram.fun | TRAM | Forminator (select_2, name_1, etc.) |
| tramparty.pl | TRAM | Smart form (sm-city, sm-name, etc.) |
| partyboat.fun | BOAT | Forminator |
| boatparty.pl | BOAT | Smart form |
| busparty.fun | BUS | Forminator / Smart form |

All 5 forms send to a **single shared webhook** — one Make.com scenario handles everything.

**Output: ClickUp task in `EVSO → CRM → NOWE FORMULARZE`**

From there, ClickUp native automations route the task to the correct salesperson list based on city:
- SPRZEDAŻ KUBA
- SPRZEDAŻ IZA  
- SPRZEDAŻ WIKTOR
- (and others as configured)

---

## Make.com Scenario — module-by-module

### Module 1 — Webhook (`gateway:CustomWebHook`)
- Single entry point for all 5 forms
- Hook ID: 4056662 (`Forms_ai_clickup`)
- Accepts any form submission payload

### Module 33 — Field Normalization (`util:SetVariables`)
Maps raw form fields to unified variable names. Handles **two different form field naming conventions** using `ifempty()` fallbacks:

| Variable | Forminator field | Smart form field | Notes |
|----------|-----------------|------------------|-------|
| `miasto` | `select_2` | `sm-city` | Strips placeholder text "Wybierz miejscowość" |
| `rodzaj_imprezy` | `select_3` | — | Strips placeholder |
| `pakiet` | `select_1` | — | Strips placeholder |
| `imie_nazwisko` | `name_1` | `sm-name` | Strips placeholder |
| `email` | `email_1` | `sm-email` | Lowercased |
| `telefon` | `phone_1` | `sm-phone` | Strips placeholder |
| `liczba_osob` | `number_1` | `liczba-uczestnikow` | Strips placeholder |
| `opis` / `client_txt` | `textarea_1` | `sm-message` | Strips "Opisz swój pomysł" placeholder |
| `data_eventu` | `name_2` | `termin` | Strips "Data" placeholder |
| `godzina` | `name_3` | `godzina` | Strips "Godzina" placeholder |
| `source_url` | `current_url` | `wyslano_ze_strony` | Source website URL |

### Module 7 — Filter + Brand Detection (`util:SetVariables`)
**Filter "Niepusty formularz":** Blocks processing if ALL of: miasto, rodzaj_imprezy, pakiet, imie_nazwisko, email, telefon, liczba_osob, opis, data_eventu, godzina are empty simultaneously. Prevents spam/test submissions from creating tasks.

**Brand and service detection** from `source_url`:

| URL contains | `source_brand` | `service_type` |
|-------------|----------------|---------------|
| partyboat.fun | PARTYBOAT.FUN | BOAT |
| boatparty.pl | BOATPARTY.PL | BOAT |
| partytram.fun | PARTYTRAM.FUN | TRAM |
| tramparty.pl | TRAMPARTY.PL | TRAM |
| busparty | BUSPARTY.FUN | BUS |

### Module 52 — AI Classification + Extraction (`anthropic-claude:createAMessage`)
**Model:** Claude Haiku 4.5 (`claude-haiku-4-5-20251001`)  
**Temperature:** 0.1 (consistent, low-variation output)  
**Output format:** Native JSON Schema (no markdown wrapping, no stripping needed)

**What the AI does in one call:**
1. **Classifies** the inquiry into one of 4 categories
2. **Extracts** data from the client's free-text description — but ONLY for fields marked as `[PUSTE]` (empty in the form)
3. **Generates** a 1-2 sentence Polish summary

**Classification categories:**
- `goracy_lead` — has city + date + event type + contact → ready to quote
- `niekompletny_lead` — intent is clear, but key data missing
- `zapytanie_ogolne` — general pricing/availability question, not a concrete inquiry
- `spam` — unrelated, advertising, or incomprehensible

**Confidence levels:** `wysoka` / `srednia` / `niska`

**Extraction logic:** AI receives form data with fields labeled `[PUSTE]` for missing values. It reads ONLY from the free-text `client_txt` field. It does NOT overwrite existing form data. Fields already filled = `null` in the `ekstrakcja` object.

**Extraction normalizations:**
- `miasto` → nominative case, UPPERCASE (e.g., "KRAKÓW", "WROCŁAW")
- `rodzaj_imprezy` → one of: "Kawalerski", "Panieński", "Urodziny", "Firmówka", "Inne"
- `liczba_osob` → numeric string only ("60")
- `data_eventu` → YYYY-MM-DD format
- `godzina` → HH:MM format ("21" → "21:00")

**JSON Schema enforced:**
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

### Module 54 — Merge (`util:SetVariables`)
Merges form data with AI extraction. **Form data always has priority.** AI fills only what was blank.

```
merged_X = ifempty(form_X, ai_extraction_X)
```

Special handling:
- `merged_miasto` → forced `UPPER(TRIM())`
- `merged_rodzaj_imprezy` → normalized to UPPERCASE for ClickUp dropdown matching
- `merged_data` → `parseDate()` with fallback: tries YYYY-MM-DD, DD.MM.YYYY, DD-MM-YYYY formats
- `merged_email` → lowercased

### Module 55 — ClickUp Mapping + Completeness (`util:SetVariables`)
Converts merged values into ClickUp dropdown indices and calculates completeness.

**Completeness score formula (8 fields, 0-100%):**
```
round((has_name + has_email + has_phone + has_city + has_date + 
       has_time + has_people_count + has_event_type) / 8 × 100)
```
Uses `length(trim(ifempty(field, ""))) > 0` check — correctly handles null and empty string.

**City → ClickUp index mapping (post-merge):**

| City | Index |
|------|-------|
| WROCŁAW | 0 |
| POZNAŃ | 1 |
| KRAKÓW | 2 |
| GDAŃSK | 3 |
| ŁÓDŹ | 4 |
| BYDGOSZCZ | 5 |
| TORUŃ | 6 |
| KATOWICE | 7 |
| SZCZECIN | 8 |
| WARSZAWA | 9 |
| SOPOT | 10 |

**Service → ClickUp index:** BOAT=0, TRAM=1, BUS=2

**Event type → ClickUp index:** KAWALERSKI=0, PANIEŃSKI=1, URODZINY=2, FIRMÓWKA=3, INNE=4

**Package → ClickUp index:** FUN=0, PARTY=1, LUX=2, INDYVIDUAL!=3, INDIVIDUAL II=4

**Source → ClickUp index:** MAIL=0, TELEFON=1, PARTYBOAT.FUN=2, BOATPARTY.PL=3, PARTYTRAM.FUN=4, TRAMPARTY.PL=5

**Task title format:**
```
DD-MM-YYYY | MIASTO | USŁUGA | IMIĘ NAZWISKO
```
Falls back to: `BRAK DATY`, `BRAK MIASTA`, `BRAK USŁUGI`, `BRAK IMIENIA` if missing.

### Module 20 — AI Label Mapping (`util:SetVariables`)
Converts AI response codes to human-readable Polish labels for ClickUp:

| AI code | ClickUp label |
|---------|--------------|
| goracy_lead | Gorący lead |
| niekompletny_lead | Niekompletny lead |
| zapytanie_ogolne | Zapytanie ogólne |
| spam | Spam |
| (error/missing) | Do weryfikacji |

### Module 2 — Create ClickUp Task (`clickup:createTaskInList`)
**Destination:** `EVSO / CRM / NOWE FORMULARZE` (list_id: 901508747847)  
**Status on creation:** `nowe zapytania`

**Custom Fields populated:**

| Field label | UUID | Value source |
|-------------|------|-------------|
| ILOŚĆ OSÓB | `19536d62-...` | `merged_liczba_osob` |
| TELEFON | `234795d0-...` | `merged_telefon` |
| DATA EVENTU | `29799a60-...` | `clickup_data_eventu` (parsed date) |
| MIASTO | `35ee7b29-...` | `clickup_miasto` (dropdown index) |
| POZYSKANIE | `8546df9d-...` | `clickup_pozyskanie` (dropdown index) |
| TYP IMPREZY | `b65ab644-...` | `clickup_typ_imprezy` (dropdown index) |
| PAKIET | `bec02f93-...` | `clickup_pakiet` (dropdown index) |
| EMAIL | `d8265463-...` | `merged_email` |
| GODZINA EVENTU | `da316850-...` | `merged_godzina` |
| USŁUGA | `ea3c032c-...` | `clickup_usluga` (dropdown index) |

**Task description (content):**
```
Zapytanie z formularza:

Oryginalny tekst: [client's free text]
Uwagi AI: [AI-generated Polish summary]
```

---

## What ClickUp does after task creation (native automations)

Tasks land in `NOWE FORMULARZE`. ClickUp automations then route them automatically:

```
NOWE FORMULARZE
     │
     ├─ MIASTO = WROCŁAW / POZNAŃ → SPRZEDAŻ KUBA  
     ├─ MIASTO = KRAKÓW / GDAŃSK → SPRZEDAŻ WIKTOR
     ├─ MIASTO = BYDGOSZCZ / ... → SPRZEDAŻ IZA
     └─ (other conditions as configured)
```

These automations run natively in ClickUp — no Make involvement needed.

---

## Data flow summary

```
Client fills form on website
         │
         ▼
Forminator/Smart form → Webhook (Make)
         │
         ▼ Module 33
Normalize fields (handle 2 form types, strip placeholders)
         │
         ▼ Module 7 [FILTER: not empty]
Detect brand + service from URL
         │
         ▼ Module 52
Claude Haiku 4.5:
  • Classify inquiry (4 categories)
  • Extract missing data from free text
  • Generate summary in Polish
  • JSON schema enforced → clean structured output
         │
         ▼ Module 54
Merge: form data + AI extraction
  (form always wins, AI fills blanks)
         │
         ▼ Module 55
Map to ClickUp indices
Calculate completeness score (0-100%)
Format task title + date
         │
         ▼ Module 20
Translate AI codes → Polish labels
         │
         ▼ Module 2
Create ClickUp task in NOWE FORMULARZE
  + populate all 10 custom fields
         │
         ▼ [ClickUp native automation]
Route task to correct salesperson list
```

---

## Key technical decisions made during M1

**1. Single webhook for all 5 forms**  
One scenario handles everything. `ifempty()` fallbacks resolve field name differences between form types. No duplicate scenarios to maintain.

**2. `createAMessage` with JSON Schema output**  
Native structured output from Anthropic API — no markdown wrapping, no regex stripping, no JSON Parse module needed. Output accessed directly via `52.jsonResponse.klasyfikacja` etc. Temperature 0.1 for consistency.

**3. AI extracts only from [PUSTE] fields**  
The prompt explicitly marks empty fields with `[PUSTE]`. AI is instructed to fill ONLY those and return `null` for fields that already have form data. Prevents AI from overwriting clean form values with its own interpretation.

**4. Post-merge completeness scoring**  
Score calculated AFTER AI extraction — so a client who wrote everything in free text gets a realistic completeness score, not a falsely low one. Uses `length() > 0` check to correctly handle both `null` and `""` in Make.

**5. Filter before AI call**  
Completely empty submissions (test clicks, bot pings) are dropped before reaching Claude. Saves API cost and avoids creating garbage tasks.

**6. NOWE FORMULARZE → ClickUp automations**  
Routing to salesperson-specific lists is done via native ClickUp automations, not Make. Keeps Make lean and puts routing logic where the team can configure it without touching the integration.

---

## Limitations (known, accepted)

- **No email intake in M1.** Cold emails from clients not captured by this scenario — addressed in Milestone 2.
- **No AI fields in ClickUp custom fields.** `KLASYFIKACJA_AI`, `AI_PODSUMOWANIE`, `KOMPLETNOŚĆ`, `AI_CONFIDENCE` fields were designed but the final working blueprint stores the AI summary in task description (`Uwagi AI`) rather than dedicated Custom Fields. Can be added if ClickUp workspace has these fields created.
- **Date parsing assumes current year is 2026.** AI is told the current year; if client writes "20 czerwca" without year, AI fills "2026-06-20". Will need updating in 2027.
- **Service detection from URL.** If a form is submitted without a valid source URL (e.g., direct API call), `service_type` will be empty and ClickUp USŁUGA field will not be set.
- **City spelling variations.** AI normalizes to standard UPPERCASE, but uncommon spelling or abbreviations may still miss the ClickUp dropdown mapping (→ empty field instead of wrong value).

---

## Files

| File | Description |
|------|-------------|
| `WEBHOOK -- AI -- CLICKUP.blueprint.json` | Production blueprint (client's Make account) |
| `EVSO_Form_Intake_v2.blueprint.json` | Reference blueprint from design phase |
| `EVSO_Meeting_M001_Form_Intake_Redesign.md` | DevTeam meeting that produced the design |
| `EVSO_Milestone2_Specification.md` | Next milestone: email ↔ ClickUp threading |

---

*Documented: 2026-04-30 based on production blueprint analysis.*
