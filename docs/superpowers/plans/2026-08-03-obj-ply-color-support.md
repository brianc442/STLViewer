# OBJ & PLY Color Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add OBJ and PLY loading to `stl_viewer.html`, rendering the color data those formats carry (PLY vertex colors, OBJ's `v x y z r g b` extension, OBJ+MTL+texture), with a global toggle to switch between true color and the existing flat theme color.

**Architecture:** `stl_viewer.html` stays a single static file with no build step. Three more Three.js r128 example loaders (`OBJLoader`, `MTLLoader`, `PLYLoader`, `BufferGeometryUtils`) are added as plain global-namespace `<script>` tags, matching the existing `three.min.js` include. The model-construction code is refactored into shared `buildMeshModel()`/`buildPointCloudModel()` helpers so STL/PLY/OBJ all funnel through the same bookkeeping, and every colored model carries two materials (`materialColor`/`materialFlat`) so the new "Show color" toggle is a cheap `mesh.material` swap.

**Tech Stack:** Vanilla JS, Three.js r128 (global `THREE` namespace, no modules), no package manager, no test framework — verification is manual, in-browser.

## Global Constraints

- No build step or bundler — `stl_viewer.html` stays a single static file; new loaders are additional CDN `<script>` tags in the existing plain-global-namespace style.
- Loader versions must match the pinned Three.js r128 (`https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js`) — use the r128 `examples/js` (non-module, `THREE.X = function(){...}`) loader builds, not `examples/jsm`.
- No server-side code and no network fetches of user-supplied files — `.mtl`/texture resolution for OBJ uses only files present in the same drop/selection batch, resolved via `URL.createObjectURL`.
- Multi-texture OBJs (more than one distinct texture map per object) are out of scope — degrade to a flat per-region baked vertex-color fallback (spec "Non-goals").
- Manual browser verification only — there is no automated test harness in this repo. Every task's test step is a concrete browser action with a stated pass/fail criterion.
- Spec: `docs/superpowers/specs/2026-08-03-obj-ply-color-support-design.md`

---

## Fixture files used across tasks

All test fixtures live in `test-fixtures/` at the repo root. They're created in the task that first needs them.

---

### Task 1: Add Three.js example loader scripts

**Files:**
- Modify: `stl_viewer.html:6` (immediately after the existing `three.min.js` script tag)

**Interfaces:**
- Consumes: nothing (first task).
- Produces: global `THREE.OBJLoader`, `THREE.MTLLoader`, `THREE.PLYLoader`, `THREE.BufferGeometryUtils` constructors/namespaces, available to every later task.

- [ ] **Step 1: Add the four loader `<script>` tags**

In `stl_viewer.html`, immediately after the existing line:

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
```

add:

```html
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/loaders/OBJLoader.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/loaders/MTLLoader.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/loaders/PLYLoader.js"></script>
<script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/utils/BufferGeometryUtils.js"></script>
```

- [ ] **Step 2: Verify the loaders load with no console errors**

Open `stl_viewer.html` directly in a browser (`start stl_viewer.html` from a PowerShell prompt in the repo root, or double-click it in Explorer) with DevTools open on the Console tab.

Expected: no red errors in the console (no 404s, no "THREE is not defined"). In the Console, run:

```js
[THREE.OBJLoader, THREE.MTLLoader, THREE.PLYLoader, THREE.BufferGeometryUtils].every(Boolean)
```

Expected: `true`.

If any of the four scripts 404s, replace only that one line with the cdnjs equivalent (`https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/examples/js/loaders/<Name>.js`, or `.../examples/js/utils/BufferGeometryUtils.js`) and re-check.

- [ ] **Step 3: Commit**

```bash
git add stl_viewer.html
git commit -m "Add OBJ/PLY/MTL Three.js loader scripts"
```

---

### Task 2: Refactor model construction into shared builders with dual-material support

This is the foundational refactor every later task builds on. It changes no visible behavior for STL — it's purely restructuring so PLY/OBJ can plug into the same pipeline.

**Files:**
- Modify: `stl_viewer.html` — `loadSTLData()` (currently ~line 1095), `setTheme()` (currently ~line 734), `setModelOpacity()` (currently ~line 1241), `removeModel()` (currently ~line 1218), `renderModelList()` (currently ~line 1254), `updateClipping()` (currently ~line 1571), the mousedown pick-base handler's `meshList` (currently ~line 871), `computeSectionOutlineAll()` (currently ~line 1526), and the `.model-meta` CSS rule (currently ~line 175).

**Interfaces:**
- Consumes: `THEME_3D`, `currentTheme`, `scene`, `models`, `modelIdCounter`, `wireOn`, `relayoutModels()`, `recomputeCombinedBounds()`, `frameCameraOnModels()`, `refreshSection()`, `cutPlane` (all pre-existing).
- Produces (used by every later task):
  - `buildFlatMaterial(): THREE.MeshPhongMaterial`
  - `buildWireMaterial(): THREE.MeshBasicMaterial`
  - `colorModeLabel(model): string`
  - `finalizeNewModel(model): void` — pushes to `models`, relayouts, reframes camera, renders the list, refreshes an active section.
  - `buildMeshModel(geometry: THREE.BufferGeometry, fileName: string, format: string, colorMode: 'flat'|'vertexColor'|'texture'|'material', colored: boolean, size: THREE.Vector3, materialColorOverride?: THREE.Material): model` — builds mesh + wireframe + group, calls `finalizeNewModel`, returns the model.
  - Model object shape (every field every later task can rely on): `{ id, name, group, mesh, wireMesh, isPointCloud, format, colorMode, colored, materialFlat, materialColor, triCount, vertCount, size, opacity }`.

- [ ] **Step 1: Add `buildFlatMaterial()` and `buildWireMaterial()` helpers**

Add these two functions directly above `loadSTLData` in `stl_viewer.html`:

```js
function buildFlatMaterial(){
  return new THREE.MeshPhongMaterial({
    color: THEME_3D[currentTheme].modelColor,
    specular: 0x111111,
    shininess: 18,
    flatShading: true,
    side: THREE.DoubleSide,
    transparent: false,
    opacity: 1,
    clippingPlanes: []
  });
}

function buildWireMaterial(){
  return new THREE.MeshBasicMaterial({
    color: 0xff9d3d, wireframe: true, transparent: true, opacity: 0.35, clippingPlanes: []
  });
}

function colorModeLabel(m){
  switch(m.colorMode){
    case 'texture': return 'Textured';
    case 'vertexColor': return 'Vertex color';
    case 'material': return 'Material color';
    default: return 'Flat';
  }
}
```

- [ ] **Step 2: Add `finalizeNewModel()` and `buildMeshModel()`**

Add these two functions directly below the helpers from Step 1:

```js
function finalizeNewModel(model){
  models.push(model);
  relayoutModels();
  recomputeCombinedBounds();
  frameCameraOnModels();
  renderModelList();

  document.getElementById('emptyState').style.display = 'none';
  document.getElementById('btnDrawSection').disabled = false;
  document.getElementById('btnPickBase').disabled = false;

  if(cutPlane) refreshSection();
  return model;
}

function buildMeshModel(geometry, fileName, format, colorMode, colored, size, materialColorOverride){
  const materialFlat = buildFlatMaterial();
  const materialColor = colored ? (materialColorOverride || null) : null;

  const meshObj = new THREE.Mesh(geometry, (colored && materialColor) ? materialColor : materialFlat);

  const wireGeo = geometry.clone();
  const wireMeshObj = new THREE.Mesh(wireGeo, buildWireMaterial());
  wireMeshObj.visible = wireOn;

  const group = new THREE.Group();
  group.add(meshObj);
  group.add(wireMeshObj);
  scene.add(group);

  const model = {
    id: ++modelIdCounter,
    name: fileName,
    group, mesh: meshObj, wireMesh: wireMeshObj,
    isPointCloud: false,
    format, colorMode, colored,
    materialFlat, materialColor,
    triCount: geometry.attributes.position.count / 3,
    vertCount: geometry.attributes.position.count,
    size, opacity: 100
  };
  return finalizeNewModel(model);
}
```

Note: this step wires the initial material as "color if available, else flat" — Task 3 will make that respect the `showColor` toggle state.

- [ ] **Step 3: Rewrite `loadSTLData()` to use the shared builder**

Replace the body of `loadSTLData` (from `const geometry = new THREE.BufferGeometry();` through the closing of the function, i.e. everything after the two existing `alert(...)` early-returns) with:

```js
function loadSTLData(buffer, fileName){
  let parsed;
  try {
    parsed = parseSTL(buffer);
  } catch(err){
    alert('Could not parse "' + fileName + '" as STL: ' + err.message);
    return;
  }
  if(!parsed.positions || parsed.positions.length < 9){
    alert('No triangle data found in "' + fileName + '" — is this a valid STL file?');
    return;
  }

  const geometry = new THREE.BufferGeometry();
  geometry.setAttribute('position', new THREE.BufferAttribute(parsed.positions, 3));
  geometry.computeVertexNormals();
  geometry.computeBoundingBox();

  const bbox = geometry.boundingBox;
  const size = new THREE.Vector3();
  bbox.getSize(size);
  const center = new THREE.Vector3();
  bbox.getCenter(center);
  geometry.translate(-center.x, -center.y, -center.z);
  geometry.computeBoundingBox();

  buildMeshModel(geometry, fileName, parsed.format, 'flat', false, size);
}
```

This drops the old inline model-bookkeeping (pushing to `models`, `relayoutModels()`, etc.) since `buildMeshModel` → `finalizeNewModel` now does it.

- [ ] **Step 4: Update `setTheme()` to only recolor the flat material**

Replace:

```js
    models.forEach((m) => {
      m.mesh.material.color.setHex(modelColor);
      m.mesh.material.needsUpdate = true;
    });
```

with:

```js
    models.forEach((m) => {
      m.materialFlat.color.setHex(modelColor);
      m.materialFlat.needsUpdate = true;
    });
```

- [ ] **Step 5: Update `setModelOpacity()` to apply to every material a model has**

Replace the body of `setModelOpacity` with:

```js
function setModelOpacity(id, pct){
  const m = models.find(m => m.id === id);
  if(!m) return;
  m.opacity = pct;
  const a = pct / 100;
  [m.materialFlat, m.materialColor].forEach((mat) => {
    if(!mat) return;
    mat.transparent = a < 1;
    mat.opacity = a;
    mat.needsUpdate = true;
  });
  if(m.wireMesh){
    m.wireMesh.material.opacity = 0.35 * a;
    m.wireMesh.material.needsUpdate = true;
  }
}
```

- [ ] **Step 6: Update `removeModel()` disposal for two materials**

Replace:

```js
    m.mesh.geometry.dispose(); m.mesh.material.dispose();
    m.wireMesh.geometry.dispose(); m.wireMesh.material.dispose();
```

with:

```js
    m.mesh.geometry.dispose();
    m.materialFlat.dispose();
    if(m.materialColor) m.materialColor.dispose();
    if(m.wireMesh){ m.wireMesh.geometry.dispose(); m.wireMesh.material.dispose(); }
```

- [ ] **Step 7: Update `updateClipping()` for two materials**

Replace the body of `updateClipping` with:

```js
function updateClipping(){
  const planes = (clipEnabled && cutPlane) ? [getClipPlane()] : [];
  models.forEach((m) => {
    m.materialFlat.clippingPlanes = planes; m.materialFlat.needsUpdate = true;
    if(m.materialColor){ m.materialColor.clippingPlanes = planes; m.materialColor.needsUpdate = true; }
    if(m.wireMesh){ m.wireMesh.material.clippingPlanes = planes; m.wireMesh.material.needsUpdate = true; }
  });
}
```

- [ ] **Step 8: Exclude point-cloud models from pick-base raycasting**

In the `mousedown` handler, replace:

```js
      const meshList = models.map(m => m.mesh);
```

with:

```js
      const meshList = models.filter(m => !m.isPointCloud).map(m => m.mesh);
```

- [ ] **Step 9: Exclude point-cloud models from section-outline computation**

Replace the body of `computeSectionOutlineAll` with:

```js
function computeSectionOutlineAll(plane){
  let all = [];
  models.forEach((m) => {
    if(m.isPointCloud) return;
    const positions = m.mesh.geometry.attributes.position.array;
    all = all.concat(computeSectionOutlineForModel(positions, m.group.position, plane));
  });
  return new Float32Array(all);
}
```

(`models` has no point-cloud entries yet at this point in the plan — this is forward-compatible groundwork for Task 5.)

- [ ] **Step 10: Add the color-mode tag to the model list, and point/triangle count switching**

Replace the `.model-meta` line inside `renderModelList`'s template string:

```js
        <div class="model-meta">${m.format} · ${m.triCount.toLocaleString()} tris · ${m.size.x.toFixed(1)}×${m.size.z.toFixed(1)}×${m.size.y.toFixed(1)}</div>
```

with:

```js
        <div class="model-meta">${m.format} · <span class="model-color-tag">${colorModeLabel(m)}</span> · ${m.isPointCloud ? m.vertCount.toLocaleString() + ' pts' : m.triCount.toLocaleString() + ' tris'} · ${m.size.x.toFixed(1)}×${m.size.z.toFixed(1)}×${m.size.y.toFixed(1)}</div>
```

- [ ] **Step 11: Add `.model-color-tag` CSS**

In the `<style>` block, immediately after the existing `.model-meta{...}` rule, add:

```css
  .model-color-tag{ color: var(--accent); }
```

- [ ] **Step 12: Regression-test STL loading**

Open `stl_viewer.html` in a browser with DevTools console open. Drag in any existing `.stl` file (or create a trivial one — any STL you have handy).

Confirm, with no console errors:
- Model renders exactly as before (flat theme-colored, matte gray/blue-gray depending on theme).
- Model list row shows `Binary · Flat · N tris · WxDxH` (or `ASCII · Flat · ...`).
- Wireframe toggle still works.
- Opacity slider still works.
- Theme toggle (Light mode switch) still recolors the model.
- Section-cut draw + pick-base still work on it.
- Removing the model still cleans up (no console errors, model disappears).

- [ ] **Step 13: Commit**

```bash
git add stl_viewer.html
git commit -m "Refactor model construction into shared builders with dual-material support"
```

---

### Task 3: Add the global "Show color" toggle

**Files:**
- Modify: `stl_viewer.html` — Display section markup (currently ~line 592, after the "Light mode" toggle row), and the script section near the other toggle wiring (currently ~line 2014, near `tWire`).
- Modify: `buildMeshModel()` (from Task 2) to respect the toggle's initial state.

**Interfaces:**
- Consumes: `models`, `buildMeshModel` (Task 2).
- Produces: module-level `let showColor = true`, `applyColorToggle(): void`, `#tColor` DOM element. Every later loader task's model-building code must set its mesh's initial `.material` using `showColor`, matching the pattern in Step 3 below.

- [ ] **Step 1: Add the toggle row markup**

In the Display `<div class="section">`, immediately after the "Light mode" toggle row (`<div class="toggle-row">...tTheme...</div>`) and before the Brightness `model-opacity-row`, add:

```html
      <div class="toggle-row">
        <label for="tColor">Show color</label>
        <div class="switch on" id="tColor"></div>
      </div>
```

- [ ] **Step 2: Wire the toggle in JS**

Near the other simple toggles (e.g. right after the `tWire` wiring block), add:

```js
  let showColor = true;
  const tColor = document.getElementById('tColor');
  tColor.addEventListener('click', () => {
    showColor = !showColor;
    tColor.classList.toggle('on', showColor);
    applyColorToggle();
  });
  function applyColorToggle(){
    models.forEach((m) => {
      if(!m.colored || !m.materialColor) return;
      m.mesh.material = showColor ? m.materialColor : m.materialFlat;
    });
  }
```

- [ ] **Step 3: Make new models respect the current toggle state at creation time**

In `buildMeshModel` (Task 2, Step 2), replace:

```js
  const meshObj = new THREE.Mesh(geometry, (colored && materialColor) ? materialColor : materialFlat);
```

with:

```js
  const meshObj = new THREE.Mesh(geometry, (colored && materialColor && showColor) ? materialColor : materialFlat);
```

(`showColor` must be declared, per Step 2 above, before `buildMeshModel` is ever called — since `buildMeshModel` is defined earlier in the file than the toggle wiring, this relies on JS function-scoping / hoisting of the outer `let showColor` being in scope by the time `buildMeshModel` *executes* — not when it's *defined*. Since all loads happen after the whole script runs once and event listeners fire later, this is safe. If a linter or the code review here flags it, moving the `let showColor = true;` declaration up next to `let models = [];` is an equally valid fix — do that instead if any load-order issue appears in testing.)

- [ ] **Step 4: Manual verification (no colored models exist yet, so this only checks nothing breaks)**

Open `stl_viewer.html`, load an STL. Click "Show color" on and off several times.

Confirm: no console errors, the switch visually toggles on/off, and the STL model's appearance doesn't change (it has no `materialColor`, so `applyColorToggle` is a no-op for it).

- [ ] **Step 5: Commit**

```bash
git add stl_viewer.html
git commit -m "Add global Show color toggle"
```

---

### Task 4: PLY solid-mesh loading (flat and vertex-colored)

**Files:**
- Create: `test-fixtures/colored-quad.ply`
- Create: `test-fixtures/flat-quad.ply`
- Modify: `stl_viewer.html` — add `loadPLYData()`, extend `handleFile()`'s dispatch, extend dropzone copy and `#fileInput accept`.

**Interfaces:**
- Consumes: `buildMeshModel` (Task 2).
- Produces: `loadPLYData(buffer: ArrayBuffer, fileName: string): void`.

- [ ] **Step 1: Create the colored PLY fixture**

Create `test-fixtures/colored-quad.ply`:

```
ply
format ascii 1.0
element vertex 4
property float x
property float y
property float z
property uchar red
property uchar green
property uchar blue
element face 2
property list uchar int vertex_indices
end_header
0 0 0 255 0 0
1 0 0 0 255 0
1 1 0 0 0 255
0 1 0 255 255 0
3 0 1 2
3 0 2 3
```

- [ ] **Step 2: Create the flat (uncolored) PLY fixture**

Create `test-fixtures/flat-quad.ply`:

```
ply
format ascii 1.0
element vertex 4
property float x
property float y
property float z
element face 2
property list uchar int vertex_indices
end_header
0 0 0
1 0 0
1 1 0
0 1 0
3 0 1 2
3 0 2 3
```

- [ ] **Step 3: Add `loadPLYData()`**

Add this function directly below `loadSTLData` in `stl_viewer.html`:

```js
function loadPLYData(buffer, fileName){
  let geometry;
  try {
    geometry = new THREE.PLYLoader().parse(buffer);
  } catch(err){
    alert('Could not parse "' + fileName + '" as PLY: ' + err.message);
    return;
  }
  if(!geometry.attributes.position || geometry.attributes.position.count < 1){
    alert('No vertex data found in "' + fileName + '" — is this a valid PLY file?');
    return;
  }

  const hasColor = !!geometry.attributes.color;
  const isPointCloud = !geometry.index;

  if(isPointCloud){
    geometry.computeBoundingBox();
    const bbox = geometry.boundingBox;
    const size = new THREE.Vector3();
    bbox.getSize(size);
    const center = new THREE.Vector3();
    bbox.getCenter(center);
    geometry.translate(-center.x, -center.y, -center.z);
    geometry.computeBoundingBox();
    buildPointCloudModel(geometry, fileName, 'PLY', hasColor, size);
    return;
  }

  const nonIndexed = geometry.toNonIndexed();
  if(!nonIndexed.attributes.position || nonIndexed.attributes.position.count < 3){
    alert('No triangle data found in "' + fileName + '" — is this a valid PLY file?');
    return;
  }
  nonIndexed.computeBoundingBox();
  const bbox = nonIndexed.boundingBox;
  const size = new THREE.Vector3();
  bbox.getSize(size);
  const center = new THREE.Vector3();
  bbox.getCenter(center);
  nonIndexed.translate(-center.x, -center.y, -center.z);
  nonIndexed.computeVertexNormals();
  nonIndexed.computeBoundingBox();

  buildMeshModel(nonIndexed, fileName, 'PLY', hasColor ? 'vertexColor' : 'flat', hasColor, size);
}
```

Note: this calls `buildPointCloudModel`, which does not exist until Task 5. That's fine to reference now (JS resolves it at call time, not definition time) — it will not be reachable until a face-less PLY is actually loaded, which only happens starting in Task 5's testing. Do not test point-cloud PLYs in this task.

- [ ] **Step 4: Dispatch `.ply` files to `loadPLYData()`**

Replace `handleFile`:

```js
  function handleFile(file){
    const reader = new FileReader();
    reader.onload = (e) => loadSTLData(e.target.result, file.name);
    reader.onerror = () => alert('Could not read "' + file.name + '".');
    reader.readAsArrayBuffer(file);
  }
```

with:

```js
  function fileExtension(name){
    const idx = name.lastIndexOf('.');
    return idx === -1 ? '' : name.slice(idx + 1).toLowerCase();
  }

  function handleFile(file){
    const ext = fileExtension(file.name);
    const reader = new FileReader();
    reader.onerror = () => alert('Could not read "' + file.name + '".');
    if(ext === 'ply'){
      reader.onload = (e) => loadPLYData(e.target.result, file.name);
      reader.readAsArrayBuffer(file);
    } else {
      reader.onload = (e) => loadSTLData(e.target.result, file.name);
      reader.readAsArrayBuffer(file);
    }
  }
```

- [ ] **Step 5: Extend dropzone copy and file input `accept`**

Replace:

```html
        <div>Drop .stl files here<br>or click to browse — add as many as you like</div>
      </div>
      <input type="file" id="fileInput" accept=".stl" multiple>
```

with:

```html
        <div>Drop .stl or .ply files here<br>or click to browse — add as many as you like</div>
      </div>
      <input type="file" id="fileInput" accept=".stl,.ply" multiple>
```

- [ ] **Step 6: Manual verification**

Open `stl_viewer.html`, drop in `test-fixtures/colored-quad.ply`.

Confirm: renders as a two-triangle quad with a visible red→green / blue→yellow gradient across its 4 corners (Phong-shaded, flat faces); model list row reads `PLY · Vertex color · 2 tris · ...`; toggling "Show color" off turns it the flat theme gray/blue-gray, toggling back on restores the gradient; opacity slider and section-cut both still work on it.

Then drop in `test-fixtures/flat-quad.ply`.

Confirm: renders in the plain flat theme color (no color data in this file); model list row reads `PLY · Flat · 2 tris · ...`; toggling "Show color" has no visible effect on it (it has no `materialColor`).

- [ ] **Step 7: Commit**

```bash
git add stl_viewer.html test-fixtures/colored-quad.ply test-fixtures/flat-quad.ply
git commit -m "Add PLY solid-mesh loading with vertex color support"
```

---

### Task 5: PLY point-cloud loading

**Files:**
- Create: `test-fixtures/points-quad.ply`
- Modify: `stl_viewer.html` — add `buildPointCloudModel()`.

**Interfaces:**
- Consumes: `finalizeNewModel`, `THEME_3D`, `currentTheme`, `showColor` (Tasks 2–3), `loadPLYData`'s point-cloud branch (Task 4, already calls this).
- Produces: `buildPointCloudModel(geometry: THREE.BufferGeometry, fileName: string, format: string, colored: boolean, size: THREE.Vector3): model`. Later OBJ tasks (6) call this too.

- [ ] **Step 1: Create the point-cloud PLY fixture**

Create `test-fixtures/points-quad.ply` — a vertex-only PLY (no `element face` at all, so `THREE.PLYLoader` produces geometry with `index === null`):

```
ply
format ascii 1.0
element vertex 4
property float x
property float y
property float z
property uchar red
property uchar green
property uchar blue
end_header
0 0 0 255 0 0
1 0 0 0 255 0
0 1 0 0 0 255
0 0 1 255 255 0
```

- [ ] **Step 2: Add `buildPointCloudModel()`**

Add this function directly below `buildMeshModel` (Task 2, Step 2):

```js
function buildPointCloudModel(geometry, fileName, format, colored, size){
  const pointSize = Math.max(size.length() * 0.01, 0.005);
  const materialFlat = new THREE.PointsMaterial({
    color: THEME_3D[currentTheme].modelColor, size: pointSize
  });
  const materialColor = colored ? new THREE.PointsMaterial({ vertexColors: true, size: pointSize }) : null;

  const pointsObj = new THREE.Points(geometry, (colored && materialColor && showColor) ? materialColor : materialFlat);

  const group = new THREE.Group();
  group.add(pointsObj);
  scene.add(group);

  const model = {
    id: ++modelIdCounter,
    name: fileName,
    group, mesh: pointsObj, wireMesh: null,
    isPointCloud: true,
    format, colorMode: colored ? 'vertexColor' : 'flat', colored,
    materialFlat, materialColor,
    triCount: 0,
    vertCount: geometry.attributes.position.count,
    size, opacity: 100
  };
  return finalizeNewModel(model);
}
```

- [ ] **Step 3: Make `applyColorToggle()` (Task 3) also cover point clouds**

It already does — it's written generically against `m.colored`/`m.materialColor`/`m.mesh.material`, all of which point clouds have. No change needed; this step is just confirming it in testing below.

- [ ] **Step 4: Manual verification**

Open `stl_viewer.html`, drop in `test-fixtures/points-quad.ply`.

Confirm: renders as 4 small colored dots (red, green, blue, yellow), not a solid surface; model list row reads `PLY · Vertex color · 4 pts · ...` (not "tris"); the "Draw section line" and "Pick flat base" buttons remain enabled (other models can still use them) but clicking either and then clicking/dragging on the point cloud does nothing (no face to hit) — confirm no console error when you try; toggling "Show color" switches it between colored dots and flat theme-colored dots; removing it cleans up with no console errors.

- [ ] **Step 5: Commit**

```bash
git add stl_viewer.html test-fixtures/points-quad.ply
git commit -m "Add PLY point-cloud rendering"
```

---

### Task 6: OBJ loading (plain, vertex-color extension, and point-cloud fallback)

**Files:**
- Create: `test-fixtures/plain-quad.obj`
- Create: `test-fixtures/vertex-color-quad.obj`
- Create: `test-fixtures/points-only.obj`
- Modify: `stl_viewer.html` — add `loadOBJData()`, `finalizeSingleMaterialOBJ()`, `loadOBJPointCloudFallback()`; restructure `handleFiles`/`handleFile` into the batch+companions form; extend dropzone copy and `#fileInput accept`.

**Interfaces:**
- Consumes: `buildMeshModel`, `buildPointCloudModel` (Tasks 2, 5).
- Produces:
  - `handleFiles(fileList): void` — now batches a drop/selection, separating `.obj/.ply/.stl` from companion files (`.mtl`, images).
  - `handleFile(file, companions: Map<string,File>): void`
  - `loadOBJData(objText: string, fileName: string, companions: Map<string,File>): void` — entry point used by `handleFile`; Task 7 extends its body to actually use `companions` for MTL.
  - `finalizeSingleMaterialOBJ(child: THREE.Mesh, fileName: string): void`
  - `loadOBJPointCloudFallback(objText: string, fileName: string): void`

- [ ] **Step 1: Create the plain OBJ fixture (no color)**

Create `test-fixtures/plain-quad.obj`:

```
v 0 0 0
v 1 0 0
v 1 1 0
v 0 1 0
f 1 2 3
f 1 3 4
```

- [ ] **Step 2: Create the vertex-colored OBJ fixture**

Create `test-fixtures/vertex-color-quad.obj` (the `v x y z r g b` extension, colors as 0–1 floats):

```
v 0 0 0 1 0 0
v 1 0 0 0 1 0
v 1 1 0 0 0 1
v 0 1 0 1 1 0
f 1 2 3
f 1 3 4
```

- [ ] **Step 3: Create the point-cloud OBJ fixture (no `f` lines at all)**

Create `test-fixtures/points-only.obj`:

```
v 0 0 0 1 0 0
v 1 0 0 0 1 0
v 0 1 0 0 0 1
v 0 0 1 1 1 0
```

- [ ] **Step 4: Add `loadOBJData()`, `finalizeSingleMaterialOBJ()`, and `loadOBJPointCloudFallback()`**

Add these three functions directly below `loadPLYData`:

```js
function loadOBJData(objText, fileName, companions){
  // MTL support lands in Task 7 — for now, always load without materials.
  finishOBJLoad(objText, fileName, null, companions);
}

function finishOBJLoad(objText, fileName, mtlText, companions){
  const manager = new THREE.LoadingManager();
  manager.setURLModifier((url) => {
    const base = url.split('/').pop().toLowerCase();
    const file = companions.get(base);
    if(!file){
      console.warn(`Texture "${url}" referenced by "${fileName}" was not found in this batch — using material color only.`);
      return url;
    }
    return URL.createObjectURL(file);
  });

  let materials = null;
  if(mtlText){
    try {
      const mtlLoader = new THREE.MTLLoader(manager);
      materials = mtlLoader.parse(mtlText, '');
      materials.preload();
    } catch(err){
      console.warn(`Could not parse material file for "${fileName}": ${err.message}`);
      materials = null;
    }
  }

  const objLoader = new THREE.OBJLoader(manager);
  if(materials) objLoader.setMaterials(materials);

  let group;
  try {
    group = objLoader.parse(objText);
  } catch(err){
    alert('Could not parse "' + fileName + '" as OBJ: ' + err.message);
    return;
  }

  const meshChildren = [];
  group.traverse((child) => { if(child.isMesh) meshChildren.push(child); });

  if(meshChildren.length === 0){
    loadOBJPointCloudFallback(objText, fileName);
    return;
  }

  if(meshChildren.length === 1){
    finalizeSingleMaterialOBJ(meshChildren[0], fileName);
  } else {
    finalizeMultiMaterialOBJ(meshChildren, fileName);
  }
}

function finalizeSingleMaterialOBJ(child, fileName){
  let geometry = child.geometry;
  if(!geometry.attributes.position || geometry.attributes.position.count < 3){
    alert('No triangle data found in "' + fileName + '" — is this a valid OBJ file?');
    return;
  }
  if(geometry.index) geometry = geometry.toNonIndexed();

  const hasVertexColor = !!geometry.attributes.color;
  const srcMat = Array.isArray(child.material) ? child.material[0] : child.material;
  const hasTexture = !!(srcMat && srcMat.map);
  const hasFlatKd = !!(srcMat && srcMat.color && srcMat.color.getHex() !== 0xffffff);

  geometry.computeBoundingBox();
  const bbox = geometry.boundingBox;
  const size = new THREE.Vector3();
  bbox.getSize(size);
  const center = new THREE.Vector3();
  bbox.getCenter(center);
  geometry.translate(-center.x, -center.y, -center.z);
  geometry.computeVertexNormals();
  geometry.computeBoundingBox();

  let colorMode = 'flat';
  let colored = false;
  let materialColorOverride = null;

  if(hasTexture){
    colorMode = 'texture'; colored = true;
    materialColorOverride = new THREE.MeshPhongMaterial({
      color: srcMat.color ? srcMat.color.clone() : new THREE.Color(0xffffff),
      map: srcMat.map,
      specular: 0x111111, shininess: 18, flatShading: true,
      side: THREE.DoubleSide, transparent: false, opacity: 1, clippingPlanes: []
    });
  } else if(hasVertexColor){
    colorMode = 'vertexColor'; colored = true;
  } else if(hasFlatKd){
    colorMode = 'material'; colored = true;
    materialColorOverride = new THREE.MeshPhongMaterial({
      color: srcMat.color.clone(),
      specular: 0x111111, shininess: 18, flatShading: true,
      side: THREE.DoubleSide, transparent: false, opacity: 1, clippingPlanes: []
    });
  }

  buildMeshModel(geometry, fileName, 'OBJ', colorMode, colored, size, materialColorOverride);
}

function loadOBJPointCloudFallback(objText, fileName){
  const positions = [];
  const colors = [];
  let hasColor = false;
  const vertexRe = /^v\s+([-\d.eE+]+)\s+([-\d.eE+]+)\s+([-\d.eE+]+)(?:\s+([-\d.eE+]+)\s+([-\d.eE+]+)\s+([-\d.eE+]+))?/gm;
  let m;
  while((m = vertexRe.exec(objText)) !== null){
    positions.push(parseFloat(m[1]), parseFloat(m[2]), parseFloat(m[3]));
    if(m[4] !== undefined){
      hasColor = true;
      colors.push(parseFloat(m[4]), parseFloat(m[5]), parseFloat(m[6]));
    } else {
      colors.push(1, 1, 1);
    }
  }
  if(positions.length < 3){
    alert('No vertex data found in "' + fileName + '" — is this a valid OBJ file?');
    return;
  }

  const geometry = new THREE.BufferGeometry();
  geometry.setAttribute('position', new THREE.BufferAttribute(new Float32Array(positions), 3));
  if(hasColor) geometry.setAttribute('color', new THREE.BufferAttribute(new Float32Array(colors), 3));

  geometry.computeBoundingBox();
  const bbox = geometry.boundingBox;
  const size = new THREE.Vector3();
  bbox.getSize(size);
  const center = new THREE.Vector3();
  bbox.getCenter(center);
  geometry.translate(-center.x, -center.y, -center.z);
  geometry.computeBoundingBox();

  buildPointCloudModel(geometry, fileName, 'OBJ', hasColor, size);
}
```

`finalizeMultiMaterialOBJ` is referenced above but not defined yet — that's Task 8. It's unreachable until then because none of this task's fixtures produce more than one mesh child.

- [ ] **Step 5: Restructure `handleFiles`/`handleFile` into a batch with a companions map**

Replace:

```js
  function handleFiles(fileList){
    Array.from(fileList || []).forEach(handleFile);
  }

  function fileExtension(name){
    const idx = name.lastIndexOf('.');
    return idx === -1 ? '' : name.slice(idx + 1).toLowerCase();
  }

  function handleFile(file){
    const ext = fileExtension(file.name);
    const reader = new FileReader();
    reader.onerror = () => alert('Could not read "' + file.name + '".');
    if(ext === 'ply'){
      reader.onload = (e) => loadPLYData(e.target.result, file.name);
      reader.readAsArrayBuffer(file);
    } else {
      reader.onload = (e) => loadSTLData(e.target.result, file.name);
      reader.readAsArrayBuffer(file);
    }
  }
```

with:

```js
  function fileExtension(name){
    const idx = name.lastIndexOf('.');
    return idx === -1 ? '' : name.slice(idx + 1).toLowerCase();
  }

  function handleFiles(fileList){
    const files = Array.from(fileList || []);
    const companions = new Map(); // lowercase filename -> File, for .mtl and texture images in this batch
    const loadable = [];
    files.forEach((file) => {
      const ext = fileExtension(file.name);
      if(ext === 'obj' || ext === 'ply' || ext === 'stl'){
        loadable.push(file);
      } else {
        companions.set(file.name.toLowerCase(), file);
      }
    });
    loadable.forEach((file) => handleFile(file, companions));
  }

  function handleFile(file, companions){
    const ext = fileExtension(file.name);
    const reader = new FileReader();
    reader.onerror = () => alert('Could not read "' + file.name + '".');
    if(ext === 'obj'){
      reader.onload = (e) => loadOBJData(e.target.result, file.name, companions || new Map());
      reader.readAsText(file);
    } else if(ext === 'ply'){
      reader.onload = (e) => loadPLYData(e.target.result, file.name);
      reader.readAsArrayBuffer(file);
    } else {
      reader.onload = (e) => loadSTLData(e.target.result, file.name);
      reader.readAsArrayBuffer(file);
    }
  }
```

- [ ] **Step 6: Extend dropzone copy and file input `accept`**

Replace:

```html
        <div>Drop .stl or .ply files here<br>or click to browse — add as many as you like</div>
      </div>
      <input type="file" id="fileInput" accept=".stl,.ply" multiple>
```

with:

```html
        <div>Drop .stl, .obj, or .ply files here<br>or click to browse — add as many as you like</div>
      </div>
      <input type="file" id="fileInput" accept=".stl,.obj,.ply" multiple>
```

- [ ] **Step 7: Manual verification**

Open `stl_viewer.html`, drop in `test-fixtures/plain-quad.obj`.

Confirm: renders as a flat theme-colored quad; model list row reads `OBJ · Flat · 2 tris · ...`.

Drop in `test-fixtures/vertex-color-quad.obj`.

Confirm: renders with the same red/green/blue/yellow gradient look as the PLY equivalent in Task 4; model list row reads `OBJ · Vertex color · 2 tris · ...`; Show-color toggle works on it.

Drop in `test-fixtures/points-only.obj`.

Confirm: renders as 4 colored dots (point cloud), model list row reads `OBJ · Vertex color · 4 pts · ...`; section-cut/pick-base don't error when attempted against it.

- [ ] **Step 8: Commit**

```bash
git add stl_viewer.html test-fixtures/plain-quad.obj test-fixtures/vertex-color-quad.obj test-fixtures/points-only.obj
git commit -m "Add OBJ loading with vertex-color extension and point-cloud fallback"
```

---

### Task 7: OBJ + MTL + texture batch loading

**Files:**
- Create: `test-fixtures/kd-quad.obj`, `test-fixtures/kd-quad.mtl`
- Create: `test-fixtures/textured-quad.obj`, `test-fixtures/textured-quad.mtl`, `test-fixtures/textured-quad.png`
- Create: `test-fixtures/missing-mtl.obj`
- Modify: `stl_viewer.html` — `loadOBJData()` to actually look for and load a companion `.mtl`; extend dropzone copy and `#fileInput accept`.

**Interfaces:**
- Consumes: `finishOBJLoad` (Task 6, already accepts an `mtlText` parameter — this task is what actually populates it).
- Produces: no new function names: `loadOBJData` keeps its Task 6 signature, its body changes.

- [ ] **Step 1: Create the Kd-only (no texture) OBJ+MTL fixture**

Create `test-fixtures/kd-quad.mtl`:

```
newmtl KdMat
Kd 0.200 0.600 0.900
```

Create `test-fixtures/kd-quad.obj`:

```
mtllib kd-quad.mtl
usemtl KdMat
v 0 0 0
v 1 0 0
v 1 1 0
v 0 1 0
f 1 2 3
f 1 3 4
```

- [ ] **Step 2: Create the textured OBJ+MTL+PNG fixture**

Create `test-fixtures/textured-quad.mtl`:

```
newmtl TexMat
Kd 1.000 1.000 1.000
map_Kd textured-quad.png
```

Create `test-fixtures/textured-quad.obj`:

```
mtllib textured-quad.mtl
v 0 0 0
v 1 0 0
v 1 1 0
v 0 1 0
vt 0 0
vt 1 0
vt 1 1
vt 0 1
usemtl TexMat
f 1/1 2/2 3/3
f 1/1 3/3 4/4
```

Create `test-fixtures/textured-quad.png` — a minimal valid 1×1 pixel PNG, via Bash:

```bash
echo "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNk+A8AAQUBAScY42YAAAAASUVORK5CYII=" | base64 -d > test-fixtures/textured-quad.png
```

- [ ] **Step 3: Create the "missing companion" fixture**

Create `test-fixtures/missing-mtl.obj` (references an `.mtl` that is deliberately never created):

```
mtllib does-not-exist.mtl
v 0 0 0
v 1 0 0
v 1 1 0
f 1 2 3
```

- [ ] **Step 4: Update `loadOBJData()` to find and load a companion MTL**

Replace:

```js
function loadOBJData(objText, fileName, companions){
  // MTL support lands in Task 7 — for now, always load without materials.
  finishOBJLoad(objText, fileName, null, companions);
}
```

with:

```js
function loadOBJData(objText, fileName, companions){
  const mtlMatch = objText.match(/^mtllib\s+(.+)$/m);
  const mtlName = mtlMatch ? mtlMatch[1].trim().toLowerCase() : null;
  const mtlFile = mtlName ? companions.get(mtlName) : null;

  if(!mtlFile){
    if(mtlName){
      console.warn(`"${fileName}" references mtllib "${mtlName}" but no matching file was included in this batch — loading without material color.`);
    }
    finishOBJLoad(objText, fileName, null, companions);
    return;
  }

  const mtlReader = new FileReader();
  mtlReader.onerror = () => {
    console.warn(`Could not read "${mtlFile.name}" — loading "${fileName}" without material color.`);
    finishOBJLoad(objText, fileName, null, companions);
  };
  mtlReader.onload = (e) => finishOBJLoad(objText, fileName, e.target.result, companions);
  mtlReader.readAsText(mtlFile);
}
```

- [ ] **Step 5: Extend dropzone copy and file input `accept`**

Replace:

```html
        <div>Drop .stl, .obj, or .ply files here<br>or click to browse — add as many as you like</div>
      </div>
      <input type="file" id="fileInput" accept=".stl,.obj,.ply" multiple>
```

with:

```html
        <div>Drop .stl, .obj, or .ply files here — for textured OBJs, drop the .mtl and image together<br>or click to browse — add as many as you like</div>
      </div>
      <input type="file" id="fileInput" accept=".stl,.obj,.ply,.mtl,.jpg,.jpeg,.png" multiple>
```

- [ ] **Step 6: Manual verification — Kd-only material**

Open `stl_viewer.html`. Select both `test-fixtures/kd-quad.obj` and `test-fixtures/kd-quad.mtl` together (multi-select in the file picker, or drag both at once) and drop/open them.

Confirm: renders as a flat blue quad (Kd `0.2, 0.6, 0.9`); model list row reads `OBJ · Material color · 2 tris · ...`; Show-color toggle switches it between blue and flat theme color.

- [ ] **Step 7: Manual verification — textured material**

Select `test-fixtures/textured-quad.obj`, `test-fixtures/textured-quad.mtl`, and `test-fixtures/textured-quad.png` together and drop/open them.

Confirm: no console errors; the quad loads without throwing (texture is a single pixel so the surface will appear as a solid color sampled from that pixel — this is only confirming the texture pipeline doesn't error, not a visual color check); model list row reads `OBJ · Textured · 2 tris · ...`; Show-color toggle works on it.

- [ ] **Step 8: Manual verification — missing companion fallback**

Open `test-fixtures/missing-mtl.obj` alone (do not include any `.mtl`).

Confirm: it still loads successfully as a flat theme-colored triangle (does not error or alert); model list row reads `OBJ · Flat · 1 tris · ...`; DevTools console shows the warning `"missing-mtl.obj" references mtllib "does-not-exist.mtl" but no matching file was included in this batch — loading without material color.`

- [ ] **Step 9: Commit**

```bash
git add stl_viewer.html test-fixtures/kd-quad.obj test-fixtures/kd-quad.mtl test-fixtures/textured-quad.obj test-fixtures/textured-quad.mtl test-fixtures/textured-quad.png test-fixtures/missing-mtl.obj
git commit -m "Add OBJ+MTL+texture batch loading with missing-companion fallback"
```

---

### Task 8: OBJ multi-material merge fallback

**Files:**
- Create: `test-fixtures/two-material.obj`, `test-fixtures/two-material.mtl`
- Modify: `stl_viewer.html` — add `finalizeMultiMaterialOBJ()` (referenced but undefined since Task 6).

**Interfaces:**
- Consumes: `THREE.BufferGeometryUtils.mergeBufferGeometries` (Task 1), `buildMeshModel` (Task 2), `finishOBJLoad`'s existing call site (Task 6).
- Produces: `finalizeMultiMaterialOBJ(meshChildren: THREE.Mesh[], fileName: string): void`.

- [ ] **Step 1: Create the two-material OBJ+MTL fixture**

Create `test-fixtures/two-material.mtl`:

```
newmtl RedMat
Kd 1.000 0.000 0.000

newmtl BlueMat
Kd 0.000 0.000 1.000
```

Create `test-fixtures/two-material.obj` (two quads side by side, one red, one blue):

```
mtllib two-material.mtl
v 0 0 0
v 1 0 0
v 1 1 0
v 0 1 0
v 2 0 0
v 2 1 0
usemtl RedMat
f 1 2 3
f 1 3 4
usemtl BlueMat
f 2 5 6
f 2 6 3
```

- [ ] **Step 2: Add `finalizeMultiMaterialOBJ()`**

Add this function directly below `finalizeSingleMaterialOBJ`:

```js
function finalizeMultiMaterialOBJ(meshChildren, fileName){
  const geometries = meshChildren.map((child) => {
    let geo = child.geometry;
    if(geo.index) geo = geo.toNonIndexed();
    if(!geo.attributes.color){
      const mat = Array.isArray(child.material) ? child.material[0] : child.material;
      const c = (mat && mat.color) ? mat.color : new THREE.Color(0xffffff);
      const count = geo.attributes.position.count;
      const colors = new Float32Array(count * 3);
      for(let i = 0; i < count; i++){
        colors[i * 3] = c.r; colors[i * 3 + 1] = c.g; colors[i * 3 + 2] = c.b;
      }
      geo.setAttribute('color', new THREE.BufferAttribute(colors, 3));
    }
    return geo;
  });

  const merged = THREE.BufferGeometryUtils.mergeBufferGeometries(geometries, false);
  if(!merged || !merged.attributes.position || merged.attributes.position.count < 3){
    alert('No triangle data found in "' + fileName + '" — is this a valid OBJ file?');
    return;
  }

  merged.computeBoundingBox();
  const bbox = merged.boundingBox;
  const size = new THREE.Vector3();
  bbox.getSize(size);
  const center = new THREE.Vector3();
  bbox.getCenter(center);
  merged.translate(-center.x, -center.y, -center.z);
  merged.computeVertexNormals();
  merged.computeBoundingBox();

  buildMeshModel(merged, fileName, 'OBJ', 'material', true, size);
}
```

- [ ] **Step 3: Manual verification**

Select `test-fixtures/two-material.obj` and `test-fixtures/two-material.mtl` together and drop/open them.

Confirm: no console errors; renders as two adjacent quads, one solid red and one solid blue (no crash from the merge); model list row reads `OBJ · Material color · 4 tris · ...`; Show-color toggle switches the whole merged mesh between the red/blue coloring and flat theme color; section-cut and pick-base both still work on the merged mesh (it's a normal single `Mesh` as far as those tools are concerned).

- [ ] **Step 4: Full regression pass**

With all 8 tasks done, do one combined check: load `test-fixtures/flat-quad.ply` (STL-style flat), `test-fixtures/colored-quad.ply` (vertex color), and `test-fixtures/textured-quad.obj` + its `.mtl` + `.png` (texture) all into the same scene together.

Confirm: all three sit side-by-side (existing `relayoutModels` behavior), the combined bounding box frames all three (`frameCameraOnModels`), drawing one section line slices through all three at once, and toggling "Show color" affects the vertex-colored and textured ones but not any plain STL/flat ones in the scene.

- [ ] **Step 5: Commit**

```bash
git add stl_viewer.html test-fixtures/two-material.obj test-fixtures/two-material.mtl
git commit -m "Add OBJ multi-material merge fallback"
```
