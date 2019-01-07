---
title: DirectByteBuffer堆外内存详解
date: 2018-07-27 16:33:58
tags:
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
可以看到, DirectByteBuffer通过直接调用base=unsafe.allocateMemory(size)操作堆外内存, 返回的是该堆外内存的直接地址, 存放在address中, 以便通过address进行堆外数据的读取与写入。 unsafe的使用可以参考: 参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/11/10/LockSupport%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB/">LockSupport原理分析</a>
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

可能有些人会好奇: 为什么IO操作不直接使用堆内内存? 这是因为堆内内存会发生GC移动操作, 对象移动后, 其绝对内存地址也会发生改变, 而gc时对象移动操作很频繁, 不可能每次移动堆内数据, IO时缓存的buffer也跟着一起移动。这样也是不合理的。 而IO操作直接使用堆外内存则没有了这一限制。同时jvm中IO操作的Buffer必须是DirectBuffer(可查看IO.write/read函数)
# 总结
在JVM中, 一般只有通过DirectByteBuffer这一种方式操作堆外内存, 平时说的堆外内存泄漏, 也就是指的DirectByteBuffer里面的堆外内存发生泄漏。只有我们正确掌握了
