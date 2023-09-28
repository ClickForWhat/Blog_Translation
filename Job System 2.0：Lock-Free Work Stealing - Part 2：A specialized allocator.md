*本文档译自 blog.molecular-matters.com 的 "Job System 2.0" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
正如在上一篇文章中所承诺的，今天我们将看看如何在分配任务系统中的任务时摆脱 `new` 和 `delete`。只要我们愿意为此牺牲一些内存，就可以用一种更有效的方式来处理分配。由此带来的性能提升是巨大的，当然是值得的。


## 为什么new和delete是低效的 - *Why using new and delete is slow*
---
如前所述，`new` 基本上在后台调用一个通用分配器。该分配器必须同时满足小、中和（非常）大的分配。此外，`new` 本身就是线程安全的，它使用互斥锁或临界区等同步原语来保护可变的共享数据不受竞争的影响。

因此，在像我们这样的特殊情况下，我们能够击败 `new` 和 `delete` 的性能也就不足为奇了。尽管通用分配器的性能和特性不断得到改进，但它们永远无法击败定制的分配器。

还要记住，使用 `new` 分配实例会迫使我们在这些实例上调用 `delete`，以避免内存泄漏。这反过来又限制了我们将作业的删除推迟到帧的末尾，并将它们存储在辅助数组中。此外，在 `Finish()` 函数中还需要另一个原子计数器。


## 一个池分配器？ - *A pool allocator?*
---
我们的作业系统只进行 `job` 类型的分配，所以使用[池分配器/自由列表](https://blog.molecular-matters.com/2012/09/17/memory-allocation-strategies-a-pool-allocator/)听起来非常合适。使用（无锁）池分配器确实提高了性能，但仍然不能解决必须推迟删除作业实例的问题。


## 一个特殊的分配器 - *A specialized allocator*
---
最重要的是要意识到我们在每一帧都在重复同样的事情。我们生成并分配 *N* 个作业，并在一帧结束时删除所有的 *N* 个作业。那么，为什么不完全放弃分配和释放工作呢？

这可以通过使用预先分配的 *Job* 实例的全局数组轻松完成。因为 *Job* 结构是 *POD* 类型，所以不必担心调用构造函数或析构函数。我们可以在初始化作业系统时分配一个包含4096个作业的数组，将该数组用作环形缓冲区，用于处理分配请求，并在完全关闭作业系统时释放该数组，例如在应用程序退出时。

我们所需要的只是一个全局的 *Job* 数组，和一个原子计数器，它可以作为一种无锁的环形缓冲区分配器，如果你想这样称呼它的话。然后 `Allocate()` 函数简单地变成：

```C++
static Job g_jobAllocator[MAX_JOB_COUNT];
static uint32_t g_allocatedJobs = 0u;
 
Job* AllocateJob(void)
{
	const uint32_t index = atomic::Increment(&g_allocatedJobs);
	return &g_jobAllocator[(index-1) % MAX_JOB_COUNT];
}
```

正如你可能知道的那样，模数表达式  `(index-1) % MAX_JOB_COUNT` 可以转换为二进制与运算，如果 `MAX_JOB_COUNT` 是2的幂，例如4096：

```C++
return &g_jobAllocator[(index-1u) & (MAX_JOB_COUNT-1u)];
```

可以看到，原子计数器是一个单调递增的整数，永远不会重置，因此我们使用模数操作访问全局  `g_jobAllocator` 数组，有效地将其转换为环缓冲区。在产品代码中，你应该确保添加额外的措施，以确保在单个帧中分配的作业不超过 `MAX_JOB_COUNT`。此外，你可以 `memset()` 拷贝 *Job* 实例（或对其进行时间戳），以确保代码不会触及过去帧中的条目。

使用这种方法需要注意的一件重要事情是，我们不再需要在一个帧期间删除作业。这意味着我们可以去掉全局辅助数组和相应的原子计数器，从而简化了 `Finish()` 函数的实现：

```C++
void Finish(Job* job)
{
	const int32_t unfinishedJobs = atomic::Decrement(&job->unfinishedJobs);
	
	if ((unfinishedJobs == 0) && (job->parent))
	{
	    Finish(job->parent);
	}
}
```

这种方法所需的预分配内存量在总体规划中可以忽略不计。每帧4096个作业，数组需要 4096 \* 64 = 256 KB，这在今天的平台上根本不算什么。

尽管这个实现在性能方面比原来的方法有了很大的提高，但我们仍然可以做得更好。热心的读者现在应该知道接下来会发生什么。


## 线程本地化 - *Going thread-local*
---
那么，如何才能比每次分配只使用一个原子操作做得更好呢?当然，那就是根本不使用原子操作！原子操作比互斥锁之类的操作便宜得多，但它们也[不是免费的](https://fgiesen.wordpress.com/2014/08/18/atomics-and-contention/)。

通过将计数器和预分配的作业数组移动到线程本地存储，我们不再需要任何原子操作来从环形缓冲区分配作业：

```C++
Job* AllocateJob(void)
{
	const uint32_t index = g_allocatedJobs++;
	return &g_jobAllocator[index & (MAX_JOB_COUNT - 1u)];
}
```

在上面的示例代码中，在线程本地存储上分配了 `g_allocatedJobs` 和 `g_jobAllocator`。注意，这里没有任何原子操作或昂贵的函数调用。这是最便宜的了。

还有一点需要注意的是，与之前的方法相比，这需要更多的内存，因为所有工作线程现在都需要一个预先分配 `MAX_JOB_COUNT_JOB` 大小的实例数组。在8个工作线程的情况下，这将使内存需求从 256 KB 增加到2 MB，我仍然认为这是一个小数目，至少在8核机器上是这样。


## 性能 - *Performance*
---
同样，性能是在 3.4 GHz 的 Intel 酷睿 i7-2600K CPU上测量的，具有4个具有超线程的物理内核(= 8个逻辑内核)。通过改进的实现，现在的运行时间如下所示：

|   | Old | **New** | Perf. increase |
|---|---|---|---|
|Single jobs|18.5 ms|**9.9 ms**|1.86x|
|parallel_for|5.3 ms|**1.35 ms**|3.92x|


## 展望 - *Outlook*
---
下一次，我们将讨论作业系统的核心部分：无锁工作窃取队列的实现。