# Tech Stack — WXT · React · TypeScript · Manifest V3

The technologies this extension uses. Use the current stable release of each; the lockfile pins versions, not this file. Anything not listed is added only after approval: the build plan names what the feature needs and recommends the most secure, efficient and standard option for this stack.

The extension is the client only. The backend (NestJS or FastAPI, built with the Backend Blueprint) is the single authority for data, business rules and authentication. The extension holds no business logic, is not the API, and runs no auth of its own. Its background is a **gateway and browser layer** — the session, every backend call, tabs, storage, alarms, notifications, context menus, the identity flow — never a second backend. Chrome, Edge, Firefox and Safari are one client with one codebase; the browsers differ in packaging and a few APIs, never in features.

## Application

| Need | Choice |
| --- | --- |
| Framework | WXT: one codebase, a generated Manifest V3 per browser target, file-based entrypoints, HMR in development, `wxt zip` for the stores |
| Manifest | V3 everywhere. The background is a service worker on Chromium and Safari and an event page on Firefox; WXT emits the right one per target from one `background` entrypoint |
| Browser targets | `chrome-mv3`, `edge-mv3` (the same code, built separately so it carries its own `X-Platform`, id and redirect URL), `firefox-mv3`; `safari-mv3` only when `CLAUDE.md` turns Safari on |
| Language | TypeScript in strict mode, in every context |
| Browser API | The `browser` namespace WXT provides (promise-based, polyfilled where a browser needs it); never `chrome.*` directly |
| UI | React, with React Compiler on through WXT's React module |
| Surfaces | Popup, side panel (Chromium's `sidePanel`, Firefox's `sidebar_action`, one entrypoint), options page, and content UI — each a WXT entrypoint |
| Routing | React Router (library mode) with `createMemoryRouter` inside each surface; a surface opens on the route the background or a deep link names. No URL bar to reflect |
| Styling | Tailwind CSS; content UI styled inside its shadow root so the host page's CSS and ours never meet |
| UI components | shadcn/ui — copied into the repo and owned there, built on Radix primitives; a `Portal` container inside the shadow root for content UI · `lucide-react` icons · `sonner` toasts in the popup and side panel |
| Content UI | WXT's `createShadowRootUi`: a shadow root with the design tokens and the component styles injected, mounted at the anchors `docs/design/surfaces.md` names |
| Long lists | `@tanstack/react-virtual` |
| Animation | Motion (`motion/react`) for transitions and micro-interactions; nothing heavier by default |
| Server data | TanStack Query in each UI surface, its cache persisted to `storage.session` through the persist plugin so a reopened popup paints from the last data while it refetches |
| HTTP | Native `fetch` **in the background**, through the project's `gateway` module; surfaces reach it over typed messaging (`api:fetch`, wrapped by `lib/api.ts`), content scripts only through their feature messages. Nothing else calls the backend |
| API types | `openapi-typescript`, generated from the backend's OpenAPI spec into `src/shared/api.ts`, imported by every context |
| Messaging | `@webext-core/messaging`: one typed protocol in `src/shared/messages.ts` with Zod payload schemas, used by every surface and content script to reach the background, and by the background to push events back |
| Global UI state | Zustand, per surface — only for state the server does not own (panels, drafts, selection) |
| Persisted preferences | WXT's `storage` (`storage.defineItem`) over `storage.local`, with versions and migrations: preferences, dismissed hints, the last route. Never tokens, never server data |
| Session | Access token in `storage.session`, refresh token in `storage.local`; the gateway refreshes (`security` → Tokens and sessions). No surface or content script ever holds a token |
| Sign-in | `browser.identity.launchWebAuthFlow` to the backend's sign-in page with PKCE, returning through the browser's identity redirect URL · the email-and-password form in the popup, calling the backend through the gateway. On Safari, which has no `identity` API, provider sign-in is the email-and-password form or a tab opened on the backend's sign-in URL that redirects to an `https` page on the product's web domain, caught with `tabs.onUpdated` — decided in the setup plan when Safari is on. No embedded sign-in page, no provider SDK |
| Forms · validation | React Hook Form · Zod, through `@hookform/resolvers` |
| Environment | Build-time constants per environment through WXT's `.env.{mode}` files and `import.meta.env`, declared and validated with Zod in `src/shared/env.ts` |
| Mocking | This extension's in-process mock handlers in the background for the build-before-integration phase (`coding-standards` → Mock mode) · the same handlers behind a messaging double in UI tests · MSW (`msw/node`) for the gateway's own tests |
| Dates | `date-fns` |
| Fonts | The system font stack, or font files bundled under `public/fonts/` as `DESIGN.md` names them. Nothing loaded from the network: Manifest V3 forbids remote code and the CSP forbids remote styles |
| Background work | `browser.alarms` for anything periodic (the worker is suspended between events; `setInterval` does not survive), `browser.notifications`, `browser.contextMenus`, `browser.commands` for keyboard shortcuts, `browser.tabs` only with the permission the feature justified |
| Permissions | `activeTab` and optional permissions (`permissions.request` at the moment of need) by default; host permissions only for the origins the product's single purpose needs, listed and justified in `wxt.config.ts` |
| Payments | The backend's hosted checkout, opened in a new tab; the outcome comes from the backend. No checkout inside the extension |
| Real-time | A WebSocket opened by the background's gateway while a surface holds the port, authenticated the way the api-docs say — the Socket.IO `auth` payload, or a single-use ticket fetched over an authenticated request (a browser's `WebSocket` and `EventSource` cannot carry a header) — or SSE over `fetch` with the bearer header and a heartbeat under 30 seconds; forwarded over the port, closed when the last port closes |
| Logging | A thin logger over `console` with levels, scrubbed as `security` says; silent at `info` and below in production |
| Error tracking | `@sentry/browser` in every context, initialised with the extension's release and the context name; source maps uploaded at build time |
| Product analytics *(optional)* | `posthog-js` bundled, with autocapture off and the bundled version pinned by the lockfile. The `analytics` rule applies when it is installed |
| Lint · format · types | ESLint (`typescript-eslint`, `eslint-plugin-react-hooks`, `eslint-plugin-jsx-a11y`, `eslint-plugin-testing-library`) · Prettier with `prettier-plugin-tailwindcss` · `tsc` · `web-ext lint` on the Firefox build |
| Package manager | pnpm, with a committed lockfile; its settings — the supply-chain ones `security` requires — in `pnpm-workspace.yaml` |
| Unit and integration tests | Vitest with WXT's plugin (`WxtVitest`) and `@webext-core/fake-browser` (an in-memory `browser`) · React Testing Library · `@testing-library/user-event` · `@testing-library/jest-dom` |
| E2E and smoke tests | Playwright on the built Chromium extension (persistent context, `--load-extension`), against the popup, the side panel, the options page, and content UI on fixture pages served locally |
| Firefox verification | `web-ext lint` in CI; `web-ext run` and the manual checklist on a real Firefox |
| Store publishing | `wxt submit` for the Chrome Web Store (the Chrome zip), Firefox Add-ons (with the sources zip `wxt zip -b firefox` produces) and Edge Add-ons (the Edge zip) from the production workflow; `web-ext sign --channel unlisted --upload-source-code` for the staging Firefox build. Safari has no automated path: `wxt build -b safari`, then Apple's `safari-web-extension-packager` and App Store submission by hand |
| Bundle analysis | `wxt build --analyze`, run by hand |

**Added when a feature needs them**, through the build plan: `@tanstack/react-table` for data tables, Recharts (shadcn charts) for charts, `browser.offscreen` for work that needs a DOM in the background (clipboard, audio, parsing), `declarativeNetRequest` where the product's purpose is request modification, `browser.scripting` for on-demand injection into `activeTab`, `webextension-polyfill` types where a browser-specific API is used behind a guard.

**Deliberately not used:** remotely hosted code of any kind (scripts, `eval`, `new Function`, remote `importScripts`) — Manifest V3 forbids it and the stores reject it; `externally_connectable` unless the product is built to talk to its own web app, and then only to that origin; `web_accessible_resources` beyond the assets content UI must load; broad host permissions by default; Plasmo (stalled) and bare Vite plugins (no cross-browser or testing story).

## Services

| Need | Local | Staging | Production |
| --- | --- | --- | --- |
| The extension | `wxt dev` for Chromium with HMR, `wxt dev -b firefox` for Firefox, each loading an unpacked build into a fresh profile | Unpacked zips per target from Pipeline 2 as workflow artifacts, plus a Firefox build signed on the **unlisted** channel | Listed on the Chrome Web Store, Firefox Add-ons and Edge Add-ons from Pipeline 3; the App Store where Safari is on |
| Extension identity | `{name} Dev`, a fixed dev id (`key` in the Chromium manifest, `gecko.id` on Firefox) | `{name} Staging`, a fixed staging id, separate from dev | `{name}`, the two store-assigned Chromium ids (the Chrome Web Store's, fixed through its public `key` once known, and Edge Add-ons' own) and the production `gecko.id` the project chooses |
| Backend API | Mock mode, passing un-mocked units through to staging | Staging backend; mock mode with passthrough while any unit is un-integrated | Production backend, direct |
| Sign-in redirect | The dev ids' identity redirect URLs, registered with the staging backend | The staging ids' redirect URLs, registered with the staging backend | The production ids' redirect URLs — one per browser, the Chrome and Edge ids differing — registered with the production backend; on Safari, the `https` page on the product's web domain |
| Error tracking | — | Sentry, `staging` environment | Sentry, `production` environment |
| Analytics *(optional)* | — | PostHog, staging project | PostHog, production project |
| CI/CD | — | GitHub Actions on Linux runners | Same |
| Store listing | — | — | The listing per store from `docs/release/`, with the privacy policy and support pages on the product's web domain |

- **Three extensions side by side.** Development, staging and production builds have different names, icons and ids from one `wxt.config.ts` keyed by `APP_ENV`, so all three install in one browser and a tester never mistakes one for another. The ids are fixed because the sign-in redirect URL is derived from the id and must be registered with the backend.
- **Updates are the stores'.** There is no feed to run: each store pushes a published version to installed browsers on its own schedule, usually within hours, sometimes days. Users who have not opened their browser lag behind, so the backend's minimum-version endpoint still applies. The Chrome Web Store and Firefox Add-ons can roll back by republishing the previous package under a new, higher version; Edge cannot.
- **Environment values are baked in at build.** The API URL and the rest are build-time constants, so each environment is its own build.
- **Every published version passes a store review.** Chrome, Firefox and Edge each review before publishing; what they check is in `security` → Store compliance and `/extension-release`. A review can take hours or days, so a release is never urgent by the time it reaches users.
- **Sentry** receives source maps at build time, scrubs what `security` says, and alerts on new issues per environment.
- **Media** the product uploads goes straight to the backend's bucket with presigned URLs the backend issues, and is served from its CDN; the extension never holds storage credentials.
