*本文档译自 blog.molecular-matters.com 的 "Memory System" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
正如上次所承诺的，今天我们要来看看池分配器如何在 *O(1)* 复杂度下以任意顺序做到分配/释放指定大小的内存。


## 用途 - *Use case*
---
池分配器非常有助于分配/释放特定大小的对象，这些对象通常必须动态创建/销毁，例如武器子弹、实体实例、刚体等。

由于这些对象的动态特性，它们中的大多数都是以完全随机的顺序创建/销毁的。因此，希望能够以尽可能小的碎片来分配/释放内存。池分配器非常适合这种情况。


## 它是如何工作的 - *How does it work?*
---
简单地说，池分配器一次分配一块内存，并将该内存划分为正好适合 *M* 个大小为 *N* 的实例的块/箱/池（或之类的东西）。例如，假设我们同时最多有 256 个子弹，每个子弹的大小为 32 字节。因此，池分配器将一次性分配 256\*32 = 8192 个字节，将其划分为块，然后用于分配/释放大小为 32 字节的对象。

但这些分配是如何进行的呢？我们如何保证 *O(1)* 复杂度?？如何在没有碎片的情况下按任何顺序进行分配？


## 自由链表 - *Freelists*
---
上述所有问题的答案都可以归结为所谓的[自由列表](http://en.wikipedia.org/wiki/Free_list)的使用。自由列表在被分配的内存内部存储一个具有空闲槽的链表。将它们存储在适当位置是至关重要的，因为没有 `std::vector`、`std::list` 或类似的工具来跟踪空闲槽。它全部存储在那些预分配的池的内存中。

通常采用的方式如下：在每个槽（假设是 32 字节）所在的池内存里，将指向下一个槽的指针存储在槽的前几个字节里，以此来连接到下一个槽。

假设我们的内存池位于内存中的位置 *0x0*，那么布局将如下所示：

```
         +---------+---------+---------+---------+
         | 0x20    | 0x40    | 0x60    | nullptr |
         +---------+---------+---------+---------+
		
         ^         ^         ^         ^
         |         |         |         |
address: 0x0       0x20      0x40      0x60
```

上面的图形中，块表示内存中的槽，底部一行（*address*）显示内存中的地址。可以看到，*0x0* 的内存将包含一个指向 *0x20* 的指针，*0x20* 将包含一个指向 *0x40* 的指针，以此类推。这样，我们就在内存池中形成了一个侵入式链表。

这里有一点需要注意：只要槽是空闲的，我们就可以在这 32 个字节中存储任何我们想要的东西。当一个槽被使用时，我们不需要额外存储任何东西，因为这个槽已经被占用了，因此不再是空闲列表的一部分。我们所要做的就是每当自由列表被分配时移除一个槽，并在它被释放时再次将槽其添加到链表中。


## 实现 - *Implementation*
---
让我们看一下自由列表的内部工作原理：

```C++
class Freelist
{
public:
	Freelist(void* start, void* end, size_t elementSize, size_t alignment, size_t offset);
	
	inline void* Obtain(void);
	
	inline void Return(void* ptr);
	
private:
	Freelist* m_next;
};
```

我们需要存储的唯一成员是指向自由列表的指针，它在内存中充当别名，代表指向内存池中当前空闲槽的指针。

暂时不考虑对齐和偏移需求，初始化自由列表非常简单：

```C++
union
{
	void* as_void;
	char* as_char;
	Freelist* as_self;
};
 
// 假设 as_self 指向空闲列表中的第一个条目
m_next = as_self;
as_char += elementSize;
 
// 初始化自由列表 - 使每个元素的每个 m_next 都指向列表中的下一个元素
Freelist* runner = m_next;
for (size_t i = 1; i < numElements; ++i)
{
	runner->m_next = as_self;
	runner = as_self;
	as_char += elementSize;
}
 
runner->m_next = nullptr;
```

有了侵入式链表，分配/释放内存实际上变成了 *O(1)* 操作，操作上只是普通的链表代码：

```C++
inline void* Freelist::Obtain(void)
{
	// 检测是否有剩余
	if (m_next == nullptr)
	{
	    // 已经消耗完毕
	    return nullptr;
	}
	
	// 从自由列表的头部获取一个元素
	Freelist* head = m_next;
	m_next = head->m_next;
	return head;
}
 
inline void Freelist::Return(void* ptr)
{
	// 将返回的元素放在空闲列表的开头
	Freelist* head = static_cast<Freelist*>(ptr);
	head->m_next = m_next;
	m_next = head;
}
```

我将自由列表实现移到它自己的类中，以便池分配器的非增长和增长变体都可以使用它。此外，它对其他事情也很方便。


## 对齐要求 - *Alignment requirements*
---
为了满足对齐要求，我们需要将槽在内存池中偏移一次，这样所有槽都可以满足相同的偏移和对齐要求。这种方法的缺点是，一个池分配器永远无法满足多个不同的偏移量要求，但这种情况应该很少发生。

在 *Molecule* 中，实现的方式是用户在构造池分配器时必须提供最大对象大小和最大对齐要求。然后，分配器将满足所有 `对齐量 <= 最大对齐要求` 和 `对象大小 <= 最大对象大小` 的约束，并在所有其他情况下简单地进行断言。这至少能允许用户从同一个池中分配不同大小的对象（例如 4，8，12，16），如果需要，可以使用不同的对齐方式（例如 4，8，16，32）。

控制权掌握在用户手中，因此由用户可以决定使用不同的池进行不同的分配（通常推荐这样做），或者在内存池中使用一些浪费的内存。


## 用例 - *Usage*
---
例子相当简单。下面展示了一个自由列表，它能够满足最多 32 字节的分配和 8 字节的对齐。请注意，自由列表占用一定的内存空间，它在其中初始化自己。这确保了自由列表可以用于栈上、堆上以及不同分配器（非增长和增长）的分配。

```C++
ME_ALIGNED_SYMBOL(char memory[1024], 8) = {};
 
core::Freelist freelist(memory, memory + 1024, 32, 8, 0);
 
// 分配一个 32 字节的槽，与 8 字节的边界对齐
void* object0 = freelist.Obtain();
 
// 分配另一个 32 字节的槽，与 8 字节的边界对齐
void* object1 = freelist.Obtain();
 
// 获得的槽可以按任何顺序返回
freelist.Return(object1);
freelist.Return(object0);
```

上面代码中的池分配器只是将一个 *Freelist* 实例作为成员，并将所有 `Allocate()` 调用转发给 `Freelist.Obtain()`，并将所有 `Free()` 调用转发给 `Freelist.Return()`。此外，它断言分配大小和对齐请求符合用户提供的初始最大大小。


## 展望 - *Outlook*
---
池分配器是我们讨论的最后一个不增长的分配器。从下一篇文章开始，我们将研究如何通过虚拟内存和从操作系统分配物理页面的方式来实现增长的分配器。