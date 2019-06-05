---
title: Netty PoolChunk原理探究
date: 2018-07-20 18:48:20
tags:
toc: true
---
Netty分配内存(0~16M)与回收主要是在PoolChunk上完成的, 在内存分配时希望是有序的。当没有内存可分配时, 一次申请到16M PoolChunk内存, 用户实际使用时可能一次性不能使用16M, 所以逻辑上将PoolChunk划分成不同的块, 使用平衡二叉树进行管理内存的分配, 构建的二叉树结构如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/PoolChunk1.png" height="400" width="450"/>
该二叉树将PoolChunk分11层, 第一层为1个16M, 第二层为2个8MB,第三层为4个4MB的内存块, 直到第11层为2048个8KB的内存块,  8kb的内存块称之为page。
+ 如果我们申请16M的内存, 那么将直接在该二叉树第一层申请。
+ 若申请32K的内存, 那么在该二叉树第8层申请。
+ 若申请8K的内存, 那么将直接在第11层申请。
*实际申请内存大小时, size一定限定为2^x, 参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/05/23/Netty%E5%86%85%E5%AD%98%E5%AD%A6%E4%B9%A0/">Netty PoolArea原理探究</a>
接下来介绍PoolChunk的成员属性:
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
<img src="https://kkewwei.github.io/elasticsearch_learning/img/PoolChunk2.png" height="200" width="250"/>
Netty为每一层分配的一个层号, 根据层号可以直接获取该节点剩余最大可分配的内存空间。
当第11层下标为2048的节点被分配出去:
1. 则该节点的层号被修改为12, 表示该节点不可再分配。
2. 同时id为1024的父节点层号改为11, id为512的节点层号改为10, ..., id为1的节点层号改为2.

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
在PoolArea.allocateNormal()中调用newChunk产生PoolChunk
```
        protected PoolChunk<ByteBuffer> newChunk(int pageSize, int maxOrder,
                int pageShifts, int chunkSize) {
            //
            if (directMemoryCacheAlignment == 0) {
                return new PoolChunk<ByteBuffer>(this,
                         //allocateDirect(chunkSize)会直接创建直接内存， 用直接内存地址构建的DirectByteBuffer
                        allocateDirect(chunkSize), pageSize, maxOrder,
                        pageShifts, chunkSize, 0);  //pageShifts= log8k = 13
            }
            final ByteBuffer memory = allocateDirect(chunkSize
                    + directMemoryCacheAlignment);
            return new PoolChunk<ByteBuffer>(this, memory, pageSize,
                    maxOrder, pageShifts, chunkSize,
                    offsetCacheLine(memory));
```
我们需要关注下如何通过allocateDirect()产生堆外内存的:
```
         private static ByteBuffer allocateDirect(int capacity) {
            //如果不是调用cleaner来回收对象，那么将使用DirectByteBuff来回收对象
            return PlatformDependent.useDirectBufferNoCleaner() ?
                    PlatformDependent.allocateDirectNoCleaner(capacity) : ByteBuffer.allocateDirect(capacity);
        }
        static ByteBuffer allocateDirectNoCleaner(int capacity) {
        return newDirectBuffer(UNSAFE.allocateMemory(capacity), capacity);
```
最终是通过直接内存地址的address来产生DirectByteBuffer的, 此时该对象没有cleaner成员变量, 将不能通过cleaner来回收直接内存。
接下来看看如何是从PoolChunk中分配内存的:
```
    long allocate(int normCapacity) {
        if ((normCapacity & subpageOverflowMask) != 0) {
            return allocateRun(normCapacity);  //// 大于等于pageSize时返回的是可分配normCapacity的节点的id
        } else {  //分配小于8k
            return allocateSubpage(normCapacity); //（small）返回是第几个long,long第几位
        }
    }
```
+ 返回值handle可以定位出该内存块的处在该PoolChunk中具体的位置信息。
+ 若申请的内存大于等于page(8k), 进入allocateRun进行申请, 返回值handler就是找到的节点在二叉树中的下标(int长度); 若申请的内存小于8K, 进入allocateSubpage进行申请, 返回长度是个long。

## 分配大于page的内存
我们首先看allocateRun是如何操作的
```
     private long allocateRun(int normCapacity) {//64k
         //算出当前大小的内存需要在那一层完成分配
        int d = maxOrder - (log2(normCapacity) - pageShifts);
        int id = allocateNode(d);
        if (id < 0) {
            return id;
        }
        freeBytes -= runLength(id);
        return id;
    }
```
1. 首先计算出在二叉树哪层分配内存, 比如申请32k的内存, 那么d = maxOrder - (log2(normCapacity) - pageShifts) = 11 - (log2(32k) - 13) = 9, 说明只能在该二叉树第9层找到合适的节点。 为啥减13, 因为默认pag为8k, log2(8k)=13。
2. 开始进入二叉树对应的d层中通过allocateNode查找哪个节点还没有分配出去:
```
    private int allocateNode(int d) {
        int id = 1;
        int initial = - (1 << d); // has last d bits = 0 and rest all = 1
        byte val = value(id);
        //若第一层的深度不够，那么该chunkend不够分配
        if (val > d) { // unusable
            return -1;
        }
        // id & initial == 1 << d for all ids at depth d, for < d it is 0
        while (val < d || (id & initial) == 0) {
            id <<= 1;
            val = value(id);
            if (val > d) {
                id ^= 1;
                val = value(id);
            }
        }
        byte value = value(id);
        assert value == d && (id & initial) == 1 << d :
        String.format("val = %d, id & initial = %d, d = %d", value, id & initial, d);
        setValue(id, unusable); // mark as unusable
        updateParentsAlloc(id);
        return id; //返回的是查找到的那个节点的下标
    }
```
主要做了如下工作:
1. 从根节点开始遍历, 首先检查第1层的层号, 若大于申请的层号, 那么该节点不够申请的大小, 直接退出。(`若不退出的话, 那就意味该二叉树一定可以找到大小为d层号的节点`, 并且在该节点的下标一定>=2^d)
2. 若当前节点层数<d, 或者当前节点的下标 < 2^d, 那么继续下一层左孩子节点查找, 直到找到某一个节点的层数==目前层数d, 则完成查找。若发现该节点剩余大小不够分配, 则在兄弟节点继续查找。
如下图, 当查找层号为11的节点, 找到符合下标id>=x 2^11的节点 && 层号 == 11的节点 , 只能在下标为2049的那个节点。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/PoolChunke_allocation_select.png" height="300" width="350"/>
其中 (id & initial) == 0) 等价于id <2^d, 作用: 若当前节点的下标< 2^s, 则会继续在当前节点的孩子节点查找。
3. 将成功找到的那个节点层号标为不可分配unusable, 意味着已经分配出去了。
4. 更新该节点的所有祖父父节点层号:
```
   private void updateParentsAlloc(int id) {
        while (id > 1) {  //开始更新父类节点的值
            int parentId = id >>> 1;
            byte val1 = value(id);
            byte val2 = value(id ^ 1);  //获取相邻节点的值
            byte val = val1 < val2 ? val1 : val2;
            setValue(parentId, val);
            id = parentId;
        }
    }
```
父节点的层号选取两个子节点层号最小的那个层号, 表示该父节点能分配的最大内存。

## 分配小于page的内存
我们来看是如何分配小于8k的内存。
```
    private long allocateSubpage(int normCapacity) {
        // Obtain the head of the PoolSubPage pool that is owned by the PoolArena and synchronize on it.
        // This is need as we may add it back and so alter the linked-list structure.
        PoolSubpage<T> head = arena.findSubpagePoolHead(normCapacity);  //// 找到arena中对应阶级的subpage头节点，不存数据的头结点
        synchronized (head) {
            int d = maxOrder; // subpages are only be allocated from pages i.e., leaves   subpage只能从叶子节点开始找起
            int id = allocateNode(d); //只在叶子节点找到一个为8k的节点，肯定可以找到一个节点
            if (id < 0) {
                return id;
            }

            final PoolSubpage<T>[] subpages = this.subpages;
            final int pageSize = this.pageSize;

            freeBytes -= pageSize;//（就是一个16M的空间）

            int subpageIdx = subpageIdx(id);  //第几个PoolSubpage（叶子节点）
            PoolSubpage<T> subpage = subpages[subpageIdx];
            if (subpage == null) {  //说明这个PoolSubpagte还没有分配出去
                subpage = new PoolSubpage<T>(head, this, id, runOffset(id), pageSize, normCapacity);
                subpages[subpageIdx] = subpage;
            } else {
                subpage.init(head, normCapacity);
            }
            return subpage.allocate();
        }
    }
```
主要做了如下事情:
+ 我们需要知道, 小于8k的内存分配都是在叶子节点里面分配的, 首先先从二叉树中查找层号为11(叶子节点)的可用节点。
+ 查看该节点是第几个叶子节点: subpageIdx。
+ 获取该叶子节点对应的PoolSubpage, subpage为null的可能为: PoolSubpage释放时, 并没有从subpages中取出, 该PoolSubpage还存放在subpages的数组里, 可参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/22/Netty-PoolSubpage%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/">Netty-PoolSubpage原理探究</a> free()函数, 至于从PoolSubpage中分配内存的过程放在<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/22/Netty-PoolSubpage%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/">Netty-PoolSubpage原理探究</a>中详细描述。

## 释放内存
上面讲的在allocate中申请内存时, 返回的是一个handle , 该释放内存时的参数也是该handle。
```
    void free(long handle) {
        int memoryMapIdx = memoryMapIdx(handle);
        int bitmapIdx = bitmapIdx(handle);

        if (bitmapIdx != 0) { // free a subpage
            PoolSubpage<T> subpage = subpages[subpageIdx(memoryMapIdx)];
            assert subpage != null && subpage.doNotDestroy;

            // Obtain the head of the PoolSubPage pool that is owned by the PoolArena and synchronize on it.
            // This is need as we may add it back and so alter the linked-list structure.
            PoolSubpage<T> head = arena.findSubpagePoolHead(subpage.elemSize);
            synchronized (head) {
                if (subpage.free(head, bitmapIdx & 0x3FFFFFFF)) {
                    return;
                }
            }
        }
        freeBytes += runLength(memoryMapIdx);
        setValue(memoryMapIdx, depth(memoryMapIdx));
        updateParentsFree(memoryMapIdx);
    }
```
做了如下事情:
+ 首先通过handle获取属于哪个page: memoryMapIdx、属于PoolSubpage里面哪个子内存块:bitmapIdx。
```
    private static int memoryMapIdx(long handle) { //低32位
        return (int) handle;
    }
    ////高32放着一个PoolSubpage里面哪段的哪个，低32位放着哪个叶子节点
    private static int bitmapIdx(long handle) { //高32位
        return (int) (handle >>> Integer.SIZE);
    }
```
handlee低32位字段即为memoryMapIdx, 高32为字段即为bitmapIdx。
+ 判是PoolSubpage的子块bitmapIdx不为0, 那么一定是小于8K的内存释放, 请参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/22/Netty-PoolSubpage%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/">Netty-PoolSubpage原理探究</a>toHandle函数
+ 若是PoolSubpage的子块bitmapIdx为0 , 那么一定是大于8K的内存释放, 释放时修改该节点祖辈的层号, 修改该chunk的大小。

# 总结

Netty PoolChunk讲完了, 主要理解二叉树的构建, 当分配大于8K的内存时, 怎么从二叉树中查找合适的节点, 怎么释放该二叉树上的节点块就可以了。 若分配小于8K的内存块, 主要是在子节点内部分配, 将放在<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/22/Netty-PoolSubpage%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/">Netty-PoolSubpage原理探究</a>详细探究。