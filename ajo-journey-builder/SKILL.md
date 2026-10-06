---
name: ajo-journey-builder
description: >-
  Build customer journeys in Adobe Journey Optimizer (AJO) efficiently and consistently from a
  plain-language description. Use this whenever the user wants to create, design, draft, or outline an
  AJO journey — welcome/onboarding, cart or browse abandonment, win-back/re-engagement, post-purchase,
  transactional, audience-qualification, or any multi-step messaging flow. Trigger it even when the
  user just describes the flow ("when someone subscribes, send a welcome email then a reminder")
  without saying "journey". The skill resolves every named thing — entry event, audience, action,
  channel configuration, content template — from the live AJO configuration or asks when unsure;
  defaults email sends to the channel configuration whose name contains "marketing"; and always
  decides creatives with the user (create vs template, then offer an update pass) using the brand
  guidelines. Portable: works the same in Claude and in Adobe Coworker.
---

# AJO Journey Builder

Turn a description of a customer flow into a correct, consistent Adobe Journey Optimizer journey:
a resolved **entry**, an orchestrated **flow**, **action** nodes wired to real channel configurations,
and **creatives** decided with the user under the brand guidelines.

## Why this skill exists

Hand-built journeys drift: an action points at a channel configuration that doesn't exist, an email
goes out on a transactional surface with no opt-out, an audience name is guessed, or creatives are
generated silently off-brand. This skill removes that drift with one discipline — **resolve every
named thing from the live AJO configuration or ask** — plus consistent recipes, a "marketing" email
default, and brand-guided creatives.

Principles it guarantees:
- **Resolve or ask, never invent.** Entry events, audiences, actions, channel configurations, and
  content templates are matched against what actually exists in AJO; if there's no confident match,
  the skill asks rather than fabricating a name.
- **Email defaults to the "marketing" channel configuration** (case-insensitive name match); other
  channels use their available configuration; ambiguity → ask.
- **Creatives are decided with the user** — create vs template — and the user is always offered an
  update pass, with the brand guidelines applied.

## Portability (Adobe Coworker and elsewhere)

Self-contained: SKILL.md plus the three files in `references/`. It refers to AJO capabilities by role —
a **configuration-read** capability (list actions/audiences/channel-configs/templates), a
**journey-create/publish** capability, and a **content-authoring** capability. Whatever environment
runs it maps those to its own tools. If a capability is missing, use "Graceful degradation".

---

## Workflow

### Step 0 — Name the journey

Ask for the journey name if not given (suggest one via the brand-guidelines convention
`<Program> - <Intent> (<channel/cadence>)`), and confirm before building. Capture an optional
description.

### Step 1 — Understand the flow (take the prompt literally)

Get the plain-language flow. Identify: how profiles **enter**, the **steps** in order (waits,
conditions, sends), which **channels/actions** are used, and any branching. Build only what the prompt
says; ask about anything genuinely missing rather than inventing it.

### Step 2 — Resolve the entry (audience or event/action)

Per `references/journey-concepts.md` §1–2, pick and resolve the entry:
- **Audience start** → Read Audience (batch/scheduled) or Audience Qualification (real-time
  entrance/exit). **Use an available AEP audience** — resolve it by name; if no confident match,
  **ask which audience to use**. Confirm cadence (once/recurring) or entrance/exit behavior.
- **Event / action start** → resolve the trigger. If the prompt **starts with an action**, grab the
  **actions available in AJO's configuration** and match it; if unsure, **ask what action to use to
  start the journey**. Then determine the real entry the action hangs off (unitary event or Read
  Audience) and ask if it isn't clear.

Never wire an entry whose audience/event/action is a name you invented.

### Step 3 — Resolve each action and its channel configuration

For every send/action node (`references/journey-concepts.md` §3–4):
- **Email** → use an available channel configuration and **default to the one whose name contains
  "marketing"** (case-insensitive). If several match, pick the closest and state it; if none/ambiguous,
  **ask**. (Marketing emails require an opt-out link.)
- **SMS / push / other** → use the available configuration for that channel; **ask** if more than one
  exists and intent is unclear.
- **Custom action** → resolve the specific configured action; ask if unsure.
- Confirm each chosen configuration name back to the user.

### Step 4 — Decide creatives (create vs template), then offer an update

For each action that needs content (`references/journey-concepts.md` §6):
1. **Ask the user: create the creatives, or use a template?**
   - **Template** → resolve/select an available **content template** (ask which if unsure); optionally
     add **fragments** (email header/footer/banner).
   - **Create** → author new content following `references/brand-guidelines.md`.
2. **Then ask: would you like to update the creatives?** Iterate on subject/preview/body/CTA/imagery/
   personalization, applying the brand guidelines to every change.
3. Enforce: marketing email includes an **opt-out link**; personalization fields are real with
   fallbacks; subject/preview within brand length rules.

Apply `references/brand-guidelines.md` to everything authored or updated.

### Step 5 — Assemble the journey (consistent structure)

Lay out `[Entry]` → `{orchestration: waits/conditions}` → `((actions))` → exits, with alternative
paths on actions/data steps for timeout/error. Reuse a matching blueprint from
`references/journey-recipes.md` for consistency. Present the outline (entry, each node with its
resolved action/channel config, creative decision, waits/conditions, branches) for confirmation.

### Step 6 — Validate (gate — must pass before build)

- [ ] **Name** set and confirmed.
- [ ] **Entry resolved** to a real event/audience (or asked); one entry; Business event first if used.
- [ ] **Every action resolved** to a real channel/custom action (or asked).
- [ ] **Channel configs resolved**; email uses the **marketing** config unless it's a transactional
      message (then a transactional config, confirmed).
- [ ] **Creative decided with the user** (create vs template) and an update pass offered for each.
- [ ] **Brand guidelines applied**; **marketing emails include an opt-out link**.
- [ ] **No invented names** anywhere — everything is resolved or confirmed.
- [ ] Waits/conditions/branches and exits are explicit.

### Step 7 — Build / publish

If a journey-create capability is available, build the journey with the resolved entry, activities,
actions, channel configurations, and creatives; validate; publish only on user confirmation. Learn the
exact journey payload shape by mirroring an existing journey rather than guessing. **Cap build attempts
at 2** — if it still won't validate, **save the journey as a draft and report** (with the error and the
outline) rather than iterating further.

### Step 8 — Report

Summarize: name, entry, the ordered flow with each resolved action + channel configuration, creative
decisions, brand-guideline notes (e.g. opt-out included), and the created/draft journey ID. Offer the
outline back so the journey can be reproduced.

---

## Graceful degradation

If create/config-read capabilities aren't available, still deliver full value: the resolved-as-far-as-
possible journey **outline**, a **click-path** in the Journey Designer (which entry to add; each
activity in order; which channel configuration to select — email = the "marketing" one; where waits/
conditions go), the creative decisions, and any authored creative drafts per the brand guidelines.
Where a name couldn't be resolved, mark it clearly as "confirm in AJO" instead of inventing one.

## Reference files

- `references/journey-concepts.md` — entry types, activities, actions, the **resolve-or-ask** golden
  rule, channel configurations (email → "marketing" default), audiences, and the creatives decision.
  Read when resolving anything or choosing the entry.
- `references/journey-recipes.md` — common journey blueprints (welcome, cart/browse abandonment,
  win-back, audience-qualification, post-purchase, transactional, action-first). Read for consistent
  structure.
- `references/brand-guidelines.md` — the brand standards applied to all authored/updated creatives.
  Customize per brand. Read whenever creating or updating a creative.

## Guardrails

- **Resolve or ask — never invent** an entry, audience, action, channel configuration, or template.
- **Email defaults to the "marketing" channel configuration**; ask when it's ambiguous.
- **Always decide creatives with the user** (create vs template) and **offer an update pass**; apply
  the brand guidelines; marketing emails must include an opt-out link.
- Cap build attempts at 2, then save a draft and report.
- Never switch AJO org/sandbox context without asking.
