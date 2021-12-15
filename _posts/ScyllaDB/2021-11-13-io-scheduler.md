---
title: ScyllaDB 学习(六) -- disk I/O scheduler
layout: post
excerpt: 介绍 seastar 的 disk I/O scheduler 实现。
categories: ScyllaDB

---

{% include toc %}

最近 disk I/O scheduler 又有了[变化](https://github.com/scylladb/seastar/commit/b0d13a444e90ffb945c8a1ce733a8308edba5315)，所以分析的版本变为了：[98d99b9f3dfeabd9148df0284f557df14ef0893d](https://github.com/scylladb/seastar/tree/98d99b9f3dfeabd9148df0284f557df14ef0893d)。要对 disk I/O 进行调度同样需要先建模、再标记请求、最后实现优先级控制。在介绍这些之前先来看一下 I/O 方式的选择。

## Why asynchronous direct I/O?

三年前写过一些 I/O 相关的[东西](/rocksdb/io)，当时只介绍了几种 sync I/O 方式，没涉及到 async I/O，也没有对比几种方式的优劣，而且有些信息在现在看来不太对，现在对 I/O 有了些新的认识，所以根据现在的认知再来写一些，主要是对比不同的 I/O 方式并从中选择最适合存储系统的。

Linux 下 disk I/O 按特性分有这两类：

- buffered vs direct
- sync vs async

先来选是 buffered 还是 direct，前者是最常见的 I/O 方式，没有对齐要求，读写都会经过 page cache；后者有对齐要求且不经过 page cache。

- page cache：夸张点说，这东西对存储系统毫无作用。
  - 写是先拷贝到 page cache，如果用 `fsync()` 之类的立刻就会 flush 到 disk，那写到 page cache 就纯属浪费了。如果不立刻调用 `fsync()`，kernel 在 dirty page 过多或过老时会帮你 flush dirty，而且 flush 的时机不是由存储系统决定的，说不定在压力大时触发而严重影响到前端流量，很像 LSM-tree，一开始写的很开心，时间长了后台工作的影响不能忽视。
  - 读也是先检查 page cache，没有命中 cache 再从 disk 读到 page cache 再拷贝到用户程序。一般存储系统都会自己做 cache，比如 rocksdb 的 block cache，page cache 内的数据毫无作用还浪费了内存，不如多给 block cache 点内存。如果没有自己做 cache，page cache 也不是好的 cache 选择，毕竟 kernel 是通用的，肯定没有系统针对自己特点做的 cache 更合适。
- amplication：其实也是和 page cache 相关的，buffered I/O 以 page 为单位，通常最小也是 4KiB，读写不在 page boundary 的都会放大到以 page 为单位。direct I/O 对 block size、对齐有要求，所以也会有放大，但最小单位可以到 512B，相比 buffered I/O 小一些。buffered read 还有 readahead 机制，利用不当的话，比如在完全随机读且未关闭它的场景下，读放大会非常大。readahead 可能给一些人造成了 direct I/O 性能不如 buffered I/O 的假象，但 direct I/O 用更大的 I/O size 或者自己做 readahead 性能不会有差距，而且 readahead 对整个文件生效，不适用于访问模式不固定的场景，用 direct I/O 自己做 readahead 更灵活。
- 不确定性：buffered I/O 不能确定读写是否会阻塞，而 direct I/O 一定会阻塞，行为更可预期。系统见的多了，更喜欢稳定、确定的系统，即使是稳定的慢，不要那种时快时慢或者逐渐衰减的，因为把稳定的系统做快（应该）比把快的系统做稳定容易。

kernel 为 buffered I/O 做的所有工作，在用户态用 direct I/O 都可以做到，而且能做的更适合自己，还能消除掉 buffered I/O 带来无用消耗。direct I/O 相比 buffered I/O 唯一的缺点是不够易用，对 I/O size、buffer address、file offset 有对齐要求，但毕竟都做存储了，这点困难还是可以克服的，所以没理由不选 direct I/O。

现在来选是 sync 还是 async。要利用好现代的高速硬盘要么有大的 I/O size 来打满带宽，要么有高的 I/O depth 来打满 IOPS，但对于 OLTP 等对低延迟有要求的系统而言，攒大的 I/O size 通常意味着更高的延迟，所以 OLTP 系统更倾向于 IOPS 密集型（不考虑后台的整理工作，如 compaction、GC，这类工作通常有很大的 I/O size，只需要很小的 I/O depth 就能打满带宽，但这些工作毕竟对前端业务贡献有限，而且通常需要限制它们来减少对前端业务的影响）：

- 如果用 sync I/O，每个线程只能贡献一个 I/O depth，高的 I/O depth 就意味着多的线程和多的 context switch，而且每次 I/O 都有 syscall 和 context switch 的开销。context switch 其实非常影响 I/O 系统性能，不仅仅因为它本身的开销，还因为 I/O 线程执行的延迟，毕竟即使 I/O 完成了，I/O 线程也要等轮到它执行才能继续工作。

- 如果用 async I/O，单个线程就能打满 disk，也有能力消除掉 context switch，而且 linux AIO 支持 batch 提交 I/O 请求，syscall 的开销也就可以忽略不计了。如果需要对 disk I/O 进行控制，async I/O 也是最好的选择，因为要控制也只是控制 inflight 请求数量也就是 I/O depth，用 sync I/O 的话就意味着线程数量可能要跟着变化，代价高且不灵活，而 async I/O 只需要记录 inflight 数量即可。

从我个人经验看，如果 sync I/O 是在业务路径而不是只做 I/O 路径上的系统，对 disk 的使用都不会好，因为 disk I/O 会阻塞后续 I/O 请求的创建，而用了 async I/O 自然就会想要用纯异步模型，disk 就会有充足的压力，这时需要考虑的更多是如何做 batch 和流控。很多系统用类似 posix AIO 的方式也就是单独的线程池搭配上 sync I/O 来模拟 async I/O，那不如直接用真正的 async I/O。

linux I/O 还有个异类 `mmap`，是 sync buffered I/O，相比普通的 sync buffered I/O 而言，唯一的好处是没有内存拷贝，但其他的缺点样样不落，而且因为能像内存一样直接访问，用了 `mmap` 的程序一般也不会自己做 cache 之类的，全交给 kernel 来做，连缓解 buffered I/O 缺点的机会都没有。`mmap` 是靠内存映射实现的，文件大时 page table 消耗的内存无法忽视，尤其在 disk : memory 比重高的系统。总结一下就是 [Andy Pavlo](https://www.cs.cmu.edu/~pavlo/) 说过的：要把永远不要在数据库中用 `mmap` 刻到他的墓碑上。

`seastar` 因为是 thread-per-core 架构自然就只能用 async I/O 了，在 io-uring 还没出现之前，linux AIO 只支持 direct I/O，所以是 async direct I/O，但也不仅仅只因为 async 就选择了这种方式，async direct I/O 也是最适合存储系统的 I/O 方式。

----

Reference:

- [Different I/O Access Methods for Linux, What We Chose for Scylla, and Why](https://www.scylladb.com/2017/10/05/io-access-methods-scylla/)
- [Modern storage is plenty fast. It is the APIs that are bad.](https://itnext.io/modern-storage-is-plenty-fast-it-is-the-apis-that-are-bad-6a68319fbc1a)
- [Direct I/O writes: the best way to improve your credit score.](https://itnext.io/direct-i-o-writes-the-best-way-to-improve-your-credit-score-bd6c19cdfe46)

## Caveats of DIO/AIO

async direct I/O 用得少的原因除了异步不容易使用外，它本身也不够易用，有很多坑。

### Direct I/O

从 [open(2)](http://man7.org/linux/man-pages/man2/open.2.html) 看到，自 linux 2.6.0 以后，DIO 对 I/O size、buffer address、file offset 的对齐要求由按 filesystem 的要求变为了按 block device 的要求对齐即可，后者可通过 `ioctl(2) BLKSSZGET` 或者 `blockdev --getss` 获取（内部使用了前者）。

```
   O_DIRECT
       The O_DIRECT flag may impose alignment restrictions on the length
       and address of user-space buffers and the file offset of I/Os.
       In Linux alignment restrictions vary by filesystem and kernel
       version and might be absent entirely.  However there is currently
       no filesystem-independent interface for an application to
       discover these restrictions for a given file or filesystem.  Some
       filesystems provide their own interfaces for doing so, for
       example the XFS_IOC_DIOINFO operation in xfsctl(3).

       Under Linux 2.4, transfer sizes, the alignment of the user
       buffer, and the file offset must all be multiples of the logical
       block size of the filesystem.  Since Linux 2.6.0, alignment to
       the logical block size of the underlying storage (typically 512
       bytes) suffices.  The logical block size can be determined using
       the ioctl(2) BLKSSZGET operation or from the shell using the
       command:

           blockdev --getss
```

DIO 按照 4KiB 对齐一定可行，但是在程序里自动获取 block device 的信息比较麻烦，想要降低对齐要求不是那么容易。`ioctl(2) BLKSSZGET` 只适用于 block device，需要通过 file path 找到 block device path，而且还要 root 权限，所以这个方式基本不可用：

```c
fd = open("/dev/xxx", O_RDONLY);
fstat(fd, &st);
if (S_ISBLK(st.st_mode)) {
    ioctl(fd, BLKSSZGET, &block_size);
}
```

`lsblk -t` 不需要 root 权限就可以在命令行获取 block device 的信息，所以可以参考它的[实现](https://github.com/util-linux/util-linux/blob/master/misc-utils/lsblk.c)：通过读取 `/sys/block` 和 `/sys/dev/block` 获取 block device 的信息。这里涉及到几个 [`sysfs(5)`](https://man7.org/linux/man-pages/man5/sysfs.5.html)，简单介绍下：

- `/sys/devices`：包含系统内所有 device 的信息，I/O 相关的信息都在 block device 的 [`queue`](https://www.kernel.org/doc/Documentation/block/queue-sysfs.rst) 目录下，比如 I/O scheduler 配置、logical block size 等。
- `/sys/block`：包含 `/sys/devices` 里所有 block device 的符号链接，但只包含 root device，不包含 partition。
- `/sys/dev/block`：类似 `/sys/block`，但也包含了 partition。

用 [`stat(2)`](https://man7.org/linux/man-pages/man2/lstat.2.html) 获取文件的 `st_dev` 后就能找到对应的 block device 的符号链接路径：`/sys/dev/block/major:minor/`，但有些情况下该路径下会没有 `queue` 目录，比如是 partition 的话需要找到对应的 root 才行。看了下 `seastar` 和 `rocksdb` 对这部分的处理：

- [`seastar`](https://github.com/scylladb/seastar/blob/8d15e8e67a414b2dc5deec7097648cad6ed4c749/apps/iotune/iotune.cc#L115)：它先通过 block device 路径的 `slaves` 目录逐层向上找到没有 `slaves` 目录的一层（`slaves` 目录没找到文档介绍，只找到了 [sysfs representation of stacked devices (dm/md)](https://lwn.net/Articles/172689/)），如果该层有 `partition` 文件的话，说明这是个 partition，就跳到父目录下找 `queue` 目录。
- [`rocksdb`](https://github.com/facebook/rocksdb/blob/66b31c509832a1e3daa6880bca097c9a27a9b6b5/env/io_posix.cc#L468)：它通过比较 block device 路径的父目录是否是 `block` 或者当前目录是 `nvme0n1p1` 之类的（包含 p）来决定是否要在父目录下找 `queue` 目录。相比起来，应该 `seastar` 的方式更科学些。

即使用了 DIO，仍需要用 `fsync()`/`fdatasync()` 或者 `O_SYNC`/`O_DSYNC` 来保证数据的持久性（后者效果应该等价于前者，且能减少 syscall 调用和文件系统的影响，但也提升了 batch 难度，以后会单独写一篇 disk I/O 的文章，就不在这里展开了。）

```
The O_DIRECT flag on its own makes an effort to transfer data synchronously,
but does not give the guarantees of the O_SYNC flag that data and 
necessary metadata are transferred. To guarantee synchronous I/O,
O_SYNC must be used in addition to O_DIRECT.

O_SYNC provides synchronized I/O file integrity completion, meaning 
write operations will flush data and all associated metadata to the 
underlying hardware. O_DSYNC provides synchronized I/O data integrity 
completion, meaning write operations will flush data to the underlying 
hardware, but will only flush metadata updates that are required to 
allow a subsequent read operation to complete successfully. Data 
integrity completion can reduce the number of disk operations that
are required for applications that don't need the guarantees of file 
integrity completion.
```

DIO 虽然跳过了 page cache，但 block device 也有 cache（[Ensuring data reaches disk](https://lwn.net/Articles/457667/)），DIO 也就是直接写到了这个 cache 里，如果数据没有从这里写下去的话，掉电仍可能丢数据，所以需要 fsync。fsync 的性能非常依赖 block device cache 的特性，如果它是 non-volatile、battery-backed 的性能就会很好；如果不带掉电保护的，fsync 性能就会很差：

- [从共识算法开谈 - 硬盘性能的最大几个误解](https://zhuanlan.zhihu.com/p/55658164)
- [Fsync Performance on Storage Devices](https://www.percona.com/blog/2018/02/08/fsync-performance-storage-devices/)
- 我的 PC 4KB randwrite IOPS 可以到 150k，但 fsync 延迟有几毫秒。

这也涉及到了另外一个问题：IOPS 高不意味着 fsync 性能就好，因为用 `fio` 测 IOPS 一般会用 `libaio` 且不 fsync 来测，如果要测 fsync 的性能，推荐用 [pg_test_fsync](https://www.postgresql.org/docs/12/pgtestfsync.html)。在看 [`queue`](https://www.kernel.org/doc/Documentation/block/queue-sysfs.rst) 文档时发现了 `write_cache` 的配置，看起来是告诉 kernel 如何对待 fsync 一类的操作，如果是 `write back` 就会从 block device cache 刷数据下去，`write through` 就不会刷因为已经直接写到 block device 了。我的 PC 默认是 `write back`，改成 `write through` 后 fsync 性能提升了 100 倍，但它不会改变 block device 的工作模式，所以擅自从 `write back` 改为 `write through` 会有丢数据的风险，不过看起来文件系统和 block device 有配合不好的可能，这时候就需要改它了 [XFS, nobarrier and the 4.13 Linux kernel](http://smalldatum.blogspot.com/2018/01/xfs-nobarrier-and-413-linux-kernel.html)。

