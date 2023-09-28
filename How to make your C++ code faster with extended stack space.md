*本文档译自 buildingblock.ai 的 "extended stack space" 文章，作者 Ryan Burn*


## 概述 - *Overview*
---
假设我们有一个函数：

```C++
void f() {
	std::array<double, 100> my_array;
	// 对 my_array 做一些操作
}
```

`my_array` 会在哪里？答案是编译器将生成代码，在调用堆栈上为数组留出一个内存区域。这比从堆中提取内存的代码要高效得多：

```C++
void f_p() {
  std::unique_ptr<double[]> my_array_p{ new double[100] };
  // 对 my_array_p 做一些操作
}
```

为什么？

+ 堆栈的分配和释放可以直接通过增加和减少跟踪堆栈顶部的计数器来完成。相比之下，堆内存是通过跟踪空闲内存块的数据结构来管理的。这样的分配需要遍历数据结构以找到合适的块，然后更新数据结构以指示所使用的内存区域不再可用。
+ 从堆栈分配的连续内存区域具有局部性。通过定位相邻的区域，当我们访问或修改内存时，我们将得到更少的缓存行丢失。

但是，如果我们不知道在编译时数组需要多大，会发生什么呢？

```C++
void f(int n) {
  // 我们需要一个至少有 n 个元素的空间的数组
  // ...
}
```

我们还能从堆栈中分配内存吗？

非常特殊的函数 [`alloca`](https://man7.org/linux/man-pages/man3/alloca.3.html) 给了我们一种方法。我们可以这样写：

```C++
void f(int n) {
	auto my_array = static_cast<double*>(alloca(sizeof(double) * n));
	// ...
}
```

但我们需要小心。如果 `n` 太大，我们可能会超出可用的堆栈空间并使程序崩溃。

此外，`alloca` 使用不太方便。`alloca` 仅在当前函数内操作，因此无法使用辅助函数对其进行扩展。

幸运的是，有一种解决方案既提供了 `alloca` 的所有优点，又更加安全和可用。下面是它的工作原理。

首先，我们编写一个简单的类来管理字节堆栈：

```C++
template <size_t N>
class stack {
public:
	void* top() noexcept { return static_cast<void*>(data_.data() + size_); }
	
	void* allocate(size_t num_bytes) noexcept {
	    void* ptr = this->top();
	    size_ += num_bytes;
	    return ptr;
	}
	
	void deallocate(size_t num_bytes) noexcept {
	    size_ -= num_bytes;
	}
private:
	std::array<std::byte, N> data_{};
	size_t size_{ 0 };
};
```

然后使用这个类定义一个 `thread_local` 的全局变量，作为扩展堆栈空间：

```C++
constexpr size_t stack_extension_size_v = 1ull << 21;
thread_local constinit stack<stack_extension_size_v> stack_extension_v{};
```

现在我们定义一个自定义内存资源，它继承了 `std::pmr::memory_resource`。自定义资源从扩展堆栈空间进行分配，并在其析构函数中解分配所有内容。如果堆栈已满，则返回到所提供的上游资源。

```C++
class stackext_resource final : public std::pmr::memory_resource {
public:
	explicit stackext_resource(std::pmr::memory_resource* upstream =
                                 std::pmr::get_default_resource()) noexcept;
	
	~stackext_resource() noexcept override {
	    // 解分配从 stack_extension_v 处获取的内存
	}
	
private:
	std::pmr::monotonic_buffer_resource upstream_;
	size_t size_{ 0 };
	
	// std::pmr::memory_resource
	void* do_allocate(size_t num_bytes, size_t alignment) noexcept override {
	    // 尝试从 stack_extension_v 获取内存; 若未成功，从 upstream 处获取
	}
	
	void do_deallocate(void* /*指针*/, size_t /*字节大小*/,
                     size_t /*对齐值*/) noexcept override {
	    // do nothing
	}
	
	bool do_is_equal(const std::pmr::memory_resource& other) const noexcept override {
		return this == &other;
	}
};
```

使用自定义资源，我们可以轻松地配置标准容器，以便从扩展堆栈中进行分配。

```C++
void f() {
	stackext_resource resource;
	std::pmr::map<std::pmr::string, my_struct> m{ &resource };
	// 当我们使用 m 时, 它会从 extended stack 的空间获取内存
}
```

与 `alloca` 一样，`stackext_resource` 只需要简单的算术运算来管理分配，并提供出色的局部性。但与 `alloca` 不同的是，我们可以很容易地将它与标准数据结构一起使用，并且没有超出可用堆栈空间和使程序崩溃的危险。

*要获得完整的源代码，请查看项目 [bbai-mem](https://github.com/rnburn/bbai-mem)。*