---
layout:     post
title:      leveldb之Log读写
subtitle:   文件映射
date:       2020-02-10
author:     BY
header-img: img/post-bg-mma-3.jpg
catalog: true
tags:
    - leveldb
---
>为了防止写入内存的数据库因为进程异常、系统掉电等情况发生丢失，leveldb在写内存之前会将本次写操作的内容写入日志文件中。


## 理论


在leveldb中，有两个memory db，以及对应的两份日志文件。其中一个memory db是可读写的，当这个db的数据量超过预定的上限时，便会转换成一个不可读的memory db，与此同时，与之对应的日志文件也变成一份frozen log。

而新生成的immutable memory db则会由后台的minor compaction进程将其转换成一个sstable文件进行持久化，持久化完成，与之对应的frozen log被删除。
![](https://leveldb-handbook.readthedocs.io/zh/latest/_images/two_log.jpeg)

### 日志结构
为了增加读取效率，日志文件中按照block进行划分，每个block的大小为32KiB。每个block中包含了若干个完整的chunk。

一条日志记录包含一个或多个chunk。每个chunk包含了一个7字节大小的header，前4字节是该chunk的校验码，紧接的2字节是该chunk数据的长度，以及最后一个字节是该chunk的类型。其中checksum校验的范围包括chunk的类型以及随后的data数据。

chunk共有四种类型：full，first，middle，last。一条日志记录若只包含一个chunk，则该chunk的类型为full。若一条日志记录包含多个chunk，则这些chunk的第一个类型为first, 最后一个类型为last，中间包含大于等于0个middle类型的chunk。

由于一个block的大小为32KiB，因此当一条日志文件过大时，会将第一部分数据写在第一个block中，且类型为first，若剩余的数据仍然超过一个block的大小，则第二部分数据写在第二个block中，类型为middle，最后剩余的数据写在最后一个block中，类型为last。
![](https://leveldb-handbook.readthedocs.io/zh/latest/_images/journal.jpeg)

### 日志内容
日志的内容为写入的batch编码后的信息。

具体的格式为：

![](https://leveldb-handbook.readthedocs.io/zh/latest/_images/journal_content.jpeg)

- [leveldb_handbook](https://leveldb-handbook.readthedocs.io/zh/latest/journal.html)
- [Skip Lists](https://www.csee.umbc.edu/courses/341/fall01/Lectures/SkipLists/skip_lists/skip_lists.html)
- [LevelDB源码剖析之基础部件-SkipList](https://www.jianshu.com/p/6624befde844)
- [leveldb 源码分析(三) – Write](https://youjiali1995.github.io/storage/leveldb-write/)
- [理解 C++ 的 Memory Order](https://senlinzhan.github.io/2017/12/04/cpp-memory-order/)
