---
layout:     post
title:      leveldb之Varint、Key
subtitle:   编码
date:       2020-02-08
author:     BY
header-img: img/post-bg-debug.png
catalog: true
tags:
    - leveldb
---

> leveldb中编码如何省空间

levelDB设计时希望尽量能够节省时间，实现了Varint这种编码方式，其实这种方式在[protobuf编码](https://www.wandouip.com/t5i125413/）等地方也都存在类似技巧的影子。

#### 理论背景
Varint是一种比较特殊的整数类型，它包含有Varint32和Varint64两种，它相比于int32和int64最大的特点是长度可变。
-    varint是一种紧凑的表示数字的方法，它用一个或多个字节来表示一个数字，值越小的数字使用越少的字节数。比如对于int32类型的数字，一般需要4个byte来表示。采用Varint，对于很小的int32类型的数字，则可以用1个字节来表示。大的数字则可能需要5个字节来表示。从统计的角度来说，一般不会所有的消息中的数字都是大数，因此大多数情况下，采用Varint 后，可以用更少的字节数来表示数字信息
-    varint中的每个字节的最高位（bit）有特殊含义，如果该位为1，表示后续的字节也是这个数字的一部分，如果该位为0，则结束。其他的7位（bit）都表示数字。

![](https://lrita.github.io/images/posts/leveldb/number_300_varint.png)

#### 实现要点
- int32,可能需要5个字节才能存放{5 * 8 - 5(标识位) > 32}
- int64,最多需要需要(10 * 8 - 10 > 64)10位来保存
	
#### 源码分析  

关于MemTable的内容前面已经讲的差不多了，但不知道读者有没有注意到这几个函数：
VarintLength
EncodeVarint32
EncodeFixed64
GetVarint32Ptr
DecodeFixed64
GetLengthPrefixedSlice





	
	
- [LevelDB源码剖析之Varint](http://mingxinglai.com/cn/2013/01/leveldb-varint32/)
- [Leveldb varint 解析](https://ce39906.github.io/2018/04/17/Leveldb-varint-%E8%A7%A3%E6%9E%90/)
- [LevelDB源码剖析之基础部件-SkipList](https://www.jianshu.com/p/6624befde844)
- [leveldb 源码分析(三) – Write](https://youjiali1995.github.io/storage/leveldb-write/)
- [理解 C++ 的 Memory Order](https://senlinzhan.github.io/2017/12/04/cpp-memory-order/)
