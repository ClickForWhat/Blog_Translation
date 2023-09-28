*本文档译自 bitsquid.blogspot.com 的 "Building a Data-Oriented Entity System" 系列文章，作者 Niklas Frykholm*


## 概述 - *Overview*
----
在上一篇文章中，我讨论了 *TransformComponent* 的设计。今天，我们将看看如何将实体存储为资源。


## 动态数据和静态数据 - *Dynamic and Static Data*
---
我非常喜欢将资源编译成大块的二进制数据，这些二进制数据可以直接读入内存并按原样使用，而不需要反序列化或引用补丁。

这需要做到两件事：

+ 首先，数据在生成时交换到正确的字节序。
+ 其次，资源的内部引用必须使用偏移量的方式，而不是使用指针，因为我们不知道加载的资源在内存中的位置，所以我们希望避免指针重定向。

最初时，这个方案看起来有点复杂。但是这实际上比反序列化和引用重定向之类的东西简单得多。

不过要注意，这个方案仅用于静态数据，比如网格，纹理等等。如果一个资源的每个实例，其数据都可能更改，我们则必须把它存在其他地方。比如一个实例需要改变颜色，我们不可能把它存到和其他实例共享的内存区域里。

通常情况下，我们将动态数据组织成资源实例，并让该实例引用静态资源数据。许多资源实例可以使用相同的资源数据，从而节省内存。

```
---------------                -----------------
|Instance of A| --------+----> |A resource data|
---------------         |      -----------------
                        |
---------------         |
|Instance of A| --------+
---------------

---------------                -----------------
|Instance of B| --------+----> |B resource data|
---------------         |      -----------------
                        |
---------------         |
|Instance of B|---------+
---------------
```

我们通常硬编码实例的内容。例如，我们知道颜色是我们想要修改的东西，所以我们将它添加到实例数据中。顶点和索引缓冲区不能更改，因此进入静态数据。当我们创建资源实例时，我们用资源数据中的实例默认值初始化实例数据。

你可以用另一种方式来做。你可以假定每个资源实例都可以修改所有内容，而不是对每个实例可以修改的数据进行硬编码。在实例数据中，使用灵活的键值储存来存储实例数据和资源数据之间的增量。

这比硬编码更灵活，因为它允许你覆盖每个资源实例的所有内容，甚至是纹理或顶点数据。它还可以节省内存，因为实例数据将只包含你实际覆盖的内容，而不是可能被覆盖的所有内容。因此，如果有许多使用默认值的实例，则不必为它们存储任何数据。

另一方面，这种方法也有缺点。访问 *value* 变得更加复杂和昂贵，因为我们总是需要执行额外的查询来确定实例是否覆盖了默认值。

我们目前没有在引擎的任何地方使用这种方法。但我认为在某些情况下它是有意义的。

不管怎样，我有点跑题了。回到实体系统。

主要问题是，我希望实体系统的运行方式，是非常动态的。组件可以在运行时添加和删除，可以加入子实体，可以更改属性。处理静态数据的组件，比如 *MeshComponent*，是通过引用一个包含网格数据的单独的 *MeshResource* 来完成的。没有网格数据存储在组件本身。

由于实体系统中的所有内容都是动态的，那么这里只有实例数据。我们在资源数据中唯一拥有的是实例数据的模板。从本质上讲，这只是一组用于设置实例数据的“指令”。在遵循了这些指令之后，实例不需要再引用回资源。


## 定义资源格式 - *Defining the Resource Format*
---
这样一个实体资源只包含某种“指令”来设置实体。那么它究竟是什么样的？让我们从必需的东西开始：

```C++
struct EntityResource
{
    unsigned num_components;
    ComponentData components[num_components];
    unsigned num_children;
    EntityResource children[num_children];
};
```

注意：以上当然不是合法的C++代码。我这里使用某种类似C的伪代码，允许诸如动态大小的结构体之类的东西来描述数据布局。我以前写过关于需要一种语言来描述数据布局的[文章](http://bitsquid.blogspot.se/2012/11/a-formal-language-for-data-definitions.html)。

*ComponentData* 的确切二进制布局由每个组件类型定义，但让我们使用通用的包装格式：

```C++
struct ComponentData
{
    unsigned component_identifier;
    unsigned size;
    char data[size];
};
```

现在我们可以访问 *num_children* 参数，而不必遍历所有组件及其大小来知道我们需要在资源中向前跳转多远才能到达 *num_children* 字段。

这在实践中可能重要，也可能不重要。也许，在处理完所有组件数据之后，我们只需要 *num_children* 的值，此时我们已经有了一个指向正确位置的资源指针。但出于习惯，我总是把固定大小的数据放在第一位，以防我们可能需要它。

有时，为这些类型的资源添加偏移量表是有意义的，这样我们就可以快速查找特定组件或子组件的偏移量，而不必遍历所有内存并计算大小：

```C++
struct EntityResource
{
    unsigned num_components;
    unsigned num_children;
    unsigned offset_to_component_data[num_components];
    unsigned offset_to_child_data[num_children];
    ComponentData components[num_components];
    EntityResource children[num_children];
};
```

通过这种布局，我们可以获得第 *i* 个组件和第 *j* 个子组件的数据：

```C++
struct EntityResourceHeader
{
    unsigned num_components;
    unsigned num_children;
};

const EntityResourceHeader *resource;

const unsigned *offset_to_component_data = (const unsigned *)(resource + 1);

ComponentData *data_i = (const ComponentData *)
    ((const char *)resource + offset_to_component_data[i]);

const unsigned *offset_to_child_data = (const unsigned *)
    (offset_to_component_data + num_components);
    
EntityResourceHeader *child_j = (const EntityResourceHeader *)
    ((const char *)resource + offset_to_child_data[j]);
```

当你第一次遇到这样的代码时，这里面所有类型转换和指针的处理可能会让你感到困惑。但是，如果仔细思考发生了什么以及数据是如何在内存中布局的，那么它真的很容易理解。你所犯的任何错误都很可能导致很容易发现的巨大崩溃，而不是狡猾的微妙漏洞。过一段时间你就会习惯这种操作。

但是，不管怎样，我又偏离正题了，因为实际上，对于我们的目的，我们不需要这些查找表。我们将从头到尾遍历内存，每次创建一个组件。因为我们不需要在不同的组件之间跳转，所以我们不需要查找表。

我们需要的是某种存储多个资源的方法。如果我们处理的是包含单个实体的预制体类型的资源，那么只存储一个实体是很好的。但是，关卡呢？它可能包含一堆实体。所以最好有一个能存储所有实体的资源类型。

好吧，没什么大不了的，我们知道怎么做：

```C++
struct EntitiesResource
{
    unsigned num_entities;
    EntityResource entities[num_entities];
};
```

这样，就可以了吗？


## 枢轴！ - *Pivot！*
---
在这个行业工作一段时间后，你可能会对性能何时重要、何时不重要有一种直觉。当然，直觉不一定可靠，所以不要忘记测试、测试、再测试。但关卡加载/生成往往是性能至关重要的领域之一。

一个关卡很容易拥有1万个或更多的对象，有时候你想要快速生成它们，比如当玩家重新开始关卡时。所以我们有必要花点时间思考如何快速生成关卡。

从资源布局来看，我们的生成算法似乎非常直接：

+ 创建第一个实体
  + 添加它的第一个组件
  + 添加第二个组件
  + ......
  + 创建它的子实体
+ 创建第二个实体
  + 添加第一个组件
  + 添加第二个组件
  + ......
+ ......

这是如此简单和直接，似乎没有什么改进的余地。而且我们在遍历组件时线性遍历资源内存数据，所以我们是缓存友好的，对吗？

嗯，不完全是。我们违反了面向数据设计的基本原则之一：一起做类似的事情。如果我们线性地写出实际执行的操作，而不是像上面那样用层次结构描述，就更容易看出来：

-   创建实体 *A*
-   创建 *A* 的 `TransformComponent`
-   创建 *A* 的 `MeshComponent`
-   创建 *A* 的 `ActorComponent`
-   创建实体 *B*
-   创建 *B* 的 `TransformComponent`
-   创建 *B* 的 `MeshComponent`
-   …

好多了。我们还可以更进一步。

与其告诉 *EntityManager* 创建一个实体100次，不如告诉它创建100个实体。这样，如果创建多个实体有任何批量效益，*EntityManager* 就可以利用它。我们对组件做同样的处理：

+ 创建实体 （*A*，*B*）
+ 创建（*A*，*B*）的  `TransformComponent`
+ 创建（*A*，*B*）的  `MeshComponent`
+ 创建 *A* 的  `ActorComponent`

注意我们在这里是如何遇到和使用面向数据的原则和指导方针的：

+ 线性访问内存。
+ 如果有一个，可能就会有更多。
+ 将相似的对象和操作组合在一起。
+ 一次对多个对象执行操作，而不是一次对一个对象执行操作。

让我们重新编写数据格式，以展示这新的想法：

```C++
struct EntityResource
{
    unsigned num_entities;
    unsigned num_component_types;
    ComponentTypeData component_types[num_component_types];
};

struct ComponentTypeData
{
    unsigned component_identifier;
    unsigned num_instances;
    unsigned size;
    unsigned entity_index[num_instances];
    char instance_data[size];
};
```

对于每个组件，我们存储一个标识符 `component_identifier`，以便我们知道它是 *MeshComponent* 还是 *TransformComponent* 等。然后存储此类型组件的实例数量以及这些实例的数据大小。

注意，现在当我们遍历格式时，我们可以通过一次跳转跳过未知组件类型的所有实例，而不是一个接一个地忽略它们。这虽然不太重要，但有趣的是，面向数据的重组通常会使许多不同类型的操作更高效，而不仅仅是最初的目标操作。

`entity_index` 用于将组件与实体关联起来。假设我们创建了五个实体：*A*、*B*、*C*、*D* 和 *E*，以及两个 `ActorComponent`。我们需要知道每个 `ActorComponent` 应该属于哪个实体。我们通过简单地将实体的索引存储在 `entity_index` 中来实现。因此，如果 `entity_index` 数组为 { 2, 3 }，则这两个组件分属于 *C* 和 *D*。

在新的布局中有一件事我们没有处理：子实体。

但是子实体在概念上与其他实体并无不同。我们可以将它们添加到 `num_entities` 中，并把它们的组件实例也添加到 `ComponentTypeData` 中，就像我们对任何其他实体所做的那样。

我们唯一需要关心的是如何存储父子关系。我们可以将其存储为 *TransformComponent* 数据的一部分，或者我们可以仅存储一个数组，该数组每个槽表示一个实体，槽内的值表示当前实体的父实体索引（如果是根实体，索引就是 `UINT_MAX`）：

```C++
struct EntityResource
{
    unsigned num_entities;
    unsigned num_component_types;
    unsigned parent_index[num_entities];
    ComponentTypeData component_types[num_component_types];
};
```

如果在上面 *A*, *B*, *C*, *D*, *E* 的例子中，`parent_index` 数组是 { `UINT_MAX`, 0, 1, 1, 2 }，那么实体间的关系将是：

```
A --- B --- C --- E
      |
      + --- D
```


## 实现的细节 - *Implementation Details*
---
这篇文章已经太长了，所以我只是简单地说一下如何组织实现这个目标。

在引擎中，我们有一个用于编译实体到资源文件的类 *EntityCompiler* 和一个从资源文件生成实体的类 *EntitySpawner*。

可以编译成资源数据的组件需要在 *EntityCompiler* 中注册自己，这样当 *EntityCompiler* 遇到这种类型的组件数据时就可以调用它。

暂时忽略一些细节，如错误处理、端序转换和跟踪依赖之类的，它看起来会是这样：

```C++
typedef Buffer (*CompileFunction)(const JsonData &config, NittyGritty &ng);

void register_component_compiler(const char *name, CompileFunction f,
    int spawn_order);
```

`CompileFunction` 接受一些描述组件的 *JSON* 配置数据，并返回资源数据二进制 *Blob*。请注意，`CompileFunction` 一次只对一个组件进行操作，因为我们不太关心编译实体时的性能。

在注册时，我们指定一个名称，比如 “*mesh_component*”。如果在 *JSON* 数据中找到该名称，则 *EntityCompiler* 将把组件数据的编译重定向到该函数。该名称也将被散列到组件的 `component_identifier` 中。

`spawn_order` 用于指定不同组件的编译顺序，并通过扩展，指定它们的生成顺序。原因是，一些组件可能会使用其他组件。例如，*MeshComponent* 想知道实体在哪里，所以它在实体中寻找 *TransformComponent*。因此，*TransformComponent* 必须在 *MeshComponent* 之前创建。

注册组件生成器和注册组件编译器的方法类似：

```C++
typedef void (*SpawnFunction)(const Entity *entity_lookup,
    unsigned num_instances, const unsigned *entity_index, const char *data);

void register_component_spawner(const char *name, SpawnFunction f);
```

在这里，`entity_lookup` 允许我们在资源数据中查找实体的索引，以便在资源生成的第一步中创建它。`num_instances` 是应该创建的组件实例的数量，`entity_index` 是来自 *ComponentTypeData* 的实体索引，它让我们可以查找哪个实体应该拥有这个组件。

因此，`entity_lookup[entity_index[i]]` 给出了应该拥有第 *i* 个组件实例的实体。

最后，`data` 是一个指针，指向来自 *ComponentTypeData* 的实例数据。

今天就到这里吧。下一次，我们将看一个具体的例子。