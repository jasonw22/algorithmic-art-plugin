# Color Science for Algorithmic Art

## Philosophy

Color is the most immediate emotional channel in generative art — a piece's palette determines
mood, depth, and readability before any shape is perceived. Yet most creative coding frameworks
default to sRGB, a color space designed for monitors, not human perception. Moving through
sRGB linearly produces muddy midpoints (red→green passes through brown), uneven brightness
steps, and palettes that feel artificial.

**Perceptual color spaces** — Lab, LCH, Oklab, Oklch — model color the way humans see it.
Equal numeric steps produce equal visual steps. Interpolation stays vivid. Complementary
colors have matched visual weight. Working in perceptual space is the single highest-leverage
upgrade for algorithmic art aesthetics.

## Perceptual Color Spaces

### sRGB (Standard)
The default web color space. Channels: Red, Green, Blue (0–255 or 0–1).
- **Pros**: universal, fast, direct hardware mapping
- **Cons**: perceptually non-uniform — equal RGB steps ≠ equal visual steps; interpolation
  produces muddy intermediate hues

### HSL / HSV
Hue-Saturation-Lightness / Hue-Saturation-Value. Common in creative coding.
- **Pros**: intuitive hue control, easy rainbow palettes
- **Cons**: "lightness" is not perceptual — HSL yellow at L=50% is far brighter than blue at L=50%.
  Palettes look unbalanced. HSV is even worse for perceptual uniformity.

### Lab (CIELAB, D50/D65)
The first perceptually uniform space (CIE 1976). Channels: L* (lightness 0–100),
a* (green↔red), b* (blue↔yellow).
- **Pros**: perceptually uniform for small color differences
- **Cons**: not perfectly uniform for large differences; a*/b* axes are unintuitive

### LCH (CIELCH)
Polar form of Lab. Channels: L (lightness), C (chroma/saturation), H (hue angle 0–360°).
- **Pros**: perceptually uniform + intuitive hue/saturation control. The best general-purpose
  perceptual space for generative art. Easy to construct harmonious palettes by rotating hue.
- **Cons**: some high-chroma LCH values fall outside sRGB gamut — clamp before display.

### Oklab / Oklch (2021)
Björn Ottosson's improvement on Lab/LCH. Better hue linearity — gradients through Oklab
maintain consistent hue where Lab can shift. Oklch is its polar form.
- **Pros**: state-of-the-art perceptual uniformity, especially for gradients. CSS Color Level 4
  supports `oklch()` natively. Ideal for generative art palettes.
- **Cons**: newer, less library support outside web/thi.ng. Minor differences from LCH in practice.

## Palette Generation Techniques

### Cosine Gradients (Inigo Quilez)

Generate infinitely smooth palettes from just 4 coefficient vectors (a, b, c, d):

```
color(t) = a + b * cos(2π * (c * t + d))
```

Where t ∈ [0, 1] maps to a position along the gradient. Each coefficient is a vec3 (RGB):
- **a**: bias (overall brightness/darkness)
- **b**: amplitude (contrast range)
- **c**: frequency (how many color cycles)
- **d**: phase (where each channel's cycle starts — this controls the hue relationships)

#### Classic Cosine Palette Presets

```javascript
// In JavaScript (p5.js or plain canvas):
function cosinePalette(t, a, b, c, d) {
  return [
    a[0] + b[0] * Math.cos(Math.PI * 2 * (c[0] * t + d[0])),
    a[1] + b[1] * Math.cos(Math.PI * 2 * (c[1] * t + d[1])),
    a[2] + b[2] * Math.cos(Math.PI * 2 * (c[2] * t + d[2])),
  ];
}

// Presets (a, b, c, d as [r, g, b]):
const PALETTES = {
  rainbow:   { a: [0.5,0.5,0.5], b: [0.5,0.5,0.5], c: [1.0,1.0,1.0], d: [0.00,0.33,0.67] },
  sunset:    { a: [0.5,0.5,0.5], b: [0.5,0.5,0.5], c: [1.0,1.0,1.0], d: [0.00,0.10,0.20] },
  ocean:     { a: [0.5,0.5,0.5], b: [0.5,0.5,0.5], c: [1.0,1.0,0.5], d: [0.80,0.90,0.30] },
  fire:      { a: [0.5,0.5,0.5], b: [0.5,0.5,0.5], c: [2.0,1.0,0.0], d: [0.50,0.20,0.25] },
  electric:  { a: [0.5,0.5,0.5], b: [0.5,0.5,0.5], c: [1.0,1.0,1.0], d: [0.30,0.20,0.20] },
  pastel:    { a: [0.8,0.5,0.4], b: [0.2,0.4,0.2], c: [2.0,1.0,1.0], d: [0.00,0.25,0.25] },
  neon:      { a: [0.5,0.5,0.5], b: [0.5,0.5,0.5], c: [1.0,0.7,0.4], d: [0.00,0.15,0.20] },
  grayscale: { a: [0.5,0.5,0.5], b: [0.5,0.5,0.5], c: [0.0,0.0,0.0], d: [0.00,0.00,0.00] },
};
```

```glsl
// In GLSL (shader mode):
vec3 cosinePalette(float t, vec3 a, vec3 b, vec3 c, vec3 d) {
  return a + b * cos(6.28318 * (c * t + d));
}
```

**Design tip**: vary the **d** (phase) vector to explore entirely different palettes.
The hue relationships are almost entirely controlled by phase offsets between channels.

### LCH/Oklch Range-Based Theme Generation

Define a palette by constraining ranges in perceptual space rather than picking specific colors.
This produces coherent palettes with controlled variety — the approach used by thi.ng/color.

```javascript
// Define a theme as ranges in LCH space:
const theme = {
  hue:    { center: 30, range: 40 },    // warm hues (oranges, reds, yellows)
  chroma: { min: 40, max: 90 },          // moderate to vivid saturation
  light:  { min: 30, max: 85 },          // avoid extremes
};

// Generate N colors from the theme:
function generateTheme(theme, n, rng) {
  const colors = [];
  for (let i = 0; i < n; i++) {
    const h = theme.hue.center + (rng() - 0.5) * 2 * theme.hue.range;
    const c = theme.chroma.min + rng() * (theme.chroma.max - theme.chroma.min);
    const l = theme.light.min + rng() * (theme.light.max - theme.light.min);
    colors.push(lchToRgb(l, c, h)); // convert to sRGB for display
  }
  return colors;
}
```

**Theme presets** (LCH ranges):
- **Warm Earth**: hue 20–60, chroma 30–70, lightness 25–75
- **Cool Ocean**: hue 180–240, chroma 20–60, lightness 30–80
- **Neon Night**: hue 260–340, chroma 80–130, lightness 40–70
- **Forest**: hue 80–160, chroma 25–65, lightness 20–60
- **Monochrome**: any single hue ± 5°, chroma 0–40, full lightness range

### Harmonic Color Schemes

Classical harmony rules from color theory, applied in LCH/Oklch for perceptual accuracy:

| Scheme | Rule | Description |
|--------|------|-------------|
| Complementary | H₂ = H₁ + 180° | Maximum contrast — tension, vibrance |
| Split-complementary | H₂ = H₁ + 150°, H₃ = H₁ + 210° | High contrast but more nuanced than complementary |
| Triadic | H₂ = H₁ + 120°, H₃ = H₁ + 240° | Balanced, vibrant — works well for three-element compositions |
| Tetradic (square) | +90°, +180°, +270° | Rich palette, works when one color dominates |
| Analogous | H₁ ± 30° | Harmonious, calm — nature-inspired |

```javascript
// Generate a triadic scheme in LCH:
function triadicScheme(baseHue, chroma, lightness) {
  return [
    lchToRgb(lightness, chroma, baseHue),
    lchToRgb(lightness, chroma, (baseHue + 120) % 360),
    lchToRgb(lightness, chroma, (baseHue + 240) % 360),
  ];
}

// Vary lightness/chroma per stop for depth:
function triadicWithDepth(baseHue, rng) {
  return [
    lchToRgb(70, 60, baseHue),                     // light, moderate
    lchToRgb(45, 80, (baseHue + 120) % 360),        // dark, vivid
    lchToRgb(85, 35, (baseHue + 240) % 360),        // very light, soft
  ];
}
```

### Perceptual Interpolation

When interpolating between colors, the color space matters enormously:

```javascript
// BAD: sRGB interpolation (muddy midpoints)
function lerpRGB(c1, c2, t) {
  return [
    c1[0] + (c2[0] - c1[0]) * t,
    c1[1] + (c2[1] - c1[1]) * t,
    c1[2] + (c2[2] - c1[2]) * t,
  ];
}
// Red → Green passes through ugly brown/gray

// GOOD: Oklab interpolation (vivid midpoints)
function lerpOklab(c1, c2, t) {
  const lab1 = rgbToOklab(c1);
  const lab2 = rgbToOklab(c2);
  const mixed = [
    lab1[0] + (lab2[0] - lab1[0]) * t,
    lab1[1] + (lab2[1] - lab1[1]) * t,
    lab1[2] + (lab2[2] - lab1[2]) * t,
  ];
  return oklabToRgb(mixed);
}
// Red → Green passes through vivid yellow
```

### Multi-Stop Gradients

Create smooth gradients with multiple color stops, interpolated in perceptual space:

```javascript
function multiGradient(colors, t) {
  // colors: array of [r,g,b] in sRGB
  // t: 0–1 position along gradient
  const n = colors.length - 1;
  const segment = Math.min(Math.floor(t * n), n - 1);
  const localT = (t * n) - segment;
  return lerpOklab(colors[segment], colors[segment + 1], localT);
}
```

## Oklab Conversion Functions

### JavaScript (for p5.js / Canvas)

```javascript
// sRGB → linear RGB
function srgbToLinear(x) {
  return x <= 0.04045 ? x / 12.92 : Math.pow((x + 0.055) / 1.055, 2.4);
}

// linear RGB → sRGB
function linearToSrgb(x) {
  return x <= 0.0031308 ? 12.92 * x : 1.055 * Math.pow(x, 1/2.4) - 0.055;
}

// sRGB [0-1] → Oklab [L, a, b]
function rgbToOklab(rgb) {
  let r = srgbToLinear(rgb[0]);
  let g = srgbToLinear(rgb[1]);
  let b = srgbToLinear(rgb[2]);

  let l = 0.4122214708 * r + 0.5363325363 * g + 0.0514459929 * b;
  let m = 0.2119034982 * r + 0.6806995451 * g + 0.1073969566 * b;
  let s = 0.0883024619 * r + 0.2817188376 * g + 0.6299787005 * b;

  l = Math.cbrt(l); m = Math.cbrt(m); s = Math.cbrt(s);

  return [
    0.2104542553 * l + 0.7936177850 * m - 0.0040720468 * s,
    1.9779984951 * l - 2.4285922050 * m + 0.4505937099 * s,
    0.0259040371 * l + 0.7827717662 * m - 0.8086757660 * s,
  ];
}

// Oklab [L, a, b] → sRGB [0-1]
function oklabToRgb(lab) {
  let l = lab[0] + 0.3963377774 * lab[1] + 0.2158037573 * lab[2];
  let m = lab[0] - 0.1055613458 * lab[1] - 0.0638541728 * lab[2];
  let s = lab[0] - 0.0894841775 * lab[1] - 1.2914855480 * lab[2];

  l = l*l*l; m = m*m*m; s = s*s*s;

  return [
    clamp01(linearToSrgb(+4.0767416621 * l - 3.3077115913 * m + 0.2309699292 * s)),
    clamp01(linearToSrgb(-1.2684380046 * l + 2.6097574011 * m - 0.3413193965 * s)),
    clamp01(linearToSrgb(-0.0041960863 * l - 0.7034186147 * m + 1.7076147010 * s)),
  ];
}

function clamp01(x) { return Math.max(0, Math.min(1, x)); }

// Oklch (polar) ↔ Oklab
function oklabToOklch(lab) {
  return [lab[0], Math.sqrt(lab[1]*lab[1] + lab[2]*lab[2]), Math.atan2(lab[2], lab[1]) * 180/Math.PI];
}
function oklchToOklab(lch) {
  const hRad = lch[2] * Math.PI / 180;
  return [lch[0], lch[1] * Math.cos(hRad), lch[1] * Math.sin(hRad)];
}

// LCH convenience: generate a color from lightness, chroma, hue
function lchToRgb(l, c, h) {
  return oklabToRgb(oklchToOklab([l / 100, c / 150, h]));
  // Normalize: L in 0-100 → 0-1, C in 0-150 → 0-1 (approx)
}
```

### GLSL (for shader mode)

```glsl
// Oklab in GLSL — for perceptual gradients on the GPU
vec3 srgbToLinear(vec3 c) {
  return mix(c / 12.92, pow((c + 0.055) / 1.055, vec3(2.4)), step(0.04045, c));
}

vec3 linearToSrgb(vec3 c) {
  return mix(12.92 * c, 1.055 * pow(c, vec3(1.0/2.4)) - 0.055, step(0.0031308, c));
}

vec3 rgbToOklab(vec3 c) {
  c = srgbToLinear(c);
  float l = pow(0.4122 * c.r + 0.5363 * c.g + 0.0514 * c.b, 1.0/3.0);
  float m = pow(0.2119 * c.r + 0.6807 * c.g + 0.1074 * c.b, 1.0/3.0);
  float s = pow(0.0883 * c.r + 0.2817 * c.g + 0.6300 * c.b, 1.0/3.0);
  return vec3(
    0.2105 * l + 0.7936 * m - 0.0041 * s,
    1.9780 * l - 2.4286 * m + 0.4506 * s,
    0.0259 * l + 0.7828 * m - 0.8087 * s
  );
}

vec3 oklabToRgb(vec3 lab) {
  float l = lab.x + 0.3963 * lab.y + 0.2158 * lab.z;
  float m = lab.x - 0.1056 * lab.y - 0.0639 * lab.z;
  float s = lab.x - 0.0895 * lab.y - 1.2915 * lab.z;
  l = l*l*l; m = m*m*m; s = s*s*s;
  return linearToSrgb(clamp(vec3(
    4.0767 * l - 3.3077 * m + 0.2310 * s,
   -1.2684 * l + 2.6098 * m - 0.3413 * s,
   -0.0042 * l - 0.7034 * m + 1.7076 * s
  ), 0.0, 1.0));
}

// Interpolate in Oklab for perceptually uniform gradients:
vec3 mixOklab(vec3 rgb1, vec3 rgb2, float t) {
  return oklabToRgb(mix(rgbToOklab(rgb1), rgbToOklab(rgb2), t));
}
```

## Application to Algorithm Families

### Flow Fields
Map particle **age**, **speed**, or **cumulative distance** to a cosine palette or multi-stop
gradient. Interpolate in Oklab for smooth color transitions along trails.

### Fractals
Use **iteration count** mapped through a cosine palette for classic Mandelbrot/Julia coloring.
Orbit trap coloring benefits enormously from perceptual interpolation — orbit distance → Oklab
gradient produces smoother, more vivid results than direct RGB mapping.

### Reaction-Diffusion
Map chemical concentration (0–1) through a multi-stop gradient in Oklab. Use LCH theme
generation to create biologically-inspired palettes (coral: warm oranges/pinks; lichen:
cool greens/teals; petri dish: vivid neons on dark background).

### Strange Attractors
Point-cloud attractors accumulate density via low-opacity plotting. Use separate palettes
for low-density (dark, cool) and high-density (bright, warm) regions. The density→color
mapping is most effective when designed in LCH to ensure the brightness gradient is
perceptually linear.

### Cellular Automata
Two-state CA benefit from high-contrast complementary pairs chosen in LCH (matched lightness
contrast). Multi-state CA (Brian's Brain, continuous automata) map state values through
cosine palettes or LCH gradients.

### Tiling
Assign tile types or regions perceptually-spaced colors from a harmonic scheme. Analogous
schemes (±30° hue) produce calm, decorative patterns. Triadic schemes create more dynamic
compositions. Use LCH to ensure tiles of different colors have equal visual weight.

## Key References

- **Björn Ottosson** — creator of Oklab: bottosson.github.io/posts/oklab/
- **Inigo Quilez** — cosine palettes: iquilezles.org/articles/palettes/
- **thi.ng/color** — 16 color models, cosine gradients, LCH theme generation, harmonic schemes
- **CSS Color Level 4** — native `oklch()` support in browsers
- **Bartosz Ciechanowski** — interactive explainers on color perception
