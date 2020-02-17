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
     

### 为什么要compaction?

compaction可以提高数据的查询效率，没有经过compaction，需要从很多SST file去查找，而做过compaction后，只需要从有限的SST文件去查找，大大的提高了随机查询的效率，另外也可以删除过期数据

### 什么时候可能进行compaction?
 
    1. database open的时候(DB::Open)
    2. write的时候( DBImpl::Write(DBImpl::MakeRoomForWrite) )
    3. read的时候( DBImpl::Get )



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

leveldb在Open/Get/Write时都有可能做
Compaction: DB::Open/DBImpl::Get/DBImpl::Write(DBImpl::MakeRoomForWrite) =>DBImpl::MaybeScheduleCompaction => env_->Schedule(&DBImpl::BGWork, this) => (1) 启后台线程 PosixEnv::Schedule (2) DBImpl::BGWork => DBImpl::BackgroundCall => DBImpl::BackgroundCompaction 



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
      //我们可以把mem_复制给imm_，并为mem_重新分配一个新的Memtable了
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
从MakeRoomForWrite函数返回之后，mem_中都会有足够空间可写了.最后的MaybeScheduleCompaction就是开启一个背景线程。背景线程主要做两件事情:

       1. 将imm_写入磁盘生成一个新的sstable
       2. 对各个level中的文件进行合并，避免某个level中的文件过多，以及删除掉一些过期或者已经被用户调用delete删除的key-value。 

```objc
void DBImpl::MaybeScheduleCompaction() {
  mutex_.AssertHeld();
  if (bg_compaction_scheduled_) {
    // Already scheduled
  } else if (shutting_down_.Acquire_Load()) {
    // DB is being deleted; no more background compactions
  } else if (!bg_error_.ok()) {
    // Already got an error; no more changes
  } else if (imm_ == NULL &&
             manual_compaction_ == NULL &&
             !versions_->NeedsCompaction()) {
    // No work to be done
  } else {
    bg_compaction_scheduled_ = true;
    env_->Schedule(&DBImpl::BGWork, this);  // start new thread to compact memtable , the start point is "BGWork"
  }
}
```
从这里我们可以看到，每个时刻，leveldb只允许一个背景线程存在。这里需要加锁主要也是这个原因，防止某个瞬间两个线程同时开启背景线程。当确定当前数据库中没有背景线程，也不存在错误，同时确实有工作需要背景线程来完成，就通过env_->Schedule(&DBImpl::BGWork, this)启动背景线程，前面的bg_compaction_scheduled_设置主要是告诉其他线程当前数据库中已经有一个背景线程在运行了。
```objc
void DBImpl::BackgroundCall() {
  MutexLock l(&mutex_);
  assert(bg_compaction_scheduled_);
  if (shutting_down_.Acquire_Load()) {
    // No more background work when shutting down.
  } else if (!bg_error_.ok()) {
    // No more background work after a background error.
  } else {
    BackgroundCompaction();//背景线程的核心工作
  }

  bg_compaction_scheduled_ = false;

  // Previous compaction may have produced too many files in a level,
  // so reschedule another compaction if needed. 
  MaybeScheduleCompaction();
  bg_cv_.SignalAll();  // 后台的compaction操作完成
}
```
前两个判断语句主要是判断当前数据库是不是被关闭以及背景线程是否出错了。shutting_down_只会在DB的析构函数中被设置。

如果都没有问题那就进入else里面的BackgroundCompaction处理背景线程的核心工作。

正如我们前面所说，BackgroundCompaction里面主要处理两个工作：

    如果当前的imm_非空，则将其写盘生成一个新的sstable
    对各个level的文件进行合并，避免level中文件过多，以及删掉被删除的key-value(因为leveldb里面采用的是lazy delete的方法，用户调用delete时没有真正删除元素，只有在背景线程对文件进行合并时才会真的删除元素)。

背景线程完成compaction工作之后，它会尝试继续开启一个背景线程。因为在背景线程执行的时候，其他的用户线程可能已经向mem_写入了很多数据，而imm_在BackgroundCompaction已经被写入磁盘变为空，所以可能此时imm又已经被写满的mem_赋值了，所以应该尝试继续开启新的线程对imm_进行写盘。

不管开启新的背景线程是否成功，当前这个已经完成任务的老旧背景线程都将结束。其实从MaybeScheduleCompaction函数中我们可以看到，只要没有shut_down和背景线程没有出错，一直都会有一个背景线程在后面运行

bg_cv_.SignalAll()将会唤醒睡眠的用户线程，因为当一个背景线程执行完成之后，用户线程的执行条件可能已经满足了，比如level0中经过合并后，没有那么多文件了。

**每个时刻系统中只允许一个背景线程，背景线程负责两个工作：1. 将imm_写盘。2. 对level之间的文件进行合并。imm_始终记录的是上一个写满的mem_，每当一个mem_写满时，它都会赋值给imm_，同时重新分配一个Memtable给mem_，这样就可以避免将memtable写盘时影响用户写数据。**

```objc

void DBImpl::BackgroundCompaction() {
  mutex_.AssertHeld();

  if (imm_ != NULL) {
    CompactMemTable();//将imm_写到level 0
    return;
  }

  Compaction* c;
  bool is_manual = (manual_compaction_ != NULL);
  InternalKey manual_end;
  if (is_manual) {//在设置manual_compaction_ 的时候被执行，它主要是用于测试数据库的compaction功能，用于debug
    ManualCompaction* m = manual_compaction_;
    c = versions_->CompactRange(m->level, m->begin, m->end);
    m->done = (c == NULL);
    if (c != NULL) {
      manual_end = c->input(0, c->num_input_files(0) - 1)->largest;
    }
    Log(options_.info_log,
        "Manual compaction at level-%d from %s .. %s; will stop at %s\n",
        m->level,
        (m->begin ? m->begin->DebugString().c_str() : "(begin)"),
        (m->end ? m->end->DebugString().c_str() : "(end)"),
        (m->done ? "(end)" : manual_end.DebugString().c_str()));
  } else {
    c = versions_->PickCompaction();//找出最适合compaction的level,它将返回可以进行compaction的level中的文件元信息，这些元信息存储在Compaction类中
  }

  Status status;
  if (c == NULL) {//如果c为空，说明没有文件需要进行compaction
    // Nothing to do
  } else if (!is_manual && c->IsTrivialMove()) {//这个条件分支主要是处理level+1中没有文件需要和level中的那个文件进行合并的情况。
  //这种情况，很简单，直接把level中的那个需要合并的文件移动到level+1中即可
    // Move file to next level
    assert(c->num_input_files(0) == 1);
    FileMetaData* f = c->input(0, 0);
    c->edit()->DeleteFile(c->level(), f->number);
    c->edit()->AddFile(c->level() + 1, f->number, f->file_size,
                       f->smallest, f->largest);
    status = versions_->LogAndApply(c->edit(), &mutex_);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    VersionSet::LevelSummaryStorage tmp;
    Log(options_.info_log, "Moved #%lld to level-%d %lld bytes %s: %s\n",
        static_cast<unsigned long long>(f->number),
        c->level() + 1,
        static_cast<unsigned long long>(f->file_size),
        status.ToString().c_str(),
        versions_->LevelSummary(&tmp));
  } else {// 否则进行compaction
  //此时说明level+1中有文件和level中的那个需要合并的文件key范围重合。
  //因此需要将这个文件和level+1中的那些文件进行合并操作。主体工作是在DoCompactionWork函数中完成
    CompactionState* compact = new CompactionState(c);
    status = DoCompactionWork(compact);
    if (!status.ok()) {
      RecordBackgroundError(status);
    }
    CleanupCompaction(compact);
    c->ReleaseInputs();
    DeleteObsoleteFiles();
  }
  delete c;

  if (status.ok()) {
    // Done
  } else if (shutting_down_.Acquire_Load()) {
    // Ignore compaction errors found during shutting down
  } else {
    Log(options_.info_log,
        "Compaction error: %s", status.ToString().c_str());
  }

  if (is_manual) {
    ManualCompaction* m = manual_compaction_;
    if (!status.ok()) {
      m->done = true;
    }
    if (!m->done) {
      // We only compacted part of the requested range.  Update *m
      // to the range that is left to be compacted.
      m->tmp_storage = manual_end;
      m->begin = &m->tmp_storage;
    }
    manual_compaction_ = NULL;
  }
}
```

leveldb的compaction操作主要是由DoCompactionWork函数完成：
```objc
Status DBImpl::DoCompactionWork(CompactionState* compact) {
  const uint64_t start_micros = env_->NowMicros();
  int64_t imm_micros = 0;  // Micros spent doing imm_ compactions

  Log(options_.info_log,  "Compacting %d@%d + %d@%d files",
      compact->compaction->num_input_files(0),
      compact->compaction->level(),
      compact->compaction->num_input_files(1),
      compact->compaction->level() + 1);

  assert(versions_->NumLevelFiles(compact->compaction->level()) > 0);
  assert(compact->builder == NULL);
  assert(compact->outfile == NULL);
  if (snapshots_.empty()) {
    compact->smallest_snapshot = versions_->LastSequence();
  } else {
    compact->smallest_snapshot = snapshots_.oldest()->number_;
  }

  // Release mutex while we're actually doing the compaction work
  mutex_.Unlock();
  //获得一个可以遍历需要compaction的所有文件(level和level+1中所有需要进行compaction操作的文件)的迭代器，
  //每个迭代器对应一个key-value,这样，我们通过这个迭代器就可以找到compaction结构中的inputs_数组里面的所有文件的key-value。
  Iterator* input = versions_->MakeInputIterator(compact->compaction);
  input->SeekToFirst();
  Status status;
  ParsedInternalKey ikey;
  std::string current_user_key;
  bool has_current_user_key = false;
  SequenceNumber last_sequence_for_key = kMaxSequenceNumber;
  for (; input->Valid() && !shutting_down_.Acquire_Load(); ) {////每个input对应的是一个 K/V
  //循环的主体工作是判断当前迭代器对应的key是否应该加入到新合并生成的文件中
    // Prioritize immutable compaction work
    if (has_imm_.NoBarrier_Load() != NULL) {
      const uint64_t imm_start = env_->NowMicros();
      mutex_.Lock();
      //每次循环开始先判断当前的imm_是否为空，如果为空的话，先将它写入磁盘，
      //这主要是为了防止imm_没有及时写盘造成用户线程不能写mem。
      if (imm_ != NULL) {
        CompactMemTable();////这里就是将imm_写入磁盘中
        bg_cv_.SignalAll();  // Wakeup MakeRoomForWrite() if necessary
      }
      mutex_.Unlock();
      imm_micros += (env_->NowMicros() - imm_start);
    }

    //把迭代器对应的key提取出来，因为在此之前我们可能以及遍历过多个key-value了，
    //也就是可能已经将多个key-value写入到新的sstable中了
    Slice key = input->key();
    //这里通过ShouldStopBefore函数判断是否符合生成一个新的sstable的条件，
    //如果符合的话就将这个sstable写盘，如果不符合的话，就继续往里面加key-value
    if (compact->compaction->ShouldStopBefore(key) &&
        compact->builder != NULL) {
      status = FinishCompactionOutputFile(compact, input);
      if (!status.ok()) {
        break;
      }
    }

    // Handle key/value, add to state, etc.
    bool drop = false;
    if (!ParseInternalKey(key, &ikey)) {//不合法的key
      // Do not hide error keys
      current_user_key.clear();
      has_current_user_key = false;
      last_sequence_for_key = kMaxSequenceNumber;
    } else {
      if (!has_current_user_key ||
          user_comparator()->Compare(ikey.user_key,
                                     Slice(current_user_key)) != 0) {////如果等于0，表示这个key和之前度过的key相同，不必再添加了
        // First occurrence of this user key
        current_user_key.assign(ikey.user_key.data(), ikey.user_key.size());
        has_current_user_key = true;
        last_sequence_for_key = kMaxSequenceNumber;
      }
      //对于过期的key，通过检查这个key的序列号，判断它是否在系统快照中，如果在的话，即使它是过期key也不能丢弃
      if (last_sequence_for_key <= compact->smallest_snapshot) {    
        // Hidden by an newer entry for same user key
        drop = true;    // (A)
      } else if (ikey.type == kTypeDeletion &&
                 ikey.sequence <= compact->smallest_snapshot &&
                 compact->compaction->IsBaseLevelForKey(ikey.user_key)) {
        //对于非过期的key，检查这个key的type，看它是否是kTypeDeletion，即是否已经被用户删除了。
        // For this user key:
        // (1) there is no data in higher levels
        // (2) data in lower levels will have larger sequence numbers
        // (3) data in layers that are being compacted here and have
        //     smaller sequence numbers will be dropped in the next
        //     few iterations of this loop (by rule (A) above).
        // Therefore this deletion marker is obsolete and can be dropped.
        
        //至此为止，当前key要么是第一次出现，要么是有快照保护，好像不能丢弃。但是事情还没完，
        //我们还不能据此判断该key是否应该被保存下来，我们还要判断它的type，但是事情远没有我们想得那么简单，
        //除了判断key的type之外，我们还要做其他判断.也就是说，当一个key为kTypeDeletion时它还不一定是要被删除的。为什么呢。

        //想象一个场景：用户在调用delete删除一个key时，这个key在数据库中有一个过期的key存在，
        //而这个过期key还来不及和这个被删除的key合并，如果在这种情况下，我们直接将这个被标记为删除的key丢弃，
        //那数据库中还会存在一个过期的key，而这个过期的key在丢弃那个被删除的key时瞬间就变成正常不过期的key了，
        //于是下次读key时，会读到这个本应该过期的key，按道理应该是找不到key才对。所以为了系统正常运行，
        //我们每次丢弃一个标记为kTypeDeletion的key时，必须保证数据库中不存在它的过期key，否则就得将它保留，
        //直到后面它和这个过期的key合并为止，合并之后再丢弃。

        //所以这里会调用IsBaseLevelForKey函数判断level+2及level+2之后的所有level中没有和这个标记为删除的key相同的key，
        //只要有，就肯定是过期key了
        drop = true;
      }

      last_sequence_for_key = ikey.sequence;
    }
#if 0
    Log(options_.info_log,
        "  Compact: %s, seq %d, type: %d %d, drop: %d, is_base: %d, "
        "%d smallest_snapshot: %d",
        ikey.user_key.ToString().c_str(),
        (int)ikey.sequence, ikey.type, kTypeValue, drop,
        compact->compaction->IsBaseLevelForKey(ikey.user_key),
        (int)last_sequence_for_key, (int)compact->smallest_snapshot);
#endif

    if (!drop) {
      // Open output file if necessary
      if (compact->builder == NULL) {
        status = OpenCompactionOutputFile(compact);
        if (!status.ok()) {
          break;
        }
      }
      if (compact->builder->NumEntries() == 0) {
        compact->current_output()->smallest.DecodeFrom(key);
      }
      compact->current_output()->largest.DecodeFrom(key);
      compact->builder->Add(key, input->value());//把这个键值加入新建的sstable文件中

      // Close output file if it is big enough
      //// 如果当前结果文件已经足够大，则关闭文件，以后的compaction结果再放到新的文件中
      if (compact->builder->FileSize() >=
          compact->compaction->MaxOutputFileSize()) {
        status = FinishCompactionOutputFile(compact, input);
        if (!status.ok()) {
          break;
        }
      }
    }

    input->Next();
  }

  if (status.ok() && shutting_down_.Acquire_Load()) {
    status = Status::IOError("Deleting DB during compaction");
  }
  if (status.ok() && compact->builder != NULL) {
    status = FinishCompactionOutputFile(compact, input);
  }
  if (status.ok()) {
    status = input->status();
  }
  delete input;
  input = NULL;

  CompactionStats stats;
  stats.micros = env_->NowMicros() - start_micros - imm_micros;
  for (int which = 0; which < 2; which++) {
    for (int i = 0; i < compact->compaction->num_input_files(which); i++) {
      stats.bytes_read += compact->compaction->input(which, i)->file_size;
    }
  }
  for (size_t i = 0; i < compact->outputs.size(); i++) {
    stats.bytes_written += compact->outputs[i].file_size;
  }

  mutex_.Lock();
  stats_[compact->compaction->level() + 1].Add(stats);

  if (status.ok()) {
    status = InstallCompactionResults(compact);
  }
  if (!status.ok()) {
    RecordBackgroundError(status);
  }
  VersionSet::LevelSummaryStorage tmp;
  Log(options_.info_log,
      "compacted to: %s", versions_->LevelSummary(&tmp));
  return status;
}
```

## 参考

- [leveldb资料整理 ](https://www.iteye.com/blog/hideto-1328921)
- [leveldb 源码分析(四) – Compaction](https://youjiali1995.github.io/storage/leveldb-compaction/)
- [leveldb源码剖析---DBImpl::MakeRoomForWrite函数的实现](https://blog.csdn.net/swartz2015/article/details/66972106)
- [leveldb源码剖析----compaction](https://blog.csdn.net/Swartz2015/article/details/67633724)


