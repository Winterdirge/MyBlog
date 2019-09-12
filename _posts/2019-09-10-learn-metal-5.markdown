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

### 从纹理描述符创建纹理

使用`MTLTextureDescriptor`对象配置纹理的像素格式、纹理的宽高等属性，然后通过`newTextureWithDescriptor:`来创建纹理

```objc
MTLTextureDescriptor *textureDescriptor = [[MTLTextureDescriptor alloc] init];

// Indicate that each pixel has a blue, green, red, and alpha channel, where each channel is
// an 8-bit unsigned normalized value (i.e. 0 maps to 0.0 and 255 maps to 1.0)
textureDescriptor.pixelFormat = MTLPixelFormatBGRA8Unorm;

// Set the pixel dimensions of the texture
textureDescriptor.width = image.width;
textureDescriptor.height = image.height;

// Create the texture from the device by using the descriptor
id<MTLTexture> texture = [_device newTextureWithDescriptor:textureDescriptor];
```

Metal创建`MTLTexture`对象，并为纹理数据分配内存，纹理创建时为纹理分配的内存是未初始化的，所以需要将数据拷入该纹理。

### 复制图像数据到纹理

Metal管理纹理内存，并不提供直接访问该内存的方法，所以无法直接获取纹理数据的内存指针，直接拷贝数据。相反，只能调用`MTLTexture`对象的方法将数据从可以访问的内存复制到纹理中，反之亦然。

在本文例子中，`AAPLImage`对象为图像数据申请了内存，所以需要拷贝这块内存。

使用`MTLRegion`结构体定义需要更新纹理的哪块区域，这个例子使用图像数据生成这个纹理，所以需要创建一个覆盖真个纹理的区域。

```objc
MTLRegion region = {
    { 0, 0, 0 },
    { image.width, image.height, 1}
};
```

图像数据以行的形式排列，需要告诉Metal原始图像中行之间的偏移。图像加载代码以一种紧凑的格式创建了图像数据，因此后续行的像素数据紧跟前一行，行之间的偏移量就是每行的确切长度，也就是每个像素的字节数乘以图像的宽度。

```objc
NSUInteger bytesPerRow = 4 * image.width;
```

使用`replaceRegion:mipmapLevel:withBytes:bytesPerRow:`方法来填充纹理数据：

```objc
[texture replaceRegion:region
            mipmapLevel:0
              withBytes:image.data.bytes
            bytesPerRow:bytesPerRow];
```

### 将纹理映射到图元

不能单独渲染纹理，必须将其映射到图元（由顶点阶段输出并光栅化为片段的几何基本体，本例中是一对三角形）。每个片段都需要知道要渲染纹理的哪部分，我们使用纹理坐标定义这种映射，将纹理图像的坐标映射到几何体表面坐标。

对于2D纹理来说，归一化的坐标是x、y方向上的[0,1]，（0.0， 0.0）指纹理数据的第一个字节（图像的左上角）处的像素，（1.0， 1.0）指纹理数据的最后一个字节（图像右下角）处的像素。

![](/assets/images/2019/learn_metal_10.png)

在顶点结构体中增加纹理坐标：

```objc
typedef struct {
    // Positions in pixel space. A value of 100 indicates 100 pixels from the origin/center.
    vector_float2 position;

    // 2D texture coordinate
    vector_float2 textureCoordinate;
}AAPLVertex;
```

在顶点数据中，将四边形的角映射到纹理的角：

```objc
static const AAPLVertex quadVertices[] =
{
    // Pixel positions, Texture coordinates
    { {  250,  -250 },  { 1.f, 1.f } },
    { { -250,  -250 },  { 0.f, 1.f } },
    { { -250,   250 },  { 0.f, 0.f } },

    { {  250,  -250 },  { 1.f, 1.f } },
    { { -250,   250 },  { 0.f, 0.f } },
    { {  250,   250 },  { 1.f, 0.f } },
};
```

将纹理坐标上传到片段着色器，需要在`RasterizerData`结构体增加`textureCoordinate`变量：

```objc
typedef struct
{
    // The [[position]] attribute qualifier of this member indicates this value is
    // the clip space position of the vertex when this structure is returned from
    // the vertex shader
    float4 position [[position]];

    // Since this member does not have a special attribute qualifier, the rasterizer
    // will interpolate its value with values of other vertices making up the triangle
    // and pass that interpolated value to the fragment shader for each fragment in
    // that triangle.
    float2 textureCoordinate;

} RasterizerData;
```

在顶点着色器中，将纹理坐标写入`textureCoordinate`字段，然后将其传递到光栅化阶段，光栅化阶段会在四边形的三角片段上对这些坐标进行插值。

```objc
out.textureCoordinate = vertexArray[vertexID].textureCoordinate;
```

### 计算纹理颜色

通过对纹理进行采样可以获取纹理某个位置的颜色值，要采用纹理，片段着色器需要纹理坐标和一个对纹理的引用。除了从光栅化阶段传入的参数之外，还需要传入一个`texture2d`类型并用`[[texture(index)]]`修饰符限定的颜色纹理参数，该参数引用一个即将被采样的纹理。

```objc
fragment float4 samplingShader(RasterizerData in [[stage_in]], texture2d<half> colorTexture [[texture(APPLTextureIndexBaseColor)]])
```

使用内置函数`sample()`来采样纹理数据，该方法有两个参数，一个是采样器(textureSampler)，描述了如何采样纹理，另一个是纹理坐标(in.textureCoordinate)，描述了采样位置。该方法会获取一个或多个像素，并返回一个根据这些像素计算出的颜色值。

当渲染区域和纹理大小不同时，采样器可以使用不同的算法来计算`sample()`函数应该返回的像素颜色。设置`mag_filter`模式将会指定渲染区域比纹理大的时候采样器应该返回的颜色，`min_filter`将会指定渲染区域小于纹理大小时采样器对返回颜色的计算方式。如果将两个filter都设置为线性模式，采样器将对给定纹理坐标周围的颜色做一个平均，将得出一个比较平滑的输出图像。

```objc
constexpr sampler textureSampler (mag_filter::linear,
                                  min_filter::linear);

// Sample the texture to obtain a color
const half4 colorSample = colorTexture.sample(textureSampler, in.textureCoordinate);
```

>感兴趣的小伙伴可以实验下，改变正方形的大小，看看滤波器是怎么工作的。

### 编码绘制参数

编码和提交绘制命令的过程和上几篇文章相同，这里不再赘述。本例的不同之处在于片段着色器多了一个参数，当你编码命令参数时需要设置片段着色器的纹理参数，本文采用`AAPLTextureIndexBaseColor`在OC和MSL代码中指明纹理索引。

```objc
[renderEncoder setFragmentTexture:_texture
                          atIndex:AAPLTextureIndexBaseColor];
```

###### TO BE CONTINUED