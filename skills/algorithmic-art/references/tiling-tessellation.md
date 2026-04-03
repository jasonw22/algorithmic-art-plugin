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

## Tessellation Algorithms

Beyond tiling (covering a plane with shapes), **tessellation** subdivides existing shapes into
smaller pieces. These are composable — apply one, then another, recursively:

### Ear Clipping
Decompose any simple polygon into triangles by repeatedly cutting "ear" triangles (convex
vertices whose diagonal lies inside the polygon). The standard polygon triangulation method.

### Quad Fan
Split a polygon into quads radiating from the centroid. Each edge of the original polygon
becomes the base of a quad, with the centroid as the opposite edge. Produces pinwheel-like
patterns.

### Edge Split
Add a vertex at each edge midpoint, then retriangulate. Each triangle becomes 4 smaller
triangles. Repeated application produces smooth, regular subdivision.

### Inset
Shrink a polygon inward by a factor, producing a smaller copy inside and a "rim" between
the original and the inset. The rim can be further tessellated (ear cut, quad fan) for
decorative effect. Recursive inset creates nested frames.

### Rim Triangles
After insetting, triangulate the rim between the original polygon and the inset polygon.
Connects corresponding vertices of outer and inner polygons with triangles.

### Composable Tessellation Patterns

Chain operations for increasingly complex results:

```
hexagon → inset(0.15) → [inner hex, rim]
  rim → ear cut → triangles → inset(0.1) each → nested triangles
  inner hex → quad fan → quads → inset(0.2) each → framed quads
```

This composable approach (inspired by thi.ng/geom-tessellate) lets you expose tessellation
strategy as a parameter. A dropdown like "Style: Inset + Ear Cut / Quad Fan / Edge Split"
dramatically changes the visual character from the same starting geometry.

**Parameters**: tessellation method, inset factor, recursion depth, alternation pattern

## SDF-Based Tiling

Signed distance fields offer a different approach to tiling — instead of placing discrete
shapes, evaluate a distance function with domain repetition:

```javascript
// Infinite grid of rounded squares via SDF + domain repetition
function tiledSDF(px, py, spacing, cornerRadius) {
  // Repeat domain
  const rx = ((px % spacing) + spacing) % spacing - spacing * 0.5;
  const ry = ((py % spacing) + spacing) % spacing - spacing * 0.5;
  // Rounded box SDF
  return sdRoundedBox(rx, ry, spacing * 0.35, spacing * 0.35, cornerRadius);
}
```

SDF tiling enables effects impossible with discrete geometry:
- **Smooth morphing** between tile shapes (interpolate two SDF functions)
- **Organic boolean operations** (smooth union/subtraction between tiles)
- **Distance-based coloring** (contour rings, glow, gradient fill based on distance)
- **Polar tiling** (radial repetition creates mandala-like patterns)

See `references/sdf-2d.md` for full 2D SDF primitives, boolean operations, and domain
manipulation functions.

## Data-Driven Geometry for SVG Export

For tiling and tessellation art, separate geometry generation from rendering (a key pattern
from the thi.ng ecosystem). Store tiles as data structures first, then render:

```javascript
// Generate tiles as data
const tiles = [];
for (let row = 0; row < rows; row++) {
  for (let col = 0; col < cols; col++) {
    tiles.push({
      type: random() > 0.5 ? "arc-left" : "arc-right",
      x: col * size, y: row * size,
      vertices: computeVertices(col, row, size),
    });
  }
}

// Render to canvas (in sketchDraw)
for (const tile of tiles) {
  drawTileToCanvas(p, tile);
}

// Export to SVG (override export button)
function exportSVG() {
  let svg = `<svg xmlns="http://www.w3.org/2000/svg" width="${w}" height="${h}">`;
  for (const tile of tiles) {
    svg += tileToSVGPath(tile);
  }
  svg += '</svg>';
  // ... download blob
}
```

This produces clean, resolution-independent vector output — essential for print and plotter work.
