# GPU Particle Systems — Three.js GPGPU Reference

## Overview

CPU particle systems (updating positions in JavaScript, uploading each frame) hit a wall
around 50K–100K particles. GPU particle systems move the simulation entirely onto the GPU,
enabling millions of particles at 60fps. The key insight from Cinder's progressive particle
samples: start with CPU particles, then migrate to texture-based GPGPU or compute shaders
when you need scale.

In Three.js, the standard approach is **texture-based GPGPU** via `GPUComputationRenderer` —
particle state lives in float textures, fragment shaders run the physics, and a Points mesh
reads the result. This achieves the same effect as Cinder's transform feedback approach
through a different mechanism.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  GPUComputationRenderer (ping-pong float textures)  │
│                                                     │
│  Position Texture (RGBA float)                      │
│    R = x, G = y, B = z, A = life/age               │
│                                                     │
│  Velocity Texture (RGBA float)                      │
│    R = vx, G = vy, B = vz, A = mass/extra          │
│                                                     │
│  Simulation shaders read prev frame → write next    │
└──────────────────────┬──────────────────────────────┘
                       │ position texture sampled by
                       ▼
┌─────────────────────────────────────────────────────┐
│  THREE.Points with custom ShaderMaterial            │
│  Vertex shader: read position from texture by UV    │
│  Fragment shader: render particle appearance         │
└─────────────────────────────────────────────────────┘
```

## Three.js Scene Mode Implementation

### Setup with GPUComputationRenderer

```javascript
import { GPUComputationRenderer } from 'three/addons/misc/GPUComputationRenderer.js';

function sceneSetup(THREE, scene, camera, renderer, params, seed) {
  const WIDTH = params.texSize || 256;  // 256×256 = 65,536 particles
  const COUNT = WIDTH * WIDTH;

  // --- GPU Computation ---
  const gpuCompute = new GPUComputationRenderer(WIDTH, WIDTH, renderer);

  // Initialize position texture
  const posTex = gpuCompute.createTexture();
  const posData = posTex.image.data;
  for (let i = 0; i < COUNT; i++) {
    const i4 = i * 4;
    const theta = seededRandom() * Math.PI * 2;
    const phi = Math.acos(2 * seededRandom() - 1);
    const r = seededRandom() * 2;
    posData[i4 + 0] = r * Math.sin(phi) * Math.cos(theta);  // x
    posData[i4 + 1] = r * Math.sin(phi) * Math.sin(theta);  // y
    posData[i4 + 2] = r * Math.cos(phi);                     // z
    posData[i4 + 3] = seededRandom();                         // life (0–1)
  }

  // Initialize velocity texture
  const velTex = gpuCompute.createTexture();
  const velData = velTex.image.data;
  for (let i = 0; i < COUNT; i++) {
    const i4 = i * 4;
    velData[i4 + 0] = (seededRandom() - 0.5) * 0.01;
    velData[i4 + 1] = (seededRandom() - 0.5) * 0.01;
    velData[i4 + 2] = (seededRandom() - 0.5) * 0.01;
    velData[i4 + 3] = 1.0;  // mass
  }

  // Add computation variables (links texture name to shader)
  const posVar = gpuCompute.addVariable('texturePosition', positionShader, posTex);
  const velVar = gpuCompute.addVariable('textureVelocity', velocityShader, velTex);

  // Set dependencies (both shaders can read both textures)
  gpuCompute.setVariableDependencies(posVar, [posVar, velVar]);
  gpuCompute.setVariableDependencies(velVar, [posVar, velVar]);

  // Set uniforms accessible from simulation shaders
  posVar.material.uniforms.u_time = { value: 0 };
  posVar.material.uniforms.u_dt = { value: 0.016 };
  velVar.material.uniforms.u_time = { value: 0 };
  velVar.material.uniforms.u_dt = { value: 0.016 };
  velVar.material.uniforms.u_attractorPos = { value: new THREE.Vector3(0, 0, 0) };
  velVar.material.uniforms.u_noiseScale = { value: params.noiseScale || 1.5 };
  velVar.material.uniforms.u_noiseStrength = { value: params.noiseStrength || 0.5 };

  gpuCompute.init();

  // --- Renderable Points ---
  // Create UV references so vertex shader can look up particle position from texture
  const geometry = new THREE.BufferGeometry();
  const uvs = new Float32Array(COUNT * 2);
  for (let i = 0; i < COUNT; i++) {
    const x = (i % WIDTH) / WIDTH;
    const y = Math.floor(i / WIDTH) / WIDTH;
    uvs[i * 2] = x;
    uvs[i * 2 + 1] = y;
  }
  geometry.setAttribute('reference', new THREE.BufferAttribute(uvs, 2));

  // Dummy position attribute (actual positions come from texture)
  geometry.setAttribute('position', new THREE.BufferAttribute(new Float32Array(COUNT * 3), 3));

  const material = new THREE.ShaderMaterial({
    uniforms: {
      texturePosition: { value: null },  // set each frame
      textureVelocity: { value: null },
      u_pointSize: { value: params.pointSize || 1.5 },
      u_color1: { value: new THREE.Color(params.color1 || '#4488ff') },
      u_color2: { value: new THREE.Color(params.color2 || '#ff4488') },
    },
    vertexShader: renderVertexShader,
    fragmentShader: renderFragmentShader,
    transparent: true,
    blending: THREE.AdditiveBlending,
    depthWrite: false,
  });

  const points = new THREE.Points(geometry, material);
  points.frustumCulled = false;  // particles span the whole scene
  scene.add(points);

  camera.position.set(0, 0, 5);
  scene.background = new THREE.Color(0x000000);

  return { gpuCompute, posVar, velVar, points };
}
```

### Simulation Shaders

#### Position Update Shader
```glsl
// positionShader — runs per texel (per particle)
uniform float u_time;
uniform float u_dt;

void main() {
  vec2 uv = gl_FragCoord.xy / resolution.xy;
  vec4 pos = texture2D(texturePosition, uv);
  vec4 vel = texture2D(textureVelocity, uv);

  // Euler integration
  pos.xyz += vel.xyz * u_dt * 60.0;

  // Age the particle
  pos.w -= u_dt * 0.1;

  // Respawn dead particles at origin with jitter
  if (pos.w <= 0.0) {
    pos.xyz = vec3(0.0);
    pos.w = 1.0;
  }

  gl_FragColor = pos;
}
```

#### Velocity Update Shader
```glsl
// velocityShader — runs per texel (per particle)
uniform float u_time;
uniform float u_dt;
uniform vec3 u_attractorPos;
uniform float u_noiseScale;
uniform float u_noiseStrength;

// Include a 3D noise function (simplex or Perlin) here
// See shaders-glsl.md for noise implementations

void main() {
  vec2 uv = gl_FragCoord.xy / resolution.xy;
  vec4 pos = texture2D(texturePosition, uv);
  vec4 vel = texture2D(textureVelocity, uv);

  // Attractor force (pull toward point)
  vec3 toAttractor = u_attractorPos - pos.xyz;
  float dist = length(toAttractor);
  vec3 attractForce = normalize(toAttractor) / (dist * dist + 0.1) * 0.5;

  // Noise-based force field (curl noise for divergence-free flow)
  float eps = 0.01;
  vec3 p = pos.xyz * u_noiseScale;
  // Approximate curl via finite differences of 3D noise
  vec3 curlForce = vec3(
    snoise(p + vec3(0, eps, 0)) - snoise(p - vec3(0, eps, 0))
     - (snoise(p + vec3(0, 0, eps)) - snoise(p - vec3(0, 0, eps))),
    snoise(p + vec3(0, 0, eps)) - snoise(p - vec3(0, 0, eps))
     - (snoise(p + vec3(eps, 0, 0)) - snoise(p - vec3(eps, 0, 0))),
    snoise(p + vec3(eps, 0, 0)) - snoise(p - vec3(eps, 0, 0))
     - (snoise(p + vec3(0, eps, 0)) - snoise(p - vec3(0, eps, 0)))
  ) / (2.0 * eps) * u_noiseStrength;

  // Damping
  vel.xyz *= 0.98;

  // Accumulate forces
  vel.xyz += (attractForce + curlForce) * u_dt;

  // Speed limit
  float speed = length(vel.xyz);
  if (speed > 1.0) vel.xyz = vel.xyz / speed * 1.0;

  gl_FragColor = vel;
}
```

### Render Shaders

```glsl
// renderVertexShader — reads position from GPU computation texture
uniform sampler2D texturePosition;
uniform sampler2D textureVelocity;
uniform float u_pointSize;

attribute vec2 reference;  // UV into the computation texture
varying float vLife;
varying float vSpeed;

void main() {
  vec4 pos = texture2D(texturePosition, reference);
  vec4 vel = texture2D(textureVelocity, reference);

  vLife = pos.w;
  vSpeed = length(vel.xyz);

  vec4 mvPos = modelViewMatrix * vec4(pos.xyz, 1.0);
  gl_Position = projectionMatrix * mvPos;

  // Size attenuation (smaller when far)
  gl_PointSize = u_pointSize * (300.0 / -mvPos.z);
}
```

```glsl
// renderFragmentShader
uniform vec3 u_color1;
uniform vec3 u_color2;

varying float vLife;
varying float vSpeed;

void main() {
  // Soft circle
  float d = length(gl_PointCoord - 0.5) * 2.0;
  if (d > 1.0) discard;
  float alpha = smoothstep(1.0, 0.3, d) * vLife;

  // Color by speed
  vec3 color = mix(u_color1, u_color2, clamp(vSpeed * 5.0, 0.0, 1.0));

  gl_FragColor = vec4(color, alpha);
}
```

### Animation Loop

```javascript
function sceneAnimate(THREE, scene, camera, state, params, time, delta) {
  const { gpuCompute, posVar, velVar, points } = state;

  // Update simulation uniforms
  posVar.material.uniforms.u_time.value = time;
  posVar.material.uniforms.u_dt.value = Math.min(delta, 0.05);
  velVar.material.uniforms.u_time.value = time;
  velVar.material.uniforms.u_dt.value = Math.min(delta, 0.05);

  // Step the simulation
  gpuCompute.compute();

  // Feed the result textures to the render material
  points.material.uniforms.texturePosition.value =
    gpuCompute.getCurrentRenderTarget(posVar).texture;
  points.material.uniforms.textureVelocity.value =
    gpuCompute.getCurrentRenderTarget(velVar).texture;
}
```

## Integration Schemes

### Euler (simplest, what's shown above)
```glsl
pos += vel * dt;
vel += force * dt;
```

### Verlet (more stable, better energy conservation)
Cinder's particle sphere samples use Verlet integration — store previous position instead
of velocity:
```glsl
vec3 prevPos = texture2D(texturePrevPosition, uv).xyz;
vec3 curPos = texture2D(texturePosition, uv).xyz;
vec3 newPos = 2.0 * curPos - prevPos + force * dt * dt;
// Write curPos to prevPosition texture, newPos to position texture
```
Verlet is excellent for cloth simulation and constraint-based systems where you need
positional stability.

### Runge-Kutta 4 (most accurate, 4× cost)
Evaluate force at 4 points per step. Overkill for most generative art — use when simulating
real physics (orbital mechanics, fluid) where drift matters.

## Common Force Patterns for Generative Art

### Noise Flow Field
Sample 3D noise at particle position → use as velocity or force. Animate by scrolling
the noise z-coordinate with time. See the curl noise approach in the velocity shader above.

### Point Attractors / Repulsors
```glsl
vec3 dir = attractor - pos;
float d = length(dir);
vec3 force = normalize(dir) * strength / (d * d + softening);
```
Use `softening` (0.01–0.5) to prevent infinite force at zero distance.

### Sinusoidal Attractors (from Cinder's NVidiaComputeParticles)
```glsl
vec3 attractor = vec3(sin(time * 0.5) * 3.0, cos(time * 0.7) * 2.0, sin(time * 0.3) * 3.0);
```
Multiple oscillating attractors with different frequencies create complex orbital patterns.

### Spherical Containment
```glsl
float d = length(pos);
if (d > radius) vel += -normalize(pos) * (d - radius) * bounceStrength;
```

## Texture Size Guide

| Texture Size | Particle Count | Typical Use |
|---|---|---|
| 64 × 64 | 4,096 | Subtle accent, low-end devices |
| 128 × 128 | 16,384 | Medium density, mobile-safe |
| 256 × 256 | 65,536 | Dense fields, desktop |
| 512 × 512 | 262,144 | Very dense, needs decent GPU |
| 1024 × 1024 | 1,048,576 | Extreme density, high-end GPU only |

Always expose texture size as a parameter with a sensible default (256).

## Tips

- **AdditiveBlending + dark background** is the classic generative particle look — particles
  glow brighter where they overlap
- **depthWrite: false** prevents z-fighting artifacts in particle clouds
- **frustumCulled: false** on the Points mesh — GPU particles can be anywhere, and Three.js
  can't know the bounding box from the texture
- **Multiple species**: use separate GPUComputationRenderer instances or pack species ID into
  the alpha channel to create predator/prey or multi-flock systems
- **Trail rendering**: render to an FBO with no clear (or alpha-blended fade), then composite
  onto the screen. Combines with the multipass-buffers.md technique
- **3D noise textures**: pre-bake a 3D noise volume into a `THREE.Data3DTexture` and sample
  it in the velocity shader — cheaper than computing noise per-particle per-frame
