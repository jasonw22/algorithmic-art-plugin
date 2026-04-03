# nannou — 3D Rendering Reference

## Overview

nannou supports 3D rendering through its wgpu backend. While nannou's `Draw` API is
primarily 2D, 3D work is achieved through:

1. **Draw API with 3D transforms** — simple 3D positioning of 2D primitives (lines, meshes)
2. **Custom wgpu pipelines** — full GPU access for custom shaders, compute, advanced 3D
3. **Camera control** — manual perspective/orbit camera implementation

This reference covers 3D techniques in nannou. See `nannou.md` for the base nannou API
(2D drawing, color, noise, keyboard, export).

## 3D with the Draw API

nannou's `Draw` supports 3D coordinates. Shapes can be positioned anywhere in 3D space,
and a perspective camera transforms them to screen:

```rust
use nannou::prelude::*;

struct Model {
    camera_angle: f32,
    camera_distance: f32,
    camera_height: f32,
    seed: u32,
    paused: bool,
}

fn model(app: &App) -> Model {
    app.new_window()
        .size(1200, 800)
        .title("3D Piece")
        .view(view)
        .key_pressed(key_pressed)
        .build()
        .unwrap();

    println!("Controls:");
    println!("  Left/Right — orbit camera");
    println!("  Up/Down    — camera height");
    println!("  +/-        — zoom in/out");
    println!("  S          — save PNG");
    println!("  N/P        — next/prev seed");
    println!("  Space      — pause/resume");

    Model {
        camera_angle: 0.0,
        camera_distance: 8.0,
        camera_height: 4.0,
        seed: 42,
        paused: false,
    }
}

fn update(app: &App, model: &mut Model, _update: Update) {
    if model.paused { return; }
    // Animate camera orbit
    model.camera_angle += 0.005;
}

fn view(app: &App, model: &Model, frame: Frame) {
    let draw = app.draw();
    let win = app.window_rect();
    draw.background().color(BLACK);

    // ── Perspective camera transform ──
    // Build a look-at view matrix and perspective projection
    let eye = vec3(
        model.camera_angle.cos() * model.camera_distance,
        model.camera_height,
        model.camera_angle.sin() * model.camera_distance,
    );
    let target = vec3(0.0, 0.0, 0.0);
    let up = vec3(0.0, 1.0, 0.0);

    let aspect = win.w() / win.h();
    let fov = std::f32::consts::PI / 3.0;  // 60 degrees

    // Apply perspective transform to the draw instance
    let draw = draw
        .scale_y(-1.0)  // flip y for screen coordinates
        .transform(perspective_view_matrix(eye, target, up, fov, aspect, 0.1, 100.0));

    // Now draw in 3D world space
    draw.ellipse()
        .x_y_z(0.0, 0.0, 0.0)
        .radius(0.5)
        .color(RED);

    // Draw a grid on the xz plane
    for i in -10..=10 {
        let f = i as f32;
        draw.line()
            .start(pt3(f, 0.0, -10.0))
            .end(pt3(f, 0.0, 10.0))
            .weight(1.0)
            .color(rgba(1.0, 1.0, 1.0, 0.1));
        draw.line()
            .start(pt3(-10.0, 0.0, f))
            .end(pt3(10.0, 0.0, f))
            .weight(1.0)
            .color(rgba(1.0, 1.0, 1.0, 0.1));
    }

    draw.to_frame(app, &frame).unwrap();
}

fn key_pressed(app: &App, model: &mut Model, key: Key) {
    match key {
        Key::Left => model.camera_angle -= 0.1,
        Key::Right => model.camera_angle += 0.1,
        Key::Up => model.camera_height += 0.5,
        Key::Down => model.camera_height -= 0.5,
        Key::Equals => model.camera_distance = (model.camera_distance - 0.5).max(1.0),
        Key::Minus => model.camera_distance += 0.5,
        Key::S => app.main_window().capture_frame(format!(
            "{}_{}.png", app.exe_name().unwrap(), app.elapsed_frames()
        )),
        Key::N => { model.seed += 1; }
        Key::P => { model.seed = model.seed.saturating_sub(1); }
        Key::Space => { model.paused = !model.paused; }
        _ => {}
    }
}
```

## Perspective View Matrix

nannou doesn't provide a built-in perspective camera, so build one manually.
Add this helper to your sketch:

```rust
/// Build a combined perspective projection × look-at view matrix.
/// Returns a Mat4 suitable for `draw.transform()`.
fn perspective_view_matrix(
    eye: Vec3, target: Vec3, up: Vec3,
    fov_y: f32, aspect: f32, near: f32, far: f32,
) -> Mat4 {
    let view = Mat4::look_at_rh(eye, target, up);
    let proj = Mat4::perspective_rh(fov_y, aspect, near, far);
    proj * view
}
```

This requires `glam` (nannou re-exports it via `nannou::prelude::*` which includes `Vec3`,
`Mat4`, `vec3`, `pt3`).

## 3D Drawing Primitives

nannou's `Draw` API positions 2D shapes in 3D space:

```rust
// Position any shape with x_y_z
draw.ellipse().x_y_z(1.0, 2.0, 3.0).radius(0.5).color(RED);
draw.rect().x_y_z(0.0, 0.0, -1.0).w_h(2.0, 2.0).color(BLUE);

// 3D lines
draw.line()
    .start(pt3(0.0, 0.0, 0.0))
    .end(pt3(3.0, 2.0, 1.0))
    .weight(2.0)
    .color(WHITE);

// 3D polyline (open path through 3D points)
let points_3d: Vec<Vec3> = (0..100).map(|i| {
    let t = i as f32 * 0.1;
    vec3(t.cos() * 2.0, t * 0.5, t.sin() * 2.0)
}).collect();
draw.polyline().weight(2.0).points(points_3d).color(CYAN);

// 3D mesh (triangle fan, custom vertices)
draw.mesh()
    .points(vec![pt3(0.0, 1.0, 0.0), pt3(-1.0, -1.0, 0.5), pt3(1.0, -1.0, -0.5)])
    .color(GREEN);

// Per-vertex coloring
let colored_pts: Vec<(Vec3, Hsla)> = (0..360).map(|i| {
    let a = deg_to_rad(i as f32);
    let r = 3.0;
    (vec3(a.cos() * r, 0.0, a.sin() * r), hsla(i as f32 / 360.0, 0.8, 0.6, 1.0))
}).collect();
draw.polyline().weight(2.0).points_colored(colored_pts);
```

## 3D Rotation

Apply rotations to draw calls using `.rotate_x()`, `.rotate_y()`, `.rotate_z()`:

```rust
draw.rect()
    .x_y_z(0.0, 0.0, 0.0)
    .w_h(2.0, 2.0)
    .rotate_x(app.time * 0.5)
    .rotate_y(app.time * 0.3)
    .color(STEELBLUE);
```

## Particle Systems in 3D

```rust
struct Particle {
    pos: Vec3,
    vel: Vec3,
    color: Hsla,
    life: f32,
}

struct Model {
    particles: Vec<Particle>,
    // ... camera fields, seed, etc.
}

fn update(app: &App, model: &mut Model, _update: Update) {
    let dt = 1.0 / 60.0;
    for p in &mut model.particles {
        // Apply forces (noise field, gravity, attraction)
        let noise_force = vec3(
            perlin.get([p.pos.x as f64 * 0.01, p.pos.y as f64 * 0.01, p.pos.z as f64 * 0.01]) as f32,
            perlin.get([p.pos.y as f64 * 0.01, p.pos.z as f64 * 0.01, p.pos.x as f64 * 0.01]) as f32,
            perlin.get([p.pos.z as f64 * 0.01, p.pos.x as f64 * 0.01, p.pos.y as f64 * 0.01]) as f32,
        );
        p.vel += noise_force * 0.1;
        p.vel *= 0.99;  // damping
        p.pos += p.vel * dt;
        p.life -= dt * 0.01;
    }
    model.particles.retain(|p| p.life > 0.0);
}

fn view(app: &App, model: &Model, frame: Frame) {
    let draw = app.draw();
    // ... camera setup ...
    for p in &model.particles {
        draw.ellipse()
            .x_y_z(p.pos.x, p.pos.y, p.pos.z)
            .radius(0.05)
            .color(p.color);
    }
    draw.to_frame(app, &frame).unwrap();
}
```

## 3D Strange Attractors

All the attractor formulas from `strange-attractors.md` extend to 3D naturally:

```rust
// Lorenz attractor
fn lorenz_step(p: Vec3, sigma: f32, rho: f32, beta: f32, dt: f32) -> Vec3 {
    let dx = sigma * (p.y - p.x);
    let dy = p.x * (rho - p.z) - p.y;
    let dz = p.x * p.y - beta * p.z;
    p + vec3(dx, dy, dz) * dt
}

// Aizawa attractor
fn aizawa_step(p: Vec3, a: f32, b: f32, c: f32, d: f32, e: f32, f: f32, dt: f32) -> Vec3 {
    let dx = (p.z - b) * p.x - d * p.y;
    let dy = d * p.x + (p.z - b) * p.y;
    let dz = c + a * p.z - p.z.powi(3) / 3.0
           - (p.x * p.x + p.y * p.y) * (1.0 + e * p.z)
           + f * p.z * p.x.powi(3);
    p + vec3(dx, dy, dz) * dt
}

// Thomas attractor (elegant symmetric)
fn thomas_step(p: Vec3, b: f32, dt: f32) -> Vec3 {
    let dx = p.y.sin() - b * p.x;
    let dy = p.z.sin() - b * p.y;
    let dz = p.x.sin() - b * p.z;
    p + vec3(dx, dy, dz) * dt
}
```

Render attractors as colored polylines or accumulate points and render as a point cloud.

## 3D Flow Fields

Extend 2D curl noise to 3D:

```rust
use nannou::noise::{NoiseFn, Perlin, Seedable};

fn curl_3d(noise: &Perlin, p: Vec3, scale: f64, eps: f64) -> Vec3 {
    let x = p.x as f64 * scale;
    let y = p.y as f64 * scale;
    let z = p.z as f64 * scale;

    // Partial derivatives via central differences
    let dny_dz = (noise.get([x, y, z + eps]) - noise.get([x, y, z - eps])) / (2.0 * eps);
    let dnz_dy = (noise.get([x + 100.0, y + eps, z]) - noise.get([x + 100.0, y - eps, z])) / (2.0 * eps);
    let dnx_dz = (noise.get([x + 200.0, y, z + eps]) - noise.get([x + 200.0, y, z - eps])) / (2.0 * eps);
    let dnz_dx = (noise.get([x + 100.0, y, z + eps]) - noise.get([x + 100.0, y, z - eps])) / (2.0 * eps);
    let dny_dx = (noise.get([x, y + eps, z]) - noise.get([x, y - eps, z])) / (2.0 * eps);
    let dnx_dy = (noise.get([x + 200.0, y + eps, z]) - noise.get([x + 200.0, y - eps, z])) / (2.0 * eps);

    vec3(
        (dny_dz - dnz_dy) as f32,
        (dnz_dx - dnx_dz) as f32,
        (dnx_dy - dny_dx) as f32,
    )
}
```

## Custom wgpu Pipelines (Advanced)

For advanced 3D rendering (custom shaders, compute, real mesh rendering with lighting),
use nannou's wgpu API directly:

```rust
use nannou::wgpu;

struct Model {
    render_pipeline: wgpu::RenderPipeline,
    vertex_buffer: wgpu::Buffer,
    uniform_buffer: wgpu::Buffer,
    bind_group: wgpu::BindGroup,
}

fn model(app: &App) -> Model {
    let window = app.main_window();
    let device = window.device();

    // Compile shaders
    let shader = device.create_shader_module(wgpu::ShaderModuleDescriptor {
        label: Some("shader"),
        source: wgpu::ShaderSource::Wgsl(include_str!("shader.wgsl").into()),
    });

    // Create pipeline, buffers, bind groups...
    // See nannou wgpu examples for complete patterns
    todo!()
}
```

### WGSL Shader Example (vertex + fragment)

Create `src/shader.wgsl`:

```wgsl
struct Uniforms {
    mvp: mat4x4<f32>,
    time: f32,
    _pad: vec3<f32>,
};

@group(0) @binding(0) var<uniform> uniforms: Uniforms;

struct VertexInput {
    @location(0) position: vec3<f32>,
    @location(1) normal: vec3<f32>,
    @location(2) color: vec3<f32>,
};

struct VertexOutput {
    @builtin(position) clip_position: vec4<f32>,
    @location(0) world_normal: vec3<f32>,
    @location(1) color: vec3<f32>,
};

@vertex
fn vs_main(in: VertexInput) -> VertexOutput {
    var out: VertexOutput;
    out.clip_position = uniforms.mvp * vec4<f32>(in.position, 1.0);
    out.world_normal = in.normal;
    out.color = in.color;
    return out;
}

@fragment
fn fs_main(in: VertexOutput) -> @location(0) vec4<f32> {
    let light_dir = normalize(vec3<f32>(1.0, 1.0, 1.0));
    let diffuse = max(dot(in.world_normal, light_dir), 0.0);
    let ambient = 0.15;
    let lit = in.color * (ambient + diffuse * 0.85);
    return vec4<f32>(lit, 1.0);
}
```

### Dependencies for Advanced 3D

```toml
[dependencies]
nannou = "0.19"
glam = "0.24"        # mat4, vec3, quaternion math
noise = "0.8"        # additional noise types
bytemuck = { version = "1", features = ["derive"] }  # for uniform buffer casting
```

## 3D Generative Sculpture Techniques

### L-Systems in 3D (branching trees)
```rust
struct Turtle {
    pos: Vec3,
    dir: Vec3,
    up: Vec3,
    width: f32,
}

impl Turtle {
    fn forward(&mut self, dist: f32) -> (Vec3, Vec3) {
        let start = self.pos;
        self.pos += self.dir * dist;
        (start, self.pos)
    }

    fn yaw(&mut self, angle: f32) {
        let rot = Mat3::from_axis_angle(self.up, angle);
        self.dir = rot * self.dir;
    }

    fn pitch(&mut self, angle: f32) {
        let right = self.dir.cross(self.up).normalize();
        let rot = Mat3::from_axis_angle(right, angle);
        self.dir = rot * self.dir;
        self.up = rot * self.up;
    }

    fn roll(&mut self, angle: f32) {
        let rot = Mat3::from_axis_angle(self.dir, angle);
        self.up = rot * self.up;
    }
}
```

### Recursive 3D Subdivision
```rust
fn subdivide_tetrahedron(vertices: &[Vec3; 4], depth: u32, result: &mut Vec<[Vec3; 4]>) {
    if depth == 0 {
        result.push(*vertices);
        return;
    }
    // Compute midpoints of all 6 edges
    let m01 = (vertices[0] + vertices[1]) * 0.5;
    let m02 = (vertices[0] + vertices[2]) * 0.5;
    let m03 = (vertices[0] + vertices[3]) * 0.5;
    let m12 = (vertices[1] + vertices[2]) * 0.5;
    let m13 = (vertices[1] + vertices[3]) * 0.5;
    let m23 = (vertices[2] + vertices[3]) * 0.5;

    // Recurse into sub-tetrahedra (Sierpinski-like)
    subdivide_tetrahedron(&[vertices[0], m01, m02, m03], depth - 1, result);
    subdivide_tetrahedron(&[m01, vertices[1], m12, m13], depth - 1, result);
    subdivide_tetrahedron(&[m02, m12, vertices[2], m23], depth - 1, result);
    subdivide_tetrahedron(&[m03, m13, m23, vertices[3]], depth - 1, result);
}
```

### Mesh Deformation with Noise
```rust
fn deform_sphere(points: &mut Vec<Vec3>, noise: &Perlin, time: f32, scale: f64, amount: f32) {
    for p in points.iter_mut() {
        let n = p.normalize();
        let noise_val = noise.get([
            n.x as f64 * scale,
            n.y as f64 * scale,
            n.z as f64 * scale + time as f64 * 0.3
        ]) as f32;
        let base_radius = p.length();
        *p = n * (base_radius + noise_val * amount);
    }
}
```

## Trail / Accumulation in 3D

The 2D overlay technique (see `nannou.md`) works in 3D too — draw a semi-transparent
background-colored rect each frame instead of clearing fully:

```rust
fn view(app: &App, model: &Model, frame: Frame) {
    let draw = app.draw();
    let win = app.window_rect();

    if frame.nth() == 0 {
        draw.background().color(BLACK);
    } else {
        draw.rect().wh(win.wh()).color(srgba(0, 0, 0, 15));  // low-alpha overlay
    }

    // Draw 3D elements on top (with perspective transform)
    // ...

    draw.to_frame(app, &frame).unwrap();
}
```

## Performance Tips for 3D

- Use `cargo run --release` — critical for 3D; debug builds are ~10x slower
- Batch draw calls: one `draw.polyline()` with 10k points > 10k `draw.line()` calls
- For 100k+ objects, use custom wgpu pipelines with instanced rendering
- The `Draw` API batches internally, but complex scenes may benefit from manual wgpu
- Use `nannou::noise::Perlin` over `Simplex` — Perlin is faster for most use cases
- Cache computed geometry in the Model rather than recomputing each frame

## Key Differences from Three.js 3D

| Concept | Three.js (template-3d) | nannou 3D |
|---------|----------------------|-----------|
| Camera | Built-in PerspectiveCamera + OrbitControls | Manual matrix construction |
| Lighting | Scene graph lights (ambient, directional, point) | Manual in shader or fake with color |
| Meshes | BufferGeometry + Material system | Draw API or custom wgpu pipeline |
| Shaders | GLSL via ShaderMaterial | WGSL via wgpu pipeline |
| Controls | Mouse (orbit/pan/zoom) | Keyboard |
| Performance sweet spot | 10k–100k objects | 100k+ with custom pipeline |
| Best for | Interactive 3D scenes, lighting, materials | Raw particle systems, attractor paths, line art |
