*本文档译自 bitsquid.blogspot.com 的 "Building a Data-Oriented Entity System" 系列文章，作者 Niklas Frykholm*


## 概述 - *Overview*
----
在上一篇文章中，我谈到了实体管理器的设计以及我们如何处理游戏实体的创建和销毁。

在这篇文章中，我们将看看组件是如何实现的。

简要回顾一下：我们系统中的组件不是单独的对象，相反，特定类型的组件都由该类型的组件管理器处理。组件管理器完全控制组件如何在内部存储数据以及如何更新数据。


## 组件案例 - *A Component Example*
---
为了更进一步讨论，我们虚构一个组件来处理点质量（*point mass*）对象。对于每个组件实例，我们希望存储以下数据：

```C++
Entity entity;          ///< Entity owner
float mass;             ///< Mass of object
Vector3 position;       ///< Object's position
Vector3 velocity;       ///< Object's velocity
Vector3 acceleration;   ///< Object's acceleration
```

组件需要一些函数来访问数据和进行物理模拟。

我们为什么要存储组件的所有者实体，这可能不是不言自明的，但稍后它将派上用场。

注意，这不是一个真实的例子。我们的引擎中并没有这样的组件，也许这不是最好或最有趣的设计，但它给了我们一些值得讨论的东西。


## 组件数据布局 - *Component Data Layout*
----
在考虑如何在组件管理器中布局数据时，我们有两个目标：

+ 给定一个实体，我们希望能够快速查找该实体的组件数据。
+ 我们希望组件数据紧密地封装在内存中，以获得良好的缓存性能。

让我们先看看第二个目标。

实际的缓存性能取决于CPU的工作方式以及代码中的数据访问模式。你可以花很多时间试着去围绕它好好想想，但这里有一个简单的经验法则：

**将数据打包到顺序访问的数组中。**

只有当你试图修复因此诊断出的性能问题时，再让它更复杂。

通常比较好的方法是使用 *SoA* 的方式。也就是说，每个字段单独成为在内存中的数组，每个组件实例有一个条目。

```
[entity_1]  [entity_2]  [entity_3] ... // array1
[mass_1]    [mass_2]    [mass_3]   ... // array2
[pos_1]     [pos_2]     [pos_3]    ... // array3
[vel_1]     [vel_2]     [vel_3]    ... // ...
[acc_1]     [acc_2]     [acc_3]    ...
```

将每个字段单独存储的好处是，只处理某些字段的代码不必在其他字段上浪费宝贵的缓存空间。

你甚至可以更进一步，把 `Vector3` 的每个 *x*, *y* 和 *z* 分量分别放入它自己的数组中。这样做的一个好处是，依借此法，你可以进行更有效的 *SIMD* 计算。但是对于本例，让我们将事情简化一点，只把 `Vector3` 存储在一起。由于数据的布局完全封装在 *ComponentManager* 类中，如果我们需要更苛刻的性能，我们总是可以回头重新设计它。

实现这种数据布局的最简单方法是为每个组件使用 *Array*：

```C++
class PointMassComponentManager {
    struct InstanceData {
        Array<Entity> entity;
        Array<float> mass;
        Array<Vector3> position;
        Array<Vector3> velocity;
        Array<Vector3> acceleration;
    };
    InstanceData _data;
};
```

这种方法工作得很好，但它确实意味着数据被存储在五个单独分配的内存缓冲区中。所以我用了不同的方法。我直接分配一整个内存缓冲区，然后让 *entity*，*mass* 等，指向该缓冲区的不同部分。

```C++
struct InstanceData {
    unsigned n;          ///< Number of used instances.
    unsigned allocated;  ///< Number of allocated instances.
    void *buffer;        ///< Buffer with instance data.
	
    Entity *entity;
    float *mass;
    Vector3 *position;
    Vector3 *velocity;
    Vector3 *acceleration;
};
InstanceData _data;

void allocate(unsigned sz)
{
    assert(sz > _data.n);
	
    InstanceData new_data;
    const unsigned bytes = sz * (sizeof(Entity) + sizeof(float) +
        3 * sizeof(Vector3));
    new_data.buffer = _allocator.allocate(bytes);
    new_data.n = _data.n;
    new_data.allocated = sz;
	
    new_data.entity = (Entity *)(new_data.buffer);
    new_data.mass = (float *)(new_data.entity + sz);
    new_data.position = (Vector3 *)(new_data.mass + sz);
    new_data.velocity = new_data.position + sz;
    new_data.acceleration = new_data.velocity + sz;
	
    memcpy(new_data.entity, _data.entity, _data.n * sizeof(Entity));
    mempcy(new_data.mass, _data.mass, _data.n * sizeof(float));
    memcpy(new_data.position, _data.position, _data.n * sizeof(Vector3));
    memcpy(new_data.velocity, _data.velocity, _data.n * sizeof(Vector3));
    memcpy(new_data.acceleration, _data.acceleration,
        _data.n * sizeof(Vector3));
	
    _allocator.deallocate(_data.buffer);
    _data = new_data;
}
```

这避免了任何可能存在于 *Array* 类中的隐藏开销，并且我们只需要跟踪一次分配。这对缓存和内存分配系统都更好。

旁注：我打算编写一个分配粒度为 *4K* 的内存系统。也就是说，没有传统的堆分配器，只有一个页分配器，你必须设计你的系统，使它们只适用于大的分配。


## 访问数据 - *Accessing Data*
---
让我们开始考虑第一个问题，即如何从实体映射到它的组件数据。为了简单起见，我们现在假设每个实体不支持多个组件。

在数据布局内部，我们通过 *mass*、*position* 等数组的索引来引用特定组件实例。我们需要一种从实体映射到对应索引的方法。

你可能还记得上次文章中，每个实体拥有一个独一无二的 *ID*。一种选择是直接用这个作为下标。

如果游戏中的所有实体都拥有这一组件，这便是一种不错的方法。但如果不是这种情况，我们的数组将包含许多与缺少组件的实体相对应的内存空洞。这将浪费内存，但也会浪费性能，因为我们将用空白数据填充缓存。

我们可以通过使用一定程度的间接性来改进这一点：

```C++
Array<unsigned> _map;
```

这里，`_map` 允许我们根据实体 *ID* 查找组件索引。这样好多了，因为现在只有 `_map` 数组有空洞，而不是数据数组有，这意味着洞越来越少。

但是，只有当我确定组件几乎是通用的，并且查找对性能至关重要时，我才会使用这种方法。在大多数情况下，我认为散列索引是一种更好的方法：

```C++
HashMap<Entity, unsigned> _map;
```

这样使用的内存更少，而且查找仍然相当快。由于从实体到组件实例索引的查找涉及一个额外的步骤，我们希望在API中反映这一点，而不是强迫用户在访问同一组件的不同字段时进行多次查找。像这样：

```C++
/// Handle to a component instance.
struct Instance { int i; };

/// Create an instance from an index to the data arrays.
Instance make_instance(int i) { Instance inst = {i}; return inst; }

/// Returns the component instance for the specified entity or a nil instance
/// if the entity doesn't have the component.
Instance lookup(Entity e) { return make_instance(_map.get(e, 0)); }

float mass(Instance i) { return _data.mass[i.i]; }
void set_mass(Instance i, float mass) { _data.mass[i.i] = mass; }
Vector3 position(Instance i) { return _data.position[i.i]; }
...
```

为了支持每个实体有多个组件实例，你可以在组件数据中添加 *next* 实例字段，该字段允许你遍历属于同一实体的组件实例的链表。这留给读者作为练习。


## 组件更新 - *Component Updates*
---
由于组件数据按顺序排列在内存中，因此编写一个为所有实体模拟物理的函数非常简单：

```C++
void simulate(float dt)
{
    for (unsigned i = 0; i < _data.n; ++i) {
        _data.velocity[i] += _data.acceleration[i] * dt;
        _data.position[i] += _data.velocity[i] * dt;
    }
}
```

这个函数按顺序遍历内存，这为我们提供了良好的缓存性能。如果需要的话，它也很容易进行轮廓化、矢量化和并行化。

附带评论：我有点讨厌被称为 `update()` 的方法。这是基于继承的设计遗留下来的糟糕物件。如果你花点时间考虑一下，你几乎总能想出比 `update()` 更好、更有信息量的名字。


## 销毁组件 - *Destroying Components*
---
在销毁组件时，我们要确保数据数组保持紧密封装。我们可以通过将最后一个元素移动到我们想要删除的组件的位置来实现这一点。我们还必须为相应的实体更新 `_map` 中的索引映射。

```C++
void destroy(unsigned i)
{
    unsigned last = _data.n - 1;
    Entity e = _data.entity[i];
    Entity last_e = _data.entity[last];
	
    _data.entity[i] = _data.entity[last];
    _data.mass[i] = _data.mass[last];
    _data.position[i] = _data.position[last];
    _data.velocity[i] = _data.velocity[last];
    _data.acceleration[i] = _data.acceleration[last];
	
    _map[last_e] =  i;
    _map.erase(e);
	
    --_n;
}
```

另一个问题是，当实体被销毁时，我们如何处理组件的销毁。你可能还记得，实体没有它所拥有的组件列表。此外，当实体销毁时，如果要求用户自行手动销毁正确的组件，看起来又很麻烦。

我们将使用两种方法。

需要立即销毁的组件（可能是因为它们持有外部资源）可以向 *EntityManager* 注册销毁回调，当实体被销毁时将调用该回调。

然而，对于更简单的组件，比如点质量组件，并不要求组件与实体必须同时被销毁。我们可以利用这一点，并使用垃圾收集的方法来惰性地销毁组件，而不是花费内存和精力来存储回调列表：

```C++
void gc(const EntityManager &em)
{
    unsigned alive_in_row = 0;
    while (_data.n > 0 && alive_in_row < 4) {
        unsigned i = random_in_range(0, _data.n - 1);
        if (em.alive(_data.entity[i])) {
            ++alive_in_row;
            continue;
        }
        alive_in_row = 0;
        destroy(i);
    }
}
```

在这里，我们随机选取组件索引，如果对应的实体已被销毁，则销毁它们。这样做，直到连续击中四个实体。

这段代码的优点是，如果没有被销毁的实体（只有四次循环），它几乎不需要任何成本。但是当存在大量被破坏的实体时，组件将很快被销毁。

在下一篇文章中，我们将介绍处理父实体和子实体之间链接的 *Transform* 组件。