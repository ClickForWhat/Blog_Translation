*本文档译自 buildingblock.ai 的 "allocator-aware-smart-ptr" 文章，作者 Ryan Burn*


## 概述 - *Overview*
---
如何以及在哪里分配内存会对应用程序的性能产生巨大影响。

*C++ 17* 引入了多态分配器和分配器感知类型的概念，使得部署自定义分配策略比以往任何时候都更容易。

虽然 *C++ 17* 已经打包了许多有用的分配器感知类型，但它缺少一个重要的组件：分配器感知的智能指针。

编写一个分配器感知的智能指针是棘手的，但并非不可能。本指南将向您展示如何操作。


## 分配器感知类型意味着什么 - *What Does it Mean for a Type to be Allocator-aware?*
---
在深入了解分配器感知的智能指针的细节之前，让我们简要讨论一下若一个类型是分配器感知的，那究竟意味着什么。

假设有一个类型，叫做 `my_container`，那么分配器感知可以：

1. 提供查询它的分配器的机制：
```C++
my_container::allocator_type
  // 这可以向我们展示分配器的类型;
  // 在本教程中，我们假设它总是 std::pmr::polymorphic_allocator

const my_container& cont = /* 一个 my_container 实例的引用*/;
cont.get_allocator(); // 返回和 `cont` 关联的分配器
```

2. 允许你指定要使用的分配器的重载构造函数：
```C++
// 假设有如下声明/初始化
my_container cont{ "abc", 123 };

// 那么我们也可以这样做
std::pmr::polymorphic_allocator alloc = /* 某个自定义分配器 */
my_container cont{ "abc", 123, alloc };
```

感知分配器的类型是可组合的。如果 `my_container` 的数据成员也是分配器感知的，那它会在构造时将其分配器转发给它们。如果我们这么写：

```C++
std::pmr::polymorphic_allocator<> alloc = /* 某个自定义分配器 */
std::pmr::vector<std::pmr::string> v{ alloc };
v.emplace_back("abc123");
```

那么 `v` 本身和它拥有的所有字符串都将从 `alloc` 中分配。

不过，分配器感知类型的移动构造函数和移动赋值操作符需要满足某些要求。

假设我们构造一个分配器感知类型的实例，然后用不同的分配器移动构造另一个实例：

```C++
std::pmr::polymorphic_allocator<> alloc1 = /* 某个分配器 */
std::pmr::polymorphic_allocator<> alloc2 = /* 和 alloc1 不同的另一个分配器 */
assert(alloc1 != alloc2);

my_container cont1{ alloc1 };
my_container cont2{ std::move(cont1), alloc2 };
```

`my_container` 将无法把它拥有的数据从 `cont1` 直接移动到 `cont2`。它仍必须从 `alloc2` 中分配一片内存，然后再将 `cont1` 的数据复制进去。

为什么？

1. 首先，与分配器相关联的 `memory_resource` 的生命周期可能不同。若我们把内存数据从 `cont1` 移到了 `cont2`，如果某时刻销毁了 `alloc1` 的资源，则可能会留下悬空指针。
2. 分配器感知软件的一个用例是最大化内存局部性并防止内存弥散。

假设有如下代码：

```C++
std::pmr::unsynchronized_pool_resource pool;
std::pmr::polymorphic_allocator<> alloc1{ &pool };
std::pmr::vector<my_container> collection{ alloc1 };

// 我们的程序运行了一段时间，collection 里已经有一些数据

my_container cont = /* 从我们的程序中生成并使用全局分配器的一些容器 */;
collection.emplace_back(std::move(cont));

// later... traverse and process collection
```

当我们将 `cont` 移动插入到 `collection` 中时，将使用 `alloc1` 分配 `my_container` 的一个新实例，并使用 `cont` 和 `alloc1` 的拷贝调用移动构造函数。

乍一看，这似乎是多余的步骤；但是通过强制使用 `alloc1` 分配 `collection` 的内存，我们可以确保 `collection` 在内存中具有局部性布局。

之后，如果同时访问和修改 `collection` 中的连续条目，那么对于性能而言，进行额外分配所付出的成本和更好的局部性相比，后者对性能的提升将弥补前者带来的损失。

*Bloomberg 进行了广泛的基准测试，以衡量局部性对性能的重要性。他们对各种场景进行了基准测试，发现在许多情况下，保持局部性的分配器可以将性能提高4到8倍。[参考](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0089r1.pdf)*


## 如何构建分配器感知的智能指针 - *How Can We Make a Smart Pointer Allocator-aware?*
---
既然我们已经讨论了分配器感知类型的要求，那么让我们考虑如何创建一个分配器感知的智能指针。

我们假设像 `std::unique_ptr` 一样，我们的智能指针也有只移动语义，所以我们可以将类勾画为：

```C++
template <class T>
class managed_ptr {
public:
    using allocator_type = std::pmr::polymorphic_allocator<>;
	
    managed_ptr() noexcept = default;
	
	// 普通构造函数
    template <class U>
      requires std::convertible_to<U*, T*> && 
		       // ???
    managed_ptr(U* ptr, allocator_type alloc = {}) noexcept {
	    // ???
    }
	
	// 弃置的拷贝构造函数
    managed_ptr(const managed_ptr&) = delete;
	
	// 移动构造函数
    template <class U>
      requires std::convertible_to<U*, pointer>
    managed_ptr(managed_ptr<U>&& other) noexcept {
	    // ???
    }
    
	// 指定分配器的“移动构造”函数
    template <class U>
      requires std::convertible_to<U*, pointer>
    managed_ptr(managed_ptr<U>&& other, allocator_type alloc)
      : alloc_{ alloc }
    {
	    this->operator=(std::move(other));
    }
	
	// 析构函数
    ~managed_ptr() noexcept {
	    his->reset();
    }
	
	// 弃置的拷贝赋值运算符
    managed_ptr& operator=(const managed_ptr&) = delete;
	
	// 移动赋值运算符
    template <class U>
      requires std::convertible_to<U*, pointer>
    managed_ptr& operator=(managed_ptr<U>&& other) {
	    // ???
    }
	
    allocator_type get_allocator() const noexcept {
	    return alloc_;
    }
	
    void reset() noexcept {
	    // ???
    }
	
    // 此处省略其他标准指针方法，如 get、operator*、operator->、release 等
	
private:
    allocator_type alloc_;
    T* ptr_ = nullptr;
    // ???
};
```

`managed_ptr` 允许在构造时指定分配器，并允许查询分配器，但是如何填充方法以满足分配器感知类型的其他要求呢？

回想一下，智能指针可以指向基类，因此编写如下代码应该是合法的：

```C++
class A {
public:
    virtual ~A() noexcept = default;
	
    // ....
};

class B : public A {
	//...
};

std::pmr::polymorphic_allocator<> alloc1 = /* 某个自定义分配器 */
managed_ptr<A> ptr1{ alloc1.new_object<B>(/* 参数 */), alloc1 };

std::pmr::polymorphic_allocator<> alloc2 = /* 另一个自定义分配器 */
managed_ptr<A> ptr2{ std::move(ptr1), alloc2 };
```

由于分配器并不相等，`ptr2` 必须从 `alloc2` 中为派生类型 `B` 分配新的内存，并移动构造 `B` 的实例，以便让它的构造等同于：

```C++
managed_ptr<A> ptr2{
    alloc2.new_object<B>(std::move(*static_cast<B*>(ptr1.get()))),// 移动构造
	alloc2
};
ptr1.reset();
```

为此，我们需要对派生类的移动构造函数进行类型擦除。

但这还不够。假设析构一个 `managed_ptr` 对象：

```C++
managed_ptr<A> ptr1{ alloc1.new_object<B>(/* 参数 */), alloc1 };
ptr1.reset();
```

与全局 `delete` 不同，`std::pmr::polymorphic_allocator<>` 在进行解分配时需要传递分配的大小。

我们将使用一个函数指针来擦除这两个信息。现在让我们填充构造函数。

```C++
template <class T>
class managed_ptr {
    using pointer_operator = void* (*)(void*, std::pmr::polymorphic_allocator<>, bool);
public:
    // ...
    template <class U>
      requires std::convertible_to<U*, T*> &&
               std::move_constructible<U>
    managed_ptr(U* ptr, allocator_type alloc = {}) noexcept
      : alloc{ alloc }, ptr_{ ptr }
    {
	    operator_ =
	    [](void* ptr, allocator_type alloc, bool construct) {
	        U* derived = static_cast<U*>(ptr);
	        if (construct) {
			    return static_cast<void*>(
				    alloc.new_object<U>(std::move(*derived)) // 移动构造
			    );
	        } else {
	            std::allocator_traits<allocator_type>::destroy(alloc, derived);
	            alloc.deallocate_object(derived);
	            return nullptr;
	        }
        };
    }
	
    template <class U>
      requires std::convertible_to<U*, pointer>
    managed_ptr(managed_ptr<U>&& other) noexcept {
      : alloc_{other.alloc},
        ptr_{other.release()},
        operator_{other.operator_}
    {}
	
private:
    allocator_type alloc_;
    T* ptr_ = nullptr;
    pointer_operator operator_;
};
```

现在我们可以使用 `operator_` 来填充移动赋值操作符：

```C++
template <class T>
class managed_ptr {
	// ...
    template <class U>
      requires std::convertible_to<U*, pointer>
    managed_ptr& operator=(managed_ptr<U>&& other) {
	    operator_ = other.operator_;
	    if (alloc_ == other.alloc_) {
		    // 若分配器是一样的，就不需要重新分配
	        ptr_ = other.release();
	        return *this;
	    }
	    // 使用正确的分配器重新分配，然后移动构造派生类型的实例。
	    ptr_ = static_cast<T*>(
	        operator_(static_cast<void*>(other.ptr_), alloc_, true)
	    );
	    other.reset();
	    return *this;
    }
	// ...
};
```

类似地，我们可以使用 `operator_` 来实现 `reset` 方法：

```C++
template <class T>
class managed_ptr {
	// ...
	void reset() noexcept {
	    if (ptr_ == nullptr) {
		    return;
	    }
	    operator_(static_cast<void*>(ptr_), alloc_, false);
	    ptr_ = nullptr;
	}
	// ...
};
```

这种方法适用于大多数情况，但是多重继承呢？

假设我们有：

```C++
class A1 {
public:
	virtual ~A1() noexcept = default;
	
	// ...
private:
	int a1;
}

class A2 {
public:
	virtual ~A2() noexcept = default;
	
	// ...
private:
	int a2;
};

class B : public A1, public A2 {
	// ....
};

std::pmr::polymorphic_allocator<> alloc = /* 分配器 */
auto bptr = alloc.new_object<B>(/* 参数 */);
managed_ptr<A2> ptr{ bptr, alloc };
```

在这种情况下，`ptr.get()` 和 `bptr` 不会指向相同的内存区域，因此 `operator_` 函数将无法工作。

这可以通过添加一个 `void*` 数据成员来修复，该成员跟踪用于构造和新分配偏移的原始指针。

我不会在本指南中讨论所有细节，但是你可以在这里[查看完整的源代码](https://github.com/rnburn/bbai-mem/blob/master/bbai/memory/management/managed_ptr.h)。


## 例程：解析 Json - *An Example: Parsing Json*
---
让我们在一个示例中看看 `managed_ptr` 是如何工作的。

我们将研究一个解析 *json* 简化子集的应用程序。例如，它只处理整数和整数数组：

```json
[
  1, 2, 3,
  [4, 5, [6], 7]
]
```

我们将用多态类型表示解析后的 *json*。

```C++
enum class json_value_type { number, array };

class json_value {
public:
	virtual ~json_value() noexcept = default;
	
	virtual json_value_type type() const noexcept = 0;
};

class json_number final : public json_value {
public:
	explicit json_number(int value) noexcept : value_{value} {}
	
	// json_value
	json_value_type type() const noexcept override { 
		return json_value_type::number; 
	}
	
private:
	int value_;
};

class json_array final : public json_value {
	using vector_type = std::pmr::vector<managed_ptr<const json_value>>;
	
public:
	using allocator_type = std::pmr::polymorphic_allocator<>;
	
	json_array() noexcept = default;
	
	explicit json_array(allocator_type alloc) noexcept : values_{alloc} {}
	
	explicit json_array(vector_type&& values, allocator_type alloc = {}) noexcept
	    : values_{std::move(values), alloc} {}
	
	allocator_type get_allocator() const noexcept {
	    return values_.get_allocator();
	}
	
	json_array(json_array&& other) noexcept = default;
	
	json_array(json_array&& other, allocator_type alloc) noexcept
	    : values_{std::move(other.values_), alloc} {}
	
	json_array& operator=(json_array&& other) noexcept = default;
	
	// ...
	
	// json_value
	json_value_type type() const noexcept override {
	    return json_value_type::array;
	}
	
private:
	vector_type values_;
};
```

我们提供了一个将 *json* 解析为 `managed_ptr` 的函数：

```C++
void parse_json(managed_ptr<json_value>& json, std::string_view s);
```

通过使用 `managed_ptr`，我们支持为解析后的 *json* 使用自定义分配器。

为了了解这样做的好处，让我们编写一个小基准测试。

我们将解析 *json*，然后递归地对数组中的所有数字求和。我们测试了一个将 *json* 解析为[扩展堆栈空间](https://buildingblock.ai/extended-stack)的版本和一个使用带有 `std::unique_ptr` 的标准分配的版本。

```C++
const std::string_view s1 = R"(
[
  1, 7, -2,
  [10, -12, 100, 15, -77],
  [[10], [9, 2], [[37]]],
  [[[[]]], [8, -9, [-8, [-1, [10, [5], 8]]]]],
  [1, [2, [3, [4, [5, [6, [7, [8, [9, [10, 11, 12]]]]]]]]]]
]
)";

static void BM_parse_sum_json_managed_stackext(benchmark::State& state) {
	for (auto _ : state) {
	    stackext_resource resource;
	    managed_ptr<json_value> json{&resource};
	    parse_json(json, s1);
	    auto sum = sum1(json.get());
	    benchmark::DoNotOptimize(sum);
	}
}
BENCHMARK(BM_parse_sum_json_managed_stackext);

static void BM_parse_sum_json_managed_stackext_wink(benchmark::State& state) {
	for (auto _ : state) {
	    stackext_resource resource;
	    std::pmr::polymorphic_allocator alloc{&resource};
	    auto json = alloc.new_object<managed_ptr<json_value>>();
	      // note: we wink out here to avoid the unnecessary call to
	      // the destructor.
	    parse_json(*json, s1);
	    auto sum = sum1(json->get());
	    benchmark::DoNotOptimize(sum);
	}
}
BENCHMARK(BM_parse_sum_json_managed_stackext_wink);
```

下表展示了结果：

|类别|时间（纳秒）|
|---|---|
|全局分配器|2917|
|stackext (w/o wink)|2690|
|stackext (w/ wink)|2468|