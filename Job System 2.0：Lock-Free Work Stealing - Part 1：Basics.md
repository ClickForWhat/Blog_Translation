*本文档译自 blog.molecular-matters.com 的 "Job System 2.0" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
回到2012年，我在 *Mocular* 中写过关于实现任务调度器的文章。如今3年过去了，现在是时候给旧系统一个长久以来应得的提升。

新 *Job System* 的要求如下：

+ 基本实现需要更简单。*Job* 本身需要相当简洁，但可以允许我们可以在实现之上构建高级算法，例如 `parallel_for`。
+ *Job System* 需要实现自动负载均衡。
+ 在适用的情况下，应逐步用无锁的替代品替换系统的部分部件，从而提高性能。
+ 系统需要支持动态并行性：必须能够在作业仍在运行时更改依赖关系并向其添加依赖项。这需要允许高级原语（如 `parallel_for`）动态地将给定的工作负载拆分为更小的作业。

今天，我们将在关键部分使用锁来研究新作业系统的基本实现。即使在使用锁的时候，在实现无锁之前，我也要指出一些陷阱。


## 基本知识 - *The very basics*
---
与旧的任务调度器类似，我们的新 *Job System* 基本工作如下：

+ 拥有 *N* 个工作线程，它们可以不断地从队列中获取作业并执行它。
+ 有 *N* 个物理核心，就创建 *N*-1 个工作线程。
+ 主线程也被认为是工作线程，可以帮助执行作业。

这一次有一个主要的不同：我们的 *Job System* 现在实现了一个称为[“作业窃取”](https://en.wikipedia.org/wiki/Work_stealing)的概念，这意味着每个工作线程都有自己的作业队列，而不是使用一个全局队列将所有作业推入其中。使用全局作业队列会产生很多争用，特别是当涉及多个线程时。

作业窃取是一个简单但有效的概念：

+ 新作业总是被推送到调用线程的队列中。
+ 每当工作线程想要处理一个作业时，它首先尝试从自己的队列中弹出一个作业。如果队列中没有作业，线程将尝试从其他工作线程的队列中窃取作业。
+ `Push()` 和 `Pop()` 操作只能由拥有此队列的工作线程调用。
+ 操作 `Steal()` 只能由不拥有此队列的工作线程调用。

最后两项很重要，并导致以下观察结果：

+ `Push()` 和 `Pop()` 可以在队列的一端（私有端）工作，而 `Steal()` 可以在另一端（公共端）工作。
+ 私有端可以以后进先出（*LIFO*）的方式工作，以更好地利用缓存，而公共端以先进先出（*FIFO*）的方式工作，以更好地平衡工作。

当谈到作业窃取时，这种双端数据结构通常被称为 **作业窃取队列**。这种作业窃取队列的一个非常重要的好处是可以以无锁的方式实现它。

在 C++ 中，基本实现如下所示：

```C++
// main function of each worker thread
while (workerThreadActive)
{
	Job* job = GetJob();
	if (job)
	{
		Execute(job);
	}
}
 
Job* GetJob(void)
{
	WorkStealingQueue* queue = GetWorkerThreadQueue();
	
	Job* job = queue->Pop();
	if (IsEmptyJob(job))
	{
	    //此job是空的，不合法。这意味着此作业队列已经空了，因此尝试从其他作业队列窃取作业
	    unsigned int randomIndex = GenerateRandomNumber(0, g_workerThreadCount+1);
	    WorkStealingQueue* stealQueue = g_jobQueues[randomIndex];
	    if (stealQueue == queue)
	    {
		    //防止偷到自己的队列
		    Yield();
		    return nullptr;
	    }
		
	    Job* stolenJob = stealQueue->Steal();
	    if (IsEmptyJob(stolenJob))
	    {
		    //若我们没能从另一个队列中窃取工作，所以现在只能让出我们的时间片
		    Yield();
		    return nullptr;
	    }
	    
	    return stolenJob;
	}
	
	return job;
}
 
void Execute(Job* job)
{
	(job->function)(job, job->data);
	Finish(job);
}
```

在这一点上，我故意省略了 `Finish()` 函数的细节，它将在后面讨论。


## 什么是 Job ？ - *What is a job?*
---
遵循“保持简单”的要求，作业需要存储至少两样东西：一个指向需要被执行的函数的指针，以及一个可选的父作业。

此外，我们还需要一个计数器，用于跟踪[处理依赖关系](https://blog.molecular-matters.com/2012/04/25/building-a-load-balanced-task-scheduler-part-3-parent-child-relationships/)的未完成作业的数量。为了避免[伪共享](https://blog.molecular-matters.com/2012/07/09/building-a-load-balanced-task-scheduler-part-4-false-sharing/)，我们添加了填充以确保 `Job` 对象至少占用一整条缓存线：

```C++
struct Job
{
	JobFunction function;
	Job* parent;
	int32_t unfinishedJobs; // atomic
	char padding[];
};
```

注意，`unfinishedJobs` 成员被标记为原子变量。在 *Molecule* 中，它可以通过使用 *Windows* 上的 `Interlocked*` [函数](https://msdn.microsoft.com/en-us/library/windows/desktop/ms684122(v=vs.85).aspx)来改变。在 C++ 11 中，可以使用 `std::atomic`。我还省略了填充数组的大小，因为它对于32位和64位是不同的，并且由于涉及到几个 `sizeof()` 操作符而使代码变得混乱，但复杂性并不显著。

与作业关联的作业函数接受两个参数：它所属的作业和与作业关联的数据。

```C++
typedef void (*JobFunction)(Job*, const void*);
```


## 与作业关联的数据 - *Associating data with a job*
---
我不喜欢旧的任务调度器的一点是，用户必须持有作业的数据，直到作业完成。当作业数据可以存储在栈上时，这不是一个问题，但它有时会从堆中进行不必要的分配，我希望在新系统中消除这些分配。

幸运的是，有一种简单的解决方案可以存储属于作业的数据：我们可以将其存储在 *Job* 结构中。

那个填充空隙的数组是存储作业数据的理想选择。这个数组没有被用来存东西，而且无论如何我们都需要它，所以为什么不好好利用它呢？在 *Molecule* 中，只要给定的数据符合编译时检查的要求，与作业相关的数据就会被 `memcpy` 写入填充数组。如果数据太大而无法就地存储，用户总是可以从堆中分配数据，并将指向数据的指针传递给作业系统。


## 添加作业 - *Adding jobs*
---
将作业推送到系统中总是分两步完成：首先，创建作业。其次，将作业添加到系统中。将这个操作分成两个部分允许我们实现动态并行性，这是前面提到的需求之一。

在C++中，使用以下函数之一创建作业：

```C++
Job* CreateJob(JobFunction function)
{
	Job* job = AllocateJob();
	job->function = function;
	job->parent = nullptr;
	job->unfinishedJobs = 1;
	
	return job;
}
 
Job* CreateJobAsChild(Job* parent, JobFunction function)
{
	atomic::Increment(&parent->unfinishedJobs);
	
	Job* job = AllocateJob();
	job->function = function;
	job->parent = parent;
	job->unfinishedJobs = 1;
	
	return job;
}
```

请注意，我省略了接受附加数据的函数，这些数据被写入填充数组。

目前，`AllocateJob()` 只是通过调用 `new` 来分配并返回一个新的 *Job* 对象。可以看到，当创建一个作业作为已存在作业的子作业时，父作业的 `unfinishedJobs` 成员将自动递增。这需要原子地完成，因为其他线程可能会将不同的作业作为子作业添加到同一作业中，从而导致数据竞争。

将新创建的作业添加到系统通过调用 `Run()` 来完成：

```C++
void Run(Job* job)
{
	WorkStealingQueue* queue = GetWorkerThreadQueue();
	queue->Push(job);
}
```


## 等待作业 - *Waiting for a job*
---
当然，一旦我们向系统添加了一些作业，我们需要能够检查它们是否已经完成，并在此期间做一些其他有意义的事情。这是通过调用 `Wait()` 实现的：

```C++
void Wait(const Job* job)
{
	// wait until the job has completed. in the meantime, work on any other job.
	while (!HasJobCompleted(job))
	{
	    Job* nextJob = GetJob();
	    if (nextJob)
	    {
		    Execute(nextJob);
	    }
	}
}
```

可以通过比较 `unfinishedJobs` 与0来确定作业是否已完成。如果计数器大于0，则作业本身或其任何子作业到目前为止尚未完成。如果计数器为零，则所有相关作业都已完成。


## 实践中的系统 - *The system in practice*
---
下面的简单示例创建了一堆添加到系统中的单个空作业：

```C++
void empty_job(Job*, const void*)
{
}
 
for (unsigned int i = 0; i < N; ++i)
{
	Job* job = jobSystem::CreateJob(&empty_job);
	jobSystem::Run(job);
	jobSystem::Wait(job);
}
```

当然，这是低效的，因为我们单独地创建、运行和等待每个作业。尽管如此，这仍然是衡量作业系统创建、添加和运行作业的开销的一个很好的测试。

另一个示例再次创建单个作业，但将它们作为一个根作业的子作业运行：

```C++
Job* root = jobSystem::CreateJob(&empty_job);
for (unsigned int i = 0; i < N; ++i)
{
	Job* job = jobSystem::CreateJobAsChild(root, &empty_job);
	jobSystem::Run(job);
}
jobSystem::Run(root);
jobSystem::Wait(root);
```

这样效率更高，因为创建和运行作业现在是与执行系统中已有的作业并行进行的。


## 完成并删除作业 - *Finishing and deleting jobs*
---
我们快做完了。不过，我们还需要通知它们的父任务”子任务已经完成！“，来正确地结束任务。我们需要删除我们分配的所有作业。你可能想这样编写 `Finish()` 函数：

```C++
void Finish(Job* job)
{
	const int32_t unfinishedJobs = atomic::Decrement(&job->unfinishedJobs);
	if (unfinishedJobs == 0)
	{
	    if (job->parent)
	    {
		    Finish(job->parent);
	    }
	    delete job;
	}
}
```

我们首先原子地减少 `unfinishedJobs` 计数器。如前所述，只要这个计数器达到0，这个作业和它的所有子任务都完成了，所以我们需要告诉父任务。之后，我们可以删除作业，因为我们不再需要它。然而，有一个不那么微妙的错误，你能发现它吗？

问题是，此时不允许我们删除作业。仍然可能有线程在等待这个作业，它正打算调用 `HasJobCompleted()` 来检查这个作业是否已经完成。这将导致线程访问不再属于该进程的内存，从而导致访问冲突或读取垃圾值。一种解决方案是将作业的删除推迟到稍后的时间点，但你仍然必须小心这一点：

```C++
void Finish(Job* job)
{
	const int32_t unfinishedJobs = atomic::Decrement(&job->unfinishedJobs);
	if (unfinishedJobs == 0)
	{
	    const int32_t index = atomic::Increment(&g_jobToDeleteCount);
	    g_jobsToDelete[index - 1] = job;
		
		if (job->parent)
	    {
		    Finish(job->parent);
	    }
	    atomic::Decrement(&job->unfinishedJobs);
	}
}
```

请注意，在将作业添加到全局数组并通知父级之后，此代码将再次减少计数器。现在， `unfinishedJobs` 表示作业完成的值为-1而不是0。在根作业完成执行后，可以安全地删除为该帧分配的所有作业。

在这种特殊情况下，将 `unfinhedjobs` 设置为-1而不使用原子指令是安全的，但是代码需要额外的编译器屏障（以及其他平台上的内存屏障）才能正确。


## 实施细节 - *Implementation details*
---
关于如何实现这篇文章中提到的一些事情的一些注意事项：

+ 访问工作线程队列容易通过使用线程本地索引来完成。在 *Windows/MSVC* 上，这可以通过使用 [`__declspec(thread)`](https://msdn.microsoft.com/en-us/library/9w1sdazb.aspx) 或 [`TlsAlloc`](https://msdn.microsoft.com/en-us/library/windows/desktop/ms686801(v=vs.85).aspx) 来实现。
+ 让出线程的时间片可以通过使用 [`_mm_pause`、`Sleep(1)`、`Sleep(0)` 或其他变体](https://randomascii.wordpress.com/2012/06/05/in-praise-of-idleness/)来完成。你应该始终确保工作线程在无事可做时不会消耗100%的CPU时间。事件、信号量或条件变量可用于此目的。


## 性能 - *Performance*
---
使用上面描述的作业系统，我进行了两个测试来测试系统的性能和开销。第一个测试创建65000个独立运行的空作业，如上面的示例所示。第二个测试也创建了65000个作业，但是使用了几个 `parallel_for` 循环，这些循环递归地将它们的工作分解为更小的作业。

性能是在 3.4 GHz 的 *Intel* 酷睿 *i7-2600K* CPU 上测量的，有4个具有超线程的物理内核（= 8个逻辑内核）。

运行时间如下：
```
Single jobs：   18.5 ms
parallel_for：  5.3 ms
```

有几件事值得注意：

+ 使用 `parallel_for` 更能代表实际的工作负载，因为可以并行地创建和添加作业。
+ 作业系统使用 `new` 和 `delete` 来分配作业，效率并不高。
+ 作业窃取队列的实现使用锁。


## 展望 - *Outlook*
---
下一次，我们将研究如何消除 `new` 和 `delete`，从而简化过程中的 `Finish()` 函数。之后，我们将处理工作窃取队列的无锁实现。最后但并非最不重要的是，我们将看看如何实现高级算法，如并行使用这个作业系统。


## 免责声明 - *Disclaimer*
---
本文假设采用 x86 架构和强内存模型。如果你不了解底层含义，那么在其他平台上工作时，最好使用 C++ 11 和具有顺序一致性的 `std::atomic`。