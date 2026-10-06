# Journey Concepts and Resolution Rules

The building blocks of an AJO journey, and — most importantly — the rule that every named thing
(action, audience, channel configuration, content template) must be **resolved from the live AJO
configuration or confirmed with the user**, never invented.

> **Golden rule:** never fabricate the name/ID of an action, audience, channel configuration, event,
> or content template. Resolve it from AJO's configuration (list what's available and match the
> user's intent), and if you can't resolve it confidently, **ask**. Guessing a name that doesn't
> exist is the fastest way to a broken journey.

---

## 1. A journey = one entry + ordered activities

Every journey has exactly one **entry** (how profiles get in), then a flow of **activities**
(orchestration + actions), then exits. Keep this shape consistent across journeys.

### Entry — how profiles enter (pick one)

| Entry | Journey type | When to use | Enters |
|-------|--------------|-------------|--------|
| **Unitary event** | Unitary event journey | Real-time trigger tied to one person ("when someone purchases / submits a form / logs in / abandons a cart") | One profile, in real time |
| **Audience Qualification event** | Audience Qualification journey | React in real time when a profile **enters or exits** an AEP audience | One profile, on qualify/disqualify |
| **Read Audience** | Read Audience journey | Batch/scheduled: pull everyone in an AEP audience once or on a recurring schedule | All profiles in the audience, at run time |
| **Business event** | Business event journey | Global signal (restock, price drop, flight change) broadcast to an audience; AJO auto-adds a Read Audience after it | Profiles in the targeted audience |

Notes that keep entries correct:
- A journey has **one** entry, and a Business event must be the **first** activity.
- "Start from an **audience**" → Read Audience (batch/scheduled) or Audience Qualification (real-time
  entrance/exit). Ask which cadence the user wants if it isn't clear.
- Read Audience / Audience Qualification journeys **cannot** contain a Jump activity.

### Activities — the flow

- **Orchestration:** Read Audience, **Wait/Timer**, **Condition** (branch on profile attribute,
  audience membership, or real-time event), Optimize, Jump (limited), etc.
- **Action:** delivers a communication or calls an external system (§3).
- Add an **alternative path** on actions/data steps so the journey continues on timeout/error.

---

## 2. Starting a journey "with an action"

AJO journeys always begin with an entry (event or audience) — an action is a downstream node, not an
entry by itself. When the prompt frames the start **as an action** ("start a journey that sends the
welcome email", "kick off with the loyalty API call"):

1. **Resolve the action** against AJO's available actions (§3) — list them and match the user's
   intent. If you can't match it confidently, **ask which action to use**.
2. **Determine the real entry** the action hangs off: usually a unitary event (e.g. "when they
   subscribe") or a Read Audience (e.g. "for everyone in audience X"). If the prompt doesn't make the
   entry clear, ask ("What should trigger this — an event, or reading an audience?").
3. Assemble entry → (optional wait/condition) → the resolved action.

Never wire up a journey whose action or entry is a name you invented. Resolve or ask.

---

## 3. Actions: channel actions and custom actions

Two kinds of action activity — both must be resolved from configuration:

- **Channel actions** (built in): **Email, SMS/RCS/MMS, Push, In-app, Web, Code-based**. Each send
  uses a **channel configuration** (§4).
- **Custom actions:** user-configured calls to third-party systems (Journey settings →
  Configurations → Actions). Resolve the specific custom action from what's configured, or ask.

When the user names an action, match it to a configured action. When they describe a channel ("send
an email/SMS"), that's a channel action — resolve its channel configuration next.

---

## 4. Channel configurations (and the email → "marketing" default)

A **channel configuration** (Channels → General settings → Channel configurations) defines the surface
for a send — including its **category: Marketing or Transactional**. There can be several per channel.

Resolution rules:
- **Email:** default to the channel configuration whose **name contains "marketing"** (case-
  insensitive). If several match, pick the closest and state your choice; if none match or it's
  ambiguous, **ask** which configuration to use. (Marketing-type emails must include an opt-out link —
  see §6 and brand guidelines.)
- **SMS / other channels:** use the available configuration for that channel; if more than one exists
  and intent is unclear, **ask**. Don't assume a transactional config for a marketing send or vice
  versa — the category lives in the configuration.
- Always confirm the chosen configuration name back to the user so they can correct it.

Never type a channel-configuration name from memory; list what exists and match, or ask.

---

## 5. Audiences

"Start from an audience" or "branch on audience membership" both need a **real AEP audience**:

- Resolve the audience by name from what's available in the sandbox. If you can't find a confident
  match, **ask which audience to use** (offer the closest matches).
- For entry, choose Read Audience (batch/scheduled) vs Audience Qualification (real-time
  entrance/exit) per §1.
- A yellow warning appears if you use a batch audience that has never been calculated — don't enter a
  brand-new batch audience immediately.

---

## 6. Creatives / content for an action

Whenever an action needs content (email, SMS, push, code-based), decide the creative source **with the
user** — do not silently generate content:

1. **Ask: create the creatives, or use a template?**
   - **Use a template** → pick an existing **content template** (resolve/list available templates;
     templates are scoped to HTML or JSON by the chosen channel configuration). Ask which template if
     unsure. Reusable **fragments** (Content Management → Fragments; visual fragments are email-only,
     max 30 per delivery, nest 1 level) can be dropped in for headers/footers/banners.
   - **Create the creatives** → author new content following `references/brand-guidelines.md` (voice,
     logo, color, type, layout, subject-line and CTA rules, mandatory footer/opt-out).
2. **Then ask: would you like to update the creatives?** Offer to iterate on subject line, preview
   text, body, CTA, imagery, personalization fields — and apply the brand guidelines to every change.
3. **Guardrails to enforce in content:** a **Marketing** email must include an **opt-out link**;
   transactional does not require one. Keep personalization fields real (map to profile/event fields);
   keep subject/preview within brand length rules.

Apply `references/brand-guidelines.md` to all authored or updated creatives so every journey looks and
sounds consistent.

---

## 7. Resolving from configuration vs asking — quick policy

- **Can list configuration** (an AJO config-read capability is available): list actions / audiences /
  channel configurations / content templates, match the user's intent, state your pick, let them
  correct.
- **Cannot list, or no confident match:** **ask** a single focused question with the likely options.
- **Never** proceed on an invented name. Consistency and correctness come from resolving against real
  configuration, not from guessing.
