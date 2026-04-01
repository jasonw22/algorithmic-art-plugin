# Recursion & Subdivision

## Philosophy

Recursive subdivision explores **compositional hierarchy** — the idea that a whole is made of
parts, which are themselves wholes made of smaller parts. This mirrors how we perceive the world:
a forest is trees is branches is leaves is veins.

Piet Mondrian's grid paintings (though hand-composed) inspired algorithmic subdivision that asks:
**what happens when a machine decides where to divide?** The results range from architectural
floor plans to abstract compositions to procedural game maps — all from the same core idea of
splitting space and making choices at each level.

## Key Algorithms

### Recursive Rectangle Subdivision (Mondrian-style)
Start with a rectangle. Randomly choose to split horizontally or vertically.
Recurse on each half. Stop based on minimum size or depth limit.
Color the terminal rectangles from a palette.

```
function subdivide(x, y, w, h, depth) {
  if (depth == 0 || w < minSize || h < minSize) {
    drawRect(x, y, w, h, randomColor());
    return;
  }
  if (random() < 0.5 || w > h * 1.5) {
    // vertical split
    let split = random(w * 0.3, w * 0.7);
    subdivide(x, y, split, h, depth - 1);
    subdivide(x + split, y, w - split, h, depth - 1);
  } else {
    // horizontal split
    let split = random(h * 0.3, h * 0.7);
    subdivide(x, y, w, split, depth - 1);
    subdivide(x, y + split, w, h - split, depth - 1);
  }
}
```

**Parameters**: max depth, min cell size, split bias (H vs V), gap/border width, palette

### Quadtree Subdivision
Divide a square into 4 equal quadrants. For each quadrant, decide whether to subdivide further
based on some criterion (randomness, image brightness, distance from center).

When driven by an image: subdivide more where detail is high (edges, contrast changes).
This creates a resolution-adaptive mosaic.

**Parameters**: max depth, subdivision probability/threshold, min cell size, fill style

### Binary Space Partition (BSP)
Like Mondrian but with controlled aspect ratios. Used in game level generation.
Split the largest dimension, ensuring children don't get too narrow.

**Parameters**: max depth, min room size, max aspect ratio, corridor width

### Circle Packing
Not strictly subdivision, but a recursive space-filling strategy. Place the largest circle
that fits, then recursively fill remaining space with smaller circles.

**Parameters**: min radius, max radius, max attempts per frame, fill density

### Recursive Triangulation
Start with a triangle. Subdivide by connecting edge midpoints (producing 4 child triangles).
Optionally displace midpoints for fractal terrain effects.

**Parameters**: depth, displacement amplitude, displacement decay per level

## Notable Artists & Works

- **Piet Mondrian** — the inspiration, *Composition with Red, Blue, and Yellow* (1930)
- **Georg Nees** — early computer art pioneer, *Schotter* (1968), recursive displacement
- **Vera Molnár** — systematic exploration of geometric variation and randomness
- **William Kolomyjec** — early algorithmic subdivision art
- **Manfred Mohr** — hypercube projections and systematic geometric decomposition

## p5.js Implementation Notes

- Render mode: **svg** (rectangles, lines, polygons — all vector-native)
- Implement as recursive functions. Pass the p5 instance through.
- Borders/gaps: draw slightly smaller than the allocated space, leaving gaps that become
  the composition's grid lines. Use `rectMode(CORNER)`.
- Color palettes: define an array of colors, pick randomly or by depth level.
  Deeper subdivisions → lighter/darker variants creates visual hierarchy.
- Animation: subdivide one level per frame for a "building" animation effect.
  Track the subdivision tree and expand leaves over time.
- For circle packing: maintain a list of placed circles, check intersection before placing.
  Use spatial hashing for performance with many circles.
- Random seed: use `p.randomSeed()` for reproducible compositions. Expose seed as parameter.
