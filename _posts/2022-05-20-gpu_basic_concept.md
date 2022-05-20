---
layout: post
title: GPU硬件和软件概念理解
date: 2022-05-20
tags: GPU   
---

## 一、基础概念

### GPU

显存+计算单元

- 显存（Global Memory）：

  显存是在GPU板卡上的DRAM，类似于CPU的内存，就是那堆DDR啊，GDDR5啊之类的。特点是容量大（可达16GB），速度慢，CPU和GPU都可以访问。

- 计算单元（Streaming Multiprocessor）：

  执行计算的。每一个SM都有自己的控制单元（Control Unit），寄存器（Register），缓存（Cache），指令流水线（execution pipelines）。

#### 1、物理概念

​	GPU可以看成是流处理器簇 Streaming Multiprocessors（SM）的阵列。如图所示

​	![ln84mxev5g](https://Lzhang-hub.github.io/images/posts/ml/ln84mxev5g.jpeg)

​	单个SM图：

​	![20180525110906547](https://Lzhang-hub.github.io/images/posts/ml/20180525110906547.png)

![image-20220505105829615](https://Lzhang-hub.github.io/images/posts/ml/image-20220505105829615.png)

- **streaming processor(sp)**：最基本的处理单元，也称为cuda core，也称为thread(软件层面的概念)。GPU进行并行计算，也就是很多个sp同时做处理。每一个线程都有自己的寄存器内存和local memory。
- **streaming multiprocessor(sm)**： 多个sp加上其他的一些资源组成一个sm, 其他资源也就是存储资源，共享内存，寄储器等。可见，一个SM中的所有SP是先分成warp的，是共享同一个memory和instruction unit（指令单元）。从硬件角度讲，一个GPU由多个SM组成（当然还有其他部分），一个SM包含有多个SP（以及还有寄存器资源，shared memory资源，L1cache，scheduler，SPU，LD/ST单元等等） A100中有108个SM，每个SM有64个Thread，所有总共有6192个core。

#### 2、软件概念

- **thread：**一个CUDA的并行程序会被以许多个threads来执行。
- **block：**数个threads会被群组成一个block，同一个block中的threads可以同步，也可以通过shared memory通信。
- **grid**：多个blocks则会再构成grid。
- **warp：**GPU执行程序时的调度单位，通常一个SM中的SP(thread)会分成几个warp(也就是SP在SM中是进行分组的，物理上进行的分组)，，目前cuda的warp的大小为32，同在一个warp的线程，以不同数据资源执行相同的指令,这就是所谓SIMT

#### 3、软硬件关系

**一个网格相当于一个GPU设备，网格下分成多个线程块，线程块则对应的SM，每个线程块又分为多个线程，每个线程相当于一个CUDA核。**

**线程块对应的是SM，即软件层面的线程块会被分到硬件层面的SM，在SM中执行线程块又被分成warp，通过SM中warp scheduler去调度。**