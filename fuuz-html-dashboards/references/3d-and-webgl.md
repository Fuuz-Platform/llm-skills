# 3D and WebGL in a Fuuz screen

**WebGL works.** Verified in a Fuuz screen's embedded page with hardware acceleration — not
a software fallback. Two production dashboards ship on it: the Badger meter line (a 3D plant
floor) and the Manifold knowledge graph (a force-directed graph of a whole tenant).

## The library

**three.js r128 UMD.** Stay on r128 — later releases moved to ES modules, which the inline
`<script>` pattern cannot load without a bundler.

```js
var renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 2));   // cap at 2; retina x3 is wasted
renderer.setSize(W, H);
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;
```

Cap the pixel ratio. An animating WebGL canvas is not free, and a wallboard left running on a
weak floor PC is exactly where that bill arrives.

**three.js needs numbers, not CSS variables.** Resolve your theme colours to integers before
handing them to a material — `getComputedStyle(el).getPropertyValue('--x')` returns a string
the renderer cannot use.

## Models: compress before you inline

Raw GLB from a scan, Tripo or a CAD export is tens of megabytes. Draco compression makes it
inline-embeddable. Measured on the Badger machine library:

| Model | Raw | Draco | Reduction |
|---|--:|--:|--:|
| CNC machine | 47.4 MB | **262 KB** | 181× |
| Extrusion machine | 54.7 MB | **321 KB** | 170× |
| Hydraulic press | 62.9 MB | **372 KB** | 169× |
| Chemical reactor | 57.4 MB | **363 KB** | 158× |

At 262–372 KB a model embeds directly in the page: **no model hosting, no CORS, no CDN at
runtime.** Use the compression kit in the monorepo (`3d-model-compression-kit`), and keep the
raw sources out of git — only the compressed derivatives are committed.

## Do not load libraries from a CDN at runtime

The older guidance in this skill suggested `<script src="https://cdnjs…">` to keep the data URI
small. Both production dashboards **fetch the libraries at build time and inline them** instead.
A CDN adds a runtime dependency, a network round trip on every render, and a failure mode on a
floor network that may not have egress. Fetch once into a `lib/` directory, inline, and solve
the size problem with compression — see `references/large-payloads.md`.
