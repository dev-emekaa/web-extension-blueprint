# Security

Security policy for this extension. The backend enforces every rule about who may do what; this extension's job is to never leak, never trust what it did not verify, and never take more of the browser than it needs. An extension differs from a web page in three ways that shape everything here: it runs with privileges a page never gets, across every site the user visits; part of it runs *inside* pages that may be hostile; and once published it is reviewed by stores whose rules are themselves a security requirement. The mechanism in this stack is in `coding-standards`; this file is the policy.

## The page is hostile

Content scripts run inside third-party pages. Everything there — the DOM, its text, its attributes, its events, its globals — is attacker-controlled.

- **Content scripts read, they do not trust.** Anything read from a page is parsed with a Zod schema before it crosses messaging, bounded in length, and treated as text, never as HTML or a URL to open. A page cannot make the extension do anything the feature did not design for.
- **Content scripts hold nothing.** No tokens, no user data beyond what the mounted UI shows, no secrets, no API URLs that matter. Compromising a content script yields the page it is already on and nothing else.
- **The isolated world is the boundary.** Content scripts never run in the page's main world, never read its cookies, storage or globals, and never expose functions to it. A feature that needs the main world (`world: 'MAIN'`) is a build-plan decision with its own review, and what it injects is a leaf: it can post messages to the isolated world and receive nothing privileged back.
- **Content UI is sealed.** It renders inside a shadow root with its own styles; it never injects into the page's DOM beyond its own hosts, never writes page-controlled text into its own DOM as HTML, and uses the page's text only through React's text rendering.
- **Narrow matches.** A content script runs only on the hosts the feature needs, declared as the narrowest match patterns that work; `<all_urls>` is a product decision for products whose single purpose is every page, approved in the setup plan, and never a convenience.
- **Nothing is `web_accessible_resources`** unless content UI must load it (fonts, an image); what is listed is restricted to the matching hosts, never `<all_urls>`, because a listed resource lets pages detect and fingerprint the extension.

## The background is the trust boundary

- **Every message handler validates its payload** with the protocol's Zod schema before running, and checks the sender: `sender.id` is this extension; a message that only surfaces may send comes from an extension page — `sender.url` starts with `runtime.getURL('')` (`sender.tab` is present for the full-tab page and absent for the popup and side panel, so it is not the test); a message a content script may send comes from a tab whose URL matches the content script's declared hosts. A message that fails either check is dropped and logged, never answered.
- **Content scripts call features, not the API.** `api:fetch` is a surface-only message; a content script sends the feature's named messages, each listed in the protocol with the hosts allowed to send it, and the background decides which endpoint that means and with what parameters. A hostile page can then ask only for what the feature designed.
- **The protocol exposes capabilities, never primitives.** `tabs:open` takes an `https:` URL and opens it; nothing exposes `browser.tabs` wholesale, nothing exposes `fetch` with an arbitrary URL, nothing exposes storage by key. A capability that is not in the protocol does not exist.
- **Nothing listens from outside.** `externally_connectable` is absent, so no web page can message the extension; where the product's own web app must, it is listed by exact origin, its messages go through the same protocol and validation, and nothing it sends is trusted more than a content script's. `runtime.onMessageExternal` is never registered otherwise.
- **The background runs no remote code.** No `eval`, no `new Function`, no `importScripts` of a remote URL, no script tags built at runtime, no "plugins" fetched from the backend. Manifest V3 forbids it and every store rejects it; this rule exists so nobody tries to work around that.

## Content Security Policy and the surfaces

- **The extension pages' CSP** is the strict Manifest V3 default made stricter: `script-src 'self'`, `object-src 'none'`, no `unsafe-eval`, no `unsafe-inline` for scripts; `style-src 'self' 'unsafe-inline'` (no remote styles; the inline allowance exists because shadcn's Radix primitives and the dev server inject `<style>` elements, and Manifest V3 restricts only scripts); images from `'self'`, `data:` and the backend's CDN; `connect-src` the API origin, Sentry, and analytics where installed, per environment. It is set in `wxt.config.ts` per environment and a test asserts it on the built manifest.
- **Surfaces load only bundled code.** No third party's page is ever rendered in a surface — no iframe of a sign-in page, a checkout, or documentation; those open in a tab. No `<iframe>` at all unless a feature's build plan justifies one with `sandbox`.
- **Surfaces render text, not markup.** `dangerouslySetInnerHTML` is forbidden; rich content from the backend goes through a sanitiser configured with an allowlist, once, in one component.
- **Deep inputs are validated.** A route carried by `?route=`, a context-menu selection, a notification's payload, an omnibox query: each is parsed by the background with a schema before any surface sees it. An invalid one opens the home route.

## Permissions

Permissions are the attack surface and the review surface at once; every one must be earned.

- **The default is `activeTab`**, which grants the current page only after the user acts on the extension, and **optional permissions** requested at the moment of need from the surface's own click handler (never relayed through messaging, which loses the user-action status on Firefox and after the first `await` on Chromium), with a sentence in the UI saying why. Install-time permissions are only those the product cannot function without.
- **Host permissions are listed origins**, each justified in `wxt.config.ts` and in `docs/release/`. The API origin is always one of them (per environment), so the background's requests to the backend bypass CORS and no other origin's rules apply; `<all_urls>` and wildcard-TLD patterns need the product's single purpose to be "every page" and the setup plan's approval.
- **Dangerous permissions are opt-in by plan**: `tabs` (every URL the user visits), `history`, `bookmarks`, `cookies`, `webRequest`, `downloads`, `clipboardRead`, `nativeMessaging`, `debugger`. None is a default, none is added for convenience, and each one added changes the store disclosures, which `/extension-release` updates before the version ships.
- **A new permission in an update** disables the extension until the user accepts it, on Chromium. A version that adds one is a product decision, announced in the release notes, and never bundled with a bug fix users need.
- **Permissions are checked, not assumed.** A feature behind an optional permission checks `permissions.contains` before it runs and renders the permission-needed state when it is missing; a permission the user revokes in the browser is handled the same way, through `permissions.onRemoved`.

## Tokens and sessions

- **The access token lives in `storage.session`** (memory-backed, cleared when the browser closes, readable only by trusted extension contexts by default — the access level is never raised to content scripts). It is a short-lived JWT the backend issues.
- **The refresh token lives in `storage.local`**, because an extension has no OS secret store. The browser isolates extension storage from web pages and from other extensions; the remaining threat is software on the machine with the user's privileges, which is the same threat a browser's own cookies face. The mitigations are the backend's rotation on every refresh with a short grace window, revocation on sign-out, and the minimal permissions above. The token is never copied anywhere else, never logged, never sent to Sentry, and never read by a surface or a content script: only `background/session/` touches it.
- **Sign-in never embeds a page.** `browser.identity.launchWebAuthFlow` opens the backend's sign-in page in a browser-owned window, with a PKCE challenge and a random `state` that the background checks on return; the backend redirects to the browser's identity redirect URL — derived from the extension's fixed id, one per browser (the Chrome and Edge ids differ in production), registered with the backend per environment — with a one-time code, never a token. Safari has no `identity` API: there the flow runs in a tab and returns to an `https` page on the product's web domain, registered the same way. Email-and-password sign-in posts through the gateway from the popup's own form. No provider SDK, no provider page in a surface.
- **A refresh failure signs out** and clears everything the session touched (`coding-standards` → Session). Two refreshes cannot race: the background takes a lock.
- **Sign-out revokes on the backend first**, then clears locally, then tells every surface and content script, so no stale UI shows another user's data.

## Data in the browser

What may be stored where, and for how long.

| Data | Where | Rule |
| --- | --- | --- |
| Access token | `storage.session` | Trusted contexts only; gone when the browser closes |
| Refresh token | `storage.local` | Only `background/session/` reads it; removed on sign-out and on account deletion |
| Persisted query cache | `storage.session` | Only data a signed-in user may see on any screen; keyed by user id; dropped on sign-out and sign-in. Never `security`-sensitive fields (payment details, identity documents, health data) |
| Preferences | `storage.local` | Theme, layout, dismissed hints, the last route. Nothing personal |
| Mock state (non-production) | `storage.session` | Never in production builds |
| Page data read by content scripts | Nowhere | Parsed, sent, forgotten; never persisted on the client |
| Files the user saves | The browser's downloads, through `browser.downloads` with a filename the extension chose | Never a path the page suggested |
| Anything else | Nowhere | A new persisted item is a build-plan decision with the reason it is needed |

- **`storage.sync` is not used** for anything personal: it is synced through the user's browser account to every device, and its limits are small. Preferences may sync when the product wants that and `CLAUDE.md` says so.
- **Uninstall leaves nothing sensitive behind** because nothing sensitive was ever written in the clear; `runtime.setUninstallURL` may point at the product's feedback page, and carries no identifier.

## Transport and the gateway

- **HTTPS only**, to the API origin `env.ts` names for the environment. The CSP's `connect-src` enforces it; a request to another origin fails before it leaves.
- **Every request carries `X-Platform` and `X-App-Version`** — `extension-chrome`, `extension-firefox`, `extension-edge` or `extension-safari` from the build target, and the version from `runtime.getManifest().version` — so the backend knows which builds still call an old shape.
- **The bearer header is the only credential.** The gateway never sends cookies (`credentials: 'omit'`): with host permission on the API origin the browser would attach the web app's cookies, and the extension must never ride another client's session.
- **Responses are validated at the edge** where the api-docs give a schema; the generated types are the contract and a response that does not match is an `ApiError`, not a crash.
- **Downloads and uploads** carry no credentials in URLs: the backend issues a short-lived presigned URL per request, the extension uses it once.

## Updates and supply chain

- **The stores deliver updates and verify packages**; the extension never updates itself, never loads code from anywhere but its package, and never changes behaviour on a remote flag in a way that adds capability (a flag may hide or show what the package already contains, never fetch what it does not).
- **Store accounts are the signing keys.** Two-factor authentication on every store account, the API credentials the production workflow uses scoped to publishing this item, held in the GitHub `production` environment, rotated when someone with access leaves. The Chromium `key` that fixes the dev and staging ids is a public key and may live in the repo; the production item's key is the store's.
- **New dependencies wait:** pnpm's `minimumReleaseAge` is set to seven days in `pnpm-workspace.yaml`, so a package version published an hour ago cannot be installed, which blocks most hijacked releases. Every dependency ships to users as part of a package the stores review; a compromised one is a compromised extension.
- **`pnpm audit` runs in CI**; a high or critical finding blocks the merge unless the PR records why it does not apply.
- **CI workflows pin actions to a commit SHA**, give each job the least token permission, and expose store credentials only to the production deploy job on `main` and `hotfix/*`.
- **Secret scanning with push protection** is on for the repository.

## Secrets and environment

- **The extension holds no secrets.** Everything in the package is readable by anyone who installs it: the API URL, the Sentry DSN and an analytics key are public values by design, and the backend treats them as such. Anything that would be a secret (a payment key, a provider secret, a signing key) lives in the backend or the store account, never in the package.
- **Environment values come from `.env.{mode}` files** read at build time and validated in `shared/env.ts`; production values are in the repo because they are public, and the build refuses a production package whose `APP_ENV` and API URL disagree.

## Third-party SDKs

- **Each SDK is approved in a build plan** with what it collects and what it adds to the disclosures; bundled from the lockfile, never loaded from a CDN (the CSP would block it and the stores would reject it).
- **Sentry is declared and switchable.** Error reports are data collection: the Firefox manifest declares `technicalAndInteraction` as an optional data-collection permission in every build, Sentry initialises on Firefox only while that permission is granted (`permissions.getAll().data_collection`, feature-detected) and stops on `permissions.onRemoved`, and the Chrome and Edge privacy forms disclose it.
- **Sentry receives no personal data**: `sendDefaultPii` off, the user identified by the backend's id only, request bodies and page URLs scrubbed — a content-script error carries the host's registrable domain, never the page — and breadcrumbs filtered.
- **Analytics carries no personal data and no page content**: no URLs, no selections, no page text in events or properties; the user is identified by the backend's id only after sign-in and reset on sign-out. Consent is asked where the product's users require it, and nothing is sent before it. The `analytics` rule has the detail.

## Payments

- **Payments happen in the backend's hosted checkout in a tab**, never inside a surface or a page; the outcome comes from the backend's state (polled or pushed), never from a URL parameter.
- **Entitlements are the backend's.** The extension renders what `me` says the user may do and never decides on its own.

## Errors and logging

- **Logs carry no secrets, tokens, personal data or page content.** `logger.ts` scrubs known keys (`authorization`, `token`, `password`, `email`, `selection`, `text`) and truncates values; production logs at `warn` and above only.
- **Error messages shown to users are the backend's message keys or `DESIGN.md`'s copy**, never a stack trace, a URL, or a page's text echoed back.

## Personal data

- **Collect only what the screen spec asks for**, and show only what the user's role needs for the action in front of them.
- **Page data is the user's, not the product's.** A feature reads from a page only what its purpose needs, sends it only to the backend, and the disclosures say so. Nothing is read from pages the user is not acting on.
- **No personal data in links, analytics, logs, Sentry or error messages.**
- **Account deletion is in the extension**: the flow renders what the backend provides, with the backend's re-authentication step, and sign-out and a local wipe follow. **Data export** opens the backend's signed link in a tab.

## Store compliance

The stores' policies are security requirements: a listing that violates them is removed, and an extension that overreaches is the kind they exist to stop.

- **One purpose.** The extension does one thing the listing states, and every permission, every host, every byte of data collected is necessary to that purpose. Data collected for the purpose is used for nothing else.
- **Disclosed and justified.** Every permission has a written justification; every kind of data collected or transmitted is disclosed in each store's privacy form and the Firefox manifest's `data_collection_permissions` (`technicalAndInteraction` as an optional entry wherever Sentry ships, which is every build of this blueprint; `none` only for a build with no error tracking and no analytics); the privacy policy on the product's web domain says the same things in the same words. A change to what is collected is disclosed before the version that makes it ships, and users are told in the extension.
- **Nothing hidden.** No obfuscated code, no behaviour that differs for the reviewer, no remote switches that add capability, no affiliate or ad injection, no changing the user's search, home page or tabs without an explicit action. Minified code is fine; code the reviewer cannot read is not.
- **Reputation is the product's.** A takedown removes the extension from every user at once and can end the publisher account. `/extension-release` runs the compliance checklist before every submission for this reason.

## Backend contract

What this client assumes the backend provides. Items 1, 3 and 4 are lines in the Backend Blueprint's security and services rules (v1.4.0 and later); 2 and 6 are in its build skill; 5 is what its api-docs document per endpoint; 7 is its security rule's real-time line (the handshake payload or the single-use ticket). A backend that differs is a conflict to raise before the first integration, never a workaround on this side.

1. **Tokens in the body:** for a request carrying the platform header (`X-Platform: extension-chrome`, `extension-firefox`, `extension-edge` or `extension-safari`), sign-in and refresh return the access and refresh tokens in the JSON response, not as cookies; sign-out revokes the session. The access token is a short-lived JWT, the refresh token is opaque and rotated on every refresh, with the grace window of about ten seconds for the token just rotated.
2. **Minimum supported version:** an endpoint callable without a session, which the extension calls on startup and on an alarm with its platform and version, returning `supported`, `soft` or `force`, held per platform — one floor per browser, because each store publishes on its own schedule. The place a `force` sends the user is the store listing, in `CLAUDE.md`.
3. **Native sign-in with a registered redirect URL:** the same OAuth flow the backend runs for its native clients — the extension opens the backend's sign-in URL with a PKCE challenge and `state`; the backend handles the provider and redirects with a one-time code to a redirect the client registered, which for this client is the browser's identity redirect URL (`https://{id}.chromiumapp.org/` on Chrome and on Edge, each with its own id, the `extensions.allizom.org` URL Firefox derives, and on Safari an `https` page on the product's web domain) per environment, not a custom scheme; the extension exchanges the code and verifier for its tokens.
4. **Account deletion:** an endpoint, behind re-authentication, that the extension's deletion flow calls.
5. **Errors as documented:** 401 means no valid session (refresh or sign in), 403 means a valid session without permission (never refresh), and validation failures and every client-actionable error carry the shape and stable code the api-docs document for that endpoint.
6. **Additive changes only** once this client is published, or a coordinated deploy with a deprecation window for a breaking one, as the backend's build rules say; the backend reads the platform and version headers to know which versions still call an old shape, and raises its minimum only to a version every store has published.
7. **Real-time connections are reachable without cookies or headers:** a WebSocket accepts the token in the handshake payload (Socket.IO's `auth`) or a single-use ticket issued to an authenticated request, as the backend's security rule provides for a client with neither a header nor cookies; SSE accepts the bearer header on a plain `fetch`. CORS is not involved: the extension holds a host permission for the API origin, so the browser sends its requests without a preflight and the backend's credentialed-CORS rules for the web client do not apply to it.
