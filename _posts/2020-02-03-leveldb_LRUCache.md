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

    LRUHandle        //链表
    HandleTable      //哈希表
    LRUCache         //LRU缓存
    ShardedLRUCache  //LRU缓存分组


```
- (void)setTarget:(NSString *)target {
    if (target == _target) return;
    id pre = _target;
    [target retain];//1.先保留新值
    _target = target;//2.再进行赋值
    [pre release];//3.释放旧值
}
```

因为在 *并行队列* `DISPATCH_QUEUE_CONCURRENT` 中*异步* `dispatch_async` 对 `target`属性进行赋值，就会导致 target 已经被 `release`了，还会执行 `release`。这就是向已释放内存对象发送消息而发生 crash 。


### 但是

我敲了这段代码，执行的时候发现并不会 crash~

```objc
@property (nonatomic, strong) NSString *target;
dispatch_queue_t queue = dispatch_queue_create("parallel", DISPATCH_QUEUE_CONCURRENT);
for (int i = 0; i < 1000000 ; i++) {
    dispatch_async(queue, ^{
    	// ‘ksddkjalkjd’删除了
        self.target = [NSString stringWithFormat:@"%d",i];
    });
}
```

原因就出在对 `self.target` 赋值的字符串上。博客的最后也提到了 - *‘上述代码的字符串改短一些，就不会崩溃’*，还有 `Tagged Pointer` 这个东西。

我们将上面的代码修改下：


```objc
NSString *str = [NSString stringWithFormat:@"%d", i];
NSLog(@"%d, %s, %p", i, object_getClassName(str), str);
self.target = str;
```

输出：

```
0, NSTaggedPointerString, 0x3015
```

发现这个字符串类型是 `NSTaggedPointerString`，那我们来看看 Tagged Pointer 是什么？

### Tagged Pointer

Tagged Pointer 详细的内容可以看这里 [深入理解Tagged Pointer](http://www.infoq.com/cn/articles/deep-understanding-of-tagged-pointer)。

Tagged Pointer 是一个能够提升性能、节省内存的有趣的技术。

- Tagged Pointer 专门用来存储小的对象，例如 **NSNumber** 和 **NSDate**（后来可以存储小字符串）
- Tagged Pointer 指针的值不再是地址了，而是真正的值。所以，实际上它不再是一个对象了，它只是一个披着对象皮的普通变量而已。
- 它的内存并不存储在堆中，也不需要 malloc 和 free，所以拥有极快的读取和创建速度。




### 参考：

- [从一道网易面试题浅谈OC线程安全](https://www.jianshu.com/p/cec2a41aa0e7)

- [深入理解Tagged Pointer](http://www.infoq.com/cn/articles/deep-understanding-of-tagged-pointer)

- [【译】采用Tagged Pointer的字符串](http://www.cocoachina.com/ios/20150918/13449.html)

