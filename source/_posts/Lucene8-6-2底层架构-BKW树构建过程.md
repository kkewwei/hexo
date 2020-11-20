---
title: Lucene8.6.2底层架构-BKW树构建过程
date: 2020-11-01 16:22:41
tags: Lucene、BKW树、Point
toc: true
categories: Lucene
---
针对数值型的倒排索引，Lecene从6.X引入了BKD树结构，BKD全称：Block K-Dimension Balanced Tree。在此之前，数值型查找和String结构一样，使用<a href="https://kkewwei.github.io/elasticsearch_learning/2020/02/25/Lucene8-2-0%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-%E8%AF%8D%E5%85%B8fst%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/">FST结构</a>）建立索引，FST结构针对精确匹配存在较大的优势，但是数值型很大部分使用场景为范围查找, BKD树就是解决这类使用场景的，将多维简化为一维时，结构就是bst(二叉查找树)。

# 数据放入内存中
BKD树支持多维范围，数值型包括int,float,point等， 这里就以int类型写入作为示例，将`age`建构为三维：
```
Document document = new Document();
document.add(new IntPoint("age", i, i*i, i%20));
indexWriter.addDocument(document); 
```
IntPoint内部会将多维转变为一维数组，转变过程比较简单，比如int，将转变为长度为3*4=12的byte数组。真正开始在内存中建立索引结构是在`DefaultIndexingChain.indexPoint()处`:
```
  private void indexPoint(int docID, PerField fp, IndexableField field) {
    fp.pointValuesWriter.addPackedValue(docID, field.binaryValue());// 这里添加下数据
  }
   public void PointValuesWriter.addPackedValue(int docID, BytesRef value) { // Point有多个值的话，都会堆砌到一个BytesRef中
    bytes.append(value);
    docIDs[numPoints] = docID; 
    if (docID != lastDocID) {
      numDocs++;
      lastDocID = docID;
    }
    numPoints++;
  }
```
其中，使用ByteBlockPool弹性扩容的功能存储byte数组的value， docIDs记录的是第几个元素对应的文档号。numPoints记录的是元素的个数（一个元素可以由多个域构成），因为存在一个字段，会存储多个元素的情况。

# 将内存中的结构flush到磁盘
flush到文件中指的是形成一个segment，触发条件有两个（同<a href="https://kkewwei.github.io/elasticsearch_learning/2019/10/29/Lucenec%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-fdt-fdx%E6%9E%84%E5%BB%BA%E8%BF%87%E7%A8%8B/#%E5%88%B7%E5%88%B0fdx%E6%96%87%E4%BB%B6">fdx</a>，<a href="https://kkewwei.github.io/elasticsearch_learning/2019/11/15/Lucene%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-dvm-dvm%E6%9E%84%E5%BB%BA%E8%BF%87%E7%A8%8B/#%E5%88%B7%E6%96%B0%E5%88%B0%E6%96%87%E4%BB%B6">dvm</a>、<a href="https://kkewwei.github.io/elasticsearch_learning/2020/02/28/Lucene8-2-0%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-tim-tip%E8%AF%8D%E5%85%B8%E7%BB%93%E6%9E%84%E5%8E%9F%E7%90%86%E7%A0%94%E7%A9%B6/#flush%E5%88%B0%E6%96%87%E4%BB%B6%E4%B8%AD">词典建立</a>一样）：
1.lucene建立的索引结构占用内存或者缓存文档书超过阈值。该check会在每次索引完一个文档后(详见`flushControl.doAfterDocument`)。
2.用户主动调用indexWriter.flush()触发。
刷新时，也就是同时建立BKD树的时候，我们首先进入`PointValuesWriter.flush()`：
```
  public void flush(SegmentWriteState state, Sorter.DocMap sortMap, PointsWriter writer) throws IOException {
    // 封装了读取元素&文档的方法
    PointValues points = new MutablePointValues() {
      // 给每个元素都编了号。若之后对该域每个元素进行排序，也仅仅是对这个号排序。
      final int[] ords = new int[numPoints]; 
      {
        for (int i = 0; i < numPoints; ++i) {
          ords[i] = i; 
        }
      }
      // ords存放的是排序后的元素编号，通过docIDs来查找真正的docId
      @Override
      public int getDocID(int i) {  return docIDs[ords[i]]; }
     // 读取第i个元素的值
      @Override
      public void getValue(int i, BytesRef packedValue) {
        final long offset = (long) packedBytesLength * ords[i]; // 这个数据的偏移量
        packedValue.length = packedBytesLength;
        bytes.setRawBytesRef(packedValue, offset);
      }
     // 第i个元素的第k位
      @Override
      public byte getByteAt(int i, int k) {
        final long offset = (long) packedBytesLength * ords[i] + k;
        return bytes.readByte(offset); // 从BytePool中读取offset
      }
    };
    final PointValues values = points; 
    writer.writeField(fieldInfo, reader); 
  }
```
这里用MutablePointValues封装了读取每个元素的方法，需要进入Lucene86PointsWriter.writeField看下，我们需要知道下，Lucene86PointsWriter定义了BKD每个叶子存放的元素不能超过512个（BKDWriter.DEFAULT_MAX_POINTS_IN_LEAF_NODE），大小不能超过16MB（BKDWriter.DEFAULT_MAX_MB_SORT_IN_HEAP）：
```
  @Override
  public void writeField(FieldInfo fieldInfo, PointsReader reader) throws IOException {
    PointValues values = reader.getValues(fieldInfo.name);
    try (BKDWriter writer = new BKDWriter(writeState.segmentInfo.maxDoc(),
                                          writeState.directory,
                                          writeState.segmentInfo.name,
                                          fieldInfo.getPointDimensionCount(),
                                          fieldInfo.getPointIndexDimensionCount(),
                                          fieldInfo.getPointNumBytes(),
                                          maxPointsInLeafNode,
                                          maxMBSortInHeap,
                                          values.size())) {

      if (values instanceof MutablePointValues) {
        Runnable finalizer = writer.writeField(metaOut, indexOut, dataOut, fieldInfo.name, (MutablePointValues) values);
        if (finalizer != null) {
          metaOut.writeInt(fieldInfo.number);
          finalizer.run();
        }
        return;
      }
    }
  }
```
BKDWriter函数就是构建BKD数的核心类， 需要继续进入BKDWriter.writeField->writeFieldNDims看如何构建的，我们以元素为2+维进行介绍，一维的是其简化版：
```
    private Runnable writeFieldNDims(IndexOutput metaOut, IndexOutput indexOut, IndexOutput dataOut, String fieldName, MutablePointValues values) throws IOException {
      finished = true;

      pointCount = values.size();
      // 统计多少个叶子节点，一个叶子节点存放512个元素
      final int numLeaves = Math.toIntExact((pointCount + maxPointsInLeafNode - 1) / maxPointsInLeafNode);
      final int numSplits = numLeaves - 1;

      // 第1位放置当前级的Dim，第2部分防止分割时候的当前值。记录了每个NodeId都是从哪离开始切割的。完全二叉树，1，2，3
      final byte[] splitPackedValues = new byte[numSplits * bytesPerDim];
      final byte[] splitDimensionValues = new byte[numSplits];
      final long[] leafBlockFPs = new long[numLeaves]; // 每个叶子在dim中开始存放的位置
      // 获取每个维度的最大值与最小值
      // compute the min/max for this slice
      computePackedValueBounds(values, 0, Math.toIntExact(pointCount), minPackedValue, maxPackedValue, scratchBytesRef1);
      for (int i = 0; i < Math.toIntExact(pointCount); ++i) {
        docsSeen.set(values.getDocID(i));
      }
      // 开始构造BKD树
      final long dataStartFP = dataOut.getFilePointer();
      // 将统计每个维度拆分的次数，若存在某个维度切分次数不足最大的一半，那么本次将选择这个维度切分，以便尽量避免每个维度拆分次数差距过大，而导致查询毛刺
      final int[] parentSplits = new int[numIndexDims]; 
      build(0, numLeaves, values, 0, Math.toIntExact(pointCount), dataOut,
              minPackedValue.clone(), maxPackedValue.clone(), parentSplits,
              splitPackedValues, splitDimensionValues, leafBlockFPs,
              new int[maxPointsInLeafNode]);

      scratchBytesRef1.length = bytesPerDim;
      scratchBytesRef1.bytes = splitPackedValues;
      BKDTreeLeafNodes leafNodes  = new BKDTreeLeafNodes() {
        @Override
        public long getLeafLP(int index) {
          return leafBlockFPs[index];
        }

        @Override
        public BytesRef getSplitValue(int index) {
          scratchBytesRef1.offset = index * bytesPerDim;
          return scratchBytesRef1;
        }

        @Override
        public int getSplitDimension(int index) {
          return splitDimensionValues[index] & 0xff;
        }

        @Override
        public int numLeaves() {
          return leafBlockFPs.length;
        }
      };

      return () -> {
        try { // metaOut:kdm文件   indexOut:kdi文件
          writeIndex(metaOut, indexOut, maxPointsInLeafNode, leafNodes, dataStartFP); // 写dim文件
        } catch (IOException e) {
          throw new UncheckedIOException(e);
        }
      };
  }
```
该函数主要做了如下事情：
1.计算了该BDK的叶子数
2.首先统计每个维度的最大值&最小值， 以便决定是从哪个维度开始切分
3.进入`BKDWriter.build()`函数开始递归构建每个维度。这里先简要描述BKD树的构建的模型：
<img src="https://kkewwei.github.io/elasticsearch_learning/img/bkd1.png" height="160" width="250"/>
对于当前所有元素，原则上，查找当前维度的最大最小差值，以这个维度将数据切split开，递归直到当前所有元素的个数不能超过256个（叶子节点）
`BKDWriter.build`具体如何实现上述切分过程分成两个阶段：
1.切分到叶子节点后具体处理。
2.当没切分到叶子节点时，递归切分。

## 当不是叶子节点时，需要split
这里看下如何还不是叶子节点时的切分过程：
```
      final int splitDim;
      // compute the split dimension and partition around it
      if (numIndexDims == 1) { // 只有一个维度
        splitDim = 0;
      } else { // 至少两个维度的数据
        // for dimensions > 2 we recompute the bounds for the current inner node to help the algorithm choose best
        // split dimensions. Because it is an expensive operation, the frequency we recompute the bounds is given
        // by SPLITS_BEFORE_EXACT_BOUNDS. // 大于2个维度的话，为了最好的拆分，需要重新找下最大值和最小值
        if (numLeaves != leafBlockFPs.length && numIndexDims > 2 && Arrays.stream(parentSplits).sum() % SPLITS_BEFORE_EXACT_BOUNDS == 0) {
          computePackedValueBounds(reader, from, to, minPackedValue, maxPackedValue, scratchBytesRef1);
        }
        splitDim = split(minPackedValue, maxPackedValue, parentSplits);// 每个维度最大值和最小值差值的最大的那个维度。决定以这个维度开始拆分
      }
      // 左子树叶子节点
      // How many leaves will be in the left tree:
      int numLeftLeafNodes = getNumLeftLeafNodes(numLeaves);
      // How many points will be in the left tree:
      final int mid = from + numLeftLeafNodes * maxPointsInLeafNode; // 那么重点节点的index编号
      // 确定最大值和最小值相同的前缀长度
      int commonPrefixLen = FutureArrays.mismatch(minPackedValue, splitDim * bytesPerDim,
              splitDim * bytesPerDim + bytesPerDim, maxPackedValue, splitDim * bytesPerDim,
              splitDim * bytesPerDim + bytesPerDim);
      if (commonPrefixLen == -1) {
        commonPrefixLen = bytesPerDim;
      }
      // 通过基数排序+快排实现了排序，保证中间数左右有序
      MutablePointsReaderUtils.partition(numDataDims, numIndexDims, maxDoc, splitDim, bytesPerDim, commonPrefixLen,
              reader, from, to, mid, scratchBytesRef1, scratchBytesRef2);

      final int rightOffset = leavesOffset + numLeftLeafNodes;
      final int splitOffset = rightOffset - 1; // 拆分时那个节点的偏移量
      // set the split value
      final int address = splitOffset * bytesPerDim;
      splitDimensionValues[splitOffset] = (byte) splitDim;// 以哪个节点哪个维度开始切分
      reader.getValue(mid, scratchBytesRef1);// 把中间那个值给读取出来，放在scratchBytesRef1中，然
      System.arraycopy(scratchBytesRef1.bytes, scratchBytesRef1.offset + splitDim * bytesPerDim, splitPackedValues, address, bytesPerDim);

      byte[] minSplitPackedValue = ArrayUtil.copyOfSubArray(minPackedValue, 0, packedIndexBytesLength); //从minPackedValue中copy一份最小值
      byte[] maxSplitPackedValue = ArrayUtil.copyOfSubArray(maxPackedValue, 0, packedIndexBytesLength); //从maxPackedValue中copy一份最大值
      System.arraycopy(scratchBytesRef1.bytes, scratchBytesRef1.offset + splitDim * bytesPerDim,
              minSplitPackedValue, splitDim * bytesPerDim, bytesPerDim); //重新定义左子树的最小值
      System.arraycopy(scratchBytesRef1.bytes, scratchBytesRef1.offset + splitDim * bytesPerDim,
              maxSplitPackedValue, splitDim * bytesPerDim, bytesPerDim);//

      // recurse
      parentSplits[splitDim]++; // 统计哪个维度被切分了
      build(leavesOffset, numLeftLeafNodes, reader, from, mid, out,// 左中
              minPackedValue, maxSplitPackedValue, parentSplits,
              splitPackedValues, splitDimensionValues, leafBlockFPs, spareDocIds);
      build(rightOffset, numLeaves - numLeftLeafNodes, reader, mid, to, out,// 中又
              minSplitPackedValue, maxPackedValue, parentSplits,
              splitPackedValues, splitDimensionValues, leafBlockFPs, spareDocIds);
      parentSplits[splitDim]--;
    }
```
根据当前数据进行切分，主要做了如下事情：
1.判断是否需要重新(精确)统计所有元素所有维度最大值、最小值，仅当每4次（SPLITS_BEFORE_EXACT_BOUNDS）才统计，因为精确统计所有元素的边界是一个非常昂贵的操作。
2.在`BKDWriter.split()`根据最大值最小值边界差距最大的那个维度确定以哪个维度切分。同时为了考虑每个维度被选中切分的次数不能差距太大，规定了，每个维度切分次数不能相差2倍，若切分次数太少了，会强制选择切分次数最小的那个维度。
3.通过构建完全二叉树来计算左子树、右子树分别有多少个叶子节点。
4.确定在被切分的维度上，最大值和最小值有多少个相同的前缀。
5.使用`MutablePointsReaderUtils.partition`来进行排序。保证当前维度下，左子树的值小于右子树。
6.重新产生左子树的所有维度的最大值`maxSplitPackedValue`和右子树的所有维度的最小值：minPackedValue
7.分别处理左子树和右子树。
切分过程如下：
<img src="https://kkewwei.github.io/elasticsearch_learning/img/bkd1.png" height="350" width="350"/>

我们在看下`MutablePointsReaderUtils.partition`如何在splitDim维度上保证左边的数值小于等于mid对应的值，右边的数大于等于mid对应的值。这里其实使用了堆排&快排的思想完成的排序。
```
  public static void partition(int numDataDim, int numIndexDim, int maxDoc, int splitDim, int bytesPerDim, int commonPrefixLen,
                               MutablePointValues reader, int from, int to, int mid,
                               BytesRef scratch1, BytesRef scratch2) {
    final int dimOffset = splitDim * bytesPerDim + commonPrefixLen; // 相同纬度的数据，从这个位开始不一致了（绝对值）
    final int dimCmpBytes = bytesPerDim - commonPrefixLen; // 需要比较的位数
    final int dataOffset = numIndexDim * bytesPerDim; // 整个point的数据结尾位置
    final int dataCmpBytes = (numDataDim - numIndexDim) * bytesPerDim + dimCmpBytes; // 该元素不相同的个数
    final int bitsPerDocId = PackedInts.bitsRequired(maxDoc - 1);
    new RadixSelector(dataCmpBytes + (bitsPerDocId + 7) / 8) { //  这里位数为两类，可以从byteAt()看出，读取每一类的方式也不一样
    // 第一类就是普通的dimCmpBytes，读取的是不相同的字符；第二类是 (bitsPerDocId + 7) / 8， 读取的是文档Id，就是说把文档Id作为了桶的一部分
      @Override
      protected Selector getFallbackSelector(int k) { // 使用快排进行排序
        final int dataStart = (k < dimCmpBytes) ? dataOffset : dataOffset + k - dimCmpBytes;
        final int dataEnd = numDataDim * bytesPerDim;
        return new IntroSelector() {

          final BytesRef pivot = scratch1;
          int pivotDoc;

          @Override
          protected void swap(int i, int j) {
            reader.swap(i, j);
          }

          @Override
          protected void setPivot(int i) {
            reader.getValue(i, pivot);
            pivotDoc = reader.getDocID(i);
          }

          @Override
          protected int comparePivot(int j) {
            if (k < dimCmpBytes) {
              reader.getValue(j, scratch2);
              int cmp = FutureArrays.compareUnsigned(pivot.bytes, pivot.offset + dimOffset + k, pivot.offset + dimOffset + dimCmpBytes,
                  scratch2.bytes, scratch2.offset + dimOffset + k, scratch2.offset + dimOffset + dimCmpBytes);
              if (cmp != 0) {
                return cmp;
              }
            }
            if (k < dataCmpBytes) {
              reader.getValue(j, scratch2);
              int cmp = FutureArrays.compareUnsigned(pivot.bytes, pivot.offset + dataStart, pivot.offset + dataEnd,
                  scratch2.bytes, scratch2.offset + dataStart, scratch2.offset + dataEnd);
              if (cmp != 0) {
                return cmp;
              }
            }
            return pivotDoc - reader.getDocID(j); // 通过文档大小相比较
          }
        };
      }

      @Override
      protected void swap(int i, int j) {
        reader.swap(i, j);
      }
      // 可以从maxLength=dataCmpBytes + (bitsPerDocId + 7) / 8可以看出，属于不同的读法，
      @Override
      protected int byteAt(int i, int k) { // 第i个文档，第k个byte
        if (k < dimCmpBytes) { // 读取的是dataCmpBytes中的数据
          return Byte.toUnsignedInt(reader.getByteAt(i, dimOffset + k));
        } else if (k < dataCmpBytes) {
          return Byte.toUnsignedInt(reader.getByteAt(i, dataOffset + k - dimCmpBytes));
        } else { // 读取的是docId的高位
          final int shift = bitsPerDocId - ((k - dataCmpBytes + 1) << 3); // 通过(k - dataCmpBytes)去掉原本影响，，每次向右移8位,比如第一次是22，第二次是14
          return (reader.getDocID(i) >>> Math.max(0, shift)) & 0xff; // 应该仅仅是为了hash，从docId中取值
        }
      }
    }.select(from, to, mid);
  }
```
排序逻辑我们需要注意两个地方：
1.每个元素当前切分维度splitDim的逻辑位数：dataCmpBytes + (bitsPerDocId + 7) / 8。我们当使用基数排序时，会确定每个元素的位数，然后可以从高位->地位的排序。同样，这里对每个元素的第splitDim维度进行了虚拟位数。
2.
我们看下RadixSelector里最核心的排序算法部分：
```
  private void radixSelect(int from, int to, int k, int d, int l) {
    final int[] histogram = this.histogram; //统计的是每一字符多少个数据
    Arrays.fill(histogram, 0); // 每次使用都得清空

    final int commonPrefixLength = computeCommonPrefixLengthAndBuildHistogram(from, to, d, histogram);
    if (commonPrefixLength > 0) { // 所有元素中有相同的前缀
      // if there are no more chars to compare or if all entries fell into the
      // first bucket (which means strings are shorter than d) then we are done
      // otherwise recurse
      if (d + commonPrefixLength < maxLength
          && histogram[0] < to - from) {
        radixSelect(from, to, k, d + commonPrefixLength, l); // 忽略过相同的前缀部分
      }
      return;
    }
    assert assertHistogram(commonPrefixLength, histogram);
    // 没有相同的元素
    int bucketFrom = from; //
    for (int bucket = 0; bucket < HISTOGRAM_SIZE; ++bucket) { // 遍历每一个桶
      final int bucketTo = bucketFrom + histogram[bucket]; // 获取桶里面数据的范围

      if (bucketTo > k) { // 若读取的数据个数超过中位数，
        partition(from, to, bucket, bucketFrom, bucketTo, d); // 通过快排的思想，完成左边小于桶，右边大于桶

        if (bucket != 0 && d + 1 < maxLength) {// 最中间的那个桶继续排序
          // all elements in bucket 0 are equal so we only need to recurse if bucket != 0
          select(bucketFrom, bucketTo, k, d + 1, l + 1);
        }
        return;
      }
      bucketFrom = bucketTo;
    }
    throw new AssertionError("Unreachable code");
  }
```



