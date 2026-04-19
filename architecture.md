# ARCHITECTURE.md: SNES9x 3DS Stereoscopic 3D

## 1. Core Objective
Transform the 2D SNES output into a 3D "Diorama" by separating Background Layers (BG0-3) and Sprites into distinct Z-planes with horizontal parallax offsets when the 3DS 3D slider is active.

## 2. Technical Stack
- **Graphics API:** `citro3d` (PICA200 GPU)
- **Shaders:** PICA200 Assembly (`.v.pica`)
- **Key Logic:** Hybrid CPU Z-shifting + GPU Horizontal Parallax.

## 3. The 3D Pipeline (Hybrid Design)
### CPU Responsibility (`source/Snes9x/gfxhw.cpp`)
- **Layer Separation:** Assign unique Z-depths to each SNES layer to prevent Z-fighting and establish depth order.
- **Uniform Updates:** Pass the Inter-Ocular Distance (IOD) from the hardware slider to the GPU via a new shader uniform: `stereoIOD`.

### GPU Responsibility (`shader_tiles.v.pica`)
- **Parallax Shift:** The vertex shader reads the `stereoIOD` uniform and the vertex's Z-depth to apply a horizontal offset ($X \pm \Delta$).

---

## 4. Key Code Landmarks

### Hardware Interface
- **Slider Reading:** `gpu3dsGetIOD()` in `source/3dsgpu.cpp`. 
  - Uses `osGet3DSliderState()` to return a value (0.0 to 1.0) scaled by `IOD_MAX_PIXELS`.

### Rendering Engine
- **Main Entry:** `gpu3dsDrawSnesScreen()` in `source/3dsimpl_gpu.cpp`.
- **Draw Calls:** `gpu3dsDrawLayers()` and `gpu3dsDrawTiledLayer()`.
- **Actual Hardware Call:** `C3D_DrawArrays` / `C3D_DrawElements` inside `gpu3dsDraw()` (`source/3dsgpu.cpp`).

### Shader Core
- **Primary Shader:** `shader_tiles.v.pica` (Handles BG0-3 and OBJs).
- **Secondary Shader:** `shader_mode7.v.pica` (Handles Mode 7).
- **Uniform Slots:** Currently using #0-5. **`stereoIOD` will be assigned to slot #6.**

---

## 5. Implementation Status
- [x] **Discovery:** Located hardware registers and draw loops.
- [x] **Architecture:** Selected "Option C: Hybrid" for performance.
- [ ] **Phase 1:** Uniform and Shader preparation (In Progress).
- [ ] **Phase 2:** Dual-pass rendering and Z-offset logic.