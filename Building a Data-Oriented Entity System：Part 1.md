*本文档译自 bitsquid.blogspot.com 的 "Building a Data-Oriented Entity System" 系列文章，作者 Niklas Frykholm*


## 概述 - *Overview*
----
我们最近打算把实体组件系统加入到 *Bitsquid* 引擎里。

你可能有点惊讶 *Bitsquid* 引擎竟然不是基于组件设计的。但实际上，这方面的需求并不大。因为游戏玩法代码通常是用 Lua 而不是 C++ 编写的，所以我们不会遇到常见的复杂、深层次的继承结构问题，这些问题才会促使人们转向基于组件的设计。当然，在引擎内部中很少使用继承。

但当我们扩展我们的插件系统时，我们需要一种方法让 C++ 插件赋予游戏对象新的功能和能力。这使得组件体系结构非常适合。


## 实体和组件 - *Entities and Components*
---
在 *Bitsquid* 引擎中，我们一直努力保持系统间的解耦和面向数据设计，我们希望在组件架构中使用相同的方法。因此，在我们的系统中，实体（*Entity*）不是某种堆分配的对象。相反，实体只是一个整数，一个标识特定实体的唯一 *ID*：

```C++
struct Entity
{
	unsigned id;
};
```

*EntityManager* 是一个特殊的类，用于跟踪活动的实体。

组件也不是对象。相反，组件是由 *ComponentManager* 处理的东西。*ComponentManager* 的任务是将实体与组件关联起来。例如，*DebugNameComponentManager* 可用于将调试名称与实体关联起来：

```C++
class DebugNameComponentManager
{
public:
	void set_debug_name(Entity e, const char *name);
	const char *debug_name(Entity e) const;
};
```

关于这种解耦设计，有两点值得注意。

首先，在这个设计中没有 *DebugNameComponent* 类来处理单个 *DebugName* 组件。这是不需要的，因为所有组件数据都是由 *DebugNameComponentManager* 内部管理的。管理器可以决定在内部使用堆分配的 *DebugNameComponent* 对象。但并不是一定要这样，通常，以另一种方式排列数据会更有效。例如，作为单个连续缓冲区中的数组结构。在以后的文章中，我将展示一些这样的例子。

其次，我们不保存实体拥有的所有组件的列表。只有 *DebugNameComponentManager* 知道一个实体是否有一个 *DebugName* 组件，如果你想操作那个组件，你必须通过 *DebugNameComponentManager* 来做。不存在所谓的“抽象”组件。

所以一个实体拥有什么组件只能通过在不同的组件管理器中注册组件来定义。因此插件可以借助新的组件管理器来扩展系统。

一个实体可能拥有多个同类型的组件，但这是否有意义由组件管理器决定。例如， *DebugNameComponentManager* 只允许一个 *DebugName* 与一个实体相关联。但是*MeshComponentManager* 允许一个实体拥有多个网格。

管理器负责执行更新组件所需的任何计算。更新是指更新一个组件管理器，而不是更新一个实体，当组件管理器更新时，它会一次更新它的所有组件。这意味着通用的计算过程聚集在一起，并且所有数据都在缓存中。它还使我们更易分析 *Update* 的性能表现、更易配置多线程或将负载平衡到外部处理器。所有这些都将转化为巨大的性能优势。


## 实体管理器 - *The EntityManager*
---
我们希望能够使用实体的 *ID* 作为弱引用。也就是说，给定一个实体的 *ID*，我们希望能够判断它是否指的是一个活生生的实体。

拥有一个弱引用系统是很重要的，因为如果我们只有强引用，那么如果实体销毁，我们必须通知所有可能持有该实体引用的人，以便他们可以删除它。这既昂贵又麻烦，特别是在引用可能由其他线程或 Lua 代码持有的情况下。

为了开始使用弱引用，我们使用 *EntityManager* 类来跟踪所有活动的实体。最简单的方法就是使用集合：

```C++
class EntityManager
{
	HashSet<Entity> _entities;
	Entity _next;
public:
	Entity create()
	{
		++_next.id;
		while (alive(_next))
			++_next.id;
		_entities.insert(_next);
		return _next;
	}
	
	bool alive(Entity e)
	{
		return _entities.has(e);
	}
	
	void destroy(Entity e)
	{
		_entities.erase(e);
	}
};
```

这非常好，但是因为 `alive()` 函数是被频繁调用的核心代码，所以我们希望它的运行速度比设置/插入更快。通过将实体 *ID* 拆分为 *index* 和 *generation* 部分，我们可以将其更改为简单的数组查找：

```C++
const unsigned ENTITY_INDEX_BITS = 22;
const unsigned ENTITY_INDEX_MASK = (1 << ENTITY_INDEX_BITS) - 1;

const unsigned ENTITY_GENERATION_BITS = 8;
const unsigned ENTITY_GENERATION_MASK = (1 << ENTITY_GENERATION_BITS) - 1;

struct Entity
{
	unsigned id;
	
	unsigned index() const { return id & ENTITY_INDEX_MASK; }
	unsigned generation() const {
		return (id >> ENTITY_INDEX_BITS) & ENTITY_GENERATION_MASK;
	}
};
```

这里的想法是 *index* 部分直接能获取实体在数组中的索引/下标。*generation* 部分用来区分在同一个位置上创建的实体。当我们创建以及销毁实体时，我们可能会在数组的同一个位置进行。我们可以在创建或销毁时，通过改变 *generation* 的值来获得一个独一无二的 *ID*。

在我们的系统中，我们限制使用30位作为实体 *ID*。这样做的原因是我们需要将它放入一个32位的指针中，以便能够使用 Lua 的 *light userdata* 来存储它。我们还需要从这个指针中窃取两个比特，以便将它与我们在引擎中使用的其他类型的轻用户数据区分开来。

如果你没有这个限制，或者你只针对64位平台，那么使用更多的位作为 *ID* 可能是个好主意。

我们把30位分割，其中22位作为 *index*，8位作为 *generation*。这意味着我们最多支持400万个同时存在的实体。这也意味着我们只能区分在同一数组位置上创建的256个不同的实体。如果在相同的位置中创建了超过256个实体，则 *generation* 的值将溢出并回到原值，并且我们的新实体将获得与旧实体相同的 *ID*。

为了防止这种情况经常发生，我们需要确保我们不会太频繁地重用相同的位置。有很多种可能的方法。我们的解决方案是将回收的索引放在队列中，并且仅在该队列包含至少 `MINIMUM_FREE_INDICES = 1024` 项时重用该队列中的值。因为我们有256个 *generation*，所以ID将永远不会重新出现，直到它的索引在队列中运行256圈。因此，这意味着必须创建和销毁至少256 * 1024个实体，直到 *ID* 能够重新出现。这似乎是相当安全的，但如果你愿意，你可以多分配些位来继续提高安全性。例如，如果你不需要400万个实体，你可以从 *index* 中的一些 *bit* 分给 *generation*。

*generation* 只有8位的好处是，*lookup* 数组中每个实体只需要8位的空间。这节省了内存，也为我们提供了更好的性能，因为我们可以在缓存中容纳更多数据。使用此解决方案，*EntityManager* 的代码变成：

```C++
class EntityManager
{
	Array<unsigned char> _generation;
	Deque<unsigned> _free_indices;
	
public:
	Entity create()
	{
		unsigned idx;
		if (_free_indices.size() > MINIMUM_FREE_INDICES) 
		{
			idx = _free_indices.front();
			_free_indices.pop_front();
		} 
		else 
		{
			_generation.push_back(0);
			idx = _generation.size() - 1;
			XENSURE(idx < (1 << ENTITY_INDEX_BITS));
		}
		return make_entity(idx, _generation[idx]);
	}
	
	bool alive(Entity e) const
	{
		return _generation[e.index()] == e.generation();
	}
	
	void destroy(Entity e)
	{
		const unsigned idx = e.index();
		++_generation[idx];
		_free_indices.push_back(idx);
	}
};
```

在下一篇文章中，我们将介绍组件的设计。