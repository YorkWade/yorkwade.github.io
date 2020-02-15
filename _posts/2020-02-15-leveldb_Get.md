---
layout:     post
title:      leveldb之Get
subtitle:   
date:       2020-02-15
author:     BY
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - leveldb
---



level 0的数据是Imuable memtable直接dump到磁盘的，所以文件与文件之间的key有可能重叠的。而level n（n>0）中每个sst文件之间key是不重叠的，且key在level中是全局有序的（注意是该level中）

那么在每一层中是如何查找key的呢？答案很简单，不外乎两个步骤：<br>

    1、找到所有可能含有该key的文件列表fileList；
    2、遍历fileList查找key；

第2步就是读取文件内容找出key而已，那么1是如何实现的呢？这里我们有必要复习一下前面的内容。我们除了sst文件（实际数据文件），leveldb还有manifest文件，该文件保存了每个sst文件在哪一层，最小key是啥，最大key是啥？所以：
我们通过读取manifest文件就能知道key有可能在哪一个sst文件中！

leveldb读流程主要如下：

```obj
1、获取版本号，只读取该版本号之前的数据；
2、在memtable中查找
3、在Imuable memtable中查找
4、在sstable（磁盘文件）中查找（先在TableCache中查找）
```

```objc
Status DBImpl::Get(const ReadOptions& options,
                   const Slice& key,
                   std::string* value) {
  Status s;
  MutexLock l(&mutex_);
  SequenceNumber snapshot;
  //版本号,可以读取指定版本的数据,否则读取最新版本的数据.
  //注意:读取的时候数据也是会插入的,假如Get请求先到来,而Put后插入一条数据,这时候新数据并不会被读取到!
  if (options.snapshot != NULL) {
    snapshot = reinterpret_cast<const SnapshotImpl*>(options.snapshot)->number_;
  } else {
    snapshot = versions_->LastSequence();
  }

  //分别获取到memtable和Imuable memtable的指针
  MemTable* mem = mem_;
  MemTable* imm = imm_;
  Version* current = versions_->current();
  //增加reference计数,防止在读取的时候并发线程释放掉memtable的数据
  mem->Ref();
  if (imm != NULL) imm->Ref();
  current->Ref();

  bool have_stat_update = false;
  Version::GetStats stats;

  // Unlock while reading from files and memtables
  {
    mutex_.Unlock();
    // First look in the memtable, then in the immutable memtable (if any).
    //LookupKey是由key和版本号的封装.用来查找,不然每次都要传两个参数.把高耦合的参数合并成一个数据结构!
    LookupKey lkey(key, snapshot);
    if (mem->Get(lkey, value, &s)) { //memtable中查找
      // Done
    } else if (imm != NULL && imm->Get(lkey, value, &s)) { //Imuable memtable中查找
      // Done
    } else {
      s = current->Get(options, lkey, value, &stats);  //sstable中查找(内存中找不到就会进入这一步)
      have_stat_update = true;
    }
    mutex_.Lock();
  }

  if (have_stat_update && current->UpdateStats(stats)) {
    MaybeScheduleCompaction();  //检查是否要进行compaction操作
  }

  //释放引用计数. ps:自己维护一套这样的机制思维要非常清晰,否则很容易出bug.
  mem->Unref();
  if (imm != NULL) imm->Unref();
  current->Unref();
  return s;
}
```
Version是管理某个版本的所有sstable的类，就其导出接口而言，无非是遍历sstable，查找k/v。以及为compaction做些事情，给定range，检查重叠情况。
而它不会修改它管理的sstable这些文件，对这些文件而言它是只读操作接口
```objc

Status Version::Get(const ReadOptions& options,
                    const LookupKey& k,
                    std::string* value,
                    GetStats* stats) {
  Slice ikey = k.internal_key();
  Slice user_key = k.user_key();
  const Comparator* ucmp = vset_->icmp_.user_comparator();
  Status s;

  stats->seek_file = NULL;
  stats->seek_file_level = -1;
  FileMetaData* last_file_read = NULL;
  int last_file_read_level = -1;

  // We can search level-by-level since entries never hop across
  // levels.  Therefore we are guaranteed that if we find data
  // in an smaller level, later levels are irrelevant.
  std::vector<FileMetaData*> tmp;
  FileMetaData* tmp2;
  for (int level = 0; level < config::kNumLevels; level++) {
    
    /*-----------------找到可能包含key的文件列表begin------------------------*/
    size_t num_files = files_[level].size();
    if (num_files == 0) continue;

    // Get the list of files to search in this level
    FileMetaData* const* files = &files_[level][0];
    if (level == 0) { //level0特殊对待,key有可能重叠，有可能在任何一个level0的文件中
      // Level-0 files may overlap each other.  Find all files that
      // overlap user_key and process them in order from newest to oldest.
      tmp.reserve(num_files);
      for (uint32_t i = 0; i < num_files; i++) {// 遍历level 0下的所有sstable文件
        FileMetaData* f = files[i];
        if (ucmp->Compare(user_key, f->smallest.user_key()) >= 0 &&
            ucmp->Compare(user_key, f->largest.user_key()) <= 0) {
          tmp.push_back(f); //如果查找key落在该文件大小范围,则加到文件列表供下面进一步查询
        }
      }
      if (tmp.empty()) continue;

      std::sort(tmp.begin(), tmp.end(), NewestFirst);
      files = &tmp[0];
      num_files = tmp.size();
    } else {
      // Binary search to find earliest index whose largest key >= ikey.
      //二分查找，找到第一个largest key >=ikey的file index
      uint32_t index = FindFile(vset_->icmp_, files_[level], ikey); 
      if (index >= num_files) {
        files = NULL;
        num_files = 0;
      } else {
        tmp2 = files[index];
        if (ucmp->Compare(user_key, tmp2->smallest.user_key()) < 0) {
          // All of "tmp2" is past any data for user_key
          files = NULL;
          num_files = 0;
        } else {
          files = &tmp2;
          num_files = 1;
        }
      }
    }
    /*-----------------找到可能包含key的文件列表end------------------------*/


    /*-----------------遍历文件查找key begin------------------------*/
    for (uint32_t i = 0; i < num_files; ++i) {          //如果num_files不为0,说明key有可能在这些文件中
      if (last_file_read != NULL && stats->seek_file == NULL) {
        // We have had more than one seek for this read.  Charge the 1st file.
        stats->seek_file = last_file_read;
        stats->seek_file_level = last_file_read_level;
      }

      FileMetaData* f = files[i];
      last_file_read = f;
      last_file_read_level = level;

      Saver saver;
      saver.state = kNotFound;
      saver.ucmp = ucmp;
      saver.user_key = user_key;
      saver.value = value;
      s = vset_->table_cache_->Get(options, f->number, f->file_size,
                                   ikey, &saver, SaveValue); //在cache中读取文件的内容
      if (!s.ok()) {
        return s;
      }
      switch (saver.state) { //查找结果返回!
        case kNotFound:
          break;      // Keep searching in other files继续查找，知道找出的所有文件都查找完
        case kFound:
          return s;
        case kDeleted:
          s = Status::NotFound(Slice());  // Use empty error message for speed
          return s;
        case kCorrupt:
          s = Status::Corruption("corrupted key for ", user_key);
          return s;
      }
    }
    /*-----------------遍历文件查找key begin------------------------*/
    
  }

  return Status::NotFound(Slice());  // Use an empty error message for speed
}
```




## 业务流程
![](https://images2017.cnblogs.com/blog/1183530/201801/1183530-20180116202929646-1073704541.png)

### 参考 
- [Leveldb源码分析--16](https://blog.csdn.net/sparkliang/article/details/8820517)
- [Leveldb源码分析--17](https://blog.csdn.net/sparkliang/article/details/8914374)
- [leveldb源码解析之三Get实现](https://www.jianshu.com/p/d1e7efacc394)

