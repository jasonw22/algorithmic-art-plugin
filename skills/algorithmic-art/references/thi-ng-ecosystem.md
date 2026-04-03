# thi.ng/umbrella — Functional Creative Coding Ecosystem

## Overview

**thi.ng/umbrella** is a monorepo of 214+ composable TypeScript/JavaScript packages for
functional programming, data-driven development, and creative coding. Created by Karsten Schmidt
(toxi/toxiclibs fame), it represents a fundamentally different approach to generative art:
functional composition over imperative state, data as shapes over class hierarchies, and
multi-target rendering from a single geometry description.

**thi.ng/genart-api** is a companion project — a platform-independent API for browser-based
generative art that decouples artworks from specific presentation platforms (fx(hash), EditArt,
galleries, personal sites). Switching platforms requires only swapping a `<script>` tag.

## Philosophy & Design Principles

Unlike p5.js (imperative, global state, setup/draw loop) or Three.js (OOP scene graph),
thi.ng is:

- **Functional**: No global state. Everything is composable pure functions and immutable data.
- **Data-driven**: Geometry is data (plain arrays/objects), not class instances. The hiccup
  format (nested arrays as markup) means the same shape data renders to Canvas, SVG, or WebGL
  without modification.
- **Modular**: Each of 214 packages is independently publishable and tree-shakeable. You only
  bundle what you use.
- **Multi-target**: Same code renders to Canvas 2D, SVG (for plotters/print), and WebGL.

## Art-Relevant Packages

### Geometry & Spatial
- **@thi.ng/geom** — 30+ polymorphic 2D/3D shape types with 60+ operations (transform,
  tessellate, boolean ops, convex hull, simplify, resample, offset, subdivision)
- **@thi.ng/geom-tessellate** — 9 tessellation algorithms: ear cut, quad fan, edge split,
  inset, rim triangles, tri fan variants. Supports recursive/iterative multi-pass tessellation.
- **@thi.ng/geom-sdf** — 2D signed distance fields from geometry shapes. SDF combinators
  (union, difference, intersection with chamfer/round/smooth blending). Domain modifiers
  for repetition, mirroring, polar patterns. Contour extraction back to polygons.
- **@thi.ng/geom-voronoi** — Voronoi diagrams and Delaunay triangulation

### Vectors & Math
- **@thi.ng/vectors** — ~900 functions for n-dimensional vector math with specialized
  loop-free 2D/3D/4D variants. Supports strided/SOA memory layouts.
- **@thi.ng/math** — Clamping, stepping, mixing/lerp, modular arithmetic
- **@thi.ng/matrices** — 2x3, 3x3, 4x4 matrix operations

### Color
- **@thi.ng/color** — 16 color models (sRGB, linear RGB, HSL, HSV, HSI, HCY, Lab D50/D65,
  LCH, Oklab, Oklch, XYZ, xyY, YCC). Features: CSS parsing, cosine gradients, multi-stop
  gradients in any color space, probabilistic theme generation with LCH ranges, harmonic
  color schemes (complementary, triadic, tetradic), color sorting, distance calculations.

### Noise & Randomness
- **@thi.ng/random** — Seedable PRNGs (SFC32, Xoshiro128, XorShift128, XorWow, XsAdd).
  Statistical distributions (Gaussian, exponential, geometric, uniform). Weighted random
  selection, unique index generation.

### Rendering
- **@thi.ng/hiccup-canvas** — Render hiccup shape trees to Canvas 2D (groups, shapes,
  gradients, paths). Direct integration with @thi.ng/geom via `IToHiccup`.
- **@thi.ng/hiccup-svg** — Generate SVG from hiccup structures. Same geometry renders to
  both Canvas and SVG without modification.
- **@thi.ng/pixel** — Typed array pixel buffers with multiple formats, blitting, filtered
  resize, channel manipulation, dithering, dominant color extraction.
- **@thi.ng/webgl** — WebGL 1/2 abstraction with declarative shader/geometry definitions.

### Shaders (GPU)
- **@thi.ng/shader-ast** — Write shaders in TypeScript, cross-compile to GLSL ES 1.0/3.0
  or JavaScript (CPU fallback). Type-checked AST, dead code elimination.
- **@thi.ng/shader-ast-stdlib** — ~230 portable shader functions: noise (simplex, hashes),
  SDF primitives (2D and 3D), raymarching helpers, lighting models, fog, easing, oscillators,
  Porter-Duff blending. Ported from Inigo Quilez, Ashima Arts, Dave Hoskins.

### Data & Composition
- **@thi.ng/transducers** — ~170 composable transducers, reducers, and generators. The
  backbone for functional data processing throughout the ecosystem.
- **@thi.ng/rstream** — Reactive streams for event handling and animation loops.

## Key Patterns Worth Adopting

### 1. Data-Driven Geometry → Multi-Target Rendering

The hiccup pattern (representing shapes as nested arrays) allows the same geometry description
to render to multiple targets:

```typescript
// Shape is just data — an array describing a circle
const shape = ["circle", { fill: "#f00", stroke: "#000" }, [100, 200], 50];

// Render to Canvas 2D
drawCanvas(ctx, shape);

// Or render to SVG string
const svg = serialize(convertTree(shape));
```

**Lesson for the skill**: When building geometric compositions (tilings, subdivisions,
L-systems), separate geometry generation from rendering. Store shapes as data structures,
then render them. This makes SVG export trivial — the same geometry that draws to canvas
can construct an SVG string.

### 2. Tessellation as Composable Operations

thi.ng/geom-tessellate treats tessellation as composable, stackable transformations:

```
polygon → earCut → triangles
polygon → inset(0.2) → smaller polygon + rim
polygon → edgeSplit → subdivided polygon
```

Operations can be chained: `polygon → inset → earCut → edgeSplit` produces increasingly
complex results from a simple starting shape.

**Lesson for the skill**: Tessellation-based art benefits from this composable approach.
Rather than hardcoding a specific tessellation, expose the tessellation strategy as a
parameter (dropdown: "Ear Cut", "Quad Fan", "Edge Split", "Inset") and apply recursively.

### 3. SDF Composition with Smooth Booleans

thi.ng/geom-sdf demonstrates constructing complex shapes from simple primitives via smooth
boolean operations with controllable blending:

- `smoothUnion(circle, rect, 0.1)` — organic merge
- `smoothSubtract(hole, shape, 0.05)` — soft carving
- Domain repetition: `repeat(shape, [spacing_x, spacing_y])` for infinite tiling
- Polar repetition: radial symmetry from a single shape

**Lesson for the skill**: See `references/sdf-2d.md` for full 2D SDF reference with
implementations in both JavaScript and GLSL.

### 4. Perceptual Color as Default

thi.ng/color's approach of working in Oklab/LCH by default (not sRGB) produces
dramatically better palettes:

- Cosine gradients for infinitely smooth palettes from 4 coefficient vectors
- LCH range constraints for probabilistic theme generation
- Harmonic schemes computed in perceptual space

**Lesson for the skill**: See `references/color-science.md` for full color science reference.
All palette generation in the skill should prefer perceptual interpolation.

### 5. GenArt API Parameter System

thi.ng/genart-api defines 17 parameter types beyond basic sliders:

- **Ramp**: Curve-based time-varying interpolation (linear/smooth/exponential). A parameter
  that changes over the lifetime of the piece — useful for evolving compositions.
- **WeightedChoice**: Selection from options with explicit probability weights
- **XY**: 2D normalized position parameter (click a point on a 2D plane)
- **Vector**: n-dimensional parameter
- **Image**: Grayscale/RGBA pixel buffer as parameter input

**Lesson for the skill**: The Ramp parameter concept is particularly interesting for animated
pieces — a value that automatically evolves over time along a user-defined curve, rather
than staying static or requiring manual keyframing.

### 6. Seedable PRNG Architecture

thi.ng uses SFC32 with 128-bit seeds, with the PRNG sourced from a platform adapter. This
ensures:
- Deterministic reproduction from seed
- Platform-independent results (same seed → same output regardless of where it runs)
- Clean separation between "randomness source" and "randomness consumer"

**Lesson for the skill**: The skill already uses seeded randomness, but the 128-bit seed
approach could provide better distribution for complex pieces that consume millions of
random values.

## Notable Example Projects

These thi.ng/umbrella examples demonstrate techniques directly applicable to algorithmic art:

| Example | Technique | Relevance |
|---------|-----------|-----------|
| `ifs-fractal` | Barnsley fern via IFS | Fractal rendering with functional transforms |
| `geom-tessel` | Animated recursive tessellations | Composable tessellation operations |
| `poly-subdiv` | Animated polygon subdivisions | Subdivision as iterative refinement |
| `geom-voronoi-mst` | Poisson-disk + Voronoi + MST | Combining spatial algorithms |
| `poisson-circles` | Poisson-disc sampling | Even spatial distribution |
| `color-themes` | Probabilistic LCH themes | Perceptual palette generation |
| `shader-ast-raymarch` | Raymarching in TypeScript | Type-safe shader authoring |
| `shader-ast-noise` | GPU noise functions | Portable noise implementations |
| `pixel-sorting` | Interactive pixel sorting | Image manipulation techniques |
| `pixel-dither` | Dithering algorithms | Ordered/error-diffusion dithering |
| `geom-terrain-viz` | 2.5D hidden-line terrain | Terrain visualization |
| `cellular-automata` | Transducer-based CA | Functional cellular automata |
| `geom-hexgrid` | Hex grid tessellations | Hexagonal grid generation |
| `layout-gridgen` | Randomized nested grids | Recursive layout subdivision |
| `canvas-recorder` | Animated typographic grid | Self-modifying compositions |

## When to Reference thi.ng Patterns

- **Tessellation art**: reference composable tessellation operations
- **Color palette design**: reference perceptual color (Oklab/LCH) and cosine gradients
- **SDF compositions**: reference smooth boolean operators and domain operations
- **Dual Canvas/SVG output**: reference data-driven geometry separation
- **Functional approach to generative art**: reference transducer-based composition

## Key Links

- **thi.ng/umbrella**: github.com/thi-ng/umbrella — 214 packages, ~185 examples
- **thi.ng/genart-api**: github.com/thi-ng/genart-api — platform-independent generative art API
- **Karsten Schmidt**: toxi.co — veteran creative coder, author of toxiclibs (Java) and thi.ng
