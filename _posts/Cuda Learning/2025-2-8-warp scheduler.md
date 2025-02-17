---
layout: post
title: Warp Scheduler
tags: [cuda learning]
---

参考资料：

[Introduction to NVIDIA Nsight Compute - A CUDA Kernel Profiler](https://www.youtube.com/watch?v=nYSdsJE2zMs)

[A CUDA Kernel Profiler](https://developer.download.nvidia.com/video/gputechconf/gtc/2019/presentation/s9345-cuda-kernel-profiling-using-nvidia-nsight-compute.pdf)

[【CUDA调优指南】Warp Scheduler讲解](https://www.bilibili.com/video/BV1EhrTYcEdf?spm_id_from=333.788.videopod.episodes&vd_source=d2914dc08c654d5cdb584d78d1806154)

## warp scheduler定义
Warp Scheduler 负责调度和管理 warp（线程束）在 SM (Streaming Multiprocessor) 上的执行。

以Volta架构为例，每个SM有4个Warp Scheduler，每个Warp Scheduler对应一个Warp Pool。不同架构中Warp Pool的大小是不同的，Volta中每个Warp Scheduler负责16个warp slot，而Turing中每个Warp Scheduler负责8个warp slot。
但是每个cycle只能最多只能一个warp被issue。

![image](https://github.com/user-attachments/assets/95e8151b-5d79-4e9a-9120-80aefebf35de)

## warp的状态

根据自身的执行状态和是否被 warp scheduler 选取，可以将 warp 分为 2 类：unused warp和active warp。其中active warp又可细分为三类。

* unused warp：没有被 allocate 到 warp slot 中，依然在 warp pool 中等待的 warp；
* active warp：被 allocate 到 warp slot 中维护的 warp，stalled warp，eligible warp 和 selected warp 的总和；
  * stalled warp：因为一些原因（指令没有加载完成、依赖项没有得到等）而等待的 warp；
  * eligible warp：准备好被派发指令的 warp；
  * selected warp：被 warp scheduler 选中派发指令的 warp。
 
## warp scheduler执行流程

1. 初始状态：5 个 active warp，其中 2 个 eligible warp，3 个 stalled warp；

![image](https://github.com/user-attachments/assets/0d7fdff7-216f-4964-be44-a80d886f0255)

2. 最简单的Scheduler策略，就是随机选取一个eligible warp进行issue，此时选取了slot3

![image](https://github.com/user-attachments/assets/e2025160-cfd4-4df0-abc4-ef9f8458a125)

3. 周期 N + 1，选取slot4进行issue，同时slot3因为在周期N被issue了，且其指令至少需要数个周期，因此此时其状态变为stalled

![image](https://github.com/user-attachments/assets/bd399d00-82ab-4d77-b342-e33f247f9c92)

4. 周期 N + 2：warp4 指令执行完成后退出，剩余所有的 warp 都为 stalled warp，该周期没有可以选取派发指令的 warp

![image](https://github.com/user-attachments/assets/bb6fbcf7-803d-4b97-9402-5980d64432cf)

5. 周期 N + 3：2 个新的 warp 从 warp pool 中被 allocate 到 warp slot 中，都为 stalled warp；warp1 和 warp2 的状态变为 selected warp，选取 warp2 派发指令。

![image](https://github.com/user-attachments/assets/b4b12924-7909-4a8c-a8c8-21e558b13f8c)

## Nsight Compute中的warp statics

如下图所示，在这4个周期中，我们可以得到下面的指标

* cycles_active：4（周期数量）
* warps_active：20（所有有颜色的方块数量）
* warps_active / cycles_active：5（平均每个周期有颜色的方块数量）
* achieved_occupancy：5 / 8 = 62.5%（平均每个周期有颜色的方块数量 / Warp Slot 的数量）
* warps_stalled：15（深绿色方块数量）
* warps_eligible：5（浅绿色方块数量）
* warps_issued：3（浅绿色斜线方块数量）
* warps_issued / cycles_active：0.75（有 warp 可以选取派发指令的周期比例）
* issue_slot_utilization：75%（同上）

![image](https://github.com/user-attachments/assets/7e750e6c-920a-4e69-a82a-8c68506144a5)



