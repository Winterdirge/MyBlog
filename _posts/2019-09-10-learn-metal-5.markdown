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

和OpenGL一样，在Metal中我们也是使用纹理来绘制和处理图像。纹理是结构化的纹理元素集合，这些纹理元素通常叫做纹素或像素。这些纹理元素的确切配置取决于纹理类型。