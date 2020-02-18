---
layout:     post
title:      leveldb之Put
subtitle:   Batch
date:       2020-02-18
author:     BY
header-img: img/post-bg-ios10.jpg
catalog: true
tags:
    - leveldb
---

leveldb 的写操作很简单，先写 WAL，然后写到 memtable

## 源码分析

```objc
DBImpl::Put -> DB::Put -> DBImpl::Write
```
```objc
Status DB::Put(const WriteOptions& opt, const Slice& key, const Slice& value) {
  WriteBatch batch;  
  batch.Put(key, value);//将key-value加入到writebatch中
  return Write(opt, &batch);
}
```

```objc
void WriteBatch::Put(const Slice& key, const Slice& value) {
//Count函数计算当前的writebatch中有多少对key-value
//setCount 将当前的键值对数加1，因为这里新加入了一对键值
  WriteBatchInternal::SetCount(this, WriteBatchInternal::Count(this) + 1);
  //将键值对的type加入到rep末尾
  rep_.push_back(static_cast<char>(kTypeValue));
  //将键值对(包括他们的长度)加入到rep中
  PutLengthPrefixedSlice(&rep_, key);
  PutLengthPrefixedSlice(&rep_, value);
}
```
从Writebatch类的实现中我们知道，writebatch空间布局具有如下形式：

![](https://img-blog.csdn.net/20170327120724033?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvU3dhcnR6MjAxNQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


```objc
Status DBImpl::Write(const WriteOptions& options, WriteBatch* my_batch) {
  Writer w(&mutex_);
  w.batch = my_batch;
  w.sync = options.sync;
  w.done = false;

  MutexLock l(&mutex_);
  writers_.push_back(&w);//将Writer放入到writers_中，Writers_是一个双端队列-deque
  //只有队首的才会执行后续操作,这主要是为了保证每次只有一个消费者对writers_队列进行处理。
  //leveldb将放入队首的生产者选为消费者
  while (!w.done && &w != writers_.front()) {
    w.cv.Wait();
  }
  if (w.done) {
    return w.status;
  }

  // May temporarily unlock and wait.
  Status status = MakeRoomForWrite(my_batch == NULL);//检查内存中的Memtable是否有足够的空间可供写
  uint64_t last_sequence = versions_->LastSequence();//写入的数据的最大序列号
  Writer* last_writer = &w;
  if (status.ok() && my_batch != NULL) {  // NULL batch is for compactions
    WriteBatch* updates = BuildBatchGroup(&last_writer);//生产者队列中的所有任务组合成一个大的任务
    //每个键值对都会对应一个序列号，而且是独一无二的
    WriteBatchInternal::SetSequence(updates, last_sequence + 1);//sequence number记录的就是当前加入leveldb中的键值对个数
    last_sequence += WriteBatchInternal::Count(updates);//Count函数返回writebatch中的key-value对数

    // Add to log and apply to memtable.  We can release the lock
    // during this phase since &w is currently responsible for logging
    // and protects against concurrent loggers and concurrent writes
    // into mem_.
    {
      mutex_.Unlock();
      status = log_->AddRecord(WriteBatchInternal::Contents(updates));
      bool sync_error = false;
      if (status.ok() && options.sync) {
        status = logfile_->Sync();
        if (!status.ok()) {
          sync_error = true;
        }
      }
      if (status.ok()) {
        status = WriteBatchInternal::InsertInto(updates, mem_);//函数把updates里面的所有K-V添加到Memtable中
        //这个地方是不加锁的，因此虽然InsertInto可能会执行较长时间，但是它也不会影响其他生产者线程向队列中添加任务
      }
      mutex_.Lock();
      if (sync_error) {
        // The state of the log file is indeterminate: the log record we
        // just added may or may not show up when the DB is re-opened.
        // So we force the DB into a mode where all future writes fail.
        RecordBackgroundError(status);
      }
    }
    if (updates == tmp_batch_) tmp_batch_->Clear();

    versions_->SetLastSequence(last_sequence);
  }

  while (true) {
    Writer* ready = writers_.front();
    writers_.pop_front();
    if (ready != &w) {
      ready->status = status;
      ready->done = true;//通知相应任务的生产者线程说明它所添加的任务已经处理完毕了
      ready->cv.Signal();
    }
    if (ready == last_writer) break;
  }

  // Notify new head of write queue
  if (!writers_.empty()) {
    writers_.front()->cv.Signal();
  }

  return status;
}

```

## 参考
- [leveldb源码剖析--数据写入(DBImpl::Write)](https://blog.csdn.net/Swartz2015/article/details/66970885)
- [leveldb 源码分析(三) – Write](https://youjiali1995.github.io/storage/leveldb-write/)
