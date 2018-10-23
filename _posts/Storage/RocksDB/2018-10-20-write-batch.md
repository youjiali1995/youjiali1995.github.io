---
title: RocksDB 源码分析 -- Write Batch
layout: post
excerpt: 介绍 Write Batch 实现。
categories: RocksDB
---

{% include toc %}

`RocksDB` 和 `LevelDB` 相同，不支持并行写，原因有几个：
1. `memtable` 可能不支持并行写。
2. 单纯的并行写不能保证 `WAL` 和操作的执行顺序一致，要有同步机制。

一条条的写 `WAL` 性能、执行命令不高，所以 `LevelDB/RocksDB` 会把多个写操作合并由单个 `leader` 线程来执行，`leader` 执行完成后通知其他线程写操作完成。
`LevelDB` 的实现很简单，写操作添加到 `deque` 中，队首的 `leader` 合并其他写操作来执行，`deque` 由 `mutex` 保护，通知由 `cond var` 实现。

## JoinBatchGroup
`RocksDB` 的实现做了很多优化，用于提高性能。每个 `Writer` 会处于以下状态之一：
* `STATE_INIT`
* `STATE_GROUP_LEADER`
* `STATE_MEMTABLE_WRITER_LEADER`
* `STATE_PARALLEL_MEMTABLE_WRITER`
* `STATE_COMPLETED`
* `STATE_LOCKED_WAITING`

初始时，每个 `Writer` 都是 `STATE_INIT`。`Writer` 添加到队列中来竞争 `leader`:
* `Writer` 实现为双向链表结点: `Writer* link_older/link_newer`。
* `WriteThread` 中维护原子变量的链表头，指向最新添加的 `Writer`: `std::atomic<Writer*> newest_writer_`。
* 采用 `lock-free` 方式将 `Writer` 添加到链表中。
```cpp
bool WriteThread::LinkOne(Writer* w, std::atomic<Writer*>* newest_writer) {
  assert(newest_writer != nullptr);
  assert(w->state == STATE_INIT);
  Writer* writers = newest_writer->load(std::memory_order_relaxed);
  while (true) {
    ...
    w->link_older = writers;
    if (newest_writer->compare_exchange_weak(writers, w)) {
      return (writers == nullptr);
    }
  }
}
```

第一个添加成功的，也就是添加时 `newest_writer_` 为 `nullptr` 的 `Writer` 成为 `leader`，设置状态为 `STATE_GROUP_LEADER`，其余的 `Writer` 等待 `leader` 通知状态变更。
`leader` 从链表中从后往前挑选 `Writer` 构成 `WriteGroup`，`WriteGroup` 有大小限制，最大 `1MB`，防止构建耗时太多(`EnterAsBatchGroupLeader()`)，之后合并 `WriteBatch` 写入 `WAL`、遍历 `WriteGroup` 写入 `memtable`，
最后通知其他 `Writer` 操作完成(`ExitAsBatchGroupLeader()`)。

## 通知
`RocksDB` 主要针对通知操作做了很多优化，因为发现从 `FUTEX_WAKE` 到 `FUTEX_WAIT` 返回平均需要 `10us`，延迟太大，所以 `RocksDB` 尽量不使用 `cond var` 来通知状态变更：
> For reference, on my 4.0 SELinux test server with support for syscall auditing enabled, the minimum latency between FUTEX_WAKE to returning from FUTEX_WAIT is
2.7 usec, and the average is more like 10 usec.  That can be a big drag on RocksDB's single-writer design.

每个 `Writer` 有 `std::atomic<uint8_t> state` 标记状态，也是通过它来检测状态变更。当 `leader` 执行完成后，会选出新的 `leader` 并设置 `follower` 的状态为 `STATE_COMPLETED`:
```cpp
static WriteThread::AdaptationContext eabgl_ctx("ExitAsBatchGroupLeader");
void WriteThread::ExitAsBatchGroupLeader(WriteGroup& write_group,
                                         Status status) {
  Writer* leader = write_group.leader;
  Writer* last_writer = write_group.last_writer;
  ...
    Writer* head = newest_writer_.load(std::memory_order_acquire);
    if (head != last_writer ||
        !newest_writer_.compare_exchange_strong(head, nullptr)) {
      assert(head != last_writer);

      CreateMissingNewerLinks(head);
      assert(last_writer->link_newer->link_older == last_writer);
      last_writer->link_newer->link_older = nullptr;
      SetState(last_writer->link_newer, STATE_GROUP_LEADER); // 设置 WriteGroup 后一个 Writer 为 leader
    }

    while (last_writer != leader) {
      last_writer->status = status;
      auto next = last_writer->link_older;
      SetState(last_writer, STATE_COMPLETED); // 设置 follower 完成
      last_writer = next;
    }
  ...
}
```

非 `leader` 的 `Writer` 等待状态变更分为 `3` 个阶段(`AwaitState()`):
1. `Busy loop using "pause" for 1 micro sec`
2. `Busy loop using "yield" for 100 micro sec (default)`
3. `Blocking wait`

### 1. Busy loop using "pause"
循环 `pause` 检测状态变更持续约 `1us`：
```cpp
  for (uint32_t tries = 0; tries < 200; ++tries) {
    state = w->state.load(std::memory_order_acquire);
    if ((state & goal_mask) != 0) {
      return state;
    }
    port::AsmVolatilePause();
  }
```

`asm volatile("pause")` 用于提高 `spin-wait loop` 的性能:
> Improves the performance of spin-wait loops. When executing a “spin-wait loop,” a Pentium 4 or Intel Xeon processor suffers a severe performance penalty 
> when exiting the loop because it detects a possible memory order violation. The PAUSE instruction provides a hint to the processor that the code sequence 
> is a spin-wait loop. The processor uses this hint to avoid the memory order violation in most situations, which greatly improves processor performance. For this reason, 
> it is recommended that a PAUSE instruction be placed in all spin-wait loops.

### 2. Busy loop using "yield"
如果等待时间很长或者有其他线程在竞争使用相同的核(`involuntary context switch`)时，就不适合使用 `spin-loop`，太消耗 `CPU` 资源而且利用率不高，
所以 `RocksDB` 又实现了灵活的 `spin-loop`，会在合适的时机跳出循环。

`RocksDB` 有如下配置：
* `allow_concurrent_memtable_write`: 开启 `yielding spin-loop`，能够提高吞吐。
* `write_thread_max_yield_usec`: `yielding spin-loop` 持续的最长时间，默认为 `100us`。
* `write_thread_slow_yield_usec`: 当 `yield` 调用耗时超过该值时，就认为有其他线程使用了相同的 `CPU`，默认为 `3us`，因为当没有其他线程在相同的核时，`yield` 就不会发生 `context switch`，耗时在 `1us` 内。

当 `spin-loop` 超过 `write_thread_max_yield_usec` 或者 `yield` 耗时大于 `write_thread_slow_yield_usec` 的次数达到 `kMaxSlowYieldsWhileSpinning` 次就会跳出循环
(固定 `3` 次，若是默认配置，则 `3` 次耗时 `9us` 就和使用 `cond var` 通知的平均耗时 `10us` 差不多):
```cpp
      auto spin_begin = std::chrono::steady_clock::now();
      size_t slow_yield_count = 0;

      auto iter_begin = spin_begin;
      while ((iter_begin - spin_begin) <=
             std::chrono::microseconds(max_yield_usec_)) {
        std::this_thread::yield();

        state = w->state.load(std::memory_order_acquire);
        if ((state & goal_mask) != 0) {
          would_spin_again = true;
          break;
        }

        auto now = std::chrono::steady_clock::now();
        if (now == iter_begin ||
            now - iter_begin >= std::chrono::microseconds(slow_yield_usec_)) {
          ++slow_yield_count;
          if (slow_yield_count >= kMaxSlowYieldsWhileSpinning) {
            update_ctx = true;
            break;
          }
        }
        iter_begin = now;
      }
```

除了配置作为开关以外，还有动态的开关 `yield_credit`，只有该值大于等于 `0` 时或者 `update_ctx` 为 `true` 时( `1/256` 的概率)，才会使用 `yielding spin-loop`:
```cpp
  if (max_yield_usec_ > 0) {
    update_ctx = Random::GetTLSInstance()->OneIn(sampling_base);

    if (update_ctx || yield_credit.load(std::memory_order_relaxed) >= 0) {
    ...
    }
    ...
  }
```

`yield_credit` 初始值为 `0`，会根据 `spin-loop` 的结果动态调整:
* 因慢 `yield` 跳出循环会减少。
* 若(未)超时有 `1/256` 的概率(增加)减少，因为在理想情况下，成功的概率很高，若每次都成功都增加会导致 `yield_credit` 很大，少量的超时或者慢 `yield` 带来的衰减就微不足道了。
```cpp
  if (update_ctx) {
    // Since our update is sample based, it is ok if a thread overwrites the
    // updates by other threads. Thus the update does not have to be atomic.
    auto v = yield_credit.load(std::memory_order_relaxed);
    // fixed point exponential decay with decay constant 1/1024, with +1
    // and -1 scaled to avoid overflow for int32_t
    //
    // On each update the positive credit is decayed by a facor of 1/1024 (i.e.,
    // 0.1%). If the sampled yield was successful, the credit is also increased
    // by X. Setting X=2^17 ensures that the credit never exceeds
    // 2^17*2^10=2^27, which is lower than 2^31 the upperbound of int32_t. Same
    // logic applies to negative credits.
    v = v - (v / 1024) + (would_spin_again ? 1 : -1) * 131072;
    yield_credit.store(v, std::memory_order_relaxed);
  }
```

### 3. Blocking wait
最后阶段就是用 `cond var` 来通知，但 `RocksDB` 也做了很多优化:
```cpp
// 等待
uint8_t WriteThread::BlockingAwaitState(Writer* w, uint8_t goal_mask) {
  w->CreateMutex();

  auto state = w->state.load(std::memory_order_acquire);
  assert(state != STATE_LOCKED_WAITING);
  if ((state & goal_mask) == 0 &&
      w->state.compare_exchange_strong(state, STATE_LOCKED_WAITING)) { // write-release
    std::unique_lock<std::mutex> guard(w->StateMutex());
    w->StateCV().wait(guard, [w] {
      return w->state.load(std::memory_order_relaxed) != STATE_LOCKED_WAITING;
    });
    state = w->state.load(std::memory_order_relaxed);
  }
  assert((state & goal_mask) != 0);
  return state;
}

// 通知
void WriteThread::SetState(Writer* w, uint8_t new_state) {
  auto state = w->state.load(std::memory_order_acquire); // read-acquire
  if (state == STATE_LOCKED_WAITING ||
      !w->state.compare_exchange_strong(state, new_state)) { // read-acquire
    assert(state == STATE_LOCKED_WAITING);

    std::lock_guard<std::mutex> guard(w->StateMutex());
    assert(w->state.load(std::memory_order_relaxed) != new_state);
    w->state.store(new_state, std::memory_order_relaxed);
    w->StateCV().notify_one();
  }
}
```

这里有很多细节需要注意：
* `mutex` 和 `cond var` 是 `lazy create` 的，只有第一次使用 `blocking wait` 时才会创建，使用 `aligned_storage` 是为了避免动态内存分配：
```cpp
struct Writer {
    ...
    std::aligned_storage<sizeof(std::mutex)>::type state_mutex_bytes;
    std::aligned_storage<sizeof(std::condition_variable)>::type state_cv_bytes;
    ...
    void CreateMutex() {
      if (!made_waitable) {
        made_waitable = true;
        new (&state_mutex_bytes) std::mutex;
        new (&state_cv_bytes) std::condition_variable;
      }
    }
}
```
* 通过设置 `Writer` 状态为 `STATE_LOCKED_WAITING` 保证了 `leader` 通知时 `mutex + cond var` 是已构造完成的，参看上面标记的 `acquire/release`。
* `leader` 只会设置 `follower` 状态为 `goal state`，`follower` 只会设置状态为 `STATE_LOCKED_WAITING`，所以 `compare_exchange_strong` 失败时可以采取适当的措施。
* `std::condition_variable::wait()` 等价于 `while (!pred()) wait(lock);`，保证了即使 `notify_one()` 丢失也不会一直等待。
