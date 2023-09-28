*本文档译自 blog.molecular-matters.com 的 "Adventures in data-oriented design" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
游戏开发中常见的一项任务是根据某种层次结构来变换数据。今天，我们要看一个可能是此类任务中最著名的例子：根据骨骼层级变换关节。

具体来说，我们将研究简单的基于 *OOP* 的常规实现与面向数据的设计之间的区别。我们将讨论实现和性能方面的差异。我们将再次看到，面向数据的设计和面向对象的编程并不矛盾。


## 骨骼动画 - *Skeletal animation*
----
在开始考虑可能的实现之前，我们首先需要快速确定骨骼动画所需的步骤。

通常，一个动画网格（如角色）由几个关节组成，这些关节以层级方式相互连接。动画数据为每个不同部件指定动画曲线，以此来构成一个关节的变换，包括用于平移的x、y和z值（还有缩放，如果支持的话），用于旋转的x、y、z和w值（存储为四元数）。

![[skeleton.jpg]]
<center>包含96个关节的骨骼</center>

每个关节的动画数据都存在它们局部坐标系中，因为这样更容易制作动画，并形成更自然的动画效果。这意味着无论何时对动画曲线进行采样，所有的变换都存储在局部坐标系中，并且需要转换到全部坐标系中，以便渲染关节或角色蒙皮。因此，对动画曲线进行采样的结果可以为每个关节进行一个变换，这被称为 *local pose*。

请注意，如果我们想在不同的关节变换之间混合（通常使用 [*LERP*](http://en.wikipedia.org/wiki/Lerp_%28computing%29) 或 [*SLERP*](http://en.wikipedia.org/wiki/Slerp) 完成），这些混合是在 *local pose* 上执行的。在混合、分层等操作完成后，所有的关节都变换成它们的 *global pose*。这是通过关节层级进行的。

概括一下，这些是需要执行的步骤：

1. 算出我们要在哪个时间点对动画曲线进行采样。这受到动画的速度，是否循环等影响。采样结果被称为 *local pose*。
2. 可以选择混合不同的 *local pose*，以平滑动画。
3. 根据层级结构将 *local pose* 转换为 *global pose*。

我故意省略了其他步骤，如布娃娃系统（这些步骤会影响 *global pose*，并且需要来回倒腾 *local pose*（IK, FK））以及其他的系统，以便专注于这篇文章的关键部分。


## 示例 - *Demo setup*
----
我们将在下面的示例中测量不同实现之间的差异，只对1000个角色执行上述的步骤。我们的计时器将测量执行第3步（遍历骨骼层级并以此变换关节）1000次所需的时间。在这个示例中，我们将使用由96个关节组成的角色（如上面的图片所示）。

![[characters.jpg]]
<center>一千个角色组成的军队，它们将在不同的时间点采样动画资源</center>


## 初步的实现 - *Initial implementation*
----
骨骼的层级数据结构的简单实现如下所示：

```C++
struct Joint
{
	math::matrix4x4_t globalPose;
	math::matrix4x4_t localPose;
	std::vector<Joint*> children;
};
```

相当简单：每个关节存储一个 *local pose*、一个 *global pose* 和一个子关节数组。然后骨骼就可以简单定义为关节的数组，就是这样。为了从根开始变换骨骼，可以使用一个简单的递归函数：

```C++
class Skeleton
{
public:
	void LocalToGlobalPose(void)
	{
	    // start at the root
	    LocalToGlobalPose(&joints[0], math::MatrixIdentity());
	}
	 
private:
	void LocalToGlobalPose(Joint* joint, math::matrix4x4_arg_t parentTransform)
	{
		joint->globalPose = math::MatrixMul(parentTransform, joint->localPose);
		 
	    // propagate the transformation to the children
	    for (size_t i = 0; i < joint->children.size(); ++i)
	    {
		    LocalToGlobalPose(joint->children[i], joint->globalPose);
	    }
	}
	 
	Joint* joints;
};
```

上面从根开始遍历骨骼层级，并将父节点的变换传播到其子节点。这是通过存储计算出的关节 *global pose*，然后将其传递给子节点来实现的。代码很短，应该是不言自明的。

所以问题是，使用上面的代码计算1000个角色的 *global pose* 需要多长时间？答案是3.8ms。现在还不能确定这是好是坏，但我们确实可以做得更好，否则我就不会发这文章了。


## 面向数据的方案 - *A data-oriented approach*
----
这里要注意到的第一件事相当简单，而且大多数人都不会感到惊讶：我们不再为每个关节存储子关节，而是反过来为每个关节存储父关节。因此，我们存储的不是 `std::vector<Joint*> children`，而是 `Joint* parent`。

下一项优化也很简单：无论如何，我们都将骨骼的所有关节存储在数组中。所以为什么存储索引而不是指针呢？假设任意骨骼不会超过65536个关节（我觉得这是一个安全的假设），我们可以以 `uint16_t parent`  的方式储存而不是像前面那样用指针。

那么好处是？

+ 存储一个 `uint16_t` 比存储一个指针需要的空间更少，特别是在64位系统上。我们节省了空间，并且在计算 *global pose* 时需要访问的内存更少。
+ 现在，骨骼类是一个可平凡复制的类型，你可以用 `memcpy` 之类的办法在内存中移动它。***Similarly, a whole skeleton can be loaded from disk using a single binary read without having to worry about pointer-fixups or similar.***  仅此一点就值得对数据进行这样的改造。

剩下的问题是：我们该如何更改代码，使我们仍然能以适当的顺序遍历骨骼层级，以及我们如何摆脱递归？

为了解决这个问题，有一个关键点需要注意：只要我们总是以相同的顺序遍历骨骼层级，我们就可以以这种顺序分配和存储关节，并使它扁平化（*flattening*）来让我们能够线性遍历关节数组。看看上面的递归代码，我们可以看到代码以所谓的深度优先遍历骨骼层级。

现在，想象一下我们正在遍历骨骼层级，用单调递增的数字对访问的关节进行编号。关节0将是根，关节1则是骨骼层级中按照深度优先遍历的下一个，依此类推。这意味着如果我们遍历关节数组，一个关节i只能有一个满足 `j < i` 的关节j作为它的父关节。这极大地简化了遍历代码，并摆脱了递归。

最后要做的一件事是将布局从 *AoS* （*array-of-structures*）更改为 *SoA*（*structure-of-arrays*），剩下的内容如下：

```C++
class Skeleton
{
private:
	uint16_t* hierarchy;
	math::matrix4x4_t* localPoses;
	math::matrix4x4_t* globalPoses;
};
```

这个变化的好处是，当我们使用 `hierarchy[i]` 访问某一个关节的父关节时，接下来要遍历的几十个关节也会被读入同一个缓存行（*cache line*）的缓存中。访问 `localPose[i]` 也是如此。

所以计算 *global pose* 的代码是什么样的呢？它变得出奇的简单：

```C++
void LocalToGlobalPose(void)
{
	// the root has no parent
	globalPoses[0] = localPoses[0];
	
	for (unsigned int i = 1; i < NUM_JOINTS; ++i)
	{
	    const uint16_t parentJoint = hierarchy[i];
	    globalPoses[i] = math::MatrixMul(globalPoses[parentJoint], localPoses[i]);
	}
}
```

很漂亮，没有什么比这更简洁的了。

如上所述，正因为 `parentJoint < i` 恒成立，意味着我们将只访问已经计算好的 *global pose*。之前的递归遍历消失了，原先的递归遍历现在由数组中关节的位置隐式给出，我们以深度优先的方式给它们编号。

这在性能上有什么不同？对于相同数量的计算，这段代码现在需要3.1ms。与之前的3.8ms相比，速度提高了18%以上！


## 结语 - *Conclusion*
----
我们所做的一些简单更改不仅大大提高了性能，还极大地简化了代码本身。我们不需要放弃面向对象的原则，我们甚至不需要改变调用代码。我们仍然保有我们的骨骼类，并能够在其上调用 `LocalToGlobalPose()`。

简洁、快速、简单。