*本文档译自 blog.molecular-matters.com 的 "Stateless, layered, multi-threaded rendering" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
本系列的最后一篇文章总结了以下问题：在多个线程向同一个 *bucket* 中添加命令的情况下，我们如何有效地为单个命令包分配内存？如何在整个命令包的存储和提交过程中保证良好的缓存利用率？

这就是我们今天要解决的问题。我想展示命令包糟糕的内存分配会如何影响多线程渲染的性能，以及我们的替代方案是什么。


## 系列的文章 - *Previously in the series*
----
这是系列相关博客文章的第四篇。其他文章的完整列表：

-   [Part 1: Basics](https://molecularmusings.wordpress.com/2014/11/06/stateless-layered-multi-threaded-rendering-part-1/ "Stateless, layered, multi-threaded rendering – Part 1")
-   [Part 2: Stateless API Design](https://molecularmusings.wordpress.com/2014/11/13/stateless-layered-multi-threaded-rendering-part-2-stateless-api-design/ "Stateless, layered, multi-threaded rendering – Part 2: Stateless API Design")
-   [Part 3: API Design Details](https://molecularmusings.wordpress.com/2014/12/16/stateless-layered-multi-threaded-rendering-part-3-api-design-details/ "Stateless, layered, multi-threaded rendering – Part 3: API Design Details")
-   Part 4: Memory Management & Synchronization


## 开始 - *Setup*
----
重要的事情先做。我们的基准测试的设置如下所示：

+ 所有测试都在英特尔酷睿 *i7-2600K*，*3.40 GHz*，4核CPU上进行。CPU有4个物理内核，但由于启用了超线程，有8个逻辑内核。
+ 当使用 *Molecule* 的任务调度器在多线程环境中运行时，每个可用的逻辑核心都有一个工作线程，也就是一共8个。
+ 基准测试一次运行提交了10,000个网格组件和10,000个灯光组件。每个网格组件被添加到两个  *bucket* 中：*GBuffer bucket* 和 *ShadowMap bucket*，因此每个 *bucket* 有10,000个绘制调用。每个灯光组件都需要进行 *Map*，以便更新常量缓冲区（导致10,000次 *Map* 操作），每个灯光组件还向 *Lighting bucket* 发起一个绘制调用。总而言之，这将导致40,000次 `AddCommand()`/`AppendCommand()` 操作。
+ 网格和灯光组件已经到位，所以没有额外的工作，如视锥剔除或类似的东西。但是，向 *bucket* 中添加命令会产生一些人为的额外工作，以便让测试更接近真实的工作负载。
+ 使用 `GenerateKey()` 生成的键通过某种分布方式已经经过排序，不会以完全连续的方式遍历数据包。
+ API提交不会与渲染API（如 *D3D*）产生真正的交互，而只是根据所有输入参数计算哈希值。检查这个散列以确保所有实现都正确工作。
+ 以进行1000次基准测试的平均时间作为测试结果。
+ 在每次基准测试之间，通过随机访问（读和写）16MB与测试无关的数据来清除缓存。


## 版本 1 - *Version 1*
----
为了感受一下在单线程场景中基准测试需要多长时间，我们使用上一篇文章中描述的实现来运行它。我们感兴趣的两件事如下：

-   如何为各个命令包分配内存？

```C++
template <typename T>
CommandPacket Create(size_t auxMemorySize)
{
	return ::operator new(GetSize<T>(auxMemorySize));
}
```

在本例中，我们使用全局 `operator new` 为命令包分配内存。

+ 如何向 *bucket* 中添加命令？

```C++
template <typename U>
U* AddCommand(Key key, size_t auxMemorySize)
{
	CommandPacket packet = commandPacket::Create<U>(auxMemorySize);
	{
		const unsigned int current = m_current++;
	    m_keys[current] = key;
	    m_packets[current] = packet;
	}
	commandPacket::StoreNextCommandPacket(packet, nullptr);
	commandPacket::StoreBackendDispatchFunction(packet, U::DISPATCH_FUNCTION);
	
	return commandPacket::GetCommand<U>(packet);
}
```

注意，这个版本的 `AddCommand()` 不是线程安全的，这在本例中没有问题，因为在这个版本中我们只使用单线程。

下面是版本1的所花时间：

-   添加命令：**6.65ms**。
-   提交命令：**0.7ms**。

让我们看看我们能做些什么？


## 版本 2 - *Version 2*
----
最明显的对性能有害的事情是使用全局 *new* 分配每个命令包。首先，这不能保证各个包在内存中彼此接近，这降低了提交命令包时的缓存命中率。第二，全局 *new* 很慢，这不是实现的错，而是我们的错。*new*、*malloc* 等是作为通用分配器而开发出来的，试图在平均情况下做到最好。然而，我们确切地知道我们要分配什么，以及分配的生命周期是什么：一帧。我们可以在一帧之后把所有东西都扔掉。

因此，我们用线性内存分配器代替对全局 `operator new` 的调用，这在单线程情况下只是递增一个指针：

```C++
void* alloc = m_current;
m_current += size;
return alloc;
```

我们只是简单地分配一个内存块（比如4MB）给我们所有的 *bucket* 使用，并在一帧期间使用它来分配任何 *bucket* 所做的所有分配。

这给了我们以下结果：

-   添加命令：**4.55ms**。
-   提交命令：**0.7ms**。

添加命令现在是1.46倍的速度，很好！这很简单。现在，让我们加入一些线程。


## 版本 3 - *Version 3*
----
在这个多线程场景中，我们使用 *Molecule* 的任务调度器将工作分配给8个不同的工作线程。每向 *bucket* 中添加50个组件，就会创建一个任务。这意味着我们将为调度器创建400个任务（网格和灯光组件各为200个），让它们并行运行，并等待它们全部完成。这是使用调度器API公开的简单父/子关系来完成的。

为命令包分配内存，我们要重新使用全局 `operator new`，因为它是线程安全的（我们还没有线程安全的内存分配实现）。

为了向 *bucket* 中添加命令，我们对实现做了一个简单的更改：

```C++
template <typename U>
U* AddCommand(Key key, size_t auxMemorySize)
{
	CommandPacket packet = commandPacket::Create<U>(auxMemorySize);
	
	{
	    const int32_t current = core::atomic::Increment(&m_current);
	    m_keys[current] = key;
	    m_packets[current] = packet;
	}
	
	commandPacket::StoreNextCommandPacket(packet, nullptr);
	commandPacket::StoreBackendDispatchFunction(packet, U::DISPATCH_FUNCTION);
	
	return commandPacket::GetCommand<U>(packet);
}
```

我们现在使用原子操作，而不是执行 `m_current++`，原子地递增 `m_current` 的值，在调用函数之前返回其初始值。在 *x86-64*上，这可以通过在汇编代码中使用 `InterlockedIncrement`、`InterlockedExchangeAdd` 或 [*LOCK XADD*](http://x86.renejeschke.de/html/file_module_x86_id_327.html) 中的任意一个来完成。

	Note：在具有弱序内存模型或CPU指令重排的平台上工作时（如Xbox360、PlayStation 3和Wii U中的PowerPC内核），你必须通过插入正确的Memory Barrier（或使用Acquire/Release语义）来确保原子指令不会被重新排序。此外，你需要确保其他内核可以通过使用适当的Memory Barrier（如lwsync或sync）在内存中看到原子操作的结果。

有了这些变化，时间花费如下：

-  添加命令：**1.85ms**。
-  提交命令：**0.7ms**。

使用相同的分配器从单线程提交到多线程提交给我们带来了3.54倍的加速，这是在4核CPU上所期望的。任务调度程序确实也有开销，而且它的可伸缩性可以更好，但目前我们不需要担心这一点。

让我们再次尝试线性分配器，但这次是在多线程环境中。


## 版本 4 - *Version 4*
----
使线性分配器实现线程安全是很容易的，我们只需要使用原子操作来操作线性分配器的指针：

```C++
const int32_t currentOffset = core::atomic::Add(&m_offset, size);
return m_start + currentOffset;
```

让我们看看在多线程的情况下，从使用全局 `operator new` 到使用线性分配器，是否也能获得1.46倍的加速？

-  添加命令：**1.72ms**。
-  提交命令：**0.7ms**。

Hmm......这只有1.07倍的提升，我们做错了什么！？

实际上有几点要说明。首先，原子操作不像获取锁那样昂贵，但它们也[不是没有开销](https://fgiesen.wordpress.com/2014/08/18/atomics-and-contention/)的。其次，发生了一些[伪共享](https://zh.wikipedia.org/wiki/%E4%BC%AA%E5%85%B1%E4%BA%AB)，因为所有的内核写入位于同一缓存行（*cache line*）上的内存目标，至少在某些时候是这样。它并没有那么糟糕，因为与缓存行大小相比（64 *Bytes*），我们的分配大小相当大（20/36 *Bytes*）。但是，这种现象仍然存在。


## 版本 5 - *Version 5*
----
以使用更多内存为代价，我们可以通过将每个分配的大小四舍五入为缓存行大小的倍数来排除伪共享，确保每个线程/内核完全拥有整个缓存行：

 ```C++
const int32_t currentOffset = core::atomic::Add(&m_offset, RoundUp(size, 64));
return m_start + currentOffset;
```

现在时间花费如下：

-  添加命令：**1.65ms**。
-  提交命令：**0.9ms**。

添加命令现在更快了，多亏了伪共享的消除。然而，添加命令方面获得的提升在提交命令那里又丢掉了。原因是我们现在比以前涉及更多的内存，就这么简单。


## 版本 6 - *Version 6*
----
我们需要找到一个解决方案，以避免伪共享，但同时不会导致提交命令涉及更多的内存。解决方案是使用[线程局部存储（*TLS*）](http://en.wikipedia.org/wiki/Thread-local_storage)，或者更具体地说：线程自己的线性分配器。

因为每个线程都有自己的线性分配器，不受任何其他线程的影响，所以我们不需要使用原子操作。此外，因为每个分配器都有自己的内存块，所以不可能发生伪共享。当然，这种解决方案的一个缺点是，我们需要预先分配更多的内存，因为我们需要为每个线程分配器分配一个块。

实现很简单，假设你的引擎/代码库支持 *TLS*：

```C++
char* memory = tlsLinearMemory;
void* alloc = memory;
memory += size;
 
// write back
tlsLinearMemory = memory;
return alloc;
```

有了这个，时间花费是：

-  添加命令：**1.45ms**。
-  提交命令：**0.7ms**。

还不错。提交命令的速度再次加快，添加命令的速度比使用全局 `operator new` 的原始解决方案快1.27倍。

不过，还有一件事要做。现在仍然会发生伪共享，尽管是在不同的地方：当在 `AddCommand()` 中存储键和命令包到数组里时，它们是紧挨着写入的，但这写入操作可能是由完全不同的线程进行的。


## 版本 7 - *Version 7*
----
这是实现的最后一个版本，它消除了在 `AddCommand()` 内部发生的伪共享。同样，我们可以使用 *TLS* 来帮助我们，但这一次不那么容易。

基本思想如下：在添加新命令时，每个进入 `AddCommand()` 的线程都将获得几个条目（例如32），这些条目可用于该线程添加的此命令和后续命令。这意味着当一个线程第一次为一个 *bucket* 输入 `AddCommand()` 时，它会得到32个条目，并立即使用其中的一个。因此，由该线程完成的对 `AddCommand()` 的接下来31个调用不需要任何类型的同步或原子操作，而是可以使用剩余31个条目中的下一个。一旦剩余条目的数量为零，线程将继续获得接下来的32个条目。

每个线程现在都自行写入数组中彼此相邻的32个条目，由于每个存储的键至少占用2个字节，并且缓存行大小为64，因此完全消除了伪共享。此外，我们为每个线程、每个 *bucket* 调用 `AddCommand()` 32次后才执行一次原子操作。

但是实现起来并不简单，因为现在每个 *bucket* 本质上是线程局部存储，而不是全局存储。在代码中，它看起来是这样的：

```C++
// <bucket.h>
template <typename U>
U* AddCommand(Key key, size_t auxMemorySize)
{
	CommandPacket packet = commandPacket::Create<U>(auxMemorySize);
	
	// store key and pointer to the data
	{
	    int32_t id = tlsThreadId;
		
	    // fetch number of remaining packets in this layer for the thread with this id
	    int32_t remaining = m_tlsRemaining[id];
	    int32_t offset = m_tlsOffset[id];
		
	    if (remaining == 0)
	    {
		    // no more storage in this block remaining, get new one
		    offset = core::atomic::Add(&m_current, 32);
		    remaining = 32;
			
		    // write back
		    m_tlsOffset[id] = offset;
		}
		
		int32_t current = offset + (32 - remaining);
		m_keys[current] = key;
	    m_packets[current] = packet;
	    
	    // write back
	    m_tlsRemaining[id] = remaining - 1;
	}
	
	commandPacket::StoreNextCommandPacket(packet, nullptr);
	commandPacket::StoreBackendDispatchFunction(packet, U::DISPATCH_FUNCTION);
	
	return commandPacket::GetCommand<U>(packet);
}
```

在这个实现中，我们将每次调用 `AddCommand()` 产生一个原子操作优化成每32次调用 `AddCommand()` 产生一个原子操作。每次 `AddCommand()` 需要访问2-3个线程局部变量。

这在性能方面带来了什么？

-  添加命令：**1.25ms**。
-  提交命令：**0.7ms**。

使用最后一个实现，与多线程场景中的原始实现相比，我们的速度提高了1.48倍。这意味着，在多线程里从全局 `operator new` 到线性分配器的优化给我们带来了与单线程情况完全相同的加速——但前提是我们消除了多线程里大多数同步点和伪共享！


## 总结 - *Summary*
----
```Text
ST: 单线程
MT: 多线程
-----------------------------------------------------------
版本                               添加(ms)    提交(ms)
-----------------------------------------------------------
版本 1
(ST, global new)                    6.65         0.7
-----------------------------------------------------------
版本 2
(ST, linear)                        4.55         0.7
-----------------------------------------------------------
版本 3
(MT, global new)                    1.85         0.7
-----------------------------------------------------------
版本 4
(MT, linear)                        1.72         0.7
-----------------------------------------------------------
版本 5
(MT, linear, size aligned to 64)    1.65         0.9
-----------------------------------------------------------
版本 6
(MT, thread-local linear)           1.45         0.7
-----------------------------------------------------------
版本 7
(MT, thread-local linear & bucket)  1.25         0.7
-----------------------------------------------------------

Speedup going from global new to linear (ST): 1.46x
Speedup going from global new to linear (MT): 1.48x
Speedup going from naïve single-threaded (Version 1) to multi-threaded (Version 7): 5.32x
```


## 结语 - *Conclusion*
----
通过观察上面的不同实现，我们可以得出什么结论呢？我想到了一些事情：

+ 注意内存是如何分配的，以及如何在代码中访问内存。
+ 警惕伪共享。
+ 原子操作不是免费的。
+ 最后但并非最不重要的是：与其试图使线程间共享数据的同步点尽可能短，不如尝试从一开始就不采用共享数据。不共享可变数据比任何类型的同步都要好。双重缓冲会有所帮助。线程局部存储也可以提供帮助。

这些都是我在设计第3部分中的API时牢记的事情。我试图使分配方案的改变尽可能简单和非侵入性，因为我知道，一旦我们完全切换到多线程实现，这些东西将发挥作用。