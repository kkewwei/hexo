---
title: Netty PoolArea内存学习
date: 2018-05-23 22:31:13
tags:
---
# 简介
Netty内存主要分为两种: DirectByteBuf和HeapByteBuf, 实际上就是堆外内存和堆内内存。 堆外内存通过堆内对象进行操控, 堆外内存又称直接内存。 自从JDK1.4开始, 增加了NIO, 可以直接Native函数在堆外构建直接内存。Netty作为服务器架构技术, 拥有大量的网络数据传输, 当我们进行网络传输时, 必须将数据拷贝到直接内存, 合理利用好直接内存, 能够大量减少堆内数据和直接内存考虑, 显著地提高性能。 但是堆外内存也有一定的缺点, 它不像堆内内存一样由进程主动垃圾回收, 因此, netty主动创建了Pool和unpool的概念。
### Pool和Unpool
字面意思, 分别是池化内存和非池化内存。`池化内存`的管理方式是首先申请一大块内存, 然后再慢慢使用, 当使用完成释放后, 再将该部分内存放入池子中, 等待下一次的使用。`非池化内存`就是普通的内存使用, 需要时直接申请, 释放时直接释放。 可以通过参数`Dio.netty.allocator.type`确定netty默认使用内存的方式, 目前neetty针对pool做了大量的支持, 这样内存使用直接交给了netty管理, 减轻了开发的难度。 所以在netty4时候, 默认使用pool方式。
这样的话, 内存分为四种: PoolDireBuf、UnpoolDireBuf、PoolHeapBuf、UnpoolHeapBuf。netty底层默认使用的PoolDireBuf, 也就是本文章要将的重点。



