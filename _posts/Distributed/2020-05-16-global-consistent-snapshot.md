---
title: Global Consistent Snapshot
excerpt: 介绍在分布式数据库中如何得到  Global Consistent Snapshot。
layout: post
categories: Distributed

---

{% include toc %}

Snapshot Isolation 相信大家都不陌生，几乎所有主流的数据库都实现了这一隔离级别，但今儿不聊 SI 的事儿！就聊 snapshot！现在大多数数据库都采用了 MVCC，所以只需要获取一个版本号就能得到 snapshot。snapshot 在数据库中非常有用，可以用于实现多种隔离级别，比如 SI 就是在事务开始时获取一个当时的 snapshot，而 Read Committed 可以认为是每个语句开始时获取一个当时的 snapshot（RC 不要求提供 snapshot，但大多数数据库提供的都是 consistent read），但在分布式数据库中获取合适的版本号却不是那么容易。

## Consistency

Isolation 只要求多个事务执行的结果和以某种串行顺序执行的结果相同，但是对顺序没有任何约束，也就是说可以任意排序，比如后开始的事务可能看不到前面已提交事务的结果，因为被重排序到前面了。当然这非常反人类，没有数据库会这么做，都会按照 real-time 施加约束，保证后开始的事务一定能看到之前提交事务的结果，这在单机数据库中很容易实现，但在分布式数据库中并不容易。

为了给顺序施加约束，就引入了 consistency，比如 linearizability，serializability + linearizability 就构成了 strict serializability。spanner 论文里提到了另一个词 external consistency，对于数据库而言，这等价于 strict serializability，这个词提供了一个很好的角度来观测分布式数据库的行为：如果有一个外部的上帝视角知道所有事务的先后顺序，并且数据库执行的顺序与该顺序相同，那就是满足 external consistency 的。

约束越强，性能越差，而且 linearizability 不是那么好实现的，所以就有了更弱的 consistency。对于 SI 而言，除了有要读最新 snapshot 的 conventional SI 外，还有允许从任意旧 snapshot 读的 generalized SI、要读到进程自己提交过事务的 prefix-consistent SI 等。

## Timestamp

snapshot 一般都是用 MVCC 实现的：

* 事务在开始时获取 TS<sub>start</sub>，提交时获取 TS<sub>commit</sub>。
* 数据版本用 TS<sub>commit</sub> 标记，事务读取 TS<sub>start</sub> 之前最新的数据。
* TS 保证单调递增。

TS 单调递增使得后开始事务的 TS<sub>start</sub> 一定大于已提交事务的 TS<sub>commit</sub>，同时事务的 TS<sub>commit</sub> 单调递增，事务的执行顺序和 TS<sub>commit</sub> 顺序相同，从而保证了外部一致性。

为什么不用 TS<sub>start</sub> 来提交数据呢？因为要保证 snapshot 的有效性，即读取了某一版本（TS<sub>0</sub>）的数据后，不允许有新的已提交数据的 TS<sub>data</sub> ∈ [Ts<sub>0</sub>, TS<sub>start</sub>)，否则就不满足可重复读了。除非采用类似 Timestamp Ordering Concurrency Control 的办法：

* 数据由 TS<sub>start</sub> 标记，同时记录 TS<sub>maxRead</sub>，即读取到该版本数据的事务的最大的 TS<sub>start</sub>。
* 不允许 TS<sub>start</sub> 小于 TS<sub>maxRead</sub> 的事务提交，从而保证了 snapshot 的有效性。

但这办法带来的问题是读也会造成写（需要修改 TS<sub>maxRead</sub>），所以更常用的还是获取 2 次 TS。

## True Time

因为时钟偏移问题，在分布式场景中想要实现单调递增的 TS 比较麻烦，一般需要有中心化的 TSO，这带来的问题是取 TS 消耗过大，尤其在跨数据中心场景下。Spanner 使用 True Time 解决了该问题并能保证外部一致性。

Spanner 的数据也是多版本由 TS<sub>commit</sub> 标记，但读写事务采用 S2PL，所以不需要获取 TS<sub>start</sub> 就能读取最新的数据（S2PL 读要加锁），事务的执行顺序由释放锁的瞬间决定，也就是由 TS<sub>commit</sub> 决定，所以论文里说只要保证后开始事务的 TS<sub>commit</sub> 大于之前事务的 TS<sub>commit</sub> 就能保证外部一致性：

>  if the start of a transaction T2 occurs after the commit of a transaction T1, then the commit timestamp of T2 must be greater than the commit timestamp of T1.

使用 True Time 能获取当前时钟的误差范围：TT.now() ∈ [earliest, latest]，Spanner 取 TT.now().latest 作为 TS<sub>commit</sub> 并等到 TT.after(TS<sub>commit</sub>) 时才提交数据（Commit Wait），从而保证了上面的不变量。

Spanner 使用 S2PL 又用 MVCC 的原因是为了让只读事务不需要加锁，只需要获取 TS<sub>read</sub> 并读取之前版本数据即可，但 TS<sub>read</sub> 该如何确定并能保证外部一致性呢？直接取 TT.now() 作为 TS<sub>read</sub> 肯定能读到之前提交完成事务的数据，但是有可能后执行的只读事务的 TS<sub>read</sub> 比先执行的小，可能会破坏外部一致性。其实对于只读事务来说 TS<sub>read</sub> 就相当于是它的 TS<sub>commit</sub>，也要保证单调递增，也要比之前所有的 TS<sub>read</sub> 都大，所以最简单的做法也是取 TT.now().latest，但也有可能需要等待一段时间。

Spanner 的只读事务比较复杂，而且有很多值得借鉴的地方，值得再详细写一下。为了实现 snapshot 读，只读事务可能会被阻塞，比如：

1. 有正在提交的事务，且可能该事务的 TS<sub>commit</sub> 小于 TS<sub>read</sub>。
2. spanner 允许读 replica，要保证从 replica 读也能保证外部一致性。

对于第一种情况，事务提交时各 paxos leader 会记录当时的时间戳 TS<sub>prepare</sub>，事务最终的 TS<sub>commit</sub> 一定大于所有参与者的 TS<sub>prepare</sub>，所以 TS<sub>prepare</sub> 是 TS<sub>commit</sub> 的下界，当 TS<sub>read</sub> 大于 TS<sub>prepare</sub> 时就要阻塞直到事务提交。

对于第二种情况，paxos leader 要同步 TS<sub>prepare</sub> 给 replica，replica 也要有办法检查当前副本是不是足够新。因为 paxos group 是按顺序 apply 数据，所以如果 replica 最新的 TS<sub>apply</sub> 大于 TS<sub>read</sub> 就可以提供读服务。为了避免没有写入导致的读阻塞，spanner 里 leader 会定期更新 TS<sub>apply</sub>，follower 也可以按需发请求给 leader 来更新 TS<sub>apply</sub>，从而保证之后的 TS<sub>commit</sub> 都比 TS<sub>read</sub> 大。（这里的 TS<sub>apply</sub> 类似 raft readIndex 里的 commitIndex，区别是从事务角度不需要向 leader 发请求就能知道是否满足一致性）

为了缓解读阻塞的问题，可以从两方面考虑：

1. 阻塞的颗粒度：上面维护的 TS<sub>prepare</sub> 和 TS<sub>apply</sub> 可以按照 group 级别、 key range 甚至 key 级别。
2. TS<sub>read</sub> 尽可能小：对于知道要读哪些 partition 的读操作而言，可以从各 group leader 获取 TS<sub>apply</sub>，然后选择其中最大的作为 TS<sub>read</sub>，这能保证读到最新的数据。



> **注意**：
>
> 不能直接从各 paxos group 直接读最新的数据，各 partition 最新的 snapshot 构不成 global snapshot，因为有可能你读了一个 partition 最新 snapshot 后，一个事务更新了该 partition 和另一个 parition，这时候再读另一个 partition 就会读到事务的部分修改，破坏了原子性。需要从各 partition 协商出 TS 才能读。

## Clock SI

外部一致性当然很好，但实现它有很多代价，比如中心化的 TSO，比如特殊的硬件设备原子钟。如果牺牲掉部分一致性，或许能得到更通用的解法，更高的性能。业界有很多这方面的探索，比如 Clock SI。

Clock SI 是去中心化的算法，各 parition 自行维护 TS<sub>local</sub>，算法也非常简单：

1. 事务由 TS<sub>start</sub> 和 TS<sub>commit</sub> 标记，TS<sub>start</sub> 从本地取。
2. 提交时检查 write conflict，使用 2PC 保证原子性。prepare 会获取各 partition 的 TS<sub>local</sub>，选其中最大的作为 TS<sub>commit</sub>。
3. 读操作除了被正在提交的事务阻塞外，如果 TS<sub>start</sub> 比要读的 partition 的 TS<sub>local</sub> 大的话，也会被阻塞。

第2点保证了新事务的 TS<sub>commit</sub> 一定大于之前读事务的 TS<sub>start</sub>。第3点的原因是可能有新事务的 TS<sub>commit</sub> 比当前读事务的 TS<sub>start</sub> 小，所以要等到 partition 的 TS<sub>local</sub> 超过它才行，否则会破坏 snapshot。Clock SI 可以很好的处理其他 partition 时钟滞后的情况，但如果本地时钟滞后会发生什么呢？比如：

1. 本地取的 TS<sub>start</sub> 小于要读的 partition 的 TS<sub>local</sub>，可能读不到之前已提交的数据。
2. 事务提交时从其他 partition 那选了最大的 TS<sub>commit</sub>，之后本地再读时读不到，也无法修改（write conflict）。

所以 Clock SI 并不能提供最新的 snapshot，它实现的是 generalized SI。或者由客户端维护之前事务的 TS<sub>latest</sub>，并传给之后的事务，来实现 session consistency。

## Logical Clock

Clock SI 有很多可以优化的地方，比如当事务的 TS<sub>start</sub> 大于 partition 本地时钟的话，使用 TS<sub>start</sub> 更新该 partition 本地时钟的话是不是就不用阻塞了？当事务的 TS<sub>commit</sub> 大于 partiton 本地时钟的话，使用 TS<sub>commit</sub> 更新该 partition 本地时钟的话是不是之后事务就能读到最新数据也能及时修改了呢？这其实就是 logical clock 的想法。

有时候我们并不需要对所有事务排序，只需要对相交事务排序即可，比如修改了同一行的不同事务的 TS<sub>commit</sub> 要单调递增，logical clock 就可以实现这一点。它的实现方式一般是各节点自己维护 clock，当有事件发生时增加 clock，当收到其他节点的事件时要更新 clock。

### CockroachDB

CRDB 的事务非常值得学习，它使用 Hybrid Logical Clock 来实现去中心化的分布式事务，HLC 在 LC 基础上增加了 physical 部分，使得 clock 更有实际意义，并且不会无限增长。

CRDB 各节点维护 HLC，在有事务请求时会更新 HLC，数据同样使用 MVCC，由 TS<sub>commit</sub> 标记。事务开始时在本地获取临时的 TS<sub>commit</sub>，在执行过程中可能因为某些原因被往前推，比如：

* 要修改的 key 被其他 TS 更大的事务读过了，则当前事务的 TS<sub>commit</sub> 一定要大于 TS<sub>maxRead</sub>。
* 要修改的 key 有更新的版本，则当前事务的 TS<sub>commit</sub> 一定要大于最新的版本。
* 写下的 write intent 被其他 TS 更大且优先级更高的事务读到了，为了避免阻塞，会往前推写事务的 TS<sub>commit</sub>。

CRDB 实现的是 SSI，要求事务等价于在某一点瞬间执行完成，但事务执行总有过程，比如从 TS<sub>start</sub> 到 TS<sub>commit</sub>，CRDB 在提交时会检查之前读的数据在 [TS<sub>start</sub>, TS<sub>commit</sub>) 间是否有修改，没有的话说明把 TS<sub>start</sub> 提升到 TS<sub>commit</sub> 也会得到相同的结果，所以可以认为事务是在 TS<sub>commit</sub> 这一点执行的，满足 serializable。所以 TS<sub>commit</sub> 被往前推的话可能导致事务提交失败，需要重启。

CRDB 中最有趣的是如何保证读一定能读到之前已提交的数据。事务开始时获取的临时 TS<sub>commit</sub> 其实是期望它能等价于在这一点执行，所以也会使用它来读数据，但因为时钟偏移的问题，有可能之前已提交数据的版本更新，所以 CRDB 维护了不确定区间 [TS<sub>start</sub>, TS<sub>start</sub> + maxClockOffset)，只要遇到在这区间内的数据都认为是要读到的。一旦读到了不确定区间的数据，就需要检查之前事务执行的结果是否还有效（检查方式和提交时相同），无效的话就需要使用新的 TS<sub>start</sub> 重启事务，但不确定区间并不会随着重启而改变，因为事务启动的时间并没变化的。CRDB 还有优化来缩小不确定区间的范围，比如事务维护访问过节点的 HLC，版本号在这之后的数据一定不是在事务启动之前提交的，就可以缩小访问各节点时的不确定区间，同时每个节点最多只会重启一次（使用节点当时的 HLC 重启即可）。CRDB 的做法相当于是在读的时候**有可能**等一会，而 spanner 是提交时一定等一会。

从上面写的来看，好像 HLC 不是特别必要，但 HLC 解决了之前提到的 Clock SI 的问题，减少了事务无谓的阻塞和重试。

### Casual Reverse

CRDB 并不能实现 external consistency，考虑如下场景：

1. Txn<sub>1</sub> 读 row<sub>1</sub> 和 row<sub>2</sub>，同时
2. Txn<sub>2</sub> 修改 row<sub>1</sub>，完成后
3. Txn<sub>3</sub> 修改 row<sub>2</sub>。

如果满足 external consistency 的话，则顺序一定是 123, 213, 231 其中之一，但 CRDB 可能出现 312：

1. row<sub>1</sub>, row<sub>2</sub> 在不同节点上。
2. Txn<sub>1</sub> 读完 row<sub>1</sub> 所在节点时，Txn<sub>2</sub> 还未写 write intent。
3. Txn<sub>1</sub> 读 row<sub>2</sub> 时，Txn<sub>3</sub> 已提交，且 Txn<sub>3</sub>'s TS<sub>commit</sub> < Txn<sub>1</sub>'s TS<sub>commit</sub> < Txn<sub>2</sub>'s TS<sub>commit</sub>。

这种异常出现条件非常苛刻，而且这种 casual reverse 影响又有多大呢？

## 总结

在分布式系统中为了满足一致性和隔离性，等一会是非常重要且实用的做法。
