---
title: Lucene8.2.0底层架构-ByteBlockPool结构分析
date: 2019-10-06 16:04:05
tags: Lucene、ByteBlockPool
toc: true
categories: Lucene
---
Lucene在建立倒排索引结构时, 需要存储大量的byte数组, 并不能固定需要在一个数组中存放多少byte。数组不能扩展的特性并不能很好的解决动态扩容的问题, Lucene自己开发了类ByteBlockPool来满足这一特性。针对int类型开发了IntBlockPool, 两者原理相同, 本文就以ByteBlockPool存储细节来展开介绍。

# 属性介绍
先看下相关属性:
```
  // 真正存放byte的地方, 还是可以扩容的。
  public byte[][] buffers = new byte[10][];
  //指向二元buffers的第一元指针, 当前正在向第几个byte[]存放数据
  private int bufferUpto = -1;
 //当前正在写入的byte[]内的相对偏移量，就是buffer内可分配的起始位置。
  public int byteUpto = BYTE_BLOCK_SIZE;
  //当前块buffer
  public byte[] buffer;
 // buffer在所有buffers的位置（当前buffer在全局的偏移量），(bufferUpto -1)*BYTE_BLOCK_SIZE
  public int byteOffset = -BYTE_BLOCK_SIZE;
  // 当使用完当前buffer后, 会再次申请一个byte[]来存放byte。每次申请一个byte[], 都对应了一个级别, 每个级别可以申请的内存大小都是固定的。NEXT_LEVEL_ARRAY定义当前级别的下一个级别是第几级, LEVEL_SIZE_ARRAY对应了每个级别可以申请的byte[]大小。最多有9级别, 当超过第9级别后, 后面再申请的话, 只能是第9级。这里使用了一个小技巧来实现这个操作, NEXT_LEVEL_ARRAY[8]=9,表示第8级别后面是第9级, NEXT_LEVEL_ARRAY[9]=9表示第9级后面还是第9级。
  public final static int[] NEXT_LEVEL_ARRAY = {1, 2, 3, 4, 5, 6, 7, 8, 9, 9};
  public final static int[] LEVEL_SIZE_ARRAY = {5, 14, 20, 30, 40, 40, 80, 80, 120, 200};
  // 定义了最开始的分配byte[]的级别
  public final static int FIRST_LEVEL_SIZE = LEVEL_SIZE_ARRAY[0];
```
ByteBlockPool使用二进制buffers来真正存储数据, 当需要扩容时候, 产生新的数组来存储byte。

# 产生buffer
当前`buffer`不足以存放byte时, 就会产生新的32kb大小的byte[]来存放数据。
```
 // buffer已经不足以新的申请需求, 那么就产生新的buffer来存放, 一次申请不能超过32kb大小空间
 public void nextBuffer() {
    // buffers中已存满buffer了, 需要扩容
    if (1+bufferUpto == buffers.length) {
      byte[][] newBuffers = new byte[ArrayUtil.oversize(buffers.length+1,
                                                        NUM_BYTES_OBJECT_REF)][];
      // 只是拷贝了数组引用，并没有copy数组, 这里体现了ByteBlockPool的优势
      System.arraycopy(buffers, 0, newBuffers, 0, buffers.length);
      buffers = newBuffers;
    }
    //产生一个新的buffer, 大小为32kb, 供slice来分配数据
    buffer = buffers[1+bufferUpto] = allocator.getByteBlock();
    bufferUpto++;//buffers写入位置+1

    byteUpto = 0;//buffer写入位置
    byteOffset += BYTE_BLOCK_SIZE;//指针位置初始化
  }
```
产生新buffer过程, 就是对ByteBlockPool扩容的过程, 相对于二进制或者List扩容, ByteBlockPool的优势在于不对数据进行移动, 移动的仅仅是数组指针。 扩容时产生一个32k的byte数组。ByteBlockPool仅仅管理这些byte[]的指针。

# 从buffer中产生slice
ByteBlockPool申请一个空闲的32k的数组buffer。每次写入byte时, 首先会从这个buffer中申请一块区域存放byte, 这块区域就称为一个slice。 我们看下ByteBlockPool是如何管理从buffer中申请slice的。
```
  // 从当前buffer中申请一个size大小的slice
  public int newSlice(final int size) {
     // 当前buffer剩余可用空间装不下size了, 那么我们就需要产生一个新的buffer了
    if (byteUpto > BYTE_BLOCK_SIZE-size)
      nextBuffer();
    // 此时能保证当前buffer一定可以存的下size了
    final int upto = byteUpto;
    byteUpto += size;
    buffer[byteUpto-1] = 16;//赋值为16 调用时 会从byteUpto开始写入，当遇到buffer[pos]位置不为0 时，会调用allocSlice方法 通过 16&15得到当前NEXT_LEVEL_ARRAY中level
    return upto; //申请的slice的相对起始位置
  }
```
新申请的slice的大小是FIRST_LEVEL_SIZE:5, 该函数主要做了如下事情:
1.检查当前buffer剩余空间是否能满足size大小的slice申请, 若不满足, 另外创建一个新的buffer作为当前工作buffer。
2.在当前buffer中申请新的slice, 将该slice最后一位存放16. 代表下一级申请slice的级别大小:16&15=1。
申请过程如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_bytebuffer.png" height="200" width="600"/>
可以看到, 若buffer剩余空间不足以存储, 会去申请新的buffer。

# 扩容现存的slice
ByteBlockPool主要解决的问题是数组动态扩容的问题, 我们不确定未来还有多少字符串需要继续写入, 我们首次申请的slice区域很可能就不够用, 而allocSlice解决的就是对当前slice扩容的问题。扩容前我们首先回答如下几个问题:
什么时候需要扩容? 扩容多大? 每个slice存储结尾都会存放下一个申请内存的slice, 当即将写入的位置不为0, 则说明说明当前byte代表着slice已经存满, 该byte代表着扩容slice申请数组的一个级别。
```
// slice代表当前buffer, upto代表slice结束位置, slice[upto]存放的是下一级别
public int allocSlice(final byte[] slice, final int upto) {
    // 确定下一级别
    final int level = slice[upto] & 15;
    final int newLevel = NEXT_LEVEL_ARRAY[level];
    // 下一级别申请的大小
    final int newSize = LEVEL_SIZE_ARRAY[newLevel];

    // 当前buffer是否还够下一级别的byte[]的申请
    if (byteUpto > BYTE_BLOCK_SIZE-newSize) {
      nextBuffer();
    }
     // 当前块的起始位置
    final int newUpto = byteUpto;
     // upto的在整个buffers的绝对位置
    final int offset = newUpto + byteOffset;
    // 目前buffer内新的可写入位置
    byteUpto += newSize;
    // 从当前级别空余3位出来存储当前slice扩容下一级别的起始位置
    // 第一步, 先空出
    buffer[newUpto] = slice[upto-3];
    buffer[newUpto+1] = slice[upto-2];
    buffer[newUpto+2] = slice[upto-1];
     // 空出来的3位连同级别位存放位置
    // Write forwarding address at end of last slice:
    slice[upto-3] = (byte) (offset >>> 24);
    slice[upto-2] = (byte) (offset >>> 16);
    slice[upto-1] = (byte) (offset >>> 8);
    slice[upto] = (byte) offset;
    // 在新扩容的slice结尾填写下一级别
    // Write new level:
    buffer[byteUpto-1] = (byte) (16|newLevel);
    return newUpto+3;
  }
```
主要过程用图像表示如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_bytebuffer1.png" height="250" width="600"/>
其中address是在整个buffers中的绝对位置。 一个buffer可以产生不同的slice, 每个slice不需要连续, 相当于一个链串结构。 slice扩容分为9级, 达到9级后新申请的内存大小都按照第9级别来分配。

# 总结
ByteBlockPool通过管理字符串二维数组方式实现动态扩容, 而且扩容成本也不高。lucene中在字符串存储上使用了较多较为巧妙的方法, 正是这些巧妙方法, 使得lucene有着比较优秀的性能。


