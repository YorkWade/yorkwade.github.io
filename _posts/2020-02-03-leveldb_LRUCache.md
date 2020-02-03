---
layout:     post
title:      leveldb之LRUCache
subtitle:   LRUCache分析
date:       2020-02-03
author:     BY
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - iOS
---


## 前言

LRU(Least Recently Used) Cache是一种缓存替换算法，如果超过容量则应该“近期最少使用”的原则更新缓存内容。实现时，需要利用hash表的快速访问的特性，又要有按时间排序经常增删节点（不易使用连续内存的数组）的双链表。还需要考虑，在多线程访问的情况下，对内存的释放。

![](https://cloud.githubusercontent.com/assets/1736354/6984935/92033a96-da60-11e4-8754-66135bb0d233.png)

## 一个C++实现


```objc
    /*************************************************************************
        > File Name: lru_cache_template.hpp
        > Author: ce39906
        > Mail: ce39906@163.com
        > Created Time: 2018-04-02 16:44:58
     ************************************************************************/
    #ifndef LRU_CACHE_TEMPLATE_HPP
    #define LRU_CACHE_TEMPLATE_HPP
    #include <unordered_map>
    template <class Key,class Value>
    struct Node
    {
        Node(Key k,Value v) :
            key(k), value(v), prev(NULL), next(NULL)
        {
        }
        Node()
        {
        }
        Key key;
        Value value;
        Node* prev;
        Node* next;
    };
    template <class Key, class Value>
    class LruCache
    {
      public:
        LruCache(int capacity) :
            size_(0), capacity_(capacity)
        {
            head = new Node<Key,Value>;
            tail = new Node<Key,Value>;
            head->next = tail;
            head->prev = NULL;
            tail->next = NULL;
            tail->prev = head;
            container_.clear();
        };
        ~LruCache()
        {
            while(head)
            {
                Node<Key,Value>* temp = head->next;
                delete head;
                head = temp;
            }
        };
        void Put(Key key ,Value value)
        {
            //insert
            if (container_.find(key) == container_.end())
            {
                //not full
                if (size_ != capacity_)
                {
                    Node<Key,Value>* data = new Node<Key,Value>(key,value);
                    attach(data);
                    container_.insert(std::make_pair(key,data));
                    size_++;
                }
                else
                {
                    Node<Key,Value>* temp = tail->prev;
                    container_.erase(temp->key);
                    detach(temp);
                    if (temp)
                        delete temp;
                    Node<Key,Value>* data = new Node<Key,Value>(key,value);
                    attach(data);
                    container_.insert(std::make_pair(key,data));
                }
            }
            else //update
            {
                Node<Key,Value>* data = container_[key];
                detach(data);
                if (data)
                    delete data;
                data = new Node<Key,Value>(key,value);
                container_[key] = data;
                attach(data);
            }
        }
        Value Get(Key key)
        {
            //find
            if (container_.find(key) != container_.end())
            {
                Node<Key,Value>* data = container_[key];
                detach(data);
                attach(data);
                return data->value;
            }
            else // not find
            {
                return Value();
            }
        }
      private:
        int size_;
        int capacity_;
        std::unordered_map<Key,Node<Key,Value>*> container_;
        Node<Key,Value>* head;
        Node<Key,Value>* tail;
        void attach(Node<Key,Value>* node)
        {
            Node<Key,Value>* temp = head->next;
            head->next = node;
            node->next = temp;
            node->prev = head;
            temp->prev = node;
        }
        void detach(Node<Key,Value>* node)
        {
            node->prev->next = node->next;
            node->next->prev = node->prev;
        }
    };
    #endif
```

### leveldb 中的LRUCache

leveldb中主要涉及4个数据结构，是依次递进的关系，分别是：

- LRUHandle        //链表
- HandleTable      //哈希表
- LRUCache         //LRU缓存
- ShardedLRUCache  //LRU缓存分组

看到leveldb的这几个类名，我有一种骂娘的感觉，LRUHandle应该为LRUNode，HandleTable应该命名为HanshTable。也许没真正理解作者的逼格。

### LRUHandle
```objc
    struct LRUHandle {
      void* value;
      void (*deleter)(const Slice&, void* value);
      LRUHandle* next_hash;
      LRUHandle* next;
      LRUHandle* prev;
      size_t charge;      // TODO(opt): Only allow uint32_t?
      size_t key_length;
      bool in_cache;      // Whether entry is in the cache.
      uint32_t refs;      // References, including cache reference, if present.
      uint32_t hash;      // Hash of key(); used for fast sharding and comparisons
      char key_data[1];   // Beginning of key

      Slice key() const {
        // For cheaper lookups, we allow a temporary Handle object
        // to store a pointer to a key in "value".
        if (next == this) {
          return *(reinterpret_cast<Slice*>(value));
        } else {
          return Slice(key_data, key_length);
        }
      }
    };
```
### HandleTable
作者自己实现Hashmap，采用拉链法实现，也就是在冲突发生时，需要使用链表来解决冲突问题。工业级的hash表需要考虑扩容。hash函数使用取模方法，但是取模的实现上，比较反人类，linux的无锁队列中，也有类似实现，这可能就是大牛的不同之处吧。

```objc
class HandleTable {
     public:

     ...

     private:

      uint32_t length_;             //纪录的就是当前hash桶的个数
      uint32_t elems_;              //整个hash表中一共存放了多少个元素
      LRUHandle** list_;            //二维指针，每一个指针指向一个桶的表头位置
     ...
    };
```

```objc
LRUHandle* Insert(LRUHandle* h) {
    LRUHandle** ptr = FindPointer(h->key(), h->hash);//在hash表中查找key
    LRUHandle* old = *ptr;      //老的元素返回，LRUCache会将相同key的老元素释放，详情看LRUCache的Insert函数。
    h->next_hash = (old == NULL ? NULL : old->next_hash);
    *ptr = h;
    if (old == NULL) {          //如果hash表中没有该值
      ++elems_;
      if (elems_ > length_) {
        // Since each cache entry is fairly large, we aim for a small
        // average linked list length (<= 1).
        Resize();               //扩容
      }
    }
    return old;
  }
```

```objc
  // Return a pointer to slot that points to a cache entry that
  // matches key/hash.  If there is no such cache entry, return a
  // pointer to the trailing slot in the corresponding linked list.
  LRUHandle** FindPointer(const Slice& key, uint32_t hash) {
    LRUHandle** ptr = &list_[hash & (length_ - 1)];             //变态，hash & (length_ - 1)等于hash%length
    while (*ptr != NULL &&
           ((*ptr)->hash != hash || key != (*ptr)->key())) {    //如果多个链表值，遍历链表查找
      ptr = &(*ptr)->next_hash;
    }
    return ptr;
  }
```
### LRUCache
有了链表和哈希表，就要看LRUCache如何实现“近期最少使用”功能了。

```objc
// A single shard of sharded cache.
class LRUCache {
    ...
  // Initialized before use.
  size_t capacity_;

  // mutex_ protects the following state.
  port::Mutex mutex_;
  size_t usage_;            //LRU链表已用长度，就是用来计算删除和替换LRU队列的。

  // Dummy head of LRU list.
  // lru.prev is newest entry, lru.next is oldest entry.
  LRUHandle lru_;           //双向链表

  HandleTable table_;       //哈希表
  
  ...
};
```

```objc
// A single shard of sharded cache.

Cache::Handle* LRUCache::Insert(
    const Slice& key, uint32_t hash, void* value, size_t charge,
    void (*deleter)(const Slice& key, void* value)) {
  MutexLock l(&mutex_);

  LRUHandle* e = reinterpret_cast<LRUHandle*>(
      malloc(sizeof(LRUHandle)-1 + key.size()));
  e->value = value;
  e->deleter = deleter;
  e->charge = charge;
  e->key_length = key.size();
  e->hash = hash;
  e->refs = 2;  // One from LRUCache, one for the returned handle
  memcpy(e->key_data, key.data(), key.size());
  LRU_Append(e);            //将最新值放入LRU队列最前端
  usage_ += charge;         //已用数值增加

  LRUHandle* old = table_.Insert(e);
  if (old != NULL) {        //如果哈希表中存在已有值。
    LRU_Remove(old);        //将LRU队列中已存在的值删除
    Unref(old);             //引用计数减一
  }

  while (usage_ > capacity_ && lru_.next != &lru_) {    //如果已用数值大于队列容量
    LRUHandle* old = lru_.next;                         
    LRU_Remove(old);                                    //删除最旧值
    table_.Remove(old->key(), old->hash);               //哈希表中相应删除
    Unref(old);
  }

  return reinterpret_cast<Cache::Handle*>(e);
}
```

```objc
// A single shard of sharded cache.
当时leveldb编写比较早，还没有智能指针，这部分可以使用share_ptr代替
void LRUCache::Unref(LRUHandle* e) {
  assert(e->refs > 0);
  e->refs--;
  if (e->refs <= 0) {                   //当计数为0时
    usage_ -= e->charge;
    (*e->deleter)(e->key(), e->value);  //删除节点
    free(e);
  }
}
```
查找和删除函数，水到渠成。
```objc
Cache::Handle* LRUCache::Lookup(const Slice& key, uint32_t hash) {
  MutexLock l(&mutex_);
  LRUHandle* e = table_.Lookup(key, hash);
  if (e != NULL) {
    e->refs++;
    LRU_Remove(e);
    LRU_Append(e);
  }
  return reinterpret_cast<Cache::Handle*>(e);
}
```

```objc
void LRUCache::Erase(const Slice& key, uint32_t hash) {
  MutexLock l(&mutex_);
  LRUHandle* e = table_.Remove(key, hash);
  if (e != NULL) {
    LRU_Remove(e);
    Unref(e);
  }
}
```

### LRUHandle

ShardedLRUCache类，实际上到S3，一个标准的LRU Cache已经实现了，为何还要更近一步呢？答案就是速度！
为了多线程访问，尽可能快速，减少锁开销，ShardedLRUCache内部有16个LRUCache，查找Key时首先计算key属于哪一个分片，分片的计算方法是取32位hash值的高4位，然后在相应的LRUCache中进行查找，这样就大大减少了多线程的访问锁的开销。它就是一个包装类，实现都在LRUCache类中。


```objc
static const int kNumShardBits = 4;
static const int kNumShards = 1 << kNumShardBits;

class ShardedLRUCache : public Cache {
 private:
  LRUCache shard_[kNumShards];          //LRUCache数组
  port::Mutex id_mutex_;
  uint64_t last_id_;

  static inline uint32_t HashSlice(const Slice& s) {
    return Hash(s.data(), s.size(), 0);
  }

  static uint32_t Shard(uint32_t hash) {
    return hash >> (32 - kNumShardBits);
  }

 public:
  explicit ShardedLRUCache(size_t capacity)
      : last_id_(0) {
    const size_t per_shard = (capacity + (kNumShards - 1)) / kNumShards;
    for (int s = 0; s < kNumShards; s++) {
      shard_[s].SetCapacity(per_shard);
    }
  }
  virtual ~ShardedLRUCache() { }
  virtual Handle* Insert(const Slice& key, void* value, size_t charge,
                         void (*deleter)(const Slice& key, void* value)) {
    const uint32_t hash = HashSlice(key);
    return shard_[Shard(hash)].Insert(key, hash, value, charge, deleter);
  }
  virtual Handle* Lookup(const Slice& key) {
    const uint32_t hash = HashSlice(key);
    return shard_[Shard(hash)].Lookup(key, hash);
  }
  virtual void Release(Handle* handle) {
    LRUHandle* h = reinterpret_cast<LRUHandle*>(handle);
    shard_[Shard(h->hash)].Release(handle);
  }
  virtual void Erase(const Slice& key) {
    const uint32_t hash = HashSlice(key);
    shard_[Shard(hash)].Erase(key, hash);
  }
  virtual void* Value(Handle* handle) {
    return reinterpret_cast<LRUHandle*>(handle)->value;
  }
  virtual uint64_t NewId() {
    MutexLock l(&id_mutex_);
    return ++(last_id_);
  }
};
```

### 参考：

- [leveldb中的LRUCache设计](https://bean-li.github.io/leveldb-LRUCache/)
- [leveldb源码分析1](https://blog.csdn.net/sparkliang/article/details/8567602)



