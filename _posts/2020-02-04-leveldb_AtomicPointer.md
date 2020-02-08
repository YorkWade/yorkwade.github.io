---
layout:     post
title:      leveldb之AtomicPointer
subtitle:   内存屏障
date:       2020-02-04
author:     BY
header-img: img/post-bg-mma-2.jpg
catalog: true
tags:
    - leveldb
---

AtomicPointer 是 leveldb 提供的一个原子指针操作类，使用了基于内存屏障的同步访问机制，这比用锁和信号量的效率要高。


## 理论

内存屏障是一种无锁编程的方法，类似于锁和信号量，但是它有更高的效率。

无锁编程的概念做一般应用层开发的会较少接触到，因为多线程的时候对共享资源的操作一般是用锁来完成的。锁本身对这个任务完成的很好，但是存在性能的问题，也就是在对性能要求很高的，高并发的场景下，锁会带来性能瓶颈。所以在一些如数据库这样的应用或者linux 内核里经常会看到一些无锁的并发编程。

锁是一个高层次的接口，隐藏了很多并发编程时会出现的非常古怪的问题。当不用锁的时候，就要考虑这些问题。主要有两个方面的影响：编译器对指令的排序和cpu对指令的排序。他们排序的目的主要是优化和提高效率。排序的原则是在单核单线程下最终的效果不会发生改变。单核多线程的时候，编译器的乱序就会带来问题，多核的时候，又会涉及cpu对指令的乱序。memory-ordering-at-compile-time和memory-reordering-caught-in-the-act里提到了乱序导致的问题。

如果不使用任何同步机制（例如 mutex 或 atomic），在多线程中读写同一个变量，那么，程序的结果是难以预料的。简单来说，编译器以及 CPU 的一些行为，会影响到程序的执行结果：

-    即使是简单的语句，C++ 也不保证是原子操作。
-    CPU 可能会调整指令的执行顺序。
-    在 CPU cache 的影响下，一个 CPU 执行了某个指令，不会立即被其它 CPU 看见。

　　原子操作说的是，一个操作的状态要么就是未执行，要么就是已完成，不会看见中间状态。例如，在 C++11 中，下面程序的结果是未定义的：
```objc
          int64_t i = 0;     // global variable
Thread-1:              Thread-2:
i = 100;               std::cout << i;
```
C++ 并不保证i = 100是原子操作，因为在某些 CPU Architecture 中，写入int64_t需要两个 CPU 指令，所以 Thread-2 可能会读取到i在赋值过程的中间状态。
另一方面，为了优化程序的执行性能，CPU 可能会调整指令的执行顺序。为阐述这一点，下面的例子中，让我们假设所有操作都是原子操作：
```objc
          int x = 0;     // global variable
          int y = 0;     // global variable
		  
Thread-1:              Thread-2:
x = 100;               while (y != 200)
y = 200;                   ;
                       std::cout << x;
```
如果 CPU 没有乱序执行指令，那么 Thread-2 将输出100。然而，对于 Thread-1 来说，x = 100;和y = 200;这两个语句之间没有依赖关系，因此，Thread-1 允许调整语句的执行顺序：

```objc
Thread-1:
y = 200;
x = 100;
```
在这种情况下，Thread-2 将输出0或100。
CPU cache 也会影响到程序的行为。下面的例子中，假设从时间上来讲，A 操作先于 B 操作发生：
```objc
          int x = 0;     // global variable
		  
Thread-1:                      Thread-2:
x = 100;    // A               std::cout << x;    // B
```
尽管从时间上来讲，A 先于 B，但 CPU cache 的影响下，Thread-2 不能保证立即看到 A 操作的结果，所以 Thread-2 可能输出0或100。
从上面的三个例子可以看到，多线程读写同一变量需要使用同步机制，最常见的同步机制就是std::mutex和内存屏障（MemoryBarrier）/std::atomic。然而，从性能角度看，通常使用后者会获得更好的性能。

同样的，为了解决这样的问题，语言上有一些语句提供屏障的效果，保证屏障前后指令执行的顺序性。而且，庆幸的是，一般，能保证cpu内存屏障的语句也会自动保证编译器级的屏障。注意，不同的cpu的内存模型（即对内存中的指令的执行顺序如何进行的模型）是不一样的，很辛运的，x86/64是的内存模型是强内存模型，它对cpu的乱序执行的影响是最小的。

A strong hardware memory model is one in which every machine instruction comes implicitly withacquire and release semantics. As a result, when one CPU core performs a sequence of writes, every other CPU core sees those values change in the same order that they were written.

因此在x86/64上可以不用考虑cpu的内存屏障，只需要在必要的时候考虑编译器的乱序问题即可。

CPU可以保证**指针**操作的原子性，但编译器、CPU指令优化--重排序(reorder)可能导致指令乱序，在多线程情况下程序运行结果不符合预期。关于重排序说明如下:
- 单核单线程时，重排序保证单核单线程下程序运行结果一致。
- 单核多线程时，编译器reorder可能导致运行结果不一致。参见[《memory-ordering-at-compile-time》](https://preshing.com/20120625/memory-ordering-at-compile-time/)。
- 多核多线程时，编译器reorder、CPU reorder将导致运行结果不一致。参见[《memory-reordering-caught-in-the-act》](https://www.jianshu.com/p/5b317882dda6)。[【译】Memory Reordering Caught in the Act](https://www.jianshu.com/p/5b317882dda6)

简单来说，在不跨越cacheline情况下，Intel处理器保证指针操作的原子性；跨域cacheline情况下，部分处理器提供了原子保证。在通常情况下，C++ new出来的指针及对象内部数据都是cacheline对其的，但如果使用 align 1 byte或者采用c++ placement new等特性时可能出现指针对象跨越cacheline的情况。
在LevelDB中，指针操作是cacheline对齐的，因此问题一种NoBarrier_*的指针操作本身是原子的。




## 实现要点

- **问**：代码中NoBarrier_Store/NoBarrier_Load操作只是最简单的指针操作，那么这些操作是原子的么？
  **答**：在不跨越cacheline情况下，Intel处理器保证指针操作的原子性；跨域cacheline情况下，部分处理器提供了原子保证。LevelDB场景下不存在跨cacheline场景，因此这部分操作是原子的。
- **问**：Acquire_Load/Release_Store操作增加了MemoryBarrier操作，其作用是什么？又如何保证原子性呢？
  **答**：增加Memory Barrier是为了避免编译器重排序，保证MemoryBarrier前的全部操作真正在Memory Barrier前执行。
- **问**：为何要设计这样两组操作？
  **答**：性能。NoBarrier_Store/NoBarrier_Load的性能要优于Acquire_Load/Release_Store，但Acquire_Load/Release_Store可以避免编译器优化，由此保证load/store时指针里面的数据一定是最新的。
- **问**：LevelDB代码中如何选择何时使用何种操作？
  **答**：时刻小心。在任意一个用到指针的场景，结合上下文+并发考量选择合适的load/store方法。当然，一个比较保守的做法是，所有的场景下都使用带Memory Barrier的load/store方法，仅当确定可以使用NoBarrier的load/store方法才将其替换掉。


## 源码实现
windows版本：
```objc
// Storage for a lock-free pointer
class AtomicPointer {
 private:
  std::atomic<void *> rep_;
 public:
  AtomicPointer() { }
  explicit AtomicPointer(void* v) : rep_(v) { }
  inline void* Acquire_Load() const {
    return rep_.load(std::memory_order_acquire);
  }
  inline void Release_Store(void* v) {
    rep_.store(v, std::memory_order_release);
  }
  inline void* NoBarrier_Load() const {
    return rep_.load(std::memory_order_relaxed);
  }
  inline void NoBarrier_Store(void* v) {
    rep_.store(v, std::memory_order_relaxed);
  }
};
```
std::atomic是C++11提供的原子模板类，可实现数据同步的无锁设计。
在这种模型下，store()使用memory_order_release，而load()使用memory_order_acquire。这种模型有两种效果，第一种是可以限制 CPU 指令的重排：
- 在store()之前的所有读写操作，不允许被移动到这个store()的后面。
- 在load()之后的所有读写操作，不允许被移动到这个load()的前面。
除此之外，还有另一种效果：假设 Thread-1 store()的那个值，成功被 Thread-2 load()到了，那么 Thread-1 在store()之前对内存的所有写入操作，此时对 Thread-2 来说，都是可见的。

linux版本：MemoryBarrier函数实现是inline且嵌入一条汇编指令__asm__ __volatile__(“” : : : “memory”);，__volatile__表示阻止编译器对该值进行优化，强制变量使用精确内存地址（非 cache或register），memory表示对内存有修改操作，需要重新读入，该指令能够阻止编译器乱序，但不能阻止CPU乱序执行（在SMP体系下）
```objc
inline void MemoryBarrier() {
  // Seehttp://gcc.gnu.org/ml/gcc/2003-04/msg01180.html for a discussion on
  // this idiom. Also seehttp://en.wikipedia.org/wiki/Memory_ordering.
  __asm__ __volatile__("": : : "memory");
}

class AtomicPointer {
 private:
  void* rep_;
 public:
  AtomicPointer() { }
  explicitAtomicPointer(void* p) : rep_(p) {}
  inlinevoid* NoBarrier_Load()const { return rep_; }
  inlinevoid NoBarrier_Store(void* v) { rep_ = v; }
  inlinevoid* Acquire_Load()const {
    void* result = rep_;
    MemoryBarrier();
    returnresult;
  }
  inlinevoid Release_Store(void* v) {
    MemoryBarrier();
    rep_ = v;
  }
};
```





>参考 

>- [LevelDB : AtomicPointer](http://www.voidcn.com/article/p-poodsusd-sm.html)
>- [《Memory Ordering at Compile Time》](https://preshing.com/20120625/memory-ordering-at-compile-time/)
>- [LevelDB源码剖析之基础部件-AtomicPointer](https://www.jianshu.com/p/3161784e7573)
>- [【译】Memory Reordering Caught in the Act](https://www.jianshu.com/p/5b317882dda6)
>- [leveldb 源码分析(三) – Write](https://youjiali1995.github.io/storage/leveldb-write/)
>- [理解 C++ 的 Memory Order](https://senlinzhan.github.io/2017/12/04/cpp-memory-order/)



