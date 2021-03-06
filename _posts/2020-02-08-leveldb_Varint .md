---
layout:     post
title:      leveldb之Varint
subtitle:   编码
date:       2020-02-08
author:     BY
header-img: img/post-bg-debug.png
catalog: true
tags:
    - leveldb
---

> leveldb中编码如何省空间

levelDB设计时希望尽量能够节省时间，实现了Varint这种编码方式，其实这种方式在[Protocol Buffer](https://www.wandouip.com/t5i125413/)等地方也都存在类似技巧的影子。

#### 理论背景
Varint是一种比较特殊的整数类型，它包含有Varint32和Varint64两种，它相比于int32和int64最大的特点是长度可变。
-    varint是一种紧凑的表示数字的方法，它用一个或多个字节来表示一个数字，值越小的数字使用越少的字节数。比如对于int32类型的数字，一般需要4个byte来表示。采用Varint，对于很小的int32类型的数字，则可以用1个字节来表示。大的数字则可能需要5个字节来表示。leveldb用Varint来表示长度（Key的长度，value的长度），大部分不会超过4字节，可以用更少的字节数来表示数字信息。
-    varint中的每个字节的最高位（bit）有特殊含义，如果该位为1，表示后续的字节也是这个数字的一部分，如果该位为0，则结束。其他的7位（bit）都表示数字。

![](https://lrita.github.io/images/posts/leveldb/number_300_varint.png)

#### 实现要点
- int32,可能需要5个字节才能存放{5 * 8 - 5(标识位) > 32}
- int64,最多需要需要(10 * 8 - 10 > 64)10位来保存
	
#### 源码分析  

```objc
inline char* Varint::Encode32(char* sptr, uint32 v) {
  // Operate on characters as unsigneds
  unsigned char* ptr = reinterpret_cast<unsigned char*>(sptr);
  static const int B = 128;//(二进制为 1000 0000，16进制为0x80)
  if (v < (1<<7)) {//判断是需要用1个字节编码
    *(ptr++) = v;
  } else if (v < (1<<14)) {//判断是需要用2个字节编码
    *(ptr++) = v | B;//置最高位为1，以下均是。
    *(ptr++) = v>>7;
  } else if (v < (1<<21)) {//判断是需要用3个字节编码
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = v>>14;
  } else if (v < (1<<28)) {//判断是需要用4个字节编码
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = (v>>14) | B;
    *(ptr++) = v>>21;
  } else {//判断是需要用5个字节编码
    *(ptr++) = v | B;
    *(ptr++) = (v>>7) | B;
    *(ptr++) = (v>>14) | B;
    *(ptr++) = (v>>21) | B;
    *(ptr++) = v>>28;
  }
  return reinterpret_cast<char*>(ptr);
}
```
大牛能写这样的烂代码？再看EncodeVarint64，逻辑一样，实现就比较高级了。
```objc
char* EncodeVarint64(char* dst, uint64_t v) {
  static const int B = 128;//(二进制为 1000 0000，16进制为0x80)
  unsigned char* ptr = reinterpret_cast<unsigned char*>(dst);
  while (v >= B) {//是否还需要2个或2个以上字节编码
    *(ptr++) = (v & (B-1)) | B;//（B-1）二进制为(0111 1111)
    v >>= 7;//编码从低位到高位
  }
  *(ptr++) = static_cast<unsigned char>(v);//编码的最高字节
  return reinterpret_cast<char*>(ptr);
}
```
两种实现方式，是为了让读者更容易看懂编码方式吧

Varint 解码

理解了编码的原理，再来看解码就很轻松了，直接调用GetVarint32Ptr 函数，该函数处理value < 128的情况，即varint只占一个字节的情况，对于varint 大于一个字节的情况，GetVarint32Ptr调用GetVarint32PtrFallback来处理。
```objc
inline const char* GetVarint32Ptr(const char* p,
                                  const char* limit,
                                  uint32_t* value) {
  if (p < limit) {
    uint32_t result = *(reinterpret_cast<const unsigned char*>(p));
    if ((result & 128) == 0) {
      *value = result;
      return p + 1;
    }
  }
  return GetVarint32PtrFallback(p, limit, value);
}
```
在GetVarint32Ptr和GetVarint32PtrFallback函数中，参数p 是指向一个包含varint的字符串，limit在调用的时候都是赋值为limit= p + 5, 这是因为varint最多占用5个字节。value用于存储返回的int值。
```objc
const char* GetVarint32PtrFallback(const char* p,
                                   const char* limit,
                                   uint32_t* value) {
  uint32_t result = 0;
  for (uint32_t shift = 0; shift <= 28 && p < limit; shift += 7) {//shift <= 28最多移动4个字节，p < limit不超过5字节
    uint32_t byte = *(reinterpret_cast<const unsigned char*>(p));
    p++;
    if (byte & 128) {
      // More bytes are present
      result |= ((byte & 127) << shift);//127二进制(0111 1111)
    } else {
      result |= (byte << shift);
      *value = result;
      return reinterpret_cast<const char*>(p);
    }
  }
  return NULL;
}
```
```objc
const char* GetVarint64Ptr(const char* p, const char* limit, uint64_t* value) {
  uint64_t result = 0;
  for (uint32_t shift = 0; shift <= 63 && p < limit; shift += 7) {
    uint64_t byte = *(reinterpret_cast<const unsigned char*>(p));
    p++;
    if (byte & 128) {
      // More bytes are present
      result |= ((byte & 127) << shift);
    } else {
      result |= (byte << shift);
      *value = result;
      return reinterpret_cast<const char*>(p);
    }
  }
  return NULL;
}
```





	
	
- [LevelDB源码剖析之Varint](http://mingxinglai.com/cn/2013/01/leveldb-varint32/)
- [Leveldb varint 解析](https://ce39906.github.io/2018/04/17/Leveldb-varint-%E8%A7%A3%E6%9E%90/)
- [Protocol Buffer 序列化原理大揭秘](https://www.wandouip.com/t5i125413/)
