# Audience Patterns (generic)

Reusable, business-agnostic patterns showing how each shape renders as an audience with the
**Attributes logic and Events logic kept separate**. Substitute your own attributes, events, values,
windows, and counts. Placeholders:

- `<Attribute>` — a profile field path (Individual Profile), e.g. a trait or a predicted score.
- `<Event>`, `<Event A>`, `<Event B>` — an event, identified by an `<eventType>` **or** by
  `Any event where (<Field> equals <Value>)` (see `eventtypes.md` §3.5).
- `<N>` frequency (default "at least 1"), `<window>`/`<unit>` a time window, `<gap>` a sequence gap.

**Two rules gate every pattern (never violate):**
- **Time only when referenced** — include a window (`and occurs in last <N> <unit>(s)`) only if the
  prompt mentioned time; preserve the unit ("24 hours" → hours). No time reference → no window, no
  prompt.
- **Exclusion only when negated** — an `Exclude any …` row/step appears only if the prompt used
  negation (did not / didn't / never / without / did not happen / did not occur …).

Each pattern below shows the **two-block output** and the **Segment Builder wording**. Strip the
window or the exclusion when a given prompt doesn't call for it.

---

## Pattern 1 — Attribute only

*"People whose `<Attribute>` `<operator>` `<value>`."* No events at all.

Attributes:
- `Include <Attribute> <operator> <value>`

Events: *(none)*

```
Include <Attribute> <operator> <value>
```

---

## Pattern 2 — Single event

*"People who did `<Event>`"* (optionally "…in `<window>`", optionally "…at least `<N>` times").

Attributes: *(none)*

Events:
- `Include at least <N> <Source> event where (<Field> equals <Value>)`  [+ window if referenced]

```
Events
(Include audience who Include at least <N> <Source> event where (<Field> equals <Value>)) and occurs in last <N> <unit>(s)
```
No time in the prompt → drop `and occurs …`. `<Source>` is the eventType, or `Any` for a
field-matched event.

---

## Pattern 3 — Attribute + event (separate blocks)

*"People whose `<Attribute>` `<operator>` `<value>` **and** who did `<Event>` [in `<window>`]."*
The attribute and the event stay in different blocks, joined with AND.

Attributes:
- `Include <Attribute> <operator> <value>`

Events:
- `Include at least <N> <Source> event where (<Field> equals <Value>)`  [+ window if referenced]

```
Include <Attribute> <operator> <value>
Events
(Include audience who Include at least <N> <Source> event where (<Field> equals <Value>)) and occurs in last <N> <unit>(s)
```
The attribute never moves into the Events block, and the event never moves into the Attributes block.

---

## Pattern 4 — Event include + event exclude (set logic)

*"People who did `<Event A>` **but did not** do `<Event B>` [in `<window>`]."* Unordered; the
exclusion exists **because the prompt negated `<Event B>`**.

Events:
- `Include at least 1 <Source> event where (<Field A> equals <Value A>)`
- `Exclude any <Source> event where (<Field B> equals <Value B>)`

```
Events
(Include audience who Include at least 1 <Source> event where (<Field A> equals <Value A>)
Exclude any <Source> event where (<Field B> equals <Value B>)) and occurs in last <N> <unit>(s)
```
The exclude targets `<Event B>` only. Remove it entirely if the prompt had no negation.

---

## Pattern 5 — Sequence, both steps included ("A then B")

*"People who did `<Event A>` **then** did `<Event B>`."* Order matters (prompt says "then"), no
negation → both steps are includes.

```
Events
(Include audience who Include at least 1 <Source> event where (<Field A> equals <Value A>)
then within <gap> Include at least 1 <Source> event where (<Field B> equals <Value B>)) and occurs in last <N> <unit>(s)
```
`<gap>` defaults to 10 minutes unless the prompt states one. Drop `and occurs …` if no time was
referenced.

---

## Pattern 6 — Sequence with a negated step ("A then B did not happen")

*"People who did `<Event A>` **then** `<Event B>` **did not happen** [in `<window>`]."* The
funnel-drop shape. "then" → sequence; "did not happen" → `<Event B>` is an **Exclude** step.

```
Events
(Include audience who Include at least 1 <Source> event where (<Field A> equals <Value A>)
then within <gap> Exclude any <Source> event where (<Field B> equals <Value B>)) and occurs in last <N> <unit>(s)
```
If the prompt also named an attribute, it renders as a separate `Include <Attribute> …` row in the
Attributes block — it stays out of the sequence.

---

## Naming convention (for repeatability)

Propose names that encode intent so audiences are self-documenting and re-creatable:

```
<intent> - <key attributes/events> (<window if any>)
```

Always ask the user to confirm or override the name — it is a required input, never auto-final.
