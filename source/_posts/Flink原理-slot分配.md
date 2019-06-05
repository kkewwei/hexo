---
title: Flink原理-SubTask申请slot及部署
date: 2019-03-12 20:04:11
tags: Flink、Slot分配、SubTask申请slot, SubTask部署
toc: true
---
本文将从ExecutionGraph开始向后讲起, ExecutionGraph定义了Job的并发逻辑结构, 作为任务执行的以后一层逻辑结构, 也是最核心数据结构。为了让大家有全局的了解, 先盗一张广为引用的Graph转换图:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/Flink_Graph.png" height="900" width="800"/>
具体来说, 本文讲述subTask申请slot以及, 部署到TaskManager上的过程。
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
其中, scheduleMode分EAGER和LAZY_FROM_SOURCES, EAGER表示立刻去调度部署所有的Task。实际scheduleMode是从JobGraph.getScheduleMode()取值的, 为eager。
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
+ 当所有的subTask都确定好部署TaskManager的slot后, 通过execution.deploy()将subTask部署上去。
接下来, 将分别围绕这两件事讲解。
## 确定subTask分配的slot
通过getVerticesTopologically()获取所有的ExecutionJobVertex, 然后依次轮询给每个ExecutionJobVertex都分配一个slot, 其中轮询的ExecutionJobVertex是有先后顺序的, 从source开始分配slot, 直到sink。后面可以看到, 上游分配到哪个tm上, 会影响下游的slot分配。 我们进入allocateResourcesForAll看下是如何给一个ExecutionJobVertex所有的subTask分配slot的。
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
每一个ExecutionJobVertex都对应着一批ExecutionVertex(也就是subTask), 可以看到, 这里轮询每个ExecutionVertex进行申请一个slot。
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
			/** 获取subTask即将分配的"偏好位置集合"，也就是分配时，优先考虑分配在这些节点上，一般是input节点所在节点 */
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
+ 在calculatePreferredLocations中确定从该subTask对应ExecutionJobVertex的所有上游中找到最合适的上游"偏向位置集合"。
+ 通过SlotPool$ProviderAndOwner.allocateSlot继续确定从"偏向位置集合"找到一个共享slot。
我们知道, 多个subTask允许共享slot, 细节后面会详细描述。 那么当前subTask与哪些已经分配的subTask共享slot呢? 下游subTask与哪个上游subTask共享slot呢?
<img src="https://kkewwei.github.io/elasticsearch_learning/img/Flink_slot_allocate.png" height="300" width="350"/>

flink会根据subTask上游slot的分配来确定当前slot的分配:
```
public Collection<CompletableFuture<TaskManagerLocation>> getPreferredLocationsBasedOnInputs() {
		// 如果没有输入，则返回空集合，否则，基于上游分布确定偏好位置
		if (inputEdges == null) {
			return Collections.emptySet();
		}
		else {
			Set<CompletableFuture<TaskManagerLocation>> locations = new HashSet<>(getTotalNumberOfParallelSubtasks());
			Set<CompletableFuture<TaskManagerLocation>> inputLocations = new HashSet<>(getTotalNumberOfParallelSubtasks());
			// go over all inputs
			for (int i = 0; i < inputEdges.length; i++) {
				inputLocations.clear();
				ExecutionEdge[] sources = inputEdges[i];
				if (sources != null) {
					for (int k = 0; k < sources.length; k++) {
						// 获取当前source源所属的taskManager位置
						CompletableFuture<TaskManagerLocation> locationFuture = sources[k].getSource().getProducer().getCurrentTaskManagerLocationFuture();
						// add input location
						inputLocations.add(locationFuture);
						// inputs which have too many distinct sources are not considered
						// 如果某个输入源有太多的节点分布，则不考虑这个输入源的节点位置了
						if (inputLocations.size() > MAX_DISTINCT_LOCATIONS_TO_CONSIDER) {
							inputLocations.clear();
							break;
						}
					}
				}// 保留具有最少分布位置的输入的位置
				// keep the locations of the input with the least preferred locations
				if (locations.isEmpty() || // nothing assigned yet  第一个source
				        // 找到上游节点所处tm最少的的那个上游
						(!inputLocations.isEmpty() && inputLocations.size() < locations.size())) {
					locations.clear();
					locations.addAll(inputLocations);
				}
			}
			return locations.isEmpty() ? Collections.emptyList() : locations;
		}
	}
```
处理过程如下:
+ 若该ExecutionVertex没有上游(例如source), 那么返回为空, 没有"偏好位置集合", 之后将申请新的slot。
+ 若当前ExecutionVertex有属于不同JobVertex多个ExecutionJobVertex的上游, 那么当前sub分配到哪些共享slot的可选路径只能是: 属于同一个JobVertex的上游节点个数最少。上图的话, 就会选择source2的所有subTask作为"偏好位置集合"。我们接下来看第二步, 最终会进入到allocateSharedSlot决定subTask分配到哪些"偏好位置集合"里slot上。
```
private CompletableFuture<LogicalSlot> allocateSharedSlot(
		SlotRequestId slotRequestId,
		ScheduledUnit task,
		SlotProfile slotProfile,
		boolean allowQueuedScheduling,
		Time allocationTimeout) {
		// allocate slot with slot sharing
		final SlotSharingManager multiTaskSlotManager = slotSharingManagers.computeIfAbsent(
			 //默认都是一个, default对应的那个
			task.getSlotSharingGroupId(),
			id -> new SlotSharingManager(
				id,
				this,
				providerAndOwner));
		final SlotSharingManager.MultiTaskSlotLocality multiTaskSlotLocality;
		try {
			if (task.getCoLocationConstraint() != null) {
				multiTaskSlotLocality = allocateCoLocatedMultiTaskSlot(
					task.getCoLocationConstraint(),
					multiTaskSlotManager,
					slotProfile,
					allowQueuedScheduling,
					allocationTimeout);
			} else {
			     // 跑到这里
				multiTaskSlotLocality = allocateMultiTaskSlot(
					task.getJobVertexId(),
					multiTaskSlotManager,
					slotProfile,
					allowQueuedScheduling,
					allocationTimeout);
			}
		}
		final SlotSharingManager.SingleTaskSlot leaf = multiTaskSlotLocality.getMultiTaskSlot().allocateSingleTaskSlot(
			slotRequestId,
			task.getJobVertexId(),
			multiTaskSlotLocality.getLocality());
		return leaf.getLogicalSlotFuture();
	}
```
该函数主要做了两件事:
1. 通过allocateMultiTaskSlot产生MultiTaskSlotLocality, 里面包含从"偏向位置集合"中选取的部署当前subTask共享的slot。
2. 产生SingleTaskSlot, 当前SingleTaskSlot作为MultiTaskSlot的一个子叶子节点。
再继续跟进代码前, 我们需要了解两个变量resolvedRootSlots、unresolvedRootSlots。共享slot都会从这两个变量中获取, 这两个变量为共享组所拥有, 默认共享组为default。
unresolvedRootSlots: 当当前subTask正在确认部署到那个slot中时, 会将该slot保存在unresolvedRootSlots; 当确定好部署到哪个slot时, 会将该信息从unresolvedRootSlots中移除, 并放入resolvedRootSlots中  当我们查找是否有可利用的slot时, 会从这些变量中查找。
我们再进入正题, 看allocateMultiTaskSlot看是如何给subTask分配slot的:
```
	private SlotSharingManager.MultiTaskSlotLocality allocateMultiTaskSlot(
			//groupId指的同一个JobVertex的id
			AbstractID groupId,
			SlotSharingManager slotSharingManager,
			SlotProfile slotProfile,
			boolean allowQueuedScheduling,
			Time allocationTimeout) throws NoResourceAvailableException {
		//过滤"偏好位置集合"的位置中不属于相同groupId的位置, 这里主要是为了避免同一个ExecutionJobVertex中不同的SubTask分配到同一个slot中。
		// check first whether we have a resolved root slot which we can use
		SlotSharingManager.MultiTaskSlotLocality multiTaskSlotLocality = slotSharingManager.getResolvedRootSlot(
			groupId,
			//  LocationPreferenceSchedulingStrategy
			schedulingStrategy,
			slotProfile);
	    //从"偏好位置集合"中到合适的slot后, 就直接返回了。
		if (multiTaskSlotLocality != null && multiTaskSlotLocality.getLocality() == Locality.LOCAL) {
			return multiTaskSlotLocality;
		}
		......
		if (allowQueuedScheduling) {
			//在unresolvedRootSlots中查找不属于同一个JobVertex的slot
			SlotSharingManager.MultiTaskSlot multiTaskSlotFuture = slotSharingManager.getUnresolvedRootSlot(groupId);  // 为null
			if (multiTaskSlotFuture == null) {
				//没有找到合适的可利用的的slot, 那么将去向ResurceNameger申请新的TaskManger, 这是最后一步
				final CompletableFuture<AllocatedSlot> futureSlot = requestNewAllocatedSlot(
					allocatedSlotRequestId,
					slotProfile.getResourceProfile(),
					allocationTimeout); //300s
				//将新产生的futureSlot, 放入resolvedRootSlots中, 这样之后申请slot时, 该slot可以被共享。
				multiTaskSlotFuture = slotSharingManager.createRootSlot(
					multiTaskSlotRequestId,
					futureSlot,
					allocatedSlotRequestId);
				futureSlot.whenComplete(
					(AllocatedSlot allocatedSlot, Throwable throwable) -> {
						final SlotSharingManager.TaskSlot taskSlot = slotSharingManager.getTaskSlot(multiTaskSlotRequestId);
						if (taskSlot != null) {
							// still valid
							if (!(taskSlot instanceof SlotSharingManager.MultiTaskSlot) || throwable != null) {
								taskSlot.release(throwable);
							} else {
								if (!allocatedSlot.tryAssignPayload(((SlotSharingManager.MultiTaskSlot) taskSlot))) {
									taskSlot.release(new FlinkException("Could not assign payload to allocated slot " +
										allocatedSlot.getAllocationId() + '.'));
								}
							}
						}
					});
			}
			return SlotSharingManager.MultiTaskSlotLocality.of(multiTaskSlotFuture, Locality.UNKNOWN);
		}
	}
```
该函数主要逻辑如下:
+ 从resolvedRootSlots、unresolvedRootSlots中查找是否有可共享的slot。
+ 若没有, 向ResourceManager申请TaskManager以获取slot。
+ 将申请的slot信息也存放入unresolvedRootSlots中, 等成功申请后再存放入resolvedRootSlots。
我们再接着看是如何向ResourceManager申请TaskManager的。
```
	private CompletableFuture<AllocatedSlot> requestNewAllocatedSlot(
			SlotRequestId slotRequestId,
			ResourceProfile resourceProfile,
			Time allocationTimeout) {
		final PendingRequest pendingRequest = new PendingRequest(
			slotRequestId,
			resourceProfile);
		// register request timeout
		FutureUtils  //30s超时
			.orTimeout(pendingRequest.getAllocatedSlotFuture(), allocationTimeout.toMilliseconds(), TimeUnit.MILLISECONDS)
			.whenCompleteAsync(  //当结束完成时需要做的事情
				(AllocatedSlot ignored, Throwable throwable) -> {
					if (throwable instanceof TimeoutException) {
						timeoutPendingSlotRequest(slotRequestId);
					}
				},
				getMainThreadExecutor());
		if (resourceManagerGateway == null) {  // 为null
			stashRequestWaitingForResourceManager(pendingRequest);  // 会跑到这里
		} else {
			requestSlotFromResourceManager(resourceManagerGateway, pendingRequest);
		}
		return pendingRequest.getAllocatedSlotFuture();
	}
```
可以看到:
+ 首先查看resourceManagerGateway是否连接上, 若没有连接上, 将请求暂时缓存起来, 待连接上之后再申请。
+ 若已经初始化之后, 会去向ResourceManager申请TaskManager。
### 缓存申请Slot的请求
大致思路是先缓存申请slot的请求, 直到flink ResourceManager注册完成后, 再去申请, 我们看下整体细节。首先去查看哪里开始对resourceManagerGateway进行初始化的。 首先回到最开始准备执行ExecutionGraph的时候:
```
     private Acknowledge startJobExecution(JobMasterId newJobMasterId) throws Exception {
		 //这里会开始尝试连接rm， 会去和resourceManager建立联系
		startJobMasterServices();
		resetAndScheduleExecutionGraph();
     }
     private void startJobMasterServices() throws Exception {
		slotPool.start(getFencingToken(), getAddress());
		// 这里比较重要，会进去启动申请tm的请求
		reconnectToResourceManager(new FlinkException("Starting JobMaster component."));
		// StandaloneLeaderRetrievalService
		resourceManagerLeaderRetriever.start(new ResourceManagerLeaderListener());
	}
```
在reconnectToResourceManager中, 会去尝试初始化, 调用connectToResourceManager:
```
private void connectToResourceManager() {
		resourceManagerConnection = new ResourceManagerConnection( //很重要
			log,
			jobGraph.getJobID(),
			resourceId,
			getAddress(),
			getFencingToken(),
			resourceManagerAddress.getAddress(),
			resourceManagerAddress.getResourceManagerId(),
			scheduledExecutorService);
		resourceManagerConnection.start();  //从这里进去，获取注册ResourceManger
	}
```
在ResourceManagerConnection中定义了onRegistrationSuccess, 会去调用establishResourceManagerConnection()函数, 我们进入resourceManagerConnection.start()看下如何建立注册的。
```
    public void start() {
        // ResourceManagerConnection
        final RetryingRegistration<F, G, S> newRegistrationn = createNewRegistration();
		if (REGISTRATION_UPDATER.compareAndSet(this, null, newRegistration)) {
		    //会从这里进去，很重要，比如注册ResourceManager
			newRegistration.startRegistration();
		}
	}
```
在createNewRegistration中, 新建注册:
```
	private RetryingRegistration<F, G, S> createNewRegistration() {
		RetryingRegistration<F, G, S> newRegistration = checkNotNull(generateRegistration());  //跑进去
		CompletableFuture<Tuple2<G, S>> future = newRegistration.getFuture();
		future.whenCompleteAsync(
			(Tuple2<G, S> result, Throwable failure) -> {
				if (failure != null) {
				......
				} else {
					targetGateway = result.f0;
					//注意进来，进行pending task分配，调用ResourceManagerConnection.onRegistrationSuccess()
					onRegistrationSuccess(result.f1);
				}
			}, executor);
		return newRegistration;
	}
```
当注册完成并且没有抛出异常时, 说明注册完成了, 则调用之前的ResourceManagerConnection.onRegistrationSuccess()进行连接, 最终会进去SlotPool.connectToResourceManager()
```
public void connectToResourceManager(ResourceManagerGateway resourceManagerGateway) {
		// 开始申请之前被pending的请求
		for (PendingRequest pendingRequest : waitingForResourceManager.values()) {
			requestSlotFromResourceManager(resourceManagerGateway, pendingRequest);
		}
		waitingForResourceManager.clear();
	}
```
当完成flink ResourceManager注册、连接后, 我们会逐个申请之前被挂起的请求。然后开始走之后描述的正常申请slot流程。

### 向ResourceManager申请slot
从requestSlotFromResourceManager()中最终会进入registerSlotRequest
```
    public boolean registerSlotRequest(SlotRequest slotRequest) throws SlotManagerException {
	    PendingSlotRequest pendingSlotRequest = new PendingSlotRequest(slotRequest);
		pendingSlotRequests.put(slotRequest.getAllocationId(), pendingSlotRequest);
		try {
		    internalRequestSlot(pendingSlotRequest);
		}
		return true;
    }
    private void internalRequestSlot(PendingSlotRequest pendingSlotRequest) throws ResourceManagerException {
		//是否发现目前拥有的slot
		TaskManagerSlot taskManagerSlot = findMatchingSlot(pendingSlotRequest.getResourceProfile());
		if (taskManagerSlot != null) {
			allocateSlot(taskManagerSlot, pendingSlotRequest);
		} else {
			resourceActions.allocateResource(pendingSlotRequest.getResourceProfile()); //跑到这里
		}
	}
```
internalRequestSlot做了如下逻辑:
+ 通过findMatchingSlot检查是否有现成可用的slot, 其中freeSlots包含着availiable slot. 比如在每当有新的TaskManager向JobManager注册时, 就会调用SlotManager.registerSlotRequest(), 在freeSlots中注册该TM可用的slot。若有可用slot时候, 就会调用allocateSlot进行分配。
+ 若没有可用空闲slot, 通过allocateResource申请TM, 最终会调用YarnResourceManager.requestYarnContainer进行申请。
```
    // //resource 是当前申请的container情况，比如<memory:6552, vCores:4>
    private void requestYarnContainer(Resource resource, Priority priority) {
        //获取当前实际正在申请的slot个数
		int pendingSlotRequests = getNumberPendingSlotRequests();
		// 通过当前正在申请的Contaainer个数*numberOfTaskSlots计算出预计当前正在申请的slot个数
		int pendingSlotAllocation = numPendingContainerRequests * numberOfTaskSlots;
		//若当前实际申请的slot个数 > 预计申请的slot个数, 那么需要去申请新的container, 来满足实际申请的slot个数
		if (pendingSlotRequests > pendingSlotAllocation) {
		    // 向yarn 发送申请container的请求。
			resourceManagerClient.addContainerRequest(new AMRMClient.ContainerRequest(resource, null, null, priority));
			resourceManagerClient.setHeartbeatInterval(FAST_YARN_HEARTBEAT_INTERVAL_MS);
			//正在申请的container统计自增
			numPendingContainerRequests++;
		}
	}
```
### 申请container成功
当向yarn成功申请到container之后, 会回调YarnResourceManager.onContainersAllocated通知jobManager。
```
	public void onContainersAllocated(List<Container> containers) {
		runAsync(() -> {
			for (Container container : containers) {
				if (numPendingContainerRequests > 0) {
					numPendingContainerRequests--;
					final String containerIdStr = container.getId().toString();
					final ResourceID resourceId = new ResourceID(containerIdStr);
					workerNodeMap.put(resourceId, new YarnWorkerNode(container));
					try {
						// 产生tm启动命令
						ContainerLaunchContext taskExecutorLaunchContext = createTaskExecutorLaunchContext(
							container.getResource(),
							containerIdStr,
							container.getNodeId().getHost());
                       // 启动TaskManager
						nodeManagerClient.startContainer(container, taskExecutorLaunchContext);
					}
				}
			}
			// if we are waiting for no further containers, we can go to the
			// regular heartbeat interval
			if (numPendingContainerRequests <= 0) {
				resourceManagerClient.setHeartbeatInterval(yarnHeartbeatIntervalMillis);
			}
		});
	}
```
回调函数主要做了如下逻辑:
1. 确定启动taskManager的命令。
2. 通过yarn启动taskManager。
我们来放一张整体逻辑图像:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/Flink_slot_allocate1.png" height="300" width="800"/>

## 部署subTask到对应的slot
当确定好subTask部署到一个TaskManager的slot上之后, 在scheduleEager中就开始调用Execution.deploy()进行部署。
```
public void deploy() throws JobException {
		final LogicalSlot slot  = assignedResource;  // SingleLogicalSlot
		try {
			final TaskDeploymentDescriptor deployment = vertex.createDeploymentDescriptor(
				attemptId,
				slot, // SingleLogicalSlot
				taskRestore,
				attemptNumber);
			// null taskRestore to let it be GC'ed
			taskRestore = null;
			final TaskManagerGateway taskManagerGateway = slot.getTaskManagerGateway();
			final CompletableFuture<Acknowledge> submitResultFuture = taskManagerGateway.submitTask(deployment, rpcTimeout);
		}
	}
    public CompletableFuture<Acknowledge> submitTask(TaskDeploymentDescriptor tdd, Time timeout) {
        // 实际会去调用AkkaInvocationHandler，而去和tm通信，而不是跑到TaskExecutor.submitTask
		return taskExecutorGateway.submitTask(tdd, jobMasterId, timeout);
    }
```
可以看到:
+ 首先生成TaskDeploymentDescriptor, 包含部署subTask的所有信息。
+ 调用taskManagerGateway.submitTask(deployment, rpcTimeout)进行部署subTask, JM接收RPC可参考:<a href="https://kkewwei.github.io/elasticsearch_learning/2019/04/20/Flink%E5%8E%9F%E7%90%86-Akka%E9%80%9A%E4%BF%A1%E6%A8%A1%E5%9D%97/">link原理-Akka通信原理</a> 。
TaskManager端通过TaskExecutor.subTask()接受到JobManager发出的部署SubTask的申请, 这样就完成SubTask部署了。

# SlotSharingGroup及共享slot
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
每个共享组的id都是相同的。

# MultiTaskSlot及SingleTaskSlot
MultiTaskSlot及SingleTaskSlot都继承TaskSlot, 每当subTask申请到一个未被共享的slot时, 就会产生一个MultiTaskSlot, 它代表着一个TM上的slot, 管理着该slot被共享的情况。 实际分配给每个subTask时, 会单独产生一个SingleTaskSlot, 然后每次被MultiTaskSlot管理着。之后若共享slot时, 分配到的都是同一个MultiTaskSlot, 不同的是每次每次分配都产生新的SingleTaskSlot。

# 参考文档
https://ci.apache.org/projects/flink/flink-docs-release-1.8/concepts/runtime.html
https://ci.apache.org/projects/flink/flink-docs-release-1.8/internals/job_scheduling.html
http://wuchong.me/blog/2016/05/03/flink-internals-overview/
http://wuchong.me/blog/2016/05/09/flink-internals-understanding-execution-resources/