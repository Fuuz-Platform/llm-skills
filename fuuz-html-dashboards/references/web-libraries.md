# Web Libraries for Fuuz Screens & Dashboards — Reference Guide

A vetted catalog of open-source web libraries that fit the Fuuz dashboard pattern
(backend flow → data URI → embedded webpage element, OR direct screen embedding
via Markdown/Webpage elements with CDN-loaded scripts).

## Selection criteria

Every library on this list meets all five:

1. **Permissive open-source license** — MIT, BSD, or Apache 2.0. No GPL/AGPL,
   no commercial-only, no "freemium with attribution" gotchas.
2. **Available on cdnjs.cloudflare.com or jsDelivr** — both reachable from the
   client browser when the iframe HTML loads. No npm-build-step required.
3. **Self-hostable when needed** — for restricted plant networks we can inline
   or self-host. Anything that phones home is excluded.
4. **No mandatory API key, no telemetry by default** — must work fully offline
   after first load with no external service dependency.
5. **Production-grade maturity** — not abandoned, not pre-1.0, not solo-dev
   with last commit in 2022.

## How to read this guide

Each entry has:
- **What it does** — one-line purpose
- **CDN** — exact script tag to paste, current as of 2026
- **Bundle size** — relevant when inlining for restricted networks
- **Fuuz use cases** — concrete screens or dashboards where this earns its keep
- **Tradeoffs** — when NOT to use it

---

# Category 1 — Charting & Time-Series

These cover 80% of dashboard needs: KPI cards, trend lines, distribution
histograms, comparison bars. Pick **one primary** for the suite; don't mix
unless a specific niche requires it.

## Apache ECharts ⭐ Top recommendation for new dashboards

**What it does:** Full-spectrum charting library — bar, line, area, scatter,
pie, gauge, radar, sankey, treemap, sunburst, heatmap, candlestick, parallel
coordinates, graph/network, geo maps. Canvas-backed by default, with optional
SVG renderer. Built-in animations, theming, data downsampling for large series.

**CDN:**
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/echarts/5.5.1/echarts.min.js"></script>
```

**Bundle size:** ~1 MB minified (~250 KB gzipped). Tree-shakeable down to
~100 KB for specific chart types if you build a custom bundle.

**License:** Apache 2.0

**Fuuz use cases:**
- **OEE trend dashboard** — line chart of Availability/Performance/Quality over
  shift, with gauge for current overall OEE. Gauge series is purpose-built for
  the "speedometer" pattern operators expect.
- **Scrap reason Pareto** — bar + cumulative line on dual axes, sorted DESC.
- **Production by area treemap** — instantly shows which areas contributed
  most to the week's output, sized by quantity.
- **State category sunburst** — outer ring shows individual states (Running,
  Held, Setup, etc.), inner ring rolls up to PackML category. Click-through
  drill on the screen.
- **Throughput sankey** — material flow from incoming materials → workunits →
  outputs, with band thickness = quantity. Visualizes co/by-product splits.
- **Workunit constraint heatmap** — y-axis = workunit, x-axis = hour, color =
  `currentConstraintScore`. Spot temporal patterns (every shift change?
  every Monday morning?).

**Tradeoffs:** Bundle is larger than Chart.js. For a screen that needs ONE
simple bar chart, Chart.js is lighter. For a dashboard with five or more
chart types, ECharts wins on consistency and total bundle size.

---

## Chart.js — Lightest option for simple charts

**What it does:** Canvas-based charting with 8 core types (line, bar, radar,
doughnut/pie, polar, scatter, bubble, area). Strong defaults, minimal config
required for good-looking output.

**CDN:**
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.7/chart.umd.min.js"></script>
```

**Bundle size:** ~205 KB minified, ~70 KB gzipped. Tree-shakeable to ~14 KB
for basic bar/line only.

**License:** MIT

**Fuuz use cases:**
- **Embedded KPI card sparklines** — tiny trend lines inline within Markdown
  elements showing the last 24 hours of a metric.
- **Single-purpose dashboards** — a screen that exists to show ONE chart
  (e.g., "Today's Hourly Output") — no reason to load ECharts.
- **Operator-facing simple visuals** — when the audience isn't analysts and
  doesn't need treemaps or sankeys.

**Tradeoffs:** No gauge series (only doughnut). No sankey, no treemap, no
sunburst. If you might need any of those later, start with ECharts.

---

## D3.js — Custom visualizations only

**What it does:** Not a chart library — a low-level DOM/SVG/canvas manipulation
toolkit with data binding, scales, axes, transitions, geo projections, force
simulations. Used as the foundation for many other libraries.

**CDN:**
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/d3/7.9.0/d3.min.js"></script>
```

**Bundle size:** ~270 KB minified.

**License:** ISC (BSD-equivalent)

**Fuuz use cases:**
- **Bespoke visualizations** that no charting library covers — e.g., a
  process-flow diagram with custom vessel shapes laid out by an algorithm,
  or a circular work-order schedule clock visualization.
- **Geographic projections** — if we ever need to show a multi-plant rollup
  on a real-world map without the full overhead of Leaflet.
- **Force-directed layouts** for material flow networks (use Cytoscape
  instead if you want the graph theory algorithms too).

**Tradeoffs:** Steep learning curve. Every chart is hand-rolled. Use only
when the higher-level libraries genuinely don't have what you need. For most
business charts, ECharts gets you there in 1/10 the code.

---

## ApexCharts — Strong middle ground

**What it does:** Modern, animation-rich charting with excellent default
aesthetics. SVG-based (good for small datasets, accessible, exportable).
Annotation support is best-in-class — overlay text, lines, shaded regions on
any chart natively, which is unusual.

**CDN:**
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/apexcharts/3.54.1/apexcharts.min.js"></script>
```

**Bundle size:** ~450 KB minified.

**License:** MIT

**Fuuz use cases:**
- **Charts that need annotated thresholds** — control limits on an SPC chart,
  target lines on production rate, "shift change" vertical markers on a
  24-hour timeline. ApexCharts does this with one config option.
- **Customer-facing reports** — defaults look more polished than Chart.js
  without theming work.

**Tradeoffs:** SVG-based means worse performance with >5k data points.
For high-frequency manufacturing telemetry, ECharts canvas wins.

---

## FusionCharts — Already in use; keep for Gantt

Already established in your stack (Campaign Management Gantt). Keep using for:
- **Gantt charts** specifically — best-in-class for production schedule views
- Anything where you've already built FusionCharts skill knowledge

Don't extend its footprint to other chart types when ECharts/Chart.js can do
the job for free.

---

# Category 2 — Network/Graph Visualization

For when you need to show relationships, dependencies, material flow paths,
or hierarchies that don't fit a tree.

## Cytoscape.js ⭐ Top recommendation for graph viz

**What it does:** Network/graph visualization with built-in graph theory
algorithms (PageRank, betweenness centrality, shortest path, BFS/DFS).
Multiple layout algorithms (force-directed, breadthfirst, concentric, grid,
dagre extension for hierarchical). MIT license, used in academic
bioinformatics — battle-tested in production.

**CDN:**
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/cytoscape/3.30.2/cytoscape.min.js"></script>
```

**Bundle size:** ~370 KB minified.

**License:** MIT

**Fuuz use cases:**
- **Product strategy process visualization** — show the
  `ProductStrategyProcess → ProductStrategyProcessStep` graph as a DAG
  (directed acyclic graph). Operators see WHERE in the recipe a step lives.
- **BOM explosion tree** — multi-level "what goes into what" with collapsible
  branches. Tree layout for clarity, click-to-expand on demand.
- **Material genealogy / traceability** — given a finished lot, show every
  input lot that contributed, recursing through `productionHistory` → input
  lots → their production histories. Each node clickable to drill in.
  This is direct alignment with your traceability tool work-in-progress.
- **Workunit succession network** — show which workunits feed which other
  workunits based on `productStrategyProcessStep` ordering. Material flow
  becomes literal arrows on a map.
- **Quality defect cause-and-effect (Ishikawa)** — when you need an actual
  fishbone diagram from defect/cause data rather than a hand-drawn image.

**Tradeoffs:** Performance ceiling around 5,000–10,000 nodes. For larger
graphs use Sigma.js with WebGL. For most manufacturing networks (a few
hundred nodes max), Cytoscape is the right call.

---

## Sigma.js + Graphology — Large graphs only

**What it does:** WebGL-rendered graph visualization. Pairs with the
`graphology` library which holds the graph data and provides algorithms.

**CDN:**
```html
<script src="https://cdn.jsdelivr.net/npm/graphology@0.25.4/dist/graphology.umd.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/sigma@3.0.0/dist/sigma.min.js"></script>
```

**Bundle size:** ~150 KB combined.

**License:** MIT

**Fuuz use cases:**
- **Enterprise-wide supplier/customer network** — only if you have thousands
  of entities to visualize at once. For sub-1000-node graphs Cytoscape is
  easier to work with.
- **Real-time live network of equipment connectivity** — if we ever expose
  OPC UA topology, Sigma's WebGL backbone handles it without frame drops.

**Tradeoffs:** Lower-level API than Cytoscape. No built-in graph algorithms
without graphology extensions. Use only when you've hit a real perf
bottleneck.

---

## vis.js Network — Built-in physics, easiest API

**What it does:** Network graph with built-in physics simulation, drag-and-drop
node positioning, hierarchical layout, clustering. Less feature-rich than
Cytoscape but quicker to first render.

**CDN:**
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/vis-network/9.1.9/standalone/umd/vis-network.min.js"></script>
```

**Bundle size:** ~500 KB minified.

**License:** Apache 2.0

**Fuuz use cases:**
- **Operator-facing process flow** where physics-based "settling" is desired
  — nodes spring into place rather than appearing on a rigid grid.
- **Quick prototypes** of network views before deciding if Cytoscape is needed.

**Tradeoffs:** Heavier than Cytoscape, less algorithmic depth. For Fuuz use
cases, Cytoscape covers the same ground better — use vis.js only if a stake-
holder specifically asks for physics-based layout.

---

# Category 3 — 3D / WebGL

## Three.js ⭐ Already proven in your stack

**What it does:** The foundational 3D library — meshes, materials, lighting,
shadows, cameras, animation, particle systems. Already used in your WMS
wallboard and now in the process plant bottleneck visualizer.

**CDN:**
```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
```

(Note: r128 is your established baseline. r170+ has breaking API changes
around materials and modular imports — stay on r128 unless we have a reason
to migrate.)

**Bundle size:** ~600 KB minified.

**License:** MIT

**Fuuz use cases:**
- **3D bottleneck visualizer** (delivered)
- **3D AGV warehouse wallboard** (delivered)
- **3D campaign/order spatial timeline** — orders as floating cards arranged
  along a 3D axis (time × workunit × priority). Wallboard candidate.
- **Equipment digital twin previews** — when you want to show a 3D model of
  the actual reactor/dryer/line rather than an abstract vessel.
- **Plant overview rotation** — multi-area facility rendered as a 3D campus.

**Tradeoffs:** Stay on r128 for our patterns. Don't load r170 — too many
breaking changes to be worth the risk on operator-facing screens.

---

## Babylon.js — Don't bother (for our use case)

5× the bundle size of Three.js (1.4 MB), more game-engine-oriented features
we don't need (physics, audio, XR). Skip unless we ever build a true
interactive operator training simulator.

---

# Category 4 — Mapping / Geography / Floor Plans

## Leaflet ⭐ Top recommendation for any map-shaped problem

**What it does:** Lightweight interactive map library. Works with traditional
geographic tiles AND arbitrary coordinate systems — meaning you can use it as
a "factory floor plan viewer" where the "map" is a PNG/SVG floor plan and
the "markers" are workunit positions.

**CDN:**
```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.min.css"/>
<script src="https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.min.js"></script>
```

**Bundle size:** ~150 KB JS + ~15 KB CSS.

**License:** BSD-2

**Fuuz use cases:**
- **Plant floor plan overlay** — load a facility floor plan as a tile or
  image layer, then overlay workunit markers with `currentConstraintScore`
  driving the marker color. Operators see exactly where on the floor the
  problem is. This is the 2D version of the 3D bottleneck viz — cheaper to
  build, no WebGL required, and works on lower-end terminals.
- **Multi-site rollup** — true geographic map showing every plant in the
  enterprise, with rollup KPIs in popups.
- **Logistics view** — inbound/outbound truck dock status, AGV positions on
  a floor plan, material movements between buildings.
- **Indoor maps with floor switching** — use `leaflet-indoor` extension to
  add a floor selector when buildings have multiple levels (mezzanine,
  basement storage, second-floor offices).

**Tradeoffs:** Vector tiles look better with MapLibre. For purely indoor /
floor plan work without true geography, Leaflet is simpler.

---

## MapLibre GL JS — Vector tiles, advanced styling

**What it does:** WebGL-rendered vector tile maps. Fork of the open-source
Mapbox GL JS. Supports 3D extrusions (`fill-extrude-height`) for things like
building footprints with height-encoded data.

**CDN:**
```html
<link href="https://cdn.jsdelivr.net/npm/maplibre-gl@4.7.1/dist/maplibre-gl.min.css" rel="stylesheet"/>
<script src="https://cdn.jsdelivr.net/npm/maplibre-gl@4.7.1/dist/maplibre-gl.min.js"></script>
```

**Bundle size:** ~800 KB minified.

**License:** BSD-3

**Fuuz use cases:**
- **3D extruded plant heatmap** — buildings/areas extruded in height by
  bottleneck score. Visually striking for executive wallboards.
- **Vector indoor map** with crisp text and smooth zoom — when Leaflet's
  raster approach starts looking dated at large monitors.

**Tradeoffs:** Heavier than Leaflet, vector tile prep can be a project of
its own. Use only when the project specifically calls for it.

---

# Category 5 — Data Grids / Tables

## Tabulator ⭐ For when Fuuz's native table isn't enough

**What it does:** Feature-rich data grid with sorting, filtering, grouping,
inline editing, frozen columns, virtual scrolling for large datasets,
multiple export formats (CSV, JSON, PDF, XLSX).

**CDN:**
```html
<link href="https://cdnjs.cloudflare.com/ajax/libs/tabulator/6.3.0/css/tabulator.min.css" rel="stylesheet"/>
<script src="https://cdnjs.cloudflare.com/ajax/libs/tabulator/6.3.0/js/tabulator.min.js"></script>
```

**Bundle size:** ~440 KB minified.

**License:** MIT

**Fuuz use cases:**
- **Heavyweight historical lookup screens** — production history queries that
  return 5k–50k rows. Native Fuuz tables degrade past a few thousand; Tabulator
  virtual-scrolls smoothly into the tens of thousands.
- **Excel-like operator entry screens** — when an operator needs to enter
  data across many cells quickly (test results, bulk lot adjustments).
- **Inline-editable BOM management** — adjustable quantity/UOM per row with
  immediate validation.

**Tradeoffs:** For most screens, the native Fuuz table is the right call —
it's already integrated with the platform's data binding, action steps, and
permission model. Only reach for Tabulator when you need a capability the
native table doesn't support and the cost of building it into a custom
HTML screen is justified.

---

## AG Grid Community — Avoid unless you have a known reason

Free tier is capable but the licensing model has caused friction in
enterprise deployments. Tabulator covers the same ground under pure MIT.

---

# Category 6 — In-Browser AI / ML

## Transformers.js ⭐ Browser-side AI without server cost

**What it does:** Hugging Face's library for running pre-trained AI models
directly in the browser via ONNX Runtime. Text classification, embeddings,
NER, summarization, translation, even small LLMs. Models cache locally after
first load — true offline operation.

**CDN:**
```html
<script type="module">
  import { pipeline } from "https://cdn.jsdelivr.net/npm/@huggingface/transformers@3.0.2";
</script>
```

**Bundle size:** ~250 KB for the library. Models range from ~25 MB (small
classifiers) to ~1 GB (small LLMs). Models load on first use and cache.

**License:** Apache 2.0

**Fuuz use cases:**
- **Operator-entered text classification** — free-text downtime reasons get
  automatically categorized into your taxonomy (mechanical / electrical /
  material / operator-error). Zero round-trip to server.
- **Semantic search across BOMs, recipes, or knowledge base** — embed your
  catalog client-side, query by meaning rather than exact text match. Solves
  the "I don't remember the product code but I know what it is" problem.
- **Anomaly detection on telemetry** — small classifier model trained on
  normal cycle patterns, flags abnormal ones inline without server calls.
- **Quick-text-summarization for shift handover notes** — bullet-point the
  long-form text that operators dump into shift logs.

**Tradeoffs:** First-load model download is significant (10 MB to 1 GB). Not
a fit for one-shot screens — best for "always-on" wallboards that stay open
for hours. Inference on main thread blocks UI — use a Web Worker for any
serious work. For complex reasoning, the Anthropic API call (which you
already have via the Campaign AI Insight feature) is still the right path.

---

## TensorFlow.js — Custom model training/inference

**What it does:** Run TensorFlow models in the browser. Lower-level than
Transformers.js — bring your own model, manage your own pipeline.

**CDN:**
```html
<script src="https://cdn.jsdelivr.net/npm/@tensorflow/tfjs@4.22.0/dist/tf.min.js"></script>
```

**Bundle size:** ~1 MB.

**License:** Apache 2.0

**Fuuz use cases:**
- **Custom vision quality checks** — if QA cameras feed images into the
  browser, you can run a defect-classifier model client-side. Specific niche.
- **Custom forecasting models** — trained externally, exported to TF.js
  format, deployed inline for "what will throughput look like in 4 hours"
  predictions.

**Tradeoffs:** Higher complexity than Transformers.js. Only worth it if
you're bringing custom-trained models. For off-the-shelf NLP/vision tasks
Transformers.js is easier.

---

## ONNX Runtime Web — Pure inference engine

Lower-level than Transformers.js (which wraps it). Use directly only when
running models that don't have a Transformers.js pipeline. Niche.

---

# Category 7 — Diagrams & Flowcharts (Text-Driven)

## Mermaid — Diagrams from text

**What it does:** Render flowcharts, sequence diagrams, Gantt charts, state
diagrams, ER diagrams from plain-text definitions. Zero canvas/SVG knowledge
required from the author.

**CDN:**
```html
<script src="https://cdn.jsdelivr.net/npm/mermaid@11.4.0/dist/mermaid.min.js"></script>
```

**Bundle size:** ~2 MB minified (large because it bundles many parsers).

**License:** MIT

**Fuuz use cases:**
- **Generated process flow documentation** — given a `ProductStrategyProcess`
  + its steps, output a Mermaid diagram string in a screen Markdown element
  and have Mermaid render it. Documentation that always reflects current
  data.
- **State machine visualization** — render PackML states + transitions for
  training or operator reference, generated from the actual state list
  configuration of the workunit.
- **Generated SOP diagrams** — workflow steps rendered from data, so changes
  to the process show up in the visualization without manual diagram updates.

**Tradeoffs:** Large bundle. Each diagram is fully re-rendered (no
incremental updates). Best for "documentation-style" screens, not real-time
operations.

---

# Category 8 — Specialized Visualizations

## Plotly.js — Scientific plotting

**What it does:** Statistical/scientific charts with built-in interactivity
— 3D scatter, contour plots, parallel coordinates, ternary plots, violin
plots, statistical distributions.

**CDN:**
```html
<script src="https://cdn.jsdelivr.net/npm/plotly.js-dist@2.35.2/plotly.min.js"></script>
```

**Bundle size:** ~3.5 MB (heavyweight).

**License:** MIT

**Fuuz use cases:**
- **SPC charts with proper statistical overlays** — control limits,
  confidence bands, distribution underlays. The annotation flexibility
  beats ECharts for serious SPC work.
- **Multivariate quality analysis** — parallel coordinates plot showing
  every QC parameter against pass/fail outcomes.
- **3D process surface plots** — temperature × pressure × yield as a
  surface mesh, for process engineering screens.

**Tradeoffs:** Very large bundle. Use for engineer-facing analytics only,
never for operator-facing wallboards.

---

## Tone.js — Audible alerts

**What it does:** Web Audio framework for synthesizing tones programmatically.

**CDN:**
```html
<script src="https://cdn.jsdelivr.net/npm/tone@15.0.4/build/Tone.js"></script>
```

**Bundle size:** ~250 KB.

**License:** MIT

**Fuuz use cases:**
- **Audible wallboard alerts** — when a workunit transitions to a critical
  state, play a tone. Better than visual-only on noisy plant floors. Use
  sparingly — operators learn to ignore over-alerted dashboards.

**Tradeoffs:** Requires user interaction to start audio context (browser
security). Sound cues must be carefully designed not to become noise.

---

## Tippy.js / Floating UI — Tooltips & popovers

**What it does:** Lightweight tooltip/popover positioning library.

**CDN:**
```html
<script src="https://cdn.jsdelivr.net/npm/@popperjs/core@2.11.8"></script>
<script src="https://cdn.jsdelivr.net/npm/tippy.js@6.3.7"></script>
```

**Bundle size:** ~50 KB combined.

**License:** MIT

**Fuuz use cases:**
- **Rich tooltips on dashboard elements** — hover a workunit in the 3D
  visualizer (interactive mode), show full status card without modal/click.
- **Inline help indicators** on complex configuration screens.

---

# Category 9 — Date/Time Utilities (Already in use)

## Moment.js + moment-timezone — Already standard

You're already using `$moment` in the Fuuz sandbox. For client-side HTML
(post-iframe) where you need consistent date handling, the same library
loads via CDN:

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/moment.js/2.30.1/moment.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/moment-timezone/0.5.46/moment-timezone-with-data.min.js"></script>
```

For new client-side work, **Luxon** is the modern replacement (~75 KB, the
official successor) — but Moment is fine for symmetry with the server-side
patterns we already use.

---

# Decision Matrix — "I need to show X"

| Need | Use |
|------|-----|
| Bar / line / gauge / pie | ECharts (or Chart.js if it's the only chart on screen) |
| Treemap / sunburst / sankey | ECharts |
| OEE-style gauge speedometer | ECharts gauge series |
| Custom one-off visualization | D3 |
| Network / dependency graph | Cytoscape.js |
| Material genealogy tree | Cytoscape.js (breadth-first layout) |
| 3D anything | Three.js (stay on r128) |
| Factory floor plan with markers | Leaflet (with floor plan as image overlay) |
| Multi-plant geographic rollup | Leaflet (with OSM tiles) |
| 10k+ row data tables | Tabulator |
| Spreadsheet-like editing | Tabulator (or stay with Fuuz native if simple) |
| Text classification / NLP / embeddings | Transformers.js |
| Custom-trained ML inference | TensorFlow.js |
| Flowchart from data | Mermaid |
| SPC / statistical plots | Plotly.js |
| Tooltips on custom visuals | Tippy.js |
| Audible alerts | Tone.js |
| Gantt schedules | FusionCharts (already in your stack) |

---

# Anti-Patterns to Avoid

1. **Don't load multiple chart libraries on the same screen.** Pick ECharts
   OR Chart.js OR Plotly — not all three. Bundle bloat hits operator
   terminals hard.
2. **Don't use commercial libraries (Highcharts, Syncfusion, AG Grid
   Enterprise, Tom Sawyer) unless explicitly licensed.** Some have
   "free for non-commercial" terms that don't cover manufacturing
   deployments.
3. **Don't load libraries from npm CDNs the customer's network might
   block.** Stick to cdnjs.cloudflare.com (most-allowlisted CDN) and have
   a self-host fallback ready for restricted plants.
4. **Don't pull entire UI frameworks (Material UI, Ant Design) into a Fuuz
   data-URI screen.** They're React-bound and assume a build step. The Fuuz
   webpage element runs vanilla HTML — frameworks add hundreds of KB for
   features you don't use.
5. **Don't put real-time WebSocket libraries (Socket.IO, Pusher) in the
   data-URI HTML.** The iframe lifecycle makes persistent connections
   wasteful — let Fuuz's `pollInterval` drive refresh instead.

---

# Network considerations — plant-floor reality

Before committing to ANY CDN-loaded library in production:

1. **Ask the customer's IT** whether `cdnjs.cloudflare.com` and
   `cdn.jsdelivr.net` are reachable from the plant network.
2. **Have a self-host fallback ready.** For restricted plants, the libraries
   in this list can all be inlined or hosted on the customer's web server.
3. **Subresource Integrity (SRI) for CDN scripts** — for security-conscious
   customers, add `integrity="sha384-..." crossorigin="anonymous"` to the
   script tag. cdnjs provides SRI hashes on every version page.
4. **Bundle size budget per dashboard** — aim for under 1 MB total of
   loaded JS/CSS. Above that, refresh perceived latency suffers,
   especially on plant-floor terminals with marginal hardware.

---

# Suggested first-build picks for the next round of screens

Based on what you've already shipped and what's logical next:

1. **OEE / Operational Efficiency dashboard** → ECharts (gauge + multi-series
   line). High-frequency refresh, theme-aware, dark wallboard friendly.
2. **Material genealogy / traceability tool** → Cytoscape.js with
   breadth-first layout. Each input lot is a node, parent-child via
   `productionHistory` back-references. Click drills into the upstream
   production run.
3. **Process strategy visual editor (read-only first)** → Cytoscape.js with
   `dagre` extension for top-down DAG layout of ProductStrategyProcess →
   Steps → required EquipmentUnits.
4. **Plant floor 2D map** → Leaflet with the customer's actual floor plan
   image as the base layer. Workunit positions stored as `customData` on
   workunit, marker color = `flowClassification`. Lower-cost alternative
   to the 3D viz for sites that don't want 3D.
5. **Free-text downtime reason auto-categorization** → Transformers.js with
   a small text-classification pipeline. Reduces operator data-entry
   friction.

Each one slots into the same backend-flow → data-URI → webpage element
pattern you already have working.
