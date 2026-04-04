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
| **Three.js Shader** | HTML (GLSL) | Raymarching, volumetric rendering, 3D SDF fractals (Mandelbulb, Menger), fullscreen shader art |
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

## Features

- Full-viewport canvas with responsive resizing
- Interactive sidebar with parameter controls (sliders, dropdowns, color pickers, toggles)
- Seed system for reproducible outputs across all modes
- PNG export (all modes) and SVG export (p5.js, thi.ng)
- Pause, reset, and seed navigation controls
- Extensible reference library covering color science, SDFs, shaders, tiling, noise, and more

## Reference Library

The skill includes deep reference material on:

- **Color science** — Oklab/Oklch perceptual spaces, cosine gradient palettes, harmonic schemes
- **2D SDFs** — Primitives, boolean operations, domain transforms, rendering techniques
- **GLSL shaders** — Easing functions, lighting models, noise, blending, SDF operators
- **Tiling & tessellation** — Composable tessellation algorithms, SDF-based tiling, data-driven geometry
- **Flow fields & noise** — Curl noise, fbm, functional composition patterns
- **Fractals & L-systems** — SDF-based fractals, domain folding, L-system grammars
- **Cellular automata** — Functional CA patterns, multi-state coloring
- **Strange attractors** — Functional iteration, perceptual color mapping
- **thi.ng ecosystem** — Patterns from the thi.ng/umbrella monorepo and genart-api

## License

MIT
