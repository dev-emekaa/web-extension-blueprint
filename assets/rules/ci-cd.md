---
paths:
  - ".github/**"
  - "wxt.config.*"
  - "web-ext.config.*"
  - ".env*"
  - "e2e/**"
---

# CI/CD

How code moves from a pull request to users' browsers. The checks and tools are named in `tech-stack`; what each test layer proves is in `testing`; the human steps around a release — listings, disclosures, review outcomes, rollout — are `/extension-release`'s.

## Principles

- **Every merge to `main` deploys to staging automatically.** Production submits only by a manual trigger, after sign-off.
- **Production ships the exact commit verified on staging**, never "latest `main`".
- **Each environment is its own build**, because the API URL, the extension id, the name and the icon are baked in.
- **Every published version passes a store review.** The pipeline submits; a store publishes. Nothing reaches users without that step, and the pipeline never pretends otherwise.
- **The extension never deploys ahead of a backend change it depends on.** The backend deploys first and stays compatible; this client follows.
- **A published version is recalled only by publishing another**, so it is checked in real browsers before submission, rolled out gradually where a store allows, and fixed forward — or rolled back where a store offers it (Chrome and Firefox do; Edge does not).

## Environments

| | Development | Staging | Production |
| --- | --- | --- | --- |
| **Built by** | A developer: `wxt dev` (Chromium) and `wxt dev -b firefox` | Pipeline 2, on every merge | Pipeline 3, manually |
| **Targets** | The one being worked on | `chrome-mv3`, `edge-mv3`, `firefox-mv3`, and `safari-mv3` where Safari is on | The same |
| **Distributed by** | Not distributed | Workflow artifacts: one zip per target to load unpacked, plus the Firefox build signed on the **unlisted** channel for a real install | The Chrome Web Store, Firefox Add-ons and Edge Add-ons listings; the App Store where Safari is on |
| **Identity** | `{name} Dev`, the fixed dev ids | `{name} Staging`, the fixed staging ids | `{name}`, the production ids: the Chrome Web Store's and Edge Add-ons' Chromium ids, and the chosen `gecko.id` |
| **Version** | `package.json` | `{package.json version}.{run number}` — a fourth segment the stores and browsers accept | `package.json` version, which must be higher than the last `v*` tag |
| **Env values** | `.env` | `.env.staging` | `.env.production` |
| **Error tracking** | — | Sentry `staging` | Sentry `production` |

`wxt.config.ts` reads `APP_ENV` and sets the name, icons, ids (`key` for the Chromium builds except production Edge, which takes the store's id, `browser_specific_settings.gecko.id` for Firefox), permissions, CSP and `data_collection_permissions` per environment; the `.env.{mode}` file injects the same environment's constants. Everything builds on Linux runners: there is no platform matrix, only a target matrix. Playwright runs the extension headless through its `chromium` channel, so no display server is needed.

Where the product ships to Safari, Pipeline 3 also produces the `safari-mv3` build; the Xcode wrapper, signing and App Store submission are `/extension-release`'s manual steps on a Mac, because they cannot run on these runners.

## Repo rules

- `main` is protected: no direct pushes, Pipeline 1 required to merge, one approving review where the team has more than one developer.
- Squash merge, so every commit on `main` is one reviewed PR.
- The version in `package.json` is bumped by the PR that prepares a release (`/extension-release` Step 2), never by the pipeline, so every production build is reproducible from its commit.
- Staging deploys run one at a time, in merge order, so `staging-current` never moves backwards.
- Dependabot or Renovate PRs go through Pipeline 1 like any other; one that touches WXT, React or a content-script dependency is labelled `e2e`.
- The `staging` and `production` GitHub environments hold the store credentials (`staging`: the Firefox API key and secret for unlisted signing; `production`: the Chrome Web Store, Firefox Add-ons and Edge Add-ons publishing credentials and the Sentry token), and deploy only from `main` and `hotfix/*`; labelled-PR E2E runs need no secrets.

## Pipelines

| Pipeline | Trigger | What it does | Deploys to |
| --- | --- | --- | --- |
| **1. PR checks** | Pull request opened or updated against `main`; a push to `hotfix/*` | Static checks, unit and integration tests, every target built and its manifest checked, `web-ext lint`; Playwright on an `e2e` label | Nothing |
| **2. Staging deploy** | Push to `main` — every merge — except docs-only changes (`docs/**`, `*.md`); a push to `hotfix/*` | Re-runs the checks, builds every target with the staging identity, signs the Firefox build unlisted, uploads the artifacts, moves `staging-current`, runs the full Playwright suite | Staging |
| **3. Production deploy** | Manual — release owners only | Builds every target from the verified commit, smoke-tests the Chromium build, submits each package to its store, tags the release | The stores' review queues |

### 1. PR checks

**Required to merge.** Target: under ten minutes.

1. Install from the lockfile, cached.
2. In parallel: type check (three projects), lint and format check, unit and integration tests, dependency audit.
3. `wxt build` for every target with `APP_ENV=development`; a script asserts each built manifest's permissions, hosts, CSP and id against the environment's expectations; `web-ext lint` on the Firefox output.
4. On a PR labelled `e2e` (a manifest or permission change, a content-script change, a messaging or session change, a dependency bump of WXT or React, anything risky): the Playwright suite on the `chrome-mv3` build in mock mode.

### 2. Staging deploy

Runs automatically, one at a time in merge order.

1. Re-run Pipeline 1's checks on the merged commit.
2. Build every target with `APP_ENV=staging`, the staging version and `API_MODE=mock` (passthrough sends every integrated unit's traffic to the staging backend); `wxt zip` each.
3. Sign the Firefox zip on the unlisted channel (`web-ext sign --channel unlisted --upload-source-code` with the staging id and the sources zip, so a later review cannot invalidate it), producing a signed `.xpi` a tester installs on a real Firefox with no review wait.
4. Upload the zips and the `.xpi` as the run's artifacts, with a 30-day retention.
5. Move the `staging-current` tag to this commit, only if every target built and signed.
6. Run the full Playwright suite on the staging `chrome-mv3` build, against the staging backend.
7. Upload the source maps to Sentry's `staging` environment.
8. Post the result to the team channel, if there is one: commit, version, pass or fail, the artifact links, and the changes since the last production release.

**After the workflow:** manual QA loads the staging builds (unpacked on Chrome and Edge, the signed `.xpi` on Firefox) and runs the staging checklist on the browser matrix, including an update over the previous staging build. A red Playwright run or a failed checklist means no production submission until fixed. When the release relies on backend changes, tell the backend which backend commit was verified; that is the backend's production sign-off.

### 3. Production deploy

**Trigger:** GitHub → Actions → Deploy Production → Run workflow. Only release owners can run it; running it is the approval. Its one optional input, `commit`, defaults to `staging-current`; a commit SHA releases exactly that commit, and branch names are rejected, so `main` can never be deployed by mistake.

1. Check the actor is a release owner; stop if not. Show on the run page what is shipping: the commit, its message, the version from `package.json`, and the changes since the last release. Stop if the version is not higher than the last `v*` tag.
2. Check out the commit. Tests are not re-run: this commit already passed Pipelines 1 and 2.
3. Build every target with `APP_ENV=production` and `wxt zip` each; `wxt zip -b firefox` also produces the sources zip Firefox Add-ons requires alongside a bundled package (checked once by hand at the first release, so it holds what a reviewer needs to rebuild). A build with mock mode on fails here, by `wxt.config.ts`.
4. Run the Playwright smoke suite on the production `chrome-mv3` build, read-only, with the smoke account.
5. If it fails, stop: nothing was submitted.
6. If it passes: submit each package with `wxt submit` (`--chrome-zip` the Chrome zip, `--firefox-zip` with `--firefox-sources-zip`, `--edge-zip` the Edge zip, from the credentials in the `production` environment) — the Chrome Web Store, Firefox Add-ons on the listed channel, Edge Add-ons; where the Chrome item is eligible for a percentage rollout, `/extension-release` sets the percentage in the dashboard (or through the Web Store API's `setPublishedDeployPercentage`) before publishing; upload the source maps to Sentry's `production` environment; upload the zips as the run's artifacts; add the `vX.Y.Z` tag. The stores review from here; `/extension-release` follows each review and the rollout.

## Hotfixes

`main` usually holds unreleased work, so a fix for the live release cannot simply ride `main`: every unreleased merge would ship with it.

1. Branch `hotfix/vX.Y.Z` from the live release tag and cherry-pick the fix, with a patch version bump; the same fix is merged to `main` through a normal PR.
2. A push to `hotfix/*` runs Pipeline 2 against that branch: checks, staging builds, the Playwright suite, the manual checklist reduced to the flows the fix touched. It never moves `staging-current`.
3. Pipeline 3 is run with that commit's SHA.
4. The branch is deleted once the fix is published and on `main`.

A hotfix still waits for store review. For a fix that cannot wait, the backend's minimum version is the only lever (Rollback below).

## Releasing alongside the backend

- A release that needs new endpoints waits for the backend's production deploy, then submits. Order: backend, then the extension, always.
- A breaking backend change is coordinated through the backend's deprecation machinery: the backend tolerates both shapes across the window, the extension ships a version that reads the new one, and the backend raises the minimum supported version per browser only to that version, once every store has published it and old versions have stopped calling. Browsers update extensions within hours to days, but a browser that is not opened does not update, so the window is weeks, not hours.
- The release notes name the backend commit the release was verified against.

## Tags

| Tag | Moves? | Set by | Purpose |
| --- | --- | --- | --- |
| `staging-current` | Yes, on every successful staging deploy | Pipeline 2 | The commit live on staging |
| `vX.Y.Z` | Never | Pipeline 3 | A permanent record of each submitted release, matching the version users see |

There is no staging branch; the tag is the record. Comparing two `v*` tags shows exactly what changed between releases. In a monorepo the tags carry the client: `staging-current-extension`, `extension-v…`.

**Version numbers** are semver written as the stores accept: up to four dot-separated integers, no prerelease strings. Patch for fixes, minor for features, major for a restructure; the fourth segment is the staging run and never appears in a production version.

## Rollback

| Situation | What can be done | How | Time |
| --- | --- | --- | --- |
| **In review, problem found** | Withdraw it where the store still holds it | Cancel the Chrome submission (deferred publish, or the Web Store API's cancel) and the Edge submission while it is in certification; Firefox publishes on signing, so there is nothing to withdraw there | Minutes |
| **Chrome rollout in progress, problem found** | Stop it reaching more users, or undo it | Hold the deploy percentage; or roll back: the Web Store republishes the last 100% version under a new, higher version number you give it, live within a minute without review, pushed to every Chrome user by auto-update, discarding any pending Chrome submission | Minutes |
| **Published, problem found** | Chrome and Firefox: roll back; Edge: nothing is recalled | Chrome as above; Firefox Add-ons republishes the previous approved version under a new, higher version number, which users receive on their next update check; Edge is fixed forward with a hotfix version through Pipeline 3, through certification again | Minutes for a rollback; hours to days for a hotfix |
| **A version must not be run at all** | Force it out | The backend raises the minimum supported version above it, per browser; the force-update screen opens the store listing | Hours, plus the time browsers take to update |

A rollback consumes a version number on that store: the next release's `package.json` version must exceed every number any store has published, including rollbacks, which Pipeline 3's tag check does not know about, so the release record notes each rollback's number. Edge Add-ons has no rollback: a release there is always fixed forward with a higher version. Because every store reviews a hotfix, the fastest path to users is the smallest, cleanest one.

## Infrastructure

The store developer accounts (Chrome Web Store, Firefox Add-ons, Edge Partner Center, and the Apple Developer Program where Safari is on), the fixed extension ids and their identity redirect URLs registered with the backend per environment, the store API credentials, the GitHub environments and branch protection, the privacy policy and support pages on the product's web domain, and the Sentry projects are the steps only a person can do; `/extension-setup` writes them into `docs/infra/infra-setup.md` so each environment can be set up start to finish from that doc.
