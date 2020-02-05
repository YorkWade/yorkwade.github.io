---
layout:     post
title:      leveldb之SkipList
subtitle:   媲美红黑树
date:       2020-02-05
author:     BY
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - leveldb
---



![](http://mjrnxewya3t1in23ybpwjw59.wpengine.netdna-cdn.com/wp-content/uploads/buchecha-marcus-almeida-roger-gracie.jpg)


SkipList称之为跳表，可实现O(lgN)级别的插入、删除。和Map、set等典型的红黑树数据结构相比，且实现简单，其问题在于性能与插入数据的随机性有关。

由William Pugh于1990年在在 Communications of the ACM June 1990, 33(6) 668-676 发表了[Skip lists: a probabilistic alternative to balanced trees](https://www.cl.cam.ac.uk/teaching/0506/Algorithms/skiplists.pdf)）  



### 理论背景

我们都知道，AVL树有着严格的O(logN)的查询效率，但是由于插入过程中可能需要多次旋转，导致插入效率较低，因而才有了在工程界更加实用的红黑树。

但是红黑树有一个问题就是在并发环境下使用不方便，比如需要更新数据时，Skip需要更新的部分比较少，锁的东西也更少，而红黑树有个平衡的过程，在这个过程中会涉及到较多的节点，需要锁住更多的节点，从而降低了并发性能。
SkipList实现简单。

单链表

![](http://igoro.com/wordpress/wp-content/uploads/2008/07/list.png)

在单向链表中进行查找的时间复杂度是O(N)。此外，由于插入、删除操作首先都要使用查找操作，因此如何提高查找的性能成为关键问题。
一种改善查找性能的方法，建立多级链表

![](http://igoro.com/wordpress/wp-content/uploads/2008/07/multilist.png)

那么查找7的过程是这样的（红色箭头）：

![](http://igoro.com/wordpress/wp-content/uploads/2008/07/multilist-search.png)





### 实现要点

- 以怎样的概率分层

William Pugh的论文中描述：
Choosing a Random LevelInitially, we discussed a probability distribution where half ofthe nodes that have level i pointers also have level i+1 point-ers. To get away from magic constants, we say that a fractionp of the nodes with level i pointers also have level i+1 point-ers. (for our original discussion, p = 1/2). Levels are generatedrandomly by an algorithm equivalent to the one in Figure 5.Levels are generated without reference to the number of ele-ments in the list.

```objc
randomLevel()
    lvl := 1
    -- random() that returns a random value in [0...1)
    while random() < p and lvl < MaxLevel do
        lvl := lvl + 1
    return lvl
```


## 源码分析



- [LevelDB : SkipList](https://huntinux.github.io/leveldb-skiplist.html)
- [LevelDB源码剖析之Arena内存管理](http://mingxinglai.com/cn/2013/01/leveldb-arena/)
