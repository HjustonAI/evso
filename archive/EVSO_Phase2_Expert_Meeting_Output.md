# Meeting Output: EVSO Phase 2 — Design Workshop for Post-Intake Automation

**Date**: 2026-03-31
**Format**: Structured Workshop
**Team Charter**: EVSO DevTeam (Kurek, Sowa, Bryła, Wysocka)
**Meeting ID**: M-002

---

## Meeting Goal

**As stated**: "Start Phase 2 with Team Meeting. Exclude multilanguage, analytic dashboard, and automatic lead routing. Focus solely on rest."

**As understood**: Design the implementation architecture for four Phase 2 features that were deferred from Phase 1: (1) draft reply generation, (2) lead deduplication, (3) follow-up reminders, and (4) ZLECENIA workflow automation. The meeting must produce concrete, convergent design decisions for each — not ideation. Three items from the original Phase 1 deferred list are explicitly excluded: multi-language support, analytics dashboard, and automatic lead routing.

**Target outcome**: Action plan — specific design decisions for each feature, sufficient to write an implementation guide of the same quality as the Phase 1 guide.

---

## Context Summary

### What Is Known

- [FACT] Phase 1 has been designed and delivers: enhanced form intake via Make.com, new email intake via Make.com, AI classification/extraction using Claude Haiku, 5 new Custom Fields (ŹRÓDŁO_KANAŁ, KLASYFIKACJA_AI, AI_PODSUMOWANIE, KOMPLETNOŚĆ, AI_CONFIDENCE), 2 new ClickUp Views, 1 ClickUp Automation.
- [FACT] Phase 1 scenarios already call Anthropic API for classification/summary. The API connection, key storage, and error handling patterns are established.
- [FACT] NOWE ZAPYTANIA is the central intake list with 11 original + 5 new Custom Fields. Tasks use LEAD-#### IDs and naming convention `[DATE|BRAK DATY] | [CITY|WYBIERZ MIASTO] | [SERVICE] | [NAME]`.
- [FACT] Seller-specific pipelines exist: SPRZEDAŻ KUBA, SPRZEDAŻ WIKTOR, SPRZEDAŻ DAWID, SPRZEDAŻ KRIS. Observed statuses on SPRZEDAŻ DAWID board: NOWE ZAPYTANIE → WYSŁANO OFERTĘ → PRZEDZWONKA → ZASTANAWIA SIĘ → SZANSA SPRZEDAŻY → [truncated].
- [FACT] ZLECENIA is a separate folder/list with statuses: ZLECENIA BEZ ZALICZEK, REZERWACJA JEDNOSTKI, ZLECENIA, REZYGNACJA, ANULOWANY, DO ROZLICZENIA, ZREALIZOWANE, 2025, PRZEGRANY. Task naming extends to include PACKAGE.
- [FACT] ClickUp Email in Task is already used — the email composer is visible on lead tasks.
- [FACT] Task descriptions store the original customer inquiry.
- [FACT] Activity panel on tasks shows chronological history of changes, comments, and emails.

### What Is Uncertain

| Unknown | Why it matters | Assumed in absence |
|---------|---------------|-------------------|
| U-001: Current CRM→ZLECENIA handoff process | Determines whether automation should move tasks or create new ones | The meeting provides two alternative designs; engineer must verify |
| U-002: Consistent "won" status across all seller pipelines | Without it, there's no reliable trigger for ZLECENIA automation | Assume we must add a SPRZEDANE status to each seller list |
| U-003: EVSO's brand voice/tone for email replies | Draft reply quality depends on matching the company's communication style | Engineer must collect 5-10 real example replies before writing the system prompt |
| U-004: ClickUp plan capabilities for time-based Automations | Some Automations (e.g., "task has been in status for X days") require specific ClickUp plans | Assume Business plan; verify before implementation |
| U-005: Whether seller pipelines share Custom Field definitions with NOWE ZAPYTANIA | Affects whether field data transfers cleanly when tasks are moved between lists | Assume Custom Fields are at Folder level (CRM folder) so they're shared |

### Constraints Applied

1. **Single engineer, limited time** — total Phase 2 must not exceed 15 engineer-days.
2. **No-code/low-code first** — use ClickUp native Automations over Make scenarios wherever possible.
3. **AI assists, humans decide** — draft replies are suggestions, never auto-sent.
4. **Manual fallback** — every automation must work when Make or AI is down.
5. **Safe for non-technical handover** — sales team must be able to use all features without engineering support.
6. **Phase 1 must remain stable** — changes to existing Phase 1 scenarios must be additive (router branches), not destructive.

---

## Expert Participation

| Expert | Role | Participation Level | Primary Contribution Area |
|--------|------|-------------------|--------------------------|
| Marta Kurek | ClickUp Systems Architect | Primary | ClickUp structure for all 4 features; ZLECENIA transition design; native Automation capabilities |
| Tomasz Sowa | Make Automation Architect | Primary | Make scenario design for dedup and draft replies; Data Store architecture; scenario modification strategy |
| Aleksander Bryła | AI Systems & Reliability Architect | Primary | Draft reply prompt design; confidence gating; AI boundary enforcement; dedup matching logic |
| Karolina Wysocka | Sales & Service Process Designer | Primary | Sales team workflow impact; draft reply tone/usefulness; follow-up process design; ZLECENIA handoff requirements |

All four experts are Primary — each feature touches all four domains.

---

## Discussion Summary

The workshop was decomposed into four work items, sequenced by dependency:

1. **Follow-up reminders** (simplest, least risk — sets a baseline pattern)
2. **Lead deduplication** (must be solved before draft replies, to avoid drafting replies for duplicates)
3. **Draft reply generation** (highest user-visible impact, depends on stable dedup)
4. **ZLECENIA workflow automation** (most uncertain, requires investigation)

---

### Work Item 1: Follow-up Reminders

#### Marta Kurek — ClickUp Systems Architect

[FACT] ClickUp supports time-based Automations natively: "When due date arrives" and "When task has been in status for [duration]" are both available trigger types on Business plans and above.

[RECOMMENDATION] Use ClickUp's native Due Date mechanism rather than building a Make polling scenario. The approach: Phase 1 Make scenarios should be updated to set Due Date = `now + 48 hours` on every newly created task where KLASYFIKACJA_AI is NOT "Spam." Then a ClickUp Automation fires when the Due Date arrives: if the task still has no Assignee, escalate it by setting Priority to Urgent and posting a comment that pings the team.

This is cleaner than "status duration" triggers because: (a) Due Date is visible in Calendar and List views, giving the team a deadline without opening the task, and (b) it doesn't depend on the task remaining in a specific status — if someone changes the status but doesn't assign it, the reminder still fires.

[FACT] The ClickUp comment can tag specific users or a Channel. The CRM Channel already exists.

#### Tomasz Sowa — Make Automation Architect

[INFERENCE] Agreed — ClickUp Automations are the correct tool here. Building this in Make would require a scheduled scenario (e.g., run every hour, search for tasks with expired Due Date, filter, send notification). That's more fragile, adds operations costs, and is slower to trigger.

[RECOMMENDATION] The only Make-side change needed: add a `Due Date` set operation to both Phase 1 scenarios (Form Intake Enhanced and Email Intake). This is one additional field in the existing ClickUp Create Task module — approximately 5 minutes of configuration per scenario.

Edge case: If a task is created during an AI fallback (KLASYFIKACJA_AI = "Do weryfikacji"), it should STILL get a Due Date, because these are the tasks most likely to need attention.

#### Aleksander Bryła — AI Systems & Reliability Architect

No AI involvement needed. This is purely deterministic — rule-based, no model. Correct design.

#### Karolina Wysocka — Sales & Service Process Designer

[RECOMMENDATION] The 48-hour window is right for NOWE ZAPYTANIA. Don't build complex multi-tier SLA rules. One rule: unassigned + due → flag.

Critical clarification: The reminder should NOT fire for tasks classified as Spam. Spam tasks have no urgency and flagging them wastes attention. The Due Date should only be set for non-spam leads.

Second important point: The ClickUp Automation comment should say something specific and actionable, not just "overdue." Suggested text: `⚠️ To zapytanie czeka na obsługę od 48h. Proszę o przypisanie lub oznaczenie jako nieistotne.`

**Moderator note:** No disagreement on this work item. The team converged rapidly on ClickUp-native Due Date approach. Probed for dissent: "What if ClickUp Automations are unreliable?" — Marta confirmed Automations are stable on Business plans and above. Tomasz confirmed the Make-side change is trivial. Convergence is genuine.

---

### Work Item 2: Lead Deduplication

#### Tomasz Sowa — Make Automation Architect

[RECOMMENDATION] Use a Make Data Store as the lookup table. Here is the complete mechanism:

**On every new task creation (both scenarios):**
1. After task is created in ClickUp, store a record in the Data Store: `{ email: <customer_email>, task_id: <clickup_task_id>, created_at: <timestamp> }`
2. BEFORE task creation, query the Data Store: "Is there a record with this email where created_at is within the last 7 days?"
3. If NO match → proceed normally (create new task).
4. If MATCH found → do NOT create a new task. Instead: (a) add a comment to the existing task with the new inquiry content, (b) update KOMPLETNOŚĆ if the new submission provides previously-missing fields, (c) add a tag to the existing task.

**Data Store structure:**
- Name: `EVSO_Lead_Dedup`
- Key: `email` (primary key)
- Fields: `email` (text), `task_id` (text), `created_at` (date), `phone` (text, optional — for future matching)

The Data Store auto-expires approach: Make doesn't support TTL natively, so add a scheduled cleanup scenario that runs weekly and deletes records older than 7 days.

Edge case to handle: If the dedup match triggers, and the new submission has fields that the original task was missing (e.g., original had no date, new submission has date), the automation should UPDATE the existing task's Custom Fields with the new data. This turns a duplicate into a data enrichment opportunity.

#### Aleksander Bryła — AI Systems & Reliability Architect

[RECOMMENDATION] Email matching is deterministic — no AI needed. Correct.

Phone normalization: If we ever add phone-based matching, normalize all phone numbers to E.164 format before storing and comparing. For now, email-only is the right scope — simpler, more reliable, and email is the most consistent identifier across channels.

[FACT] An important logical sequencing concern: the dedup check MUST happen BEFORE the AI classification call. If we detect a duplicate, we skip the AI call entirely (saves cost, avoids generating a classification for data that won't become its own task). The scenario flow becomes: Webhook → Normalize → **Dedup Check** → (if new) AI Call → Create Task → Store in Data Store; (if duplicate) → Update Existing Task → Skip AI.

This changes the module order in both Phase 1 scenarios. This is an important structural modification, not just an additive branch.

#### Marta Kurek — ClickUp Systems Architect

[RECOMMENDATION] When a duplicate is merged into an existing task, the following ClickUp changes should happen:

1. **Comment added** with full new inquiry content, clearly labeled: `--- DODATKOWY KONTAKT ({{source_channel}}) --- {{date}} ---` followed by the new message body.
2. **Custom Field update:** New field `LICZBA_KONTAKTÓW` (Number, default 1) — incremented on each merge.
3. **Tag:** Add tag `WIELOKROTNY KONTAKT` to the task. This is visible in list views and signals to the handlowiec that the customer reached out multiple times.
4. **KOMPLETNOŚĆ recalculation:** If the new submission fills previously-empty fields, update both the fields and the KOMPLETNOŚĆ score.

New Custom Field needed: `LICZBA_KONTAKTÓW` (Number, integer, default value 1, on NOWE ZAPYTANIA list). Purpose: tracks how many times this customer has reached out about this inquiry.

#### Karolina Wysocka — Sales & Service Process Designer

[RECOMMENDATION] Multiple contacts from the same person is a positive buying signal. The dedup system should make this VISIBLE, not silent.

When a duplicate is merged, the comment should be prominent enough that the handlowiec notices it. Incrementing LICZBA_KONTAKTÓW and adding the tag are both good — but the most impactful action is: if the task has no Assignee and LICZBA_KONTAKTÓW goes above 1, set Priority to Urgent. This person is actively trying to reach us — they deserve fast attention.

**Moderator — surfacing disagreement:**

**Disagreement: Time window for dedup matching**

- **Tomasz**: 7 days. Practical, Data Store stays small, covers the typical window.
- **Karolina**: 14 days would be safer. Some customers wait a week before trying the other channel.
- **Aleksander**: 7 days. Longer windows increase false positive risk — same person, genuinely new inquiry for a different date.

The moderator asked Karolina: "What happens if a 14-day window catches a genuinely new inquiry?" She acknowledged: "The merged comment would be confusing. The handlowiec would see an older inquiry they already handled, with a new one merged into it. That's worse than a missed dedup."

**Resolution:** 7 days, stored as a configurable scenario variable so the team can adjust after observing real data.

**Disagreement: Scenario modification approach**

- **Aleksander**: Dedup check must go BEFORE the AI call to save cost and avoid redundant classification.
- **Tomasz**: This means restructuring the module order in both Phase 1 scenarios, not just adding a branch. That's riskier than a clean additive change.

The moderator asked Tomasz: "What's the actual risk of reordering modules?" Tomasz: "If I move the dedup check before the AI call, and the Data Store lookup fails, it could block the entire scenario. The AI call failing had its own error handler — now I need error handling on the Data Store lookup too."

**Resolution:** Dedup check goes before AI call (Aleksander's argument on cost and logic is correct), BUT the Data Store lookup gets its own error handler: on lookup failure, assume "no match" and proceed normally. This way, Data Store downtime degrades to "no dedup" rather than "no intake."

---

### Work Item 3: Draft Reply Generation

#### Karolina Wysocka — Sales & Service Process Designer

[RECOMMENDATION] This is the feature the sales team will feel most. Two distinct use cases with different reply templates:

**Use case A — Incomplete lead reply:** The customer submitted an inquiry but key fields are missing (date, city, number of people). The draft should warmly acknowledge their interest, confirm what we understood, and ask specifically for what's missing. This is the HIGHEST-VALUE use case because it's the most repetitive — handlowcy write variations of "thanks for writing, could you tell us the date, city, and group size?" dozens of times per week. The template is formulaic enough that AI can handle it well.

**Use case B — Hot lead acknowledgment:** The customer submitted a complete inquiry. The draft should acknowledge receipt, confirm the key details back to them, express enthusiasm, and state next steps (e.g., "we'll prepare an offer" or "someone will call you"). This is valuable but more sensitive — the tone must match EVSO's brand.

[CRITICAL] I want to flag something none of us can resolve in this meeting: **we don't have example replies from EVSO.** Without real examples of how the team currently writes to customers, the AI prompt will produce generic text. The engineer MUST collect 5-10 real reply examples from the sales team before writing the system prompts. This is a hard prerequisite.

[RECOMMENDATION] Deploy in stages:
- Stage 1: Incomplete lead drafts only (weeks 1-2)
- Stage 2: After team validates tone and gives feedback, add hot lead drafts (week 3)

This gives the team a safe on-ramp to AI-generated content.

#### Aleksander Bryła — AI Systems & Reliability Architect

[RECOMMENDATION] Technical design for the draft reply system:

**Model:** Same `claude-haiku-4-5-20251001` — the task is simple text generation, Haiku is sufficient.

**When to generate:** At the end of the intake Make scenario, AFTER task creation. Add a router branch that fires for KLASYFIKACJA_AI = "Gorący lead" or "Niekompletny lead" only. Never generate drafts for Spam or Zapytanie ogólne.

**Where to store the draft:** As a ClickUp comment on the task, prefixed with a clear marker so the handlowiec knows it's AI-generated and a draft. NOT in the task description (that's the original inquiry) and NOT as a Custom Field (too short, not editable in-place).

**AI boundaries — strict:**
1. Draft MUST NOT contain any pricing, discounts, or financial commitments.
2. Draft MUST NOT promise specific dates, availability, or capacity.
3. Draft MUST NOT make commitments on behalf of EVSO (e.g., "we will definitely accommodate").
4. Draft MUST be in Polish, matching EVSO's semi-casual professional tone.
5. Draft MUST be clearly labeled as AI-generated.

**Confidence gating:** If AI_CONFIDENCE from classification was "Niska", do NOT generate a draft. The classification itself is uncertain — a draft based on a bad classification would be misleading.

**Two separate system prompts are needed:**

For incomplete leads — the prompt receives: which fields are missing, what fields are present, the original inquiry text, the customer's name. It generates a warm follow-up asking for the specific missing details.

For hot leads — the prompt receives: all extracted fields, the original inquiry text, the customer's name, the service type. It generates a confirmation/acknowledgment that reflects back what we understood.

Both prompts need the EVSO brand voice calibrated from real example replies. Without that, I would flag the output as "generic — verify before deploying." [Uncertainty U-003 applies here.]

**Failure fallback:** If the draft generation API call fails, no comment is posted. The task exists without a draft — the handlowiec writes manually, as they do today. This is a graceful degradation, not a critical failure.

#### Tomasz Sowa — Make Automation Architect

[RECOMMENDATION] Implementation in Make:

The draft generation is an additive branch to the existing Phase 1 scenarios. After the ClickUp Create Task module, add:

1. **Router** — checks KLASYFIKACJA_AI value AND AI_CONFIDENCE
2. **Branch A (Incomplete lead):** HTTP call to Anthropic API with incomplete-lead prompt → Parse response → ClickUp: Create Task Comment
3. **Branch B (Hot lead):** HTTP call to Anthropic API with hot-lead prompt → Parse response → ClickUp: Create Task Comment
4. **Branch C (all others):** No action — scenario ends

The comment is created using the ClickUp module `Create a Comment` on the task just created. The comment body includes the draft text prefixed with the AI marker.

Important: The draft generation branch must have its own error handler separate from the main scenario. If draft generation fails, the task has already been created successfully — we just don't get a draft. The error handler should: log the failure, skip the comment, and let the scenario complete normally.

For the dedup path: If a duplicate was detected and we're merging into an existing task (no new task creation), do NOT generate a draft. The original task may already have a draft or be in active handling.

#### Marta Kurek — ClickUp Systems Architect

[RECOMMENDATION] The draft comment format should be:

```
🤖 WERSJA ROBOCZA ODPOWIEDZI (wygenerowana przez AI — do edycji przed wysłaniem)
---
[draft text here]
---
⚠️ Przejrzyj i dostosuj przed wysłaniem. Nie zawiera cen ani zobowiązań.
```

This is a comment, not an email draft. The handlowiec reads it, copies the useful parts into the email composer, edits as needed, and sends. This preserves full human control.

[FACT] ClickUp's email composer in the task supports copy-paste. The workflow is: open task → read AI draft in comments → open email composer → paste/edit → send. This is natural for the team because they already use the comment area and email composer in the same task view.

No new Custom Fields needed for this feature. The draft lives in the comment stream.

**Moderator — surfacing disagreement:**

**Disagreement: Stage deployment vs. both at once**

- **Karolina**: Deploy incomplete-lead drafts first (Stage 1), hot-lead drafts later (Stage 2). Reason: tone risk. If hot-lead drafts sound wrong, the team loses trust in the whole system.
- **Tomasz**: The technical cost of adding both at once is near-zero — it's two branches of the same router. Delaying Stage 2 means re-deploying later, which is unnecessary engineering overhead.
- **Aleksander**: Both prompts need brand voice calibration. If we don't have example replies, BOTH will be generic. But the incomplete-lead prompt is more formulaic ("could you tell us X, Y, Z?") and less tone-sensitive. So Karolina's staging makes sense from a risk perspective.

The moderator asked Tomasz: "Is there a technical cost to building both branches now but only enabling one?" Tomasz: "I could build both branches and disable the hot-lead branch with a scenario variable toggle. Zero re-deploy cost later, and the code is already tested."

**Resolution:** Build both branches in the scenario. Deploy with hot-lead branch DISABLED via scenario variable. Enable it after 1-2 weeks of team feedback on incomplete-lead drafts. This satisfies Karolina's risk concern AND Tomasz's efficiency concern.

**Disagreement: Prerequisite — real reply examples**

- **Karolina**: Hard prerequisite. Without example replies, don't deploy.
- **Tomasz**: This blocks the entire feature on a non-technical dependency (collecting examples from the sales team).
- **Aleksander**: Karolina is right. The prompt without calibration will produce generic corporate text. But we can write a "calibratable" prompt with a `{{brand_voice_examples}}` variable that starts empty and gets filled once examples are collected. The initial prompt works — just produces blander output.

**Resolution:** Collecting 5-10 real reply examples is a Day 1 task in the implementation sequence. The system prompt includes a brand voice section. If examples aren't available by deployment day, deploy with the generic version and update the prompt when examples arrive. Mark as U-003 in the uncertainty register.

---

### Work Item 4: ZLECENIA Workflow Automation

#### Marta Kurek — ClickUp Systems Architect

[CRITICAL UNCERTAINTY] We do not know the current CRM→ZLECENIA handoff process. The screenshots confirm ZLECENIA tasks exist, but not how they get there.

**Two possible current-state scenarios:**

**Scenario A — Task move:** The task is moved from a seller pipeline list (e.g., SPRZEDAŻ KUBA) to the ZLECENIA list. The task keeps its ID, history, and fields. Some fields may need to be added or updated post-move.

**Scenario B — New task creation:** A new task is created in ZLECENIA with relevant data copied from the CRM task. The CRM task stays in the seller pipeline (perhaps moved to a "WYGRANA" or archive status). The ZLECENIA task is a fresh record with its own history.

[FACT] ZLECENIA task names include PACKAGE in the naming convention: `[DATE] | [CITY] | [SERVICE] | [CLIENT] | [PACKAGE]` — this is different from CRM naming which doesn't always include PACKAGE. This suggests data enrichment happens during the transition.

[FACT] ZLECENIA has its own status flow (ZLECENIA BEZ ZALICZEK → REZERWACJA JEDNOSTKI → ZLECENIA → etc.) which is unrelated to CRM statuses. This is a separate workflow domain.

[RECOMMENDATION] Before building any automation, the engineer must verify: (a) Is it a move or new task? (b) What data gets added during transition? (c) Is there a consistent trigger point (a specific status or manual action)?

[RECOMMENDATION] Regardless of which scenario is current, we need a consistent "sale won" status across all seller pipelines. Add `SPRZEDANE` as a status to every seller pipeline list. This is the trigger point for automation.

#### Tomasz Sowa — Make Automation Architect

[RECOMMENDATION] I'll design for both scenarios. The engineer implements whichever matches reality.

**Design A — Task Move approach:**

If the current process is moving tasks, the automation should be a ClickUp Automation (native):
- Trigger: When task status changes to `SPRZEDANE` in any CRM seller list
- Action: Move task to ZLECENIA list + Change status to `ZLECENIA BEZ ZALICZEK` (first operational status)

Then a Make scenario handles the post-move enrichment:
- Trigger: ClickUp webhook — task updated (status change to ZLECENIA BEZ ZALICZEK in ZLECENIA list)
- Module 1: Get task details
- Module 2: Update task name to add PACKAGE: `{{date}} | {{city}} | {{service}} | {{client}} | {{package}}`
- Module 3: Set any ZLECENIA-specific fields
- Module 4: Post comment: "Task przeniesiony z CRM. Zlecenie utworzone automatycznie."

**Design B — New Task Creation approach:**

A Make scenario creates the ZLECENIA task:
- Trigger: ClickUp webhook — task status changed to `SPRZEDANE` in any CRM seller list
- Module 1: Get full CRM task details (all Custom Fields)
- Module 2: Create new task in ZLECENIA list with mapped fields and extended naming convention
- Module 3: Set CRM task status to `ARCHIWUM` or similar end-state
- Module 4: Add comment on ZLECENIA task linking back to original CRM task
- Module 5: Add comment on CRM task linking to new ZLECENIA task

Design B is more complex but cleaner architecturally — CRM and ZLECENIA remain fully separate domains.

[INFERENCE] Based on the data visible in screenshots (ZLECENIA tasks have different field emphasis — OBSŁUGA, PAKIET are prominent; there's no visible LEAD-#### on ZLECENIA tasks), Design B (new task creation) is more likely the current pattern. But this is inference, not confirmed.

#### Aleksander Bryła — AI Systems & Reliability Architect

No AI involvement. The CRM→ZLECENIA transition is deterministic. The data is already structured — there's nothing to classify or extract.

One minor observation: if the CRM task has an AI_PODSUMOWANIE, it could be useful to carry it forward to the ZLECENIA task as operational context. But this is a nice-to-have, not a requirement.

#### Karolina Wysocka — Sales & Service Process Designer

[RECOMMENDATION] The trigger must be a HUMAN action (changing status to SPRZEDANE), not an automatic evaluation. The decision "this lead is won" is a business judgment that must remain with the handlowiec.

Field mapping from CRM to ZLECENIA — minimum viable set:

| CRM Field | ZLECENIA Mapping | Notes |
|-----------|-----------------|-------|
| Task name | Extended: add PAKIET | Naming convention requires PACKAGE |
| MIASTO | MIASTO | Direct copy |
| USŁUGA | USŁUGA | Direct copy |
| PAKIET | PAKIET | Direct copy |
| DATA EVENTU | Due date / DATA EVENTU | This becomes the operational deadline |
| EMAIL | EMAIL | For operational contact |
| TELEFON | TELEFON | For operational contact |
| ILOŚĆ OSÓB | ILOŚĆ OSÓB | Needed for capacity planning |
| GODZINA EVENTU | GODZINA EVENTU | Operational timing |
| Description | Description | Preserve original inquiry + any conversation |

[FACT] ZLECENIA has visible fields not present in CRM intake: OBSŁUGA (e.g., "BEZOBSŁUGOWY"). This field must be set manually during or after the transition — it's an operational decision, not captured at intake.

"The automation creates the ZLECENIA task with all transferable data. The operations team then fills in OBSŁUGA, JEDNOSTKA, and any other delivery-specific fields. That's the correct division of labor."

**Moderator — surfacing disagreement:**

**Disagreement: Design A (move) vs. Design B (new task)**

- **Marta**: Both are valid. Design B is cleaner architecturally but creates data duplication. Design A preserves the full history in one place.
- **Tomasz**: Design B is more robust because CRM and ZLECENIA can evolve independently. With Design A, moving a task to a list with different Custom Field definitions can cause field visibility issues.
- **Karolina**: The sales team needs to see their won deals in the CRM pipeline even after the ZLECENIA task exists — for commission tracking, for showing progress. If we move the task out, it disappears from their view.

The moderator asked: "Karolina, does that mean Design B is required to keep a CRM record?" Karolina: "Yes. The CRM task should stay in the seller pipeline, marked as SPRZEDANE. A new ZLECENIA task is created. Both exist."

**Resolution:** Design B (new task creation) is the recommended approach. The CRM task stays in the seller pipeline with status SPRZEDANE. A new task is created in ZLECENIA with mapped fields. Cross-links are added as comments on both tasks. HOWEVER: the engineer must verify this matches the current manual process before implementing.

**Disagreement: Invest 4 days into ZLECENIA automation vs. reduce scope**

- **Marta**: ZLECENIA automation is the highest-risk item because of U-001. It's also in a separate operational domain. If total time is constrained, this is the one to cut or reduce.
- **Tomasz**: Agreed, but the engineer can spend Day 1 investigating, and if the process is clear, the implementation is straightforward. If investigation reveals unexpected complexity, we can defer parts.
- **Karolina**: This saves the most manual time per event. Every won deal requires 5-10 minutes of manual task creation in ZLECENIA. At maybe 20-30 events per month, that's 2-5 hours of repetitive work.

**Resolution:** Include ZLECENIA automation but with an explicit "investigation day" (Day 1 of the work item). If investigation reveals the process is more complex than expected, the engineer has authority to reduce scope to: (a) add SPRZEDANE status only, and (b) document the process for Phase 3 implementation.

---

## Disagreements

### Disagreement 1: Dedup Time Window

- **Parties**: Tomasz + Aleksander vs. Karolina
- **Type**: Predictive (what customer behavior to expect)
- **Position A (7 days)**: Shorter window reduces false positives. Keeps Data Store small. A 14-day window could merge genuinely different inquiries from the same person.
- **Position B (14 days)**: Some customers wait a week before trying another channel. Missing these duplicates defeats the purpose.
- **What would change minds**: Real data on the gap between first and second contact for the same customer. After 1 month of operation, analyze the Data Store to see how many dedup matches occur and at what age.
- **Resolution**: 7 days, implemented as a configurable variable. Revisit after 30 days of operation with real data.

### Disagreement 2: Draft Reply Staging

- **Parties**: Karolina vs. Tomasz
- **Type**: Values (risk tolerance for customer-facing content)
- **Position A (staged)**: Deploy incomplete-lead drafts first, hot-lead drafts later. Tone risk is real and damaging.
- **Position B (both at once)**: Technical cost is zero; delaying is unnecessary overhead.
- **What would change minds**: If the brand voice examples are collected on Day 1 and both prompts are validated by the sales team before deployment, simultaneous deployment becomes safe.
- **Resolution**: Build both, deploy staged. Hot-lead branch disabled via variable toggle. Enable after team feedback.

### Disagreement 3: ZLECENIA Design A vs. B

- **Parties**: (No strong opposition — more of a clarification need)
- **Type**: Factual (depends on what the current process actually is)
- **Position A (task move)**: Simpler, preserves history.
- **Position B (new task)**: Cleaner separation, CRM record preserved for sales team.
- **What would change minds**: Knowing the current manual process.
- **Resolution**: Design B recommended. Engineer verifies before implementing. If current process is actually task-move and team prefers it, use Design A.

---

## Convergence

### Synthesis

The team converged on four features with clear technical designs:

1. **Follow-up reminders** — Lightest touch. Due Date set by Make at task creation; ClickUp native Automation flags overdue unassigned tasks. No new scenarios needed. ~1.5 days.

2. **Lead deduplication** — Moderate complexity. Make Data Store lookup before task creation in both existing scenarios. On match: merge as comment + update fields + tag. 7-day window, configurable. Requires reordering modules in Phase 1 scenarios (dedup before AI). ~3 days.

3. **Draft reply generation** — Highest user impact. Router branch added to existing scenarios after task creation. Two AI prompts (incomplete-lead, hot-lead). Stored as ClickUp comments. Staged deployment: incomplete first, hot later. Hard prerequisite: collect real reply examples. ~3.5 days.

4. **ZLECENIA automation** — Highest uncertainty. New SPRZEDANE status added to all seller pipelines. New Make scenario creates task in ZLECENIA from CRM data. Design B (new task). Requires investigation day. ~4 days (including investigation).

**Total: ~12 days.** Within the 15-day constraint with 3 days of buffer.

The key trade-off accepted by the entire team: Phase 2 modifies the existing Phase 1 scenarios (dedup check reordering, draft generation branch, Due Date field) rather than creating entirely separate scenarios. This is more efficient but carries the risk of introducing regressions in the stable intake flow. Mitigation: test each modification against the Phase 1 smoke test checklist before deploying.

### Decision Records

| Field | Content |
|-------|---------|
| **ID** | D-001 |
| **Decision** | Follow-up reminders use ClickUp native Due Date + Automation, not Make scenarios |
| **Rationale** | ClickUp handles time-based triggers natively and reliably. Building in Make would add fragility (polling), cost (operations), and latency. Due Date also provides visual deadline in Calendar/List views. |
| **Dissent** | None |
| **Confidence** | High — all four experts agree, ClickUp docs confirm capability |
| **Revisit if** | ClickUp plan doesn't support the needed Automation trigger type; or if external notifications (Slack, SMS) are needed — those would require Make |

| Field | Content |
|-------|---------|
| **ID** | D-002 |
| **Decision** | Lead deduplication uses Make Data Store with email-only matching, 7-day configurable window |
| **Rationale** | Email is the most reliable cross-channel identifier. Phone matching adds normalization complexity. 7 days covers the typical inquiry-to-follow-up window without creating false positive risk. Data Store is simpler than ClickUp API search (which would require paginated queries across variable-length lists). |
| **Dissent** | Karolina preferred 14 days. Acknowledged 7-day risk of missing late follow-ups but accepted that false positives are worse. |
| **Confidence** | Medium — the 7-day window is an educated guess. Real data will confirm or require adjustment. |
| **Revisit if** | After 30 days, if analysis shows significant duplicate misses beyond the 7-day window |

| Field | Content |
|-------|---------|
| **ID** | D-003 |
| **Decision** | Dedup check is positioned BEFORE the AI classification call in the scenario module order |
| **Rationale** | Saves API cost on duplicates. Logically correct: a duplicate doesn't need independent classification. Error handling on Data Store lookup defaults to "no match" to prevent dedup failures from blocking intake. |
| **Dissent** | Tomasz noted the risk of scenario restructuring but accepted with the error-handler mitigation. |
| **Confidence** | High — logic is sound, error handling covers the risk |
| **Revisit if** | Data Store has reliability issues in production |

| Field | Content |
|-------|---------|
| **ID** | D-004 |
| **Decision** | Draft replies are stored as ClickUp task comments with AI-generated marker, not in Custom Fields or description |
| **Rationale** | Comments are visible in the Activity panel, editable in context, and don't interfere with the task's structured data. The description must remain the original inquiry archive. Custom Fields are too short for reply text. |
| **Dissent** | None |
| **Confidence** | High |
| **Revisit if** | ClickUp introduces a native "draft email" API that could be used instead |

| Field | Content |
|-------|---------|
| **ID** | D-005 |
| **Decision** | Draft reply deployment is staged: incomplete-lead drafts first, hot-lead drafts enabled after 1-2 weeks of feedback |
| **Rationale** | Incomplete-lead replies are more formulaic and lower-risk. Staged deployment builds team trust before exposing them to the more tone-sensitive hot-lead drafts. Technical cost of staging is zero (variable toggle). |
| **Dissent** | Tomasz: no technical reason to stage. Accepted the process risk argument. |
| **Confidence** | High — this is a process management decision, not a technical one |
| **Revisit if** | Brand voice examples are collected early AND team explicitly says "turn on both" |

| Field | Content |
|-------|---------|
| **ID** | D-006 |
| **Decision** | ZLECENIA automation uses Design B (new task creation), with SPRZEDANE status as trigger |
| **Rationale** | CRM task must remain in seller pipeline for sales team visibility. ZLECENIA is a separate operational domain with different data needs. New task creation provides clean separation. Cross-links preserve traceability. |
| **Dissent** | Marta noted Design A (move) is simpler but accepted Design B on Karolina's workflow argument. |
| **Confidence** | Medium — depends on U-001 (current process verification). Engineer has authority to switch to Design A if investigation reveals it's the established pattern. |
| **Revisit if** | Investigation shows current process is task-move AND sales team prefers losing the CRM record |

| Field | Content |
|-------|---------|
| **ID** | D-007 |
| **Decision** | ZLECENIA work item includes an explicit investigation day; scope can be reduced if process is more complex than expected |
| **Rationale** | U-001 is unresolvable in this meeting. Allocating investigation time prevents building on incorrect assumptions. Scope reduction authority prevents the ZLECENIA work item from consuming disproportionate time. |
| **Dissent** | None |
| **Confidence** | High — this is a risk management decision |
| **Revisit if** | Investigation reveals a trivially simple process (then skip the buffer time) |

### Trade-Off Map

- **Choosing to modify Phase 1 scenarios (dedup + draft branches) vs. creating new scenarios**: Gains efficiency and code reuse. Costs regression risk on stable intake flow. Favored by Tomasz (less duplication) and Aleksander (dedup must precede AI call). Mitigated by re-running Phase 1 smoke tests.

- **Choosing 7-day dedup window vs. 14-day**: Gains precision (fewer false positives). Costs recall (may miss some late duplicates). Favored by Tomasz + Aleksander. Karolina accepted with configurable-variable mitigation.

- **Choosing staged draft deployment vs. simultaneous**: Gains trust and safety with the sales team. Costs 1-2 weeks delay on hot-lead drafts. Favored by Karolina + Aleksander. Tomasz accepted with toggle mechanism.

- **Choosing Design B (new ZLECENIA task) vs. Design A (move)**: Gains clean domain separation and CRM record preservation. Costs additional complexity (two tasks, cross-links, potential data drift). Favored by Karolina (sales visibility) and Tomasz (architectural cleanliness).

---

## Uncertainty Register

| ID | Unknown | Impact if Wrong | Resolvable? | Proposed Action | Owner |
|----|---------|----------------|-------------|-----------------|-------|
| U-001 | Current CRM→ZLECENIA handoff process (move vs. new task) | Wrong automation design; wasted engineering time | Yes — ask the team or observe their workflow | Engineer investigates on ZLECENIA Day 1 | Tomasz Sowa |
| U-002 | Whether all seller pipelines share the same statuses | SPRZEDANE status might need different names or positions in different lists | Yes — check each list in ClickUp | Engineer checks all 4 seller lists during ZLECENIA setup | Marta Kurek |
| U-003 | EVSO's brand voice for customer replies | Draft replies may sound generic or off-brand | Partially — collecting examples helps but calibration is iterative | Collect 5-10 real reply examples from sales team before prompt writing | Karolina Wysocka |
| U-004 | ClickUp plan tier (Business vs. lower) for Automation capabilities | Time-based Automation triggers may not be available | Yes — check ClickUp subscription | Engineer verifies on Day 1 of implementation | Marta Kurek |
| U-005 | Custom Field inheritance across CRM folder lists | Moving/copying tasks between lists may lose Custom Field values | Yes — test with one task | Engineer tests during ZLECENIA investigation day | Marta Kurek |
| U-006 | Make Data Store performance at EVSO's volume | At ~100 leads/week and 7-day retention, the Data Store holds ~100 records max. Should be fine, but untested. | Yes — monitor after deployment | Monitor Data Store size and lookup latency for first 2 weeks | Tomasz Sowa |
| U-007 | Whether WPForms webhook payload includes enough info for dedup check BEFORE AI call | If email isn't available at the dedup step, the check can't happen | Yes — test webhook payload structure | Verify during scenario modification | Tomasz Sowa |

---

## Open Questions

1. **How does the sales team currently create ZLECENIA tasks?** — Matters because it determines the automation design. Best answered by: observing or asking the team. Current assumption: Design B (new task creation).

2. **Does EVSO have brand voice guidelines or example reply templates?** — Matters because draft reply quality depends on tone calibration. Best answered by: Karolina collects from the sales team. Current assumption: generic professional Polish tone, refined after example collection.

3. **Are all seller pipelines on the same ClickUp list template?** — Matters because SPRZEDANE status and automation triggers must work across all lists. Best answered by: engineer checks each list. Current assumption: similar structure, may need per-list configuration.

4. **What ClickUp plan is EVSO on?** — Matters for Automation feature availability. Best answered by: check account settings. Current assumption: Business plan or higher.

---

## Next Work Package

| Priority | Action | Rationale | Depends On |
|----------|--------|-----------|------------|
| High | Collect 5-10 real customer reply examples from sales team | Hard prerequisite for draft reply prompt calibration (U-003) | Sales team availability |
| High | Verify current CRM→ZLECENIA handoff process (U-001) | Determines ZLECENIA automation design | Access to observe or interview team |
| High | Verify ClickUp plan tier for Automation capabilities (U-004) | Confirms follow-up reminder approach | ClickUp admin access |
| High | Write Phase 2 Implementation Guide based on this meeting's decisions | Produces the actionable build document | This meeting's output |
| Medium | Check all seller pipeline lists for status consistency (U-002) | Needed before adding SPRZEDANE status | ClickUp access |
| Medium | Test Custom Field inheritance when moving tasks between CRM lists (U-005) | Informs ZLECENIA field mapping | ClickUp access |

---

## Artifacts to Update

| Artifact | Action | Specific Change | Triggered By |
|----------|--------|----------------|-------------|
| Phase 1 Make Scenario: Form Intake Enhanced | Update | Add Due Date field set; add Data Store write; reorder modules for dedup check before AI; add router branch for draft reply generation | D-001, D-002, D-003, D-004 |
| Phase 1 Make Scenario: Email Intake | Update | Same changes as Form Intake Enhanced | D-001, D-002, D-003, D-004 |
| ClickUp: NOWE ZAPYTANIA Custom Fields | Update | Add field: LICZBA_KONTAKTÓW (Number, default 1) | D-002 |
| ClickUp: NOWE ZAPYTANIA Automations | Create | New Automation: "When Due Date arrives + no Assignee → Priority = Urgent + comment" | D-001 |
| ClickUp: All seller pipeline lists | Update | Add status: SPRZEDANE | D-006 |
| Make: New Data Store | Create | EVSO_Lead_Dedup with email/task_id/created_at/phone fields | D-002 |
| Make: New Scenario | Create | EVSO - ZLECENIA Task Creator (triggered by SPRZEDANE status) | D-006 |
| Make: New Scenario | Create | EVSO - Dedup Cleanup (weekly, deletes records > 7 days) | D-002 |
| WIKI PRACOWNIKA | Update | Add documentation for: dedup behavior, draft reply feature, follow-up reminder logic, ZLECENIA auto-creation | All decisions |
| Phase 1 Smoke Tests | Update | Re-run full Phase 1 smoke test suite after scenario modifications to confirm no regressions | D-003 (module reordering) |

---

## Meeting Metadata

- **Format used**: Structured Workshop
- **Experts activated**: 4 of 4
- **Decisions made**: 7
- **Uncertainties logged**: 7
- **Open questions**: 4
- **Estimated confidence in primary recommendation**: Medium-High (high for features 1-3, medium for ZLECENIA due to U-001)
