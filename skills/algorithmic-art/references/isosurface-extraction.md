# Isosurface Extraction (3D SDFs → Mesh)

## Philosophy

A 3D scalar field — sampled on a regular grid — defines an implicit surface
wherever the field crosses a chosen *isovalue*. Mesh extraction turns that
implicit surface into vertices and triangles you can render, intersect, or
3D-print. The technique unlocks an entire class of mathematical objects that
exist only as equations: triply-periodic minimal surfaces (TPMS), quaternion
fractals, summed-noise blobs, gradient-field topographies.

Why mesh them at all when raymarching the SDF directly is also possible?
Three reasons that matter for art:

1. **Composition** — meshes drop into a scene graph alongside particles,
   splats, instanced geometry. Raymarched scenes live in their own
   fullscreen shader and don't compose with object-space content.
2. **Lighting** — proper PBR / shadows / transparency on a mesh; raymarchers
   require hand-rolling all of it inside the shader.
3. **Export** — once meshed you can write glTF, save for Blender, send to a
   3D printer or laser cutter.

## The Field-Sampling Pattern

Always the same shape: pick bounds and a resolution, evaluate the field at
every grid corner, store flat in a Float32Array (or numpy array).

```javascript
function sampleField(field, dims, bounds, fn) {
  const [nx, ny, nz] = dims;
  const dx = (bounds.max[0] - bounds.min[0]) / (nx - 1);
  const dy = (bounds.max[1] - bounds.min[1]) / (ny - 1);
  const dz = (bounds.max[2] - bounds.min[2]) / (nz - 1);
  let i = 0;
  for (let z = 0; z < nz; z++) {
    const wz = bounds.min[2] + z * dz;
    for (let y = 0; y < ny; y++) {
      const wy = bounds.min[1] + y * dy;
      for (let x = 0; x < nx; x++) {
        const wx = bounds.min[0] + x * dx;
        field[i++] = fn(wx, wy, wz);
      }
    }
  }
}
```

Resolution scales as O(N³) — 64³ ≈ 260k samples (cheap), 128³ ≈ 2M (~1 second
in JS), 256³ ≈ 16M (workers / wasm territory). Pick the smallest grid that
captures features at the scale you care about.

Pull common loop-invariant trig/expensive math out of inner loops — for
periodic fields like the gyroid, precompute `sin(k·wx)` per row.

## Naive Surface Nets

Mikola Lysenko's **Naive Surface Nets** is the simplest workable extractor
that produces clean, smooth output. No 256-entry lookup tables (unlike
Marching Cubes), no template specializations (unlike Dual Marching Cubes).
~80 lines of JS, two passes:

1. **Pass A** — for every cell whose 8 corners straddle the isovalue, place
   one vertex at the centroid of the edge-zero crossings.
2. **Pass B** — for every axis-aligned grid edge with a sign change, emit a
   quad connecting the four cell-centroid vertices that share that edge.

```javascript
function naiveSurfaceNets(field, dims, bounds, isovalue = 0) {
  const [nx, ny, nz] = dims;
  const idx = (x, y, z) => x + nx * (y + ny * z);
  const cellVertex = new Int32Array(nx * ny * nz).fill(-1);

  const cornerOffsets = [
    [0,0,0],[1,0,0],[1,1,0],[0,1,0],
    [0,0,1],[1,0,1],[1,1,1],[0,1,1],
  ];
  const edgeList = [
    [0,1],[1,2],[2,3],[3,0],
    [4,5],[5,6],[6,7],[7,4],
    [0,4],[1,5],[2,6],[3,7],
  ];

  const positions = [];
  const indices = [];

  // Pass A — vertex per cell with sign changes
  for (let z = 0; z < nz - 1; z++)
  for (let y = 0; y < ny - 1; y++)
  for (let x = 0; x < nx - 1; x++) {
    const c = [
      field[idx(x,   y,   z  )], field[idx(x+1, y,   z  )],
      field[idx(x+1, y+1, z  )], field[idx(x,   y+1, z  )],
      field[idx(x,   y,   z+1)], field[idx(x+1, y,   z+1)],
      field[idx(x+1, y+1, z+1)], field[idx(x,   y+1, z+1)],
    ];
    let sx = 0, sy = 0, sz = 0, n = 0;
    for (const [a, b] of edgeList) {
      const va = c[a], vb = c[b];
      if ((va < isovalue) !== (vb < isovalue)) {
        const t = (isovalue - va) / (vb - va);
        const oa = cornerOffsets[a], ob = cornerOffsets[b];
        sx += oa[0] + t * (ob[0] - oa[0]);
        sy += oa[1] + t * (ob[1] - oa[1]);
        sz += oa[2] + t * (ob[2] - oa[2]);
        n++;
      }
    }
    if (n > 0) {
      const cx = (x + sx/n) / (nx - 1);
      const cy = (y + sy/n) / (ny - 1);
      const cz = (z + sz/n) / (nz - 1);
      cellVertex[idx(x, y, z)] = positions.length / 3;
      positions.push(
        bounds.min[0] + cx * (bounds.max[0] - bounds.min[0]),
        bounds.min[1] + cy * (bounds.max[1] - bounds.min[1]),
        bounds.min[2] + cz * (bounds.max[2] - bounds.min[2]),
      );
    }
  }

  // Pass B — quads across grid edges with sign change.
  // Each axis-aligned edge is shared by 4 cells; emit 2 triangles per quad.
  for (let z = 1; z < nz - 1; z++)
  for (let y = 1; y < ny - 1; y++)
  for (let x = 1; x < nx - 1; x++) {
    const v0 = field[idx(x, y, z)];

    const vx = field[idx(x+1, y, z)];
    if ((v0 < isovalue) !== (vx < isovalue)) {
      const a = cellVertex[idx(x, y-1, z-1)], b = cellVertex[idx(x, y,   z-1)];
      const cc = cellVertex[idx(x, y,   z  )], d = cellVertex[idx(x, y-1, z  )];
      if (a >= 0 && b >= 0 && cc >= 0 && d >= 0) {
        if (v0 < isovalue) indices.push(a, b, cc, a, cc, d);
        else               indices.push(a, cc, b, a, d, cc);
      }
    }
    const vy = field[idx(x, y+1, z)];
    if ((v0 < isovalue) !== (vy < isovalue)) {
      const a = cellVertex[idx(x-1, y, z-1)], b = cellVertex[idx(x  , y, z-1)];
      const cc = cellVertex[idx(x  , y, z  )], d = cellVertex[idx(x-1, y, z  )];
      if (a >= 0 && b >= 0 && cc >= 0 && d >= 0) {
        if (v0 < isovalue) indices.push(a, d, cc, a, cc, b);
        else               indices.push(a, b, cc, a, cc, d);
      }
    }
    const vz = field[idx(x, y, z+1)];
    if ((v0 < isovalue) !== (vz < isovalue)) {
      const a = cellVertex[idx(x-1, y-1, z)], b = cellVertex[idx(x  , y-1, z)];
      const cc = cellVertex[idx(x  , y  , z)], d = cellVertex[idx(x-1, y  , z)];
      if (a >= 0 && b >= 0 && cc >= 0 && d >= 0) {
        if (v0 < isovalue) indices.push(a, b, cc, a, cc, d);
        else               indices.push(a, cc, b, a, d, cc);
      }
    }
  }

  return { positions: new Float32Array(positions), indices: new Uint32Array(indices) };
}
```

Three.js wiring:

```javascript
const { positions, indices } = naiveSurfaceNets(field, dims, bounds, 0.0);
const geom = new THREE.BufferGeometry();
geom.setAttribute("position", new THREE.BufferAttribute(positions, 3));
geom.setIndex(new THREE.BufferAttribute(indices, 1));
geom.computeVertexNormals();
```

Surface Nets rounds sharp corners slightly (vertices at edge-crossing
centroids smooth feature edges by ~half a cell). Acceptable for organic
implicit forms; if you need sharp features, use **Dual Contouring** instead.

## When to Use Marching Cubes Instead

Marching Cubes (Lorensen & Cline, 1987) produces triangles that more
faithfully follow the original field but at the cost of large lookup tables
(256-case edge & triangle tables). Pick MC over Surface Nets when:

- Source data has sharp expected features (CT/MRI segmentations, voxel art).
- You want a battle-tested implementation with library support — Three.js
  ships `MarchingCubes` in `examples/jsm/objects/` (designed for metaballs;
  field is mutable, can be repurposed).
- You need exact topology consistency between adjacent cells.

For most algorithmic-art use cases (smooth implicit surfaces from math),
Surface Nets is simpler and arguably prettier.

## 3D Implicit Surface Catalog

### Triply-Periodic Minimal Surfaces (TPMS)

Soap-film geometries that divide space into two interpenetrating channels.
Common in butterfly-wing scales, beetle exoskeletons, photonic crystals,
3D-printed lattices, scaffolds for biological tissue engineering.

```javascript
// Schoen's Gyroid (1970) — chiral, no straight lines or planes of symmetry
gyroid(x, y, z) = sin(k·x)·cos(k·y) + sin(k·y)·cos(k·z) + sin(k·z)·cos(k·x)

// Schwarz P (Primitive) — orthogonal channels
schwarzP(x, y, z) = cos(k·x) + cos(k·y) + cos(k·z)

// Schwarz D (Diamond) — diamond-cubic packing
schwarzD(x, y, z) = sin(k·x)·sin(k·y)·sin(k·z)
                  + sin(k·x)·cos(k·y)·cos(k·z)
                  + cos(k·x)·sin(k·y)·cos(k·z)
                  + cos(k·x)·cos(k·y)·sin(k·z)

// Neovius — narrow tubes meeting at large chambers
neovius(x, y, z) = 3·(cos(k·x) + cos(k·y) + cos(k·z))
                 + 4·cos(k·x)·cos(k·y)·cos(k·z)
```

`k` is the wavenumber (period = 2π/k). Mesh at isovalue 0 for the *minimal*
surface; shift away from 0 to get a chiral solid network with thicker walls.

### Quaternion Fractals

Distance-estimator fractals can be mesh-extracted by sampling the DE on a
grid. The **Mandelbulb** (z → z^p + c in spherical coordinates) produces an
infinitely detailed organic blob; the **Mandelbox** produces crystalline
folded geometry. Resolution-bound — mesh quality follows grid resolution.
For real-time exploration, raymarch the SDF directly via
[shaders-glsl.md](shaders-glsl.md) instead of meshing.

### Voronoi-Based Implicits

```javascript
// Implicit cellular surface from a 3D Voronoi diagram
voronoiF1F2(x, y, z) = F2(p) - F1(p)   // walls between cells
```

Where F1 is the distance to the nearest seed and F2 is the distance to the
second-nearest. Crystalline / cracked-earth aesthetic. See
[voronoi-noise.md](voronoi-noise.md) for distance-metric variants.

### Metaballs / Implicit Blobs

Sum of Gaussian-like falloffs around point seeds:

```javascript
metaballs(p) = Σᵢ rᵢ² / max(|p - cᵢ|², ε) - threshold
```

Mesh at `0`. Smooth blobby surfaces that merge as seeds approach. Animate
seeds along flow fields or attractor trajectories for organic motion.

### Composite Fields

Most interesting forms come from composing the above:

```javascript
field(p) = noise3D(p * 0.4) * 0.5 + gyroid(p, 1.0) * 0.5
// — gyroid topology distorted by Perlin domain warp

field(p) = max(box_sdf(p, [2,2,2]), -gyroid(p, 1.5))
// — gyroid carved out of a cube via CSG (intersection with negated gyroid)
```

CSG operators on SDFs: `min` = union, `max` = intersection, `max(a, -b)` =
subtraction. (Same operators as 2D — see [sdf-2d.md](sdf-2d.md) — they
generalize directly.)

## Compositional Patterns

- **SDF mesh as scaffold + splats as atmosphere**. Mesh provides solid
  scaffold geometry, splats fill the interstitial volume as light/dust.
  See [gaussian-splats.md](gaussian-splats.md). With overlapping
  transparency, see [order-independent-transparency.md](order-independent-transparency.md).
- **Layer multiple isosurfaces of the same field**. Different isovalues
  give nested concentric surfaces. With increasing transparency outward,
  produces glowing volumetric layering.
- **SDF-driven instanced geometry**. Sample the SDF at random points,
  instance a small object (cube, sphere, custom mesh) wherever the field
  is below threshold — produces a "voxelized" rendering of the implicit
  form with arbitrary instance geometry.

## Notable Practitioners & Sources

- **Inigo Quilez** — comprehensive SDF primitives & operators reference at
  [iquilezles.org/articles/distfunctions/](https://iquilezles.org/articles/distfunctions/).
- **Andy Lomas** — *Cellular Forms*, *Mitosis*, *Aggregation* — reaction-diffusion
  and cellular growth solved on grids and isosurface-meshed.
- **Anders Hoff (inconvergent)** — differential growth on meshes; complementary
  to implicit-surface meshing.
- **libfive / Antimony** — F-Rep (functional representation) authoring environments
  that compile expression trees to meshable SDFs.
- **Mikola Lysenko** — original Surface Nets reference implementation and
  extensive blog series on isosurface methods.
