# MPM Fluid — Material Point Method Reference

## Overview

Material Point Method (MPM) is a hybrid Eulerian-Lagrangian simulation technique: state
lives on **particles** (Lagrangian), but momentum is resolved each step by transferring
to a **background grid** (Eulerian), solving there, and transferring back. It's the
method behind large-scale fluid, sand, snow, and elastic-body simulations in film — and
since 2024 it's been running in real-time in the browser via WebGPU compute at hundreds
of thousands of particles.

This reference covers the MLS-MPM variant (Moving Least Squares MPM) which is the
simplest useful form for generative art, and the implementation pattern used by recent
WebGPU work (Arellano, matsuoka-601). It assumes the WebGPU compute substrate
described in `references/webgpu-compute.md`.

## Why MPM vs grid-based fluid (multipass-buffers.md)

| Aspect | Grid fluid (WebGL ping-pong) | MPM fluid (WebGPU compute) |
|--------|------------------------------|----------------------------|
| State location | Texture cells | Particles + temporary grid |
| Advection | Backward trace lookup | Free — particles carry their own position |
| Free surfaces (air-water interface) | Hard — requires level sets | Automatic — particles simply exist or don't |
| Splashes, droplets | Poor | Excellent |
| Mixing multiple materials | Hard | Natural — per-particle material tag |
| Max scale (browser) | ~512² grid cells | ~300k particles |
| Implementation complexity | Medium | High |

Choose MPM when the piece benefits from droplets, splashes, free surfaces, or the
aesthetic of discrete particles carrying color/identity. Choose grid fluid
(`multipass-buffers.md`) when the target is a full-screen smooth field (smoke, ink in
water, dye advection).

## Algorithm (MLS-MPM)

Each frame executes five compute passes:

```
1. Clear grid
   For each grid cell: mass = 0, velocity = 0

2. Particle-to-grid (P2G)
   For each particle p:
     Compute quadratic B-spline weights w[i,j] for the 3×3 cells around p
     For each (i,j) in those 9 cells:
       cell.mass      += w[i,j] * p.mass
       cell.velocity  += w[i,j] * p.mass * (p.velocity + C_p · (cell_pos − p.pos))
     where C_p is the particle's affine velocity matrix (APIC/MLS)

3. Grid update
   For each cell with mass > 0:
     cell.velocity = cell.velocity / cell.mass      (convert momentum → velocity)
     cell.velocity += gravity * dt                  (apply external forces)
     Enforce boundary conditions (zero velocity in walls)

4. Grid-to-particle (G2P)
   For each particle p:
     Compute weights w[i,j]
     new_vel = sum over 9 cells of w[i,j] * cell.velocity
     new_C   = sum over 9 cells of w[i,j] * cell.velocity ⊗ (cell_pos − p.pos)
                * (4 / h²)
     p.velocity = new_vel
     p.C        = new_C

5. Particle update
   For each particle p:
     p.deformation = (I + dt * p.C) * p.deformation
     p.pos += p.velocity * dt
     Constitutive model step (fluid: pressure from deformation Jacobian)
```

`h` is the grid spacing (typically 1.0 in grid-space coordinates). The 3×3 stencil
comes from the quadratic B-spline (width-2) kernel centered at the particle.

### Constitutive models

What makes the substance behave like water vs honey vs jelly is the **constitutive
model** applied in step 5.

#### Fluid (weakly compressible)

Simplest model. Pressure is derived from the volume ratio `J = det(deformation)`:

```wgsl
// J < 1 means compressed; J > 1 means expanded
// Stiffness k controls how "stiff" the fluid is (resistance to compression)
let J = determinant(p.deformation);
let pressure = k * (J - 1.0);

// Apply pressure force via stress tensor
let stress = -pressure * identity_matrix;
p.C = p.C + stress * (dt * volume / h / h);
```

Tuning: `k` in range 1000–10000 gives recognizable water behavior. Higher = stiffer,
more "splashy". Lower = gooey.

#### Neo-Hookean elastic

For elastic solids (rubber, rigid blobs):

```wgsl
let F = p.deformation;
let J = determinant(F);
let stress = mu * (F * transpose(F) - identity) + lambda * log(J) * identity;
```

`mu` and `lambda` are Lamé parameters.

#### Snow / sand / plasticity

Apply a return-mapping step that clamps the deformation's singular values, shedding
any "plastic" deformation beyond a yield surface. Explained fully in Stomakhin et al.
2013 ("A Material Point Method for Snow Simulation"). For generative art the fluid
and elastic models usually cover the aesthetic goals without the return-map machinery.

## WebGPU implementation notes

### Data layout

```wgsl
struct Particle {
  pos: vec2<f32>,
  vel: vec2<f32>,
  C: mat2x2<f32>,               // affine velocity
  deformation: mat2x2<f32>,      // deformation gradient F
  material: u32,                 // 0=fluid, 1=elastic, 2=...
  mass: f32,
  volume: f32,
  _pad: f32,
};

struct GridCell {
  // Use atomic fixed-point for P2G scatter
  mass: atomic<u32>,                      // multiply float by 1e6
  momentum: array<atomic<i32>, 2>,
  // Cleared each frame
};
```

2D MPM is the practical upper bound for browser real-time; 3D MPM at the same quality
needs discrete-GPU resources beyond the browser's reach in early 2026.

### The atomic P2G trick

Because many particles can contribute to the same grid cell, P2G must use atomics.
WGSL provides `atomicAdd` on `u32` and `i32` storage. Multiply floats by a scale
factor (1e6) and cast:

```wgsl
atomicAdd(&grid[ci].mass, u32(contribution * 1e6));
atomicAdd(&grid[ci].momentum[0], i32(contribution * vel.x * 1e6));
```

In grid-update, divide by 1e6 to recover floats. Chose the scale to fit the expected
magnitude in 32-bit integer range (~4.29e9 for u32, ~2.15e9 for i32).

### Quadratic B-spline weights

```wgsl
fn bspline_weights(x: f32) -> vec3<f32> {
  // x = fractional position within cell, in [0, 1]
  let x0 = x + 0.5;
  let x1 = x - 0.5;
  let x2 = x - 1.5;
  return vec3<f32>(
    0.5 * (1.5 - x0) * (1.5 - x0),  // weight for cell i-1
    0.75 - x1 * x1,                   // weight for cell i
    0.5 * (x2 + 1.5) * (x2 + 1.5),   // weight for cell i+1
  );
}
```

Use the 2D tensor product: `w[i,j] = wx[i] * wy[j]` for the 9 neighbor cells.

### Time-step stability

MPM requires `dt * max_velocity / h < CFL_bound` (Courant-Friedrichs-Lewy). Typical
browser-real-time settings:

| Parameter | Typical value |
|-----------|---------------|
| Grid resolution | 128×128 or 256×256 |
| Grid spacing `h` | 1.0 (grid space) |
| `dt` | 1 / 240 s (4× sub-step per 60fps frame) |
| Particle count | 10k–300k (200k is a comfortable desktop target) |
| `k` (fluid stiffness) | 3000 |
| Gravity | vec2(0, -9.8) or scaled |

If the sim blows up: reduce `dt`, lower stiffness `k`, or increase sub-steps per frame.

## Presets for generative art

Expose a "Material" preset dropdown in the PARAMS:

| Preset | k | mu | lambda | gravity |
|--------|-----|------|--------|---------|
| Water | 3000 | 0 | 0 | 9.8 |
| Honey | 500 | 0 | 0 | 6.0 |
| Lava | 1500 | 0 | 0 | 9.8 (with high viscosity) |
| Jelly | 0 | 1000 | 1000 | 9.8 |
| Rubber | 0 | 3000 | 3000 | 9.8 |
| Sand | 2000 | 500 | 500 | 9.8 (with plasticity) |

Apply the preset in `sketchSetup` by writing to the uniform buffer. Let individual
parameters (`k`, gravity scale, particle count) remain adjustable under the preset.

## Rendering

Three good options:

### Dot sprites (simplest)
Draw each particle as an instanced quad. Color by velocity magnitude, material ID, or
density (sample the grid mass). Cheap; gives a "sparkling water" look.

### Metaball / density surface
Render particles into a screen-space density buffer (fullscreen pass, accumulate per
particle with a Gaussian splat). Threshold or shade to get a smooth surface. Good for
a painterly fluid look.

### Screen-space fluid rendering (SSF)
Rasterize particles as depth points, blur the depth buffer to reconstruct a smooth
surface, shade with a normal derived from the blurred depth. Produces the
filmic-looking water surface you'd expect from offline renders. Complex — start with
dot sprites unless the aesthetic demands it.

## Parameter design

Per the skill's parameter-naming convention, expose parameters in the language of the
phenomenon, not the math:

| Concept | Parameter | Not |
|---------|-----------|-----|
| Viscosity feel | "Thickness" (0 = water, 1 = honey) | stiffness k |
| Gravity | "Gravity" | grav_y |
| Particle count | "Density" | count |
| Material stiffness | "Springiness" or "Elasticity" | lambda/mu |
| Sub-steps | (hide as an advanced control) | — |
| Material picker | "Material" dropdown | — |

Interaction: mouse drag applies an impulse to nearby particles — trivial to add in the
particle update pass and makes the piece far more engaging.

## Limitations and gotchas

- **Quadratic weights (3×3 stencil)** are the standard choice. Cubic (4×4) give
  smoother results but cost 78% more work per particle. Stick with quadratic for
  browser real-time.
- **Incompressibility isn't exact.** Weakly-compressible MPM cheats by using a stiff
  pressure. For visibly incompressible fluid at a small scale, consider a PIC/FLIP
  pressure solve (significantly more complex).
- **Boundary conditions.** The simplest is to clamp particle positions inside the
  simulation region and zero grid velocities at the walls. More physical: use an SDF
  of the container and reflect velocities along the SDF normal.
- **Scaling to 3D.** The algorithm generalizes (3×3×3 = 27 neighbors, 3×3 matrix
  becomes 3×3, etc.). 3D MPM at useful quality is not yet achievable in browser
  real-time; target offline rendering or desktop-class GPUs.

## Key references

- **Jiang, Schroeder, Teran 2015** — "The Affine Particle-In-Cell Method" (APIC) —
  the precursor to MLS-MPM
- **Hu, Fang, Ge, Qu, Zhu, Pradhana, Jiang 2018** — "A Moving Least Squares Material
  Point Method with Displacement Discontinuity and Two-Way Rigid Body Coupling"
  (MLS-MPM) — the paper the browser implementations are based on
- **Stomakhin et al. 2013** — "A Material Point Method for Snow Simulation"
- **nialltl MPM tutorial** (nialltl.neocities.org/articles/mpm_guide.html) — Clearest
  available MPM walkthrough for practitioners
- **matsuoka-601 WebGPU fluid** — public demonstration of real-time 300k-particle
  MPM in the browser (cited in `references/sources.md`)
- `webgpu-compute.md` — required substrate for this technique
- `gpu-particles.md` — simpler particle-system patterns for smaller-scale work
- `multipass-buffers.md` — the grid-fluid alternative; different tradeoffs
