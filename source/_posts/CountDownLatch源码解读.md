---
title: CountDownLatch源码解读
date: 2018-09-24 16:47:06
tags:
---
CountDownLatch也是线程同步的一个工具, 底层也是使用AQS(参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/09/23/ReentrantLock%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">ReentrantLock源码解读</a>)来进行锁的互斥。
CountDownLatch与ReentrantLock的主要区别是:
+ CountDownLatch是一个共享锁, ReentrantLock是一个独占锁。
+ CountDownLatch中state初始值为n, 代表一个锁被分成了n份。 好比门钥匙, 有一个门需要n份钥匙聚齐后才能打开。 若门打开后, 第一个通过的那个人可以告诉排队等待的人, 然后依次经过。而ReentrantLock中state为0, 表示锁没有被占用, 比如有一个很窄的门, 每次只能通过一个人, 虽拥有那个钥匙(state=1), 谁才能过那道门。 若有很多人等, 那么就要排队了。 若新来一个人来时, 门恰好是开着的, 他能忽略排队的人过去的话, 这就是非公平锁, 若需要进入等待队列的话, 那就是公平锁。
CountDownLatch并不存在公不公平锁的概念, CountDownLatch的这个门打开后, 进门的并发并没有限制, 任何人只要发现门打开了, 就可以进入。 而ReentrantLock对应的门, 设置了每次进门的并发只能是1, 所以需要排序进入。
CountDownLatch使用方法如下:
```
CountDownLatch countDownLatch = new CountDownLatch(3);
countDownLatch.await();
countDownLatch.countDown();
```
await()方法检查state是否为0, 不为0则阻塞当前线程, countDown()把当前state减一。
## countDown()
countDown()对state减1一, :
```
    public final boolean releaseShared(int arg) {
          //如果state为0， 那么就说明可以唤醒await()了
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```
主要做了如下两个事情:
+ 对state减一
+ 若state为0, 那么开始唤醒睡眠的线程, 唤醒过程如下:
```
    private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                //若是singal，那么就会通知
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                //按道理这种情况不会发生，若发生了，那么下个节点要无条件传播
                else if (ws == 0 && !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            // 如果发生了变动，那么再继续释放，可能存在多个线程同时释放等待队列的线程
            if (h == head)                    // loop if head changed,
                break;
        }
    }
```
这个函数在countDown()和await()中都会被调用(在doAcquireSharedInterruptibly()中, 当阻塞线程从LockSupport.park(this)中醒来, 就会调用), 注意这里是一个死循环, 从头结点开始检查每个node的waitStatus, 直到等待队列没有要唤醒的线程为止, 主要做了如下判断:
+ 若节点waitStatus为signal, 那么就设置当前节点为0(初始化节点), 并且通过unparkSuccessor()唤醒等待队里里面的后继节点(该函数可以参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/09/23/ReentrantLock%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">ReentrantLock源码解读</a>)。
+ 若当前节点waitStatus为0是(初始化状态, 比如刚将头结点从signal变成了0), 那么设置h为PROPAGATE, 表示状态需要向后传递。 实际查找代码, 并没有发现哪里显示使用Node.PROPAGATE这个条件的, 这步实际并没有看出存在的意义。
+ 若h==head, 说明tail==head, 所有节点已经唤醒。那么此时才可以退出。
需要知道的是, 若节点对应的线程从等待队列中唤醒, 节点此时并没有从等待队列中去掉, 实际在await()中从等待队列中去掉而被回收的。


## await()
await()实际就是检查state是否为0, 若不为0, 那么本节点就加入等待队列中。
```
    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0) //如果为0， 就说明获取到了，不为0， 则说明没有获取到
            doAcquireSharedInterruptibly(arg);
    }
```
主要做了如下事情:
+ 首先检查是否有中断信号, 若有的话, 就直接抛异常, 否则LockSupport.lock()就会被直接唤醒而没有意义(中断信号就可以直接使lock()失效)。
+ 检查state是否为0, 若为0, 就说明直接获取了锁。这里可以体现CountDownLatch并没有公平锁的概念
+ 若state > 0, 则需要将该线程加入等待队列。
加入等待队列在doAcquireSharedInterruptibly中完成的。
```
    private void doAcquireSharedInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.SHARED); //创建共享锁
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) { //如果前继节点是头结点，
                    int r = tryAcquireShared(arg); //如果获取到锁， 一定值大于0的
                    if (r >= 0) { //如果为0，就说明可以退出了
                        setHeadAndPropagate(node, r); //向下传播释放锁的信号，
                        p.next = null; // help GC
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
注意该函数中p.next = null操作, 此时p已经从等待队列链中完全脱离了, 该节点就可以等待gc回收了。该函数做了如下事情:
+ 首先将该节点以SHARED方式创建节点, 并加入等待队列。在addWaiter()中实现, 参数为Node.SHARED, 此时等待队列如下:
<img src="http://owsl7963b.bkt.clouddn.com/CountDownLatch1.png" height="200" width="450"/>
+ 开始自旋, 进行判断:
1. 若当前节点的前继节点是head, 并且state=0, 那么说明该线程获取到了锁, 重新设置head, 并且向后传播(setHeadAndPropagate)。
2. 通过调用shouldParkAfterFailedAcquire判断是否可以直接睡眠(可参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/09/23/ReentrantLock%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">ReentrantLock源码解读</a>), 若可以的话, 就直接去睡眠。

向后传播通过setHeadAndPropagate()完成, 也是比较简单的:
```
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node); //node前继节点就是头结点
        /*
         * Try to signal next queued node if:
         *   Propagation was indicated by caller,
         *     or was recorded (as h.waitStatus either before
         *     or after setHead) by a previous operation
         *     (note: this uses sign-check of waitStatus because
         *      PROPAGATE status may transition to SIGNAL.)
         * and
         *   The next node is waiting in shared mode,
         *     or we don't know, because it appears null
         *
         * The conservatism in both of these checks may cause
         * unnecessary wake-ups, but only when there are multiple
         * racing acquires/releases, so most need signals now or soon
         * anyway.
         */
        if (propagate > 0 || h == null || h.waitStatus < 0 || //如果
            (h = head) == null || h.waitStatus < 0) { //这里是重复的
            Node s = node.next;
            if (s == null || s.isShared()) //本节点是共享的
                doReleaseShared();
        }
    }
```
首先修改头结点, 其次判断判断后继节点是否是共享的(nextWaiter == SHARED), 前面可知, 每个线程构造等待节点时, 传递的nextWaiter=SHARED, 也恰好满足条件。共享锁唤醒操作在await()里有介绍(doReleaseShared())。

## 超时等待
在项目使用中, 若有一个countDown()得不到执行, 那么awit()线程将永远阻塞下去, 这是一个比较严重的事情, ReentrantLock给我们提供了超时等待的机制:
```
CountDownLatch.await(100000, TimeUnit.MILLISECONDS)
```
指的是, 超时等待100s, 自动退出, `并不会因为超时没有获取到锁而抛出异常`。这里doAcquireSharedNanos在睡眠前, 将剩余超时时间与spinForTimeoutThreshold(默认1ms)做对比, 若小于1ms, 说明超时时间太短, 就没有必要再去睡眠, 而采取自旋的方式。
doAcquireSharedNanos与非超时的函数doAcquireShared区别主要就是底层一个调用了LockSupport.parkNanos(this, nanosTimeout), 一个调用了LockSupport.parkNanos(this), 别的并没有区别。
## 总结
CountDownLatch获取锁时候, 调用await()时, 只要state为0即可。 而state降低通过countDown()实现。该锁属于共享锁, 当state为0后, 会逐渐通知等待队列中的线程。该类大部分操作与ReentrantLock都是相似的。