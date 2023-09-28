*本文档译自 bitsquid.blogspot.com 的 "Building a Data-Oriented Entity System" 系列文章，作者 Niklas Frykholm*


## 概述 - *Overview*
----
在上一篇文章中，我大致讨论了组件的设计。今天，我将集中讨论一个特定的组件，*TransformComponent*。


## Transform 组件 - *The Transform Component*
---
*TransformComponent* 的目的是允许实体在世界中定位。它处理实体在世界中的定位和父子级关系。例如，你可能希望将车轮实体链接到汽车实体，以便车轮在移动时跟随汽车。从这个意义上说，*TransformComponent* 就是构成引擎世界场景的组件。


## 设计目标 - *Design Decisions*
---
#### 所有实体都应该有 Transform 组件吗？

在某些引擎中，每个实体都必须有一个 *Transform* 组件，即使它只是一个纯粹的逻辑实体，在世界中没有真正的位置。对我来说，强迫一个实体拥有一个没有真正意义的位置似乎很奇怪。我还希望实体尽可能没有昂贵的代价。因此，最好将 *Transform* 组件设为可选的，就像其他组件一样。没有 *Transform* 组件的实体在世界中没有任何位置。

实际上，谈论这个世界有点用词不当。*Bitsquid* 引擎并没有一个所有东西都必须生存的单一世界。相反，你可以创建多个世界，每个世界都有自己的对象。所以你可以为主游戏中设置一个世界，为暂时用不到的物体设置一个世界，为屏幕加载再设置一个世界，等等。

这比在主游戏世界的某个秘密隐藏地点设置某种“库存室”要好得多。

每个世界都有自己的 *TransformComponent* 管理器，如果需要，实体可以在其中的几个管理器中创建 *Transform* 组件。所以相同的实体可以存在于不同的游戏世界中，并被放置在不同的地方。由于 *MeshComponent* 管理器也存在于每个世界中，实体在每个世界中可以有不同的图形表示。

这有点深奥，我不希望很多实体使用它，但在某些情况下它可能会很有趣。例如，玩家的鹤嘴锄可以同时存在于主游戏世界和库存室世界中，但仍然作为同一实体进行管理。

#### 实体结构图和模型结构图

在我们的设计中，我们需要处理两种结构图（*scene graph*）。

第一种是我们已经讨论过的，即由实体及其关联子实体构成的图。

第二种是实体内的节点图。例如，一个角色实体可能有一个包含数百个骨骼的模型，这些骨骼有单独的动画。

这两张图之间应该有什么关系？

在以前的引擎代码中，我一直将这两个图视为同一系统的一部分。将模型结构图链接到实体结构图中的节点，并计算它们在世界空间中的变换。这将创建一个更新顺序依赖项。因为在实体结构图计算出世界位置之前，我们无法计算模型结构图的世界位置。这限制了我们可以并行做的事情。

我决定将这两个概念解耦。模型结构图不会计算世界空间的姿态（位置、旋转等），而是计算相对于实体姿态的姿态。这意味着我们可以在不了解实体姿态的情况下评估动画并计算模型姿态（当然，忽略了世界空间的限制，但它们将在稍后的过程中处理）。

当然，它还要求我们将模型的节点变换与实体变换相乘，以获得模型节点实际的世界位置。

我还没有完成模型结构图的设计，但也许我会有机会在以后的帖子中回到这个问题上。

#### 立即更新还是延迟更新

在以前的引擎中，我总是将世界变换延迟更新。例如，更改节点的局部 *Transform* 不会立即更新其世界 *Transform*（以及子节点的世界 *Transform*）。相反，它将简单地在实体中设置一个脏标志。稍后，我将计算所有脏节点（及其子节点）的世界 *Transform*，作为一个单独的过程。

这样做的优点是，我们永远不会多次计算一个节点的世界 *Transform* （在一次 *update* 中）。

考虑最坏的情况，一个很长的节点链：

```
[ node_1 ] ---> [ node_2 ] ---> [ node_3 ] ---> ... ---> [ node_n ]
```

使用延迟更新并只改变每个节点的局部姿态的办法，我们仍然只需要 *O(n)* 次计算来计算所有的世界变换。对于即时更新，我们在父变换发生变化时立即计算所有子变换的世界变换，这样将需要 *O(n^2)* 次计算。

不过另一方面，使用延迟更新有一个缺点。每当我们请求一个对象的世界位置时，我们不会得到它的实际世界位置，而是它在上一帧的世界位置（除非我们在真正实施变换之后请求）。这可能会导致许多混乱和微妙的错误。解决这些问题通常需要一些丑陋的技巧，比如在不同时间强制更新结构图。

所以我们应该选哪一个？

我认为，通过将模型结构图与实体结构图解耦，即时更新的性能问题就不那么严重了。在模型结构图中肯定有同时运动的长链节点（想想一个角色挥舞鞭子的动画）。但是我猜想，同时移动的长链对象在实体结构图中不太常见。

注意，如果只是根实体在移动，则不会出现性能问题。在这种情况下，立即更新和延迟更新都是 *O(n)*。只有当父节点和子节点同时移动时，即时更新的效果才会变差。

我预期没有很长的实体链（n <= 5 ??），我也预期这些链中的所有对象不会同时移动。所以我决定采用即时更新，这样我们总是有准确的世界转换。

注意：如果我们因此遇到性能问题，我们可以创建一个API函数，允许我们在执行单个世界 *Transform* 更新时同时设置多个局部 *Transform*，从而获得 *O(n)* 性能。

#### 关于延迟更新的旁注

请注意，如果希望执行延迟更新，则需要对实体数组进行排序，以便父实体始终出现在子实体之前。这样你就可以从头到尾遍历数组进行计算，并保证父实体的世界变换在计算其子实体的世界变换之前已经计算过了。

此外，你肯定不会希望遍历整个数组来查找脏对象：

```C++
for (int i = 0; i < n; ++i) {
    if (dirty[i])
        transform(i);
}
```

通常情况下，在一个场景中，每次只有一小部分物体在移动（可能只有1%）。所以循环遍历所有对象，即使只是检查标志，也会浪费很多时间。一个更好的解决方案是将所有脏对象排序到数组的末尾，这样我们就可以只遍历它们：

```C++
for (int i = first_dirty; i < n; ++i)
    transform(i);
```

因为我们只需要对数组进行部分排序，所以我们不需要运行昂贵的 *O(n log n)* 排序算法（否则为了避免 *O(n)* 的更新而运行 *O(n log n)* 的排序，这有点本末倒置）。相反，我们可以通过交换来实现这一点。

当一个节点（*D*）变脏时，我们通过将它与脏列表前的一个元素（*X*）交换将其移动到脏列表的首位，并将 `first_dirty` 减一：

```
Before:
                                =============== dirty ===============
|   |   |   | D |   |   |   | X |   |   |   |   |   |   |   |   |   |

After:
	                        ================= dirty =================
|   |   |   | X |   |   |   | D |   |   |   |   |   |   |   |   |   |
```

我们对节点的所有子节点做同样的处理，以此类推。当我们处理脏数组中的项时，每当我们发现子元素的父元素位于数组的较后位置时，我们就交换子元素和父元素，确保父元素先于子元素处理：

```
                            ================= dirty =================
|   |   |   |   |   |   |   |   |   |   | C |   |   | P |   |   |   |
                                          ^

                            ================= dirty =================
|   |   |   |   |   |   |   |   |   |   | P |   |   | C |   |   |   |
                                          ^
```

我们还需要一种方法来将元素从脏列表中删除，否则它将无限期地继续增长。我们可以清除每一帧的列表，但随着元素在列表中的移入和移出，这可能会导致大量的交换。一个更好的方法可能是检查一个元素是否在五帧左右的时间内没有移动，在这种情况下，我们将其从脏列表中删除。这样可以避免交换那些总是在移动的元素。

当使用即时更新策略时，对列表进行排序并不那么重要，但是我们可以使用类似的交换策略来确保父节点及其子节点在数组中保持在一起，以便即时更新是缓存友好的。


## 实现 - *Implementation*
---
有了完整的设计思想，实现就没有什么问题了。就像在上一篇文章中一样，我们将所有实例的 *Transform* 组件数据存储在单个大内存块中：

```C++
struct Instance { int i; };

/// Instance data.
struct InstanceData {
    unsigned size;              ///< Number of used entries in arrays
    unsigned capacity;          ///< Number of allocated entries in arrays
    void *buffer;               ///< Raw buffer for data.
	
    Entity *entity;             ///< The entity owning this instance.
    Matrix4x4 *local;           ///< Local transform with respect to parent.
    Matrix4x4 *world;           ///< World transform.
    Instance *parent;           ///< The parent instance of this instance.
    Instance *first_child;      ///< The first child of this instance.
    Instance *next_sibling;     ///< The next sibling of this instance.
    Instance *prev_sibling;     ///< The previous sibling of this instance.
};
```

`parent` 数组、`first_child` 数组、`next_sibling` 数组和 `prev_sibling` 数组都存储实例索引。我们可以通过跟踪 `first_child` 和它的 `next_sibling` 来找到特定实体的所有子元素。

我们可以用它来做即时更新：

```C++
void TransformComponent::set_local(Instance i, const Matrix4x4 &m)
{
    _data.local[i.i] = m;
    Instance parent = _data.parent[i.i];
    Matrix4x4 parent_tm = is_valid(parent) ? 
	    _data.world[ parent.i ] : matrix4x4_identity();
    transform(parent_tm, i);
}

void TransformComponent::transform(Instance i, const Matix4x4 &parent)
{
	_data.world[i.i] = _data.local[i.i] * parent;
	
    Instance child = _data.first_child[i.i];
    while (is_valid(child)) {
       transform(child, _data.world[i.i]);
       child = _data.next_sibling[child.i];
    }
}
```

注意：为了便于阅读，我将其写成递归函数，但你可能希望将其重写为迭代函数，以获得更好的性能。

请注意，当交换数组中的两个实例时（执行如上所述的脏列表交换或对数组进行排序），除了交换数组中的元素外，还需要注意保持所有 `parent`、`first_child`、`next_sibling` 和 `prev_sibling` 引用的有效性。这可能会有点麻烦，特别是当你更改引用并试图同时遍历这些引用列表时。当你想交换两个实例 \[A\] 和 \[B\] 时，我的建议是使用数组末尾的位置（即 \[size\]）作为临时存储槽，而不是试图一次做所有事情。

```C++
// Move element at A (and all references to it) to size.
[size] <--- [A]

// Now nothing refers to A, so we can safely move element at B 
// (and references to it) to A.
[A] <--- [B]

// And finally move the element at size to B.
[B] <-- [size]
```

在下一篇文章中，我将介绍如何将实体编译到资源文件。