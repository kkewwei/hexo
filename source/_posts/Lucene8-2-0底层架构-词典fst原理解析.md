---
title: Lucene8.2.0底层架构-词典fst原理解析
date: 2020-02-25 19:29:57
tags: Lucene、词典、FST
toc: true
categories: Lucene
---
lucene中使用fst(Finite State Transducer)来对字典压缩存储、快速查询词典, fst结构紧凑, 在时间复杂度和空间复杂度之间均衡, 和HashMap结构相比, 该结构体共享字符串前缀和后缀以达到精简内存使用, FST逻辑上构成了一个有向无环图, 本文将对fst结构体的构建过程进行分析。
# 使用示例
在输入字符串时词典时, 必须经过排序, 这样产生的fst才最小。
```
        // 输入字符串
        String inputValues[] = {"abcde","abdde","abede","accde","adcde"};
        // 对应字符串的附加output
        Long outputValues[] = {1L,2L,3L,4L,5L};
        PositiveIntOutputs outputs = PositiveIntOutputs.getSingleton();
        //构建FST
        Builder<Long> builder = new Builder<>(FST.INPUT_TYPE.BYTE1,outputs);
        IntsRefBuilder scratchInts = new IntsRefBuilder();
        for (int i = 0; i < inputValues.length; i++) {
            BytesRef scratchBytes =  new BytesRef(inputValues[i].getBytes());
            scratchInts.copyUTF8Bytes(scratchBytes);
            builder.add(scratchInts.toIntsRef(),outputValues[i]);
        }
        // 构建fst结构
        FST<Long> fst = builder.finish();
        // 读取该字符串对应的值
        Long value = Util.get(fst, new BytesRef("abcde"));
        System.out.println(value);
```
以上示例我们使用fst作为容器存储字符串, 通过fst查询获得相应out的过程, 看起来是不是和HashMap使用场景很像, 我们可以看下fst构成的<a href="http://examples.mikemccandless.com/fst.py?terms=abcde%2F1%0D%0Aabdde%2F2%0D%0Aabede%2F3%0D%0Aaccde%2F4%0D%0Aadcde%2F4&cmd=Build+it%21">逻辑存储结构</a>。

# Builder
我们可以看到, fst主要是在Builder完成结构构建的, 实际存储是通过FST来完成的, 为了更好地理解, 我们先对该Builder属性做个介绍:
该类的属性:
```
  // 共享后缀使用的, 存储当前存在的后缀,
  private final NodeHash<T> dedupHash;
  // 真正保存构建的fst结构的地方
  final FST<T> fst;
  // 缓存的前一个字符串
  private final IntsRefBuilder lastInput = new IntsRefBuilder();
  // 缓存的前一个字符串形成的UnCompiledNode。前一个字符串的不同后缀会在当前字符串写入时候freeze存储到fst中。
  private UnCompiledNode<T>[] frontier;
  是当前添加的字符串所形成的状态节点，而前面添加的字符串形成的状态节点通过指针相互引用。
  // 记录的当前节点所有边的长度，临时变量。
  int[] reusedBytesPerArc = new int[4];
  // 默认为true
  boolean allowArrayArcs;
  // 和FST中的bytes是同一个对象, 真正存储fst结构的地方。
  BytesStore bytes;
```
1.UnCompiledNode和CompiledNode
Node作为FST的节点, 分为以上两种角色, UnCompiledNode表示该节点还可能后来的字符共享, 由于输入的字符串有先后顺序, 若不能共享的节点经过freezeTail处理后(compile), 就变成了CompiledNode。
UnCompiledNode的主要属性:
```
  final Builder<T> owner;
    // 该节点的出边条数
    public int numArcs;
    // 该节点总共的出边
    public Arc<T>[] arcs;
    // 表示从Node出发的Arc数量是0，当Node是Final Node时，output才有值。
    public boolean isFinal;
     //这个节点上UnCompiledNode进来的边
    public long inputCount;
    // 从根节点到本节点的深度
    public final int depth;
```
CompiledNode属性node表示该节点对应的边存放在fst中的address。
2.Arc
Arc表示FST中的边。
```
    // 每个表代表着一个字符,
    public int label;
    // 该边目标节点
    public Node target;
    // 表示当前Arc上附带的值, 可以认为key对应的value。
    public T output;
```
# 构建
真正开始构建fst结构是从`Builder.add`中开始的。
```
  public void add(IntsRef input, T output) throws IOException { // output: BytesRef
    // 比较当前字符串和前一个字符串相同的前缀长度
    int pos1 = 0;
    int pos2 = input.offset;
    final int pos1Stop = Math.min(lastInput.length(), input.length);
    while(true) {
      frontier[pos1].inputCount++;
      //System.out.println("  incr " + pos1 + " ct=" + frontier[pos1].inputCount + " n=" + frontier[pos1]);
      if (pos1 >= pos1Stop || lastInput.intAt(pos1) != input.ints[pos2]) {
        break;
      }
      pos1++;
      pos2++;
    }
    final int prefixLenPlus1 = pos1+1; // 公共长度+1

    // 扩容frontier, 最起码保证frontier能够存放当前的字符串(前一个字符串一定可以存放的下)
    if (frontier.length < input.length+1) {
      final UnCompiledNode<T>[] next = ArrayUtil.grow(frontier, input.length+1);
      for(int idx=frontier.length;idx<next.length;idx++) {
        next[idx] = new UnCompiledNode<>(this, idx);
      }
      frontier = next;
    }
    // 前一个字符串的不同后缀不会再共享了, 那么我们就冰冻前一个字符串不同的后缀, 并存储。
    freezeTail(prefixLenPlus1);
    // 针对当前字符串不同的后缀, 首先构建对应的边
    for(int idx=prefixLenPlus1;idx<=input.length;idx++) {
      frontier[idx-1].addArc(input.ints[input.offset + idx - 1],
                             frontier[idx]);
      frontier[idx].inputCount++;
    }
   // 最后一个边设置为true
    final UnCompiledNode<T> lastNode = frontier[input.length];
    if (lastInput.length() != input.length || prefixLenPlus1 != input.length + 1) {
      lastNode.isFinal = true;
      lastNode.output = NO_OUTPUT;
    }

    // 遍历针对冲突的output，尽可能在相同的前缀里共享。
    for(int idx=1;idx<prefixLenPlus1;idx++) {
      final UnCompiledNode<T> node = frontier[idx];
      final UnCompiledNode<T> parentNode = frontier[idx-1];
      // 既然可以共享前缀，那么前缀边和本边是一致的，那么一定是一条直线，没有分叉。获取当前边的output
      final T lastOutput = parentNode.getLastOutput(input.ints[input.offset + idx - 1]);
      final T commonOutputPrefix;// BytesRef
      final T wordSuffix;// BytesRef
       // 读取上一个的output，若存在，检查是否可以和新的合并
      if (lastOutput != NO_OUTPUT) {
        // 获取第一条共享边和output的相同前缀
        commonOutputPrefix = fst.outputs.common(output, lastOutput);
        // 扣除相同的output前缀
        wordSuffix = fst.outputs.subtract(lastOutput, commonOutputPrefix); // lastOutput-commonOutputPrefix=不同的后缀部分
        //并修正该边的output
        parentNode.setLastOutput(input.ints[input.offset + idx - 1], commonOutputPrefix);
        //将剩余后缀一定到下一个节点对应的边。
        node.prependOutput(wordSuffix);
      } else {
        commonOutputPrefix = wordSuffix = NO_OUTPUT;
      }
      // 检查输入时候的output, 并在此向后循环
      output = fst.outputs.subtract(output, commonOutputPrefix);
    }
    // 本次输入的内容和上次输入的内容一样，则重新分配output
    if (lastInput.length() == input.length && prefixLenPlus1 == 1+input.length) {
      lastNode.output = fst.outputs.merge(lastNode.output, output);
    } else {
      // 将剩余output放到新创建的最前的那条边
      frontier[prefixLenPlus1-1].setLastOutput(input.ints[input.offset + prefixLenPlus1-1], output);
    }
    // 处理完了, 替换上一次的字符串。
    lastInput.copyInts(input);
  }
```
该函数主要做了如下几个事情:
1. 获取相同前缀长度
2. 调用`freezeTail`将前一个字符串保存在frontier中的不同后缀冷冻起来, 也就是说前一个字符串不同的后缀将不会被别的字符串共享, 那么我们就可以将这个后缀(前一个字符串的)给存储起来。存储的时候还会做到共享后缀
3. 前一个字符串的后缀在frontier中已经废弃, 此时将当前字符串的后缀产生UnCompiledNode, 存放到frontier中。
4. 前后两个字符串尽量共享相同的前缀output。
5. 将当前写入的字符串output经过共享后, 剩余部分写入当前后缀第一个字符串对应的边上。

我们接下来关注下`Builder.freezeTail()`是如何做到共享后缀, 并存储的。
```
  private void freezeTail(int prefixLenPlus1) throws IOException {
    //这里downTo大于等于1可以保证根节点不会被写入到FST中去，根节点必须要所有节点写完之后才能写到FST
    final int downTo = Math.max(1, prefixLenPlus1);
    // 节点idx，从后向前，只压缩存储不同的后缀
    for(int idx=lastInput.length(); idx >= downTo; idx--) {
      final UnCompiledNode<T> node = frontier[idx]; // 结尾那个NULL的node
      final UnCompiledNode<T> parent = frontier[idx-1];
        final boolean isFinal = node.isFinal || node.numArcs == 0;
        // 用compileNode 函数将 Node写入FST，得到一个 CompiledNode，替换掉 Arc的target, compileNode可能返回的是一个已经写入FST的 Node对应边的存储地址，这样就达到了共享 Node 的目的
        parent.replaceLast(lastInput.intAt(idx-1),
                             compileNode(node, 1+lastInput.length()-idx),
                             nextFinalOutput,
                             isFinal);
        }
      }
    }
  }
```
为啥函数名称是freezeTail, 因为是从最后一个节点开始存储, 这样便于查找相同的后缀。 针对每个UnCompiledNode调用compileNode, 将该节点转变为CompiledNode。 接下来再一起看下`Builder.compileNode`是如何做到共享存储的。
```
  private CompiledNode compileNode(UnCompiledNode<T> nodeIn, int tailLength) throws IOException {
    final long node;
    long bytesPosStart = bytes.getPosition();
    if (dedupHash != null && (doShareNonSingletonNodes || nodeIn.numArcs <= 1) && tailLength <= shareMaxTailLength) {// doShareNonSingletonNodes默认为false
       // 说明该节点是结尾。
      if (nodeIn.numArcs == 0) {
        // Node的Arc数量为0，fst.addNode 会返回 -1。已经将nodeId的value存放了fst中
        node = fst.addNode(this, nodeIn);
        lastFrozenNode = node;
      } else {
        // 将该节点存储到dedupHash中, 若已经存在的话, 就直接共享了。
        node = dedupHash.add(this, nodeIn);
      }
    } else {
      // 把node写入fst中
      node = fst.addNode(this, nodeIn);
    }
    long bytesPosEnd = bytes.getPosition();
    // 若fst中有新增存储的话, 就说明该节点没有被共享。
    if (bytesPosEnd != bytesPosStart) {
      // The FST added a new node:
      lastFrozenNode = node;
    }
    // 这里将这个nodeIn给清空了，意味着该边被compile了，
    nodeIn.clear();
    final CompiledNode fn = new CompiledNode();
    fn.node = node;
    return fn;
```
该函数主要作用实际调用`dedupHash.add`将当前节点存储到fst中, 若已经存在的话, 则共享已经存在的节点。这里可能有人有疑惑, 边才是存储的真正的字符串, 存储节点作用不大呀, 实际上,每个节点存储时, 将边对应的lebal给考虑进来了。
我们看下`dedupHash.add`是如何共享存储的。
```
  public long add(Builder<T> builder, Builder.UnCompiledNode<T> nodeIn) throws IOException {
    // 该节点hash出一个值
    final long h = hash(nodeIn);
    long pos = h & mask;
    int c = 0;
    while(true) {
      final long v = table.get(pos);// 没有便是0
      if (v == 0) {
        // freeze & add
        // 该节点在fst中的存储位置
        final long node = fst.addNode(builder, nodeIn);
        //System.out.println("  now freeze node=" + node);
        assert hash(node) == h : "frozenHash=" + hash(node) + " vs h=" + h;
        count++;
        // node=fst中该词的存储位置
        table.set(pos, node);
        // Rehash at 2/3 occupancy:
        if (count > 2*table.size()/3) {
          rehash();
        }
        return node;
      } else if (nodesEqual(nodeIn, v)) {
        // same node is already here
        return v;
      }
      // quadratic probe
      pos = (pos + (++c)) & mask;
    }
  }
```
该函数主要是hash值判断节点是否存在:
1.若不为0, 则说明该节点存在, 那么就可以共享, hash返回值即为该节点在fst中的存储位置, 并返回。
2.若返回值为0, 则说明该节点不存在, 调用`fst.addNode`将该节点存储起来, 并将该节点及output放入hash列表中。
hash取值存在, 则认为该节点开头的后缀都是已经存储, 可以直接共享。我们看下hash函数取了哪些变量:
```
  private long hash(Builder.UnCompiledNode<T> node) {
    final int PRIME = 31;
    long h = 0;
    for (int arcIdx=0; arcIdx < node.numArcs; arcIdx++) {
      final Builder.Arc<T> arc = node.arcs[arcIdx];
      h = PRIME * h + arc.label; // label  i的节点边的lebal决定了i-1边的lebal
      long n = ((Builder.CompiledNode) arc.target).node;
      h = PRIME * h + (int) (n^(n>>32));
      h = PRIME * h + arc.output.hashCode(); // output相关
      h = PRIME * h + arc.nextFinalOutput.hashCode(); //nextFinalOutput相关
      if (arc.isFinal) {
        h += 17;
      }
    }
    return h & Long.MAX_VALUE;
  }
```
可以看下, 针对单个UnCompiledNode进行hash, 取值了如下变量:所有的边lebal, 该节点后续节点的node值、output值、nextFinalOutput值。这里可能会有疑问, 下图的n2节点和n节点, hash值一样, 那么可以确定n2和n节点对应的后缀字符串相同吗?  其实与`arc.target.node`关系很大, 该函数指的是当前节点后一个节点在内存中的存储位置, 若后一个节点共享, 使用相同的存储位置, 说明后一个节点是共享, 使用相同后缀, 只要当前节点再相同 ,那么当前节点也使用相同后缀, 而达到共享存储的目的;
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_fst1.png" height="160" width="250"/>
我们先留着这个疑问, 看下CompiledNode.node值是如何计算得到的吧。
我们再接着了解`fst.addNode`是如何返回node的
```
  long addNode(Builder<T> builder, Builder.UnCompiledNode<T> nodeIn) throws IOException {
    T NO_OUTPUT = outputs.getNoOutput();
     // 处理没有Arc的空节点
    if (nodeIn.numArcs == 0) {
      if (nodeIn.isFinal) {
        return FINAL_END_NODE;
      } else {
        return NON_FINAL_END_NODE;
      }
    }
    // 记录当前Node的Address
    final long startAddress = builder.bytes.getPosition();
    // 判断是否用FIXED_ARRAY方式存Arc的信息
    final boolean doFixedArray = shouldExpand(builder, nodeIn);
    if (doFixedArray) { // 此时为false
      if (builder.reusedBytesPerArc.length < nodeIn.numArcs) {
        builder.reusedBytesPerArc = new int[ArrayUtil.oversize(nodeIn.numArcs, 1)];
      }
    }
    long lastArcStart = builder.bytes.getPosition();
    int maxBytesPerArc = 0;
    // 对该节点直连的下游遍历
    for(int arcIdx=0; arcIdx < nodeIn.numArcs; arcIdx++) {
      final Builder.Arc<T> arc = nodeIn.arcs[arcIdx];
      final Builder.CompiledNode target = (Builder.CompiledNode) arc.target; // 目标节点
      int flags = 0;
      ......
      builder.bytes.writeByte((byte) flags);
      writeLabel(builder.bytes, arc.label);
      ......
       // 若使用相同长度, 那么记录下每条边占用的存储bytes的最大值
      if (doFixedArray) {
        builder.reusedBytesPerArc[arcIdx] = (int) (builder.bytes.getPosition() - lastArcStart); // 记录下每个Arc的长度
        lastArcStart = builder.bytes.getPosition();
        maxBytesPerArc = Math.max(maxBytesPerArc, builder.reusedBytesPerArc[arcIdx]);// 记录最长的Arc的长度
      }
    }
    // 该节点每条边占用fst存储相同的长度
    if (doFixedArray) {
      // header(byte) + numArcs(vint) + numBytes(vint)
      final int MAX_HEADER_SIZE = 11;
      assert maxBytesPerArc > 0;
      int labelRange = nodeIn.arcs[nodeIn.numArcs - 1].label - nodeIn.arcs[0].label + 1;
      // 直接赋值为false
      boolean writeDirectly = labelRange > 0 && labelRange < Builder.DIRECT_ARC_LOAD_FACTOR * nodeIn.numArcs;
      writeDirectly = false;
      byte header[] = new byte[MAX_HEADER_SIZE];
      ByteArrayDataOutput bad = new ByteArrayDataOutput(header);
      if (writeDirectly) {
        bad.writeByte(ARCS_AS_ARRAY_WITH_GAPS);
        bad.writeVInt(labelRange);
      } else {
      // 写入标志位，代表ARC是FIXED_ARRAY，就是前面header长度内容
        bad.writeByte(ARCS_AS_ARRAY_PACKED);
        bad.writeVInt(nodeIn.numArcs);// 写入Arc的数量
      }
      bad.writeVInt(maxBytesPerArc); // 写入最大的那个outtput
      int headerLen = bad.getPosition();
      final long fixedArrayStart = startAddress + headerLen;
      // 默认为false，直接存储到fst中
      if (writeDirectly) {
        writeArrayWithGaps(builder, nodeIn, fixedArrayStart, maxBytesPerArc, labelRange);
      } else {
        // 每条边按照相同的长度存储到fst
        writeArrayPacked(builder, nodeIn, fixedArrayStart, maxBytesPerArc);
      }
      // now write the header
      builder.bytes.writeBytes(startAddress, header, 0, headerLen);
    }
    当前节点在fst中存储的终点位置
    final long thisNodeAddress = builder.bytes.getPosition()-1;
    // 二进制数组值收尾颠倒
    builder.bytes.reverse(startAddress, thisNodeAddress);
    builder.nodeCount++;
    return thisNodeAddress;
  }
```
该函数主要做了如下事情:
1.将该节点的flags、label存储到fst的bytes中。
2.首先判断存储存储该节点每条边是否需要等长存储, 若等长存储的话, 然后再调用`writeArrayPacked`重写bytes进行等长存储。
3.调用`bytes.reverse`将该节点的存储byte颠倒存储。

当写入所有字符串后, 再调用`builder.finish()`完成了整个fst的构建。
```
  public FST<T> finish() throws IOException {
    final UnCompiledNode<T> root = frontier[0];
     // 参数为0, 表示将所有frontier维护的除根节点之外的所有缓存节点都写入FST
    freezeTail(0);
    //结束FST的写入
    fst.finish(compileNode(root, lastInput.length()).node); // 将头也写入
    return fst;
  }
```
该函数主要做了如下事情:
1.首先调用`freezeTail(0)`将frontier除根之外的所有节点存储到fst中。
2.调用`compileNode`将根节点存储到fst中
3.通过`fst.finish`将fst的根边缓存起来。

我们再看下`fst.finish`是怎么缓存根节点的所有边的。
```
  void finish(long newStartNode) throws IOException {
    startNode = newStartNode;
    // 将bytes实际缓存的数据写入真正的存储中
    bytes.finish();
    cacheRootArcs(); //
  }
   private void cacheRootArcs() throws IOException {
    final Arc<T> arc = new Arc<>();
    //读取root节点的第一条边
    getFirstArc(arc);
    // 边存在的话
    if (targetHasArcs(arc)) {
      final BytesReader in = getBytesReader();
      // 128个。第一个节点是根节点
      Arc<T>[] arcs = (Arc<T>[]) new Arc[0x80];
      // 从in的target处开始读取边信息，放入arc中。读取第一个节点也即根节点
      readFirstRealTargetArc(arc.target, arc, in);
      int count = 0;
      while(true) {
        // 若读取lebel大于128. 就退出
        if (arc.label < arcs.length) {
          arcs[arc.label] = new Arc<T>().copyFrom(arc);
        } else {
          break;
        }
        if (arc.isLast()) {
          break;
        }
        readNextRealArc(arc, in);
        count++;
      }
      int cacheRAM = (int) ramBytesUsed(arcs);
      // Don't cache if there are only a few arcs or if the cache would use > 20% RAM of the FST itself:
      if (count >= FIXED_ARRAY_NUM_ARCS_SHALLOW && cacheRAM < ramBytesUsed()/5) {
        cachedRootArcs = arcs;
        cachedArcsBytesUsed = cacheRAM;
      }
    }
  }
```
该函数主要做了如下事情:
1.生成128的数组, root节点最多有128条边, 为什么呢? asic码规定了字母最多128个, 字符串也就最多拥有128个不同首字母。这里将字母编码成了10进制的asic码。
2.依次读取root节点的所有边。
3.若root的边小于5条, 或者缓存占用内存大于1/5的话, 那么就没有缓存必要了。

最终存放在fst.bytes中的结构<a href="http://examples.mikemccandless.com/fst.py?terms=abcde%2F1%0D%0Aabdde%2F2%0D%0Aabede%2F3%0D%0Aaccde%2F4%0D%0Aadcde%2F5%0D%0A&cmd=Build+it%21">如下</a>:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_fst5.png" height="150" width="600"/>

## 构建fst图示例
我们就以开始写入第二个字符串abdde的过程进行介绍fst的构建过程。
1.在第二个字符串写入之前存储如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_fst2.png" height="140" width="700"/>
2.在写入abdde时,对第一个字符串freezeTail后的存储如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_fst3.png" height="300" width="850"/>
3.完成abdde写入后的存储如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/lucene_fst4.png" height="300" width="850"/>

# 查询
查询主要通过` Long value = Util.get(fst, new BytesRef("abcde"));`来实现的, 我们看下该函数具体做了哪些事情:
```
  public static<T> T get(FST<T> fst, BytesRef input) throws IOException {
    // 仅仅将根节点的存储位置放入arc中
    final FST.Arc<T> arc = fst.getFirstArc(new FST.Arc<T>());
    // 遍历每一个边
    for(int i=0;i<input.length;i++) {
      if (fst.findTargetArc(input.bytes[i+input.offset] & 0xFF, arc, arc, fstReader) == null) {
        return null;
      }
      // 对查找的边的output进行组合
      output = fst.outputs.add(output, arc.output);
    }
    if (arc.isFinal()) {
      return fst.outputs.add(output, arc.nextFinalOutput);
    }
  }
```
我们主要看下针对每个字母, `fst.findTargetArc`是如何进行查找的。
```
  // 从in中读取一个字符串labelToMatch对应的边, 该边的终点和follow指向的终点相同, 将该边放入arc中
  private Arc<T> findTargetArc(int labelToMatch, Arc<T> follow, Arc<T> arc, BytesReader in, boolean useRootArcCache) throws IOException {
    in.setPosition(follow.target);
    byte flags = in.readByte();
    // 首先判断缓存的root节点的边是否缓存了, 若缓存了。直接到对应的元素中取值
    if (useRootArcCache && cachedRootArcs != null && follow.target == startNode && labelToMatch < cachedRootArcs.length) {
    final Arc<T> result = cachedRootArcs[labelToMatch];
    // 该字母对用的
    if (result == null) {
        return null;
      } else {
        arc.copyFrom(result);
        return arc;
      }
    }
    if (flags == ARCS_AS_ARRAY_WITH_GAPS) {
       ......
    } else if (flags == ARCS_AS_ARRAY_PACKED) {
    // 该节点下游的所有边被长存储
      arc.numArcs = in.readVInt();
      arc.bytesPerArc = in.readVInt();
      arc.posArcsStart = in.getPosition();
      // 节点所有的边都排好序了, 二分查找该边。
      int low = 0;
      int high = arc.numArcs - 1;
      while (low <= high) {
        int mid = (low + high) >>> 1;
        in.setPosition(arc.posArcsStart - (arc.bytesPerArc * mid + 1));
        int midLabel = readLabel(in);
        final int cmp = midLabel - labelToMatch;
        if (cmp < 0) {
          low = mid + 1;
        } else if (cmp > 0) {
          high = mid - 1;
        } else {
          arc.arcIdx = mid - 1;
          return readNextRealArc(arc, in);
        }
      }
      return null;
    }
    // 首先读取根节点的第一条边, 查看是否符合查询。
    readFirstRealTargetArc(follow.target, arc, in);
    while(true) {
      if (arc.label == labelToMatch) {
        return arc;
      } else {
        // 继续读取该节点下一条边。
        readNextRealArc(arc, in);
      }
    }
  }
```
从该节点的所有边查找条件分4种情况:
1.若根缓存有数据, 那么直接从缓存中查找数据。
2.使用ARCS_AS_ARRAY_WITH_GAPS方式读取边。(满足等长存储的情况下, 是否直接存储。已经废弃, 可参考写入时FST类中656行, writeDirectly直接被置为false).
3.若该节点所有边都按照等长排序, 那么二分查找该节点下所有的边。这里存储时, 若边太多或者太深, 将使用空间换时间的方式加快查找速度, 每条边使用相同的存储空间以快速遍历。
4.否则线性遍历该节点下所有的边, 直到找到符合的边。

# 参考
https://blog.51cto.com/sbp810050504/1361551
http://examples.mikemccandless.com/fst.py