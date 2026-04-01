# nannou — Rust Creative Coding Framework

## Overview

[nannou](https://nannou.cc) is an open-source creative coding framework for Rust, inspired by
Processing, openFrameworks, and Cinder. It provides a batteries-included environment for
generative art, audiovisual installations, and interactive simulations.

**When to choose nannou over p5.js:**
- High-performance particle systems (100k+ particles)
- GPU-accelerated rendering via wgpu
- Large-scale print-resolution output
- Users who prefer Rust's type system and performance
- Multi-window or installation-oriented work

## Core Architecture

nannou uses a Model-Update-View pattern:

| Function | Purpose | Analogy (p5.js) |
|----------|---------|-----------------|
| `model()` | One-time setup, create window, init state | `setup()` |
| `update()` | Per-frame state mutation | Top of `draw()` before rendering |
| `view()` | Per-frame rendering (immutable borrow of model) | `draw()` |

The separation of update (mutable) and view (immutable) enforces clean architecture.

## Coordinate System

- Origin at **center** of window (not top-left like p5.js)
- **Y-axis points up** (not down like p5.js)
- `app.window_rect()` gives the full window bounds: `.left()`, `.right()`, `.top()`, `.bottom()`, `.w()`, `.h()`

## Drawing API

All drawing goes through the `Draw` struct:

```rust
let draw = app.draw();
draw.background().color(BLACK);

// Shapes
draw.ellipse().x_y(0.0, 0.0).radius(50.0).color(RED);
draw.rect().x_y(100.0, 0.0).w_h(80.0, 60.0).color(BLUE);
draw.line().start(pt2(-100.0, 0.0)).end(pt2(100.0, 0.0)).weight(2.0).color(WHITE);
draw.tri().points(pt2(0.0, 50.0), pt2(-50.0, -50.0), pt2(50.0, -50.0)).color(GREEN);

// Polyline (open path)
let points = (0..100).map(|i| {
    let x = i as f32 * 5.0 - 250.0;
    let y = (x * 0.05).sin() * 100.0;
    pt2(x, y)
});
draw.polyline().weight(2.0).points(points).color(WHITE);

// Polygon (closed, filled)
let vertices = vec![pt2(0.0, 100.0), pt2(-80.0, -50.0), pt2(80.0, -50.0)];
draw.polygon().points(vertices).color(STEELBLUE);

// Colored vertices (per-vertex color)
let colored_points = (0..360).map(|i| {
    let angle = deg_to_rad(i as f32);
    let r = 200.0;
    let point = pt2(angle.cos() * r, angle.sin() * r);
    let color = hsla(i as f32 / 360.0, 0.8, 0.6, 1.0);
    (point, color)
});
draw.polyline().weight(2.0).points_colored(colored_points);

// Text
draw.text("Hello").font_size(24).color(WHITE).x_y(0.0, 0.0);

// Transformations — apply to individual draw calls
draw.ellipse().x_y(0.0, 0.0).radius(50.0).rotate(app.time);

// Commit to frame
draw.to_frame(app, &frame).unwrap();
```

## Color

nannou re-exports the `palette` crate as `nannou::color`. Common usage:

```rust
use nannou::prelude::*;

// Named colors
let c = RED;
let c = STEELBLUE;
let c = BLACK;

// RGBA (0.0–1.0 floats)
let c = rgba(1.0, 0.0, 0.0, 0.5);

// HSLA (hue 0.0–1.0, saturation, lightness, alpha)
let c = hsla(0.6, 0.8, 0.5, 1.0);

// Hex-like construction
let c = Srgba::new(0.2, 0.4, 0.8, 1.0);

// From u8 values
let c = srgba(200u8, 100u8, 50u8, 255u8);

// Lerp between colors
let a = hsla(0.0, 0.8, 0.5, 1.0);
let b = hsla(0.6, 0.8, 0.5, 1.0);
// Use manual interpolation for component lerp
```

## Noise

nannou re-exports the `noise` crate:

```rust
use nannou::noise::{NoiseFn, Perlin, Seedable};

let perlin = Perlin::new().set_seed(42);

// 2D noise — returns f64 in roughly [-1, 1]
let val = perlin.get([x as f64 * 0.01, y as f64 * 0.01]);

// 3D noise (use time as z for animation)
let val = perlin.get([x as f64 * 0.01, y as f64 * 0.01, t as f64 * 0.005]);
```

For curl noise / flow fields, compute the gradient numerically:
```rust
fn curl_2d(noise: &Perlin, x: f64, y: f64, scale: f64, eps: f64) -> (f64, f64) {
    let dx = (noise.get([x, y + eps]) - noise.get([x, y - eps])) / (2.0 * eps);
    let dy = (noise.get([x + eps, y]) - noise.get([x - eps, y])) / (2.0 * eps);
    (-dx * scale, dy * scale)  // rotate 90 degrees for divergence-free field
}
```

## Random

```rust
use nannou::rand::prelude::*;
use nannou::rand::rngs::StdRng;
use nannou::rand::SeedableRng;

// Seeded RNG for reproducibility
let mut rng = StdRng::seed_from_u64(42);
let val: f32 = rng.gen_range(-1.0..1.0);

// nannou convenience (not seeded — uses thread_rng)
let val = random_f32();            // 0.0 to 1.0
let val = random_range(-1.0, 1.0); // min to max
```

## Keyboard Interaction

Use `key_pressed` for discrete events and `app.keys.down` for held-key state:

```rust
fn key_pressed(app: &App, model: &mut Model, key: Key) {
    match key {
        Key::S => {
            // Save frame as PNG
            app.main_window().capture_frame(format!(
                "{}_{}.png",
                app.exe_name().unwrap(),
                app.elapsed_frames()
            ));
        }
        Key::R => { /* reset / randomize seed */ }
        Key::Space => { model.paused = !model.paused; }
        Key::Up => { model.some_param += 0.1; }
        Key::Down => { model.some_param -= 0.1; }
        Key::N => { model.seed += 1; /* reinit */ }
        Key::P => { model.seed = model.seed.saturating_sub(1); /* reinit */ }
        _ => {}
    }
}
```

Print the key bindings at startup so the user knows what's available:
```rust
fn model(app: &App) -> Model {
    // ... window setup ...
    println!("Controls:");
    println!("  S     — save PNG");
    println!("  R     — reset with random seed");
    println!("  N/P   — next/previous seed");
    println!("  Space — pause/resume");
    println!("  Up/Down — adjust parameter");
    // ...
}
```

## Blend Modes

nannou provides blend mode constants for compositing. Apply via `draw.color_blend()`:

```rust
use nannou::prelude::*;

// Available blend modes
let blends = [
    BLEND_NORMAL,
    BLEND_ADD,              // Additive — great for glowing particles, light effects
    BLEND_SUBTRACT,
    BLEND_REVERSE_SUBTRACT,
    BLEND_DARKEST,          // Min blending — moody, shadow-heavy looks
    BLEND_LIGHTEST,         // Max blending — ethereal, blown-out highlights
];

// Apply to a draw instance — all subsequent calls use this blend
let draw = draw.color_blend(BLEND_ADD);
draw.ellipse().x_y(0.0, 0.0).radius(50.0).color(RED);
```

`BLEND_ADD` is especially useful for particle systems and flow fields — overlapping
semi-transparent particles glow brighter where they converge, creating natural light
accumulation effects.

## Trail / Overlay Technique

Instead of clearing the background each frame, draw a semi-transparent rectangle over the
entire window. Previous frames fade gradually, creating motion trails:

```rust
fn view(app: &App, model: &Model, frame: Frame) {
    let draw = app.draw();
    let win = app.window_rect();

    if frame.nth() == 0 {
        draw.background().color(BLACK);
    } else {
        // Overlay with low alpha — lower = longer trails
        draw.rect().wh(win.wh()).hsla(0.0, 0.0, 0.0, 0.08);
    }

    // Draw current frame's elements on top...
}
```

This is the standard technique for flow field particle trails, agent paths, and attractor
visualizations. Adjust the alpha value (0.01–0.15) to control trail length.

## Frame Export

```rust
// Single frame capture
app.main_window().capture_frame("output.png");

// With seed in filename
app.main_window().capture_frame(format!("piece_seed{}_{}.png", model.seed, app.elapsed_frames()));
```

### Frame Sequence Export (for video)

Capture every frame as a numbered PNG for later assembly into video (e.g., with ffmpeg):

```rust
fn view(app: &App, model: &Model, frame: Frame) {
    // ... drawing code ...
    draw.to_frame(app, &frame).unwrap();

    // Save every frame as numbered PNG
    let path = app.project_path()
        .unwrap()
        .join(app.exe_name().unwrap())
        .join(format!("{:04}", frame.nth()))
        .with_extension("png");
    app.main_window().capture_frame(path);
}
```

### Hi-Res Capture (Print Quality)

For output larger than the window (e.g., 4K/8K for prints), render to an offscreen texture
using a dedicated `Draw` instance and `Renderer`:

```rust
use nannou::prelude::*;

struct Model {
    texture: wgpu::Texture,
    draw: nannou::Draw,
    renderer: nannou::draw::Renderer,
    texture_capturer: wgpu::TextureCapturer,
    texture_reshaper: wgpu::TextureReshaper,
}

fn model(app: &App) -> Model {
    let texture_size = [3840, 2160]; // 4K UHD
    let [win_w, win_h] = [texture_size[0] / 4, texture_size[1] / 4];
    let w_id = app.new_window().size(win_w, win_h).view(view).build().unwrap();
    let window = app.window(w_id).unwrap();
    let device = window.device();
    let sample_count = window.msaa_samples();

    let texture = wgpu::TextureBuilder::new()
        .size(texture_size)
        .usage(wgpu::TextureUsages::RENDER_ATTACHMENT | wgpu::TextureUsages::TEXTURE_BINDING)
        .sample_count(sample_count)
        .format(wgpu::TextureFormat::Rgba16Float)
        .build(device);

    let draw = nannou::Draw::new();
    let descriptor = texture.descriptor();
    let renderer = nannou::draw::RendererBuilder::new()
        .build_from_texture_descriptor(device, descriptor);
    let texture_capturer = wgpu::TextureCapturer::default();
    let texture_view = texture.view().build();
    let texture_reshaper = wgpu::TextureReshaper::new(
        device, &texture_view, sample_count,
        texture.sample_type(), sample_count, Frame::TEXTURE_FORMAT,
    );

    Model { texture, draw, renderer, texture_capturer, texture_reshaper }
}
```

Then in `update()`, draw to `model.draw`, render with `model.renderer.render_to_texture()`,
and capture with `model.texture_capturer.capture()`. In `view()`, use
`model.texture_reshaper.encode_render_pass()` to display the downscaled texture.
See the nannou `draw_capture_hi_res` example for the complete pattern.

## Window Configuration

```rust
app.new_window()
    .size(1200, 800)                 // initial size in pixels
    .title("Piece Title")
    .resizable(true)                 // default is true
    .view(view)
    .key_pressed(key_pressed)
    .mouse_pressed(mouse_pressed)    // optional
    .mouse_moved(mouse_moved)        // optional
    .build()
    .unwrap();
```

## Performance Tips

- Use `cargo run --release` for real-time work — debug builds are ~10x slower
- For particle systems, store positions in a `Vec<Vec2>` and iterate with `.iter()` / `.iter_mut()`
- nannou's `Draw` batches draw calls — thousands of shapes per frame is fine in release mode
- For pixel-level manipulation, use `nannou::image` and `nannou::wgpu::Texture`

## Common Patterns

### Palette System
```rust
fn palette(name: &str) -> Vec<Hsla> {
    match name {
        "sunset" => vec![
            hsla(0.02, 0.9, 0.6, 1.0),  // warm red
            hsla(0.08, 0.95, 0.6, 1.0),  // orange
            hsla(0.12, 0.9, 0.7, 1.0),   // golden
            hsla(0.55, 0.4, 0.3, 1.0),   // deep blue
        ],
        "ocean" => vec![
            hsla(0.5, 0.7, 0.4, 1.0),
            hsla(0.55, 0.8, 0.5, 1.0),
            hsla(0.45, 0.6, 0.6, 1.0),
            hsla(0.6, 0.5, 0.3, 1.0),
        ],
        _ => vec![hsla(0.0, 0.0, 1.0, 1.0)],
    }
}
```

### Map Range (equivalent to p5's `map()`)
```rust
// nannou provides map_range
let mapped = map_range(val, in_min, in_max, out_min, out_max);
```

### Time-based Animation
```rust
fn view(app: &App, model: &Model, frame: Frame) {
    let t = app.time;                    // seconds since start (f32)
    let frames = app.elapsed_frames();   // frame count (u64)
    // ...
}
```

## Dependencies

Minimal `Cargo.toml`:
```toml
[package]
name = "piece-name"
version = "0.1.0"
edition = "2021"

[dependencies]
nannou = "0.19"
```

If you need additional noise types or math:
```toml
[dependencies]
nannou = "0.19"
noise = "0.8"          # if nannou's re-export is insufficient
glam = "0.24"          # vec math (nannou already re-exports some)
```

## Notable Practitioners

- **Manoloide** — prolific nannou artist, known for organic particle systems and flow fields
- **MacTuitui** — geometric and generative prints using nannou
- **The nannou community** — active on Discord, sharing sketches and techniques

## Key Differences from p5.js

| Concept | p5.js | nannou |
|---------|-------|--------|
| Coordinate origin | Top-left | Center |
| Y direction | Down | Up |
| Color range | 0–255 (default) | 0.0–1.0 (float) |
| Random seed | `randomSeed(n)` | `StdRng::seed_from_u64(n)` |
| Noise | `noise(x, y)` | `perlin.get([x, y])` |
| Frame rate | `frameRate(60)` | Vsync by default (~60fps) |
| Save frame | N/A (canvas.toBlob) | `capture_frame("file.png")` |
| State | Global variables | `Model` struct |
| Mutability | Implicit | Explicit (Rust ownership) |
