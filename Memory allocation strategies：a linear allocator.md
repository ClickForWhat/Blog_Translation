*本文档译自 blog.molecular-matters.com 的 "Memory System" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
在接下来的几周里，我将详细介绍 *Molecule* 内部的内存分配器是如何工作的，今天将从一个简单的、不增长的线性分配器开始，以便首先介绍一些基础内容。

如果你还没有读过之前的文章，那么现在是阅读 *Molecule* 的内存系统及其内部工作原理的好时机：[第 1 部分](https://molecularmusings.wordpress.com/2011/07/05/memory-system-part-1/)、[第 2 部分](https://molecularmusings.wordpress.com/2011/07/07/memory-system-part-2/)、[第 3 部分](https://molecularmusings.wordpress.com/2011/07/08/memory-system-part-3/)、[第 4 部分](https://molecularmusings.wordpress.com/2011/07/15/memory-system-part-4/)和[第 5 部分](https://molecularmusings.wordpress.com/2011/08/03/memory-system-part-5/)。

如果你对这些的细节不感兴趣，下面是内存系统简要的工作原理：

+ 实例是通过 `ME_NEW`/`ME_DELETE` 来进行分配/释放的。
+ 所谓的 *Memory Arena* 负责实际分配/释放内存并正确构建其实例或数组。
+ *Memory Arena* 还负责处理可以使用策略配置的边界检查和内存标记等调试工具。
+ 在内部，*Memory Arena* 在分配/释放原始内存时总是会引用一个分配器。

这意味着任何不同的分配器都遵循一个非常简单的接口，并且永远不需要知道是否启用了边界检查、内存跟踪或类似的功能。它唯一的工作就是分配/释放原始内存。

那么，线性分配器的接口就可如下所示：

```C++
class LinearAllocator
{
public:
	explicit LinearAllocator(size_t size);
	LinearAllocator(void* start, void* end);
	
	void* Allocate(size_t size, size_t alignment, size_t offset);
	
	inline void Free(void* ptr);
	
	inline void Reset(void);
	
private:
	char* m_start;
	char* m_end;
	char* m_current;
};
```

这里有几件事需要注意：

+ `Allocate()` 和 `Free()` 不是虚函数。分配器既可以被用来作为 *Memory Arena* 的一部分，也可以单独使用，用来分配“无类型”的内存（例如每帧的 *GPU* 命令缓冲）。在前一种情况下，*Memory Arena* 只是将分配器作为其基于策略的模板参数之一。在后一种情况下，对于像增长指针这样简单的操作，没有虚函数开销。
+ 其中一个构造函数接受一个内存范围 \[start，end)，这就是之后将用于分配的范围。它既可以是栈内存（固定大小或通过 `alloca()` 分配），也可以是堆内存（通过 `VirtualAlloc()` 之类的分配）。这允许我们既可以将容器或其他数据结构存在堆上，也可以暂时性地存在栈上，这是相当有用的功能。
+ 另一个构造函数在为给定大小分配物理内存时在内部使用页分配器。我们将在下一篇文章中讨论虚拟内存和物理页分配器，讨论像 `VirtualAlloc()` 这样的 *API* 函数。
+ 这种类型的分配器处理的内存区域永远不会增长。
+ 线性分配器不能释放单个分配，而是在调用 `Reset()` 时一次性释放所有分配。`Free()` 本质上是一个空函数，而 `Reset()` 只是将内部指针成员重置到起点。

唯一需要讨论的是 `Allocate()` 方法。可以看到，它接受第三个参数 `offset`，这可能会让你想知道它究竟是用来做什么的。

这样做的原因很简单：当然，我们希望能够以任意对齐方式分配任何给定大小的内存，但有时我们不希望分配器返回的内存被对齐，而是对齐到一些偏移量，因为我们内部需要在边界检查时添加标记数据或其他东西，比如边界字节。

让我们以边界检查为例来说明这种情况。假设我们想分配 120 个字节，对齐到 16 字节边界。若此时不进行边界检查，那么就可以直接调用 `linearAllocator.Allocate(120, 16, 0)`。但进行边界检查时，我们就要在被分配内存的头尾各增加 4 个字节的边界标记，这会让分配的需求增长到 128 字节。另外，我们仍想确保返回给用户的内存是 16 字节对齐的。如果没有额外的偏移量，我们将得到对齐到 16 字节边界的 128 字节，其中头尾各添加 4 字节的边界检查，并将结果指针返回给用户，这回在此过程中破坏对齐。这就是附加 `offset` 参数的作用：在这种情况下，我们可以简单地调用 `linearAllocator.Allocate(128, 16, 4)` 相当于说“给我分配 128字节，并确保返回的指针 +4 后对齐到 16 字节的边界”。

尽管这种对齐限制可以在 *Memory Arena* 内以一种通用的方式处理，但我决定让分配器来处理这个问题，因为它可以减少由于对齐而浪费的内存，并且在大多数情况下可以很容易地实现。

```C++
void* LinearAllocator::Allocate(size_t size, size_t alignment, size_t offset)
{
	// 先加上偏移量（留足余量）, 然后对齐它, 最后再减回来
	m_current = pointerUtil::AlignTop(m_current + offset, alignment) - offset;
	
	void* userPtr = m_current;
	m_current += size;
	
	if (m_current >= m_end)
	{
	    // out of memory
	    return nullptr;
	}
	
	return userPtr;
}
```

们所要做的就是首先添加偏移量，对齐结果指针，然后再次减去偏移量。这确保了适当的对齐和最少浪费的空间。

这基本上就是实现一个简单的线性分配器的全部内容。

在接下来的文章中，我们将研究一些事情，比如简单的基于栈（*LIFO*）的分配器，如何实现增长分配器、虚拟内存/页面分配器和池分配器。