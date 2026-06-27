# Vulkan enforcement and media fixes

`vk_use_ogl_for_media` is an opt-in graphics fallback for devices where Vulkan samples media, HDR, or wide-color content incorrectly. When enabled, SurfaceFlinger keeps Vulkan for ordinary composition, OpenGL/Ganesh renders SurfaceFlinger media-sensitive client-composition passes, and HWUI processes not marked system-or-persistent use SkiaGL for TextureView and thumbnail content that is already rendered into an app buffer before SurfaceFlinger sees it.

## Status

- Scope: affected legacy devices or older SOCs.
- Default: disabled.
- Toggle: `persist.sys.vk_use_ogl_for_media=true`.
- Applies after SurfaceFlinger and affected app processes are recreated, or after the device is rebooted.

## Motivation

This research starts from Android's Vulkan-first graphics policy declaration:

- Vulkan is the primary low-level graphics API on Android.
- Vulkan is the preferred Android interface to the GPU.
- OpenGL ES remains supported, but it is no longer under active feature development.
- Android 15 and newer include ANGLE as a path for running OpenGL ES over Vulkan, with a roadmap for shipping ANGLE as the GL system driver on more new devices.

Axion needs Vulkan enabled for UI features such as skia glass blur. The goal is to keep affected devices aligned with that Vulkan-first direction while avoiding media, HDR, and thumbnail color issues on targets with known Vulkan sampling problems.

## Background

SurfaceFlinger composes visible layers from producers such as OpenGL ES, Canvas, Vulkan, camera, media decoder, and HDR image paths. Hardware Composer handles layers that can be composed by display hardware. RenderEngine handles layers that require client composition.

Decoded video buffers can carry video dataspaces, YUV-oriented pixel formats, or protected allocation requirements. On devices with a Vulkan media sampling issue, media-containing client composition needs a renderer choice that is independent from the renderer used for regular SurfaceFlinger UI layers.

HDR and Ultra HDR images can enter RenderEngine through HDR transfer dataspaces, high-bit-depth RGB buffers, per-layer luminance metadata, or the gainmap tonemapping path used by screen capture. Thumbnail and preview pixels can also enter an app ViewRoot through TextureView or image drawing before SurfaceFlinger receives the app buffer. These paths need backend isolation on devices where Vulkan import or sampling changes color.

## Problem

On the OnePlus Nord N30 5G SE (`larry`), the Vulkan RenderEngine path produced tinted frames in Instagram Reels. The issue appeared during app resume, launch transitions, Reels scrolling, and overlays above video such as the volume panel or notification shade.

Disabling Vulkan globally avoids the media issue but removes Vulkan from normal composition. The fallback narrows the change to affected devices: SurfaceFlinger stays Vulkan-first for ordinary composition, persistent system UI processes stay on their normal renderer, and media-sensitive RenderEngine passes plus HWUI processes not marked system-or-persistent avoid Vulkan sampling.

## Approach

At RenderEngine creation time, when Vulkan is selected and the flag is enabled, SurfaceFlinger creates a wrapper around two RenderEngine instances:

1. The primary Vulkan RenderEngine handles default composition.
2. A secondary OpenGL/Ganesh RenderEngine handles media-sensitive composition.
3. ExternalTexture buffer mapping is mirrored to both engines.
4. `AxRenderEngineMedia` carries media classification and writes a narrow `axContentHint` on LayerSettings for video, camera, image, thumbnail, HDR, Ultra HDR, and picture-profile paths.
5. Each draw selects OpenGL when any layer in the pass contains media-sensitive content.
6. `AxHwuiMedia` routes HWUI processes not marked system-or-persistent to SkiaGL while the property is enabled, covering TextureView and thumbnail paths that are rendered into an app buffer before SurfaceFlinger sees them.
7. HDR gainmap tonemapping selects OpenGL through the same wrapper.
8. SurfaceFlinger draws without media-sensitive content stay on Vulkan.
9. Protected content uses OpenGL only when the OpenGL engine supports protected content.

The property is read during RenderEngine and HWUI renderer setup, not during per-frame drawing.

## Source layout

| Area | Source | Responsibility |
|---|---|---|
| RenderEngine wrapper | `frameworks/native/libs/renderengine/RenderEngine.cpp` | Creates the primary Vulkan engine and secondary OpenGL/Ganesh engine, mirrors `ExternalTexture` mapping, selects OpenGL for media-sensitive draws, and keeps protected content on the primary path unless the OpenGL engine supports protected content. |
| Media classifier | `frameworks/native/libs/renderengine/AxRenderEngineMedia.cpp` | Classifies video, camera, image, HDR, Ultra HDR, gainmap, and media-like vendor buffers from dataspace, pixel format, usage, HDR metadata, and content hints. |
| Content hint | `frameworks/native/libs/renderengine/include/renderengine/LayerSettings.h` | Stores `axContentHint` beside the buffer metadata so media identity can survive later RenderEngine decisions. |
| SurfaceFlinger layer bridge | `frameworks/native/services/surfaceflinger/LayerFE.cpp` | Writes the content hint from the live layer snapshot before client composition. |
| Cached-set bridge | `frameworks/native/services/surfaceflinger/CompositionEngine/src/planner/CachedSet.cpp`, `Flattener.cpp`, and `OutputLayer.cpp` | Preserves `axContentHint` when video or photo layers are flattened into a cached override buffer and later replayed. |
| HWUI process policy | `frameworks/base/libs/hwui/AxHwuiMedia.cpp` and `Properties.cpp` | Selects SkiaGL for HWUI processes not marked system-or-persistent while the property is enabled, covering TextureView and app-rendered preview pixels before they reach SurfaceFlinger. |

## Media detection

A composition pass uses the media fallback when any layer has a buffer and `AxRenderEngineMedia` finds an `axContentHint`, layer dataspace, gralloc buffer dataspace, buffer pixel format, buffer usage, or HDR metadata that matches a media-sensitive path.

Detected video dataspaces include BT.601, limited-range BT.709, and BT.2020 variants. Limited-range BT.709 is detected even when SurfaceFlinger rewrites SMPTE 170M transfer to sRGB through `debug.sf.treat_170m_as_sRGB`.

Detected image and HDR signals include picture profiles, HDR/SDR headroom, luminance metadata, BT.2020 dataspaces, and high-bit-depth RGB buffers paired with wide-color or HDR dataspaces. Detected formats include vendor-private media formats, `YCbCr_420_888`, `YV12`, `YCBCR_P010`, 4:2:2 YUV formats, luma formats, FP16 RGBA, and 10-bit RGB formats when paired with a media-sensitive dataspace. Video decoder, video encoder, camera, and image encoder usage bits also select the fallback.

The classifier is deliberately narrower than a generic wide-color test. A normal RGB UI layer does not select the OpenGL RenderEngine only because it is Display P3 or 10-bit; it needs a media-oriented format, usage bit, video/HDR dataspace, picture profile, HDR headroom, or luminance signal.

SurfaceFlinger calls `AxRenderEngineMedia` when a layer carries a picture profile, HDR metadata, HDR/SDR headroom, visual-media dataspace, media-oriented pixel format, or media/camera/image usage bit. Planner cached sets preserve `axContentHint` so a cached video/photo group remains media-sensitive if a later client-composition pass samples the cached override buffer.

HWUI calls `AxHwuiMedia` before selecting SkiaVulkan. When the property is enabled, HWUI processes not marked system-or-persistent use SkiaGL so TextureView-backed previews, thumbnails drawn into app UI, and other media pixels already baked into a ViewRoot buffer do not rely on HWUI Vulkan sampling. Persistent processes such as SystemUI keep the normal Vulkan renderer when Vulkan is the configured default.

Ultra HDR gainmap tonemapping uses the fallback RenderEngine directly when the wrapper is enabled.

## RenderEngine selection

The wrapper delegates each draw to the primary or media RenderEngine. Hardware Composer policy and SurfaceFlinger composition decisions remain unchanged.

Selection logic:

```cpp
RenderEngine& selectRenderEngine(const std::vector<LayerSettings>& layers) const {
    if (hasMediaBuffer(layers) &&
        (!hasProtectedBuffer(layers) || mMedia->supportsProtectedContent())) {
        return *mMedia;
    }
    return *mPrimary;
}
```

ExternalTexture buffers are mapped to both engines so either backend can render the selected pass:

```cpp
void mapExternalTextureBuffer(const sp<GraphicBuffer>& buffer, bool isRenderable) override {
    mPrimary->mapExternalTextureBuffer(buffer, isRenderable);
    mMedia->mapExternalTextureBuffer(buffer, isRenderable);
}
```

## Device configuration

Enable the fallback only on affected devices:

```sh
persist.sys.vk_use_ogl_for_media=true
```

Leave the fallback disabled on devices where Vulkan media sampling is correct. This flag is not a generic performance option.

## Runtime inspection

Use `gfxinfo` to verify the intended renderer split after enabling the property and recreating affected processes:

```sh
adb shell getprop persist.sys.vk_use_ogl_for_media
adb shell dumpsys gfxinfo com.android.systemui | grep '^Pipeline='
adb shell dumpsys gfxinfo com.ss.android.ugc.trill | grep '^Pipeline='
```

Expected state on an affected device with Vulkan enabled:

```text
com.android.systemui: Pipeline=Skia (Vulkan)
com.ss.android.ugc.trill: Pipeline=Skia (OpenGL)
```

`dumpsys SurfaceFlinger` should also show both Vulkan and GLES RenderEngine instances when the SurfaceFlinger wrapper is active.

## Validation scenarios

| Area | Scenario | Expected result |
|---|---|---|
| Instagram Reels | Resume app directly into a playing Reel | No tinted frames during resume or launch transition |
| Instagram Reels | Scroll continuously between Reels | No Vulkan tint during scroll; video remains visible |
| Instagram Reels | Half-scroll or peek next Reel, then release | Current video does not stay black or hidden |
| Overlay over video | Show volume panel above TikTok or Reels video | Video color remains correct |
| Shade over video | Expand notification shade over playing video | Blur renders and video remains visible |
| Photos | Open HDR, Ultra HDR, and Display P3 image surfaces | Photos and thumbnails keep expected color during client composition |
| Screen capture | Capture HDR or Ultra HDR content with a gainmap | SDR output and gainmap are generated without Vulkan tint |
| Non-media SF UI | Open and close shade, launch apps, and trigger blur without video | SurfaceFlinger UI composition continues to use the Vulkan path |
| App previews | Open TextureView-backed thumbnails or video previews | Affected app HWUI reports SkiaGL and previews keep expected color |
| SystemUI renderer | Query `dumpsys gfxinfo com.android.systemui` with the property enabled | SystemUI reports Skia Vulkan when Vulkan is the configured default |
| TikTok renderer | Query `dumpsys gfxinfo com.ss.android.ugc.trill` with the property enabled | TikTok reports Skia OpenGL for TextureView and app-rendered previews |
| Protected playback | Play DRM or protected content where available | Protected content is not forced through an unsupported OpenGL path |
| Opt-out | Boot with `persist.sys.vk_use_ogl_for_media=false` | Default single RenderEngine behavior is preserved |

## References

- Android graphics architecture: https://source.android.com/docs/core/graphics
- SurfaceFlinger and WindowManager: https://source.android.com/docs/core/graphics/surfaceflinger-windowmanager
- Hardware Composer HAL: https://source.android.com/docs/core/graphics/hwc
- Vulkan: https://source.android.com/docs/core/graphics/arch-vulkan
- Use Vulkan for graphics: https://developer.android.com/games/develop/vulkan/overview
- Vulkanised 2024: Vulkan on Android: https://www.youtube.com/watch?v=0Z-J0XBmvEw&t=31
- Implement OpenGL ES and EGL: https://source.android.com/docs/core/graphics/implement-opengl-es
- BufferQueue and Gralloc protected buffers: https://source.android.com/docs/core/graphics/arch-bq-gralloc
