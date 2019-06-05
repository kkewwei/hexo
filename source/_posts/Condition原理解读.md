---
title: Condition接口原理介绍及ArrayBlockingQueue、LinkedBlockingQueue实现
date: 2017-10-02 19:28:01
tags:
toc: true
---
任何一个java对象都拥有wait()/nitify方法, 它们通过与synchronized结合(参考<a href="https://kkewwei.github.io/elasticsearch_learning/2016/10/27/Java%E7%BA%BF%E7%A8%8B%E7%9F%A5%E8%AF%86%E5%B0%8F%E7%BB%93/">Java 线程知识小结(-)</a>)来实现线程之间的通信机制。 在锁方面Lock与Condition接口配合也实现了相同的功能, 但是它们之间的原理是不相同的。使用上两者的区别如下(图片摘自<a href="https://www.jianshu.com/p/be2dc7c878dc">java并发编程之Condition</a>)):
<img src="https://kkewwei.github.io/elasticsearch_learning/img/Condition1.png" height="400" width="500"/>
# 基本使用
我们将以最简单的消费者与生产者的示例讲解Condition与Lock配合使用的例子:
```
public class ConditonTest {
    //当前产品个数
    public static int count = 0;
    static ReentrantLock reentrantLock = new ReentrantLock();
    //产品为空时, 那么消费者就得生产者来唤醒, 消费者就得等待生产者生产商品这个条件
    static  Condition empty = reentrantLock.newCondition();
    public static void main(String[] args) {
        ExecutorService pool = Executors.newFixedThreadPool(2);
        Thread t3 = new Consumer("A", ConditonTest.reentrantLock, ConditonTest.empty);
        Thread t1 = new Producder("B", ConditonTest.reentrantLock, ConditonTest.empty);
        Thread t2 = new Producder("C", ConditonTest.reentrantLock, ConditonTest.empty);
        Thread t4 = new Consumer("D", ConditonTest.reentrantLock, ConditonTest.empty);
        Thread t5 = new Consumer("E", ConditonTest.reentrantLock, ConditonTest.empty);
        // 执行各个线程
        pool.execute(t1);
        pool.execute(t2);
        pool.execute(t3);
        pool.execute(t4);
        pool.execute(t5);
        pool.shutdown();
    }
}
class Producder extends Thread {
    private String name;
    Lock lock;
    Condition empty;
    public Producder(String name, Lock lock, Condition empty) {
        this.name = name;
        this.lock = lock;
        this.empty = empty;
    }
    public void run() {
        lock.lock(); // 获取锁
        ConditonTest.count ++;
        System.out.println(name + "产生一个");
        empty.signalAll(); // 唤醒所有等待线程。
        lock.unlock(); // 释放锁
    }
}
class Consumer extends Thread {
    private String name;
    Lock lock;
    Condition empty;
    public Consumer(String name, Lock lock, Condition empty) {
        this.name = name;
        this.lock = lock;
        this.empty = empty;
    }
    public void run() {
        lock.lock(); // 获取锁
        try {
            while ( ConditonTest.count == 0) {
                System.out.println(name + "阻塞中");
                // 阻塞取款操作, await之后就隐示自动释放了lock，直到被唤醒自动获取
                empty.await();
                System.out.println(name + "被唤醒");
            }
            ConditonTest.count --;
            System.out.println(name + "消费一个");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock(); // 释放锁
        }
    }
}
```
以上只是使用一个Condition条件队列(具体真实应用可参考ArrayBlockingQueue、LinkedBlockingQueue), 在实际应用中, 可以使用多个条件ConditionObject, ConditionObject都是通过调用reentrantLock.newCondition()中产生的, 该类在AbstractQueuedSynchronizer中定义。 条件队列结构如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/Condition2.png" height="250" width="450"/>
每当线程调用Condition.wait()时, 该线程将会通过尾插发放入该条件队列; 当别的线程调用Condition.signal()时, 该线程将从等待队列转移到AQS阻塞队列(参考<a href="https://kkewwei.github.io/elasticsearch_learning/2017/07/23/ReentrantLock%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">ReentrantLock源码解读</a>), 使用比较简单。
注意阻塞队列和条件队列结构的区别:
+ 阻塞队列拥有head, head不存放任何线程; 由tail指定结尾; 条件队列首尾由firstWaiter,lastWaiter指定, 第一个线程即为firstWaiter。
+ 阻塞队列由next, pre连接; 条件队列由nextWaiter连接

# wait()
wait主要是将该线程存放在Condition队列等待被唤醒。empty.await()实际调用的AbstractQueuedSynchronizer中的wait函数:
```
   public final void await() throws InterruptedException {
        //检查当前中断标志位, 若之前有中断, 那么直接向外抛出异常
       if (Thread.interrupted())
           throw new InterruptedException();
       //将该线程添加到CONDITION队列中
       Node node = addConditionWaiter();
       //在该节点加入condition队列中等待前，await则需要释放掉当前线程占有的锁
       int savedState = fullyRelease(node);
       int interruptMode = 0;  //
       //检查该线程是否在阻塞队列中。线程首先加入条件队列，若有signal发生，会被signal转移到阻塞队列中
       while (!isOnSyncQueue(node)) {
          //已经加入条件队列, 开始进行睡眠
          LockSupport.park(this);
          //被唤醒了, 可能是别的线程调用了signal; 也可能是别的线程调用了中断。线程可能在条件队列, 也可能在阻塞队列
          if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
               break;
           //等于0说明是被signal唤醒的，将在下次while时直接退出循环
       }
       //此时在阻塞队列中了
       if (acquireQueued(node, savedState) && interruptMode != THROW_IE) //从阻塞队列中唤醒了
           interruptMode = REINTERRUPT;
       if (node.nextWaiter != null) // clean up if cancelled
           unlinkCancelledWaiters();
       if (interruptMode != 0)
           reportInterruptAfterWait(interruptMode); //
   }
```
主要做了如下事情:
+ 检查线程中断位, 若该线程存在中断信号则抛出异常
+ 将该线程加入条件队列, waitStatus置为Node.CONDITION, 见addConditionWaiter
+ 释放被占用的锁, 释放的savedState的值说明下次还需要获取的值。见fullyRelease
+ 检查线程是否阻塞队列, 见isOnSyncQueue。 若不在阻塞队列的话, 若在条件队列, 则开始调用LockSupport.park(this)睡眠。 若醒过来了存在两种可能: 别的线程调用了中断; 别的线程调用了signal, 需要继续判断是哪种情况, 见checkInterruptWhileWaiting。
+ 从while退出, 说明已经在阻塞队列了, 进入acquireQueued(参考<a href="https://kkewwei.github.io/elasticsearch_learning/2017/07/23/ReentrantLock%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">ReentrantLock源码解读</a>)等待被别的线程释放锁来唤醒
+ 清理条件队列中剩余线程, 若线程的node状态不为Node.CONDITION, 将从该队列中清理掉, 见unlinkCancelledWaiters函数。
+ 检查中断信息, 若存在中断, 中断是在调用signal前的话, 说明唤醒原因是因为中断, 那么线程直接向上抛出异常; 若中断是在调用signal之后的话, 那么唤醒原因是因为signal。将中断复位给线程。见reportInterruptAfterWait()
以下代码判断线程是否在阻塞队列中:isOnSyncQueue:
```
    final boolean isOnSyncQueue(Node node) {
         //只有在等待队列,waitStatus才会等于CONDITION; node.prev !=null,不代表一定是在阻塞队列中, 也可能在迁移的路上
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
         #若设置了, 肯定是在阻塞队列中。那就说明条件队列的next一定为null
        if (node.next != null) // If has successor, it must be on queue，
            return true;
        /*
         * node.prev can be non-null, but not yet on queue because
         * the CAS to place it on queue can fail. So we have to
         * traverse from tail to make sure it actually made it.  It
         * will always be near the tail in calls to this method, and
         * unless the CAS failed (which is unlikely), it will be
         * there, so we hardly ever traverse much.
         */
        return findNodeFromTail(node);  //同归对别在阻塞队列中彻头彻尾找这个节点
    }
```
而checkInterruptWhileWaiting主要是为了检查线程从LockSupport.park(this)只醒来的原因:
```
private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                  //至少线程是被中断过
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;  //返回0说明一定是被signasl唤醒的
        }
```
这里判断, 若不存在中断信号的话, 那么线程一定是被别的线程调用signal唤醒的, 那么进入transferAfterCancelledWait判断线程中断发生的是时间
```
 final boolean transferAfterCancelledWait(Node node) {
         //线程被signal唤醒，只会是Node.SIGNAL，而不是Node.CONDITION
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            enq(node);//说明是被interput的
            return true;
        }
        /*  此后说明是被signal唤醒的，
         * If we lost out to a signal(), then we can't proceed  //若果我们从
         * until it finishes its enq().  Cancelling during an
         * incomplete transfer is both rare and transient, so just
         * spin.
         */
         //等待调用singla的线程将本节点加入阻塞队列，该线程等待这个操作完成, 若还没有完成的话, 就等待
        while (!isOnSyncQueue(node))
            Thread.yield();  //偶尔释放下cpu，线程从运行到就绪状态
        //说明是被singla唤醒的，同时已经被其他的线程加入了阻塞队列
        return false;
    }
```
总结一下, 此时线程被唤醒了:
+ 不存在中断信号, 说明若线程被signal唤醒的, 那么返回0
+ 存在中断信号, 而线程的statue为Node.CONDITION, 那么说明线程是被中断唤醒的, 之后将会向外抛出InterruptedException异常。
+ 存在中断信号, 而线程的statue不为Node.CONDITION, 那么说明该中断信号是在线程被signal唤醒之后才出现的, 那么就不会抛向外抛出异常, 而是将中断信号再置位给线程。

# signal()
再来了解下signal是如何将唤醒线程的
```
private void doSignal(Node first) {

            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) && //这里while是
                     (first = firstWaiter) != null);
        }
```
通过不停地循环, 只是希望从条件队列中找到一个status=CONDITION的节点, 并将该节点移动到阻塞队列中。移动的过程如下:
```
  final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
         //如果不是CONDITION， 则返回再看条件队列中下一个元素
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
         //将该元素从条件队列移动到等待队列
        Node p = enq(node);
         //waitStatus>0的话，只能是cancel, 或者设置signal失败，那么唤醒该线程
        int ws = p.waitStatus;
         //若设置signal成功的话，wait函数那里的循环可能没有感觉到; 若没有感觉到会对调用yield减缓循环次数
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            //目前线程已经处于阻塞队列, 唤醒该线程, 尝试再获取锁
            LockSupport.unpark(node.thread);
        return true;
    }
```
我们现在回顾下, Condition的wait和signal整体逻辑如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/Condition3.png" height="270" width="600"/>
# 总结
Condition的wait和signal必须和锁配合使用, 以上只是结合ReentrantLock使用。ArrayBlockingQueue的实现原理与开头的示例类似, 基于ReentrantLock及Condition实现, Condition也可以与其他AQS结合, 比如CountDownLatch、ReentrantReadWriteLock, Semaphore, 使用大同小异, 这里就不一一讲解了。

