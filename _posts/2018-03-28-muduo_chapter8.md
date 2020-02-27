---
layout:     post
title:      muduo阅读笔记之地8章
subtitle:   muduo网络库设计与实现
date:       2018-03-28
author:     BY
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - muduo
---


-   poll是level trigger，可以通过strace命令只管看到poll的参数列表。
-   定时器的接口，存在线程安全问题：mudu解决不是加锁，而是将操作转移到IO线程来进行。（通过函数指针boost::function<void()>，添加到IO线程中，让其执行）
-   c++标准保证std::vector的元素排列跟数组一样，可以&*vec.begin,vec.size()传数组指针和长度。
-   可以利用容器的swap，将加锁的对象交换到局部变量中，从而减小临界区的长度。
-   EventLoop::wakeup通过写一个字节数据，唤醒IO线程。
-   《accept() able Strategies for Improvig Web Server Performance》accept接受策略
-   暂时错误：EAGAIN、EINTER、EMFILE、ECONNABORED，要忽略此次错误；致命错误：ENFILE、ENOMEM，要终止程序。
-   依赖尽量单向。
-   TcpConnection是一次性的，不可再生，一旦连接断开就没啥用了。
-   从数组pollfds_ 中删除元素，可将待删除的元素与组后一个元素交换，再pollfds_.pop_back()，复杂度为O(1)。
-   level trigger，只调用一次read，不用反复调用read直到其返回EAGAIN。不会因某个连接数据量大，影响其他连接。因此，某些情况下edge trigger不见得比level trigger效率高。
-   level trigger，发送数据时，只在需要时关注writable事件，否则造成busy loop.
-   发送数据，如果发送了部分数据，需要把数据缓存起来，并开始关注writable事件；如果缓存中有待发数据，不能尝试发送，会造成数据乱序。
-   SIGPIPE默认行为时终止进程，再命令行程序中时合理的，但是在网络编程中，应该忽略SIGPIPE,方法::signal(SIGPIPE,SIG_IGN)
-   TCP No Delay选项禁用Nagle算法。避免连续发包出现延迟，对于低延迟网络服务很重要。方法：int optval=on?1:0 ::setsockopt(sockfd_,IPPROTO_TCP,TCP_NODELAY,&optval,sizeof optval)；
-   TCP keepalive选项是探查tcp连接是否存在。如果应用层有心跳，这个选项不是必须的。
-   发送数据速度高于对方接收数据速度，会造成数据在本地内存堆积，需要设置高低水位
   
- Connector实现的难点：
        1、socket是一次性的，一旦出错（比如对方拒绝连接），就无法恢复只能关闭重来。</br>
        2、错误代码：EAGAIN是真错误，表明端口用完。EINPROGRESS：正在连接。socket可，写不一定成功连接，还需要使用getsocketopt再次确认一下。</br>
        3、重试间隔应逐渐延长，0.5s、1、2、4、直至30s。back-off.</br>
        4、系统分配端口号，如果服务器和客户端在本机，可能会出现自连接。即客户端ip和端口号=服务器的ip和端口号。</br>
     
-   连接断开后初次重试应具有随机性，避免服务器崩溃，客户端同时发起连接，造成SYN丢包。SYN系统默认重发间隔为3秒。
-   poll每次返回整个文件描述符数组，需要便利数组找出有IO事件的文件描述符。epoll返回的是活动的fd列表，遍历的数组会小很多。再并发数较大而活动连接比例不高时，epoll比poll高效。

## 我的总结：
        1、多线程尽量不加锁，使用boost::function<void()>和boost::bind，让函数执行在一个线程中执行。
        2、可以利用容器的swap，将加锁的对象交换到局部变量中，从而减小临界区的长度。
        3、c++标准保证std::vector的元素排列跟数组一样，可以&*vec.begin,vec.size()传数组指针和长度。
        4、发送数据，如果发送了部分数据，需要把数据缓存起来，并开始关注writable事件；如果缓存中有待发数据，不能尝试发送，会造成数据乱序。
        5、socket是一次性的，一旦出错（比如对方拒绝连接），就无法恢复只能关闭重来。

## 附录1 function<void()>
最近开始写一个线程池，期间想用一个通用的函数模板来使得各个线程执行不同的任务，找到了Boost库中的function函数。
Boost::function是一个函数包装器，也即一个函数模板，可以用来代替拥有相同返回类型，相同参数类型，以及相同参数个数的各个不同的函数。
```objc
 1 #include<boost/function.hpp>
 2 #include<iostream>
 3 typedef boost::function<int(int ,char)> Func;
 4 
 5 int test(int num,char sign)
 6 {
 7    std::cout<<num<<sign<<std::endl
 8 }
 9 
10 int main()
11 {
12   Func f;
13   f=&test;  //or f=test
14   f(1,'A');
15 }
```
这样在不同的地方用不同的函数来替代 f 可以得到类似于C++中多态的效果。
但是这样有一定的局限性，例如我想实现的线程池需要执行不同的任务，这些任务的返回类型，函数参数个数，参数类型肯定是不同的，所以不能用上面的方法实现，那么怎么样定义一个函数模板来包含各种返回类型，函数参数个数，参数类型不同的各种函数呢？
我们可以定义如下的类型
1 typedef boost::function<void()> Func;
2 //or
3 typedef boost::function<void(void)> Func;
void 类型(空类型)其实是C中四种数据类型之一，其余三个为基本类型，构造类型，指针型。
空类型主要是用来修饰返回类型与函数参数的，不能用了定义变量，void i 是错误的。
可以这样认为空类型是一个抽象的基类，不能用了定义变量，但是它可以表示所有的类型，这样也不难理解，void* 指针能够不用强制转换成不同类型的指针了。
void类型的返回类型表示我们可以返回各种不同的类型了，那我们怎么样传入参数呢？可以用boost::bind。
``` objc
 1 #include<boost/function.hpp>
 2 #include<boost/bind.hpp>
 3 #include<iostream>
 4 typdef boost::function<void(void)> Func;
 5 
 6 int test(int num)
 7 {
 8    std::cout<<"In test"<<std::endl;    
 9 }
10 
11 int main()
12 {
13   Func f(boost::bind<test,6>);
14   f();
15 }
```
使用bind来传入各个参数，形成一个通用的函数模板。
不过由于f()的返回值是void类型，所以我们不能有以下写法：
```objc
1 int result=f();
2 //or
3 std::cout<<f()<<std<<endl;
```
不过没有关系，我们可以从参数中传出结果。
