---
name: extension-issue-fix
description: Resolve issues in the extension's built or integrated screens — bugs, errors in any context, wrong behaviour, design corrections, accessibility and performance problems, browser-specific defects, a target site changing its markup, store-review findings, and changes the backend's client handoff requires — through validation, a failing test first, a surgical fix, doc updates, verification in every target browser, and a note on how the fix ships, since every fix is a new version through the stores' review. Use for anything that changes existing UI, flows, content scripts, permissions or integrations; not for designing, building or integrating a new unit.
---

# Extension Issue Fix

## 0. Project Configuration

Read these from `CLAUDE.md`; ask once for anything missing and offer to record it.

| Key | What it is |
| --- | --- |
| **Project name** · **Audience · client** | Which product and client this repo is. |
| **Backend repo path** | For `pnpm docs:pull` and `pnpm docs:push`. |
| **Staging API URL** | For reproducing and verifying against the real backend. |
| **Dev server command** · **Test commands** · **E2E command** · **API types command** | How to run, test and regenerate. |
| **Browsers** · **Target sites** | Which the product ships and runs on, so a fix is verified on each. |
| **Production status** | Not published yet / published. Decides how a visible change is rolled out, whether a fix goes through a hotfix branch, and whether a permission change is a release decision. |

## 1. Scope

This skill handles any change to what already exists:

- a bug or crash in a screen, flow or integration, found by me, by QA, by Sentry (a surface, a content script or the background), by a store reviewer, or by a user — in one browser or all;
- a **target site changing its markup**, so an anchor is no longer found or content UI mounts in the wrong place;
- a change the backend's **client handoff** message requires, after `/backend-issue-fix` changed or added endpoints;
- a design correction (a screen drifted from its approved export or from `DESIGN.md`), an accessibility, performance or startup problem, a copy change, a store-listing, permission-justification or disclosure correction;
- a small product change to an existing screen that I ask for directly, with the docs updated to match.

It does not design, build or integrate a new unit: that is `/extension-build` and `/extension-integrate`. A "fix" that needs a new screen is routed there. Preparing and submitting a release is `/extension-release`.

I will send one issue or a list. For each, follow Section 3.

## 2. Constraints

These apply to every change.

- **Standards as written:** `coding-standards`, `security`, `testing`, `DESIGN.md`; `ci-cd` where a workflow, `wxt.config.ts` or an `.env` file changes; `analytics` where installed. This skill does not restate them.
- **Diffs are minimal and surgical.** Touch only what the issue needs. No drive-by refactors, no restyling, no "while I'm here".
- **The docs stay true.** A fix that changes what a screen sends or shows updates its screen spec and hands it to the backend; a fix that changes how a screen looks updates its design export; a fix that changes an event updates `docs/analytics-events.md`; a fix that changes an environment variable updates `.env.example` and the infra guide; a fix that changes a protocol capability updates the contract and its tests; a fix that changes a permission, a host, or what the extension stores, reads from pages or sends updates the justifications and disclosures in `docs/release/`, and a fix that touches an anchor refreshes its fixture page. Docs never mention the issue; they describe the corrected behaviour as if it had always been so.
- **The backend's behaviour is not worked around.** If the issue is on the backend, it is logged in `docs/integration-handoff/backend-issues-extension.md` (format in `/extension-integrate` Step 5) and the client waits.
- **Every fix says how it ships.** There is no over-the-air layer: every fix is a new version through the stores' review, which `/extension-release` submits and follows. A fix for the live release goes through a hotfix branch (`ci-cd` → Hotfixes) so unreleased work on `main` does not ship with it; a fix that can wait rides the next release from `main`. Say which before the fix is approved, and whether the fix touches the background, a content script, the manifest or a permission, because those widen what must be verified, label the PR `e2e`, and — for a permission — disable the extension for Chromium users until they accept it, which makes it a release decision, never part of a bug fix.
- **Business rules are not reinterpreted in a fix.** A screen that enforces a limit differently from the rules is corrected to the rules; a rule that seems wrong is raised with me, not changed on the client.
- **No new dependency** outside `tech-stack` without asking. If a fix cannot be made without one, say which requirement it fails and wait.
- **Once the extension is published**, a visible change to a flow users rely on ships behind the unit readiness switch, or a remote flag where the product has one, or in a coordinated release, as `ci-cd` → Releasing alongside the backend says, and never breaks a version that users still run: a browser that is not opened does not update, so old versions keep calling the backend for weeks.

## 3. Per-Issue Workflow

### Step 1 — Validate

1. **Pull first**: `pnpm docs:pull`, commit what it changed, so the spec and handoff file you will touch are the backend's current versions.
2. **Reproduce it**: in mock mode when the issue is client-side, against staging (`pnpm dev:staging`) when it involves the backend, in the browser it was reported in and then in the others. For an error, start from the Sentry event: the stack, the breadcrumbs, the context it came from (which surface, which content script, the background), the browser and its version, the extension version, and for a content script the host's domain. A background or content-script error is reproduced in the browser it came from before it is called fixed. For an anchor that stopped matching, open the live target site and compare its markup with the fixture page. Look at the screen; read the logs and the network calls. An issue that cannot be reproduced is reported as such with what was tried, not fixed speculatively.
3. **Confirm it against the docs**: the design export and `DESIGN.md` (is it a design drift?), the screen spec and app flow (is the screen doing what the flow says?), the business rules (is the limit or state right?), the api-docs (is the backend returning what it documents?).
4. **Classify it:**
   - **Client bug** — the code does not do what the docs say, in one browser or all. Fix here.
   - **Browser defect** — the code is right and a browser, a browser version or a library misbehaves. Fix with the narrowest guard (`import.meta.env.BROWSER`, a feature check), record the version it affects, and note when the guard can be removed.
   - **Target-site change** — the code is right and the site's markup moved. Refresh the fixture page from the live site (recording the date), fix the anchor to the narrowest selector that survives the change, and note the site in the release notes.
   - **Backend bug** — the endpoint does not do what the api-docs say. Log it; stop on this issue.
   - **Doc error** — the docs disagree with each other or with an approved design. Report it to me with the options; do not pick one.
   - **Change request** — the docs and code agree, and I want different behaviour. Treat as Step 2 with the doc updates included.
   - **Handoff adoption** — a backend handoff lists the change. Go to Section 4.

### Step 2 — Plan → **wait for my approval**

For anything beyond a one-line fix, list per issue: the root cause, the fix, the files, the tests to add or change, the docs to update, and anything the fix changes for the user. Collect the questions. Proceed only after my approval; trivial fixes (a typo, a token, a label) may be batched into one approval.

### Step 3 — Failing test first

Write the test that reproduces the issue at the lowest layer that can: a unit test for logic, an integration test for a screen state or an interaction, a content test on the refreshed fixture for an anchor, a Playwright journey only when the issue is in the built extension, a browser capability or the real backend. A background bug gets its test in the background project, against the fake browser. Run it; it fails.

### Step 4 — Fix

Make the change. Run the test; it passes. Run the suite for the unit and for anything that shares the changed component; everything passes. Look at the screen again in every target browser, light and dark, at 200% zoom, in the state that was broken; look at content UI on the real target site.

### Step 5 — Docs

Per the constraints: the screen spec (committed, then `pnpm docs:push --specs {file}` for the file that holds the changed screen — its canonical area file, or its phase file before promotion), the design export, `images.md`, `analytics-events.md`, `.env.example`, the infra guide, `e2e/README.md`, the fixture pages, the justifications and disclosures in `docs/release/` — whichever the fix touched. A backend issue found here goes in the handoff file: write the entry, commit, `pnpm docs:push --handoff`; if the push is refused, pull (the file merges) and push again. `DESIGN.md` itself changes only through `/design-system`.

### Step 6 — Verify

Walk every flow the fix touches, in mock mode or against staging as the issue required, in every target browser and every flow that shares the changed component: Chromium through Playwright on the built extension; Firefox (`wxt dev -b firefox`) and the real target sites from my report — give me the checklist for those and I run it. **Present the walk in the chat**, screen by screen, each item marked verified by Playwright or reported by me, and end with anything that failed. A failure is fixed through Steps 3 and 4 again.

### Step 7 — Report and deliver

For each issue: what it was, the root cause, what changed, the tests added, the docs updated, what the user will notice, and **how it ships** — the next release from `main`, or a hotfix branch from the live tag through `/extension-release`. Then a PR: branch off `origin/main` (or the hotfix branch), push, give me the link. Never create the PR yourself. When I report the merge — in this session or the next — confirm that Pipeline 2's run on it passed, including its Playwright suite and `web-ext lint`.

## 4. Adopting a Backend Handoff

A client handoff message from `/backend-issue-fix` lists items tagged `FIXED` / `NEW` / `UPDATED` / `REMOVED`, grouped by feature area, with an "Action required on the client side" section split into **Breaking** and **Deprecated**, and the list of docs updated.

1. `pnpm docs:pull`, commit what it changed, regenerate `src/shared/api.ts`, and read the handoff with the updated api-docs beside it.
2. **Breaking items first.** For each: the hook, handler, fixture, form schema or state wiring that reads the old shape, changed to the new one; a test proving the new shape; the screen looked at.
3. **Then `UPDATED` and `FIXED` items**: where a client workaround or a pending screen was waiting, finish the wiring and set the handoff file's entry to `verified`.
4. **`NEW` items** are integrated only where an existing screen in this unit already needs them per its spec; a new screen is `/extension-build`'s work.
5. **`REMOVED` items**: delete the hooks and handlers, and anything that called them.
6. **Background behaviour** the handoff notes (state that now changes on its own) is reflected the way `/extension-integrate` Step 3 says.
7. **Deprecations** the handoff lists are adopted so the backend can raise the minimum version later; the report names the first published version that no longer reads the old shape, so the floor for each `X-Platform` value is raised only to that version once every store has published it.
8. Tests, verification against staging (Step 6), and the report, which names the backend commit verified.

## 5. Definition of Done

- [ ] Every issue reproduced and classified; backend issues logged, doc errors reported.
- [ ] Every client fix has a test that failed before and passes after.
- [ ] The suite passes: this unit's and everything sharing the changed components; the fix was looked at in every target browser, and content UI on the real target sites.
- [ ] Every doc the fix touched is updated and pushed, with no mention of the issue.
- [ ] The verification walk was presented and everything it caught is fixed.
- [ ] The report says how each fix ships; the PR link was given; Pipeline 2's run on the merge passed.
