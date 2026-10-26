# web-extension-blueprint

**Design, build and release a product's browser extension for Chrome, Edge and Firefox with Claude Code, connected to the real backend.**

Every surface — the popup, the side panel, full pages and the UI it adds to other sites — follows one design system. Each part of the extension is designed for your approval, built and tested on realistic mock data, then connected to the backend's real endpoints as they're ready. Releases go through the Chrome Web Store, Firefox Add-ons and Edge Add-ons, with Safari as an option. Seven Claude Code skills and the rules behind them, for WXT.

**Works with** the [Backend Blueprint](https://github.com/dev-emekaa/backend-blueprint), which it is a client of: it reads the backend's product docs and API docs, and sends back screen specs and issues.

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
| `--analytics` | Also install the analytics rule, for products that track usage with PostHog |
| `--personal` | Install skills to `~/.claude/skills` instead of the project |
| `--force` | Overwrite existing files. Default is to skip them |
| `--dry-run` | Show what would change, write nothing |

There is one stack — WXT, React, TypeScript, Manifest V3, Tailwind CSS, shadcn/ui, TanStack Query — so there is no `--stack` option. Re-running is safe: existing files are skipped and listed.

### What it writes

```
.claude/
  skills/        7 skills
  rules/         6 rules (plus analytics.md with --analytics)
CLAUDE.md        documentation map · audiences · project configuration
docs/
  product-docs/  business-rules/  app-flows/  api-docs/  api-testing/            arrive from the backend
  screen-specs/  design/screens/  integration-handoff/  infra/  release/         written here
```

`docs/feature-areas.md` isn't created — it arrives from the backend.

### After installing

1. Fill in the **audiences** and **project configuration** in `CLAUDE.md` — above all, the path to the backend repo, the extension's single purpose and the sites it works on. The last two decide which permissions it asks for, and the permissions decide what the stores accept. Anything you leave blank, the skill that needs it asks for once.
2. Run `/extension-setup`. It builds the foundation and pulls the backend's docs.
3. Run `/design-system` once the PRD and feature areas are in.
4. Run `/extension-build` for the first feature area.

Commit `.claude/` with your project: cloud sessions and scheduled runs only read project skills. `--personal` is for throwaway work.

### Updating

```bash
npx github:dev-emekaa/web-extension-blueprint init --force
```

- Skills and rules are overwritten, including any edits you made — run it on a clean working tree and read the diff before committing.
- `CLAUDE.md` is never overwritten: it holds your project's values.
- To pin a version: `npx github:dev-emekaa/web-extension-blueprint#v1.0.2 init`. If an update doesn't seem to land: `npm cache clean --force`.

---

## How the extension gets built

Each step needs the one before it; a skill whose inputs don't exist yet stops and says so.

```
From the backend: feature areas · business rules · app flows · (later) API docs
 │
 ├─ /extension-setup      the foundation, once; pulls the backend's docs
 ├─ /design-system        DESIGN.md, once
 └─ /extension-build      per feature area, in order:
      first, once: which surfaces the extension uses, and their shells
      design   every screen it needs, designed for your approval        → Designed
      build    screens on mock data, in every browser, with tests       → Built
               → /screen-specs, sent to the backend                     → Specced
      ↓ once every area is Specced, the backend designs its database, then
        builds the areas in order, writing each one's API docs
    /extension-integrate  real endpoints in place of the mocks          → Integrated
    /extension-release    submitted to the stores, reviewed, rolled out

/extension-issue-fix — any time, for anything already built
```

**Later phases** follow the backend's: the unit becomes a feature, and `/extension-build` reads the existing screens before designing new ones.

### How the extension is put together

The extension holds no business logic — the backend owns the data and the rules. Its background script holds the sign-in and makes every request to the backend; the popup, side panel and pages only ask it. The parts that run inside other people's websites treat those pages as hostile: they read only what their feature needs, keep their UI sealed off from the page, and never hold a token. The extension asks for the fewest permissions its purpose needs, because permissions are both what an attacker can use and what the stores review.

---

## The skills

### `/extension-setup`
The foundation, once: the WXT project with a separate identity for development, staging and production, and every permission with its reason; the background that holds the sign-in and talks to the backend, and the typed messages the rest of the extension uses to reach it; the data layer; the mock API that lets screens run before the backend exists; the UI it adds to other sites; tests, error tracking, the CI/CD pipelines that submit to the stores; the docs sync with the backend repo; and a guide to the store accounts and other steps only you can do.
**Writes** the foundation, `docs/infra/infra-setup.md`

### `/design-system`
How everything looks, moves and reads — in a small popup, a side panel, and on websites the product doesn't control: researches the best extensions in the product's category, proposes a direction for your approval, then writes `DESIGN.md` — colours, type, every component and its states, each surface's sizes, how its UI sits on any page without picking up that page's styles, keyboard shortcuts, motion, writing style and accessibility — and applies it to the code.
**Writes** `DESIGN.md`

### `/extension-build`
Designs and builds one feature area's screens: lists every screen its flows need, on which surface, with its states — signed out, permission needed, nothing found on the page — and any permission it would add; designs them for your approval, then builds them for every browser on realistic mock data with images and tests, saves copies of the target sites' pages to test against, checks every flow on the real sites, and writes the screen specs the backend builds from.
**Writes** the designs, screens, mocks, test pages, tests and screen specs

### `/extension-integrate`
Replaces the mocks with the real backend once its API docs land: checks the screens against the specs, connects every endpoint, handles every documented error, logs anything the backend must fix, and walks every flow on staging in every browser.
**Writes** the integration, the backend-issues file

### `/extension-issue-fix`
Changes what already exists — including a target site changing its pages, or a store review finding: reproduces the issue in every browser, writes a failing test, fixes it, updates the docs it touched (including permission reasons and privacy disclosures), says whether the fix waits for the next release or goes out as a hotfix, flags any permission change, and adopts changes the backend hands over.
**Writes** the fix, its tests and test pages, the docs it changes

### `/extension-release`
Takes a release from "what should ship" to "live for users": bumps the version, brings the release notes, store listings, permission reasons, privacy disclosures and screenshots up to date, runs the pipeline that submits to the three stores, has you check the build in real browsers on the real sites, follows each store's review, and watches crash rates as it rolls out — gradually on Chrome once the extension has enough users, at once elsewhere. Covers the first release, and the optional Safari build.
**Writes** `docs/release/`

### `/screen-specs`
What each screen sends to the backend and shows from it — the same skill the Backend Blueprint ships, run here on the built screens.
**Writes** `docs/screen-specs/`

---

## The rules

Loaded by Claude Code at the start of every session — except `ci-cd.md`, which loads only when deploy files are touched.

| Rule | Covers |
| --- | --- |
| `documentation-conventions.md` | Which docs come from the backend, which this repo owns, and each screen's progress from Draft to Integrated |
| `tech-stack.md` | The tools — WXT, React, Tailwind, shadcn/ui, TanStack Query, Vitest, Playwright, `web-ext`, Sentry — and the environments, browsers and stores |
| `coding-standards.md` | How the extension is structured and written: the background and messaging, data, sign-in, mock mode, surfaces, the UI on other sites, permissions, performance |
| `security.md` | Other sites as hostile, the background as the trust boundary, permissions, tokens, secrets, payments, personal data and store rules — and what the backend must provide |
| `testing.md` | The test layers, saved test pages for the target sites, the browsers to check in, and a test for every bug |
| `ci-cd.md` | PR checks, staging builds, store submission, rollback when the stores control what's published, releasing alongside the backend |
| `analytics.md` *(with `--analytics`)* | Event naming, no personal data or page content, consent |

---

## Working with the backend

- **Sharing docs:** `pnpm docs:pull` brings the backend's docs here; `pnpm docs:push` sends screen specs and backend issues there. `/extension-setup` installs both, reading the backend repo's path from `CLAUDE.md`.
- **Issues:** `/extension-integrate` logs what the backend must fix in `backend-issues-extension.md`; the backend's `/backend-issue-fix` fixes it and hands back a message that `/extension-issue-fix` adopts.
- **The backend's side:** it must meet `security.md` → Backend contract — tokens, sign-in through the browser's own flow, the minimum-version check, account deletion, and real-time connections without cookies. Backend Blueprint v1.4.0 or later does.
- **Before the first release:** install the Backend Blueprint with `--store-client` and set its `Client release status`. From then on it stops making changes that would break versions already installed.
- **Another client for the same product:** each client repo is separate. `/design-system` takes the brand from the first client's `DESIGN.md`, so the product looks like itself everywhere.

---

## The folder version

A downloadable alternative to the installer, built from the same files:

```bash
npm run package
```

It writes `dist/web-extension-blueprint/` and `dist/web-extension-blueprint-v{version}.zip`. Its guide is `folder/README.md`.

### Releasing

```bash
npm version patch        # bumps package.json, commits and tags (patch · minor · major)
git push --follow-tags   # pushes the commit and the tag
```

The zip appears under the release's **Assets** as `web-extension-blueprint-v{version}.zip` — that is the file to share, not the Source code archives. A tag that doesn't match `package.json` fails the build.

---

## Requires

- Node 22 or later, and Claude Code
- For releases: a Chrome Web Store developer account, a Firefox Add-ons account and an Edge Partner Center account — `/extension-setup`'s guide walks through them. Safari also needs an Apple Developer Program membership and a Mac
- For design research: Mobbin, connected through its MCP — `/design-system` and `/extension-build` use it for references
- For the `/design` canvas: a recent Claude Code signed in to a claude.ai account — without it, `/extension-build` designs as HTML prototypes instead

**License** proprietary — all rights reserved.
