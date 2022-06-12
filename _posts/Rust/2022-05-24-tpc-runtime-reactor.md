---
title: 写一个 Rust TPC Runtime(二) -- Reactor
layout: post
excerpt: 介绍 Reactor 的设计和实现。
categories: Rust
---

{% include toc %}

reactor 要做哪些事情呢？笼统点说所有不能立即完成的任务都需要 reactor 的参与，比如 I/O 和定时器。reactor 需要在事件就绪或者完成时唤醒对应的 task。

## backend

Linux 下的 reactor backend 目前有 3 种：

- `epoll`：支持除文件 fd 外（socket、eventfd、timerfd、pipe 等）事件的就绪与否，是 readiness-based。
- AIO：在 `epoll` 基础上增加了 disk I/O 的完成与否，是 readiness-based 和 completion-based。
- io_uring：在 AIO 基础上支持了大量系统调用的异步，是 completion-based。

readiness-based 和 completion-based 其实没啥区别，把等待事件就绪看作是事件完成就统一起来了，区别主要在于它们和 reactor backend 交互的资源不同：readiness-based 只有 fd，而 completion-based 除 fd 外通常还会有 buffer，因为要读写。reactor backend 执行异步操作需要用到用户态程序传进来的资源，这就带来了一个问题：如何保证这些资源是安全的？ 

- fd：fd 就是个进程级别的内核数据结构的索引（数组下标），重复 open、close 会得到相同的值。假如 reactor backend 已经提交了异步操作，而 fd 被 close 了，甚至又 open 了一个相同值的 fd 会发生什么呢？不同的 reactor backend 行为不同：
  - `epoll`：fd 被 close 就会从 `epoll` 兴趣列表里移除，保证了安全。即使没有这种保证，因为只是返回可读可写，也不会对应用程序有什么影响，而且 `epoll` 还可能有虚假唤醒，本身就有这种问题。
  - AIO/io_uring：不清楚这俩是什么行为，如果自身没有保证安全性的话，可能会出现数据错误，因为读写了非预期的 fd。
- buffer：必须要保证异步操作未完成时 buffer 是有效的，否则就内存不安全了。

支持 completion-based 的难点就在于资源管理。Rust 的 future 是 cancelable 的，drop 了就行，而 drop 一个被 poll 过的 future 就可能有上面的问题，因为传给 reactor backend 的资源随着 drop future 被释放了，reactor backend 也不支持立即生效的 cancel 操作。那什么时候会 drop 被 poll 过且未完成的 future 呢？可能是这个 runtime 支持 cancel future 功能，或者这个 future 被包在了另一个 future 里，且该 future 完成了，比如用 [`select!`](https://docs.rs/futures/0.3.6/futures/macro.select.html) 。Rust 针对这个问题有过很多讨论，如 [Async/Await - The challenges besides syntax - Cancellation](https://gist.github.com/Matthias247/ffc0f189742abf6aa41a226fe07398a8)、[Notes on io-uring](https://without.boats/blog/io-uring/)，目前 Rust 的类型系统无法实现 reactor 异步操作未完成时禁止 drop 对应的 future，所以对于 completion-based reactor 大都选择 reactor 也持有资源，由 reactor 来保证即使外层的 future 被 drop 了资源仍有效，比如把资源的 ownership 从 future 转移给 reactor，future 完成后再转移回来；资源用引用计数来追踪。

> Logical ownership is the only way to make this work in Rust’s current type system: **the kernel must own the buffer.** There’s no sound way to take a borrowed slice, pass it to the kernel, and wait for the kernel to finish its IO on it, guaranteeing that the concurrently running user program will not access the buffer in an unsynchronized way. Rust’s type system has no way to model the behavior of the kernel *except* passing ownership. I would strongly encourage everyone to move to an ownership based model, because I am very confident it is the only sound way to create an API.

### reactor trait

reactor 要能支持各种系统调用的异步，一种实现方式是提供类似 io_uring 的接口，毕竟现在只有它支持丰富的系统调用异步，也就是有 sqe 能代表各种系统调用，cqe 代表各种系统调用的结果。reactor backend 在自己的类型和统一的类型间转换，如 `epoll` 的 `epoll_event`、AIO的 `iocb` 和 `io_event`。对于不支持异步的系统调用，可以扔到单独的线程来执行。

```rust
trait Reactor {
    fn kernel_submit_work(&mut self) -> usize;
    fn reap_kernel_completions(&mut self) -> usize;
    fn enqueue(&mut self, sqe: SubmissionQueueEntry) -> CompletionQueueEntry;
}
```

这种 trait 比较裸，还可以给常用的接口如 `readable`、`writeable`、`read_some` 和 `write_some` 单独提供方法，其他的系统调用才用上面的接口。但我没有用上面的实现🤣，现在的实现类似 [`seastar`](https://github.com/scylladb/seastar/blob/7a039e7dc8e2d9407603d9b9030423442ea3ce10/src/core/reactor_backend.hh#L168-L215)，因为我先实现的 AIO backend，它只支持少量的系统调用，就给这几个常用接口单独实现了，等后面增加更多系统调用时再实现类似上面的接口吧。

```rust
trait Reactor {
    fn kernel_submit_work(&mut self) -> usize;
    fn reap_kernel_completions(&mut self) -> usize;
    fn readable(&mut self, fd: RawFd) -> Source;
    fn writable(&mut self, fd: RawFd) -> Source;
    fn readable_or_writeable(&mut self, fd: RawFd) -> Source;
    fn read_some(&mut self, fd: RawFd, buf: DmaBuffer, offset: usize) -> Source;
    fn write_some(&mut self, fd: RawFd, buf: DmaBuffer, offset: usize) -> Source;
}
```

### `Source`

在上面的 trait 中看到了 `Source` 这东西，也就是前面说的 cqe，这是用来和上层应用交互的，它实现了 `Future` trait 来获取系统调用的结果。`Source` 需要在 reactor 和上层应用间共享数据，reactor 通过它来设置 result 并唤醒 future，而上层用它来获取 result。在 Rust 里要共享当然是用 `Rc`/`Arc` 啦，我这里的实现共享的是 [`Slab`](https://docs.rs/slab/latest/slab/)，因为在 Rust 里用 raw pointer 作为 `epoll` 等的 user data 不太安全，所以 user data 是 `Slab` 里的索引，reactor 和 `Source` 通过共享 `Slab` 找到对应的数据来通信，这也避免了每次异步操作都有内存分配。所以 `Source` 的结构如下：

```rust
#[derive(Clone)]
struct Pool(Rc<RefCell<Slab<Inner>>>);

struct Source {
    key: usize,
    pool: Pool,
}

struct Inner {
    fd: RawFd,
    tp: Type,
    state: State,
    waker: Option<Waker>,
    result: Option<std::io::Result<usize>>,
}
```

- `Type` 类似前面说的 sqe，表明了系统调用的类型且**拥有**资源。`Type` 和 `State` 用于保证上面说的 completion-based 的资源有效性：`Type` 拥有资源且只有当异步操作不在执行时 drop `Source` 才会从 `Pool` 中移除，否则会在异步操作完成时由 reactor 移除。
- 观察一下 `epoll`、AIO 和 io_uring 用于获取结果的结构体会发现，所有系统调用的结果对应到 Rust 的类型就是 `std::io::Result<usize>`。

现在的实现有两个未解决的问题：

1. 不适用于 `epoll`！`epoll` 和 AIO、io_uring 相比有两个特殊的点：一是可以不 oneshot；二是一次能返回两个结果（readable 和 writable）。像 AIO 这种对相同 fd 先后调用 poll read 和 poll write 会返回两次结果，而 `epoll` 第二次需要用 `EPOLL_CTL_MOD` 且只会返回一次，但 `epoll_event` 里会有两个就绪事件，这导致只能找到一个 `Source`，也就只能唤醒一个 future。fix 的话 `Source` 要能保存两个 `Waker`。
2. 没有保证 `RawFd` 的安全性，因为 `RawFd` 就是个整型，拥有它也不能防止文件被 close，需要实现 reference-counted Fd。

### `epoll`

TODO：如何减少 `epoll_ctl` 调用。

### AIO

AIO 使用的注意点见之前的[博客](/scylladb/disk-io/)吧。

### io_uring

TODO

## 总结

写完发现没什么好写的，淦！下一篇应该会先实现个简单的 executor，然后实现下文件相关的类型，再写个类似 fio 的东西来测测性能。