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
 
 leveldb使用版本控制（元数据管理）进行支持的。外部用户可以以快照的方式使用文件和数据。
### Version
LeveDB用Version表示一个版本的元信息，Version中主要包括一个FileMetaData指针的二维数组，分层记录了所有的SST文件信息。FileMetaData数据结构用来维护一个文件的元信息，包括文件大小，文件编号，最大最小值，引用计数等，其中引用计数记录了被不同的Version引用的个数，保证被引用中的文件不会被删除。除此之外，Version中还记录了触发Compaction相关的状态信息，这些信息会在读写请求或Compaction过程中被更新。每当这个时候都会有一个新的对应的Version生成，并插入VersionSet链表头部。
```obj
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

```obj
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
```obj
Manifest Block := N * Item
Item := [kComparator] comparator
		or [kLogNumber] 64位log_number
		or [kPrevLogNumber] 64位pre_log_number
		or [kNextFileNumber] 64位next_file_number_
		or [kLastSequence] 64位last_sequence_
		or [kCompactPointer] 32位level + 变长的key
		or [kDeletedFile] 32位level + 64位文件号
		or [kNewFile] 32位level + 64位 文件号 + 64位文件长度 + smallest key + largest key
```
```obj
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

那么将如何从旧版本生成新版本了？看下下段VersionSet::LogAndApply的代码：
```obj
Status VersionSet::LogAndApply(VersionEdit* edit, port::Mutex* mu) {
  ...
  Version* v = new Version(this);
  {
    Builder builder(this, current_);
    builder.Apply(edit);  //将版本的变化即VersionEdit应用到VersionSet的compact_pointers, deleted_files及added_files。
    builder.SaveTo(v);//根据VersionSet中的deleted_files以及added_files, 从base Version(即current_)生成新Version的数据内容
  }
  Finalize(v);
  ...
}
```

```obj
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

## 参考
- [庖丁解LevelDB之版本控制](https://catkang.github.io/2017/02/03/leveldb-version.html)
- [Leveldb二三事](https://segmentfault.com/a/1190000009707717?utm_source=tag-newest)
- [leveldb version机制](https://www.cnblogs.com/ewouldblock7/p/3721088.html)
