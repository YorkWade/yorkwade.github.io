---
layout:     post
title:      leveldb之Version
subtitle:   
date:       2020-02-15
author:     BY
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - leveldb
---





相较与OC来说更易读了：

```objc
let dispatch_time = dispatch_time(DISPATCH_TIME_NOW, Int64(60 * NSEC_PER_SEC))
```



```obj
1、获取版本号，只读取该版本号之前的数据；
2、在memtable中查找
3、在Imuable memtable中查找
4、在sstable（磁盘文件）中查找
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
          break;      // Keep searching in other files
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


```obj
1、获取版本号，只读取该版本号之前的数据；
2、在memtable中查找
3、在Imuable memtable中查找
4、在sstable（磁盘文件）中查找
```

## 业务流程
![](https://images2017.cnblogs.com/blog/1183530/201801/1183530-20180116202929646-1073704541.png)

### 参考 
- [Leveldb源码分析--16](https://blog.csdn.net/sparkliang/article/details/8820517)
