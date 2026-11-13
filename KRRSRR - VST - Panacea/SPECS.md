# Panacea: specifications

Polyphonic wavetable synthesizer for Akai MPC standalone hardware. Version 1.0.0, by KRRSRR.

## Inspiration

Panacea started from a desktop spec ("Panacea SDB") that used **Serum 2 by Xfer Records** as its feature reference: two wavetable oscillators with morphing, warp and unison, a sub and noise oscillator, a multimode filter, envelopes, LFOs, a modulation matrix and an FX rack. In practice the layout lands closer to Serum 1 (two main oscillators) than Serum 2 (three oscillators and several oscillator types).

What was taken: the **workflow ideas**. Wavetable position as the main morph control, warp modes that bend the oscillator's phase, per-source filter routing, source → destination modulation with amounts, and macros.

What was not taken: any code, wavetables, presets, sound files, interface design, graphics or naming. Everything in Panacea was written from scratch:

- the DSP is original;
- the 9 factory wavetables are generated from mathematical formulas in `src/wavetable.h`;
- the presets are original;
- the interface follows KRRSRR's own "Mod Matrix" design.

The cross-oscillator warp modes (true FM, phase modulation, ring modulation) are general synthesis techniques and are implemented independently.

Panacea is not affiliated with, endorsed by or connected to Xfer Records. "Serum" is a trademark of its owner and is mentioned only to describe where the inspiration came from.

## General

| | |
|---|---|
| Type | Instrument (synth), 0 inputs / 2 outputs |
| Format | VST 2.4 binary interface, MPC Plugin Builder SDK framework (clean-room ABI, no Steinberg code) |
| Platform | MPC OS 3.10.1+, 64-bit (MPC Live III, MPC XL); builds for 32-bit MPCs from source |
| Sample rate | 44.1 kHz (MPC fixed) |
| Latency | 0 samples |
| Polyphony | 1–8 voices; Poly, Mono, Legato |
| Parameters | 166 (stable IDs, saved in projects) |
| Built-in presets | 10 |
| Touch UI | 12 pages on 6 function keys; each card row is one Q-Link bank of four |
| Memory | about 11 MB of wavetables, shared by all instances (built once when the plugin loads) |
| Licence | MIT |

## Signal flow

```
Osc A ─┐                         ┌─> Filter ─┐
Osc B ─┼─ Send on (per source) ──┘           ├─> × Amp env × velocity ─> sum of voices
Sub   ─┤                                     │
Noise ─┴─ Send off ──────────────────────────┘
sum ─> Distortion ─> EQ ─> Mod FX ─> Delay ─> Reverb ─> Comp ─> Width ─> Volume ─> safety limiter ─> out
```

## Oscillators A and B (identical)

| Control | Range |
|---|---|
| Table | 9 factory + 4 user slots |
| WT Pos | 0–100 % (smooth crossfade between frames, or Step mode) |
| Octave / Semi / Fine | −4…+4 oct, −12…+12 st, ±100 ct |
| Level, Pan | 0–100 %, ±100 |
| Unison | 1–8 voices |
| Detune | 0–100 % (up to ±1 semitone) |
| Blend | 0–100 % (centre versus side voices) |
| Width | 0–100 % |
| Uni Mode | Linear, Harmonic, Stack (semitone steps) |
| Phase, Rand Phase | 0–360°, 0–100 % |
| Warp | Off, Sync, Bend +, Bend −, PWM, Asym +, Asym −, Mirror, Quantize, FM X-Osc (true linear through-zero FM), PD X-Osc, Ring X-Osc |
| Warp Amt | 0–100 % |

**Wavetable engine**

- Up to 64 frames of 2048 samples per table.
- Each frame is converted with an FFT into **11 band-limited mip levels**: level *k* keeps harmonics 1…1024/2ᵏ, stored at 8 samples per period of the top harmonic, with a 512-sample minimum.
- Playback picks the lowest level whose top harmonic stays below 0.45 × the sample rate.
- Warp modes that add harmonics move up a level instead of oversampling.

**Factory tables** (generated): Basic, Harm Build, Pulse PWM, Vowels, Sync Sweep, Digital FM, Bit Steps, Spec Comb, Noise Cloud.

**User tables**: `Content/Wavetables/User1–4.wav`, read when the plugin loads.

- Frames of 2048 samples, up to 64 frames, mono or the first channel.
- A file shorter than 2048 samples is treated as a single cycle and resampled to 2048.

## Sub and noise

| Source | Controls |
|---|---|
| Sub (PolyBLEP) | Sine, Triangle, Saw, Square, Pulse 25 %, Pulse 12.5 %; Octave −2…+2; Level; Pan |
| Noise (generated) | White, Pink, Brown, Crackle, Pitched (sample-and-hold that follows the key); Level; Pan; Color (dark ↔ bright) |

## Filter (one stereo filter per voice)

| | |
|---|---|
| Types | LP 12, LP 24, HP 12, HP 24, BP 12, Notch (zero-delay-feedback state-variable filter), Ladder (4-pole TPT with feedback saturation), Comb +, Comb −, Formant (a-e-i-o-u morph) |
| Controls | Cutoff 20 Hz–20 kHz, Resonance, Drive, Env2>Cutoff ±96 st, Keytrack 0–100 %, Mix, Vel>Cutoff 0–48 st |
| Routing | Send A / B / Sub / Noise: each source goes through the filter or bypasses it |

## Modulation

| | |
|---|---|
| Envelopes | Amp (ENV 1), Env 2, Env 3: ADSR, 1 ms–30 s, exponential curves |
| LFOs 1–4 | Sine, Triangle, Saw Up, Saw Down, Square, S&H, Smooth Rnd. Modes: Free (locked to song position when synced), Trig, Env (one-shot). Rate 0.01–40 Hz or sync 1/64–8 bars. Rise, Delay (0–5 s each), Phase |
| Mod slots | 8 × (Source, Destination, Amount ±100 %, Curve: Linear / Exp / Log / S-Curve) |
| Sources | Env 2–3, LFO 1–4, Velocity, Note, Mod Wheel, Aftertouch (channel and poly), Macro 1–8, Random (per note) |
| Destinations | Pitch; per oscillator: Pitch, WT Pos, Warp, Level, Pan, Detune; Sub Level, Noise Level, Cutoff, Resonance, Flt Drive, Amp, Pan, LFO 1–4 Rate |
| Macros | 8 (0–100 %), available as mod sources |
| Timing | Modulation is computed every 16 samples. Pitch, WT position and gains are ramped per sample. MIDI is sample-accurate |

## FX (fixed order)

| Unit | Controls |
|---|---|
| Distortion | Soft, Tube, Hard, Diode, Fold, Crush; Drive, Tone, Mix |
| EQ | Low shelf 200 Hz, high shelf 4 kHz, ±12 dB |
| Mod FX | Chorus, Flanger, Phaser (6-stage); Rate, Depth, Feedback, Mix |
| Delay | Tempo sync or 1–2000 ms, Feedback, Ping Pong, Tone, Mix (4 s buffer) |
| Reverb | 8-line feedback delay network; Size, Decay 0.3–20 s, Damping, Pre-delay 0–200 ms, Mix |
| Comp | One-knob glue compressor |
| Width | 0–200 % (mid/side) |

## Measured performance

These figures are from the SDK host simulator and a test harness on a desktop CPU core. The MPC is roughly 10–40× slower.

| Measurement | Result |
|---|---|
| CPU, typical presets (8 notes held) | 0.3–1.8 % of a 128-frame block |
| CPU, worst case (8 voices × 2 × 8 unison, FM, LP 24, all FX) | 3.6 % |
| Aliasing, saw at C7 (2093 Hz), filter off | non-harmonic components −90 dB relative to the fundamental |
| Host simulator | pass at block sizes 32, 128 and 512 (state round trip, parameter sweep, no NaN, denormals or clipping) |

## Not in this version

These are compared with the desktop spec and Serum 2:

- a third oscillator, and the sample, granular, spectral and multisample oscillator types;
- a second filter, dual warp, and more than 4 LFOs or 3 envelopes;
- drawable LFO curves, drag-and-drop modulation, and a reorderable FX rack;
- a wavetable editor or import screen;
- MPE, an arpeggiator and a clip sequencer.

Some of these are platform limits: the MPC has no plugin windows and modest CPUs. Others were left out to keep the scope small.
