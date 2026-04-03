# Three.js — 3D Scene Mode Reference

## Overview

Three.js is the standard JavaScript library for 3D graphics in the browser. The 3D template
(`assets/template-3d.html`) uses Three.js with ES module imports via importmap. In `"scene"`
render mode, you build a scene graph of meshes, lights, and cameras — the template handles
the render loop, OrbitControls, and sidebar/seed infrastructure.

**When to use scene mode vs shader mode:**
- **Scene mode** — 3D objects, particle systems, instanced geometry, generative sculptures,
  architectural forms, physical simulations, anything with distinct geometry
- **Shader mode** — Fullscreen GPU effects: raymarching, fractals, volumetric rendering,
  reaction-diffusion on GPU, Shadertoy-style pieces (see `shaders-glsl.md`)

## Template Contract

Implement two functions:

```javascript
function sceneSetup(THREE, scene, camera, renderer, params, seed) {
  // scene: THREE.Scene — add objects here
  // camera: PerspectiveCamera at (0, 0, 5) — reposition as needed
  // renderer: WebGLRenderer — configure shadows, tone mapping, etc.
  // params: current parameter values
  // seed: current seed number
  // Return: optional state object passed to sceneAnimate
}

function sceneAnimate(THREE, scene, camera, state, params, time, delta) {
  // state: whatever sceneSetup returned
  // time: elapsed seconds (float)
  // delta: seconds since last frame
}
```

The template provides:
- `OrbitControls` (mouse drag to rotate, scroll to zoom, right-drag to pan)
- `preserveDrawingBuffer: true` for PNG export
- `ACESFilmicToneMapping` and `SRGBColorSpace` by default
- `window.seededRandom()` — a mulberry32 PRNG seeded from the current seed

## Geometry

### Built-in Geometries
```javascript
// Primitives
new THREE.BoxGeometry(w, h, d, wSegs, hSegs, dSegs)
new THREE.SphereGeometry(radius, widthSegs, heightSegs)
new THREE.CylinderGeometry(radiusTop, radiusBot, height, radialSegs)
new THREE.TorusGeometry(radius, tube, radialSegs, tubularSegs)
new THREE.TorusKnotGeometry(radius, tube, tubularSegs, radialSegs, p, q)
new THREE.PlaneGeometry(w, h, wSegs, hSegs)
new THREE.IcosahedronGeometry(radius, detail)  // great for organic subdivision
new THREE.OctahedronGeometry(radius, detail)
new THREE.TetrahedronGeometry(radius, detail)
new THREE.ConeGeometry(radius, height, radialSegs)
new THREE.RingGeometry(innerR, outerR, thetaSegs)
new THREE.DodecahedronGeometry(radius, detail)
```

### BufferGeometry (custom meshes)
```javascript
const geo = new THREE.BufferGeometry();
const vertices = new Float32Array([
  -1, -1, 0,   1, -1, 0,   0, 1, 0  // triangle
]);
geo.setAttribute('position', new THREE.BufferAttribute(vertices, 3));
geo.computeVertexNormals();
```

### Instanced Meshes (high-performance many-objects)
```javascript
const geo = new THREE.IcosahedronGeometry(0.1, 1);
const mat = new THREE.MeshStandardMaterial({ color: 0xffffff });
const count = 10000;
const mesh = new THREE.InstancedMesh(geo, mat, count);

const dummy = new THREE.Object3D();
const color = new THREE.Color();

for (let i = 0; i < count; i++) {
  dummy.position.set(
    seededRandom() * 10 - 5,
    seededRandom() * 10 - 5,
    seededRandom() * 10 - 5
  );
  dummy.scale.setScalar(0.5 + seededRandom() * 0.5);
  dummy.updateMatrix();
  mesh.setMatrixAt(i, dummy.matrix);
  color.setHSL(seededRandom(), 0.7, 0.6);
  mesh.setColorAt(i, color);
}
mesh.instanceMatrix.needsUpdate = true;
mesh.instanceColor.needsUpdate = true;
scene.add(mesh);
```

To animate instanced meshes, update matrices per frame and set `mesh.instanceMatrix.needsUpdate = true`.

### Points (particle systems)
```javascript
const count = 50000;
const positions = new Float32Array(count * 3);
const colors = new Float32Array(count * 3);

for (let i = 0; i < count; i++) {
  positions[i * 3]     = (seededRandom() - 0.5) * 20;
  positions[i * 3 + 1] = (seededRandom() - 0.5) * 20;
  positions[i * 3 + 2] = (seededRandom() - 0.5) * 20;

  const c = new THREE.Color().setHSL(seededRandom(), 0.8, 0.6);
  colors[i * 3]     = c.r;
  colors[i * 3 + 1] = c.g;
  colors[i * 3 + 2] = c.b;
}

const geo = new THREE.BufferGeometry();
geo.setAttribute('position', new THREE.BufferAttribute(positions, 3));
geo.setAttribute('color', new THREE.BufferAttribute(colors, 3));

const mat = new THREE.PointsMaterial({
  size: 0.05,
  vertexColors: true,
  transparent: true,
  opacity: 0.8,
  blending: THREE.AdditiveBlending,
  depthWrite: false,
});

const points = new THREE.Points(geo, mat);
scene.add(points);
```

To animate, modify `geo.attributes.position.array[...]` and set `geo.attributes.position.needsUpdate = true`.

## Materials

```javascript
// Physically-based (responds to lights)
new THREE.MeshStandardMaterial({
  color: 0x4488ff,
  metalness: 0.5,
  roughness: 0.3,
  emissive: 0x000000,
  emissiveIntensity: 0.0,
  wireframe: false,
  transparent: true,
  opacity: 0.9,
  side: THREE.DoubleSide,
})

// Unlit flat color (no lights needed)
new THREE.MeshBasicMaterial({ color: 0xff0000, wireframe: true })

// Normal-colored (debug / aesthetic)
new THREE.MeshNormalMaterial()

// Phong (cheaper shading)
new THREE.MeshPhongMaterial({ color: 0x00ff00, shininess: 100 })

// Line material
new THREE.LineBasicMaterial({ color: 0xffffff, linewidth: 1 })
```

## Lighting

```javascript
// Ambient (base illumination)
scene.add(new THREE.AmbientLight(0x404040, 0.5));

// Directional (sun-like)
const dirLight = new THREE.DirectionalLight(0xffffff, 1.0);
dirLight.position.set(5, 10, 7);
scene.add(dirLight);

// Point (omni-directional)
const pointLight = new THREE.PointLight(0xff8844, 1.0, 50);
pointLight.position.set(0, 3, 0);
scene.add(pointLight);

// Hemisphere (sky + ground gradient)
scene.add(new THREE.HemisphereLight(0x88ccff, 0x443322, 0.6));

// Spot
const spot = new THREE.SpotLight(0xffffff, 1.0, 30, Math.PI / 6, 0.5, 1);
spot.position.set(0, 10, 0);
scene.add(spot);
```

### Shadows
```javascript
renderer.shadowMap.enabled = true;
renderer.shadowMap.type = THREE.PCFSoftShadowMap;

dirLight.castShadow = true;
dirLight.shadow.mapSize.width = 2048;
dirLight.shadow.mapSize.height = 2048;

mesh.castShadow = true;
groundPlane.receiveShadow = true;
```

## Camera

The template provides a PerspectiveCamera at `(0, 0, 5)`. Adjust in `sceneSetup`:

```javascript
camera.position.set(0, 5, 10);
camera.lookAt(0, 0, 0);
camera.fov = 45;
camera.updateProjectionMatrix();
```

For animated camera (orbiting, fly-through):
```javascript
// In sceneAnimate:
camera.position.x = Math.cos(time * 0.3) * radius;
camera.position.z = Math.sin(time * 0.3) * radius;
camera.lookAt(0, 0, 0);
```

## Scene Background

```javascript
scene.background = new THREE.Color(0x000000);

// Fog (depth atmosphere)
scene.fog = new THREE.Fog(0x000000, 5, 30);           // linear
scene.fog = new THREE.FogExp2(0x000000, 0.05);         // exponential
```

## Helpers (useful during development)

```javascript
scene.add(new THREE.AxesHelper(5));
scene.add(new THREE.GridHelper(10, 10));
scene.add(new THREE.PointLightHelper(pointLight, 0.5));
```

## Common 3D Generative Patterns

### Generative Sculpture (deformed mesh)
```javascript
const geo = new THREE.IcosahedronGeometry(2, 5);
const pos = geo.attributes.position;
for (let i = 0; i < pos.count; i++) {
  const v = new THREE.Vector3().fromBufferAttribute(pos, i);
  const noise = perlin3D(v.x * 0.5, v.y * 0.5, v.z * 0.5);  // your noise fn
  v.normalize().multiplyScalar(2 + noise * 0.5);
  pos.setXYZ(i, v.x, v.y, v.z);
}
geo.computeVertexNormals();
```

### 3D Lattice / Grid
```javascript
const size = 10, step = 1;
for (let x = -size; x <= size; x += step) {
  for (let y = -size; y <= size; y += step) {
    for (let z = -size; z <= size; z += step) {
      // Place object at (x, y, z) based on noise or rule
    }
  }
}
```

### Line-Based Structures (attractors, paths)
```javascript
const points = [];
let x = 0.1, y = 0, z = 0;
for (let i = 0; i < 100000; i++) {
  // Lorenz attractor step
  const dx = 10 * (y - x) * 0.001;
  const dy = (x * (28 - z) - y) * 0.001;
  const dz = (x * y - 2.667 * z) * 0.001;
  x += dx; y += dy; z += dz;
  points.push(new THREE.Vector3(x, y, z));
}
const geo = new THREE.BufferGeometry().setFromPoints(points);
const mat = new THREE.LineBasicMaterial({ color: 0x88ccff });
scene.add(new THREE.Line(geo, mat));
```

### Tubes from Curves
```javascript
const curvePoints = [/* THREE.Vector3 array */];
const curve = new THREE.CatmullRomCurve3(curvePoints);
const tubeGeo = new THREE.TubeGeometry(curve, 200, 0.05, 8, false);
const tubeMat = new THREE.MeshStandardMaterial({ color: 0xff4444 });
scene.add(new THREE.Mesh(tubeGeo, tubeMat));
```

## Noise in Three.js

Three.js has no built-in noise. Include a noise implementation in your sketch code.
A compact 3D Perlin noise (copy into Section 3):

```javascript
// Compact simplex-style 3D noise — paste this into your sketch
// Returns values in approximately [-1, 1]
function noise3D(x, y, z) {
  // Use a hash-based approach with seededRandom for determinism
  // Or include a full simplex noise implementation
}
```

For production quality, inline a simplex noise implementation or use the `u_time` dimension
of 2D noise fields. The shader mode (`shaders-glsl.md`) has built-in GLSL noise functions
that are more performant for heavy noise use.

## Post-Processing

Three.js supports multi-pass post-processing via the EffectComposer:

```javascript
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';

const composer = new EffectComposer(renderer);
composer.addPass(new RenderPass(scene, camera));
composer.addPass(new UnrealBloomPass(
  new THREE.Vector2(window.innerWidth, window.innerHeight),
  1.5,   // strength
  0.4,   // radius
  0.85   // threshold
));

// In sceneAnimate, replace renderer.render with:
// composer.render();  — but since the template calls renderer.render,
// store the composer on state and override in animate:
```

**Note:** When using EffectComposer, the template's automatic `renderer.render(scene, camera)`
still runs. To use the composer instead, set `renderer.autoClear = false` in sceneSetup and
call `composer.render()` in sceneAnimate, then set a flag to skip the template's render.
Or simply render your scene entirely through the composer by storing it on the returned state
and checking for it.

Available post-processing passes (via importmap `three/addons/`):
- `UnrealBloomPass` — glow / bloom effect
- `BokehPass` — depth of field
- `FilmPass` — film grain + scanlines
- `GlitchPass` — screen distortion
- `SSAOPass` — screen-space ambient occlusion
- `SMAAPass` — anti-aliasing

## Performance Tips

- Use `InstancedMesh` for many identical objects (1000s to 100,000s)
- Use `Points` for particle systems (millions of points)
- Set `material.flatShading = true` for faceted low-poly aesthetic (also faster)
- Use `THREE.LOD` for distance-based detail levels
- Limit shadow map resolution and shadow-casting objects
- Use `BufferGeometry` always (legacy `Geometry` is removed in modern Three.js)
- Call `renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))` to cap on retina displays

## Coordinate System

- Y-axis points up (same as nannou, unlike p5.js 2D)
- Right-handed coordinate system
- OrbitControls: left-drag rotates, scroll zooms, right-drag pans

## Notable 3D Generative Art Practitioners

- **Andreas Gysin** (ertdfgcvb) — ASCII-meets-3D procedural work
- **Matt DesLauriers** (mattdesl) — creative coding with Three.js, generative landscapes
- **Raven Kwok** — computational 3D sculptures and simulations
- **Nervous System** — nature-inspired 3D generative design
- **Refik Anadol** — large-scale data-driven 3D installations
