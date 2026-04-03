# Algorithmic Art — Learning Resources

A curated list of sources for learning the algorithms, history, and philosophy behind
algorithmic and generative art. Organized by type.

## Books

| Title | Author(s) | Focus |
|-------|-----------|-------|
| *The Nature of Code* | Daniel Shiffman | Physics simulation, agents, fractals, CA — the essential creative coding textbook |
| *Generative Art* | Matt Pearson | p5.js/Processing-based introduction to generative art concepts |
| *The Fractal Geometry of Nature* | Benoit Mandelbrot | Foundational text on fractals and natural form |
| *A New Kind of Science* | Stephen Wolfram | Cellular automata as a computational paradigm |
| *The Algorithmic Beauty of Plants* | P. Prusinkiewicz & A. Lindenmayer | L-systems and botanical modeling — free PDF online |
| *Form+Code* | Casey Reas et al. | Design, art, and architecture through computation |
| *Design by Numbers* | John Maeda | Computational approach to design, precursor to Processing |
| *Chaos: Making a New Science* | James Gleick | Accessible introduction to chaos theory and strange attractors |
| *Vehicles: Experiments in Synthetic Psychology* | Valentino Braitenberg | Simple agents, complex behavior — philosophical foundation for agent art |
| *10 PRINT CHR$(205.5+RND(1)); : GOTO 10* | Montfort et al. | Deep analysis of one line of generative code — cultural and technical |
| *Generative Design* | Groß, Bohnacker, Laub, Lazzeroni | Comprehensive creative coding reference — color, shape, type, image, noise, oscillation. Ported to nannou (see below) |

## Papers

| Title | Author(s) | Year | Topic |
|-------|-----------|------|-------|
| "The Chemical Basis of Morphogenesis" | Alan Turing | 1952 | Reaction-diffusion, pattern formation |
| "Deterministic Nonperiodic Flow" | Edward Lorenz | 1963 | Strange attractors, chaos |
| "Flocks, Herds, and Schools" | Craig Reynolds | 1987 | Boids algorithm |
| "An Image Synthesizer" | Ken Perlin | 1985 | Perlin noise |
| "Simplex Noise Demystified" | Stefan Gustavson | 2005 | Clear explanation of simplex noise |
| "Penrose Tiling" | Roger Penrose | 1974 | Aperiodic tiling |
| "Multi-Scale Turing Patterns" | Jonathan McCabe | 2010 | Layered reaction-diffusion |
| "Protrusion: Magnetic Fluid Sculpture" | Sachiko Kodama | 2008 | Ferrofluid art, magnetic field visualization |
| "Anisotropic Diffusion in Image Processing" | Weickert | 1998 | Directional diffusion tensors, foundational for field-coupled RD |

## Websites & Online Resources

- **The Coding Train** (thecodingtrain.com) — Daniel Shiffman's video tutorials, the single best
  resource for learning creative coding with p5.js
- **OpenProcessing** (openprocessing.org) — Community gallery of Processing/p5.js sketches with source code
- **Shadertoy** (shadertoy.com) — GPU shader gallery, many algorithmic art techniques in GLSL
- **Tyler Hobbs' Essays** (tylerxhobbs.com/essays) — Deep writing on flow fields, generative aesthetics, and process
- **Inconvergent** (inconvergent.net) — Anders Hoff's explorations of generative algorithms with thorough writeups
- **Generative Artistry** (generativeartistry.com) — Step-by-step tutorials recreating classic generative artworks
- **The Book of Shaders** (thebookofshaders.com) — GPU-side procedural graphics (noise, patterns, SDF)
- **Algorithmic Art** (algorithmicart.net) — Historical archive of early computer art
- **Paul Bourke's Pages** (paulbourke.net) — Encyclopedic reference for fractals, attractors, and geometry
- **nannou Examples** (github.com/nannou-org/nannou/tree/master/examples) — Official nannou
  examples covering draw API, textures, blend modes, hi-res capture, wgpu compute shaders,
  and templates. Essential reference for nannou-mode sketches.
- **nannou Nature of Code** (github.com/nannou-org/nannou/tree/master/nature_of_code) — Rust
  ports of Daniel Shiffman's *Nature of Code* examples: vectors, forces, oscillation, particle
  systems, agents/steering, cellular automata, fractals, genetic algorithms. The canonical
  reference for physics-based generative art in nannou.
- **nannou Generative Design** (github.com/nannou-org/nannou/tree/master/generative_design) —
  Rust ports of *Generative Design* book examples: color, shape, random/noise, oscillation
  figures, dynamic data structures, image processing. Includes the definitive nannou
  flow-field agent pattern (`m_1_5_04.rs`) and Lissajous figure generation (`m_2_5_02.rs`).

## Courses

- **The Nature of Code** (Kadenze / YouTube) — Daniel Shiffman
- **Creative Coding with p5.js** (Domestika) — Various instructors
- **Generative Art & Computational Creativity** (Coursera/edX) — Various universities

## Communities

- **r/generative** (Reddit) — Active generative art community
- **r/creativecoding** (Reddit) — Creative coding discussion
- **Processing Forum** (discourse.processing.org) — Official Processing/p5.js community
- **fxhash** (fxhash.xyz) — Generative art NFT platform with open-source sketches

## Toolkits & Ecosystems

- **thi.ng/umbrella** (github.com/thi-ng/umbrella) — Karsten Schmidt's monorepo of 214+
  composable TypeScript packages for functional creative coding. Key art-relevant packages:
  `@thi.ng/geom` (2D/3D geometry), `@thi.ng/color` (16 color spaces, cosine gradients,
  LCH theme generation), `@thi.ng/geom-sdf` (2D signed distance fields with smooth booleans),
  `@thi.ng/geom-tessellate` (9 composable tessellation algorithms), `@thi.ng/shader-ast`
  (write shaders in TypeScript, cross-compile to GLSL/JS), `@thi.ng/shader-ast-stdlib`
  (~230 portable shader functions), `@thi.ng/random` (seedable PRNGs, distributions).
  ~185 example projects demonstrating generative art techniques.
- **thi.ng/genart-api** (github.com/thi-ng/genart-api) — Platform-independent API for
  browser-based generative art. Decouples artworks from presentation platforms (fx(hash),
  EditArt, personal sites) via pluggable adapters. Defines a sophisticated parameter system
  (17 types including Ramp, WeightedChoice, XY, Vector, Image), seedable PRNG architecture,
  state machine, and offline rendering for video export. Valuable as an architecture reference
  for parameter design and platform-independent art.
- **Karsten Schmidt (toxi)** (toxi.co) — Veteran creative coder and generative artist.
  Creator of toxiclibs (Java, used extensively in Processing community), thi.ng (TypeScript/
  Clojure), and author of influential work on functional approaches to computational geometry,
  SDF composition, and perceptual color science.

---

*When adding new techniques to this skill, also add relevant sources here.*
