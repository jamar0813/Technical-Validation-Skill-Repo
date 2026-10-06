# Journey Recipes (common patterns)

Reusable journey blueprints for consistency. Each shows the **entry**, the **flow**, and the **action
+ creative** decisions. Everything named — audiences, actions, channel configurations, templates — is a
**placeholder to resolve** from the live AJO configuration or confirm with the user (see
`journey-concepts.md` golden rule). Email sends default to the **"marketing"** channel configuration.

Notation: `[Entry]` → `{Orchestration}` → `((Action))`. "Ask" marks a required user decision.

---

## Recipe 1 — Welcome / onboarding

*New subscriber gets a welcome series.*

- `[Unitary event: subscription/sign-up]` (resolve the event; ask if unsure)
- → `((Email — welcome))` using the **marketing** email configuration
  - Creative: **Ask** create vs template → author/select per brand guidelines → **Ask** update?
- → `{Wait 3 days}`
- → `{Condition: opened/clicked?}`
  - Yes → `((Email — getting-started tips))`
  - No → `((Email — reminder))`
- → Exit

---

## Recipe 2 — Cart abandonment recovery

*Abandoned checkout, no purchase.*

- `[Unitary event: cart abandoned / checkout started]` (resolve; ask if unsure)
- → `{Wait 1 hour}`
- → `{Condition: purchased since entry?}`
  - Purchased → Exit
  - Not purchased → `((Email — cart reminder))` (marketing config)
    - Creative: **Ask** create vs template → per brand guidelines → **Ask** update?
- → `{Wait 1 day}` → `{Condition: purchased?}` → No → `((SMS — last-chance))` (resolve SMS config)
- → Exit

---

## Recipe 3 — Browse abandonment

*Viewed products, didn't add to cart.*

- `[Unitary event: product viewed]` (resolve) → `{Wait}` → `{Condition: added to cart?}`
  - No → `((Email — "still interested?"))` (marketing config; creative Ask create/template + Ask update)
- → Exit

---

## Recipe 4 — Win-back / re-engagement (audience entry)

*Lapsed customers pulled from an audience on a schedule.*

- `[Read Audience: lapsed customers]` (resolve the audience; ask which if unsure; recurring schedule)
- → `((Email — we miss you + offer))` (marketing config; creative Ask create/template + Ask update)
- → `{Wait 5 days}` → `{Condition: engaged?}` → No → `((Push or SMS — final offer))` (resolve config)
- → Exit

---

## Recipe 5 — Audience qualification promo

*React the moment someone qualifies for a segment.*

- `[Audience Qualification: entrance into audience X]` (resolve audience; confirm entrance/exit/both)
- → `((Email — targeted offer))` (marketing config; creative Ask create/template + Ask update)
- → Exit

---

## Recipe 6 — Post-purchase nurture

*Deepen the relationship after a purchase.*

- `[Unitary event: purchase]` (resolve)
- → `((Email — thank you))` (marketing config)  ·  creative Ask create/template + Ask update
- → `{Wait 7 days}` → `((Email — how-to / cross-sell))`
- → Exit

---

## Recipe 7 — Transactional confirmation

*Order/booking/password confirmation.*

- `[Unitary event: order placed / password reset]` (resolve)
- → `((Email — confirmation))` using a **Transactional** channel configuration (NOT the marketing
  default — confirm the transactional config; transactional emails don't require an opt-out link)
  - Creative: usually a **template**; still **Ask** create vs template, then **Ask** update?
- → Exit

---

## Recipe 8 — Action-first framing (resolve the entry)

*Prompt says "build a journey that sends the loyalty-points update."*

- **Resolve the action** ("loyalty-points update") against available actions; ask if no match.
- **Determine the entry**: ask whether it should trigger on an event (e.g. points changed) or read an
  audience — then wire `[Entry]` → `(( resolved action ))`.
- Creative (if it's a message action): Ask create vs template → per brand guidelines → Ask update?
- → Exit

---

## Consistency checklist for every recipe

- Entry resolves to a real event/audience (or asked).
- Every action resolves to a real channel/custom action (or asked); email uses the **marketing**
  configuration unless it's transactional.
- Creative source decided **with the user** (create vs template) and offered an update pass.
- Brand guidelines applied; marketing emails include an opt-out link.
- Waits/conditions make the timing and branching explicit.
- Journey has a clear name and clean exits.
