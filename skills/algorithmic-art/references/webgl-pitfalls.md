# WebGL Pitfalls & Debugging Reference

## Overview

Common gotchas, precision issues, and debugging techniques for GLSL shaders running in
WebGL (via Three.js). Knowing these saves hours of debugging mysterious visual artifacts.

## Precision Issues

### Float Precision

WebGL fragment shaders default to `mediump` on mobile. Always declare precision:

```glsl
precision highp float;  // Required for most generative art
```

Without this, you'll see banding, jitter, and broken noise on mobile devices.

### Large Coordinate Values

Floats lose precision far from origin. For noise/patterns at large coordinates:

```glsl
// BAD: coordinates grow with time, precision degrades
float n = snoise(uv * 100.0 + u_time * 50.0);

// GOOD: keep coordinates bounded by using fract or mod
float n = snoise(uv * 100.0 + fract(u_time * 0.1) * 500.0);

// GOOD: use domain offset instead of time-based drift
vec2 p = uv * 100.0;
p += vec2(sin(u_time), cos(u_time)) * 10.0;
```

### Integer Overflow in Hashes

Hash functions using `sin(x) * 43758.5453` break for very large inputs. Wrap first:

```glsl
// BAD: dot product can be huge
float h = fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453);

// BETTER: reduce input range first
vec2 q = fract(p * 0.1031);
q += dot(q, q.yx + 33.33);
float h = fract((q.x + q.y) * q.x);
```

## Common Visual Bugs

### Black Screen

Checklist:
1. **Division by zero**: `1.0 / x` where x can be 0 → use `1.0 / (x + 0.001)`
2. **NaN propagation**: `sqrt(negative)`, `log(0)`, `0.0/0.0` → clamp inputs
3. **Uniforms not set**: check that `shaderUniforms` returns all declared uniforms
4. **gl_FragColor never written**: every code path must write to output
5. **Infinite loop**: ensure raymarch/loop has a guaranteed exit condition

```glsl
// Defensive: catch NaN and replace with black
if (any(isnan(color)) || any(isinf(color))) color = vec3(0.0);
```

### Banding / Posterization

Insufficient precision in gradients. Fixes:
- Use `highp float`
- Add dithering: `color += (hash(gl_FragCoord.xy) - 0.5) / 255.0`
- Use smooth interpolation (`smoothstep`, `mix`) instead of hard thresholds

### Z-Fighting / Flickering

In raymarching, the surface threshold is too large or too small:

```glsl
// BAD: fixed threshold fails at different distances
if (d < 0.001) return t;

// GOOD: scale threshold with distance (compensates for float precision)
if (d < 0.001 * t) return t;
```

### Seams in Repeated Domains

`mod` can produce seams at cell boundaries. Ensure SDF evaluation checks neighboring cells:

```glsl
// Check 3x3 neighborhood for finite repetition
vec3 closest = vec3(1e10);
for (int z = -1; z <= 1; z++)
for (int x = -1; x <= 1; x++) {
  vec3 cell = opRepLim(p + vec3(x, 0, z) * spacing, spacing, limit);
  closest = min(closest, vec3(sdShape(cell)));
}
```

### Mach Bands (Perceived Banding on Smooth Gradients)

The eye exaggerates contrast at gradient transitions. Mitigate with:
- Gamma correction: `pow(color, vec3(1.0/2.2))`
- Dithering (see above)
- Subtle noise overlay

## Debugging Techniques

### Visual Debuggers

Encode internal values as color to diagnose issues:

```glsl
// Visualize normals (map [-1,1] to [0,1])
color = n * 0.5 + 0.5;

// Visualize UV coordinates
color = vec3(uv, 0.0);

// Visualize distance field (bands)
color = vec3(0.5 + 0.5 * sin(d * 50.0));

// Visualize step count (heatmap: green=few, red=many)
float ratio = float(steps) / float(MAX_STEPS);
color = mix(vec3(0, 1, 0), vec3(1, 0, 0), ratio);

// Visualize depth (white=close, black=far)
color = vec3(1.0 - t / maxDist);

// Checkerboard UV test pattern
float check = mod(floor(uv.x * 10.0) + floor(uv.y * 10.0), 2.0);
color = vec3(check);
```

### Performance Profiling

- **Step count heatmap**: shows where the raymarcher works hardest
- **Resolution reduction**: `renderer.setPixelRatio(0.5)` to test if GPU-bound
- **Simplify incrementally**: comment out expensive operations one by one
- **FPS counter**: `performance.now()` between frames in JavaScript

## Mobile / Cross-Platform Issues

### No `dFdx`/`dFdy`/`fwidth` in Some Contexts

These require `OES_standard_derivatives` extension (available in WebGL 1 with extension,
always available in WebGL 2). Three.js handles this automatically in most cases.

### Texture Limits

- WebGL 1: 8 texture units minimum
- WebGL 2: 16 texture units minimum
- Maximum texture size: typically 4096×4096 (mobile) to 16384×16384 (desktop)
- Check: `gl.getParameter(gl.MAX_TEXTURE_SIZE)`

### Loop Limits

Some mobile GPUs unroll all loops at compile time. Avoid:
- Loops with very high iteration counts (>256 on mobile)
- Dynamic loop bounds (use `#define MAX_STEPS 128` and `if (i >= steps) break`)

### Rendering Differences

- sRGB handling varies between devices
- Floating-point texture support requires `EXT_color_buffer_float` (WebGL 2)
- `THREE.FloatType` render targets may fall back to `HalfFloatType` on mobile

## Useful GLSL Diagnostics

```glsl
// Is this value NaN?
bool isNaN(float x) { return x != x; }

// Is this value infinite?
bool isInf(float x) { return abs(x) > 1e30; }

// Clamp to safe range (prevent NaN/Inf propagation)
vec3 safeColor(vec3 c) {
  return clamp(c, 0.0, 100.0);  // allow HDR but prevent infinity
}
```

## Key References

- **WebGL2 Fundamentals** — webgl2fundamentals.org
- **Three.js WebGL debugging** — Three.js docs on common issues
- **Khronos WebGL Wiki** — official compatibility and precision tables
- **Shadertoy FAQ** — common shader debugging techniques
