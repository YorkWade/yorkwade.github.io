---
layout:     post
title:      leveldb之MemTable
subtitle:   所有的应用都是内存数据库加上业务逻辑
date:       2020-02-09
author:     BY
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - leveldb
    
---

>在Leveldb中，所有内存中的KV数据都存储在Memtable中,内部使用[SkipList](https://yorkwade.github.io/2020/02/05/leveldb_SkipList/)实现。当Memtable写入的数据占用内存到达指定数量，则自动转换为Immutable Memtable，等待Dump到磁盘中，系统会自动生成新的Memtable供写操作写入新数据。

## 实现要点
- 高效插入、高效查询。索引的数据结构
- 多线程并发时的同步。

leveldb使用skiplist作为索引数据结构。
- 为什么不使用hash索引结构？<br>
    因为hash（无序），区间搜索效率低。hash变满时，继续增长的代价昂贵，hash冲突时序如则的处理逻辑。<br>
- 为什么不使用B-Tree更为成熟的数据结构？<br>
    SkipList具有同样的查询效率，实现简单，且数据**加锁**更容易。
    
Key的编码
LevelDb的Memtable中KV对是根据Key大小有序存储的，在系统插入新的KV时，LevelDb要把这个KV插到合适的位置上以保持这种Key有序性。
其中有三种key：UserKey,LookUpKey和InternalKey<br>
**UserKey**，是用户输入的Key，Slice字符串类型<br>
**InternalKey**，是userkey+sequence+type组成<br>
```objc
    |user key|sequence number|type|<br>
    internal key size=key_size+8<br>
```
**LookUpKey**,由InternalKey的长度+InternalKey组成<br>
```objc
    |internal key size|internalkey|<br>
```
![](https://bean-li.github.io/assets/LevelDB/leveldb-keys.png)

了解了以上的Key，可明白skiplist中存储的键为LookUpKey+value的长度+value<br>
VarInt(interbal key size)len | internal key | VarInt(value) len | value |
![](https://pic4.zhimg.com/80/v2-662b22e9fb3639adf416135d7200085b_hd.jpg)


## 源码实现
``` obj
class InternalKeyComparator;
class Mutex;
class MemTableIterator;

class MemTable {
 public:
  // MemTables are reference counted.  The initial reference count
  // is zero and the caller must call Ref() at least once.
  explicit MemTable(const InternalKeyComparator& comparator);

  // Increase reference count.
  void Ref() { ++refs_; }

  // Drop reference count.  Delete if no more references exist.
  void Unref() {
    --refs_;
    assert(refs_ >= 0);
    if (refs_ <= 0) {
      delete this;
    }
  }

  // Returns an estimate of the number of bytes of data in use by this
  // data structure.
  //
  // REQUIRES: external synchronization to prevent simultaneous
  // operations on the same MemTable.
  size_t ApproximateMemoryUsage();

  // Return an iterator that yields the contents of the memtable.
  //
  // The caller must ensure that the underlying MemTable remains live
  // while the returned iterator is live.  The keys returned by this
  // iterator are internal keys encoded by AppendInternalKey in the
  // db/format.{h,cc} module.
  Iterator* NewIterator();

  // Add an entry into memtable that maps key to value at the
  // specified sequence number and with the specified type.
  // Typically value will be empty if type==kTypeDeletion.
  void Add(SequenceNumber seq, ValueType type,
           const Slice& key,
           const Slice& value);

  // If memtable contains a value for key, store it in *value and return true.
  // If memtable contains a deletion for key, store a NotFound() error
  // in *status and return true.
  // Else, return false.
  bool Get(const LookupKey& key, std::string* value, Status* s);

 private:
  ~MemTable();  // Private since only Unref() should be used to delete it

  struct KeyComparator {
    const InternalKeyComparator comparator;
    explicit KeyComparator(const InternalKeyComparator& c) : comparator(c) { }
    int operator()(const char* a, const char* b) const;
  };
  friend class MemTableIterator;
  friend class MemTableBackwardIterator;

  typedef SkipList<const char*, KeyComparator> Table;

  KeyComparator comparator_;
  int refs_;
  Arena arena_;
  Table table_;

  // No copying allowed
  MemTable(const MemTable&);
  void operator=(const MemTable&);
};

```
MemTable的核心组件有三个：
- [SkipList](https://yorkwade.github.io/2020/02/05/leveldb_SkipList/)
- [Arena](https://yorkwade.github.io/2020/02/04/leveldb_Arena/)
- KeyComparator

MemTable的主要接口两个：
- Add
- Get



##参考

- [LevelDb 源码阅读--memtable](https://zhuanlan.zhihu.com/p/79064869)
