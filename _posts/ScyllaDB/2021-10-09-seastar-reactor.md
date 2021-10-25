---
title: ScyllaDB 学习(三) -- seastar reactor
layout: post
excerpt: 介绍 seastar 的启动流程和架构。
categories: ScyllaDB
---

{% include toc %}

版本：[8ed9771ae98130d5adbccdaea4ecbac738170939](https://github.com/scylladb/seastar/tree/8ed9771ae98130d5adbccdaea4ecbac738170939)

## main loop

`seastar` 的启动和 Rust runtime 类似，需要运行一个 root future，也就是这里的 `func`，当 root future 完成时 `seastar` 也就会退出：

```cpp
int app_template::run(int ac, char ** av, std::function<future<int> ()>&& func);
```

`app_template::run()` 里会调用 `smp::configure()` 根据命令行参数进行配置和初始化，包括根据 `hwloc` 分配 CPU 和内存资源、根据 disk 配置分配 I/O 资源、选择网络协议栈和 reactor 实现等，会在之后写到对应部分时再详细介绍。默认配置下 `seastar` 会使用所有的 CPU 和内存，每个 core 上都会 pin 住一个线程运行 `reactor::run()` 也就是 event loop。

简单来说，`reactor::run()` 就是不断 poll 各种事件源把 task 放进 task queue 里再处理。task queue 是包含多个 sub task queue 的优先级队列，队列的优先级根据每个 sub task queue 的 CPU 运行时间和权重来排序，会在之后介绍 CPU scheduler 时详细介绍。

```cpp
int reactor::run() {
    ...
    std::function<bool()> check_for_work = [this] () {
        return poll_once() || have_more_tasks();
    };
    while (true) {
        run_some_tasks();
        if (_stopped) {
         	...
            break;
        }
        // poll tasks
        if (check_for_work()) {
            ...
        } else {
            // go to sleep
            ...
        }
    }
}
```
## task

`task` 类似 Rust future trait，每个任务都要继承它。每个 `task` 都和某个 `scheduling_group` 绑定，这也是和 CPU scheduler 相关的。`seastar` 里继承了 `task` 的核心结构是 `continuation`，会在之后一篇文章里详细介绍 `seastar` 的 future/promise/continuation 实现。

```cpp
class task {
    scheduling_group _sg;
public:
    explicit task(scheduling_group sg = current_scheduling_group()) noexcept : _sg(sg) {}
    virtual void run_and_dispose() noexcept = 0;
    scheduling_group group() const { return _sg; }
    ...
}
```

运行 `task` 只需要 schedule 到 task queue 即可，对单个 task queue 来说就是不断 pop 再调用 `task::run_and_dispose()`。

```cpp
void schedule(task* t) noexcept {
    engine().add_task(t);
}
```

## poller

`poller` 是 `reactor` 和其他线程、操作系统、硬件等交互的接口，它的作用就是发送（接收）消息给（从）其他部分。`seastar` 里有多种事件源，比如因为是 thread-per-core 架构，线程间同步通过 message passing，所以要定期 poll message queue；AIO 等异步操作的完成也需要主动收割(reap)。现在各种 bypass kernel 或者高性能的系统调用也都提供了 poll mode，比如 DPDK PMD、io_uring，相比用中断来通知，poll mode 能避免 context switch，从而有更好的 cache locality；也能做到 zero copy，避免在用户态和内核态之间复制数据。

```cpp
struct pollfn {
    virtual ~pollfn() {}
    // Returns true if work was done (false = idle)
    virtual bool poll() = 0;
    // Checks if work needs to be done, but without actually doing any
    // returns true if works needs to be done (false = idle)
    virtual bool pure_poll() = 0;
    // Tries to enter interrupt mode.
    //
    // If it returns true, then events from this poller will wake
    // a sleeping idle loop, and exit_interrupt_mode() must be called
    // to return to normal polling.
    //
    // If it returns false, the sleeping idle loop may not be entered.
    virtual bool try_enter_interrupt_mode() = 0;
    virtual void exit_interrupt_mode() = 0;
};
```

`poller` 都继承了上面的 `pollfn`，`reactor::run()` 里的 `check_for_work()` 就是 poll 一遍注册的所有 `poller`，有新的 task 就 schedule 到 task queue。在默认配置下，`seastar` 会在 poll mode 下运行，也就是一直 poll，即使是空闲状态 CPU 使用率也会保持在 100%，但也提供了 interrupt mode 选项，当空 poll 超过一定时间后会进入睡眠状态，等待有事件就绪后才会 poll。`reactor` 里的 `poller` 如下：

```cpp
    poller smp_poller(std::make_unique<smp_pollfn>(*this));
    poller reap_kernel_completions_poller(std::make_unique<reap_kernel_completions_pollfn>(*this));
    poller io_queue_submission_poller(std::make_unique<io_queue_submission_pollfn>(*this));
    poller kernel_submit_work_poller(std::make_unique<kernel_submit_work_pollfn>(*this));
    poller final_real_kernel_completions_poller(std::make_unique<reap_kernel_completions_pollfn>(*this));
    poller batch_flush_poller(std::make_unique<batch_flush_pollfn>(*this));
    poller execution_stage_poller(std::make_unique<execution_stage_pollfn>());
    poller syscall_poller(std::make_unique<syscall_pollfn>(*this));
    poller drain_cross_cpu_freelist(std::make_unique<drain_cross_cpu_freelist_pollfn>());
    poller expire_lowres_timers(std::make_unique<lowres_timer_pollfn>(*this));
    poller sig_poller(std::make_unique<signal_pollfn>(*this));
```

大部分看名字也能看出来是处理哪部分的，简单介绍一下：

1. `smp_poller`：处理 `reactor` 间通信的，每对 `reactor` 都有单独的 `smp_message_queue`。
2. `reap_kernel_completions_poller`：收割 AIO 请求。
3. `io_queue_submission_poller`：处理 I/O queue 里的请求，这里涉及到 I/O scheduler。
4. `kernel_submit_work_poller`：提交 AIO 请求，也包括 socket 事件相关的，如 `epoll_wait()`。
5. `batch_flush_poller`：发送网络响应。
6. `execution_stage_poller`：还不清楚做什么用的，好像是攒 batch 相关的。
7. `syscall_poller`：因为不是所有 syscall 都支持异步，所以每个 `reactor` 都有个线程用来执行阻塞式的 syscall，这个 poller 就是检查这些的。
8. `drain_cross_cpu_freelist`：因为 memory 都是预先给每个 core 分配好的，跨线程释放的内存会还给对应的 `reactor` 来释放。
9. `expire_lowres_timers`：处理超时的 timer。lowres(低精度)是因为这是 poll 时才检测定时器超时，poll 调用的间隔决定了 timer 的精度。highres 由其他机制实现，timer 这部分可能会单独介绍一下。
10. `sig_poller`：处理触发的 signal，signal 触发时会设置对应的 bit，poll 时再调用真正的 signal handler。`seastar` 里 signal 用的还挺多，因为这是为数不多能中断程序执行的操作。

## reactor backend

`seastar` 里的 `reactor` 是处理所有事件的 event loop，和 disk I/O、socket event 还有定时器相关的事件是由 `reactor_backend` 实现的，从方法名字也能看出来上面的 `reap_kernel_completions_poller` 和 `kernel_submit_work_poller` 由它实现。

```cpp
class reactor_backend {
public:
    virtual bool reap_kernel_completions() = 0;
    virtual bool kernel_submit_work() = 0;
    virtual bool kernel_events_can_sleep() const = 0;
    virtual void wait_and_process_events(const sigset_t* active_sigmask = nullptr) = 0;

    virtual future<> readable(pollable_fd_state& fd) = 0;
    virtual future<> writeable(pollable_fd_state& fd) = 0;
    virtual future<> readable_or_writeable(pollable_fd_state& fd) = 0;
    virtual void forget(pollable_fd_state& fd) noexcept = 0;
    ...
    virtual pollable_fd_state_ptr make_pollable_fd_state(file_desc fd, pollable_fd::speculation speculate) = 0;
};
```

`reactor_backend` 有两个实现：`epoll` 和 AIO，当 AIO 可用时默认是 AIO 实现。disk I/O 两者用的都是 linux AIO，disk I/O 相关的会有单独一篇文章来介绍，这里只介绍下网络相关的。两者 network I/O 实现相同，在 posix network stack 下用的都是 socket API，区别在于网络事件相关的前者用的是 `epoll` 而后者还是 linux AIO，linux 4.18 新增了 [`IOCB_CMD_POLL`](https://lwn.net/Articles/743714/)，从而 AIO 也能用于 poll socket event，相比 `epoll` 有 10% 性能提升，因为能减少 syscall(`epoll_wait`&`epoll_ctl`)。以 `readable()` 实现为例介绍一下两者的区别：

* `pollable_fd_state` 记录了 socket 注册、触发的事件等，`epoll`/AIO backend 都在此基础上增加了要调用的 callback 等。

  ```cpp
  class pollable_fd_state {
      unsigned _refs = 0;
  public:
      void speculate_epoll(int events) { events_known |= events; }
      file_desc fd;
      bool events_rw = false;   // single consumer for both read and write (accept())
      int events_requested = 0; // wanted by pollin/pollout promises
      int events_epoll = 0;     // installed in epoll
      int events_known = 0;     // returned from epoll
      ...
  };
  ```

* `epoll`：`readable()` 就是设置 `events_requested |= EPOLLIN `，用 `epoll_ctl()` 注册 LT + `EPOLLIN`。`seastar` 没用 oneshot，虽然这会更通用，但会增加 `epoll_ctl()` 的调用次数，它用别的方式也达到了同样的效果：在 `epoll_wait()` 返回第一次事件就绪时不会清理掉注册的事件，但会清理 `events_requested`，当下一次 `epoll_wait()` 再返回了该事件就绪时发现 `events_requested` 里没有对应事件再用 `epoll_ctl()` 删掉，效果就是当一直对某事件感兴趣时，比如一直调用 `readable()`， 也只会有一次 `epoll_ctl()` 调用，当不再感兴趣时也只会多一次 `epoll_wait()` 的唤醒。
* AIO：AIO 只有 oneshot 模式，它的 `readable()` 就只是准备好 `iocb`，在 `kernel_submit_work()` 里再一次性 `io_submit()`，这是 AIO 能减少 syscall 的一个原因：它支持 batch，而 `epoll_ctl()` 一次只能操作一个 fd。

说完了事件是如何处理的，就到了如何调用 callback 了。event loop 实现一般都是在 `epoll_event.data.ptr` 或者 `iocb.aio_data` 里保存 socket 对应类的指针，类里会有 callback，事件就绪时就调用对应的 callback，`seastar` 也是这样实现的，不过它的 callback 是完成 `promise`。`readable()` 返回的是 `future` 用 `then()` 就能串起来 callback，典型的用法就是 `_backend->readable(fd).then([&fd] { fd.read()... })`，这极大的改善了开发体验。

## 总结

`seastar` 的架构是比较常见的，优化都在细节和实现上。下一篇会介绍 future/promise/continuation 的实现。