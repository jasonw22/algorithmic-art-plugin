# Strange Attractors

## Philosophy

Strange attractors visualize **deterministic chaos** — systems governed by precise equations
that nonetheless produce behavior so complex it appears random. The Lorenz attractor, discovered
in 1963 while modeling weather, revealed that tiny differences in starting conditions lead to
wildly divergent paths — the "butterfly effect."

The philosophical weight: **the universe can be lawful and unpredictable at the same time**.
Strange attractors are the shapes chaos makes when it settles — never repeating, never escaping,
infinitely detailed. They are portraits of order hiding inside disorder.

## Key Algorithms

### Lorenz Attractor
```
dx/dt = σ(y - x)
dy/dt = x(ρ - z) - y
dz/dt = xy - βz
```
Classic parameters: σ=10, ρ=28, β=8/3. Produces the iconic butterfly shape.
Integrate with small dt (0.005-0.01) and draw the 3D path projected to 2D.

**Parameters**: σ (sigma), ρ (rho), β (beta), dt, rotation angle, trail length

### Clifford Attractor
```
x(n+1) = sin(a·y(n)) + c·cos(a·x(n))
y(n+1) = sin(b·x(n)) + d·cos(b·y(n))
```
Four parameters (a, b, c, d) that produce dramatically different patterns.
Iterate millions of times, plotting each point with low opacity.

Interesting parameter ranges: a,b ∈ [-2, 2], c,d ∈ [-2, 2]

**Parameters**: a, b, c, d, iterations per frame, point opacity, color mode

### De Jong Attractor
```
x(n+1) = sin(a·y(n)) - cos(b·x(n))
y(n+1) = sin(c·x(n)) - cos(d·y(n))
```
Similar to Clifford but produces different structural families. Very sensitive to parameters.

### Hénon Map
```
x(n+1) = 1 - a·x(n)² + y(n)
y(n+1) = b·x(n)
```
Classic: a=1.4, b=0.3. Produces a fractal curve. Simple but foundational.

### Thomas Attractor
```
dx/dt = sin(y) - b·x
dy/dt = sin(z) - b·y
dz/dt = sin(x) - b·z
```
Dissipation parameter b controls complexity. Near b=0.2, produces beautiful 3D knots.

## Notable Artists & Works

- **Edward Lorenz** — the original butterfly, born from weather modeling
- **Jürgen Richter-Gebert** — mathematical visualization and attractor art
- **Julius Horsthuis** — fractal/attractor VR environments
- **Chaotic Atmospheres (blog)** — systematic exploration of attractor aesthetics

## p5.js Implementation Notes

- Render mode: **canvas** (point-cloud plotting with millions of low-opacity points)
- For 2D attractors (Clifford, de Jong): iterate in batches per frame (10,000-100,000 points).
  Use `p.stroke(r, g, b, alpha)` with very low alpha (1-5) and `p.point(x, y)`.
- Scale coordinates to canvas: find attractor bounds empirically, then `map()` to canvas dimensions.
- For 3D attractors (Lorenz, Thomas): use Euler integration, project 3D→2D with simple
  perspective or parallel projection. Rotate slowly for visual interest.
- Color: map point density, position, or velocity to color. Accumulation buffers give
  the classic attractor "glow" effect naturally through point overlap.
- Background: use solid black (`background(0)` once in setup, never clear). Points accumulate.
- Reset: provide a button to clear and re-randomize starting point.
- Performance: `p.noStroke()` + `p.fill()` + `p.rect(x,y,1,1)` is often faster than `p.point()`.
  Or use `pixels[]` directly for maximum throughput.

## Functional Iteration Pattern

The thi.ng/transducers ecosystem models attractor iteration as composable functional pipelines
rather than imperative loops:

```javascript
// Attractor as a pure function: state → next_state
function cliffordStep(a, b, c, d) {
  return ([x, y]) => [
    Math.sin(a * y) + c * Math.cos(a * x),
    Math.sin(b * x) + d * Math.cos(b * y),
  ];
}

// Generate N points as a lazy sequence
function* attractorSequence(stepFn, start, n) {
  let state = start;
  for (let i = 0; i < n; i++) {
    state = stepFn(state);
    yield state;
  }
}

// Collect points as data, then render (enables both Canvas and SVG output)
const step = cliffordStep(a, b, c, d);
const points = [...attractorSequence(step, [0.1, 0.1], 100000)];
```

This separation of iteration from rendering allows:
- **Density histogram**: accumulate point counts into a 2D grid, then color by density
  for high-quality attractor visualization (more precise than low-alpha point plotting)
- **SVG export**: convert point cloud to SVG circles or a density-mapped raster
- **Parameter sweeps**: generate multiple attractors with different parameters in one pass

## Perceptual Color for Attractors

Point-cloud attractors benefit greatly from perceptual color mapping:

- **Density → color**: accumulate points into a 2D histogram, then map density through a
  cosine gradient or Oklab multi-stop gradient. Low density = dark/cool, high density =
  bright/warm. Using Oklab ensures the brightness gradient is perceptually linear.
- **Velocity → color**: compute `|state(n+1) - state(n)|` per point, map speed to palette.
  Fast-moving regions (near repelling fixed points) get one color, slow regions (near
  attracting manifolds) get another.
- **Position → color**: map x, y, or angle from origin to palette for spatial color variation.

See `references/color-science.md` for cosine gradient presets, Oklab conversion functions,
and LCH theme generation.
