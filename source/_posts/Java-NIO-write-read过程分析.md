---
title: Java NIO Write过程分析
date: 2017-04-10 23:35:58
tags: NIO, write, read
toc: true
categories: Java NIO及网络通信
---
在<a href="https://kkewwei.github.io/elasticsearch_learning/2017/04/10/JAVA-NIO%E5%8E%9F%E7%90%861/#JAVA-NIO%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8">JAVA NIO通信分析</a>中, 说明了客户端与服务器端是如何建立连接的, 那么本文将在上文的基础上, 了解服务器端是如何根据建立的连接进行写和读的。由于读写流程基本一致, 这里只分析如何进行写入的。
# write
服务器端调用了`SocketChannel.write(ByteBuffer.wrap( msg.getBytes()));`, 此时开启了写的流程:
```
  public int write(ByteBuffer buf) throws IOException { \
        if (buf == null)
            throw new NullPointerException();
        // 可以看到, 管道只要有一个在写, 那么久不让别的线程使用了
        synchronized (writeLock) {
            // SocketChannelImp state=ST_CONNECTED
            ensureWriteOpen();
            int n = 0;
            try {
                begin();
                synchronized (stateLock) {
                    if (!isOpen())
                        return 0;
                    writerThread = NativeThread.current();
                }
                for (;;) {
                    n = IOUtil.write(fd, buf, -1, nd); // 成功的话，就是写入成功的size
                    // 如果发现退出时因为中断原因, 并且管道还正常打开的话, 那么就继续写入。
                    if ((n == IOStatus.INTERRUPTED) && isOpen())
                        continue;
                    // 这里对IOS_UNAVAILABLE/EOF进行了识别, 若异常为IOS_UNAVAILABLE/EOF, 则没有问题, 但是还是直接退出了。
                    //返回的是成功写入的条数, -1 代表写入到文件末尾了, -2代表写入缓存满了/没有可以读取的了
                    return IOStatus.normalize(n);
                }
            } finally {
                writerCleanup();
                end(n > 0 || (n == IOStatus.UNAVAILABLE));
                synchronized (stateLock) {
                    if ((n <= 0) && (!isOutputOpen))
                        throw new AsynchronousCloseException();
                }
                assert IOStatus.check(n);
            }
        }
    }
```
这里主要做了如下工作:
1. 调用begin, 向当前线程的blocker属性赋值, 以便在写入时, 能影响中断信号。
2. 循环调用IOUtil.write向套接字写入数据, 返回的n表示当前写入的byte大小
3.当写入完成后, 调用writerCleanup清理现场。比如将writerThread置位, 代表没有线程写入了, 检查是否有必要进行将管道置为失败
<!-- more -->

## begin
我们看下写入时候是如何响应中断信号的, 进入bgin后
```
    protected final void begin() {
        if (interruptor == null) {
            interruptor = new Interruptible() {
                    public void interrupt(Thread target) {
                        synchronized (closeLock) {
                            if (!open)
                                return;
                            open = false;
                            interrupted = target;
                            try {
                                AbstractInterruptibleChannel.this.implCloseChannel();
                            } catch (IOException x) { }
                        }
                    }};
        }
        blockedOn(interruptor);
        Thread me = Thread.currentThread();
        if (me.isInterrupted())
            interruptor.interrupt(me);
    }
```
看起来和建立连接时的响应中断原理一样, 通过设置blocker属性实现, 我们看下AbstractInterruptibleChannel.this.implCloseChannel()又做了哪些工作:
```
    protected final void implCloseChannel() throws IOException {
        implCloseSelectableChannel();
        synchronized (keyLock) {
            int count = (keys == null) ? 0 : keys.length;  // 当前通道都关闭了，cancel只是将key放入cancelledKeys中，在下次正式选择开始的时候再一一清除；
            for (int i = 0; i < count; i++) {
                SelectionKey k = keys[i];
                if (k != null)
                    k.cancel();
            }
        }
    }
```
这里主要做了两件事情:
1. 调用implCloseSelectableChannel对套接字从网络层面真正关闭。
2. 调用k.cancel()对该管道上的所有事件从selector中取消, 包括取消保存在keys中映射关系、调用`register0(kq, channel.getFDVal(), 0, 0)`从kqueue中取消;。
我们重点关注下从网络层层面取消的过程:
```
  protected void implCloseSelectableChannel() throws IOException {
        synchronized (stateLock) {
            isInputOpen = false;
            isOutputOpen = false;
            // Close the underlying file descriptor and dup it to a known fd
            // that's already closed.  This prevents other operations on this
            // channel from using the old fd, which might be recycled in the
            // meantime and allocated to an entirely different channel.
            //
            if (state != ST_KILLED)
                nd.preClose(fd);

            // Signal native threads, if needed.  If a target thread is not
            // currently blocked in an I/O operation then no harm is done since
            // the signal handler doesn't actually do anything.
            //
            // 向写线程发送发唤醒信号。
            if (readerThread != 0)
                NativeThread.signal(readerThread);
            if (writerThread != 0)
                NativeThread.signal(writerThread);
            // If this channel is not registered then it's safe to close the fd
            // immediately since we know at this point that no thread is
            // blocked in an I/O operation upon the channel and, since the
            // channel is marked closed, no thread will start another such
            // operation.  If this channel is registered then we don't close
            // the fd since it might be in use by a selector.  In that case
            // closing this channel caused its keys to be cancelled, so the
            // last selector to deregister a key for this channel will invoke
            // kill() to close the fd.
            //
            // 如果这个管道上没有被注册关注的事件, 那么就可以安全的关闭, 若有的话, 可能有的线程正在进行读写, 此时关闭管道的任务交给最后一个事件来完成。
            if (!isRegistered())
                kill();
        }
    }
```
大致做了如下事情:
1. 调用SocketDispatcher.preClose(fd)半关闭。
2. 若此时有读写函数的话, 那么通过调用NativeThread.signal唤醒这些线程的
3. 若这个管道上没有被注册关注的事件, 那么就可以安全的关闭。
我们一一进行分析, 首先我们先看下SocketDispatcher类的实现吧:
```class SocketDispatcher extends NativeDispatcher {
    int read(FileDescriptor fd, long address, int len) throws IOException {
        return FileDispatcherImpl.read0(fd, address, len);
    }
     // 返回的是写入的成功数
    int write(FileDescriptor fd, long address, int len) throws IOException {
        return FileDispatcherImpl.write0(fd, address, len);
    }
    void close(FileDescriptor fd) throws IOException {
        FileDispatcherImpl.close0(fd);
    }
   // 将fd半关闭
    void preClose(FileDescriptor fd) throws IOException {
        FileDispatcherImpl.preClose0(fd);
    }
}
```
可以看到其实通过write()和read()底层都是调用FileDispatcherImpl本地函数, 看来这个FileDispatcherImpl本地实现不一般:
```
static int preCloseFD = -1;     /* File descriptor to which we dup other fd's
                                   before closing them for real */
JNIEXPORT void JNICALL
Java_sun_nio_ch_FileDispatcherImpl_init(JNIEnv *env, jclass cl)
{
    int sp[2];
    // SOCK_STREAM代表tcp协议
    if (socketpair(PF_UNIX, SOCK_STREAM, 0, sp) < 0) {
        JNU_ThrowIOExceptionWithLastError(env, "socketpair failed");
        return;
    }
    preCloseFD = sp[0];
    close(sp[1]);
}

JNIEXPORT jint JNICALL
Java_sun_nio_ch_FileDispatcherImpl_read0(JNIEnv *env, jclass clazz,
                             jobject fdo, jlong address, jint len)
{
    jint fd = fdval(env, fdo);
    void *buf = (void *)jlong_to_ptr(address);

    return convertReturnVal(env, read(fd, buf, len), JNI_TRUE);
}
...
...
JNIEXPORT void JNICALL
Java_sun_nio_ch_FileDispatcherImpl_preClose0(JNIEnv *env, jclass clazz, jobject fdo)
{
    jint fd = fdval(env, fdo); // 获取即将关闭的fd
    if (preCloseFD >= 0) {
        if (dup2(preCloseFD, fd) < 0)  // 用fd指定 产生新的文件表指定同一个节点表（类似inode概念），若fd已经存在，则关闭之。
            JNU_ThrowIOExceptionWithLastError(env, "dup2 failed");
    }
}
```
&#160; &#160; &#160; &#160;我们可以看到在init函数中使用socketpair产生了一对套接字, 它比较好玩, 我们知道, <a href="https://kkewwei.github.io/elasticsearch_learning/2017/04/10/JAVA-NIO%E5%8E%9F%E7%90%861/#%E4%BA%A7%E7%94%9FSelector">pipe()</a>也产生了一对套接字, 一边可以写, 另一边可以读,属于半双工的, 而通过socketpair产生的2个套接字都属于全双工, 也就是说任何一个套接字既可以读, 也可以写。那么每个SocketChannaleImp产生这样的套接字有什么用呢? 而且还在初始化时候的close一个套接字? 这就得从tcp断开连接说起了, 由于fd被作用于多线程环下, 我们不知道管道在什么时候被某个线程调用close了, 但是假如还有别的线程也正在进行read或者write(), 那么很可能就读取或者写入的是个不完整的数据。所以就引入了半关闭套接字。半关闭指的是关闭一对套双工套接字的其中一个, 然后我们对另外一个套接字进行读写的话, 那么会返回如下异常:`Program received signal SIGPIPE, Broken pipe`。
&#160; &#160; &#160; &#160;我们再继续看preClose0函数, 调用dup2函数。 要了解dup2函数的意思, 我们需要套接字在linux内核的数据结构, 每个进程会在运行期间打开一些文件, 文件描述符可以在/proc/pid/fd中可以看到映射关系。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/fd.png" height="300" width="350"/>
fd控制文件的写入实际是通过文件表项来实现的, 而int dup2(int oldfd, int newfd)的作用是产生一个文件描述符, 同时指向oldfd指向的文件表项, 若newfd已经打开, 则先将其关闭。 这样我们通过oldfd操作文件, 则也会反馈到newfd上, 类似软连的作用。了解了这些, 我们再来看这个函数就简单了。fd会首先关闭原来的链接, 同时指向一个已经关闭的连接上, 若别的线程在对这个套接字进行读写的时候, 会立马得到Broken pipe的异常, 而不是读取或写入一个错误的文件。
&#160; &#160; &#160; &#160;我们再来看第二点:当前线程是如何通过NativeThread.signal()唤醒读写线程的, 那我们不得不查看NativeThread的本地实现了:
```
#ifdef __linux__
/* Also defined in src/solaris/native/java/net/linux_close.c */
#define INTERRUPT_SIGNAL (__SIGRTMAX - 2)

static void
nullHandler(int sig){}

#endif

// 为什么要有这个init呢，其实不用这不操作也许不会有问题，但因为不能确保INTERRUPT_SIGNAL没有被其他线程install过，如果sa_handler对应函数不是空操作，则在使用这个信号时会对当前线程有影响
JNIEXPORT void JNICALL
Java_sun_nio_ch_NativeThread_init(JNIEnv *env, jclass cl)
{
#ifdef __linux__

    /* Install the null handler for INTERRUPT_SIGNAL.  This might overwrite the
     * handler previously installed by java/net/linux_close.c, but that's okay
     * since neither handler actually does anything.  We install our own
     * handler here simply out of paranoia; ultimately the two mechanisms
     * should somehow be unified, perhaps within the VM.
     */
    // 定义了信号量
    sigset_t ss;
    struct sigaction sa, osa;
    // 定义接收到信号后的操作。 这里什么也不做
    sa.sa_handler = nullHandler;
    sa.sa_flags = 0;
    // 初始化sigaction的sa_mask位
    sigemptyset(&sa.sa_mask);
    // 向当前线程安装信号量sa, 信号为INTERRUPT_SIGNAL
    if (sigaction(INTERRUPT_SIGNAL, &sa, &osa) < 0) //如果成功，则表示INTERRUPT_SIGNAL这个信号安装成功了
        JNU_ThrowIOExceptionWithLastError(env, "sigaction");
#endif
}
// 获取当前线程id
JNIEXPORT jlong JNICALL
Java_sun_nio_ch_NativeThread_current(JNIEnv *env, jclass cl)
{
#ifdef __linux__
    return (long)pthread_self();
#else
    return -1;
#endif
}
// 唤醒线程
JNIEXPORT void JNICALL
Java_sun_nio_ch_NativeThread_signal(JNIEnv *env, jclass cl, jlong thread)
{
#ifdef __linux__
    // kill线程的同时, 向线程发送INTERRUPT_SIGNAL信号
    if (pthread_kill((pthread_t)thread, INTERRUPT_SIGNAL))
        JNU_ThrowIOExceptionWithLastError(env, "Thread signal failed");
#endif
}

```
我们需要知道的是, 若我们对当前进程安装了信号量, 那么该信号量对当前进程的所有线程都是有效的, 可以通过pthread_kill显示kill线程, 发送INTERRUPT_SIGNAL信号量。
&#160; &#160; &#160; &#160;我们再来是怎么彻底关闭管道的:
```
   public void kill() throws IOException {
        synchronized (stateLock) {
            if (state == ST_KILLED)
                return;
            if (state == ST_UNINITIALIZED) {
                state = ST_KILLED;
                return;
            }
            assert !isOpen() && !isRegistered();
            // Postpone the kill if there is a waiting reader
            // or writer thread. See the comments in read() for
            // more detailed explanation.
            //若此时没有读写线程, 那就直接标记关闭
            if (readerThread == 0 && writerThread == 0) {
                nd.close(fd);
                state = ST_KILLED;
            } else {
            // 此时有读或者写
                state = ST_KILLPENDING;
            }
        }
    }
```
这里主要是将半打开的套接字给真正关闭了。

## write
我们再循环调用IOUtil.write继续写入:
```
    static int write(FileDescriptor fd, ByteBuffer src, long position, NativeDispatcher nd) throws IOException {
        if (src instanceof DirectBuffer)
            return writeFromNativeBuffer(fd, src, position, nd);
        // Substitute a native buffer
        int pos = src.position();
        int lim = src.limit();
        assert (pos <= lim);
        int rem = (pos <= lim ? lim - pos : 0);
        // 若不是DirectBuff，需要先转变为DirectByteBuffer。首先是从本地缓存中拿这个DirectByteBuffer
        ByteBuffer bb = Util.getTemporaryDirectBuffer(rem);
        try {
             // 跑到DirectByteBuffer.put里面，使用unsafe.copyMemory拷贝数据
            bb.put(src);
            bb.flip();
            // Do not update src until we see how many bytes were written
            src.position(pos);

            int n = writeFromNativeBuffer(fd, bb, position, nd);  // 从 DirectByteBuffer中向管道中写数据
            if (n > 0) {
                // now update src
                src.position(pos + n);
            }
            return n;
        } finally {
            Util.offerFirstTemporaryDirectBuffer(bb);
        }
    }
```
主要做了两个事情:
1. 若src不是DirectByteBuffer, 那么将转变为DirectByteBuffer。其中这里还用上了LocalThread, 每个线程最多缓存1024个DirectByteBuffer, 由buffers保存。 每次首先从buffers中查找, 找不到了, 再去创建DirectByteBuffer。读/写存放数据的ByteBuffer必须是DirectByteBuffer, 主要原因是HeapByteBuffer由垃圾回收机管理内存, HeapByteBuffer在内存中的位置可能会被垃圾回收机转移, 而网络通信时会将内存地址传递给c组成的本地函数。若内存地址发生了改变, c收到的地址就失效了。 所以需要将内存地址放在垃圾回收器够不到的地方。
2. 将写入对象从HeapByteBuffer变为DirectByteBuffer后, 调用Util.offerFirstTemporaryDirectBuffer()进行写入。
3. 若使用完了, 通过offerFirstTemporaryDirectBuffer将直接内存放入缓存或者释放掉。
```
    private static int writeFromNativeBuffer(FileDescriptor fd, ByteBuffer bb,
                                             long position, NativeDispatcher nd) throws IOException
    {
        int pos = bb.position();
        int lim = bb.limit();
        assert (pos <= lim);
        int rem = (pos <= lim ? lim - pos : 0);
        int written = 0;
        if (rem == 0)
            return 0;
        if (position != -1) {
            written = nd.pwrite(fd,
                                ((DirectBuffer)bb).address() + pos,
                                rem, position);
        } else {
            // 能写入多少, 则写入多少
            written = nd.write(fd, ((DirectBuffer)bb).address() + pos, rem);  // nd=SocketDispatcher
        }
        if (written > 0)
            bb.position(pos + written);
        // 本地写入的大小
        return written;
    }
```
真正写入调用的是FileDispatcherImpl.c文件的Java_sun_nio_ch_FileDispatcherImpl_write0本地函数
```
JNIEXPORT jint JNICALL
Java_sun_nio_ch_FileDispatcherImpl_write0(JNIEnv *env, jclass clazz,
                              jobject fdo, jlong address, jint len)
{
    jint fd = fdval(env, fdo);
    void *buf = (void *)jlong_to_ptr(address);

    return convertReturnVal(env, write(fd, buf, len), JNI_FALSE); // write返回的是写入的成功数
}
```
实际通过write写入, 然后靠convertReturnVal对写入结果进行了判断:
```
jint
convertReturnVal(JNIEnv *env, jint n, jboolean reading)
{    // 正常写入
    if (n > 0) /* Number of bytes written */
        return n;
    else if (n == 0) {  //写入为
        if (reading) {
            return IOS_EOF; /* EOF is -1 in javaland */
        } else {
            return 0;
        }
    }
     // 临时性写入/读取失败, 比如缓存没有数据可读取, 或者缓存满了, 写不进去数据了。之后继续尝试即可, 不算异常
    else if (errno == EAGAIN)
        return IOS_UNAVAILABLE;
   // 说明是因为接收到了中断信号退出的。
    else if (errno == EINTR)
        return IOS_INTERRUPTED;
    else {
        const char *msg = reading ? "Read failed" : "Write failed";
        JNU_ThrowIOExceptionWithLastError(env, msg);
        return IOS_THROWN;
    }
}
```
这里对异常进行了稍微判断, 只要返回值不为大于0、EAGAIN或者EINTR, 就认为写入失败了。

# 总结
write/read返回值主要分为四种: 正常写入byte大小; EOF代表读取完成了; UNAVAILABLE将被转变为0, 表示写入时网络缓存没有了, 或者读取时网络缓存没有可以读取的了; 以及其他的异常类型。

参考:
https://www.oschina.net/question/138146_26027
