*本文档译自 blog.molecular-matters.com 的 "Memory System" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
让我们从上次结束的地方继续。现在，我想讨论如何使用虚拟内存系统构建可以不断增长的分配器。这篇文章描述了如何构建一个类栈的分配器，它可以自动增长到给定的最大大小。

正如在上一篇文章中所解释的那样，虚拟内存允许我们保留地址空间，而无需为其分配任何物理内存。我们可以利用这一点，构建一个预先保留地址空间的分配器，并在需要时分配物理内存。在这方面，分配器的行为类似于不增长的堆栈分配器，但不会受限于在固定的内存区域内工作。

这样的分配器在加载层级数据等情况下非常有用，不同的层级需要的内存量是不同的。使用一个不断增长的分配器可以节省开发时间，因为当某个层级的数据超过以前的最坏的内存使用情况时，不需要不断更改固定大小分配器的上限。


## 例子 - *Example*
---
考虑一款游戏，其中用于关卡驻留资源的最大内存被定义为 *128MB*。我们想要的是一个分配器，它能保留 *256MB* 的地址空间（为了安全起见），并在需要时以 *1MB* 的块分配物理内存。

构建这样一个分配器的基本步骤很简单：

+ 保留 *256MB* 的地址空间。
+ 存储保留地址空间的起始位置和结束位置。
+ 存储物理地址空间的起始位置和结束位置（起始 = 0字节）。
+ 对于每次内存分配：
	+ 计算出所需的物理内存量。
	+ 如果仍然有足够的物理内存剩余，则将物理地址空间的起始位置偏移（见上文）。
	+ 如果没有足够的内存，在当前物理地址空间的末尾分配 *1MB* 的页面，直到分配合适为止。
	+ 更新物理地址空间的开始和结束。

用代码表示的话，就像这样：

```C++
GrowingStackAllocator::GrowingStackAllocator(unsigned int maxSizeInBytes, unsigned int growSize)
  : m_virtualStart(virtualMemory::ReserveAddressSpace(maxSizeInBytes))
  , m_virtualEnd(m_virtualStart + maxSizeInBytes)
  , m_physicalCurrent(m_virtualStart)
  , m_physicalEnd(m_virtualStart)
  , m_growSize(growSize)
{
}
```

在构造函数中，我们只保留地址空间并存储它的开始和结束。


## 分配内存 - *Allocating memory*
---
```C++
void* GrowingStackAllocator::Allocate(size_t size, size_t alignment, size_t offset)
{
	const size_t allocationSize = size;
	
	// 计算出合适的分配大小和可能的偏移量
	...
	
	m_physicalCurrent = core::pointerUtil::AlignTop(m_physicalCurrent + offset, alignment) - offset;
	
	// 检测是否有足够的物理空间剩余
	if (m_physicalCurrent + size > m_physicalEnd)
	{
	    // 物理内存不足。检查地址空间是否还有剩余，可以从中分配新的物理页。
	    // 请记住，虚拟内存必须始终以页面大小的块来分配，这就是为什么我们将所需的
	    // 大小四舍五入到 m_growSize 的倍数。
	    const size_t neededPhysicalSize = bitUtil::RoundUpToMultiple(size, m_growSize);
	    if (m_physicalEnd + neededPhysicalSize > m_virtualEnd)
	    {
		    // 分配无法塞进地址空间，内存不足
		    return nullptr;
	    }
		
	    // 在当前分配的页面末尾分配新的内存页面
	    virtualMemory::AllocatePhysicalMemory(m_physicalEnd, neededPhysicalSize);
	    m_physicalEnd += neededPhysicalSize;
	}
	
	// 必要时存储分配信息
	...
	
	// 返回分配的内存地址给用户
	return m_physicalCurrent;
}
```

如上所述，只要没有足够的物理内存来分配，我们就分配新的页面。在这种情况下，新页面将在已经提交了物理内存的地址范围的末尾分配。如果在专用于此分配器的地址空间中没有足够的剩余空间，则内存不足。

分配器增长的粒度由用户在构造函数中指定。这是关于分配性能与浪费内存的典型性能/内存的权衡。如果分配器以 *64KB* 的页面增长，它必须通过虚拟内存系统进行许多小的分配，但最多只浪费 *64KB*，这并不多。相比之下，以 *1MB* 的页面增长将进行更少次数的分配，但可能会浪费近 *1MB* 的内存，这些内存被物理内存提供，但从未使用过。


## 释放内存 - *Freeing memory*
---
还有一件事需要讨论，那就是分配器在释放内存时的行为。要考虑的主要问题是，若在某一时间释放了足够多的内存，分配器是否应该向系统返还物理内存页。根据分配行为，这可能是好的，也可能是坏的。

在释放层级驻留资源时，通常这是几个大的分配，并且我们希望尽快释放物理内存。在使用分配器进行许多小分配的情况下，保留提交给地址空间的物理内存可能更有益。否则，每当分配跨越页面边界时，导致页面不断被分配、释放、再分配，分配器就会遭受性能损失。

在 *Molecule* 中处理这个问题的方式是，无论何时释放分配，分配器都不会将物理内存返回给操作系统。相反，不再需要的物理内存必须通过调用 `Purge()` 显式释放，这是自增长分配器提供的额外方法。在堆栈分配器不断增长的情况下，它只是检查哪些页面不再需要，并将它们返回给操作系统：

```C++
void GrowingStackAllocator::Purge(void)
{
	// 我们需要释放任何分配不再需要的所有物理内存页面，
	// 同时确保我们不会释放当前指向的页面（请记住，虚拟内存仅在页面大小粒度中工作）。
	// 我们通过将当前物理内存指针舍入到下一个增长大小边界来实现这一点，
	// 并从中释放所有物理内存。
	char* addressToFree = pointerUtil::AlignTop(m_physicalCurrent, m_growSize);
	const unsigned int sizeToFree = safe_static_cast(
		m_physicalEnd - addressToFree);
	virtualMemory::FreePhysicalMemory(addressToFree, sizeToFree);
	
	m_physicalEnd = addressToFree;
}
```


## 总结 - *Conclusion*
---
有了不断增长的利用虚拟内存系统的分配器，我们就可以在不浪费物理内存的情况下，为开发过程中所需的内存量指定一个安全的上限。此外，知道某些分配在哪个地址范围内可以在调试时提供巨大的帮助。