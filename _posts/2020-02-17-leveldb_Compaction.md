---
layout:     post
title:      leveldb之Compaction
subtitle:   归并
date:       2020-02-17
author:     BY
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - leveldb
---

leveldb是LSM数据库，Log Structured Merge中，Merge就对应着Compaction。
     
## 前言

### 为什么要compaction?

compaction可以提高数据的查询效率，没有经过compaction，需要从很多SST file去查找，而做过compaction后，只需要从有限的SST文件去查找，大大的提高了随机查询的效率，另外也可以删除过期数据

### 什么时候可能进行compaction?
 
    1. database open的时候(DB::Open)
    2. write的时候( DBImpl::Write(DBImpl::MakeRoomForWrite) )
    3. read的时候( DBImpl::Get )

上述操作最终会调用DBImpl::MaybeScheduleCompaction => env_->Schedule(&DBImpl::BGWork, this) => (1) 启后台线程 PosixEnv::Schedule (2) DBImpl::BGWork => DBImpl::BackgroundCall => DBImpl::BackgroundCompaction

### 触发条件

levelDb的compaction机制和过程与Bigtable所讲述的是基本一致的，Bigtable中讲到三种类型的compaction: minor ，major和full。所谓minor Compaction，就是把memtable中的数据导出到SSTable文件中；major compaction就是合并不同层级的SSTable文件，而full compaction就是将所有SSTable进行合并。LevelDb包含其中两种，minor和major。

**Memtable Compaction**

当 memtable 大小超过 Options.write_buffer_size 时(默认 4MB)，会在下一次写操作时将当前的 memtable 转为 immutable memtable，创建新的 memtable，并触发 immutable memtable 的 compaction。compaction 会由单独的线程来执行。

memtable compaction 的过程很简单，顺序遍历 memtable 将所有的 key/value 转储为 sstable 格式即可(不会清理无用数据)，生成的 sstable 不一定在 level-0，只要满足上面的保证即可。 要注意的是，leveldb 为了防止 sstable 数量太多会对写操作进行流控：

    当 level-0 sstable 数量达到 kL0_SlowdownWritesTrigger(8) 时，每个写操作会 sleep(1ms)。
    当前 memtable 已满需要 compaction 但之前的 immutable memtable compaction 还未完成时，会等待之前的完成。
    当 level-0 sstable 数量达到 kL0_StopWritesTrigger(12) 时，会等待 level-0 compaction 完成。
    
**Sstable Compaction**

触发 sstable compaction 的条件如下：

    level-0：sstable 文件个数超过 kL0_CompactionTrigger(4)。因为 level-0 是从 sstable 直接转储而来，所以用个数限制而不是大小。
    其他 level：高层的 sstable 会按照 max_file_size(2MB) 进行切割，当一层的 sstable 总大小超过阈值时会触发，最高层无大小限制。
    每个文件还有 seek 的次数限制，超过次数会进行 compaction，防止读多写少的场景下，compaction 不会触发。

挑选参与 compaction 的文件分2步：

    执行 compaction 的 level：leveldb 会记录每个 level 上次 compaction 的最大的 key，下一次时会挑选在这之后的文件，防止后面的文件一直不会被选到。
    高一层的文件：挑选和低一层的文件有重叠的所有文件。高一层的总的 key range 可能会覆盖到更多的低一层的文件，所以会进行 expand，同时为了防止 compaction 太大， 会有一定的限制。

### 源码分析
```objc

// REQUIRES: mutex_ is held
// REQUIRES: this thread is currently at the front of the writer queue
Status DBImpl::MakeRoomForWrite(bool force) {
  mutex_.AssertHeld();
  assert(!writers_.empty());
  bool allow_delay = !force;
  Status s;
  while (true) {//该循环会一直执行，直到mem_中有空间可供写
    if (!bg_error_.ok()) {////背景线程执行有错误，直接返回
      // Yield previous error
      s = bg_error_;
      break;
    } else if (
        allow_delay &&
        versions_->NumLevelFiles(0) >= config::kL0_SlowdownWritesTrigger) {
      // We are getting close to hitting a hard limit on the number of
      // L0 files.  Rather than delaying a single write by several
      // seconds when we hit the hard limit, start delaying each
      // individual write by 1ms to reduce latency variance.  Also,
      // this delay hands over some CPU to the compaction thread in
      // case it is sharing the same core as the writer.
      mutex_.Unlock();
      //如果发现level 0 中的文件过多，则说明需要背景线程进行合并，因此需要等待背景线程合并完成之后再写入，所以延迟了写。
      env_->SleepForMicroseconds(1000);
      allow_delay = false;  // Do not delay a single write more than once
      mutex_.Lock();
    } else if (!force &&
               (mem_->ApproximateMemoryUsage() <= options_.write_buffer_size)) {// mem_中有空间可供写
      // There is room in current memtable循环只有在这里才会正确地返回
      break;
    } else if (imm_ != NULL) {//mem_没有空间
      // We have filled up the current memtable, but the previous
      // one is still being compacted, so we wait.
      //此时试图将mem_赋值给imm_，然后重新分配一个mem_供用户写，但在此之前必须先检查imm_是否为空，
      //如果它不为空的话，说明上次赋值给imm_的mem还没有被背景线程写入磁盘，那只能等待了，不然就会覆盖掉之前的mem_。
      Log(options_.info_log, "Current memtable full; waiting...\n");
      bg_cv_.Wait();
    } else if (versions_->NumLevelFiles(0) >= config::kL0_StopWritesTrigger) {
      // There are too many level-0 files.
      //level 0 中的sstable又增加了一个文件，我们必须判断此时level 0 中的文件数量是否太多，太多的话那就还得等一会
      Log(options_.info_log, "Too many L0 files; waiting...\n");
      bg_cv_.Wait();
    } else {//imm_为空，mem_没有空间可写
      // Attempt to switch to a new memtable and trigger compaction of old
      assert(versions_->PrevLogNumber() == 0);
      uint64_t new_log_number = versions_->NewFileNumber();
      WritableFile* lfile = NULL;
      s = env_->NewWritableFile(LogFileName(dbname_, new_log_number), &lfile);
      if (!s.ok()) {
        // Avoid chewing through file number space in a tight loop.
        versions_->ReuseFileNumber(new_log_number);
        break;
      }
      delete log_;//每个log对应一个mem_,因为在这之后这个mem_将不再改变，所以不再需要这个log了
      delete logfile_;
      logfile_ = lfile;
      logfile_number_ = new_log_number;
      log_ = new log::Writer(lfile);
      imm_ = mem_;//后台将启动对imm_ 进行写磁盘的过程
      has_imm_.Release_Store(imm_);
      mem_ = new MemTable(internal_comparator_);
      mem_->Ref();
      force = false;   // Do not force another compaction if have room
      // 有可能启动后台compaction线程的地方（因为可能后台compactiong线程已经启动了）,后台只对imm_处理，不会对mem_处里
      MaybeScheduleCompaction();
    }
  }
  return s;
}
```
从MakeRoomForWrite函数返回之后，mem_中都会有足够空间可写了





## 结语

- [leveldb资料整理 ](https://www.iteye.com/blog/hideto-1328921)
- [leveldb 源码分析(四) – Compaction](https://youjiali1995.github.io/storage/leveldb-compaction/)
- [leveldb源码剖析---DBImpl::MakeRoomForWrite函数的实现](https://blog.csdn.net/swartz2015/article/details/66972106)

