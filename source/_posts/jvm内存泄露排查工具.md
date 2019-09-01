---
title: jvm内存泄露排查工具
date: 2016-12-15 21:12:10
tags: perftools、jcmd、pmap
toc: true
categories: 工具学习
---
本文介绍几个不常用的内存泄露排查工具:perftools、pmap、jcmd。

## perftools
perftools是一款比较好的分析堆外内存泄漏的工具, 原理: 通过使用自己实现的libtcmalloc.so来替换原有的内存分配函数, 来达到监控内存分配的目的

#### 安装
perftools安装需要依赖:libunwind, 首先安装libunwind:
+ wget https://github.com/libunwind/libunwind/releases/download/v1.2.1/libunwind-1.2.1.tar.gz
+ tar -xvf libunwind-1.2.1.tar.gz
+ cd libunwind-1.2.1
+ ./configure  --prefix=/home/target/libunwind
+ make
+ make install
其次安装perftools:
+ wget https://github.com/gperftools/gperftools/releases/download/gperftools-2.6.1/gperftools-2.6.1.tar.gz
+ tar -xvf gperftools-2.6.1.tar.gz
+ cd gperftools-2.6.1
+ ./configure --prefix=/home/target/gperftools
+ make (安装时候, 可能会报g++: command not found, 切换到root: yum -y install gcc+ gcc-c++, 需要再次执行上一步, 否则会报异常)
+ make install
我们还需要设置一些环境变量:
export LD_PRELOAD=/home/target/gperftools/lib/libtcmalloc.so
目的：在程序启动时自动链接libtcmalloc.so
export HEAPPROFILE=/you_directory/heap.hprof
目的: 内存才能监控将产生很多.heap, 设置文件存放位置
export HEAP_PROFILE_ALLOCATION_INTERVAL=10000000
目的: 内存使用多少, 会产生一个.heap文件。 默认每使用1G, 产生一个.bin文件。

#### 使用
+ 直接启动jvm进程, perftools会监控内存使用
<img src="https://kkewwei.github.io/elasticsearch_learning/img/jvm_memory_leak.png" height="250" width="850"/>
+ 查看产生的heap.hprof.0001.heap文件
<img src="https://kkewwei.github.io/elasticsearch_learning/img/jvm_memory_leak1.png" height="450" width="850"/>
其中对每列的介绍如下:
The first column contains the direct memory use in MB(当前函数目前使用的直接内存大小).
The fourth column contains memory use by the procedure and all of its callees(当前函数及其调用者使用的直接内存).
The second and fifth columns are just percentage representations of the numbers in the first and fifth columns.
The third column is a cumulative sum of the second column (i.e., the kth entry in the third column is the sum of the first k entries in the second column.)(目前占用的总堆外内存大小)

## jcmd
jcm是一个多功能工具, 功能如下:
```
$user@host: /usr/local/java18/bin/jcmd 25266 help
25266:
The following commands are available:
JFR.stop
JFR.start
JFR.dump
JFR.check
VM.native_memory
VM.check_commercial_features
VM.unlock_commercial_features
ManagementAgent.stop
ManagementAgent.start_local
ManagementAgent.start
GC.rotate_log
Thread.print         // 与jstack功能相同
GC.class_stats
GC.class_histogram    // 与jmap -histo功能相同
GC.heap_dump         //与jmap -dump功能相同
GC.run_finalization
GC.run
VM.uptime
VM.flags                   //查看更完整的启动命令, 包括默认的参数, 比如垃圾回收器
VM.system_properties      //查看系统参数
VM.command_line          //查看启动命令
VM.version
help
```
我们将着重介绍VM.native_memory, 该参数可以支持我们追踪堆内内存的申请。 不过我们必须在进程启动时添加-XX:NativeMemoryTracking=detail参数才可用。 开启该参数将导致jvm进程5-10%性能损耗。
其参数如下:

|参数|介绍|
|:-|:-|
|summary|汇总的内存展示|
|detail|详细的内存展示, 不仅包括summary, 还包括每个内存块地址及具体分配者|
|baseline|当前线程拥有的内存块作为基准线, 与调用detail.diff summary.diff时刻的线程比较得出增量情况|
|detail.diff|内存增量的详情|
|summary.diff|内存增量的汇总情况|

VM.native_memory显示的内存包含`堆内内存、Code区域、通过unsafe.allocateMemory(DirectByteBuffer实际也是由前者产生)，但是不包含其他Native Code（C代码）申请的堆外内存`。

#### VM.native_memory
用法: jcmd pid VM.native_memory detail
```
44387:
Native Memory Tracking:
Total: reserved=1653891KB, committed=355703KB
                  //同启动参数一致
-                 Java Heap (reserved=102400KB, committed=102400KB)
                            (mmap: reserved=102400KB, committed=102400KB)
                   //类需要的内存
-                 Class (reserved=1091740KB, committed=39836KB)
                            (classes #426)
                            (malloc=34972KB #267)
                            (mmap: reserved=1056768KB, committed=4864KB)
                  //thread #57，表示57个线程，分配的总内存有57807KB，平均一个线程是1MB。
-                  Thread (reserved=57807KB, committed=57807KB)
                            (thread #57)
                            (stack: reserved=57568KB, committed=57568KB)
                            (malloc=173KB #315)
                            (arena=66KB #112)
                   //JIT的代码缓存
-                  Code (reserved=249648KB, committed=3560KB)
                            (malloc=48KB #336)
                            (mmap: reserved=249600KB, committed=3512KB)
                   //GC需要的内存
-                  GC (reserved=43020KB, committed=42824KB)
                            (malloc=39208KB #204)
                            (mmap: reserved=3812KB, committed=3616KB)
                   //供编译器自身操作使用的
-                  Compiler (reserved=135KB, committed=135KB)
                            (malloc=4KB #41)
                            (arena=131KB #3)
                   //
-                  Internal (reserved=107397KB, committed=107397KB)
                            (malloc=107365KB #1972)
                            (mmap: reserved=32KB, committed=32KB)
                   //保留字符串（Interned String）的引用与符号表引用
-                    Symbol (reserved=1374KB, committed=1374KB)
                            (malloc=918KB #92)
                            (arena=456KB #1)
     //NMT本身也需要内存
-    Native Memory Tracking (reserved=185KB, committed=185KB)
                            (malloc=107KB #1646)
                            (tracking overhead=78KB)

-               Arena Chunk (reserved=186KB, committed=186KB)
                            (malloc=186KB)

Virtual memory map:
//JVM heap内存大小100M, 地址范围0x00000000f9c00000-0x0000000100000000, 由ParallelScavengeHeap::initialize来申请的内存块
[0x00000000f9c00000 - 0x0000000100000000] reserved 102400KB for Java Heap from
    //下面这个怀疑是该函数在内存中的地址
    [0x00007ff092b43bd2] ReservedSpace::initialize(unsigned long, unsigned long, bool, char*, unsigned long, bool)+0xc2
    [0x00007ff092b4452e] ReservedHeapSpace::ReservedHeapSpace(unsigned long, unsigned long, bool, char*)+0x6e
    [0x00007ff092b1249b] Universe::reserve_heap(unsigned long, unsigned long)+0x8b
    [0x00007ff0929cc5c4] ParallelScavengeHeap::initialize()+0x84

        [0x00000000ffe80000 - 0x0000000100000000] committed 1536KB from
            [0x00007ff092a16ac3] PSVirtualSpace::expand_by(unsigned long)+0x53
            [0x00007ff092a17a85] PSYoungGen::initialize_virtual_space(ReservedSpace, unsigned long)+0x75
            [0x00007ff092a183ee] PSYoungGen::initialize(ReservedSpace, unsigned long)+0x3e
            [0x00007ff09236fc85] AdjoiningGenerations::AdjoiningGenerations(ReservedSpace, GenerationSizer*, unsigned long)+0x345

        [0x00000000f9c00000 - 0x00000000ffe80000] committed 100864KB from
            [0x00007ff092a16ac3] PSVirtualSpace::expand_by(unsigned long)+0x53
            [0x00007ff092a06b77] PSOldGen::initialize(ReservedSpace, unsigned long, char const*, int)+0xb7
            [0x00007ff09236fcda] AdjoiningGenerations::AdjoiningGenerations(ReservedSpace, GenerationSizer*, unsigned long)+0x39a
            [0x00007ff0929cc716] ParallelScavengeHeap::initialize()+0x1d6
    ......
[0x00007ff093b40000 - 0x00007ff093c41000] reserved and committed 1028KB for Thread Stack from
    [0x00007ff092af4fd6] Threads::create_vm(JavaVMInitArgs*, bool*)+0x1e6
    [0x00007ff09275e244] JNI_CreateJavaVM+0x74
    [0x00007ff09360245e] JavaMain+0x9e

[0x00007ff093c49000 - 0x00007ff093c4b000] reserved 8KB for GC from
    [0x00007ff092b43d66] ReservedSpace::initialize(unsigned long, unsigned long, bool, char*, unsigned long, bool)+0x256
    [0x00007ff092b43e0b] ReservedSpace::ReservedSpace(unsigned long, unsigned long, bool, char*, unsigned long)+0x1b
    [0x00007ff092a0a1c2] ParallelCompactData::create_vspace(unsigned long, unsigned long)+0x92
    [0x00007ff092a0c718] PSParallelCompact::initialize()+0x178

        [0x00007ff093c49000 - 0x00007ff093c4b000] committed 8KB from
            [0x00007ff092a16ac3] PSVirtualSpace::expand_by(unsigned long)+0x53
            [0x00007ff092a0a274] ParallelCompactData::create_vspace(unsigned long, unsigned long)+0x144
            [0x00007ff092a0c718] PSParallelCompact::initialize()+0x178
            [0x00007ff0929cc8c5] ParallelScavengeHeap::initialize()+0x385
Details:

[0x00007ff092b1ad2b] Unsafe_AllocateMemory+0x1db
[0x00007ff07d015994]
                             //展示了DirectByteBuffer内存申请情况, 供92个内存块, 总共占用94MB大小
                             (malloc=94208KB #92)

[0x00007ff0923cca65] ArrayAllocator<unsigned long, (MemoryType)7>::allocate(unsigned long)+0x175
[0x00007ff092a0092f] ParCompactionManager::initialize(ParMarkBitMap*)+0x28f
[0x00007ff092a0a105] PSParallelCompact::post_initialize()+0xb5
[0x00007ff0929cc535] ParallelScavengeHeap::post_initialize()+0x25
                             (malloc=34816KB #34)

[0x00007ff092a11d45] ArrayAllocator<StarTask, (MemoryType)1>::allocate(unsigned long)+0x175
[0x00007ff092a10160] PSPromotionManager::PSPromotionManager()+0x1f0
[0x00007ff092a106ad] PSPromotionManager::initialize()+0x13d
[0x00007ff092b11f43] universe_post_init()+0x883
                             (malloc=34816KB #34)
```
至于每个内存块存放的详情, 可以在pmap里查看。可见, 我们可以根据detail可以了解每块内存是由谁申请的。

#### 内存增量diff
执行顺序:
+ /usr/local/java18/bin/jcmd 8202 VM.native_memory baseline 作为内存基准
+ /usr/local/java18/bin/jcmd 8202 VM.native_memory detail.diff 根据内存基准来比较增量
结果如下:
```
Native Memory Tracking:
Total: reserved=1617492KB +13331KB, committed=319304KB +13331KB
-                 Java Heap (reserved=102400KB, committed=102400KB)
                            (mmap: reserved=102400KB, committed=102400KB)
                          ......
                 Internal (reserved=70999KB +13312KB, committed=70999KB +13312KB)
                            (malloc=70967KB +13312KB #1914 +13)
                            (mmap: reserved=32KB, committed=32KB)

-                    Symbol (reserved=1374KB, committed=1374KB)
                            (malloc=918KB #92)
                            (arena=456KB #1)

-    Native Memory Tracking (reserved=183KB +19KB, committed=183KB +19KB)
                            (malloc=106KB +15KB #1628 +212)
                            (tracking overhead=77KB +4KB)

-               Arena Chunk (reserved=186KB, committed=186KB)
                            (malloc=186KB)

[0x00007f5c46ed7d2b] Unsafe_AllocateMemory+0x1db
[0x00007f5c31015994]
                             (malloc=35840KB +13312KB #35 +13)

[0x00007f5c46cc4883] MemBaseline::aggregate_virtual_memory_allocation_sites()+0x173
[0x00007f5c46cc4b08] MemBaseline::baseline_allocation_sites()+0x1d8
[0x00007f5c46cc5185] MemBaseline::baseline(bool)+0x635
[0x00007f5c46d34188] NMTDCmd::execute(DCmdSource, Thread*)+0x178
                             (malloc=1KB +1KB #17 +17)
```
如上图, 可以轻易发现Unsafe_AllocateMemory diff期间,增加了13次内存分配, 新增了13MB内存申请。

## pmap
pmap可以查看进程使用的内存分布, 包括所有堆内和堆外内存。
使用: pmap -x 9752 | sort -n -r -k 2, 并不需要root或者别的参数就可以执行, 查看进程的内存映射信息
```
mapped: 5341520K    writeable/private: 308668K    shared: 3836K
Address           Kbytes       Mode   Offset Device  Mapping
0000000100080000 1048064       0       0 -----    [ anon ]
00007fb789360000  242304       0       0 -----    [ anon ]
00000000f9c00000  102912   23776   23776 rw---    [ anon ]
00007fb6a216c000   96848      44       0 r----  locale-archive
00007fb784021000   65404       0       0 -----    [ anon ]
00007fb780021000   65404       0       0 -----    [ anon ]
00007fb77c021000   65404       0       0 -----    [ anon ]
00007fb778021000   65404       0       0 -----    [ anon ]
00007fb774021000   65404       0       0 -----    [ anon ]
00007fb770021000   65404       0       0 -----    [ anon ]
00007fb76c021000   65404       0       0 -----    [ anon ]
00007fb768021000   65404       0       0 -----    [ anon ]
00007fb764021000   65404       0       0 -----    [ anon ]
00007fb760021000   65404       0       0 -----    [ anon ]
00007fb75c021000   65404       0       0 -----    [ anon ]
00007fb758021000   65404       0       0 -----    [ anon ]
00007fb750021000   65404       0       0 -----    [ anon ]
00007fb74c021000   65404       0       0 -----    [ anon ]
00007fb748021000   65404       0       0 -----    [ anon ]
00007fb740021000   65404       0       0 -----    [ anon ]
00007fb738021000   65404       0       0 -----    [ anon ]
00007fb730021000   65404       0       0 -----    [ anon ]
00007fb728021000   65404       0       0 -----    [ anon ]
00007fb720021000   65404       0       0 -----    [ anon ]
00007fb718021000   65404       0       0 -----    [ anon ]
00007fb710021000   65404       0       0 -----    [ anon ]
00007fb708021000   65404       0       0 -----    [ anon ]
00007fb700021000   65404       0       0 -----    [ anon ]
00007fb7a1802000    2048       0       0 -----  libpthread-2.12.so
00007fb7a15ea000    2048       0       0 -----  libjli.so
```
Address:  start address of map  映像起始地址, 就是真实内存地址
Kbytes:  size of map in kilobytes  映像大小
RSS:  resident set size in kilobytes  驻留集大小
Dirty:  dirty pages (both shared and private) in kilobytes  脏页大小
Mode:  permissions on map 映像权限: r=read, w=write, x=execute, s=shared, p=private (copy on write)
Mapping:  file backing the map , or '[ anon ]' for allocated memory, or '[ stack ]' for the program stack.  映像支持文件,[anon]为已分配内存 [stack]为程序堆栈
Offset:  offset into the file  文件偏移
Device:  device name (major:minor)  设备名
第一行的值:
+ mapped 表示该进程映射的虚拟地址空间大小，也就是该进程预先分配的虚拟内存大小，即ps出的vsz
+ writeable/private  表示进程所占用的私有地址空间大小，也就是该进程实际使用的内存大小
+ shared 表示进程和其他进程共享的内存大小
根据此处的address和NMT可判断出该内存块属于哪个空间的。我们可以通过如下方式检查是否存在内存泄露
`while true; do pmap -d  3066 | tail -1; sleep 1; done`
```
mapped: 5411424K    writeable/private: 378492K    shared: 3836K
mapped: 5412452K    writeable/private: 379520K    shared: 3836K
mapped: 5413480K    writeable/private: 380548K    shared: 3836K
mapped: 5414508K    writeable/private: 381576K    shared: 3836K
mapped: 5415536K    writeable/private: 382604K    shared: 3836K
```
# 参考
http://goog-perftools.sourceforge.net/doc/heap_profiler.html
https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr007.html
https://www.cnblogs.com/duanxz/archive/2012/08/09/2630284.html
https://tech.meituan.com/2019/01/03/spring-boot-native-memory-leak.html
https://www.cnblogs.com/sidesky/p/10009241.html
https://yq.aliyun.com/articles/227924
https://www.cnblogs.com/ggjucheng/p/3348439.html