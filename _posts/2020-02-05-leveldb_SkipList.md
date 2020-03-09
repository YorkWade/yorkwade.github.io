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




SkipList称之为跳表，可实现O(lgN)级别的插入、删除。和Map、set等典型的红黑树数据结构相比，且实现简单，其问题在于性能与插入数据的随机性有关。并发更容易，涉及的节点少。
 



### 理论背景


由William Pugh于1990年在在 Communications of the ACM June 1990, 33(6) 668-676 发表了[Skip lists: a probabilistic alternative to balanced trees](https://www.cl.cam.ac.uk/teaching/0506/Algorithms/skiplists.pdf)） 

我们都知道，AVL树有着严格的O(logN)的查询效率，但是由于插入过程中可能需要多次旋转，导致插入效率较低，因而才有了在工程界更加实用的红黑树。但是红黑树有一个问题就是在并发环境下使用不方便，比如需要更新数据时，Skip需要更新的部分比较少，锁的东西也更少，而红黑树有个平衡的过程，在这个过程中会涉及到较多的节点，需要锁住更多的节点，从而降低了并发性能。SkipList空间使用率也不错，与平衡树有着相同的复杂度，节省空间，每个检点平均1.33个指针。

SkipList比较突出的优点就是实现简单。

单链表

![](http://igoro.com/wordpress/wp-content/uploads/2008/07/list.png)

在单向链表中进行查找的时间复杂度是O(N)。此外，由于插入、删除操作首先都要使用查找操作，因此如何提高查找的性能成为关键问题。
一种改善查找性能的方法，建立多级链表

![](http://igoro.com/wordpress/wp-content/uploads/2008/07/multilist.png)

那么查找7的过程是这样的（红色箭头）：

![](http://igoro.com/wordpress/wp-content/uploads/2008/07/multilist-search.png)





### 实现要点

- 跳表的关键参数：上下层节点个数比（概率），层数

分层算法决定了数据插入的Level，SkipList的平衡性如何全权由分层算法决定。极端情况下，假设SkipList只有Level-0层，SkipList将弱化成自排序List。此时查找、插入、删除的时间复杂度均为O(n)，而非O(Log(n))。
分层概率和最大层数，为为SkipList的关键参数，与性能直接相关。
William Pugh的论文中描述：
```objc
randomLevel()
    lvl := 1
    -- random() that returns a random value in [0...1)
    while random() < p and lvl < MaxLevel do
        lvl := lvl + 1
    return lvl
```
leveldb中SkipList的概率为1/4，这使得上层节点的数量约为下层的1/4。那么，当设定MaxHeight=12时，根节点为1时，约可均匀容纳Key的数量为4^11= 4194304(约为400W)。MaxHeight=12为经验值，在百万数据规模时，尤为适用。 程序中修改MaxHeight时，在数值变小时，性能上有明显下降，但当数值增大时，甚至增大到10000时，和默认的 MaxHeight= 12相比仍旧无明显差异，内存使用上也是如此

- 并发读写

读值本身并不会改变SkipList的结构，因此多个读之间不存在并发问题。
而当读、写同时存在时，SkipList通过AtomicPointer(原子指针)及结构调整上的小技巧达到“无锁”并发。首先，leveldb的MemTable只做插入不做删除（这由其业务特性决定），节点一旦被添加到SkipList中，其层级结构将不再发生变化，Node中的唯一成员：port::AtomicPointer next_[1] 大小不会再发生改变。

skiplist 只需要支持如下场景的线程安全即可：
-    一写多读：写操作不影响正在进行的读操作。
-    写之后读：写操作之后的读要立刻读到最新的写入。
-    写之后写：写操作之后继续写入要保证线程安全。



## 源码分析

SkipList的定义如下：
```objc
template<typename Key, class Comparator>
class SkipList {
 private:
  struct Node;//节点

 public:
  // Create a new SkipList object that will use "cmp" for comparing keys,
  // and will allocate memory using "*arena".  Objects allocated in the arena
  // must remain allocated for the lifetime of the skiplist object.
  explicit SkipList(Comparator cmp, Arena* arena);

  // Insert key into the list.
  // REQUIRES: nothing that compares equal to key is currently in the list.
  void Insert(const Key& key);

  // Returns true iff an entry that compares equal to key is in the list.
  bool Contains(const Key& key) const;

  // Iteration over the contents of a skip list
 ...省略了Iteraor迭代器的定义

 private:
  enum { kMaxHeight = 12 };//最大高度为12

  // Immutable after construction
  Comparator const compare_;
  Arena* const arena_;    // Arena used for allocations of nodes 

  Node* const head_;

  // Modified only by Insert().  Read racily by readers, but stale
  // values are ok.
  port::AtomicPointer max_height_;   // Height of the entire list

  inline int GetMaxHeight() const {
    return static_cast<int>(
        reinterpret_cast<intptr_t>(max_height_.NoBarrier_Load()));
  }

  // Read/written only by Insert().
  Random rnd_;

  Node* NewNode(const Key& key, int height);
  int RandomHeight();                   //随机生成高度
  bool Equal(const Key& a, const Key& b) const { return (compare_(a, b) == 0); }

  // Return true if key is greater than the data stored in "n"
  bool KeyIsAfterNode(const Key& key, Node* n) const;

  // Return the earliest node that comes at or after key.
  // Return NULL if there is no such node.
  //
  // If prev is non-NULL, fills prev[level] with pointer to previous
  // node at "level" for every level in [0..max_height_-1].
  Node* FindGreaterOrEqual(const Key& key, Node** prev) const;

  // Return the latest node with a key < key.
  // Return head_ if there is no such node.
  Node* FindLessThan(const Key& key) const;

  // Return the last node in the list.
  // Return head_ if list is empty.
  Node* FindLast() const;

  // No copying allowed
  SkipList(const SkipList&);
  void operator=(const SkipList&);
};
```

节点的定义，封装了线程安全的原则操作：
```objc
// Implementation details follow
template<typename Key, class Comparator>
struct SkipList<Key,Comparator>::Node {
  explicit Node(const Key& k) : key(k) { }

  Key const key; // 保存的key

  // Accessors/mutators for links.  Wrapped in methods so we can
  // add the appropriate barriers as necessary.
  Node* Next(int n) {// 获取当前节点在指定level的下一个节点
    assert(n >= 0);
    // Use an 'acquire load' so that we observe a fully initialized
    // version of the returned Node.
    return reinterpret_cast<Node*>(next_[n].Acquire_Load());
  }
  void SetNext(int n, Node* x) {// 将当前节点在指定level的下一个节点设置为x
    assert(n >= 0);
    // Use a 'release store' so that anybody who reads through this
    // pointer observes a fully initialized version of the inserted node.
    next_[n].Release_Store(x);
  }

  // No-barrier variants that can be safely used in a few locations.
  Node* NoBarrier_Next(int n) {// 无内存屏障版本next
    assert(n >= 0);
    return reinterpret_cast<Node*>(next_[n].NoBarrier_Load());
  }
  void NoBarrier_SetNext(int n, Node* x) {// 无内存屏障版本set
    assert(n >= 0);
    next_[n].NoBarrier_Store(x);
  }

 private:
  // Array of length equal to the node height.  next_[0] is lowest level link.
  port::AtomicPointer next_[1]; // 数组的长度就是该节点的高度，next_[0]是最底层的链表
};

```

NewNode是用Arena的内存池分配的。
```objc
template<typename Key, class Comparator>
typename SkipList<Key,Comparator>::Node*
SkipList<Key,Comparator>::NewNode(const Key& key, int height) {
  char* mem = arena_->AllocateAligned(
      sizeof(Node) + sizeof(port::AtomicPointer) * (height - 1));
  return new (mem) Node(key);
}
```

插入操作：
    首先更新插入节点的next指针，此处无并发问题。
    修改插入位置前一节点的next指针，此处采用SetNext处理并发。
    由最下层向上插入可以保证当前层一旦插入后，其下层已更新完毕并可用。
当然，多个写之间的并发SkipList时非线程安全的，在LevelDB的MemTable中采用了另外的技巧来处理写并发问题。
```objc
template<typename Key, class Comparator>
void SkipList<Key,Comparator>::Insert(const Key& key) {
  // TODO(opt): We can use a barrier-free variant of FindGreaterOrEqual()
  // here since Insert() is externally synchronized.
  Node* prev[kMaxHeight];
  Node* x = FindGreaterOrEqual(key, prev); //寻找第一个大于等于 Key 值的节点，同时将该节点的前置节点存放在数组 prev 中

  // Our data structure does not allow duplicate insertion
  assert(x == NULL || !Equal(key, x->key)); // 不允许插入重复数据

  int height = RandomHeight();   // 生成随机的高度height
  if (height > GetMaxHeight()) { // 如果height大于最大高度，将增加层次中的前驱设为head
    for (int i = GetMaxHeight(); i < height; i++) {
      prev[i] = head_;
    }
    //fprintf(stderr, "Change height from %d to %d\n", max_height_, height);

    // It is ok to mutate max_height_ without any synchronization
    // with concurrent readers.  A concurrent reader that observes
    // the new value of max_height_ will see either the old value of
    // new level pointers from head_ (NULL), or a new value set in
    // the loop below.  In the former case the reader will
    // immediately drop to the next level since NULL sorts after all
    // keys.  In the latter case the reader will use the new node.
    // max_height_没有任何并发保护
    // 读线程在读到新的max_height_同时，对应的层级指针(new level pointer from head_)可能是原有的NULL，
    // 也有可能是部分更新的层级指针。如果是前者将直接跳到下一level继续查找，如果是后者，新插入的节点将被启用。
    max_height_.NoBarrier_Store(reinterpret_cast<void*>(height)); // 更新当前高度max_height_
  }

  x = NewNode(key, height);          // 生成一个新节点
  for (int i = 0; i < height; i++) { // 将该节点串接到所有的链表中
    // NoBarrier_SetNext() suffices since we will add a barrier when
    // we publish a pointer to "x" in prev[i].
    x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));
    prev[i]->SetNext(i, x);// 内部使用内存屏障同步操作，在store()之前的所有读写操作，不允许被移动到这个store()的后面，防止重排序
  }
}
```
需要注意的是使用 AtomicPointer 的地方:

-    Skiplist::AtomicPointer max_height_：
-        插入：使用 NoBarrier_Store 设置。
-        查找：使用 NoBarrier_Load 读取。
-    Node::AtomicPointer next_[1]：
-        插入：设置被插入结点的 next_[] 时用的是 NoBarrier_Store，设置 prev 结点的 next_[] 时使用 Release_Store。
-        查找：使用 AcquireLoad 读取 next_[]

成对 Acquire_Load/Release_Store 能够保证在不同线程间立刻可见，且 Release_Store 之前的修改对 Acquire_Load 之后立即可见。因为查找是按照从高层向低层、从小 到大的顺序遍历，而插入的时候是按照从低层到高层、用 Release_Store 设置 prev 结点的 next_[]，确保了当观察到新插入的结点时，后续的遍历一定是个完好的 skiplist。

![](https://youjiali1995.github.io/assets/images/leveldb/skiplist.png)

需要注意一点，max_height_ 只保证了原子性，没有保证对 max_height_ 可见性，也没有保证对 next_[] 的可见性，但都不会影响读的结果。假设插入增大了 max_height_:

-    读操作观察到了 max_height_ 的更新，对应上面两种情况分别是:
-        新增的 level 的 head_ 都指向 NULL，leveldb 保证了 skiplist 中 NULL 是最大的，所以会立刻向下层查找。
-        在某一层观察到了新插入的 key，继续遍历。
-    读操作未观察到 max_height_ 的更新，直接从低层开始遍历不影响 skiplist 的查找。

        



随机生成高度
```objc
template<typename Key, class Comparator>
int SkipList<Key,Comparator>::RandomHeight() {
  // Increase height with probability 1 in kBranching
  static const unsigned int kBranching = 4;
  int height = 1;
  while (height < kMaxHeight && ((rnd_.Next() % kBranching) == 0)) {//随机插入节点层数，底层节点数为上层节点数数的4倍
    height++;
  }
  assert(height > 0);
  assert(height <= kMaxHeight);
  return height;
}
```

可对比看一下redis的实现，也使用了0.25（1/4）的概率：
```objc
#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^32 elements */
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */
/* Returns a random level for the new skiplist node we are going to create.
 * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL
 * (both inclusive), with a powerlaw-alike distribution where higher
 * levels are less likely to be returned. */
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```


接着看FindGreaterOrEqual的实现：
```objc
// Return the earliest node that comes at or after key.
// Return NULL if there is no such node.
//
// If prev is non-NULL, fills prev[level] with pointer to previous
// node at "level" for every level in [0..max_height_-1].
template<typename Key, class Comparator>
typename SkipList<Key,Comparator>::Node* SkipList<Key,Comparator>::FindGreaterOrEqual(const Key& key, Node** prev)
    const {
  Node* x = head_; // x指向head
  int level = GetMaxHeight() - 1; // 获得当前的高度，从最顶层level开始查找
  while (true) {
    Node* next = x->Next(level);  
    if (KeyIsAfterNode(key, next)) { // 要查找的值在前node后面，则继续在该层进行查找
      // Keep searching in this list
      x = next;
    } else { // 要查找的值小于或等于当前node，则继续该该层进行查找
      if (prev != NULL) prev[level] = x; // 记录在该层的前驱
      if (level == 0) { // 所有层次的链表均遍历完，返回该位置。有可能是找到了，也有可能是第一个大于该值的位置
        return next;
      } else {
        // Switch to next list
        level--; // 转到下一层链表
      }
    }
  }
}
```

```objc
template<typename Key, class Comparator>
bool SkipList<Key,Comparator>::Contains(const Key& key) const {
  Node* x = FindGreaterOrEqual(key, NULL);
  if (x != NULL && Equal(key, x->key)) {
    return true;
  } else {
    return false;
  }
}
```
查找操作只会有下面2种情形：

-    观察不到新插入的结点。
-    在某一层观察到新插入的结点，且更低层也会观察到，也就是完好的 skiplist。

当读写同时发生时，上述两种情况都有可能发生，但都不会影响正确的结果，因为不会查找正在插入的 key(Sequence 只有写操作完成才会更新)。 

- [LevelDB : SkipList](https://huntinux.github.io/leveldb-skiplist.html)
- [Skip Lists](https://www.csee.umbc.edu/courses/341/fall01/Lectures/SkipLists/skip_lists/skip_lists.html)
- [LevelDB源码剖析之基础部件-SkipList](https://www.jianshu.com/p/6624befde844)
- [leveldb 源码分析(三) – Write](https://youjiali1995.github.io/storage/leveldb-write/)
- [理解 C++ 的 Memory Order](https://senlinzhan.github.io/2017/12/04/cpp-memory-order/)
- [深入浅出leveldb之MemTable](https://blog.csdn.net/weixin_42663840/article/details/82263358)

