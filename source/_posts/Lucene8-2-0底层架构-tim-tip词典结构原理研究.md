---
title: Lucene8.2.0底层架构-tim/tip词典结构原理研究
date: 2020-02-28 13:30:35
tags: Lucene、词典、tim、tip、doc、pos、fst、倒排索引
toc: true
categories: Lucene
---
Lucene中主要有两类倒排索引结构， 一种是词典结构， 涉及tim、tip、doc、pos。另外一种就是词典向量，涉及tvd,tvm（参考<a href="https://kkewwei.github.io/elasticsearch_learning/2020/03/02/Lucene8-2-0%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-tvd-tvm%E8%AF%8D%E5%85%B8%E5%90%91%E9%87%8F%E7%BB%93%E6%9E%84%E7%A0%94%E7%A9%B6/">Lucene8.2.0底层架构-tvd/tvm词典向量结构研究</a>）, 这者统计的信息大体相同，这两个类都继承自`TermsHashPerField`，都是对segment内相同域名所有文档共享这两个类。 
前者由`FreqProxTermsWriterPerField`构建, 后者由`TermVectorsConsumerPerField`构建。两者之间结构相似，先放一张图让大家对该对象有个大致的印象：
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_tim1.png" height="500" width="400"/>
可以看到，将两者连接起来的是bytePool（结构可参考<a href="https://kkewwei.github.io/elasticsearch_learning/2019/10/06/Lucene8-2-0%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-ByteBlockPool%E7%BB%93%E6%9E%84%E5%88%86%E6%9E%90/">Lucene8.2.0底层架构-ByteBlockPool结构分析</a>）,该对象就是存放的term内容，在词典和termVector构建之间共享以节约内存使用，不过前者产生的termId是segment该域级别唯一的，而后者是文档级别该域唯一。
Lucene查询中使用最多的就是词典结构, 根据term查询在哪些文档中存在, 也被称为倒排索引， 倒排索引结构如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_tim2.png" height=350" width="350"/>
由图可知，只要知道term，我们就可以很好地知道该term在每个document每个域的词频，位置，offset等信息，本文就以词典构建过程来进行深入研究。
# 词典在内存中构建
在对字典字段设置时, 可以进行如下设置:
```
   FieldType fieldType = new FieldType();
   // 对term建立倒排索引存储的数据
   fieldType.setIndexOptions(IndexOptions.DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS);
   fieldType.setTokenized(true);//分词
   fieldType.setStoreTermVectors(true);//分词
   fieldType.setOmitNorms(true);//分词
   fieldType.setStoreTermVectorOffsets(true);//分词
   fieldType.setStoreTermVectorPayloads(true);//分词
   fieldType.setStoreTermVectorPositions(true);//分词
```
1.setIndexOptions的参数含义:
```
  // 不建立词的倒排索引
  NONE,
  // 仅仅对词的建立索引结构
  DOCS,
  // 在termVector中仅存储词频
  DOCS_AND_FREQS,
  // 在termVector中存储词频和词position
  DOCS_AND_FREQS_AND_POSITIONS,
  // 在termVector中存储词频和词position和offset
  DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS,
```
这些参数将作用于词典中, doc仅仅是建立termId->doc的映射; freq还统每个域中每个单词的频率, postion统计了每个单词在每个域中的位置, offset统计了每个单词在每个域中的偏移量。
2. setStoreTermVector...()主要是是否产生termVector, 只有当setIndexOptions设置为非NONE, 设置这些参数才有效。
这两类参数含义很像, 前一个作用于词典的信息统计, 词典是全局型的; 而后一个设置作用于termVector, 统计的是单个域内的。

我们从`DefaultIndexingChain.processField()`开始讲解， 首先检查该字段设置:
```
    if (fieldType.indexOptions() != IndexOptions.NONE) {
      // 每个字段只会保存一个 PerField 对象。
      fp = getOrAddField(fieldName, fieldType, true);
      // 这个文档中这个域不是重复写入
      boolean first = fp.fieldGen != fieldGen;
      // 创建倒排索引
      fp.invert(field, first);
      // 域是第一次写入
      if (first) {
        // 这里才是真正存放，和fieldHash存放的是一个对象。真正统计的是当前文档所有的域。
        fields[fieldCount++] = fp;
        fp.fieldGen = fieldGen;
      }
    } else {
      // 若我们fieldType=NONE, 那么关于存储termVector设置都是不合理的。
      verifyUnIndexedFieldType(fieldName, fieldType);
    }
```
该函数主要做了两件事:
1.检查该字段是否已经写入过。
2.调用`fp.inver`对该字段进行分词。
```
    public void invert(IndexableField field, boolean first) throws IOException { // PerFieldl里面开始
      if (first) { // 第一次该字段被写入
        invertState.reset(); // 每次写入一个新的文档，这里都会被清空
      }
      IndexableFieldType fieldType = field.fieldType();

      final boolean analyzed = fieldType.tokenized() && docState.analyzer != null; // 是否分词
       // 对域的值进行了分词
      try (TokenStream stream = tokenStream = field.tokenStream(docState.analyzer, tokenStream)) {
        // 针对TermVectorsConsumerPerField里面termvector参数进行设置
        termsHashPerField.start(field, first);
        while (stream.incrementToken()) {
          // 每个词增量为1
          // 对position和offset进行统计。
          invertState.position += posIncr; // 每个词新增1
          invertState.lastPosition = invertState.position;
          // 词的起始位置
          int startOffset = invertState.offset + invertState.offsetAttribute.startOffset();
          // 这个词的末尾
          int endOffset = invertState.offset + invertState.offsetAttribute.endOffset();
          // 该文档该域上一个词的截止为止
          invertState.lastStartOffset = startOffset;
          // 真正对词进行
          termsHashPerField.add();
        }
        stream.end();
      }
      if (analyzed) { // 若分词的话，
        invertState.position += docState.analyzer.getPositionIncrementGap(fieldInfo.name);
        invertState.offset += docState.analyzer.getOffsetGap(fieldInfo.name);
      }
    }
  }
```
该函数主要做了如下事情:
1.置空invertState统计, 每个文档每个域将分别统计。
2.统计该词的相关信息, 放入invertState
3.调用`termsHashPerField.add()`开始对一个词读取相应索引结构。
4. 若分词的话, 需要对每个域值增加position和offset统计。对于域multiValue字段使用，用于设置多值数据间的间隔。
我们需要重点关注第三步:
```
  void add() throws IOException {
     // 获取当前term的termId, 此时获取的termId在semgment内该域都是唯一的
    int termID = bytesHash.add(termAtt.getBytesRef());  
    // 该term第一次写入
    if (termID >= 0) {
      //  intPool的当前buffer不够用就申请新的byte[]
      if (numPostingInt + intPool.intUpto > IntBlockPool.INT_BLOCK_SIZE) {
        intPool.nextBuffer();
      }
      //  intPool的当前buffer不够用就申请新的byte[]
      if (ByteBlockPool.BYTE_BLOCK_SIZE - bytePool.byteUpto < numPostingInt*ByteBlockPool.FIRST_LEVEL_SIZE) {
        bytePool.nextBuffer();
      }
      //此处 streamCount 为 2,表明在intPool中,一个词将占用2位,一个是指向该词的bytePool中docId&freq存储位置,一个指向该词的bytePool中position&offset存储位置。
      intUptos = intPool.buffer;
      // int当前buffer内的可分配位置
      intUptoStart = intPool.intUpto;  
      // 先在intPool中申请2个位置
      intPool.intUpto += streamCount;  
      // 第i个词在intintPool中（为了快速找到该词的int位置绝对起始起始位置
      postingsArray.intStarts[termID] = intUptoStart + intPool.intOffset; 
     //在 bytePool 中分配两个空间,一个放 freq 信息,一个放 prox 信息的。
      for(int i=0;i<streamCount;i++) {
        // 返回的是该slice的相对起始位置，存放docId&freq
        final int upto = bytePool.newSlice(ByteBlockPool.FIRST_LEVEL_SIZE);
         // 存放position&offset
        intUptos[intUptoStart+i] = upto + bytePool.byteOffset; 
      }
      // 把该termId的使用byteBlockPool起始位置给记录下来
      postingsArray.byteStarts[termID] = intUptos[intUptoStart]; 
      // 开始向两个slice中存放该词的docId&freq和position&offset。
      newTerm(termID);
    } else { 
      // 这说明该词已经出现过一次， 返回该词的termId， 使用小技巧，将返回termId编码为负值
      termID = (-termID)-1; 
      // 返回该词在在intPool中的起始位置
      int intStart = postingsArray.intStarts[termID];
      // intPool中当前使用的buffer的相对起始位置
      intUptos = intPool.buffers[intStart >> IntBlockPool.INT_BLOCK_SHIFT];
      intUptoStart = intStart & IntBlockPool.INT_BLOCK_MASK;
      // 向两个slice中追加该词的docId&freq和position&offset。
      addTerm(termID);
    }
    // 开始统计termVector需要的bytePool中docId&freq和bytePool中position&offset
    if (doNextCall) { 
      nextPerField.add(postingsArray.textStarts[termID]); 
    } 
  }
```
该函数主要针对当前termId做了如下事情：
1.通过`bytesHash.add`确定该词的termId, termId在Segment内全局唯一。
2.若该term是第一次写入该segment，那么在intPool中申请2个byte，在bytePool中申请两个slice。int的两个byte作为指针，指向申请的两个slice。然后调用`addTerm`， 统计该termId的docId&freq和position&offset， 分别放入这两个slice中。
3.若该term的termId已经存在， 那么调用`addTerm`统计两类之别分别放入两个slice。
4.调用`nextPerField.add`构建termVector的索引结构， 将在<a href="https://kkewwei.github.io/elasticsearch_learning/2020/03/02/Lucene8-2-0%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-tvd-tvm%E8%AF%8D%E5%85%B8%E5%90%91%E9%87%8F%E7%BB%93%E6%9E%84%E7%A0%94%E7%A9%B6/">Lucene8.2.0底层架构-tvd/tvm词典向量结构研究</a>中重点介绍。

我们首先看下`bytesHash.add`是如何做到存储termValue的。
```
    final int length = bytes.length;
    // 使用探针法， 直到找到当前要存储的bytes所在的有效槽位
    final int hashPos = findHash(bytes);
    int e = ids[hashPos]; 
    //如果为-1，则是新的term
    if (e == -1) {
      final byte[] buffer = pool.buffer;
      final int bufferUpto = pool.byteUpto;// 获取内存池的起始可用位置
      count++;
      // 记录对应termId在bytePool中的内容。freqProxPostingsArray.textStarts和bytesStart是同一个对象
      bytesStart[e] = bufferUpto + pool.byteOffset;
      // 在pool首先存储len(bytes),在存储值
      if (length < 128) {
        buffer[bufferUpto] = (byte) length;
        pool.byteUpto += length + 1;
        assert length >= 0: "Length must be positive: " + length;
        System.arraycopy(bytes.bytes, bytes.offset, buffer, bufferUpto + 1,
            length);
      } else {
        buffer[bufferUpto] = (byte) (0x80 | (length & 0x7f));
        buffer[bufferUpto + 1] = (byte) ((length >> 7) & 0xff);
        pool.byteUpto += length + 2;
        System.arraycopy(bytes.bytes, bytes.offset, buffer, bufferUpto + 2,
            length);
      }
      ids[hashPos] = e; 
      return e;
    }
    // 该term已经存在，直接返回termId
    return -(e + 1);
  }
```
本函数主要是将term存储到pool中， 存储时通过hash快速term存放的位置。

我们再看下`newTerm`怎么统计的新产生的term：
```
  void newTerm(final int termID) {
    final FreqProxPostingsArray postings = freqProxPostingsArray; //

    postings.lastDocIDs[termID] = docState.docID;
    //DOCS_AND_FREQS_AND_POSITIONS_AND_OFFSETS递进
    if (!hasFreq) { 
      ......
    } else { 
      // 统计词频
      postings.lastDocCodes[termID] = docState.docID << 1;
      postings.termFreqs[termID] = getTermFreq();
      // 存储offset, position到第第二个slice中
      if (hasProx) { 
        // 向stream1中存储了proxCode
        writeProx(termID, fieldState.position); 
        if (hasOffsets) {
          // 向stream1中存储了offserCode,
          writeOffsets(termID, fieldState.offset);
        }
      }
      fieldState.maxTermFrequency = Math.max(postings.termFreqs[termID], fieldState.maxTermFrequency);
    }
    fieldState.uniqueTermCount++;
  }
```
该函数主要统计了如下信息：
1.统计position和offset。
2.这里并没有立即统计词频， 词频必须等该文档该域写完后才能统计。

我们再看下如何向一个已经存在的term统计freq、position及offset：
```
  void addTerm(final int termID) {
    final FreqProxPostingsArray postings = freqProxPostingsArray;
    if (!hasFreq) {//不需要词频
      ......
    } else if (docState.docID != postings.lastDocIDs[termID]) { 
      // 上一个文档的该域所有term已经处理完了
      if (1 == postings.termFreqs[termID]) {  // 词频为1
        writeVInt(0, postings.lastDocCodes[termID]|1); 
      } else {
        writeVInt(0, postings.lastDocCodes[termID]);
        writeVInt(0, postings.termFreqs[termID]);
      }
      //初始化当前文档的词频
      postings.termFreqs[termID] = getTermFreq(); 
      fieldState.maxTermFrequency = Math.max(postings.termFreqs[termID], fieldState.maxTermFrequency);
      //  这次出现文档-上次出现文档
      postings.lastDocCodes[termID] = (docState.docID - postings.lastDocIDs[termID]) << 1;
      postings.lastDocIDs[termID] = docState.docID;
      if (hasProx) {
        // 保存termId
        writeProx(termID, fieldState.position);
        if (hasOffsets) {
          // 保存offset
          postings.lastOffsets[termID] = 0;
          writeOffsets(termID, fieldState.offset);
        }
      }
      fieldState.uniqueTermCount++;
    } else { 
      // 当前文档当前域的term还没有处理完
      postings.termFreqs[termID] = Math.addExact(postings.termFreqs[termID], getTermFreq()); 
      // 增加词频
      fieldState.maxTermFrequency = Math.max(fieldState.maxTermFrequency, postings.termFreqs[termID]);
      if (hasProx) { 
        // 继续统计post
        writeProx(termID, fieldState.position-postings.lastPositions[termID]);
        if (hasOffsets) {
          // 统计offser
          writeOffsets(termID, fieldState.offset);
        }
      }
    }
  }
```
向bytePool中已经存在的termId：
1.这里会检查该词最后一次出现的DocId是否和本文档一致 若不一直，则说明上一个文档该term已经处理完了，需要将文档DocId，词频保存带第一个slice中。可能这里有疑问，为啥该词若是第一次出现时不需要检文档Id是否发生了变化，因为该词第一次出现时，上一个文档更不存在。
2.将词position&offset保存到第二个slice中。

最终建立的逻辑结构如下所示：
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_tim3.png" height="400" width="400"/>
freqProxPostingsArray里面数组下标就是termId, 如何展示的是termId=1的term 倒排索引存放情况。

# flush到文件中
flush到文件中指的是形成一个segment，触发条件有两点（同<a href="https://kkewwei.github.io/elasticsearch_learning/2019/10/29/Lucenec%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-fdt-fdx%E6%9E%84%E5%BB%BA%E8%BF%87%E7%A8%8B/#%E5%88%B7%E5%88%B0fdx%E6%96%87%E4%BB%B6">fdx</a>，<a href="https://kkewwei.github.io/elasticsearch_learning/2019/11/15/Lucene%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-dvm-dvm%E6%9E%84%E5%BB%BA%E8%BF%87%E7%A8%8B/#%E5%88%B7%E6%96%B0%E5%88%B0%E6%96%87%E4%BB%B6">dvm</a>一样）：
1.lucene建立的索引结构占用内存或者缓存文档书超过阈值, 会在每次索引一个文档的时候检查。
2.用户主动通过indexWriter.flush()触发。

本节就从`BlockTreeTermsWriter.write`开始介绍：
```
  public void write(Fields fields, NormsProducer norms) throws IOException {
    String lastField = null;
    // 遍历该segment内的每个域
    for(String field : fields) {
      lastField = field;
      Terms terms = fields.terms(field); 
      if (terms == null) {
        continue;
      }
       // 遍历FreqProxTermsWriterPerField里面每个termId使用的，读取term的顺序前按照字符串排好序了
      TermsEnum termsEnum = terms.iterator();
      // 一个域单独产生一个
      TermsWriter termsWriter = new TermsWriter(fieldInfos.fieldInfo(field)); 
      while (true) {
        // 这里会从FreqProxPostingsArray.textStarts循环遍历每一个termId。
        BytesRef term = termsEnum.next(); 
        if (term == null) {
          break;
        }
        // 将该term加入词典及建立字典索引结构。
        termsWriter.write(term, termsEnum, norms); 
      }
      termsWriter.finish();// 完成field 的构建。每个单词一个finish
    }
  }
```
该函数主要是遍历该segment中每个域里所有的词， 然后调用`termsWriter.write`建立词典结构：
```
    public void write(BytesRef text, TermsEnum termsEnum, NormsProducer norms) throws IOException {
      // 将该term的倒排索引读取出来并建立索引结构
      BlockTermState state = postingsWriter.writeTerm(text, termsEnum, docsSeen, norms); // 针对的是一个词
      if (state != null) {
        // 将当前词加入词典中
        pushTerm(text); 
        PendingTerm term = new PendingTerm(text, state); 
        pending.add(term);//当前term加入待索引列表
        sumDocFreq += state.docFreq; // 该词在多少文档中出现过
        sumTotalTermFreq += state.totalTermFreq; //该词总的出现频次
        numTerms++;  //
        if (firstPendingTerm == null) {
          firstPendingTerm = term; // 写入的第一个词
        }
        lastPendingTerm = term;
      }
    }
```
该函数主要做了如下事情：
1.调用`postingsWriter.writeTerm()`，将该term的倒排索引结构放入doc文件中。
2.调用`pushTerm()`将该词加入词典中。

## 单个词的倒排索引结构落盘
我们看下term的倒排索引结构如何放入doc文件中的，进入`PushPostingsWriterBase.writeTerm()`：
```
  @Override
  public final BlockTermState writeTerm(BytesRef term, TermsEnum termsEnum, FixedBitSet docsSeen, NormsProducer norms) throws IOException {
     // 读取doc,pos等文件的起始位置
    startTerm(normValues);
    // 从bytePool中读取了该词的stream0,stream
    postingsEnum = termsEnum.postings(postingsEnum, enumFlags); 
    int docFreq = 0;
    long totalTermFreq = 0; //该文档总的词频
    while (true) { 
       // 依次读取这个docID的freq
      int docID = postingsEnum.nextDoc();
      if (docID == PostingsEnum.NO_MORE_DOCS) {
        break;
      }
      // 该词在多少个文档中出现过
      docFreq++; 
      docsSeen.set(docID); // 在这个文档中可见
      int freq;
      if (writeFreqs) {
        // 该文档该词的词频。已经在postingsEnum.nextDoc()中给解析出来了
        freq = postingsEnum.freq(); 
        totalTermFreq += freq;
      } else {
        freq = -1;
      } //检查读取该词的的docId，freq是否达到一个block。若达到了，则将DocId、freq压缩到doc文件中，并构建调表结构。
      startDoc(docID, freq); 
      if (writePositions) {
        for(int i=0;i<freq;i++) { // 对每个词的每个position都读取出来
          int pos = postingsEnum.nextPosition(); // 第几个position，将startOffset和endOffset都解析出来了
          BytesRef payload = writePayloads ? postingsEnum.getPayload() : null;
          int startOffset;
          int endOffset;
          if (writeOffsets) {
            startOffset = postingsEnum.startOffset();
            endOffset = postingsEnum.endOffset();
          } else {
            startOffset = -1;
            endOffset = -1;
          } // 检查读取的该词position是否达到block（128个），若达到就存放到pos和pay文件中
          addPosition(pos, payload, startOffset, endOffset); // 保存增量的freq。
        }
      }
      // 检查该词存储的文档个数是否达到block，写到后，更新本地保存的FilePointer，清空缓存的文档数。
      finishDoc(); 
    }
    if (docFreq == 0) { // 总文档数
      return null;
    } else { 
      // 给这个block赋予一些元数据
      BlockTermState state = newTermState(); 
      // 存在多少个文档中
      state.docFreq = docFreq; 
      // 该词总共出现的次数
      state.totalTermFreq = writeFreqs ? totalTermFreq : -1; 
      // 当前term读取完了。将跳表信息填入doc
      finishTerm(state); 
      return state;
    }
  }
```
该函数主要做了如下事情：
1.遍历该term所有的docId,position、offset，通过`startDoc()`对每128个词建立跳表；通过`addPosition()`将每128个position构建一个block写入pos。
2.遍历完了该term下所有词后， 调用`finishTerm`首先将缓存doc-freq写入doc文件， 然后将该词的跳表结构入doc文件，见缓存的position-offset写入pos文件。
我们需要关注些如何通过`Lucene50PostingsWriter.startDoc()`构建跳表结构的：
```
 @Override
  public void startDoc(int docID, int termDocFreq) throws IOException {
    // 已经将一批docId,freq已经被压缩到了doc文件中，作为一个block。再对这个block建立跳表点。
    if (lastBlockDocID != -1 && docBufferUpto == 0) { 
      skipWriter.bufferSkip(lastBlockDocID, competitiveFreqNormAccumulator, docCount, 
          lastBlockPosFP, lastBlockPayFP, lastBlockPosBufferUpto, lastBlockPayloadByteUpto);
    }
    final int docDelta = docID - lastDocID;
    docDeltaBuffer[docBufferUpto] = docDelta;
    if (writeFreqs) {
      freqBuffer[docBufferUpto] = termDocFreq; // 存储词频
    }
    docBufferUpto++;
    docCount++;
    //每128个term->freq作为一个Block存储到doc中
    if (docBufferUpto == BLOCK_SIZE) { 
      forUtil.writeBlock(docDeltaBuffer, encoded, docOut); 
      if (writeFreqs) {
        // 把128个缓存的文档freq缓存到doc中
        forUtil.writeBlock(freqBuffer, encoded, docOut); 
      }
    }
  }
```
该函数主要做了如下事情：
1.将该term的doc,freq保存到docDeltaBuffer、freqBuffer中
2.每128个docId-freq，通过`ForUtil.writeBlock`建立一个block, 放入doc文档。
3.针对每个block，调用`Lucene50SkipWriter.bufferSkip`建立跳表结构。

我们主要关注下，如何在`MultiLevelSkipListWriter.bufferSkip()`针对单个term每个block建立跳表结构的：
```angular2html
  public void bufferSkip(int df) throws IOException {
    int numLevels = 1; 
    // 统计可以构建几级跳表
    // 第一级别跳表针对每128个doc-freq建立一个节点
    df /= skipInterval;  
    // 之后第n级别级别都是128*8^(n-1)个文档建立一个跳表节点
    while ((df % skipMultiplier) == 0 && numLevels < numberOfSkipLevels) {
      numLevels++;
      df /= skipMultiplier;
    }
    long childPointer = 0;
    // level值从小到大，在每层 level的字节数组末尾写入skip point
    for (int level = 0; level < numLevels; level++) {
      // 将 skip point写入 skipBuffer[level]中
      writeSkipData(level, skipBuffer[level]);
      long newChildPointer = skipBuffer[level].getFilePointer();
      if (level != 0) { // 缓存第几级别
        // 后一个level记录前一个level使用的缓存大小
        skipBuffer[level].writeVLong(childPointer); 
      }
      childPointer = newChildPointer;
    }
  }
```
该函数主要是针对该term存储的每一个block（文档和freq）建立跳表结构，跳表第一级为每128（skipInterval）个block ，此后第n级跳表为每128*8^（n-1）个block。计算出了多少级（numLevels）， 然后针对这个block在每个级别上都建立跳表结构。我们具体看下`Lucene50SkipWriter.writeSkipData()`记录了每个跳表的具体数据：
```
  protected void writeSkipData(int level, IndexOutput skipBuffer) throws IOException {
    // 计算当前level层，docId的delta值
    int delta = curDoc - lastSkipDoc[level];
    skipBuffer.writeVInt(delta);
     // 该层级上一次的文档Id
    lastSkipDoc[level] = curDoc;
    // 写入 doc文件的偏移量的 delta值, 记录下该跳跃点在doc使用的大小
    skipBuffer.writeVLong(curDocPointer - lastSkipDocPointer[level]); 
    lastSkipDocPointer[level] = curDocPointer; // 记录当前doc占用的其实内存

    if (fieldHasPositions) { // 向skipBuffer写入pos,doc,pay偏移量

      skipBuffer.writeVLong(curPosPointer - lastSkipPosPointer[level]); // 记录下该跳跃点在pos使用的大小
      lastSkipPosPointer[level] = curPosPointer;
      skipBuffer.writeVInt(curPosBufferUpto); // 记录下当前缓存的文档数

      if (fieldHasPayloads) {
        skipBuffer.writeVInt(curPayloadByteUpto);
      }

      if (fieldHasOffsets || fieldHasPayloads) {
        skipBuffer.writeVLong(curPayPointer - lastSkipPayPointer[level]); // 记录下该跳跃点在pay使用的大小
        lastSkipPayPointer[level] = curPayPointer;
      }
    }

    CompetitiveImpactAccumulator competitiveFreqNorms = curCompetitiveFreqNorms[level];
    assert competitiveFreqNorms.getCompetitiveFreqNormPairs().size() > 0;
    if (level + 1 < numberOfSkipLevels) {
      curCompetitiveFreqNorms[level + 1].addAll(competitiveFreqNorms);
    }
    writeImpacts(competitiveFreqNorms, freqNormOut); // 向freqNormOut中写入影响因子
    skipBuffer.writeVInt(Math.toIntExact(freqNormOut.getFilePointer()));
    freqNormOut.writeTo(skipBuffer); // 把freqNormOut数据向skipBuffer中写入
    freqNormOut.reset();
    competitiveFreqNorms.clear();
  }
```
每个跳表中主要存放了如下对象：


## 构建词典结构
