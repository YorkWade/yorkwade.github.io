---
layout:     post
title:      leveldb之AtomicPointer
subtitle:   无锁编程
date:       2020-01-28
author:     BY
header-img: img/post-bg-mma-2.jpg
catalog: true
tags:
    - leveldb
---

AtomicPointer 是 leveldb 提供的一个原子指针操作类，使用了基于内存屏障的同步访问机制，这比用锁和信号量的效率要高。


## 理论

内存屏障是同步的一种方法，类似于锁和信号量，但是它有更高的效率。

CPU可以保证指针操作的原子性，但编译器、CPU指令优化--重排序(reorder)可能导致指令乱序，在多线程情况下程序运行结果不符合预期。关于重排序说明如下:
- 单核单线程时，重排序保证单核单线程下程序运行结果一致。
- 单核多线程时，编译器reorder可能导致运行结果不一致。参见[《memory-ordering-at-compile-time》](https://preshing.com/20120625/memory-ordering-at-compile-time/)。
- 多核多线程时，编译器reorder、CPU reorder将导致运行结果不一致。参见[《memory-reordering-caught-in-the-act》](https://www.jianshu.com/p/5b317882dda6)。


## 实现要点



这里包括 **关节功能+核心控制**、**基础动作模式**、**基础力量**、**综合体能**、**专项运动**。他们在逻辑上互为基础和进阶，关节是动作的基础，动作承载力量，力量支撑各个运动素质，而专项是各个运动素质在具体运动中的表现。


#### 什么是基础动作模式？

简单地说就是，所有动作肢体特有的运动程序。人体就这么些零件，所以很多的动作之间都存在着些许的共性，我们将这些共性提炼出来并进行功能上的抽象，那么就形成了我们现在所要说的基础动作模式——**双腿蹲**、**单腿蹲**、**推**、**拉**、**旋转**、**屈髋**。

- **蹲**：分为单腿蹲、双腿蹲。对应的训练动作有剪蹲和深蹲。
- **推**：分为水平推、竖直推。对应的训练动作是卧推和实力举。
- **拉**：分为竖直拉、水平拉。竖直拉包括引体向上、高位下拉，水平拉包括弹力带划船等等。
- **屈髋**：最具代表性的动作就是硬拉。
- **旋转**：动作比较复杂，在训练当中比较少出现，适合比较资深的训练者，比如说劈和砍，比如下劈球，比如拿锤子砸轮胎。前期不建议做，当你有一定训练水平的时候再去做旋转类动作。

![](https://ws4.sinaimg.cn/large/006tKfTcgy1fhg20yeticj30go0ptdmg.jpg)






>参考 

>- [《体能训练之金字塔》](https://preshing.com/20120625/memory-ordering-at-compile-time/)
>- [零基础健身者的运动发展流程](http://www.jianshenjiaolian.com.cn/lingjichu-fazhan.html)
