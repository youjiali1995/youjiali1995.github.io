---
title: 写一个 Rust TPC Runtime(零) -- Preview
layout: post
categories: Rust
published: false
---

计划从零写一个 rust TPC async runtime，原因有这几个：一是我看过几个但没自己写过，总觉得差点意思；二是它对系统至关重要，最好能自己掌控和定制；三是要支持 `epoll`、AIO 和 io_uring，目前的 rust runtime 要么只支持 `epoll` 没有 async disk I/O，要么只支持 io_uring，对 kernel 要求较高。

这个系列不会再介绍 rust runtime 的[基本概念](/rust/async)，大概会分为这几个部分来介绍：

- Task：如何设计和实现高性能的 Task。
- Reactor：如何设计接口来同时支持 readiness-based(`epoll`、AIO poll) 和 completion-based(AIO, io_uring)，以及这些 backend 的使用。
- I/O trait：如何设计对 readiness-based 和 completion-based 友好的 I/O trait。
- 可选：timer、NUMA friendly、CPU scheduler 等。
