---
name: snes-3ds-platform
description: Technical specs for 3DS PICA200 GPU and SNES PPU layer mapping.
---
# 3DS & SNES Technical Knowledge
When architecting 3D features for this emulator:

## PICA200 GPU Constraints
- The 3DS uses a **Fixed Function Pipeline** with limited programmable vertex shaders.
- Stereoscopic 3D requires rendering the scene twice: once for the Left Eye and once for the Right Eye.
- Use `gfxSet3D(true)` to enable the 3D slider logic.

## SNES PPU Layer Mapping
- BG0-BG3 are the background layers.
- In 2D mode, these are flattened. In 3D mode, we must assign a `Z-depth` to each.
- Priority: SNES sprites (OAM) usually sit "between" BG layers. The architecture must account for Sprite-to-BG priority bitmasking.

## Development Environment
- This project uses **devkitPro** and **libctru**.
- Header files are in `include/` and `source/`.
- Use `iprintf` or the 3DS console for debugging, as standard `stdout` is not visible.