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