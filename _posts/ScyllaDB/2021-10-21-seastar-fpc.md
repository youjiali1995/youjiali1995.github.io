---
title: ScyllaDB 学习(四) -- seastar f/p/c
layout: post
excerpt: 介绍 seastar 的 future/promise/continuation 实现。
categories: ScyllaDB
---

{% include toc %}

## 异步编程

介绍实现之前先简单介绍一下异步编程的几种模式。首先是 callback 这种，写出来的代码是这样的：

```cpp
async_func1([] {
    async_func2([] {
        async_func3(
            ...
        )
    })
})
```

操作需要通过 callback 传进去，当嵌套的异步操作越来越多时，缩进也越来越深，如果是不支持 lambda 的语言比如 C，写起来就更头疼了，callback 需要提前定义好，定义和执行的顺序也反过来了：

```c
...
void callback1() {
    async_func3(...)
}
void callback2() {
    async_func2(callback1)
}
async_func1(callback2)
```

那用 f/p/c 会变成什么样呢？async function 不再是等时机到了调用 callback，而是返回包含返回值的 future，callback 再通过 then 串起来，所有的操作都在同样的缩进：

```cpp
async_func1().then([] (auto result) {
    async_func2(result)
}).then([] (auto result) {
    async_func3(result)
})
...
```

但 f/p/c 和同步编程相比还是不方便，变量的生命周期管理、lambda 等都带来了很多噪声。用 async/await 后，await 能获取 future 的结果，写起来就和同步编程差不多了：

```rust
let result1 = async_func1().await;
let result2 = async_func2(result1).await;
let result3 = async_func3(result2).await;
...
```

## future/promise/continuation

通过 f/p/c，`seastar` 统一了异步操作的接口：

1. 所有的异步操作只需要保存 `promise`，返回 `future`，异步操作完成时通过 `promise.set_value()` 通知 `future`；
2. caller 只需要得到 `future`，callback 通过 `then()` 串起来且会在合适的时机被调用。

```cpp
future<> async_func() {
    ...
    auto pr = promise<>();
    auto fut = pr.get_future();
    // save the `promise` somewhere and call `promise.set_value()` when the async function is done to resolve the corresponding `future`.
    ...
    return fut;
}

// caller
async_func().then([] () { ... }).then([] () { ... })...;
```

future/promise 很像 oneshot spsc channel，promise 是发送端，future 是接收端，不过是非阻塞式的。`future` 的结果除了用 `then()` 获取，也可以通过 `future.get()`，但只有 `future` 已经完成或者是在 `seastar::thread` 里执行才行，否则程序会挂掉，`seastar::thread` 是基于 future/promise 实现的有栈协程，之后会简单介绍一下。

现在来看一下 future/promise 是如何实现的，先自己想一下：

* future/promise 是互相关联的，必然会用指针互相指着；
* future/promise 需要共享一些数据，如状态、返回值。

思想差不多是这样，忽略 C++ 复杂的模板元编程的话，实现就很简单，因为 `seastar` 里 f/p/c 始终运行在单个线程，也没有 data race。future/promise 自身不需要动态内存分配，和用到它的地方一起分配即可，但默认的 move 行为会导致地址失效，所以在 move constructor/assignment 里妥善更新了指针。

![fp](/assets/images/scylladb/seastar-fp.svg)

只有 future/promise 并不能改善异步编程的体验，更重要的是支持 continuation，也就是 `then()`。continuation 是当 future ready 时执行的 callback，所以肯定是在 `promise.set_value()` 里做了些什么。确实是这样，上面图里的 `promise` 保存了 `task`，就是[上篇文章](/scylladb/seastar-reactor/#task)里写的执行单元，`promise.set_value()` 会把 `task` 扔进 task queue 里执行。

```cpp
template <typename... A>
void promise_base_with_type::set_value(A&&... a) noexcept {
    if (auto *s = get_state()) {
        s->set(std::forward<A>(a)...);
        make_ready<urgent::no>();
    }
}

template <promise_base::urgent Urgent>
void promise_base::make_ready() noexcept {
    if (_task) {
        if (Urgent == urgent::yes) {
            ::seastar::schedule_urgent(std::exchange(_task, nullptr));
        } else {
            ::seastar::schedule(std::exchange(_task, nullptr));
        }
    }
}
```

`then()` 返回的也是 `future` 所以可以无限 `then()` 下去，那么 continuation 是如何和 future/promise 关联的呢？先想一下它的执行流程：

1. 第一个 `future` 完成后要执行第一个 callback，所以 `then()` 要把 callback 设为 `promise._task`；
2. 第一个 callback 返回的 `future` 完成要执行第二个 callback，也需要把 callback 设为它对应的 `promise._task`。
3. 如此往复，`promise` 串起了所有的 callback，是链表状的结构。

只完成第一步很简单，`then()` 把 `promise` 和 `continuation` 关联起来：`continuation` 保存了 callback 且实现了 `task` 接口；future result 通过 `continuation._state` 传递。

![c](/assets/images/scylladb/seastar-c.svg)

第二步就复杂些，它和第一步的区别在于第一步的 future/promise 是已经存在的，而第二步的 future/promise 需要执行 callback 才会有，那么 `then()` 返回的 `future` 是和谁关联的？它的 `promise` 又是如何和下一个 callback 关联的呢？

![then](/assets/images/scylladb/seastar-then.svg)

上图里相同颜色的 future/promise 是初始时相互关联的，1.1-1.2 是第一次 `then()` 的流程，2.1-2.2 是第二次 `then()`，3.1-3.3 是 `continuation.run_and_dispose()` 的流程：

1. 第一次 `then()` 就是上面说的那样，把 `promise` 和 `continuation` 关联起来。`continuation` 里还保存了和 `then()` 返回的 `future` 相关联的 `promise`，用于串起来下一个 `continuation`。
2. 第二次 `then()` 和第一次唯一的区别是 `continuation` 是和上一个的 `continuation._pr` 关联，从而串起来 `continuation`。
3. 当第一个 future ready 后，会运行第一个 `continuation`，callback 返回的 `future` 对应的 `promise` 会被  `continuation._pr`  覆盖，从而当 future ready 后能执行下一个 `continuation`。

因为没空间存放 `continuation`，所以每次 `then()` 都需要动态内存分配一次。ready future 的 `then()` 会特殊处理：

- 调用 `then()` 的 future 是 ready 的话，callback 会立即执行，避免了上面的流程。
- `continuation.run_and_dispose()` 返回 ready future 的话，会 `make_ready<urgent::yes>()`，也就会调用 `seastar::schedule_urgent()`，和 `seastar::schedule()` 的区别在于 urgent 会把 task 放到队首，也就会立即执行 task，而非 urgent 是放到队尾，需要等一批 task。 

### 和 Rust 的区别

`seastar` 的 f/p/c 只在 `seastar` 里使用，不是通用的设计，和 Rust 的区别有这几点：

- 只支持单线程使用。
- `seastar` 的 future 只有 ready 时才会运行且只运行一次，Rust 可以 poll 多次 future。
- `seastar` 的 future 不需要 spawn 给 runtime 就会运行，因为创建 future 的 async function 已经开始异步操作了，想要在 `then()` 里 spawn background 只需要不返回对应的 future。Rust 的 future 需要 poll 才会有效，所以需要 spawn 给 runtime。

## seastar::thread

f/p/c 在需要多次运行单个 callback 的场景用起来很不方便，比如在 loop 里，因为每次运行都需要创建 `continuation`、串起来 callback 等，`seastar` 需要为常见的 control flow 都实现特定的类。

```cpp
void sync_func() {
    std::cout << "Hi.\n";
    for (int i = 1; i < 4; i++) {
        sleep(1);
        std::cout << i << "\n";
    }
}

future<> async_func() {
    std::cout << "Hi.\n";
    return seastar::do_for_each(boost::counting_iterator<int>(1),
        boost::counting_iterator<int>(4), [] (int i) {
        return seastar::sleep(std::chrono::seconds(1)).then([i] {
            std::cout << i << "\n";
        });
    });
}
```

为了解决这个问题，`seastar` 基于 f/p/c 实现了有栈协程，允许在 `seastar::thread` 里使用 `future.get()` 来等待 future ready：

```cpp
future<> async_func() {
    seastar::thread th([] {
        std::cout << "Hi.\n";
        for (int i = 1; i < 4; i++) {
            seastar::sleep(std::chrono::seconds(1)).get();
            std::cout << i << "\n";
        }
    });
    ...
}
```

`future.get()` 是 switch point，未 ready 的 future 在 `seastar::thread` 环境里调用会主动 switch out，同时设置对应的 `promise._task` 为 switch in，`promise.set_value()` 就会继续执行这个 `future`。`seastar` 用 `getcontext()`/`makecontext()`/`setcontext()` 为每个 `seastar::thread` 分配了 128KiB 的栈，switch in/out 是用 `setjmp()`/`longjmp()` 实现的，因为 `seastar::thread` 会和 f/p/c 在相同线程使用，`seastar` 用链表记录了创建 `seastar::thread` 的调用栈的 `jmpbuf`，以便 `future.get()` 要阻塞时 switch 回去继续返回 `future`。

这里就简单介绍一下 `seastar::thread` 的实现，感兴趣的可以看看上面几个库函数的 man page 和源码。因为 `seastar::thread` 有 128KiB 的栈、切换代价也比较大，所以在高并发场景下不建议用它。`seastar` 还支持了 C++ coroutine，这不在我学习范围内，就不介绍了。

## 总结

`seastar` 的基础部分大概都介绍完了，后面会开坑有趣的部分，下一篇会介绍 CPU scheduler。