# Web Extension Blueprint

A docs-first workflow for designing, building, integrating and releasing browser extensions with Claude Code. Seven skills and the rules they follow take a product from its requirements to a designed, built, tested, integrated and store-published extension for Chrome, Edge and Firefox (and Safari where the product wants it) — and every screen traces back to a reviewed document and an approved design.

It is a frontend half of a pair. The **Backend Blueprint** writes the product docs this one reads (feature areas, business rules, app flows, api-docs) and reads the screen specs this one writes. The stack is **WXT · React · TypeScript · Manifest V3 · Tailwind CSS · shadcn/ui · TanStack Query**, published through the browser stores.

---

## What's inside

```
web-extension-blueprint/
  skills/                7 skills, one folder each
  rules/                 the rules every project uses, plus one optional
  templates/
    CLAUDE.md            the blocks to add to your project's CLAUDE.md
```

---

## Install

Everything goes into your project's `.claude/` folder, which Claude Code reads. In a terminal, from your project's root — set `BP` to wherever you unzipped this folder:

```bash
BP=~/Downloads/web-extension-blueprint
mkdir -p .claude/rules docs
cp -R "$BP/skills" .claude/
cp "$BP"/rules/*.md .claude/rules/
```

Then:

1. **Remove the optional rule if you don't need it:** `analytics.md` — keep it only if the product tracks user behaviour with PostHog.
2. **Add the CLAUDE.md blocks.** Paste `templates/CLAUDE.md` into your project's `CLAUDE.md` (create the file if there is none). Fill in the audiences and the project configuration now — the backend repo path, the single purpose and the target sites above all — or leave them; each skill asks for anything missing, once.
3. **Check it worked.** Start Claude Code and run `/context` — the rules appear under Memory files (`ci-cd.md` only once a deploy file is touched). Type `/` and the seven skills are listed.

> `.claude` is a hidden folder. To see it in Finder, press **Cmd + Shift + .**

**Commit `.claude/` with your project.** Cloud sessions and scheduled runs only read project skills, and the rules should travel with the code.

---

## Which skill when

**Starting a new extension**

1. `/extension-setup` — the foundation, once: the WXT scaffold with its surfaces and the typed messaging protocol, the extension identity per environment, the session and the network gateway in the background, the data layer, mock mode, the content-script mount, permissions and entry points, tests, the store-publishing pipelines, and the docs sync that pulls the backend's docs into `docs/`.
2. `/design-system` — the design system, once, after the PRD and feature areas exist. Researches the niche's best extensions, proposes a direction for your approval, writes `DESIGN.md` at the project root.
3. `/extension-build` — one feature area per session, in the order the backend's feature areas give, or the build stage of a whole phase in one session once every area is designed. First its screens are designed on the canvas and approved; then they are built for every browser on mock data, tested, and specced for the backend.
4. The backend builds the area from your specs and writes its api-docs.
5. `/extension-integrate` — one feature area per session, or the whole phase: replaces the mocks with the real endpoints and verifies against staging in every browser.
6. `/extension-release` — the first release: the listings, permission justifications, privacy disclosures, screenshots, the browser check, submission to the stores, review, rollout.

**Adding a new phase**

1. The backend's `/feature-requirements` and `/phase-review` produce the phase's FRD, feature files and frontend actions; `pnpm docs:pull` brings them here.
2. `/extension-build` — one feature per session; it reads the existing screens before designing new ones.
3. `/extension-integrate` — one feature per session, or the whole phase.
4. `/extension-release` — a new version through the stores.

**Fixing an issue, or adopting a backend change**

`/extension-issue-fix` — reproduces in the browser it was reported in and then the others, classifies (including a target site that changed its markup), writes the failing test, fixes surgically, updates the docs and disclosures, verifies, says whether the fix rides the next release or goes out as a hotfix, and delivers a PR.

---

## The skills

**`/extension-setup`** — Builds the foundation every screen stands on: the project per the tech stack, `wxt.config.ts` with a development, staging and production identity each and every permission justified, the typed messaging protocol and the background handlers that validate payloads and senders, the session module and the gateway in the background, TanStack Query per surface persisted to session storage, generated API types, the in-process mock mode that serves screens before the backend exists, design tokens, the content-script mount with shadow roots and anchors, permissions, context menus, commands, notifications, alarms and the version check, the identity sign-in flow, Vitest with the fake browser, MSW and Playwright, Sentry, the three CI/CD workflows, the docs sync, and an infra guide for the store accounts, ids, redirect URLs and GitHub steps only you can do.

**`/design-system`** — Writes `DESIGN.md`, the source of truth for how every surface looks, moves, feels and reads: principles, colors and typography with contrast checked, spacing and each surface's shell and sizes, shape and depth, icons and imagery, motion, every component with every state, how content UI sits on a page it did not design, keyboard and commands, screen and state patterns including signed-out and permission-needed, content and voice including the store listing's, accessibility. Researches the best extensions and compact web products in the niche first and proposes a direction for your approval.

**`/extension-build`** — Designs and builds one feature area's screens: every screen every flow needs, on its surface, with its states and any permission it would add, inventoried for your approval; designed on the `/design` canvas and iterated with you; then built for every target browser with realistic mock data through the real data layer, with the anchors, entry points and permission requests wired through the protocol, fixture pages written for every anchor, images sourced and checked, tests and a Playwright journey written, every flow walked on the built extension and on the real target sites against the approved designs, and the screen specs written for the backend.

**`/extension-integrate`** — Replaces the mocks with the backend's real endpoints from the api-docs: an audit first, then the hooks aligned and handlers removed, every documented error and background behaviour wired, the backend-facing capabilities (the identity sign-in flow, minimum version, deletion, the checkout return) and every content-script feature message integrated with the unit that owns them, an orphan-endpoint check, a handoff file for anything the backend must fix, tests on the real shapes, and a walk of every flow against staging in every browser.

**`/extension-issue-fix`** — Resolves anything in what already exists: bugs, errors in any context, browser defects, a target site that changed its markup, design drift, accessibility or performance problems, store-review findings, change requests, and the changes a backend handoff requires. Failing test first, surgical fix, docs and disclosures updated, flows verified in every browser, and a clear answer on how the fix ships — the next release, or a hotfix from the live tag — with any permission change flagged as a release decision.

**`/extension-release`** — Takes a release from "what should ship" to "live for users": the release decided on the tracker with every manifest change called out, the version bumped by PR, release notes and the listing, permission justifications, privacy disclosures and screenshots brought up to date with the code, the production pipeline that smoke-tests and submits to the Chrome Web Store, Firefox Add-ons and Edge Add-ons, a check of the build in real browsers on the real target sites, each store's review followed and its findings classified, a Chrome rollout raised step by step while the crash-free rate holds and halted when it does not, and a record of every release. Covers the first release too, and the optional Safari build.

**`/screen-specs`** — What each screen sends to the backend and shows from it, written from the built screens. The same skill the Backend Blueprint ships.

---

## The rules

Rules load into every Claude Code session and apply to all the code it writes.

| Rule | What it covers |
| --- | --- |
| `documentation-conventions.md` | Which docs arrive from the backend and which this repo owns, how screen specs change hands, the UI tracker, and the six readiness values from `Draft` to `Integrated`. |
| `tech-stack.md` | The tools the extension uses, the browsers it targets, and the environments, identities and stores each build runs with. Anything not listed is proposed in a build plan and approved first. |
| `coding-standards.md` | The architecture, the four contexts and their structure, the messaging protocol and the gateway with offline as a state, mock mode, data layer, session across worker restarts, surfaces and routing, content scripts and the host page, components, state, forms, styling in surfaces and in shadow roots, permissions and the worker's lifecycle, performance, errors and code style. |
| `security.md` | The page as hostile, the background as the trust boundary, CSP, permissions, tokens and sessions, data in the browser, transport, updates and supply chain, secrets, third-party SDKs, payments, logging, personal data, store compliance — and the contract the backend must meet. |
| `testing.md` | The test layers and what each proves, environments and data, fixture pages, the browser matrix, and the rule that every bug gets a test that reproduces it. |
| `ci-cd.md` | PR checks, the automatic staging deploy that produces the zips and a signed Firefox build and moves `staging-current`, the manual production deploy of the verified commit that submits to the three stores, the hotfix branch, and what rollback means when the stores publish and a browser is only ever moved to another version. Loads only when deploy files are touched. |
| `analytics.md` *(optional)* | Explicit, named events with no personal data and no page content, identify and reset, consent, the Firefox data-collection declaration. |

---

**Requires** Claude Code, a WXT project (the setup skill creates it), and for the releases a Chrome Web Store developer account, a Firefox Add-ons account and an Edge Partner Center account (Safari also needs an Apple Developer Program membership and a Mac). The `/design` canvas needs a recent Claude Code signed in to a claude.ai account; the build skill falls back to HTML prototypes without it. Updates replace these files — any edits you made to a skill or rule are overwritten, so keep your own changes in `CLAUDE.md` or new rule files.

© dev-emeka. All rights reserved. Licensed for use in your own projects; not for redistribution.
