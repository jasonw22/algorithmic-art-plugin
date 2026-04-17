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
| *Tracing the Line* | Generative Hut (ed.) | 100-artist anthology of contemporary pen-plotter and drawing-machine work. Useful reference for how hybrid digital/analog practitioners present and document physical output |
| *Meridian* | Matt DesLauriers | Long-form generative artwork + printed catalog (2021–2022). Pairs a code release with a physical book — canonical template for the "code + printed object" format |

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
- **Codrops** (tympanus.net/codrops) — Deep technical tutorials on creative web rendering. Their 2025 WebGPU review surveys the current state of browser GPU compute for creative coding
- **Generative Hut** (generativehut.com) — Publications, interviews, and community for generative art with a strong plotter/hybrid focus. Publisher of *Tracing the Line*
- **Pen Plotter Artwork** (penplotterartwork.com) — Artist profiles and plotter-specific tutorials
- **UUNA TEK** (uunatek.com) — Drawing-machine manufacturer that curates detailed lists of contemporary plotter artists and techniques
- **Dirt Alley Design** (dirtalleydesign.com) — Michelle Chandra's blog with very practical plotter tutorials: pen selection, watercolor-with-plotter, riso workflows, plotter-to-print pipelines
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
- **Frontend Masters — Creative Coding with Canvas & WebGL** (Matt DesLauriers) — Browser-based generative art, code-as-tool framing
- **Frontend Masters — Color Science for Designers & Developers** (Matt DesLauriers) — Perceptual color spaces, palette design — direct companion to `color-science.md`
- **fxhash explainers** (Daniel Catt, ciphrd) — Approachable on-chain generative art tutorials published through fxhash's community channels
- **Matt DesLauriers — open workshop repos** (github.com/mattdesl) — `workshop-generative-art` and `workshop-p5-intro` are de-facto starting points used by many educators

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

## Contemporary Practitioners (2020s)

A current roster for the art-historical-context step of the workflow. Named artists whose
ongoing work is worth citing when explaining a piece's lineage or choosing influences for
a new piece.

### Long-form / on-chain generative

- **Tyler Hobbs** (tylerxhobbs.com) — Flow fields, *Fidenza*, long-form generative essays; effectively named the "long-form generative art" category
- **Matt DesLauriers** (mattdesl.com) — *Meridian*, *Pattern Language* (2024 — Åkle weaving–inspired installation), *The Sferic Project* (2023–ongoing — Earth's natural radio/atmospherics), *Sierra* (2024 — print + digital with Avant Arte); prolific writer and workshop author
- **Zancan** — Procedural botanical work on Tezos; deep fidelity in simulated plant systems
- **William Mapan** — Long-form on Art Blocks / fxhash; painterly generative systems
- **Melissa Wiederrecht** — On-chain generative work; notable writer on the field
- **ciphrd** (Baptiste Crespy) — Founder of fxhash; writes deep process pieces on the fxhash blog and Medium
- **Licia He** (liciahe.com) — Pen-plot paintings created with custom code driving an AxiDraw as a painting instrument; explicit treatment of the plotter as performer

### Plotter / hybrid digital-analog

- **LB Allix** — Daily plotter output; consistent practice as a discipline
- **CMD_DRAW** — Plotter combined with 3D and motion work
- **Medusa Gen** — Large-format plotter abstraction
- **Barry Spencer** — Generative typography paired with hand-crafted ceramics
- **Reuben** — Brush pens on large-format plotters for painterly, human-feeling work
- **Mechanic Art** — Plotter-based brush painting with watercolor/acrylic; custom software treating the plotter as an artistic instrument rather than the art itself ("plotter as instrument, not subject")
- **Michelle Chandra — Dirt Alley Design** (dirtalleydesign.com) — Draws each print to order with a robotic drawing machine; designs inspired by patterns in the natural world. Commercial model for small-scale plotter practice

### WebGPU / real-time technical

- **Hector Arellano** — Pioneering WebGPU fluid-simulation work on the web
- **matsuoka-601** — WebGPU fluid simulation at the cutting edge of what the browser can sustain (300k particles, real-time MLS-MPM)
- **Mustafa Ali** — Node-based shader editors built on Three.js's WebGPU/TSL path
- **Absulit** — Author of the POINTS library, a wrapper around WebGPU shader setup aimed at creative coders

### Educators worth following (for workshop design)

- **Daniel Shiffman** (thecodingtrain.com) — Still the gateway for most people coming from zero; *The Nature of Code*
- **Daniel Catt** (revdancatt) — Generative artist publishing approachable explainers on fxhash
- **Tyler Hobbs** — Essays rather than courses; "Flow Fields" and "Long-Form Generative Art" writeups have onboarded more people than many formal programs
- **Matt DesLauriers** — Frontend Masters + open workshop repos; positions for both artists-curious-about-code and coders-curious-about-art
- **Generative Hut / Pen Plotter Artwork** — Community-scale teaching; models for building an audience around a hybrid practice

## Adjacent skills

For techniques that bridge into AI-adjacent territory — radiance fields / NeRF, differentiable
rendering, neural cellular automata, Slang (Khronos) shading language for neural graphics,
CLIP-guided generation, diffusion-seeded simulations — see the sibling
`generative-ai-art` skill. That skill handles compositions where an AI model is a first-class
participant in the generation loop; this skill handles algorithmic art where math and code
alone produce the image.

---

*When adding new techniques to this skill, also add relevant sources here.*
