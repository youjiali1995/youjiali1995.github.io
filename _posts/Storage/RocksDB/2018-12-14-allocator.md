---
title: RocksDB 源码分析 -- Allocator
layout: post
excerpt: 介绍 Allocator 实现。
categories: RocksDB
---

{% include toc %}

`RocksDB` 内部实现了简单的 `Allocator`，在一些地方会用来分配内存，如 `InlineSkipList` 的 `Node`。

## Arena
`Arena` 的实现很简单，先分配 `block`，然后从 `block` 中分配，支持对齐分配和对齐无要求的分配，为了减少对齐导致的内存浪费，会从 `block` 的两端分别分配。

`Arena` 还会用 `mmap(MAP_HUGETLB)` 分配 `huge page`，可以降低 `TLB` 内存使用，减少 `TLB miss`，见
[Allocating Some Indexes and Bloom Filters using Huge Page TLB](https://github.com/facebook/rocksdb/wiki/Allocating-Some-Indexes-and-Bloom-Filters-using-Huge-Page-TLB)。

## ConcurrentArena

