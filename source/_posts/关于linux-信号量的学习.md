---
title: 关于linux 信号量的学习
date: 2019-09-01 19:29:45
tags:
toc: true
categories: Linux网络编程
---
我们可能经常会使用kill -9 pid 来kill某个进程, 其实-9就是某一种信号量, 今天我们将简单讲解下信号量的使用。 linux中信号就是软件中断, 它提供了一种处理异步时间的方法: 终端用户键入中断键, 则会通过信号机构停止一个程序。产生信号的事件对于进程而言则是随机出现的。 进程表態只测试一个变量来判别是否发生了一个信号, 而是必须告诉内核"在此信号发生时, 请执行以下操作", 可以要求系统在某个信号按照以下三种的其中一种方式操作:
(1) 忽略信号, 大多数信号都可使用这种方式处理, 但是有两只中信号决不能被忽略, 它们是SIGKILL和SIGSTOP。这两种信号不能被忽略的原因是: 它们向超级用户提供一种使进程终止或者停止的可靠方法。另外, 如果忽略某种由硬件异常产生的信号(例如非法存储访问或者除以0), 则金恒的行为是未定义的。
(2) 捕获信号, 为了做到这一点要通知内核在某种信号发生时, 调用一个用户函数。在用户函数中, 可执行用户希望对这种事件进行的处理, 例如, 若编写一个命令解释器, 当用户用键盘产生中断信号后, 很可能希望用户返回到程序的主循环, 终止系统正在为用户执行的命令。 若果捕获到SIGCHILD信号, 则表示子进程已经终止, 所以此信号的捕获可以调用waitpid以取得该子进程的进程ID已经它的最终状态。 又例如, 如果进程创建了临时文件, 那么可能要为SIGTERM信号编写一个信号捕获函数以清理临时文件(kill命令传送的信号默认信号时终止信号)。
(3) 执行系统默认动作。对于大部分信号的系统默认动作是终止该进程。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/signal_1.png" height="450" width="500"/>
<img src="https://kkewwei.github.io/elasticsearch_learning/img/signal_2.png" height="250" width="480"/>
想了解每个信号的含义, 可以参考《Unix环境高级编程》一书, 每种信号可以由一个数字表示, 这里罗列一些常见的:
```
     Some of the more commonly used signals:
     1       HUP (hang up)
     2       INT (interrupt)
     3       QUIT (quit)
     6       ABRT (abort, 调用abort时触发)
     9       KILL (non-catchable, non-ignorable kill)
     14      ALRM (alarm clock)
     15      TERM (software termination signal)
     30      SIGUSR1 (user defined signal 1)
     31      SIGUSR2 (user defined signal 2)
```
在系统默认动作列中, "终止w/core"表示在进程当前工作目录的core文件中复制了该进程的存储图像(该文件名为core)。 在以下条件下不产生core文件:
(a) 程是设置-用户-ID, 而且当前用户并非程序文件的所有者
(b) 进程是设置-组-ID ,而且当前用户并非该程序文件的组所有者
(c)用户没有写当前工作目录的许可权
(d) 文件太大。我们经常使用`ulimit -c unlimited`设置core文件大小。

# sinal函数和sigaction函数
我们想自定义某些信号的处理函数, 那就需要用到sinal函数和sigaction函数, 还有别的常见的修改指定信号相关联操作的函数, 再次不罗列了。
我们直接使用示例进行说明吧:
```
#include <stdio.h>
#include <unistd.h>
#include <signal.h>
#include <errno.h>
#include <pthread.h>
void *thread_function(void *arg) {
    printf("My PID is %ld\n", (long)pthread_self());
    signal(SIGUSR1, sig_usr);
    // 读数被卡
    char buf[511];
    read(STDIN_FILENO, buf, 511);
    return NULL;
}
int main(void) {
    pthread_t mythread;
      if (pthread_create( &mythread, NULL, thread_function, NULL) ) {
        printf("error creating thread.");
    }
    sleep(1);
    // kill子线程, 向子线程发送SIGUSR1信号
    pthread_kill(mythread, SIGUSR1);
    printf("My PID is %d\n", getpid());
    for(;;) {
        pause();
    }
    return 0;
}

// 信号处理函数
static void sig_usr(int signum) {
    if(signum == SIGUSR1) {
        printf("SIGUSR1 received\n");
    }
    else if(signum == SIGUSR2) {
        printf("SIGUSR2 received\n");
    }
    else
        printf("signal %d received\n", signum);
    }
}
int main1(void) {
    char buf[512];
    int  n;

    struct sigaction sa_usr;
    sa_usr.sa_flags = 0;
    sa_usr.sa_handler = sig_usr;
    //自定义信号处理函数, sa_usr里面存放了信号处理函数
    sigaction(SIGUSR1, &sa_usr, NULL);
    sigaction(SIGHUP, &sa_usr, NULL);
    sigaction(SIGQUIT, &sa_usr, NULL);
    //自定义信号处理函数, 信号处理函数可以直接放在外面
    signal(SIGUSR2, sig_usr);
    printf("My PID is %d\n", getpid());
    while(1) {
        // 读数据被阻塞
        if((n = read(STDIN_FILENO, buf, 511)) == -1) {
            // 返回值代表read被中断
            if(errno == EINTR) {
                printf("read is interrupted by signal\n");
            }
        }
        else {
            buf[n] = '\0';
            printf("%d bytes read: %s\n", n, buf);
        }
    }
    return 0;
}
```
1. 针对main1()运行时, 将会打印当前pid。 然后我们可以在终端环境下输入`kill -SIGUSR1 pid` `kill -SIGUSR1 pid`, 可以看到进程接收到信号量后, 交由自定义函数处理的。
2. 针对main()运行时, 我们可以通过`pthread_kill`显示对某个线程发送信号量。
sinal函数和sigaction函数使用都比较简单, sigaction会代替早期实现的sinal。

参考:
《Unix环境高级编程》第十章 信 号