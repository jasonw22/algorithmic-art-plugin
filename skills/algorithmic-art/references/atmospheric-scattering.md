# Atmospheric Scattering Reference

## Overview

Atmospheric scattering simulates how light interacts with air molecules and aerosol
particles, producing realistic skies, sunsets, god rays, and aerial perspective (distant
objects appear bluish and hazy). Two scattering types dominate:

- **Rayleigh scattering** — small molecules (N₂, O₂), wavelength-dependent (blue scatters
  more), creates blue skies and orange/red sunsets
- **Mie scattering** — larger particles (dust, water droplets), wavelength-independent,
  creates hazy glow around the sun and overcast skies

## Physics Summary

### Rayleigh Scattering

Scattering coefficient: proportional to `1/λ⁴` (shorter wavelengths scatter much more)

```glsl
// Rayleigh scattering coefficients for RGB
// Based on Earth atmosphere at sea level
const vec3 betaR = vec3(5.5e-6, 13.0e-6, 22.4e-6);  // per meter
```

Phase function (how much light scatters in each direction):

```glsl
float rayleighPhase(float cosTheta) {
  return 0.75 * (1.0 + cosTheta * cosTheta);
}
```

### Mie Scattering

Scattering coefficient: roughly wavelength-independent, controlled by turbidity.

```glsl
const vec3 betaM = vec3(21e-6);  // per meter (adjust for haze)
```

Phase function (strongly forward-scattering):

```glsl
// Henyey-Greenstein phase function
// g: anisotropy factor, 0.0 = isotropic, 0.99 = very forward-peaked
float miePhase(float cosTheta, float g) {
  float g2 = g * g;
  float num = (1.0 - g2);
  float denom = pow(1.0 + g2 - 2.0 * g * cosTheta, 1.5);
  return num / (4.0 * 3.14159 * denom);
}
```

### Beer-Lambert Attenuation

Light intensity decreases exponentially with optical depth:

```glsl
// transmittance = how much light survives through a medium
vec3 transmittance(vec3 beta, float dist) {
  return exp(-beta * dist);
}
```

## Simple Sky Shader

A minimal but effective sky model for procedural landscapes:

```glsl
vec3 sky(vec3 rd, vec3 sunDir) {
  // Sun position
  float sunDot = max(dot(rd, sunDir), 0.0);

  // Rayleigh: blue overhead, orange at horizon
  vec3 rayleigh = vec3(0.2, 0.4, 0.8) * (1.0 - 0.5 * rd.y);

  // Mie: sun glow
  float mie = pow(sunDot, 64.0) * 0.5 + pow(sunDot, 512.0) * 1.0;
  vec3 mieColor = vec3(1.0, 0.9, 0.7) * mie;

  // Gradient from horizon to zenith
  float horizon = pow(1.0 - max(rd.y, 0.0), 4.0);
  vec3 horizonColor = vec3(0.8, 0.6, 0.4) * horizon;

  // Sun disk
  float sunDisk = smoothstep(0.999, 0.9995, sunDot);
  vec3 sun = vec3(1.5, 1.3, 1.0) * sunDisk;

  return rayleigh + mieColor + horizonColor + sun;
}
```

## Full Physical Sky Model

Traces a ray through the atmosphere, accumulating scattering at each step:

```glsl
const float ATMOSPHERE_RADIUS = 6.42e6;  // meters
const float PLANET_RADIUS = 6.37e6;
const float SCALE_HEIGHT_R = 8000.0;    // Rayleigh scale height
const float SCALE_HEIGHT_M = 1200.0;    // Mie scale height
const vec3 BETA_R = vec3(5.5e-6, 13.0e-6, 22.4e-6);
const vec3 BETA_M = vec3(21e-6);
const float MIE_G = 0.76;              // Mie anisotropy
const int NUM_SAMPLES = 16;
const int NUM_LIGHT_SAMPLES = 8;

// Ray-sphere intersection (returns distances to entry and exit)
vec2 raySphere(vec3 ro, vec3 rd, float radius) {
  float b = dot(ro, rd);
  float c = dot(ro, ro) - radius * radius;
  float disc = b * b - c;
  if (disc < 0.0) return vec2(-1.0);
  float sq = sqrt(disc);
  return vec2(-b - sq, -b + sq);
}

vec3 atmosphere(vec3 rd, vec3 sunDir) {
  vec3 ro = vec3(0.0, PLANET_RADIUS + 1.0, 0.0);  // camera on surface

  vec2 atmoHit = raySphere(ro, rd, ATMOSPHERE_RADIUS);
  if (atmoHit.y < 0.0) return vec3(0.0);

  float stepLen = atmoHit.y / float(NUM_SAMPLES);
  vec3 totalR = vec3(0.0);
  vec3 totalM = vec3(0.0);
  float optDepthR = 0.0;
  float optDepthM = 0.0;

  for (int i = 0; i < NUM_SAMPLES; i++) {
    vec3 pos = ro + rd * (float(i) + 0.5) * stepLen;
    float height = length(pos) - PLANET_RADIUS;

    // Local density
    float densityR = exp(-height / SCALE_HEIGHT_R) * stepLen;
    float densityM = exp(-height / SCALE_HEIGHT_M) * stepLen;
    optDepthR += densityR;
    optDepthM += densityM;

    // Light ray (sun direction) optical depth
    vec2 sunHit = raySphere(pos, sunDir, ATMOSPHERE_RADIUS);
    float sunStepLen = sunHit.y / float(NUM_LIGHT_SAMPLES);
    float sunOptR = 0.0, sunOptM = 0.0;

    for (int j = 0; j < NUM_LIGHT_SAMPLES; j++) {
      vec3 sunPos = pos + sunDir * (float(j) + 0.5) * sunStepLen;
      float sunH = length(sunPos) - PLANET_RADIUS;
      sunOptR += exp(-sunH / SCALE_HEIGHT_R) * sunStepLen;
      sunOptM += exp(-sunH / SCALE_HEIGHT_M) * sunStepLen;
    }

    vec3 tau = BETA_R * (optDepthR + sunOptR) + BETA_M * 1.1 * (optDepthM + sunOptM);
    vec3 attenuation = exp(-tau);

    totalR += densityR * attenuation;
    totalM += densityM * attenuation;
  }

  float cosTheta = dot(rd, sunDir);
  float phaseR = rayleighPhase(cosTheta);
  float phaseM = miePhase(cosTheta, MIE_G);

  vec3 sunIntensity = vec3(20.0);
  return sunIntensity * (BETA_R * phaseR * totalR + BETA_M * phaseM * totalM);
}
```

## Aerial Perspective (Fog with Scattering)

Apply atmospheric effects to scene objects based on distance:

```glsl
vec3 applyAerialPerspective(vec3 objectColor, float dist, vec3 rd, vec3 sunDir) {
  // Extinction (objects fade with distance)
  vec3 extinction = exp(-(BETA_R + BETA_M) * dist);

  // In-scattering (atmosphere color added along view ray)
  float cosTheta = dot(rd, sunDir);
  vec3 inscatter = BETA_R * rayleighPhase(cosTheta) + BETA_M * miePhase(cosTheta, 0.76);
  inscatter *= (1.0 - extinction) * vec3(10.0);  // sun intensity scale

  return objectColor * extinction + inscatter;
}
```

## Sunset / Golden Hour

Adjust the sun direction to near-horizon for warm sunset tones. The physical model
handles this naturally — long path length through atmosphere increases Rayleigh scattering,
removing blue and leaving orange/red:

```glsl
// Sun near horizon = more dramatic scattering
vec3 sunDir = normalize(vec3(0.5, 0.05, 1.0));  // just above horizon

// For animated day/night cycle:
float dayPhase = u_time * 0.1;
vec3 sunDir = normalize(vec3(cos(dayPhase), sin(dayPhase), 0.3));
```

## God Rays (Volumetric Light Shafts)

Screen-space radial blur from the sun position:

```glsl
vec3 godRays(sampler2D sceneTex, vec2 uv, vec2 sunScreenPos, float intensity) {
  vec2 delta = (sunScreenPos - uv) * (1.0 / 64.0);
  vec3 accumulated = vec3(0.0);
  vec2 sampleUV = uv;
  float weight = 1.0;
  float decay = 0.96;

  for (int i = 0; i < 64; i++) {
    sampleUV += delta;
    vec3 sample = texture2D(sceneTex, sampleUV).rgb;
    accumulated += sample * weight;
    weight *= decay;
  }

  return accumulated * intensity / 64.0;
}
```

## Integration with Terrain/Landscape

Atmospheric scattering typically pairs with terrain rendering:

1. Render terrain via raymarching (see `terrain-rendering.md`)
2. Apply aerial perspective based on hit distance
3. If ray misses terrain, render sky via atmosphere function

```glsl
void main() {
  vec2 uv = (gl_FragCoord.xy - 0.5 * u_resolution) / u_resolution.y;
  vec3 ro = vec3(0, 2, 0);
  vec3 rd = normalize(vec3(uv, -1.5));
  vec3 sunDir = normalize(vec3(0.5, 0.3, 1.0));

  float t = raymarchTerrain(ro, rd);

  vec3 color;
  if (t > 0.0) {
    vec3 p = ro + rd * t;
    color = shadeTerrain(p, sunDir);
    color = applyAerialPerspective(color, t, rd, sunDir);
  } else {
    color = atmosphere(rd, sunDir);
  }

  color = acesToneMap(color);
  gl_FragColor = vec4(color, 1.0);
}
```

## Key References

- **Sébastien Hillaire** — "A Scalable and Production Ready Sky and Atmosphere Rendering Technique"
- **Nishita et al.** — "Display of the Earth Taking into Account Atmospheric Scattering"
- **Shadertoy "atmosphere"** — community implementations
- **Inigo Quilez** — aerial perspective and fog in raymarched scenes
