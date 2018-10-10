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

`Redis` 所有操作都在内存中执行，性能很高。不开启持久化，同机房内单节点简单的 `get/set` 操作 `QPS` 在 `10w` 级别，`latency` 在 `1ms` 以内。

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
而 `always` 需要保证每条写命令的安全性，所以在写失败时不能返回成功响应，`Redis` 最近才修复一个相关的 `BUG`，因为事件循环是先处理读事件，然后是写事件，如果连接之前注册了写事件，
会在一个事件循环里把当前请求的响应发送出去，这就违背了先写 `AOF` 再写响应的原则，有可能导致数据丢失，修复之后会先处理写事件再处理读事件。

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

`Redis` 的 `Replication` 是 `RSM` 模型，采用异步复制，提供最终一致性，而且是 `last-failover-wins`，这种方式性能很高；只负责复制，不负责高可用，高可用由其他 `layer` 提供，如 `Redis Sentinel`、`Redis Cluster`。

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

### Async Replication
对于 `master` 而言，`slave` 只是特殊的 `client` 而已，`master` 对 `slave` 也是如此。`master` 会把执行完成的写命令作为 `slave` 的 `reply` 发送给 `slave`，`slave` 一般会拒绝
写操作，但会执行 `master` 发过来的，`slave` 也不会发送响应给 `master`。`master` 和 `slave` 之间是 `TCP` 长连接，`Redis` 依赖 `TCP` 的可靠性，认为要么超时、要么报错，否则数据一定会到达 `slave`，
所以在增量同步的过程中不会对 `slave` 的同步进度进行校验。

### 数据一致性
`RSM` 的模型是当所有状态机按照相同顺序执行相同的命令时，所有状态机的状态是一致的。为了保证一致性，需要保证 `slave` 执行的命令都是从 `master` 传来的，所以内部的一些功能如过期、淘汰等就需要调整。

`master` 在过期或者淘汰时，会给 `slave` 发送 `DELETE` 命令。`slave` 不主动删除过期的数据，但是访问过期数据时会返回空，会等待 `master` 传来删除命令再删除。但是 `slave` 在目前的实现中会触发淘汰，
即使 `maxmemory` 设置相同，也有可能 `slave` 先淘汰，这会导致主从数据不一致，在之后的版本可能会修改。

`Redis` 提供的是最终一致性，当 `slave` 和 `master` 的连接断开时，如果设置了 `slave-serve-stale-data`，`slave` 仍然会接收读请求。

### Partial Resync
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

### Keepalive
`master` 和 `slave` 间有保活机制，超时会释放连接:
* `master` 定期给 `slave` 发送 `PING`。
* `slave` 定期给 `master` 发送 `REPLCONF ACK offset`，`master` 记录各个 `slave` 的 `repl_ack_time`，同时 `master` 会记录各个 `slave` 的 `Replication Offset`，用于 `WAIT` 命令。

在 `Full Resync` 时，`slave` 会清空数据并加载 `RDB`，可能会阻塞很久。为了防止这时候超时，`slave` 会在清空数据和加载数据时间歇性发送 `\n` 给 `master` 来保活。

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
`master` 同时处理写请求，也就是发生了 `split brain`，当网络恢复后，旧 `master` 触发 `Full Resync` 会导致数据丢失。异步复制想要在这种情况下不丢失任何数据很困难，`Redis` 为了减少丢失，有如下配置：
* `min-slaves-to-write`
* `min-slaves-max-lag`

只有当 `master` 的 `online slave` 中有大于等于 `min-slaves-to-write` 个 `slave` 的 `repl_ack_time <= min-slaves-max-lag` 才处理写命令。

### 级联 slave
若是所有 `slave` 都挂载在 `master` 上，会给 `master` 带来很大的复制压力。`Redis` 支持级联 `slave`，此时 `slave` 会作为代理，把从 `master` 接收到的数据发送给自己的 `slave`，自己不再产生复制数据，
保证所有节点数据的一致。

级联 `slave` 的也从 `slave` 侧的 `Replication Backlog` 中受益，短暂断连只需要部分同步。若一个有 `slave` 的节点挂载到其他节点上，会立刻释放掉所有 `slave` 的连接，
当该节点与新 `master` 同步完成后，会接受自己 `slave` 的同步请求，保证了数据的一致。

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
`Redis Sentinel` 提供了 `Redis` 的高可用，主要功能是在 `master` 不可用的情况下，执行主从切换。`Redis Sentinel` 同样是分布式的，每个 `sentinel` 可以监控多组 `replicas`，每组 `replicas` 同时被
多个 `sentinel` 监控。监控相同 `replicas` 的 `sentinel` 共同协作判定 `master` 是否出错和执行主从切换，能够减少 `false positive` 的发生并提供 `sentinel` 的高可用。

`Redis Sentinel` 没用过，也不太了解，我认为 `sentinel` 需要考虑的几个问题如下：
* 节点的发现和节点配置变更(增加或删除节点)，节点包括 `redis` 和 `sentinel` 节点。
* 节点之间如何通信。
* 如何判断一个 `master` 不可用。
* 如何执行 `failover`。
* 异常情况处理，如 `failover` 执行失败。
* 使用方式。

### Architecture
`Redis Sentinel` 和 `Redis Server` 是一套代码，在使用 `sentinel` 模式时，只支持 `sentinel` 的命令，并增加了 `sentinel` 定时器，所有操作都通过定时器触发。`sentinel` 与其他节点的通信
使用 `hiredis` 的异步接口，嵌入在 `Redis` 的 `eventloop` 中，因为在相同线程的 `eventloop` 中，使用起来和同步接口一样方便，但有着异步接口的灵活性和性能。`sentinel` 与节点的通信同样使用 `TCP` 长连接，
依托于 `TCP` 的顺序性和可靠性，极大简化实现，如不用处理过时消息和考虑消息的顺序等。

### 节点发现
`sentinel` 的元数据都保存在 `sentinel.conf` 中，`sentinel` 在启动时加载 `sentinel monitor <name> <host> <port> <quorum>` 找到需要监控的 `master` 地址，每组 `replicas` 使用 `name` 区分，相应的
监控配置也是针对不同的 `name`。`sentinel` 会为每个节点建立连接，包括 `Redis` 节点和其他的 `sentinel` 节点，
除 `sentinel` 节点外，每个节点会建立 `2` 个连接：一个用于发送命令；一个是 `pub/sub` 连接，接收订阅的 `channel` 消息。

在正常情况下，`sentinel` 会每隔 `SENTINEL_INFO_PERIOD(10s)` 给每个 `Redis` 节点发送 `INFO` 命令，`master` 的 `INFO REPLICATION` 中有 `slave` 的信息，`sentinel` 通过这些信息找到 `master` 对应 `slave` 的
地址，并添加进监控。

`sentinel` 会订阅每个 `Redis` 的 `SENTINEL_HELLO_CHANNEL(__sentinel__:hello)`，同时每隔 `SENTINEL_PUBLISH_PERIOD(2s)` 对每组 `replicas` 各节点的 `SENTINEL_HELLO_CHANNEL` 发布一条：
> sentinel_ip,sentinel_port,sentinel_runid,current_epoch,
> master_name,master_ip,master_port,master_config_epoch.

`sentinel` 通过接收到的订阅消息，找到监控相同 `master_name` 的 `sentinel`。

### 异常检测
`sentinel` 每隔最多 `SENTINEL_PING_PERIOD(1s)` 给每个节点发送 `PING` 来判断节点是否正常，若超过 `sentinel down-after-milliseconds <name> <milliseconds>` 配置的时间未接收到有效响应就会认为该节点不可用。
单个 `sentinel` 认为的节点不可用是 `SDOWN(Subjectively Down)`，对于 `SDOWN` 状态的 `master`，`sentinel` 每隔 `SENTINEL_ASK_PERIOD(1s)` 向其他 `sentinel` 发送 `SENTINEL IS-MASTER-DOWN-BY-ADDR <ip> <port> <current-epoch> *` 
询问它的状态，只有大于等于 `quorum` 个 `sentinel` 在 `SENTINEL_ASK_PERIOD*5(5s)` 内同时认为该节点不可用才会变为 `ODOWN(Objectively Down)` 并执行 `failover`，这样能够减少 `false positive` 的发生，`quorum` 使用  `sentinel monitor <name> <host> <port> <quorum>` 配置，不要求是 `majority`。

### Leader Election
因为有多个 `sentinel` 同时监控一组 `replicas`，需要选举出一个 `leader` 来执行 `failover`，`Redis Sentinel` 使用类似 `Raft` 的 `leader election` 算法：
* 使用 `epoch` 来区分每一次选举，每个 `epoch` 只能有一个 `leader`，每个 `sentinel` 在一个 `epoch` 只会给一个 `sentinel` 投票，收到 `majority` 的投票的 `sentinel` 成为 `leader`。

`Redis Sentinel` 中有多个 `epoch`，这里解释一下:
* `current_epoch`：`sentinel` 维护的单调递增的 `epoch`，会在发起选举时增加，在监控相同 `replicas` 的 `sentinel` 间同步，类似 `Raft` 的 `term`。
* `leader_epoch`：记录投票的 `epoch`，保证每个 `epoch` 只投给一个节点。
* `failover_epoch`：发起投票时的 `epoch`，用于区分不同的 `failover`。
* `config_epoch`：`replicas` 的主从配置 `epoch`，即完成的 `failover_epoch`。

在发现 `master` 处于 `ODOWN` 时，`sentinel` 会增加自己的 `current_epoch`，并向其他 `sentinel` 发送 `SENTINEL IS-MASTER-DOWN-BY-ADDR <ip> <port> <current-epoch> <runid>`，这个命令除了返回某个 `master` 的状态
还会发起投票，若目标 `sentinel` 发现参数 `req_epoch` 比之前投票的 `leader_epoch` 大就会投票并更新 `leader_epoch` 和投票的 `leader`，否则就返回当前的 `leader_epoch` 和 `leader`。收到 `majority` 投票的 `sentinel` 就会成为 `leader` 执行 `failover`。

### Failover
`leader` 会挑选合适的 `slave` 成为新的 `master`：
* 排除 `DOWN` 状态、太久没回应 `PING` 或者 `INFO`、与 `master` 断连太久、优先级为 `0` 的 `slave`。优先级使用 `slave-priority` 配置。
* 在剩下的 `slave` 中按照优先级、`repl_offset`、`runid` 进行排序，优先级越小、`repl_offset` 越大、`runid` 越小则选中机会越大。

挑选出 `slave` 后发送 `SLAVEOF NO ONE` 成为新 `master`，从 `INFO` 中发现 `role` 从 `slave->master` 后，将其他 `slave` 复制到新 `master` 上，从而完成 `failover`。

### 异常处理
写出理想情况下正常工作的代码很容易，要写出在任何环境下都能正常工作的代码就很困难，分布式系统中大部分代码都是为了处理异常情况。首先来看 `leader election` 相关的异常，`Redis Sentinel` 会遇到和 `Raft` 同样的问题，
如 `split vote`、选举失败等，最终的解决办法都是重新选举，同时为了减少 `split vote` 会增加一些随机性。`sentinel failover-timeout <master-name> <milliseconds>` 设置了 `failover` 的超时时间。`sentinel` 在投票给其他节点时，
会记录 `failover_start_time`，在 `2*failover_timeout` 的时间内不会对相同 `master` 发起新一轮 `failover` 的选举。当发起选举的 `sentinel` 超过 `failover_timeout` 仍未选出有效的 `leader`，会进行新一轮选举。
为了减少 `split vote`，`Redis Sentinel` 在 `2` 个地方增加了随机性：
* `hz` 会在 `[CONFIG_DEFAULT_HZ, 2*CONFIG_DEFAULT_HZ)` 随机变化，从而各 `sentinel` 定时器调用频率不同。
* `failover_start_time` 会在当前时间基础上随机增加最多 `SENTINEL_MAX_DESYNC(1s)`，降低同时发生超时的概率。

`Redis Sentinel` 在 `2012` 年就实现了，而 `Raft` 在 `2014` 年发表，除选举算法外，其余的实现差别很大。`Redis Sentinel` 的 `leader` 和 `Raft` 有几点不同，这增加了复杂性：
* 只有 `failover` 时才会选出 `leader`，`leader` 是一次性的。
* 即使 `leader` 在 `sentinel` 角度看来是有效的，但是从 `Redis` 角度来看可能是无效的，比如 `leader` 无法连上任意一个 `Redis` 节点，也就无法完成 `failover`。
* `leader` 需要执行的 `failover` 的复杂性，如某些 `slave` 不能立刻复制到新的 `master`、在 `failover` 过程中原先的 `master` 恢复了、新的 `master` 在 `failover` 过程中挂了。
* `leader` 不是和 `follower` 进行交互，而是与其他 `Redis` 节点交互，所以不太容易采取 `Raft` 一样的策略通过 `majority` 的提交来保证正确性，正确性对于 `sentinel` 来说是保证 `failover` 的唯一性。
* `sentinel` 会同时监控多组 `replicas`，可能同时成为多组 `replicas` 的 `leader`。

其中最显著的差别是第 `4` 点，这极大影响了 `sentinel` 的实现。`Redis Sentinel` 只保证了相同 `epoch` 不会有多个 `leader`，但是可能会有多个处于不同 `epoch` 的 `leader` 在执行 `failover`，
这就出现了 `split brain`，在 `Redis Sentinel` 目前的实现中是不可避免的，因为 `leader` 在执行 `failover` 的过程中，不再与其他 `sentinel` 交互:
* `leader` 节点不会发送 `heartbeat` 来防止其他 `sentinel` 重新选举。
* `failover` 的执行不用经过 `majority` 的提交。
* `leader` 不会 `step down`，即使有 `epoch` 更高的 `leader` 产生。这是因为如果发生了 `network partition`，因为不需要 `majority` 的保证，仍然无法避免 `split brain`。

`split brain` 无法避免，但是 `Redis Sentinel` 会解决 `split brain`，不会影响结果。这简化了 `Redis Sentinel` 对于前 `2` 个问题的处理：
* 将 `leader election` 和 `failover` 作为一个整体，非 `leader` 的 `failover_timeout` 针对的是 `failover` 完成而不是 `leader election` 完成，若超时会触发选举选出新的 `leader` 即使之前的 `leader` 还存在。

`failover` 完成对于不同状态的 `sentinel` 有不同的含义：
* 对于 `leader` 而言，要等待所有 `slave` 都复制到新的 `master`。
* 对于非 `leader` 而言，新 `master` 的 `SLAVEOF NO ONE` 完成 `failover` 就完成。当 `leader` 发现挑选的 `slave` 转变为 `master` 后，会通过 `SENTINEL_HELLO_CHANNEL` 发送 `hello` 消息给其他 `sentinel`，消息中包含新 `master` 的地址，
其他 `sentinel` 收到这个消息时会重置该 `replicas` 的状态，切换到新的 `master` 配置，从而开始监控新的 `master`。

现在可以来看一下 `sentinel` 是如何处理 `split brain` 了，原则如下：
* 若某个 `leader` 率先完成了 `SLAVEOF NO ONE` 操作，则其他 `leader` 中断 `failover`。
* 若多个 `leader` 同时完成 `SLAVEOF NO ONE` 操作，则 `epoch` 最大的胜利(`last failover wins`)，其余 `leader` 中断 `failover`。

`leader` 会记录执行 `failover` 时的 `epoch`，当 `SLAVEOF NO ONE` 操作完成时，会设置对应 `replicas` 的 `config_epoch` 为 `failover_epoch`。通过 `hello` 消息，其余 `sentinel` 会接收到每组 `replicas` 的 `config_epoch`，
若比自己的大就会中断 `failover`，更新配置。

若是 `leader` 之间无法通信，但是都能连上 `Redis` 节点，且 `leader` 挑选不同的 `slave` 为新的 `master`，可能导致多个 `leader` 一直执行不同配置的 `failover`。`Redis Sentinel` 很巧妙的解决了这个问题:
* `sentinel` 通过 `Redis` 节点作为中介通信 `hello` 消息，而不是直接通信。
* 在这种实现下，`leader` 之间无法通信的情况只能是不同 `leader` 能够连上的 `Redis` 节点之间无重叠，这就回退到了 `Redis Replication` 的 `split brain`，`sentinel` 是无能为力的，`Redis` 会通过配置减少数据丢失。

`failover` 操作本身就比较复杂，在很多阶段都会发生异常:
* `leader` 在执行 `failover` 的每个阶段也会有 `failover_timeout` 超时时间，超时会中断 `failover`。`sentinel` 通过超时重新选举能够解决大部分问题，如挑选的 `slave` 挂了、执行 `failover` 的 `sentinel` 挂了。
* 若是 `failover` 过程中新 `master` 又挂了，会重新选举执行新一轮 `failover`；若老 `master` 恢复了，`failover` 会继续执行。
* 对于不能立刻复制到新 `master` 的 `slave`，如之前 `ODOWN` 的 `master`，该次 `failover` 的 `leader sentinel` 会一直重试直到成功；其他的 `sentinel` 也会帮助设置，为了处理 `leader sentinel` 挂掉的场景。

### 多组 Replicas
`sentinel` 会同时监控多组 `replicas`，所以单个 `sentinel` 可能同时成为多组 `replicas` 的 `leader`。`Redis Sentinel` 对于一个 `epoch` 只会有一个 `leader`，但不要求当前的 `leader` 状态一定是 `current_epoch` 的，
所以会保持不同 `epoch` 的 `leader` 状态，从而支持同时 `failover` 多组 `replicas`。

### 配置变更
配置变更包括增加或删除新的节点(`Redis` 节点或者 `sentinel` 节点)、监控新的 `replicas` 等。因为 `sentinel` 有自动发现节点的机制，所以增加节点很容易，而删除节点是通过 `reset` 当前状态重新发现实现的。值得一提的是，
和 `Raft` 一样，在给一组 `replicas` 增加或者删除 `sentinel` 时，`Redis` 建议一次只增加或删除一个，防止同一 `epoch` 产生多个 `leader`。

### Manual Failover
手动执行 `failover` 要通过 `sentinel` 执行，否则 `sentinel` 发现配置不同，会强制恢复到 `sentinel` 认为的主从配置。逻辑和自动 `failover` 相同，接收到 `failover` 的 `sentinel` 会立刻发起选举。

### 使用
因为 `failover` 会变更 `replicas` 的主从状态，客户端需要重新路由。`sentinel` 在发送 `SLAVEOF` 命令时，会让 `Redis` 断连所有客户端，客户端在重连时可以先从 `sentinel` 获取当前 `master` 的地址再连接。
`sentinel` 还提供了调用脚本的功能，在发生 `failover` 导致配置变更时，会调用用户设置的脚本，可以执行一些通知或者客户端配置变更等操作，同时 `sentinel` 还会在事件发生时，向相应的 `channel` 广播消息，其他
服务可以订阅这些消息进行处理。

`sentinel` 的部署要精心考虑，因为:
* `sentinel` 根据与 `Redis` 的通信来发现问题。
* 根据与其他 `sentinel` 的通信结果来确定问题。
* 触发的 `failover` 是不能取消的(上面提到了 `split brain` 时的冲突解决有可能取消 `failover`，但对于 `Redis` 而言，`failover` 一定发生了)。

在某些场景下不发生 `failover` 会更好，如 `sentinel` 和 `Redis` 都是两机房部署且 `majority sentinel` 和 `majority Redis` 在不同的机房，`master` 在 `majority Redis` 的机房，此时发生 `network partition`。
`majority sentinel` 会 `failover` 出新的 `master`，若是配置了 `min-slaves-to-write` 为 `quorum`，新选举的 `master` 会拒绝写入，原先的 `master` 仍然正常写入，当网络恢复后，会强制让原先的 `master` 复制
新的 `master` 就导致了数据的丢失。

为了减少 `false positive` 和数据丢失，`Redis` 建议 `sentinel` 和客户端部署在一起，保证了 `sentinel` 和客户端对 `Redis` 的可用认知是一致的。但一般而言，会将 `Redis` 作为服务提供给用户，`sentinel` 自然
不能部署在用户侧，通常是一组 `sentinel` 部署在多个机房同时监控成千上百个 `Redis` 实例，多机房(>=3)部署是为了实现机房容灾，保证单个机房挂掉不会影响 `majority` 的 `sentinel`。

## Cluster
`Redis` 各模块间耦合度很低，修改少量代码即可增加新的功能，如 `Replication` 是在 `Persistence` 之上实现的，`Redis Cluster` 是在 `Replication` 之上实现的。`Redis Cluster` 增加了 `sharding` 和 `HA` 功能，
提供了开箱即用的分布式存储。

### Sharding
`Redis Cluster` 将整个数据空间划分为 `16384` 个 `slots`，使用 `crc16` 将每个 `key` 散列到对应的 `slot` 实现 `sharding`。每个 `master` 节点可以负责 `0~16384` 个 `slot`，但只有有 `slot` 的
`master` 节点可以参与 `failover` 的投票，为了实现容灾，至少要有 `3` 个。

在集群搭建阶段使用 `CLUSTER ADDSLOTS` 命令给每个 `master` 分配负责的 `slots`，这些信息会同步给其他节点来提供路由功能。当访问不属于该节点负责的 `slot` 时，或者访问的是负责
该 `slot` 的 `master` 的 `slave` 时(使用 `READONLY` 命令就不会返回)，会返回 `-MOVED` 错误，客户端可以根据这个信息刷新路由表。`Sharding` 会影响操作多个 `key` 的命令和事务，集群模式下只支持所有 `key` 在
相同 `slot` 的相关命令，因为若是支持跨 `slot`，可能这次都落在同一个节点，但因为 `resharding` 导致之后的执行失败。

当要扩缩容时，就要迁移 `slot`，分为如下几步：
1. 在目标节点设置 `CLUSTER SETSLOT slot IMPORTING src-node-id`；
2. 在源节点设置 `CLUSTER SETSLOT slot MIGRATING dst-node-id`；
3. 在源节点调用 `CLUSTER GETKEYSINSLOT slot count` 获取 `slot` 的 `key`(`Redis` 使用 `radix-tree` 保存 `slot->key` 的映射，会给每个 `key` 加上 `little-endian` 格式的 `slot` 前缀)，
然后使用 `MIGRATE` 命令迁移到目标节点。
4. 当 `slot` 中所有 `key` 都迁移完成后，(建议)对集群内所有 `master` 设置 `CLUSTER SETSLOT slot NODE dst-node-id`，更新路由信息。

按 `key` 级别进行迁移能够减少阻塞，降低迁移的影响。客户端访问迁移中的 `slot` 会按照特定的协议：
1. 先访问源节点，若 `key` 存在，则在源节点执行命令，否则返回 `-ASK` 错误。
2. 在目标节点先执行 `ASKING` 命令，然后执行真正的命令。

这种策略保证了不影响迁移中 `slot` 的正常访问，只是会增加重定向，但是 `slot` 迁移状态是节点自己保存的，不会同步给其他节点，导致客户端访问迁移节点的 `slave` 时不会正确重定向，这会带来问题：
* 访问源节点的 `slave` 时发现 `key` 不存在并返回 `null`，其实已经迁移到目标节点了。
* 访问目标节点的 `slave` 时会重定向到源节点，导致循环重定向。

但是为了简单和一致性，`MIGRATE` 命令是阻塞的，它的执行逻辑如下：
* 源节点 `dump` `key` 为 `RDB` 格式并使用 `syncio` 发送 `RESTORE-ASKING` 命令给目标节点；
* 目标节点 `restore` 完成之后返回 `OK`；
* 源节点接收到目标节点响应后，删除 `key`，`MIGRATE` 命令完成。

以上所有操作都是阻塞的，这时候节点无法处理请求，更严重的是无法处理心跳信息，可能会导致 `failover`，这是 `Redis Cluster` 亟需解决的问题，好消息是 `non-blocking migrate` 一直在努力中。

### Gossip
`Redis Cluster` 各节点间使用端口 `port+10000` 通信，复用处理请求的 `eventloop`，由定时器来触发集群间的交互工作。它是无中心节点的，采用 `Gossip` 协议来实现节点发现、异常检测和更新配置等。
简单来说，`Gossip` 协议就是各节点周期性向集群内随机一个或几个节点发送消息，经过一定时间后，集群内各节点会对集群的状态达成一致。消息中还会携带自己知晓的部分其他节点的信息，
其他节点发现自己未知晓的节点就会将它加入集群列表中，从而实现节点的传播。`Redis Cluster` 的实现如下：
* 各节点每秒随机挑选集群内一个节点发送 `PING` 消息，`PING` 消息中包含该节点的相关信息，同时包括它知晓的集群内 `1/10` 个节点的信息(`Gossip section`)。
* 接收方返回 `PONG` 响应，`PING` 和 `PONG` 只有类型不同，其余都相同。若发现在附加信息中有未知节点，会进行 `handshake`，完成之后加入集群。

### HA
`Redis Cluster` 中各节点通过 `PING` 来发现异常，若超过 `cluster-node-timeout` 未收到 `PONG` 就会标记该节点为 `PFAIL`。为了避免未被挑选发送 `PING` 导致的异常，如果在 `cluster-node-timeout/2` 的时间内
未收到某节点的 `PONG` 且未发送 `PING` 就会强制发送 `PING`。`Gossip section` 中会携带发送方视角的节点状态，为了加快故障检测的速度， `PFAIL` 状态的节点信息一定会添加在 `Gossip section` 中(在最初的实现中，
`PFAIL` 的节点同样是按照 `1/10` 挑选的)，如果在 `2*cluster-node-timeout` 时间范围内 `majority` 的 `master` 节点同时认为一个节点 `PFAIL` 了，就会标记该节点为 `FAIL` 状态，同时广播 `FAIL` 消息强制其他节点标记该节点为 `FAIL`，加快触发 `failover`。

`Redis Cluster` 与 `Redis Sentinel` 类似，同样有顺序递增、在集群内统一的 `current epoch`。`failover` 的区别在于是由 `slave` 触发，当 `slave` 发现自己的 `master` `FAIL` 时，就会给其他节点发起投票请求(`FAILOVER_AUTH_REQUEST`)，
只有 `slot` 个数不为零的 `master` 节点才能投票(后面的 `master` 均指可以投票的)，且每个 `epoch` 只能投票一次。当收到 `majority` 的 `master` 投票后，该 `slave` 成为新的 `master`。

为了减少数据丢失，同样要尽量选择较新的 `slave` 成为新的 `master`：
* 断连太久的 `slave` 不会触发 `failover`，见配置 `cluster-slave-validity-factor`。
* `slave` 在发送投票请求前会延迟 `500 + random()%500 + rank*1000` 毫秒，根据各 `slave` 的 `replication offset`(`PING` 消息中携带) 计算出 `rank`，
最新的 `rank` 为 `0`，越老的 `rank` 越大，使得最新的 `slave` 最先发起投票，同时 `random` 能够减少 `split-vote`。

为了增加 `failover` 的成功率，每个 `master` 在 `2*cluster-node-timeout` 时间范围内只会对某组 `replicas` 的一个 `slave` 投票。每个 `epoch` 只会选出一个 `leader`，但是这个 `leader` 是集群范围内的，
也就是说多个不同 `master` 的 `slave` 同时发起 `failover` 也会增加 `split-vote` 的风险。发生 `split-vote` 会导致 `failover` 恢复很慢，
因为重试 `failover` 需要 `max(2*cluster-node-timeout, 2000) * 2` 毫秒。

比较奇怪的是，`quorum` 是由集群内至少有 `1` 个 `slot` 的 `master` 节点个数决定的，它决定了标记 `FAIL` 和 `failover` 的投票个数。`failover` 投票只能由负责 `slot` 的 `master` 执行，
但 `FAIL` 状态是由集群内所有的 `master` 标记，这不统一。

### Config epoch
`Redis Cluster` 和 `Redis Sentinel` 相比除了主从配置，还多了 `slots` 配置，主从配置只需要在主从间传递即可，但是 `slots` 配置要在整个集群间传递，这增加了复杂度。`Redis Cluster` 巧妙的将这两种配置统一了起来：
* `config epoch` 只用于标记节点负责的 `slots`。当某个 `slot` 配置有冲突时，选择 `config epoch` 大的。
* 当节点发现自己的 `slots` 全部移交给其他节点负责了，就会成为该节点的 `slave`。

`config epoch` 是配置发生变更时的 `current epoch`，且 `config epoch` 不是全局统一的，因为 `slots` 迁移比较频繁、数量又多，
若要是全局统一的，则每迁移完成一个 `slot` 就要共识一下，代价太大。所以 `config epoch` 是由节点自己维护，当迁移完成时，
会不需要共识的增加 `config epoch` 来宣布 `slots` 配置的更新(`Redis Cluster` 的实现保证是集群中已知的最大的，因为要保证该节点的 `config epoch` 比原先负责该 `slot` 的节点大)。

`failover` 的结果就是新的 `master` 负责老 `master` 的所有 `slots`，集群通信消息中会携带自己的 `config epoch` 和负责的 `slots` 信息，
其他节点会根据 `slots` 配置的 `config epoch` 进行更新。如果节点发现自己的 `slots` 配置(`slave` 的配置即它的 `master` 的配置)与消息有冲突，且在更新 `slots` 配置之后，
自己不再负责 `slots`，就会成为消息发送者的 `slave`，从而实现主从配置变更。

只有集群中 `majority` 的 `master` 共识某节点 `FAIL` 才会触发 `failover`，那为什么 `failover` 仍需要一轮投票呢？目的是确保 `failover` 成功，因为 `failover` 的投票机制保证了 `config epoch` 是共识产生的，
就能保证新 `master` 的 `config epoch` 是最大的。而且产生的 `config epoch` 一定是唯一的，能够保证 `last-failover-wins`。

### Config epoch 冲突
`config epoch` 是用来解决单个 `slot` 的配置冲突的，需要保证单个 `slot` 在特定的 `config epoch` 只属于一个节点，如果不同的 `master` 负责的 `slots` 不重叠，即使有着相同的 `config epoch` 也
不会有什么问题，但考虑如下场景，两个节点在迁移 `slots`：
* 向目标节点设置 `CLUSTER SETSLOT slot NODE dst-node-id`，这个命令会使目标节点不需要共识的增加 `config epoch`，但还未同步到源节点。
* 源节点发生主从切换，`slave` 更新了 `config epoch`，同时获得原先 `master` 的 `slots`。(`slots` 迁移状态是 `master` 节点各自维护的，不会进行传播)。
* 此时源节点和目标节点都认为该 `slot` 属于自己，并且有可能两个节点的 `config epoch` 是一样的，导致 `slots` 配置出现冲突且无法自动修复。

`Redis Cluster` 为了解决这个问题，要求每个 `master` 的 `config epoch` 都不同，若不同的 `master` 有着相同的 `config epoch` 就会进行冲突解决：
* `node-id` 小的节点增加 `current epoch` 并设置其 `config epoch`。

这种做法能够保证一段时间后，集群内的 `slots` 配置是统一的，但统一不意味着有效。还是上面的场景，如果源节点的 `node-id` 比目标节点小，就会以源节点的配置为准，但此时该 `slot` 的所有 `key` 都
已经迁移到目标节点中了，更严重的是，目标节点发现不再属于自己的 `slot` 中还有 `key`，会清空 `slot`，导致整个 `slot` 的数据就丢失了。

### slots 相关操作
如果不理解 `Redis Cluster` 的实现，尤其是 `config epoch`，就会对 `slots` 相关的命令感到困惑。主要有这几个命令：
* `CLUSTER ADDSLOTS/DELSLOTS`：这两个命令只更新节点负责的 `slots` 配置，不会改变 `config epoch`。如果不关心 `slot` 中的数据，也可用这两个命令 `resharding`：在**所有**节点 `delslots`，在目标节点 `addslots`。
这是由 `slots` 配置冲突解决的实现决定的，节点只会将一个 `slot` 转移给 `config epoch` 大的节点，只在源节点 `delslots` 其他节点仍然认为该 `slot` 由源节点负责，直到其他节点宣布它的该 `slot` 负责才会更新配置。
* `CLUSTER SETSLOT slot IMPORTING/MIGRATING node-id`：这两个命令设置 `slot` 的迁移状态，一定要先在目标节点设置 `MIGRATING`，然后在源节点设置 `IMPORTING`，这是由路由策略决定的。
* `CLUSTER SETSLOT slot NODE node-id`：这个命令在不同的节点使用有不同的效果，除了会设置该 `slot` 由 `node-id` 对应节点负责并清理 `slot` 迁移状态，在目标节点使用还会更新 `config epoch`，所以要保证目标节点的调用一定成功。

因为 `slots` 的操作比较复杂，`redis-cli` 中提供了 `cluster` 相关工具来处理。

### 节点 migrate
`Redis Cluster` 有 `2` 种形式的节点 `migrate`：
* 当一个 `master` 有 `slots` 但没有 `slave` 时，`slave` 个数最多的 `master` 的 `slave` 就可能会迁移到该节点，详见 `cluster-migration-barrier` 配置。
* 主从切换时，也相当于是节点迁移。

这里主要讨论第二种，节点迁移的原因是发现自己不再负责 `slots`，这会造成一些奇怪的现象，比如缩容时，当 `master` 的 `slots` 迁移空时，其 `slave` 一定会复制到其他节点上，
而且因为消息的传播顺序，还不能确定复制到哪个节点。如果 `CLUSTER SETSLOT slot NODE dst-node-id` 命令只在目标节点执行未在源节点执行，就连 `master` 也会复制到其他节点。

### Manual Failover
`Redis Cluster` 的 `Manual failover` 更加可靠，命令格式为：`CLUSTER FAILOVER [FORCE|TAKEOVER]`。完整的流程如下：
1. `slave` 发送 `MFSTART` 消息给 `master`。
2. `master` 收到后停止处理客户端命令，并发送最新的 `replication offset` 给 `slave`。
3. `slave` 的 `replication offset` 与 `master` 一致后，进行 `failover`，流程和 `HA` 中相同，除了不会等待。

当 `master` 挂掉时，完整流程无法执行，需要增加选项：
* `FORCE` 选项会直接进行第 `3` 步。
* `TAKEOVER` 选项会直接增加 `config epoch`，不需要 `majority` 的共识，一般用在很极端的场景，比如集群部署在两个机房，其中一个机房挂掉导致无法自动 `failover`。

### 减少数据丢失
当节点发现 `majority` 的有 `slots` 的 `master` 节点都 `PFAIL|FAIL` 时，就会认为 `CLUSTER_DOWN` 并拒绝读写请求，因为很可能 `majority` 选出了新的 `master`。
认为 `CLUSTER_DOWN` 的节点(重启的节点默认为 `CLUSTER_DOWN`)重新连上 `majority` 后，会等待一段时间再接收读写请求，目的是等待更新集群配置。

## 总结
从毕业以来一直在做 `Redis` 相关的工作，也是 `Redis` 带我走进了存储、分布式的大门。`Redis` 的代码写的很通俗易懂，各模块划分清晰，很容易上手进行开发，但一旦涉及到分布式，就有很多细节需要考虑，
我也不能保证我理解的是正确的。以这篇博客作为总结，`Redis` 的学习就告一段落了，后面会重点学习数据库相关的知识。
