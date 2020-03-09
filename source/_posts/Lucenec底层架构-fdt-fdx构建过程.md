---
title: Lucene8.2.0底层架构-fdt/fdx构建过程
date: 2019-10-29 07:47:47
tags: Lucene、StoredField
toc: true
categories: Lucene
---
StoredField文件主要存放文档->域->value的关系, 每个文档的所有域都保存一起,再保存下一行, 行式存储, 正排索引结果, 不对字段分词, 便于读取任何字段的value。 fdt存放文档的value及一级索引索引, fdx存放二级索引结构。 本文就以这两个文件结构的建立->文件写入磁盘的过程来进行详细介绍, 代码主要可见CompressingStoredFieldsWriter。

# 内存索引结构的建立
索引结构建立是指将文档按照规范存储到内存中, 当对字段进行如下设置时, 才会建立该索引文档:
```
     FieldType fieldType = new FieldType();
     fieldType.setStored(true);
```
实际是在`DefaultIndexingChain.processField()`里面开始检查字段是否设置了该参数:
```
    // Add stored fields:
    if (fieldType.stored()) {
      if (fp == null) {
        fp = getOrAddField(fieldName, fieldType, false);
      }
      if (fieldType.stored()) {
        // 域的值
        String value = field.stringValue();
        try {
          // 面向行的存储，docValue是面向列的存储, field值存放在CompressingStoredFieldsWriter的bufferedDocs中
          storedFieldsConsumer.writeField(fp.fieldInfo, field);
        }
      }
    }
```
该函数主要做了两件事:
1. 检查该域之前文档是否已经写入过。详情可参考:
2. 若设置了stored=true, 那么最终进入到进入CompressingStoredFieldsWriter.writeField, 将字段的value缓存起来。
```
  public void writeField(FieldInfo info, IndexableField field) {
    ++numStoredFieldsInDoc;
    int bits = 0;
    Number number = field.numericValue();
    if (number != null) {
      if (number instanceof Byte || number instanceof Short || number instanceof Integer) {
        bits = NUMERIC_INT;
      } else if (number instanceof Long) {
        bits = NUMERIC_LONG;
      }
      ......
      string = null;
      bytes = null;
    } else {
      bytes = field.binaryValue();
      // 是二进制数的话
      if (bytes != null) {
        bits = BYTE_ARR;
        string = null;
      } else {
        bits = STRING; // 是字符串的话
        string = field.stringValue();
      }
    }
    // 低三位代表类型，高5位代表字段编号
    final long infoAndBits = (((long) info.number) << TYPE_BITS) | bits;
    //写入字段编号，字段类型
    bufferedDocs.writeVLong(infoAndBits);
    if (bytes != null) {
      bufferedDocs.writeVInt(bytes.length);
      bufferedDocs.writeBytes(bytes.bytes, bytes.offset, bytes.length);
    } else if (string != null) {
      bufferedDocs.writeString(string);
    } else {
      if (number instanceof Byte || number instanceof Short || number instanceof Integer) {
        bufferedDocs.writeZInt(number.intValue());
      } else if (number instanceof Long) {
        writeTLong(bufferedDocs, number.longValue());
      }
      ......
      }
    }
  }
```
该函数主要做了如下两件事情:
1.读取字段value, 判断字段类型。字段value类型分为以下五种:
```
  static final int         STRING = 0x00;
  static final int       BYTE_ARR = 0x01;
  static final int    NUMERIC_INT = 0x02;
  static final int  NUMERIC_FLOAT = 0x03;
  static final int   NUMERIC_LONG = 0x04;
  static final int NUMERIC_DOUBLE = 0x05;
```
存储时使用3bit表示。
2.向内存对象bufferedDocs存储value值, bufferedDocs使用`byte[] bytes`存储数据, 内存中的建立的索引结构如下所示:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_fdt1.png" height="160" width="900"/>
需要明确以下两点:
+ 所有文档相同字段名称的fieldNuber是一样的。
+ 所有doc所有域依次按先后顺序存储起来, stored=true构建的存储结构是按行存储的, 属于正排索引, 通过该索引结构很方便读取每个文档每个字段的值。

# flush()/产生fdt文件
这里flush的主要作用是将内存中的所有文档结构生成一个chunk, 并将这些结构刷新到fdt文件中, 成为一个chunk, 并不会产生一个segment。每当完成对一个文档建立了索引后, 便会检查缓存在内存中的文档数/bufferedDocs树是否超过阈值,超过了则会触发flush。一个chunk在内存中最大占用16KB/128个文档, 参数在构建CompressingStoredFieldsFormat时给写死了, 详细代码可参考: DefaultIndexingChain.finishStoredFields(), 本节从CompressingStoredFieldsWriter.finishDocument开始介绍。
```
  public void finishDocument() throws IOException {
     // this.numStoredFields存放的当前缓存的每个文档中设置了stred=true的域个数
    if (numBufferedDocs == this.numStoredFields.length) {
      final int newLength = ArrayUtil.oversize(numBufferedDocs + 1, 4);
      this.numStoredFields = ArrayUtil.growExact(this.numStoredFields, newLength);
      endOffsets = ArrayUtil.growExact(endOffsets, newLength);
    }
    // 该doc有几个域需要存储
    this.numStoredFields[numBufferedDocs] = numStoredFieldsInDoc;
    numStoredFieldsInDoc = 0;
    // endOffsets存储的是该文档的内容在bufferedDocs中的位置, 便于查找每个文档所有域内容
    endOffsets[numBufferedDocs] = bufferedDocs.getPosition();
    // 缓存文档数+1
    ++numBufferedDocs;
    // 该bufferedDocs使用大小超过chunk限制16k了，或者该bufferedDocs存储文档书超过chunk限制128了。那么就该创建一个chunk
    if (triggerFlush()) {
      // 将缓存中的文档创建一个chunk, 建立一级索引
      flush();
    }
  }
```
该函数主要功能:
+ 缓存该文档的信息, 包括该文档中stored=true的域个数、文档byte在bufferedDocs占用的结束位置。
+ 检查建立索引缓存的文档个数/bufferedDocs长度是否超过阈值, 超过的话, 调用flush将缓存的数据建立一个chunk, 作为一级索引。
我们看下flush怎么用缓存的文档产生一个chunk:
```
  // 两个地方会调用：1. 每次写完128/16K文档时 2.用户调用flush，若还有缓存文档，则会主动调用
  private void flush() throws IOException {
    // 该文档占用的空间  OutputStreamIndexOutput   fdx
    indexWriter.writeIndex(numBufferedDocs, fieldsStream.getFilePointer());
    // transform end offsets into lengths
    final int[] lengths = endOffsets;
    for (int i = numBufferedDocs - 1; i > 0; --i) {
      // 转化每个文档的offset->文档的长度
      lengths[i] = endOffsets[i] - endOffsets[i - 1];
    }
    // 若存储总长度大于32kb, 就需要分段分开压缩
    final boolean sliced = bufferedDocs.getPosition() >= 2 * chunkSize;
    // 完成numStoredFields与lengths的落盘
    writeHeader(docBase, numBufferedDocs, numStoredFields, lengths, sliced);
    // compress stored fields to fieldsStream
    // 若切分的话，就切分压缩存储，一次压缩写入不能超过chunk大小（16k）
    if (sliced) {
      // chunk太大了, 分16k分别存储
      for (int compressed = 0; compressed < bufferedDocs.getPosition(); compressed += chunkSize) {
        compressor.compress(bufferedDocs.getBytes(), compressed, Math.min(chunkSize, bufferedDocs.getPosition() - compressed), fieldsStream);
      }
    } else {
      compressor.compress(bufferedDocs.getBytes(), 0, bufferedDocs.getPosition(), fieldsStream); // LZ4FastCompressor.compress并写入了
    }
    // reset
    docBase += numBufferedDocs;
    numBufferedDocs = 0;
    bufferedDocs.reset();
    numChunks++;
  }
```
该刷新过程主要是将部分索引结构刷到fdt中, 具体事情如下:
1. 调用indexWriter.writeIndex(), 缓存该chunk的文档数、在fdt中记录的起始位置, 为fdx文件构建一级索引结构。
2. 获取每个文档在fdt中的长度。
3. 若内存中缓存的所有文档长度大于2*16kb, 则将bufferedDocs中的数据切分压缩存储到fdt中。
4. 清空bufferedDocs中的数据。
fdt文件结构如下所示:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_fdt2.png" height="160" width="550"/>


我们需要了解下indexWriter.writeIndex()缓存了哪些索引数据:
```
  void writeIndex(int numDocs, long startPointer) throws IOException {
    // 在block满的时候才会一起写入fdx
    if (blockChunks == blockSize) {
      writeBlock(); // 足够一个block了
      reset();
    }
    // 只有经过writeBlock后才会置-1
    if (firstStartPointer == -1) {
      firstStartPointer = maxStartPointer = startPointer;
    }
    // 这个chunk的文档数
    docBaseDeltas[blockChunks] = numDocs;
    // fdt文件的绝对位置
    startPointerDeltas[blockChunks] = startPointer - maxStartPointer;
    ++blockChunks;
    blockDocs += numDocs;
    totalDocs += numDocs;
    maxStartPointer = startPointer; // 当前写入的最大值
  }
```
1.检查当前缓存的chunk个数是否达到阈值1024, 每个chunk文档数最大个数128个。若达到了, 调用writeBlock()将chunk的索引结构写入fdx中, 我们在第三节会详细介绍该函数。
2.记录该chunk的文档数、该chunk在fdt中的起始位置。

# 刷到fdx文件
刷新到fdx文件，只会产生一个block, block记录的是一批chunk的索引结构, 一个fdx可以存储不止一个block, 产生一个block主要有一下两种情况:
1.内存缓存的chunk个数超过阈值1024, 会在每次产生一个block的时候检查。
2.用户主动通过indexWriter.flush()触发（此时一个block的chunk数小于1024）。
结束一个fdx文件的写入（产生一个segment）, 有一下两种情况:
1.lucene建立的索引结构占用内存或者缓存文档书超过阈值, 会在每次索引一个文档的时候检查。
2.用户主动通过indexWriter.flush()触发。
针对第一种情况, 每当对一个文档建立完成后, 就会检查缓存的文档数或者lucene构建索引共占用的内存是否超过阈值, 我们可以通过如下参数设置这些阈值:
```
   indexWriterConfig = (IndexWriterConfig) indexWriter.getConfig();
   //统计的是距上次flush到目前内存中缓存的数据
   indexWriterConfig.setMaxBufferedDocs(201);
   indexWriterConfig.indexWriterConfig.setRAMBufferSizeMB(16)
```
lucene默认内存参数设置16mb, 文档个数缓存无限制。而在es中, 默认也没有对文档个数限制, 而对总的内存使用做出限制, 可以通过indices.memory.index_buffer_size设置, 默认10%内存, 若超过了内存限制, 则会进行所有索引文件的刷新工作, 当然也包括fdt/fdx文件的构建。 检查是否超过阈值的代码可参考DocumentsWriterFlushControl.doAfterDocument()。
实际完成一个fdx文件写入的代码可以看下StoredFieldsConsumer.flush()函数:
```
  void flush(SegmentWriteState state, Sorter.DocMap sortMap) throws IOException {
    try {
      // CompressingStoredFieldsWriter, 将所有chunk的索引结构写入fdx。新建的索引结构写入新的fdt、fdt中
      writer.finish(state.fieldInfos, state.segmentInfo.maxDoc());
    } finally {
      IOUtils.close(writer);
      // 每flush一次, 就会清空writer。下次写入时产生一个新的writer
      writer = null;
    }
  }
```
这里我们可以看到一个事实: 每次由于内存满了而刷新时, CompressingStoredFieldsWriter对象就被置空。也就是说, 若因为构建索引的内存超过阈值, 那么将所有chunk组成一个block, 写入fdx文档。 之后新来的数据全部写到新的fdt及fdx文件中。
我们需要重点看下这里的finish做了那些事情:
```
  public void finish(FieldInfos fis, int numDocs) throws IOException {
    // 若此时还有缓存的文档不够一个chunk文档数, 则将这些缓存的文档构建一个chunk。
    if (numBufferedDocs > 0) {
      flush();
      // chunk不够128个文档
      numDirtyChunks++; // incomplete: we had to force this flush
    }
    // 向fdx写入chunk的索引结构。
    indexWriter.finish(numDocs, fieldsStream.getFilePointer());
    fieldsStream.writeVLong(numChunks); // fdt
    fieldsStream.writeVLong(numDirtyChunks);
    // 关闭fdt文件。
    CodecUtil.writeFooter(fieldsStream);
  }
```
主要事情如下:
1.若内存缓存有文档(不足一个chunk), 则强制产生一个chunk。比如用户主动调用indexWriter.flush()时, 内幕才能缓存的文档就不足一个chunk。
2.调用indexWriter.finish()将chunk的索引结构写入fdx。
3.结尾fdt文件写入。

再接着看下fdx中的block存储了一批chunk的哪些索引结构:
```
  // 只有fdx/fdt文件时候,
  void finish(int numDocs, long maxPointer) throws IOException {
    // 内存中有chunk还没刷到fdx中, 则将这些chunk
    if (blockChunks > 0) {
      writeBlock();
    }
    fieldsIndexOut.writeVInt(0); // end marker
    fieldsIndexOut.writeVLong(maxPointer);
    CodecUtil.writeFooter(fieldsIndexOut);
  }
  private void writeBlock() throws IOException {
    assert blockChunks > 0;
    // 向fdx中写入当前block中chunk的个数
    fieldsIndexOut.writeVInt(blockChunks);
    // doc bases
    final int avgChunkDocs;
    if (blockChunks == 1) {
      avgChunkDocs = 0;
    } else {
     // 去掉最后一个chun空的主要原因是chunk可能不满128个文档
      avgChunkDocs = Math.round((float) (blockDocs - docBaseDeltas[blockChunks - 1]) / (blockChunks - 1)); // 这个block中每个chunk的平均文档个数
    }
    // 这个block的其实文档在整个fdt中的文档起始位置
    fieldsIndexOut.writeVInt(totalDocs - blockDocs);
     // 再储存这个block的chunk平均文档树
    fieldsIndexOut.writeVInt(avgChunkDocs);
    int docBase = 0;
    long maxDelta = 0;
    // 仅仅是为了获取delta的最大差值
    for (int i = 0; i < blockChunks; ++i) {
     // 与标准相差多少文档
      final int delta = docBase - avgChunkDocs * i;
       // 编码映射：比如-n映射成2n-1。 获取最大的误差
      maxDelta |= zigZagEncode(delta);
      docBase += docBaseDeltas[i];
    }
    final int bitsPerDocBase = PackedInts.bitsRequired(maxDelta);
     // 根据maxDelta来确定需要几位来装数
    fieldsIndexOut.writeVInt(bitsPerDocBase);
    PackedInts.Writer writer = PackedInts.getWriterNoHeader(fieldsIndexOut,
        PackedInts.Format.PACKED, blockChunks, bitsPerDocBase, 1);
    docBase = 0;
    // 这里只存储实际文档个数与理论个数的差值
    for (int i = 0; i < blockChunks; ++i) {
      final long delta = docBase - avgChunkDocs * i;
      assert PackedInts.bitsRequired(zigZagEncode(delta)) <= writer.bitsPerValue();
      writer.add(zigZagEncode(delta));
      docBase += docBaseDeltas[i];
    }
    writer.finish();
    // 该 block 在 fdx 文件的起始位置指针
    fieldsIndexOut.writeVLong(firstStartPointer);
    final long avgChunkSize;
    if (blockChunks == 1) {
      avgChunkSize = 0;
    } else {
      avgChunkSize = (maxStartPointer - firstStartPointer) / (blockChunks - 1);
    }
    fieldsIndexOut.writeVLong(avgChunkSize);
    long startPointer = 0;
    maxDelta = 0; // 为了找到最大的delta
    for (int i = 0; i < blockChunks; ++i) {
      startPointer += startPointerDeltas[i];
      final long delta = startPointer - avgChunkSize * i;
      maxDelta |= zigZagEncode(delta);
    }
    final int bitsPerStartPointer = PackedInts.bitsRequired(maxDelta);
    fieldsIndexOut.writeVInt(bitsPerStartPointer); // fdx
    writer = PackedInts.getWriterNoHeader(fieldsIndexOut, PackedInts.Format.PACKED,
        blockChunks, bitsPerStartPointer, 1);
    startPointer = 0;
    for (int i = 0; i < blockChunks; ++i) {
      startPointer += startPointerDeltas[i];
      final long delta = startPointer - avgChunkSize * i;
      assert PackedInts.bitsRequired(zigZagEncode(delta)) <= writer.bitsPerValue();
      writer.add(zigZagEncode(delta));
    }
    writer.finish();
  }
```
writeBlock主要记录以下几个指标:
1.blockChunks: 当前block的chunks数。
2.docBase: 这个block的其实文档在整个fdt中的文档起始位置, 一个fdt可能存放不只一个block。
3.avgChunkDocs: 该block所有chunk的平均文档数
4.bitsPerDocBase: 每个chunk文档书与平均文档数相差个数需要几位bit表示: 比如chunk文档数分别是: 5,3,8, 则需要1位。 该参数是为了保存他们的差值, 减少存储所需要的空间。
5.blockChunks个docBaseDelta: 存储blockChunks个chunk与标准文档的差值, 这里不存储每个文档的原值, 而是存储差值, 是为了减少存储所需的空间。
6.firstStartPointer: 该block在整个fdt存储的起始位置
7.avgChunksSize: 该block所有chunk的平均length
8.blockChunks个startPointDelta: 存储blockChunks个chunk与标准size的差值。 通过该值可以知道每个chunk在fdt中的起始位置, 存储原理与第5项一致。
9.numberChunks: 所有block中总的chunk个数。
10.numberDirtyChunks: fdx可能存放多个block, 一个block最多存放1024个chunk, 该值统计的是所有block的chunk小于1024的个数, 其中有的chunk可能不够128个文档, 称之为dirtyChunk。一般fdx文件中, 只有最后一个block的最后一个chunk小于128个文档。
下图展示了fdt文件和fdx文件的结构:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_fdt.png" height="350" width="900"/>

# 总结
整体过程可以总结如下: 依次将文档写入内存中, 若缓存的文档数或者bufferedDocs占用的内存超过阈值, 则触发一次flush, 该flush将之前的存储在内存中的文档产生一个chunk, 当达到blockSize次flush后, 产生一个block。当达到一个block后, 才开始向fdt文件和fdx文件写入索引结构。一个fdx可能包含多个block的索引结构。