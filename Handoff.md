# Handoff — MotionKit

Single-file Three.js animation studio. HTML + CSS + every animation + every
page lives in `index.html` (~14k lines, ~550 KB). No build step, no
`node_modules`, no bundler. Drop into any browser, or serve locally.

Last checkpoint: **2026-05-26**. Repo: <https://github.com/nick-a8c/motionkit>
(deployed at <https://nick-a8c.github.io/motionkit/>).

### 2026-05-26 session — Visu collection + opentype SVG + font polish

Four new Playground tools, real-vector SVG export upgrade, and a long
tail of font / fit / UX polish.

**Four new Playground tools** (cards now read top-to-bottom: Grid →
Concentric → Word Snake → Letter Warp → Composer → Glyph → Vinyl → Waves →
Dots → Particles → Halftone — 11 tools total). Each one is a self-contained
2D-canvas effect ported from a visu.haus-style handoff folder, wrapped as a
MotionKit animation via a `THREE.CanvasTexture` on a fullscreen plane:

| Tool          | Section | What it does |
|---|---|---|
| **Letter Warp** | 3.6 | Tiled letterform with sine-wave offsets + Gaussian cursor amplitude bump |
| **Word Snake**  | 3.7 | Phrase walks an Archimedean spiral with arc-length-correct spacing |
| **Grid**        | 3.8 | Word tiles rows × cols with cells expanding around a 2D Gaussian peak |
| **Concentric**  | 3.9 | Nested ring outlines of a word via `destination-out` compositing |

Grid is the **default** tool on first load (used to be Composer); the
bootstrap falls back to Composer if Grid is removed.

**Gallery thumbnails now mount eagerly.** `ThumbManager.register` schedules
a `mount + renderOne` via `setTimeout 0` so every card shows a static
preview at load — you don't have to hover to reveal them. Ticking (live
animation) still only happens while hovering, so idle CPU is unchanged.

**Pulse effect on every Playground tool except Vinyl.** Composer already
had it via its `SEGMENTS.effects` registry; Glyph picked it up by adding
three missing effect flags to the composer-style picker; Waves / Dots /
Particles / Halftone got a per-tool `Pulse` controls section + per-frame
radial-Gaussian displacement (shader-side for Dots/Halftone, anchor-side
for Waves, particle-velocity-side for Particles). Vinyl is raster-baked so
the canvas effect doesn't fit.

**Live-aspect canvases play nicely with all new tools.** Each new tool's
update() resizes its offscreen canvas to match `window.MotionPlaygroundView`
when the dropdown switches to 16:9, so glyphs aren't stretched on aspect
changes. Particles' field-canvas was also widened (was hardcoded square
FIELD_SIZE; now scales with aspect).

**opentype.js wired into the SVG export.** Loaded async from CDN; a new
`window.MotionFontTrace` global owns the font URL map (fontsource TTFs for
Inter, Anton, Bebas Neue, Oswald, Roboto Condensed), a per-font cache, and
a `glyphPath(font, ch, fs)` helper that returns the recentred true cubic
Béziers as a `d`-string + bbox. Grid's `update()` pre-fetches the active
font; its `toSvg` emits real opentype outlines when available, falling
back to a marching-squares trace + midpoint-Bézier smooth for system
fonts. Concentric's letters are wired the same way.

**Per-tool SVG export quality matrix** (after the upgrade):

| Tool        | SVG output | Notes |
|---|---|---|
| Composer    | Real polylines | Per-Line2 thickness preserved |
| Glyph       | Real polylines + polygons | Same as Composer |
| Vinyl       | Base64 image | Compositing-based, no vector |
| Waves       | Real polylines | One per row |
| Dots        | Real circles / rects / rings | Sampled from mask canvas |
| Particles   | Real circles / polygons | InstancedMesh matrix walk |
| Halftone    | Mixed circles / rings / images | Library mode embeds raster |
| **Letter Warp** | `<defs>` source `<text>` + ~1830 `<clipPath>` + `<use>` | True vector for the warp tile grid |
| **Word Snake** | Real per-letter `<text>` along spiral | Editable type in Illustrator/Figma |
| **Grid**    | opentype outlines (`<path>`) for Inter/Anton/Bebas Neue/Oswald/Roboto Condensed | Falls back to marching-squares smoothed trace for system fonts |
| **Concentric** | **No SVG export** — button greys out | See below |

**Concentric SVG was removed after extensive trial.** First attempt
emitted stroked `<text>` outlines — only one ring per letter. Second tried
`<filter><feMorphology>` to dilate the letter silhouette per ring — works
in browsers, but Figma silently strips filters. Third tried raster
distance-transform + marching-squares-trace per ring — produced
~3 MB files that still weren't great. Decided to drop the export. The
toolbar SVG button greys out whenever the active tool's `toSvg` is
missing OR its optional `canExportSvg(params)` returns false.

**`canExportSvg(params)` hook** lets any tool conditionally enable the
SVG button. Grid uses it to return true only when the active font has an
opentype URL — system fonts (Helvetica, Arial, Georgia, Times, Courier,
Impact, Verdana, Tahoma) make the button grey, since they'd fall back to
the marching-squares trace which doesn't reach type-quality smoothness.
Inter / Anton / Bebas Neue / Oswald / Roboto Condensed keep the button
enabled. The greying machinery is centralised in `refreshExportSvgButton()`
called from `selectAnimation`, every choice-control click handler, and
the main loop tick (no-op when state unchanged).

**Font controls expanded**:
- **Lab font dropdown** got a second "Weight" dropdown next to it. Each
  font carries a `weights: [{label, weight}]` array; default chosen via
  `defaultWeightFor(entry)` (prefers 700, else heaviest). Rasterisers
  read `recipe.subject.fontWeight` instead of hardcoding 900. The Google
  Fonts `<link>` was expanded to load 400/500/700/900 for the multi-weight
  faces; single-weight display faces (Bebas Neue, Anton, Archivo Black,
  DM Serif Display) show a one-option disabled dropdown.
- **Lab font dropdown** styled as a rounded white pill with a custom SVG
  chevron (CSS `--stage-aspect` style siblings live nearby). The Knockout
  font + embedded base64 face that was briefly added is gone.
- **Font load race fixed**: changing fonts re-renders once
  `document.fonts.load()` resolves, so the first selection doesn't measure
  with the fallback face (the old bug: click twice to apply).

**Per-font Vertical offset on Concentric AND Grid.** Different fonts have
different cap-height vs. ascender ratios, so the same baseline math
overflows the cell/canvas for tall-ascent fonts. Both tools got a
`Vertical offset` slider (−0.5 to +0.5 in unit-height) and Grid also got
a `Character size` slider (0.2× to 2×) that multiplies the auto-fit
sx/sy. Applied identically on canvas and SVG export so what you see
matches what you export.

**Defaults updated per user spec** (`INITIAL_DEFAULTS`):
- Letter Warp: text `Queen`, vertical offset −0.03, bg `#5B77FF`, letter `#F2E9D8`.
- Word Snake: phrase `A8C` (max 12 chars), paper `#5B77FF`, inkInner + inkOuter `#F2E9D8`.
- Grid: palette `#5B77FF / #EF3B2D / #000000 / #f0ff1f / #ffffff / #a5ee8b / #2205ff`.
- Concentric: word `A8C`, bg + fontColor `#5b77ff`, ink + ringColor `#F2E9D8`.

**Two TDZ hotfixes** at the end of the session unblocked the controls
panel from going blank on first load:
- `SECTION_OPEN_STATE` was declared with `const` near `buildControls`,
  but the bootstrap's `selectAnimation` runs earlier in the IIFE and
  reached buildControls in the const's TDZ → silent throw.
- Same shape for `_lastSvgBtnOk` declared near `refreshExportSvgButton`.
Both moved to the top of the Section 4 IIFE alongside the other state
variables.

### 2026-05-20 micro-session

Three small but user-visible fixes:

- **Lab layer stacking now matches the Layers panel.** Layer index 0 (top of
  the panel) draws last so it composites in front. Implemented as
  `applyLayerRenderOrder()` called at the end of `rerender()` — sets
  `renderOrder = (n - 1 - i)` on each layer's group and traverses children.
  Reordering via ↑/↓ or drag-and-drop already calls `rerender()`, so the
  ordering follows automatically.
- **"+ Add layer" now inserts at index 0** (top of the stack) instead of
  pushing to the end. Mirrors the existing `+ Effects` behaviour. The new
  layer becomes active. Combined with the renderOrder fix above, a freshly
  added shape layer visibly sits on top.
- **Playground section collapse state persists per tool within the session.**
  A new module-level `SECTION_OPEN_STATE` map (keyed by `anim.id`) remembers
  every header toggle. `buildControls` consults it on tool re-entry; the
  per-tool `defaultOpenSections` whitelist is only applied for sections the
  user has not yet touched. Resolves the prior behaviour where switching
  Composer → Glyph → Composer reset everything back to defaults.
- **Lab HTML export — EFFECTS layers carry through.** Previously the
  exporter built renderers per `subject.type + '_' + subject.style`, which
  produced an `effects_undefined` fallback that wrote "Unsupported style:
  undefined" into `document.body` on load. Fixed by keeping EFFECTS layers in
  the bake (so per-layer indices stay aligned with the studio's cascade rule)
  but skipping them during renderer-key collection and the build/tick loops.
  In addition:
  - `LAYER_EFFECTS[i]` is now the **cumulative** motion list for layer i
    (local + every preceding non-hidden EFFECTS-layer's effects, in stack
    order), pre-computed at bake time.
  - The runtime pixel chain is aggregated at bake time from every non-hidden
    EFFECTS-layer's `pixelEffects` (in stack order) plus the legacy
    `recipe.globalPixel` array.
  - `writeParticleTransforms(0)` calls from build bodies now default
    `layerIdx` to a new module-level `currentLayerIdx` set by `activate(i)`.
  - The renderer's clear color picks the first non-effects layer's bgColor
    (mirrors studio's `firstBg` rule).
  - Pre-existing bug: `buildHersheyStrokes.toString()` references
    `recipe.subject.subjectScale`, which doesn't exist in the standalone
    runtime — added `.replace(/recipe\.subject\b/g, 'R')` to the rewrite
    chain.
  - **Still not supported in HTML export**: Solid Fill / Outline mesh
    deformation. Those renderer bodies use a textured plane (text_fill) or an
    un-subdivided ShapeGeometry (svg_fill / outline), so per-vertex
    deformation has nowhere to land. Marching-squares + subdivision + a
    runtime `writeFillTransforms` would need to be ported in.
- **Playground "Scene scale" + "Model scale".** Two distinct knobs:
  - **Scene scale** (every tool): `state.group.scale.setScalar(p.sceneScale ?? 1)`
    — resizes the entire rendered output as a unit, useful for fitting an
    exported animation to a target frame.
  - **Model scale** (Glyph / Dots / Particles / Halftone only): multiplies
    the text-rasterisation target box, so the underlying letterforms grow or
    shrink at draw time. Each tool's mask cache key now includes
    `modelScale` so the rasteriser invalidates correctly.
  Both sliders 0.1×–2×, default 1.
- **Per-Lab-layer transparent background.** Each shape layer's Background
  row in the Subject panel now has a "Transparent" checkbox alongside the
  color picker. When checked, that layer is excluded when `rerender()` picks
  the first non-effects layer to drive the canvas clear color; if every
  shape layer is transparent the canvas clears with alpha 0. The HTML
  export runtime + the export's CSS `background:` declaration mirror the
  same rule. Persists in v2 snapshots via the existing `Object.assign`
  spread on subjects.
- **Playground live aspect ratio.** The 1:1 / 16:9 dropdown now resizes the
  live canvas (not just the HTML export). `.stage-wrap`'s `aspect-ratio`
  reads from a `--stage-aspect` CSS variable; the dropdown's `change`
  handler writes it and calls `resize()` synchronously. The shared
  Playground `resize()` was rewritten to honor real width × height,
  recompute the orthographic camera's frustum (holds `VIEW` as the short
  axis so the long axis grows with aspect), and updates `perspCam.aspect`.
  A new `window.MotionPlaygroundView = { w, h, halfW, halfH, aspect }` is
  refreshed every resize so tools that need the live frustum (Dots,
  Halftone) can read it. Dots and Halftone widen their grid into the
  extra canvas area; the other five tools stay centered with empty
  flanks (intentional). `eventToWorld()` also reads the live frustum so
  mouse input maps correctly on non-square ratios.
- **Playground SVG export.** New "SVG" toolbar button. Each animation
  declares a `toSvg(state, params)` method that returns an array of SVG
  element strings already mapped into a `0 0 1024 1024` viewBox. The
  top-level handler (around line 11442) wraps them with a header + optional
  background `<rect>` (skipped when Transparent is on) and downloads as
  `{slug}.svg`. Shared serialization helpers live on
  `window.MotionSvgHelpers` (defined in Section 4): `polyline`, `polygon`,
  `circle`, `ring`, `rect`, `text`, `image`, `path`, plus coordinate
  mapping (`mapX/mapY/mapR`) and `readLineGeomPositions` for pulling flat
  positions out of `THREE.LineGeometry`'s interleaved buffer. Lab's existing
  `exportSvg` was lightly refactored to alias these helpers, so it still
  works unchanged.
  - Per-tool implementations: Composer / Glyph / Waves emit `<polyline>` per
    `THREE.Line2`; Dots samples its mask canvas and emits `<circle>` /
    `<rect>` / ring `<path>` per grid cell; Particles walks an InstancedMesh
    and emits `<circle>` or `<polygon>` per instance; Halftone walks an
    InstancedMesh and emits per-cell rotated `<polygon>` / `<circle>` /
    ring (and embeds each library-mode `CanvasTexture` as a base64
    `<image>` in library mode); Vinyl falls back to embedding its composed
    ring+text canvas as a single base64 `<image>`.
  - Overlays (Gradient / Pixelate / Halftone post-process) are dropped from
    SVG output by design — they're pixel-space and can't be vectorized.
    Use PNG to capture the composited output. Implemented as `state.group.scale.setScalar(p.modelScale ?? 1)`
  at the top of each update. To keep cursor effects coherent at non-1 scales,
  each tool's update wraps its body in a snapshot/restore that brings the
  cursor field into the model's local frame:
  ```js
  const __mouseRaw__ = state.mouse;
  if (state.mouse && __ms__ !== 1) state.mouse = { x: state.mouse.x / __ms__, y: state.mouse.y / __ms__ };
  try { /* existing update body */ } finally { state.mouse = __mouseRaw__; }
  ```
  Dots uses a `Vector2` uniform (`state.material.uniforms.cursorUv.value`)
  instead of a plain `state.mouse`; snapshot clones, restore uses `.copy()`,
  and the scaling formula is `(uv - 0.5) / ms + 0.5` since the uniform is in
  centred world-UV space. Vinyl has no cursor effect so the snapshot was
  skipped.
- **Defaults refresh** for Composer / Glyph / Waves / Dots / Particles /
  Halftone in `INITIAL_DEFAULTS` (line ~5412). Newly-mapped keys worth
  noting: Waves uses `shapeSoftness` / `cursorBoost` / `cursorFreqBoost` /
  `overlayGradientPosition`; Halftone uses `blurPx` / `cellGap` / `warpAmp` /
  `warpSpeed` / `warpScale` / `thresholdJitter` / `rotJitter`.

Earlier backups in `backups/`. The 2026-05-18 backup predates the Lab vector
fill / outline mesh, the deformed-mesh SVG export, the six new Lab effects,
the per-tool save-to-Playground flow, the export-row redesign, and the polish
pass — all listed below.

---

## Run it

```bash
cd "MotionKit"
python3 -m http.server 8765
# open http://localhost:8765
```

`.claude/launch.json` (in the `_01` workspace root) declares the same command
for the in-IDE preview, with `--directory` pointing at this folder.

---

## Top-level structure

- **Pages** (top nav): `Landing` · `Playground` · `Lab`. Landing is now the
  default on first load. `Playground` is the renamed `Create` page; the
  internal page id is still `create` to avoid breaking saved state.
- **Playground tools** (cards on the right, top-to-bottom): Grid ·
  Concentric · Word Snake · Letter Warp · Composer · Glyph · Vinyl · Waves ·
  Dots · Particles · Halftone (**11 in total**). The first four were added
  in the 2026-05-26 session; Grid is the default on first load.
- **Lab** is a separate top-level page with a fundamentally different
  architecture: layered subjects (text/SVG) + EFFECTS layers + global pixel
  post-processing. Every Lab fill / outline is now a **real triangulated
  vector mesh** that the effect pipeline deforms per-vertex.

---

## File map (`index.html`)

Every JS section is delimited by a top-level `// SECTION X.Y:` comment. The
HTML export for Playground tools relies on those exact markers to slice
sections.

| Section | What's there |
|---|---|
| `// SECTION 1:` | Shared utilities: `MotionNoise` (`hash2`, `valueNoise2D`, `loopingNoise`, `envelopePulse`, `perlin3`, `fbm3`). Exposed on `window.MotionNoise`. |
| `// SECTION 2:` | **Glyph** animation. Hershey simplex raw data + `getGlyph` + `buildTextPath`. Now also includes hidden Composer-style Primitive + Effects checkbox params; the actual picker UI is built in Section 4. |
| `// SECTION 3:` | **Vinyl** animation. |
| `// SECTION 3.4:` | **Waves** animation. SVG mode is a two-button choice (Fill / Outline). |
| `// SECTION 3.46:` | **Dots** animation. Default palette is now light gray + red + neutral gray. |
| `// SECTION 3.47:` | **Particles** animation (internal id `gradient`). |
| `// SECTION 3.48:` | **Halftone** animation. Wave effect + cursor attractor added this/prior session; cell coverage allows 0–200%. |
| `// SECTION 3.5:` | **Composer** — segment registry (sources / primitives / effects) + composer animation. Each primitive now clones its `LineMaterial` per item so the line-thickness gradient can vary `linewidth` per anchor. Three new motion effects (`orbit`, `glitch`, `pulse`) live here; Mirror/Kaleidoscope was removed earlier in the session. |
| `// SECTION 3.6:` | **Letter Warp** animation (2D canvas → CanvasTexture → fullscreen plane). |
| `// SECTION 3.7:` | **Word Snake** animation (Archimedean spiral, drag-to-rotate via accumulated angular delta). |
| `// SECTION 3.8:` | **Grid** animation. Hosts opentype.js helpers on `window.MotionFontTrace`; Concentric reuses them. Also declares `canExportSvg(p)` returning true only for fonts with `FONT_OT_URLS` entries. |
| `// SECTION 3.9:` | **Concentric** animation. Canvas-side renders normally; SVG export is intentionally absent — toolbar SVG button greys out automatically. |
| `// SECTION 4:` | Main app — renderer, page nav, categories grid, `selectAnimation`, `buildControls` (with `hidden`-flagged controls + per-tool `defaultOpenSections`), mouse interactions, exports (JSON / PNG @ 1×/2×/3× / HTML with PAN/ZOOM gating / **SVG** with `canExportSvg` greying), `buildSegmentPickers` (Composer + Glyph), shared overlay shaders (gradient / pixelate / halftone), `MotionSvgHelpers`, and the **Lab module** (~3000 lines, an IIFE near the bottom of Section 4). |

Every Playground animation IIFE ends with `window.MotionAnimations.push({...})`.

---

## Playground architecture model

Same as the previous handoff — single global `window.MotionAnimations`
registry, per-anim `ANIM_STATE[anim.id] = { params, defaults }`,
`INITIAL_DEFAULTS` patches in Section 4, recipe-driven Composer with exposed
`window.MotionSegments`, etc. New this session:

- **Shared overlay system**. Composer / Glyph / Waves / Particles get an
  appended `OVERLAY_CONTROLS` section trio (Gradient / Pixelate / Halftone)
  that drives a render-target chain wrapped around the main loop. Gradient
  mask is shape-only via `bgColor` distance.
- **PNG export** at 1×/2×/3× resolution. Renderer is `alpha: true,
  premultipliedAlpha: false` so transparent PNGs are real. Exports
  temporarily resize the drawing buffer to 2048/4096/6144 px, run the overlay
  chain, dataURL, restore.
- **HTML export** has PAN + ZOOM checkboxes that gate `mousedown`-pan and
  `wheel`-zoom listeners in the runtime template. JS-only export was removed
  from the UI (the function is still used internally by Composer's HTML
  export path).
- **Tool polish**: Composer's name defaults to "The Big O" with cursor
  strength `-0.8`; default-open sections are curated per tool (e.g., Composer
  shows only `Source — Circle` + `Cursor`, everything else collapsed). Defaults
  in `INITIAL_DEFAULTS` got a big refresh for Dots / Waves / Particles /
  Halftone — see the diff.
- **Save to Playground**. Lab's "Save to Create" button writes
  `{id, name, recipe}` to `localStorage["motionkit-saved-tools"]`. They
  render at the bottom of the Playground grid with a yellow `lab` tile;
  click → switches to Lab + `applySavedRecipe`; hover × removes.

---

## Landing page

Vertically centered hero plus a single tagline. The hero renders **211 Line2
instances** along the SVG-star outline (parsed inline at startup via
`window.MotionSegments.parseSvgAsset`). Each Line2 carries its own
`LineMaterial`; per-anchor `linewidth` thickens by up to **40%** with a
cosine-feathered 400px radius around the cursor — pointer position is mapped
to world coords inside the canvas's `mousemove` handler.

---

## Lab page

### Architecture

```js
recipe = {
  subjects: [
    // shape layer
    { type: 'text', content: 'A8C', style: 'fill',
      color: '#3858E9', bgColor: '#F2F2F2',
      subjectScale: 0.9, meshDensity: 97, strokeWidth: 8,
      particleCount, particleSize, cellSize,
      particleMode, particleDotStyle, particleOutlineThickness,
      particleRepulsion, fontFamily,
      gaussianBlur, roundedCorners,
      effects: [], hidden: false, svg: null },
    // effects layer (auto-created on first global motion/pixel click)
    { type: 'effects', label: 'EFFECTS',
      effects: [{id, params}], pixelEffects: [{id, params}], hidden: false },
    // …
  ],
  activeLayer: 0,
  globalMotion: [],   // legacy; new pickers route into the active EFFECTS layer
  globalPixel:  [],
  transparent: false,
};
```

`recipe.subject` is a **non-enumerable getter alias** for
`recipe.subjects[recipe.activeLayer]`.

### Per-layer state (`layerStates[i]`)

Each layer owns its own:

- `group`, `lineMat`, `fillMat`
- `baseAnchors`, `particleMesh`, `workingAnchors`, `dotSize` (Particles /
  Halftone)
- `fillMesh`, `fillBaseVerts`, `fillWorking` (Solid Fill / Outline — both use
  the same vector-mesh pipeline)
- `renderables` (objects to dispose/clear on rerender)

Module globals (`lineMat`, `fillMat`, `baseAnchors`, `particleMesh`,
`renderables`, `group`, `loadedSvg`, `_workingAnchors`, `dotSize`, `fillMesh`,
`fillBaseVerts`, `fillWorking`, `R` for the active recipe in the HTML-export
runtime) are aliases pointing at the active layer's state. `activateLayer(i)`
re-points all of them; `saveActiveLayer()` writes any reassignments back to
the layer state.

### Type × Style matrix (8 renderers)

| | Text | SVG |
|---|---|---|
| Outline | Stroked text → canvas → **marching squares → ShapeGeometry + holes + subdivide → fillMat mesh** | Stroked SVG path → canvas → same marching squares pipeline |
| Solid fill | Filled text → canvas → marching squares → ShapeGeometry + holes + subdivide → fillMat mesh | SVG path points → THREE.Shape (with hole containment) → ShapeGeometry + subdivide → fillMat mesh |
| Particles | Hershey strokes (line) or rasterised mask (fill) → arc-length distribution → InstancedMesh | SVG paths or rasterised mask → arc-length distribution → InstancedMesh |
| Halftone | Rasterised text → Bayer 4×4 dither grid → InstancedMesh | Rasterised SVG fill → Bayer dither |

Outline and Fill share the **`_labVectorFill`** marker and the
`extractMeshBoundary` SVG-export path.

### Solid Fill / Outline — full vector mesh

The big lift this session. Both styles now produce a real triangulated polygon
mesh whose vertices the effect pipeline mutates each frame:

1. **Contour extraction**.
   - Text & SVG-outline: rasterise into a 1024² alpha mask; run
     `marchingSquares(mask,w,h)` (16-case edge-segment table +
     endpoint-matched threading) to get sub-pixel-accurate closed contours.
   - SVG fill: use the asset's path points directly — already polygonal.
2. **Shape assembly**. `contoursToShapes(worldContours)` classifies by signed
   area (`polyArea > 0` = CCW outer, `< 0` = CW hole), assigns each hole to
   the smallest containing outer via `pointInPoly`, and emits
   `THREE.Shape` objects with `shape.holes` populated. For the text path the
   Y-flip from canvas → world inverts winding, so each polygon is reversed
   before classification.
3. **Triangulation**. `new THREE.ShapeGeometry(shapes)`.
4. **Subdivision**. `subdivideGeometry(geom, passes)` runs midpoint splits
   (every triangle → 4 smaller ones) with a shared-edge midpoint cache so
   neighbouring triangles can't crack apart under heavy deformation.
   `meshDensity` 4–100 maps to 0/1/2/3 passes.
5. **Render**. Mesh uses the layer's `fillMat` (color, no texture). Marked
   `_labVectorFill = true`.
6. **Deform**. `writeFillTransforms(t)` seeds `fillWorking[i]` from
   `fillBaseVerts`, runs (local effects) → (cascade from earlier
   effects-layers) → (global motion), writes mutated x/y back to
   `geometry.attributes.position.array` and sets `needsUpdate = true`. Rot
   and scale anchor fields are ignored — they have no meaning on a 2D mesh.
7. **SVG export**. `extractMeshBoundary(geom)` walks the indexed triangle
   list: edges shared by exactly one triangle are boundary edges. Directed
   edges (from each triangle's winding) chain into closed loops. Each loop
   becomes an `M … L … Z` subpath in a single
   `<path d="…" fill-rule="evenodd">`, so the deformation + holes survive
   exactly.

### Effects layers

User-visible model: every motion/pixel effect lives on an EFFECTS layer.
The col-3 pickers no longer write to `recipe.globalMotion` /
`recipe.globalPixel` directly — they target the active layer's `effects` /
`pixelEffects` arrays. Picking a Motion or Pixel effect from col 3 while a
shape layer is active auto-creates an EFFECTS layer at index 0
(`recipe.subjects.unshift`, `layerStates.unshift(makeLayerState())`). The
"+ Effects" button also unshifts so its cascade reaches every layer beneath.

Cascade rule: an EFFECTS layer at index `j` applies its effects to every
non-effects layer at index `i > j` (the layers rendered after it). The
cumulative-cascade flag is rebuilt per tick.

Param panels for selected effects render in **col 2 under the Layer name**
input. The Motion + Pixel pickers live in col 3 with the order-badge
convention (numbered top-to-bottom). The orientation **flips visually** for
params: the latest-added effect shows as `1.` at the top of the list because
it runs last and visually dominates everything below — apply order in `tick`
is unchanged.

Lab-only filter list (`LAB_MOTION_HIDDEN`) hides `rotation-wave` and
`spiral-attractor`. Pixel effects removed entirely from Lab: `blur`,
`vignette`, `posterize`. They're still available in Composer (the motion
ones).

### Six new effects (this session)

Motion (live in `SEGMENTS.effects`, available everywhere):

- **Orbit** — per-anchor rotation around a movable centre with `1/r` speed
  boost for inner anchors.
- **Glitch** — per-anchor jagged offsets quantised to frames; `Chance` picks
  what fraction flickers each frame.
- **Pulse** — radial Gaussian-bell displacement that sweeps outward from a
  centre at `Period`/`Band width`.

Pixel (in `LAB_PIXEL_EFFECTS`, Lab-only):

- **Bloom** — bright-pixel extraction + 7×7 blur + additive blend.
- **Kaleidoscope** — polar fold + intra-segment mirror.
- **Edge detect** — 3×3 Sobel on luminance, mixed between Edge / Fill colors.

### Per-style defaults (new)

`STYLE_DEFAULTS` table applied on every style swap. Only listed keys are
overwritten:

- Solid fill (initial): subjectScale 0.9, meshDensity 97, color `#3858E9`,
  bg `#F2F2F2`.
- Outline: adds strokeWidth 15.
- Particles: subjectScale 0.9, mode `fill`, outline thickness 0.16,
  repulsion 0.13, count 460, size 18.
- Halftone: dot style `outline`, thickness 0.19, repulsion 0.27, particle
  size 8, cell size 10.

### Lab UI specifics

- **Layers panel** now uses SVG icons for up/down/×/eye. Rows are
  HTML5-draggable with `drop-before` / `drop-after` indicators; reorder swaps
  both `recipe.subjects` and `layerStates` and rewrites `activeLayer`.
- **Card heights** are pinned to the stage canvas height every tick (cheap
  bail when size unchanged). Each card scrolls internally.
- **Pause / Restart**: `labPlayback` carries `paused`, `lastT`, `startOffset`.
  Shift-drag on the canvas scrubs `labPlayback.lastT` while paused.
- **Transparent toggle** is a checkbox under Pause/Restart. When on, clear
  color uses alpha 0, overriding `bgColor`.
- **Section/effect collapse** chevron is `▾` open / `^` collapsed (text swap,
  no CSS rotation).
- **EFFECTS layer** rendering is skipped (no group children); `hidden` on an
  EFFECTS layer disables its motion cascade *and* its pixel chain.
- **Font dropdown** seeds from `LAB_FONTS` (System mono + 15 Google Fonts).
  `<link>` is injected once per session.

### Exports

- **JSON snapshot (v2)**: subjects[] + activeLayer + globalMotion +
  globalPixel + transparent. SVG assets inlined as raw XML on the layer's
  subject. v1 (single-subject) format still loadable.
- **SVG**: walks every layer's renderables and emits real SVG elements
  (polyline, polygon, circle, text). For Solid Fill / Outline meshes,
  `extractMeshBoundary` + `<path fill-rule="evenodd">` carries the deformed
  vertex positions.
- **PNG**: oversamples to 2048² (CSS untouched), runs the pixel chain, grabs
  `toDataURL`. Restores renderer state after.
- **HTML**: still active-layer-only (multi-layer + global motion + pixel
  chain in the runtime is the remaining big follow-up).

---

## New gotchas this session

| Trap | Symptom | Fix |
|---|---|---|
| Marching squares on a Y-flipped mask produces inverted winding | Letter holes render as filled, outers vanish | After mapping pixel coords → world Y-up, reverse each polygon before passing to `contoursToShapes`. |
| ShapeGeometry doesn't add interior vertices | Heavy deformation looks angular | Midpoint subdivision with a shared-edge midpoint cache (so neighbouring triangles share new verts). |
| `recipe.subjects.unshift` without also unshifting `layerStates` | Newly inserted EFFECTS layer reads the wrong per-layer state | Always pair `recipe.subjects.unshift(layer)` with `layerStates.unshift(makeLayerState())`. |
| `+ Effects` button pushed at end of stack | EFFECTS layer never reached the shape layer via cascade | Switched to `unshift` so its effects flow downward to every layer beneath. |
| `controls-host-meta` flex-collapsing to 0px in the Playground middle column | Name input visually stacked on top of the Recipe heading | `flex-shrink: 0` on `.controls-host-meta` + `#segment-pickers`. |
| Canvas filter `blur(N) contrast(20)` gave fuzzy "rounded corners" | Visible halo, edges not crisp | Render with `blur(N)` only, then `getImageData` + hard alpha threshold (>127 → 255, else 0). Gaussian blur is now a separate optional pass. |
| Renderer `alpha:false` made transparent PNG impossible | PNG always had a solid bg | Create Playground & Lab renderers with `alpha: true, premultipliedAlpha: false`. Set clear color alpha 0 when Transparent is on. |
| Aliased edges in transparent PNG export | Stair-stepping at viewport size | Oversample: `renderer.setSize(2048,2048,false)` + update Line2 `material.resolution` for the export, then restore. |
| Hidden controls still rendering | Duplicate UI between left panel and middle-column Recipe picker | Added a `hidden: true` flag in `buildControls`. The param still seeds `ANIM_STATE.params`, but no row renders. |
| Composer Circle source + Circle outline primitive was disabled | "Circle outline" greyed out | `SEGMENT_COMPATIBILITY['parametric-circle']` was missing `'circle-outline'`. Added. |
| Two "Enable gradient" checkboxes (thickness gradient + overlay gradient) | Confusing labels | Renamed the thickness one to "Enable thickness gradient". |
| Transparent overlap of identical-color shape layers reads as "both deformed" | User thought EFFECTS was leaking onto the wrong layer | Cascade is correctly scoped (verified by hiding the deformed layer). Documented; offer per-EFFECTS-layer targeting as a future feature if needed. |

Pre-session gotchas (still apply): `<\/script>` escaping, GLSL `half` keyword,
`align-self: start`, exposed segment helpers, `extractMethod` regex for method
shorthand, per-row materials for varying thickness, `HTML_EXPORT_SECTIONS` map
updates.

---

## Decisions log this session

- **Lab Solid Fill is a real triangulated mesh, not a textured plane.** The
  raster-to-grid intermediate (PlaneGeometry + CanvasTexture, ~2 sessions
  ago) was replaced because vector silhouettes survive zoom and SVG export.
- **Marching squares for text contour extraction**. Browsers don't expose
  font path data without an external parser; rasterise → MS → triangulate is
  the most reliable single-file path.
- **EFFECTS layer pickers replace the old global-motion / global-pixel
  arrays.** Saved JSON snapshots still carry the legacy arrays — both are
  read at apply time, but new clicks never write to them.
- **Particle / Halftone Outline dot style** uses `RingGeometry` instead of a
  two-mesh ring approximation.
- **"Save to Create" lives in `localStorage`**, not in JSON files. Each entry
  carries its full snapshot so loading is self-contained.
- **PNG always supersamples to 2048+**. Display-size PNG was visibly stair-
  stepped on transparent backgrounds; the fixed 2k baseline is the simplest
  knob that solves it across all tools.
- **`hidden: true` flag in `buildControls`** chosen over deleting the
  control entry, so params keep seeding `ANIM_STATE` while the row stays
  out of the left panel.
- **Mirror/Kaleidoscope motion effect was removed earlier in the session**
  (the Kaleidoscope *pixel* effect this session is unrelated). Snapshots
  referencing it silently skip — defensive lookup is
  `SEG.effects[id] || LAB_MOTION_EXTRA[id]`.

---

## Open / paused threads

- **Pixel effects on an EFFECTS layer affect the whole canvas, not just
  layers below.** Motion effects cascade per stack-position (correct), but
  pixel effects are aggregated into a single screen-space post-process
  chain that runs over the entire rendered frame regardless of which
  EFFECTS layer they live on. User has been told about this and explicitly
  parked it — they want the per-position rule eventually. Doing it
  properly requires segmenting the render pipeline: walk the layer stack,
  render each run between EFFECTS layers into its own off-screen target,
  apply that EFFECTS layer's pixel chain to the accumulated buffer, then
  composite the next run on top. Studio + HTML export runtime both need
  the refactor to stay in sync. ~couple hundred LoC.
- **HTML export for Lab — still active-layer only.** Multi-layer composition,
  global motion, and pixel post-process in the exported runtime is the
  largest remaining lift (~300–500 LoC). The studio-side rendering already
  works exactly right; this is a runtime-template extension.
- **Per-EFFECTS-layer targeting** ("apply to only layer X, not all below")
  — user mentioned it but didn't ask for it.
- **Effects on Glyph with per-effect param tuning**. Glyph currently has
  Composer-style Primitive + Effects pickers in the middle column, but each
  picked effect runs with the SEGMENTS-default control values. Dynamic
  per-effect prefixed params (`eff0_strength`, `eff0_scale`, …) would give
  full parity with Composer.
- **Save-to-Playground for Lab tools currently navigates back into Lab to
  view/edit them.** A nicer experience would render the saved tool in
  Playground's stage directly, but that requires teaching the Playground
  loop how to host a Lab recipe — non-trivial.
- **Drag-and-drop Layer reorder** works in Lab; could also be added to the
  per-effect blocks in Lab's col 2 (currently they have ↑/↓ buttons only).
- **Distortion card / Poles** — removed entirely. If you want them back, the
  git history shows the IIFEs that were deleted.

---

## Where to start in a fresh chat

1. Read this file.
2. Scan `// SECTION` markers in `index.html` (~30 seconds).
3. Start the dev server and verify (~2 minutes):
   - Landing page loads, hero star animates, lines thicken under cursor.
   - Playground → click each tool card (Composer, Glyph, Vinyl, Waves, Dots,
     Particles, Halftone); each should render with the curated defaults.
   - Lab → "A8C" renders as solid blue on light gray. Switch styles and the
     per-style defaults kick in. Add an EFFECTS layer + Position noise → the
     fill mesh deforms.
4. Ask the user what they want to do next; don't assume.

## Workflow preferences (still applicable)

- **Asks for many questions before big features.** Use `AskUserQuestion` with
  3–4 multi-choice questions per round.
- **Verifies via screenshots, not text reports.** Use `preview_screenshot`
  after each meaningful change.
- **Prefers signed sliders** + `Softness` for falloff curves.
- **"Lock in defaults" requests are common** — patch `INITIAL_DEFAULTS`,
  not segment defaults (which would affect all consumers).
- **Backups before risky work.** When the user says "let's wrap this up",
  they often want a snapshot preserved.
- **`node --check` for syntax verification without browser reload** — works
  on isolated JS but not on `.html`. Eval-in-preview is the substitute.
- **"Drag and drop" usually means the mental model**, not literal DnD UI —
  multi-select with order badges is what they actually want. Lab Layers do
  have real HTML5 drag-and-drop, but that was a specific ask.
- **One feature per session.** When context is tight, wrap and update this
  file rather than pushing through a partial implementation.
- **Terse chat output.** The user explicitly wants minimal narration —
  toolcalls speak for themselves; only summarise what landed at the end of
  the turn.
