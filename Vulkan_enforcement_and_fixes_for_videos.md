# Vulkan media fallback

`vk_use_ogl_for_media` is an opt-in SurfaceFlinger RenderEngine fallback for devices where Vulkan samples decoded video buffers incorrectly. When enabled, Vulkan remains the default renderer for ordinary UI composition, and OpenGL/Ganesh renders client-composition passes that contain media buffers.

## Status

- Scope: affected Qualcomm devices.
- Default: disabled.
- Toggle: `persist.sys.vk_use_ogl_for_media=true`.
- Applies after SurfaceFlinger is recreated or the device is rebooted.

## Motivation

This research starts from Android's Vulkan-first graphics policy declaration:

- Vulkan is the primary low-level graphics API on Android.
- Vulkan is the preferred Android interface to the GPU.
- OpenGL ES remains supported, but it is no longer under active feature development.
- Android 15 and newer include ANGLE as a path for running OpenGL ES over Vulkan, with a roadmap for shipping ANGLE as the GL system driver on more new devices.

Axion needs Vulkan enabled for UI features such as Skia glass blur. The goal is to keep affected devices aligned with that Vulkan-first direction while avoiding video issues on targets with known Vulkan media sampling problems.

## Background

SurfaceFlinger composes visible layers from producers such as OpenGL ES, Canvas, Vulkan, camera, and media decoder paths. Hardware Composer handles layers that can be composed by display hardware. RenderEngine handles layers that require client composition.

Decoded video buffers can carry video dataspaces, YUV-oriented pixel formats, or protected allocation requirements. On devices with a Vulkan media sampling issue, media-containing client composition needs a renderer choice that is independent from the renderer used for regular UI layers.

## Problem

On the OnePlus Nord N30 5G SE (`larry`), the Vulkan RenderEngine path produced tinted frames in Instagram Reels. The issue appeared during app resume, launch transitions, Reels scrolling, and overlays above video such as the volume panel or notification shade.

Disabling Vulkan globally avoids the media issue but removes Vulkan from normal UI rendering. The fallback narrows the change to media sampling: video buffers avoid the Vulkan sampling path, while non-media UI composition continues to use Vulkan.

## Approach

At RenderEngine creation time, when Vulkan is selected and the flag is enabled, SurfaceFlinger creates a wrapper around two RenderEngine instances:

1. The primary Vulkan RenderEngine handles default composition.
2. A secondary OpenGL/Ganesh RenderEngine handles media-containing composition.
3. ExternalTexture buffer mapping is mirrored to both engines.
4. Each draw selects OpenGL when any layer in the pass contains media.
5. Draws without media stay on Vulkan.
6. Protected media uses OpenGL only when the OpenGL engine supports protected content.

The property is read during RenderEngine setup, not during per-frame drawing.

## Media detection

A composition pass uses the media fallback when any layer has a buffer and either the layer dataspace or the buffer pixel format matches a video path.

Detected video dataspaces include BT.601, BT.709, and BT.2020 variants. Detected video formats include implementation-defined YUV buffers, `YCbCr_420_888`, `YV12`, and `YCBCR_P010`.

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

## Validation scenarios

| Area | Scenario | Expected result |
|---|---|---|
| Instagram Reels | Resume app directly into a playing Reel | No tinted frames during resume or launch transition |
| Instagram Reels | Scroll continuously between Reels | No Vulkan tint during scroll; video remains visible |
| Instagram Reels | Half-scroll or peek next Reel, then release | Current video does not stay black or hidden |
| Overlay over video | Show volume panel above TikTok or Reels video | Video color remains correct |
| Shade over video | Expand notification shade over playing video | Blur renders and video remains visible |
| Non-media UI | Open and close shade, launch apps, and trigger blur without video | UI continues to use the Vulkan path |
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
