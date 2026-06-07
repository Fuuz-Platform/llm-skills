# Saved script template

The complete structure of the JavaScript saved backend script that builds the dashboard. Adapt the body to your specific dashboard but keep the scaffolding.

## Why a saved script (not inline)

Inline scripts in the data flow designer have several downsides:

- Can't be version-controlled separately
- No syntax highlighting on large blocks
- Inline web-flow JS has more restrictions than saved scripts — no `?.`, no `??`, `var` only, no `for...of`, no `Map`/`Set`
- Hard to test in isolation

Saved backend scripts:

- Full ES2020+ syntax (`?.`, `??`, `const`/`let`, `for...of`, `Map`, `Set`, arrow functions, template literals)
- Can be re-used across flows
- Receive `$` as their input (whatever the upstream node bundled)
- Cannot access `$state`, `$metadata`, `$components`, or any other web-flow global. Anything they need must come through `$`.

## Skeleton

```javascript
/**
 * dashboardBuilder.js — generates the dashboard HTML as a data URI
 *
 * Input ($): {
 *   automatedGuidedVehicle: { edges: [...] },  // or whatever your queries return
 *   themeMode: 'Dark' | 'Light',
 *   timeZone: 'America/New_York',
 *   filters: { areaId, workCenterId, ... }
 * }
 * Output: { dashboardUri: 'data:text/html;charset=utf-8,...' }
 */

// ── HTML template ──────────────────────────────────────────
// The template is split into HEAD + tail with a JSON.stringify'd state object
// injected between them. This avoids any need to escape the state inside the HTML.
const TPL_HEAD = `<!DOCTYPE html>
<html lang="en" data-theme="dark">
<head>
<meta charset="UTF-8">
<title>Dashboard</title>
<style>
  :root, :root[data-theme="dark"] {
    --bg: #0b0f17; --panel: #131826; --text: #e2e8f0; --dim: #94a3b8;
    --accent: #3b82f6; --good: #22c55e; --warn: #f59e0b; --danger: #ef4444;
    --border: rgba(148,163,184,0.14);
  }
  :root[data-theme="light"] {
    --bg: #f1f5f9; --panel: #ffffff; --text: #0f172a; --dim: #475569;
    --accent: #2563eb; --good: #16a34a; --warn: #d97706; --danger: #dc2626;
    --border: rgba(15,23,42,0.10);
  }
  body {
    background: var(--bg); color: var(--text);
    font-family: Roboto, Helvetica, Arial, sans-serif;
    margin: 0; padding: 20px;
    opacity: 0; transition: opacity 0.35s ease-out;
  }
  body.ready { opacity: 1; }
  /* ...your dashboard styles... */
</style>
</head>
<body>
<div id="root"></div>
<script>
const STATE = `;

const TPL_TAIL = `;
// Use the timezone we received from the flow
const TZ = STATE.timeZone || 'UTC';
function formatTime(iso) {
  if (!iso) return '';
  return new Intl.DateTimeFormat('en-US', {
    timeZone: TZ,
    hour: '2-digit', minute: '2-digit', second: '2-digit'
  }).format(new Date(iso));
}

// Render the dashboard from STATE
function render() {
  // ...your dashboard rendering logic here...
  document.body.classList.add('ready');
}

render();
</script>
</body>
</html>`;

// ── Encoding polyfill ──────────────────────────────────────
// See references/encode-uri-polyfill.md for full explanation
function encodeURIComponentPolyfill(str) {
  const unreserved = {};
  const safe = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_.~";
  for (let i = 0; i < safe.length; i++) unreserved[safe.charCodeAt(i)] = true;
  const out = [];
  const hex = (n) => { const h = n.toString(16).toUpperCase(); return h.length === 1 ? '0' + h : h; };
  for (let i = 0; i < str.length; i++) {
    const c = str.charCodeAt(i);
    if (c < 0x80) {
      out.push(unreserved[c] ? str.charAt(i) : '%' + hex(c));
    } else if (c < 0x800) {
      out.push('%' + hex(0xC0 | (c >> 6)) + '%' + hex(0x80 | (c & 0x3F)));
    } else if (c >= 0xD800 && c <= 0xDBFF) {
      const next = str.charCodeAt(i + 1);
      const cp = 0x10000 + ((c & 0x3FF) << 10) + (next & 0x3FF);
      i++;
      out.push('%' + hex(0xF0 | (cp >> 18))
             + '%' + hex(0x80 | ((cp >> 12) & 0x3F))
             + '%' + hex(0x80 | ((cp >> 6) & 0x3F))
             + '%' + hex(0x80 | (cp & 0x3F)));
    } else {
      out.push('%' + hex(0xE0 | (c >> 12))
             + '%' + hex(0x80 | ((c >> 6) & 0x3F))
             + '%' + hex(0x80 | (c & 0x3F)));
    }
  }
  return out.join('');
}

// ── State builders ─────────────────────────────────────────
// Shape your query results into the structure the client-side renderer expects.
// Keep this small — most data shaping should happen in JSONata before the script.

function buildDashboardState(input) {
  // Example: transform AGV edges into a flat array
  const edges = input?.automatedGuidedVehicle?.edges ?? [];
  const items = edges
    .map((edge) => edge?.node)
    .filter(Boolean)
    .map((node) => ({
      name: node.name ?? '',
      battery: node.batteryPercent ?? 0,
      // ...other fields
    }));

  return {
    items,
    timeZone: input.timeZone ?? 'UTC',
    filters: input.filters ?? {}
  };
}

// ── Main ───────────────────────────────────────────────────
function main(input) {
  // Theme resolution. The flow bundled themeMode into the payload; default Dark.
  const rawTheme = input?.themeMode ?? 'Dark';
  const theme = String(rawTheme).toLowerCase() === 'light' ? 'light' : 'dark';

  // Build the state object that the client-side JS will consume
  const state = buildDashboardState(input);

  // Inject theme into the html element BEFORE concatenation so CSS applies
  // on first paint with no flash of unstyled content
  const themedHead = TPL_HEAD.replace('data-theme="dark"', 'data-theme="' + theme + '"');

  const html = themedHead + JSON.stringify(state) + TPL_TAIL;
  const dataUri = 'data:text/html;charset=utf-8,' + encodeURIComponentPolyfill(html);

  return { dashboardUri: dataUri };
}

return main($);
```

## Critical conventions in this template

1. **The HTML is split into head and tail strings** with `JSON.stringify(state)` between them. This is the safest way to inject data into HTML — no escaping issues, no SQL-injection-style risks, no string concatenation inside the HTML's own `<script>` block.

2. **Theme is swapped via `replace`** on the head template before concatenation. Don't try to inject theme via `<script>` at runtime — you'll get a flash of dark theme before light kicks in.

3. **Timezone is read from the state** in the client-side script using `Intl.DateTimeFormat`. Never use `Date.toLocaleString()` without `timeZone` — it picks up the browser's local TZ which is wrong for users in other regions.

4. **`body.ready` class** with opacity transition masks the brief flash during iframe reload. Set it on first frame.

5. **The polyfill is inlined**. Don't try to require/import it — saved scripts can't load modules.

## What this template doesn't include

If your dashboard needs them, add:

- **Three.js / D3 / Chart.js** — inline via `<script src="https://cdnjs.cloudflare.com/...">` in the head. The Fuuz environment has internet access in the iframe context.
- **Persistent state across refresh** — see `references/smooth-refresh.md`
- **Multiple data sources** — extend `buildDashboardState` to read additional query results from input
- **Filter display** — render the active filters somewhere in the dashboard header for operator clarity

## Don't do this

```javascript
// ❌ WRONG — saved scripts can't see $state
const tz = $state.context.REQUEST.timeZone;

// ❌ WRONG — return wrapped in payload
return { payload: { dashboardUri: dataUri } };

// ❌ WRONG — encodeURIComponent doesn't exist
const uri = 'data:text/html,' + encodeURIComponent(html);

// ❌ WRONG — btoa doesn't exist either
const uri = 'data:text/html;base64,' + btoa(html);

// ❌ WRONG — partial encoding misses dozens of characters
const uri = 'data:text/html,' + html.replace(/[<>'"]/g, '_');
```
