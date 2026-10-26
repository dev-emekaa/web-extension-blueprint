## Documentation map

| Type | Location |
| --- | --- |
| Feature-areas registry *(from the backend)* | `docs/feature-areas.md` |
| PRD and product brief *(from the backend)* | `docs/product-docs/` |
| Business rules *(from the backend)* | `docs/business-rules/` |
| App flows *(from the backend)* | `docs/app-flows/` |
| API docs *(from the backend)* | `docs/api-docs/` |
| API testing *(from the backend)* | `docs/api-testing/` |
| Phase folders *(from the backend, plus this repo's UI tracker and the phase's screen specs)* | `docs/phase-{n}/` |
| UI tracker, first phase | `docs/ui-implementation-order.md` |
| Design system | `DESIGN.md` (project root) |
| Style tile *(only without the canvas)* | `docs/design/style-tile.html` |
| Surface map | `docs/design/surfaces.md` |
| Approved designs | `docs/design/screens/` |
| Image prompts | `docs/design/images.md` |
| Screen specs | `docs/screen-specs/` |
| Backend issues handoff | `docs/integration-handoff/` |
| Analytics events *(with the analytics rule)* | `docs/analytics-events.md` |
| Infra setup | `docs/infra/` |
| Release records | `docs/release/` |

## Audiences and clients

<!-- Copy the table from the backend's CLAUDE.md, then mark which audience this repo serves.
     An audience is a group of roles using its own separate product; a client is a separately designed app.
     The Chrome, Edge, Firefox and Safari builds are one client, extension. -->

| Audience | Primary | Clients |
| --- | --- | --- |

| This repo is | Value |
| --- | --- |
| Audience | |
| Client | extension |

## Project configuration

<!-- The extension skills read these. Anything missing, they ask for once; /extension-setup fills in the commands and identifiers.
     Builds are published through the browser stores. The backend repo path is used by the docs sync scripts. -->

| Key | Value |
| --- | --- |
| Project name | |
| Stack / framework | {{STACK}} |
| Backend repo path | |
| Sibling client repo path | the web, mobile or desktop client's repo, where one exists — whichever client's `DESIGN.md` came first is the brand's source |
| Backend stack | |
| Backend OpenAPI spec | URL, once the backend is built — blank until then |
| Staging API URL | |
| Production API URL | |
| Browsers | Chrome · Edge · Firefox — which the product ships; Safari off unless the PRD asks for it |
| Extension ids | the fixed dev and staging ids (Chromium `key`, Firefox `gecko.id`); in production the Chrome Web Store id and the Edge Add-ons id, each assigned at the first upload, and the `gecko.id` the project chooses |
| Single purpose | one sentence: what the extension does, as the store listing will state it |
| Target sites | the hosts content scripts run on, or "the current page only (activeTab)", or "every page" with the reason |
| Surfaces | popup · side panel · options page · full-tab page · content UI — which the product has |
| Dev server command | |
| Test commands | |
| E2E command | |
| API types command | |
| Docs sync commands | |
| Design system | not written yet — `/design-system` updates this |
| Design canvas | /design (Claude Design in Claude Code) |
| Design references | Mobbin, through its MCP, when connected. Without it: — *(web research, your screenshots, or both; asked once if blank)*. Your own screenshots, if any, go in `docs/design/references/` (optional, never committed) |
| Image generation | Gemini, through its MCP |
| Stock photo providers | Unsplash / Pexels — which are configured; their keys live in `.env` |
| Analytics | off |
| Sign-in providers | email and password · the backend's OAuth providers through the identity flow — which the PRD asks for |
| Payments | none · the backend's hosted checkout, in a tab |
| Store listings | the Chrome Web Store item, the Firefox Add-ons listing, the Edge Add-ons listing — their URLs once they exist |
| Production status | not published yet |
| Production deploy trigger | Deploy Production workflow, run manually on the verified commit |
| Release owners | GitHub usernames allowed to run the production deploy |
| Popup paint budget | 200 ms to a painted shell from the persisted cache |
| Content-script budget | 100 KB per content-script bundle |
| Package budget | 4 MB zipped |
| Crash-free floor | 99.5% of sessions, below which a rollout halts |
