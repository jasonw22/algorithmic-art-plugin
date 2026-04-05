# Water & Ocean Rendering Reference

## Overview

Realistic water in shaders combines wave geometry (displacement), lighting (reflection,
refraction, Fresnel), and sub-surface effects (caustics, color absorption). Techniques
range from simple sine-wave surfaces to physically-based Gerstner wave models.

## Wave Models

### Sine Wave Sum (Simple)

Layer multiple sine waves with different frequencies and directions:

```glsl
float simpleWaves(vec2 p, float time) {
  float h = 0.0;
  // Wave 1
  h += sin(p.x * 1.0 + time * 1.2) * 0.3;
  // Wave 2 (different direction)
  h += sin(dot(p, vec2(0.7, 0.7)) * 1.5 + time * 0.8) * 0.2;
  // Wave 3 (higher frequency detail)
  h += sin(dot(p, vec2(-0.3, 0.9)) * 3.0 + time * 1.5) * 0.1;
  return h;
}
```

### Gerstner Waves (Physical)

Trochoid waves — each point moves in a circle, creating sharp crests and flat troughs.
This is the standard for realistic ocean surfaces.

```glsl
// Single Gerstner wave contribution
// dir: wave direction, steepness: 0–1 (Q parameter), wavelength, speed
vec3 gerstnerWave(vec2 p, float time, vec2 dir, float steepness, float wavelength, float speed) {
  float k = 6.28318 / wavelength;
  float c = speed;
  float a = steepness / k;  // amplitude from steepness
  float phase = k * (dot(dir, p) - c * time);

  return vec3(
    dir.x * a * cos(phase),     // x displacement
    a * sin(phase),              // y (height) displacement
    dir.y * a * cos(phase)       // z displacement
  );
}

// Combine multiple Gerstner waves for complex ocean surface
vec3 oceanSurface(vec2 p, float time) {
  vec3 displacement = vec3(0.0);

  // Primary swell
  displacement += gerstnerWave(p, time, normalize(vec2(1.0, 0.3)), 0.25, 10.0, 2.0);
  // Secondary swell
  displacement += gerstnerWave(p, time, normalize(vec2(0.5, 0.8)), 0.15, 6.0, 1.5);
  // Wind chop
  displacement += gerstnerWave(p, time, normalize(vec2(-0.2, 1.0)), 0.1, 3.0, 1.0);
  // Detail ripples
  displacement += gerstnerWave(p, time, normalize(vec2(0.8, -0.3)), 0.05, 1.5, 0.8);

  return displacement;
}

// Normal from Gerstner displacement (via finite differences)
vec3 oceanNormal(vec2 p, float time) {
  float eps = 0.01;
  vec3 c = oceanSurface(p, time);
  vec3 dx = oceanSurface(p + vec2(eps, 0.0), time) - c;
  vec3 dz = oceanSurface(p + vec2(0.0, eps), time) - c;
  return normalize(cross(
    vec3(eps, dx.y, 0.0),
    vec3(0.0, dz.y, eps)
  ));
}
```

### FBM-Based Waves (Artistic)

Layered noise gives organic, non-repeating waves. Less physical but great for stylized water:

```glsl
float fbmWaves(vec2 p, float time) {
  float h = 0.0;
  float amp = 0.5;
  float freq = 1.0;
  mat2 rot = mat2(0.8, 0.6, -0.6, 0.8);  // decorrelate octaves

  for (int i = 0; i < 5; i++) {
    h += amp * sin(dot(p * freq, vec2(0.7, 0.7)) + time * freq * 0.5);
    p = rot * p;
    amp *= 0.5;
    freq *= 2.0;
  }
  return h;
}
```

## Lighting

### Fresnel Reflection/Refraction

Water reflects more at glancing angles (Fresnel effect):

```glsl
float fresnelSchlick(float cosTheta, float f0) {
  return f0 + (1.0 - f0) * pow(1.0 - cosTheta, 5.0);
}

vec3 waterShading(vec3 p, vec3 normal, vec3 viewDir, vec3 sunDir, vec3 skyColor, vec3 deepColor) {
  // Fresnel
  float NdotV = max(dot(normal, viewDir), 0.0);
  float fresnel = fresnelSchlick(NdotV, 0.02);  // water F0 ≈ 0.02

  // Reflection (sky)
  vec3 reflDir = reflect(-viewDir, normal);
  vec3 reflection = skyColor;  // or sample environment

  // Refraction (underwater color, absorption)
  vec3 refraction = deepColor;

  // Blend by Fresnel
  vec3 color = mix(refraction, reflection, fresnel);

  // Specular highlight (sun)
  vec3 halfVec = normalize(sunDir + viewDir);
  float spec = pow(max(dot(normal, halfVec), 0.0), 256.0);
  color += vec3(1.0, 0.95, 0.8) * spec;

  return color;
}
```

### Subsurface Scattering Approximation

Light passing through thin wave crests creates a translucent glow:

```glsl
vec3 subsurfaceScattering(vec3 normal, vec3 viewDir, vec3 sunDir, float waveHeight) {
  // Light passes through thin wave crests
  float sss = pow(max(dot(viewDir, -sunDir), 0.0), 4.0);
  sss *= max(waveHeight, 0.0);  // stronger at wave peaks
  return vec3(0.1, 0.6, 0.3) * sss;  // greenish translucent color
}
```

### Underwater Color Absorption

Water absorbs red light first, then green, leaving blue at depth:

```glsl
vec3 underwaterColor(float depth) {
  // Absorption coefficients (red absorbed fastest)
  vec3 absorption = vec3(0.45, 0.08, 0.04);  // per meter
  return exp(-absorption * depth);
}
```

## Caustics

Light patterns on the sea floor from refraction through the surface:

```glsl
// Simplified caustic pattern using Voronoi
float caustics(vec2 p, float time) {
  float c = 0.0;
  // Two layers of Voronoi at different speeds create shimmering effect
  for (int i = 0; i < 2; i++) {
    float scale = 3.0 + float(i) * 2.0;
    float speed = 0.5 + float(i) * 0.3;
    vec2 uv = p * scale + time * speed * vec2(0.3, 0.1);
    vec2 id = floor(uv);
    vec2 f = fract(uv);

    float minDist = 1.0;
    for (int y = -1; y <= 1; y++) {
      for (int x = -1; x <= 1; x++) {
        vec2 neighbor = vec2(x, y);
        vec2 point = hash22(id + neighbor);
        point = 0.5 + 0.5 * sin(time * 0.5 + 6.28 * point);
        minDist = min(minDist, length(neighbor + point - f));
      }
    }
    c += minDist;
  }
  return pow(c * 0.5, 3.0) * 4.0;  // sharpen the pattern
}
```

## Foam

White foam at wave crests and shorelines:

```glsl
float foam(vec2 p, float waveHeight, float time) {
  // Foam appears at wave crests (high displacement)
  float foamMask = smoothstep(0.3, 0.5, waveHeight);

  // Noisy foam texture
  float noiseVal = fbm(p * 8.0 + time * 0.3, 4);
  foamMask *= smoothstep(0.2, 0.5, noiseVal);

  return foamMask;
}
```

## Complete Ocean Scene

```glsl
void main() {
  vec2 uv = (gl_FragCoord.xy - 0.5 * u_resolution) / u_resolution.y;
  vec3 ro = vec3(0.0, 3.0, 0.0);
  vec3 rd = normalize(vec3(uv.x, uv.y - 0.3, -1.0));
  vec3 sunDir = normalize(vec3(0.5, 0.3, 1.0));

  // Intersect with ocean plane (y = 0)
  float t = -ro.y / rd.y;

  vec3 color;
  if (t > 0.0 && rd.y < 0.0) {
    vec2 hitPos = ro.xz + rd.xz * t;

    // Wave displacement
    vec3 disp = oceanSurface(hitPos, u_time);
    vec3 normal = oceanNormal(hitPos, u_time);

    // View direction
    vec3 viewDir = normalize(ro - vec3(hitPos.x + disp.x, disp.y, hitPos.y + disp.z));

    // Shading
    vec3 skyCol = sky(reflect(-viewDir, normal), sunDir);
    vec3 deepCol = vec3(0.0, 0.05, 0.1);

    color = waterShading(vec3(hitPos.x, disp.y, hitPos.y), normal, viewDir, sunDir, skyCol, deepCol);
    color += subsurfaceScattering(normal, viewDir, sunDir, disp.y);

    // Foam at crests
    float f = foam(hitPos, disp.y, u_time);
    color = mix(color, vec3(0.9), f);

    // Distance fog
    float dist = length(vec2(t, ro.y));
    color = mix(color, vec3(0.5, 0.6, 0.7), 1.0 - exp(-dist * 0.003));
  } else {
    color = sky(rd, sunDir);
  }

  color = acesToneMap(color);
  gl_FragColor = vec4(color, 1.0);
}
```

## Key References

- **Jerry Tessendorf** — "Simulating Ocean Water" (Gerstner wave math, FFT ocean)
- **Shadertoy "ocean"** — community implementations (search iq's "Seascape")
- **GPU Gems 1, Ch. 1** — "Effective Water Simulation from Physical Models"
- **Inigo Quilez** — "Seascape" Shadertoy (compact, beautiful ocean shader)
