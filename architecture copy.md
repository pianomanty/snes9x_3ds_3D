# SNES9x 3DS: Stereoscopic 3D Architecture

## 1. The Core Objective
Separate the SNES Background Layers (BG0, BG1, BG2, BG3) and Sprites into distinct Z-planes when 3DS Stereoscopic mode is enabled.

## 2. Technical Stack
- **Base:** Snes9x (C++ source)
- **Platform:** Nintendo 3DS (devkitPro / libctru)
- **Graphics:** PICA200 GPU (Fixed-function fragment stage, programmable vertex shaders)

## 3. The Rendering Pipeline (Internal Knowledge)
- **Current State:** The 3DS port likely uses a hardware-accelerated "compositor" that takes the SNES output and draws it as a single quad (or series of quads) to the 3DS screen.
- **Goal State:**
   - **Left Eye Pass:** Draw BG layers with a negative horizontal offset ($X - \Delta$) based on depth.
   - **Right Eye Pass:** Draw BG layers with a positive horizontal offset ($X + \Delta$).
   - **Z-Buffer:** Use the 3DS's 24-bit depth buffer to prevent layer bleeding.

## 4. Key Codebase Landmarks
- `source/3ds/gfx.cpp`: Likely handles the PICA200 initialization and frame swapping.
- `source/ppu.cpp`: Where the SNES background logic lives.
- `source/3ds/video.cpp`: The bridge between SNES pixels and 3DS textures.

---

## Current Rendering Analysis

### 1. Where is the 3D Slider Value Being Read?

**Location:** [`source/3dsgpu.cpp:44`](source/3dsgpu.cpp:44)

The 3D slider value is read using the Citrus library function `osGet3DSliderState()`:

```cpp
float gpu3dsGetIOD()
{
    float sliderVal = osGet3DSliderState();
    return sliderVal * IOD_MAX_PIXELS;
}
```

- **Function:** `gpu3dsGetIOD()` (Get Inter-Ocular Distance)
- **Location:** `source/3dsgpu.cpp`, lines 42-46
- **Return Value:** The slider value (0.0 to 1.0) multiplied by `IOD_MAX_PIXELS` (3.0f), resulting in a maximum offset of 3 pixels
- **Usage:** This value is used to determine the horizontal offset for left/right eye rendering when 3D is enabled

**Note:** The actual hardware register read is handled internally by `osGet3DSliderState()` from the Citrus library, which reads from the 3DS's depth slider hardware register.

---

### 2. Which File Handles the Final Drawing of SNES Background Layers to the 3DS Screens?

**Primary File:** [`source/3dsimpl_gpu.cpp`](source/3dsimpl_gpu.cpp)

The final drawing of SNES background layers is handled by the following functions:

#### Main Entry Point: `gpu3dsDrawSnesScreen()`
**Location:** [`source/3dsimpl_gpu.cpp:419`](source/3dsimpl_gpu.cpp:419)

```cpp
void gpu3dsDrawSnesScreen() {
    SLayerList *list = &GPU3DSExt.layerList;
    
    if (!list->verticesTotal || list->hasSkippedSections)
        return;

    LAYER_ID drawOrder[8] = {
        LAYER_BACKDROP,
        LAYER_OBJ,
        LAYER_BG0,
        LAYER_BG1,
        LAYER_BG2,
        LAYER_BG3,
        LAYER_COLOR_MATH,
        LAYER_BRIGHTNESS,
    };

    // ... layer processing and drawing ...
    
    gpu3dsDrawMode7Texture();
    gpu3dsDrawLayers(list);
    gpu3dsResetLayers(list);
}
```

#### Layer Drawing Functions:

1. **`gpu3dsDrawLayers()`** - [`source/3dsimpl_gpu.cpp:308`](source/3dsimpl_gpu.cpp:308)
   - Draws all SNES layers (backdrop, objects, BG0-BG3, color math, brightness)
   - Handles both main and sub screen targets
   - Uses depth buffer for window_lr layer

2. **`gpu3dsDrawTiledLayer()`** - [`source/3dsimpl_gpu.cpp:248`](source/3dsimpl_gpu.cpp:248)
   - Draws tiled layers (BG0-BG3, OBJ)
   - Uses `C3D_DrawArrays()` and `C3D_DrawElements()` for rendering

3. **`gpu3dsDrawVerticalSectionLayer()`** - [`source/3dsimpl_gpu.cpp:227`](source/3dsimpl_gpu.cpp:227)
   - Draws vertical section layers (window_lr, backdrop, color math, brightness)
   - Handles per-section state changes

#### C3D/Citro3d Calls:

The actual C3D rendering calls are in [`source/3dsimpl_gpu.cpp:243`](source/3dsimpl_gpu.cpp:243) and [`source/3dsimpl_gpu.cpp:279`](source/3dsimpl_gpu.cpp:279):

```cpp
// In gpu3dsDrawVerticalSectionLayer()
gpu3dsDraw(&GPU3DS.vertices[section->vboId], NULL, section->count, section->from);

// In gpu3dsDrawTiledLayer()
gpu3dsDraw(&GPU3DS.vertices[vboId], (void *)(indices + batchFrom), batchCount);
```

The `gpu3dsDraw()` function (defined in [`source/3dsgpu.cpp:346`](source/3dsgpu.cpp:346)) makes the actual C3D calls:

```cpp
if (indices != NULL) {
    C3D_DrawElements(list->primitive, count, C3D_UNSIGNED_SHORT, indices);
} else if (from >= 0) {
    C3D_DrawArrays(list->primitive, from, count);
} else {
    C3D_DrawArrays(list->primitive, list->from, list->count);
}
```

---

### 3. Does the Current Implementation Have a 'Left Eye' and 'Right Eye' Rendering Loop?

**Answer: No, the current implementation does NOT have separate left/right eye rendering loops.**

#### Evidence:

1. **Single Render Target per Frame:**
   Looking at [`source/3dsgpu.cpp:386-404`](source/3dsgpu.cpp:386), the `gpu3dsClearScreen()` function shows that while there's support for clearing both left and right targets when `isTopStereo` is true, the actual rendering happens to a single target:

```cpp
bool gpu3dsClearScreen(gfxScreen_t screen, bool isTopStereo) {
    SCREEN_TARGET targetId = screen == GFX_TOP ? SCREEN_TARGET_LEFT : SCREEN_TARGET_BOTTOM;
    
    if (!C3D_FrameDrawOn(GPU3DS.screenTargets[targetId])) {
        return false;
    }
    
    C3D_RenderTargetClear(GPU3DS.screenTargets[targetId], C3D_CLEAR_COLOR, 0, 0);
    
    if (isTopStereo && screen != GFX_BOTTOM) {
        C3D_RenderTargetClear(GPU3DS.screenTargets[SCREEN_TARGET_RIGHT], C3D_CLEAR_COLOR, 0, 0);
        C3D_FrameDrawOn(GPU3DS.screenTargets[SCREEN_TARGET_RIGHT]);
    }
    
    return true;
}
```

2. **No Stereo Rendering Loop in `gpu3dsDrawSnesScreen()`:**
   The `gpu3dsDrawSnesScreen()` function in [`source/3dsimpl_gpu.cpp:419-476`](source/3dsimpl_gpu.cpp:419) does NOT contain any left/right eye rendering loop. It simply:
   - Collects layer data
   - Calls `gpu3dsDrawMode7Texture()`
   - Calls `gpu3dsDrawLayers()`
   - Resets layers

3. **No Per-Eye Offset Application:**
   The `gpu3dsGetIOD()` function (which reads the 3D slider) is defined but **not used** in the actual rendering pipeline. The depth values from the SNES PPU are used for Z-buffering within the shader, but no horizontal offset is applied based on the 3D slider.

4. **Single Pass Rendering:**
   The rendering pipeline in `gpu3dsDrawLayers()` processes each layer once per frame, drawing to either the main or sub target, but not both with different offsets.

#### Current Behavior:

The current implementation:
- **Draws to a single screen target** per frame (either top or bottom, depending on settings)
- **Uses the 3D slider value** only for potential future use (the `gpu3dsGetIOD()` function exists but isn't called during rendering)
- **Does NOT perform stereo rendering** - there's no left eye pass followed by a right eye pass
- **Does NOT apply horizontal offsets** based on depth values for stereoscopic 3D

#### What Would Be Needed for Stereo 3D:

To implement proper stereoscopic 3D, the following would be required:

1. **Dual Pass Rendering Loop:**
   ```cpp
   // Left eye pass
   C3D_FrameDrawOn(GPU3DS.screenTargets[SCREEN_TARGET_LEFT]);
   applyHorizontalOffset(-iod);
   gpu3dsDrawSnesScreen();
   
   // Right eye pass
   C3D_FrameDrawOn(GPU3DS.screenTargets[SCREEN_TARGET_RIGHT]);
   applyHorizontalOffset(+iod);
   gpu3dsDrawSnesScreen();
   ```

2. **Depth-Based Offset Application:**
   The depth values from the SNES PPU (stored in `GFX.Z1` and `GFX.Z2`) would need to be used to calculate per-pixel or per-tile horizontal offsets.

3. **Shader Modifications:**
   The vertex shaders (`shader_tiles.v.pica`, `shader_mode7.v.pica`) would need to apply the horizontal offset based on depth values.

---

## Summary

| Question | Answer |
|----------|--------|
| **Where is the 3D slider read?** | `source/3dsgpu.cpp:44` via `osGet3DSliderState()` in `gpu3dsGetIOD()` |
| **Which file handles final drawing?** | `source/3dsimpl_gpu.cpp` - specifically `gpu3dsDrawSnesScreen()`, `gpu3dsDrawLayers()`, `gpu3dsDrawTiledLayer()` |
| **Does it have left/right eye rendering?** | **No** - The current implementation draws to a single target per frame with no stereo rendering loop |

The current implementation is essentially a **2D renderer** that happens to have the infrastructure for 3D (depth buffer, slider reading) but does not actually perform stereoscopic rendering. The depth values from the SNES PPU are used for Z-buffering within the shader, but no actual stereo separation is applied.
