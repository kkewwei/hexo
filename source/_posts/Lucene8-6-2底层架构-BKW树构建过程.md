---
title: Lucene8.6.2底层架构-BKW树构建过程
date: 2020-11-01 16:22:41
tags: Lucene、BKW树、Point
toc: true
categories: Lucene
---
针对数值型的倒排索引，Lecene从6.X引入了BKD树结构，BKD全称：Block K-Dimension Balanced Tree。在此之前，数值型查找和String结构一样，使用<a href="https://kkewwei.github.io/elasticsearch_learning/2020/02/25/Lucene8-2-0%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-%E8%AF%8D%E5%85%B8fst%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/">FST结构</a>）建立索引，FST结构针对精确匹配存在较大的优势，但是数值型很大部分使用场景为范围查找, BKD树就是解决这类使用场景的。若我们将多维简化为一维时，结构就是bst(二叉查找树)。

# 数据放入内存中
BKD树支持多维范围，数值型包括int,float,point等， 这里就以int类型写入作为示例，将`age`建构为三维：
```
Document document = new Document();
document.add(new IntPoint("age", i, i*i, i%20));
indexWriter.addDocument(document); 
```
IntPoint内部会将多维转变为一维数组，转变过程比较简单，比如int，将转变为长度为3*4=12的byte数组。真正开始在内存中建立索引结构是在`DefaultIndexingChain.indexPoint()`处:
```
  private void indexPoint(int docID, PerField fp, IndexableField field) {
    fp.pointValuesWriter.addPackedValue(docID, field.binaryValue());
  }
  // Point有多个值的话，都会堆砌到一个BytesRef中
   public void PointValuesWriter.addPackedValue(int docID, BytesRef value) { 
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
## 构建kdd文件
flush到文件中指的是形成一个segment，触发条件有两个（同<a href="https://kkewwei.github.io/elasticsearch_learning/2019/10/29/Lucenec%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-fdt-fdx%E6%9E%84%E5%BB%BA%E8%BF%87%E7%A8%8B/#%E5%88%B7%E5%88%B0fdx%E6%96%87%E4%BB%B6">fdx</a>，<a href="https://kkewwei.github.io/elasticsearch_learning/2019/11/15/Lucene%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-dvm-dvm%E6%9E%84%E5%BB%BA%E8%BF%87%E7%A8%8B/#%E5%88%B7%E6%96%B0%E5%88%B0%E6%96%87%E4%BB%B6">dvm</a>、<a href="https://kkewwei.github.io/elasticsearch_learning/2020/02/28/Lucene8-2-0%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-tim-tip%E8%AF%8D%E5%85%B8%E7%BB%93%E6%9E%84%E5%8E%9F%E7%90%86%E7%A0%94%E7%A9%B6/#flush%E5%88%B0%E6%96%87%E4%BB%B6%E4%B8%AD">词典建立</a>一样）：
1.lucene建立的索引结构占用内存或者缓存文档数超过阈值。该check会在每次索引完一个文档后(详见`flushControl.doAfterDocument`)。
2.用户主动调用indexWriter.flush()触发。
刷新建立BKD树时，我们首先进入`PointValuesWriter.flush()`：
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
        // 这个数据的偏移量
        final long offset = (long) packedBytesLength * ords[i]; 
        packedValue.length = packedBytesLength;
        bytes.setRawBytesRef(packedValue, offset);
      }
     // 第i个元素的第k位
      @Override
      public byte getByteAt(int i, int k) {
        final long offset = (long) packedBytesLength * ords[i] + k;
         // 从BytePool中读取offset
        return bytes.readByte(offset);
      }
    };
    final PointValues values = points; 
    writer.writeField(fieldInfo, reader); 
  }
```
这里用MutablePointValues封装了读取每个元素的方法，需要进入Lucene86PointsWriter.writeField看下。我们需要知道，Lucene86PointsWriter定义了BKD每个叶子存放的元素不能超过512个（BKDWriter.DEFAULT_MAX_POINTS_IN_LEAF_NODE），大小不能超过16MB（BKDWriter.DEFAULT_MAX_MB_SORT_IN_HEAP）：
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
      // 每个叶子在kdd中开始存放的位置
      final long[] leafBlockFPs = new long[numLeaves]; 
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
2.首先统计每个维度的最大值&最小值， 以便决定是从哪个维度开始切分排序
3.进入`BKDWriter.build()`函数开始递归构建每个维度。原则上，对于当前所有元素，首先按照512个元素为1个节点，放入完全二叉树的叶子中。按照从跟节点从上向下切分所有的叶子节点，切分前查找每个维度的最大最小差值，以这个维度将切分的左右子树保持局部有序。
4.将构建好的BKD树元数据存放在kdi和kdm文件中.

`BKDWriter.build`将切分过程&局部有序分成两个阶段：
1.切分到叶子节点后也保证叶子内某个维度有序。
2.当没切分到叶子节点时，保证左右子树局部有序。

### 当不是叶子节点时，需要split，保证某一维度有序
这里看下如何还不是叶子节点时的切分过程：
```
      final int splitDim;
      // compute the split dimension and partition around it
      // 只有一个维度
      if (numIndexDims == 1) { 
        splitDim = 0;
        // 至少两个维度的数据
      } else { 
        // for dimensions > 2 we recompute the bounds for the current inner node to help the algorithm choose best
        // split dimensions. Because it is an expensive operation, the frequency we recompute the bounds is given
        // by SPLITS_BEFORE_EXACT_BOUNDS.
         // 大于2个维度的话，为了最合适的拆分，需要重新找下最大值和最小值
        if (numLeaves != leafBlockFPs.length && numIndexDims > 2 && Arrays.stream(parentSplits).sum() % SPLITS_BEFORE_EXACT_BOUNDS == 0) {
          computePackedValueBounds(reader, from, to, minPackedValue, maxPackedValue, scratchBytesRef1);
        }
        // 查找每个维度最大值和最小值差值的最大的那个维度，决定以这个维度开始拆分
        splitDim = split(minPackedValue, maxPackedValue, parentSplits);
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
       // 拆分时那个节点的偏移量
      final int splitOffset = rightOffset - 1;
      // set the split value
      final int address = splitOffset * bytesPerDim;
      // 以哪个节点哪个维度开始切分
      splitDimensionValues[splitOffset] = (byte) splitDim;
 
      reader.getValue(mid, scratchBytesRef1);
      System.arraycopy(scratchBytesRef1.bytes, scratchBytesRef1.offset + splitDim * bytesPerDim, splitPackedValues, address, bytesPerDim);

      byte[] minSplitPackedValue = ArrayUtil.copyOfSubArray(minPackedValue, 0, packedIndexBytesLength); //从minPackedValue中copy一份最小值
      byte[] maxSplitPackedValue = ArrayUtil.copyOfSubArray(maxPackedValue, 0, packedIndexBytesLength); //从maxPackedValue中copy一份最大值
       //重新定义左子树的最小值
      System.arraycopy(scratchBytesRef1.bytes, scratchBytesRef1.offset + splitDim * bytesPerDim,
              minSplitPackedValue, splitDim * bytesPerDim, bytesPerDim);
      System.arraycopy(scratchBytesRef1.bytes, scratchBytesRef1.offset + splitDim * bytesPerDim,
              maxSplitPackedValue, splitDim * bytesPerDim, bytesPerDim);

      // recurse
      // 统计哪个维度被切分了
      parentSplits[splitDim]++; 
      // 左中
      build(leavesOffset, numLeftLeafNodes, reader, from, mid, out,
              minPackedValue, maxSplitPackedValue, parentSplits,
              splitPackedValues, splitDimensionValues, leafBlockFPs, spareDocIds);
      // 中又
      build(rightOffset, numLeaves - numLeftLeafNodes, reader, mid, to, out,
              minSplitPackedValue, maxPackedValue, parentSplits,
              splitPackedValues, splitDimensionValues, leafBlockFPs, spareDocIds);
      parentSplits[splitDim]--;
    }
```
根据当前数据进行排序切分，主要做了如下事情：
1.判断是否需要重新(精确)统计from->to所有维度最大值、最小值，仅当父类节点在当前维度每切分4次（SPLITS_BEFORE_EXACT_BOUNDS）才统计，因为精确统计所有元素的边界是一个非常昂贵的操作。
2.在`BKDWriter.split()`根据最大值最小值边界差距最大的那个维度确定以哪个维度排序。同时为了考虑每个维度被选中切分排序的次数不能差距太大，规定了，每个维度切分排序次数不能相差2倍，若切分次数太少了，会强制选择切分次数最小的那个维度。
3.通过构建完全二叉树来计算左子树、右子树分别有多少个叶子节点。
4.确定在被切分的维度上，最大值和最小值有多少个相同的前缀，以便压缩存储。
5.使用`MutablePointsReaderUtils.partition`来进行排序。保证当前维度下，左子树的值小于右子树。
6.重新恢复左子树的所有维度的最大值：`maxSplitPackedValue`和右子树的所有维度的最小值：`minSplitPackedValue`
7.分别递归左子树和右子树，再继续查找分隔维度并在此维度上进行排序。最终形成了每个子树只在一个维度保证了左右有序。切分过程如下：
<img src="https://kkewwei.github.io/elasticsearch_learning/img/bkd1.png" height="350" width="350"/>

我们再看下`MutablePointsReaderUtils.partition`如何在splitDim维度上保证左边的数值小于等于mid对应的值、右边的数大于等于mid对应的值。这里其实使用了堆排&快排的思想完成的排序。
```
  public static void partition(int numDataDim, int numIndexDim, int maxDoc, int splitDim, int bytesPerDim, int commonPrefixLen,
                               MutablePointValues reader, int from, int to, int mid,
                               BytesRef scratch1, BytesRef scratch2) {
    // 这个元素内的偏移量：该维度
    final int dimOffset = splitDim * bytesPerDim + commonPrefixLen; 
    // 需要比较的位数
    final int dimCmpBytes = bytesPerDim - commonPrefixLen; 
    // 整个point的数据结尾位置
    final int dataOffset = numIndexDim * bytesPerDim; 
    // 该元素不相同的个数
    final int dataCmpBytes = (numDataDim - numIndexDim) * bytesPerDim + dimCmpBytes; 
    final int bitsPerDocId = PackedInts.bitsRequired(maxDoc - 1);
     //  这里位数为两类，可以从byteAt()看出，读取每一类的方式也不一样
    new RadixSelector(dataCmpBytes + (bitsPerDocId + 7) / 8) {
    // 第一类就是普通的dimCmpBytes，读取的是不相同的字符；第二类是 (bitsPerDocId + 7) / 8， 读取的是文档Id，就是说把文档Id作为了桶的一部分
      @Override
      // 使用快排进行排序
      protected Selector getFallbackSelector(int k) { 
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
            // 通过文档大小相比较
            return pivotDoc - reader.getDocID(j); 
          }
        };
      }
      @Override
      protected void swap(int i, int j) {
        reader.swap(i, j);
      }
      // 可以从maxLength=dataCmpBytes + (bitsPerDocId + 7) / 8可以看出，属于不同的读法，
      @Override
       // 第i个元素，第k个byte
      protected int byteAt(int i, int k) {
      // 读取的是dataCmpBytes中的数据
        if (k < dimCmpBytes) { 
          return Byte.toUnsignedInt(reader.getByteAt(i, dimOffset + k));
        } else if (k < dataCmpBytes) {
          return Byte.toUnsignedInt(reader.getByteAt(i, dataOffset + k - dimCmpBytes));
        } else {
         // 读取的是docId的高位，通过(k - dataCmpBytes)去掉原本影响
          final int shift = bitsPerDocId - ((k - dataCmpBytes + 1) << 3);
          // 应该仅仅是为了hash，从docId中取值 
          return (reader.getDocID(i) >>> Math.max(0, shift)) & 0xff; 
        }
      }
    }.select(from, to, mid);
  }
```
排序逻辑我们需要注意两个地方：
1.每个元素当前切分维度splitDim的逻辑位数：dataCmpBytes + (bitsPerDocId + 7) / 8。我们当使用基数排序时，会确定每个元素的位数，然后可以从高位->低位的排序。同样，这里对每个元素的第splitDim维度产生虚拟位数。针对from-to的元素的第splitDim维度，首先使用从不相同的byte进行排序，其次基于docId进行排序。
2.byteAt中定义了逻辑位数的获取方法，k表示逻辑元素的位数，当k小于该元素该维度的不同起始位时，获取具体的byte；当k超过不同位数dimCmpBytes时，则比较文档Id大小。
我们看下RadixSelector里最核心的排序算法部分：
```
  private void radixSelect(int from, int to, int k, int d, int l) {
     //统计的是每一字符多少个数据
    final int[] histogram = this.histogram;
    // 每次使用都得清空
    Arrays.fill(histogram, 0); 

    final int commonPrefixLength = computeCommonPrefixLengthAndBuildHistogram(from, to, d, histogram);
    // 所有元素中有相同的前缀
    if (commonPrefixLength > 0) { 
      // if there are no more chars to compare or if all entries fell into the
      // first bucket (which means strings are shorter than d) then we are done
      // otherwise recurse
      if (d + commonPrefixLength < maxLength
          && histogram[0] < to - from) {
        // 忽略过相同的前缀部分
        radixSelect(from, to, k, d + commonPrefixLength, l); 
      }
      return;
    }
    assert assertHistogram(commonPrefixLength, histogram);
    // 没有相同的元素
    int bucketFrom = from;
    // 遍历每一个桶
    for (int bucket = 0; bucket < HISTOGRAM_SIZE; ++bucket) { 
      // 获取桶里面数据的范围
      final int bucketTo = bucketFrom + histogram[bucket]; 
       // 若读取的数据个数超过中位数，
      if (bucketTo > k) {
       // 通过快排的思想，完成左边小于桶，右边大于桶
        partition(from, to, bucket, bucketFrom, bucketTo, d);
        // 最中间的那个桶继续排序
        if (bucket != 0 && d + 1 < maxLength) {
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
该函数主要做了如下事情：
1.通过computeCommonPrefixLengthAndBuildHistogram统计第splitDim维度逻辑位数第d位相同的前缀commonPrefixLength，并且统计每个字母出现的次数histogram。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/bkd2.png" height="300" width="570"/>
2.若commonPrefixLength大于0，代表该维度逻辑位数为d位有相同的前缀。同时检测到还有逻辑位数没有排序完，那么直接跳到逻辑位数不同的位数继续进行排序。
3.否则，该维度逻辑位数为d位没有相同的前缀，那么就统计下，第k（初始化时为mid）个数逻辑位数d的值在哪个histogram维度内，然后调用`partition()`按照快排的思想将找出bucketTo-bucketFrom的值放在中间，这样，左边的值都小于中间的那组值，右边的值都大于中间的那组值。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/bkd3.png" height="350" width="450"/>
4.继续递归，完成第k个所在那档的元素所在splitDim维度逻辑为数第d+1位有序，直到这档splitDim维度逻辑完全有序。
此时，所有元素在splitDim维度维度，形成了相对排序：以k为分隔， 第k个元素左边的所有元素均小于等于k，第k个元素右边的所有元素均大于等于k。

### 当递归到叶子节点时，需要split
此时叶子节点内的元素并没有按照某一个维度有序。每个叶子的处理顺序比较有序，是从第一个叶子、第二个、第三个、、、、最后一个叶子的顺序进行的。这部分主要是将叶子节所有元素点如何高效的存储起来。
```
      // 叶子节点的起始位置
      final int count = to - from; 
      //计算这批数据里面每个域相同的前缀
      // Compute common prefixes
      Arrays.fill(commonPrefixLengths, bytesPerDim);
      // 读取第from个元素所有维度的值
      reader.getValue(from, scratchBytesRef1);
      //从下个开始，找相同的前缀 
      for (int i = from + 1; i < to; ++i) { 
        reader.getValue(i, scratchBytesRef2);
        // 比较每个维度从from->to中相同的前缀
        for (int dim=0;dim<numDataDims;dim++) { 
          final int offset = dim * bytesPerDim;
          int dimensionPrefixLength = commonPrefixLengths[dim];
          commonPrefixLengths[dim] = FutureArrays.mismatch(scratchBytesRef1.bytes, scratchBytesRef1.offset + offset, // 不同数据的起点
                  scratchBytesRef1.offset + offset + dimensionPrefixLength,
                  scratchBytesRef2.bytes, scratchBytesRef2.offset + offset,
                  scratchBytesRef2.offset + offset + dimensionPrefixLength);
          if (commonPrefixLengths[dim] == -1) {
            // 两个字符串一模一样，那么不修改相同前缀长度
            commonPrefixLengths[dim] = dimensionPrefixLength;
          }
        }
      }

      // Find the dimension that has the least number of unique bytes at commonPrefixLengths[dim]
      FixedBitSet[] usedBytes = new FixedBitSet[numDataDims];
      for (int dim = 0; dim < numDataDims; ++dim) {
         // 所有字符不一样长
        if (commonPrefixLengths[dim] < bytesPerDim) {
         //因为最多有128个字符，这里用256位就满足了.只有不一样的才会被赋值
          usedBytes[dim] = new FixedBitSet(256); 
        }
      } // 统计不一样的那个维度，去重之后可以分为多少个字符
      for (int i = from + 1; i < to; ++i) {
        for (int dim=0;dim<numDataDims;dim++) {
          if (usedBytes[dim] != null) { // 该维度值不一样
            byte b = reader.getByteAt(i, dim * bytesPerDim + commonPrefixLengths[dim]);
            usedBytes[dim].set(Byte.toUnsignedInt(b));
          }
        }
      }
      // 统计两个维度中distinct字母最少的那个维度
      int sortedDim = 0; 
       // distinct后的值个数
      int sortedDimCardinality = Integer.MAX_VALUE;
      for (int dim = 0; dim < numDataDims; ++dim) {
        if (usedBytes[dim] != null) {
          final int cardinality = usedBytes[dim].cardinality();
          if (cardinality < sortedDimCardinality) {
            sortedDim = dim;
            sortedDimCardinality = cardinality;
          }
        }
      } // 将数据以最集中的那个维度排序
      // 每个维度都有一系列数据，这系列数据在某位开始不同，统计开始不同这位有多少个distinct个数据，找到这几个维度中，distinct最小的那个维度，进行排序
      // sort by sortedDim
      MutablePointsReaderUtils.sortByDim(numDataDims, numIndexDims, sortedDim, bytesPerDim, commonPrefixLengths,
              reader, from, to, scratchBytesRef1, scratchBytesRef2);

      BytesRef comparator = scratchBytesRef1;
      BytesRef collector = scratchBytesRef2;
       // 读取排序后的第一个元素，被比较的数
      reader.getValue(from, comparator);
      // 获取的是元素（全维度）与后面一个元素不相同的个数
      int leafCardinality = 1; 
      for (int i = from + 1; i < to; ++i) {
         // 读取下一个元素, collector是最新的数
        reader.getValue(i, collector); 
        // 几个维度，只有前面一个和后面有一个不相同，就leafCardinality+1
        for (int dim =0; dim < numDataDims; dim++) { 
          // 从不同之处开始比较 
          final int start = dim * bytesPerDim + commonPrefixLengths[dim];
          final int end = dim * bytesPerDim + bytesPerDim;
          // 如果不是完全一样
          if (FutureArrays.mismatch(collector.bytes, collector.offset + start, collector.offset + end,
                  comparator.bytes, comparator.offset + start, comparator.offset + end) != -1) {
            // 每个value都不同
            leafCardinality++; 
            // 在交换collector和comparator的值，是想前后比较是否一致
            BytesRef scratch = collector;
            collector = comparator;
            comparator = scratch;
            // 直接退出了, 所以交换没啥用
            break; 
          }
        }
      }
      // Save the block file pointer:
      leafBlockFPs[leavesOffset] = out.getFilePointer(); // kdd
      // Write doc IDs
      int[] docIDs = spareDocIds;
      for (int i = from; i < to; ++i) {
        // 获取from->to之间的文档Id
        docIDs[i - from] = reader.getDocID(i); 
      }
      //System.out.println("writeLeafBlock pos=" + out.getFilePointer());
       // 把文档Id给存储起来了
      writeLeafBlockDocs(scratchOut, docIDs, 0, count); 
      // 存储相同的前缀
      // Write the common prefixes:
       // copy第一个词
      reader.getValue(from, scratchBytesRef1);
      System.arraycopy(scratchBytesRef1.bytes, scratchBytesRef1.offset, scratch1, 0, packedBytesLength);
      // 存储前缀
      writeCommonPrefixes(scratchOut, commonPrefixLengths, scratch1); 
      // Write the full values:
      IntFunction<BytesRef> packedValues = new IntFunction<BytesRef>() {
        @Override
        public BytesRef apply(int i) {
          reader.getValue(from + i, scratchBytesRef1);
          return scratchBytesRef1;
        }
      };
      // 再写入叶子剩余数据
      writeLeafBlockPackedValues(scratchOut, commonPrefixLengths, count, sortedDim, packedValues, leafCardinality);
      // // 写入kdd文件
      out.writeBytes(scratchOut.getBytes(), 0, scratchOut.getPosition()); 
      scratchOut.reset();
```
具体做了如下事情：
1.遍历from-to所有元素，依次比较每个元素同一个维度相同的前缀长度，存放在commonPrefixLengths。
2.统计每个维度不相同前缀元素的cardinaliy，统计时使用长度为256的FixedBitSet，代表着256个字符。
3.遍历每个维度，找出cardinaliy值最小的那个维度sortedDim。
4.基于sortedDim维度，调用`MutablePointsReaderUtils.sortByDim`使用快排原理保证叶子内所有元树在sortedDim维度有序。
5.统计from-to个元素的cardinaliy，这里每个维度都要对比。
6.存储from-to个元素的docId
7.存储每个维度相同的前缀。
8.使用`BKDWriter.writeLeafBlockPackedValues()`存储from-to个具体的元素。
kdd文件结构如下：
<img src="https://kkewwei.github.io/elasticsearch_learning/img/bkd5.png" height="100" width="900"/>


## 构建kdm和kdi文件
当对所有元素进行排序后，开始存储BKD树的每个子节点和叶子节点，会进入到：
```
 private void writeIndex(IndexOutput metaOut, IndexOutput indexOut, int countPerLeaf, BKDTreeLeafNodes leafNodes, long dataStartFP) throws IOException {
    byte[] packedIndex = packIndex(leafNodes);
    writeIndex(metaOut, indexOut, countPerLeaf, leafNodes.numLeaves(), packedIndex, dataStartFP);
  }
```
该函数主要做了两件事：
1.在`packIndex`中压缩存储BKD树子节点和叶子节点。
2.在`writeIndex`中存储压缩后的数据，及BKD元数据。

### 压缩转存BKD树
压缩BKD转存的核心函数是`recursePackIndex`，采用递归的方式转存，以中序遍历的方式对BKD树进行处理，首先先存储中间飞叶子的信息，然后再分别对左右叶子节点进行处理:
```
  // 到了叶子节点
 if (numLeaves == 1) { 
      if (isLeft) {
        return 0;
      } else {
        long delta = leafNodes.getLeafLP(leavesOffset) - minBlockFP;
        writeBuffer.writeVLong(delta);
        return appendBlock(writeBuffer, blocks);
      }
      // 不是叶子节点
    } else { 
      long leftBlockFP;
      if (isLeft) {
        // 若是左子树，leftBlockFP就是父节点的minBlockFP
        leftBlockFP = minBlockFP;
      } else {
        // 右子树的最小leftBlockFP，就是当前右子树包含的的所有叶子节点的第一个，也就是leavesOffset对应的叶子节点
        leftBlockFP = leafNodes.getLeafLP(leavesOffset); 
        long delta = leftBlockFP - minBlockFP;
        assert leafNodes.numLeaves() == numLeaves || delta > 0 : "expected delta > 0; got numLeaves =" + numLeaves + " and delta=" + delta;
        writeBuffer.writeVLong(delta);
      }

      int numLeftLeafNodes = getNumLeftLeafNodes(numLeaves);
      final int rightOffset = leavesOffset + numLeftLeafNodes;
      final int splitOffset = rightOffset - 1;
      // 和构建时一样，是以哪个维度切分的，然后address指向下一个位置（value值）
      int splitDim = leafNodes.getSplitDimension(splitOffset);
      BytesRef splitValue = leafNodes.getSplitValue(splitOffset);// 这个维度切分时的值
      int address = splitValue.offset;

      //System.out.println("recursePack inner nodeID=" + nodeID + " splitDim=" + splitDim + " splitValue=" + new BytesRef(splitPackedValues, address, bytesPerDim));
       // 查找切分的那个值和之前切分之间的之间相同的前缀
      // find common prefix with last split value in this dim:
      int prefix = FutureArrays.mismatch(splitValue.bytes, address, address + bytesPerDim, lastSplitValues,
              splitDim * bytesPerDim, splitDim * bytesPerDim + bytesPerDim);
      if (prefix == -1) {
        prefix = bytesPerDim;
      }

      int firstDiffByteDelta;
      if (prefix < bytesPerDim) { // 两次切分的值是不同的
        //System.out.println("  delta byte cur=" + Integer.toHexString(splitPackedValues[address+prefix]&0xFF) + " prev=" + Integer.toHexString(lastSplitValues[splitDim * bytesPerDim + prefix]&0xFF) + " negated?=" + negativeDeltas[splitDim]);
        firstDiffByteDelta = (splitValue.bytes[address+prefix]&0xFF) - (lastSplitValues[splitDim * bytesPerDim + prefix]&0xFF);
        // 第二次作为切分阶度，那么就开始获取diff
        if (negativeDeltas[splitDim]) {
           // 取相反数
          firstDiffByteDelta = -firstDiffByteDelta; 
        }
        assert firstDiffByteDelta > 0; 
      } else {
        firstDiffByteDelta = 0;
      }
      // 将prefix、splitDim和firstDiffByteDelta打包编码到同一个vint中，也很容易解码出来。见BKDReader.readNodeData()中287行编码
      // pack the prefix, splitDim and delta first diff byte into a single vInt:
      int code = (firstDiffByteDelta * (1+bytesPerDim) + prefix) * numIndexDims + splitDim;
      writeBuffer.writeVInt(code);

      // write the split value, prefix coded vs. our parent's split value:
      int suffix = bytesPerDim - prefix;
      byte[] savSplitValue = new byte[suffix];
      if (suffix > 1) {// 不完全一样
        // 把这个split分词的那个词后半段存储起来
        writeBuffer.writeBytes(splitValue.bytes, address+prefix+1, suffix-1);
      }

      byte[] cmp = lastSplitValues.clone(); // 不再是同一个对象
      // 把lastSplitValues中的不相同的后缀全部copy到savSplitValue中
      System.arraycopy(lastSplitValues, splitDim * bytesPerDim + prefix, savSplitValue, 0, suffix); // 临时保存
      // 将splitPackedValues中该不相同的后缀copy，放到lastSplitValues对应的位置
      // copy our split value into lastSplitValues for our children to prefix-code against
      System.arraycopy(splitValue.bytes, address+prefix, lastSplitValues, splitDim * bytesPerDim + prefix, suffix);
      // 将writeBuffer存储的值，以[]byte方式放入blocks中，放的code，词的后半缀
      int numBytes = appendBlock(writeBuffer, blocks); 

      // placeholder for left-tree numBytes; we need this so that at search time if we only need to recurse into the right sub-tree we can
      // quickly seek to its starting point
      int idxSav = blocks.size();
      // 占位符，后面会赋值，会记录左子树的长度
      blocks.add(null); 
      // 若我们以splitDim维度进行了切分，那么之后再次在维度
      boolean savNegativeDelta = negativeDeltas[splitDim];
      negativeDeltas[splitDim] = true;
      int leftNumBytes = recursePackIndex(writeBuffer, leafNodes, leftBlockFP, blocks, lastSplitValues, negativeDeltas, true,
              leavesOffset, numLeftLeafNodes);
      if (numLeftLeafNodes != 1) {
        writeBuffer.writeVInt(leftNumBytes);
      } else { // 最左边的那个叶子节点
        assert leftNumBytes == 0: "leftNumBytes=" + leftNumBytes;
      }
       // 存储的leftNumBytes的长度
      int numBytes2 = Math.toIntExact(writeBuffer.getFilePointer());
      byte[] bytes2 = new byte[numBytes2];
      writeBuffer.writeTo(bytes2, 0);
      writeBuffer.reset();
      // replace our placeholder:
      blocks.set(idxSav, bytes2);

      negativeDeltas[splitDim] = false;// 置位
      int rightNumBytes = recursePackIndex(writeBuffer,  leafNodes, leftBlockFP, blocks, lastSplitValues, negativeDeltas, false,
              rightOffset, numLeaves - numLeftLeafNodes);
      // 这里会复位
      negativeDeltas[splitDim] = savNegativeDelta; 
      // restore lastSplitValues to what caller originally passed us:
      // 再放回去
      System.arraycopy(savSplitValue, 0, lastSplitValues, splitDim * bytesPerDim + prefix, suffix); 
      // 当前非叶子节点存储使用的空间，分中+左右占用
      return numBytes + bytes2.length + leftNumBytes + rightNumBytes;
    }
```
按照中序遍历的方式存储，比如处理node1节点:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/bkd4.png" height="550" width="570"/>
其中：
deltaFP：leftBlockFP - minBlockFP，minBlockFP是父节点最左边的子节点，leftBlockFP是该节点的子节点。
code: (firstDiffByteDelta * (1+bytesPerDim) + prefix) * numIndexDims + splitDim，firstDiffByteDelta是当前节点切分维度的value-该相同维度上一个父节点切分的value。这里采用了编码，使之存储三个数值。
最终，BKD树存储在了数组blocks中。

### 存储bkm和bki文件
在`BKDWriter.writeIndex`文件中，bki文件存储了blocks的二进制数，而bkm文件存储了BKD树的元数据信息：
```
 private void writeIndex(IndexOutput metaOut, IndexOutput indexOut, int countPerLeaf, int numLeaves, byte[] packedIndex, long dataStartFP) throws IOException {
     // dim文件写入
    CodecUtil.writeHeader(metaOut, CODEC_NAME, VERSION_CURRENT);
    metaOut.writeVInt(numDataDims);
    metaOut.writeVInt(numIndexDims);
     // 每个页节点的元素个数
    metaOut.writeVInt(countPerLeaf);
    metaOut.writeVInt(bytesPerDim);

    // 统计该树每个维度最大最小值
    metaOut.writeVInt(numLeaves);
    metaOut.writeBytes(minPackedValue, 0, packedIndexBytesLength);
    metaOut.writeBytes(maxPackedValue, 0, packedIndexBytesLength);

    metaOut.writeVLong(pointCount);
    metaOut.writeVInt(docsSeen.cardinality());
    // 把非叶子节点在文件中存储位置给读取出来
    metaOut.writeVInt(packedIndex.length);
    metaOut.writeLong(dataStartFP);
    // If metaOut and indexOut are the same file, we account for the fact that
    // writing a long makes the index start 8 bytes later.
    metaOut.writeLong(indexOut.getFilePointer() + (metaOut == indexOut ? Long.BYTES : 0));
   // bki文件
    indexOut.writeBytes(packedIndex, 0, packedIndex.length);
  }
```

# 总结
BKD树主要运用在范围多维查找，在空间上，按照完全二叉树结构，将数据分为左右两部分，找到所有元素每个维度[min,max]差距最大的维度，在该维度按照照左子树完全小于中间值，右子树完全大于间的值。通过范围查找，能够快速定位出docId。