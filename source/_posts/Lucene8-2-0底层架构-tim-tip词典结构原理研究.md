---
title: Lucene8.2.0底层架构-tim/tip词典结构原理研究
date: 2020-02-28 13:30:35
tags: Lucene、词典、tim、tip、doc、pos、fst、倒排索引
toc: true
categories: Lucene
---
Lucene中主要有两类倒排索引结构， 一种是词典结构， 涉及tim、tip、doc、pos。另外一种就是词典向量,统计的是单个词在文档内的，涉及tvd,tvm（参考<a href="https://kkewwei.github.io/elasticsearch_learning/2020/03/02/Lucene8-2-0%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-tvd-tvm%E8%AF%8D%E5%85%B8%E5%90%91%E9%87%8F%E7%BB%93%E6%9E%84%E7%A0%94%E7%A9%B6/">Lucene8.2.0底层架构-tvd/tvm词典向量结构研究</a>）, 这者统计的信息大体相同，这两个类都继承自`TermsHashPerField`，都是对segment内相同域名所有文档共享这两个类。 
前者由`FreqProxTermsWriterPerField`构建, 后者由`TermVectorsConsumerPerField`构建。两者之间结构相似，前者统计单个词在segment内所有文档当前域的词频等，后者统计单个词在当前文档当前域的词频等。先放一张图让大家对这两个对象有个大致的了解：
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_tim1.png" height="500" width="400"/>
可以看到，将两者连接起来的是bytePool（结构可参考<a href="https://kkewwei.github.io/elasticsearch_learning/2019/10/06/Lucene8-2-0%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-ByteBlockPool%E7%BB%93%E6%9E%84%E5%88%86%E6%9E%90/">Lucene8.2.0底层架构-ByteBlockPool结构分析</a>）,该对象就是存放的term内容，在词典和TermVector构建之间共享以节约内存使用，不过前者产生的termId在当前segment相同域内唯一的，而后者仅仅在文档该域内唯一。
Lucene查询中使用最多的就是词典结构, 根据term查询在哪些文档中存在, 也被称为倒排索引， 倒排索引结构如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_tim2.png" height="350" width="400"/>
由图可知，只要知道termId，我们就可以很好地知道该term在每个document每个域的词频，位置，offset等信息，本文就以词典构建过程来进行深入研究。
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
2.setStoreTermVector...()主要作用在TermVector, 只有当setIndexOptions设置为非NONE, 设置这些参数才有效。
这两类参数含义很像, 前一个作用于词典的信息统计, 词典是全局型的; 而后一个设置作用于termVector, 统计的是单个文档单个域内的。

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
4.若分词的话, 需要对每个域值增加position和offset统计。对于域multiValue字段使用，用于设置多值数据间的间隔。
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
该函数对当前termId, 做了如下事情：
1.通过`bytesHash.add`确定该词的termId, termId在Segment内全局唯一。
2.若该term是第一次写入该segment，那么在intPool中申请2个byte，在bytePool中申请两个slice。int的两个byte作为指针，指向申请的两个slice。然后调用`addTerm`， 统计该termId的docId&freq和position&offset， 分别放入这两个slice中，结构将在后图展示。
3.若该term的termId已经存在， 那么调用`newTerm`统计信息，分别放入以上两个slice中。
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

我们再看下`newTerm`怎么统计新产生的term：
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
该函数主要统计了该词如下信息：
1.统计position和offset。
2.这里并没有立即统计该词的词频， 词频必须等该文档该域写完后才能统计。（只有当相同词的docId发生变化了才开始统计）

我们再看下如何对一个已经存在的term统计freq、position及offset：
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
此时bytePool中已经存在termId：
1.这里会检查该词最后一次出现的DocId是否和本文档一致 若不一直，则说明上一个文档该term已经处理完了，需要将文档DocId，词频保存带第一个slice中。可能这里有疑问，为啥该词若是第一次出现时不需要检文档Id是否发生了变化，因为该词第一次出现时，上一个文档更不存在。
2.将词position&offset保存到第二个slice中。

最终建立的内存结构如下所示：
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_tim3.png" height="400" width="400"/>
freqProxPostingsArray里面数组下标就是termId, 上图展示的是termId=1的term 倒排索引存放情况。

# flush到文件中
flush到文件中指的是形成一个segment，触发条件有两个（同<a href="https://kkewwei.github.io/elasticsearch_learning/2019/10/29/Lucenec%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-fdt-fdx%E6%9E%84%E5%BB%BA%E8%BF%87%E7%A8%8B/#%E5%88%B7%E5%88%B0fdx%E6%96%87%E4%BB%B6">fdx</a>，<a href="https://kkewwei.github.io/elasticsearch_learning/2019/11/15/Lucene%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-dvm-dvm%E6%9E%84%E5%BB%BA%E8%BF%87%E7%A8%8B/#%E5%88%B7%E6%96%B0%E5%88%B0%E6%96%87%E4%BB%B6">dvm</a>一样）：
1.lucene建立的索引结构占用内存或者缓存文档书超过阈值。该check会在每次索引完一个文档后。
2.用户主动调用indexWriter.flush()触发。

两种情况最终都会跑到`BlockTreeTermsWriter.write`：
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
      // 完成field 的构建。每个单词一个finish
      termsWriter.finish();
    }
  }
```
该函数首先遍历该segment中每个域里所有的词，将所有词的倒排结构存放在doc文件，然后调用`termsWriter.write`建立词典结构，最后调用`termsWriter.finish()`将词典索引结构FST放入tip文件中。
```
    public void write(BytesRef text, TermsEnum termsEnum, NormsProducer norms) throws IOException {
      // 将该term的倒排索引读取出来并建立索引结构
      BlockTermState state = postingsWriter.writeTerm(text, termsEnum, docsSeen, norms); // 针对的是一个词
      if (state != null) {
        // 将当前词加入词典中
        pushTerm(text); 
        PendingTerm term = new PendingTerm(text, state);
        //当前term加入待索引列表
        pending.add(term);
        // 该词在多少文档中出现过
        sumDocFreq += state.docFreq;
        //该词总的出现频次
        sumTotalTermFreq += state.totalTermFreq; 
        numTerms++;  //
        if (firstPendingTerm == null) {
          // 写入的第一个词
          firstPendingTerm = term;
        }
        lastPendingTerm = term;
      }
    }
```
该函数主要做了如下事情：
1.调用`postingsWriter.writeTerm()`，将该term的倒排索引结构放入doc文件中。
2.调用`pushTerm()`将该词加入词典中。

## 单个词的倒排索引结构落盘
我们看下当前域的term的倒排索引结构如何放入doc文件中的，进入`PushPostingsWriterBase.writeTerm()`：
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
      } //检查读取该词的的docId，freq是否达到一个block。若达到了，则将DocId、freq压缩到doc文件中，并构建跳表结构。
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
1.遍历当前域的该term下所有的docId,position、offset，通过`startDoc()`对每128个词建立跳表放入内存；通过`addPosition()`将每128个position构建一个block写入pos。
2.遍历完了该term下所有词后， 调用`finishTerm`首先将缓存doc-freq写入doc文件， 将缓存的position-offset写入pos文件， 并然后将该词的包含doc&pos&pay的跳表结构入doc文件。
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
```
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
该函数主要是针对该term存储的每一个block（文档和freq）建立跳表结构，跳表第一级为每128（skipInterval）个block ，此后第n级跳表为每128*8^（n-1）个block。计算出了跳表需要多少级（numLevels）， 然后针对这个block在每个级别上都建立跳表结构。我们具体看下`Lucene50SkipWriter.writeSkipData()`如何构建跳表的：
```
  protected void writeSkipData(int level, IndexOutput skipBuffer) throws IOException {
    // 计算当前level层，docId的delta值
    int delta = curDoc - lastSkipDoc[level];
    skipBuffer.writeVInt(delta);
     // 该层级上一次的文档Id
    lastSkipDoc[level] = curDoc;
    // 写入 doc文件的偏移量的delta值, 记录下该跳跃点在doc使用的大小
    skipBuffer.writeVLong(curDocPointer - lastSkipDocPointer[level]);
    // 记录当前doc占用的其实内存
    lastSkipDocPointer[level] = curDocPointer;
    // 向skipBuffer写入pos,doc,pay偏移量
    if (fieldHasPositions) { 
      // 记录下该跳跃点在pos使用的大小
      skipBuffer.writeVLong(curPosPointer - lastSkipPosPointer[level]); 
      lastSkipPosPointer[level] = curPosPointer;
      // 记录下当前缓存的文档数
      skipBuffer.writeVInt(curPosBufferUpto);
      if (fieldHasPayloads) {
        skipBuffer.writeVInt(curPayloadByteUpto);
      }
      if (fieldHasOffsets || fieldHasPayloads) {
        // 记录下该跳跃点在pay使用的大小
        skipBuffer.writeVLong(curPayPointer - lastSkipPayPointer[level]); 
        lastSkipPayPointer[level] = curPayPointer;
      }
    }
    // 向freqNormOut中写入加权系数
    writeImpacts(competitiveFreqNorms, freqNormOut);
    skipBuffer.writeVInt(Math.toIntExact(freqNormOut.getFilePointer()));
    // 把freqNormOut数据向skipBuffer中写入
    freqNormOut.writeTo(skipBuffer); 
    freqNormOut.reset();
    competitiveFreqNorms.clear();
  }
```
每个跳表元素中主要存放了如下对象：
1.当前级相邻的两个跳表对应的文档差值、文档存储指针差值
2.pos文件跳表对应的差值.
3.offset/payload文档对应的差值。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_tim4.png" height="100" width="350"/>

doc存储的docId-freq及跳表结构如下所示：
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_tim5.png" height="300" width="800"/>


## 每个域构建词典结构
在对单个term建立完跳表结构后，就开始将该term加入词典中，词典作为倒排索引的核心结构，通过词典，我们可以快速在doc、pos文件中找到每个词的文档ID、词频。构建词典采取了一些做法: 相似性比较高的term可以构成一个子term（专业名称是子FST结构），子term再重新再和别的term组成一个更大的FST。本节看下如何构建词典结构的，每个词都是通过`pushTerm(text)`加入词典结构的：
```
    private void pushTerm(BytesRef text) throws IOException {
      int limit = Math.min(lastTerm.length(), text.length); // 本term和上一个term最小值
      int pos = 0;
       // 计算当前term与前一个term的公共前缀长度+1, 不相同的那个
      while (pos < limit && lastTerm.byteAt(pos) == text.bytes[text.offset+pos]) {
        pos++;
      }
      // 尽量找一批term，从后向前是为了更多可能取相似性前缀
      for(int i=lastTerm.length()-1;i>=pos;i--) { // last
        // // 计算与栈顶的Entry的公共前缀为 i 的Entry的数量
        // How many items on top of the stack share the current suffix
        // we are closing:
         // 当前存量与多少后缀是不同的
        int prefixTopSize = pending.size() - prefixStarts[i];
         // pending词的个数最少25个。
        if (prefixTopSize >= minItemsInBlock) {
          // 将最后prefixTopSize产生一个fst，放入PendingBlock中，然后再放入pending中
          writeBlocks(i+1, prefixTopSize);
          prefixStarts[i] -= prefixTopSize-1;
        }
      }
      if (prefixStarts.length < text.length) { // prefixStarts达不到最大size的话，会不断扩容
        prefixStarts = ArrayUtil.grow(prefixStarts, text.length);
      }
      // 修改不同的后缀。
      for(int i=pos;i<text.length;i++) { 
        prefixStarts[i] = pending.size();
      }
      lastTerm.copyBytes(text); // 缓存上一次的term
    }
```
该函数主要是判断已经输入的哪些相似的term可以组合成一个子FST结构。因为字典term输入都是按序排好了的，很多单词相似性非常高，可以利用FST结构搜索，本节主要看如何找到这些相似的terms。我们需要了解2个关键变量：
+ prefixStarts[i]
存放的是term中第i个元素与前一个term该元素不一致时， pending中目前存储的term个数，该数组反映的是这一堆terms的相似性。我们以如下terms为例：
```
     当前写入顺序                   当前term写入时，prefixStarts值         pending.size() - prefixStarts[i]最大值：
0    61 62 63                      0 0 0                               \
1    61 62 63 64                   0 0 0 1                             1-1=0
2    61 62 64                      0 0 2 1                             2-1=1
3    61 62 65 66 30                0 0 3 3 3                           3-3=0
4    61 62 65 66 31                0 0 3 3 4                           4-4=0
5    61 62 65 66 31 30             0 0 3 3 4 5                         5-5=0
6    61 62 65 66 31 30 30          0 0 3 3 4 5 6                       6-6=0
7    61 62 65 66 31 30 30 30       0 0 3 3 4 5 6 7                     7-7=0
8    61 62 65 66 31 30 30 30 30    0 0 3 3 4 5 6 7 8                   8-8=0
9    61 62 65 66 31 30 30 30 31    0 0 3 3 4 5 6 7 9                   9-9=0
10   61 62 65 66 31 30 30 30 32    0 0 3 3 4 5 6 7 10                  10-10=0
11   61 62 65 66 31 30 30 30 33    0 0 3 3 4 5 6 7 11                  11-11=0
12   61 62 65 66 31 30 30 30 34    0 0 3 3 4 5 6 7 12                  12-12=0
13   61 62 65 66 31 30 30 30 35    0 0 3 3 4 5 6 7 13                  13-13=0
14   61 62 65 66 31 30 30 30 36    0 0 3 3 4 5 6 7 14                  14-14=0
15   61 63                        0 15 3 3 4 5 6 7 14                  15-3=12
```
prefixStarts数组的变化如图所示：每层第一次出现的字母的位置为i，则currentRerm[i]!=lastTerm[i], prefixStarts[i]=pending.size()。而仅当pending.size() - prefixStarts[i] > minItemsInBlock（线上默认配置25）时， 会截取这部分terms进行构建子FST结构。 假如minItemsInBlock=10， 那么我们会选择从3-14行共12个term产生子FST。
+ PendingTerm与PendingBlock
单个term放在PendingTerm里面；多个相似性高的term组建起来，将这些terms相似前缀作为一个term，构建成一个子字典，形成一个PendingBlock，来代替这组相似的terms和别term继续组建更广泛的字典。

`TermsWriter.writeBlocks()`介绍了如何将选出来的相似性terms组成一个子字典的（FST）:
 ```
    void writeBlocks(int prefixLength, int count) throws IOException {
      int lastSuffixLeadLabel = -1;
      // 如果这个值为true，那么这个block中至少有一个term
      boolean hasTerms = false;
      // 如果这个值为true，那么这个block中至少有一个新产生的block
      boolean hasSubBlocks = false;
      // 计算起始位置, 从后向前取值
      int start = pending.size()-count;
      // 终止位置
      int end = pending.size();
      // 记录下一个 block 的起始位置
      int nextBlockStart = start;
      // floor block 中第一个term 的后缀的第一个字符当成 Lead label
      int nextFloorLeadLabel = -1;
      for (int i=start; i<end; i++) {
        // term 的后缀的第一个term
        PendingEntry ent = pending.get(i);
        // 保存了树中某个节点下的各个Term的byte
        int suffixLeadLabel;
        // term 的后缀的第一个字符
        if (ent.isTerm) {
          PendingTerm term = (PendingTerm) ent;
          // 和前缀一样长
          if (term.termBytes.length == prefixLength) { 
            suffixLeadLabel = -1; // 后缀值
          } else {
            // 就是prefixLength上某个字符
            suffixLeadLabel = term.termBytes[prefixLength] & 0xff; 
          }
        } else {
          PendingBlock block = (PendingBlock) ent;
          // 不同那个后缀
          suffixLeadLabel = block.prefix.bytes[block.prefix.offset + prefixLength] & 0xff; 
        }
        // 第prefixLength个字符不一致。
        if (suffixLeadLabel != lastSuffixLeadLabel) {
          // 这个block的的长度
          int itemsInBlock = i - nextBlockStart;
          // 如果当前term总个数超过minItemsInBlock，且剩余term个数大于maxItemsInBlock，则将当前所有term构建一个FST, 产生一个block
          if (itemsInBlock >= minItemsInBlock && end-nextBlockStart > maxItemsInBlock) {
            // 若这个block小于这次满足需求的总block, 那么就拆分
            boolean isFloor = itemsInBlock < count; 
            newBlocks.add(writeBlock(prefixLength, isFloor, nextFloorLeadLabel, nextBlockStart, i, hasTerms, hasSubBlocks));
            hasTerms = false;
            hasSubBlocks = false;
            // 下一个block的不相同的字母。
            nextFloorLeadLabel = suffixLeadLabel;
            // 记录下一个block的起始位置
            nextBlockStart = i;
          }
          // 更新term 的后缀的第一个字符
          lastSuffixLeadLabel = suffixLeadLabel;
        }
        if (ent.isTerm) {
          hasTerms = true;
        } else {
          hasSubBlocks = true;
        }
      }
      // 最后一个
      if (nextBlockStart < end) {
        int itemsInBlock = end - nextBlockStart;
        boolean isFloor = itemsInBlock < count;
        newBlocks.add(writeBlock(prefixLength, isFloor, nextFloorLeadLabel, nextBlockStart, end, hasTerms, hasSubBlocks));
      }
      PendingBlock firstBlock = newBlocks.get(0);
      // 将一个block的信息写入FST结构中（保存在其成员变量index中），FST是有限状态机的缩写，其实就是将一棵树的信息保存在其自身的结构中，而这颗树是由所有Term的每个byte形成的
      firstBlock.compileIndex(newBlocks, scratchBytes, scratchIntsRef); // scratchBytes，scratchIntsRef还没有存储
      // 对每个写入磁盘的block的前缀 prefix构建一个FST索引 // 所有block的FST索引联合成一个FST索引，并将联合的FST写入 root block
      // Remove slice from the top of the pending stack, that we just wrote:
      pending.subList(pending.size()-count, pending.size()).clear();
      pending.add(firstBlock); // 向里面写入了一个block
      newBlocks.clear();
    }
```
该函数主要的作用是：
1.将每minItemsInBlock(默认值25)个Term，通过`writeBlock()`组合成一个Term（目的是减少最顶层词典中词的个数，为PendingBlock），继续和这组相似的term构建FST结构。哪些相似的term可以组合成一个term呢？符合条件是这样的：
+ 该term和上一个term的第prefixLength个字符不同。
+ 只要每隔minItemsInBlock（默认25）个词就可以组合成一个term。
这里maxItemsInBlock（默认48）的主要作用是：只要剩余terms个数太少，那么将剩余terms和当前block一起组合成PendingBlock，一个block里面term最多只能为maxItemsInBlock个。
2.将剩余的PendTerm组装成一个PendingBlock。这里需要注意下，在1、2步骤组装成的PendingBlock时，会将子PengingBlock里面的FST读取出来，在第3步构建时，放入FST中。
3.调用`firstBlock.compileIndex()`将1,2产生的所有PendingBlock，构建成一个FST。FSP再放入第一个PendingBlock，继续放入pending中进行词典组装。

我们再继续看下`BlockTreeTermsWriter.writeBlock()`是怎么将这批terms组成一个PendingBlock。
```
    private PendingBlock writeBlock(int prefixLength, boolean isFloor, int floorLeadLabel, int start, int end,
                                    boolean hasTerms, boolean hasSubBlocks) throws IOException {
      long startFP = termsOut.getFilePointer(); // 获取当前block在tim中的起始位置
      boolean hasFloorLeadLabel = isFloor && floorLeadLabel != -1;
      final BytesRef prefix = new BytesRef(prefixLength + (hasFloorLeadLabel ? 1 : 0));
      System.arraycopy(lastTerm.get().bytes, 0, prefix.bytes, 0, prefixLength);
      prefix.length = prefixLength;
      int numEntries = end - start;
      int code = numEntries << 1; // block的term个数
      if (end == pending.size()) {
        // Last block:
        code |= 1;  // 标志是最后一个block
      }
      termsOut.writeVInt(code); // tim
      boolean isLeafBlock = hasSubBlocks == false;// 如果这个值为true，那么这个block中没有一个是新产生的block
      //System.out.println("  isLeaf=" + isLeafBlock);
      final List<FST<BytesRef>> subIndices; // 存放的是子fst结构
      boolean absolute = true;
       // block仅有terms, 而没有PendingBlock
      if (isLeafBlock) { 
        // Block contains only ordinary terms:
        subIndices = null;
         // 遍历每一个term，将其不同的后缀给存储起来
        for (int i=start;i<end;i++) { 
          PendingEntry ent = pending.get(i);
          PendingTerm term = (PendingTerm) ent;
          BlockTermState state = term.state;
           // 该词后缀长度
          final int suffix = term.termBytes.length - prefixLength;
          // 写入后缀内容
          suffixWriter.writeVInt(suffix);
          suffixWriter.writeBytes(term.termBytes, prefixLength, suffix);
          // 该词在多少文件中出现过
          statsWriter.writeVInt(state.docFreq);  
          if (fieldInfo.getIndexOptions() != IndexOptions.DOCS) {
            // 多出来的词频
            statsWriter.writeVLong(state.totalTermFreq - state.docFreq); 
          }
          // 将state中的元数据，抽取出来放在longs,bytesWriter中
          // Write term meta data， 将doc的block存放在这里
          // 将doc.pos,pay信息放入longs中
          postingsWriter.encodeTerm(longs, bytesWriter, fieldInfo, state, absolute);
           // 写入doc + pos + pay 三个在对应文件中与上个term的差值
          for (int pos = 0; pos < longsSize; pos++) {
            metaWriter.writeVLong(longs[pos]);
          }
          // 将byte写入metaWriter中
          bytesWriter.writeTo(metaWriter); 
          bytesWriter.reset();
          absolute = false;
        }
      } else { 
        // 该block子pending中包含子Block
        subIndices = new ArrayList<>();
        for (int i=start;i<end;i++) {
          PendingEntry ent = pending.get(i);
          if (ent.isTerm) {
            ......
          } else {
            PendingBlock block = (PendingBlock) ent;
            final int suffix = block.prefix.length - prefixLength;
            suffixWriter.writeVInt((suffix<<1)|1);
            suffixWriter.writeBytes(block.prefix.bytes, prefixLength, suffix);
            suffixWriter.writeVLong(startFP - block.fp);
            subIndices.add(block.index);
          }
        }
      }
      // 写入suffixWriter中的长度，写入tim中
      termsOut.writeVInt((int) (suffixWriter.getFilePointer() << 1) | (isLeafBlock ? 1:0)); 
       // 将suffixWriter中的byte[]写入tim中
      suffixWriter.writeTo(termsOut);
      suffixWriter.reset();
      // Write term stats byte[] blob
      termsOut.writeVInt((int) statsWriter.getFilePointer()); //写入长度
      statsWriter.writeTo(termsOut); // 写入内容
      statsWriter.reset();
      // Write term meta data byte[] blob
      termsOut.writeVInt((int) metaWriter.getFilePointer());
      metaWriter.writeTo(termsOut); //
      metaWriter.reset();
      if (hasFloorLeadLabel) { // 这里给多加了一个字段
        prefix.bytes[prefix.length++] = (byte) floorLeadLabel;
      }
      // 产生一个新的PendingBlock
      return new PendingBlock(prefix, startFP, hasTerms, isFloor, floorLeadLabel, subIndices); 
    }
```
该函数主要是将start-end长度的terms组装成一个PendingBlock, 针对PendingTerm采集的term信息如下：
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_tim6.png" height="350" width="800"/>
然后将suffixWriter、statsWriter、metaWriter依次写入tim文件中。若这批terms中有PendingBlock的话，只在suffixWriter记录该block前缀、该block在tim中的存储位置，并将该block下所有的FST保存在该新创建的PendingBlock中，以便后续根据这些FST产生新的FST。


我们再继续看下firstBlock.compileIndex()，是如何将`writeBlocks()`产生的所有Block合并产生一个FST的：
```
    public void compileIndex(List<PendingBlock> blocks, RAMOutputStream scratchBytes, IntsRefBuilder scratchIntsRef) throws IOException {
       // 写入scratchBytes比较重要，作为FST数的
       //写入scratchBytes：最高62为存放tim起始位置，低一位存放是否有terms，最低位存放是否是floor机器。
      scratchBytes.writeVLong(encodeOutput(fp, hasTerms, isFloor));
      if (isFloor) {
        // 同时将每个block在tim中的fp也保存起来
        scratchBytes.writeVInt(blocks.size()-1); 
        // 遍历每一个block
        for (int i=1;i<blocks.size();i++) { 
          PendingBlock sub = blocks.get(i); 
          // 下一个block与上个block相比，不同的字符
          scratchBytes.writeByte((byte) sub.floorLeadByte); 
          scratchBytes.writeVLong((sub.fp - fp) << 1 | (sub.hasTerms ? 1 : 0));
        }
      }
      // 这里会新产生以Builder
      final ByteSequenceOutputs outputs = ByteSequenceOutputs.getSingleton();
      final Builder<BytesRef> indexBuilder = new Builder<>(FST.INPUT_TYPE.BYTE1,
                                                           0, 0, true, false, Integer.MAX_VALUE,
                                                           outputs, true, 15);
      // scratchBytes里面的长度。存的是fp&hasTerms&isFloor
      final byte[] bytes = new byte[(int) scratchBytes.getFilePointer()]; 
      scratchBytes.writeTo(bytes, 0);
      // 将本身的字符放入FST中
      indexBuilder.add(Util.toIntsRef(prefix, scratchIntsRef), new BytesRef(bytes, 0, bytes.length));
      scratchBytes.reset();
      // 将 sub-block 的所有 index 写入indexBuilder(比较重要)
      // Copy over index for all sub-blocks
      for(PendingBlock block : blocks) {
        if (block.subIndices != null) {
          // 遍历所有的block
          for(FST<BytesRef> subIndex : block.subIndices) {
          // 当做普通字符串再加入新的fst中
            append(indexBuilder, subIndex, scratchIntsRef);
          }
          block.subIndices = null;
        }
      }
      // 生成新的FST
      index = indexBuilder.finish();
    }
```
该函数主要是将产生的PengingBlock组装成一个FST， 主要做了如下工作：
1) 构建FST的output, output里面存放了如下信息：
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_tim8.png" height="180" width="650"/>
output能够快速定位block在tim中的存储位置，主要由一下几部分构成：
+ block0在tim文件中的起始位置。
+ 即将构建的FST是否有term。
+ blockCount-1个数组，分别反映了该fst下blockCount-1个子block在tim中的位置。
+ floorLeadByte后续紧接着的block的terms第一个term最后一个字符(与本block的terms最后一个term不相同的字符)
2) 将输入的产生的PendingBlock列表构建成一个FST，构建原理可以参考：<a href="https://kkewwei.github.io/elasticsearch_learning/2020/02/25/Lucene8-2-0%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-%E8%AF%8D%E5%85%B8fst%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/">Lucene8.2.0底层架构-词典fst原理解析</a>
构建出来的FST如下所示：
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_tim7.png" height="400" width="400"/>
也就是说，只要通过FST的output0，就可以找到block0、block1、block2、block3在tim中存储的位置。其中block0存放着整个FST结构，和别的term再次构建FST.

## FST结构写入tip中
当前域所有term放入词典后，开始将当前域的词典索引结构FSP放入tip文件中：
```
    public void finish() throws IOException {
      if (numTerms > 0) {
        pushTerm(new BytesRef());
        // 将剩余的再次产生一个PendingTerm
        pushTerm(new BytesRef());
        writeBlocks(0, pending.size()); 
        // 有个最终root的 block
        // We better have one final "root" block:
        final PendingBlock root = (PendingBlock) pending.get(0);
        // Write FST to index
        indexStartFP = indexOut.getFilePointer();  
        // 将fst写入tip文件
        root.index.save(indexOut); 
        // 该域写入的第一个词
        BytesRef minTerm = new BytesRef(firstPendingTerm.termBytes); 
        // 该域写入的最后一个词
        BytesRef maxTerm = new BytesRef(lastPendingTerm.termBytes); 
        fields.add(new FieldMetaData(fieldInfo,
                                     ((PendingBlock) pending.get(0)).index.getEmptyOutput(),
                                     numTerms,
                                     indexStartFP,
                                     sumTotalTermFreq,
                                     sumDocFreq,
                                      // 该词有多少个文档
                                     docsSeen.cardinality(),
                                     longsSize,
                                     minTerm, maxTerm));
      }
    }
```
该函数主要走了如下事情：
1.通过第一次调用`pushTerm(new BytesRef())`将pending中缓存的PendingBlocks组装成一个PendingBlock。
2.调用`writeBlocks(0, pending.size())`将最后一个block里缓存的子fst构成一个最终的FST。
3.将这个FST写入tip文件中。

# 总结
单个term的词频等信息保存正在Doc文件中，使用跳表结构来快速索引。segment每个域都有个词典结构，词典索引结构FST存放在tip文件中， tim中存放了每个PendingBlock里所有term的内容，以及每个词的统计信息， 每个词的倒排索引结构其实是存放在doc文件中。doc中存放了跳表结构，可以快速定位某个词的词频等信息。

# 参考
https://kkewwei.github.io/elasticsearch_learning/2020/02/25/Lucene8-2-0%E5%BA%95%E5%B1%82%E6%9E%B6%E6%9E%84-%E8%AF%8D%E5%85%B8fst%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/
https://www.amazingkoala.com.cn/Lucene/suoyinwenjian/2019/0401/43.html
