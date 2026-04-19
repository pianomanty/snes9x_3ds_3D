# SNES9x 3DS: Stereoscopic 3D Implementation Plan

## Analysis Summary

### Can We Intercept Individual SNES Background Layers?

**Yes, but not at the draw call level - we must intercept at the vertex building level.**

The rendering pipeline has three distinct phases:

1. **Vertex Building Phase** (`source/Snes9x/gfxhw.cpp`)
    - Called during SNES PPU rendering loop
    - Functions like `S9xDrawBackgroundHardwarePriority0Inline()` build vertex data
    - This is where we can intercept and apply Z offsets

2. **Commit Phase** (`source/Snes9x/gfxhw.cpp:523`)
    - `S9xCommitLayerSection()` stores render state after building vertices
    - Data is stored in `GPU3DSExt.layerList` for later drawing

3. **Draw Phase** (`source/3dsimpl_gpu.cpp:308`)
    - `gpu3dsDrawSnesScreen()` -> `gpu3dsDrawLayers()` -> `gpu3dsDrawTiledLayer()`
    - Vertex data is sent to GPU via `C3D_DrawArrays()`/`C3D_DrawElements()`
    - **Too late to modify Z here** - vertices are already built

---

## Graphics Stack Analysis

### Libraries Used

Based on the Makefile and code analysis:

| Library | Purpose | Location |
|---------|---------|----------|
| **citro3d** (custom v1.7.1) | 3D graphics API (PICA200 GPU) | `libs/citro3d/` (custom fork) |
| **libctru** | 3DS system API (hardware access) | devkitPro standard |
| **libpng** | PNG image loading | devkitPro standard |
| **libz** | Compression | devkitPro standard |
| **libm** | Math functions | devkitPro standard |

**Key Finding:** The project uses a **custom fork of citro3d** (v1.7.1) from devkitPro, with a patch applied (`citro3d-uniforms-maxdirty.patch`). This is the primary 3D graphics library.

### Shader Pipeline

The project uses **PICA200 GPU shaders** for rendering:

| Shader | Purpose | Type |
|--------|---------|------|
| `shader_tiles.v.pica` | Main SNES background layers (BG0-BG3, OBJ) | Vertex Shader |
| `shader_mode7.v.pica` | Mode 7 background layers | Vertex Shader |
| `shader_screen.v.pica` | UI/Overlay rendering | Fragment Shader |

**Important Finding:** The main display quad **DOES use a vertex shader** (`shader_tiles.v.pica`). This means we can potentially move parallax logic into the GPU shader instead of the CPU.

---

## Shader Analysis

### `shader_tiles.v.pica` (Main Background Layers)

**Uniforms Available:**
- `projection[4]` (#0-3) - Projection matrix
- `textureScale` (#4) - Texture scaling
- `textureOffset` (#5) - Texture offset (0.0 or 1.0 for mode5/6 hi-res flipping)

**Current Behavior:**
- The shader handles texture flipping, alpha computation, and depth computation
- **No parallax/stereo offset logic currently exists**
- The `textureOffset` uniform is only used for mode5/6 hi-res flipping

**Parallax Implementation Opportunity:**
We can add a new uniform to apply horizontal parallax offset based on depth values:

```pica
; New uniform for stereo parallax
.fvec       stereoOffset         ; #6 - (iod * 256, 0, 0, 0)

; In the shader, apply offset to x coordinate based on depth:
; outpos.x = inpos.x + (depth > 0 ? stereoOffset.x : -stereoOffset.x)
```

### `shader_mode7.v.pica` (Mode 7 Background)

**Uniforms Available:**
- `projection[4]` (#0-3) - Projection matrix
- `textureScale` (#4) - Texture scaling
- `textureOffset` (#5) - Not used here
- `updateFrame` (#6) - Frame counter for animation

**Parallax Implementation Opportunity:**
Similar to `shader_tiles.v.pica`, we can add a stereo offset uniform.

### `shader_screen.v.pica` (UI Overlay)

**Uniforms Available:**
- `projection[4]` (#0-3) - Projection matrix
- `textureScale` (#4) - Texture scaling

**Note:** This is a fragment shader for UI rendering, not used for SNES background layers.

---

## Recommended Implementation Strategy

### Option A: GPU Shader-Based Parallax (RECOMMENDED)

**Location:** Modify `shader_tiles.v.pica` and `shader_mode7.v.pica`

**Advantages:**
- Maximum performance (GPU handles parallax, not CPU)
- No CPU overhead for per-vertex calculations
- Cleaner separation of concerns
- More efficient for large numbers of vertices

**Implementation:**

1. **Add new uniform to vertex shaders:**

```pica
; In shader_tiles.v.pica
.fvec       stereoIOD            ; #6 - Inter-ocular distance in pixels

; In shader_mode7.v.pica
.fvec       stereoIOD            ; #6 - Inter-ocular distance in pixels
```

2. **Apply parallax in shader:**

```pica
; In shader_tiles.v.pica, after computing x position:
; outpos.x = inpos.x + (depth > 0 ? -stereoIOD.x : +stereoIOD.x)
; Note: Lower Z = closer to viewer, so negative offset for foreground

; Example implementation:
cmp     r0.z, lt, lt, r2.y           ; if depth < 16384 (foreground)
jmpc    cmp.x, apply_negative_offset
; Background: add positive offset
mad     outpos.x, r0.x, r1.x, r0.x   ; outpos.x = inpos.x + stereoIOD
jmp     done_offset
apply_negative_offset:
; Foreground: subtract offset
mad     outpos.x, r0.x, -r1.x, r0.x  ; outpos.x = inpos.x - stereoIOD
done_offset:
```

3. **Update shader uniforms from CPU:**

```cpp
// In source/Snes9x/gfxhw.cpp, before drawing:
if (gpu3dsIs3DEnabled()) {
    float iod = gpu3dsGetIOD();
    if (iod > 0.0f) {
        // Set stereoIOD uniform for vertex shaders
        float stereoIOD[4] = {iod, 0.0f, 0.0f, 0.0f};
        shaderInstanceSetUniformFloat(GPU3DS.shaders[SPROGRAM_TILES].shaderProgram.vertexShader, 
                                       GPU3DS.shaderULocs[ULOC_STEREO_IOD], stereoIOD, 4);
        shaderInstanceSetUniformFloat(GPU3DS.shaders[SPROGRAM_MODE7].shaderProgram.vertexShader,
                                       GPU3DS.shaderULocs[ULOC_STEREO_IOD], stereoIOD, 4);
    }
}
```

**Pros:**
- Maximum performance (GPU handles parallax)
- No CPU overhead
- Cleaner code separation
- More scalable

**Cons:**
- Requires shader modification
- More complex shader logic
- Debugging may be harder

---

### Option B: CPU-Based Parallax (Original Approach)

**Location:** `source/Snes9x/gfxhw.cpp`

Modify the background drawing functions to apply Z offsets based on the 3D slider value:

```cpp
static inline void S9xApplyStereoZOffset(int* depth0, int* depth1) {
    if (!gpu3dsIs3DEnabled()) return;
    
    float iod = gpu3dsGetIOD();
    if (iod <= 0.0f) return;
    
    // Convert pixel offset to depth units (256 = 1 pixel)
    int offset = (int)(iod * 256);
    
    // Apply offset: depth0 gets negative (closer), depth1 gets positive (farther)
    *depth0 -= offset;
    *depth1 += offset;
}
```

**Pros:**
- Simpler to implement
- No shader modification needed
- Easier to debug

**Cons:**
- CPU overhead for per-vertex calculations
- Less efficient for large numbers of vertices
- May impact performance on slower hardware

---

### Option C: Hybrid Approach (RECOMMENDED FOR BALANCE)

**Location:** Both CPU and GPU

1. **Apply coarse Z offset in CPU** (for layer separation)
2. **Apply fine parallax in GPU shader** (for smooth depth effect)

**Implementation:**

1. **CPU Layer (gfxhw.cpp):**
   - Apply basic Z offset for layer separation (as in Option B)
   - This ensures proper depth ordering between layers

2. **GPU Layer (shader_tiles.v.pica):**
   - Apply additional parallax based on depth value
   - Use shader uniform for smooth, GPU-accelerated offset

**Pros:**
- Best of both worlds
- CPU handles layer separation, GPU handles fine parallax
- Good performance with clean separation
- Easier to tune and debug

**Cons:**
- More complex implementation
- Requires both CPU and GPU changes

---

## Implementation Recommendation

**Recommended Approach: Option C (Hybrid)**

This approach provides the best balance of performance, maintainability, and flexibility:

1. **CPU Layer (gfxhw.cpp):**
   - Apply basic Z offset for layer separation
   - Ensures proper depth ordering between BG0-BG3 layers

2. **GPU Layer (shader_tiles.v.pica, shader_mode7.v.pica):**
   - Apply additional parallax based on depth value
   - GPU-accelerated for maximum performance
   - Smooth depth effect without CPU overhead

**Implementation Steps:**

### Phase 1: GPU Shader Modifications

1. **Add stereoIOD uniform to `shader_tiles.v.pica`:**
   ```pica
   .fvec       stereoIOD            ; #6 - Inter-ocular distance in pixels
   ```

2. **Add parallax logic to `shader_tiles.v.pica`:**
   ```pica
   ; After computing x position, apply parallax based on depth
   cmp     r0.z, lt, lt, r2.y           ; if depth < 16384 (foreground)
   jmpc    cmp.x, apply_negative_offset
   ; Background: add positive offset
   mad     outpos.x, r0.x, r1.x, r0.x   ; outpos.x = inpos.x + stereoIOD
   jmp     done_offset
   apply_negative_offset:
   ; Foreground: subtract offset
   mad     outpos.x, r0.x, -r1.x, r0.x  ; outpos.x = inpos.x - stereoIOD
   done_offset:
   ```

3. **Add same uniform to `shader_mode7.v.pica`:**
   ```pica
   .fvec       stereoIOD            ; #6 - Inter-ocular distance in pixels
   ```

4. **Add parallax logic to `shader_mode7.v.pica`:**
   Similar to `shader_tiles.v.pica`

### Phase 2: CPU Shader Uniform Updates

1. **Add uniform location to `source/3dsgpu.h`:**
   ```cpp
   typedef enum {
       ULOC_PROJECTION,
       ULOC_TEX_SCALE,
       ULOC_TEX_OFFSET,
       ULOC_UPDATE_FRAME,
       ULOC_STEREO_IOD,  // New uniform location
       ULOC_COUNT
   } SGPU_SHADER_ULOC;
   ```

2. **Update shader uniform locations in `source/3dsgpu.cpp`:**
   ```cpp
   GPU3DS.shaderULocs[ULOC_STEREO_IOD] = shaderInstanceGetUniformLocation(
       GPU3DS.shaders[SPROGRAM_TILES].shaderProgram.vertexShader, "stereoIOD");
   ```

3. **Set stereoIOD uniform before drawing in `source/Snes9x/gfxhw.cpp`:**
   ```cpp
   if (gpu3dsIs3DEnabled()) {
       float iod = gpu3dsGetIOD();
       if (iod > 0.0f) {
           float stereoIOD[4] = {iod, 0.0f, 0.0f, 0.0f};
           shaderInstanceSetUniformFloat(GPU3DS.shaders[SPROGRAM_TILES].shaderProgram.vertexShader,
                                         GPU3DS.shaderULocs[ULOC_STEREO_IOD], stereoIOD, 4);
           shaderInstanceSetUniformFloat(GPU3DS.shaders[SPROGRAM_MODE7].shaderProgram.vertexShader,
                                         GPU3DS.shaderULocs[ULOC_STEREO_IOD], stereoIOD, 4);
       }
   }
   ```

### Phase 3: CPU Z Offset (Optional)

For proper layer separation, apply basic Z offset in CPU:

```cpp
static inline void S9xApplyStereoZOffset(int* depth0, int* depth1) {
    if (!gpu3dsIs3DEnabled()) return;
    
    float iod = gpu3dsGetIOD();
    if (iod <= 0.0f) return;
    
    int offset = (int)(iod * 256);
    *depth0 -= offset;
    *depth1 += offset;
}
```

---

## Summary

| Approach | Performance | Complexity | Maintainability | Recommendation |
|----------|-------------|------------|-----------------|----------------|
| Option A (GPU Only) | Excellent | High | Medium | Good for experienced developers |
| Option B (CPU Only) | Good | Low | High | Good for quick implementation |
| **Option C (Hybrid)** | **Excellent** | **Medium** | **High** | **Recommended for best balance** |

**Final Recommendation:** Implement **Option C (Hybrid)** for the best balance of performance, maintainability, and flexibility.

---

## Implementation Checklist

### Phase 1: GPU Shader Modifications
- [ ] Add `stereoIOD` uniform to `shader_tiles.v.pica`
- [ ] Add parallax logic to `shader_tiles.v.pica`
- [ ] Add `stereoIOD` uniform to `shader_mode7.v.pica`
- [ ] Add parallax logic to `shader_mode7.v.pica`
- [ ] Test shader compilation

### Phase 2: CPU Shader Uniform Updates
- [ ] Add `ULOC_STEREO_IOD` to `SGPU_SHADER_ULOC` enum in `source/3dsgpu.h`
- [ ] Update `gpu3dsInitializeShaderUniformLocations()` in `source/3dsgpu.cpp`
- [ ] Set `stereoIOD` uniform in `source/Snes9x/gfxhw.cpp` before drawing
- [ ] Test with 3D slider at minimum (should be 2D)
- [ ] Test with 3D slider at maximum (should have depth separation)

### Phase 3: CPU Z Offset (Optional)
- [ ] Add `S9xApplyStereoZOffset()` helper function to `source/Snes9x/gfxhw.cpp`
- [ ] Modify `S9xDrawBackgroundHardwarePriority0Inline()` to apply offset
- [ ] Modify `S9xDrawBackgroundMode7Hardware()` to apply offset
- [ ] Test with various SNES games
- [ ] Profile performance

---

## Notes

- The current implementation already has the infrastructure for 3D (`gpu3dsGetIOD()`, `gpu3dsIs3DEnabled()`) but doesn't use it during rendering
- The depth values from SNES PPU (`GFX.Z1`, `GFX.Z2`) are already being passed to the drawing functions
- The vertex shaders (`shader_tiles.v.pica`, `shader_mode7.v.pica`) can be modified to apply parallax in the GPU
- The existing layer separation (BG0-BG3, OBJ) will naturally create the 3D effect when Z offsets are applied
- Using GPU-based parallax (Option C) provides the best performance with minimal CPU overhead
