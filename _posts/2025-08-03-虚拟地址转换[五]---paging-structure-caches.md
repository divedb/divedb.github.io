---
date: 2025-08-03
layout: post
title: 虚拟地址转换[五] - paging structure caches
categories: Linux
tags: [Linux, 内存, 虚拟内存] 
---

回顾上文`TLB entry`和`PTE entry`的组成，你有没有意识到，在多级页表系统中，`TLB`其实只是最后一级`PTE`的缓存（对于`large pages`的`TLB`则最后一级是`PDE`或者`PDPTE`，本文以下的讨论都是针对非`large page`的情况），这和在单级页表中是一样的。

多级页表的查找是一个串行的，链式的过程。试想一下，访问在虚拟地址空间里连续的两个`pages`（比如虚拟地址分别为`0x0000123456789000`和`0x000012345678A000`），而这两个`pages`的`PGD entry, PUD entry, PMD entry`都是一样的，只有`PTE`不一样。

难道在通过`N`次内存访问查找到第一个`page`的物理页面号后，在访问相邻的（虚拟地址层面）第二个`page`时还需要再老老实实，一步一步的从`PGD`往下找？这对于在追求性能方面可谓无所不用其极的现代处理器来说，是不可接受的。

既然`PTE`都可以被缓存，那前面几级的`PMD, PUD, PGD`也应该可以被缓存吗？是的，以intel的`x86-64`架构为例，它支持`PDE cache`（使用64位虚拟地址的`bits 47:21`作为`index/tag`，对应`linux`里的`PMD entry`），`PDPTE cache`（使用64位虚拟地址的`bits 47:30`作为`index/tag`，对应`linux`里的`PUD entry`），`PML4 cache`（使用64位虚拟地址的`bits 47:39`作为`index/tag`，对应`linux`里的`PGD entry`）。

除了最后一级页表`PTE`的`entry`是直接指向`page`外，其他级的页表的`entry`都是指向下一级页表首地址的，因此这些级的页表被称为**paging structure**，所以`PDE cache`，`PDPTE cache`和`PML4 cache`被统称为`paging structure caches`。在`ARM`中，这些`caches`被称为`table walk caches`（名字应该是来自`MMU`里的`table walk unit`）。

如果发生`TLB miss`，相当于`PTE cache`中没找到，那就

1.  从`PDE cache`中找，如果找到了则获得对应`PT`页表的首地址，可继续在`PT`中索引到`PTE`，需要1次内存访问；
2.  没找到再从`PDPTE cache`中找，如果找到了则获得对应`PD`页表的首地址，然后在`PD->PT`中索引，需要2次内存访问；
3.  还没找到再从`PML4 cache`中找，如果找到了则获得对应`PDPT`页表的首地址，然后在`PDPT->PD->PT`中索引，需要3次内存访问；
4.  `PML4 cache`中也没有的话，那只能去`DRAM`里按照`PML4->PDPT->PD->PT`找了，需要4次内存访问。

可见，从`TLB`到`DRAM`，越往后，所需要的内存访问次数越多。

![img](https://pic3.zhimg.com/v2-90f7241556a312ba982a3f0299f9d8f2_1440w.jpg)

同`TLB`一样，`paging structure caches`需要使用专用的`cache`来实现，对这些`cache`的支持与否取决于不同处理器的具体设计。以`intel`的`Haswell`为例（`i7-4770`）为例，其`PDE cache`含有32个`entries`。

[这篇文章](https://zhuanlan.zhihu.com/p/66971714)曾讲到`TLB`中可能含有`PCID/ASID`来分别不同的进程，同样的，`paging structure caches`也是使用虚拟地址的位域子集作为`index/tag`的，如果在进程切换的时候不希望整个被`flush`掉，也需要含有`PCID/ASID`。

其实，页表既然是放在普通内存中的，自然也可以被缓存到普通`cache`中，`MMU`在不得不去查找页表之前，也会先去普通`cache`里看看，实在没有再极不情愿的去访问内存。以`x86`处理器为例，`CR3`寄存器和每级页表的`entry`都含有一个叫`PCD`（`page-level cache disable`）的位，可以控制该`entry`对应的下级页表（对于`PTE`对应的就是`page`）是否需要被缓存到普通`cache`中。

![img](https://picx.zhimg.com/v2-5b1d04c6430408709ab67b22b5c75c53_1440w.jpg)

我的理解是，这种`cache`（`intel`的手册中称为`page-level cache`）缓存的是一个页表，而`paging structure caches`缓存的是页表中的某一项，后者效率更高，所以尝试访问的顺序应该是`TLB -> paging structure caches -> 普通cache中的页表 -> 内存中的页表`。

​																																转载：[虚拟地址转换[五] - paging structure caches](https://zhuanlan.zhihu.com/p/65774094)

<hr/>

✅ 在`x86-64`（`IA-32e`） 架构中的四级页表名称与缩写

`Intel`的64位架构使用四级页表，分别如下：

| 层级 | 缩写    | 全称                                 | 每项控制多少内存          |
| ---- | ------- | ------------------------------------ | ------------------------- |
| 1    | `PML4E` | `Page Map Level 4 Entry`             | 控制`512 GiB`             |
| 2    | `PDPTE` | `Page Directory Pointer Table Entry` | 控制`1 GiB`               |
| 3    | `PDE`   | `Page Directory Entry`               | 控制`2 MiB`               |
| 4    | `PTE`   | `Page Table Entry`                   | 控制`4 KiB`（标准页大小） |

它们组成了64位虚拟地址的逐级映射：

```bash
[VA 63–48] ignored
[VA 47–39] → PML4E
[VA 38–30] → PDPTE
[VA 29–21] → PDE
[VA 20–12] → PTE
[VA 11–0 ] → Offset within 4KiB page
```

✅ 在`Linux`内核术语中对应的名称（更常见于内核源码）

`Linux`使用了更抽象、通用的术语来表示页表层级，特别是为了跨平台兼容。

| 层级（从上到下） | `Linux`名称 | 对应于`x86-64`硬件名称        | 含义/缩写解释           |
| ---------------- | ----------- | ----------------------------- | ----------------------- |
| 1                | `PGD`       | `PML4`                        | `Page Global Directory` |
| 2                | `P4D`       | 一般恒等于`PGD`（内核中保留） | `Page 4th Directory`    |
| 3                | `PUD`       | `PDPTE`                       | `Page Upper Directory`  |
| 4                | `PMD`       | `PDE`                         | `Page Middle Directory` |
| 5                | `PTE`       | `PTE`                         | `Page Table Entry`      |

>   ❗注意：`PGD → P4D → PUD → PMD → PTE`是`Linux`的通用叫法，
>    `x86-64`实际只用到四级，但为了统一架构，`Linux`中保留了五级接口（有的层级恒等于上一层）。

⛓️ 举个流程图例：

```bash
1. CPU 访问虚拟地址 VA
2. 查 TLB：
   - 如果命中 → 得到物理地址 → 成功！
   - 如果未命中 → 进入 Page Walk：

       ┌─────────────┐
       ↓             │
     查 PML4[va[47:39]]  ←—— Paging Structure Cache 尝试加速
       ↓
     查 PDP[va[38:30]]   ←—— Paging Structure Cache
       ↓
     查 PD[va[29:21]]    ←—— Paging Structure Cache
       ↓
     查 PT[va[20:12]]
       ↓
     得到 PTE → 填入 TLB → 完成地址转换
```

