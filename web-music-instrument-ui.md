# Web Music Instrument UI Research

Research on building web-based UIs for music instruments — synths, modular synths, drum machines — plus how to build beautiful reverb effects like Plateau in Cardinal.

*Compiled February 2026*

---

## 1. Core Web Audio Technologies

### Web Audio API
The foundation. `AudioContext` creates a graph of audio nodes:
- `OscillatorNode` — sine, square, sawtooth, triangle, custom via `PeriodicWave`
- `BiquadFilterNode` — lowpass, highpass, bandpass, notch, allpass, peaking, lowshelf, highshelf
- `GainNode` — amplitude control, used to build ADSR envelopes by scheduling `linearRampToValueAtTime()` / `exponentialRampToValueAtTime()`
- `AnalyserNode` — real-time FFT for oscilloscopes and spectrum analyzers
- `ConvolverNode` — convolution reverb with impulse responses
- `DelayNode` — up to 180s delay, essential for feedback effects
- `WaveShaperNode` — distortion via transfer function curves
- `StereoPannerNode` — simple left/right panning

### WebMIDI API
`navigator.requestMIDIAccess()` → `MIDIAccess` object with inputs/outputs maps. ~73% browser support (Chromium-based). **WEBMIDI.js** library simplifies: note events, CC messages, channel filtering. Connects hardware controllers (Launchpad, keyboards, knob boxes) directly to web synths.

### AudioWorklet
Custom DSP running on a dedicated audio thread:
- Extends `AudioWorkletProcessor` with a `process(inputs, outputs, parameters)` method
- Processes in 128-sample blocks (~3ms timing budget at 44.1kHz)
- Communicates with main thread via `MessagePort`
- Replaces deprecated `ScriptProcessorNode`
- Essential for custom reverb, distortion, granular synthesis

### WebAssembly (WASM)
Compile C++/Rust DSP code to near-native speed in the browser:
- SIMD support for parallel sample processing
- `wasm-bindgen` (Rust) or Emscripten (C++) for JS interop
- Run inside AudioWorklet for real-time DSP
- How Cardinal brings VCV Rack to the browser

### Web Audio Modules (WAM) 2.0
Open standard — "VST for the web." Interoperable plugin format:
- Host/plugin architecture with parameter discovery
- MIDI and audio I/O
- GUI separation from DSP
- ACM paper: dl.acm.org/doi/fullHtml/10.1145/3487553.3524225

---

## 2. Frameworks & Libraries

| Library | Purpose | Key Feature |
|---|---|---|
| **Tone.js** | High-level Web Audio framework | DAW-like transport, prebuilt synths/effects, PolySynth |
| **Faust** | DSP language → WASM | `reverbs.lib`, online IDE, TypeScript rewrite (2025) |
| **RNBO (Cycling '74)** | Max/MSP → Web export | Powers Ableton Learning Synths |
| **Elementary Audio** | Declarative JS audio | `@elemaudio/core` + `web-renderer` |
| **webaudio-controls** | WebComponent GUI parts | Knobs, sliders, switches, keyboards |
| **NexusUI** | HTML5 audio interfaces | dial, slider, keyboard, matrix, vinyl, tilt, 20+ controls |
| **Root** | Web audio UI components | Logarithmic response curves, 8 bespoke components |
| **Csound Web** | DSP language in browser | WASM compilation, HTML5 controls |
| **SuperCollider.js** | SC client for JS/TS | Node.js + browser support |

### Tone.js Details
```
npm install tone
```
- `Tone.Synth`, `Tone.FMSynth`, `Tone.AMSynth`, `Tone.PolySynth`
- `Tone.Transport` — global BPM, scheduling, looping
- Effects: `Tone.Reverb`, `Tone.Freeverb`, `Tone.JCReverb`, `Tone.Delay`, `Tone.Chorus`
- `Tone.Players` for sample-based instruments
- Works well with React, Svelte, Vue

### Faust Details
Functional DSP language that compiles to WASM, C++, LLVM:
- `reverbs.lib` includes: Schroeder, Moorer, Freeverb, Dattorro, JPverb, Greyhole, Keith Barr
- Online IDE: faustide.grame.fr
- TypeScript rewrite in 2025 improves web integration
- Can generate AudioWorklet processors directly

---

## 3. UI Components — How to Build Them

### Knobs (Rotary Encoders)

**SVG approach** — `AudioKnobs` (github.com/Megaemce/AudioKnobs)
- Moog-style SVG knobs with dynamic shadows
- Built with Inkscape, fully styleable
- Mouse and touch control

**CSS approach** — `conic-gradient()` for knob position indicator:
```css
.knob-indicator {
  background: conic-gradient(
    #E3C330 0deg,
    #E3C330 calc(var(--angle) * 1deg),
    transparent calc(var(--angle) * 1deg)
  );
  border-radius: 50%;
}
```

**Web Components:**
- `webaudio-controls` — framework-agnostic, sprite-based or SVG
- `html5-knob` (github.com/denilsonsa/html5-knob) — pure JS + SVG
- `control-knob` (github.com/slipmatio/control-knob) — Vue 3 + ARIA

**Interaction pattern:**
```js
knob.addEventListener('pointerdown', (e) => {
  knob.setPointerCapture(e.pointerId); // smooth off-target rotation
  const startY = e.clientY;
  const startValue = currentValue;

  const onMove = (e) => {
    const delta = startY - e.clientY; // drag up = increase
    currentValue = clamp(startValue + delta * sensitivity, min, max);
    requestAnimationFrame(() => updateKnobVisual(currentValue));
  };

  knob.addEventListener('pointermove', onMove);
  knob.addEventListener('pointerup', () => {
    knob.removeEventListener('pointermove', onMove);
  }, { once: true });
});
```

### Sliders / Faders
- `webaudio-controls` — horizontal and vertical
- AudioKit Controls — knobs, sliders, XYPads, joysticks
- Touch-friendly with `touch-action: none` CSS

### Patch Cables (Modular Synth)

**SVG Bezier curves** for cable routing:
```
B(t) = (1-t)³P0 + 3(1-t)²tP1 + 3(1-t)t²P2 + t³P3
```

Implementation:
- SVG overlay layer above modules with `pointer-events: none` (except on ports)
- Interactive connection points (ports/jacks) with `pointer-events: all`
- Drag-to-connect: mousedown on output port → draw temp cable → mouseup on input port
- Cable path updates when modules are dragged (recalculate control points)
- Signal-animated cables: stroke color/width pulses with audio amplitude

### Step Sequencer Grids

CSS Grid layout:
```css
.sequencer {
  display: grid;
  grid-template-columns: repeat(16, 1fr);
  grid-template-rows: repeat(4, 1fr); /* kick, snare, hihat, clap */
  gap: 2px;
}
.step { cursor: pointer; border-radius: 2px; }
.step.active { background: #E3C330; }
.step.playing { box-shadow: 0 0 8px #E3C330; }
```

- Click toggles `.active` class
- Transport highlights current step with `.playing`
- BPM sync via `Tone.Transport` or `setInterval` with `AudioContext.currentTime`
- Pattern arrays: `patterns[drumIndex][stepIndex] = true/false`

### Oscilloscope / Spectrum Analyzer

**Canvas 2D + AnalyserNode:**
```js
const analyser = audioCtx.createAnalyser();
analyser.fftSize = 2048;
const dataArray = new Uint8Array(analyser.frequencyBinCount);

function draw() {
  requestAnimationFrame(draw);
  analyser.getByteTimeDomainData(dataArray); // waveform
  // or: analyser.getByteFrequencyData(dataArray); // spectrum
  ctx.clearRect(0, 0, width, height);
  ctx.beginPath();
  dataArray.forEach((v, i) => {
    const x = (i / dataArray.length) * width;
    const y = (v / 255) * height;
    i === 0 ? ctx.moveTo(x, y) : ctx.lineTo(x, y);
  });
  ctx.stroke();
}
```

**WebGL** — for high-performance visualization:
- `gl-waveform` (github.com/dy/gl-waveform) — O(n) update, O(c) render
- Shader-based rendering for complex visualizations

**Peaks.js** (github.com/bbc/peaks.js) — BBC's waveform display library for audio file visualization with zoom and interaction.

### Piano Roll
- `webaudio-pianoroll` (github.com/g200kg/webaudio-pianoroll) — 4 edit modes
- `react-piano-roll` — flexible note representation
- `Reactronica` (reactronica.com) — multi-track sequencing + piano roll
- `react-piano` (github.com/kevinsqi/react-piano) — interactive keyboard

---

## 4. Modular Synth Architecture Patterns

### Drag-and-Drop Module Racks

**Libraries:**
- **React Flow** (reactflow.dev) — node-based UIs with drag-and-drop, best for React
- **Draggable JS** (shopify.github.io/draggable) — modular drag & drop by Shopify
- **interact.js** (interactjs.io) — drag, resize, multi-touch gestures
- **Gridstack.js** (gridstackjs.com) — grid-based drag-and-drop layouts
- **FormKit DnD** (drag-and-drop.formkit.com) — framework-agnostic, ~5KB gzipped

**Pattern:**
1. Container component = rack (drop target)
2. Module component = draggable unit with controls
3. Position state: `{ id, x, y, type, params }` per module
4. Collision detection + grid snapping for tidy layout
5. Connection state: `[{ from: {moduleId, portId}, to: {moduleId, portId} }]`

### Software Architecture

**Publish-Subscribe (EventBus):**
```ts
class EventBus {
  private listeners = new Map<string, Set<Function>>();
  on(event: string, cb: Function) { /* ... */ }
  emit(event: string, data: any) { /* ... */ }
}
// Modules communicate via events, not direct references
bus.emit('oscillator:frequency', 440);
bus.on('oscillator:frequency', (freq) => osc.frequency.value = freq);
```

**Plugin Architecture:**
- Dynamic module loading (lazy imports)
- Type-safe module interface: `{ inputs: Port[], outputs: Port[], params: Param[], process() }`
- Isolated logic per module

**Feature-Folder Structure:**
```
src/
  components/       # React/Vue/Svelte UI components
    Knob.tsx
    Slider.tsx
    ModuleRack.tsx
    PatchCable.tsx
  synth/            # Audio engine (no UI dependencies)
    Oscillator.ts
    Filter.ts
    Envelope.ts
    SynthEngine.ts
  hooks/            # Custom React hooks
    useAudio.ts
    useMIDI.ts
    useTransport.ts
  state/            # State management
    synthStore.ts
    patternStore.ts
    patchStore.ts
```

**State Management:**
- **Jotai** (jotai.org) — atomic state, optimized re-renders, great for many independent params
- **Zustand** (github.com/pmndrs/zustand) — minimalist hook-based stores
- Custom hooks: `useAudio()`, `useMIDI()`, `useAudioParameter()`, `useTransport()`

---

## 5. Visual Design — Skeuomorphic Audio UI

### CSS Techniques

**Metallic / brushed metal textures:**
```css
.panel {
  background:
    repeating-linear-gradient(90deg, transparent, rgba(255,255,255,0.03) 1px, transparent 2px),
    repeating-linear-gradient(0deg, transparent, rgba(0,0,0,0.02) 1px, transparent 3px),
    linear-gradient(135deg, #2a2a2a 0%, #1a1a1a 50%, #2a2a2a 100%);
}
```

**3D depth and lighting:**
- `perspective: 800px` on container
- `transform: rotateX(2deg)` for subtle tilt
- Multi-layer `box-shadow` for realistic depth:
  ```css
  box-shadow:
    0 1px 0 rgba(255,255,255,0.1) inset,  /* top highlight */
    0 -1px 0 rgba(0,0,0,0.3) inset,       /* bottom shadow */
    0 4px 12px rgba(0,0,0,0.5);            /* drop shadow */
  ```

**SVG filters for realistic bevels:**
```xml
<filter id="bevel">
  <feGaussianBlur in="SourceAlpha" stdDeviation="2" result="blur"/>
  <feOffset in="blur" dx="1" dy="1" result="offset"/>
  <feSpecularLighting in="offset" surfaceScale="5" specularConstant="0.8"
    specularExponent="20" lighting-color="#fff" result="specular">
    <fePointLight x="-5000" y="-10000" z="20000"/>
  </feSpecularLighting>
  <feComposite in="specular" in2="SourceAlpha" operator="in"/>
  <feComposite in="SourceGraphic" operator="arithmetic" k1="0" k2="1" k3="1" k4="0"/>
</filter>
```

**Performance:**
- GPU acceleration: `will-change: transform`, `transform: translate3d(0,0,0)`
- Avoid animating `width`, `height`, `top`, `left` — use `transform` instead
- Batch DOM reads/writes to prevent layout thrashing
- CSS custom properties for runtime theming: `--knob-color`, `--panel-bg`, `--accent`

### Accessibility

Essential for custom audio controls:
- `role="slider"` with `aria-valuemin`, `aria-valuemax`, `aria-valuenow`, `aria-label`
- `tabindex="0"` for keyboard focus
- Arrow keys: ±1 step. Shift+Arrow: ±10 steps. Home/End: min/max
- `aria-live="polite"` on value display for screen reader announcements
- High-contrast focus indicators (not just color)

---

## 6. Web-Based Reverb — Plateau / Dattorro Style

### Dattorro Plate Reverb Algorithm

*Jon Dattorro, "Effect Design Part 1: Reverberator and Other Filters" (1997)*
Paper: ccrma.stanford.edu/~dattorro/EffectDesignPart1.pdf

**Signal flow:**
```
Input → Pre-delay → Bandwidth EQ (lowpass)
  → Input Diffusion 1 (2x all-pass)
  → Input Diffusion 2 (2x all-pass)
  → Tank
    ├─ Left loop:  all-pass (LFO mod) → delay → damping (lowpass) → decay feedback
    └─ Right loop: all-pass (LFO mod) → delay → damping (lowpass) → decay feedback
  → Output taps (7 taps from various points in the tank)
```

**Key insight:** LFO-modulated all-pass filter delays prevent metallic ringing — the modulation breaks up periodic resonances, creating lush, organic-sounding decay.

**Parameters:**
| Parameter | Range | Effect |
|---|---|---|
| Pre-delay | 0–100ms | Time before reverb onset |
| Bandwidth | 0–1 | Input lowpass (darkens input) |
| Input Diffusion 1 | 0–1 | Smears early reflections |
| Input Diffusion 2 | 0–1 | Further smearing |
| Decay | 0–1 | Reverb tail length (0.99+ = near-infinite) |
| Decay Diffusion 1 | 0–1 | Tank diffusion |
| Decay Diffusion 2 | 0–1 | Tank diffusion |
| Damping | 0–1 | High-frequency absorption (natural rooms absorb highs) |
| Excursion Rate | 0.1–2 Hz | LFO speed for modulation |
| Excursion Depth | 0–16 samples | LFO depth |
| Wet/Dry | 0–1 | Mix |

### Web Implementations

**DattorroReverbNode** — github.com/khoin/DattorroReverbNode
- Production-ready AudioWorklet implementation
- Full Dattorro algorithm with modulation
- Drop-in for any Web Audio project

**@ondas/dattorro-reverb** — NPM package
- Cubic interpolation for smooth modulation
- Sample-rate independent (works at 44.1k, 48k, 96k)

**Faust reverbs.lib** — Complete reverb algorithm collection:
- `dm.dattorro_rev` — Dattorro plate
- `dm.freeverb_demo` — Freeverb
- `dm.jpverb_demo` — JPverb (lush algorithmic)
- `dm.greyhole_demo` — Greyhole (diffuse delays)
- `dm.zita_rev1` — Zita-Rev1 (high quality room)
- All compilable to AudioWorklet WASM

**MVerb** — github.com/martineastwood/mverb
- Single-file C++ implementation of Dattorro
- Easy to compile to WASM with Emscripten
- Clean, readable code for study

**Rust → WASM** path:
- Write reverb in Rust with `no_std` for minimal binary size
- Compile with `wasm-pack` + `wasm-bindgen`
- Run in AudioWorklet for real-time processing
- SIMD support for parallel sample processing

### Freeverb (Simpler Alternative)

Jezar's Freeverb — simpler than Dattorro, still sounds good:
- 8 parallel feedback comb filters → 4 series all-pass filters per channel
- Right channel: +23 samples offset for stereo decorrelation
- Parameters: room size, damping, wet/dry, width

Available as:
- NPM: `freeverb` (github.com/mmckegg/freeverb)
- Tone.js: `Tone.Freeverb` and `Tone.JCReverb`
- Faust: `dm.freeverb_demo`

### Shimmer Reverb

The Brian Eno / Valhalla Shimmer technique:
```
Input → Reverb → Output
           ↑         ↓
           └── Pitch Shift (+12 semitones) ←┘
```
- Feedback loop through reverb + pitch shifter creates eternally rising ethereal wash
- Pitch shift via `PitchShifterNode` or granular pitch shifting in AudioWorklet
- Key: keep feedback < 1.0 to prevent runaway gain

### Making Reverb Sound Beautiful

1. **Modulate all-pass filters** with slow LFOs (0.1–2 Hz) — chorusing effect prevents metallic sound
2. **Frequency-dependent damping** — high frequencies decay faster (natural room behavior)
3. **Stereo decorrelation** — offset delay line lengths between L/R channels
4. **Freeze / infinite sustain** — set decay to 1.0, mute input = frozen wash
5. **Proper input diffusion** — enough smearing that individual echoes aren't audible
6. **Pre-delay tuning** — 20–40ms for "room size" perception
7. **Multi-tap output** — 7+ output taps from different tank positions for rich spatial image

### Convolution vs Algorithmic for Web

| | Algorithmic | Convolution |
|---|---|---|
| CPU | Lighter | Heavier (FFT-based) |
| Download | No extra files | Large IR files (100KB–2MB each) |
| Parameters | Full real-time control | Only wet/dry, limited tweaking |
| Sound | Idealized, clean | Realistic room capture |
| Best for | Synths, effects, creative | Realistic room simulation |

**Verdict:** Algorithmic preferred for web synths — lighter, more controllable, no downloads.

---

## 7. Notable Open-Source Projects to Study

| Project | URL | Notes |
|---|---|---|
| **Cardinal** | cardinal.kx.studio | VCV Rack in browser via WASM, includes Plateau reverb |
| **Viktor NV-1** | nicroto.github.io/viktor | 3 oscillators, MIDI, patches, WebAudio |
| **web-synth** | github.com/ameobea/web-synth | Browser DAW with modular patching, live looping |
| **Patchcab** | github.com/spectrome/patchcab | Eurorack-style Web Audio synth |
| **Zupiter** | z.musictools.live | Web modular synth, shareable patches |
| **web-audio-synth** | github.com/Smilebags/web-audio-synth | MIDI input, mic, WAV export |
| **Dittytoy** | dittytoy.net | Code-driven music playground |
| **Chrome Music Lab** | musiclab.chromeexperiments.com | Educational Web Audio experiments |
| **Ableton Learning Synths** | learningsynths.ableton.com | Built with RNBO |
| **Strudel** | strudel.cc | TidalCycles in browser |
| **Gibber** | gibber.cc | Live coding, FM/granular synth |
| **openDAW** | github.com/andremichelle/openDAW | Open source DAW |
| **DattorroReverbNode** | github.com/khoin/DattorroReverbNode | Production AudioWorklet reverb |
| **MVerb** | github.com/martineastwood/mverb | Single-file C++ Dattorro reverb |
| **Faust Reverbs** | github.com/LucaSpanedda/Digital_Reverberation_in_Faust | Reverb experiments |
| **AudioKnobs** | github.com/Megaemce/AudioKnobs | Moog-style SVG knobs |
| **modularsynth** | github.com/mattybrad/modularsynth | JS modular synth + Arduino HW |

---

## 8. Key Academic References

- **Jon Dattorro** — "Effect Design Part 1: Reverberator and Other Filters" (1997) — ccrma.stanford.edu/~dattorro/EffectDesignPart1.pdf
- **CCRMA** — Physical Audio Signal Processing — dsprelated.com/freebooks/pasp/
- **CCRMA** — Shimmer Audio Effect paper
- **Valhalla DSP** — Schroeder Reverbs blog series (valhalladsp.com/2009/05/30/schroeder-reverbs-the-beginning/)
- **Web Audio Modules 2.0** — ACM paper — dl.acm.org/doi/fullHtml/10.1145/3487553.3524225
- **Julius O. Smith** — Mathematics of the DFT / Spectral Audio Signal Processing — ccrma.stanford.edu/~jos/

---

## 9. Curated Resource Lists

- github.com/notthetup/awesome-webaudio — Web Audio API resources
- github.com/olilarkin/awesome-musicdsp — Music DSP resources
- github.com/BillyDM/awesome-audio-dsp — Audio DSP learning resources
- github.com/alemangui/web-audio-resources — Web audio resources
- webaudio.github.io/demo-list — W3C Audio Working Group demo list
- github.com/clarkio/awesome-web-audio — Another Web Audio collection
