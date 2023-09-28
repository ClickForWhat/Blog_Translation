*本文档译自 blog.molecular-matters.com 中关于自定义C++委托的文章，作者 Stefan Reinalter*


## 概述 - *Overview*
---
虽然C#等其他语言提供了开箱即用的类型安全回调/委托，但遗憾的是C++没有提供这样的功能。但是委托和事件提供了一种很好的方式来“耦合”完全不相关的类，而无需编写大量样板代码。

这篇文章描述了此类委托和事件的通用且类型安全的实现，使用了高级C++功能，如非类型模板参数和模板偏特化，支持自由函数和成员函数，没有任何限制以及动态内存分配。

同样，让我们从一个如何使用委托的简单示例开始。假设 *MyDelegate* 是一个接受整数的委托，不返回任何值（如何声明这样的委托将在后面展示）：

```C++
void FreeFunction(int)
{
	// do something
}
 
class Class
{
public:
	void MemberFunction(int)
	{
		// do something
	}
};
 
MyDelegate delegate;
delegate.Bind(&FreeFunction);  // free function
delegate.Invoke(10);           // calls the bound function
 
Class c;
delegate.Bind(&Class::MemberFunction, &c);
delegate.Invoke(10);           // calls the bound function
```

从示例中可以看出，我们希望能够将任意的自由函数以及成员函数绑定到委托，只要它们与委托的签名匹配——在本例中为 `void (int)`。这两者之间的唯一区别是，成员函数只能在其类的实例上调用，因此我们需要将 *class* 对象传递给委托。

但是我们如何在内部存储指向自由函数和成员函数的指针呢？


## 指向函数的指针 - *Pointers to functions in C++*
---
在委托类中存储指向函数的指针很容易：

```C++
class Delegate
{
	typedef void (*FreeFunction)(int);
	
public:
	void Bind(FreeFunction function)
	{
	    m_function = function;
	}
	
	void Invoke(int ARG0)
	{
	    (m_function)(ARG0);
	}
	
private:
	FreeFunction m_function;
};
```

如果你以前从未在C++中使用过指向函数的指针，*typedef* 定义了一个类型 `FreeFunction`，它是指向任何接受整数并返回 *void* 的自由函数的指针。这样的自由函数可以是任何全局函数、名称空间内的函数或类静态函数，等不是类普通成员函数的函数。


## 指向成员函数的指针 - *Pointers to member functions*
---
在c++中，指向成员函数的指针是完全不同的东西。

首先，它们的类型不同于普通的函数指针：

```C++
void FreeFunction(int);       // type: void (*)(int)
 
class Class
{
	void MemberFunction(int);   // type: void (Class::*)(int)
};
```

可以看出，指向成员函数的指针也携带类的类型，因此指向成员函数的指针不能在不同的类之间交换。

其次，指向自由函数的指针与 *void\** 的大小相同，而和指向成员函数的指针不同。这意味着指向不同成员函数的指针可以有不同的大小：

```C++
struct Base
{
	virtual void Do(int) {}
};
 
struct Derived1 : public Base
{
	virtual void Do(int);
};
 
struct Derived2 : public Base
{
	virtual void Do(int);
};
 
struct MultipleInheritance : public Derived1, public Derived2
{
	virtual void Do(int);
};
 
struct VirtualMultipleInheritance : virtual public Derived1, virtual public Derived2
{
	virtual void Do(int);
};
 
ME_LOG0("size: %d", sizeof(&Base::Do));
ME_LOG0("size: %d", sizeof(&Derived1::Do));
ME_LOG0("size: %d", sizeof(&Derived2::Do));
ME_LOG0("size: %d", sizeof(&MultipleInheritance::Do));
ME_LOG0("size: %d", sizeof(&VirtualMultipleInheritance::Do));
```

在使用 *Visual Studio 2010* 的机器上，我得到了以下输出：

	size: 4
	size: 4
	size: 4
	size: 8
	size: 12

这可能会让人感到惊讶，但标准并没有规定指向成员函数的指针的严格大小。通常，在 *Visual Studio* 中，指向成员函数的指针将占用4、8或12字节（分别用于单继承、多继承和虚拟继承）。更深入的讨论在[这里](http://blogs.msdn.com/b/oldnewthing/archive/2004/02/09/70002.aspx)。

上述也是反对将指向成员函数的指针转换为 *void\** 的一个非常有力的论据，它们并不适合，而且你的程序被破坏了（即使它可能在你的机器上工作正常）。

回到委托，如何将指向成员函数的指针转换为相同类型？一种办法是将类型擦除与抽象基类结合使用：

```C++
class AbstractBase
{
public:
	virtual void CallFunction(int ARG0) = 0;
};
 
template <class C>
class MemberFunctionWrapper : public AbstractBase
{
	typedef void (C::*MemberFunction)(int);
	
public:
	MemberFunctionWrapper(MemberFunction memberFunction)
	{
	    m_memberFunction = memberFunction;
	}
	
	virtual void CallFunction(int ARG0)
	{
	    (m_instance->*m_memberFunction)(ARG0);
	}
	
private:
	C* m_instance;
	MemberFunction m_memberFunction;
};
```

使用类型擦除，类 `AbstractBase` 的具体实现仍然知道他们正在处理的类的类型（模板`<class C>`），并且可以在没有任何讨厌的强制转换或其他非法语句的情况下调用成员函数。`AbstractBase` 的实例只使用C++的虚拟函数机制将函数调用调度到底层实现。这意味着我们可以通过存储 `AbstractBase` 实例来存储指向成员函数的指针：

```C++
AbstractBase* function = new MemberFunctionWrapper<Class>(&Class::Do);
```

这当然是一种有效且合法的方法，但它有使用动态内存分配以及通过虚函数添加另一层间接的缺点。

让我们回到最初的委托，看看是否有替代解决方案。如果我们可以将指向成员函数的指针转换为指向普通函数的指针，我们只需要存储后者，而完全不关心成员函数。

我们可以尝试以以下方式编写这样一个包装器函数：

```C++
template <class C>
void WrapMemberFunction(C* instance, void (C::*memberFunction)(int), int ARG0)
{
	(instance->*memberFunction)(ARG0);
}
```

上面的方法是可行的，但是我们不能将它存储在 `Delegate` 类中，因为 `WrapMemberFunction` 与普通的自由函数具有不同的签名：

```C++
typedef void (*FreeFunction)(int);
typedef void (*WrapperFunction)(C*, void (C::*)(int), int);
```

类型 *C* 仍然潜伏在 `typedef` 中，这样我们需要一个额外的 *C\** 参数，它是成员函数指针被调用的实例。

如果要在自由函数中引入一个额外的、未使用的参数，并使用普通的、老式的 *void\** 指针，那么我们又回到了糟糕的原点：

```C++
void WrapFreeFunction(void* instance, void (*freeFunction)(int), int ARG0)
{
	// instance is unused
	(freeFunction)(ARG0);
}
 
template <class C>
void WrapMemberFunction(C* instance, void (C::*memberFunction)(int), int ARG0)
{
	(static_cast<C*>(instance)->*memberFunction)(ARG0);
}
```

注意，这里使用 *void\** 并没有牺牲类型安全，函数模板总是知道原始类型，并正确地将其重新强制转换（根据标准，任何指针转换到 *void\** 再转回来是安全的）。

第二个参数仍然不匹配。我们需要彻底消除它，我们可以通过使用C++的一个高级模板功能来做到这一点——使用[非类型模板参数](http://publib.boulder.ibm.com/infocenter/comphelp/v8v101/index.jsp?topic=%2Fcom.ibm.xlcpp8a.doc%2Flanguage%2Fref%2Fnon-type_template_parameters.htm)，将指向自由函数的指针和指向成员函数的指针作为模板参数传递。

这意味着我们可以将委托转换为以下内容：

```C++
class Delegate
{
	typedef void* InstancePtr;
	typedef void (*InternalFunction)(InstancePtr, int);
	typedef std::pair<InstancePtr, InternalFunction> Stub;
	
	// turns a free function into our internal function stub
	template <void (*Function)(int)>
	static ME_INLINE void FunctionStub(InstancePtr, int ARG0)
	{
	    // we don't need the instance pointer because we're dealing with free functions
	    return (Function)(ARG0);
	}
	
	// turns a member function into our internal function stub
	template <class C, void (C::*Function)(int)>
	static ME_INLINE void ClassMethodStub(InstancePtr instance, int ARG0)
	{
	    // cast the instance pointer back into the original class instance
	    return (static_cast<C*>(instance)->*Function)(ARG0);
	}
	
public:
	Delegate(void)
	    : m_stub(nullptr, nullptr)
	{
	}
	
	/// Binds a free function
	template <void (*Function)(int)>
	void Bind(void)
	{
	    m_stub.first = nullptr;
	    m_stub.second = &FunctionStub<Function>;
	}
	
	/// Binds a class method
	template <class C, void (C::*Function)(int)>
	void Bind(C* instance)
	{
	    m_stub.first = instance;
	    m_stub.second = &ClassMethodStub<C, Function>;
	}
	
	/// Invokes the delegate
	void Invoke(int ARG0) const
	{
	    ME_ASSERT(m_stub.second != nullptr, "Delegate unbounded.Call Bind() first.")();
	    return m_stub.second(m_stub.first, ARG0);
	}
	
private:
	Stub m_stub;
};
```

自由函数和成员函数包装器现在共享完全相同的签名，因此可以存储在 `Delegate` 类中。委托现在既可以用于自由函数，也可以用于成员函数：

```C++
void FreeFunction(int)
{
	// do something
}
 
class Class
{
public:
	void MemberFunction(int)
	{
	    // do something
	}
};
 
MyDelegate delegate;
delegate.Bind<&FreeFunction>();  // free function
delegate.Invoke(10);             // calls the bound function
 
Class c;
delegate.Bind<Class, &Class::MemberFunction>(&c);
delegate.Invoke(10);             // calls the bound function
```

剩下要做的就是将委托转换为类模板，这样它就可以用于任意返回类型和参数。

这可以通过使用模板偏特化轻松实现：

```C++
// base template
template <typename T>
class Delegate {};
 
template <typename R>
class Delegate<R ()>
{
	// implementation for zero arguments, omitted
};
 
template <typename R, typename ARG0>
class Delegate<R (ARG0)>
{
	// implementation for one argument, omitted
};
 
template <typename R, typename ARG0, typename ARG1>
class Delegate<R (ARG0, ARG1)>
{
	// implementation for two arguments, omitted
};
 
// other partial template specializations
// ...
 
// defining any delegate in user-code
typedef Delegate<void (int, const char*) MyDelegate;
typedef Delegate<bool (const Vec2&)> IntersectionDelegate;
// etc.
```

就我个人而言，我并没有手工编写所有的实现，而是使用了[基于宏的方法](http://altdevblogaday.com/2011/07/12/abusing-the-c-preprocessor/)，并在 *Visual Studio* 中使用了预构建步骤。这样，只需编写一次实现，但保持所有的好处（可读性、可调试性），就像您实际手工编写了所有实现一样。

在性能方面，委托实现提供了与简单的自由函数指针完全相同的性能，因为指针被作为模板参数传递，并且 *stub* 函数被标记为 `ME_INLINE(forceinline)`，迫使编译器在调用点进行内联调用（可使用 *Visual Studio 2010* 进行分析）。


## 事件 - *Events*
---
使用相同的方法，我们可以通过存储 *stub* 数组而不是单个 *stub*，轻松地将上述委托实现扩展到一个成熟的事件系统。

在 *Molecule* 中，事件系统分为 `event` 和相应的 `event::Sink`。`Event::Sink` 负责存储某个事件的所有侦听器，并且可以使用 `Event::Bind()` 将这样的接收器绑定到事件。使用 `Event::Signal()`，绑定接收器内的所有侦听器都将收到事件通知。

将事件系统拆分为这两部分，可以将 `event` 作为类中的成员存储在某个位置，从而允许通知不同的接收器/侦听器，同时永远不会向类的用户公开 `Signal()` 功能。


## 用例 - *Usage*
---
例如，*Molecule* 已经提供了一个完全通用的热重载系统，它大量使用了事件系统。只需在某个地方定义一个 `DirectoryWatcher`，并连接一些事件侦听器：

```C++
// store some event listeners in a sink
DirectoryWatcher::ActionEvent::Sink eventSink;
eventSink.AddListener<&ConfigHotReloader::OnAction>();
eventSink.AddListener<&TextureReloader::OnAction>();
eventSink.AddListener<&ResourceManager::OnAction>();
eventSink.AddListener<&User_OnFileAction>();  // some user-defined function
 
// hook them to the directory watcher
DirectoryWatcher directoryWatcher(currentDirectory);
directoryWatcher.Bind(&eventSink);
 
// tick the directory watcher once a frame
directoryWatcher.Tick();
```

使用委托和事件系统耦合不相关类的另一个例子是将物理设备的低级输入绑定到高级逻辑设备，但这将在另一篇文章中讨论。