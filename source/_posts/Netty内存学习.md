---
title: Netty PoolArea内存原理探究
date: 2018-05-23 22:31:13
tags:
---
# 简介
Netty内存主要分为两种: DirectByteBuf和HeapByteBuf, 实际上就是堆外内存和堆内内存。 堆外内存通过堆内对象进行操控, 堆外内存又称直接内存。 自从JDK1.4开始, 增加了NIO, 可以直接Native函数在堆外构建直接内存。Netty作为服务器架构技术, 拥有大量的网络数据传输, 当我们进行网络传输时, 必须将数据拷贝到直接内存, 合理利用好直接内存, 能够大量减少堆内数据和直接内存考虑, 显著地提高性能。 但是堆外内存也有一定的缺点, 它进程主动垃圾回收,垃圾回收效率也极低, 因此, netty主动创建了Pool和Unpool的概念。
## Pool和Unpool区别
字面意思, 分别是池化内存和非池化内存。`池化内存`的管理方式是首先申请一大块内存, 然后再慢慢使用, 当使用完成释放后, 再将该部分内存放入池子中, 等待下一次的使用。`非池化内存`就是普通的内存使用, 需要时直接申请, 释放时直接释放。 可以通过参数`Dio.netty.allocator.type`确定netty默认使用内存的方式, 目前neetty针对pool做了大量的支持, 这样内存使用直接交给了netty管理, 减轻了开发的难度。 所以在netty4时候, 默认使用pool方式。
这样的话, 内存分为四种: PoolDireBuf、UnpoolDireBuf、PoolHeapBuf、UnpoolHeapBuf。netty底层默认使用的PoolDireBuf类型的内存, 这些内存主要由PoolArea管理, 这也是本文的重点。
# 内存分配
线程调用如下接口来获取内存:
```
    protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
        PoolThreadCache cache = threadCache.get();   //获取PoolThreadCache，若针对该缓存有的话，则获取
        PoolArena<ByteBuffer> directArena = cache.directArena;

        final ByteBuf buf;
        if (directArena != null) { //针对缓存已经有了，16个，选取了其中的一个
            buf = directArena.allocate(cache, initialCapacity, maxCapacity);
        } else {
            buf = PlatformDependent.hasUnsafe() ?
                    UnsafeByteBufUtil.newUnsafeDirectByteBuf(this, initialCapacity, maxCapacity) :
                    new UnpooledDirectByteBuf(this, initialCapacity, maxCapacity);
        }

        return toLeakAwareBuffer(buf);
    }
```
主要做的事:
+ 获取该线程绑定的PoolThreadCache(可参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/14/Netty-PoolThreadCache%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6/">Netty PoolArea内存原理探究</a>)
+ 从绑定的PoolThreadCache中获取PoolArena, 从PoolArena中开始真正分配内存。

# PoolArena
PoolArena作为Netty底层核心内存管理类, 主要原理是首先申请一些内存块, 不同的属性管理不同大小的内存块。下图描述了Netty主要的属性对象
<img src="http://owsl7963b.bkt.clouddn.com/PoolArea.png" height="400" width="450"/>





