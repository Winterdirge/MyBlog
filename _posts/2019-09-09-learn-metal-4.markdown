---
layout:     post
title:      Metal探索之旅(四)
subtitle:   Synchronizing CPU and GPU Work
date:       2019-09-09 11:08:45 GMT+8
author:     "Yang Guang"
header-style: text
tags:
    - iOS
    - 技术
    - 渲染
    - GPU
    - Metal
---

### 前面

上一篇文章我们介绍了使用Metal来绘制一个简单的彩色三角形，本篇文章的主要内容是绘制一堆这样的三角形，并使三角形以正弦曲线的方式排列起来。通过本文，你将会学到如何管理数据依赖关系，并且避免CPU和GPU由于交换数据引起的处理暂停。

![](/assets/images/2019/learn_metal_06.png)

### 了解数据依赖和处理器暂停的解决方案

资源共享会在处理器之间创建数据依赖，GPU读取数据前，CPU必须完成写入操作。如果GPU读取数据在CPU写入之前，那么GPU就会读取未定义的资源数据。如果GPU在CPU写入时读取数据，那么就会读取错误的资源数据。

![](/assets/images/2019/learn_metal_07.png)

这些个数据之间的依赖关系就导致了CPU和GPU之间的处理暂停，每个处理器必须要等待另一个完成之后才可以开始自己的工作。

然而，GPU和GPU是独立的处理器，你可以通过使用一个资源的多个实例让它们同时工作。每一帧，你必须向着色器提供相同的参数，但是这并不意味着你需要引用相同的资源对象。相反，你可以创建包含一个资源不同实例的池子，每当进行渲染时使用不同的实例。例如，CPU向缓冲区写入渲染n+1帧所需要的位置数据，同时GPU从缓冲区读取第n帧渲染所需要的位置信息。通过使用一个缓冲区的不同实例，只要你持续进行渲染，CPU和GPU就可以同时工作，避免暂停。

![](/assets/images/2019/learn_metal_08.png)

### 使用CPU初始化数据

和上篇文章一样，初始化一个结构体用来存储顶点数据，结构体有两个参数一个是顶点位置，一个是顶点颜色。

```cpp
typedef struct {
    vector_float2 position;
    vector_float4 color;
} AAPLVertex;
```

同样，构建三角形的三个顶点信息。

```objc
+ (const AAPLVertex *)vertices {
    const float TriangleSize = 64;
    static const AAPLVertex triangleVertices[] = {
        //pixel positions                              RGBA colors
        { { -0.5 * TriangleSize, -0.5 * TriangleSize }, { 1, 1, 1, 1 } },
        { {  0.0 * TriangleSize, +0.5 * TriangleSize }, { 1, 1, 1, 1 } },
        { { +0.5 * TriangleSize, -0.5 * TriangleSize }, { 1, 1, 1, 1 } }
    };
    return triangleVertices;
}
```

由于我们要绘制很多个这样的三角形，所以需要提供每个三角形的位置以及颜色信息。

```objc
NSMutableArray *triangles = [[NSMutableArray alloc] initWithCapacity:NumTriangles];

// Initialize each triangle.
for(NSUInteger t = 0; t < NumTriangles; t++)
{
    vector_float2 trianglePosition;

    // Determine the starting position of the triangle in a horizontal line.
    trianglePosition.x = ((-((float)NumTriangles) / 2.0) + t) * horizontalSpacing;
    trianglePosition.y = 0.0;

    // Create the triangle, set its properties, and add it to the array.
    AAPLTriangle * triangle = [AAPLTriangle new];
    triangle.position = trianglePosition;
    triangle.color = Colors[t % NumColors];
    [triangles addObject:triangle];
}
_triangles = triangles;
```

### 分配存储空间

计算需绘制三角形的所有顶点所需要的内存空间大小，你需要绘制50个三角形，每个三角形有3个顶点，总共有150个顶点，每个顶点所占空间即为结构体`AAPLVertex`所占空间大小。

```objc
const NSUInteger triangleVertexCount = [AAPLVertex vertexCount];
_totalVertexCount = triangleVertexCount * _triangles.count;
const NSUInteger triangleVertexBufferSize = _totalVertexCount * sizeof(AAPLVertex);
```

初始化多个缓冲区用来存储顶点数据的多个拷贝，对于每个缓冲区来说，都需要申请足够的内存来存储150个顶点信息。

```objc
for (NSUInteger bufferIndex = 0; bufferIndex < MaxFramesInFlight; bufferIndex++) {
    _vertexBuffers[bufferIndex] = [_device newBufferWithLength:triangleVertexBufferSize
                                                       options:MTLResourceStorageModeShared];
    _vertexBuffers[bufferIndex].label = [NSString stringWithFormat:@"Vertex Buffer #%lu", (unsigned long)bufferIndex];
}
```

初始化之前，`_vertexBuffers`数组中的缓冲区实例的内容都为空。

### 使用CPU更新数据

在每帧，`draw(in:)`渲染循环开始时，需要使用CPU更新缓冲区实例中的数据内容。

```objc
// Vertex data for the current triangles.
AAPLVertex *currentTriangleVertices = _vertexBuffers[_currentBuffer].contents;

// Update each triangle.
for(NSUInteger triangle = 0; triangle < NumTriangles; triangle++)
{
    vector_float2 trianglePosition = _triangles[triangle].position;

    // Displace the y-position of the triangle using a sine wave.
    trianglePosition.y = (sin(trianglePosition.x/waveMagnitude + _wavePosition) * waveMagnitude);

    // Update the position of the triangle.
    _triangles[triangle].position = trianglePosition;

    // Update the vertices of the current vertex buffer with the triangle's new position.
    for(NSUInteger vertex = 0; vertex < triangleVertexCount; vertex++)
    {
        NSUInteger currentVertex = vertex + (triangle * triangleVertexCount);
        currentTriangleVertices[currentVertex].position = triangleVertices[vertex].position + _triangles[triangle].position;
        currentTriangleVertices[currentVertex].color = _triangles[triangle].color;
    }
}
```

在你更新完一个缓冲区数据后，在这一帧的剩余时间内你无法通过CPU去获取该缓冲区的数据。

>Note:  
在你提交命令缓冲并引用一个数据缓冲区实例之前，必须要完成所有的CPU写入操作。否则，在CPU写入的同时，GPU可能开始读取这个缓冲，进而引起错误。

### 编码GPU命令

然后，我们需要在渲染过程中编码渲染命令，该命令会使用上一步中的缓冲区。

```objc
[renderEncoder setVertexBuffer:_vertexBuffers[_currentBuffer]
                        offset:0
                       atIndex:AAPLVertexInputIndexVertices];

// Set the viewport size.
[renderEncoder setVertexBytes:&_viewportSize
                       length:sizeof(_viewportSize)
                      atIndex:AAPLVertexInputIndexViewportSize];

// Draw the triangle vertices.
[renderEncoder drawPrimitives:MTLPrimitiveTypeTriangle
                  vertexStart:0
                  vertexCount:_totalVertexCount];
```