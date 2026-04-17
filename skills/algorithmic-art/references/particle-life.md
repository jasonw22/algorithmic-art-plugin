# Particle Life — Asymmetric-Force Swarm Reference

## Overview

Particle Life (popularized by Jeffrey Ventrella and Tom Mohr's mid-2010s experiments, with
a recent revival on WebGPU) is a family of particle-system simulations built around one
deceptively simple idea: *make forces between species asymmetric*. Species A attracts
species B, but species B repels species A. That single asymmetry generates recognizable
emergent behaviors — predator-prey chases, orbit systems, membrane-like boundaries,
breathing flocks — from otherwise trivial physics.

It sits next to classical boids/flocking (`references/generative-agents.md`) but occupies
a very different aesthetic space: flocking converges on smooth, schooling motion; Particle
Life produces restless, organismal dynamics that look almost biological.

## The core idea

Each particle belongs to a species (color). Between any two particles, the force is
governed by a **force matrix** `F[i][j]` — a square `S × S` matrix where `S` is the
number of species. The entry `F[i][j]` is the force species `i` feels from species `j`
at the characteristic distance. Crucially, `F[i][j]` is *not* required to equal
`F[j][i]` — breaking Newton's third law is the whole point.

```
         toward species →
         j=0    j=1    j=2    j=3
   i=0 [ 0.2  -0.1   0.3    0.5 ]
   i=1 [ 0.4   0.1  -0.2    0.0 ]  ← species i=1 is attracted to j=0 (0.4)
   i=2 [ 0.1   0.3  -0.3    0.2 ]       but j=0 is repelled by i=1 (-0.1)
   i=3 [-0.2   0.4   0.1    0.0 ]
         (species i feels this force toward species j)
```

A common force curve has three regions:

```
 force
   │
   │    ___________
   │   /           \
 0 │──╱             ╲──────────
   │ ╱               ╲
   │╱                 ╲___
   │                      ╲
   │                       ╲___
   └───────────────────────────── distance
   0  rmin          rmax
```

- `d < rmin`: strong repulsion (prevents particles from merging)
- `rmin < d < rmax`: attractive/repulsive force set by `F[i][j]` (positive = attract)
- `d > rmax`: zero force (cutoff for efficiency)

Typical values: `rmin = 0.1`, `rmax = 0.3` in a unit-square world.

## WGSL compute shader

```wgsl
struct Particle {
  pos: vec2<f32>,
  vel: vec2<f32>,
  species: u32,
  _pad: f32,
};

struct Uniforms {
  time: f32,
  resolution: vec2<f32>,
  seed: f32,
  count: u32,
  species_count: u32,
  dt: f32,
  friction: f32,
  rmin: f32,
  rmax: f32,
  force_scale: f32,
};

@group(0) @binding(0) var<uniform> U: Uniforms;
@group(0) @binding(1) var<storage, read> p_in: array<Particle>;
@group(0) @binding(2) var<storage, read_write> p_out: array<Particle>;
// Row-major: F[i * S + j] is force i feels from j
@group(0) @binding(3) var<storage, read> F: array<f32>;

fn force_between(d: f32, f_attract: f32) -> f32 {
  // d is normalized distance in [0, rmax]
  if (d < U.rmin) {
    // Strong repulsion inside rmin
    return -1.0 * (1.0 - d / U.rmin);
  } else if (d < U.rmax) {
    // Attractive/repulsive in the attract zone
    let t = (d - U.rmin) / (U.rmax - U.rmin);
    // Tent shape: peaks in the middle, zero at the ends
    return f_attract * (1.0 - abs(2.0 * t - 1.0));
  }
  return 0.0;
}

@compute @workgroup_size(64)
fn cs_main(@builtin(global_invocation_id) gid: vec3<u32>) {
  let i = gid.x;
  if (i >= U.count) { return; }

  let self = p_in[i];
  var accel = vec2<f32>(0.0);

  // O(N²) — acceptable up to ~5000 particles. For more, use spatial hashing.
  for (var j: u32 = 0u; j < U.count; j = j + 1u) {
    if (j == i) { continue; }
    let other = p_in[j];
    let delta = other.pos - self.pos;
    let dist = length(delta);
    if (dist < 0.0001 || dist > U.rmax) { continue; }

    let f_a = F[self.species * U.species_count + other.species];
    let f = force_between(dist, f_a);
    accel = accel + normalize(delta) * f * U.force_scale;
  }

  var new_vel = self.vel + accel * U.dt;
  new_vel = new_vel * (1.0 - U.friction);

  var new_pos = self.pos + new_vel * U.dt;

  // Wrap-around (toroidal) world — swap for reflection if you prefer
  new_pos.x = fract(new_pos.x);
  new_pos.y = fract(new_pos.y);

  var out = self;
  out.pos = new_pos;
  out.vel = new_vel;
  p_out[i] = out;
}
```

## Scaling to large N: spatial hashing

Naive O(N²) works to ~5k particles. For 20k–200k, use a uniform-grid spatial hash:

1. **Bin pass:** place each particle into a grid cell based on `pos / rmax`. Use atomic
   counters to track cell sizes, then a prefix sum to build a compact particle-index
   array sorted by cell.
2. **Force pass:** for each particle, only iterate over particles in its cell and the 8
   surrounding cells.

This drops complexity from O(N²) to O(N) assuming uniform density. Implementation is
three-to-four compute passes per frame; see `webgpu-compute.md` for the scatter/gather
patterns with atomics.

## Force-matrix presets

The force matrix is the character of the simulation. Expose it as a "Preset" dropdown
with known-good matrices, plus a "Randomize matrix" button that draws each entry from
a uniform distribution in `[-1, 1]` (seeded, of course). Some named behaviors:

| Preset | Character | Notes |
|--------|-----------|-------|
| **Predator-Prey** | `F[A][B] = +0.5, F[B][A] = -0.5` | Species A chases B; B flees A |
| **Orbit** | Antisymmetric matrix (`F[i][j] = -F[j][i]`) | Species orbit each other rather than pursue |
| **Membrane** | `F[A][A] = +0.3, F[B][B] = +0.3, F[A][B] = -0.3` | Species form separated clumps with clear boundaries |
| **Swarm** | Uniformly mild attraction `F[i][j] ≈ +0.2` | Smooth flocking across species |
| **Chaos** | Fully randomized | Anything goes; some seeds are boring, some stunning |
| **Flower** | Species 0 weakly attracts all others; others weakly attract species 0 | Forms radial patterns around species-0 cores |

Always offer a "Random matrix" button. Discovery is half the experience.

## Parameter design

Per the skill's parameter-naming convention, use the language of the phenomenon:

| Concept | Parameter | Not |
|---------|-----------|-----|
| Particle count | "Population" | count |
| Interaction radius | "Reach" | rmax |
| Minimum separation | "Personal space" | rmin |
| Force strength | "Intensity" | force_scale |
| Motion damping | "Water resistance" or "Friction" | friction |
| Number of colors | "Species" | species_count |
| Force matrix | "Preset" dropdown + "Randomize" button | — |
| Wrap world | "Toroidal" toggle | — |

Good default: 4 species, 2000 particles, `rmin=0.1`, `rmax=0.3`, `force_scale=0.4`,
`friction=0.08`, `dt=0.016`.

## Aesthetic notes

- **Color by species, not position.** Particle Life is about identity, and mapping
  species to distinct hues (use perceptual-color presets from `color-science.md`) makes
  the dynamics legible.
- **Tiny particles + additive blending.** Point size 1–2 px with additive blending
  produces beautifully dense swarms where overlapping species produce new hues.
- **Trails optional.** If rendering to a fading accumulation buffer instead of clearing
  each frame, the piece reads more like a drawing than an animation. See
  `multipass-buffers.md` for the technique.
- **Long time-horizons pay off.** Some matrices take 30+ seconds of real-time
  simulation before settling into their characteristic pattern. Build patience into
  the piece — don't randomize so often the user never sees settled behavior.

## CPU variants (small N)

For 100–500 particles, Particle Life runs comfortably in p5.js or nannou on CPU. The
algorithm is identical; just iterate in JS/Rust. Useful for:

- Prototyping and debugging new force-matrix presets before moving to WebGPU
- Pen-plotter output: CPU Particle Life → trail accumulation → SVG export
  (see `plotter-workflow.md`)
- Pieces where the low particle count is the aesthetic (e.g. ~30 large, slow particles
  reading as "cells")

## Relation to flocking / boids

Boids (`references/generative-agents.md`) are a special case: one species, symmetric
forces (cohesion + separation + alignment). Particle Life generalizes by dropping the
symmetry requirement and the alignment force, and adding species labels. Many of the
aesthetics of flocking can be reproduced by choosing a nearly-symmetric force matrix
with mild positive self-attraction.

## Key references

- **Jeffrey Ventrella — Clusters** (ventrella.com/Clusters) — Early exploration of
  asymmetric-force particle life
- **Tom Mohr's Particle Life** — Widely cloned JavaScript implementation; source for
  many of the known-good matrices
- **Code Parade — Particle Life** (YouTube) — Popular video introduction with
  intuitions about why certain matrices produce certain behaviors
- `webgpu-compute.md` — substrate for large-N Particle Life
- `generative-agents.md` — symmetric-force precursor (boids, flocking)
- `color-science.md` — perceptual color palettes for species labeling
- `multipass-buffers.md` — trail-rendering technique via accumulation buffer
