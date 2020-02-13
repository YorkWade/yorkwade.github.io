---
layout:     post
title:      leveldb之SSTable读写
subtitle:   文件序列化设计
date:       2020-02-12
author:     BY
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - leveldb
---

> leveldb 的SSTable的文件如何设计。

## 设计要点
### Data Block设计
很多key可能有重复的字节，比如“hellokitty”和”helloworld“是两个相邻的key，由于key中有公共的部分“hello”，因此，如果将公共的部分提取，可以有效的节省存储空间。

处于这种考虑，LevelDb采用了前缀压缩(prefix-compressed)，由于LevelDb中key是按序排列的，这可以显著的减少空间占用。另外，每间隔16个keys(目前版本中options_->block_restart_interval默认为16)，LevelDb就取消使用前缀压缩，而是存储整个key(我们把存储整个key的点叫做重启点，实际也是跳跃表)。

### Index Block设计
典型的Data Block大小为4KB，而sstable文件的典型大小为2MB，也就说，一个sstable中，存在很多data block块，如果用户要查找某个key，该key到底位于which data block?

如果没有index block，只有data block的起始位置和长度 （这个信息一定要有，否则无法确定查找，因为data block是变长的，并不是固定大小的），当用户查找某个key的时候，尽管是数据是有序的，可是还是不得不线性地找遍所有data block。任何一次查找要不得不读取2MB左右的内容，势必造成性能的恶化。

（其实也可以采用瞎猜算法，比如有256个data block的位置信息，先去找第128个 data block，判断第一个key和要找的key的关系，来二分查找。但是这种思路已经非常逼近index block的实现了。）

上帝说要有光，所有就有了光。Leveldb说要有索引，所有就有了index block。

index block的组织形式 data block的组织形式是一模一样的，不同点的地方在于存储的内容。data block存放的客户输入的key-value，而 index block存放的是 索引，SSTable 中有几个 DataBlock，IndexBlock 中就有几个键值对。

如果上一个data block的最后一个key是 “helloleveldb”， 而当前data block的第一个key是“helloworld ”，那么index block就会在新block插入第一个key “helloworld”的时候，计算出两个data block之间的分割key（比如leveldb中算出来的hellom）。比分割key（hellom）小的key，在上一个data block，比分割key(hellom)大的key在下一个data block，因此
```obj
key = 分割key
value = 上一个data block的（offset，size）
```
通过这种手段，可以快速地定位出我们要找的key位于哪个data block(当然也可能压根不存在)

	
	
>参考：
- [SSTable之1](https://blog.csdn.net/sparkliang/article/details/8635821)
- [SSTable之2](https://blog.csdn.net/sparkliang/article/details/8653370)
- [SSTable之3](https://blog.csdn.net/sparkliang/article/details/8681759)
- [SSTable之4](https://blog.csdn.net/sparkliang/article/details/8708892)
- [深入 LevelDB 数据文件 SSTable 的结构](http://blog.itpub.net/31561269/viewspace-2636368/)
