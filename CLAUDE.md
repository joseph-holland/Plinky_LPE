# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Plinky LPE (Lucky Phoenix Edition) is embedded firmware for the Plinky/Plinky+ hardware synthesizer, built on STM32L476 ARM Cortex-M4. This is a real-time audio synthesizer with touchscreen interface, running at ~48kHz sample rate with strict timing constraints.

## Build Commands

### Primary Build (Command Line)
```bash
cd sw/nocube_makefile
make clean
make -j BUILD_TYPE=RELEASE  # or DEBUG
python nocube_binmaker.py   # Creates .uf2 files for bootloader
```

### Quick Release Build
```bash
./builds/build_release_nocube_and_then_run_binmaker_for_release.sh
```

### Alternative Build (STM32CubeIDE)
- Import projects from `sw/` and `bootloader/` directories
- Build configurations: Debug, Release
- Used for debugging with integrated tools

## Architecture

### Core Structure
- **`sw/Core/Src/plinky/`** - Main firmware organized by function:
  - `hardware/` - HAL for ADC/DAC, codec, MIDI, SPI, touchstrips
  - `synth/` - Audio engine (arpeggiator, LFOs, sequencer, strings)
  - `ui/` - User interface (LEDs, OLED, pad actions)
  - `gfx/` - Graphics rendering system
  - `usb/` - USB MIDI and web editor functionality
  - `data/` - Lookup tables, fonts, icons

### Key Design Patterns
- **Hardware Abstraction**: Clean separation between hardware drivers and application logic
- **Real-time Audio**: Tick-based processing with sample-accurate timing
- **Modular Voice Management**: Strings module handles polyphony and voice allocation
- **Centralized Timing**: Time module coordinates arpeggiator, sequencer, and LFO sync
- **Parameter System**: Comprehensive parameter management with preset storage

### Critical Components
- **Audio Pipeline**: Real-time synthesis at 48kHz with multiple oscillators, filters, effects
- **Sequencer**: Pattern-based step sequencer with swing timing and external sync
- **Hardware Interface**: 8 pressure-sensitive pads, rotary encoder, OLED display, LED arrays
- **USB Subsystem**: MIDI device + web-based editor using TinyUSB

## Development Considerations

### Real-time Constraints
- Audio callback runs at ~48kHz - no blocking operations allowed
- Sample-accurate sequencing requires precise timing calculations
- External clock sync (MIDI, CV) must maintain sub-millisecond accuracy

### Memory Management
- Limited STM32L4 RAM/Flash requires careful allocation
- Audio buffers pre-allocated at startup
- Preset/sample storage uses external flash

### Hardware Variants
- Runtime detection of Plinky vs Plinky+ hardware
- Hardware-specific features gated by detection
- Version-specific I/O configurations in hardware/ modules

### Build System Notes
- `nocube_makefile/` provides command-line builds outside STM32CubeIDE
- Python scripts handle binary post-processing for bootloader compatibility
- Desktop emulator in `emu/` for UI development without hardware

## Testing
Run desktop emulator for UI/logic testing:
```bash
cd sw/emu
# Build and run with your C++ toolchain (requires ImGui)
```

Hardware testing requires flashing to actual Plinky device via USB bootloader (.uf2 files) or ST-Link debugger.

## Manual

More details on use of Plinky can be found in the manual here: plinky_manual.md
Use this to understand the full UI/UX and how a user actually uses Plinky.