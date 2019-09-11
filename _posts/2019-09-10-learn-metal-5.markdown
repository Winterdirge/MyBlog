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

### 加载和格式化图像数据

我们可以手动创建或更新纹理内容，这一过程将在本文后面进行介绍。手动创建可能出于以下原因：

* 可以自定义数据图像的存储格式
* 需要在运行时生成纹理内容
* 从服务器拉取流式纹理数据，或者需要动态更新纹理内容

文中使用`AAPLImage`类从`TGA`文件加载和解析图像数据，该类将TGA文件中像素数据转换为Metal可以理解的像素格式。例子中使用图像的元数据创建一个Metal纹理，并将像素数据拷贝到该纹理。

>**NOTE**:  
`AAPLImage`类不是本文的讨论重点，这个类只是阐明了与Metal框架无关的基本图像操作，它的作用是方便加载图像数据，并将其转换为Metal的像素格式。如果对此比较感兴趣，也可以写一个自己的加载方法。

Metal需要所有的纹理都转换为`MTLPixelFormat`中限定的值，这些格式描述了纹理中像素数据的布局。文中例子使用`MTLPixelFormatBGRA8Unorm`格式，这种格式的每个像素有32比特，每8个比特组成一个部分，每个部分以蓝色、绿色、红色、透明度的顺序排列。如下图所示。

![](/assets/images/2019/learn_metal_09.png)

在填充Metal纹理之前，必须将图像数据格式化为纹理的像素格式。`TGA`文件可以提供32位每像素或24位每像素等格式的数据，使用32位每像素的`TGA`文件已经按照`BGRA`的格式排列好了，所以只需要复制像素数据即可。要转换24位每像素的BGR图像，请复制红色、绿色、蓝色通道，并将透明度通道设置为255，该值表明完全不透明的像素数据。

```objc
// Initialize a source pointer with the source image data that's in BGR form
uint8_t *srcImageData = ((uint8_t*)fileData.bytes +
                         sizeof(TGAHeader) +
                         tgaInfo->IDSize);

// Initialize a destination pointer to which you'll store the converted BGRA
// image data
uint8_t *dstImageData = mutableData.mutableBytes;

// For every row of the image
for(NSUInteger y = 0; y < _height; y++) {

    // If bit 5 of the descriptor is not set, flip vertically
    // to transform the data to Metal's top-left texture origin
    NSUInteger srcRow = (tgaInfo->topOrigin) ? y : _height - 1 - y;

    // For every column of the current row
    for(NSUInteger x = 0; x < _width; x++) {

        // If bit 4 of the descriptor is set, flip horizontally
        // to transform the data to Metal's top-left texture origin
        NSUInteger srcColumn = (tgaInfo->rightOrigin) ? _width - 1 - x : x;

        // Calculate the index for the first byte of the pixel you're
        // converting in both the source and destination images
        NSUInteger srcPixelIndex = srcBytesPerPixel * (srcRow * _width + srcColumn);
        NSUInteger dstPixelIndex = 4 * (y * _width + x);

        // Copy BGR channels from the source to the destination
        // Set the alpha channel of the destination pixel to 255
        dstImageData[dstPixelIndex + 0] = srcImageData[srcPixelIndex + 0];
        dstImageData[dstPixelIndex + 1] = srcImageData[srcPixelIndex + 1];
        dstImageData[dstPixelIndex + 2] = srcImageData[srcPixelIndex + 2];

        if(tgaInfo->bitsPerPixel == 32) {

            dstImageData[dstPixelIndex + 3] =  srcImageData[srcPixelIndex + 3];
        } else {
            
            dstImageData[dstPixelIndex + 3] = 255;
        }
    }
}
_data = mutableData;
```