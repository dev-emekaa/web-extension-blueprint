# Testing

How this extension is tested. The tools are named in `tech-stack`; how tests are set up in this stack is in `coding-standards`; which pipeline runs what is in `ci-cd`. What each build, integration or fix must cover is set by those skills.

## Principles

- **Test the way users use the extension:** find elements by role, label or text, interact, assert on what is visible — in a surface, or in content UI on a page. Never on component internals, state or implementation details.
- **Mock at the boundary the context has.** A surface's boundary is messaging, so its tests replace the messaging client with a double backed by the shared mock handlers. The background's boundary is the network, so the gateway's tests use MSW. A content script's boundary is the page, so its tests run it against a fixture page. Never mock our own modules, hooks or child components.
- **Most confidence comes from integration tests.** A small E2E suite guards the critical journeys on the built extension in Chromium. Firefox is verified by `web-ext lint` and a person, because Playwright cannot load extensions there. Production only ever receives a commit already verified on staging, and every published version also passes a store review, so a release is never a surprise to anyone.

## Layers

| Layer | Proves | Share | Runs against | When |
| --- | --- | --- | --- | --- |
| **Static** | Types in every context, lint, formatting, bad test patterns, accessibility lint, the background imports no DOM, the manifest (`web-ext lint` on the Firefox build; a check that the Chromium manifest's permissions, CSP and ids match `wxt.config.ts` per environment) | Always on | The source and the built manifests | Before commit, and in every pipeline |
| **Unit** | Pure logic: utils, formatters, Zod schemas, query-key builders, store reducers, the page-data parsers; the background's modules — the gateway, the session, the message handlers and their sender checks, deep-input validation, alarms | ~20–25% | Nothing; MSW for the gateway; the fake browser's storage | Locally, on every PR and merge |
| **Integration** | A whole screen rendered with its real children, providers, router and forms, over the messaging double; a content script run against a fixture page, mounting its UI and sending its messages | ~55–65% | jsdom and the fake browser | Locally, on every PR and merge |
| **E2E** | 5–20 critical journeys through the built extension in Chromium: surfaces, content UI on fixture pages, the sign-in flow against a stub | ~10–15% | The built `chrome-mv3` output | On PRs labelled `e2e`; the full suite on the staging build; smoke on the production build |
| **Manual QA** | The integrated flows a person should walk once, in real browsers on real sites | — | The staging build | Before every production submission, from a 10–15 minute checklist and the browser matrix |

Coverage is read to find untested logic, never enforced as a percentage.

**Deliberately not used:** snapshot tests of component trees; unit tests of presentational components with no logic; Puppeteer and Selenium (Playwright covers E2E); E2E against live third-party sites in CI (they change, rate-limit and forbid automation — content-script journeys run on fixture pages that copy the real site's relevant markup); mocking our own modules.

## Environments and data

| | Unit | Integration | E2E | Manual QA |
| --- | --- | --- | --- | --- |
| **Backend** | None; MSW for the gateway | The messaging double over the shared handlers | PR build: mock mode, passing through to staging · Staging build: the real backend, on its seed | Staging, on its seed |
| **Session** | The fake browser's `storage.session` and `storage.local` | A signed-in `me` through `renderWithProviders` | The mock auth until the sign-in unit is integrated; after, the email-and-password form with the seeded test users (the identity flow is in the browser check) | Seeded test users, both sign-in paths |
| **Data** | Built per test | Fixtures from `features/*/mocks/`, overridden per test through the double | The backend's seed | The backend's seed |
| **Pages** | Fixture HTML strings | `src/test/fixtures/*.html` in jsdom | `e2e/fixtures/` served by Playwright's web server | The real sites the product targets |
| **Browser** | The fake browser | The fake browser | Playwright's Chromium, a fresh profile per run | The browser matrix below |

- **Automated tests never touch production data.** Production smoke tests are read-only, with a dedicated low-privilege account, on the production build loaded unpacked.
- **E2E tests depend on the backend's seed.** They use the records the seed marks as disposable and clean up what they create; they never reseed staging on their own, because manual QA and the backend's own verification share it. A reseed is asked for, through the backend's reseed dev-tool endpoint, when the seed has drifted.
- **Integration tests start from the shared mock handlers** so a screen is tested against the same shapes it was built on; a test that needs a different response overrides the one handler it needs. Once a unit is integrated and its handlers are removed, its tests answer its endpoints from the updated fixtures through the same per-test overrides.
- **Fixture pages are copies, kept honest.** Each fixture under `e2e/fixtures/` records the site and date it was taken from; when a host site changes its markup, the fixture is refreshed in the same PR as the anchor fix, so the tests prove the fix against the markup that broke it.
- **E2E runs use a fresh profile** per run (a temporary `userDataDir`), so no run inherits another's session, preferences or storage.

## Unit tests

- **No network, timers or randomness** unless faked. Schemas, formatters and key builders run for real; the background's modules run against the fake browser, reset between tests, with `browser.alarms` and `browser.identity` faked where a test needs them.
- **The gateway** is tested against MSW with the fixtures: headers added (including `X-Platform` and `X-App-Version`), `credentials: 'omit'`, refresh single-flight under the lock, retry once, sign-out on a failed refresh, `ApiError` shapes, network failure mapped to offline.
- **The session** is tested across a simulated worker restart: tokens written, the module re-imported with storage intact, the status still signed in; a missing refresh token is signed out; a rejected one clears everything in the documented order.
- **Every message handler** is tested through the protocol: a valid payload reaches the module, an invalid one is rejected before it does, a sender that is not this extension is refused, a surface-only message from a content script is refused, a content message from an undeclared host is refused.
- **Deep inputs** — `?route=`, context-menu data, notification payloads, identity return URLs with a wrong `state` — are tested with malformed and hostile values.
- **Page-data parsers** are tested with oversized, malformed and script-bearing input.
- **Group by the function under test, one behaviour per test**, named for the behaviour: `refuses a content message from a host outside the declared matches`.

## Integration tests

- **Render the screen through `renderWithProviders`**, with a fresh `QueryClient` per test, retries off and no persister, the memory router at the screen's route, and the messaging double answering from the fixtures.
- **Drive it as a user would**: `user-event`, not `fireEvent`; `findBy*` for what appears after a request; `getByRole` before `getByTestId`, and a test id only where no accessible query exists. Keyboard paths are tested too: Tab order, Enter, Escape.
- **Cover the screen's states**, not only the happy one: loading, empty, error, offline, the permission-needed state, the signed-out state, and every conditional state the screen spec lists. The popup's screens are rendered at the popup's width.
- **Forms:** submit valid data and assert the request the double received; submit invalid data and assert the field errors; return a 422 from the double and assert it maps onto the fields.
- **Background events** (`session:changed`, `cache:invalidate`, a version state) are fired through the double and the screen's response asserted.
- **Content scripts:** load the fixture page, run `main` with a fake `ctx`, and assert the anchor was found, the UI mounted once (and once only after a DOM mutation), the message sent carries the parsed data and nothing else, the styles stayed inside the shadow root, and `ctx.onInvalidated` unmounts everything. A fixture with no anchor asserts nothing mounted and no message sent.
- **Each test sets up its own double and leaves no shared state.** Tests pass in any order.

## E2E tests

- **Critical journeys only:** open the popup and see the signed-out state; sign in with email and password (the identity flow is in the browser check, not in E2E); the primary flow of each feature area in its surface; content UI appearing on its fixture page and completing its action; a context-menu or shortcut entry opening the right surface at the right route; the options page saving a preference; the update banner and the force-update screen against a stubbed version endpoint; and anything that would make the product unusable if broken. The list is kept in `e2e/README.md` with the reason each journey is there.
- **Against the built extension, never `wxt dev`:** Playwright launches a persistent Chromium context with `--load-extension` on the `chrome-mv3` build, headless through the `chromium` channel, reaches the background through `context.serviceWorkers()`, and opens surfaces at `chrome-extension://{id}/…`. Fixture pages are served by Playwright's web server. The service worker suspends after ~30 seconds idle; a test that drives it directly handles the restart rather than assuming a live handle.
- **Chromium only, by the tool's limits.** Firefox and Safari are not in E2E; their verification is the manual checklist and `web-ext lint`.
- **Stable by design:** role-based selectors, `expect` with auto-waiting, no fixed sleeps, a fresh profile per run. A flaky test is fixed, not retried until it passes.

## Smoke tests

- **Three to five read-only flows** on the production build loaded unpacked: the popup opens and paints within the budget, sign-in works, the main screens render their data, content UI appears on its fixture page, the version check completes. Until the sign-in unit is integrated the sign-in is the mock auth's; after, it is the smoke account against the real backend.
- **Run before every submission** — on the staging build after a staging deploy, and on the production build before it is submitted to the stores — and a failure stops the release as `ci-cd` says.

## Browser matrix

The manual checklist runs on, at least:

| Browser | Why |
| --- | --- |
| Chrome, current stable | The largest install base; the Chromium build; the popup, side panel and `chromiumapp.org` redirect |
| Edge, current stable | The `edge-mv3` build, its own store, id and redirect URL, a different side panel chrome |
| Firefox, current release | The event-page background, `sidebar_action`, the `allizom.org` redirect, `data_collection_permissions` in the install prompt, `web-ext run` |
| Safari, current | Only when `CLAUDE.md` turns Safari on: the converted app, its permissions UI |

On each: a fresh install (the install prompt's permissions and disclosures read as the listing says), an update from the previous version with the session surviving, both sign-in paths, every content-UI site the product targets on its real pages, light and dark following the browser theme, 200% zoom and the largest font setting, the popup at its minimum and maximum height, a keyboard-only pass of the primary flow, the browser restarted mid-session, the network cut mid-flow, and the extension disabled and re-enabled.

## When something breaks

- **A bug found anywhere gets a test that reproduces it**, at the lowest layer that can (usually integration; a background bug at unit; a host-site breakage as a refreshed fixture plus a content test). The test fails before the fix, passes after, and ships in the same PR.
- **A bug that manual QA catches twice becomes an automated test.**
- **A host site changing its markup is a bug class of its own**: the fixture is refreshed from the live page, the anchor fixed, the content test proves it, and the release note says which site.
- **An error in Sentry gets a test when it can be reproduced**, and a note on the release when it cannot; a browser-specific error is reproduced in that browser before it is called fixed.
- **A failing test is never skipped, deleted or loosened** to turn a pipeline green unless the PR states why.
