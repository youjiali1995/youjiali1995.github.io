---
title: TPC Raft Store(一) -- Raft
layout: post
excerpt: 简单介绍下几个 raft 库的实现。
categories: TiKV
published: false
---

{% include toc %}

## Raft 流程

### sync raft

以未使用 async-ready 的 [raft-rs](https://github.com/tikv/raft-rs) 为例，也相当于是 [etcd/raft](https://github.com/etcd-io/etcd/tree/main/raft)，一条 raft log 要经历如下流程才能被 apply：

1. leader [`propose`](https://github.com/tikv/raft-rs/blob/52d84aac8734369d81c2d77413ea3ab8e58e0af9/src/raw_node.rs#L354)s requests
2. leader handles [`ready`](https://github.com/tikv/raft-rs/blob/52d84aac8734369d81c2d77413ea3ab8e58e0af9/src/raw_node.rs#L478):
   1. send [`messages`](https://github.com/tikv/raft-rs/blob/52d84aac8734369d81c2d77413ea3ab8e58e0af9/src/raw_node.rs#L245) to followers
   2. persist [`proposed`](https://github.com/tikv/raft-rs/blob/52d84aac8734369d81c2d77413ea3ab8e58e0af9/src/raw_node.rs#L97) logs
3. follower receives messages from the leader and handles ready:
   1. persist proposed logs
   2. send messages to the leader
4. leader receives messages from followers and handles ready:
   1. apply [`committed`](https://github.com/tikv/raft-rs/blob/52d84aac8734369d81c2d77413ea3ab8e58e0af9/src/raw_node.rs#L244) logs

leader 可以并行处理 ready 的几个步骤，follower 需要持久化 log 后才能发送 message 给 leader。在理想情况下，每条 raft log 只需要 1 RTT + 1 disk I/O 的延迟就可以被 apply 到状态机：

- 1 RTT：log replication 的网络延迟。
- 1 disk I/O：所有 replica 并行持久化 raft log。

但在实际实现中要达成上述目标不是那么容易。

首先未使用 async-ready 的 raft-rs 是同步流程，单个 raft group 的 ready 需要处理完([`advance_append`](https://github.com/tikv/raft-rs/blob/52d84aac8734369d81c2d77413ea3ab8e58e0af9/src/raw_node.rs#L669))才能处理后面的请求，就不可避免地导致请求会多经历几次 disk I/O 延迟，最多要经历 4 次 disk I/O 才能被 apply：propose 时等待 leader 之前的 log 持久化完成 + 复制时等待 follower 之前的 log 持久化完成（并行持久化减少了发生的概率）+ 该 log 的持久化延迟 + 收到复制成功响应时等待 leader 后面的 log 持久化完成。如果是 multi-raft + 同步 I/O 的模型，那不同 raft group 间也会互相影响。

除了同步的 ready 流程外，raft 内部的循环不变量也可能会引入其他的顺序限制。通常来说 raft 内部 index 的大小关系如下：

```rust
1. (truncated + 1) == first <= last == persisted
2. (optional) applied <= committed
3. (optional) applied <= persisted
```

这里的 index 都是已持久化的，不包含还未持久化的 log index。

- `applied <= committed` 这个关系是可选的，因为 raft 论文不要求持久化 `committed`，虽然 `raft-rs` 把 `committed` 也包含在 hard state 里了（好像是为了保证 [conf change 的正确性](https://github.com/etcd-io/etcd/issues/7625#issuecomment-489232411)），`applied` 更是和状态机相关的，但保证这个关系会更安全。这可能会导致 apply 要多等一次写 `committed` 的 I/O 延迟，因为通常会把 raft 流程拆成两个 pipeline：raft + apply，`committed` 是在 raft 阶段写的，`applied` 是在 apply 阶段写的，就可能出现 `applied > committed` 的情况，所以要等 `committed` 更新后才能 apply。v4.0 之前的 TiKV 就有这个问题，后来实现了 [early apply](https://github.com/tikv/tikv/pull/6154)，通过在 apply 阶段也写 `committed` 来保证上述的关系。
- `applied <= persisted` 意味着只有已持久化的 raft log 才能被 apply 到状态机，因为 leader 和 follower 是并行持久化的，就有可能出现 `committed > persisted` 的情况。这个关系也不是 raft 论文要求的，raft-rs 要求保证这个关系来简化实现，而且同步的 ready 流程也必然满足这个关系。

### async-ready

raft-rs 里实现了 [async-ready](https://github.com/tikv/raft-rs/pull/403) 来提升 TiKV 的性能，它去掉了同步处理 ready 的限制，允许我们更好地利用异步 I/O，当然也引入了新的复杂度。async-ready 在获取 ready 后不需要同步持久化 hard state 和 raft logs，但要求能通过 [`Storage`](https://github.com/tikv/raft-rs/blob/a9d37b766d19d52d86e2cd7b407d8b366ce6c03e/src/storage.rs#L106) 获取到，调用 [`advance_append_async`](https://github.com/tikv/raft-rs/blob/master/src/raw_node.rs#L697) 后即可继续处理后面的消息和请求。当 ready 持久化完成后，需要按顺序调用 [`on_persist_ready`](https://github.com/tikv/raft-rs/blob/a9d37b766d19d52d86e2cd7b407d8b366ce6c03e/src/raw_node.rs#L617) 来更新 persisted 等状态，目前仍有 `applied <= persisted` 的限制。

### parallel raft

raft 是基于 Replicated State Machine 模型的，所以对 log 操作有顺序性要求，比如 log 要按顺序 append、commit 和 apply，[`PolarFS`](https://www.vldb.org/pvldb/vol11/p1849-cao.pdf) 实现了 parallel raft 降低了顺序性约束来提升性能。`PolarFS` 用的是 RDMA + SPDK 的纯异步实现，就 raft 这一层而言，顺序 append 限制了 I/O depth 也就限制了异步 I/O 的效果，也限制了用多连接来复制 raft log 的效果。乱序 commit 是为了乱序 apply 服务的，是和状态机相关的，需要能判断 raft log 间的依赖关系，只有无依赖的才能乱序。

parallel raft 是针对特定状态机实现的优化，可以用更多的资源来提升单个 raft group 的性能。async-ready 即使要求按顺序  [`on_persist_ready`](https://github.com/tikv/raft-rs/blob/a9d37b766d19d52d86e2cd7b407d8b366ce6c03e/src/raw_node.rs#L617) 和 [`advance_apply_to`](https://github.com/tikv/raft-rs/blob/a9d37b766d19d52d86e2cd7b407d8b366ce6c03e/src/raw_node.rs#L709)，它也能提供并发的 append 和 apply raft log 的能力，而且 async-ready 已经能满足 thread-per-core 模型的需求了。 

## Raft 实现

### [raft-rs](https://github.com/tikv/raft-rs)([etcd/raft](https://github.com/etcd-io/etcd/tree/main/raft))

只提供了 raft 算法实现，需要用户来实现基础组件和线程模型，见上面和[之前的博客](/raft/etcd-raft/)。

### [openraft](https://github.com/datafuselabs/openraft)

基于 tokio 的纯异步的 raft 实现，同样需要用户来实现 [`RaftNetwork`](https://github.com/datafuselabs/openraft/blob/db047b978e08fdb6f23bc4c27f94d4bd36ab925b/openraft/src/network.rs#L45) 和 [`RaftStorage`](https://github.com/datafuselabs/openraft/blob/db047b978e08fdb6f23bc4c27f94d4bd36ab925b/openraft/src/storage/mod.rs#L160)。openraft 比 raft-rs 易用一些，`RaftStorage` 包含了[状态机的接口](https://github.com/datafuselabs/openraft/blob/db047b978e08fdb6f23bc4c27f94d4bd36ab925b/openraft/src/storage/mod.rs#L229)，也不需要用户来选择线程模型，直接调用接口即可。它内部将部分可以并行的任务划分为 tokio task，通过配置 tokio 即可选择不同的线程模型，见 [Threads(tasks)](https://datafuselabs.github.io/openraft/threading.html#threadstasks)。

openraft 的异步只是实现上的异步，单个 raft group 的流程还是同步的，而且目前消息和请求是一个个处理的，没有攒 batch，apply 也没有拆成 tokio task，所以性能很一般。好像也不容易实现好 multi-raft，感觉它不好攒 batch。

### [dragonboat](https://github.com/lni/dragonboat)

开箱即用的 mult-raft 实现，内置了默认的网络层和存储层，默认的 log engine 是 [sharded pebble](https://github.com/lni/dragonboat/blob/8521aeb79dc1df80032b490bac60ff5ad7aa917f/internal/logdb/sharded.go#L34)。它内部分为多个 execution stage，每个 stage 由[可配置数量](https://github.com/lni/dragonboat/blob/8521aeb79dc1df80032b490bac60ff5ad7aa917f/config/config.go#L870)的 goroutine 来处理，每个 goroutine 一次会处理一批 raft group，每个 raft group 的流程是同步的。看起来是个非常不错的 multi-raft 实现，性能很好的样子。

### [scylladb](https://github.com/scylladb/scylla/tree/master/raft)

raft 在 scylladb 里使用地越来越广泛了。因为是基于 seastar 实现的，所以是 thread-per-core + 纯异步的，它内部有 io 和 apply 两个 fiber，分别用于持久化 raft log 和 apply 状态机。io fiber 虽然是串行的，但 raft 流程是异步的，即在 io 过程中 raft group 仍能处理消息和请求，而 openraft 是处理请求和消息、持久化 raft log、apply 都在一个 fiber 里。

## 总结

raft 协议虽然“简单易懂”，但要高性能的实现并不容易。对于 TPC raft store 而言，用 raft-rs 模仿 scylladb 的实现是个不错的选择，下一篇文章会介绍 raft engine 的实现。