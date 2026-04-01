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
