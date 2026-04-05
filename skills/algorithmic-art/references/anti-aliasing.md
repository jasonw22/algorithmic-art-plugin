# Anti-Aliasing Strategies Reference

## Overview

Aliasing — jagged edges, shimmering, Moiré patterns — occurs when a continuous signal
is sampled at discrete pixel locations. Anti-aliasing (AA) techniques smooth these
artifacts. In shader art, AA is especially important for SDFs, sharp edges, and
high-frequency patterns.

## Analytical Anti-Aliasing

The cheapest and best method: use `smoothstep` to soften edges over a pixel-width band.

```glsl
// Anti-aliased edge from SDF
float aaEdge(float d, vec2 resolution) {
  float pixelWidth = 1.0 / resolution.y;  // approximate pixel size in UV space
  return smoothstep(pixelWidth, -pixelWidth, d);
}

// For arbitrary functions: use fwidth() for automatic pixel-scale derivative
float aaEdgeFwidth(float d) {
  float fw = fwidth(d);  // screen-space rate of change
  return smoothstep(fw, -fw, d);
}

// Anti-aliased line
float aaLine(float d, float thickness) {
  float fw = fwidth(d);
  return smoothstep(thickness + fw, thickness - fw, abs(d));
}

// Anti-aliased circle (from SDF)
float aaCircle(vec2 p, float r) {
  float d = length(p) - r;
  return 1.0 - smoothstep(-fwidth(d), fwidth(d), d);
}
```

### fwidth() Explained

`fwidth(x)` returns `abs(dFdx(x)) + abs(dFdy(x))` — the rate of change of `x` across
neighboring pixels. This gives you the pixel scale at any point, accounting for perspective,
zoom, and UV distortion. It's the recommended way to compute anti-aliasing widths.

```glsl
// Pattern anti-aliasing: smooth version of step()
float aaStep(float edge, float x) {
  float fw = fwidth(x);
  return smoothstep(edge - fw * 0.5, edge + fw * 0.5, x);
}

// Smooth version of mod-based patterns
float aaGrid(vec2 p, float lineWidth) {
  vec2 fw = fwidth(p);
  vec2 grid = abs(fract(p - 0.5) - 0.5) / fw;
  float line = min(grid.x, grid.y);
  return 1.0 - min(line, 1.0);
}
```

## Supersampling (SSAA / MSAA)

Render multiple sub-pixel samples and average them. Brute-force but universally effective.

### Fixed Grid (2×2)

```glsl
void main() {
  vec3 total = vec3(0.0);
  vec2 pixelSize = 1.0 / u_resolution;

  // 2×2 grid: 4 samples per pixel
  for (int y = 0; y < 2; y++) {
    for (int x = 0; x < 2; x++) {
      vec2 offset = (vec2(x, y) - 0.5) * 0.5;
      vec2 uv = (gl_FragCoord.xy + offset) / u_resolution;
      total += computeColor(uv);
    }
  }

  gl_FragColor = vec4(total / 4.0, 1.0);
}
```

### Rotated Grid (RGSS — 4 samples, better quality than 2×2)

```glsl
void main() {
  vec3 total = vec3(0.0);

  // Rotated grid offsets (better coverage than axis-aligned)
  const vec2 offsets[4] = vec2[4](
    vec2(-0.125, -0.375),
    vec2( 0.375, -0.125),
    vec2(-0.375,  0.125),
    vec2( 0.125,  0.375)
  );

  for (int i = 0; i < 4; i++) {
    vec2 uv = (gl_FragCoord.xy + offsets[i]) / u_resolution;
    total += computeColor(uv);
  }

  gl_FragColor = vec4(total / 4.0, 1.0);
}
```

### Stochastic / Jittered

Random sub-pixel offsets (requires good PRNG, see `path-tracing.md`):

```glsl
void main() {
  vec3 total = vec3(0.0);
  int samples = 8;

  for (int i = 0; i < samples; i++) {
    vec2 jitter = hash22(gl_FragCoord.xy + float(i) * 100.0) - 0.5;
    vec2 uv = (gl_FragCoord.xy + jitter) / u_resolution;
    total += computeColor(uv);
  }

  gl_FragColor = vec4(total / float(samples), 1.0);
}
```

### Adaptive Supersampling

Only supersample where needed (edges, high contrast areas):

```glsl
void main() {
  vec2 uv = gl_FragCoord.xy / u_resolution;
  vec3 center = computeColor(uv);

  // Check if neighbors differ significantly
  vec3 right = computeColor(uv + vec2(1, 0) / u_resolution);
  vec3 up = computeColor(uv + vec2(0, 1) / u_resolution);
  float diff = length(center - right) + length(center - up);

  if (diff > 0.1) {
    // High contrast: supersample
    vec3 total = center;
    total += computeColor(uv + vec2(-0.25, -0.25) / u_resolution);
    total += computeColor(uv + vec2( 0.25, -0.25) / u_resolution);
    total += computeColor(uv + vec2( 0.25,  0.25) / u_resolution);
    center = total / 4.0;
  }

  gl_FragColor = vec4(center, 1.0);
}
```

## Temporal Anti-Aliasing (TAA)

Accumulate sub-pixel jitter across frames (pairs naturally with path tracing):

```glsl
// Each frame uses a different sub-pixel offset
uniform int u_frame;

void main() {
  // Halton-like sequence for temporal jitter
  vec2 jitter = vec2(
    fract(float(u_frame) * 0.5 + 0.25),
    fract(float(u_frame) * 0.333 + 0.125)
  ) - 0.5;

  vec2 uv = (gl_FragCoord.xy + jitter * 0.5) / u_resolution;
  vec3 color = computeColor(uv);

  // Blend with previous frame (exponential moving average)
  vec3 prev = texture2D(u_prevFrame, gl_FragCoord.xy / u_resolution).rgb;
  color = mix(prev, color, 0.1);  // 0.1 = strong temporal smoothing

  gl_FragColor = vec4(color, 1.0);
}
```

## When to Use Which

| Technique | Cost | Quality | Best For |
|-----------|------|---------|----------|
| `smoothstep`/`fwidth` | Free | Good for edges | SDF edges, grid lines, patterns |
| 2×2 grid | 4× | Decent | Simple shaders where cost is acceptable |
| Rotated grid (RGSS) | 4× | Better than 2×2 | General purpose, diagonal edges |
| Stochastic | N× | Good (noisy) | Complex scenes, pairs with denoising |
| Temporal AA | ~Free | Excellent (ghosting risk) | Progressive renders, path tracing |
| Adaptive | 1–4× | Good | Complex scenes with large flat areas |

## Key References

- **Inigo Quilez** — filtering and anti-aliasing of procedural patterns
- **Real-Time Rendering** — Akenine-Möller et al., Chapter on Anti-Aliasing
- **GPU Gems 2, Ch. 22** — "Fast Prefiltered Lines"
- **The Book of Shaders** — anti-aliased shapes with `smoothstep`
