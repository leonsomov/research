# Building Interactive Instruments, Modular Synthesizers, Drum Machines, and Effects for the Web

A comprehensive research document covering the Web Audio API ecosystem, frameworks, synthesis techniques, UI patterns, and notable projects for building browser-based audio applications.

---

## Table of Contents

1. [Web Audio API Fundamentals](#1-web-audio-api-fundamentals)
2. [Frameworks and Libraries](#2-frameworks-and-libraries)
3. [Building Modular Synth Modules](#3-building-modular-synth-modules)
4. [Building Drum Machines](#4-building-drum-machines)
5. [Building Effects Processors](#5-building-effects-processors)
6. [UI Patterns for Audio Apps](#6-ui-patterns-for-audio-apps)
7. [Performance Optimization](#7-performance-optimization)
8. [Notable Web Audio Projects](#8-notable-web-audio-projects)

---

## 1. Web Audio API Fundamentals

### AudioContext and the Audio Node Graph

The Web Audio API is a high-level JavaScript API for processing and synthesizing audio in web applications. At its core is the **AudioContext**, which represents an audio-processing graph built from interconnected **AudioNode** objects.

```
[Source Nodes] --> [Processing Nodes] --> [Destination Node (speakers)]
```

Key concepts:

- **AudioContext**: The central object that manages and plays all sounds. Created via `new AudioContext()`. It exposes `currentTime` (a high-precision hardware clock), `sampleRate`, and the `destination` node (speakers).
- **AudioNode**: The building block of the audio graph. Each node has inputs and/or outputs and performs a specific function (oscillator, filter, gain, etc.).
- **Connections**: Nodes are connected via `node.connect(destination)`. The graph is directed and acyclic. A single output can fan out to multiple inputs, and multiple outputs can merge into a single input.

Built-in node types include:

| Category | Nodes |
|----------|-------|
| **Sources** | `OscillatorNode`, `AudioBufferSourceNode`, `MediaElementAudioSourceNode`, `MediaStreamAudioSourceNode` |
| **Processing** | `GainNode`, `BiquadFilterNode`, `ConvolverNode`, `DynamicsCompressorNode`, `WaveShaperNode`, `DelayNode`, `StereoPannerNode`, `PannerNode` |
| **Analysis** | `AnalyserNode` |
| **Routing** | `ChannelSplitterNode`, `ChannelMergerNode` |
| **Custom** | `AudioWorkletNode` |

Minimal example:

```javascript
const ctx = new AudioContext();
const osc = ctx.createOscillator();
const gain = ctx.createGain();

osc.type = 'sawtooth';
osc.frequency.value = 440;
gain.gain.value = 0.3;

osc.connect(gain);
gain.connect(ctx.destination);
osc.start();
```

### AudioWorklet for Custom DSP

**AudioWorklet** replaced the deprecated `ScriptProcessorNode` and is the modern mechanism for custom audio processing. It runs on a dedicated audio rendering thread, providing low-latency, glitch-free processing.

Architecture:

1. **Main Thread**: Create an `AudioWorkletNode` and add it to the graph.
2. **Audio Thread**: An `AudioWorkletProcessor` class runs inside `AudioWorkletGlobalScope`, processing 128-sample render quanta at the audio sample rate.

```javascript
// main.js
await ctx.audioWorklet.addModule('my-processor.js');
const myNode = new AudioWorkletNode(ctx, 'my-processor');
myNode.connect(ctx.destination);

// my-processor.js
class MyProcessor extends AudioWorkletProcessor {
  process(inputs, outputs, parameters) {
    const output = outputs[0];
    for (let channel = 0; channel < output.length; channel++) {
      for (let i = 0; i < output[channel].length; i++) {
        output[channel][i] = Math.random() * 2 - 1; // white noise
      }
    }
    return true; // keep alive
  }
}
registerProcessor('my-processor', MyProcessor);
```

Key features:

- **AudioParam support**: Custom parameters declared via a static `parameterDescriptors` getter, supporting both `a-rate` (per-sample) and `k-rate` (per-block) automation.
- **MessagePort**: Bidirectional communication between the main thread and the processor for sending configuration, waveform data, etc.
- **No DOM access**: The worklet runs in a minimal global scope with no DOM, `setTimeout`, or `fetch`.

Upcoming improvements (expected in the next Web Audio API revision, targeting Q4 2026):

- **Configurable Render Quantum**: Allow developers to configure the size of the render quantum beyond the fixed 128 samples.
- **`performance.now()` in AudioWorklet**: High-resolution timer availability within the audio thread.

### WebAssembly for Performance-Critical DSP

WebAssembly (WASM) enables near-native performance for computationally intensive DSP operations inside AudioWorklet processors.

**Why WASM for audio?**

- Deterministic execution time (no garbage collection pauses)
- Access to SIMD instructions for vectorized math
- Ability to port existing C/C++/Rust DSP code to the web
- Consistent performance characteristics

**Integration pattern**: AudioWorklet + WASM

```javascript
// processor.js (runs on audio thread)
class WasmProcessor extends AudioWorkletProcessor {
  constructor() {
    super();
    // WASM module is instantiated here or passed via port
  }

  process(inputs, outputs) {
    // Pass audio buffer pointers to WASM
    // WASM processes samples in-place
    // Copy results to outputs
    return true;
  }
}
```

**Toolchains**:

- **Emscripten**: Compile C/C++ to WASM. Includes `wasm_audio_worklets` API for direct AudioWorklet integration.
- **wasm-pack / wasm-bindgen**: Compile Rust to WASM with JavaScript bindings.
- **AssemblyScript**: TypeScript-like language that compiles to WASM.

### WebMIDI API for Hardware Controller Integration

The Web MIDI API enables browser-based applications to communicate with MIDI hardware: keyboards, drum pads, knob controllers, and any other MIDI-capable device.

```javascript
const midi = await navigator.requestMIDIAccess();

// List inputs
for (const input of midi.inputs.values()) {
  console.log(input.name);
  input.onmidimessage = (event) => {
    const [status, note, velocity] = event.data;
    // status: 0x90 = note on, 0x80 = note off, 0xB0 = CC
    console.log(`Status: ${status}, Note: ${note}, Velocity: ${velocity}`);
  };
}

// Send MIDI output
for (const output of midi.outputs.values()) {
  output.send([0x90, 60, 127]); // Note On, Middle C, max velocity
}
```

**Browser support**: Chrome, Edge, Opera, and Brave support WebMIDI. Firefox has it behind a flag. Safari does not support it yet.

**Helper libraries**:

- **WEBMIDI.js** (`webmidi` npm package): High-level abstraction with methods like `playNote()`, `sendPitchBend()`, and event listeners for `noteon`, `controlchange`, etc.
- **@AkikoKumagara/web-midi-api**: TypeScript-first WebMIDI wrapper.

The W3C published a working draft of the WebMIDI specification in January 2025.

### Scheduling and Timing (Lookahead Pattern)

Precise timing is critical for musical applications. The Web Audio API provides two distinct clocks:

1. **`AudioContext.currentTime`**: Hardware-driven, high-precision clock (sample-accurate). Used for scheduling audio events.
2. **`setTimeout` / `setInterval`**: JavaScript event loop timer, subject to jitter (typically 4-10ms variance).

**The problem**: You cannot use `setTimeout` alone for musical timing (it drifts and jitters), and you cannot use `AudioContext.currentTime` alone (you cannot set callbacks on it).

**The solution: Lookahead scheduling**

The canonical pattern, described in Chris Wilson's "A Tale of Two Clocks":

```javascript
const LOOKAHEAD = 25.0;      // ms - how often the scheduler runs
const SCHEDULE_AHEAD = 0.1;  // seconds - how far ahead to schedule

let nextNoteTime = 0.0;
let currentNote = 0;
let tempo = 120;

function scheduler() {
  while (nextNoteTime < ctx.currentTime + SCHEDULE_AHEAD) {
    scheduleNote(currentNote, nextNoteTime);
    advanceNote();
  }
  setTimeout(scheduler, LOOKAHEAD);
}

function scheduleNote(note, time) {
  // Schedule audio events using the precise audio clock
  const osc = ctx.createOscillator();
  osc.connect(ctx.destination);
  osc.start(time);
  osc.stop(time + 0.05);
}

function advanceNote() {
  const secondsPerBeat = 60.0 / tempo;
  nextNoteTime += secondsPerBeat / 4; // 16th notes
  currentNote = (currentNote + 1) % 16;
}
```

Key principles:

- **Decouple scheduling rate from lookahead**: The `setTimeout` interval should be shorter than the lookahead window to provide overlap/safety margin.
- **Schedule ahead**: Always look into the near future so that even if a `setTimeout` callback is late, audio events are already queued.
- **Use `AudioParam.setValueAtTime()`** and related methods (`linearRampToValueAtTime`, `exponentialRampToValueAtTime`) for sample-accurate parameter changes.

---

## 2. Frameworks and Libraries

### Tone.js -- High-Level Music Framework

- **URL**: https://tonejs.github.io/
- **GitHub**: https://github.com/Tonejs/Tone.js (~13.9k stars)
- **License**: MIT

Tone.js is the most widely used high-level Web Audio framework. It provides a complete toolkit for music creation.

**Core features**:

- **Synths**: `Synth`, `FMSynth`, `AMSynth`, `PolySynth`, `MonoSynth`, `MetalSynth`, `PluckSynth`, `MembraneSynth`, `NoiseSynth`, `Sampler`
- **Effects**: Reverb, Delay, Chorus, Phaser, Tremolo, Vibrato, Distortion, BitCrusher, PingPongDelay, FeedbackDelay, Freeverb, JCReverb, AutoFilter, AutoPanner, AutoWah, Compressor, EQ3, Limiter
- **Transport**: Global clock with BPM, time signatures, looping, scheduling callbacks at musical positions (`"1:2:3"` notation)
- **Signals**: Audio-rate signal math (`Add`, `Multiply`, `Scale`, `Signal`)
- **Patterns**: `Pattern`, `Sequence`, `Loop`, `Part` for musical sequencing
- **Sources**: `Player`, `GrainPlayer`, `Oscillator`, `Noise`, `UserMedia`
- **Time abstractions**: Notation time (`"4n"`, `"8t"`), transport time (`"1:2:0"`), frequency (`"C4"`)

```javascript
import * as Tone from 'tone';

const synth = new Tone.PolySynth(Tone.Synth).toDestination();
const seq = new Tone.Sequence((time, note) => {
  synth.triggerAttackRelease(note, '8n', time);
}, ['C4', 'E4', 'G4', 'B4'], '4n');

Tone.Transport.bpm.value = 128;
Tone.Transport.start();
seq.start(0);
```

**Strengths**: Mature, well-documented, large community, covers most common music production needs.
**Weaknesses**: Larger bundle size, abstractions can hide underlying Web Audio behavior, performance ceiling for complex patches.

### Elementary Audio -- Functional Reactive Audio Programming

- **URL**: https://www.elementary.audio/
- **GitHub**: https://github.com/elemaudio/elementary
- **License**: MIT

Elementary takes a functional, declarative approach to audio programming. You describe your audio graph as a pure function of your application state; the library efficiently diffs and updates the underlying audio engine.

**Core concepts**:

- **Rendering functions**: Composable functions that return audio graph descriptions.
- **Core library** (`@elemaudio/core`): Signal processing primitives -- `el.cycle` (sine), `el.saw`, `el.square`, `el.triangle`, `el.biquad`, `el.delay`, `el.convolve`, `el.compress`, etc.
- **Web renderer** (`@elemaudio/web-renderer`): Runs in the browser via AudioWorklet.
- **Node renderer** (`@elemaudio/offline-renderer`): Runs in Node.js for offline processing.
- **Plugin renderer**: Build native VST/AU plugins from the same code.

```javascript
import { el } from '@elemaudio/core';
import WebRenderer from '@elemaudio/web-renderer';

const core = new WebRenderer();

// Describe audio as a function
function synth(freq, gate) {
  const env = el.adsr(0.01, 0.1, 0.7, 0.3, gate);
  const osc = el.saw(freq);
  const filtered = el.lowpass(el.mul(env, 2000), 1.0, osc);
  return el.mul(env, filtered);
}

core.render(synth(440, 1), synth(440, 1)); // stereo
```

**Strengths**: Elegant functional API, efficient diffing engine, cross-platform (web + native plugins), excellent for reactive UI frameworks (React, Svelte).
**Weaknesses**: Smaller community, steeper conceptual learning curve for imperative-minded developers.

### Faust -- DSP Language Compiling to WebAssembly/AudioWorklet

- **URL**: https://faust.grame.fr/
- **GitHub**: https://github.com/grame-cncm/faust
- **License**: GPL-2.0

Faust (Functional Audio Stream) is a domain-specific language for signal processing developed at GRAME (Lyon, France). It compiles to highly optimized C++, LLVM, Rust, and crucially, **WebAssembly + AudioWorklet**.

**How it works**:

1. Write DSP code in Faust's block-diagram algebra.
2. The Faust compiler generates optimized WebAssembly.
3. The generated code runs inside an AudioWorklet in the browser.
4. The Faust Web IDE (https://faustide.grame.fr/) lets you edit, compile, and run Faust programs entirely in the browser.

```faust
// Simple resonant lowpass filter
import("stdfaust.lib");
freq = hslider("frequency", 1000, 20, 20000, 1);
q = hslider("Q", 1, 0.1, 10, 0.01);
process = fi.resonlp(freq, q, 1);
```

**Key features**:

- **faust2webaudio**: Architecture file that generates a complete Web Audio node.
- **faust2wasm**: Generates standalone WASM modules.
- **Faust Libraries**: Extensive standard library (`stdfaust.lib`) with filters, oscillators, effects, physical models, etc.
- **PWA support**: Faust programs can be deployed as Progressive Web Apps.

**Strengths**: Decades of DSP research behind the library, mathematically rigorous, generates highly optimized code, excellent for complex DSP algorithms.
**Weaknesses**: Requires learning a new language, compilation step adds complexity, smaller web-specific community.

### RNBO (Max/MSP) -- Export Max Patches to Web

- **URL**: https://rnbo.cycling74.com/
- **Developer**: Cycling '74

RNBO (pronounced "rainbow") is a library and export toolchain within Max that lets you create instruments and effects as Max patches and deploy them to multiple targets including the **web** (via WebAssembly), VST/AU plugins, Raspberry Pi, and C++ source code.

**Web export workflow**:

1. Build your DSP in Max using `rnbo~` objects.
2. Export to Web target -- generates WASM + JS wrapper.
3. Embed the RNBO engine into any website. The exported JavaScript API exposes parameters, MIDI input, and audio I/O.

```javascript
// Loading an RNBO device in the browser
import { createDevice } from '@rnbo/js';

const patchUrl = 'export/patch.export.json';
const device = await createDevice({ context: audioContext, patchUrl });
device.node.connect(audioContext.destination);

// Set parameters
device.parametersById.get('cutoff').value = 0.5;
```

**Strengths**: Visual patching environment (Max), battle-tested DSP from decades of Max/MSP development, no web programming needed for DSP, robust parameter and MIDI handling.
**Weaknesses**: Requires Max license (commercial), generated WASM bundles can be large, less flexibility for web-specific optimizations.

### WAM 2.0 (Web Audio Modules) -- Plugin Standard for Web

- **URL**: https://www.webaudiomodules.com/
- **GitHub**: https://github.com/webaudiomodules
- **Spec paper**: https://dl.acm.org/doi/fullHtml/10.1145/3487553.3524225

WAM 2.0 is an open standard for Web Audio plugins, analogous to VST/AU for desktop DAWs. It defines a common API for plugins and hosts to interoperate.

**Architecture**:

- **WAMPlugin**: The plugin interface -- exposes parameters, handles MIDI, processes audio.
- **WAMHost**: The host (DAW) interface -- loads plugins, routes audio, manages state.
- **WAMProcessor**: Runs on the audio thread (AudioWorkletProcessor).
- **WAMController**: Runs on the main thread, handles UI.

**Key features**:

- High-priority audio thread access
- Sample-accurate event scheduling
- TypeScript definitions
- Extensions for video, 3D, WebGL shaders, parameter modulation between WAMs
- SDK with examples and ready-to-use plugins

**Ecosystem**:

- **Amped Studio**: Commercial web DAW that hosts WAM 2.0 plugins.
- **FAUST**: Can generate WAM-compatible output.
- **Plugin collections**: Open-source repositories with dozens of effects and instruments.

**Strengths**: Industry-backed standard, interoperability between plugins and hosts, professional audio thread scheduling.
**Weaknesses**: Smaller adoption compared to desktop plugin formats, requires both plugin and host to implement the standard.

### Csound -- Classic DSP Language with Web Export

- **URL**: https://csound.com/wasm/
- **GitHub**: https://github.com/csound/csound

Csound is one of the oldest and most comprehensive sound synthesis languages, originally written in 1985. It now runs in the browser via WebAssembly.

**Web implementation**:

- **WebAudio Csound** (`@csound/browser`): JavaScript/TypeScript library for running Csound in the browser via AudioWorklet + WASM.
- **Csound Web IDE**: https://ide.csound.com/ -- Edit and run Csound code directly in the browser.
- **Csound 7 Playground**: The latest version (Csound 7, announced September 2025) is available in the web IDE with new language features.

```javascript
import { Csound } from '@csound/browser';

const csound = await Csound({ audioContext: ctx });
await csound.compileCsdText(`
<CsoundSynthesizer>
<CsOptions>
-odac
</CsOptions>
<CsInstruments>
sr = 48000
ksmps = 32
nchnls = 2
instr 1
  aout vco2 0.3, 440
  outs aout, aout
endin
</CsInstruments>
<CsScore>
i 1 0 2
</CsScore>
</CsoundSynthesizer>
`);
await csound.start();
```

**Strengths**: Thousands of opcodes, 40 years of DSP algorithms, extensive documentation and academic resources, stable and battle-tested.
**Weaknesses**: Older syntax paradigm, large WASM binary, steeper learning curve for web developers.

### SuperCollider -- via scsynth.wasm (SuperSonic)

- **URL**: https://sonic-pi.net/supersonic/demo.html
- **GitHub**: https://github.com/samaaron/supersonic
- **scsynth-wasm-builds**: https://github.com/rd--/scsynth-wasm-builds

SuperCollider's `scsynth` audio engine has been compiled to WebAssembly, making it available in the browser as **SuperSonic** (developed by Sam Aaron, creator of Sonic Pi).

**How it works**:

- The C++ codebase of `scsynth` (SuperCollider's synthesis server) is compiled to WASM using Emscripten.
- Runs in NRT (non-real-time) mode within an AudioWorklet context.
- OSC-based API for triggering synths with sample-accurate timing.
- Dennis Scheiba submitted a pull request in late 2024 to make WebAssembly an official SuperCollider compile target.

**Strengths**: Vast library of UGens (unit generators), powerful SynthDef architecture, decades of computer music research.
**Weaknesses**: Experimental/prototype status, large WASM binary, NRT mode has inherent limitations.

### Framework Comparison Matrix

| Feature | Tone.js | Elementary | Faust | RNBO | WAM 2.0 | Csound | SuperSonic |
|---------|---------|------------|-------|------|---------|--------|------------|
| **Language** | JavaScript | JavaScript | Faust DSL | Visual (Max) | JS/TS + WASM | Csound | SynthDef |
| **Paradigm** | Imperative OOP | Functional reactive | Functional | Visual patching | Plugin API | Score + Orchestra | Server/Client |
| **Custom DSP** | Via AudioWorklet | Built-in | Native | Via Gen~ | Via AudioWorklet | 1500+ opcodes | 700+ UGens |
| **WASM support** | No (pure JS) | Yes | Yes (compiles to) | Yes (exports to) | Yes | Yes | Yes |
| **Plugin format** | No | Yes (native) | Yes (many) | Yes (VST/AU/Web) | Yes (WAM) | No | No |
| **Bundle size** | ~200KB | ~150KB | Varies | Varies | Varies | ~2MB | ~3MB+ |
| **Learning curve** | Low | Medium | High | Medium (if you know Max) | Medium | High | High |
| **Community** | Very large | Growing | Academic + music | Max community | Niche | Academic | Academic |
| **Best for** | Rapid prototyping, apps | Reactive UI apps | Optimized DSP | Max users going web | Plugin ecosystems | Academic/research | Experimental |

---

## 3. Building Modular Synth Modules

### VCO (Voltage-Controlled Oscillator)

The oscillator is the fundamental sound source. The Web Audio API provides `OscillatorNode` with built-in waveforms.

**Basic waveforms**:

```javascript
const osc = ctx.createOscillator();
osc.type = 'sine';      // pure tone, no harmonics
osc.type = 'sawtooth';  // all harmonics, bright and buzzy
osc.type = 'square';    // odd harmonics, hollow and reedy
osc.type = 'triangle';  // odd harmonics (weaker), soft and mellow
osc.frequency.value = 440; // Hz
```

**Wavetable synthesis**:

Custom waveforms can be created using `PeriodicWave`:

```javascript
const real = new Float32Array([0, 0.5, 0.3, 0.1, 0.05]); // cosine terms
const imag = new Float32Array([0, 0, 0, 0, 0]);           // sine terms
const wave = ctx.createPeriodicWave(real, imag);
osc.setPeriodicWave(wave);
```

For advanced wavetable synthesis (morphing between tables, like Serum or Vital), you need an AudioWorklet:

```javascript
// In the processor:
// - Store multiple wavetables as Float32Arrays
// - Interpolate between tables based on a "position" parameter
// - Use phase accumulator to read samples from the current table
// - Linearly interpolate between adjacent samples for anti-aliasing
```

**FM (Frequency Modulation) synthesis**:

FM synthesis is achieved by connecting one oscillator's output to another's frequency input:

```javascript
const carrier = ctx.createOscillator();
const modulator = ctx.createOscillator();
const modGain = ctx.createGain();

carrier.frequency.value = 440;    // carrier frequency
modulator.frequency.value = 440;  // C:M ratio 1:1
modGain.gain.value = 200;         // modulation depth (Hz)

modulator.connect(modGain);
modGain.connect(carrier.frequency);
carrier.connect(ctx.destination);

carrier.start();
modulator.start();
```

For more complex FM (multi-operator, feedback), an AudioWorklet is recommended. Casey Primozic's FM synth demonstrates Rust + WASM + SIMD for browser-based FM synthesis: https://cprimozic.net/blog/fm-synth-rust-wasm-simd/

### VCF (Voltage-Controlled Filter)

Filters shape the harmonic content of the sound. The Web Audio API provides `BiquadFilterNode` with several types:

```javascript
const filter = ctx.createBiquadFilter();
filter.type = 'lowpass';   // removes frequencies above cutoff
filter.type = 'highpass';  // removes frequencies below cutoff
filter.type = 'bandpass';  // passes a band of frequencies
filter.type = 'notch';     // removes a band of frequencies
filter.type = 'allpass';   // changes phase relationships
filter.type = 'peaking';   // boost/cut at frequency
filter.type = 'lowshelf';  // boost/cut below frequency
filter.type = 'highshelf'; // boost/cut above frequency

filter.frequency.value = 1000; // cutoff frequency
filter.Q.value = 5;            // resonance (0.0001 to 1000)
filter.gain.value = 0;         // gain in dB (for peaking/shelf types)
```

**State-variable filter (SVF)**:

For a more flexible, self-oscillating filter with simultaneous LP/HP/BP/Notch outputs, implement an SVF in AudioWorklet:

```javascript
// SVF algorithm (per sample):
// hp = input - (2 * q * bp) - lp
// bp = hp * f + bp_prev
// lp = bp * f + lp_prev
// notch = hp + lp
// where f = 2 * sin(PI * cutoff / sampleRate)
```

**Filter modulation** is key to classic synth sounds. Connecting an envelope or LFO to `filter.frequency` via `AudioParam`:

```javascript
// Envelope controls filter cutoff
filter.frequency.setValueAtTime(200, ctx.currentTime);
filter.frequency.linearRampToValueAtTime(4000, ctx.currentTime + 0.1);
filter.frequency.exponentialRampToValueAtTime(200, ctx.currentTime + 0.5);
```

### VCA (Voltage-Controlled Amplifier)

A VCA is simply a `GainNode` with its `gain` parameter controlled by an envelope, LFO, or other modulation source.

```javascript
const vca = ctx.createGain();
vca.gain.value = 0; // start silent

// Tremolo: modulate gain with LFO
const lfo = ctx.createOscillator();
const lfoGain = ctx.createGain();
lfo.frequency.value = 5; // 5 Hz tremolo
lfoGain.gain.value = 0.3; // depth

lfo.connect(lfoGain);
lfoGain.connect(vca.gain);
lfo.start();
```

### Envelope Generators (ADSR)

ADSR envelopes control the amplitude or filter cutoff over the lifecycle of a note.

```javascript
function triggerADSR(param, attack, decay, sustain, release, time) {
  param.cancelScheduledValues(time);
  param.setValueAtTime(0, time);
  // Attack
  param.linearRampToValueAtTime(1.0, time + attack);
  // Decay
  param.linearRampToValueAtTime(sustain, time + attack + decay);
  // Sustain holds until release is triggered
}

function releaseADSR(param, release, time) {
  param.cancelScheduledValues(time);
  param.setValueAtTime(param.value, time);
  param.linearRampToValueAtTime(0, time + release);
}
```

For multi-stage envelopes (beyond ADSR), use `setValueCurveAtTime()` or implement custom envelope shapes in an AudioWorklet.

**Important**: Use `setTargetAtTime()` for exponential (more natural-sounding) curves instead of `linearRampToValueAtTime()`:

```javascript
// More natural-sounding attack (exponential approach)
param.setTargetAtTime(1.0, time, attack / 3); // time constant = attack/3
```

### LFO (Low-Frequency Oscillator)

LFOs are modulation sources typically operating below 20 Hz. In Web Audio, they are simply `OscillatorNode` instances at low frequencies connected to `AudioParam` targets.

```javascript
const lfo = ctx.createOscillator();
const lfoDepth = ctx.createGain();

lfo.type = 'sine';
lfo.frequency.value = 4; // 4 Hz
lfoDepth.gain.value = 50; // modulation depth

lfo.connect(lfoDepth);
lfoDepth.connect(filter.frequency); // filter wobble
// or: lfoDepth.connect(osc.frequency); // vibrato
// or: lfoDepth.connect(vca.gain);      // tremolo
lfo.start();
```

For more complex LFO shapes (sample-and-hold, random, stepped), use AudioWorklet.

### Sequencers

**Step sequencer**:

```javascript
class StepSequencer {
  constructor(steps = 16, bpm = 120) {
    this.steps = new Array(steps).fill(null);
    this.bpm = bpm;
    this.currentStep = 0;
    this.isPlaying = false;
  }

  setStep(index, note) {
    this.steps[index] = note;
  }

  // Use lookahead scheduling (see Section 1)
  tick(time) {
    const note = this.steps[this.currentStep];
    if (note) {
      this.triggerNote(note, time);
    }
    this.currentStep = (this.currentStep + 1) % this.steps.length;
  }
}
```

**Euclidean rhythms**:

Euclidean rhythms distribute `k` beats as evenly as possible across `n` steps. The algorithm is based on Bjorklund's algorithm (related to Euclid's algorithm for GCD).

```javascript
function euclidean(steps, pulses) {
  const pattern = [];
  let counts = new Array(pulses).fill(1);
  let remainders = new Array(steps - pulses).fill(0);

  // Bjorklund's algorithm
  while (remainders.length > 1) {
    const newCounts = [];
    const newRemainders = [];
    const min = Math.min(counts.length, remainders.length);

    for (let i = 0; i < min; i++) {
      newCounts.push([...counts[i] || [counts[i]], ...remainders[i] || [remainders[i]]]);
    }
    // ... (full implementation distributes remainders iteratively)
  }
  return pattern; // e.g., euclidean(8, 3) => [1,0,0,1,0,0,1,0]
}
```

Common Euclidean patterns that correspond to world music rhythms:

- `E(3,8)` = Cuban tresillo
- `E(5,8)` = Cuban cinquillo
- `E(7,16)` = Brazilian samba
- `E(5,12)` = South African Venda rhythm

### Effects

See [Section 5](#5-building-effects-processors) for detailed implementations.

### Modular Patching in the Browser

The key challenge of browser-based modular synthesis is implementing a flexible **patching system** -- connecting module outputs to module inputs with virtual cables.

**Architecture patterns**:

1. **Direct Web Audio connections**: Use `node.connect()` / `node.disconnect()`. Simplest approach, limited to Web Audio's built-in routing.
2. **Virtual signal bus**: AudioWorklet-based signal router that manages connections in a centralized processor.
3. **Control-rate patching**: Use `MessagePort` or `SharedArrayBuffer` for parameter-level connections (LFO -> cutoff, envelope -> gain).

**Virtual cable UI** (see Section 6 for details):

- SVG `<path>` elements with cubic Bezier curves for a natural hanging-cable look.
- Canvas-based rendering for better performance with many cables.
- HTML5 drag-and-drop for connecting jacks.

### Notable Browser-Based Modular Projects

**Zupiter**
- URL: https://z.musictools.live/
- Description: Web-based modular synthesizer inspired by Pure Data and Max/MSP. Written entirely in JavaScript using Web Audio and Web MIDI APIs. Build synths by creating nodes and connecting them. Play via computer keyboard, built-in sequencer, or external MIDI controller.
- Blog post: https://pointersgonewild.com/2019/10/06/zupiter-a-web-based-modular-synthesizer/

**Patchcab**
- URL: https://patch.cab
- GitHub: https://github.com/spectrome/patchcab
- Description: Modular Eurorack-style synthesizer built with Tone.js and Svelte. Inspired by VCV Rack. Community-made modules can be shared and remixed.

**NoiseCraft**
- GitHub: https://github.com/maximecb/noisecraft
- Description: Free, open-source visual programming language and synthesis platform. Runs in Chrome/Edge. Built on Web Audio and Web MIDI APIs.

**Web Audio Synth**
- URL: https://smilebags.github.io/web-audio-synth/
- GitHub: https://github.com/Smilebags/web-audio-synth
- Description: Eurorack-inspired modular synth with AudioWorklet-based modules. Chrome/Edge only.

**Web Synth (by Ameobea)**
- GitHub: https://github.com/Ameobea/web-synth
- Description: Browser-based DAW and audio synthesis platform with dozens of effects, synths, and modules. Supports modular-style patching, dynamic custom code, and live looping. Built with Rust + WASM.

---

## 4. Building Drum Machines

### Sample-Based Approach

The most straightforward method: load audio files and trigger them on a grid.

```javascript
async function loadSample(url) {
  const response = await fetch(url);
  const arrayBuffer = await response.arrayBuffer();
  return await ctx.decodeAudioData(arrayBuffer);
}

function playSample(buffer, time, gain = 1.0) {
  const source = ctx.createBufferSource();
  const gainNode = ctx.createGain();
  source.buffer = buffer;
  gainNode.gain.value = gain;
  source.connect(gainNode);
  gainNode.connect(ctx.destination);
  source.start(time);
  return source;
}

// Usage
const kick = await loadSample('/samples/kick.wav');
const snare = await loadSample('/samples/snare.wav');
const hihat = await loadSample('/samples/hihat.wav');

// Trigger on beat
playSample(kick, ctx.currentTime);
```

**Velocity**: Map velocity to gain (and optionally to filter cutoff or sample selection):

```javascript
function playSampleWithVelocity(buffer, time, velocity) {
  const gain = velocity / 127; // MIDI velocity 0-127
  playSample(buffer, time, gain * gain); // quadratic curve feels more natural
}
```

### Synthesis-Based Approach

Synthesizing drum sounds from scratch provides more flexibility and parameter control.

**Kick drum** (sine wave with pitch envelope):

The TR-808 kick uses a sine oscillator with a rapid pitch sweep from a high frequency down to a low fundamental.

```javascript
function synthKick(time) {
  const osc = ctx.createOscillator();
  const gain = ctx.createGain();

  osc.type = 'sine';
  osc.frequency.setValueAtTime(150, time);        // start frequency
  osc.frequency.exponentialRampToValueAtTime(50, time + 0.05); // pitch drop

  gain.gain.setValueAtTime(1.0, time);
  gain.gain.exponentialRampToValueAtTime(0.001, time + 0.5); // decay

  osc.connect(gain);
  gain.connect(ctx.destination);
  osc.start(time);
  osc.stop(time + 0.5);
}
```

For a punchier kick, layer a short noise click at the attack:

```javascript
function synthKickPunchy(time) {
  // Body (sine with pitch envelope)
  synthKick(time);

  // Click (short burst of noise)
  const noise = createNoiseBuffer(0.02); // 20ms of white noise
  const click = ctx.createBufferSource();
  const clickGain = ctx.createGain();
  const clickFilter = ctx.createBiquadFilter();

  click.buffer = noise;
  clickFilter.type = 'highpass';
  clickFilter.frequency.value = 800;
  clickGain.gain.setValueAtTime(0.5, time);
  clickGain.gain.exponentialRampToValueAtTime(0.001, time + 0.02);

  click.connect(clickFilter);
  clickFilter.connect(clickGain);
  clickGain.connect(ctx.destination);
  click.start(time);
}
```

**Snare drum** (noise + tone):

A snare combines a short tonal component (body) with filtered noise (snare wires).

```javascript
function synthSnare(time) {
  // Tonal body
  const body = ctx.createOscillator();
  const bodyGain = ctx.createGain();
  body.type = 'triangle';
  body.frequency.setValueAtTime(200, time);
  body.frequency.exponentialRampToValueAtTime(100, time + 0.03);
  bodyGain.gain.setValueAtTime(0.7, time);
  bodyGain.gain.exponentialRampToValueAtTime(0.001, time + 0.15);
  body.connect(bodyGain);
  bodyGain.connect(ctx.destination);
  body.start(time);
  body.stop(time + 0.15);

  // Noise (snare wires)
  const noise = ctx.createBufferSource();
  noise.buffer = createNoiseBuffer(0.2);
  const noiseFilter = ctx.createBiquadFilter();
  noiseFilter.type = 'highpass';
  noiseFilter.frequency.value = 1000;
  const noiseGain = ctx.createGain();
  noiseGain.gain.setValueAtTime(1.0, time);
  noiseGain.gain.exponentialRampToValueAtTime(0.001, time + 0.2);
  noise.connect(noiseFilter);
  noiseFilter.connect(noiseGain);
  noiseGain.connect(ctx.destination);
  noise.start(time);
}
```

**Hi-hat** (bandpass-filtered noise, or multiple square waves):

The TR-808 hi-hat uses 6 square wave oscillators at specific non-harmonic frequencies, summed and filtered. A simpler approach uses bandpass-filtered noise:

```javascript
function synthHiHat(time, open = false) {
  const noise = ctx.createBufferSource();
  noise.buffer = createNoiseBuffer(open ? 0.3 : 0.05);
  const filter = ctx.createBiquadFilter();
  filter.type = 'bandpass';
  filter.frequency.value = 10000;
  filter.Q.value = 1;
  const gain = ctx.createGain();
  const duration = open ? 0.3 : 0.05;
  gain.gain.setValueAtTime(0.3, time);
  gain.gain.exponentialRampToValueAtTime(0.001, time + duration);
  noise.connect(filter);
  filter.connect(gain);
  gain.connect(ctx.destination);
  noise.start(time);
}

// For more authentic 808 hi-hat, use 6 square waves:
// Frequencies: 800, 1047, 1480, 1175, 1567, 1109 Hz
// Sum -> bandpass filter -> VCA with envelope
```

**Helper: noise buffer generator**:

```javascript
function createNoiseBuffer(duration) {
  const sampleRate = ctx.sampleRate;
  const length = sampleRate * duration;
  const buffer = ctx.createBuffer(1, length, sampleRate);
  const data = buffer.getChannelData(0);
  for (let i = 0; i < length; i++) {
    data[i] = Math.random() * 2 - 1;
  }
  return buffer;
}
```

### Step Sequencer UI Patterns

A typical drum machine step sequencer has a grid where:

- **Rows** = instruments (kick, snare, hi-hat, etc.)
- **Columns** = time steps (usually 16 per bar)
- **Cells** = on/off toggles (with optional velocity)

```
        1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16
Kick    X  .  .  .  X  .  .  .  X  .  .  .  X  .  .  .
Snare   .  .  .  .  X  .  .  .  .  .  .  .  X  .  .  .
HiHat   X  .  X  .  X  .  X  .  X  .  X  .  X  .  X  .
```

Implementation considerations:

- Highlight the current playing step with a CSS class or animation.
- Use CSS Grid or Flexbox for the layout.
- Handle touch events for mobile.
- Store patterns as 2D arrays or objects: `{ kick: [1,0,0,0,1,0,0,0,...], snare: [...] }`.

### Swing and Humanization

**Swing**: Delays even-numbered 16th notes by a percentage:

```javascript
function getSwingTime(stepIndex, stepDuration, swingAmount) {
  // swingAmount: 0.0 = no swing, 1.0 = full triplet swing
  if (stepIndex % 2 === 1) {
    return stepDuration * swingAmount * 0.33; // delay even steps
  }
  return 0;
}
```

MPC-style swing percentages (50% = no swing, 66% = triplet feel):

```javascript
function mpcSwing(stepIndex, stepDuration, swingPercent) {
  // swingPercent: 50 = straight, 66 = triplet, 75 = heavy
  const ratio = swingPercent / 100;
  if (stepIndex % 2 === 0) {
    return 0;
  }
  return stepDuration * (ratio - 0.5) * 2;
}
```

**Humanization**: Add subtle random timing and velocity variations:

```javascript
function humanize(time, velocity, timingAmount = 0.005, velocityAmount = 15) {
  const timeOffset = (Math.random() - 0.5) * 2 * timingAmount;
  const velOffset = Math.round((Math.random() - 0.5) * 2 * velocityAmount);
  return {
    time: time + timeOffset,
    velocity: Math.max(1, Math.min(127, velocity + velOffset))
  };
}
```

### Notable Drum Machine Projects

**iO-808**
- URL: https://io808.com/
- GitHub: https://github.com/vincentriemer/io-808
- Description: Fully recreated web-based TR-808 drum machine using React, Redux, and the Web Audio API. Faithful recreation of the original interface and sound.

**HTML5 Drum Machine (by dmeldrum6)**
- GitHub: https://github.com/dmeldrum6/WebAudio-Drum-Machine
- Description: Professional web-based drum machine with 5 kits, advanced swing timing, and real-time audio synthesis. Built with vanilla JS and Web Audio API.

**KickDrum.io**
- URL: https://kickdrum.io/
- Description: Free online drum machine with a 16-step sequencer. No download required.

**Dev.Opera Tutorial**
- URL: https://dev.opera.com/articles/drum-sounds-webaudio/
- Description: Detailed tutorial on synthesizing drum sounds (kick, snare, hi-hat) with the Web Audio API, covering the actual TR-808 synthesis techniques.

---

## 5. Building Effects Processors

### Delay

**Simple feedback delay**:

```javascript
function createDelay(ctx, delayTime = 0.3, feedback = 0.5, mix = 0.5) {
  const input = ctx.createGain();
  const output = ctx.createGain();
  const delay = ctx.createDelay(5.0); // max delay 5 seconds
  const feedbackGain = ctx.createGain();
  const dryGain = ctx.createGain();
  const wetGain = ctx.createGain();

  delay.delayTime.value = delayTime;
  feedbackGain.gain.value = feedback;
  dryGain.gain.value = 1 - mix;
  wetGain.gain.value = mix;

  // Dry path
  input.connect(dryGain);
  dryGain.connect(output);

  // Wet path (with feedback loop)
  input.connect(delay);
  delay.connect(feedbackGain);
  feedbackGain.connect(delay); // feedback loop
  delay.connect(wetGain);
  wetGain.connect(output);

  return { input, output, delay, feedbackGain };
}
```

**Ping-pong delay**:

Uses two delay lines panning left and right alternately:

```javascript
function createPingPongDelay(ctx, delayTime = 0.3, feedback = 0.5) {
  const input = ctx.createGain();
  const output = ctx.createGain();
  const delayL = ctx.createDelay(5.0);
  const delayR = ctx.createDelay(5.0);
  const feedbackGain = ctx.createGain();
  const panL = ctx.createStereoPanner();
  const panR = ctx.createStereoPanner();
  const merger = ctx.createChannelMerger(2);

  delayL.delayTime.value = delayTime;
  delayR.delayTime.value = delayTime;
  feedbackGain.gain.value = feedback;
  panL.pan.value = -1;
  panR.pan.value = 1;

  input.connect(delayL);
  delayL.connect(panL);
  panL.connect(output);
  delayL.connect(delayR);    // left feeds right
  delayR.connect(panR);
  panR.connect(output);
  delayR.connect(feedbackGain);
  feedbackGain.connect(delayL); // right feeds back to left

  return { input, output, delayL, delayR, feedbackGain };
}
```

**Tape delay emulation**:

Tape delay characteristics include wow/flutter (pitch modulation), saturation, and high-frequency rolloff in the feedback loop:

```javascript
function createTapeDelay(ctx, delayTime = 0.3, feedback = 0.5) {
  const { input, output, delay, feedbackGain } = createDelay(ctx, delayTime, feedback);

  // Wow/flutter: modulate delay time with slow LFO
  const lfo = ctx.createOscillator();
  const lfoGain = ctx.createGain();
  lfo.frequency.value = 0.5; // slow modulation
  lfoGain.gain.value = 0.002; // subtle pitch wobble
  lfo.connect(lfoGain);
  lfoGain.connect(delay.delayTime);
  lfo.start();

  // High-frequency rolloff in feedback loop
  const filter = ctx.createBiquadFilter();
  filter.type = 'lowpass';
  filter.frequency.value = 4000; // darken repeats

  // Insert filter into feedback path
  feedbackGain.disconnect();
  feedbackGain.connect(filter);
  filter.connect(delay);

  return { input, output };
}
```

### Reverb

**Convolution reverb** (using impulse responses):

The Web Audio API provides `ConvolverNode` which performs convolution with an impulse response (IR) buffer.

```javascript
async function createConvolutionReverb(ctx, irUrl, mix = 0.5) {
  const convolver = ctx.createConvolver();
  const response = await fetch(irUrl);
  const arrayBuffer = await response.arrayBuffer();
  convolver.buffer = await ctx.decodeAudioData(arrayBuffer);

  const input = ctx.createGain();
  const output = ctx.createGain();
  const dryGain = ctx.createGain();
  const wetGain = ctx.createGain();

  dryGain.gain.value = 1 - mix;
  wetGain.gain.value = mix;

  input.connect(dryGain);
  dryGain.connect(output);
  input.connect(convolver);
  convolver.connect(wetGain);
  wetGain.connect(output);

  return { input, output, convolver, wetGain, dryGain };
}
```

Free impulse response libraries:

- Open AIR: https://www.openair.hosted.york.ac.uk/
- EchoThief: http://www.echothief.com/
- Fokke van Saane's IR collection

**Algorithmic reverb** (Schroeder / Freeverb):

For real-time, tunable reverb without loading IR files. Tone.js includes both `Freeverb` and `JCReverb` implementations.

A basic Schroeder reverb consists of parallel comb filters followed by series allpass filters:

```
Input --> [Comb1] -+
      --> [Comb2] -+--> [Allpass1] --> [Allpass2] --> Output
      --> [Comb3] -+
      --> [Comb4] -+
```

```javascript
// Simplified Freeverb structure (actual implementation in AudioWorklet)
// 8 parallel comb filters with different delay times
const combDelays = [1557, 1617, 1491, 1422, 1277, 1356, 1188, 1116]; // samples
// 4 series allpass filters
const allpassDelays = [225, 556, 441, 341]; // samples

// Each comb filter: input -> delay -> output, with feedback and lowpass in loop
// Each allpass filter: input -> delay -> output, with feedforward and feedback
```

Tone.js makes this trivial:

```javascript
const reverb = new Tone.Freeverb({ roomSize: 0.8, dampening: 3000 }).toDestination();
const synth = new Tone.Synth().connect(reverb);
```

### Distortion

**Waveshaping distortion** uses `WaveShaperNode` with a custom transfer function curve:

```javascript
function createDistortion(ctx, amount = 50) {
  const shaper = ctx.createWaveShaper();
  shaper.curve = makeDistortionCurve(amount);
  shaper.oversample = '4x'; // reduces aliasing
  return shaper;
}

// Soft clipping curve
function makeDistortionCurve(amount) {
  const samples = 44100;
  const curve = new Float32Array(samples);
  const deg = Math.PI / 180;
  for (let i = 0; i < samples; i++) {
    const x = (i * 2) / samples - 1;
    curve[i] = ((3 + amount) * x * 20 * deg) /
               (Math.PI + amount * Math.abs(x));
  }
  return curve;
}

// Hard clipping curve
function makeHardClip(threshold) {
  const curve = new Float32Array(44100);
  for (let i = 0; i < 44100; i++) {
    const x = (i * 2) / 44100 - 1;
    curve[i] = Math.max(-threshold, Math.min(threshold, x));
  }
  return curve;
}

// Tube-style saturation (tanh)
function makeTubeSaturation(drive) {
  const curve = new Float32Array(44100);
  for (let i = 0; i < 44100; i++) {
    const x = (i * 2) / 44100 - 1;
    curve[i] = Math.tanh(x * drive);
  }
  return curve;
}
```

### Chorus / Flanger / Phaser

All three effects are based on **modulated delay lines**; they differ in delay time range, feedback, and modulation.

| Effect | Delay Time | Feedback | Modulation |
|--------|-----------|----------|------------|
| **Chorus** | 7-30 ms | Little/none | Slow (0.1-5 Hz) |
| **Flanger** | 0.1-10 ms | Medium-high | Slow (0.1-5 Hz) |
| **Phaser** | Allpass chain | Medium | Slow (0.1-5 Hz) |

**Chorus**:

```javascript
function createChorus(ctx, rate = 1.5, depth = 0.003, delay = 0.02) {
  const input = ctx.createGain();
  const output = ctx.createGain();
  const delayNode = ctx.createDelay(0.1);
  const lfo = ctx.createOscillator();
  const lfoGain = ctx.createGain();
  const dryGain = ctx.createGain();
  const wetGain = ctx.createGain();

  delayNode.delayTime.value = delay;
  lfo.frequency.value = rate;
  lfoGain.gain.value = depth;
  dryGain.gain.value = 0.7;
  wetGain.gain.value = 0.7;

  lfo.connect(lfoGain);
  lfoGain.connect(delayNode.delayTime);

  input.connect(dryGain);
  dryGain.connect(output);
  input.connect(delayNode);
  delayNode.connect(wetGain);
  wetGain.connect(output);

  lfo.start();
  return { input, output };
}
```

**Flanger**: Same structure as chorus but with shorter delay times (0.1-10ms) and feedback:

```javascript
function createFlanger(ctx, rate = 0.5, depth = 0.002, delay = 0.005, feedback = 0.7) {
  const input = ctx.createGain();
  const output = ctx.createGain();
  const delayNode = ctx.createDelay(0.02);
  const feedbackGain = ctx.createGain();
  const lfo = ctx.createOscillator();
  const lfoGain = ctx.createGain();

  delayNode.delayTime.value = delay;
  feedbackGain.gain.value = feedback;
  lfo.frequency.value = rate;
  lfoGain.gain.value = depth;

  lfo.connect(lfoGain);
  lfoGain.connect(delayNode.delayTime);

  input.connect(output); // dry
  input.connect(delayNode);
  delayNode.connect(output); // wet
  delayNode.connect(feedbackGain);
  feedbackGain.connect(delayNode); // feedback

  lfo.start();
  return { input, output };
}
```

**Phaser**: Uses a chain of allpass filters with modulated frequency instead of a delay line:

```javascript
function createPhaser(ctx, rate = 0.5, depth = 1000, stages = 4) {
  const input = ctx.createGain();
  const output = ctx.createGain();
  const lfo = ctx.createOscillator();
  const lfoGain = ctx.createGain();
  const filters = [];

  lfo.frequency.value = rate;
  lfoGain.gain.value = depth;

  for (let i = 0; i < stages; i++) {
    const filter = ctx.createBiquadFilter();
    filter.type = 'allpass';
    filter.frequency.value = 1000 + i * 500;
    filter.Q.value = 0.5;
    lfoGain.connect(filter.frequency); // modulate each stage
    filters.push(filter);
  }

  // Chain filters
  input.connect(output); // dry
  let prev = input;
  for (const filter of filters) {
    prev.connect(filter);
    prev = filter;
  }
  prev.connect(output); // wet (phase-shifted signal mixes with dry)

  lfo.connect(lfoGain);
  lfo.start();
  return { input, output };
}
```

### Compressor

**Built-in `DynamicsCompressorNode`**:

```javascript
const compressor = ctx.createDynamicsCompressor();
compressor.threshold.value = -24;  // dB above which compression starts
compressor.knee.value = 30;        // dB range for smooth transition
compressor.ratio.value = 12;       // compression ratio
compressor.attack.value = 0.003;   // seconds
compressor.release.value = 0.25;   // seconds
// compressor.reduction (read-only): current gain reduction in dB
```

**Custom compressor** (via AudioWorklet) for more control (sidechain, lookahead, RMS/peak detection):

```javascript
// Pseudocode for custom compressor in AudioWorkletProcessor
class CompressorProcessor extends AudioWorkletProcessor {
  process(inputs, outputs) {
    const input = inputs[0][0];
    const sidechain = inputs[1]?.[0] || input; // optional sidechain input
    const output = outputs[0][0];

    for (let i = 0; i < input.length; i++) {
      // Envelope detection (RMS or peak)
      const level = Math.abs(sidechain[i]);
      this.envelope = this.envelope * this.attackCoeff + level * (1 - this.attackCoeff);

      // Gain computation
      const dbLevel = 20 * Math.log10(this.envelope + 1e-6);
      let gainReduction = 0;
      if (dbLevel > this.threshold) {
        gainReduction = (dbLevel - this.threshold) * (1 - 1 / this.ratio);
      }
      const linearGain = Math.pow(10, -gainReduction / 20);

      output[i] = input[i] * linearGain * this.makeupGain;
    }
    return true;
  }
}
```

### EQ (Equalization)

Chain multiple `BiquadFilterNode` instances for a parametric EQ:

```javascript
function createParametricEQ(ctx, bands) {
  // bands: [{ frequency, gain, Q, type }]
  const filters = bands.map(band => {
    const filter = ctx.createBiquadFilter();
    filter.type = band.type || 'peaking';
    filter.frequency.value = band.frequency;
    filter.gain.value = band.gain;
    filter.Q.value = band.Q || 1;
    return filter;
  });

  // Chain filters in series
  for (let i = 0; i < filters.length - 1; i++) {
    filters[i].connect(filters[i + 1]);
  }

  return {
    input: filters[0],
    output: filters[filters.length - 1],
    bands: filters
  };
}

// 3-band EQ example
const eq = createParametricEQ(ctx, [
  { frequency: 80, gain: 3, Q: 0.7, type: 'lowshelf' },
  { frequency: 1000, gain: -2, Q: 1.5, type: 'peaking' },
  { frequency: 8000, gain: 2, Q: 0.7, type: 'highshelf' }
]);
```

### Spatial Audio

**StereoPannerNode** (simple left-right panning):

```javascript
const panner = ctx.createStereoPanner();
panner.pan.value = -0.5; // -1 (left) to 1 (right)
```

**PannerNode** (3D spatial audio):

```javascript
const panner = ctx.createPanner();
panner.panningModel = 'HRTF';       // head-related transfer function
panner.distanceModel = 'inverse';
panner.refDistance = 1;
panner.maxDistance = 10000;
panner.rolloffFactor = 1;
panner.coneInnerAngle = 360;

// Position the source in 3D space
panner.positionX.value = 2;
panner.positionY.value = 0;
panner.positionZ.value = -3;

// Update listener position
ctx.listener.positionX.value = 0;
ctx.listener.positionY.value = 0;
ctx.listener.positionZ.value = 0;
ctx.listener.forwardX.value = 0;
ctx.listener.forwardY.value = 0;
ctx.listener.forwardZ.value = -1;
```

**HRTF panning**: The `'HRTF'` panning model uses Head-Related Transfer Functions to simulate how sound reaches each ear differently based on the sound source's position. This provides realistic 3D spatialization for headphone listening.

**Ambisonics libraries**:

- **Omnitone** (Google): FOA and HOA ambisonic decoding and binaural rendering. Supports 1st order (4 channels), 2nd order (9 channels), and 3rd order (16 channels).
  - GitHub: https://github.com/GoogleChrome/omnitone

- **Resonance Audio** (Google): Higher-level spatial audio SDK built on Omnitone. Designed for scalable real-time 3D audio with many sources. VR-ready.
  - URL: https://resonance-audio.github.io/resonance-audio/develop/web/developer-guide

- **JSAmbisonics**: JavaScript library for FOA and HOA processing using Web Audio.
  - GitHub: https://github.com/polarch/JSAmbisonics

---

## 6. UI Patterns for Audio Apps

### Knobs (Rotary Encoders)

Knobs are the most common control in synthesizer UIs. Several implementation approaches:

**SVG-based knob**:

```html
<svg viewBox="0 0 100 100" width="60" height="60">
  <!-- Background arc -->
  <path d="M 20 80 A 40 40 0 1 1 80 80" fill="none" stroke="#333" stroke-width="4"/>
  <!-- Value arc -->
  <path d="M 20 80 A 40 40 0 0 1 50 10" fill="none" stroke="#0af" stroke-width="4"/>
  <!-- Indicator line -->
  <line x1="50" y1="50" x2="50" y2="15" stroke="#fff" stroke-width="2"
        transform="rotate(135, 50, 50)"/>
</svg>
```

**CSS rotation knob**:

```css
.knob {
  width: 50px;
  height: 50px;
  border-radius: 50%;
  background: conic-gradient(from 225deg, #0af 0deg, #0af var(--angle), #333 var(--angle));
  position: relative;
}
.knob::after {
  content: '';
  position: absolute;
  width: 2px;
  height: 20px;
  background: white;
  top: 5px;
  left: 50%;
  transform-origin: bottom center;
  transform: rotate(var(--rotation));
}
```

**Interaction pattern**: Knobs typically respond to vertical mouse drag (up = increase, down = decrease). This avoids the awkward circular dragging motion.

```javascript
class Knob {
  constructor(element, min = 0, max = 1, value = 0.5) {
    this.element = element;
    this.min = min;
    this.max = max;
    this.value = value;
    this.sensitivity = 0.005;

    element.addEventListener('pointerdown', (e) => {
      this.startY = e.clientY;
      this.startValue = this.value;
      element.setPointerCapture(e.pointerId);
    });

    element.addEventListener('pointermove', (e) => {
      if (e.buttons === 0) return;
      const delta = (this.startY - e.clientY) * this.sensitivity;
      this.value = Math.max(this.min, Math.min(this.max, this.startValue + delta));
      this.updateVisual();
      this.onChange?.(this.value);
    });
  }
}
```

### Sliders / Faders

Vertical faders for channel strips, horizontal sliders for parameters. Use native `<input type="range">` with CSS customization, or build custom with pointer events:

```css
input[type="range"] {
  -webkit-appearance: none;
  appearance: none;
  width: 120px;
  height: 6px;
  background: #333;
  border-radius: 3px;
  outline: none;
}

input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  width: 16px;
  height: 24px;
  background: #ddd;
  border-radius: 3px;
  cursor: pointer;
}

/* Vertical fader */
.vertical-fader input[type="range"] {
  writing-mode: vertical-lr;
  direction: rtl;
  height: 200px;
  width: 6px;
}
```

### Waveform Visualization (AnalyserNode + Canvas)

**Time-domain waveform** (oscilloscope):

```javascript
const analyser = ctx.createAnalyser();
analyser.fftSize = 2048;
const bufferLength = analyser.frequencyBinCount;
const dataArray = new Uint8Array(bufferLength);

function drawWaveform(canvas) {
  const canvasCtx = canvas.getContext('2d');
  const WIDTH = canvas.width;
  const HEIGHT = canvas.height;

  function draw() {
    requestAnimationFrame(draw);
    analyser.getByteTimeDomainData(dataArray);

    canvasCtx.fillStyle = '#000';
    canvasCtx.fillRect(0, 0, WIDTH, HEIGHT);
    canvasCtx.lineWidth = 2;
    canvasCtx.strokeStyle = '#0f0';
    canvasCtx.beginPath();

    const sliceWidth = WIDTH / bufferLength;
    let x = 0;
    for (let i = 0; i < bufferLength; i++) {
      const v = dataArray[i] / 128.0;
      const y = (v * HEIGHT) / 2;
      if (i === 0) canvasCtx.moveTo(x, y);
      else canvasCtx.lineTo(x, y);
      x += sliceWidth;
    }

    canvasCtx.lineTo(WIDTH, HEIGHT / 2);
    canvasCtx.stroke();
  }
  draw();
}
```

### Spectrum Analyzer (FFT Visualization)

**Bar-graph spectrum**:

```javascript
function drawSpectrum(canvas) {
  const canvasCtx = canvas.getContext('2d');
  const WIDTH = canvas.width;
  const HEIGHT = canvas.height;

  analyser.fftSize = 256;
  const bufferLength = analyser.frequencyBinCount;
  const dataArray = new Uint8Array(bufferLength);

  function draw() {
    requestAnimationFrame(draw);
    analyser.getByteFrequencyData(dataArray);

    canvasCtx.fillStyle = '#000';
    canvasCtx.fillRect(0, 0, WIDTH, HEIGHT);

    const barWidth = (WIDTH / bufferLength) * 2.5;
    let x = 0;
    for (let i = 0; i < bufferLength; i++) {
      const barHeight = (dataArray[i] / 255) * HEIGHT;
      const hue = (i / bufferLength) * 240;
      canvasCtx.fillStyle = `hsl(${hue}, 100%, 50%)`;
      canvasCtx.fillRect(x, HEIGHT - barHeight, barWidth, barHeight);
      x += barWidth + 1;
    }
  }
  draw();
}
```

**Spectrogram** (frequency vs. time waterfall):

```javascript
function drawSpectrogram(canvas) {
  const canvasCtx = canvas.getContext('2d');
  const WIDTH = canvas.width;
  const HEIGHT = canvas.height;
  let xPos = 0;

  analyser.fftSize = 1024;
  const bufferLength = analyser.frequencyBinCount;
  const dataArray = new Uint8Array(bufferLength);

  function draw() {
    requestAnimationFrame(draw);
    analyser.getByteFrequencyData(dataArray);

    // Draw one vertical column per frame
    for (let i = 0; i < bufferLength; i++) {
      const value = dataArray[i];
      const y = HEIGHT - (i / bufferLength) * HEIGHT;
      const hue = value * 1.2;
      canvasCtx.fillStyle = `hsl(${hue}, 100%, ${value / 5}%)`;
      canvasCtx.fillRect(xPos, y, 1, HEIGHT / bufferLength);
    }

    xPos = (xPos + 1) % WIDTH;
    // Clear next column
    canvasCtx.fillStyle = '#000';
    canvasCtx.fillRect(xPos, 0, 2, HEIGHT);
  }
  draw();
}
```

### Piano Roll / Grid Editors

Piano rolls display notes on a time-frequency grid, typically implemented with HTML5 Canvas for performance.

Key components:

- **Grid**: Horizontal lines for pitches, vertical lines for time divisions.
- **Notes**: Colored rectangles positioned by pitch (y) and start time (x), width = duration.
- **Interactions**: Click to add, drag to move/resize, right-click to delete.
- **Scroll/Zoom**: Horizontal zoom for time, vertical scroll for pitch range.
- **Playhead**: Animated vertical line showing current position.

Libraries that provide piano roll components:

- **MIDI.js**: Rendering and playback of MIDI files.
- **Flat.io** / **Noteflight**: Commercial notation editors with piano roll views.
- **Pianoroll (npm)**: Lightweight piano roll widget.

### Virtual Cables / Patch Cord UI

Virtual cables connect module outputs to inputs, mimicking physical patch cables in modular synths.

**Implementation approaches**:

1. **SVG path with cubic Bezier**: Most common. The cable sags naturally using control points offset downward.

```javascript
function drawCable(x1, y1, x2, y2) {
  const sag = Math.abs(x2 - x1) * 0.5 + 50; // sag based on distance
  const midY = Math.max(y1, y2) + sag;
  return `M ${x1} ${y1} C ${x1} ${midY}, ${x2} ${midY}, ${x2} ${y2}`;
}
```

2. **Canvas-based**: Better performance with many cables. Redraw all cables each frame.

3. **WebGL**: For very high cable counts with glow/physics effects.

**Interaction flow**:

1. User clicks on an output jack -> start drawing a temporary cable.
2. Cable follows the mouse cursor.
3. User releases on a valid input jack -> connection is established.
4. Click on a cable or jack to disconnect.

### UI Component Libraries

**NexusUI**
- URL: https://nexus-js.github.io/ui/
- GitHub: https://github.com/nexus-js/ui
- Components: Dial, Slider, Button, Toggle, RadioButton, Number, Select, Sequencer, Piano, Multislider, Pan2D, Position, Envelope, Spectrogram, Meter, Oscilloscope, Tilt (accelerometer)
- Built with SVG. Designed specifically for web audio interfaces.

**webaudio-controls**
- URL: http://g200kg.github.io/webaudio-controls/docs/
- GitHub: https://github.com/g200kg/webaudio-controls
- Components: Knobs, sliders, switches, parameter displays, keyboards
- Built as **Web Components** (custom elements). Supports filmstrip/sprite images for knob skins. MIDI controller support built-in.

**Custom Web Components** (recommended for new projects):

```javascript
class AudioKnob extends HTMLElement {
  static get observedAttributes() {
    return ['value', 'min', 'max', 'label'];
  }

  constructor() {
    super();
    this.attachShadow({ mode: 'open' });
    // ... render knob with SVG in shadow DOM
  }

  attributeChangedCallback(name, oldVal, newVal) {
    // Update visual on attribute change
  }

  get value() { return parseFloat(this.getAttribute('value')); }
  set value(v) { this.setAttribute('value', v); }
}

customElements.define('audio-knob', AudioKnob);
```

Usage:

```html
<audio-knob value="0.5" min="0" max="1" label="Cutoff"></audio-knob>
```

---

## 7. Performance Optimization

### AudioWorklet vs. ScriptProcessorNode

| Feature | AudioWorklet | ScriptProcessorNode (deprecated) |
|---------|-------------|----------------------------------|
| **Thread** | Dedicated audio rendering thread | Main thread |
| **Latency** | Low (consistent) | High (subject to main thread jank) |
| **Buffer size** | Fixed 128 samples | Configurable (256-16384) |
| **GC risk** | Minimal (no DOM access) | High (shares heap with main thread) |
| **WASM** | Full support | Limited |
| **Status** | Standard | Deprecated (will be removed) |

**Always use AudioWorklet** for custom processing. ScriptProcessorNode should only appear in legacy compatibility code.

### SharedArrayBuffer for Thread Communication

AudioWorklet runs on a separate thread. Communication options:

1. **MessagePort** (`postMessage`): Asynchronous, involves serialization. Acceptable for infrequent, non-real-time messages (loading samples, changing modes).

2. **SharedArrayBuffer (SAB)**: Zero-copy, lock-free communication between threads. Essential for real-time data transfer.

```javascript
// Main thread
const sab = new SharedArrayBuffer(4096);
const sharedArray = new Float32Array(sab);

const node = new AudioWorkletNode(ctx, 'my-processor');
node.port.postMessage({ type: 'init', buffer: sab });

// Processor thread
class MyProcessor extends AudioWorkletProcessor {
  constructor() {
    super();
    this.port.onmessage = (e) => {
      if (e.data.type === 'init') {
        this.sharedBuffer = new Float32Array(e.data.buffer);
      }
    };
  }

  process(inputs, outputs) {
    // Read/write shared memory - no allocation, no GC
    const input = inputs[0][0];
    for (let i = 0; i < input.length; i++) {
      this.sharedBuffer[i] = input[i]; // share data with main thread
    }
    return true;
  }
}
```

**Important**: SharedArrayBuffer requires cross-origin isolation headers:

```
Cross-Origin-Opener-Policy: same-origin
Cross-Origin-Embedder-Policy: require-corp
```

For lock-free communication, use `Atomics.store()` / `Atomics.load()` for indices, and ring buffers for streaming data.

### WASM SIMD for DSP

WebAssembly SIMD (Single Instruction, Multiple Data) enables processing multiple audio samples simultaneously using 128-bit vector operations.

**Performance impact**: SIMD can provide 2-4x speedup for vectorizable DSP operations (FIR filters, FFT, mixing, gain, etc.).

**Example**: Processing 4 float samples in parallel:

```rust
// Rust with wasm SIMD (via core::arch::wasm32)
use core::arch::wasm32::*;

pub fn apply_gain_simd(buffer: &mut [f32], gain: f32) {
    let gain_vec = f32x4_splat(gain);
    let chunks = buffer.len() / 4;

    for i in 0..chunks {
        let offset = i * 4;
        let samples = v128_load(buffer[offset..].as_ptr() as *const v128);
        let result = f32x4_mul(samples, gain_vec);
        v128_store(buffer[offset..].as_mut_ptr() as *mut v128, result);
    }
}
```

**Browser support**: WASM SIMD is supported in Chrome, Firefox, Safari, and Edge (since 2021-2022). It is considered stable for production use.

**Toolchains**:

- **Rust**: Use `#[target_feature(enable = "simd128")]` and `core::arch::wasm32` intrinsics, or use the `wide` / `packed_simd` crates.
- **C/C++ via Emscripten**: Use `-msimd128` flag. SSE/SSE2 intrinsics are automatically mapped to WASM SIMD.
- **AssemblyScript**: Built-in SIMD support via `v128` type.

### Avoiding GC Pauses in the Audio Thread

Garbage collection pauses on the audio thread cause audible glitches (clicks, pops, dropouts). Strategies:

1. **Pre-allocate all buffers** during initialization, not during `process()`.
2. **Avoid object creation** in `process()`: no `new`, no array literals, no closures.
3. **Use typed arrays** (`Float32Array`, `Int32Array`) instead of regular arrays.
4. **Use WASM**: WebAssembly has its own linear memory without GC.
5. **Avoid string operations** in the audio thread.
6. **Pool and reuse objects** if dynamic allocation is unavoidable.

```javascript
// BAD: allocates on every process call
process(inputs, outputs) {
  const filtered = inputs[0][0].filter(x => x > 0); // creates new array
  const result = { left: outputs[0][0], right: outputs[0][1] }; // creates object
}

// GOOD: pre-allocated, zero-allocation processing
constructor() {
  super();
  this.tempBuffer = new Float32Array(128);
}

process(inputs, outputs) {
  const input = inputs[0][0];
  const output = outputs[0][0];
  for (let i = 0; i < 128; i++) {
    output[i] = input[i] * this.gain; // no allocation
  }
  return true;
}
```

### Buffer Size Tradeoffs

The AudioWorklet render quantum is fixed at **128 samples** (~2.9ms at 44.1kHz). This is not configurable in the current spec (though a "Configurable Render Quantum" feature is planned for the next revision).

**Latency chain**: Input buffer + processing + output buffer

| Sample Rate | 128 samples | 256 samples | 512 samples |
|------------|------------|------------|------------|
| 44100 Hz | 2.9 ms | 5.8 ms | 11.6 ms |
| 48000 Hz | 2.7 ms | 5.3 ms | 10.7 ms |
| 96000 Hz | 1.3 ms | 2.7 ms | 5.3 ms |

For responsive instruments (keyboards, drum pads), aim for total round-trip latency under **10ms** (ideally under 5ms). This requires:

- Small buffer sizes (128 or 256 samples)
- Efficient processing (WASM preferred)
- Low-latency audio backend (OS-dependent)

For playback-oriented applications (DAWs, sequencers), higher latency (256-512 samples) provides more stability against dropouts.

**Platform considerations**:

- **macOS / iOS**: Excellent audio latency (CoreAudio). ~5ms achievable.
- **Windows**: Variable. WASAPI exclusive mode is good; shared mode adds latency.
- **Linux**: PulseAudio adds latency; JACK or PipeWire provide low latency.
- **Android**: Historically poor; improving with AAudio/Oboe. ~20-40ms typical.

---

## 8. Notable Web Audio Projects

### Music Education and Interactive Experiences

**Ableton Learning Music**
- URL: https://learningmusic.ableton.com/
- Description: Interactive tutorial that teaches the basics of music making (beats, melodies, basslines, chords, song structure) in the browser. You can export creations as Ableton Live Sets.
- Tech: Custom Web Audio implementation.

**Chrome Music Lab**
- URL: https://musiclab.chromeexperiments.com/
- GitHub: https://github.com/googlecreativelab/chrome-music-lab
- Description: Collection of experiments for exploring how music works, all built with Web Audio API. Includes Song Maker, Shared Piano, Rhythm, Spectrogram, Sound Waves, Arpeggios, Kandinsky, Melody Maker, Voice Spinner, Harmonics, Piano Roll, and Oscillators.
- Tech: Tone.js, Web Audio API, WebMIDI.

### Synthesizers

**Viktor NV-1**
- URL: https://nicroto.github.io/viktor/
- Description: Full-featured browser synth inspired by the Minimoog. Three oscillators, envelopes, filters, LFOs, effects. Supports MIDI keyboard input. Patch save/load/export. Over 130 preset patches.
- Tech: Web Audio API, WebMIDI.

**WebSynths**
- URL: https://www.websynths.com/
- Description: Browser-based synthesizer with 130+ preset patches. Save/load user patches. Computer keyboard, mouse, and MIDI input.

**Synth Kitchen**
- URL: https://synthkitchen.com/
- Description: Modular synthesis environment in the browser. Connect modules with virtual cables to build custom synth patches.

### Live Coding Environments

**Strudel**
- URL: https://strudel.cc/
- GitHub: https://codeberg.org/uzu/strudel (migrated from GitHub)
- Description: Web-based live coding environment for algorithmic patterns. A JavaScript port of TidalCycles (Haskell). Write code to generate evolving musical patterns in real time. No installation needed. Used in algorave performances.
- License: AGPL-3.0

**Gibber**
- URL: https://gibber.cc/
- GitHub: https://github.com/gibber-cc/gibber
- Description: Audiovisual live coding environment for the browser. Combines music synthesis, sequencing, and ray-marching 3D graphics. Supports collaborative performance (multiple coders in real-time). Used in education at 30+ universities.
- Creator: Charlie Roberts (UC Santa Barbara)

**Dittytoy**
- URL: https://dittytoy.net/
- GitHub: https://github.com/reindernijhoff/dittytoy-package
- Description: Platform for creating generative music using a minimalistic JavaScript API (inspired by Sonic Pi). Build synths, filters, and polyrhythms with simple code loops. Every ditty is open source. Runs entirely in the browser.
- Creator: Reinder Nijhoff

### DAWs and Production Tools

**Amped Studio**
- URL: https://ampedstudio.com/
- Description: Commercial web-based DAW. Hosts WAM 2.0 plugins. Multi-track recording, MIDI editing, virtual instruments and effects.

**BandLab**
- URL: https://www.bandlab.com/
- Description: Free web-based DAW and social music platform. Multi-track recording, effects, virtual instruments, collaboration.

**Soundtrap** (by Spotify)
- URL: https://www.soundtrap.com/
- Description: Cloud-based music and podcast studio. Collaboration features, loops, instruments, effects.

**Web Synth (by Ameobea)**
- URL: https://synth.ameo.dev/
- GitHub: https://github.com/Ameobea/web-synth
- Description: Browser-based DAW and audio synthesis platform with dozens of effects, synths, and modules. Modular patching, dynamic custom code, live looping. Built with Rust + WASM.

### Modular and Experimental

**Zupiter**
- URL: https://z.musictools.live/
- Description: Web-based modular synthesizer. Create nodes, connect them, build custom synths. Computer keyboard and MIDI input.

**NoiseCraft**
- GitHub: https://github.com/maximecb/noisecraft
- Description: Free, open-source visual programming language for sound synthesis. Node-based interface for building audio graphs.

**Patchcab**
- URL: https://patch.cab
- GitHub: https://github.com/spectrome/patchcab
- Description: Modular Eurorack-style synthesizer. Built with Tone.js + Svelte. Community modules.

### Audio Visualization

**wavesurfer.js**
- URL: https://wavesurfer.xyz/
- Description: Navigable waveform visualization library built on Web Audio API and Canvas. Widely used for audio players, editors, and annotation tools.

### Other Notable Projects

**SuperSonic**
- URL: https://sonic-pi.net/supersonic/demo.html
- GitHub: https://github.com/samaaron/supersonic
- Description: SuperCollider's scsynth engine running in the browser as a Web AudioWorklet via WebAssembly. Developed by Sam Aaron (creator of Sonic Pi).

**Csound Web IDE**
- URL: https://ide.csound.com/
- Description: Run Csound code directly in the browser. Csound 7 (released September 2025) available. Thousands of opcodes for synthesis and processing.

**Faust Web IDE**
- URL: https://faustide.grame.fr/
- Description: Write, compile, and run Faust DSP programs in the browser. Generates WebAssembly. Export to various formats.

---

## Appendix: Quick Reference Links

### Specifications and Documentation

| Resource | URL |
|----------|-----|
| Web Audio API Spec | https://www.w3.org/TR/webaudio/ |
| Web Audio API (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API |
| Web MIDI API Spec | https://www.w3.org/TR/webmidi/ |
| AudioWorklet (MDN) | https://developer.mozilla.org/en-US/docs/Web/API/AudioWorklet |
| Web Audio API Book (Boris Smus) | https://webaudioapi.com/book/ |

### Frameworks and Libraries

| Library | URL | GitHub |
|---------|-----|--------|
| Tone.js | https://tonejs.github.io/ | https://github.com/Tonejs/Tone.js |
| Elementary Audio | https://www.elementary.audio/ | https://github.com/elemaudio/elementary |
| Faust | https://faust.grame.fr/ | https://github.com/grame-cncm/faust |
| RNBO | https://rnbo.cycling74.com/ | (commercial) |
| WAM 2.0 | https://www.webaudiomodules.com/ | https://github.com/webaudiomodules |
| Csound (web) | https://csound.com/wasm/ | https://github.com/csound/csound |
| SuperSonic | https://sonic-pi.net/supersonic/ | https://github.com/samaaron/supersonic |
| WEBMIDI.js | https://webmidijs.org/ | https://github.com/djipco/webmidi |
| NexusUI | https://nexus-js.github.io/ui/ | https://github.com/nexus-js/ui |
| webaudio-controls | http://g200kg.github.io/webaudio-controls/ | https://github.com/g200kg/webaudio-controls |
| Omnitone | (Google spatial audio) | https://github.com/GoogleChrome/omnitone |
| Resonance Audio | https://resonance-audio.github.io/ | https://github.com/resonance-audio/ |
| wavesurfer.js | https://wavesurfer.xyz/ | https://github.com/katspaugh/wavesurfer.js |

### Curated Lists

| Resource | URL |
|----------|-----|
| awesome-webaudio | https://github.com/notthetup/awesome-webaudio |
| Chrome Web Audio Samples | https://googlechromelabs.github.io/web-audio-samples/ |
| Audio Worklet Design Patterns | https://developer.chrome.com/blog/audio-worklet-design-pattern/ |
| A Tale of Two Clocks | https://web.dev/articles/audio-scheduling |

### Key Articles and Tutorials

| Topic | URL |
|-------|-----|
| Scheduling and Timing | https://web.dev/articles/audio-scheduling |
| AudioWorklet Patterns | https://developer.chrome.com/blog/audio-worklet-design-pattern/ |
| Synthesizing Drum Sounds | https://dev.opera.com/articles/drum-sounds-webaudio/ |
| Synthesizing Hi-Hats | http://joesul.li/van/synthesizing-hi-hats/ |
| FM Synth with Rust/WASM/SIMD | https://cprimozic.net/blog/fm-synth-rust-wasm-simd/ |
| Wavetable Synth with Rust/WASM | https://cprimozic.net/blog/buliding-a-wavetable-synthesizer-with-rust-wasm-and-webaudio/ |
| Web Audio Effects with Rust/WASM | https://whoisryosuke.com/blog/2025/web-audio-effect-library-with-rust-and-wasm |
| Convolution Reverb | https://itnext.io/convolution-reverb-and-web-audio-api-8ee65108f4ae |
| Web Audio Spatialization | https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Web_audio_spatialization_basics |
| Building a Modular Synth | https://medium.com/geekculture/building-a-modular-synth-with-web-audio-api-and-javascript-d38ccdeca9ea |

---

*Last updated: 2026-02-26*
