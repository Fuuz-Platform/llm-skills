# Troubleshooting

Common failure modes and how to diagnose them. Organized by symptom.

## The dashboard doesn't appear at all (blank iframe)

**Likeliest cause: dynamic fields are wrong.**

Check the embedded webpage element's `dynamicFields`:

```json
{
  "payload": [],
  "context": ["components.DashboardForm.data.dashboardUri"]
}
```

If the dynamic fields don't include the path to `dashboardUri`, the webpage element doesn't re-render when the flow returns. The path is whatever your dashboard Form's `elementName` is — if you renamed it to `MyForm`, use `components.MyForm.data.dashboardUri`.

**Second cause: webpage element permissions disabled.**

Open the webpage element config and enable all permissions. Inline JavaScript needs them to execute. Without script permission, the URI loads but nothing runs.

**Third cause: the URI is being wrapped.**

If the flow returns `{ payload: { dashboardUri: "..." } }`, the screen binding `$components.DashboardForm.data.dashboardUri` resolves to `undefined`. The Response node must return `{ dashboardUri: "..." }` at the top level — no wrapping. The data path then resolves to a value, and the webpage element loads it.

## "Cannot read property 'X' of undefined" in the flow

The saved script is reading a path that doesn't exist in its input.

**Diagnostic step:** add a `_debug` block to the script return:

```javascript
return {
  dashboardUri: '...',
  _debug: {
    inputKeys: Object.keys($),
    expectedKey: $.automatedGuidedVehicle,  // whatever you expected
    actualShape: JSON.stringify($).slice(0, 500)
  }
};
```

Run the flow standalone, check the response. The `_debug` block shows what the script actually received versus what you expected. Usually the cause is one of:

- The Collect node didn't list the right output key
- The JSONata bundler ran in the wrong order (data was overwritten)
- A query failed silently and produced no output (no `try/catch` around it)

Remove `_debug` once fixed.

## The dashboard renders but the data is empty

The URI is being built but the query results aren't reaching the script.

**Check the Collect node first.** It must list every output key from the queries upstream. Missing keys silently drop data.

**Then check the JSONata bundler.** A common mistake:

```jsonata
$merge([
  { "themeMode": $state.context.REQUEST.themeMode },
  $
])
```

This works, but if you accidentally write it the other way around, `$` overwrites the theme. Order matters in `$merge` — later objects override earlier ones.

## The dashboard renders but in the wrong theme

The theme isn't reaching the script, or the script isn't detecting it.

**Check the order:**

1. Screen passes `themeMode: $metadata.settings.ThemeMode` in the `$executeFlow` variables.
2. Flow's Request node receives it as a variable.
3. setContext stores it under `REQUEST.themeMode`.
4. JSONata bundler attaches `themeMode: $state.context.REQUEST.themeMode` to the script's input.
5. Saved script reads `$.themeMode`.
6. Saved script does `'data-theme="' + theme + '"'` substitution before encoding.

If any step is missed, the script falls back to dark.

**Verify in the URI itself.** Decode the URI (paste into address bar) and view source. Look for `<html lang="en" data-theme="dark">` vs `data-theme="light"`. If the HTML still says `dark` when light was requested, the substitution failed — verify the script's `replace` call matches the actual placeholder.

## Times display in the wrong timezone

Two common causes:

**(a) The script isn't getting the timezone.** Same diagnostic as theme — trace the value through Request → setContext → bundle → script.

**(b) The renderer uses `toLocaleString()` without `timeZone`.** This picks up the browser's local timezone. On a centralized server or a kiosk in a remote region, this is wrong. Always pass the user's timezone explicitly:

```javascript
new Intl.DateTimeFormat('en-US', {
  timeZone: STATE.timeZone || 'UTC',
  hour: '2-digit', minute: '2-digit'
}).format(new Date(isoString))
```

## The dashboard renders the first time but doesn't refresh

The webpage element isn't re-rendering when the flow output changes.

**Check `dynamicFields.context`** — it must include the path to `dashboardUri`. If it lists only `payload` fields, the webpage element doesn't watch the response.

**Alternatively, the flow may not be re-executing.** If you have a Refresh button, verify it calls `$components.DashboardForm.fn.refresh(...)`, not just changing a filter value. Refreshing the Form is what triggers the flow re-run.

## "URI too long" or browser refuses to load

`data:` URIs have practical browser limits, typically around 2MB. If your dashboard's HTML + state exceeds this, the browser silently rejects it.

**Fixes (in order of preference):**

1. **Move state-heavy content out of the URI.** Don't inline thousands of records in `STATE`. Aggregate server-side, send summary stats only.
2. **Strip whitespace from the HTML template.** Use a minifier — backticked template literals preserve every space and newline. For Three.js with hundreds of `<script>` lines, this matters.
3. **Compress the page** — gzip it in the flow and inflate with `DecompressionStream` in the page. This is the measured fix and it beats every alternative; see `large-payloads.md`. Reaching for a runtime CDN to shrink the URI trades a size problem for a network-dependency problem.
4. **Switch to the postmessage pattern** — see `references/postmessage-pattern.md`. Instead of regenerating the entire iframe on refresh, send delta updates via postMessage to a persistent iframe.

## Refresh causes a visible flash / glitch

Iframe reload always causes a brief blank moment. Mitigate by:

1. **Body fade-in transition** — see the saved script template. Set body opacity to 0 in CSS, add `body.ready` class on first frame with transition. Smooths the appearance.
2. **Set the iframe's parent background** to match the dashboard's background color. If the parent screen renders a white background during reload, you'll see a flash regardless of body opacity. Set `background-color: #0b0f17` (or matching theme color) on the webpage element's container.
3. **Persist state via `window.name`** — see `references/smooth-refresh.md`. Camera position, hidden items, interpolation start points all survive the iframe reload.
4. **Throttle refresh frequency.** Auto-refreshing every 2 seconds compounds the perception. 8-15s feels live and gives users time to read.
5. **Consider postMessage architecture.** For true zero-flash live updates, drop the iframe-reload pattern entirely.

## "encodeURIComponent is not defined" in the flow

You used `encodeURIComponent` directly. It doesn't exist in the V8 sandbox. Use the polyfill from `references/encode-uri-polyfill.md`.

## "Identifier 'X' has already been declared" in the inline HTML's JS

Duplicate `let`/`const` declaration in the inline `<script>` block. Common causes:

- Pasting two versions of the same code into the template
- A loop unintentionally redeclaring at module scope
- Persistent state code declared twice

Solution: open the rendered HTML in a browser, check the console, find the line number, fix the duplicate.

## "Cannot access 'X' before initialization"

Temporal dead zone error — a `let` or `const` is being used before its declaration is reached.

In the saved script: hoist the declaration to the top of the function, or use `var` (function-scoped).

In the inline HTML's JS: usually means a function declared elsewhere references a variable declared later. Move the variable's declaration up.

## The webpage element loads but shows the URI as text

The Fuuz webpage element is configured to display content, not navigate to URIs. Verify:

- `path` field is set to the `dashboardUri` binding (not the URI string itself)
- The "render mode" setting is set to iframe/URL mode, not inline-content mode
- The data binding is resolving to a string starting with `data:text/html;charset=utf-8,...` (not the literal text `${dashboardUri}` or similar)

## Cascading filters don't cascade

The child filter's `where` clause isn't referencing the parent's value.

Example — WorkCenter SelectInput's query should have:
```graphql
where: { areaId: { _eq: $components.FilterForm.data.areaId } }
```

If the WorkCenter list shows all WorkCenters regardless of Area selection, the `where` clause is missing or pointing at a different data path.

Also verify the parent filter's value is actually stored at the expected path. SelectInput stores the selected value at `$components.FilterForm.data.<fieldName>`. If the field name was changed, the binding breaks silently.

## The flow runs successfully but the screen shows nothing

The flow may be returning data, but the screen's Form isn't updating. Try:

1. Open the Fuuz screen's runtime debugger and inspect `$components.DashboardForm.data`. Is `dashboardUri` populated?
2. Check the Form's `pageLoadAction` — does it actually call `$executeFlow("yourFlowId", ...)` with the right flow ID?
3. Verify the flow ID hasn't changed (system flows occasionally get re-keyed during deployment).
4. Check the screen designer's network tab in browser dev tools — is the flow call happening at all? Is it returning 200?
