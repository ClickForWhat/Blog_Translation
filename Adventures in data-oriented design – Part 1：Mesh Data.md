*本文档译自 blog.molecular-matters.com 的 "Adventures in data-oriented design" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
让我们面对一个现实，现代处理器（无论是PC、主机还是移动平台）的性能主要由内存访问模式决定。尽管如此，面向数据的设计仍然被认为是一种新颖的东西，近几年慢慢地为人所知，这种情况确实需要改变。让同事修复你的代码并提高它的性能真的不是你编写蹩脚代码的借口（从性能的角度来看）。

这篇文章是关于在 *Molecule* 引擎中如何以面向数据的方式完成某些事情的系列文章的第一篇，同时仍然使用面向对象的概念。关于面向数据设计的一个常见误解是，和面向对象相比，它更像C那样的面向过程，因此更不容易维护。但事实并非如此。我们今天要看的具体示例是如何组织网格数据，但让我们先从预备知识开始。


## 面向对象设计 - *Data-oriented design*
----
关于面向数据的设计大家已经说了很多，也写了很多，网上也有一些不错的[资源](http://www.asawicki.info/news_1422_data-oriented_design_-_links_and_thoughts.html)。*Mike Acton* （*Insomniac Games* 的技术总监）是面向数据设计的知名倡导者，他在网上也有一些有趣的[幻灯片](http://macton.smugmug.com/gallery/8936708_T6zQX#593426709_ZX4pZ)。

我不会详细介绍面向数据设计这个广泛的概念，所以让我快速总结一下面向数据设计对我来说是什么：

1. 首先考虑数据，其次考虑代码。类层次结构并不重要，但数据访问模式很重要。
2. 想想你的游戏中的数据是如何被访问的，它是如何被转换的，以及你最终会用它做什么，例如粒子，蒙皮，刚体等等。
3. 有一个，就会有很多。试着从数据流的角度思考。
4. 注意虚函数、函数指针和成员函数指针的开销。

为了让每个人都意识到第一点和第二点的重要性，考虑一个非常简单的例子：

```C++
char* data = pointerToSomeData;
unsigned int sum = 0;
for(unsigned int i = 0; i < 1000000; ++i, ++data) {
  sum += *data;
}
```

在上面的例子中，我们将一百万个字节累加，仅此而已。这里没有花哨的技巧，没有隐藏的C++开销。在我的电脑上，这个循环大约需要0.7ms。

让我们稍微改变一下循环，并再次测量它的性能：

```C++
char* data = pointerToSomeData;
unsigned int sum = 0;
for(unsigned int i = 0; i < 1000000; ++i, data += 16) {
	sum += *data;
}
```

唯一改变的是我们现在每隔16个元素才累加，也就是说我们把 `++data` 变成了 `data += 16`。请注意，我们仍然取恰好100万个元素的和，只是这次是不同的元素。

这个循环花了多少时间？5ms。让我为你解释一下：这个循环比原来的慢了7倍多，尽管我们访问的元素数量相同。性能与内存访问以及如何充分利用处理器的缓存息息相关。

**更新**：*Shanee Nishry* 在她的博客上进行了类似的[性能实验](http://shaneenishry.com/blog/2015/03/26/data-oriented-design-matters/)，并在不同的架构上进行了测试。

记住第三点，“有一个，就会有很多。试着从数据流的角度思考”。啥？这是什么意思？这很简单：如果当下你在游戏中有一个纹理、网格、音效、刚体等，你肯定（不久之后）就会有更多。以一种可以同时处理多个对象的方式来编写代码。这并不意味着你需要摆脱 *Mesh* 和 *Texture* 类，并把它们分别变成 *MeshList* 和 *TextureList*。这将在后面的示例中展示。

最后但同样重要的是，第4点：注意虚函数、函数指针和成员函数指针的开销。这是我非常关心的事情。这是一些缺乏经验的程序员经常忽略（或忘记）的东西，我总是确保教我的学生虚函数调用之类的东西的潜在代价。尽管如此，它们迟早多多少少会在内部循环里被使用，进而降低你的性能。

坦率地说，你不应该在对象的低级层级上使用虚函数，函数指针，成员函数指针，比如网格的每个图元集合，粒子系统中的每个粒子，纹理中的每个 *texel*；你明白我说的意思。它们可能会导致指令缓存和数据缓存无效，然后降低你的性能。


## 以面向数据的方式处理静态网格数据 - *Dealing with static mesh data in a data-oriented way*
----
话虽如此，让我们从今天的例子开始，看看如何以面向数据的方式处理静态网格数据。当然有很多方法可以组织数据，所以我们感兴趣的是完全面向对象的设计和更关注数据访问模式的设计之间的性能差异。

我们示例的场景如下：

+ 我们想要渲染500个静态网格体，并对它们执行视锥剔除。
+ 每个网格平均有一个顶点缓冲区，索引缓冲区和大约3个子网格。
+ 每个子网格存储一个起始索引和所属网格的顶点缓冲区/索引缓冲区中使用的索引数量。此外，每个子网格都有一个材质和一个轴对齐包围盒。
+ 材质包含一个漫反射纹理和一个光照贴图。

当然这没有什么特别复杂的东西，这使得我们更容易看到两种方法之间的区别。


## 按部就班的 OOP 方案 - *The by-the-book OOP approach*
----
让我们从面向对象的类开始设计：

```C++
class ICullable
{
public:
	virtual bool Cull(Frustum* frustum) = 0;
};
 
class SubMesh : public ICullable
{
public:
	virtual bool Cull(Frustum* frustum) {
		m_isVisible = frustum->IsVisible(m_boundingBox);
	}
	 
	void Render(void) {
		if (m_isVisible) {
			m_material->Bind();
			context->Draw(m_startIndex, m_numIndices);
	    }
	}
	
	 
private:
	unsigned int m_startIndex;
	unsigned int m_numIndices;
	Material* m_material;
	AABB m_boundingBox;
	bool m_isVisible;
};
 
class IRenderable
{
public:
	virtual void Render(void) = 0;
};
 
class Mesh : public IRenderable
{
public:
	virtual void Render(void) {
    context->Bind(m_vertexBuffer);
    context->Bind(m_indexBuffer);
	
    for (size_t i = 0; i < m_subMeshes.size(); ++i) {
		m_subMeshes[i]->Render();
    }
}
 
private:
	VertexBuffer* m_vertexBuffer;
	IndexBuffer* m_indexBuffer;
	std::vector<SubMesh*> m_subMeshes;
};
```

希望你们中的一些人在看到这样的设计时已经流下了眼泪。有人可能会认为以上是对面向对象设计的过度夸张，但我向你保证，我见过比这更糟糕的游戏。最糟糕的例子大概是这样的：

```C++
class GameObject : public IPositionable, public IRenderable, public IScriptable, ...
```

我不记得确切的细节了，但实际的实现继承了6个或者7个类，其中一些有实现，另一些只有纯虚拟函数，另一些有虚拟函数的唯一目的是让动态强制类型转换合法。我并不想指责任何人，而是想说这样的设计存在于已发行的商业游戏中，这并不是虚构的例子。不过让我们回到主题上。

回到最初的例子上来，它有什么不好的？嗯，有几点：

1. *IRenderable* 和 *ICullable* 两个接口在每次调用 `Render()` 和 `Cull()` 时分别造成一个虚拟函数调用。此外，虚函数几乎永远不能内联，这进一步降低了性能。
2. 剔除网格时糟糕的内存访问模式。为了读取 *AABB* （`m_boundingBox`），一整条缓存线（在现代架构中为64字节）将被读取到处理器的缓存中，无论你是否需要。这可以归结为访问更多不必要的数据，这对性能不利，正如我们在第一个简单示例中看到的那样（总和为100万个字节）。
3. 渲染子网格时糟糕的内存访问模式。每个子网格检查自己的 `m_isVisible` 标志，这将再次将更多的数据拉入缓存。和上面例子中一样的坏行为。
4. 从更高的层次看，内存访问模式也不正确。上面的第2点和第3点假设所有网格和子网格的数据都是线性排列在内存中的。在实践中，这种情况很少发生，因为程序员通常只是在某个位置 *new/delete*，然后就完成了。
5. 未来会写下的算法的糟糕内存访问模式。例如，要为几何图形渲染阴影贴图，不需要绑定材质，只需要起始索引和索引的数量。但每次我们访问它们时，我们也将其他不需要的数据拉入缓存。
6. 糟糕的多线程适应性。如何在不同的CPU上执行多个子网格的剔除？如何在SPU上进行剔除？

那么如何改善这种情况呢？我们要如何组织数据，让它在现在和将来都能提供良好的性能？


## DOD 方案 - *The data-oriented design approach*
----
显然第一步要去掉的是这两个接口，以及与它们关联的虚函数调用。相信我，你不需要这些接口。

我们可以改变的另一件事是确保需要一起访问的数据在内存中彼此相邻地分配。此外，如果这是隐式完成的，那么程序员就不必担心内存分配了。

按照面向数据的路线，设计可能是这样的：

```C++
struct SubMesh
{
	unsigned int startVertex;
	unsigned int numIndices;
};
 
class Frustum
{
public:
	void Cull(const float* in_aabbs, unsigned bool* out_visible, unsigned int numAABBs)
	{
    // for each AABB in in_aabbs:
    // - determine visibility
    // - store visibility in out_visible
	}
};
 
class Mesh
{
public:
	void Cull(Frustum* frustum)
	{
		frustum->Cull(&m_boundingBoxes[0], &m_visibility[0], m_boundingBoxes.size());
	}
	 
	void Render(void)
	{
	    context->Bind(m_vertexBuffer);
	    context->Bind(m_indexBuffer);
		 
	    for (size_t i = 0; i < m_visibleSubMeshes.size(); ++i)
	    {
		    if (m_visibility[i]) 
		    {
		        m_materials[i]->Bind();
		        const SubMesh& sm = m_subMeshes[i];
		        context->Draw(sm.startIndex, sm.numIndices);
		    }
	    }
	}
	 
private:
	VertexBuffer* m_vertexBuffer;
	IndexBuffer* m_indexBuffer;
	 
	std::vector<float> m_boundingBoxes;
	std::vector<Material*> m_materials;
	std::vector<SubMesh> m_subMeshes;
	std::vector<bool> m_visibility;
};
```

让我们来看看这种方法的特点：

1. 剔除子网格时良好的内存访问模式。为了剔除 *N* 个元素，内存中要访问 *6N* 个浮点数。可见性被写入内存中的独立目标，因此也没有开销。
2. 渲染子网格时良好的内存访问模式。所有需要的数据都连续存储在内存中，我们只将实际需要的数据加载到缓存中。
3. 为将来的计划提供良好的内存访问模式。如果我们想要子网格渲染阴影贴图，我们只需要访问 `m_subMeshes`，不需要其他的东西了。
4. 更高层次看，也有良好的内存访问模式。不需要手动分配内存来使元素连续。`std::vector` 总是连续的，简单数组也是如此。如果你愿意，还可以进一步将所有 *Mesh* 实例放到一个单独的容器中。
5. 多线程更容易了。如果剔除（或任何其他操作）需要多个线程（或SPU）进行拆分，我们可以将数据划分为块，并将每个数据块提供给不同的线程。在这些情况下不需要同步、互斥锁或类似的东西，而且解决方案的伸缩性几乎完美，可以很容易地利用4核、8核或n核处理器，这将是PC和主机的未来。

我并不是说这个解决方案是完美的，当然不是，它只是一个例子。这是我的进一步的改进点：

1. 我们可以直接存储子网格数据，而不是将 *bool* 存储到 `out_visible` 数组中，从而在 `Mesh::Render()` 中渲染可见网格时节省了分支和进一步的间接。
2. 更进一步，如果我们愿意，我们可以直接将渲染命令存储到GPU缓冲区中，例如在PS3上写入命令缓冲区。
3. 如果你的 *content pipeline* 支持，可以将所有静态数据烘焙到一个单一的大 *Mesh* 中。整个关卡的所有静态网格数据都保证在内存中是连续的，所以你可以自动获得上述解决方案的好处。

从这样的设计更改中，我们可以看到什么样的性能改进？在我进行的实验中（500个网格，每个子网格3个，不计算 *Direct3D11 API* 的调用开销），两种方案之间的差异约为1.5ms。

如果这听起来不算多，想想1.5ms实际上占了60FPS所需的帧目标时间（16ms）的10%。此外，我们在这里讨论的只是500个网格，并且只涉及静态网格数据的设计。如果面向数据的设计应用于整个引擎，那么性能收益将是巨大的。

正如所见，面向数据的设计大获全胜。在这篇文章的结尾，我要说的是，你现在需要开始关心你的数据访问模式，我是认真的。[*Scott Meyers*](http://skillsmatter.com/podcast/home/cpu-caches-and-why-you-care) 也是。