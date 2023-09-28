*本文档译自 blog.molecular-matters.com 的 "Task Scheduler" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
随着多核硬件在基于PC/主机的游戏和移动平台上成为常态，利用每一个处理器核心和线程都变得至关重要。因此，构建技术来减轻编写正确的多线程代码的负担相当关键。

为了实现这一点，本系列将尝试解释如何构建负载平衡、作业窃取的任务调度器。

与手动执行代码线程相比，自动任务调度具有某些好处：

+ 更好的可扩展性，更容易通过自动负载平衡利用 *N* 个核或线程。
+ 更不容易出错的是，依赖关系可以表示为任务之间简单的父子关系。
+ 更容易从函数和数据并行性中获益。

在我看来，你可以从任务调度器中获得的最大好处是增加了常见的、每帧都有的任务的并行性（例如动画更新，碰撞检测，渲染）。另一方面，需要几个帧才能完成的永久性任务，如几何体流送、音频等，最好手工完成。


## 最简单的调度器 - *The simplest scheduler*
---
在我们开始处理更高级的技术（如负载平衡和工作窃取）之前，让我们尝试构建一个真正简单的调度器，它只不过是将独立的任务委托给多个核心。

任务调度器的基本思想很简单：

+ 为系统中每个可用的核心或硬件线程创建一个工作线程。每个工作线程帮助执行交给调度器的任务。
+ 将任务放入队列中，每个工作线程从该队列中提取待处理的任务。

在固定硬件（例如主机）上，你可以确保创建尽可能多的线程，以确保系统中不会出现线程过多或不足的情况。此外，你可以为每个工作线程分配固定的硬件线程，这使得性能在某种程度上更可预测。在 *PS3* 上，每个可用的 *SPU* 都可以被视为一个工作线程。

在像 *PC* 这样的其他平台上，实际上创建 `num_core` 或 `num_core - 1` 个线程之间并没有太大的区别，因为操作系统上下文切换无论如何都无法预测或控制。

在伪代码中，调度器实现如下所示：

```C++
// at startup:
CreateWorkerThreads(numAvailableHardwareThreads);
 
for each (WorkerThread)
{
	WorkOnTasks();
}
 
// submit work during a frame:
for each 1000 particles
{
	AddTaskToScheduler(particleData + N * 1000, 1000);
}
 
// ...
// do other work in between, and finally synchronize
WaitUntilTasksAreFinished();
```

我们现在不关心高级特性，`AddTaskToScheduler()` 只是将任务添加到全局队列中，可以这样实现：

```C++
AddTaskToScheduler(task)
{
	LockSynchronizationPrimitive();
	globalTaskQueue.Add(task);
	UnlockSynchronizationPrimitive();
}
```

剩下的是 `WorkOnTasks()` 函数，它由每个工作线程运行：

```C++
while (threadShouldRun)
{
	WaitUntilTaskIsAvailable();
	 
	LockSynchronizationPrimitive();
	task = globalTaskQueue.GetAndRemove();
	UnlockSynchronizationPrimitive();
	
	Execute(task);
}
```

我特意暂时省略了 C++ 实现的细节。在当前实现中需要注意的一件事是，向全局队列添加任务或从全局队列中删除任务时，需要跨多个线程进行适当的同步。不管实际使用的底层同步原语（例如互斥锁、临界区）是什么，这些同步点都会导致大量[线程争用](http://stackoverflow.com/questions/1970345/what-is-thread-contention)。我们的任务和线程越多，就会引起越多的竞争——这就是作业窃取的出发点，这将在本系列的未来部分中讨论。

还缺少的最后一件事是 `WaitUntilTasksAreFinished()` 的实现：就实现复杂性而言，最简单的解决方案是简单地等待，直到队列中没有更多的项目。然而，这需要在访问全局队列时获取另一个锁，这可能会产生更多的争用。更好的解决方案是使用原子变量，它表示仍然可用的任务数。


## 实现细节 - *Implementation details*
---
如果你想自己实现一个简单的调度器，这里有一些起点：

+ 使用一个简单的同步原语（如临界区）来保护队列免受并发访问。在 `volatile` 变量上使用自旋锁或忙碌等待可能将在其他平台上造成问题。性能优势主要是通过减少争用而获得的，而不是使单个锁操作稍微快一点。消除争用的来源将在该系列的未来部分中进行处理。
+ 在等待任务可用时使用[条件变量](http://stackoverflow.com/questions/2476235/what-are-common-uses-of-condition-variables-in-c)。工作线程等待条件变量发出信号，而 `AddTaskToScheduler()` 则在添加任务后立即发出条件信号。这还有一个额外的好处，即当队列中没有任务时，工作线程不需要占用 CPU 资源。
+ 原子变量可以使用 *Windows* 上的 *Interlocked\** 函数来实现。其他平台或操作系统应该也提供类似的功能——如果没有，你可以始终通过内联汇编代码（x86上使用 *LOCK* 前缀）使用硬件指令。


## 不足与展望 - *Deficiencies and outlook*
---
在这篇文章的结尾，让我们看看当前简单的调度程序的不足之处：

+ 全局队列上的锁操作会产生大量线程争用。
+ 向系统中添加任务只能以串行方式完成。
+ 无负载平衡。即使每个任务都包含相同数量的要完成的工作，也不能保证线程在同一时间完成任务，从而导致部分工作线程处于空闲状态。
+ 任务之间没有依赖关系。
+ 在等待任务完成时，调用线程处于空闲状态，直到工作线程完成为止。

尽管实现简单是一个很好的起点，但是这个基本调度器在提供本文开头所讨论的优点之前还需要进行大量改进。未来的帖子将讨论如何添加依赖关系，以及如何在 *N* 个核心之间自动负载平衡工作。