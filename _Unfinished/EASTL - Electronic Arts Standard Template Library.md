*本文档译自 open-std.org 的 "[Electronic Arts Standard Template Library](https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2007/n2271.html#Motivation)" 一文，作者 Paul Pedriana*


## 概述 - *Abstract*
----
游戏平台和游戏设计的需求和其他平台的软件有所不同。重要的是，游戏软件需要大量内存，但现实往往有所限制。游戏软件还面临其他的限制，比如（与需求相比）不够强大的处理器缓存、不够快的处理器、不一样的内存对齐需求。这导致了游戏必须更谨慎地使用内存和处理器。

C++ 标准库的容器、迭代器和算法或许能满足各种游戏编程的潜在需求。然而，标准库的弱点和不足阻止了它成为高性能游戏的最优选择。最重要的弱点则是它的内存分配模型。

*EA* 进行了对 C++ 标准库的扩展和部分重新设计，产生了替代品（*EASTL*），以便以可移植和一致的方式解决这些弱点。本文描述了游戏软件开发问题，当前 C++ 标准的弱点，以及 *EASTL* 为解决这些弱点而使用的部分解决方案。


## 介绍 - *Introduction*
---
本文的目的是解释 *EASTL* 的设计和背后的动机，以便它可以帮助 C++ 社区更好地理解游戏软件开发的需求。本文档不是某种编程建议，尽管 *EASTL* 的一些更改和扩展可以构成此类讨论的基础。但 *EASTL* 的大部分内容对于任何类型的 C++ 软件开发都是有用的。

本文档描述了 *EA* 内部开发的 *STL* 实现（*EASTL*），作为 C++ 标准库定义的 *STL* 的替代和扩展。其中 *STL* 指 C++ 标准库的容器、迭代器和算法组件，以下称为 *std STL*（其中 *std* 指 *std* 命名空间，而 *STL* 中的 "S" 指标准 C++ ）。这里的 C++ 标准指 [ISO 14882 (1998)](https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2007/n2271.html#cpp_standard) 以及 2003 年的更新。*std STL* 的大部分设计都是优秀的，并且达到了预期的目的。然而，它的某些方面使其难以使用，而其他方面则使其无法发挥其应有的性能。对于游戏开发者来说，最根本的弱点是标准内存分配器的设计，而正是这个弱点导致了 *EASTL* 的诞生。其次是缺乏内存友好型的 *std STL* 容器设计。下面还将讨论一些其他原因。

我们希望那些阅读本文档的人对 *std STL* 可能不适用于所有用途的想法持开放态度。在本文撰写之前，关于 *EASTL* 的草图已经展示给了一些游戏开发行业之外的人。在某些情况下，我们发现有一种反应是拒绝一个备选的 *STL*，并假设某人一定是误解或滥用了 *STL*。但在解释了游戏开发和高性能软件的问题，并将其与当前供应商的标准 *STL* 设计和实现进行比较后，人们通常会减少他们的怀疑。事实上，我们发现那些拥有最广泛和最深刻的 *STL* 经验的人是那些对 *EASTL* 最热情的人。尽管如此，我们对 C++ 标准库及其在设计和实现方面所做的伟大工作，特别是在经历了漫长而艰难的实现过程之后，仍怀有极大的敬意。

这个文档分为以下几个章节。第一章总结了创造 *EASTL* 的动机和它的设计。剩下的几章如下：

-   [*EASTL* 的动机](https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2007/n2271.html#Motivation)
-   [标准 *STL* 和 *EASTL* 的区别](https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2007/n2271.html#Differences)
-   [*EASTL* 的功能摘要](https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2007/n2271.html#Functionality)
-   [游戏软件的问题](https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2007/n2271.html#game_software_issues)
-   [*std::allocator*](https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2007/n2271.html#std_allocator)
-   [*eastl::allocator*](http://www.amazon.com/Effective-STL-Specific-Standard-Template/dp/0201749629)
-   [*EASTL* 容器](https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2007/n2271.html#EASTL_containers)
-   [*EASTL* 的缺点](https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2007/n2271.html#EASTL_flaws)
-   [附录](https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2007/n2271.html#Appendix)
-   [致谢](https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2007/n2271.html#Acknowledgements)
-   [引用](https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2007/n2271.html#References)


## EASTL 的动机 - *Motivation for EASTL*
---







