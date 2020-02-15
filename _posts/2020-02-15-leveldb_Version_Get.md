---
layout:     post
title:      leveldb之Version
subtitle:   
date:       2020-02-15
author:     BY
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - leveldb
---



相较与OC来说更易读了：

```objc
let dispatch_time = dispatch_time(DISPATCH_TIME_NOW, Int64(60 * NSEC_PER_SEC))
```

## 业务流程
![](https://images2017.cnblogs.com/blog/1183530/201801/1183530-20180116202929646-1073704541.png)

- 参考 [GCD 在 Swift 3 中的玩儿法](https://www.swiftcafe.io/2016/10/16/swift-gcd/)
