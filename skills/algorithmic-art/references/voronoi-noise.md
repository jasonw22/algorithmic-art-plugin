# Voronoi / Cellular Noise Reference

## Overview

Voronoi noise (also called cellular noise or Worley noise) partitions space by proximity
to a set of random seed points. Unlike Perlin/simplex noise which produces smooth gradients,
Voronoi creates cell-like patterns — bubbles, cracked earth, scales, crystal formations,
soap films, biological cells, and stone walls.

## Core Algorithm

1. Divide space into a grid
2. Place one random point (feature point) per grid cell
3. For each pixel, find distances to nearby feature points
4. Use those distances to generate patterns

## GLSL Implementation

### Basic Voronoi (F1 — Distance to Nearest)

```glsl
// Hash function for cell point positions
vec2 voronoiHash(vec2 cell) {
  cell = vec2(dot(cell, vec2(127.1, 311.7)), dot(cell, vec2(269.5, 183.3)));
  return fract(sin(cell) * 43758.5453);
}

// Animated hash (points move over time)
vec2 voronoiHashAnimated(vec2 cell, float time) {
  vec2 h = voronoiHash(cell);
  return 0.5 + 0.5 * sin(time + 6.28318 * h);
}

// F1 Voronoi: distance to nearest point
float voronoiF1(vec2 p) {
  vec2 cell = floor(p);
  vec2 frac = fract(p);
  float minDist = 1.0;

  for (int y = -1; y <= 1; y++) {
    for (int x = -1; x <= 1; x++) {
      vec2 neighbor = vec2(x, y);
      vec2 point = voronoiHash(cell + neighbor);
      float d = length(neighbor + point - frac);
      minDist = min(minDist, d);
    }
  }
  return minDist;
}
```

### F1 and F2 (Two Nearest Distances)

```glsl
// Returns vec2(F1, F2) — distances to nearest and second-nearest points
vec2 voronoiF1F2(vec2 p) {
  vec2 cell = floor(p);
  vec2 frac = fract(p);
  float f1 = 1.0;
  float f2 = 1.0;

  for (int y = -1; y <= 1; y++) {
    for (int x = -1; x <= 1; x++) {
      vec2 neighbor = vec2(x, y);
      vec2 point = voronoiHash(cell + neighbor);
      float d = length(neighbor + point - frac);

      if (d < f1) {
        f2 = f1;
        f1 = d;
      } else if (d < f2) {
        f2 = d;
      }
    }
  }
  return vec2(f1, f2);
}
```

### Cell ID (Which Cell Am I In?)

```glsl
// Returns distance, cell ID, and cell center
struct VoronoiResult {
  float dist;
  vec2 cellId;
  vec2 cellCenter;
};

VoronoiResult voronoiFull(vec2 p) {
  vec2 cell = floor(p);
  vec2 frac = fract(p);
  float minDist = 1.0;
  vec2 closestId = vec2(0.0);
  vec2 closestCenter = vec2(0.0);

  for (int y = -1; y <= 1; y++) {
    for (int x = -1; x <= 1; x++) {
      vec2 neighbor = vec2(x, y);
      vec2 id = cell + neighbor;
      vec2 point = voronoiHash(id);
      vec2 center = neighbor + point;
      float d = length(center - frac);

      if (d < minDist) {
        minDist = d;
        closestId = id;
        closestCenter = center;
      }
    }
  }

  VoronoiResult r;
  r.dist = minDist;
  r.cellId = closestId;
  r.cellCenter = closestCenter + cell;
  return r;
}
```

## Distance Metrics

Different distance functions create different cell shapes:

```glsl
// Euclidean (round cells — default)
float distEuclidean(vec2 a, vec2 b) {
  return length(a - b);
}

// Manhattan (diamond-shaped cells)
float distManhattan(vec2 a, vec2 b) {
  vec2 d = abs(a - b);
  return d.x + d.y;
}

// Chebyshev (square cells)
float distChebyshev(vec2 a, vec2 b) {
  vec2 d = abs(a - b);
  return max(d.x, d.y);
}

// Minkowski (generalized — p=1 Manhattan, p=2 Euclidean, p=∞ Chebyshev)
float distMinkowski(vec2 a, vec2 b, float p) {
  vec2 d = abs(a - b);
  return pow(pow(d.x, p) + pow(d.y, p), 1.0 / p);
}
```

To use a different metric, replace `length(neighbor + point - frac)` in the Voronoi
functions with your chosen distance function.

## Pattern Variations

### Edge Detection (F2 - F1)

The difference between first and second nearest distances highlights cell boundaries:

```glsl
vec2 f = voronoiF1F2(p * scale);
float edges = f.y - f.x;
// edges ≈ 0 at boundaries, > 0 inside cells
float edgeLine = 1.0 - smoothstep(0.0, 0.05, edges);
```

### Cracked Earth / Stone

```glsl
float crackedEarth(vec2 p) {
  vec2 f = voronoiF1F2(p * 4.0);
  float cracks = smoothstep(0.0, 0.04, f.y - f.x);
  // Add noise variation inside cells
  float cellNoise = snoise(p * 10.0) * 0.1;
  return cracks + cellNoise * cracks;
}
```

### Bubble / Soap Film

```glsl
vec3 bubbles(vec2 p, float time) {
  vec2 f = voronoiF1F2(p * 3.0 + time * 0.1);
  // Iridescent color from distance
  vec3 color = palette(f.x * 3.0, vec3(0.5), vec3(0.5), vec3(1.0), vec3(0.0, 0.33, 0.67));
  // Thin edge highlight
  float edge = smoothstep(0.02, 0.0, f.y - f.x);
  color += edge * 0.5;
  return color;
}
```

### Crystal / Gem Facets

```glsl
float crystal(vec2 p) {
  float v = voronoiF1(p * 5.0);
  // Flat facets with sharp edges
  return floor(v * 8.0) / 8.0;
}
```

### Organic Cells (Biology)

```glsl
vec3 biologicalCells(vec2 p, float time) {
  // Animated cell points
  vec2 cell = floor(p * 4.0);
  vec2 frac = fract(p * 4.0);
  float f1 = 1.0;
  vec2 closestId = vec2(0.0);

  for (int y = -1; y <= 1; y++) {
    for (int x = -1; x <= 1; x++) {
      vec2 neighbor = vec2(x, y);
      vec2 point = voronoiHashAnimated(cell + neighbor, time * 0.5);
      float d = length(neighbor + point - frac);
      if (d < f1) { f1 = d; closestId = cell + neighbor; }
    }
  }

  // Color per cell from hash
  float cellHue = fract(dot(closestId, vec2(0.13, 0.27)));
  vec3 color = palette(cellHue, vec3(0.5), vec3(0.5), vec3(1.0), vec3(0.0, 0.1, 0.2));

  // Membrane (edge darkening)
  color *= smoothstep(0.0, 0.15, f1);

  // Nucleus
  float nucleus = smoothstep(0.12, 0.08, f1);
  color = mix(color, color * 0.3, nucleus);

  return color;
}
```

## 3D Voronoi

For volumetric or 3D surface texturing:

```glsl
vec3 voronoiHash3(vec3 cell) {
  cell = vec3(
    dot(cell, vec3(127.1, 311.7, 74.7)),
    dot(cell, vec3(269.5, 183.3, 246.1)),
    dot(cell, vec3(113.5, 271.9, 124.6))
  );
  return fract(sin(cell) * 43758.5453);
}

float voronoi3D(vec3 p) {
  vec3 cell = floor(p);
  vec3 frac = fract(p);
  float minDist = 1.0;

  for (int z = -1; z <= 1; z++)
  for (int y = -1; y <= 1; y++)
  for (int x = -1; x <= 1; x++) {
    vec3 neighbor = vec3(x, y, z);
    vec3 point = voronoiHash3(cell + neighbor);
    minDist = min(minDist, length(neighbor + point - frac));
  }
  return minDist;
}
```

## FBM Voronoi (Multi-Octave)

Layer Voronoi at different scales like FBM:

```glsl
float fbmVoronoi(vec2 p, int octaves) {
  float value = 0.0;
  float amp = 0.5;
  float freq = 1.0;

  for (int i = 0; i < octaves; i++) {
    value += amp * voronoiF1(p * freq);
    freq *= 2.0;
    amp *= 0.5;
  }
  return value;
}
```

## JavaScript Implementation (p5.js / thi.ng)

```javascript
function voronoiF1(px, py, scale, points) {
  // points: array of {x, y} in [0, 1] range
  let minDist = Infinity;
  const sx = px * scale, sy = py * scale;

  for (const pt of points) {
    const dx = sx - pt.x * scale;
    const dy = sy - pt.y * scale;
    minDist = Math.min(minDist, Math.sqrt(dx * dx + dy * dy));
  }
  return minDist;
}

// Grid-accelerated version (for real-time use)
function voronoiGrid(px, py, scale) {
  const cx = Math.floor(px * scale);
  const cy = Math.floor(py * scale);
  const fx = (px * scale) - cx;
  const fy = (py * scale) - cy;
  let f1 = 1.0, f2 = 1.0;

  for (let dy = -1; dy <= 1; dy++) {
    for (let dx = -1; dx <= 1; dx++) {
      const hash = pseudoHash2D(cx + dx, cy + dy);
      const d = Math.hypot(dx + hash.x - fx, dy + hash.y - fy);
      if (d < f1) { f2 = f1; f1 = d; }
      else if (d < f2) { f2 = d; }
    }
  }
  return { f1, f2 };
}
```

## Key References

- **Steven Worley** — "A Cellular Texture Basis Function" (1996, original paper)
- **Inigo Quilez** — iquilezles.org/articles/voronoise — smooth Voronoi variations
- **Stefan Gustavson** — "Simplex noise demystified" (includes Voronoi discussion)
- **Book of Shaders, Ch. 12** — Cellular noise tutorial
