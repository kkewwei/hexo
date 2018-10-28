---
title: Semaphore源码解读
date: 2017-08-15 17:56:32
tags:
---
Semaphore信号量底层也是使用AQS(参考<a href="https://kkewwei.github.io/elasticsearch_learning/2017/07/23/ReentrantLock%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">ReentrantLock源码解读</a>)来进行锁的互斥。
基本使用如下:
```
Semaphore semp = new Semaphore(2, fasle);
//若被别的线程通过中断唤醒了, 那么就直接抛出异常
public void acquire();
//若被别的线程通过中断唤醒了, 那么将中断再放到本线程里退出
public void acquireUninterruptibly()
//尝试获取锁, 若获取不到就直接退出
public boolean tryAcquire()
//再unit内等待获取锁, 获取不到就直接抛出异常
public boolean tryAcquire(long timeout, TimeUnit unit) throws InterruptedException

//释放一个锁。
public void release()
```
Semaphore构造函数中, 2代表获取锁的并发, false代表非公平锁。信号量Semaphore的用户是限制访问的并发, 最大只能两个线程获取锁, 别的线程线程获取时都只能等着。statue最大为2, 代表可剩余的锁个数
|锁类型|介绍|
|:-|:-|
|ReentrantLock|互斥可重入锁, 获取锁并发为1, 谁获取锁谁可以执行, 否则阻塞|
|CountDownLatch|共享锁, 类似n个钥匙一起才能打开一个锁, 打开之后会唤醒所有阻塞的线程再一起执行|
|ReentrantReadWriteLock|有读锁和写锁同时构成, 读锁之间共享锁, 写锁——读锁会互斥|
|Semaphore|并发控制所, 允许同时只有n个线程可以访问, 别的线程只能阻塞, 仅当一个线程释放锁才能唤醒另外一个阻塞的线程|
# acquire获取过程
锁获取与其他几个AQS获取过程一样:
```
     public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
            //如果为0， 就说明获取到了，不为0， 则说明没有获取到
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```
+ 首先该线程是否有中断信号, 若有的话,直接退出
+ 尝试获取锁, 若获取到了, 那么就回退
+ 否则通过调用<a href="https://kkewwei.github.io/elasticsearch_learning/2017/08/24/CountDownLatch%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">CountDownLatch源码解读</a>doAcquireSharedInterruptibly()将自己加入等待队列。
唯一的区别就是Semaphore调用自己实现的tryAcquireShared来尝试获取锁。
```
        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors()) //首先检查是否有别的线程比当前登的时间更长
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
```
示例展示的是公平锁锁, 做了如下检查:
+ 检查是否已经有线程在排队, 若有的话, 那么获取锁失败(体现先来后到的原则)
+ 若没有线程在排队, 那么尝试获取acquires, 若state大于remaining个, 那么获取成果获取锁, 否则获取锁失败。
我们需要知道, Semaphore调用的doAcquireSharedInterruptibly来进入阻塞队列排队, 本身设置为SHRAD模式, 若别的线程将本线程唤醒后, 本线程也会把唤醒信号分享给后续阻塞线程, 然后大家一起去竞争锁。

# 释放锁release
释放锁会调用releaseShared:
```
    public final boolean releaseShared(int arg) {
         //如果status彻底为0， 那么就说明可以唤醒await()了
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
```
在tryReleaseShared将锁还给state(默认+1), 使用了for(;;)为了一定的释放成功才可以退出, 释放成功了, 尝试去调用doReleaseShared(具体参考<a href="https://kkewwei.github.io/elasticsearch_learning/2017/08/24/CountDownLatch%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">CountDownLatch源码解读</a>)来继续唤醒新的head节点。

# 总结
Semaphore信号量可以限制并发访问的次数, 使用起来也比较简单; 也分公平锁和费公平锁, 与ReentrantLock讲解的概念一样。
