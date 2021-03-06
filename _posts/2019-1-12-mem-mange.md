---
layout:     post
title:      "InnoDB-内存管理系统"
author:     "liulei"
header-img: "img/post-bg-miui6.jpg"
tags:
    - InnoDB
---
# 内存管理系统

innoDB 存储引擎采用内存堆（memory heap)的方式来进行内存对象的管理。其优点是可以一次性分配大块的内存，而不是通常的按需分配方法。这样的分配方式可以将多次的内存分配合并为单次进行，之后的内存请求就可以在InnoDB 引擎内部进行，从而减少了频繁使用malloc 和 free带来的时间与潜在性能的开销。将使用建立内存堆的方式分配的方法称为缓冲池分配，将使用Malloc 分配内存的方法称为动态分配。



![](.\图片\360截图170602255868102.jpg)

通过调用函数`mem_area_alloc`从通用内存池建立一个内存堆或者为内存堆增加一个内存块。内存堆也可以通过`buf_frame_alloc`从缓冲池分配**一整页**大小的内存空间。

![](.\图片\360截图187201175410955.jpg)

上面所示的是内存堆的栈结构图：`mem_block_t`来表示内存堆中每次从操作系统或缓冲池中分配的内存块。从本质上看，内存堆就是一系列相连的内存块。每个内存块的头部都包含`mem_block_info_t`用来存储内存块元数据信息。其中的`base `字段用来连接内存堆中各**内存块**的链表基节点，仅在堆中第一个内存块中定义。