# Smooth Refresh

Iframe reloads cause visible flashes — the browser tears down the old DOM and parses the new one. There's always a brief frame where the iframe goes blank or white. This is unavoidable with the iframe-reload pattern, but you can hide most of it.

## The full smoothing stack

### 1. Body fade-in transition

CSS:
```css
body {
  opacity: 0;
  transition: opacity 0.35s ease-out;
}
body.ready { opacity: 1; }
```

JS, run on first animation frame:
```javascript
requestAnimationFrame(() => {
  document.body.classList.add('ready');
});
```

This masks the brief render-incomplete period right after the iframe loads. The user sees the dashboard fade in rather than pop in.

### 2. Match iframe background to dashboard background

If the screen's webpage element (or its parent container) has a white default background, you'll see a white flash during reload even with body fade-in — the flash happens BEFORE your CSS loads.

Set the parent container's background in the screen design to match the dashboard's background color. For dark theme dashboards, set `background: #0b0f17` on the webpage element container. For light, `#f1f5f9` or whatever your background variable resolves to.

### 3. Persist state via `window.name`

The browser preserves `window.name` across same-origin navigation. For a data URI, the new "page" is the new data URI — same origin — so `window.name` survives the reload. Use this to keep state.

```javascript
// Read persisted state early, before any rendering
let persistentState = {};
try {
  persistentState = window.name ? JSON.parse(window.name) : {};
} catch (e) { persistentState = {}; }

// Apply persisted state
if (persistentState.camera) {
  applyCamera(persistentState.camera);
}
if (persistentState.hiddenItems) {
  for (const name of persistentState.hiddenItems) hideItem(name);
}

// Save state on every relevant interaction
function savePersistentState() {
  const state = {
    camera: getCurrentCamera(),
    hiddenItems: getHiddenItemNames(),
    // ...whatever else needs to survive
  };
  try { window.name = JSON.stringify(state); } catch (e) {}
}

// Save periodically as a safety net
setInterval(savePersistentState, 2000);
// Save on every interaction
document.addEventListener('click', savePersistentState);
// Save right before unload
window.addEventListener('beforeunload', savePersistentState);
```

### 4. Interpolate animated state across refresh

For dashboards with moving elements (vehicles on a path, animated values), the persistent state can store the last animation position. When the new iframe loads, it picks up from where the old one left off instead of snapping to the data's current value.

```javascript
// On load, read last-known position from persistentState
const lastPos = persistentState.itemPositions?.[itemId] ?? null;
const targetPos = computeTargetPos(item);

if (lastPos != null) {
  // Animate from last-known to current target over ~500ms
  animateBetween(lastPos, targetPos, 500);
} else {
  // First load — snap to target
  setPosition(itemId, targetPos);
}
```

### 5. Throttle refresh cadence

Auto-refreshing every 1-2 seconds compounds the perception of flash. 8-15 seconds feels live and gives operators time to read the screen. Configure the screen's refresh interval (or the Form's `intervals` property) accordingly.

## The limits of iframe-reload smoothing

Even with everything above, iframe-reload has fundamental limits:

- The browser MUST tear down and rebuild the DOM
- Inline `<script>` tags MUST re-execute, including library bundle parsing (Three.js init alone is ~100ms)
- Animation timers reset
- Any DOM-attached state not in `window.name` is lost

For dashboards that need true zero-flash live updates, switch to the postMessage architecture. See `references/postmessage-pattern.md`.

## Quick decision: smoothing vs postMessage

Use smoothing (this file) when:
- Refresh cadence is >= 8s
- Dashboard initialization is fast (<200ms)
- Users don't watch for sub-second changes

Use postMessage when:
- Updates must appear within a single frame
- Refresh cadence is <= 2s
- Initialization is heavy (Three.js scene with hundreds of meshes)
- Continuous animations span across refreshes
