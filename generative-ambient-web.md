# Generative Ambient Music for the Web

A comprehensive reference for coding generative ambient music in the browser -- covering the Web Audio API, Tone.js, generative algorithms, libraries, practical code patterns, and performance considerations.

---

## Table of Contents

1. [Web Audio API Fundamentals](#1-web-audio-api-fundamentals)
2. [Tone.js](#2-tonejs)
3. [Generative Music Techniques](#3-generative-music-techniques)
4. [Key Libraries](#4-key-libraries)
5. [Practical Code Patterns](#5-practical-code-patterns)
6. [Notable References](#6-notable-references)
7. [Performance](#7-performance)
8. [Comparison Table](#8-comparison-table)
9. [Sources](#9-sources)

---

## 1. Web Audio API Fundamentals

### 1.1 AudioContext

The `AudioContext` is the central object of the Web Audio API. It represents an audio-processing graph built from audio modules (nodes) linked together.

```javascript
// Default -- uses device's preferred sample rate (commonly 44100 or 48000)
const ctx = new AudioContext();

// With options
const ctx = new AudioContext({
  latencyHint: 'playback',  // lowest power draw, best for non-interactive generative music
  sampleRate: 44100,
});
```

**`latencyHint` values:**

| Value | Behaviour |
|---|---|
| `"interactive"` (default) | Lowest latency, higher power draw |
| `"balanced"` | Tradeoff between latency and power |
| `"playback"` | Highest latency, lowest power -- best for generative ambient |
| `<number>` | Preferred maximum latency in seconds |

**Key properties:**

| Property | Type | Description |
|---|---|---|
| `currentTime` | `double` (read-only) | Ever-increasing hardware time in seconds. The scheduling clock. |
| `sampleRate` | `float` (read-only) | Sample rate in Hz. Cannot change after creation. |
| `state` | `string` (read-only) | `"suspended"`, `"running"`, or `"closed"` |
| `destination` | `AudioDestinationNode` | Final output node (the speakers) |
| `audioWorklet` | `AudioWorklet` | Entry point for loading AudioWorklet processors |

**State management:**

```javascript
const ctx = new AudioContext();
// State will be 'suspended' if created without user gesture

ctx.addEventListener('statechange', () => {
  console.log('AudioContext state:', ctx.state);
});

await ctx.resume();   // Resume (returns a Promise)
await ctx.suspend();  // Pause processing, reduce CPU
await ctx.close();    // Release all resources -- permanent
```

**Robust initialization pattern:**

```javascript
let ctx = null;

function ensureAudioContext() {
  if (!ctx) {
    ctx = new AudioContext({ latencyHint: 'playback', sampleRate: 44100 });
  }
  if (ctx.state === 'suspended') {
    return ctx.resume();
  }
  return Promise.resolve();
}

document.getElementById('start').addEventListener('click', async () => {
  await ensureAudioContext();
  // Now safe to create nodes and schedule audio
});
```

### 1.2 Oscillators

`OscillatorNode` generates a constant periodic waveform. It has zero inputs and one output. Once stopped, it cannot be restarted -- you must create a new instance.

**Built-in waveform types:**

| Type | Character |
|---|---|
| `"sine"` | Pure tone, no harmonics. Foundation of ambient drones. |
| `"triangle"` | Odd harmonics only, falls off at 1/n^2. Softer than square. |
| `"sawtooth"` | All harmonics, falls off at 1/n. Bright, buzzy. |
| `"square"` | Odd harmonics only, falls off at 1/n. Hollow, clarinet-like. |
| `"custom"` | Set automatically when using `setPeriodicWave()`. |

```javascript
const osc = new OscillatorNode(ctx, {
  type: 'sine',
  frequency: 220,    // Hz (AudioParam, a-rate)
  detune: 0,         // cents (AudioParam, a-rate)
});

osc.connect(ctx.destination);
osc.start();
osc.stop(ctx.currentTime + 5);
// After stop(), the node is dead. Create a new one to play again.
```

**AudioParam automation methods:**

| Method | Description |
|---|---|
| `setValueAtTime(value, time)` | Instant change at precise time |
| `linearRampToValueAtTime(value, endTime)` | Linear interpolation to target |
| `exponentialRampToValueAtTime(value, endTime)` | Exponential interpolation (target must not be 0) |
| `setTargetAtTime(target, startTime, timeConstant)` | Asymptotic approach (never fully reaches target) |
| `setValueCurveAtTime(values, startTime, duration)` | Follow a Float32Array of values |
| `cancelScheduledValues(startTime)` | Remove all events from startTime onward |
| `cancelAndHoldAtTime(cancelTime)` | Cancel future events, hold current value |

```javascript
osc.frequency.setValueAtTime(440, ctx.currentTime);
osc.frequency.linearRampToValueAtTime(880, ctx.currentTime + 3);
osc.frequency.exponentialRampToValueAtTime(110, ctx.currentTime + 5);
osc.frequency.setTargetAtTime(330, ctx.currentTime, 2); // time constant 2s
osc.detune.linearRampToValueAtTime(100, ctx.currentTime + 2); // up one semitone
```

**Custom waveforms with PeriodicWave:**

```javascript
const real = new Float32Array([0, 0, 0, 0, 0]);
const imag = new Float32Array([0, 1, 0.5, 0.25, 0.125]); // fundamental + falling harmonics

const wave = ctx.createPeriodicWave(real, imag, { disableNormalization: false });
const osc = new OscillatorNode(ctx, { frequency: 110 });
osc.setPeriodicWave(wave);  // type becomes "custom"
```

### 1.3 Audio Scheduling -- The Two-Clock Technique

`setTimeout`/`setInterval` run on the main thread with tens of milliseconds of jitter. `audioContext.currentTime` runs on a separate high-priority audio thread with hardware precision, but has no callback mechanism.

The solution: **schedule events ahead of time** using the audio clock, but use JavaScript timers to know when to schedule.

```javascript
const SCHEDULE_AHEAD_TIME = 0.1;  // seconds -- how far ahead to schedule
const TIMER_INTERVAL = 25;        // ms -- how often setTimeout fires

let nextNoteTime = 0.0;

function scheduleNote(time) {
  const osc = ctx.createOscillator();
  const gain = ctx.createGain();
  osc.connect(gain).connect(ctx.destination);

  osc.frequency.value = 220;
  gain.gain.setValueAtTime(0.3, time);
  gain.gain.exponentialRampToValueAtTime(0.001, time + 0.1);

  osc.start(time);
  osc.stop(time + 0.1);
}

function scheduler() {
  while (nextNoteTime < ctx.currentTime + SCHEDULE_AHEAD_TIME) {
    scheduleNote(nextNoteTime);
    nextNoteTime += 0.5; // half-second interval
  }
  setTimeout(scheduler, TIMER_INTERVAL);
}
```

**For generative ambient**, the same pattern schedules evolving events:

```javascript
let nextEventTime = ctx.currentTime;

function ambientScheduler() {
  while (nextEventTime < ctx.currentTime + SCHEDULE_AHEAD_TIME) {
    scheduleAmbientEvent(nextEventTime);
    nextEventTime += 2 + Math.random() * 6; // random 2-8 second intervals
  }
  setTimeout(ambientScheduler, TIMER_INTERVAL);
}
```

### 1.4 AudioWorklet

`AudioWorklet` replaces the deprecated `ScriptProcessorNode`. Code runs on the dedicated audio rendering thread -- zero additional latency, no main-thread interference.

```
Main Thread                          Audio Rendering Thread
+-----------------------+            +---------------------------+
|  AudioWorkletNode     |<--port-->  |  AudioWorkletProcessor    |
|  (extends AudioNode)  | MessagePort|  process() called every   |
|  .parameters          |            |  128 frames (~2.9ms @44k) |
+-----------------------+            +---------------------------+
```

**Processor file (`noise-processor.js`):**

```javascript
class NoiseProcessor extends AudioWorkletProcessor {
  static get parameterDescriptors() {
    return [{
      name: 'amplitude',
      defaultValue: 0.5,
      minValue: 0,
      maxValue: 1,
      automationRate: 'a-rate',
    }];
  }

  process(inputs, outputs, parameters) {
    const output = outputs[0];
    const amplitude = parameters.amplitude;

    for (let channel = 0; channel < output.length; channel++) {
      const outputChannel = output[channel];
      for (let i = 0; i < outputChannel.length; i++) {
        const amp = amplitude.length > 1 ? amplitude[i] : amplitude[0];
        outputChannel[i] = (Math.random() * 2 - 1) * amp;
      }
    }
    return true; // keep alive
  }
}

registerProcessor('noise-processor', NoiseProcessor);
```

**Main thread usage:**

```javascript
await ctx.audioWorklet.addModule('noise-processor.js');

const noiseNode = new AudioWorkletNode(ctx, 'noise-processor', {
  numberOfInputs: 0,
  numberOfOutputs: 1,
  outputChannelCount: [2],
});

noiseNode.parameters.get('amplitude').linearRampToValueAtTime(0.0, ctx.currentTime + 10);
noiseNode.connect(ctx.destination);
```

### 1.5 Key Nodes for Ambient

**GainNode -- volume control and envelopes:**

```javascript
const gain = new GainNode(ctx, { gain: 0 });
gain.gain.setValueAtTime(0, ctx.currentTime);
gain.gain.linearRampToValueAtTime(0.5, ctx.currentTime + 3);
// Use setTargetAtTime for organic-sounding fades:
gain.gain.setTargetAtTime(0.001, ctx.currentTime + 10, 2);
```

**BiquadFilterNode -- tone shaping:**

```javascript
const filter = new BiquadFilterNode(ctx, {
  type: 'lowpass',
  frequency: 1000,
  Q: 1.0,
});
// Slowly sweep for evolving texture:
filter.frequency.exponentialRampToValueAtTime(4000, ctx.currentTime + 20);
```

Filter types: `lowpass`, `highpass`, `bandpass`, `lowshelf`, `highshelf`, `peaking`, `notch`, `allpass`.

**ConvolverNode -- reverb via impulse responses:**

The single most important effect for ambient music.

```javascript
async function createReverb(ctx, url) {
  const convolver = new ConvolverNode(ctx, { normalize: true });
  const response = await fetch(url);
  convolver.buffer = await ctx.decodeAudioData(await response.arrayBuffer());
  return convolver;
}

// Wet/dry mix:
source.connect(dryGain).connect(ctx.destination);
source.connect(reverb).connect(wetGain).connect(ctx.destination);
```

**Generating a synthetic impulse response (no file needed):**

```javascript
function createSyntheticIR(ctx, duration = 3, decay = 2) {
  const length = ctx.sampleRate * duration;
  const impulse = ctx.createBuffer(2, length, ctx.sampleRate);

  for (let ch = 0; ch < 2; ch++) {
    const data = impulse.getChannelData(ch);
    for (let i = 0; i < length; i++) {
      data[i] = (Math.random() * 2 - 1) * Math.exp(-i / (ctx.sampleRate * decay));
    }
  }
  return impulse;
}
```

**DelayNode -- echo and feedback:**

```javascript
const delay = new DelayNode(ctx, { maxDelayTime: 5.0 });
delay.delayTime.value = 0.4;
const feedback = new GainNode(ctx, { gain: 0.6 });

source.connect(delay);
delay.connect(feedback);
feedback.connect(delay);  // feedback loop
delay.connect(ctx.destination);
```

**StereoPannerNode -- spatial movement:**

```javascript
const panner = new StereoPannerNode(ctx, { pan: 0 });
const lfo = new OscillatorNode(ctx, { type: 'sine', frequency: 0.05 });
const lfoGain = new GainNode(ctx, { gain: 0.8 });
lfo.connect(lfoGain).connect(panner.pan);
lfo.start();
```

**DynamicsCompressorNode, WaveShaperNode, AnalyserNode** are also essential for taming peaks, adding warmth, and visualization respectively.

### 1.6 Autoplay Policies

All major browsers block audio not initiated by a user gesture.

**What counts as a gesture:** `click`, `pointerdown`, `touchstart`, `touchend`, `keydown`, `keyup`. **Not** `mouseenter`, `mousemove`, `scroll`, or programmatic focus.

```javascript
// Pattern: create early, resume on gesture
const ctx = new AudioContext();
// Set up entire graph while suspended...

document.getElementById('start').addEventListener('click', async () => {
  if (ctx.state === 'suspended') await ctx.resume();
  startMusic();
});
```

**Resilient wrapper:**

```javascript
['click', 'touchstart', 'keydown'].forEach(event => {
  document.addEventListener(event, () => {
    if (ctx.state === 'suspended') ctx.resume();
  }, { once: true });
});
```

---

## 2. Tone.js

Tone.js is a Web Audio framework for creating interactive music in the browser. It wraps the Web Audio API with high-level abstractions: a global transport, prebuilt synthesizers and effects, and composable building blocks.

**Installation:**

```bash
npm install tone
```

```javascript
import * as Tone from "tone";
```

### 2.1 Core Concepts

**Starting audio (autoplay):**

```javascript
document.querySelector("#start").addEventListener("click", async () => {
  await Tone.start();
  console.log("Audio context is running");
});
```

**Transport -- the master timeline:**

```javascript
const transport = Tone.getTransport();
transport.bpm.value = 55;        // ambient tempo
transport.start();
transport.bpm.rampTo(48, 300);   // slow down over 5 minutes
```

**Time notation:**

| Format | Example | Meaning |
|---|---|---|
| Note value | `"4n"`, `"8n"`, `"2n."`, `"8t"` | Quarter, eighth, dotted half, triplet eighth |
| Measure | `"1m"`, `"4m"` | 1 measure, 4 measures |
| Bars:Beats:16ths | `"4:3:2"` | Bar 4, beat 3, sixteenth 2 |
| Seconds | `1.5` | 1.5 seconds |
| Frequency | `"0.25hz"` | 4-second period |
| Relative | `"+1m"` | 1 measure from now |
| Quantized | `"@1m"` | Next measure boundary |

### 2.2 Synths

**Tone.Synth -- basic monophonic:**

```javascript
const synth = new Tone.Synth({
  oscillator: { type: "sine" },
  envelope: { attack: 2, decay: 1, sustain: 0.8, release: 4 }
}).toDestination();

synth.triggerAttackRelease("C4", "2n");
```

**Tone.FMSynth -- complex evolving timbres:**

```javascript
const fm = new Tone.FMSynth({
  harmonicity: 3,
  modulationIndex: 10,
  envelope: { attack: 3, decay: 1, sustain: 0.6, release: 5 },
  modulationEnvelope: { attack: 4, sustain: 1, release: 5 }
}).toDestination();

fm.triggerAttackRelease("A2", 8);
```

**Tone.PolySynth -- chords and pads:**

```javascript
const pad = new Tone.PolySynth(Tone.Synth, {
  maxPolyphony: 8,
  options: {
    oscillator: { type: "sine" },
    envelope: { attack: 3, decay: 1, sustain: 0.8, release: 6 }
  }
}).toDestination();

pad.triggerAttack(["C3", "E3", "G3", "B3"]);
pad.triggerRelease(["C3", "E3", "G3", "B3"], "+8");
```

**Tone.NoiseSynth -- wind, ocean, breath:**

```javascript
const noise = new Tone.NoiseSynth({
  noise: { type: "pink" },  // pink, white, or brown
  envelope: { attack: 4, decay: 2, sustain: 0.5, release: 6 }
}).toDestination();
noise.volume.value = -15;
noise.triggerAttackRelease(10);
```

**Tone.GrainPlayer -- granular synthesis:**

```javascript
const grains = new Tone.GrainPlayer({
  url: "field-recording.mp3",
  loop: true,
  grainSize: 0.2,
  overlap: 0.1,
  playbackRate: 0.5,
  detune: -200,
}).toDestination();

Tone.loaded().then(() => grains.start());
```

**Tone.Sampler -- multi-sample instrument:**

```javascript
const sampler = new Tone.Sampler({
  urls: { C4: "C4.mp3", A4: "A4.mp3" },
  release: 3,
  baseUrl: "https://tonejs.github.io/audio/salamander/",
}).toDestination();
```

### 2.3 Effects

**Connection methods:**

```javascript
// Series:
synth.chain(chorus, delay, reverb, Tone.Destination);

// Parallel:
synth.fan(reverb, delay);
reverb.toDestination();
delay.toDestination();
```

**Reverb (convolution, auto-generated IR):**

```javascript
const reverb = new Tone.Reverb({ decay: 8, wet: 0.7, preDelay: 0.1 }).toDestination();
await reverb.generate();
```

**Freeverb (algorithmic, no async needed):**

```javascript
const freeverb = new Tone.Freeverb({ roomSize: 0.9, dampening: 3000, wet: 0.8 }).toDestination();
```

**FeedbackDelay:**

```javascript
const delay = new Tone.FeedbackDelay({ delayTime: "8n.", feedback: 0.6, wet: 0.4 }).toDestination();
```

**PingPongDelay:**

```javascript
const pingPong = new Tone.PingPongDelay({ delayTime: "4n", feedback: 0.5, wet: 0.35 }).toDestination();
```

**Chorus (must call .start()):**

```javascript
const chorus = new Tone.Chorus({ frequency: 0.3, delayTime: 3.5, depth: 0.7, wet: 0.5 })
  .toDestination().start();
```

**AutoFilter (self-modulating filter sweep):**

```javascript
const autoFilter = new Tone.AutoFilter({
  frequency: "2m", baseFrequency: 200, octaves: 4, depth: 0.8, wet: 1
}).toDestination().start();
```

**Complete ambient effects chain:**

```javascript
const chorus = new Tone.Chorus(0.3, 3.5, 0.7).start();
const delay = new Tone.FeedbackDelay("8n.", 0.5);
const reverb = new Tone.Reverb({ decay: 10, wet: 0.8 });
await reverb.generate();

const pad = new Tone.PolySynth(Tone.FMSynth, {
  options: {
    harmonicity: 2,
    modulationIndex: 3,
    envelope: { attack: 3, decay: 1, sustain: 0.7, release: 5 },
    modulationEnvelope: { attack: 4, sustain: 1, release: 4 }
  }
}).chain(chorus, delay, reverb, Tone.Destination);

pad.volume.value = -12;
delay.wet.value = 0.3;
chorus.wet.value = 0.4;
```

### 2.4 Sequencing and Patterns

**Tone.Loop:**

```javascript
const loop = new Tone.Loop((time) => {
  const notes = ["C3", "E3", "G3", "B3", "D4"];
  const note = notes[Math.floor(Math.random() * notes.length)];
  synth.triggerAttackRelease(note, "2n", time);
}, "1m");

loop.start(0);
Tone.getTransport().start();
```

**Tone.Sequence (evenly-spaced, null = rest):**

```javascript
const seq = new Tone.Sequence((time, note) => {
  synth.triggerAttackRelease(note, "4n", time);
}, ["C3", "E3", null, "G3", ["B3", "A3"], null, "D3", null], "4n");
seq.start(0);
```

**Tone.Pattern (arpeggiator-style):**

```javascript
const pattern = new Tone.Pattern((time, note) => {
  synth.triggerAttackRelease(note, "2n", time);
}, ["C3", "D3", "E3", "G3", "A3"], "random");
pattern.interval = "2n";
pattern.start(0);
```

**Humanize and probability:**

```javascript
loop.probability = 0.7;    // 70% chance each iteration fires
loop.humanize = "32n";     // random timing offset up to a 32nd note
```

### 2.5 Signals and LFO Modulation

```javascript
// LFO modulating filter cutoff
const lfo = new Tone.LFO({ frequency: 0.1, min: 200, max: 2000, type: "sine" });
lfo.connect(filter.frequency);
lfo.start();

// Modulate delay feedback for evolving echoes
const feedbackLFO = new Tone.LFO(0.05, 0.2, 0.8);
feedbackLFO.connect(delay.feedback);
feedbackLFO.start();

// Modulate FMSynth's modulation index for timbral evolution
const modLFO = new Tone.LFO("8m", 1, 15);  // one cycle every 8 measures
modLFO.connect(fmSynth.modulationIndex);
modLFO.start();
```

---

## 3. Generative Music Techniques

### 3.1 Eno-Style Overlapping Loops (Phase Music)

Brian Eno's *Ambient 1: Music for Airports* (1978) uses overlapping loops of different lengths. The key: **loops with incommensurable durations** create combinations that rarely repeat.

From the original *Music for Airports* Track 2/1 -- seven vocal loops:

| Loop | Duration |
|------|----------|
| High Ab | 17.8s |
| Eb | 16.2s |
| C | 20.1s |
| High F | 19.6s |
| Low Ab | 21.3s |
| Low F | 24.7s |
| Db | 31.8s |

**Using prime-number durations maximizes the repeat cycle:**

```javascript
const LOOP_DURATIONS = [17, 19, 23, 29, 31, 37, 41]; // all prime, in seconds
// LCM = ~235,000 years before exact repetition
```

**Implementation:**

```javascript
function musicForAirports(ctx) {
  const reverb = new ConvolverNode(ctx);
  reverb.buffer = createSyntheticIR(ctx, 5, 2);
  reverb.connect(ctx.destination);

  const loops = [
    { note: 'F4',  duration: 19.7, delay: 4.0 },
    { note: 'Ab4', duration: 17.8, delay: 8.1 },
    { note: 'C5',  duration: 21.3, delay: 5.6 },
    { note: 'Db5', duration: 22.1, delay: 12.6 },
    { note: 'Eb5', duration: 18.4, delay: 9.2 },
    { note: 'F5',  duration: 20.0, delay: 14.1 },
    { note: 'Ab5', duration: 17.7, delay: 3.1 },
  ];

  loops.forEach(({ note, duration, delay }) => {
    function play() {
      const osc = ctx.createOscillator();
      const gain = ctx.createGain();
      const now = ctx.currentTime;

      osc.type = 'sine';
      osc.frequency.value = noteToFreq(note);
      gain.gain.setValueAtTime(0, now);
      gain.gain.linearRampToValueAtTime(0.15, now + 2);
      gain.gain.setValueAtTime(0.15, now + 4);
      gain.gain.linearRampToValueAtTime(0, now + 6);

      osc.connect(gain).connect(reverb);
      osc.start(now);
      osc.stop(now + 6.5);
    }

    setTimeout(play, delay * 1000);
    setInterval(play, duration * 1000);
  });
}
```

**Tone.js version with prime-length loops:**

```javascript
const reverb = new Tone.Reverb({ decay: 12, wet: 0.75 });
await reverb.generate();
const delay = new Tone.PingPongDelay("4n", 0.4);

const synth1 = new Tone.Synth({
  oscillator: { type: "triangle" },
  envelope: { attack: 0.8, decay: 0.3, sustain: 0.4, release: 3 }
}).chain(delay, reverb, Tone.Destination);

const synth2 = new Tone.Synth({
  oscillator: { type: "sine" },
  envelope: { attack: 1.2, decay: 0.5, sustain: 0.3, release: 4 }
}).chain(delay, reverb, Tone.Destination);

// 7-note sequence and 11-note sequence at different rates = phasing
const notes1 = ["C4", "E4", "G4", "B4", "D5", "A4", "F4"];
let i1 = 0;
const loop1 = new Tone.Loop((time) => {
  synth1.triggerAttackRelease(notes1[i1++ % notes1.length], "2n", time);
}, "4n");

const notes2 = ["G3", "B3", "D4", "F4", "A4", "C5", "E4", "G4", "B4", "D5", "F5"];
let i2 = 0;
const loop2 = new Tone.Loop((time) => {
  synth2.triggerAttackRelease(notes2[i2++ % notes2.length], "2n", time);
}, "4n.");  // dotted quarter = different rate

loop1.start(0);
loop2.start(0);
Tone.getTransport().bpm.value = 60;
Tone.getTransport().start();
```

### 3.2 Markov Chains

A Markov chain models sequences where the next state depends on the current state (or last N states). States are notes; transitions are weighted probabilities.

**First-order Markov chain:**

```javascript
class MarkovMelodyGenerator {
  constructor(scale, transitionMatrix) {
    this.scale = scale;
    this.matrix = transitionMatrix; // matrix[i][j] = P(scale[j] | scale[i])
    this.currentIndex = Math.floor(Math.random() * scale.length);
  }

  weightedRandom(weights) {
    let r = Math.random();
    let cumulative = 0;
    for (let i = 0; i < weights.length; i++) {
      cumulative += weights[i];
      if (r <= cumulative) return i;
    }
    return weights.length - 1;
  }

  next() {
    this.currentIndex = this.weightedRandom(this.matrix[this.currentIndex]);
    return this.scale[this.currentIndex];
  }
}

// Dorian mode with stepwise-motion bias
const DORIAN = ['D3', 'E3', 'F3', 'G3', 'A3', 'Bb3', 'C4', 'D4'];
const TRANSITIONS = [
  //  D3    E3    F3    G3    A3    Bb3   C4    D4
  [ 0.05, 0.30, 0.10, 0.15, 0.10, 0.05, 0.05, 0.20 ], // from D3
  [ 0.25, 0.05, 0.30, 0.10, 0.15, 0.05, 0.05, 0.05 ], // from E3
  [ 0.10, 0.25, 0.05, 0.30, 0.10, 0.10, 0.05, 0.05 ], // from F3
  [ 0.10, 0.10, 0.20, 0.05, 0.30, 0.10, 0.10, 0.05 ], // from G3
  [ 0.10, 0.05, 0.10, 0.25, 0.05, 0.25, 0.15, 0.05 ], // from A3
  [ 0.05, 0.05, 0.05, 0.10, 0.30, 0.05, 0.30, 0.10 ], // from Bb3
  [ 0.05, 0.05, 0.05, 0.05, 0.15, 0.25, 0.05, 0.35 ], // from C4
  [ 0.30, 0.05, 0.05, 0.10, 0.10, 0.05, 0.25, 0.10 ], // from D4
];

const gen = new MarkovMelodyGenerator(DORIAN, TRANSITIONS);
```

**Second-order Markov chain** (considers previous two notes):

```javascript
class SecondOrderMarkov {
  constructor() {
    this.transitions = new Map(); // "noteA,noteB" -> Map(nextNote -> count)
  }

  train(sequence) {
    for (let i = 0; i < sequence.length - 2; i++) {
      const key = `${sequence[i]},${sequence[i + 1]}`;
      if (!this.transitions.has(key)) this.transitions.set(key, new Map());
      const counts = this.transitions.get(key);
      const next = sequence[i + 2];
      counts.set(next, (counts.get(next) || 0) + 1);
    }
  }

  next(prev1, prev2) {
    const key = `${prev1},${prev2}`;
    const counts = this.transitions.get(key);
    if (!counts) return prev2; // fallback

    const total = Array.from(counts.values()).reduce((a, b) => a + b, 0);
    let r = Math.random() * total;
    for (const [note, count] of counts) {
      r -= count;
      if (r <= 0) return note;
    }
    return Array.from(counts.keys()).pop();
  }
}
```

### 3.3 Stochastic / Probabilistic Selection

**Weighted random:**

```javascript
function weightedChoice(items, weights) {
  const total = weights.reduce((sum, w) => sum + w, 0);
  let r = Math.random() * total;
  for (let i = 0; i < items.length; i++) {
    r -= weights[i];
    if (r <= 0) return items[i];
  }
  return items[items.length - 1];
}

// Root and fifth heavily weighted for drone-like tonal center:
const NOTES   = ['D3', 'E3', 'F3', 'G3', 'A3', 'Bb3', 'C4', 'D4'];
const WEIGHTS = [ 0.25, 0.05, 0.15, 0.10, 0.20, 0.05,  0.10, 0.10];
```

**Gaussian humanization (Box-Muller transform):**

```javascript
function gaussianRandom(mean = 0, stddev = 1) {
  let u1 = Math.random();
  while (u1 === 0) u1 = Math.random();
  const u2 = Math.random();
  return stddev * Math.sqrt(-2.0 * Math.log(u1)) * Math.cos(2.0 * Math.PI * u2) + mean;
}

function humanize(onsets, timingJitter = 0.015) {
  return onsets.map(t => t + gaussianRandom(0, timingJitter));
}
```

**Probability gate with breathing density:**

```javascript
class ProbabilityGate {
  constructor(baseDensity = 0.5, modulationDepth = 0.3, modulationPeriod = 30) {
    this.base = baseDensity;
    this.depth = modulationDepth;
    this.period = modulationPeriod;
    this.start = Date.now() / 1000;
  }

  shouldFire() {
    const elapsed = Date.now() / 1000 - this.start;
    const mod = Math.sin(2 * Math.PI * elapsed / this.period);
    const probability = Math.max(0, Math.min(1, this.base + mod * this.depth));
    return Math.random() < probability;
  }
}
```

**Random walk melody:**

```javascript
class RandomWalkMelody {
  constructor(scale, startIndex, maxStep = 2) {
    this.scale = scale;
    this.index = startIndex;
    this.maxStep = maxStep;
  }

  next() {
    const step = Math.floor(Math.random() * (2 * this.maxStep + 1)) - this.maxStep;
    this.index = Math.max(0, Math.min(this.scale.length - 1, this.index + step));
    return this.scale[this.index];
  }
}
```

### 3.4 Cellular Automata

Simple local rules produce complex, fractal-like patterns. **Rule 30** produces pseudo-random patterns (good for rhythm). **Rule 110** is Turing-complete (good for melodic motifs).

```javascript
class ElementaryCA {
  constructor(size, ruleNumber) {
    this.size = size;
    this.rule = {};
    for (let i = 0; i < 8; i++) {
      const pattern = [(i >> 2) & 1, (i >> 1) & 1, i & 1].join('');
      this.rule[pattern] = (ruleNumber >> i) & 1;
    }
    this.cells = new Uint8Array(size);
    this.cells[Math.floor(size / 2)] = 1;
  }

  step() {
    const next = new Uint8Array(this.size);
    for (let i = 0; i < this.size; i++) {
      const left = this.cells[(i - 1 + this.size) % this.size];
      const center = this.cells[i];
      const right = this.cells[(i + 1) % this.size];
      next[i] = this.rule[`${left}${center}${right}`];
    }
    this.cells = next;
    return this.cells;
  }
}
```

**Mapping CA to music (3 strategies):**

1. **Direct mapping** -- active cells trigger notes; cell position = scale degree
2. **Column extraction** -- read one column as a melodic voice
3. **Density mapping** -- row density controls filter cutoff, reverb mix, tempo

```javascript
const ca = new ElementaryCA(12, 30); // Rule 30, 12 cells

// Use as rhythm/density controller:
setInterval(() => {
  ca.step();
  const density = ca.cells.reduce((s, c) => s + c, 0) / ca.cells.length;
  masterGain.gain.linearRampToValueAtTime(0.2 + density * 0.4, ctx.currentTime + 2);
  filter.frequency.setTargetAtTime(200 + density * 4000, ctx.currentTime, 1);
}, 5000);
```

### 3.5 L-Systems (Lindenmayer Systems)

Parallel string rewriting systems that produce self-similar, fractal structures. Musical interpretation uses a "turtle graphics" metaphor:

| Symbol | Action |
|--------|--------|
| `F` | Play current note, advance time |
| `+` | Pitch up one scale step |
| `-` | Pitch down one scale step |
| `[` | Push state (save pitch, duration, time) |
| `]` | Pop state (creates branching/polyphony) |
| `>` | Longer duration |
| `<` | Shorter duration |

```javascript
class LSystem {
  constructor(axiom, rules, iterations) {
    this.axiom = axiom;
    this.rules = rules;
    this.iterations = iterations;
  }

  generate() {
    let current = this.axiom;
    for (let i = 0; i < this.iterations; i++) {
      let next = '';
      for (const char of current) {
        next += this.rules[char] || char;
      }
      current = next;
    }
    return current;
  }
}

// Fractal plant melody -- self-similar branching
const plant = new LSystem('F', { 'F': 'F[+F]F[-F]F' }, 3);
const str = plant.generate();
// "F[+F]F[-F]F[+F[+F]F[-F]F]F[+F]F[-F]F[-F[+F]F[-F]F]F[+F]F[-F]F"
```

**Interpreter:**

```javascript
class LSystemInterpreter {
  constructor(scale) {
    this.scale = scale;
  }

  interpret(str) {
    const score = [];
    let pitch = Math.floor(this.scale.length / 2);
    let time = 0, dur = 0.5, vel = 80;
    const stack = [];

    for (const sym of str) {
      switch (sym) {
        case 'F':
          score.push({
            note: this.scale[Math.max(0, Math.min(pitch, this.scale.length - 1))],
            time, duration: dur, velocity: vel
          });
          time += dur;
          break;
        case '+': pitch = Math.min(pitch + 1, this.scale.length - 1); break;
        case '-': pitch = Math.max(pitch - 1, 0); break;
        case '[': stack.push({ pitch, time, dur, vel }); break;
        case ']':
          if (stack.length) {
            const s = stack.pop();
            pitch = s.pitch; time = s.time; dur = s.dur; vel = s.vel;
          }
          break;
        case '>': dur *= 1.5; break;
        case '<': dur *= 0.667; break;
      }
    }
    return score;
  }
}
```

### 3.6 Constraint-Based Composition

Define harmonic rules, voice leading constraints, and let a solver find valid voicings:

```javascript
class ConstraintComposer {
  constructor(scale, voices, ranges) {
    this.scale = scale;
    this.voices = voices;
    this.ranges = ranges;
    this.constraints = [];
  }

  addConstraint(fn) { this.constraints.push(fn); }

  isValid(chord, prev) {
    return this.constraints.every(fn => fn(chord, prev, this.scale));
  }

  generateChord(prev = null, maxAttempts = 1000) {
    for (let i = 0; i < maxAttempts; i++) {
      const chord = this.ranges.map(r => {
        const valid = this.scale.filter(n => n >= r.min && n <= r.max);
        return valid[Math.floor(Math.random() * valid.length)];
      });
      if (this.isValid(chord, prev)) return chord;
    }
    return null;
  }

  generateProgression(length) {
    const prog = [];
    let prev = null;
    for (let i = 0; i < length; i++) {
      const chord = this.generateChord(prev);
      if (chord) prog.push(chord);
      prev = chord;
    }
    return prog;
  }
}

// Example constraints:
// No parallel fifths/octaves
composer.addConstraint((chord, prev) => {
  if (!prev) return true;
  for (let i = 0; i < chord.length; i++) {
    for (let j = i + 1; j < chord.length; j++) {
      const int = Math.abs(chord[i] - chord[j]) % 12;
      const prevInt = Math.abs(prev[i] - prev[j]) % 12;
      if ((int === 7 && prevInt === 7) || (int === 0 && prevInt === 0)) {
        if (Math.sign(chord[i] - prev[i]) === Math.sign(chord[j] - prev[j])) return false;
      }
    }
  }
  return true;
});

// Smooth voice leading (max 5 semitones per voice)
composer.addConstraint((chord, prev) => {
  if (!prev) return true;
  return chord.every((n, i) => Math.abs(n - prev[i]) <= 5);
});

// Common tone
composer.addConstraint((chord, prev) => {
  if (!prev) return true;
  return chord.some((n, i) => n === prev[i]);
});
```

### 3.7 Spectral Freeze

Capture a single FFT frame's magnitude spectrum and hold it indefinitely -- infinite sustain of the harmonic content at that moment. Requires an AudioWorklet for real-time processing:

```javascript
// spectral-freeze-processor.js (simplified concept)
class SpectralFreezeProcessor extends AudioWorkletProcessor {
  constructor() {
    super();
    this.frozenMagnitudes = null;
    this.isFrozen = false;
    this.phaseAccum = null;

    this.port.onmessage = (e) => {
      if (e.data.type === 'freeze') this.isFrozen = true;
      if (e.data.type === 'unfreeze') {
        this.isFrozen = false;
        this.frozenMagnitudes = null;
      }
    };
  }

  // In process(): perform FFT, if frozen capture magnitudes once,
  // then continuously resynthesize using frozen magnitudes with advancing phases.
  // See full implementation in the spectral processing section of the sources.
}

registerProcessor('spectral-freeze-processor', SpectralFreezeProcessor);
```

**Other spectral techniques:**
- **Spectral smear** -- Gaussian-blur the magnitude spectrum for diffuse pad textures
- **Spectral gate** -- only pass bins above a threshold for sparse, resonant textures
- **Whisperization** -- randomize all phases for breathy, noise-like textures
- **Robotization** -- zero all phases for metallic, bell-like textures

### 3.8 Granular Synthesis

Break audio into tiny grains (10-100ms), reassemble with transformations.

**Window functions:**

```javascript
const GrainWindows = {
  hanning(size) {
    const w = new Float32Array(size);
    for (let i = 0; i < size; i++) w[i] = 0.5 * (1 - Math.cos(2 * Math.PI * i / size));
    return w;
  },
  gaussian(size, sigma = 0.25) {
    const w = new Float32Array(size);
    const center = size / 2, denom = sigma * center;
    for (let i = 0; i < size; i++) {
      const x = (i - center) / denom;
      w[i] = Math.exp(-0.5 * x * x);
    }
    return w;
  },
  tukey(size, alpha = 0.5) {
    const w = new Float32Array(size);
    const taper = Math.floor(alpha * size / 2);
    for (let i = 0; i < size; i++) {
      if (i < taper) w[i] = 0.5 * (1 - Math.cos(Math.PI * i / taper));
      else if (i > size - taper - 1) w[i] = 0.5 * (1 - Math.cos(Math.PI * (size - 1 - i) / taper));
      else w[i] = 1;
    }
    return w;
  }
};
```

**Granular engine:**

```javascript
class GranularSynth {
  constructor(ctx, buffer) {
    this.ctx = ctx;
    this.buffer = buffer;
    this.output = ctx.createGain();
    this.params = {
      grainSize: 0.08,       // 80ms
      grainDensity: 20,      // grains/sec
      position: 0.5,         // 0-1 in buffer
      positionSpread: 0.02,
      pitch: 1.0,
      pitchSpread: 0.05,
      panSpread: 0.3,
      windowType: 'hanning',
      reverse: 0.0,          // probability of reverse grains
    };
    this.isPlaying = false;
    this.nextGrainTime = 0;
  }

  connect(dest) { this.output.connect(dest); return this; }

  scheduleGrain(time) {
    const p = this.params;
    const grainSamples = Math.floor(p.grainSize * this.ctx.sampleRate);
    const window = GrainWindows[p.windowType](grainSamples);

    const pos = Math.max(0, Math.min(1, p.position + (Math.random() - 0.5) * 2 * p.positionSpread));
    const bufferPos = pos * (this.buffer.duration - p.grainSize);
    const rate = p.pitch + (Math.random() - 0.5) * 2 * p.pitchSpread;
    const pan = Math.max(-1, Math.min(1, (Math.random() - 0.5) * 2 * p.panSpread));
    const shouldReverse = Math.random() < p.reverse;

    const grain = this.ctx.createBuffer(this.buffer.numberOfChannels, grainSamples, this.ctx.sampleRate);
    for (let ch = 0; ch < this.buffer.numberOfChannels; ch++) {
      const src = this.buffer.getChannelData(ch);
      const dst = grain.getChannelData(ch);
      const start = Math.floor(bufferPos * this.ctx.sampleRate);
      for (let i = 0; i < grainSamples; i++) {
        let idx = start + (shouldReverse ? grainSamples - 1 - i : i);
        idx = Math.max(0, Math.min(src.length - 1, idx));
        dst[i] = src[idx] * window[i];
      }
    }

    const source = this.ctx.createBufferSource();
    source.buffer = grain;
    source.playbackRate.value = Math.abs(rate);

    const panner = this.ctx.createStereoPanner();
    panner.pan.value = pan;

    source.connect(panner).connect(this.output);
    source.start(time);
  }

  start() {
    if (this.isPlaying) return;
    this.isPlaying = true;
    this.nextGrainTime = this.ctx.currentTime;

    this.interval = setInterval(() => {
      const grainInterval = 1 / this.params.grainDensity;
      while (this.nextGrainTime < this.ctx.currentTime + 0.1) {
        this.scheduleGrain(this.nextGrainTime);
        this.nextGrainTime += grainInterval + (Math.random() - 0.5) * grainInterval * 0.5;
      }
    }, 25);
  }

  stop() {
    this.isPlaying = false;
    clearInterval(this.interval);
  }
}
```

---

## 4. Key Libraries

### 4.1 Strudel

Live-coding music pattern language for the browser -- a port of TidalCycles to JavaScript. Zero-install REPL at [strudel.cc](https://strudel.cc/).

```javascript
// Strudel ambient patch
setcps(0.25)
stack(
  chord("<Dm9 Am9 FMaj7 Em7>/8").voicing().s("gm_epiano1:1")
    .room(0.8).lpf(sine.range(400, 2000).slow(16)).gain(0.4).delay(0.5),
  n("<0 1 2 3>").s("breath").slow(8).room(1).gain(0.15)
    .lpf(sine.range(200, 800).slow(20))
).size(6)
```

### 4.2 Gibber

Creative coding environment for live-coding music in the browser at [gibber.cc](https://gibber.cc/). Plain JavaScript with `.seq()` method for sequencing any parameter.

```javascript
a = FM('glockenspiel')
  .note.seq( Rndi(0, 12), [1/4, 1/2, 1, 2].rnd() )
  .fx.add( Delay({ time: 1/6, feedback: 0.5 }) )
  .fx.add( Reverb() )
```

### 4.3 Elementary Audio

Functional, declarative audio DSP library. Describes audio graphs as pure function compositions with a virtual audio graph (like a virtual DOM for audio).

```javascript
import { el } from '@elemaudio/core';
import WebRenderer from '@elemaudio/web-renderer';

function ambientLayer(freq, lfoRate, lfoDepth) {
  const lfo = el.mul(el.cycle(lfoRate), lfoDepth);
  const osc = el.cycle(el.add(freq, lfo));
  return el.lowpass(el.add(400, el.mul(el.cycle(lfoRate * 0.3), 300)), 1.0, osc);
}

const left = el.add(el.mul(ambientLayer(110, 0.05, 2), 0.3), el.mul(el.noise(), 0.02));
const right = el.add(el.mul(ambientLayer(165, 0.07, 3), 0.3), el.mul(el.noise(), 0.02));
await core.render(left, right);
```

### 4.4 RNBO (Cycling '74)

Export Max/MSP patches to WebAssembly via cloud compiler. Each device becomes an AudioWorkletNode.

```javascript
import { createDevice } from "@rnbo/js";

const response = await fetch("export/patch.export.json");
const patcher = await response.json();
const device = await createDevice({ context: ctx, patcher });
device.node.connect(ctx.destination);

// Control parameters from generative JS code:
device.parametersById.get("reverb_mix").value = 0.75;
```

### 4.5 Faust

Functional audio DSP language compiled to WebAssembly. Extremely concise for DSP; use JS for generative logic.

```faust
import("stdfaust.lib");
freq = hslider("freq", 220, 50, 2000, 0.01);
detune = hslider("detune", 0.5, 0, 5, 0.01);
osc1 = os.sawtooth(freq);
osc2 = os.sawtooth(freq + detune);
process = (osc1 + osc2) / 2 : fi.resonlp(800, 0.5);
```

```javascript
import { FaustMonoDspGenerator, FaustCompiler } from "@grame/faustwasm";
const generator = new FaustMonoDspGenerator();
await generator.compile(compiler, "AmbientPad", dspCode, "-I libraries/");
const node = await generator.createNode(ctx);
node.connect(ctx.destination);
```

### 4.6 SuperCollider via WebAssembly (SuperSonic)

Full SuperCollider `scsynth` engine in the browser via WASM AudioWorklet. Alpha status, but the synthesis core is solid.

```html
<script type="module">
import { SuperSonic } from "https://unpkg.com/supersonic-scsynth@latest";

const sonic = new SuperSonic({ /* base URLs */ });
await sonic.init();
await sonic.loadSynthDef("sonic-pi-prophet");

sonic.send("/s_new", "sonic-pi-prophet", -1, 0, 0,
  "note", 60, "attack", 2.0, "release", 4.0, "cutoff", 80);
</script>
```

---

## 5. Practical Code Patterns

### 5.1 Complete Generative Ambient Engine (Web Audio API)

```javascript
async function createAmbientEngine(ctx) {
  const osc1 = new OscillatorNode(ctx, { type: 'sine', frequency: 110 });
  const osc2 = new OscillatorNode(ctx, { type: 'triangle', frequency: 110.5 });
  const oscGain1 = new GainNode(ctx, { gain: 0.3 });
  const oscGain2 = new GainNode(ctx, { gain: 0.2 });
  const filter = new BiquadFilterNode(ctx, { type: 'lowpass', frequency: 800, Q: 2 });
  const delay = new DelayNode(ctx, { maxDelayTime: 2.0 });
  delay.delayTime.value = 0.75;
  const delayFeedback = new GainNode(ctx, { gain: 0.45 });
  const delayMix = new GainNode(ctx, { gain: 0.5 });
  const reverb = new ConvolverNode(ctx);
  reverb.buffer = createSyntheticIR(ctx, 5, 2);
  const reverbMix = new GainNode(ctx, { gain: 0.6 });
  const panner = new StereoPannerNode(ctx, { pan: 0 });
  const panLFO = new OscillatorNode(ctx, { type: 'sine', frequency: 0.07 });
  const panDepth = new GainNode(ctx, { gain: 0.5 });
  panLFO.connect(panDepth).connect(panner.pan);
  const compressor = new DynamicsCompressorNode(ctx, {
    threshold: -20, knee: 10, ratio: 4, attack: 0.01, release: 0.3
  });

  // Wiring
  osc1.connect(oscGain1).connect(filter);
  osc2.connect(oscGain2).connect(filter);
  filter.connect(panner);
  filter.connect(delay);
  delay.connect(delayFeedback).connect(delay);
  delay.connect(delayMix).connect(panner);
  panner.connect(reverb).connect(reverbMix).connect(compressor);
  panner.connect(compressor);
  compressor.connect(ctx.destination);

  osc1.start(); osc2.start(); panLFO.start();

  // Evolving filter
  (function evolve() {
    const freq = 300 + Math.random() * 1500;
    filter.frequency.setTargetAtTime(freq, ctx.currentTime, 4 + Math.random() * 6);
    setTimeout(evolve, 5000 + Math.random() * 10000);
  })();
}
```

### 5.2 Complete Ambient Scene (Tone.js)

```javascript
import * as Tone from "tone";

async function createAmbientScene() {
  // Effects bus
  const masterReverb = new Tone.Freeverb({ roomSize: 0.92, dampening: 2000, wet: 0.75 })
    .toDestination();
  const masterDelay = new Tone.PingPongDelay({ delayTime: "4n.", feedback: 0.35, wet: 0.25 })
    .connect(masterReverb);

  // Layer 1: Drone
  const drone = new Tone.FMSynth({
    harmonicity: 1.5, modulationIndex: 4,
    envelope: { attack: 10, sustain: 1, release: 15 },
    modulationEnvelope: { attack: 12, sustain: 1, release: 12 }
  }).connect(masterReverb);
  drone.volume.value = -16;

  const droneLFO = new Tone.LFO(0.015, 1, 10);
  droneLFO.connect(drone.modulationIndex);
  droneLFO.start();

  // Layer 2: Pad chords
  const pad = new Tone.PolySynth(Tone.Synth, {
    maxPolyphony: 8,
    options: {
      oscillator: { type: "fattriangle", count: 3, spread: 15 },
      envelope: { attack: 4, decay: 1, sustain: 0.7, release: 6 }
    }
  }).connect(masterDelay);
  pad.volume.value = -20;

  const chords = [
    ["C3", "G3", "B3", "E4"],
    ["A2", "E3", "G3", "C4"],
    ["F2", "C3", "A3", "D4"],
    ["D3", "A3", "F3", "B3"],
  ];
  let ci = 0;
  const chordLoop = new Tone.Loop((time) => {
    pad.triggerAttackRelease(chords[ci++ % chords.length], "3m", time);
  }, "4m");

  // Layer 3: Generative melody
  const melody = new Tone.Synth({
    oscillator: { type: "sine" },
    envelope: { attack: 0.8, decay: 0.3, sustain: 0.2, release: 3 }
  }).connect(masterDelay);
  melody.volume.value = -22;

  const scale = ["C4", "D4", "E4", "G4", "A4", "C5", "D5", "E5"];
  const melodyPattern = new Tone.Pattern((time, note) => {
    melody.triggerAttackRelease(note, "2n", time);
  }, scale, "random");
  melodyPattern.interval = "2n";
  melodyPattern.probability = 0.6;

  // Layer 4: Noise wash
  const noise = new Tone.NoiseSynth({
    noise: { type: "pink" },
    envelope: { attack: 8, sustain: 1, release: 10 }
  });
  const noiseFilter = new Tone.AutoFilter({
    frequency: 0.05, baseFrequency: 80, octaves: 4
  }).connect(masterReverb).start();
  noise.connect(noiseFilter);
  noise.volume.value = -26;

  // Start
  await Tone.start();
  const transport = Tone.getTransport();
  transport.bpm.value = 55;

  drone.triggerAttack("C2");
  noise.triggerAttack();
  chordLoop.start(0);
  melodyPattern.start(0);
  transport.start();

  // Slow tempo drift
  transport.bpm.rampTo(48, 300);
}

document.querySelector("#start").addEventListener("click", createAmbientScene);
```

### 5.3 Generative Drone with LFO Modulation

```javascript
const drone = new Tone.FMSynth({
  harmonicity: 1.5, modulationIndex: 5,
  oscillator: { type: "sine" },
  envelope: { attack: 8, decay: 2, sustain: 1, release: 10 },
  modulation: { type: "triangle" },
  modulationEnvelope: { attack: 10, sustain: 1, release: 10 }
});

const filter = new Tone.Filter(600, "lowpass", -24);
const reverb = new Tone.Freeverb({ roomSize: 0.95, dampening: 2000, wet: 0.85 });
drone.chain(filter, reverb, Tone.Destination);

const filterLFO = new Tone.LFO({ frequency: 0.03, min: 200, max: 1200, type: "sine" });
filterLFO.connect(filter.frequency);
filterLFO.start();

const modLFO = new Tone.LFO({ frequency: 0.02, min: 2, max: 12, type: "triangle" });
modLFO.connect(drone.modulationIndex);
modLFO.start();

drone.triggerAttack("C2"); // indefinite drone
```

### 5.4 Wind Chime / Bell Pattern

```javascript
const reverb = new Tone.Reverb({ decay: 10, wet: 0.8 });
await reverb.generate();

const bell = new Tone.MetalSynth({
  frequency: 300,
  envelope: { attack: 0.001, decay: 2.5, release: 0.3 },
  harmonicity: 5.1, modulationIndex: 20, resonance: 5000, octaves: 2
}).connect(reverb);
reverb.toDestination();
bell.volume.value = -22;

const freqs = [200, 300, 400, 500, 600, 800, 1000];

const chimeLoop = new Tone.Loop((time) => {
  bell.frequency.setValueAtTime(freqs[Math.floor(Math.random() * freqs.length)], time);
  bell.triggerAttackRelease("32n", time);
}, "8n");

chimeLoop.probability = 0.15;  // sparse
chimeLoop.humanize = "16n";
chimeLoop.start(0);
Tone.getTransport().bpm.value = 80;
Tone.getTransport().start();
```

### 5.5 All-Techniques Combined Engine

```javascript
class AmbientEngine {
  constructor(ctx) {
    this.ctx = ctx;
    this.master = ctx.createGain();
    this.master.gain.value = 0.5;
    this.master.connect(ctx.destination);
    this.gate = new ProbabilityGate(0.3, 0.2, 60);
  }

  init() {
    // Eno-style phase layers
    const notes = ['D3', 'A3', 'D4', 'F#4', 'A4'];
    const durations = [17, 19, 23, 29, 31]; // prime seconds
    notes.forEach((note, i) => {
      const play = () => {
        if (this.gate.shouldFire()) this.playTone(note, 8);
      };
      setTimeout(play, Math.random() * 5000);
      setInterval(play, durations[i] * 1000);
    });

    // Markov melody
    const scale = ['D4','E4','F#4','G4','A4','B4','C#5','D5'];
    const markov = new MarkovMelodyGenerator(scale, /* matrix */);
    const playNext = () => {
      if (this.gate.shouldFire()) this.playTone(markov.next(), 3);
      setTimeout(playNext, 2000 + gaussianRandom(2000, 800));
    };
    setTimeout(playNext, 3000);

    // CA density controller
    const ca = new ElementaryCA(8, 30);
    setInterval(() => {
      ca.step();
      const density = ca.cells.reduce((s, c) => s + c, 0) / ca.cells.length;
      this.master.gain.linearRampToValueAtTime(0.2 + density * 0.4, this.ctx.currentTime + 2);
    }, 5000);
  }

  playTone(note, duration) { /* ... create osc, gain, connect, schedule ... */ }
}
```

---

## 6. Notable References

### 6.1 Generative.fm

[generative.fm](https://generative.fm/) by Alex Bainter. 50+ generative ambient pieces running in the browser, all built on Tone.js.

**Architecture:**
- Each piece is a self-contained npm package exporting an `activate` function
- Receives `AudioContext`, destination, and sample library
- Uses Tone.js Transport, Loop, Part, Pattern for scheduling
- Weighted random note selection, probability gates, slowly evolving parameters
- Lerna monorepo, React frontend, static hosting
- Samples from Versilian Studios Chamber Orchestra

**Typical piece pattern:**

```javascript
export default async function activate({ context, destination, sampleLibrary }) {
  const piano = await sampleLibrary.request(context, 'vsco2-piano-mf');
  const reverb = new Tone.Reverb({ decay: 8, wet: 0.7 }).connect(destination);
  const sampler = new Tone.Sampler(piano).connect(reverb);

  const notes = ['C3', 'E3', 'G3', 'B3', 'D4'];

  function scheduleNext() {
    const note = notes[Math.floor(Math.random() * notes.length)];
    const delay = 1 + Math.random() * 8;
    Tone.Transport.scheduleOnce((time) => {
      sampler.triggerAttackRelease(note, 2 + Math.random() * 6, time, 0.2 + Math.random() * 0.3);
      scheduleNext();
    }, `+${delay}`);
  }

  for (let i = 0; i < 3; i++) scheduleNext();
  return () => { sampler.dispose(); reverb.dispose(); };
}
```

### 6.2 Brian Eno Techniques Summary

| Album / Work | Year | Technique |
|---|---|---|
| *Discreet Music* | 1975 | Two melodic loops (63s, 68s) phasing against each other |
| *Music for Airports* | 1978 | 7-8 tape loops per track, incommensurable lengths |
| *Generative Music 1* | 1996 | SSEYO Koan software -- probabilistic MIDI generation |
| *Reflection* | 2017 | iOS app generating infinite ambient in real-time |

**Core principles:**
- Use loops of prime or incommensurable lengths
- Fewer notes, longer durations, wide reverb
- Let the system make decisions, not the composer
- "The composer is the gardener" -- set up conditions for music to grow

### 6.3 Tero Parviainen's Tutorial

[JavaScript Systems Music](https://teropa.info/blog/2016/07/28/javascript-systems-music.html) -- the definitive tutorial on implementing Eno-style generative systems in Web Audio. Covers Music for Airports, Steve Reich's phase music, and more with interactive examples.

Also: [How Generative Music Works](https://teropa.info/loop/) -- interactive visual explainer of phase-based generative music.

### 6.4 Ambient Garden

[ambient.garden](https://ambient.garden/) -- an interactive 3D generative audio landscape. Built with custom "tealib" audio framework, some Faust DSP compiled to WASM, and Three.js. Also released as a streaming album. Open source at [github.com/pac-dev/AmbientGarden](https://github.com/pac-dev/AmbientGarden).

---

## 7. Performance

### 7.1 AudioWorklet

Always use `AudioWorklet` for custom DSP instead of the deprecated `ScriptProcessorNode`. AudioWorklet runs on the dedicated audio rendering thread, avoiding main-thread jitter.

### 7.2 GC Avoidance

Garbage collection pauses can cause audio glitches. Strategies:
- **Pre-allocate buffers** -- create Float32Arrays once, reuse them
- **Object pools** -- pool oscillator/gain node wrappers for reuse
- **Avoid closures in hot paths** -- the `process()` method runs every ~2.9ms
- **Use TypedArrays** -- `Float32Array`, `Uint8Array` avoid boxed numbers
- **Don't call `postMessage` every process cycle** -- batch updates

```javascript
// BAD: allocating every process() call
process(inputs, outputs) {
  const data = new Float32Array(128); // GC pressure!
  // ...
}

// GOOD: pre-allocated
constructor() {
  super();
  this.tempBuffer = new Float32Array(128);
}
process(inputs, outputs) {
  const data = this.tempBuffer; // reuse
  // ...
}
```

### 7.3 Mobile Limits

- **Single AudioContext** -- iOS Safari allows only one
- **Lower polyphony** -- target 8-16 simultaneous voices, not 64
- **Shorter convolution IRs** -- use 1-2 second IRs on mobile, not 5+
- **Reduced sample rates** -- 44100 is fine; avoid 96000
- **Background audio** -- may be suspended when app is backgrounded

### 7.4 Autoplay Policies

See [Section 1.6](#16-autoplay-policies). Always require a user gesture before creating or resuming an AudioContext.

### 7.5 Memory

- Dispose unused nodes: `node.disconnect()` or `toneNode.dispose()`
- Release AudioBuffers: set `buffer = null` when no longer needed
- Monitor with `performance.memory` (Chrome) or browser DevTools

### 7.6 Node Count

Each Web Audio node has overhead. For dense generative textures:
- Reuse nodes where possible (e.g., a single `PolySynth` vs. many individual `Synth` instances)
- Disconnect idle paths
- Use a single reverb bus rather than per-voice reverb

---

## 8. Comparison Table

| Library | Type | Learning Curve | Best For | Realtime | Active | License |
|---|---|---|---|---|---|---|
| **Web Audio API** | Native browser API | Medium-High | Full control, custom DSP | Yes | Stable spec | N/A |
| **Tone.js** | JS framework wrapping Web Audio | Low-Medium | Most generative web audio projects | Yes | Yes | MIT |
| **Strudel** | Live-coding pattern language | Medium | Pattern-based generative music, live performance | Yes | Very active | AGPL-3.0 |
| **Gibber** | Live-coding environment | Low-Medium | Audiovisual live coding, education | Yes | Moderate | MIT |
| **Elementary Audio** | Functional declarative DSP | Medium | Declarative audio apps, portable DSP | Yes | Active | MIT |
| **RNBO** | Visual patcher to web export | Low (visual) | Max/MSP users deploying to web | Yes | Commercial | Proprietary |
| **Faust** | DSP language to WASM | High | High-performance custom DSP | Yes | Very active | GPL-2.0 * |
| **SuperSonic** | Full SC engine in browser | High | Advanced synthesis, full SC power | Yes | Alpha | MIT + GPL-3.0 |

\* Faust compiler is GPL-2.0, but output code can be freely licensed.

**Which tool for what:**

| Goal | Recommended |
|---|---|
| Quick prototype in browser | Strudel or Gibber (zero install) |
| Polished generative web app | Tone.js or Elementary Audio |
| Maximum DSP control | Faust (custom DSP) or raw Web Audio API + AudioWorklet |
| Visual patching to web | RNBO |
| Study generative architecture | Generative.fm source code |
| Full synthesis engine | SuperSonic (SuperCollider WASM) |

---

## 9. Sources

### Web Audio API
- [AudioContext -- MDN](https://developer.mozilla.org/en-US/docs/Web/API/AudioContext)
- [OscillatorNode -- MDN](https://developer.mozilla.org/en-US/docs/Web/API/OscillatorNode)
- [AudioParam -- MDN](https://developer.mozilla.org/en-US/docs/Web/API/AudioParam)
- [A Tale of Two Clocks -- web.dev](https://web.dev/articles/audio-scheduling)
- [Using AudioWorklet -- MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Using_AudioWorklet)
- [Audio Worklet -- Chrome Developers](https://developer.chrome.com/blog/audio-worklet)
- [BiquadFilterNode -- MDN](https://developer.mozilla.org/en-US/docs/Web/API/BiquadFilterNode)
- [ConvolverNode -- MDN](https://developer.mozilla.org/en-US/docs/Web/API/ConvolverNode)
- [Autoplay policy in Chrome](https://developer.chrome.com/blog/autoplay/)
- [Autoplay guide -- MDN](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Autoplay)
- [Web Audio API Best Practices -- MDN](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API/Best_practices)

### Tone.js
- [Tone.js Official Site](https://tonejs.github.io/)
- [Tone.js GitHub](https://github.com/Tonejs/Tone.js)
- [Tone.js API Docs](https://tonejs.github.io/docs/)
- [Tone.js Wiki -- Events, Signals, Transport, Time](https://github.com/Tonejs/Tone.js/wiki)
- [Tone.js Examples](https://tonejs.github.io/examples/)

### Generative Techniques
- [JavaScript Systems Music -- Tero Parviainen](https://teropa.info/blog/2016/07/28/javascript-systems-music.html)
- [How Generative Music Works -- Tero Parviainen](https://teropa.info/loop/)
- [Deconstructing Brian Eno's Music for Airports -- Reverb Machine](https://reverbmachine.com/blog/deconstructing-brian-eno-music-for-airports/)
- [Brian Eno's Endless Music Machines -- Gorilla Sun](https://www.gorillasun.de/blog/brian-enos-endless-music-machines/)
- [Generating Music Using Markov Chains -- HackerNoon](https://hackernoon.com/generating-music-using-markov-chains-40c3f3f46405)
- [Generative Music with JavaScript -- Randomness](https://meleyal.github.io/generative-music-with-javascript/generative/randomness)
- [Score Generation with L-Systems -- Prusinkiewicz (1986)](https://algorithmicbotany.org/papers/score.icmc86.pdf)
- [Algorithmic Composition -- Wikipedia](https://en.wikipedia.org/wiki/Algorithmic_composition)
- [Oblique Strategies -- Wikipedia](https://en.wikipedia.org/wiki/Oblique_Strategies)

### Libraries
- [Strudel REPL](https://strudel.cc/) | [Codeberg](https://codeberg.org/uzu/strudel)
- [Gibber](https://gibber.cc/) | [GitHub](https://github.com/gibber-cc/gibber)
- [Elementary Audio](https://www.elementary.audio/) | [GitHub](https://github.com/elemaudio/elementary)
- [RNBO -- Cycling '74](https://rnbo.cycling74.com/) | [Web Export](https://rnbo.cycling74.com/learn/the-web-export-target)
- [Faust](https://faust.grame.fr/) | [FaustWasm GitHub](https://github.com/grame-cncm/faustwasm)
- [SuperSonic](https://github.com/samaaron/supersonic)

### Projects and Architecture
- [Generative.fm](https://generative.fm/) | [Generators GitHub](https://github.com/generativefm/generators)
- [Alex Bainter -- Building Generative Music Systems](https://alexbainter.gumroad.com/l/generative-music-systems)
- [Ambient Garden](https://ambient.garden/) | [GitHub](https://github.com/pac-dev/AmbientGarden)

### Granular and Spectral
- [Granular Synthesis in the Browser -- DEV](https://dev.to/hexshift/granular-synthesis-in-the-browser-using-web-audio-api-and-audiobuffer-slicing-2o9h)
- [Window Functions for Granular Synthesis](https://michaelkrzyzaniak.com/AudioSynthesis/2_Audio_Synthesis/11_Granular_Synthesis/1_Window_Functions/)
- [PhaseVocoderJS -- GitHub](https://github.com/echo66/PhaseVocoderJS)
- [ZYA Granular Synthesiser](https://zya.github.io/granular/)
