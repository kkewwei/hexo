---
title: JAVA NIO原理1
date: 2017-04-10 10:54:35
tags:
toc: true
---
JAVA NIO作为Java网络通信模块最基本的单元, 而在实践中, 我们都会接触到网络通信, 搞懂JAVA NIO, 对我们排查问题起到事半功倍的效果。 本文将带你进入JAVA NIO函数及底层本地函数的世界。
# JAVA NIO的基本使用
我们仍然以一个网上的典型示例入手:
## java版实现
```
import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.SocketOptions;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;
public class NIOServer {
    static Selector selector;  // 选择器
    public static void main(String[] args) throws Exception
    {
        //打开一个通道
        ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
        serverSocketChannel.socket().setSoTimeout(10000);
        serverSocketChannel.socket().setReuseAddress(true);
        //通道设置非阻塞
        serverSocketChannel.configureBlocking(false);
        //绑定端口号, 100表示该端口可以支持的最大连接数
        serverSocketChannel.bind(new InetSocketAddress("localhost", 8080, 100));
        //注册
        selector = Selector.open(); // KQueueSelectorImpl
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        while (true)
        {
            selector.select();
            Iterator<SelectionKey> ite = selector.selectedKeys().iterator();
            while (ite.hasNext())
            {
                SelectionKey key = ite.next();
                if (key.isAcceptable())
                {
                    SocketChannel channel = serverSocketChannel.accept();
                    //SocketChannelImp，返回的是一个已完成三次三次握手的链接
                    channel.configureBlocking(false);
                    channel.register(selector, SelectionKey.OP_READ);
                }
                else if (key.isReadable())
                {
                    SocketChannel channel = (SocketChannel) key.channel();
                    ByteBuffer buffer = ByteBuffer.allocate(256);
                    int i = channel.read(buffer);
                    if (i != -1)
                    {
                        String msg = new String(buffer.array()).trim();
                        System.out.println("NIO server received message =  " + msg);
                        channel.write(ByteBuffer.wrap( msg.getBytes()));
                    } else
                    {
                        channel.close();
                    }
                }
                ite.remove();
            }
        }
    }
}
```
客户端代码如下:
```
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;
public class NioClient
{
    /**
     * 通道
     */
    SocketChannel channel;
    public void initClient(String host, int port) throws IOException
    {
        //构造socket连接
        InetSocketAddress servAddr = new InetSocketAddress(host, port);
        //打开连接
        this.channel = SocketChannel.open(servAddr);
    }
    public void sendAndRecv(String words) throws IOException
    {
        byte[] msg = new String(words).getBytes();
        ByteBuffer buffer = ByteBuffer.wrap(msg);
        System.out.println("Client sending: " + words);
        channel.write(buffer);
        buffer.clear();
        channel.read(buffer);
        System.out.println("Client received: " + new String(buffer.array()).trim());
        channel.close();
    }
    public static void main(String[] args) throws IOException
    {
        NioClient client = new NioClient();
        client.initClient("localhost", 8080);
        client.sendAndRecv("I am a client");
    }
}
```
其实Java NIO封装了linux网络通信的接口, java 通过调用本地函数间接调用linux网络通信接口, 我们也给出linux是如何实现通信的。
## C++实现
Server 实现如下:
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
    serv_addr.sin_addr.s_addr = inet_addr("127.0.0.1");  //具体的IP地址
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
    if (clnt_sock < 0) { // 有异常退出
        if (errno == EAGAIN)
            return -10;
        if (errno == EINTR)
            return -9;
        return -8;
    }
    // Get client IP:Port and Server IP:Port.
    // struct sockaddr_in c, s;
    // socklen_t cLen = sizeof(c);
    // socklen_t sLen = sizeof(s);
    // getsockname(clnt_sock, (struct sockaddr*) &s, &sLen); // ! use clnt_sock here.
    // getpeername(clnt_sock, (struct sockaddr*) &c, &cLen); // ! use clnt_sock here.
    // printf("Client: %s:%d\nServer: %s:%d\n", inet_ntoa(c.sin_addr), ntohs(c.sin_port), inet_ntoa(s.sin_addr), ntohs(s.sin_port));
    //向客户端发送数据
    char str[] = "http://c.biancheng.net/socket/";
    int j = write(clnt_sock, str, sizeof(str));
    //关闭套接字
    close(clnt_sock);
    close(serv_sock);
    printf("connetc over");
    return 0;
}
```
Client 实现如下:
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
    close(sock);
    return 0;
}
```
建立通信过程是一样的, 大致如下:
<img src="https://kkewwei.github.io/elasticsearch_learning/img/nio.png" height="300" width="400"/>, 本文将围绕server端每一个步骤结合java本地实现进行详细介绍。

# 产生IO复用选择器
服务器端第一步就是调用ServerSocketChannel.open()函数, 首先产生IO复路选择器提供者:
```
    public static SelectorProvider provider() {
        synchronized (lock) {
            if (provider != null)
                return provider; // 确保只有一个被初始化
            return AccessController.doPrivileged(
                new PrivilegedAction<SelectorProvider>() {
                    public SelectorProvider run() {
                            // 从参数中查找是否有指定的
                            if (loadProviderFromProperty())
                                return provider;
                            // 从jar中的目录META-INF/services配置文件中找java.nio.channels.spi.SelectorProvider=class指定的类
                            if (loadProviderAsService())
                                return provider;
                            // 那么就是用默认的实现KQueueSelectorProvider
                            provider = sun.nio.ch.DefaultSelectorProvider.create();
                            return provider;
                        }
                    });
        }
    }
```
选择这个provider过程如下:
1. 查找是否通过参数制定了那个provider, 可以通过-Djava.nio.channels.spi.SelectorProvider制定
2. 继续在jar中的目录META-INF/services中查找
3.使用默认实现KQueueSelectorProvider, 之后会产生KQueueSelectorImpl类, 作为复路选择器的实现类, 之后会详细讲解。

# 设置非阻塞
套接字默认为阻塞的, 当发出一个不能完成的套接字调用时, 进程将阻塞, 可能阻塞的调用分为以下四个方面(参考《UNIX网络编程卷1: 套接字联网API》:
1. 输入操作, 比如read, readv, recv, recvfrom, recvmsg5个函数, 对一个TCP套接字调用这些函数时, 而且该套接字的接收缓存区没有数据可读取, 该线程将睡眠, 直到有一些数据到达, 既然TCP是字节流协议, 该进程的唤醒就是只要有一些数据到达, 这些数据既可能是单个字节, 也可能是一个完整地TCP分节中的数据, 如果想要等到某个固定树木的数据可读为止, 那么可以调用readn函数, 或者指定MSG_WAITALL标志。<br>
&#160; &#160; &#160; &#160;既然UDP是数据报协议, 如果一个阻塞的UDP套接字为空, 对它调用的进程将被投入睡眠, 直到有UDP数据报到达。
&#160; &#160; &#160; &#160;对于非阻塞的套接字, 如果一个操作不能被满足(对于TCP套接字至少有一个字节可读, 对于UDP至少有一个完整地数据报可读))。
2. 输出操作, 包括write, writev, send, sendto和sendmsg共5个函数, 对于一个TCP套接字,内核将从应用程序的缓冲区到该套接字的发送缓冲区复制数据。对于阻塞的套接字, 如果其发送缓冲区没有空间, 该进程将被阻塞, 直到有空间为止。
&#160; &#160; &#160; &#160;对于非阻塞TCP套接字, 如果其发送缓冲区根本没有空间, 输出函数调用将立即返回一个EWOULDBLOCK错误, 如果其发送缓冲区有一些空间, 返回值将是内核能够㢟到该缓冲区中的字节数, 这个字节数也成为不足计数。
&#160; &#160; &#160; &#160;UDP套接字不存在真正的发送缓冲区, 内核只是复制应用程序进程数据,并把它沿协议栈向下传递, 渐次冠以UDP首部和IP首部, 因此对于一个阻塞的UDP套接字, 输出函数将不会因为与TCP套接字一样的原因而阻塞, 不顾有可能因为其他的原因阻塞。
3. 接受外来连接(服务器端), 即accept函数, 如果对一个阻塞的套接字调用accept函数, 并且尚无新的连接到达, 该线程将阻塞。
&#160; &#160; &#160; &#160;如果对一个非阻塞的套接字调用accpet函数, 并且尚无新的连接到大, accept将立即返回一个AWOULDBLOCK错误。
4. 发起外来连接(客户端), 即调用TCP的connect函数。(connect同样可用于UDP, 不过它不能使一个真正的连接建立起来, 它只是是内核保存对端的IP和端口号)。TCP连接的建立涉及一个三次握手过程, 而且connect函数一直要等到客户端收到对于自己的SYN的ACK为止才返回。 这意味着TCP的每一个connect总会阻塞其调用进程至少一个到服务器的RTT时间。
在代码里面, 我们可以通过`channel.configureBlocking(false)`设置管道的非阻塞。在JAVA NIO中, 若我们对ServerSocketChannelImp和SocketChannelImp不设置非阻塞, 那么程序将检查不通过。

# open
有了复路选择器提供者之后, 就该产生ServerSocketChannelImpl了, 任何与client建立建立都是由它完成的, 我们看下初始化做了什么工作
```
    ServerSocketChannelImpl(SelectorProvider sp) throws IOException {
        super(sp);
        this.fd =  Net.serverSocket(true);
        this.fdVal = IOUtil.fdVal(fd);
        this.state = ST_INUSE;  // 只有三个状态，未使用，正使用，被kill
    }
    static FileDescriptor serverSocket(boolean var0) {
        return IOUtil.newFD(socket0(isIPv6Available(), var0, true, fastLoopback));
    }
    public static FileDescriptor newFD(int i) {
        FileDescriptor fd = new FileDescriptor();
        setfdVal(fd, i);
        return fd;
    }
```
可以看到初始化主要做了如下工作:
1. 通过调用java nio Java_sun_nio_ch_Net_socket0 获取文件描述符fd(int类型)
2. 产生FileDescriptor对象fd, 然后调用nio函数setfdVal, 对这个赋值, 将数字类型的fd放入FileDescriptor的类中。
3. 通过nio IOUtil.fdVal获取FileDescriptor中的fd属性值, 保存在ServerSocketChannelImpl的fdVal中
4. 设置ServerSocketChannelImpl的状态为ST_INUSE 正使用状态。
FileDescriptor里面的fd作为java与底层c++沟通的桥梁。 从FileDescriptor中获取fd ,也是通过java nio饶了一圈, 而不是直接从java FileDescriptor, 原因是FileDescriptor fd属性被设置成了private(保证fd的安全性)
我们接下来看java nio Java_sun_nio_ch_Net_socket0是如何实现的:
```
// 网络协议相关参数https://linux.die.net/man/2/socket
JNIEXPORT int JNICALL
Java_sun_nio_ch_Net_socket0(JNIEnv *env, jclass cl, jboolean preferIPv6,
                            jboolean stream, jboolean reuse)
{
    int fd;
     //通信协议, 默认都是SOCK_STREAM
    int type = (stream ? SOCK_STREAM : SOCK_DGRAM);
#ifdef AF_INET6
    int domain = (ipv6_available() && preferIPv6) ? AF_INET6 : AF_INET; //前面是ipv6, 后面代表ipv4
#else
    # AF_INET代表协议是ipv4, 默认都是ipv4
    int domain = AF_INET;
#endif
    // 第三个参数protocol执行通信用的协议, 0意味着使用默认的, 获得网络通信的fd
    fd = socket(domain, type, 0);
    // 它返回的就是socket file descriptor
    if (fd < 0) {
        return handleSocketError(env, errno);
    }
    ......

    if (reuse) { // 如果是地址复用地址, 那么就设置参数
        int arg = 1;
        if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, (char*)&arg,
                       sizeof(arg)) < 0) {
            JNU_ThrowByNameWithLastError(env,
                                         JNU_JAVANETPKG "SocketException",
                                         "Unable to set SO_REUSEADDR");
            close(fd);
            return -1;
        }
    }

......
    // 默认返回的是fd, 若出错, 那么返回值 < 0
    return fd;
}
```
可以看到本地函数实现也是比较简单的。此时我们获得了ServerSocketChannelImpl, 里面都linux socket通信的fd, 保存在了FileDescriptor中

# bind和listen
在java中, ServerSocketChannelImpl.bin()操作实际上调用了本地函数bind和listen, 我们看下具体做了哪些事情。
```
    public ServerSocketChannel bind(SocketAddress local, int backlog) throws IOException {
        synchronized (lock) {
            //首先检查ServerSocketChannelImpl是否被关闭
            if (!isOpen())
                throw new ClosedChannelException();
            // 只要localAddress不为null, 就算被绑定了
            if (isBound())
                throw new AlreadyBoundException();
            InetSocketAddress isa = (local == null) ? new InetSocketAddress(0) :
                Net.checkAddress(local);
            SecurityManager sm = System.getSecurityManager();
            if (sm != null)
                sm.checkListen(isa.getPort());
            NetHooks.beforeTcpBind(fd, isa.getAddress(), isa.getPort());
            Net.bind(fd, isa.getAddress(), isa.getPort());
            Net.listen(fd, backlog < 1 ? 50 : backlog);
            synchronized (stateLock) { //这对将地址绑定到端口上
                localAddress = Net.localAddress(fd); // 从底层引擎层获取服务器地址与主机，顺便检测是否一定开启了监听
            } // 可以看下http://www.samirchen.com/get-client-server-ip-port/
        }// 在bind以后就可以调用getsockname来获取IP和Port，虽然这没有什么太多的意义
        return this;
    }
```
可以看到, 该函数主要就是调用了本地函数Net.bind() 和Net.listen()函数, 其中前者用户端口绑定, 后者监听该端口。我们分别看下两个本地函数做了什么:
## Net.bind
Net.bind调用jvm的
```
JNIEXPORT void JNICALL // fdo 文件对象java/io/FileDescriptor   iao：地址localhost/127.0.0.1
Java_sun_nio_ch_Net_bind0(JNIEnv *env, jclass clazz, jobject fdo, jboolean preferIPv6,
                          jboolean useExclBind, jobject iao, int port)
{
    SOCKADDR sa;
    int sa_len = SOCKADDR_LEN;
    int rv = 0;
    //将java的InetAddress转换为c的struct socketAddr
    if (NET_InetAddressToSockaddr(env, iao, port, (struct sockaddr *)&sa, &sa_len, preferIPv6) != 0) {
      return;
    }
   //调用bind方法:int bind(int sockfd, struct sockaddr* addr, socklen_t addrlen)套接字是用户程序与内核交互信息的枢纽，它自身没有网络协议地址和端口号等信息,在进行网络通信的时候，必须把一个套接字与一个地址相关联。很多时候内核会我们自动绑定一个地址，然而有时用户可能需要自己来完成这个绑定的过程，以满足实际应用的需要；
   //最典型的情况是一个服务器进程需要绑定一个众所周知的地址或端口以等待客户来连接。对于客户端，很多时候并不需要调用bind方法，而是由内核自动绑定；性能测试的时候为了在一台机器上发起海量连接端口的限制，会在一个机器上配置多个ip地址，建立连接时，绑定ip
    rv = NET_Bind(fdval(env, fdo), (struct sockaddr *)&sa, sa_len);
    if (rv != 0) { //绑定失败
        handleSocketError(env, errno);
    }
}
```
可以看到, Java_sun_nio_ch_Net_bind0首先将InetAddress转变为linux支持的sockaddr地址, 其次获取FileDescriptor中保存的fd, 最后通过bind(fd)绑定本地端口

## Net.listen()
```
JNIEXPORT void JNICALL
Java_sun_nio_ch_Net_listen(JNIEnv *env, jclass cl, jobject fdo, jint backlog)
{  //首先获取文件描述符兑现的fd值
    if (listen(fdval(env, fdo), backlog) < 0)
        handleSocketError(env, errno);
}
```
而在listen本地函数中, 我们需要了解一个参数backlog, 表示监听该端口, 可支持的最大的连接数, 在netty的NetUtil.java中, 会去读取服务器端/proc/sys/net/core/somaxconn文件该值。然后再去调用linux的listen函数去监听该端口

# 产生Selector
Select主要处理IO复用, 能够监听多个多个链路的IO事件, 目前针对Java服务器的非阻塞编程基本都是基于epoll的：EPollSelectorImpl， mac上测试时是KQueueSelectorImpl, 我们就以后者为例讲解。
我们看下KQueueSelectorImpl成员变量:
```
class KQueueSelectorImpl
    extends SelectorImpl
{
    // File descriptors used for interrupt
    //将从select中唤醒使用的
    protected int fd0;
    protected int fd1;

    // The kqueue manipulator
    // KQueueSelectorImpl是上层逻辑，真正偏底层交道的还是KQueueArrayWrapper(比如封装32/64位),
    KQueueArrayWrapper kqueueWrapper;

    // Count of registered descriptors (including interrupt)
    // 多少注册的描述符
    private int totalChannels;

    // Map from a file descriptor to an entry containing the selection key
    // 存放的是管道FD对应着SelectionKey.OP_ACCEPT 产生的SelectionKeyImpl
    private HashMap<Integer,MapEntry> fdMap;

    // True if this Selector has been closed
    private boolean closed = false;

    // Lock for interrupt triggering and clearing
    private Object interruptLock = new Object();
    // 中断已触发标志
    private boolean interruptTriggered = false;

    // used by updateSelectedKeys to handle cases where the same file
    // descriptor is polled by more than one filter
    private long updateCount;

    // Used to map file descriptors to a selection key and "update count"
    // (see updateSelectedKeys for usage).
    private static class MapEntry {
        SelectionKeyImpl ski;
        long updateCount;
        MapEntry(SelectionKeyImpl ski) {
            this.ski = ski;
        }
    }

    /**
     * Package private constructor called by factory method in
     * the abstract superclass Selector.
     */
    KQueueSelectorImpl(SelectorProvider sp) {
        super(sp);  //IOUtil.makePipe作用可参考：https://cloud.tencent.com/developer/article/1007497
         // 开启管道，每个进程各自有不同的用户地址空间，任何一个进程的全局变量在另一个进程中都看不到，所以进程之间要交换数据必须通过内核，在内核中开辟一块缓冲区，进程1把数据从用户空间拷到内核缓冲区，进程2再从内核缓冲区把数据读走，内核提供的这种机制称为进程间通信（IPC，InterProcess Communication）
        long fds = IOUtil.makePipe(false);
        //读端
        fd0 = (int)(fds >>> 32);
         //写段
        fd1 = (int)fds;
        kqueueWrapper = new KQueueArrayWrapper();
        kqueueWrapper.initInterrupt(fd0, fd1);
        fdMap = new HashMap<>();
        totalChannels = 1;
    }
```
+ IO多路复用KQueueArrayWrapper
我们首先看IO多路复用底层实现的KQueueArrayWrapper的工作原理:
```
   static {
        IOUtil.load();  //  啥都不做
        initStructSizes();  // 会在native中对字段进行初始化
        String datamodel = java.security.AccessController.doPrivileged(
            new sun.security.action.GetPropertyAction("sun.arch.data.model")
        );
        is64bit = datamodel.equals("64");
    }
    KQueueArrayWrapper() {
        int allocationSize = SIZEOF_KEVENT * NUM_KEVENTS;
        keventArray = new AllocatedNativeObject(allocationSize, true);
        // keventArrayAddress保存着该内存块物理地址
        keventArrayAddress = keventArray.address();
        // Java_sun_nio_ch_KQueueArrayWrapper_init  这里回去通过kqueue()初始化一个句柄, 获得IO复路监听者
        kq = init();
    }
```
在linux层面, 每个被监听/响应的事件都是由一个kevent构成, 构成结构如下:
```
struct kevent {
	uintptr_t	ident;		/* identifier for this event  比如该事件关联的文件描述符*/*/
	int16_t		filter;		/* filter for event   可以指定监听类型，如EVFILT_READ，EVFILT_WRITE，EVFILT_TIMER等 */
	uint16_t	flags;		/* general flags 可以指定事件操作类型，比如EV_ADD，EV_ENABLE， EV_DELETE等 */
	uint32_t	fflags;		/* filter-specific flags */
	intptr_t	data;		/* filter-specific data */
	void		*udata;		/* opaque user data identifier 可以携带的任意数据地址 */
};
```
可以看出这里只支持NUM_KEVENTS(windows上200个, mac上128个)个事件, 响应直接存放在AllocatedNativeObject中, 内部通过`unsafe.allocateMemory(size + ps)`方式直接分配内存及管理, 然后通过initStructSizes对AllocatedNativeObject内存块尽心管理, 比如要获取监听的第i个事件的文件描述符:
```
    int getDescriptor(int index) {
        int offset = SIZEOF_KEVENT*index + FD_OFFSET;
        /* The ident field is 8 bytes in 64-bit world, however the API wants us
         * to return an int. Hence read the 8 bytes but return as an int.
         */
        if (is64bit) {
          long fd = keventArray.getLong(offset);  // 返回监听列表的第几个监听对象
          assert fd <= Integer.MAX_VALUE;
          return (int) fd;
        } else {
          return keventArray.getInt(offset);
        }
    }
```
首先获取AllocatedNativeObject Array中第index个事件的相对位置, 然后再通过调用keventArray.getInt(offset)本地函数直接获取该内存地址的值。
我们也需要知道`kq = init();`, 它作为IO复用, java与linux关联起来的核心fd, 该函数会去调用底层`int kq = kqueue()`, 我们先对多路复用进行一个简单的描述吧:
`kueue是在UNIX上比较高效的IO复用技术。所谓的IO复用，就是同时等待多个文件描述符就绪，以系统调用的形式提供。如果所有文件描述符都没有就绪的话，该系统调用阻塞，否则调用返回，允许用户进行后续的操作。常见的IO复用技术有select, poll, epoll以及kqueue等等。其中epoll为Linux独占，而kqueue则在许多UNIX系统上存在，包括OS X（好吧，现在叫macOS了。。）`(<a href="https://www.cppentry.com/bencandy.php?fid=104&id=138645">参考</a>)
简单来说, 我们可以向kqueue注册多个感兴趣的事件, 比如读写。然后调用kevent()等待, 当输入流/输出流就绪时, 就会返回。 这里我们还要理解的地点就是, 只要一次注册, kqueue()每次返回时, 都对检查之前注册的事件。 除非使用如下命令显示取消对事件的继续关注:
```
    EV_SET(&changes[1], fd, EVFILT_WRITE, EV_DELETE, 0, 0, 0);

```

+  fd0和fd1
我们接下来看下属性fd0和fd1, 调用`IOUtil.makePipe(false);`产生, 实际调用了linux pipe()来完成的。
```
JNIEXPORT jlong JNICALL   // blocking传递过来为false
Java_sun_nio_ch_IOUtil_makePipe(JNIEnv *env, jobject this, jboolean blocking)
{
    int fd[2];
    // 可参考：https://cloud.tencent.com/developer/article/1007497
    if (pipe(fd) < 0) {
        JNU_ThrowIOExceptionWithLastError(env, "Pipe failed");
        return 0;
    }
    if (blocking == JNI_FALSE) {
        if ((configureBlocking(fd[0], JNI_FALSE) < 0)
            || (configureBlocking(fd[1], JNI_FALSE) < 0)) {
            JNU_ThrowIOExceptionWithLastError(env, "Configure blocking failed");
            close(fd[0]);
            close(fd[1]);
            return 0;
        }
    }
    return ((jlong) fd[0] << 32) | (jlong) fd[1];
}
```
我们调用linux的pipe(fd)函数时, 会在内核中开辟一块缓冲区（称为管道）用于通信，它有一个读端一个写端，然后通过fd参数传出给用户程序两个文件描述符，fd[0]指向管道的读端，fd[1]指向管道的写端（很好记，就像0是标准输入1是标准输出一样）。所以管道在用户程序看起来就像一个打开的文件，通过read(fd[0]);或者write(fd[1]);向这个文件读写数据其实是在读写内核缓冲区。pipe函数调用成功返回0，调用失败返回-1。为啥要生成fd[2]呢, 是为了调用selector.wakeup()将程序从selector.select()中唤醒, 我们接下来看下`kqueueWrapper.initInterrupt(fd0, fd1)`实现就更清楚了。
```
    void initInterrupt(int fd0, int fd1) {
        outgoingInterruptFD = fd1;
        incomingInterruptFD = fd0;
         // 这里还去注册一下， 主要是为了别的线程使用wakeup唤醒该线程，针对fd0注册一个读感兴趣
        register0(kq, fd0, 1, 0);
    }
```
知道了kqueue的原理, 我们就好理解唤醒原理了selector.wakeup()原理了, 只要有感兴趣的发生, 就会唤醒, 那我们不妨专门为唤醒产生一对fd, 将读端注册到kqueue中, 当需要唤醒时, 我们就仅仅向写端写一个字符就可以唤醒了。我们看下selector.wakeup是不是就这样实现的:
```
    public Selector wakeup() {
        synchronized (interruptLock) {
            if (!interruptTriggered) {  // 只能被唤醒一次
                kqueueWrapper.interrupt();
                interruptTriggered = true;
            }
        }
        return this;
    }
    void interrupt() {
        interrupt(outgoingInterruptFD);
    }
```
outgoingInterruptFD保存着写段的地址, 看下interrupt本地函数实现:
```
JNIEXPORT void JNICALL
Java_sun_nio_ch_KQueueArrayWrapper_interrupt(JNIEnv *env, jclass cls, jint fd)
{   //wakeup唤醒select的方法
    char c = 1;
    if (1 != write(fd, &c, 1)) {
        JNU_ThrowIOExceptionWithLastError(env, "KQueueArrayWrapper: interrupt failed");
    }
}
```
可以看到, 果然是向写端的fd中写入了一个字符。

# register
接着我们需要向selector注册感兴趣的事件SelectionKey.OP_ACCEPT了, 可以看下是怎么实现的:
```
    public final SelectionKey register(Selector sel, int ops,
                                       Object att)
        throws ClosedChannelException
    {
        synchronized (regLock) {
            if (!isOpen())
                throw new ClosedChannelException();
            if ((ops & ~validOps()) != 0)
                throw new IllegalArgumentException();
            if (blocking)
                throw new IllegalBlockingModeException();
            //
            SelectionKey k = findKey(sel);
            if (k != null) {
                k.interestOps(ops);
                k.attach(att);
            }
            if (k == null) {
                // New registration
                synchronized (keyLock) {
                    if (!isOpen())
                        throw new ClosedChannelException();
                    k = ((AbstractSelector)sel).register(this, ops, att);
                    addKey(k);
                }
            }
            return k;
        }
    }
```
register函数主要做了如下事情:
1. 首先检测ServerSocketChannelImpl并没有被关闭。
2. 检查ServerSocketChannelImpl是非阻塞的。
3. 检查目前是否该管道是否在selector上面已经注册了, 若注册了, 那么修改关注的事件。否则就调用register去KQueueSelectorImpl中注册这个事件。
这里可能有个疑问, 假如我们向kqueue()的套接字注册了OP_READ事件, 这里再次向其注册OP_WRITE事件, 那么此时对同一个管道向Selector注册了两个事件。而这里注册OP_WRITE时调用findKey找到了第一次注册的SelectionKey, 然后再覆盖感兴趣的事件, 不就把OP_READ注册信息给覆盖了吗? 答案是真的覆盖了, 并且在向kqueue注册第二个事件时会将第一个事件给取消了。先晒一段代码:
```
// 可以看到var2.events与Net.POLLIN和Net.POLLOUT相与, write或者read只能有一个为1。
this.register0(this.kq, var3.getFDVal(), var2.events & Net.POLLIN, var2.events & Net.POLLOUT);
```
```
JNIEXPORT void JNICALL
Java_sun_nio_ch_KQueueArrayWrapper_register0(JNIEnv *env, jobject this,
                                             jint kq, jint fd, jint r, jint w)
{
    struct kevent changes[2];
    struct kevent errors[2];
    struct timespec dontBlock = {0, 0};
    // 注册监听事件， 这里可以看出，要么只能是读，要么只能是写，只能选择一样。选择一样，那么另外一样只能取消关注。
    // if (r) then { register for read } else { unregister for read }
    // if (w) then { register for write } else { unregister for write }
    // Ignore errors - they're probably complaints about deleting non-
    //   added filters - but provide an error array anyway because
    //   kqueue behaves erratically if some of its registrations fail.
    // read或者write事件只能选择一个, 注册了一个, 那么另外一个将被取消。
    EV_SET(&changes[0], fd, EVFILT_READ,  r ? EV_ADD : EV_DELETE, 0, 0, 0);
    EV_SET(&changes[1], fd, EVFILT_WRITE, w ? EV_ADD : EV_DELETE, 0, 0, 0);
    kevent(kq, changes, 2, errors, 2, &dontBlock);
}
```
