---
title: Netty PoolThreadCache源码探究
date: 2018-07-14 19:04:06
tags:
---
NioEventLoop在为数据分配存放的内存时, 会首先尝试从线程本地缓存中去申请, 只有当本地缓存中申请失败, 才会考虑从全局内存中申请, 本地缓存的管理者就是PoolThreadCache对象。 Netty自己实现了类似LocalThread的类来充当线程缓存: PoolThreadLocalCache, 本节将充分围绕这两个类的源代码进行描述。
# PoolThreadCache
Netty自己进行内存管理, 将内存主要分为Tiny, small, normal等size不等的块。 在PoolThreadCache中将缓存也按照size进行划分, 下图是PoolThreadCache的内部整体结构图:
<img src="http://owsl7963b.bkt.clouddn.com/PoolThreadCache2.png" height="400" width="450"/>\
图中只展示了small类型数组的大小, 为4, 而tiny、normal数组的大小分别分512、 12。 每个数据元素代表着缓存不同类型大小的对象。 比如回收size为32B的对象, 将相应的内存块放在tiny类型数组、下标为1 (log(32>>4))的queue中。
下面大致介绍下PoolThreadCache里面的属性作用
```
    final PoolArena<byte[]> heapArena;
    final PoolArena<ByteBuffer> directArena;

    // Hold the caches for the different size classes, which are tiny, small and normal.
    private final MemoryRegionCache<byte[]>[] tinySubPageHeapCaches;//tiny内存缓存的个数。默认为512
    private final MemoryRegionCache<byte[]>[] smallSubPageHeapCaches;//small内存缓存的个数,默认为256个
    private final MemoryRegionCache<ByteBuffer>[] tinySubPageDirectCaches;
    private final MemoryRegionCache<ByteBuffer>[] smallSubPageDirectCaches;
    private final MemoryRegionCache<byte[]>[] normalHeapCaches;  //normalCacheSize缓存的个数，默认为64
    private final MemoryRegionCache<ByteBuffer>[] normalDirectCaches;  //Netty把大于pageSize小于chunkSize的空间成为normal内存


    private final Thread deathWatchThread;
    //线程消亡后，释放资源
    private final Runnable freeTask;
    //freeSweepAllocationThreshold  在本地线程每分配freeSweepAllocationThreshold 次内存后，检测一下是否需要释放内存。
    private int allocations;

```
heapArena与directArena作用一样, 根据用户使用direct内存还是heap内存Laur来确定使用哪个块。这里将directArena与PoolThreadCache绑定
