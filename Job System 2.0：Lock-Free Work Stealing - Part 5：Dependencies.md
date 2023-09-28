*本文档译自 blog.molecular-matters.com 的 "Job System 2.0" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
在今天的文章中，我们将讨论新工作系统的最后一部分：增加工作之间的依赖关系。


## 这个系列的其他文章 - *Other posts in the series*
---
[Part 1](https://blog.molecular-matters.com/2015/08/24/job-system-2-0-lock-free-work-stealing-part-1-basics/)：解释了新的作业系统的基本设计，和对“作业窃取”的简要说明。
[Part 2](https://blog.molecular-matters.com/2015/09/08/job-system-2-0-lock-free-work-stealing-part-2-a-specialized-allocator/)：对线程本地分配的机制细节作深入讨论。
[Part 3](https://blog.molecular-matters.com/2015/09/25/job-system-2-0-lock-free-work-stealing-part-3-going-lock-free/)：讨论作业窃取队列的无锁实现。
[Part 4](https://blog.molecular-matters.com/2015/11/09/job-system-2-0-lock-free-work-stealing-part-4-parallel_for/)：介绍了高级算法。


## 保持简单 - *Keep it simple*
---
对于新的工作系统，我希望它与之前 1.0 版本中的方法相比，降低了实现的复杂性。当然，无锁窃取工作队列的实现也确实相当复杂，但是这段代码是自成一体的，这使得其余代码相对容易理解。

之前，向作业系统引入依赖关系总是意味着必须引入第二个队列（用于存储尚未执行的作业），必须重新计算现有的函数，并且必须重新编写一些代码，以便考虑依赖关系。

这一次，我想做尽可能简单的事情：通过连接的方式让用户显式地声明依赖关系。

我们现在明确地规定，只要一个作业完成，就立即运行它的所有后续作业，而不是说每当这个作业完成时，就尝试运行可能正在等待它的任何其他作业。

在这种情况下，立即运行作业意味着将它们推入作业队列，这样负载平衡就可以接管并确保所有核心/线程都能平等地完成工作。

在代码中，这意味着我们现在要执行以下操作：

```C++
AddContinuation(ancestor, dependency1);
AddContinuation(ancestor, dependency2);
```

实现这一点的最简单的解决方案是将所有的连接直接作为数据存储在 *Job* 结构中。这就是我们要做的！


## 支持连接 - *Supporting continuations*
---
将 *Job* 结构体的大小从 64 字节增加到 128 字节，允许我们存储以下内容：

```C++
struct Job
{
	JobFunction function;
	Job* parent;
	int32_t unfinishedJobs;
	char data[52];
	int32_t continuationCount; // new
	Job* continuations[15];    // new
};
```

在产品代码中，我们可以节省一些位置，因为我们不需要存储指针，而是可以存储偏移量。即使有偏移量，我们可能也不需要一个完整的 16 位偏移量。

这使得每个作业只能支持 52 字节的数据和 16 个连接。也可以分配更多的数据空间，减少连接。我们甚至可以合并数据和连接数组，动态地存储数据，并在其中添加一些运行时断言。

向现有作业添加连接非常简单：

```C++
void AddContinuation(Job* ancestor, Job* continuation)
{
	const int32_t count = atomic::Increment(&ancestor->continuationCount);
	ancestor->continuations[count - 1] = continuation;
}
```

当然，我们需要确保父作业尚未运行 `Run()`，并且 `continuation` 数组中仍有空间。请注意，必须原子地增加 `continuationCount` 的数量，以确保在其他线程试图添加其连接的情况下不会引入数据竞争。

调整作业系统以支持连续也很简单，我们只需要稍微修改 `Finish()` 函数：

```C++
void Finish(Job* job)
{
	[...]
	if (job->parent)
	{
	    Finish(job->parent);
	}
	
	// run follow-up jobs
	for (int32_t i = 0; i < job->continuationCount; ++i)
	{
	    PushToQueue(job->continuations[i]);
	}
	[...]
}
```

基本上就是这样，坚持 *KIS*S 原则。与旧方法相比，新的实现有一些优点：

+ 实现更简单。
+ 从属关系健壮，一切都是开箱即用。
+ 不再需要（可能）无限的堆栈空间，所有内存都是预先分配的。

当然，每个 *Job* 现在需要两倍的内存，但在甚至手机都提供 *512MB* 内存的时代，我不认为这是一个大问题。此外，对于使用 128 字节高速缓存行的体系结构，我们希望无论如何都要提高大小，以避免伪共享。


## 总结 - *Conclusion*
---
最后，这一部分是作业系统 2.0 系列的总结。希望你喜欢！