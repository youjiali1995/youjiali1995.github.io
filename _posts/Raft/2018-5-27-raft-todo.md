---
title: Raft 笔记(八) -- TODO
excerpt: Raft 笔记的未来计划。
layout: post
categories: Raft
---

{% include toc %}

花了差不多一个月时间又重新学习了 `Raft`，并看了 `etcd/raft` 的实现，`Raft` 看上去很简单，但是有很多的 `corner case`，想要实现稳定的、性能高的并不容易。
`Raft` 笔记先告一段落，但仍有许多工作要做。

## etcd
`etcd/raft` 只实现了 `Raft`，其余的如 `Storage` 和 `RPC` 都由用户处理，可以学习下 `etcd` 是如何实现和使用 `etcd/raft` 的。

## Lease holder
[cockroach](https://github.com/cockroachdb/cockroach) 实现了 [range lease](https://github.com/cockroachdb/cockroach/blob/master/docs/RFCS/20160210_range_leases.md)，
可以直接读 `lease holder` 就能保证线性一致性，不需要 `ReadIndex` 的 `heartbeat`，也不会像 `lease read` 损失一致性，
`PingCAP` 应该也实现了类似的机制: [通过 raft 的 leader lease 来解决集群脑裂时的 stale read 问题](https://pingcap.com/blog-cn/stale-read/)。

`lease holder` 是 `Raft-agnostic` 的，没找到很详细的资料，好像是受这篇论文启发的：
[Paxos Quorum Leases: Fast Reads Without Sacrificing Writes](http://www.cs.cmu.edu/afs/cs/Web/People/dga/papers/leases-socc2014.pdf)

## Quorum read
`Raft` 的读虽然可以发送给 `follower`，但还是要从 `leader` 获取 `readIndex`，`leader` 的压力会很大。使用 `quorum read` 可以利用 `follower` 读，减小 `leader` 的压力，
提高读的吞吐量和性能: [Improving Read Scalability in Raft-like consensus protocols](https://www.usenix.org/system/files/conference/hotcloud17/hotcloud17-paper-arora.pdf)

## Multi-raft
一般会将数据分片(sharding)，每个分片都由一个 `raft group` 管理。每台机器上会有多个 `raft group`，若每个 `raft group` 都单独的进行 `RPC`，流量会很大，`multi-raft`
将多个 `raft group` 的消息和心跳合并，减轻网络的压力。多个 `raft group` 逻辑上独立，物理上共用一些东西。
