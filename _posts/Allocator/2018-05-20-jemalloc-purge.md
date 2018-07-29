---
title: jemalloc purge改进
layout: post
excerpt: 介绍 4.0.3 版本以后，jemalloc 对 purge 的改进。
categories: Allocator
---

{% include toc %}

`Redis` 之前一直使用 `4.0.3` 版本的 `jemalloc`，对 `dirty pages` 的回收比较暴力，相当于 `stop the world`。我们线上 `Redis` 偶尔也会出现 `RSS` 瞬间降低 `1GB`
并伴随大量报错的场景，之前看 `jemalloc` 源码也是为了解决这个问题。

最近(2018-05-24)，`Redis` 把 `jemalloc` 升级到了 `5.0.1` 版本，这次主要看一下对 `purge` 的改进。

## decay-based purge
`4.0.3` 是根据当前 `active_pages : dirty_pages` 的比例进行 `purge`，比如 `lg_dirty_mult:3`，当 `active_parges / dirty_pages < 8` 时，就会把多余的 `purge` 掉，当比例在 `8` 波动时，就会频繁 `purge`，造成性能下降。  

`4.1.0` 新增了 `decay-based purge`，不再根据比例进行 `purge`，而是配置 `dirty page` 的存在时间，在这段时间内按照 [Smootherstep](https://en.wikipedia.org/wiki/Smoothstep) 平滑的进行 `purge(1->0)`:
![image](/assets/images/Smoothstep_and_Smootherstep.png)

若使用 `decay-based purge`，`jemalloc` 在 `decay_time`(默认10s) 时间内分段(200次)进行 `dirty pages` 的回收，避免突发的大量 `dirty page` 回收影响性能。因为在 `purge` 的过程中，
会有新的 `dirty page` 产生，所以将整个 `purge` 划分为 `SMOOTHSTEP_NSTEPS(200)` 组，每组分别负责 `decay_interval(decay_time / SMOOTHSTEP_NSTEPS)` 时间内产生的 
`dirty page` 的回收，每组能保留的 `dirty pages` 数量都是 `Smootherstep` 曲线，总的能保留的 `dirty page` 数量为 `200` 组的叠加，超出的会 `purge`。

### 实现
`arena` 中新增了如下字段：
```c
/*
 * Approximate time in seconds from the creation of a set of unused
 * dirty pages until an equivalent set of unused dirty pages is purged
 * and/or reused.
 */
ssize_t			decay_time;	// 产生的 dirty pages 会在 decay_time 时间后全部 purge，默认为 10s。
/* decay_time / SMOOTHSTEP_NSTEPS. */
nstime_t		decay_interval; // decay 分为 SMOOTHSTEP_NSTEPS(200) 组，这是每组的时间间隔。
/*
 * Time at which the current decay interval logically started.  We do
 * not actually advance to a new epoch until sometime after it starts
 * because of scheduling and computation delays, and it is even possible
 * to completely skip epochs.  In all cases, during epoch advancement we
 * merge all relevant activity into the most recently recorded epoch.
 */
nstime_t		decay_epoch; // 每组 decay 的起始时间
/* decay_deadline randomness generator. */
uint64_t		decay_jitter_state; // 随机种子，用于计算下面的 decay_deadline
/*
 * Deadline for current epoch.  This is the sum of decay_interval and
 * per epoch jitter which is a uniform random variable in
 * [0..decay_interval).  Epochs always advance by precise multiples of
 * decay_interval, but we randomize the deadline to reduce the
 * likelihood of arenas purging in lockstep.
 */
nstime_t		decay_deadline; // 每组 decay 的终止时间
/*
 * Number of dirty pages at beginning of current epoch.  During epoch
 * advancement we use the delta between decay_ndirty and ndirty to
 * determine how many dirty pages, if any, were generated, and record
 * the result in decay_backlog.
 */
size_t			decay_ndirty; // 每组 decay 开始时的 arena->ndirty(dirty page 数量)
/*
 * Memoized result of arena_decay_backlog_npages_limit() corresponding
 * to the current contents of decay_backlog, i.e. the limit on how many
 * pages are allowed to exist for the decay epochs.
 */
size_t			decay_backlog_npages_limit; // 每次 purge 能够剩下的 dirty page 数量
/*
 * Trailing log of how many unused dirty pages were generated during
 * each of the past SMOOTHSTEP_NSTEPS decay epochs, where the last
 * element is the most recent epoch.  Corresponding epoch times are
 * relative to decay_epoch.
 */
size_t			decay_backlog[SMOOTHSTEP_NSTEPS]; // 倒序(最后的为最新的 decay)保存了每组 decay 产生的 dirty page 数量，
                                                    // 根据这个计算能够保存的 dirty page 数量。
```

新创建 `arena` 时，会调用 `arena_decay_init()` 设置 `decay` 初始状态：
* `decay_time`：配置的值，产生的 `dirty pages` 会在 `decay_time` 时间后全部 `purge`，默认为 `10s`。
* `decay_interval`：`decay_time / SMOOTHSTEP_NSTEPS`，`decay` 分为 `SMOOTHSTEP_NSTEPS(200)` 组，这是每组的时间间隔。
* `decay_epoch`：每组 `decay` 的起始时间，设置为当前时间。
* `decay_jitter_state`：产生 `decay_deadline` 的随机种子，以 `arena` 地址为值。
* `decay_deadline`：随机产生在 `[decay_epoch + decay_interval, decay_epoch + 2*decay_interval)` 间。
* `decay_ndirty`: 设置为 `arena->ndirty`，每组 `decay` 开始时 `arena` 的 `dirty page` 个数。
* `decay_backlog_npages_limit`：这次 `purge` 之后能够保留的 `dirty page` 数量。
* `decay_backlog[SMOOTHSTEP_NSTEPS]`：初始化为0，保存每组 `decay` 产生的 `dirty page` 数量。

因为按照时间进行 `purge`，所以需要时钟来触发 `decay purge`。`jemalloc` 使用内存分配和释放的次数作为时钟，当次数到达 `1000` 次时，尝试进行一次 `decay purge`:
```c
JEMALLOC_ALWAYS_INLINE void
arena_decay_ticks(tsd_t *tsd, arena_t *arena, unsigned nticks)
{
	ticker_t *decay_ticker;

	if (unlikely(tsd == NULL))
		return;
	decay_ticker = decay_ticker_get(tsd, arena->ind);
	if (unlikely(decay_ticker == NULL))
		return;
        // 当 tick 为0时，才 purge，默认是1000次
	if (unlikely(ticker_ticks(decay_ticker, nticks)))
		arena_purge(arena, false);
}
```

之后会调用 `arena_maybe_purge_decay()` 尝试 `decay purge`:
```c
static void
arena_maybe_purge_decay(arena_t *arena)
{
	nstime_t time;
	size_t ndirty_limit;

	assert(opt_purge == purge_mode_decay);

	/* Purge all or nothing if the option is disabled. */
	if (arena->decay_time <= 0) {
		if (arena->decay_time == 0)
			arena_purge_to_limit(arena, 0);
		return;
	}

	nstime_copy(&time, &arena->decay_epoch);
	if (unlikely(nstime_update(&time))) { // 设置 time 为当前时间
		/* Time went backwards.  Force an epoch advance. */
		nstime_copy(&time, &arena->decay_deadline); // 当时间回退时，设置时间为 deadline
	}

	if (arena_decay_deadline_reached(arena, &time)) // arena->decay_deadline <= time 到达 deadline
		arena_decay_epoch_advance(arena, &time);  // 更新 epoch 和 统计

	ndirty_limit = arena_decay_npages_limit(arena); // 计算这次 purge 能够剩下的 dirty page 数量

	/*
	 * Don't try to purge unless the number of purgeable pages exceeds the
	 * current limit.
	 */
	if (arena->ndirty <= ndirty_limit)
		return;
	arena_purge_to_limit(arena, ndirty_limit); // 进行 purge，purge 之后剩下的数量为 ndirty_limit
}
```

最关键的为 `arena_decay_epoch_advance()`：
```c
static void
arena_decay_epoch_advance(arena_t *arena, const nstime_t *time)
{
	uint64_t nadvance;
	nstime_t delta;
	size_t ndirty_delta;

	assert(opt_purge == purge_mode_decay);
	assert(arena_decay_deadline_reached(arena, time));

	nstime_copy(&delta, time);
	nstime_subtract(&delta, &arena->decay_epoch);
	nadvance = nstime_divide(&delta, &arena->decay_interval); // 计算过去了多少 interval
	assert(nadvance > 0);

	/* Add nadvance decay intervals to epoch. */
	nstime_copy(&delta, &arena->decay_interval);
	nstime_imultiply(&delta, nadvance);
	nstime_add(&arena->decay_epoch, &delta); // 设置 decay_epoch 为这次 decay 的起始时间

	/* Set a new deadline. */
	arena_decay_deadline_init(arena);

	/* Update the backlog. */
        /* arena->decay_backlog[] 保存了每组 decay 产生的 dirty page 数量，最后的为最新的(因为 smootherstep 是从0~1)
         * 当过去 interval 时，decay_backlog[] 中的数据会左移，相当于时间窗口移动
         */
	if (nadvance >= SMOOTHSTEP_NSTEPS) {
		memset(arena->decay_backlog, 0, (SMOOTHSTEP_NSTEPS-1) *
		    sizeof(size_t));
	} else {
		memmove(arena->decay_backlog, &arena->decay_backlog[nadvance],
		    (SMOOTHSTEP_NSTEPS - nadvance) * sizeof(size_t));
		if (nadvance > 1) {
			memset(&arena->decay_backlog[SMOOTHSTEP_NSTEPS -
			    nadvance], 0, (nadvance-1) * sizeof(size_t));
		}
	}
        // 这次 decay 时间间隔内产生的 dirty page 数量
	ndirty_delta = (arena->ndirty > arena->decay_ndirty) ? arena->ndirty -
	    arena->decay_ndirty : 0;
	arena->decay_ndirty = arena->ndirty; // 保留当前的 dirty page 数量，为了下一次计算 ndirty_delta
	arena->decay_backlog[SMOOTHSTEP_NSTEPS-1] = ndirty_delta; // 保存这次 decay 产生的 dirty page 数量
	arena->decay_backlog_npages_limit =
	    arena_decay_backlog_npages_limit(arena); // 根据 smootherstep 计算这次 decay 之后能够剩下的 dirty page 数量
}
```

## tow-phase decay purge
`jemalloc 4.0.3` 版本在 `Linux` 系统下是调用 `madvise(..., MADV_DONTNEED)` 来回收内存，这是强制的、立即生效的(`RSS` 立即释放)，所以 `overhead` 会比较大。

`Linux 4.5` 之后支持 `MADV_FREE`，内存不再立即回收，当操作系统有内存压力后才会回收，`overhead` 很小。`jemalloc 5.0.0` 实现了 `two-phase purge`：
* `page` 分为 `3` 种状态：`dirty -> muzzy -> clean`
* `purge` 分为 `2` 个阶段：
    * `dirty -> muzzy`: 调用 `madvise(..., MADV_FREE)` 由操作系统释放
    * `muzzy -> clean`: 调用 `madvise(..., MADV_DONTNEED)` 强制释放
* `2` 个阶段均使用 `decay-based purge`，分别设置配置。可以设置 `lazy-purging` 的时间短(快速回收)，`fored-purging` 的时间长(慢速回收)。

## background thread
`jemalloc 5.0.0` 实现了 `asynchronous decay-driven unused dirty page purging`，开启之后，最多创建 `CPU` 个数的线程，并设置 `CPU` 亲和力，`decay-based purging` 会由 `background thread` 来执行，
不影响正常的工作。

## 结论
高版本 `jemalloc` 对 `purge` 进行了很多优化，能够减少内存回收的影响。
