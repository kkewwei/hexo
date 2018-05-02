---
title: BaseFuture及AbstractQueuedSynchronizer源码学习
date: 2017-12-17 03:54:43
tags: AbstractQueuedSynchronizer,源码,BaseFuture
---
## 简介
AbstractQueuedSynchronizer，顾名思义，抽象队列同步器，作为抽象类，使用FIFO链，实现了锁的语义`BaseFuture`。BaseFuture作为AbstractQueuedSynchronizer的实现类，赋予了锁的具体含义，定义了什么时候解锁及阻塞。
### AbstractQueuedSynchronizer详解
#### 属性                         
`state`
AbstractQueuedSynchronizer主要针对属性state来实现锁的含义，用户通过针对state赋予不同的值，实现不同锁的含义，在多线程针对state的操作，必须保证state状态的原子性，使用了`volatile`关键字，这里没使用Synchronized来保证原子性的原因:
+ state的状态修改不依赖历史的值，很适合volatile使用场景，设置了volatile后，也能保证state修改的可见性。
+ Synchronized实现互斥的成本要比volatile很高。
其他wiki中说的实现以下两种方法的原子性：
+ AbstractQueuedSynchronizer.getState()
+ AbstractQueuedSynchronizer.setState(int)
也都是通过volatile来实现的。
`Node`
AbstractQueuedSynchronizer实现了FIFO队列，该队列代表着目前阻塞的线程，每个元素都是都由一个Node构成，Node结构如下：
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
|waitStatus|当前节点的状态:<br>CANCELLED:当前线程取消执行，<br>SIGNAL:当前节点的后继节点的线程需要唤醒<br>CONDITION:当前节点在等待condition,也就处在condition队列中<br>PROPAGATE:<br>0:
|prev|当前节点的前一个节点
|next|当前节点的后继节点|
|thread|当前节点所拥有的线程|
|nextWaiter|链接指向下一个处于condition队列的节点
FIFO队列如下如：[FIFO队列](../img/BaseFuture_AbstractQueuedSynchronizer/1513529116336-image.png)