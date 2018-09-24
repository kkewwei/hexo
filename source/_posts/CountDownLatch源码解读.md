---
title: CountDownLatch源码解读
date: 2018-09-24 16:47:06
tags:
---
CountDownLatch也是线程同步的一个工具, 底层也是使用AQS(参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/09/23/ReentrantLock%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">ReentrantLock源码解读</a>)来进行锁的互斥。
CountDownLatch与ReentrantLock的主要区别是:
+ CountDownLatch是一个共享锁, ReentrantLock是一个独占锁。
+ CountDownLatch中state初始值为n, 代表等待n个条件满足, 好比有一个门需要n个钥匙聚齐后才能进行下面的流程, 可能有很多人需要通过这个门, 大家在门外围成一团, 若门打开后, 第一个通过的那个人可以告诉等待的人, 大家可以蜂拥一起过。而ReentrantLock中state为0, 表示锁没有被占用, 比如有一个很窄的门, 每次只能通过一个人, 锁那道锁(state置为1), 谁才能过那道门。 若有很多人等, 那么就要排队了, 若新来一个人来时, 门恰好是开着的, 他能忽略排队的人过去的话, 这就是非公平锁, 若需要进入等待队列的话, 那就是公平锁。排队的话, 是前面的人过完了,后面的人再继续过, 必须排队一个个过。
CountDownLatch使用方法如下:
```
CountDownLatch countDownLatch = new CountDownLatch(3);
countDownLatch.await();
countDownLatch.countDown();
```
await()方法表示检查state是否为0, 不为0则阻塞当前线程, countDown()表示当前state减一。
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
