---
title: Netty-PoolSubpage原理探究
date: 2018-07-22 01:02:45
tags:
---
Netty中大于8K的内存是通过PoolChunk来分配的, 小于8k的内存是通过PoolSubpage分配的, 本章将详细描述如何通过PoolSubpage分配小于8K的内存。当申请小于8K的内存时, 会从分配一个8k的叶子节点, 若用不完的话, 存在很大的浪费, 所以通过PoolSubpage来管理8K的内存, 如下图
<img src="http://owsl7963b.bkt.clouddn.com/PoolSubpage.png" height="400" width="450"/>
每一个PoolSubpage都会与PoolChunk里面的一个叶子节点映射起来, 然后将PoolSubpage根据用户申请的ElementSize化成几等分, 之后只要再次申请ElementSize大小的内存, 将直接从这个PoolSubpage中分配。
下面是PoolSubpage的构造函数:
```
   PoolSubpage(PoolSubpage<T> head, PoolChunk<T> chunk, int memoryMapIdx, int runOffset, int pageSize, int elemSize) {
        this.chunk = chunk;
        //与PoolChunkPage中哪个节点映射一起来
        this.memoryMapIdx = memoryMapIdx;
        this.runOffset = runOffset;
        //该叶子节点的大小
        this.pageSize = pageSize;
        //bitmap的每一位都描述的是一个element的使用情况
        bitmap = new long[pageSize >>> 10];
        init(head, elemSize);
    }
    //init根据当前需要分配的内存大小，确定需要多少个bitmap元素
    void init(PoolSubpage<T> head, int elemSize) {   ///elemSize代表此次申请的大小，比如申请64byte，那么这个page被分成了8k/64=2^7=128个
        doNotDestroy = true;
        this.elemSize = elemSize;
        if (elemSize != 0) {
            maxNumElems = numAvail = pageSize / elemSize; //被分成了128份64大小的内存
            nextAvail = 0;
            bitmapLength = maxNumElems >>> 6;  //6代表着long长度 = 2
            if ((maxNumElems & 63) != 0) {//低6位
                bitmapLength ++;
            }

            for (int i = 0; i < bitmapLength; i ++) {
                bitmap[i] = 0;
            }
        }
        addToPool(head);
    }
```
有人会有疑问: 为啥bitmap = new long[pageSize >>> 10], 因为小于8K的内存, 最小是以16b为单位来分配的, 一个long类型为64位, 最多需要pageSize/(16\*64)个long就可以将一个PoolSubpage中所有element是否分配描述清楚了, log2(16*64)=10。
然后再调用init来对PoolSubpage结构进行初始化:
1. 总共可以分成`pageSize/elemSize`个element。 bitmap所有元素也不一定需要全部用上, 实际会用`maxNumElems >>> 6`个long就可以了。
2. 然后根据头插法将该PoolSubpage插入tinySubpagePools或者smallSubpagePools(参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/05/23/Netty%E5%86%85%E5%AD%98%E5%AD%A6%E4%B9%A0/">Netty PoolArea原理探究</a>)对应级别的链中。

# PoolSubpage的内存分配
```
    long allocate() { //找到一位
        if (elemSize == 0) {
            return toHandle(0);
        }
        if (numAvail == 0 || !doNotDestroy) {
            return -1;
        }
        final int bitmapIdx = getNextAvail(); //64进制
        int q = bitmapIdx >>> 6;//第几位long
        int r = bitmapIdx & 63; //这个long第几位
        assert (bitmap[q] >>> r & 1) == 0;
        bitmap[q] |= 1L << r;  //将相关位置置为1
        if (-- numAvail == 0) { //这个page没有再能够提供的bit位
            removeFromPool(); //从可分配链中去掉
        }
        return toHandle(bitmapIdx);
    }

```
若可分配的element个数为0, 则将相应可分配链中去掉(该链结构可参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/05/23/Netty%E5%86%85%E5%AD%98%E5%AD%A6%E4%B9%A0/">Netty PoolArea原理探究</a>)。 PoolSubpage在初始化时候已经规定了可分配内存块的大小, 所以调用的函数中不需要告诉需要分配的内存大小, 在PoolSubpage中查找可用的element通过findNextAvail0实现的:
```
    private int findNextAvail0(int i, long bits) {
        final int maxNumElems = this.maxNumElems; //包含的段个数
        final int baseVal = i << 6; //64进制，第2位，最大数也只是8*64 + 64

        for (int j = 0; j < 64; j ++) {
            if ((bits & 1) == 0) { //bits哪位为0，局说明哪位可用
                int val = baseVal | j; //第4个long+9  i<<6 + j
                if (val < maxNumElems) { //不能大于总段数
                    return val;
                } else {
                    break;
                }
            }
            bits >>>= 1;  //一位位找，直到找到某位为0
        }
        return -1;
    }
```
主要做了如下事情:
1. 遍历bitmap中所有元素, 找到一个bit为0的元素, 说明还有element没有分配出去。
2. 检查该元素每一个bit是否为0, 找到后, 返回值为`i << 6 | j`, 包含第几个元素 + 元素内第几个bit位。
再返回到allocate函数中, 将该元素对应的bit置为1, 并构造handle:
```
    private long toHandle(int bitmapIdx) {
        ////高32放着一个PoolSubpage里面哪段的哪个，低32位放着哪个叶子节点
        return 0x4000000000000000L | (long) bitmapIdx << 32 | memoryMapIdx;
    }
```
可以得知针对小于8K的分配, 返回的handle一定是大于int类型的数据, 若handle高位不为0, 则该handle映射的内存块一定是小于8K的内存块。该handle将page、element的位置信息全部编码进去了, 这些信息也很容易解码出来。

# PoolSubpage的内存释放
```
    boolean free(PoolSubpage<T> head, int bitmapIdx) {
        if (elemSize == 0) {
            return true;
        }
        int q = bitmapIdx >>> 6;
        int r = bitmapIdx & 63;
        assert (bitmap[q] >>> r & 1) != 0;
        bitmap[q] ^= 1L << r;

        setNextAvail(bitmapIdx);

        if (numAvail ++ == 0) {
            addToPool(head);
            return true;
        }

        if (numAvail != maxNumElems) {
            return true;
        } else {
            // Subpage not in use (numAvail == maxNumElems)
            if (prev == next) {
                // Do not remove if this subpage is the only one left in the pool.
                return true;
            }

            // Remove this subpage from the pool if there are other subpages left in the pool.
            doNotDestroy = false;
            removeFromPool();
            return false;
        }
    }
```
释放的时候并不会将该PoolSubpage从subpages(见<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/20/Netty-PoolChunk%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/">Netty PoolChunk原理探究</a>)中取出, 释放逻辑也比较简单, 除了将bitmap中相应long置为0, 别的工作就是检查该PoolSubpage是否需要加入对应的链或者从对应的链中取出:
1. 若当前可用的element为0, 则说明已经从相应可分配链中去掉了, 此时再通过头插法加入对应的可分配链中, 说明还在使用。
2. 若目前剩余可用element小于最大剩余可用的, 同样说明还在使用。
3. 反之说明该PoolSubpage没有element被分配出去。 若该级别的链大于1个PoolSubpage还在使用, 则返回该链可以释放了; 只剩余这个PoolSubpage还没有释放, 那暂时先不释放等待之后的内存分配。

# 总结

PoolSubpage主要管理小于8K的内存分配, 内存返回的handle可以唯一确定PoolChunk、poolSubpage中具体哪个element。 当PoolSubpage没有使用的内存时, 可能从对应的链中取出。同时一定没有从subpages中去掉。