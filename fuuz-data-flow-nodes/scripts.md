# Scripts

Nodes for running transformation scripts against the workflow state payload.

## State Bindings in Expressions

Both JSONata and JavaScript expressions share the same state bindings. The full workflow state is accessible via `$state`:

| Binding | Description |
|---------|-------------|
| `$` | In JSONata, the contextual focus — at the root of a script this is the input payload, but inside a nested path (e.g., `$.items.( $ )`) it shifts to the current item. In JavaScript, `$` is always the input payload. |
| `$$` | The root input payload — always refers to the top-level input regardless of nesting depth. Use this to escape back to the root from within a nested JSONata scope. |
| `$state.context` | Persistent key-value store set by Set Context / Merge Context nodes |
| `$state.claims` | Authentication claims from the initiating user or system |
| `$state.lastError` | Error details when routed through a Try Catch handler (`message`, `stack`, `node`) |

Context values persist across nodes once set (via Set Context or Merge Context) and remain available to all downstream nodes unless explicitly removed by a Remove From Context node.

For detailed coverage of JSONata and JavaScript expression syntax, scoping rules, and available functions, see the **fuuz-expressions** skill.

---

## JSONata (Transform)

| | |
|---|---|
| **Name** | `transform` |
| **Title** | JSONata |
| **Responsibility** | transition |
| **Description** | Runs a script using the JSONata language on the workflow state. This is the primary transformation node for most data flow logic. |

### Properties

| Property | Type | Format | Required | Description |
|----------|------|--------|----------|-------------|
| `transform` | string | jsonata | Yes | A JSONata script to run on the workflow state. |

Standard output port.

**Validation:** minimumNoteLength: 20, requireChangedName, requireInputNode, requireWalkthrough

### Example Configuration

```json
{
  "transform": "{ \"fullName\": firstName & ' ' & lastName, \"total\": $sum(items.price) }"
}
```

---

## JavaScript

| | |
|---|---|
| **Name** | `javascriptTransform` |
| **Title** | JavaScript |
| **Responsibility** | transition |
| **Description** | Runs a script using the JavaScript language on the workflow state. |

### Properties

| Property | Type | Format | Required | Description |
|----------|------|--------|----------|-------------|
| `transform` | string | javascript | Yes | A JavaScript script to run on the workflow state. |

Standard output port.

### Example Configuration

```json
{
  "transform": "const items = $$.items.map(i => ({ ...i, total: i.qty * i.price })); return items;"
}
```

---

## Saved Script

| | |
|---|---|
| **Name** | `savedTransformV2` |
| **Title** | Saved Script |
| **Responsibility** | transition |
| **Description** | Transforms the state payload using a saved script. The script can be JSONata or another supported language. |

### Properties

| Property | Type | Format | Required | Description |
|----------|------|--------|----------|-------------|
| `transformScriptLanguage` | string | -- | Yes | The language used by the transform. Default: `"JSONata"`. |
| `transformId` | string | -- | Yes | The saved script to run. |
| `requestTransform` | string | jsonata | Yes | Transform to populate the script request. Default: `"$"`. |
| `responseTransform` | string | jsonata | Yes | Transform to reformat the script output. Default: `"$"`. |
| `enableAdvancedConfiguration` | boolean | -- | Yes | When true, Input, Output, and Advanced Configuration transforms become configurable. Default: `false`. |

Standard output port.

---

## Data Mapping

| | |
|---|---|
| **Name** | `dataMapping` |
| **Title** | Data Mapping |
| **Responsibility** | transition |
| **Description** | Transforms the state payload using a saved data mapping. |

### Properties

| Property | Type | Format | Required | Description |
|----------|------|--------|----------|-------------|
| `dataMappingId` | string | -- | Yes | The data mapping to run on the payload. |
| `transformScriptLanguage` | string | -- | Yes | The language used by the transform. Default: `"JSONata"`. |
| `requestTransform` | string | jsonata | Yes | Transform to populate the data mapping request. Default: `"$"`. |
| `responseTransform` | string | jsonata | Yes | Transform to reformat the data mapping output. Default: `"$"`. |
| `enableAdvancedConfiguration` | boolean | -- | Yes | Enables configuration of the Script Language and Request/Response transforms. Default: `false`. |

Standard output port.

---

## Object Diff

| | |
|---|---|
| **Name** | `objectDiff` |
| **Title** | Object Diff |
| **Responsibility** | transition |
| **Description** | Compares two objects and returns the differences. |

### Properties

| Property | Type | Format | Required | Description |
|----------|------|--------|----------|-------------|
| `inputTransform` | string | jsonata | No | Transform to reformat the input payload. Default: `"$"`. |
| `originalObj` | string | jsonata | Yes | The first comparison object. |
| `updatedObj` | string | jsonata | Yes | The second comparison object. |
| `ignore` | array | jsonata | No | Array of key paths to exclude from comparison. |
| `detailed` | boolean | -- | Yes | Shows breakdown of added, removed, and changed properties. Default: `true`. |
| `outputTransform` | string | jsonata | No | Transform to format the output. Default: `"$"`. |
| `enableAdvancedConfiguration` | boolean | -- | Yes | When true, Input/Output transforms become configurable. Default: `false`. |

Standard output port.

### Example Configuration

```json
{
  "originalObj": "$state.context.original",
  "updatedObj": "$",
  "ignore": ["updatedAt", "version"],
  "detailed": true
}
```

---

## Saved Script (Legacy) -- *Deprecated*

| | |
|---|---|
| **Name** | `savedTransform` |
| **Title** | Saved Script (Legacy) |
| **Responsibility** | transition |
| **Description** | Legacy version of the Saved Script node. Deprecated: use the current Saved Script node (`savedTransformV2`) from the toolbox. |

### Properties

| Property | Type | Format | Required | Description |
|----------|------|--------|----------|-------------|
| `transformId` | string | -- | Yes | The saved script to run. |

Standard output port.
