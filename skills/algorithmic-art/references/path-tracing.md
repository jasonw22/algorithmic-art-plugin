# Path Tracing / Global Illumination Reference

## Overview

Path tracing simulates light transport by tracing rays that bounce randomly through a
scene, accumulating color at each bounce. Over many samples (frames), this converges to
physically accurate global illumination — soft shadows, color bleeding, caustics, ambient
occlusion — all emerge naturally from the algorithm.

In real-time shaders, we accumulate samples across frames using the multipass buffer
system (see `multipass-buffers.md`), progressively refining the image.

## Core Algorithm

```
For each pixel:
  1. Cast a ray from the camera through the pixel
  2. If it hits a surface:
     a. Sample a random direction in the hemisphere above the surface
     b. Multiply accumulated color by the surface's BRDF
     c. Continue tracing from the hit point in the new direction
  3. If it hits a light source (or the sky), add the light contribution
  4. Repeat for N bounces (Russian Roulette for termination)
  5. Average the result with previous frames
```

## GLSL Implementation

### Random Number Generation

Path tracing needs high-quality per-pixel random numbers that change each frame:

```glsl
// Per-pixel RNG state seeded by pixel position + frame number
uint rngState;

void initRNG(vec2 fragCoord, int frame) {
  rngState = uint(fragCoord.x) * 1973u + uint(fragCoord.y) * 9277u + uint(frame) * 26699u;
}

uint pcgHash(uint input) {
  uint state = input * 747796405u + 2891336453u;
  uint word = ((state >> ((state >> 28u) + 4u)) ^ state) * 277803737u;
  return (word >> 22u) ^ word;
}

float randomFloat() {
  rngState = pcgHash(rngState);
  return float(rngState) / 4294967295.0;
}

vec2 randomVec2() {
  return vec2(randomFloat(), randomFloat());
}
```

### Hemisphere Sampling

Sample random directions proportional to cosine (Lambertian importance sampling):

```glsl
// Cosine-weighted hemisphere sample (importance sampling for diffuse)
vec3 cosineHemisphere(vec3 normal) {
  float r1 = randomFloat();
  float r2 = randomFloat();

  // Map to hemisphere using Malley's method
  float r = sqrt(r1);
  float theta = 6.28318 * r2;
  float x = r * cos(theta);
  float y = r * sin(theta);
  float z = sqrt(1.0 - r1);

  // Build TBN from normal
  vec3 up = abs(normal.y) < 0.999 ? vec3(0, 1, 0) : vec3(1, 0, 0);
  vec3 tangent = normalize(cross(up, normal));
  vec3 bitangent = cross(normal, tangent);

  return normalize(tangent * x + bitangent * y + normal * z);
}

// Uniform hemisphere sample (for testing / comparison)
vec3 uniformHemisphere(vec3 normal) {
  float r1 = randomFloat();
  float r2 = randomFloat();

  float sinTheta = sqrt(1.0 - r1 * r1);
  float phi = 6.28318 * r2;
  vec3 dir = vec3(sinTheta * cos(phi), r1, sinTheta * sin(phi));

  // Build TBN
  vec3 up = abs(normal.y) < 0.999 ? vec3(0, 1, 0) : vec3(1, 0, 0);
  vec3 tangent = normalize(cross(up, normal));
  vec3 bitangent = cross(normal, tangent);

  return normalize(tangent * dir.x + bitangent * dir.z + normal * dir.y);
}
```

### Fibonacci Sphere Sampling

Quasi-uniform distribution on a hemisphere (for AO or fixed-sample-count effects):

```glsl
vec3 fibonacciHemisphere(int i, int n, vec3 normal) {
  float phi = float(i) * 2.399963;  // golden angle
  float cosTheta = 1.0 - (float(i) + 0.5) / float(n);
  float sinTheta = sqrt(1.0 - cosTheta * cosTheta);

  vec3 dir = vec3(cos(phi) * sinTheta, cosTheta, sin(phi) * sinTheta);

  // Orient to normal
  vec3 up = abs(normal.y) < 0.999 ? vec3(0, 1, 0) : vec3(1, 0, 0);
  vec3 tangent = normalize(cross(up, normal));
  vec3 bitangent = cross(normal, tangent);

  return normalize(tangent * dir.x + normal * dir.y + bitangent * dir.z);
}
```

### The Path Tracer

```glsl
vec3 pathTrace(vec3 ro, vec3 rd, vec3 sunDir) {
  vec3 throughput = vec3(1.0);  // accumulated BRDF weights
  vec3 radiance = vec3(0.0);   // accumulated light

  for (int bounce = 0; bounce < 4; bounce++) {
    float t = raymarch(ro, rd);

    if (t < 0.0) {
      // Hit sky — add sky light contribution
      radiance += throughput * sky(rd, sunDir);
      break;
    }

    vec3 p = ro + rd * t;
    vec3 n = calcNormal(p);
    vec3 albedo = getMaterial(p);

    // Direct lighting (next event estimation)
    vec3 shadowRayDir = sunDir;
    float shadowT = raymarch(p + n * 0.01, shadowRayDir);
    if (shadowT < 0.0) {
      // Sun is visible
      float NdotL = max(dot(n, shadowRayDir), 0.0);
      radiance += throughput * albedo * NdotL * vec3(4.0);  // sun intensity
    }

    // Indirect lighting — bounce in random direction
    rd = cosineHemisphere(n);
    ro = p + n * 0.01;
    throughput *= albedo;  // Lambertian BRDF cancels the cos/pi from importance sampling

    // Russian Roulette (probabilistic termination after bounce 2)
    if (bounce > 1) {
      float p = max(throughput.r, max(throughput.g, throughput.b));
      if (randomFloat() > p) break;
      throughput /= p;  // compensate for terminated paths
    }
  }

  return radiance;
}
```

### Progressive Accumulation

Accumulate samples across frames for convergence:

```glsl
uniform sampler2D u_prevFrame;  // previous accumulated result
uniform int u_frame;

void main() {
  vec2 uv = gl_FragCoord.xy / u_resolution;
  initRNG(gl_FragCoord.xy, u_frame);

  // Jittered ray for anti-aliasing
  vec2 jitter = (randomVec2() - 0.5) / u_resolution;
  vec2 screenUV = (gl_FragCoord.xy + jitter - 0.5 * u_resolution) / u_resolution.y;

  vec3 ro = cameraPos;
  vec3 rd = normalize(vec3(screenUV, -1.5));
  // Apply camera rotation to rd...

  vec3 color = pathTrace(ro, rd, sunDir);

  // Blend with previous frames
  if (u_frame > 0) {
    vec3 prev = texture2D(u_prevFrame, uv).rgb;
    float weight = 1.0 / float(u_frame + 1);
    color = mix(prev, color, weight);
  }

  gl_FragColor = vec4(color, 1.0);
}
```

## PBR Material Model

### Cook-Torrance BRDF

For reflective/metallic surfaces, use the microfacet BRDF:

```glsl
// GGX/Trowbridge-Reitz normal distribution
float distributionGGX(vec3 N, vec3 H, float roughness) {
  float a = roughness * roughness;
  float a2 = a * a;
  float NdotH = max(dot(N, H), 0.0);
  float NdotH2 = NdotH * NdotH;
  float denom = NdotH2 * (a2 - 1.0) + 1.0;
  return a2 / (3.14159 * denom * denom);
}

// Smith's geometry function
float geometrySmith(float NdotV, float NdotL, float roughness) {
  float r = roughness + 1.0;
  float k = (r * r) / 8.0;
  float ggx1 = NdotV / (NdotV * (1.0 - k) + k);
  float ggx2 = NdotL / (NdotL * (1.0 - k) + k);
  return ggx1 * ggx2;
}

// Fresnel-Schlick
vec3 fresnelSchlick(float cosTheta, vec3 F0) {
  return F0 + (1.0 - F0) * pow(1.0 - cosTheta, 5.0);
}

// Full Cook-Torrance specular BRDF evaluation
vec3 cookTorrance(vec3 N, vec3 V, vec3 L, vec3 albedo, float metallic, float roughness) {
  vec3 H = normalize(V + L);
  float NdotV = max(dot(N, V), 0.001);
  float NdotL = max(dot(N, L), 0.001);
  float NdotH = max(dot(N, H), 0.0);
  float HdotV = max(dot(H, V), 0.0);

  // F0: reflectance at normal incidence
  vec3 F0 = mix(vec3(0.04), albedo, metallic);

  float D = distributionGGX(N, H, roughness);
  float G = geometrySmith(NdotV, NdotL, roughness);
  vec3 F = fresnelSchlick(HdotV, F0);

  // Specular
  vec3 specular = (D * G * F) / (4.0 * NdotV * NdotL + 0.001);

  // Diffuse (metals have no diffuse)
  vec3 kD = (1.0 - F) * (1.0 - metallic);
  vec3 diffuse = kD * albedo / 3.14159;

  return (diffuse + specular) * NdotL;
}
```

### GGX Importance Sampling

For efficient specular bounce directions in path tracing:

```glsl
vec3 importanceSampleGGX(vec3 N, float roughness) {
  float r1 = randomFloat();
  float r2 = randomFloat();

  float a = roughness * roughness;
  float phi = 6.28318 * r1;
  float cosTheta = sqrt((1.0 - r2) / (1.0 + (a * a - 1.0) * r2));
  float sinTheta = sqrt(1.0 - cosTheta * cosTheta);

  vec3 H = vec3(sinTheta * cos(phi), cosTheta, sinTheta * sin(phi));

  // Orient to normal
  vec3 up = abs(N.y) < 0.999 ? vec3(0, 1, 0) : vec3(1, 0, 0);
  vec3 tangent = normalize(cross(up, N));
  vec3 bitangent = cross(N, tangent);

  return normalize(tangent * H.x + N * H.y + bitangent * H.z);
}
```

## Denoising / Smoothing

Real-time path tracers need denoising to look good at low sample counts:

```glsl
// Simple temporal accumulation (already in progressive accumulation above)
// For more aggressive denoising, use bilateral filtering:

vec3 bilateralDenoise(sampler2D tex, vec2 uv, vec2 resolution) {
  vec3 center = texture2D(tex, uv).rgb;
  vec3 result = vec3(0.0);
  float totalWeight = 0.0;

  for (int y = -2; y <= 2; y++) {
    for (int x = -2; x <= 2; x++) {
      vec2 offset = vec2(x, y) / resolution;
      vec3 sample = texture2D(tex, uv + offset).rgb;

      float spatial = exp(-float(x*x + y*y) / 4.0);
      float range = exp(-dot(sample - center, sample - center) * 50.0);
      float weight = spatial * range;

      result += sample * weight;
      totalWeight += weight;
    }
  }
  return result / totalWeight;
}
```

## Performance Tips

- Start with 1-2 bounces for real-time; increase for stills
- Use Russian Roulette after bounce 2 to avoid wasting time on dim paths
- Importance sampling (cosine hemisphere for diffuse, GGX for specular) dramatically
  reduces variance
- Progressive accumulation: stop when the camera moves, restart from 0
- Lower resolution for complex scenes (the path tracer resolution doesn't need to match
  the display resolution)

## Key References

- **Peter Shirley** — "Ray Tracing in One Weekend" series
- **PBRT (Physically Based Rendering)** — Pharr, Jakob, Humphreys
- **Shadertoy path tracing examples** — search "path trace" for real-time implementations
- **Learn OpenGL: PBR** — Cook-Torrance BRDF tutorial
- **Disney Principled BRDF** — Brent Burley's PBR model paper
