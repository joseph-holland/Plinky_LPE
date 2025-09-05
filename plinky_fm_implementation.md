# Plinky FM Synthesis Implementation Guide

## Executive Summary

This document provides a detailed technical implementation plan for adding 2-operator FM synthesis to Plinky. The implementation maintains full stereo output with 2 carrier oscillators per voice, proper phase accumulation, and seamless integration with the existing audio pipeline.

---

## 1. Architecture Overview

### 1.1 FM Synthesis Design
- **2-op FM per voice**: 2 carriers (stereo) + 1 shared modulator
- **8 voices polyphony**: Maintains existing voice count
- **Integration point**: When `P_SHAPE = -512` (exactly -100%)
- **Audio pipeline position**: Replaces oscillator generation in `apply_subtractive_lpg_noise()`

### 1.2 Oscillator Allocation
```
Voice oscillators (4 per voice):
- osc[0]: LEFT carrier
- osc[1]: RIGHT carrier  
- osc[2]: Modulator (shared for both carriers)
- osc[3]: Reserved (future: second modulator or feedback)
```

---

## 2. Critical Code Modifications

### 2.1 Detection Function (`sw/Core/Src/plinky/synth/synth.c`)

Add after `using_sampler()` function:
```c
// Line ~50, after sampler includes
static inline bool using_fm(void) {
    // FM mode active when P_SHAPE is exactly -512 (-100%)
    return cur_preset.params[P_SHAPE][0] == -512;
}
```

### 2.2 FM Synthesis Function (`sw/Core/Src/plinky/synth/synth.c`)

Add new function before `apply_subtractive_lpg_noise()`:
```c
// Line ~204, before apply_subtractive_lpg_noise
static void apply_fm_lpg_noise(u8 voice_id, Voice* voice, float goal_lpg, 
                                float noise_diff, float drive, float resonance, u32* dst) {
    float glide = lpf_k(param_val_poly(P_GLIDE, voice_id) >> 2) * (0.5f / SAMPLES_PER_TICK);
    
    // FM parameters (using sampler params when in FM mode)
    s32 fm_index_raw = param_val_poly(P_SCRUB, voice_id);           // Pad 1 top
    s32 fm_ratio_raw = param_val_poly(P_SCRUB_JIT, voice_id);       // Pad 1 bottom
    s32 carrier_wave = param_val_poly(P_GR_SIZE, voice_id);         // Pad 2 top
    s32 mod_wave = param_val_poly(P_GR_SIZE_JIT, voice_id);         // Pad 2 bottom
    s32 fm_env_amt = param_val_poly(P_PLAY_SPD, voice_id);          // Pad 3 top
    
    // Convert parameters to usable values
    float fm_index = fm_index_raw * (8.0f / 65536.0f);  // 0-8 range
    float fm_ratio = 0.5f + (fm_ratio_raw * (8.0f / 65536.0f)); // 0.5-8.5 range
    float fm_env_scale = fm_env_amt * (1.0f / 65536.0f);
    
    // Get modulator oscillator
    Osc* modulator = &voice->osc[2];
    
    // Calculate modulator frequency (carrier frequency * ratio)
    s32 base_pitch = (voice->osc[0].pitch + voice->osc[1].pitch) / 2;
    s32 mod_pitch = base_pitch + (s32)(log2f(fm_ratio) * PITCH_PER_OCTAVE);
    modulator->goal_phase_diff = maxi(65536, (s32)(table_interp(pitches, mod_pitch + PITCH_BASE) * (65536.f * 128.f)));
    
    // Glide for modulator
    int dd_mod = (int)((modulator->goal_phase_diff - modulator->phase_diff) * glide);
    
    // Process stereo carriers
    for (u8 carrier_id = 0; carrier_id < 2; carrier_id++) {
        s16* osc_dst = ((s16*)dst) + carrier_id;  // LEFT (0) or RIGHT (1)
        
        Osc* carrier = &voice->osc[carrier_id];
        
        // Glide for carrier
        int dd_carrier = (int)((carrier->goal_phase_diff - carrier->phase_diff) * glide);
        
        // Get smoother state for this channel
        float y1 = voice->lpg_smoother[carrier_id].y1;
        float y2 = voice->lpg_smoother[carrier_id].y2;
        
        // Envelope tracking
        float osc_lpg = voice->env1_lvl;
        float osc_lpg_diff = (goal_lpg - osc_lpg) * (1.f / SAMPLES_PER_TICK);
        
        // Noise setup
        float noise = voice->noise_lvl;
        int rand_table_pos = rand() & 16383;
        
        // Local phase accumulators
        u32 mod_phase = modulator->phase;
        s32 mod_phase_diff = modulator->phase_diff;
        u32 carrier_phase = carrier->phase;
        s32 carrier_phase_diff = carrier->phase_diff;
        
        // FM synthesis loop - generate SAMPLES_PER_TICK samples
        for (u8 i = 0; i < SAMPLES_PER_TICK; ++i) {
            // Update modulator phase
            mod_phase_diff += dd_mod;
            mod_phase += mod_phase_diff;
            
            // Generate modulator signal (sine wave for now)
            s32 mod_signal = (s32)((mod_phase >> 16) - 32768);  // Convert to signed
            
            // Apply FM with envelope scaling
            float fm_depth = fm_index * (1.0f + fm_env_scale * voice->env1_norm);
            s32 freq_mod = (s32)(mod_signal * fm_depth);
            
            // Update carrier with modulation
            carrier_phase_diff += dd_carrier;
            s32 modulated_phase_diff = carrier_phase_diff + (freq_mod << 8);
            carrier_phase += modulated_phase_diff;
            
            // Generate carrier output (saw wave for now, can be selected later)
            s32 out = (s32)(carrier_phase >> 4);
            
            // Apply noise
            s16 n = ((s16*)rndtab)[rand_table_pos++];
            noise += noise_diff;
            
            // Apply filter and envelope
            osc_lpg += osc_lpg_diff;
            y1 += (out * drive + n * noise - (y2 - y1) * resonance - y1) * osc_lpg;
            y1 *= 0.999f;
            y2 += (y1 - y2) * osc_lpg;
            y2 *= 0.999f;
            
            // Write to output buffer (maintains stereo)
            s32 smooth_lpg = FLOAT2FIXED(y2, 0);
            *osc_dst = SATURATE16(*osc_dst + smooth_lpg);
            osc_dst += 2;  // Skip to next sample of same channel
        }
        
        // Save state back
        carrier->phase = carrier_phase;
        carrier->phase_diff = carrier_phase_diff;
        voice->lpg_smoother[carrier_id].y1 = y1;
        voice->lpg_smoother[carrier_id].y2 = y2;
    }
    
    // Save shared modulator state
    modulator->phase_diff += dd_mod * SAMPLES_PER_TICK;
    voice->env1_lvl = goal_lpg;
    voice->noise_lvl = noise;
}
```

### 2.3 Integration Point (`sw/Core/Src/plinky/synth/synth.c`)

Modify `run_voice()` function around line 402:
```c
// Line ~402 in run_voice()
// apply low pass gate and noise
if (using_sampler())
    apply_sample_lpg_noise(voice_id, voice, goal_lpg, noise_diff, drive, dst);
else if (using_fm())
    apply_fm_lpg_noise(voice_id, voice, goal_lpg, noise_diff, drive, resonance, dst);
else
    apply_subtractive_lpg_noise(voice_id, voice, goal_lpg, noise_diff, drive, resonance, dst);
```

### 2.4 Oscillator Initialization (`sw/Core/Src/plinky/synth/synth.c`)

Modify `generate_oscs()` to handle FM mode around line 90:
```c
// In generate_oscs(), after line 89 (before oscillator loop)
if (using_fm()) {
    // For FM mode, only set up carrier pitches
    // Modulator pitch will be calculated in apply_fm_lpg_noise based on ratio
    for (u8 osc_id = 0; osc_id < 2; ++osc_id) {  // Only carriers (0,1)
        // ... existing pitch calculation code ...
        voice->osc[osc_id].pitch = osc_pitch;
        voice->osc[osc_id].goal_phase_diff = 
            maxi(65536, (s32)(table_interp(pitches, osc_pitch + PITCH_BASE) * (65536.f * 128.f)));
    }
    return;  // Skip the normal 4-oscillator setup
}
// ... existing oscillator loop for non-FM modes ...
```

---

## 3. Parameter System Integration

### 3.1 Parameter Mapping (Active when `using_fm()` returns true)

| Pad | Shift | Parameter | FM Function | Range |
|-----|-------|-----------|-------------|--------|
| 1 | Top | P_SCRUB | FM Index | 0-8 |
| 1 | Bottom | P_SCRUB_JIT | FM Ratio | 0.5-8.5 |
| 2 | Top | P_GR_SIZE | Carrier Waveform | 0-3 (sin/saw/sqr/tri) |
| 2 | Bottom | P_GR_SIZE_JIT | Modulator Waveform | 0-3 (sin/saw/sqr/tri) |
| 3 | Top | P_PLAY_SPD | FM Envelope Amount | 0-100% |
| 3 | Bottom | P_PLAY_SPD_JIT | FM Feedback (future) | 0-100% |

### 3.2 UI Modifications (`sw/Core/Src/plinky/ui/pad_actions.c`)

Add FM parameter handling in pad action functions:
```c
// In handle_param_pad() or equivalent
if (using_fm() && shift_state == SS_SHIFT_A) {
    // Remap row 5 pads to FM parameters
    switch (pad_id) {
        case 32: return P_SCRUB;      // FM Index
        case 33: return P_GR_SIZE;     // Carrier Wave
        case 34: return P_PLAY_SPD;    // FM Env Amount
        // etc...
    }
}
```

---

## 4. Waveform Selection

### 4.1 Waveform Generation Functions

Add waveform selection for both carrier and modulator:
```c
static inline s32 generate_waveform(u32 phase, u8 waveform_type) {
    switch (waveform_type) {
        case 0: // Sine
            return (s32)sine_table[(phase >> 22) & 1023] << 8;
        case 1: // Saw
            return (s32)(phase >> 4);
        case 2: // Square
            return (phase & 0x80000000) ? 32767 : -32768;
        case 3: // Triangle
            return (s32)(abs((s32)(phase >> 15) - 32768) - 16384) << 1;
        default:
            return 0;
    }
}
```

---

## 5. OLED Visualization

### 5.1 FM Display (`sw/Core/Src/plinky/ui/oled_viz.c`)

Add FM visualization when in FM mode:
```c
void draw_fm_visualization(void) {
    if (!using_fm()) return;
    
    // Display dual waveforms (carrier + modulator)
    // Show FM index and ratio values
    // Display modulation depth visually
    
    gfx_draw_str_small(0, 0, "FM");
    // Draw carrier waveform on left
    // Draw modulator waveform on right
    // Show index/ratio values below
}
```

---

## 6. Testing & Verification

### 6.1 Critical Test Points

1. **Stereo Output Verification**
   - Confirm LEFT carrier on channel 0
   - Confirm RIGHT carrier on channel 1
   - Test with headphones for proper stereo separation

2. **Phase Accumulation**
   - Verify carrier and modulator frequencies are correct
   - Check for aliasing at high frequencies
   - Test glide/portamento with FM

3. **Parameter Ranges**
   - FM Index: Smooth transition from 0 (no modulation) to 8 (heavy modulation)
   - FM Ratio: Musical intervals (0.5, 1, 2, 3, 4, etc.)
   - Envelope modulation of FM index

4. **Integration Tests**
   - Switching between FM and other synthesis modes
   - Preset save/load with FM parameters
   - MIDI control of FM voices

### 6.2 Debug Output Points

Add debug logging at key points:
```c
#ifdef DEBUG_FM
    printf("FM: idx=%.2f ratio=%.2f carrier_f=%d mod_f=%d\n", 
           fm_index, fm_ratio, carrier->phase_diff, modulator->phase_diff);
#endif
```

---

## 7. Optimization Opportunities

### 7.1 Performance Optimizations
- Pre-calculate FM depth per tick instead of per sample
- Use lookup tables for sine modulator
- Consider fixed-point math for FM calculations

### 7.2 Future Enhancements
- Multiple FM algorithms (parallel, series, feedback)
- Per-operator envelopes
- Operator sync/reset options
- FM-specific LFO destinations

---

## 8. Risk Mitigation

### 8.1 Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Mono output | Single channel write | Ensure both carriers write to separate channels |
| No output | Phase not advancing | Verify phase accumulation for all oscillators |
| Aliasing | High mod frequencies | Implement band-limiting or oversampling |
| CPU overload | Complex calculations | Optimize inner loop, use fixed-point |

### 8.2 Fallback Strategy
If FM synthesis causes issues, the implementation can be disabled by checking for a flag:
```c
#define ENABLE_FM_SYNTH 1  // Set to 0 to disable
```

---

## 9. Implementation Sequence

1. **Phase 1**: Core FM synthesis
   - Implement `using_fm()` detection
   - Add basic `apply_fm_lpg_noise()` with sine waves
   - Verify stereo output works correctly

2. **Phase 2**: Parameter integration
   - Map FM parameters to UI pads
   - Add parameter value scaling
   - Test parameter ranges

3. **Phase 3**: Waveform selection
   - Implement waveform generation functions
   - Add waveform selection parameters
   - Test different carrier/modulator combinations

4. **Phase 4**: Polish & optimization
   - Add OLED visualization
   - Optimize performance
   - Fine-tune parameter ranges

---

## 10. Code Safety Checklist

- [ ] Bounds checking on all array accesses
- [ ] Phase overflow handling (32-bit wraparound)
- [ ] Parameter value clamping
- [ ] Null pointer checks for voice and dst buffers
- [ ] Smooth parameter transitions to avoid clicks
- [ ] Proper state initialization on mode switch

---

## References

- Plinky audio pipeline: `sw/Core/Src/plinky/synth/synth.c`
- Parameter system: `sw/Core/Src/plinky/synth/param_defs.h`
- Audio constants: `sw/Core/Src/plinky/defs.h`
- FM theory: Yamaha DX7 implementation papers