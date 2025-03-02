---
layout: post
title: Memory Barrier笔记 --- Cache and MESI Protocol
tags: [Memory Barrier]
---

前人对Memory Barrier的总结已经很完善了，我在这边只是记录下自己对其的理解，并无新意，只是作为个人的笔记而已。想要详细了解的话推荐阅读下面这篇译文，如果英语好的话可以直接看原文。

[【译】内存屏障：软件黑客的硬件视角](https://kongjun18.github.io/posts/memory-barriers-a-hardware-view-for-software-hackers/)

## 1、缓存结构(Cache Structure)

现代CPU的运行速度远快于内存的存取速度，现代CPU会利用缓存来加快这一步骤。CPU的缓存通常可以在几个Cycle内完成访问。

![image](https://github.com/user-attachments/assets/ae2c9ec3-cc02-4db3-9f4b-3109eee79cce)


* cache line <br>
CPU cache 会以固定的字节长度访问内存，从 16 字节到 256 字节不等。

* cache miss (startup cache miss or warmup cache miss)<br>
CPU第一次访问cache的时候，由于cache都是空的，导致`cache miss`。cache miss导致CPU必须直接访问内存，引起性能下降。

* capacity miss <br>
一段时间后，CPU cache被填满，后续未命中需要从 cache 中淘汰一个数据项以便为新读取的数据项腾出位置。

* associativity miss <br>
当我们访问一个已经被淘汰的缓存行的时候，所导致的缓存未命中

* write miss <br>
当CPU需要将数据写入的时候，其必须保证当前数据只能有当前CPU所有，其它CPU的缓存中没有其副本。如果数据项在 CPU 的 cache 中是只读的，这种情况称为 “write miss”。

* communication miss <br>
当其他CPU尝试访问该数据项时，将会发生缓存未命中。此次未命中是由于第一个个CPU为执行写操作已将其他CPU对该数据项置为失效状态所致。此类缓存未命中被定义为"通信未命中"，其根源在于多个CPU对于共享数据项需要保持一致性。


## 2、 缓存一致性协议(MESI协议）

### 2.1、MESI States
缓存一致性协议通过管理缓存行状态来确保数据一致性，防止数据不一致或丢失。
此类协议可能包含数十种复杂状态，但针对基础原理分析需求，我们仅需关注四状态MESI（Modified/Exclusive/Shared/Invalid）缓存一致性协议的状态机模型。

Cache line通过额外维护一个两bit的状态标识（存储于元数据区），用于精确跟踪每个缓存行的协同状态。

| States  | Explaination |
| --- | --- |
| Modified   | 1、对应着CPU最新的store操作，且该缓存行数据保证不存在于其他CPU缓存中。<br>2、Modified的缓存行要么最终会被写回内存，要么通过缓存一致性协议转交其他缓存<br>3、Modified的缓存行被替换之前，必须先完成2的步骤
| Exclusive   | 1、Exclusive的缓存行未被修改，因此内存中的数据是最新的，而修改状态下缓存行的数据已经被修改，内存中的数据可能过时。<br>2、CPU可以随时修改Exclusive的缓存行数据而无需与其他CPU协商。<br>3、当需要替换该缓存行时，可以直接丢弃而不必写回内存。 |
| Shared |1、Shared状态的缓存行可能被其他CPU缓存所持有，因此当前CPU在执行存储操作前必须与其他缓存执行一致性协商<br>2、和Exclusive类似，内存中的数据是最新的 <br>3、Shared状态的缓存行可直接清除，而无需将其写回内存或通过缓存一致性协议将数据转移到另外的CPU |
| Invalid | 1、Invalid状态的缓存行是空的，不包含数据<br>2、当新数据进入缓存时，系统会优先选择处于invalid状态的缓存行，因为替换其他状态的缓存行可能导致未来的缓存未命中，带来性能损失  |



缓存一致性协议通过内置的消息协调机制，确保缓存行在CPU间的状态迁移符合全局一致性要求，确保所有的CPU的缓存视图是一致的。


### 2.2、MESI Protocol Messages

在共享总线架构的多核系统中，以下消息类型即可实现完整的缓存一致性协议。

| Message Type | Explaination|
|---|---|   
| **Read**  | 包含了想要读取的缓存行的物理地址 | 
| **Read Response**  | 1、该message包含了之前Read数据请求的数据<br>2、该message的来源是内存或者其他CPU的缓存，例如其它CPU的某个modified状态的缓存行含有所请求的数据，那么该缓存会返回这个消息
| **Invalidate** | 包含了需要被remove的cacheline的地址，如果其它CPU缓存拥有该缓存行，即清除本地对应副本并发送确认响应，确保全局缓存一致|
| **Invalidate Acknowledge** | 当CPU接收到Invalidate请求后，必须从自己的缓存中移除指定数据，然后发送确认响应。|
| **Read Invalidate**| 1、该message除了包含想要读取的缓存行的物理地址，同时强制要求其他缓存立即移除对应数据副本<br>2、该message要求接收方同时提供**Read Response**（携带最新数据）和一组**Invalidate Acknowledge**（来自所有受影响缓存）作为响应|
|**Write Back**  |1、message中含地址和数据，用于将处于`modified`状态的缓存行写回内存<br>2、允许缓存按需逐出处于"已修改(Modified)"状态的缓存行，从而释放存储空间以容纳新数据 |


### 2.3、MESI State Diagram

cache line的状态转换如下图所示，下面将会逐一说明。


![image](https://github.com/user-attachments/assets/cacbc2ab-9937-4fb8-b924-c771574d38a5)



| transaction | explaination |
|---|---|
| **a** | 将缓存行数据回写至主存，但CPU仍保留该缓存行副本并保留后续修改权限。此转换需**Write Back**消息。 |
| **b** | 对处于**Exclusive**状态的缓存行执行写操作。此转换属于本地原子操作，无需任何协议消息交互。 |
| **c** | 当处于**Modified**状态的缓存行接收到**Read Invalidate**请求时，其必须执行本地副本无效化操作，并同步发送**Read Response**（携带最新数据）及**Invalidate Acknowledge**，完成数据转移与状态更新。 |
| **d** | CPU对未缓存的数据执行原子性`read modify-write`操作。通过发送**Read Invalidate**发起协议流程，经由**Read Response**获取数据。该转换需在收到所有的**Invalidate Acknowledge**后完成状态更新。 |
| **e**  |CPU对本地缓存中处于只读状态的数据执行原子性`read modify-write`操作。其必须广播**Invalidate**消息，并在收集全部**Invalidate Acknowledge**后方可完成状态改变。 |
| **f**  | 当其他CPU发起缓存行读取请求，而当前CPU缓存正好持有该数据，且其处于**Modified**状态，当前CPU将会提供数据响应，并可能伴随主存回写操作。该转换由接收**Read**触发，通过发送携带目标数据的**Read Response**完成协议交互。 |
| **g** | 其他CPU发起缓存行读取请求，而当前CPU缓存正好持有该数据，且其处于**Exclusive**状态，那么发起请求的CPU其数据可能从当前CPU缓存获取或直接从内存读取。但是无论数据来源如何，当前CPU将保留只读副本。该转换由接收**Read**消息触发，通过**Read Response**完成数据交付。 |
| **h** | 当前CPU对缓存行执行写操作的时候，将会广播**Invalidate**消息。其必须接收到所有的**Invalidate Acknowledge**消息或者其他CPU通过**Writeback**消息逐出该缓存行，使当前CPU成为最后一个持有者，其完成该状态转换 |
| **i** | 当其他CPU执行原子`read modify-write`操作，其所请求的数据正好由当前CPU的缓存唯一持有，即**Excludsive**状态，因此必须使其无效。此转换由接收**Read Invalidate**消息启动，本CPU需同步发送**Read Response**（提供数据副本）与**Invalidate Acknowledge**完成协议交互。|
| **j** | 当CPU要存储(Store)不在其缓存中的数据项时，其发送**Read Invalidate**消息，并等待响应。其必须同时收到**Read Response**（获取数据）和所有的**Invalidate Acknowledge**的消息之后，方可完成初始状态转换。待实际的写操作完成后，缓存行将通过转换(b)进入**Modified**状态。|
| **k** | CPU加载(Load)不在其缓存中的数据项时，其通过发送**Read**消息启动流程，在收到对应的**Read Response**后完成数据加载(Load)，并完成状态转换。|
| **l** | 当其他CPU对多核持有的缓存行执行写(Store)操作时，本CPU接收**Invalidate**请求消息后，立即清除本地只读副本，并返回**Invalidate Acknowledge**| 


### 2.4、MESI Protocol Example

![image](https://github.com/user-attachments/assets/f61e4f14-db32-464d-8837-239bf46af535)

**缓存行数据流转全流程分析**  
**初始状态**：地址0x00数据驻留主存，总共由4个CPU，所有CPU缓存行处于"无效(Invalid)"状态，内存数据标记为有效(V)。  

| 操作序列 | 执行CPU | 操作类型           | CPU0缓存状态 | CPU1缓存状态 | CPU2缓存状态 | CPU3缓存状态 | 内存状态(V/I) |  
|----------|--------|--------------------|--------------|--------------|--------------|--------------|--------------|  
| 1        | CPU0   | 加载地址0x00       | 0x00(S)      | -            | -            | -            | V            |  
| 2        | CPU3   | 加载地址0x00       | 0x00(S)      | -            | -            | 0x00(S)      | V            |  
| 3        | CPU0   | 加载地址0x08       | 0x08(S)      | -            | -            | 0x00(S)      | V            |  
| 4        | CPU2   | 预载地址0x00(RDINV)| 0x08(S)      | -            | 0x00(E)      | -            | V            |  
| 5        | CPU2   | 写入地址0x00       | 0x08(S)      | -            | 0x00(M)      | -            | I            |  
| 6        | CPU1   | 原子递增地址0x00   | 0x08(S)      | 0x00(M)      | -            | -            | I            |  
| 7        | CPU1   | 加载地址0x08       | 0x08(S)      | 0x08(S)      | -            | -            | 0x00(V)      |  

**关键流程解析**：  
1. **初始加载阶段**（操作1-2）  
   - CPU0通过"读请求"获取0x00数据，状态转为Shared  
   - CPU3同样加载该地址，形成多CPU共享状态，内存数据保持有效  

2. **缓存替换阶段**（操作3）  
   - CPU0加载新地址0x08触发缓存替换机制，通过invalidate操作移除0x00数据  
   - 此时CPU3仍持有0x00的共享副本，内存数据还是有效的

3. **独占获取阶段**（操作4）  
   - CPU2写操作需求，发送"Read Invalidate"  
   - 触发CPU3缓存无效化，CPU2获得独占(E)状态，内存数据仍有效  

4. **修改状态跃迁**（操作5）  
   - CPU2执行写入操作，状态升级为已修改(M)  
   - 内存数据标记为无效(I)，标识缓存副本为唯一权威数据源  

5. **原子操作影响**（操作6）  
   - CPU1通过"读无效化"消息嗅探到CPU2的修改数据  
   - 完成原子递增后持有修改(M)状态，内存数据仍处于过期状态  

6. **数据回写阶段**（操作7）  
   - CPU1加载新地址0x08触发回写机制  
   - 通过"回写消息"将0x00数据持久化至内存，恢复内存有效性(V)  

**最终系统状态**：  
- CPU0、CPU1持有地址0x08的共享副本  
- 地址0x00数据已回写至内存，各CPU缓存中无有效副本  
- 内存数据恢复一致性，验证了MESI协议在复杂操作场景下的可靠性  
