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
    - CPU
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

### 提交并执行GPU命令

不多说，上几篇文章说过

```objc
[commandBuffer commit];
```

然后GPU开始工作，在顶点着色器`RasterizerData vertex shader`中读取顶点缓冲区，该顶点着色器会将顶点缓冲区作为其中一个输入参数。

```objc
vertex RasterizerData
vertexShader(const uint vertexID [[ vertex_id ]],
             const device AAPLVertex *vertices [[ buffer(AAPLVertexInputIndexVertices) ]],
             constant vector_uint2 *viewportSizePointer  [[ buffer(AAPLVertexInputIndexViewportSize) ]])
```

### 在你的APP中复用多个缓冲区实例

和上面描述的一样，每一帧都要执行以下几步，当两个处理器都完成它们的工作时，一个完整的帧才会完成。

1. 向缓冲对象中写入数据
2. 编码命令引用该缓冲区
3. 提交包含编码命令的命令缓冲区
4. 从缓冲区中读取数据

当一帧的工作完成后，CPU和GPU不再需要这一帧使用数据缓冲实例。然而，丢弃一个使用过的缓冲区实例，然后在每一帧重新创建一个缓冲区对象的代价较高，并且较为浪费。因此，在你的APP中使用一个FIFO队列，复用这些个缓冲区，该队列的最大长度被设置为3，由`MaxFramesInFlight`记录。

```objc
static const NSUInteger MaxFramesInFlight = 3;
```

每帧开始时，我们需要从`_vertexBuffer`队列更新下一个缓冲区内容，我们按顺序循环便利队列，每帧只更新一个渲染缓冲区实例，在每三帧结束时，重新返回队列的开头。

```objc
// Iterate through the Metal buffers, and cycle back to the first when you've written to the last.
_currentBuffer = (_currentBuffer + 1) % MaxFramesInFlight;

// Update buffer data.
[self updateState];
```

>Note:  
Core Animation提供优化过的可显示资源（通常被叫做`drawable`）方便我们渲染内容并在屏幕上显示。Drawable是高效但昂贵的系统资源，因此Core Animation限制了我们可以在程序中同时使用的drawable数量。默认限制为3，但是我们可以通过`maximumDrawableCount`属性将其设置为2（仅支持2和3）。由于最大值为3，因此我们创建3个缓冲区实例，我们不需要创建比最大可绘制对象数量更多的缓冲区。

### 管理CPU和GPU的工作速度

当一个缓冲区有多个实例时，我们可以让CPU在一个实例上为n+1帧工作，同时GPU在另一个实例上为第n帧工作。这种实现使GPU和CPU同时工作，提高了app的效率。但是，需要管理app的工作速率，以便不超过可用缓冲区实例的数量。

要管理应用程序的工作速率，请使用信号量等待整帧完成，防止CPU比GPU快太多引发错误。信号量不是一个Metal相关对象，可以使用它来多线程或多处理器访问共享资源。信号量有一个相关计数值，可以控制其递增或者递减，该值表示处理器已启动或者完成了对一个资源的访问。在我们的例子中，信号量用来控制CPU和GPU对缓冲区实例的访问。

我们使用`MaxFramesInFlight`初始化信号量，刚好匹配缓冲区实例数量，该值表示我们的应用最多同时处理3帧数据。

```objc
_inFlightSemaphore = dispatch_semaphore_create(MaxFramesInFlight);
```

渲染开始时，将信号量减一，这表明我们已经准备在新的一帧上开始工作了。如果信号量小于零，它会使CPU等待，直到你增加了信号量。

```objc
dispatch_semaphore_wait(_inFlightSemaphore, DISPATCH_TIME_FOREVER);
```

渲染结束时，我们注册了一个命令缓冲区的`completion handler`，当GPU完成命令缓冲区的命令执行后，会调用该`handler`，这时候就可以将信号量加一。这表明了我们已经完成了指定帧的所有工作，可以复用该帧的数据缓冲区了。

```objc
__block dispatch_semaphore_t block_semaphore = _inFlightSemaphore;
[commandBuffer addCompletedHandler:^(id<MTLCommandBuffer> buffer) {
     dispatch_semaphore_signal(block_semaphore);
}];
```

由于我们每帧只有一个命令缓冲区，所以当收到回调时就表明GPU已经完成了该帧的工作。

### 设置缓冲区的可变性

我们的应用程序在单个线程执行每帧的渲染设置。首先，使用CPU将数据写入缓冲区实例，之后，编码引用缓冲区实例的编码命令，最后，提交用于GPU执行的命令缓冲区。由于这些个任务是按顺序在单个线程上执行的，可以保证应用程序在编码引用缓冲区实例的命令时已经完成了缓冲区的数据写入。

此顺序允许你将缓冲区实例标记为不可变，当配置渲染管线描述符时，将缓冲区实例索引处的顶点缓冲区的`mutablility`属性设置为`MTLMutability.immutable`

```objc
pipelineStateDescriptor.vertexBuffers[AAPLVertexInputIndexVertices].mutability = MTLMutabilityImmutable;
```

Metal可以优化不可变缓冲区的性能，但不能优化可变缓冲区。 为获得最佳结果，请尽可能使用不可变缓冲区。

### TO BE CONTINUED...