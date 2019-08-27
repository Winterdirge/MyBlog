---
layout:     post
title:      Metal探索之旅(三)
subtitle:   Hello Triangle
date:       2019-08-20 20:37:55 GMT+8
author:     "Yang Guang"
header-style: text
tags:
    - iOS
    - 技术
    - 渲染
    - GPU
    - Metal
---

# 前面

说起`Hello Triangle`，学过OpenGL的人应该都知道`你好，三角形`，这算是OpenGL经典入门案例了。对于Metal的学习，我们也将从绘制一个简单的三角形开始。

上一片文章中，我们仅仅是将一个视图擦除为纯色，并没有在此基础上进行进行内容绘制，本文将在此基础上通过编写`Metal Shader`，来完成一个彩色三角形的绘制。

# Metal渲染管线

渲染管线处理绘画命令，并将数据写入渲染目标。渲染管线有多个阶段，有的阶段使用可编程的shader处理，有的对数据进行配置和修正。本文主要介绍渲染管线的三个主要阶段：顶点化、光栅化以及片段化。其中，顶点和片段处理阶段是可编程的，可使用MSL对其进行编程实现。光栅化阶段会进行插值处理。

![](/assets/images/2019/metal_render_pipeline.png)

渲染开始从一个绘制命令开始，该命令包括顶点数量以及需要渲染的图元类型，例如

```objectivec
[renderEncoder drawPrimitives:MTLPrimitiveTypeTriangle 
                  vertexStart:0 
                  vertexCount:3];
```

顶点处理阶段会为每个顶点提供数据，当足够的顶点处理完成之后，渲染管线会对图元进行光栅化处理，确定渲染目标的哪些像素在图元边界的内部，片段处理阶段会决定渲染目标的这些像素写入的具体值。

本文会涉及如何编写顶点和片段着色器，如何创建渲染管线状态对象，以及如何编码绘制命令使用渲染管线。

# 通过自定义的渲染管线定义数据的处理方式

顶点函数为顶点生成数据，片段函数为片段生成数据，但是我们可以决定它们怎么工作。我们可以配置管线的各个阶段，这就意味着我们需要知道管线会生成什么以及如何生成这些结果。说白了就是，想自定义管线的处理内容，需要对管线非常了解才行。

管线中我们有三个地方可以定义管线中传入的内容

* 渲染管线的输入，由你的app决定，传入到顶点处理阶段
* 顶点处理阶段的输出，该输出会用于光栅化阶段
* 片段处理阶段的输入，又你的app提供，或者由光栅化阶段生成

本文中输入数据为顶点位置和颜色信息，为了表示我们在顶点着色器中对顶点进行了哪些转换，输入坐标将使用自定义的坐标空间，以像素为单位，从视图的中间开始计算。但是，这些坐标需要转化到Metal的坐标系统。

首先定义顶点数据结构

```c
typdef struct {
    vector_float2 position;
    vector_float4 color;
} AAPLVertex;
```

至于`vector_float2`和`vector_float4`代表什么，可以参考`simd library`。

初始化三角形的三个顶点信息

```c
static const AAPLVertex triangleVertices[] = 
{
    //2D positions,        RGBA colors
    { {  250, -250 },     { 1, 0, 0, 1 } },
    { { -250, -250 },     { 0, 1, 0, 1 } },
    { {    0,  250 },     { 0, 0, 1, 1 } },
};
```

顶点着色器为每一个顶点生成需要数据，所以它需要提供颜色和转换后的位置信息。我们将其定义如下

```c
typdef struct {
    float4 position [[position]];
    float4 color;
} RasterizerData;
```

其中，`[[position]]`关键字表示当该结构作为结果返回时，该字段是顶点的位置信息。