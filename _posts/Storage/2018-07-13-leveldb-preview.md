---
title: leveldb 源码分析(一) -- Preview
layout: post
categories: Storage
---

久仰 `leveldb` 大名，但是因为不会 `C++` 就一直没看。花了将近一个月的空闲时间学习了下 `C++` 和 `leveldb`，`C++` 的确比较复杂，好在因为时代的限制，`leveldb` 没有用到更新
的标准，使用的语法都很简单，看起来不吃力。

## 版本
目前的最新版本 [v1.20](https://github.com/google/leveldb/tree/v1.20)。

## 参考资料
* [doc](https://github.com/google/leveldb/tree/v1.20/doc) 配合源码里的注释已经足够了。
* 网上的博客看到的最好的是 [CatKang](https://github.com/CatKang) 的 [庖丁解Leveldb](http://catkang.github.io/2017/01/07/leveldb-summary.html) 系列，写的很好，既详细又不拖泥带水。
