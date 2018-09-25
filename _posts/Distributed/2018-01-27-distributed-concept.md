---
title: 分布式基本概念
layout: post
categories: Distributed
---

## CAP

## Consistency

## Strong consistency
Linearizability
  A history h of operations is linearizable if there is a linear order of the
  completed operations such that:
    - For every completed operation in h, the operation returns the same result in
      the execution as the operation would return if every operation was completed
      one by one in order h.
    - If an operation op1 completes (gets a response) before op2 begins (invokes),
      then op1 precedes op2 in h.

Sequential consistency
  Another popular definitions for strong consistency
    All Puts are in total order
    All Puts/Gets must be consistent with client-order
  Doesn't guarantee that Get sees result of Put if Put completed in real-time before Get

Time-line consistency
  Put()s in total order
  Get() can return the result of any Put()

Serializability is a correctness condition that is typically used for
systems that provide transactions; that is, systems that support
grouping multiple operations into an atomic operations.

Linearizability is typically used for systems without transactions.
When the Zookeeper paper refers to "serializable" in their definition
of linearizability, they just mean a serial order.


Wait-free
A wait-free implementation of
a concurrent data object is one that guarantees that any process can
complete any operation in a finite number of steps, regardless of the
execution speeds of the other processes. This definition was
introduced in the following paper by Herlihy:
https://cs.brown.edu/~mph/Herlihy91/p124-herlihy.pdf

Isolation

1. dirty read: one transaction reads another transaction’s uncommitted writes (a “dirty read”)
2. dirty write: what happens if the earlier write is part of a transaction that has not yet committed,
so the later write overwrites an uncommitted value? This is called a dirty write

3. Read Commited:
  1. When reading from the database, you will only see data that has been committed (no dirty reads).
  2. When writing to the database, you will only overwrite data that has been com‐ mitted (no dirty writes).
  3. 实现:
    * databases prevent dirty writes by using row-level locks: when a transaction wants to modify a particular object (row or document), it must first acquire a lock on that object
    * for every object that is written, the database remembers both the old committed value and the new value set by the transaction that currently holds the write lock.
      While the transaction is ongoing, any other transactions that read the object are simply given the old value.
      Only when the new value is committed do transactions switch over to reading the new value.

4. nonrepeatable read or read skew: she would see a different value ($600) than she saw in her previous query
  * Snapshot isolation: The idea is that each transaction reads from a consistent snapshot of the database—that is,
    the trans‐ action sees all the data that was committed in the database at the start of the transaction.
    Even if the data is subsequently changed by another transaction, each transaction sees only the old data from that particular point in time.

5. lost update: read-modify-write cycle(p 243)
6. phantom: where a write in one transaction changes the result of a search query in another transaction.
  * write skew: two transactions read the same objects, and then update some of those objects.
