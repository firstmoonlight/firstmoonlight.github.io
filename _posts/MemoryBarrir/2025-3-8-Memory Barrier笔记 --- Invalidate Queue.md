---
layout: post
title: Memory Barrier笔记 --- Cache and MESI Protocol
tags: [Memory Barrier]
---

由于Store Buffer很小，随着大量的Store操作，Store Buffer会很快被填满，导致其后续的Store操作不得不等待，直到其他CPU中的Invalidate操作完成，并返回`Invcalidate Ack`消息。

为了提升性能， 一些CPU引入了`Invalidate Queue`来专门处理`Invalidate`消息。


## **1、Invalidate Queue**

**Invalidate Acknowledge耗时长的原因**

* CPU首先必须确保相应的缓存行确实被无效化，但是如果缓存处于繁忙状态，那么这个无效化的操作可能会延迟
* 如果一个CPU短时间短时间内接收到了大量的**Invalidate**消息，那么该CPU会花费大量的时间来处理这些消息，这导致了其他对应的CPU都会停滞等待

**Invalidate Queue解决的问题**

* Invalidate Queue基于这样的一个现实，即CPU在返回ACK的时候，并不需要真正的将对应的Cache Line invalidate，它可以等到后续需要对该Cache Line的进行读写的时候再执行Invalidate操作。这有点像内存的[延迟分配](https://www.cnblogs.com/whiteBear/p/16729327.html)。
* CPU接收到Invalidate请求之后，会将该Invalidate消息push到`Invalidate Queue`中，并立即返回`Invalidate Acknowledge`消息。

![[Pasted image 20250305175847.png]]
## **2、Invalidate Queues and Memory Barriers**

### **2.1、Invalidate Queue引入的风险**

和 Store Buffer 一样， Invalidate Queue的引入同样在多CPU架构下的数据可见性问题。

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

**初始状态**

* 变量"a"和"b"的初始值均为0，且满足以下条件：
- `a`存在于在CPU0缓存和CPU1缓存中，为只读状态（Shared状态）
 * `b`由CPU 0独占（`Exclusive`或`Modified`状态）

**程序执行流程**

我们按操作顺序分步骤展示各CPU行为与缓存状态变化：

| **Step** | CPU0 Operation                | CPU1 Operation           | CacheLine State                         | MESI Message                       | 结果/问题                    |
| -------- | ----------------------------- | ------------------------ | --------------------------------------- | ---------------------------------- | ------------------------ |
| 1        | 执行`a=1`，缓存行状态为只读(Shared)      | -                        | CPU0将`a=1`写入Store Buffer                | CPU0发送**Invalidate消息**要求CPU1的`a`失效 | `a=1`暂存未提交               |
| 2        | -                             | 执行`while(b==0)`，`b`缓存未命中 | -                                       | CPU1发送**Read消息**请求`b`的缓存行          | CPU1进入循环等待               |
| 3        | -                             | 接收Invalidate消息，入队后立即响应   | CPU1的`a`缓存行标记为待失效（仍在队列未处理）              | CPU1返回**Invalidate Ack**           | CPU0收到响应可继续执行            |
| 4        | 执行`smp_mb()`，强制Store Buffer刷新 | -                        | CPU0的`a=1`提交到缓存行，状态转为Modified           | -                                  | `a=1`全局可见性仍受限于CPU1的无效化队列 |
| 5        | 执行`b=1`，缓存行已独占                | -                        | CPU0直接写入`b=1`，状态保持Modified              | -                                  | `b=1`立即可见性仅限本地           |
| 6        | 接收CPU1的Read请求                 | -                        | CPU0的`b`缓存行降级为Shared                    | CPU0发送`b=1`的**Read Response**      | CPU1将获取`b=1`             |
| 7        | -                             | 接收`b=1`的缓存行              | CPU1缓存`b=1`（状态Shared）                   | -                                  | CPU1退出循环                 |
| 8        | -                             | 执行`assert(a==1)`         | CPU1从本地缓存读取旧值`a=0`（Invalidate Queue未处理） | -                                  | **断言失败**                 |
| 9        | -                             | 后台处理Invalidate Queue     | CPU1的`a`缓存行失效                           | -                                  | 后续读取`a`将触发缓存未命中          |
| 10       | -                             | 重新读取`a`时发送Read请求         | CPU0返回`a=1`的缓存行                         | CPU1获取新值`a=1`（但断言已失败）              | **数据最终一致，但程序已错误终止**      |

我们重点关注Step 3，4，8，9。

- 🔴 Step 8：因`Invalidate Queue`延迟处理导致断言失败  
- 🟡 Step 3/9：`Invalidate Queue`的异步处理引发可见性延迟  
- 🟢 Step 4：内存屏障仅保证本地提交，不解决远端可见性  


### **2.2、Invalidate Queue下的Memory Barrier**

**Memory Barrier**
* 当CPU执行内存屏障时，它会标记当前**Invalidate Queue**中的所有条目，并强制该CPU后续的所有加载操作等待，直到所有标记的条目都已应用到缓存中。

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
	smp_mb();
	assert(a == 1);
}
```


**程序执行流程**

同样的我们按操作顺序分步骤展示各CPU行为与缓存状态变化：

| Step      | CPU0 Operation                                          | CPU1 Operation                               | Cache Line State               | MESI Message                         | 结果/影响                             |
| --------- | ------------------------------------------------------- | -------------------------------------------- | ------------------------------ | ------------------------------------ | --------------------------------- |
| **1**     | 执行`a=1`，但`a`的缓存行处于只读（Shared）状态                          | -                                            | CPU0将`a=1`写入**Store Buffer**   | CPU0发送**Invalidate消息**要求CPU1的`a`缓存失效 | `a=1`暂存未全局可见                      |
| **2**     | -                                                       | 执行`while(b==0)`，`b`缓存未命中                     | -                              | CPU1发送**Read消息**请求`b`的缓存行            | CPU1进入循环等待                        |
| **3**     | -                                                       | 接收Invalidate消息，将其加入**Invalidate Queue**并立即响应 | CPU1的`a`缓存行标记为待失效（但尚未处理）       | CPU1返回**Invalidate Ack**             | CPU0可继续执行，但CPU1的`a`仍为旧值           |
| **4**     | 收到Invalidate Ack后，执行`smp_mb()`，将`a=1`从Store Buffer提交到缓存 | -                                            | CPU0的`a`缓存行状态转为**Modified**    | -                                    | `a=1`对本地可见，但CPU1仍持有旧值             |
| **5**     | 执行`b=1`，`b`缓存行已独占（Exclusive/Modified）                   | -                                            | CPU0直接更新`b=1`，状态保持**Modified** | -                                    | `b=1`立即可见性仅限本地                    |
| **6**     | 收到CPU1的Read请求                                           | -                                            | CPU0的`b`缓存行降级为**Shared**       | CPU0发送`b=1`的**Read Response**        | CPU1将获取`b=1`                      |
| **7**     | -                                                       | 接收`b=1`的缓存行并加载                               | CPU1缓存`b=1`（状态**Shared**）      | -                                    | CPU1退出循环                          |
| ==**8**== | ==-==                                                   | ==执行**内存屏障（如smp_mb()）**==                    | ==-==                          | ==-==                                | ==CPU1暂停执行，等待处理Invalidate Queue== |
| **9**     | -                                                       | 处理Invalidate Queue中的条目                       | CPU1的`a`缓存行失效（状态转为**Invalid**） | -                                    | 后续读取`a`将触发缓存未命中                   |
| **10**    | -                                                       | 执行`assert(a==1)`，发现`a`缓存失效                   | -                              | CPU1发送**Read消息**请求`a`                | 断言因缓存未命中暂时未执行                     |
| **11**    | 收到CPU1的Read请求                                           | -                                            | CPU0的`a`缓存行降级为**Shared**       | CPU0发送`a=1`的**Read Response**        | CPU1将获取`a=1`                      |
| **12**    | -                                                       | 接收`a=1`的缓存行并加载                               | CPU1缓存`a=1`（状态**Shared**）      | -                                    | **断言成功**                          |

---

和之前不同的是，在Step 8，CPU1遇到了内存屏障，因此它先处理`Invalidate Queue`中的数据，保证了在之后的`Load a`的时候，不再从自己的缓存行中直接读取。


## **3、Read and Write Memory Barriers**

---
* **读内存屏障（Read Memory Barrier）仅标记 Invalidate Queue**
* **写内存屏障（Write Memory Barrier）仅标记 Store Buffer**
* **全内存屏障（Full Memory Barrier）则同时操作两者**

---
- **读内存屏障**仅对执行它的CPU上的`Load`操作保证顺序，确保屏障前的所有`Load`操作在屏障后的任何`Load`操作之前完成。
- **写内存屏障**仅对执行它的CPU上的`Store`操作保证顺序，确保屏障前的所有`Store`操作在屏障后的任何`Store`操作之前完成。
- **全内存屏障**则同时对`Store`和`Load`操作保证顺序，但同样仅限于执行屏障的CPU。
---
