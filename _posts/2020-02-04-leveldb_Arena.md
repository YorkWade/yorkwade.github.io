---
layout:     post   				    # 使用的布局（不需要改）
title:      leveldb之Arena 		   #  标题
subtitle:   内存分配器               #
date:       2020-02-04 				# 时间
author:     yorkwade				# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - leveldb
---

## 概述

LevelDB里面为了防止出现内存分配的碎片，采用了自己写的一套内存分配管理器，名称为Arena。他不是我们整个leveldb统一使用的传统意义上的内存分配器，主要是为skiplist也就是memtable服务。skiplist里面记录的是用户传进来的key/value，这些字符串有长有短，放到内存中的时候，很容易导致内存碎片。所以这里写了一个统一的内存管理器，其内部还是调用new。skiplist/memtable要申请内存的时候，就利用Arena分配器来分配内存。当skiplist/memtable要释放的时候，就直接通过Arena类的block_把所有申请的内存释放掉。




### 实现要点

- 如何管理内存块
- 何时申请更大的内存
- 如何字节对齐
- 多线程如何分配内存 （为MemTable独享，不涉及多线程及数据共享，所以读写均使用了不需要内存屏障的版本）

## 接口和属性


```objc
class Arena {
 public:
  Arena();
  ~Arena();

  // Return a pointer to a newly allocated memory block of "bytes" bytes.
  char* Allocate(size_t bytes);             //分配内存

  // Allocate memory with the normal alignment guarantees provided by malloc
  char* AllocateAligned(size_t bytes);      //分配对齐的内存

  // Returns an estimate of the total memory usage of data allocated
  // by the arena (including space allocated but not yet used for user
  // allocations).
  // 统计所有由Arena生成的内存总数
  // 这里面可能包含一些内存碎片
  // 所以返回的是近似值
  size_t MemoryUsage() const {
    return blocks_memory_ + blocks_.capacity() * sizeof(char*);
  }

 private:
  char* AllocateFallback(size_t bytes);
  char* AllocateNewBlock(size_t block_bytes);

  // 当前申请的内存块的指针
  // Allocation state
  char* alloc_ptr_;
  size_t alloc_bytes_remaining_;            // 剩余的bytes数

  // 记录所有已经分配的内存块
  // Array of new[] allocated memory blocks
  std::vector<char*> blocks_;


  // Bytes of memory in blocks allocated so far
  size_t blocks_memory_;

  // No copying allowed
  Arena(const Arena&);
  void operator=(const Arena&);
};
```

![](https://i.imgur.com/ZB75F7t.png)

### 分配内存
```objc
char* Arena::AllocateNewBlock(size_t block_bytes) {
  char* result = new char[block_bytes];
  blocks_.push_back(result);
  memory_usage_.NoBarrier_Store(//内存屏障
      reinterpret_cast<void*>(MemoryUsage() + block_bytes + sizeof(char*)));
  return result;
}

```
内存屏障

CPU可以保证指针操作的原子性，但编译器、CPU指令优化--重排序(reorder)可能导致指令乱序，在多线程情况下程序运行结果不符合预期。关于重排序说明如下:
- 单核单线程时，重排序保证单核单线程下程序运行结果一致。
- 单核多线程时，编译器reorder可能导致运行结果不一致。参见《memory-ordering-at-compile-time》。
- 多核多线程时，编译器reorder、CPU reorder将导致运行结果不一致。参见《memory-reordering-caught-in-the-act》。
避免编译器Reorder通常的做法是引入Compiler Barrier(或称之为Memory Barrier)，避免CPU Reorder通常的做法是引入CPU Barrier(或称之为Full Memory Barrier)。
X86 CPU的赋值(Store)和读取(Load)操作天然可以做到无锁。那memory barrier这个名词是哪里蹦出来的呢? Load是原子性操作, CPU不会Load流程走到一半, 就切换到另一个线程去了, 也就是Load本身是不会在多线程环境下产生问题的. 真正导致问题的是做这个操作的时机不确定!
- 1. 编译器有可能让指令乱序, 比如, int a=b; long c=b; 编译器一旦判定a和c没有依赖性, 就有权力让这两个取值操作以任意顺序执行. 因为有可能有CPU指令可以一下取4个int, 乱序可以凑个整.
- 2. CPU会让指令乱序, 原因同上, 但额外还有个原因是分支预测. AB线程都读写一个中间量c, B在处理c, 你预期B好了, A才会取. 但万一A分支预测成功, B在处理的时候, A已经提前Load c进寄存器, 这就没得玩了...
所以, 必须要有指令告诉CPU和编译器, 不要改变这个变量的存取顺序. 这就是Memory Barrier了. call MemoryBarrier保证前后一行是严格按照代码顺序的. 




```objc
char* Arena::AllocateFallback(size_t bytes) {
  if (bytes > kBlockSize / 4) { //如果size不大于kBlockSize的四分之一，就在当前空闲的内存block 中分配，否则，直接向系统申请
    // Object is more than a quarter of our block size.  Allocate it separately
    // to avoid wasting too much space in leftover bytes.
    char* result = AllocateNewBlock(bytes);
    return result;
  }
  // We waste the remaining space in the current block.
  alloc_ptr_ = AllocateNewBlock(kBlockSize);
  alloc_bytes_remaining_ = kBlockSize;

  char* result = alloc_ptr_;
  alloc_ptr_ += bytes;
  alloc_bytes_remaining_ -= bytes;
  return result;
```


```objc
char* Arena::AllocateNewBlock(size_t block_bytes) {
  char* result = new char[block_bytes];
  blocks_memory_ += block_bytes;
  blocks_.push_back(result);
  return result;
}
```
### 分配对齐内存

```objc
char* Arena::AllocateAligned(size_t bytes) {
  const int align = (sizeof(void*) > 8) ? sizeof(void*) : 8;//在32位系统下是4 ,64位系统下是8 
  assert((align & (align-1)) == 0);   // Pointer size should be a power of 2
  size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align-1);//当前指针模4的值,这种取模方法属于牛人技巧
  size_t slop = (current_mod == 0 ? 0 : align - current_mod);//还差 slop = align - current_mod多个字节，内存才是对其
  size_t needed = bytes + slop;//内存对齐
  char* result;
  if (needed <= alloc_bytes_remaining_) {
    result = alloc_ptr_ + slop;
    alloc_ptr_ += needed;
    alloc_bytes_remaining_ -= needed;
  } else {
    // AllocateFallback always returned aligned memory
    result = AllocateFallback(bytes);
  }
  assert((reinterpret_cast<uintptr_t>(result) & (align-1)) == 0);
  return result;
}
```



### 参考：

- [LevelDB源码解析27. Arena内存分配器](https://zhuanlan.zhihu.com/p/45843295)
- [LevelDB源码剖析之Arena内存管理](http://mingxinglai.com/cn/2013/01/leveldb-arena/)
- [深入浅出leveldb之MemTable](https://blog.csdn.net/weixin_42663840/article/details/82263358)
