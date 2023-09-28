*本文档译自 blog.molecular-matters.com 的 "Memory System" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
在我们真正深入研究 *Molecule* 引擎内存系统的内部工作原理之前，我们需要先了解一些基础知识。今天，让我们来重新审视一下 `new`，`delete` 和它们的朋友们。其中有一些令人惊讶的微妙之处，从我进行的采访来看，有时甚至资深人员也会混淆有关 `new` 和 `delete` 内部工作原理。

为了使事情更简单，我们不关心具体某个类的 `new` 和 `delete`，我们也不关心异常。


## new操作符/operator new* - *new operator/operator new*
---
让我们看一个包含 `new` 的简单语句：

```C++
T* i = new T;
```

这是 `new` 操作符的最简单形式。它在幕后到底做了什么？

1. 首先，调用 `operator new`  来分配类型 `T` 的存储空间。
2. 然后，`T` 的构造函数被调用，它在之前 `operator new` 返回的内存地址上构造 `T` 的新实例。

如果 `T` 是基本类型（例如 `int`，`float`），或者没有构造函数，那么就没有构造函数会被调用。上面的语句将调用最简单形式的 `operator new`：

```C++
void* operator new(size_t bytes);
```

编译器将自动调用参数为给定类型大小的 `operator new`，在本例中为 `sizeof(T)`。

到目前为止，一切顺利。但是我们还没有完成 `new` 操作符的全部工作。还有第二个版本的 `new` 操作符，即所谓的 `placement new`：

```C++
void* memoryAddress = (void*)0x100;
T* i = new (memoryAddress) T; // placement new
```

这可以用来在内存中的某个位置构造类的实例，这实际上是直接调用构造函数的唯一方法。这里不会产生内存分配，上面的语句实际上调用了 `operator new` 的不同重载，如下所示：

```C++
void* operator new(size_t bytes, void* ptr);
```

这种形式的 `operator new` 不会分配任何内存，最后只是返回指针。

`new` 操作符的 *placement* 语法非常强大，因为我们可以添加自己的 `operator new` 重载。唯一的规则是，每个 `operator new` 的第一个参数必须始终是 `size_t` 类型表示的对象大小，编译器将自动将其传递给它。

让我们看一个例子：

```C++
void* operator new(size_t bytes, const char* file, int line)
{
  // allocate bytes
}
 
// calls operator new(sizeof(T), __FILE__, __LINE__) to allocate memory
T* i = new (__FILE__, __LINE__) T;
```

不考虑全局 `operator new` 和类 `operator new` 之间的差异，每次使用 `placement new` 都可以总结如下：

```C++
// calls operator new(sizeof(T), a, b, c, d) to allocate memory
T* i = new (a, b, c, d) T;
```

等价于：

```C++
T* i = new (operator new(sizeof(T), a, b, c, d)) T;
```

调用 `operator new` 的神奇之处是这一切都是编译器完成的。此外，`operator new` 的每个重载都可以直接调用（如果你想的话），并且我们可以自行重载并做任何想做的事情。如果我们愿意，我们甚至可以使用模板：

```C++
template <class ALLOCATOR>
void* operator new(size_t bytes, ALLOCATOR& allocator, const char* file, int line)
{
	return allocator.Allocate(bytes);
}
```

当我们希望使用不同的分配器时，这将派上用场。使用 *placement* 语法，可以通过以下语句使用不同分配器的内存：

```C++
T* i = new (allocator, __FILE__, __LINE__) T;
```


## delete操作符/delete操作 - *delete operator/operator delete*
---
在先前新建的实例上调用 `delete` 操作符将首先调用析构函数，然后调用 `operator delete`。但是，与 `new` 不同的是：无论使用哪种形式的 `new` 创建实例，都将始终调用相同版本的 `operator delete`：

```C++
// calls operator new(sizeof(T), a, b, c, d)
// calls T::T()
T* i = new (a, b, c, d) T;
 
// calls T::~T()
// calls operator delete(void*)
delete i;
```

编译器调用相应 `operator delete` 的唯一时间是在 `operator new` 内部抛出异常时，因此我们可以借此在将异常传播到调用站点之前正确释放内存。这也是为什么每次重载 `operator new`都必须有相应版本的 `operator delete` 的原因。但现在让我们回到正题。

和 `operator new` 类似，`operator delete` 可以直接调用（如果你想的话）：

```C++
template <class ALLOCATOR>
void operator delete(void* ptr, ALLOCATOR& allocator, const char* file, int line)
{
	allocator.Free(ptr);
}
 
// call operator delete directly
operator delete(i, allocator, __FILE__, __LINE__);
```

但是，请记住，析构函数会被 `delete` 操作符自动调用，但 `operator delete` 不会。因此，在上面的示例中，需要手动调用析构函数：

```C++
// call the destructor before operator delete
i->~T();
 
// call operator delete directly
operator delete(i, allocator, __FILE__, __LINE__);
```

如果使用 `placement new` 创建实例，则必须始终手动调用析构函数。在这样的实例上使用 `delete` 操作符会产生未定义的行为（因为内存从来没有通过调用 `new` 来分配）。

到目前为止，我们只处理了 `new` 和 `delete` 的非数组版本。


## new[]/delete[] - *new[]/delete[]*
---
真正有趣的地方来了！大部分人并未发觉它，但是这是相当普遍、基本的，例如 `new[]` / `delete[]`。这里将涉及到编译器的魔法。*C++* 标准只规定了 `new[]` 和 `delete[]` 应该做什么，而没有规定如何做。

让我们从一个简单的示例开始：

```C++
int* i = new int [3];
```

上面通过调用 `new[]` 分配了三个 `int` 的空间，以及没有构造函数被调用。和 `operator new` 类似，我们能重载 `operator new[]` 或使用 *placement* 语法。

```C++
// our own version of operator new[]
void* operator new[](size_t bytes, const char* file, int line);
 
// calls the above operator new[]
int* i = new (__FILE__, __LINE__) int [3];
```

`delete[]` 和 `operator delete[]` 的行为与 `delete` 和 `operator delete` 相同。可以直接调用 `operator delete[]`，但必须确保手动调用析构函数。

但非 *POD* 的类型会发生什么？

```C++
struct Test
{
	Test(void)
	{
	    // do something
	}
	
	~Test(void)
	{
	    // do something
	}
	
	int a;
};
 
Test* i = new (__FILE__, __LINE__) Test [3];
```

虽然 `Test` 的大小等于 4，但我们的 `operator new[]` 接受的 `size_t` 参数将会是 16。为什么？首先思考数组会如何被 *delete*：

```C++
delete[] i;
```

编译器必须以某种方式知道要删除多少 `Test` 类型的实例，否则它不能调用实例的析构函数。因此，几乎每个编译器在调用 `new[]` 时会做的事情如下：

+ 对于 `T` 类型的 *N* 个实例，请求从 `operator new[]` 中分配的 `sizeof(T) * N + 4` 个字节。
+ 存储数字 *N* 到前四个字节。
+ 使用 `placement new` 构造 *N* 个实例，从第五个字节开始。
+ 返回第五个字节的地址给用户。

最后一点尤其重要：假设你重载了 `operator new[]` ，返回了内存地址 *0x100*，实例 `Test*` 的地址将是 *0x104*！这 16 个字节内存布局如下：

```c++
0x100: 03 00 00 00    -> 存储实例的数量（由编译器生成的代码）
0x104: ?? ?? ?? ??    -> i[0], Test* i
0x108: ?? ?? ?? ??    -> i[1]
0x10c: ?? ?? ?? ??    -> i[2]
```

当用户调用 `delete[]`，编译器将插入代码，通过给定指针的上四个字节读取实例的数量 *N*，然后按照逆序调用析构函数——如果被删除的类型是非 *POD* 的话。否则，不会增加 4 字节的开销，因为不需要调用析构函数（就像上面的 `new int[3]` 示例一样）。

不幸的是，当使用我们自己的 `operator new`、`operator new[]`、`operator delete` 和 `operator delete[]` 的重载时，这种编译器定义的行为会导致问题。尽管可以直接调用 `operator delete[]`，但还是需要弄清楚要调用多少析构函数（如果有的话）。

但我们做不到。

原因是我们永远无法确定编译器是否在分配中插入了额外的 4 个字节。这完全依赖于编译器。它可能会这样做，但也可能会与某些用户定义的类型发生严重冲突。每个编译器的行为都不一定相同。

这也是为什么对使用 `new[]` 分配的实例使用 `delete` 极有可能会崩溃。编译器生成的代码可能会试图访问不属于他的内存（使用 `delete[]` 删除通过 `new` 进行分配的内存），或数组的实例没被正确地析构（使用 `delete` 删除通过 `new[]` 进行分配的内存），以及其他未提及的情况。

不过，了解了调用 `new`、`new[]`、`delete` 和 `delete[]` 的幕后操作后，我们可以构建自己的分配函数，它可以正确处理所有类型的简单分配和数组分配，可以使用我们的自定义分配器，提供文件名和行号等附加信息。本系列的下一篇文章将展示如何做到这些。

同时，一定要阅读下面的文章，它们也解释了全局 `operator new` 和类 `operator new` 的概念：

+ [new (C++)](http://publib.boulder.ibm.com/infocenter/comphelp/v7v91/index.jsp?topic=%2Fcom.ibm.vacpp7a.doc%2Flanguage%2Fref%2Fclrc05cplr199.htm "C++ new operator")  
+ [delete (C++)](http://publib.boulder.ibm.com/infocenter/comphelp/v7v91/index.jsp?topic=%2Fcom.ibm.vacpp7a.doc%2Flanguage%2Fref%2Fclrc05cplr199.htm "delete (C++)")
