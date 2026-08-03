# OBJ & PLY Support with Color Rendering — Design

## Goal

`stl_viewer.html` currently loads only STL, which has no color information — every
model is rendered in one flat, theme-driven color. Add support for **OBJ** and
**PLY** files, and render the color information those formats can carry:

- PLY per-vertex RGB(A)
- OBJ's common `v x y z r g b` vertex-color extension
- OBJ + MTL (+ texture image) material color / texture mapping

A global toggle lets the user switch between showing detected color and the
existing flat theme-colored look, per loaded scene.

## Non-goals

- Multi-texture OBJs (more than one distinct texture map across an object) —
  degrades to a flat per-region color fallback rather than full texturing.
- Fetching `.mtl`/texture files from a server or URL — only files the user
  selects/drops locally are considered.
- Editing/exporting color data.

## Library choice

`stl_viewer.html` already loads Three.js r128 as a single global-namespace
`<script src=".../three.min.js">` with no build step. Rather than hand-rolling
OBJ/PLY/MTL parsers (binary PLY endianness, face fan-triangulation, MTL syntax,
texture coordinate handling), add three more CDN `<script>` tags for the
matching r128 **non-module** example loaders:

- `examples/js/loaders/OBJLoader.js`
- `examples/js/loaders/MTLLoader.js`
- `examples/js/loaders/PLYLoader.js`
- `examples/js/utils/BufferGeometryUtils.js` (for the multi-material merge
  fallback)

These attach to the global `THREE` namespace, matching the file's existing
plain-script pattern — no bundler, no module system change.

## File intake & multi-file bundling

The dropzone and file input (`accept`, currently `.stl`) extend to
`.stl,.obj,.ply,.mtl` plus common image extensions (`.jpg,.jpeg,.png`).

Today, `handleFiles` reads and loads every file independently and
immediately. It changes to process one **batch** — one drop event or one
file-picker selection — together:

1. Split the incoming `FileList` into "loadable" files (`.stl`/`.obj`/`.ply`)
   and "companion" files (`.mtl`, images).
2. Stage companions into a `basename → File` lookup map for this batch.
3. Load each loadable file. For an `.obj`, scan its text for a `mtllib <name>`
   line; if a matching `.mtl` is staged, parse it (see below) before parsing
   the OBJ geometry.

This only matches files that are selected/dropped together in the same batch,
per the user's confirmed workflow (select/drag the `.obj` + `.mtl` + image
together, same as most desktop viewers).

## Format parsing

**STL** — unchanged (existing hand-rolled binary/ASCII parser).

**PLY** — parsed with `THREE.PLYLoader`, which natively populates `position`,
`normal`, `uv`, and `color` BufferAttributes when present in the file (ASCII or
binary, either endianness).

**OBJ** — parsed with `THREE.OBJLoader`, which supports the `v x y z r g b`
vertex-color convention natively. If a companion `.mtl` was found:
   - Build a `THREE.LoadingManager` with `setURLModifier` that redirects any
     texture filename referenced in the `.mtl` (e.g. `map_Kd scan.jpg`) to
     `URL.createObjectURL(file)` for the matching staged image file. This
     needs no server — works from `file://` or any static host.
   - Parse the `.mtl` text with `THREE.MTLLoader(manager)`, call
     `.preload()` to kick off texture loads, and hand the resulting materials
     to `THREE.OBJLoader(manager).setMaterials(materials)` before parsing the
     OBJ text.

## Color modes

Every loaded model gets a `colored: boolean` flag:

| Case | `colored` | Material |
|---|---|---|
| STL, or OBJ/PLY with no color/material data | `false` | flat `MeshPhongMaterial`, theme color |
| PLY vertex colors, or OBJ `v x y z r g b` | `true` | `vertexColors: true`, base white |
| OBJ + MTL, single material, has a texture | `true` | material `.map` = loaded texture, color = `Kd` |
| OBJ + MTL, single material, no texture | `true` | material color = `Kd` |
| OBJ + MTL, multiple materials (rare) | `true` | geometries merged via `BufferGeometryUtils`; each submesh's resolved color (Kd, or a textured submesh's own Kd as an approximation) is baked into a per-vertex color attribute on the merged mesh — a flat-per-region fallback, not true multi-texturing |

## Global "Show color" toggle

A new toggle row, **Show color** (default on), is added to the Display panel
alongside Wireframe / Ground grid / Auto-rotate / Light mode.

Every model with `colored: true` builds **two** materials at load time:

- `materialColor` — the true-color material as described above.
- `materialFlat` — a plain `MeshPhongMaterial` in the current theme's flat
  model color, identical in kind to what an uncolored STL gets.

`model.mesh.material` points at whichever is currently active. Flipping the
global toggle reassigns `.material` on every colored model's mesh between
`materialColor` and `materialFlat` — no property mutation on a shared
material, so there's no risk of a stale `map`/`vertexColors` flag surviving a
toggle. Uncolored models (`colored: false`) are unaffected — they only ever
have the one theme-following material, exactly as today.

Interaction with existing controls:

- **Theme toggle** (`setTheme`) — changes from unconditionally recoloring
  every model's material to only recoloring `materialFlat` on every model
  (both the active one on plain models, and the standby one on colored
  models). This keeps the flat fallback correct the instant color is toggled
  off, even if the theme changed while color was on.
- **Opacity slider** — applies `transparent`/`opacity` to *both* materials on
  a colored model, so toggling color doesn't reset opacity.
- **Wireframe overlay** — unaffected; it's always a separate flat-orange
  overlay mesh regardless of color mode.
- **Model list color-mode tag** (see UI section) stays visible regardless of
  toggle state — it reflects what the file *contains*, not what's currently
  displayed.

## Geometry normalization

The section-cut outline/clip code and the reorient ("pick flat base") code
both assume a flat, non-indexed triangle soup: `position[i..i+8]` is one
triangle's three vertices, with no index buffer. That's what the existing STL
parser already produces directly.

`OBJLoader`/`PLYLoader` typically return **indexed** geometry. After loading,
every new geometry is passed through `geometry.toNonIndexed()` before
anything else touches it. This correctly expands `color`/`uv`/`normal`
attributes alongside `position` (built-in Three.js behavior), so no color
data is lost. This keeps section-cut, clipping, and pick-base working
unchanged for every format, with no changes needed to that existing code.

## Point clouds

If a PLY/OBJ has vertex data but zero faces, it's rendered as `THREE.Points`
with a small `PointsMaterial({ vertexColors: true })` if vertex colors are
present, or a flat theme-following point color otherwise (subject to the same
`colored` flag and Show-color toggle as meshes).

Point-cloud models are excluded from:
- the section-cut raycasting/outline computation loop, and
- the "pick flat base" raycasting target list,

since neither operation is meaningful without faces. No wireframe overlay is
built for point-cloud models.

## UI changes

- Dropzone copy and `#fileInput accept` extend to cover `.obj`, `.ply`,
  `.mtl`, and common image extensions.
- Each row in the Loaded Models list gains a short color-mode tag next to the
  existing format label, e.g. `OBJ · Textured`, `PLY · Vertex color`,
  `STL · Flat`, `OBJ · Material color`. This is informational only — it does
  not change with the Show-color toggle.
- New **Show color** toggle row in the Display section, next to the existing
  Wireframe / Ground grid / Auto-rotate / Light mode toggles.

## Error handling

- Unparseable files keep the existing `alert(...)` pattern used by STL today.
- A `.mtl` or texture referenced by an OBJ but not found in the same batch
  logs a console warning and falls back to flat theme color — it does not
  block the load.

## Testing plan

Manual verification in-browser (this is a static HTML file with no test
harness):

1. Load an existing `.stl` — confirm no regression (flat theme color, section
   cut, pick-base, wireframe, opacity all still work).
2. Load a PLY with per-vertex color, no faces present (point cloud) — renders
   as colored points; section-cut/pick-base tools stay disabled/no-op for it.
3. Load a PLY with per-vertex color and faces — renders as a shaded colored
   mesh; section-cut and pick-base work on it.
4. Load a plain OBJ with only `v`/`f` lines (no color, no mtllib) — renders
   flat theme color like STL.
5. Load an OBJ using the `v x y z r g b` extension — renders vertex-colored.
6. Load an OBJ + matching MTL + JPG/PNG texture dropped together — renders
   textured; toggling Show color off reverts it to flat theme color and back
   on restores the texture.
7. Load an OBJ + MTL with multiple materials — confirm the flat-per-region
   fallback color bake, not a crash.
8. Toggle theme (light/dark) while a colored model is loaded, both with Show
   color on and off — confirm the flat material always tracks the theme, and
   the true-color material is never touched by theme.
9. Adjust the per-model opacity slider on a colored model, then toggle Show
   color — confirm opacity persists across the toggle.
10. Mixed scene: one STL + one colored PLY + one textured OBJ loaded together
    — confirm relayout, combined bounding box/camera framing, and section cut
    across all three still work.
