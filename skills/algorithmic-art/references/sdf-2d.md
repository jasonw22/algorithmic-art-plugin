# 2D Signed Distance Fields (SDFs)

## Philosophy

A signed distance field is an elegant unification: every point in space knows its relationship
to every shape — how far away it is, and whether it's inside or outside. This single number
per point enables **smooth boolean operations** (union, subtraction, intersection with rounded
or chamfered edges), **infinite repetition** through domain manipulation, and **organic blending**
of forms — techniques that are awkward or impossible with traditional polygon-based geometry.

2D SDFs bring GPU shader techniques to canvas-based art. Where 3D raymarching is well-known
in shader art (see `shaders-glsl.md`), 2D SDFs are equally powerful for flat compositions:
generative logos, pattern design, morphing shapes, and plotter-ready vector art.

## Core Concept

An SDF function returns:
- **Positive** values outside the shape
- **Negative** values inside the shape
- **Zero** on the boundary (the surface/edge)

```
d = sdf(point) → float
```

To render: color pixels where `d < 0` (inside), or use `smoothstep` for anti-aliased edges.

## 2D Primitives

### JavaScript (p5.js / Canvas)

```javascript
function sdCircle(px, py, r) {
  return Math.sqrt(px*px + py*py) - r;
}

function sdBox(px, py, bx, by) {
  const dx = Math.abs(px) - bx;
  const dy = Math.abs(py) - by;
  return Math.sqrt(Math.max(dx,0)**2 + Math.max(dy,0)**2)
       + Math.min(Math.max(dx, dy), 0);
}

function sdSegment(px, py, ax, ay, bx, by) {
  const pax = px-ax, pay = py-ay, bax = bx-ax, bay = by-ay;
  const h = Math.max(0, Math.min(1, (pax*bax + pay*bay) / (bax*bax + bay*bay)));
  const dx = pax - bax*h, dy = pay - bay*h;
  return Math.sqrt(dx*dx + dy*dy);
}

function sdEquilateralTriangle(px, py, r) {
  const k = Math.sqrt(3);
  let x = Math.abs(px) - r;
  let y = py + r / k;
  if (x + k*y > 0) { const t = (-x + k*y) / 2; x -= t; y -= k*t; } // clamp
  x -= Math.max(-2*r, Math.min(0, x));
  return -Math.sqrt(x*x + y*y) * Math.sign(y);
}

function sdHexagon(px, py, r) {
  const k = Math.sqrt(3) / 2;
  let x = Math.abs(px), y = Math.abs(py);
  const dot = Math.min(0, -2 * k * (k*x + 0.5*y));
  x -= dot * k; y -= dot * 0.5;
  x -= Math.max(0, Math.min(r, x));
  return Math.sqrt(x*x + (y - r)**2) * Math.sign(y - r);
}

function sdStar(px, py, r, n, m) {
  // n-pointed star, m controls inner radius ratio
  const an = Math.PI / n;
  const en = Math.PI / m;
  const acs = [Math.cos(an), Math.sin(an)];
  const ecs = [Math.cos(en), Math.sin(en)];
  let x = Math.abs(px), y = Math.abs(py);
  // rotate into first sector
  const bn = Math.atan2(x, y) % (2 * an) - an;
  const l = Math.sqrt(x*x + y*y);
  x = Math.cos(bn) * l; y = Math.abs(Math.sin(bn) * l);
  x -= r * acs[0]; y -= r * acs[1];
  const dot = Math.max(0, Math.min(r * ecs[1] / ecs[0], x * ecs[0] + y * ecs[1]));
  x -= dot * ecs[0]; y -= dot * ecs[1];
  return Math.sqrt(x*x + y*y) * Math.sign(x);
}

function sdRoundedBox(px, py, bx, by, r) {
  const dx = Math.abs(px) - bx + r;
  const dy = Math.abs(py) - by + r;
  return Math.sqrt(Math.max(dx,0)**2 + Math.max(dy,0)**2)
       + Math.min(Math.max(dx, dy), 0) - r;
}
```

### GLSL (shader mode)

```glsl
float sdCircle(vec2 p, float r) { return length(p) - r; }

float sdBox(vec2 p, vec2 b) {
  vec2 d = abs(p) - b;
  return length(max(d, 0.0)) + min(max(d.x, d.y), 0.0);
}

float sdSegment(vec2 p, vec2 a, vec2 b) {
  vec2 pa = p-a, ba = b-a;
  float h = clamp(dot(pa,ba)/dot(ba,ba), 0.0, 1.0);
  return length(pa - ba*h);
}

float sdHexagon(vec2 p, float r) {
  const vec3 k = vec3(-0.866025404, 0.5, 0.577350269);
  p = abs(p);
  p -= 2.0 * min(dot(k.xy, p), 0.0) * k.xy;
  p -= vec2(clamp(p.x, -k.z*r, k.z*r), r);
  return length(p) * sign(p.y);
}

float sdStar5(vec2 p, float r, float rf) {
  // 5-pointed star, rf = inner radius fraction
  const vec2 k1 = vec2(0.809016994, -0.587785252);
  const vec2 k2 = vec2(-k1.x, k1.y);
  p.x = abs(p.x);
  p -= 2.0 * max(dot(k1, p), 0.0) * k1;
  p -= 2.0 * max(dot(k2, p), 0.0) * k2;
  p.x = abs(p.x);
  p.y -= r;
  vec2 ba = rf * vec2(-k1.y, k1.x) - vec2(0, 1);
  float h = clamp(dot(p, ba) / dot(ba, ba), 0.0, r);
  return length(p - ba * h) * sign(p.y * ba.x - p.x * ba.y);
}

float sdRoundedBox(vec2 p, vec2 b, float r) {
  vec2 q = abs(p) - b + r;
  return min(max(q.x, q.y), 0.0) + length(max(q, 0.0)) - r;
}
```

## Boolean Operations

### Hard Boolean

```javascript
// JavaScript
function opUnion(d1, d2) { return Math.min(d1, d2); }
function opSubtract(d1, d2) { return Math.max(-d1, d2); }
function opIntersect(d1, d2) { return Math.max(d1, d2); }
```

```glsl
// GLSL
float opUnion(float d1, float d2) { return min(d1, d2); }
float opSubtract(float d1, float d2) { return max(-d1, d2); }
float opIntersect(float d1, float d2) { return max(d1, d2); }
```

### Smooth Boolean (Organic Blending)

The smoothness factor `k` controls the blending radius. Larger k = rounder blend.

```javascript
// JavaScript
function opSmoothUnion(d1, d2, k) {
  const h = Math.max(0, Math.min(1, 0.5 + 0.5 * (d2 - d1) / k));
  return d2 * (1-h) + d1 * h - k * h * (1 - h);
}

function opSmoothSubtract(d1, d2, k) {
  const h = Math.max(0, Math.min(1, 0.5 - 0.5 * (d2 + d1) / k));
  return d2 * (1-h) + (-d1) * h + k * h * (1 - h);
}

function opSmoothIntersect(d1, d2, k) {
  const h = Math.max(0, Math.min(1, 0.5 - 0.5 * (d2 - d1) / k));
  return d2 * (1-h) + d1 * h + k * h * (1 - h);
}
```

### Chamfer & Round Boolean

```javascript
// Chamfer union: flat 45° bevel at junction
function opChamferUnion(d1, d2, r) {
  return Math.min(Math.min(d1, d2), (d1 - r + d2) * Math.SQRT1_2);
}

// Round union: circular profile at junction
function opRoundUnion(d1, d2, r) {
  const u = Math.max(r - d1, r - d2, 0);
  return Math.max(r, Math.min(d1, d2)) - Math.sqrt(u * u + u * u) * 0.5;
}
```

## Domain Operations

### Repetition (Infinite Tiling)

```javascript
// Infinite 2D repetition
function opRepeat2D(px, py, spacingX, spacingY) {
  return [
    ((px % spacingX) + spacingX) % spacingX - spacingX * 0.5,
    ((py % spacingY) + spacingY) % spacingY - spacingY * 0.5,
  ];
}
```

```glsl
vec2 opRepeat(vec2 p, vec2 spacing) {
  return mod(p + 0.5 * spacing, spacing) - 0.5 * spacing;
}
```

### Finite Repetition (N copies)

```glsl
vec2 opRepeatLim(vec2 p, float spacing, vec2 limit) {
  return p - spacing * clamp(round(p / spacing), -limit, limit);
}
```

### Polar Repetition (Radial Symmetry)

```javascript
// Repeat around origin with n-fold symmetry
function opPolar(px, py, n) {
  const angle = Math.atan2(py, px);
  const sector = Math.PI * 2 / n;
  const a = ((angle % sector) + sector) % sector - sector * 0.5;
  const r = Math.sqrt(px*px + py*py);
  return [r * Math.cos(a), r * Math.sin(a)];
}
```

```glsl
vec2 opPolar(vec2 p, float n) {
  float angle = atan(p.y, p.x);
  float sector = 6.28318 / n;
  angle = mod(angle + sector * 0.5, sector) - sector * 0.5;
  return length(p) * vec2(cos(angle), sin(angle));
}
```

### Mirror

```javascript
function opMirrorX(px) { return Math.abs(px); }
function opMirrorXY(px, py) { return [Math.abs(px), Math.abs(py)]; }
```

### Rotation

```javascript
function opRotate(px, py, angle) {
  const c = Math.cos(angle), s = Math.sin(angle);
  return [px * c - py * s, px * s + py * c];
}
```

## Rendering 2D SDFs

### Anti-Aliased Edge Rendering (p5.js)

```javascript
function drawSDF(p, sdfFunc, w, h) {
  p.loadPixels();
  const pixelScale = 2 / Math.min(w, h); // normalize to [-1,1] range
  for (let y = 0; y < h; y++) {
    for (let x = 0; x < w; x++) {
      // Center and normalize coordinates
      const px = (x - w/2) * pixelScale;
      const py = (h/2 - y) * pixelScale;
      const d = sdfFunc(px, py);

      // Anti-aliased edge: smooth transition over ~1 pixel
      const edge = 1 - smoothstep(-pixelScale, pixelScale, d);

      const idx = (y * w + x) * 4;
      p.pixels[idx]   = edge * 255; // R
      p.pixels[idx+1] = edge * 255; // G
      p.pixels[idx+2] = edge * 255; // B
      p.pixels[idx+3] = 255;        // A
    }
  }
  p.updatePixels();
}

function smoothstep(a, b, x) {
  const t = Math.max(0, Math.min(1, (x - a) / (b - a)));
  return t * t * (3 - 2 * t);
}
```

### Contour Lines / Rings

```javascript
// Draw concentric contour rings from an SDF
function contourColor(d, spacing, thickness) {
  const rings = Math.abs(((d / spacing) % 1) - 0.5) * 2;
  return smoothstep(1 - thickness, 1, rings);
}
```

```glsl
// GLSL contour rings
float contour(float d, float spacing, float thickness) {
  return smoothstep(1.0 - thickness, 1.0, abs(fract(d / spacing) - 0.5) * 2.0);
}
```

### Glow / Distance-Based Color

```glsl
// Soft glow around SDF boundary
vec3 glow(float d, vec3 color, float intensity) {
  return color * intensity / (abs(d) + 0.01);
}

// Map distance to palette
vec3 distanceColor(float d, vec3 inside, vec3 outside, float edgeWidth) {
  float t = smoothstep(-edgeWidth, edgeWidth, d);
  return mix(inside, outside, t);
}
```

## Composing Complex Scenes

Build complex 2D compositions by combining primitives, booleans, and domain operations:

```javascript
function scene(px, py, time) {
  // Polar repetition: 6-fold symmetry
  const [rx, ry] = opPolar(px, py, 6);

  // Repeated circles along each arm
  const circle = sdCircle(rx - 0.5, ry, 0.15);

  // Central hexagon
  const hex = sdHexagon(px, py, 0.3);

  // Smooth union of all elements
  let d = opSmoothUnion(circle, hex, 0.1);

  // Subtract a pulsing hole
  const hole = sdCircle(px, py, 0.1 + Math.sin(time) * 0.05);
  d = opSmoothSubtract(hole, d, 0.05);

  return d;
}
```

## Performance Notes

- **CPU (p5.js)**: Evaluating an SDF per pixel is O(width × height). For real-time animation,
  keep canvas size moderate (400–800px) or render to a smaller buffer and scale up.
  Use `pixelDensity(1)` to avoid 4× cost on retina displays.
- **GPU (GLSL)**: 2D SDFs are trivially fast on GPU. Complex scenes with dozens of primitives
  and operations still run at 60fps. Prefer shader mode for real-time interactive SDF art.
- **Vectorization**: SDFs can be converted back to polygons via marching squares (contour
  extraction at d=0). This enables plotter-ready SVG output from SDF compositions.

## Key References

- **Inigo Quilez** — iquilezles.org/articles/distfunctions2d/ — definitive 2D SDF catalog
- **thi.ng/geom-sdf** — 2D SDF from geometry with smooth boolean operators, domain modifiers,
  and contour extraction back to polygons
- **Mercury SDF library** (mercury.sexy/hg_sdf/) — GLSL SDF functions and domain operations
