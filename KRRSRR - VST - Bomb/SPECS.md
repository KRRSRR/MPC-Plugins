# Bomb: specifications

All-in-one drum processor for Akai MPC standalone hardware. Version 1.0.0, by KRRSRR.

## Inspiration

Bomb was inspired by **Knock 2 by Plugins That Knock (DECAP)**, a desktop drum processor. Its idea is to chain a set of focused drum modules (transient shaping, saturation, a sub layer, top-end air, clipping and compression), give each a character mode and its own Mix and Input, and let you reorder them.

What was taken: that **overall concept and module line-up**, plus the idea of a randomise button and zero-latency operation.

What was not taken: any code, presets, sound files, interface design, graphics, or the original's control and mode names. Bomb's modes are named after how they behave (for example Snap / Smack / Thump, Warm / Bite / Fuzz / Crush, Clean / Grab / Smooth / Pump). The DSP is written from scratch, and the interface follows KRRSRR's own design with a hard-pink accent.

Bomb also adds things the reference doesn't describe in the same form:

- one **Bomb** macro over all modules;
- **Compare**, an A/B at matched loudness;
- automatic gain matching per module;
- an output ceiling;
- a Dice button with per-module locks and Undo;
- tempo-synced compressor release from the MPC's clock;
- sub hits that start on the kick's polarity, with Replace / Layer / Duck options;
- band focus for the transient shaper.

Bomb is not affiliated with, endorsed by or connected to Plugins That Knock or DECAP. "Knock" is a trademark of its owner and is mentioned only to describe where the inspiration came from.

## General

| | |
|---|---|
| Type | Effect, 2 inputs / 2 outputs |
| Format | VST 2.4 binary interface, MPC Plugin Builder SDK framework (clean-room ABI, no Steinberg code) |
| Platform | MPC OS 3.10.1+, 64-bit (MPC Live III, MPC XL); builds for 32-bit MPCs from source |
| Sample rate | 44.1 kHz |
| Latency | 0 samples (IIR filters only, no look-ahead) |
| Parameters | 71 (stable IDs) |
| Built-in presets | 7: Init, Kick, Snare, Hats, Break, Drum Bus, Master Glue |
| Touch UI | 8 pages on 6 function keys |
| Licence | MIT |

## Signal flow

```
in → Input → [Slot 1 → … → Slot 6] → Mix (vs. input) → Compare → Output → Ceiling → out
```

- **Slots 1–6** each pick a module (or Off). The default order is Sub → Punch → Comp → Saturate → Air → Clip.
- Every module has On, Mode, In (drive, compensated afterwards), Mix (parallel blend) and Lock (protects it from Dice).
- **Exact when idle:** with every module off, the output equals the input to within float rounding (< 1e-7). Band processing adds `x + (g − 1)·band`, so untouched bands are exact.

## Modules

| Module | Modes | Controls | DSP |
|---|---|---|---|
| **Punch** | Snap, Smack, Thump | Attack ±100 %, Sustain ±100 %, Speed, Focus (Full / Lows / Mids / Highs) | Fast/slow envelope ratio drives up to +18 / −24 dB of gain. 3-band split at 150 Hz and 2.5 kHz (Linkwitz-Riley, mid is the exact remainder) |
| **Saturate** | Warm, Bite, Fuzz, Crush | Drive (0–30 dB), Tone (tilt), Bias (asymmetry) | 2× oversampled (8th-order IIR), DC blocker, auto gain match. Crush reduces bit depth and sample rate |
| **Sub** | Layer, Replace | Level, Note (C–B) + Octave 0–3, Decay 30–3000 ms, Drop 0–24 st, Sensitivity, Duck, Color | Kick detection in the low band. Sine with an 808-style pitch drop (35 ms), started on the kick's polarity. Duck or Replace removes the original lows below 120 Hz |
| **Air** | Soft (8 kHz), Bright (5 kHz), Open (12 kHz), Presence (4 kHz) | Amount 0–12 dB, Tame, Width 0–200 % | Parallel high boost that backs off on spiky highs. Width applies to the highs above 3 kHz only |
| **Clip** | Hard, Soft, Round | Drive 0–24 dB, Clip Ceiling −12…0 dB | 2× oversampled, auto gain match |
| **Comp** | Clean (4:1), Grab (8:1, fast, light drive), Smooth (3:1, release slows as gain reduction grows), Pump (12:1) | Amount, Attack 0.1–100 ms, Release 10–1000 ms or tempo sync (1/32–1/2), Multiband | Feed-forward, soft knee, stereo-linked. Multiband uses the same 3-band split, one detector per band |

## Global

| Control | Range / function |
|---|---|
| Bomb | 0–100 %: scales every module's intensity (50 % = as set, 100 % = 2×) |
| Input / Output | ±24 dB / −24…+12 dB |
| Mix | 0–100 % (whole chain against the input) |
| Compare | dry input at the processed loudness, 20 ms crossfade |
| Auto Gain | level matching for Saturate, Clip and Comp (~300 ms) |
| Ceiling | −12…0 dB: linear to 90 %, then a soft knee that never passes the ceiling |
| Oversample | 2× for Saturate and Clip (off halves the CPU) |
| Dice / Undo Dice | randomise the unlocked modules within musical ranges / swap back (and forth) |

## Measured performance

These figures are from an 8-second synthetic drum loop on a desktop core with 128-frame blocks. The MPC is roughly 10–40× slower.

| Measurement | Result |
|---|---|
| CPU, presets | 0.5–1.0 % of a block |
| CPU, everything on (multiband, oversampling) | 1.0 % (0.5 % without oversampling) |
| Host simulator | pass at block sizes 32, 128 and 512 |
| Transparency, all modules off | < 1e-7 difference |

## Not in this version

These are compared with the reference:

- Linear Phase Mode: it needs latency and more CPU than the MPC has to spare.
- Live gain-reduction and intensity meters: an MPC skin can't show plugin-driven meters reliably.
- A separate harmonic "purr"-style module.
- Five Air modes (Bomb has four).
- Hardware-modelled compressor emulations: the Comp modes are behaviour presets, not models of specific units.
