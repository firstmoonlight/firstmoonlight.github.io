---
layout: post
title: Linux中创建swap file
tags: [Linux]
---

本博客主要介绍如何在linux中创建swap file。

## 参考资料
[How To Create A Swap File In Linux](https://digitizor.com/create-swap-file-ubuntu-linux/)

## 背景描述
在编译openssl的过程中，出现报错virtual memory exhausted: Cannot allocate memory，发现是内存不够，查看swap，发现系统的swap为0，因此决定创建一个swap file，作为内存的补充。
```
                   总计         已用        空闲      共享    缓冲/缓存    可用
内存：        1992         130        1628           0         233        1719
交换：              0             0              0

```


## 概述
The swap space is an area of the hard disk that is used when the RAM runs out of memory. If your system has enough RAM it is generally not necessary to have swap space. However, if you have installed Linux without creating any swap partition and later want some swap space, you can easily do so. You can either create a swap partition or create a swap file.The swap file approach is simpler and safer if you want to add swap space post OS installation. Beside, swap file and swap partition have the same performance now. I carried out the instructions I am giving here on Ubuntu 10.10, but it should work on any other distributions as well.The first thing we need to do is create an empty file of the required size. To do this open the Terminal and execute the command given below. In this example I am creating a 3GB swap file. You will not probably need that much. So, enter your own value.

## 操作步骤
```

$ cd /
$ sudo dd if=/dev/zero of=swapfile bs=1M count=3000

To set it as 1GB, change the count value (3000 in the example above) to 1000, 1500 for 1.5GB etc.
Now change the file created to a swap file with the command below.

$ sudo mkswap swapfile

Turn on the swap file with the command,

$ sudo swapon swapfile

To ensure that the swap file is turned on automatically at system startup, open fstab.

$ sudo nano etc/fstab

And add the line given below. Save and close.

/swapfile none swap sw 0 0That is it. 

You can check if the system is using the swap file you created with the command

$ cat /proc/meminf
```
