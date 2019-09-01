---
title: Flink原理-TaskManager处理SubTask
date: 2019-04-03 12:10:04
tags:
toc: true
categories: Flink
---
在了解TaskManager处理task之前, 我们先看下该JVM启动过程。 TaskManager端yarn启动的类为YarnTaskExecutorRunner, TM端存放当前可利用的slot信息存放在TaskManagerServices.taskSlotTable里面, slot默认资源为new ResourceProfile(cpu=1.0, men=42)。
在<a href="https://kkewwei.github.io/elasticsearch_learning/2019/03/12/Flink%E5%8E%9F%E7%90%86-slot%E5%88%86%E9%85%8D/#%E9%83%A8%E7%BD%B2subTask%E5%88%B0%E5%AF%B9%E5%BA%94%E7%9A%84slot">Flink原理-Slot申请及SubTask部署</a>中, 我们知道, subTask会在JobManager中通过TaskManagerGateway.submitTask(deployment, rpcTimeout)进行部署subTask。TaskManager会通过TaskExecutor.subTask()接收到该部署请求。
```public CompletableFuture<Acknowledge> submitTask(
			TaskDeploymentDescriptor tdd,
			JobMasterId jobMasterId,
			Time timeout) {
		try {
			final JobID jobId = tdd.getJobId();
			//获得与jobManager通信的链接
			final JobManagerConnection jobManagerConnection = jobManagerTable.get(jobId);
			// re-integrate offloaded data:
			try {
				tdd.loadBigData(blobCacheService.getPermanentBlobService()); //啥事不做
			} catch (IOException | ClassNotFoundException e) {
				throw new TaskSubmissionException("Could not re-integrate offloaded TaskDeploymentDescriptor data.", e);
			}
			// deserialize the pre-serialized information
			final JobInformation jobInformation;
			final TaskInformation taskInformation;
			try {
				jobInformation = tdd.getSerializedJobInformation().deserializeValue(getClass().getClassLoader());
				taskInformation = tdd.getSerializedTaskInformation().deserializeValue(getClass().getClassLoader());
			}
			Task task = new Task(
				jobInformation,
				taskInformation,
				tdd.getExecutionAttemptId(),
				tdd.getAllocationId(),
				tdd.getSubtaskIndex(),
				tdd.getAttemptNumber(),
				tdd.getProducedPartitions(),
				tdd.getInputGates(),
				tdd.getTargetSlotNumber(),
				taskExecutorServices.getMemoryManager(),
				taskExecutorServices.getIOManager(),
				taskExecutorServices.getNetworkEnvironment(),
				taskExecutorServices.getBroadcastVariableManager(),
				taskStateManager,
				taskManagerActions,
				inputSplitProvider,
				checkpointResponder,
				blobCacheService,
				libraryCache,
				fileCache,
				taskManagerConfiguration,
				taskMetricGroup,
				resultPartitionConsumableNotifier,
				partitionStateChecker,
				getRpcService().getExecutor());
			log.info("Received task {}.", task.getTaskInfo().getTaskNameWithSubtasks());
			boolean taskAdded;
			try {
			    // 将task放入taskSlotTable.taskSlotMappings中
				taskAdded = taskSlotTable.addTask(task);
			}
			if (taskAdded) {
			    //这里比较重要，开始进行task执行
				task.startTaskThread();
				return CompletableFuture.completedFuture(Acknowledge.get());
			}
		}
	}

```
TM接收到请求后, 首先构建Task(Runnable), 然后通过task.startTaskThread()将这个线程运行起来, 真正执行任务的是run函数:
```
public void run() {

		// ----------------------------
		//  Initial State transition
		// ----------------------------
		while (true) {
			ExecutionState current = this.executionState;
			// 首先将Task状态置为DEPLOYING状态, 状态不修改成功, 不停地尝试
			if (current == ExecutionState.CREATED) {
				if (transitionState(ExecutionState.CREATED, ExecutionState.DEPLOYING)) {
					// success, we can start our work
					break;
				}
			}
			else if (current == ExecutionState.FAILED) {
				// we were immediately failed. tell the TaskManager that we reached our final state
				notifyFinalState();
				if (metrics != null) {
					metrics.close();
				}
				return;
			}
			else if (current == ExecutionState.CANCELING) {
				if (transitionState(ExecutionState.CANCELING, ExecutionState.CANCELED)) {
					// we were immediately canceled. tell the TaskManager that we reached our final state
					notifyFinalState();
					if (metrics != null) {
						metrics.close();
					}
					return;
				}
			}
		}

		// all resource acquisitions and registrations from here on
		// need to be undone in the end
		Map<String, Future<Path>> distributedCacheEntries = new HashMap<>();
		AbstractInvokable invokable = null;

		try {
			// ----------------------------
			//  Task Bootstrap - We periodically
			//  check for canceling as a shortcut
			// ----------------------------

			// activate safety net for task thread
			LOG.info("Creating FileSystem stream leak safety net for task {}", this);
			FileSystemSafetyNet.initializeSafetyNetForThread();

			blobService.getPermanentBlobService().registerJob(jobId);

			// first of all, get a user-code classloader
			// this may involve downloading the job's JAR files and/or classes
			LOG.info("Loading JAR files for task {}.", this);

			userCodeClassLoader = createUserCodeClassloader();
			final ExecutionConfig executionConfig = serializedExecutionConfig.deserializeValue(userCodeClassLoader);

			if (executionConfig.getTaskCancellationInterval() >= 0) {
				// override task cancellation interval from Flink config if set in ExecutionConfig
				taskCancellationInterval = executionConfig.getTaskCancellationInterval();
			}

			if (executionConfig.getTaskCancellationTimeout() >= 0) {
				// override task cancellation timeout from Flink config if set in ExecutionConfig
				taskCancellationTimeout = executionConfig.getTaskCancellationTimeout();
			}

			if (isCanceledOrFailed()) {
				throw new CancelTaskException();
			}

			// ----------------------------------------------------------------
			// register the task with the network stack
			// this operation may fail if the system does not have enough
			// memory to run the necessary data exchanges
			// the registration must also strictly be undone
			// ----------------------------------------------------------------

			LOG.info("Registering task at network: {}.", this);

			network.registerTask(this);

			// add metrics for buffers
			this.metrics.getIOMetricGroup().initializeBufferMetrics(this);

			// register detailed network metrics, if configured
			if (taskManagerConfig.getConfiguration().getBoolean(TaskManagerOptions.NETWORK_DETAILED_METRICS)) {
				// similar to MetricUtils.instantiateNetworkMetrics() but inside this IOMetricGroup
				MetricGroup networkGroup = this.metrics.getIOMetricGroup().addGroup("Network");
				MetricGroup outputGroup = networkGroup.addGroup("Output");
				MetricGroup inputGroup = networkGroup.addGroup("Input");

				// output metrics
				for (int i = 0; i < producedPartitions.length; i++) {
					ResultPartitionMetrics.registerQueueLengthMetrics(
						outputGroup.addGroup(i), producedPartitions[i]);
				}

				for (int i = 0; i < inputGates.length; i++) {
					InputGateMetrics.registerQueueLengthMetrics(
						inputGroup.addGroup(i), inputGates[i]);
				}
			}

			// next, kick off the background copying of files for the distributed cache
			try {
				for (Map.Entry<String, DistributedCache.DistributedCacheEntry> entry :
						DistributedCache.readFileInfoFromConfig(jobConfiguration)) {
					LOG.info("Obtaining local cache file for '{}'.", entry.getKey());
					Future<Path> cp = fileCache.createTmpFile(entry.getKey(), entry.getValue(), jobId, executionId);
					distributedCacheEntries.put(entry.getKey(), cp);
				}
			}

			if (isCanceledOrFailed()) {
				throw new CancelTaskException();
			}

			// ----------------------------------------------------------------
			//  call the user code initialization methods
			// ----------------------------------------------------------------

			TaskKvStateRegistry kvStateRegistry = network.createKvStateTaskRegistry(jobId, getJobVertexId());

			Environment env = new RuntimeEnvironment(
				jobId,
				vertexId,
				executionId,
				executionConfig,
				taskInfo,
				jobConfiguration,
				taskConfiguration,
				userCodeClassLoader,
				memoryManager,
				ioManager,
				broadcastVariableManager,
				taskStateManager,
				accumulatorRegistry,
				kvStateRegistry,
				inputSplitProvider,
				distributedCacheEntries,
				producedPartitions,
				inputGates,
				network.getTaskEventDispatcher(),
				checkpointResponder,
				taskManagerConfig,
				metrics,
				this);
            // 加载用户Task, 比如SourceStreamTask(Source: Custom Source (2/2))
			// now load and instantiate the task's invokable code
			invokable = loadAndInstantiateInvokable(userCodeClassLoader, nameOfInvokableClass, env);

			// ----------------------------------------------------------------
			//  actual task core work
			// ----------------------------------------------------------------

			// we must make strictly sure that the invokable is accessible to the cancel() call
			// by the time we switched to running.
			this.invokable = invokable;

			// switch to the RUNNING state, if that fails, we have been canceled/failed in the meantime
			if (!transitionState(ExecutionState.DEPLOYING, ExecutionState.RUNNING)) {
				throw new CancelTaskException();
			}

			// notify everyone that we switched to running
			notifyObservers(ExecutionState.RUNNING, null);
			taskManagerActions.updateTaskExecutionState(new TaskExecutionState(jobId, executionId, ExecutionState.RUNNING));

			// make sure the user code classloader is accessible thread-locally
			executingThread.setContextClassLoader(userCodeClassLoader);
            // 真正开始执行这个task。
			// run the invokable
			invokable.invoke();

			// make sure, we enter the catch block if the task leaves the invoke() method due
			// to the fact that it has been canceled
			if (isCanceledOrFailed()) {
				throw new CancelTaskException();
			}

			// ----------------------------------------------------------------
			//  finalization of a successful execution
			// ----------------------------------------------------------------

			// finish the produced partitions. if this fails, we consider the execution failed.
			for (ResultPartition partition : producedPartitions) {
				if (partition != null) {
					partition.finish();
				}
			}

			// try to mark the task as finished
			// if that fails, the task was canceled/failed in the meantime
			if (transitionState(ExecutionState.RUNNING, ExecutionState.FINISHED)) {
				notifyObservers(ExecutionState.FINISHED, null);
			}
			else {
				throw new CancelTaskException();
			}
		}
		finally {
			try {
				LOG.info("Freeing task resources for {} ({}).", taskNameWithSubtask, executionId);

				// clear the reference to the invokable. this helps guard against holding references
				// to the invokable and its structures in cases where this Task object is still referenced
				this.invokable = null;

				// stop the async dispatcher.
				// copy dispatcher reference to stack, against concurrent release
				ExecutorService dispatcher = this.asyncCallDispatcher;
				if (dispatcher != null && !dispatcher.isShutdown()) {
					dispatcher.shutdownNow();
				}

				// free the network resources
				network.unregisterTask(this);

				// free memory resources
				if (invokable != null) {
					memoryManager.releaseAll(invokable);
				}

				// remove all of the tasks library resources
				libraryCache.unregisterTask(jobId, executionId);
				fileCache.releaseJob(jobId, executionId);
				blobService.getPermanentBlobService().releaseJob(jobId);

				// close and de-activate safety net for task thread
				LOG.info("Ensuring all FileSystem streams are closed for task {}", this);
				FileSystemSafetyNet.closeSafetyNetAndGuardedResourcesForThread();

				notifyFinalState();
			}
		}
	}

```
该函数主要做了如下事情:
1. 加载jar, 配置文件。
2. 确认invoke函数, invoke函数在JobVertex产生的时候构建的, 比如在StreamingJobGraphGenerator.createJobVertex()中, 而这个invoke最初是在产生StreamNode时候赋值的, 比如在StreamingJobGraphGenerator.transform->transformOneInputTransform->StreamGraph.addOperator:
```
public <IN, OUT> void addOperator(
			Integer vertexID,
			String slotSharingGroup,
			@Nullable String coLocationGroup,
			StreamOperator<OUT> operatorObject,
			TypeInformation<IN> inTypeInfo,
			TypeInformation<OUT> outTypeInfo,
			String operatorName) {

		if (operatorObject instanceof StoppableStreamSource) {
			addNode(vertexID, slotSharingGroup, coLocationGroup, StoppableSourceStreamTask.class, operatorObject, operatorName);
		} else if (operatorObject instanceof StreamSource) {
			addNode(vertexID, slotSharingGroup, coLocationGroup, SourceStreamTask.class, operatorObject, operatorName);
		} else {
			addNode(vertexID, slotSharingGroup, coLocationGroup, OneInputStreamTask.class, operatorObject, operatorName);
		}
		......
```
可以看到, invoke可以为SourceStreamTask.class或者 OneInputStreamTask.class。 我们以最常见的OneInputStreamTask.class继续讲解。invokable.invoke()最终将跑到OneInputStreamTask.run里面
```
	protected void run() throws Exception {
		// cache processor reference on the stack, to make the code more JIT friendly
		final StreamInputProcessor<IN> inputProcessor = this.inputProcessor;

		while (running && inputProcessor.processInput()) {
			// all the work happens in the "processInput" method
		}
	}
```
可以看到, 每条数据就循环处理一次, 我们看下每条数据是如何处理的:
```
    public boolean processInput() throws Exception {
		if (isFinished) {
			return false;
		}
		if (numRecordsIn == null) {
			try {
				numRecordsIn = ((OperatorMetricGroup) streamOperator.getMetricGroup()).getIOMetricGroup().getNumRecordsInCounter();
			} catch (Exception e) {
				LOG.warn("An exception occurred during the metrics setup.", e);
				numRecordsIn = new SimpleCounter();
			}
		}
       //这个while是用来处理单个元素的（不要想当然以为是循环处理元素的）
		while (true) {            //2.会利用这个反序列化器得到下一个数据记录，并进行解析（是用户数据还是watermark等等），然后进行对应的操作
			if (currentRecordDeserializer != null) {
				DeserializationResult result = currentRecordDeserializer.getNextRecord(deserializationDelegate); //得到下一条数据

				if (result.isBufferConsumed()) {
					currentRecordDeserializer.getCurrentBuffer().recycleBuffer();
					currentRecordDeserializer = null;
				}

				if (result.isFullRecord()) {
					StreamElement recordOrMark = deserializationDelegate.getInstance();
					//如果元素是watermark，就准备更新当前channel的watermark值（并不是简单赋值，因为有乱序存在）
					if (recordOrMark.isWatermark()) {
						// handle watermark
						statusWatermarkValve.inputWatermark(recordOrMark.asWatermark(), currentChannel);
						continue;
					//如果元素是status，就进行相应处理。可以看作是一个flag，标志着当前stream接下来即将没有元素输入（idle），或者当前即将由空闲状态转为有元素状态（active）。同时，StreamStatus还对如何处理watermark有影响。通过发送status，上游的operator可以很方便的通知下游当前的数据流的状态。
					} else if (recordOrMark.isStreamStatus()) {
						// handle stream status
						statusWatermarkValve.inputStreamStatus(recordOrMark.asStreamStatus(), currentChannel);
						continue;
					//LatencyMarker是用来衡量代码执行时间的。在Source处创建，携带创建时的时间戳，流到Sink时就可以知道经过了多长时间
					} else if (recordOrMark.isLatencyMarker()) {
						// handle latency marker
						synchronized (lock) {
							streamOperator.processLatencyMarker(recordOrMark.asLatencyMarker());
						}
						continue;
					//这里就是真正的，用户的代码即将被执行的地方。
					} else {
						// now we can do the actual processing
						StreamRecord<IN> record = recordOrMark.asRecord();
						synchronized (lock) {
							numRecordsIn.inc();
							streamOperator.setKeyContextElement1(record);
							// 每个算子开始真正处理数据
							streamOperator.processElement(record);
						}
						return true;
					}
				}
			}
            //1.程序首先获取下一个buffer，这一段代码是服务于flink的FaultTorrent机制的，后面我会讲到，这里只需理解到它会尝试获取buffer，然后赋值给当前的反序列化器
			final BufferOrEvent bufferOrEvent = barrierHandler.getNextNonBlocked();
			if (bufferOrEvent != null) {
				if (bufferOrEvent.isBuffer()) {
					currentChannel = bufferOrEvent.getChannelIndex();
					currentRecordDeserializer = recordDeserializers[currentChannel];
					currentRecordDeserializer.setNextBuffer(bufferOrEvent.getBuffer());
				}
				else {
					// Event received
					final AbstractEvent event = bufferOrEvent.getEvent();
					if (event.getClass() != EndOfPartitionEvent.class) {
						throw new IOException("Unexpected event: " + event);
					}
				}
			}
			else {
				isFinished = true;
				if (!barrierHandler.isEmpty()) {
					throw new IllegalStateException("Trailing data in checkpoint barrier handler.");
				}
				return false;
			}
		}
	}

```
每条数据都是通过streamOperator.processElement(record)来完成处理的, 以map算子为例, streamOperator为StreamMap:
 ```
 @Internal
public class StreamMap<IN, OUT>
		extends AbstractUdfStreamOperator<OUT, MapFunction<IN, OUT>>
		implements OneInputStreamOperator<IN, OUT> {
	private static final long serialVersionUID = 1L;
	public StreamMap(MapFunction<IN, OUT> mapper) {
		super(mapper);
		chainingStrategy = ChainingStrategy.ALWAYS;
	}
	@Override
	public void processElement(StreamRecord<IN> element) throws Exception {
		output.collect(element.replace(userFunction.map(element.getValue())));
	}
}
 ```
 可以看到, 我们只用负责定义map就可以了。