---
title: Redis源码阅读计划
excerpt: 现在工作中主要做Redis相关的工作，包括Redis内核的开发、管理和运维。需要对Redis源码有很深入的理解，所以打算读下Redis源码，并在这里记录。
layout: post
categories: Redis
---

{% include toc %}

现在工作中主要做Redis相关的工作，包括Redis内核的开发、管理和运维。需要对Redis源码有很深入的理解，所以打算读下Redis源码，并在这里记录。

## Redis版本
Redis源码以[4.0.2](https://github.com/antirez/redis/releases/tag/4.0.2)版本为主，也会紧跟着[github](https://github.com/antirez/redis)上的`unstable`分支。

## 工具
阅读工具主要是Vim和Iterm2。

## 阅读计划
Redis主要有下面几种模式:  
  1. 单节点  
  2. 主从  
  3. 哨兵实现高可用  
  4. 集群  

会按照上面的顺序进行阅读。
