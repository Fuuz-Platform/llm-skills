---
name: fuuz-html-dashboards
description: Build HTML/data-URI dashboards rendered in Fuuz screens via a backend flow + embedded webpage element. Use whenever a Fuuz user asks for a "dashboard", "wallboard", "live screen", "operator screen", "data visualization", "3D scene", "interactive widget", "real-time view", or any custom HTML output for a Fuuz screen. Also use when a user asks how to embed Three.js, D3, Chart.js, ECharts, Cytoscape, Leaflet, Tabulator, Plotly, Mermaid, Transformers.js, FusionCharts, or any other web library in a Fuuz screen, OR asks which library to use for a manufacturing visualization (OEE gauge, Pareto, material genealogy, plant floor map, traceability tree, sankey, etc.). Covers the full pipeline — flow design, GraphQL queries, JSONata bundling, saved-script HTML building, percent-encoded data URIs, theming, timezone, smooth refresh, screen embedding, and library selection. Also covers WebGL and 3D scenes, Draco model compression, real-time updates over the platform data-change subscription bus, reading across multiple tenants, and getting an oversized page under the data-URI limit with gzip. Always trigger this skill for dashboard work, even if the user just says "make me a screen that shows X."
---

# Fuuz HTML Dashboards

Build custom HTML dashboards that render inside a Fuuz screen. The HTML is generated server-side by a backend flow, encoded as a `data:text/html` URI, and embedded via the screen's webpage element.

## Why this pattern

Fuuz screens are powerful for forms and tables but limited for custom visualization. Pushing the HTML through a backend flow lets you:

- Use rich client-side rendering (Three.js, D3, Chart.js, raw SVG/Canvas)
- Apply complex business logic to shape data before render
- Cache the result via the flow's response handling
- Theme/locale/timezone-aware output without screen-side gymnastics

The cost: HTML is built once per refresh and shipped as a complete document. You're not doing live updates inside a persistent iframe (that's a different pattern — see `references/postmessage-pattern.md`).

## When to use this skill — and when NOT to

**Use it for:**
- Custom HMIs (warehouse wallboards, fleet maps, operator dashboards)
- Anything 3D (Three.js, Babylon)
- Heavy data visualization needing custom CSS/animations
- Interactive widgets with non-standard UI (timelines, gantts, network graphs)
- Theming/styling beyond what standard Fuuz components support

**Don't use it for:**
- Simple forms, tables, KPI cards — Fuuz native components are faster and integrate better
- Things that need to react to in-screen events without a full refresh (use postmessage pattern instead)
- Anything where the encoded page would reach **2,048 KB** — Chrome silently refuses to navigate a frame to a `data:` URI at or above that. This is a budget, not a wall: gzip + `DecompressionStream` took a 2,516 KB page down to ~1.2 MB. See `references/large-payloads.md` before giving up on a build.

## Related skills (load before building)

Always read these alongside this skill:

- **`fuuz-data-flow`** — Backend flow architecture, node wiring, request/response shape
- **`fuuz-data-flow-nodes`** — Specific node config (Source, Query, Collect, JSONata, Script, Response)
- **`fuuz-graphql`** — Query patterns, Connection shape, pagination, aggregations
- **`fuuz-expressions`** — JSONata syntax and constraints, JavaScript sandbox rules
- **`fuuz-screen-design`** — Form element, embedded webpage, `$executeFlow`, dynamic fields

If the user asks about styling tokens (colors, glass surfaces, dark/light), also read `fuuz-project-ui-styling.md` from the project knowledge.

## Mandatory workflow

When a user asks for a dashboard, walk them through these four steps in order. Confirm assumptions before building.

### Pre-work — gather requirements

Before writing anything, ask:

1. **What data sources?** Which models, which fields, any aggregations? Get specific. "Show production orders" is too vague — ask "filtered how, grouped by what, sorted by what."
2. **What filters?** Will the user filter the dataset on-screen? If yes, list each filter and its data source. Filter fields must cascade properly on related IDs (e.g., `areaId` → `workCenterId` → `workUnitId`) so picking an Area narrows the WorkCenter choices.
3. **Refresh cadence?** Manual button only, or auto-refresh every N seconds?
4. **Theme + timezone?** Confirm the flow will pass `$metadata.settings.ThemeMode` and `$metadata.settings.TimeZone` so the dashboard renders correctly for the user's preferences.

Never skip this. Building before requirements wastes work.

### Step 1 — Create the backend flow

Flow type: **system** (not web — system flows can be called via `$executeFlow` from a screen and have access to `$metadata`).

Standard node sequence:

```
Request → setContext → try/catch → Source → Query(s) → Collect → JSONata bundle → Saved Script → Response
                                                                       │
                                                            ┌──────────┴──────────┐
                                                            │  validates input,    │
                                                            │  bundles $state.context,
                                                            │  attaches theme/tz   │
                                                            └─────────────────────┘
```

Detail on each node:

- **Request** — entry point. Variables coming from the screen-side `$executeFlow({ variables })` call land here. Always define every variable the screen will send, including filter values.

- **setContext** — IMMEDIATELY after Request. Store EVERY variable from the request under a `REQUEST` key in context: `setContext({ REQUEST: { areaId: $.areaId, themeMode: $.themeMode, ... } })`. This is the single source of truth for downstream nodes. Never reach back to `$.variableName` after this point — always go through context.

- **try/catch** — wrap the whole external-call section. Unhandled query failures = silent flow death.

- **Source** — kicks off the query branch. Usually one Source per logical dataset.

- **Query(s)** — one query per model. Apply filters from context (`$state.context.REQUEST.areaId`). Select only the fields the dashboard actually uses. Always add `first:` pagination limits — never issue an uncapped query.

- **Collect** — CRITICAL: collect ALL output keys from upstream queries. If you have three queries each emitting different keys (e.g., `workOrders`, `inventories`, `automatedGuidedVehicle`), the Collect node must list all three. Multiple output keys are treated separately by the flow — leaving any out drops that data from the payload.

- **JSONata bundler** — Bundles `$state.context.REQUEST` (and anything else the script needs) into the payload alongside the query results. The saved script doesn't see `$state` — it only sees `$` (its input). So if the script needs the theme, this node must put it there.

  Template:
  ```jsonata
  $merge([
    $,
    {
      "themeMode": $state.context.REQUEST.themeMode,
      "timeZone": $state.context.REQUEST.timeZone,
      "filters": $state.context.REQUEST
    }
  ])
  ```

- **Saved Script (JavaScript)** — Generates the HTML and returns the data URI. For most dashboards use a **saved backend script** (not inline) so it can be tested, version-controlled, and reused. Saved scripts have full ES2020+ — `?.` and `??` work here, unlike in inline web-flow scripts.

  Inline Script nodes are also valid — and sometimes preferred — when:
  - The dashboard is small / single-use (one screen, one purpose, won't be reused)
  - You need direct `$state` access without a JSONata bundler (inline scripts CAN read `$state.context` directly)
  - You need `$appConfig` access (also only available inline, not in saved scripts)
  - The script is short enough that splitting it into a saved file adds more friction than it removes

  The trade-off: inline web-flow JS is more restricted (no `?.`, no `??`, `var` only, no `for...of`, no `Map`/`Set` — see `references/v8-sandbox-constraints.md`). Pick the tool that fits the job.

  Whichever you pick, follow `references/saved-script-template.md` for the structure — it works for both contexts with the input variable name swapped (`$` for saved scripts, the inline-script convention for inline).

- **Response** — returns `{ dashboardUri: <data uri>, ...optional metadata }`. ALWAYS use `dashboardUri` as the key. NEVER wrap the output inside `payload`. The screen binds directly to `$components.Form.data.dashboardUri`.

### Step 2 — Build the screen

**The rendering element rules are non-negotiable:**

1. **Only render an iframe/embedded webpage inside a Form**. Never put it inside a Card, Table cell, Container, or other element. The webpage element's data binding and refresh lifecycle are coupled to Form-style data containers. Putting it elsewhere produces inconsistent loading and silent failures.

2. **If you need cards (for user selection, filter chips, an item list, a hover-rich layout), use Markdown elements** — NOT a webpage with inline JS. Markdown can render styled HTML (`<div>`, `<span>` with inline styles, even `<details>/<summary>`), but no JavaScript. This is sufficient for almost all card-like UI: hover effects via CSS, status pills via inline styles, clickable rows via the screen's action steps wired to the markdown element's `onClick`. Cards for interaction = markdown. The webpage element is for the dashboard render, NOT for individual cards.

A simple two-element pattern:

1. **Filter Form** (optional, only if filters were requested)
   - Standard Fuuz inputs (SelectInput for picklists, DateInput for date ranges, etc.)
   - **Cascading filters**: child filters use their parent's value in `where`. E.g., the WorkCenter SelectInput's `query.where` filters by `areaId: $components.FilterForm.data.areaId`. When the user picks an Area, the WorkCenter list narrows automatically.
   - A "Refresh" button or onChange handler that calls `$executeFlow("flowId", { variables: {...} })` on the dashboard Form.

2. **Dashboard Form** — invisible/zero-height form that holds the flow result.
   - `pageLoadAction` = `$executeFlow("flowId", { variables: { themeMode: $metadata.settings.ThemeMode, timeZone: $metadata.settings.TimeZone, ...filters } })`
   - The flow's response populates `$components.DashboardForm.data` — so `data.dashboardUri` is the data URI we built.

3. **Embedded Webpage element — INSIDE the Form**
   - `path` = `$components.DashboardForm.data.dashboardUri`
   - **Enable all permissions** in the webpage config — the inline JS needs them to run. At minimum: scripts, same-origin iframe behavior, popups if your dashboard opens new tabs.
   - **Dynamic fields**:
     ```json
     {
       "payload": [],
       "context": ["components.DashboardForm.data.dashboardUri"]
     }
     ```
     This makes the webpage re-render whenever `dashboardUri` changes, but doesn't trigger on every payload mutation.
   - Set width/height to `100%` and `flexGrow: true` so it fills the available space.

### Step 3 — Wire the filters

If filters are involved, complete this loop:

1. Filter Form inputs save their values to `$components.FilterForm.data.*`.
2. Refresh button calls:
   ```
   $components.DashboardForm.fn.refresh({
     variables: {
       themeMode: $metadata.settings.ThemeMode,
       timeZone: $metadata.settings.TimeZone,
       areaId: $components.FilterForm.data.areaId,
       workCenterId: $components.FilterForm.data.workCenterId,
       // ...other filters
     }
   })
   ```
3. Each filter variable lands in the flow's Request node.
4. The setContext node stores them under `REQUEST.*`.
5. The Query nodes reference them via `$state.context.REQUEST.areaId` in their `where` clauses.
6. Cascading filters guarantee related IDs only show valid options — picking an Area filters the WorkCenter list to that Area's children.

### Step 4 — Verify and iterate

After first build, verify:

1. **Run the flow standalone** — open the flow designer, click "Run with sample variables" with realistic test values. The Response should contain `dashboardUri` as a long `data:text/html;charset=utf-8,...` string.
2. **Decode the URI manually** if anything looks wrong — copy the URI to a browser address bar and load it. The HTML should render with the test data.
3. **Open the screen** — pageLoadAction fires, flow runs, webpage element loads the URI. Confirm the dashboard renders.
4. **Test filter changes** — change a filter value, hit refresh, confirm the URI changes and the webpage reflects the new data.
5. **Test theme switching** — flip the user's theme setting in Fuuz. Refresh the screen. The dashboard should render in the matching theme.

## Critical rules — never break these

1. **NEVER use base64 encoding** for the data URI. The V8 sandbox lacks `btoa`, and base64 inflates payload size by ~33%. Use the percent-encoding polyfill (see `references/encode-uri-polyfill.md`) — it's the only encoding that works.

2. **NEVER wrap output in `payload`**. The Response key must be `dashboardUri` at the top level. Wrapping breaks the screen's binding path.

3. **NEVER access `$state` from a saved script**. Saved scripts only see `$` (their input). All state must be bundled via the upstream JSONata node. Inline scripts in web flows CAN use `$state` and `$appConfig` directly — that's one reason to choose inline over saved when state access is the main thing the script needs.

4. **NEVER assume `encodeURIComponent` exists**. It doesn't in the V8 sandbox. Bring your own polyfill — `references/encode-uri-polyfill.md` has the full version with UTF-8 + surrogate pair handling.

5. **NEVER hardcode theme colors**. Use CSS variables and a `:root[data-theme="light"]` override block. The saved script swaps `<html data-theme="dark">` to `<html data-theme="light">` at build time based on `$metadata.settings.ThemeMode`.

6. **NEVER render times without a timezone**. Pass `$metadata.settings.TimeZone` through to the script and use `Intl.DateTimeFormat` with the `timeZone` option. Server-side or browser-default timezones produce wrong values for users in other regions.

7. **NEVER use uncapped queries**. Every query needs `first:` and ideally `where:` filters from context. A dashboard that pulls 50,000 records is a dashboard that doesn't load.

8. **NEVER render iframes/embedded webpages inside Cards, Tables, or non-Form containers**. They go inside a Form, period. For card-like UI, use a Markdown element with styled HTML and no JavaScript.

9. **NEVER use named GraphQL operations in `$query()`**. Inline queries inside flow `$query()` calls must be anonymous — `query(...)`, not `query MyQueryName(...)`. Named operations 400 silently.

10. **NEVER have an LLM generate pre-styled HTML at scale**. If a dashboard incorporates LLM-generated content (Anthropic API integration for AI insights), have the LLM produce compact prose or a structured object, then style it in the saved script. Output tokens cost ~5× input tokens, and LLMs produce inconsistent inline CSS — the math doesn't work.

## Inline JS and external libraries

**Inline JavaScript inside the dashboard HTML is fully supported.** The browser runs whatever you put in `<script>` tags. This is how Three.js, D3, Chart.js, FusionCharts, ECharts, Cytoscape, Mermaid — anything client-side — gets into a Fuuz dashboard.

**Public CDN libraries are allowable.** Reference them via `<script src="https://cdnjs.cloudflare.com/...">` or `<script src="https://cdn.jsdelivr.net/...">` in the HTML head.

**Always consult `references/web-libraries.md` before picking a library.** It's a vetted catalog of every library that meets the Fuuz selection criteria (permissive license, CDN-available, self-hostable, no telemetry, production-grade), organized by category — charting, network/graph, 3D, mapping/floor-plans, data grids, in-browser ML, diagrams, specialized viz. Each entry includes the exact CDN script tag, bundle size, license, Fuuz-specific use cases, and tradeoffs. There's also a quick decision matrix ("I need to show X → use Y") and a list of anti-patterns to avoid. Use it both to find the right library AND to confirm a library someone else suggested doesn't have a hidden gotcha (commercial license, telemetry, abandoned project).

The catalog's headline picks for the most common needs:
- **Charting** — ECharts (top pick for new dashboards), Chart.js (lightest for single charts), FusionCharts (keep for Gantt)
- **Network/graph** — Cytoscape.js
- **3D** — Three.js (stay on r128)
- **Maps / floor plans** — Leaflet
- **Heavyweight tables** — Tabulator
- **In-browser ML** — Transformers.js
- **Diagrams from text** — Mermaid

**The caveat — client network access matters.** The dashboard HTML runs in the user's browser via the iframe, so the CDN must be reachable FROM THE CLIENT, not from the Fuuz server. Most plant-floor environments have unrestricted internet, but some don't:

- Manufacturing networks may be air-gapped or have strict allowlists
- Some sites block all CDNs as a policy
- Corporate proxies may strip certain requests
- VPN-restricted networks may rewrite or block CDN domains

**Before committing to a CDN library, ask the user:**
- "Will this dashboard run on a network with general internet access, or a restricted plant network?"
- "Are there approved CDNs, or do libraries need to be self-hosted?"
- "Is there a corporate proxy that might block external requests?"

If client network access is restricted, two alternatives:

1. **Inline the library code** into the saved script's TPL_HEAD. The polyfill pattern is the same — concatenate the library text into the HTML. Three.js r128 is ~600 KB minified, D3 ~270 KB. **Inline, do not load from a CDN at runtime** — both production 3D dashboards fetch their libraries at build time into a `lib/` directory and embed them. A runtime CDN adds a network round trip on every render and fails outright on a floor network without egress. If inlining puts you over budget, compress the page rather than reaching for a CDN — see `references/large-payloads.md`.

2. **Self-host the library** on the same domain as Fuuz, and reference it relatively. Requires coordination with the platform admin but works for restricted environments.

**For security-conscious customers**, add Subresource Integrity (SRI) hashes to CDN script tags: `integrity="sha384-..." crossorigin="anonymous"`. cdnjs provides SRI hashes on every version page. This prevents a compromised CDN from serving altered code.

**Bundle size budget — aim for under 1 MB total of loaded JS/CSS per dashboard.** Above that, refresh perceived latency suffers, especially on plant-floor terminals with marginal hardware. Don't load multiple chart libraries on the same screen — pick one and stick with it.

For most users, the public CDN is fine — but always ask, because finding out the library can't load AFTER you've shipped the dashboard wastes everyone's time.

## Defaults — always ask, then confirm

When a user requests a dashboard, ALWAYS ask about these before writing code:

- **Filters?** — "What filters should the user be able to apply to this view?" Default to none if they decline, but always offer.
- **Cascade?** — "If they pick an Area, should the Work Center list narrow to only that Area's centers?" Default: yes.
- **Theme/timezone?** — Default behavior: theme-aware via `$metadata.settings.ThemeMode`, timezone-aware via `$metadata.settings.TimeZone`. Confirm before building.
- **Refresh?** — "Manual refresh only, or auto-refresh every N seconds?" Default: manual.
- **Data volume?** — "Roughly how many records will this dashboard typically show?" If >500, redesign with aggregation queries rather than raw fetch.

## Reference files

Load these as needed during build:

- `references/saved-script-template.md` — Complete saved-script skeleton with polyfill, theme detection, error handling
- `references/encode-uri-polyfill.md` — The percent-encoding polyfill, with explanation of why it's needed
- `references/web-libraries.md` — Vetted catalog of CDN-loadable libraries for charting, graphs, 3D, maps, tables, ML, diagrams. Read BEFORE picking any library for a dashboard. Includes decision matrix, license info, bundle sizes, Fuuz-specific use cases, and anti-patterns
- `references/troubleshooting.md` — Common errors and fixes
- `references/v8-sandbox-constraints.md` — What's missing from each context (web flow JS, JSONata, saved scripts)
- `references/smooth-refresh.md` — `window.name` persistence pattern for iframe reloads
- `references/postmessage-pattern.md` — Alternative architecture for true live updates (no iframe reload)
- `references/dispatcher-pattern.md` — One flow, multiple action outputs — for screens with several views from the same dataset
- `references/large-payloads.md` — The measured 2,048 KB limit and the gzip + `DecompressionStream` pattern that gets a page under it
- `references/3d-and-webgl.md` — WebGL is verified working. three.js r128 setup, and Draco model compression at 150–180x
- `references/live-subscriptions.md` — Real-time updates over the platform's own Socket.IO change bus, instead of polling
- `references/cross-tenant.md` — Reading more than one tenant: per-tenant token exchange, and which render element makes it possible
- `references/theming.md` — Following the viewer's ThemeMode, and reacting to a change without a reload
- `templates/minimal-dashboard.html` — A simple working dashboard template you can adapt

## Quick sanity checklist (use every build)

Before declaring a dashboard done:

- [ ] Flow type is `system`, not `web`
- [ ] `setContext` node runs immediately after `Request`, with all variables under `REQUEST`
- [ ] `try/catch` wraps all external calls
- [ ] Every query has `first:` pagination
- [ ] Every query selects only needed fields
- [ ] Every `$query()` GraphQL uses anonymous `query(...)`, not a named operation
- [ ] `Collect` node lists every output key from upstream queries
- [ ] JSONata bundler attaches `themeMode`, `timeZone`, and `filters` to the payload (and `appConfig` if the script needs design tokens)
- [ ] Saved script is a saved backend script — or an inline script if state/appConfig access justifies it
- [ ] Saved script outputs `{ dashboardUri: "data:text/html;charset=utf-8,..." }` — NOT wrapped in `payload`
- [ ] URI uses percent-encoding via polyfill — NOT base64
- [ ] HTML has `<html data-theme="dark">` swapped at build time
- [ ] CSS uses variables; `:root[data-theme="light"]` override block exists
- [ ] All times rendered with `Intl.DateTimeFormat` and the user's timezone
- [ ] Embedded webpage element is **inside a Form** (never inside a Card or other element)
- [ ] Card-like UI uses Markdown elements with styled HTML, NOT inline-JS webpages
- [ ] Screen's webpage element has dynamic fields `{ payload: [], context: ["components.DashboardForm.data.dashboardUri"] }`
- [ ] Screen webpage permissions are all enabled
- [ ] Filter cascade is wired (child filter `where` references parent filter value)
- [ ] If a CDN library is used — checked `references/web-libraries.md` to confirm license, maturity, and bundle size are acceptable
- [ ] CDN libraries (if used) — confirmed reachable from the client network, or inlined
- [ ] Only one chart library per dashboard (no mixing ECharts + Chart.js + Plotly)
- [ ] Total loaded JS/CSS budget under 1 MB per dashboard
- [ ] Tested standalone in flow designer with sample variables
- [ ] Tested with theme switched both directions
