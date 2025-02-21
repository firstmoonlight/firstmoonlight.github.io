---
layout: post
title: Rooline Model笔记
tags: [cuda learning]
---

本文主要介绍roofline model的概念以及其在nsight compute中的作用

参考资料：

[Roofline Model与深度学习模型的性能分析](https://zhuanlan.zhihu.com/p/34204282)

## 相关概念
### **1. 计算平台的两个指标：算力 π 与带宽 β**

- **算力** π ：也称为计算平台的**性能上限**，指的是一个计算平台倾尽全力每秒钟所能完成的浮点运算数。单位是 `FLOPS` or `FLOP/s`。

```text
π : Maximum FLOPs Per Second
```

- **带宽** β ：也即计算平台的**带宽上限**，指的是一个计算平台倾尽全力每秒所能完成的内存交换量。单位是`Byte/s`。

```text
β : Maximum Memory Access Per Second
```


* **计算强度上限 Imax** ：两个指标相除即可得到计算平台的**计算强度上限**。它描述的是在这个计算平台上，单位内存交换最多用来进行多少次计算。单位是`FLOPs/Byte`。
```text
Imax = π / β 
```

### **2. 模型的两个指标：计算量 与 访存量**

* **计算量**：指的是输入单个样本（对于CNN而言就是一张图像），模型进行一次完整的前向传播所发生的浮点运算个数，也即模型的**时间复杂度**。单位是 `#FLOP` or `FLOPs`。

* **访存量**：指的是输入单个样本，模型完成一次前向传播过程中所发生的内存交换总量，也即模型的**空间复杂度**。在理想情况下（即不考虑片上缓存），模型的访存量就是模型各层权重参数的内存占用（Kernel Mem）与每层所输出的特征图的内存占用（Output Mem）之和。单位是`Byte`。

* **模型的计算强度** I ：由计算量除以访存量就可以得到模型的计算强度，它表示此模型在计算过程中，每`Byte`内存交换到底用于进行多少次浮点运算。单位是`FLOPs/Byte`。可以看到，模计算强度越大，其内存使用效率越高。

* **模型的理论性能 P** ：我们最关心的指标，即模型**_在计算平台上_所能达到的每秒浮点运算次数（理论值）**。单位是 `FLOPS` or `FLOP/s`。


##  **Roof-line Model**

roof-line model显示的是**模型在一个计算平台的限制下，到底能达到多快的浮点计算速度**。这里的模型我们也可以简单理解为一个kernel函数。

它说明了这样的一个问题，即“**计算量为A且访存量为B的模型在算力为C且带宽为D的计算平台所能达到的理论性能上限E是多少**”。

### **1. Roof-line 的形态**

Roof-line的图形如下图所示，它显示了一个类似屋檐的图案。Y轴表示算力的 大小，X轴表示计算强度，其斜率代表了带宽。这里面有两个值是固定的，平台算力上限`π`以及平台带宽`β`。

![image](https://github.com/user-attachments/assets/eb22112a-e0f0-4c3d-8d15-7bdd42ccf55f)

### **2. Roof-line 划分出的两个瓶颈区域**

$$ P= \begin{cases}  \beta * I ,&\text{When I < $I_{max}$, Memory Bound}\\ 
\pi,&\text{When I $\geq$ $I_{max}$, Compute Bound} \end{cases} $$

### **Compute Bound**

不管模型的计算强度 I 有多大，它的理论性能 P 最大只能等于计算平台的算力 π 。当模型的计算强度 $I$ 大于计算平台的计算强度上限 $I_{max}$ 时，模型在当前计算平台处于 `Compute-Bound`状态，即模型的理论性能 P 受到计算平台算力 π 的限制，无法与计算强度 I 成正比。

### **Memory Bound**

当模型的计算强度 $I$ 小于计算平台的计算强度上限$I_{max}$ 时，此时算力 $P$ 随着计算强度 $I$ 的增强而不断上升，此时制约模型性能的指标变为了带宽 $\beta$ ，即模型此时处于`Memory-Bound` 状态。在该状态下，我们可以提升计算强度，即在单个thread中加入大量计算，以增加每内存的计算次数，使模型到达`Compute Bound`状态。如果带宽未到达平台的最大带宽，那么修改kernal增加带宽来改进模型性能也是一种选择。

## **Nsight Compute 实例分析**

### **1.  Memory Bound**

如下图，此时算力`P = 25440993788`，计算强度`I = 0.10`，应该为`0,96`左右，图上显示并非在0.10这条线上。计算得到带宽$\beta$ = 264G/s左右，和下图的`Memory Throuput`一样。

![Pasted image 20250220105540](https://github.com/user-attachments/assets/9a719985-278d-438c-a45d-ad4f3b251746)

![Pasted image 20250220110251](https://github.com/user-attachments/assets/b1d87be9-2b42-4a01-9b23-2d5b6e2dbb9d)


```C++
static __global__ void sumArrays1GPU(float* a, float* b, float* res, int offset, int n)
{
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    res[i] = a[i] + b[i];
}
```
### **2. Compute Bound**

如下图所示，我们在kernal中将计算次数增大，从roofline model可以看出，此时我们已经到达了`Compute Bound`状态，理论上已经最大限度上发挥了GPU的性能。

![Pasted image 20250220105603](https://github.com/user-attachments/assets/6687348e-b1c5-4688-9315-3d3f35602c12)


![Pasted image 20250220110954](https://github.com/user-attachments/assets/45d447fe-4700-40f7-a471-3dd3ff5fc843)


```C++
static __global__ void sumArrays2GPU(float* a, float* b, float* res, int offset, int n)
{
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    float x = a[i];
    float y = b[i];
    float z = 0;

    for (int k = 0; k < 1000; ++k) {
        z += (x + y + k);
    }

    res[i] = z;
}
```
