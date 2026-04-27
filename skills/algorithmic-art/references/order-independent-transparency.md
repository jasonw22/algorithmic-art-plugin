# Order-Independent Transparency (OIT)

## Philosophy

Standard alpha blending is order-dependent: transparent surfaces must be
sorted back-to-front per pixel for the blend to be correct. This works for
sortable scenes (a few separated transparent panels) and falls apart for
self-overlapping geometry (a gyroid, a curling smoke volume, a glass
sculpture with internal cavities) and for scenes mixing many particles with
mesh transparency (additive splats threading through alpha-blended walls).

**Order-Independent Transparency** sidesteps the sort. Each fragment writes
into accumulation buffers using commutative operations, then a final
composite pass produces the correct (or visually plausible) result
regardless of submission order.

This reference covers **Weighted Blended OIT** (McGuire & Bavoil, 2013) —
the most widely-used real-time approximation. It's a single-pass-per-RT,
geometry-traversal-light technique that works in WebGL 2 / WebGPU and
produces pleasing results for low-frequency transparency (frosted glass,
volumetric haze, foliage). For scenes with sharp depth ordering between
crisp transparent surfaces, depth peeling or per-pixel linked lists give
better quality at higher cost.

## When OIT Solves a Real Problem

- **Self-overlapping transparent geometry** that cannot be cleanly sorted —
  TPMS surfaces, leafy meshes, hair, glass with internal structure.
- **Mixed transparent + additive in the same scene** — alpha-blended walls
  with additive particles glowing through; sort-flipping causes whole
  objects to pop in front of each other.
- **Many overlapping transparent particles or splats** where per-particle
  sort would be expensive.

OIT is **not needed** for: a single transparent object on an opaque
background; UI overlays; cleanly separable transparent layers.

## The Pipeline (6 phases, shared depth)

Four render targets share one depth texture so depth tests propagate across
passes. Order:

```
Phase 0  opaque       clear bg + shared depth, render opaque geometry
Phase A  alpha-depth  alpha objects' nearest depth → shared depth (color writes off)
Phase 1  alpha-accum  alpha objects → accumRT (Σ c·α·w)        [additive blend]
Phase 2  alpha-reveal alpha objects → revealRT (∏(1−α))         [multiplicative blend]
Phase 3  additive     additive objects → additiveRT             [depth-tested against alpha-depth]
Phase 4  composite    final = mix(accum/accum.a, opaque, reveal) + additive  →  canvas
```

Phase A is what lets the additive layer be occluded by alpha geometry while
phases 1 and 2 ignore that depth (their materials are `depthTest: false`),
so both faces of self-overlapping alpha geometry contribute to the blend.

## Render Target Setup (Three.js)

```javascript
const w = renderer.domElement.width;
const h = renderer.domElement.height;

const sharedDepth = new THREE.DepthTexture(w, h);
sharedDepth.type = THREE.UnsignedInt248Type;
sharedDepth.format = THREE.DepthStencilFormat;

const opaqueRT   = new THREE.WebGLRenderTarget(w, h, {
  type: THREE.UnsignedByteType, format: THREE.RGBAFormat, depthTexture: sharedDepth });
const accumRT    = new THREE.WebGLRenderTarget(w, h, {
  type: THREE.HalfFloatType,    format: THREE.RGBAFormat, depthTexture: sharedDepth });
const revealRT   = new THREE.WebGLRenderTarget(w, h, {
  type: THREE.UnsignedByteType, format: THREE.RedFormat,  depthTexture: sharedDepth });
const additiveRT = new THREE.WebGLRenderTarget(w, h, {
  type: THREE.HalfFloatType,    format: THREE.RGBAFormat, depthTexture: sharedDepth });
```

`HalfFloat` for accum and additive — color values can exceed 1.0 (additive
glow, weighted accumulation). `UnsignedByte` is fine for revealage (it's
in [0,1] anyway). Multiple RTs sharing one `depthTexture` is the trick that
makes the cross-pass depth dependencies work — Three.js attaches the same
depth-stencil texture to each FBO.

## Material Variants

The material per object changes per pass. Tag each transparent object with
`userData.oit = { kind, regularMat, accumMat?, revealMat?, depthMat? }` and
swap by mode:

```javascript
function setOITPass(scene, mode) {
  scene.traverse(obj => {
    const o = obj.userData && obj.userData.oit;
    if (!o) return;
    if (!o.regularMat) o.regularMat = obj.material;
    if      (mode === 'alpha-depth')  { obj.visible = (o.kind === 'alpha');    if (obj.visible) obj.material = o.depthMat; }
    else if (mode === 'alpha-accum')  { obj.visible = (o.kind === 'alpha');    if (obj.visible) obj.material = o.accumMat; }
    else if (mode === 'alpha-reveal') { obj.visible = (o.kind === 'alpha');    if (obj.visible) obj.material = o.revealMat; }
    else if (mode === 'additive')     { obj.visible = (o.kind === 'additive'); if (obj.visible) obj.material = o.regularMat; }
    else if (mode === 'opaque')       { obj.visible = false; }
    else                              { obj.visible = true; obj.material = o.regularMat; }
  });
}
```

### Accum material — writes `(c·α·w, α·w)` with additive blend

```glsl
// vertex
varying float vDepth;
void main() {
  vec4 mv = modelViewMatrix * vec4(position, 1.0);
  vDepth = -mv.z;
  gl_Position = projectionMatrix * mv;
}

// fragment
varying float vDepth;
uniform vec3 uColor;
uniform float uOpacity;
void main() {
  float a = uOpacity;
  // Weight: closer fragments dominate; tune the constants per scene scale.
  float w = clamp(10.0 / (1.0 + vDepth * vDepth * 0.05), 0.01, 100.0);
  gl_FragColor = vec4(uColor * a * w, a * w);
}
```

```javascript
// material flags
{
  blending: THREE.CustomBlending,
  blendEquation: THREE.AddEquation,
  blendSrc: THREE.OneFactor, blendDst: THREE.OneFactor,
  depthWrite: false, depthTest: false, side: THREE.DoubleSide,
  transparent: true,
}
```

### Revealage material — writes `(α, 0, 0, 0)` with multiplicative blend

```glsl
// fragment
uniform float uOpacity;
void main() {
  // dst' = dst * (1 - src.r), accumulating ∏(1-α) per pixel
  gl_FragColor = vec4(uOpacity, 0.0, 0.0, 0.0);
}
```

```javascript
{
  blending: THREE.CustomBlending,
  blendEquation: THREE.AddEquation,
  blendSrc: THREE.ZeroFactor, blendDst: THREE.OneMinusSrcColorFactor,
  depthWrite: false, depthTest: false, side: THREE.DoubleSide,
  transparent: true,
}
```

Clear `revealRT` to white (1.0) before the pass — that's the multiplicative
identity. Each fragment multiplies it down by `(1 - α)`.

### Depth-only material — writes depth, no color

```javascript
new THREE.MeshBasicMaterial({
  colorWrite: false,
  depthWrite: true,
  depthTest: true,
  side: THREE.DoubleSide,
});
```

## Composite Shader

A fullscreen quad samples all four buffers:

```glsl
varying vec2 vUv;
uniform sampler2D uOpaque, uAccum, uReveal, uAdditive;
void main() {
  vec4 accum   = texture2D(uAccum,    vUv);
  float reveal = texture2D(uReveal,   vUv).r;
  vec3  opaque = texture2D(uOpaque,   vUv).rgb;
  vec3  add    = texture2D(uAdditive, vUv).rgb;
  vec3 alphaCol = accum.rgb / max(accum.a, 1e-4);
  vec3 col = mix(alphaCol, opaque, reveal) + add;
  gl_FragColor = vec4(col, 1.0);
}
```

Reading the composite as the formula:

```
final = (1 − reveal) · (Σ c·α·w / Σ α·w)   ← weighted average alpha color
      +    reveal     · opaque              ← background bleeds through
      + additive                            ← additive layer on top
```

The weighted average is what makes WBOIT *approximate* — fragments
contribute to the average by their per-pixel depth weight, not by their
true depth order. Pick a weight function that emphasizes closer fragments
and the result usually reads correctly.

## The McGuire Weight Function

The original 2013 paper proposes a few weight functions. A practical form
for scenes with depth in roughly [1, 50]:

```glsl
float w = clamp(0.03 / (1e-5 + pow(depth / 200.0, 4.0)), 0.01, 3000.0);
```

For scenes with smaller depth range (e.g. an attractor + scaffold confined
to ~10 units), this saturates and loses depth discrimination. Use a tuned
version such as the rational form in the accum material above
(`10.0 / (1.0 + d² · 0.05)`), or substitute view-linear depth normalized
to your scene's actual range.

## Hijacking renderer.render Without Modifying the Render Loop

If your render harness owns the per-frame loop and calls `renderer.render`,
override at the renderer instance level so the harness's call transparently
runs the pipeline:

```javascript
const oit = setupOIT(THREE, renderer);
const origRender = renderer.render.bind(renderer);
renderer.render = function (s, c) {
  if (oitEnabled && s !== oit.compositeScene) {
    oitRender(THREE, renderer, s, c, oit, origRender);
  } else {
    origRender(s, c);
  }
};
```

Inside `oitRender`, call `origRender` for each phase's geometry submission
(it just renders the scene with whatever materials and render target you
have currently set). The `s !== oit.compositeScene` guard prevents
infinite recursion when the composite quad itself is being rendered.

## Tradeoffs vs Alternatives

| Technique | Cost | Quality | When |
|-----------|------|---------|------|
| **Sort + alpha** | cheap | exact for non-overlapping | sortable scenes |
| **Depth pre-pass + alpha** | 1 extra pass | exact for one layer; back surfaces vanish | "frosted glass" feel; what you get with `depthWrite: true` on a transparent material |
| **Weighted Blended OIT** (this) | 4 RTs + 5 passes | approximate; pleasing for low-frequency transparency | self-overlapping geometry, mixed alpha+additive |
| **Dual depth peeling** | N geometry passes for 2N layers | near-exact for limited layers | when you need crisp depth order between transparent surfaces |
| **Per-pixel linked lists / A-buffer** | requires SSBOs / atomics | exact | WebGPU only; "I have a budget" tier |

## When OIT is Wrong For Your Scene

- **Crisp transparent objects with strong depth order** (a stack of glass
  panels). WBOIT averages, you want the discrete layered look — sort and
  blend instead.
- **Single-object transparency**. Just sort the geometry's faces, or use
  `depthWrite: true` for the cheaper "frosted" effect.
- **Performance-bound projects on weak hardware**. WBOIT requires HalfFloat
  RTs and 5 passes — has measurable cost on integrated GPUs.

## Notable References & Practitioners

- **Morgan McGuire & Louis Bavoil**, "Weighted Blended Order-Independent
  Transparency," *Journal of Computer Graphics Techniques* 2(2), 2013 — the
  original paper, freely available at jcgt.org.
- **Cesium** — production WBOIT implementation in a globe renderer; their
  weight function tuning is a useful real-world reference.
- **Inigo Quilez** — *Multipass real-time rendering* notes touching on
  shared-depth multi-pass setups.
- For the broader OIT design space (depth peeling, A-buffer, Moment-Based
  OIT), the Wolfgang Engel *GPU Pro* / *GPU Zen* book series is the
  comprehensive reference.
