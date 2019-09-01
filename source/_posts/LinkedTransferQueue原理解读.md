---
title: LinkedTransferQueue原理分析
date: 2018-12-15 18:59:58
tags:
toc: true
categories: Java学习
---
LinkedTransferQueue作为无边界的阻塞队列, 同时继承了TransferQueue和AbstractQueue。 相较于其他的阻塞队列:  LinkedTransferQueue特殊之处在于TransferQueue接口。 TransferQueue说明如下:
>* Transfers the element to a waiting consumer immediately, if possible. More precisely, transfers the specified element immediately if there exists a consumer already waiting to receive it (in {@link #take} or timed {@link #poll(long,TimeUnit) poll}), otherwise returning {@code false} without enqueuing the element.

说的是LinkedTransferQueue不仅实现了普通BlockingQueue的功能, 另一个优点就是: 当有消费者等待数据时, 生产者可以直接将数据交给消费者而不是再进入队列。 与LinkedBlockingQueue相比, LinkedBlockingQueue在take和put操作时, 都是通过lock来控制, 当高并发操作take和put操作, 锁的获取和释放都是比较影响性能的。 而LinkedTransferQueue对这种使用进行了改进, 当生产者存放数据时, 发现有消费者等待消费数据, 生产者可以调用transfer直接将数据交给消费者, 而不用通过阻塞队列来传递数据, 减少了锁的释放与获取。
# LinkedTransferQueue类简介
LinkedTransferQueue的阻塞队列的节点Node设计如下:
```
 static final class Node {
        // 若为true, 则说明节点是个数据节点, 否则, 则说明是个请求节点
        final boolean isData;
         // initially non-null if isData; CASed to match
        volatile Object item;
        volatile Node next;
         // null until waiting  若不为null，则说明是reservation，有一个线程在等待
        volatile Thread waiter;
}
```
LinkedTransferQueue中的阻塞队列节点分为两种, 数据节点和请求节点。当生产者调用put时, 若没有消费者需要数据, 直接将数据存放入阻塞队列中, 那么将生成数据节点(isData=true、item为用户存放的数据, waiter为该被阻塞的线程)。当调用take节点时, 若没有数据可供取走, 那么将在阻塞队列中建立请求节点(isData=false、item为null, waiter为该被阻塞的线程)。若阻塞队列未匹配节点为请求节点, 新来的也为请求, 那么将通过尾加的方式存放; 若新来一个数据节点, 那么就可以和阻塞队列中的节点匹配, 数据就不会放到阻塞队列中。注意: 阻塞队列中未匹配节点模式一致, 而不是说阻塞队列所有节点模式一致。 阻塞队列中被匹配的节点Node变化情况如下:

|阻塞队列节点被匹配前后|数据节点|阻塞节点|
|:------:|:------:|:------:|
|匹配前|isData=true, item=data|isData=false, item=null|
|匹配后|isData=true, item=null|isData=false, item=this|
由此, 无论是数据节点还是阻塞节点, 我们可以总结出如下结论:
+ 匹配前: ((item != null) == isData ) && (item != this)
+ 匹配后: ((item == null) == isData ) || (item == this)
该结论将在之后的代码中可见。我们也需要了解一个概念, 匹配:阻塞队列中数据节点或者请求节点, 若没有对应的线程来消费它, 那么这个节点就是未匹配的。反之就是匹配的。被匹配过的节点会延迟脱离队列, 为了减少head和tail的操作频率, 使用的松弛距离: head和tail不一定时刻指的队列的头和尾。
在LinkedTransferQueue从阻塞队列中存放数据的函数:
```
   public void put(E e) {
        xfer(e, true, ASYNC, 0);
   }
   public boolean offer(E e, long timeout, TimeUnit unit) {
        xfer(e, true, ASYNC, 0);
        return true;
   }
   public boolean add(E e) {
        xfer(e, true, ASYNC, 0);
        return true;
    }
    public boolean tryTransfer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {
        if (xfer(e, true, TIMED, unit.toNanos(timeout)) == null)
            return true;
        if (!Thread.interrupted())
            return false;
        throw new InterruptedException();
    }
```
从阻塞队列中取数据的函数:
```
    public E poll() {
        return xfer(null, false, NOW, 0);
    }
    public E take() throws InterruptedException {
        E e = xfer(null, false, SYNC, 0);
        if (e != null)
            return e;
        Thread.interrupted(); //清空中断， 抛出异常
        throw new InterruptedException();
    }
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        E e = xfer(null, false, TIMED, unit.toNanos(timeout));
        if (e != null || !Thread.interrupted())
            return e;
        throw new InterruptedException();
    }
```
有两个点需要我们注意下:
1. 这些函数读取/插入数据实际调用的都是xfer这一个函数, 是不是觉得很神奇, 我们将在之后详细介绍该函数的实现。
2. xfer中区别中包含NOW, SYNC, TIMED等参数的不同。
```
    /*
     * Possible values for "how" argument in xfer method.
     */
    // for untimed poll, tryTransfer  NOW就是取数据时, 发现没有数据, 线程立刻返回不追加元素到阻塞队列中等待。
    private static final int NOW   = 0;

    // for offer, put, add  ASYNC就是添加数据时, 若消费者来不及消费, 只是将数据放到阻塞队列中, 而线程正常返回。
    //这部分是LinkedTransferQueue作为阻塞队列正常用法
    private static final int ASYNC = 1;

    // for transfer, take  SYNC用于消费/生产者存放/拉去数据时, 若满足不了的话, 将请求存放在阻塞队列, 同时线程也将阻塞
     //这部分是LinkedTransferQueue的特性所在
    private static final int SYNC  = 2;

    // for timed poll, tryTransfer  TIMED用于超时时间内读取或者存放数据时, 若满足不了的话, 将请求存放在阻塞队列, 同时线程超时阻塞。
    private static final int TIMED = 3;
```

# xfer函数
xfer作为LinkedTransferQueue里面最核心的函数, 在读取数据都依赖它, 接下来将看具体的实现。
```
private E xfer(E e, boolean haveData, int how, long nanos) {
        //首先检查传递的数据是否符合要求
        if (haveData && (e == null))
            throw new NullPointerException();
        Node s = null;                        // the node to append, if needed
        //这里将不断循环重试+goto跳转, 直到明确return
        retry:
        for (;;) {// restart on append race
            // find & match first node, 从头开始查找
            for (Node h = head, p = h; p != null;) {
                boolean isData = p.isData;
                Object item = p.item;
                // unmatched: 若是生产者说明生产者在等待消费者拿数据; 若是消费者说明在等待生产者生产数据
                if (item != p && (item != null) == isData) {
                     //can't match 模式一致的话, 那么只能将
                    if (isData == haveData)
                        break;  //两个节点是相同类型，不用match了，去下一步
                    // match 数据类型不同，要么放数据， 要么取数据，不成功，则循环重试
                    1. 当生产者存放数据时, 发现队列中已经有消费者在等待了, 那么直接将数据交给消费者
                    2. 当消费者获取数据时, 发现队列中已经有生产者在等待被消费, 那么消费者直接拿走数据。
                    //cas成功意味着匹配完成。如果本次是请求，则item原本是数据，e就为null反之则e是数据，item原本为null
                    if (p.casItem(item, e)) {
                          //head指针可能在p的上游
                        //循环的目的是：发现当前被使用的节点不是head节点, 那么就会清理head节点, 确保松弛距离<2
                        for (Node q = p; q != h;) {
                            Node n = q.next;  // update by 2 unless singleton
                            //发现head还没有变化, 将head指向当前节点的下一个节点
                            if (head == h && casHead(h, n == null ? q : n)) {
                                //设置next等于自己，从队列中移除了，等待被GC
                                h.forgetNext();
                                break;
                            }
                             // advance and retry， 发现变化了, 则移动head ,尽量保持head与未匹配节点的距离<2
                            if ((h = head)   == null ||
                                  //cas 失败后则检查当前head指针距离下一个有效节点是否大于2.大于则再次循环，否则退出。头节点的松弛长度由这段代码决定。从代码上可以看出，松弛距离是2.
                                (q = h.next) == null || !q.isMatched())
                                break;        // 除非松弛度小于2(head节点向后到未匹配节点距离小于2才退出)
                        }
                        //唤醒那个节点, 若不是线程阻塞, 那么waiter将为null
                        LockSupport.unpark(p.waiter);
                        return LinkedTransferQueue.<E>cast(item);
                    }
                }
                Node n = p.next; //找下一个节点
                // Use head if p offlist  如果p已经脱离队列，则从head继续找
                p = (p != n) ? n : (h = head);
            }
            //操作不是立刻返回的
            if (how != NOW) {                 // No matches available
                if (s == null)  //第一个进来的
                    s = new Node(e, haveData);
                 //ASYNC, SYNC, TIMED类型接口都会将请求追加到队尾，返回队列的上一个节点
                Node pred = tryAppend(s, haveData);
                if (pred == null)
                    continue retry;           // lost race vs opposite mode
                if (how != ASYNC)
                    //只有SYNC或者TINED情况, 线程才会被阻塞。只有部分接口才会阻塞线程
                    return awaitMatch(s, pred, e, (how == TIMED), nanos);
            }
            return e; // not waiting   立刻返回的，
        }
    }
```
xfer函数做了如下事情:
+ 检查传递进来的isData与e是否匹配的。
+ 从阻塞队列头开始进行查找是否有节点未匹配&&与本次请求匹配, 若与本次请求匹配, 本次请求直接返回, 唤醒睡眠的那个线程(有线程睡眠), 同时清理队列, 保持head与未匹配节点的距离<2(松弛距离)。
+ 若发现队列第一个未匹配节点与本次请求模式一致的, 本请求将会加入阻塞队列中(tryAppend)。
+ 若请求类型是异步或者超时等待的, 那么将同时阻塞线程(awaitMatch)。
阻塞队列为了减轻并发带来的效率问题, 引入了松弛距离的概念: 向阻塞队列中添加或者读取元素时, 并不会立刻去修改head或者tail指针, 而是只需保证:head向后与未匹配的节点距离不能超过2; tail向后与真正链表末尾节点距离不能超过2, 如下图所示:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/LinkedTransferQueue1.png" height="250" width="500"/>
head的松散距离<2会在节点与线程匹配之后、当发现匹配节点不为head时就会进行检查, 而tail的松散距离检查会在tryAppend中存放数据后进行。
```
    //tryAppend就是尽全力将节点追加到队列最后
    private Node tryAppend(Node s, boolean haveData) {
        //从tail开始循环, tail不一定是队列真正最后的节点
        for (Node t = tail, p = t;;) {        // move p to last node and append
            Node n, u;                        // temps for reads of next & tail
            //当队列没有任何节点时
            if (p == null && (p = head) == null) {
                if (casHead(null, s))
                    return s;                 // initialize
            }
            //检查队列当前节点是否可成为s的前继节点: 模式一致&&当前节点未被匹配, 模式不同的两个节点同时想成为head就会出现这种情况
            else if (p.cannotPrecede(haveData))
                return null; // lost race vs opposite mode 否则返回从头开始检查
            //若发现tail不是最后一个的话, 还可以向后移动
            else if ((n = p.next) != null)    // not last; keep traversing
                //tail节点发生变化的话, p直接指向tail; 或者p后移一个继续检查。
                p = p != t && t != (u = tail) ? (t = u) : // stale tail
                    (p != n) ? n : null;      // restart if off list
            //如果CAS失败，那么说明别的节点修改了next指针, 这往后移动一个重来一次。
            else if (!p.casNext(null, s))
            //此时说明p一定找到了最后一个节点,
                p = p.next;                   // re-read on CAS failure
            else {
                 //开始检查p与tail的松弛距离, 如果tail距离最终节点距离>2, 则tail继续向后移动。
                if (p != t) {                 // update if slack now >= 2，
                    while ((tail != t || !casTail(t, s)) &&    //t为tail时候
                           (t = tail)   != null &&    //tail不为null
                           (s = t.next) != null && // advance and retry      //tail后面一个也不为null
                           (s = s.next) != null && s != t);    //tail的后面的后面也不为null。即有节点的话，就一直往后面找
                }
                return p; //返回添加节点的上一个节点
            }
        }
    }
```

若当前请求得不到满足的话, 那么当前线程将以节点形式加入队列而被阻塞:
```
//为了减少睡眠后的线程切换消耗, 这里线程不会立马阻塞, 而是会循环一段时间, 发现没有被别的模式相反的节点屁屁额, 没有希望了才会去睡眠
private E awaitMatch(Node s, Node pred, E e, boolean timed, long nanos) {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        Thread w = Thread.currentThread();
        int spins = -1; // initialized after first item and cancel checks
        //randomYields为了满足并发场景下的随机数获取, 为了解决在同一时刻多个线程获取随机数时的不一致问题。
        ThreadLocalRandom randomYields = null; // bound if needed
        for (;;) {
            Object item = s.item;
            if (item != e) {
                //matched  说明该节点被别的线程唤醒了，被匹配后, 节点item要么被置位null, 要么被放元素。
                // assert item != s;
                //item置为为本身, 则节点脱离集群, 更容易被回收; waiter置为null
                s.forgetContents();           // avoid garbage
                return LinkedTransferQueue.<E>cast(item);
            }
            //若有中断信号, 或者超超时小于0 , 则从队列中清除当前线程
            if ((w.isInterrupted() || (timed && nanos <= 0)) &&
                    s.casItem(e, s)) {        // cancel ，让item等于自己，这样类似匹配过了的效果
                unsplice(pred, s);
                return e;
            }
            //根据不同的模式考虑循环多少次, 若队列模式已经变了, 那么当前节点很快被匹配, 会旋转192次; 若pre被匹配了, 那么本节点也很快被匹配; 或者旋转128次
            if (spins < 0) {                  // establish spins at/near front
                if ((spins = spinsFor(pred, s.isData)) > 0)
                    randomYields = ThreadLocalRandom.current();
            }
            else if (spins > 0) {             // spin
                --spins;
                if (randomYields.nextInt(CHAINED_SPINS) == 0)   //偶发性释放cpu， 64次可能释放一次
                    Thread.yield();           // occasionally yield
            }
            //spins=0，已经循环了spins词，那么 设置当前线程为s节点上的阻塞线程， 然后下次循环就会调用park
            else if (s.waiter == null) {
                s.waiter = w;                 // request unpark then recheck
            }
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos > 0L)
                    LockSupport.parkNanos(this, nanos);
            }
            else {
                LockSupport.park(this);
            }
        }
    }
```
# 示例
由于过程比较繁琐, 为了更好理解, 接下来将顺序展示不同线程执行tansfer(a), transfer(b), tansfer(c), take(), take(), take(), transfer(d)/take()的操作。
1. 线程thread1、thread2、thread3分别进行tansfer(a)、tansfer(b)、tansfer(c), 过程比较简单, 没啥好说的。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/LinkedTransferQueue2.png" height="200" width="300"/>
<img src="https://kkewwei.github.io/elasticsearch_learning/img/LinkedTransferQueue3.png" height="200" width="350"/>
<img src="https://kkewwei.github.io/elasticsearch_learning/img/LinkedTransferQueue4.png" height="200" width="400"/>
tansfer()主要是将节点尾插法进入队列; 若tail离最后的节点相差大于2, 则移动tail满足需求。
2. 线程thread4执行take()时, 找到匹配的节点, 置位null。 并唤醒thread1; thread1还会将item=this, waiter=null
<img src="https://kkewwei.github.io/elasticsearch_learning/img/LinkedTransferQueue5.png" height="400" width="650"/>
线程thread5执行take()时, 找到匹配的节点, 置位null, 并唤醒thread2. 发现当前匹配节点不是head节点, 那么则移动head节点直到松弛距离为2。 thread2还会将item=this, waiter=null。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/LinkedTransferQueue6.png" height="400" width="700"/>
线程thread4执行take()时, 找到匹配的节点, 置位null。 并唤醒thread1; thread1还会将item=this, waiter=null
<img src="https://kkewwei.github.io/elasticsearch_learning/img/LinkedTransferQueue7.png" height="350" width="600"/>
3. 若此时线程thread7进行了tansfer(c), 仅仅将线程thread7以数据节点添加至队列中。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/LinkedTransferQueue8.png" height="250" width="800"/>
若此时线程thread7进行了take(), 此时也添加到了末尾, 注意此时队列模式发生了变化。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/LinkedTransferQueue9.png" height="250" width="750"/>
4. 可能有人会好奇, 队列真正的头结点无法遍历, 这样会不会存在内存泄露的问题? 我们大可不必担心, 仍然以例子来说明, 在thread7进行了take()的基础上, thread8线程进行了put(4)操作:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/LinkedTransferQueue10.png" height="400" width="900"/>
可以看到, 队列头两个无用的节点会自动脱离队列, 自然会被回收。

# 总结
LinkedTransferQueue与其他阻塞队列相比, 比较大的区别就是也可以阻塞put线程, 此时当有当take()操作时, take线程是不会进入队列的, 而是直接将put()线程唤醒。 结构上采取松弛距离<2来达到在高并发下减少互斥锁的操作而加快了效率。

# 参考
https://segmentfault.com/a/1190000016460411
https://www.zybuluo.com/eric1989/note/698826
http://ifeve.com/buglinkedtransferqueue-bug/