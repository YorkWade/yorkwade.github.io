---
layout:     post
title:      muduo阅读笔记之第5章
subtitle:   高效的多线程日志
date:       2018-03-18
author:     BY
header-img: img/post-bg-mma-0.png
catalog: true
tags:
    - muduo
---

日志分为：   
        诊断日志
        交易日志（transaction log）
关键进程，日志要记录：
1、收到每条内部消息的id
2、收到每条外部消息的全文。
3、发出每条消息的全文，每条消息都应有全局唯一的id。
4、关键内部状态的变更。
日志不光是给程序员看的，更多的时候是给运维人员看的。是典型的生产者消费者问题。
muduo没有使用标准库的iostream,而是自己写的LogStream，主要考虑性能（&11.6.6）
muduo::Logger::setLogLevel()能够调整日志级别，即使生效
日志的目的地只有一个，本地文件，日志滚动是必须的（文件大小和时间），方便归档。往网络写日志是不靠谱的。
日志名如下：process_name.20120620-144022.hostname.3605.log
日志的压缩与归档不是日志应有的功能，应该由专门的程序或脚本来做,不应动业务程序。
笔者认为：日志不应该循环，以免磁盘空间写满。丢失关键日志。
日志常见问题：程序崩溃，最后若干条日志丢失，更不能每条日志写完就刷到磁盘里，影响性能。muduo定期（默认3秒）flush到磁盘。每条日志内存中的日志都带有cookie（sentry），其值为某个            函数地址，通过core dump中查找cookie能够找到尚未来得及写入的磁盘消息。
日志消息内容：
尽量一行。可以grep -o '20120603 08:02..' | sort |uniq -c查找。
精确微妙。每条用gettimeofday获取当前时间，没有什么性能损失。
使用GMT时区（Z）。对于跨洲分布式系统可以省去本地时区转换麻烦。
打印线程id。

7200rpmSATA硬盘写日志，磁盘带宽约是110M/s，日志库应该能瞬时写满这个带宽。（每条日志110字节，1秒要写100万条日志）。如果cpu跑满和磁盘带宽可以做到1秒10万条那么1秒写10万条，能腾出90%的资源干正事。muduo的日志库每秒200万条，能撑满两个千兆网或4个SATA组合。
笔者认为一个程序同时写多个日志文件是非常罕见的，这个可以留给log archiver来分流，不必做到日志库中。不做filter日志过滤器能见效开销。
## 双缓冲技术：
        准备两块buffer:A和B。前段往A填数据，后端负责将B的数据写入文件。当A写满后，交换A和B。前端往B写，如此反复。
实际实现中，可以采用四缓冲（前端2个，后端2个），可以进一步减少或避免日志前端的等待。也可根据需求进一步增加buffer数目。
日志堆积，可丢掉多于入职buffer，腾出内存。

linux设置core dump路径和名称：
    sysctlshezhi kernel.core_pattern（也可修改/proc/sys/kernel/core_pattern）

## 我的总结：
    1、日志不光是给程序员看的，更多的时候是给运维人员看的。
    2、生产者消费者模型中常用双缓冲技术。
    3、boost::ptr_vector<T>来实现指针交换，移动（prt_container::move）和对象生命周期管理。
    4、7200rpmSATA硬盘写日志，磁盘带宽约是110M/s，日志库应该能瞬时写满这个带宽。（每条日志110字节，1秒要写100万条日志）
## 附录1：
boost::ptr_vector
        boost::ptr_vector专门用于动态分配的对象，它使用起来更容易也更高效。 boost::ptr_vector 独占它所包含的对象，因而容器之外的共享指针不能共享所有权，这跟 std::vector<boost::shared_ptr<int> > 相反。
除了boost::ptr_vector之外，专门用于管理动态分配对象的容器还包括：boost::ptr_deque， boost::ptr_list，boost::ptr_set，boost::ptr_map，boost::ptr_unordered_set和 boost::ptr_unordered_map。这些容器等价于C++标准里提供的那些。
## 附录2：
ConcurrentHashMap的锁分段技术
     HashTable容器在竞争激烈的并发环境下表现出效率低下的原因，是因为所有访问HashTable的线程都必须竞争同一把锁，那假如容器里有多把锁，每一把锁用于锁容器其中一部分数据，那么当多线程访问容器里不同数据段的数据时，线程间就不会存在锁竞争，从而可以有效的提高并发访问效率，这就是ConcurrentHashMap所使用的锁分段技术，首先将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。
