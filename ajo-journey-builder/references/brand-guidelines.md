# Brand Guidelines (applied to all creatives)

These guidelines are applied whenever the skill **creates or updates** a creative (email, SMS, push,
code-based). They exist so every journey looks and sounds consistent.

> **Customize this file for your brand.** The values below are placeholders with sensible defaults.
> Replace them with your real brand standards before relying on the skill for production content. If a
> value here conflicts with something the user says in the prompt, follow the user and note the
> deviation.

---

## Voice and tone
- **Voice:** <e.g. warm, confident, plain-spoken; never hype-y or jargon-heavy>.
- **Tone by context:** welcome = friendly; transactional = clear and reassuring; win-back = warm, low-
  pressure; promo = energetic but honest.
- **Reading level:** aim for ~grade 7–8; short sentences; active voice.
- **Person:** address the reader as "you"; use the brand as "we".

## Naming and sender identity
- **Sender/brand name:** <Brand Name> (match the channel configuration's sender name).
- **Journey naming convention:** `<Program> - <Intent> (<channel/cadence>)`, e.g.
  `Onboarding - Welcome (email)`. Keep names human-readable and consistent.

## Logo and imagery
- **Logo:** <where the logo lives / fragment name>; clear space = logo height on all sides; never
  recolor or distort.
- **Imagery:** <photography vs illustration style>; use approved assets only; always set descriptive
  **alt text**; avoid text baked into images.

## Color palette
- **Primary:** <#HEX> · **Secondary:** <#HEX> · **Accent/CTA:** <#HEX>
- **Text:** <#HEX on light> ; **Background:** <#HEX>
- Maintain **WCAG AA** contrast (≥ 4.5:1 for body text).

## Typography
- **Headings:** <font, weight, size>. **Body:** <font, size ~14–16px, line-height ~1.5>.
- Provide web-safe fallbacks; don't rely on fonts that won't render in email clients.

## Email structure (default layout)
1. Preheader (hidden preview text) — set it intentionally, don't leave it blank.
2. Header with logo.
3. Hero / headline.
4. Body (one clear idea; scannable).
5. **Primary CTA button** (see CTA rules).
6. Secondary content / fragments as needed.
7. **Footer:** brand address, preference center link, and — for **Marketing** emails — a required
   **opt-out / unsubscribe link** (transactional emails don't require it).

## Subject line and preview text
- **Subject:** <= ~50 characters; front-load the value; no ALL CAPS; emoji only if on-brand.
- **Preview text:** <= ~90 characters; complements (doesn't repeat) the subject.

## CTA rules
- One primary CTA per message; verb-first label (e.g. "Start now", "See your offer").
- Button uses the Accent/CTA color with AA-contrast text; descriptive link text (never "click here").

## Personalization
- Use real profile/event fields; always define a **fallback/default** (e.g. "there" for first name).
- Never expose raw field names or empty tokens in the rendered message.

## SMS / push specifics
- **SMS:** <= 160 chars where possible; include brand identifier and, for marketing, opt-out
  instructions (e.g. "Reply STOP"); use a link shortener consistent with brand.
- **Push:** title <= ~40 chars, body <= ~120 chars; clear value; deep-link to the right place.

## Compliance
- Marketing messages require consent and an opt-out; honor channel-configuration category
  (Marketing vs Transactional).
- Include required legal/footer text for the region.

---

When creating or updating a creative, apply the relevant sections above, then show the user the result
and ask if they'd like to update it further.
