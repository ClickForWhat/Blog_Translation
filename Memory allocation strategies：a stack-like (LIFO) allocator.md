*本文档译自 blog.molecular-matters.com 的 "Memory System" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
上一篇文章，我们讨论了线性分配器，它可能是所有内存分配器中最简单的。这一次，我们将详细介绍如何实现一个非增长的类似栈的分配器，以及经典的用例。

当谈论类栈分配器时，我指的是行为类似于栈数据结构的分配器，遵循后进先出（*LIFO*）原则。这与栈分配或堆分配、调用堆栈或类似的东西无关。


## 用例 - *Use case*
---
这样的类栈分配器对于按层次顺序（下称层级）的加载来说是非常方便的，在层级加载中需要进行大量的分配，并且它们都可以按反向顺序回滚。此外，它还可以用于做一些分配，这些分配需要在特定的生命周期中保持驻留，并在顶部添加额外的分配。有关更多用例和更详细的描述，请查看 *Jason Gregory* 撰写的优秀书籍[《*Game Engine Architecture*》](http://www.amazon.com/Game-Engine-Architecture-Jason-Gregory/dp/1568814135/ref=sr_1_1?ie=UTF8&qid=1346085331&sr=8-1&keywords=game+engine+architecture)。

类栈分配器最有用的扩展之一通常被称为双端堆栈分配器。它也遵循后进先出原则，但是可以从堆栈的两端进行分配，允许用户在堆栈的不同端进行不同类型的分配。这种分配器最著名的用例还是层级加载。

如果你不够小心，加载层级很容易导致大量碎片，因为在加载资源时需要进行临时分配。作为一个例子，考虑这样一种情况，加载资源需要一个临时缓冲区 A，将从这个临时缓冲区创建驻留的层级数据 B。你不能释放 A，因为你需要在 B 中生成数据，你也不能在 B 之后释放 A，因为我们需要保证后进先出的分配。这种碎片化的一个典型例子是读取 *XML/JSON* 文件，并从中创建/分配其他内容，然后尝试释放读取的文件。

这就是双端堆栈分配器发挥作用的地方。它所需要的只是从一端分配驻留数据，并从另一端分配临时数据。只要两端在加载过程中不相交或重叠，你就拥有与使用简单的堆栈分配器完全相同的可用内存量，但好处是根本没有碎片。


## 接口 - *Interface*
---
了解了为什么以及何时需要它，让我们尝试构建一个类栈分配器。该接口类似于上次的线性分配器：

```C++
class StackAllocator
{
public:
	StackAllocator(void* start, void* end);
	
	void* Allocate(size_t size, size_t alignment, size_t offset);
	
	void Free(void* ptr);
	
private:
	char* m_start;
	char* m_end;
	char* m_current;
};
```

我们的非增长分配器同样可以在栈和堆上使用，类似于我们上次看到的线性分配器。

当然，实现是不同的，因为我们需要能够以类似后进先出的方式释放分配。让我们暂时假设用户以正确的顺序调用 `Allocate()` 和 `Free()`，我们稍后会考虑添加调试检查。


## 实现 - *Implementation*
---
我们如何释放内存？非常简单，只需将内部缓冲区指针重置为分配之前的位置。首先要意识到的是，我们不能只使用用户提供的指针并将内部指针 `m_current` 设置到那个指针上，因为这可能导致部分未使用的内存将永远无法回收。

作为一个简单的例子，考虑以下内容：我们还没有进行任何分配，并且我们正处于首地址为 *0x4004* 的缓冲区。如果我们想要分配任何对齐到 16 字节边界的东西，我们将得到地址*0x4010*。如果这个地址稍后被释放，我们只是把当前的缓冲区指针指向这个地址，我们就丢失了 12 个字节。

因此，我们需要做的是将原始指针存储在每次分配的内存的附近。这很容易做到，只需将其存储在用户分配之前，并确保所有内容仍然正确对齐。在讨论线性分配器时，我们已经看到了如何做到这一点（使用 *offset* 参数）。

然后实现看起来如下所示：

 ```C++
namespace
{
	static const size_t SIZE_OF_ALLOCATION_OFFSET = sizeof(uint32_t);
	static_assert(SIZE_OF_ALLOCATION_OFFSET == 4, "Allocation offset has wrong size.");
}
 
void* StackAllocator::Allocate(size_t size, size_t alignment, size_t offset)
{
	// 在分配之前存储偏移值
	size += SIZE_OF_ALLOCATION_OFFSET;
	offset += SIZE_OF_ALLOCATION_OFFSET;
	
	// 记录当前的已分配内存的偏移值
	const uint32_t allocationOffset = safe_static_cast<uint32_t>(m_current - m_start);
	
	// 先偏移指针, 再对齐它, 最后把偏移减回来
	m_current = core::pointerUtil::AlignTop(m_current + offset, alignment) - offset;
	
	// 检查是否有充足空间
	if (m_current + size > m_end)
	{
		// out of memory
	    return nullptr;
	}
	
	union
	{
	    void* as_void;
	    char* as_char;
	    uint32_t* as_uint32;
	};
	// 记录当前的指针
	as_char = m_current;
	
	// 在前 4 个字节存储已分配内存的偏移值
	*as_uint32 = allocationOffset;
	as_char += SIZE_OF_ALLOCATION_OFFSET;
	
	// 返回给用户的指针
	void* userPtr = as_void;
	m_current += size;
	return userPtr;
}
```

这里有几件事值得注意：

+ 每次分配都会多产生 4 字节的开销（`sizeof(uint32_t)`），因为在每次分配之前存储偏移量。
+ 这里存储偏移量而不是指针，因为这样可以最小化 64 位系统上的开销，这是基于单个类栈分配器不会分配超过 *4GB* 内存的假设。
+ `size` 和 `offset` 都需要增加 4 字节。这确保了传递给用户的指针将被正确对齐，而不是对齐我们内部用于存储缓冲区开始的偏移量的地址。

现在，释放分配相当简单，从被分配内存的上 4 个字节中获取偏移量，并将内部缓冲区指针设置为该偏移量：

```C++
void StackAllocator::Free(void* ptr)
{
	ME_ASSERT_NOT_NULL(ptr);
	
	union
	{
	    void* as_void;
	    char* as_char;
	    uint32_t* as_uint32;
	};
	
	as_void = ptr;
	
	// grab the allocation offset from the 4 bytes right before the given pointer
	as_char -= SIZE_OF_ALLOCATION_OFFSET;
	const uint32_t allocationOffset = *as_uint32;
	
	m_current = m_start + allocationOffset;
}
```


## 检查错误调用 - *Checking for erroneous calls*
---
除此之外，你还可以在 *Molecule* 中启用类栈分配器的错误检查，它将断言对 `Allocate()` 和 `Free()` 的调用总是按照 *LIFO* 的顺序进行。

这是通过将分配的 *ID* 存储在分配前面的额外 4 个字节中来实现的，类似于我们存储基本偏移量的方式。*ID* 只不过是一个计数器，在每次调用 `Allocate()` 时递增，在每次调用 `Free()` 时递减。每当调用 `Free()` 时，只需获取分配 *ID* 并将其与当前分配 *ID* 进行比较，断言它们是相同的。


## 展望 - *Outlook*
---
下一篇文章，我们将看到池分配器如何在 *O(1)* 时间内以任意顺序分配一定大小的对象，而不会出现碎片。