---
layout:     post
title:      Conda速记
subtitle:   
date:       2020-02-10 15:18:00 GMT+8
author:     "Yang Guang"
header-style: text
tags:
    - 技术
---

>翻译自官方文档，只是为了记录一下

## Conda命令

Conda命令是管理各种软件包安装的主要接口，它可以
* 可查询Anaconda软件包索引以及当前已安装的Anaconda包
* 创建新的conda环境
* 在已有的conda环境下安装和更新包

>Tip: 我们可以将许多常用命令前面的两个破折号简写为一个并加上该选项的第一个首字母，例如，--name和-n，--envs和-e相同。

## Conda packages

conda软件包是压缩的tarball文件(.tar.bz2)或.conda文件，它包括
* 系统级别的库
* Python或其他模块
* 可执行程序和其他组件
* info目录下的元数据
* 直接安装到安装前缀中的文件集合

conda跟踪软件包和平台的依赖关系，conda包格式在平台和操作系统之间是相同的。
只有文件(包括符号链接)是conda软件包的一部分，目录则不包括在内，目录根据需要创建和删除，但是我们不能直接从一个tar存档创建一个空目录。

### .conda文件格式

.conda文件格式是conda4.7引入的，它比压缩包更紧凑，因此速度更快。

为了支持.conda格式的初始内部压缩格式，我们选择了Zstandard(zstd)。只要libarchive支持该格式，实际使用什么压缩格式都没有关系。随着更高级的压缩算法的发展，压缩格式可能会在将来发生变化，但是我们并不需要更改.conda格式，只需要更新libarchieve即可为.conda文件添加新的压缩格式。

这些压缩文件可以比其等效的bzip2文件小得多。此外，它们的解压缩速度更快。.conda是首选的文件格式（尽管我们会继续提供串联的.tar.bz2文件），但要尽可能使用.conda格式。

[更多关于.conda文件格式](https://www.anaconda.com/understanding-and-improving-condas-performance/)

>conda 4.7及之后，我们不能使用.conda结尾的软件包名，因为那样会和软件包的.conda文件冲突

### 使用软件包

搜索软件包
```
conda search scipy
```
安装软件包
```
conda install scipy
```
安装[conda-build](https://docs.conda.io/projects/conda-build/en/latest/index.html)之后，我们可以编译自己的软件包
```
conda build my_fun_package
```

### 包结构
```
.
├── bin
│   └── pyflakes
├── info
│   ├── LICENSE.txt
│   ├── files
│   ├── index.json
│   ├── paths.json
│   └── recipe
└── lib
    └── python3.5
```
* bin包含包的相关二进制文件
* lib包含相关的库文件，例如.py文件
* info包含包的元数据

### 元包

当conda包仅用于元数据并且不包含任何文件时，将其称为元包。 元软件包可能包含对几个核心低级库的依赖关系，并且可能包含指向在执行时自动下载的软件文件的链接。 元数据包用于捕获元数据，并使复杂的数据包规范更简单。

元软件包的一个示例是anaconda，它会将Anaconda安装程序中的所有软件包收集在一起。 命令`conda create -n envname anaconda`创建的环境与从Anaconda安装程序创建的环境完全匹配。 您可以使用`conda metapackage`命令创建元包。


#### Anaconda元包
Anaconda元包用于[Anaconda Distribution](https://docs.anaconda.com/anaconda/)安装程序的创建，因此它们具有一组与之关联的软件包。每个安装程序版本都有一个版本号，该版本号对应于特定版本的特定软件包集合。特定版本的软件包集合封装在Anaconda元软件包中。

Anaconda元软件包包含几个核心的低层库，包括压缩，加密，线性代数和一些GUI库。

[阅读有关Anaconda元软件包和Anaconda发行版的更多信息](https://www.anaconda.com/whats-in-a-name-clarifying-the-anaconda-metapackage/)。

#### 互斥量元包
互斥量元软件包是一个非常简单的具有名称的软件包。它不需要任何依赖关系或构建步骤。互斥量元软件包通常是配方中的“输出”，该配方构建了另一个软件包的某些变体。互斥量元软件包可作为一种工具来帮助实现具有不同名称的软件包之间的互斥。

让我们看一些有关如何使用互斥量元数据包针对不同的BLAS实现构建NumPy的示例。

##### 使用BLAS变体构建NumPy

如果使用MKL构建NumPy，则还需要使用MKL一起构建SciPy，scikit-learn以及其他任何使用BLAS的东西。重要的是要确保将这些“变量”（使用一组特定的选项构建的软件包）安装在一起，并且切勿与其他BLAS实现一起安装。这是为了避免崩溃，运行缓慢或数值问题。排列这些库既是构建时的问题，也是安装时的问题。我们将展示如何使用元数据包来满足这一需求。

让我们从元数据包`blas=1.0=mkl`开始：https://github.com/AnacondaRecipes/intel_repack-feedstock/blob/e699b12/recipe/meta.yaml#L108-L112

请注意，`mkl`是一串`blas`。

当有人将mkl-devel软件包用作构建时依赖项时，该metapackage将使用`run_exports`作为依赖项自动添加：https://github.com/AnacondaRecipes/intel_repack-feedstock/blob/e699b12/recipe/meta.yaml#L124

同样，这是OpenBLAS的元包：https://github.com/AnacondaRecipes/openblas-feedstock/blob/ae5af5e/recipe/meta.yaml#L127-L131

和OpenBLAS的`run_exports`，作为openblas-devel的一部分：https://github.com/AnacondaRecipes/openblas-feedstock/blob/ae5af5e/recipe/meta.yaml#L100

从根本上说，conda的互斥模型取决于包装名称。OpenBLAS和MKL显然不是相同的软件包名称，因此并不互斥。没有什么可以阻止conda一次安装。没有什么可以阻止conda通过MKL安装NumPy和通过OpenBLAS安装SciPy。元包使我们能够实现互斥。它统一了单个程序包名称上的选项，但具有不同的构建字符串。使用run_exports自动添加metapackage有助于确保库使用者（依赖库的包生成器）将具有正确的依赖项信息，以实现统一的运行时库集合。

##### 使用BLAS变体安装NumPy
要指定所需的NumPy变体，可以潜在地指定所需的BLAS库：
```
conda install numpy mkl
```
但是，这实际上并不妨碍选择OpenBLAS。 MKL和它的依赖项都不是互斥的（这意味着它们没有相似的名称和不同的版本/构建字符串）。

此途径可能导致混合BLAS产生一些歧义和解决方案，因此建议使用metapackage。要以明确的方式指定由MKL驱动的NumPy，可以指定互斥体程序包（直接或间接）：

```
conda install numpy "blas=*=mkl"
```

但是，有一种更简单的方法可以解决此问题。例如，您可能想尝试另一个具有所需互斥体软件包作为依赖项的软件包。

OpenBLAS的“ nomkl”软件包具有以下功能：https://github.com/AnacondaRecipes/openblas-feedstock/blob/ae5af5e/recipe/meta.yaml#L133-L147

不应将"nomkl"用作依赖项。严格来说，这是一个实用程序，方便用户从MKL（默认）切换到OpenBLAS。

MKL如何成为默认值？求解器需要一种优先于某些软件包优先于其他软件包的方法。我们通过一个较旧的conda功能（称为track_features）实现了这一目的，该功能最初用于其他目的。

##### Track_features
conda的优化目标之一是最大程度地减少指定所需规格所需的track_features数量。通过将track_features添加到一个或多个选项中，conda会将其优先级降低或“权衡”。最低优先级的软件包是将导致环境中激活大多数track_features的软件包。许多变种中的默认软件包是将导致最少track_features激活的软件包。

但是有一个陷阱：任何track_features必须是唯一的。没有两个软件包可以提供相同的track_feature。因此，我们的标准做法是将track_features附加到与我们希望成为非默认值的内容相关联的metapackage。

再看一下OpenBLAS食谱：https://github.com/AnacondaRecipes/openblas-feedstock/blob/ae5af5e/recipe/meta.yaml#L127-L137

此附加的track_features条目是为什么在OpenBLAS上选择MKL的原因。 MKL没有任何与其关联的track_features。如果有3个选项，则将0个track_features附加到默认选项，然后将1个track_features附加到下一个首选选项，最后将2个附加到最不喜欢的选项。但是，由于您通常只关心一个默认值，所以通常在默认选项之外的所有选项中添加1个track_feature就足够了。

##### 更多信息
作为参考，Windows上的Visual Studio版本对齐方式也使用互斥量元软件包。 https://github.com/AnacondaRecipes/aggregate/blob/9635228/vs2017/meta.yaml#L24

### Noarch软件包
Noarch软件包是不特定于体系结构的软件包，因此仅需构建一次。 Noarch软件包是通用软件包或Python软件包。 Noarch通用软件包允许用户在conda软件包中分发文档，数据集和源代码。 Noarch Python软件包如下所述。

在`meta.yaml`的`build`部分中将这些程序包声明为`noarch`会减少共享的CI资源。因此，所有符合资格成为noarch软件包的软件包都应这样声明。

#### Noarch Python
build部分中的`noarch:python`指令生成仅需构建一次的纯Python包。

通过在安装时整理平台和特定于Python版本的差异，Noarch Python软件包减少了在不同体系结构和Python版本上构建多个不同纯Python软件包的开销。

为了符合Noarch Python软件包的要求，必须满足以下所有条件：

* 没有编译的扩展。
* 没有后链接，前链接或前取消链接脚本。
* 没有特定于操作系统的构建脚本。
* 没有特定于Python版本的要求。
* 除Python版本外，不跳过。如果配方仅是py3，请删除“跳过”语句，并在“主机和运行”部分的Python上添加版本约束。
* 不使用2to3。
* 不使用setup.py中的脚本参数。
* 如果`console_script`入口点位于setup.py中，则它们将列在`meta.yaml`中。
* 没有激活脚本。
* 不依赖于conda。

>注意
尽管`noarch：python`不适用于选择器，但适用于版本限制。`skip: True ＃ [py2k]`有时可以在主机中运行受限的Python版本并运行小节，例如：`python >= 3`，而不仅仅是`python`。
仅`console_script`入口点必须在`meta.yaml`中列出。其他入口点与`noarch`不冲突，因此不需要额外的处理。

阅读有关conda的noarch软件包的[更多信息](https://www.anaconda.com/condas-new-noarch-packages/)。

### 链接和取消链接脚本
您可以选择在链接和取消链接步骤之前和之后执行脚本。有关更多信息，请参阅[添加前链接，后链接和前取消链接脚本](https://docs.conda.io/projects/conda-build/en/latest/resources/link-scripts.html)。

### 更多信息
进一步了解如何[管理软件包](https://conda.io/projects/conda/en/latest/user-guide/getting-started.html#managing-pkgs)。在[软件包规范](https://conda.io/projects/conda/en/latest/user-guide/concepts/pkg-specs.html)中了解有关软件包元数据，存储库结构和索引以及软件包匹配规范的更多信息。
