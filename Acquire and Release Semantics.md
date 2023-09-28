*本文档译自 preshing.com 中关于 "Lock-Free Programming" 主题的文章，作者 Jeff Preshing*


## 概述 - *Overview*
----
一般来说，在[无锁编程](http://preshing.com/20120612/an-introduction-to-lock-free-programming)中，线程可以通过两种方式操作共享内存：它们可以相互竞争资源，或者相互配合将信息从一个线程传递到另一个线程。获取（*acquire*）和释放（*release*）语义对后者至关重要：在线程之间可靠地传递信息。事实上，我冒昧地猜测，不正确地使用或缺少获取和释放语义是无锁编程中引发错误的主要原因。

在这篇文章中，我将演示在 *C++* 中实现获取和释放语义的各种方法。我会大致介绍 *C++* 11 原子库标准，因此你不需要了解它。请注意，这里不会讨论[顺序一致性](http://preshing.com/20120612/an-introduction-to-lock-free-programming#sequential-consistency)的无锁编程。我们将在多核或多处理器环境中直接处理内存序。

不幸的是，“获取/释放语义”这一术语似乎比术语“无锁”更糟糕，因为你在网上搜索得越多，你会发现越多看似矛盾的定义。*Bruce Dawson* 在[白皮书](http://msdn.microsoft.com/en-us/library/windows/desktop/ee418650.aspx)的中间部分给出了几个很好的定义（归功于 *Herb Sutter*）。我想提供几个我自己的定义，这些定义与 *C++* 11 *Atomic* 背后的原理密切相关：

首先，获取语义（*Acquire semantics*） 可视作一个属性，只应用于从共享内存中进行**读取**的操作，无论这些操作是[读取-修改-写入（*RMW*）](http://preshing.com/20120612/an-introduction-to-lock-free-programming#atomic-rmw)操作还是普通的加载操作。我们将该操作视为读获取（*read-acquire*）操作。获取语义可以阻止对读获取操作**之后**（按程序顺序）的任何读或写操作进行内存重排序。

![[read_acquire.png]]

释放语义（*Release semantics*）可视作一个属性，只应用于**写入**共享内存的操作，无论这些操作是读取-修改-写入操作还是普通的存储操作。我们将该操作视为写释放（*write-release*）操作。释放语义可以阻止对写释放操作**之前**（按程序顺序）的任何读或写操作进行内存重排序。

![[write_release.png]]

一旦你理解了上面的定义，就不难看出获取和释放语义可以使用我在[上一篇文章](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations)中详细描述的内存屏障类型的简单组合来实现。屏障必须（以某种方式）放在读获取操作之后，或在写释放之前。\[ ***更新*** ：请注意，这些屏障其实在技术上比单个内存操作的获取和释放语义要求更严格，不过它们确实达到了预期的效果。\]

![[acq_rel_barriers.png]]

很酷的是，获取和释放语义都不需要使用 `#StoreLoad` 屏障，这通常是一种更昂贵的内存屏障类型。例如，在 *PowerPC* 上，`lwsync`（*lightweitght sync* 的缩写）指令同时充当三个`#LoadLoad`、`#LoadStore` 和 `#StoreStore` 屏障，但比 `sync` 指令便宜，后者包含一个 `#StoreLoad` 屏障。


## 显式的平台特定Fence指令 - With Explicit Platform-Specific Fence Instructions
---
获得内存屏障的一种方法是发出显式的 *fence* 指令。让我们从一个简单的例子开始。假设我们正在为 *PowerPC* 编写代码，`lwsync()` 是一个编译器内部函数，它发出 `lwsync` 指令。由于 `lwsync` 提供了如此多的屏障类型，我们可以在下面的代码中使用它来根据需要建立获取或释放语义。在线程 1 中，对 `Ready` 的存储变为写释放，而在线程 2 中，对 `Ready` 的加载变为读获取。

![[platform_fences.png]]

如果我们让所有线程执行，并且看到了 `r1 == 1`，那我们就能确保在线程 1 中分配的 `A` 的值成功传递给了线程 2。因此，我们可以保证 `r2 == 42`。在我[之前的文章](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations)中，我已经给 `#LoadLoad` 和 `#StoreStore` 做了一个冗长的类比来说明这是如何工作的，所以我不会在这里重复解释。

在正式的术语中，我们说 `Ready` 的存储同步于读取。我在这里写了一篇关于同步的[文章](http://preshing.com/20130823/the-synchronizes-with-relation)。要使这种技术在一般情况下起作用，获取和释放语义必须应用于相同的变量——在本例中，是 `Ready`，同时它的加载和存储操作都必须是原子的。这里，`Ready` 是一个简单对齐的 `int`，因此在 *PowerPC* 上操作已经是原子的。


## 在可移植C++ 11中使用围栏 - With Fences in Portable C++11
---
上面的示例是特定于编译器和处理器的。支持多平台的一种方法是将代码转换为 *C++* 11。所有 *C++* 11 的标识符都存在于 `std` 命名空间中，因此为了使下面的例子简洁，让我们假设语句`using namespace std` 已经被放在代码的某个地方。

*C++* 11 标准的原子库定义了一个可移植的 `atomic_thread_fence()` 函数，该函数接受一个参数来指定 *fence* 的类型。这个参数有几个可能的值，但我们在这里最感兴趣的值是 `memory_order_acquire` 和 `memory_order_release`。我们将使用这个函数来代替 `lwsync()`。

在完成这个示例之前，还需要进行一个更改。在 *PowerPC* 上，我们知道 `Ready` 上的两个操作都是原子性的，但我们不能对每个平台都做出这样的假设。为了确保这些操作在所有平台上都是原子的，我们需要把 `Ready` 的类型从 `int` 改成 `atomic<int>`。我知道，这修改有点傻，因为 `int` 的对齐加载和存储在当今存在的每个现代 *CPU* 上都已经是原子的了。我将在关于 *synchronized-with* 的帖子中详细介绍这一点，但现在，让我们采取在理论上100%正确的行为。`A`不需要进行任何更改。

![[cpp11_fences.png]]

上面的参数 `memory_order_relaxed` 的意思是“确保这些操作是原子操作，但不要强加任何未指明的排序约束/内存屏障”。

上述两个 `atomic_thread_fence()` 调用都可以（希望是）在 *PowerPC* 上作为 `lwsync` 的实现。同样，它们都可以在 *ARM* 上发出一条 `dmb` 指令，我认为这至少与 *PowerPC* 的 `lwsync` 一样有效。在 *x86/64* 上，两个 `atomic_thread_fence()` 调用可以简单地用编译器屏障实现，因为通常情况下，*x86/64* 上的每个加载都意味着获取语义，而每个存储都意味着释放语义。这也是 *x86/x64* 经常被称为[强序](http://preshing.com/20120930/weak-vs-strong-memory-models)的原因。


## 在可移植C++ 11中不使用围栏 - *Without Fences in Portable C++11*
---
在 *C++* 11 中，可以直接在 `Ready` 上实现获取和释放语义，而无需发出显式的 *fence* 指令。你只需要在 `Ready` 上的操作上直接指定内存排序约束：

![[cpp11_no_fences.png]]

可以把它看作是将每个 *fence* 指令包含到 `Ready` 本身的操作中。\[ ***更新*** ：请注意，此形式与单独使用 *fence* 的版本[不完全相同](http://preshing.com/20131125/acquire-and-release-fences-dont-work-the-way-youd-expect) ；从技术上讲，它没有那么严格。\] 编译器将发出获得所需屏障效果所需的指令。特别是，在 *Itanium* 上，每个操作都可以很容易地实现为单个指令： `ld.acq` 和 `st.rel`。和前面一样，`r1 == 1` 表示关系已同步，这对 `r2 == 42` 做出了保证。

这实际上是 *C++* 11 中表达获取和释放语义的首选方式。实际上，前面示例中使用的原子线程 `fence()` 函数是在标准创建的相对较晚的时候添加的。


## 使用锁时的获取/释放语义 - *Acquire and Release While Locking*
---
正如你所看到的，这篇文章中的例子都没有利用由获取和释放语义提供的 `#LoadStore` 屏障。实际上，只有 `#LoadLoad` 和 `#StoreStore` 部分在这篇文章中是必需的。这只是因为我选择了一个简单的例子，可以让我们专注于 *API* 和语法。

当使用获取和释放语义来实现（互斥）锁时，`#LoadStore` 部分变得至关重要。实际上，这就是名称的由来：获取锁意味着获取语义，而释放锁意味着释放语义。两者之间的所有内存操作都包含在一个漂亮的”屏障三明治“中，防止任何不希望的内存跨边界重新排序。

![[acq_rel_lock.png]]

在这里，获取和释放语义确保在持有锁时所做的所有修改将对获取锁的下一个线程完全可见。每个锁的实现，甚至是你自己创建的锁，都应该提供这些保证。同样，这都是关于在线程之间可靠地传递信息，特别是在多核或多处理器环境中。

在后续的文章中，我将展示 *C++* 11 代码的工作演示，在真实的硬件上运行，如果不使用获取和释放语义，可以清楚地观察到它会中断。