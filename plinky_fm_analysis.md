# Analysis: Adding FM Synthesis to Plinky

## Summary

This document provides a technical overview and UI strategy for adding 2-op Frequency Modulation (FM) synthesis to the Plinky synthesizer, balancing new sonic capabilities with the constraints of Plinky’s hardware, firmware, and front panel.

---

## 1. FM Synthesis Overview

**FM synthesis** creates complex sounds by modulating the frequency of one oscillator (the "carrier") with another ("modulator"). Even a simple 2-operator architecture (one carrier, one modulator per voice) enables metallic, bell-like, and evolving digital timbres.

- **Classic FM**: Both oscillators are usually sine waves, but other waveforms yield richer harmonics.
- **FM Parameters**: FM index (depth), FM ratio (modulator/carrier frequency), and oscillator waveforms.

---

## 2. Plinky Architecture: Opportunities & Constraints

### Opportunities
- **Polyphony**: Plinky supports 8 voices, each can have its own FM pair (one panned left, the other right)
- **Flexible firmware**: Existing oscillator routines can be extended.
- **Touch surface and modulation sources**: Can modulate FM parameters for expressive control.

### Constraints
- **Front panel space**: Nearly all pads/params are in use, except the 5th row (outside Sampler mode) - we should switch the sampler params to be FM params when in FM mode.
- **Firmware/UI complexity**: Must avoid confusing parameter overload.

---

## 3. FM Implementation Plan

### A. **Oscillator & DSP Changes**
- Add a modulator oscillator to each voice in firmware.
- Implement per-voice FM: modulator output modifies carrier frequency.
- Use sine wave as default; allow waveform selection also via param pad.

### B. **Parameter Mapping & UI**
- **FM Mode Selection**: Cycle oscillator type pad through Saw, Granular, FM. 
- To activate FM we could modify the existing Sound (Synthesizer) - Shape param (from plinky manual)
  - _hold_  _tap_ Controls the shape of the oscillators in Plinky. When exactly 0%, you get 4 sawtooths per voice. When positive, you blend smoothly through 16 ROM wavetable shapes, (2 per voice,) provided by @miunau. When negative, you get PWM control of pulse/square shapes, (also 2 per voice.)
  - Proposed changes: We could add FM mode to activate when the shape param is set to -100 only.
- When FM mode is selected:
  - Repurpose the 5th row pads for FM controls (except in Sampler mode).
  - Example 5th row mapping:
    - Pad 1 - top shift: Carrier waveform select
    - Pad 1 - bottom shift: Modulator waveform select
    - Pad 2 - top shift: FM index (depth)
    - Pad 2 - bottom shift: FM ratio
    - Pad 3 - top shift: FM envelope amount
    - Other row 5 param pads: Reserved for future FM features (mod source, randomize, etc.)

- **Waveform Selection**: Cycles through available waveforms (Sine, Saw, Square, Triangle), can be done with encoder or slider too and blends between each waveform type as user goes through the options.
- **Parameter Modulation**: Allow existing LFO/CV sources to modulate FM index or ratio.
- **Contextual UI**: When not in FM mode, 5th row pads retain their current functions.

### C. **Firmware Logic**
- On FM mode entry, remap 5th row pads to FM parameters.
- Save FM settings in presets.
- Ensure DSP routines support switching between oscillator modes without glitching.

---

## 4. Compatibility & User Experience

- **Backwards compatibility**: Saw, Granular and PWM modes unchanged.
- **No hardware changes needed**: All logic/UI changes are firmware-based.
- **Manual update**: Clearly document FM mode and pad mapping.

---

## 5. Risks & Testing

- **CPU load**: FM synthesis is efficient, but test polyphony for glitches.
- **UI clarity**: Contextual pad behavior must be visually/feedback clear.
- **Preset migration**: Ensure old presets load safely and new FM params are robust.

---

## 6. Visualization & User Feedback

### OLED Display
- **Dual waveform display**: Show carrier and modulator waveforms side-by-side
- **Real-time FM visualization**: Display how modulation depth affects the carrier waveform
- **Waveform morphing feedback**: Visual indication of current waveform blend position between Sine/Saw/Square/Triangle
- **Parameter values**: Show current FM index, ratio, and envelope amount numerically

### LED Feedback
- **Row 5 pad LEDs**: Light up to indicate which FM parameters are active/being edited
- **Brightness mapping**: LED intensity reflects parameter values (FM index, ratio)
- **Shift state indication**: Different LED patterns for top/bottom shift states
- **FM mode indicator**: Distinct LED pattern when P_SHAPE = -100% (FM mode active)

### Benefits
- Makes abstract FM concepts visually tangible
- Provides immediate feedback during parameter adjustment  
- Educational value for understanding FM synthesis
- Consistent with Plinky's existing visual interface patterns

---

## 7. Summary Table: FM Controls in FM Mode

| Pad | Top Shift | Bottom Shift |
|-----|-----------|--------------|
| 1   | Carrier waveform | Modulator waveform |
| 2   | FM Index (Depth) | FM Ratio |
| 3   | FM Envelope Amount | - |
| 4-8 | Reserved/Future use | Reserved/Future use |

---

## 8. Next Steps

1. Prototype per-voice FM in firmware.
2. Implement contextual UI pad mapping for FM mode.
3. Test for polyphony, performance, and UX clarity.
4. Update manual and presets handling.

---

**References**  
- [Plinky Manual](https://plinkysynth.com/docs/manual)
- [Plinky Public GitHub](https://github.com/plinkysynth/plinky_public)
