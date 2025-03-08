---
layout: post
title: Memory Barrier笔记 --- Cache and MESI Protocol
tags: [Memory Barrier]
---

尽管前述的缓存结构在重复读写时具有良好性能，但对于首次写入某个缓存行时其性能较差。以图4为例，当CPU0要写入一个被CPU1缓存的缓存行时，必须等待该缓存行到达，导致CPU0长时间停滞。 但是我们其实并没有必要让CPU0长时间停滞，因为它无论如何都会无条件覆盖该缓存行的数据。
![[Pasted image 20250303132430.png]]

## 1、**Store Buffer**

如下图所示，我们通过在Cache和CPU之间引入一层Store Buffer来进行这个优化。当CPU0写入的时候，它直接写入到Store Buffer中，然后再继续执行，而无需等待CPU1的Invalidate Acknowledge消息。当CPU1的消息返回之后，再将Store Buffer中的数据写入到缓存中。

但是增加一个Store Buffer虽然提升了速度，但是引入了两个新的复杂问题，需要我们在后面两节中确认。
![[Pasted image 20250303132734.png]]

## 2、**Store Forwarding**

如果简单的应用上述的`Store Buffer`模型来执行下面的代码，可能会触发assert，详情如下：

```
a = 1; 
b = a + 1;
assert(b == 2);
```

| Step | CPU0 Operation                             | CPU1 Operation                                 |
| ---- | ------------------------------------------ | ---------------------------------------------- |
| 0    | 执行指令`a=1`                                  |                                                |
| 1    | 查询变量`a`的缓存行，发现未命中（Cache Miss）              |                                                |
| 2    | 发送**Read Invalidate**消息，要求获取`a`所在缓存行的独占所有权 |                                                |
| 3    | 将`a=1`写入存**Store Buffer**，继续执行后续指令         |                                                |
| 4    |                                            | 接收**Read Invalidate**消息，返回`a`的缓存行（值仍为0，并移除本地副本 |
| 5    | 开始执行`b=a+1`                                |                                                |
| 6    | 收到来自CPU1的缓存行（此时`a`的值仍为0）                   |                                                |
| 7    | 从新缓存行加载`a`，获得旧值0                           |                                                |
| 8    | 将**Store Buffer**中的`a=1`提交到缓存行，更新`a`为1     |                                                |
| 9    | 基于旧值0计算`b=0+1=1`，将结果写入`b`的缓存行              |                                                |
| 10   | 执行`assert(b==2)`失败（实际值1≠预期值2）              |                                                |

其原因在于，当CPU0写操作`a=1`被暂存于`Store Buffer`，未即时更新到Cache，导致后续读操作`a+1`直接从缓存加载旧值，造成数据不一致（具体看6，7，8的步骤）。

解决方法：为了解决这个问题，硬件工程师引入"Store Fowarding"机制，即CPU在执行Load操作时会检查自己的**Store Buffer**，从而避免一些反直觉的问题。

![[Pasted image 20250304151004.png]]

## 3、**Store Buffers and Memory Barriers**

### 3.1 、**Store Buffer Issue**

我们需要处理的第二个由`Store Buffer`引入的问题涉及到了数据在多个CPU上的共享。如下图所示。
- CPU0执行`foo()`：`a = 1; b = 1;`
- CPU1执行`bar()`：`while (b == 0); assert(a == 1);`
- 初始状态：
	* 变量`a`初始值0，仅存在于CPU1缓存（状态：Modified）  
	- 变量`b`初始值0，由CPU0独占（状态：Exclusive）  

```c++
// CPU0
void foo(void) 
{ 
	a = 1; 
	b = 1; 
} 

// CPU1
void bar(void) 
{ 
	while (b == 0) continue; 
	assert(a == 1); 
}
```

**CPU0 和 CPU1 可能按照下面的顺序执行，导致assert failed.**

| Step  | CPU0 Operation               | CPU1 Operation                                      | Caheline State           | MESI Message              |     |
| ----- | ---------------------------- | --------------------------------------------------- | ------------------------ | ------------------------- | --- |
| **1** | 执行`a=1`，缓存未命中                | -                                                   | CPU0将`a=1`写入Store Buffer | CPU0发送`Read Invalidate`请求 |     |
| **2** | -                            | 执行`while(b==0)`，（`load b`，但是由于b在初始时刻由CP0单独持有），缓存未命中 | -                        | CPU1发送`Read`请求获取`b`       |     |
| **3** | 执行`b=1`（Store b，由于独占拥有，直接写入） |                                                     | CPU0缓存`b`状态：Modified     | -                         |     |
| **4** | 接收到CPU1的`Read`请求             | -                                                   | CPU0缓存`b`状态：Shared       | CPU0返回`b=1`的响应            |     |
| **5** | -                            | 接收`b=1`，退出循环                                        | CPU1缓存`b`状态：Shared       | -                         |     |
| **6** | -                            | 执行`assert(a==1)`（加载旧值0）                             | CPU1缓存`a`仍为旧值0           | CPU1断言失败                  |     |
| **7** | 接收`Read Invalidate`响应        | 接收`Read Invalidate`请求                               | CPU1缓存`a`失效              | CPU1返回`a=0`的响应            |     |
| **8** | 将Store Buffer的`a=1`提交到缓存     | -                                                   | CPU0缓存`a`状态：Modified     | -                         |     |

从代码的原意看，CPU0 和 CPU1之间的执行应该是由顺序的：`a = 1; -> b = 1; -> jump out while (b == 0) -> assert(a == 1); `，且CPU1可以看到CPU0对a和b的改动。

其问题的根源在于，**CPU0在`Store a`的时候，并没有stall等待CPU1的Invalidate Ack消息，而是继续执行`Store b`**。导致CPU 1 load b 的时候，CPU0 返回的**Read Response**消息，先于CPU 0的**Read Invalidate**的消息到达，导致CPU1 跳出循环，并直接读取其自己缓存中的a。

### 3.2 、**Store Buffer With Memory Barrier**

硬件无法感知到上述的语义，即a和b之间的依赖关系，因此硬件通过暴露给软件**memory barrier**指令，由软件从更高的层次告知硬件数据的依赖关系。

#### 代码
```c++
// CPU0
void foo(void) 
{ 
	a = 1; 
	smp_mb();
	b = 1; 
} 

// CPU1
void bar(void) 
{ 
	while (b == 0) continue; 
	assert(a == 1); 
}
```

* `smp_mb()`
	* CPU**要么简单地暂停执行，直到 Store Buffer 被清空后再继续**；
	* CPU**要么利用 Store Buffer 暂存后续的存储操作，直到缓冲区中所有先前的存储操作都已提交**。

#### 模拟流程

我们通过第二种方式，来模拟上述加入`smp_mb()`的流程

**初始条件**：  
- 变量`a`初始值0，仅存在于CPU1缓存（状态：Modified）  
- 变量`b`初始值0，由CPU0独占（状态：Exclusive）  
- CPU0执行`a=1; smp_mb(); b=1;`  
- CPU1执行`while(b==0); assert(a==1);`  

| Step   | CPU0 Operation                   | CPU1 Operation         | CacheLine State              | MESI Messag               |     |
| ------ | -------------------------------- | ---------------------- | ---------------------------- | ------------------------- | --- |
| **1**  | 执行`a=1`，缓存未命中                    | -                      | CPU0将`a=1`写入Store Buffer     | CPU0发送`Read Invalidate`请求 |     |
| **2**  | -                                | 执行`while(b==0)`，缓存未命中  | -                            | CPU1发送`Read`请求获取`b`       |     |
| **3**  | 执行`smp_mb()`，标记`a=1`             | -                      | Store Buffer标记`a=1`          | -                         |     |
| **4**  | **执行`b=1`，缓存命中但Store Buffer有标记** | -                      | **`b=1`写入Store Buffer（未标记）** | -                         |     |
| **5**  | 接收CPU1的`Read`请求                  | -                      | CPU0缓存`b`降级为Shared           | CPU0返回`b=0`的响应            |     |
| **6**  | -                                | 接收`b=0`，继续循环           | CPU1缓存`b`状态：Shared           | -                         |     |
| **7**  | -                                | 再次执行`while(b==0)`      | -                            | -                         |     |
| **8**  | 接收`Read Invalidate`响应            | 接收`Read Invalidate`请求  | CPU1缓存`a`失效                  | CPU1返回`a=0`的响应            |     |
| **9**  | **将`a=1`提交到缓存（状态：Modified）**     | -                      | CPU0缓存`a`状态：Modified         | -                         |     |
| **10** | **检测到`a=1`已提交，准备提交`b=1`**        | -                      | CPU0缓存`b`状态：Shared           | CPU0发送`Invalidate`请求      |     |
| **11** | -                                | 接收`Invalidate`请求       | CPU1缓存`b`失效                  | CPU1返回`Invalidate Ack`    |     |
| **12** | -                                | 执行`while(b==0)`，缓存未命中  | -                            | CPU1发送`Read`请求获取`b`       |     |
| **13** | 接收`Invalidate Ack`               | -                      | CPU0缓存`b`状态：Exclusive        | -                         |     |
| **14** | 将`b=1`提交到缓存（状态：Modified）         | -                      | CPU0缓存`b`状态：Modified         | -                         |     |
| **15** | 接收CPU1的`Read`请求                  | -                      | CPU0缓存`b`降级为Shared           | CPU0返回`b=1`的响应            |     |
| **16** | -                                | 接收`b=1`，退出循环           | CPU1缓存`b`状态：Shared           | -                         |     |
| **17** | -                                | 执行`assert(a==1)`，缓存未命中 | -                            | CPU1发送`Read`请求获取`a`       |     |
| **18** | 接收`Read`请求，返回`a=1`               | -                      | CPU0缓存`a`降级为Shared           | CPU0返回`a=1`的响应            |     |
| **19** | -                                | 接收`a=1`，断言成功           | CPU1缓存`a`状态：Shared           | -                         |     |

我们关注Step4, Step9, Step10.
* Step 4 ： 我们执行Store b的时候，由于Memory Barrier的左右，`b = 1`不再直接写入缓存中，而是写入`Store Buffer`中，如果`Store Buffer`满了的话，会Stall，直到`Store Buffer`中的数据写入到Cache中。
* Step 9 和 Step 10 ： `b = 1`只有在`a = 1`被清楚Store Buffer之后，才会继续，保证了b和a之间的依赖关系。
