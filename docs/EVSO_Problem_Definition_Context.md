# EVSO — Problem Context for Lead Intake, Qualification, and Inquiry Handling

## Purpose of this document

This file is a **problem-definition context** for an LLM that will later help design and reason about a system to improve how inquiries are received, understood, qualified, and handled.

It is intentionally focused on:
- the **business problem**,
- the **nature of incoming inquiries**,
- the **operational realities**,
- the **constraints and expectations** around the future solution.

It is **not** a solution document.  
It does **not** propose architecture, tooling decisions, automations, or implementation details.

---

## Core problem statement

The company receives customer inquiries related to event organization through multiple channels. These inquiries vary greatly in structure, completeness, and intent. Some are very precise, while others are vague, incomplete, or loosely phrased. The current handling process creates operational load and makes it difficult to process inquiries consistently, quickly, and with high confidence.

The task to be solved is therefore:

> Define and later design a reliable way to capture, understand, qualify, organize, and support the handling of incoming customer inquiries, so that the team is meaningfully relieved from manual chaos while preserving control over important decisions and customer communication.

---

## Business context

The company operates in the event / entertainment / hospitality space and offers a **broad and variable event portfolio**.

The offer includes:
- multiple cities across Poland,
- different event formats,
- different vehicles / venues / service modes,
- both private and business events,
- both simple and more complex multi-element requests.

Examples of event types and service variety include:
- tram events,
- boat events,
- bus events,
- bachelor parties,
- bachelorette parties,
- birthdays,
- corporate events,
- custom events,
- short single-activity events,
- more complex multi-day requests.

This means incoming inquiries cannot be treated like a narrow, highly standardized lead form. The company operates in a domain where requests are often semi-structured or fully unstructured.

---

## Public entry points / inquiry sources

The company has multiple public-facing websites that are part of the inquiry funnel:

- `partytram.fun`
- `partyboat.fun`
- `busparty.fun`

These websites are similar in structure and act as customer entry points for event inquiries.

Customers can enter the process in at least two major ways:

1. **Website forms**
2. **Direct emails**

This means the future system must be described with awareness that not every inquiry starts from the same type of input, and not every inquiry contains the same fields or level of clarity.

---

## Nature of incoming inquiries

Incoming inquiries are highly variable.

Some inquiries contain a full or nearly full set of useful information, for example:
- city,
- event type,
- package,
- email,
- phone number,
- preferred date,
- hour,
- number of participants,
- free-text description.

Other inquiries are incomplete and may contain only a small subset of the above.  
Some may include only:
- a short free-text message,
- an email address,
- a phone number,
- a rough intent,
- or only a broad idea with no clear operational details.

Some inquiries are also **not straightforward sales leads**. They may instead be:
- very general questions,
- broad requests for information,
- loosely defined event ideas,
- edge cases,
- spam,
- irrelevant messages,
- unclear messages that require interpretation before the team can even decide what they are.

This variability is central to the problem.

---

## Why the problem is difficult

The difficulty does not come only from volume.  
It comes from the **combination** of:

- multiple entry channels,
- inconsistent data quality,
- different event categories,
- different cities and operational contexts,
- mixed structured and unstructured inputs,
- occasional spam or irrelevant messages,
- the need for fast and competent customer response,
- the need to preserve control over the commercial process,
- the need to reduce repetitive manual work without creating fragile complexity.

In other words, the challenge is not just “receive leads.”  
The real challenge is to **transform messy real-world customer intent into something workable for an internal team**.

---

## Current operational pain

At a high level, the business is dealing with a lead/inquiry flow that is burdensome because:

- inquiries do not arrive in one clean format,
- information must often be interpreted manually,
- different team members may need to understand or continue the same case,
- incomplete inquiries require additional follow-up,
- some inquiries are not worth full manual effort,
- volume is high enough to create recurring overload,
- the team wants help from AI, but the process still needs to remain practical and trustworthy.

A key reality is that the team does **not** need a fully autonomous system that handles everything without humans.  
What is needed is a system that is **genuinely helpful** in a messy operational environment.

---

## Approximate scale and intensity

The current influx is significant enough to matter operationally.  
The team has indicated that incoming messages can reach approximately **100 per week**, which is enough to create real pressure and inefficiency if the flow is not handled consistently.

This is not a “massive enterprise ticketing” scale, but it is absolutely large enough that:
- manual triage becomes expensive,
- inconsistency becomes costly,
- follow-ups get harder,
- and process quality becomes dependent on individual discipline.

---

## What the future system must conceptually support

Without prescribing implementation, the future task clearly involves a system that can conceptually support the following:

### 1. Intake across multiple channels
The system must account for both:
- website-based inquiry submission,
- and direct email-based inquiry initiation.

### 2. Understanding mixed-quality inputs
The system must be able to deal with:
- highly structured submissions,
- partially filled submissions,
- vague free-text descriptions,
- and messages that do not clearly match a predefined template.

### 3. Distinguishing between different kinds of inbound messages
The system must help distinguish between:
- real leads,
- incomplete but valid leads,
- general inquiries,
- edge cases,
- and spam / irrelevant submissions.

### 4. Preserving the original customer message
The original wording and source message matter and should remain understandable as part of the operational record.

### 5. Supporting a human team
The process must help humans:
- understand the inquiry faster,
- see what is missing,
- decide what to do next,
- and communicate more efficiently.

### 6. Handling follow-up needs
Because many inquiries are incomplete, the process must conceptually support situations where additional information is needed before meaningful commercial handling can continue.

### 7. Supporting later communication
An inquiry is not just a one-time input. It often becomes an ongoing conversation. The future problem framing must therefore acknowledge that communication continuity matters.

### 8. Remaining reliable in messy real-world usage
The eventual solution must be useful in normal day-to-day operations, not only under ideal, neatly structured input conditions.

---

## Actors involved

The later solution will need to make sense for several roles:

### Customers
People or organizations submitting inquiries through websites or email.

### Sales / inquiry-handling team
Internal users who need to:
- review new inquiries,
- decide whether they are valid,
- understand what is missing,
- respond,
- move the case forward.

### Operations / event delivery side
Some inquiries eventually become real events, meaning the upstream qualification quality affects downstream delivery quality.

### AI / automation layer
There is a clear intention to use AI and automation as part of the future solution, but as a supporting layer rather than an unquestioned decision-maker.

### Technical implementer(s)
The company has technical capability, but not an unlimited engineering team. The implementation burden matters.

---

## Elements that are in play

The following elements are relevant to the overall problem space:

- public websites,
- website forms,
- email inboxes,
- customer free-text messages,
- different cities,
- different service categories,
- different event types,
- internal team handling,
- AI assistance,
- automation tooling,
- existing operational systems,
- data normalization needs,
- communication continuity.

These are all part of the real problem context, even if the eventual design will decide to emphasize some more than others.

---

## Important characteristics of the desired future outcome

The team has already indicated strong preferences about the nature of the eventual outcome.

The future solution should be:

### Simple
It should not become a fragile overengineered machine that only one person understands.

### Effective
It should produce real operational relief, not just an impressive-looking automation.

### Reliable
It should behave predictably in real conditions, especially when data is incomplete, inconsistent, or messy.

### Helpful
It should reduce repetitive cognitive load and assist the team meaningfully.

### Practical for a small technical team
The company has technical competence, but the system should remain realistic for a setup where implementation and maintenance capacity are limited.

### Human-compatible
The system should support the team’s work, not fight it.

---

## What success would look like conceptually

A successful future solution would mean that incoming inquiries no longer feel like raw chaos. Instead, they become:

- easier to understand,
- easier to classify,
- easier to continue,
- easier to respond to,
- and easier to manage consistently across the team.

From the team’s perspective, success would mean:
- less manual triage,
- fewer ambiguous handoffs,
- less time wasted on low-value or unclear messages,
- better visibility into what a message actually is,
- and more confidence that important inquiries are not mishandled.

From the business perspective, success would mean:
- higher operational consistency,
- better response quality,
- better speed,
- and reduced loss caused by disorder.

---

## Constraints and realism

The context of the task includes several practical realities:

- the company already has a live operating environment,
- some basic automations already exist,
- inquiry volume is material,
- the offering is broad and non-trivial,
- the input quality is inconsistent,
- the team wants AI to help,
- but the team does not want unnecessary technical complexity,
- and the solution must be realistic for limited implementation capacity.

This means that later solution design must take into account not only “what is possible,” but also:
- what is maintainable,
- what is understandable,
- and what is robust enough to use every day.

---

## What this document intentionally does not include

This context file does **not** define:
- system architecture,
- workflow design,
- tool selection,
- automation logic,
- data model details,
- prompt design,
- AI policy,
- routing rules,
- implementation roadmap.

Those topics belong to later stages.

This file exists only to define the **problem space clearly and completely enough** that a later LLM can reason about the task from the right starting point.

---

## Condensed problem summary

EVSO needs to improve how customer inquiries are handled across websites and direct email. The company offers many kinds of events across many cities, so incoming messages vary greatly in structure, completeness, and clarity. Some contain all key details, some are partial, some are broad or ambiguous, and some are spam.

The current state creates manual burden and inconsistency. The business wants AI and automation to help, but the future solution must remain simple, reliable, practical, and genuinely useful. The problem to solve is not only technical intake, but the broader transformation of messy inbound customer intent into something that an internal team can understand, continue, and respond to effectively.

---

## Suggested usage of this file

This file should be provided to an LLM when the LLM needs to understand:
- what business problem is being solved,
- why the problem is difficult,
- what kinds of inputs and variability exist,
- what outcomes matter,
- and what constraints should shape future design work.

It should be treated as a **problem-definition context**, not as a technical blueprint.
