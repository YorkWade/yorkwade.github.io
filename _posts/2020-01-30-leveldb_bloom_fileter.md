---
layout:     post
title:      leveldb之布隆过滤器
subtitle:   以小博大
date:       2020-01-30
author:     BY
header-img: img/post-bg-BJJ.jpg
catalog: true
tags:
    - BJJ
---


![Marcus'Buchecha'Almeida - 现任IBJJF绝对冠军。这家伙很坚强，相信我！图片由BJJ Pix的William Burkhardt提供  。](http://mjrnxewya3t1in23ybpwjw59.wpengine.netdna-cdn.com/wp-content/uploads/buchecha-marcus-almeida-roger-gracie.jpg)

### 理论简述

布隆过滤器（bloom filter）是通过多个hash算法来共同判断某个元素是否在某个集合内，利用多个随机数的小概率，来实现的一种高效的数据结构。当我们在 bloom filter 查找 key 时，有返回两种情况：
   1、 key 不存在，那么 key 一定不存在。
   2、 key 存在，那么 key 可能存在。
也就是说 bloom filter 具有一定的误判率，但是空间利用率更高，牺牲一点小概率也是可以接受的。

### 数学结论
bloom filter 使用多个hash函数映射到“位数组”中，因此空间利用率很高。
如何选择位数组长度？
选择多少个hash函数？
多个hash函数，可咋整？
过滤器中存储多少个key？


数学结论有：
当k(hash 函数个数)，m(bit数组大小)，n(插入元素个数)满足下式的时候，可以保证最低的误差率
1、为了获得最优的准确率，当k = ln2 * (m/n)时，布隆过滤器获得最优的准确性；
2、在哈希函数的个数取到最优时，要让错误率不超过є，m至少需要取到最小值的1.44倍

具体理论证明，一堆公式，不想多说，可参考
https://en.wikipedia.org/wiki/Bloom_filter
https://blog.csdn.net/jiaomeng/article/details/1495500
https://en.wikipedia.org/wiki/Double_hashing

![](https://en.wikipedia.org/wiki/File:Bloom_filter.svg)



### leveldb中的实现

```objc
class BloomFilterPolicy : public FilterPolicy {
private:
    size_t bits_per_key_; // 位数组大小m/插入的元素数量n
    size_t k_; // 哈希函数的个数

public:
    explicit BloomFilterPolicy(int bits_per_key)
        : bits_per_key_(bits_per_key) {
            // 当k=ln(2)*(m/n)时出错的概率最小
            k_ = static_cast<size_t>(bits_per_key * 0.69);  // 0.69 =~ ln(2)
            // 将哈希函数个数k_控制在1~30之间
            if (k_ < 1) k_ = 1;
            if (k_ > 30) k_ = 30;
    }
    
    // keys: 插入的元素;  n: 插入元素的数量; dst: 输出的位数组 
    virtual void CreateFilter(const Slice* keys, int n, std::string* dst) const {
        size_t bits = n * bits_per_key_; // 位数组的大小
        if (bits < 64) bits = 64; // 通过限制最小的位数组大小，降低错误率

        size_t bytes = (bits + 7) / 8; // 计算需要分配的空间
        bits = bytes * 8;

        const size_t init_size = dst->size();
        dst->resize(init_size + bytes, 0);
        dst->push_back(static_cast<char>(k_)); // 将哈希函数个数存放到数组开头
        char* array = &(*dst)[init_size];
        for (int i = 0; i < n; i++) {
            // 使用double-hashing模拟多个哈希函数
            // https://en.wikipedia.org/wiki/Double_hashing
            // h(i,k) = (h1(k) + i*h2(k)) % T.size
            // h1(k) = h, h2(k) = delta, h(i,k) = bitpos
            uint32_t h = BloomHash(keys[i]);
            const uint32_t delta = (h >> 17) | (h << 15);
            for (size_t j = 0; j < k_; j++) {
                const uint32_t bitpos = h % bits;
                // array数组上的每个char有8个位
                // 将array上相应的位设置为1
                array[bitpos/8] |= (1 << (bitpos % 8));
                h += delta;
            }
        }
    }

```

### 参考：

- [leveldb源码bloom fileter](https://www.dazhuanlan.com/2019/10/15/5da5258ad13fb/)

