# Cellular Automata

## Philosophy

Cellular automata embody **emergence from simplicity** — the discovery that profound complexity
can arise from trivially simple rules applied to a grid of cells. Stephen Wolfram's *A New Kind
of Science* (2002) argued this is a fundamental principle of the universe itself: simple programs,
not equations, may underlie physical reality.

John Conway's Game of Life (1970) demonstrated that four rules about birth, survival, and death
could produce self-replicating patterns, gliders, computers, and infinite variety — all from an
initial configuration of on/off cells. The philosophical implication: **life-like behavior needs
no life-like rules**.

## Key Algorithms

### Elementary Cellular Automata (1D, Wolfram)
A row of cells, each 0 or 1. Each cell's next state depends on its current state and two neighbors
(3 cells → 8 possible configurations → 256 possible rules, numbered 0-255).

Famous rules:
- **Rule 30**: chaotic, used for random number generation
- **Rule 90**: Sierpinski triangle
- **Rule 110**: proven Turing-complete
- **Rule 184**: traffic flow model

Display: each generation as a new row, producing a 2D spacetime diagram.

**Parameters**: rule number, initial condition, cell size, generations

### Conway's Game of Life (2D)
Grid of cells, alive or dead. Each frame:
1. Live cell with 2-3 live neighbors survives
2. Dead cell with exactly 3 live neighbors becomes alive
3. All other cells die or stay dead

**Parameters**: grid size, cell size, initial density, speed, wrap-around toggle

### Variations
- **Brian's Brain**: three states (on, dying, off) — produces persistent chaotic motion
- **Langton's Ant**: single agent on a grid, turns based on cell color, produces emergent highway
- **Wireworld**: four states, simulates electronic circuits
- **Continuous automata**: smooth state values, smooth transition rules — organic textures

## Notable Artists & Works

- **Casey Reas** — co-creator of Processing, *Process* series exploring cellular-like agent systems
- **John Horton Conway** — Game of Life itself is an artwork of mathematical elegance
- **Stephen Wolfram** — systematic exploration of rule space as visual catalog
- **Daniel Shiffman** — educational visualizations of CA in *The Nature of Code*

## p5.js Implementation Notes

- Render mode: **canvas** for pixel-level (fast, handles large grids) or
  **svg** for rectangle-per-cell (vector-clean, works for smaller grids ≤100×100)
- Use two arrays (current/next generation), swap each frame
- For 1D CA: draw one row per generation, scrolling down. `set(x, y, color)` or `pixels[]`.
- For Life: double-buffered grid. Use modular arithmetic for wrap-around edges.
- Performance: for large grids (>200×200), use typed arrays and avoid object allocation.
  `Uint8Array` is ideal for binary states.
- Interaction: let users click to toggle cells or paint patterns before starting.
  `mousePressed()` / `mouseDragged()` mapped to grid coordinates.
- Seed patterns: random with configurable density, or classic structures
  (glider, glider gun, R-pentomino, acorn).

## nannou Implementation Notes

- Same double-buffered grid approach as p5.js, but use `Vec<u8>` or `Vec<i32>` for state.
- For rendering, draw colored rectangles per cell: `draw.rect().x_y(wx, wy).w_h(size, size).color(c);`
  In release mode, nannou batches draw calls — grids up to ~200x200 render smoothly this way.
- For larger grids, render to an `image::ImageBuffer` and display as a texture (same approach
  as reaction-diffusion — see `references/nannou.md`).

**Hexagonal grids** — The nannou Nature of Code example (`7_hexagon_cells.rs`) demonstrates
CA on hexagonal grids using `draw.polygon().points(hex_vertices)`:

```rust
let n_sides = 6;
let points = (0..n_sides).map(|i| {
    let phase = i as f32 / n_sides as f32;
    let x = radius * (TAU * phase).cos();
    let y = radius * (TAU * phase).sin();
    pt2(x, y)
});
draw.polygon()
    .x_y(cell_x, cell_y)
    .color(fill)
    .stroke(BLACK)
    .points(points);
```

Hex grid layout: offset every other row by `1.5 * cell_width`. Row spacing is
`sin(60°) * cell_width`. This produces visually richer CA than square grids — six
neighbors instead of four (or eight with diagonals) creates different emergent dynamics.
