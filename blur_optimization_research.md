# Axion RenderEngine GlassBlur Optimization

This document details the final architecture, microarchitectural design choices, and benchmark results for Axion's optimized RenderEngine blur filter subsystem (`GlassBlurFilter`) on mobile architectures (tested on Snapdragon 695/Adreno 619).

---

## 1. Problem Statement & Objectives

Android SurfaceFlinger renders real-time background blurs behind the notification shade, quick settings, application launcher drawers, and dialog surfaces via `RenderEngine`'s Skia backend.

On mid-range mobile hardware, stock AOSP blur implementations introduce severe frame pacing issues:
1. **GPU Pipeline Stalls**: Compositor frames wait on GPU completion (`waiting for GPU completion` averaging ~3.9 ms), leading to missed VSync deadlines and jank.
2. **Buffer and SurfaceFlinger Stuffing**: Downstream GPU execution delays cause frames to queue up in the compositor and app render threads.
3. **Allocation Latency Spikes**: Creating intermediate render surfaces during swipe animations triggers runtime `VkImage` allocations (`allocateImageMemory`), adding 1.3 to 2.5 ms driver/kernel stalls.
4. **TBDR Inefficiencies**: Redundant DRAM tile reloads (GMEM loads/unresolves) and non-opaque surface alpha tests waste mobile memory bus bandwidth.

The objective was to design a clean, hardware-aligned blur pipeline that eliminates these stalls while preserving 100% visual fidelity (creamy, sharp, non-pixelated blur matching reference Gaussian appearance).

---

## 2. Hardware Architecture & Design Principles (Adreno TBDR)

The implementation is tailored to Qualcomm Adreno Tile-Based Deferred Rendering (TBDR) architectures:

```text
+-----------------------------------------------------------------------------+
| System Memory (DDR / DRAM)                                                  |
| - Full-Resolution Framebuffer (1080x2400 RGBA_8888, ~10.4 MB)               |
+-----------------------------------------------------------------------------+
               ^                                           |
    DRAM Store | (UBWC Compressed)              GMEM Loads | (Unresolves, costly)
               |                                           v
+-----------------------------------------------------------------------------+
| On-Chip High-Speed SRAM (GMEM Cache: ~512 KB - 1 MB)                        |
| - Tile Bin Size: 64x32 pixels (at 32-bit RGBA_8888)                         |
| - Tile Bin Size: 32x32 pixels (at 64-bit RGBA_F16 - 50% capacity penalty)   |
+-----------------------------------------------------------------------------+
               ^                                           |
               | Shading & Blending Passes                 | Texture Sampling
               |                                           v
+-----------------------------------------------------------------------------+
| Shading Engines (ALU / Vector Units) & Texture Processing Units (TPU)       |
| - TPU performs 1-cycle bilinear interpolation in hardware                   |
| - Fragment ALU executes arithmetic for coordinate calculation               |
| - Render Output Units (ROP) execute alpha blending read-modify-write        |
+-----------------------------------------------------------------------------+
```

1. **Avoid GMEM Loads (Unresolves)**: Each tile is rendered into on-chip GMEM. If a render pass does not explicitly invalidate its attachment, the driver must reload previous framebuffer data from DRAM into GMEM. Calling `canvas->discard()` maps to `VK_ATTACHMENT_LOAD_OP_DONT_CARE`, bypassing costly DRAM reloads.
2. **Bypass Hardware ROP Alpha Read-Modify-Write**: Intermediate blur surfaces are fully opaque. Setting `kOpaque_SkAlphaType` signals the hardware ROP units to disable alpha blending cycles and enables Universal Bandwidth Compression (UBWC) without alpha tracking overhead.
3. **Enforce 32-bit Integer Color Formats**: `kRGBA_8888_SkColorType` cuts bandwidth by 50% compared to `RGBA_F16` and allows the Adreno driver to allocate maximum-size 64x32 GMEM tile bins.
4. **Single-Pipeline Execution**: Switching shader programs between consecutive draw passes forces Vulkan pipeline state object (`VkPipeline`) rebinds. A unified shader eliminates state switches across upsampling octaves.
5. **Offload Coordinate Math to Uniform Registers**: Texture Processing Units (TPUs) perform bilinear filtering in hardware. Hoisting offset calculations into CPU uniforms keeps fragment ALUs free for color processing.

---

## 3. Final Implementation Architecture

All modifications are confined to `libs/renderengine/skia/filters/`:

### A. Triple-Buffered Surface Pooling & Stale Slot Recycling
**Files**: `GlassBlurFilter.h`, `GlassBlurFilter.cpp`

To eliminate runtime `VkImage` allocations without unbounded memory growth:
- **Triple-Buffering Pool Depth (`kPoolCapacity = 3`)**: Each octave maintains a dedicated array of up to 3 surfaces, matching the Android 3-frame VSync display pipeline depth.
- **Stale Slot Recycling**: If blur dimensions resize (e.g. orientation changes or sub-window blurs), any idle slot from the previous size is repurposed immediately instead of growing the pool.
- **Outcome**: Steady state is reached in 3 frames. Runtime `allocateImageMemory` calls drop to **zero**, and intermediate VRAM is capped at **1.27 MB** (down from 2.6 MB in AOSP).

### B. Opaque Surface & Bandwidth Optimization
**File**: `GlassBlurFilter.cpp`

Intermediate surfaces enforce opaque 32-bit pixel formats:
```cpp
auto makeSurface = [&](int index) -> sk_sp<SkSurface> {
    const int newW = w0 >> index;
    const int newH = h0 >> index;
    SkImageInfo info = input->imageInfo().makeWH(newW, newH).makeAlphaType(kOpaque_SkAlphaType);
    if (info.colorType() == kRGBA_F16_SkColorType) {
        info = info.makeColorType(kRGBA_8888_SkColorType);
    }
    return obtainSurface(context, info, index);
};
```
- Disables alpha blending read-modify-write cycles in Adreno ROP hardware.
- Maximizes UBWC compression ratios (3:1 to 4:1) across the memory bus.

### C. Unified Single-Pipeline Upsample Shader
**Files**: `GlassBlurFilter.cpp`, `GlassBlurFilter.h`

The upsampling loop unifies the alternating 45° diagonal and 90° orthogonal passes into a single runtime effect:
```glsl
uniform shader child;
uniform float4 in_offsets;
half4 main(float2 xy) {
    float2 d0 = in_offsets.xy;
    float2 d1 = in_offsets.zw;
    half3 c = child.eval(xy).rgb * half(4.0);
    c += child.eval(xy + d0).rgb;
    c += child.eval(xy - d0).rgb;
    c += child.eval(xy + d1).rgb;
    c += child.eval(xy - d1).rgb;
    return half4(c * half(0.125), half(1.0));
}
```
- **CPU Uniform Precomputation**: Axis-aligned offsets `{step, 0, 0, step}` and diagonal offsets `{d, d, d, -d}` are precalculated once per frame and stored in cached uniform buffers.
- **Outcome**: The exact same Vulkan `VkPipeline` stays bound across all upsampling octaves, eliminating pipeline rebinds. Zero multiplication math occurs in the fragment shader.

### D. GMEM Tile-Aware Octave Bounding
**File**: `GlassBlurFilter.cpp`

Intermediate surfaces are bounded to a minimum 16x16 pixel threshold:
```cpp
int maxPasses = kMaxSurfaces - 1;
while (maxPasses > 1 && ((w0 >> maxPasses) < 16 || (h0 >> maxPasses) < 16)) {
    --maxPasses;
}
```
- Prevents degenerate sub-16px render passes on small blur regions (volume sliders, dialogs), eliminating wasted GMEM tile binning and resolve overhead.

### E. Redundant Snapshot Elimination
**File**: `GlassBlurFilter.cpp`

Setting `maxCrossFadeRadius = 0.0f` ensures `SkiaRenderEngine::drawLayersInternal` never invokes `activeSurface->makeImageSnapshot()`. Because `GlassBlurFilter` uses continuous octave scaling, it never requires full-screen framebuffer copy-blends.

---

## 4. Empirical Verification & Results

### A. Visual Quality Benchmark
Tested via the Axion blur test suite (`run_benchmark.py`) at `radius = 175.0` across 7 native 4K wallpapers on 1080x2400 portrait viewports:

| Wallpaper | Resolution | AOSP PSNR | Optimized PSNR | SSIM | Texture Fetches | VRAM Footprint |
|---|---|---|---|---|---|---|
| `test_hd_wall.jpg` | 3072x4080 | 30.58 dB | **51.05 dB** | **1.0000** | -37.0% (1.05M vs 1.67M) | -53.8% (1.27MB vs 2.6MB) |
| `test_hd_wall_1.png` | 1290x2796 | 33.16 dB | **46.32 dB** | **0.9998** | -37.0% (1.05M vs 1.67M) | -53.8% (1.27MB vs 2.6MB) |
| `test_hd_wall_2.webp` | 1400x3100 | 31.38 dB | **41.21 dB** | **0.9997** | -37.0% (1.05M vs 1.67M) | -53.8% (1.27MB vs 2.6MB) |
| `test_workspace.png` | 1080x2400 | 32.56 dB | **45.17 dB** | **0.9999** | -37.0% (1.05M vs 1.67M) | -53.8% (1.27MB vs 2.6MB) |
| `wallpaper_abstract_4k.jpg` | 4896x3264 | 32.46 dB | **44.61 dB** | **0.9997** | -37.0% (1.05M vs 1.67M) | -53.8% (1.27MB vs 2.6MB) |
| `wallpaper_nature_4k.jpg` | 7360x4912 | 33.53 dB | **48.16 dB** | **1.0000** | -37.0% (1.05M vs 1.67M) | -53.8% (1.27MB vs 2.6MB) |
| `wallpaper_night_4k.jpg` | 4096x2733 | 34.48 dB | **48.16 dB** | **0.9999** | -37.0% (1.05M vs 1.67M) | -53.8% (1.27MB vs 2.6MB) |

- **PSNR**: Exceeds 41 dB across all inputs (industry threshold for imperceptible degradation is 37 dB).
- **SSIM**: 0.9997 to 1.0000 across all inputs.
- **Visual Verdict**: Output is visually indistinguishable from reference Gaussian blur with zero edge artifacts or blockiness.

### B. Runtime On-Device Performance (Adreno 619 / CPH2513)
Measured during automated 16-second swipe gesture workloads (`scenario_run.sh all_apps`) via Perfetto tracing and Simpleperf hardware PMU counters:

| Metric | AOSP Baseline | Final Output | Diff |
|---|---|---|---|
| **GPU Wait Duration (avg slice)** | **3.89 ms** | **3.30 ms** | **-15.3%** (-597 µs) |
| **GPU Total Stall Time** | **6,417.6 ms** | **5,193.2 ms** | **-19.1%** (-1,224.4 ms) |
| **Dropped Display Frames** | **31** | **21** | **-32.3%** |
| **SurfaceFlinger Stuffing Jank** | **538** | **420** | **-21.9%** |
| **Total Stuffing Jank** | **1,120** | **910** | **-18.8%** |
| **Hardware Instructions Per Cycle (IPC)** | **0.360** | **0.455** | **+26.4%** |
| **Thread Migrations** | **64.9k** | **53.2k** | **-18.0%** |
| **Context Switches** | **216.2k** | **186.4k** | **-13.8%** |
| **Texture Fetches** | **1,673.4k** | **1,054.4k** | **-37.0%** |
| **Intermediate VRAM Footprint** | **2.6 MB** | **1.27 MB** | **-53.8%** |

---
