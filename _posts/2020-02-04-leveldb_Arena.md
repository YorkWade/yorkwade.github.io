---
layout:     post   				    # 使用的布局（不需要改）
title:      My First Post 				# leveldb之Arena 
subtitle:   Hello World, Hello Blog #内存分配器
date:       2020-02-04 				# 时间
author:     BY 						# 作者
header-img: img/post-bg-2015.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - 生活
---

## 概述

LevelDB里面为了防止出现内存分配的碎片，采用了自己写的一套内存分配管理器，名称为Arena。他不是我们整个leveldb统一使用的传统意义上的内存分配器，主要是为skiplist也就是memtable服务。skiplist里面记录的是用户传进来的key/value，这些字符串有长有短，放到内存中的时候，很容易导致内存碎片。所以这里写了一个统一的内存管理器，其内部还是调用new。skiplist/memtable要申请内存的时候，就利用Arena分配器来分配内存。当skiplist/memtable要释放的时候，就直接通过Arena类的block_把所有申请的内存释放掉。

![](https://cloud.githubusercontent.com/assets/1736354/6984935/92033a96-da60-11e4-8754-66135bb0d233.png)


### 需求分析

leveldb中主要涉及4个数据结构，是依次递进的关系，分别是：

- LRUHandle        //链表
- HandleTable      //哈希表
- LRUCache         //LRU缓存
- ShardedLRUCache  //LRU缓存分组

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


### 申请新快
```objc
char* Arena::AllocateNewBlock(size_t block_bytes) {
  char* result = new char[block_bytes];
  blocks_.push_back(result);
  memory_usage_.NoBarrier_Store(
      reinterpret_cast<void*>(MemoryUsage() + block_bytes + sizeof(char*)));
  return result;
}

```

```objc
char* Arena::AllocateFallback(size_t bytes) {
  if (bytes > kBlockSize / 4) {
    // Object is more than a quarter of our block size.  Allocate it separately
    // to avoid wasting too much space in leftover bytes.
    char* result = AllocateNewBlock(bytes);
    return result;
  }
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
  const int align = (sizeof(void*) > 8) ? sizeof(void*) : 8;
  assert((align & (align-1)) == 0);   // Pointer size should be a power of 2
  size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align-1);
  size_t slop = (current_mod == 0 ? 0 : align - current_mod);
  size_t needed = bytes + slop;
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
