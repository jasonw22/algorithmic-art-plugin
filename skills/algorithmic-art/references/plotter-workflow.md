# Plotter Workflow — Hybrid Digital/Analog Reference

## Overview

A pen plotter is a two-axis machine that drives a pen across paper under direct control
from an SVG (or g-code-like language). The physical output is usually the point of the
piece, not a side product. The plotter scene is a distinct tradition inside generative
art, with its own aesthetics, materials, and failure modes.

This reference is for pieces whose final form is plotted on paper. Code that produces an
SVG suitable for the plotter is necessary but not sufficient — good plotter pieces
account for pen physics, path order, material behavior, and the interaction of ink with
paper. Use `references/thing-2d.md` when you need the SVG-export mechanics; use this
file when the **physical output matters as much as the algorithm**.

The framing worth holding in mind (paraphrased from contemporary practice): *the plotter
is the instrument, not the subject*. The goal is not to show that a machine made the
mark; it is to exploit the plotter's particular capabilities — consistent pressure,
fine lines, tireless repetition — in the service of a drawing.

## Hardware context

| Plotter | Paper size | Pen types | Typical price | Community |
|---------|-----------|-----------|---------------|-----------|
| **AxiDraw (V3, SE/A3, MiniKit)** | A4 up to A3 | Any — even brushes, dip pens, markers | $475–$1,200 | Large, active |
| **NextDraw** | A4 up to A2 | Same as AxiDraw; successor line | $500–$1,500 | Growing |
| **iDraw 2.0 / iDraw H** | A4 up to A2 | Any | $300–$700 | Medium |
| **EleksDraw, plot4u** | A4 | Limited to pen-holder fit | $150–$300 | Smaller |
| **AxiDraw "MiniKit" / CNC builds** | Variable | Variable | <$300 DIY | Maker-leaning |

This reference targets AxiDraw-class machines (the de-facto standard in the community).
Techniques transfer directly to NextDraw and iDraw; DIY machines may need their own
tuning.

## Software pipeline

```
Generative code (any mode)
    │
    ▼
  SVG output  ←─ The product of your sketch; see thing-2d.md's sketchSVG()
    │
    ▼
Path optimization  ← Visit order, merge, simplify
    │
    ▼
Plotter driver  ← AxiDraw: Inkscape extension or Saxi (browser-based)
    │
    ▼
Physical drawing
```

### Path optimization tools

- **vpype** (github.com/abey79/vpype) — Python CLI for plotter SVGs. Optimizations:
  `linemerge` (join touching endpoints), `linesort` (minimize pen-up travel with
  TSP/2-opt), `linesimplify` (reduce vertex count), `filter` (drop short lines). De
  facto standard.
- **vsketch** — Python library built on vpype for sketch-centric workflows. Good when
  generative code is already in Python.
- **Saxi** (github.com/nornagon/saxi) — Browser-based plotter driver with its own path
  optimizer built in. Simplest end-to-end for AxiDraw.
- **Inkscape + AxiDraw extension** — Official path; adequate but slower workflow.

A minimal post-export pipeline for a complex SVG:

```bash
vpype read sketch.svg \
      linemerge \
      linesort \
      linesimplify --tolerance 0.1mm \
      write sketch-plot.svg
```

This typically reduces total plot time 30–50% without any visual change.

## Generating plotter-friendly geometry

### Prefer polylines over rasterized paths

The SVG should consist of geometric paths — `<path>`, `<polyline>`, `<line>`, `<circle>`,
`<rect>` — **not** rasterized edge traces from a bitmap. Use the vector-native SVG
export route documented in `SKILL.md` (building an SVG string from your geometry data
and overriding the export button). Edge-detected SVGs from a bitmap produce thousands
of tiny disconnected segments that plot poorly.

### Line density vs plot time

A rough budget: AxiDraw plots ~0.5 meters of pen-down line per minute at default speed.
A piece with 500 meters of line takes ~15 hours. Plan for this. Fewer, more considered
lines often beat dense hatching.

### Hatching

For filled shapes, generate hatching paths in code rather than relying on SVG `fill`
attributes (plotters cannot fill — they only stroke). See `line-art-contours.md` for
the hatching implementation pattern. Typical hatch spacing: 0.3–1.0 mm depending on
pen width. Cross-hatching, contour-following hatches, and stippling are all worth
exploring.

### Curves

Plotters rasterize Bézier curves into polyline segments. If the SVG contains many short
Bézier segments, the plotter will move in staccato steps. Either:
- Emit smooth polylines (many vertices) directly, or
- Use `vpype linesimplify` to resample curves into uniform segments

### Registration marks

If the piece uses multiple plotter passes (color changes, multiple pens), include
registration marks — small crosses or ticks at fixed positions in every pass. These
let you re-register the paper between passes.

## Pens and media

The choice of pen and medium is where plotter practice becomes an art-material question.

### Pen families

| Pen type | Line quality | Lifetime | Good for |
|----------|--------------|----------|----------|
| **Fineliner / technical pen** (Staedtler Pigment Liner, Sakura Pigma Micron, Copic Multiliner, Rotring Rapidograph) | Clean, consistent | Very long (hours) | Technical drawings, hatching, fine detail. The default plotter pen |
| **Fountain pen** | Variable, expressive | Medium (requires refill) | Classic ink-on-paper look; handwriting emulation |
| **Brush pen** (Pentel Pocket Brush, Tombow ABT) | Highly variable, painterly | Shorter | Expressive, organic marks; pressure-variable plotters exploit this |
| **Gel pen** (Gelly Roll, Sakura Souffle) | Opaque, glossy | Medium | White/metallic on dark paper; rich saturated color |
| **Paint pen / Posca** | Opaque, thick | Medium, needs priming | Bold color on any paper; multi-layer coverage |
| **Dip pen** (nib + ink) | Wildly variable, beautiful | Very short (refills every few minutes) | Handheld-drawing feel; rewarded by proximity to the machine |
| **Watercolor brush in plotter** | Painterly, blurry | Requires refill strategy | See "Watercolor workflow" below |

### Paper

- **Smooth drawing paper / Bristol** (~90–250 gsm): the safe default for fineliners
- **Cotton hot-press watercolor paper** (300 gsm): essential for wet media; heavier
  tooth blurs lines slightly but survives multiple passes
- **Rives BFK, Arches** — printmaker's papers; excellent for plotted work destined for
  framing
- **Kraft / colored paper**: for gel-pen and paint-pen work; the paper color becomes
  part of the palette

Always tape the paper down at the corners. Paper movement mid-plot is the most common
way to ruin a long plot.

## Multi-pass workflows

A single-pass, single-pen plot is the entry point. The interesting territory is
multi-pass: multiple pen colors, or the same pen over several decisions.

### Color layers

Split the SVG into color-layered groups (via `<g inkscape:label="red">` or similar).
Plot one layer, pause, swap the pen, plot the next. `vpype layout` and Saxi both
understand multi-layer SVGs.

### Intentional misregistration

Slight intentional offset between passes reads as "risograph" (see below) — a
distinctive generative-print aesthetic.

### Pen-weight variation

Same color, different pen widths across passes produces a hierarchy of marks (wireframe
vs contour vs hatching). A good default: start with the finest pen (foundation lines),
build up with thicker pens (emphasis).

## Watercolor / brush workflow

Treating the plotter as a painting instrument rather than a drawing one — mounting a
watercolor brush or brush pen in the pen holder, and letting the medium's bleed and
drying contribute to the final piece — is a well-developed subgenre pioneered by Licia
He, Mechanic Art, and Reuben.

Core techniques:

- **Prime the brush, not the paper:** load the brush with pigment/water before starting
  each pass; the first few strokes of each run have distinct character
- **Exploit drying variance:** adjacent strokes that dry at different rates bleed into
  each other; plan path order to make this a feature
- **Let the plotter pause:** many practitioners write scripts that pause the plotter
  periodically so the artist can re-load the brush or add water manually
- **Low-speed settings:** AxiDraw pen-down speed around 10–25 mm/s lets watercolor
  settle properly
- **Non-flat pen lift:** some brush workflows use custom G-code that keeps the brush
  *in contact* with the paper through what would otherwise be a travel move,
  producing dragged tails

The aesthetic goal is usually ambiguity between the plotter's consistency and the
medium's unpredictability. A piece that looks clearly "machine-made" or clearly
"handmade" is probably missing the point.

## Riso integration

Risograph printing pairs beautifully with plotted originals — the riso flattens
subtle variation and emphasizes color-layer structure, while still preserving the
hand-made feel. A common pipeline (popularized by Michelle Chandra / Dirt Alley
Design):

1. Plot one-color originals for each layer (cyan, magenta, pink, yellow)
2. Scan each plotted sheet
3. Send the scans to a riso printer for layered reproduction
4. Produce short editions (10–30 copies)

This produces editioned prints rather than unique drawings — a viable small-scale
commercial model.

## Plotter as performance

Many plotter artists foreground the act of plotting: running the plot during gallery
hours, live-streaming the draw, or building installations where the plotter is part
of the piece. Matt DesLauriers's *Pattern Language* (2024) and *The Sferic Project*
are recent examples. For this kind of presentation, plot time, sound, and motion
become part of the design.

## AxiDraw-specific settings cheat sheet

| Setting | Default | Notes |
|---------|---------|-------|
| Pen-up position | 60% | Higher = more travel clearance; wears servo |
| Pen-down position | 40% | Lower = harder press; depends on pen stiffness |
| Pen-up speed | 75% | |
| Pen-down speed (drawing) | 25% | Slow down for wet media; up to 80% for dry fine liner |
| Acceleration | 75% | Lower for brittle pen tips |
| Servo timeout | 60 s | Pen lifts after idle time |

Adjust pen-down position per pen type — stiff fineliners want 42–45%; brushes need
much less contact force (65–72%).

## Practitioners worth studying

- **Licia He** (liciahe.com) — Treats AxiDraw as a painting instrument; custom software
  drives pen-and-watercolor compositions
- **Mechanic Art** — Plotter-based brush painting with watercolor and acrylic; "plotter
  as instrument, not subject"
- **Reuben** — Brush pens on large-format plotters for painterly work
- **Michelle Chandra — Dirt Alley Design** (dirtalleydesign.com) — Draws each print to
  order; riso pipeline; extensive practical tutorials
- **LB Allix** — Daily plotter practice
- **CMD_DRAW** — Plotter combined with 3D and motion work
- **Barry Spencer** — Generative typography + ceramics (plotter + physical craft)
- **Matt DesLauriers** — *Meridian*, *Pattern Language*, *Sferic Project*; plots as part
  of installations

For curated surveys: **UUNA TEK** (uunatek.com) publishes regular artist lists;
**Generative Hut** published *Tracing the Line* (100-artist plotter anthology, 2024) —
the best single reference for how the plotter scene presents itself physically.

## Key references

- **Generative Hut** (generativehut.com) — Publications and plotter-scene community
- **Pen Plotter Artwork** (penplotterartwork.com) — Artist profiles + tutorials
- **Dirt Alley Design blog** (dirtalleydesign.com) — Pen selection, watercolor, riso
- **vpype docs** (vpype.readthedocs.io) — Path-optimization CLI
- **AxiDraw user guide** (evilmadscientist.com/2021/axidraw-documentation/) — Hardware
  reference
- `thing-2d.md` — SVG export mechanics via `sketchSVG()`
- `line-art-contours.md` — Generating plotter-friendly geometry (silhouettes, hatching,
  edges)
- `sources.md` — Contemporary practitioners and publications
