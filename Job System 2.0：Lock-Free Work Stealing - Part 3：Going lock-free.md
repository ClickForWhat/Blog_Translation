*本文档译自 blog.molecular-matters.com 的 "Job System 2.0" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
本周，我们将最终解决作业系统的核心问题：无锁作业窃取队列的实现。请继续阅读低层级编程。


## 本系列其他文章 - *Other posts in the series*
---
[Part 1](https://blog.molecular-matters.com/2015/08/24/job-system-2-0-lock-free-work-stealing-part-1-basics/): 描述了新工作制度的基础，并总结了作业窃取的行为。 
[Part 2](https://blog.molecular-matters.com/2015/09/08/job-system-2-0-lock-free-work-stealing-part-2-a-specialized-allocator/): 详细介绍线程本地分配机制。


## 回顾 - *Recap*
---
记住作业窃取队列需要提供的三个操作：

+ `Push()`：将作业添加到队列的私有端（LIFO）。
+ `Pop()`：从队列的私有端（LIFO）删除作业。
+ `Steal()`：从队列的公共端（FIFO）窃取作业。

进一步记住，`Push()` 和 `Pop()` 只能由拥有队列的线程调用，因此不能并发调用。任何其他线程都可以在任何时间与 `Push()` 和 `Pop()` 并发地调用 `Steal()`。


## 带锁的实现 - *A locking implementation*
---
在深入研究无锁编程领域之前，让我们首先研究一个使用传统锁的实现。从概念上讲，我们如何构建一个像双端队列（*deque*）一样的数据结构，一端后进先出，另一端先进先出？

幸运的是，这可以很容易地通过使用两个索引来表示队列的两端来解决，正如 [*D. Chase* 和 *Y. Lev* 所描述的那样：“动态循环的作业窃取队列”](http://neteril.org/~jeremie/Dynamic_Circular_Work_Queue.pdf)。如果我们暂时假设我们有无限的内存，因此有一个包含无限数量条目的数组，我们可以引入两个计数器，称为 *bottom* 和 *top* ，它们具有以下属性：

+ *Bottom* 表示数组中的下一个可用槽，下一个作业将被推到该槽中。`Push()` 首先存储作业，然后对底部进行递增。
+ 类似地，`Pop()` 从底部递减，然后返回存储在数组中该位置的作业。
+ *top* 表示下一个可以被窃取的元素（即队列中最顶层的元素）。`Steal()` 从数组中的该槽中获取作业，并使 *top* 递增。

需要注意的一个重要现象是，在任何给定的时间点，*bottom* - *top* 表示队列中当前存储的作业数量。如果 *bottom* 小于或等于 *top*，则队列为空，没有东西可偷。

另一个有趣的事实是 `Push()` 和 `Pop()` 都只改变底部，而 `Steal()` 只改变顶部。这是一个非常重要的属性，可以最大限度地减少工作窃取队列所有者的同步开销，并被证明对无锁实现是有利的。

为了更好地说明这种双端队列是如何工作的，请看下面在队列上执行的操作列表：

|**操作**|**bottom**|**top**|**大小 (bottom – top)**|
|---|---|---|---|
|空队列|0|0|0|
|Push|1|0|1|
|Push|2|0|2|
|Push|3|0|3|
|Steal|3|1|2|
|Pop|2|1|1|
|Pop|1|1|0|

正如之前所说的，`Push()` 和 `Pop()` 以 *LIFO* 的方式工作，`Steal()` 以 *FIFO* 的方式工作。在上面的例子中，第一次调用 `Steal()` 将返回索引0的作业，而随后调用 `Pop()` 将依次返回索引2和1处的作业。

在 C++ 代码中，作业窃取队列的三个操作的实现如下所示：

```C++
void Push(Job* job)
{
    ScopedLock lock(criticalSection);
	
    m_jobs[m_bottom] = job;
    ++m_bottom;
}
 
Job* Pop(void)
{
    ScopedLock lock(criticalSection);
	
    const int jobCount = m_bottom - m_top;
    if (jobCount <= 0)
    {
        // no job left in the queue
        return nullptr;
    }
	
    --m_bottom;
    return m_jobs[m_bottom];
}
 
Job* Steal(void)
{
    ScopedLock lock(criticalSection);
	
    const int jobCount = m_bottom - m_top;
    if (jobCount <= 0)
    {
        // no job there to steal
        return nullptr;
    }
	
    Job* job = m_jobs[m_top];
    ++m_top;
    return job;
}
```

现在唯一要做的就是把无界的无限数组变成环形数组。这可以通过相应地包装 *bottom* 和 *top* 来实现，例如使用模运算。但这将使计算在队列中的任务数量变得更加困难，因为我们必须考虑到这种环状的结构。一个更好和更简单的解决方案是在访问数组时才对 *bottom* 和 *top* 应用模操作。只要存储在队列中的元素数是2的幂，这就只不过是一个二进制与操作：

```C++
static const unsigned int NUMBER_OF_JOBS = 4096u;
static const unsigned int MASK = NUMBER_OF_JOBS - 1u;
 
void Push(Job* job)
{
    ScopedLock lock(criticalSection);
	
    m_jobs[m_bottom & MASK] = job;
    ++m_bottom;
}
 
Job* Pop(void)
{
    ScopedLock lock(criticalSection);
	
    const int jobCount = m_bottom - m_top;
    if (jobCount <= 0)
    {
        // no job left in the queue
        return nullptr;
    }
	
    --m_bottom;
    return m_jobs[m_bottom & MASK];
}
 
Job* Steal(void)
{
    ScopedLock lock(criticalSection);
	
    const int jobCount = m_bottom - m_top;
    if (jobCount <= 0)
    {
        // no job there to steal
        return nullptr;
    }
	
    Job* job = m_jobs[m_top & MASK];
    ++m_top;
    return job;
}
```

这就是使用传统锁的工作窃取队列的实现。


## 前情提要：无锁编程 - *Prerequisites: Lock-free programming*
---
无锁编程是一个巨大的主题，已经有很多关于它的文章。与其重复已经说过的，不如在这里推荐一些好的文章，我希望每个人在继续这篇文章之前都能读到：

- [无锁编程的简介](http://preshing.com/20120612/an-introduction-to-lock-free-programming/)：涉及技术、原子操作、顺序一致性和内存排序。这是一本很棒的无锁编程101必读文章！
- [编译时的内存序](http://preshing.com/20120625/memory-ordering-at-compile-time/)：讨论编译器指令重排和内存屏障，并提供有见地的示例。
- [内存序重排探秘](http://preshing.com/20120515/memory-reordering-caught-in-the-act/)：展示了 *x86/64* 上发生的内存序重排的示例，尽管 *Intel* 的 *x86/64* 体系结构提供了强序的内存模型。
- [强/弱内存模型](http://preshing.com/20120930/weak-vs-strong-memory-models/)：讨论弱和强内存模型，以及顺序一致性。

总的来说，*Jeff Preshing* 的博客是一个充满了关于无锁编程文章的金矿。如果你有疑问，看看他的博客。


## 无锁作业窃取队列 - *A lock-free work-stealing queue*
---
看完了所有的文章吗？好。

假设现在没有编译器重排，也绝对没有内存重排，让我们以无锁的方式一个接一个地实现这三个操作。稍后我们将讨论关于编译器和内存屏障的事。

首先考虑 `Push()` 的无锁实现：

```C++
void Push(Job* job)
{
    long b = m_bottom;
    m_jobs[b & MASK] = job;
    m_bottom = b + 1;
}
```

如果其他操作同时发生，会发生什么？`Pop()` 不能并发执行，所以我们只需要考虑 `Steal()`。然而， `Steal()` 只从 *top* 写入，从 *bottom* 读取。因此，最糟糕的情况是，在第5行执行完之前（执行完表示新条目已经可用）， `Push()` 被 `Steal()` 抢占。但这是无关紧要的，因为它只意味着 `Steal()` 不能窃取已经在存储在那里的项目。

接下来是 `Steal()` 操作：

```C++
Job* Steal(void)
{
    long t = m_top;
    long b = m_bottom;
    if (t < b)
    {
        // 非空的队列
        Job* job = m_jobs[t & MASK];
		
        if (_InterlockedCompareExchange(&m_top, t + 1, t) != t)
        {
            //同时执行的其他窃取或弹出操作已经从队列中移除了一个元素
            return nullptr;
        }
		
        return job;
    }
    
    // 空队列
    return nullptr;
}
```

只要 *top* 小于 *bottom*，说明队列中仍有待窃取的作业。如果队列不为空，则该函数首先读取存储在数组中的作业，然后尝试使用 [*Compare-And-Swap*](https://msdn.microsoft.com/en-us/library/windows/desktop/ms683560(v=vs.85).aspx) 操作增加 *top*。如果 *CAS* 失败，则说明并发的 `Steal()` 或 `Pop()` 操作已经成功地从队列中删除了一个作业。

注意，在执行 *CAS* 之前读取作业是很重要的，因为在 *CAS* 完成之后发生的并发 `Push()` 操作可能会覆盖数组中的位置（别忘记这是个环形数组）。

这里需要注意的一个非常关键的事情是，*top* 总是在 *bottom* 之前读取，以确保这些值表示内存的一致视图。另外，如果在读取 *bottom* 之后和执行 *CAS* 之前并发的 `Pop()` 清空了要读取的条目，可能会出现微妙的竞争。我们需要确保并发的 `Pop()` 和 `Steal()` 操作不会同时返回队列中剩余的最后一个作业，这可以通过尝试使用 *CAS* 操作修改 `Pop()` 实现中的 *top* 来实现。

`Pop()` 操作是其中最有趣的一个：

```C++
Job* Pop(void)
{
	//通过操作 bottom 抢占此位置
    long b = m_bottom - 1;
    m_bottom = b;
    long t = m_top;
    
    if (t == b || t < b)// 非空队列
    {
        Job* job = m_jobs[b & MASK];
        if (t != b)//队列里仍然至少有超过一个作业 (t < b)
        {
            return job;
        }
		
        // 队列里只剩最后一个作业 （t == b)
        if (_InterlockedCompareExchange(&m_top, t + 1, t) != t)//竞争
        {
            // 和 Steal() 竞争失败，数据已被取走
            job = nullptr;
        }
		
        m_bottom = t + 1;
        return job;
    }
    else
    {
        // 队列已空，置回和 top 相同的值
        m_bottom = t;
        return nullptr;
    }
}
```

与 `Steal()` 的实现相反，这一次我们需要确保在尝试读取 *top* 之前先递减 *bottom*。否则，并发的 `Steal()` 操作可能会在 `Pop()` 不注意的情况下从队列中删除多个作业。 

另外，如果队列已经是空的，我们需要将其重置为“规范”的空状态，也就是 *bottom* == *top*。

只要队列中还有几个作业，我们就可以简单地返回它，而无需执行任何额外的原子操作。然而，正如在上面的实现说明中指出的那样，当只剩下一个作业的时候，我们需要保护代码免受其他并发调用的 `Steal()` 的竞争。

如果这种情况发生了，代码将执行 *CAS* 以递增 *top* 并检查我们是否在与并发的 `Steal()` 操作的竞争中获胜。这个操作的结果只有两种可能：

+ *CAS* 成功了，我们赢得了与 `Steal()` 的竞争。在这种情况下，我们设置 *bottom* = *t* + 1，这将队列设置为规范的空状态。
+ *CAS* 失败了，我们输掉了与 `Steal()` 的竞争。在这种情况下，我们返回一个空作业，但仍然设置 *bottom* = *t* + 1。但这是为什么？因为赢得竞争的那个 `Steal()` 操作成功地设置了 *top* = *t* + 1，所以我们仍然必须将队列设置为空状态。


## 添加编译器和内存屏障 - *Adding compiler and memory barriers*
---
到目前为止，一切顺利。不过在目前的形式下，实现还无法工作，因为我们完全将编译器和内存序排除在外。这需要解决。

考虑 `Push()` 操作：

```C++
void Push(Job* job)
{
    long b = m_bottom;
    m_jobs[b & MASK] = job;
    m_bottom = b + 1;
}
```

没有人保证编译器不会重新排序上面的任何语句。具体来说，我们不能确保我们总是先将作业存储在数组中，然后再通过增加 *bottom* 来向其他线程发出信号——可能会反过来，这将导致其他线程窃取尚未存在的作业。

在这种情况下，我们需要的是一个编译器屏障：

```C++
void Push(Job* job)
{
    long b = m_bottom;
    m_jobs[b & MASK] = job;
	
    // ensure the job is written before b+1 is published to other threads.
    // on x86/64, a compiler barrier is enough.
    COMPILER_BARRIER;
	
    m_bottom = b + 1;
}
```

请注意，在x86/64上，编译器屏障就足够了，因为强序内存模型不允许存储与其他存储一起重新排序。在其他平台（*PowerPC*、*ARM*…）上，你将需要一个 *Memmory Fence*。此外，请注意，在这种情况下，存储操作也不需要以原子方式执行，因为改变 *bottom* 的唯一其他操作是 `Pop()`，它不能同时执行。

类似地，我们也需要在 `Steal()` 的实现中设置编译器屏障：

```C++
Job* Steal(void)
{
    long t = m_top;
	
    // 保证 top 总是先于 bottom 读取
    // 加载不会与x86上的其他加载一起重排，因此编译器屏障就足够了。
    COMPILER_BARRIER;
	
    long b = m_bottom;
    if (t < b)
    {
        // non-empty queue
        Job* job = m_jobs[t & MASK];
		
        // Interlocked 函数充当编译器屏障，并确保读取发生在CAS之前。
        if (_InterlockedCompareExchange(&m_top, t+1, t) != t)
        {
            // 同时执行的窃取或弹出操作从队列中移除了一个元素。
            return nullptr;
        }
		
        return job;
    }
    else
    {
        // empty queue
        return nullptr;
    }
}
```

在这里，我们需要一个编译器屏障来确保读 *top* 确实发生在读 *bottom* 之前。此外，我们还需要另一个屏障来保证在 *CAS* 之前执行对数组的读取。然而，在这种情况下，*Interlocked* 函数隐式地充当编译器屏障。

还有更多步骤要做：

```C++
Job* Pop(void)
{
    long b = m_bottom - 1;
    m_bottom = b;
	
    long t = m_top;
    if (t <= b)
    {
        // non-empty queue
        Job* job = m_jobs[b & MASK];
        if (t != b)
        {
            // there's still more than one item left in the queue
            return job;
        }
		
        // this is the last item in the queue
        if (_InterlockedCompareExchange(&m_top, t+1, t) != t)
        {
            // failed race against steal operation
            job = nullptr;
        }
		
        m_bottom = t+1;
        return job;
    }
    else
    {
        // deque was already empty
        m_bottom = t;
        return nullptr;
    }
}
```

这里最重要的部分是前三行。这是我们需要真正内存屏障的罕见情况之一，即使在英特尔的 x86/64 体系结构上也是如此。更具体地说，仅仅在存储到 *bottom* = *b* 和从 *long* *t* = *top* 读取之间添加一个编译器屏障是不够的，因为内存模型明确允许“加载可以用旧的存储重新排序到不同的位置”，这正是这里的情况。

这意味着前几行代码需要按如下方式修复：

```C++
long b = m_bottom - 1;
m_bottom = b;
 
MEMORY_BARRIER;
 
long t = m_top;
```

注意，在这种情况下，[*SFENCE*](http://x86.renejeschke.de/html/file_module_x86_id_289.html) 和 [*LFENCE*](http://x86.renejeschke.de/html/file_module_x86_id_155.html) 都还不够，它实际上需要是一个完整的 [*MFENCE*](http://x86.renejeschke.de/html/file_module_x86_id_170.html) 屏障。或者，我们也可以使用 *Interlocked* 操作，如 [*XCHG*](http://x86.renejeschke.de/html/file_module_x86_id_328.html)，而不是完整的屏障。*XCHG* 在内部像一个屏障，但更节省一些：

```C++
long b = m_bottom - 1;
_InterlockedExchange(&m_bottom, b);
 
long t = m_top;
```

`Pop()` 实现的其余部分可以保持原样。与 `Steal()` 类似，*CAS* 操作充当编译器屏障，并且执行 *bottom* = *t* + *1* 不需要是原子的，因为不可能有任何并发操作也写入 *bottom*。


## 性能 - *Performance*
---
无锁实现在性能方面给我们带来了什么？下面是结果：

| |Basic|Thread-local allocator|**Lock-free deque**|Increase vs. first impl.|Increase vs. second impl.|
|---|---|---|---|---|---|
|Single jobs|18.5 ms|9.9 ms|2.93 ms|**6.31x**|**3.38x**|
|parallel_for|5.3 ms|1.35 ms|0.76 ms|**6.97x**|**1.78x**|

与我们的第一个实现相比，无锁和使用线程本地分配器使我们的性能提高了近7倍。即使在使用线程本地分配器的基础上，无锁实现的执行速度也提高了1.8到3.3倍。


## 展望 - *Outlook*
---
今天就到这里。下次，我们将看到如何在这个系统之上实现像 *parallel_for* 这样的高级算法。