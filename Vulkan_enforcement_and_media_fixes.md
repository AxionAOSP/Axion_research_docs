# Vulkan enforcement and media fixes

This document explains how Axion enables Vulkan across Android while fixing video and photo color bugs on older chips like the Snapdragon 695 (Adreno 619).

## Summary

Vulkan is modern, fast, and makes UI features like glass blur smooth. But on some chips, the phone's Vulkan driver has a bug that makes videos and photos look green, purple, or washed out.

We fixed this by keeping the entire phone on Vulkan, while using a small helper to convert video frames to clean colors before Vulkan draws them.

- **Status:** Setting available for affected chips.
- **Setting name:** `persist.sys.vk_use_ogl_for_media=true`.
- **Default:** Disabled (`false`).
- **Normal devices:** Phones with good Vulkan drivers run 100% native Vulkan with zero overhead.

## Why Axion forces Vulkan

Android is moving away from OpenGL and making Vulkan the main graphics engine. Axion enables Vulkan everywhere for the following reasons:

1. **Smooth glass blur (`AxBlur`):** The Axion glass blur only runs with skia blur.
2. **Less stutter and lag:** Vulkan saves compiled shaders to storage. This stops the micro-stutters you often see when opening apps or swiping between tasks.

## What went wrong with our first attempt

In our first test, we tried to turn off Vulkan for any app that plays media. That failed because modern apps mix UI and video together:

- **The Recent Apps screen broke:** Launcher3 Recents Apps may contain media buffers/layers. Because we forced Launcher3 to Vulkan, opening Recent Apps caused vulkan glitches on some devices.
- **Most apps lost Vulkan:** Instagram, TikTok, and web browsers lost Vulkan completely. They were forced onto OpenGL, losing all the benefits of Vulkan.

## Solution

It is not possible to render a window half in OpenGL and half in Vulkan at the same time. The window can only run on one pipeline (vulkan is what we want).

Instead of switching entire apps between OpenGL and Vulkan, we fix only the video frames:

1. The app UI, text, buttons, and blurs run on **Vulkan**.
2. When a video frame arrives from TikTok or Instagram Reels, a small helper converts only that video frame to clean RGB colors using OpenGL.
3. Vulkan then takes that clean frame and draws it onto the screen alongside the rest of the UI.
4. When you open the Recent Apps screen, the system takes preview screenshots in standard sRGB colors so Launcher3 never runs into vulkan bugs.

## Research flowchart

```text
[ FIRST ATTEMPT: PER-APP SWITCH ]
Turn off Vulkan for any app that uses video. Force Launcher to Vulkan.
         |
         v
[ RESULT: FAILED ]
- Third-party apps lost Vulkan speed and blurs.
- Launcher broke in Recent Apps because preview cards contain video.
         |
         v
[ ROOT-CAUSE ANALYSIS ]
- A single app window cannot mix two graphics drivers.
- The phone's Vulkan driver fails when reading video color formats.
         |
         v
[ SOLUTION: CONVERT FRAMES AT THE SOURCE ]
- Every app and the home screen run 100% on Vulkan.
- Convert only video frames to clean RGB before Vulkan draws them.
- Save Recent Apps preview cards in clean sRGB colors.
         |
         v
[ VERIFIED ON REAL HARDWARE (larry / Snapdragon 695) ]
- Instagram, Launcher, and SystemUI all run on Vulkan.
- No green tint, no crashes, smooth 120Hz scrolling.
```

## How the system decides what to do

```text
                       [ New Frame Arrives ]
                                 |
         +-----------------------+-----------------------+
         |                                               |
         v                                               v
[ Inside an App Window ]                       [ Fullscreen / System ]
         |                                               |
Runs on Vulkan.                                Is it a preview screenshot?
Is it a video view (TextureView)?              +---------+---------+
         |                                     | Yes               | No
   +-----+-----+                               v                   v
   | Yes       | No                    Save as clean       Display processor (HWC)
   v           v                       sRGB image.         draws video directly.
Is it a video  Standard UI                     |                   |
format (YUV)?  (Drawn with Vulkan)             v                   v
   |                                   Recent Apps card    Zero GPU load;
 +-+---+                               shows in Vulkan     no color bugs.
 | Yes | No                            without bugs.
 v     v
Convert  Draw with
to RGB   native
first.   Vulkan.
 |
 v
Drawn cleanly
with Vulkan.
```

## Where the code lives

To keep the system clean and avoid cluttering Android source code, all conversion logic lives in a separate library:

| Area | File path | What it does |
|---|---|---|
| **Conversion library** | `axion_sdk/ax_graphics/` | Standalone helper that converts video frames to clean RGB colors. |
| **Video view hook** | `frameworks/base/libs/hwui/AutoBackendTextureRelease.cpp` | Catches video frames in apps like TikTok and Reels, sending them to the converter. |
| **HDR photo hook** | `frameworks/base/libs/hwui/SkiaCanvas.cpp` | Stops broken HDR shaders from blowing out bright spots in photos. |
| **Full Vulkan policy** | `frameworks/base/libs/hwui/Properties.cpp` | Keeps Vulkan enabled for all apps. |
| **Screenshot safety** | `frameworks/native/services/surfaceflinger/SurfaceFlinger.cpp` | Makes sure task screenshots are saved in standard sRGB colors. |

## How to turn on the setting

If your phone has the Vulkan color bug, enable the setting in terminal or adb:

```sh
persist.sys.vk_use_ogl_for_media=true
```

If your phone does not have the color bug, leave this setting off (`false`). Normal phones run 100% native Vulkan without doing any conversion.

## How to check if it is working

You can verify that all apps are running on Vulkan using adb:

```sh
# 1. Check if the setting is on
adb shell getprop persist.sys.vk_use_ogl_for_media

# 2. Check that your apps are running on Vulkan
adb shell dumpsys gfxinfo com.android.launcher3 | grep '^Pipeline='
adb shell dumpsys gfxinfo com.android.systemui | grep '^Pipeline='
adb shell dumpsys gfxinfo com.instagram.android | grep '^Pipeline='
adb shell dumpsys gfxinfo com.ss.android.ugc.trill | grep '^Pipeline='

# All apps should print:
# Pipeline=Skia (Vulkan)
```

## Test results on real hardware (OnePlus Nord N30 5G SE)

| Test case | With the fix (`prop=true`) | On normal phones (`prop=false`) |
|---|---|---|
| **Home screen and app drawer** | Vulkan (Smooth glass blur) | Vulkan (Smooth glass blur) |
| **Recent Apps screen** | Vulkan (Clean preview cards, no tint) | Vulkan (Normal previews) |
| **Instagram Reels / TikTok** | App on Vulkan; video colors are clean | App on Vulkan; video colors are clean |
| **YouTube fullscreen** | Display hardware plays video directly | Display hardware plays video directly |
| **HDR photos (Google Photos)** | Clean colors without blown-out bright spots | Native Vulkan HDR rendering |
| **Notification shade over video** | Smooth glass blur over playing video | Smooth glass blur over playing video |
| **Pipeline check** | All apps report Vulkan | All apps report Vulkan |

## References

- [Android graphics architecture overview](https://source.android.com/docs/core/graphics)
- [SurfaceFlinger and WindowManager interaction](https://source.android.com/docs/core/graphics/surfaceflinger-windowmanager)
- [Hardware Composer (HWC) HAL implementation](https://source.android.com/docs/core/graphics/hwc)
- [Vulkan architecture on Android](https://source.android.com/docs/core/graphics/arch-vulkan)
- [Using Vulkan for Android graphics](https://developer.android.com/games/develop/vulkan/overview)
- [Vulkanised 2024: Vulkan on Android](https://www.youtube.com/watch?v=0Z-J0XBmvEw&t=31)
- [Implementing OpenGL ES and EGL](https://source.android.com/docs/core/graphics/implement-opengl-es)
- [BufferQueue and Gralloc architecture](https://source.android.com/docs/core/graphics/arch-bq-gralloc)
- [Android NDK AHardwareBuffer API reference](https://developer.android.com/ndk/reference/group/a-hardware-buffer)
- [Khronos VK_ANDROID_external_memory_android_hardware_buffer specification](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_ANDROID_external_memory_android_hardware_buffer.html)
- [Khronos VK_KHR_sampler_ycbcr_conversion specification](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VK_KHR_sampler_ycbcr_conversion.html)
- [Android WebView OpenGL ES on Vulkan Functor (`VkInteropFunctorDrawable`)](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/libs/hwui/pipeline/skia/VkInteropFunctorDrawable.cpp)
