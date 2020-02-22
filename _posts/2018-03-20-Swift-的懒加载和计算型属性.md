---
layout:     post
title:      muduo阅读笔记之第6章
subtitle:   muduo网络库简介
date:       2018-03-20
author:     BY
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - muduo
---


muduo是静态库 分布式系统发布动态库成本很高
核心库之依赖TR1
muduo库，只需要掌握5个关键类，Buffer，EventLoop，TcpConnection，TcpClient，TcpServer
Buffer仿Netty ChannelBuffer的buffer class。用户不需要调用read/write
InetAddress不解析域名，gethostbyname解析域名会阻塞IO线程。
Socket是RAII handle，
Poller是Poller和EPollPoller基类，采用“电平触发”.(水平触发)
线程模型：TcpServer支持1、单线程（accept和TcpConnection同一个线程做IO）2、多线程（accept与Eventloop同一个线程，另外创建一个EventLoopThreadPool，新到的连接会按照round-robin分配到线程池中)
tcp网络编程本质论
转换思维模式：
        由“主动调用recv、accept、send的思路”转变成“注册回调函数，事件通知"。基于事件的非阻塞网络编程，是编写高性能并发网络服务程序的主流模式。（类似win32消息循环，避免使用阻塞调用，使得窗口失去响应。）

基于事件的非阻塞网络编程主要是处理三个半事件：

        1、连接建立事件（包括服务端客户端）。
        2、连接断开事件（包括主动断开和被动断开）。
        3、消息到达事件（处理分包、缓冲区设计等）。
        4、消息发送完毕的半个事件（低流量服务不关心，如何保证先发送完缓冲区的数据再断开连接，shutdown半关闭）。

基于事件的非阻塞网络编程存在的问题：

        1、如果要主动关闭，如何保证对方已经收到全部数据？
 	2、如果应用有缓存，如何保证先发送完缓冲中的数据，再断连接？
 	3、客户端如何定期重试？
        4、用边沿触发（edge trigger）还是电平触发（level trigger）。如果是电平触发，何时关注EPOLLOUT事件？会不会造成busy-loop？如果是边缘触发，如何便面漏读造成饥饿？epoll一定比poll快吗？
        5、为什么要使用应用层发送缓冲区？假设应用要发送40kb，但操作系统的tcp发送缓冲区只有25kb，那么剩下的15kb怎么办？tcp发送缓冲区小于发送的数据，不能等待os缓冲区可用（会阻塞），应用层应缓存起来，等socket可写，立即发送数据，若程序要发送新的数据前，有未发出的数据，应该追加得到缓冲区末尾，一并发出，防止打乱数据顺序。
        6、为什么要使用应用层接受缓冲区？如果一次收到的数据不够一个完整数据包，应缓存在应用层缓冲区里，等剩余数据收到后一并处理。如果数据分小段频繁到达，都触发文件描述符可读事件程序能否还能正常工作。
        7、如何设计并使用缓冲区？我们，减少系统调用，一次读取很多数。如果连接数很多，每个连接都分配各自的缓冲区，内存将很大，并且大多数时候这些缓冲区的使用率很低，怎么办？
        8、如何做应用层的流量控制？如果接受处理缓慢，数据会不会一直堆积在发送方，造成内存暴涨。
        9、如何实现定时器？并使之与网络io在一个线程，以避免锁。
python twisted是一款非常好的网络库，也采用reactor作为基本模型。
可用telnet扮演客户端测试程序。
性能评测
测试对象：

	1、boost 1.4.0中的asio 1.4.3
	2、asio 1.4.5（）
	3、libevent 2.0.6-rc
	4、muduo 0.0.1
方法：        

        1、[asio的测试代码](http://asio.cvs.sourceforge.net/viewvc/asio/asio/src/tests/performance)\[asio性能测试方法](http://think-async.com/Asio/LinuxPerformanceImprovements)
 	2、使用pingpong协议测试吞吐量的常用方法。pingpong协议：客户端是服务器都实现echo协议。源自asio性能测试。pingpong消息的大小均为16kb。muduo吞吐量大于libevent2的原因：libevent2一次最多从网络中读取4096字节，而moduo为16384，系统调用少。
        3、击鼓传花方法，1000个网络连接，从第一个连接写一个字节，从这个连接的另一头读出这个字节，再发给第2个连接，一次类推，记录开始到结束时间，比较谁更早结束。源自libevent的性能测试文档。
        4、章亦春的HTTP echo模块。
        5、[ZeroMQ自带的延迟和吞吐量测试](http://www.zeromq/results:perf-howtopingpong)翻版。


Parallel Pipelining的意义[《以小见大-那些基于protobuf的五花八门的RPC(2)》](http://blog.csdn.net/lanphaday/archive/2011/04/11/6316099.aspx)
[《分布式系统的工程化开发方法》](http://blog.csdn.net/Solstice/article/details/5950190)
[《谈谈数独》](http://blog.csdn.net/Solstice/article/details/2096209)
Sudoku 是一个计算密集型的任务（见7.4中关于性能的分析），其瓶颈在于cpu。

负载均衡（Load Balance）

常见的并发网络服务器程序设计方案
可伸缩性网络编程，POSA2已经进行相当全面的总结，还有另外三篇值得思考的
http://bulk.fefe.de/scalable-networking.pdf
http://www.kegel.com/c10k.html
http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf
在reactor的线程中，如果收到数据进行延迟操作，不应该适用sleep之类的阻塞调用，正确做法：应该注册超时回调。

	方案0：accept +read/write
            阻塞IO，一次服务一个客户。
	方案1：accept+fork
            process-per-connection。
	方案2：accept+thread
            thread-pe-connection。 
	方案3：prefork
	方案4：pre threaded
前四种都属于阻塞式网络编程。

	方案5：reactor

IO复用其实复用的不是IO连接，而是复用线程。使用select/poll几乎肯定要配合非阻塞io，而使用非阻塞io，肯定要使用应用层buffer，原因见7.4       
reactor的意义在于将消息（IO事件）分发到用户提供的处理函数，并保持网络部分通用代码不变，独立于用户的业务逻辑。
适合IO密集型的应用，不适合cpu密集型的应用。注意：避免在事件回调中执行耗时操作，阻塞线程。（与windows消息循环非常类似）
 	注意由于只有一个线程，因此事件是顺序处理的，一个线程同时只能做一件事情。事件的优先级不能保证，因为从poll返回之后到下一次调用poll进入等待之前这段时间内，线程不会被其他链接上的数据或事件抢占。如果我们想延迟计算（把compute()推迟100ms），那么也不能用sleep()之类的阻塞调用，而应该注册超时回调，以避免阻塞当前IO线程。
	
	方案6：reactor+thread-per-task
            创建线程有开销，可以用threadpool避免。计算有乱序情况，可以用协议中使用id区分。
	方案7：reactor+workthread
            一个计算线程方案。workthread，不能充分利用cpu。最多占用12.5%的cpu资源。不会乱序。
	方案8：reactor+threadpool
            如果IO的压力比较大，reactor处理不过来，使用方案9。
	方案9：reactors in threads（reactor pool）
            muduo内置的多线程方案，也是netty内置的多线程方案。有一个主的reactor，负责accept接收连接，然后把连接挂到某个sub reactor上（采用round-robin方式选择sub reactor），并在那个sub reactor中完成该连接的所有 操作。由于一个链接完全由一个线程管理，那么请求的顺序性可以保证。该方案解决方案8一个reactor出现处理饱和。优化突然请求，可考虑方案11.
	方案10：reactors in processes
            Nginx的内置方案。如果连接之间无交互，该方案是很好选择，可以热升级。
	方案11：reactors+threadpool
            方案8和方案9混合。适应突发IO处理（reactors）,又能适应突发计算（thread pool）。reactors的个数的设置：ZeroMQ给出建议（http://www.zeromq.org/area:faq#toc3），按照每千兆比特每秒的吞吐量配一个reactor的比例，设置个数。运行在千兆以太网上的网络程序，并且没有计算量（可忽略），并且连接没有优先级之分，用一个reactor就足以应付网络IO。如果tcp连接有优先级之分，正确的做法：把优先级高的连接单独用一个reactor来处理。

## 我的总结：
        1、基于事件的非阻塞网络编程，是编写高性能并发网络服务程序的主流模式。
        2、基于事件的非阻塞网络编程主要是处理三个半事件：（连接建立、连接断开、消息到达、消息发送完毕（半））
        3、使用pingpong协议测试吞吐量的常用方法。
        4、在reactor的线程中，如果收到数据进行延迟操作，不应该适用sleep之类的阻塞调用，正确做法：应该注册超时回调。
        5、最佳模型：reactors+thread pool，reactors解决突发IO，thread pool解决突发计算。
        6、按照每千兆比特每秒的吞吐量配一个reactor的比例，设置个数。
	7、一个socket只由一个线程来处理。

## 附录1：level trigger和edge trigger
### 概述 
   边缘触发 是指每当状态变化时发生一个io事件；
   条件触发 是只要满足条件就发生一个io事件；
### 详述 
                int select(int n, fd_set *rd_fds, fd_set *wr_fds, fd_set *ex_fds, struct timeval *timeout);
     select用到了fd_set结构,此处有一个FD_SETSIZE决定fd_set的容量，FD_SETSIZE默认1024，可以通过ulimit -n或者setrlimit函数修改之。
                int poll(struct pollfd *ufds, unsigned int nfds, int timeout);
     作为select的替代品，poll的参数用struct pollfd数组(第一个参数)来取代fd_set，数组大小自己定义，这样的话避免了FD_SETSIZE给程序带来的麻烦。
     每次的 select/poll操作，都需要建立当前线程的关心事件列表，并挂起此线程到等待队列中 直到事件触发或者timeout结束，同时select/poll返回后也需要对传入的句柄列表做一次扫描来dispatch。随着连接数增 加，select和poll的性能是严重非线性下降。
epoll(linux), kqueue(freebsd), /dev/poll(solaris):
作为针对select和poll的升级（可以这么理解:)）,主要它们做了两件事情

	1. 避免了每次调用select/poll时kernel分析参数建立事件等待结构的开销，kernel维护一个长期的事件关注列表，应用程序通过句柄修改这个列表和捕获I/O事件。
	2. 避免了select/poll返回后，应用程序扫描整个句柄表的开销，Kernel直接返回具体的事件列表给应用程序。
 
同时还有两种触发机制：
水平触发(level-triggered，也被称为条件触发)LT: 只要满足条件，就触发一个事件(只要有数据没有被获取，内核就不断通知你)
边缘触发(edge-triggered)ET: 每当状态变化时，触发一个事件
     “举个读socket的例子，假定经过长时间的沉默后，现在来了100个字节，这时无论边缘触发和条件触发都会产生一个read ready notification通知应用程序可读。应用程序读了50个字节，然后重新调用api等待io事件。这时水平触发的api会因为还有50个字节可读从 而立即返回用户一个read ready notification。而边缘触发的api会因为可读这个状态没有发生变化而陷入长期等待。 因此在使用边缘触发的api时，要注意每次都要读到socket返回EWOULDBLOCK为止，否则这个socket就算废了。而使用条件触发的api 时，如果应用程序不需要写就不要关注socket可写的事件，否则就会无限次的立即返回一个write ready notification。大家常用的select就是属于水平触发这一类，长期关注socket写事件会出现CPU 100%的毛病。
 
### epoll的优点： 
1.支持一个进程打开大数目的socket描述符(FD) 
    select 最不能忍受的是一个进程所打开的FD是有一定限制的，由FD_SETSIZE设置，默认值是2048。对于那些需要支持的上万连接数目的IM服务器来说显 然太少了。这时候你一是可以选择修改这个宏然后重新编译内核，不过资料也同时指出这样会带来网络效率的下降，二是可以选择多进程的解决方案(传统的 Apache方案)，不过虽然linux上面创建进程的代价比较小，但仍旧是不可忽视的，加上进程间数据同步远比不上线程间同步的高效，所以也不是一种完 美的方案。不过 epoll则没有这个限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左 右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。
2.IO效率不随FD数目增加而线性下降 
    传统的select/poll另一个致命弱点就是当你拥有一个很大的socket集合，不过由于网络延时，任一时间只有部分的socket是"活跃"的， 但是select/poll每次调用都会线性扫描全部的集合，导致效率呈现线性下降。但是epoll不存在这个问题，它只会对"活跃"的socket进行 操作---这是因为在内核实现中epoll是根据每个fd上面的callback函数实现的。那么，只有"活跃"的socket才会主动的去调用 callback函数，其他idle状态socket则不会，在这点上，epoll实现了一个"伪"AIO，因为这时候推动力在os内核。在一些 benchmark中，如果所有的socket基本上都是活跃的---比如一个高速LAN环境，epoll并不比select/poll有什么效率，相 反，如果过多使用epoll_ctl,效率相比还有稍微的下降。但是一旦使用idle connections模拟WAN环境,epoll的效率就远在select/poll之上了。
3.使用mmap加速内核与用户空间的消息传递。 
    这点实际上涉及到epoll的具体实现了。无论是select,poll还是epoll都需要内核把FD消息通知给用户空间，如何避免不必要的内存拷贝就 很重要，在这点上，epoll是通过内核于用户空间mmap同一块内存实现的。而如果你想我一样从2.5内核就关注epoll的话，一定不会忘记手工 mmap这一步的。
4.内核微调 
    这一点其实不算epoll的优点了，而是整个linux平台的优点。也许你可以怀疑linux平台，但是你无法回避linux平台赋予你微调内核的能力。 比如，内核TCP/IP协议栈使用内存池管理sk_buff结构，那么可以在运行时期动态调整这个内存pool(skb_head_pool)的大小 --- 通过echo XXXX>/proc/sys/net/core/hot_list_length完成。再比如listen函数的第2个参数(TCP完成3次握手 的数据包队列长度)，也可以根据你平台内存大小动态调整。更甚至在一个数据包面数目巨大但同时每个数据包本身大小却很小的特殊系统上尝试最新的NAPI网 卡驱动架构。

## 附录2：Round Robin 
轮叫调度（Round Robin Scheduling）算法就是以轮叫的方式依次将请求调度不同的服务器，即每次调度执行i = (i + 1) mod n，并选出第i台服务器。算法的优点是其简洁性，它无需记录当前所有连接的状态，所以它是一种无状态调度。
