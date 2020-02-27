---
layout:     post
title:      muduo阅读笔记之第7章
subtitle:   muduo编程示例
date:       2018-03-25
author:     BY
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - muduo
---

> 

### 五个简单协议
- discard - 丢弃所有收到的数据；</br>
- daytime - 服务端 accept 连接之后，以字符串形式发送当前时间，然后主动断开连接；</br>
              用 netcat 扮演客户端，运行结果如下：</br>
```obj
$ nc 127.0.0.1 2013 
2011-02-02 03:31:26.622647    # 服务器返回的时间字符串
```
- time - 服务端 accept 连接之后，以二进制形式发送当前时间（从 Epoch 到现在的秒数），然后主动断开连接；我们需要一个客户程序来把收到的时间转换为字符串。</br>
用 netcat 扮演客户端，并用 hexdump 来打印二进制数据，运行结果如下：</br>
```obj
$ nc 127.0.0.1 2037 | hexdump -C 
00000000  4d 48 d0 d5                                       |MHÐÕ| 
00000004
```
注意其中考虑到了如果数据没有一次性收全，已经收到的数据会暂存在 Buffer 里，以等待下一次机会，程序也不会阻塞。这样即便服务器一个字节一个字节地发送数据，代码还是能正常工作，		这也是非阻塞网络编程必须在用户态使用接受缓冲的主要原因。</br>
- echo - 回显服务，把收到的数据发回客户端；</br>
这段代码实现的不是行回显(line echo)服务，而是有一点数据就发送一点数据。这样可以避免客户端恶意地不发送换行字符，而服务端又必须缓存已经收到的数据，导致服务器内存暴涨。但这个程序还是有一个安全漏洞，即如果客户端故意不断发生数据，但从不接收，那么服务端的发送缓冲区会一直堆积，导致内存暴涨。解决办法可以参考下面的 chargen 协议。</br>
- chargen - 服务端 accept 连接之后，不停地发送测试数据。</br>
netcat扮演客户端</br>

### 文件传输
   支持上万客户连接，内存消耗只与并发连接有关，跟文件大小无关。</br>
   解决内存问题，采用分割成小块发送，先发送文件的前64k数据，等这块数据发送完毕时，在发送下64k，如此往复。需每个conncetion保存用户的上下文（FILE*）。整个程序可以共享一个文件描述符，然后每个连接记住自己当前的偏移量。还需要限制最大并发连接数，避免客户端恶心发起连接，不接受数据，耗尽文件描述符。</br>
   
**TcpConnection::Send()注意事项：</br>**
1、返回值void，用户不必关心send成功发送了多少字节，muduo库会保证把数据发给对方。</br>
2、send()非阻塞。客户端只管把一条消息准备好，调用send发送，即便tcp窗口满了，也绝不会阻塞当前线程。</br>
3、send是线程安全的、原子的。</br>
4、提供send(void* message,size_t len);可以发送任意字节序列。</br>
5、send(const StringPiece& message)可发送std::string  和const char*。StringPiece是google发明的专门用于传输字符串参数的class。</br>
6、send(Buffer*)不是const引用，因为函数中可能用Buffer::swap高效的交换数据。</br>
7、如果支持c++11，可以增加对右值引用的重载，可以使用move避免内存拷贝。</br>
**关闭连接，但数据已经在路上，不会漏收**
        shutdown：先关本地“写”，再关本地“读”。（::shutdown(sockfd,SHUT_WR)） TCP half-close</br>
        close：关“读写”。</br>
        muduo 把“主动关闭连接”这件事情分成两步来做，如果要主动关闭连接，它会先关本地“写”端，等对方关闭之后，再关本地“读”端。</br>
        如何得知被动关闭：当read返回0时，得知对方关闭，应该关闭连接（如果不关闭，一直半开着，可能会造成泄漏）。</br>
        shutdown是tcp half-close。当read返回0时，你还可以发数。</br>
        完整的流程是：我们发完了数据，于是 shutdownWrite，发送 TCP FIN 分节，对方会读到 0 字节，然后对方通常会关闭连接，这样 muduo 会读到 0 字节，然后 muduo 关闭连接。（思考题，在 shutdown() 之后，muduo 回调 connection callback 的时间间隔大约是一个 round-trip time，为什么？）</br>
        参考：
      《TCP/IP 详解》第一卷第 18.5 节，TCP Half-Close。</br>
      《UNIX 网络编程》第一卷第三版第 6.6 节， shutdown() 函数。</br>
boost.asio的聊服务器</br>
**TCP分包（“粘包”问题）**
        分包：从字节流中识别并截取（还原）一个个消息，这是网络编程的基本要求。“粘包命题”是个伪命题。</br>
        需要协议能够区分包。（特殊字符如\r\n结尾，或者最常见的是包头加消息长度字段等）文件描述符。a会不会错误的把消息发给c？</br>
如何防止串话？a服务器要从连接收到的数据发给b。b有可能随时断开连接，而新建立连接的c可能敲好复用了b的</br>
muduo有微创新，预留8字节的空间。</br>
        注意是否支持内存对齐，否则可能出现dump。</br>
        需要注意，消息可能分几次到达。需要代码判断。</br>
```objc
#ifndef MUDUO_EXAMPLES_ASIO_CHAT_CODEC_H  
#define MUDUO_EXAMPLES_ASIO_CHAT_CODEC_H  
  
#include <muduo/base/Logging.h>  
#include <muduo/net/Buffer.h>  
#include <muduo/net/Endian.h>  
#include <muduo/net/TcpConnection.h>  
  
#include <boost/function.hpp>  
#include <boost/noncopyable.hpp>  
  
class LengthHeaderCodec : boost::noncopyable  
{  
 public:  
  typedef boost::function<void (const muduo::net::TcpConnectionPtr&,  
                                const muduo::string& message,  
                                muduo::Timestamp)> StringMessageCallback;  
  
  explicit LengthHeaderCodec(const StringMessageCallback& cb)  
    : messageCallback_(cb)  
  {  
  }  
/*消息到达的回调函数*/  
  void onMessage(const muduo::net::TcpConnectionPtr& conn,  
                 muduo::net::Buffer* buf,  
                 muduo::Timestamp receiveTime)  
  {  
  /*这里可能有多条信息一起到达*/  
    while (buf->readableBytes() >= kHeaderLen) // kHeaderLen == 4  
    {  
      // FIXME: use Buffer::peekInt32()  
        /*这里的消息包括消息头(包头)和消息尾(包体)*/  
      const void* data = buf->peek(); //这里只是查看一下数据而已，并没有取出数据  
      /*读出的是对方发过来的网络字节序(大端)的前4个字节(header)*/  
      int32_t be32 = *static_cast<const int32_t*>(data); // SIGBUS  
        /*把网络字节转为主机字节序*/  
      const int32_t len = muduo::net::sockets::networkToHost32(be32);  
        /*这里假设消息的包体长度不超过64k */  
      if (len > 65536 || len < 0) //消息不合法  
      {  
        LOG_ERROR << "Invalid length " << len;  
        conn->shutdown();  // FIXME: disable reading  
        break;  
      }  
  
      else if (buf->readableBytes() >= len + kHeaderLen)  
      {  
        buf->retrieve(kHeaderLen);  
        /*这里还没有取出消息的包体，只是peek一下*/  
        muduo::string message(buf->peek(), len);  
        /*回调应用程序，让应用层来处理包体*/  
        messageCallback_(conn, message, receiveTime);  
        /*取出包体*/  
        buf->retrieve(len);  
      }  
      /*未达到完整的一条消息*/  
      else  
      {  
        break;  
      }  
    }  
  }  
  
  // FIXME: TcpConnectionPtr  
    /*编码函数*/  
  void send(muduo::net::TcpConnection* conn,  
            const muduo::StringPiece& message)  
  {  
    muduo::net::Buffer buf;  
    buf.append(message.data(), message.size());  
    int32_t len = static_cast<int32_t>(message.size());  
    int32_t be32 = muduo::net::sockets::hostToNetwork32(len);  
    buf.prepend(&be32, sizeof be32);  
    /*编完码后,发送出去*/  
    conn->send(&buf);  
  }  
  
 private:  
  StringMessageCallback messageCallback_;  
  const static size_t kHeaderLen = sizeof(int32_t);  
};  
```
        复杂的情况使用状态机解码。
### 编解码器（LengthHeaderCodec）</br>
        应该让用户代码只关心“消息到达”而不是“数据到达”。</br>
        codec 的基本功能之一是做 TCP 分包：确定每条消息的长度，为消息划分界限。在 non-blocking 网络编程中，codec 几乎是必不可少的。如果只收到了半条消息，那么不会触发消息回调，数据会停留在 Buffer 里（数据已经读到 Buffer 中了），等待收到一个完整的消息再通知处理函数。</br>    

### 为什么non-blocking 网络编程中应用层buffer是必须的</br>
  C10k问题。  《The C10K problem》</br>
《The C10K problem》英文原版地址：http://www.kegel.com/c10k.html</br>
《The C10K problem》中文译文地址：地址1、地址2</br>
作者希望muduo库在易用性上有所作为。</br>
IO线程只能阻塞在IO multiplexing函数上，这样一来，应用层的缓冲是必须的，每个tcp socket都应该有stateful的input buffer和output buffer。
收发都有可能发生一次动作只完成部分数据，剩下部分数据的情况，所以需要缓冲区继续完成剩余部分。</br>
TcpConnection 必须要有 output buffer</br>
考虑一个常见场景：程序想通过 TCP 连接发送 100k 字节的数据，但是在 write() 调用中，操作系统只接受了 80k 字节（受 TCP advertised window 的控制，细节见 TCPv1），你肯定不想在原地等待，因为不知道会等多久（取决于对方什么时候接受数据，然后滑动 TCP 窗口）。程序应该尽快交出控制权，返回 event loop。在这种情况下，剩余的 20k 字节数据怎么办？</br>
对于应用程序而言，它只管生成数据，它不应该关心到底数据是一次性发送还是分成几次发送，这些应该由网络库来操心，程序只要调用 TcpConnection::send() 就行了，网络库会负责到底。网络库应该接管这剩余的 20k 字节数据，把它保存在该 TCP connection 的 output buffer 里，然后注册 POLLOUT 事件，一旦 socket 变得可写就立刻发送数据。当然，这第二次 write() 也不一定能完全写入 20k 字节，如果还有剩余，网络库应该继续关注 POLLOUT 事件；如果写完了 20k 字节，网络库应该停止关注 POLLOUT，以免造成 busy loop。（Muduo EventLoop 采用的是 epoll level trigger，这么做的具体原因我以后再说。）如果程序又写入了 50k 字节，而这时候 output buffer 里还有待发送的 20k 数据，那么网络库不应该直接调用 write()，而应该把这 50k 数据 append 在那 20k 数据之后，等 socket 变得可写的时候再一并写入。如果 output buffer 里还有待发送的数据，而程序又想关闭连接（对程序而言，调用 TcpConnection::send() 之后他就认为数据迟早会发出去），那么这时候网络库不能立刻关闭连接，而要等数据发送完毕，见我在《为什么 muduo 的 shutdown() 没有直接关闭 TCP 连接？》一文中的讲解。</br>
综上，要让程序在 write 操作上不阻塞，网络库必须要给每个 tcp connection 配置 output buffer。</br>
TcpConnection 必须要有 input buffer</br>
TCP 是一个无边界的字节流协议，接收方必须要处理“收到的数据尚不构成一条完整的消息”和“一次收到两条消息的数据”等等情况。一个常见的场景是，发送方 send 了两条 10k 字节的消息（共 20k），接收方收到数据的情况可能是：</br>
```obj
一次性收到 20k 数据
分两次收到，第一次 5k，第二次 15k
分两次收到，第一次 15k，第二次 5k
分两次收到，第一次 10k，第二次 10k
分三次收到，第一次 6k，第二次 8k，第三次 6k
```
其他任何可能</br>
网络库在处理“socket 可读”事件的时候，必须一次性把 socket 里的数据读完（从操作系统 buffer 搬到应用层 buffer），否则会反复触发 POLLIN 事件，造成 busy-loop。（Again, Muduo EventLoop 采用的是 epoll level trigger，这么做的具体原因我以后再说。）</br>
那么网络库必然要应对“数据不完整”的情况，收到的数据先放到 input buffer 里，等构成一条完整的消息再通知程序的业务逻辑。这通常是 codec 的职责，见陈硕《Muduo 网络编程示例之二：Boost.Asio 的聊天服务器》一文中的“TCP 分包”的论述与代码。</br>
所以，在 tcp 网络编程中，网络库必须要给每个 tcp connection 配置 input buffer。</br>

        muduo采用epoll的level trigger，1、与传统poll兼容；2、易于编程；3、不必等待出现EAGIN。

Muduo Buffer 的设计考虑了常见的网络编程需求，试图在易用性和性能之间找一个平衡点，目前这个平衡点更偏向于易用性。</br>
        buffer设计参考Netty的ChannelBuffer和libevent 1.4x的evbuffer。</br>
### Muduo Buffer 的设计要点：
对外表现为一块连续的内存(char*, len)，以方便客户代码的编写。
其 size() 可以自动增长，以适应不同大小的消息。它不是一个 fixed size array (即 char buf[8192])。
内部以 vector of char 来保存数据，并提供相应的访问函数。
Buffer 其实像是一个 queue，从末尾写入数据，从头部读出数据

![](https://images.cnblogs.com/cnblogs_com/Solstice/201104/201104171223595707.png)

![](https://images.cnblogs.com/cnblogs_com/Solstice/201104/201104171223592850.png)

readIndex 和 writeIndex 满足以下不变式(invariant):
0 ≤ readIndex ≤ writeIndex ≤ data.size()

buffer操作
1、初始状态：
![](https://images.cnblogs.com/cnblogs_com/Solstice/201104/201104171223592817.png)

2、如果有人向 Buffer 写入了 200 字节，那么其布局是：
![](https://images.cnblogs.com/cnblogs_com/Solstice/201104/201104171224005292.png)

3、如果有人从 Buffer read() & retrieve() （下称“读入”）了 50 字节
![](https://images.cnblogs.com/cnblogs_com/Solstice/201104/201104171224009719.png)

4、然后又写入了 200 字节，writeIndex 向后移动了 200 字节，readIndex 保持不变
![](https://images.cnblogs.com/cnblogs_com/Solstice/201104/201104171224007734.png)

5、接下来，一次性读入 350 字节，请注意，由于全部数据读完了，readIndex 和 writeIndex 返回原位以备新一轮使用
![](https://images.cnblogs.com/cnblogs_com/Solstice/201104/20110417122400209.png)


解决减少内存占用（如果有 10k 个连接，每个连接一建立就分配 64k 的读缓冲的话，将占用 640M 内存，而大多数时候这些缓冲区的使用率很低。）
        在栈上准备一个 65536 字节的 stackbuf，然后利用 readv() 来读取数据，iovec 有两块，第一块指向 muduo Buffer 中的 writable 字节，另一块指向栈上的 stackbuf。这样如果读入的数据不多，那么全部都读到 Buffer 中去了；如果长度超过 Buffer 的 writable 字节数，就会读到栈上的 stackbuf 里，然后程序再把 stackbuf 里的数据 append 到 Buffer 中。利用了临时栈上空间，避免开巨大 Buffer 造成的内存浪费，也避免反复调用 read() 的系统开销（通常一次 readv() 系统调用就能读完全部数据）
buffer的size() 可以自动增长
        内部使用vector，当内存空间不够时，vector会重新分配空间，原来的指针会失效，所以使用int类型的索引操作偏移，而不是指针。

***Zero copy ?***
如果对性能有极高的要求，受不了 copy() 与 resize()，那么可以考虑实现分段连续的 zero copy buffer 再配合 gather scatter IO
libevent 2.0.x 的设计方案。TCPv2介绍的 BSD TCP/IP 实现中的 mbuf 也是类似的方案，Linux 的 sk_buff 估计也差不多

![](https://images.cnblogs.com/cnblogs_com/Solstice/201104/201104171224051699.png)

性能
        prepend预留8字节空间，是为了序列化消息后，在前面添加消息长度，用空间换时间。
        目前最常用的千兆以太网的裸吞吐量是 125MB/s，扣除以太网 header、IP header、TCP header之后，应用层的吞吐率大约在 115 MB/s 上下。而现在服务器上最常用的 DDR2/DDR3 内存的带宽至少是 4GB/s，比千兆以太网高 40 倍以上。就是说，对于几 k 或几十 k 大小的数据，在内存里边拷几次根本不是问题，因为受以太网延迟和带宽的限制，跟这个程序通信的其他机器上的程序不会觉察到性能差异。
        如果你实现的服务程序要跟数据库打交道，那么瓶颈常常在 DB 上，应该把精力投入在 DB 调优上。
Muduo 的设计目标之一是吞吐量能让千兆以太网饱和，也就是每秒收发 120 兆字节的数据。这个很容易就达到，不用任何特别的努力。

### 使用Protobuf进行网络编程
        需要解决的问题：
            1、应用程序自己切分消息。方案：消息前面加个固定长度的帧头，带长度。
            2、发送发需要把类型信息传给接受放。方案：需要由发送方把类型信息传给给接收方，接收方根据type n
ame自动创建message对象。

![](https://img-my.csdn.net/uploads/201104/3/0_13018174768fBe.gif)

从协议层面设计区分消息类型。
protobuf有意不加长度和消息类型，只有在使用tcp长连接，且在一个连接上传递不止一种消息的情况下，需要上种打包方式（长度和类型）。
tcp收到数据，需要Codec编解码拦截（半条消息），收到完整消息后，需要dispatcher根据消息类型，通过维护消息到回到函数的映射，分发给相应的回调函数。
### 定时器
时间相关任务：
1、获取当前时间，计算时间间隔。计时使用gettimeofday(毫秒级)，用户态调用开销小
2、转换时区和日期计算。使用tz database(也叫tzdata，多线程可能有问题)
3、定时操作。定时使用timerfd_create、timerfd_gettime、timerfd_settime。把时间变成文件描述符，融入reactor。不用sleep
计算统计吞吐量：每秒发送的字节数
### 测量网络延时：通过协议（NTP）计算。


时间差：校时使用，注意server和client不再一个时钟域，需要用时钟偏移（clock offset）矫正。在实际应用中，clock offset 要经过一个低通滤波才能使用，不然偶然性太大。
Nagle:广域网中，应用程序记录的发包时间与操作系统真正发包时间差不再是可以忽略的小间隔，需要设置TCP_NODELAY参数。

### timeing wheel踢掉空闲连接，限制最大连接数。
       本文的代码见 http://code.google.com/p/muduo/source/browse/trunk/examples/idleconnection
<Hashed and hierarchiacl timing wheels:efficient data structures for implementing a timer facility>  http://www.cs.columia.edu/~nahum/w6998/papers/sosp87-timing-wheels.pdf

如果一个连接连续几秒钟（后文以 8s 为例）内没有收到数据，就把它断开，为此有两种简单粗暴的做法：
每个连接保存“最后收到数据的时间 lastReceiveTime”，然后用一个定时器，每秒钟遍历一遍所有连接，断开那些 (now - connection.lastReceiveTime) > 8s 的 connection。这种做法全局只有一个 repeated timer，不过每次 timeout 都要检查全部连接，如果连接数目比较大（几千上万），这一步可能会比较费时。
每个连接设置一个 one-shot timer，超时定为 8s，在超时的时候就断开本连接。当然，每次收到数据要去更新 timer。这种做法需要很多个 one-shot timer，会频繁地更新 timers。如果连接数目比较大，可能对 reactor 的 timer queue 造成压力。
使用timing wheel能避免上述两种做法的缺点。连接超时不需要精确定时，只要大致 8 秒钟超时断开就行，多一秒少一秒关系不大。处理连接超时可以用一个简单的数据结构：8 个桶组成的循环队列。第一个桶放下一秒将要超时的连接，第二个放下 2 秒将要超时的连接。每个连接一收到数据就把自己放到第 8 个桶，然后在每秒钟的 callback 里把第一个桶里的连接断开，把这个空桶挪到队尾。这样大致可以做到 8 秒钟没有数据就超时断开连接。更重要的是，每次不用检查全部的 connection，只要检查第一个桶里的 connections，相当于把任务分散了。
Simple timing wheel 的基本结构是一个循环队列（boost::circular_buffer），还有一个指向队尾的指针 (tail)，这个指针每秒钟移动一格，就像钟表上的时针，timing wheel 由此得名。

连接超时被踢掉的过程
假设在某个时刻，conn 1 到达，把它放到当前格子中，它的剩余寿命是 7 秒。此后 conn 1 上没有收到数据。

1 秒钟之后，tail 指向下一个格子，conn 1 的剩余寿命是 6 秒。

又过了几秒钟，tail 指向 conn 1 之前的那个格子，conn 1 即将被断开。

下一秒，tail 重新指向 conn 1 原来所在的格子，清空其中的数据，断开 conn 1 连接。

连接刷新
如果在断开 conn 1 之前收到数据，就把它移到当前的格子里。

收到数据，conn 1 的寿命延长为 7 秒。

时间继续前进，conn 1 寿命递减，不过它已经比第一种情况长寿了。

多个连接
timing wheel 中的每个格子是个 hash set，可以容纳不止一个连接。
比如一开始，conn 1 到达。

随后，conn 2 到达，这时候 tail 还没有移动，两个连接位于同一个格子中，具有相同的剩余寿命。（下图中画成链表，代码中是哈希表。）

几秒钟之后，conn 1 收到数据，而 conn 2 一直没有收到数据，那么 conn 1 被移到当前的格子中。这时 conn 1 的寿命比 conn 2 长。


now-lastReceivetTime>timeout，需要全局只有一个repeated timer，而且每次要检查。
### 广播服务
        分布式的观察者模型，增加多个subscriber，不用修改publisher，实现解耦。
        


应用层广播在分布式系统中涌出很大。如体育比分转播，负载监控，状态监控trouble shooting。
 广播中，要将消息发送给1000个订阅者，只能一个个发。多线程使用一个全局锁会把多线程退化成单线程执行。  thread local 技巧，把1000个订阅分给4个线程，每个线程的操作基本是无锁的。代码见examples/asio/chat/server_threaded_highperformance.cc

### 串并转换
       把多个客户连接汇聚城一个内部tcp连接，让backend专心处理业务，无须关系多连接的并发。
        

        当从 client connection 收到数据，如何得知其 id ？
        boost::any 存放connection id
### socks4a代理服务器
        

            tunnel
Socks4a 的协议非常简单，请参考维基百科 http://en.wikipedia.org/wiki/SOCKS#SOCKS_4a 。


## 我的总结：
   1、文件传输，应达到内存消耗只与并发连接有关，跟文件大小无关。解决内存问题，采用分割成小块发送，每个conncetion保存用户的上下文（FILE*）整个程序可以共享一个文件描述符。</br>
   2、关闭连接使用shutdown （半关闭，即关闭“发”，还能收）而不是close，不会漏收。</br>
   3、被动关闭可以通过read返回值为0判断。</br>
   4、codec 的基本功能之一是做 TCP 分包：等待收到完整包（可能分几次），才通知dispatcher，通知对应消息的回调。</br>
   5、protobuf适用于自定义协议，需要增加包长和消息类型。</br>
   6、要让程序在 write 操作上不阻塞，网络库必须要给每个 tcp connection 配置 output buffer</br>



## 附录1：shutdown hafl-close
        

tcp半关闭：连接的一端结束它发送后还能接受来自另一端数据。
## 附录2：thread local
        ThreadLocal是什么
　　早在JDK 1.2的版本中就提供java.lang.ThreadLocal，ThreadLocal为解决多线程程序的并发问题提供了一种新的思路。使用这个工具类可以很简洁地编写出优美的多线程程序。
　　当使用ThreadLocal维护变量时，ThreadLocal为每个使用该变量的线程提供独立的变量副本，所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。
　　从线程的角度看，目标变量就象是线程的本地变量，这也是类名中“Local”所要表达的意思。
　　所以，在Java中编写线程局部变量的代码相对来说要笨拙一些，因此造成线程局部变量没有在Java开发者中得到很好的普及。
ThreadLocal的接口方法
ThreadLocal类接口很简单，只有4个方法，我们先来了解一下：
void set(Object value)设置当前线程的线程局部变量的值。
public Object get()该方法返回当前线程所对应的线程局部变量。
public void remove()将当前线程局部变量的值删除，目的是为了减少内存的占用，该方法是JDK 5.0新增的方法。需要指出的是，当线程结束后，对应该线程的局部变量将自动被垃圾回收，所以显式调用该方法清除线程的局部变量并不是必须的操作，但它可以加快内存回收的速度。
protected Object initialValue()返回该线程局部变量的初始值，该方法是一个protected的方法，显然是为了让子类覆盖而设计的。这个方法是一个延迟调用方法，在线程第1次调用get()或set(Object)时才执行，并且仅执行1次。ThreadLocal中的缺省实现直接返回一个null。
　　值得一提的是，在JDK5.0中，ThreadLocal已经支持泛型，该类的类名已经变为ThreadLocal<T>。API方法也相应进行了调整，新版本的API方法分别是void set(T value)、T get()以及T initialValue()。
　　ThreadLocal是如何做到为每一个线程维护变量的副本的呢？其实实现的思路很简单：在ThreadLocal类中有一个Map，用于存储每一个线程的变量副本，Map中元素的键为线程对象，而值对应线程的变量副本。我们自己就可以提供一个简单的实现版本：
[java] view plain copy print?


package com.test;  
  
public class TestNum {  
    // ①通过匿名内部类覆盖ThreadLocal的initialValue()方法，指定初始值  
    private static ThreadLocal<Integer> seqNum = new ThreadLocal<Integer>() {  
        public Integer initialValue() {  
            return 0;  
        }  
    };  
  
    // ②获取下一个序列值  
    public int getNextNum() {  
        seqNum.set(seqNum.get() + 1);  
        return seqNum.get();  
    }  
  
    public static void main(String[] args) {  
        TestNum sn = new TestNum();  
        // ③ 3个线程共享sn，各自产生序列号  
        TestClient t1 = new TestClient(sn);  
        TestClient t2 = new TestClient(sn);  
        TestClient t3 = new TestClient(sn);  
        t1.start();  
        t2.start();  
        t3.start();  
    }  
  
    private static class TestClient extends Thread {  
        private TestNum sn;  
  
        public TestClient(TestNum sn) {  
            this.sn = sn;  
        }  
  
        public void run() {  
            for (int i = 0; i < 3; i++) {  
                // ④每个线程打出3个序列值  
                System.out.println("thread[" + Thread.currentThread().getName() + "] --> sn["  
                         + sn.getNextNum() + "]");  
            }  
        }  
    }  
}  


 通常我们通过匿名内部类的方式定义ThreadLocal的子类，提供初始的变量值，如例子中①处所示。TestClient线程产生一组序列号，在③处，我们生成3个TestClient，它们共享同一个TestNum实例。运行以上代码，在控制台上输出以下的结果：
thread[Thread-0] --> sn[1]
thread[Thread-1] --> sn[1]
thread[Thread-2] --> sn[1]
thread[Thread-1] --> sn[2]
thread[Thread-0] --> sn[2]
thread[Thread-1] --> sn[3]
thread[Thread-2] --> sn[2]
thread[Thread-0] --> sn[3]
thread[Thread-2] --> sn[3]
考察输出的结果信息，我们发现每个线程所产生的序号虽然都共享同一个TestNum实例，但它们并没有发生相互干扰的情况，而是各自产生独立的序列号，这是因为我们通过ThreadLocal为每一个线程提供了单独的副本。                                         java.lang.ThreadLocal<T>的具体实现
        那么到底ThreadLocal类是如何实现这种“为每个线程提供不同的变量拷贝”的呢？先来看一下ThreadLocal的set()方法的源码是如何实现的：
[java] view plain copy print?


/** 
    * Sets the current thread's copy of this thread-local variable 
    * to the specified value.  Most subclasses will have no need to 
    * override this method, relying solely on the {@link #initialValue} 
    * method to set the values of thread-locals. 
    * 
    * @param value the value to be stored in the current thread's copy of 
    *        this thread-local. 
    */  
   public void set(T value) {  
       Thread t = Thread.currentThread();  
       ThreadLocalMap map = getMap(t);  
       if (map != null)  
           map.set(this, value);  
       else  
           createMap(t, value);  
   }  

在这个方法内部我们看到，首先通过getMap(Thread t)方法获取一个和当前线程相关的ThreadLocalMap，然后将变量的值设置到这个ThreadLocalMap对象中，当然如果获取到的ThreadLocalMap对象为空，就通过createMap方法创建。
线程隔离的秘密，就在于ThreadLocalMap这个类。ThreadLocalMap是ThreadLocal类的一个静态内部类，它实现了键值对的设置和获取（对比Map对象来理解），每个线程中都有一个独立的ThreadLocalMap副本，它所存储的值，只能被当前线程读取和修改。ThreadLocal类通过操作每一个线程特有的ThreadLocalMap副本，从而实现了变量访问在不同线程中的隔离。因为每个线程的变量都是自己特有的，完全不会有并发错误。还有一点就是，ThreadLocalMap存储的键值对中的键是this对象指向的ThreadLocal对象，而值就是你所设置的对象了。
为了加深理解，我们接着看上面代码中出现的getMap和createMap方法的实现：
[java] view plain copy print?


/** 
 * Get the map associated with a ThreadLocal. Overridden in 
 * InheritableThreadLocal. 
 * 
 * @param  t the current thread 
 * @return the map 
 */  
ThreadLocalMap getMap(Thread t) {  
    return t.threadLocals;  
}  
  
/** 
 * Create the map associated with a ThreadLocal. Overridden in 
 * InheritableThreadLocal. 
 * 
 * @param t the current thread 
 * @param firstValue value for the initial entry of the map 
 * @param map the map to store. 
 */  
void createMap(Thread t, T firstValue) {  
    t.threadLocals = new ThreadLocalMap(this, firstValue);  
}  

接下来再看一下ThreadLocal类中的get()方法:
[java] view plain copy print?


/** 
 * Returns the value in the current thread's copy of this 
 * thread-local variable.  If the variable has no value for the 
 * current thread, it is first initialized to the value returned 
 * by an invocation of the {@link #initialValue} method. 
 * 
 * @return the current thread's value of this thread-local 
 */  
public T get() {  
    Thread t = Thread.currentThread();  
    ThreadLocalMap map = getMap(t);  
    if (map != null) {  
        ThreadLocalMap.Entry e = map.getEntry(this);  
        if (e != null)  
            return (T)e.value;  
    }  
    return setInitialValue();  
}  

再来看setInitialValue()方法：
[java] view plain copy print?


/** 
    * Variant of set() to establish initialValue. Used instead 
    * of set() in case user has overridden the set() method. 
    * 
    * @return the initial value 
    */  
   private T setInitialValue() {  
       T value = initialValue();  
       Thread t = Thread.currentThread();  
       ThreadLocalMap map = getMap(t);  
       if (map != null)  
           map.set(this, value);  
       else  
           createMap(t, value);  
       return value;  
   }  

　　获取和当前线程绑定的值时，ThreadLocalMap对象是以this指向的ThreadLocal对象为键进行查找的，这当然和前面set()方法的代码是相呼应的。


　　进一步地，我们可以创建不同的ThreadLocal实例来实现多个变量在不同线程间的访问隔离，为什么可以这么做？因为不同的ThreadLocal对象作为不同键，当然也可以在线程的ThreadLocalMap对象中设置不同的值了。通过ThreadLocal对象，在多线程中共享一个值和多个值的区别，就像你在一个HashMap对象中存储一个键值对和多个键值对一样，仅此而已。
        Thread同步机制的比较
　　ThreadLocal和线程同步机制相比有什么优势呢？ThreadLocal和线程同步机制都是为了解决多线程中相同变量的访问冲突问题。
　　在同步机制中，通过对象的锁机制保证同一时间只有一个线程访问变量。程序设计和编写难度相对较大。
　　而ThreadLocal则从另一个角度来解决多线程的并发访问。ThreadLocal会为每一个线程提供一个独立的变量副本，每一个线程都拥有自己的变量副本，从而也就没有必要对该变量进行同步了。ThreadLocal提供了线程安全的共享对象，在编写多线程代码时，可以把不安全的变量封装进ThreadLocal。
        概括起来说，对于多线程资源共享的问题，同步机制采用了“以时间换空间”的方式，而ThreadLocal采用了“以空间换时间”的方式。前者仅提供一份变量，让不同的线程排队访问，而后者为每一个线程都提供了一份变量，因此可以同时访问而互不影响。
        总结一句话就是一个是锁机制进行时间换空间，一个是存储拷贝进行空间换时间。


附录3：Nagel算法
1. Nagel算法
        TCP/IP协议中，无论发送多少数据，总是要在数据前面加上协议头，同时，对方接收到数据，也需要发送ACK表示确认。为了尽可能的利用网络带宽，TCP总是希望尽可能的发送足够大的数据。（一个连接会设置MSS参数，因此，TCP/IP希望每次都能够以MSS尺寸的数据块来发送数据）。Nagle算法就是为了尽可能发送大块数据，避免网络中充斥着许多小数据块。
        Nagle算法的基本定义是任意时刻，最多只能有一个未被确认的小段。 所谓“小段”，指的是小于MSS尺寸的数据块，所谓“未被确认”，是指一个数据块发送出去后，没有收到对方发送的ACK确认该数据已收到。
        Nagle算法的规则（可参考tcp_output.c文件里tcp_nagle_check函数注释）：
      （1）如果包长度达到MSS，则允许发送；
      （2）如果该包含有FIN，则允许发送；
      （3）设置了TCP_NODELAY选项，则允许发送；
      （4）未设置TCP_CORK选项时，若所有发出去的小数据包（包长度小于MSS）均被确认，则允许发送；
      （5）上述条件都未满足，但发生了超时（一般为200ms），则立即发送。
        Nagle算法只允许一个未被ACK的包存在于网络，它并不管包的大小，因此它事实上就是一个扩展的停-等协议，只不过它是基于包停-等的，而不是基于字节停-等的。Nagle算法完全由TCP协议的ACK机制决定，这会带来一些问题，比如如果对端ACK回复很快的话，Nagle事实上不会拼接太多的数据包，虽然避免了网络拥塞，网络总体的利用率依然很低。
        Nagle算法是silly window syndrome(SWS)预防算法的一个半集。SWS算法预防发送少量的数据，Nagle算法是其在发送方的实现，而接收方要做的时不要通告缓冲空间的很小增长，不通知小窗口，除非缓冲区空间有显著的增长。这里显著的增长定义为完全大小的段（MSS）或增长到大于最大窗口的一半。
注意：BSD的实现是允许在空闲链接上发送大的写操作剩下的最后的小段，也就是说，当超过1个MSS数据发送时，内核先依次发送完n个MSS的数据包，然后再发送尾部的小数据包，其间不再延时等待。（假设网络不阻塞且接收窗口足够大）
        举个例子，比如之前的blog中的实验，一开始client端调用socket的write操作将一个int型数据（称为A块）写入到网络中，由于此时连接是空闲的（也就是说还没有未被确认的小段），因此这个int型数据会被马上发送到server端，接着，client端又调用write操作写入‘\r\n’（简称B块），这个时候，A块的ACK没有返回，所以可以认为已经存在了一个未被确认的小段，所以B块没有立即被发送，一直等待A块的ACK收到（大概40ms之后），B块才被发送。整个过程如图所示：

          这里还隐藏了一个问题，就是A块数据的ACK为什么40ms之后才收到？这是因为TCP/IP中不仅仅有nagle算法，还有一个TCP确认延迟机制 。当Server端收到数据之后，它并不会马上向client端发送ACK，而是会将ACK的发送延迟一段时间（假设为t），它希望在t时间内server端会向client端发送应答数据，这样ACK就能够和应答数据一起发送，就像是应答数据捎带着ACK过去。在我之前的时间中，t大概就是40ms。这就解释了为什么'\r\n'（B块）总是在A块之后40ms才发出。
        当然，TCP确认延迟40ms并不是一直不变的，TCP连接的延迟确认时间一般初始化为最小值40ms，随后根据连接的重传超时时间（RTO）、上次收到数据包与本次接收数据包的时间间隔等参数进行不断调整。另外可以通过设置TCP_QUICKACK选项来取消确认延迟。
        关于TCP确认延迟的详细介绍可参考：http://blog.csdn.net/turkeyzhou/article/details/6764389
2. TCP_NODELAY 选项
        默认情况下，发送数据采用Negale 算法。这样虽然提高了网络吞吐量，但是实时性却降低了，在一些交互性很强的应用程序来说是不允许的，使用TCP_NODELAY选项可以禁止Negale 算法。
        此时，应用程序向内核递交的每个数据包都会立即发送出去。需要注意的是，虽然禁止了Negale 算法，但网络的传输仍然受到TCP确认延迟机制的影响。
3. TCP_CORK 选项
        所谓的CORK就是塞子的意思，形象地理解就是用CORK将连接塞住，使得数据先不发出去，等到拔去塞子后再发出去。设置该选项后，内核会尽力把小数据包拼接成一个大的数据包（一个MTU）再发送出去，当然若一定时间后（一般为200ms，该值尚待确认），内核仍然没有组合成一个MTU时也必须发送现有的数据（不可能让数据一直等待吧）。
        然而，TCP_CORK的实现可能并不像你想象的那么完美，CORK并不会将连接完全塞住。内核其实并不知道应用层到底什么时候会发送第二批数据用于和第一批数据拼接以达到MTU的大小，因此内核会给出一个时间限制，在该时间内没有拼接成一个大包（努力接近MTU）的话，内核就会无条件发送。也就是说若应用层程序发送小包数据的间隔不够短时，TCP_CORK就没有一点作用，反而失去了数据的实时性（每个小包数据都会延时一定时间再发送）。
4. Nagle算法与CORK算法区别

  Nagle算法和CORK算法非常类似，但是它们的着眼点不一样，Nagle算法主要避免网络因为太多的小包（协议头的比例非常之大）而拥塞，而CORK算法则是为了提高网络的利用率，使得总体上协议头占用的比例尽可能的小。如此看来这二者在避免发送小包上是一致的，在用户控制的层面上，Nagle算法完全不受用户socket的控制，你只能简单的设置TCP_NODELAY而禁用它，CORK算法同样也是通过设置或者清除TCP_CORK使能或者禁用之，然而Nagle算法关心的是网络拥塞问题，只要所有的ACK回来则发包，而CORK算法却可以关心内容，在前后数据包发送间隔很短的前提下（很重要，否则内核会帮你将分散的包发出），即使你是分散发送多个小数据包，你也可以通过使能CORK算法将这些内容拼接在一个包内，如果此时用Nagle算法的话，则可能做不到这一点。


