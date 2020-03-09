---
title: Lucene8.2.0底层架构-dvd/dvm构建过程
date: 2019-11-15 13:54:10
tags: Lucene、DocValue
toc: true
categories: Lucene
---
DocValue主要作用是聚合, 不对字段分词, 属于正排索引结构, 面向列式存储。   分为两部分: DocValueData(dvd)、DocValueMeta(dvm)、

# 内存索引结构的建立
DocValue不对域的value分词, 而termVector会对域进行分词, 也就是说, 若设置了docValue, 那么再设置termVector将抛异常。对于每次即将刷新到文档dvd/dvm中缓存的文档, 对于相同域名称的域的term将存放到一起, 形成列式存储, 然后再放下个域的term。 字段只有如下设置, 才会存储DocValue索引:
```
fieldType.setDocValuesType(DocValuesType.SORTED_SET);
document.add(new Field("city", "sssaas".getBytes(), fieldType));
```
将在`DefaultIndexingChain.processField()`里面开始检查字段是否设置了该参数:
```
    DocValuesType dvType = fieldType.docValuesType();
    // docValue必须不分词的字段才行
    if (dvType != DocValuesType.NONE) {
      if (fp == null) {
      该字段初始化PerField, 本次刷新到文当前, 相同域的PerField全局共享。
        fp = getOrAddField(fieldName, fieldType, false);
      }
      //创建docValue，主要是为了聚合使用, 面向列的存储，一列的元素放在一个field中。
      indexDocValue(fp, dvType, field);
    }
```
真正开始建立docValue的逻辑是在`indexDocValue()`中:
```
private void indexDocValue(PerField fp, DocValuesType dvType, IndexableField field) throws IOException {
    if (fp.fieldInfo.getDocValuesType() == DocValuesType.NONE) {
      // 把字段-docvalu全局存储起来
      fieldInfos.globalFieldNumbers.setDocValuesType(fp.fieldInfo.number, fp.fieldInfo.name, dvType);
    }
    fp.fieldInfo.setDocValuesType(dvType);

    int docID = docState.docID;

    switch(dvType) {
      case NUMERIC:
        if (fp.docValuesWriter == null) {
          fp.docValuesWriter = new NumericDocValuesWriter(fp.fieldInfo, bytesUsed);
        }
        ((NumericDocValuesWriter) fp.docValuesWriter).addValue(docID, field.numericValue().longValue());
        break;
      ......
      case SORTED_SET:
        if (fp.docValuesWriter == null) { // 所有文档所有域全局唯一
          fp.docValuesWriter = new SortedSetDocValuesWriter(fp.fieldInfo, bytesUsed);
        }
        ((SortedSetDocValuesWriter) fp.docValuesWriter).addValue(docID, field.binaryValue());
        break;
      default:
        throw new AssertionError("unrecognized DocValues.Type: " + dvType);
    }
  }
```
该函数主要做了如下几件事:
1.将该域关于docValue的设置存储在`FieldInfos.docValuesType`中, 同时验证传递进来的文档该域docValue设置是否发生了变化。
2.针对不同类型的docValue, 产生不同的docValuesWriter, 然后在内存中建立docValue的索引类型。
docValue类型`DocValuesType`分为6种:
```
  // 不开启docvalue时的状态，默认
  NONE,
  // 单个数值类型的docvalue主要包括（int，long，float，double）
  NUMERIC,
  // 二进制类型值对应不同的codes最大值可能超过32766字节
  BINARY,
  // 存储字符串+单值
  SORTED,
  // 存储数值类型的有序数组列表   数值或日期或枚举字段+多值
  SORTED_NUMERIC,
  // 可以存储多值域的docvalue值，但返回时，仅仅只能返回多值域的第一个docvalue
  // es针对不分词字段，默认是这个设置
  SORTED_SET,
```
我们这里就以域设置:`SORTED_SET`为例进行介绍, 真正来建立docValue结构使用的是:SortedSetDocValuesWriter, 在此将对该类进行简单介绍:
```
  // 真正存放value值的地方，每个value都是唯一的
  final BytesRefHash hash;
  // 存储每个文档每个域termId，相同Id的算两个。在未刷新到文档前，所有文档相同域的值都会放到这里
  private PackedLongValues.Builder pending; // stream of all termIDs,
  // pendingCounts统计的是每个文档中同名字段的个数，而pending统计的是每个同名字段的termId
  private PackedLongValues.Builder pendingCounts; // termIDs per doc
  private DocsWithFieldSet docsWithField;
  private final FieldInfo fieldInfo;
  // 正在处理每个doc的field
  private int currentDoc = -1;
  // 当前文档当前域对每个value分配的一个termId，根据这个termId去hash里面映射value。为数组的原因可能是域值是数组型
  private int currentValues[] = new int[8];
   // 当前文档当前域存放的第几个词，作为currentValues的下标。每写完一个文档的一个域，把数据转到pending中了，就清0。
  private int currentUpto;
```
我们进入addValue看下是如何存储的:
```
    // 该doc第一次写入, 说明上个文档的value已经写完了, 将上个文档给存储起来
    if (docID != currentDoc) {
      finishCurrentDoc();
      currentDoc = docID;
    }
    // 存储当前文档的value存放到currentValues中
    addOneValue(value);
    updateBytesUsed();
  }
```
该函数主要做了如下两件事:
1.检查文档ID是否与上一个一致, 若是的话, 则说明上一个文档该域已经写完了, 则调用`finishCurrentDoc()`。
2.将当前文档termId当前域写入currentValues
这两个变量将在写完一个文档时处理, 具体是在`finishCurrentDoc()`中:
```
private void finishCurrentDoc() { // 把上个doc的值给存储起来
    if (currentDoc == -1) { // 目前是提交commit时候内存中的文档数
      return;
    }
    Arrays.sort(currentValues, 0, currentUpto);
    int lastValue = -1;
    int count = 0; // 一个文档中，同名的域有几个
    for (int i = 0; i < currentUpto; i++) { // 最大只能为0
      int termID = currentValues[i];
      // if it's not a duplicate
      if (termID != lastValue) {
        pending.add(termID); // record the term id
        count++;
      }
      lastValue = termID;
    }
    // record the number of unique term ids for this doc
    pendingCounts.add(count);
    maxCount = Math.max(maxCount, count);
    currentUpto = 0; // 这里清0了
    docsWithField.add(currentDoc); // 存起来
  }
```
1.首先对currentValues按照字符大小进行排序(默认一个文档一个域只有一个value, 极少数一个域多个value)
2.轮训将该域每个value的termId放入pending中, 并将域的value个数放入pendingCounts(默认每个域的value个数都是1)。
3.将该文档ID放入docsWithField中。
存储结构如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_docvalue1.png" height="250" width="850"/>
其中docsWithField采取压缩方式存储所有的文档Id, 若所有文档都是连续的, 则cost=lastCost, 此时不占用任何内存。
若pending、pendingCounts中存放的termId个数超过1024个, 将按照bit位数进行压缩存储, 每次压缩就是values中的一个元素:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_docvalue2.png" height="250" width="850"/>

# 刷新到文件
为了更方面的了解dvd/dvm结构, 先上这两个文件的结构图:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_docvalue3.png" height="320" width="900"/>
DocValue刷新到文件的情况与fdt/fdx(详情参考<a href="https://kkewwei.github.io/elasticsearch_learning/2019/10/29/Lucenec%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-fdt-fdx%E6%9E%84%E5%BB%BA%E8%BF%87%E7%A8%8B/">Lucene底层架构-fdt/fdx构建过程</a>)一样:
1.lucene建立的索引结构占用内存超过阈值, 会在每次索引一个文档的时候检查。
2.用户主动通过indexWriter.flush()触发。
本文就从DefaultIndexingChain.writeDocValues()开始介绍:
```
  private void writeDocValues(SegmentWriteState state, Sorter.DocMap sortMap) throws IOException {
    DocValuesConsumer dvConsumer = null;
    try {
      // 是个hash链表结构, segment内唯一的域
      for (int i=0;i<fieldHash.length;i++) {
        PerField perField = fieldHash[i];
        while (perField != null) {
          if (perField.docValuesWriter != null) {
            if (dvConsumer == null) {
              // lazy init
              DocValuesFormat fmt = state.segmentInfo.getCodec().docValuesFormat();
              dvConsumer = fmt.fieldsConsumer(state); // PerFieldDocValuesFormat$FieldsWriter
            }
            // 把内存缓存的数据给刷到pending中
            if (finishedDocValues.contains(perField.fieldInfo.name) == false) {
              perField.docValuesWriter.finish(maxDoc); // 进来SortedSetDocValuesWriter.finish()
            }
            perField.docValuesWriter.flush(state, sortMap, dvConsumer); // 要进来看下，docvalue真正向磁盘写入
            perField.docValuesWriter = null; // 置空了
          }
          perField = perField.next;
        }
      }
      success = true;
    } finally {
      if (success) {
        // 向dvd/dvm写入footer并关闭文档
        IOUtils.close(dvConsumer);
      }
    }
  }
```
该函数将循环每个域,针对每个域分别处理:
1.调用docValuesWriter.finish()将该域内存中的数据刷新到pending中。
2.调用docValuesWriter.flush()将该域中的值刷新到dvd/dvm中。
3.将docValuesWriter置空, 下次将在该域的写入时重新创建该对象(在内存中创建docValue索引结构的时候:`DefaultIndexingChain.processField()`会去检查)
4.调用IOUtils.close(dvConsumer)向文档dvd/dvm写入footer并关闭文档。

我们看下docValuesWriter.flush()做了哪些事情:
```
 @Override
  public void flush(SegmentWriteState state, Sorter.DocMap sortMap, DocValuesConsumer dvConsumer) throws IOException {
    final int valueCount = hash.size();
    final PackedLongValues ords;
    final PackedLongValues ordCounts;
    final int[] sortedValues;
    final int[] ordMap;
    if (finalOrdCounts == null) {
      // 将pending中的剩余元素也压缩成一个values中的元素, 存放的是所有文档中该域的termdId
      ords = pending.build();
      // 存放每个文档该域value的个数, 默认都是1
      ordCounts = pendingCounts.build();
      // 按照每个termId对应的value进行排序, 返回的是对应termId数组
      sortedValues = hash.sort();
      // ordMap元素的对应的hash中的存储值就是termId的写入顺序, ordMap与sortedValues映射关系正好相反
      ordMap = new int[valueCount];
      for(int ord=0;ord<valueCount;ord++) {
         // // dvd只存放按照字符排序的字符。比如想知道第三个写入的数据是哪个,就需要ordMap和排序好的字符数组结合获取
        ordMap[sortedValues[ord]] = ord;
      }
    }
    ......
    dvConsumer.addSortedSetField(fieldInfo,
         new EmptyDocValuesProducer() {
            @Override
            public SortedSetDocValues getSortedSet(FieldInfo fieldInfoIn) {
                  final SortedSetDocValues buf =  // distinct(doc词)的个数,
                   new BufferedSortedSetDocValues(sortedValues, ordMap, hash, ords, ordCounts, maxCount, docsWithField.iterator());
                   if (sorted == null) {
                       return buf;
                   }
             }
         });
  }
```
该函数主要是获取如下变量:
+ ords: 按写入顺序存放每个文档的该域对应的termIds, 不过termIds已经被压缩了(按照占用byte空间压缩)。
+ ordCounts: 按写入顺序存放每个文档该域对应的值个数, 默认都是1。
+ sortedValues: 按照每个termId对应的value进行排序, 存放对应termId顺序
+ ordMap: 存放的是文档每个域的写入顺序, ordMap与sortedValues映射关系正好相反, 主要是为了获取term的写入顺序。
描述比较抽象, 我们举例说明:
写入顺序:        b   d    c    a
order:          0   1    2    3
sortedValues:   3   0    2    1
ordMap:         1   3    2    0
currentDoc:     1   3    2    0
实际存储的是排序好的数组:[a,b,c,d]。若此时我们想知道termId为3的term是啥? ordMap[3]=0, 结合[a,b,c,d]可知, 第三个写入的是a, 也就是termId=3的value是a。

接着调用PerFieldDocValuesFormat$FieldsWriter.addSortedSetField()继续刷新:
```
     public void addSortedSetField(FieldInfo field, DocValuesProducer valuesProducer) throws IOException {
      getInstance(field).addSortedSetField(field, valuesProducer);
    }
```
该函数做了两件事情:
1.调用getInstance(field), 对本次刷新创建_n_Lucene80_0.dvm、_n_Lucene80_0.dvd文件。其中n代表segment编号, 0代表不止Lucene80格式一种。这两个索引文件对该segment所有域共享的, 当写完第一个域docValue索引结构, 会append第二个域的docValue索引结构。
2.调用Lucene80DocValuesConsumer.addSortedSetField()向域dvd、dvm文件写入docValue结构。
```
public void addSortedSetField(FieldInfo field, DocValuesProducer valuesProducer) throws IOException {
    // 域number
    meta.writeInt(field.number);
    // 该域docValue类型
    meta.writeByte(Lucene80DocValuesFormat.SORTED_SET);
    SortedSetDocValues values = valuesProducer.getSortedSet(field); // BufferedSortedSetDocValues
    int numDocsWithField = 0;
    long numOrds = 0;
    for (int doc = values.nextDoc(); doc != DocIdSetIterator.NO_MORE_DOCS; doc = values.nextDoc()) {
      numDocsWithField++;  // 可能存在一个文档域名相同的域存在俩个
      for (long ord = values.nextOrd(); ord != SortedSetDocValues.NO_MORE_ORDS; ord = values.nextOrd()) {
        numOrds++; // 该域有几个termId
      }
    }
    // 每个文档域名相同的域只有一个
    if (numDocsWithField == numOrds) { //
      meta.writeByte((byte) 0); // multiValued (0 = singleValued)
      doAddSortedField(field, new EmptyDocValuesProducer() { //单值类型
        @Override
        public SortedDocValues getSorted(FieldInfo field) throws IOException {
          // 返回MaxValue
          return SortedSetSelector.wrap(valuesProducer.getSortedSet(field), SortedSetSelector.Type.MIN);
        }
      });
      return;
    }
    // 此时是多值类型的
    ......
  }
```
本函数主要是向dvm存储部分字段:
1.fieldNumer: 字段编号
2.该字段的docValueType
3.检查该域所有文档中相同域名的域有几个, 我们仅考虑最常见的情况: 每个文档相同域名的域只有一个。在进入Lucene80DocValuesConsumer.doAddSortedField做了哪些事情:
```
  private void doAddSortedField(FieldInfo field, DocValuesProducer valuesProducer) throws IOException {
     //SortedSetSelector$MinValue 域中多个value, 排序后只取最小的那个。
    SortedDocValues values = valuesProducer.getSorted(field);
    int numDocsWithField = 0;
    for (int doc = values.nextDoc(); doc != DocIdSetIterator.NO_MORE_DOCS; doc = values.nextDoc()) {
      // 同一个文档相同域名的域可能有两个, 这里统计该域所有域的个数
      numDocsWithField++;
    }
    ......
    meta.writeInt(numDocsWithField); // 文档个数
    if (values.getValueCount() <= 1) {
      ......
    } else {
      int numberOfBitsPerOrd = DirectWriter.unsignedBitsRequired(values.getValueCount() - 1); // 获取的是segment内唯一的域个数
      meta.writeByte((byte) numberOfBitsPerOrd); // bitsPerValue
      // 获取data文档目前写入的位置
      long start = data.getFilePointer();
      // ordsOffset   开始从文档这里写入
      meta.writeLong(start);
      DirectWriter writer = DirectWriter.getInstance(data, numDocsWithField, numberOfBitsPerOrd); // DirectWriter，就是简单地长度压缩
       //SortedSetSelector$MinValue
      values = valuesProducer.getSorted(field);
      for (int doc = values.nextDoc(); doc != DocIdSetIterator.NO_MORE_DOCS; doc = values.nextDoc()) { // 遍历这numDocsWithField
         // dvd中存储经过排序后的每个termId, 仅仅存入缓存
        writer.add(values.ordValue());
      }
       // 将termId编码写入dvd中
      writer.finish();
      meta.writeLong(data.getFilePointer() - start); // ordsLength
    }
    addTermsDict(DocValues.singleton(valuesProducer.getSorted(field)));
  }
```
该函数主要是向dvd中按照写入顺序写入所有的termId, 并在dvm中建立对应的索引结构, 同时存储了如下变量:
+ fieldNumer: 域的编号, segment内唯一
+ docValueType: docValue类型, 前面也已经介绍。
+ numDocsWithField: 该域所有文档中相同域名的域有多少个

然后调用Lucene80DocValuesConsumer.addTermsDict针对这些termId建立一级索引结构:
```
  private void addTermsDict(SortedSetDocValues values) throws IOException {
    final long size = values.getValueCount(); // 词的个数
    meta.writeVLong(size);
    long numBlocks = (size + Lucene80DocValuesFormat.TERMS_DICT_BLOCK_MASK) >>> Lucene80DocValuesFormat.TERMS_DICT_BLOCK_SHIFT; // 一个block为16
    long ord = 0;
    long start = data.getFilePointer();
    int maxLength = 0;
    TermsEnum iterator = values.termsEnum(); // SortedDocValuesTermsEnum
    // 遍历按照字母排序好的所有distinct(term)词
    for (BytesRef term = iterator.next(); term != null; term = iterator.next()) {
      // 每16个词, 记录一次索引
      if ((ord & Lucene80DocValuesFormat.TERMS_DICT_BLOCK_MASK) == 0) {
        writer.add(data.getFilePointer() - start);
        // 每16个词, 记录一次整个词
        data.writeVInt(term.length);
        data.writeBytes(term.bytes, term.offset, term.length);
      } else {
        final int prefixLength = StringHelper.bytesDifference(previous.get(), term);
        final int suffixLength = term.length - prefixLength;
        // 相同前缀用低4位，不同的后缀用高4位。在dvd中记录每个词的与前个词比较的相同个数,不同后缀长度
        data.writeByte((byte) (Math.min(prefixLength, 15) | (Math.min(15, suffixLength - 1) << 4))); // 高4位和低4位
       // 记录不同的后缀长度
        data.writeBytes(term.bytes, term.offset + prefixLength, term.length - prefixLength);
      }
      maxLength = Math.max(maxLength, term.length);
      previous.copyBytes(term);
      ++ord;
    }
    writer.finish(); // 将每隔15个词在dvm中的位置给记录下来
    ......
    // 第2层，记录 term 字典的索引，values 是按照值 hash 排过序的，这里每 1024 条抽取一个作为索引，加速查询
    // Now write the reverse terms index
    writeTermsIndex(values);
  }
```
该函数主要有三个作用:
1.将所有的term按照字符串排序的顺序写入dvd中。写入term时采用压缩方式, 通过和前一个写入的词做比较, 获取相同的前缀长度、不同的后缀长度并存储, 再存储词的后缀内容到dvd。
2.每隔16个词作为一级索引的元素,写入dvd中, 将此时dvd中的写入pointer写入dvm中, 以便快速定位。
3.调用writeTermsIndex, 对term建立第二级索引结构。
同时向dvm中存放如下变量:
+ TermDictBlockShift: 提示一级索引词的间隔为16。
+ termCounts: 词的个数。

我们接着看下Lucene80DocValuesConsumer.writeTermsIndex中二级索引结构是如何创建的:
```
  // TermsDict是16个term一个索引，而 TermsIndex是1024个词一个索引结构
  private void writeTermsIndex(SortedSetDocValues values) throws IOException {
    // segment范围内所有文档相同域distinct(term)的个数
    final long size = values.getValueCount();
    long start = data.getFilePointer();
    long numBlocks = 1L + ((size + Lucene80DocValuesFormat.TERMS_DICT_REVERSE_INDEX_MASK) >>> Lucene80DocValuesFormat.TERMS_DICT_REVERSE_INDEX_SHIFT);
    ByteBuffersDataOutput addressBuffer = new ByteBuffersDataOutput();
    DirectMonotonicWriter writer;
    try (ByteBuffersIndexOutput addressOutput = new ByteBuffersIndexOutput(addressBuffer, "temp", "temp")) {
      writer = DirectMonotonicWriter.getInstance(meta, addressOutput, numBlocks, DIRECT_MONOTONIC_BLOCK_SHIFT); // 也是使用这玩意写入数据
      TermsEnum iterator = values.termsEnum();// SortedDocValuesTermsEnum
      BytesRefBuilder previous = new BytesRefBuilder();
      long offset = 0;
      long ord = 0;
      // 遍历按照字母排序好的所有distinct(term)词
      for (BytesRef term = iterator.next(); term != null; term = iterator.next()) {
        //第1024*(x+1)个词建立索引
        if ((ord & Lucene80DocValuesFormat.TERMS_DICT_REVERSE_INDEX_MASK) == 0) {
        // 存储的是第二级别相同长度
          writer.add(offset);
          final int sortKeyLength;
          if (ord == 0) {
            // no previous term: no bytes to write
            sortKeyLength = 0;
          } else {
            sortKeyLength = StringHelper.sortKeyLength(previous.get(), term); // 相同的前缀
          }
          offset += sortKeyLength;
          data.writeBytes(term.bytes, term.offset, sortKeyLength);
        } else if ((ord & Lucene80DocValuesFormat.TERMS_DICT_REVERSE_INDEX_MASK) == Lucene80DocValuesFormat.TERMS_DICT_REVERSE_INDEX_MASK) { // 1024*x + 1023
          // 每次找到第1024*x+1023个词，主要是为了获取该词，为第1024*(x+1)个词找相同的前缀。
          previous.copyBytes(term);
        }
        ++ord;
      }
      writer.add(offset);
      writer.finish();
      meta.writeLong(start);
      meta.writeLong(data.getFilePointer() - start);
      start = data.getFilePointer();
      addressBuffer.copyTo(data);
      meta.writeLong(start); // 往meta中写入
      meta.writeLong(data.getFilePointer() - start);
    }
  }
```
该函数主要是为了构建二级索引结构, 遍历排序好的词典:
1.每隔1024*x+1023个词, 记录当前termId
2.每1024*x+1023+1个词, 将该词与前面一个词进行比较, 记录prefix长度并累加到offset, 存放在dvm中, 记录suffix内容到dvd中。
同时存放了如下变量: TERMS_DICT_REVERSE_INDEX_SHIFT, 提示二级索引间隔为1024个term。

# 总结
docValue作为列式存储, 不对域值分词, 所以整个域就会当成一个单独的term存储。在dvd/dvm中, 每次将一个域的所有term存储完后, 再接着存储第二个域的所有词。dvd中主要存储termId、termValue。 dvm中存储term的一级、二级索引元素, 以便于快速找到termDict中的每个词。一级索引是每16个词,存储一个词的value, 其中前15个词由于已经排序存储, 采用相同前缀压缩存储, 词典存储可以节省很多空间。对于二级索引, 每1024个词, 取样一个节点作为索引节点。

