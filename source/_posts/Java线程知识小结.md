---
title: Java 线程知识小结(-)
date: 2016-10-27 14:18:30
tags:
toc: true
categories: Java学习
---
# wait/notify/notifyAll:
wait/notify用于线程通信的等待/通知模型, 这两个函数被定义在java.lang.Object类中, 被声明为final函数, 不可复写。使用时, 线程A调用object.wait(), 释放cpu进入等待; 线程B调用object.notify()来唤醒A线程。以下是基本的用法:
```
public static Object lock = new Object();

public void run(){//线程A
        try {
            synchronized (lock) {
                lock.wait(); //进入睡眠进行等待
                ...
            }

        } catch (Exception e) {
            ...
        }
}

public void run(){//线程B
        try {
            synchronized (lock) {
                lock.notify(); //唤醒A线程
                ...
            }

        } catch (Exception e) {
            ...
        }
}

```
线程执行状态如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/wait_notify.png" height="300" width="800"/>
需要注意一下几点:
+ wait/notify两个操作都必须要和synchronized(lock)配合使用, 这里可以这么理解: 每个object都拥有一个WatiQueue等待队列, 存放着调用lock.lock()被阻塞的线程, 每次向这个等待队列添加或者删除线程时, 为了保证对该队列操作的互斥性, 使用synchronized来达到目的;
+ 线程B调用lock.notify()后, 线程A并不能立刻从lock.wait()中醒来, 此时只是`线程B把线程A从lock对象的WatiQueue中移动到了SynchrozizedQueue中`, 线程A的状态由wait变化为blocked
+ 线程B完全执行完 synchronized(lock){}块后, 线程A才能继续执行。也就是说A从wait()返回的前提是获取到了锁。
+ 若线程A与B之间调用顺序不能反了, 若B先执行的话, 那么A将永远不能被唤醒（底层也是通过<a href="https://kkewwei.github.io/elasticsearch_learning/2018/11/10/LockSupport%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/#Parker-park">Parker::unpark</a>实现）。与直接使用LockSupport(park/unpark)相比很大的区别。
notify的作用只是唤醒一个object.wait()状态的线程, 唤醒哪个线程, 与线程优先级等有关 notifyAll的作用是唤醒全部object.wait()的线程去抢。

# Join
join函数的作用是等别的线程退出后再继续执行, 基本使用如下:
```
        Thread thread = new Thread(new a2());
        thread.start();
        thread.join();
```
当前线程调用thread.join()之后, 便会等待thread执行完再继续执行, join函数代码如下:
```
     public final synchronized void join(long millis) throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;
        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {//没有设置超时时间
            while (isAlive()) {
                wait(0);
            }
        } else {//超时等待的话
            while (isAlive()) { //循环检查
                long delay = millis - now;
                if (delay <= 0) { //超时时间到了之后退出
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
```
实际底层依靠的是wait(delay)函数 + 循环来达到等待的效果, 当wait等待时, cpu释放了资源, 则join()等待过程也是释放了cpu。