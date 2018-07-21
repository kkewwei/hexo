---
title: Netty PoolChunk原理探究
date: 2018-07-20 18:48:20
tags:
---
Netty分配内存与回收主要是在PoolChunk上完成的, 在内存分配时希望是有序的。对于申请到的16M PoolChunk内存, 用户实际使用时可能一次性不能使用16M, 所以逻辑上划分成不同的块, 使用平衡二叉树进行管理, 二叉树结构如下:
<img src="http://owsl7963b.bkt.clouddn.com/PoolChunk1.png" height="400" width="450"/>
该二叉树将PoolChunk分11层, 第一层为1个16M, 第二层为2个8MB,第三层为4个4MB的内存块, 直到第11层为2048个8KB的内存块,  8kb的内存块称之为page。
+ 如果我们申请16M的内存, 那么将直接在该二叉树第一层申请。
+ 若申请32K的内存, 那么在该二叉树第8层申请。
+ 若申请8K的内存, 那么将直接在第11层申请。
*实际内存申请内存大小时, size一定限定为2的幂次方, 参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/05/23/Netty%E5%86%85%E5%AD%98%E5%AD%A6%E4%B9%A0/">Netty PoolArea原理探究</a>*
接下来仔细介绍PoolChunk的成员属性:
```
    //该PoolChunk所属的PoolArena, 上层PoolArena控制着在哪块PoolArena上分配
    final PoolArena<T> arena;。
    //对外内存: DirectByteBuffer, 堆内内存 byte[]。
    final T memory;
    // 是内存池还是非内存池方式
    final boolean unpooled;
    private final byte[] memoryMap;  //PoolChunk的物理视图是连续的PoolSubpage,用PoolSubpage保持，而memoryMap是所有PoolSubpage的逻辑映射，映射为一颗平衡二叉数，用来标记每一个PoolSubpage是否被分配。下文会更进一步说明
    private final byte[] depthMap;    //而depthMap中保存的值表示各个id对应的深度，是个固定值，初始化后不再变更。
    //与叶子节点个数相同, 一个叶子节点可以映射PoolSupage中一个元素, 若叶子节点与该元素完成了映射, 说明该叶子节点已经被分配出去了
    private final PoolSubpage<T>[] subpages;
    /** Used to determine if the requested capacity is equal to or greater than pageSize. */
    //用来判断申请的内存是否超过pageSize大小
    private final int subpageOverflowMask;
    //每个PoolSubpage的大小，默认为8192个字节（8K)
    private final int pageSize;
    //pageSize 2的 pageShifts幂
    private final int pageShifts;
    // 平衡二叉树的深度，
    private final int maxOrder;
    //PoolChunk的总内存大小,chunkSize =   (1<<maxOrder) * pageSize。
    private final int chunkSize;
    // PoolChunk由maxSubpageAllocs个PoolSubpage组成, 默认2048个。
    private final int maxSubpageAllocs;
    /** Used to mark memory as unusable */
    //标记为已被分配的值，该值为 maxOrder + 1=12, 当memoryMap[id] = unusable时，则表示id节点已被分配
    private final byte unusable;

    //当前PoolChunk剩余可分配内存, 初始为16M。
    private int freeBytes;

    //一个PoolChunk分配后，会根据其使用率挂在一个PoolChunkList中(q050, q025...)
    PoolChunkList<T> parent;
```
当我们需要分配内存时, 会在二叉树上查找满足大小的节点, 我们需要考虑的一个问题: 若申请了第11层下标为2048节点的8k内存, 下标为1024的父节点怎么才能避免被不被16k的申请所申请到呢?
<img src="http://owsl7963b.bkt.clouddn.com/PoolChunke%20allocation.png" height="400" width="450"/>
Netty为每一层分配的一个层号, 根据层号可以直接获取该节点剩余最大可分配的内存空间。
当第11层下标为2048的节点被分配出去:
1. 则该节点的层号被修改为12, 表示该节点不可再分配。
2. 同时id为1024的父节点层号改为11, id为512的节点层号改为10, id为1的节点层号改为2.

当申请16k的内存时, 分配的节点的层号必须等于(11-log(16k/8k)) = 10, id为1024的节点自然被淘汰了。动态记录每个节点层数的成员属性为memoryMap, 记录每个节点深度的成员属性为depthMap, 其中memoryMap值是可以动态改变的, 而depthMap是静态不变的。
构造函数中该成员斌量初始过程如下:
```
        memoryMap = new byte[maxSubpageAllocs << 1];
        depthMap = new byte[memoryMap.length];
        int memoryMapIndex = 1;
        for (int d = 0; d <= maxOrder; ++ d) { // move down the tree one level at a time
            int depth = 1 << d;
            for (int p = 0; p < depth; ++ p) {
                // in each level traverse left to right and set value to the depth of subtree
                memoryMap[memoryMapIndex] = (byte) d;
                depthMap[memoryMapIndex] = (byte) d;
                memoryMapIndex ++;
            }
        }
```
实际操作中, 针对数据构造二叉树, 实际从memoryMap下标为1的节点开始使用, 数组元素个数为4096。

# PoolChunk分配内存
下面看如何是从PoolChunk中分配内存的:
```
    long allocate(int normCapacity) {
        if ((normCapacity & subpageOverflowMask) != 0) {
            return allocateRun(normCapacity);  //// 大于等于pageSize时返回的是可分配normCapacity的节点的id
        } else {  //分配小于8k
            return allocateSubpage(normCapacity); //（small）返回是第几个long,long第几位
        }
    }
```
针对申请的不同内存大小, 从不同对象中分配。若申请的内存大于page(8k), 进入allocateRun进行申请, 若申请的内存大小小于8K, 进入allocateSubpage进行申请。
我们首先看allocateRun是如何操作的
```
     private long allocateRun(int normCapacity) {//64k
        int d = maxOrder - (log2(normCapacity) - pageShifts);  //算出当前阶层
        int id = allocateNode(d);
        if (id < 0) {
            return id;
        }
        freeBytes -= runLength(id);
        return id;
    }
```