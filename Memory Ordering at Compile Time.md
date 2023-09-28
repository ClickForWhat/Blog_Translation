*本文档译自 preshing.com 中关于 "Lock-Free Programming" 文章，作者 Jeff Preshing*


## 概述 - *Overview*
----
在你写下 *C/C++* 源代码时和它在 *CPU* 上执行时，这些代码的内存交互行为可能会根据某些规则重新排序。对内存顺序的更改是由编译器（在编译时）和处理器（在运行时）进行的，所有这些都是为了能让代码运行得更快。

编译器开发人员和 *CPU* 供应商普遍遵循的内存重排序的基本规则可以表述如下：

	不应该修改单线程程序的行为。

根据这一原则，内存序重排这一现象在程序员编写单线程代码时几乎是被忽视的。当然，即使是在多线程中，它也常常被忽视，因为像互斥锁、信号量和事件之类的东西，就是被用来在调用它们的地方阻止内存序重排的。它几乎只在无锁技术中被加以重视——当内存在线程之间共享，而且没有任何互斥保护的时候——这时小鸡子终于露出了黑脚，内存序重排的影响可以[清楚地观察到](http://preshing.com/20120515/memory-reordering-caught-in-the-act)。

请注意，可以为多核平台编写无锁代码，而无需考虑内存重新排序的麻烦。正如我在[无锁编程的介绍](http://preshing.com/20120612/an-introduction-to-lock-free-programming)中提到的，可以利用顺序一致的类型，例如 *Java* 的 `volatile` 变量或 *C++* 11 的原子类型，只不过可能会以牺牲一点性能为代价。这里我就不详细讲了。在这篇文章中，我将重点关注编译器对常规、非顺序一致类型的内存排序的影响。


## 编译指令重排序 - *Compiler Instruction Reordering*
---
众所周知，编译器的工作是将人类可读的源代码转换为 *CPU* 可读的机器代码。在这种转换过程中，编译器可以自由地采取许多措施。

其中之一的自由就是把指令重新排序——请再次注意，这只在不改变单线程程序的行为的情况下。这种指令重排序通常只在启用编译器优化时发生。考虑下面的函数：

```C++
int A, B;

void foo()
{
    A = B + 1;
    B = 0;
}
```

如果我们使用 *GCC* 4.6.1 编译这个函数而不进行编译器优化，它将生成以下机器码，我们可以使用 `-S` 选项查看汇编代码列表。可以看到，对全局变量 `B` 的内存存储发生在对 `A` 的存储之后，和源代码中的一样。

```
$ gcc -S -masm=intel foo.c
$ cat foo.s
        ...
        mov     eax, DWORD PTR _B  (redo this at home...)
        add     eax, 1
        mov     DWORD PTR _A, eax
----->  mov     DWORD PTR _B, 0
        ...
```

将其与使用 `-O2` 启用优化时产生的汇编代码进行比较：

```
$ gcc -O2 -S -masm=intel foo.c
$ cat foo.s
        ...
        mov     eax, DWORD PTR B
----->  mov     DWORD PTR B, 0
        add     eax, 1
        mov     DWORD PTR A, eax
        ...
```

这一次，编译器选择行使它的自由，在存储到 `A` 之前先存储到 `B`，为什么不这样做呢？内存排序的基本规则没有被打破。单线程程序永远不会知道其中的区别。

另一方面，这种编译器重排序在编写无锁代码时可能会导致问题。下面是一个经常被引用的示例，其中使用共享标志来标识一些共享数据已经可用。

```C++
int Value;
int IsPublished = 0;
 
void sendValue(int x)
{
    Value = x;
    IsPublished = 1;
}
```

想象一下，如果编译器把对 `IsPublished` 的赋值排到 `Value = x` 之前会发生什么事。即使在单处理器系统上，我们也会遇到一个问题：一个线程很可能会被两个存储之间的操作系统抢占，让其他线程相信 `Value` 已经更新，而事实上它还没有更新。

当然，编译器可能不会对这些操作进行重新排序，生成的机器代码能以无锁操作正常工作，例如在任何具有[强内存模型](http://preshing.com/20120930/weak-vs-strong-memory-models)的多核 *CPU*（如 *x86/64*）上，或者在单处理器环境中。如果碰上了这样的事情，我们是幸运的。当然，最好还是能识别共享数据的内存交互行为重新排序的可能性，并执行正确的排序。


## 显式编译器屏障 - *Explicit Compiler Barriers*
---
防止编译器重新排序的最简单方法是使用一个称为编译器屏障的特殊指令。我已经在之前的文章中演示了编译器屏障。以下是 *GCC* 中的完整编译器屏障。在 *Microsoft Visual C++* 中，[`_ReadWriteBarrier`](http://msdn.microsoft.com/en-us/library/f20w0x5e.aspx) 有同样的效果。

```C++
int A, B;

void foo()
{
    A = B + 1;
    asm volatile("" ::: "memory");
    B = 0;
}
```

有了这个改变，我们可以保持优化的启用状态，内存指令将保持我们希望的顺序。

```
$ gcc -O2 -S -masm=intel foo.c
$ cat foo.s
        ...
        mov     eax, DWORD PTR _B
        add     eax, 1
        mov     DWORD PTR _A, eax
----->  mov     DWORD PTR _B, 0
        ...
```

类似地，如果我们希望保证 `sendMessage` 示例正常工作，并且只关心单处理器系统，那么我们必须在这里引入编译器屏障。不仅发送操作需要一个编译器屏障，以防止存储区重新排序，而且接收端在加载之间也需要一个编译器屏障。

```C++
#define COMPILER_BARRIER() asm volatile("" ::: "memory")

int Value;
int IsPublished = 0;

void sendValue(int x)
{
    Value = x;
    COMPILER_BARRIER();          // prevent reordering of stores
    IsPublished = 1;
}

int tryRecvValue()
{
    if (IsPublished)
    {
        COMPILER_BARRIER();      // prevent reordering of loads
        return Value;
    }
    return -1;  // or some other value to mean not yet received
}
```

正如我提到的，编译器屏障足以防止单处理器系统上的内存重新排序。但现在是 2012 年，如今，多核计算已经成为常态。如果我们想确保我们的内存交互在多处理器环境和任何 *CPU* 架构中按照我们希望的顺序发生，那么只有编译器屏障是不够的。我们需要发出 *CPU Fence* 指令，或者在运行时执行任何充当内存屏障的操作。我将在下一篇文章 [Memory Barriers Are Like Source Control Operations](http://preshing.com/20120710/memory-barriers-are-like-source-control-operations) 中介绍更多关于这些的内容。

*Linux* 内核通过预处理器宏（如 `smb_rmb`）公开了几个 *CPU Fence*指令，在为单处理器系统编译时，这些宏被[简化为简单的编译器屏障](http://lxr.free-electrons.com/source/arch/powerpc/include/asm/barrier.h#L40)。


## 隐式编译器屏障 - *Implied Compiler Barriers*
---
还有其他方法可以防止编译器重新排序。实际上，我刚才提到的 *CPU Fence* 指令也起到了编译器屏障的作用。下面是 *PowerPC* 的一个示例 *CPU Fence* 指令，它在 *GCC* 中被定义为宏：

```C++
#define RELEASE_FENCE() asm volatile("lwsync" ::: "memory")
```

在代码中放置 `RELEASE_FENCE` 的任何地方，除了编译器重排序之外，它还将防止某些类型的处理器重排序。它可以用来使我们的 `sendValue` 函数在多处理器环境中是安全的。

```C++
void sendValue(int x)
{
    Value = x;
    RELEASE_FENCE();
    IsPublished = 1;
}
```

在新的 *C++ 11*（以前称为 *C++ 0x*）原子库标准中，每个非 *relaxed* 的原子操作也能充当编译器屏障。

```C++
int Value;
std::atomic<int> IsPublished(0);

void sendValue(int x)
{
    Value = x;
    // <-- reordering is prevented here!
    IsPublished.store(1, std::memory_order_release);
}
```

正如你所期望的那样，每个包含编译器屏障的函数本身也都必须充当编译器屏障，即使函数是内联的。（然而，微软的文档表明，在早期版本的 *Visual C++* 编译器中可能不是这样。啧啧！）

```C++
void doSomeStuff(Foo* foo)
{
    foo->bar = 5;
    sendValue(123);       // prevents reordering of neighboring assignments
    foo->bar2 = foo->bar;
}
```

事实上，大多数函数调用都充当编译器屏障，无论它们内部是否包含自己的编译器屏障（这不包括内联函数、使用 `pure` 属性声明的函数以及使用链接时代码生成的情况）。除了这些情况，对外部函数的调用甚至比编译器障碍更强，因为编译器不知道函数的副作用是什么。它必须忘记它对该函数可能可见的内存所做的任何假设。

仔细想想，这很有道理。在上面的代码片段中，假设我们的 `sendValue` 实现存在于一个外部库中。编译器如何知道 `sendValue` 是否依赖于 `foo->bar` 的值？它怎么知道 `sendValue` 不会修改内存中的 `foo->bar`？因此，为了遵守内存排序的基本规则，它不能围绕对 `sendValue` 的外部调用重新排序任何内存操作。类似地，它必须在调用完成后从内存中加载 `foo->bar` 的新值，而不是假设它仍然等于 5，即使启用了优化。

```
$ gcc -O2 -S -masm=intel dosomestuff.c
$ cat dosomestuff.s
        ...
        mov    ebx, DWORD PTR [esp+32]
----->  mov    DWORD PTR [ebx], 5            // Store 5 to foo->bar
        mov    DWORD PTR [esp], 123
        call    sendValue                    // Call sendValue
----->  mov    eax, DWORD PTR [ebx]          // Load fresh value from foo->bar
        mov    DWORD PTR [ebx+4], eax
        ...
```

正如你所看到的，在许多情况下，编译器指令重排序是被禁止的，甚至当编译器必须从内存中重新加载某些值时也是如此。我相信这些隐藏的规则是人们长期以来一直认为在正确编写的多线程代码中通常不需要 *C* 中的 `volatile` 数据类型的很大一部分[原因](http://www.kernel.org/doc/Documentation/volatile-considered-harmful.txt)。


## 无中生有的存储 - *Out-Of-Thin-Air Stores*
---
觉得指令重排序使无锁编程变得棘手？更恐怖的是，在 *C++* 11 标准化之前，没有技术上的规则阻止编译器使用更糟糕的技巧。特别是，编译器可以在之前明明没有存储指令的地方自己加上存储指令。下面是一个非常简化的示例，灵感来自 [*Hans Boehm*](http://www.hpl.hp.com/techreports/2004/HPL-2004-209.pdf) 在[多篇文章](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2007/n2338.html)中提供的示例。

```C++
int A, B;

void foo()
{
    if (A)
        B++;
}
```

虽然在实践中不太可能，但并没有规定不能编译器在检查 `A` 之前将 `B` 存到寄存器，从而产生相当于以下的机器码：

```C++
void foo()
{
    register int r = B;    // Promote B to a register before checking A.
    if (A)
        r++;
    B = r;                 // Surprise! A new memory store where there previously was none.
}
```

内存排序的基本规则仍然遵循。单线程应用程序也不会知道。但在多线程环境中，我们现在有了一个函数，它会清除其他线程中同时对 `B` 所做更改（总是重新赋值 `B`）——即使 `A` 为0。最初的代码并没有这样做。尽管几十年来我们一直在用 *C/C++* 编写多线程和无锁代码，但这种模糊的、技术上不可能的原因导致人们一直在说 *C++* 对线程支持不好。

我不知道有谁在实践中成为了这种无中生有的存储的受害者。也许这只是因为对于我们倾向于编写的无锁代码类型，没有很多适合这种模式的优化机会。我想，如果我发现这种类型的编译器转换正在发生，我会寻找一种方法使编译器屈服。如果这发生在你身上，请在评论中告诉我。

无论如何，新的 *C++* 11 标准明确禁止编译器在可能引入数据竞争的情况下执行此类行为。这些措辞可以在最新的 [*C++* 11 工作草案 §1.10.22](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3337.pdf) 中找到：

	编译器转换引入了抽象机无法修改的对共享内存位置的潜在赋值，这种转换通常被本标准排除。


## 为什么编译器要重排序？ - *Why Compiler Reordering?*
---
正如我在开始时提到的，编译器修改内存交互顺序的原因与处理器进行性能优化的原因相同。这种优化是现代 *CPU* 复杂性的直接结果。

可能这话有些冒险，但不知何故，我怀疑编译器在 80 年代初就做了大量的指令重排，当时 *CPU* 最多只有几十万个晶体管。我认为这没有多大意义。但从那时起，摩尔定律为 *CPU* 设计者提供了大约 10000 倍数量的晶体管，这些晶体管被用于流水线、存储器预取、*ILP* 以及最近的多核等技巧。由于其中的一些特性，我们已经看到了某些体系结构中，程序指令的顺序可以在性能上产生显著差异。

1993 年发布的第一款英特尔奔腾处理器，带有所谓的 *U* 和 *V* 管道，是第一款我真正记得人们谈论管道和指令排序重要性的处理器。然而，最近，当我在 *Visual Studio* 中进行 *x86* 反汇编时，我真的很惊讶指令的重新排序如此之少。另一方面，在我在 *Playstation* 3 上进行 *SPU* 反汇编过程中，我发现编译器真的很受欢迎。这些只是一些轶事经历；它可能不能反映其他人的经验，当然也不应该影响我们在无锁代码中强制执行内存排序的方式。