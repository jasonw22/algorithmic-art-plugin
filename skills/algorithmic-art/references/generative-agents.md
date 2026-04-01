# Generative Agents & Typography

## Philosophy

Agent-based generative art embodies **autonomous behavior** — individual entities following simple
rules that, in aggregate, produce complex collective phenomena. This mirrors nature's own design
strategy: flocking birds, schooling fish, and swarming ants all exhibit emergent order without
central control.

John Maeda's work at MIT Media Lab explored how **computation transforms typography and graphic
design** — not just as a tool but as a creative medium. Casey Reas and Ben Fry, through Processing,
democratized the idea that **code is a material for making art**, like paint or clay.

The insight: when you give simple agents simple desires (seek, flee, align, wander), beauty
emerges from their social interactions, not from any top-down composition.

## Key Algorithms

### Boids (Reynolds Flocking)
Three rules per agent:
1. **Separation**: steer away from nearby neighbors
2. **Alignment**: steer toward average heading of nearby neighbors
3. **Cohesion**: steer toward average position of nearby neighbors

Each rule produces a steering force. Weight and sum them.

```
separation_force = weighted sum of (away from each neighbor within separation_radius)
alignment_force  = average_velocity_of_neighbors - self.velocity
cohesion_force   = average_position_of_neighbors - self.position
acceleration     = w1 * separation + w2 * alignment + w3 * cohesion
```

**Parameters**: separation/alignment/cohesion weights, perception radius, max speed, max force,
agent count, trail rendering

### Steering Behaviors (Reynolds)
Building blocks for autonomous agents:
- **Seek**: steer toward target
- **Flee**: steer away from target
- **Arrive**: seek but decelerate near target
- **Wander**: steer toward a point on a circle projected ahead, jittered each frame
- **Obstacle avoidance**: project ahead, steer away from obstacles
- **Path following**: steer toward nearest point on a path, with look-ahead

Combine these for complex behavior without complex code.

**Parameters**: behavior weights, wander circle radius/distance, max speed, force limits

### Autonomous Agents (Reas-style)
Simpler than boids: agents with position, velocity, and a few behavioral rules.
Draw their trails. The art emerges from the accumulated paths.

Casey Reas' *Process* series: agents follow curves defined by mathematical functions,
interact through proximity, and leave ink-like traces.

**Parameters**: agent count, speed, rule set, trail opacity, interaction radius

### Generative Typography
Algorithms applied to letterforms:
- **Particle text**: place particles along glyph outlines, apply physics
- **Growth text**: L-system or diffusion-limited aggregation growing from letter shapes
- **Displacement text**: warp letter geometry with noise or flow fields
- **Kinetic typography**: animate letter properties (position, scale, rotation) over time

For glyph outlines in p5.js, use `p.textToPoints()` from p5.Font to get point arrays.

**Parameters**: font, text content, point density, force/displacement amounts, animation speed

### Diffusion-Limited Aggregation (DLA)
Random walkers stick to a growing crystal structure on contact. Produces branching,
organic, coral-like forms. Can be seeded from a point, line, or text outline.

**Parameters**: particle count, sticking probability, seed shape, max cluster size

## Notable Artists & Works

- **Craig Reynolds** — invented boids (1986), foundational flocking simulation
- **Casey Reas** — *Process* series, co-creator of Processing
- **Ben Fry** — data-driven generative art, co-creator of Processing
- **John Maeda** — *Design by Numbers*, *Maeda@Media*, computational design pioneer
- **LIA** — real-time generative audiovisual performances with agent systems
- **Marius Watz** — geometric agent-based generative sculptures

## p5.js Implementation Notes

- Render mode: depends on visual style.
  - Particle trails with transparency → **canvas** (semi-transparent overdraw)
  - Clean line trails → **svg** (store paths as polylines)
- Agent class: store position (p5.Vector), velocity, acceleration, trail history.
  `applyForce(f)`, `update()`, `display()` pattern.
- Use `p.createVector()` for all vector math. `p5.Vector.sub()`, `.normalize()`, `.limit()`.
- Trail rendering: store last N positions. Draw as connected line segments or fading points.
- `textToPoints()`: requires loading a font with `p.loadFont()` in `preload`. Use OpenType
  fonts (.otf/.ttf). Returns array of `{x, y}` objects along glyph outlines.
- For DLA: maintain a grid (occupied/empty) and a set of walkers. Each frame, move walkers
  randomly and check adjacency to occupied cells. Use spatial indexing for performance.
- Performance: 100-500 agents with trails is comfortable. For 1000+, skip rendering every
  other frame or reduce trail length.

## nannou Implementation Notes

**Steering behaviors** — The nannou Nature of Code examples implement Reynolds' steering
behaviors with a clean Vehicle struct pattern:

```rust
struct Vehicle {
    position: Vec2,
    velocity: Vec2,
    acceleration: Vec2,
    max_speed: f32,
    max_force: f32,
    history: VecDeque<Vec2>,  // trail buffer
}

impl Vehicle {
    fn apply_force(&mut self, force: Vec2) {
        self.acceleration += force;  // add mass division here if needed
    }

    fn seek(&mut self, target: Vec2) {
        let desired = (target - self.position).normalize() * self.max_speed;
        let steer = (desired - self.velocity).clamp_length_max(self.max_force);
        self.apply_force(steer);
    }

    fn update(&mut self) {
        self.velocity = (self.velocity + self.acceleration).clamp_length_max(self.max_speed);
        self.position += self.velocity;
        self.acceleration *= 0.0;
        // Trail history
        self.history.push_back(self.position);
        if self.history.len() > 100 { self.history.pop_front(); }
    }
}
```

**Trail rendering** — Use `VecDeque<Vec2>` for position history. Draw as a polyline or as
individually colored points (fade alpha based on age):

```rust
if vehicle.history.len() > 1 {
    let vertices = vehicle.history.iter().map(|v| (pt2(v.x, v.y), srgba(0.0, 0.0, 0.0, 1.0)));
    draw.polyline().weight(1.0).points_colored(vertices);
}
```

**Particle systems with forces** — The Nature of Code examples (`chp_04_systems/`) demonstrate
the classic emitter + particle + force pattern in nannou:
- `ParticleSystem` struct holds `Vec<Particle>` and an origin `Point2`
- `apply_force(f: Vec2)` broadcasts a force to all particles
- `apply_repeller(r: &Repeller)` computes inverse-square repulsion per particle
- Update loop iterates in reverse for safe removal: `for i in (0..len).rev() { ... remove(i) }`
- Particles have `life_span: f32` decremented each frame; draw with `rgba(r, g, b, life_span / 255.0)`

**Organic growth with quadtree spatial indexing** — MacTuitui's `tree.rs` example demonstrates
space-colonization growth: circular "Things" branch outward from a root, using a quadtree
for efficient collision detection. Each Thing tracks parent/children indices, energy propagates
from root to leaves, and new branches spawn at random angles biased by parent direction.
This pattern is excellent for generative tree/coral/root structures.

**Agent count**: nannou in release mode handles 2000+ agents with per-frame draw calls
comfortably. For 10,000+ agents, consider the overlay/trail technique rather than drawing
all trail history each frame.
