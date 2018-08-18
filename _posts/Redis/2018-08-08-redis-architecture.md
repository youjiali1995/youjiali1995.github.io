---
title: Redis 从单机到分布式
excerpt: 填坑，这篇文章算是对 Redis 系列的一个总结。
layout: post
categories: Redis
---

{% include toc %}

版本 [5.0-rc4](https://github.com/antirez/redis/tree/5.0-rc4)。

## Model
`Redis` 的模型是单线程 `I/O` 多路复用，优点是简单，所有操作都是原子顺序执行的，既简化了编程实现，也方便了客户端的使用。
单线程就要求每个操作不能阻塞太久，`Redis` 内部处处体现了均摊的思想：把耗时长的操作拆分为多次，每次只执行一小部分，如 `incremental rehash`。
`Redis` 其实还有多个 `background I/O` 线程，用于执行一些比较耗时的操作：如 `fsync`、`close` 和 `lazy free`。

`Redis` 所有操作都在内存中执行，性能很高。不开启持久化，同机房内简单的 `get/set` 操作 `QPS` 在 `10w` 级别，`latency` 在 `1ms` 以内；若使用 `pipeline`，`QPS` 可达
`50w`，`latency` 也就 `1ms` 上下。

## Persistence
`Redis` 持久化有 `2` 种方式：`RDB` 和 `AOF`，是对性能和数据完整性的权衡：
* `RDB` 对性能几乎无影响，丢的数据多。
* `AOF` 对性能有影响，丢的数据少。

### RDB
`RDB` 是 `fork` 出子进程，对当前的数据集做一次 `snapshot` 记录到磁盘上。对主进程的影响只有 `fork` 的耗时，但 `fork` 是一个比较重的操作，
耗时一般在数十毫秒，`RDB` 也会对磁盘 `I/O` 造成很大压力，所以不能频繁使用。

`RDB` 的格式紧凑，记录的信息也更丰富，如淘汰相关的 `LRU` 和 `LFU` 信息，加载速度也快，但两次 `RDB` 之间的写入就有可能丢掉。

### AOF
若开启 `AOF`，`Redis` 会在返回给客户端成功响应前将写操作记录到文件中。`Redis` 不是在处理命令时，就记录 `AOF` 并返回响应，
而是在进入下一轮事件循环前，批量写 `AOF` 和客户端响应，这样可以减少 `write` 的调用次数，一般而言 `batch` 写比写多次性能高。

`Redis` 目前的实现是在主线程阻塞写 `AOF`，所以开启 `AOF` 对性能有影响。持久化除了 `write` 操作，`fsync` 的调用也很重要：
* `write`：若打开文件时未使用 `O_SYNC || O_DSYNC` 之类的标志，数据会存放在内核缓冲区中，等待操作系统定期刷新到磁盘上，`Linux` 一般为 `30s`。能够保证进程挂掉的情况下数据的安全性。
* `fsync`：把内核缓冲区的数据刷新到磁盘上，从而保证断电或其他情况下数据的安全性。

因为 `fsync` 是个非常重的操作，`Redis` 提供了 `appendfsync` 配置由用户选择 `fsync` 的调用频率:
* `no`：从不调用 `fsync`。
* `everysecond`：每秒在 `bio` 线程调用一次 `fsync`，同时作者写发现在另外一个线程的 `fsync` 会阻塞其他线程对相同文件的 `write` 操作，见 [fsync() on a different thread: apparently a useless trick](http://oldblog.antirez.com/post/fsync-different-thread-useless.html)。
* `always`：每次写 `AOF` 之后都会在主线程调用一次 `fsync`，最强的保证。

不同的 `appendfsync` 配置对应了不同的数据安全性要求，也影响了 `Redis` 对写 `AOF` 出错时的处理，当磁盘出问题或空间不足时，`write` 操作可能会出错或者 `short write`：
* `no || everysecond`：当写 `AOF` 出错时，会尝试 `truncate` 掉 `short write` 的部分，然后每秒重试写 `AOF`，在写成功之前会拒绝新的写请求。
* `always`：立刻退出。

采取上面的处理方式是因为 `Redis` 在发生写 `AOF` 错误时，仍会把成功的响应返回给客户端。因为 `no || everysecond` 本身就不能保证所有数据的安全性，所以丢失少量写命令是可以接受的。
而 `always` 需要保证每条写命令的安全性，所以在写失败时不能返回成功响应。

`AOF` 的格式即 `RESP` 格式，逐条执行来恢复数据，所以恢复速度较 `RDB` 慢。`AOF` 的格式不紧凑，且冗余数据较多，当 `AOF` 文件大小增长超过一定比例时，会触发 `aofrewrite`：
`fork` 出子进程，对当前的数据集做一次 `AOF` 格式的 `snapshot`。在 `aofrewrite` 过程中的增量命令仍会写入当前的 `AOF` 文件，同时累积在 `aof_rewrite_buf_blocks` 中，当
`aofrewrite` 结束时把增量命令写入新的 `AOF` 文件中，然后 `rename` 覆盖掉原先的 `AOF` 文件，完成原子的切换。为了避免写增量数据阻塞太久，主进程会将增量命令通过管道发送给
子进程，从而保证最终需要写的增量数据很少。当一个文件被 `rename` 覆盖掉时，若没有 `fd` 指向该文件，则该文件会被 `unlink`；若有 `fd`，则在 `fd` 引用计数为 `0` 时，`unlink` 该文件。
`Redis` 为了防止 `rename` 操作的阻塞，`unlink` 的影响会放在 `bio` 线程。

### aof-use-rdb-preamble
`RDB` 和 `AOF` 在之前版本的实现是不相干的，不过没理由不能把 `RDB` 和 `AOF` 结合起来:
* `RDB` 记录某一时刻的 `snapshot`，`AOF` 记录之后的增量命令。
* 当文件大小超过一定限制时，执行一次 `compaction`，创建新的 `RDB`，增量命令仍然是 `AOF`。

`aof-use-rdb-preamble` 就是类似的机制，逻辑和 `aofrewrite` 相同，区别只是子进程做 `RDB` 而不是 `AOF`。

## Replication
1. 什么标记数据集

server.master_repl_offset 是添加在 repl_backlog 中的数据数量
client.reploff 是应用的量
ping 竟然也更新 repl_offset
\n 保活 双向的


slave 发送的是要求的下一个 byte 的 offset，server.master_repl_offset 是添加到 backlog 的总的字节数，server.repl_backlog_off 是 backlog 中第一个 byte 的offset，所以比较时
才会那样比较。shift master 的时候也会 + 1
2. 同步过程
slave 发送 cached master？

3. 主从切换

4. 级联 slave

## Sentinel

## Cluster

## 一些坑
