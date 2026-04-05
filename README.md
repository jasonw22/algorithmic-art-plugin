# Algorithmic Art Plugin for Claude Code

Create algorithmic art as standalone files with interactive parameter controls and export capabilities. Supports six output modes spanning 2D, 3D, shaders, and compiled Rust.

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

## Output Modes

| Mode | Format | Best For |
|------|--------|----------|
| **p5.js** | HTML (2D) | Interactive 2D sketches, quick iteration, particle systems, generative agents |
| **thi.ng** | HTML (2D) | Perceptual color (Oklab/Oklch), 2D SDF composition, plotter-ready SVG, functional workflows |
| **Three.js Scene** | HTML (3D) | 3D generative sculptures, procedural landscapes, particle systems with scene graph and OrbitControls |
| **Three.js Shader** | HTML (GLSL) | Raymarching, PBR, volumetric rendering, 3D SDF fractals, ocean/terrain/atmosphere, GPU simulation, fullscreen shader art |
| **nannou 2D** | Rust | High-performance 2D creative coding, compiled applications, high-res print output |
| **nannou 3D** | Rust | 3D generative art with wgpu, perspective camera, compiled performance |

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
| Flow fields & noise | Perlin/simplex noise, curl noise, particle traces |
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
| Analytic ray tracing | Exact ray-primitive intersection, refraction, reflection |

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

Quick recipes compose multiple techniques for common goals like photorealistic SDF scenes, procedural landscapes, GPU simulation art, organic forms, and abstract shader art.

## Reference Library

The skill includes 29 deep reference files covering:

**Algorithm families:**
- **Color science** — Oklab/Oklch perceptual spaces, cosine gradient palettes, harmonic schemes
- **2D SDFs** — Primitives, boolean operations, domain transforms, rendering techniques
- **Tiling & tessellation** — Composable tessellation algorithms, SDF-based tiling, data-driven geometry
- **Flow fields & noise** — Curl noise, fbm, functional composition patterns
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
- **Sound synthesis** — Shader audio, oscillators, envelopes, WebAudio integration
- **WebGL pitfalls** — Precision issues, common bugs, visual debugging, mobile compatibility

**Platform references:**
- **thi.ng ecosystem** — Functional creative coding, composable tessellation, genart-api
- **Three.js 3D** — Scene graph, geometry, materials, lighting, instanced meshes
- **nannou 2D/3D** — Rust creative coding, wgpu pipelines, WGSL shaders

## License

MIT
