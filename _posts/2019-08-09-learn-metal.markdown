---
layout:     post
title:      Metal探索之旅（一）
subtitle:   Metal的渲染流程
date:       2019-08-09 20:01:55 GMT+8
author:     "Yang Guang"
header-style: text
tags:
    - iOS
    - 技术
    - 渲染
    - 并行计算
    - GPU
    - Metal
---

>本文的大部分内容来源于苹果官方文档，主要介绍如何使用图像处理器渲染高级3D图形并执行数据并行计算

# 前面

图形处理器(GPU:Graphics processeror)旨在快速渲染图形并执行并行计算，如果你想要直接与设备上的可用GPU通信，那么请使用Metal框架，渲染复杂的3D场景或执行高级科学计算时可使用Metal来达到最佳性能。

Metal可以与其他框架密切配合，使用MetalKit可以简化在屏幕上获取Metal内容的流程，使用"Metal Performance Shader"可以自定义渲染任务以及利用现有的功能库。

许多高级的Apple框架都是基于Metal构建的，包括Core Image、SpriteKit、SceneKit。使用这些高级框架可以使你免受GPU编程细节的影响，但是编写自定义的Metal代码可以使你获得高效的性能。

下面我们会通过几个官方的例子来熟悉一下Metal的工作流程。

# 使用GPU执行计算

## 基本步骤

* 将一个简单的C函数转换为MSL(Metal Shading Language)
* 找到一个可用GPU
* 创建pipeline准备在GPU上运行MSL函数
* 申请一块GPU可以访问的存储空间用来存储GPU处理的数据
* 创建一个命令缓存(command buffer)，编码GPU命令用来处理数据
* 将command buffer提交到commannd queue，GPU读取queue中的命令进行执行

## 将C函数转换为MSL

以简单的向量相加为例来进行说明，首先一个C语言编写简单的加法函数：

```c
void add_arrays(const float* inA, const float* inB, float* result, int length) {
    for (int index = 0; index < length; ++index) {
        result[index] = inA[index] + inB[index];
    }
}
``` 

可以看到，结果向量中的每个元素都是单独计算的，所以可以安全的同时对它们进行计算。想要在GPU中完成这个工作，首先需要将上面的方法`翻译为`GPU可以看懂的语言，也就是MSL。MSL是为GPU编程而设计的一种C++的变种语言，书写起来和C++有点像，但是它的后缀名是`.metal`。在Metal中，在GPU上运行的代码叫做`着色器`，因为历史上它们首先用于计算3D图形中的颜色。

Xcode构建Metal文件时会创建一个默认的Metal库，并将其嵌入到应用程序中，我们可以从这个库中获取`.metal`文件中编写的函数的指针。

改写后的MSL代码如下：

```metal
kernel void add_arrays(device const float* inA, 
                       device const float* inB, 
                       device float* result,
                       uint index [[thread_position_in_grid]]) {
    //the for-loop is replaced with a collection of threads, each of which calls this function.
    result[index] = inA[index] + inB[index];
}
```

对比上面两段代码，我们可以看到它们之间的明显差异。

首先，`kernel`关键字表明

* 该函数是一个公有的GPU函数，公有函数是你的App可以看到的唯一函数，这种函数不能被其他着色器方法调用。

* 计算函数（又叫计算内核compute kernel）通过线程网格来执行并行计算。

`device`关键字，该关键字表明指针位于设备的地址空间中。MSL为内存定义了几个不相交的地址空间，每当在MSL中声明指针时，都必须提供一个关键字来声明其地址空间，使用设备的地址空间可以声明GPU可以读写的持久内存。

其次，MSL代码中移除了for循环，改为`compute grid`中的多个线程调用，方便向量中的元素由不同线程计算。