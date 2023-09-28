*本文档译自 blog.molecular-matters.com 的 "Task Scheduler" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
在本系列的这一部分中，我们将详细讨论 *Molecule* 的任务模型，并查看底层 C++ 代码和需要注意的一些细微之处，以及一些独特的优化机会。

在我们开始充实关于任务和底层任务模型的细节之前，让我们先确定常见的游戏引擎任务是什么样子的：

+ 简单的任务：他们需要一堆不可变的数据，并简单地执行他们的任务。
+ 流任务：它们需要不可变的数据，以及具有一对一映射的输入和输出流。
+ 过滤/发散任务：同样，它们需要不可变的数据，但输入/输出比不是1:1，而是N:1或1:N。例如，考虑一个任务，该任务计算例如每32个元素的平均值，并为它们输出一个值——每32个输入元素就有一个输出元素。


## 任务模型 - *The task model*
---
用 *Molecule* 的概念，每个任务由以下部分组成：

+ 一个可以对给定数据起作用的”核心“。
+ 不可变数据，称为 *KernelData*。
+ （可选）流数据，包括 *InputStreams* 和 *OutputStreams*。
+ （可选）其他需要的数据，例如过滤/发散任务，或未来其他尚未确定的任务。

考虑到上面的内容，一个可能的经典实现如下所示：

```C++
// simple task, acts as a base class
class Task
{
public:
	explicit Task(const KernelData& kernelData);
	
	/// Executes the task, works on kernel data
	virtual void Execute(void);
	
private:
	KernelData m_kernelData;
};
 
// streaming task
class StreamingTask : public Task
{
public:
	StreamingTask(const KernelData& kernelData, const InputStream& inputStream, const OutputStream& outputStream);
	
	/// Executes the task, works on kernel and streaming data
	virtual void Execute(void);
	
private:
	KernelData m_kernelData;
	InputStream m_input;
	OutputStream m_output;
};
```

每个任务可以通过 `virtual Execute()` 函数做任何他想做的事情，不同的任务使用不同的数据。一切都很好，但事实并非如此：

+ 迟早需要从某个地方分配任务。拥有不同大小的任务可能导致内存碎片。
+ 虚函数不能很好地用于 *PS3* 上的 *SPU*。

我们可以做得更好，下面是在 *Molecule* 中使用的建议实现：

```C++
typedef core::traits::Function<void (const TaskData&)>::Pointer Kernel;
 
struct TaskData
{
	void* m_kernelData;
	
	union
	{
	    // for streaming tasks
	    struct StreamingData
	    {
		    uint32_t m_elementCount;
		    void* m_inputStreams[4];
		    void* m_outputStreams[4];
	    } m_streamingData;
	};
};
```

正如你所看到的，`Kernel` 只不过是一个函数指针，指向一个接受 `TaskData` 参数的函数。 `TaskData` 本身由 `m_kernelData` 数据和不同任务所需的可选数据组成。

注意，`m_StreamingData` 被放在一个联合体中，这意味着每当添加可选数据时，我可以简单地将其放入联合体中的另一个结构体中，而无需改变 `TaskData` 的大小（只要新结构体的大小不大于 `m_StreamingData`）。

这背后的基本原理是，一个任务可以是简单任务、流任务、过滤/发散任务，但不能同时是几个任务。用户确切地知道他需要访问哪些成员，因为他是向调度程序提交任务的人。这使我们能够使 *Task* 具有固定的大小，这对以后的优化很有用：

```C++
struct Task
{
	Kernel m_kernel;
	TaskData m_taskData;
	// more members here, explained later
};
```

不幸的是，一些程序员一开始就把所有东西都塞进继承树，就会忘记还有联合体之类的东西。


## 任务调度 - *Scheduling tasks*
---
在解释了底层任务模型之后，工作实际上是如何交给调度器的呢？有不同的重载用于将任务交给调度程序，每个重载负责准备由不同的工作线程处理的任务：

```C++
namespace scheduler
{
	TaskId AddTask(const KernelData& kernelData, Kernel kernel);
	TaskId AddStreamingTask(const KernelData& kernelData, const InputStream& is0, const OutputStream& os0, uint32_t elementCount, Kernel kernel);
	// more overloads taking 2-4 streams
};
```

让我们仔细看看 `AddTask()` 函数：

```C++
TaskId AddTask(const KernelData& kernelData, Kernel kernel)
{
	Task* task = ObtainTask();
	task->m_kernel = kernel;
	task->m_taskData.m_kernelData = kernelData.m_data;
	
	QueueTask(task);
	
	return GetTaskId(task);
}
```

该函数只是获取一个新的 `task`，设置适当的成员，将其添加到任务的全局队列中，并返回一个 `TaskId`，调用者使用它来标识这个特定的任务。我们不想让用户直接访问内部 `task` 成员，因此我们需要某种标识，可以用来将 `task` 映射到 `TaskId`，反之亦然。注意，`std::map` 并不是我们想要的！


## 分配并标识任务 - *Allocating and identifying tasks*
---
在我们讨论如何为 `Task` 生成 `TaskId` 之前，让我们仔细看看 `Task` 是如何实际分配的。还记得我们想确保 `Task` 具有一定的大小吗？这样做的原因是，我们可以使用自由列表来分配和释放任务，这给了我们 *O(1)* 的效率，并且没有碎片。*Molecule* 中的[自由列表](http://en.wikipedia.org/wiki/Free_list)实现获取一个供自由列表使用的内存范围，并将指向下一个元素的指针直接存储在每个条目中——这意味着每个条目都有点像“内存中的并集”。

条目可以处于已分配状态，这样就意味着内存中的数据已经可以直接被 *interpreted* 为 `Task` 。或者条目可以处于释放状态，这意味着前几个字节被解释为指向下一个自由条目的指针。

实现如下：

```C++
class Freelist
{
  typedef Freelist Self;
 
public:
  Freelist(void* start, void* end, size_t elementSize);
 
  void* Obtain(void);
 
  void Return(void* ptr);
 
private:
  Self* m_next;
};
 
Freelist::Freelist(void* start, void* end, size_t elementSize)
  : m_next(nullptr)
{
	union
	{
	    void* as_void;
	    char* as_char;
	    Self* as_self;
	};
	 
	as_void = start;
	m_next = as_self;
	
	const size_t numElements = (static_cast<char*>(end) - as_char) / elementSize;
	
	as_char += elementSize;
	
	// initialize the free list - make every m_next at each element point to the next element in the list
	Self* runner = m_next;
	for (size_t i=1; i<numElements; ++i)
	{
	    runner->m_next = as_self;
	    runner = runner->m_next;
		 
	    as_char += elementSize;
	}
	
	runner->m_next = nullptr;
}
 
void* Freelist::Obtain(void)
{
	// obtain one element from the head of the free list
	Self* head = m_next;
	m_next = head->m_next;
	return head;
}
 
void Freelist::Return(void* ptr)
{
	// put the returned element at the head of the free list
	Self* head = static_cast<Self*>(ptr);
	head->m_next = m_next;
	m_next = head;
}
```

一个包含1024个任务的空闲列表可以简单地如下所示构造：

```C++
const unsigned int TASK_POOL_SIZE = sizeof(Task)*1024;
char taskPoolMemory[TASK_POOL_SIZE];
Freelist taskPool(taskPoolMemory, taskPoolMemory + TASK_POOL_SIZE, sizeof(Task));
```

仍然存在的问题是，我们如何通过 `TaskId` 来识别任务？因为每个 `Task` 都是从自由列表中分配的，而自由列表是从一个连续的内存区域分配的，所以我们可以简单地将每个 `Task` 转换为内存范围开始的偏移量（例如上面的 `taskPoolMemory`），就像下面的函数一样：

```C++
inline TaskId GetTaskId(Task* task)
{
	return task - reinterpret_cast<Task*>(taskPoolMemory);
}
 
inline Task* GetTask(TaskId taskId)
{
	return reinterpret_cast<Task*>(taskPoolMemory) + taskId;
}
```


## 用户代码 - *Client code*
---
有了上面的函数，我们可以像下面的例子一样向调度程序添加任务：

```C++
// setup data for streaming skinning task
scheduler::KernelData transformationData(transformations, NUM_JOINTS * sizeof(matrix_4x4));
scheduler::InputStream weightedVertexData(weightedVertices, sizeof(WeightedVertex));
scheduler::OutputStream transformedVertexData(transformedVertices, sizeof(TransformedVertex));
 
// schedule skinning task
scheduler::TaskId skinningTask = scheduler::AddStreamingTask(transformationData, weightedVertexData, transformedVertexData, NUM_VERTICES, &SkinningKernel);
scheduler::Wait(skinningTask);
 
// setup data and schedule checksum task
scheduler::KernelData checksumData(&data, sizeof(ChecksumKernelData));
scheduler::TaskId checksumTask = scheduler::AddTask(checksumData, &CheckSumKernel);
scheduler::Wait(checksumTask);
```

请注意，`KernelData`、`InputStream` 和 `OutputStream` 不仅将数据的地址作为参数，而且还需要额外的参数，如数据的大小和流的步幅。这是必要的，以便调度器知道 *PS3* 上的 *PPU* 和 *SPU* 之间需要 *DMAed* 多少数据。

每个 `scheduler::Add*()` 函数返回的 `TaskId` 可用于通过调用 `scheduler:: wait()` 等待各自的任务完成。


## 任务同步 - *Synchronizing with tasks*
---

