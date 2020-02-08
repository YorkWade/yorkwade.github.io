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
- 索引的数据结构<br>
  1、高效插入、高效查询。索引的数据结构<br>
  2、多线程并发时的同步。<br>
leveldb使用skiplist作为索引数据结构。<br>
为什么不使用hash索引结构？<br>
    因为hash（无序），区间搜索效率低。hash变满时，继续增长的代价昂贵，hash冲突时序如则的处理逻辑。<br>
为什么不使用B-Tree更为成熟的数据结构？<br>
    SkipList具有同样的查询效率，实现简单，且数据**加锁**更容易。
    
- Key的编码<br>
1、同一key的多次插入的排序<br>
2、不同key的排序<br>
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
![](https://bean-li.github.io/assets/LevelDB/leveldb_key.png)

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

先看比较器接口
```objc
class Comparator {
 public:
  virtual ~Comparator();
 
 
  // Three-way comparison.  Returns value:
  //   < 0 iff "a" < "b",
  //   == 0 iff "a" == "b",
  //   > 0 iff "a" > "b"
  virtual int Compare(const Slice& a, const Slice& b) const = 0;
 
 
  // The name of the comparator.  Used to check for comparator
  // mismatches (i.e., a DB created with one comparator is
  // accessed using a different comparator.
  //
  // The client of this package should switch to a new name whenever
  // the comparator implementation changes in a way that will cause
  // the relative ordering of any two keys to change.
  //
  // Names starting with "leveldb." are reserved and should not be used
  // by any clients of this package.
  //返回comparator的名字
  virtual const char* Name() const = 0;
 
 
  // Advanced functions: these are used to reduce the space requirements
  // for internal data structures like index blocks.
 
 
  // If *start < limit, changes *start to a short string in [start,limit).
  // Simple comparator implementations may return with *start unchanged,
  // i.e., an implementation of this method that does nothing is correct.
  // 这两个函数作用是减少像index blocks这样的内部数据结构占用的空间。
  //找出 [start, limit）之间的一个短的串，主要作用是降低一些存储空间。
  virtual void FindShortestSeparator(
      std::string* start,
      const Slice& limit) const = 0;
 
 
  // Changes *key to a short string >= *key.
  // Simple comparator implementations may return with *key unchanged,
  // i.e., an implementation of this method that does nothing is correct.
  // 将*key变成一个比原*key大的短字符串，并赋值给*key返回。
  //作用类似，但无上端限制
  virtual void FindShortSuccessor(std::string* key) const = 0;
};
```
从comparator的接口来看，就想要对重复字符串存储，省空间。<br>
有两个比较器继承该接口，一个是InternalKeyComparator（1、user key按升序排列，2、sequence number按降序排列，3、value type按降序排列），<br>一个是BytewiseComparatorImpl（按字典逐字节序进行比较的,这里就不多说了）

```objc
int InternalKeyComparator::Compare(const Slice& akey, const Slice& bkey) const {
  // Order by:
  //    increasing user key (according to user-supplied comparator)
  //    decreasing sequence number
  //    decreasing type (though sequence# should be enough to disambiguate)
  int r = user_comparator_->Compare(ExtractUserKey(akey), ExtractUserKey(bkey));
  if (r == 0) {
    const uint64_t anum = DecodeFixed64(akey.data() + akey.size() - 8);
    const uint64_t bnum = DecodeFixed64(bkey.data() + bkey.size() - 8);
    if (anum > bnum) {
      r = -1;
    } else if (anum < bnum) {
      r = +1;
    }
  }
  return r;
}
```

```objc
void InternalKeyComparator::FindShortestSeparator(
      std::string* start,
      const Slice& limit) const {
  // Attempt to shorten the user portion of the key
  Slice user_start = ExtractUserKey(*start);
  Slice user_limit = ExtractUserKey(limit);
  std::string tmp(user_start.data(), user_start.size());
  user_comparator_->FindShortestSeparator(&tmp, user_limit);
  if (tmp.size() < user_start.size() &&
      user_comparator_->Compare(user_start, tmp) < 0) {
    // User key has become shorter physically, but larger logically.
    // Tack on the earliest possible number to the shortened user key.
    PutFixed64(&tmp, PackSequenceAndType(kMaxSequenceNumber,kValueTypeForSeek));
    assert(this->Compare(*start, tmp) < 0);
    assert(this->Compare(tmp, limit) < 0);
    start->swap(tmp);
  }
}
```
1）该函数取出Internal Key中的user_key字段，根据用户指定的comparator找到短字符串并替换user_start。此时user_start物理上是变短了，但是逻辑上却变大了<br>
2）如果user_start被替换了，就用新的user_start更新Internal Key，并使用最大的sequence number。否则start保持不变。<br>


Add主要就是按照| VarInt(Internal Key size) len | internal key |VarInt(value) len |value|的格式，pack到一块内存中，然后调用:table_.Insert(buf);
```objc
void MemTable::Add(SequenceNumber s, ValueType type,
                   const Slice& key,
                   const Slice& value) {
  // Format of an entry is concatenation of:
  //  key_size     : varint32 of internal_key.size()
  //  key bytes    : char[internal_key.size()]
  //  value_size   : varint32 of value.size()
  //  value bytes  : char[value.size()]
  size_t key_size = key.size();
  size_t val_size = value.size();
  size_t internal_key_size = key_size + 8;
  const size_t encoded_len =
      VarintLength(internal_key_size) + internal_key_size +
      VarintLength(val_size) + val_size;
  char* buf = arena_.Allocate(encoded_len);
  char* p = EncodeVarint32(buf, internal_key_size);
  memcpy(p, key.data(), key_size);
  p += key_size;
  EncodeFixed64(p, (s << 8) | type);
  p += 8;
  p = EncodeVarint32(p, val_size);
  memcpy(p, value.data(), val_size);
  assert((p + val_size) - buf == encoded_len);
  table_.Insert(buf);
}
```

Get核心逻辑是一个Seek函数，根据传入的LookupKey得到在memtable中存储的key，然后调用Skip list::Iterator的Seek函数查找。Seek直接调用Skip list的FindGreaterOrEqual(key)接口，返回大于等于key的Iterator。然后取出user key判断时候和传入的user key相同，如果相同则取出value，如果记录的Value Type为kTypeDeletion，返回Status::NotFound(Slice())。
```objc
bool MemTable::Get(const LookupKey& key, std::string* value, Status* s) {
  Slice memkey = key.memtable_key();
  Table::Iterator iter(&table_);
  iter.Seek(memkey.data());
  if (iter.Valid()) {
    // entry format is:
    //    klength  varint32
    //    userkey  char[klength]
    //    tag      uint64
    //    vlength  varint32
    //    value    char[vlength]
    // Check that it belongs to same user key.  We do not check the
    // sequence number since the Seek() call above should have skipped
    // all entries with overly large sequence numbers.
    const char* entry = iter.key();
    uint32_t key_length;
    const char* key_ptr = GetVarint32Ptr(entry, entry+5, &key_length);
    if (comparator_.comparator.user_comparator()->Compare(
            Slice(key_ptr, key_length - 8),
            key.user_key()) == 0) {
      // Correct user key
      const uint64_t tag = DecodeFixed64(key_ptr + key_length - 8);
      switch (static_cast<ValueType>(tag & 0xff)) {
        case kTypeValue: {
          Slice v = GetLengthPrefixedSlice(key_ptr + key_length);
          value->assign(v.data(), v.size());
          return true;
        }
        case kTypeDeletion:
          *s = Status::NotFound(Slice());
          return true;
      }
    }
  }
  return false;
}
```
```

##参考

- [LevelDb 源码阅读--memtable](https://zhuanlan.zhihu.com/p/79064869)
- [leveldb设计分析之memtable](https://blog.csdn.net/brainkick/article/details/48679345)

