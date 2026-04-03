# Fractals & L-Systems

## Philosophy

Fractals embody **self-similarity** — the idea that the same structure repeats at every scale,
from coastlines to blood vessels to galaxies. Benoit Mandelbrot's 1982 *The Fractal Geometry
of Nature* argued that classical geometry fails to describe the natural world, and that fractional
dimensions reveal a hidden order in apparent chaos.

L-systems (Lindenmayer systems) formalize **growth as rewriting**. Aristid Lindenmayer created
them in 1968 to model plant development. A simple string-rewriting grammar produces startlingly
lifelike branching structures — encoding the idea that complexity in nature emerges from recursive
application of simple rules.

## Key Algorithms

### Mandelbrot Set
For each point c in the complex plane, iterate z = z² + c starting from z = 0.
Color based on escape iteration count. Points that never escape are "in" the set.

```
z(n+1) = z(n)² + c
escape when |z| > 2
```

**Parameters**: center (real, imag), zoom, max iterations, color mapping

### Julia Sets
Same iteration z = z² + c, but c is fixed and you vary the starting z across the plane.
Each value of c produces a different Julia set — connected when c is in the Mandelbrot set,
dust-like when outside.

**Parameters**: c_real, c_imag, zoom, max iterations

### L-Systems
A grammar: alphabet, axiom (start string), production rules.
Interpret the final string as turtle graphics commands.

Classic rules:
- **Koch curve**: F → F+F−F−F+F (angle 90°)
- **Sierpinski**: F → F−G+F+G−F, G → GG (angle 60°)
- **Plant**: F → FF, X → F−[[X]+X]+F[+FX]−X (angle 25°)
- **Dragon curve**: F → F+G, G → F−G (angle 90°)

**Parameters**: axiom, rules, angle, iterations, segment length, randomness

### Barnsley Fern (IFS)
Four affine transformations applied randomly with weighted probability.
Each maps a point to a new point; over millions of iterations, a fern emerges.

**Parameters**: transformation coefficients, probability weights

## Notable Artists & Works

- **Mandelbrot** — foundational visualization of fractal sets
- **Karl Sims** — evolved virtual creatures using fractal-like genetic programs
- **William Latham** — organic fractal sculptures, *The Conquest of Form*
- **Jock Cooper** — fractal art prints exploring color and depth in the Mandelbrot set

## p5.js Implementation Notes

- Mandelbrot/Julia: use `loadPixels()` / `updatePixels()` with `pixels[]` array for performance.
  Render mode: **canvas** (pixel-level).
- L-systems: parse string then draw with `translate()`/`rotate()`/`line()`.
  Render mode: **svg** (vector-native, all line segments).
- Use `push()`/`pop()` for bracket-based branching in L-systems (the `[` and `]` commands).
- For deep zooms on Mandelbrot, consider reducing canvas size or using perturbation theory
  approximations — standard float64 loses precision around zoom 10^14.
- Color mapping: map iteration count to HSB for smooth gradients.

## nannou Implementation Notes

- **Mandelbrot/Julia**: Render per-pixel to an `image::ImageBuffer`, then display as a
  texture (same approach as reaction-diffusion). This is far faster than drawing individual
  rectangles per pixel. See `references/nannou.md` for the texture-from-image pattern.
- **L-systems**: Use nannou's transform chaining — `draw.x_y(x, y).rotate(angle)` — for
  turtle graphics. Store the turtle state stack as `Vec<(Vec2, f32)>` for push/pop branching.
  Draw segments with `draw.line().start(p1).end(p2).weight(w).color(c);`
- **Recursive trees**: The Nature of Code fractal examples (`chp_08_fractals/`) demonstrate
  basic recursive branching. For more sophisticated growth, see MacTuitui's `tree.rs` —
  a space-colonization algorithm using quadtree spatial indexing for collision avoidance.

### Space-Colonization / Organic Growth

MacTuitui's `tree.rs` (in nannou's `examples/offline/`) demonstrates a sophisticated
growth pattern useful for generative trees, corals, roots, and neural networks:

- **Things** (nodes) grow outward from a root, branching probabilistically
- Each node tracks `parent: Option<usize>` and `children: Vec<usize>` as indices
- **Energy** propagates from root to leaves — nodes only grow when they have energy
- **Quadtree** provides O(log n) spatial queries for collision detection
- New branches spawn at angles biased by the parent-to-child direction, with controlled randomness
- Growth terminates when a node collides with another or hits a boundary

This produces organic, asymmetric branching structures — far more natural-looking than
pure recursive L-systems. Combine with per-generation color mapping for depth visualization.

## SDF-Based Fractals (2D)

Signed distance fields offer an alternative approach to 2D fractal construction. Instead of
iterating complex numbers or rewriting strings, build fractal-like forms by composing SDF
primitives with domain operations:

- **Domain fold + SDF primitive**: Apply `abs()` (mirror) and rotation before evaluating
  an SDF primitive. Repeated folding creates Sierpinski-like fractal structures.
- **Iterated SDF**: Apply a sequence of domain transforms (fold, scale, translate) in a loop,
  then evaluate a simple SDF (sphere, box). The iteration count controls fractal detail.

```glsl
// Sierpinski-like 2D fractal via domain folding
float fractalSDF(vec2 p, int iterations) {
  float scale = 1.0;
  for (int i = 0; i < iterations; i++) {
    p = abs(p) - 0.5;           // fold (mirror at ±0.5)
    p *= mat2(0.866, -0.5, 0.5, 0.866); // rotate 30°
    scale *= 2.0;
  }
  return sdBox(p, vec2(0.5)) / scale;
}
```

See `references/sdf-2d.md` for 2D SDF primitives and boolean operators, and
`references/shaders-glsl.md` for 3D SDF fractals (Mandelbulb, Menger sponge).

## Perceptual Coloring for Fractals

Classic Mandelbrot/Julia coloring maps iteration count to a palette. Using perceptual color
spaces dramatically improves the result:

- **Oklab interpolation** between palette stops prevents the banding and muddy transitions
  common with sRGB gradients in fractal zoom animations.
- **Cosine gradients** (see `references/color-science.md`) are particularly well-suited —
  the smooth periodic nature of cosine palettes matches the cyclic structure of escape-time
  coloring. Four coefficient vectors produce infinite smooth variation.
- **Orbit trap coloring** benefits from mapping trap distance through an Oklab gradient rather
  than direct RGB mapping, producing more vivid and perceptually even results.
