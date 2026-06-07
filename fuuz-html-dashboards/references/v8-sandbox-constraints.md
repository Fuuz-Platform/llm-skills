# V8 Sandbox Constraints (by context)

Fuuz runs JavaScript in several different contexts, each with different available features. Get this wrong and you'll spend hours debugging cryptic errors.

## The three contexts

| Context | Where it runs | What's available |
|---|---|---|
| **JSONata expression** | Screen transforms, validation, flow transforms | JSONata stdlib + `$query`, `$mutate`, `$integrate`, `$state`, `$metadata`, `$components` |
| **Inline web flow JavaScript** | Script nodes inside a web flow | Restricted ES5-ish JavaScript + `$state` access |
| **Saved backend script** | Reusable scripts called from flows | Full ES2020+ but isolated input (no `$state`) |

Pick the right context for the job. Use saved scripts whenever the logic is non-trivial.

## JSONata constraints

These are the hardest to debug because errors are silent or vague.

- **No `??`** (nullish coalescing) anywhere in JSONata
- **No `?.`** (optional chaining) anywhere in JSONata
- **Function bodies must be single expressions** — no `;` separators, wrap in `{}` if needed
- **`$map` with named function references fails silently** — use inline lambdas: `$nodes.{}` path mapping pattern instead
- **`$append([], ...)` required** to prevent singleton unwrapping when building arrays
- **`$$.$variableName`** for outer-scope access from inside `$evalHandlebarsTemplate` or similar
- **`setContext` takes a single object argument** — `setContext({ key: value })`, not `setContext("key", value)`
- **`$type(val) = "number"`** instead of `$number()` for null-safe numeric type checks
- **`$exists()` is unreliable** for null checks — prefer explicit comparison
- **Relay query result root is `$.modelName.edges.node`** — adjust based on actual model name

## Inline web flow JavaScript constraints

This is the most restrictive context. Scripts attached as inline-script nodes in web flows.

- **No `??`** — use explicit ternary: `val != null ? val : default`
- **No `?.`** — use `&&` chains: `obj && obj.field && obj.field.code`
- **No `let`/`const`** — only `var`
- **No `for...of`** — use indexed `for (var i = 0; i < arr.length; i++)`
- **No `Map`, `Set`** — use plain objects (`{}`) and inclusion checks
- **No `Infinity`, no `isFinite()`** — use sentinel values like `9999999999999`
- **No template literals (backticks)** in some sandbox versions — use string concatenation
- **`$state` is accessible** — read/write flow context

When in doubt, write code that would pass an ES5 linter.

## Saved backend script constraints

The most permissive context — but with one big restriction.

- **Full ES2020+ syntax works** — `?.`, `??`, `const`/`let`, `for...of`, `Map`, `Set`, arrow functions, template literals, destructuring, spread
- **NO `$state`, `$metadata`, `$components` access** — saved scripts only see their input as `$`
- **NO browser globals** — no `window`, `document`, `fetch`, `XMLHttpRequest`, `console` (well, sometimes `console.log` works, sometimes silent — don't rely on it)
- **NO encoding/decoding helpers** — `encodeURIComponent`, `decodeURIComponent`, `btoa`, `atob` all missing
- **NO `require()`** or ES module imports — code must be self-contained
- **NO filesystem, no network calls** — pure data transformation only
- **`Date` works**, including `Date.now()`, `new Date(iso)`, `.toISOString()`
- **`JSON.parse` / `JSON.stringify` work**
- **`Intl.DateTimeFormat` works** — use it for timezone-aware formatting
- **`Math.*` works**

## Cross-context gotchas

### `$state.context` lives outside the saved script

Inline web flow scripts can read/write `$state.context.foo` directly. Saved scripts cannot. If you need context data in a saved script, the upstream flow must bundle it into the script's input via a JSONata transform:

```jsonata
$merge([$, {
  "themeMode": $state.context.REQUEST.themeMode,
  "timeZone": $state.context.REQUEST.timeZone,
  "userId": $state.context.REQUEST.userId
}])
```

### `$appConfig` also outside saved scripts

Same restriction. Inline scripts and JSONata both see `$appConfig` (the platform-wide app config — design system tokens, demo config, integration settings). Saved scripts don't. If a saved script needs design tokens or other app config, bundle them via the same JSONata transform:

```jsonata
$merge([$, {
  "appConfig": $appConfig
}])
```

Then read `$.appConfig.designSystem.colors` etc. from inside the saved script.

### `setContext` signature differences

- **Inline web flow JS**: `setContext("key", value)` works
- **JSONata transforms in flows**: `setContext({ key: value })` — single object argument
- **Saved scripts**: cannot call `setContext` at all — they only return data

### `$query()` does NOT support named GraphQL operations

When using `$query()` inline inside JSONata or a flow node, the GraphQL string must use the anonymous operation form:

```graphql
# ✓ CORRECT
query($where: WorkOrderWhereInput, $first: Int) {
  workOrder(where: $where, first: $first) {
    total
    edges { node { id name } }
  }
}

# ✗ WRONG — named operations 400 silently
query GetWorkOrders($where: WorkOrderWhereInput, $first: Int) {
  workOrder(where: $where, first: $first) {
    total
    edges { node { id name } }
  }
}
```

This trips people up because the GraphQL playground/IDE allows (and even encourages) named operations. Strip the operation name for `$query()` calls.

### Updating screen context from a flow response

If your dashboard flow needs to push data BACK into the screen's context (for KPI tiles bound to screen variables, for example), use `mergeContext` in a JSONata transform:

```jsonata
(
  $ctx := {
    "aiInsight": {
      "md": $coalesce([$.insightMd, ""]),
      "model": $coalesce([$.insightModel, ""])
    }
  };
  $components.Screen.fn.mergeContext($ctx)
)
```

Then the screen can bind to `$state.context.aiInsight.md` directly. This is the recommended pattern for dashboard outputs that other screen elements depend on — write to screen context once via the flow, then any element on the screen can read it.

### `$log` works… sometimes

`$log` may output to the flow's execution log in some contexts but not others. Don't rely on it for production debugging. Instead, return diagnostic data in the payload:

```javascript
return {
  dashboardUri: '...',
  _debug: {
    inputKeys: Object.keys($),
    themeMode: $.themeMode,
    edgeCount: $.automatedGuidedVehicle?.edges?.length ?? 0
  }
};
```

When done debugging, remove the `_debug` block.

### Markdown elements support HTML — but no JavaScript

Fuuz markdown rendering allows inline HTML with CSS — `<div>`, `<span>`, `<details>`, `<summary>`, inline styles, etc. This is enough for card-like UI with hover effects, status pills, and even collapsible sections.

What markdown does NOT support is JavaScript. No `<script>` tags. No `onclick` handlers (well, sometimes they render but don't execute reliably). For interactivity that needs JS, you need either:
- Action steps on the markdown element (click handlers wired through the screen designer, not inline)
- An embedded webpage element with the inline JS inside (and that element MUST live inside a Form, never a Card)

## Recommended approach for complex logic

1. Do as much shaping as possible in JSONata upstream (filtering, projection, simple transforms)
2. Bundle the prepared data + context into the saved script's input
3. Use the saved script ONLY for HTML/URI generation — pure rendering

This keeps the saved script small and focused, and puts the data-shaping logic where it's easiest to debug (JSONata expressions show errors at evaluation time in the designer).
