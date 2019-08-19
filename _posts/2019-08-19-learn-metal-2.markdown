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

# 创建渲染过程描述符

在进行绘制时，GPU将绘制结果存在纹理中，纹理其实就是GPU可访问的、存储图像数据的一块内存。本文中，MTKView默认已经创建了我们绘制所需的所有纹理，它会创建多个纹理，以便绘制一个纹理的时候显示另一个。

绘制时需要指明渲染过程，该过程是绘制一组纹理的命令序列。使用该序列时，纹理也被叫做渲染目标。我们需要使用渲染过程描述符来创建一个渲染过程，该描述符是`MTLRenderPassDescriptor`的一个实例。本文不需要创建该描述符，因为MTKView默认已经创建该描述符。

```objc
MTLRenderPassDescriptor *renderPassDescriptor = mtkView.currentRenderPassDescriptor;
if (renderPassDescriptor == nil)
{
    return;
}
```

渲染过程描述符定义了一组渲染目标，以及渲染开始和结束时需要对它们进行怎样的处理。MTKView包含一个渲染过程描述符，该描述符具有指向视图纹理之一的单色附件，并通过视图的属性配置渲染过程。默认情况下，这意味着渲染开始时，渲染目标将被擦除为与视图属性`clearColor`相匹配的颜色，在渲染结束时所有的更改都将存回纹理。

# 创建渲染过程

我们通过使用`MTLRenderCommandEncoder`将渲染过程编码到命令缓冲区来创建渲染过程，命令缓冲区相关内容上篇文章已经简单涉及。

```objc
id<MTLRenderCommandEncoder> commandEncoder = [commandBuffer renderCommandEncoderWithDescriptor:renderPassDescriptor];
```

本文我们不编码任何命令，只是简单的擦除纹理。

```objc
[commandEncoder endEncoding];
```

# 屏幕展示

绘制纹理不会自动在屏幕上显示新内容，事实上，只有某些纹理可在屏幕上显示。Metal中，可在屏幕上显示的纹理由可绘制对象管理，要显示纹理，需要展示可绘制对象。

MTKView自动创建了可绘制对象用来管理它的纹理，通过MTKView的`currentDrawable`属性来获取可绘制对象，该对象包含了渲染过程的目标纹理，它是一个`CAMetalDrawable`对象，是一个连接到`CoreAnimation`的对象。

```objc
id<MTLDrawable> drawable = view.currentDrawable;
```

在命令缓冲区上调用`present(_:)`方法，传入可绘制对象。

```objectivec
[commandBuffer presentDrawable:drawable];
```

该方法告诉Metal，当命令缓冲区被安排执行时，Metal应该与Core Animation协调以完成渲染后的纹理展示。当Core Animation呈现纹理时，它将成为视图的新内容。本文中，被擦除的纹理将成为视图的新内容，此更改将在Core Animation更新其他界面元素时一起进行。

# 提交命令缓冲

```objectivec
[commandBuffer commit];
```

# 最后

至于MTKView更多的属性，请阅读MTKView头文件。