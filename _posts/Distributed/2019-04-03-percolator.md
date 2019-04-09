---
title: Percolator
excerpt: Large-scale Incremental Processing Using Distributed Transactions and Notifications
layout: post
categories: Distributed
mathjax: true
---

{% include toc %}

*Percolator* 是在 *Bigtable* 的单行事务的基础上实现的支持跨行、*Snapshot Isolation* 的分布式事务。原子性仍是由 *2PC* 保证，不过事务的所有状态都保存在 *Bigtable* 中，由 *Bigtable* 提供高可用和一致性，
客户端只实现事务的逻辑，所以任何客户端都可以认为是 *Coordinator*，解决了传统 *2PC* 中 *Coordinator* 挂掉可能需要人为 *rollback/roll forward* 的问题。

## Bigtable
*Bigtable* 通过 $$ \{row, column, timestamp\} $$ 确定一个 *cell*：
> Each cell in a Bigtable can contain multiple versions of the same data; these versions are indexed by timestamp.

这里的 *timestamp* 只是 *key* 的一部分，不意味 *Bigtable* 支持 *SI*，*Bigtable* 提供的 *API* 支持操作范围 *timestamp* 的 *key*。
*Bigtable* 中每个 *key* 确实有多个 *version*，但只是为了减少冲突来提高性能：
> The only mutable data structure that is accessed by both reads and writes is the memtable.
> To reduce contention during reads of the memtable, we make each memtable row copy-on-write and allow reads and writes to proceed in parallel.

*Bigtable* 支持的是单行事务，对单行事务的操作是原子的，也就是单行的 *Serializable*：
> Every read or write of data under a single row key is atomic (regardless of the number of different columns being read or written in the row).  
> Bigtable supports single-row transactions, which can be used to perform atomic read-modify-write sequences on data stored under a single row key.

## Snapshot Isolation
*SI* 一般是通过 *MVCC* 实现的，但是对 *SI* 的定义和要提供的功能不同，实现方式也不同：
* 一些认为 *SI* 只需要提供 *Snapshot* 即可，即 *RR*；一些认为还需要解决 *Lost Update*。
* 一些只提供当时的 *Snapshot*；一些还要提供过去的 *Snapshot*。

通常的实现方式有2种：
1. 给每个事务分配一个单调递增的 $$ txn\_id $$，每个数据由 $$ \{create\_txn\_id, delete\_txn\_id\} $$ 标记，更新操作为 *delete + create*。事务只能看到最新的且未删除的 $$ create\_txn\_id < txn\_id $$ 的数据。
2. 每个事务开始时分配 $$ start\_ts $$，结束时分配 $$ commit\_ts $$，数据由 $$ commit\_ts $$ 标记。事务只能看到最新的 $$ commit\_ts < start\_ts $$ 的数据。

为了保证 *RR*，第一种方式要忽略所有 $$ txn\_id $$ 小于自己但在自己开始之后才提交的数据，但这也意味着用相同的 $$ txn\_id $$ 可能会读到不同的 *Snapshot*；
第二种方式一般会提供过去的 *Snapshot*，要保证事务提交时分配的 $$ commit\_ts $$ 比所有的 $$ start\_ts \&\& commit\_ts $$ 都大且持续到事务提交完成，即提交相当于是原子的。

要解决 *Lost Update* 会在提交时检查是否有其他的事务更新的相同的数据，虽然粒度比较大(更新不意味着一定发生 *Lost Update*)。

## Percolator
*Percolator* 采取的是第二种方式来实现 *SI*，*timestamp* 从 *TSO* 获取，且会解决 *Lost Update*，采用乐观锁，只有提交时才会检查冲突。
![image](/assets/images/percolator/si.png)

*Percolator* 每行数据使用3个 *Column*：

| Column | Use |
| ------ | ---- |
| lock | An uncommitted transaction is writing this cell; contains the location of primary lock |
| write | Committed data present; stores the Bigtable timestamp of the data |
| data | Stores the data itself |

* *lock* 即锁表，为了解决 *write-write* 冲突，即可能发生的 *Lost Update*。
* *write* 保存最新已提交数据的 $$ start\_ts $$。
* *data* 保存数据。

### Write
*Percolator* 累积所有的写操作，提交时再检查冲突，可以减少 *RPC* 的次数:
![image](/assets/images/percolator/write.png){:width="60%"}

每个写事务会有一个 *Primary Key* 作为 *commit point*，只要 *Primary Key* 成功提交，整个事务就是提交成功的。若是在提交过程中出错，其他客户端可以查看 *Primary Key* 的提交状态，判断是要 *rollback* 或是 *roll forward*。
提交分2步：
1. 对所有写的行 *Prewrite*：
    1. 检查写写冲突：32行检查是否有其他事务在 $$ \{start\_ts, ∞\} $$ 提交了新的写，防止 *Lost Update*；34行检查写写冲突，可能有正在提交的事务。
    2. 加锁：*lock* 列中记录 *Primary Key*；*data* 列写入数据。
2. *Commit* 所有行 ：
    1. 先提交 *Primary Key*：*write* 列保存 $$ commit\_ts \Rightarrow start\_ts $$ 的映射，其他事务只能读到最大的 $$ commit\_ts < start\_ts $$ 的数据；删除 *lock* 解锁。
    2. 再提交 *Secondary Key*。

### Read
![image](/assets/images/percolator/get.png){:width="60%"}

*read* 操作比较简单，从 *write* 列中找到最新的 $$ commit\_ts < start\_ts $$ 的 *key*，再从 *data* 列中读。需要注意的是12行，要检查读的 *key* 是否有锁，这可能和一些人对 *MVCC* 的认知有冲突，一般会认为
*MVCC* 没有读写、写读冲突，但在上面 *Snapshot Isolation* 中提到提交成功时要保证 $$ commit\_ts $$ 是最大的，即提交是原子的，在 *Percolator* 中提交无法做到原子性，所以为了保证 *SI*，读也要检查写锁。

### Clean Up
提交时出错会遗留下锁，这里是通过超时来判断是否要清理，也就是14行 *BackoffAndMaybeCleanupLock()* 要做的，可以有多种实现方式，论文中没有具体给出:
1. *Bigtable* 提供的 *API* 能返回一定 *timestamp* 范围内的 *key/value*，所以12行可以构造出 $$ \{primary\_key+lock+start\_ts\}$$，先查找该 *key* 是否存在，若存在则删掉进行 *rollback*，所以在 *commit* 阶段(53行)
要再次判断 *Primary Lock* 是否存在，数据竞争通过 *Bigtable* 的行级事务避免。
2. 若不存在，则需要遍历 *Primary Key* 的 *write* 列查找是否有 $$ \{primary\_key+write+commit\_ts \Rightarrow start\_ts\} $$，存在的话则找到了 $$ commit\_ts $$ 进行 *roll forward*，没找到 *rollback*。这也是 *write* 列
要保存 $$ start\_ts $$ 的原因。

*rollback* 就是删掉的 *lock + data* 列，*roll forward* 就是写 *write* 列删 *lock* 列，实现上也可以有多种选择：
* 惰性处理：只处理访问到的 *key*，但要注意34行未进行 *clean up*。
* 遍历找到所有的 $$ \{secondary\_key+lock+start\_ts \Rightarrow primary\_key\} $$ 处理。

惰性处理不需要遍历 *lock* 列查找 *key*，但每遇到一个 *lock* 就要遍历查找 *Primary Key* 的 *write* 列。第二种方式可能单次的处理时间长些，但一劳永逸。要注意的是清理不能只靠客户端触发，应该有定时任务之类的来
清理。

## Summary
*Percolator* 利用了 *Bigtable* 的特性，使得实现上很简单，不需要考虑很多的数据竞争，但是 *RPC* 和写入的代价很高，想要高性能的实现并不容易。因为分布式和乐观锁的原因，在写入冲突较多的场景下，*abort* 会很频繁，
因为这里没有全局的 *transaction manager*，所以不能等待，可能会发生死锁。
