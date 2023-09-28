*本文档译自 blog.molecular-matters.com 的 "Stateless, layered, multi-threaded rendering" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
在本系列的前一部分中，我讨论了如何设计无状态渲染API，但遗漏了一些细节。这一次，我将讨论这些细节以及评论中出现的一些问题，甚至还将展示当前实现的部分内容。


## 命令块 - *Command bucket*
----
在本系列的第一部分中，我介绍了这样一个想法，即并非所有层都需要存储相同大小的键，例如，用于将对象渲染到阴影贴图中的层可能只需要16位键，而 *G-Buffer* 层需要64位键。

按 *Molecule* 的术语来，负责存储绘制调用（及其相关的键和数据）的东西被称为 *CommandBucket*，它是一个轻量级的类模板，将键类型作为模板参数。

```C++
template <typename T>
class CommandBucket
{
	typedef T Key;
	...
};
```

渲染器会创建渲染场景所需的所有 *CommandBucket*，将它们存储为成员，在 *Render()* 方法中使用绘制调用填充它们。或者，你可以让 *CommandBucket* 的创建和销毁完全由数据驱动并且可配置。

但是 *CommandBucket* 具体需要存储什么，以及如何存储它？

+ *N* 个键，对应 *N* 个绘制调用。
+ *N* 个绘制调用的数据。

注意，每个绘制调用数据的数量和类型很大程度上取决于绘制调用的类型。因此，绘制调用所需的所有数据都存储在单独的内存区域中，而 *CommandBucket* 仅存储指向该区域的指针：

```C++
template <typename T>
class CommandBucket
{
	typedef T Key;
	...
	
private:
	Key* m_keys;
	void** m_data;
};
```

注意，键及其数据存储在两个单独的数组中。原因是某些排序算法在排序操作期间不会直接对元素进行交换，而是用已经排好序的元素的索引填充一个单独的数组，因此必须让算法接触更少的数据。

当创建 *CommandBucket* 时，它负责分配键数组和数据指针数组的内存，并存储所有的渲染目标（*Render Target*）以及视图矩阵、投影矩阵和视口，当提交命令时使用。这背后的基本原理是，你很可能使用相同的摄像头和视口渲染到某个层，所以没必要在每次绘制调用中都指定该信息。

此外，这意味着每个 *CommandBucket* 只能保存一定数量的绘制调用，具体数量是在创建 *bucket* 时指定的，如下面的示例所示：

```C++
CommandBucket<uint64_t> gBufferBucket(2048, rt1, rt2, rt3, rt4, dst, 
									  viewMatrix, projMatrix);

```

当然，视图和投影矩阵一般是由相机对象或某种相机系统提供的，但这是一个完全不同的主题。


## 命令 - *Commands*
----
现在我们有了存储绘制调用的 *bucket*，那么如何向 *bucket* 中添加命令呢？命令到底具体是什么？

命令是渲染后端能够理解的自包含信息，并存储在 *bucket* 中。命令可以识别任何类型的绘制调用（非索引渲染、索引渲染、实例化渲染）或任何其他操作，例如将数据复制到常量缓冲区。

每个命令都是一个简单的 *POD*，它包含后端执行此个命令所需的相关数据。下面三个结构体都是简单命令的示例：

```C++
namespace commands
{
	struct Draw
	{
		uint32_t vertexCount;
	    uint32_t startVertex;
		 
	    VertexLayoutHandle vertexLayoutHandle;
	    VertexBufferHandle vertexBuffer;
	    IndexBufferHandle indexBuffer;
	};
	 
	struct DrawIndexed
	{
	    uint32_t indexCount;
	    uint32_t startIndex;
	    uint32_t baseVertex;
		 
	    VertexLayoutHandle vertexLayoutHandle;
	    VertexBufferHandle vertexBuffer;
	    IndexBufferHandle indexBuffer;
	};
	 
	struct CopyConstantBufferData
	{
		ConstantBufferHandle constantBuffer;
	    void* data;
	    uint32_t size;
	};
}
```

因为每个命令都是一个单独的 *POD*，所以将它们放入 *bucket* 就变得很简单。我们可以添加一个方法，该方法接受一个键，为存储命令腾出空间，在内部数组中存储指向数据的指针，并将 *POD* 实例交给用户：

```C++
template <typename U>
U* CommandBucket::AddCommand(Key key)
{
	U* data = AllocateCommand<U>();
	
	// store key and pointer to the data
	AddKey(key);
	AddData(data);
	
	return data;
}
```

这仍然非常简单，但是还有一些地方我们还没有讨论，比如在访问数组时添加同步，以及如何为命令分配内存。我们稍后将重新讨论这个话题，因为现在有更重要的事情需要先讨论。

现在，假设我们有一个 *bucket*，我们以以下方式填充它：

```C++
for (size_t i = 0; i < meshComponents.size(); ++i)
{
	MeshComponent* mesh = &meshComponents[i];
	
	commands::DrawIndexed* dc = gBuffer
		.AddCommand<commands::DrawIndexed>(GenerateKey(mesh->aabb, mesh->material));
	
	dc->vertexLayoutHandle = mesh->vertexLayout;
	dc->vertexBuffer = mesh->vertexBuffer;
	dc->indexBuffer = mesh->indexBuffer;
	dc->indexCount = mesh->indexCount;
	dc->startIndex = 0u;
	dc->baseVertex = 0u;
}
```

与上一篇文章中介绍的两个备选方案相比，请注意，在创建绘制调用并将其插入 *bucket* 之后，不需要调用另一个方法。在调用 `AddCommand()` 之后，该命令完全属于你，你只需填充它的所有成员。这就足够了。所有写入操作都将直接写入一个连续的内存块，而不需要任何额外的复制操作，这后面会详细介绍。

负责绘制调用的 *bucket* 现在为每个网格组件保存一个索引绘制调用。在填充了所有命令之后，我们可以根据它们的键对它们进行排序：

```C++
gBufferBucket.Sort();
lightingBucket.Sort();
deferredBucket.Sort();
postProcessingBucket.Sort();
hudBucket.Sort();
```

为了对 *bucket* 中的命令排序，我们可以使用任何我们想使用的排序算法。这里需要注意的一件事是，每个 `CommandBucket::Sort()` 可以在不同的线程上运行，对所有 *bucket* 进行并行排序。

在所有的 *bucket* 都被排序之后，我们可以将它们提交到渲染后端：

```C++
gBufferBucket.Submit();
lightingBucket.Submit();
deferredBucket.Submit();
postProcessingBucket.Submit();
hudBucket.Submit();
```

提交过程必须从一个线程完成，因为它不断地与图形API（*D3D*, *OpenGL*）进行交互，向GPU提交工作。但是，这个线程是主线程还是专用的渲染线程并不重要。


## 提交命令 - *Submission process*
----
但是我们如何向图形API提交命令呢？我们所拥有的只是一个键和一个指向相关数据的指针。这显然是不够的，因此我们需要为每个命令添加某种额外的标识符。

实现它的其中一种方法是为每个命令添加一个标识符（例如枚举值），将其与键和数据一起存储，然后实现类似于下面代码段的 `Submit()` ：

```C++
void Submit(void)
{
	SetViewMatrix();
	SetProjectionMatrix();
	SetRenderTargets();
	
	for (unsigned int i = 0; i < commandCount; ++i)
	{
	    Key key = m_keys[i];
	    void* data = m_data[i];
	    uint16_t id = m_ids[i];
		
	    // decode the key, and set shaders, textures, constants, etc. if the material has changed.
	    DecodeKey();
		
	    switch (id)
	    {
		case command::Draw::ID:
	    // extract data for a Draw command, and call the backend
	    break;
			
		case command::DrawIndexed::ID:
		// extract data for a DrawIndexed command, and call the backend
	    break;
			
		...;
		}
	}
}
```

这是一个可能的解决方案，但我不建议这样做。为什么？

首先，我们做了两次相同的事情。在将命令存储到 *bucket* 中时，我们首先识别它（例如，将 *U::ID* 存储到我们的 *m_ids* 数组中），然后又在巨大的 *switch* 语句中再次识别它。

其次，硬编码的 *switch* 语句使得添加新命令变得困难而乏味，如果我们不能访问源代码，就不可能添加用户定义的命令。

有一个更好更简单的解决方案：函数指针。


## 后端分配器 - *Backend dispatch*
----
我们可以直接存储一个指针，指向一个知道如何处理某个命令的函数，并将其转发到渲染后端，而不是将命令的 *ID* 存储在 *bucket* 中。这就是所谓的后端分配器。

后端分配器是一个仅由简单转发功能组成的名称空间：

```C++
namespace backendDispatch
{
	namespace cmd = commands;
	void Draw(const void* data)
	{
	    const cmd::Draw* realData = union_cast<const cmd::Draw*>(data);
	    backend::Draw(realData->vertexCount, realData->startVertex);
	}
	
	void DrawIndexed(const void* data)
	{
		const cmd::DrawIndexed* realData = union_cast<const cmd::DrawIndexed*>(data);
		backend::DrawIndexed(realData->indexCount, 
		realData->startIndex, realData->baseVertex);
	}
	
	void CopyConstantBufferData(const void* data)
	{
	    const cmd::CopyConstantBufferData* realData = 
	    union_cast<const cmd::CopyConstantBufferData*>(data);
	    
	    backend::CopyConstantBufferData(
	    realData->constantBuffer, realData->data, realData->size);
	}
}
```

后端分配器中的每个函数都具有相同的签名，因此我们可以使用 *typedef* 存储指向某个函数的指针：

```C++
typedef void (*BackendDispatchFunction)(const void*);
```

后端命名空间中包含的函数仍然是唯一直接与图形API产生交互的函数，例如通过API使用 *D3D Device*。

因此，让我们回过头来重新审视 *CommandBucket* 及其 `AddCommand()` 方法。除了命令之外，我们现在还需要存储一个指向后端函数的指针。实际上，除了上述内容之外，我们还需要存储另外两个尚未讨论的内容：

第一个是指向其他命令的指针，该命令需要与此命令一起提交，并且具有相同的键。如果我们存储一个指向另一个命令的指针，我们就构建了一个侵入式链表，允许我们处理总是需要以特定顺序提交的绘制调用和命令，无论分配给它们的键是什么。这个问题在评论中出现了不止一次，当我们提交绘制调用时需要，例如，首先需要将数据复制到常量缓冲区，然后提交绘制调用。侵入式链表允许我们链接任意数量的命令。

第二，某些命令需要辅助内存来储存稍后向API提交绘制调用时所需的中间数据。最好的例子就是用几个字节的数据更新一个常量缓冲区，比如光源信息。这些字节隐藏在辅助内存中，并在提交命令时从那里复制到常量缓冲区中。


## 命令包 - *Command packets*
----
因为我们不再只是在 *bucket* 中存储单个命令，所以我们引入了命令包的概念。*bucket* 现在存储的是命令包，每个包包含以下数据：

+ `Void*`：指向下一个命令包的指针（如果有的话）。
+ `BackendDispatchFunction`：指向负责将调用分派到后端的函数的指针。
+ `Cmd`：实际命令（所需数据）。
+ `Char[]`：命令所需的辅助内存（可选）。

每当用户想要向 *bucket* 中添加 *Cmd* 类型的命令时，我们也需要为其他东西腾出空间。为此，我只需使用适当的 `sizeof()` 操作符分配足够大的原始内存以容纳所有数据，并将各个部分强制转换为所需的类型。为了使其工作，使用一些静态断言确保所有命令都是 *POD* 结构体。

最后，一个用于辅助的名称空间负责执行所有偏移量计算和强制转换：

```C++
typedef void* CommandPacket;
 
namespace commandPacket
{
	static const size_t OFFSET_NEXT_COMMAND_PACKET = 0u;
	static const size_t OFFSET_BACKEND_DISPATCH_FUNCTION = OFFSET_NEXT_COMMAND_PACKET + sizeof(CommandPacket);
	static const size_t OFFSET_COMMAND = OFFSET_BACKEND_DISPATCH_FUNCTION + sizeof(BackendDispatchFunction);
	
	template <typename Cmd>
	CommandPacket Create(size_t auxMemorySize)
	{
		//Ultimately, we will use our own allocator instead of operator new.
		return ::operator new(GetSize<Cmd>(auxMemorySize));
	}
	
	template <typename Cmd>
	size_t GetSize(size_t auxMemorySize)
	{
		return OFFSET_COMMAND + sizeof(Cmd) + auxMemorySize;
	};
	
	CommandPacket* GetNextCommandPacket(CommandPacket packet)
	{
	    return union_cast<CommandPacket*>(reinterpret_cast<char*>(packet) + OFFSET_NEXT_COMMAND_PACKET);
	}
	
	template <typename Cmd>
	CommandPacket* GetNextCommandPacket(Cmd* command)
	{
	    return union_cast<CommandPacket*>(reinterpret_cast<char*>(command) - OFFSET_COMMAND + OFFSET_NEXT_COMMAND_PACKET);
	}
	
	BackendDispatchFunction* GetBackendDispatchFunction(CommandPacket packet)
	{
	    return union_cast<BackendDispatchFunction*>(reinterpret_cast<char*>(packet) + OFFSET_BACKEND_DISPATCH_FUNCTION);
	}
	
	template <typename Cmd>
	Cmd* GetCommand(CommandPacket packet)
	{
	    return union_cast<Cmd*>(reinterpret_cast<char*>(packet) + OFFSET_COMMAND);
	}
	
	template <typename Cmd>
	char* GetAuxiliaryMemory(Cmd* command)
	{
	    return reinterpret_cast<char*>(command) + sizeof(Cmd);
	}
	
	void StoreNextCommandPacket(CommandPacket packet, CommandPacket nextPacket)
	{
	    *commandPacket::GetNextCommandPacket(packet) = nextPacket;
	}
	
	template <typename Cmd>
	void StoreNextCommandPacket(Cmd* command, CommandPacket nextPacket)
	{
	    *commandPacket::GetNextCommandPacket<Cmd>(command) = nextPacket;
	}
	
	void StoreBackendDispatchFunction(CommandPacket packet, BackendDispatchFunction dispatchFunction)
	{
	    *commandPacket::GetBackendDispatchFunction(packet) = dispatchFunction;
	}
	
	const CommandPacket LoadNextCommandPacket(const CommandPacket packet)
	{
	    return *GetNextCommandPacket(packet);
	}
	
	const BackendDispatchFunction LoadBackendDispatchFunction(const CommandPacket packet)
	{
		return *GetBackendDispatchFunction(packet);
	}
	
	const void* LoadCommand(const CommandPacket packet)
	{
	    return reinterpret_cast<char*>(packet) + OFFSET_COMMAND;
	}
};
```

注意，目前 `Create()` 使用全局的 `operator new` 来分配原始内存。在实际实现中，我们将使用自己的线性分配器，以确保所有命令都连续地存储在内存中，当我们需要遍历 `Submit()` 方法中的命令时，这对缓存更加友好。


## 再探命令包 - *Revisiting the command bucket*
----
准备好命令包后，向 *bucket* 中添加命令的实际代码如下所示：

```C++
template <typename U>
U* AddCommand(Key key, size_t auxMemorySize)
{
	CommandPacket packet = commandPacket::Create<U>(auxMemorySize);
	
	// store key and pointer to the data
	{
	    // TODO: add some kind of lock or atomic operation here
	    const unsigned int current = m_current++;
	    m_keys[current] = key;
	    m_packets[current] = packet;
	}
	
	commandPacket::StoreNextCommandPacket(packet, nullptr);
	commandPacket::StoreBackendDispatchFunction(packet, U::DISPATCH_FUNCTION);
	
	return commandPacket::GetCommand<U>(packet);
}
```

处理完上面标记的 *TODO* 之后，我们还可以开始从任意数量的线程添加命令。作为第一个实现，我们可以简单地添加一个临界区来使代码工作，但显然有更好的解决方案，这是我想在本系列的下一篇文章中写的内容。

当然，现在每个命令还需要存储一个指向后端分配器的函数指针，*Draw* 命令就是一个例子：

```C++
struct Draw
{
	static const BackendDispatchFunction DISPATCH_FUNCTION;
	
	uint32_t vertexCount;
	uint32_t startVertex;
	
	VertexLayoutHandle vertexLayoutHandle;
	VertexBufferHandle vertexBuffer;
	IndexBufferHandle indexBuffer;
};
static_assert(std::is_pod<Draw>::value == true, "Draw must be a POD.");
 
const BackendDispatchFunction Draw::DISPATCH_FUNCTION = &backendDispatch::Draw;
```


## 自定义命令 - *Custom commands*
----
如前所述，以这种方式使用函数指针还允许我们支持用户定义的命令。例如，你可以通过定义 *POD* 来创建自己的命令，为它实现一个完全自定义的调度函数，并将该命令添加到任何 *bucket*，甚至将其链接到其他命令。


## 链接命令 - *Chaining commands*
----
现在我们的命令包还存储了指向下一个命令包的指针，我们可以将命令附加到其他命令：

```C++
template <typename U, typename V>
U* AppendCommand(V* command, size_t auxMemorySize)
{
	CommandPacket packet = commandPacket::Create<U>(auxMemorySize);
	
	// append this command to the given one
	commandPacket::StoreNextCommandPacket<V>(command, packet);
	
	commandPacket::StoreNextCommandPacket(packet, nullptr);
	commandPacket::StoreBackendDispatchFunction(packet, U::DISPATCH_FUNCTION);
	
	return commandPacket::GetCommand<U>(packet);
}
```

请注意，在这种情况下，我们不需要在数组中存储新的键/值对，因为附加到另一个命令的每个命令都需要具有相同的键。

下面的示例展示了如何使用新的 *bucket* API将命令链接在一起：

```C++
for (unsigned int i = 0; i < directionalLights.size(); ++i)
{
	PerDirectionalLightConstants constants =
	{ directionalLights[i].diffuse, directionalLights[i].specular };
	
	commands::CopyConstantBufferData* copyOperation = lightingBucket
		.AddCommand<commands::CopyConstantBufferData>(someKey, sizeof(PerDirectionalLightConstants));
	
	copyOperation->data = commandPacket::GetAuxiliaryMemory(copyOperation);
	copyOperation->constantBuffer = directionalLightsCB;
	memcpy(copyOperation->data, &constants, sizeof(PerDirectionalLightConstants));
	copyOperation->size = sizeof(PerDirectionalLightConstants);
	
	commands::Draw* dc = lightingBucket
		.AppendCommand<commands::Draw>(copyOperation, 0u);
	
	dc->vertexCount = 3u;
	dc->startVertex = 0u;
}
```


## 再探提交命令 - *Revisiting the submission process*
----
当然，有了命令包之后，还需要调整 `Submit()` 方法。通过使用后端分配器，我们可以摆脱 `switch` 语句，并且可以使用一个简单的循环来遍历命令链表：

```C++
void Submit(void)
{
	// ... same as before
	for (unsigned int i = 0; i < m_current; ++i)
	{
	    // ... same as before
	    CommandPacket packet = m_packets[i];
	    do
	    {
		    SubmitPacket(packet);
		    packet = commandPacket::LoadNextCommandPacket(packet);
	    } while (packet != nullptr);
	}
}
 
void SubmitPacket(const CommandPacket packet)
{
	const BackendDispatchFunction function = commandPacket::LoadBackendDispatchFunction(packet);
	
	const void* command = commandPacket::LoadCommand(packet);
	function(command);
}
```


## 总结 - *Recap*
----
这篇文章已经很长了，比我预期的要长得多。但是，让我们回顾一下这篇文章介绍的概念：

+ 命令：传递给后端分配器的自包含数据。每个命令都类似于一个简单的操作，如索引绘制、将数据复制到常量缓冲区等。每个命令都被实现为 *POD* 结构体。
+ 后端分配器：简单的转发功能，从命令中提取数据，并将它们转发到图形API。其中的每个函数处理不同的命令。
+ 命令包：一个命令包存储一个命令，以及其他数据，如后端函数的指针，命令可能需要的任何辅助内存，以及用于链接其他命令的侵入式链表。
+ 命令链接：需要按一定顺序提交的命令可以链接在一起。
+ *bucket*：用于存放命令包和任意大小的键。
+ 多线程渲染：命令可以从多个线程并行地添加到 *bucket* 中。唯一的两个同步点是内存分配和将键-值对存储到命令包数组中。
+ 多线程排序：每个 *bucket* 可以独立地并行排序。

尽管这已经是本系列的第3部分，但仍有一些事情我们还没有详细讨论：

+ 内存管理：如何分配内存来存储键和指向命令包的指针？在多个线程向同一个 *bucket* 中添加命令的情况下，我们如何为单个命令包分配内存？如何在整个命令包的存储和提交过程中保证良好的缓存利用率？我们可以使用一个连续的内存块吗？
+ 键的生成：键包含哪些信息？我们如何有效地构建它？

就目前而言，我们的渲染过程做到了无状态、分层/分块，但它的多线程渲染功能仍然可以大大提高。*Until next time*！
