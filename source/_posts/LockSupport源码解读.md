---
title: LockSupport原理分析
date: 2018-11-10 17:16:56
tags: LockSupport
---
LockSupport作为并发的基础, 在CountDownLatch、ReentrantLock、Semaphore、ReentrantReadWriteLock中都是作为阻塞/唤醒线程的基本工具, 因此, 很有必要了解LockSupport的用法及原理, 本文将从LockSupport基本用法、 类主要函数及相关知识、底层基本原理三方面分别描述。
# 使用
基本用法:
```
class Thread1 extends Thread {
    public void run() {
        try {
            ......
            LockSupport.park();
            ......
        } catch (Exception e) {
        }
    }
}
Thread thread1 = new Thread1();
class Thread2 extends Thread {
    public void run() {
        try {
            ......
            LockSupport.unpark(thread1);
            ......
        } catch (Exception e) {
        }
    }
}
```
当某个线程调用LockSupport.park()时, 该线程将睡眠并同时交出cpu使用。当别的线程调用LockSupport.unpark(thread1)时, 该线程将被唤醒。可见使用及其简单且灵活。park()与unpark()调用没有先后顺序, 这也是与wait()/notify()(参考<a href="https://kkewwei.github.io/elasticsearch_learning/2016/10/27/Java%E7%BA%BF%E7%A8%8B%E7%9F%A5%E8%AF%86%E5%B0%8F%E7%BB%93/">Java 线程知识小结</a>)相比更灵活的一点。也就是说先调用了unpark(), 再调用park(), thread1也是可以继续执行而不被阻塞。后面将详细介绍该原理。

# LockSupport源码解析
## Unsafe
认识LockSupport, 我们就必须要了解Unsafe。 java作为安全的高级程序语言, 屏蔽了直接与内存打交道的途径。同时java也给我们留了一些后门:Unsafe。Unsafe只听名字就知道是个不安全的类, 它可以直接操作内存, 只有受信任的类才可以使用它。我们不能通过这种方式获取到Unsafe:`Unsafe unsafe = sun.misc.Unsafe.getUnsafe();`因为内部会检查该类的加载器是否是启动类加载器，若不是的话，则直接抛出异常；一般都是通过反射实现该类的获取：
```
// 应用类中获取Unsafe
Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
theUnsafe.setAccessible(true);
Unsafe UNSAFE = (Unsafe) theUnsafe.get(null);

//直接操纵内存修改成员变量值
Test test = new Test();
Field filed = test.getClass().getDeclaredField("a");
long ageOffset = UNSAFE.objectFieldOffset(filed);
//直接通过修改内存来修改成员变量值
UNSAFE.putInt(test, ageOffset, 100);
```
## parkBlockerOffset和parkBlocker
在LockSupport中有这么一段static代码块:
```
private static final sun.misc.Unsafe UNSAFE;
//获取parkBlocker在内存中的偏移量, 就是说谁把该线程block住了
private static final long parkBlockerOffset;
try {
    // 因为LockSupport是受信任的类, 所以才可以通过这种方式产生Unsafe
    UNSAFE = sun.misc.Unsafe.getUnsafe();
    // 线程类类型
    Class<?> tk = Thread.class;
    //先是通过反射机制获取Thread类的parkBlocker字段对象
     parkBlockerOffset = UNSAFE.objectFieldOffset(tk.getDeclaredField("parkBlocker"));
} catch (Exception ex) { throw new Error(ex); }
```
该函数可以获取到该线程类成员变量parkBlocker在内存中的偏移量parkBlockerOffset, 然后就可以通过`public static void park(Object blocker)`显示指示该类被什么阻塞的了。然后在jstack时, 将可以看到如下信息:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/LockSupport1.png" height="150" width="600"/>
提示你设置的阻塞对象是啥, park()代码如下:
```
 public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        UNSAFE.park(false, 0L);
        setBlocker(t, null);
}
```
这里也可以看出LockSupport实际调用了Unsafe.park/unpark函数。

# Unsafe底层park/unpark原理
Unsafe类中函数基本都是Native属性, 在虚拟机源代码/hotspot/src/share/vm/prims/unsafe.cpp(L1561)中, 定义了Unsafe类Native与c++语言函数之间对应关系, 如下所示:
 ```
 // These are the methods for 1.8.0
static JNINativeMethod methods_18[] = {
    {CC"getObject",        CC"("OBJ"J)"OBJ"",   FN_PTR(Unsafe_GetObject)},
    {CC"putObject",        CC"("OBJ"J"OBJ")V",  FN_PTR(Unsafe_SetObject)},
    {CC"getObjectVolatile",CC"("OBJ"J)"OBJ"",   FN_PTR(Unsafe_GetObjectVolatile)},
    {CC"putObjectVolatile",CC"("OBJ"J"OBJ")V",  FN_PTR(Unsafe_SetObjectVolatile)},
    {CC"getAddress",         CC"("ADR")"ADR,             FN_PTR(Unsafe_GetNativeAddress)},
    {CC"putAddress",         CC"("ADR""ADR")V",          FN_PTR(Unsafe_SetNativeAddress)},
    {CC"allocateMemory",     CC"(J)"ADR,                 FN_PTR(Unsafe_AllocateMemory)},
    {CC"reallocateMemory",   CC"("ADR"J)"ADR,            FN_PTR(Unsafe_ReallocateMemory)},
    {CC"freeMemory",         CC"("ADR")V",               FN_PTR(Unsafe_FreeMemory)},
    {CC"objectFieldOffset",  CC"("FLD")J",               FN_PTR(Unsafe_ObjectFieldOffset)},
    {CC"staticFieldOffset",  CC"("FLD")J",               FN_PTR(Unsafe_StaticFieldOffset)},
    {CC"staticFieldBase",    CC"("FLD")"OBJ,             FN_PTR(Unsafe_StaticFieldBaseFromField)},
    {CC"ensureClassInitialized",CC"("CLS")V",            FN_PTR(Unsafe_EnsureClassInitialized)},
    {CC"arrayBaseOffset",    CC"("CLS")I",               FN_PTR(Unsafe_ArrayBaseOffset)},
    {CC"arrayIndexScale",    CC"("CLS")I",               FN_PTR(Unsafe_ArrayIndexScale)},
    {CC"addressSize",        CC"()I",                    FN_PTR(Unsafe_AddressSize)},
    {CC"pageSize",           CC"()I",                    FN_PTR(Unsafe_PageSize)},
    {CC"defineClass",        CC"("DC_Args")"CLS,         FN_PTR(Unsafe_DefineClass)},
    {CC"allocateInstance",   CC"("CLS")"OBJ,             FN_PTR(Unsafe_AllocateInstance)},
    {CC"monitorEnter",       CC"("OBJ")V",               FN_PTR(Unsafe_MonitorEnter)},
    {CC"monitorExit",        CC"("OBJ")V",               FN_PTR(Unsafe_MonitorExit)},
    {CC"tryMonitorEnter",    CC"("OBJ")Z",               FN_PTR(Unsafe_TryMonitorEnter)},
    {CC"throwException",     CC"("THR")V",               FN_PTR(Unsafe_ThrowException)},
    {CC"compareAndSwapObject", CC"("OBJ"J"OBJ""OBJ")Z",  FN_PTR(Unsafe_CompareAndSwapObject)},
    {CC"compareAndSwapInt",  CC"("OBJ"J""I""I"")Z",      FN_PTR(Unsafe_CompareAndSwapInt)},
    {CC"compareAndSwapLong", CC"("OBJ"J""J""J"")Z",      FN_PTR(Unsafe_CompareAndSwapLong)},
    {CC"putOrderedObject",   CC"("OBJ"J"OBJ")V",         FN_PTR(Unsafe_SetOrderedObject)},
    {CC"putOrderedInt",      CC"("OBJ"JI)V",             FN_PTR(Unsafe_SetOrderedInt)},
    {CC"putOrderedLong",     CC"("OBJ"JJ)V",             FN_PTR(Unsafe_SetOrderedLong)},
    {CC"park",               CC"(ZJ)V",                  FN_PTR(Unsafe_Park)},
    {CC"unpark",             CC"("OBJ")V",               FN_PTR(Unsafe_Unpark)}
};
 ```
可以看出, 我们关注的park和unpark分别对应了Unsafe_Park与Unsafe_Unpark。 下面展示了Unsafe_Park的实现方式:
```
UNSAFE_ENTRY(void, Unsafe_Park(JNIEnv *env, jobject unsafe, jboolean isAbsolute, jlong time))
  UnsafeWrapper("Unsafe_Park");
  EventThreadPark event;
#ifndef USDT2
  HS_DTRACE_PROBE3(hotspot, thread__park__begin, thread->parker(), (int) isAbsolute, time);
#else /* USDT2 */
   HOTSPOT_THREAD_PARK_BEGIN(
                             (uintptr_t) thread->parker(), (int) isAbsolute, time);
#endif /* USDT2 */
  JavaThreadParkedState jtps(thread, time != 0);
  thread->parker()->park(isAbsolute != 0, time);   //进的是这个函数
#ifndef USDT2
  HS_DTRACE_PROBE1(hotspot, thread__park__end, thread->parker());
#else /* USDT2 */
  HOTSPOT_THREAD_PARK_END(
                          (uintptr_t) thread->parker());
#endif /* USDT2 */
  if (event.should_commit()) {
    oop obj = thread->current_park_blocker();
    event.set_klass((obj != NULL) ? obj->klass() : NULL);
    event.set_timeout(time);
    event.set_address((obj != NULL) ? (TYPE_ADDRESS) cast_from_oop<uintptr_t>(obj) : 0);
    event.commit();
  }
UNSAFE_END
```
我们可以清晰得看到, 真正调用的是thread->parker()->park(isAbsolute != 0, time)方法。
## thread和Parker
我们需要了解下vm中thread是如何组成的, JavaThread定义在/hotspot/src/share/vm/prims/runtime/thread.hpp(L1740), 关于Parker成员变量声明如下:
```
private:
  Parker*    _parker;
public:
  Parker*     parker() { return _parker; }
```
可以看出, 每个thread类中都包含一个Parker。
Parker定义在/hotspot/src/share/vm/runtime/park.hpp(L56), 定义如下:
```
class Parker : public os::PlatformParker {
private:
  //当_counter只能在0和1之间取值, 当为1时, 代表该类被unpark调用过, 更多的调用, 也不会增加_counter的值, 当该线程调用park()时, 不会阻塞, 同时_counter立刻清零。当为0时, 调用park()会被阻塞。使用volatile来修饰
  volatile int _counter ;
  Parker * FreeNext ;
  JavaThread * AssociatedWith ; // Current association
public:
  Parker() : PlatformParker() {
    _counter       = 0 ;
    FreeNext       = NULL ;
    AssociatedWith = NULL ;
  }
```
PlatformParker继承自Parker, 定义在/hotspot/src/os/linux/vm/os_linux.cpp(L234)
```
class PlatformParker : public CHeapObj<mtInternal> {
  protected:
    enum {
        REL_INDEX = 0,
        ABS_INDEX = 1
    };
    int _cur_index;  // which cond is in use: -1, 0, 1
    pthread_mutex_t _mutex [1] ;
    pthread_cond_t  _cond  [2] ; // one for relative times and one for abs.

  public:       // TODO-FIXME: make dtor private
    ~PlatformParker() { guarantee (0, "invariant") ; }

  public:
    PlatformParker() {
      int status;
      status = pthread_cond_init (&_cond[REL_INDEX], os::Linux::condAttr());
      assert_status(status == 0, status, "cond_init rel");
      status = pthread_cond_init (&_cond[ABS_INDEX], NULL);
      assert_status(status == 0, status, "cond_init abs");
      status = pthread_mutex_init (_mutex, NULL);
      assert_status(status == 0, status, "mutex_init");
      _cur_index = -1; // mark as unused
    }
};
```
说这么多, 我们主要是为了让你知道_counter, pthread_mutex_t _mutex[1], pthread_cond_t _cond[2]的存在。_mutex[1]和_cond[2]都是为了配合完成c++层面线程的阻塞与互斥等操作。

## Parker.park()
```
void Parker::park(bool isAbsolute, jlong time) {
  // Ideally we'd do something useful while spinning, such
  // as calling unpackTime().
  // Optional fast-path check:
  // Return immediately if a permit is available.
  // We depend on Atomic::xchg() having full barrier semantics
  // since we are doing a lock-free update to _counter.
   //这里通过原子操作来完成_counter清零操作。 若_counter之前>0, 那么说明之前该线程被unpark()过, 就可以直接返回而不被阻塞。
  if (Atomic::xchg(0, &_counter) > 0) return;
  Thread* thread = Thread::current();
  assert(thread->is_Java_thread(), "Must be JavaThread");  //判断一定的是java线程
  JavaThread *jt = (JavaThread *)thread; //类强制转化
  // Optional optimization -- avoid state transitions if there's an interrupt pending.
  // Check interrupt before trying to wait
  //进入睡眠等待前先检查是否有中断信号, 若有中断信号也直接返回。
  if (Thread::is_interrupted(thread, false)) {
    return;
  }
  // Next, demultiplex/decode time arguments
  timespec absTime;
  //如果是按参数小于0，或者绝对时间，那么可以直接返回
  if (time < 0 || (isAbsolute && time == 0) ) { // don't wait at all
    return;
  }
   //如果时间大于0，判断阻塞超时时间或阻塞截止日期，同时将时间赋值给absTime
  if (time > 0) {
    unpackTime(&absTime, isAbsolute, time);
  }
  // Enter safepoint region
  // Beware of deadlocks such as 6317397.
  // The per-thread Parker:: mutex is a classic leaf-lock.
  // In particular a thread must never block on the Threads_lock while
  // holding the Parker:: mutex.  If safepoints are pending both the
  // the ThreadBlockInVM() CTOR and DTOR may grab Threads_lock.
  ThreadBlockInVM tbivm(jt);
  // Don't wait if cannot get lock since interference arises from
  // unblocking.  Also. check interrupt before trying wait
  //再次检查, 如果有中断信号。直接返回; 或者申请互斥锁失败，则直接返回pthread_mutex_trylock返回0。任何其他返回值都表示错误。
  //函数pthread_mutex_trylock是POSIX 线程pthread_mutex_lock的非阻塞版本。
  if (Thread::is_interrupted(thread, false) || pthread_mutex_trylock(_mutex) != 0) {
    return;
  }
  //此时已经通过_mutex将该代码进行了互斥操作, 那么直接对_counter都是安全的
  int status ;
  如果count>0, 说明之前原子操作赋值为0没有成功。 而_counter> 0, 线程可以直接不阻塞而返回
  if (_counter > 0)  { // no wait needed
     //将_counter直接清零
    _counter = 0;
    //释放锁并返回， 返回0代表释放锁成功
    status = pthread_mutex_unlock(_mutex);
    assert (status == 0, "invariant") ; //这里会去检查一下是否成功了
    // Paranoia to ensure our locked and lock-free paths interact
    // correctly with each other and Java-level accesses.
    OrderAccess::fence(); //这个函数是HotSpot VM对JMM的内存屏障一个具体的实现函数;
    return;
  }
#ifdef ASSERT
  // Don't catch signals while blocked; let the running threads have the signals.
  // (This allows a debugger to break into the running thread.)
  sigset_t oldsigs;
  sigset_t* allowdebug_blocked = os::Linux::allowdebug_blocked_signals();
  pthread_sigmask(SIG_BLOCK, allowdebug_blocked, &oldsigs);
#endif
  OSThreadWaitState osts(thread->osthread(), false /* not Object.wait() */);
  jt->set_suspend_equivalent();
  // cleared by handle_special_suspend_equivalent_condition() or java_suspend_self()

  assert(_cur_index == -1, "invariant");
     //若没有超时时间，那么本线程将进入睡眠状态并释放cpu、释放对_mutex的锁定，等待其他线程调用pthread_cond_signal唤醒该线程；唤醒后会获取对_mutex的锁定的锁定
  if (time == 0) {
    _cur_index = REL_INDEX; // arbitrary choice when not timed
    status = pthread_cond_wait (&_cond[_cur_index], _mutex) ;
  } else {
    _cur_index = isAbsolute ? ABS_INDEX : REL_INDEX;
     //开始真正的阻塞，超时等待，或者其他线程pthread_cond_signal唤醒该线程
    status = os::Linux::safe_cond_timedwait (&_cond[_cur_index], _mutex, &absTime) ;
    if (status != 0 && WorkAroundNPTLTimedWaitHang) {
      pthread_cond_destroy (&_cond[_cur_index]) ;
      pthread_cond_init    (&_cond[_cur_index], isAbsolute ? NULL : os::Linux::condAttr());
    }
  }
  _cur_index = -1;
  assert_status(status == 0 || status == EINTR ||
                status == ETIME || status == ETIMEDOUT,
                status, "cond_timedwait");

#ifdef ASSERT
  pthread_sigmask(SIG_SETMASK, &oldsigs, NULL);
#endif
    //该线程被唤醒了, 同时也对_mutex加锁了, 置位_counter是线程安全的
  _counter = 0 ;
  //解锁_mutex
  status = pthread_mutex_unlock(_mutex) ;
  assert_status(status == 0, status, "invariant") ;
  // Paranoia to ensure our locked and lock-free paths interact
  // correctly with each other and Java-level accesses.
  OrderAccess::fence(); //内存屏障
  // If externally suspended while waiting, re-suspend
  if (jt->handle_special_suspend_equivalent_condition()) {
    jt->java_suspend_self();
  }
}
```
Parker::park主要做了如下事情:
+ 检查_counter>0(别的线程调用过unpark), 则原子操作清零。线程不用睡眠并返回。
+ 检查该线程是否有中断信号, 有的话,清掉并返回。
+ 尝试通过pthread_mutex_trylock对_mutex加锁来达到线程互斥。
+ 检查_counter是否>0, 若成立,说明第一步原子清零操作失败。检查park是否设置超时时间, 若设置了通过safe_cond_timedwait进行超时等待; 若没有设置,调用pthread_cond_wait进行阻塞等待。 这两个函数都在阻塞等待时都会放弃cpu的使用。 直到别的线程调用pthread_cond_signal唤醒
+ 直接_counter=0清零。
+ 通过pthread_mutex_unlock释放mutex的加锁。
需要了解下: safe_cond_timedwait/pthread_cond_wait在执行之前肯定已经获取了锁_mutex, 在睡眠前释放了锁, 在被唤醒之前, 首先再取唤醒锁。
## Parker.unpark()
唤醒操作相对简单:
```
void Parker::unpark() {
  int s, status ;
  //首先是互斥获取锁
  status = pthread_mutex_lock(_mutex);
  assert (status == 0, "invariant") ;
  s = _counter;
  //只要把这个状态置为1就行了，就是说多次调用unpack()没啥意义
  _counter = 1;
   //s只能为0，说明没有人调用unpark
  if (s < 1) {
    // thread might be parked
    if (_cur_index != -1) {
      // thread is definitely parked
      //线程已经处于parker状态了
      if (WorkAroundNPTLTimedWaitHang) {
       //pthread_cond_signal可以唤醒pthread_cond_wait()被&_cond[_cur_index]阻塞的线程
        status = pthread_cond_signal (&_cond[_cur_index]);
        assert (status == 0, "invariant");
        //解锁
        status = pthread_mutex_unlock(_mutex);
        assert (status == 0, "invariant");
      } else {
        status = pthread_mutex_unlock(_mutex);
        assert (status == 0, "invariant");
        status = pthread_cond_signal (&_cond[_cur_index]);
        assert (status == 0, "invariant");
      }
    } else {
    //仅仅解锁
      pthread_mutex_unlock(_mutex);
      assert (status == 0, "invariant") ;
    }
  } else {
    pthread_mutex_unlock(_mutex);
    assert (status == 0, "invariant") ;
  }
}
```
unpark()主要做了如下事情:
+ 首先获取锁_mutex。
+ 对_counter置为1, 而不管之前什么值, 这里说明无论多少函数调用unpark(), 都是无效的, 只会记录一次。
+ 检查线程是否已经被阻塞了, 若已经阻塞了,调用pthread_cond_signal唤醒唤醒。
+ 释放对_mutex的锁定。

# 总结
LockSupport在阻塞线程时, 更多的是依靠操作系统实现来进行的, 在底层实现时, 也是不忘考虑线程中断。 整体来说LockSupport灵活简单而且功能强大。
参考:
https://www.ibm.com/developerworks/cn/linux/thread/posix_thread1/index.html
https://www.ibm.com/developerworks/cn/linux/thread/posix_thread2/index.html
https://www.ibm.com/developerworks/cn/linux/thread/posix_thread3/index.html
http://www.fanyilun.me/2016/11/19/Thread.interrupt()%E7%9B%B8%E5%85%B3%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/