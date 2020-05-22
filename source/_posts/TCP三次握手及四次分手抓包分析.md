---
title: TCP三次握手及四次分手抓包分析
date: 2018-04-01 13:01:32
tags:
toc: true
categories: Linux网络编程
---
对于TCP建立连接时的三次握手及断开连接时的四次分手, 相信我们大致原理都是很熟悉的, 具体介绍可以百度搜索, 今天我们将仅从抓包的角度实际验证时, 以更好地掌握这2个过程。
我们以如下代码来实验:
## Linux通信过程
server:
```
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/fcntl.h>
#include <netinet/in.h>
#include <errno.h>
int main(){
    //创建套接字
    int serv_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    //将套接字和IP、端口绑定
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));  //每个字节都用0填充
    serv_addr.sin_family = AF_INET;  //使用IPv4地址
    serv_addr.sin_addr.s_addr = inet_addr("0.0.0.0");  //具体的IP地址
    serv_addr.sin_port = htons(1234);  //端口
    bind(serv_sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    // 可执行各种描述符控制操作
    // int flags = fcntl(serv_sock, F_GETFL);
    // int newflags = (flags | O_NONBLOCK);
    // fcntl(serv_sock, F_SETFL, newflags);
    //进入监听状态，等待用户发起请求
    listen(serv_sock, 20);
    //接收客户端请求
    struct sockaddr_in clnt_addr;
    socklen_t clnt_addr_size = sizeof(clnt_addr);
    int clnt_sock = accept(serv_sock, (struct sockaddr*)&clnt_addr, &clnt_addr_size);
    char str[] = "http://c.biancheng.net/socket/";
    int j = write(clnt_sock, str, sizeof(str));
    sleep(6);
    close(clnt_sock);
    sleep(100);
    return 0;
}
```
client:
```
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
int main(){
    //创建套接字
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    //向服务器（特定的IP和端口）发起请求
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));  //每个字节都用0填充
    serv_addr.sin_family = AF_INET;  //使用IPv4地址
    serv_addr.sin_addr.s_addr = inet_addr("127.0.0.1");  //具体的IP地址
    serv_addr.sin_port = htons(1234);  //端口
    connect(sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    //读取服务器传回的数据
    char buffer[40];
    read(sock, buffer, sizeof(buffer)-1);
    socklen_t  sa_len = sizeof(serv_addr);
    printf("Message form server: %s\n", buffer);
    //关闭套接字
    sleep(5);
    close(sock);
    sleep(1000);
    return 0;
}
```
然后根据WireShark进行抓包, 抓到的包如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/tcp.png" height="200" width="900"/>
我们对抓包进行简单说明:
1.2.3分别是三次握手过程,4是服务器端向客户端发送数据,5是客户端对服务器端的数据进行确认,6,7,8,9分别是四次分手过程。结合到3次握手及4次分手过程如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/tcp_connect_close.png" height="600" width="400"/>
从这个过程中可以知道如下几点:
1. 客户端或者服务器每发送一条数据, 对端都会对这条数据进行确认, 确认时, ack=接受的sqe+len(data)。
2. 当客户端向服务器发送close之后, 底层网络就会向服务器发送[FIN, ACK]帧, 服务器端接收到这个帧后, 立马会向客户端返回ack。 这两个部分分别对应着分手过程的第一、二步骤, 也即是抓包的6、7行。
3. 客户端调用了close之后, 若服务器端不调用close, 那么服务器端将长时间处于FIN_WAIT2阶段, 除非超时2小时,才会关闭。
4. 服务器端调用了close之后, 经过验证, 此时客户端连接还处于CLOSE_WAIT, 等待2MSL(60s)超时彻底关闭, 也就是说client在CLOSE_WAIT会停留60s才彻底释放链接。
5.只有当客户端彻底处于close阶段, 才会释放套接字链接, 包括port。服务器也是同样的。
这里还需要明确的一点就是：服务器端主要主动调用close函数，也会进入time_wait阶段。

# JAVA NIO通信关闭管理
我们在nio通信时, 会通过调用SocketChannelImpl.close来关闭通信管道, 这种方式和linux C++版本的close(sock)还是有一些区别的, 实际关闭网络管道靠的是<a href="https://kkewwei.github.io/elasticsearch_learning/2017/04/10/Java-NIO-write-read%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%90/#begin">implCloseSelectableChannel</a>, 该函数做了如下事情:
1. 调用SocketDispatcher.preClose(fd)半关闭。
2. 若此时有读写函数的话, 那么通过调用NativeThread.signal唤醒这些线程的。
3. 若这个管道上没有被注册关注的事件, 那么就可以安全的关闭。
这里并没有直接通过close(sock)方式关闭, 客户端和服务器端主动关闭管道, 过程都是一样的, 这里将主要客户端关闭过程来说明, 我们将这段话转变为c++版本的过程如下:
```
int main(){
    ......
    char buffer[40];
    read(sock, buffer, sizeof(buffer)-1);
    sleep(3);
    socklen_t  sa_len = sizeof(serv_addr);
    printf("Message form server: %s\n", buffer);

    //下面是关闭过程
    int socket_pair[2];
    // 首先获得双工套接字
    socketpair(AF_UNIX, SOCK_STREAM, 0, socket_pair);
    // 关闭其中一边的套接字
    close(socket_pair[0]);
    sleep(3);
    // 将原写入sock套接字关闭, 并指向新的socket_pair[1]指向的文件表
    dup2(socket_pair[1], sock);
    // 关闭这个文件表
    close(sock);
    sleep(3);
    return 0;
}
```
可以看到, 其实在`dup2(socket_pair[1], sock);`中就将原始套接字关闭了, 达到了`close(sock)`的效果, 通过抓包也实际验证了`dup2`函数会向服务器端发送发送分手过程的第一阶段。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/tcp_connect_close1.png" height="200" width="900"/>
可以由抓包看到, 在第6s开始向server发送分手过程的第一阶段, 也就对应着`dup2`过程。

## close()之后的链路情况
假如client主动调用了close之后, 链路情况是如何的呢? 我们分两种情况:
1. client此时还能做啥?  此时系统将对该套接字设置为不可读, 任何client对该套接字的读取操作将抛出异常而崩溃退出, 并产生信号量SIGPIPE。 在写出的方向, 系统内核尝试将缓存区的数据发送给对方, 并向对端发送FIN信号, client进入FINA_WAIT_1阶段。之后client对该套接字的写操作将抛出异常而崩溃退出, 并产生信号量SIGPIPE。
2. 服务端若在client端关闭之后, 继续进行读写操作会发生什么事呢? 第一次读取时, client会返回RST响应, 告诉服务器端, 我已经关闭了, read/write返回值为0。 若服务器端续向这个套接字进行read/write, 那么读写返回值将抛出异常而崩溃退出, 并产生信号量SIGPIPE。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/tcp_connect_close4.png" height="250" width="180"/>
以上体现了整个包传输过程, 当client调用close后,若此后server再向client发送数据时, 客户端还可以接收到数据, 并返回一个RST包给服务器端。 若此后服务器再次发送数据, 服务器端将直接抛出异常, 并产生SIGPIPE信号量。下图抓包过程也可以体现这个过程。
 + 1-3是连接建立过程
 + 4-5是client发送数据及server ack过程。
 + 6-7是client调用close及server ack过程。
 + 8-9是server发送数据及client 回应RST过程。
 此后任何发送及read操作, 都将失败, 并产生SIGPIPE信号。
<img src="https://kkewwei.github.io/elasticsearch_learning/img/tcp_connect_close3.png" height="200" width="900"/>


