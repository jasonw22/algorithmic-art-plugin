# thi.ng Mode — Functional Canvas 2D Reference

## Overview

The thi.ng mode uses the `assets/template-thing.html` template. It renders to a full-viewport
Canvas 2D context with a built-in utility library (`tng`) providing perceptual color (Oklab),
2D signed distance fields, seeded PRNG (SFC32), vector math, and Perlin noise.

This mode is best for:
- **Perceptual color art** — pieces where palette quality is paramount (cosine gradients,
  Oklab interpolation, LCH-based themes)
- **2D SDF compositions** — smooth boolean shapes, domain repetition, polar symmetry
- **Functional/data-driven workflows** — geometry as data, enabling clean SVG export
- **Plotter/print-ready output** — via the `sketchSVG()` vector export function
- **Noise-based generative art** — seeded Perlin noise with fBm, deterministic from seed

## Template Contract

Set `renderMode: "thing"` in SKETCH_META. Implement these functions:

```javascript
const SKETCH_META = {
  title: "Piece Title",
  concept: "Brief philosophical description",
  technique: "Algorithm family name",
  renderMode: "thing"
};

const PARAMS = {
  // Same format as all other templates
};

// Called once per seed. Initialize data structures.
// tng.random is already seeded from the seed control.
// Return a state object (or undefined).
function sketchSetup(ctx, w, h, tng) {
  // ctx = Canvas 2D rendering context
  // w, h = canvas dimensions
  // tng = utility library (color, sdf, random, vec, math, noise)
  return { /* your state */ };
}

// Called every frame.
function sketchDraw(ctx, w, h, params, tng, state, time, delta) {
  // time = seconds since start
  // delta = seconds since last frame
  // Draw to ctx using Canvas 2D API + tng utilities
}

// OPTIONAL: Return SVG string for clean vector export.
// If not implemented, SVG export falls back to edge detection.
function sketchSVG(w, h, params, tng, state) {
  return '<svg xmlns="http://www.w3.org/2000/svg" ...>...</svg>';
}
```

## The `tng` Utility Object

### tng.random — Seeded PRNG (SFC32)

Deterministic randomness seeded from the seed control. Same seed = same output.

```javascript
tng.random.float()            // [0, 1)
tng.random.range(5, 10)       // float in [5, 10)
tng.random.int(0, 100)        // integer in [0, 100)
tng.random.bool()             // true/false (50/50)
tng.random.bool(0.3)          // true with 30% probability
tng.random.pick(array)        // random element from array
tng.random.shuffle(array)     // return shuffled copy
tng.random.gaussian(0, 1)     // Gaussian distribution (mean, stddev)
tng.random.weightedPick(items, weights)  // weighted random selection
```

### tng.color — Perceptual Color (Oklab / Oklch)

```javascript
// Oklch: perceptual color from Lightness (0-1), Chroma (0-0.4), Hue (0-360)
tng.color.oklch(0.7, 0.15, 30)           // → [r, g, b] (0-1)

// Oklab interpolation (no muddy midpoints)
tng.color.lerpOklab([1,0,0], [0,0,1], 0.5)  // → vivid purple, not gray

// Cosine gradient palettes
tng.color.palette(t, "sunset")            // named preset: rainbow, sunset, ocean, fire, electric, pastel, neon
tng.color.cosine(t, a, b, c, d)           // custom coefficients (each a [r,g,b] array)

// Multi-stop gradient in Oklab
tng.color.gradient([[1,0,0], [0,1,0], [0,0,1]], t)  // 3-stop gradient, t in [0,1]

// Convert to CSS for ctx.fillStyle / ctx.strokeStyle
tng.color.toCSS([0.8, 0.2, 0.1])         // → "rgb(204,51,26)"
tng.color.toCSSA([0.8, 0.2, 0.1], 0.5)   // → "rgba(204,51,26,0.5)"

// Parse hex
tng.color.fromHex("#ff4400")              // → [1, 0.267, 0]

// Raw Oklab conversion
tng.color.rgbToOklab([r, g, b])           // sRGB → Oklab
tng.color.oklabToRgb([L, a, b])           // Oklab → sRGB
```

**Cosine palette design tips:**
- Vary the **d** (phase) vector to explore different palettes — phase offsets control hue
- Set **c** (frequency) to `[1,1,1]` for single-cycle, `[2,1,0]` for multi-cycle
- Set **b** (amplitude) smaller for softer palettes, larger for vivid
- Access presets directly: `tng.color.palettes.sunset.a` etc.

### tng.sdf — 2D Signed Distance Fields

#### Primitives
All primitives are centered at origin. Translate point before calling.

```javascript
// Point relative to shape center:
const px = x - centerX, py = y - centerY;

tng.sdf.circle(px, py, radius)
tng.sdf.box(px, py, halfW, halfH)
tng.sdf.roundedBox(px, py, halfW, halfH, cornerRadius)
tng.sdf.segment(px, py, ax, ay, bx, by)    // distance to line segment
tng.sdf.hexagon(px, py, radius)
tng.sdf.ring(px, py, radius, thickness)
```

#### Boolean Operations
```javascript
tng.sdf.union(d1, d2)                       // hard min
tng.sdf.subtract(d1, d2)                    // carve d1 from d2
tng.sdf.intersect(d1, d2)                   // hard max

tng.sdf.smoothUnion(d1, d2, k)              // organic blend (k = blend radius)
tng.sdf.smoothSubtract(d1, d2, k)           // soft carving
tng.sdf.smoothIntersect(d1, d2, k)          // soft intersection
```

#### Shape Modifiers
```javascript
tng.sdf.round(d, r)      // add rounding to any SDF
tng.sdf.onion(d, t)      // hollow out (shell of thickness t)
```

#### Domain Operations
Transform the point *before* evaluating the SDF to create repetition, symmetry, etc.

```javascript
// Infinite grid repetition
const [rx, ry] = tng.sdf.opRepeat(px, py, spacingX, spacingY);
const d = tng.sdf.circle(rx, ry, 0.1);

// N-fold radial symmetry (mandala, snowflake)
const [sx, sy] = tng.sdf.opPolar(px, py, 6);  // 6-fold
const d = tng.sdf.box(sx - 0.5, sy, 0.1, 0.02);

// Mirror across axes
const [mx, my] = tng.sdf.opMirror(px, py);

// Rotation
const [rx, ry] = tng.sdf.opRotate(px, py, angle);
```

#### Rendering SDF to Canvas
```javascript
// Fill entire canvas with SDF-based coloring:
tng.sdf.render(ctx, w, h, sceneSDF, colorFunc);

// sceneSDF(px, py) → distance (in normalized coords, ≈ -1 to 1)
// colorFunc(distance, px, py) → [r, g, b, a] (0-1 each)

// Example:
function sceneSDF(px, py) {
  const circle = tng.sdf.circle(px, py, 0.5);
  const box = tng.sdf.box(px - 0.3, py, 0.2, 0.2);
  return tng.sdf.smoothUnion(circle, box, 0.1);
}

function colorFunc(d, px, py) {
  if (d < 0) {
    const c = tng.color.palette(Math.abs(d) * 3, "ocean");
    return [c[0], c[1], c[2], 1];
  }
  // Glow effect outside
  const glow = 0.02 / (Math.abs(d) + 0.02);
  return [glow * 0.5, glow * 0.8, glow, 1];
}

tng.sdf.render(ctx, w, h, sceneSDF, colorFunc);
```

### tng.noise — Seeded Perlin Noise

```javascript
// 2D Perlin noise (approximately -1 to 1)
tng.noise.noise2d(x, y)

// Fractal Brownian Motion
tng.noise.fbm(x, y, octaves, lacunarity, persistence)
// Defaults: octaves=4, lacunarity=2.0, persistence=0.5
```

Noise is seeded automatically when the seed changes. Same seed = same noise field.

### tng.vec — 2D Vector Math

Vectors are `[x, y]` arrays. All functions return new arrays (no mutation).

```javascript
tng.vec.add(a, b)         tng.vec.sub(a, b)
tng.vec.scale(v, s)       tng.vec.len(v)
tng.vec.lenSq(v)          tng.vec.normalize(v)
tng.vec.dot(a, b)         tng.vec.cross(a, b)    // scalar 2D cross product
tng.vec.rotate(v, angle)  tng.vec.lerp(a, b, t)
tng.vec.dist(a, b)        tng.vec.angle(v)       // atan2
tng.vec.fromAngle(a, r)   // angle + optional radius → [x, y]
```

### tng.math — Scalar Utilities

```javascript
tng.math.clamp(x, lo, hi)
tng.math.lerp(a, b, t)
tng.math.map(x, inLo, inHi, outLo, outHi)
tng.math.smoothstep(edge0, edge1, x)
tng.math.fract(x)         // fractional part
tng.math.mod(x, y)        // always-positive modulo
tng.math.TAU              // 2π
tng.math.PI               // π
tng.math.HALF_PI          // π/2
```

## Common Patterns

### Pattern 1: SDF Composition with Perceptual Color

```javascript
function sketchDraw(ctx, w, h, params, tng, state, time) {
  function scene(px, py) {
    // Polar repetition: 8-fold symmetry
    const [sx, sy] = tng.sdf.opPolar(px, py, 8);
    // Repeated circles along each arm
    const arm = tng.sdf.circle(sx - 0.4, sy, 0.08);
    // Central ring
    const ring = tng.sdf.ring(px, py, 0.2, 0.03);
    // Smooth union
    return tng.sdf.smoothUnion(arm, ring, 0.05);
  }

  function colorize(d, px, py) {
    const t = tng.math.smoothstep(0.01, -0.3, d);
    const c = tng.color.palette(t + time * 0.1, "ocean");
    const bg = tng.color.fromHex(params.bgColor);
    const mixed = tng.color.lerpOklab(bg, c, t);
    return [mixed[0], mixed[1], mixed[2], 1];
  }

  tng.sdf.render(ctx, w, h, scene, colorize);
}
```

### Pattern 2: Particle Flow Field with Data-Driven SVG Export

```javascript
function sketchSetup(ctx, w, h, tng) {
  const particles = [];
  for (let i = 0; i < 500; i++) {
    particles.push({
      pos: [tng.random.range(0, w), tng.random.range(0, h)],
      trail: [],
      hue: tng.random.range(0, 360),
    });
  }
  return { particles };
}

function sketchDraw(ctx, w, h, params, tng, state, time) {
  // Semi-transparent overlay for trail effect
  ctx.fillStyle = tng.color.toCSSA(tng.color.fromHex(params.bgColor), 0.02);
  ctx.fillRect(0, 0, w, h);

  const scale = params.noiseScale;
  for (const p of state.particles) {
    const nx = p.pos[0] / w * scale;
    const ny = p.pos[1] / h * scale;
    const angle = tng.noise.fbm(nx, ny, 4) * tng.math.TAU;
    const vel = tng.vec.fromAngle(angle, params.speed);
    const prev = [...p.pos];
    p.pos = tng.vec.add(p.pos, vel);
    p.trail.push([...p.pos]);

    // Wrap
    if (p.pos[0] < 0) p.pos[0] += w;
    if (p.pos[0] > w) p.pos[0] -= w;
    if (p.pos[1] < 0) p.pos[1] += h;
    if (p.pos[1] > h) p.pos[1] -= h;

    // Draw
    const c = tng.color.oklch(0.7, 0.12, p.hue);
    ctx.strokeStyle = tng.color.toCSSA(c, 0.4);
    ctx.beginPath();
    ctx.moveTo(prev[0], prev[1]);
    ctx.lineTo(p.pos[0], p.pos[1]);
    ctx.stroke();
  }
}

// Clean SVG export from trail data
function sketchSVG(w, h, params, tng, state) {
  let svg = `<svg xmlns="http://www.w3.org/2000/svg" width="${w}" height="${h}" viewBox="0 0 ${w} ${h}">`;
  svg += `<rect width="${w}" height="${h}" fill="${params.bgColor}"/>`;
  for (const p of state.particles) {
    if (p.trail.length < 2) continue;
    const c = tng.color.oklch(0.7, 0.12, p.hue);
    const css = tng.color.toCSS(c);
    let d = `M${p.trail[0][0].toFixed(1)},${p.trail[0][1].toFixed(1)}`;
    for (let i = 1; i < p.trail.length; i++) {
      d += `L${p.trail[i][0].toFixed(1)},${p.trail[i][1].toFixed(1)}`;
    }
    svg += `<path d="${d}" fill="none" stroke="${css}" stroke-width="0.5" opacity="0.6"/>`;
  }
  svg += '</svg>';
  return svg;
}
```

### Pattern 3: Tessellation with Data Geometry

```javascript
function sketchSetup(ctx, w, h, tng) {
  // Generate tile data
  const tiles = [];
  const size = 60;
  const cols = Math.ceil(w / size), rows = Math.ceil(h / size);
  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      tiles.push({
        x: c * size, y: r * size, size,
        rotation: tng.random.pick([0, 1, 2, 3]),
        hue: tng.random.range(0, 360),
      });
    }
  }
  return { tiles };
}

function sketchDraw(ctx, w, h, params, tng, state) {
  ctx.clearRect(0, 0, w, h);
  ctx.fillStyle = params.bgColor;
  ctx.fillRect(0, 0, w, h);

  for (const tile of state.tiles) {
    const c = tng.color.oklch(0.7, 0.1, tile.hue);
    ctx.save();
    ctx.translate(tile.x + tile.size/2, tile.y + tile.size/2);
    ctx.rotate(tile.rotation * tng.math.HALF_PI);

    // Truchet arc
    ctx.strokeStyle = tng.color.toCSS(c);
    ctx.lineWidth = 3;
    ctx.beginPath();
    ctx.arc(-tile.size/2, -tile.size/2, tile.size/2, 0, tng.math.HALF_PI);
    ctx.stroke();
    ctx.beginPath();
    ctx.arc(tile.size/2, tile.size/2, tile.size/2, tng.math.PI, tng.math.PI + tng.math.HALF_PI);
    ctx.stroke();

    ctx.restore();
  }
}

// Clean vector SVG export
function sketchSVG(w, h, params, tng, state) {
  let svg = `<svg xmlns="http://www.w3.org/2000/svg" width="${w}" height="${h}">`;
  svg += `<rect width="${w}" height="${h}" fill="${params.bgColor}"/>`;
  for (const tile of state.tiles) {
    const c = tng.color.oklch(0.7, 0.1, tile.hue);
    const cx = tile.x + tile.size/2, cy = tile.y + tile.size/2;
    const r = tile.size / 2;
    const css = tng.color.toCSS(c);
    // Generate arc paths based on rotation
    svg += `<g transform="translate(${cx},${cy}) rotate(${tile.rotation * 90})">`;
    svg += `<path d="M${-r},0 A${r},${r} 0 0,1 0,${-r}" fill="none" stroke="${css}" stroke-width="3"/>`;
    svg += `<path d="M${r},0 A${r},${r} 0 0,1 0,${r}" fill="none" stroke="${css}" stroke-width="3"/>`;
    svg += `</g>`;
  }
  svg += '</svg>';
  return svg;
}
```

## When to Choose thi.ng Mode vs. p5.js

| Consideration | thi.ng Mode | p5.js Mode |
|--------------|-------------|------------|
| Perceptual color needed | **Best choice** — Oklab/Oklch built-in | Must implement manually |
| SDF composition | **Best choice** — built-in primitives + smooth booleans | Must implement manually |
| SVG / plotter output | **Best choice** — `sketchSVG()` for clean vectors | Edge detection fallback only |
| Beginner-friendly | Moderate — Canvas 2D API | Easier — p5.js abstractions |
| Existing p5.js code | Convert manually | **Best choice** — direct use |
| Particle systems | Good — Canvas 2D is fast | Good — p5.js handles well |
| Pixel manipulation | Good — `ctx.getImageData()`/`putImageData()` | Good — `loadPixels()`/`updatePixels()` |
| 3D | Not supported — use Three.js modes | Not supported — use Three.js modes |

## Key Differences from p5.js Mode

1. **No p5 instance** — use standard Canvas 2D API (`ctx.fillRect`, `ctx.beginPath`, etc.)
2. **No `p.noise()`** — use `tng.noise.noise2d(x, y)` or `tng.noise.fbm(x, y)`
3. **No `p.random()`** — use `tng.random.float()`, `tng.random.range(min, max)`, etc.
4. **Color via `tng.color`** — perceptual by default, not sRGB
5. **State is explicit** — returned from `sketchSetup`, passed to `sketchDraw`
6. **SVG export is first-class** — implement `sketchSVG()` for clean vector output
