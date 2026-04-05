# Superformula — Parametric Shape Reference

## Overview

The superformula (Johan Gielis, 2003) is a single parametric equation that generalizes the
circle into an enormous family of natural-looking shapes: starfish, flowers, leaves, crystals,
gears, and everything in between. It's one of the highest return-on-investment equations in
generative art — six parameters produce infinite organic variety.

Cinder's `SuperformulaGPU` sample demonstrates GPU-accelerated superformula meshes with
animated parameter interpolation. The math is pure and portable to any platform.

## The Equation

The superformula defines a radius as a function of angle θ:

```
r(θ) = ( |cos(m₁·θ/4) / a|^n₂ + |sin(m₂·θ/4) / b|^n₃ )^(-1/n₁)
```

Parameters:
- **m₁, m₂** — symmetry order (integer-ish values = rotational symmetry count)
- **a, b** — scaling (usually both 1.0)
- **n₁** — master shape exponent (controls overall inflation/deflation)
- **n₂, n₃** — detail exponents (control concavity of lobes)

When m₁ = m₂ = m, a = b = 1 (the common case), it simplifies to:

```
r(θ) = ( |cos(m·θ/4)|^n₂ + |sin(m·θ/4)|^n₃ )^(-1/n₁)
```

## Shape Presets

| Name | m | n₁ | n₂ | n₃ | Character |
|------|---|-----|-----|-----|-----------|
| Circle | 4 | 2 | 2 | 2 | Perfect circle |
| Square | 4 | 100 | 100 | 100 | Rounded square (high n → sharp corners) |
| Triangle | 3 | 100 | 100 | 100 | Rounded triangle |
| Starfish | 5 | 0.3 | 0.3 | 0.3 | Five-armed starfish |
| Flower | 6 | 1 | 1 | 1 | Six-petaled flower |
| Asteroid | 4 | 0.5 | 0.5 | 0.5 | Four-pointed star with concave sides |
| Clover | 3 | 0.5 | 0.5 | 0.5 | Three-lobed clover |
| Gear | 12 | 10 | 10 | 10 | Gear-like with 12 teeth |
| Petal | 1 | 0.5 | 1.5 | 0.5 | Asymmetric leaf/petal |
| Blob | 7 | 0.2 | 1.7 | 1.7 | Organic amoeba-like |
| Spike | 5 | 0.1 | 1.7 | 1.7 | Spiky radial form |
| Cushion | 4 | 2 | 0.5 | 0.5 | Puffy rounded square |

## 2D Implementation (p5.js / Canvas)

```javascript
function superformula(theta, m, n1, n2, n3, a = 1, b = 1) {
  const t1 = Math.abs(Math.cos(m * theta / 4) / a);
  const t2 = Math.abs(Math.sin(m * theta / 4) / b);
  const r = Math.pow(Math.pow(t1, n2) + Math.pow(t2, n3), -1 / n1);
  return r;
}

// Draw a superformula shape
function drawSuperformula(cx, cy, scale, m, n1, n2, n3, steps = 360) {
  p.beginShape();
  for (let i = 0; i <= steps; i++) {
    const theta = (i / steps) * Math.PI * 2;
    const r = superformula(theta, m, n1, n2, n3) * scale;
    const x = cx + r * Math.cos(theta);
    const y = cy + r * Math.sin(theta);
    p.vertex(x, y);
  }
  p.endShape(p.CLOSE);
}
```

## 3D Supershape (spherical product)

To create 3D supershapes, evaluate two superformulas at latitude (φ) and longitude (θ),
then combine on a sphere:

```javascript
function supershape3D(theta, phi, m1, n11, n12, n13, m2, n21, n22, n23) {
  const r1 = superformula(theta, m1, n11, n12, n13);
  const r2 = superformula(phi, m2, n21, n22, n23);

  const x = r1 * Math.cos(theta) * r2 * Math.cos(phi);
  const y = r1 * Math.sin(theta) * r2 * Math.cos(phi);
  const z = r2 * Math.sin(phi);

  return { x, y, z };
}

// Generate mesh vertices
function buildSupershapeMesh(res, params) {
  const vertices = [];
  for (let i = 0; i <= res; i++) {
    const phi = (i / res) * Math.PI - Math.PI / 2;  // -π/2 to π/2
    for (let j = 0; j <= res; j++) {
      const theta = (j / res) * Math.PI * 2;         // 0 to 2π
      const p = supershape3D(theta, phi,
        params.m1, params.n11, params.n12, params.n13,
        params.m2, params.n21, params.n22, params.n23
      );
      vertices.push(p.x, p.y, p.z);
    }
  }
  return vertices;
}
```

### Three.js Scene Mode

```javascript
function createSupershapeGeometry(THREE, res, params) {
  const geo = new THREE.BufferGeometry();
  const positions = [];
  const normals = [];
  const indices = [];

  // Generate grid of vertices
  for (let i = 0; i <= res; i++) {
    const phi = (i / res) * Math.PI - Math.PI / 2;
    for (let j = 0; j <= res; j++) {
      const theta = (j / res) * Math.PI * 2;
      const p = supershape3D(theta, phi, /* params */);
      positions.push(p.x, p.y, p.z);
    }
  }

  // Generate triangle indices
  for (let i = 0; i < res; i++) {
    for (let j = 0; j < res; j++) {
      const a = i * (res + 1) + j;
      const b = a + 1;
      const c = a + (res + 1);
      const d = c + 1;
      indices.push(a, c, b);
      indices.push(b, c, d);
    }
  }

  geo.setAttribute('position', new THREE.Float32BufferAttribute(positions, 3));
  geo.setIndex(indices);
  geo.computeVertexNormals();
  return geo;
}
```

## GLSL Implementation (shader mode)

```glsl
float superformula(float theta, float m, float n1, float n2, float n3) {
  float t1 = abs(cos(m * theta / 4.0));
  float t2 = abs(sin(m * theta / 4.0));
  return pow(pow(t1, n2) + pow(t2, n3), -1.0 / n1);
}

// In a vertex shader or raymarching SDF:
vec3 supershape(float theta, float phi, float m1, float n11, float n12, float n13,
                float m2, float n21, float n22, float n23) {
  float r1 = superformula(theta, m1, n11, n12, n13);
  float r2 = superformula(phi, m2, n21, n22, n23);
  return vec3(
    r1 * cos(theta) * r2 * cos(phi),
    r1 * sin(theta) * r2 * cos(phi),
    r2 * sin(phi)
  );
}
```

## Animation Techniques

### Parameter Morphing
Interpolate between preset parameter sets for smooth shape transitions:
```javascript
function lerpParams(presetA, presetB, t) {
  const e = easeInOutCubic(t);  // smooth the interpolation
  return {
    m:  presetA.m  * (1 - e) + presetB.m  * e,
    n1: presetA.n1 * (1 - e) + presetB.n1 * e,
    n2: presetA.n2 * (1 - e) + presetB.n2 * e,
    n3: presetA.n3 * (1 - e) + presetB.n3 * e,
  };
}
```
Cycle through presets on a timer for hypnotic morphing loops.

### Noise-Modulated Parameters
```javascript
const m = baseM + noise(time * 0.1) * 2;
const n1 = baseN1 + noise(time * 0.15 + 100) * 0.5;
```
Produces organic breathing/pulsing shapes.

### Radial Noise Displacement
Apply noise displacement to the radius after computing the superformula:
```javascript
const r = superformula(theta, m, n1, n2, n3);
const displaced = r * (1 + noise(theta * 3, time) * 0.2);
```

## Compositional Patterns

- **Nested supershapes**: draw multiple at decreasing scale with different parameters
- **Grid of variations**: tile the canvas with supershapes, varying one parameter per cell
- **Particle-filled**: use the superformula boundary as a containment shell for particles
- **Extruded profiles**: use 2D superformula cross-sections along a 3D spine (like Cinder's
  Extrude sample but with parametric profiles instead of font glyphs)
- **Boolean combinations**: intersect/union two 3D supershapes via SDF operations

## Parameter Design

Expose these as named presets with a dropdown, plus individual sliders for exploration:
- **m**: integer range 1–12 (but allow float for transitions)
- **n1**: range 0.1–20 (most interesting between 0.1–5)
- **n2, n3**: range 0.1–20
- **resolution**: 60–360 steps for 2D, 30–100 subdivisions for 3D
- **scale**: overall size multiplier
