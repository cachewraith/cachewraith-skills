---
name: design-ui
description: "Design a UI mockup — a page, screen, flow, component, or whole app — and publish it as a Claude Artifact, working the way a human product designer does: brief first, real content before layout, a deliberate system for type, color, and spacing, every state designed, then a critique pass before it ships. Always loads the ui-ux-pro-max skill (nextlevelbuilder/ui-ux-pro-max-skill) first and follows its priority rules and design-system search. Invoke when the user asks to design, mock up, prototype, wireframe, or redesign a UI, or asks for a design in Claude Design or as an artifact. Skip when they want an existing UI's code fixed or wired up in their own repo with no mockup, or want a chart or dashboard of real data (that is dataviz)."
---

# Design UI

Produce a mockup a designer would sign off on, published as an Artifact. Two sources
carry this skill and neither is optional:

1. **ui-ux-pro-max** — the rules and design-system data. Read it before designing anything.
2. **The human process below** — the order a working designer makes decisions in. A
   mockup that skips steps looks generated, however polished the pixels are.

## 1. Load ui-ux-pro-max first

Before the brief, before any layout, get the upstream skill into context. In order:

1. **Installed?** Look for it — it may be a plugin or a plain skill:
   ```bash
   find ~/.claude ~/.claude-accounts .claude -path '*ui-ux-pro-max*' -name SKILL.md 2>/dev/null | head -3
   ```
   If found, read that `SKILL.md`, and note the directory: its `scripts/search.py` and
   `references/` live beside it.
2. **Not installed** — fetch it and read it in full:
   ```
   https://raw.githubusercontent.com/nextlevelbuilder/ui-ux-pro-max-skill/main/.claude/skills/ui-ux-pro-max/SKILL.md
   https://raw.githubusercontent.com/nextlevelbuilder/ui-ux-pro-max-skill/main/.claude/skills/ui-ux-pro-max/references/quick-reference.md
   ```
   Without the local data the search script cannot run. Tell the user once, in one line,
   that the design-system search is unavailable and they can install it with
   `/plugin marketplace add nextlevelbuilder/ui-ux-pro-max-skill` then
   `/plugin install ui-ux-pro-max@ui-ux-pro-max-skill` — then carry on using its priority
   table and quick-reference rules, and label choices from them as built-in defaults.

When the script is available, use it the way its own skill says:

- New page, screen, or product → `search.py "<product> <industry> <keywords>" --design-system -p "<Name>"`.
  Add `--density 8` for dashboards and admin tools, `--density 2-3` for marketing pages.
- One specific concern (a modal's focus, a form's errors) → one `--domain` query, 2–5 terms.
- Retry once on an empty or off-topic result, then fall back and say so. Never present a
  zero-result search as data.
- If `design-system/<slug>/MASTER.md` already exists in the project, read it and design
  inside it. Never `--persist --force` without the user saying so.

Its search output is a recommendation. The user's request and the project's existing
design outrank it.

## 2. The brief

Designers do not open a canvas until they can say what the screen is for. Pull these out
of the request and the project; do not interview the user for what you can read:

| Question | Where it usually comes from |
|---|---|
| Who uses it, and in what context (desk, phone on a commute, a kiosk) | The request; the product's domain |
| The **one primary task** on this screen | The request — if there are three, the screen has a hierarchy problem |
| Product type and tone (SaaS tool, storefront, portfolio, internal admin) | The request; the repo's README |
| Platform and breakpoints (web, iOS, Android, desktop) | The request; the stack in `package.json`, `pubspec.yaml`, etc. |
| An existing brand or design system to honor | Tailwind config, CSS variables, a `design-system/` folder, a Figma link |

Ask **one** question, and only when a wrong guess would waste the mockup — usually the
primary task or the platform. Everything else, pick a sensible default and state it in the
reply.

## 3. How a human designs it

Work in this order. Each step constrains the next; doing them out of order is what makes a
screen feel assembled rather than designed.

**a. Content before layout.** Write the real copy and data first: actual headings, actual
product names, plausible numbers, a user named something other than John Doe. Lorem ipsum
hides every length and hierarchy problem the real content would expose.

**b. Structure in grey.** Place blocks by importance, not by habit. Put the primary task
where the eye lands first — top-left for scanning layouts (F-pattern), center for a
single-action page. Headings start with the information-bearing word. Choose the familiar
pattern for the job (Jakob's Law): users know how a settings page, a checkout, and a data
table work, and novelty there costs them.

**c. Hierarchy with as few tools as possible.** Importance comes from size, weight, and
contrast — try weight and color before reaching for size.
- At most three text sizes per region, and at most two elements that are "large".
- De-emphasize secondary content (lighter grey, smaller, regular weight) instead of
  shouting the primary. Labels are a last resort: `12 left in stock` beats `Stock: 12`, and
  where a label stays, it is quieter than its value.
- One primary action per view. Secondary actions are outlined or text buttons; destructive
  ones are not red until the moment of confirmation.

**d. Spacing and grouping.** A spacing scale, not ad-hoc numbers: 4 / 8 / 12 / 16 / 24 /
32 / 48 / 64. Space *between* groups is clearly larger than space *within* them — that gap
does the grouping, so borders and cards are needed less often than you think. Start with
too much whitespace and remove it.

**e. The system.** Define before styling, as CSS custom properties:
- **Type** — one family (two at most: display + text). A modular scale, 16px body,
  line-height ~1.5 for text and ~1.2 for headings, line length 45–75 characters.
- **Color** — 8–10 greys (no pure black; start from a very dark grey), one primary with a
  full 100–900 ramp, and semantic colors (success, warning, danger, info) with their own
  tints. Hand-pick the shades; don't generate them with `lighten()`. Text meets 4.5:1
  contrast (3:1 for large text) in both light and dark themes.
- **Shape and depth** — one radius scale, one shadow scale that means elevation, not
  decoration.

**f. Every state, not just the happy path.** A designer hands off the screen in all of its
lives. For each data region design: empty (what goes here, and the button that fills it),
loading (skeleton that reserves the space), error (what went wrong, why, and what to do —
in plain language), partial, and overflow (the 40-character name, the 10,000-row table).
For each control: default, hover, focus, active, disabled.

**g. Responsive.** Design the narrowest breakpoint as a real layout, not a squeezed desktop.
Touch targets at least 44×44px with 8px between them. No horizontal page scroll.

## 4. Build the Artifact

- Call `Artifact` with `action: "quickstart"` and `intent: "design"` first — it names the
  artifact type and design systems this account has. Follow what it returns; if the user
  already has a design system there, use it instead of inventing one.
- Load `artifact-design` when the quickstart result tells you to, and follow its page
  contract (tokens on `:root`, dark-mode overrides, allowed CDNs, 16px mobile gutter).
- Make it clickable where interaction is the point: tabs switch, modals open, the form
  validates inline. A static picture of an interactive thing hides its hardest decisions.
- When showing states or breakpoints, lay them side by side on the canvas, labelled —
  "Empty", "Loading", "Error", "Mobile 375" — so the reviewer compares instead of hunts.
- SVG icons from one set (Lucide, Heroicons, Phosphor). Never emoji as icons.

## 5. Critique before publishing

Designers review their own work before anyone else sees it. Run this pass and fix what
fails — do not ship a list of known problems.

1. **Squint test.** Blur your eyes (or imagine a 10px blur): is the primary task still the
   first thing you see? Are the groups still visible as groups?
2. **ui-ux-pro-max priorities 1→10** — accessibility, touch, performance, style fit,
   responsive, type and color, motion, forms, navigation, charts. With the skill installed,
   run through its `references/pro-rules.md` pre-delivery checklist for app UI.
3. **Nielsen's heuristics**, the ones mockups fail most: system status visible, an exit
   from every modal and flow, consistent labels for the same action, errors prevented
   before they are reported, options shown rather than remembered.
4. **Copy.** Buttons are 2–4 words and start with a verb that names the result
   (`Save changes`, `Delete folder`) — never `OK` or `Submit`. Same action, same word,
   everywhere.
5. **Anti-generic check.** If the screen could belong to any product — centered hero,
   purple gradient, three identical feature cards — it has not used the brief. Change
   something that only this product would have.

## 6. Reply

Publish, then keep the reply short:

- The artifact link.
- One or two lines on the direction: the style and palette chosen and *why* for this
  product, and whether it came from a ui-ux-pro-max design-system match or from its
  built-in defaults.
- Any assumption you made in the brief that the user may want to change.
- At most one line on what you would design next (the next screen in the flow, a dark
  theme pass). No summary of what the mockup already shows — they can see it.
