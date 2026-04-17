---
name: algorithmic-art
description: >
  Use this skill to produce algorithmic art whenever a user wants visual art, beauty,
  or aesthetic exploration through code and math. Supports seven output modes: p5.js 2D (HTML),
  thi.ng functional Canvas 2D (HTML), Three.js 3D scene (HTML), Three.js GLSL shader (HTML),
  WebGPU compute + WGSL (HTML), nannou 2D (Rust), or nannou 3D (Rust).
  This is the default skill for ANY request where the primary goal is creating something visually
  compelling through computation. Trigger for: generative art, fractal explorers, beautiful
  cellular automata (Conway's Game of Life), procedural patterns, particle animations, Penrose
  tilings, tessellations, flow fields, L-systems, reaction-diffusion, strange attractors,
  Mandelbrot/Julia sets, kaleidoscopes, geometric patterns, noise-based visuals, organic growth
  simulations, recursive subdivision, album/poster art from code, animated backgrounds from
  particles/algorithms, interactive parameter-driven sketches, high-res generative prints,
  educational math visualizations, 3D generative sculptures, raymarched fractals (Mandelbulb,
  Menger sponge), volumetric rendering, SDF art, GLSL shader art, WebGPU compute shaders, WGSL,
  Particle Life, MPM fluid / particle-based fluid simulation, large-scale particle systems,
  3D particle systems, 3D strange attractors, generative architecture, procedural landscapes,
  and plotter / AxiDraw output. Also trigger when users want to make something "look beautiful"
  or "cool" using algorithms, even without technical terminology. Skip for: games with win/lose
  mechanics, data dashboards, general web/UI design, or algorithm implementations focused on
  correctness not aesthetics.
argument-hint: "[mode] [description] — modes: p5, thing, scene, shader, webgpu, nannou, nannou3d"
allowed-tools: Read Write Glob Grep WebSearch WebFetch
hooks:
  PostToolUse:
    - matcher: tool == 'Write' && output.filePath.endsWith('.html')
      hooks:
        - type: command
          command: start "" "$output.filePath"
---

# Algorithmic Art

Create algorithmic art in one of seven output modes:
- **p5.js** — standalone 2D HTML files with interactive parameter controls and PNG/SVG export
- **thi.ng** — standalone 2D HTML files with built-in perceptual color (Oklab), 2D SDFs, seeded PRNG, and clean SVG export
- **Three.js Scene** — standalone 3D HTML files with scene graph, lighting, OrbitControls, and PNG export
- **Three.js Shader** — standalone HTML files running a fullscreen GLSL fragment shader (raymarching, SDFs, fractals, volumetric)
- **WebGPU** — standalone HTML files with WGSL compute + render pipelines. Best for large-scale particle systems (Particle Life, MPM fluid), compute-heavy simulation, anything that outgrows WebGL's ping-pong model
- **nannou 2D** — compiled Rust applications using [nannou](https://nannou.cc) for 2D creative coding
- **nannou 3D** — compiled Rust applications using nannou with 3D perspective camera and wgpu

Every piece of algorithmic art expresses an idea — a philosophy about emergence, order, chaos,
nature, or perception. When creating a piece, always articulate the concept driving it, not just
the technique. The technique serves the idea.

## Workflow

When a user requests algorithmic art:

### 0. Choose the Output Mode

**Check if the user specified a mode via arguments first.** When invoked as
`/algorithmic-art [mode] [description]`, the first argument (`$0`) selects the mode directly:

| `$0` value | Mode |
|-------------|------|
| `p5` | p5.js (HTML, 2D) |
| `thing` | thi.ng (HTML, 2D) |
| `scene` or `3d` | Three.js Scene (HTML, 3D) |
| `shader` or `glsl` | Three.js Shader (HTML, GLSL) |
| `webgpu` or `compute` or `wgsl` | WebGPU Compute + WGSL (HTML) |
| `nannou` or `rust` | nannou 2D (Rust) |
| `nannou3d` | nannou 3D (Rust) |

If `$0` matches one of these, skip the mode question entirely and use the remaining arguments
as the art description. If `$0` doesn't match (or no arguments were given), ask the user which
output format they want:

1. **p5.js** (HTML, 2D) — Opens instantly in a browser. Best for interactive 2D parameter
   exploration, quick iteration, and sharing via a single file. Includes a sidebar with live
   controls, seed system, and PNG/SVG export.
2. **thi.ng** (HTML, 2D) — Opens instantly in a browser. Best for perceptual color art
   (Oklab/Oklch built-in), 2D SDF compositions (smooth booleans, domain repetition),
   plotter-ready SVG export, and functional/data-driven workflows. Includes the same sidebar,
   seed system, and PNG/SVG export. Uses Canvas 2D API directly (no p5.js dependency).
3. **Three.js Scene** (HTML, 3D) — Opens instantly in a browser. Best for 3D generative
   sculptures, particle systems, instanced geometry, architectural forms, and any piece with
   distinct 3D objects. Includes OrbitControls (mouse orbit/pan/zoom), sidebar with live
   controls, seed system, and PNG export.
4. **Three.js Shader** (HTML, 3D/GPU) — Opens instantly in a browser. Best for fullscreen GPU
   effects: raymarched SDFs, 3D fractals (Mandelbulb, Menger sponge), volumetric rendering,
   GPU reaction-diffusion, domain warping, and Shadertoy-style pieces. Same sidebar and seed
   system. Requires GLSL knowledge.
5. **WebGPU** (HTML, GPU compute) — Opens instantly in a browser (Chrome/Edge/Safari 17+).
   Best for large particle counts, MLS-MPM fluid, Particle Life, compute-heavy simulation, and
   any piece that outgrows the WebGL ping-pong model. Exposes a compute+render pipeline via
   three sketch hooks (`sketchSetup`, `computePass`, `renderPass`). Same sidebar and seed
   system. Requires WGSL knowledge.
6. **nannou 2D** (Rust) — Compiled native app. Best for high-performance 2D rendering,
   large-scale generative prints, and users who prefer Rust.
7. **nannou 3D** (Rust) — Compiled native app. Best for high-performance 3D particle systems,
   3D attractors, custom wgpu shader pipelines, and Rust-based 3D generative work.

If the user has already specified a preference (e.g., "make me a nannou app", "p5.js sketch",
"3D fractal", "raymarched"), skip the question and proceed with that mode. If the context makes
one choice obvious, infer without asking:
- "open in browser" / "HTML" → p5.js (2D) or Three.js (3D)
- "thi.ng" / "Oklab" / "perceptual color" / "SDF composition" / "plotter" / "functional" → thi.ng
- "Rust project" / "compiled" → nannou
- "3D sculpture" / "3D particles" / "orbit camera" → Three.js Scene or nannou 3D
- "raymarching" / "SDF" / "Mandelbulb" / "shader" / "GLSL" → Three.js Shader
- "fractal" without "3D" → p5.js (2D); "3D fractal" → Three.js Shader
- "SVG export" / "vector output" / "plotter-ready" → thi.ng (best SVG support)
- "WebGPU" / "compute shader" / "WGSL" / "particle life" / "MPM fluid" / "100k+ particles" → WebGPU
- "plotter" / "AxiDraw" / "pen plotter" / "watercolor-with-plotter" / "hybrid analog" → thi.ng (for SVG) + consult `references/plotter-workflow.md`

The rest of this workflow applies to all modes. Sections specific to one mode are marked
**[p5.js]**, **[thi.ng]**, **[Three.js Scene]**, **[Three.js Shader]**, **[WebGPU]**, **[nannou 2D]**, or **[nannou 3D]**.

### 1. Interpret the Request

Identify whether the request maps to a known algorithm family or requires something new.

**Known families** (see `references/` for each):
| Family | Reference File | Key Concepts |
|--------|---------------|--------------|
| Fractals & L-systems | `fractals-lsystems.md` | Self-similarity, recursive growth, Mandelbrot, Julia, tree grammars |
| Reaction-diffusion | `reaction-diffusion.md` | Turing patterns, Gray-Scott, morphogenesis, emergent texture |
| Flow fields & noise | `flow-fields-noise.md` | Perlin, simplex, curl noise, particle traces, organic movement |
| Cellular automata | `cellular-automata.md` | Wolfram rules, Conway, emergence from simple rules |
| Strange attractors | `strange-attractors.md` | Lorenz, Clifford, de Jong, chaos theory, deterministic unpredictability |
| Tiling & tessellation | `tiling-tessellation.md` | Penrose, Truchet, aperiodic order, Islamic geometry |
| Recursion & subdivision | `recursion-subdivision.md` | Mondrian-style, quadtree, space partitioning, compositional hierarchy |
| Generative agents & typography | `generative-agents.md` | Maeda, Reas, autonomous agents, flocking, steering behaviors |
| WebGPU compute | `webgpu-compute.md` | WGSL, compute shaders, storage buffers, ping-pong compute, large-scale GPU simulation |
| Particle fluid (MPM) | `mpm-fluid.md` | Material Point Method, particle-grid transfers, APIC/MLS, real-time fluid |
| Particle Life | `particle-life.md` | Asymmetric-force emergent swarms, predator-prey dynamics, species force matrix |
| Plotter workflow | `plotter-workflow.md` | AxiDraw, path planning (vpype/Saxi), pen/medium choice, multi-pass, watercolor-with-plotter |

**Cross-cutting references** (applicable to all algorithm families and output modes):

| Reference | Key Concepts |
|-----------|--------------|
| `color-science.md` | Perceptual color spaces (Oklab, LCH), cosine gradients, LCH theme generation, harmonic schemes, perceptual interpolation |
| `sdf-2d.md` | 2D signed distance fields, smooth boolean operators, domain repetition/mirroring/polar symmetry, anti-aliased rendering |
| `thi-ng-ecosystem.md` | Functional creative coding patterns, composable tessellation, data-driven geometry, multi-target rendering, GenArt API parameter design |

**Mode-specific references:**

| Reference | Modes | Key Concepts |
|-----------|-------|--------------|
| `nannou.md` | nannou 2D | Rust/nannou API, coordinate system, noise, color, drawing primitives |
| `thing-2d.md` | thi.ng | Canvas 2D API + tng utilities (Oklab color, 2D SDFs, SFC32 PRNG, Perlin noise, vector math), sketchSVG() for clean vector export |
| `threejs-3d.md` | Three.js Scene | Scene graph, geometry, materials, lighting, instanced meshes, particles, post-processing |
| `shaders-glsl.md` | Three.js Shader | GLSL, SDFs, raymarching, 3D fractals, noise in GLSL, volumetric rendering, CSG operations, PBR lighting, tone mapping, polar UV, easing, fog, blending |
| `multipass-buffers.md` | Three.js Shader | Ping-pong framebuffers for GPU simulation (fluid, cellular automata, reaction-diffusion) |
| `post-processing.md` | Three.js Shader, Three.js Scene | Bloom, vignette, chromatic aberration, film grain, CRT, tone mapping, color grading |
| `gpu-particles.md` | Three.js Scene | GPGPU particle systems via GPUComputationRenderer, texture-based simulation, Verlet integration, curl noise forces, attractor patterns |
| `superformula.md` | All modes | Gielis superformula for parametric organic shapes (2D + 3D supershapes), parameter morphing, preset library |
| `voronoi-noise.md` | Three.js Shader, p5.js, thi.ng | Voronoi/cellular noise, distance metrics, F1/F2 patterns, edge detection, 3D Voronoi |
| `path-tracing.md` | Three.js Shader | Monte Carlo path tracing, Cook-Torrance PBR, importance sampling, progressive accumulation |
| `atmospheric-scattering.md` | Three.js Shader | Rayleigh/Mie scattering, physical sky, aerial perspective, god rays |
| `water-ocean.md` | Three.js Shader | Gerstner waves, Fresnel, subsurface scattering, caustics, foam |
| `terrain-rendering.md` | Three.js Shader | Procedural heightfields, ridged noise, biome materials, terrain raymarching |
| `anti-aliasing.md` | Three.js Shader | Supersampling (RGSS, stochastic), analytical AA with fwidth, temporal AA |
| `procedural-2d-patterns.md` | Three.js Shader | Checkerboard, brick, hex grid, Truchet, stripes, polka dots in GLSL |
| `analytic-raytracing.md` | Three.js Shader | Exact ray-primitive intersection (sphere, box, plane, cylinder), reflection/refraction |
| `sound-synthesis.md` | Three.js Shader | Shader audio, oscillators, envelopes, drums, WebAudio integration |
| `audio-reactive-mappings.md` | All modes | Audio analysis, frequency band extraction, smoothing, beat detection, audio-to-visual property mappings, audio-driven post-processing |
| `webgl-pitfalls.md` | Three.js Shader | Precision issues, common bugs, visual debugging, mobile compatibility |
| `nannou-3d.md` | nannou 3D | 3D perspective camera, 3D particles, wgpu pipelines, WGSL shaders, 3D attractors |
| `line-art-contours.md` | Three.js Scene, Three.js Shader | Silhouette/crease/boundary extraction, screen-space edge detection, inverted-hull outlines, toon shading, SVG export, hatching |
| `webgpu-compute.md` | WebGPU | WGSL primer, compute pipeline model, storage buffers, uniform buffer layout, template hook API, atomic P2G pattern |
| `mpm-fluid.md` | WebGPU (primary); Three.js Shader (grid-based fluid alternative) | MLS-MPM particle fluid, 5-pass frame structure, quadratic B-spline weights, constitutive models, atomic P2G |
| `particle-life.md` | WebGPU (primary); p5.js, nannou (CPU variants) | Asymmetric force matrix, species presets, O(N²) WGSL baseline, spatial hashing for large N |
| `plotter-workflow.md` | thi.ng (primary), any mode producing SVG | Path planning (vpype/Saxi), pen/medium selection, multi-pass color, watercolor-with-plotter, AxiDraw settings |

All algorithm families above apply to all output modes — the family reference files describe
the math, while the mode-specific references cover the implementation patterns for each platform.

If the request references a technique not covered here, **search the web** for reference material,
then save what you learn as a new `.md` file in `references/` following the same structure as the
existing files. This skill grows over time.

### 2. Articulate the Concept

Before writing code, explain the philosophy or concept driving the piece. Examples:
- "This piece explores **emergence** — how complex organic patterns arise from two simple chemicals
  reacting and diffusing, as Turing proposed in 1952."
- "This combines **deterministic chaos** (the Lorenz attractor's sensitive dependence on initial
  conditions) with **Perlin noise flow fields** to contrast two kinds of unpredictability."

If the user provides their own concept or philosophy, honor it and connect it to the techniques
you choose.

### 3. Build the Sketch

#### [p5.js] Build as 2D HTML

Read the p5.js template at `assets/template.html`. This template must not be modified — it provides:
- Full-viewport canvas that fills the browser window
- **Sidebar control panel** (right side, toggled via hamburger button) with styled HTML controls
  generated automatically from the PARAMS config object
- **Seed system**: seed input, prev/next/random buttons. `p.randomSeed()` and `p.noiseSeed()`
  are called automatically before `sketchSetup()`. Same seed = same output.
- PNG export (canvas.toBlob) and SVG export (edge detection fallback) buttons
- Pause, Reset, and window resize handling
- Export filenames include the seed number for reproducibility

All sketches use the standard p5.js canvas renderer (P2D). Do not use the SVG renderer.
For vector-native techniques (tilings, L-systems, subdivision), build your own SVG export by
constructing an SVG string from the geometry data and downloading it as a blob — this produces
cleaner vector output than edge detection. Override the export button behavior in your sketch
code if needed.

To create a piece, copy the template and fill in the three designated sections:

```javascript
// ═══════════════════════════════════════════════
// SECTION 1: METADATA
// ═══════════════════════════════════════════════
const SKETCH_META = {
  title: "Piece Title",
  concept: "Brief philosophical description",
  technique: "Algorithm family name",
  renderMode: "canvas"
};

// ═══════════════════════════════════════════════
// SECTION 2: PARAMETERS
// Each becomes a sidebar control, grouped by folder.
// Supported types:
//   Slider:   { value: 0.5, min: 0, max: 1, step: 0.01, label: "Name", folder: "Section" }
//   Dropdown: { value: "opt1", options: ["opt1", "opt2"], label: "Name", folder: "Section" }
//   Color:    { value: "#ff0000", type: "color", label: "Name", folder: "Section" }
//   Boolean:  { value: true, label: "Name", folder: "Section" }
// ═══════════════════════════════════════════════
const PARAMS = {
  depth: { value: 6, min: 2, max: 8, step: 1, label: "Subdivisions", folder: "Structure" },
  palette: { value: "Sunset", options: ["Sunset", "Ocean", "Neon"], label: "Palette", folder: "Color" },
  bgColor: { value: "#1a1a1a", type: "color", label: "Background", folder: "Color" },
  showGrid: { value: false, label: "Show Grid", folder: "Debug" },
};

// ═══════════════════════════════════════════════
// SECTION 3: SKETCH LOGIC
// p.randomSeed() and p.noiseSeed() are already set.
// ═══════════════════════════════════════════════
function sketchSetup(p, width, height) { /* one-time init */ }
function sketchDraw(p, width, height, params) { /* per-frame drawing */ }
```

The template reads these and handles everything else — sidebar generation, seed management,
export, resize. Do not add dat.gui or build custom HTML controls.

#### [thi.ng] Build as Functional Canvas 2D HTML

Read the thi.ng template at `assets/template-thing.html`. This template must not be modified —
it provides:
- Full-viewport Canvas 2D context that fills the browser window
- Same sidebar control panel, seed system, and PNG/SVG export as other templates
- **Built-in `tng` utility library** with perceptual color (Oklab/Oklch), cosine gradients,
  2D SDF primitives + smooth boolean operators + domain operations, seeded SFC32 PRNG,
  2D Perlin noise with fBm, and vector math
- Clean SVG export via optional `sketchSVG()` function (falls back to edge detection if not implemented)

Set `renderMode: "thing"` in SKETCH_META. Copy the template and fill in three sections:

```javascript
const SKETCH_META = {
  title: "Piece Title",
  concept: "Brief philosophical description",
  technique: "Algorithm family name",
  renderMode: "thing"
};

const PARAMS = {
  // Same format as all other templates
};

function sketchSetup(ctx, w, h, tng) {
  // ctx = Canvas 2D context, tng = utility library
  // tng.random is already seeded from the seed control
  // Return a state object (or undefined)
}

function sketchDraw(ctx, w, h, params, tng, state, time, delta) {
  // time = seconds since start, delta = seconds since last frame
  // Draw to ctx using Canvas 2D API + tng utilities
}

// OPTIONAL: implement for clean vector SVG export
function sketchSVG(w, h, params, tng, state) {
  return '<svg xmlns="http://www.w3.org/2000/svg" ...>...</svg>';
}
```

See `references/thing-2d.md` for the full `tng` API reference including color, SDF,
random, noise, vector, and math utilities, plus common patterns for SDF composition,
particle flow fields with SVG export, and tessellation with data geometry.

**Key advantages over p5.js mode:**
- Perceptual color (Oklab/Oklch) built into `tng.color` — no manual conversion needed
- 2D SDF composition with smooth booleans via `tng.sdf` — ideal for shape-based art
- First-class SVG export via `sketchSVG()` — plotter-ready vector output from geometry data
- Seeded SFC32 PRNG via `tng.random` — deterministic, high-quality randomness
- Seeded Perlin noise via `tng.noise` — deterministic noise fields
- No external library dependency — everything is built into the template

#### [Three.js Scene] Build as 3D HTML

Read the 3D template at `assets/template-3d.html`. This template must not be modified — it provides:
- Full-viewport WebGL canvas via Three.js (ES modules via importmap)
- **OrbitControls**: left-drag to rotate, scroll to zoom, right-drag to pan
- Same sidebar control panel, seed system, and PNG export as the 2D template
- `preserveDrawingBuffer: true`, ACES tone mapping, SRGB color space
- `window.seededRandom()` — a mulberry32 PRNG seeded from the current seed
- Proper scene disposal/rebuild on seed change or reset

Set `renderMode: "scene"` in SKETCH_META. Copy the template and fill in three sections:

```javascript
const SKETCH_META = {
  title: "Piece Title",
  concept: "Brief philosophical description",
  technique: "Algorithm family name",
  renderMode: "scene"
};

const PARAMS = {
  count: { value: 1000, min: 100, max: 10000, step: 100, label: "Object Count", folder: "Structure" },
  // ... same format as 2D template
};

// Called once — add meshes, lights, helpers to scene.
// Camera is a PerspectiveCamera at (0, 0, 5). Reposition as needed.
// Return a state object for use in sceneAnimate (or undefined).
function sceneSetup(THREE, scene, camera, renderer, params, seed) { }

// Called every frame. time = elapsed seconds, delta = frame delta.
function sceneAnimate(THREE, scene, camera, state, params, time, delta) { }
```

See `references/threejs-3d.md` for geometry, materials, lighting, instanced meshes,
particles, post-processing, and common 3D generative patterns.

#### [Three.js Shader] Build as GLSL Shader HTML

Uses the same 3D template (`assets/template-3d.html`) with `renderMode: "shader"`.
The template renders a fullscreen quad and passes your fragment shader via ShaderMaterial.

```javascript
const SKETCH_META = {
  title: "Piece Title",
  concept: "Brief philosophical description",
  technique: "Algorithm family name",
  renderMode: "shader"
};

const PARAMS = {
  power: { value: 8.0, min: 2, max: 16, step: 0.1, label: "Fractal Power", folder: "Structure" },
  // ...
};

// Return custom uniforms. The template automatically provides:
//   u_time (float), u_resolution (vec2), u_mouse (vec2), u_seed (float)
function shaderUniforms(params, seed) {
  return {
    u_power: { value: params.power },
  };
}

// Return a GLSL fragment shader string.
function fragmentShader() {
  return `
    uniform float u_time;
    uniform vec2 u_resolution;
    uniform float u_power;
    void main() {
      vec2 uv = (gl_FragCoord.xy - 0.5 * u_resolution) / u_resolution.y;
      // ... raymarching, SDF, fractal logic ...
      gl_FragColor = vec4(color, 1.0);
    }
  `;
}

// Optional: sync custom uniforms to param changes per frame.
function shaderAnimate(uniforms, params, time) {
  uniforms.u_power.value = params.power;
}
```

See `references/shaders-glsl.md` for SDF primitives, CSG operations, raymarching,
3D fractals (Mandelbulb, Menger sponge), noise in GLSL, volumetric effects, and palettes.

##### Shader Technique Routing Table

When a shader mode request arrives, use this table to identify the **primary technique**
and **combinable techniques** to read from `references/`. Read the primary reference first,
then pull in combinable references as needed for the full implementation.

| User Intent / Keywords | Primary Reference | Combinable References |
|------------------------|-------------------|----------------------|
| "raymarched scene", "SDF sculpture", "organic shapes" | `shaders-glsl.md` (SDFs + raymarching) | `post-processing.md`, `anti-aliasing.md` |
| "Mandelbulb", "Menger sponge", "3D fractal" | `shaders-glsl.md` (3D Fractals) | `post-processing.md`, `anti-aliasing.md` |
| "photorealistic", "PBR", "metallic/rough" | `path-tracing.md` | `shaders-glsl.md` (SDFs), `post-processing.md`, `anti-aliasing.md` |
| "global illumination", "path trace", "bounced light" | `path-tracing.md` | `multipass-buffers.md`, `shaders-glsl.md` |
| "sky", "sunset", "atmosphere", "god rays" | `atmospheric-scattering.md` | `terrain-rendering.md`, `post-processing.md` |
| "ocean", "water", "waves", "sea" | `water-ocean.md` | `atmospheric-scattering.md`, `post-processing.md` |
| "terrain", "landscape", "mountains", "procedural land" | `terrain-rendering.md` | `atmospheric-scattering.md`, `post-processing.md` |
| "reaction-diffusion", "Turing pattern" (GPU) | `multipass-buffers.md` | `shaders-glsl.md` (noise), `post-processing.md` |
| "fluid", "smoke", "ink in water" | `multipass-buffers.md` | `post-processing.md` |
| "Game of Life", "cellular automata" (GPU) | `multipass-buffers.md` | `post-processing.md` |
| "Voronoi", "cells", "cracked", "bubbles" | `voronoi-noise.md` | `shaders-glsl.md` (noise), `post-processing.md` |
| "bloom", "glow", "film grain", "CRT", "retro" | `post-processing.md` | (any scene technique) |
| "kaleidoscope", "mandala", "radial", "spiral" (shader) | `shaders-glsl.md` (Polar UV) | `voronoi-noise.md`, `post-processing.md` |
| "checkerboard", "brick", "hex grid", "Truchet" (shader) | `procedural-2d-patterns.md` | `shaders-glsl.md`, `post-processing.md` |
| "glass sphere", "mirror", "refraction" | `analytic-raytracing.md` | `shaders-glsl.md` (lighting), `post-processing.md` |
| "line art", "contour", "outline", "silhouette", "ink drawing" | `line-art-contours.md` | `threejs-3d.md`, `post-processing.md` |
| "toon", "cel shading", "cartoon", "comic" | `line-art-contours.md` (Toon + Outline) | `threejs-3d.md`, `post-processing.md` |
| "wireframe art", "blueprint", "technical drawing", "pen and ink" | `line-art-contours.md` | `threejs-3d.md` |
| "hatching", "cross-hatch", "engraving", "woodcut" | `line-art-contours.md` (Hatching) | `shaders-glsl.md`, `post-processing.md` |
| "anti-aliased", "smooth edges", "supersampled" | `anti-aliasing.md` | (any scene technique) |
| "audio", "sound", "music", "audio-reactive" | `audio-reactive-mappings.md` | `sound-synthesis.md`, (any visual technique) |
| "WebGL bug", "black screen", "precision", "debug" | `webgl-pitfalls.md` | — |
| "particle life", "asymmetric swarm", "predator-prey particles" (→ switch to WebGPU mode) | `particle-life.md` | `webgpu-compute.md`, `color-science.md`, `multipass-buffers.md` (trails) |
| "MPM fluid", "particle fluid", "splash", "droplets" (→ switch to WebGPU mode) | `mpm-fluid.md` | `webgpu-compute.md`, `gpu-particles.md` |
| "WebGPU", "compute shader", "WGSL", "large particle count" (→ switch to WebGPU mode) | `webgpu-compute.md` | technique-specific reference per piece |

**Quick Recipes** — common multi-technique compositions:

1. **Photorealistic SDF Scene**: `shaders-glsl.md` (SDFs + raymarching + lighting) → `path-tracing.md` (PBR) → `post-processing.md` (tone mapping + bloom + vignette) → `anti-aliasing.md` (RGSS)
2. **Procedural Landscape**: `terrain-rendering.md` → `atmospheric-scattering.md` → `water-ocean.md` (if water present) → `post-processing.md` (tone mapping + fog)
3. **GPU Simulation Art**: `multipass-buffers.md` (ping-pong setup) → technique-specific shader (RD from `reaction-diffusion.md`, fluid, or CA) → `post-processing.md` (color grading)
4. **Organic Forms**: `shaders-glsl.md` (smooth SDF unions + domain warping) → `voronoi-noise.md` (surface texture) → `post-processing.md` (bloom + vignette)
5. **Abstract Shader Art**: `shaders-glsl.md` (Polar UV + noise) → `procedural-2d-patterns.md` → `post-processing.md` (chromatic aberration + film grain)
6. **Stylized 2D Pattern**: `procedural-2d-patterns.md` → `voronoi-noise.md` → `post-processing.md` (CRT or vignette)
7. **Line Art / Ink Drawing**: `line-art-contours.md` (edge extraction or screen-space detection) → `threejs-3d.md` (geometry) → `post-processing.md` (paper texture + vignette)
8. **Toon / Cel Shaded**: `line-art-contours.md` (toon shading + outlines) → `threejs-3d.md` (scene setup) → `post-processing.md` (posterize + bloom)
9. **Large-Scale Particle Simulation** (WebGPU mode): `webgpu-compute.md` (compute+render pattern) → technique-specific (`particle-life.md`, `mpm-fluid.md`, or custom) → `color-science.md` (species palettes) → optional `multipass-buffers.md` (trails) → optional `post-processing.md` (bloom)
10. **Plotted Hybrid Piece** (thi.ng mode for geometry → physical output): `thing-2d.md` (composition + `sketchSVG()`) → `line-art-contours.md` (if contour/hatching-based) → `plotter-workflow.md` (path optimization, pen choice, multi-pass)

#### [WebGPU] Build as WebGPU Compute HTML

Read the WebGPU template at `assets/template-webgpu.html`. This template must not be modified
— it provides:
- Full-viewport WebGPU canvas with graceful fallback for unsupported browsers
- Shared uniform buffer layout already bound at `@group(0) @binding(0)` of your WGSL (time,
  resolution, seed, count, dt; see the layout comment in the template)
- Same sidebar control panel, seed system, PNG export, and resize handling as the other HTML
  templates
- `window.seededRandom()` for CPU-side deterministic randomness (mulberry32), seeded from the
  current seed input

Set `renderMode: "webgpu"` in SKETCH_META. Copy the template and fill in the three designated
sections — metadata, PARAMS, and the three sketch hooks:

```javascript
const SKETCH_META = {
  title: "Piece Title",
  concept: "Brief philosophical description",
  technique: "Algorithm family name",
  renderMode: "webgpu"
};

const PARAMS = {
  // Same format as all other templates. Changing `count` triggers a sketch rebuild.
};

// Called once (and again whenever the seed changes or a rebuild-triggering param changes).
// Return a state object whose fields are used by computePass/renderPass.
function sketchSetup(device, context, format, params, seed, uniformBuffer) {
  // Create shader module(s), storage buffers, pipelines, bind groups.
  // Bind `uniformBuffer` at @group(0) @binding(0) in your WGSL.
  // For ping-pong compute, create two storage buffers and two bind groups; alternate between them.
  return { /* handles used by computePass and renderPass */ };
}

// Called every frame, before renderPass. Record compute dispatches on `encoder`.
function computePass(device, encoder, state, params, time, delta) {
  // device.queue.writeBuffer(...) any per-frame uniforms or params
  const cp = encoder.beginComputePass();
  cp.setPipeline(state.computePipeline);
  cp.setBindGroup(0, state.bindGroup);
  cp.dispatchWorkgroups(Math.ceil(state.count / 64));
  cp.end();
}

// Called every frame, after computePass. Record the render pass onto `view` (the current
// canvas view).
function renderPass(device, encoder, view, state, params, time, delta) {
  const rp = encoder.beginRenderPass({
    colorAttachments: [{
      view, loadOp: "clear", storeOp: "store",
      clearValue: { r: 0, g: 0, b: 0, a: 1 },
    }],
  });
  rp.setPipeline(state.renderPipeline);
  rp.setBindGroup(0, state.bindGroup);
  rp.draw(6, state.count);
  rp.end();
}
```

See `references/webgpu-compute.md` for the WGSL primer, pipeline setup details, uniform-buffer
layout, atomic patterns for scatter writes (used in MPM), and library-ecosystem notes
(POINTS, Three.js TSL, Slang). Technique-specific references:
- `references/mpm-fluid.md` for particle-based fluid (MLS-MPM)
- `references/particle-life.md` for asymmetric-force swarm systems
- `references/gpu-particles.md` (WebGL predecessor) for algorithmic patterns that apply to
  both substrates (forces, integrators, respawn strategies)

**Browser note:** Chrome, Edge, Safari 17+ ship WebGPU by default. Firefox requires
enabling `dom.webgpu.enabled` in `about:config` as of early 2026. The template shows a
friendly fallback message when WebGPU isn't available.

#### [nannou 2D] Build as Rust Project

Read the nannou project template at `assets/nannou-template/`. Create a new Cargo project
directory for the piece. The project structure is:

```
piece-name/
├── Cargo.toml
├── src/
│   └── main.rs
└── README.md        (optional — brief run instructions)
```

See `references/nannou.md` for the full nannou reference including API patterns, coordinate
system, color handling, noise, and export. The key structure of a nannou sketch:

```rust
use nannou::prelude::*;
use nannou::noise::{NoiseFn, Perlin, Seedable};

struct Model {
    // Piece state — algorithm data, parameters, cached geometry
}

fn main() {
    nannou::app(model).update(update).run();
}

fn model(app: &App) -> Model {
    app.new_window()
        .size(1200, 800)
        .title("Piece Title")
        .view(view)
        .key_pressed(key_pressed)
        .build()
        .unwrap();

    Model {
        // Initialize state
    }
}

fn update(_app: &App, model: &mut Model, _update: Update) {
    // Per-frame state updates — advance simulation, respond to parameter changes
}

fn view(app: &App, model: &Model, frame: Frame) {
    let draw = app.draw();
    draw.background().color(BLACK);

    // Drawing code — use draw.ellipse(), draw.line(), draw.polyline(), etc.

    draw.to_frame(app, &frame).unwrap();
}

fn key_pressed(app: &App, model: &mut Model, key: Key) {
    match key {
        Key::S => app.main_window().capture_frame(format!(
            "{}_{}.png",
            app.exe_name().unwrap(),
            app.elapsed_frames()
        )),
        Key::R => { /* reset state */ }
        Key::Space => { /* pause/resume */ }
        _ => {}
    }
}
```

**nannou conventions:**
- Parameters live as fields on the `Model` struct. Use keyboard controls to adjust them at
  runtime (arrow keys, number keys, etc.). Document the key bindings in a `println!` at startup.
- Seed system: store a `seed: u32` field on Model. Use `Key::N` for next seed, `Key::P` for
  previous, `Key::R` for random. Apply via `Perlin::new().set_seed(model.seed)` and
  nannou's `random_range` with seeded RNG.
- Export: `Key::S` captures the current frame as PNG. For high-res export, render to a
  texture at a multiple of the window size, then save that.
- The coordinate system is centered at (0, 0) with y-up. Window dimensions are
  `app.window_rect().w()` and `.h()`.
- Use `nannou::noise` (re-exported from the `noise` crate) for Perlin/Simplex noise.
- Use `nannou::color` (re-exported from `palette` crate) for color — supports `hsla`, `rgba`,
  `Srgba`, `LinSrgba`, named colors, and conversions between them.

**Cargo.toml template:**
```toml
[package]
name = "piece-name"
version = "0.1.0"
edition = "2021"

[dependencies]
nannou = "0.19"
```

After creating the project, tell the user to run it with `cargo run` (or `cargo run --release`
for better performance). Note that the first build will take a few minutes to compile nannou's
dependencies.

#### [nannou 3D] Build as 3D Rust Project

Same project structure as nannou 2D, but with a 3D perspective camera and 3D drawing.
See `references/nannou-3d.md` for the full 3D reference.

Key differences from nannou 2D:
- Build a perspective view matrix manually (`Mat4::look_at_rh` + `Mat4::perspective_rh`)
  and apply it via `draw.transform(matrix)`. See `nannou-3d.md` for the
  `perspective_view_matrix` helper.
- Add camera state to the Model: `camera_angle`, `camera_distance`, `camera_height`.
  Use arrow keys and +/- for camera control.
- Use `x_y_z()` and `pt3()` for 3D positioning of shapes.
- Use `.rotate_x()`, `.rotate_y()`, `.rotate_z()` for 3D rotation.
- For advanced 3D (custom lighting, real meshes), use custom wgpu pipelines with WGSL shaders.
- Add `glam = "0.24"` to Cargo.toml for matrix/quaternion math if needed.

### 5. Design Parameters

Parameters are the user's creative controls — and also the piece's vocabulary. Name parameters
in the language of the concept, not the language of the algorithm. If the piece is about erosion,
call it "Weathering Intensity" not "noise_scale". If it's about flocking, call it "Social Distance"
not "separation_radius". This connects the user's interaction to the artwork's meaning and makes
exploration more intuitive.

Design them thoughtfully:
- **Structural parameters**: Change the fundamental character (e.g., number of iterations, rule selection)
- **Aesthetic parameters**: Change the look without changing the structure (e.g., color, stroke weight, scale)
- **Behavioral parameters**: Change dynamics (e.g., speed, diffusion rate, attraction strength)
- **Palette parameters**: Always include color/palette controls. Use dropdown params with named
  palettes (e.g., "Sunset", "Ocean", "Neon", "Monochrome") so users can dramatically change the
  feel without understanding color theory. Include at least 3-4 palette options. If the user
  requests a specific palette (like "warm"), still provide multiple variations within that theme.
  See `references/color-science.md` for cosine gradient presets, perceptual color spaces,
  and LCH theme generation techniques.

**Organize parameters into folders** using the `folder` field. The sidebar groups controls
into sections by folder name. Use folders like "Structure", "Appearance", "Behavior", "Color"
to keep the panel organized. The sidebar starts hidden — the user clicks the hamburger
button (top-right) to reveal it.

**For techniques with sensitive parameter spaces** (reaction-diffusion, strange attractors),
include a "Preset" dropdown parameter that sets multiple related parameters to known-good
combinations. This is critical — users should be able to select "Coral", "Mitosis", or
"Stripes" and immediately see an interesting result, rather than fumbling with feed/kill
rates blindly. When the preset changes, update the other parameter values accordingly
in the draw loop (check if preset changed since last frame).

Provide sensible defaults that produce a visually compelling result out of the box. The default
state should be the "hero" view of the piece.

### 6. Explain and Deliver

When presenting the finished piece, include:
1. **Title and concept** — what idea the piece expresses
2. **Technique breakdown** — what algorithms are at work and why they were chosen
3. **Parameter guide** — **[p5.js / Three.js]** what each sidebar control does and interesting
   ranges to explore; **[nannou]** what each keyboard shortcut does and interesting parameter values
4. **Art historical context** — connections to artists, movements, or foundational work
5. **The output** — **[p5.js / Three.js]** the HTML file, saved and ready to open in a browser;
   **[nannou]** the Cargo project directory, with `cargo run` instructions and a note that
   the first build compiles dependencies (~2-5 min)
6. **[Three.js Scene]** Note the mouse controls: left-drag to orbit, scroll to zoom, right-drag to pan
7. **[Three.js Shader]** Explain what the shader does conceptually — raymarching, SDF construction, etc.

## Combining Techniques

When a request combines multiple families (e.g., "flow field with Truchet tiles" or
"reaction-diffusion driving an L-system"), think about how the techniques interact:

- **Layering**: One technique provides a background/texture, another provides foreground structure
- **Driving**: One technique's output parameterizes another (noise drives attractor parameters)
- **Hybridizing**: Merge the core algorithms (e.g., cellular automata rules applied to a tiling grid)

Explain the interaction model you chose and why.

## When Techniques Are Unknown

If the user references a technique, algorithm, artist, or artwork you don't have in `references/`:

1. Search the web for the technique, algorithm, or artist's approach
2. Find the mathematical or algorithmic basis
3. Create a new reference file in `references/` with:
   - History and philosophy
   - Core algorithm description
   - Key parameters
   - Notable practitioners
   - p5.js implementation notes
4. Then proceed with the sketch as normal

This is how the skill's knowledge base expands. Every new technique learned benefits future requests.

## Suggested Learning Resources

See `references/sources.md` for a curated list of books, papers, websites, and courses
on algorithmic art. When adding new techniques, also add relevant sources to this file.

## Pixel Buffer Gotcha

When using `p.createGraphics()` for offscreen buffers that you'll manipulate via `loadPixels()`/`updatePixels()`, always call `buffer.pixelDensity(1)` immediately after creation. Without this, high-DPI displays cause the pixel array to be larger than `width * height * 4`, and manual pixel indexing (`(y * w + x) * 4`) will only fill a fraction of the buffer — the rendering appears squished into the top-left corner.

## Export Details

The template provides PNG and SVG export buttons in the bottom bar.

- **PNG Export**: Uses `canvas.toBlob()` to capture the current frame. Works reliably for
  all canvas-based sketches.
- **SVG Export (raster fallback)**: The template's default SVG export runs edge detection
  (Sobel operator → contour tracing → SVG path generation) on the current canvas frame.
  This produces an SVG approximation — good for bold/high-contrast pieces, less precise
  for subtle gradients. The edge detection threshold is adjustable in the GUI.
- **SVG Export (vector-native override)**: For techniques that are inherently geometric
  (tilings, L-systems, subdivision, line-based flow fields), build a proper SVG string
  from your geometry data in the sketch code and override the SVG export button's click
  handler. This produces clean, resolution-independent output. Example:
  ```javascript
  // In sketchSetup or sketchDraw, store geometry for SVG export
  // Then override the export button after init:
  document.getElementById("btn-svg").onclick = function() {
    let svg = '<svg xmlns="http://www.w3.org/2000/svg" ...>';
    // ... build SVG from stored geometry ...
    svg += '</svg>';
    const blob = new Blob([svg], {type: "image/svg+xml"});
    const a = document.createElement("a");
    a.href = URL.createObjectURL(blob);
    a.download = "export.svg";
    a.click();
  };
  ```
