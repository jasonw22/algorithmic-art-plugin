---
name: algorithmic-art
description: >
  Use this skill to produce standalone p5.js HTML files whenever a user wants visual art, beauty,
  or aesthetic exploration through code and math. This is the default skill for ANY request where
  the primary goal is creating something visually compelling through computation. Trigger for:
  generative art, fractal explorers, beautiful cellular automata (Conway's Game of Life),
  procedural patterns, particle animations, Penrose tilings, tessellations, flow fields, L-systems,
  reaction-diffusion, strange attractors, Mandelbrot/Julia sets, kaleidoscopes, geometric patterns,
  noise-based visuals, organic growth simulations, recursive subdivision, album/poster art from code,
  animated backgrounds from particles/algorithms, interactive parameter-driven sketches, high-res
  generative prints, and educational math visualizations. Also trigger when users want to make
  something "look beautiful" or "cool" using algorithms, even without technical terminology.
  Skip for: games with win/lose mechanics, GLSL/shader-only work, data dashboards, general
  web/UI design, or algorithm implementations focused on correctness not aesthetics.
---

# Algorithmic Art

Create algorithmic art as standalone HTML files using p5.js, with interactive parameter controls
and both bitmap (PNG) and vector (SVG) export.

Every piece of algorithmic art expresses an idea — a philosophy about emergence, order, chaos,
nature, or perception. When creating a piece, always articulate the concept driving it, not just
the technique. The technique serves the idea.

## Workflow

When a user requests algorithmic art:

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
3. **Parameter guide** — what each slider does and interesting ranges to explore
4. **Art historical context** — connections to artists, movements, or foundational work
5. **The HTML file** — saved and ready to open in a browser

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
