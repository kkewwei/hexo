---
title: Netty对象回收池Recycler原理详解
date: 2019-01-16 09:09:39
tags: Cycler
toc: true
---
同Netty内存池(可参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/05/23/Netty%E5%86%85%E5%AD%98%E5%AD%A6%E4%B9%A0/">Netty PoolArea原理探究</a>)一样, 为了增强Netty高性能并发能力, 减少通用对象分配的损耗, 也采用了对象池的概念。 当需要某个对象时, 首先从对象池中获取该对象, 当使用完成后, 将对象释放到对象池中, 这样达到重复使用对象的效果。基本使用如下:
```
public class Cycli {
    private static final Recycler<Cycler> CyclerRecycler = new Recycler<Cycler>() {
        @Override
        protected Cycler newObject(Handle<Cycler> handle) {
            return new Cycler(handle);
        }
    };
    static final class Cycler {
        private String value;
        public void setValue(String value) {
            this.value = value;
        }
        private Recycler.Handle<Cycler> handle;
        public Cycler(Recycler.Handle<Cycler> handle) {
            this.handle = handle;
        }
        public void recycle() {
            handle.recycle(this);
        }
    }
    public static void  main(String[] args) {
        // 1、从回收池获取对象
        Cycler cycler1 = CyclerRecycler.get();
        // 2、开始使用对象
        cycler1.setValue("hello,java");
        // 3、回收对象到对象池
        cycler1.recycle();
        // 4、从回收池获取对象
        Cycler cycler2 = CyclerRecycler.get();
        //比较从对象池中获取的对象即为之前释放的对象
        System.out.print(cycler1 == cycler2);
    }
}
```
使用比较简单, 主要定义了如下几个对象:
+ 定义CyclerRecycler, 作为对象池的入口, 定义newObject()函数, 若对象池中没有可用对象, 则新建对象。
+ 定义被回收的对象Cycler, 包含成员变量handle, 该handler与该对象和stack绑定的。
+ 通过CyclerRecycler.get()从对象池中获取对象; 通过Cycler1.recycle()释放该对象到对象池。

##  WeakOrderQueue、Stack介绍
对象池通过Recycler里面WeakOrderQueue、Stack 2个类来实现。 首先放一张图来展示一个stack中两者的关系:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/Netty_Cycler1.png" height="450" width="650"/>
+ 每个线程都拥有自己的对象池, 该对象池结构如上图所示, stack作为本线程对象池的核心, 通过FastThreadLocal来实现每个线程的本地化。
+ 本线程回收本线程产生的对象时, 会将对象以DefaultHandle的形式存放在stack的elements数组中; 若本线程thread1回收其它线程thread2产生的对象时, 将该对象放到thread2对应stack的一个WeakOrderQueue的Link中。 也就是说一个WeakOrderQueue节点存放着一个其他线程帮着本线程回收本线程生产的对象。每个stack的WeakOrderQueue链表节点个数不能超过2*cpu, 可以通过io.netty.recycler.maxDelayedQueuesPerThread控制。 也就是说最多有2*cpu个线程帮着回收对象。
+ 每个link存放的对象是有限的, Link中DefaultHandle[]最多存放16个对象。 若thread1回收thread2产生的对象装满了一个Link, 则会再产生一个link继续存放。
+ 当前线程从对象池中拿对象时, 首先从stack的elements中获取, 若没有的话, 将尝试从当前WeakOrderQueue节点cursor的Link中的数组对象transfer到stack的elements, 再从stack的elements中获取对象。
+ stack的element数组最大长度32768, 可以通过io.netty.recycler.maxCapacityPerThread控制; 而Link节点中每个DefaultHandle数组默认长度16, 可以通过io.netty.recycler.linkCapacity控制;
通过elements及Link完成了整个对象池的构建。

# 从线程池获取对象
通过调用Recycler.get()来完成:
```
    public final T get() {
         // 若置为0, 将handle置为Noop_HANDLE, 代表着不被回收
        if (maxCapacityPerThread == 0) {
            return newObject((Handle<T>) NOOP_HANDLE);
        }
        // 获取当前线程对应的Stack
        Stack<T> stack = threadLocal.get();
        // 从对象池获取对象
        DefaultHandle<T> handle = stack.pop();
        // 若对象池中没有对象,则调用子类的newObject方法创建新的对象
        if (handle == null) {
            handle = stack.newHandle();
            handle.value = newObject(handle);
        }
        return (T) handle.value;
    }
```
主要做了如下事情:
+ 首先获取本线程对应的唯一stack, 从该stack中获取对象。
+ 若对象池中没有对象, 则主动调用newObject产生一个对象。同时完成了handle与对象、stack的绑定。
我们接下来看如何通过stack.pop()来从对象池中获取对象:
```
        DefaultHandle<T> pop() {
            //统计着elements中存放的对象个数
            int size = this.size;
           //若elements没有可用对象
            if (size == 0) {
                //就尝试从别的线程帮着回收的对象中转移一些到elements中, 也就是从WeakOrderQueue中转移一些数据出来
                if (!scavenge()) {
                    return null;
                }
                size = this.size;
            }
            size --;
            DefaultHandle ret = elements[size];
            elements[size] = null;
            //在stack的lastRecycledId及recycleId一定是相等的
            if (ret.lastRecycledId != ret.recycleId) {
                throw new IllegalStateException("recycled multiple times");
            }
            ret.recycleId = 0;
            ret.lastRecycledId = 0;
            this.size = size;
            return ret;
        }
```
对象在从对象池中被获取时, recycleId及lastRecycledId都被清零。
我们看scavenge是如何回收内存的。
```
        boolean scavenge() {
            //尝试从WeakOrderQueue中转移数据DefaultHandle到stack的elements中
            if (scavengeSome()) {
                return true;
            }

            // reset our scavenge cursor
            prev = null;
            cursor = head;
            return false;
        }
        boolean scavengeSome() {
             //cursor属性保存了上一次对WeakorderQueueu列表的浏览位置，每一次都从上一次的位置继续，这是一种FIFO的处理策略
            WeakOrderQueue prev;
            WeakOrderQueue cursor = this.cursor;
            //若游标为null, 则是第一次从WeakorderQueueu链中获取元素
            if (cursor == null) {
                prev = null;
                cursor = head;
                //若不存在任何WeakorderQueueu, 退出
                if (cursor == null) {
                    return false;
                }
            } else {
                prev = this.prev;
            }
            boolean success = false;
            //循环的不停地从WeakOrderQueue中找到一个可用的Link
            do {
                //从WeakOrderQueue中转移数据到element数组中。
                if (cursor.transfer(this)) {
                    success = true;
                    break;
                }
                WeakOrderQueue next = cursor.next;
                //如果当前处理的WeakOrderQueue所在的线程已经消亡，则尽可能的提取里面的数据，之后从列表中删除这个WeakOrderQueue。注意owner使用WeakReference<Thread>定义, 当线程消亡后, 通过cursor.owner.get()自然变为null
                if (cursor.owner.get() == null) {
                    // If the thread associated with the queue is gone, unlink it, after
                    // performing a volatile read to confirm there is no data left to collect.
                    // We never unlink the first queue, as we don't want to synchronize on updating the head.
                    //如果消亡的线程还有数据，
                    if (cursor.hasFinalData()) {
                        for (;;) {
                            //尽量将该线程对应的WeakOrderQueue里面link对应的对象迁移到elements中
                            if (cursor.transfer(this)) {
                                success = true;
                            } else {
                                break;
                            }
                        }
                    }
                   //将消亡的那个WeakOrderQueue从链中去掉
                    if (prev != null) {
                        prev.setNext(next);
                    }
                } else {
                    prev = cursor;
                }
                cursor = next;
            } while (cursor != null && !success);
            this.prev = prev;
            this.cursor = cursor;
            return success;
        }
```
若stack的elements中没有对象, 那么把对象从Link的DefautHandle[]中迁移到stack的elements中:
```
    boolean transfer(Stack<?> dst) {
            Link head = this.head;
            //WeakOrderQueue中整个Link链为空, 则直接退出
            if (head == null) {
                return false;
            }
            //说明head已经被读取完了，需要将head指向当前WeakOrderQueue的下一个link
            if (head.readIndex == LINK_CAPACITY) {
                if (head.next == null) {
                    return false;
                }
                //当前链节点换头
                this.head = head = head.next;
            }
            //获取当前可读的下标
            final int srcStart = head.readIndex;
            //当前link write的下标
            int srcEnd = head.get();
            //总共可读长度
            final int srcSize = srcEnd - srcStart;
            if (srcSize == 0) {
                return false;
            }
            //计算即将写到elements中起始与终点位置
            final int dstSize = dst.size;
            final int expectedCapacity = dstSize + srcSize;
            //如果超过stack当前能装下的最大elements个数
            if (expectedCapacity > dst.elements.length) {
                //将stack的elements扩容
                final int actualCapacity = dst.increaseCapacity(expectedCapacity);
                srcEnd = min(srcStart + actualCapacity - dstSize, srcEnd);
            }
            if (srcStart != srcEnd) {
                final DefaultHandle[] srcElems = head.elements;
                final DefaultHandle[] dstElems = dst.elements;
                int newDstSize = dstSize;
                //每个元素都开始从源迁移到目的地
                for (int i = srcStart; i < srcEnd; i++) {
                    DefaultHandle element = srcElems[i];
                    //对象在被回收时, recycleId、lastRecycledId都是0, 若直接被会受到stack的element中时, recycleId=lastRecycledId=thread_id; 若被会受到Link中时, lastRecycledId被修改成当前thread_id, recycleId仍为0, 当元素从Link迁移至stack的elements时, recycleId=astRecycledId。
                    if (element.recycleId == 0) {
                        element.recycleId = element.lastRecycledId;
                    } else if (element.recycleId != element.lastRecycledId) {
                        throw new IllegalStateException("recycled already");
                    }
                    srcElems[i] = null;
                    //为了防止stack的elements扩张太快, 实际每8个迁移的对象中只取1个, 7个都被丢弃了
                    if (dst.dropHandle(element)) {
                        // Drop the object.
                        continue;
                    }
                    element.stack = dst;
                    dstElems[newDstSize ++] = element;
                }
                // 若当前WeakOrderQueue的head已经被迁移完了, 需要从队列中抛弃
                if (srcEnd == LINK_CAPACITY && head.next != null) {
                    // Add capacity back as the Link is GCed.
                    //增加每个线程帮另一个线程最多回收的限制
                    reclaimSpace(LINK_CAPACITY);
                    this.head = head.next; //当前WeakOrderQueue更新head
                }
                //更新该head对象可读下标
                head.readIndex = srcEnd;
                if (dst.size == newDstSize) {
                    return false;
                }
                //更新stack可用对象的个数
                dst.size = newDstSize;
                return true;
            } else {
                // The destination stack is full already.
                return false;
            }
        }
```
从对象池中获取对象步骤总结如下:
1. 检查stack的elements中是否有可剩余的DefaultHandle。
2. 若没有的话, 从cursor的head开始查找当前WeakorderQueue, 并检查WeakorderQueue对应的线程是否还存活着, 若对应的帮着回收的线程不再了, 则调用transfer将该WeakorderQueue对应的所有link中的数组循环迁移到elements中, 迁移的时候每8个丢弃7个, 只有一个被回收。
3. 若对应线程还存活着, 则调用transfer进行回收当前WeakorderQueue中的一个link的所有DefaultHandle[]到stack的elements中。

# 向对象池中存放对象
如上例所示, 释放对象时调用cycler1.recycle()即可, 最终会调用与当前对象绑定的stack.push():
```
        void push(DefaultHandle<?> item) {
            Thread currentThread = Thread.currentThread();
            //如果本线程就是产生对象的那个县城，那么直接把该对象放到stack的elements数组里
            if (thread == currentThread) {
                // The current Thread is the thread that belongs to the Stack, we can try to push the object now.
                pushNow(item);
            } else {
                // The current Thread is not the one that belongs to the Stack, we need to signal that the push
                // happens later.
                //如果该stack不是本线程的stack，那么把该DefaultHandle放到该stack的WeakOrderQueue中
                pushLater(item, currentThread);
            }
        }
```
若回收对象的线程就是产生对象的线程, 那么直接将对象放到本stack对应的elements中。
```
        private void pushNow(DefaultHandle<?> item) {
            if ((item.recycleId | item.lastRecycledId) != 0) {
                throw new IllegalStateException("recycled already");
            }
            #俩都直接赋值相等, 则说明对象处于stack的elements中等待被读取。
            item.recycleId = item.lastRecycledId = OWN_THREAD_ID;
            int size = this.size;
            //在push到对象池时, 也会丢弃7/8的元素
            if (size >= maxCapacity || dropHandle(item)) {
                // Hit the maximum capacity or should drop - drop the possibly youngest object.
                return;
            }
            //直接把DefaultHandle放到stack的数组里，如果数组满了那么扩展该数组为当前2倍大小
            if (size == elements.length) {
                elements = Arrays.copyOf(elements, min(size << 1, maxCapacity));
            }
            elements[size] = item;
            this.size = size + 1;
        }
```
直接存放对象时, 对象池也会丢弃7/8的对象。
若回收对象的线程不是产生对象的线程, 我们来看下是如何将对象放到Link的数组中的:
```
   private void pushLater(DefaultHandle<?> item, Thread thread) {
            // we don't want to have a ref to the queue as the value in our weak map
            // so we null it out; to ensure there are no races with restoring it later
            // we impose a memory ordering here (no-op on x86)
            //DELAYED_RECYCLED里存放了当前线程向所有stack中插入的WeakOrderQueue的映射关系
            Map<Stack<?>, WeakOrderQueue> delayedRecycled = DELAYED_RECYCLED.get();
            //获取到当前线程向stack插入的WeakOrderQueue节点
            WeakOrderQueue queue = delayedRecycled.get(this);
            if (queue == null) {
                //每个stack/线程最多能向maxDelayedQueues（2*cpu）个线程的WeakOrderQueue队列添加回收的的对象
                if (delayedRecycled.size() >= maxDelayedQueues) {//如果已经向maxDelayedQueues个线程插入过数据, 那么将1个伪造的WeakOrderQueue（DUMMY）放到delayedRecycled中，并丢弃该对象（DefaultHandle）
                    // Add a dummy queue so we know we should drop the object
                    delayedRecycled.put(this, WeakOrderQueue.DUMMY);
                    return;
                }
                // Check if we already reached the maximum number of delayed queues and if we can allocate at all.
               //别的线程最多向这个stack的WeakOrderQueue插入16384个对象, 检查是否可以插入, 若可以插入, 就向这个stack头插法新建WeakOrderQueue对象
                if ((queue = WeakOrderQueue.allocate(this, thread)) == null) {
                    // drop object
                    return;
                }
                delayedRecycled.put(this, queue);
             //已经插入满了
            } else if (queue == WeakOrderQueue.DUMMY) {
                // drop object
                return;
            }
            queue.add(item); //向WeakOrderQueue对应的Link插入对象
        }
```
DELAYED_RECYCLED实际保存的是每个线程向别的stack插入WeakOrderQueue的对应关系, 下图是一个线程保存向别的stack插入WeakOrderQueue的映射关系。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/Netty_Cycler2.png" height="350" width="450"/>
找到对应的WeakOrderQueue后, 调用add向对应的Link中插入对象:
```
        void add(DefaultHandle<?> handle) {
            //这里仅仅修改lastRecycledId值, recycledId的修改时从WeakOrderQueue的link迁移到stack的elements的时候
            handle.lastRecycledId = id;
            Link tail = this.tail;
            int writeIndex;
            //若当前Link已经写满了, 那么我们再新一个Link存放对象
            if ((writeIndex = tail.get()) == LINK_CAPACITY) {
                if (!reserveSpace(availableSharedCapacity, LINK_CAPACITY)) {
                    // Drop it.
                    return;
                }
                // We allocate a Link so reserve the space
                this.tail = tail = tail.next = new Link();
                writeIndex = tail.get();
            }
            tail.elements[writeIndex] = handle;
            //本Link所处的stack即为handle.stack。在对象池中可以清空, 在被转移到stack的elements时重新赋值。
            handle.stack = null;
            // we lazy set to ensure that setting stack to null appears before we unnull it in the owning thread;
            // this also means we guarantee visibility of an element in the queue if we see the index updated
           //修改内存偏移地址为8的值，但是修改后不保证立马能被其他的线程看到。
            tail.lazySet(writeIndex + 1);  //https://github.com/netty/netty/issues/8215
        }
```
可以看到, 向Link中插入对象时, 仅改变对象的lastRecycledId值, 而没有改变recycledId值。

# 总结
Netty回收对象也不是把所有对象全部回收, 为了防止回收对象过多, 会在直接存入stack的elements和从Link转移到stack的elements时会丢弃7/8的废弃对象。Netty中使用对象回收的地方很多, 一个高频使用就是PooledUnsafeDirectByteBuf, 首先申请16M内存作为内存池时, 按需分配小的内存块, 这些小内存块都会被PooledUnsafeDirectByteBuf管理着。 而减少PooledUnsafeDirectByteBuf对象创建次数, 也增强了netty高性能传输数据的能力。
