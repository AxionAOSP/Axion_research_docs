# AxBoostFwk / AxBurstEngine Research

## Problem

Devices running vanilla AOSP firmware (MTK and Qualcomm) suffer poor UI performance because OEM BSP boost framework patches are absent. AOSP development communities reuse Pixel's `libperfmgr` to compensate, and some maintainers apply kernel/device-tree optimizations. This helps Qualcomm devices but remains insufficient for MTK chipsets.

### Known AOSP Performance Issues

1. **Janky QS Panel expansion** when the notification shade is full or a media player is showing
   - Root cause: massive layout operations and high bitmap usage during expansion, with total CPU usage reaching ~40%
2. **Janky activity/fragment transitions** on Qualcomm, extremely laggy on MTK
   - Root cause: kernel scheduling differences; Qualcomm handles bursty transitions better than MTK. Midrange MTK devices show severe lag during activity transitions on AOSP baseline.

---

## Research Findings

Perfetto analysis revealed scheduling shortcomings caused by the absence of BSP boost framework patches.

### Animation Thread Pinning

Critical UI components get pinned to small/efficiency cores during animations. EAS (Energy-Aware Scheduling) is animation-unaware and prefers efficient cores over performance cores because the instantaneous load/utilization signal is too low.

### Generic Power Hints

AOSP `PowerManager` hint calls are generic and do not cover MTK/QCOM-specific boost points. `libperfmgr` relies on all hints plus native/AIDL/HAL calls, which is insufficient for MTK. Qualcomm devices run acceptably without BSP patches but still underperform compared to stock firmware.

---

## Solution: AxBoostFwk

A generic performance engine that brings near-stock firmware performance to AOSP devices, avoiding the performance degradation that occurs when BSP patches are absent. The engine is modeled on QCOM/MTK `BoostFramework` and MTK's PowerHAL strategies.

---

## Solution: Janky QS Panel

### Problem

The default AOSP QS Panel expansion is smooth until the user has many notifications or a media player showing.

### OEM Strategies Observed

OEMs manipulate SystemUI scheduling to real-time priority during shade animations:
- Transsion/Vivo: `SCHED_FIFO`
- Nothing: `SCHED_RR`

We focused on NothingOS strategies due to how close NothingOS is to AOSP.

### NothingOS QS Shade Strategy

NothingOS elevates SystemUI and Launcher to a restricted cpuset during shade expansion:

```
QS Shade expands:
  1. Boost active
  2. SystemUI elevated to RT scheduling (SCHED_RR)
  3. Restricted cpuset util clamp raised to max
  4. Top-app cpuset restricted to efficient cores
  5. Restricted cpuset restricted to performance cores

QS Shade collapses:
  1. Boost deactivates
  2. Scheduling restored to SCHED_OTHER
  3. Restricted cpuset util clamp reduced to 0
  4. Restricted cpuset widened to all cores
```

#### NothingOS Reference: NtBoostAdjusterImpl

From `nt-services/sources/com/nothing/server/performance/NtBoostAdjusterImpl.java`:

**RT SCHED_RR elevation for animation threads** (lines 1413-1451):
```java
// Boost thread to SCHED_RR priority 1
Process.setThreadScheduler(tid, 1073741826, 1);  // magic value = 0x40000002 = SCHED_RR

// Restore to SCHED_OTHER
Process.setThreadScheduler(tid, 0, 0);
Process.setThreadPriority(originalPriority);
```

**Animation boost elevates both UI thread and RenderThread** (lines 1783-1828):
```java
void animationBoost(int pid, int renderTid, long duration) {
    Process.setThreadScheduler(pid, 1073741826, 1);    // UI thread -> SCHED_RR
    Process.setThreadScheduler(renderTid, 1073741826, 1); // RenderThread -> SCHED_RR
    // auto-restore after duration if duration > 0
}
```

**Restricted cpuset + uclamp for animations** (lines 1283-1319):
```java
void a0(int pid, int enable) {
    FileUtils.stringToFile("/dev/cpuctl/restricted/cgroup.procs", pid);
    FileUtils.stringToFile("/dev/cpuctl/restricted/cpu.uclamp.min", "100");
    FileUtils.stringToFile("/dev/cpuctl/restricted/cpu.uclamp.max", "100");
    adjustCpusetCpus("/dev/cpuset/restricted/cpus", "6-7", 0L);  // big cores only
}
```

#### NothingOS Reference: NTCpuBindController

From `nt-systemui/sources/com/nothing/systemui/util/NTCpuBindController.java`:

SystemUI-side boost client with per-animation-type bitmask flags (lines 21-40):
```java
static final int REQUEST_ANIMATION_BOOST_TYPE_FLING_NOTIFICATION_PANEL_VIEW = 1;
static final int REQUEST_ANIMATION_BOOST_TYPE_TRACKING_NOTIFICATION_PANEL_VIEW = 2;
static final int REQUEST_ANIMATION_BOOST_TYPE_SPEED_UP_NOTIFICATION_PANEL_VIEW_EXPAND = 4;
static final int REQUEST_ANIMATION_BOOST_TYPE_UNLOCK = 8;
static final int REQUEST_ANIMATION_BOOST_TYPE_SPEED_UP_QS_EXPANSION_ANIMATION = 64;
static final int REQUEST_ANIMATION_BOOST_TYPE_SWIPE_DOWN_NOTIFICATION_ANIMATION = 128;
```

CPU limiting during shade expansion (lines 139-196):
```java
void setLimitForegroundAppCpu(boolean limit) {
    // limits top-app cpuset to 0-4 during expansion
}
void setLimitOtherProcessCpu(boolean limit) {
    // limits camera-daemon and dex2oat to 0-1
}
```

### AxionOS QS Shade Solution

We implemented an identical strategy to NothingOS for QS panel jank, which worked on Google Pixels and Nothing Phone (2a).

**Implementation flow:**

```
StatusBarManagerService.onPanelRevealed(numItems)
  -> AxUiFirstManager.onPanelRevealed(items)
    -> SvpResourceManager.lock()
      -> writes /dev/cpuset/top-app/cpus = little cores
      -> writes /dev/cpuset/foreground/cpus = little cores
    -> AxUiFirstSceneManager.setSvpContext(true)
    -> force-reconcile top app and launcher
      -> top app gets THREAD_GROUP_SVP (big cores)
      -> launcher gets THREAD_GROUP_SVP (big cores)
    -> AxUiFirstThreadOps.applyCriticalRoles()
      -> main thread: uclamp min=384, max=1024
      -> render thread: uclamp min=512, max=1024, SCHED_FIFO/RR
      -> HWUI threads: uclamp min=256, max=800

StatusBarManagerService.onPanelHidden()
  -> AxUiFirstManager.onPanelHidden()
    -> SvpResourceManager.unlock()
      -> restores /dev/cpuset/top-app/cpus = all cores
      -> restores /dev/cpuset/foreground/cpus = all cores
    -> AxUiFirstSceneManager.setSvpContext(false)
    -> reset all thread roles to SCHED_OTHER
```

**Key files:**
- `StatusBarManagerService.java:1904`: `onPanelRevealed()` hook
- `StatusBarManagerService.java:1934`: `onPanelHidden()` hook
- `AxUiFirstManager.java:411`: `onPanelRevealed()` implementation
- `AxUiFirstManager.java:427`: `onPanelHidden()` implementation
- `SvpResourceManager.java:114-123`: cpuset lock/unlock

---

## Solution: Activity Transition Lag

Root cause: parsing animation XML via `AnimationUtils` is a major contributor to transition jank. Programmatic animation construction almost eliminated jank on Nothing Phone (2a). BSP-style activity launch boosts further improved the output.

### AxionOS Activity Transition Boost Flow

```
ActivityManagerService starts process
  -> procStart handler thread elevated to SCHED_FIFO priority 1 + THREAD_GROUP_SVP
    (AMS.java:2528-2529)

Choreographer.doFrame() fires in app process
  -> AxBoostFwk.acquireHint(OP_LAUNCHER_ICON_LAUNCH, duration)
    -> binder to ActivityManagerService
      -> AxBoostManager.acquireHint(opcode, durMs)
        -> native_perf_hint() via JNI
          -> C++ AxPerformance hint system
```

**Key files:**
- `ActivityManagerService.java:2528-2529`: procStart SCHED_FIFO + SVP
- `AxBoostFwk.java:108`: client-side `acquireHint()`
- `AxBoostManager.java:53`: server-side `acquireHint()`

---

## Solution: SurfaceFlinger CPU Policy (SfCpuPolicy)

SfCpuPolicy is an AxionOS-specific SurfaceFlinger scheduling optimizer that dynamically adjusts uclamp floors, CPU affinity, and task profiles for SF's critical threads.

### Architecture

```
SfCpuPolicy (namespace: android::scheduler)
  |
  |-- onFrameStart(timestamp, vsyncPeriod)
  |     -> updates Hz bucket (60/90/120)
  |     -> computeUclampMin() -> per-Hz uclamp floor
  |     -> computeAffinityGroup() -> big/small/prime/all
  |     -> writeUclamp() for main, HWC, RE threads
  |     -> SetSingleThreadAffinity() for placement
  |     -> SetTaskProfiles() -> SvpPolicy or ProcessCapacityMax
  |
  |-- onFrameEnd(timestamp)
  |     -> detects heavy frames (duration > vsyncPeriod * earlyHeavy)
  |     -> sets sEarlyFrameBoost -> forces big cores + raised uclamp
  |
  |-- onSpeedUpRE(tid)
  |     -> pins RenderEngine to big cores + uclamp boost
  |
  |-- notifyHwcHwbinderTid()
  |     -> captures HWC hwbinder TID, applies uclamp
  |
  |-- onPerformanceMode(enabled)
  |     -> toggles big-core affinity
  |
  |-- onScreenRecording(recording)
  |     -> prime-core affinity + uclamp boost
```

### Key Code: Uclamp Computation

From `frameworks/native/services/surfaceflinger/Scheduler/SfCpuPolicy.cpp:253-285`:

```cpp
unsigned int computeUclampMin() {
    // Per-Hz uclamp floors
    // 60Hz -> 82, 90Hz -> 102, 120Hz -> 106
    unsigned int base = sUclampMinForHz[sHzBucket];
    // Clamp to [uclampLower=106, uclampUpper=344]
    base = std::clamp(base, sUclampLower, sUclampUpper);
    // Boost for screen recording
    if (sScreenRecording) base = std::max(base, sBoostSr);
    // Boost for early frame detection
    if (sEarlyFrameBoost) base = std::max(base, sBoostEarly);
    return sSuspended ? 0 : base;
}
```

### Key Code: Affinity Computation

From `SfCpuPolicy.cpp:287-306`:

```cpp
int computeAffinityGroup() {
    if (sSuspended)           return kAffinityGroupSmall;
    if (sScreenRecording)     return kAffinityGroupPrime;
    if (sPerformanceMode)     return kAffinityGroupBig;
    if (sEarlyFrameBoost)     return kAffinityGroupBig;
    if (sHzBucket >= 90)      return kAffinityGroupBig;
    return kAffinityGroupAll;
}
```

### Key Code: Uclamp Write via sched_setattr

From `SfCpuPolicy.cpp:396-418`:

```cpp
void writeUclampForThread(int tid, unsigned int uclampMin) {
    sched_attr attr = {};
    attr.size = sizeof(attr);
    attr.sched_flags = (SCHED_FLAG_KEEP_ALL | SCHED_FLAG_UTIL_CLAMP_MIN);
    attr.sched_util_min = uclampMin;
    syscall(__NR_sched_setattr, tid, &attr, 0);
}
```

### Call Sites in SurfaceFlinger

| Location | Call | Trigger |
|---|---|---|
| `main_surfaceflinger.cpp:93` | `registerMainThread()` | Process startup |
| `SurfaceFlinger.cpp:2876` | `onFrameStart(...)` | Every commit cycle start |
| `SurfaceFlinger.cpp:3041` | `onScreenRecording(...)` | Recording layers detected |
| `SurfaceFlinger.cpp:3044` | `onVpLpEnable(...)` | Refresh rate <= 30fps |
| `SurfaceFlinger.cpp:3464` | `onFrameEnd(...)` | After composite completes |
| `SurfaceFlinger.cpp:6181-6182` | `onPowerSuspend/Foreground` | Display power mode change |
| `Scheduler/Scheduler.cpp:804` | `onRefreshRateChanged(...)` | Refresh rate change |

### SfCpuPolicy Properties

All under `persist.sys.sf.cpupolicy.*`:

| Property | Default | Purpose |
|---|---|---|
| `min_60` | 82 | Uclamp floor at 60Hz |
| `min_90` | 102 | Uclamp floor at 90Hz |
| `min_120` | 106 | Uclamp floor at 120Hz |
| `lowbound_uclamp_min` | 106 | Uclamp lower bound |
| `upbound_uclamp_min` | 344 | Uclamp upper bound |
| `boost_early` | -1 | Uclamp boost on early frame detection |
| `boost_sr` | -1 | Uclamp boost during screen recording |
| `early_heavy` | 2 | Heavy frame multiplier (x vsync period) |

---

## Solution: SVP (Super Visual Process) Scheduling

SVP is an AxionOS-custom scheduling tier that sits between `TOP_APP` and `SYSTEMUI` in priority. It provides real-time scheduling with maximum utilization clamping for latency-critical threads.

### SVP Cgroup Configuration

From `system/core/rootdir/init.rc`:

```
# cpuset: big + prime cores (configurable)
/dev/cpuset/svp/cpus = ${persist.sys.axion_cpu_svp:-0-7}

# cpuctl: maximum CFS priority
/dev/cpuctl/svp/cpu.shares = 20480          # 20x default 1024
/dev/cpuctl/svp/cpu.uclamp.min = 100
/dev/cpuctl/svp/cpu.uclamp.max = 100
/dev/cpuctl/svp/cpu.uclamp.latency_sensitive = 1

# schedtune: maximum boost
/dev/stune/svp/schedtune.boost = 100
/dev/stune/svp/schedtune.prefer_idle = 1
```

### SvpPolicy Task Profile

From `system/core/libprocessgroup/profiles/task_profiles.json`:

The `SvpPolicy` composite profile joins a thread to three cgroup controllers simultaneously:
- `cpuset/svp`: CPU affinity to big/prime cores
- `cpu/svp` (cpuctl): CFS scheduling with uclamp pinned to 100
- `schedtune/svp`: schedtune boost 100 + prefer_idle
- Also sets `SCHED_RR` priority 1 and timer slack 50000ns

### Who Gets SVP

| Component | File:Line | Mechanism |
|---|---|---|
| SurfaceFlinger main thread | `SurfaceFlinger.cpp:984` | `SetTaskProfiles(0, {"SvpPolicy"})` at startup |
| SurfaceFlinger (dynamic) | `SurfaceFlinger.cpp:10465-10491` | `sfBindControll()` via binder code 1048 |
| EventThread (VSync dispatch) | `Scheduler/EventThread.cpp:337-346` | SvpPolicy + SCHED_FIFO priority 2 |
| BackgroundExecutor (high-pri) | `BackgroundExecutor.cpp:35-47` | SvpPolicy + affinity to core 0 |
| AMS procStart handler | `ActivityManagerService.java:2528-2529` | SCHED_FIFO priority 1 + THREAD_GROUP_SVP |
| Perf-list apps (top app) | `AxUiFirstManager.java:774-779` | `topGroupForPackage()` returns SVP |

### SVP in AxUiFirstConfig

From `frameworks/base/services/core/java/com/android/server/uifirst/AxUiFirstConfig.java:62-63`:

```java
static final int EXCLUSIVE_BIG_CORE_GROUPS =
    (1 << Process.THREAD_GROUP_SVP) | (1 << Process.THREAD_GROUP_SYSTEMUI);
```

SVP and SystemUI are the only groups that get exclusive big-core access. When either group is active, `AxUiFirstThreadOps.nonExclusiveGroup()` redirects other threads away from big cores.

### SVP in AxBurstEngine

From `frameworks/base/services/core/java/com/android/server/am/AxBurstEngine.java:125,129`:

```java
void writeDefaultCpusets(DeviceData data) {
    native_set_boost_data(
        new String[]{ CPU_SVP, CPU_SYSTEMUI },
        new String[]{ data.svpCpus, data.svpCpus }
    );
}
```

SVP and SystemUI share the same CPU mask (big + prime cores), written via `native_set_boost_data()` JNI call.

---

## Solution: AxUiFirst UI Thread Scheduling Engine

AxUiFirst is the AxionOS thread scheduling engine that manages per-thread uclamp, cpuset placement, RT scheduling elevation, and frame rescue for UI-critical processes.

### Architecture

```
Client Process (app)                    System Server
─────────────────                       ─────────────
UiFirstScenarioManager                  AxExtServiceFactory
  │ dispatch(Scenario)                    │ getUiFirstManager()
  ├─ AxUiFirstManager (client)            ├─ AxUiFirstManager (server)
  │   onSceneStart/End()                  │   adjustTopApp / adjustUxProcess
  │   processFrameHint()                  │   onPanelRevealed/Hidden
  │                                        │   registerUiFirstThread
  ├─ AxBoostFwk                           │   evaluateFrameRescue
  │   acquireHint()                        │   boostThread / boostScene
  │   rescueFrame()                        │
  │                                        ├─ AxUiFirstThreadOps
  ├─ AxUiFirstThreads                     │   applyUclamp() -> Process.setThreadUtilClamp()
  │   register(role) -> AM binder           │   moveToBoostGroup() -> Process.setThreadGroupAndCpuset()
  │                                        │   applyCriticalRoles()
  └─ HandlerThread                        │   resetThreadRole() -> scheduleAsRegularPriority()
      auto-register via getUiFirstRole()   │
                                           ├─ RenderThreadManager
                                           │   applyTopRuntimePolicy() -> FIFO/RR scheduling
                                           │
                                           ├─ ThreadBoostManager
                                           │   boost(tid) -> scheduleAsRoundRobinPriority()
                                           │
                                           ├─ FrameBoostManager
                                           │   evaluateRescue() -> rescue levels
                                           │
                                           ├─ SvpResourceManager
                                           │   lock/unlock -> cpuset restrictions
                                           │
                                           ├─ AxUiFirstThreadContext
                                           │   boost/demote system threads by scene
                                           │
                                           ├─ AxBurstEngine
                                           │   acquireHint() -> AxBoostManager
                                           │   rescueFrame() -> IAxUiFirstManager
                                           │
                                           └─ AxBoostManager
                                               native_perf_hint() -> JNI
                                               native_set_boost_data() -> cpuset config
                                               JNI: com_android_server_am_AxPerformance.cpp
```

### Thread Roles and Uclamp Defaults

From `AxUiFirstConfig.java:40-47`:

| Role | Uclamp Min | Uclamp Max | Early Wakeup Min |
|---|---|---|---|
| UI (main thread) | 384 | 1024 | 640 |
| RENDER (RenderThread) | 512 | 1024 | 768 |
| GL (GLThread) | 384 | 900 | -- |
| HWUI_TASK (HWUI workers) | 256 | 800 | 384 |

### Per-Thread Uclamp Application

From `AxUiFirstThreadOps.java:35-52`:

```java
static void applyUclamp(int tid, int min, int max) {
    // Check XML overrides first
    int[] override = AxPerfConfig.getUxThreadUclamp(roleName);
    if (override != null) { min = override[0]; max = override[1]; }
    // Cache to avoid redundant syscalls
    if (sAppliedUclamp.get(tid) == pack(min, max)) return;
    Process.setThreadUtilClamp(tid, min, max);
    sAppliedUclamp.put(tid, pack(min, max));
}
```

### RT Scheduling Elevation

From `RenderThreadManager.java:207-233`:

```java
void applyRuntimeScheduling(int pid, int renderTid, int mode) {
    switch (mode) {
        case FIFO:
            ActivityManagerService.scheduleAsFifoPriority(renderTid, true);
            break;
        case RR:
            ActivityManagerService.scheduleAsRoundRobinPriority(renderTid, true);
            break;
        case NONE:
            ActivityManagerService.scheduleAsRegularPriority(renderTid, true);
            break;
    }
}

int topSchedulingMode(int pid, int renderTid) {
    if (bgMgr.useUiBoost()) return FIFO;  // CPU load high
    if (bgMgr.getUiRtLevel() == RR) return RR;
    return NONE;
}
```

### Frame Rescue

From `FrameBoostManager.java:181`:

| Rescue Level | Threshold | Duration Range |
|---|---|---|
| LIGHT | frame > 5ms | 16-32ms |
| HEAVY | frame > 11ms | 32-64ms |
| CROSS | frame > 20ms | 64-80ms |

Consecutive-miss dampening: max 3 consecutive misses before suppressing same-level rescue.

Flow:
```
Choreographer.doFrame()
  -> AxBoostFwk.onFrameRealDraw(elapsedNs)
    -> AxBoostManager.onFrameRealDraw(elapsedNs)
      -> AxUiFirstManager.evaluateFrameRescue(pid, actualDurationNs)
        -> FrameBoostManager.evaluateRescue()
          -> returns LIGHT/HEAVY/CROSS
      -> AxBoostManager.acquireHint(opcode, duration)
        -> native_perf_hint() via JNI
```

### CPU Load Monitoring

From `AxUiFirstManager.java:885-918`:

```
CpuLoadMonitor reads CpuMonitorInternal
  >= 70% availability -> tier 0 (low load)
  >= 40% availability -> tier 1 (medium)
  < 40% availability  -> tier 2 (high)

Tier propagated to:
  AxBackgroundManager.onCpuLoadTierChanged(tier)
    -> useUiBoost()     = tier >= 1
    -> getUiRtLevel()   = tier >= 2 ? FIFO : (tier >= 1 ? RR : NONE)
```

### System Thread Boost/Demote

From `AxUiFirstThreadContext.java`:

| State | Threads | Action |
|---|---|---|
| BOOST | UI, ANIM, DISPLAY, FG | uclamp boost (min 640 for UI/ANIM when CPU tier >= 1) |
| DEMOTE | IO, SUPPORT | moved to BACKGROUND cpuset, uclamp dampened to [0, 256] |
| BASELINE | all | restored to FOREGROUND cpuset, uclamp [0, 1024] |

---

## Solution: AxBurstEngine / AxBoostManager

### AxBurstEngine

From `frameworks/base/services/core/java/com/android/server/am/AxBurstEngine.java`:

Central orchestrator that wires together DeviceData, BoostSettingsRepository, SurfaceFlinger core control, and the hint system.

**Key flows:**
```
systemReady()
  -> creates DeviceData (reads CPU topology, builds cpuset masks)
  -> creates BoostSettingsRepository
  -> binds SurfaceFlinger core control (transaction 1048)
  -> writes default cpusets via native_set_boost_data()

acquireHint(opcode, durMs)
  -> delegates to AxBoostManager.acquireHint()

rescueFrame(pid, actualDurationNs)
  -> IAxUiFirstManager.evaluateFrameRescue()
  -> IAxUiFirstManager.getRescueDurationMs()
  -> maps rescue level to opcode (LIGHT/HEAVY/CROSS)
  -> fires hint
```

### AxBoostManager

From `frameworks/base/services/core/java/com/android/server/am/AxBoostManager.java`:

**Native JNI methods** (backed by `com_android_server_am_AxPerformance.cpp:1416-1418`):

| Method | Purpose |
|---|---|
| `native_perf_hint(opcode, durMs)` | Core perf hint acquisition |
| `native_perf_hint_rel(handle)` | Release hint |
| `native_update_hint_target(pid, renderTid, protectedTarget)` | Set hint target process |
| `native_set_boost_data(paths[], values[])` | Write cpuset/perf config |
| `native_set_thermal_ceiling/remove_thermal_ceiling` | Thermal throttling |
| `native_set_cpu_freq_bound/remove_cpu_freq_bounds` | CPU frequency bounds |

### AxBoostFwk Client API

From `frameworks/base/core/java/android/app/AxBoostFwk.java`:

50+ opcodes covering all boost scenarios:

| Category | Opcodes |
|---|---|
| Launch | `OP_LAUNCHER_ICON_LAUNCH`, `OP_LAUNCHER_DRAWER_LAUNCH` |
| Scroll/Fling | `OP_SCROLL_VERTICAL`, `OP_FLING_VERTICAL`, `OP_SCROLL_HORIZONTAL` |
| Touch | `OP_TOUCH_DOWN`, `OP_TOUCH_UP`, `OP_DRAG_START` |
| Frame | `OP_FRAME_PREFETCHER`, `OP_FRAME_INPUT_END`, `OP_FRAME_PRE_ANIM`, `OP_FRAME_DRAW_STEP`, `OP_FRAME_VSYNC` |
| Render | `OP_BOOST_RENDERTHREAD`, `OP_BOOST_RENDER_THREAD` |
| Shade | `OP_SHADE`, `OP_SHADE_COLLAPSE` |
| System | `OP_SYSTEM_GAME`, `OP_PACKAGE_INSTALL_BOOST`, `OP_UI_BOOST` |

### NothingOS Dragonite Reference

From `nt-services/sources/com/nothing/server/performance/Dragonite.java` (4392 lines):

Nothing's scene-based performance engine. Key patterns:

**RT elevation for animation threads including HWUI workers** (lines 2845-2887):
```java
void D1(int pid, int renderTid, boolean boost) {
    Process.setThreadScheduler(pid, 1073741826, 1);       // UI thread -> SCHED_RR
    Process.setThreadScheduler(renderTid, 1073741826, 1);  // RenderThread -> SCHED_RR
    for (int hwuiTid : hwuiTids) {
        Process.setThreadScheduler(hwuiTid, 1073741826, 1); // each HWUI worker -> SCHED_RR
    }
    // on restore: UI/RT get priority -10 (high nice), HWUI get -2
}
```

**Dragonite's own handler threads run at SCHED_RR** (lines 4207-4208):
```java
Process.setThreadScheduler(dragoniteThread.getThreadId(), 1073741826, 1);
Process.setThreadScheduler(dragoniteManagerThread.getThreadId(), 1073741826, 1);
```

**Process freezing during animations** (lines 1453-1549):
```java
// Freezes background processes (adj >= 250, < 900) during animations
// Uses Process.setProcessFrozen() to free CPU for foreground
```

### NothingOS MediaTek PowerHAL Resources

From `framework/sources/com/mediatek/powerhalmgr/PowerHalMgr.java`:

| Resource | Value | Purpose |
|---|---|---|
| `PERF_RES_SCHED_UCLAMP_MIN_TA` | 21005056 | Top-app uclamp min |
| `PERF_RES_SCHED_UCLAMP_MIN_RT` | 21005312 | RT uclamp min |
| `PERF_RES_SCHED_PREFER_CPU_FG` | 21184768 | Foreground CPU preference |
| `PERF_RES_SCHED_PREFER_IDLE_RT` | 20988928 | RT prefer idle |
| `PERF_RES_CPUFREQ_PERF_MODE` | 4276224 | CPU performance mode |

---

## Results: UI Performance

AxBoostFwk fixed poor UI performance on Play Store and Instagram. The improvement is not a silver bullet, but users can now scroll smoothly across different apps.

---

## Problems: Engine Stability

Due to limited test devices, the engine may produce bad outputs on other devices. Some devices get wrong configurations or even freeze due to bad scheduling. We are working to address these issues and need more logs/perfetto/bugreports from the community.

## Problems: Ax Perf Configs

Not all maintainers/builders may want to include AxBurstEngine perf configs out of the box due to complexity. In future updates we plan to make the engine perf config setup easier to maintain by introducing common perf XML configs.

## Problems: AxPerformance Hint Handling Overheads

We are still figuring out the proper design for the hint system. The native C++ implementation was chosen for performance, as Java-side overhead is quite high.

---

## Appendix: Cpuset/Cpuctl Hierarchy

From `system/core/rootdir/init.rc`:

```
/dev/stune/top-app          -> max CPU boost, all cores
/dev/stune/foreground       -> high priority
/dev/stune/background       -> little cores only
/dev/stune/system-background -> system tasks on little cores
/dev/stune/systemui         -> dedicated for SystemUI (uclamp.max=100, uclamp.min=100)
/dev/stune/l-background     -> lowest priority, uclamp.max=25%
/dev/stune/rt, svp, audio-app, dex2oat, nnapi-hal, camera-daemon
```

## Appendix: ax_process_utils

From `axion_sdk/ax_process_utils/src/ax_process_utils.cpp`:

Provides `SetSingleThreadAffinity(tid, group)` and `SetThreadAffinity(tid, group)` via `sched_setaffinity()`.

CPU topology groups:
- **Small** (group 1): `persist.sys.axion_cpu_small` or default `0,1,2,3`
- **Big** (group 0): `persist.sys.axion_cpu_big` or capacity-derived or `4,5,6,7`
- **Prime** (group 4): `persist.sys.axion_cpu_prime` or highest-capacity core(s)
- **All** (group 2): all online CPUs
- **Balanced** (group 3): half small + half big

## Appendix: NothingOS Custom Cpuset Groups

Nothing adds two custom cpuset groups not in standard AOSP:
- `/dev/cpuset/nt_foreground/`: restricted to `0-2` (little cores) during input boost, expanded to `0-7` otherwise
- `/dev/cpuset/ntcam-algo/`: camera algorithm processing group

## Appendix: Key System Properties

| Property | Source | Purpose |
|---|---|---|
| `persist.sys.axion_cpu_svp` | AxionOS | SVP cpuset mask |
| `persist.sys.axion_cpu_big/small/prime` | AxionOS | CPU topology |
| `persist.sys.sf.cpupolicy.*` | AxionOS | SfCpuPolicy tuning |
| `persist.sys.boost_adjuster.enable` | NothingOS | NtBoostAdjuster enable |
| `persist.sys.animationboost_uclamp_min` | NothingOS | Animation uclamp min |
| `persist.dragonite.debug` | NothingOS | Dragonite debug |
