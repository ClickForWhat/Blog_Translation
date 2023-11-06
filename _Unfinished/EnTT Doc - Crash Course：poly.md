*本文档译自 skypjack.github.io 中 EnTT 框架的静态多态库介绍文章，作者 Skypjack*

# 简介 - *Introduction*

静态多态性在 C++ 中是一个非常强大的工具，尽管有时候弄起来很麻烦。本模块旨在使其简单易用。

该库允许你实现*概念*，它是某个具体类需要实现的接口，不过具体类却不必从公共基类继承。

这是静态多态性的优点之一，也是 `poly` 类模板提供的通用包装器的优点之一。

用户得到的是一个可以直接传递的对象，而不是像使用动态多态性时那样通过引用或指针传递。

由于 `poly` 类模板在内部使用了 [`entt::any`](https://skypjack.github.io/entt/namespaceentt.html#a4846741b8f485584c196304f588b94ad)，所以它也支持 `entt::any` 的大部分特性。其中最重要的是，可以为现有的非托管对象创建别名。这允许用户在保持对象所有权的同时利用静态多态性。

同样，`poly` 类模板也受益于 `entt::any` 提供的小缓冲区优化，因此最小化了分配的数量，尽可能地避免了大量分配。

# 其他的库 - *Other libraries*

关于静态多态性有一些非常有趣的库。其中，我喜欢的两个是：

+ **dyno**：一个不错的运行时多态库。
+ **Poly**：一个类模板，让定义一个类型擦除的多态对象包装器变得更容易。

诚然，前者是一个实验性的库，有许多有趣的想法。不过我对某些功能在实际项目中的实用性有些怀疑，但也许是因为我在此缺乏经验。在我看来，它唯一的缺陷是 API，我发现它比其他解决方案稍微麻烦一些。后者无疑是这个模块的灵感来源，尽管我在最终 API 的实现和一些功能上选择了不同的方案。

不管怎样，它们的作者都是 C++ 社区的大师，我只需要向他们学习。

# 概念和实现 - *Concept and implementation*

要创建类型擦除的多态对象包装器的第一件事是定义一个*概念*，类型必须遵守它。

为此，标准库提供了一个类，它既支持推导的接口，也支持完整定义的接口。尽管拥有自动推导的接口十分方便，且能让用户写更少的代码，但是有一些限制。因此，我们可以通过为静态的虚表提供自定的定义来避免这种推导。

一旦定义好了接口，就需要一个通用实现来实现这个概念本身。

同样，在这种情况下，库允许基于类型或类型族进行自定义，以便能够在必要时考虑非泛型情景。

### 推导的接口 - *Deduced interface*

以下展示了一种方法，用于引入基于推导的接口：

```C++
struct Drawable : entt::type_list<> {
    template<typename Base>
    struct type : Base {
        void draw() { this->template invoke<0>(*this); }
    };
    // ...
};
// entt::type_list
// 是一个用于记录类型的类，仅此而已
```

容易看出，它从一个空类型列表（[`entt::type_lise<>`](https://skypjack.github.io/entt/structentt_1_1type__list.html)）继承而来。值得一提，函数也可以是 const，可以接受任意数量的形参，并返回非 void 类型：

```C++
struct Drawable : entt::type_list<> {
    template<typename Base>
    struct type : Base {
        bool draw(int pt) const { return this->template invoke<0>(*this, pt); }
    };
    // ...
};
```

在这个例子中，所有参数都是在 `this` 之后传递给 `invoke`，`invoke` 的返回值是内部调用返回的值。

`invoke` 是通过 `Base` 注入到概念的名字， 必须要继承 `Base`。不幸的是，由于 C++ 的语法规则，`this->template` 的形式是必须的。不过你可以使用外部调用的方案：

```C++
struct Drawable : entt::type_list<> {
    template<typename Base>
    struct type : Base {
        void draw() const { entt::poly_call<0>(*this); }
    };
    // ...
};
```

一旦定义了概念，用户必须提供它的通用实现，以便告诉 `any` 类型如何满足概念的要求。这是通过概念本身的别名模板完成的。作为模板参数传递给 `invoke` 或 `poly_call` 的索引指的是如何定义这个别名。

### 完整定义的接口 - *Defined interface*

完整定义的概念与推导接口的概念没有什么不同，唯一的区别是这次类型列表不是空的：

```C++
struct Drawable: entt::type_list<void()> {
    template<typename Base>
    struct type: Base {
        void draw() { entt::poly_call<0>(*this); }
    };
	
    // ...
};
```

同样，函数的参数和返回值可以不是 `void`。此外，当要绑定到函数的方法为 `const` 时，类型列表中的函数类型必须也是` const`：

```C++
struct Drawable: entt::type_list<bool(int) const> {
    template<typename Base>
    struct type: Base {
        bool draw(int pt) const { return entt::poly_call<0>(*this, pt); }
    };
	 
    // ...
};
```

如果推导出的类型和自行定义的类型相同，那用户为什么要自己定义呢？

实际上，这限制可以通过手动定义静态虚表来解决。

当有东西被推导时，有一个隐含的约束。

如果一个接口暴露了一个名为 `draw` 的函数，函数类型是 `void()`，那么就满足概念：

+ 通过公开具有相同名称和相同签名的成员函数的类。
+ 或者通过 lambda 使用接口本身的现有成员函数。

换句话说，不可能使用不属于接口的函数，即使它们是实现该概念的类型的一部分。类似地，不可能在静态虚表中推导出类型不同于接口本身关联成员函数类型的函数。

显式定义静态虚表可以减少推导步骤，并在提供概念的实现时提供最大的灵活性。

### 实现一个概念 - *Fulfill a concept*

概念的 `impl` 别名模板用于定义如何实现它：

```C++
struct Drawable: entt::type_list<> {
    // ...
	
    template<typename Type>
    using impl = entt::value_list<&Type::draw>;
};
// entt::value_list
// 一个用来记录常量的类，仅此而已。
```

在这个例子里，泛型类的 `draw` 方法足以满足 `Drawable` 概念的要求。

支持成员函数和自由函数来实现以下概念：

```C++
template<typename Type>
void print(Type &self) { self.print(); }
 
struct Drawable : entt::type_list<void()> {
    // ...
	
    template<typename Type>
    using impl = entt::value_list<&print<Type>>;
};
```

同样，如果函数的参数类型和返回类型支持与静态虚表中函数的相关类型进行转换，实际实现的函数类型可能会有所不同，因为它是在内部擦除的。

此外，`self` 参数并不是系统所严格要求的，如果不需要，可以省略。

有关详细信息，请参阅内联文档。

# 继承 - *Inheritance*

概念的继承简单直接。因此，如果有必要，很容易构建概念的层次结构。唯一的约束是，层次结构中的所有概念必须属于同一族，也就是说，它们必须要么全部推导，要么全部定义。

对于一个推导出来的概念，继承可以在几个步骤中实现：

```C++
struct DrawableAndErasable: entt::type_list<> {
    template<typename Base>
    struct type : typename Drawable::template type<Base> {
        static constexpr auto base = 
	        std::tuple_size_v<typename entt::poly_vtable<Drawable>::type>;
        void erase() { entt::poly_call<base + 0>(*this); }
    };
 
    template<typename Type>
    using impl = entt::value_list_cat_t<
        typename Drawable::impl<Type>,
        entt::value_list<&Type::erase>
    >;
};
```

静态虚表是空的，并且必须保持为空。

另一方面，`type` 不再继承 `Base`。相反，它将其模板形参转发给基类公开的类型。在内部，基类的静态虚表的大小用作本地索引的偏移量。

最后，通过 `value_list_cat_t`，实现包括将新函数附加到先前的列表中。

对于已定义的概念，类型列表的扩展方式与上述概念的实现类似。为此，声明一个允许将概念转换为其底层类型列表对象的函数是很有用的：

```C++
template<typename... Type>
entt::type_list<Type...> as_type_list(const entt::type_list<Type...> &);
```

该定义不是严格要求的，因为函数只通过如下的 `decltype` 使用：

```C++
struct DrawableAndErasable: entt::type_list_cat_t<
	    decltype(as_type_list(std::declval<Drawable>())),
	    entt::type_list<void()>>
{
    // ...
};
```

与上面类似，`type_list_cat_t` 用于将底层静态虚表与新的函数类型连接起来。其他所有内容都相同。

# 静态多态性

一旦定义了概念和实现，就可以使用 `poly` 类模板来包装满足需求的实例：

```C++
using drawable = entt::poly<Drawable>;
 
struct circle {
    void draw() { /* ... */ }
};
 
struct square {
    void draw() { /* ... */ }
};
 
// ...
 
drawable instance{ circle{} };
instance->draw();
 
instance = square{};
instance->draw();
```

这个类提供了广泛的构造函数，从默认的构造函数（它返回一个未初始化的 `poly` 对象）到复制和移动构造函数，以及就地创建对象的能力。

除此之外，还有一个构造函数允许用户将非托管对象包装在 `poly` 实例中（const 或非 const 对象）。

```C++
circle shape;
drawable instance{ std::in_place_type<circle &>, shape };
```

类似地，还可以从现有对象创建 `poly` 的非所有权副本：

```C++
drawable other = instance.as_ref();
```

在这两种情况下，尽管 `poly` 对象的接口没有改变，但它不会构造任何元素或销毁引用的对象。