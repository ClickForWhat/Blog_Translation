*本文档译自 learn.microsoft.com 中关于 "Lockless Programming" 的文章，作者 Bruce Dawson*


## 概述 - *Overview*
---
无锁编程是一种在多个线程之间安全地共享、更改数据的方法，无需获取和释放锁。这听起来像是灵丹妙药，但无锁编程是复杂而微妙的，有时并不能带来它所承诺的好处。*Xbox 360* 上的无锁编程尤其复杂。

无锁编程是多线程编程的有效技术，但不应轻易使用。 在使用之前，必须了解其复杂性，并且应仔细测量，以确保它确实能带来预期的收益。在许多情况下，应该使用更简单和更快的解决方案，例如更少地在线程间共享数据。

正确且安全地使用无锁编程需要对硬件和编译器有大量了解。本文概述了尝试使用无锁编程技术时要考虑的一些问题。


## 有锁编程 - *Programming with Locks*
---
编写多线程代码时，通常需要在线程之间共享数据。 如果多个线程同时读取和写入共享数据，则可能发生内存损坏。 解决此问题的最简单方法是使用锁。 例如，如果希望一次只能由一个线程执行 *ManipulateSharedData*，则可以使用 *CRITICAL_SECTION*（临界区） 来保证这一点，如以下代码所示：

```C++
// Initialize
CRITICAL_SECTION cs;
InitializeCriticalSection(&cs);

// Use
void ManipulateSharedData()
{
    EnterCriticalSection(&cs);
    // Manipulate stuff...
    LeaveCriticalSection(&cs);
}

// Destroy
DeleteCriticalSection(&cs);
```

此代码相当简单明了，很容易判断它是正确的。 但是，使用锁编程有几个潜在的缺点。 例如，如果两个线程都尝试获取相同的两个锁，但以不同的顺序获取它们，则可能会出现死锁。 如果某个程序持有锁的时间过长（由于设计不佳或线程已被优先级较高的线程交换掉），则其他线程可能会被阻塞很长时间。 此风险在 *Xbox 360* 上尤其大，因为软件线程是由开发者分配给一个硬件线程的，操作系统不会将它们转移到另一个硬件线程上，即使这个线程是空闲的。*Xbox 360* 也没有针对优先级反转的保护，即高优先级线程在等待低优先级线程释放锁的同时在循环中自旋。 最后，如果延迟的过程调用或中断服务例程尝试获取锁，则可能会出现死锁。


## 无锁编程 - *Lockless Programming*
---
顾名思义，无锁编程是一系列技术，用于在不使用锁的情况下安全地操作共享数据。 无锁算法可用于传递消息、共享列表和数据队列以及其他任务。

执行无锁编程时，必须应对两个挑战：非原子操作和指令重排序。


## 非原子操作 - *Non-Atomic Operations*
---
原子操作是不可分割的，即保证其他线程永远不会在操作完成一半时看到该操作。 原子操作对于无锁编程非常重要，因为如果没有它们，其他线程可能会看到半写入的值，或者其他不一致的状态。

在所有新式处理器上，可以假定自然对齐的本机类型的读取和写入是原子的。 只要内存总线至少与正在读取或写入的类型一样宽，*CPU* 可以在单个总线事务中读取和写入这些类型，从而使其他线程无法看到它们处于半完成状态。 在 *x86* 和 *x64* 上，不能保证大于 8 个字节的读取和写入是原子的。这意味着流式 *SIMD* 扩展（*SSE*）寄存器的16字节读写和序列操作等可能不是原子的。

没有自然对齐的类型读取和写入（例如，写入跨越四字节边界的 `DWORD`）不保证是原子的。 *CPU* 可能需要以多个总线事务的形式执行这些读取和写入操作，这可能导致另一个线程修改或查看正在读取或写入一半的数据。

复合操作（例如递增共享变量时发生的读取-修改-写入等一系列操作）不是原子操作。 在 *Xbox 360* 上，这些操作以多个指令 (*lwz*、*addi* 和 *stw*) 实现，线程可能在操作序列进行到一半时被换出。 在 x86 和 x64 上，有一个指令 (*inc*) ，可用于递增内存中的变量。如果使用此指令，则递增变量的行为在单处理器系统上是原子的，但在多处理器系统上仍不是原子的。 在基于 *x86* 和 *x64* 的多处理器系统上使 *inc* 原子需要使用 *lock* 前缀，这会阻止另一个处理器在 *inc* 指令的读取和写入之间执行自己的读取-修改-写入操作序列。

以下代码显示了部分示例：

```C++
// This write is not atomic because it is not natively aligned.
DWORD* pData = (DWORD*)(pChar + 1);
*pData = 0;

// This is not atomic because it is three separate operations.
++g_globalCounter;

// This write is atomic.
g_alignedGlobal = 0;

// This read is atomic.
DWORD local = g_alignedGlobal;
```


## 保证原子性 - *Guaranteeing Atomicity*
---
可以通过组合以下各项来确保使用原子操作：

- 自然的原子操作
- 用于包装复合操作的锁
- 实现了常用复合操作的操作系统函数（的原子版本）

正如上所述，递增变量不是原子操作，如果在多个线程上执行递增可能会导致数据损坏。

```C++
// This will be atomic.
g_globalCounter = 0;

// This is not atomic and gives undefined behavior
// if executed on multiple threads
++g_globalCounter;
```

*Win32* 附带一系列函数，这些函数提供多个常见操作的原子读取-修改-写入版本。 这些是 *InterlockedXxx* 系列函数。 如果共享变量的所有修改都使用这些函数，则修改将是线程安全的。

```C++
// Incrementing our variable in a safe lockless way.
InterlockedIncrement(&g_globalCounter);
```


## 重排序 - *Reordering*
---
一个更微妙的问题是重排序。读取和写入操作并不总是按照你在代码中编写的顺序发生，这可能会导致非常混乱的问题。在许多多线程算法中，线程先写入一些数据，再写入一个标志，该标志告诉其他线程数据已准备就绪。这称为 ***write-release***。如果对写入进行重排序，其他线程可能会先看到标志被设置，然后才能看到写入的数据。

同样，在许多情况下，线程先读取标志，然后读取一些共享数据（如果该标志指出线程已获取对共享数据的访问权限）。这称为 ***read-acquire***。 如果读取被重排序，则数据可能在读取标志之前从共享存储中读取，并且看到的值可能不是最新的。

读取和写入的重新排序可以由编译器（通常在编译时）和处理器（通常在执行时）完成。 编译器和处理器已经进行这种重新排序很多年了，这在单处理器计算机上，问题不那么大。这是因为读取和写入的 *CPU* 重新排列在单处理器计算机上是不可见的， (非设备驱动程序代码不属于设备驱动程序) ，并且编译器对读写的重排在单处理器机器上不太可能引起问题。

如果编译器或 *CPU* 重新排列以下代码中显示的写入操作，另一个线程可能会看到活动标志已设置，同时仍会看到 *x* 或 *y* 的旧值。读取时可能会发生类似的重新排列。

在此代码中，一个线程将新条目添加到 *sprite* 数组：

```C++
// Create a new sprite by writing its position into an empty
// entry and then setting the ‘alive' flag. If ‘alive' is
// written before x or y then errors may occur.
g_sprites[nextSprite].x = x;
g_sprites[nextSprite].y = y;
g_sprites[nextSprite].alive = true;
```

在下一个代码块中，另一个线程读取 *sprite* 数组：

```C++
// Draw all sprites. If the reads of x and y are moved ahead of
// the read of ‘alive' then errors may occur.
for(int i = 0; i < numSprites; ++i)
{
    if(g_sprites[nextSprite].alive)
    {
        DrawSprite(g_sprites[nextSprite].x,
                g_sprites[nextSprite].y);
    }
}
```

为了使这个 *sprite* 系统正常运作，我们需要防止编译器和 *CPU* 对读写进行重新排序。


## 了解 CPU 对写入的重新排列 - *Understanding CPU Rearrangement of Writes*
---
一些 *CPU* 会重新安排写操作，这使得指令的执行以非原程序的顺序在外部被其他处理器或设备可见。尽管这种重新排列对单线程非驱动程序代码永远不可见，但它可能在多线程代码中导致问题。

### Xbox 360 - *Xbox 360*

虽然 *Xbox 360 CPU* 不会重新排序指令，但它会重新排列写操作，这些操作在指令本身之后完成。这种写操作的重新安排是 *PowerPC* 内存模型特别允许的。

*Xbox 360* 上的写入不会直接进入 *L2* 缓存。相反，为了提高 *L2* 缓存写带宽，它们先通过存储队列，然后再到存储-收集（*store-gather*）缓冲区。存储-收集缓冲区允许在一次操作中将 64 字节块写入 *L2* 缓存。有 8 个存储-收集缓冲区，允许对多个不同的内存区域进行高效写入。

存储-收集缓冲区通常按先入先出（*FIFO*）的顺序写入 *L2* 缓存。但是，如果写入的目标缓存行不在 *L2* 缓存中，就需要从内存提取目标缓存行，这可能导致该写入被延迟。

即使存储-收集缓冲区本身以严格的 *FIFO* 顺序写入 *L2* 缓存，这并不能保证单个写入按顺序写入 *L2* 缓存。 例如，假设 *CPU* 写入位置 0x1000，然后写入位置 0x2000，最后写入位置 0x1004。第一次写入分配给一个存储-收集缓冲区，并将其放在队列的前面。 第二个写入分配另一个存储-收集缓冲区，并将其放在队列中的下一个。第三次写入将其数据添加到第一个存储-收集缓冲区，该缓冲区保留在队列的前面。因此，第三次写入最终在第二次写入之前进入 *L2* 缓存。

存储-收集缓冲区导致的重新排序根本无法预测，特别是核心上的两个线程共享存储-收集缓冲区时，从而使存储-收集缓冲区的分配和清空变得高度可变。

这是如何重新排序写入的一个示例。 可能还有其他可能性。

### x86 和 x64 - *x86 and x64*

即使 *x86* 和 *x64 CPU* 对指令重新排序，但它们通常不会相对于其他写入对写入操作重新排序。 写合并（*Write-Combined*）内存存在一些例外情况。 此外，一连串的操作（*MOVS* 和 *STOS*）和 16 字节 *SSE* 写入允许在内部重新排序，但除此之外，写入之间不会彼此重新排序。


## 了解 CPU 对读取的重新排列 - *Understanding CPU Rearrangement of Reads*
---
一些 *CPU* 会重新安排读取顺序，以便它们以有效地非程序顺序读自共享存储。这种重新排列对单线程非驱动程序代码永远不可见，但在多线程代码中可能会导致问题。

### Xbox 360 - *Xbox 360*

缓存未命中可能会导致某些读取延迟，这实际上会导致读取共享内存时的乱序，并且这些缓存未命中所消耗的时间从根本上是不可预知的。 预取和分支预测也会导致数据来自共享内存时的无序。 这里只列出导致读取重新排序的几个可能，还可能存在其他可能性。 *PowerPC* 内存模型专门允许这种读取重新排列。

### x86 和 x64 - *x86 and x64*

尽管 *x86* 和 *x64 CPU* 会对指令进行重排序，但它们通常不会相对于其他读取对读取操作重新排序。不过，一连串的操作（*MOVS* 和 *STOS*）和 16 字节 *SSE* 读取允许在内部重新排序，但除此之外，读取之间不会彼此重新排序。


## 其他的重排序 - *Other Reordering*
---
即使 *x86* 和 *x64 CPU* 不会相对于其他写入对写入重排，或相对于其他读取对读取重排，但没有规定不会对写入和读取之间进行重排。 具体而言，如果程序写入一个位置，然后从其他位置读取，则可能实际上先从共享内存中读取数据，然后再写入数据。 这种重新排序可能会破坏某些算法，例如 *Dekker* 的互斥算法。 在 *Dekker* 算法中，每个线程都设置一个标志来指示它想要进入临界区，然后检查另一个线程的标志，以查看另一个线程是否已在临界区或试图进入临界区。 初始代码如下所示。

```C++
volatile bool f0 = false;
volatile bool f1 = false;

void P0Acquire()
{
    // Indicate intention to enter critical region
    f0 = true;
    // Check for other thread in or entering critical region
    while (f1)
    {
        // Handle contention.
    }
    // critical region
    ...
}


void P1Acquire()
{
    // Indicate intention to enter critical region
    f1 = true;
    // Check for other thread in or entering critical region
    while (f0)
    {
        // Handle contention.
    }
    // critical region
    ...
}
```

这里存在的问题是 `P0Acquire` 中 `f1` 的读取可以在对 `f0` 写入到共享存储之前进行。 同时在 `P1Acquire` 中，可以在对 `f1` 写入到共享存储之前，对 `f0` 进行读取。 最终结果是两个线程将其标志设置为 `true`，并且两个线程都将另一个线程的标志视为 `false`，因此它们都进入了临界区。 因此，虽然在基于 *x86* 和 *x64* 的系统上重新排序的问题不如 *Xbox 360* 上常见，但它们仍可能发生。 在上述任何平台上，在没有硬件内存屏障的情况下，*Dekker* 的算法无法正常工作。

*x86* 和 *x64 CPU* 不会把写入重新排到先前的读操作之前。*x86* 和 *x64 CPU* 只会在目标位置不同的情况下，把读取重新排到先前的写操作之前。

而 *PowerPC CPU* 可以在写之前对读重新排序，也可以在读之前对写重新排序，只要它们的地址不同。


## 重排序的关键摘要 - *Reordering Summary*
---
与 *x86* 和 *x64 CPU* 相比，*Xbox 360 CPU* 会对内存操作进行更激进的重排，如下表所示。 有关更多详细信息，请参阅处理器文档。

|重新排序行为|x86 和 x64|Xbox 360|
|---|---|---|
|读取移动到读取之前|不允许|允许|
|写入移动到写入之前|不允许|允许|
|写入移动到读取之前|不允许|允许|
|读取移动到写入之前|允许|允许|


## Read-Acquire和Write-Release屏障 - *Read-Acquire and Write-Release Barriers*
---
*read-acquire* 和 *write-release* 屏障是用于防止读和写重排序的主要结构。*read-acquire* 是对一个标志或其他变量的读取，以获得资源的所有权，并附带一个防止重新排序的屏障。类似地，*write-release* 是对标志或其他变量的写操作，以确认放弃资源的所有权，并附带一个防止重新排序的屏障。

*Herb Sutter* 的正式定义是：

- *read-acquire* 执行于某些读取或写入之前，而这些跟在 *read-acquire* 后的读取或写入（同一线程的）将按程序的顺序执行。
- *write-release* 执行于某些读取或写入之后，而这些处于 *write-release* 前的读取或写入（同一线程的）将按程序的顺序执行。

当代码获取某些内存的所有权时，无论是通过获取锁还是通过从共享链表中提取项（没有锁），始终需要读取——测试标志或指针以查看是否已获取内存的所有权。 此读取可能是 *InterlockedXxx* 操作的一部分，在这种情况下，它虽然涉及读和写，但读表明是否获得了所有权。在获得内存的所有权之后，通常从该内存中读取或写入值。再次地，在获得所有权之后才执行这些读取和写入非常重要。*read-acquire* 屏障保证了这一点。

当某些内存的所有权被释放时，无论是通过释放锁还是通过将项压入共享链表，总是会涉及到一个写操作，通知其他线程该内存现在可供它们使用。虽然你的代码拥有内存的所有权，但它可能从内存中读取或写入内存，因此在释放所有权之前就执行这些读取和写入非常重要。*write-release* 屏障保证了这一点。

你可以将 *read-acquire* 或 *write-release* 屏障直接视为一体的操作，这是最简单直接的。 但是，它们有时必须由两部分组成：读/写本身和一个不允许读/写（移动位置而导致）越过它的屏障。在这种情况下，屏障放置的位置至关重要。对于 *read-acquire* 屏障，首先读取标志，然后放置屏障，然后是对共享数据的读写。对于 *write-release* 屏障，首先是读写共享数据，然后是屏障，再然后写入标志。

```C++
// Read that acquires the data.
if(g_flag)
{
    // Guarantee that the read of the flag executes before
    // all reads and writes that follow in program order.
    BarrierOfSomeSort();
	
    // Now we can read and write the shared data.
    int localVariable = sharedData.y;
    sharedData.x = 0;
	
    // Guarantee that the write to the flag executes after all
    // reads and writes that precede it in program order.
    BarrierOfSomeSort();
    
    // Write that releases the data.
    g_flag = false;
}
```

*read-acquire* 和 *write-release* 之间的唯一区别是内存屏障的位置。*read-acquire* 在锁操作之后具有屏障，而 *write-release* 在锁操作之前具有屏障。在这两种情况下，屏障均位于对锁定内存的引用和对锁的引用之间。

若要了解为什么在获取和发布数据时都需要屏障，最好的（和最准确的）视角是将这些屏障视为与共享内存同步的保证，而不是与其他处理器同步的保证。如果一个处理器使用 *write-release* 将数据结构释放到共享内存，而另一个处理器使用 *read-acquire* 从共享内存访问该数据结构，则代码将正常工作。如果任一处理器未使用适当的屏障，则数据共享可能会失败。

使用正确的屏障来防止平台的编译器和 *CPU* 重新排序至关重要。

使用操作系统提供的同步原语的优点之一是，所有这些原语已经包含适当的内存屏障。


## 阻止编译器重新排序 - *Preventing Compiler Reordering*
---
编译器的工作是积极地优化代码以提高性能。这包括在有帮助的地方和不会改变行为的地方重新排列指令。由于 *C++* 标准从未提及多线程处理，并且编译器不知道哪些代码需要线程安全，因此在决定可以安全地进行哪些重新排列时，编译器会先假定你的代码是单线程的。 因此，当不允许对读取和写入进行重新排序时，需要告知编译器。

使用 *Visual C++*，可以使用编译器内部 [*_ReadWriteBarrier*](https://msdn.microsoft.com/library/f20w0x5e(v=VS.71).aspx)来阻止编译器重新排序。 在代码中插入 *\_ReadWriteBarrier* 时，编译器不会在代码中移动读取和写入操作。

```C++
#if _MSC_VER < 1400
    // With VC++ 2003 you need to declare _ReadWriteBarrier
    extern "C" void _ReadWriteBarrier();
#else
    // With VC++ 2005 you can get the declaration from intrin.h
#include <intrin.h>
#endif
// Tell the compiler that this is an intrinsic, not a function.
#pragma intrinsic(_ReadWriteBarrier)

// Create a new sprite by filling in a previously empty entry.
g_sprites[nextSprite].x = x;
g_sprites[nextSprite].y = y;
// Write-release, barrier followed by write.
// Guarantee that the compiler leaves the write to the flag
// after all reads and writes that precede it in program order.
_ReadWriteBarrier();
g_sprites[nextSprite].alive = true;
```

在以下代码中，另一个线程从 *sprites* 数组中读取：

```C++
// Draw all sprites.
for(int i = 0; i < numSprites; ++i)
{
    // Read-acquire, read followed by barrier.
    if(g_sprites[nextSprite].alive)
    {
    
        // Guarantee that the compiler leaves the read of the flag
        // before all reads and writes that follow in program order.
        _ReadWriteBarrier();
        DrawSprite(g_sprites[nextSprite].x,
                g_sprites[nextSprite].y);
    }
}
```

请务必了解，*\_ReadWriteBarrier* 不会插入任何其他指令，并且它不会阻止 *CPU* 重新排列读取和写入，它只会阻止编译器重新排列它们。因此，在 *x86* 和 *x64* 上实施 *write-release* 屏障时， *\_ReadWriteBarrier* 就足够了（因为 *x86* 和 *x64* 不会对写入重新排序，正常写入足以释放锁），但在大多数其他情况下，我们仍需要防止 *CPU* 重新排序读取和写入。

在写入不可缓存的写合并内存时，你也可以使用 *\_ReadWriteBarrier*，以防止写入重新排序。在这种情况下，*\_ReadWriteBarrier* 保证写入按处理器首选的线性顺序进行，从而有助于提高性能。

还可以使用 [*\_ReadBarrier*](https://msdn.microsoft.com/library/z055s48f(v=VS.80).aspx) 和 [*\_WriteBarrier*](https://msdn.microsoft.com/library/65tt87y8(v=VS.80).aspx) 内部函数更精确地控制编译器重新排序。编译器不会跨 *\_ReadBarrier* 重排读取，也不会跨 *\_WriteBarrier* 重排写入。


## 阻止 CPU 重新排序 - *Preventing CPU Reordering*
---
*CPU* 重新排序比编译器重新排序更微妙。你永远无法直接看到它发生，你只能看到莫名其妙的 *bug*。为了防止 *CPU* 对读取和写入进行重新排序，需要在某些处理器上使用内存屏障指令。在 *Xbox 360* 和 *Windows* 上，内存屏障指令的通用名称是 [*MemoryBarrier*](https://learn.microsoft.com/zh-cn/windows/win32/api/winnt/nf-winnt-memorybarrier)。 此宏是针对每个平台适当实现的。

在 *Xbox 360* 上，*MemoryBarrier* 被定义为 *lwsync*（意思是轻量级同步），也可通过 *ppcintrinsics.h* 中定义的 *\_\_lwsync* 内部函数获取。*\_\_lwsync* 还可充当编译器的内存屏障，防止编译器重新排列读取和写入。

*lwsync* 指令是 *Xbox 360* 上的内存屏障，可将一个处理器核心与 *L2* 缓存同步。它保证 *lwsync* 之前的所有写入操作都会在后续写入之前将其写入 *L2* 缓存。它还保证任何在 *lwsync* 之后的读取不会从 *L2* 读取比先前读取更旧的数据。它无法阻止的一种重新排序是读操作先于写操作移动到不同的地址。因此，事实上 *lwsync* 强制实施和 *x86* 和 *x64* 处理器上默认内存排序一样的内存排序。若要获取完整的内存排序，需要更昂贵的同步指令（也称为重量级同步），但在大多数情况下，这不是必需的。下表显示了 *Xbox 360* 上的内存重新排序选项。

|Xbox 360 的重新排序行为|不执行同步|lwsync|sync|
|---|---|---|---|
|读取移动到读取之前|允许|不允许|不允许|
|写入移动到写入之前|允许|不允许|不允许|
|写入移动到读取之前|允许|不允许|不允许|
|读取移动到写入之前|允许|允许|不允许|

*PowerPC* 还具有同步指令 *isync* 和 *eieio*（用于控制对 *caching-inhibited* 内存的重新排序）。正常同步时不需要这些同步指令。

在 *Windows* 上，[*MemoryBarrier*](https://learn.microsoft.com/zh-cn/windows/win32/api/winnt/nf-winnt-memorybarrier) 在 *Winnt.h* 中定义，并提供不同的内存屏障指令，具体取决于你是编译 *x86* 还是 *x64*。内存屏障指令能充当一个完整的屏障，防止所有读和写跨屏障重新排序。因此，*Windows* 上的 *MemoryBarrier* 提供了比在 *Xbox 360* 上更强大的重排序保证。

在 *Xbox 360* 和许多其他 *CPU* 上，还有一种方法可以阻止 *CPU* 的读取重新排序。 如果读取指针，然后使用该指针加载其他数据，==the CPU guarantees that the reads off of the pointer are not older than the read of the pointer==。==If your lock flag is a pointer and if all reads of shared data are off of the pointer==，则可以省略 *MemoryBarrier*，以适度节省性能。

```C++
Data* localPointer = g_sharedPointer;
if(localPointer)
{
    // No import barrier is needed--all reads off of localPointer
    // are guaranteed to not be reordered past the read of
    // localPointer.
    int localVariable = localPointer->y;
    // A memory barrier is needed to stop the read of g_global
    // from being speculatively moved ahead of the read of
    // g_sharedPointer.
    int localVariable2 = g_global;
}
```

*MemoryBarrier* 指令仅阻止对可缓存内存的读取和写入重新排序。 如果将内存分配为 *PAGE_NOCACHE* 或 *PAGE_WRITECOMBINE*（这是 *Xbox 360* 上设备驱动程序作者和游戏开发人员的常用技术）， 则 *MemoryBarrier* 对访问此内存没有影响。大多数开发人员不需要同步不可缓存的内存。 这超出了本文的范围。


## 互锁函数和 CPU 重新排序 - *Interlocked Functions and CPU Reordering*
---
有时，获取或释放资源的读取或写入操作可以使用 *InterlockedXxx* 函数的其中一个完成。在 *Windows* 上，这件事情大大简化了：因为在 *Windows* 上，*InterlockedXxx* 函数都是完整的内存屏障。它们前后实际上都有一个 *CPU* 内存屏障，这意味着它们本身都是完整的 *read-acquire* 或 *write-release* 屏障。

在 *Xbox 360* 上，*InterlockedXxx* 函数不包含 CPU 内存屏障。它们会阻止编译器对读取和写入进行重新排序，但不会阻止 *CPU* 重新排序。因此，在大多数情况下，在 *Xbox 360* 上使用 *InterlockedXxx* 函数时，应在它们前面或后面加上 一个 *\_\_lwsync*，使它们成为 *read-acquire* 或 *write-release* 屏障。为方便起见和更易于阅读，许多 *InterlockedXxx* 函数都有 *Acquire* 和 *Release* 版本。它们已经附带了内置的内存屏障。例如，*InterlockedIncrementAcquire* 在 *\_\_lwsync* 内存屏障之后执行一个互锁增量，以提供完整的 *read-acquire* 功能。

建议使用 *InterlockedXxx* 函数的 *Acquire* 和 *Release* 版本（大多数函数在 *Windows* 上也可用，且不会造成性能损失），以使意图更加明显，并更轻松地在正确位置获取内存屏障指令。 在 *Xbox 360* 上使用任何没有内存屏障的 *InterlockedXxx* 时，应非常仔细地检查，因为它通常是一个错误。

此示例演示一个线程如何使用 *InterlockedXxxSList* 函数的 *Acquire* 和 *Release* 版本将任务或其他数据传递给另一个线程。 *InterlockedXxxSList* 函数是一系列函数，用于维护无锁的共享单链表。 请注意，这些函数的 *Acquire* 和 *Release* 变体在 *Windows* 上不可用，但这些函数的常规版本在 *Windows* 上是一个完整的内存屏障。

```C++
// Declarations for the Task class go here.

// Add a new task to the list using lockless programming.
void AddTask(DWORD ID, DWORD data)
{
    Task* newItem = new Task(ID, data);
    InterlockedPushEntrySListRelease(g_taskList, newItem);
}

// Remove a task from the list, using lockless programming.
// This will return NULL if there are no items in the list.
Task* GetTask()
{
    Task* result = (Task*)
        InterlockedPopEntrySListAcquire(g_taskList);
    return result;
}
```


## 可变变量和重新排序 - *Volatile Variables and Reordering*
---
*C++* 标准规定，*volatile* 变量的读取不能缓存，*volatile* 变量的写入不能延迟，并且 *volatile* 变量的读写不能相互移动。这对于与硬件设备通信是足够的，这是 *C++* 标准中的 `volatile` 关键字的目的。

但是，标准的保证不足以使用 *volatile* 进行多线程处理。*C++* 标准不会阻止编译器对 *volatile* 变量和非 *volatile* 变量之间的读取和写入重新排序，也没有说明如何阻止 *CPU* 重新排序。

*Visual C++ 2005* 超越了标准 *C++*，为 *volatile* 变量访问定义多线程友好的语义。从 *Visual C++ 2005* 开始，对 *volatile* 变量的读取被定义为具有 *read-acquire* 语义，对 *volatile* 变量的写入被定义为具有 *write-release* 语义。这意味着编译器不会重新排列任何读取和写入操作，更进一步地，在 *Windows* 上，它将确保 *CPU* 也不会这样做。

请务必了解，这些新保证仅适用于 *Visual C++ 2005* 和 *Visual C++* 的未来版本。其他供应商的编译器通常会实现不同的语义，而没有 *Visual C++ 2005* 的额外保证。此外，在 *Xbox 360* 上，编译器不会插入任何指令来阻止 *CPU* 重新排序读取和写入。


## Lock-Free数据管道示例 - *Example of a Lock-Free Data Pipe*
---
管道是一种结构，它允许一个或多个线程写入数据，然后由其他线程读取。管道的无锁版本可以是在线程之间传递工作的一种优雅而有效的方式。*DirectX SDK* 提供了 *LockFreePipe*，这是一个单读、单写的无锁管道，可以在 *dxutlockfreeppe .h* 中获得。在 *Xbox 360 SDK* 中的 *atglockfreeppe .h* 中也可以使用相同的 *LockFreePipe*。

当两个线程具有生产者/消费者关系时，可以使用 *LockFreePipe*。生产者线程可以将数据写入管道，供消费者线程稍后处理，而不会阻塞。如果管道填满，写入将失败，生产者线程稍后必须重试，但仅当生产者线程过于领先时才会发生这种情况。如果管道清空，读取将失败，并且消费者线程稍后必须重试，但仅当消费者线程实在没有工作可做时，才会发生这种情况。如果两个线程达到良好的平衡，并且管道足够大，则管道允许它们顺利传递数据，且无延迟或阻塞。


## Xbox 360 上的性能 - *Xbox 360 Performance*
---
*Xbox 360* 上的同步指令和函数的性能将因正在运行的代码的不同而不同。如果另一个线程当前拥有锁，则获取锁需要更长的时间。如果其他线程写入同一缓存行，[*InterlockedIncrement*](https://learn.microsoft.com/en-us/windows/win32/api/winnt/nf-winnt-interlockedincrement) 和临界区操作将花费更长的时间。存储队列的内容也可能会影响性能。因此，所有这些数字都只是通过非常简单的测试生成的近似值。

但是，即使是从非常简单的测试生成的一些度量值也很有用：

- *lwsync* 测量结果为花费 33-48 个周期。
- *InterlockedIncrement* 测量结果为花费 225-260 个周期。
- 获取或释放临界区需要大约 345 个周期。
- 获取或释放互斥锁需要大约 2350 个周期。


## Windows 上的性能 - *Windows Performance*
---
*Windows* 上的同步指令和函数的性能因处理器类型和配置以及正在运行的代码的不同而不同。多核和多套接字系统执行同步指令通常需要更长的时间，如果另一个线程当前拥有该锁，则获取锁需要更长的时间。

但是，即使是从非常简单的测试生成的一些度量值也很有用：

- *MemoryBarrier* 测量结果为花费 20-90 个周期。
- *InterlockedIncrement* 测量结果为花费 36-90 个周期。
- 获取或释放临界区需要大约 40-100 个周期。
- 获取或释放互斥锁需要大约 750-2500 个周期。

这些测试是在一系列不同处理器上的 *Windows XP* 系统上完成的。较短的时间是在单处理器机器上，较长的时间是在多处理器机器上。

虽然获取和释放锁比使用无锁编程更昂贵，但本质上更少地共享数据是更好的做法，从根本上避免了成本。


## 关于性能的想法 - *Performance Thoughts*
---
获取或释放一个临界区包括一个内存屏障、一个 *InterlockedXxx* 操作和一些额外的检查，以处理递归并在必要时回退到互斥锁。你应该小心实现自己的临界区，因为在循环中自旋等待锁释放，而不回退到互斥锁，可能会浪费相当大的性能。当然，换句话说，对于竞争严重但不长时间保持的临界区，你应该考虑使用 [*InitializeCriticalSectionAndSpinCount*](https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-initializecriticalsectionandspincount)，这样操作系统将自旋一小段时间等待临界区可用，而不是在试图获取临界区时，如果该临界区已被拥有，就直接使用互斥锁。为了确定临界区可以从短暂的自旋中获益，有必要测量特定锁的等待时长，并加以比较。

如果将共享堆用于内存分配，则每次内存分配和释放的默认行为都涉及获取锁。随着线程数量和分配数量的增加，性能会从趋于平缓，到最终开始下降。让每个线程各使用一个堆，或者减少分配的数量，可以避免这种锁带来的瓶颈。

如果一个线程正在生成数据，而另一个线程正在使用数据，则它们最终可能会频繁共享数据。如一个线程正在加载资源，而另一个线程正在呈现场景，则可能会发生这种情况。如果绘制线程在每次绘制调用中引用共享数据，则锁开销将很高。如果每个线程都有专用数据结构，然后每帧同步一次（或更少），则可以实现更好的性能。

不能保证无锁算法一定比使用锁的算法快。在尝试不使用锁之前，应检查锁是否确实导致了性能问题，并且应该测量无锁代码是否确实提高了性能。


## 平台差异摘要 - *Platform Differences Summary*
---
+ *InterlockedXxx* 功能在 *Windows* 上阻止 *CPU* 读/写重新排序，但在 *Xbox 360* 上则不然。
+ 使用 *Visual Studio C++2005* 读取和写入 *volatile* 变量可以防止 *Windows* 上的 *CPU* 读/写重新排序，但在 *Xbox 360* 上，它只能防止编译器读/写的重新排序。
+ 写入在 *Xbox 360* 上会重新排序，但在 *x86* 或 *x64* 上不会。
+ 读取在 *Xbox 360* 上被重新排序，但在 *x86* 或 *x64* 上，它们仅相对于写入进行重新排序，并且仅当读取和写入的目标位置不同时。


## 建议 - *Recommendations*
---
- 尽可能使用锁，因为它们更易于正确使用。
- 避免过于频繁地使用锁，这样锁的成本就不会陡增。
- 避免锁的时间过长，以避免长时间停滞。
- 在适当的时候使用无锁编程，但要确保所获得的收益与复杂性相匹配。
- 在禁止使用其他锁的情况下，例如在延迟过程调用和普通代码之间共享数据时，请使用无锁编程或自旋锁。
- 仅使用经证明是正确的标准无锁编程算法。
- 执行无锁编程时，请务必根据需要使用 *volatile* 标志变量和内存屏障指令。
- 在 *Xbox 360* 上使用 *InterlockedXxx* 时，请使用 *Acquire* 和 *Release* 变体。


## 参考 - *References*
---
- MSDN 库  -  “[**volatile (C++)**](https://msdn.microsoft.com/library/12a04hfd(v=VS.71).aspx) “。C++ 语言参考。
- 万斯·莫里森  -  “[了解多线程应用中Low-Lock技术的影响](https://learn.microsoft.com/zh-cn/archive/msdn-magazine/2005/october/understanding-low-lock-techniques-in-multithreaded-apps)”。MSDN 杂志，2005 年 10 月。
- 里昂，迈克尔  -  “[PowerPC 存储模型和 AIX 编程](https://www-128.ibm.com/developerworks/eserver/articles/powerpc.mdl)”。IBM developerWorks，2005 年 11 月 16 日。
- 麦肯尼，保罗 E  -  “[现代微处理器中的内存序，第二部分](https://www.linuxjournal.com/article/8212)”。Linux Journal，2005 年 9 月。[本文包含一些 x86 详细信息。]
- Intel Corporation  -  “[Intel® 64 体系结构内存排序](https://www.cs.cmu.edu/~410-f10/doc/Intel_Reordering_318147.pdf)”。2007 年 8 月。 [适用于 IA-32 和 Intel 64 处理器。]
- 尼布勒，埃里克  -  “[行程报告：有关 C++ 线程的临时会议](https://www.artima.com/cppsource/threads_meeting.html)”。C++ 源，2006 年 10 月 17 日。
- 哈特， 托马斯 E. 2006.  -  “[快速实现无锁同步：内存回收的性能影响](https://www.cs.toronto.edu/~tomhart/papers/hart_ipdps06.pdf)”。2006 年国际并行和分布式处理研讨会 (IPDPS 2006) ，希腊罗得岛，2006 年 4 月。