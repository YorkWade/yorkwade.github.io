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
有些编译器默认会在编译期间对代码进行优化，从而改变汇编代码的指令执行顺序，如果你是在单线程上运行可能会正常，但是在多线程环境很可能会发生问题(如果你的程序对指令的执行顺序有严格的要求)。
而内存屏障就可以阻止编译器在编译期间优化我们的指令顺序，为你的程序在多线程环境下的正确运行提供了保障，但是不能阻止 CPU 在运行时重新排序指令。
如果你想实现下面这样的功能，那你可以考虑内存屏障：
修改一个内存中的变量之后，其余的 CPU 和 Cache 里面该变量的原始数据失效，必须从内存中重新获取这个变量的值

作者：程序小歌
链接：https://www.jianshu.com/p/11bbf63a4e35
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

作者：程序小歌
链接：https://www.jianshu.com/p/11bbf63a4e35
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

CPU可以保证**指针**操作的原子性，但编译器、CPU指令优化--重排序(reorder)可能导致指令乱序，在多线程情况下程序运行结果不符合预期。关于重排序说明如下:
- 单核单线程时，重排序保证单核单线程下程序运行结果一致。
- 单核多线程时，编译器reorder可能导致运行结果不一致。参见[《memory-ordering-at-compile-time》](https://preshing.com/20120625/memory-ordering-at-compile-time/)。
- 多核多线程时，编译器reorder、CPU reorder将导致运行结果不一致。参见[《memory-reordering-caught-in-the-act》](https://www.jianshu.com/p/5b317882dda6)。

简单来说，在不跨越cacheline情况下，Intel处理器保证指针操作的原子性；跨域cacheline情况下，部分处理器提供了原子保证。在通常情况下，C++ new出来的指针及对象内部数据都是cacheline对其的，但如果使用 align 1 byte或者采用c++ placement new等特性时可能出现指针对象跨越cacheline的情况。
在LevelDB中，指针操作是cacheline对齐的，因此问题一种NoBarrier_*的指针操作本身是原子的。那么，为何还需要Acqiure_Load和Release_Store呢？


- **问**：代码中NoBarrier_Store/NoBarrier_Load操作只是最简单的指针操作，那么这些操作是原子的么？
  **答**：在不跨越cacheline情况下，Intel处理器保证指针操作的原子性；跨域cacheline情况下，部分处理器提供了原子保证。LevelDB场景下不存在跨cacheline场景，因此这部分操作是原子的。
- **问**：Acquire_Load/Release_Store操作增加了MemoryBarrier操作，其作用是什么？又如何保证原子性呢？
  **答**：增加Memory Barrier是为了避免编译器重排序，保证MemoryBarrier前的全部操作真正在Memory Barrier前执行。
- **问**：为何要设计这样两组操作？
  **答**：性能。NoBarrier_Store/NoBarrier_Load的性能要优于Acquire_Load/Release_Store，但Acquire_Load/Release_Store可以避免编译器优化，由此保证load/store时指针里面的数据一定是最新的。
- **问**：LevelDB代码中如何选择何时使用何种操作？
  **答**：时刻小心。在任意一个用到指针的场景，结合上下文+并发考量选择合适的load/store方法。当然，一个比较保守的做法是，所有的场景下都使用带Memory Barrier的load/store方法，仅当确定可以使用NoBarrier的load/store方法才将其替换掉。

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

>- [《Memory Ordering at Compile Time》](https://preshing.com/20120625/memory-ordering-at-compile-time/)
>- [LevelDB源码剖析之基础部件-AtomicPointer](https://www.jianshu.com/p/3161784e7573)
>- [译】Memory Reordering Caught in the Act](https://www.jianshu.com/p/5b317882dda6)
