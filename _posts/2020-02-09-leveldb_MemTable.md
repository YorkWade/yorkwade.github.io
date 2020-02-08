---
layout:     post
title:      leveldb之MemTable
subtitle:   所有的应用都是内存数据库加上业务逻辑
date:       2002-02-09
author:     BY
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - leveldb
    
---

>在Leveldb中，所有内存中的KV数据都存储在Memtable中,内部使用[SkipList](https://yorkwade.github.io/2020/02/05/leveldb_SkipList/)实现。当Memtable写入的数据占用内存到达指定数量，则自动转换为Immutable Memtable，等待Dump到磁盘中，系统会自动生成新的Memtable供写操作写入新数据。

## 为终端添加一个快捷键打开方式

levedb三个存储区域：Memtable，Immutable Memtable和SSTable中的。Memtable，Immutable Memtable结构一样，差别在于：<br>
    memtable 允许写入跟读取。<br>
    immutable memtable只读。<br>



![](https://ww2.sinaimg.cn/large/006tKfTcgy1fckb184f74j319v0q01kx.jpg)

新建文稿

![](https://ww1.sinaimg.cn/large/006tKfTcgy1fckb6zzo28j30mo0fvgn7.jpg)

创建一个服务

![](https://ww1.sinaimg.cn/large/006tKfTcgy1fckb93qmy5j30g00fh0vq.jpg)

![](https://ww2.sinaimg.cn/large/006tKfTcgy1fckbfe8o0zj30t10lb0wv.jpg)

![](https://ww1.sinaimg.cn/large/006tKfTcgy1fckbff4e7pj30t10lbwis.jpg)

修改框内的脚本

```
on run {input, parameters}
	tell application "Terminal"
		reopen
		activate
	end tell
end run

```

运行：`command + R`，如果没有问题，则会打开终端

![](https://ww2.sinaimg.cn/large/006tKfTcgy1fckaqdd2m1j30t10lb42a.jpg)

![](https://ww3.sinaimg.cn/large/006tKfTcgy1fckaq4nn9hj30iy0daaan.jpg)

保存：`Command + S`，将其命名为`打开终端`或你想要的名字

设置快捷键

在 **系统偏好设置** -> **键盘设置** -> **快捷键** -> **服务**

选择我们创建好的 '**打开终端**'，设置你想要的快捷键，比我我设置了`⌘+空格`

![](https://ww4.sinaimg.cn/large/006tKfTcgy1fckbvaixhnj30kw0ihq67.jpg)

到此，设置完成。

聪明的你也许会发现，这个技巧能为所有的程序设置快捷启动。

将脚本中的 `Terminal` 替换成 其他程序就可以

```
on run {input, parameters}
    tell application "Terminal"
        reopen
        activate
    end tell
end run

```

## 黑技能

既然学了 `Automator` ，那就在附上一个黑技能吧。为你的代码排序。在 **Xcode8**以前，有个插件能为代码快速排序，不过时过境迁~ 对于没用的插件而且又有患有强迫症的的小伙伴，只能手动排序了（😂）.

首先还是创建一个服务

创建一个`Shell`脚本，

勾选:`用输出内容替换所选文本`

输入：`sort|uniq` 

保存： 存为`Sort & Uniq`

![](https://ww4.sinaimg.cn/large/006tKfTcgy1fckd40rgwmj30rt0ildiy.jpg)

**选中你的代代码** -> **鼠标右键** -> **Servies** -> **Sort&Uniq**

![](https://ww2.sinaimg.cn/large/006tKfTcgy1fckd6tx1dzj30h90b7mzm.jpg)

排序后的代码：

![](https://ww3.sinaimg.cn/large/006tKfTcgy1fckd6lak55j309j05y3yo.jpg)

