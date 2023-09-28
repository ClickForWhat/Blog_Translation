*本文档译自 The Machinery Engine Dev Blog 的 "Multi-Threading The Truth" 一文，作者 Niklas Gray*


## 概述 - *Overview*
----
在过去的几周里，我一直在做引擎的线程安全相关的工作。我发现网上关于如何在实际的系统中执行多线程的资料相对较少（如果需求是比单个全局锁更快的话）。我找到的大多数文章都侧重于简单的数据类型，比如链表。所以我想分享一下我的经历。

*The Truth* 是我们对于存储应用程序数据的集中式系统的半开玩笑的名称。它基于 *ID*、对象、类型和属性。系统中的对象以 `uint64_t` 为 *ID* 进行标识。每个对象都有一组基于对象类型的属性（布尔，整型，浮点，字符串等等）。它的API看起来像这样：

```C++
uint64_t asset_id = the_truth->create_object_of_type(tt, asset_type);
the_truth->set_string(tt, asset_id, ASSET_NAME_PROPERTY, "foo");
```

*The Truth* 是其他系统读取和写入数据的地方。当然，它们也可以使用自己的内部缓冲区（双缓冲），以获得更好的性能，但如果它们想与外部世界共享数据，则需要通过 *The Truth*。

除了基本的数据存储，*The Truth* 还有其他特性，比如子对象（其他对象拥有的对象）、引用（对象之间）、原型（作为其他对象的模板/预制件）和更新通知。


## 设计目标 - *Goals of the design*
---
因为 *The Truth* 是一个低层级系统，我们希望它能够处理大量的数据，所以高性能的设计是非常重要的。把整个 *The Truth* 都锁在一把锁上当然是不行的。我们需要有许多线程同时访问 *The Truth*。与此同时，我们也不想从 *The Truth* 获取每一条数据时都加锁，因为这也可能代价高昂。

我们希望的是这样的东西：

+ 读操作不需要任何等待/锁。
+ 写操作可能需要锁，但是它们只对实际接触到的数据加锁。线程应该能够同时写入不同的对象而不会产生争用。

这里有一个问题很明显：如果 `get()` 没有使用任何锁，我们如何确保我们正在读取的数据不会被同时进行的 `set()` 操作破坏？

幸运的是，原子性的概念拯救了我们。在现代硬件上，对齐的64位值的写入和读取是原子的。也就是说，其他线程将总是要么看到旧的，要么看到新的，而不是看到一半新的，一半旧的。因此，只要我们只写64位的值，`get()` 和 `set()` 就可以在同一个对象上同时运行，而不会产生问题。

不幸的是，我们需要存储更大的对象，例如字符串或集合。不过，我们可以用指针处理这些问题。我们可以在对象中写入一个新的 `char*`，而不是一段一段地写入字符串，否则可能会导致值混乱。这是一个64位的值，可以以原子方式写入。

事实上，我们可以更进一步。我们可以创建一个对象的全新副本，更改一些值，然后在查找表中自动将旧副本替换为新副本，而不是只写入单个值。

![[lookup-table.png]]
<center>查找表</center>

我们可以将这种复制和替换隐式地完全隐藏在 *get/set* 函数接口后面，但这有两个缺点：

+ 读入者可能看到处于半写状态的对象。也就是说，即使每个单独的属性要么完全写入，要么不写入，属性本身也可以来自对象的不同版本。例如，读者可以看到 *y = 5* 和 *z = 2*，一个从未存在过的对象。
+ 如果写入者想要写入多个属性（通常是这种情况），我们将不得不在后台复制对象多次，这是低效的。

因此，我们让接口更加显式地完成这一操作。我们需要一个 `read()` 或 `write()` 操作来获取指向对象的指针，并需要一个 `commit()` 操作来完成写操作：

```C++
const object_o *reader = the_truth->read(tt, id);
float x = the_truth->get_float(tt, reader, POSITION_X_PROPERTY);
float y = the_truth->get_float(tt, reader, POSITION_Y_PROPERTY);
float z = the_truth->get_float(tt, reader, POSITION_Z_PROPERTY);

object_o *writer = the_truth->write(tt, id);
the_truth->set_float(tt, writer, POSITION_Y_PROPERTY, 5.0f);
the_truth->set_float(tt, writer, POSITION_Z_PROPERTY, 5.0f);
the_truth->commit(tt, writer);
```

请注意，通过此更改，`reader` 的指针可能指向新对象，也可能指向旧对象，因此 `reader` 将看到 (7, 3, 2) 或 (7, 5, 5)，但不会看到 (7, 5, 2)。

还要注意，我们在这里所做的与版本控制的工作方式非常相似——我们正在创建对象的新版本。这也是为什么我选择 `commit()` 这个名字。

最后要注意的是，我们不能在 `commit()` 时顺便删除旧对象，因为可能仍然有 `reader` 在使用那个旧对象（稍后会详细介绍）。


## 关于一致性的说明 - *Some notes on consistency*
---
通过这种设计，我们可以避免读入写了一半的值和对象。但是我们仍然会有其他的一致性问题。

最简单的一个是写冲突。如果同时对一个对象进行了两个写入怎么办？它们都将获得旧对象的副本，更改一些值，然后提交。最后提交的人将覆盖其他写入者的值。

当多个对象相互依赖时，可能会出现更复杂的一致性问题。例如，写入者可能会这样做：

1. 移除对象 *A* 对对象 *B* 的引用。
2. 删除对象 *B*。

读入者可能在步骤1发生之前读取对象 *A* 并获取对 *B* 的引用，然后在步骤2发生之后尝试访问B，这将失败，因为 *B* 现在已被删除。

在数据库世界中，类似的一个经典例子是在账户之间转移资金。若把从一个账户取钱并把钱存入另一个账户分别视为单独的操作，那么读入者可以看到钱凭空出现。

有趣的是，我们可以在不强制让读入者加锁的情况下解决这两个问题。

为了解决写冲突，我们可以使用所有现代CPU上都可以使用的 [*compare-and-swap*](http://web.archive.org/web/20220316014137/https://en.wikipedia.org/wiki/Compare-and-swap)（*CAS*）指令，也可以通过 C11 和 C++ 11 中的原子库使用。只有当64位值与我们的期望值相匹配时，*CAS* 才会写入64位值。否则，它将保留旧值。它还会告诉我们写入是否成功。

我们可以修改一下 `commit()` 的逻辑，使其不只是写入新指针，而是使用我们最初用于获取对象的写拷贝的指针执行 *CAS*。这意味着 `commit()` 只会在对象从我们获得它以来没有改变的情况下才会成功。在上面的示例中，第二个写入者的 `commit()` 将失败，对象将保留第一个写入者写入的值。

为了解决多个对象的一致性问题，例如货币转账，我们需要某种交易机制。换句话说，我们需要能够同时将更改提交给多个对象，并让读者一次看到所有这些更改。起初，这可能看起来很棘手，因为我们一次只能写一个原子值。但我们可以重复以前的技巧，当我们需要写一个更大的值时，我们只需写一个指向该值的指针。

在这种情况下，我们可以替换的指针是指向整个查找表的指针。通过替换查找表，我们可以随心所欲地替换其中任意多的单个对象。

![[hierarchical-lookup-table.png]]
<center>替换查找表</center>

读者将得到一个指向旧查找表或新查找表的指针。因此，读者将看到 (A, B, C, D, E) 或 (A, B, C, D, E)。

我们也可以将它与 *CAS* 技术结合起来，只在根指针没有改变的情况下替换它。这允许我们将任意数量的对象作为单个事务进行更改，该事务要么成功要么失败。与关系数据库中的事务模型非常相似。

另一件有趣的事情是：如果你了解 *git* 如何在内部处理树对象和 *blob*，你可以看到这种方法实际上与 *git* 非常相似。
