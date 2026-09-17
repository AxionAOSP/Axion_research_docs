# AxDragonite Performance Engine

AxDragonite is AxionOS's scene-based performance and scheduling engine. It is an AOSP-compatible rewrite of the NothingOS "Dragonite" architecture, modularized into the Axion SDK to eliminate hardcoded board-specific configs and minimize invasive code changes in core Android frameworks.

The engine coordinates CPU cluster topology detection, Linux cgroups (cpuctl, cpuset, schedtune), real-time thread scheduling (`SCHED_RR`), CPU frequency scaling, process freezing, and SurfaceFlinger thread placement during interactive workloads.

---

## Why AxDragonite Exists

Standard AOSP builds running on Qualcomm and MediaTek platforms often suffer from stutter and frame drops during transient animations and gestures:

1. **Animation-Unaware Kernel Scheduling:** Energy-Aware Scheduling (EAS) relies on instantaneous load metrics. During sudden UI events (notification shade pulls, gesture navigation, app switching), EAS keeps critical threads on small/efficiency cores because instantaneous utilization registers as low.
2. **Resource Contention from Background Tasks:** Non-critical background applications and background services consume cycles on high-performance cores during touch interaction and app startup.
3. **Absence of OEM Boost Frameworks:** Stock vendor ROMs bundle proprietary boost daemons (Qualcomm `BoostFramework`, MediaTek `PowerHAL`) that inject aggressive frequency and scheduler hints. When running vanilla AOSP without these proprietary binaries, devices lack targeted boosts for UI scenes.
4. **SurfaceFlinger and UI Thread Migration:** SurfaceFlinger's main thread, its RenderEngine thread, and application HWUI threads often drift across efficiency cores during frame production.

AxDragonite solves these problems by dynamically discovering hardware topology and asserting direct scheduler, cgroup, and frequency controls when specific user-facing scenes start and stop.

---

## High-Level Architecture

AxDragonite is divided into five cooperating layers across the ROM:

```text
+-----------------------------------------------------------------------------+
| SystemUI Layer (axion_sdk / packages/SystemUI)                              |
| - AxShadeExpansionBoostStartable       - AxKeyguardUnlockBoostStartable     |
| - AxBiometricAuthBoostStartable        - AxLightRevealScrimStartable        |
| - AxDozeAnimationBoostStartable        - AxVolumeDialogBoostStartable       |
| - AxWakefulnessBoostStartable          - NotificationPanel / StackScroll    |
+-----------------------------------------------------------------------------+
                                      |
                                      v
+-----------------------------------------------------------------------------+
| Client API (axion_sdk: com.android.axion.dragonite.AxDragonite)             |
| - acquire(sceneId, duration, bundle) / release(sceneId)                     |
| - Scene helper methods (onAppLaunch, onFling, onShadeExpand, etc.)          |
+-----------------------------------------------------------------------------+
                                      | IPC: IActivityManager.sceneBoostAcquire
                                      v
+-----------------------------------------------------------------------------+
| Framework Hooks (frameworks/base)                                           |
| - ActivityManagerService (IPC entry point)  - ProcessList (fork/start/kill) |
| - ActivityStarter & ActivityRecord          - AudioService (ringtone mode)  |
| - DisplayPolicy & DisplayContent            - GameManagerService (Game mode)|
| - PointerEventDispatcher / PhoneWindowManager (Touch/Input Boost)           |
| - OomAdjuster / AxOomAdjusterHelper (ax_foreground cgroup routing)          |
+-----------------------------------------------------------------------------+
                                      |
                                      v
+-----------------------------------------------------------------------------+
| Server Engine (axion_sdk: com.android.server.axdragonite)                   |
| - AxDragonite (Coordinator, lease manager, Worker & Timer threads at SCHED_RR)
| - AxCpuClusterManager (Dynamic sysfs CPU topology scanner)                  |
| - AxPerfEnhancer (Freq scaling, schedtune, uclamp, shares, PowerManager)     |
| - AxUIBooster (App UI + HWUI thread discovery and SCHED_RR promotion)       |
| - AxBoostAdjuster (Animation boosts, touch input boost, timer management)   |
| - AxNamedThreadAffinityFeature (Thread name scanner: RenderThread, GL, etc.)|
| - AxFreezerController (cgroup v2 / v1 background process freezer)           |
| - AxFrameInsertManager (debug.choreographer.skipwarning toggle)             |
+-----------------------------------------------------------------------------+
            |                                               |
            v Backdoor Binder (2007)                        v Cgroups & Sched
+------------------------------------+     +----------------------------------+
| SurfaceFlinger (frameworks/native) |     | Linux Kernel / init (system/core)|
| - SurfaceFlinger::bindSFThread     |     | - /dev/cpuset/ax_foreground      |
|   Pins SF main & RenderEngine      |     | - sched_setaffinity / SCHED_RR   |
|   threads to performance cores     |     | - kswapd pin / per-task boost    |
+------------------------------------+     +----------------------------------+
```

---

## Core Components and Mechanisms

### 1. Dynamic CPU Cluster Topology Detection (`AxCpuClusterManager`)

Unlike OEM solutions that require hardcoded CPU core numbers per device model, AxDragonite reads device topology dynamically at boot:
- Reads `/sys/devices/system/cpu/possible` to determine total online/possible core counts.
- Reads `/sys/devices/system/cpu/cpu*/cpufreq/cpuinfo_max_freq` for every core.
- Groups cores by maximum frequency into clusters:
  - Single cluster
  - Dual cluster: Little, Big
  - Tri cluster: Little, Mid, Prime
  - Quad+ cluster
- Calculates CPU bitmasks and cpuset strings dynamically:
  - `LittleMask`: Lowest frequency core group.
  - `BigMask`: Higher capacity core groups.
  - `PrimeMask`: Highest frequency single or dual core.
  - `BoostMask`: Big + Prime cores combined.
  - `EfficiencyPoolMask`: Cores designated for low-power background execution.
  - `PerformancePoolMask`: Cores designated for latency-critical UI execution.

These masks drive all thread placement and cpuset string generations (`0-3`, `4-7`, etc.) automatically across diverse SoC designs without device-tree patches.

### 2. Dedicated Execution Threads

`AxDragonite` initializes two persistent `HandlerThread` instances within system server:
- `AxDragoniteWorker`: Handles I/O, sysfs writes, process scanning, and binder calls.
- `AxDragoniteTimer`: Handles lease timeout callbacks.

Both threads are elevated to real-time round-robin scheduling (`SCHED_RR | SCHED_RESET_ON_FORK`) with priority `1` to prevent system server handler congestion from delaying performance controls.

### 3. Lease-Based Scene Management

Boost requests are tracked as leases (`LeaseRecord`):
- Each acquire operation generates an incremental integer handle.
- A lease associates a `sceneId`, target PID, requesting package, timeout callback, and boosted thread IDs.
- A lease automatically expires after its scene duration via `AxDragoniteTimer`, or can be explicitly ended via `sceneBoostRelease(handle)`.
- When multiple leases are active concurrently, `AxDragonite` reconciles them by applying the highest requested boost level. When a lease releases, it re-evaluates remaining leases; only when zero leases require a resource are CPU frequencies, cpusets, and thread priorities restored to baseline.

### 4. Scene Definitions and Durations

| Scene Name | Scene ID | Default Duration | Boost Level | RenderThread Boost | Kswapd Pinning | Description |
|---|---|---|---|---|---|---|
| `SCENE_AX_APP_START` | 1 | 1200 ms | Heavy | Yes | Yes | Generic app launch fallback |
| `SCENE_FLING` / `SCENE_SCROLL` | 2 | 600 ms / 400 ms | Light | Yes | No | List and window fling/scroll |
| `SCENE_DATA_LOADING` | 3 | 500 ms | Light | Yes | No | Network/disk loading scenes |
| `SCENE_FOLDER_ANIMATION` | 4 | 500 ms | Light | Yes | No | Launcher folder opening/closing |
| `SCENE_DRAG_AND_DROP` | 5 | 500 ms | Light | Yes | No | UI drag interactions |
| `SCENE_AX_NOTIFICATION_EXPAND` | 100 | 600 ms | Light | Yes | No | Status bar shade pulling down |
| `SCENE_AX_UNLOCK` | 101 | 800 ms | Heavy | Yes | Yes | Keyguard unlock sequence |
| `SCENE_AX_SYSTEMUI_ANIMATION` | 102 | 500 ms | Light | Yes | No | Doze, scrim, volume dialog transitions |
| `SCENE_APP_LAUNCH_COLD` | 103 | 1200 ms | Heavy | Yes | Yes | Cold application process launch |
| `SCENE_APP_LAUNCH_WARM` | 104 | 800 ms | Heavy | Yes | No | Warm activity start |
| `SCENE_APP_EXIT_ANIM` | 105 | 400 ms | Light | Yes | No | Returning to home screen |
| `SCENE_ROTATION` | 106 | 600 ms | Light | Yes | No | Screen orientation change |
| `SCENE_CAMERA_OPEN` | 201 | 1500 ms | Heavy | Yes | Yes | Camera subsystem initialization |
| `SCENE_CAMERA_CAPTURE` | 202 | 800 ms | Heavy | No | No | Camera shutter/capture sequence |
| `SCENE_GAME_MODE` | 203 | Persistent | Heavy | Yes | Yes | Active game in foreground |
| `SCENE_BIOMETRIC_UNLOCK` | 301 | 2000 ms / 600 ms | Heavy | Yes | No | Fingerprint / face auth verification |
| `SCENE_RECENT_TASK_SLIDE` | 401 | 400 ms | Light | Yes | No | Overview / Recents carousel swipe |
| `SCENE_QUICK_SWITCH_APP` | 402 | 600 ms | Heavy | Yes | No | Gesture bar quick-switch gesture |

### 5. Hardware & Governor Tuning (`AxPerfEnhancer`)

When a scene is acquired, `AxPerfEnhancer` modifies kernel knobs:

- **CPU Frequency Bounds:**
  - Saves initial `/sys/devices/system/cpu/cpu*/cpufreq/scaling_min_freq`.
  - Light Boost: Raises `scaling_min_freq` to 60% of cluster max frequency.
  - Heavy Boost: Raises `scaling_min_freq` to 85% of cluster max frequency.
  - Restores original minimum frequencies when all leases clear.
- **CFS / Schedtune / Cpuctl:**
  - `top-app` `schedtune.boost`: Set to `20` (Light) or `40` (Heavy).
  - `top-app` `cpu.uclamp.min`: Set to `300` (Light) or `600` (Heavy).
  - `top-app` `cpu.uclamp.latency_sensitive`: Set to `1`.
  - `top-app` `cpu.shares`: Raised to `2048` (Light) or `4096` (Heavy).
  - `background` `cpu.shares`: Throttled down to `128` during Heavy boosts.
- **AOSP PowerManager:** Calls `PowerManagerInternal.setPowerBoost(Boost.INTERACTION, duration)` and sets `Mode.LAUNCH` during heavy boosts.
- **Kswapd Pinning:** Writes the efficiency cluster mask to `/proc/ax_dragonite/kswapd_pin` during cold launches, unlocks, and camera startup to keep memory reclaim routines off big cores.
- **WALT / Task Boost:** Writes target PID to `/proc/ax_dragonite/boost` or `/proc/sys/walt/nt_sched_per_task_boost`.

### 6. Process & HWUI Thread Boosting (`AxUIBooster`)

For scenes with `boostRenderThread = true`:
1. The target process is moved into `/dev/cpuctl/top-app`.
2. AxDragonite inspects `/proc/<pid>/task/` and reads `/proc/<pid>/task/<tid>/comm` to discover:
   - `RenderThread`
   - Worker threads named `hwuiTask*` or `HwuiTask*`
3. Thread promotion:
   - Heavy Boost: Elevates the main thread, RenderThread, and HWUI worker threads to `SCHED_RR | SCHED_RESET_ON_FORK` priority `1`.
   - Light Boost: Elevates threads to `Process.THREAD_PRIORITY_TOP_APP_BOOST` (-10).
   - CPU Affinity: Sets thread affinity strictly to `BoostMask` (big and prime cores).
4. Reference Counting: A process may have multiple overlapping boost triggers. Priority and affinities are restored to `SCHED_OTHER` with all-core affinity only after all boosts for that PID release.

### 7. Named Thread Affinity Feature (`AxNamedThreadAffinityFeature`)

When processes start or are boosted, AxDragonite inspects thread names to enforce cluster affinities:

| Thread Name (`comm`) | Assigned Cluster Mask | Reason |
|---|---|---|
| `RenderThread` | `BoostMask` (Big + Prime) | Frame composition and command buffer submission |
| `CrRendererMain` | `BigMask` (Big cores) | Chromium / WebView main rendering pipeline |
| `UnityMain` | `BoostMask` (Big + Prime) | Unity engine game loop |
| `GLThread` | `BoostMask` (Big + Prime) | OpenGL rendering loops |
| `MainThread` | `BoostMask` (Big + Prime) | App main loop |
| `AudioTrack` | `LittleMask` (Little cores) | Low CPU audio streaming thread |

If supported by the kernel, rules are also written to `/proc/ax_named_thread_affinity/`.

### 8. The `ax_foreground` Cpuset & Self-Contained OOM Routing (`AxOomAdjusterHelper`)

In standard AOSP, processes transition between `top-app`, `foreground`, and `background`. NothingOS added an `nt_foreground` cpuset to isolate background tasks during touch interactions. AxDragonite implements this cleanly through `/dev/cpuset/ax_foreground`:

1. **Cpuset Registration:** `system/core` adds `SP_AX_FOREGROUND = 9` to `sched_policy.h`, creates `/dev/cpuset/ax_foreground` in `init.rc`, and registers `CPUSET_SP_AX_FOREGROUND` in `task_profiles.json`.
2. **OOM Adjuster Routing:** When `OomAdjuster` updates process scheduling groups, `AxOomAdjusterHelper` intercepts apps with `curAdj` between `200` (Foreground Service) and `800` (Cached Activity). If the app is not a system UI, Axion internal app, or media process, it is assigned to `THREAD_GROUP_AX_FOREGROUND`.
3. **Dynamic Throttling on Touch / Mode Shifts:**
   - Normal state: `/dev/cpuset/ax_foreground` is configured with all standard foreground cores.
   - Input Boost / Game Mode / Camera Open: AxDragonite calls `limitAxForeground(true)`. The `ax_foreground`, `background`, and `dex2oat` cpusets are instantly restricted to `EfficiencyPoolMask` (little cores).
   - This immediately evicts non-critical apps and background services from performance cores during touch gestures, game sessions, or camera launches.

### 9. Input Boost (`AxBoostAdjuster`)

Touch and motion events intercepted by `PointerEventDispatcher` and `PhoneWindowManager.interceptMotionBeforeQueueingNonInteractive` call `AxDragonite.inputBoost()`:
- Activates an 800ms input boost window.
- Restricts `ax_foreground`, `background`, and `dex2oat` cpusets to efficiency cores.
- If repeated touch events occur, the timer extends by 800ms without re-triggering redundant filesystem writes.
- When the timer expires, normal cpuset allocations are restored.

### 10. SurfaceFlinger Core Affinity Binding (`frameworks/native`)

SurfaceFlinger exposes a backdoor binder transaction (`code 2007` on `ISurfaceComposer`):
- `SurfaceFlinger::bindSFThread(bool enable, uint32_t cpuset)`:
  - Extracts the SurfaceFlinger main thread TID (`gettid()`) and RenderEngine TID (`getRenderEngine().getRenderEngineTid()`).
  - Uses `sched_setaffinity()` to bind both threads directly to the `BoostMask` when `enable` is true.
  - Reverts to all-core affinity (`0xffffffff`) when disabled.
- Triggered during `SCENE_APP_LAUNCH_COLD`, `SCENE_AX_APP_START`, and `SCENE_FLING`.

### 11. Process Freezing (`AxFreezerController`)

During cold app launches and camera startup, AxDragonite freezes background processes to eliminate CPU contention:
- Detects cgroup v2 support (`/sys/fs/cgroup/cgroup.controllers`).
- Locates background apps under `/sys/fs/cgroup/uid_<uid>/pid_<pid>/`.
- Writes `1` to `cgroup.freeze` to freeze processes during launch.
- Falls back to cgroup v1 freezer (`/sys/fs/cgroup/freezer/freezer.state`) on older kernels.
- Thaws all frozen processes when the launch lease ends or when an app process is killed.

### 12. Audio Ringtone Dex2oat Restriction

In `AudioService`, incoming calls change audio mode to `MODE_RINGTONE`. AxDragonite restricts `dex2oat` compilation cpusets to small cores (`adjustCpusetCpus("dex2oat", null, 0L)`). When the audio mode returns to `MODE_NORMAL`, the restriction is lifted (`duration = -1L`). This prevents background package compilation from causing audio stutter or delay during incoming calls.

### 13. SystemUI Reactive Boost Startables

AxDragonite integrates into SystemUI via Dagger (`AxDragoniteStartablesModule`):

| Startable Class | Monitored Interface | Boost Trigger |
|---|---|---|
| `AxShadeExpansionBoostStartable` | `ShadeExpansionListener`, `ShadeInteractor.qsExpansion` | Triggers `onShadeExpand()` when shade or QS expansion fraction is between 0.0 and 1.0; collapses when settled. |
| `AxKeyguardUnlockBoostStartable` | `KeyguardUnlockAnimationListener`, `KeyguardStateController` | Triggers `onUnlock()` on animation start and `onKeyguardDismiss()` when keyguard starts going away. |
| `AxBiometricAuthBoostStartable` | `BiometricUnlockController.BiometricUnlockEventsListener` | Triggers `onBiometricAuth()` on non-none biometric modes. |
| `AxDozeAnimationBoostStartable` | `StatusBarStateController.StateListener` | Triggers `onDozeTransition()` when doze amount changes between 0.0 and 1.0. |
| `AxLightRevealScrimStartable` | `LightRevealScrimInteractor.revealAmount` | Triggers `onLightReveal()` when reveal amount is active (0.001 to 0.999). |
| `AxVolumeDialogBoostStartable` | `VolumeDialogController.Callbacks` | Triggers `onVolumeDialog()` when volume dialog is shown; ends when dismissed. |
| `AxWakefulnessBoostStartable` | `WakefulnessLifecycle.Observer` | Triggers `onWakeUp()` when device begins waking up. |

In addition, SystemUI components invoke AxDragonite directly:
- `NotificationPanelViewController`: Calls `AxDragonite.onFling()` when closing the shade panel, and `AxDragonite.onFlingEnd()` on finish.
- `NotificationStackScrollLayout`: Calls `AxDragonite.onNotificationStackScroll()` when the stack is dragged, and ends when dragging stops.

---

## Code Implementation Map

| Repository | Path | Role / Functionality |
|---|---|---|
| **system/core** | `libprocessgroup/include/processgroup/sched_policy.h` | Defines `SP_AX_FOREGROUND = 9`. |
| **system/core** | `libprocessgroup/sched_policy.cpp` | Maps `ax_foreground` to `SP_AX_FOREGROUND`, profile `CPUSET_SP_AX_FOREGROUND`. |
| **system/core** | `libprocessgroup/profiles/task_profiles.json` | Cgroup join actions for `ax_foreground`. |
| **system/core** | `rootdir/init.rc` | Creates `/dev/cpuset/ax_foreground` and assigns system permissions. |
| **frameworks/native** | `services/surfaceflinger/SurfaceFlinger.cpp` | Implements `bindSFThread` and backdoor binder transaction `2007`. |
| **frameworks/native** | `services/surfaceflinger/SurfaceFlinger.h` | Declares `bindSFThread`. |
| **frameworks/base** | `core/java/android/app/IActivityManager.aidl` | IPC declarations: `sceneBoostAcquire`, `sceneBoostRelease`, `isSceneIdExist`. |
| **frameworks/base** | `services/core/java/com/android/server/am/ActivityManagerService.java` | Routes `sceneBoostAcquire` / `release` to `AxDragonite.getInstance()`. |
| **frameworks/base** | `core/java/android/os/Process.java` | Adds `THREAD_GROUP_AX_FOREGROUND = 8` and `setThreadAffinity(tid, mask)`. |
| **frameworks/base** | `core/jni/android_util_Process.cpp` | JNI implementation for `android_os_Process_setThreadAffinityCpus`. |
| **frameworks/base** | `services/core/java/com/android/server/am/OomAdjuster.java` | Directs `curAdj` 200..800 processes into `THREAD_GROUP_AX_FOREGROUND`. |
| **frameworks/base** | `services/core/java/com/android/server/am/ProcessList.java` | Hooks for process fork (`onProcessForked`), start (`onProcessStarted`), and death (`onProcessKilled`). |
| **frameworks/base** | `services/core/java/com/android/server/wm/ActivityStarter.java` | Triggers app launch scene boost on activity start. |
| **frameworks/base** | `services/core/java/com/android/server/wm/ActivityRecord.java` | Tracks window drawing, visibility, and warm launch boosts. |
| **frameworks/base** | `services/core/java/com/android/server/wm/DisplayPolicy.java` | Triggers `SCENE_ROTATION` and fling gesture boosts. |
| **frameworks/base** | `services/core/java/com/android/server/wm/DisplayContent.java` | Tracks focused package name for targeted fling boost. |
| **frameworks/base** | `services/core/java/com/android/server/wm/PointerEventDispatcher.java` | Triggers `inputBoost()` on touch motion events. |
| **frameworks/base** | `services/core/java/com/android/server/policy/PhoneWindowManager.java` | Triggers `inputBoost()` on non-interactive motion queueing. |
| **frameworks/base** | `services/core/java/com/android/server/audio/AudioService.java` | Restricts `dex2oat` cpusets during ringtone audio playback. |
| **frameworks/base** | `services/core/java/com/android/server/app/GameManagerService.java` | Toggles `SCENE_GAME_MODE` on foreground game state change. |
| **frameworks/base** | `packages/SystemUI/.../NotificationPanelViewController.java` | Shade fling boost hooks. |
| **frameworks/base** | `packages/SystemUI/.../NotificationStackScrollLayout.java` | Notification stack drag scroll boost hooks. |
| **axion_sdk** | `ax_dragonite/src/com/android/axion/dragonite/AxDragonite.kt` | Kotlin client singleton with helper methods. |
| **axion_sdk** | `ax_dragonite_common/.../AxDragoniteConstants.java` | Scene IDs, opcodes, durations, IPC bundle keys. |
| **axion_sdk** | `ax_dragonite_common/.../AxDragoniteInternal.java` | Internal client helper delegating to `ActivityManager.getService()`. |
| **axion_sdk** | `ax_dragonite_server/.../AxDragonite.java` | Central coordinator singleton, lease tracking, worker/timer threads. |
| **axion_sdk** | `ax_dragonite_server/.../AxCpuClusterManager.java` | Automatic sysfs CPU topology scanner and core mask generator. |
| **axion_sdk** | `ax_dragonite_server/.../AxPerfEnhancer.java` | CPU frequency min bounds, cgroup writes, PowerManager hints. |
| **axion_sdk** | `ax_dragonite_server/.../AxUIBooster.java` | Scans and elevates app UI + HWUI threads to `SCHED_RR`. |
| **axion_sdk** | `ax_dragonite_server/.../AxBoostAdjuster.java` | Animation boost and touch input boost coordinator. |
| **axion_sdk** | `ax_dragonite_server/.../AxNamedThreadAffinityFeature.java` | Thread name pattern matching and CPU affinity assignment. |
| **axion_sdk** | `ax_dragonite_server/.../AxFreezerController.java` | Background cgroup v2 / v1 process freezing during cold launches. |
| **axion_sdk** | `ax_dragonite_server/.../AxOomAdjusterHelper.java` | Evaluates if a process should be assigned to `THREAD_GROUP_AX_FOREGROUND`. |
| **axion_sdk** | `ax_dragonite_server/.../AxActivityCustomizationUtil.java` | Launch and fling duration overrides for specific packages. |
| **axion_sdk** | `ax_dragonite_server/.../AxFrameInsertManager.java` | Toggles `debug.choreographer.skipwarning` during flings. |
| **axion_sdk** | `ax_dragonite_server/.../AxPerfTraceManager.java` | Perfetto / Atrace scene counter and slice tracer. |
| **axion_sdk** | `ax_dragonite_systemui/.../Ax*BoostStartable.kt` | SystemUI Dagger CoreStartable boost modules. |

---

## Verification and Diagnostics

### 1. Dumpsys Inspection

You can view the active subsystem state, detected topology, core masks, and active leases:

```sh
adb shell dumpsys activity service com.android.server.am.ActivityManagerService
# or dump AxDragonite directly via internal interface:
adb shell dumpsys activity
```

The dump displays:
- Core count and cluster count detected from sysfs.
- Hexadecimal bitmasks for Little, Big, Prime, and Boost core sets.
- Current active leases, including handle, scene ID, target PID, and package name.

### 2. Atrace / Perfetto Tracing

AxDragonite records trace markers under `Trace.TRACE_TAG_ACTIVITY_MANAGER`:
- **Slice Tag:** `AxDragonite:AcquireScene_<sceneId>_H<handle>`
- **Counter:** `AxDragonite:ActiveScene` (tracks the current active scene ID, resetting to `0` when inactive)

Capture a trace during an app launch or shade pull:

```sh
adb shell perfetto -o /data/local/tmp/trace.perfetto -t 5s am sched freq
```

In the resulting trace:
1. `AxDragoniteWorker` and `AxDragoniteTimer` handler executions show up at `SCHED_RR` priority 1.
2. The target app's main thread and `RenderThread` switch to `SCHED_RR` during launch.
3. CPU frequency scaling curves show clusters jump to the target minimum boost floors.
