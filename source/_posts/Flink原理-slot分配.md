---
title: Flink原理-Slot申请及Task部署
date: 2019-03-12 20:04:11
tags:
---
本文将从ExecutionGraph开始向后讲起, 关于JobManager是如何进行slot的分配及部署的, ExecutionGraph定义了TaskManager的并发分布结构, 作为任务执行的以后一层逻辑结构, 也是最核心数据结构。为了让大家有全局的了解, 先盗一张广为引用的Graph转换图:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/Flink_Graph.png" height="900" width="800"/>
具体来说, 本文讲述这些Task是如何部署到TaskManager上的。
# Task分配及部署
代码将从ExecutionGraph.scheduleExecutionGraph()开始讲解, 进入:
```
	public void scheduleForExecution() throws JobException {
		final long currentGlobalModVersion = globalModVersion;
		if (transitionState(JobStatus.CREATED, JobStatus.RUNNING)) {
			final CompletableFuture<Void> newSchedulingFuture;
			switch (scheduleMode) {
				case LAZY_FROM_SOURCES:
					newSchedulingFuture = scheduleLazy(slotProvider);
					break;
				case EAGER:
					newSchedulingFuture = scheduleEager(slotProvider, allocationTimeout);
					break;
				default:
					throw new JobException("Schedule mode is invalid.");
			}
		}
		...
	}
```
其中, scheduleMode分EAGER和LAZY_FROM_SOURCES, EAGER表示立刻去调度部署所有的Task。实际scheduleMode是从JobGraph.getScheduleMode()取值的, 为eager。</p>
我们再进入scheduleEager看是如何调度task的。
```
    private CompletableFuture<Void> scheduleEager(SlotProvider slotProvider, final Time timeout) {
	    ......
		 //都是每个JobGraph
		for (ExecutionJobVertex ejv : getVerticesTopologically()) {
			// these calls are not blocking, they only return futures
			Collection<CompletableFuture<Execution>> allocationFutures = ejv.allocateResourcesForAll(
				slotProvider, // SlotPool$ProviderAndOwner
				queued,
				LocationPreferenceConstraint.ALL,
				allPreviousAllocationIds,
				timeout);
			allAllocationFutures.addAll(allocationFutures);
		}
		//只有当所有Execution都分配到了槽位才继续进行部署
		final ConjunctFuture<Collection<Execution>> allAllocationsFuture = FutureUtils.combineAll(allAllocationFutures);
		final CompletableFuture<Void> currentSchedulingFuture = allAllocationsFuture
			.thenAccept(
				(Collection<Execution> executionsToDeploy) -> {
					for (Execution execution : executionsToDeploy) {
						try {
						   // 最后挨个调用execution.deploy()进行部署子task，部署的模式是发送命令到
						   execution.deploy();
						} catch (Throwable t) {
						   ......
						}
					}
				})
			......
		return currentSchedulingFuture;
	}`
```
scheduleEager主要做了两件事:
+ 通过allocateResourcesForAll确定每个subTask将要部署的slot。若没有合适的TaskManager, 那么通过yarn去申请TaskManager。
+ 当所有的subTask都确定好部署的TaskManager后, 通过execution.deploy()将subTask部署上去。
接下来, 将分别围绕这两件事讲解。
## 确定subTask部署的slot
通过getVerticesTopologically()获取所有的ExecutionJobVertex, 然后依次轮询给每个ExecutionJobVertex都分配一个slot, 其中轮询的ExecutionJobVertex是有先后顺序的, 从source开始确定, 直到sink。也就是说上游分配到哪个tm上, 会影响下游的slot分配。 我们进入allocateResourcesForAll看下是如何分配给一个ExecutionJobVertex所有的subTask分配slot的。
```
    public Collection<CompletableFuture<Execution>> allocateResourcesForAll(
			SlotProvider resourceProvider, // SlotPool$ProviderAndOwner
			boolean queued,  //true
			LocationPreferenceConstraint locationPreferenceConstraint, //ALL
			@Nonnull Set<AllocationID> allPreviousExecutionGraphAllocationIds,
			Time allocationTimeout) {
		final ExecutionVertex[] vertices = this.taskVertices;
		final CompletableFuture<Execution>[] slots = new CompletableFuture[vertices.length];
		for (int i = 0; i < vertices.length; i++) {
			final Execution exec = vertices[i].getCurrentExecutionAttempt();
			final CompletableFuture<Execution> allocationFuture = exec.allocateAndAssignSlotForExecution(
				resourceProvider,
				queued,
				locationPreferenceConstraint,
				allPreviousExecutionGraphAllocationIds,
				allocationTimeout);
			slots[i] = allocationFuture;
		}
		return Arrays.asList(slots);
	}
```
每一个ExecutionJobVertex都对应着一批ExecutionVertex(也就是subTask), 可以看到, 这里轮询每个ExecutionVertex进行分配。
```
    public CompletableFuture<Execution> allocateAndAssignSlotForExecution(
			SlotProvider slotProvider, //SlotPool$ProviderAndOwner
			boolean queued,
			LocationPreferenceConstraint locationPreferenceConstraint,  //ALL
			@Nonnull Set<AllocationID> allPreviousExecutionGraphAllocationIds,
			Time allocationTimeout) throws IllegalExecutionStateException {
		final SlotSharingGroup sharingGroup = vertex.getJobVertex().getSlotSharingGroup();
		if (transitionState(CREATED, SCHEDULED)) {
		    // 默认情况下, 所有的subTask的共享组均为default
			final SlotSharingGroupId slotSharingGroupId = sharingGroup != null ? sharingGroup.getSlotSharingGroupId() : null;
			lastAllocation != null ? Collections.singletonList(lastAllocation) : Collections.emptyList();
			/** 获取当前任务分配槽位所在节点的"偏好位置集合"，也就是分配时，优先考虑分配在这些节点上，一般是input节点所在节点 */
			// calculate the preferred locations
			final CompletableFuture<Collection<TaskManagerLocation>> preferredLocationsFuture = calculatePreferredLocations(locationPreferenceConstraint);
		    //上游子task地方全部确定了，才能继续确定下游子task位置
			final SlotRequestId slotRequestId = new SlotRequestId();
			final CompletableFuture<LogicalSlot> logicalSlotFuture = preferredLocationsFuture
				.thenCompose(
					(Collection<TaskManagerLocation> preferredLocations) ->
						slotProvider.allocateSlot(//SlotPool$ProviderAndOwner
							slotRequestId,
							toSchedule,
							queued,
							new SlotProfile(
								ResourceProfile.UNKNOWN,
								preferredLocations,
								previousAllocationIDs,
								allPreviousExecutionGraphAllocationIds),
							allocationTimeout));
			return logicalSlotFuture.thenApply(
				(LogicalSlot logicalSlot) -> {
				    //
					if (tryAssignResource(logicalSlot)) {
						return this;
					}
				});
		}
	}
```
该函数主要做了两件事情:
+ 在calculatePreferredLocations中确定该subTask对应的上游所有subTask的分配情况,


# SlotSharingGroup
Flink 允许相同SlotSharingGroup的subTask共享同一个slot, 好处主要有俩:
+ A Flink cluster needs exactly as many task slots as the highest parallelism used in the job. No need to calculate how many tasks (with varying parallelism) a program contains in total.
+ It is easier to get better resource utilization. Without slot sharing, the non-intensive source/map() subtasks would block as many resources as the resource intensive window subtasks. With slot sharing, increasing the base parallelism in our example from two to six yields full utilization of the slotted resources, while making sure that the heavy subtasks are fairly distributed among the TaskManagers.
默认情况下, SubTask使用相同的slot共享组: Default, task共享slot过程可以参考:<a href="https://ci.apache.org/projects/flink/flink-docs-release-1.8/concepts/runtime.html#task-slots-and-resources">如何共享slot</a>
这里将阐述SlotSharingGroup是如何生成并起作用的:
在JobGraph产生的时候调用setSlotSharingAndCoLocation()函数:
```
	private void setSlotSharingAndCoLocation() {
		final HashMap<String, SlotSharingGroup> slotSharingGroups = new HashMap<>();
		for (Entry<Integer, JobVertex> entry : jobVertices.entrySet()) {
			final StreamNode node = streamGraph.getStreamNode(entry.getKey());
			final JobVertex vertex = entry.getValue();// slotSharingGroupKey为默认值default
			final String slotSharingGroupKey = node.getSlotSharingGroup();
			final SlotSharingGroup sharingGroup;
			if (slotSharingGroupKey != null) {
			    // 可以看到, 所有的task的group都为default, 都将放入到同一个SlotSharingGroup中
				sharingGroup = slotSharingGroups.computeIfAbsent(slotSharingGroupKey, (k) -> new SlotSharingGroup());
				vertex.setSlotSharingGroup(sharingGroup);
			} else {
				sharingGroup = null;
			}
```
这样, 所有的JobVertex都引用了同一个SlotSharingGroup。 而
```
    public void setSlotSharingGroup(SlotSharingGroup grp) {
		if (this.slotSharingGroup != null) {
			this.slotSharingGroup.removeVertexFromGroup(id);
		}
		this.slotSharingGroup = grp;
		if (grp != null) {
			grp.addVertexToGroup(id);
		}
	}
```
# 参考文档
https://ci.apache.org/projects/flink/flink-docs-release-1.8/concepts/runtime.html
https://ci.apache.org/projects/flink/flink-docs-release-1.8/internals/job_scheduling.html
http://wuchong.me/blog/2016/05/03/flink-internals-overview/
http://wuchong.me/blog/2016/05/09/flink-internals-understanding-execution-resources/