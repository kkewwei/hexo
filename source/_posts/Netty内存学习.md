---
title: Netty PoolArea原理探究
date: 2018-05-23 22:31:13
tags: PoolArena
---
# 简介
Netty内存主要分为两种: DirectByteBuf和HeapByteBuf, 实际上就是堆外内存和堆内内存。堆外内存又称直接内存, 通过io.netty.noPreferDirect参数设置。 自从JDK1.4开始, 增加了NIO, 可以直接Native函数在堆外构建直接内存。Netty作为服务器架构技术, 拥有大量的网络数据传输, 当我们进行网络传输时, 必须将数据拷贝到直接内存, 合理利用好直接内存, 能够大量减少堆内数据和直接内存考虑, 显著地提高性能。 但是堆外内存也有一定的缺点, 它进程主动垃圾回收,垃圾回收效率也极低, 因此, netty主动创建了Pool和Unpool的概念。
## Pool和Unpool区别
字面意思, 分别是池化内存和非池化内存。`池化内存`的管理方式是首先申请一大块内存, 然后再慢慢使用, 当使用完成释放后, 再将该部分内存放入池子中, 等待下一次的使用, 这样的话, 可以减少垃圾回收次数, 提高处理性能。`非池化内存`就是普通的内存使用, 需要时直接申请, 释放时直接释放。 可以通过参数`Dio.netty.allocator.type`确定netty默认使用内存的方式, 目前netty针对pool做了大量的支持, 这样内存使用直接交给了netty管理, 减轻了直接内存回收的压力。 所以在netty4时候, 默认使用pool方式。
这样的话, 内存分为四种: PoolDireBuf、UnpoolDireBuf、PoolHeapBuf、UnpoolHeapBuf。netty底层默认使用的PoolDireBuf类型的内存, 这些内存主要由PoolArea管理, 这也是本文的重点。
# 内存分配
线程调用如下接口来获取内存:
```
    protected ByteBuf newDirectBuffer(int initialCapacity, int maxCapacity) {
        PoolThreadCache cache = threadCache.get();
        PoolArena<ByteBuffer> directArena = cache.directArena;

        final ByteBuf buf;
        if (directArena != null) {
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
+ 获取该线程绑定的PoolThreadCache(可参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/14/Netty-PoolThreadCache%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6/">Netty PoolThreadCache原理探究</a>)
+ 从绑定的PoolThreadCache中获取PoolArena, 从PoolArena中开始真正分配内存。

# PoolArena
PoolArena作为Netty底层内存池核心管理类, 主要原理是首先申请一些内存块, 不同的成员变量来完成不同大小的内存块分配。下图描述了Netty最重要的成员变量:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/PoolArea.png" height="400" width="450"/>
netty将池化内存块划分为3个类型:
```
    enum SizeClass {
        Tiny,
        Small,
        Normal
    }
```
Tiny主要解决16b-498b之间的内存块分配, small解决分配512b-4kb的内存分配, normal解决8k-16m的内存分配。
大致了解了这些, 为了更详细的了解分配细节, 首先对PoolArena成员变量进行简单分析。
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

    // Number of thread caches backed by this arena. 该PoolArea被多少线程引用。
    final AtomicInteger numThreadCaches = new AtomicInteger();

```
PoolArea申请内存时根据申请的大小使用不同对象进行分配:
+ tinySubpagePools分配[16b, 496b]之间的内存大小, 数组中每个元素以16b为一个单位增长, 比如申请分配16b的内存, 将在下标为0对应的链中分配; 申请32b的内存, 将在下标为1对应的链中分配。
+ smallSubpagePools分配[512b, 4k]之间的内存大小, 分配结构同tinySubpagePools一样。
+ q050、q025、q000、qInit、q075主要负责分配[8k, 16M]大小的内存, 其存放的元素都是大小为16M的PoolChunk, 这几个成员变量不同的是元素PoolChunk的使用率不同, 比如q025里面存放的chunk使用率为[25%, 75%]。 若需要申请[16b, 4k]的内存、而tinySubpagePools、smallSubpagePools没有合适的内存块时, 会从这些对象包含的PoolChunk中分配8k的叶子节点供重新划分结构进行分配。
他们存储的属性PoolChunk可以在不同的属性中移动, 其中:<p>
&nbsp;&nbsp;若q025中某个PoolChunk使用率大于25%之后, 该PoolChunk将别移动到q050中。
&nbsp;&nbsp;若q050中某个PoolChunk使用率小于50%之后, 该PoolChunk将别移动到q025中。
&nbsp;&nbsp;若qInit使用率为0, 也不会释放该节点。
&nbsp;&nbsp;若q000使用率为0, 会被释放掉。

numThreadCaches负责统计该PoolChunk被多少NioEventLoop线程绑定, 具体可见<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/14/Netty-PoolThreadCache%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6/">Netty PoolThreadCache原理探究</a>

## PoolArena的内存分配
线程分配内存主要从两个地方分配: PoolThreadCache和PoolArena
<img src="https://kkewwei.github.io/elasticsearch_learning/img/PoolArea1.png" height="300" width="350"/>
其中PoolThreadCache线程独享, PoolArena为几个线程共享。
netty真正申请内存时, 首先便是调用PoolArena.allocate()函数:
```
 private void allocate(PoolThreadCache cache, PooledByteBuf<T> buf, final int reqCapacity) {
        final int normCapacity = normalizeCapacity(reqCapacity);
         // capacity < pageSize   小于8k
        if (isTinyOrSmall(normCapacity)) {
            int tableIdx;
            PoolSubpage<T>[] table;
            boolean tiny = isTiny(normCapacity);
            if (tiny) { // < 512
                 //若从缓冲中取得该值
                if (cache.allocateTiny(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                tableIdx = tinyIdx(normCapacity);
                table = tinySubpagePools;
            } else {  //small
                if (cache.allocateSmall(this, buf, reqCapacity, normCapacity)) {
                    // was able to allocate out of the cache so move on
                    return;
                }
                tableIdx = smallIdx(normCapacity);
                table = smallSubpagePools;
            }

            final PoolSubpage<T> head = table[tableIdx];

            /**
             * Synchronize on the head. This is needed as {@link PoolChunk#allocateSubpage(int)} and
             * {@link PoolChunk#free(long)} may modify the doubly linked list as well.
             */
             //小于8k的
            synchronized (head) {
                 //如果分配完会从当前级别链上去掉
                final PoolSubpage<T> s = head.next;
                 ///该型号的tiny的内存已经分配的有一个了
                if (s != head) {
                    assert s.doNotDestroy && s.elemSize == normCapacity;
                    long handle = s.allocate();//高32放着一个PoolSubpage里面哪段的哪个，低32位放着哪个叶子节点
                    assert handle >= 0;
                    s.chunk.initBufWithSubpage(buf, handle, reqCapacity);
                    incTinySmallAllocation(tiny);
                    return;//如果从链中找到就返回，
                }
            }
            //没有找到的话，就从Poolpage中分一个
            synchronized (this) {
                //说明head并没有分配值，是第一次分配。
                allocateNormal(buf, reqCapacity, normCapacity);
            }

            incTinySmallAllocation(tiny);
            return;
        }
        if (normCapacity <= chunkSize) { //小于16M
            if (cache.allocateNormal(this, buf, reqCapacity, normCapacity)) {  //cache=PoolThreadCache,本地是否已经有了
                // was able to allocate out of the cache so move on
                return;
            }
            synchronized (this) {
                allocateNormal(buf, reqCapacity, normCapacity);
                ++allocationsNormal;
            }
        } else {
            // Huge allocations are never served via the cache so just call allocateHuge
            allocateHuge(buf, reqCapacity); //大于16M，则分配大内存
        }
    }

```

PoolArena.allocate()分配内存主要考虑先尝试从缓存中, 然后再尝试从PoolArena分配。tiny和small申请过程一样, 以下都以tiny申请为例。具体过程如下:
1). 对申请的内存进行规范化, 就是说只能申请某些固定大小的内存, 比如tiny范围的16b倍数的内存, small范围内512b, 1k, 2k, 4k范围内存, normal范围内8k, 16k,..., 16m范围内内存, 始终是2幂次方的数据。申请的内存不足16b的,按照16b去申请。
2). 判断是否是小于8K的内存申请, 若是申请Tiny/Small级别的内存:
+ 首先尝试从cache中申请, 具体申请过程参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/14/Netty-PoolThreadCache%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6/">Netty PoolThreadCache原理探究</a>
+ 若在cache中申请不到的话, 接着会尝试从tinySubpagePools中申请, 首先计算出该内存在tinySubpagePools中对应的下标, 下标计算公式如下:
```
    static int tinyIdx(int normCapacity) {  //申请内容小于512，下标
        return normCapacity >>> 4;  //在tiny维护的链中找到合适自己位置的下标, 除以16，就是下标了
    }
    static int smallIdx(int normCapacity) {
        int tableIdx = 0;
        int i = normCapacity >>> 10; //首先是512 = 2^10
        while (i != 0) {
            i >>>= 1;
            tableIdx ++;
        }
        return tableIdx;
```
可以看出, normCapacity/16就是tiny级别的下标, normCapacity/1024就是small级别的下标。 然后再获取tinySubpagePools对应级别的内存的头结点head。
+ 检查对应链串是否已经有PoolSubpage可用, 若有的话, 直接进入PoolSubpage.allocate进行内存分撇, 具体可见<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/22/Netty-PoolSubpage%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/">Netty-PoolSubpage原理探究</a>, 并且根据handle初始化这块内存块。
+ 若没有可分配的内存, 则会进入allocateNormal进行分配
3). 若分配normal类型的类型, 首先也会尝试从缓存中分配, 然后再考虑从allocateNormal进行内存分配。
4). 若分配大于16m的内存, 则直接通过allocateHuge()从内存池外分配内存。
### 分配[16b, 16m]内存
接着上述过程, 会进入allocateNormal进行内存分配
```
private void allocateNormal(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
        if (q050.allocate(buf, reqCapacity, normCapacity) || q025.allocate(buf, reqCapacity, normCapacity) ||
            q000.allocate(buf, reqCapacity, normCapacity) || qInit.allocate(buf, reqCapacity, normCapacity) ||
            q075.allocate(buf, reqCapacity, normCapacity)) {
            return;//第一次进行内存分配时，chunkList没有chunk可以分配内存
        }
        //跑到Direct里面newChunk了, 将 产生第一个chunk
        // Add a new chunk.   https://www.jianshu.com/p/c4bd37a3555b  就是传说中的平衡树
        PoolChunk<T> c = newChunk(pageSize, maxOrder, pageShifts, chunkSize);  //需通过方法newChunk新建一个chunk进行内存分配，并添加到qInit列表中
        long handle = c.allocate(normCapacity); //取到平衡树里面哪个下标,比如256
        assert handle > 0;
        c.initBuf(buf, handle, reqCapacity);
        qInit.add(c); //第一次分配的话，都会放入qInit
    }
```
1. 首先会依次检查q050、q025、q000、qInit、q075链中的PoolArea, 是否能否分配该大小的内存, 检查分配过程如下:
```
    boolean allocate(PooledByteBuf<T> buf, int reqCapacity, int normCapacity) {
        if (head == null || normCapacity > maxCapacity) { //head是可以直接寸数据的
            // Either this PoolChunkList is empty or the requested capacity is larger then the capacity which can
            // be handled by the PoolChunks that are contained in this PoolChunkList.
            return false;
        }
        for (PoolChunk<T> cur = head;;) {
            long handle = cur.allocate(normCapacity); //取得哪个坐标下的某个值
            if (handle < 0) { //在poolchunk中没有找到能装得下的，那么继续找下一个
                cur = cur.next;
                if (cur == null) {
                    return false;
                }
            } else {
                cur.initBuf(buf, handle, reqCapacity);
                if (cur.usage() >= maxUsage) {//chunked量用超了则移动向下一个链
                    remove(cur);
                    nextList.add(cur);
                }
                return true;
            }
        }
    }
```
会轮训该链所有PoolChunk, 直到找到一个符合要求的内存块, 当分配完成后, 检查该PoolChunk是否因为使用率超过阈值需要放到别的队列中。
2. 若没有找到, 会去内存中申请一个PoolChunk的内存块, 在该PoolChunk中分配normCapacity大小的内存, 参考见<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/20/Netty-PoolChunk%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/">Netty PoolChunk原理探究</a>
3. 对PoolChunk进行初始化, 并将该PoolChunk加入qInit的链中。
这里有一个细节需要了解下, q050、q025、q000、qInit、q075按照这个顺序排序, 也就是说当在这几个对象都有可分配的内存时, 优先从 q050中分配, 最后从q075中分配。这样安排的考虑是:
+ 将PoolChunk分配维持在较高的比例上。
+ 保存一些空闲度比较大的内存, 以便大内存的分配。

# 总结
非内存池化的内存分配没有什么好说的, 并没有组织成什么结构来分配, 内存的释放主要由PoolChunk和PoolSubpage来释放。 本文主要讲了从poolArena上层结构tinySubpagePools、mallSubpagePools、050、q025、q000、qInit、q075分配内存、 大致的步骤, 至于从每个对象具体如何分配内存, 请看相关文章<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/20/Netty-PoolChunk%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/">Netty PoolChunk原理探究</a>、<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/22/Netty-PoolSubpage%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/">Netty-PoolSubpage原理探究</a>.
