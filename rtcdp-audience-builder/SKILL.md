---
name: rtcdp-audience-builder
description: >-
  Build accurate, repeatable audiences (segments) in Adobe Real-Time CDP / Adobe Experience
  Platform from a plain-language description of the target group. Use this whenever the user wants
  to create, define, draft, translate, or validate an RTCDP / AEP audience or segment — including
  cart/browse abandonment, purchaser, lapsed-customer, loyalty, or any include/exclude or
  sequential ("did A then not B") logic. Trigger it even when the user just describes a group of
  people they want to reach ("people who checked out but didn't buy", "Gold members who lapsed")
  without saying the word "audience". The skill enforces the core correctness rule that profile
  attributes and experience events live in **separate logic blocks** — profile attributes in the
  Attributes container, experience events (keyed by eventType or an event field) in the Events
  container, never mixed — and it always asks the user for the audience name. Portable: works the same
  in Claude and in Adobe Coworker.
---

# RTCDP Audience Builder

Turn a description of a target group into a correct, repeatable Real-Time CDP audience whose
**Attributes logic and Events logic are kept separate**, with eventTypes/fields sourced from the
right schema. The deliverable is the audience itself — the two containers and their rules, rendered
the way the Segment Builder shows them.

## Why this skill exists

Hand-built RTCDP audiences go wrong in predictable ways: an event condition gets written into the
attribute logic (so it never matches), an event has no time window (so it over-qualifies), an
exclusion blanket-removes profiles instead of targeting one eventType, or `eventType` strings are
guessed. This skill removes those failure modes with a fixed classify → validate → preview → save
workflow and a small semantic language the user can reason about.

The invariants it guarantees:
- **Profile attributes** (who someone *is*) come from the **Individual Profile union schema** and
  live in the **Attributes** container.
- **Experience events** (what someone *did*) come from the **ExperienceEvent union schema**, are
  keyed by **`eventType`** (or by a field match), are **time-scoped when the prompt references
  time**, and live in the **Events** container.
- The **Attributes logic** and the **Events logic** are **separate blocks**. A profile attribute
  never appears in the Events logic, and an event/`eventType`/event-field condition never appears in
  the Attributes logic. The separation is structural (two distinct containers), so it holds by
  construction, not by luck.
- The audience is authored **UI-editable** — built with attribute-based composition so a marketer can
  reopen and edit it in the Segment Builder, never as a read-only raw-PQL segment (Step 7).

## Portability (Adobe Coworker and elsewhere)

This skill is self-contained — SKILL.md plus the three files in `references/`. It refers to RTCDP
capabilities by role, not by a specific vendor tool binding: an **audience-preview** capability, an
**audience-create** capability, and an **inspect/search** capability. Whatever environment runs it
(Claude, Adobe Coworker, another agent with AEP tools) should map those roles to whatever tools it
has. If a capability is missing, use the "Graceful degradation" fallback rather than failing.

---

## Workflow

Work through these steps in order. Do not skip step 0 (the name) or step 4 (validation).

### Step 0 — Get the audience name (required)

Always ask the user what to name the audience, unless they already gave one. Do not invent a final
name silently. Offer a suggestion using the convention in `references/recipes.md`
(`<intent> - <key events/attrs> (<window>)`), but let the user confirm or override. Also capture an
optional description.

### Step 1 — Understand the target group (and take the prompt literally)

Get the user's plain-language definition of who should be in the audience, then build **only what the
prompt actually says**. Do not add conditions, windows, or exclusions the user did not express — the
most common cause of a wrong audience is the builder inferring intent that wasn't there.

For each distinct idea in the prompt:
- Is it a trait or an action? (drives Attributes vs Events — see Step 2)
- For each action: which behavior, and how many times (**frequency**, default "at least 1")?
- Does **order** matter? ("did A *then* B" → a sequence.)
- Target entity: people (`profile`, default) or B2B `account`?

**Two rules that override any default (this is the whole point of this step):**

1. **Time — only when referenced.** Add a time window (and only *ask* about time) **only if the
   prompt contains a time reference** — e.g. "in the last 24 hours", "this week", "last month",
   "today", "within 30 days". If the prompt has **no** time reference, do **not** add a window, do
   **not** propose a default, and do **not** prompt for one; the event condition is all-time. When a
   window is given, **preserve the user's unit** — "24 hours" stays hours (`in last 24 hour(s)` /
   `<= 24 hours before now`), do not convert it to "1 day".

2. **Exclusion — only when referenced.** Create an exclusion (a negated / "Exclude any" / `!C`
   step) **only if the prompt contains negation language**: *did not, didn't, does not, doesn't, did
   not happen, did not occur, has not / hasn't, have not, never, without, except, excluding, no
   [event]*. If there is no negation, **every** condition is an include — even one that follows
   "then". For example, "A then B" with no negation is two includes in sequence; "A then B did not
   happen" makes B an exclusion.

Ask only for what's genuinely missing to render the rule — never to add scope the user didn't ask
for.

### Step 2 — Classify every condition

Read `references/eventtypes.md` and apply its decision rule to each condition:
- Standing trait ("is Gold", "lives in CA", "opted in", a **predicted/propensity score** like
  `creditCardUpgradePrediction = 1`) → **profile attribute** → Attributes container (Attributes
  logic only).
- Point-in-time action ("bought", "added to cart", "opened email", "**viewed page X**") →
  **experience event** → Events container.

**An event is identified by a data source plus optional field conditions — pick the right mode:**

- **By `eventType`** — the precise mode. Anchor on the matching enum value (e.g.
  `commerce.checkouts`). Best when the behavior maps to a standard/custom eventType.
- **By event field, using the "Any event" source** — when the user identifies the action by a
  *name or attribute* rather than a type. This is exactly how "page view equals X" works: the data
  source is **Any event** and the match is on the page-name field (**Name**, path
  `web.webPageDetails.name`). It renders as `Any event where (Name equals "X")`. The user's
  vocabulary drives this: "page view equals 'Credit Card Application Started'" →
  `Any event where (Name equals "Credit Card Application Started")`, **not** a `commerce.*`
  eventType.

Both modes produce **event** conditions in the **Events** container — never attributes. See
`references/eventtypes.md` §3.5 for the field-match catalog (page name, link name, etc.).

Propose the concrete value that fits the user's intent (the eventType from
`references/eventtypes.md` §3, or the field + value for a "page view"/"Any event" match) and state
your choice so they can correct it. Custom eventTypes and custom field values are valid — use them
verbatim if the user names one. When building against a live sandbox, confirm exact strings and
field paths per `references/eventtypes.md` §4 (learn the shape from an existing audience).

### Step 3 — Write the semantic spec

Express the whole audience in the semantic DSL below. This is the human-reviewable "source" for the
audience — showing it to the user makes the logic reviewable and the audience re-creatable later.

```yaml
audience:
  name: "<required — from Step 0>"
  description: "<optional>"
  entity: profile            # profile | account
  mergePolicy: <optional id or name>
  logic:
    # INCLUDE: profiles must satisfy these (combined with AND unless you set `match: any`)
    include:
      - attribute: <profileFieldPath>      # e.g. homeAddress.countryCode, creditCardUpgradePrediction
        operator: equals|not_equals|greater_than|less_than|contains|starts_with|exists|not_exists
        value: <value>
      # An event, identified EITHER by eventType OR by source: any + where (field match):
      - event:
          eventType: <eventType>           # mode A: e.g. commerce.checkouts
          within: <window>                 # OPTIONAL — include ONLY if the prompt referenced time
          atLeast: <n>                     # or exactly / atMost / between: [n, m]; default atLeast: 1
      - event:
          source: any                      # mode B: "Any event" data source
          where:                           # field condition(s) that identify the event
            - field: web.webPageDetails.name   # display label "Name" (a page view)
              operator: equals
              value: "<page name>"
          within: <window>                 # OPTIONAL — omit entirely if no time reference in prompt
          atLeast: <n>
    # EXCLUDE: include ONLY if the prompt used negation language (did not / didn't / never / without…)
    # Each block targets ONE event (type or field match) only.
    exclude:
      - event:
          eventType: <eventType>           # or source: any + where: [...]
          within: <window>                 # OPTIONAL — only if time referenced
          atLeast: 1
    # SEQUENCE: ordered "A then B" (use when the prompt says "then" or order matters)
    sequence:
      within: <window>                     # OPTIONAL overall window — only if time referenced; preserve the user's unit
      gap: <duration>                      # gap between steps; default "10 minutes" for sequences (silent, never prompted)
      steps:
        # each step is include or exclude; exclude ONLY if the prompt negated that step
        - { verb: include, atLeast: 1, source: any, where: [ { field: web.webPageDetails.name, operator: equals, value: "A" } ] }
        - { verb: exclude,             source: any, where: [ { field: web.webPageDetails.name, operator: equals, value: "B" } ] }  # "B did not happen"
```

Notes:
- **`within` is optional.** Include a window on an event/sequence **only if the prompt referenced
  time**, and preserve the user's unit ("24 hours" → hours, not "1 day"). No time reference → no
  window, all-time, and don't prompt for one (Step 1, rule 1).
- **Exclusion is negation-driven.** A step is `verb: exclude` (or an `exclude` block) **only if the
  prompt negated it** (Step 1, rule 2). "A then B" with no negation = two `include` steps. "A then B
  did not happen" = include A, exclude B.
- Put each event condition in its own block/step; the same event may appear as both an include and an
  exclude with different windows.
- **Sequence gap:** a sequence renders "then within `<gap>` …". If the prompt states a gap
  ("then within 10 minutes"), use it; otherwise apply the default `gap: 10 minutes` silently (a
  default is not a prompt). Make the default configurable via user preference.
- An event's identity is `eventType` (mode A) **or** `source: any` + `where` field conditions
  (mode B). Mode B is how "page view equals X" works.
- See `references/recipes.md` for generic patterns (attribute-only, event-only, attribute+event,
  sequence with/without a negated step).

### Step 4 — Validate (gate — must pass before preview/save)

Run this checklist against the spec. If any item fails, fix it and re-check.

- [ ] **Name present.** `audience.name` is set and confirmed by the user.
- [ ] **Every attribute is a profile trait**, referencing an Individual Profile field and expressed
      in the **Attributes logic only** — it carries no event or `eventType`.
- [ ] **Every event is an experience event**, identified by a real `eventType` **or** by
      `source: any` + a `where` field match, and expressed in the **Events logic only**.
- [ ] **Time only when referenced.** An event/sequence has a `within` **only if the prompt
      referenced time**; no fabricated or default windows. Where present, the unit matches the
      prompt ("24 hours" → hours, not days).
- [ ] **Separate logic, no leakage:** the Attributes logic contains only profile attributes and the
      Events logic contains only events — neither borrows a condition from the other.
- [ ] **Exclusion only when referenced.** There is an excluded step/block **only if the prompt used
      negation** (did not / didn't / never / without / did not happen / did not occur …). No negation
      → no exclusion, even after "then". Each exclusion targets exactly one event, not a blanket
      removal.
- [ ] **Order intent matches structure:** if the prompt says "then" or order matters, it's a
      `sequence`; otherwise plain include/exclude.
- [ ] **Entity/merge policy** are sensible for the use case (account rules → `entity: account`).
- [ ] **Relational safety:** if the rule is relational/cross-entity *and* references a person
      experience event, it will be refused — decompose into a segment-of-segments (see
      `references/logic-and-rendering.md`). Plain attribute+event profile audiences are fine.

### Step 5 — Render the audience output (Attributes logic + Events logic, kept separate)

The deliverable is the audience the way the Segment Builder shows it: an **Attributes** block and an
**Events** block, presented as two clearly separate sets of logic, joined at the audience level with
AND/OR. Always show them under distinct headings so the separation is obvious — never interleave an
event into the Attributes block or an attribute into the Events block.

Match the exact phrasing RTCDP's Segment Builder uses, so what you show equals what they'll see in
the tool.

**Attributes logic** — one row per profile attribute:
- `Include <fieldPath> <operator> <value>`  (e.g. `Include creditCardUpgradePrediction equals 1`)

**Events logic** — one row per event condition:
- Positive event row: `Include at least <N> <Source> event where (<Field> equals <Value>)`
- Negated event row (only when the prompt negated it): `Exclude any <Source> event where (<Field> equals <Value>)`
- A sequential step after the first is prefixed `then within <gap> ` (gap default 10 minutes).
- The Events block is time-scoped **only if time was referenced**, preserving the unit:
  `and occurs in last <N> <unit>(s)` (e.g. `in last 24 hour(s)`). No time reference → omit the
  `and occurs …` clause entirely.

`<Source>` is `Any` for a `source: any` event, or the eventType for a mode-A event. See
`references/logic-and-rendering.md` for the full wording rules (time, frequency, exclusion,
sequences).

Sequence shape for a negated second step with a 24-hour window:
```
Events
(Include audience who Include at least <N> <Source> event where (<Field> equals <Value>)
then within <gap> Exclude any <Source> event where (<Field> equals <Value>)) and occurs in last <N> <unit>(s)
```

Worked target — the credit-card prompt ("page view equals 'Credit Card Application Started' then
where page view equals 'Credit Card Application Submit' did not happen in the last 24 hours"). "did
not happen" ⇒ Submit is an **Exclude**; "24 hours" ⇒ `in last 24 hour(s)` (unit preserved); gap
defaults to 10 minutes; there is no attribute in this prompt, so the Attributes block is empty:
```
Events
(Include audience who Include at least 1 Any event where (Name equals Credit Card Application Started)
then within 10 minutes Exclude any Any event where (Name equals Credit Card Application Submit)) and occurs in last 24 hour(s)
```

Present the two blocks to the user and confirm before preview/save.

### Step 6 — Preview (validate + size)

If an audience-preview capability is available, submit the audience definition to estimate size and
confirm it is valid:
1. Assemble the definition by mirroring an existing audience's structure (see
   `references/logic-and-rendering.md`). Do **not** guess the definition internals from memory.
2. Submit → get a `previewId`. Then fetch with the same `previewId` until `state = RESULT_READY`.
   Never treat an interim `NEW`/`0` as the real size (false zero).
3. If the size looks implausible (0, or the whole base), re-examine: usually an event ended up in the
   Attributes logic, a window is wrong, or an exclude is too broad. Fix and re-preview.

For the current Adobe RTCDP MCP, the preview tool is `preview_audience_membership`
(`predicateModel` = `_xdm.context.profile` or `_xdm.context.account`). It needs `imsOrgId` and
`sandboxName`; if you don't have them, ask — never switch org/sandbox context on your own.
`search_organizations` lists available orgs.

### Step 7 — Save (as a UI-editable audience, not read-only PQL)

Author the audience so a marketer can still open and edit it in the Segment Builder afterward. That
means building it with the Segmentation API's **attribute-based composition**, not a raw PQL string.

When an audience-create capability is available:

1. **Use `"compositionType": "attribute"`.** This produces a visually editable segment. **Do NOT**
   use `"compositionType": "pql/text"` with a raw PQL string — that creates a read-only, code-based
   segment that cannot be modified in the UI, which defeats the purpose of this skill.
2. **Pre-resolve the full schema path for every field**, starting from `_xdm.context.profile` (and
   the ExperienceEvent union for event fields). Confirm each path against the sandbox before building
   — don't guess. Attribute-based composition requires real, fully-qualified field references.
3. **Construct the expression payload as a structured attribute tree** — a node per condition with
   its `type`, `value`, and `operator`, combined by group nodes — mirroring the two-block structure
   (Attributes logic and Events logic kept separate). Do not fabricate the tree shape from memory;
   learn it from an existing attribute-composed audience (`references/logic-and-rendering.md` §7).
4. Save using the confirmed name, description, entity, and merge policy.
5. **Cap authoring at 2 attempts.** If it still won't validate after a second try, **save the
   audience as a draft and report that** — including the validation error and the two-block output so
   the user can finish it in the UI. Do **not** keep iterating past two attempts.

Report the created (or draft) audience's ID/name and the preview size.

### Step 8 — Report

Summarize: name, plain-language intent, the two-block breakdown (Attributes logic and Events logic
with eventTypes/fields and any windows), any exclusions/sequences, the preview size, and the saved ID
(if saved). Offer the semantic spec back to the user so they can store it and re-create the audience
later.

---

## Graceful degradation

If a create (or even preview) capability is not available in the current environment, still deliver
full value: the validated semantic spec, the two-block breakdown (Attributes logic and Events logic),
the Segment Builder wording, and a **click-path** — which attribute rows to add; which event rows to
add with which `eventType`/field, windows, and counts; which rows to mark **Exclude**; and where to
use the **Then** connector for sequences. Correct-and-buildable always beats emitting a guessed
definition.

## Reference files

- `references/eventtypes.md` — the two union schemas, the attribute-vs-event decision rule, the
  standard `eventType` catalog, identifying events by a field ("Any event"), and how to confirm
  values against a live sandbox. Read when classifying conditions or choosing eventTypes/fields.
- `references/logic-and-rendering.md` — the two containers and how to keep their logic separate, the
  Segment Builder wording rules (time only when referenced, frequency, exclusion only when negated,
  sequences with gaps), and a brief note on preview/create. Read when rendering the audience output.
- `references/recipes.md` — generic audience patterns (attribute-only, event-only, attribute+event,
  sequence with/without exclusion) and the naming convention. Read for reusable structures.

## Guardrails

- Always ask for and confirm the audience **name** (Step 0).
- **Build only what the prompt says.** Add a time window only when time is referenced (preserve the
  unit); add an exclusion only when the prompt uses negation. Never infer either.
- **Keep the Attributes logic and Events logic separate** — never place an event in the Attributes
  block or an attribute in the Events block; never preview or save a spec that fails the Step 4
  checklist.
- **Save UI-editable.** Build with `"compositionType": "attribute"` and a structured attribute tree
  over fully-resolved schema paths — never `"compositionType": "pql/text"`. Cap authoring at 2
  attempts, then save as a draft and report (Step 7).
- Never guess `eventType` strings or field paths for a live build — confirm against the sandbox.
- Never switch IMS org or sandbox context without asking the user.
