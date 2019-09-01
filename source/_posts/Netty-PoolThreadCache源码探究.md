---
title: Netty PoolThreadCache原理探究
date: 2018-07-14 19:04:06
tags:
toc: true
---
NioEventLoop在为数据分配存放的内存时, 会首先尝试从线程本地缓存中去申请, 只有当本地缓存中申请失败, 才会考虑从全局内存中申请, 本地缓存的管理者就是PoolThreadCache对象。 Netty自己实现了类似LocalThread的类来充当线程缓存: PoolThreadLocalCache, 本节将充分围绕这两个类的源代码进行描述。
# PoolThreadCache
Netty自己进行内存管理, 将内存主要分为Tiny, small, normal等size不等的区间。 在PoolThreadCache中将缓存也按照size进行划分, 下图是PoolThreadCache的内部整体结构图:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/PoolThreadCache2.png" height="400" width="450"/>
图中只展示了small类型数组的大小(为4), 而tiny、normal数组的大小分别分32、 3。 每个数据元素代表着缓存不同类型大小的对象。 比如回收size为32B的对象, 将相应的内存块放在tiny类型数组、下标为1 (log(32>>4))的queue中。 本文章大量用到某一级别的缓存块, 举个例子:normal级别的缓存块有[8k, 16k, 32k]。
下面大致介绍下PoolThreadCache里面的属性作用:
```
    final PoolArena<byte[]> heapArena;
    final PoolArena<ByteBuffer> directArena;

    // Hold the caches for the different size classes, which are tiny, small and normal.
    //tiny内存缓存的个数。默认为512
    private final MemoryRegionCache<byte[]>[] tinySubPageHeapCaches;
    //small内存缓存的个数,默认为256个
    private final MemoryRegionCache<byte[]>[] smallSubPageHeapCaches;
    private final MemoryRegionCache<ByteBuffer>[] tinySubPageDirectCaches;
    private final MemoryRegionCache<ByteBuffer>[] smallSubPageDirectCaches;
    //normalCacheSize缓存的个数，默认为64
    private final MemoryRegionCache<byte[]>[] normalHeapCaches;
    private final MemoryRegionCache<ByteBuffer>[] normalDirectCaches;

    private final int freeSweepAllocationThreshold;

    private final Thread deathWatchThread;
    //线程消亡后，释放资源
    private final Runnable freeTask;
    //freeSweepAllocationThreshold  在本地线程每分配freeSweepAllocationThreshold 次内存后，检测一下是否需要释放内存。
    private int allocations;

```
`heapArena`与`directArena`作用一样, 根据用户使用direct内存还是heap内存来确定使用哪个块。由构造函数可以看出directArena与PoolThreadCache绑定了, 同时PoolThreadCache也与某个NioEventLoop对应的线程绑定的, 所以该NioEventLoop线程都与唯一的directArena(&heapArena)绑定着, 这样相对减轻了线程间申请内存导致互斥的发生。`smallSubPageHeapCaches`数组长度为4(如上图所示), 依次缓存[512K, 1024k, 2048k, 4096k]大小的缓存, 每个的元素对应的缓存queue个数不能超过256个; 而tinySubPageHeapCaches数组缓存的是[16B, 32B, ... , 496B]大小的内存块, 每个元素对应的缓存queue个数不能超过512个。`normalHeapCaches`数组结构相同, 但是只缓存[8k, 16k, 32k]大小的内存块, 每个元素对应的缓存queue个数不超过64个。 normal最大内存块为16m, 而缓存仅仅缓存最大32k内存的原因是这是一种巨大的开销: 试想仅仅16m对应的级别存储, 就可缓存16M*64大小的内存块放在内存, 而这些内存块等着被新分配出去而没有主动释放, 存在巨大的浪费。 至于tiny、small、normal缓存每一等级划分规则, 可参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/05/23/Netty%E5%86%85%E5%AD%98%E5%AD%A6%E4%B9%A0/"> Netty PoolArea内存原理探究</a>
 由于normalHeapCaches的特殊性, 如下展示该部分的代码实现:
 ```
 private static <T> MemoryRegionCache<T>[] createNormalCaches(
            int cacheSize, int maxCachedBufferCapacity, PoolArena<T> area) {
        if (cacheSize > 0) {
            int max = Math.min(area.chunkSize, maxCachedBufferCapacity); //默认32k
            //normalHeapCaches 数组中的元素的大小，是以2的幂倍pageSize递增的
            int arraySize = Math.max(1, log2(max / area.pageSize) + 1);
            //只缓存8k，16k，32k的缓存，太大的话，内存扛不住，若最大缓存32m的话，缓存64*32M个，太大了，扛不住
            @SuppressWarnings("unchecked")
            MemoryRegionCache<T>[] cache = new MemoryRegionCache[arraySize];
            for (int i = 0; i < cache.length; i++) {
                cache[i] = new NormalMemoryRegionCache<T>(cacheSize);
            }
            return cache;
        } else {
            return null;
        }
    }
 ```
 最大缓存的大小由io.netty.allocator.maxCachedBufferCapacity来指定, 默认缓存最大32k。

以下展示缓存中分配内存的过程, 以从normal级别缓存分配内存为例:
```
     boolean allocateNormal(PoolArena<?> area, PooledByteBuf<?> buf, int reqCapacity, int normCapacity) {
        return allocate(cacheForNormal(area, normCapacity), buf, reqCapacity);
    }
```
在cacheForNormal中根据normCapacity确定从normalSubPageDirectCaches对应级别获取缓存内存块, 接着开始分配内存:
```
    @SuppressWarnings({ "unchecked", "rawtypes" })
    private boolean allocate(MemoryRegionCache<?> cache, PooledByteBuf buf, int reqCapacity) {
        if (cache == null) {
            // no cache found so just return false here
            return false;
        }
        boolean allocated = cache.allocate(buf, reqCapacity);
        if (++ allocations >= freeSweepAllocationThreshold) {
            allocations = 0;
            trim();
        }
        return allocated;
    }
```
做了如下事情:
+ 若没有该级别的缓存块, 则直接退出。
+ 若缓存有该级别的缓存块, 则将该缓存块分配出去, 同时判断从PoolThreadCache缓存中成功分配内存的次数是否达到阈值freeSweepAllocationThreshold(8192次), 若达到阈值, 则在trim()中尝试释放分配率很低的缓存块, 以免内存泄漏。

trim()是如何确定哪些缓存块需要释放呢? 它会分别检查tiny、small、normal类型缓存块, 并轮训其中每一级别缓存块, 调用MemoryRegionCache.trim()检查是否需要释放, 以下以检查normal类型16k级别缓存块为例来说明:
```
        public final void trim() {
            int free = size - allocations;
            allocations = 0;

            // We not even allocated all the number that are
             //只有从该级别分配大于预定值，tiny：512，small:256 , normal:64次
            if (free > 0) {
                free(free);//才不会释放该缓存
            }
        }
```
`size`表示该16K级别的queue单位十年内必须分配多少次, 才不会释放(默认64个)
`allocations`表示在达到MemoryRegionCache成功分配freeSweepAllocationThreshold次缓存中、从16K级别的缓存块中分配的缓存次数。
free大于0表示成功分配freeSweepAllocationThreshold次缓存中时间内,从当前缓存中分配的次数allocations小于阈值size, 该缓存队列需要释放。 如果从16KB级缓存队列中成功分配的缓存次数超过size(64次), 则不会释放级别缓存queue。若没有从该级别缓存队列中成功分配一次, 那么该级别的缓存queue存放的缓存块将全部释放。

## PoolThreadLocalCache
文章开头讲了, 线程首先从本地缓存分配内存。PoolThreadCache主要解决了了如何从本地缓存分配内存, 而本地缓存如何与该线程联系在一起的呢? 这就是PoolThreadLocalCache起的作用。
PoolThreadLocalCache是全局唯一的, 任何线程分配内存, 都会调用同一个PoolThreadLocalCache.get()获取PoolThreadCache。 PoolThreadLocalCache继承了FastThreadLocal, PoolThreadLocalCache.get()实际调用了FastThreadLocal.get()方法:
```
    public final V get() {
        return get(InternalThreadLocalMap.get());
    }

    public final V get(InternalThreadLocalMap threadLocalMap) {
         //想得到该层级缓存，发现没有，那么只能去初始话一个
        Object v = threadLocalMap.indexedVariable(index);
        if (v != InternalThreadLocalMap.UNSET) {
            return (V) v;
        }

        return initialize(threadLocalMap);
    }
```
在InternalThreadLocalMap中定义了slowThreadLocalMap属性, 该类型是我们熟悉的ThreadLocal。
```
static final ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap = new ThreadLocal<InternalThreadLocalMap>();
```
若从本地缓冲中获取不到PoolThreadCache, 则会调用PoolThreadLocalCache.initialize()初始一个:
```
        protected synchronized PoolThreadCache initialValue() {
            final PoolArena<byte[]> heapArena = leastUsedArena(heapArenas); //找出被别的NioEventLoop使用最少次数多PoolArea
            final PoolArena<ByteBuffer> directArena = leastUsedArena(directArenas);

            if (useCacheForAllThreads || Thread.currentThread() instanceof FastThreadLocalThread) {
                return new PoolThreadCache( //为每一级别增加缓存
                        heapArena, directArena, tinyCacheSize, smallCacheSize, normalCacheSize,
                        DEFAULT_MAX_CACHED_BUFFER_CAPACITY, DEFAULT_CACHE_TRIM_INTERVAL);
            }
            // No caching for non FastThreadLocalThreads.
            return new PoolThreadCache(heapArena, directArena, 0, 0, 0, 0, 0);
        }
```
这里我们需要注意leastUsedArena()函数。 Netty默认会产生NioEventLoop |work|个PoolArea块, 至于该线程绑定哪个PoolArea呢, 是根据该PoolArea被多少线程绑定次数来依据的。 被越少的线程绑定到一起, 分配内存发生冲突的概率越小。 这里选择被绑定次数最低的那个PoolArea来构建PoolThreadCache。
至此, 线程与PoolThreadCache实现了一一绑定。 之后该线程分配内存, 都会利用PoolThreadLocalCache.get()获取PoolThreadCache, 然后利用里面的PoolArea来完成的。

