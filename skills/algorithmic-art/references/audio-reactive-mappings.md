# Audio-Reactive Mappings Reference

## Overview

This reference covers the **consumption** side of audio-reactive art: how to analyze incoming
audio and map frequency/amplitude data to visual parameters. For audio **generation** (oscillators,
envelopes, drums, shader synthesis), see `sound-synthesis.md`.

The core pipeline: **Audio Source -> Analysis -> Band Extraction -> Smoothing -> Visual Mapping**

## Audio Analysis Setup

### Web Audio API AnalyserNode

```javascript
// Resume AudioContext on first user interaction (click/keypress)
// AudioContext starts suspended — visuals silently fail without this
document.addEventListener('click', () => audioCtx.resume(), { once: true });

const audioCtx = new AudioContext();
const analyser = audioCtx.createAnalyser();
analyser.fftSize = 2048;              // Power of 2: 32–32768. Higher = finer frequency resolution
analyser.smoothingTimeConstant = 0.8; // 0 = no smoothing (jittery), 1 = frozen. 0.7–0.85 is typical

// Connect a source (microphone, audio element, oscillator, etc.)
const source = audioCtx.createMediaElementSource(audioElement);
source.connect(analyser);
analyser.connect(audioCtx.destination);

// Frequency data buffer (half of fftSize)
const frequencyData = new Uint8Array(analyser.frequencyBinCount);
```

### Microphone Input

```javascript
async function connectMicrophone(audioCtx, analyser) {
  const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
  const mic = audioCtx.createMediaStreamSource(stream);
  mic.connect(analyser);
  // Don't connect mic to destination — causes feedback
}
```

### Critical: Logarithmic Frequency Mapping

FFT bins are linearly spaced, but human hearing is logarithmic. The first 100 bins of a
2048-sample FFT might cover 0–2kHz while the last 924 cover 2–22kHz. Always use logarithmic
mapping when displaying or extracting frequency bands.

```javascript
// Linear bin index to frequency
function binToFreq(bin, fftSize, sampleRate) {
  return bin * sampleRate / fftSize;
}

// Frequency to bin index (for band boundary lookup)
function freqToBin(freq, fftSize, sampleRate) {
  return Math.round(freq * fftSize / sampleRate);
}
```

## Frequency Band Extraction

### Three-Band Split (Bass / Mid / High)

The workhorse mapping for most audio-reactive art. Extract energy in three perceptual bands.

```javascript
function extractBands(frequencyData, fftSize, sampleRate) {
  const len = frequencyData.length;

  // Approximate frequency boundaries
  // Bass: 20–250 Hz, Mid: 250–4000 Hz, High: 4000–16000 Hz
  const bassBins  = [freqToBin(20, fftSize, sampleRate),   freqToBin(250, fftSize, sampleRate)];
  const midBins   = [freqToBin(250, fftSize, sampleRate),  freqToBin(4000, fftSize, sampleRate)];
  const highBins  = [freqToBin(4000, fftSize, sampleRate), freqToBin(16000, fftSize, sampleRate)];

  return {
    bass:  avgRange(frequencyData, bassBins[0], bassBins[1]),
    mid:   avgRange(frequencyData, midBins[0], midBins[1]),
    high:  avgRange(frequencyData, highBins[0], highBins[1]),
  };
}

function avgRange(data, start, end) {
  let sum = 0;
  const s = Math.max(0, Math.floor(start));
  const e = Math.min(data.length, Math.floor(end));
  for (let i = s; i < e; i++) sum += data[i];
  return (e > s) ? sum / ((e - s) * 255) : 0;  // Normalize to 0–1
}
```

### Multi-Band Split

For finer control, split into more bands:

```javascript
const BAND_RANGES = [
  { name: 'sub',        lo: 20,    hi: 60    },  // Sub-bass (felt more than heard)
  { name: 'bass',       lo: 60,    hi: 250   },  // Bass (kick drum, bass guitar)
  { name: 'lowMid',     lo: 250,   hi: 1000  },  // Low-mid (vocals, guitar body)
  { name: 'mid',        lo: 1000,  hi: 4000  },  // Mid (vocal presence, snare attack)
  { name: 'highMid',    lo: 4000,  hi: 8000  },  // High-mid (cymbal body, consonants)
  { name: 'high',       lo: 8000,  hi: 16000 },  // High (sibilance, air, shimmer)
];
```

## Smoothing

Raw FFT data is noisy frame-to-frame. Layer smoothing on top of `smoothingTimeConstant` for
per-band control and musically useful response curves.

### Exponential Moving Average (EMA)

The fundamental smoother. Higher factor = smoother but laggier. Lower = responsive but jittery.

```javascript
class Smoother {
  constructor(factor = 0.9) {
    this.factor = factor;
    this.value = 0;
  }
  update(input) {
    this.value = this.factor * this.value + (1 - this.factor) * input;
    return this.value;
  }
  reset() { this.value = 0; }
}
```

**Recommended factors by band:**

| Band | Factor | Character |
|------|--------|-----------|
| Bass | 0.85 | Responsive enough to feel punchy, smooth enough to avoid flicker |
| Mid | 0.90 | Moderate smoothing for melodic content |
| High | 0.92 | Heavy smoothing — raw highs are spiky and distracting |

### Full Mapper Class

Combines band extraction and per-band smoothing into one reusable object.

```javascript
class AudioReactiveMapper {
  constructor(analyser) {
    this.analyser = analyser;
    this.fftSize = analyser.fftSize;
    this.buffer = new Uint8Array(analyser.frequencyBinCount);
    this.sampleRate = analyser.context.sampleRate;
    this.smoothers = {
      bass: new Smoother(0.85),
      mid:  new Smoother(0.90),
      high: new Smoother(0.92),
    };
  }

  getValues() {
    this.analyser.getByteFrequencyData(this.buffer);
    const raw = extractBands(this.buffer, this.fftSize, this.sampleRate);
    return {
      bass: this.smoothers.bass.update(raw.bass),
      mid:  this.smoothers.mid.update(raw.mid),
      high: this.smoothers.high.update(raw.high),
    };
  }
}
```

## Beat Detection

### Threshold-Based Beat Detector

Detects beats by comparing current energy to a rolling average. Simple and effective for
kick-drum-heavy music.

```javascript
class BeatDetector {
  constructor(threshold = 1.3, decayRate = 0.98, minInterval = 200) {
    this.threshold = threshold;  // Current must exceed average by this ratio
    this.decayRate = decayRate;  // How fast the average decays
    this.minInterval = minInterval; // Minimum ms between beats (prevents doubles)
    this.average = 0;
    this.lastBeatTime = 0;
  }

  detect(bass, now = performance.now()) {
    this.average = Math.max(bass, this.average * this.decayRate);
    const isBeat = bass > this.average * this.threshold
                   && (now - this.lastBeatTime) > this.minInterval;
    if (isBeat) this.lastBeatTime = now;
    return isBeat;
  }
}
```

## Beat Response Patterns

Reusable classes that translate a boolean beat trigger into smooth visual animations.

### BeatFlash — Instantaneous Intensity Decay

Jumps to 1.0 on beat, then exponentially decays. Good for background flashes, glow pulses,
emissive intensity bursts.

```javascript
class BeatFlash {
  constructor(decayRate = 0.95) {
    this.intensity = 0;
    this.decayRate = decayRate;
  }

  trigger() { this.intensity = 1; }

  update() {
    this.intensity *= this.decayRate;
    return this.intensity;
  }
}

// Usage: background flash on beat
const flash = new BeatFlash(0.93);
// In render loop:
if (isBeat) flash.trigger();
const i = flash.update();
ctx.fillStyle = `rgba(0, 245, 255, ${i})`;
```

### BeatPop — Scale Bounce

Pops to a target scale on beat, then lerps back to 1.0. Good for mesh/element scaling,
UI "bounce" effects.

```javascript
class BeatPop {
  constructor(popScale = 1.3, decayRate = 0.9) {
    this.scale = 1;
    this.targetScale = 1;
    this.popScale = popScale;
    this.decayRate = decayRate;
  }

  trigger() { this.targetScale = this.popScale; }

  update() {
    this.scale += (this.targetScale - this.scale) * 0.3;  // Lerp toward target
    this.targetScale = 1 + (this.targetScale - 1) * this.decayRate;  // Decay target back to 1
    return this.scale;
  }
}
```

### BeatSpawner — Cooldown-Gated Emission

Spawns particles (or any discrete event) on beat with a cooldown to prevent rapid-fire
double triggers.

```javascript
class BeatSpawner {
  constructor(onSpawn, minInterval = 200) {
    this.onSpawn = onSpawn;
    this.cooldown = 0;
    this.minInterval = minInterval;
  }

  check(isBeat, deltaMs = 16) {
    this.cooldown = Math.max(0, this.cooldown - deltaMs);
    if (isBeat && this.cooldown <= 0) {
      this.onSpawn();
      this.cooldown = this.minInterval;
    }
  }
}
```

## Visual Property Mappings

### Frequency Band -> Visual Property Table

A practical cheat sheet for which bands drive which visual properties. These are starting
points — experiment with cross-mappings for unexpected results.

| Visual Property | Best Band | Why |
|----------------|-----------|-----|
| Scale / size pulse | Bass | Kick drums produce strong low-frequency transients — visceral "thump" |
| Rotation speed | Mid | Melodic content gives smooth, musical rotation |
| Position vibration / shake | High | Hi-hats and cymbals produce rapid spiky energy — good for jitter |
| Brightness / emissive | Bass | Volume peaks align with bass hits in most music |
| Color hue shift | Mid | Maps melodic changes to color changes perceptually |
| Opacity / fade | Overall RMS | Tracks the full-spectrum "loudness" of the signal |
| Bloom intensity | Bass | Bass hits = visual "glow" impact |
| Chromatic aberration | High | High-frequency spikes create glitch-like displacement |
| Particle emission rate | Bass (beat) | Burst particles on beat for rhythmic spawning |
| Vignette darkness | Beat flash | Close in on beat, then release — creates visual "breathing" |

### Scale / Size

```javascript
// Continuous bass-reactive scale (mesh or element)
function applyScale(target, bass) {
  const scale = 1 + bass * 0.3;  // Range: 1.0–1.3
  // Three.js:
  target.scale.setScalar(scale);
  // DOM:
  // target.style.transform = `scale(${scale})`;
}

// Beat-driven pop (use BeatPop class)
const pop = new BeatPop(1.4, 0.92);
// In render loop:
if (isBeat) pop.trigger();
mesh.scale.setScalar(pop.update());
```

### Position / Movement

```javascript
// High-frequency vibration (jitter)
function applyShake(target, high) {
  const shake = high * 5;  // Pixels or world units
  const x = (Math.random() - 0.5) * shake;
  const y = (Math.random() - 0.5) * shake;
  target.position.x = x;
  target.position.y = y;
}

// Smooth wave motion driven by mid
function applyWaveMotion(target, mid, time) {
  target.position.y = Math.sin(time * 2) * mid * 2;
}
```

### Rotation

```javascript
// Continuous rotation with energy-driven speed
class SpinController {
  constructor(baseSpeed = 0.01) {
    this.rotation = 0;
    this.baseSpeed = baseSpeed;
  }
  update(energy) {
    this.rotation += this.baseSpeed + energy * 0.05;
    return this.rotation;
  }
}
```

### Color / Brightness

```javascript
// Brightness from overall volume
function applyBrightness(element, volume) {
  const brightness = 0.5 + volume * 0.5;  // 50%–100%
  element.style.filter = `brightness(${brightness})`;
}

// Color shift from frequency balance
function reactiveColor(bass, mid, high) {
  // More bass = warm (red/magenta), more high = cool (cyan)
  const r = Math.floor(bass * 255);
  const g = Math.floor(mid * 100);
  const b = Math.floor(high * 255);
  return `rgb(${r}, ${g}, ${b})`;
}

// Three.js emissive glow
function applyGlow(material, bass) {
  material.emissiveIntensity = 1 + bass * 3;
}
```

### Opacity

```javascript
function applyOpacity(element, volume) {
  element.style.opacity = 0.3 + volume * 0.7;  // Never fully invisible
}
```

## Audio-Driven Post-Processing

### Bloom

Bass drives bloom intensity — visual "impact" on every hit.

```javascript
// Three.js EffectComposer / postprocessing library
function updateBloom(bloomPass, audioData) {
  bloomPass.intensity = 1 + audioData.bass * 2;
  bloomPass.luminanceThreshold = 0.3 - audioData.high * 0.2;  // More bloom on high energy
}
```

### Chromatic Aberration

High frequencies drive RGB offset — creates glitchy, energetic feel.

```javascript
function updateChroma(chromaPass, audioData) {
  const offset = 0.001 + audioData.high * 0.005;
  chromaPass.offset.set(offset, offset * 0.5);
}
```

### Vignette

Beat-driven vignette creates a "breathing" tunnel effect.

```javascript
const vignetteFlash = new BeatFlash(0.95);

function updateVignette(vignettePass, isBeat) {
  if (isBeat) vignetteFlash.trigger();
  vignettePass.darkness = 0.5 + vignetteFlash.update() * 0.4;  // Darken on beat, release
}
```

## GLSL Audio Uniforms

Pass audio data into fragment shaders as uniforms. The three-band split is the most common pattern.

```javascript
// In shaderUniforms():
return {
  u_bass:   { value: 0 },
  u_mid:    { value: 0 },
  u_high:   { value: 0 },
  u_beat:   { value: 0 },  // 0 or 1, smoothed
};

// In shaderAnimate():
const audio = mapper.getValues();
const isBeat = beatDetector.detect(audio.bass);
uniforms.u_bass.value = audio.bass;
uniforms.u_mid.value = audio.mid;
uniforms.u_high.value = audio.high;
uniforms.u_beat.value = isBeat ? 1.0 : uniforms.u_beat.value * 0.95;
```

```glsl
// In fragment shader:
uniform float u_bass;
uniform float u_mid;
uniform float u_high;
uniform float u_beat;

// Bass-reactive glow
float glow = smoothstep(0.5 - u_bass * 0.3, 0.0, dist);

// Beat-reactive color flash
vec3 color = mix(baseColor, flashColor, u_beat);

// Mid-driven domain warping
vec2 warpedUV = uv + 0.1 * u_mid * vec2(sin(uv.y * 10.0), cos(uv.x * 10.0));

// High-frequency chromatic split
float r = texture(tex, uv + u_high * 0.01).r;
float g = texture(tex, uv).g;
float b = texture(tex, uv - u_high * 0.01).b;
```

### FFT as 1D Texture

For full-spectrum visualization (not just three bands), pass the entire FFT as a texture:

```javascript
const fftTexture = new THREE.DataTexture(
  frequencyData, frequencyData.length, 1,
  THREE.RedFormat, THREE.UnsignedByteType
);
fftTexture.needsUpdate = true;  // Set every frame after getByteFrequencyData()

// In shaderUniforms():
return { u_fft: { value: fftTexture } };
```

```glsl
uniform sampler2D u_fft;

// Sample frequency at normalized position (0 = lowest, 1 = highest)
float freq = texture(u_fft, vec2(uv.x, 0.5)).r;
```

## CSS Variable Bridge (DOM-Based Art)

For p5.js or thi.ng modes where multiple DOM elements react to audio, push values to
CSS custom properties once per frame and let CSS do the work:

```javascript
// In animation loop:
const root = document.documentElement;
root.style.setProperty('--audio-bass', audio.bass);
root.style.setProperty('--audio-mid', audio.mid);
root.style.setProperty('--audio-high', audio.high);
```

```css
.reactive-element {
  transform: scale(calc(1 + var(--audio-bass) * 0.2));
  filter: brightness(calc(0.5 + var(--audio-mid) * 0.5));
  opacity: calc(0.3 + var(--audio-high) * 0.7);
}
```

## Complete Integration Example

Wiring everything together in the Three.js Scene template:

```javascript
function sceneSetup(THREE, scene, camera, renderer, params, seed) {
  // 1. Audio context (resume on interaction)
  const audioCtx = new AudioContext();
  document.addEventListener('click', () => audioCtx.resume(), { once: true });

  // 2. Audio source
  const audio = new Audio('music.mp3');
  audio.crossOrigin = 'anonymous';
  const source = audioCtx.createMediaElementSource(audio);

  // 3. Analyser
  const analyser = audioCtx.createAnalyser();
  analyser.fftSize = 2048;
  analyser.smoothingTimeConstant = 0.8;
  source.connect(analyser);
  analyser.connect(audioCtx.destination);

  // 4. Mapper + beat detector + response objects
  const mapper = new AudioReactiveMapper(analyser);
  const beatDetector = new BeatDetector(1.3, 0.98, 250);
  const beatFlash = new BeatFlash(0.93);
  const beatPop = new BeatPop(1.3, 0.9);

  // 5. Scene objects
  const sphere = new THREE.Mesh(
    new THREE.SphereGeometry(1, 64, 64),
    new THREE.MeshStandardMaterial({ color: '#111', emissive: '#00f5ff', emissiveIntensity: 1 })
  );
  scene.add(sphere);

  audio.play();

  return { mapper, beatDetector, beatFlash, beatPop, sphere };
}

function sceneAnimate(THREE, scene, camera, state, params, time, delta) {
  const audio = state.mapper.getValues();
  const isBeat = state.beatDetector.detect(audio.bass);

  if (isBeat) {
    state.beatFlash.trigger();
    state.beatPop.trigger();
  }

  // Scale on beat
  state.sphere.scale.setScalar(state.beatPop.update());

  // Emissive on bass
  state.sphere.material.emissiveIntensity = 1 + audio.bass * 3;

  // Rotation on mid
  state.sphere.rotation.y += 0.01 + audio.mid * 0.03;
}
```

## Anti-Patterns

### 1. Forgetting AudioContext Resume
**Symptom**: Visualization runs but no audio data flows. All values are zero.
**Cause**: Browsers require user interaction before AudioContext can produce output.
**Fix**: `document.addEventListener('click', () => audioCtx.resume(), { once: true })`

### 2. Linear Frequency Display
**Symptom**: Bass dominates the visual, treble is invisible.
**Cause**: FFT bins are linearly spaced; first ~5% of bins cover 0–1kHz where most energy lives.
**Fix**: Use logarithmic bin mapping or perceptual band splits (see Band Extraction above).

### 3. No Smoothing
**Symptom**: Visuals are jittery and seizure-inducing.
**Cause**: Raw FFT data changes drastically frame-to-frame.
**Fix**: Use `analyser.smoothingTimeConstant` (0.7–0.85) AND per-band EMA smoothers.

### 4. Animation Loop Leak
**Symptom**: Multiple render loops running simultaneously after reset/rebuild.
**Cause**: `requestAnimationFrame` not cancelled on cleanup.
**Fix**: Store the animation frame ID and call `cancelAnimationFrame(id)` on teardown.

### 5. Canvas Not Sized for Retina
**Symptom**: Blurry visuals on high-DPI displays.
**Fix**: Multiply canvas dimensions by `devicePixelRatio` and handle resize events.

## Performance Tips

1. **Lower fftSize** if you only need three bands: 512 or 1024 is plenty (vs 2048 default)
2. **Skip audio frames**: Update audio data every 2nd or 3rd frame if visuals are smooth enough
3. **Batch DOM updates**: Use `requestAnimationFrame` and update all elements in one pass
4. **CSS variables over per-element JS**: One `setProperty` call vs N style mutations
5. **Avoid allocations in the render loop**: Reuse typed arrays, don't create new objects per frame
