# Terrain Rendering Reference

## Overview

Procedural terrain in shaders generates landscapes entirely from noise functions —
no mesh or heightmap textures needed. The terrain is raymarched against a height function,
then shaded with materials, lighting, and atmospheric effects.

## Height Functions

### Basic FBM Terrain

```glsl
float terrainHeight(vec2 p) {
  float h = 0.0;
  float amp = 1.0;
  float freq = 0.005;  // controls scale of terrain features
  mat2 rot = mat2(0.8, 0.6, -0.6, 0.8);

  for (int i = 0; i < 8; i++) {
    h += amp * snoise(p * freq);
    freq *= 2.0;
    amp *= 0.5;
    p = rot * p;  // rotate each octave to reduce axis-aligned artifacts
  }
  return h * 50.0;  // scale to world units
}
```

### Ridged Noise (Mountains)

Creates sharp ridges by taking the absolute value of noise and inverting:

```glsl
float ridgedNoise(vec2 p, int octaves) {
  float h = 0.0;
  float amp = 1.0;
  float freq = 1.0;
  float prev = 1.0;

  for (int i = 0; i < octaves; i++) {
    float n = abs(snoise(p * freq));
    n = 1.0 - n;          // invert: valleys become ridges
    n = n * n;             // sharpen
    n *= prev;             // scale by previous octave (erosion-like)
    h += n * amp;
    prev = n;
    freq *= 2.0;
    amp *= 0.5;
  }
  return h;
}
```

### Terraced / Stepped Terrain

```glsl
float terraced(float h, float steps) {
  return floor(h * steps) / steps;
}

// Smooth terracing
float smoothTerraced(float h, float steps, float smoothness) {
  float st = h * steps;
  float f = fract(st);
  f = smoothstep(0.0, smoothness, f) * smoothstep(1.0, 1.0 - smoothness, f);
  return (floor(st) + f) / steps;
}
```

### Erosion-Like Detail

Add high-frequency detail that follows the slope (domain warping by gradient):

```glsl
float erodedTerrain(vec2 p) {
  float h = terrainHeight(p);

  // Compute gradient
  float eps = 0.5;
  float hx = terrainHeight(p + vec2(eps, 0.0));
  float hz = terrainHeight(p + vec2(0.0, eps));
  vec2 gradient = vec2(hx - h, hz - h) / eps;

  // Add detail warped by slope direction
  float detail = snoise((p + gradient * 10.0) * 0.1) * 2.0;
  h += detail * (1.0 - smoothstep(0.0, 2.0, length(gradient)));

  return h;
}
```

## Terrain Raymarching

Standard sphere-tracing doesn't work for heightfields (they're not SDFs). Instead,
use a stepped raymarch with adaptive step size:

```glsl
float raymarchTerrain(vec3 ro, vec3 rd) {
  float t = 0.1;
  float lastH = 0.0;
  float lastY = 0.0;

  for (int i = 0; i < 200; i++) {
    vec3 p = ro + rd * t;
    float h = terrainHeight(p.xz);

    if (p.y < h) {
      // Linear interpolation for sub-step accuracy
      float tPrev = t - (t - lastH) * (lastY - terrainHeight((ro + rd * (t - 0.5)).xz))
                    / (p.y - h - lastY + lastH);
      return mix(t - 1.0, t, (lastY - lastH) / (lastY - lastH + h - p.y));
    }

    lastH = h;
    lastY = p.y;

    // Adaptive step: larger steps when far from surface
    t += max(0.1, (p.y - h) * 0.3);

    if (t > 1000.0) break;
  }
  return -1.0;
}

// Simpler (and more robust) version:
float raymarchTerrainSimple(vec3 ro, vec3 rd, float maxDist) {
  float t = 0.0;
  for (int i = 0; i < 300; i++) {
    vec3 p = ro + rd * t;
    float h = terrainHeight(p.xz);
    float delta = p.y - h;

    if (delta < 0.01 * t) return t;  // scale threshold with distance

    t += delta * 0.4;  // conservative step (0.3–0.5)
    if (t > maxDist) break;
  }
  return -1.0;
}
```

## Normal Calculation

Compute terrain normals via central differences on the height function:

```glsl
vec3 terrainNormal(vec2 p) {
  float eps = 0.5;  // larger epsilon = smoother normals
  float hL = terrainHeight(p - vec2(eps, 0));
  float hR = terrainHeight(p + vec2(eps, 0));
  float hD = terrainHeight(p - vec2(0, eps));
  float hU = terrainHeight(p + vec2(0, eps));
  return normalize(vec3(hL - hR, 2.0 * eps, hD - hU));
}
```

## Material / Biome Assignment

Assign materials based on height, slope, and noise:

```glsl
vec3 terrainMaterial(vec3 p, vec3 normal) {
  float height = p.y;
  float slope = 1.0 - normal.y;  // 0 = flat, 1 = vertical

  // Snow (high altitude, flat areas)
  vec3 snow = vec3(0.95, 0.95, 0.97);
  float snowMask = smoothstep(30.0, 40.0, height) * smoothstep(0.5, 0.3, slope);

  // Rock (steep slopes)
  vec3 rock = vec3(0.35, 0.3, 0.25);
  float rockMask = smoothstep(0.3, 0.6, slope);

  // Grass (low altitude, flat)
  vec3 grass = vec3(0.15, 0.3, 0.1);

  // Sand (very low altitude)
  vec3 sand = vec3(0.6, 0.5, 0.3);
  float sandMask = smoothstep(2.0, 0.0, height);

  // Blend materials
  vec3 color = grass;
  color = mix(color, sand, sandMask);
  color = mix(color, rock, rockMask);
  color = mix(color, snow, snowMask);

  // Add noise variation
  color *= 0.85 + 0.3 * snoise(p.xz * 0.5);

  return color;
}
```

## Shadows on Terrain

Trace a shadow ray from the surface point toward the sun along the heightfield:

```glsl
float terrainShadow(vec3 p, vec3 sunDir) {
  float t = 0.5;
  float shadow = 1.0;

  for (int i = 0; i < 64; i++) {
    vec3 sp = p + sunDir * t;
    float h = terrainHeight(sp.xz);
    float delta = sp.y - h;

    if (delta < 0.0) return 0.0;  // in shadow

    // Soft shadow
    shadow = min(shadow, 8.0 * delta / t);

    t += max(0.5, delta * 0.5);
    if (t > 200.0) break;
  }
  return clamp(shadow, 0.0, 1.0);
}
```

## LOD (Level of Detail)

Reduce noise octaves with distance to maintain performance:

```glsl
float terrainLOD(vec2 p, float dist) {
  int octaves = int(mix(8.0, 2.0, clamp(dist / 500.0, 0.0, 1.0)));
  // Use fewer octaves for distant terrain
  float h = 0.0;
  float amp = 1.0;
  float freq = 0.005;
  for (int i = 0; i < octaves; i++) {
    h += amp * snoise(p * freq);
    freq *= 2.0;
    amp *= 0.5;
  }
  return h * 50.0;
}
```

## Complete Terrain Scene

```glsl
void main() {
  vec2 uv = (gl_FragCoord.xy - 0.5 * u_resolution) / u_resolution.y;

  // Camera
  vec3 ro = vec3(u_time * 5.0, 60.0, u_time * 3.0);  // fly forward
  vec3 target = ro + vec3(10.0, -5.0, 10.0);
  vec3 forward = normalize(target - ro);
  vec3 right = normalize(cross(vec3(0, 1, 0), forward));
  vec3 up = cross(forward, right);
  vec3 rd = normalize(uv.x * right + uv.y * up + 1.5 * forward);

  vec3 sunDir = normalize(vec3(0.5, 0.35, 0.8));

  float t = raymarchTerrainSimple(ro, rd, 800.0);
  vec3 color;

  if (t > 0.0) {
    vec3 p = ro + rd * t;
    vec3 n = terrainNormal(p.xz);

    // Material
    vec3 albedo = terrainMaterial(p, n);

    // Lighting
    float diff = max(dot(n, sunDir), 0.0);
    float shadow = terrainShadow(p + n * 0.1, sunDir);
    float ao = 0.5 + 0.5 * n.y;  // cheap AO from normal

    vec3 ambient = vec3(0.15, 0.2, 0.3) * ao;
    color = albedo * (ambient + vec3(1.0, 0.9, 0.7) * diff * shadow);

    // Aerial perspective (see atmospheric-scattering.md)
    float fogAmount = 1.0 - exp(-t * 0.002);
    vec3 fogColor = vec3(0.5, 0.6, 0.7);
    color = mix(color, fogColor, fogAmount);
  } else {
    // Sky
    color = sky(rd, sunDir);
  }

  color = acesToneMap(color);
  gl_FragColor = vec4(color, 1.0);
}
```

## Performance Tips

- Keep raymarch step count under 300 for terrain
- Use adaptive step size (step proportional to height above terrain)
- Reduce FBM octaves for the raymarching pass; use full octaves only for final shading
- Pre-compute normals at lower resolution if needed
- LOD: fewer octaves at greater distance

## Key References

- **Inigo Quilez** — "Elevated" and "Rainforest" Shadertoy demos
- **GPU Gems 3, Ch. 1** — "Generating Complex Procedural Terrains Using the GPU"
- **Sebastian Lague** — Procedural terrain generation tutorials
- **Red Blob Games** — terrain generation concepts
