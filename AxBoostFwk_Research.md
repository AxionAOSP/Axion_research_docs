### AxBoostFwk/AxBurstEngine research

## Problem
Devices like mtk/qcom suffers poor performance when using vanilla AOSP firmware. This results to some devices loosing their BSP boost framework patches and optimizations.
AOSP development communities were reusing pixel's libperfmgr to improve the performance on AOSP, some AOSP developers/maintainers do kernel/device tree optimizations as well, 
this improves performance on QCOM devices but is still not enought on chipsets like mtk.

Known problems on AOSP:
1. janky QS Panel expansion if notification shade is full of notifications or a media player is showing 
 - caused by the design of the qs panel expansion itself - layout operations during expansion were massive and high bitmap usages also contributes to the issue total cpu usage can lead up to %40
2. activity/fragment transitions were janky on QCOM but extremely laggy on MTK devices
 - this may be caused by kernel optimizations, QCOM seems to handle it better than mtk - we see very little jank on AOSP baseline tests, while midrange MTKs suffer on extremely laggy activity transition

## Research
Our research shows the following statistics:

during our perfetto analysis, we found out shortcomings that occurs because the BSP boost framework patches does not exist on AOSP. 

# animations
we often see critical components getting pinned to small cores during animations which contributes to jank, 
the reason this happens was because EAS is not aware of animations and prefers efficient cores 
over performance cores because the load/utilization is not high enough.

# performance
power manager hint calls on AOSP were generic and does not cover cases well like MTK/QCOM boost points,
libperfmgr basically relies on all of the hints plus the native cpp calls or aidl/hal calls, 
which where still not enough especially for MTKs, QCOM can be fine on most cases without the BSP patches and run smoothly 
but not the same performance compared to the stock firmware
  
## Our Solution: AxBoostFwk
to fix the following problems, we decided to introduce a generic performance engine that will help devices get near stock firmware performance and avoid performance degradation while using AOSP.
this engine is based on QCOM/MTKs BoostFramework and MTK's power hal strategies.

# Solution: Janky QS Panel 
The default QS Panel expansion on AOSP was smooth until user have tons of notifications and media player showing, what we found out on our research:
OEMs tends to manipulate the priority of the SystemUI to Real time scheduling, transsion/vivo uses SCHED_FIFO, nothing uses SCHED_RR, they do these on animations as well.
We focused on NothingOS strategies due to how close NothingOS is to AOSP.

NothingOS strategy:

Scheduling - SystemUI and Launcher were intercepted to go to restricted cpuset instead of top-app

QS Shade expands -> boost active -> elevated to RT scheduling (SCHED_RR) -> raises restricted cpuset util clamp to max -> restricts top-app cpuset to efficient cores | restrict restricted cpuset to performance cores 
QS Shade collapse - boost deactivates - scheduling restored to SCHED_OTHER -> reduces restricted cpuset util clamp to 0 -> widens restricted cpuset to all cores

We are using identical strategy to fix the qs panel janked which worked for google pixels and nothing 2a.

# Solution: Activity transition lag
we found out that parsing animations xml via AnimationUtils contributes to the issue, doing things programticaly greatly - almost removed the janks on nothing 2a, 
the addition of the BSP activity launch boosts also improved the output

# results for ui performance
The AxBoostFwk was able to fix poor ui performance on playstore, instagram, while the output is not revolutionary, users can now scroll smoothly across different apps.

# problems: engine stability
due to limited test devices, the engine may produce bad outputs on others devices, some devices were getting wrong configuration and even freezing due to bad scheduling, we are still working on how to address 
these issues and hope for more logs/perfetto/bugreports from the community.

# problems: ax perf configs
not all maintainers/builder may want to include axburstengine perf configs out of the box due to complexity, 
in future updates we plan to make the engine perf configs setup more easier to maintain by introducing common perf xml configs

# problesm: AxPerformance hint handling overheads
we are still figuring our the proper design for the hint system, but it was written in c++ for performance and due to the overhead in java is quite high.
