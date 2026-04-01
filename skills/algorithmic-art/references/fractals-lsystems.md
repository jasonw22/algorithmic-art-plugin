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
