---
layout:     post
title:      leveldb之Log读写
subtitle:   文件映射
date:       2020-02-10
author:     BY
header-img: img/post-bg-mma-3.jpg
catalog: true
tags:
    - leveldb
---
>为了防止写入内存的数据库因为进程异常、系统掉电等情况发生丢失，leveldb在写内存之前会将本次写操作的内容写入日志文件中。


## 理论


在leveldb中，有两个memory db，以及对应的两份日志文件。其中一个memory db是可读写的，当这个db的数据量超过预定的上限时，便会转换成一个不可读的memory db，与此同时，与之对应的日志文件也变成一份frozen log。

而新生成的immutable memory db则会由后台的minor compaction进程将其转换成一个sstable文件进行持久化，持久化完成，与之对应的frozen log被删除。
![](https://leveldb-handbook.readthedocs.io/zh/latest/_images/two_log.jpeg)

### 日志结构
为了增加读取效率，日志文件中按照block进行划分，每个block的大小为32KiB。每个block中包含了若干个完整的chunk。

一条日志记录包含一个或多个chunk。每个chunk包含了一个7字节大小的header，前4字节是该chunk的校验码，紧接的2字节是该chunk数据的长度，以及最后一个字节是该chunk的类型。其中checksum校验的范围包括chunk的类型以及随后的data数据。

chunk共有四种类型：full，first，middle，last。一条日志记录若只包含一个chunk，则该chunk的类型为full。若一条日志记录包含多个chunk，则这些chunk的第一个类型为first, 最后一个类型为last，中间包含大于等于0个middle类型的chunk。

由于一个block的大小为32KiB，因此当一条日志文件过大时，会将第一部分数据写在第一个block中，且类型为first，若剩余的数据仍然超过一个block的大小，则第二部分数据写在第二个block中，类型为middle，最后剩余的数据写在最后一个block中，类型为last。
![](https://izualzhy.cn/assets/images/leveldb/log_format.png)



## 实现要点

leveldb使用[Windws内存映射文件](http://blog.tk-xiong.com/archives/933)来实现（来自Windows核心编程 – 第十七章 第三节 – 使用内存映射文件）<br>

要使用内存映射文件，需要执行下面三个步骤：<br>
    1、创建或打开一个文件内核对象，该对象标识了我们想要用作内存映射文件的那个磁盘文件。<br>
    2、创建一个文件映射内核对象来告诉系统文件的大小以及我们打算如何访问文件。<br>
    3、高速系统把文件映射对象的部分或全部映射到进程的地址空间中。<br>
用完内存映射文件后，必须执行下面三个步骤来做清理工作：<br>
    1、告诉系统从进程地址空间中取消对文件映射内核对象的映射<br>
    2、关闭文件映射内核对象<br>
    3、关闭文件内核对象<br>
主要代码如下：
``` objc
HANDLE hFile = CreateFile(...);
HANDLE hFileMapping = CreateFileMapping(hFile, ...);
PVOID pvFile = MapViewOfFile(hFileMapping, ...);
 
//Use the memory-mapped file.
 
UnmapViewOfFile(pvFile);
CloseHandle(hFileMapping);
CloseHandle(hFile);
```

## 源码分析
```objc
class Writer {
 public:
  // Create a writer that will append data to "*dest".
  // "*dest" must be initially empty.
  // "*dest" must remain live while this Writer is in use.
  explicit Writer(WritableFile* dest);
  ~Writer();

  Status AddRecord(const Slice& slice);

 private:
  WritableFile* dest_;
  int block_offset_;       // Current offset in block

  // crc32c values for all supported record types.  These are
  // pre-computed to reduce the overhead of computing the crc of the
  // record type stored in the header.
  uint32_t type_crc_[kMaxRecordType + 1];

  Status EmitPhysicalRecord(RecordType type, const char* ptr, size_t length);

  // No copying allowed
  Writer(const Writer&);
  void operator=(const Writer&);
};
```

```objc
Status Writer::AddRecord(const Slice& slice) {
  const char* ptr = slice.data();
  size_t left = slice.size();

  // Fragment the record if necessary and emit it.  Note that if slice
  // is empty, we still want to iterate once to emit a single
  // zero-length record
  Status s;
  bool begin = true;
  do {
    const int leftover = kBlockSize - block_offset_;
    assert(leftover >= 0);
    if (leftover < kHeaderSize) {
      // Switch to a new block
      if (leftover > 0) {
        // Fill the trailer (literal below relies on kHeaderSize being 7)
        assert(kHeaderSize == 7);
        dest_->Append(Slice("\x00\x00\x00\x00\x00\x00", leftover));
      }
      block_offset_ = 0;
    }

    // Invariant: we never leave < kHeaderSize bytes in a block.
    assert(kBlockSize - block_offset_ - kHeaderSize >= 0);

    const size_t avail = kBlockSize - block_offset_ - kHeaderSize;
    const size_t fragment_length = (left < avail) ? left : avail;

    RecordType type;
    const bool end = (left == fragment_length);
    if (begin && end) {
      type = kFullType;
    } else if (begin) {
      type = kFirstType;
    } else if (end) {
      type = kLastType;
    } else {
      type = kMiddleType;
    }

    s = EmitPhysicalRecord(type, ptr, fragment_length);
    ptr += fragment_length;
    left -= fragment_length;
    begin = false;
  } while (s.ok() && left > 0);
  return s;
}
```


```objc
Status Writer::EmitPhysicalRecord(RecordType t, const char* ptr, size_t n) {
  assert(n <= 0xffff);  // Must fit in two bytes
  assert(block_offset_ + kHeaderSize + n <= kBlockSize);

  // Format the header
  char buf[kHeaderSize];
  buf[4] = static_cast<char>(n & 0xff);
  buf[5] = static_cast<char>(n >> 8);
  buf[6] = static_cast<char>(t);

  // Compute the crc of the record type and the payload.
  uint32_t crc = crc32c::Extend(type_crc_[t], ptr, n);
  crc = crc32c::Mask(crc);                 // Adjust for storage
  EncodeFixed32(buf, crc);

  // Write the header and the payload
  Status s = dest_->Append(Slice(buf, kHeaderSize));
  if (s.ok()) {
    s = dest_->Append(Slice(ptr, n));
    if (s.ok()) {
      s = dest_->Flush();
    }
  }
  block_offset_ += kHeaderSize + n;
  return s;
}
```

leveldb使用文件映射来进行日志记录
```objc
class Win32MapFile : public WritableFile
{
public:
    Win32MapFile(const std::string& fname);

    ~Win32MapFile();
    virtual Status Append(const Slice& data);
    virtual Status Close();
    virtual Status Flush();
    virtual Status Sync();
    BOOL isEnable();
private:
    std::string _filename;
    HANDLE _hFile;
    size_t _page_size;
    size_t _map_size;       // How much extra memory to map at a time
    char* _base;            // The mapped region
    HANDLE _base_handle;	
    char* _limit;           // Limit of the mapped region
    char* _dst;             // Where to write next  (in range [base_,limit_])
    char* _last_sync;       // Where have we synced up to
    uint64_t _file_offset;  // Offset of base_ in file
    //LARGE_INTEGER file_offset_;
    // Have we done an munmap of unsynced data?
    bool _pending_sync;

    // Roundup x to a multiple of y
    static size_t _Roundup(size_t x, size_t y);
    size_t _TruncateToPageBoundary(size_t s);
    bool _UnmapCurrentRegion();
    bool _MapNewRegion();
    DISALLOW_COPY_AND_ASSIGN(Win32MapFile);
    BOOL _Init(LPCWSTR Path);
};
```




- [leveldb_handbook](https://leveldb-handbook.readthedocs.io/zh/latest/journal.html)
- [Windws内存映射文件](http://blog.tk-xiong.com/archives/933)
- [leveldb笔记之2:日志](https://izualzhy.cn/leveldb-log)

