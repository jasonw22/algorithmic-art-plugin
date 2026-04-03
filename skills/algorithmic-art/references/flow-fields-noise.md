# Flow Fields & Noise

## Philosophy

Flow fields express **organic movement** — the sense that invisible forces guide visible forms,
like wind through grass or current through water. They bridge the gap between randomness and order:
each particle follows a deterministic path, yet the ensemble creates fluid, unpredictable beauty.

Ken Perlin invented Perlin noise in 1983 for the film *Tron*, seeking a way to make computer
graphics look less sterile. His insight: **structured randomness** — noise that varies smoothly
in space and time, unlike the harsh static of random numbers. This became the foundation of
natural-looking procedural textures and terrain.

## Key Algorithms

### Perlin / Simplex Noise
Gradient noise functions that produce smooth, continuous random values at any point in space.
p5.js provides `noise(x, y, z)` (Perlin) built-in.

- **Octaves**: layer multiple noise frequencies for fractal detail (fBm)
- **Lacunarity**: frequency multiplier between octaves (typically 2.0)
- **Persistence**: amplitude multiplier between octaves (typically 0.5)

```javascript
// Fractal Brownian Motion
function fbm(x, y, octaves, lacunarity, persistence) {
  let value = 0, amplitude = 1, frequency = 1, maxAmp = 0;
  for (let i = 0; i < octaves; i++) {
    value += amplitude * noise(x * frequency, y * frequency);
    maxAmp += amplitude;
    amplitude *= persistence;
    frequency *= lacunarity;
  }
  return value / maxAmp;
}
```

### Flow Field (Vector Field)
A grid of angles, each derived from noise. Particles read the angle at their position
and move in that direction. Over time, their trails reveal the field's structure.

```
angle(x, y) = noise(x * scale, y * scale) * TWO_PI * multiplier
```

**Parameters**: noise scale, particle count, speed, trail length/opacity, angle multiplier

### Curl Noise
Take the curl of a 2D noise field to get a divergence-free vector field.
Particles in curl noise never converge or diverge — they flow in closed or spiraling paths.

```
curl_x = (noise(x, y+ε) - noise(x, y-ε)) / (2ε)
curl_y = -(noise(x+ε, y) - noise(x-ε, y)) / (2ε)
```

This produces especially beautiful, fluid particle trails.

### Domain Warping
Feed noise into itself: `noise(x + noise(x,y), y + noise(x,y))`.
Produces swirling, marble-like distortions. Stack multiple layers for extreme warping.

## Notable Artists & Works

- **Tyler Hobbs** — *Fidenza* (2021), one of the most celebrated generative art collections,
  built on flow fields with curated color palettes
- **Zach Lieberman** — flow field experiments in openFrameworks
- **Matt DesLauriers** — pen-plotter flow field prints
- **Manolo Gamboa Naon** — dense, layered flow field compositions

## p5.js Implementation Notes

- Render mode: **canvas** for particle systems (thousands of overlapping semi-transparent dots),
  or **svg** for line-trace style (fewer particles, explicit line segments).
- Trail effect: don't call `background()` every frame. Instead, draw a semi-transparent
  rectangle: `p.fill(0, 0, 0, 10); p.rect(0, 0, w, h);` — this fades old positions.
- For SVG mode: store particle paths as polylines, draw with `beginShape()`/`vertex()`/`endShape()`.
- Particle count: 500-5000 for canvas mode; 50-500 for SVG mode.
- `p.noiseSeed()` with a fixed seed for reproducibility; expose seed as a parameter.
- Use `p.noiseDetail(octaves, falloff)` to control built-in Perlin noise complexity.
- Color: map particle age, speed, or position to color for visual richness.

## nannou Implementation Notes

The canonical nannou flow field pattern uses an **Agent struct** that tracks current and
previous positions, moving through a 3D Perlin noise field (x, y for space, z for time):

```rust
use nannou::noise::{NoiseFn, Perlin, Seedable};

struct Agent {
    pos: Vec2,
    pos_old: Vec2,
    step_size: f32,
    z_noise: f32,       // per-agent z offset for variation
}

impl Agent {
    fn update(&mut self, noise: &Perlin, noise_scale: f64, noise_strength: f64) {
        self.pos_old = self.pos;
        self.z_noise += 0.01; // animate through noise z-axis

        let angle = noise.get([
            self.pos.x as f64 / noise_scale,
            self.pos.y as f64 / noise_scale,
            self.z_noise as f64,
        ]) as f32 * noise_strength as f32;

        self.pos.x += angle.cos() * self.step_size;
        self.pos.y += angle.sin() * self.step_size;
    }
}
```

**Key patterns from the nannou Generative Design examples (`m_1_5_04.rs`):**
- **Agent count**: 1000–5000 agents is comfortable in release mode
- **Trail rendering**: Use the overlay technique — draw a semi-transparent rect each frame
  instead of clearing (see `references/nannou.md` → Trail / Overlay Technique)
- **Draw modes**: Toggle between line segments (`draw.line().start(old).end(new)`) and
  ellipses (`draw.ellipse().xy(pos).radius(r)`) with keyboard shortcuts
- **Wrapping**: When agents leave the window bounds, wrap to the opposite edge and reset
  `pos_old` to avoid cross-screen streaks
- **Per-agent variation**: Randomize `step_size`, `z_noise` offset, and color per agent.
  Use `random_f32()` at creation time, not per-frame
- **Color**: Assign HSL color based on a random value at agent creation. Use agent `z_noise`
  or initial randomizer to split agents across two hue ranges for visual richness
- **Blend modes**: `BLEND_ADD` with dark background creates glowing convergence effects.
  Apply with `let draw = draw.color_blend(BLEND_ADD);`

## Perceptual Color for Flow Fields

Flow field aesthetics improve dramatically with perceptual color mapping:

- **Cosine gradients** (see `references/color-science.md`) produce infinitely smooth palettes
  from 4 coefficient vectors — ideal for mapping particle age or distance to color.
- **Oklab interpolation** prevents the muddy midpoints that sRGB interpolation produces when
  blending between distant hues (e.g., red → blue trails passing through vivid purple instead
  of gray).
- **LCH theme generation** constrains palette variety to a perceptual range — use narrow hue
  ranges (±15°) for calm, monochromatic fields or wide ranges (±60°) for vibrant compositions.

Map color to particle properties:
```javascript
// Map particle cumulative distance to cosine palette
const t = particle.totalDistance / maxExpectedDistance;
const [r, g, b] = cosinePalette(t, palette.a, palette.b, palette.c, palette.d);
p.stroke(r * 255, g * 255, b * 255, trailAlpha);
```

## Functional Composition Pattern

The thi.ng ecosystem demonstrates a functional alternative to the imperative agent loop.
Instead of mutable agent objects, model the flow field as a composable pipeline:

1. **Field function**: `(x, y, t) → angle` — pure function, no state
2. **Particle step**: `(position, field) → newPosition` — deterministic transform
3. **Trail accumulation**: collect positions into polylines as data
4. **Rendering**: draw polylines from data (enables both Canvas and SVG output)

This separation of data from rendering makes SVG export trivial — the same trail data that
draws to canvas can construct SVG `<polyline>` elements directly.
