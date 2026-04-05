# Procedural 2D Patterns (GLSL) Reference

## Overview

Procedural 2D patterns are computed mathematically from UV coordinates — no textures needed.
They're infinitely scalable, tileable, and parameterizable. These patterns serve as building
blocks for floors, walls, fabrics, backgrounds, and abstract compositions.

## Grid Patterns

### Checkerboard

```glsl
float checkerboard(vec2 uv, float scale) {
  vec2 grid = floor(uv * scale);
  return mod(grid.x + grid.y, 2.0);
}

// Anti-aliased version
float checkerboardAA(vec2 uv, float scale) {
  vec2 p = uv * scale;
  vec2 fw = fwidth(p);
  // Analytical filtering
  vec2 i = 2.0 * (abs(fract(p - 0.25) - 0.5) - abs(fract(p - 0.75) - 0.5));
  return 0.5 - 0.5 * i.x * i.y;
}
```

### Grid Lines

```glsl
float gridLines(vec2 uv, float scale, float thickness) {
  vec2 grid = abs(fract(uv * scale - 0.5) - 0.5);
  vec2 fw = fwidth(uv * scale);
  vec2 lines = smoothstep(thickness + fw, thickness - fw, grid);
  return max(lines.x, lines.y);
}
```

### Dot Grid

```glsl
float dotGrid(vec2 uv, float scale, float radius) {
  vec2 cell = fract(uv * scale) - 0.5;
  float d = length(cell);
  return 1.0 - smoothstep(radius - fwidth(d), radius, d);
}
```

## Brick / Masonry

```glsl
float brickPattern(vec2 uv, vec2 brickSize, float mortarWidth) {
  // Offset every other row
  vec2 p = uv / brickSize;
  float row = floor(p.y);
  p.x += mod(row, 2.0) * 0.5;  // half-brick offset

  vec2 cell = fract(p) - 0.5;
  vec2 fw = fwidth(p);

  // Mortar lines
  float mortarX = smoothstep(0.5 - mortarWidth - fw.x, 0.5 - mortarWidth, abs(cell.x));
  float mortarY = smoothstep(0.5 - mortarWidth - fw.y, 0.5 - mortarWidth, abs(cell.y));
  float mortar = max(mortarX, mortarY);

  return 1.0 - mortar;  // 1 = brick, 0 = mortar
}

// With per-brick color variation
vec3 coloredBricks(vec2 uv, vec2 brickSize) {
  vec2 p = uv / brickSize;
  float row = floor(p.y);
  p.x += mod(row, 2.0) * 0.5;

  vec2 cellId = floor(p);
  float brick = brickPattern(uv, brickSize, 0.05);

  // Per-brick color from hash
  float h = fract(sin(dot(cellId, vec2(127.1, 311.7))) * 43758.5453);
  vec3 color = mix(vec3(0.6, 0.25, 0.15), vec3(0.75, 0.35, 0.2), h);

  return mix(vec3(0.4), color, brick);  // mortar color vs brick color
}
```

## Hexagonal Grid

```glsl
// Returns: xy = cell-local coords, zw = cell ID
vec4 hexGrid(vec2 uv, float scale) {
  vec2 p = uv * scale;

  // Two candidate hex centers
  vec2 a = mod(p, vec2(1.0, sqrt(3.0))) - vec2(0.5, sqrt(3.0) * 0.5);
  vec2 b = mod(p - vec2(0.5, sqrt(3.0) * 0.5), vec2(1.0, sqrt(3.0))) - vec2(0.5, sqrt(3.0) * 0.5);

  // Choose the closer center
  vec2 g = (length(a) < length(b)) ? a : b;
  vec2 id = p - g;

  return vec4(g, floor(id));
}

// Hex distance (for drawing hex shapes)
float hexDist(vec2 p) {
  p = abs(p);
  return max(dot(p, normalize(vec2(1.0, sqrt(3.0)))), p.x);
}

// Complete hex tiling with edges
float hexTiling(vec2 uv, float scale, float edgeWidth) {
  vec4 h = hexGrid(uv, scale);
  float d = hexDist(h.xy);
  float fw = fwidth(d);
  return smoothstep(0.5 - edgeWidth - fw, 0.5 - edgeWidth, d);
}
```

## Truchet Tiles (GLSL)

```glsl
// Quarter-circle Truchet
float truchetPattern(vec2 uv, float scale) {
  vec2 p = uv * scale;
  vec2 cell = floor(p);
  vec2 f = fract(p) - 0.5;

  // Random rotation per cell (0 or 1)
  float flip = step(0.5, fract(sin(dot(cell, vec2(127.1, 311.7))) * 43758.5453));

  // Flip coordinates
  if (flip > 0.5) f = vec2(-f.y, f.x);

  // Quarter circles from two corners
  float d1 = abs(length(f - vec2(-0.5, -0.5)) - 0.5);
  float d2 = abs(length(f - vec2(0.5, 0.5)) - 0.5);
  float d = min(d1, d2);

  float fw = fwidth(d);
  return smoothstep(0.05 + fw, 0.05 - fw, d);
}

// Multi-scale Truchet with line thickness
float truchetLines(vec2 uv, float scale, float lineWidth) {
  vec2 p = uv * scale;
  vec2 cell = floor(p);
  vec2 f = fract(p) - 0.5;

  float h = fract(sin(dot(cell, vec2(127.1, 311.7))) * 43758.5453);
  if (h > 0.5) f.x = -f.x;

  float d1 = length(f - vec2(-0.5)) - 0.5;
  float d2 = length(f - vec2(0.5)) - 0.5;

  float fw = fwidth(uv.x * scale);
  float line1 = smoothstep(lineWidth + fw, lineWidth - fw, abs(d1));
  float line2 = smoothstep(lineWidth + fw, lineWidth - fw, abs(d2));

  return max(line1, line2);
}
```

## Stripes & Waves

```glsl
// Diagonal stripes
float diagonalStripes(vec2 uv, float scale, float duty) {
  float d = fract((uv.x + uv.y) * scale);
  float fw = fwidth((uv.x + uv.y) * scale);
  return smoothstep(duty - fw, duty + fw, d);
}

// Wavy stripes
float wavyStripes(vec2 uv, float scale, float waveFreq, float waveAmp) {
  float wave = sin(uv.x * waveFreq) * waveAmp;
  float d = fract((uv.y + wave) * scale);
  return smoothstep(0.45, 0.55, d);
}

// Concentric rings
float concentricRings(vec2 uv, vec2 center, float scale) {
  float d = length(uv - center) * scale;
  return smoothstep(0.45, 0.55, fract(d));
}

// Spiral
float spiral(vec2 uv, float arms, float twist) {
  float r = length(uv);
  float theta = atan(uv.y, uv.x);
  float d = fract((theta * arms / 6.28318 + r * twist));
  return smoothstep(0.45, 0.55, d);
}
```

## Polka Dots (Hexagonal Packing)

```glsl
float polkaDots(vec2 uv, float scale, float radius) {
  // Hexagonal packing (denser than square grid)
  vec2 p = uv * scale;
  vec2 a = mod(p, 2.0) - 1.0;
  vec2 b = mod(p + 1.0, 2.0) - 1.0;
  float da = length(a);
  float db = length(b);
  float d = min(da, db);

  return 1.0 - smoothstep(radius - fwidth(d), radius, d);
}
```

## Noise-Warped Patterns

Combine procedural patterns with noise for organic variations:

```glsl
vec3 warpedCheckerboard(vec2 uv, float time) {
  // Domain warp with fbm
  vec2 warp = vec2(
    fbm(uv * 3.0 + time * 0.1, 4),
    fbm(uv * 3.0 + vec2(5.0) + time * 0.1, 4)
  );
  float check = checkerboard(uv + warp * 0.1, 8.0);
  return mix(vec3(0.1), vec3(0.9), check);
}
```

## Key References

- **Inigo Quilez** — procedural pattern filtering and anti-aliasing
- **The Book of Shaders, Ch. 9-10** — patterns and tiling
- **Stefan Gustavson** — procedural textures without stored data
- **Shadertoy** — search "pattern", "tiling", "truchet" for community examples
