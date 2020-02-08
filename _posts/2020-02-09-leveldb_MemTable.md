---
layout:     post
title:      leveldb之MemTable
subtitle:   所有的应用都是内存数据库加上业务逻辑
date:       2002-02-09
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


levedb三个存储区域：Memtable，Immutable Memtable和SSTable中的。Memtable，Immutable Memtable结构一样，差别在于：<br>
    memtable 允许写入跟读取。<br>
    immutable memtable只读。<br>



![](https://ww2.sinaimg.cn/large/006tKfTcgy1fckb184f74j319v0q01kx.jpg)


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


在 **系统偏好设置** -> **键盘设置** -> **快捷键** -> **服务**

选择我们创建好的 '**打开终端**'，设置你想要的快捷键，比我我设置了`⌘+空格`

![](https://ww4.sinaimg.cn/large/006tKfTcgy1fckbvaixhnj30kw0ihq67.jpg)

到此，设置完成。

聪明的你也许会发现，这个技巧能为所有的程序设置快捷启动。

将脚本中的 `Terminal` 替换成 其他程序就可以

```
on run {input, parameters}
    tell application "Terminal"
        reopen
        activate
    end tell
end run

```

## 黑技能

既然学了 `Automator` ，那就在附上一个黑技能吧。为你的代码排序。在 **Xcode8**以前，有个插件能为代码快速排序，不过时过境迁~ 对于没用的插件而且又有患有强迫症的的小伙伴，只能手动排序了（😂）.

首先还是创建一个服务

创建一个`Shell`脚本，

勾选:`用输出内容替换所选文本`

输入：`sort|uniq` 

保存： 存为`Sort & Uniq`

![](https://ww4.sinaimg.cn/large/006tKfTcgy1fckd40rgwmj30rt0ildiy.jpg)

**选中你的代代码** -> **鼠标右键** -> **Servies** -> **Sort&Uniq**

![](https://ww2.sinaimg.cn/large/006tKfTcgy1fckd6tx1dzj30h90b7mzm.jpg)

排序后的代码：

![](https://ww3.sinaimg.cn/large/006tKfTcgy1fckd6lak55j309j05y3yo.jpg)

