---
title: leveldb 源码分析(四) -- Compaction
layout: post
excerpt: 介绍 leveldb 的 compaction 实现。
categories: Storage
---

{% include toc %}

`compaction` 和读密切相关，`sstable` 文件的格式、`sstable` 的组织都会影响读的性能。`leveldb` 的 `compaction` 有两种：
*  `memtable` 大小超过阈值时，转储为 `level-0` 的 `sstable`。
* `sstable` 的 `compaction`。

`compaction` 的思路在[之前的文章](https://youjiali1995.github.io/storage/leveldb-architecture/) 有介绍，不再赘述。

## sstable
对 `sstable` 的操作只有2种：从 `sstable` 中查找 `key`和遍历 `sstable`。所以对 `sstable` 的要求如下：
* 生成时要顺序写。
* 支持高效的查找。
* 要支持遍历。

`sstable` 的格式如下：
![image](/assets/images/leveldb/sstable.png)

`sstable` 划分为 `Block` 存储，不同 `Block` 存储不同的数据：
* `Data Block`：有序存储 `key value` 数据。
* `Meta Block`：存储 `metadata`，目前 `leveldb` 中只存储 `filter` 信息。
* `Metaindex Block`：`handle` 用于索引 `Block`，包括 `Block` 的 `offset` 和 `size`。`Metaindex Block` 保存 `Meta Block` 的索引，目前只保存 `filter.name` 和 `filter Block` 索引。
* `Index Block`：保存每个 `Data Block` 最后一个 `key` 和该 `Data Block` 的索引。
* `Footer`：固定 `48 Bytes`，保存 `Metaindex Block` 和 `Index Block` 的索引，是解析的起点。

`sstable` 生成时按顺序写入。当读取某个 `key` 时，按照如下顺序：
* 解析 `Footer`，找到 `Metaindex Block` 和 `Index Block` 的位置。
* 查找 `Index Block` 找到 `key` 属于的 `Data Block` 的位置。
* 在 `Data Block` 中查找 `key`。
* 若是有 `filter`，会在查找 `Data Block` 前先查找 `filter`，只有存在时才查找 `Data Block`。

`leveldb` 在读取 `sstable` 时会把整个文件映射进来(`mmap`)，查找很高效。

### Block
`Block` 保存 `key/value` 对，并会进行压缩和校验(`checksum`)，格式如下：
![image](/assets/images/leveldb/block.png)

`Block` 中第一个 `key` 是完整的，按顺序解析时，通过拼接和前一个 `key` 重叠的加上 `Delta` 部分即可得到当前完整的 `key`。按顺序存放的 `key` 之间很可能会有重叠，
尤其是 `leveldb` 中很多 `key` 只有 `sequence` 不同，使用这种存储方式能够极大的降低数据存储量。因为 `Block` 的大小可能很大，
如 `leveldb` 默认配置 `Data Block` 大小为 `4KB`，存放的 `key/value` 可能很多，每次查找都要从头开始会很慢，而且只要开头的完整 `key` 有损坏，整个 `Block` 的数据都无法使用。
`Block` 增加了 `restart points` 来解决这些问题：
* 每隔一定数量的 `key/value` 会有一个 `restart point`，如 `Data Block` 默认间隔为 `16`。`restart point` 的 `shared bytes` 为 `0`，存放完整的 `key`。
* `Trailer` 中记录 `restart points` 的 `offset`，查找时先二分查找 `restart points` 找到 `key` 所在的起始 `restart point`，然后顺序查找即可，避免了从头开始查找。

`Block` 的查找算法如下：
* 二分查找 `restart points`：找到最后一个 `key < target` 的 `restart point`。
* 然后从 `restart point` 开始顺序遍历，直到找到第一个 `key >= target`。

`Block` 还会进行压缩和校验，目前只支持 `snappy` 压缩。`Block` 不强制存放 `key/value`，如 `filter Block` 中只存放了 `filter`。

### Data Block
`Data Block` 按照上面格式存放 `key/value`，其中 `key` 是 `InternalKey` 的格式，包括 `sequence` 和 `type`，`value` 格式不变。当 `Data Block` 的大小超过配置的 `block_size`(默认为 `4KB`)时，
会创建一个新的 `Data Block`。

需要注意根据 `Block` 的查找算法，若查找的 `key` 正好是 `restart point`，那么会从上一个 `restart point` 开始查找，不影响功能正确，但会多遍历一些 `key`。

### Meta Block
目前 `leveldb` 中只有 `filter` 一种 `metadata`。`leveldb` 中使用 `bloom filter`，用于快速判断 `key` 是否存在，它的原理如下:
* 对于一个 `key` 集合，使用固定大小的位图，假设 `key` 的个数为 `n`，位图大小为 `m bits`。
* 集合内每个 `key` 计算出一组 `hash` 值，将对应的 `bit` 置位，假设每个 `key` 使用 `k bits`。
* 查询 `key` 时使用相同的算法计算出 `k bits`，若每个 `bit` 都为 `1`，则这个 `key` 很**可能**存在；若不都为 `1`，则这个 `key` **一定**不在。

`bloom filter` 的关键就是降低 `false positive` 的概率，`false positive` 即 `bloom filter` 认为这个 `key` 存在但不存在，它的概率如下:
![images](/assets/images/leveldb/bloomfilter_false_positive.gif)

一般情况下，`key` 的数量是已知的，位图的大小是根据 `key` 个数决定的，那么 `n` 和 `m` 是固定，最优的 `k` 为： `k = ln2 * m/n`。`m/n` 即 `bits_per_key`，
所以 `leveldb` 中 `bloom filter` 会乘以 `ln2(0.69)` 得到 `hash` 的次数，不过并不是计算多次 `hash`，而是计算一次 `hash`，
之后的 `bit` 直接加上一个 `delta` 得到，避免了多次 `hash` 带来的性能消耗：
```cpp
  explicit BloomFilterPolicy(int bits_per_key)
      : bits_per_key_(bits_per_key) {
    // We intentionally round down to reduce probing cost a little bit
    k_ = static_cast<size_t>(bits_per_key * 0.69);  // 0.69 =~ ln(2)
    if (k_ < 1) k_ = 1;
    if (k_ > 30) k_ = 30;
  }
```

`Filter Block` 的格式如下:
![image](/assets/images/leveldb/filter_block.png)

`leveldb` 是为每个 `Data Block` 创建一个 `bloom filter`，要注意创建 `bloom filter` 的 `key` 要是 `UserKey` 而不是 `InternalKey`。
需要查找某个 `Data Block` 前，先查找该 `Block` 对应的 `bloom filter`，这就带来另一个需求，快速查找 `Data Block`对应的 `bloom filter`。`leveldb` 使用如下方式:
* 为每个 `Data Block` 生成一个 `Filter`。
* 按照 `Data Block` 的 `offset` 生成 `filter` 索引，目前是每 `2KB(1 << kFilterBaseLg)` 生成一个。当查找 `Data Block` 对应的 `filter` 时，只要
查找第 `Data Block Offset >> kFilterBaseLg` 个 `Filter Offset` 即可。按照 `2KB` 划分的原因是因为 `Data Block` 的大小不固定，超过 `4KB` 会切换到新的，但可能会超很多。

其实感觉为整个 `sstable` 生成一个 `bloom filter` 即可。

### Metaindex Block
目前只存放 `filter.name` 到 `Filter Block` 的索引。

### Index Block
`Index Block` 保存 `Data Block` 的索引，且每个 `key` 都是 `restart point`:
* `key`：大于等于该 `DataBlock` 最后一个 `key`，小于后一个 `DataBlock` 第一个 `key`。这么做可以降低存储的数据量，比如最后一个 `key` 很长，但比它大且比后一个小的 `key` 很短。
* `Value`：该 `Block` 的 `offset` 和 `size`。

存放每个 `DataBlock` 最后一个 `Key` 而不是第一个 `Key` 的原因是，`DataBlock` 找到的是第一个大于等于 `target` 的。

### Footer
`Footer` 是解析的起点，大小固定为 `48 Bytes`：
* `Metaindex Block Handle`：`20 bytes`，`offset` 和 `size` 都使用 `varint64` 存储。
* `Index Block Handle`：同上。
* `Magic Number`：用于判断检测文件有效性。

## memtable compaction
当 `memtable` 大小超过 `Options.write_buffer_size` 时(默认 4MB)，会在下一次写操作时将当前的 `memtable` 转为 `immutable memtable`，创建新的 `memtable`，并触发
`immutable memtable` 的 `compaction`。`compaction` 由单独的线程来执行。

## sstable compaction

## Version
1. snapshot 链表
2. 主要是为了 iter 的有效性。snapshot 是 compaction ？
