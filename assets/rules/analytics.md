# Analytics

Rules for product analytics in this extension, installed only when the product tracks user behaviour. The tool is PostHog, named in `tech-stack`; what may never be sent is in `security` → Third-party SDKs; what must be disclosed to the stores is in `security` → Store compliance.

## Setup

- **`posthog-js` bundled, in the surfaces only.** The library is a lockfile dependency, never loaded from PostHog's CDN (the CSP and the stores both forbid it); its `connect-src` entry is the one network allowance it gets. Content scripts and the background send no analytics: a content script would carry a tracker onto every page it runs on, and the background has no user in front of it. A content-UI action worth measuring sends its feature message, and the surface or the background records the outcome through the feature's event.
- **One client per surface, initialised once**, in a provider at the surface's root, with autocapture, session replay and pageview capture off. Every event is explicit. Events queue in `storage.session` and flush in batches, so a popup that closes mid-batch loses nothing.
- **Identify after sign-in, reset on sign-out.** The distinct id is the backend's user id; the person properties are the role, plan and account age, never a name or email. The browser and extension version are properties of every event; the pages the user visits never are.
- **Environments are separate projects.** Development builds send nothing; staging and production each have their own key, baked in per build.
- **It is disclosed.** Installing this rule means the product collects usage data as well as the error reports every build already sends: the Chrome Web Store privacy form, the Firefox `data_collection_permissions` (`technicalAndInteraction`, already declared for Sentry), the Edge disclosures and the privacy policy all say so before the first version with analytics ships, and `/extension-release` checks that they do.

## Events

- **Named `object_action`** in snake case, past tense for things that happened: `clip_saved`, `invite_sent`, `filter_applied`. No surface names in event names; the surface is a property.
- **Properties are enums, numbers and ids**, never free text, never page content, never a URL or hostname, never personal data. An amount is a number; a status is the backend's enum value; "where it happened" is the surface and the feature, not the site.
- **One event per user intent**, fired from the handler that completed it (after the mutation succeeds), not on render and not on click.
- **Screen views** are tracked once per route change from each surface's router, with the route pattern (`/clips/:id`), not the concrete path, and the surface as a property. Content UI mounting is not a screen view; the action the user takes in it is the event.
- **Every event is listed in `docs/analytics-events.md`** with its properties and the question it answers. An event nobody can say the purpose of is removed.

## Feature flags

Where the product uses them: flags are read through the PostHog client with a default, loaded once per surface open and cached in `storage.local` so an offline open still has an answer, and never gate a security decision or add a capability the package does not already contain (a flag may hide or show, never fetch code or widen permissions). A flag that has been fully rolled out for a release is removed from the code.

## Consent

Where the product's users require it, nothing is sent before consent, the choice is a persisted preference applied in every surface, and the provider respects it on initialisation. Consent UI follows the design system's content rules: plain words, equal buttons. On Firefox the install prompt already shows the `technicalAndInteraction` declaration, which the user can turn off in the browser; the extension reads that choice through the permissions API (the `data_collection` entry Firefox adds to `permissions.getAll()` and `permissions.onRemoved`, as `security` → Third-party SDKs says for Sentry) and sends nothing when it is off.
