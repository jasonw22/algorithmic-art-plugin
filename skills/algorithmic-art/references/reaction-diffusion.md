# Reaction-Diffusion

## Philosophy

In 1952, Alan Turing published "The Chemical Basis of Morphogenesis" — proposing that biological
patterns (spots, stripes, spirals) arise from two chemicals reacting and diffusing at different
rates. This was radical: **pattern without a blueprint**, form emerging from interaction alone.

Reaction-diffusion embodies **emergence** and **morphogenesis** — the idea that the stunning
complexity of leopard spots, coral textures, and fingerprints needs no master plan, only local
chemical conversations between neighboring cells.

## Key Algorithms

### Gray-Scott Model
Two chemicals A and B on a 2D grid. A is fed in, B is removed. They react: A + 2B → 3B.

```
∂A/∂t = Dₐ∇²A - AB² + f(1-A)
∂B/∂t = D_b∇²B + AB² - (k+f)B
```

- `Dₐ`, `D_b` — diffusion rates (A typically diffuses ~2x faster than B)
- `f` — feed rate (how fast A is replenished)
- `k` — kill rate (how fast B decays)

Different (f, k) pairs produce radically different patterns. These are narrow sweet spots —
small changes can collapse the simulation to all-A or all-B. Always expose these as named
presets rather than raw sliders:

| Preset Name | f | k | Dₐ | D_b | Visual Character |
|-------------|-------|-------|------|------|------------------|
| Mitosis | 0.055 | 0.062 | 1.0 | 0.5 | Dividing cells, organic blobs |
| Coral | 0.030 | 0.057 | 1.0 | 0.5 | Labyrinthine, brain-like folds |
| Spots | 0.025 | 0.060 | 1.0 | 0.5 | Stable circular dots |
| Worms | 0.078 | 0.061 | 1.0 | 0.5 | Elongated stripes and tendrils |
| Solitons | 0.039 | 0.058 | 1.0 | 0.5 | Pulsing, self-replicating dots |
| Maze | 0.029 | 0.057 | 1.0 | 0.5 | Tight labyrinthine corridors |
| Bubbles | 0.012 | 0.050 | 1.0 | 0.5 | Expanding hollow rings |

Implement presets as a dropdown parameter. When the user selects a preset, update f, k, Dₐ,
and D_b to the preset values and re-seed the simulation. The user can then fine-tune from
the preset starting point. This is critical for usability — without presets, users will
struggle to find interesting parameter regions and will mostly see blank or fully-filled screens.

### Laplacian (∇²) Computation
Use a 3×3 convolution kernel for the discrete Laplacian:
```
[0.05  0.2  0.05]
[0.2  -1.0  0.2 ]
[0.05  0.2  0.05]
```

### Anisotropic / Field-Coupled Diffusion

The standard Gray-Scott model uses **isotropic** diffusion — chemicals spread equally in all
directions. By coupling an external vector field (magnetic, gravitational, flow-based) to the
diffusion tensor, patterns become **anisotropic** — stretching and aligning along field lines.

This simulates phenomena like ferrofluids under magnetic fields, where Turing patterns elongate
into filaments tracing the invisible field geometry.

**Implementation:** Replace the isotropic Laplacian for chemical B with a directional blend:

1. Compute the external field direction `(nx, ny)` and magnitude at each cell
2. Compute directional second derivative **along** field lines:
   `L_along = B(x+nx, y+ny) + B(x-nx, y-ny) - 2·B(x,y)` (bilinear sampling for sub-pixel)
3. Compute directional second derivative **perpendicular** to field lines:
   `L_perp = B(x-ny, y+nx) + B(x+ny, y-nx) - 2·B(x,y)`
4. Blend with the isotropic Laplacian:
   ```
   effective = anisotropy * field_magnitude.min(1.0)
   L_aniso = (1 + effective) · L_along + (1 - 0.5·effective) · L_perp
   L_final = L_aniso · 0.5 + L_iso · (1 - effective)
   ```

**Optional drift/advection term:** Add a term that pushes chemical B along field lines using
upwind differencing: `drift · (B_backward - B_forward) / 2`. This creates streaming effects
where patterns flow toward or away from field sources.

**Key parameters:**
- `anisotropy` (0–1): How much the field warps diffusion direction. 0 = standard isotropic RD.
- `drift` (0–1): How much the field advects the pattern chemical along its lines.
- Field source positions and polarities (user-interactive placement works well)

**Field sources:** Point monopoles are simplest — `B = strength · polarity · r̂ / (|r|² · scale + ε)`.
Dipoles, quadrupoles, or Perlin-noise-based fields create different pattern geometries. Multiple
sources with opposing polarity create rich saddle-point topologies.

**Visual character:** Labyrinthine folds become directional filaments near poles; spots elongate
into teardrops; worms align into parallel bundles following field lines. The effect is most
dramatic with "Coral" and "Spots" presets where the base patterns have strong isotropic symmetry
that the field visibly breaks.

### FitzHugh-Nagumo
A simplified model of neural excitation. Produces spiral waves and traveling pulses.
Two variables: activator (v) and inhibitor (w).

**Parameters**: epsilon (time scale separation), a, b (inhibitor dynamics)

## Notable Artists & Works

- **Karl Sims** — *Primordial Dance*, reaction-diffusion on GPU
- **Jonathan McCabe** — multi-scale Turing patterns, stunning organic textures
- **Andy Lomas** — *Morphogenetic Creations*, 3D reaction-diffusion sculptures
- **Sage Jenson** — biological simulation art, flowing organic forms
- **Sachiko Kodama** — *Protrusion* series, ferrofluid sculpture under magnetic fields;
  pioneered the use of magnetic fluids as a dynamic art medium, making invisible
  magnetic field geometry visible through physical pattern formation

## p5.js Implementation Notes

- Render mode: **canvas** (pixel-level simulation)
- Use two flat arrays (current/next) for A and B. Swap each frame.
- `loadPixels()` / `updatePixels()` to write colors from the B concentration.
- Run multiple simulation steps per frame for visible evolution (10-20 steps/frame).
- For performance, avoid creating objects per pixel. Use typed arrays: `new Float32Array(w*h)`.
- Color mapping: map B concentration to a gradient. Low B = background, high B = pattern color.
  HSB mode works well for organic palettes.
- Grid resolution can be lower than canvas — simulate at half-res and upscale with `image()`.
- **Critical**: When using `createGraphics()` for offscreen pixel buffers, call `buffer.pixelDensity(1)` immediately after creation. Without this, high-DPI displays make the pixel array larger than expected, and manual `(y * w + x) * 4` indexing only fills a fraction of the buffer.
- Seed the simulation by setting B=1 in a small region (circle, line, or noise pattern).
