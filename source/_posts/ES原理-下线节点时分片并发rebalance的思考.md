---
title: ES原理-下线节点时分片并发rebalance的思考
date: 2019-06-16 22:29:49
tags:
toc: true
---
# 现象
最近在下线ES5.6.8集群节点时候, 发现了一个很奇怪的现象, 我先把cluster参数给贴出来看下:
```
PUT _cluster/settings
{
    "transient": {
        "cluster": {
         "routing": {
            "rebalance": {
               "enable": "all"
            },
            "allocation": {
               "allow_rebalance": "always",
               "cluster_concurrent_rebalance": "1",
               "node_concurrent_recoveries": "1",
               "node_initial_primaries_recoveries": "1",
               "enable": "all"
            }
         }
      }
    }
}
```
然后我通过如下命令下线4个节点:
```
PUT _cluster/settings
{
    "transient": {
         "cluster.routing.allocation.exclude._ip": "ip1,ip2,ip3,ip4"
    }
}
```
发现此时集群上这4个节点分别有一个分片开始进行move, what for? 按我们的理解, move操作的并发应该被我们通过参数`cluster_concurrent_rebalance`控制为1, 可是为啥为4。好像这个下线节点时候shard的并发与下线节点个数一致, 而不是受我们控制。

# 源码查看
带着上面的疑问, 试图从代码中找到原因, 我们知道, 任何集群元数据变动都会跳到BalancedShardsAllocator.allocate():
```
    public void allocate(RoutingAllocation allocation) {
        if (allocation.routingNodes().size() == 0) {
            /* with no nodes this is pointless */
            return;
        }
        final Balancer balancer = new Balancer(logger, allocation, weightFunction, threshold);
        balancer.allocateUnassigned();
        balancer.moveShards();
        balancer.balance();
    }
```
这里操作都是依据分片分配到各个节点的打分为依据来操作, 主要分为三步: 分配unassigned分片, move分片, 均衡分片。 在开始的现象, 明显不是第一步, 只可能是move分片、或者均衡分片。
## move分片
我们首先进入move分片看下能否解释现象:
```
        public void moveShards() {
            // Iterate over the started shards interleaving between nodes, and check if they can remain. In the presence of throttling
            // shard movements, the goal of this iteration order is to achieve a fairer movement of shards from the nodes that are
            // offloading the shards.
            //实际上，每次moveShards只会从每个node中选择一个shard迁移
            for (Iterator<ShardRouting> it = allocation.routingNodes().nodeInterleavedShardIterator(); it.hasNext(); ) {
                ShardRouting shardRouting = it.next();
                final MoveDecision moveDecision = decideMove(shardRouting);
                if (moveDecision.isDecisionTaken() && moveDecision.forceMove()) {
                    final ModelNode sourceNode = nodes.get(shardRouting.currentNodeId());
                    final ModelNode targetNode = nodes.get(moveDecision.getTargetNode().getId());
                    sourceNode.removeShard(shardRouting);
                    Tuple<ShardRouting, ShardRouting> relocatingShards = routingNodes.relocateShard(shardRouting, targetNode.getNodeId(),
                        allocation.clusterInfo().getShardSize(shardRouting, ShardRouting.UNAVAILABLE_EXPECTED_SHARD_SIZE), allocation.changes());
                    targetNode.addShard(relocatingShards.v2());
                    if (logger.isTraceEnabled()) {
                        logger.trace("Moved shard [{}] to node [{}]", shardRouting, targetNode.getRoutingNode());
                    }
                } else if (moveDecision.isDecisionTaken() && moveDecision.canRemain() == false) {
                    logger.trace("[{}][{}] can't move", shardRouting.index(), shardRouting.id());
                }
            }
        }
```
这里我们需要注意一个事实: 这里轮训所有分片是有顺序的, 依次从每个节点上选择一个分片判断, 首先第一轮: 选择第一个节点上的第一个分片, 然后第二个节点上的第一个分片..., 最后一个节点的第一个分片, 再开始第二轮: 第一个节点上第二个分片......。
然后再进入decideMove看下具体的move的逻辑:
```
        /**
         * Makes a decision on whether to move a started shard to another node.  The following rules apply
         * to the {@link MoveDecision} return object:
         *   1. If the shard is not started, no decision will be taken and {@link MoveDecision#isDecisionTaken()} will return false.
         *   2. If the shard is allowed to remain on its current node, no attempt will be made to move the shard and
         *      {@link MoveDecision#canRemainDecision} will have a decision type of YES.  All other fields in the object will be null.
         *   3. If the shard is not allowed to remain on its current node, then {@link MoveDecision#getAllocationDecision()} will be
         *      populated with the decision of moving to another node.  If {@link MoveDecision#forceMove()} ()} returns {@code true}, then
         *      {@link MoveDecision#targetNode} will return a non-null value, otherwise the assignedNodeId will be null.
         *   4. If the method is invoked in explain mode (e.g. from the cluster allocation explain APIs), then
         *      {@link MoveDecision#nodeDecisions} will have a non-null value.
         */
        public MoveDecision decideMove(final ShardRouting shardRouting) {
            if (shardRouting.started() == false) {
                // we can only move started shards
                return MoveDecision.NOT_TAKEN;
            }
            final boolean explain = allocation.debugDecision();
            final ModelNode sourceNode = nodes.get(shardRouting.currentNodeId());
            assert sourceNode != null && sourceNode.containsShard(shardRouting);
            RoutingNode routingNode = sourceNode.getRoutingNode();
            Decision canRemain = allocation.deciders().canRemain(shardRouting, routingNode, allocation);  //确定该节点是否还能存储在当前节点上
            if (canRemain.type() != Decision.Type.NO) {
                return MoveDecision.stay(canRemain);
            }
            sorter.reset(shardRouting.getIndexName());
            /*
             * the sorter holds the minimum weight node first for the shards index.
             * We now walk through the nodes until we find a node to allocate the shard.
             * This is not guaranteed to be balanced after this operation we still try best effort to
             * allocate on the minimal eligible node.
             */
            Type bestDecision = Type.NO;
            RoutingNode targetNode = null;
            final List<NodeAllocationResult> nodeExplanationMap = explain ? new ArrayList<>() : null;
            int weightRanking = 0;
            for (ModelNode currentNode : sorter.modelNodes) {
                if (currentNode != sourceNode) {
                    RoutingNode target = currentNode.getRoutingNode();
                    // don't use canRebalance as we want hard filtering rules to apply. See #17698
                    // 查看当前分片
                    Decision allocationDecision = allocation.deciders().canAllocate(shardRouting, target, allocation);
                    if (explain) {
                        nodeExplanationMap.add(new NodeAllocationResult(
                            currentNode.getRoutingNode().node(), allocationDecision, ++weightRanking));
                    }
                    // TODO maybe we can respect throttling here too?
                    if (allocationDecision.type().higherThan(bestDecision)) {
                        bestDecision = allocationDecision.type();
                        if (bestDecision == Type.YES) {
                            targetNode = target;
                            if (explain == false) {
                                // we are not in explain mode and already have a YES decision on the best weighted node,
                                // no need to continue iterating
                                break;
                            }
                        }
                    }
                }
            }
            return MoveDecision.cannotRemain(canRemain, AllocationDecision.fromDecisionType(bestDecision),
                targetNode != null ? targetNode.node() : null, nodeExplanationMap);
        }
```
该函数主要做了如下逻辑:
+ 若该分片不处于Started, 那么将不进行move。
+ 首先通过canRemain检查该分片是否还可以继续存留在该节点上, 主要由以下几个因素决定:
1. awareness(机房)
2. 磁盘空间
3. ip排除(就是本次修改的参数)
4. 每个节点上最多分配个数(total_shards_per_node)。
很显然, 本次过程中ip排除了,那么该分片不能再待在本节点上了。
+ 那么在所有节点上找一个节点, 该分片可以迁移上去。找的依据是canAllocate, canAllocate决定因素并没有包含cluster_concurrent_rebalance, 但是增加了node_concurrent_outgoing_recoveries、node_concurrent_incoming_recoveries。这两个参数的含义是: 决定分片是否可以分配到某个节点上的依据是该节点上正在迁入/迁出的分片是否达到阈值, 若没有达到阈值, 就可以分配。
`重点:` 这就解释了为啥通过调整_ip时, 为啥不能通过cluster_concurrent_rebalance控制rebalance的并发了, ip调整时并发是通过迁入/迁出并发控制的, 也就是我们最开始观察到的现象, 正因为我们设置了node_concurrent_recoveries, 导致每个分片上迁出并发只能是1, 那么设置exclude.ip为4个时, 我们在前端看到有4个分片正处于rebalance。这里完全都没想过使用cluster_concurrent_rebalance来控制迁移并发的。这里不用cluster_concurrent_rebalance, 官方也给出了<a href="https://github.com/elastic/elasticsearch/pull/17698">原因</a>:
```
#14259 added a check to honor rebalancing policies (i.e., rebalance only on green state) when moving shards due to changes in allocation filtering rules. The rebalancing policy is there to make sure that we don't try to even out the number of shards per node when we are still missing shards. However, it should not interfere with explicit user commands (allocation filtering) or things like the disk threshold wanting to move shards because of a node hitting the high water mark.
#14259 was done to address #14057 where people reported that using allocation filtering caused many shards to be moved at once. This is however a none issue - with 1.7 (where the issue was reported) and 2.x, we protect recovery source nodes by limitting the number of concurrent data streams they can open (i.e., we can have many recoveries, but they will be throttled). In 5.0 we came up with a simpler and more understandable approach where we have a hard limit on the number of outgoing recoveries per node (on top of the incoming recoveries we already had).
```
大致就是说, rebalance策略主要是为了解决的是: 当我们未对全局分片有足够了解的时候(当全局分片并未处于完全均衡的时候), 我们并不会去干扰每个节点的分片个数。当然, rebalance更不应该去干扰显示的用户命令比如分片排除, 或者达到磁盘阈值这样的情况。这里并发是通过Incoming/OutComing这样的并发去控制的。原因清楚了吧。
+ 找到一个可以迁移到分片后, 然后通过routingNodes.relocateShard修改relocatingShards值, 这个值就是我们在前端看到的正在rebalance的个数。

## 均衡分片
既然现象是由move分片来解释了, 那么我们也来了解均衡分片大致做了那些事情呢?
+ 通过allocation.deciders().canRebalance(allocation).type() 首先检查是否可以进行rebalance。通过ClusterRebalanceType来控制:
1. 若设置为ALWAYS, 那么是可以进行rebalance的。
2. 若设置为INDICES_PRIMARIES_ACTIVE, 那么当只有所有主分片处于active的时候才可以进行rebalance的。
3. 若设置为INDICES_ALL_ACTIVE, 那么是禁止rebalance的。
+ 若可以进行rebalance, 然后进入balanceByWeights()。
```
        private void balanceByWeights() {
            final AllocationDeciders deciders = allocation.deciders();
            final ModelNode[] modelNodes = sorter.modelNodes;
            final float[] weights = sorter.weights;
            //轮训所有索引
            for (String index : buildWeightOrderedIndices()) {
                IndexMetaData indexMetaData = metaData.index(index);
                // find nodes that have a shard of this index or where shards of this index are allowed to be allocated to,
                // move these nodes to the front of modelNodes so that we can only balance based on these nodes
                // 首先选择与该索引有关的节点, 这些节点, 那么该索引有分配在上面分配, 要么该索引可以分配到该节点上。默认情况下, 除了exclude外, 所有节点都是相关的。
                int relevantNodes = 0;
                for (int i = 0; i < modelNodes.length; i++) {
                    ModelNode modelNode = modelNodes[i];
                    if (modelNode.getIndex(index) != null
                        || deciders.canAllocate(indexMetaData, modelNode.getRoutingNode(), allocation).type() != Type.NO) {
                        // swap nodes at position i and relevantNodes
                        modelNodes[i] = modelNodes[relevantNodes];
                        modelNodes[relevantNodes] = modelNode;
                        relevantNodes++;
                    }
                }
                // 若相关节点少于2个,就没有rebalance的必要了
                if (relevantNodes < 2) {
                    continue;
                }
                // 对相关节点与该索引之间进行打分, 分值从最小到最大进行排序(分值越大说明分配越不合理)
                sorter.reset(index, 0, relevantNodes);
                int lowIdx = 0;
                int highIdx = relevantNodes - 1;
                while (true) {
                    final ModelNode minNode = modelNodes[lowIdx];
                    final ModelNode maxNode = modelNodes[highIdx];
                    advance_range:
                    // 假如最大分值的节点, 该索引有分片存在
                    if (maxNode.numShards(index) > 0) {
                        //计算最大和最小分值差大于阈值
                        final float delta = absDelta(weights[lowIdx], weights[highIdx]);
                        //若差值小于阈值1
                        if (lessThan(delta, threshold)) {

                            if (lowIdx > 0 && highIdx-1 > 0 // is there a chance for a higher delta?
                                && (absDelta(weights[0], weights[highIdx-1]) > threshold) // check if we need to break at all
                                ) {  //low和high差距大于阈值，那么还是可以继续找的
                                /* This is a special case if allocations from the "heaviest" to the "lighter" nodes is not possible
                                 * due to some allocation decider restrictions like zone awareness. if one zone has for instance
                                 * less nodes than another zone. so one zone is horribly overloaded from a balanced perspective but we
                                 * can't move to the "lighter" shards since otherwise the zone would go over capacity.
                                 *
                                 * This break jumps straight to the condition below were we start moving from the high index towards
                                 * the low index to shrink the window we are considering for balance from the other direction.
                                 * (check shrinking the window from MAX to MIN)
                                 * See #3580
                                 */
                                break advance_range;
                            }
                            if (logger.isTraceEnabled()) {
                                logger.trace("Stop balancing index [{}]  min_node [{}] weight: [{}]  max_node [{}] weight: [{}]  delta: [{}]",
                                        index, maxNode.getNodeId(), weights[highIdx], minNode.getNodeId(), weights[lowIdx], delta);
                            }
                            break; //low和high差距小于阈值，那么完全不用找了。直接退出当前索引的rebalance，进行下一个索引的rebalance
                        }
                        if (logger.isTraceEnabled()) {
                            logger.trace("Balancing from node [{}] weight: [{}] to node [{}] weight: [{}]  delta: [{}]",
                                    maxNode.getNodeId(), weights[highIdx], minNode.getNodeId(), weights[lowIdx], delta);
                        }
                        /* pass the delta to the replication function to prevent relocations that only swap the weights of the two nodes.
                         * a relocation must bring us closer to the balance if we only achieve the same delta the relocation is useless */
                        if (tryRelocateShard(minNode, maxNode, index, delta)) {
                            /*
                             * TODO we could be a bit smarter here, we don't need to fully sort necessarily
                             * we could just find the place to insert linearly but the win might be minor
                             * compared to the added complexity
                             */
                            weights[lowIdx] = sorter.weight(modelNodes[lowIdx]);
                            weights[highIdx] = sorter.weight(modelNodes[highIdx]);
                            sorter.sort(0, relevantNodes);
                            lowIdx = 0;
                            highIdx = relevantNodes - 1;
                            // 再继续查找当前索引在别的节点上是都有更合适的分配。
                            continue;
                        }
                    }
                    // 若没有分配, 则开始low+1 进行第二轮重新开始。
                    //第一轮：首先第一轮从high不变， low每次增加1，向higt靠近，直到low和high一样， 第二轮：然后high--， 继续low向higt靠近；再第三轮，这样实际循环次数是(high-low)(high-low)/2, 很像一个倒着的乘法表
                    if (lowIdx < highIdx - 1) {
                        /* Shrinking the window from MIN to MAX
                         * we can't move from any shard from the min node lets move on to the next node
                         * and see if the threshold still holds. We either don't have any shard of this
                         * index on this node of allocation deciders prevent any relocation.*/
                        lowIdx++;
                    } else if (lowIdx > 0) {
                        /* Shrinking the window from MAX to MIN
                         * now we go max to min since obviously we can't move anything to the max node
                         * lets pick the next highest */
                        lowIdx = 0;
                        highIdx--;
                    } else {
                        /* we are done here, we either can't relocate anymore or we are balanced */
                        break;
                    }
                }
            }
        }
```
rebalance时, 以index来循环, 大概逻辑就是:
1. 针对每个索引在每个节点上分配进行一个打分。打分依据是(该索引是否在所有节点是否分配均衡&&所有shard是否在所有节点分配均衡), 分支越低, 说明分配越合理。
2. 从分值高低差的阈值来判断是否需要rebalance。阈值为1。

# 总结
若仅仅增加节点, 那么就是rebalance操作, 此时cluster_concurrent_rebalance是生效的。 若做了exclude操作, 那么就变成了move操作, 此时node_concurrent_recoveries就该生效了。 move, rebalance, allocation等操作都是由多个决策器一起决定如何分配的, 只要合理使用各种决策器, 那么分片分配就能被我们合理的掌握了。
