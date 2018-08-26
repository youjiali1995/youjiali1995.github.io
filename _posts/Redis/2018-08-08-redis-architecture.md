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
`aofrewrite` 结束时把增量命令写入新的 `AOF` 文件中，然后 `rename` 覆盖掉原先的 `AOF` 文件，完成原子的切换。

`Redis` 为了避免阻塞，进行了很多优化，如：
* 为了避免写增量数据阻塞太久，主进程会将增量命令通过管道发送给子进程，从而保证最终需要写的增量数据很少。
* 当一个文件被 `rename` 覆盖掉时，若没有 `fd` 指向该文件，则该文件会被 `unlink`；若有 `fd`，则在 `fd` 引用计数为 `0` 时，`unlink` 该文件。`Redis` 为了防止 `rename` 操作的阻塞，`unlink` 的影响会放在 `bio` 线程。

### aof-use-rdb-preamble
`RDB` 和 `AOF` 在之前版本的实现是不相干的，不过没理由不能把 `RDB` 和 `AOF` 结合起来:
* `RDB` 记录某一时刻的 `snapshot`，`AOF` 记录之后的增量命令。
* 当文件大小超过一定限制时，执行一次 `compaction`，创建新的 `RDB`，增量命令仍然是 `AOF`。

`aof-use-rdb-preamble` 就是类似的机制，逻辑和 `aofrewrite` 相同，区别只是子进程做 `RDB` 而不是 `AOF`。

## Replication
`Redis` 是纯内存数据库，磁盘上的数据只是为了重启时恢复数据，而不像其他的存储引擎如 `leveldb`、`rocksdb` 等，既用于恢复，也存放数据。若能够保证没有丢失数据的可能，或者
这种可能性很低，就可以降低对数据安全性的要求，甚至不开启持久化，从而获得最高的性能。

`Replication` 可以降低丢失数据的风险，若是多副本三机房部署，所有节点同时挂掉的可能性几乎为零。`Replication` 还能扩展读的性能，通过读 `slave`、写 `master` 来分摊 `master` 的
压力。

`Redis` 的 `Replication` 是 `RSM` 模型，采用异步复制，提供最终一致性，这种方式性能很高；只负责复制，不负责高可用，高可用由其他 `layer` 提供，如 `Redis Sentinel`、`Redis Cluster`。

后面的介绍不会按照 `Redis` 现有的复制流程介绍，但会介绍一些功能的演进。

### Full Resync
当一个 `slave` 第一次连上 `master` 时，会触发 `Full Resync`，需要将 `master` 的所有数据同步到 `slave`。流程类似 `aof-use-rdb-preamble` 的 `aofrewrite`，`master` 触发
`RDB`，然后把 `RDB` 发送给 `slave`，在这个过程中的增量命令会累积在 `slave` 的 `reply buffer` 中，发送完 `RDB` 之后再把增量命令同步到 `slave`。

有 `2` 种方式发送 `RDB`：
* `disk`：子进程将完整 `RDB` 写到文件中，之后由主进程每次读取部分数据发送给 `slave`。
* `socket`：子进程在生成 `RDB` 时通过阻塞 `socket I/O` 直接发送给 `slave`，复制结果会通过 `pipe` 返回给主进程。

`Redis` 通过 `rio` 接口很好的封装了不同方式的区别，同时 `Redis` 也为减少 `RDB` 的次数做了一些优化：
* 如果有 `disk` 类型的 `Full Resync` 正在进行，会复用这次 `RDB` 和 `slave` 的信息，避免了下一次 `RDB`。
* 触发 `socket` 类型 `Full Resync` 时，会等待 `repl-diskless-sync-delay` 秒再进行 `RDB`，为了一次性 `sync` 多个 `slave`。

### 增量同步
对于 `master` 而言，`slave` 只是特殊的 `client` 而已，`master` 对 `slave` 也是如此。`master` 会把接受到的写命令作为 `slave` 的 `reply` 发送给 `slave`，`slave` 一般会拒绝
写操作，但会执行 `master` 发过来的。`master` 和 `slave` 之间是 `TCP` 长连接，`Redis` 依赖 `TCP` 的可靠性，认为要么超时、要么报错，否则数据一定会到达 `slave`，
所以在增量同步的过程中不会对 `slave` 的同步进度进行校验。

### 数据一致性
`RSM` 的模型是当所有状态机按照相同顺序执行相同的命令时，所有状态机的状态是一致的。为了保证一致性，需要保证 `slave` 执行的命令都是从 `master` 传来的，所以内部的一些功能如过期、淘汰等就需要调整。

`master` 在过期或者淘汰时，会给 `slave` 发送 `DELETE` 命令。`slave` 不主动删除过期的数据，但是访问过期数据时会返回空，会等待 `master` 传来删除命令再删除。但是 `slave` 在目前的实现中会触发淘汰，
即使 `maxmemory` 设置相同，也有可能 `slave` 先淘汰，这会导致主从数据不一致，在之后的版本可能会修改。

### 短暂断连后的同步
在之前的实现中，当 `master` 和 `slave` 之间的连接出问题时，会重走一遍 `Full Resync` 的流程，既浪费 `CPU` 和网络资源，也没有必要。为了缓解这个问题，
`Redis` 增加了 `3` 个元素，实现了短暂断连后的部分同步：
* `Replication ID`
* `Replication Backlog`
* `Replication Offset`

思想很简单，将 `master` 接收到的最新的部分写命令缓冲在 `Replication Backlog` 中，由 `Replication Offset` 确定 `master` 和 `slave` 之间的同步差距，若差距在
 `Replication Backlog` 中只需发送增量命令；若落后太多，则重走 `Full Resync`。这是简单且有效的方法，能够解决大部分场景的问题。

在 `Redis` 中，`Replication Backlog` 实现为大小固定的环形缓冲区 `repl_backlog`，大小由 `repl-backlog-size` 配置。`master` 发送给 `slave` 的数据会添加到 `repl_backlog` 中，并记录其中
数据的范围：
* `master_repl_offset`：添加到 `repl_backlog` 中的总字节数，即 `master` 侧的 `Replication Offset`。
* `repl_backlog_off`：`repl_backlog` 中第一个字节的 `offset`，从 `1` 开始。
* `repl_backlog_histlen`：`repl_backlog` 中数据的长度。

`slave` 接收到 `master` 发送的命令时，会记录接收并应用到的命令的总字节数 `Replication Offset`，当主从之间连接断开后，`slave` 会发送期望的下一字节的 `offset`，`master` 根据
这个信息判断数据是否在 `Replication Backlog` 中来进行部分同步。

部分同步带来的一个新问题是如何区分不同的数据集，也即不同的历史，之前的实现每次断连或者复制其他的 `master` 都会触发 `Full Resync` 所以能够保证主从之间的数据是一致的，而
现在需要保证相同的历史才能进行部分同步。`Replication ID` 就用于区分不同的历史，在 `Redis` 中，`Replication ID` 是 `40` 字节的随机数。每个能够独立接收写命令的节点有不同的 `Replication ID`；
成功 `Full Resync` 的主从有相同的 `Replication ID`，也即相同的历史。

增加了部分同步后的完整流程如下：
1. `slave` 发送自己的 `Replication ID` 和期望的下一字节的 `Replication Offset`。
2. `master` 接收到后进行比较：
    * `Replication ID` 相同且 `Replication Offset` 在 `Replication Backlog` 中时，进行部分同步。
    * 否则，`master` 在发送 `RDB` 之前先发送当前的 `Replication ID` 和 `Replication Offset`，当 `slave` 接收并加载完 `RDB` 时，设置为 `master` 传来的 `Replication ID` 和 `Replication Offset`，从而保证
    主从之间有着相同的历史和数据。

### 主从切换
主从切换用途很多，比如 `master` 挂了，要切换到新的 `master`；`master` 机器有问题或者要调整机房等。在目前的实现中，主从切换分2步：
1. 对一个 `slave` 执行 `SLAVEOF no one`，成为新的主。
2. 对其他节点执行 `SLAVEOF ip port`，成为新 `master` 的 `slave`。

当 `slave` 提升到 `master` 时会改变自己的 `Replication ID`，因为开始了一段新的历史，`Replication Offset` 不会变。这是需要的，考虑如下场景：
* `slave` 提升为 `master` 时仍使用原先的 `Replication ID`，且接收了客户端的写命令，增加了 `Replication Offset`。
* 原先的 `master` 可能由于 `network partition` 或者 `SLAVEOF ` 发送延迟，仍接收到了客户端的写请求，增加了 `Replication Offset`。
* 此时旧 `master` 和新 `master` 成功执行部分同步，导致主从间数据不一致。

需要理解的是 `Replication ID` 不是为了区分数据集，而是为了区分接受命令的历史。

一旦主从切换，因为新 `master` 的 `Replication ID` 变化，必然导致所有 `slave` 的 `Full Resync`。为了主从切换时也可以部分同步，做出如下改变：
* 所有 `slave` 都会创建 `Replication Backlog` 记录从 `master` 传来的数据流，之前为了节省内存，只有 `master` 会创建。需要注意的是，`slave` 只能保存接收完整的命令，若是保存了部分命令，且
该 `slave` 成为新的主，会导致复制数据流的 `corruption`。
* 新的 `master` 会保存旧 `master` 的 `Replication ID(replid2)` 和应用到的 `Replication Offset(second_replid_offset)`。新 `master` 也会根据旧 `master` 的
同步进度来判断 `slave` 能否部分同步。需要注意的是，比新 `master` 同步进度新的 `slave` 不能部分同步，即使 `Replication Offset` 在 `Replication Backlog` 中。

### 数据丢失
因为 `Reids` 是异步复制，在某些情况下，数据丢失无法避免，如 `master` 接收到客户端写命令并返回成功，但没有同步给任何一个 `slave` 就挂掉。我们能做的就是在
所有节点正常工作时，保证不会丢失数据，及尽可能减少出错情况下丢失的数据。

在正常工作的情况下，唯一的风险点是主从切换：如果挑选的新 `master` 没有最新的数据。一种解决办法是类似 `Raft` 的 `leadership transfer`：`leader` 拒绝写，
等到目标 `transferee` 的 `log` 追上 `leader` 时让 `transferee` 立刻选举。`Redis` 没有原生提供这个功能，需要一些额外的处理：
1. 给旧 `master` 发送 `MULTI, INFO REPLICATION, CLIENT PAUSE duration, EXEC` 命令，获取当前 `master` 的 `Replication Offset` 并停止处理客户端请求。
2. 等待目标 `slave` 接收到 `master` 的最新数据，执行 `SLAVEOF no one` 成为新的 `master`。
3. 对其他 `slave` 和旧 `master` 执行 `SLAVEOF`，挂载到新的 `master` 上，要确保这一步之前没有新的写命令路由到旧 `master`。

这一套流程很麻烦，需要额外的工具。幸运的是，在 `Redis Cluster` 实现中能够确保主从切换时不丢失数据。

在出错的情况下，外在表现都是 `master` 连不通，但是有 2 种可能：
* `master` 挂了。
* `master` 被 `network partition` 了。

为了尽量减少数据丢失，需要选择最新的 `slave` 成为新的 `master`，通过比较各 `slave` 的 `Replication Offset` 即可。但是如果是 `network partition`，可能还有客户端连接在旧 `master` 上，此时有 2 个
`master` 同时处理写请求，也就是发生了脑裂，当网络恢复后，旧 `master` 触发 `Full Resync` 会导致数据丢失。异步复制想要在这种情况下不丢失任何数据很困难，`Redis` 为了减少丢失，有如下配置：
* `min-slaves-to-write`
* `min-slaves-max-lag`

只有当 `master` 的 `online slave` 中有超过 `min-slaves-to-write` 个 `slave` 的 `repl_ack_time <= min-slaves-max-lag` 才处理写命令。

*注*：`master` 和 `slave` 间有保活机制，超时会释放连接:
* `master` 定期给 `slave` 发送 `PING`。
* `slave` 定期给 `master` 发送 `REPLCONF ACK offset`。`master` 会记录各个 `slave` 的 `Replication Offset`，用于 `WAIT` 命令。

### 级联 slave
若是所有 `slave` 都挂载在 `master` 上，会给 `master` 带来很大的复制压力。`Redis` 支持级联 `slave`，此时 `slave` 会作为代理，把从 `master` 接收到的数据发送给自己 `slave`，保证所有节点
数据的一致。

级联 `slave` 的也从 `slave` 侧的 `Replication Backlog` 中受益，短暂断连只需要部分同步。若一个有 `slave` 的节点挂载到其他节点上，会立刻释放掉所有 `slave` 的连接，
当该节点与新 `master` 同步完成后，会接受自己 `slave` 的同步请求。

### 重启
`RDB` 文件中会保存 `replication` 相关的信息，如 `selected db`、`Replication ID` 和 `Replication Offset`，只有在配置文件中设置了 `slaveof`，重启时才会恢复这些信息尝试与 `master` 部分同步。

### 向后兼容
`Redis` 的主从复制实现一直在变化，从最初的 `Sync`、`Psync1.0` 到现在的 `Psync2.0`，`Redis` 会尽量向后兼容，因为主从复制一个很大的用途就是版本升级：通过挂载新版本的 `slave` 并替换掉旧版本的节点，
实现 `Redis` 的升级。在主从复制交互中，`slave` 会通过 `REPLCONF CAPA` 命令告诉 `master` 自己支持的功能，`master` 会忽略不支持的功能，只使用该 `slave` 支持的功能。

除了协议会影响兼容性，`RDB` 的版本同样会影响，`RDB` 中会保存 `RDB Version`，旧版本的 `Redis` 不能加载新版本的 `RDB` 文件。

### 未来
在目前的 `Redis` 实现中，`Replication` 和 `Persistence` 是不相关的，但是 `RDB+AOF` 天然提供了主从复制需要的一切数据。在未来，每个节点可以记录 `AOF` 中命令的索引，且主从间记录 `Operation Offset`，
来实现 `AOF` 部分同步。

## Sentinel

## Cluster

## 一些坑
