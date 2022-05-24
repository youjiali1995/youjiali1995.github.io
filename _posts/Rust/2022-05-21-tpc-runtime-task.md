---
title: 写一个 Rust TPC Runtime(一) -- Task
layout: post
excerpt: 介绍 Task 的设计和实现。
categories: Rust
---

{% include toc %}

task 是 stateful future，需要保存 future、执行状态和 executor 的一些信息。task 的性能至关重要，主要就是要减少内存分配次数和大小。

## 接口

runtime 一般都会提供 spawn 方法来执行一个并发的 future，且需要返回能够获取返回值的 future。以 [`async-task`](https://github.com/smol-rs/async-task/blob/de0c79d171e95d1cbbd4becf678cc43ea689551e/src/runnable.rs#L45-L52) 为例，`Runnable` 是 stateful future，`Task<F::Output>` 是获取 `Future::Output` 的 future。

```rust
pub fn spawn<F, S>(future: F, schedule: S) -> (Runnable, Task<F::Output>)
where
    F: Future + Send + 'static,
    F::Output: Send + 'static,
    S: Fn(Runnable) + Send + Sync + 'static,
{
    ...
}
```

[`tokio`](https://github.com/tokio-rs/tokio/blob/221bb94b9c78f137cb0675f62b6d2e26cdf8d987/tokio/src/runtime/task/mod.rs#L251-L272) 也大同小异，[`glommio`](https://github.com/DataDog/glommio/blob/676d180b298387621ebae488a746fca7ecd3cc12/glommio/src/task/task_impl.rs#L27-L51) 和 [`monoio`](https://github.com/bytedance/monoio/blob/a3eaf61ba0b0cf5b44fc5e420ea816cda19df736/monoio/src/task/mod.rs#L72-L80) 分别是借鉴的 `async-task` 和 `tokio`，[`yatp`](https://github.com/tikv/yatp/blob/5f3d58002b383bfd0014e271ae58261ecc072de3/src/task/future.rs#L180-L196) 因为是 `Future<Output = ()>` 就有一些区别。

## 内存分配

future 的类型各不相同，而 executor 需要保存统一类型的 task，就不能有范型 ，需要做类型擦除：

- 一种做法是用 trait object，也就是
  ```rust
  struct Task {
      ...
      future: Box<dyn Future<Output = ()>>,
  }
  ```
  但这会导致两次内存分配：一次 future 的 trait object，一次 task，也限制了 `Future<Output>` 的类型。
- 另一种就是分配包含 future 在内的一整块内存，future 的类型信息通过 vtable 来保存，类似 [RawWaker](https://doc.rust-lang.org/std/task/struct.RawWaker.html)。task 只要保存这块内存的地址即可。

`tokio` [早期](https://tokio.rs/blog/2019-10-scheduler#reducing-allocations)用的方案一，现在常见的 runtime 都用的方案二。task 里大概需要保存这些信息：

- future 和 output：不会同时存在，可以共用一块内存。
- 状态信息：future 的状态转换有一定约束，比如已经完成的 future 就不应该 wake 和 poll 了，task 就需要保存 future 的当前状态，通常引用计数也会保存在这里。
- 获取 output 的 waker：用于 future ready 后唤醒获取 output 的 future。
- vtable：包含所有需要类型信息的函数，类型信息可通过函数的范型附带上。
- scheduler（可选）：wake 时调用的方法。有些 runtime 会有多种线程和调度模型，就需要保存这部分信息，如果只有一种的话那写死就行。

## 状态转换

task 的状态转换和 task 的定位、runtime 的实现有关，比如 `async-task` 的目标是通用的 task 实现，就需要支持多线程即用 CAS 来执行状态转换，需要考虑各种可能出现 data race 的情况，`async-task` 的实现见[之前的博客](/rust/async/#async-task)。

典型的 task 状态转换大致如下：

1. 创建 task，状态为 idle；
2. task 扔进 task queue，状态为 scheduled；
3. task 在 task queue 中被处理到，状态为 running；
   - ready：状态为 completed；
   - pending：注册 waker 到 reactor。reactor 调用 wake 再把 task 扔进 task queue，状态为 scheduled。

上面只是最基本的状态转换，task 有些约束需要遵循，比如不应该 poll 已经完成的 task；不应该把 task 同时扔进 task queue 多次，否则就可能 poll 已经完成的 task 或者多个线程同时 poll 单个 task。Rust 的异步抽象无法防止上述情况，全靠 runtime 实现来保证。我这里实现的是 TPC 的 runtime，reactor 也在相同的线程，所以没有 data race，而且 runtime 会有一定约束，比如 task 创建后会立刻 schedule、reactor 不会 wake 多次 task，状态转换就会简单很多。至于引用计数，task、获取结果的 future 和 waker 均算一个，注意好所有权转移即可。

## MISC

- [`Waker`](https://doc.rust-lang.org/beta/std/task/struct.Waker.html) 是 `Send + Sync` 的，这里的实现不满足要求，可能会出现在其他线程调用了 waker 的情况，比如用 [`flume`](https://github.com/zesterer/flume) 扔 task 给 executor 线程时就可能调用 receiver 注册的 waker。解决办法是要实现 foreign wake，通过判断是否在 executor 线程来选择 wake 的实现，比如把 task 扔进一个 thread-safe 的 queue 并唤醒 executor 线程。

## seastar

`seastar` 里分为 future/promise/continuation。`seastar` 里的 async function 是立即执行的，future/promise 可以看作是 spsc oneshot channel，只用于通知结果不包含需要运行的 task，也就无需动态内存分配，而 rust 的 future 只有 poll 了才生效，通常也不会为立刻 ready 的 future 做优化，所以需要内存分配。

continuation 是 future ready 后运行的 task，这里才需要动态内存分配一次。continuation 只有在能获取上一个 future 结果时才会运行且只会运行一次，所以也就没有各种状态转换需要处理，而 rust 允许 poll 和 wake 多次 future，在有了 async/.await 后 future 会有 sub-future，这种行为就成为了常态，也就很难像 `seastar` 这样做，各有利弊吧。