---
title: ReentrantLock源码解读
date: 2018-09-23 19:47:30
tags:
---
ReentrantLock作为线程之间相互通信的工具, 在实际项目中较多的被使用到, 了解ReentrantLock, 就不得不提AbstractQueuedSynchronizer(AQS), 本文章将对这两个类展开详述。
# 简介
AbstractQueuedSynchronizer，顾名思义，抽象队列同步器，作为抽象类，使用FIFO链，实现了锁的语义, 在CountDownLatch、Semaphore都可以看到该类的实现。
## AbstractQueuedSynchronizer详解
接下来将首先介绍两个重要的属性变量:
`state`
AbstractQueuedSynchronizer主要针对属性state来实现锁的含义，用户通过针对state赋予不同的值，实现不同锁的含义。 在多线程针对state的操作，必须保证state状态的原子性，使用了`volatile`关键字，这里没使用Synchronized来保证原子性的原因:
+ state的状态修改不依赖历史的值，很适合volatile使用场景，设置了volatile后，也能保证state修改的可见性。
+ Synchronized实现互斥的成本要比volatile很高。
`Node`
AbstractQueuedSynchronizer实现了FIFO队列，该队列存放着目前阻塞的线程，每个元素都是都由一个Node构成，Node结构如下：
```
{
     volatile int waitStatus;
     volatile Node prev;
     volatile Node next;
     volatile Thread thread;
     Node nextWaiter;

}```

|属性|介绍|
|:-|:-|
|waitStatus|当前节点的状态:<br>CANCELLED:当前线程取消执行, 值为1，<br>SIGNAL:当该节点释放锁的时候,需要唤醒后继节点, 值为-1<br>CONDITION:当前节点在等待某种condition发生, 值为-2<br>PROPAGATE: 当前节点主要共享锁, 当节点设置为该值, 那么无条件向后传递锁释放的的信号, 值为-3<br>0: 表示初始状态
|prev|当前节点的前一个节点
|next|当前节点的后继节点|
|thread|当前节点所拥有的线程|
|nextWaiter|链接指向下一个处于condition队列的节点
AQS中等待锁的线程队列与运行线程结构如下:
<img src="http://owsl7963b.bkt.clouddn.com/AQS.png" height="250" width="300"/>
### ReentrantLock详解
ReentrantLock作为可重入的独享锁, 分为两类, 公平锁FairSync和非公平锁NonfairSync, 公平的体现在于: 当已经有线程处于等待状态时(等待队列不为空), 新来需要获取锁的线程能否可能插队先获取锁, 可以的话, 就是非公平锁; 不能立马获取到锁, 而必须排队的就是公平锁。
本文就以公平锁的获取与释放作为主线进行讲解。
```
    public static ReentrantLock lock = new ReentrantLock(true);
    lock.lock()
```
这样开始尝试获取锁, 实际调用的FairSync.acquire(1), 这里取值`1`的含义可以理解为尝试将state状态从0设置为1, 当status状态为0时, 说明是没有锁的。 真正调用的是AbstractQueuedSynchronizer函数的acquire:
```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            //说明本次退出中间有线程调用过thread.interrput(), 这里将中断信号复原。
            selfInterrupt();
    }
```
主要做了如下几个事情:
+ 尝试去获取锁, 若获取到了, 就直接退出。
+ 若没有获取到锁, 那么将当前线程构成一个Node, 放入线程阻塞队列, 线程进入睡眠等待。
+ 若本次没有获取到锁、从阻塞队列中被唤醒, 并且acquireQueued()返回true, 那么说明该线程被别人调用了中断, 我们需要将该中断再置位向外传递。(parkAndCheckInterrup()把中断信号清掉了)
### 尝试获取锁
代码在FairSync中定义了:
```
        protected final boolean tryAcquire(int acquires) {
             final Thread current = Thread.currentThread();
            int c = getState();//首先读取state变量
            if (c == 0) {
                 ////判断sate值是否为0,在这里0就表示初始状态
                if (!hasQueuedPredecessors() &&
                    //采用CAS原子操作修改sate的值，
                    compareAndSetState(0, acquires)) {
                    //如果修改成功，则将AQS的执行者设置为currentThread；这里的执行者其实就是获得执行权的线程
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //当前线程就是抢占线程，那么是可以直接进入的
            else if (current == getExclusiveOwnerThread()) {
                //注意这里，会+acquires，可重入式的，每次都得释放，不然锁就不会释放
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            //否则就不能获取到，获取不到锁，但是不会改变state的值
            return false;
        }
```
FairSync尝试获取锁的过程比较简单: 若status为0, 那么说明锁还没有被使用, 可以获取返回true, 否则就返回false, 我们需要注意的几个地方:
1. 若status为0, 这里会进行hasQueuedPredecessors()判断, 只有等待队列中没有节点, 才能立刻获取到, 这里可以体现公平锁FIFO的属性, 只要有线程处于等待队里, 那么该节点就该去等待
2. 第二个条件体现了可重入锁的性质, 只要获取锁的线程就是当前线程, 那么该线程照样可以获取到, 只是将state增加了。 同时说明, 同一线程两次调用lock.lock(), 那么一定需要两次lock.unlock()进行解锁才行。
3. 若当前线程获取不到锁, 是不会对status的值产生任何改变的。

#### 加入等待队列
若没有获取到锁, 则开始将线程加入等待队列addWaiter:
```
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        //尾插法，尾部不为空
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```
首先将当前线程作为参数构造等待节点Node, 传递的mode为EXCLUSIVE, 然后进行尾插法, 若等待队列不为空, 通过compareAndSetTail()原子操作将当前节点node设置为tail节点。
若目前还没有等待的节点, 那么构造等待队列:
```
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            //初始化，只要没值，先把头和尾给初始化了再继续
            if (t == null) { // Must initialize
                //头部应该是空，这里要设置成new node()
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                //尾部目前应该是t,然后尾部设置为node，里面尾部tail已经设置指向了node
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```
注意这里for()死循环, 直到当前节点构建出来了等待队列才会退出, 否则不停地重试(重试的原因是可能其他线程也在构造或者向等待线程插入节点, 允许操作失败), 构建完成后, 等待队列如下:
<img src="http://owsl7963b.bkt.clouddn.com/AQS1.png" height="200" width="250"/>
这里需要注意的是, 最开始的的时候, 会虚构出来了一个Node()作为head节点, 可以理解代表着当前拥有锁的那个线程对应的节点。
#### 设置等待队列
线程加入等待队列后, 是不能够立马跑去睡眠的, 还需要检查等待队列前继节点是否符合要求, 只有当前继节点waitState为SIGNAL, 那么本节点才可以去睡觉:
```
    //如果在整个等待过程中被中断过，则返回true，否则返回false。
    final boolean acquireQueued(final Node node, int arg) {
        //说明没有获取成功，退出时因为发生异常了
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) { //开始自旋
                final Node p = node.predecessor(); //查找前继节点
                //该节点前节点是头结点，并且获取到了锁,只要不满足这个条件，该节点将一直阻塞下去
                if (p == head && tryAcquire(arg)) {
                    setHead(node); //那么设置该节点为头结点
                    p.next = null; // help GC 抛弃该节点，等待被回收
                    failed = false;
                    return interrupted;
                } //说明没有获取到锁，是否需要睡眠等待
                if (shouldParkAfterFailedAcquire(p, node) &&
                   //等待又唤醒，可能是别人调用了LockSupport.unlock()，也有可能别人调用了thread.interrupt()唤醒的
                   parkAndCheckInterrupt())
                    //如果因为是被别人用thread.interrupt()唤醒的话，并不会退出并继续等待
                    interrupted = true;
            }
        } finally {
            if (failed)  //这里基本是执行不到的，除非遇到非运行时异常
                cancelAcquire(node);
        }
    }
```
该线程开始`自旋`, 主要做了如下检查:
+ 检查前继节点是否是head, 尝试获取锁(此时statue为0) ,若能够获取到锁, 说明头结点已经对应的那个线程已经释放了锁, 本节点又是作为等待队列排在最前面那个节点(head节点指向了释放锁那个线程), 直接获取锁。
+ 否则说明没有获取到锁, 那么检查该线程是否可以睡眠:
```
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;//头结点，waitState默认为0
        //前驱节点为消息通知模式，当释放锁或者取消时，会通知下个节点
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true; //那么本节点可以放心睡眠
        //前节点被取消了，自己加塞到前面，前继节点被无引用了，过会就会被丢弃
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev; //改变链路
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */ //
             //若是非cancel和非signal(比如任何节点加入时， statue都是0，等待后继节点改变)，将前节点设置为通知信号，等待被通知
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```
这里主要检查前继节点的waitStatus字段, 最前面介绍node时对取值也有详细的介绍:
1. 当前继节点为SIGNAL, 那么说明本节点可以放心去睡眠了, 因为前继节点的线程释放锁的时候, 会通过LockSupport.unlock()唤醒。
2. 若前继节点为cancel状态, 那么向前找, 直到找到一个不为cancel的节点, 并将为cancel的节点从整个等待队列中去掉, 以便gc回收。
3. 若找到一个非signal、非cancel的前继节点, 将该前继续节点状态置为signal, 以便前继节点唤醒后继节点。
在释放节点时(release()), 只要当前状态不为0, 就会唤醒后继节点。此时等待队列如下:
<img src="http://owsl7963b.bkt.clouddn.com/AQS2.png" height="200" width="250"/>

有个问题: 这里为啥在else那里不直接可以去睡眠呢?
假如前继节点释放锁的时候，此时发现自己不为SIGNAL，那么就不唤醒后继节点， 此时后继节点将自己设置为了SIGNAL， 那么此时设置也是无用的，形成了死等待。 所以自己在睡眠之前，再去检查下前继节点是否已经释放了锁，若释放了锁，就直接执行，没有释放锁，才能安慰睡觉。
+ 若可以睡眠了, 那么线程就通过LockSupport.park(this)进入睡眠。
```
    private final boolean parkAndCheckInterrupt() {
        //能否响应中断请求, 从等待中退出，但是不会抛出异常
        LockSupport.park(this);
        //检测当前线程是否有中断，若有中断, 那么清空中断,把信号向外传递
        return Thread.interrupted();
    }
```
这里我们需要知道, 该线程从睡眠中被唤醒, 有可能是通过LockSupport.unpark(this)、也有可能是通过thread.interrupt()方式唤醒的, 第一种唤醒是有意义的, 对于第二种唤醒并没有意义,我们在acquireQueued中自旋时会忽略这种情况。
至此, 获取锁的过程已经全部完成, 整体过程如下:
<img src="http://owsl7963b.bkt.clouddn.com/AQS3.png" height="250" width="800"/>
要么获取到锁, 那么线程进入等待队列安心睡眠。
### 释放锁
释放锁只需要调用sync.release(1)就行了, 实际调用的AbstractQueuedSynchronizer里面的函数:
```
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            //当前节点为signal状态，需要唤醒后继节点
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
主要做了两个事情:
+ 尝试将状态复位, 比如status置为0, 排他线程置为null.
+ 若等待队列有节点, 并且当前节点不为0(初始化), 那么就会去尝试唤醒后继一个有效的节点:
```
      private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        //置零当前线程所在的结点状态，允许失败。
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        //节点被取消了，cancel 才大于1
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                //从后向前找，找到最近一个有效的节点
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null) //反正叫醒后继节点
            LockSupport.unpark(s.thread);
    }
```
唤醒后继节点也是比较简单的:
1. 首先将本节点waitStatus置为0(初始值)
2. 如果后继节点被取消了(waitStatus>0), 那么在后继节点中找到一个最靠近的、非cancel状态的节点, 然后唤醒这个节点上的线程。 这里不用将cancel状态的节点从队列中去掉, 在节点尝试获取锁的时候会自动干这个事。
### 总结
线程在获取锁的时候, 主要根据ReentrantLock里面的状态status来识别是否可以获取锁, 若为0, 那么锁未被获取; 若为1, 说明锁被一个线程获取; 若大于1, 说明发生了线程重入。 若没有获取到, 则将自己加入等待队列, 然后睡眠。 线程在释放锁时, 也会唤醒等待队列排在前面的线程。