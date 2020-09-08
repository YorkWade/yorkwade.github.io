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

.sst文件，按照Data Block、Filter Block、MetaIndex Block、Index Block、Footer构成。从名字中可以看出，前面四种类型得结构，都是Block，有着相同得结构，Data+CompressionType+CRC,只是内容存储得内容存储得不一样。 最后一个Footer，非Block，总共48字节，记录着这个sst中索引得偏移，根据此信息，可以遍历查找到文件得Data Block。
![](http://img.blog.itpub.net/blog/2019/02/19/5a2031cee73c91fa.jpeg?x-oss-process=style/bb)

### Data Block
很多key可能有重复的字节，比如“hellokitty”和”helloworld“是两个相邻的key，由于key中有公共的部分“hello”，因此，如果将公共的部分提取，可以有效的节省存储空间。

处于这种考虑，LevelDb采用了**前缀压缩(prefix-compressed)**，由于LevelDb中key是按序排列的，这可以显著的减少空间占用。另外，每间隔16个keys(目前版本中options_->block_restart_interval默认为16)，LevelDb就取消使用前缀压缩，而是存储整个key(我们把存储整个key的点叫做重启点，实际也是跳跃表)。

![](https://images2015.cnblogs.com/blog/384029/201612/384029-20161218120252261-1805212835.png)
![](https://images2015.cnblogs.com/blog/384029/201612/384029-20161218120307292-925209927.png)


### Filter Block
如果没有开启布隆过滤器，FilterBlock 这个块就是不存在的。FilterBlock 在一个 SSTable 文件中可以存在多个，每个块存放一个过滤器数据。不过就目前 LevelDB 的实现来说它最多只能有一个过滤器，那就是布隆过滤器。

布隆过滤器用于加快 SSTable 磁盘文件的 Key 定位效率。如果没有布隆过滤器，它需要对 SSTable 进行二分查找，Key 如果不在里面，就需要进行多次 IO 读才能确定，查完了才发现原来是一场空。布隆过滤器的作用就是避免在 Key 不存在的时候浪费 IO 操作。通过查询布隆过滤器可以一次性知道 Key 有没有可能在里面。

单个布隆过滤器中存放的是一个定长的位图数组，该位图数组中存放了若干个 Key 的指纹信息。这若干个 Key 来源于 DataBlock 中连续的一个范围。FilterBlock 块中存在多个连续的布隆过滤器位图数组，每个数组负责指纹化 SSTable 中的一部分数据。
```c++
struct FilterEntry {
  byte[] rawbits;
}

struct FilterBlock {
  FilterEntry[n] filterEntries;
  int32[n] filterEntryOffsets;
  int32 offsetArrayOffset;
  int8 baseLg;  // 分割系数
}
```

其中 baseLg 默认 11，表示每隔 2K 字节（2<<11）的 DataBlock 数据（压缩后），就开启一个布隆过滤器来容纳这一段数据中 Key 值的指纹。如果某个 Value 值过大，以至于超出了 2K 字节，那么相应的布隆过滤器里面就只有 1 个 Key 值的指纹。每个 Key 对应的指纹空间在打开数据库时指定。

// 每个 Key 占用 10bit 存放指纹信息
options.SetFilterPolicy(levigo.NewBloomFilter(10))


这里的 2K 字节的间隔是严格的间隔，这样才可以通过 DataBlock 的偏移量和大小来快速定位到相应的布隆过滤器的位置 FilterOffset，再进一步获得相应的布隆过滤器位图数据。

![](http://img.blog.itpub.net/blog/2019/02/19/0b6a648e2f7140ff.jpeg?x-oss-process=style/bb)

Filter Block也就是前面sstable中的meta block，位于data block之后。
如果打开db时指定了FilterPolicy，那么每个创建的table都会保存一个filter block，table中的metaindex就包含一条从”filter.<N>到filter block的BlockHandle的映射，其中”<N>”是filter policy的Name()函数返回的string。
Filter block存储了一连串的filter值，其中第i个filter保存的是block b中所有的key通过FilterPolicy::CreateFilter()计算得到的结果，block b在sstable文件中的偏移满足[ i*base ... (i+1)*base-1 ]。

当前base是2KB，举个例子，如果block X和Y在sstable的起始位置都在[0KB, 2KB-1]中，X和Y中的所有key调用FilterPolicy::CreateFilter()的计算结果都将生产到同一个filter中，而且该filter是filter block的第一个filter。

Filter block也是一个block，其格式遵从block的基本格式：|block data| type | crc32|。

Filter Block相关源码，参考[FilterPolicy&Bloom之2](https://blog.csdn.net/sparkliang/article/details/8755499)

### MetaIndex Block 
MetaIndexBlock 存储了前面一系列 FilterBlock 的元信息，它在结构上和 DataBlock 是一样的，只不过里面 Entry 存储的 Key 是带固定前缀的过滤器名称，Value 是对应的 FilterBlock 在文件中的偏移量和长度。
```c++
key = "filter." + filterName
// value 定义了数据块的位置和大小
struct BlockHandler {
  varint offset;
  varint size;
}
```

就目前的 LevelDB，这里面最多只有一个 Entry，那么它的结构非常简单，如下图所示

![](http://img.blog.itpub.net/blog/2019/02/19/c445b5bbefed183d.jpeg?x-oss-process=style/bb)

### Index Block
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
![](http://img.blog.itpub.net/blog/2019/02/19/2b6d73036dc4a1d4.jpeg?x-oss-process=style/bb)

### Footer
它的占用空间很小只有 48 字节，内部只存了几个字段。下面我们用伪代码来描述一下它的结构
```obj
// 定义了数据块的位置和大小
struct BlockHandler {
  varint offset;
  varint size;
}

struct Footer {
  BlockHandler metaIndexHandler;  // MetaIndexBlock的文件偏移量和长度
  BlockHandler indexHandler; // IndexBlock的文件偏移量和长度
  byte[n] padding;  // 内存垫片
  int32 magicHighBits;  // 魔数后32位
  int32 magicLowBits; // 魔数前32位
}
```

Footer 结构的中间部分增加了内存垫片，其作用就是将 Footer 的空间撑到 48 字节。结构的尾部还有一个 64位的魔术数字 0xdb4775248b80fb57，如果文件尾部的 8 字节不是这个数字说明文件已经损坏。这个魔术数字的来源很有意思，它是下面返回的字符串的前64bit。
```obj
$ echo http://code.google.com/p/leveldb/ | sha1sum
db4775248b80fb57d0ce0768d85bcee39c230b61
```

IndexBlock 和 MetaIndexBlock 都只有唯一的一个，所以分别使用一个 BlockHandler 结构来存储偏移量和长度。

![](http://img.blog.itpub.net/blog/2019/02/19/0b6a648e2f7140ff.jpeg?x-oss-process=style/bb)

### 物理结构
除了 Footer 之外，其它部分都是 Block 结构，在名称上也都是以 Block 结尾。所谓的 Block 结构是指除了内部的有效数据外，还会有额外的压缩类型字段和校验码字段。
```obj
struct Block {
  byte[] data;
  int8 compressType;
  int32 crcValue;
}
```

每一个 Block 尾部都会有压缩类型和循环冗余校验码（crcValue），这会要占去 5 字节。如果是压缩类型，块内的数据 data 会被压缩。校验码会针对压缩和的数据和压缩类型字段一起计算循环冗余校验和。压缩算法默认是 snappy ，校验算法是 crc32。


## 源码
源码基本按照上述规则填写，并无技巧。参考中的四篇博文，已经解析很详细了。
```obj
Status TableBuilder::Finish() {
  Rep* r = rep_;
  Flush();
  assert(!r->closed);
  r->closed = true;

  BlockHandle filter_block_handle, metaindex_block_handle, index_block_handle;

  // Write filter block
  if (ok() && r->filter_block != NULL) {
    WriteRawBlock(r->filter_block->Finish(), kNoCompression,
                  &filter_block_handle);
  }

  // Write metaindex block
  if (ok()) {
    BlockBuilder meta_index_block(&r->options);
    if (r->filter_block != NULL) {
      // Add mapping from "filter.Name" to location of filter data
      std::string key = "filter.";
      key.append(r->options.filter_policy->Name());
      std::string handle_encoding;
      filter_block_handle.EncodeTo(&handle_encoding);
      meta_index_block.Add(key, handle_encoding);
    }

    // TODO(postrelease): Add stats and other meta blocks
    WriteBlock(&meta_index_block, &metaindex_block_handle);
  }

  // Write index block
  if (ok()) {
    if (r->pending_index_entry) {
      r->options.comparator->FindShortSuccessor(&r->last_key);
      std::string handle_encoding;
      r->pending_handle.EncodeTo(&handle_encoding);
      r->index_block.Add(r->last_key, Slice(handle_encoding));
      r->pending_index_entry = false;
    }
    WriteBlock(&r->index_block, &index_block_handle);
  }

  // Write footer
  if (ok()) {
    Footer footer;
    footer.set_metaindex_handle(metaindex_block_handle);
    footer.set_index_handle(index_block_handle);
    std::string footer_encoding;
    footer.EncodeTo(&footer_encoding);
    r->status = r->file->Append(footer_encoding);
    if (r->status.ok()) {
      r->offset += footer_encoding.size();
    }
  }
  return r->status;
}
```
>参考：
- [SSTable之1](https://blog.csdn.net/sparkliang/article/details/8635821)
- [SSTable之2](https://blog.csdn.net/sparkliang/article/details/8653370)
- [SSTable之3](https://blog.csdn.net/sparkliang/article/details/8681759)
- [SSTable之4](https://blog.csdn.net/sparkliang/article/details/8708892)
- [FilterPolicy&Bloom之2](https://blog.csdn.net/sparkliang/article/details/8755499)
- [深入 LevelDB 数据文件 SSTable 的结构](http://blog.itpub.net/31561269/viewspace-2636368/)
- [LevelDB的sstable解读](https://blog.csdn.net/qq_22054285/article/details/85757341)

