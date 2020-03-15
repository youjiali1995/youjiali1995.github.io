---
title: Reading Notes
excerpt: 好久没读书和论文了，以后都会在这里记录，满分 10 ☆。
layout: post
categories: Reading
---

{% include toc %}

## 2020

### 03-15 | *Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases* | ?☆

HLC 算法很简单，就是在 physical lock 基础上使用了 logical clock 算法来保证 happened-before 关系。因为现在 TiDB 的悲观事务获取 timestamp 很频繁，我在想办法降低这部分的开销，所以更关心 HLC 如何用在 Snapshot Isolation 数据库上，这个论文里只简单提了一下，对我帮助不大，需要再研究下 cockroachdb 的事务。

### 03-15 | *Time, Clocks, and the Ordering of Events in a Distributed System* | ?☆

论文讨论了时间、时钟、事件顺序之间的关系。无法简单的用时间、时钟来确定分布式系统中事件的顺序，不同进程间的事件顺序要靠消息通信来确定，但也只能做到偏序或者说部分全序，因为不相交的事件不需要通信，也就无需确定顺序。若想要实现整个系统的全序对物理时钟有更严格的要求。

### 03-14 | *Database Internals* | 7☆

挺新的书，花了 2 周时间过了一遍，尴尬的是大部分我都懂，我不了解的看了也还是不了解。这本书更多的讲述的是「术」而不是「道」，介绍了很多问题的多种解决办法，相对而言还是更推荐 *DDIA*。不过这本书可以当 CMU15-445 的参考资料，第一部分几乎全覆盖了。