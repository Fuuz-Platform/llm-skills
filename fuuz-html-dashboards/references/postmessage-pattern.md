# PostMessage Pattern (Alternative Architecture)

For dashboards that need truly seamless live updates — no iframe reload, no flash, no animation reset — drop the iframe-reload pattern and switch to postMessage.

## The architecture

```
┌─────────────────┐         postMessage          ┌─────────────────┐
│  Fuuz Screen    │ ───────────────────────────▶ │  Iframe (data:) │
│  (parent page)  │                              │                 │
│                 │ ◀─────────────────────────── │  Persistent     │
└─────────────────┘     postMessage (ready)      │  Three.js scene │
        │                                         └─────────────────┘
        │ executes flow on interval / event
        │ gets new data
        ▼
    pushes data to iframe via postMessage
```

Instead of regenerating the URI each refresh, you:

1. Load the iframe ONCE with the dashboard's static structure (HTML + JS + CSS, no data).
2. On the screen side, run the data flow on a timer or event.
3. When the flow returns, post the data into the iframe via `postMessage`.
4. The iframe's JS receives the message and updates the scene/cards/chart in place.

## When to use this

- Refresh cadence ≤ 2s
- Dashboard has heavy initialization (Three.js scene, many DOM nodes)
- Continuous animations need to span across refreshes
- Operators are watching for instant updates

## Trade-offs

**Pros:**
- Zero flash
- Sub-millisecond updates
- Animations don't reset
- `window.name` no longer needed (state survives naturally)

**Cons:**
- More complex setup
- Iframe and parent must coordinate via message protocol
- Initial load time is the same — only subsequent updates are faster
- Can't easily cache the rendered dashboard URL (URL is generic now)

## Implementation outline

### Saved script changes

The script now builds the SHELL — empty dashboard structure, no data. The URI is static (or only changes when theme/structure changes).

```javascript
function main(input) {
  // ... theme detection ...
  const html = themedHead + JSON.stringify({}) + TPL_TAIL;
  // Same encoding as before
  const dataUri = 'data:text/html;charset=utf-8,' + encodeURIComponentPolyfill(html);
  return {
    dashboardUri: dataUri,
    // ALSO return the data separately for postMessage delivery
    payload: buildDashboardData(input)
  };
}
```

### Inline HTML's JS changes

The iframe's JS listens for postMessage and applies updates:

```javascript
let dashboardState = STATE; // initial state (may be empty)

window.addEventListener('message', (event) => {
  // Validate origin in production
  if (event.data?.type === 'dashboardUpdate') {
    applyUpdate(event.data.payload);
  }
});

// Tell parent we're ready to receive
parent.postMessage({ type: 'dashboardReady' }, '*');

function applyUpdate(payload) {
  // Diff against current state, update only what changed
  for (const item of payload.items) {
    const existing = findItem(item.id);
    if (existing) {
      updateItemInPlace(existing, item);  // smooth animation
    } else {
      addItem(item);
    }
  }
  // Remove items no longer in payload
  removeStaleItems(payload.items);
}
```

### Screen-side wiring

The screen's webpage element loads the URI once (it never changes after first load). The dashboard Form's flow runs on interval and pushes the result to the iframe:

```javascript
// In screen action (e.g., onInterval or onFlowResult):
const iframe = document.querySelector('#webpage-element iframe');
const payload = $components.DashboardForm.data.payload;
iframe.contentWindow.postMessage({
  type: 'dashboardUpdate',
  payload: payload
}, '*');
```

Wire the action to fire when the Form's data changes (via `onValueChange` or whatever your screen designer calls it).

### Re-render on theme change

When the theme switches, you DO need to reload the iframe (theme is baked into the static HTML structure). Detect theme changes on the screen side and refresh the Form, which will rebuild the URI with the new theme.

## Caveats

- **CORS**: data URIs have unique origins by browser. `parent.postMessage` works because `*` accepts any target origin, but `event.origin` will be `null` in the iframe. Don't rely on origin validation — instead, validate message structure (`event.data.type` check).
- **Bidirectional**: the iframe can post back to the parent (e.g., "user clicked an AMR card"). Use this for filter interactions inside the dashboard.
- **Initial state**: the iframe needs SOMETHING to render on first load. Either include initial data in the URI, or render a loading state and wait for the first postMessage.
- **State sync**: if the iframe persists state internally (e.g., camera position), the parent doesn't need to know about it. The parent only pushes data updates.

## Decision tree

Use the iframe-reload pattern (default) if:
- This is your first Fuuz dashboard — start simple
- Refresh cadence is fine at 8-30s
- The dashboard is mostly static views

Switch to postMessage if:
- You've shipped the iframe-reload version and operators want sub-second updates
- The dashboard has heavy initialization and refresh feels heavy
- You need continuous animations that span across refreshes (e.g., AGV motion paths)

PostMessage is a significant refactor. Don't start there. Get the iframe-reload version working first, then consider this if the smoothness isn't enough.
