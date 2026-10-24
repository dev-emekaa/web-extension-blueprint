# Coding Standards — WXT · React · TypeScript · Manifest V3

How code is written in this extension. The tools are named in `tech-stack`; security policy is in `security`; what tests must prove is in `testing`. Where a rule here touches those, it gives the mechanism in this stack and leaves the policy to them.

## 1. Architecture

Four contexts, one direction of trust. The surfaces draw. Content scripts observe pages they do not own. The background holds the session and talks to the backend. The backend decides everything else.

| Context | Runs in | Owns | Never |
| --- | --- | --- | --- |
| **Background** (service worker on Chromium and Safari; event page on Firefox) | The browser, with the extension's privileges | The session and its tokens, every API call (the gateway), mock mode, alarms, notifications, context menus, shortcuts, the identity flow, tab and window actions, logging | Business rules, UI, data the backend owns, a global variable it expects to survive |
| **Surfaces** — popup, side panel, options page, full-tab pages | Extension pages (`chrome-extension://`, `moz-extension://`) | React, routing, TanStack Query, forms, UI state | Tokens, network calls of their own, the host page |
| **Content scripts** | Inside third-party pages, in the isolated world | Finding the anchors a feature attaches to, rendering content UI in a shadow root, sending validated messages | Tokens, network calls, trusting anything the page gives them, running on pages they were not granted |
| **Backend** | — | Data, validation, business rules, authentication, authorization, issuing and rotating tokens, entitlements | — |

Business rules are enforced by the backend. The extension repeats a rule only where the user would otherwise wait for a round trip to learn it (a form field's limit, a disabled action), and that repetition is a convenience, never the enforcement.

### Data flow

One path. A screen renders a query hook → the hook calls `api.fetch` from `lib/api.ts` → the typed message reaches the background → the background's `gateway` adds the bearer header and calls the backend (or dispatches to a mock handler) → the response returns as plain data → the cache updates and the screen re-renders.

**Rules:**

1. Default `staleTime` above zero (60 seconds unless an entity says otherwise). Refetch on window focus is on in the side panel and options page; the popup refetches on open.
2. One `QueryClient` per surface, in `lib/query-client.ts`, persisted to `storage.session` so a reopened popup paints from the last data while it refetches; `onlineManager` fed by `navigator.onLine` and the gateway's offline events.
3. Query keys are defined once per entity, in the feature's `api/keys.ts`, and used by every hook and every surface.
4. Mutations go through TanStack Query to the gateway, then invalidate the affected keys — in every open surface, through the background's `cache:invalidate` event.
5. No `useEffect` to fetch. Data comes from a query hook.
6. **Offline is a state, not an error.** A query with cached data shows it with the offline indicator; a query with none shows the offline state; a mutation fails fast with a retry toast. An offline queue for mutations is a product decision made in a build plan, never a default.
7. **Messaging carries requests, not data models.** The protocol exposes a small, stable set of capabilities; it never grows a message per screen.
8. **Content scripts ask for features, not endpoints.** A content script never calls `api.fetch`; it sends the feature's named message (`lookup:term`, `clip:save`) and the background decides which endpoint that means. A hostile page can then request only what the feature allows.

## 2. Project structure

```
wxt.config.ts                   manifest per target: name, ids, permissions with justifications, CSP, keyed by APP_ENV
web-ext.config.ts               Firefox run and sign settings
.env · .env.staging · .env.production   build-time constants; APP_ENV and API_MODE
e2e/                            Playwright specs against the built extension, fixture pages, README.md listing the journeys
public/
  icon/                         per size, per environment variant
  fonts/                        only when DESIGN.md names a bundled font
  _locales/                     the manifest's name and description, when localised
src/
  entrypoints/                  WXT's entrypoints — one per surface, content script and the background
    background.ts               defineBackground: wires the modules in src/background/
    popup/                      index.html + main.tsx mounting the popup app; routes.tsx, its route table
    sidepanel/                  the side panel (sidebar_action on Firefox); routes.tsx
    options/                    the options page; routes.tsx
    app/                        the full-tab page for flows the popup cannot hold; routes.tsx
    <name>.content.ts           one content script per host-page concern, with its matches
  shared/                       imported by every context; no context-specific imports
    messages.ts                 the messaging protocol: every message, its Zod payload and result, which contexts may send it
    api.ts                      generated from OpenAPI — never edited
    env.ts                      the build-time constants, validated
    types.ts                    types every context uses (session state, version state)
    shortcuts.ts                the command table: id, default keys, label
  background/
    index.ts                    register() of every module below
    messaging.ts                registers every handler from the protocol, validating payloads and senders
    session/                    tokens in storage.session and storage.local, refresh, the identity flow, sign-out
    gateway/
      core.ts                   base URL, mock dispatch in mock mode, JSON, ApiError
      fetch.ts                  bearer header, platform and version headers, single-flight refresh-and-retry
      realtime.ts               the authenticated stream while a surface holds a port, forwarded over it
    alarms.ts                   periodic work: the version check, cache housekeeping
    menus.ts                    context menu items → routes or feature messages
    commands.ts                 keyboard shortcuts from shared/shortcuts.ts
    notifications.ts
    tabs.ts                     opening URLs, the side panel, the full-tab page at a route
    version.ts                  the minimum-version check and its state
    mocks/
      index.ts                  collects every feature's handlers; the gateway reads it outside production
      auth.ts                   mock sign-in, sign-out, refresh and me, until the auth area's own mocks replace them
    logger.ts
  features/
    <feature-area>/             one folder per feature area, named after docs/feature-areas.md
      api/
        keys.ts                 query keys
        types.ts                provisional types while the area is mocked; deleted at integration
        queries.ts              useX() hooks
        mutations.ts            useXMutation() hooks
      components/               screens and parts that belong to this area only
      content/                  this area's content UI: anchors, the mounted components
      schemas/                  Zod schemas for this area's forms and for the page data its content scripts read
      mocks/
        fixtures.ts             typed sample data
        handlers.ts             mock handlers: method + path → response (run in the background; reused by tests)
      store.ts                  Zustand store, only if the area has client-only state
      index.ts                  what other code may import from this area, and the area's readiness
  components/
    ui/                         shadcn components, owned and edited here
    shared/                     product-wide pieces: surface shell, header, empty state, error state, offline indicator, sign-in prompt, permission prompt, update banner
  content/
    mount.ts                    createShadowRootUi with the tokens and styles; mount, unmount, invalidation
    anchors.ts                  the observer that finds anchors and keeps mounts idempotent
    page.ts                     reading page data safely: selection, the anchor's text, validated
  lib/
    api.ts                      the surfaces' typed wrapper over the api:fetch message; ApiError
    messaging.ts                the typed client from shared/messages.ts
    query-client.ts             createQueryClient(surface) with the storage.session persister
    storage.ts                  storage.defineItem for every preference, with versions
    utils.ts                    cn() and small pure helpers
  hooks/                        product-wide hooks (useMe, useSession, useOnline, useVersionState, usePermission, useShortcut)
  styles/
    globals.css                 Tailwind theme: the DESIGN.md tokens as CSS variables
    content.css                 the same tokens in pixels, for shadow roots
  test/
    api-double.ts               the messaging double over the shared mock handlers, for surface tests
    fixtures/                   HTML fixture pages for content-script tests
```

**Dependency direction, surfaces and content:** `lib` ← `components` ← `hooks` ← `features` ← `entrypoints`. Nothing imports upward, and a feature never imports another feature's internals: cross-feature use goes through the other feature's `index.ts`, and a component needed by two features moves to `components/shared/`. **Across contexts:** `background`, `content` and the surfaces import from `shared` and never from each other; a content script imports nothing that touches `document` of an extension page, and the background imports nothing that touches a DOM. The build fails on a `window` reference in `background/`.

**Entrypoints stay thin.** An entrypoint mounts an app or registers modules and nothing else. Screens, hooks and schemas live in `features/`. A screen that belongs to several areas lives in the area whose entity's lifecycle it advances, as the screen specs decide. Folders are always feature areas: a later phase's feature puts each of its screens in the area it belongs to, never in a folder named after the feature.

**The background is organised by capability**, one module per file or folder above, each exporting a `register()` that adds its message handlers from the protocol and its event listeners. Listeners are registered at the top level of the worker, synchronously, because a worker woken by an event runs only the listeners it registered before going idle.

## 3. Messaging and the gateway

**The protocol** (`shared/messages.ts`) is the only place a message is named. Each entry has a name, a Zod schema for the payload, a Zod schema for the result, and the contexts allowed to send it (`surface`, `content`, or both). `lib/messaging.ts` builds the typed client from it; `background/messaging.ts` registers a handler for each, parses the payload with its schema before the handler runs, and checks the sender (`security` → The background is the trust boundary). A message that is not in the protocol does not exist.

**The protocol is small:** `api:fetch` (surfaces only), `session:status` · `session:signIn` · `session:signOut`, `prefs:get` · `prefs:set`, `tabs:open` (an `https:` URL, user action only), `tabs:openSurface` (the full-tab page at a route; the side panel is opened by the surface that was clicked — Section 6), `permissions:request` (for the background's own entry points only — Section 12), `version:state`, `cache:invalidate` (background → surfaces), the feature messages content scripts send (`<area>:<action>`, each listed with its allowed hosts), and a `realtime` port for streams. Events from the background to open surfaces go over `browser.runtime` messages the surfaces subscribe to through `on(event, handler)`, which returns an unsubscribe function.

**The gateway** (`background/gateway/`) is the extension's one path to the backend, in three pieces:

| Piece | Does |
| --- | --- |
| `core` | Base URL (the real API, or the in-process mock dispatch in mock mode), JSON encoding and parsing, turns every failure into `ApiError` |
| `fetch` | Adds `Authorization: Bearer` from the session, the platform and version headers, refreshes and retries once on 401, maps network failures to the offline state |
| `realtime` | Opens the stream while a surface holds the port, authenticated the way the api-docs say: a WebSocket with the Socket.IO `auth` payload or a single-use ticket fetched over an authenticated request and sent in the URL (a browser's `WebSocket` and `EventSource` cannot carry a header, and the gateway sends no cookies), or SSE over `fetch` with the bearer header and a streamed body, kept alive by a heartbeat under 30 seconds because a quiet stream does not keep the worker alive; forwards messages over the port, closes when the last port closes or the session ends |

**`ApiError`** carries `status`, `code` (the backend's error code), `message` and, for validation failures, `fieldErrors` keyed by field path. It crosses messaging as plain data and is rebuilt in `lib/api.ts`. Every failure is handled by status, the same way everywhere:

| Status | Handling |
| --- | --- |
| **401** | The gateway refreshes once, retries once; if the refresh fails, the background signs out (Section 5) |
| **403** | Show the no-access state |
| **404** | Show the not-found state |
| **422 / validation** | Map `fieldErrors` onto the form |
| **409, 429, and other client errors** | Show the backend's message in a toast |
| **5xx** | Toast with a retry |
| **Network failure** | Offline: cached data with the indicator, or the offline state; a mutation shows a retry toast |

Types have two eras, split at integration. **While a feature is mocked**, no backend exists to generate from, so its types are provisional: hand-written in the feature's `api/types.ts` from the shapes its screens imply, used by its hooks, fixtures and handlers. **At integration** they are deleted and the hooks import from `shared/api.ts`, generated from the backend's OpenAPI spec (`pnpm api:types`) and regenerated whenever the backend's api-docs change; from then on, nobody writes a type for a built endpoint by hand. **Unknown enum values are tolerated**: a `switch` on a backend enum always has a default branch that renders something sensible, so the backend can add a value without breaking an installed version that has not updated yet.

### Mock mode

Before a unit is integrated, its screens run on mock data that goes **through the real data layer**: the same hooks, the same messaging, the same gateway, the same types. Only where the requests go differs.

- `APP_ENV` other than `production` with `API_MODE=mock` makes the gateway's `core` dispatch each request to the handlers collected in `background/mocks/index.ts`, **inside the background**, with no server. With the mode off (`direct`, the default), `core` calls the configured API URL.
- **Passthrough.** A path with no handler is sent to the real API with the real `Authorization` header, so integrated units reach the real backend while mocked ones stay local. Integration removes a unit's handlers and its traffic falls through. The staging workflow's build step always sets `API_MODE=mock`: with passthrough, a staging build with no handlers left behaves as `direct`, and a later phase's units built before their backend exists run on their handlers without a workflow change. `background/mocks/index.ts` and the dispatch stay for the product's lifetime; production never carries them.
- **The unit that owns sign-in is integrated first.** Until it is, every environment runs on the mock auth and only its handlers answer, because the real backend would reject a mock token; after it, sign-in reaches the real backend and every passthrough call carries a real token. Nothing else is integrated before it.
- Each feature's `mocks/handlers.ts` maps `METHOD /path` to a resolver taking `{ params, query, body, session }` and returning the response, using the typed fixtures in `mocks/fixtures.ts`. The handlers are plain functions with no `browser` imports, so the background runs them in mock mode and surface tests run them behind the messaging double. `session` is the access token's decoded payload (`sub`, the user id); it is never verified, which is acceptable only because mock mode does not exist in production. Resolvers keep state in `storage.session` (the worker restarts, so memory would not do: a created record must appear in the next list) and simulate latency.
- Fixtures are realistic: real-looking names, dates and amounts, every state the screen specs describe, edge cases (long text, empty lists, the maximum count). A mock that always succeeds hides the error states the design asked for, so handlers can be told to fail (`?mock=error`).
- **Mock auth** (`background/mocks/auth.ts`) handles sign-in, sign-out, refresh and `me` until the sign-in unit is integrated: it returns JWT-shaped tokens carrying the same claims the backend's tokens carry, through the same `session` module, so the gateway and the guards run unchanged. The sign-in unit's own `mocks/` replaces it when that unit is built, and that unit's integration removes it.
- **Real-time channels are not mocked.** In mock mode a screen that would subscribe polls its query against handler state instead; the real channel is wired at integration, as the api-docs specify.
- **Production builds carry no mock mode.** `wxt.config.ts` throws when `APP_ENV=production` is built with the mode on, the gateway imports the mock module only outside production, and a test proves the mode is ignored when `APP_ENV` is `production`.
- **A unit not yet `Integrated` is unreachable in production.** Once the extension is published, a later phase's built-but-not-integrated screens land on `main` before their endpoints exist. Each feature's `index.ts` exports its readiness; the surfaces mount a unit's routes, the shell shows its entries, and the content scripts mount its content UI only when `isIntegrated(area) || !isProduction`; `/extension-integrate` flips the readiness when the unit is integrated; and a test proves no un-integrated route or anchor is reachable with `APP_ENV=production`.

## 4. Data layer

- **One hook per query, one per mutation**, in the feature's `api/`. Components never send messages directly; they call hooks, and hooks call `lib/api.ts`.
- **Keys** are hierarchical and built by functions: `clipKeys.all`, `clipKeys.list(filters)`, `clipKeys.detail(id)`. Invalidate the narrowest key that covers the change.
- **Lists** use `useInfiniteQuery` for cursor pagination and `useQuery` with `placeholderData: keepPreviousData` for page numbers, matching whichever the api-docs give, rendered through `@tanstack/react-virtual` past ~100 rows. Filters and sort live in the route's state when a surface should reopen on them, otherwise in local state.
- **Mutations** invalidate on settle, not on success alone, and tell the background to broadcast `cache:invalidate` so the side panel and a popup open at the same time agree. Optimistic updates are for actions where the user expects instant feedback (toggles, reordering) and always roll back from the snapshot on error.
- **No retries on 401, 403, 404 or 422.** Retry other failures twice with backoff, queries only; mutations never retry on their own.
- **Derived data is computed with `select`** or a memo, never copied into state.
- **Real-time** updates write into the cache with `setQueryData` or invalidate. A surface opens its port when it mounts a screen that needs the stream and closes it on unmount; the background keeps the connection only while a port is open.
- **The persisted cache** lives in `storage.session` (cleared when the browser closes), holds only what `security` → Data in the browser allows, is keyed by user id, and is dropped on sign-out and sign-in. Persisting to `storage.local` is a product decision made in a build plan.

## 5. Session

The backend issues, rotates and checks tokens. The background carries them and triggers refreshes; surfaces and content scripts only know whether a session exists.

| Token | Lives in | Why |
| --- | --- | --- |
| Access (JWT, minutes) | `storage.session` | Memory-backed, cleared when the browser closes, unreadable by content scripts; survives the worker's own restarts |
| Refresh (opaque, days, rotated) | `storage.local` | Survives browser restarts; isolated from web pages and other extensions by the browser |

- **The worker has no memory.** Every session read goes through `storage.session`; nothing is cached in a module variable without being re-read on wake. The single-flight refresh is a `navigator.locks` lock named for the refresh, so two events waking the worker together still refresh once.
- **Wake:** a surface opening or a content script's first message asks `session:status`; the background reads the refresh token, refreshes if the access token is missing or within a minute of expiry (reading `exp` without verifying it — the backend verifies), and answers. A missing or rejected refresh token means signed out. A network failure is neither: the refresh token stays, the surface renders its offline state from the persisted cache, and the refresh runs on the next request.
- **The gateway** refreshes single-flight, skips the logic for the refresh route itself, and signs out if the refresh fails.
- **Sign-in** is `browser.identity.launchWebAuthFlow` to the backend's sign-in page with a PKCE challenge and `state`, returning to `browser.identity.getRedirectURL()`; `session` checks `state`, exchanges the one-time code with the verifier for the tokens-in-body response, stores them, and broadcasts `session:changed`. Email-and-password sign-in, where the product has it, posts through the gateway from the popup's form. On Safari, which has no `identity` API, provider sign-in is a tab the background opens on the backend's sign-in URL, redirected to an `https` page on the product's web domain that the background watches with `tabs.onUpdated`, reads the code from, and closes — or the email-and-password form alone, as the setup plan decided. **No sign-in page is ever embedded in a surface or injected into a page.**
- **Sign-out** calls the backend, then clears in this order: `storage.session` (the access token and the persisted cache), the refresh token in `storage.local`, any store that holds the user's data, and broadcasts `session:changed` so every surface returns to the signed-out shell and every content script unmounts its UI. Sign-in clears the persisted cache too, so the next user never sees the previous one's data.
- **Guards** are the surfaces': the signed-in routes mount only when `session.status === 'signedIn'`, from `useSession` fed by the background's events; content UI mounts only when signed in, unless the feature is designed for signed-out use. **A guard is a convenience, never authorization**: the backend decides access on every request.
- **Current user:** one `useMe` query. Its roles show or hide UI through `RoleGate`; the backend enforces them.

## 6. Surfaces, routing and the host page

- **Each surface has a job.** The **popup** is for a glance and a quick action: it is at most 800×600, closes when it loses focus, and never holds a multi-step flow or an unsaved form of any length. The **side panel** is for work alongside the page: it persists, follows the tab, and holds the product's main screens. The **options page** holds settings and the account. The **full-tab page** holds anything that needs room or must survive a click elsewhere (onboarding, a long form, a review). `docs/design/surfaces.md` says which screen lives where.
- **Routes** come from `createMemoryRouter` per surface, from the surface's `routes.tsx`, which imports screens from `features/`: every screen with its loading, error and not-found states, an error boundary per surface, and the initial route taken from the surface's `?route=` query, which the background sets when it opens a surface (`tabs:openSurface`). Dynamic segments and route state are validated at the top of the screen with a Zod schema from the area's `schemas/`; an invalid one renders the not-found state, not a crash. The popup remembers its last route in preferences and reopens on it.
- **Entry points into the extension** — the toolbar icon, the context menu, a keyboard command, a notification click, the omnibox where the product has one — are handled by the background, which validates what they carry (a URL, a selection, an id) and opens the right surface at the right route. A link into a protected screen remembers its target through sign-in.
- **Content UI** is mounted by `content/mount.ts` through `createShadowRootUi` at the anchors `features/<area>/content/` finds: a shadow root per mount, the design tokens and the component styles injected into it, `Portal` containers for dialogs and menus inside it so nothing escapes to the page. Mounts are idempotent (an anchor already mounted is skipped), observed with a debounced `MutationObserver` scoped to the narrowest ancestor, and cleaned up through WXT's `ctx.onInvalidated` so an updated extension never leaves an orphaned script running.
- **The page is read, never trusted.** `content/page.ts` reads a selection or an anchor's text and parses it with the feature's Zod schema before it crosses messaging; it never reads cookies, storage or globals of the page, never injects into the main world (`world: 'MAIN'` is a build-plan decision with its own security review), and never modifies the page beyond its own shadow hosts. Matches are the narrowest patterns the feature needs, and `run_at: 'document_idle'`.
- **Browser differences are guarded once.** `import.meta.env.BROWSER` selects the side panel API (`sidePanel` on Chromium, `sidebar_action` on Firefox), the redirect URL shape, and the few APIs one browser lacks (`offscreen` and `identity` on Safari, `offscreen` on Firefox); a guard tests for Firefox or Safari and treats everything else as Chromium, so Edge is never left out; the guard lives in `background/` or `lib/`, never in a screen. Every browser gets every screen and every state.
- **Gated browser calls run in the click handler that earned them.** `permissions.request`, `sidePanel.open` and `sidebarAction.open` may only run in response to a user action: Firefox does not carry that status through messaging at all, and Chromium carries it only until the handler's first `await`. So the surface the user clicked makes the call itself, synchronously in its handler (through the guard in `lib/`); the background makes these calls only from its own entry points — a context-menu click, a command (checked on Firefox in the browser check), the action click. Content UI cannot request a permission (the API is absent in content scripts) and cannot open the sidebar on Firefox; it hands off to a surface.
- **Keyboard commands** come from one table in `shared/shortcuts.ts`, declared in the manifest through `wxt.config.ts` and read by `useShortcut` for tooltips; users can rebind them in the browser, so labels read the live binding from `browser.commands.getAll()`.

## 7. Components

- **shadcn components are ours.** Edit them in `components/ui/` to match `DESIGN.md`; never wrap one to restyle it. Add components with the shadcn CLI, then adjust.
- **Variants with `cva`**, states with `data-*` attributes (`data-state="open"`), styling through Tailwind. No inline styles except for values computed at runtime.
- **One component per file**, named in PascalCase, file in kebab-case (`clip-card.tsx`). A screen is `{screen-name}-screen.tsx`; its parts live beside it in a folder once there are more than two. Content UI components live in the area's `content/`.
- **Props are typed with an interface**, named `{Component}Props`, and destructured. Boolean props read as predicates (`isOpen`, `canEdit`). Callbacks start with `on`.
- **Composition over configuration**: slots and children rather than a dozen boolean props. A component that takes more than seven props is usually two components.
- **Every list has an empty state**, every async component a loading, error and offline state, every gated feature a permission-needed state, as the design system specifies. A screen is not done with only its happy state.
- **Keyboard first.** Every action reachable by Tab, Enter activates, Escape dismisses (and closes the popup only when nothing else is open), focus visible and returned when a dialog closes. Content UI never traps focus on the host page and never steals it on mount.
- **Accessibility is part of the component**: semantic elements, labels on every input, `aria-*` only where semantics do not already say it, contrast from the system, reduced motion respected. `eslint-plugin-jsx-a11y` is on and not disabled per line. Content UI is announced as a region with the product's name so a screen reader can tell it from the page.

## 8. State

In this order; use the first that fits.

1. **Server state** — TanStack Query. Anything the backend owns.
2. **Route state** — the memory router's params and state. The current entity, the active tab, a filter a reopened surface should reproduce.
3. **Form state** — React Hook Form. Never mirrored into a store.
4. **Local state** — `useState` in the component that owns it.
5. **Global UI state** — Zustand, one store per surface and concern, for what survives navigation and nothing above fits: selection, an unsaved draft the full-tab page keeps. Stores hold no server data and no derived data. State that must survive the popup closing goes to the next level.
6. **Persisted preferences** — `storage.local` through `lib/storage.ts` (`storage.defineItem` with a version and migrations): theme, layout, dismissed hints, the popup's last route. Never tokens, never server data; `security` says what may be persisted.

No React Context for state that changes often; it re-renders every consumer. Context is for dependency injection (theme, the query client).

## 9. Forms

- **One Zod schema per form**, in the feature's `schemas/`, derived from the unit's business rules and the screen spec's field constraints — same names, same limits. The type is `z.infer`, never written twice.
- **React Hook Form with `zodResolver`**, validating on blur and then on change, submit disabled while submitting, never while merely invalid. `Enter` submits from a single-line field; `Cmd/Ctrl+Enter` from a multi-line one.
- **Server validation wins.** A 422 maps `fieldErrors` onto the fields with `setError`; an error with no field goes to the form's root error.
- **Submission goes through a mutation hook.** The form never sends a message.
- **Forms live where they survive.** Anything longer than a field or two lives in the side panel or the full-tab page, never in the popup, because the popup closes on a stray click and takes the draft with it. A draft worth keeping is saved to `storage.session` as the user types.
- Inputs are controlled through RHF's `Controller` only for components that need it (selects, date pickers); native inputs use `register`.

## 10. Effects

- `useEffect` is for synchronising with something outside React: a background event, a storage change, a page mutation. Not for fetching, not for deriving state, not for responding to a prop change that could be computed during render.
- **Every subscription returns its unsubscribe** (`on(event, handler)` and `storage.watch` both return one); an effect that forgets fires its handler twice after the next render and leaks across the surface's life.
- Every async effect guards against the component having unmounted — and in content UI, against the mount having been invalidated.
- Event handlers do the work that an event causes. If logic runs "after X changes", ask whether it belongs in the handler that changed X.

## 11. Styling

- **Tokens, not values.** Colors, spacing, radii, shadows and type sizes come from the CSS variables `styles/globals.css` defines from `DESIGN.md`, exposed as Tailwind utilities. Arbitrary values (`w-[13px]`, `text-[#333]`) need a comment saying why the scale did not fit.
- **`cn()` merges classes**; conditional classes go through it, never through string concatenation.
- **Dark mode follows the browser** by default (`prefers-color-scheme`, which the browser maps from its own theme); `globals.css` redefines the semantic variables under `.dark`, so a component writes `bg-background` once. A manual override, where the product offers one, is a persisted preference applied at each surface's root and inside each shadow root.
- **Content UI is styled in pixels.** `rem` resolves against the host page's root font size, which the extension does not control, so `styles/content.css` defines the same tokens in `px` and the content build of Tailwind uses a pixel root. The shadow root resets inherited properties (`:host { all: initial }` plus the tokens) so the page's fonts, colors and `line-height` never leak in, and ours never leak out.
- **Surfaces have fixed shapes.** The popup renders at the width `DESIGN.md` names (320–400 px) and grows in height to its content up to 600; the side panel is fluid from 320 px up; the options and full-tab pages centre their content at a maximum width. Nothing clips or becomes unreachable at 200% browser zoom or at the largest font setting.
- **Class order** is Prettier's (`prettier-plugin-tailwindcss`), so diffs stay readable.
- **Motion** follows `DESIGN.md`'s Motion section: durations and easings from its table, reduced motion respected everywhere through the `prefers-reduced-motion` media query, layout animations only through Motion's `layout` prop. Content UI animates its own appearance only; it never animates the host page.

## 12. Permissions, the browser and the worker

- **Permissions are declared once**, in `wxt.config.ts`, each with a comment stating the feature that needs it and the justification `/extension-release` will file with the stores. `activeTab` is the default way to reach the current page; a host permission is declared only for origins the product's single purpose needs; everything else is an **optional permission** the surface requests itself with `browser.permissions.request`, in the click handler, after a sentence in the UI says what it is for (Section 6: the request never goes through messaging), and checked with `usePermission` before the feature renders its full state; the background learns the outcome from `permissions.onAdded` and `permissions.onRemoved`, and `permissions:request` exists only for its own entry points. A denied permission renders the permission-needed state, never an error.
- **Everything periodic is an alarm.** `browser.alarms` with a period of at least 30 seconds (the browsers' floor), handled by a top-level listener; `setTimeout` and `setInterval` in the background do not survive the worker's suspension. The version check runs on startup and then on an alarm a few times a day.
- **The worker is restartable by design.** Any state it needs is in `storage.session` or `storage.local`; any in-flight work that must finish (an upload, a multi-step sign-in) keeps its progress in storage and resumes from a listener. The realtime connection and open ports keep the worker alive only while there is traffic.
- **External URLs** open through `tabs:open`, which the background allows for `https:` (and `mailto:`) on a user action only, in a new tab. Payments open the backend's hosted checkout the same way; the outcome comes from the backend's state, never from a URL.
- **Notifications** are the browser's, through the background; a click opens the right surface at the right route. They carry a title and a line, never personal data.
- **Clipboard** writes happen on an explicit user action, from a surface; a content script writes to it only when the feature is built around that action. The extension never reads the clipboard without a user action, and never from the background.
- **Updates:** the stores deliver them; the background listens to `runtime.onUpdateAvailable` and lets the update apply when no surface is open (`runtime.reload`), so a user mid-task is never interrupted. The force-update screen (`useVersionState` with `supported` / `soft` / `force` from the backend's minimum-version endpoint) blocks the surfaces under `force` and opens the store listing.

## 13. Performance

- **The popup paints within its budget** (`CLAUDE.md`, default 200 ms to a painted shell from the persisted cache); its bundle carries only its screens, with the side panel's and options page's split by entrypoint, and it does no work on open beyond its route's query.
- **Content scripts are light.** Each content script's bundle stays under the budget `CLAUDE.md` records (default 100 KB), matches only the hosts it needs, does nothing until an anchor is found, and loads its UI bundle lazily (`import()`) when it is. A content script never polls; it observes.
- **The background wakes fast**: no heavy module at the top level, listeners registered synchronously, work deferred into the handler that needs it. Heavy work (parsing a large page, image processing) runs in an `offscreen` document where the browser has one, or in the surface that asked, never for long in the worker.
- **Surfaces stay responsive**: long lists virtualised, heavy computation in a Web Worker.
- **`memo`, `useMemo` and `useCallback` are applied where a profile shows the cost**, not by habit. React Compiler is on and makes most of them redundant.
- **Images** are sized for the display scale and served from the backend's CDN; icons are bundled.
- **Memory** is watched: a surface that closes releases its subscriptions and ports; a content UI that unmounts releases its observers; a leak check (open and close the side panel ten times, navigate ten pages with content UI) is part of the setup verification.
- **The total package is checked** when a dependency is added and stays under the budget `CLAUDE.md` records (default 4 MB zipped), because store reviews read bundle size as a signal.

## 14. Errors and observability

- **Errors render, they do not vanish.** A failed query shows the error state with retry; a failed mutation shows a toast with the backend's message; an uncaught render error reaches the surface's error boundary, never a blank popup. Content UI that fails removes itself and reports; it never breaks the host page.
- **Sentry captures** in every context, tagged with the context name and the extension version: uncaught errors, boundary errors, failed mutations with the request's method and path (never its body), the background's unhandled rejections. A content-script error is tagged with the host's registrable domain, never the page URL. Expected `ApiError`s (401, 403, 404, 422) and offline failures are not reported.
- **Logging goes through `logger.ts`** in every context, scrubbed as `security` says, at `warn` and above in production. No `console.log` in committed code.
- **User-facing messages come from the backend's message keys** where the business rules define them, and from `DESIGN.md` → Content and Voice otherwise.

## 15. Code style

- **TypeScript strict**, no `any`, no non-null assertions without a comment, no `as` casts to silence the compiler. Narrow with type guards; model variants as discriminated unions. One `tsconfig` per context (`background`, `surfaces`, `content`) so a DOM type cannot leak into the background and an extension-page type cannot leak into a content script.
- **Imports** in groups, blank line between: React, third-party, `wxt/*` and `@webext-core/*`, `@shared/`, `@/` aliases, relative, then styles. Type imports with `import type`. No default exports except where WXT's entrypoints require them.
- **Naming:** `camelCase` for values and functions, `PascalCase` for components and types, `SCREAMING_CASE` for true constants, `use` prefix for hooks, `is`/`has`/`can` for booleans. Files `kebab-case`. Folders in `features/` match the slugs in `docs/feature-areas.md`. Messages `domain:action` (`session:signOut`, `clips:save`); storage items `domain.key`.
- **Functions are small and do one thing**; a file over ~300 lines is a sign to split by responsibility.
- **Comments explain why, not what.** A `TODO` carries the ticket or issue it refers to.
- **Dead code is deleted**, not commented out. Git keeps the history.
- **Formatting and linting pass before commit**, enforced by a pre-commit hook; CI repeats the check, and `web-ext lint` runs on the Firefox build.

## 16. Tests in this stack

What tests must prove is in `testing`. In this stack:

- Vitest with WXT's plugin and three projects: `surfaces` (jsdom, `@testing-library/react`, `user-event`, `jest-dom`), `background` (Node) and `content` (jsdom, loading a fixture page's HTML), each with `@webext-core/fake-browser` as `browser`, reset between tests.
- Surface tests render through `renderWithProviders` (a fresh `QueryClient` with retries off and no persister, the memory router at the screen's route, the theme) with **the messaging client replaced by the test double** in `src/test/api-double.ts`, which answers `api:fetch` from the shared mock handlers, answers `session:status` as signed in, and lets a test override one handler, fail one call, or fire an event.
- Background tests exercise modules directly: the gateway against MSW (`msw/node`) with the fixtures; message handlers through the protocol with valid and invalid payloads and with senders that must be refused; `session` against the fake browser's storage, including a simulated worker restart (module re-import with storage intact).
- Content tests load a fixture page into jsdom, run the content script's `main` with a fake `ctx`, and assert on the shadow root: the anchor found, the UI mounted once, the message sent with the parsed data, the unmount on invalidation.
- Test files sit beside the code (`clip-card.test.tsx`); E2E specs in `e2e/`, run with Playwright against the built Chromium extension, with content-script journeys on the fixture pages served locally.
