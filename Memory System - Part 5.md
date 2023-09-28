*本文档译自 blog.molecular-matters.com 的 "Memory System" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
这是关于 *Molecule* 内存系统系列文章的最后一篇。今天，我们将研究这个谜题的最后一部分，将不同的东西粘合在一起，比如原始内存分配、边界检查、内存跟踪等等。

在本系列的上一篇文章中，我们确定了内存系统的许多不同用例，并讨论了它的需求。具体来说，我们希望能够任意地将不同的功能组合到简单的 *Memory Arena* 中。正如你可能已经猜到的那样，[基于策略的设计](https://molecularmusings.wordpress.com/2011/06/29/designing-extensible-modular-classes/)是一种非常强大的工具，可以帮助你完成此任务。

所以，事不宜迟，以下是我在 *Molecule* 中提出的基本 *Memory Arena* 实现的解决方案：

```C++
template <class AllocationPolicy, class ThreadPolicy, class BoundsCheckingPolicy, class MemoryTrackingPolicy, class MemoryTaggingPolicy>
class MemoryArena
{
public:
    template <class AreaPolicy>
    explicit MemoryArena(const AreaPolicy& area)
        : m_allocator(area.GetStart(), area.GetEnd())
    {
    }
	 
    void* Allocate(size_t size, size_t alignment, const SourceInfo& sourceInfo)
    {
        m_threadGuard.Enter();
		 
        const size_t originalSize = size;
        const size_t newSize = size + BoundsCheckingPolicy::SIZE_FRONT + BoundsCheckingPolicy::SIZE_BACK;
		 
        char* plainMemory = static_cast<char*>(m_allocator.Allocate(newSize, alignment, BoundsCheckingPolicy::SIZE_FRONT));
		 
        m_boundsChecker.GuardFront(plainMemory);
        m_memoryTagger.TagAllocation(plainMemory + BoundsCheckingPolicy::SIZE_FRONT, originalSize);
        m_boundsChecker.GuardBack(plainMemory + BoundsCheckingPolicy::SIZE_FRONT + originalSize);
		 
        m_memoryTracker.OnAllocation(plainMemory, newSize, alignment, sourceInfo);
		 
        m_threadGuard.Leave();
		 
        return (plainMemory + BoundsCheckingPolicy::SIZE_FRONT);
    }
	 
    void Free(void* ptr)
    {
        m_threadGuard.Enter();
		 
        char* originalMemory = static_cast<char*>(ptr) - BoundsCheckingPolicy::SIZE_FRONT;
        const size_t allocationSize = m_allocator.GetAllocationSize(originalMemory);
		 
        m_boundsChecker.CheckFront(originalMemory);
        m_boundsChecker.CheckBack(originalMemory + allocationSize - BoundsCheckingPolicy::SIZE_BACK);
		 
        m_memoryTracker.OnDeallocation(originalMemory);
         
        m_memoryTagger.TagDeallocation(originalMemory, allocationSize);
		 
        m_allocator.Free(originalMemory);
		 
        m_threadGuard.Leave();
    }
	 
private:
	AllocationPolicy m_allocator;
	ThreadPolicy m_threadGuard;
	BoundsCheckingPolicy m_boundsChecker;
	MemoryTrackingPolicy m_memoryTracker;
	MemoryTaggingPolicy m_memoryTagger;
};
```

通过使用模板替换 `MemoryArena` 实现的不同部分，功能被分成多个类，每个类只负责一件事。此外，我们可以使用一个简单的 `typedef` 组合我们想要的任何东西，如下面的例子所示：

```C++
#if ME_DEBUG || ME_RELEASE
	typedef MemoryArena<LinearAllocator, SingleThreadPolicy, SimpleBoundsChecking, SimpleMemoryTracking, SimpleMemoryTagging> ApplicationArena;
#else
	typedef MemoryArena<LinearAllocator, SingleThreadPolicy, NoBoundsChecking, NoMemoryTracking, NoMemoryTagging> ApplicationArena;
#endif
```

在发行版构建中，将使用所有的空实现，它们只包含空内联函数：

```C++
class NoBoundsChecking
{
public:
	static const size_t SIZE_FRONT = 0;
	static const size_t SIZE_BACK = 0;
	
	inline void GuardFront(void*) const {}
	inline void GuardBack(void*) const {}
	
	inline void CheckFront(const void*) const {}
	inline void CheckBack(const void*) const {}
};
 
class NoMemoryTracking
{
public:
	inline void OnAllocation(void*, size_t, size_t, const SourceInfo&) const {}
	inline void OnDeallocation(void*) const {}
};
 
class NoMemoryTagging
{
public:
	inline void TagAllocation(void*, size_t) const {}
	inline void TagDeallocation(void*, size_t) const {}
};
```


## 调试并排除故障- *Debugging*
---
我们不再依赖隐藏在源代码本身深处的 `#if/#endif` 子句，而是使用编译器的模板机制来构建我们的代码，我们可以使用 `typedef` 任意切换不同类型的调试辅助工具，甚至不需要访问实现的源代码。

例如，每当发现内存泄漏时，*SimpleMemoryTracking* 实现都会只做一件事——它只是在每次分配/释放时分别递增/递减一个计数器。

一旦我知道代码包含泄漏，我就可以简单地切换到 `typedef` 中的 *ExtendedMemoryTracking* 实现，并准确地获知泄漏发生的位置。这有助于避免一直使用调试进行构建，因为它们用于跟踪目的使用了太多内存，太多的开销导致缓慢。

边界检查也是如此，用户可以轻松地在不同的实现之间切换。曾经想要为那些只发生在发行版本中的难以发现的 *bug* 启用扩展边界检查吗？你现在可以这么做了！


## 线程安全 - *Thread-Safety*
---
正如本系列的上一篇文章所述，并非每个分配都需要是线程安全的。例如，*Molecule* 的应用程序 *Arena* 默认是单线程的，因为那时没有其他线程参与——这就是 *Level Arena* 的作用。

本质上，单线程什么都不需要做：

```C++
class SingleThreadPolicy
{
public:
	inline void Enter(void) const {}
	inline void Leave(void) const {}
};
```

其他 *Arena* 则需要它们的分配是线程安全的，这就是 *MultiThreadPolicy* 发挥作用的地方：

```C++
template <class SynchronizationPrimitive>
class MultiThreadPolicy
{
public:
	inline void Enter(void)
	{
	    m_primitive.Enter();
	}
	
	inline void Leave(void)
	{
	    m_primitive.Leave();
	}
	
private:
	SynchronizationPrimitive m_primitive;
};
```

如你所见，该策略也接受一个模板参数，该参数提供了要使用的同步原语。这样，用户就可以自己决定需要哪种原语。大多数情况下，这将是一个互斥锁或临界区，但在一些罕见的情况下，自旋锁被证明是更快的（例如，对于多线程渲染队列中的线性分配器），你完全可以使用它，而不必支付其他原语更大的开销。


## 内存区 - *Memory areas*
---
内存区域定义了从哪里分配内存。最常见的区域是堆区和栈区。前者直接从操作系统分配内存，而后者使用 `alloca()` 在堆栈上分配内存。此外，还可以为 GPU内存（例如 *PS3*）、写合并内存（*Xbox360*）、外部内存（*Wii*）等留出空间。

同样，所有这些都可以与任何分配器和所需的任何类型的调试辅助相结合。


## 扩展性 - *Extensibility*
---
基础的 *Memory Arena* 实现可以很容易地扩展，而无需触及源代码。例如，*Molecule* 提供了一个记录应用程序中每次分配的工具，这些分配可以稍后在外部工具中重播（所谓的内存重播）。这可以通过简单地将一个分配器连接到另一个分配器来实现：

```C++
template <class Arena>
class RecordingArena
{
public:
	void* Allocate(size_t size, size_t alignment, const SourceInfo& sourceInfo)
	{
	    // send info via TCP/IP...
	    return m_arena.Allocate(size, alignment, sourceInfo);
	}
	
	void Free(void* ptr)
	{
	    // send info via TCP/IP...
	    m_arena.Free(ptr);
	}
	
private:
	Arena m_arena;
};
```

`RecordingArena` 并不关心内存实际上如何分配，它只时通过 *TCP/IP* 发送信息到外部的应用中，然后使用任意 `Arena`（由模板参数提供）作为实际的内存分配/释放实现。


## 用法 - *Usage*
---
使用 *Memory Arena* 和指定内存区非常简单。假如你希望在栈上分配一个对象池，并限制在单个线程中，我们可以：

```C++
typedef MemoryArena<PoolAllocator, SingleThreadPolicy, NoBoundsChecking, NoMemoryTracking, NoMemoryTagging> ST_PoolStackArena;
 
void SomeFunction()
{
    char* stackArea[2048];
    ST_PoolStackArena arena(stackArea);
	
    MyObject* obj = ME_NEW(MyObject, arena);
    // ...
    ME_DELETE(obj, arena);
}
```

又或者是一个多线程的基于堆栈的分配器从 *GPU* 内存分配：

```C++
typedef MemoryArena<StackBasedAllocator, MultiThreadPolicy<CriticalSection>, NoBoundsChecking, NoMemoryTracking, NoMemoryTagging> MT_StackBasedArena;
 
// ...
GpuArea gpuArea(16 * 1024 * 1024);
MT_StackBasedArena gpuArena(gpuArea);
 
TextureData* data = ME_NEW(TextureData, gpuArena);
// ...
ME_DELETE(data , gpuArena);
}
```

再或者你需要一个通用的，线程安全的分配器与各种调试功能、用于临时单帧分配的线性无锁分配器？你懂的，可能性是无限的。


## 结论 - *Conclusion*
---
关于 *Molecule* 内存系统的迷你系列到此结束。到目前为止，我对内存系统非常满意：泄漏和内存覆盖可以很快找到，碎片可以保持在绝对最小（通过为作业使用正确的分配器，应用程序范围和关卡级别的分配可以实现零碎片，以及多种单帧分配、*I/O* 分配等），并且可以很容易地扩展。