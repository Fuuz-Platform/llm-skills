# Following the viewer's theme

A dashboard should match the app around it, and change when the viewer changes it — without a
reload.

## Resolving the mode

Three sources, in order. Take the first that yields `light` or `dark`:

1. **`window.__THEME__`** — a dynamic field the flow injects from `$metadata.settings.ThemeMode`.
   Authoritative when present, and the only option for a page that is *not* same-origin.
2. **The parent's stored setting** — `JSON.parse(parent.localStorage.getItem('settings')).ThemeMode`.
   Works because a `DisplayText` `srcdoc` page is same-origin. Wrap it in `try/catch`: an
   `EmbeddedWebpage` `data:` URI is cross-origin and this throws.
3. **`window.matchMedia('(prefers-color-scheme: light)')`** — the OS preference, as the floor.

```js
function themeMode() {
  var m = (window.__THEME__ || '').toLowerCase();
  if (m !== 'light' && m !== 'dark') {
    try { m = (JSON.parse(parent.localStorage.getItem('settings') || '{}').ThemeMode || '').toLowerCase(); }
    catch (e) { /* cross-origin — fall through */ }
  }
  if (m !== 'light' && m !== 'dark') {
    m = (window.matchMedia && window.matchMedia('(prefers-color-scheme: light)').matches) ? 'light' : 'dark';
  }
  return m;
}
```

## Reacting to a change

Stamp `data-theme` on the root element and drive every colour from CSS custom properties, so a
switch is one attribute write rather than a re-render. Watch the parent's setting and restamp
when it changes — the viewer flips the app's theme and the dashboard follows, with no reload and
no loss of scroll, selection or camera position.

## For 3D

three.js needs **numbers, not CSS variables**. Resolve each colour once per theme change and
push the integers into the materials — a `var(--x)` string handed to a material is silently
wrong.
