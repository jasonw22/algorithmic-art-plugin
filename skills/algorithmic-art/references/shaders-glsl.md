# GLSL Shaders — Fragment Shader Mode Reference

## Overview

Shader mode runs a full-screen fragment shader on the GPU. Every pixel is computed in parallel,
making this ideal for techniques that are prohibitively expensive on CPU: raymarching signed
distance fields, volumetric rendering, fractal exploration (Mandelbulb, Mandelbox),
GPU reaction-diffusion, domain repetition, and real-time fluid effects.

Shader mode builds on the Three.js 3D template (`assets/template-3d.html`) with
`renderMode: "shader"`. The template renders a fullscreen quad and passes your fragment shader
through a `ShaderMaterial`.

## Template Contract

Implement these functions:

```javascript
function shaderUniforms(params, seed) {
  // Return a Three.js uniforms object with your custom uniforms.
  // The template automatically provides these (do NOT redeclare):
  //   u_time       — float, elapsed seconds
  //   u_resolution — vec2, viewport size in pixels
  //   u_mouse      — vec2, normalized mouse position (0–1, bottom-left origin)
  //   u_seed       — float, current seed value
  return {
    u_scale: { value: params.scale || 1.0 },
    u_iterations: { value: params.iterations || 64 },
    u_color1: { value: new THREE.Color(params.color1 || "#ff4400") },
  };
}

function fragmentShader() {
  // Return a GLSL fragment shader string.
  return `
    uniform float u_time;
    uniform vec2 u_resolution;
    uniform vec2 u_mouse;
    // ... your uniforms ...
    void main() {
      vec2 uv = gl_FragCoord.xy / u_resolution;
      // ... your shader code ...
      gl_FragColor = vec4(color, 1.0);
    }
  `;
}

function shaderAnimate(uniforms, params, time) {
  // Optional: sync custom uniforms to param changes each frame.
  // u_time is updated automatically — only needed for custom uniforms.
  uniforms.u_scale.value = params.scale;
}
```

## GLSL Essentials

### Types
```glsl
float, vec2, vec3, vec4    // scalars and vectors
mat2, mat3, mat4           // matrices
int, ivec2, ivec3, ivec4   // integers
bool, bvec2, bvec3, bvec4  // booleans
sampler2D                   // textures
```

### Built-in Functions (most useful for generative art)
```glsl
// Math
abs, sign, floor, ceil, fract, mod, clamp, mix, step, smoothstep
min, max, pow, exp, log, sqrt, inversesqrt

// Trigonometry
sin, cos, tan, asin, acos, atan

// Vector
length(v), distance(a, b), dot(a, b), cross(a, b), normalize(v), reflect(I, N)

// Matrix
mat2(cos(a), -sin(a), sin(a), cos(a))  // 2D rotation matrix
```

### Common Patterns
```glsl
// Centered UV coordinates (-1 to 1, aspect-corrected)
vec2 uv = (gl_FragCoord.xy - 0.5 * u_resolution) / u_resolution.y;

// Polar coordinates
float r = length(uv);
float theta = atan(uv.y, uv.x);

// 2D rotation
mat2 rot(float a) { float c = cos(a), s = sin(a); return mat2(c, -s, s, c); }
uv *= rot(u_time * 0.5);

// Smooth pulse
float pulse(float x, float center, float width) {
  return smoothstep(center - width, center, x) - smoothstep(center, center + width, x);
}
```

## Noise Functions (GLSL)

### Hash (fast pseudo-random)
```glsl
float hash(vec2 p) {
  p = fract(p * vec2(123.34, 456.21));
  p += dot(p, p + 45.32);
  return fract(p.x * p.y);
}

vec2 hash2(vec2 p) {
  p = vec2(dot(p, vec2(127.1, 311.7)), dot(p, vec2(269.5, 183.3)));
  return fract(sin(p) * 43758.5453);
}
```

### Value Noise
```glsl
float valueNoise(vec2 p) {
  vec2 i = floor(p);
  vec2 f = fract(p);
  f = f * f * (3.0 - 2.0 * f);  // smoothstep
  float a = hash(i);
  float b = hash(i + vec2(1, 0));
  float c = hash(i + vec2(0, 1));
  float d = hash(i + vec2(1, 1));
  return mix(mix(a, b, f.x), mix(c, d, f.x), f.y);
}
```

### Simplex-style 2D Noise
```glsl
// Ashima Arts simplex noise (compact version)
vec3 mod289(vec3 x) { return x - floor(x / 289.0) * 289.0; }
vec2 mod289(vec2 x) { return x - floor(x / 289.0) * 289.0; }
vec3 permute(vec3 x) { return mod289((x * 34.0 + 1.0) * x); }

float snoise(vec2 v) {
  const vec4 C = vec4(0.211324865405187, 0.366025403784439,
                     -0.577350269189626, 0.024390243902439);
  vec2 i  = floor(v + dot(v, C.yy));
  vec2 x0 = v - i + dot(i, C.xx);
  vec2 i1 = (x0.x > x0.y) ? vec2(1, 0) : vec2(0, 1);
  vec4 x12 = x0.xyxy + C.xxzz;
  x12.xy -= i1;
  i = mod289(i);
  vec3 p = permute(permute(i.y + vec3(0, i1.y, 1)) + i.x + vec3(0, i1.x, 1));
  vec3 m = max(0.5 - vec3(dot(x0,x0), dot(x12.xy,x12.xy), dot(x12.zw,x12.zw)), 0.0);
  m = m * m; m = m * m;
  vec3 x = 2.0 * fract(p * C.www) - 1.0;
  vec3 h = abs(x) - 0.5;
  vec3 a0 = x - floor(x + 0.5);
  m *= 1.79284291400159 - 0.85373472095314 * (a0*a0 + h*h);
  vec3 g;
  g.x = a0.x * x0.x + h.x * x0.y;
  g.yz = a0.yz * x12.xz + h.yz * x12.yw;
  return 130.0 * dot(m, g);
}
```

### 3D Noise (for volumetric / animated 2D)
```glsl
// Use the 2D noise with a time-shifted coordinate:
float noise3D(vec3 p) {
  float xy = snoise(p.xy + p.z * 0.7);
  float yz = snoise(p.yz + p.x * 0.7);
  float xz = snoise(p.xz + p.y * 0.7);
  return (xy + yz + xz) / 3.0;
}
```

### Fractal Brownian Motion (fBm)
```glsl
float fbm(vec2 p, int octaves) {
  float value = 0.0;
  float amplitude = 0.5;
  float frequency = 1.0;
  for (int i = 0; i < octaves; i++) {
    value += amplitude * snoise(p * frequency);
    frequency *= 2.0;
    amplitude *= 0.5;
  }
  return value;
}
```

### Domain Warping
```glsl
float warpedNoise(vec2 p) {
  vec2 q = vec2(fbm(p, 4), fbm(p + vec2(5.2, 1.3), 4));
  vec2 r = vec2(fbm(p + 4.0 * q + vec2(1.7, 9.2), 4),
                fbm(p + 4.0 * q + vec2(8.3, 2.8), 4));
  return fbm(p + 4.0 * r, 4);
}
```

## Signed Distance Fields (SDFs)

SDFs are the foundation of raymarching. A distance function returns the shortest distance
from a point to a surface — negative inside, positive outside, zero on the surface.

### 3D Primitives
```glsl
float sdSphere(vec3 p, float r) {
  return length(p) - r;
}

float sdBox(vec3 p, vec3 b) {
  vec3 q = abs(p) - b;
  return length(max(q, 0.0)) + min(max(q.x, max(q.y, q.z)), 0.0);
}

float sdTorus(vec3 p, vec2 t) {
  vec2 q = vec2(length(p.xz) - t.x, p.y);
  return length(q) - t.y;
}

float sdCylinder(vec3 p, float r, float h) {
  vec2 d = abs(vec2(length(p.xz), p.y)) - vec2(r, h);
  return min(max(d.x, d.y), 0.0) + length(max(d, 0.0));
}

float sdPlane(vec3 p, vec3 n, float h) {
  return dot(p, n) + h;
}

float sdCapsule(vec3 p, vec3 a, vec3 b, float r) {
  vec3 ab = b - a, ap = p - a;
  float t = clamp(dot(ap, ab) / dot(ab, ab), 0.0, 1.0);
  return length(ap - ab * t) - r;
}

float sdOctahedron(vec3 p, float s) {
  p = abs(p);
  return (p.x + p.y + p.z - s) * 0.57735027;
}
```

### CSG Operations (combining shapes)
```glsl
float opUnion(float d1, float d2) { return min(d1, d2); }
float opSubtract(float d1, float d2) { return max(-d1, d2); }
float opIntersect(float d1, float d2) { return max(d1, d2); }

// Smooth union (organic blend)
float opSmoothUnion(float d1, float d2, float k) {
  float h = clamp(0.5 + 0.5 * (d2 - d1) / k, 0.0, 1.0);
  return mix(d2, d1, h) - k * h * (1.0 - h);
}

// Smooth subtraction
float opSmoothSubtract(float d1, float d2, float k) {
  float h = clamp(0.5 - 0.5 * (d2 + d1) / k, 0.0, 1.0);
  return mix(d2, -d1, h) + k * h * (1.0 - h);
}
```

### Domain Operations (repetition, distortion)
```glsl
// Infinite repetition
vec3 opRep(vec3 p, vec3 spacing) {
  return mod(p + 0.5 * spacing, spacing) - 0.5 * spacing;
}

// Finite repetition (N copies)
vec3 opRepLim(vec3 p, float spacing, vec3 limit) {
  return p - spacing * clamp(floor(p / spacing + 0.5), -limit, limit);
}

// Twist around Y
vec3 opTwist(vec3 p, float k) {
  float c = cos(k * p.y), s = sin(k * p.y);
  mat2 m = mat2(c, -s, s, c);
  return vec3(m * p.xz, p.y);
}

// Bend around X
vec3 opBend(vec3 p, float k) {
  float c = cos(k * p.x), s = sin(k * p.x);
  mat2 m = mat2(c, -s, s, c);
  vec2 bent = m * p.xy;
  return vec3(bent, p.z);
}

// Displacement (noise-based deformation)
float opDisplace(vec3 p, float d, float scale) {
  return d + snoise3D(p * scale) * 0.2;
}
```

## Raymarching

### Basic Raymarcher
```glsl
float sceneSDF(vec3 p) {
  // Combine your SDF primitives here
  float sphere = sdSphere(p, 1.0);
  float box = sdBox(p - vec3(2, 0, 0), vec3(0.8));
  return opSmoothUnion(sphere, box, 0.5);
}

vec3 calcNormal(vec3 p) {
  const float h = 0.0001;
  const vec2 k = vec2(1, -1);
  return normalize(
    k.xyy * sceneSDF(p + k.xyy * h) +
    k.yyx * sceneSDF(p + k.yyx * h) +
    k.yxy * sceneSDF(p + k.yxy * h) +
    k.xxx * sceneSDF(p + k.xxx * h)
  );
}

float raymarch(vec3 ro, vec3 rd) {
  float t = 0.0;
  for (int i = 0; i < 128; i++) {
    vec3 p = ro + rd * t;
    float d = sceneSDF(p);
    if (d < 0.001) return t;
    if (t > 100.0) break;
    t += d;
  }
  return -1.0;  // miss
}

void main() {
  vec2 uv = (gl_FragCoord.xy - 0.5 * u_resolution) / u_resolution.y;

  // Camera
  vec3 ro = vec3(0, 0, 3);                    // ray origin
  vec3 rd = normalize(vec3(uv, -1.0));         // ray direction

  float t = raymarch(ro, rd);

  vec3 color = vec3(0.05);  // background
  if (t > 0.0) {
    vec3 p = ro + rd * t;
    vec3 n = calcNormal(p);
    vec3 lightDir = normalize(vec3(1, 1, 1));
    float diff = max(dot(n, lightDir), 0.0);
    float spec = pow(max(dot(reflect(-lightDir, n), -rd), 0.0), 32.0);
    color = vec3(0.2, 0.5, 0.8) * (0.2 + 0.8 * diff) + vec3(1.0) * spec * 0.5;
  }

  gl_FragColor = vec4(color, 1.0);
}
```

### Ambient Occlusion (cheap)
```glsl
float calcAO(vec3 p, vec3 n) {
  float occ = 0.0;
  float sca = 1.0;
  for (int i = 0; i < 5; i++) {
    float h = 0.01 + 0.12 * float(i);
    float d = sceneSDF(p + n * h);
    occ += (h - d) * sca;
    sca *= 0.95;
  }
  return clamp(1.0 - 3.0 * occ, 0.0, 1.0);
}
```

### Soft Shadows
```glsl
float softShadow(vec3 ro, vec3 rd, float mint, float maxt, float k) {
  float res = 1.0;
  float t = mint;
  for (int i = 0; i < 64; i++) {
    float h = sceneSDF(ro + rd * t);
    res = min(res, k * h / t);
    if (h < 0.001 || t > maxt) break;
    t += clamp(h, 0.02, 0.1);
  }
  return clamp(res, 0.0, 1.0);
}
```

## 3D Fractals

### Mandelbulb
```glsl
float mandelbulb(vec3 p, float power) {
  vec3 z = p;
  float dr = 1.0;
  float r = 0.0;

  for (int i = 0; i < 15; i++) {
    r = length(z);
    if (r > 2.0) break;

    float theta = acos(z.z / r);
    float phi = atan(z.y, z.x);
    dr = pow(r, power - 1.0) * power * dr + 1.0;

    float zr = pow(r, power);
    theta *= power;
    phi *= power;

    z = zr * vec3(sin(theta)*cos(phi), sin(phi)*sin(theta), cos(theta));
    z += p;
  }

  return 0.5 * log(r) * r / dr;
}
```

### Menger Sponge
```glsl
float mengerSponge(vec3 p, int iterations) {
  float d = sdBox(p, vec3(1.0));
  float s = 1.0;
  for (int i = 0; i < iterations; i++) {
    vec3 a = mod(p * s, 2.0) - 1.0;
    s *= 3.0;
    vec3 r = abs(1.0 - 3.0 * abs(a));
    float c = sdBox(r, vec3(1.0)) / s;
    d = max(d, c);
  }
  return d;
}
```

## Color Techniques

### Palette Function (Inigo Quilez)
```glsl
// Creates smooth color palettes from 4 vec3 parameters
vec3 palette(float t, vec3 a, vec3 b, vec3 c, vec3 d) {
  return a + b * cos(6.28318 * (c * t + d));
}

// Example palettes:
// Rainbow:  palette(t, vec3(0.5), vec3(0.5), vec3(1.0), vec3(0.00, 0.33, 0.67))
// Sunset:   palette(t, vec3(0.5), vec3(0.5), vec3(1.0), vec3(0.00, 0.10, 0.20))
// Ocean:    palette(t, vec3(0.5), vec3(0.5), vec3(1.0, 1.0, 0.5), vec3(0.80, 0.90, 0.30))
// Fire:     palette(t, vec3(0.5), vec3(0.5), vec3(2.0, 1.0, 0.0), vec3(0.50, 0.20, 0.25))
```

### Orbit Trap Coloring (for fractals)
```glsl
// Track minimum distance to geometric features during iteration
float minDist = 1e10;
vec3 trapColor;
for (int i = 0; i < maxIter; i++) {
  // ... fractal iteration ...
  float d = length(z.xy);  // distance to origin in xy-plane
  if (d < minDist) {
    minDist = d;
    trapColor = palette(d, ...);
  }
}
```

## Volumetric Effects

### Ray-marched Fog / Clouds
```glsl
vec3 volumetric(vec3 ro, vec3 rd, float maxDist) {
  vec3 color = vec3(0.0);
  float transmittance = 1.0;
  float stepSize = maxDist / 64.0;

  for (float t = 0.0; t < maxDist; t += stepSize) {
    vec3 p = ro + rd * t;
    float density = fbm(p * 0.5 + u_time * 0.1, 5) * 0.5 + 0.5;
    density = max(density - 0.4, 0.0) * 2.0;  // threshold

    if (density > 0.01) {
      vec3 lightColor = vec3(1.0, 0.9, 0.7) * density;
      color += transmittance * lightColor * stepSize;
      transmittance *= exp(-density * stepSize * 2.0);
    }

    if (transmittance < 0.01) break;
  }
  return color;
}
```

## Connecting Params to Uniforms

Map sidebar parameters to shader uniforms for interactive control:

```javascript
const PARAMS = {
  power:    { value: 8.0, min: 2, max: 16, step: 0.1, label: "Fractal Power", folder: "Structure" },
  detail:   { value: 64, min: 16, max: 256, step: 1, label: "Ray Steps", folder: "Quality" },
  palette:  { value: "Rainbow", options: ["Rainbow", "Sunset", "Ocean", "Fire"], label: "Palette", folder: "Color" },
  bgColor:  { value: "#0a0a0a", type: "color", label: "Background", folder: "Color" },
};

function shaderUniforms(params, seed) {
  return {
    u_power: { value: params.power },
    u_maxSteps: { value: params.detail },
    u_paletteId: { value: ["Rainbow", "Sunset", "Ocean", "Fire"].indexOf(params.palette) },
  };
}

function shaderAnimate(uniforms, params, time) {
  uniforms.u_power.value = params.power;
  uniforms.u_maxSteps.value = params.detail;
  uniforms.u_paletteId.value = ["Rainbow", "Sunset", "Ocean", "Fire"].indexOf(params.palette);
}
```

## Pixel Ratio Gotcha

The template's `u_resolution` uniform accounts for `renderer.getPixelRatio()` automatically —
it reports the actual canvas pixel dimensions, not the CSS dimensions. This means
`gl_FragCoord.xy / u_resolution` correctly maps to [0,1] on all displays.

If you ever construct resolution values yourself (e.g., for a secondary render target or
a custom uniform), always multiply by the pixel ratio:
```javascript
const pr = renderer.getPixelRatio();
myUniform.value.set(window.innerWidth * pr, window.innerHeight * pr);
```
Without this, shaders will only fill a fraction of the canvas on high-DPI screens.

## Performance Tips

- Keep raymarch step count as low as possible (64–128 for most scenes)
- Use early termination (`if (t > maxDist) break`)
- Lower the resolution for complex shaders: `renderer.setPixelRatio(1)` in sceneSetup
- Use `smoothstep` instead of `if` for anti-aliased edges
- Precompute expensive values outside loops when possible
- For animated fractals, use lower iteration counts and compensate with post-processing

## Key References

- **Inigo Quilez** (iq) — [iquilezles.org/articles](https://iquilezles.org/articles/) —
  definitive SDF reference, distance functions, soft shadows, AO, palettes
- **Shadertoy** — massive community gallery of fragment shaders
- **The Book of Shaders** — beginner-friendly GLSL tutorial
- **Mercury SDF library** — comprehensive collection of SDF operations
