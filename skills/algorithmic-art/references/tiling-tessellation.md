# Tiling & Tessellation

## Philosophy

Tessellation explores **how shapes fill space** — a question that bridges mathematics, architecture,
and decorative art across millennia. Islamic geometric art perfected complex tilings centuries before
Western mathematics formalized them, encoding spiritual ideas about infinity and divine order into
interlocking stars and polygons.

Penrose tilings (1974) shattered the assumption that only periodic patterns can tile a plane.
Their **aperiodic order** — structured but never repeating — prefigured the discovery of
quasicrystals in nature. Truchet tiles demonstrate that **maximum variety from minimum means**
is possible: a single tile with a simple asymmetry, placed randomly, generates rich visual texture.

## Key Algorithms

### Truchet Tiles
A square tile divided by a diagonal arc (quarter-circle in two opposite corners).
Place randomly rotated on a grid. Adjacent arcs connect to form winding paths.

Variations:
- **Classic**: quarter-circle arcs, two orientations
- **Multi-scale**: recursive Truchet (subdivide tiles that are themselves Truchet)
- **Smith tiles**: triangular divisions instead of arcs
- **10PRINT**: The classic `10 PRINT CHR$(205.5+RND(1)); : GOTO 10` — diagonal lines

**Parameters**: grid size, tile size, arc style, randomness/seed, line weight

### Penrose Tiling (P3 — Rhombus)
Two rhombus shapes (thin and thick) with matching rules that enforce aperiodicity.
Generated via **deflation** (subdivision): start with a large shape, recursively split
into smaller copies following specific geometric rules.

**Parameters**: deflation depth, scale, coloring rule, stroke weight

### Islamic Geometric Patterns
Constructed from a **tessellation of regular polygons** (usually on a square or hexagonal grid),
with lines connecting midpoints of polygon edges. The overlay pattern forms stars and rosettes.

Construction steps:
1. Lay out a base grid of polygons
2. Find midpoints of edges
3. Connect midpoints with straight lines following angle rules
4. The resulting network forms the geometric pattern

**Parameters**: grid type (square, hexagonal), polygon sides, contact angle, pattern depth

### Voronoi / Delaunay
Voronoi: divide plane into regions closest to each seed point.
Delaunay: triangulation that is the dual of Voronoi.
Together they produce organic cell-like structures.

**Parameters**: seed count, seed distribution (random, Poisson disc, grid-jittered), line weight

### Wang Tiles
Small set of square tiles with colored edges. Tiles can only be placed adjacent if edge colors
match. Different tile sets produce different textures — a way to create seamless infinite patterns.

## Notable Artists & Works

- **M.C. Escher** — the master of tessellation art, bridging impossible geometry and pattern
- **Roger Penrose** — aperiodic tiling, quasicrystal mathematics
- **Jean-Marc Castera** — contemporary Islamic geometric pattern design
- **Craig Kaplan** — computational approaches to Islamic star patterns
- **Christoph Lauter** — parametric Truchet tile explorations

## p5.js Implementation Notes

- Render mode: **svg** (all tile-based patterns are inherently vector — lines, arcs, polygons)
- Truchet: nested `for` loops over grid, randomly choose rotation, draw arcs with `arc()`.
- Penrose: implement deflation recursively. Store tiles as vertex lists, subdivide.
  Most elegant as a recursive function that takes a triangle/rhombus and returns children.
- Islamic patterns: use `beginShape()` / `vertex()` / `endShape()` for the star networks.
  Compute geometry from polar coordinates centered on each polygon.
- Voronoi: use Fortune's algorithm or brute-force for small counts. For p5.js, the
  `d3-delaunay` library can be loaded from CDN and interoperates well.
- Color: tiling lends itself to two-coloring (alternating), gradient mapping by position,
  or color-by-tile-type for Penrose (thin vs thick rhombus).
- Scale matters: start with visible tile size (20-50px) so structure is legible.
