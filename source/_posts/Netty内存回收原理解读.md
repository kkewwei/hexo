---
title: Netty堆外内存回收原理解读
date: 2019-01-12 15:22:41
tags:
---
Netty堆外内存通过DirectByteBuffer实现管理, 它会首先申请16M的直接内存块大小, 放入DirectByteBuffer, 由PoolChunk映射这16MB的内存块, 通过PoolChunk的分配来完成该直接内存块使用与释放。 每当用户申请小块内存时, 都从这16M的内存中分配, 当该部分内存使用完后, 会释放到PoolChunk内存池中, 而不是彻底释放。 可以看出, netty每次释放直接内存并没有使用DirectByteBuffer自带Cleaner来释放(具体可以参考<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/27/DirectByteBuffer%E5%A0%86%E5%A4%96%E5%86%85%E5%AD%98%E8%AF%A6%E8%A7%A3/">DirectByteBuffer堆外内存详解</a>), 使用PoolChunk管理直接内存的使用情况的好处也是很清晰的: 直接申请与释放堆外内存是个很大的开销, 若通过PoolChunk管理直接内存使用后, 可以循环使用该部分直接内存, 这样才能满足netty的高性能特性。 本文将讲述netty释放直接内存的原理及细节。
而DirectByteBuffer封装在PooledUnsafeDirectByteBuf, netty层面也主要操作后者, 两者的关系图如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/Netty堆外内存回收1.png" height="250" width="350"/>
了解这两者对应关系, 对理解该文有一定的帮助。

# PlatformDependent及PlatformDependent0简介
PlatformDependent及PlatformDependent0主要是用来确定重要参数配置的, 比如netty是否需要使用Unsfa, 当前使用的java版本等, 了解这些参数变量, 有助于更方面了解直接内存的使用。
### PlatformDependent
```
    //是否有Unsafe, 拥有了Unsafe, 我们可以方便的操控直接内存,可以通过-Dio.netty.noUnsafe及-Dio.netty.tryUnsafe参数来决定, 默认是可以
    private static final boolean HAS_UNSAFE = hasUnsafe0();
    //用户优先使用堆外内存还是堆内内存, 通过-Dio.netty.noPreferDirect参数控制
    private static final boolean DIRECT_BUFFER_PREFERRED =HAS_UNSAFE && !SystemPropertyUtil.getBoolean("io.netty.noPreferDirect", false);
    //当前netty能够使用的最大堆外内存, 默认与堆内内存相等
    private static final long MAX_DIRECT_MEMORY = maxDirectMemory0();
     //与unsafe结合可以直接获取任何堆内array对象的直接地址
    private static final long BYTE_ARRAY_BASE_OFFSET = byteArrayBaseOffset0();
    //与unsafe结合可以获取任何对象的直接内存地址
    private static final int ADDRESS_SIZE = addressSize0();
     //默认为true，则代表着netty将自己通过PoolCHunk来实现直接内存的回收与分配, 而不是使用DirectByteBuffer自带的功能来回收直接内存
    private static final boolean USE_DIRECT_BUFFER_NO_CLEANER;
    //直接内存地址的使用量, 每次新申请和释放16M堆外内存, 都会统计到该变量中, 实际项目中, 我们可以通过反射拿到该变量值以观察堆外内存是否出现泄漏
    private static final AtomicLong DIRECT_MEMORY_COUNTER;
    //最大堆外内存
    private static final long DIRECT_MEMORY_LIMIT;
    private static final ThreadLocalRandomProvider RANDOM_PROVIDER;
    //自己弄一个cleaner, 当16M内存被完全释放时, 会调用该变量完成。
    private static final Cleaner CLEANER;
    static {
         ......
        long maxDirectMemory = SystemPropertyUtil.getLong("io.netty.maxDirectMemory", -1);
        //PlatformDependent0.hasDirectBufferNoCleanerConstructor()会检查可以通过反射+直接内存地址来构建DirectByteBuffer对象
        if (maxDirectMemory == 0 || !hasUnsafe() || !PlatformDependent0.hasDirectBufferNoCleanerConstructor()) {
            USE_DIRECT_BUFFER_NO_CLEANER = false;
            DIRECT_MEMORY_COUNTER = null;
        } else {
            //代表释放DirectByteBuffer里面的直接内存, 不使用自带的cleaner机制, 通过UNSAFE.freeMemory(address)释放的。具体见PoolArea.destroyChunk()方法选择。
            USE_DIRECT_BUFFER_NO_CLEANER = true;
            if (maxDirectMemory < 0) {
                maxDirectMemory = maxDirectMemory0();
                if (maxDirectMemory <= 0) {
                    DIRECT_MEMORY_COUNTER = null;
                } else {
                    DIRECT_MEMORY_COUNTER = new AtomicLong();
                }
            } else {
                DIRECT_MEMORY_COUNTER = new AtomicLong();
            }
        }
        DIRECT_MEMORY_LIMIT = maxDirectMemory;
        if (!isAndroid() && hasUnsafe()) {
            if (javaVersion() >= 9) {
                CLEANER = CleanerJava9.isSupported() ? new CleanerJava9() : NOOP;
            } else {
                CLEANER = CleanerJava6.isSupported() ? new CleanerJava6() : NOOP;
            }
        } else {
            CLEANER = NOOP;
        }
    }
```
我们可以看下CleanerJava6如何进行直接内存释放的:
```
final class CleanerJava6 implements Cleaner {
    //随便产生了一个DircetByteBuff，里面的cleaner的直接内存地址
    private static final long CLEANER_FIELD_OFFSET;
    //DircetByteBuff中cleaner的直接内存地址
    private static final Method CLEAN_METHOD;
    static {
        long fieldOffset = -1;
        Method clean = null;
        Throwable error = null;
        if (PlatformDependent0.hasUnsafe()) {
            //先产生一个DirectByteBuffer
            ByteBuffer direct = ByteBuffer.allocateDirect(1);
            try {
            //根据反射, 获取到该对象的cleaner域
                Field cleanerField = direct.getClass().getDeclaredField("cleaner");
                fieldOffset = PlatformDependent0.objectFieldOffset(cleanerField);
                Object cleaner = PlatformDependent0.getObject(direct, fieldOffset);
                clean = cleaner.getClass().getDeclaredMethod("clean");
                //调用clean，测试回收DirectByteBuffer里面的直接内存
                clean.invoke(cleaner);
            } catch (Throwable t) {
                // We don't have ByteBuffer.cleaner().
                fieldOffset = -1;
                clean = null;
                error = t;
            }
        } else {
            error = new UnsupportedOperationException("sun.misc.Unsafe unavailable");
        }
        if (error == null) {
            logger.debug("java.nio.ByteBuffer.cleaner(): available");
        } else {
            logger.debug("java.nio.ByteBuffer.cleaner(): unavailable", error);
        }
        CLEANER_FIELD_OFFSET = fieldOffset; //DircetByteBuff中cleaner的直接内存地址
        CLEAN_METHOD = clean;   ////DircetByteBuff中cleaner的直接内存地址
    }
    @Override
    public void freeDirectBuffer(ByteBuffer buffer) {
        if (!buffer.isDirect()) {
            return;
        }
        try {
            //获取该buffer里面的cleanerd对象
            Object cleaner = PlatformDependent0.getObject(buffer, CLEANER_FIELD_OFFSET);
            if (cleaner != null) {
                //主动调用clean()函数以回收该内存
                CLEAN_METHOD.invoke(cleaner);
            }
        } catch (Throwable cause) {
            PlatformDependent0.throwException(cause);
        }
    }
}
```
使用CleanerJava6释放直接内存的一个前提就是存在cleaner成员变量, 当使用DirectByteBuffer(int cap)产生的DirectByteBuffer, 其cleaner才会存在。 而使用private DirectByteBuffer(long addr, int cap)则不会产生cleaner对象, 而netty默认使用后者产生DirectByteBuffer对象(见<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/20/Netty-PoolChunk%E5%8E%9F%E7%90%86%E6%8E%A2%E7%A9%B6/">allocateNormal.allocateDirect()</a>)。
### PlatformDependent0
```
    //与Unsafe配合, 可以获取任何对象任何成员变量的值
    private static final long ADDRESS_FIELD_OFFSET;
    //可以通过该地址直接获取数组在内存中的初始地址， 参考https://www.jianshu.com/p/9819eb48716a
    private static final long BYTE_ARRAY_BASE_OFFSET;
    //DirectByteBuff对象构造器，参数包括直接内存address
    private static final Constructor<?> DIRECT_BUFFER_CONSTRUCTOR;
    //是否明确不让使用Unsafe
    private static final boolean IS_EXPLICIT_NO_UNSAFE = explicitNoUnsafe0();
    //通过Unsafe来控制直接内存
    private static final Object INTERNAL_UNSAFE;
    static final Unsafe UNSAFE;
    private static final boolean UNALIGNED;
    static {
        final ByteBuffer direct;
        Field addressField = null;
        Method allocateArrayMethod = null;
        Unsafe unsafe;
        Object internalUnsafe = null;
        if (isExplicitNoUnsafe()) {
            direct = null;
            addressField = null;
            unsafe = null;
            internalUnsafe = null;
        } else {
            //这里只是尝试分配一个内存
            direct = ByteBuffer.allocateDirect(1);
            // 尝试通过反射获取Unsafe对象
            final Object maybeUnsafe = AccessController.doPrivileged(new PrivilegedAction<Object>() {
                @Override
                public Object run() {
                    try {
                        //反射， 强制获取该类属性
                        final Field unsafeField = Unsafe.class.getDeclaredField("theUnsafe");
                        Throwable cause = ReflectionUtil.trySetAccessible(unsafeField);
                        if (cause != null) {
                            return cause;
                        }
                        // the unsafe instance
                        return unsafeField.get(null);
                    } catch (NoSuchFieldException e) {
                        ......
                        return e;
                    }
                }
            });
            if (maybeUnsafe instanceof Exception) {
                unsafe = null;
            } else {
                unsafe = (Unsafe) maybeUnsafe;
            }
            //检车copyMemory可用
            if (unsafe != null) {
                final Unsafe finalUnsafe = unsafe;
                final Object maybeException = AccessController.doPrivileged(new PrivilegedAction<Object>() {
                    @Override
                    public Object run() {
                        try {
                            finalUnsafe.getClass().getDeclaredMethod(
                                    "copyMemory", Object.class, long.class, Object.class, long.class, long.class);
                            return null;
                        } catch (NoSuchMethodException e) {
                            ......
                        }
                    }
                });
                if (maybeException == null) {
                    logger.debug("sun.misc.Unsafe.copyMemory: available");
                } else {
                    // Unsafe.copyMemory(Object, long, Object, long, long) unavailable.
                    unsafe = null;
                }
            }
            if (unsafe != null) {
                //测试通过Unsafe获取DirectByteBuffer.address()可用
                final Unsafe finalUnsafe = unsafe;
                final Object maybeAddressField = AccessController.doPrivileged(new PrivilegedAction<Object>() {
                    @Override
                    public Object run() {
                        try {
                            final Field field = Buffer.class.getDeclaredField("address");
                            final long offset = finalUnsafe.objectFieldOffset(field);
                            final long address = finalUnsafe.getLong(direct, offset);
                            if (address == 0) {
                                return null;
                            }
                            return field;
                        } catch (NoSuchFieldException e) {
                           ......
                        }
                    }
                });
                if (maybeAddressField instanceof Field) {
                    addressField = (Field) maybeAddressField;
                } else {
                    unsafe = null;
                }
            }
            if (unsafe != null) {
                long byteArrayIndexScale = unsafe.arrayIndexScale(byte[].class);
                if (byteArrayIndexScale != 1) {
                    unsafe = null;
                }
            }
        }
        UNSAFE = unsafe;
        if (unsafe == null) {
            ADDRESS_FIELD_OFFSET = -1;
            BYTE_ARRAY_BASE_OFFSET = -1;
            UNALIGNED = false;
            DIRECT_BUFFER_CONSTRUCTOR = null;
            ALLOCATE_ARRAY_METHOD = null;
        } else {
            //构建通过private DirectByteBuffer(long addr, int cap)获取DirectByteBuffer对象的构造体
            Constructor<?> directBufferConstructor;
            long address = -1;
            try {
                //返回的是DirectByteBuff的构造器
                final Object maybeDirectBufferConstructor =
                        AccessController.doPrivileged(new PrivilegedAction<Object>() {
                            @Override
                            public Object run() {
                                try {
                                    final Constructor<?> constructor =  //对应DirectByteBuffer(long addr, int cap)
                                            direct.getClass().getDeclaredConstructor(long.class, int.class);
                                    Throwable cause = ReflectionUtil.trySetAccessible(constructor);
                                    if (cause != null) {
                                        return cause;
                                    }
                                    return constructor;
                                } catch (NoSuchMethodException e) {
                                    return e;
                                } catch (SecurityException e) {
                                    return e;
                                }
                            }
                        });
                if (maybeDirectBufferConstructor instanceof Constructor<?>) {
                    address = UNSAFE.allocateMemory(1);
                    try {
                         //这里是尝试产生一个直接内存测试一用
                        ((Constructor<?>) maybeDirectBufferConstructor).newInstance(address, 1);
                        directBufferConstructor = (Constructor<?>) maybeDirectBufferConstructor;
                    } catch (InstantiationException e) {
                        directBufferConstructor = null;
                        ......
                    }
                } else {
                    directBufferConstructor = null;
                }
            } finally {
                if (address != -1) {
                   //测试完成后再释放这个节点
                    UNSAFE.freeMemory(address);
                }
            }
            DIRECT_BUFFER_CONSTRUCTOR = directBufferConstructor;
            ADDRESS_FIELD_OFFSET = objectFieldOffset(addressField);
            BYTE_ARRAY_BASE_OFFSET = UNSAFE.arrayBaseOffset(byte[].class);
        }
    }
```
# 直接内存的释放
当数据通过channel发送出去后(见<a href="https://kkewwei.github.io/elasticsearch_learning/2018/05/04/Netty-Http%E9%80%9A%E4%BF%A1%E7%BC%96%E7%A0%81%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB/">Netty Http通信源码二(编码)分析</a>), 然后就开始准备着释放直接内存
```
      // Release the fully written buffers, and update the indexes of the partially written buffer.
     in.removeBytes(writtenBytes); //释放内存资源
```
最后调用的remove():
```
     public boolean remove() {
        Entry e = flushedEntry;
        if (e == null) {
            clearNioBuffers();
            return false;
        }
        Object msg = e.msg; //PooledUnsafeDirectByteBuf
        ChannelPromise promise = e.promise;
        int size = e.pendingSize;
        removeEntry(e);
        if (!e.cancelled) {
            // only release message, notify and decrement if it was not canceled before.
            ReferenceCountUtil.safeRelease(msg); //释放了直接内存地址，
            safeSuccess(promise);
            decrementPendingOutboundBytes(size, false, true);
        }
        // recycle the entry
        e.recycle(); //释放Entry
        return true;
    }
```
remove主要做了三件事:
+ 调用removeEntry(e)清理ChannelOutboundBuffer里面的缓存链表。
+ 调用ReferenceCountUtil.safeRelease(msg)释放该对象的引用次数, 当引用次数为0时, 那么将直接调用PooledByteBuf.deallocate()释放该ByteBuffer。
+ 调用e.recycle()来回收Entry(原理见<a href="https://kkewwei.github.io/elasticsearch_learning/2018/05/04/Netty-Http%E9%80%9A%E4%BF%A1%E7%BC%96%E7%A0%81%E6%BA%90%E7%A0%81%E9%9dsdsdsdsd8%85%E8%AF%BB/">Ndsdsdsd源码二(编码)分析</a>)
```
    protected final void deallocate() {
        if (handle >= 0) {
            final long handle = this.handle;
            this.handle = -1;
            memory = null;
            tmpNioBuf = null;
            chunk.arena.free(chunk, handle, maxLength, cache);
            chunk = null;
            recycle(); //释放
        }
    }
```
主要做了两件事:
+ 释放直接内存DirectByteBuffer占用的内存。
+ 释放PooledUnsafeDirectByteBuf对象, 使其回收进入对象池以便下次继续使用该对象。
我们接着看area.free()是怎么操作的
```
    void free(PoolChunk<T> chunk, long handle, int normCapacity, PoolThreadCache cache) {
        if (chunk.unpooled) {
            int size = chunk.chunkSize();
            destroyChunk(chunk);
            activeBytesHuge.add(-size);
            deallocationsHuge.increment();
        } else {
            SizeClass sizeClass = sizeClass(normCapacity);
            if (cache != null && cache.add(this, chunk, handle, normCapacity, sizeClass)) {  //放入该cache的缓存队列
                // cached so not free it.
                return;
            }

            freeChunk(chunk, handle, sizeClass);
        }
    }
```
这里做判断, 针对该对象是否池化做了不同判断:
+ 针对非池化内存, 直接将该内存块给释放了
+ 针对池化内存:
1. 检查内存属性为tiny、small、normal中的一种
2. 查找是否有该属性的缓存, 在<a href="https://kkewwei.github.io/elasticsearch_learning/2018/07/14/Netty-PoolThreadCache%E6%BA%90%E7%A0%81%E6%8E%A2%E7%A9%B6/">Netty PoolThreadCache原理探究</a>我们知道, 缓存的范围只在[16B, 32kB]之间, 若内存块在这范围之内, 则将内存块放入对应的缓存中
3. 若内存块>32KB, 那么将调用freeChunk()该内存块释放到公共内存池中。
```
void freeChunk(PoolChunk<T> chunk, long handle, SizeClass sizeClass) {
        final boolean destroyChunk;
        synchronized (this) {
            destroyChunk = !chunk.parent.free(chunk, handle);
        }
        if (destroyChunk) {
            // destroyChunk not need to be called while holding the synchronized lock.
            destroyChunk(chunk);
        }
    }
```
freeChunk做了如下检查:
调用free来释放PoolChunkList中对应的节点
```
    boolean free(PoolChunk<T> chunk, long handle) {
        chunk.free(handle);
        if (chunk.usage() < minUsage) {
            //若当前PoolChunk使用率不满足当前链节点最低使用率要求, 从当前PoolchunkList中删掉chunk
            remove(chunk);
            // 根据目前PoolChunk使用率, 逐次检查q100->qinit的节点, 直到找到一个满足条件的节点
            return move0(chunk);
        }
        return true;
    }
     private boolean move0(PoolChunk<T> chunk) {
         //若找到当前链节点是qinit链， 那么说明该节点使用为0, PoolChunk对应的16MB内存完全空闲, 将会释放该直接内存块
        if (prevList == null) {
            // There is no previous PoolChunkList so return false which result in having the PoolChunk destroyed and
            // all memory associated with the PoolChunk will be released.
            assert chunk.usage() == 0;
            return false;
        }
        return prevList.move(chunk); //移动到前面的链中
    }
```
释放PoolChunk对应的16M的内存块的过程如下:
```
        protected void destroyChunk(PoolChunk<ByteBuffer> chunk) {
            if (PlatformDependent.useDirectBufferNoCleaner()) {
                //将直接通过显示调用UNSAFE.freeMemory(address)方式释放内存, 并修改DIRECT_MEMORY_COUNTER值
                PlatformDependent.freeDirectNoCleaner(chunk.memory);
            } else {
                //如果用cleaner回收内存, 将调用CleanerJava6.freeDirectBuffer()释放内存
                PlatformDependent.freeDirectBuffer(chunk.memory);
            }
        }
```
# 总结
整个netty池化内存回收过程如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/Netty堆外内存回收.png" height="400" width="950"/>
netty默认释放管理直接内存方式与DirectByteBuffer默认释放内存的方式不一致, 释放时会依次检查缓存、公共内存池, 若Poolchunk使用率为0, 那么16M直接内存将直接释放。