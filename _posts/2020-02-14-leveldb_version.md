---
layout:     post
title:      leveldb之版本控制
subtitle:   
date:       2020-02-14
author:     BY
header-img: img/post-bg-ios10.jpg
catalog: true
tags:
    - leveldb
---


## 前言
每次leveldb后台进行compact时， 会造成sst文件的变化，此时会存在下面问题

    leveldb 怎么管理因compact带来的文件变化？
    当db关闭后重新打开时，如何恢复状态？
    如何解决版本销毁文件变化和已经获取过的迭代器的冲突？在查询数据的时候是否会存在被查文件被删掉的可能呢？
 
 leveldb使用版本控制（元数据管理）进行支持的。外部用户可以以快照的方式使用文件和数据。

LeveDB用Version表示一个版本的元信息，Version中主要包括一个FileMetaData指针的二维数组，分层记录了所有的SST文件信息。FileMetaData数据结构用来维护一个文件的元信息，包括文件大小，文件编号，最大最小值，引用计数等，其中引用计数记录了被不同的Version引用的个数，保证被引用中的文件不会被删除。除此之外，Version中还记录了触发Compaction相关的状态信息，这些信息会在读写请求或Compaction过程中被更新。通过庖丁解LevelDB之概览中对Compaction过程的描述可以知道在CompactMemTable和BackgroundCompaction过程中会导致新文件的产生和旧文件的删除。每当这个时候都会有一个新的对应的Version生成，并插入VersionSet链表头部。
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

VersionSet是一个Version构成的双向链表，这些Version按时间顺序先后产生，记录了当时的元信息，链表头指向当前最新的Version，同时维护了每个Version的引用计数，被引用中的Version不会被删除，其对应的SST文件也因此得以保留，通过这种方式，使得LevelDB可以在一个稳定的快照视图上访问文件。VersionSet中除了Version的双向链表外还会记录一些如LogNumber，Sequence，下一个SST文件编号的状态信息。

![](https://img-blog.csdn.net/20150514163342237?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMjY1ODM0Ng==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![](https://image-static.segmentfault.com/515/820/515820936-59391361c779e_articlex)

VersionSet Version 示意图

通过上面的描述可以看出，相邻Version之间的不同仅仅是一些文件被删除另一些文件被删除。也就是说将文件变动应用在旧的Version上可以得到新的Version，这也就是Version产生的方式。LevelDB用VersionEdit来表示这种相邻Version的差值。

VersionEidt

为了避免进程崩溃或机器宕机导致的数据丢失，LevelDB需要将元信息数据持久化到磁盘，承担这个任务的就是Manifest文件。可以看出每当有新的Version产生都需要更新Manifest，很自然的发现这个新增数据正好对应于VersionEdit内容，也就是说Manifest文件记录的是一组VersionEdit值，在Manifest中的一次增量内容称作一个Block，其内容如下：

![](https://image-static.segmentfault.com/600/545/600545784-59391361dc06d_articlex)


## 参考
- [庖丁解LevelDB之版本控制](https://catkang.github.io/2017/02/03/leveldb-version.html)
- [Leveldb二三事](https://segmentfault.com/a/1190000009707717?utm_source=tag-newest)
-
