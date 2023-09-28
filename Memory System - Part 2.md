*本文档译自 blog.molecular-matters.com 的 "Memory System" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
在这个系列的上一篇文章，我们介绍了 `new`，`delete` 和其他变体的行为。这篇文章，我们将构建自己的迷你工具集，允许我们使用自定义的分配器创建任意类型的实例（以及它的数组），并且它符合标准。准备好一些函数模板、基于类型的分发、模板魔术和漂亮的宏！

在 *Molecule*  的内存系统中，我希望使用以下语法创建新的实例（不是合法的 C++）：

```C++
Arena arena; // one of many memory arenas
// ...
Test* test = new (arena, additionalInfo) Test(0, 1, 2);
delete (test, arena); // no placement-syntax of delete
// ...
Test* test = new (arena, additionalInfo) Test[10];
delete[] (test, arena); // no placement-syntax of delete[]
```

这里 `new` 操作符可以正常工作，但上面的操作在 *C++* 中还是不合法的，因为 `delete` 操作符没有 *placement* 语法，也就是说它只接受一个参数。我们也可以直接通过调用 `operator delete` 来完成上面所需的操作，但是我们会遇到 `operator delete[]` 的问题，就像上次讨论的那样，我们不能以编译器无关的、可移植的方式调用析构函数。

此外，我们希望将一些额外的信息，如文件名、行号、类名、内存标记等传递给 *Memory Arena*（它反过来负责分配内存），因此这是一个基于宏的解决方案。尽管如此，我们仍希望尽可能多地保留原始 *C++* 语法，因此我们的目标解决方案类似于以下代码：

```C++
Test* test = ME_NEW(Test, arena)(0, 1, 2);
ME_DELETE(test, arena);
// ...
Test* test = ME_NEW_ARRAY(Test[3], arena);
ME_DELETE_ARRAY(test, arena);
```

让我们从最不复杂的开始，逐一处理宏和底层函数的实现。


## ME_NEW - *ME_NEW*
---
类似于平常的 `new` 操作符，`ME_NEW` 首先需要给指定类型分配内存，然后在正确的位置调用构造函数。足够简单，可以只用一行完成：

```C++
#define ME_NEW(type, arena) \
new (arena.Allocate(sizeof(type), __FILE__, __LINE__)) type
```

我们所做的是使用 `placement new` 和调用 `arena. allocate()` 返回给我们的内存地址，同时使用预处理器符号 `__FILE__` 和 `__LINE__` ，向 `arena` 提供进行分配的文件名和行号。此外，在末尾添加了 `type`，因此你仍然可以在宏调用之后提供构造函数参数，就像这样：

```C++
// Test is a class taking 3 ints in the constructor
Test* test = ME_NEW(Test, arena)(0, 1, 2);
 
// the preprocessor expands the above into:
Test* test = new (arena.Allocate(sizeof(Test), "test.cpp", 123)) Test(0, 1, 2);
```

使用 `ME_NEW`，我们可以为它提供一个 *Memory Arena*，它负责分配内存，向它传递额外的信息，并且仍然具有类似于原始 `new` 操作符语法的语法。那么让我们转到下一个宏。


## ME_DELETE - *ME_DELETE*
---
使用 `ME_NEW` 创建的每个实例都需要通过调用 `ME_DELETE` 来删除。请记住，`delete` 操作符没有  *placement* 形式，因此必须使用 `operator delete` 或其他完全不同的操作符，否则无法传递额外的 `arena` 参数。无论哪种方式，都必须确保为给定实例调用析构函数。我们可以通过将对象的实际删除推迟到辅助函数中：

```C++
#define ME_DELETE(object, arena) Delete(object, arena)
```

我们的辅助函数是使用一个简单的函数模板实现的：

```C++
template <typename T, class ARENA>
void Delete(T* object, ARENA& arena)
{
    // call the destructor first...
    object->~T();
	
    // ...and free the associated memory
    arena.Free(object);
}
```

编译器可以为我们推断出这两个模板实参，因此我们不需要显式指定任何模板实参。到目前为止，一切顺利。转到下一个宏。


## ME_NEW_ARRAY - *ME_NEW_ARRAY*
---
事情开始变得越来越复杂，所以请继续阅读。我们首先需要的是一个辅助函数，它为特定类型的 *N* 个实例分配内存，并使用 `placement new` 适当地构造它们。因为它必须适用于泛型类型和 `arena`，所以我们再次使用函数模板：

```C++
template <typename T, class ARENA>
T* NewArray(ARENA& arena, size_t N, const char* file, int line)
{
	union
	{
	    void* as_void;
	    size_t* as_size_t;
	    T* as_T;
	};
	
	as_void = arena.Allocate(sizeof(T) * N + sizeof(size_t), file, line);
	
	// store number of instances in first size_t bytes
	*as_size_t++ = N;
	
	// construct instances using placement new
	const T* const onePastLast = as_T + N;
	while (as_T < onePastLast)
		new (as_T++) T;
	
	// hand user the pointer to the first instance
	return (as_T - N);
}
```

代码的作用应该从注释中清楚地显示出来。需要注意的一点是，我们为 *N* 个类型为 `T` 的实例分配了内存，并为存储实例的数量 *N* 分配了额外的大小为 `sizeof(size_t)` 字节的内存。那么，分配 3 个 `T` 类型实例后，我们的内存布局看起来像这个样子：

```
Bytes 0-3: N
Bytes 4-7: T[0]
Bytes 8-11: T[1]
Bytes 12-15: T[2]
```

返回给用户的指针偏移了 `sizeof(size_t)` 字节，因此它正确地指向第一个实例。如果我们要在 *C++* 代码中直接调用这个函数，它将看起来像下面这样：

```C++
Test* t = NewArray<Test>(arena, 3, __FILE__, __LINE__);
```

这里有一个小问题，因为类型 `T` 没有在任何地方作为函数参数（而只是作为返回类型），编译器无法推断其类型。因此，我们必须在函数调用中直接指定它，但我们没法从 `ME_NEW_ARRAY` 宏中既得知类型，又得知实例数量。你可以回想一下相应的宏调用应该是什么样子：

```C++
Test* test = ME_NEW_ARRAY(Test[3], arena);
```

我们真的想达成原先的目标，并保持这种语法，那么我们该怎么做呢？让我们尝试编写宏，并填充缺失的部分：

```C++
#define ME_NEW_ARRAY(type, arena) NewArray<?>(arena, ?, __FILE__, __LINE__)
```

我们现在需要的是某种辅助函数，它能够从单个参数中推断出类型和数量。让我们用一些模板魔术来完成这个任务！

```C++
template <class T>
struct TypeAndCount
{
};
 
template <class T, size_t N>
struct TypeAndCount<T[N]>
{
  typedef T Type;
  static const size_t Count = N;
};
```

模板 `TypeAndCount` 只接受一个模板参数，并且是完全空的。但是，通过为形式为 `T[N]` 的类型提供模板偏特化，我们可以在编译时定义类型并提取 *N*。如果你不习惯这样的编译时技巧，那么让我向你展示如何通过在宏中填充缺失的部分来使用它：

```C++
#define ME_NEW_ARRAY(type, arena) \
NewArray<TypeAndCount<type>::Type>(arena, TypeAndCount<type>::Count, __FILE__, __LINE__)
```

看到我们做的了吗？让我们通过查看调用 `ME_NEW_ARRAY(Test[3], arena)` 的相应源代码来了解所涉及的不同步骤。

首先，是预处理器的环节：

1. 宏的 `TypeAndCount<type>::Type` 部分会展开为 `TypeAndCount<Test[3]>::Type`。
2. 宏的 `TypeAndCount<type>::Count` 部分会展开为 `TypeAndCount<Test[3]>::Count`。

然后，是编译器的环节：

1. `TypeAndCount<Test[3]>::Type` 将通过模板偏特化形成 `Test`。
2.  `TypeAndCount<Test[3]>::Count` 将通过模板偏特化形成 3。

因此，我们只需将这两个值插入到 `ME_NEW_ARRAY` 宏中，而不必在宏中提供任何其他（冗余）参数。


## ME_DELETE_ARRAY - *ME_DELETE_ARRAY*
---
啊，最后一个目标了。同样，我们需要提出一个辅助函数，它首先以相反的顺序调用析构函数（按照 *C++* 标准），然后释放相关的内存。话不多说，下面是实现：

```C++
template <typename T, class ARENA>
void DeleteArray(T* ptr, ARENA& arena, NonPODType)
{
	union
	{
	    size_t* as_size_t;
	    T* as_T;
	};
	
	// user pointer points to first instance...
	as_T = ptr;
	
	// ...so go back size_t bytes and grab number of instances
	const size_t N = as_size_t[-1];
	 
	// call instances' destructor in reverse order
	for (size_t i = N; i > 0; --i)
	    as_T[i - 1].~T();
	
	arena.Free(as_size_t - 1);
}
```

通过阅读注释，实现应该是清晰的。宏也不是太难，因为我们不需要显式地指定编译器可以推导出的函数模板参数。因此，这个宏非常简单：

```C++
#define ME_DELETE_ARRAY(object, arena)    DeleteArray(object, arena)
```

这就完成了！

基本上，我们已经完成了，我们已经实现了我们最基础的目标，并且对 *POD* 和 非 *POD* 类型都有效。不过，这里还有东西可以优化：记住我们不需要对 *POD* 类型执行构造/析构，所以我们可以优化 `NewArray` 和 `DeleteArray` 函数模板。

这可以通过基于类型的分发与模板的 *type traits* 相结合来实现，但是解释和实现必须等到本系列的下一部分，因为这篇文章已经比预期的要长了。