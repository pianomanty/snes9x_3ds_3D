# SNES9x 3DS: Stereoscopic 3D Implementation Plan
**Selected Strategy:** Option C (Hybrid CPU/GPU Parallax)

## 🏛️ Project Context & Findings
* **Hardware Interface:** 3D slider is read via `osGet3DSliderState()` in `source/3dsgpu.cpp`.
* **Rendering Pipeline:** SNES layers are built into vertices in `source/Snes9x/gfxhw.cpp` and drawn via `citro3d` in `source/3dsimpl_gpu.cpp`.
* **GPU Architecture:** Uses PICA200 vertex shaders (`.v.pica`).
* **Hybrid Logic:** 1. **CPU** handles "Coarse" Z-plane separation for layer priority.
    2. **GPU** handles "Fine" horizontal parallax shifting based on those Z-planes.

---

## 🛠️ Implementation Checklist

### Phase 1: GPU Shader Preparations
- [x] **Define Uniform:** Added `ULOC_STEREO_IOD` to the `SGPU_SHADER_ULOC` enum in [`source/3dsgpu.h`](source/3dsgpu.h:82).
- [x] **Initialize Uniform:** Updated `gpu3dsInitializeShaderUniformLocations()` in [`source/3dsgpu.cpp`](source/3dsgpu.cpp:751) to link the "stereoIOD" string.
- [x] **Modify `shader_tiles.v.pica`:**
      - Added `.fvec stereoIOD` uniform at slot #6.
      - Inserted assembly logic in the `draw` section to apply horizontal offset based on vertex depth (`r0.z`).
- [x] **Modify `shader_mode7.v.pica`:**
      - Added `.fvec stereoIOD` uniform at slot #7.
      - Inserted assembly logic in the `depth_lt_16384` section to apply horizontal offset based on vertex depth.

### Phase 2: CPU Integration & Data Flow
- [ ] **Create Helper:** Add `S9xApplyStereoZOffset()` to `source/Snes9x/gfxhw.cpp` to calculate depth shifts from the slider.
- [ ] **Update Uniforms:** Modify the rendering loop to call `shaderInstanceSetUniformFloat` with the current `iod` value before draw calls.
- [ ] **Layer Interception:** - Update `S9xDrawBackgroundHardwarePriority0Inline()` to apply Z-offsets.
    - Update `S9xDrawBackgroundMode7Hardware()` to apply Z-offsets.

### Phase 3: Validation & Testing
- [ ] **Build Check:** Run `make` to ensure shader assembly and C++ compilation pass.
- [ ] **2D Parity Test:** Verify that with the 3D slider at 0, the output is pixel-identical to the original version.
- [ ] **3D Depth Test:** Test on hardware/Citra to verify BG layers separate without "breaking" the image.

---

## 📝 Technical Notes
* **Coordinate Space:** The SNES PPU uses a 256x224 coordinate system. Ensure offsets are scaled correctly for the 3DS top screen (400x240).
* **Z-Buffer:** The 3DS uses a 24-bit depth buffer. The hybrid approach must ensure Z-planes don't overlap in a way that causes flickering (Z-fighting).
* **Dual Pass:** Ensure `gpu3dsDrawSnesScreen()` is updated to render for both left and right eye targets when 3D is enabled.