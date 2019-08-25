---
title: DirectByteBuffer堆外内存详解
date: 2018-07-27 16:33:58
tags:
toc: true
---
我们知道, 在使用IO传输数据时, 首先会将数据传输到堆外直接内存中, 然后才通过网络发送出去。这样的话, 数据多了次中间copy, 能否不经过copy而直接将数据发送出去呢, 其实是可以的, 存放的位置就是本文要讲的主角:DirectByteBuffer 。JVM内存主要分为heap内存和堆外内存(一般我们也会称呼为直接内存), heap内存我们不用care, jvm能自动帮我们管理, 而堆外内存回收不受JVM GC控制, 因此, 堆外内存使用必须小心。本文就主要讲jvm中堆外内存的实现及原理。
## DirectByteBuffer使用
在程序中, 我们可以通过如下方式获取到DirectByteBuffer, 并且直接作为IO的缓存:
```
 public void sendAndRecv(String words) throws IOException
    {
        byte[] msg = new String(words).getBytes();
        ByteBuffer buffer = ByteBuffer.allocateDirect(msg.length);
        buffer.put(msg);
        //读写模式相互转化写
        buffer.flip();
        System.out.println("Client sending: " + words);
        channel.write(buffer);
        channel.close();
    }
```
DirectByteBuffer是不能直接被外界引用的, 类成员变量如下:
```
    DirectByteBuffer(int cap) {
        super(-1, 0, cap, cap);
        //是否页对齐
        boolean pa = VM.isDirectMemoryPageAligned();
        //页的大小4K
        int ps = Bits.pageSize();
        //最小申请1K，若需要页对齐，那么多申请1页，以应对初始地址的页对齐问题
        long size = Math.max(1L, (long)cap + (pa ? ps : 0));
        //检查堆外内存是否够用, 并对分配的直接内存做一个记录
        Bits.reserveMemory(size, cap);
        long base = 0;
        try {
            //直接内存的初始地址, 返回初始地址
            base = unsafe.allocateMemory(size);
        } catch (OutOfMemoryError x) {
            Bits.unreserveMemory(size, cap);
            throw x;
        }
        //对直接内存初始化
        unsafe.setMemory(base, size, (byte) 0);
        //若需要页对其，并且不是页的整数倍，在需要将页对齐（默认是不需要进行页对齐的）
        if (pa && (base % ps != 0)) {
            // Round up to page boundary //初始地址取整页，注意申请的地址为取整数页
            address = base + ps - (base & (ps - 1));
        } else {
            address = base;
        }
        //声明一个Cleaner对象用于清理该DirectBuffer内存
        cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
        att = null;
    }
```
可以看到, DirectByteBuffer通过直接调用base=unsafe.allocateMemory(size)操作堆外内存, 返回的是该堆外内存的直接地址, 存放在address中, 以便通过address进行堆外数据的读取与写入。 unsafe的使用可以参考:<a href="https://kkewwei.github.io/elasticsearch_learning/2018/11/10/LockSupport%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">LockSupport原理分析</a>
我们需要了解下, Bits.reserveMemory()如何判断堆外内存是否可用的:
```
 static void reserveMemory(long size, int cap) {  ////对分配的直接内存做一个记录
        synchronized (Bits.class) {
            if (!memoryLimitSet && VM.isBooted())
            {
                //堆外直接内存默认等于堆内内存大小, 可以通过
                maxMemory = VM.maxDirectMemory();
                memoryLimitSet = true;
            }
            // -XX:MaxDirectMemorySize limits the total capacity rather than the
            // actual memory usage, which will differ when buffers are page
            // aligned.
            //如果够分的话，则直接退出
            if (cap <= maxMemory - totalCapacity) {
                reservedMemory += size;
                totalCapacity += cap; //
                count++;
                return;
            }
        }
        //不够分的话，则调用System.gc()进行一次full gc. 一般不要在线程启动时添加-XX:+DisableExplicitGC（禁止代码显示调用gc）
        System.gc(); //只是告知机器，这里应该GC一次， 但是实际并不一定进行垃圾回收
        try {
             //再等待100ms使gc有时间完成，然后再看是否够分配
            Thread.sleep(100);
        } catch (InterruptedException x) {
            // Restore interrupt status
            Thread.currentThread().interrupt();
        }
        synchronized (Bits.class) {
            //此时不够分的话，再调用向外抛出oom
            if (totalCapacity + cap > maxMemory)
                throw new OutOfMemoryError("Direct buffer memory");
            reservedMemory += size;
            totalCapacity += cap;
            count++;
        }
    }
```
可以看到:
+ 首先检查堆外内存是否够分
+ 若不够分的话, 再进行一次full gc显示推动对堆外内存的回收, 再次尝试分配堆外内存, 不够分的话, 则抛出OOM异常。

# 堆外内存的回收
在DirectByteBuffer的构造函数中, 我们可以看到这样的一行代码`cleaner = Cleaner.create(this, new Deallocator(base, size, cap));`, 没错, 直接内存释放主要由cleaner来完成。 我们知道JVM GC并不能直接释放直接内存, 但是GC可以释放管理直接内存的DirectByteBuffer对象。 我们需要注意下cleaner的类型:
```
public class Cleaner  extends PhantomReference<Object>
```
PhantomReference并不会对对象的垃圾回收产生任何影响, 当进行gc完成后, 当发现某个对象只剩下虚引用后, 会将该引用迁移至Reference类的pending队列进行回收. 这里可以看到DirectByteBuffer被Cleaner引用着。Reference操作回收代码如下:
```
    static private class Lock { };
    private static Lock lock = new Lock();


    /* List of References waiting to be enqueued.  The collector adds
     * References to this list, while the Reference-handler thread removes
     * them.  This list is protected by the above lock object. The
     * list uses the discovered field to link its elements.
     */
    //当gc时，发现DirectByteBuffer除了PhantomReference对象引用,没有其他对象引用， 会把DirectByteBuffer放入其中，等待被回收
    private static Reference<Object> pending = null;

    /* High-priority thread to enqueue pending References
     */
    private static class ReferenceHandler extends Thread {

        ReferenceHandler(ThreadGroup g, String name) {
            super(g, name);
        }

        public void run() {
            for (;;) {
                Reference<Object> r;
                synchronized (lock) {
                    if (pending != null) {
                        r = pending;
                        pending = r.discovered;
                        r.discovered = null;
                    } else {
                        // The waiting on the lock may cause an OOME because it may try to allocate
                        // exception objects, so also catch OOME here to avoid silent exit of the
                        // reference handler thread.
                        //
                        // Explicitly define the order of the two exceptions we catch here
                        // when waiting for the lock.
                        //
                        // We do not want to try to potentially load the InterruptedException class
                        // (which would be done if this was its first use, and InterruptedException
                        // were checked first) in this situation.
                        //
                        // This may lead to the VM not ever trying to load the InterruptedException
                        // class again.
                        try {
                            try {
                                //如果没有的话，会一直等待唤醒
                                lock.wait();
                            } catch (OutOfMemoryError x) { }
                        } catch (InterruptedException x) { }
                        continue;
                    }
                }

                // Fast path for cleaners
                if (r instanceof Cleaner) {
                     //从头开始进行clena()调用
                    ((Cleaner)r).clean();
                    continue;
                }

                ReferenceQueue<Object> q = r.queue;
                if (q != ReferenceQueue.NULL) q.enqueue(r);
            }
        }
    }

    static {
        ThreadGroup tg = Thread.currentThread().getThreadGroup();
        for (ThreadGroup tgn = tg;
             tgn != null;
             tg = tgn, tgn = tg.getParent());
        Thread handler = new ReferenceHandler(tg, "Reference Handler");
        /* If there were a special system-only priority greater than
         * MAX_PRIORITY, it would be used here
         */
        handler.setPriority(Thread.MAX_PRIORITY);
        handler.setDaemon(true);
        handler.start();
    }
```
可以看出来, JV会新建名为`Reference Handler`的线程, 时刻回收被挂到pending上面的虚拟引用。 当DirectByteBuff对象仅被Cleaner引用时, Cleaner被放入pending队列, 之后调用Cleaner.clean()队列
```
 public void clean() {  //这里的clean(）会在Reference回收时显示调用
        if (!remove(this))
            return;
        try {
            thunk.run();
        } catch (final Throwable x) {
            AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        if (System.err != null)
                            new Error("Cleaner terminated abnormally", x)
                                .printStackTrace();
                        System.exit(1);
                        return null;
                    }});
        }
}
//就是一个释放直接内存的线程
private static class Deallocator  implements Runnable
{

        private static Unsafe unsafe = Unsafe.getUnsafe();

        private long address;
        private long size;
        private int capacity;

        private Deallocator(long address, long size, int capacity) {
            assert (address != 0);
            this.address = address;
            this.size = size;
            this.capacity = capacity;
        }

        public void run() {
            if (address == 0) {
                // Paranoia
                return;
            }
            unsafe.freeMemory(address); //释放地址
            address = 0;
            Bits.unreserveMemory(size, capacity); //修改统计
        }

}

```
可以看到, 此时完成了DirectByteBuff直接内存的释放。
可能有些人会好奇: 为什么IO操作不直接使用堆内内存? 这是因为堆内内存会发生GC移动操作, 对象移动后, 其绝对内存地址也会发生改变, 而gc时对象移动操作很频繁, 不可能每次移动堆内数据, IO时缓存的buffer也跟着一起移动。这样也是不合理的。 而IO操作直接使用堆外内存则没有了这一限制。同时jvm中IO操作的Buffer必须是DirectBuffer(可查看IO.write/read函数)。
# 堆外内存的检测
1. 通过DirectByteBuffer申请的堆外内存, 我们可以通过如下的方式获取到:
```
    List<BufferPoolMXBean> bufferPools = ManagementFactory.getPlatformMXBeans(BufferPoolMXBean.class);
        for (BufferPoolMXBean bufferPool : bufferPools) {
            System.out.println("name: " + bufferPool.getName() + ", getCount:  " + bufferPool.getCount() +  ", getTotalCapacity:  " + bufferPool.getTotalCapacity() + ", getMemoryUsed:  " + bufferPool.getMemoryUsed());
    }
```
将会打印如下结果:
```
name: direct, getCount:  1, getTotalCapacity:  1073741824, getMemoryUsed:  1073741824
name: mapped, getCount:  0, getTotalCapacity:  0, getMemoryUsed:  0
```
direct部分内存就是了。
2. 若我们直接通过base = unsafe.allocateMemory(size)申请内存, 此块内存已不是JVM控制的了, 我们不能再通过上面那种方式检测出来了。对于这种内存申请, 需要别的方式来获取直接内存使用状态。

# 总结
在JVM中, 一般只有通过DirectByteBuffer这一种方式操作堆外内存, 平时说的堆外内存泄漏, 也就是指的DirectByteBuffer里面的堆外内存发生泄漏。合理使用DirectByteBuffer对通信框架有着很重要的帮助, 比如netty大量的IO数据传输, 都是通过DirectByteBuffer完成的。 直接内存的申请与释放比较代价比较大, 一般都会辅助对象池来尽量高效的利用申请的对象。
