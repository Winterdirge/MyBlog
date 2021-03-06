---
layout:     post
title:      Metal探索之旅(三)
subtitle:   Hello Triangle
date:       2019-08-27 20:37:55 GMT+8
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

其中，`[[position]]`关键字表示当该结构作为结果返回时，该字段是顶点的位置信息，并且它必须被定义为`vector_float4`，你需要告诉需要光栅化的数据中哪一部分是顶点位置信息，由于Metal没有对字段的命名进行约定，所以这里就需要使用属性限定符来声明该字段是顶点数据的输出位置。

# 顶点着色器

先来看一下顶点着色器的声明

```cpp
vertex RasterizerData vertexShader(
    uint vertexID [[vertex_id]],
    constant AAPLVertex *vertices [[buffer(AAPLVertexInputIndexVertices)]],
    constant vertor_uint2 *viewportSizePointer [[buffer(AAPLVertexInputIndexViewportSize)]]    
);
```

第一个参数`vertexID`，使用`[[vertex_id]]`限定符。当执行渲染命令时，GPU会多次调用顶点着色器，为每个顶点产生唯一值。

第二个参数`vertices`，是一个包含顶点数据的数组，使用前面定义的`AAPLVertex`结构体。

为了将顶点位置转化到Metal坐标系，我们需要知道绘制三角形的viewport的大小，这个数值存储在viewportSizePointer中。

后两个参数具有`[[buffer(n)]]`限定符，在默认情况下Metal会为参数列表中的每个参数分配默认的位置(slots)。但是当我们添加`[[buffer(n)]]`限定符时，就相当于指定了参数位置，Metal将使用我们指定的位置。这种做法有一个好处，当我们修改shader时，无需在修改应用程序代码。

## 编写顶点着色器

我们的顶点着色器必须生成输出结构中的两部分数据，使用`vertexID`参数从vertices数组中获取输入的顶点信息，同时需要获取viewport尺寸信息。

```cpp
float2 pixelSpacePosition = vertices[vertexID].position.xy;

// Get the viewport size and cast to float.
vector_float2 viewportSize = vector_float2(*viewportSizePointer);
```

顶点着色器必须提供顶点在裁剪坐标中的位置信息，这些位置信息是使用四维齐次矢量`(x, y, z, w)`指定的3D点。光栅化阶段将会使用x、y、z除以w，在标准化设备空间中生成3D点，标准化设备坐标与视口大小无关。

![](/assets/images/2019/learn_metal_02.png)

标准化设备空间使用左手坐标系将位置映射到viewport中，在这个坐标系下图元会被裁剪到[-1, 1]范围中，然后被光栅化。剪切框的左下角位于[-1, 1]，右上角位于[1, 1]，正z方向远离相机(进入屏幕方向)。z方向上的可见部分位于近裁剪平面和远裁剪平面之间。

![](/assets/images/2019/learn_metal_03.png)

上图给出了输入坐标到标准设备坐标之间的转换

由于我们要绘制的是一个2D三角形，因此不需要齐次坐标系，将w坐标值设置为1，其他都设置为0。这样我们的坐标就已经在标准化的坐标空间中了，顶点着色器只需要生成x，y坐标即可，将输入坐标除以viewport的一半就可以生成标准设备坐标。

```cpp
out.position = vector_float4(0.0, 0.0, 0.0, 1.0);
out.position.xy = pixelSpacePosition / (viewportSize / 2.0);
```

对于顶点颜色来说，我们只需要直接赋值即可

```cpp
out.color = vertices[vertexID].color;
```

至此，顶点着色器告一段落，下面来看一下片段着色器。

# 编写片段着色器

一个片段可能会修改渲染目标，光栅化会确定渲染目标的哪一部分会被图元覆盖，只有中心像素在三角形中的像素才会被渲染。

![](/assets/images/2019/learn_metal_04.png)

片段着色器会处理光栅化阶段为每个顶点生成的数据，计算每个渲染目标的输出值，这些片段值会在渲染管线后面的阶段进行处理，最终被写入渲染目标。

>Note  
一个片段被叫做一个可能的更改，是因为在片段着色器之后的阶段，生成的片段可能会被抛弃，或更改写入渲染目标的内容。本文中片段着色器生成的所有值都会按原样写入渲染目标。

片段着色器的接收参数与顶点着色器输出参数一致，声明片段着色器需要使用`fragment`关键字，输入的数据结构与顶点输出的数据结构相同，但是这里需要使用`[[stage_in]]`关键字，该关键自表明这个参数是由光栅化产生的。

```cpp
fragment float4 fragmentShader(RasterizerData in [[stage_in]])
```

如果你的片段着色器需要写入多个渲染目标，必须为每个渲染目标声明一个包含字段的结构，本文只设计一个渲染目标，所以只需要指定一个float类型的vector作为函数输出即可，该输出就是要写入渲染目标的颜色值。

光栅化阶段计算每个片段的参数值，然后通过这些值调用片段着色器。光栅化阶段通过混合三角形顶点的颜色来计算每个片段的颜色，片段距离顶点越近，该顶点颜色所占的比重越大。

![](/assets/images/2019/learn_metal_05.png)

最后返回插值后的颜色作为输出

```cpp
return in.color;
```

# 创建PSO

目前为止，顶点片段着色器已完成，需要创建渲染管线来使用它们，这个过程前面几篇文章已经介绍，这里不再赘述。

```objectivec
id<MTLLibrary> defaultLibrary = [_device newDefaultLibrary];

id<MTLFunction> vertexFunction = [defaultLibrary newFunctionWithName:@"vertexShader"];
id<MTLFunction> fragmentFunction = [defaultLibrary newFunctionWithName:@"fragmentShader"];
```

下一步，需要创建`MTLRenderPipelineState`对象，与之前不同的是，我们需要创建`MTLRenderPipelineDescriptor`来配置渲染管线。

```objectivec
MTLRenderPipelineDescriptor *pipelineStateDescriptor = [[MTLRenderPipelineDescriptor alloc] init];
pipelineStateDescriptor.label = @"Simple Pipeline";
pipelineStateDescriptor.vertexFunction = vertexFunction;
pipelineStateDescriptor.fragmentFunction = fragmentFunction;
pipelineStateDescriptor.colorAttachments[0].pixelFormat = mtkView.colorPixelFormat;

_pipelineState = [_device newRenderPipelineStateWithDescriptor:pipelineStateDescriptor 
                                                         error:&error];
```

除了要指定顶点和片段着色函数之外，我们还需要指定渲染管线绘制的渲染目标的像素格式。`MTLPixelFormat`定义了像素的内存布局，包括每个像素的字节数，像素中数据的通道数，以及这些通道的位布局。本文中的例子只有一个渲染目标，所以简单的拷贝视图的像素格式到渲染管线描述对象即可。同时，渲染管线对象使用的像素格式必须与`render pass`指定的相同，本文中都使用视图的像素格式，所以它们总是相同的。

当Metal创建渲染管线状态对象时，渲染管线会将片段着色器的输出转换为渲染目标的像素格式，如果你想要指定一个不同的像素格式，就需要创建不同的管道状态对象，你可以在针对不同像素格式的多个管道中重复使用相同的着色器。

# 设置视图窗口大小

现在，我们已经有了渲染管线状态对象，已经可以渲染三角形了。我们需要做的就是使用渲染命令编码器，设置viewport，这样Metal才知道你想要在渲染目标的哪一部分绘制。

```objectivec
// Set the region of the drawable to draw into.
[renderEncoder setViewport:(MTLViewport){0.0, 0.0, _viewportSize.x, _viewportSize.y, 0.0, 1.0}];
```

# 设置管线状态

不多说，必须的一步，Metal里就是这么规定的

```objectivec
[renderEncoder setRenderPipelineState:_pipelineState];
```

# 发送参数到顶点函数

通常我们使用`MTLBuffer`来传送数据到着色器，然而当数据量很小的时候我们可以把数据直接拷贝到命令缓冲区。本文使用的就是直接拷贝，由于片段着色器使用的是光栅化输出值，因此不需要设置它的参数。

```objectivec
[renderEncoder setVertexBytes:triangleVertices 
                       length:sizeof(triangleVertices) 
                      atIndex:AAPLVertexInputIndexVertices];

[renderEncoder setVertexByte:&_viewportSize
                      length:sizeof(_viewportSize)
                     atIndex:AAPLVertexInputIndexViewportSize];

```

# 编码绘制命令

指定绘制的图元，开始索引和顶点数，当三角形绘制时，顶点着色器函数的vertexID参数，会分别以0，1，2进行调用。

```objectivec
//draw the triangle.
[renderEncoder drawPrimitives:MTLPrimitiveTypeTriangle
                  vertexStart:0
                  vertexCount:3];
```

我们可以使用相同的步骤编码更多的渲染命令，最终的图像看起来就好像是按照指定的顺序渲染的一样。（为了提高性能，GPU允许并行执行部分命令，只要最终结果看起来是按顺序呈现的即可）

# 体验颜色插值

本文中，颜色值在三角形中进行插值，这通常是我们想要的，但是有时候我们想要一个顶点生成一个值，并在整个图元时保持不变。这时候就需要另一个属性限定符`flat`，找到RasterizerData的定义，将`[[flat]]`限定符添加到它的颜色字段。

```cpp
float4 color [[flat]];
```

然后我们就会发现，渲染管线在三角形上均匀的使用第一个顶点(成为激发顶点)的颜色值，忽略另外连个顶点的颜色。我们可以使用平面着色和插值混合，只需要通过添加或省略flat限定符即可。[The Metal Shading Language](https://developer.apple.com/metal/Metal-Shading-Language-Specification.pdf)也定义了可以修改光栅化行为的其他属性限定符。

有兴趣的朋友可自行研究。

### TO BE CONTINUED