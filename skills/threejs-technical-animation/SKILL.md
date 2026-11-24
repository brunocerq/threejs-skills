---
name: threejs-technical-animation
description: "Technical and product animation: camera paths along curves, easing, a dependency-free scrubbable timeline, exploded views of glTF assemblies and animated section (clipping) planes, on WebGL or with WebGPU and TSL. Use when building guided camera tours, assembly or maintenance sequences, exploded or cutaway views, or any animation that must play, pause, reverse and scrub."
---

# Three.js Technical Animation

Targets three.js r186. Examples use WebGLRenderer; the WebGPU and TSL section uses WebGPURenderer.

Technical animation is evaluated from time, not accumulated per frame: every value is a function of the playhead, so a sequence can pause, reverse, jump to a step and scrub from a slider and always land on the same pose.

## Quick Start

A camera flies along a curve at constant speed while a section plane sweeps up through the part.

```javascript
import * as THREE from "three";

const renderer = new THREE.WebGLRenderer({ antialias: true });
renderer.setPixelRatio(window.devicePixelRatio);
renderer.setSize(window.innerWidth, window.innerHeight);
renderer.localClippingEnabled = true; // material.clippingPlanes is ignored without it
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1d2126);
scene.add(new THREE.HemisphereLight(0xffffff, 0x334455, 3));

const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);

// Section plane: keeps the side the normal points to, here y <= constant
const section = new THREE.Plane(new THREE.Vector3(0, -1, 0), 0);
const part = new THREE.Mesh(
  new THREE.TorusKnotGeometry(0.8, 0.3, 160, 24),
  new THREE.MeshStandardMaterial({
    color: 0x8fa3b8,
    side: THREE.DoubleSide, // shows the inner faces through the cut
    clippingPlanes: [section],
  }),
);
scene.add(part);

const path = new THREE.CatmullRomCurve3(
  [
    new THREE.Vector3(4, 1.5, 0),
    new THREE.Vector3(0, 2.5, 4),
    new THREE.Vector3(-4, 1, 0),
    new THREE.Vector3(0, 0.5, -4),
  ],
  true, // closed loop
);

const easeInOut = (t) => (t < 0.5 ? 4 * t ** 3 : 1 - (-2 * t + 2) ** 3 / 2);
const box = new THREE.Box3().setFromObject(part);
const loop = 10; // seconds per cycle

const timer = new THREE.Timer();
timer.connect(document);

function animate() {
  timer.update();
  const p = (timer.getElapsed() % loop) / loop; // 0..1 playhead

  camera.position.copy(path.getPointAt(p)); // arc length: constant speed
  camera.lookAt(0, 0, 0);

  section.constant = THREE.MathUtils.lerp(box.min.y, box.max.y, easeInOut(p));
  renderer.render(scene, camera);
}

renderer.setAnimationLoop(animate);
```

## Camera Paths Along Curves

### Building the Curve

```javascript
const points = [
  new THREE.Vector3(6, 2, 6),
  new THREE.Vector3(2, 1.2, 3),
  new THREE.Vector3(0, 0.8, 1.5), // close-up on the detail
  new THREE.Vector3(-3, 2, 1),
];

// curveType: "centripetal" (default, no loops or cusps), "chordal" or "catmullrom" (uses tension)
const path = new THREE.CatmullRomCurve3(points, false, "centripetal");

// getPointAt and getTangentAt use an arc-length table of arcLengthDivisions (default 200) samples;
// raise it for long paths with tight bends, before the first query
path.arcLengthDivisions = 1000;

// After editing path.points, rebuild that table or getPointAt keeps the old spacing
points[2].y = 1;
path.updateArcLengths();

// Debug line, evenly spaced by arc length
scene.add(new THREE.Line(new THREE.BufferGeometry().setFromPoints(path.getSpacedPoints(200))));
```

### Constant Speed: getPointAt, not getPoint

`getPoint(t)` moves an equal amount of curve *parameter* per step, so the camera speeds up on long segments and crawls on short ones. `getPointAt(u)` maps `u` through arc length: equal steps of `u` are equal distances.

```javascript
const position = new THREE.Vector3();
const tangent = new THREE.Vector3();

function placeCamera(u) {
  path.getPointAt(u, position);
  path.getTangentAt(u, tangent); // unit direction of travel
  camera.position.copy(position);
  camera.lookAt(position.x + tangent.x, position.y + tangent.y, position.z + tangent.z);
}
```

To spend more time on the close-up, ease `u` (see Easing) rather than moving the control points.

### Where the Camera Looks

Down the tangent for a fly-through (as in `placeCamera`), at a fixed point for an orbit-like tour, or along a second curve for full control: the target travels its own path in sync.

```javascript
const targetPath = new THREE.CatmullRomCurve3([
  new THREE.Vector3(0, 0.5, 0),
  new THREE.Vector3(0, 0.6, 0.4),
  new THREE.Vector3(0.3, 0.6, 0.6), // the detail
  new THREE.Vector3(0, 0.5, 0),
]);
const target = new THREE.Vector3();

function placeCameraWithTarget(u) {
  path.getPointAt(u, camera.position);
  targetPath.getPointAt(u, target);
  camera.lookAt(target);
}
```

For a ride that banks with the track, `path.computeFrenetFrames(segments, closed)` returns per-sample `tangents` and `binormals`: copy the binormal into `camera.up` before `lookAt`.

### Handing Back to OrbitControls

While flying, set `controls.enabled = false` and skip `controls.update()`: update() re-aims the camera at `controls.target` every frame. At the end, `controls.target.copy(lastLookTarget); controls.enabled = true; controls.update();` resumes orbiting from where the tour stopped.

## Easing

An easing maps linear progress `t` in 0..1 to eased progress in 0..1. Ease the *progress*, then interpolate — never ease a delta.

```javascript
export const Ease = {
  linear: (t) => t,
  inCubic: (t) => t ** 3,
  outCubic: (t) => 1 - (1 - t) ** 3,
  inOutCubic: (t) => (t < 0.5 ? 4 * t ** 3 : 1 - (-2 * t + 2) ** 3 / 2),
  inOutSine: (t) => -(Math.cos(Math.PI * t) - 1) / 2,
  outExpo: (t) => (t === 1 ? 1 : 1 - 2 ** (-10 * t)),
  outBack: (t) => 1 + 2.70158 * (t - 1) ** 3 + 1.70158 * (t - 1) ** 2, // overshoots, then settles
};

const v = THREE.MathUtils.lerp(-2, 2, Ease.outCubic(0.5)); // eased value between -2 and 2
```

Built in: `THREE.MathUtils.smoothstep(x, min, max)` and `smootherstep` map x between min and max to an eased 0..1.

Pick by meaning: `inOutCubic`/`inOutSine` for camera moves and parts travelling between rest poses, `outCubic` for parts arriving, `inCubic` for parts leaving, `outBack` for a part snapping into place, `linear` for section sweeps and anything a viewer measures by eye.

`THREE.MathUtils.damp(current, target, lambda, delta)` is the frame-rate-independent follow for *interactive* motion (a camera chasing a moving target). It depends on the previous frame, so it cannot be scrubbed; use the timeline below for authored sequences.

## A Dependency-Free Timeline

Tweens numeric properties on any object (`mesh.position`, a material, a `Plane`, a TSL uniform) and evaluates the whole sequence from the playhead, so `seek()` works in both directions.

```javascript
export class Timeline {
  constructor() {
    this.tweens = [];
    this.calls = [];
    this.labels = {};
    this.duration = 0;
    this.time = 0;
    this.playing = false;
    this.timeScale = 1; // -1 plays backward
    this.onComplete = null;
    this._last = new Map(); // target -> { key: value at the end of its latest tween }
  }

  // at: seconds, a label, or "label+0.5"; defaults to the current end (append)
  to(target, to, { at = this.duration, duration = 1, ease = Ease.inOutCubic, onUpdate } = {}) {
    const start = this._resolve(at);
    let last = this._last.get(target);
    if (!last) this._last.set(target, (last = {}));
    const from = {};
    const first = {};
    for (const key in to) {
      first[key] = !(key in last); // the first tween of a key owns its rest value
      from[key] = first[key] ? target[key] : last[key];
      last[key] = to[key];
    }
    this.tweens.push({ target, from, to, first, start, duration, ease, onUpdate });
    this.tweens.sort((a, b) => a.start - b.start); // stable: same-start tweens keep insertion order
    this.duration = Math.max(this.duration, start + duration);
    return this;
  }

  label(name, at = this.duration) {
    this.labels[name] = this._resolve(at);
    return this;
  }

  call(fn, at = this.duration) {
    this.calls.push({ fn, time: this._resolve(at) });
    this.duration = Math.max(this.duration, this._resolve(at));
    return this;
  }

  seek(time) {
    this.time = THREE.MathUtils.clamp(time, 0, this.duration);
    for (const tw of this.tweens) {
      if (this.time < tw.start) {
        // not started: only a key's first tween writes, so earlier tweens keep their result
        let wrote = false;
        for (const key in tw.to) {
          if (tw.first[key]) {
            tw.target[key] = tw.from[key];
            wrote = true;
          }
        }
        if (wrote) tw.onUpdate?.(0);
        continue;
      }
      const p = tw.duration > 0 ? Math.min((this.time - tw.start) / tw.duration, 1) : 1;
      const e = tw.ease(p);
      for (const key in tw.to) tw.target[key] = tw.from[key] + (tw.to[key] - tw.from[key]) * e;
      tw.onUpdate?.(e);
    }
    return this;
  }

  play() {
    if (this.timeScale > 0 && this.time >= this.duration) this.seek(0);
    if (this.timeScale < 0 && this.time <= 0) this.seek(this.duration);
    this.playing = true;
    return this;
  }

  pause() {
    this.playing = false;
    return this;
  }

  update(delta) {
    if (!this.playing) return;
    const prev = this.time;
    this.seek(this.time + delta * this.timeScale);
    for (const c of this.calls) {
      // callbacks fire only while playing forward, never while scrubbing
      if (c.time <= this.time && (c.time > prev || (prev === 0 && c.time === 0))) c.fn();
    }
    if (this.timeScale >= 0 ? this.time >= this.duration : this.time <= 0) {
      this.playing = false;
      this.onComplete?.();
    }
  }

  _resolve(at) {
    if (typeof at === "number") return at;
    const [name, offset = "0"] = at.split("+");
    if (!(name in this.labels)) throw new Error(`Unknown timeline label: ${name}`);
    return this.labels[name] + Number(offset);
  }
}
```

Rules that keep it deterministic: add the tweens of one property in time order and do not overlap them; the `from` of each later tween is the `to` of the previous one, and the first tween's `from` is the value when `to()` was called.

### Sequencing and Scrubbing

```javascript
const lid = scene.getObjectByName("Lid");
const rig = { u: 0 }; // camera progress along path

const tl = new Timeline()
  .to(rig, { u: 0.4 }, { duration: 3, ease: Ease.inOutSine, onUpdate: () => placeCamera(rig.u) })
  .label("open")
  .to(lid.position, { y: lid.position.y + 0.3 }, { duration: 0.8, ease: Ease.outCubic })
  .to(lid.rotation, { x: -1.2 }, { at: "open+0.4", duration: 1 }) // overlaps the lift
  .to(rig, { u: 1 }, { duration: 4, ease: Ease.inOutSine, onUpdate: () => placeCamera(rig.u) })
  .call(() => console.log("tour done"));

tl.seek(0); // apply the first frame before the first render
tl.play(); // then in the render loop, after timer.update(): tl.update(timer.getDelta());

// A range input as a scrubber: <input id="scrub" type="range" min="0" max="1" step="0.001">
document.querySelector("#scrub").addEventListener("input", (e) => {
  tl.pause().seek(Number(e.target.value) * tl.duration);
});
// Jump to a step: tl.pause().seek(tl.labels.open); reverse: tl.timeScale = -1; tl.play();
```

Colors: tween `material.color` with `{ r, g, b }` in linear values (`new THREE.Color(0xff6600)` already converts the hex to linear). Opacity: set `transparent: true` on the material before the first render.

## Exploded Views

An exploded view moves every part away from the assembly centre along the line from the centre to the part. Compute the directions once, at rest, and in each part's *parent* space: CAD glTFs often sit under rotated nodes or a 0.001 scale (millimetres to metres).

### Choosing the Parts

```javascript
import { GLTFLoader } from "three/addons/loaders/GLTFLoader.js";

const gltf = await new GLTFLoader().loadAsync("gearbox.glb");
const model = gltf.scene;
scene.add(model);

// Exporters wrap the assembly in single-child nodes; descend to the first node that branches.
// Its children are the parts: sub-assemblies move as one, recurse for a second level.
let assembly = model;
while (assembly.children.length === 1) assembly = assembly.children[0];
```

### Directions and the Explode Function

```javascript
function prepareExplode(root, parts) {
  root.updateWorldMatrix(true, true);
  const center = new THREE.Box3().setFromObject(root).getCenter(new THREE.Vector3());
  return parts.map((part) => {
    const partCenter = new THREE.Box3().setFromObject(part).getCenter(new THREE.Vector3());
    // World offset centre -> part, converted into the parent's space (rotation and scale)
    const a = part.parent.worldToLocal(partCenter.clone());
    const b = part.parent.worldToLocal(partCenter.clone().multiplyScalar(2).sub(center));
    return { part, home: part.position.clone(), offset: b.sub(a) };
  });
}

// amount 0 = assembled; 1 = every part twice as far from the centre as at rest
function setExplode(entries, amount) {
  for (const { part, home, offset } of entries) {
    part.position.copy(home).addScaledVector(offset, amount);
  }
}

const entries = prepareExplode(assembly, assembly.children);
setExplode(entries, 0.8);
```

Scaling each offset by the part's own distance keeps the layout readable: neighbours separate in proportion and a part at the exact centre stays put (give it a manual `offset` such as `new THREE.Vector3(0, 0.2, 0)` if it should lift).

### Stacked Parts Along an Axis

A shaft with bearings, spacers and a housing reads better exploded along the shaft:

```javascript
function prepareAxialExplode(parts, axisWorld, gap) {
  const centers = parts.map((p) => new THREE.Box3().setFromObject(p).getCenter(new THREE.Vector3()));
  const order = parts.map((_, i) => i).sort((i, j) => centers[i].dot(axisWorld) - centers[j].dot(axisWorld));
  const mid = (parts.length - 1) / 2;
  return order.map((i, rank) => {
    const part = parts[i];
    const a = part.parent.worldToLocal(centers[i].clone());
    const b = part.parent.worldToLocal(centers[i].clone().addScaledVector(axisWorld, (rank - mid) * gap));
    return { part, home: part.position.clone(), offset: b.sub(a) };
  });
}

const axial = prepareAxialExplode(assembly.children, new THREE.Vector3(1, 0, 0), 0.15); // world units
setExplode(axial, 1); // same setExplode: offset already holds the full distance
```

### Staggered Explode on the Timeline

```javascript
const tl = new Timeline();
entries.forEach((entry, i) => {
  entry.amount = 0; // the entry itself is the tween target
  tl.to(entry, { amount: 1 }, {
    at: i * 0.12, // outer parts first: sort entries by offset length beforehand
    duration: 0.9,
    onUpdate: () => entry.part.position.copy(entry.home).addScaledVector(entry.offset, entry.amount),
  });
});
```

Frame the camera on the *exploded* bounds: `setExplode(entries, 1)`, take `new THREE.Box3().setFromObject(model)`, then `setExplode(entries, 0)`. Raycasts follow the moved parts automatically, since they use world matrices.

## Animated Clipping Planes

### Section Sweep

A `Plane` keeps the half-space its normal points into (`normal · p + constant >= 0`) and discards the rest. Planes are in world space.

```javascript
renderer.localClippingEnabled = true;

// Keeps y <= constant: the cut sweeps up as constant grows
const section = new THREE.Plane(new THREE.Vector3(0, -1, 0), 0);

model.traverse((o) => {
  if (!o.isMesh) return;
  o.material.clippingPlanes = [section]; // the same Plane object everywhere: one value animates all
  o.material.clipShadows = true; // clipped parts stop casting the full shadow
  o.material.side = THREE.DoubleSide; // inner faces show through the cut
});

const box = new THREE.Box3().setFromObject(model);
section.constant = box.max.y; // start uncut: the tween's from value
tl.to(section, { constant: box.min.y }, { duration: 3, ease: Ease.linear }); // a Plane is a valid target

// Debug view of the plane (not clipped itself)
scene.add(new THREE.PlaneHelper(section, 3, 0xffaa00));
```

`renderer.clippingPlanes = [section]` clips everything in the scene, needs no `localClippingEnabled`, and leaves the materials alone.

### Cutaway Corner

`clipIntersection: true` discards a fragment only when *every* plane would discard it, which removes a wedge instead of a half-space:

```javascript
const cutX = new THREE.Plane(new THREE.Vector3(-1, 0, 0), box.max.x); // discards x > constant
const cutZ = new THREE.Plane(new THREE.Vector3(0, 0, -1), box.max.z); // discards z > constant
const center = box.getCenter(new THREE.Vector3());

model.traverse((o) => {
  if (!o.isMesh) return;
  o.material.clippingPlanes = [cutX, cutZ];
  o.material.clipIntersection = true; // removes the x > cx AND z > cz quarter only
});

// Open the corner: slide both planes from the outside in to the centre
tl.to(cutX, { constant: center.x }, { duration: 1.5 });
tl.to(cutZ, { constant: center.z }, { at: 0, duration: 1.5 });
```

A plane that rides on a moving object (a cutting tool) is defined in the object's space and moved to world space each frame: `worldPlane.copy(localPlane).applyMatrix4(cutter.matrixWorld)`, with `worldPlane` in the materials' `clippingPlanes`.

### Capping the Cut

Through a clipped closed mesh the viewer sees the inside. The cheap cap: draw the back faces in a flat cap colour. It reads as a solid section from most angles, but it is not flat and shows depth when the camera looks along the cut.

```javascript
const capMaterial = new THREE.MeshBasicMaterial({
  color: 0xd04040,
  side: THREE.BackSide,
  clippingPlanes: [section],
});
model.traverse((o) => {
  if (!o.isMesh || o.userData.isCap) return;
  o.material.side = THREE.FrontSide;
  const cap = new THREE.Mesh(o.geometry, capMaterial);
  cap.userData.isCap = true;
  o.add(cap); // child at identity: follows the part, explodes with it
});
```

An exact flat cap uses the stencil buffer, as in the official `webgl_clipping_stencil` example. Create the renderer with `stencil: true`; give every mesh two stencil-only copies clipped by the plane (`colorWrite`, `depthWrite` and `depthTest` false, `stencilWrite` true, `AlwaysStencilFunc`; the `BackSide` copy uses `IncrementWrapStencilOp` and the `FrontSide` copy `DecrementWrapStencilOp` for fail, zFail and zPass) at `renderOrder = 1`; then draw a large cap quad at `renderOrder = 1.1` with `stencilFunc: THREE.NotEqualStencilFunc`, `stencilRef: 0`, `ReplaceStencilOp` and `onAfterRender = (r) => r.clearStencil()`; draw the visible parts last (`renderOrder = 6`). Each frame, after the plane moves, place the quad on it:

```javascript
section.coplanarPoint(capQuad.position);
capQuad.lookAt(capQuad.position.clone().sub(section.normal)); // faces the removed side
```

With several planes, each plane gets its own stencil copies and cap quad, and each quad is clipped by the *other* planes.

## WebGPU and TSL

Curves, easing, the timeline and exploded views are plain CPU math and run unchanged with `import * as THREE from "three/webgpu"`. Clipping differs: `WebGPURenderer` ignores `material.clippingPlanes` and `localClippingEnabled`. Clipping planes belong to a `ClippingGroup`, which clips every descendant; the planes stay in world space and animate the same way.

```javascript
import * as THREE from "three/webgpu";
import { abs, color, positionWorld, smoothstep, uniform } from "three/tsl";

const renderer = new THREE.WebGPURenderer({ antialias: true });
await renderer.init(); // before any other use of the renderer
renderer.setSize(window.innerWidth, window.innerHeight);
document.body.appendChild(renderer.domElement);

const scene = new THREE.Scene();
scene.background = new THREE.Color(0x1d2126); // the WebGPU canvas is transparent by default
scene.add(new THREE.HemisphereLight(0xffffff, 0x334455, 2));
const camera = new THREE.PerspectiveCamera(50, window.innerWidth / window.innerHeight, 0.1, 100);
camera.position.set(3, 2, 3);
camera.lookAt(0, 0, 0);

const section = new THREE.Plane(new THREE.Vector3(0, -1, 0), 0); // keeps y <= constant
const clipping = new THREE.ClippingGroup();
clipping.clippingPlanes = [section];
clipping.clipShadows = true;
// clipping.clipIntersection = true; // the cutaway-wedge mode, on the group
scene.add(clipping);

// Glow band on the cut edge: one uniform drives the plane and the shader together
const cutHeight = uniform(0);
const material = new THREE.MeshStandardNodeMaterial({ color: 0x8fa3b8, metalness: 0.6, roughness: 0.35 });
material.side = THREE.DoubleSide;
const band = smoothstep(0, 0.04, abs(positionWorld.y.sub(cutHeight))).oneMinus(); // 1 at the cut
material.emissiveNode = color(0xff7a1a).mul(band).mul(2);

const part = new THREE.Mesh(new THREE.TorusKnotGeometry(0.8, 0.3, 160, 24), material);
clipping.add(part); // only descendants of the group are clipped

const box = new THREE.Box3().setFromObject(part);
const timer = new THREE.Timer();
timer.connect(document);

renderer.setAnimationLoop(() => {
  timer.update();
  const p = (timer.getElapsed() % 4) / 4;
  cutHeight.value = THREE.MathUtils.lerp(box.min.y, box.max.y, p);
  section.constant = cutHeight.value;
  renderer.render(scene, camera);
});
```

The timeline can tween a uniform directly: `tl.to(cutHeight, { value: 1 }, { duration: 2 })`. Nested `ClippingGroup`s combine their planes, so a global section group can hold a cutaway group for one sub-assembly. The back-face cap works under WebGPU unchanged (keep the cap meshes inside the same group); the stencil cap is written and tested for WebGLRenderer only.

## Common Mistakes

- `camera.position.copy(path.getPoint(t))` → `camera.position.copy(path.getPointAt(u))`: getPoint steps the curve parameter, so speed changes with segment length; getPointAt steps arc length.
- `camera.lookAt(path.getPointAt(u + 0.01))` → `camera.lookAt(pos.clone().add(path.getTangentAt(u)))`: on an open curve u + 0.01 runs past 1 at the end of the path, and the look-ahead jitters on uneven segments.
- `path.points[2].y = 1; path.getPointAt(u)` → `path.points[2].y = 1; path.updateArcLengths(); path.getPointAt(u)`: the arc-length table is cached, so the spacing stays that of the old curve.
- `controls.update()` in the render loop during a camera tour → `controls.enabled = false` and no update until the tour ends: update() re-aims the camera at controls.target every frame.
- `mesh.position.x += speed * delta` for an authored sequence → `tl.to(mesh.position, { x: 2 }, { duration: 1 })`: accumulated motion cannot be paused at a step, reversed or scrubbed, and drifts with frame rate.
- `value += Ease.inOutCubic(delta)` → `value = lerp(from, to, Ease.inOutCubic(elapsed / duration))`: an easing maps normalized progress, not a frame delta.
- `part.position.add(worldDirection.multiplyScalar(d))` → convert the offset into `part.parent` space with `worldToLocal` first: under a rotated or 0.001-scaled CAD node the part moves the wrong way or a thousand times too far.
- Recomputing explode directions from `Box3.setFromObject` every frame → compute them once at rest: the exploded bounds shift the centre and the parts drift.
- `material.clippingPlanes = [plane]` with no other change → also set `renderer.localClippingEnabled = true`: material planes are ignored without it.
- `material.clippingPlanes = [new THREE.Plane(n, h)]` every frame → create the Plane once and animate `plane.constant`: a new array each frame is wasted work, and a changing plane count recompiles the shader.
- `new THREE.Plane(new THREE.Vector3(0, 1, 0), h)` to remove everything above h → `new THREE.Plane(new THREE.Vector3(0, -1, 0), h)`: the plane keeps the side its normal points into; the first one keeps y >= -h.
- Stencil caps on a default renderer → `new THREE.WebGLRenderer({ stencil: true })`: WebGLRenderer creates no stencil buffer unless asked, and the cap never draws.
- `material.clippingPlanes = [section]` under WebGPURenderer → put the objects in a `ClippingGroup` with `clippingPlanes = [section]`: node materials ignore the material property.
- `smoothstep(0.04, 0, d)` in TSL → `smoothstep(0, 0.04, d).oneMinus()`: WGSL leaves smoothstep undefined when the lower edge is above the upper one.

## See Also

- `threejs-animation` - keyframe clips and the AnimationMixer, for animation authored in a DCC tool
- `threejs-interaction` - OrbitControls and raycasting, for picking a part to explode or section
- `threejs-loaders` - loading glTF assemblies
- `threejs-materials` - clipping, side and stencil properties of materials
- `threejs-fundamentals` - Timer, the render loop and camera setup
