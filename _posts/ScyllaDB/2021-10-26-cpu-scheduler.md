---
title: ScyllaDB 学习(五) -- CPU scheduler
layout: post
excerpt: 介绍 seastar 的 CPU scheduler 实现。
categories: ScyllaDB
---

{% include toc %}

`seastar` 的一个目标是 control，要能够控制所有任务的 CPU、I/O 资源的使用。要实现 control 是因为在单个系统内经常会有互相竞争的任务，而且任务的特点也不同，有些看重吞吐，所以会尽可能提高资源利用率、做大的 batch等，比如 GC、LSM tree compaction、backup/restore、add/remove nodes 等；有些看重延迟，就要维持低的使用率保证有充足的资源、每个操作尽可能小等，如 OLTP workload。`seastar` 目前实现了 CPU、disk I/O scheduler，network I/O scheduler 还未实现但在计划内，这篇文章介绍 CPU scheduler 的实现。

## 例子

先来看一下效果是什么样的。

```cpp
int main(int ac, char** av) {
    app_template app;
    return app.run(ac, av, [] {
        return seastar::async([] {
            auto sg100 = seastar::create_scheduling_group("sg100", 100).get0();
            auto ksg100 = seastar::defer([&] { seastar::destroy_scheduling_group(sg100).get(); });
            auto sg20 = seastar::create_scheduling_group("sg20", 20).get0();
            auto ksg20 = seastar::defer([&] { seastar::destroy_scheduling_group(sg20).get(); });
            auto sg50 = seastar::create_scheduling_group("sg50", 50).get0();
            auto ksg50 = seastar::defer([&] { seastar::destroy_scheduling_group(sg50).get(); });

            bool done = false;
            auto end = timer<>([&done] {
                done = true;
            });

            end.arm(10s);
            unsigned ctr100 = 0, ctr20 = 0, ctr50 = 0;
            fmt::print("running three scheduling groups with 100% duty cycle each:\n");
            when_all(
                    run_compute_intensive_tasks(sg100, var_fn(done), 5, ctr100, heavy_task),
                    run_compute_intensive_tasks(sg20, var_fn(done), 3, ctr20, light_task),
                    run_compute_intensive_tasks_in_threads(sg50, var_fn(done), 2, ctr50, medium_task)
                    ).get();
            fmt::print("{:10} {:15} {:10} {:12} {:8}\n", "shares", "task_time (us)", "executed", "runtime (ms)", "vruntime");
            fmt::print("{:10d} {:15d} {:10d} {:12d} {:8.2f}\n", 100, 1000, ctr100, ctr100 * 1000 / 1000, ctr100 * 1000 / 1000 / 100.);
            fmt::print("{:10d} {:15d} {:10d} {:12d} {:8.2f}\n", 20, 100, ctr20, ctr20 * 100 / 1000, ctr20 * 100 / 1000 / 20.);TODO: explain what with_scheduling_group group really does, how the group is "inherited" to the continuations started inside it.
            fmt::print("{:10d} {:15d} {:10d} {:12d} {:8.2f}\n", 50, 400, ctr50, ctr50 * 400 / 1000, ctr50 * 400 / 1000 / 50.);
            fmt::print("\n");

            fmt::print("running two scheduling groups with 100%/50% duty cycles (period=1s:\n");
            unsigned ctr100_2 = 0, ctr50_2 = 0;
            done = false;
            end.arm(10s);
            when_all(
                    run_compute_intensive_tasks(sg50, var_fn(done), 5, ctr50_2, heavy_task),
                    run_with_duty_cycle(0.5, 1s, var_fn(done), [=, &ctr100_2] (done_func done) {
                        return run_compute_intensive_tasks(sg100, done, 4, ctr100_2, heavy_task);
                    })
            ).get();
            fmt::print("{:10} {:10} {:15} {:10} {:12} {:8}\n", "shares", "duty", "task_time (us)", "executed", "runtime (ms)", "vruntime");
            fmt::print("{:10d} {:10d} {:15d} {:10d} {:12d} {:8.2f}\n", 100, 50, 1000, ctr100_2, ctr100_2 * 1000 / 1000, ctr100_2 * 1000 / 1000 / 100.);
            fmt::print("{:10d} {:10d} {:15d} {:10d} {:12d} {:8.2f}\n", 50, 100, 400, ctr50_2, ctr50_2 * 1000 / 1000, ctr50_2 * 1000 / 1000 / 50.);

            return 0;
        });
    });
}
```

运行的结果如下：

```
running three scheduling groups with 100% duty cycle each:
shares     task_time (us)  executed   runtime (ms) vruntime
       100            1000       5855         5855    58.55
        20             100      11661         1166    58.30
        50             400       7305         2922    58.44

running two scheduling groups with 100%/50% duty cycles (period=1s:
shares     duty       task_time (us)  executed   runtime (ms) vruntime
       100         50            1000       3360         3360    33.60
        50        100             400       6605         6605   132.10
```

这个 demo 创建了 3 个 scheduling group，即调度单元：`sg100`、`sg20`、`sg50`，shares 比例是 100 : 20 : 50，所以当满负载运行时，它们的运行时间也应该是相应的比例。第一个例子是所有 scheduling group 都满负载运行 10s，每个 scheduling group 用不同的并发度运行不同处理时间的任务，最后的运行时间比是 5855 : 1166 : 2922 ≈ 100 : 20 : 50。第二个例子是 `sg50` 满负载运行而 `sg100` 以 50% util 运行，最后的运行时间比是 `sg50` : `sg100` = 6605 : 3360 ≈ 2 : 1，因为一半的时间 `sg50` 和 `sg100` 同时运行，比例是 1 : 2，另一半时间只有 `sg50` 运行，最终的比例就是 2 : 1 了。 

从这个例子可以看出来，`seastar` 的 CPU scheduler 有这几个特点：

- 当满负载时，各 scheduling group 按比例分配 CPU 时间。
- 当有 scheduling group 空闲时，其他忙的 scheduling group 会占据所有 CPU 时间。
- 空闲的 scheduling group 突然忙起来不会导致饥饿。

能做到这几点肯定不是预先分配好固定的资源，比如按比例分配线程数或 CPU 时间，这样既不能利用所有的资源也不能自适应。

## scheduling group

要实现调度，首先要能区分任务，也就是这里的 scheduling group，用它标记了不同类型的 [`task`](/scylladb/seastar-reactor/#task)。scheduling group 内部就是个 `unsigned _id`，可以很轻量的在函数间传递。`seastar` 支持最多 16 个 scheduling group，默认创建了 `main` 和 `atexit`，所以能用的只有 14 个。每个 scheduling group 都有单独的 task queue，实现是最基础的双端队列，`seastar::schedule()` 就是把任务放到对应 task queue 里等待执行。

![task-queue](/assets/images/scylladb/seastar-task-queue.png)

要把 `task` 和 scheduling group 绑定起来需要用 `with_scheduling_group()` 来执行，那串起来的 `continuation` 是如何继承 scheduling group 的呢？答案是用 `static thread_local scheduling_group sg`，开始处理某个 task queue 时会把它设置为对应的 scheduling group，`task` 的默认构造函数就会继承它：

```cpp
class task {
    scheduling_group _sg;
    explicit task(scheduling_group sg = current_scheduling_group()) noexcept : _sg(sg) {}
    ...
}
```

## CPU scheduling

每个 scheduling group 都有运行时间比重也就是 shares，shares 越大比重越高，shares 也支持动态修改从而可以动态调整运行时间比例来实现 workload-conditioning。shares 是和 task queue 关联的，task queue 会根据 shares 统计该 scheduling group 的运行时间等信息，`reactor::run_some_tasks()` 就根据这些统计信息选择合适的 task queue 运行来实现 CPU scheduling。

```cpp
void
reactor::run_some_tasks() {
    ...
    sched_clock::time_point t_run_completed = std::chrono::steady_clock::now();
    do {
        auto t_run_started = t_run_completed;
        insert_activating_task_queues();
        task_queue* tq = pop_active_task_queue(t_run_started);
        _last_vruntime = std::max(tq->_vruntime, _last_vruntime);
        run_tasks(*tq);
        tq->_current = false;
        t_run_completed = std::chrono::steady_clock::now();
        auto delta = t_run_completed - t_run_started;
        account_runtime(*tq, delta);
        ...
    } while (have_more_tasks() && !need_preempt());
    ...
}
```

实现其实就是个优先级队列，先处理优先级高的 task queue，那优先级通过什么来决定呢？当所有 scheduling group 都忙的时候，它们的运行时间比例应该满足 runtime1 : runtime2 : runtime3 = share1 : share2 : share3，变换一下形式就能得到可比较大小的指标：runtime1/share1 = runtime2/share2 = runtime3/share3，task queue 就是用这个指标决定优先级，内部叫 `vruntime`，越小优先级越高，先左移 32 位再右移回去可能是为了避免浮点数运算。

```cpp
_reciprocal_shares_times_2_power_32 = (uint64_t(1) << 32) / _shares;

reactor::task_queue::to_vruntime(sched_clock::duration runtime) const {
    auto scaled = (runtime.count() * _reciprocal_shares_times_2_power_32) >> 32;
    // Prevent overflow from returning ridiculous values
    return std::max<int64_t>(scaled, 0);
}
```

虽然有了衡量优先级的指标，但还有很多问题需要解决：

1. 如果某个 scheduling group 之前一直空闲，它的 `vruntime` 就会很小，当突然来了大量任务时，会导致其他 scheduling group 饥饿。
2. 如果每个 scheduling group 都有大量 task，那么每个 scheduling group 都会独占很长一段时间，会导致在短时间内 CPU 时间不成比例。
3. 单个 scheduling group 内也要保证公平，如果有个 loop future 一直 ready，就会一直执行它，导致 scheduling group 内发生饥饿。

这几个问题的本质是调度需要保证在足够小的时间周期内 CPU 时间成比例，过去的执行时间也就不会影响当前的调度了。解决这个问题的常用手段是引入 timeslice，运行时间等统计信息也要按照 timeslice 来更新，每个调度单元运行完分配的 timeslice 后需要 yield 出去让调度器调度。

![cpu-scheduler](/assets/images/scylladb/seastar-CPU-scheduler.png)

先来看第一个问题是如何解决的，`reactor` 在处理 task queue 前会记录它的 `vruntime`：`_last_vruntime = std::max(tq->_vruntime, _last_vruntime)`，也就是当前最小的 `vruntime`，空闲很久的 scheduling group 来任务时会继承 `_last_vruntime`，从而保证它和别的 scheduling group 在相同的起跑线，且优先级最高。

```cpp
void
reactor::activate(task_queue& tq) {
    if (tq._active) {
        return;
    }
    // If activate() was called, the task queue is likely network-bound or I/O bound, not CPU-bound. As
    // such its vruntime will be low, and it will have a large advantage over other task queues. Limit
    // the advantage so it doesn't dominate scheduling for a long time, in case it _does_ become CPU
    // bound later.
    //
    // FIXME: different scheduling groups have different sensitivity to jitter, take advantage
    if (_last_vruntime > tq._vruntime) {
        sched_print("tq {} {} losing vruntime {} due to sleep", (void*)&tq, tq._name, _last_vruntime - tq._vruntime);
    }
    tq._vruntime = std::max(_last_vruntime, tq._vruntime);
    ...
}
```

这里可能有人会问，如果每个 task queue 来 task 都 `activate()` 的话那不是每个 scheduling group 运行时间比例都相同？不然，shares 小的 scheduling group 即使空闲了一段时间，它的 `vruntime` 也可能比 `_last_vruntime` 大，而且 `activate()` 只有 task queue 从空变为非空才会调用，这本身就说明某个 scheduling group 不是 CPU bound，如果所有 scheduling group 都这样，那整个系统也不会是 CPU bound，这时候 CPU scheduling 意义也就不大了。

解决后面两个问题需要引入 preemption，因为是 cooperative 的，所以需要在代码里主动加上 preemption check(`need_preempt()`)：

- 处理完每个 task 都会检查是否需要 yield。
- 单个长的 task 比如 loop 也需要在内部加入 preemption check。

### preemption 实现

最开始 `seastar` 是基于计数的实现，也就是运行一定个数个 task 后 yield 出去，但每个 task 的耗时不尽相同，靠计数的实现无法精确控制时间，所以后来是基于时间实现的 preemption（[reactor: switch to time-based task quotas](https://github.com/scylladb/seastar/commit/d71f1b062098c24691bf533d95615ded8112c078)）。这种 preemption 有多种实现方式，比如：

- 每次运行 task 后比较 timeslice 是否耗尽，但这需要频繁获取时间，代价比较大。

- 用 POSIX interval timer(`timer_create()`)，interval 为 timeslice，在 signal handler 里设置需要 preemption。`seastar` 最开始用的是这个方案，但即使 signal 是发给单个线程的，仍有个进程级别的锁，扩展性不好（[posix signals don't scale on large machines](https://github.com/scylladb/seastar/issues/324)），所以换成了下面的方案。

- 用 `timerfd_create()` 和单独的 thread 处理它并设置需要 preemption（[reactor: stop using signals for task_quota timer](https://github.com/scylladb/seastar/commit/c175cb527806)）。`timerfd` 用 fd 来通知不再是进程级别的，扩展性很好。单独的 thread 使用 `SCHED_FIFO` 调度，从而可以立刻抢占 `reactor` 线程的执行，但单独的 thread 引入了 context switch 的开销。

- 最后 `seastar` 用了非常 tricky 的方案（[IOCB_CMD_POLL support](https://github.com/scylladb/seastar/commit/6998902c60e27d6b8d2d92833a83da3fa0c255f0)），利用 linux AIO 支持了 `IOCB_CMD_POLL` 从而可以 poll `timerfd`，且 linux AIO 事件通知会修改 user space 的内存，通过检查这部分内存的变化得知需要 preemption。现在来看下具体实现，AIO completion ring 是下面的结构，`aio_context_t` 的值其实就是它的地址，`need_preempt()` 就是检查用于 `timerfd` 的 `aio_context_t` 对应的 `linux_aio_ring` 的 `head` 和 `tail` 是否变化。

  ```cpp
  struct linux_aio_ring {
      uint32_t id;
      uint32_t nr;
      std::atomic<uint32_t> head;
      std::atomic<uint32_t> tail;
      uint32_t magic;
      uint32_t compat_features;
      uint32_t incompat_features;
      uint32_t header_length;
  };
  
  struct preemption_monitor {
      // We preempt when head != tail
      // This happens to match the Linux aio completion ring, so we can have the
      // kernel preempt a task by queuing a completion event to an io_context.
      std::atomic<uint32_t> head;
      std::atomic<uint32_t> tail;
  };
  
  inline bool need_preempt() noexcept {
      // prevent compiler from eliminating loads in a loop
      std::atomic_signal_fence(std::memory_order_seq_cst);
      auto np = internal::get_need_preempt_var();
      // We aren't reading anything from the ring, so we don't need
      // any barriers.
      auto head = np->head.load(std::memory_order_relaxed);
      auto tail = np->tail.load(std::memory_order_relaxed);
      // Possible optimization: read head and tail in a single 64-bit load,
      // and find a funky way to compare the two 32-bit halves.
      return __builtin_expect(head != tail, false);
  }
  ```


`seastar` 有两个 [reactor backend](/scylladb/seastar-reactor/#reactor-backend)，`epoll` backend 用的是第三个方案，AIO backend 用的是最后的方案。

### task quota

task quota 就是前面写的 timeslice，可通过命令行参数 `task-quota-ms` 控制，默认是 0.5ms，所以如果是 `timerfd` 和单独线程的 preemption 实现，每个核每秒一来一回就多了 4000 次 context switch 的开销，这肯定无法接受。task quota 不仅仅用于 CPU scheduling，它还影响端到端的延迟和 I/O scheduler。在[之前的文章](/scylladb/seastar-reactor/#poller) 里介绍了 `poller`，它们是 `reactor` 和其他组件交互的接口，`poller` 其实相当于是额外的 task queue，poll 会添加新任务，而且其中可能会有高优先的任务，所以需要定期 poll，当 task quota 消耗完也就是发生 preemption 时就会 poll，task quota 也就是 poll 的间隔。I/O 相关的操作都是 poll 时处理的，所以 task quota 也是提交 disk I/O 请求的间隔，这影响了 disk I/O scheduler 的实现，而且一般来说每个请求都涉及到 I/O，假如收割 disk I/O 后就可以返回响应了，poll 的间隔也就影响了请求的延迟。

## CPU stall detector

thread-per-core 需要保证每个 task 执行时间都很短， `seastar` 实现了 CPU stall detector 用于检测出运行时间太久的 task，比如在 loop 中没加 preemption check 的。实现用的是 POSIX interval timer，在 signal handler 里比较 task 处理数量得知是否还在处理单个 task，默认配置下超过 2s 就认为是 stall 会打印出 backtrace，这利用了 signal handler 是在收到信号的线程栈里执行的机制。

## 总结

`seastar` 用非常简单、朴素的策略就实现了 CPU scheduler，当然这只是基础设施，`scylladb` 在此基础上根据 control theory 实现了 workload-conditioning，之后也会学习一下 `scylladb` 是如何使用它的。下一篇文章会介绍 disk I/O scheduler，它的实现非常复杂，也经历了很多轮的迭代演进，不像 CPU scheduler 在很早期就确定了实现，不出意外的话下一篇需要很久才能出来。 