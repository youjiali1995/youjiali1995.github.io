---
title: Redis源码阅读(七) -- radix tree
excerpt: 介绍Redis内radix tree实现。
layout: post
categories: Redis
---

`Redis`使用`radix tree`保存 `slot` 到 `key` 的映射，会在每个 `key` 加上2字节的前缀: 以 `little-endian` 存储的 `slot`，然后插入到 `radix tree` 中。
`radix tree`可以说是`trie tree`的一种优化，降低了内存使用。它的思想是将连续的只有单一子节点的节点压缩成一个节点，降低内存使用，
但这带来了另外2个问题，插入和删除时的节点分割。在数据比较稀疏或者重复数据很多时比较适合用`radix tree`而不是`hash table`。因为 `Redis` 的 `slot` 共有 `16384` 个，
前缀 `slot` 带来的影响可以忽略不计，同时如果大量 `key` 有重复，可以显著降低内存使用。
