*本文档译自 blog.molecular-matters.com 的 "Stateless, layered, multi-threaded rendering" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
继续上节的内容，今天我想介绍一些关于如何设计无状态渲染API的想法。

在讨论如何设计无状态渲染API之前，让我们快速看一下有状态渲染API通常是如何执行其任务的。


## 常规的有状态渲染 - *Conventional, stateful rendering*
----
这是每个人都知道的渲染方式：你在这里和那里设置一些状态，提交一个绘制调用，再设置一些状态，再提交一个绘制调用，等等。

通常，它看起来像下面这样：

```C++
// 1) render first object
backend::SetCullState(CULLSTATE_BACK);
backend::SetVertexBuffer(vb);
backend::SetIndexBuffer(ib);
backend::BindTexture(0u, diffuse);
backend::DrawIndexed(triCount*3, 0u, 0u);
 
// 2) render second object
backend::SetCullState(CULLSTATE_FRONT);
backend::BindTexture(0u, otherDiffuse);
backend::SetAlphaBlendState(ONE_ONE);
backend::DrawIndexed(triCount*3, 0u, 0u);
```

这种抽象的问题在于，在渲染第一个对象时设置的任何状态都会影响第二个对象的渲染，而第二个对象又会影响第三个对象的渲染，依此类推。一旦在管线中设置了状态，就会泄漏到后续的绘制调用中。如果还不是很明显的话，上面的状态抽象实际上有两个问题（不仅仅是一个！）：

+ 第一个问题：当渲染第三个对象时，如果我们忘记将其设置回 `CULLSTATE_BACK`，它将被渲染为正面剔除。*Alpha* 混合也是一样。这是两个问题中较小的一个。
+ 第二个问题：每当你不得不更改某个绘制调用的状态时，之后所有不涉及此状态的绘制调用都将导致出错。这比第一个问题要糟糕得多，因为你要么必须更改不涉及的所有状态，要么总是在提交绘制调用后将所有涉及的状态设置回其默认值。这既容易出错又乏味。

我们甚至还没有开始讨论多线程渲染。

为了详细说明第二点，想象一下如果我们将上面的代码更改为下面的代码会发生什么：

```C++
// 1) render first object
backend::SetCullState(CULLSTATE_BACK);
backend::SetVertexBuffer(vb);
backend::SetIndexBuffer(ib);
backend::SetRasterizerState(NO_DEPTH_WRITE); // <=====
backend::BindTexture(0u, diffuse);
backend::DrawIndexed(triCount*3, 0u, 0u);
 
// 2) render second object
backend::SetCullState(CULLSTATE_FRONT);
backend::BindTexture(0u, otherDiffuse);
backend::SetAlphaBlendState(ONE_ONE);
backend::DrawIndexed(triCount*3, 0u, 0u);
```

通过引入一个改变管线状态的新命令 `SetRasterizerState()`，之后的所有绘制调用也会受到我们的更改的影响，因为其他绘制调用不会更改光栅状态，也就不会重新调用 `SetRasterizerState()`。我们要么必须在第二个绘制调用中显式设置它，要么在提交第一个绘制调用后重置它。当你想把某些渲染操作从这里移动到那里，将它们插在某些函数中间的时候，情况会更糟，因为你必须始终注意“上下文状态”。就像我说的，容易出错而且乏味。


## 引入无状态API - *Introducing a stateless API*
----
在了解了上面有状态方法的明显错误之后，让我们尝试提出更好的解决方案。一个可能的解决方案是每帧都从一个干净的默认状态开始，并在提交绘制调用时将所有状态重置回默认状态。如果用户要自己执行该操作，则可能如下所示：

```C++
// at the beginning of a frame, all states are set to their default value
 
// 1) render first object
backend::SetCullState(CULLSTATE_BACK);
backend::SetVertexBuffer(vb);
backend::SetIndexBuffer(ib);
backend::BindTexture(0u, diffuse);
backend::DrawIndexed(triCount*3, 0u, 0u);
backend::ResetDefault(); // <===
 
// 2) render second object
backend::SetCullState(CULLSTATE_FRONT);
backend::BindTexture(0u, otherDiffuse);
backend::SetAlphaBlendState(ONE_ONE);
backend::DrawIndexed(triCount*3, 0u, 0u);
backend::ResetDefault(); // <===
```

当然，我们也可以把这个功能放到我们的API中，让它来处理。现在，让我们假设我们有一个大的渲染队列，它用于在一帧期间存储所有的绘制调用，然后在一帧结束时使用渲染后端进行排序和分配。然后我们可以这样做：

```C++
// 1) render first object
renderQueue::SetCullState(CULLSTATE_BACK);
renderQueue::SetVertexBuffer(vb);
renderQueue::SetIndexBuffer(ib);
renderQueue::BindTexture(0u, diffuse);
renderQueue::SubmitIndexed(triCount*3, 0u, 0u);
 
// 2) render second object
renderQueue::SetCullState(CULLSTATE_FRONT);
renderQueue::BindTexture(0u, otherDiffuse);
renderQueue::SetAlphaBlendState(ONE_ONE);
renderQueue::SubmitIndexed(triCount*3, 0u, 0u);
 
// at the end of a frame:
renderQueue::Sort();
renderQueue::Flush();
```

基本上，我们所有的 *renderQueue* 实现都必须做以下工作：

+ 跟踪当前设置的顶点缓冲区、索引缓冲区、剔除状态、*Alpha* 状态、纹理采样器等。每当有人调用 `renderQueue::Set*State()` 时，只需将相应的成员更改为新状态。
+ 对于每个 `Submit*()` 调用，在队列中插入一个新的 *draw call*。在这种情况下，我们的队列是原始内存，我们将简单地把操作的类型（一个 `DrawIndexed()` 调用）、键（用于排序）以及与绘制调用相关的所有数据（在这个例子下，即所有当前状态）都存下来。之后，我们将所有内部状态重置为默认值。
+ 在调用 `Sort()` 时，我们简单地使用例如基数排序对所有键进行排序。
+ 在调用 `Flush()` 时，我们遍历已排序的操作数组，获取类型，获取数据，并调用各自的渲染后端函数。这与实现一个简单的虚拟机非常相似。

当然，还有许多实现细节我们还没有讨论，但这基本上是它的要点。然而，这种方法有一点我相当不喜欢，当多线程渲染要掺一脚的时候。

使用多线程渲染时，我们希望能够从任何线程调用任何 `renderQueue::*` 函数，这意味着即使C++代码看起来像顺序代码，但对各种 `renderQueue::Set*()` 函数的调用其实是来自不同的线程，因此它们真正的顺序是交错的。

我们不能再在 `renderQueue` 实现中使用简单的成员来跟踪当前状态，用互斥锁（或类似的东西）来包装每个函数也不行，因为我们需要一次包装单个绘制调用的所有操作，这有太多的开销，别考虑这么做。

当然，有一个更简单、更快的解决方案：[线程局部存储](http://en.wikipedia.org/wiki/Thread-local_storage)。每个线程不是使用 `renderQueue` 中的简单成员来跟踪当前设置的状态，而是使用一个保存所有状态的线程局部结构体来跟踪它的状态。

然而，我仍然不满意这样的方法，因为这意味着每个 `renderQueue` 函数调用现在都必须访问一些线程局部变量，这与直接访问内存相比增加了开销。因此，我也在考虑以下几种选择。


## 备选方案一 - *Alternative 1*
----
第一种方法可以归结为在堆栈上创建记录状态的结构体，然后在提交绘制调用时将所有结构体复制到队列中，如下所示：

```C++
IndexedDrawCall dc;
dc.SetVertexBuffer(mesh->vertexBuffer);
dc.SetIndexBuffer(mesh->indexBuffer);
dc.SetCullState(CULLSTATE_BACK);
renderQueue::Submit(dc);
```

首先，这基本上消除了我们在上面的方法中看到的所有多线程问题。如果我们想使用一个全局队列，我们所要做的就是将调用中给出的数据复制到 `renderQueue::Submit()` （以及用于排序的键）。为此，我们可以简单地使用一个线性分配器，每次分配后递增一个指针。通过使用原子操作，我们可以简单地使分配既线程安全又快速。如果不想使用原子操作，可以改用每个线程使用线程局部队列。

其次，这将允许我们缓存某些绘制调用。对于某些静态部分，我们可以只构建一次绘制调用，把它存储在某个地方，之后提交到 `renderQueue` 中时，将不需要任何额外的工作。

第三，每个绘制调用，如 `IndexedDrawCall`, `InstancedDrawCall`, `ComputeDrawCall` 等，可以确保只存储它需要的数据，这可以减少存储单个绘制调用所需的内存数量。

不过，对于这种方法，有两点我不喜欢：

1. 绘制调用结构体的每个实例都是有状态的，这意味着用户可以在堆栈上实例化一个绘制调用结构体，提交一次，更改其状态，然后再次提交。当然，这种行为取决于用户。不推荐这么做，因为这样我们又回到了原点，可以这么说。
2. 我们访问内存的频率比实际需要的要高得多，因为我们首先更改堆栈上结构体的状态，然后将其所有数据复制到内存中的其他位置，这取决于 `renderQueue::Submit()` 将数据复制到的位置。

这就引出了我最后一个也是目前最喜欢的选择。


## 备选方案 2 - *Alternative 2*
----
你不需要在堆栈上创建绘制调用结构体，而是直接让 `renderQueue` 把一个实例交给你：

```C++
IndexedDrawCall* dc = renderQueue::CreateIndexedDrawCall();
dc->SetVertexBuffer(mesh->vertexBuffer);
dc->SetIndexBuffer(mesh->indexBuffer);
dc->SetCullState(CULLSTATE_BACK);
renderQueue::Submit(dc);
```

看起来差别不大，但我们可以在这里做一些事情：

1. 当创建一个新的绘制调用时（例如使用 `CreateIndexedDrawCall()`），我们同样可以选择使用全局队列和原子操作来分配内存，或者使用线程局部队列。我更喜欢后者（在下一篇文章中会详细介绍），但重点是创建这样一个绘制调用本质上只是在内部递增指针，将所有绘制数据的最终目的地交给用户。这意味着我们不再操作堆栈上的结构体，然后复制它，而是直接写入内存。对`Submit()` 的调用只需存储键和指向结构体存储位置的指针。
2. 因为我们可以控制如何创建绘制调用，所以我们可以很容易地确保用户不能两次提交绘制调用。我们可以通过检查 `renderQueue::Submit()` 的参数指针来做到这一点：如果它的地址小于或等于上一次提交的绘制的地址，则用户试图提交相同的绘制调用两次，这是无效的，因为这意味着使用了一个有状态的绘制调用结构。


## 结语 - *Conclusion*
----
可以看出，对于如何实现无状态API，已经有一些替代方案。我认为在设计这样的API时，重要的是要注意多线程渲染以及如何处理绘制调用数据的内存分配。

请注意，我只是简单地谈到了多线程渲染的主题。还有很多事情需要考虑，比如错误共享、如何进行分配以及何时以及如何将数据写入内存。在设计这样一个API时，我会考虑这些事情，但还没有时间写下我所有的想法和想法——这篇文章已经很长了。

进一步的，我们还没有讨论如何生成用于排序数据的键，以及我们如何尝试按层“分组”绘制调用，这在第一篇文章中有提到过。这将是下一篇文章的主题！

#### 免责声明

我还没有真正实施以上任何一项方案，所以请对此持谨慎态度。这肯定不是最终的设计，因为这些东西通常需要几次迭代，直到想出真正满意的东西。

请在评论中指出我犯的任何疏忽或错误，并随时讨论我可能错过的其他更好的选择！