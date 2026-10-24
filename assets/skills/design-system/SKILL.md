---
name: design-system
description: Create or extend a product's design system for its browser extension — DESIGN.md at the project root, the source of truth for how every surface looks, moves and reads in the popup, the side panel, the options page and on the pages its content UI joins, in the DESIGN.md format. Use once per product before the first screens are designed, after reading the product docs and researching the best extensions in its niche, and again when a later phase needs a foundation the system does not cover.
---

# Design System

You are a senior product designer and design engineer writing the design system for this product's browser extension. The result is `DESIGN.md` at the project root: one document that makes every surface, built by anyone in any session, look like it belongs to the same product, and makes that product look like itself rather than like every other AI-generated extension — including when its UI sits on a page it did not design.

It follows the DESIGN.md format (`github.com/google-labs-code/design.md`): design tokens in YAML front matter, and markdown sections that explain how to apply them. `/extension-build` reads it before every screen is designed, design tools find it at the root, and its tokens become the CSS variables the code uses. So it is precise where precision matters (values, names, states) and plain everywhere else.

## Inputs

Read these first; ask for anything missing.

| Input | What it gives you |
| --- | --- |
| Product brief and PRD, `docs/product-docs/` | Who the product is for, the problem, the positioning, the tone it should carry |
| `docs/feature-areas.md` | The areas: how many, which are data-dense, which are transactional, which live in the popup and which need the side panel or a page |
| Business rules and app flows, `docs/business-rules/`, `docs/app-flows/` | The states screens need (pending, failed, expired), the data types that recur (money, dates, statuses), the roles and how different their worlds are |
| `CLAUDE.md` | The audience this repo serves, whether a second audience shares the brand, the browsers, the surfaces the product has, the single purpose, and the target sites whose pages the content UI must sit on |
| Brand assets, if any | The logo system from the Ideas Blueprint, brand colors, a name, a tagline. The logo's color and geometry are inputs to the palette and shape language, not the other way round |
| Another client's `DESIGN.md` for the same product, if one exists (at the sibling client repo path in `CLAUDE.md`) | The brand decisions — palette, type, voice, shape — are taken from it so the product looks like itself on every client; the extension's surfaces, content UI and motion are this system's own |
| An existing `DESIGN.md` | In a later phase: extend it, never regenerate it |

If a frontend-design skill is installed in this Claude Code, read it as well; it sharpens the aesthetic judgement this skill asks for.

## Step 1 — Understand the product

Before any color is picked, write down, in a few lines each:

- **What kind of extension this is**, because it sets where the UI lives and how loud it may be. Most extensions are the first of these; say which applies where.
  - *A companion to the page* (a clipper, an assistant, an annotator, a price or data overlay): its content UI joins pages it did not design, so the system must be unmistakably the product's and still polite on anyone's page — compact, quiet, anchored, never covering what the user came for. The popup is a glance; the side panel is where work happens.
  - *A tool in its own window* (a dashboard, a manager, a workspace that happens to be an extension): the side panel and the full-tab page carry it, so the system leans compact and keyboard-first like a desktop tool, and the content UI is small or absent.
  - *A utility used in moments* (a switcher, a timer, a quick action): the popup is the product. Clear state, one decision per screen, fast.
- **Who is looking at it, and in what state.** Experts or first-timers; mid-task on another site, so attention is borrowed; in a 380-pixel popup or a side panel beside a page; at 100% or 200% zoom. This decides density, type size, and how much the interface explains itself.
- **What the brand should feel like**, in three or four plain adjectives taken from the brief, and what it must not feel like.
- **The recurring content**: the data types that appear everywhere (amounts, dates, statuses, names, avatars, long text), because the system has to make those consistent first.
- **The constraints**: accessibility level (WCAG AA is the floor, with VoiceOver, Narrator and Orca), dark mode (yes, following the OS, unless the brief says otherwise), right-to-left or a second language, brand colors that are fixed, and **browser conventions**: the system follows each browser's chrome where it must (the popup's frame, the side panel's header, the install prompt, the shortcut modifiers) and is the product's own everywhere else. Say which conventions are kept per browser and which are unified.

## Step 2 — Research the references

The goal is to learn what the best products in this space do well and why, then do it our own way. References are inspiration, never a source of truth and never something to copy.

**On Mobbin** (through its MCP, `platform: web` — extensions are catalogued with the web, and the best web products' compact components are the closest relatives of a popup and a side panel — plus the Chrome Web Store's and Firefox's design guidance for the chrome; follow the Mobbin search skill if it is installed):

1. Search flows and screens for **three to five direct competitors or neighbours** named in the brief, and **two or three category leaders** whose interface is widely admired, in the product's category or a close one (a dense tool learns from the best dense tools, whatever their domain).
2. For each, look at the actual screens — the images, not the metadata — of the surfaces this product will have: the popup's first screen, the side panel's main list and detail, a form, the options page, the content UI on a real page (how it anchors, how it stays out of the way), an empty state, the signed-out and permission-needed states.
3. Record, per reference, what makes it work: how it builds hierarchy, how much color it uses and where, its density and spacing rhythm, its type choices, how it handles states and feedback, what its components have in common, what its copy sounds like, and what it does that is distinctive. Also what it does badly, so we avoid it.

**Outside Mobbin**, where it helps: the products' own marketing sites. Where the direction draws on a product documented in the corpus at `github.com/VoltAgent/awesome-design-md`, that product's DESIGN.md shows its exact decisions; learn from it, never copy. Cite everything you looked at.

**Then extract**: a short list of the principles the strong references share, and the two or three moves that would make this product distinctive rather than generic. That list is what the system is built from.

## Step 3 — Propose the direction → **wait for my approval**

Present one direction, with the reasoning from Steps 1 and 2, and one alternative where the choice is genuinely open. Include:

- **Principles** — four to six, each one sentence, each with a consequence a builder could act on ("Density over decoration: tables default to compact rows, and whitespace is spent on grouping, not on margins").
- **Palette** — the brand color and why, the neutral scale and its temperature, the semantic colors, the accent policy (how much color a screen may carry), with hex values and the contrast ratios of text on surface, light and dark.
- **Typography** — the families, the scale, the weights in use, and the reasoning. Free fonts only, two families at most: Google Fonts by default, or another open-source font (SIL OFL or similar). Never a font that must be bought, or one licensed for personal use only.
- **Shape and depth** — the radius scale and the elevation approach (borders, shadows, or surface steps).
- **Density and layout** — the base unit, the popup's width and how it grows, the side panel's minimum, the content width on the options and full-tab pages, and each surface's shell (header, navigation, where the account entry lives). The shells are designed and built in `/extension-build`'s Phase 0 from this decision.
- **Content UI stance** — how the product's UI sits on another page: anchored or floating, how big, how it announces itself, what it never covers, and how it keeps its identity when the page's palette is anything.
- **Keyboard stance** — which actions carry browser commands, how shortcuts are shown, Escape's behaviour in each surface.
- **Motion stance** — how much, how fast, and what never moves.
- **Imagery and icon stance** — illustration, photography, neither; the icon set; the tone of imagery.
- **Voice** — how the product talks, with three example sentences.

Show it: a style tile rendered on the `/design` canvas where available, otherwise an HTML page in `docs/design/style-tile.html`, with the popup at its width and a side-panel column side by side, the palette, the type scale, three buttons in their states, an input, a list row, a card, the header, and one content-UI piece shown on a plain page and on a busy one, light and dark. Judge it with your own eyes before showing it: if it looks like a template, change it before I see it.

**What generic looks like, so you can avoid it:** a blue-purple gradient on a dark background; Inter at every size with no tracking decisions; cards inside cards with the same shadow; pastel badges in six colors; the same 8px radius on everything; glassmorphism; a web page's margins inside a popup; a popup that is a scaled-down website; content UI that floats over the page in a rounded dark box with a gradient border; icons used as decoration. Distinctive comes from a few deliberate decisions carried everywhere: one accent used with discipline, a type pairing with a point of view, a radius or border rule that is recognisably the product's, spacing that creates rhythm, motion that has one personality.

## Step 4 — Write `DESIGN.md`

Tokens go in the front matter, using only the format's keys. Values the format has no key for — shadows, motion, surface sizes, shortcuts — are written as tables in their sections. Color tokens use shadcn's variable names, so `DESIGN.md`, `globals.css` and the components share one vocabulary; each color's light value has a `-dark` twin.

```yaml
---
version: alpha
name: "{Product} Design System"
description: One paragraph that captures the whole system in words.
colors:
  background: "#…"
  background-dark: "#…"
  foreground: "#…"
  foreground-dark: "#…"
  # …the same pair for primary, primary-foreground, secondary, secondary-foreground, muted, muted-foreground,
  # accent, accent-foreground, destructive, success, warning, info, border, input, ring, card, card-foreground,
  # popover, popover-foreground, chart-1 to chart-5, sidebar, sidebar-foreground, sidebar-primary,
  # sidebar-primary-foreground, sidebar-accent, sidebar-accent-foreground, sidebar-border, sidebar-ring,
  # and the product's own tokens
typography:
  display: { fontFamily: …, fontSize: …, fontWeight: …, lineHeight: …, letterSpacing: … }
  # …h1, h2, h3, h4, body-lg, body-md, body-sm, caption, label-md, button, mono, numeric (fontFeature: '"tnum"')
rounded: { none: 0px, sm: …, md: …, lg: …, xl: …, full: 9999px }
spacing: { base: 4px, … }            # the scale, matching Tailwind's steps
components:
  button-primary: { backgroundColor: "{colors.primary}", textColor: "{colors.primary-foreground}", typography: "{typography.button}", rounded: "{rounded.md}", padding: … }
  button-primary-hover: { backgroundColor: … }
  button-primary-dark: { backgroundColor: "{colors.primary-dark}", textColor: "{colors.primary-foreground-dark}" }
  # …one entry per component and state, with a -dark entry carrying its dark colors
---
```

Component entries use only the format's properties: `backgroundColor`, `textColor`, `typography`, `rounded`, `padding`, `size`, `height`, `width`. Borders, shadows and focus rings are described in the component's section.

Then the sections, with exactly these `##` headings in this order:

```markdown
## Overview            — the system in a paragraph; the principles from Step 3; then Key Characteristics: the five to eight visual decisions that make it this product; which browser conventions are kept and which are unified
## Colors              — grouped by role (brand and accent, surface, text, semantic, charts), each with its use; the accent policy; hover, active and disabled; light and dark; a contrast table
## Typography          — the families, why, and where each loads from; a hierarchy table (token, size, weight, line height, tracking, use); measure and tracking rules; tabular figures for numbers in data
## Layout              — the base unit and the spacing scale with use; each surface's shell (popup: header and content; side panel: header, navigation, content; options and full-tab pages: content width); the popup's width and height rules; the side panel's minimum; screen anatomy (header, filters, content, actions)
## Elevation & Depth   — how hierarchy is shown (shadows, borders or surface steps), as a table of level, treatment, shadow value and use
## Shapes              — the radius scale as a table of token and use; borders; focus rings
## Iconography         — the set, sizes, stroke, when an icon stands alone and when it needs a label; the extension icon at every size the stores and toolbars need, its badge treatment, and its light-and-dark toolbar legibility
## Imagery             — photography or illustration, tone, treatment, aspect ratios, avatars, empty-state art, what never appears
## Motion              — the personality; a table of durations and easings by name; surface open and route transitions; how content UI appears and leaves; what animates and what never does (the host page); reduced motion
## Components          — one entry per component in use: anatomy, variants, sizes, every state (default, hover, focus, active, disabled, loading, error, selected), and when to use it. Buttons, inputs and fields, selects, checkboxes and radios, switches, forms, cards, lists (virtualised), tabs, the surface header, the side panel's navigation, dialogs and sheets (inside a surface, and inside a shadow root), menus, tooltips with their shortcut, toasts and inline alerts, badges and status, avatars, skeletons, empty states, the signed-out prompt, the permission-needed prompt, the update banner; the content UI pieces (the anchored button or badge, the inline card, the floating panel) with how each sits on a page; then the signature components only this product has
## Keyboard            — the command table (action, default keys on macOS and elsewhere), how shortcuts are shown in tooltips, focus order and focus rings per surface, Escape's behaviour, and how content UI handles focus without taking it from the page
## Patterns            — screen patterns (popup glance, side-panel list, detail, create/edit form on the full-tab page, settings, onboarding) and state patterns (loading, empty, error, signed out, permission needed, offline, update available, force update); which flows live in which surface, and when content UI hands off to a surface
## Content and Voice   — tone, person and tense, capitalisation, punctuation; formats for numbers, dates and money; button and menu labels; error, empty-state, confirmation, permission-request and update copy; the store listing's voice; words we use and words we never use
## Surface Sizes       — a table of the sizes the system is designed for: the popup (width, minimum and maximum height), the side panel (minimum and typical widths), the options and full-tab pages, and content UI on narrow and wide pages; how each adapts; 100% and 200% browser zoom and the largest font setting
## Accessibility       — contrast, focus, keyboard, screen readers (how content UI is announced as the product's region), motion, labels, and the checks every screen passes
## Do's and Don'ts     — the ten rules a builder is most likely to break
## Iteration Guide     — how to apply the system (reference components and tokens by name, one component at a time, the default body style, how scarce the accent is) and how a later phase extends it
```

**Rules for the content:**

- **Every value is a decision, with a use.** A token nobody is told when to use will be used wrongly.
- **Tokens are the source; prose explains them.** A value in prose always matches its token, and names it (`{colors.primary}`).
- **States are not optional.** A component entry without its disabled, loading and error states is incomplete.
- **Plain where it can be, precise where it must be.** The Overview and Content and Voice sections read like a person wrote them; the token and component sections are exact.
- **Contrast is computed, not assumed.** Every text-on-surface pair in the colors table shows its ratio and passes AA (4.5:1 body, 3:1 large text and UI). Fix the palette rather than the rule.
- **Surface differences are stated, not left to habit.** Where the popup, the side panel, the pages and the content UI differ (density, header, how a dialog opens, what Escape does), the component says what each gets; where it is the same everywhere, it says so. Content UI is specified against an unknown page: its own background, border and type, never inherited.
- **Nothing that belongs elsewhere.** No code, no component props, no business rules, no screen inventories. The system says how things look, move and read; the screen specs say what they contain.
- **No stub sections.** A section with nothing to say for this product is left out.
- **Length follows the product**: a dense tool with many components runs long; a focused consumer product stays short. Cut anything that repeats.

## Step 5 — Verify and apply

1. **Lint it:** `npx @google/design.md lint DESIGN.md`. No errors. The linter checks every component pair against 4.5:1, so a contrast warning is fixed in the palette unless the pair is large text or a non-text element that the contrast table shows at 3:1 or more. Orphaned-token warnings are expected for colors no component token can reference (borders, rings, charts); any other warning is resolved.
2. **Read it back as a builder.** Pick three screens from the app flows (one popup glance, one side-panel list, one content-UI piece on a target site) and describe each using only `DESIGN.md`. Anything you had to invent is a gap; fill it.
3. **Check the tokens** cover every shadcn variable with its `-dark` twin, and every pair in the contrast table passes.
4. **Apply the tokens** if the foundation exists: run `/extension-setup`'s design tokens step so `globals.css`, `content.css` and the shadcn components carry the system, then render the component set on a dev-only route of the full-tab page (`/design-check`, created here if it does not exist, excluded from production builds) and the content-UI pieces on the example fixture page, and look at them, light and dark, in the popup at its width, in the side panel, at 100% and 200% zoom, in Chromium and Firefox. Fix what looks wrong in the system, not in the screen.
5. **Record** in the `Design system` row of `CLAUDE.md`'s project configuration that `DESIGN.md` exists and the phase it was last extended in, and report: the direction, the references consulted, what makes this product distinctive, and any decision left open.

## Later phases

A later phase extends the system when it introduces something the system does not cover: a new component, a new data visualisation, a new surface, a new kind of content UI or a new target site whose pages the UI must sit on. Add the tokens and the section text in place, lint again, and compare with the committed version (`git show HEAD:DESIGN.md > /tmp/DESIGN.prev.md`, then `npx @google/design.md diff /tmp/DESIGN.prev.md DESIGN.md`); the token changes it lists go in the report. Update the `Design system` row in `CLAUDE.md` with the phase. Never change an existing token's value in a later phase without a screen-by-screen check of what uses it; a palette change is a project, not an extension.
