# CLAUDE.md — Project Context for Claude Code

## What This Project Is
A Flipper Zero app that identifies and decodes weather station RF protocols.
It loads a full protocol library from SD card (converted from rtl_433 corpus),
matches incoming packets, and for unknown protocols, runs on-device pattern
classification to figure out the protocol structure.

No cloud/API component. Everything runs on-device.

## Hardware
- Flipper Zero with STM32WB55RG (Cortex-M4 @ 64MHz, 256KB RAM, 1MB flash)
- CC1101 sub-GHz transceiver (433/868/915MHz)
- MicroSD card for protocol library + capture storage
- Test target: Ambient Weather WS-2902 (Fine Offset family, 915MHz OOK)

## Key Constraints
- 256KB total RAM, ~100KB used by Flipper OS
- CC1101 register values MUST come from docs/cc1101/cc1101_registers.md
  Do NOT generate register values from memory — they will be wrong
- SubGHz protocol structure MUST follow patterns in docs/flipper_sdk/
- SD card storage is effectively unlimited

## Architecture
See ARCHITECTURE.md for full design. Summary of layers:
1. RF capture via CC1101 (433/868/915MHz scanning)
2. Protocol corpus match (200+ rtl_433 definitions loaded from SD)
3. On-device pattern classifier for unknown protocols
4. All captures logged to SD for historical data

## Reference Files (read these before writing code)
- docs/cc1101/cc1101_registers.md — CC1101 register map (AUTHORITATIVE)
- docs/flipper_sdk/ — Flipper API headers and working protocol examples
- docs/rtl433_reference/ — rtl_433 decoder source (ground truth)
- docs/sample_captures/ — real WS-2902 packet captures

## Build System
External Flipper app built with fbt.
App lives in flipper_app/ directory.
See docs/flipper_sdk/example_app.fam for manifest format.

## Ground Truth Validation
WS-2902 packet structure (Fine Offset family):
- 915MHz, OOK PWM, sync word 0x2DD4
- 80 bits: ID(8) + flags(4) + temp(12) + humidity(8) + wind_speed(8)
  + wind_gust(8) + wind_dir(9) + rain(16) + CRC8(8)
- Temp: (raw - 400) × 0.1 °C
- Wind: raw × 0.34 m/s
- Rain: raw × 0.1 mm
