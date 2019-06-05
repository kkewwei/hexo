---
title: NioEventLoop篇
date: 2018-01-22 08:53:40
tags: NioEventLoop
toc: true
---
# 介绍
在ServerBootstrap初始篇初始篇中说过, 每个NioEventLoop里面, 都拥有属性thread, 用来执行对应channel产生的所有task, 该thread最早在register的时候被生成, 首先调用如下代码:
```
            ch.eventLoop().execute(new Runnable() {
                    @Override
                    public void run() {...}
            });
```
调用NioEventLoop.execute(), 然后进入到SingleThreadEventExecutor.execute(NioEventLoop的父类), 执行如下代码:
```
 public void execute(Runnable task) {
        boolean inEventLoop = inEventLoop();
        if (inEventLoop) {
            addTask(task);
        } else {
            startThread();
            addTask(task);
            if (isShutdown() && removeTask(task)) {
                reject();
            }
        }
        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }
```
首先判断NioEventLoop里面的那个线程是否启动, 若是的话, 就将当前task放进任务队列; 否则说明NioEventLoop里面执行task的那个唯一线程还没有启动, 调用startThread来启动。
## startThread
startThread用来启动NioEventLoop里面的执行线程,代码如下:
```
  executor.execute(new Runnable() { //就是一个执行器，ThreadPerTaskExecutor。只要想，可以一直启动
            @Override
            public void run() {
                thread = Thread.currentThread(); //获取当前这个线程
                if (interrupted) {
                    thread.interrupt();
                }
                boolean success = false;
                updateLastExecutionTime();
                try {
                    SingleThreadEventExecutor.this.run(); //调用NioEventLoop里面run,进行无限循环
                    success = true;
                } catch (Throwable t) {
                }
```
executor实际是ThreadPerTaskExecutor, execute将跑到ThreadPerTaskExecutor.execute():
```
 @Override
    public void execute(Runnable command) {
        threadFactory.newThread(command).start();
    }
```
这里真正唤醒了线程new Runnable()后, 这个线程就是NioEventLoop线程的核心部分, 该线程生命周期很长, 即使执行发生异常, 也不会主动退出。
因为NioEventLoop对应的线程比较重要, 弄清楚如何启动该线程对我们了解很有帮助, 我们来捋一捋这个过程,下图是涉及到的类及函数
<img src="https://kkewwei.github.io/elasticsearch_learning/img/NioEventLoop1.png" height="300" width="450"/>
1. 首先eventLoop.execute(), 主函数返回。
2. 进入了SingleThreadEventExecutor.execute(), 首先检查thread变量是否为null, 若为空并且检查state状态为not_started, 代表没有启动, 则调用executor.execute()
3. 调用executor.execute()后, 产生线程并启动, 线程的run()如上所示, 会对thread赋值, 然后调用NioEventLoop.run()开始死循环执行。

# NioEventLoop
NioEventLoop作为Netty多线程的重要类, 我们可以将其看成一个只有一个线程的线程池
task分为两类任务: 非IO型和IO型, 它们的执行时间比例由ioRatio参数控制, 默认50%,非IO型执行时间 = IO型执行时间。
+ 非IO型: 本进程内, 别的线程发送的请求, 比如将新的Context(hanlder)添加到Pipieline中等等(代码见文章开头`ch.eventLoop().execute`)
+ IO型: Accetp、Write、read等从远程节点发送过来的请求。

为了更好地理解代码, 我们需要大致了解selector.wakeup()的作用:
+ 若当前线程有由于调用selector.select()/selector.select(time)阻塞的, 那么当调用selector.wakeup()后会被立刻唤醒。
+ 若当前没有线程因为selector.select()/selector.select(time)而阻塞的函数, 当调用selector.wakeup()后, 会对下次调用selector.select()/selector.select(time)/selector.selectNow()立刻返回, 而不会被阻塞。

NioEventLoop.run()作为执行所有task执行任务的核心, 主要处理逻辑如下:

```
  @Override
    protected void run() {
        for (;;) {
            try {
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hahasTaskssTasks())) {
                    case SelectStrategy.CONTINUE:
                        continue;
                    case SelectStrategy.SELECT:
                        select(wakenUp.getAndSet(false));
                        // 'wakenUp.compareAndSet(false, true)' is always evaluated
                        // before calling 'selector.wakeup()' to reduce the wake-up
                        // overhead. (Selector.wakeup() is an expensive operation.)
                        //
                        // However, there is a race condition in this approach.
                        // The race condition is triggered when 'wakenUp' is set to
                        // true too early.
                        //
                        // 'wakenUp' is set to true too early if:
                        // 1) Selector is waken up between 'wakenUp.set(false)' and
                        //    'selector.select(...)'. (BAD)
                        // 2) Selector is waken up between 'selector.select(...)' and
                        //    'if (wakenUp.get()) { ... }'. (OK)
                        //
                        // In the first case, 'wakenUp' is set to true and the
                        // following 'selector.select(...)' will wake up immediately.
                        // Until 'wakenUp' is set to false again in the next round,
                        // 'wakenUp.compareAndSet(false, true)' will fail, and therefore
                        // any attempt to wake up the Selector will fail, too, causing
                        // the following 'selector.select(...)' call to block
                        // unnecessarily.
                        //
                        // To fix this problem, we wake up the selector again if wakenUp
                        // is true immediately after selector.select(...).
                        // It is inefficient in that it wakes up the selector for both
                        // the first case (BAD - wake-up required) and the second case
                        // (OK - no wake-up required).

                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                    default:
                        // fallthrough
                }
                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                if (ioRatio == 100) {
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        runAllTasks();
                    }
                } else {
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
```
selector.wakeup()是一个非常耗时的操作, 需要通过wakenUp变量标记在合适的时候调用selector.wakeup()来唤醒selector.select(), 当需要唤醒时, 标记为true, 就调用调用selector.wakeup()
NioEventLoop.run()部分的逻辑还是比较清楚:
(1) 检查是否累计有task:
```
@Override
    public int calculateStrategy(IntSupplier selectSupplier, boolean hasTasks) throws Exception {
        return hasTasks ? selectSupplier.get() : SelectStrategy.SELECT; //若当前没有非IO类型task时，需要
    }
```
进入selectStrategy.calculateStrategy(), 如果没有非IO task, 那么直接跳掉SelectStrategy.SELECT, 开始select; 若有task, 则立刻去执行task:
```
int selectNow() throws IOException {//相当于复写了NIO的select函数
        try {
            return selector.selectNow(); //返回的0, 直接跳出switch循环
        } finally {
            // restore wakeup state if needed
            if (wakenUp.get()) {
                selector.wakeup();
            }
        }
    }
```
若wakenUp置为true, 顺便执行selector.wakeup()使selector处于唤醒状态。
(2) 若有task, 进入select(wakenUp.getAndSet(false))进行等待。
wakenUp标志为false, 意味着新的一轮刚开始。NioEventLoop.select()与Selector.select()有异曲同工之处, 都是等待task出现, 主要代码如下:
```
private void select(boolean oldWakenUp) throws IOException {
        Selector selector = this.selector;
        try {
            int selectCnt = 0;
            long currentTimeNanos = System.nanoTime();
            long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos); //第一个任务执行执行的时间，绝对时间
            for (;;) {  //timeoutMillis下次等待需要的时间
                long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;//如果延迟任务队列中第一个任务开始执行的时间距离现在已经过了1ms,则小于0   1ms = 1000, 000ns
                if (timeoutMillis <= 0) {//距离第一个执行计划开始时间已经过了（1ms）
                    if (selectCnt == 0) { //selectCnt 用来记录selector.select方法的执行次数和标识是否执行过selector.selectNow()
                        selector.selectNow();
                        selectCnt = 1;
                    }
                    break;
                }
                // If a task was submitted when wakenUp value was true, the task didn't get a chance to call
                // Selector#wakeup. So we need to check task queue again before executing select operation.
                // If we don't, the task might be pended until select operation was timed out.
                // It might be pended until idle timeout if IdleStateHandler existed in pipeline.
                if (hasTasks() && wakenUp.compareAndSet(false, true)) {
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }
                int selectedKeys = selector.select(timeoutMillis);
                selectCnt ++;
                //如果已经存在ready的selectionKey，或者selector被唤醒，或者taskQueue不为空，或则scheduledTaskQueue不为空，则退出循环
                if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                    // - Selected something,
                    // - waken up by user, or
                    // - the task queue has a pending task.
                    // - a scheduled task is ready for processing
                    break;
                }
                long time = System.nanoTime();//selector.select(timeoutMillis);
                if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                    // timeoutMillis elapsed without anything selected.
                    selectCnt = 1;
                } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {//在某个周期内如果连续N次空轮询，则说明触发了JDK NIO的epoll死循环bug。
                    // The selector returned prematurely many times in a row.
                    // Rebuild the selector to work around the problem.
                    rebuildSelector();
                    selector = this.selector;
                    // Select again to populate selectedKeys.
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }
                currentTimeNanos = time;
            }

        } catch (CancelledKeyException e) {
        }
    }

```
可以看出代码做了如下事情:
2.1 从schedule task中取出顶点task的截止执行时间(最早开始执行那个scedule task), 若没有task的话, 超时时间取值1s。截止时间 = 此刻+超时
如果当前时间 > 截止时间 + 0.5ms的话,就立刻退出执行task。
2.2 检查当前是否有task排队。若有而且wakenUp为false, 则置位wakeup, 并唤醒selector, 并立刻退出。
+ 此时已经有task, 那么需要开始执行具体放入task, 如果不检查的话, 则进入selector.select(timeoutMillis)阻塞直到超时, 但这是没有必要的。
+ 若wakenUp为true, 代表着什么含义? 表示当前有别的线程唤醒了selector, 并向队列中放入了task, 那么执行selector.select(timeoutMillis)时会立刻返回。
别的线程向队列中添加task见(SingleThreadEventExecutor.execute(NioEventLoop的父类)部分), 其中唤醒selector的代码如下:
 ```
protected void wakeup(boolean inEventLoop) { //inEventLoop说的是NioEventLoop还没有启动
        if (!inEventLoop && wakenUp.compareAndSet(false, true)) {
            selector.wakeup();
        }
    }
 ```
 当且此时wakenUp为false才唤醒, 意味着什么呢? 此时还没有task添加过, 只用在这一轮switch第一个来的task的时候需要唤醒, 当再有任务来的时候, 没必要再次执行耗时的selector.wakeup()。
2.3 执行selector.select(timeoutMillis)
+ 若selector并没有唤醒(selector.wakeup()还在生效), 说明并没有task来, 并不需要唤醒。
+ 若若selector处于唤醒状态, 则说明此轮循环中有来的task, 需要立刻执行task。
+ 若阻塞了一段时间, 有task来, 别的线程执行了wakeup(boolean inEventLoop)函数, 阻塞也会立刻返回。

2.4 检查是否需要跳出循环:
+ 有IO task了
+ 上一轮的oldWakenUp仍然置为着, 说明因为上一轮走完, selector仍然处于唤醒状态, 需要这个唤醒作用清空。
+ 此轮有task处于penging.
+ 有schedule task截止时间已经到了。
select(boolean oldWakenUp)主要判断逻辑基本已经完成了, 为啥后面还有那么多代码? 主要是为了解决可能触发epool cpu100%的bug。这个bug的意思是selector.select(timeoutMillis)并不会超时阻塞timeoutMillis, 它会立刻返回。
这样的话, 这个函数也就失去了意义, 如果不加控制的话, 这里的for循环会无限制下去而没有意义。 解决的方法就是selector, 具体处理函数rebuildSelector0如下:
```
ivate void rebuildSelector0() {
        newSelectorTuple = openSelector();//打开一个新的Selector
        // Register all channels to the new Selector.
        int nChannels = 0;
        for (SelectionKey key: oldSelector.keys()) {////SelectionKey无效或者已经注册上了则跳过
            Object a = key.attachment();
            try {
                if (!key.isValid() || key.channel().keyFor(newSelectorTuple.unwrappedSelector) != null) {
                    continue;
                }
                int interestOps = key.interestOps();
                key.cancel();//取消SelectionKey
                SelectionKey newKey = key.channel().register(newSelectorTuple.unwrappedSelector, interestOps, a);
            }
        }
        selector = newSelectorTuple.selector;//用新的Selector替换老的Selector
        unwrappedSelector = newSelectorTuple.unwrappedSelector;
```
主要过程就是新建一个selector, 并且将原来selector等待时间都迁移过来。
如何判断是否触发了epool cpu100%的bug? 则是通过执行selector.select()函数的次数selectCnt来判断, 若当前循环次数超过`SELECTOR_AUTO_REBUILD_THRESHOLD`则说明触发了, 默认为512次。

select(wakenUp.getAndSet(false))完成后,会有这段代码
```
if (wakenUp.get()) {
        selector.wakeup(); //下次
 }
```
参考提示, 始终是想不明白这里代码的作用, 并且认为是多余的,作者的本意是为了当wakenUp为true时, selector始终处于醒着的状态, 同时在不合适的时候被阻塞。我们来反推这里代码的不合理。
假设task来了, 而selector.selector()却被阻塞没有返回, 而改代码前面存在这样的检查:
```
if (hasTasks() && wakenUp.compareAndSet(false, true)) {
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }
```
那么wakenUp只能为true, 并且selector处于阻塞状态。 而在该函数新一轮调用开始, wakenUp刚被置为为false, 从false -> 变为true, 不可能是同个函数中的下面的代码执行导致的(若执行了会立刻退出)
```
     if (hasTasks() && wakenUp.compareAndSet(false, true)) {//若果当前有task，并且是可以叫醒的，则中断selector.select
                    selector.selectNow();//selectNow()返回，否则会耽误任务执行
                    selectCnt = 1;   //
                    break;
                }
```

只可能是task来了, 同时执行了wakenUp.compareAndSet(false, true)代码 ,那么一定会执行selector.wakeup()部分, 那么selector.selector()一定会立刻返回。。
所以说, 那部分代码是没有没有意义的。

(3) 开始执行IO task和非IO task
前面也提到了, 两种任务执行的时间是成比例的, 非IO任务执行的时间 由IO任务执行的时间*比例。
## IO任务执行processSelectedKeysPlain
processSelectedKeysPlain根据selector.selectedKeys()获取到所有的IO事件,然后轮训每一个事件,对于每个事件主要处理逻辑processSelectedKey如下:
```
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
        eventLoop = ch.eventLoop();
        try {
            int readyOps = k.readyOps();
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);
                unsafe.finishConnect();
            }
            if ((readyOps & SelectionKey.OP_WRITE) != 0) { //如果是写
                ch.unsafe().forceFlush();
            }
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read(); //这里很重要，NioMessageUnsafe
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
```
这里主要逻辑是判断当前IO task的类型, 然后分别处理, 我们重点分析Accept 和read两种类型的task(这两部分的处理都抽象成read()函数)
 ###  SelectionKey.OP_ACCEPT部分
 此时实际从unsafe.read()进入的代码如下(NioMessageUnsafe.read()里面
```
            assert eventLoop().inEventLoop();
            final ChannelConfig config = config(); //NioServerSocketChannelConf
            final ChannelPipeline pipeline = pipeline();//DefaultChannelPipeline
            final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
            allocHandle.reset(config);

            boolean closed = false;
            Throwable exception = null;
            try {
                try {
                    do { //                        // 此处会调用到NioServerSocketChannel中的doReadMessages方法
                        int localRead = doReadMessages(readBuf);//将会产生一个NioSocketChannel建立C-S连接
                        if (localRead == 0) {
                            break;
                        }
                        if (localRead < 0) {
                            closed = true;
                            break;
                        }

                        allocHandle.incMessagesRead(localRead);
                    } while (allocHandle.continueReading()); //当前连接是否该继续
                } catch (Throwable t) {
                    exception = t;
                }

                int size = readBuf.size();
                for (int i = 0; i < size; i ++) {
                    readPending = false;//// 对每个连接调用pipeline的fireChannelRead
                    pipeline.fireChannelRead(readBuf.get(i));//回调到DefaultChannelPipeline里面
                }
                readBuf.clear(); //// 清理获取到的数据，下次继续使用该buf
                allocHandle.readComplete();
                pipeline.fireChannelReadComplete();

                if (exception != null) {
                    closed = closeOnReadError(exception);

                    pipeline.fireExceptionCaught(exception);
                }

                if (closed) {
                    inputShutdown = true;
                    if (isOpen()) {
                        close(voidPromise());
                    }
                }
            } finally {
                // Check if there is a readPending which was not processed yet.
                // This could be for two reasons:
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelRead(...) method
                // * The user called Channel.read() or ChannelHandlerContext.read() in channelReadComplete(...) method
                //
                // See https://github.com/netty/netty/issues/2254
                if (!readPending && !config.isAutoRead()) {
                    removeReadOp();
                }
            }
```
1. 循环遍历所有的accept请求, doReadMessages对每个请求做具体的具体,实现类在NioServerSocketChannel.doReadMessages中:
```
         SocketChannel ch = SocketUtils.accept(javaChannel()); //接受连接请求，产生一个SocketChannelImpl，
        try {
            if (ch != null) {
                buf.add(new NioSocketChannel(this, ch)); //这里就是新产生的NioSocketChannel,ch=SocketChannel
                return 1;
            }
        } catch (Throwable t) {
        }

        return 0;
```
同时遍历的时候设置了当前此轮循环处理的请求,不能超过maxMessagesPerRead,默认16个
SocketUtils.accept产生的SocketChannel是不是在NIO中很常见的方法, 产生具体的SocketChannelImp连接, 将该链接包装成NioSocketChannel, 然后放在readBuf中。
NioSocketChannel初始化, 默认监听的事件为SelectionKey.OP_READ, 同时自动拥有如下属性:
```
         this.parent = parent; //NioServerSocketChannel
        id = newId();// 分配一个全局唯一的id，默认为MAC+进程id+自增的序列号+时间戳相关的数值+随机值
        unsafe = newUnsafe(); // 初始化Unsafe, NioSocketChannel对应的是NioSocketChannelUnsafe
        pipeline = newChannelPipeline();//// 初始化pipeline, pipiline里面默认只拥有head和tail上下文事件,。
```

2.对产生的每个NioSocketChannel进行初始化, 使其设置为监听事件为SelectionKey.OP_READ。
初始化的时候, 首先调用NioServerSocketChannel的pipieline.fireChannelRead(), 开始遍历pipeLine上每个Context, 调用每个Context上面的channelRead()函数, 从HeadContext开始:
 ```
public final ChannelPipeline fireChannelRead(Object msg) { //msg是新建立的SocketChannel
        AbstractChannelHandlerContext.invokeChannelRead(head, msg); //fireChannelRead方法只是简单的往后传递事件，最终目的是向链中添加了
        return this;
    }
```
读每个Context上面执行channelRead()都以下面函数为开头, 注意该函数是以`static`注释的。
```
 static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) { ////msg是新建立的NioSocketChannel
        final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next); //pipe是同一个，
        EventExecutor executor = next.executor(); //executor = NioEventLoop， 因为
        if (executor.inEventLoop()) { //本线程是否是EventLoop线程
            next.invokeChannelRead(m); //DefaultChannelHandlerContext， 即为下面这个类
        } else {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeChannelRead(m);
                }
            });
        }
    }
````
2.1 会以根据HeadContext开始讲起:
+ 首先根据HeadContext, 找到对应的executor: 没若有, 找到对应HeadContext拥有的pipeLine, 返回该pipeLine的executor, 也就是NioServerSocketChannel的NioEventLoop。
```
 public EventExecutor executor() {  //若为空，就返回该pipLine拥有的chanel的executor， 即NioEventLoop
        if (executor == null) {
            return channel().eventLoop();
        } else {
            return executor;
        }
    }

```
+ 确定该线程即是NioEventLoop里面的执行线程, 然后调用该head的invokeChannelRead(), 但是head的invokeChannelRead()并不做任何事,仅仅是找到下一个拥有in属性的Context(即DefaultChannelHandlerContext, 即拥有handler为ServerBootstrapAcceptor)  ,然后向下传递invokeChannelRead, 会从头开始执行前面介绍的`static void invokeChannelRead`
static void invokeChannelRead
```
 private AbstractChannelHandlerContext findContextInbound() { //从Head当前位置找，直到向后找到一个inbound的，就退出
        AbstractChannelHandlerContext ctx = this;
        do {
            ctx = ctx.next;//直接找下一个
        } while (!ctx.inbound);
        return ctx;
    }
```
到第二个Context, 其中会执行`((ChannelInboundHandler) handler()).channelRead(this, msg)`, 即ServerBootstrapAcceptor.channelRead(), 如下所示:
```
 public void channelRead(ChannelHandlerContext ctx, Object msg) {
            final Channel child = (Channel) msg; //// child = NioSocketChannel
            child.pipeline().addLast(childHandler);
            setChannelOptions(child, childOptions, logger);
            for (Entry<AttributeKey<?>, Object> e: childAttrs) {
                child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
            }
            try {  // 将连接注册到childGroup中（也就是我们常说的workGroup)，注册完成如果发现注册失败则关闭此链接
                childGroup.register(child).addListener(new ChannelFutureListener() {   ///这里使用的是childGroup
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        if (!future.isSuccess()) { //如果有连接完成，但是失败的情况下
                            forceClose(child, future.cause());
                        }
                    }
                });
            } catch (Throwable t) {
            }
```
主要做的事:
+ 其中第三行的childHandler是在外层向ServerBootstrap添加的自定义处理链(比如b.childHandler(new HelloServerInitializer()))里面的handler。 此时该channel的PipeLine链上共有三个Context, 分别是HeadContext, HelloServerInitializer, TailContext.
+ 从childGroup里面轮训选择一个NioEventLoop, 将这个NioSocketchannel绑定到该NioEventLoop上面。
+ 当注册完成后, 会执行这个ChannelFutureListener, 基本什么都不会做。

其中第二步骤, 注册的代码在`ServerBootStrap初始篇`中已经展示, 为了讲解方便在此再次罗列:
```
                 boolean firstRegistration = neverRegistered;
                doRegister(); // AbstractNioChannel,// 真正的注册方法，只是将channel.regester注册到对应EventLoop的selector中
                neverRegistered = false;
                registered = true;// register状态设置为true，

                // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
                // user may already fire events through the pipeline in the ChannelFutureListener.
                pipeline.invokeHandlerAddedIfNeeded();

                safeSetSuccess(promise); //设置安全后，会去主动调用operationComplete()，会触发channel状态修改从0->accept
                pipeline.fireChannelRegistered();//// NioServerSocketChannel管道已经注册到EventLoops上了触发channelRegistered事件，
                // Only fire a channelActive if the channel has never been registered. This prevents firing
                // multiple channel actives if the channel is deregistered and re-registered.
                if (isActive()) {  //将回到NioServerSocketChannel.isActive()中,   // 第一次注册时触发fireChannelActive事件，防止deregister后再次register触发多次fireChannelActive调用
                    if (firstRegistration) {
                        pipeline.fireChannelActive();//// 这里和前面的ServerSocketChannel分析一样,最终会触发unsafe.beginRead()
                    } else if (config().isAutoRead()) {
                        // This channel was registered before and autoRead() is set. This means we need to begin read
                        // again so that we process inbound data.
                        //
                        // See https://github.com/netty/netty/issues/4805
                        beginRead();
                    }
                }
```
其中需要注意的是:
+ invokeHandlerAddedIfNeeded()会执行handlerAdded任务, 具体会执行到我们自定的编解码模板, 也就是HelloServerInitializer里面通过initChannel添加的channel, 接着会执行remove(ctx), 将HelloServerInitializer对应的Context从PipeLine中去掉, 此时队列中拥有的context如下:
HeadContext-> EncoderContext->DecoderContext->SelfCustemHanderContext->TailContext.
+ 会进入到pipeline.fireChannelActive(),  如同前面讲述的会对每个Context执行channelActive()一样, 这里也会对每个Context执行channelActive(), 其中HeadContext.channelActive()需要提一下:
```
 public void channelActive(ChannelHandlerContext ctx) throws Exception {
            ctx.fireChannelActive();

            readIfIsAutoRead(); //最终修改的是NioServerSocketChannel的可读属性
        }
```
ctx.fireChannelActive()调用的所有Context并不会做什么时, 但是该HeadContext.readIfIsAutoRead()需要我们值得注意下, 会从TailContext向前执行context.read(), 直达HeadContext.read需要我们注意下, 会执行doReadBegin
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
每个NioSocketChannel初始话的时候, readInterestOp被赋值为SelectionKey.OP_READ, 此时直接也将selectionKey赋值为可读。 基本初始化新建立的NioSocketChannel完成了。
### SelectionKey.OP_READ
 此时实际从unsafe.read()进入的代码如下(NioByteUnsafe.read()里面, 该模块涉及到自定义的编解码模块, 将在`Netty通信编解码源码解读`讲解。

## 执行非IO Task.
进入runAllTasks函数执行非IO task, timeoutNanos指的当前执行task最多使用的时间, 过程如下:
```
protected boolean runAllTasks(long timeoutNanos) {//处理非I/O任务。
        fetchFromScheduledTaskQueue();
        Runnable task = pollTask();//从
        if (task == null) {
            afterRunningAllTasks();  //SingleThreadEventLoop.afterRunningAllTasks()
            return false;
        }
        //截止时间=ScheduledFutureTask当前相对时间+ 超时
        final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
        long runTasks = 0;
        long lastExecutionTime;
        for (;;) {
            safeExecute(task);  //顺序执行所有task

            runTasks ++;

            // Check timeout every 64 tasks because nanoTime() is relatively expensive.
            // XXX: Hard-coded value - will make it configurable if it is really a problem.
            if ((runTasks & 0x3F) == 0) {  //当64个task后
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                if (lastExecutionTime >= deadline) {//当前时间超过截止时间，那么就退出
                    break;
                }
            }

            task = pollTask();
            if (task == null) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                break;
            }
        }

        afterRunningAllTasks();
        this.lastExecutionTime = lastExecutionTime;
        return true;
    }
```
主要做了如下几件事:
+ 从schedule队列取出任务向taskQueue中存放, 是一个有size<=16的、根据截至时间有优先级的阻塞队列。
+ 从taskQueue中取出最早执行的那个task, 开始执行, 每当执行64个task退出一次,处理IO task.

NioEventLoop核心函数及 OP_READ、OP_ACCEPT等基本讲完了。