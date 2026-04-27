# Algorithmic Art Plugin for Claude Code

Create algorithmic art as standalone files with interactive parameter controls and export capabilities. Supports seven output modes spanning 2D, 3D, GLSL shaders, WebGPU compute, and compiled Rust.

## Installation

```bash
claude plugin add jasonw22/algorithmic-art-plugin
```

Or for local development:

```bash
claude --plugin-dir /path/to/algorithmic-art-plugin
```

## Usage

Invoke the skill in Claude Code:

```
/algorithmic-art:algorithmic-art create a Mandelbrot set explorer
```

Or simply describe what you want:

> "Make me a flow field visualization with particle trails"

The skill automatically selects the best output mode based on your request, or you can specify one explicitly.

### Direct Mode Selection

Skip the mode selection prompt by passing a mode as the first argument:

```
/algorithmic-art:algorithmic-art shader raymarched mandelbulb fractal
/algorithmic-art:algorithmic-art p5 flow field with particle trails
/algorithmic-art:algorithmic-art scene 3D generative sculpture
/algorithmic-art:algorithmic-art thing perceptual color gradient art
/algorithmic-art:algorithmic-art webgpu particle life swarm with 50000 particles
```

| Mode Argument | Output |
|---------------|--------|
| `p5` | p5.js (HTML, 2D) |
| `thing` | thi.ng (HTML, 2D) |
| `scene` or `3d` | Three.js Scene (HTML, 3D) |
| `shader` or `glsl` | Three.js Shader (HTML, GLSL) |
| `webgpu` or `compute` or `wgsl` | WebGPU Compute + WGSL (HTML) |
| `nannou` or `rust` | nannou 2D (Rust) |
| `nannou3d` | nannou 3D (Rust) |

### Auto-Open

Generated HTML files automatically open in your default browser when written, so you see results instantly.

### Reduced Permission Prompts

The skill declares its required tools (`Read`, `Write`, `Glob`, `Grep`, `WebSearch`, `WebFetch`) upfront, reducing interruptions during the creative workflow.

## Output Modes

| Mode | Format | Best For |
|------|--------|----------|
| **p5.js** | HTML (2D) | Interactive 2D sketches, quick iteration, particle systems, generative agents |
| **thi.ng** | HTML (2D) | Perceptual color (Oklab/Oklch), 2D SDF composition, plotter-ready SVG, functional workflows |
| **Three.js Scene** | HTML (3D) | 3D generative sculptures, procedural landscapes, particle systems with scene graph and OrbitControls |
| **Three.js Shader** | HTML (GLSL) | Raymarching, PBR, volumetric rendering, 3D SDF fractals, ocean/terrain/atmosphere, GPU simulation, fullscreen shader art |
| **WebGPU** | HTML (WGSL) | Compute shaders, WGSL, large-scale particle systems (Particle Life, MPM fluid), compute-heavy simulation that outgrows WebGL's ping-pong model |
| **nannou 2D** | Rust | High-performance 2D creative coding, compiled applications, high-res print output |
| **nannou 3D** | Rust | 3D generative art with wgpu, perspective camera, compiled performance |

### WebGPU Mode Highlights

The WebGPU mode enables work that WebGL cannot sustain:

- **Compute + render pipeline** — three sketch hooks (`sketchSetup`, `computePass`, `renderPass`) for compute-driven simulation with instanced rendering
- **WGSL shaders** — modern shading language with compute entry points, storage buffers, and atomic operations
- **Large particle counts** — 100,000+ particles at 60fps on mid-range GPUs
- **Particle Life** — asymmetric-force color-group swarms (predator-prey, orbit, membrane dynamics)
- **MLS-MPM fluid** — particle-based fluid simulation with free surfaces, splashes, and droplets
- **Graceful fallback** — shows a friendly message if WebGPU isn't available (Firefox currently requires a flag)

Requires Chrome/Edge/Safari 17+ for default WebGPU support. Firefox users can enable `dom.webgpu.enabled` in `about:config`.

### thi.ng Mode Highlights

The thi.ng mode includes a built-in utility library (`tng`) with:

- **Perceptual color** — Oklab/Oklch color space, cosine gradient palettes (7 presets), Oklab interpolation
- **2D Signed Distance Fields** — Primitives (circle, box, hexagon, ring, segment), smooth boolean ops (union/subtract/intersect), domain ops (repeat, polar, mirror, rotate)
- **Seeded PRNG** — SFC32 algorithm for deterministic, reproducible output from any seed
- **Noise** — Seeded 2D Perlin noise with fractal Brownian motion (fbm)
- **Vector math** — 2D vector operations (add, sub, scale, rotate, normalize, etc.)

## Supported Techniques

| Family | Description |
|--------|-------------|
| Fractals & L-systems | Mandelbrot, Julia sets, recursive growth, tree grammars |
| Reaction-diffusion | Turing patterns, Gray-Scott model, morphogenesis |
| Flow fields & noise | Perlin/simplex noise, curl noise, noise derivatives, particle traces |
| Cellular automata | Wolfram rules, Conway's Game of Life, emergence |
| Strange attractors | Lorenz, Clifford, de Jong, deterministic chaos |
| Tiling & tessellation | Penrose, Truchet, aperiodic patterns, Islamic geometry |
| Recursion & subdivision | Mondrian-style, quadtree, space partitioning |
| Generative agents | Flocking, steering behaviors, autonomous agents |
| SDF composition | 2D/3D signed distance fields, smooth booleans, domain repetition |
| Shader art | GLSL fragment shaders, raymarching, volumetric effects |
| Perceptual color art | Oklab gradients, cosine palettes, LCH theme generation |
| Voronoi / cellular noise | Worley noise, distance metrics, F1/F2 patterns, edge detection |
| Procedural landscapes | Terrain raymarching, ridged noise, biome materials, atmospheric scattering |
| Water & ocean | Gerstner waves, Fresnel reflection, subsurface scattering, caustics |
| Path tracing / PBR | Cook-Torrance BRDF, global illumination, importance sampling |
| GPU simulation | Multipass ping-pong buffers for reaction-diffusion, fluid, cellular automata on GPU |
| Post-processing | Bloom, vignette, chromatic aberration, film grain, CRT, tone mapping |
| Line art & contours | Silhouette/crease extraction, screen-space edge detection, toon shading, SVG export |
| Superformula | Gielis parametric shapes (2D + 3D supershapes), organic form generation, parameter morphing |
| GPU particle systems | GPGPU particles via GPUComputationRenderer, texture-based simulation, curl noise forces |
| Procedural geometry | Vertex modifiers (twist, taper, noise displace, spherify, extrude-along-curve) |
| Analytic ray tracing | Exact ray-primitive intersection, refraction, reflection |
| WebGPU compute | WGSL compute shaders, storage buffers, ping-pong compute, atomic P2G scatter writes |
| MPM fluid | Material Point Method (MLS-MPM), particle-based fluid with splashes and free surfaces |
| Particle Life | Asymmetric-force color-group swarms with species force matrix and emergent dynamics |
| Plotter workflow | AxiDraw path planning (vpype/Saxi), pen/medium selection, multi-pass, watercolor-with-plotter |
| 3D isosurface extraction | Naive Surface Nets (full implementation), TPMS catalog (gyroid, Schwarz P/D, Neovius), Mandelbulb / Voronoi / metaball implicits, CSG composition |
| 3D Gaussian Splats | Direct construction of `.ply` (degree-0 SH) from algorithmic generators (attractors, flow fields, gradients), anisotropic frames, in-browser approximate billboards |
| Order-independent transparency | Weighted Blended OIT (McGuire 2013) for self-overlapping geometry and mixed alpha+additive scenes, depth-pre-pass for occlusion correctness |

## Features

- Full-viewport canvas with responsive resizing
- Interactive sidebar with parameter controls (sliders, dropdowns, color pickers, toggles)
- Seed system for reproducible outputs across all modes
- PNG export (all modes) and SVG export (p5.js, thi.ng)
- Pause, reset, and seed navigation controls
- Extensible reference library covering color science, SDFs, shaders, tiling, noise, and more

## Shader Technique Routing

For Three.js Shader mode, the skill includes an intent-based routing table that maps natural language requests to the right combination of reference files. For example:

- "procedural landscape with sunset" routes to terrain-rendering + atmospheric-scattering + post-processing
- "GPU reaction-diffusion" routes to multipass-buffers + shaders-glsl
- "glass spheres with refraction" routes to analytic-raytracing + shaders-glsl + post-processing
- "ink drawing of geometric forms" routes to line-art-contours + threejs-3d + post-processing
- "cel shaded cartoon scene" routes to line-art-contours (toon + outlines) + threejs-3d
- "particle life" or "asymmetric swarm" routes through WebGPU mode to particle-life + webgpu-compute + color-science
- "MPM fluid" or "particle fluid" routes through WebGPU mode to mpm-fluid + webgpu-compute

Quick recipes compose multiple techniques for common goals like photorealistic SDF scenes, procedural landscapes, GPU simulation art, organic forms, abstract shader art, line art / ink drawings, toon / cel shaded scenes, large-scale WebGPU particle simulations, and plotted hybrid pieces.

## Reference Library

The skill includes 40 deep reference files covering:

**Algorithm families:**
- **Color science** — Oklab/Oklch perceptual spaces, cosine gradient palettes, harmonic schemes
- **2D SDFs** — Primitives, boolean operations, domain transforms, rendering techniques
- **Tiling & tessellation** — Composable tessellation algorithms, SDF-based tiling, data-driven geometry
- **Flow fields & noise** — Curl noise, fbm, noise derivatives/gradients, functional composition patterns
- **Fractals & L-systems** — SDF-based fractals, domain folding, L-system grammars
- **Cellular automata** — Functional CA patterns, multi-state coloring
- **Strange attractors** — Functional iteration, perceptual color mapping
- **Generative agents** — Flocking, steering behaviors, autonomous agents

**Shader techniques:**
- **GLSL shaders** — SDFs, raymarching, PBR lighting, tone mapping, polar UV, noise, easing, fog, blending
- **Multipass buffers** — Ping-pong framebuffers for GPU simulation (fluid, cellular automata, reaction-diffusion)
- **Post-processing** — Bloom, vignette, chromatic aberration, film grain, CRT, color grading
- **Voronoi noise** — Cellular noise, distance metrics, F1/F2 patterns, 3D Voronoi
- **Path tracing** — Monte Carlo GI, Cook-Torrance PBR, importance sampling, progressive accumulation
- **Atmospheric scattering** — Rayleigh/Mie physical sky, aerial perspective, god rays
- **Water & ocean** — Gerstner waves, Fresnel, subsurface scattering, caustics, foam
- **Terrain rendering** — Procedural heightfields, ridged noise, biome materials, terrain raymarching
- **Anti-aliasing** — Supersampling (RGSS, stochastic), analytical AA with fwidth, temporal AA
- **Procedural 2D patterns** — Checkerboard, brick, hex grid, Truchet, stripes in GLSL
- **Analytic ray tracing** — Exact ray-primitive intersection, reflection, refraction
- **Line art & contours** — Silhouette/crease/boundary extraction, screen-space edge detection, inverted-hull outlines, toon/cel shading, hatching, SVG vector export
- **Sound synthesis** — Shader audio, oscillators, envelopes, WebAudio integration
- **WebGL pitfalls** — Precision issues, common bugs, visual debugging, mobile compatibility

**New techniques (inspired by Cinder C++ framework):**
- **GPU particles** — GPGPU particle systems via GPUComputationRenderer, Verlet integration, attractor forces, curl noise flow fields
- **Superformula** — Gielis parametric equation for organic shapes (starfish, flowers, gears, blobs), 2D/3D implementations, parameter morphing presets

**WebGPU / compute techniques:**
- **WebGPU compute** — WGSL primer, compute pipeline model, storage buffers, uniform buffer layout, atomic P2G pattern, library ecosystem (POINTS, Three.js TSL, Slang)
- **MPM fluid** — Material Point Method / MLS-MPM for particle-based fluid, 5-pass frame structure, quadratic B-spline weights, fluid/elastic/plastic constitutive models
- **Particle Life** — Asymmetric-force color-group swarms with force-matrix presets (predator-prey, orbit, membrane, swarm), O(N²) and spatial-hash scaling

**Hybrid digital/analog:**
- **Plotter workflow** — AxiDraw path planning with vpype/Saxi, pen/medium selection (fineliner, brush, fountain, gel, paint, dip), multi-pass color, watercolor-with-plotter, riso integration, contemporary practitioners

**3D scene composition:**
- **Isosurface extraction** — 3D scalar fields → mesh via Naive Surface Nets (~80 lines, no lookup tables), TPMS catalog (gyroid, Schwarz P/D, Neovius), quaternion fractals, Voronoi-based implicits, metaballs, CSG operators on SDFs
- **Gaussian splats** — 3D Gaussian Splatting `.ply` format spec (17 floats, degree-0 SH), direct construction from algorithmic generators (attractor trajectories → ribbons, flow fields → streaks, SDF gradients → oriented fur), anisotropic frame patterns, in-browser approximate rendering, tools that consume the format (SuperSplat, mkkellogg, Babylon, Brush)
- **Order-independent transparency** — Weighted Blended OIT (McGuire 2013) for mixing alpha-blended geometry with additive particles in self-overlapping scenes; 6-phase pipeline with shared depth texture, alpha-depth pre-pass for additive occlusion, material-variant pattern, `renderer.render` override technique

**Platform references:**
- **thi.ng ecosystem** — Functional creative coding, composable tessellation, genart-api
- **Three.js 3D** — Scene graph, geometry, materials, lighting, instanced meshes, procedural geometry modifiers (twist, taper, noise displace, extrude-along-curve)
- **nannou 2D/3D** — Rust creative coding, wgpu pipelines, WGSL shaders

## Contemporary Practitioners

The `sources.md` reference includes a curated roster of 2020s practitioners across long-form / on-chain (Tyler Hobbs, Matt DesLauriers, Zancan, William Mapan, Melissa Wiederrecht, ciphrd, Licia He), plotter / hybrid (LB Allix, CMD_DRAW, Medusa Gen, Barry Spencer, Reuben, Mechanic Art, Michelle Chandra / Dirt Alley Design), WebGPU / technical (Hector Arellano, matsuoka-601, Mustafa Ali, Absulit), and educators (Daniel Shiffman, Daniel Catt, Matt DesLauriers workshops, Frontend Masters). Used for art-historical context when producing pieces.

## License

MIT
