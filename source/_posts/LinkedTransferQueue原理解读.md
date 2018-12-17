---
title: LinkedTransferQueue原理分析
date: 2018-12-15 18:59:58
tags:
---
LinkedTransferQueue作为无边界的阻塞队列, 同时继承了TransferQueue和AbstractQueue。 相较于其他的阻塞队列:  LinkedTransferQueue特殊之处在于TransferQueue。 TransferQueue说明如下
>* Transfers the element to a waiting consumer immediately, if possible.
     * More precisely, transfers the specified element immediately if there exists a consumer already waiting to receive it (in {@link #take} or timed {@link #poll(long,TimeUnit) poll}), otherwise returning {@code false} without enqueuing the element.

说的是LinkedTransferQueue不仅实现了普通BlockingQueue的功能, 另一个优点就是: 还可以当有消费者等待数据时, 直接将数据交给消费者而不是再进入队列。 拿LinkedBlockingQueue相比, LinkedBlockingQueue在take和put操作时, 都是通过lock来控制, 当take和put操作并发规模上来后, 锁的获取和释放都是比较占用资源的。 而LinkedTransferQueue对这种使用进行了改进, 当生产者存放数据时, 发现有消费者等待消费数据, 生产者可以调用transfer直接将数据交给消费者, 而不用通过阻塞队列来传递数据, 减少了锁的释放与获取。
# LinkedTransferQueue类简介
LinkedTransferQueue的阻塞队列的节点Node设计如下:
```
 static final class Node {
        // 若为true,
        final boolean isData;
         // initially non-null if isData; CASed to match
        volatile Object item;
        volatile Node next;
         // null until waiting  若不为null，则说明是reservation，有一个线程在等待
        volatile Thread waiter;
}
```


