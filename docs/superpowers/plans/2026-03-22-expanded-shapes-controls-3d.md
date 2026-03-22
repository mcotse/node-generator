# Expanded Node Shapes, Controls & 3D Enhancements — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add rich variety of node shapes (10+ geometries including custom ones), shape-by-node-type mapping, advanced material controls, and 3D visual enhancements (animation, shadows, SSAO) to the Node Pathway Visualizer.

**Architecture:** All changes are in the single `index.html` file. New geometries are added to `NodeRenderer.geometries`. A shape-mapping system lets users assign different shapes to root/branch/leaf nodes. New material params (metalness, roughness, wireframe) are added to `nodeParams`. 3D enhancements include node rotation animation, auto-orbit, and optional shadow casting. Edge enhancements add dashed-line and animated-flow options.

**Tech Stack:** Three.js r170 (existing), lil-gui v0.20 (existing), no new dependencies.

---

## File Map

All changes are in `index.html`. References use class/method names (search for `class ClassName` or `methodName(` to locate).

| Location (semantic) | Changes |
|---------------------|---------|
| `NodeRenderer` constructor → `this.geometries` | Add 9 new geometries + diamond custom geometry |
| `NodeRenderer.update()` | Shape-by-type multi-mesh logic, wireframe support, scale params |
| `NodeRenderer` (new method) | `tick()` for pulse/spin animation (handles both single + multi-mesh) |
| `NodeRenderer.dispose()` | Updated to handle `this.meshes[]` array |
| `EdgeRenderer.update()` | Dashed material option, `computeLineDistances()` |
| `EdgeRenderer` (new method) | `tick()` for flow animation |
| `InteractionManager.onMouseMove/onClick` | Multi-mesh raycasting |
| `GUIManager.buildNodeFolder()` | Shape dropdown (12 options), shape-by-type, materials, animation, scale |
| `GUIManager.buildEdgeFolder()` | Dashed style, flow animation controls |
| `GUIManager.buildEnvironmentFolder()` | Shadow toggle, auto-orbit controls |
| `App` constructor → `this.nodeParams` | New defaults for materials, animation, shape-by-type, scale |
| `App` constructor → `this.edgeParams` | New defaults for style, flow animation |
| `App` constructor → `this.envParams` | New defaults for shadows, auto-orbit |
| `App` constructor (scene setup) | Shadow map config, ground plane mesh |
| `App.animate()` | Call `nodeRenderer.tick()` + `edgeRenderer.tick()` + auto-orbit |
| Top-level imports | Add `mergeGeometries` from BufferGeometryUtils |

## Task Dependencies

```
Task 1 (geometries) → Task 2 (shape-by-type) → Task 3 (materials)
                                                      ↓
                                          ┌───────────┼───────────┐
                                          ↓           ↓           ↓
                                      Task 4      Task 5      Task 6
                                     (animate)   (shadows)   (edges)
                                          ↓
                                      Task 7
                                     (scale)
```

Tasks 1→2→3 are sequential (each builds on prior). After Task 3, Tasks 4/5/6 are parallel. Task 7 depends on Task 4 (shares scale calculation in tick()).

---

### Task 1: Add New Node Geometries

**Files:**
- Modify: `index.html` → `NodeRenderer` constructor (`this.geometries` object)
- Modify: `index.html` → `GUIManager.buildNodeFolder()` (shape dropdown)
- Modify: `index.html` → top-level imports (add `mergeGeometries`)

- [ ] **Step 1: Add 9 new geometries to NodeRenderer.geometries**

In the `NodeRenderer` constructor, replace the geometries object:

```javascript
this.geometries = {
  sphere: new THREE.SphereGeometry(1, 16, 12),
  cube: new THREE.BoxGeometry(1, 1, 1),
  octahedron: new THREE.OctahedronGeometry(1),
  // New shapes
  tetrahedron: new THREE.TetrahedronGeometry(1),
  icosahedron: new THREE.IcosahedronGeometry(1, 0),
  dodecahedron: new THREE.DodecahedronGeometry(1, 0),
  cone: new THREE.ConeGeometry(0.7, 1.4, 8),
  cylinder: new THREE.CylinderGeometry(0.5, 0.5, 1.2, 8),
  torus: new THREE.TorusGeometry(0.6, 0.25, 8, 16),
  torusknot: new THREE.TorusKnotGeometry(0.5, 0.18, 48, 8),
  capsule: new THREE.CapsuleGeometry(0.4, 0.6, 4, 8),
  diamond: null, // custom — built in step 2
};
```

- [ ] **Step 2: Create custom diamond geometry**

Add a method to `NodeRenderer` that builds a diamond (two cones tip-to-tip):

```javascript
_buildDiamondGeometry() {
  const top = new THREE.ConeGeometry(0.7, 1, 6);
  top.translate(0, 0.5, 0);
  const bottom = new THREE.ConeGeometry(0.7, 0.6, 6);
  bottom.rotateX(Math.PI);
  bottom.translate(0, -0.3, 0);
  const merged = mergeGeometries([top, bottom]);
  return merged;
}
```

Import `mergeGeometries` from Three.js addons:
```javascript
import { mergeGeometries } from 'three/addons/utils/BufferGeometryUtils.js';
```

Then in the constructor, set: `this.geometries.diamond = this._buildDiamondGeometry();`

- [ ] **Step 3: Update GUI shape dropdown**

Change the shape dropdown in `buildNodeFolder` to include all shapes:

```javascript
f.add(this.app.nodeParams, 'shape', [
  'sphere', 'cube', 'octahedron', 'tetrahedron', 'icosahedron',
  'dodecahedron', 'cone', 'cylinder', 'torus', 'torusknot',
  'capsule', 'diamond'
]).name('Shape').onChange(() => this.app.updateVisuals());
```

- [ ] **Step 4: Verify in browser**

Open `index.html` in browser. Cycle through all 12 shapes in the dropdown. Each should render correctly with the existing color-by-depth gradient.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add 9 new node geometries (tetrahedron, icosahedron, dodecahedron, cone, cylinder, torus, torusknot, capsule, diamond)"
```

---

### Task 2: Shape-by-Node-Type Mapping

**Depends on:** Task 1

**Files:**
- Modify: `index.html` → `App` constructor (`this.nodeParams`)
- Modify: `index.html` → `NodeRenderer.update()` (multi-mesh rendering)
- Modify: `index.html` → `NodeRenderer.dispose()` (multi-mesh cleanup)
- Modify: `index.html` → `NodeRenderer.getNodeIdAtIndex/getIndexForNodeId` (multi-mesh lookup)
- Modify: `index.html` → `InteractionManager.onMouseMove/onClick` (multi-mesh raycasting)
- Modify: `index.html` → `GUIManager.buildNodeFolder()` (shape-by-type controls)

- [ ] **Step 1: Add shape-mapping params**

Add to `this.nodeParams` in the App constructor:

```javascript
shapeByType: false,     // toggle
rootShape: 'icosahedron',
branchShape: 'sphere',
leafShape: 'diamond',
```

- [ ] **Step 2: Update NodeRenderer to support multi-geometry rendering**

When `shapeByType` is true, the renderer creates up to 3 `InstancedMesh` objects — one per shape type. Modify `NodeRenderer`:

```javascript
update(graph, params) {
  this.dispose();
  const { shape, size, color, opacity, emissive, emissiveIntensity,
          colorByDepth, leafColor, shapeByType, rootShape, branchShape, leafShape } = params;

  const count = graph.nodes.size;
  if (count === 0) return;

  // Determine shape for each node
  const nodeEntries = Array.from(graph.nodes.entries());

  if (shapeByType) {
    // Group nodes by type -> shape
    const shapeMap = { root: rootShape, branch: branchShape, leaf: leafShape };
    const groups = {}; // shape -> [nodeEntries]
    for (const entry of nodeEntries) {
      const type = entry[1].metadata.type || 'branch';
      const s = shapeMap[type] || shape;
      if (!groups[s]) groups[s] = [];
      groups[s].push(entry);
    }
    this.meshes = [];
    this.nodeIds = [];
    let globalIndex = 0;
    for (const [shapeName, entries] of Object.entries(groups)) {
      const geo = this.geometries[shapeName] || this.geometries.sphere;
      const mat = this._makeMaterial(color, opacity, emissive, emissiveIntensity);
      const mesh = new THREE.InstancedMesh(geo, mat, entries.length);
      // ... set transforms and colors per instance (same logic as current)
      this.meshes.push(mesh);
      this.scene.add(mesh);
    }
  } else {
    // Current single-mesh path (unchanged)
  }
}
```

Always store meshes in `this.meshes` array (even single-mesh mode uses a 1-element array). Store a parallel `this.meshNodeIds` array-of-arrays mapping mesh index → nodeId arrays.

- [ ] **Step 3: Update dispose(), getNodeIdAtIndex(), getIndexForNodeId()**

```javascript
dispose() {
  for (const mesh of this.meshes) {
    this.scene.remove(mesh);
    mesh.material.dispose();
    mesh.dispose();
  }
  this.meshes = [];
  this.nodeIds = [];
  this.meshNodeIds = [];
}

getNodeIdAtIndex(meshIndex, instanceId) {
  // For multi-mesh: caller provides which mesh was hit
  return this.meshNodeIds[meshIndex]?.[instanceId];
}

getMeshIndex(mesh) {
  return this.meshes.indexOf(mesh);
}
```

Update the single-mesh code path to also use `this.meshes = [mesh]` and `this.meshNodeIds = [nodeIds]` so all downstream code works uniformly.

- [ ] **Step 4: Update InteractionManager for multi-mesh raycasting**

Modify `onMouseMove`, `onClick`, and `highlightNode` to work with `this.app.nodeRenderer.meshes` array:

```javascript
// Shared helper — use in onMouseMove and onClick
_findHitNode() {
  const meshes = this.app.nodeRenderer.meshes;
  let closestHit = null, hitMesh = null;
  for (const mesh of meshes) {
    const hits = this.raycaster.intersectObject(mesh);
    if (hits.length > 0 && (!closestHit || hits[0].distance < closestHit.distance)) {
      closestHit = hits[0];
      hitMesh = mesh;
    }
  }
  if (!closestHit || !hitMesh) return null;
  const meshIdx = this.app.nodeRenderer.getMeshIndex(hitMesh);
  const nodeId = this.app.nodeRenderer.getNodeIdAtIndex(meshIdx, closestHit.instanceId);
  return nodeId;
}
```

Update `highlightNode()` to iterate all meshes and their `meshNodeIds` when coloring instances:

```javascript
highlightNode(nodeId) {
  this.clearHighlight();
  this.highlightedNodeId = nodeId;
  const neighborIds = this.app.graph.getNeighborIds(nodeId);
  const highlightMain = new THREE.Color(0xff3300);
  const highlightNeighbor = new THREE.Color(0xff6600);

  for (let m = 0; m < this.app.nodeRenderer.meshes.length; m++) {
    const mesh = this.app.nodeRenderer.meshes[m];
    const ids = this.app.nodeRenderer.meshNodeIds[m];
    if (!mesh.instanceColor) continue;
    for (let i = 0; i < ids.length; i++) {
      if (ids[i] === nodeId) mesh.setColorAt(i, highlightMain);
      else if (neighborIds.has(ids[i])) mesh.setColorAt(i, highlightNeighbor);
    }
    mesh.instanceColor.needsUpdate = true;
  }
  // Edge highlighting (unchanged)
  const edgeIndices = this.app.graph.getConnectedEdgeIndices(nodeId);
  this.app.edgeRenderer.setHighlight(edgeIndices, '#ff6600');
}
```

- [ ] **Step 5: Add GUI controls for shape-by-type**

In `buildNodeFolder`, add after the Shape dropdown:

```javascript
f.add(this.app.nodeParams, 'shapeByType')
  .name('Shape by Type').onChange(() => this.app.updateVisuals());
f.add(this.app.nodeParams, 'rootShape', shapeList)
  .name('Root Shape').onChange(() => this.app.updateVisuals());
f.add(this.app.nodeParams, 'branchShape', shapeList)
  .name('Branch Shape').onChange(() => this.app.updateVisuals());
f.add(this.app.nodeParams, 'leafShape', shapeList)
  .name('Leaf Shape').onChange(() => this.app.updateVisuals());
```

- [ ] **Step 6: Add `_lastGraph` and `_maxDepth` storage in update()**

At the end of `NodeRenderer.update()`, store graph reference for use by `tick()` (Task 4):

```javascript
this._lastGraph = graph;
this._maxDepth = maxDepth; // already computed in the color-by-depth loop
```

- [ ] **Step 7: Verify in browser**

Toggle "Shape by Type" on. Roots should show as icosahedrons, branches as spheres, leaves as diamonds. Toggle off — all nodes should use the single "Shape" dropdown value. Test hover tooltips and click highlighting work with both modes.

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat: add shape-by-type mapping (different shapes for root/branch/leaf nodes)"
```

---

### Task 3: Advanced Material Controls

**Depends on:** Task 2

**Files:**
- Modify: `index.html` → `App` constructor (`this.nodeParams`, `this.edgeParams`)
- Modify: `index.html` → `NodeRenderer._makeMaterial()` or material creation in `update()`
- Modify: `index.html` → `EdgeRenderer.update()` material creation
- Modify: `index.html` → `GUIManager.buildNodeFolder()` and `buildEdgeFolder()`

- [ ] **Step 1: Add material params**

Add to `this.nodeParams`:

```javascript
metalness: 0.0,
roughness: 0.7,
wireframe: false,
flatShading: false,
```

- [ ] **Step 2: Update material creation in NodeRenderer**

In the material creation within `NodeRenderer.update`, add:

```javascript
const mat = new THREE.MeshStandardMaterial({
  color: new THREE.Color(color),
  transparent: opacity < 1,
  opacity,
  emissive: new THREE.Color(emissive),
  emissiveIntensity,
  metalness: params.metalness,
  roughness: params.roughness,
  wireframe: params.wireframe,
  flatShading: params.flatShading,
});
```

- [ ] **Step 3: Add edge material controls**

Add to `this.edgeParams`:

```javascript
metalness: 0.0,
roughness: 0.5,
wireframe: false,
```

Update `EdgeRenderer.update` material similarly.

- [ ] **Step 4: Add GUI controls**

In `buildNodeFolder`:

```javascript
f.add(this.app.nodeParams, 'metalness', 0, 1).name('Metalness').onChange(() => this.app.updateVisuals());
f.add(this.app.nodeParams, 'roughness', 0, 1).name('Roughness').onChange(() => this.app.updateVisuals());
f.add(this.app.nodeParams, 'wireframe').name('Wireframe').onChange(() => this.app.updateVisuals());
f.add(this.app.nodeParams, 'flatShading').name('Flat Shading').onChange(() => this.app.updateVisuals());
```

In `buildEdgeFolder`:

```javascript
f.add(this.app.edgeParams, 'wireframe').name('Wireframe').onChange(() => this.app.updateVisuals());
```

- [ ] **Step 5: Verify in browser**

- Set metalness=1, roughness=0 → shiny chrome nodes
- Toggle wireframe → all nodes show as wireframe outlines
- Toggle flat shading → faceted look on spheres

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add metalness, roughness, wireframe, flat-shading material controls"
```

---

### Task 4: Node Animation (Pulse & Spin)

**Depends on:** Task 3 (uses multi-mesh from Task 2)

**Files:**
- Modify: `index.html` → `App` constructor (`this.nodeParams`)
- Modify: `index.html` → `NodeRenderer` (add `tick()` method)
- Modify: `index.html` → `NodeRenderer.update()` (store `_lastGraph`, `_maxDepth`, `_lastParams`)
- Modify: `index.html` → `GUIManager.buildNodeFolder()`
- Modify: `index.html` → `App.animate()` loop

- [ ] **Step 1: Add animation params**

Add to `this.nodeParams`:

```javascript
animate: false,
pulseSpeed: 1.0,
pulseAmount: 0.2,
spinSpeed: 0.0,
```

- [ ] **Step 2: Add tick method to NodeRenderer**

**Important:** Must handle the multi-mesh array from Task 2. Iterate `this.meshes` and `this.meshNodeIds` in parallel.

```javascript
tick(time, params) {
  if (this.meshes.length === 0 || !params.animate) return;
  const { pulseSpeed, pulseAmount, spinSpeed, size } = params;

  for (let m = 0; m < this.meshes.length; m++) {
    const mesh = this.meshes[m];
    const nodeIdList = this.meshNodeIds[m];

    for (let i = 0; i < nodeIdList.length; i++) {
      const node = this._lastGraph.getNode(nodeIdList[i]);
      if (!node) continue;

      const depthRatio = this._maxDepth > 0 ? (node.metadata.depth || 0) / this._maxDepth : 0;
      const baseScale = node.metadata.type === 'leaf' ? 0.5 : 1 - depthRatio * 0.4;

      // Pulse: sine wave offset by depth so it ripples through the tree
      const pulse = 1 + Math.sin(time * pulseSpeed * 3 + depthRatio * Math.PI * 2) * pulseAmount;

      this.dummy.position.copy(node.position);
      this.dummy.scale.setScalar(size * Math.max(0.2, baseScale) * pulse);

      if (spinSpeed > 0) {
        this.dummy.rotation.y = time * spinSpeed + i * 0.1;
      } else {
        this.dummy.rotation.set(0, 0, 0);
      }

      this.dummy.updateMatrix();
      mesh.setMatrixAt(i, this.dummy.matrix);
    }
    mesh.instanceMatrix.needsUpdate = true;
  }
}
```

Store `this._lastGraph` and `this._maxDepth` during `update()` so `tick()` can access them without re-reading params.

- [ ] **Step 3: Call tick from App.animate**

In the `animate` method:

```javascript
animate() {
  requestAnimationFrame(() => this.animate());
  this.controls.update();
  const time = performance.now() * 0.001;
  this.nodeRenderer.tick(time, this.nodeParams);
  this.composer.render();
}
```

- [ ] **Step 4: Add GUI controls**

```javascript
f.add(this.app.nodeParams, 'animate').name('Animate');
f.add(this.app.nodeParams, 'pulseSpeed', 0.1, 5).name('Pulse Speed');
f.add(this.app.nodeParams, 'pulseAmount', 0, 0.5).name('Pulse Amount');
f.add(this.app.nodeParams, 'spinSpeed', 0, 3).name('Spin Speed');
```

- [ ] **Step 5: Verify in browser**

Toggle "Animate" on. Nodes should gently pulse in a ripple pattern from root to leaves. Increase spin speed — nodes should rotate individually.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add node pulse and spin animation with depth-based ripple"
```

---

### Task 5: 3D Environment Enhancements (Shadows, Auto-Orbit)

**Depends on:** Task 3

**Files:**
- Modify: `index.html` → `App` constructor (`this.envParams`, renderer setup, lighting setup)
- Modify: `index.html` → `App` constructor (add ground plane mesh after grid)
- Modify: `index.html` → `NodeRenderer.update()` (set `castShadow` on meshes)
- Modify: `index.html` → `GUIManager.buildEnvironmentFolder()`
- Modify: `index.html` → `App.animate()` loop

- [ ] **Step 1: Add environment params**

Add to `this.envParams`:

```javascript
shadows: false,
autoOrbit: false,
autoOrbitSpeed: 0.3,
groundReflection: false,
```

- [ ] **Step 2: Enable shadow support**

In the App constructor, after renderer setup:

```javascript
this.renderer.shadowMap.enabled = false; // toggled by param
this.renderer.shadowMap.type = THREE.PCFSoftShadowMap;
this.dirLight.castShadow = true;
this.dirLight.shadow.mapSize.set(1024, 1024);
this.dirLight.shadow.camera.near = 0.5;
this.dirLight.shadow.camera.far = 100;
this.dirLight.shadow.camera.left = -20;
this.dirLight.shadow.camera.right = 20;
this.dirLight.shadow.camera.top = 20;
this.dirLight.shadow.camera.bottom = -20;
```

Add a ground plane for shadow receiving:

```javascript
const groundGeo = new THREE.PlaneGeometry(80, 80);
const groundMat = new THREE.ShadowMaterial({ opacity: 0.15 });
this.groundPlane = new THREE.Mesh(groundGeo, groundMat);
this.groundPlane.rotation.x = -Math.PI / 2;
this.groundPlane.receiveShadow = true;
this.groundPlane.visible = false;
this.scene.add(this.groundPlane);
```

In `NodeRenderer.update`, set `this.mesh.castShadow = true;`.

- [ ] **Step 3: Auto-orbit in animate loop**

```javascript
if (this.envParams.autoOrbit) {
  const speed = this.envParams.autoOrbitSpeed;
  this.controls.autoRotate = true;
  this.controls.autoRotateSpeed = speed * 4;
} else {
  this.controls.autoRotate = false;
}
```

- [ ] **Step 4: Add GUI controls**

In `buildEnvironmentFolder`:

```javascript
f.add(this.app.envParams, 'shadows').name('Shadows').onChange(v => {
  this.app.renderer.shadowMap.enabled = v;
  this.app.groundPlane.visible = v;
  // Shadow toggle requires material recompile on all scene materials
  this.app.scene.traverse(obj => {
    if (obj.material) obj.material.needsUpdate = true;
  });
  this.app.updateVisuals();
});
f.add(this.app.envParams, 'autoOrbit').name('Auto Orbit');
f.add(this.app.envParams, 'autoOrbitSpeed', 0.1, 3).name('Orbit Speed');
```

- [ ] **Step 5: Verify in browser**

- Toggle shadows on → soft shadows on ground plane beneath tree
- Toggle auto-orbit → camera slowly rotates around the scene
- Adjust orbit speed → rotation speeds up/down

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "feat: add shadow casting, ground plane, and auto-orbit camera"
```

---

### Task 6: Edge Enhancements (Dashed Lines, Flow Animation)

**Depends on:** Task 3

**Files:**
- Modify: `index.html` → `App` constructor (`this.edgeParams`)
- Modify: `index.html` → `EdgeRenderer.update()` (dashed material + Line objects)
- Modify: `index.html` → `EdgeRenderer` (add `tick()` method)
- Modify: `index.html` → `GUIManager.buildEdgeFolder()`
- Modify: `index.html` → `App.animate()` loop

- [ ] **Step 1: Add edge params**

Add to `this.edgeParams`:

```javascript
style: 'solid',        // 'solid' | 'dashed'
dashScale: 1.0,
flowAnimate: false,
flowSpeed: 1.0,
```

- [ ] **Step 2: Add dashed material option to EdgeRenderer**

In `EdgeRenderer.update`, choose material based on style:

```javascript
let mat;
if (style === 'dashed') {
  mat = new THREE.LineDashedMaterial({
    color: new THREE.Color(color),
    transparent: opacity < 1,
    opacity,
    dashSize: 0.1 * dashScale,
    gapSize: 0.05 * dashScale,
  });
  // For dashed, use Line instead of Mesh+TubeGeometry
} else {
  mat = new THREE.MeshStandardMaterial({
    color: new THREE.Color(color),
    transparent: opacity < 1,
    opacity,
    metalness: params.metalness || 0,
    roughness: params.roughness || 0.5,
    wireframe: params.wireframe || false,
  });
}
```

When `style === 'dashed'`, use `THREE.Line` with `BufferGeometry` from curve points instead of `TubeGeometry`. This also improves performance since lines are cheaper than tubes.

**Critical:** After creating each `THREE.Line`, call `line.computeLineDistances()`. Without this, `LineDashedMaterial` renders invisible/solid — dashes require distance data in the geometry.

```javascript
const points = curve.getPoints(segments);
const lineGeo = new THREE.BufferGeometry().setFromPoints(points);
const line = new THREE.Line(lineGeo, mat);
line.computeLineDistances(); // REQUIRED for dashes to render
line.userData = { edge };
this.group.add(line);
```

- [ ] **Step 3: Add flow animation via dashOffset**

Add a `tick` method to `EdgeRenderer`:

```javascript
tick(time, params) {
  if (!params.flowAnimate) return;
  this.group.children.forEach(child => {
    if (child.material.dashSize !== undefined) {
      child.material.dashOffset = -time * params.flowSpeed;
    }
  });
}
```

For solid tubes, flow animation applies a UV offset if the material uses a texture. For simplicity in v1, flow animation only works with dashed style.

- [ ] **Step 4: Add GUI controls**

In `buildEdgeFolder`:

```javascript
f.add(this.app.edgeParams, 'style', ['solid', 'dashed']).name('Style').onChange(() => this.app.updateVisuals());
f.add(this.app.edgeParams, 'dashScale', 0.2, 3).name('Dash Scale').onChange(() => this.app.updateVisuals());
f.add(this.app.edgeParams, 'flowAnimate').name('Flow Animate');
f.add(this.app.edgeParams, 'flowSpeed', 0.1, 5).name('Flow Speed');
```

- [ ] **Step 5: Call EdgeRenderer.tick from App.animate**

```javascript
this.edgeRenderer.tick(time, this.edgeParams);
```

- [ ] **Step 6: Verify in browser**

- Switch to "dashed" style → edges render as dashed lines
- Toggle "Flow Animate" on → dashes move along edges like flowing data
- Adjust dash scale and flow speed

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat: add dashed edge style with animated flow effect"
```

---

### Task 7: Size-by-Depth & Randomized Scale

**Depends on:** Task 4 (shares scale calculation with `tick()`)

**Files:**
- Modify: `index.html` → `App` constructor (`this.nodeParams`)
- Modify: `index.html` → `NodeRenderer.update()` (scale calculation in instance loop)
- Modify: `index.html` → `NodeRenderer.tick()` (must use same scale formula)
- Modify: `index.html` → `GUIManager.buildNodeFolder()`

- [ ] **Step 1: Add scale params**

Add to `this.nodeParams`:

```javascript
sizeByDepth: true,      // already partially implemented via the depthRatio scale
sizeVariation: 0.0,     // random size jitter (0 = uniform, 1 = very varied)
minSize: 0.3,           // minimum scale multiplier
```

- [ ] **Step 2: Update NodeRenderer scale calculation**

In the instance setup loop:

```javascript
const depthScale = params.sizeByDepth
  ? Math.max(params.minSize, 1 - depthRatio * (1 - params.minSize))
  : 1;
const variation = 1 + (seededRandom() - 0.5) * params.sizeVariation;
const finalScale = size * depthScale * variation;
this.dummy.scale.setScalar(Math.max(0.05, finalScale));
```

Use the same seeded random from EdgeRenderer pattern for consistent results.

- [ ] **Step 3: Add GUI controls**

```javascript
f.add(this.app.nodeParams, 'sizeByDepth').name('Size by Depth').onChange(() => this.app.updateVisuals());
f.add(this.app.nodeParams, 'sizeVariation', 0, 1).name('Size Variation').onChange(() => this.app.updateVisuals());
f.add(this.app.nodeParams, 'minSize', 0.1, 1).name('Min Size').onChange(() => this.app.updateVisuals());
```

- [ ] **Step 4: Verify in browser**

- sizeByDepth on → nodes shrink toward leaves
- sizeVariation=0.5 → organic, varied node sizes
- minSize controls the floor

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add size-by-depth curve and random size variation controls"
```

---

## Summary

| Task | What it adds | New controls |
|------|-------------|--------------|
| 1 | 9 new geometries (12 total) | Shape dropdown expanded |
| 2 | Shape-by-type mapping | shapeByType toggle, root/branch/leaf shape pickers |
| 3 | Advanced materials | metalness, roughness, wireframe, flat shading |
| 4 | Node animation | pulse + spin with depth ripple |
| 5 | 3D environment | shadows, ground plane, auto-orbit |
| 6 | Edge enhancements | dashed lines, flow animation |
| 7 | Scale controls | size-by-depth, variation, min size |

**Total new GUI controls:** ~20
**Total node shapes:** 12 (up from 3)
**New 3D features:** shadow casting, auto-orbit, node animation, edge flow animation

**Execution order:** Tasks 1→2→3 are sequential. After Task 3, Tasks 4/5/6 can run in parallel. Task 7 follows Task 4.
