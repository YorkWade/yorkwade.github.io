---
layout:     post
title:      muduo阅读笔记之第3章
subtitle:   多线程服务器的使用场合与常用编程模型
date:       2018-03-13
author:     BY
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Swift
---


单线程服务器的常用编程模式
《UNP》第六章对IO模型有很好的总结。
io模型:

    同步阻塞
    同步非阻塞:需轮询
    同步非阻塞+io复用 :
        select 反应器 对于select/poll/epoll这三个是I/O不阻塞，但是在事件上阻塞，算是：I/O异步，事件同步的调用
    异步: 完成时操作系统通知 
        前摄器    因为没有任何的阻塞，无论是I/O上，还是事件通知上，所以，其可以让你充分地利用CPU，比起第二种同步无阻塞好处就是，
        第二种要你一遍一遍地去轮询。Nginx之所所以高效，是其使用了epoll和AIO的方式来进行I/O的。

在高性能的网络程序中，使用最广泛的模型是：non-blocking IO + IO multiplexing，即Reactor模式。（lighttpd、libevent、libev、ACE，poco C++libraries）。</br>
Proactor模式主要有，Boost.Asio和windows的IOCP，ACE也实现了Proactor。</br>
Reactor模式(non-blocking IO + IO multiplexing)基本结构： 一个事件循环，以事件驱动，事件回调（回调函数必须非阻塞）方式实现业务逻辑。</br>

Reactor，在linux下可用select/poll（伸缩性不足）和epoll实现。对于IO密集的应用是个不错的选择。lighttpd就是这样，内部的fdevent结构十分精妙，值得学习。</br>
基于事件驱动的编程模型缺点:它容易割裂业务逻辑，使其散布与多个回调函数之中，相对不容易理解和维护。</br>

多线程服务器的常用编程模型

    1、每个请求起一个线程，使用用阻塞IO操作。当请求多了，伸缩性不佳。
    2、使用线程池+阻塞式IO。
    3、使用non-blocking IO + IO multiplexing。java nio方式
    4、Leader/Follower等高级模式。
    
one loop per thread 即每个IO线程有一个event loop（Reactor），用于处理读写和定时事件。</br>
     好处：   
     
     线程数据基本固定，不会频繁创建和销毁。
     可以很方便的在县城建调配负载。
     IO事件发生的线程是固定的。
     
没有IO光有计算任务的线程，使用event loop有点浪费</br>
线程池</br>
阻塞队列（BlockQueue）是多线程编程的利器。线程池可用阻塞队列实现，生产者消费者也用阻塞队列。java util.concurrent 中的BlockingQueue教科书一致（1个mutex，2个condition vaviables）

推荐模式

    one （event）loop per thread  + thread pool。
    即，程序中有几个event (IO) loop线程（每个thread用non-blocking IO + IO multiplexing实现。）和一个线程池（线程数目根据cpu设定，并使用BlockQueue实现。）

进程间通信只用TCP </br>
好处：

    可以夸主机，具有伸缩性。其他IPC不能跨机器。tcp还能跨语言。
    两个进程同一台机器，使用共享内存提升一点性能，但让代码复杂度大大增加，不值得。何况tcp的local吞吐量一点都不低。
    
tcpcopy可用于压力测试。</br>
推荐google的Protocol Buffer可用于消息格式</br>
分布式系统中进程间使用tcp长连接。</br>
    容易定位分布式系统中的服务间的依赖关系。netstat -tpna|grep :port可列出用到的某服务的客户端地址（Foreign列）
    netstat可打印Recv-Q和Send-Q，这两个队列都应该接近0或0附近摆动。
    
        如果Recv-Q保持不变或持续增加，则意味着服务进程处理速度较慢，可能发生死锁或阻塞。
        如果Send-Q保持不变或者持续增加，可能对方服务器太忙甚至掉线。

多线程服务器的使用场合</br>
一台多核机器，提供一种服务或任务，可用的模式：

    1、运行一个单线程的进程。（编程简单，不能发挥多核计算能力。）
    2、运行一个多线程的进程。（编写复杂，提高响应，可根据cup数量扩展）
    3、运行多个单线程进程。（如果没有共享数据）
        3a、把模式1中的进程运行多分。
        3b、主进程+worker进程。
    4、运行多个多线程的进程。（汇聚2、3缺点）
    
单线程程序

    1、程序可能会fock。fork() 一般不能在多线程程序中调用，因为 Linux 的 fork() 只克隆当前线程的 thread of control，不克隆其他线程。也就是说不能一下子 fork() 出一个和父进程一样的多线程子进程，Linux 也没有 forkall() 这样的系统调用。forkall() 其实也是很难办的（从语意上），因为其他线程可能等在 condition variable 上，可能阻塞在系统调用上，可能等着 mutex 以跨入临界区，还可能在密集的计算中，这些都不好全盘搬到子进程里。看门狗进程必须坚持单线程，其他均可替换成多线程程序。
    2、单线程程序能限制程序的CPU占用率。多核机器最多占满一个core。可能非关键任务耗尽了cpu资源，请求响应慢。
    单线程的event loop可能发生优先级反转。可用多线程克服，这也是多线程主要优势。
多线程程序

    IO bound或CPU bound（例如subset sum，是NP-Complete）的服务，多线程都没有绝对意义上的优势。
    多线程，主要能提高响应速度，让IO和计算重叠，降低latency（延迟），并且能享受增加cpu数目带来的好处，提供非均质的服务（响应优先级高的事件可以用专门的线程）。

[机群管理软件（LLNL的SLURM）](https://computing.llnl.gov/tutorials/linux_clusters/)</br>
[slave实现要点](http://www.slideshare.net/chenshuo/zurg-part-1)
一台机器提供某个功能：</br>
    多线程程序中的线程大致分为3类：
    
        几个IO线程，也可有简单计算，如编码与解码。
        一个线程池，负责计算，不涉及IO。
        第三方哭所用线程，如logging、database connection。

虽然线程数据略多于core数目，但是这些线程很多时候是空闲的，可以让OS的进程调度来保证可控的延迟</br>。
round-robin轮询算法：轮叫调度（Round Robin Scheduling）算法就是以轮叫的方式依次将请求调		  度不同的服务器，即每次调度执行i = (i + 1) mod n，并选出第i台服务器。算法的优点是其简洁性，它  无需记录当前所有连接的状态，所以它是一种无状态调度</br>
32-bit linux能同时起300个左右的线程。</br>
    一个进程地址空间为4GB，用户态能访问3GB，一个线程默认栈大小是10MB,一个进程大约最多能同时启动300个线程。（如果算上数据段、代码段、堆动态库，300线程个是上限）</br>
多线程不能提高并发度（并发连接数。指one loop per thread）</br>
    如果一个线程一个连接，最多也就300个。如果one loop per thread，单个event loop处理1万个并发连接并不罕见。一个multi-loop多线程程序应该能轻松支持5万个并发连接。</br>
多线程不能提高吞吐量</br>
    计算密集型服务，不能。8个进程和一个带有8个线程的进程，理论上总耗时相当，因为CPU都是满载。但后者地产一次的响应延时更小。8个进程，每个进程压缩一个文件，而一个8线程的进程，依次压缩8个文件，后者较快拿到第一个文件，但总耗时一样。（个人理解，第一个文件由8个线程同时处理跑满cpu，可以提前完成。在跑满cpu的情况下，所有任务完成的总时间一样）</br>
[《A DesignFramework for Highly Concurrent System》](https://people.eecs.berkeley.edu/~culler/papers/events.pdf)
多线程能降低响应时间</br>
    一个请求处理需要10ms，单线程进程处理8个请求，响应事件依次为10ms、20ms、30ms……。一个多线程响应事件差不多都是10ms。单线程不能发挥多核性能，多线程能发挥多核性能。</br>
    轮询调度算法(Round-Robin Scheduling)</br>
多线程如何IO和计算重叠，降低latency
基本思路是，把IO操作（通常是写操作）通过BlockingQueue交给背的线程做，自己不必等待。
    线程不能减少工作量，即不能减少cpu时间，但可以指令，在多核上执行，让我们提早结束，其实就是统筹方法。
[《谈谈数独》](http://blog.csdn.net/Solstice/article/details/2096209)
第三方库往往有自己的线程

    libmemcached只支持同步操作。
    MySql的官方C API不支持异步操作。
    PostgreSql提供异步API。可以融入程序所用的event loop。
    
线程池大小的阻抗匹配原则：T(线程个数)*P（单个线程占用cpu的时间，0<P<=1）=C（Cpu个数）</br>
    如果cpu饱和，再多的线程，也不能提高吞吐量反而因为上下文切换的开销降低性能。</br>
    Proactor比Reactor能够提高更多的并发度和吞吐量，但不能降低延迟。</br>
    代价是代码支离破碎，不易理解。</br>

## 我的总结：
    1、使用reactor模式（ one （event）loop per thread ）能适用大部分编程需要，如果追求更高吞吐率使用Proactor，但代码逻辑支离破碎。
    2、多线程不能提高吞吐量，只能降低响应时间。IO密集型、CPU密集型，多线程都没有优势。
    3、多线程的推荐模式：
            几个IO线程（one （event）loop per thread）
            一个线程池（负责计算）
            第三方库自带线程。
    4、进程通信只用tcp长连接

