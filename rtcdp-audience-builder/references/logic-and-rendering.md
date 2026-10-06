# Audience Logic and Rendering

How a validated audience spec becomes the audience output the Segment Builder shows — with the
Attributes logic and the Events logic kept as two separate blocks. The focus here is the **audience
output**, not query syntax.

Contents:
1. The two containers (separate logic)
2. Identifying an event (eventType vs "Any event" + field)
3. Time — only when referenced
4. Frequency
5. Exclusion — only when negated, and targeted
6. Sequences ("A then B")
7. Assembling the output, and preview/create

---

## 1. The two containers (separate logic)

Every audience is two independent blocks, combined at the audience level with AND/OR:

- **Attributes** — profile-attribute conditions only. Describes *who the person is*.
- **Events** — experience-event conditions only. Describes *what the person did*.

Present them under separate headings so the split is visible. An event condition never appears in the
Attributes block, and a profile attribute never appears in the Events block. Because the two are
different containers, the separation holds structurally — it is not something to remember to do, it
is how the audience is shaped.

Row wording (matches the Segment Builder):
- Attribute row: `Include <fieldPath> <operator> <value>`
- Event row (positive): `Include at least <N> <Source> event where (<Field> equals <Value>)`
- Event row (negated): `Exclude any <Source> event where (<Field> equals <Value>)`

`<Source>` = `Any` for an "Any event" (field-matched) event, or the eventType for an
eventType-anchored event.

---

## 2. Identifying an event

An event is a **data source** plus optional **field conditions** (see `eventtypes.md` §3.5):

- **By eventType** — `<eventType>` as the source, e.g. an event where `eventType` =
  `commerce.checkouts`. Renders with the eventType as `<Source>`.
- **By field, using "Any event"** — source is `Any`, and a field identifies the event, e.g. page
  **Name** (`web.webPageDetails.name`) equals a value. Renders as
  `Any event where (Name equals "<value>")`. This is how "page view equals X" works.

Either way the condition lives in the **Events** block. Reproduce the user's labels/values verbatim;
confirm exact field paths against the sandbox for a live build.

---

## 3. Time — only when referenced

Add a time window to an event/sequence **only if the prompt referenced time**. No time reference →
no window, and don't prompt for one. When a window is present, **preserve the user's unit** — never
normalize hours to days. Render the window on the Events block as `and occurs in last <N> <unit>(s)`.

| Prompt says | Segment Builder wording |
|-------------|-------------------------|
| last 24 hours | `and occurs in last 24 hour(s)` |
| last 48 hours | `and occurs in last 48 hour(s)` |
| last 7 days | `and occurs in last 7 day(s)` |
| last 30 days | `and occurs in last 30 day(s)` |
| last month | `and occurs in last 1 month(s)` |
| today | `and occurs on Today` |
| (no time reference) | *(omit the "and occurs …" clause)* |

"24 hours" stays **hours** (`in last 24 hour(s)`), not "1 day".

---

## 4. Frequency

Frequency is how many times an event must have happened. Default is "at least 1". Wording:

| Intent | Wording |
|--------|---------|
| At least N times | `Include at least <N> … event where (…)` |
| Exactly N times | `Include exactly <N> … event where (…)` |
| At most N times | `Include at most <N> … event where (…)` |
| Between N and M | `Include between <N> and <M> … event where (…)` |

Only set a frequency the prompt asked for; otherwise use "at least 1".

---

## 5. Exclusion — only when negated, and targeted

**Create an exclusion only when the prompt uses negation** — *did not, didn't, does not, doesn't,
did not happen, did not occur, has not / hasn't, never, without, except, excluding, no [event]*. With
no negation, every condition is an include (even after "then"). Never infer an exclusion.

When one is warranted:
- It becomes an `Exclude any <Source> event where (…)` row, scoped to **only that one event** — never
  a blanket removal of the profile.
- It is its own row/step, independent of the include it pairs with. The same event may appear as both
  an include and an exclude (different windows) — that is fine and expected.

Example — "started checkout but did not purchase" → one include event row (checkout) plus one
`Exclude any` event row (purchase). "did not purchase" is what makes the purchase row an exclusion; a
prompt that just said "started checkout and purchased" would be two includes.

---

## 6. Sequences ("A then B")

Use a sequence when the prompt says "then" or order matters. Steps render in order inside the Events
block; each step after the first is prefixed `then within <gap> `:

```
Events
(Include audience who <step 0>
then within <gap> <step 1>
then within <gap> <step 2> …) and occurs in last <N> <unit>(s)
```

- **Gap:** the time allowed between consecutive steps. If the prompt states a gap ("then within 10
  minutes"), use it; otherwise apply the default **10 minutes** silently (a default is not a prompt).
  Make the default configurable via user preference.
- **Each step is include or exclude**, by the same rules as above: a step is `Exclude any …`
  **only if the prompt negated it**; otherwise `Include at least <N> …`.
- **Window:** the `and occurs in last …` clause applies to the whole sequence, and only if time was
  referenced.

Worked example — "page view equals 'Credit Card Application Started' then where page view equals
'Credit Card Application Submit' did not happen in the last 24 hours". Parse: "then" → sequence; "did
not happen" → Submit is an **Exclude** step; "24 hours" → `in last 24 hour(s)`; no gap stated →
default 10 minutes; no attribute in the prompt → Attributes block empty:

```
Events
(Include audience who Include at least 1 Any event where (Name equals Credit Card Application Started)
then within 10 minutes Exclude any Any event where (Name equals Credit Card Application Submit)) and occurs in last 24 hour(s)
```

If the prompt had **not** negated Submit, step 1 would be
`then within 10 minutes Include at least 1 Any event where (Name equals Credit Card Application Submit)`.
If it had **no** time reference, drop the `and occurs …` clause. If it also named an attribute (e.g.
`creditCardUpgradePrediction = 1`), that becomes a separate `Include creditCardUpgradePrediction
equals 1` row in the **Attributes** block — it does not go inside the Events sequence.

---

## 7. Assembling the output, and preview/create

The output to show the user is the two blocks:

```
Attributes
  Include <fieldPath> <operator> <value>
  …
Events
  (Include audience who … ) [and occurs in last <N> <unit>(s)]
```

Keep them under distinct headings; an empty block is simply omitted. Run the Step 4 checklist in
`SKILL.md` before doing anything with the result.

**Preview / create (when a capability is available).** The RTCDP MCP tools
(`preview_audience_membership`, and `create_audience` where present) take the audience **definition**,
not the natural-language wording. Do **not** hand-fabricate the definition internals from memory:

1. Fetch an existing audience of a similar shape (`search_audiences`) and read its stored definition
   to learn the exact structure this sandbox uses.
2. Build the new definition by mirroring that structure — dropping in your validated Attributes rows
   and Events rows from the spec.
3. Submit to `preview_audience_membership` (submit → `previewId`; then fetch with the same
   `previewId` until `state = RESULT_READY`) to validate it and get a size estimate. Never treat an
   interim `NEW`/`0` as the real size.
4. Save with `create_audience` if present; otherwise deliver the two-block output + a Segment Builder
   click-path (see SKILL.md "Graceful degradation").

**Author it UI-editable (attribute-based composition).** So a marketer can reopen and edit the
audience in the Segment Builder:

- Set **`"compositionType": "attribute"`**. **Never** use `"compositionType": "pql/text"` with a raw
  PQL string — that produces a read-only, code-based segment that cannot be edited visually.
- **Pre-resolve the full schema path** for every field, starting from `_xdm.context.profile` (and the
  ExperienceEvent union for event fields). Attribute composition needs real, fully-qualified
  references; confirm each against the sandbox rather than guessing.
- **Build the expression payload as a structured attribute tree**: one node per condition carrying its
  `type`, `value`, and `operator`, combined by group nodes, mirroring the two separate blocks
  (Attributes logic and Events logic). Learn the exact node shape from the fetched attribute-composed
  audience in step 1 — do not invent it.
- **Cap authoring at 2 attempts.** If it still won't validate after a second try, **save as a draft**
  and report the validation error plus the two-block output; do not keep iterating.

**Relational safety:** a relational / cross-entity rule that also references a person experience event
is refused by the platform. If you hit that, put the event condition in its own audience and reference
it from the other via a segment-of-segments. Plain profile audiences that combine attributes and
experience events are fine as one definition.

If you cannot fetch a template and are unsure of the definition internals, deliver the reviewed
two-block output + the click-path rather than a guessed definition. Correct and buildable beats fast
and wrong.
