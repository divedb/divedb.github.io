---
date: 2025-08-02
layout: post
title: 虚拟地址转换[四] - large page/hugepage
categories: Linux
tags: [Linux, 内存, 虚拟内存] 
---

[上文](https://zhuanlan.zhihu.com/p/64978946)提到，使用`large page`（在`Linux`中对应的概念叫`hugepage`）可以优化对多级页表的访问时间，那处理器硬件层面是如何实现`large page`的呢？

在页表的任何级别，指向下一级的指针可以为空，表示该范围内没有有效的虚拟地址。中间级别也可以有特殊条目（设置了`page size flag`），表明它们直接指向`large page`。

例如，在32位的`x86`系统中，支持`4MB`的`huge page`（$\text{4KB} \times 2^{10}= \text{4MB}$，`PD`跳过`PT`直接指向`page`）。在64位的`x86-64`中，支持`2MB`（$\text{4KB} \times 2^9 = \text{2MB}$，`PD`跳过`PT`直接指向`page`）和`1GB`（$\text{2MB} \times 2^9 = \text{1GB}$，`PDPT`跳过`PD`和`PTE`直接指向`page`）的`large page`。

![img](https://pic4.zhimg.com/v2-de3fdb5f2627cac7b074ad14256d2855_1440w.jpg)

如果使用`2MB`的`large page`， 则其页表查找过程是这样的：

![img](https://pic2.zhimg.com/v2-2a5c438982f7079a58ebaa7ac4ab1dff_1440w.jpg)

如果使用`1GB`的`large page`， 则其页表查找过程是这样的：

![img](https://pic1.zhimg.com/v2-e53052b5388134b14152145cc49f32f4_1440w.jpg)

`large page`的使用可以减少页表的级数，也就减少了查找页表的内存访问次数，而且对于同样`entries`数目的`TLB`，可以扩大`TLB`对内存地址的覆盖范围，减小`TLB miss`的概率。此外，因为页表本身也要占用内存空间，减少页表的大小也可以节约那么一丢丢的内存。

在使用`large page`的系统中，可能存在`4KB`，`2MB`，`1GB`等不同大小的`page`共存的情况，这就需要不同的`TLB`支持。以`intel`的某款处理器为例，其含有的`TLB`种类如下：

![img](https://pic4.zhimg.com/v2-0f6acc6314c0559954fbb079bcdf6b3d_1440w.jpg)

当然，使用`large page`也会带来一些问题，比如：

1.  由于各种内存操作基本都要求按照`page`对齐，比如一个可执行文件映射到进程地址空间，根据文件大小的不同，平均算下来会浪费掉半个`page size`的物理内存，使用`large page`的话这个消耗就显得比较大了。
2.  系统运行一段时间后，会很难再也大块的连续物理内存，这时分配`large page`将会变的很困难，所以通常需要在系统初始化的时候就划分出一段物理内存给`large page`用（类似于`DMA`的内存分配），这样就减少了一些灵活性。
3.  动态`large page`（[THP](https://zhida.zhihu.com/search?content_id=102890488&content_type=Article&match_order=1&q=THP&zhida_source=entity)）在换出到外部的`flash/disk`和从`flash/disk`换入物理内存的过程会比`normal size`的`page`带来更大的开销（可参考[这篇文章](https://zhuanlan.zhihu.com/p/117239320)）。

<hr/>

​																																	转载：[虚拟地址转换[四] - large page/hugepage](https://zhuanlan.zhihu.com/p/66427560)