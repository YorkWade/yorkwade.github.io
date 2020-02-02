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

LRU(Least Recently Used) Cache是一种缓存替换算法，如果超过容量则应该把LRU(近期最少使用)的节点删除掉。实现时，需要利用hash表的快速访问的特性，又要有按时间排序经常增删节点（不易使用连续内存的数组）的双链表。

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
```




### 参考：

- [leveldb中的LRUCache设计](https://bean-li.github.io/leveldb-LRUCache/)



