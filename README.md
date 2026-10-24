# web-extension-blueprint

**Design-to-store web extension workflow for Claude Code — the docs come first, the design system governs every surface, the backend's api-docs drive integration, and releases go through the browser stores.**

Seven Claude Code skills and the rules they depend on, installed into a WXT project as one pipeline: set up the foundation, write the design system, design and build each feature area's screens on mock data across the popup, side panel, pages and content UI, spec them for the backend, integrate the real endpoints, fix what breaks, and take each release through the Chrome Web Store, Firefox Add-ons and Edge Add-ons. It is a frontend half of a pair: the [Backend Blueprint](https://github.com/dev-emekaa/backend-blueprint) writes the product docs this one reads, and reads the screen specs this one writes. The [Web App Blueprint](https://github.com/dev-emekaa/web-app-blueprint), the [Mobile App Blueprint](https://github.com/dev-emekaa/mobile-app-blueprint) and the [Desktop App Blueprint](https://github.com/dev-emekaa/desktop-app-blueprint) are its siblings for the other clients of the same product, and the [Ideas Blueprint](https://github.com/dev-emekaa/ideas-blueprint) produces the brief, PRD and logo all of them start from.

```bash
npx github:dev-emekaa/web-extension-blueprint init
```

---

## Install

Run it in your project root. Nothing to install first — `npx` fetches this repo and runs it.

```bash
npx github:dev-emekaa/web-extension-blueprint init [options]
```

| Option | Effect |
| --- | --- |
| `--analytics` | Also install the analytics rule. For products that track user behaviour with PostHog |
| `--personal` | Install skills to `~/.claude/skills` instead of the project |
| `--force` | Overwrite existing files. Default is to skip them |
| `--dry-run` | Print what would change, write nothing |

There is one stack — WXT, React, TypeScript, Manifest V3, Tailwind CSS, shadcn/ui, TanStack Query — so no `--stack` flag. Add `--dry-run` to any command to preview it first. Re-running is safe: existing files are skipped and listed, never clobbered.

### What it writes

```
.claude/
  skills/                          7 skills
  rules/
    documentation-conventions.md   which docs arrive from the backend, which this repo owns, readiness
    tech-stack.md                  the tools, the browsers, and the environments and stores
    coding-standards.md            the four contexts, messaging and the gateway, mock mode, session, surfaces, content scripts, how code is written
    security.md                    security policy for code that runs inside other people's pages, and the contract the backend must meet
    testing.md                     test layers, what each proves, fixture pages, the browser matrix
    ci-cd.md                       pipelines, store submission, tags, rollback — loads only when deploy files are touched
    analytics.md                   --analytics only
CLAUDE.md                          documentation map · audiences · project configuration
docs/
  product-docs/  business-rules/  app-flows/  api-docs/  api-testing/     arrive from the backend
  screen-specs/  design/screens/  integration-handoff/  infra/  release/   written here
```

`docs/feature-areas.md` is deliberately **not** created: it arrives from the backend with `pnpm docs:pull`, which `/extension-setup` installs. `docs/screen-specs/` is created empty; `/screen-specs` fills it from the built screens.

### After installing

1. Fill in the **audiences** and **project configuration** blocks in `CLAUDE.md`. The backend repo path, the single purpose and the target sites matter most: the last two decide the permissions, and the permissions decide what the stores accept. Anything you leave blank, the skill that needs it asks for once.
2. Run `/extension-setup`. It builds the foundation and pulls the backend's docs.
3. Run `/design-system` once the PRD and feature areas are in.
4. Run `/extension-build` for the first feature area.

### Skills are project-scoped by default

They install to `.claude/skills/` and are meant to be committed: cloud sessions, scheduled runs and Cowork read only project skills, and rules that travel without the skills that execute them are conventions with no tooling. `--personal` exists for throwaway work.

### Updating

In a project that already has it, the installer skips existing files by design, so an update needs `--force`:

```bash
npx github:dev-emekaa/web-extension-blueprint init --force
```

`--force` overwrites, including local edits to a skill or a rule. Run it on a clean working tree and read the diff before committing. `CLAUDE.md` is never rewritten.

Pin a tag for a fixed version: `npx github:dev-emekaa/web-extension-blueprint#v1.0.0 init`. If an update doesn't seem to have landed, `npm cache clean --force`.

---

## The pipeline

Each step is gated on the one before it. A skill whose inputs don't exist yet stops and says so.

```
Backend Blueprint writes: feature-areas · business-rules · app-flows · (later) api-docs
 │
 ├─ /extension-setup   → the foundation, once; pulls the backend's docs
 ├─ /design-system     → DESIGN.md, once
 └─ /extension-build   → per feature area, in order:
      Phase 0: surface model, content-UI model, shells and entry points, once
      Design:  screen inventory → references → canvas designs → your approval   [Designed]
      Build:   screens on mock data through the real data layer, every browser  [Built]
               → /screen-specs → pushed to the backend                           [Specced]
           ↓   once every phase-1 area is Specced, the backend designs its schema,
               then builds the areas in order, writing api-docs per area
      /extension-integrate → per area as its api-docs land: real endpoints,
                             orphan audit, backend-issues handoff                 [Integrated]
      /extension-release   → submitted to the Chrome Web Store, Firefox Add-ons
                             and Edge Add-ons, reviewed, rolled out, recorded

/extension-issue-fix — any time, for anything that already exists
```

**Later phases** follow the backend's: `/feature-requirements` and `/phase-review` there produce the FRD, the feature files and the frontend actions; here the unit becomes a **feature**, tracked in `docs/phase-{n}/ui-implementation-order.md`, and `/extension-build` reads the existing screens before designing new ones.

---

## The architecture in one paragraph

The extension is a **client**, like the web, mobile and desktop apps: the backend owns the data, the rules and the auth, and the extension holds no business logic. What makes it an extension is that it runs inside the user's browser, across pages the product doesn't control, in four contexts with one direction of trust. The **background** (a service worker on Chromium, an event page on Firefox) is the gateway and the session: every request to the backend is made by it, the access token lives in `storage.session` and the refresh token in `storage.local`, and nothing else ever holds a token. The **surfaces** — popup, side panel, options page, full-tab page — are React screens that ask the background over a typed, Zod-validated messaging protocol. **Content scripts** run inside third-party pages, which the rules treat as hostile: they read what their feature needs, parse it before it crosses messaging, render UI in a sealed shadow root, and ask the background for *features*, never for endpoints. Permissions are the smallest set the product's single purpose needs, because they are both the attack surface and what the store reviews. Before the backend exists, screens run on in-process mock handlers dispatched in the background through the same gateway; integration swaps the handlers for the real endpoints without touching the screens.

---

## The skills

### `/extension-setup`
**The foundation, once.** The WXT scaffold with three TypeScript projects and an entrypoint per surface; `wxt.config.ts` keyed by environment with a development, staging and production identity each (fixed ids, so the sign-in redirect URLs can be registered with the backend), every permission with its justification, the CSP and Firefox's data-collection declaration; the typed messaging protocol in `src/shared/messages.ts` and the background handlers that validate payloads and senders; the session module and the gateway in the background (bearer header, platform and version headers, no cookies, single-flight refresh under a lock, offline as a state); TanStack Query per surface with its cache persisted to `storage.session`; generated API types; the in-process mock mode that lets screens run on realistic data before the backend exists, with passthrough to the real backend for everything already integrated; design tokens from `DESIGN.md` in both `rem` and pixel form; the content-script mount with shadow roots, anchors and invalidation; permissions, context menus, commands, notifications, alarms and the version check; the identity sign-in flow; Vitest with the fake browser, MSW and Playwright on the built extension; Sentry in every context; the three CI/CD workflows with the store submission; the docs sync with the backend repo; an infra guide for the store accounts, ids, redirect URLs and GitHub steps only you can do.
**Reads** the rules, `CLAUDE.md` · **Writes** the foundation, `docs/infra/infra-setup.md`

### `/design-system`
**The source of truth for how everything looks, moves, feels and reads.** Reads the product docs to understand what kind of extension this is — a companion to the page, a tool in its own panel, a utility used in moments — and who uses it, mid-task on someone else's site; researches the best extensions and compact web products in its niche on Mobbin and against the stores' design guidance, and reverse-engineers what makes them work; proposes a direction with a style tile for your approval, with the popup and a side-panel column side by side and one content-UI piece shown on a plain page and on a busy one; then writes `DESIGN.md` in the DESIGN.md format, linted with Google's checker — tokens that map onto shadcn's variables, every component with every state, each surface's shell and sizes, how content UI sits on an unknown page without inheriting from it, keyboard and commands, patterns for the popup glance, the side-panel list, the full-tab form, signed-out, permission-needed and offline, motion, content and voice including the listing's, accessibility — and applies the tokens to the code. Takes the brand from a sibling client's `DESIGN.md` where one exists. Built to avoid the generic.
**Reads** PRD, brief, feature areas, rules, flows, brand assets · **Writes** `DESIGN.md`

### `/extension-build`
**Design and build one unit's screens.** Derives every screen every flow needs, on which surface, with its states — signed out, permission needed, anchor not found — and any permission or host it would add, and gets the inventory approved; pulls references per screen type; designs the batch on the `/design` canvas (content UI drawn on a copy of the target site's page) and iterates with you until approved; then builds the screens for every target browser with realistic mock data through the real data layer, wires the anchors, context-menu entries, commands and permission requests through the protocol, writes fixture pages for every anchor, sources images (stock, looked at before use; generated; or a prompt in `images.md`), writes the tests and the Playwright journey, walks every flow on the built extension and on the real target sites against the approved designs, and runs `/screen-specs` so the backend can build. Two stages with a checkpoint at approval, or a whole phase's build stage in one session with subagents.
**Reads** flows, rules, DESIGN.md, existing screens · **Writes** designs, screens, mocks, fixtures, tests, screen specs

### `/extension-integrate`
**Replace mocks with the real backend.** Audits every screen against the specs and the api-docs first (build gaps, design gaps, missing endpoints, the backend-facing capabilities — the identity sign-in flow, minimum version, deletion, the checkout return — and every feature message a content script sends); aligns the hooks to the api-docs and removes the handlers; wires every documented error and background behaviour; audits for orphan endpoints; logs anything the backend must fix in a handoff file the backend reads; updates the tests to the real shapes; walks every flow against staging in every browser and names the backend commit verified. One unit per session, or a whole phase in order.
**Reads** api-docs, specs, api-testing, flows, rules · **Writes** the integration, the handoff file

### `/extension-issue-fix`
**Change what exists.** Reproduces the issue in the browser it was reported in and then the others, classifies it (client bug, browser defect, target-site change, backend bug to log, doc error to report, change request, backend handoff to adopt), writes the failing test — a refreshed fixture page and a content test for an anchor that stopped matching — fixes surgically, updates every doc the fix touched including the permission justifications and disclosures, verifies the flows it affected in every browser and on the real target sites, says whether the fix rides the next release or goes out as a hotfix from the live tag, and delivers a PR. Flags any permission change as a release decision. Adopts a backend handoff breaking-items-first.
**Reads** the issue, the code, the docs · **Writes** the fix, tests, fixtures, doc updates

### `/extension-release`
**From "what should ship" to "live for users".** Decides the release — from `main` or a hotfix — on the tracker and the staging verification, calling out every manifest change; bumps the version by PR; brings the release notes, the listing per store, the permission justifications, the privacy disclosures (Chrome's privacy form, Firefox's declaration, Edge's, and the privacy policy, all saying the same thing) and the Playwright-generated screenshots up to date with the code; runs the production pipeline, which smoke-tests and submits to the Chrome Web Store, Firefox Add-ons and Edge Add-ons; has you check the build in real browsers on the real target sites; follows each store's review and classifies its findings; raises the Chrome rollout 10% → 25% → 50% → 100% while the crash-free rate holds, halts when it does not; records every release. Covers the first release, with the production ids assigned by the first upload to each store (and the chosen `gecko.id`) registered with the backend before sign-in can work, and the optional Safari build.
**Reads** the tracker, the code, `docs/release/`, Sentry, the store dashboards · **Writes** `docs/release/`

### `/screen-specs`
**What each screen sends to the backend and shows from it.** The same skill the Backend Blueprint ships, unchanged, run here in its From designs mode on the built screens so the backend has the specs it builds against.
**Reads** the built screens, flows, rules · **Writes** `docs/screen-specs/`

---

## The rules

These install into `.claude/rules/`, loaded at the start of every session — except `ci-cd.md`, which loads only when a workflow, `wxt.config.ts`, `web-ext.config.ts`, an `.env` file or a Playwright file is touched.

- **`documentation-conventions.md`** — which docs arrive from the backend and are read-only here, which this repo owns, how screen specs change hands, the UI tracker and its six readiness values (Draft, Ready, Designed, Built, Specced, Integrated).
- **`tech-stack.md`** — the tools: WXT with Manifest V3 per browser target, React with React Compiler, React Router in memory mode per surface, Tailwind, shadcn/ui, content UI in shadow roots, TanStack Virtual, Motion, TanStack Query persisted to `storage.session`, fetch in the background's gateway over typed messaging, Zustand, WXT's storage, React Hook Form and Zod, the identity sign-in flow, Vitest with the fake browser, Playwright, `web-ext`, `wxt submit`, Sentry, optional PostHog. And the environments, identities, redirect URLs and stores.
- **`coding-standards.md`** — the architecture (the surfaces draw, content scripts observe, the background holds the session and talks to the backend, the backend decides), the project structure and dependency direction, the messaging protocol and the gateway with offline as a state, mock mode, the data layer, the session across worker restarts, surfaces and routing and the host page, components for the keyboard, state, forms where they survive, effects, styling in `rem` and in pixels, permissions, alarms and the worker's lifecycle, performance budgets per context, errors, code style, tests in this stack.
- **`security.md`** — the page is hostile, the background is the trust boundary, CSP and the surfaces, permissions, tokens and sessions, data in the browser, transport, updates and supply chain, secrets, third-party SDKs, payments, logging, personal data, store compliance, and the **backend contract**: tokens in the body for the platform header, the minimum-version endpoint per browser, the native sign-in flow with a registered https redirect URL, account deletion, status semantics, additive changes, real-time reachable without cookies or headers (handshake payload or single-use ticket).
- **`testing.md`** — static (with `web-ext lint` and a manifest assertion), unit (the background's modules against the fake browser, the gateway against MSW), integration (the bulk, over a messaging double; content scripts on fixture pages), E2E with Playwright on the built Chromium extension, smoke, manual QA on a browser matrix of Chrome, Edge, Firefox and optionally Safari; fixture pages kept honest; the regression rule.
- **`ci-cd.md`** — PR checks with every target built, its manifest asserted and `web-ext lint`, the automatic staging deploy that produces the zips and an unlisted-signed Firefox build and moves `staging-current`, the manual production deploy of the verified commit that smoke-tests and submits to the three stores and tags `v*`, the hotfix branch, releasing alongside the backend, what rollback means when the stores publish and a browser is only ever moved to another version.
- **`analytics.md`** *(with `--analytics`)* — `posthog-js` bundled in the surfaces only, explicit events named `object_action`, no personal data and no page content, identify and reset, consent, the Firefox data-collection declaration.

---

## Working with the Backend Blueprint

The two repos share the docs through `pnpm docs:pull` and `pnpm docs:push`, which `/extension-setup` installs and which read the backend repo path from `CLAUDE.md`. Pull brings the backend's docs here, including its copy of the screen specs once it owns them; `push --specs` sends one screen-spec file there and `push --handoff` sends this client's two backend-issues files (`backend-issues-extension.md` and its `-resolved` archive — each client writes only its own). The backend's `/backend-issue-fix` reads its open entries itself, appends each resolution, and writes a client handoff message that `/extension-issue-fix` adopts.

The backend must meet the contract in `security.md` → Backend contract: its `security.md` and services rule (v1.4.0 and later) state the token, native sign-in and deletion lines from its side with the extension's platform values and https redirect URLs, its build skill the minimum-version endpoint per browser; its api-docs document the platform and version headers and the error shapes. Install the Backend Blueprint with `--store-client`, so its deprecations register exists, and have its `Client release status` set before the first submission: that is the line after which it stops making breaking changes.

Where the product also has a web, mobile or desktop client built with its blueprint, the client repos are independent: each has its own `DESIGN.md` (the later ones take the brand from the first), its own handoff file, and its own pipelines. The backend serves all of them.

---

## The folder version

The downloadable, manual alternative to the installer — the skills, rules and template laid out to copy into a project by hand, with its own README. Build it from this repo:

```bash
npm run package
```

It writes `dist/web-extension-blueprint/` and `dist/web-extension-blueprint-v{version}.zip`, from the same assets the installer uses, so the two never drift apart. Its guide lives in `folder/README.md`.

### Releasing

Every release is a version tag. Pushing one builds the zip and publishes it on the repo's **Releases** page:

```bash
npm version minor        # bumps package.json, commits, and tags v1.2.0 (patch · minor · major)
git push --follow-tags   # pushes the commit and the tag
```

The zip appears under the release's **Assets** as `web-extension-blueprint-v1.2.0.zip` — that is the file to share. The **Source code** archives GitHub adds to every release are the repo itself, not the folder version. A tag that doesn't match the `package.json` version fails the build, so the zip's name always matches its contents.

---

**Requires** Node 22 or later, Claude Code, and for the releases a Chrome Web Store developer account, a Firefox Add-ons account and an Edge Partner Center account (the setup skill's infra guide walks through them; Safari also needs an Apple Developer Program membership and a Mac). The `/design` canvas needs a recent Claude Code signed in to a claude.ai account; `/extension-build` falls back to HTML prototypes without it. **License** proprietary — all rights reserved.
