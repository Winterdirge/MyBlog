---
layout:     post
title:      Metal探索之旅(五)
subtitle:   Creating and Sampling Textures
date:       2019-09-10 17:00:00 GMT+8
author:     "Yang Guang"
header-style: text
tags:
    - iOS
    - 技术
    - 渲染
    - GPU
    - Metal
---

>上篇文章我们介绍了如何有效地利用CPU和GPU来让渲染更加高效，本篇文章将介绍如何创建以及如何对纹理进行采样。

### Overview

和OpenGL一样，在Metal中我们也是使用纹理来绘制和处理图像。纹理是结构化的纹理元素集合，这些纹理元素通常叫做纹素或像素。这些纹理元素的确切配置取决于纹理类型。本文使用的纹理中的元素是以二维数组方式排列的，每个元素包含颜色信息，它们共同构建出一幅图像。纹理通过纹理映射的方式绘制到几何图元上，片段着色器通过纹理采样为每个片段生成颜色。

Metal中的纹理由`MTLTexture`对象管理，MTLTexture对象定义了纹理的格式，包括元素的大小和布局、纹理中元素数量以及这些元素的组织方式。一旦创建，纹理的格式和组织方式将无法改变。但是，我们通过渲染或复制数据等方式来修改纹理的内容。

Metal框架不提供直接将图像数据从文件加载到纹理的API，Metal本身只分配纹理资源，并提供从纹理读写数据的方法。使用Metal的应用依赖于自定义代码或其他框架，例如MetalKit、Image I/O、UIKit、AppKit，来处理图像文件。例如，可以使用`MTKTextureLoader`来执行简单的纹理加载，后面会介绍一个简单的自定义的纹理加载程序。