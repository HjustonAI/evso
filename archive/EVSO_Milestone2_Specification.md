# EVSO — Milestone 2: ClickUp Agentic Email System

**Version:** 1.0  
**Date:** 2026-04-30  
**Status:** Specification — ready for implementation planning  
**Depends on:** Milestone 1 (Form Intake AI) — ✅ COMPLETED

---

## What Milestone 1 delivered (baseline)

- Forminator form submission → webhook → Make → Claude Haiku (classify + extract from free text) → ClickUp task
- Every new lead lands in `NOWE ZAPYTANIA` as a task with: structured fields, AI classification, AI summary, completeness score
- Task naming convention: `[DATA] | [MIASTO] | [USŁUGA] | [IMIĘ]`
- Custom field `EMAIL` populated on every task — this is the foundation for M2

---

## What Milestone 2 must achieve

**Goal in one sentence:** Every email exchange between EVSO employees and a client must live inside exactly one ClickUp task — regardless of which email thread, reply subject, or new thread the client uses.

### Concrete outcome for EVSO employees:
1. Employee reads the incoming lead in ClickUp, writes a reply → email is sent to client, stays logged in the task
2. Client replies (via any mechanism) → reply appears in the same ClickUp task
3. If client starts a brand new email months later → system finds the existing task and routes the message there
4. Employee never needs to leave ClickUp to manage the conversation

---

## The Email Threading Problem — all scenarios

These are every possible scenario that must be covered. No gaps allowed.

| # | Scenario | Who sends | Via | Difficulty |
|---|----------|-----------|-----|-----------|
| S1 | New lead from form | Client | Forminator → Make webhook | ✅ Done (M1) |
| S2 | New lead from email (cold) | Client | Email directly | Planned in M1, needs M2 to complete |
| S3 | Employee sends offer/reply | EVSO employee | ClickUp email composer | Native ClickUp — needs config |
| S4 | Client replies IN THREAD | Client | Email (Reply to EVSO email) | Native ClickUp (Reply-To) |
| S5 | Client replies with changed subject | Client | Email | Make watcher + email match |
| S6 | Client starts brand new thread | Client | Email | Make watcher + email match |
| S7 | Client replies from different email address | Client | Email | Partially: match by old email + phone |
| S8 | Multiple leads from same client email | Client | Form/email | Disambiguation logic |

---

## Architecture Overview

The system has three layers working together:

```
┌─────────────────────────────────────────────────────────┐
│  LAYER 1: ClickUp Native Email (handles S3, S4)          │
│  Each task has a unique @mg.clickup.com address          │
│  Employee sends FROM task → Reply-To = task address      │
│  Client replies → auto-threads to task activity         │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│  LAYER 2: Make.com Email Watcher (handles S2, S5, S6)    │
│  Gmail Watch → match sender to EMAIL field in ClickUp    │
│  If match: add email as comment to existing task         │
│  If no match: route to NOWE ZAPYTANIA (new lead flow)   │
└──────────────────────────┬──────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────┐
│  LAYER 3: ClickUp AI (enhances S3 — employee replies)    │
│  Draft reply suggestions based on conversation history   │
│  Summarize thread on demand                              │
│  Classify follow-up intent (ready to book / still cold) │
└─────────────────────────────────────────────────────────┘
```

---

## Component Breakdown

### Component A: ClickUp Email Configuration (Layer 1)

**What it is:** ClickUp Business allows each list and each task to receive email at a dedicated address. When EVSO sends email FROM a task (using the existing email composer shown in screenshots), ClickUp automatically sets Reply-To to the task's unique address. Client replies go directly to the task's activity feed.

**What needs to be done:**

1. **Connect both EVSO Gmail accounts to ClickUp Email**
   - Settings → Integrations → Email → Connect Gmail
   - Connect both addresses currently attached
   - Recommended: `kontakt@partyboat.fun` and/or `kontakt@partytram.fun`

2. **Set up NOWE ZAPYTANIA list email address**
   - Each list in ClickUp Business has an inbound email: `listname@mg.clickup.com`
   - Emails sent here create new tasks automatically
   - This handles cold email leads (S2) without Make
   - Recommended: find the list email address in ClickUp list settings → document it → forward rules in Gmail can send unknown-sender emails here

3. **Employee workflow (S3)**
   - Open the lead task → go to Email tab (already visible in screenshots)
   - Write and send reply from within the task
   - ClickUp logs the outbound email in activity
   - Reply-To header is automatically set to task's unique address

4. **Client replies in thread (S4)**
   - Client clicks Reply on the EVSO email
   - Email routes to the task's unique address
   - Appears in task activity automatically
   - No Make needed for this scenario

**Constraint:** This ONLY works if employee consistently sends FROM within the ClickUp task. If employee replies directly from Gmail, the loop breaks. Training required.

---

### Component B: Make Scenario — Inbound Email Watcher (Layer 2)

This handles: S2 (cold email leads), S5 (changed subject replies), S6 (brand new threads from known clients).

**Trigger:** Gmail Watch — watches the two connected inboxes for new emails

**Logic flow:**

```
New email arrives in Gmail
    │
    ├─ Is this a reply to a ClickUp task email? (@mg.clickup.com in To/CC)
    │       └─ YES → ClickUp already handled it (Layer 1). SKIP.
    │
    ├─ Is sender email in ClickUp EMAIL field of any existing task?
    │       └─ Search ClickUp tasks by EMAIL custom field = sender email
    │       │
    │       ├─ YES (one match) → Add email as comment to that task
    │       │       [Comment format: "📧 Nowa wiadomość od klienta:\n\n{email_body}"]
    │       │       [Also: update task activity / add note with subject line]
    │       │
    │       ├─ YES (multiple matches) → Route to most recently created active task
    │       │       [Active = status not in: PRZEGRANY, ANULOWANY, ZREALIZOWANE]
    │       │
    │       └─ NO (unknown sender) → Create new lead task (M1 email intake flow)
    │               [Classify with Claude Haiku]
    │               [Create task in NOWE ZAPYTANIA]
```

**Modules needed:**

1. `google-email:TriggerNewEmail` — watch both Gmail accounts
2. `util:SetVariables` — extract sender email, subject, body, timestamp
3. Filter: skip if To/Reply-To contains `mg.clickup.com` (already handled by Layer 1)
4. `clickup:searchTasks` — search by EMAIL custom field value (filter: `EMAIL = sender_email`)
5. Router:
   - Branch A (found, single task): `clickup:createComment` on the matched task
   - Branch B (found, multiple tasks): sort by date_created DESC, take first, create comment
   - Branch C (not found): route to existing M1 email intake flow (classify + create task)

**Note on ClickUp search by custom field:** ClickUp API supports filtering tasks by custom field values. The query would filter on `EMAIL` field UUID = sender's email. This is possible via ClickUp's task filtering API.

**Gmail filter recommendation:**
- Create a Gmail filter that labels all incoming emails that are NOT automated/notifications with label "EVSO-CRM-INBOUND"
- Make watches only this label, reducing noise

---

### Component C: Make Scenario — Outbound Email Logger (optional, Layer 2+)

If employees sometimes send emails directly from Gmail (breaking the ClickUp loop), this safety net helps.

**Trigger:** Gmail Watch — watches SENT folder

**Logic:**
- When EVSO sends an email to a client email that matches a ClickUp task EMAIL field
- Log the sent email as a comment to that task
- This keeps the task timeline complete even if employee used Gmail directly

**Verdict:** Nice to have, not critical if employee workflow discipline is enforced. Implement in Phase 2 of M2 if needed.

---

### Component D: ClickUp AI Integration (Layer 3)

ClickUp Business with AI enabled. No external setup needed — this is native UI capability.

**Use cases for EVSO employees:**

**D1: Draft reply**
Employee opens task with client conversation history visible. Types prompt to ClickUp Brain:
> "Zaproponuj odpowiedź na tę wiadomość w oparciu o historię konwersacji i naszą ofertę. Klient pyta o BOAT w Krakowie, 60 osób, impreza firmowa."

ClickUp AI drafts a response. Employee edits and sends from the email composer.

**D2: Summarize conversation**
> "Podsumuj przebieg tej konwersacji — co klient chce, co mu wysłaliśmy, na czym skończyliśmy."

Useful when taking over a lead from another salesperson.

**D3: Classify follow-up intent**
> "Na podstawie tej konwersacji, czy klient jest gotowy do zakupu? Zaproponuj następny krok."

**D4: Generate offer text**
> "Wygeneruj tekst oferty dla tego klienta na podstawie danych z formularza i jego opisu."

**Requirements:**
- ClickUp AI enabled on workspace (already confirmed — Business plan)
- No additional configuration needed
- Employees need brief training on effective prompting

---

## Edge Case Handling

### S7: Client replies from different email address

**Problem:** Client initially sent inquiry from `klaudia@work.com`, but replies from `klaudia.personal@gmail.com`

**Detection:** No match found for `klaudia.personal@gmail.com` in EMAIL field

**Mitigation options (in priority order):**

1. **Phone number fallback (recommended):** If email doesn't match, check if phone number appears in email signature/body using regex. Search by TELEFON field. High precision for Polish business context.

2. **Name match:** Extract sender name from email headers, fuzzy-match against task names. Low precision, use only to suggest a match to employee for manual confirmation.

3. **Create new task:** If no match found, create new task and flag it as "Possible duplicate — verify EMAIL field". Employee manually merges if needed.

4. **Proactive solution:** In every outbound EVSO email, include a line: "Proszę odpowiadać na tę wiadomość z adresu {client_email}, aby system automatycznie przypisał Pani/Pana odpowiedź." — Educate client to reply in thread.

### S8: Multiple leads from same client email

**Problem:** `klaudia@work.com` has two tasks — BOAT inquiry from March and TRAM inquiry from April.

**Resolution logic:**
1. Check task status — prefer ACTIVE over CLOSED statuses
2. If both active: choose the one with most recent activity date
3. If still ambiguous: create comment on BOTH tasks with note: "⚠️ Duplikat emaila — zweryfikuj ręcznie"

### Email threading — subject line variations

| What client does | How system handles it |
|-----------------|----------------------|
| Replies in same thread | ClickUp native (Layer 1) — thread intact |
| Replies but deletes "Re:" prefix | Layer 1 still handles via Message-ID header |
| Replies and changes subject completely | Layer 2 — matched by sender email |
| Starts brand new email | Layer 2 — matched by sender email |
| Forwards a 3rd party email | Layer 2 — matched by sender email (From field) |
| Sends from same domain, different address | Layer 2 — phone fallback or manual |

---

## Third-Party Tools Assessment

The question is: is it worth adding an external tool beyond ClickUp + Make?

| Tool | What it solves | Cost | Verdict |
|------|---------------|------|---------|
| **Front.app** | Shared inbox with perfect thread tracking by sender, auto-routing, team views | ~$19-29/user/month | PASS — adds cost and another platform. Make + ClickUp native is sufficient |
| **Missive** | Collaborative email with CRM integrations | ~$18/user/month | PASS — same reason |
| **Intercom** | Full customer messaging platform | $100+/month | PASS — overkill for this use case |
| **Loops.so** | Marketing automation | N/A | PASS — wrong category |
| **Zapier** | Alternative to Make | Pay-per-task | PASS — already on Make |
| **Google Workspace Group email** | Route replies to shared inbox | Included | ✅ CONSIDER — cheap solution for shared inbox with routing |

**Recommendation: no new tools.** ClickUp Business + Make.com + Gmail covers all scenarios. The architecture above is cost-effective and doesn't add vendor complexity.

**One optional low-cost enhancement:** Google Workspace group email (`crm@evso.pl`) as a shared inbox for all CRM-related incoming email, separate from individual salesperson mailboxes. This is a cleaner setup than watching multiple individual Gmail accounts.

---

## Implementation Sequence

### Phase 1 — ClickUp Email Setup (2-3 days, no code)

1. Connect both Gmail accounts to ClickUp Email (Settings → Integrations)
2. Enable email on `NOWE ZAPYTANIA` list — get list email address
3. Test: employee sends email from task → client receives → client replies → verify it threads back to task
4. Document the employee SOP: "ALWAYS reply from within the ClickUp task, not from Gmail"
5. Configure Gmail filter to label inbound CRM emails

**Success test:** Send test email from ClickUp task → reply from client email → confirm reply appears in task activity

### Phase 2 — Make Scenario: Email Watcher (3-5 days)

1. Create Gmail Watch scenario in Make (on client's Make account)
2. Build task lookup by EMAIL custom field
3. Build router: comment on existing task / create new task
4. Test all S2, S5, S6 scenarios
5. Handle S8 (multiple tasks) disambiguation

**Success test:**
- New email from form-submitted client (`klaudia@work.com`) arrives in Gmail inbox
- Make finds the existing LEAD-7861 task
- Email body appears as comment in that task within 60 seconds

### Phase 3 — ClickUp AI Onboarding (1 day)

1. Document 4-5 effective AI prompts for EVSO sales team
2. Brief training session with handlowcy: how to use ClickUp Brain for:
   - Drafting offers
   - Summarizing leads
   - Classifying follow-up intent
3. Add prompt templates to `WIKI PRACOWNIKA` in ClickUp

### Phase 4 — Edge Case Hardening (2-3 days, after Phase 2 goes live)

1. Monitor for S7 (different sender email) cases — decide on phone fallback implementation
2. Monitor for S8 (duplicate email) cases — tune disambiguation logic
3. Optionally: build Component C (outbound email logger for Gmail-sent emails)

---

## Dependencies & Prerequisites

| Item | Status | Who |
|------|--------|-----|
| ClickUp Business plan | ✅ Active | Client |
| ClickUp AI enabled | ✅ Active | Client |
| Both Gmail accounts connected to ClickUp | ❓ Verify | EVSO + Bartek |
| Make account (client's) | ✅ Active | Client |
| EMAIL custom field populated on all tasks | ✅ Done (M1) | M1 delivered |
| ClickUp task search by custom field API access | ❓ Verify Make has this | Bartek |

---

## ClickUp Task Fields — M2 Additions

No new Custom Fields are required for Milestone 2. The existing EMAIL field is the thread identifier. However, consider adding:

| Field | Type | Purpose | Priority |
|-------|------|---------|----------|
| OSTATNIA WIADOMOŚĆ | Date | Timestamp of last email activity | Medium |
| KANAL KONTAKTU | Dropdown (Email/Telefon/Form) | Track how client is communicating | Low |
| AI DRAFT | Long Text | ClickUp AI-generated draft for next reply | Nice to have |

---

## Success Criteria

Milestone 2 is complete when ALL of the following are true:

| # | Criterion | How to verify |
|---|-----------|--------------|
| M2-SC1 | Employee sends email from ClickUp task → email delivered to client | Send test email |
| M2-SC2 | Client replies in thread → reply appears in ClickUp task activity within 2 min | Manual test |
| M2-SC3 | Client starts new email thread → Make routes to correct existing task | Manual test with known-client email |
| M2-SC4 | Cold email from unknown client → new task created in NOWE ZAPYTANIA | Manual test with unknown email |
| M2-SC5 | All employee-sent emails from ClickUp are logged in task activity | Check 5 historical sends |
| M2-SC6 | ClickUp AI can draft a contextual reply when prompted | Test with sample lead |
| M2-SC7 | Zero manual Gmail-to-ClickUp copy-pasting needed for standard scenarios | Process audit after 2-week pilot |

---

## What M2 Deliberately Does NOT Include

These are deferred to later milestones or rejected:

- **WhatsApp / SMS integration** — different channel, different complexity. Not in M2 scope.
- **Auto-reply to client** — too much automation risk without human review. Employee always controls outbound.
- **Auto-assignment to salesperson** — requires process change and team buy-in. Deferred.
- **CRM analytics dashboard** — value is there but not critical for M2 operational goal.
- **Offer generation automation** — ClickUp AI handles this assistively; full automation is Phase 3.
- **ClickUp automations for status changes based on email** — nice but complex. Deferred.

---

## Risk Register

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|-----------|
| Employees bypass ClickUp and reply from Gmail | High | High | SOP documentation + bi-weekly audit |
| ClickUp task email address not publicly reachable (spam filters) | Medium | Medium | Test with actual client email domains |
| Make Gmail Watch hits rate limits (100 leads/week) | Low | Low | Gmail Watch checks every 15min, ~672 cycles/week — fine |
| Same client email on multiple tasks → wrong task updated | Medium | Medium | Disambiguation logic in Phase 4 |
| Client changes email provider, replies from new address | Low | Medium | Phone fallback in Phase 4 |
| ClickUp API search by custom field returns partial results | Low | High | Test before building Make scenario; add verification step |

---

## Cost Estimate

| Item | Monthly Cost |
|------|-------------|
| Make.com operations (Email Watcher scenario, ~100 leads × 5 ops each) | ~500 ops/month → included in existing plan |
| ClickUp Business | Already paying |
| ClickUp AI | Already included in Business |
| Gmail | Already paying |
| New tools | $0 |
| **Total additional cost** | **$0** |

---

## Appendix — Email Routing Logic (pseudocode for Make scenario)

```
ON new_email_received (Gmail Watch):
  
  sender_email = email.from.address
  subject = email.subject
  body = email.body_text
  timestamp = email.received_at

  // Skip if this is already a ClickUp-routed email
  IF email.to contains "@mg.clickup.com" OR email.reply_to contains "@mg.clickup.com":
    SKIP (handled by ClickUp Layer 1)

  // Skip automated emails
  IF email.from contains "no-reply" OR "noreply" OR "mailer-daemon" OR "postmaster":
    SKIP

  // Search for existing task
  matching_tasks = clickup_search_tasks(
    filter: EMAIL_field = sender_email,
    filter: status NOT IN ["PRZEGRANY", "ANULOWANY", "ZREALIZOWANE"],
    sort: date_created DESC
  )

  IF matching_tasks.count == 1:
    // Perfect match
    add_comment(
      task_id = matching_tasks[0].id,
      comment = "📧 Wiadomość od klienta [{timestamp}]\nTemat: {subject}\n\n{body}"
    )
  
  ELSE IF matching_tasks.count > 1:
    // Multiple active leads from same email — use most recent
    add_comment(
      task_id = matching_tasks[0].id,  // already sorted by date DESC
      comment = "📧 Wiadomość od klienta [{timestamp}]\nTemat: {subject}\n\n{body}\n\n⚠️ Uwaga: klient ma {count} aktywnych tasków — sprawdź czy to właściwy."
    )
  
  ELSE:
    // Unknown sender — create new lead
    route_to_email_intake(sender_email, subject, body)
    // This runs the existing M1 email intake flow: Claude Haiku → classify → create task
```

---

*Specification prepared: 2026-04-30. Based on: EVSO_ClickUp_Context_Snapshot.md, working M1 blueprint (WEBHOOK → AI → CLICKUP), EVSO DevTeam meeting M-001, Phase 1 Implementation Guide.*
