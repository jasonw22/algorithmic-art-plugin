# Algorithmic Art Plugin for Claude Code

Create algorithmic art as standalone HTML files using p5.js, with interactive parameter controls and both bitmap (PNG) and vector (SVG) export.

## Installation

```bash
/plugin install <your-github-username>/algorithmic-art-plugin
```

Or for local development:

```bash
claude --plugin-dir /path/to/algorithmic-art-plugin
```

## Usage

Invoke the skill in Claude Code:

```
/algorithmic-art:algorithmic-art create a Mandelbrot set explorer
```

Or simply describe what you want:

> "Make me a flow field visualization with particle trails"

## Supported Techniques

| Family | Description |
|--------|-------------|
| Fractals & L-systems | Mandelbrot, Julia sets, recursive growth, tree grammars |
| Reaction-diffusion | Turing patterns, Gray-Scott model, morphogenesis |
| Flow fields & noise | Perlin/simplex noise, curl noise, particle traces |
| Cellular automata | Wolfram rules, Conway's Game of Life, emergence |
| Strange attractors | Lorenz, Clifford, de Jong, deterministic chaos |
| Tiling & tessellation | Penrose, Truchet, aperiodic patterns, Islamic geometry |
| Recursion & subdivision | Mondrian-style, quadtree, space partitioning |
| Generative agents | Flocking, steering behaviors, autonomous agents |

## Features

- Full-viewport canvas with responsive resizing
- Interactive sidebar with parameter controls (sliders, dropdowns, color pickers, toggles)
- Seed system for reproducible outputs
- PNG and SVG export
- Extensible reference library — learns new techniques automatically

## License

MIT
