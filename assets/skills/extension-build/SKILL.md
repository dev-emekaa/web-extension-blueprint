---
name: extension-build
description: Design and build the extension's screens for one feature area (first phase) or one feature (later phases) — the screen inventory across the popup, side panel, pages and content UI, references, canvas designs for your approval, then the screens built for every target browser with realistic mock data through the real data layer, the content-script anchors and permissions they need, images, tests, and the screen specs handed to the backend. Use when turning a unit's business rules and app flows into designed and built UI, before its backend exists.
---

# Extension Build

## 0. Project Configuration

Read these from the project's `CLAUDE.md`. If a value isn't recorded there, ask me for it once and offer to add it, rather than guessing or asking again each session.

| Key | What it is |
| --- | --- |
| **Project name** | The product this client serves. |
| **Audience · client** | Which audience and client this repo is. Decides which docs folders apply. |
| **Backend repo path** | For `pnpm docs:pull` and `pnpm docs:push`. |
| **Dev server command** | How to run the extension with `wxt dev`, in mock mode. |
| **Test commands** · **E2E command** | Unit and integration; Playwright. |
| **Browsers** · **Surfaces** · **Target sites** · **Single purpose** · **Sign-in providers** · **Payments** | Which browsers and surfaces the product has, which pages its content UI may run on, and the purpose every permission must serve. Decides which patterns a unit's screens may use and which permissions a unit may ask for. |
| **Design canvas** | `/design` where available. |
| **Design reference library** | Mobbin, through its MCP. |
| **Image generation** | Gemini through its MCP, or none. |
| **Stock photo providers** | Unsplash, Pexels — which are configured. Their keys live in `.env`, never in `CLAUDE.md`. |
| **Production status** | Not published yet / published. Decides how carefully existing screens are changed, and whether a new permission is a release decision. |

## 1. How This App Is Built

The unit of work is a **feature area** in the first phase and a **feature** in every later phase, in the order the UI tracker gives. Each unit goes through two stages:

- **Design** — every screen the unit's flows need, designed on the canvas, iterated with me, and approved. Ends with the unit marked `Designed`.
- **Build** — the approved screens built for every target browser with realistic mock data through the real data layer, tested and verified against the designs (`Built`), then specced for the backend (`Specced`).

The two stages can run in one session or in two; the tracker says where a unit is. By default a session takes one unit through whichever stage comes next. A **whole-phase build** takes every `Designed` unit through the build stage in one session (Section 6); design is never batched across units, because every batch needs my eyes.

Three docs define what a unit's screens must do, found through the `CLAUDE.md` documentation map:

- the **app flow** tells you *which screens exist and how the user moves between them*,
- the **business rules** tell you *what each screen must respect*: limits, states, who may do what,
- the **design system** tells you *how everything looks, moves, feels and reads*, on each surface and on the pages the content UI joins.

In a later phase, the feature's phase files are read together with the canonical files of every area it touches, plus the existing screen specs and the built screens' code, so every screen that already exists is known before a new one is designed.

**Throughout, "the unit" means whichever is being built**: the feature area in the first phase, the feature in a later one. The workflow is one workflow; the clauses that differ say which phase they apply to.

Nothing is designed that isn't traceable to a flow or a rule. If a screen seems needed and no flow calls for it, that is a gap to raise, not a screen to invent. We build **only the unit I name**; never ahead into other units.

---

## 2. Reference — Governing Files

### 2.1 — Inputs: read before designing

| File | What it gives you |
| --- | --- |
| `prd` (or the phase's `frd`) | The product, its roles and their permissions, the features in scope. |
| `feature-areas` | Every feature area, what each owns, the first phase's order. |
| `DESIGN.md` | The design system, at the project root. Read in full before the first screen; every design decision defers to it. |
| `app-flow-{{unit}}` | Every flow in the unit: entry trigger, each user action and system action, branches, error paths, the actor. The screen inventory is derived from this. |
| `business-rules-{{unit}}`, and `business-rules-shared.md` where it exists | Field limits, allowed transitions, role gates, formulas the screen displays, error conditions the screen must show. |
| `screen-specs-*` and the built screens *(later phases, or when the unit touches existing screens)* | What exists, exactly. |
| `docs/phase-{n}/review/` *(later phases)* | The review's actions for the frontend, applied before designing. |
| The UI tracker | Readiness and order. |

**Standards**: `coding-standards`, `security`, `testing` apply to everything built; `analytics` where installed. Follow them as written; this skill does not restate them.

### 2.2 — Artifacts: create or update as you build

| Artifact | What it is | When |
| --- | --- | --- |
| UI tracker | `docs/ui-implementation-order.md` (first phase) or `docs/phase-{n}/ui-implementation-order.md`. One row per unit in build order, status, dates, notes. Created from `feature-areas` or the backend's phase `implementation-order.md` on this skill's first run in a phase. | First run; every status change. |
| Approved designs | `docs/design/screens/{unit}/` — one export per screen and material state, numbered in flow order. | Approval. |
| `docs/design/images.md` | Prompts for images that must be generated outside the session, per the Images step. | When needed. |
| The screens | Routes in each surface's `routes.tsx`; screens and parts in `features/{area}/components/`, hooks in `features/{area}/api/`, schemas, fixtures and handlers in `features/{area}/mocks/` — where `{area}` is the feature area each screen belongs to, which in a later phase may be several per feature. | Build. |
| Shared components | New `components/shared/` pieces and shadcn additions a screen needs, generalised only when a second screen needs them. | Build. |
| Browser wiring | The content-script anchors, context-menu entries, commands, notification routes and permissions a unit's screens add, in `features/{area}/content/`, `shared/`, `wxt.config.ts` and the background's modules. | Build. |
| Fixture pages | `src/test/fixtures/` and `e2e/fixtures/`: a copy of each target site's relevant markup the unit's content UI anchors to, with the site and date it was taken from. | Build. |
| Tests | Integration tests per screen state, content tests per anchor on its fixture, unit tests for logic (and for any background handler the unit adds), a Playwright journey for the unit's primary flow. | Build. |
| Screen specs | `docs/screen-specs/screen-specs-{{unit}}.md` (or the phase file), written by `/screen-specs` from the built screens. | After the build. |

---

## 3. Global Constraints

**Fidelity to the docs**

- Every screen traces to a flow step; every flow step has a screen or a state on a screen. A flow with a missing screen is incomplete, and the audit in Step 2 is where that is caught.
- Every field a screen captures carries the business rules' constraints (as the form schema) and every state a rule defines (pending, expired, rejected) has a designed and built state.
- Where the docs are silent, ambiguous or contradictory: **stop and ask.** Do not invent product behaviour, fields, or steps. A reference app that does something the docs don't ask for is a question, not a screen.

**Fidelity to the design system**

- `DESIGN.md` wins over taste, references and habit. A screen that needs something the system lacks is extended through `/design-system`'s later-phases rule, never styled ad hoc.
- No generic defaults: no placeholder gray boxes, no lorem ipsum, no template layouts, no spinners where the system specifies skeletons, no decoration the system does not call for.
- Surface conventions are the system's decisions: where `DESIGN.md` says the popup, the side panel, the pages and the content UI differ, each is designed; where it says they are the same, one design serves all. Browsers differ in chrome only; one design serves every browser.

**Mock data, not mock logic**

- Mock data goes through the real data layer: hooks, `lib/api.ts` over messaging, the gateway, the types, as `coding-standards` → Mock mode defines. A screen never imports a fixture.
- Fixtures are realistic and cover every state the screen shows, including empty, maximum and error. Handlers keep session state so a flow can be walked end to end (create, then see it in the list, then edit it).
- No business logic is reimplemented in the client beyond what a form needs for instant feedback. A calculation the backend owns is displayed from the response, not computed.

**Engineering**

- Build only the named unit. Shared components are generalised when a second screen needs them, not before.
- Follow the existing patterns of the codebase. No new dependency outside `tech-stack` without asking; the build plan names what the unit needs and why.
- Accessibility is part of done: keyboard reach and order, focus, screen-reader labels, contrast, browser zoom, reduced motion, as `coding-standards` and `DESIGN.md` specify. Content UI is announced as the product's region and never takes focus from the page on mount.
- Every target browser, every screen. Nothing is called built that was looked at in Chromium only: Firefox is run locally with `wxt dev -b firefox`, and content UI is looked at on each target site's real page as well as on its fixture.
- Browser capabilities are designed states, not afterthoughts: a permission request and its denied state, a context-menu entry's landing screen, a command's effect, a notification's landing screen, the signed-out state, offline, the update banner — each is in the inventory when a flow needs it, and each has the state `DESIGN.md` → Patterns gives it. Anything a surface needs from the browser goes through the protocol's existing capabilities; a new message, a new permission or a new host is a contract change raised in the build plan, never added quietly. A content script asks for its feature's named messages, never for `api:fetch`.
- Changing an existing screen in a later phase is surgical: the change the feature asks for, nothing else. If the extension is published, the change keeps the integrated data layer intact, and a change that needs a new permission is flagged as a release decision, because it disables the extension for Chromium users until they accept it.

**Images**

- Every image is looked at before it is used and after it is placed. An image that was never seen is never shipped.

---

## 4. Phase 0 — Information Architecture *(once, before the first unit)*

Screens are built one unit at a time; the surfaces they sit in are not. Before the first unit's design, derive from the PRD, `feature-areas`, the single purpose and every area's flows:

- **The surface model:** which kind of screen lives on which surface — what the popup shows at a glance, what the side panel carries, what the options page holds, what needs the full-tab page — and, for each area, which surface its main screens live on, for which roles; where onboarding, sign-in, the permission requests and the force-update screen sit. Each role's view of the extension.
- **The content-UI model:** on which target sites the extension shows UI, anchored to what (a selector or a page structure described in words), as which kind of piece (an anchored button or badge, an inline card, a floating panel), and what hands off to a surface. Each anchor becomes a fixture page the tests use.
- **The route map:** the route for every area's main screens on each surface, with its dynamic segments. Routes are also the targets of context-menu entries, commands and notifications, so they are a product decision; changing them later breaks entry points.
- **The entry points and shells:** each surface's shell per `DESIGN.md` (the popup's header and content; the side panel's header, navigation and content; the pages' content width), the toolbar icon's behaviour, the context-menu entries, the command table and what each does, where search, notifications and the account entry live, and how each surface adapts to zoom and the largest font setting.
- **Global patterns the units will share:** the list screen anatomy, the detail screen anatomy, the create/edit pattern (which surface, dialog or sheet), the confirmation pattern, where errors and toasts appear per surface, the permission-request pattern, the signed-out and offline states, the update banner.

Design the shells and one example of each global pattern on the canvas, and **wait for my approval**. Then build the shells in `components/shared/`, the command table in `shared/`, the context-menu entries in the background, replacing the placeholders `/extension-setup` left, and the routers' real layouts, before the first unit. Record the surface model, the content-UI model, the entry points and the route map in `docs/design/surfaces.md`. Phase 0 ends like a unit: its tests pass, the tracker (created at the session's opening, Section 8) records it as done in its notes, and it is delivered as a PR per Step 12.

---

## 5. Workflow

### Stage A — Design

#### Step 1 — Select and scope

- Run `pnpm docs:pull` so the docs are current, and commit what it changed.
- Confirm in the UI tracker that the unit is `Ready` and the one above it is at least `Designed` (or, in whole-phase build mode, that every unit in scope is `Designed`). **Never design from a `Draft` row.** If the tracker does not exist yet, create it now from `feature-areas` (first phase) or the backend's `docs/phase-{n}/implementation-order.md` (later phase), one row per unit, all `Draft` until their docs exist, and commit it. At the start of every session, advance any `Draft` row whose Readiness condition is now met to `Ready` with the date — first phase: its business rules and app flows exist; later phase: the review has settled its rules and flows with no open founder decision (the backend's `implementation-order.md` may still read `Draft` until its screens are reviewed; that is expected, since this repo writes the screens).
- Read everything in Section 2.1 for this unit.
- In a later phase: read the existing screen specs for every area the feature touches, and open the built screens' code, so every change is made to the real current screen. Apply the review's frontend actions.
- Identify **prerequisites**: screens from other units this unit links to (a detail screen a list links into), shared components that must exist, auth states the flows assume. If a prerequisite is unbuilt, say so before starting rather than stubbing around it.

#### Step 2 — Screen inventory → **wait for my approval**

Derive, from the app flow and business rules, every screen the unit needs, and present it as a table:

| Screen | Flow(s) and step(s) | Roles | Surface · Route | Kind | States |
| --- | --- | --- | --- | --- | --- |

- **Surface:** popup, side panel, options page, full-tab page, or content UI on a named target site with its anchor.
- **Kind:** screen, dialog, sheet, content piece (anchored button or badge, inline card, floating panel), step of a flow, section of another screen.
- **States:** every materially different state the flows or rules produce: loading, empty, error, offline, signed out, permission needed and denied, the anchor-not-found state for content UI, and the entity states (draft, pending, approved, expired); form validation and submit states; a browser variant only where `DESIGN.md` says the browsers differ beyond chrome.
- **Dialogs, sheets and content pieces** are listed under their parent.
- **Permissions:** any permission or host a screen needs that the manifest does not yet declare, as its own row, because it is a product and release decision.
- Walk every flow step by step and tick the screen each step lands on. **A step with no screen is a gap**; a screen no step reaches is a question. List both.
- In a later phase: mark each row **new**, **changed** (naming the existing screen and what changes) or **unchanged**.
- Note the **reference plan**: which screen types you will research on Mobbin (one query per screen type, not per screen), among extensions and compact web products in the product's category.

Do not proceed until I approve the inventory and answer the questions.

#### Step 3 — Reference research

On Mobbin (`platform: web`, where extensions and the compact web products closest to a popup and a side panel are catalogued; follow the Mobbin search skill where installed), one search per screen type in the inventory, within the product's category where the category matters and across admired products where the pattern is universal. Look at the actual screens. For each screen type, record in two or three lines what the strong references do: layout, hierarchy, how they handle the states and the density, what their actions look like. Keep it to what this unit needs; the design system already settled the global decisions. Cite each reference by its Mobbin link.

References are never a source: coverage comes from the flows, constraints from the rules, appearance from `DESIGN.md`.

#### Step 4 — Design the screens on the canvas

Design in **batches of one flow** (or one screen type across flows where that reads better), in flow order, each batch presented for review before the next.

- **Invoke the `/design` skill** with a brief per batch: the product in one line, the design system's direction (pointing at `DESIGN.md`, which it reads), the screens with their purpose, content, actions and states from the inventory, the surface and route each has, the shell it sits in, the frame (the popup at the design system's width with its height following content; the side panel at its typical width; a page at its content width; content UI drawn on a copy of the target site's page so the artboard shows where it sits and what it must not cover), the commands each action carries, and the references' lessons. Ask for every state the inventory lists, as separate artboards where the difference is material.
- **Where `/design` is not available** in this session (it needs a recent Claude Code with a claude.ai login), design each screen as a static HTML prototype in `docs/design/screens/{unit}/prototypes/`, built from the real design tokens, and open them for review the same way. The approval and export rules below are the same.
- **Judge every artboard before showing it:** does it respect `DESIGN.md` token for token; does it carry every element the flow step needs and nothing the docs don't ask for; does it have its states; is every action reachable by keyboard and does the primary one carry its shortcut; would it survive the popup's maximum height, a narrow side panel and 200% zoom; would content UI still read as the product's on a page with the opposite palette; would a user know what to do next; does it look like this product or like a template. Fix before presenting.
- **Present the batch** with one line per screen on what it does and any decision I should look at. I will edit on the canvas or ask for changes; apply them, re-check against the rules and the system, and present again. A change that contradicts the business rules or the design system is raised, not silently applied.
- **Content is real.** Labels, empty-state copy, error messages and confirmations are written now, in the voice `DESIGN.md` defines, from the business rules' messages where they exist. They become the built screen's copy.

#### Step 5 — Approval and checkpoint

When I approve the unit's designs:

1. Export every screen and material state to `docs/design/screens/{unit}/`, named per `documentation-conventions`.
2. Record under the tracker row anything the designs decided that the specs must carry: fields shown on a screen that the flow implied but did not list, actions that moved between screens, states added.
3. Mark the unit `Designed` with the date.
4. Commit. If the session continues into the build stage, go to Step 6. Otherwise report — the screens designed with their states, the references used, the decisions recorded under the tracker row, and any question still open — and stop. The build stage starts in a fresh session from this checkpoint.

### Stage B — Build

#### Step 6 — Build plan → **wait for my approval**

From the approved designs and the docs, list before writing code:

- **Routes** to add or change, per surface, with their presentation (screen, dialog, sheet) and title; the content pieces per target site with their anchors.
- **Screens and parts**: each component with its file.
- **Shared and shadcn components** to add or generalise, with the reason.
- **Browser capabilities**: each context-menu entry, command, notification, permission request or hand-off between surfaces the screens use, which protocol message serves it, and any contract change it needs (a new message is a decision, with its payload schema, its allowed senders and hosts, and its background handler; a new permission or host is a decision with its justification, and a release decision once the extension is published). The fixture pages to write for each anchor.
- **Data**: each query and mutation hook, its key, the endpoint shape the screen implies (method, path and fields, as the best reading of the flow; the backend will settle it), the provisional types in the area's `api/types.ts`, the fixtures, and the handlers with the state they keep.
- **Forms**: each schema, its fields and the business rules each constraint comes from.
- **Images**: every image the designs use, with its source decided per Step 8.
- **Tests**: the states each screen's integration tests cover, the content tests per anchor, the unit tests for any background handler and page-data parser, the Playwright journey for the unit's primary flow.
- **New dependencies**, each with the reason, or none.
- **Gaps and conflicts**: anything the designs and docs disagree on, anything underspecified. List these as questions.

#### Step 7 — Build the screens

- Build in flow order, one screen at a time, each with every state from the inventory, matching the approved export. Run the extension in mock mode and look at each screen as you finish it, on its surface, light and dark, at 200% zoom, with the keyboard; look at content UI on its fixture page and on the real target site.
- Entrypoints stay thin; screens live in the feature folder; content pieces in the area's `content/`; data comes through hooks; forms through React Hook Form and the unit's schemas, on a surface that survives (never a long form in the popup); commands from the shared table; copy from the designs; tokens from the system, in pixels inside shadow roots; motion from the system's table. Exactly as `coding-standards` says.
- Mock handlers simulate the backend as the flow describes it, including the error responses the rules define (a 422 with field errors, a 409 on an invalid transition), so error states can be walked.
- Wire `analytics` events for the unit's intents where the rule is installed.
- Wire the browser capabilities the plan listed: the anchors and their observers, the context-menu entries and commands routed to surfaces, the permission request and its denied state, the notification routes, and any contract change the plan approved — a message with its schema, sender check and unit test; a permission with its justification comment in `wxt.config.ts` and its line in `docs/release/` — and write the fixture pages for every anchor, recording the site and date each was taken from.
- Keep every flow walkable end to end in mock mode: the handlers' state makes a created record show up where the flow says it will.

#### Step 8 — Images

For each image the designs call for, choose the source that fits:

- **Stock photography** where the image is a real-world subject: search Unsplash or Pexels through their APIs with the keys in `.env` (`CLAUDE.md` lists which providers are configured). If no key is configured, ask me for one and walk me through getting it (both offer free developer keys in minutes). Fetch the candidates, **look at them**, pick the one that matches the design's imagery rules, and record its attribution in `public/images/ATTRIBUTION.md`.
- **Generated imagery** where the image is illustrative, abstract, or has to match the system's illustration style: generate it through the Gemini MCP when connected, with a prompt that states subject, style from `DESIGN.md` → Imagery, palette, composition, aspect ratio and size. **Look at the result**; regenerate until it fits.
- **Neither available**: write the prompt into `docs/design/images.md` — one entry per image with its screen, purpose, exact size and aspect ratio, style, palette, composition, what must and must not be in it, and the file name and path it should be saved to — and build the screen with a tokenised placeholder of the right size. Tell me in the report; when I add the files and say so, place them.

After placing any image: optimise it (the right format and sizes for 1× and 2× displays; bundled in `public/images/` only when it is part of the product's chrome and counted against the package budget, otherwise served from the backend's CDN), and look at the screen again.

#### Step 9 — Tests

Follow `testing` for structure and approach.

- **Integration tests** per screen: it renders with the fixtures, each listed state renders, the primary actions fire the expected request (asserted on the messaging double), forms validate and map a 422; per content piece: it mounts once on its fixture, sends its feature message with the parsed data, renders its states, and unmounts on invalidation.
- **Unit tests** for schemas with non-trivial rules, formatters, the page-data parsers with hostile input, and any background handler's validation and sender checks.
- **E2E**: the unit's primary journey as a Playwright spec in `e2e/`, listed in its README and in the staging suite; in the smoke suite only if the journey is critical to the product.
- Run the whole suite, including earlier units'. Shared components change; if this unit broke an earlier one, fix it before continuing.

#### Step 10 — Verify against the designs

Walk every flow in the built extension in mock mode — driving it with Playwright against the `chrome-mv3` build and capturing screenshots of each surface and of content UI on its fixture page — screen by screen, state by state, and compare each with its approved export. Chromium is yours to walk with Playwright on the built extension. Firefox and the real target sites are my check: give me the checklist per flow, I run it with `wxt dev -b firefox` and on the real pages and report what I saw, and you record it. The walk you present marks each item **verified by Playwright** or **reported by me**, never as seen when it was not. Check the popup at its minimum and maximum height, a narrow side panel, dark mode, 200% zoom, the keyboard-only path through each flow with its commands, the context-menu entries the unit added, the permission request and its denied state, and the accessibility labels (the labels the test queries find). **Present the walk in the chat, flow by flow:** each screen, each browser, what was checked, by whom, and whether it matched. End with everything that did not match, and fix it before Step 11.

If a design cannot be built as approved (a layout that does not survive real content lengths, a state the design missed), say what and propose the change; a change to an approved design is raised, not made quietly.

When the walk passes, mark the unit `Built` with the date and commit.

#### Step 11 — Screen specs and handoff

1. Run `/screen-specs` in its **From designs** mode — its designs input is the unit's exports in `docs/design/screens/{unit}/`, checked against the built screens — into the canonical file in the first phase or the phase file in a later one. Review the specs against the flows and rules as that skill says. Every action's endpoint is `pending` unless it already appears in the api-docs that arrived from the backend, as `documentation-conventions` says.
2. Commit the specs, then run `pnpm docs:push --specs {file}` for the unit's file (the canonical area file, or the phase feature's file) so the backend repo receives it, and tell me the backend can run its review for this unit.
3. Mark the unit `Specced` with the date.

#### Step 12 — Update the tracker and report

- The tracker row carries its new status and date.
- Report completion:
  - **Unit** and a one-line summary of what it now lets the user do.
  - **Screens built**, with routes and the states each covers.
  - **Components added** to shared or `components/ui`, and why.
  - **Endpoints implied** by the hooks, as the backend will read them in the specs.
  - **Images** placed, and any waiting in `images.md`.
  - **Tests** — counts per layer, and confirmation that earlier units' suites pass.
  - **Verification** — the Step 10 walk passed, with what it caught.
  - **Open questions or deferred items**, if any.
  - **Contract changes**, if any: the messages added, with their handlers and tests; the permissions or hosts added, with their justifications.
  - **Fixture pages** written, with the site and date each was taken from.
- **Deliver it as a PR.** Create a branch off `origin/main`, push to it, and give me the link. Never create the PR yourself. Merging deploys to staging per `ci-cd` — the staging builds as artifacts and the signed Firefox build — where the unit runs in mock mode until it is integrated.

---

## 6. Whole-Phase Build Mode

Builds every `Designed` unit of the phase through Stage B in one session, in tracker order. My input moves to the start and the end.

1. **One plan for the phase → wait for my approval.** Step 6 for every unit, in one document, every question and new dependency collected in one place. The run covers `Designed` units in order and ends before the first that is not.
2. **Build each unit in order — Steps 7 to 9 — in a fresh context.** Give each unit a subagent with its approved plan, told to read this skill, `.claude/rules/` and `DESIGN.md` first. If subagents are unavailable, build in this session and compact between units. Never build units in parallel: they share the shells, the shared components, the messaging protocol, the manifest and the mock state.
3. **Tests gate each unit.** A unit is complete when its suites and every earlier unit's pass; it is then committed, one commit per unit on one branch. A unit that fails, or meets anything its plan did not settle, stops the run with a report.
4. **One verification walk** (Step 10) of every flow in the phase, marking each unit `Built`; then **screen specs for every unit** (Step 11), one push per unit, each marked `Specced`; one report, one PR.

---

## 7. Definition of Done

A unit is not done until all of the following are true:

- [ ] Every flow in the unit's app flow has a screen or screen state for every step, with branches and error paths.
- [ ] Every screen in the approved inventory is designed, approved, exported, and built as approved on its surface in every target browser, with every listed state, at 100% and 200% zoom; every content piece was looked at on its fixture and on the real target site.
- [ ] Every field respects the business rules' constraints; every rule-defined state is designed and built.
- [ ] Every screen defers to `DESIGN.md`; nothing is styled outside the tokens.
- [ ] Mock data goes through the real data layer; every flow walks end to end in mock mode; no fixture is imported by a screen.
- [ ] Every image was seen before use and after placement, or its prompt is in `images.md` with a correctly sized placeholder in its spot.
- [ ] Every browser capability the unit uses goes through the protocol, with its states designed; every contract change has its schema, sender check, handler and test; every permission added has its justification and its line in `docs/release/`; every anchor has its fixture page.
- [ ] Integration tests cover every screen state, the primary journey is a Playwright spec, and every suite passes, this unit's and earlier units'.
- [ ] The Step 10 walk was presented and everything it caught is fixed.
- [ ] Screen specs written from the built screens, reviewed against flows and rules, and pushed to the backend.
- [ ] The tracker row moved through `Built` to `Specced` with its dates, and the report and PR link were given.

If any box cannot be ticked, say which and why rather than reporting the unit complete.

---

## 8. Opening the session

Before Step 1, in a few lines:

1. **State the phase, the unit and the stage** — design, build, or both; or whole-phase build. If I haven't said, take the next unit in the tracker through its next stage.
2. **State the project values** from `CLAUDE.md` that change what you do: production status, the browsers, the surfaces, the target sites and the single purpose, whether the canvas, Mobbin and image generation are available, which stock photo providers are configured. Ask for any that are missing.
3. **Say whether Phase 0 is done.** If the surface model and the shells are not approved and built, we start there — after creating the UI tracker if it does not exist yet, as Step 1 describes.

Then go to Step 1. Do not summarise the workflow back to me — start it.
