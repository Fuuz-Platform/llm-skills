# Dispatcher Pattern — One Flow, Many Outputs

When a screen has multiple dashboard views — cards list, summary tiles, a Gantt chart, an iframe drill-down, an OEE breakdown — you have two choices:

1. **One flow per view.** Each view has its own backend flow, its own queries, its own saved script. Simple but duplicates query logic.
2. **One flow with a dispatcher.** The flow takes a `REQUEST.action` variable and routes to different rendering logic based on it. More complex initially, but pays off when views share data.

For dashboards that pull from the same dataset and render it different ways, the dispatcher pattern wins.

## When to use it

- Multiple views on the same screen, all reading the same underlying data
- Filters/parameters that should affect all views consistently
- You want to share KPI calculations across the views
- Drill-down patterns: user clicks a KPI → opens a detailed iframe view of just that segment

## Architecture

```
Request (variables include `action`)
  ↓
setContext({ REQUEST: { action, ...filters } })
  ↓
try/catch → Query(s) → Collect (shared data, ALL queries)
  ↓
Router node (Switch on $state.context.REQUEST.action)
  ├─ "cards"     → JSONata bundle → Saved Script "renderCards"     → { edges: [...with cardMd] }
  ├─ "summary"   → JSONata bundle → Saved Script "renderSummary"   → { summary: {...} }
  ├─ "dashboard" → JSONata bundle → Saved Script "renderDashboard" → { dashboardUri: "data:..." }
  ├─ "oee"       → JSONata bundle → Saved Script "renderOee"       → { dashboardUri: "data:..." }
  └─ "schedule"  → JSONata bundle → Saved Script "renderSchedule"  → { dashboardUri: "data:..." }
  ↓
Response
```

Each branch produces a different output shape based on what the screen needs:

| `action` | Returns | Bound to |
|---|---|---|
| `cards` (default) | `{ edges: [...] }` with `cardMd` field per node | Native Fuuz card list element |
| `summary` | `{ summary: {...} }` — totals, counts, KPIs | Markdown element or screen context |
| `dashboard` | `{ dashboardUri: "data:..." }` | Embedded webpage element (inside a Form) |
| `oee` | `{ dashboardUri: "data:..." }` | Embedded webpage element (inside a Form) |
| `schedule` | `{ dashboardUri: "data:..." }` | Embedded webpage element (inside a Form) |

## Screen-side wiring

The screen has multiple Forms, one per action. Each Form calls the same flow with a different action:

```javascript
// CardForm pageLoadAction:
$executeFlow("dashboardFlowId", {
  variables: { action: "cards", areaId: $components.FilterForm.data.areaId, ... }
})

// SummaryForm pageLoadAction:
$executeFlow("dashboardFlowId", {
  variables: { action: "summary", areaId: $components.FilterForm.data.areaId, ... }
})

// DashboardForm pageLoadAction (the iframe one):
$executeFlow("dashboardFlowId", {
  variables: { action: "dashboard", areaId: $components.FilterForm.data.areaId, ... }
})
```

Each form's response shape matches what its bound element expects.

## Inside the script — single script vs. multiple

Two variations:

**(a) Multiple saved scripts.** Each action gets its own script. Cleaner separation, easier to test individually. Worse if they share large rendering helpers — duplicated code.

**(b) Single saved script with internal dispatcher.** One script, switches on `$.type` (which the JSONata bundler maps from `$state.context.REQUEST.action`). Helpers shared. Bigger file but DRY.

Either works. For 2-3 actions with simple rendering, separate scripts are fine. For 5+ actions sharing a lot of helpers (color tokens, pill builders, date formatters, layout helpers), one script wins.

Sketch of variant (b):

```javascript
function main($) {
  const action = $.type || 'cards';

  // Shared helpers, design tokens, palette — defined once
  const C = { /* color tokens */ };
  function pill(text, color) { /* ... */ }
  function fmtTime(iso) { /* ... */ }
  function escapeHtml(s) { /* ... */ }

  // Route to the right renderer
  switch (action) {
    case 'cards':     return renderCards($);
    case 'summary':   return renderSummary($);
    case 'dashboard': return renderDashboard($);
    case 'oee':       return renderOee($);
    case 'schedule':  return renderSchedule($);
    default:          return renderCards($);  // fallback
  }

  function renderCards(input)     { /* ... */ }
  function renderSummary(input)   { /* ... */ }
  function renderDashboard(input) {
    const html = TPL_HEAD + JSON.stringify(buildState(input)) + TPL_TAIL;
    return { dashboardUri: 'data:text/html;charset=utf-8,' + encodeURIComponentPolyfill(html) };
  }
  // ... etc
}

return main($);
```

## Drill-down pattern

The dispatcher pattern enables drill-downs naturally. User clicks a KPI tile in the summary view → screen sets a new variable (`drilldownTarget`) → calls the flow with `action: "dashboard"` and the drill-down filter → embedded webpage refreshes with the detailed view.

```javascript
// Action step on KPI tile click:
$components.FilterForm.fn.setValue('drilldownTarget', 'critical-only');
$components.DashboardForm.fn.refresh();
```

The flow's setContext picks up the new `drilldownTarget`, the queries filter by it, the renderer produces the focused view.

## When NOT to use this

- Dashboards that don't share data between views — separate flows are simpler
- Single-view dashboards — adding a dispatcher buys nothing
- When the views have wildly different query needs — the shared Query/Collect section becomes a mess

The dispatcher pattern is for when views are LIKE each other (same data, different shapes), not when they're DIFFERENT from each other (different data entirely).

## Caveat: context navigation between iframe and parent

If a dashboard view contains links or actions that should change the screen's active view (e.g., a card in the cards view that should open the dashboard view), the iframe needs to communicate with the parent screen. The straightforward way:

```javascript
// In the dashboard's inline JS:
function navigateTo(action, params) {
  parent.postMessage({ fuuzNav: true, action: action, params: params }, '*');
}

// In the host screen's onMount action:
window.addEventListener("message", function(e) {
  if (e.data && e.data.fuuzNav === true && e.data.action) {
    $components.FilterForm.fn.setValue('action', e.data.action);
    $components.DashboardForm.fn.refresh();
  }
});
```

This is the same pattern described in `references/postmessage-pattern.md`, used here for navigation rather than live updates. The two patterns compose — a single dashboard can use postMessage for both navigation and live data delivery.
