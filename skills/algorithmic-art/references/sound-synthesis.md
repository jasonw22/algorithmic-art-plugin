# Sound Synthesis (Shader Audio) Reference

## Overview

Shader-based sound synthesis generates audio mathematically in a fragment shader. While
primarily a ShaderToy feature (where `mainSound` runs at 44.1kHz), the techniques can be
adapted to WebAudio + shader pipelines for our template system.

This is a niche but creative technique: algorithmic music, sonification of visuals,
audio-reactive art, and generative soundscapes.

## ShaderToy Sound Convention

ShaderToy provides a `mainSound` function that returns stereo audio at each sample:

```glsl
vec2 mainSound(float time) {
  // Return left and right channels, range [-1, 1]
  float freq = 440.0;
  float wave = sin(freq * 6.28318 * time);
  return vec2(wave);  // mono → both channels
}
```

## Basic Oscillators

```glsl
// Sine wave (pure tone)
float sine(float freq, float time) {
  return sin(freq * 6.28318 * time);
}

// Square wave (hollow, retro)
float square(float freq, float time) {
  return sign(sin(freq * 6.28318 * time));
}

// Sawtooth (bright, buzzy)
float sawtooth(float freq, float time) {
  return 2.0 * fract(freq * time) - 1.0;
}

// Triangle (soft, mellow)
float triangle(float freq, float time) {
  return abs(4.0 * fract(freq * time - 0.25) - 2.0) - 1.0;
}

// Band-limited sawtooth (reduces aliasing)
float blSaw(float freq, float time) {
  float t = fract(freq * time);
  // Polynomial approximation of band-limited saw
  return 2.0 * t - 1.0 - (2.0 * t - 1.0) * (2.0 * t - 1.0) * (2.0 * t - 1.0) / 3.0;
}
```

## Envelope (ADSR)

Shape amplitude over time for musical notes:

```glsl
// Simple attack-decay-sustain-release envelope
float adsr(float time, float attack, float decay, float sustain, float release, float noteLen) {
  if (time < 0.0) return 0.0;
  if (time < attack) return time / attack;
  time -= attack;
  if (time < decay) return 1.0 - (1.0 - sustain) * (time / decay);
  time -= decay;
  if (time < noteLen - attack - decay) return sustain;
  time -= noteLen - attack - decay;
  if (time < release) return sustain * (1.0 - time / release);
  return 0.0;
}

// Simpler exponential decay
float expDecay(float time, float decay) {
  return exp(-time * decay);
}
```

## Musical Notes

```glsl
// MIDI note to frequency (A4 = 69 = 440Hz)
float noteToFreq(float note) {
  return 440.0 * pow(2.0, (note - 69.0) / 12.0);
}

// Common note frequencies
// C4=261.63, D4=293.66, E4=329.63, F4=349.23, G4=392.00, A4=440.00, B4=493.88

// Simple sequencer
float sequencer(float time, float bpm) {
  float beatTime = 60.0 / bpm;
  float beat = floor(time / beatTime);
  float noteTime = mod(time, beatTime);

  // 8-note sequence (MIDI notes)
  float notes[8] = float[8](60.0, 62.0, 64.0, 65.0, 67.0, 69.0, 71.0, 72.0);
  int idx = int(mod(beat, 8.0));

  float freq = noteToFreq(notes[idx]);
  float env = expDecay(noteTime, 4.0);

  return sine(freq, time) * env;
}
```

## Effects

```glsl
// Reverb approximation (comb filter)
float reverb(float sample, float time, float delay, float feedback) {
  float echo = 0.0;
  float amp = feedback;
  for (int i = 1; i <= 4; i++) {
    // In a real implementation, you'd read from a delay buffer
    // This is a simplified model for single-pass shaders
    echo += amp * sin(440.0 * 6.28318 * (time - delay * float(i)));
    amp *= feedback;
  }
  return sample + echo;
}

// Distortion (soft clip)
float softClip(float x, float drive) {
  return tanh(x * drive);
}

// Low-pass filter approximation (one-pole)
// For real filtering, use multipass with state buffer
float lpf(float x, float prev, float cutoff) {
  float a = cutoff / (cutoff + 1.0);
  return prev + a * (x - prev);
}

// Chorus (pitch modulation)
float chorus(float time, float freq, float depth, float rate) {
  float mod = sin(rate * 6.28318 * time) * depth;
  return sine(freq * (1.0 + mod), time);
}
```

## Noise & Percussion

```glsl
// White noise
float noise(float time) {
  return fract(sin(time * 43758.5453) * 2.0) * 2.0 - 1.0;
}

// Kick drum (sine with pitch drop + noise)
float kick(float time) {
  float pitchEnv = exp(-time * 10.0);
  float ampEnv = exp(-time * 4.0);
  return sin(6.28318 * (160.0 * pitchEnv + 50.0) * time) * ampEnv;
}

// Snare (noise + tone)
float snare(float time) {
  float tone = sin(6.28318 * 200.0 * time) * exp(-time * 20.0);
  float ns = noise(time * 44100.0) * exp(-time * 8.0);
  return (tone * 0.3 + ns * 0.7) * exp(-time * 6.0);
}

// Hi-hat (filtered noise)
float hihat(float time) {
  return noise(time * 44100.0) * exp(-time * 40.0) * 0.3;
}

// Simple drum pattern
float drums(float time, float bpm) {
  float beatTime = 60.0 / bpm;
  float beat = mod(time, beatTime * 4.0);

  float k = kick(mod(beat, beatTime));
  float s = snare(mod(beat - beatTime * 2.0 + beatTime * 4.0, beatTime * 4.0) - beatTime * 2.0);
  float h = hihat(mod(beat, beatTime * 0.5));

  return k * 0.5 + s * 0.3 + h * 0.2;
}
```

## WebAudio Integration

To use shader-like synthesis in our template system, generate audio via WebAudio API:

```javascript
// In sceneSetup or sketchSetup:
const audioCtx = new AudioContext();
const processor = audioCtx.createScriptProcessor(4096, 0, 2);
let audioTime = 0;
const sampleRate = audioCtx.sampleRate;

processor.onaudioprocess = (e) => {
  const left = e.outputBuffer.getChannelData(0);
  const right = e.outputBuffer.getChannelData(1);

  for (let i = 0; i < left.length; i++) {
    const t = audioTime / sampleRate;
    // Your synthesis function here
    const sample = Math.sin(440 * 2 * Math.PI * t) * 0.3;
    left[i] = sample;
    right[i] = sample;
    audioTime++;
  }
};

processor.connect(audioCtx.destination);
// Note: start on user interaction (click) to comply with autoplay policy
```

For more advanced use, AudioWorklet is the modern replacement for ScriptProcessor.

## Audio-Reactive Visuals

Feed audio analysis back into visual parameters:

```javascript
const analyser = audioCtx.createAnalyser();
analyser.fftSize = 256;
const freqData = new Uint8Array(analyser.frequencyBinCount);

// In animation loop:
analyser.getByteFrequencyData(freqData);
const bass = freqData.slice(0, 4).reduce((a, b) => a + b) / (4 * 255);
const treble = freqData.slice(60, 80).reduce((a, b) => a + b) / (20 * 255);
// Pass bass/treble to shader uniforms
```

## Key References

- **Shadertoy sound examples** — search "mainSound" for community audio shaders
- **Syntopia** — shader-based music generation
- **Web Audio API** — MDN documentation for browser audio synthesis
- **The Audio Programming Book** — digital audio fundamentals
