---
title: ServerBootstrap初始篇
date: 2018-01-14 18:32:43
tags: netty4, ServerBootstrap, Initiale
toc: true
categories: Netty
---
&emsp;本文将以一个最简单的netty服务器端代码进行讲解。
# 服务器示例
 ```
 public class HelloServer {
    /**
     * 服务端监听的端口地址
     */
    private static final int portNumber = 7878;
    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup WorkGroup = new NioEventLoopGroup(4);
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup,WorkGroup);
            b.channel(NioServerSocketChannel.class);
            b.childHandler(new HelloServerInitializer());
            ChannelFuture f = b.bind(portNumber).sync();
            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
        }
    }
}

class HelloServerInitializer extends ChannelInitializer<SocketChannel> {
    @Override       //  ch = NioSocketChannel
    protected void initChannel(SocketChannel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast("frameDecoder", new LengthFieldBasedFrameDecoder(Integer.MAX_VALUE, 0, 4, 0, 4));
        pipeline.addLast("frameEncoder", new LengthFieldPrepender(4));
        pipeline.addLast("decoder", new StringDecoder(CharsetUtil.UTF_8));
        pipeline.addLast("encoder", new StringEncoder(CharsetUtil.UTF_8));
        // 自己的逻辑Handler
        pipeline.addLast("handler", new HelloServerHandler());
    }
}

class HelloServerHandler extends SimpleChannelInboundHandler<String> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) throws Exception {
        // 收到消息直接打印输出
        System.out.println(ctx.channel().remoteAddress() + " Say : " + msg);
        // 返回客户端消息 - 我已经接收到了你的消息
        ctx.writeAndFlush("Received your message !\n");
    }
    /*
     *
     * 覆盖 channelActive 方法 在channel被启用的时候触发 (在建立连接的时候)
     *
     * channelActive 和 channelInActive 在后面的内容中讲述，这里先不做详细的描述
     * */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush("Welcome to " + InetAddress.getLocalHost().getHostName() + " service!\n");
        super.channelActive(ctx);
    }
}
 ```
 # NioEventLoop和NioEventLoopGroup分析
  + NioEventLoop:是一个单线程执行器(所有),所有task的具体执行者,每个task都是一个Runnable实例。NioEventLoop内的线程池线程,默认取值为`NettyRuntime.availableProcessors() * 2)`
  + NioEventLoopGroup:每个NioEventLoop都有一个分组,NioEventLoopGroup一般分为两组parentGroup、childGroup,parentGroup管理一类NioEventLoop,这类执行器主要生成boss类的线程,实际使用时,childGroup管理的一类NioEventLoop主要生成work类的线程。

# 一些概念对应关系
  + 一个NioEventLoop可以处理分配给多个Channel(包含NioServerSocketChannel), 是一对多的关系。
  + NioEventLoop里面处理task的线程唯一。
  + Channel与NioEventLoop绑定称之为register。在它的生命周期产生的所有task内只能由固定的某一个NioEventLoop处理。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/Netty%E6%A6%82%E5%BF%B5.png" />


# 具体过程分析
 ## 首先分析AbstractBootstrap.doBind()
 ```
    private ChannelFuture doBind(final SocketAddress localAddress) {
       final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel(); //NioServerSocketChannel
        if (regFuture.cause() != null) {
            return regFuture;
        }

        if (regFuture.isDone()) {
            // At this point we know that the registration was complete and successful.
            ChannelPromise promise = channel.newPromise();
            doBind0(regFuture, channel, localAddress, promise);
            return promise;
        } else {
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.registered();

                        doBind0(regFuture, channel, localAddress, promise);
                    }
                }
            });
            return promise;
    }
```
主要干的事:
1. 生成并初始化NioServerSocketChannel,见initAndRegister():
2. 检查该channel是否应注册到selector上。若注册上去后, 才会进行真正的channel与address、事件(OP_ACCEPT)绑定(见`doBind0`)。

initAndRegister()主要作用:
(1) 生成一个NioServerSocketChannel, 实际使用的`SelectorProvider.provider().openServerSocketChannel()`;
NioServerSocketChannel构造函数如下:
```
        this.parent = parent;
        id = newId();// 分配一个全局唯一的id，默认为MAC+进程id+自增的序列号+时间戳相关的数值+随机值
        // 初始化Unsafe, NioSocketChannel对应的是NioSocketChannelUnsafe; 而NioServerSocketChannel对应着NioMessageUnsafe
        unsafe = newUnsafe();
        pipeline = newChannelPipeline();//// 初始化pipeline，
        this.ch = ch;
        this.readInterestOp = readInterestOp;
        try {
            ch.configureBlocking(false); //
        } catch (IOException e) {
            try {
                ch.close();
            } catch (IOException e2) {
                if (logger.isWarnEnabled()) {
                    logger.warn(
                            "Failed to close a partially initialized socket.", e2);
                }
            }
            throw new ChannelException("Failed to enter non-blocking mode.", e);
        }
```
+ 每一个channel都将独自拥有一个DefaultChannelPipeline, DefaultChannelPipeline主要的属性如下:
```
this.channel = ObjectUtil.checkNotNull(channel, "channel");
tail = new TailContext(this);   //只是in
head = new HeadContext(this);  //只是out

head.next = tail;
tail.prev = head;
```
基本含义就是每个DefaultChannelPipeline与一个channel绑定, 该channle对应的处理链由head和tail串联起来。
+ TailContext和HeadContext是所有Pipeline默认拥有的,他们本身同时继承了AbstractChannelHandlerContext, 另外HeadContext继承了ChannelOutboundHandler, ChannelInboundHandler两种属性, TailContext继承了ChannelOutboundHandler一种, 返回handler都是本身
+ ch传递过来的参数是SelectionKey.OP_ACCEPT, 之后会再次初始化成0(0并不是SelectionKey其中的一种), (见doRegister())
+ 将该channel设置为非block类型,这里是不是与NIO很像。
(2) 调用channel()对channel初始化(见`分析init(channel)`)
(3) 将产生的NioServerSocketChannel注册到对应EventLoop上,见register()部分。

&emsp;regFuture.isDone()当且仅当执行NioServerSocketChannel.register(selector, SelectionKey)之后, 也就是将NioServerSocketChannel注册到parentGroup管理的NioEventLoop的selector上(代码见AbstractChannel.register0()), ChannelPromise状态才置为success。 后面会详细讲解doBind0函数。

<font size=6>分析init(channel)</font>
<p>代码实际会跑到ServerBootstrap.init():

```
            ChannelPipeline p = channel.pipeline(); //DefaultChannalPipeLine
            p.addLast(new ChannelInitializer<Channel>() {
            @Override
            public void initChannel(final Channel ch) throws Exception {//NioServerSocketChannel
                final ChannelPipeline pipeline = ch.pipeline();
                ChannelHandler handler = config.handler();
                if (handler != null) {
                    pipeline.addLast(handler);
                }
                ch.eventLoop().execute(new Runnable() {//ChannelInitializer和ServerBootstrapAcceptor都是Inbound,区别就是
                    @Override
                    public void run() {
                        pipeline.addLast(new ServerBootstrapAcceptor(
                                ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                    }
                });
            }
        });
```
主要干的事是向NioServerSocketChannel的DefaultChannalPipeLine对应的处理链添加ChannelInitializer(实际也是一个InBoundHandler), ChannelInitializer对于后面还有作用, 先留个印象。<br>此时, NioServerSocketChannel对应的pipeline中handler结构如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/PipeLine.png" height="200" width="450"/>

<font size=6>p.addLast</font>
<p>具体添加代码操作如下:

```
        synchronized (this) {
            newCtx = newContext(group, filterName(name, handler), handler);
            addLast0(newCtx);
            if (!registered) {//只有这个channel被register到某个具体的EventLoop后，才会考虑执行一些任务，这里考虑的任务是将对应的handler加入到对应的pipe中,DefaultChannelPipeline是与NioServerSocketChannel一一对应的
                newCtx.setAddPending();
                callHandlerCallbackLater(newCtx, true);
                return this;
            }
            EventExecutor executor = newCtx.executor();
            if (!executor.inEventLoop()) {
                newCtx.setAddPending();
                executor.execute(new Runnable() {
                    @Override
                    public void run() {
                        callHandlerAdded0(newCtx);
                    }
                });
                return this;
            }
        }
        callHandlerAdded0(newCtx);
        return this;
```

主要操作就是:
+ 新产生一个DefaultChannelHandlerContext, 主要作用就是存放对应的handler,也就是ChannelInitializer。
+ 将DefaultChannelHandlerContext添加进DefaultChannalPipeLine的倒数第二个, 也就是tail之前。
+ 如果NioServerSocketChannel并没有注册到对应的selector上(代码见AbstractChannel.register0()), 那么将生成PendingHandlerAddedTask, 并将该task线程放入pendingHandlerCallbackHead(属于Pipeline), 等待NioServerSocketChannel被注册到对应的selector时执行(见NioMessageUnsafe.register());
  若注册了, 那么调用callHandlerAdded0()->ChannelInitializer.initChannel(ChannelHandlerContext ctx)函数中,如下:
  ```
  @SuppressWarnings("unchecked")
    private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
        if (initMap.putIfAbsent(ctx, Boolean.TRUE) == null) { // Guard against re-entrance.
            try {
                initChannel((C) ctx.channel());
            } catch (Throwable cause) {
                // Explicitly call exceptionCaught(...) as we removed the handler before calling initChannel(...).
                // We do so to prevent multiple calls to initChannel(...).
                exceptionCaught(ctx, cause);
            } finally {
                remove(ctx); //初始化完善后，删除自身。又要把最开始注册的HelloServerInitializer删掉，也是ChannelInboundHandler类型
            }
            return true;
        }
        return false;
    }
  ```
  + 这里会进入ChannelInitializer.initChannel(final Channel ch)(详见ServerBootstrap.init()), 把会向pipeLine添加ServerBootstrapAcceptor的操作当成一个task, 传递给NilEventLoop, 等待执行形成一个最终的handler链。 传递的参数也可以注意下, 有childGroup、以及自定义的HelloServerInitializer。之后新建立的连接请求SocketChannel, 将根据这两个参数创建, 之后会详解(见NioEventLoop篇)。
  + 这里还需要注意ChannelInitializer.remove(ctx)会将该匿名ChannelInitializer(见ServerBootstrap.init())从NioServerSocketChannel的pipeline中删掉。这样NioServerSocketChannel对应的pipeline结构如下:
  <img src="https://kkewwei.github.io/elasticsearch_learning/img/PipeLine2.png" height="200" width="650"/>

<font size=6>register</font><p>
根据`config().group().register(channel)`进行注册, 首先这里的group()使用的是ParentGroup里面的EventLoop, 具体从EventLoop选取哪个EventLoop来与该channel绑定呢,使用的轮训策略。每次选取都会+1。 这里分两种决策策略:PowerOfTwoEventExecutorChooser和GenericEventExecutorChooser,都实现了+1的效果, 两者的唯一区别就是求余的效果不同:
当该Group定义的EventLoop为2^n时, PowerOfTwoEventExecutorChooser使用的是位运算的方式求余, 位运算能减少计算的时间复杂度。
```
public EventExecutor next() {
    return executors[idx.getAndIncrement() & executors.length - 1];
}
```
如何判断一个数是否为2^n呢, 方法如下:
```
private static boolean isPowerOfTwo(int val) {
    return (val & -val) == val;
}
```
选出一个NioEventLoop后, 最终会进入NioMessageUnsafe.register()中(是AbstractUnsafe的函数), 该对象在NioServerSocketChannel构造函数中生成。接着会进入AbstractUnsafe.register0(), 代码如下:
```
            try {
                // check if the channel is still open as it could be closed in the mean time when the register
                // call was outside of the eventLoop
                if (!promise.setUncancellable() || !ensureOpen(promise)) {
                    return;
                }
                boolean firstRegistration = neverRegistered;
                doRegister();
                neverRegistered = false;
                registered = true;
                pipeline.invokeHandlerAddedIfNeeded();

                safeSetSuccess(promise);
                pipeline.fireChannelRegistered();
                // Only fire a channelActive if the channel has never been registered. This prevents firing
                // multiple channel actives if the channel is deregistered and re-registered.
                if (isActive()) {
                    if (firstRegistration) {
                        pipeline.fireChannelActive();
                    } else if (config().isAutoRead()) {
                        // This channel was registered before and autoRead() is set. This means we need to begin read
                        // again so that we process inbound data.
                        //
                        // See https://github.com/netty/netty/issues/4805
                        beginRead();
                    }
                }
            } catch (Throwable t) {
                // Close the channel directly to avoid FD leak.
                closeForcibly();
                closeFuture.setClosed();
                safeSetFailure(promise, t);
            }
```
doRegister()函数将会跑到AbstractNioChannel.doRegister()里面, 如下:
```
        for (;;) {
            try {
                selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
                return;
            } catch (CancelledKeyException e) {
                if (!selected) {
                    // Force the Selector to select now as the "canceled" SelectionKey may still be
                    // cached and not removed because no Select.select(..) operation was called yet.
                    eventLoop().selectNow();
                    selected = true;
                } else {
                    // We forced a select operation on the selector before but the SelectionKey is still cached
                    // for whatever reason. JDK bug ?
                    throw e;
                }
            }
        }
```
这里是不是很熟悉? 不断地轮训注册, 将该channel注册到NioEventLoop上面的Selector上面, 并且select_ops置为0, 表示什么都不感兴趣。
我们先了解NioServerSocketChannel与NioSocketChannel的继承关系:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/ServerBootstrap1.png" height="200" width="450"/>
可以看到:
+ NioServerSocketChannel在初始化时候就声明readInterestOp为OP_ACCEPT, 而NioSocketChannel在初始化时声明readInterestOp为OP_READ。
+ NioServerSocketChannel对应NioMessage, 当bossdui对应的NioEventLoop为新的连接建立NioSocketChannel时, 都会进入NioMessage.read()进行初始化。当NioSocketChannel对应着NioByteUnsafe, 当work对应的NioSocketchannel接收到数据时, 会进入NioByteUnsafe.read()只来接收数据。

继续回到register()上,  channel与selector完成register之后:
1. 执行一些挂起的任务(invokeHandlerAddedIfNeeded()), 比如p.addLast所介绍的, 此时pileline对应的的handler链如下:HEAD->ServerBootstrapAcceptor->TAIL
2. 执行safeSetSuccess(promise), 最终会去调用AbstractBootstrap.doBind()里面介绍的ChannelFutureListener.operationComplete()函数, 注意doBind0()函数, 这里将完成channel与port的绑定和channel感兴趣事件为OP_ACCEPT,具体代码见AbstractChannel.bin(), 代码如下:
```
 boolean wasActive = isActive();
            try {
                 //doBind0最终调用channel.bind方法对执行端口进行绑定
                doBind(localAddress);
            } catch (Throwable t) {
                safeSetFailure(promise, t);
                closeIfClosed();
                return;
            }
            //之前没有绑定，现在绑定了，绑定的意思是NioServerSocketChannel里面的SocketChannel的Address有值了
            if (!wasActive && isActive()) {
                invokeLater(new Runnable() {
                    @Override
                    public void run() {
                        //最终修改的是NioServerSocketChannel的可Accept属性
                        pipeline.fireChannelActive();
                    }
                });
            }
```
NioServerSocketChannel在初始化时只是将readInterestOp赋值为OP_ACCEPT, 而被注册感兴趣OP_ACCEPT是在这里, 真正调用了AbstractNioCHannel.doBeginRead来实现注册
```
    @Override
    protected void doBeginRead() throws Exception {
        // Channel.read() or ChannelHandlerContext.read() was called
        final SelectionKey selectionKey = this.selectionKey;
        if (!selectionKey.isValid()) {
            return;
        }

        readPending = true;

        final int interestOps = selectionKey.interestOps();
        if ((interestOps & readInterestOp) == 0) { //将设置可接受
            selectionKey.interestOps(interestOps | readInterestOp);
        }
    }
```

自此NioServerSocketChannel已经初始化完成, NioServerSocketChannel拥有的pipeLine的里面的上下文:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/PipeLine1.png" height="300" width="550"/>
其中第二个Context的handler为ServerBootstrapAcceptor, 它的构造时的代码如下:
```
new ServerBootstrapAcceptor(ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs)
```
currentChildHandler就是我们自定的HelloServerInitializer, 该handler包含了我们所需要的所有逻辑,这些handler将在NioEventLoop篇构造NioSocketChannel时使用。

