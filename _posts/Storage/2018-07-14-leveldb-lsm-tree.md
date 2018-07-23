---
title: leveldb 源码分析(二) -- LSM-Tree
layout: post
excerpt: 介绍 leveldb 的设计思路和各模块。
categories: Storage
---

{% include toc %}

## LSM-Tree
`leveldb` 是以 `LSM-tree` 为模型实现的。`LSM-tree` 将随机写转换为顺序写从而获得更高的写性能，大致思路如下：
* 数据分为2部分存储，共同维护一个有序的空间：
    * 内存中维护最新的写数据 `memtable`，所有的写操作都在内存中进行，同时顺序写 `WAL`。
    * 磁盘上维护较老的数据 `sstable`。
    * 读操作先读内存中的数据，然后读磁盘上的数据。
* 在合适的时机进行 `compaction`:
    * 当内存中的数据大小超过阈值时，刷新到磁盘上。
    * 在合适的时机清理、合并磁盘上的数据。

`LSM-tree` 是一种设计思想，没有固定的实现方式，其中最重要的是 `compaction` 的实现，涉及到 `compaction` 的时机、 `sstable` 的组织方式、
`memtable` 和 `sstable` 间的合并方式。`compaction` 对读写性能起着至关重要的影响，它会清理无用数据，能够降低磁盘空间使用并降低读的延迟，但是也会带来极大的 `I/O` 压力。
`compaction` 主要考虑如下几个因素:
* 写放大: 要减少 `compaction` 时涉及到的无关数据量。
* 读放大: 减少读数据时，读取的无关数据量。
* 随机 `I/O`: 尽量减少随机 `I/O`。

考虑最基本的情况，磁盘上维护一个大的 `sstable`，当 `memtable` 大小超过阈值时，与磁盘上的 `sstable` 合并，每次合并可能要将整个 `sstable` 进行重写，会带来
极大的写放大，读取时也会有大量随机 `I/O`。现在一步步进行优化:
1. 既然不能每次 `memtable` 刷新为 `sstable` 就进行合并，可以维护多个独立的 `sstable`，但是读的话就会一个一个读 `sstable`，无用数据很多，读放大也很大，所以要限制 `sstable` 的数量。
2. 当 `sstable` 数量到达阈值时 `compaction` 为一个大的 `sstable`，但这又回到了最初的那种情况，独立的 `sstable` 会和大的 `sstable` 进行合并。
3. 为了降低写放大，将大的 `sstable` 划分为多个小的、不重叠的 `sstable`，这样 `sstable` 就分为2层：`level-0` 是 `memtable` 直接转换而来，文件之间有重叠；`level-1` 由
`level-0` 合并而来，文件之间无重叠。当 `level-0` 的文件个数达到上限时，只要挑选 `level-1` 中和 `level-0` 文件有重叠的合并即可，读 `level-1` 也只要读一个文件。
但仍有可能 `level-1` 中所有的 `sstable` 都需要合并，所以又要限制 `level-1` 的大小。
4. 当 `level-1` 的 `sstable` 达到上限时，使用同样的方法 `compaction` 为 `level-2` 的 `sstable`。`level-2` 达到上限，再 `compaction` 为 `level-3`……

`leveldb` 就是类似上面的思路分层管理 `sstable`，所以叫 `leveldb`。现在看一下它的实现：
* `memtable` 由 `skiplist` 实现，只有追加操作，每个操作先顺序写到 `log` 中。
* 当 `memtable` 大小超过阈值时(默认为 `4MB`)，变为 `immutable memtable`，在后台线程 `compaction` 为 `level-0` 的 `sstable`。
* `sstable` 的大小固定，默认为 `2MB`。
* 共分为 `7` 层:
    * `level-0` 由 `memtable` 直接转化而来，文件之间有重叠。当 `level-0` 的 `sstable` 数量到达阈值时，会将 `level-0` 的所有 `sstable` 和 `level-1` 中有重叠的 `sstable` 合并为新的 `level-1` 的 `sstable`。
    * 其余 `level` 由低层 `compaction` 而来，每层有大小限制，`level-(N+1)` 的大小限制是 `level-N` 的 `10` 倍，其中`level-1` 为 `10MB`。相同 `level` 的文件之间无重叠。当 `level-N` 的
 `sstable` 大小到达阈值时，会挑选一个文件(其实不止一个)和 `level-(N+1)` 中有重叠的 `sstable` 合并为新的 `level-(N+1)` 的 `sstable`。

## 并发控制
`leveldb` 使用 `MVCC` 实现并发控制，支持单个写操作和多个读操作同时执行:
* 每个成功的写操作会更新 `db` 内部维护的顺序递增的 `sequence`。
* 读操作会获取当前最新的 `sequence`(或者使用传入的 `snapshot`)，只会读到小于等于自己的最大的 `sequence` 数据。

因为 `compaction` 会改变磁盘上文件的组织，为了不影响正在进行的操作，`leveldb` 使用 `VersionSet` 维护不同版本的 `sstable` 组织，每当有文件删除或增加时，就会创建新的
`Version` 插入到 `VersionSet`。只有当一个 `sstable` 不再被任意 `Version` 使用时才会进行删除。

## memtable
任意有序的结构都可以实现 `memtable`，`leveldb` 使用 `skiplist` 实现，因为结构简单，更容易实现 `lock-free` 的支持一写多读的。

### atomic pointer
`leveldb` 自己实现了 `atomic pointer`，具体原因可以查看 `port/atomic_pointer.h` 上面的注释，实现如下:
```cpp
#if defined(ARCH_CPU_X86_FAMILY) && defined(__GNUC__)
inline void MemoryBarrier() {
  __asm__ __volatile__("" : : : "memory");
}

class AtomicPointer {
 private:
  void* rep_;
 public:
  AtomicPointer() { }
  explicit AtomicPointer(void* p) : rep_(p) {}
  inline void* NoBarrier_Load() const { return rep_; }
  inline void NoBarrier_Store(void* v) { rep_ = v; }
  inline void* Acquire_Load() const {
    void* result = rep_; // read-acquire
    MemoryBarrier();
    return result;
  }
  inline void Release_Store(void* v) {
    MemoryBarrier();
    rep_ = v; // write-release
  }
};
```

`AtomicPointer` 提供了以下语义：
* 原子性: `Acquire_Load/Release_Store` 是原子的，不会被其他线程看到 `half-write`。
* 可见性: `Release_Store` 立即对接下来的 `Acquire_Load` 可见；`Release_Store` 之前的写命令对 `Acquire_Load` 之后的读命令可见。

我主要关注 `linux on x86/64` 下的实现，为了搞懂有一些细节需要了解：
* 多核体系下，每个 `CPU` 有独占的 `cache`，使用 `MESI` 协议来保证 `cache coherence`。为了降低同步对性能的影响，每个 `CPU` 有 `store buffer` 和 `invalidate queue` 来缓冲相应的同步消息，对同步消息处理
的时机和顺序是不确定的。
* `CPU` 在保证单线程程序的正确运行的前提下，为了提高性能会对命令做乱序处理。多线程情况下，因为 `store buffer` 和 `invalidate queue` 的存在，其他线程的修改不会立即对另外的线程可见；受到 `CPU` 乱序和
对同步消息处理的顺序影响，可见的顺序也不能保证。
* `memory barrier` 用于保证多核之间操作的执行顺序，包含4类:
    * `LoadLoad-barrier`: `memory barrier` 前(后)的 `load` 不会乱序到 `memory barrier` 后(前)。
    * `LoadStore-barrier`: `memory barrier` 前的 `load`(后的 `store`)不会乱序到 `memory barrier` 后(前)。
    * `StoreStore-barrier`: `memory barrier` 前(后) 的 `store` 不会乱序到 `memory barrier` 后(前)。
    * `StoreLoad-barrier`: `memory barrier` 前的 `store`(后的 `load`)不会乱序到 `memory barrier` 后(前)。
* `memory barrier` 只能保证 `memory oder`，也即可见性顺序，如 `x` 操作完成可见，则 `y` 操作也一定完成可见，但并不能保证可见性，`CPU` 会尽最大努力保证可见性。`memory barrier` 一般成对使用，否则一个线程有序另一个乱序，最终的结果还是乱序。
* 比 `memory barrier` 更高一层的语义是 `acquire/release`，同样也要成对使用:
    * `acquire`: 保证了在 `acquire` 之后的读写操作不会乱序到 `acquire` 前。
    * `release`: 保证了在 `release` 之前的读写操作不会乱序到 `release` 后。
* `acquire/release` 同样只保证可见性顺序，对可见的时机不能保证，需要有方法判断 `acquire` 或者 `release` 完成。一般将 `load/store` 操作和 `memory barrier` 搭配使用构成 `acquire/release` 语义，
对 `load/store` 操作和 `memory barrier` 的顺序有要求：
    * `acquire`: `load` 在前，`memory barrier` 在后，`load` 操作构成 `load-acquire`，防止了 `LoadLoad/LoadStore` 乱序。
    * `release`: `store` 在后，`memory barrier` 在前，`store` 操作构成 `write-release`，防止了 `LoadStore/StoreStore` 乱序。
    * 这种顺序很符合逻辑，只有这样才能够确保 `store` 操作完成，之前的 `release` 操作一定完成；`load` 操作不会在 `memory barrier` 之后执行。
* 构成 `acquire/release` 语义的读写操作要操作相同的变量，同时需要是原子操作。
* 除了 `CPU` 乱序，编译器也会在保证单线程程序正确性的前提下，对命令乱序处理，同样有 `compiler barrier` 保证编译的命令顺序。

`x86/64` 是 `strong memory model`，`load/store` 自带 `acquire/release` 语义，只会出现操作不同地址的 `StoreLoad` 乱序。对于 `naturally aligned native types` 且大小不超过 `memory bus` 的变量读写操作是原子的
，所以只要防止编译器乱序即可。所以 `leveldb` 的 `atomic pointer` 在 `gcc on x86` 的实现中只使用了 `compiler barrier`。

比较奇怪的是，按照我的理解 `acquire/release` 语义只能保证可见性顺序，所以不能保证对 `AtomicPointer` 的修改立即对其他线程可见，难道是可见的时间很短可以忽略不计，可能是我什么地方疏漏了或者理解有误。

参考资料：
* [An Introduction to Lock-Free Programming](http://preshing.com/20120612/an-introduction-to-lock-free-programming/)
* [Memory Ordering at Compile Time](http://preshing.com/20120625/memory-ordering-at-compile-time/)
* [Memory Barriers Are Like Source Control Operations](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations/)
* [Acquire and Release Semantics](http://preshing.com/20120913/acquire-and-release-semantics/)
* [The Synchronizes-With Relation](http://preshing.com/20130823/the-synchronizes-with-relation/)
* [Weak vs. Strong Memory Models](http://preshing.com/20120930/weak-vs-strong-memory-models/)
* [Atomic vs. Non-Atomic Operations](http://preshing.com/20130618/atomic-vs-non-atomic-operations/)
* [The JSR-133 Cookbook for Compiler Writers](http://g.oswego.edu/dl/jmm/cookbook.html)
* [Lockless Programming Considerations for Xbox 360 and Microsoft Windows](https://docs.microsoft.com/zh-cn/windows/desktop/DxTechArts/lockless-programming)
* [LINUX KERNEL MEMORY BARRIERS](https://www.kernel.org/doc/Documentation/memory-barriers.txt)

### skiplist
`leveldb` 实现的是支持一写多读的、`lock-free` 的 `skiplist`，`skiplist` 的原理不再赘述，主要看一下是如何支持一写多读的。


### arena

### sstable
