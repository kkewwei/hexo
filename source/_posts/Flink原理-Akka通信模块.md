---
title: Flink原理-Akka通信&TaskManager注册
date: 2019-04-20 23:16:59
tags:
toc: true
---
Flink对于JobManager和TaskManager之间通信采用Akka remote实现, 本文将以tm和jm之间的一次远程通信为示例进行讲解。为了读者更好地理解, 本文首先介绍AKKa相关基本知识。
# Akka基础学习
本文首先以hello world为例进行讲解:
```
public class hello {
    public static void main(String[] args) {
        // 创建系统
        final ActorSystem system = ActorSystem.create("hello");
        // 创建消息角色(): greeter1, 接受消息处理类: Greeter
        final ActorRef greeter1 = system.actorOf(Props.create(Greeter.class), "Sender1");
        final ActorRef greeter2 = system.actorOf(Props.create(Greeter.class), "Sender2");
        //消息发送greeter1 ->  greeter2, 消息内容"tell1"
        greeter2.tell("tell1", greeter1);

        //通过ask, 表示角色发送完消息, 希望对方返回。
        Future<String> future  = Patterns.ask(greeter1, "ask1", 5000).<String>mapTo(ClassTag$.MODULE$.<String>apply(String.class));
        future.onComplete(new OnComplete<String>() {
            public void onComplete(Throwable throwable, String o) throws Throwable {
                if (throwable == null) {
                    System.out.println(o);
                }
            }
        }, system.dispatcher());

        //获取greeter1的绝对路径
        String url = greeter1.path().parent().toString()+"/"+greeter1.path().name()+"#"+greeter1.path().uid();
        // 通过actorSelection, 可以查找角色, 这个函数比较重要, 我们只知道某个角色的绝对路径, 我们通过该函数, 就可以获取到该角色。一般用于远程数据传输。
        ActorSelection actorSelection = system.actorSelection(url);
        Future<ActorIdentity> identify  = Patterns.ask(actorSelection, new Identify(1), 5000).<ActorIdentity>mapTo(ClassTag$.MODULE$.<ActorIdentity>apply(ActorIdentity.class));
        identify.onComplete(new OnComplete<ActorIdentity>() {
            public void onComplete(Throwable throwable, ActorIdentity o) throws Throwable {
                if (throwable != null) {
                } else {
                    // 我们可以对比发现, 之前那个角色通过绝对路径获取到了。
                    ActorRef actorRef = o.getRef();
                    System.out.println("system.actorSelection verify:" + (actorRef==greeter1));
                }
            }
        }, system.dispatcher());
    }
    //消息处理类, 必须继承UntypedActor
    public static class Greeter extends UntypedActor {
        // 消息角色真正接收到数据的函数
        public void onReceive(Object message) {
            if(message instanceof String && ((String)message).equals("tell1")) {
                System.out.println(message);
                getSender().tell("word2", getSelf());
            } else if(message instanceof String && ((String)message).equals("ask1")) {
                System.out.println(message);
                // getSelf()表示本角色
                getSender().tell("Patterns.ask()", getSelf());
            } else {
                System.out.println("unhandled:" + message);
                unhandled(message);
            }
        }
    }
}
```
输出为:
```
tell1
unhandled:word2
ask1
Patterns.ask()
system.actorSelection verify:true
```
1. 在使用actorSelection函数的时候, 我们需要了解一个关键字<a href="https://doc.akka.io/docs/akka/2.3.6/java/untyped-actors.html#actorselection-java">Identify</a>, 它被定义为每个角色都知道其含义的消息, 当接收到Identify时, 角色自动回复ActorIdentity, 其中包含着对地址的角色。actorSelection可以获取该角色的引用, 这样就可以首先和该角色通信了。
2. ask和tell的区别是, ask希望对方角色返回结果, 而tell不需要返回结果。

# Akka RPC通信过程
## AkkaRpcService和RpcServer
在正式介绍之前, 先介绍AkkaRpcService, 作为Flink Akka通信核心类, 它包含了akka通信系统ActorSystem, 任何角色产生, 都会调用RpcService.startServer
```
    public <C extends RpcEndpoint & RpcGateway> RpcServer startServer(C rpcEndpoint) {
		CompletableFuture<Void> terminationFuture = new CompletableFuture<>();
		final Props akkaRpcActorProps;
		if (rpcEndpoint instanceof FencedRpcEndpoint) {
			akkaRpcActorProps = Props.create(FencedAkkaRpcActor.class, rpcEndpoint, terminationFuture, getVersion());
		} else {
			akkaRpcActorProps = Props.create(AkkaRpcActor.class, rpcEndpoint, terminationFuture, getVersion());
		}
		ActorRef actorRef;
		synchronized (lock) {
			actorRef = actorSystem.actorOf(akkaRpcActorProps, rpcEndpoint.getEndpointId());
			actors.put(actorRef, rpcEndpoint);
		}
		LOG.info("Starting RPC endpoint for {} at {} .", rpcEndpoint.getClass().getName(), actorRef.path());
		final String akkaAddress = AkkaUtils.getAkkaURL(actorSystem, actorRef);
		final String hostname;
		Option<String> host = actorRef.path().address().host();
		if (host.isEmpty()) {
			hostname = "localhost";
		} else {
			hostname = host.get();
		}
		Set<Class<?>> implementedRpcGateways = new HashSet<>(RpcUtils.extractImplementedRpcGateways(rpcEndpoint.getClass()));
		implementedRpcGateways.add(RpcServer.class);
		implementedRpcGateways.add(AkkaBasedEndpoint.class);
		final InvocationHandler akkaInvocationHandler;
		if (rpcEndpoint instanceof FencedRpcEndpoint) {
			// a FencedRpcEndpoint needs a FencedAkkaInvocationHandler
			akkaInvocationHandler = new FencedAkkaInvocationHandler<>(
				akkaAddress,
				hostname,
				actorRef,
				timeout,
				maximumFramesize,
				terminationFuture,
				((FencedRpcEndpoint<?>) rpcEndpoint)::getFencingToken);
			implementedRpcGateways.add(FencedMainThreadExecutable.class);
		} else {
			akkaInvocationHandler = new AkkaInvocationHandler(
				akkaAddress,
				hostname,
				actorRef,
				timeout,
				maximumFramesize,
				terminationFuture);
		}
		ClassLoader classLoader = getClass().getClassLoader();
		RpcServer server = (RpcServer) Proxy.newProxyInstance(
			classLoader,
			implementedRpcGateways.toArray(new Class<?>[implementedRpcGateways.size()]),
			akkaInvocationHandler);
		return server;
	}
```
可以看到:
 1. 角色接收到信息后, 处理的类为FencedAkkaRpcActor/AkkaRpcActor。
 2. 调用函数时的接口handler也分为FencedAkkaInvocationHandler/AkkaInvocationHandler。每次代理GateWay请求时, 首先会调用AkkaInvocationHandler.invoke()类。
我们再来介绍下: RpcServer, RpcServer作为任何一个请求终端, 每次都将从相同的AkkaRpcService中参数, 实际封装了ActorRef来进行内部数据传输。

# TM向JM注册
我们以TaskManager启动后, 会去主动连接JobMaster的ResourceManager, 以它们之间的通信为例进行讲解。YarnTaskExecutorRunner在运行主函数时, 会去调用TaskExecutor.connectToResourceManager()主动连接JobManager:
```
    private void connectToResourceManager() {
        log.info("Connecting to ResourceManager {}.", resourceManagerAddress);
		resourceManagerConnection =
			new TaskExecutorToResourceManagerConnection(
				log,
				getRpcService(),
				getAddress(),
				getResourceID(),
				taskManagerLocation.dataPort(),
				hardwareDescription,
				resourceManagerAddress.getAddress(),
				resourceManagerAddress.getResourceManagerId(),
				getMainThreadExecutor(),
				new ResourceManagerRegistrationListener());
		resourceManagerConnection.start();
	}
```
建立好TaskExecutorToResourceManagerConnection之后,看下是如何操作的:
```
	public void start() {
		final RetryingRegistration<F, G, S> newRegistration = createNewRegistration();
		if (REGISTRATION_UPDATER.compareAndSet(this, null, newRegistration)) {
			newRegistration.startRegistration();
		}
	}
```
该函数主要做了如下事情:
1. 通过createNewRegistration构建ResourceManagerRegistration对象, 并没有做其他的事。同时在父类RetryingRegistration中定义了一个CompletableFuture, 当complete时候(成功向jm注册后), 会去调用TaskExecutorToResourceManagerConnection.onRegistrationSuccess(), 之后会详细介绍。
2. 通过AkkaRpcService.startRegistration真正开始注册TaskManager。接下来看下如何向JobManager的ResourceManager注册该TaskManager。
```
    public void startRegistration() {
		try {
			// trigger resolution of the resource manager address to a callable gateway
			final CompletableFuture<G> resourceManagerFuture;
			if (FencedRpcGateway.class.isAssignableFrom(targetType)) {
			    //返回的就是clazz对象ResourceManagerGateway的代理
				resourceManagerFuture = (CompletableFuture<G>) rpcService.connect(
					targetAddress,
					fencingToken,
					targetType.asSubclass(FencedRpcGateway.class));
			} else {
				resourceManagerFuture = rpcService.connect(targetAddress, targetType);
			}
			// upon success, start the registration attempts
			CompletableFuture<Void> resourceManagerAcceptFuture = resourceManagerFuture.thenAcceptAsync(
				(G result) -> {
					log.info("Resolved {} address, beginning registration", targetName);
					// 真正想远程JobManager进行注册
					register(result, 1, initialRegistrationTimeout);
				},
				rpcService.getExecutor());
		}
	}
```
主要逻辑如下:
1. 调用rpcService.connect, 获取代理, 代理中包含指向jobManager的ResourceManager的地址(akka.tcp://flink@jobmanager:port/user/resourcemanager)的角色, 以进行rpc通信。
2. 当产生代理后, 再调用register进行真正RPC远程向JobManager的ResourceManager注册。

## 构建远程RPC调用的角色及代理
我们首先来看下是如何获取代理的:
```
private <C extends RpcGateway> CompletableFuture<C> connectInternal(
			final String address,
			final Class<C> clazz,
			Function<ActorRef, InvocationHandler> invocationHandlerFactory) {
		// 根据路径获取YarnResourceManager中JobManager ResourceManager的路径引用
		final ActorSelection actorSel = actorSystem.actorSelection(address);
		// 向该远程路径发送RPC请求, 获取对该远程路径的回话角色, 之后就可以向这个地址发送数据了
		final Future<ActorIdentity> identify = Patterns
			.ask(actorSel, new Identify(42), timeout.toMilliseconds())
			.<ActorIdentity>mapTo(ClassTag$.MODULE$.<ActorIdentity>apply(ActorIdentity.class));

		final CompletableFuture<ActorIdentity> identifyFuture = FutureUtils.toJava(identify);
		final CompletableFuture<ActorRef> actorRefFuture = identifyFuture.thenApply(
			(ActorIdentity actorIdentity) -> {
				if (actorIdentity.getRef() == null) {
				} else {
				    //获取对应的引用
					return actorIdentity.getRef();
				}
			});
		// 发送handshake请求
		final CompletableFuture<HandshakeSuccessMessage> handshakeFuture = actorRefFuture.thenCompose(
			(ActorRef actorRef) -> FutureUtils.toJava(
				Patterns
					.ask(actorRef, new RemoteHandshakeMessage(clazz, getVersion()), timeout.toMilliseconds())
					.<HandshakeSuccessMessage>mapTo(ClassTag$.MODULE$.<HandshakeSuccessMessage>apply(HandshakeSuccessMessage.class))));
		return actorRefFuture.thenCombineAsync(
			handshakeFuture,
			(ActorRef actorRef, HandshakeSuccessMessage ignored) -> {
			    //获取FencedAkkaInvocationHandler
				InvocationHandler invocationHandler = invocationHandlerFactory.apply(actorRef);
				ClassLoader classLoader = getClass().getClassLoader();
				@SuppressWarnings("unchecked")
				// 构建针对ResourceManagerGateway的代理, handler为FencedAkkaInvocationHandler
				C proxy = (C) Proxy.newProxyInstance(
					classLoader,
					new Class<?>[]{clazz},
					invocationHandler);
				return proxy;
			},
			actorSystem.dispatcher());
	}
```
连接请求也比较简单, 每次RPC调用时, 都使用ask:
1. 向ResourceManager发送Identify, 远程响应并发返回对应路径的角色
2. 向ResourceManager发送RemoteHandshakeMessage, 再次和远程确认。
3. 以上两个RPC调用完成后, 构建针对ResourceManagerGateway的代理, 其中handler为FencedAkkaInvocationHandler(rpcEndpoint=ActorRef)。

## TM向JM远程RPC注册
当获取到通信的ActorRef后, 调用register进行注册:
```
    private void register(final G gateway, final int attempt, final long timeoutMillis) {
		try {
		    // 跑到ResourceManagerConnection.invokeRegistration
			log.info("Registration at {} attempt {} (timeout={}ms)", targetName, attempt, timeoutMillis);
			CompletableFuture<RegistrationResponse> registrationFuture = invokeRegistration(gateway, fencingToken, timeoutMillis);
			// if the registration was successful, let the TaskExecutor know
			CompletableFuture<Void> registrationAcceptFuture = registrationFuture.thenAcceptAsync(
				(RegistrationResponse result) -> {
					if (!isCanceled()) {
						if (result instanceof RegistrationResponse.Success) {
							// registration successful!
							S success = (S) result;  // JobMasterRegistrationSuccess
							//表示完成了JobMaster的注册，将completionFuture置为完成
							completionFuture.complete(Tuple2.of(gateway, success));
						}
					}
				},
				rpcService.getExecutor());
		}
		catch (Throwable t) {
			completionFuture.completeExceptionally(t);
			cancel();
		}
	}
```
可以看到:
1. register中真正向ResourceManager通信的是invokeRegistration()。
2. 将completionFuture置为完成, 那么将触发之前定义的TaskExecutorToResourceManagerConnection.onRegistrationSuccess()。
我们先看invokeRegistration的实现逻辑:
```
        protected CompletableFuture<RegistrationResponse> invokeRegistration(
				ResourceManagerGateway resourceManager, ResourceManagerId fencingToken, long timeoutMillis) throws Exception {
			Time timeout = Time.milliseconds(timeoutMillis);
			//resourceManager=FencedAkkaInvocationHandler,返回TaskExecutorRegistrationSuccess， 这里会跳到JobManager
			return resourceManager.registerTaskExecutor(
				taskExecutorAddress,
				resourceID,
				dataPort,
				hardwareDescription,
				timeout);
		}
```
resourceManager.registerTaskExecutor将首先跑到FencedAkkaInvocationHandler.invoke, 最终真正发送rpc请求的是invokeRpc:
```
	private Object invokeRpc(Method method, Object[] args) throws Exception {
		String methodName = method.getName();
		Class<?>[] parameterTypes = method.getParameterTypes();
		Annotation[][] parameterAnnotations = method.getParameterAnnotations();
		Time futureTimeout = extractRpcTimeout(parameterAnnotations, args, timeout);
		//这里会构建函数名称等
		final RpcInvocation rpcInvocation = createRpcInvocationMessage(methodName, parameterTypes, args);
		Class<?> returnType = method.getReturnType();
		final Object result;
		if (Objects.equals(returnType, Void.TYPE)) {
			tell(rpcInvocation);
			result = null;
		} else if (Objects.equals(returnType, CompletableFuture.class)) {
			// execute an asynchronous call, 需要返回请求结果。
			result = ask(rpcInvocation, futureTimeout);
		} else {
			// execute a synchronous call
			CompletableFuture<?> futureResult = ask(rpcInvocation, futureTimeout);
			result = futureResult.get(futureTimeout.getSize(), futureTimeout.getUnit());
		}
		return result;
	}
```
最终通过ask将请求发送出去, 其中包括函数名, 参数等信息。
# 远程ResourceManager接收到请求
我们可以看到在构建YarnResourceManager时, resourceManagerEndpointId为`resourcemanager`, 最终其ActorRef对应的直接地址为: akka.tcp://flink@jobmanager:port/user/resourcemanager, 印证了之前TM向JM注册时, JM通信的终端就是该类。在ActorRef构建过程中, 知道JM接受处理类为AkkaRpcActor.onReceive(FencedAkkaRpcActor父类), 继续调用的是handleRpcMessage()
```
	protected void handleRpcMessage(Object message) {
		if (message instanceof RunAsync) {
			handleRunAsync((RunAsync) message);
		} else if (message instanceof CallAsync) {
			handleCallAsync((CallAsync) message);
		} else if (message instanceof RpcInvocation) {
			handleRpcInvocation((RpcInvocation) message);
		}
	}
```
flink针对不同类型的消息, 使用调用的函数, 很显然, 这里调用handleRpcInvocation:
```
	private void handleRpcInvocation(RpcInvocation rpcInvocation) {
		Method rpcMethod = null;
		try {
			String methodName = rpcInvocation.getMethodName();
			Class<?>[] parameterTypes = rpcInvocation.getParameterTypes();
			rpcMethod = lookupRpcMethod(methodName, parameterTypes);
		}
		if (rpcMethod != null) {
			try {
				// this supports declaration of anonymous classes
				rpcMethod.setAccessible(true);
				if (rpcMethod.getReturnType().equals(Void.TYPE)) {
					// No return value to send back
					rpcMethod.invoke(rpcEndpoint, rpcInvocation.getArgs());
				}
				else {
					final Object result;
					try {
						result = rpcMethod.invoke(rpcEndpoint, rpcInvocation.getArgs());
					}
					if (result instanceof CompletableFuture) {
						final CompletableFuture<?> future = (CompletableFuture<?>) result;
						Promise.DefaultPromise<Object> promise = new Promise.DefaultPromise<>();

						future.whenComplete(
							(value, throwable) -> {
								if (throwable != null) {
									promise.failure(throwable);
								} else {
								    //再回调一下
									promise.success(value);
								}
							});

						Patterns.pipe(promise.future(), getContext().dispatcher()).to(getSender());
					}
				}
			}
		}
	}
```
最终调用的是result = rpcMethod.invoke()来处理TM发送的请求。实际调用ResourceManager.registerTaskExecutor()->registerTaskExecutorInternal()
```
	private RegistrationResponse registerTaskExecutorInternal(
			TaskExecutorGateway taskExecutorGateway,
			String taskExecutorAddress,
			ResourceID taskExecutorResourceId,
			int dataPort,
			HardwareDescription hardwareDescription) {
		WorkerRegistration<WorkerType> oldRegistration = taskExecutors.remove(taskExecutorResourceId);
		// 首先做些清理, 删掉旧的
		if (oldRegistration != null) {
			// TODO :: suggest old taskExecutor to stop itself
			log.debug("Replacing old registration of TaskExecutor {}.", taskExecutorResourceId);
			// remove old task manager registration from slot manager
			slotManager.unregisterTaskManager(oldRegistration.getInstanceID());
		}
        // 跑到YarnResourceManager.workerStarted(在yarn返回container成功后，会去注册Container)
		final WorkerType newWorker = workerStarted(taskExecutorResourceId);
		if (newWorker == null) {
		    //找不到, 就说明这个container不是这个JM申请的
			log.warn("Discard registration from TaskExecutor {} at ({}) because the framework did " +
				"not recognize it", taskExecutorResourceId, taskExecutorAddress);
			return new RegistrationResponse.Decline("unrecognized TaskExecutor");
		} else {
			WorkerRegistration<WorkerType> registration =
			    //taskExecutorGateway=AkkaInvocationHandler，  newWorker = YarnWorkerNode
				new WorkerRegistration<>(taskExecutorGateway, newWorker, dataPort, hardwareDescription);
			// 统计活跃而Container。
			taskExecutors.put(taskExecutorResourceId, registration);
			taskManagerHeartbeatManager.monitorTarget(taskExecutorResourceId, new HeartbeatTarget<Void>() {
				@Override
				public void receiveHeartbeat(ResourceID resourceID, Void payload) {
					// the ResourceManager will always send heartbeat requests to the
					// TaskManager
				}
				@Override
				public void requestHeartbeat(ResourceID resourceID, Void payload) {
					taskExecutorGateway.heartbeatFromResourceManager(resourceID);
				}
			});
			return new TaskExecutorRegistrationSuccess(
				registration.getInstanceID(),
				resourceId,
				resourceManagerConfiguration.getHeartbeatInterval().toMilliseconds(),
				clusterInformation);
		}
	}
```
ResourceManager注册主要做了如下事情:
1. 从taskExecutors中删除旧的通信管道。
2. 跑到YarnResourceManager.workerStarted()里面, 从JM端根据获取当初yarn分配的Container。
3. 向taskExecutors添加新产生的管道WorkerRegistration。管道里包含TaskExecutorGateway的代理, 其中handler为AkkaInvocationHandler, 且包含连接JobManager的ActorRef。
4. JM对连接的TM添加监控。然后响应TM。

# TM接收到JM响应
当TM接收到JM响应, 就会回调之前定义的TaskExecutorToResourceManagerConnection.onRegistrationSuccess()。
```
	private void establishResourceManagerConnection(
			ResourceManagerGateway resourceManagerGateway,
			ResourceID resourceManagerResourceId,
			InstanceID taskExecutorRegistrationId,
			ClusterInformation clusterInformation) {
		// TM向JM会报本地slot的情况, 供JM来分配给申请者。
		final CompletableFuture<Acknowledge> slotReportResponseFuture = resourceManagerGateway.sendSlotReport(
			getResourceID(),
			taskExecutorRegistrationId,
			taskSlotTable.createSlotReport(getResourceID()),
			taskManagerConfiguration.getTimeout());
		// monitor the resource manager as heartbeat target
		resourceManagerHeartbeatManager.monitorTarget(resourceManagerResourceId, new HeartbeatTarget<SlotReport>() {
			@Override
			public void receiveHeartbeat(ResourceID resourceID, SlotReport slotReport) {
				resourceManagerGateway.heartbeatFromTaskManager(resourceID, slotReport);
			}
			@Override
			public void requestHeartbeat(ResourceID resourceID, SlotReport slotReport) {
				// the TaskManager won't send heartbeat requests to the ResourceManager
			}
		});
		// set the propagated blob server address
		final InetSocketAddress blobServerAddress = new InetSocketAddress(
			clusterInformation.getBlobServerHostname(),
			clusterInformation.getBlobServerPort());
		blobCacheService.setBlobServerAddress(blobServerAddress);
		establishedResourceManagerConnection = new EstablishedResourceManagerConnection(
			resourceManagerGateway,
			resourceManagerResourceId,
			taskExecutorRegistrationId);
		stopRegistrationTimeout();
	}
```
TM当收到Response, 回调函数做了如下事情:
1. 通过resourceManagerGateway.sendSlotReport向YarnResourceManager汇报当前TM可用slot, 可用slot都将保存在 JobManager ResourceManager.freeSlots里面。
2. 开始向ResourceManager上报心跳。

# 总结
可以看到, Akka通信最底层依靠的是Patterns.ask来完成, 整个通信流程也是比较清晰的。