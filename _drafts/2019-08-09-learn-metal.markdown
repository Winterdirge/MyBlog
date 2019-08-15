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

>使用图像处理器渲染高级3D图形并执行数据并行计算

# 前面

图形处理器(GPU:Graphics processeror)旨在快速渲染图形并执行并行计算，如果你想要直接与设备上的可用GPU通信，那么请使用Metal框架，渲染复杂的3D场景或执行高级科学计算时可使用Metal来达到最佳性能。

Metal可以与其他框架密切配合，使用MetalKit可以简化在屏幕上获取Metal内容的流程，使用"Metal Performance Shader"可以自定义渲染任务以及利用现有的功能库。

许多高级的Apple框架都是基于Metal构建的，包括Core Image、SpriteKit、SceneKit。使用这些高级框架可以使你免受GPU编程细节的影响，但是编写自定义的Metal代码可以使你获得高效的性能。

下面我们会通过几个官方的例子来熟悉一下Metal的工作流程。

# 使用GPU执行计算

## 基本步骤

* 将一个简单的C函数转换为MSL(Metal Shading Language)
* 找到一个可用GPU
* 创建pipeline作为MSL函数在GPU上运行的准备
* 申请一块GPU可以访问的存储空间用来存储GPU处理的数据
* 创建一个命令缓存(command buffer)，编码GPU命令用来处理数据
* 将command buffer提交到commannd queue，然后GPU才可以执行编码后的命令

## 