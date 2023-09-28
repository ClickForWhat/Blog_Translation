*本文档译自 blog.molecular-matters.com 的 "Job System 2.0" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
继续上节课的内容，今天我们将讨论如何为我们的作业系统构建高级算法，例如 `parallel_for`。


## 这个系列的其他文章 - *Other posts in the series*
---
[Part 1](https://blog.molecular-matters.com/2015/08/24/job-system-2-0-lock-free-work-stealing-part-1-basics/)：解释了新的作业系统的基本设计，和对“作业窃取”的简要说明。
[Part 2](https://blog.molecular-matters.com/2015/09/08/job-system-2-0-lock-free-work-stealing-part-2-a-specialized-allocator/)：对线程本地分配的机制细节作深入讨论。
[Part 3](https://blog.molecular-matters.com/2015/09/25/job-system-2-0-lock-free-work-stealing-part-3-going-lock-free/)：讨论作业窃取队列的无锁实现。


## 基本思路 - *The basic idea*
---
现在，我们已实现了把任务推送到作业系统中执行，并且系统能够在可用的核心上平衡任务负载。不过，到目前为止，系统还没有意识到有些作业可能更小，有些作业可能更大，而且它们都可能需要不同的时间来完成。

在游戏编程领域里，一个很普遍的场景是对固定数量的元素执行一次任务，比如对一个数组的所有元素应用同一个变换。“变换”这个操作，在不同的上下文里既可以是指剔除 10,000 个包围盒，也可以是更新 1,000 个角色的骨骼层级，又或者是对 100,000 个粒子进行模拟。无论任务是什么，都可能存在一些我们可以（也应该）利用的数据并行性。

这里就是诸如 `parallel_for` 之类的高级算法发挥作用的地方。

从概念上说，`parallel_for` 类作业背后的想法相当简单。给定元素的范围，任务应被分割为多个相等的部分，然后在系统中创建对应的新任务，并分布在不同的任务负载中执行。对于创建的每个新范围，应该调用用户提供的函数，该函数负责在给定的元素范围内执行实际的工作。

下面是一个例子，让我们考虑如何更新粒子。以最简单的形式为例，我们有一个自由函数，只接受数据数组，和表明数组元素数量的整数：

```C++
void UpdateParticles(Particle* particles, unsigned int count);
```

假设我们有 100,000 个粒子需要更新，我们可以通过生成一个作业来在同一核心上更新所有粒子，只需执行以下操作：

```C++
UpdateParticles(particles, 100000);
```

如果我们想要将更新分成 4 个大小相等的部分，我们该怎么办？这样的函数也很简单：

```C++
UpdateParticles(particles, 25000);
UpdateParticles(particles + 25000, 25000);
UpdateParticles(particles + 50000, 25000);
UpdateParticles(particles + 75000, 25000);
```

将每个调用放入单独的作业中，我们可以自动把工作负载分配到 4 个核心（如果可用）。注意，得益于我们定义 `UpdateParticles()` 函数的方式很简洁直接，工作划分也非常简单。

当然，手工完成所有这些工作既繁琐又容易出错，应该由 `parallel_for` 来负责。


## 实现 - *Implementation*
---
在最基本的形式中，`parallel_for` 需要接受以下参数：

+ 一组数据，包括指向数据的指针和存储在数组中的元素数。
+ 用于执行新生成的作业的函数。

闲话少说，大致实现如下：

```C++
Job* parallel_for(Particles* data, unsigned int count, void (*function)(Particles*, unsigned int))
{
	const parallel_for_job_data jobData = { data, count, function };
	
	Job* job = jobSystem::CreateJob(&jobs::parallel_for_job, jobData);
	return job;
}
 
struct parallel_for_job_data
{
	Particles* data;
	unsigned int count;
	void (*function)(Particles*, unsigned int);
};
```

该实现只是为 *job* 创建一个名为 *jobs::parallel_for_job* 的新作业，该作业负责递归地将范围划分为更小的范围。然后，返回的作业可以由用户运行和等待。

```C++
Job* job = parallel_for(g_particles, 100000, &UpdateParticles)
jobSystem::Run(job);
jobSystem::Wait(job);
```

*job* 本身的并行也只有少数几行：

```C++
void parallel_for_job(Job* job, const void* jobData)
{
  const parallel_for_job_data* data = static_cast<const parallel_for_job_data*>(jobData);
     
	if (data->count > 256)
	{
	    // split in two
	    const unsigned int leftCount = data->count / 2u;
	    const parallel_for_job_data leftData(data->data, leftCount, data->function);
	    Job* left = jobSystem::CreateJobAsChild(job, &jobs::parallel_for_job, leftData);
	    jobSystem::Run(left);
		
	    const unsigned int rightCount = data->count - leftCount;
	    const parallel_for_job_data rightData(data->data + leftCount, rightCount, data->function);
	    Job* right = jobSystem::CreateJobAsChild(job, &jobs::parallel_for_job, rightData);
	    jobSystem::Run(right);
	}
	else
	{
	    // execute the function on the range of data
	    (data->function)(data->data, data->count);
	}
}
```

可以看到，只要范围内剩下的元素超过 256 个，作业就会将给定范围分成两半。对于每个新创建的范围，将生成一个作业作为给定作业的子作业，使用分而治之的策略有效地将初始范围切割成更小的部分。注意，此时范围分割与这些（和其他）作业的执行将同时进行，这比让一个作业执行所有分割工作要高效一些。

当然，在这个实现中至少有两点需要修改：

+ `parallel_for` 算法目前只接受专门处理粒子的函数。这是为了使代码更具可读性，但需要修复。
+ 只要包含超过 256 个元素，范围就会被分割。仅根据元素的数量来划分策略可能不是最好的选择，我们当然希望至少能够在每个作业的基础上配置阈值。

不出所料，*C++* 模板提供了一种以高效和类型安全的方式处理这两个问题的好方法。


## 更加通用的实现 - *A more generic implementation*
---
通过为 `parallel_for` 函数和作业本身引入模板参数，可以实现接受任何类型的范围。此外，作业数据的并行还需要能够保存任意类型的指针。

类似地，拆分策略可以通过用户提供的附加模板参数提供给 `parallel_for` 算法。这允许用户根据要执行的作业选择不同的策略。

加上上述两个参数后，代码变成如下所示：

```C++
template <typename T, typename S>
Job* parallel_for(T* data, unsigned int count, void (*function)(T*, unsigned int), const S& splitter)
{
  typedef parallel_for_job_data<T, S> JobData;
  const JobData jobData(data, count, function, splitter);
 
  Job* job = jobSystem::CreateJob(&jobs::parallel_for_job<JobData>, jobData);
 
  return job;
}
 
template <typename T, typename S>
struct parallel_for_job_data
{
	typedef T DataType;
	typedef S SplitterType;
	
	parallel_for_job_data(DataType* data, unsigned int count, void (*function)(DataType*, unsigned int), const SplitterType& splitter)
    : data(data)
    , count(count)
    , function(function)
    , splitter(splitter)
	{}
	
	DataType* data;
	unsigned int count;
	void (*function)(DataType*, unsigned int);
	SplitterType splitter;
};
 
template <typename JobData>
void parallel_for_job(Job* job, const void* jobData)
{
	const JobData* data = static_cast<const JobData*>(jobData);
	const JobData::SplitterType& splitter = data->splitter;
     
	if (splitter.Split<JobData::DataType>(data->count))
	{
	    // split in two
	    const unsigned int leftCount = data->count / 2u;
	    const JobData leftData(data->data, leftCount, data->function, splitter);
	    Job* left = jobSystem::CreateJobAsChild(job, &jobs::parallel_for_job<JobData>, leftData);
	    jobSystem::Run(left);
		
	    const unsigned int rightCount = data->count - leftCount;
	    const JobData rightData(data->data + leftCount, rightCount, data->function, splitter);
	    Job* right = jobSystem::CreateJobAsChild(job, &jobs::parallel_for_job<JobData>, rightData);
	    jobSystem::Run(right);
	}
	else
	{
	    // execute the function on the range of data
	    (data->function)(data->data, data->count);
	}
}
```

现在 `parallel_for` 实现能够处理任意数据的范围，也接受分割策略作为附加参数。一个有效的策略实现只需要提供一个 `Split()` 函数来决定一个给定的范围是否应该进一步分割，下面的两个例子说明了这一点：

```C++
class CountSplitter
{
public:
	explicit CountSplitter(unsigned int count)
		: m_count(count)
	{}
	
	template <typename T>
	inline bool Split(unsigned int count) const
	{
	    return (count > m_count);
	}
	
private:
	unsigned int m_count;
};
 
class DataSizeSplitter
{
public:
	explicit DataSizeSplitter(unsigned int size)
		: m_size(size)
	{}
	
	template <typename T>
	inline bool Split(unsigned int count) const
	{
	    return (count * sizeof(T) > m_size);
	}
	
private:
	unsigned int m_size;
};
```

`CountSplitter` 仅根据元素的数量拆分范围，正如我们在最初的示例中看到的那样。另一个，`DataSizeSplitter` 是一种考虑工作集大小的策略。例如，在 *L1* 缓存大小为 32KB 的平台上，可以使用 `DataSizeSplitter` （32\*1024）来确保只有当范围的工作集不再适合 *L1* 缓存时才拆分范围。值得指出的一点是，这适用于所有类型的工作，无论他们是在处理粒子，动画还是剔除，策略都会自动执行。

对 `parallel_for` 的调用与以前相同，只是增加了一个拆分器参数：

```C++
Job* job = parallel_for(
	g_particles, 
	100000, 
	&UpdateParticles, 
	DataSizeSplitter(32 * 1024)); // Splitter
jobSystem::Run(job);
jobSystem::Wait(job);
```


## 未来的计划 - *Future work*
---
到目前为止，`parallel_for` 实现只能调用自由函数。升级它应该很容易，让它也接受成员函数，`lambda`，`std::function` 等。


## 展望 - *Outlook*
---
本系列的下一部分可能是最后一部分，将详细介绍如何处理作业之间的依赖关系。