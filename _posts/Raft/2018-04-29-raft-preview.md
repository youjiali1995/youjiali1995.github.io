---
title: Raft 笔记(一) -- Preview
layout: post
categories: Raft
---

在今年1月份时花了几个周末时间把 `6.824` 做了一下，`lab 4` 还有一些没做完，因为设计的问题，导致实现数据迁移同时可以接受请求会比较麻烦，然后就搁置了(思路和 `Redis` 类似)。
做这门课程最主要的目的还是学习一下 `Raft`，在后面的文章里会记录一下。

## 资料
主要的参考资料是2篇论文:
* [raft-extended](https://raft.github.io/raft.pdf)
* [raft-thesis](https://ramcloud.stanford.edu/~ongaro/thesis.pdf)

同时会看 [etcd/raft](https://github.com/coreos/etcd/tree/master/raft) 是如何实现和优化的。
