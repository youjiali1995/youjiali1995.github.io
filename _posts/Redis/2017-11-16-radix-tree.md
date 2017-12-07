---
title: Redis源码阅读(七) -- radix tree
excerpt: 介绍Redis内radix tree实现。
layout: post
categories: Redis
---

`Redis`内部的`radix tree`是为了实现高效的字符串到`satellite data`的映射，同时节省内存。`radix tree`可以说是`trie tree`的一种优化，
降低了内存使用。它的思想是将连续的只有单一子节点的节点压缩成一个节点，降低内存使用，但这带来了另外2个问题，插入和删除时的节点分割。
在数据比较稀疏或者重复数据很多时比较适合用`radix tree`而不是`hash table`。
