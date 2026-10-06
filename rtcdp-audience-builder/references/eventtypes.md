# EventTypes and Schema Sourcing

This file governs the single most important accuracy rule of the skill:

> **Experience events come from the ExperienceEvent union schema and are keyed by `eventType`.
> Profile attributes come from the Individual Profile union schema.
> The two must never be mixed inside one predicate.**

Read this whenever you need to classify a condition, pick an `eventType`, or explain to the
user why something belongs under Events vs Attributes.

---

## 1. The two union schemas

Real-Time Customer Profile assembles data into two "union" views per sandbox. A union schema is
the merged view of every schema that shares the same class.

| Union | XDM class | What lives here | Where it goes in an audience |
|-------|-----------|-----------------|------------------------------|
| **Individual Profile** union | `_xdm.context.profile` | Record data / **profile attributes** — traits that describe *who the person is*: name, email, address, loyalty tier, consent, computed attributes, audience memberships. Stable, one current value. | The **Attributes** container (Attributes logic). |
| **ExperienceEvent** union | `_xdm.context.experienceevent` | Time-series **events** — facts about *what the person did*, each stamped with `timestamp` and `eventType`: page views, cart adds, checkouts, purchases, app opens, email opens. Many rows per person. | The **Events** container (Events logic, time-scoped when the prompt references time). |

**Decision rule for every condition the user gives you:**

- Does it describe a *standing trait*? → profile **attribute** → Attributes container.
- Does it describe *an action that happened at a point in time*? → **event** → Events container,
  identified by its `eventType`.

If a phrase contains a verb in the past tense tied to a time ("bought", "viewed", "added",
"opened last week"), it is almost always an event. If it is a noun/adjective about the person
("is a Gold member", "lives in Canada", "has opted in"), it is an attribute.

---

## 2. `eventType` is how events are told apart

Within the ExperienceEvent union, every row carries `eventType` (path: `eventType`, sometimes
shown as `xdm.eventType`). It is an **extensible enum**: Adobe ships standard values, and an
organization may add its own custom values on top. Because it is extensible, **never assume the
user's `eventType` string** — confirm it against the sandbox (see §4) before writing the final
rule. Do, however, propose the standard value that matches the user's intent so they can confirm
fast.

Every event condition in an audience is anchored on `eventType` — e.g. an event where
`eventType = "commerce.checkouts"`. The rest of the event condition (time window, count, other event
fields) hangs off that anchor.

---

## 3. Standard `eventType` values (starting catalog)

These are Adobe standard values. Treat them as the defaults you propose; the sandbox may also
have custom ones.

### Commerce (the cart / purchase funnel)

| eventType | Meaning | Common funnel role |
|-----------|---------|--------------------|
| `commerce.productViews` | Viewed a product | Interest |
| `commerce.productListOpens` | Opened / created a cart | Cart start |
| `commerce.productListAdds` | **Added an item to cart** | Add-to-cart |
| `commerce.productListViews` | Viewed the cart | Cart review |
| `commerce.productListRemovals` | Removed an item from cart | Drop-off signal |
| `commerce.saveForLaters` | Saved item for later | Intent, deferred |
| `commerce.checkouts` | **Started / progressed checkout** | Checkout |
| `commerce.purchases` | **Completed a purchase / order** | Conversion |
| `commerce.productListReorders` | Reordered a prior cart | Repeat intent |

### Web

| eventType | Meaning |
|-----------|---------|
| `web.webpagedetails.pageViews` | Page view |
| `web.webinteraction.linkClicks` | Link/click interaction |

### Application (mobile/app)

| eventType | Meaning |
|-----------|---------|
| `application.launches` | App launched |
| `application.close` | App closed |
| `application.crashes` | App crashed |
| `application.featureUsages` | Feature used |

### Messaging / delivery (when Journey Optimizer / Campaign data is present)

| eventType | Meaning |
|-----------|---------|
| `directMarketing.emailOpened` | Email opened |
| `directMarketing.emailClicked` | Email link clicked |
| `directMarketing.emailSent` | Email sent |
| `decisioning.propositionDisplay` | Offer/proposition shown |
| `decisioning.propositionInteract` | Offer/proposition interacted with |

> This list is a convenience, not the source of truth. The source of truth is the sandbox's
> ExperienceEvent union schema. When in doubt, confirm (see §4).

---

## 3.5 Identifying events by a field, not by eventType ("Any event")

Users often name an action by a **page name or other event attribute** rather than by an eventType.
In the Segment Builder this is the **"Any event"** data source plus a field condition, and it
renders as `Any event where (<Field> <op> <value>)`. This is still an **event** (Events container,
time-scoped only when time is referenced) — it is not an attribute.

The canonical case is a **page view**: "page view equals X" means an event whose page-name field
equals X.

| User says | Data source | Field (display label) | Field path | Renders as |
|-----------|-------------|-----------------------|-----------|------------|
| "page view equals X" / "viewed page X" | Any event | **Name** | `web.webPageDetails.name` | `Any event where (Name equals "X")` |
| "clicked link X" | Any event | Link Name | `web.webInteraction.name` | `Any event where (Link Name equals "X")` |
| "screen/state X" (app) | Any event | Name | `application._id`/screen name field | `Any event where (Name equals "X")` |

Guidance:
- Use **"Any event" + field match** when the user identifies the action by a name/label (their
  example: *"page view equals 'Credit Card Application Started'"*). Reproduce their label verbatim as
  the value.
- Use **eventType** (mode A, §3) when the behavior maps cleanly to a standard/custom eventType and
  you want the stricter match.
- You *can* combine both (e.g. eventType `web.webpagedetails.pageViews` AND `Name equals "X"`) for
  precision, but match the user's expected output — if they expect `Any event`, render `Any event`.
- Confirm the field's display label and path against the sandbox when building for real (§4); "Name"
  is the common label for `web.webPageDetails.name`.

## 3.6 Predicted / computed attributes are profile attributes

Propensity/prediction scores and computed attributes (e.g. `creditCardUpgradePrediction = 1`,
`churnScore > 0.8`) describe the *person*, live on the Individual Profile union, and belong in the
**Attributes** container — even though they are model outputs. They render as
`Include <fieldPath> <operator> <value>` in the Attributes block, and carry no event.

---

## 4. Confirming values against the live sandbox (when building for real)

When you are actually creating an audience in a connected org/sandbox (not just drafting), do not
rely on memory for the exact `eventType` strings or profile field paths. Confirm them, because a
custom deployment may rename or extend things:

1. **Learn the shape from a known-good audience.** Fetch an existing audience of a similar type
   (an event-based one for events, an attribute-based one for attributes) and read how its
   definition references `eventType` and profile fields. Mirror that exact structure and casing.
   This is the most reliable way to stay correct across differently configured sandboxes, and it
   is what makes the output *repeatable*.
2. **Prefer standard values** from §3 when the user describes a standard behavior, then state the
   value you chose so the user can correct it in one step ("I'll use `commerce.purchases` for
   'bought' — say the word if your implementation uses a custom eventType").
3. **Custom eventTypes are legal.** If the user says their purchase event is, e.g.,
   `commerce.orderComplete`, use it verbatim. `eventType` being an extensible enum means custom
   strings are valid; do not "correct" a custom value to a standard one.

---

## 5. The separate-logic invariant (do not violate)

- An **attribute condition** lives in the **Attributes** block and names only Individual Profile
  fields — no event, no `eventType`, no event field. Example (correct):
  `Include homeAddress.countryCode equals US`.
- An **event condition** lives in the **Events** block, is identified by an `eventType` or an event
  field ("Any event where …"), and is time-scoped only when the prompt references time. Example
  (correct): `Include at least 1 commerce.purchases event`.
- **Never** put an event condition in the Attributes block, and never put a profile attribute in the
  Events block. If a single idea seems to need both, split it into one attribute row and one event
  row, joined at the audience level — the two blocks stay independent.

This is the invariant the user cares about most: *profile attributes stay in the Attributes logic and
events stay in the Events logic — the two are never mixed.* The validation checklist in `SKILL.md`
enforces it before anything is previewed or saved.
