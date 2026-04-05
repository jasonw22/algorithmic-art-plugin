# Multipass / Buffer Ping-Pong — GPU Simulation Reference

## Overview

Multipass rendering stores state in textures (framebuffers) and reads the previous frame's
output as input for the next. This feedback loop is the foundation for all GPU-based
simulations: fluid dynamics, cellular automata, reaction-diffusion, particle systems,
and any effect that evolves over time.

In our Three.js Shader template, multipass is implemented via **ping-pong buffers** — two
render targets that alternate roles each frame (one is read, the other is written to, then
they swap).

## Architecture

```
Frame N:
  Read from Buffer A → Run simulation shader → Write to Buffer B → Display Buffer B

Frame N+1:
  Read from Buffer B → Run simulation shader → Write to Buffer A → Display Buffer A
```

This avoids read-write hazards (you never read and write the same texture simultaneously).

## Three.js Implementation

### Setting Up Ping-Pong Buffers

```javascript
function shaderUniforms(params, seed) {
  // Create two render targets for ping-pong
  const size = Math.min(window.innerWidth, window.innerHeight);
  const options = {
    minFilter: THREE.NearestFilter,
    magFilter: THREE.NearestFilter,
    format: THREE.RGBAFormat,
    type: THREE.FloatType  // 32-bit float per channel — critical for simulation accuracy
  };
  const rtA = new THREE.WebGLRenderTarget(size, size, options);
  const rtB = new THREE.WebGLRenderTarget(size, size, options);

  return {
    u_buffer: { value: rtA.texture },
    u_resolution: { value: new THREE.Vector2(size, size) },
    u_frame: { value: 0 },
    u_feedRate: { value: params.feedRate || 0.055 },
    u_killRate: { value: params.killRate || 0.062 },
    // Store render targets as custom properties for access in shaderAnimate
    _rtA: rtA,
    _rtB: rtB,
    _ping: true,
  };
}
```

### Running the Simulation Step

```javascript
function shaderAnimate(uniforms, params, time, renderer, scene, camera) {
  // The template passes renderer, scene, camera as extra args when available

  const readRT = uniforms._ping ? uniforms._rtA : uniforms._rtB;
  const writeRT = uniforms._ping ? uniforms._rtB : uniforms._rtA;

  // Point the shader at the read buffer
  uniforms.u_buffer.value = readRT.texture;

  // Render simulation step to the write buffer
  renderer.setRenderTarget(writeRT);
  renderer.render(scene, camera);
  renderer.setRenderTarget(null);

  // Now point the display pass at the write buffer (latest state)
  uniforms.u_buffer.value = writeRT.texture;

  // Swap
  uniforms._ping = !uniforms._ping;
  uniforms.u_frame.value++;
}
```

### Initialization (Seeding the Simulation)

Most simulations need initial conditions written into the buffer. Use `u_frame == 0`
in the shader to detect the first frame:

```glsl
void main() {
  vec2 uv = gl_FragCoord.xy / u_resolution;

  if (u_frame == 0) {
    // Initial conditions: e.g., seed spots for reaction-diffusion
    float d = length(uv - 0.5);
    float seed = step(d, 0.05);  // circle of chemical B in center
    gl_FragColor = vec4(1.0 - seed, seed, 0.0, 1.0);
    return;
  }

  // Read previous state
  vec4 state = texture2D(u_buffer, uv);
  // ... evolve state ...
  gl_FragColor = vec4(newState, 1.0);
}
```

## Common Simulation Patterns

### Laplacian (Neighbor Sampling)

Most PDE simulations need the Laplacian (sum of neighbors minus center). Use a 9-point
stencil for isotropy:

```glsl
vec4 laplacian(sampler2D tex, vec2 uv, vec2 texel) {
  vec4 sum = vec4(0.0);
  // 9-point stencil weights (more isotropic than 5-point)
  sum += texture2D(tex, uv + vec2(-1, -1) * texel) * 0.05;
  sum += texture2D(tex, uv + vec2( 0, -1) * texel) * 0.2;
  sum += texture2D(tex, uv + vec2( 1, -1) * texel) * 0.05;
  sum += texture2D(tex, uv + vec2(-1,  0) * texel) * 0.2;
  sum += texture2D(tex, uv                        ) * -1.0;
  sum += texture2D(tex, uv + vec2( 1,  0) * texel) * 0.2;
  sum += texture2D(tex, uv + vec2(-1,  1) * texel) * 0.05;
  sum += texture2D(tex, uv + vec2( 0,  1) * texel) * 0.2;
  sum += texture2D(tex, uv + vec2( 1,  1) * texel) * 0.05;
  return sum;
}
```

### Gray-Scott Reaction-Diffusion (GPU)

```glsl
uniform sampler2D u_buffer;
uniform vec2 u_resolution;
uniform float u_feedRate;  // f: 0.01–0.08
uniform float u_killRate;  // k: 0.045–0.07
uniform int u_frame;

void main() {
  vec2 uv = gl_FragCoord.xy / u_resolution;
  vec2 texel = 1.0 / u_resolution;

  if (u_frame == 0) {
    // Seed: A=1 everywhere, B=1 in random spots
    float d1 = length(uv - vec2(0.4, 0.5));
    float d2 = length(uv - vec2(0.6, 0.5));
    float b = step(d1, 0.04) + step(d2, 0.03);
    gl_FragColor = vec4(1.0 - b, b, 0.0, 1.0);
    return;
  }

  vec4 state = texture2D(u_buffer, uv);
  float a = state.r;
  float b = state.g;

  vec4 lap = laplacian(u_buffer, uv, texel);
  float lapA = lap.r;
  float lapB = lap.g;

  float Da = 1.0;    // diffusion rate of A
  float Db = 0.5;    // diffusion rate of B
  float dt = 1.0;

  float abb = a * b * b;
  float newA = a + (Da * lapA - abb + u_feedRate * (1.0 - a)) * dt;
  float newB = b + (Db * lapB + abb - (u_killRate + u_feedRate) * b) * dt;

  gl_FragColor = vec4(clamp(newA, 0.0, 1.0), clamp(newB, 0.0, 1.0), 0.0, 1.0);
}
```

### Cellular Automata (GPU)

```glsl
// Conway's Game of Life in a shader
void main() {
  vec2 uv = gl_FragCoord.xy / u_resolution;
  vec2 texel = 1.0 / u_resolution;

  if (u_frame == 0) {
    // Random initial state from hash
    float h = fract(sin(dot(gl_FragCoord.xy, vec2(12.9898, 78.233))) * 43758.5453);
    gl_FragColor = vec4(step(0.5, h), 0.0, 0.0, 1.0);
    return;
  }

  float cell = texture2D(u_buffer, uv).r;

  // Count 8 neighbors
  float neighbors = 0.0;
  for (int y = -1; y <= 1; y++) {
    for (int x = -1; x <= 1; x++) {
      if (x == 0 && y == 0) continue;
      neighbors += texture2D(u_buffer, uv + vec2(x, y) * texel).r;
    }
  }

  // B3/S23 rule
  float alive = 0.0;
  if (cell > 0.5) {
    alive = (neighbors >= 2.0 && neighbors <= 3.0) ? 1.0 : 0.0;
  } else {
    alive = (neighbors >= 2.5 && neighbors <= 3.5) ? 1.0 : 0.0;
  }

  gl_FragColor = vec4(alive, cell * 0.95, 0.0, 1.0);  // .g = fade trail
}
```

### Fluid Simulation (Simplified Navier-Stokes)

For a full fluid sim, you need multiple passes per frame (advection, pressure solve,
divergence correction). A simplified approach uses advection + diffusion:

```glsl
// Single-pass simplified fluid (advection + diffusion + external force)
void main() {
  vec2 uv = gl_FragCoord.xy / u_resolution;
  vec2 texel = 1.0 / u_resolution;

  if (u_frame == 0) {
    gl_FragColor = vec4(0.0);
    return;
  }

  // Read velocity (rg) and density (b) from buffer
  vec4 state = texture2D(u_buffer, uv);
  vec2 vel = state.rg;
  float density = state.b;

  // Advection: trace backwards along velocity to find source
  vec2 source = uv - vel * texel * u_dt;
  vec4 advected = texture2D(u_buffer, source);

  // Diffusion via Laplacian
  vec4 lap = laplacian(u_buffer, uv, texel);
  vec2 newVel = advected.rg + lap.rg * u_viscosity;
  float newDensity = advected.b + lap.b * u_diffusion;

  // External force from mouse
  vec2 mouseForce = (u_mouse - uv) * 0.01 * step(length(u_mouse - uv), 0.05);
  newVel += mouseForce;

  // Density dissipation
  newDensity *= 0.998;

  gl_FragColor = vec4(newVel, newDensity, 1.0);
}
```

## Display Pass

The simulation shader stores data (not colors). A separate display pass or post-process
converts simulation state to visual output:

```glsl
// In the same shader, after simulation, convert to display color:
// Option 1: Color map the simulation state
vec3 color = palette(state.r - state.g, ...);

// Option 2: Use a second shader material for display
// (requires a more complex setup with two shader passes)
```

For simple cases, combine simulation and display in one shader. For complex cases
(fluid sim with multiple solve passes), you'll need a custom render loop.

## Performance Tips

- Use `THREE.NearestFilter` for simulation buffers (no interpolation artifacts)
- Use `THREE.FloatType` for accuracy in PDE simulations; `THREE.HalfFloatType` saves memory
  if precision allows
- Keep simulation resolution independent of display resolution — a 512×512 sim displayed
  at full screen is often sufficient
- Run multiple simulation steps per display frame for faster evolution:
  ```javascript
  for (let i = 0; i < stepsPerFrame; i++) {
    // swap and render to buffer
  }
  // then render to screen
  ```
- For mobile, prefer `THREE.HalfFloatType` and smaller simulation grids (256×256)

## Template Integration Note

The standard `shaderAnimate` function receives `(uniforms, params, time)`. To access
the renderer for ping-pong, store it during setup or use the global `window.__renderer`
that the template exposes. Check `assets/template-3d.html` for the exact API available.

## Key References

- **GPU Gems** — Chapter on GPU-based simulation techniques
- **Reaction-Diffusion Tutorial** — Karl Sims' classic paper on RD systems
- **Shadertoy multipass examples** — Search for "Buffer A" examples demonstrating feedback
- **WebGL Fundamentals** — framebuffer and render-to-texture tutorials
