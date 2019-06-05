---
title: Flink原理-Akka通信模块详解
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
1. 在使用actorSelection函数的时候, 我们需要了解一个关键字<a href="https://doc.akka.io/docs/akka/2.3.6/java/untyped-actors.html#actorselection-java">Identify</a>, 它代表着一个每个角色都知道其含义的消息, 当接收到Identify时, 角色自动回复ActorIdentity, 其中包含着对角色的引用。这样就可以获取到该角色了。
2. ask和tell的区别是, ask希望对方角色返回结果, 而tell不需要返回结果。

# Akka RPC通信过程
## AkkaRpcService
在正式介绍之前, 先介绍RpcService, 作为Flink Akka通信核心类, 它包含了akka通信系统ActorSystem, 任何角色产生, 都会调用RpcService.startServer
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
 2. 调用函数时的接口handler也分为FencedAkkaInvocationHandler/AkkaInvocationHandler。
 每次代理GateWay请求时, 首先会调用AkkaInvocationHandler.invoke()类。

# TM向JM注册
我们以TaskManager启动后, 会去主动连接JobMaster的ResourceManager, 以它们之间的通信为例进行讲解, 首先进入connectToResourceManager:
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
1. 通过createNewRegistration构建ResourceManagerRegistration, 这里只是构建赋值。同时在父类RetryingRegistration中定义了一个CompletableFuture, 当complete时候(成功向jm注册后), 会去调用TaskExecutorToResourceManagerConnection.onRegistrationSuccess(), 之后会详细介绍。
2. 通过AkkaRpcService.startRegistration真正开始注册TaskManager。接下来看下如何向JobManager的ResourceManager注册该TaskManager。
```
    public void startRegistration() {
		try {
			// trigger resolution of the resource manager address to a callable gateway
			final CompletableFuture<G> resourceManagerFuture;
			if (FencedRpcGateway.class.isAssignableFrom(targetType)) {
			    //返回的就是clazz对象ResourceManagerGateway
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
					register(result, 1, initialRegistrationTimeout);
				},
				rpcService.getExecutor());
		}
	}
```
主要逻辑如下:
1. 调用rpcService.connect进行真正的远程jm连接。
2. 当远程连接上之后, 调用register进行回调, 在

connectInternal
