---
title: gdb调试java基本用法
date: 2016-12-20 15:56:11
tags: gdb
toc: true
---
jvm进程崩溃时, 可以产生coredump文件, 该dump文件记录了崩溃时cpu、jvm当前线程、当前内存的使用情况, 学会分析该coredump对于我们排查问题十分重要, 本文将主要讲解如何通过gdb分析该coredump文件。
## coredump产生
第一步检查系统是否允许产生coredump文件
操作系统默认禁止coredump文件的产生, 可以通过`ulimit -c`查看是否允许产生coredump文件, 若显示为0, 则禁止线程崩溃时产生该core文件, 可以通过`echo "ulimit  -c unlimited" >> /etc/profile`. 我们也可以通过cat /pro/$pid/limits 查看设置情况
```
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            8388608              unlimited            bytes
Max core file size        unlimited            unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             23262                23262                processes
Max open files            4096                 4096                 files
Max locked memory         65536                65536                bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       23262                23262                signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us
```
可以看到Max core file size设置为unlimited, 是运行产生core 文件的。
第二步, 通过kill -6 pid 向进程发送abort信号产生core文件(默认文件名称)。
## gdb基本使用

|命令|简写|说明|
|:-|:-|:-|
|list|l|显示多行源代码|
|break|b|设置断点,程序运行到断点的位置会停下来|
|info|i|描述程序的状态|
|run||r|开始运行程序
|display|disp|跟踪查看某个变量,每次停下来都显示它的值|
|step|s|执行下一条语句,如果该语句为函数调用,则进入函数执行其中的第一条语句|
|next|n|执行下一条语句,如果该语句为函数调用,不会进入函数内部执行(即不会一步步地调试函数内部语句)|
|print|p|打印内部变量值|
|continue|c|继续程序的运行,直到遇到下一个断点|
|set var name=v||设置变量的值|
|start|st|开始执行程序,在main函数的第一条语句前面停下来|
|file||装入需要调试的程序|
|kill|k|终止正在调试的程序|
|watch||监视变量值的变化|
|backtrace|bt|产看函数调用信息(堆栈)|
|frame|f|查看栈帧, 比如打印当前线程栈第5行详细信息: f 5|
|quit|q|退出GDB环境|
(参考<a href="https://blog.csdn.net/zdy0_2004/article/details/80102076">gdb调试的基本使用</a>)

## 分析coredump文件
+ gdb java core
调用如上命令加载core文件(这里的core指的是core文件名称)。
1. 执行`info threads`, 其实可以通过`info`查看需要详细数据
```
#0  0xb776b424 in __kernel_vsyscall ()
(gdb) info threads
  Id   Target Id         Frame
  12   Thread 0xb6a84b40 (LWP 5553) 0xb776b424 in __kernel_vsyscall ()
  11   Thread 0xad2ffb40 (LWP 5563) 0xb776b424 in __kernel_vsyscall ()
  10   Thread 0xad350b40 (LWP 5562) 0xb776b424 in __kernel_vsyscall ()
  9    Thread 0xad3d1b40 (LWP 5561) 0xb776b424 in __kernel_vsyscall ()
  8    Thread 0xad422b40 (LWP 5560) 0xb776b424 in __kernel_vsyscall ()
  7    Thread 0xad473b40 (LWP 5559) 0xb776b424 in __kernel_vsyscall ()
  6    Thread 0xad6c4b40 (LWP 5558) 0xb776b424 in __kernel_vsyscall ()
  5    Thread 0xad715b40 (LWP 5557) 0xb776b424 in __kernel_vsyscall ()
  4    Thread 0xad796b40 (LWP 5556) 0xb776b424 in __kernel_vsyscall ()
  3    Thread 0xadee6b40 (LWP 5555) 0xb776b424 in __kernel_vsyscall ()
  2    Thread 0xb4817b40 (LWP 5554) 0xb776b424 in __kernel_vsyscall ()
* 1    Thread 0xb756f700 (LWP 5552) 0xb776b424 in __kernel_vsyscall ()
(gdb)
```
这里将展示目前已知的所有线程, *开头的代表当前处于调试的线程。这里可以通过输入info可以查看所有的命令。 切换当前调试线程可以通过`thread ID`方式, id取值上图第一列。其实我们查看每个线程在干什么, 也可以直接使用`pstack pid`即可
2. 假如想查看id为1的线程当前栈调用情况, 那么执行继续`bt`
```
#0  0xb776b424 in __kernel_vsyscall ()
(gdb) bt
#0  0xb776b424 in __kernel_vsyscall ()
#1  0xb7742178 in pthread_join (threadid=3064482624, thread_return=0xbfb1b944) at pthread_join.c:92
#2  0xb772eeda in ContinueInNewThread0 () from /home/kewei/Downloads/work_soft/jdk1.8.0_121/bin/../lib/i386/jli/libjli.so
#3  0xb772a266 in ContinueInNewThread () from /home/kewei/Downloads/work_soft/jdk1.8.0_121/bin/../lib/i386/jli/libjli.so
#4  0xb772edab in JVMInit () from /home/kewei/Downloads/work_soft/jdk1.8.0_121/bin/../lib/i386/jli/libjli.so
#5  0xb772d149 in JLI_Launch () from /home/kewei/Downloads/work_soft/jdk1.8.0_121/bin/../lib/i386/jli/libjli.so
#6  0x0804858d in main ()
```
3. 查看每个register值: `info registers`(简写`i r`)
```
        0xfffffe00	-512
ecx            0x0	0
edx            0x25d5	9685
ebx            0xb6a5dba8	-1230644312
esp            0xbfd1a6e0	0xbfd1a6e0
ebp            0xb6a5db40	0xb6a5db40
esi            0x0	0
edi            0xb772c000	-1217216512
eip            0xb7744424	0xb7744424 <__kernel_vsyscall+16>
eflags         0x246	[ PF ZF IF ]
......
```
左边是寄存器名称, 中间是寄存器地址, 右边是寄存器值, 可以通过`print $ecx`验证寄存器值
+ 查看反汇编语言
```
(gdb) disassemble
Dump of assembler code for function __kernel_vsyscall:
   0xb7744414 <+0>:	push   %ecx
   0xb7744415 <+1>:	push   %edx
   0xb7744416 <+2>:	push   %ebp
   0xb7744417 <+3>:	mov    %esp,%ebp
   0xb7744419 <+5>:	sysenter
   0xb774441b <+7>:	nop
   0xb774441c <+8>:	nop
   0xb774441d <+9>:	nop
   0xb774441e <+10>:	nop
   0xb774441f <+11>:	nop
   0xb7744420 <+12>:	nop
   0xb7744421 <+13>:	nop
   0xb7744422 <+14>:	int    $0x80
=> 0xb7744424 <+16>:	pop    %ebp
   0xb7744425 <+17>:	pop    %edx
   0xb7744426 <+18>:	pop    %ecx
   0xb7744427 <+19>:	ret
End of assembler dump.
```
也可以反汇编一个函数`disass func_name`, 如下:
```
(gdb) disass main
Dump of assembler code for function main:
   0x0000000000400620 <+0>:	push   %rbp
   0x0000000000400621 <+1>:	mov    %rsp,%rbp
   0x0000000000400624 <+4>:	sub    $0x40,%rsp
   0x0000000000400628 <+8>:	mov    0x200271(%rip),%rdx        # 0x6008a0 <const_launcher>
   0x000000000040062f <+15>:	test   %rdx,%rdx
   0x0000000000400632 <+18>:	je     0x4006a0 <main+128>
   0x0000000000400634 <+20>:	mov    0x20026d(%rip),%rax        # 0x6008a8 <const_progname>
   0x000000000040063b <+27>:	test   %rax,%rax
   0x000000000040063e <+30>:	je     0x4006b0 <main+144>
   0x0000000000400640 <+32>:	mov    %rax,0x10(%rsp)
```
+ 查看当前线程情况
```
xxxxxx@slave1:~$jstack /home/kewei/Downloads/work_soft/jdk1.8.0_121/bin/java core
Attaching to core core from executable /home/kewei/Downloads/work_soft/jdk1.8.0_121/bin/java, please wait...
Debugger attached successfully.
Client compiler detected.
JVM version is 25.121-b13
Deadlock Detection:

No deadlocks found.

Thread 9692: (state = BLOCKED)


Thread 9691: (state = BLOCKED)


Thread 9690: (state = BLOCKED)
 - java.lang.Object.wait(long) @bci=0 (Interpreted frame)
 - java.lang.ref.ReferenceQueue.remove(long) @bci=59, line=143 (Interpreted frame)
 - java.lang.ref.ReferenceQueue.remove() @bci=2, line=164 (Interpreted frame)
 - java.lang.ref.Finalizer$FinalizerThread.run() @bci=36, line=209 (Interpreted frame)
......
```
注意java命令的路径不能省略, 否则会报如下异常:
```
Attaching to core core from executable java, please wait...
Error attaching to core file: cannot open binary file
sun.jvm.hotspot.debugger.DebuggerException: cannot open binary file
	at sun.jvm.hotspot.debugger.linux.LinuxDebuggerLocal.attach0(Native Method)
```
+ 查看当前内存使用
```
xxxxx@slave1:~$ jmap /home/kewei/Downloads/work_soft/jdk1.8.0_121/bin/java core
Attaching to core core from executable /home/kewei/Downloads/work_soft/jdk1.8.0_121/bin/java, please wait...
Debugger attached successfully.
Client compiler detected.
JVM version is 25.121-b13
0x08048000	5K	/home/kewei/Downloads/work_soft/jdk1.8.0_121/bin/java
0xb7713000	127K	/lib/i386-linux-gnu/libpthread.so.0
0xb76fd000	93K	/home/kewei/Downloads/work_soft/jdk1.8.0_121/bin/../lib/i386/jli/libjli.so
0xb76f8000	13K	/lib/i386-linux-gnu/libdl.so.2
0xb7549000	1717K	/lib/i386-linux-gnu/libc.so.6
0xb7745000	131K	/lib/ld-linux.so.2
0xb6aa4000	8162K	/home/kewei/Downloads/work_soft/jdk1.8.0_121/jre/lib/i386/client/libjvm.so
0xb6a5e000	273K	/lib/i386-linux-gnu/libm.so.6
0xb6a04000	29K	/lib/i386-linux-gnu/librt.so.1
0xb7736000	54K	/home/kewei/Downloads/work_soft/jdk1.8.0_121/jre/lib/i386/libverify.so
0xb68db000	187K	/home/kewei/Downloads/work_soft/jdk1.8.0_121/jre/lib/i386/libjava.so
0xb68bf000	29K	/lib/i386-linux-gnu/libnss_compat.so.2
0xb68a6000	89K	/lib/i386-linux-gnu/libnsl.so.1
0xb689a000	41K	/lib/i386-linux-gnu/libnss_nis.so.2
0xb688d000	45K	/lib/i386-linux-gnu/libnss_files.so.2
0xb6873000	114K	/home/kewei/Downloads/work_soft/jdk1.8.0_121/jre/lib/i386/libzip.so
```

+ 通过core文件产生heapdump文件
`xxxxx@slave1:~$ jmap -dump:format=b,file=dump.hprof $JAVA_HOME/bin/java core`

# dump内存区域


# 参考
https://blog.csdn.net/zdy0_2004/article/details/80102076
https://www.cnblogs.com/xuxm2007/archive/2011/04/01/2002162.html