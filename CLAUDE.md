# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

This is **not a code repository**. It is a **documentation + design context** workspace for **EVSO**, a Polish event-organization company (party trams, party boats, party buses across Poland — `partytram.fun`, `partyboat.fun`, `busparty.fun`).

The user (Bartek) is implementing automations for EVSO. He uses this directory to:
1. Capture context about EVSO (ClickUp setup, dataflows, websites, forms, team).
2. Outsource thinking and design to AI (problem definitions, expert meetings, milestone specs, implementation guides).
3. Supervise the AI-shaped designs and implement them himself in **Make.com** + **ClickUp** + **Anthropic API (Claude Haiku)**.

There is no build, no tests, no lint. Outputs are markdown documents and occasionally a Make blueprint JSON.

## Repository layout

- `docs/EVSO_Problem_Definition_Context.md` — the foundational problem context (multi-channel inquiries, business realities, constraints). Read this first for any new design work.
- `docs/Clickup/EVSO_ClickUp_Context_Snapshot.md` — current ClickUp setup: Spaces, Lists (NOWE ZAPYTANIA, ZLECENIA), Custom Fields, Automations, naming conventions.
- `docs/Dataflows/EVSO_Lead_Intake_Dataflow_Context.md` — how leads flow from websites/forms/email into the company.
- `docs/Make/EVSO_Form_Intake_v2.blueprint.json` — exported Make.com scenario blueprint (source of truth for the Phase 1 form intake automation).
- `docs/Plans/Done/` — completed phases. Phase 1 (Form Intake AI) is done: Forminator/WPForms → Make → Claude Haiku → ClickUp task in NOWE ZAPYTANIA, with 16 Custom Fields including AI classification, summary, completeness, confidence.
- `docs/Plans/EVSO_Implementation_Guide_Phase2.md` — **the active work**. Phase 2 covers follow-up reminders, lead deduplication, AI draft replies, and ZLECENIA workflow automation.
- `docs/Plans/EVSO_Milestone2_Specification.md` — Milestone 2 spec: ClickUp Agentic Email System (every email between EVSO and a client must live inside exactly one ClickUp task). Defines 8 threading scenarios S1–S8.
- `docs/Plans/EVSO_M2_*` — supporting Milestone 2 artefacts (architecture meeting, team charter, agent evaluation, problem definition, "jak to rozwiązujemy").
- `docs/Plans/EVSO_Phase2_Expert_Meeting_Output.md` — decision records, uncertainties, and resolved disagreements from the four-expert workshop validating Phase 2.
- `docs/Teams/EVSO_DevTeam.md` — the four-persona expert panel used for AI-facilitated design meetings (Marta Kurek/ClickUp, Tomasz Sowa/Make, Aleksander Bryła/AI reliability, Karolina Wysocka/Sales process). Use these personas with the `expert-meeting-facilitator` / `expert-team-builder` skills when running design sessions.
- `translations/` — Polish translations of the implementation guides and expert meeting outputs. The user is Polish; primary documents are written in English but client-facing/team-facing artefacts often need a PL version.

## Key technical context (don't re-derive each session)

- **Stack:** Make.com (scenarios + data stores), ClickUp (Spaces/Lists/Custom Fields/Automations/Email-in-ClickUp/Brain/AI Fields), Anthropic API.
- **Model in use:** `claude-haiku-4-5-20251001` (chosen for cost on ~100 inquiries/week). API key lives in Make.com.
- **Phase 1 baseline (live):** Two Make scenarios — "EVSO - Form Intake Enhanced" and "EVSO - Email Intake". Both call Claude Haiku for classification + extraction, fall back to `KLASYFIKACJA_AI = "Do weryfikacji"` on failure. Tasks named `[DATA] | [MIASTO] | [USŁUGA] | [IMIĘ]`. The `EMAIL` custom field is the foundation Milestone 2 builds on.
- **Phase 2 / Milestone 2 invariant:** Changes to existing live scenarios must be **additive** (router branches, appended modules), never destructive. The Phase 1 pipeline must keep working.
- **Threading strategy (M2):** Three layers — (1) ClickUp native email per-task `@mg.clickup.com` address handles in-thread replies, (2) Make watcher matches off-thread emails by sender email + phone, (3) disambiguation logic for clients with multiple open leads.

## How to work in this repo

- The user expects you to **read existing context before proposing**. Problem definition, dataflow, and ClickUp snapshot files exist precisely so you don't reinvent assumptions.
- When designing, **respect the expert panel's shared rule** (from `docs/Teams/EVSO_DevTeam.md`): never propose something only because it's technically possible — only if it is simpler, more stable, or genuinely more useful than the alternative.
- Documents are the deliverable. Prefer editing existing markdown files over creating new ones. New top-level documents belong under `docs/Plans/` (active) or `docs/Plans/Done/` (after the phase ships).
- Polish and English coexist. File names and headings are typically English; document bodies vary. Match the language of the file you are editing. When the user asks for a translation, mirror the structure in `translations/`.
- Make blueprints (`*.blueprint.json`) are exported from Make.com — treat them as authoritative state, not as something to hand-edit casually. If a scenario change is needed, describe the diff in markdown; the user applies it in the Make UI and re-exports.
- Custom Field names, list names, and statuses are in Polish (e.g. `NOWE ZAPYTANIA`, `ZLECENIA`, `KLASYFIKACJA_AI`, `KOMPLETNOŚĆ`, `Do weryfikacji`). Preserve them verbatim — they are the actual identifiers in the live ClickUp workspace.
