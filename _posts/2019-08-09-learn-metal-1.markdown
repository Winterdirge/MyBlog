---
layout:     post
title:      Metal探索之旅(一)
subtitle:   使用Metal进行并行计算
date:       2019-08-09 20:01:55 GMT+8
author:     "Yang Guang"
catalog:    true
header-style: text
tags:
    - iOS
    - 技术
    - 渲染
    - 并行计算
    - GPU
    - Metal
---

>本文内容来源于苹果官方文档，主要介绍如何使用图像处理器渲染高级3D图形并执行数据并行计算

# 前面

图形处理器(GPU:Graphics processeror)旨在快速渲染图形并执行并行计算，如果你想要直接与设备上的可用GPU通信，那么请使用Metal框架，渲染复杂的3D场景或执行高级科学计算时可使用Metal来达到最佳性能。

Metal可以与其他框架密切配合，使用MetalKit可以简化在屏幕上获取Metal内容的流程，使用"Metal Performance Shader"可以自定义渲染任务以及利用现有的功能库。

许多高级的Apple框架都是基于Metal构建的，包括Core Image、SpriteKit、SceneKit。使用这些高级框架可以使你免受GPU编程细节的影响，但是编写自定义的Metal代码可以使你获得高效的性能。

下面我们会通过几个官方的例子来熟悉一下Metal的工作流程。

# 使用GPU执行计算

## 基本步骤

* 将一个简单的C函数转换为MSL(Metal Shading Language)
* 找到一个可用GPU
* 创建pipeline准备在GPU上运行MSL函数
* 申请一块GPU可以访问的存储空间用来存储GPU处理的数据
* 创建一个命令缓存(command buffer)，编码GPU命令用来处理数据
* 将command buffer提交到commannd queue，GPU读取queue中的命令进行执行

## 将C函数转换为MSL

以简单的向量相加为例来进行说明，首先一个C语言编写简单的加法函数：

```c
void add_arrays(const float* inA, const float* inB, float* result, int length) {
    for (int index = 0; index < length; ++index) {
        result[index] = inA[index] + inB[index];
    }
}
``` 

可以看到，结果向量中的每个元素都是单独计算的，所以可以安全的同时对它们进行计算。想要在GPU中完成这个工作，首先需要将上面的方法`翻译为`GPU可以看懂的语言，也就是MSL。MSL是为GPU编程而设计的一种C++的变种语言，书写起来和C++有点像，但是它的后缀名是`.metal`。在Metal中，在GPU上运行的代码叫做`着色器`，因为历史上它们首先用于计算3D图形中的颜色。

Xcode构建Metal文件时会创建一个默认的Metal库，并将其嵌入到应用程序中，我们可以从这个库中获取`.metal`文件中编写的函数的指针。

改写后的MSL代码如下：

```cpp
#include <metal_stdlib>
using namespace metal;
/// This is a Metal Shading Language (MSL) function equivalent to the add_arrays() C function, used to perform the calculation on a GPU.
kernel void add_arrays(device const float* inA, 
                       device const float* inB, 
                       device float* result,
                       uint index [[thread_position_in_grid]]) {
    //the for-loop is replaced with a collection of threads, each of which calls this function.
    result[index] = inA[index] + inB[index];
}
```

对比上面两段代码，我们可以看到它们之间的明显差异。

首先，`kernel`关键字表明

* 该函数是一个公有的GPU函数，公有函数是你的App可以看到的唯一函数，这种函数不能被其他着色器方法调用。

* 计算函数（又叫计算内核compute kernel）通过线程网格来执行并行计算。

`device`关键字，该关键字表明指针位于设备的地址空间中。MSL为内存定义了几个不相交的地址空间，每当在MSL中声明指针时，都必须提供一个关键字来声明其地址空间，使用设备的地址空间可以声明GPU可以读写的持久内存。

其次，MSL代码中移除了for循环，改为`compute grid`中的多个线程调用，方便向量中的元素由不同线程计算。同时还要使用新的索引，并指定关键字`thread_position_in_grid`，该关键字表明Metal应该为每个线程计算唯一索引，并在此参数中传递该索引。因为`add_arrays`使用1D网格（1D grid），所以索引定义为标量整数。如果要将类似的代码从C或者C++转换为MSL，需要用`grid`替换循环逻辑。

## 找到可用GPU

在App中，MTLDevice对象是对GPU的精简抽象，我们将使用它来和GPU通信，Metal为每个GPU创建一个`MTLDevice`，可以通过`MTLCreateSystemDefaultDevice()`获取默认设备对象，在MACOS中，Mac可以有多个GPU，Metal选择其中一个GPU作为默认值返回，也可以通过API获取其他设备对象，不过此处仅使用默认值作为示例。

```objc
id<MTLDevice> device = MTLCreateSystemDefaultDevice();
```

## 初始化Metal对象

`Metal`将其他与GPU相关的实体（编译过的着色器、内存缓冲和纹理）表示为对象，要创建这些特定的GPU对象，可以在`MTLDevice`上直接调用方法，或者在`MTLDevice`创建的对象上调用方法。 由设备对象直接或间接创建的所有对象仅可用于该设备对象。 使用多个GPU的应用程序将使用多个设备对象，并为每个设备创建类似的Metal对象层次结构。


### 获取Metal函数引用


想要在GPU上执行方法，所做的第一件事是加载该方法，告诉GPU这个方法在哪里。 

构建App时，Xcode会编译`add_arrays`函数并将其添加到App默认Metal库中。 我们可以使用`MTLLibrary`和`MTLFunction`对象来获取Metal库及其中包含的函数信息。 想要在GPU中获取表示`add_arrays`函数的对象，需要使用`MTLDevice`创建`MTLLibrary`对象，然后从该对象中获取表示着色器函数的`MTLFunction`对象。方法如下：


```objc

id<MTLLibrary> defaultLibrary = [device newDefaultLibrary];

if (defaultLibrary == nil)
{
    NSLog(@"Failed to find the default library.");
    return nil;
}

id<MTLFunction> addFunction = [defaultLibrary newFunctionWithName:@"add_arrays"];
if (addFunction == nil)
{
    NSLog(@"Failed to find the adder function.");
    return nil;
}

```


### 准备Metal管线


函数对象是MSL函数的代理，但它不是可执行代码。

我们可以通过创建管道将该函数转换为可执行代码，管道指定了GPU执行特定任务的步骤。在Metal中，管道由管道状态对象表示。本例中我们仅仅使用GPU的计算功能，因此创建`MTLComputePipelineState`对象。

```objc
NSError *error = nil;
id<MTLComputePipelineState> mAddFunctionPSO = [device newComputePipelineStateWithFunction:addFunction error:&error];
if (mAddFunctionPSO == nil)
{
    //  If the Metal API validation is enabled, you can find out more information about what
    //  went wrong.  (Metal API validation is enabled by default when a debug build is run
    //  from Xcode)
    NSLog(@"Failed to created pipeline state object, error %@.", error);
    return nil;
}
```

计算管道执行单一的计算功能，我们可以控制方法传入的数据，使用方法的输出数据。

创建管道状态对象时，设备对象将为该GPU完成方法的编译。该例子中我们将同步创建管道状态对象并将其直接返回给应用程序，但是需要注意编译确实需要一段时间，所以避免在性能敏感的代码中同步创建管道状态对象。

>注意：
到目前为止我们的代码中返回的对象都是作为符合相关协议的对象返回。Metal使用协议来定义大多数GPU相关对象，以抽象掉底层的具体实现，这些类的实现可能因不同的GPU而异。Metal使用类定义与GPU无关的对象，任何给定的Metal协议的参考文档都清楚地表明我们是否可以在应用程序中实现该协议，所以不会的时候要看看文档。


### 创建命令队列


要将工作发送到GPU，我们需要一个命令队列，Metal使用命令队列来安排命令。

```objc
id<MTLCommandQueue> mCommandQueue = [device newCommandQueue];
```

#### 创建数据缓冲加载数据


完成基本Metal对象初始化之后，需要加载GPU执行的数据。

GPU可以拥有自己的专用内存，也可以与操作系统共享内存。想要将数据存储在内存中并且GPU可以顺利访问，Metal和操作系统内核需要执行额外的工作，Metal使用`MTLResource`来抽象上述的内存操作，同样我们使用`MTLDevice`创建`MTLResource`。

```objc

const unsigned int arrayLength = 1 << 24;
const unsigned int bufferSize = arrayLength * sizeof(float);

id<MTLBuffer> mBufferA = [device newBufferWithLength:bufferSize options:MTLResourceStorageModeShared];
id<MTLBuffer> mBufferB = [device newBufferWithLength:bufferSize options:MTLResourceStorageModeShared];
id<MTLBuffer> mBufferResult = [device newBufferWithLength:bufferSize options:MTLResourceStorageModeShared];

[self generateRandomFloatData:mBufferA];
[self generateRandomFloatData:mBufferB];

```

例子中的资源是`MTLBuffer`对象，它们是没有预定义格式的一段内存，Metal将每个这样的缓冲区视为不透明的字节集合。但是，在着色器中使用缓冲区时需要指定格式，这意味着着色器和应用传递的数据格式一定要一致。

分配缓冲区时，您需要显式提供存储模式以确定其某些性能特征以及CPU或GPU是否可以访问它，该例子使用共享内存`MTLResourceStorageModeShared`模式，这意味着CPU和GPU都可以访问它。

使用随机数据填充缓冲区，一定要注意格式，上面分配的是`arrayLength`个float，此处随机填充时一定也要是相同数量的float。

```objc
- (void)generateRandomFloatData:(id<MTLBuffer>)buffer
{
    float* dataPtr = buffer.contents;
    
    for (unsigned long index = 0; index < arrayLength; index++)
    {
        dataPtr[index] = (float)rand()/(float)(RAND_MAX);
    }
}
```


#### 创建命令缓冲区


使用命令队列创建命令缓冲

```objc
id<MTLCommandBuffer> commandBuffer = [mCommandQueue commandBuffer];
```


#### 创建命令编码器


想要将命令写入命令缓冲区，需要使用命令编码器来编码需要执行的命令。该例子创建一个计算命令编码器，用于编码计算过程。一个计算过程包含执行计算管道的命令列表，每个计算命令都会使GPU创建一个在GPU上执行的线程网格。

```objc
id<MTLComputeCommandEncoder> computeEncoder = [commandBuffer computeCommandEncoder];
```

要对命令进行编码，需要在编码器上执行一系列方法调用。某些方法用来设置状态信息，如管道状态对象`PSO`或要传递给管道的参数。当你更改了这些状态之后，就可以将命令编码然后执行管线。编码器会将所有的状态更改，以及命令参数写入命令缓冲区。

![](/assets/images/2019/learn_metal_01.png)


#### 设置管线状态及参数


设置要执行命令的管道的管道状态对象，然后为管道需要发送到`add_arrays`的每一个参数设置数据。例子中的管道需要提供对三个缓冲区的引用，Metal按照参数在`add_arrays`函数声明中出现的顺序自动分配缓冲区参数的索引，从0开始。

```objc
[computeEncoder setComputePipelineState:mAddFunctionPSO];
[computeEncoder setBuffer:mBufferA offset:0 atIndex:0];
[computeEncoder setBuffer:mBufferB offset:0 atIndex:1];
[computeEncoder setBuffer:mBufferResult offset:0 atIndex:2];
```

你也可以为每个参数指定一个偏移量，偏移量为0表示命令将会从buffer的开始地址读取数据。然而，你可以使用一个buffer存储多个参数，然后为每个参数设置偏移量。我们没有为index参数指定任何数据，因为`add_arrays`函数将其值定义为由GPU提供。


#### 线程数



接下来，确定要创建的线程数以及如何组织这些线程。Metal可以创建1D，2D或3D网格，`add_arrays`函数使用1D数组，因此示例创建一个大小为1D的网格（dataSize x 1 x 1），Metal从该网格生成0到dataSize-1之间的索引。

```objc
MTLSize gridSize = MTLSizeMake(arrayLength, 1, 1);
```

#### 线程组大小


Metal将网格细分为一种叫做线程组的较小网格，每个线程组都是单独计算的，Metal可以将线程组分派到GPU上的不同处理单元，以加快处理速度。

```objc
NSUInteger threadGroupSize = mAddFunctionPSO.maxTotalThreadsPerThreadgroup;
if (threadGroupSize > arrayLength)
{
    threadGroupSize = arrayLength;
}
MTLSize threadgroupSize = MTLSizeMake(threadGroupSize, 1, 1);
```

应用程序向管道状态对象请求最大可能的线程组，并在该大小大于数据集大小时减小它，`maxTotalThreadsPerThreadgroup`属性给出了线程组中允许的最大线程数，这取决于用于创建管道状态对象的函数的复杂性。


#### 设置执行线程数及线程组大小


最后，编码命令然后分发给线程网格。

```objc
[computeEncoder dispatchThreads:gridSize threadsPerThreadgroup:threadgroupSize];
```

当GPU执行此命令时，它使用您先前设置的状态和命令参数来分派线程以执行计算。

您可以采用相同的步骤使用编码器将多个计算命令编码到计算过程中，而不需要执行任何冗余步骤。例如，可以设置一次管道状态对象，然后为要处理的每个缓冲区集合设置参数并编码命令。

#### 结束编码

如果没有其他命令要添加到计算过程，则结束编码过程。

```objc
[computeEncoder endEncoding];
```


#### 开始执行

通过将命令缓冲区提交到队列来运行命令缓冲区中的命令。

```objc
[commandBuffer commit];
```

命令队列创建了命令缓冲区，因此提交的缓冲区始终在该队列上。提交命令缓冲区后，Metal准备异步执行命令，GPU执行命令缓冲区中的所有命令后，Metal将命令缓冲区标记为完成。

## 等待执行结束

当GPU处理您的命令时，您的应用程序可以执行其他工作。 该例子不需要执行任何其他工作，因此只需等待命令缓冲区完成。

```objc
[commandBuffer waitUntilCompleted];
```

或者，等到Metal处理完所有命令时发送通知，也可以使用`addCompletedHandler（_ :)`，或通过读取其status属性来检查命令缓冲区的状态。

## 读取结果

命令缓冲区完成后，GPU将计算结果存储在输出缓冲区中，并且Metal会执行必要的步骤以确保CPU可以看到它们。在真实的应用程序中，我们会从缓冲区读取结果并对其执行某些操作，例如在屏幕上显示结果或将其写入文件。该例子仅仅会读取存储在输出缓冲区中的值，并进行测试以确保CPU和GPU计算出相同的结果。

```objc
- (void) verifyResults
{
    float* a = _mBufferA.contents;
    float* b = _mBufferB.contents;
    float* result = _mBufferResult.contents;

    for (unsigned long index = 0; index < arrayLength; index++)
    {
        assert(result[index] == a[index] + b[index]);
    }
}
```

# 总结

本文主要介绍了在GPU上执行计算的基本步骤，以及GPU相关的一些对象。总的来说使用GPU计算需要以下步骤：

1. 写MSL Shader，文件需要以`.metal`为后缀
2. 获取可用GPU`MTLCreateSystemDefaultDevice()`
3. 获取编译shader后的library`newDefaultLibrary`
4. 在library中找到该方法`[defaultLibrary newFunctionWithName:@"add_arrays"]`
5. 创建PSO对象`[device newComputePipelineStateWithFunction:addFunction error:&error]`，该对象会对上一步中获得的方法进行编译
6. 创建命令队列`[device newCommandQueue]`
    * 通过该队列常见命令缓冲区`[mCommandQueue commandBuffer]`
    * 通过命令缓冲区创建命令编码器`[commandBuffer computeCommandEncoder]`
    * 使用编码器编码命令
        1. 设置PSO `[computeEncoder setComputePipelineState:_mAddFunctionPSO]`
        2. 设置输入输出buffer数据
            * 需要创建输入输出的buffer并指定它们的共享模式
        3. 设置线程数，以及threadgroup size
        4. 完成编码`[computeEncoder endEncoding]`
7. 提交命令缓冲进行执行`[commandBuffer commit]`
8. 获取执行结果
