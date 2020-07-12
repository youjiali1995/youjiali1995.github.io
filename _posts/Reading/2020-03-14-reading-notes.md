---
title: Reading Notes
excerpt: 好久没读书和论文了，以后都会在这里记录，满分 10 ☆。
layout: post
categories: Reading
---

{% include toc %}

## 2020

### 07-12 | *Operating Systems: Three Easy Pieces* | 9☆

我读过很多操作系统相关的书，比如《操作系统概念》、《现代操作系统》、*CSAPP*(我倾向于把它也归为操作系统的书)、*TLPI*，这算是第五本。为什么会读那么多呢？因为操作系统实在是太重要了，只有更了解操作系统才能写出更好地代码、更快地排查问题，而且我们平时会遇到一些操作系统同样会遇到并已解决的问题，就可以从中吸取经验教训。另一个原因是常读常新，在不同的时间节点会有不同的收获。

*OSTEP* 最值得称赞的是它的叙述方式，不像很多书直接告诉你解决办法，而是先抽象问题、设定目标、一步步迭代并解决问题，可以让读者了解现代操作系统为什么需要这么做并学习解决问题的思路。如果非要说缺点的话，那就是缺乏「术」的介绍而且练习太少，对于在校学生来说需要搭配其他书籍或课程来学习，比如 *CSAPP*、MIT 6.828。

书虽好，但我也不会无缘无故就花时间在这上面，原因其实是后面准备开一系列和操作系统紧密相关的坑，希望这本书能作为大纲来指导路线，它也确实做到。

### 06-21 | *CockroachDB: The Resilient Geo-Distributed SQL Database* | 6☆

真的是巧，今年 TiDB 和 CRDB 都发了论文。不过友商的这篇论文只能打个6星（我们的论文也打不了高星😂），因为篇幅有限，还想要面面俱到就只能各部分浅显的介绍一点，不如他们官方文档和博客写的详细和有深度。强烈推荐他们的文档和博客，非常全面和易读，不管是从技术方面还是写作方面都值得学习。

### 05-12 | *Clock-SI: Snapshot Isolation for Partitioned Data Stores Using Loosely Synchronized Clocks* | 7☆

见 [Clock SI](/distributed/global-consistent-snapshot/#clock-si)。

### 04-06 | *Calvin: Fast Distributed Transactions for Partitioned Database Systems* | 5☆

这篇论文说实话没太看懂，给我的感觉是实现了非常特殊的状态机，状态机的输入是事务操作，所以只要用 paxos 之类的协议提交输入后事务状态就确定了，不需要走如 2PC 之类的 distributed commit protocol。

Calvin 是在单机存储之上实现的，数据分布在不同的 partition，每个 partition 是一个 replication group，用 paxos 之类的协议来复制。而 replica 在 Calvin 中指的是包含所有 partition 单个副本的多个节点，也就包含了所有数据。Calvin 中每个节点分为 3 层：

* Sequencer 接收客户端请求，对事务进行排序，把 transaction inputs 复制到 partition 副本。这层是分布式的，每个节点可以接收任意请求，不会按照 partition 来接收请求。不同 Sequencer 接收的请求无法排序，而且少量 transaction inputs 带来的复制代价比较大，所以 Calvin 采用的是 epoch 机制：每个 Sequencer 在 epoch 期间 batch 所有输入，再复制到 Sequencer partition 副本， 然后各 Sequencer partition 副本把 transaction inputs 下发给各 replica 内所有参与的 Scheduler，这样各 Scheduler 就可以收集到单个 epoch 来自所有 Sequencer 的输入，从而确定整个 epoch 输入的顺序（比如按照 Sequencer ID 来排序）。当然 Sequencer 提交了不一定事务就能成功提交，要看 Scheduler 执行的结果。
* Scheduler 使用确定的 lock 算法保证从 transaction inputs 并发执行也能得到确定的结果。因为是 RSM 所以要求确定的输入得到确定的结果，否则各个 replica 就不同了，这里复制的是 transaction inputs，而不是具体的操作，一个个串行执行当然可以得到确定的结果，但是性能很差，并发执行一般不能得到确定的结果，比如 serializable 级别的数据库能保证并发执行的事务是按照某种顺序串行执行的，但不会要求是哪种固定的顺序，也不要求每次都一样。Calvin 采用的是另一篇论文的 deterministic locking protocol，既能并发执行 transaction inputs 还能得到确定的结果。但为了确定性，Calvin 要提前知道事务所有的操作，所以不支持 Dependent transaction，也就是后面操作要基于前面结果的事务。那搞成 OCC 不就得了，客户端缓存所有的操作，事务提交时再发送给 Sequencer，Scheduler 采用的是 2PL 确定性算法，执行时要再检查 dependent read set 是否被修改了。
* Storage 没什么可说的，但是为了得到确定的结果，部分操作必须串行执行，所以卡在一个地方，剩下的都会卡住。Calvin 为了避免卡在 disk 上，会在 Sequencer 收到请求时预测需要的数据并预热，还会 delay 发送请求给 Scheduler。

论文的目标就是为了避免 2PC，因为在 2PC 过程中持有的锁不能释放，这会严重影响数据库的吞吐。采用 2PC 或者说锁不能执行完就释放的原因是不知道其他节点是否能成功执行，失败的原因有机器宕掉、违反事务隔离级别导致 abort 掉等。机器宕掉不应该导致 2PC 失败，搞成多副本高可用就能解决，但事务自己 abort 掉那就无法控制了，除非能提前知道其他节点执行的结果。现在的分布式数据库大都是按照 partition 来复制数据和执行请求，各个 partition 的执行结果互不知道，所以就需要 2PC，锁的存在时间也比较长：复制的耗时 + 2PC 耗时。Calvin 的思考角度不太一样，数据还是划分为 partition，也是按 partition 来复制，但复制的 transaction inputs 是面向所有 partition 的，所有 partition 的单个副本构成了状态机。使用确定性的 lock 算法保证了相同的输入能得到相同的状态，这样当 transaction inputs 提交后，每个 replica 就可以自己执行了，可以最小化锁的存在时间（没有复制和 2PC 的耗时），所有 replica 最终都会是相同的状态。

### 04-06 | *An Empirical Evaluation of In-Memory Multi-Version Concurrency Control* | 5☆

又是一篇在自研平台(Peloton)比较各种 MVCC 实现的论文，比较了 concurrency control、version storage、GC 和 index 的不同实现在不同 workload 下的性能，实现是基于纯内存数据库（越来越多的论文侧重于纯内存数据库了，毕竟硬件越来越牛逼），感觉还是什么都没说。。

现在大多数数据库都用了 MVCC，这部分的实现特别影响数据库的性能，尤其是 TiDB 这种基于 RocksDB 的 LSM-tree 的，scan 要做归并排序，如果版本特别多的话要跳过很多版本才能找到下一个有效数据，GC 来不及的话就会特别影响性能。TiKV 为了减少版本过多的影响，当 next 几次还没找到需要的数据时，会用 seek，很粗暴，效果也不好说。

### 04-06 | *An Evaluation of Distributed Concurrency Control* | 3☆

论文基于自研的平台比较了 6 种 serializable 级别的 distributed concurrency control 在典型 OLTP workload 场景下的性能，分别从读写比例、冲突程度、分布式事务涉及的 partition 数量、网络延迟等方面来比较，结论是都不太行，但是这几种协议都是自己实现的，肯定有很多可优化的地方，比较的结果不太准确。提出的解决办法有：1. 用 RDMA 之类的降低网络延迟，但对于跨数据中心场景没有办法；2. 更优的 data partition 策略，尽量将分布式事务变成单机事务；3. 使用更低的隔离级别。怎么说了和没说一样。。

### 03-15 | *Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases* | ?☆

HLC 算法很简单，就是在 physical clock 基础上使用了 logical clock 算法来保证 happened-before 关系。因为现在 TiDB 的悲观事务获取 timestamp 很频繁，我在想办法降低这部分的开销，所以更关心 HLC 如何用在 Snapshot Isolation 数据库上，这个论文里只简单提了一下，对我帮助不大，需要再研究下 cockroachdb 的事务。

### 03-15 | *Time, Clocks, and the Ordering of Events in a Distributed System* | ?☆

论文讨论了时间、时钟、事件顺序之间的关系。无法简单的用时间、时钟来确定分布式系统中事件的顺序，不同进程间的事件顺序要靠消息通信来确定，但也只能做到偏序或者说部分全序，因为不相交的事件不需要通信，也就无需确定顺序。若想要实现整个系统的全序对物理时钟有更严格的要求。

### 03-14 | *Database Internals* | 6☆

挺新的书，花了 2 周时间过了一遍，尴尬的是大部分我都懂，我不了解的看了也还是不了解。这本书更多的讲述的是「术」而不是「道」，介绍了很多问题的多种解决办法，相对而言还是更推荐 *DDIA*。不过这本书可以当 CMU15-445 的参考资料，第一部分几乎全覆盖了。
