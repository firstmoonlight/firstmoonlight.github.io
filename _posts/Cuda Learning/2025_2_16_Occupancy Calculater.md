---
layout: post
title: Occupancy Calculater
tags: [cuda learning]
---

参考资料：

[【CUDA调优指南】Occupancy Calculater](https://www.bilibili.com/video/BV1WNrHY5EeP?spm_id_from=333.788.videopod.episodes&vd_source=d2914dc08c654d5cdb584d78d1806154)

[Achieved Occupancy](https://docs.nvidia.com/gameworks/content/developertools/desktop/analysis/report/cudaexperiments/kernellevel/achievedoccupancy.htm)

## Occupancy的定义

Occupancy定义为SM上active warp数量与硬件支持的最大active warp数量的比值。该值会随着线程束的启动与结束动态变化，且不同SM的占用率可能各不相同。

例如：一个SM理论上支持M个active warp，实际上由于各种原因只能支持N个，这里的`N/M`就是occupancy。

低Occupancy会导致指令发射效率低下，因为缺乏足够的可用线程束来隐藏依赖指令间的执行延迟。当Occupancy达到足以隐藏延迟后，继续提升反而可能因线程资源竞争导致性能下降。内核性能分析的首要步骤应是检查占用率指标，并通过调整不同占用率水平观察其对内核执行时间的影响。

### Active Warp
* 在任何给定的时间点，GPU的SM可以同时处理多个Warp。但是由于资源限制（如执行单元，寄存器，共享内存等），并不是所有的Warp能够同时活跃。
* Active Warp指的是那些正在执行指令的warp。它们可能处于不同的执行阶段，比如正在执行计算任务、正在等待内存访问结果或正在执行控制流指令。

## Theoretical Occupancy
active warp的数量及其对应的occupancy存在理论上限，该上限由内核启动配置、编译选项及设备计算能力共同决定。

![image](https://github.com/user-attachments/assets/3c4b3147-8230-4f7a-8718-b24ba645d896)

我们以ada架构为例，如上图所示，一个SM中有128个core，所以至少可以放下128个thread（4个warp）。为了隐藏延时，其实可以放进去更多的warp（大于4个），因为一些warp在等待的时候，可以执行其它的warp，以确保计算单元是一直处于工作状态。
但是这个warp的数量还是有个理论上线，其由下面这些因素影响：

1. 用户设置的Block的大小会影响实际上SM可以容纳的warp的数量

2. 每个SM中的寄存器的数量是由上限的，因此每个thread使用的寄存器的数量会影响warp的数量

3. 每个SM只有固定大小的共享内存，因此每个thread分配的共享内存的数量也会影响warp的数量

### Occupancy Calculater

Nsight Compute提供了Occupancy Calculater工具来计算理论上的active warp的数量。

如下图所示，这是我们在Nsight Compute上所启动的Occupancy Calculater工具的界面，通过设置Thread Per Block, Register Per Thread, User Shared Memory Per Block，这个工具可以计算得到理论上的Occupancy。

其中Occupancy Data是我们计算得到的结果，而Allocated Resource栏目和Occupancy Limiters则是我们计算的中间结果。Physical Limit of GPU是我们设备的物理限制。

![image](https://github.com/user-attachments/assets/31bb724a-8089-44a4-a647-0e91f0af7964)

#### block size对SM可启动warp数量的影响
下图是Physical Limit of GPU栏目中和block size相关的限制：

1. 每个Block必须分配到同一个SM中

2. 每个SM最多可以放48个warp（1536个thread）

3. 每个SM最多可以放24个Block

![image](https://github.com/user-attachments/assets/d5b32a7b-002a-4980-81c2-d895bac601b8)

假设每个thread只使用很少的share memory和register，即我们只考虑block size的影响，此时设置block size为 (32, 5) 即160个thread。
* 根据2可得，一个SM最多可以放置1536个thread，即9.6个block，(1536 / 160 = 9.6)，
* 根据1可得，0.6个block是不能分配到一个block上的，因此一个SM最多只有9个block，即45个warp，所以occupancy为0.9375=45/48.

#### register对SM可启动warp数量的影响
下图是Physical Limit of GPU栏目中和register相关的限制：

1. 每个SM有65536个寄存器
2. 寄存器的分配单元大小是256个，即一次分配最少256个寄存器
3. 寄存器的分割粒度是warp
4. warp的分配粒度是4（如果warp Allocation Granularity为4，那么即使一个kernel配置只需要3个warp的寄存器资源，硬件也会按照4个warp来分配资源

![image](https://github.com/user-attachments/assets/82436555-dc25-4569-9ddc-461f8e5b7c61)

假设需要计算的数据足够多，每个线程需要51 register，设置的block size为128

* 根据3，每个thread 51 reg，所以每个warp需要51 * 32 = 1632个reg
* 根据2，1632个reg，需要按照256粒度分割，1632 / 256 = 6.375 -》 7 * 256 = 1792，所以每个warp实际需要1792个reg
* 根据1，65536个reg，每个warp需要1792，所以65536 / 1792 = 36.57 -》 36个warp
* occupancy=36/48=0.75

#### share memory对SM可启动warp数量的影响
下图是Physical Limit of GPU栏目中和share memory相关的限制：

1. 每个SM最多有16384B的share memory
2. share memory的分配单元是128B
3. 每个SM必须分配1024B share memory
4. share memory配置为32768B

![image](https://github.com/user-attachments/assets/7c5629e5-7350-4c5e-9b4f-6255393c4d21)

![image](https://github.com/user-attachments/assets/371775c2-c33d-402f-b14d-5e5f4499213c)

假设需要计算的数据足够多，每个线程需要5000B share memory，设置的block size为128

* 根据3，每个block需要1024 + 5000 = 6024B share memory
* 根据2，6024B需要按照128B粒度分配，6028 / 128 = 47.06 -> 48 * 128 = 6144，所以每个block实际需要6144B
* 根据4，一共32768B，  32768 / 6144 = 5.33，所以只能分配5.33个block
* occupancy = (5 * 3)/48 = 0.4166

![image](https://github.com/user-attachments/assets/972fb020-fffc-4792-818a-1918e9c4ba34)






