---
layout:     post
title:      leveldb之版本控制
subtitle:   
date:       2020-02-13
author:     BY
header-img: img/post-bg-ios10.jpg
catalog: true
tags:
    - leveldb
---


## leveldb版本控制原理
每次leveldb后台进行compact时， 会造成sst文件的变化，此时会存在下面问题

    leveldb 怎么管理因compact带来的文件变化？
    当db关闭后重新打开时，如何恢复状态？
    如何解决版本销毁文件变化和已经获取过的迭代器的冲突？在查询数据的时候是否会存在被查文件被删掉的可能呢？
 
当执行一次compaction后，Leveldb将在当前版本基础上创建一个新版本，当前版本就变成了历史版本。还有，如果你创建了一个Iterator，那么该Iterator所依附的版本将不会被leveldb删除。
在leveldb中，Version就代表了一个版本，它包括当前磁盘及内存中的所有文件信息。在所有的version中，只有一个是CURRENT。
VersionSet是所有Version的集合，这是个version的管理机构。
VersionEdit记录了Version之间的变化，相当于delta增量，表示又增加了多少文件，删除了文件。也就是说：Version0 + VersionEdit --> Version1。
每次文件有变动时，leveldb就把变动记录到一个VersionEdit变量中，然后通过VersionEdit把变动应用到current version上，并把current version的快照，也就是db元信息保存到MANIFEST文件中。
另外，MANIFEST文件组织是以VersionEdit的形式写入的，它本身是一个log文件格式，采用log::Writer/Reader的方式读写，一个VersionEdit就是一条log record。

### Version
LeveDB用Version表示当前磁盘及内存中的所有文件元信息，是当时版本的 sstable 组织结构，只有每次 compaction 完成时才会改变 sstable 的 组织结构并增加一个新的 Version。Version中主要包括一个FileMetaData指针的二维数组，分层记录了所有的SST文件信息。FileMetaData数据结构用来维护一个文件的元信息，包括文件大小，文件编号，最大最小值，引用计数等，其中引用计数记录了被不同的Version引用的个数，保证被引用中的文件不会被删除。除此之外，Version中还记录了触发Compaction相关的状态信息，这些信息会在读写请求或Compaction过程中被更新。每当这个时候都会有一个新的对应的Version生成，并插入VersionSet链表头部。Version通过Version* prev和*next指针构成了一个Version双向循环链表，表头指针则在VersionSet中（初始都指向自己）。
```objc
class Version {
 private:
  VersionSet* vset_; // VersionSet to which this Version belongs
  Version* next_;    // next_、prev形成双向链表
  Version* prev_;               
  int refs_;                    
 
  std::vector<FileMetaData*> files_[config::kNumLevels];//二维数组，记录所有level的全部FileMetaData文件
 
  // 基于Seek的结果stats得到的下一个等待Compact的文件（当FileMetaData文件查找到一定次数时，就需要执行合并操作）
  FileMetaData* file_to_compact_;
  int file_to_compact_level_;//需要进行合并的文件所属的level
 
  double compaction_score_;//当score>=1时，也需要进行合并
  int compaction_level_;//需要进行合并的level
};
```

![](https://img-blog.csdn.net/20150514155705427?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjY1ODM0Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
### VersionSet
VersionSet是Version集合，所有的Version都挂在VersionSet对象下面，一个db只有一个VersionSet。VersionSet是一个Version构成的双向链表，这些Version按时间顺序先后产生，记录了当时的元信息，链表头指向当前最新的Version，同时维护了每个Version的引用计数，被引用中的Version不会被删除，其对应的SST文件也因此得以保留，通过这种方式，使得LevelDB可以在一个稳定的快照视图上访问文件。VersionSet中除了Version的双向链表外还会记录一些如LogNumber，Sequence，下一个SST文件编号的状态信息。

```objc
class VersionSet {
 public:
  ...
 private:
  class Builder;
 
  void SetupOtherInputs(Compaction* c);
  Status WriteSnapshot(log::Writer* log);
  void AppendVersion(Version* v);
 
  Env* const env_;
  const std::string dbname_;
  const Options* const options_;
  TableCache* const table_cache_;//缓存部分SSTable
  const InternalKeyComparator icmp_;
  uint64_t next_file_number_;   //下一个要生成的文件的编号
  uint64_t manifest_file_number_;
  uint64_t last_sequence_;
  uint64_t log_number_;
  uint64_t prev_log_number_;  // 0 or backing store for memtable being compacted
 
  // Opened lazily
  WritableFile* descriptor_file_;//数据库的Manifest清单文件
  log::Writer* descriptor_log_;//用于写Manifest文件
  Version dummy_versions_;  // 所有Version形成的双向链表的头部
  Version* current_;        // == dummy_versions_.prev_，双向链表的尾部，即最新的Version
 
  std::string compact_pointer_[config::kNumLevels];//每一level下次要执行合并操作的起始Key值
};

```

![](https://img-blog.csdn.net/20150514163342237?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjY1ODM0Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](https://image-static.segmentfault.com/515/820/515820936-59391361c779e_articlex)

VersionSet Version 示意图

通过上面的描述可以看出，相邻Version之间的不同仅仅是一些文件被删除另一些文件被删除。也就是说将文件变动应用在旧的Version上可以得到新的Version，这也就是Version产生的方式。LevelDB用VersionEdit来表示这种相邻Version的差值。

![](https://youjiali1995.github.io/assets/images/leveldb/version_edit.png)

### VersionEidt

为了避免进程崩溃或机器宕机导致的数据丢失，LevelDB需要将元信息数据持久化到磁盘，承担这个任务的就是Manifest文件。可以看出每当有新的Version产生都需要更新Manifest，很自然的发现这个新增数据正好对应于VersionEdit内容，也就是说Manifest文件记录的是一组VersionEdit值，在Manifest中的一次增量内容称作一个Block，其内容如下：

![](https://img-my.csdn.net/uploads/201304/09/1365478054_1495.JPG)

```objc
class VersionEdit {
 public:
 ...

 private:
  friend class VersionSet;

  typedef std::set< std::pair<int, uint64_t> > DeletedFileSet;

  std::string comparator_;
  uint64_t log_number_;
  uint64_t prev_log_number_;
  uint64_t next_file_number_;
  SequenceNumber last_sequence_;
  bool has_comparator_;
  bool has_log_number_;
  bool has_prev_log_number_;
  bool has_next_file_number_;
  bool has_last_sequence_;

  std::vector< std::pair<int, InternalKey> > compact_pointers_;
  DeletedFileSet deleted_files_;
  std::vector< std::pair<int, FileMetaData> > new_files_;
};
```
可以看出恢复元信息的过程也变成了依次应用VersionEdit的过程，这个过程中有大量的中间Version产生，但这些并不是我们所需要的。LevelDB引入VersionSet::Builder来避免这种中间变量，方法是先将所有的VersoinEdit内容整理到VersionBuilder中，然后一次应用产生最终的Version，这种实现上的优化如下图所示：

![](https://image-static.segmentfault.com/600/545/600545784-59391361dc06d_articlex)


## 触发版本变化的主要有两个地方

    将memtable写入磁盘，生成新的sstable
    compaction操作

### memtable写盘
下面我们分别看一下在这两个操作下，版本信息是怎么进行管理的。
这个过程主要是在DBImpl::CompactMemTable函数中完成
以下我们摘取这个函数的部分代码分析一下
```objc
  VersionEdit edit;
  Version* base = versions_->current();
  base->Ref();
  Status s = WriteLevel0Table(imm_, &edit, base);
  base->Unref();
```
这里主要是申请了一个VersionEdit变量，并把他传入到WriteLevel0Table函数中，用于记录新生成的sstable文件信息。我们追踪到WriteLevel0Table函数里面就可以看到下面的代码：
```objc
edit->AddFile(level, meta.number, meta.file_size,
                  meta.smallest, meta.largest);
```
这个函数将新生成的sstable文件的元信息加入到VersionEdit的new_files_中。

再回到CompactMemTable函数：
```objc
  if (s.ok()) {
    edit.SetPrevLogNumber(0);
    edit.SetLogNumber(logfile_number_);  // Earlier logs no longer needed
    s = versions_->LogAndApply(&edit, &mutex_); 
  }
```


这个就是版本管理的关键了。其中LogAndApply函数将根据前面记录的版本修改信息更新当前版本，得到一个新的版本信息。同时把这个新的版本信息设置成当前VersionSet的current_，并连入版本集合的链表中。

这个函数我们后面再介绍，这里我们只需要知道根据修改信息生成新的版本就可以了。

可以看到，对于CompactMemTable，版本的修改信息很简单，就是在new_files_里面添加一个新的文件元信息。
### compaction操作

这个工作主要是在函数DBImpl::BackgroundCompaction中完成。前面我们分析的时候知道，对level和level+1中的文件进行compaction时会有两中情况

    当level+1中没有文件和level中的那个待compaction的文件的key范围重合时，此时直接将level中的那个文件移动到level+1中的那个文件即可。
    level+1中有文件和level中的那个文件key范围重合。则进行常规的compaction。

下面我们分别从这两种情况看一下版本管理流程。
```objc
else if (!is_manual && c->IsTrivialMove()) { //当input[0](level)中只有一个需要compaction的文件，input[1](level+1)中没有需要compaction的文件
    // Move file to next level
    assert(c->num_input_files(0) == 1);
    FileMetaData* f = c->input(0, 0); // 将 f 从 c->level移动到 c->level + 1中即可

    c->edit()->DeleteFile(c->level(), f->number);
    c->edit()->AddFile(c->level() + 1, f->number, f->file_size,
                       f->smallest, f->largest);
    status = versions_->LogAndApply(c->edit(), &mutex_); //只需要在edit中记录就可以了

```
对于第一种情况，VersionEdit将level中的那个文件的元信息删除。同时把这个元信息移动到level+1中，最后调用LogAndApply更新当前版本信息。

对于第二种情况，主要是由DoCompactionWork函数中InstallCompactionResults函数的完成。

BackgroundCompaction -> DoCompactionWork -> InstallCompactionResults

下面分析一个InstallCompactionResults函数：
```objc
    compact->compaction->AddInputDeletions(compact->compaction->edit());
    const int level = compact->compaction->level();
```
前面我们分析过compaction->input数组中包含了本次compaction操作所需要的所有sstable文件信息，同时compaction->output中包含了所有新生成的文件信息，因此我们很自然的想法是将inputs_中的文件信息加入到VersionEdit的deleted_files中，而将outputs中的文件信息加入到它的new_files_中，InstallCompactionResults正是这么做的。上面的AddInputDeletions函数就是将input_的文件信息加入到edit的deleted_files中，而下面的for循环
```objc
  for (size_t i = 0; i < compact->outputs.size(); i++) {
    const CompactionState::Output& out = compact->outputs[i];
    compact->compaction->edit()->AddFile(
        level + 1,
        out.number, out.file_size, out.smallest, out.largest);
  }
```
则将output_中的文件信息加入到new_files数组中。
```objc
versions_->LogAndApply(compact->compaction->edit(), &mutex_);
```
最后一如既往调用LogAndApply函数将修改信息更新到当前版本，生成一个新的版本信息


那么将如何从旧版本生成新版本了？看下下段VersionSet::LogAndApply的代码：
```objc
Status VersionSet::LogAndApply(VersionEdit* edit, port::Mutex* mu) {
  ...
  Version* v = new Version(this);
  {
    Builder builder(this, current_);
    builder.Apply(edit);  //将版本的变化即VersionEdit应用到VersionSet的compact_pointers, deleted_files及added_files。
    builder.SaveTo(v);//根据VersionSet中的deleted_files以及added_files, 从base Version(即current_)生成新Version的数据内容
  }
  Finalize(v);
  
   // Initialize new descriptor log file if necessary by creating
  // a temporary file that contains a snapshot of the current version.
  std::string new_manifest_file;
  Status s;
  if (descriptor_log_ == NULL) {//如果MANIFEST文件指针不存在，就创建并初始化一个新的MANIFEST文件。这只会发生在第一次打开数据库时。
    // No reason to unlock *mu here since we only hit this path in the
    // first call to LogAndApply (when opening the database).
    assert(descriptor_file_ == NULL);
    new_manifest_file = DescriptorFileName(dbname_, manifest_file_number_);
    edit->SetNextFile(next_file_number_);
    s = env_->NewWritableFile(new_manifest_file, &descriptor_file_);
    if (s.ok()) {
      descriptor_log_ = new log::Writer(descriptor_file_);
      s = WriteSnapshot(descriptor_log_);//这个MANIFEST文件保存了current version的快照。
    }
  }
   // Unlock during expensive MANIFEST log write
  {
    mu->Unlock();

    // Write new record to MANIFEST log
    if (s.ok()) {
      std::string record;
      edit->EncodeTo(&record);
      s = descriptor_log_->AddRecord(record);//将VersionEdit写入MANIFEST中
      if (s.ok()) {
        s = descriptor_file_->Sync();
      }
      if (!s.ok()) {
        Log(options_->info_log, "MANIFEST write: %s\n", s.ToString().c_str());
      }
    }

    // If we just created a new descriptor file, install it by writing a
    // new CURRENT file that points to it.
    if (s.ok() && !new_manifest_file.empty()) {
      s = SetCurrentFile(env_, dbname_, manifest_file_number_);//设置成Current
    }

    mu->Lock();
}
```

LogAndApply。这个函数主要是当一个Version结束时，将这个Version所做的修改(VersionEdit里)保存到一个新的Version中(VersionSet以后就使用这个Version了)，然后持久化到MANIFEST文件中(WriteSnapshot或者直接进行log record)，并将current指向最新的Version。

```objc
void SaveTo(Version* v) {
    BySmallestKey cmp;
    cmp.internal_comparator = &vset_->icmp_;
    for (int level = 0; level < config::kNumLevels; level++) {
      // Merge the set of added files with the set of pre-existing files.
      // Drop any deleted files.  Store the result in *v.
      const std::vector<FileMetaData*>& base_files = base_->files_[level];//一个根据根据file的最小key值排序的有序vector
      std::vector<FileMetaData*>::const_iterator base_iter = base_files.begin();
      std::vector<FileMetaData*>::const_iterator base_end = base_files.end();
      const FileSet* added = levels_[level].added_files; //FileSet是一个根据file的最小key值排序的有序set
      v->files_[level].reserve(base_files.size() + added->size());
　　　　　　//如下过程类似于归并排序
　　　　　　//base files:{2}, {5}, {7}
          //add files:{1}, {6}
          //过程就是{1}->files,  {2}, {5}->files, {6}->files, {7}->files.
          //当然，上步骤的过程中，还需要判断下该file是否能add到files中去，即MaybeAddFile（不在删除files里即可添加）
      for (FileSet::const_iterator added_iter = added->begin();
           added_iter != added->end();
           ++added_iter) {
        // Add all smaller files listed in base_
        for (std::vector<FileMetaData*>::const_iterator bpos
                 = std::upper_bound(base_iter, base_end, *added_iter, cmp);
             base_iter != bpos;
             ++base_iter) {
          MaybeAddFile(v, level, *base_iter);
        }

        MaybeAddFile(v, level, *added_iter);
      }

      // Add remaining base files
      for (; base_iter != base_end; ++base_iter) {
        MaybeAddFile(v, level, *base_iter);
      }

    }
  }
```
## 总结

一个健壮的数据库系统，版本信息是必不可少的。leveldb的版本信息主要是记录数据库的文件信息。随着compaction操作和memtable的写盘，数据库中的文件会不断发生变化，因此对于每次可能生成新的sstable文件或者可能删除已有的sstable文件的操作，都必须进行版本更新。这里我们从以上两个操作来跟踪了解了leveldb的版本更新工作流程。leveldb通过versionedit记录版本的修改信息，然后通过LogAndApply函数将这个修改信息更新到当前的版本(current_)，生成新的版本信息。leveldb还通过versionSet将从系统启动开始经历的所有版本信息放在一个集合中进行管理，这个集合是用链表支持的，最新的版本总是Append到链表的尾部

## 参考

- [leveldb源码剖析---版本管理](https://blog.csdn.net/Swartz2015/article/details/68062639)
- [庖丁解LevelDB之版本控制](https://catkang.github.io/2017/02/03/leveldb-version.html)
- [Leveldb二三事](https://segmentfault.com/a/1190000009707717?utm_source=tag-newest)
- [leveldb version机制](https://www.cnblogs.com/ewouldblock7/p/3721088.html)
- [Leveldb源码分析--15](https://blog.csdn.net/sparkliang/article/details/8776583)

