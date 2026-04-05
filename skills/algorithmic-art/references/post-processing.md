# Post-Processing Effects Reference

## Overview

Post-processing effects are applied after the main render, transforming the raw image into
a polished final output. These "finishing touches" dramatically improve perceived quality
with minimal code. They work in both shader mode (applied in `main()` after computing color)
and scene mode (via Three.js post-processing pipeline).

## GLSL Post-Processing (Shader Mode)

All effects below operate on a `vec3 color` that you've already computed. Apply them
at the end of `main()` before writing to `gl_FragColor`.

### Vignette

Darkens edges to draw focus to the center. The most universal finishing effect.

```glsl
vec3 vignette(vec3 color, vec2 uv, float intensity, float softness) {
  float d = length(uv - 0.5) * 2.0;  // 0 at center, ~1.4 at corners
  float vig = smoothstep(1.0, softness, d);
  return color * mix(1.0 - intensity, 1.0, vig);
}

// Usage: color = vignette(color, uv, 0.4, 0.5);
```

### Film Grain

Adds organic noise that breaks up banding and gives analog texture.

```glsl
vec3 filmGrain(vec3 color, vec2 fragCoord, float time, float intensity) {
  float grain = fract(sin(dot(fragCoord + time * 100.0, vec2(12.9898, 78.233))) * 43758.5453);
  grain = (grain - 0.5) * intensity;
  return color + grain;
}

// Usage: color = filmGrain(color, gl_FragCoord.xy, u_time, 0.05);
```

### Chromatic Aberration

Shifts RGB channels by different amounts, simulating lens imperfection.

```glsl
vec3 chromaticAberration(sampler2D tex, vec2 uv, float amount) {
  vec2 dir = (uv - 0.5) * amount;
  float r = texture2D(tex, uv + dir).r;
  float g = texture2D(tex, uv).g;
  float b = texture2D(tex, uv - dir).b;
  return vec3(r, g, b);
}

// Without texture (inline, for single-pass shaders):
vec3 chromaticAberrationInline(vec2 uv, float amount) {
  // Requires computing color at offset UVs — call your color function 3 times
  vec2 dir = (uv - 0.5) * amount;
  float r = computeColor(uv + dir).r;
  float g = computeColor(uv).g;
  float b = computeColor(uv - dir).b;
  return vec3(r, g, b);
}
```

### Bloom (Simplified Single-Pass)

Real bloom requires multipass (threshold → blur → composite). A single-pass approximation
adds glow around bright areas:

```glsl
vec3 cheapBloom(vec3 color, float threshold, float intensity) {
  vec3 bright = max(color - threshold, 0.0);
  return color + bright * intensity;
}

// Usage: color = cheapBloom(color, 0.7, 1.5);
```

### Bloom (Proper Multipass)

For real bloom, use the multipass buffer system (see `multipass-buffers.md`):

1. **Pass 1** — Render scene, threshold bright pixels into buffer
2. **Pass 2** — Horizontal Gaussian blur on the threshold buffer
3. **Pass 3** — Vertical Gaussian blur on the result
4. **Pass 4** — Composite: `finalColor = sceneColor + bloomColor * intensity`

```glsl
// Gaussian blur kernel (13-tap, separable)
vec3 gaussianBlur(sampler2D tex, vec2 uv, vec2 direction, vec2 resolution) {
  vec2 texel = direction / resolution;
  vec3 result = vec3(0.0);
  float weights[7] = float[](0.1964, 0.2969, 0.0945, 0.0104, 0.0002, 0.0, 0.0);
  // Center
  result += texture2D(tex, uv).rgb * 0.1964;
  // Symmetric taps
  for (int i = 1; i <= 6; i++) {
    float w = weights[i];
    result += texture2D(tex, uv + texel * float(i)).rgb * w;
    result += texture2D(tex, uv - texel * float(i)).rgb * w;
  }
  return result;
}
```

### CRT / Scanline Effect

```glsl
vec3 crtEffect(vec3 color, vec2 fragCoord, float scanlineIntensity, float curvature) {
  vec2 uv = fragCoord / u_resolution;

  // Barrel distortion (CRT curvature)
  vec2 centered = uv * 2.0 - 1.0;
  centered *= 1.0 + curvature * dot(centered, centered);
  uv = centered * 0.5 + 0.5;

  // Discard pixels outside the curved screen
  if (uv.x < 0.0 || uv.x > 1.0 || uv.y < 0.0 || uv.y > 1.0) return vec3(0.0);

  // Scanlines
  float scanline = sin(fragCoord.y * 3.14159) * scanlineIntensity;
  color *= 1.0 - scanline * 0.3;

  // RGB sub-pixel pattern
  float px = mod(fragCoord.x, 3.0);
  if (px < 1.0) color *= vec3(1.2, 0.8, 0.8);
  else if (px < 2.0) color *= vec3(0.8, 1.2, 0.8);
  else color *= vec3(0.8, 0.8, 1.2);

  return color;
}
```

### Barrel / Pincushion Distortion

```glsl
vec2 barrelDistort(vec2 uv, float k1, float k2) {
  vec2 centered = uv - 0.5;
  float r2 = dot(centered, centered);
  float distortion = 1.0 + k1 * r2 + k2 * r2 * r2;
  return centered * distortion + 0.5;
}
```

### Tone Mapping

Convert HDR values to displayable LDR range. Essential for raymarched scenes with
bright highlights.

```glsl
// ACES filmic tone mapping (industry standard)
vec3 acesToneMap(vec3 x) {
  float a = 2.51;
  float b = 0.03;
  float c = 2.43;
  float d = 0.59;
  float e = 0.14;
  return clamp((x * (a * x + b)) / (x * (c * x + d) + e), 0.0, 1.0);
}

// Reinhard (simple, good for soft look)
vec3 reinhardToneMap(vec3 color) {
  return color / (1.0 + color);
}

// Reinhard extended (with white point)
vec3 reinhardExtended(vec3 color, float whitePoint) {
  vec3 num = color * (1.0 + color / (whitePoint * whitePoint));
  return num / (1.0 + color);
}

// Uncharted 2 filmic (warm, cinematic)
vec3 uncharted2Helper(vec3 x) {
  float A = 0.15, B = 0.50, C = 0.10, D = 0.20, E = 0.02, F = 0.30;
  return ((x * (A * x + C * B) + D * E) / (x * (A * x + B) + D * F)) - E / F;
}
vec3 uncharted2ToneMap(vec3 color, float exposure) {
  float W = 11.2;
  vec3 curr = uncharted2Helper(color * exposure);
  vec3 whiteScale = 1.0 / uncharted2Helper(vec3(W));
  return curr * whiteScale;
}
```

### Gamma Correction

```glsl
// Linear → sRGB (apply last, after tone mapping)
vec3 gammaCorrect(vec3 color, float gamma) {
  return pow(color, vec3(1.0 / gamma));
}

// sRGB → Linear (apply first, when reading sRGB textures)
vec3 gammaToLinear(vec3 color) {
  return pow(color, vec3(2.2));
}
```

### Color Grading

```glsl
// Contrast adjustment (around midpoint 0.5)
vec3 adjustContrast(vec3 color, float contrast) {
  return (color - 0.5) * contrast + 0.5;
}

// Saturation adjustment
vec3 adjustSaturation(vec3 color, float saturation) {
  float luma = dot(color, vec3(0.2126, 0.7152, 0.0722));
  return mix(vec3(luma), color, saturation);
}

// Color temperature shift (warm/cool)
vec3 adjustTemperature(vec3 color, float temp) {
  // temp: -1 (cool/blue) to +1 (warm/orange)
  color.r += temp * 0.1;
  color.b -= temp * 0.1;
  return color;
}
```

## Composing a Post-Processing Stack

Apply effects in this order for best results:

```glsl
void main() {
  vec2 uv = gl_FragCoord.xy / u_resolution;

  // 1. Compute scene color (raymarching, SDF, etc.)
  vec3 color = computeScene(uv);

  // 2. Tone mapping (if scene has HDR values)
  color = acesToneMap(color);

  // 3. Color grading
  color = adjustContrast(color, 1.1);
  color = adjustSaturation(color, 1.2);

  // 4. Chromatic aberration (subtle)
  // color = chromaticAberrationInline(uv, 0.002);

  // 5. Vignette
  color = vignette(color, uv, 0.3, 0.6);

  // 6. Film grain (last, before gamma)
  color = filmGrain(color, gl_FragCoord.xy, u_time, 0.03);

  // 7. Gamma correction (if not handled by Three.js)
  // color = gammaCorrect(color, 2.2);

  gl_FragColor = vec4(color, 1.0);
}
```

## Three.js Scene Mode Post-Processing

For Three.js Scene mode, use the built-in post-processing pipeline:

```javascript
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';

function sceneSetup(THREE, scene, camera, renderer, params, seed) {
  const composer = new EffectComposer(renderer);
  composer.addPass(new RenderPass(scene, camera));

  const bloom = new UnrealBloomPass(
    new THREE.Vector2(window.innerWidth, window.innerHeight),
    params.bloomStrength,  // 0.0–3.0
    params.bloomRadius,    // 0.0–1.0
    params.bloomThreshold  // 0.0–1.0
  );
  composer.addPass(bloom);

  return { composer, bloom };
}

function sceneAnimate(THREE, scene, camera, state, params, time, delta) {
  state.bloom.strength = params.bloomStrength;
  state.composer.render();
  return true;  // return true to skip the template's default render call
}
```

## Key References

- **Shadertoy "Post Process" tag** — community examples of every effect
- **Three.js post-processing docs** — EffectComposer, built-in passes
- **Inigo Quilez** — film grain, vignette, and color grading patterns
- **Learn OpenGL** — HDR, bloom, gamma correction tutorials
