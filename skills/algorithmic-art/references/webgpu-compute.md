# WebGPU Compute — WGSL Reference for Generative Art

## Overview

WebGPU is the successor to WebGL: a modern, explicit GPU API exposed to the browser. For
generative art the key unlock is **compute shaders**. Where WebGL's only "compute" mechanism
was ping-pong fragment shaders writing to framebuffers (`references/multipass-buffers.md`),
WebGPU exposes dedicated compute pipelines with arbitrary storage-buffer layouts, shared
workgroup memory, and dispatch sizes in the millions.

Practically this means pieces that were impossible or painfully slow in WebGL — Particle Life
with 100k+ particles, MLS-MPM particle fluid at 300k particles, large-grid cellular automata
with multi-byte state per cell — now run at 60fps on mid-range laptops.

This reference covers the compute pipeline model, WGSL (the shading language), storage-buffer
layouts, the compute+render flow, and integration notes for the skill's `webgpu` output mode
(see `assets/template-webgpu.html`).

**Browser support (early 2026):** Chrome, Edge, Opera, Brave (stable). Safari 17+ on macOS/iOS.
Firefox behind a flag (`dom.webgpu.enabled`). Always include a graceful fallback message —
the template does this for you.

## WebGPU vs WebGL: when to switch

| Need | Use WebGL (Three.js Shader) | Use WebGPU (this mode) |
|------|-----------------------------|------------------------|
| Fullscreen shader art (Shadertoy-style) | ✅ | Slight overkill — GLSL mode is simpler |
| SDF raymarching, 3D fractals | ✅ | Fine either way; GLSL mode is simpler |
| Small grid simulations (512×512 RD, CA) | ✅ | WebGPU is overkill |
| **Large particle counts (>100k)** | ❌ painful | ✅ |
| **MPM / particle fluid** | ❌ | ✅ |
| **Particle Life at scale** | ❌ | ✅ |
| **Arbitrary data layouts per particle** | ❌ | ✅ |
| **Scatter writes / sparse updates** | ❌ | ✅ |
| Mobile-first delivery | ✅ wider support | ⚠️ still rolling out on iOS |

If the piece fits comfortably in the WebGL Shader mode's fragment+framebuffer model,
**use that mode** — it is simpler to author and runs everywhere. Reach for WebGPU when
you genuinely need compute dispatch, big storage buffers, or more than the ~65k particles
that GPGPU-via-textures can sustain.

## WGSL primer

WGSL (WebGPU Shading Language) is the shading language of WebGPU. It takes cues from Rust
and HLSL. Quick syntax reference:

```wgsl
// Struct definitions — used in uniforms and storage buffers
struct Uniforms {
  time: f32,
  resolution: vec2<f32>,
  seed: f32,
  count: u32,
};

struct Particle {
  pos: vec2<f32>,
  vel: vec2<f32>,
  life: f32,
  _pad: f32,  // WGSL structs in storage buffers must be 16-byte aligned
};

// Uniform buffer binding
@group(0) @binding(0) var<uniform> U: Uniforms;

// Read-write storage buffer (only allowed in compute shaders)
@group(0) @binding(1) var<storage, read_write> particles: array<Particle>;

// Read-only storage buffer (allowed in vertex/fragment too)
@group(0) @binding(2) var<storage, read> particles_prev: array<Particle>;

// Compute entry point — workgroup size declared in attribute
@compute @workgroup_size(64)
fn cs_main(@builtin(global_invocation_id) gid: vec3<u32>) {
  let i = gid.x;
  if (i >= U.count) { return; }  // out-of-bounds guard

  let p = particles_prev[i];
  var np = p;

  // ... simulation logic ...
  np.pos = p.pos + p.vel * 0.016;

  particles[i] = np;
}

// Vertex entry point
struct VsOut {
  @builtin(position) clip: vec4<f32>,
  @location(0) color: vec3<f32>,
};

@vertex
fn vs_main(@builtin(vertex_index) vid: u32,
           @builtin(instance_index) iid: u32) -> VsOut {
  // Draw a quad per particle using instancing
  let quad = array<vec2<f32>, 6>(
    vec2<f32>(-1.0, -1.0), vec2<f32>( 1.0, -1.0), vec2<f32>(-1.0,  1.0),
    vec2<f32>(-1.0,  1.0), vec2<f32>( 1.0, -1.0), vec2<f32>( 1.0,  1.0),
  );
  let offset = quad[vid] * 0.01;
  let p = particles[iid];
  let world = p.pos + offset;

  var out: VsOut;
  out.clip = vec4<f32>(world, 0.0, 1.0);
  out.color = vec3<f32>(0.5 + 0.5 * sin(p.life * 3.1),
                         0.5 + 0.5 * cos(p.life * 2.2),
                         1.0 - p.life);
  return out;
}

// Fragment entry point
@fragment
fn fs_main(in: VsOut) -> @location(0) vec4<f32> {
  return vec4<f32>(in.color, 1.0);
}
```

### WGSL gotchas

- **Storage-buffer structs must be 16-byte aligned.** Add padding (`_pad: f32`) to round
  struct sizes to multiples of 16 bytes. `vec3<f32>` also takes 16 bytes, not 12.
- **No implicit type conversions.** `1.0 / 2` is an error; write `1.0 / 2.0` or `f32(2)`.
- **No preprocessor.** No `#include` or `#ifdef`. Build shader strings in JS if you need
  conditional compilation.
- **`read_write` storage only in compute.** Vertex/fragment shaders can only read
  `<storage, read>` buffers. For particle rendering, use the storage buffer written by
  the compute pass as a read-only input to the vertex shader.
- **Workgroup size is part of the shader.** Declared via `@workgroup_size(x, y, z)`. Tune
  per hardware — 64 for 1D work (particles), 8×8 for 2D grids (textures), 4×4×4 for 3D.

## Compute + render architecture

The core pattern: one or more compute passes advance simulation state in storage buffers,
then a render pass reads those buffers (as `<storage, read>` or by copying to a vertex
buffer) and draws the result.

```
Each frame:
  ┌────────────────────────┐
  │  Compute pass 1..N     │  (simulation — one or more dispatches)
  │  Read storage A        │
  │  Write storage B       │
  └──────────┬─────────────┘
             │ swap ping-pong pointers
             ▼
  ┌────────────────────────┐
  │  Render pass           │  (vertex + fragment)
  │  Read storage B        │
  │  Draw to canvas        │
  └────────────────────────┘
```

Unlike WebGL, you don't need separate render targets to implement ping-pong — two storage
buffers alternate roles each frame.

## Minimal WebGPU setup (JavaScript)

The template (`assets/template-webgpu.html`) handles device init, swapchain, resize, and
the seeded PRNG for you. If you ever need to build WebGPU code outside the template,
here's the minimum:

```javascript
// 1. Request adapter + device
const adapter = await navigator.gpu?.requestAdapter();
if (!adapter) throw new Error("WebGPU not supported");
const device = await adapter.requestDevice();

// 2. Configure canvas context
const canvas = document.querySelector("canvas");
const context = canvas.getContext("webgpu");
const format = navigator.gpu.getPreferredCanvasFormat();
context.configure({ device, format, alphaMode: "premultiplied" });

// 3. Create buffers
const particleBuffer = device.createBuffer({
  size: particleCount * PARTICLE_STRIDE,
  usage: GPUBufferUsage.STORAGE | GPUBufferUsage.COPY_DST,
});

// 4. Create shader module + pipelines
const shaderModule = device.createShaderModule({ code: wgslSource });
const computePipeline = device.createComputePipeline({
  layout: "auto",
  compute: { module: shaderModule, entryPoint: "cs_main" },
});
const renderPipeline = device.createRenderPipeline({
  layout: "auto",
  vertex:   { module: shaderModule, entryPoint: "vs_main" },
  fragment: { module: shaderModule, entryPoint: "fs_main",
              targets: [{ format }] },
  primitive: { topology: "triangle-list" },
});

// 5. Per-frame: record and submit command buffer
function frame() {
  const encoder = device.createCommandEncoder();

  // Compute pass
  const cp = encoder.beginComputePass();
  cp.setPipeline(computePipeline);
  cp.setBindGroup(0, computeBindGroup);
  cp.dispatchWorkgroups(Math.ceil(particleCount / 64));
  cp.end();

  // Render pass
  const rp = encoder.beginRenderPass({
    colorAttachments: [{
      view: context.getCurrentTexture().createView(),
      loadOp: "clear",
      storeOp: "store",
      clearValue: { r: 0, g: 0, b: 0, a: 1 },
    }],
  });
  rp.setPipeline(renderPipeline);
  rp.setBindGroup(0, renderBindGroup);
  rp.draw(6, particleCount);  // 6 verts × N instances
  rp.end();

  device.queue.submit([encoder.finish()]);
  requestAnimationFrame(frame);
}
```

## Template integration

The skill's WebGPU mode exposes three hooks in `assets/template-webgpu.html`:

```javascript
// Called once. Return { pipelines, bindGroups, buffers, ... } used in passes.
function sketchSetup(device, context, format, params, seed) {
  // Create shader module, buffers, pipelines, bind groups
  // seed: integer; use with the global window.seededRandom() for CPU-side randomness,
  // and pass as a u32 uniform to WGSL for GPU-side deterministic hashing
  return { /* state */ };
}

// Called every frame. Record compute dispatches.
function computePass(device, encoder, state, params, time, delta) {
  const cp = encoder.beginComputePass();
  cp.setPipeline(state.computePipeline);
  cp.setBindGroup(0, state.computeBindGroup);
  cp.dispatchWorkgroups(Math.ceil(state.count / 64));
  cp.end();
}

// Called every frame, after computePass. Record render pass to the canvas view.
function renderPass(device, encoder, view, state, params, time, delta) {
  const rp = encoder.beginRenderPass({
    colorAttachments: [{
      view, loadOp: "clear", storeOp: "store",
      clearValue: { r: 0, g: 0, b: 0, a: 1 },
    }],
  });
  rp.setPipeline(state.renderPipeline);
  rp.setBindGroup(0, state.renderBindGroup);
  rp.draw(6, state.count);
  rp.end();
}
```

Ping-pong between two storage buffers: flip `state.readBuffer` / `state.writeBuffer`
(or swap bind-group bindings) inside `computePass`. Uniforms (time, resolution, seed)
are automatically kept in sync by the template — add custom uniforms by defining them
in your `Uniforms` struct and writing to the template's shared `uniformBuffer`
via `device.queue.writeBuffer()` at the top of `computePass`.

## Common patterns

### Uniform-buffer update each frame

```javascript
const uniforms = new Float32Array([
  time,
  window.innerWidth * devicePixelRatio,
  window.innerHeight * devicePixelRatio,
  seed,
  /* custom uniforms here */
]);
device.queue.writeBuffer(state.uniformBuffer, 0, uniforms.buffer);
```

### Multi-step per frame (sub-stepping)

Some simulations need several small timesteps per frame for stability:

```javascript
function computePass(device, encoder, state, params, time, delta) {
  const subSteps = params.subSteps || 4;
  const cp = encoder.beginComputePass();
  for (let i = 0; i < subSteps; i++) {
    cp.setPipeline(state.computePipeline);
    cp.setBindGroup(0, state.bindGroups[i % 2]);  // ping-pong bind groups
    cp.dispatchWorkgroups(Math.ceil(state.count / 64));
  }
  cp.end();
}
```

### Scatter writes + atomics

For particle-to-grid transfers (MPM) and histogram-style effects, atomic operations are
essential. WGSL supports atomics on `i32`/`u32` storage buffer fields:

```wgsl
struct GridCell {
  mass: atomic<u32>,        // fixed-point: multiply float by 1e6, cast to u32
  momentum: array<atomic<i32>, 2>,
};

@group(0) @binding(1) var<storage, read_write> grid: array<GridCell>;

@compute @workgroup_size(64)
fn particle_to_grid(@builtin(global_invocation_id) gid: vec3<u32>) {
  let i = gid.x;
  let p = particles[i];
  let cell_idx = u32(p.pos.y) * GRID_W + u32(p.pos.x);
  atomicAdd(&grid[cell_idx].mass, u32(p.mass * 1e6));
  atomicAdd(&grid[cell_idx].momentum[0], i32(p.mass * p.vel.x * 1e6));
  atomicAdd(&grid[cell_idx].momentum[1], i32(p.mass * p.vel.y * 1e6));
}
```

This is the mechanism that makes MPM (`references/mpm-fluid.md`) possible on the GPU.

### Hashing a seed inside WGSL

Deterministic per-particle randomness:

```wgsl
fn hash11(x: u32) -> u32 {
  var v = x;
  v = (v ^ 61u) ^ (v >> 16u);
  v = v + (v << 3u);
  v = v ^ (v >> 4u);
  v = v * 0x27d4eb2du;
  v = v ^ (v >> 15u);
  return v;
}
fn rand(seed: u32, i: u32) -> f32 {
  return f32(hash11(seed ^ hash11(i))) / 4294967295.0;
}

// In init:
let r = rand(u32(U.seed), i);
```

This keeps output reproducible across seed changes even on GPU.

## Performance notes

- **Workgroup size:** 64 is a safe default for 1D. Experiment with 128 or 256 for larger
  buffers. For 2D textures, 8×8 (=64 threads per group) is standard.
- **Dispatch count:** `dispatchWorkgroups(ceil(N / workgroup_size))`. Include an
  out-of-bounds guard at the top of the entry point (`if (gid.x >= U.count) { return; }`).
- **Buffer binding overhead:** creating new bind groups every frame is expensive. Create
  both ping-pong bind groups at setup time and alternate between them.
- **Readback for PNG export:** copy the canvas via `canvas.toBlob()` as usual — WebGPU
  canvases behave like any HTMLCanvasElement for blob capture.
- **Resize:** must reconfigure the context and recreate any size-dependent textures when
  the canvas resizes. The template handles this.

## Libraries and ecosystem

The skill's template uses **raw WebGPU**. This keeps templates self-contained and minimizes
the "what version is this?" problem. If you want a wrapper for ongoing work outside the
skill, consider:

- **POINTS** by Absulit (github.com/absulit/points) — WebGPU-specific creative-coding
  wrapper with a Processing-like feel. Abstracts pipeline setup, good for rapid
  prototyping.
- **Three.js r160+ WebGPURenderer** — Extends Three.js to WebGPU with a node-based
  material graph (TSL = Three Shading Language). Compiles to WebGPU *or* WebGL, so the
  same piece runs on older browsers. Mustafa Ali's node-based shader editor work is
  built on this path.
- **Slang** (shader-slang.com) — Shading language under Khronos governance that
  compiles to WGSL, GLSL, HLSL, SPIR-V, and includes automatic differentiation for
  neural-graphics workflows. Worth knowing about for research-oriented work; the
  `generative-ai-art` skill touches it for differentiable-rendering pieces.

## Practitioners

- **Hector Arellano** — Pioneering WebGPU fluid simulations on the web; widely cited as
  a reference implementation.
- **matsuoka-601** — WebGPU MLS-MPM fluid running at 300k particles in real time on
  mid-range GPUs; the current high-water mark for browser particle simulation.
- **Mustafa Ali** — Node-based shader editors on Three.js WebGPU/TSL.
- **Absulit** — POINTS library author; extensive WebGPU creative coding examples.

## Key references

- **WebGPU specification** (webgpu.org) — Authoritative reference
- **WebGPU Fundamentals** (webgpufundamentals.org) — Step-by-step tutorials; equivalent
  to webglfundamentals but for WebGPU
- **WGSL specification** (gpuweb.github.io/gpuweb/wgsl) — Language reference
- **Codrops 2025 WebGPU review** — Survey of the state of browser GPU compute for
  creative coding (cited in `references/sources.md`)
- `multipass-buffers.md` — the WebGL analogue; same ping-pong concept, different
  mechanism
- `gpu-particles.md` — GPGPU particles via Three.js; the predecessor for WebGL-era
  particle work
- `mpm-fluid.md` — particle-based fluid simulation built on this WebGPU compute substrate
- `particle-life.md` — asymmetric-force swarm simulations on the same substrate
