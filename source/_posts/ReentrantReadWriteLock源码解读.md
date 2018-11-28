---
title: ReentrantReadWriteLock源码解读
date: 2017-07-28 06:18:19
tags:
---
首先回顾下ReentrantLock、CountDownLatch的区别: ReentrantLock是互斥锁, CountDownLatch是共享锁, 有没有哪种锁能够部分场景互斥, 部分场景共享呢, 那就是本文的主角:ReentrantReadWriteLock, 也是以AQS为基础实现的第三种应用。 要注意, ReentrantReadWriteLock与ReentrantLock没有一点关系。基本使用如下:
```
ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
//获取读锁
readWriteLock.readLock().lock();
//释放读锁
readWriteLock.readLock().unlock();
//获取写锁
readWriteLock.writeLock().lock();
//获取写锁
readWriteLock.writeLock().lock();
```
根据锁的名称, 基本也能猜出大致互斥关系, 读锁与读锁共享, 读锁与写锁互斥, 写锁与写锁互斥。读锁和写锁分为公平锁和非公平锁, 默认为非公平锁, 读锁和写锁这么初始化:
```
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

```
这里首先讲下state的含义: 读锁和写锁共享同一个state, 其为int, 高16位记录读共享的次数, 低16位记录写互斥的次数, 可能有人会问, 写不是互斥吗? 这里写锁也是可重入的。所以防止越界, 读写都不能超过2^16(65536)次。
# 读锁
## 获取读锁
通过lock()获取读锁, 首先进入AQS.acquireShared(1), 如下:
```
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }
```
首先尝试获取锁, 若获取到了, 则开始共享锁, 否则加入阻塞队列。
### 尝试获取锁:
```
        protected final int tryAcquireShared(int unused) {
            /*
             * Walkthrough:
             * 1. If write lock held by another thread, fail.  先有别人写锁，直接排队
             * 2. Otherwise, this thread is eligible for       自己写锁，自己读锁，检查排队的第一个不是写锁，获取到
             *    lock wrt state, so ask if it should block    自己写锁，自己读锁，第一个排队的是写锁
             *    because of queue policy. If not, try
             *    to grant by CASing state and updating count.
             *    Note that step does not check for reentrant
             *    acquires, which is postponed to full version
             *    to avoid having to check hold count in
             *    the more typical non-reentrant case.
             * 3. If step 2 fails either because thread
             *    apparently not eligible or CAS fails or count
             *    saturated, chain to version with full retry loop.
             */
            Thread current = Thread.currentThread();
            int c = getState();
            //同一个线程先写锁再读锁是可以获取锁的
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                 //当前获取写锁的线程不是本身
                return -1;
            int r = sharedCount(c);
            //阻塞队列队列第一个不是写锁，
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                //会左移动16位
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {//若是第一个读锁
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {//就是本身线程
                    firstReaderHoldCount++;
                } else { //不是第一个获取读锁的线程
                    HoldCounter rh = cachedHoldCounter;
                    //上一个节点不存在，或者存在了不是自己的
                    if (rh == null || rh.tid != getThreadId(current))
                        //那就生成自己的，并且缓存起来
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0) //若上一个获取的节点就是自己，
                        readHolds.set(rh);
                    rh.count++; //上一次获取锁+1
                }
                return 1;
            }
            //不该获取到锁（有第一个写锁在阻塞、读达到最大值，status设置不成功）
            return fullTryAcquireShared(current);
        }
```
做了做了如下事情:
+ 首先检查是不是已经有写锁获取到锁, 同时这个获取写锁的不是自己, 那么获取锁失败
+ 做些检查工作, 若都满足, 那么该线程就获还是可以取到了读锁的。
  1.检查等待队列中, head节点不是写线程阻塞
  2.再检查读锁没有达到65536的上限
  3.同时尝试设置读锁+1, 因为读锁为高16位, compareAndSetState(c, c + SHARED_UNIT))的目的通过偏移来完成的。如果都符合条件且操作成功,  同时还需要做如下工作:
  3.1 若本线程是第一个获取到读锁的, 那么firstReader记录下该线程, firstReaderHoldCount记录了该线程获取读锁的可重入次数, 记录这些变量,是为了某种情况下读线程的可重入操作, 后面会介绍。
  3.2 若本线程是第一个获取读锁的那个线程, 重入次数+1
  3.3 若本节点不是第一个获取读锁的线程,  那么根据LocalThread记录本线程可重入的次数。 cachedHoldCounter缓存的是上次获取读锁线程的信息, 既然有了LocalThread:readHolds, 此变量不是显得多此一举? 存在的意义就是为了减少通过LocalThread.get()获取当前线程重入信息, 以减轻该操作对性能的影响。
如果上述检查和操作没有成功的话, 那么进入fullTryAcquireShared()进一步再次尝试获取锁。
```
        final int fullTryAcquireShared(Thread current) {//（有写锁在等待、读达到最大值，status设置不成功）
            /*
             * This code is in part redundant with that in
             * tryAcquireShared but is simpler overall by not
             * complicating tryAcquireShared with interactions between
             * retries and lazily reading hold counts.
             */
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {//有写锁
                    //写锁是本身？
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                        //当前获取写锁的是本线程，那么直接返回（降级锁）
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) { //是否下一个要唤醒的是写锁
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {//当前读锁线程第一个获取了读锁，那么继续可以读
                        // assert firstReaderHoldCount > 0;
                    } else { //当前有第一个写阻塞，而第一个读锁又不是自己
                        //已经有写锁等待了，获取当前（这里说的意思呢，就是检查当前是第几次可重入，如果一次都没有可重入过，那就直接失败，若不是第一个可重入，那就获取到锁）
                        if (rh == null) {
                            //一般最后一次获取所得，就是当前线程
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();//当前线程信息
                                //当前线程非可重入，在阻塞之前，要清空记录
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        //该线程若是第一次可重入，那么就也去排队吧，如果不是第第一个次可重入，那就去排队吧
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT) //是否达到了最大值，这里是可以读取超过65536的
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) { //那么去设置
                    if (sharedCount(c) == 0) { //第一次获取读锁
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else { //不是自己首先申请的读锁
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get(); //获取到本线程的锁记录
                        else if (rh.count == 0)//为0的时候都已经从线程中删掉了
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release最后获取锁的线程
                    }
                    return 1;
                }
            }
        }
```
此函数的前提条件是:要么阻塞队列第一个线程为写线程, 要么原子更新读state失败, 次函数循环执行, 就是是保证原子操作失败后的重试。
+ 首先检查是否有写锁, 如果存在写锁, 再检查获取写锁的线程是否是当前线程, 若是的话, 那么会获取到锁, 这里实现了锁降级(由写锁降为读锁)的功能。
+ 反之, 检查是否有写线程在阻塞, 若是, 若这个阻塞的线程是本身, 那么不影响获取锁。 若不是, 这里就要详细分类了, 此时的场景是别的线程获取了读锁, 而有写线程被阻塞。
1. 若本线程是第一次获取读锁, 本次获取读锁不是可重入的, 那么为了防止获取写锁的线程饿死, 禁止新的线程获取读锁, 新的读锁线程将也处于阻塞队列。 同时将本线程从readHolds中删掉。
2. 若该线程之前获取了锁, 并且还没有释放, 那么此时获取锁是允许的, 那么同意继续获取读锁, 此时算是该线程读锁的可重入。
+ 检查读锁线程是否超过阈值65536
+ 设置读锁的state.
尝试获取流程如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/ReetrantReadWriteLock1.png" height="250" width="700"/>
### 加入阻塞队列
加入阻塞队列调用的是doAcquireShared, 大致实现可参考<a href="https://kkewwei.github.io/elasticsearch_learning/2017/08/24/CountDownLatch%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">CountDownLatch源码解读</a>doAcquireSharedInterruptibly(), 这里添加的节点的nextWaiter为SHARED, 表示该节点唤醒换后, 会继续向后继节点传播该信号
## unlock()
通过unlock()释放读锁, 首先进入sync.releaseShared(1)释放:
```
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) { //如果status彻底为0， 那么就说明可以唤醒await()了
            doReleaseShared();
            return true;
        }
        return false;
    }
```
主要做了两件事:
+ 首先尝试释放读锁, 并检查读锁线程state是否为0
+ 若读锁线程state为0, 那么唤醒阻塞队列线程。
尝试释放锁的过程如下:
```
        protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            if (firstReader == current) { //本节点是第一个读取数据的线程
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;  //最开始获取读锁的线程，去掉
                else
                    firstReaderHoldCount--;
            } else {  //不是第一个读取数据的线程
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) { //把本线程访问记录从Localhost中去掉
                    readHolds.remove();
                    if (count <= 0)
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                int c = getState();
                int nextc = c - SHARED_UNIT; //
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }
```
+ 检查本线程是否是第一个获取读锁的线程, 若是的话, 分别修改firstReaderHoldCount及firstReader对应的值。
+ 反之, 修改readHolds里面关于当前线程的获取锁情况, cachedHoldCounter是为了减少ThreadLocal.get()的访问次数。
+ 开始修改state读锁的标志, 这里使用for是为了保证失败后的尝试。
若此时读锁已经全部释放, 那么返回true, 表明可以唤醒阻塞队列的线程了。
唤醒阻塞队列的线程过程doReleaseShared, 具体过程请看<a href="https://kkewwei.github.io/elasticsearch_learning/2017/08/24/CountDownLatch%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">CountDownLatch源码解读</a>doReleaseShared(), 主要做的工作就是检查后续阻塞队列, 若是signal, 那么就唤醒阻塞线程。
可以看出, readLock的获取与释放主要过程与CountDownLatch操作及其相似的, 不同的是尝试获取锁的步骤不同。
# 写锁
## lock()
写锁获取主要通过 sync.acquire(1)尝试获取:
```
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
该函数主要分为三步:
+ 尝试去获取写锁, 若获取到了, 就直接退出。
+ 若没有获取到写锁, 那么将当前线程构成一个Node, 放入线程阻塞队列, 线程进入睡眠等待。
+ 若本次没有获取到锁、从阻塞队列中被唤醒, 并且acquireQueued()返回true, 那么说明该线程被别人调用了中断, 我们需要将该中断再置位向外传递。
来看第一步:
```
        protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            Thread current = Thread.currentThread();
            //当前锁个数
            int c = getState();
            //写锁个数
            int w = exclusiveCount(c);
             ////当前锁个数 != 0（是否已经有线程持有锁），线程重入
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                //w == 0,表示写线程数为0， 有读锁； 有写锁，但是不是当前线程，也退出
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                //当前写锁， 是本身线程，可重入，但是不能超过65536个
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                //写锁可重入
                setState(c + acquires);
                return true;
            }   //当前没有锁
            //是否该阻塞， 公平锁考考虑等待队列的线程。非公平锁就不用考虑等待队列的线程，直接false
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
            setExclusiveOwnerThread(current);
            return true;
        }
```
+ 首先检查是否锁不为0(读+写)。 若读+写不为0, 而写锁为0, 说明有读锁, 本线程获取锁失败; 或者写锁也不为0, 并且获取写锁的那个线程不是本线程, 说明不是写线程的重入,也获取锁失败。 若以上两步有成功的话, 则获取锁成功。
+ 反正则说明当前state=0(没有读+写线程), 那么成功获取到锁。 writerShouldBlock()对于写锁始终未false。
再来看第二步, 也就是说明本线程没有获取到锁, 那么将本线程加入阻塞队里等待唤醒, nextWaiter设置为EXCLUSIVE,  acquireQueued(addWaiter(Node.EXCLUSIVE), arg))具体怎么实现请去查看<a href="https://kkewwei.github.io/elasticsearch_learning/2017/08/28/ReentrantReadWriteLock%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">ReentrantReadWriteLock源码解读</a>acquireQueued()
第三步也很简单, 就是把中断信号向外传递。
## unlock()
写锁释放时,调用release()方法, 如下:
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
+ 释放锁时tryRelease会做最基本的检查, 比如记录的那个获取写锁的线程是否是本线程。
+ 若成功释放, 唤醒下一个阻塞的线程,  unparkSuccessor实现可见<a href="https://kkewwei.github.io/elasticsearch_learning/2017/07/23/ReentrantLock%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">ReentrantLock源码解读</a>。
也可以看出, writedLock的获取与释放主要过程与ReentrantLock操作及其相似的, 不同的是尝试获取锁的函数不同。

# 总结
ReentrantReadWriteLock读锁与写锁可以认为分别是ReentrantLock、CountDownLatch的实现, 不同的是对state赋予的含义不同。