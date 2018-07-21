---
title: Netty PoolArea原理探究
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
PoolArena作为Netty底层核心内存管理类, 主要原理是首先申请一些内存块, 不同的成员变量分配不同大小的内存块。下图描述了Netty主要的成员变量:
<img src="http://owsl7963b.bkt.clouddn.com/PoolArea.png" height="400" width="450"/>
netty将内存块划分为3个类型:
```
    enum SizeClass {
        Tiny,
        Small,
        Normal
    }
```
Tiny主要解决16b-498b之间的内存块分配, small解决分配512b-4kb的内存分配, normal解决8k-16m的内存分配。 该图清晰地描述了PoolArena里这三个类型对应的变量: tinySubpagePools、smallSubpagePools、q050、q025、q000、qInit、q075、q100。
大致了解了这些, 为了更详细的了解分配细节, 首先对PoolArena成员变量进行简单分析
```
    //tiny级别的个数, 每次递增2^4b, tiny总共管理32个等级的小内存片:[16, 32, 48, ..., 496], 注意实际只有31个级别内存块
    static final int numTinySubpagePools = 512 >>> 4;
    //全局默认唯一的分配者, 见PooledByteBufAllocator.DEFAULT
    final PooledByteBufAllocator parent;
    // log(16M/8K) = 11,指的是normal类型的内存等级, 分别为[8k, 16k, 32k, ..., 16M]
    private final int maxOrder;
    //默认8k
    final int pageSize;
    //log(8k) =  13
    final int pageShifts;
    //默认16M
    final int chunkSize;
    //-8192
    final int subpageOverflowMask;
    //指的是small类型的内存等级: pageShifts - log(512) = 4,分别为[512, 1k, 2k, 4k]
    final int numSmallSubpagePools;
     //small类型分31个等级[16, 32, ..., 512], 每个等级都可以存放一个链(元素为PoolSubpage), 可存放未分配的该范围的内存块
    private final PoolSubpage<T>[] tinySubpagePools;
     //small类型分31个等级[512, 1k, 2k, 4k], 每个等级都可以存放一个链(元素为PoolSubpage), 可存放未分配的该范围的内存块
    private final PoolSubpage<T>[] smallSubpagePools;//存储1024-8096大小的内存
     //存储chunk(16M)使用率的内存块, 不同使用率的chunk, 存放在不同的对象中
    private final PoolChunkList<T> q050;
    private final PoolChunkList<T> q025;   //存储内存利用率25-75%的chunk
    private final PoolChunkList<T> q000;   //存储内存利用率1-50%的chunk
    private final PoolChunkList<T> qInit;  //存储内存利用率0-25%的chunk
    private final PoolChunkList<T> q075;    //存储内存利用率75-100%的chunk
    private final PoolChunkList<T> q100;   //存储内存利用率100%的chunk

    private final List<PoolChunkListMetric> chunkListMetrics;

    // Metrics for allocations and deallocations
    private long allocationsNormal;
    // We need to use the LongCounter here as this is not guarded via synchronized block.
    private final LongCounter allocationsTiny = PlatformDependent.newLongCounter();
    private final LongCounter allocationsSmall = PlatformDependent.newLongCounter();
    private final LongCounter allocationsHuge = PlatformDependent.newLongCounter();
    private final LongCounter activeBytesHuge = PlatformDependent.newLongCounter();

    // Number of thread caches backed by this arena. 该PoolArea被多少线程引用。
    final AtomicInteger numThreadCaches = new AtomicInteger();

```
PoolArea将申请的未使用的、不同大小的内存块使用不同的对象来分配完成:
+ tinySubpagePools分配[16b, 496b]之间的内存大小, 每次分配以16b为一个单位增长。
+ smallSubpagePools 分配[512b, 8k]之间的内存大小, 每次翻倍增长。
+ q050、q025、q000、qInit、q075都是分配[8k, 16M]大小的对象, 存放的元素都是大小为16M的PoolChunk, 不同的是元素PoolChunk的使用率不同, 比如q025里面存放的chunk使用率为[25%, 75%]。



