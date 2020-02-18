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


## 参考
- [leveldb源码剖析--数据写入(DBImpl::Write)](https://blog.csdn.net/Swartz2015/article/details/66970885)
- [leveldb 源码分析(三) – Write](https://youjiali1995.github.io/storage/leveldb-write/)
