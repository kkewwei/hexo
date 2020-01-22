---
title: ES段合并逻辑分析
date: 2019-10-17 19:02:48
tags: ES, Semgent Merge
toc: true
categories: Elasticsearch
---
ES中在refresh、flush过程中, 都有可能触发段合并, 段合并是将小的段合并成大的段, merge过程有两个好处:
1. 加快查询速度
2. 减少内存空间的占用
本文主要从ES7.3层面讲解是如何进行小的段文件合并成大的段文件的, 至于lucene层面的段合并, 之后也会整理一篇文档出来。

# merge触发条件
Merge首先会进入到IndexWriter的如下代码:
```
final void maybeMerge(MergePolicy mergePolicy, MergeTrigger trigger, int maxNumSegments) throws IOException {
    ensureOpen(false);
    boolean newMergesFound = updatePendingMerges(mergePolicy, trigger, maxNumSegments);
    mergeScheduler.merge(this, trigger, newMergesFound);
  }
```
我们可以看到参数trigger, 代表了所有可能触发merge的情况:
```
public enum MergeTrigger {
  /**
   * 由段flush触发merge
   */
  SEGMENT_FLUSH,
  /**
   * 可能因为一个commit触发full flush
   */
  FULL_FLUSH,
  /**
   * 用户通过接口手动触发merge
   */
  EXPLICIT,
  /**
   * 由前一个成功的merge触发的
   */
  MERGE_FINISHED,
  /**
   * IndexWriter类关闭而触发
   */
  CLOSING
}
```

# segments的挑选
我们首先需要确定, 哪些端是需要合并的、是可以一起合并的:
```
  private synchronized boolean updatePendingMerges(MergePolicy mergePolicy, MergeTrigger trigger, int maxNumSegments)
    throws IOException {
    ......
    boolean newMergesFound = false;
    final MergePolicy.MergeSpecification spec;
    // 强制合并merge或者合并完之后再次进入
    if (maxNumSegments != UNBOUNDED_MAX_MERGE_SEGMENTS) {
     ......
      // 是主动merge, 那么就去选择哪些segment需要合并
    } else {
      spec = mergePolicy.findMerges(trigger, segmentInfos, this);
    }
    # 那么将改merge注册下。
    newMergesFound = spec != null;
    if (newMergesFound) {
      final int numMerges = spec.merges.size();
      for(int i=0;i<numMerges;i++) {
        // 把需要merge的段放进来
        registerMerge(spec.merges.get(i));
      }
    }
    return newMergesFound;
  }
```
这里策略策略采用的:TieredMergePolicy, 根据名字我们就可以猜测, 每次挑选时都是按照每一阶层进行挑选的。
```
  public MergeSpecification findMerges(MergeTrigger mergeTrigger, SegmentInfos infos, MergeContext mergeContext) throws IOException {
    // 该shard正在合并的段
    final Set<SegmentCommitInfo> merging = mergeContext.getMergingSegments();
    ......
    // 按照semgnet大小排序，大的段拍起前面
    List<SegmentSizeAndDocs> sortedInfos = getSortedBySegmentSize(infos, mergeContext);
    Iterator<SegmentSizeAndDocs> iter = sortedInfos.iterator();
     // 获取所有段总的删除比率
    final double totalDelPct = 100 * (double) totalDelDocs / totalMaxDoc;
    // 允许的删除比率
    int allowedDelCount = (int) (deletesPctAllowed * totalMaxDoc / 100);
    iter = sortedInfos.iterator();
    // remove large segments from consideration under two conditions.
    // 1> Overall percent deleted docs relatively small and this segment is larger than 50% maxSegSize
    // 2> overall percent deleted docs large and this segment is large and has few deleted docs
    // 所有的没有在合并的段都遍历
    while (iter.hasNext()) {
      SegmentSizeAndDocs segSizeDocs = iter.next();
      double segDelPct = 100 * (double) segSizeDocs.delCount / (double) segSizeDocs.maxDoc;
      // 去掉大于2.5GB的段 && 删除比率低的段
      if (segSizeDocs.sizeInBytes > maxMergedSegmentBytes / 2 && (totalDelPct <= deletesPctAllowed || segDelPct <= deletesPctAllowed)) {
        iter.remove();
        tooBigCount++; // Just for reporting purposes.
        totIndexBytes -= segSizeDocs.sizeInBytes;
        allowedDelCount -= segSizeDocs.delCount;
      }
    }
    final int mergeFactor = (int) Math.min(maxMergeAtOnce, segsPerTier);
    // Compute max allowed segments in the index
    double allowedSegCount = 0;
    // 计算allowedSegCount, 接下来会着重介绍。
    return doFindMerges(sortedInfos, maxMergedSegmentBytes, mergeFactor, (int) allowedSegCount, allowedDelCount, MERGE_TYPE.NATURAL,
        mergeContext, mergingBytes >= maxMergedSegmentBytes);
  }

```
该函数主要是为了过滤哪些segment是可以合并的:
1. 首先过滤掉正在合并的段, 不能继续参与合并。
2. 检查哪些段是不可以合并的, 需要同时满足一下两个条件:
+ 段必须大于index.merge.policy.max_merged_segment/2(默认2.5G)。
+该段中需要删除的文档比率要高于index.merge.policy.deletes_pct_allowed(默认33%), 或者所有段的删除率高于该阈值(33%)。
那么剩余的segment都有可能参与合并, 也就是说, 只要所有段删除率>33%, 或者该段删除率>33%, 哪怕该段>2.5G, 也是需要参与merge的。
3. 获取allowedSegCount, 本shard允许的最大semgent个数, 也可以理想为合并前最少段的个数, 若段的个数达不到这个数, 就不能触发合并。该参数的计算很好的体现了分级合并的思想:
```
    // 每一级别合并的个数
    final int mergeFactor = (int) Math.min(maxMergeAtOnce, segsPerTier);
    // Compute max allowed segments in the index
    // 每一级别合并的平均size
    long levelSize = Math.max(minSegmentBytes, floorSegmentBytes);
    long bytesLeft = totIndexBytes;
    double allowedSegCount = 0;
    // 一直在回归计算levelsize
    while (true) {
      // 计算若在levelSize大小的level上面, 最大多少个segment个数
      final double segCountLevel = bytesLeft / (double) levelSize;
      // 若该level不超过规定的该阶segment个数
      if (segCountLevel < segsPerTier || levelSize == maxMergedSegmentBytes) {
        // 最终获取此值，允许合并的segment个数
        allowedSegCount += Math.ceil(segCountLevel);
        break;
      }
      // 每次只增加该级别做多允许的semgent个数
      allowedSegCount += segsPerTier;
      // 理想情况下, 扣除该阶最大段个数, 剩余的size
      bytesLeft -= segsPerTier * levelSize;
      // 平均size增加一阶级, 比如从2M->20M
      levelSize = Math.min(maxMergedSegmentBytes, levelSize * mergeFactor);
 }
```
我理解这种思想是每次合并时,都会找size尽量相同的semgent合并。这样限制了semgent多少个时, 才不用继续合并了。
4. 通过doFindMerges来确定真正需要合并的段。
```
  private MergeSpecification doFindMerges(List<SegmentSizeAndDocs> sortedEligibleInfos,
                                          final long maxMergedSegmentBytes,
                                          final int mergeFactor, final int allowedSegCount,
                                          final int allowedDelCount, final MERGE_TYPE mergeType,
                                          MergeContext mergeContext,
                                          boolean maxMergeIsRunning) throws IOException {
    MergeSpecification spec = null;
    // The trigger point for total deleted documents in the index leads to a bunch of large segment
    // merges at the same time. So only put one large merge in the list of merges per cycle. We'll pick up another
    // merge next time around.  本轮是否已经选择了有大于5G的merge了, 若不控制, 可能会产生一堆大段合并, 引起IO问题
    boolean haveOneLargeMerge = false;
    // 一次选择多个merge
    while (true) {
      // 去掉已经被挑选出来合并的段
      Iterator<SegmentSizeAndDocs> iter = sortedEligible.iterator();
      while (iter.hasNext()) {
        SegmentSizeAndDocs segSizeDocs = iter.next();
        if (toBeMerged.contains(segSizeDocs.segInfo)) {
          iter.remove();
        }
      }
      // OK we are over budget -- find best merge!
      MergeScore bestScore = null;
      List<SegmentCommitInfo> best = null;
      boolean bestTooLarge = false;
      long bestMergeBytes = 0;
      // 每次固定第一位, 组成一个merge. 该merge只最大的那个段
      for (int startIdx = 0; startIdx < sortedEligible.size(); startIdx++) {
        // 该merge统计的所有segment合并后的size大小（若某个段删除比例大于33%，该段size并没有被包含进来）
        实际5G是和该值比较的
        long totAfterMergeBytes = 0;
        // 以startIdx起点, 选择的一组段
        final List<SegmentCommitInfo> candidate = new ArrayList<>();
        // 合并时,尝试过合并后大于5G的机会
        boolean hitTooLarge = false;
        // 该merge中所有段的大小=totAfterMergeBytes+大于5G的段（删除比例大于33%）
        long bytesThisMerge = 0;
        // 然后逐次选择, 只要选中的个数小于阈值mergeFactor & merge的总size>5G
        for (int idx = startIdx; idx < sortedEligible.size() && candidate.size() < mergeFactor && bytesThisMerge < maxMergedSegmentBytes; idx++) {
          final SegmentSizeAndDocs segSizeDocs = sortedEligible.get(idx);
          final long segBytes = segSizeDocs.sizeInBytes;
           // 若合并的大于阈值5G
          if (totAfterMergeBytes + segBytes > maxMergedSegmentBytes) {
            // 触过顶
            hitTooLarge = true;
            // 主要是为了解决单个segment内大于5G、删除比率大于33%的段，这一个段就可以直接合并
            if (candidate.size() == 0) {
              // We should never have something coming in that _cannot_ be merged, so handle singleton merges
              candidate.add(segSizeDocs.segInfo);
              bytesThisMerge += segBytes;
            }
            //若第i个加进来, size太大了, 那么拿第i+1个段来尝试下
            continue;
          }
          candidate.add(segSizeDocs.segInfo);
          bytesThisMerge += segBytes; // 该merge合并后形成的segment的size大小
          totAfterMergeBytes += segBytes;
        }
        // 到此为止，找到了一批可以合并的segment
        // If we didn't find a too-large merge and have a list of candidates
        // whose length is less than the merge factor, it means we are reaching
        // the tail of the list of segments and will only find smaller merges.
        // Stop here. 若没有达到足够大的size, 并且合并的段不够多，那么我们已经遍历了所有的段也没有合适的，放弃继续查找了
        if (bestScore != null &&  hitTooLarge == false && candidate.size() < mergeFactor) {
          break;
        }
        // 对该merge的这批segment进行打分,得分越低越好
        final MergeScore score = score(candidate, hitTooLarge, segInfosSizes);
        // (若是第一次选出 || 得分最低) && (正在合并的所有段size<5G || 当前merge没有达到阈值5G)
        if ((bestScore == null || score.getScore() < bestScore.getScore()) && (!hitTooLarge || !maxMergeIsRunning)) {
          best = candidate;
          bestScore = score;
          bestTooLarge = hitTooLarge; // 是否超过阈值5G(有删除率>33%)
          bestMergeBytes = totAfterMergeBytes;
        }
      }
      if (best == null) {
        return spec;
      }
      // The mergeType == FORCE_MERGE_DELETES behaves as the code does currently and can create a large number of
      // concurrent big merges. If we make findForcedDeletesMerges behave as findForcedMerges and cycle through
      // we should remove this.
      // 若已经有大于5G的段待合并了，本轮又选择了大的段合并了，必须避免该情况。否则将会产生一大推大段合并。
      if (haveOneLargeMerge == false || bestTooLarge == false || mergeType == MERGE_TYPE.FORCE_MERGE_DELETES) {
        haveOneLargeMerge |= bestTooLarge;
        if (spec == null) {
          spec = new MergeSpecification();
        }
        final OneMerge merge = new OneMerge(best);
        spec.add(merge);
        if (verbose(mergeContext)) {
          message("  add merge=" + segString(mergeContext, merge.segments) + " size=" + String.format(Locale.ROOT, "%.3f MB", bestMergeBytes / 1024. / 1024.) + " score=" + String.format(Locale.ROOT, "%.3f", bestScore.getScore()) + " " + bestScore.getExplanation() + (bestTooLarge ? " [max merge]" : ""), mergeContext);
        }
      }
      // whether we're going to return this list in the spec of not, we need to remove it from
      // consideration on the next loop.
      // 最终选择了一个合适的merge。
      toBeMerged.addAll(best);
    }
    // 再继续回归选择。
  }
```
该函数选择合适的segment组成merge过程如下, while循环, 查找一个merge的过程如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/es_merge1.png" height="350" width="400"/>
+ 从第一个segment(size最大), 查找到一组segment,总size<5G
+ 从第二个segment(size最大), 查找到一组segment,总size<5G
+ 直到查找了末尾D组, 由于size<5G && count(segment) < 10, 认为没有合适的,将丢弃改组。
每找到一组, 都会通过score()计算得分, 得分越低越好。 当前一轮仅仅会从这3组中选出一组出来进行merge。

我们接下来看下score()是如何计算A、B、C组segment的得分的:
```
  protected MergeScore score(List<SegmentCommitInfo> candidate, boolean hitTooLarge, Map<SegmentCommitInfo, SegmentSizeAndDocs> segmentsSizes) throws IOException {
    long totBeforeMergeBytes = 0;
    long totAfterMergeBytes = 0;
    long totAfterMergeBytesFloored = 0;
    for(SegmentCommitInfo info : candidate) {
      final long segBytes = segmentsSizes.get(info).sizeInBytes;
      totAfterMergeBytes += segBytes;
      totAfterMergeBytesFloored += floorSize(segBytes);
      totBeforeMergeBytes += info.sizeInBytes();
    }
    // Roughly measure "skew" of the merge, i.e. how
    // "balanced" the merge is (whether the segments are
    // about the same size), which can range from
    // 1.0/numSegsBeingMerged (good) to 1.0 (poor). Heavily
    // lopsided merges (skew near 1.0) is no good; it means
    // O(N^2) merge cost over time:
    final double skew;
    if (hitTooLarge) {// 如果达到了5G上限
      // Pretend the merge has perfect skew; skew doesn't
      // matter in this case because this merge will not
      // "cascade" and so it cannot lead to N^2 merge cost
      // over time: 鼓励优先合并到最大值的, 此后就不在继续合并, 否则合并后的段还需要合并。
      final int mergeFactor = (int) Math.min(maxMergeAtOnce, segsPerTier);
       // 斜率是最低的, 可以认为是最均衡的
      skew = 1.0/mergeFactor;
    } else {
    // 若组合数=10个，skew = max(组合最大的端)/sum(组合总共的段)，1/10<skew<1 ，段大小差异越大，skew值越大
      skew = ((double) floorSize(segmentsSizes.get(candidate.get(0)).sizeInBytes)) / totAfterMergeBytesFloored;
    }
    // Strongly favor merges with less skew (smaller
    // mergeScore is better):
    double mergeScore = skew;
    // Gently favor smaller merges over bigger ones.  We
    // don't want to make this exponent too large else we
    // can end up doing poor merges of small segments in
    // order to avoid the large merges:
    // 稍微倾斜合并小段，不想指数太大，否则将出现不会合并大段的情况
    mergeScore *= Math.pow(totAfterMergeBytes, 0.05);
    // Strongly favor merges that reclaim deletes:
    final double nonDelRatio = ((double) totAfterMergeBytes)/totBeforeMergeBytes;
    // 删除的越多，得分越小, 越是优先合并
    mergeScore *= Math.pow(nonDelRatio, 2);
    final double finalMergeScore = mergeScore;
    return new MergeScore() {
      @Override
      public double getScore() {
        return finalMergeScore;
      }
    };
  }
```
总结下, 针对每个merge打分, score越小越好:
+ 如果组合大小<10个，说明是总大小到达5G才中断的，这样skew = 1/10,倾斜度最小，原因：不会再出现递归合并，可以认为算是最均匀分布的一种; 若组合数=10个，skew = max(组合最大的端)/sum(组合总共的段)，1/10<skew<1 ，段大小差异越大，skew值越大
+ skew = skew*（抛出删除后总段的大小^0.05）:我们稍微倾向于合并小的端，原因：若源源不断的产生小段，我们则会忽略大段的合并。稍微的含义:在5G的大小内，后面系数<3。
+ mergescore = skew *[不需要删除的文档/(需要删除的文档+不需要删除的文档)]^2：说明merge的段中包含越多的被删除的段，score越小。

# merge过程
merge过程将进入ConcurrentMergeScheduler.merge()中:
```
  @Override
  public synchronized void merge(IndexWriter writer, MergeTrigger trigger, boolean newMergesFound) throws IOException {
    // 遍历直到将所有需要合并的merge全部进行了才退出
    while (true) {
      OneMerge merge = writer.getNextMerge();
      if (merge == null) {
        return;
      }
      boolean success = false;
      try {
        // 新产生一个线程去处理这个merge
        final MergeThread newMergeThread = getMergeThread(writer, merge);
        mergeThreads.add(newMergeThread);
        // 更新全局维护的段合并速度（看是否有合并慢的段）,同时初始化新创建的这个的merge速度
        updateIOThrottle(newMergeThread.merge, newMergeThread.rateLimiter);
        // merge线程开始进行工作
        newMergeThread.start();
        // 更新全局维持的段合并速度，来更新存在的每个的段合并速度
        updateMergeThreads();
        success = true;
      } finally {
        if (!success) {
          writer.mergeFinish(merge);
        }
      }
    }
  }
```
合并过程比较简单, 获取所有的merge并循环:
1. 通过updateIOThrottle更新全局维护的合并速度, 并初始化当前合并速度。
2. 启动合并线程。
3. 通过updateMergeThreads更新所有段的合并速度。
就这样不断循环, 产生新的合并线程。

## 更新全局维护的合并速度
```
  private synchronized void updateIOThrottle(OneMerge newMerge, MergeRateLimiter rateLimiter) throws IOException {
     // 自动限流关闭了
    if (doAutoIOThrottle == false) {
      return;
    }
    double mergeMB = bytesToMB(newMerge.estimatedMergeBytes);// 待合并的merge大小
    // 如果待合并的merge小于50M的话，不会对IO产生太大影响, 忽略。
    if (mergeMB < MIN_BIG_MERGE_MB) {
      return;
    }
    long now = System.nanoTime();
    // 如果发现有相似的merge，则说明当前level的merge落后了。
    boolean newBacklog = isBacklog(now, newMerge);
    // 检查存在的合并是否被阻塞了
    boolean curBacklog = false;
     // 新的没有阻塞
    if (newBacklog == false) {
      // 若正在merge的线程个数超过maxThreadCount限制
      if (mergeThreads.size() > maxThreadCount) {
        curBacklog = true;
      } else {
        // Now see if any still-running merges are backlog'd:
        // 存在的线程中有相同level的merge
        for (MergeThread mergeThread : mergeThreads) {
          if (isBacklog(now, mergeThread.merge)) {
            curBacklog = true;
            break;
          }
        }
      }
    }
    double curMBPerSec = targetMBPerSec; // 初始值为20M
    // 若新的是被阻塞了,那么整体提速20%, 加快合并速度
    if (newBacklog) {
      // 合并速度提速20%
      targetMBPerSec *= 1.20;
      // 5m-1024m之间变动
      if (targetMBPerSec > MAX_MERGE_MB_PER_SEC) {
        targetMBPerSec = MAX_MERGE_MB_PER_SEC;
      }
    } else if (curBacklog) {
     // 若旧的被阻塞了，则不变
    } else {
    // 若新旧合并都没有问题, 那么没必要维持该合并速度
      // We are not falling behind: decrease IO throttle by 10%
      targetMBPerSec /= 1.10; // 那么降速10%
      if (targetMBPerSec < MIN_MERGE_MB_PER_SEC) {
        targetMBPerSec = MIN_MERGE_MB_PER_SEC;
      }
    }
    double rate;
    if (newMerge.maxNumSegments != -1) {
      rate = forceMergeMBPerSec;
    } else {
      rate = targetMBPerSec;
    }
    rateLimiter.setMBPerSec(rate);
    targetMBPerSecChanged();
  }
```
该函数主要做了如下事情:
1. 若当前merge的size < 50MB, 将不对全局合并速度产生影响, 后面我们也可以看到size < 50MB的merge就没有限速。
2. 首先检查已经存在的merge size和即将合并的线程size是否在同一个level, 若有的话, 说明新的merge合并速度慢了。更新全局合并速度, 提速20%
3. 检查已经存在的merges, 该部分合并慢了有两种可能性:
+ 当前正在合并的线程超过阈值:maxThreadCount。
+ 正在合并的merges内部是否有相同level的merge。
若当前合并慢了, 不做任何处理。若不慢的话, 全局合并速度下降10%。

## 启动合并线程
```
public void run() {
      try {
        // merge线程才会真正去调用beforeMerge，aftermerge
        doMerge(writer, merge);
        // Let CMS run new merges if necessary:
        try { // 循环检查是否还有merge需要进行
          merge(writer, MergeTrigger.MERGE_FINISHED, true);
        }
      }  finally {
        synchronized(ConcurrentMergeScheduler.this) {
          removeMergeThread();
          updateMergeThreads();
          // In case we had stalled indexing, we can now wake up
          // and possibly unstall:
          ConcurrentMergeScheduler.this.notifyAll();
        }
      }
    }
```
该函数主要做了如下事情:
+ 通过调用doMerge()进行真正合并过程。
+ 调用merge()进行循环第二章节的操作。
+ 调用updateMergeThreads()调整当前正在进行的merge速度。
### 段合并
merge线程真正进行段合并在时ElasticsearchConcurrentMergeScheduler.doMerge()里面完成的:
```
  protected void doMerge(IndexWriter writer, MergePolicy.OneMerge merge) throws IOException { // 开始合并
        int totalNumDocs = merge.totalNumDocs();
        OnGoingMerge onGoingMerge = new OnGoingMerge(merge);
        onGoingMerges.add(onGoingMerge);
        try {
            beforeMerge(onGoingMerge); // 进行合并前的检查
            super.doMerge(writer, merge); // 真正调用lucene进行合并
        } finally {
            onGoingMerges.remove(onGoingMerge);
            afterMerge(onGoingMerge); // 去调用取消限速等
        }
    }

```
合并过程分为3个阶段:
1. 调用beforeMerge, 检查当前merge线程是否超过阈值maxMergeCount, 若超过了, 那么将对该shard的所有写入都限速为0。 我们经常看到`now throttling indexing: numMergesInFlight=`类似的提示, 就是这里打印的。
2. 调用super.doMerge()进行lucene层面merge。
3. 调用afterMerge, 查看是否需要解除该shard的写入限制。

## 更新所有段合并速度
```
  protected synchronized void updateMergeThreads() {

    final List<MergeThread> activeMerges = new ArrayList<>();

    int threadIdx = 0;
    while (threadIdx < mergeThreads.size()) {
      final MergeThread mergeThread = mergeThreads.get(threadIdx);
      if (!mergeThread.isAlive()) {
        // Prune any dead threads
        mergeThreads.remove(threadIdx);
        continue;
      }
      activeMerges.add(mergeThread);
      threadIdx++;
    } // 更新维持的merge线程情况，去掉已经完成的

    // Sort the merge threads, largest first:
    // segment最大的放最前面
    CollectionUtil.timSort(activeMerges);
    final int activeMergeCount = activeMerges.size();
    int bigMergeCount = 0;
    for (threadIdx=activeMergeCount-1;threadIdx>=0;threadIdx--) {
      MergeThread mergeThread = activeMerges.get(threadIdx);
      //只是统计大于50的段合并线程个数
      if (mergeThread.merge.estimatedMergeBytes > MIN_BIG_MERGE_MB*1024*1024) {
        bigMergeCount = 1+threadIdx;
        break;
      }
    }
    for (threadIdx=0;threadIdx<activeMergeCount;threadIdx++) { // 遍历全部的端合并线程
      MergeThread mergeThread = activeMerges.get(threadIdx);
      OneMerge merge = mergeThread.merge;
      // pause the thread if maxThreadCount is smaller than the number of merge threads.
      // 若最大的超过限制个数的最大的那个几个，直接暂停速度。小于50M的不限速，别的调整规定速度。
      final boolean doPause = threadIdx < bigMergeCount - maxThreadCount;
      double newMBPerSec;
      if (doPause) { // 大于的直接为0
        newMBPerSec = 0.0;
        // 是强制合并的速度
      } else if (merge.maxNumSegments != -1) {
        newMBPerSec = forceMergeMBPerSec;
      } else if (doAutoIOThrottle == false) { //
        newMBPerSec = Double.POSITIVE_INFINITY;
         // 若端的大小小于50M，则没有限制
      } else if (merge.estimatedMergeBytes < MIN_BIG_MERGE_MB*1024*1024) {
        // Don't rate limit small merges:
        newMBPerSec = Double.POSITIVE_INFINITY;
      } else {
      // 将全局的段合并速度用来更新本身的段合并速度
        newMBPerSec = targetMBPerSec;
      }
      MergeRateLimiter rateLimiter = mergeThread.rateLimiter;
      double curMBPerSec = rateLimiter.getMBPerSec();
      // 更新本身的段合并速度
      rateLimiter.setMBPerSec(newMBPerSec);
    }
  }
```
更新所有存在的merge合并速度逻辑如下:
1. 删除已经完成的merge线程, 并将存活的merge按照size从大到小进行排序,放入activeMerges中。
2. 统计size>50MB的合并线程。
3. 遍历activeMerges:
+ 对于超过规定阈值maxThreadCount的大于50MB的merge线程进行限速, 合并速度直接降为0。注意是从最大的size的merge开始遍历的。
+ 若设置了不限速, 则取消限速。
+ 若合并线程的size<50MB, 不进行限速。
+ 若合并的merge大于>50, 同时在规定的阈值maxThreadCount之内的线程, 调整合并速度为全局速度。

有两个参数比较重要, 需要注意下: maxMergeCount和maxThreadCount:
1. maxMergeCount主要是从写入的方面限制的, 应用在merge线程真正进行merge前, 检查当前正在合并的线程个数是否超过该阈值, 若超过了的话, 该shard上所有写入都将被禁止。
2. maxThreadCount主要是从限制merge合并速度的方向限制的, 用在两个地方:
+ 确定当前正在进行的merge是否超过该阈值, 超过后, 就认为当前已存在的merge合并速度慢了(若没超过,且正在合并的merge没有相同level的mege, 就会降10%合并速度)
+ 在更新全局合并速度时, 若大于50MB的合并线程数大于该阈值, 超过的merge都会被直接暂停合并。

# 总结
本文着重从merge调度层面讲解的, 比如如何选择segment, 如何动态调整merge速度的。至于lucene层面的合并, 将在之后的文档中详细介绍。从分析角度看, merge还是有些问题的, 比如只考虑单个shard的合并速度, 却没有在node层面进行限速。