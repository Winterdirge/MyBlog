---
layout:     post
title:      Metal探索之旅(二)
subtitle:   使用Metal进行绘制
date:       2019-08-19 12:03:55 GMT+8
author:     "Yang Guang"
catalog:    true
header-style: text
tags:
    - iOS
    - 技术
    - 渲染
    - GPU
    - Metal
---

>前一篇文章我们介绍了使用Metal进行简单的并行计算，本文将介绍使用Metal进行简单内容绘制

# 准备工作

MTKView是UIView(iOS、tvOS)或NSView(macOS)的子类，它处理了许多关于获取Metal绘制内容的细节。

一个MTKView需要和一个Metal设备对象关联起来才可以正常工作，所以我们第一步需要

```objc
mtkView.device = MTLCreateSystemDefaultDevice();
```

MTKView中的其他属性允许你控制它的一些行为，例如将视图的内容擦除为纯色，需要使用`clearColor`属性。

```objc
mtkView.clearColor = MTLClearColorMake(0.0, 0.5, 1.0, 1.0);
```

本文中不涉及动画绘制，我们只在视图内容更新时绘制，所以进行如下设置

```objc
mtkView.enableSetNeedsDisplay = YES;
```

# 委派绘制任务

MTKView依赖于你的App向Metal发送命令进而生成可视内容，Metal使用代理模式通知你的App需要重绘，想要接收代理回调，需要设置MTKView的`delegate`

```objc
mtkView.delegate = someObject;
```

这个代理干了两件事

* 当内容大小发生变化时，调用`mtkView(_:drawableSizeWillChange:)`方法。当窗口包含的视图大小发生变化，或者设备的方向发生变化(iOS上)，都会调用该方法，方便你的App调整视图的大小。

* 在视图更新的时候，会调用`draw(in:)`方法。此方法中，你需要创建命令缓冲区，编码命令以便告诉GPU绘制什么，以及何时在屏幕上显示，并将该命令放入命令缓冲区放入队列以便GPU执行。

