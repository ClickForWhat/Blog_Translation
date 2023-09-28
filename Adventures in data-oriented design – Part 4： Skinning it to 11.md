*本文档译自 blog.molecular-matters.com 的 "Adventures in data-oriented design" 系列文章，作者 Stefan Reinalter*


## 概述 - *Overview*
----
在完成关于数据所有权系列的第三部分之后，我们将在本文中再次将注意力转向性能优化和数据布局。更具体地说，我们将详细介绍如何通过一些简单的代码和数据更改来优化角色蒙皮。

我们要优化的示例程序有以下功能：

+ 计算100个角色的骨骼蒙皮并渲染它们，***each character is animated & skinned individually***.
+ 每个角色在不同的动画帧上播放不同的动画，意味着角色之间没有重用数据。
+ 每个顶点最多可以受到4个骨骼的影响。
+ 不用实例化渲染，也没有网格 *LOD*。

我们将要使用的角色有大约5k个三角形，顶点有位置和法线数据，以及一组纹理坐标。这就是示例的输出：

![[skinning_100_characters_1.png]]
<center>100个独立的带动画角色</center>

我们只对角色蒙皮的性能数据感兴趣，即CPU为100个计算骨骼蒙皮所花费的时间，我们暂时对渲染不感兴趣。从上面的截图中可以看到，每个角色在三种材质中选一种，并播放三种动画中的一种（蹲下，走或跑），从随机选择的动画帧开始，因此数据不能在角色之间重复使用。


## 初步实现 - *First implementation*
----
在初版中，蒙皮的实现如下：

+ 假设每个顶点有1到4个骨骼影响。
+ 顶点属性由一个位置数据（*float3*），一个法线数据（*float3*） 和一个纹理坐标数据（*float2*）组成。
+ 计算蒙皮的代码单线程运行。
+ 计算蒙皮的代码使用 *SSE* 优化的数学函数，用于所有矩阵和向量操作。

在C++伪代码中，基本上如下所示：

```C++
for each vertex i
{
	// load joint indices and weights
	uint16_t index0 = jointIndices[i * 4 + 0];
	uint16_t index1 = jointIndices[i * 4 + 1];
	uint16_t index2 = jointIndices[i * 4 + 2];
	uint16_t index3 = jointIndices[i * 4 + 3];
	
	float weight0 = jointWeights[i * 4 + 0];
	float weight1 = jointWeights[i * 4 + 1];
	float weight2 = jointWeights[i * 4 + 2];
	float weight3 = jointWeights[i * 4 + 3];
	
	// build weighted matrix according to matrix palette and bone weights
	matrix4x4_t mat0 = MatrixMul(matrixPalette[index0], weight0);
	matrix4x4_t mat1 = MatrixMul(matrixPalette[index1], weight1);
	matrix4x4_t mat2 = MatrixMul(matrixPalette[index2], weight2);
	matrix4x4_t mat3 = MatrixMul(matrixPalette[index3], weight3);
	
	matrix4x4_t weightedMatrix = MatrixAdd(mat0, MatrixAdd(mat1, MatrixAdd(mat2, mat3)));
	
	// skin position, normal, tangent, bitangent data
	vector4_t position = srcVertexData[i].position;
	...
	
	vector4_t skinnedPosition = MatrixMul(weightedMatrix, position);
	...
	
	dstVertexData[i].position = skinnedPosition;
	...
	
	// copy unskinned data
	dstVertexData[i].uvs = srcVertexData[i].uvs;
	...
}
```

对于那些熟悉蒙皮概念的人来说，应该很清楚代码在做什么。它根据骨骼权重构造加权矩阵，并使用动画系统生成的 *matrixPalette*。然后，位置、法线之类的数据使用之前算好的加权矩阵计算蒙皮，所有剩余的未蒙皮数据，如纹理坐标，顶点颜色等，仅仅简单地复制。

这个实现，在我的系统（i7-2600K, 4x 3.40 GHz）上，计算100个角色需要4.15ms。但是这里还有很大的改进空间，所以让我们先从最简单的优化开始。


## 拆分顶点流 - *Splitting streams*
----
在上面的循环中可以注意到的第一件事是复制非蒙皮数据。尽管这些数据从未改变，但每次都必须进行复制。

为什么？大多数情况下，我们只能在重新填充顶点缓冲区的情况下 *lock/map* 数据，例如，当使用 [*D3D11_MAP_WRITE_DISCARD*](http://msdn.microsoft.com/en-us/library/windows/desktop/ff476181(v=vs.85).aspx) 时。即使在 *PC* 平台的 *API* 中有不同的 *flag* 可供选择，或者如果我们可以直接访问GPU内存（就像我们在主机上所做的那样），仅写入一部分数据无论如何都是个坏主意。大多数GPU资源驻留在 [*Write-Combined memory*](http://fgiesen.wordpress.com/2013/01/29/write-combining-is-not-your-friend/) 中，在写入数据时留下个空白是一个特别糟糕的主意。在主机上，只把部分数据写入动态缓冲区的行为可能是真正的性能杀手！

*Molecule* 解决这个问题的方法是将蒙皮组件的顶点流分成两部分。一部分包含所有需要蒙皮的数据，另一部分包含所有不需蒙皮的数据。顶点数据在资产管线中被离线拆分为两个流，并且引擎运行时在渲染时使用两个而不是一个顶点流。

这种优化完全消除了对非蒙皮数据的所有不必要复制，在我们的例子中，将性能表现从4.15ms降低到4.00ms。非蒙皮顶点数据越多（顶点颜色、第二组uv等），我们拥有的顶点越多，节省的成本就越大。


## SIMD友好的数据布局 - *SIMD-friendly data layout*
----
如在关于实现的第一版本的段落中所述，位置数据（以及任何其他蒙皮数据）作为 *float3* 存储在目的顶点流中。大多数 *SIMD* *ISAs*（如 *Intel* 的 *SSE* 指令）一次操作4个浮点数，并且只支持从内存读取/写入整个浮点数。这意味着，为了将3个单独的浮点数（*x*、*y* 和 *z*）存储到内存中，我们首先必须将它们从 *SIMD* 数据类型中提取到不同的浮点数中，然后将它们写入内存。

这很糟糕。所有数据应该尽可能长时间地停留在同一个CPU管道中。标量数据应该留在标量管道中，*SIMD* 数据应该留在 *SIMD* 管道中。将数据从一个管道移动到另一个管道总是会损害主机上的性能，从浮点数据类型移动到 *SIMD* 数据类型（反之亦然！）总是会导致 [*load-hit-store*](http://www.gamasutra.com/view/feature/132084/sponsored_feature_common_.php?print=1) 损失，因为数据必须通过内存传输！对于在整数和浮点管道之间移动数据也是如此，但是让我们先不要进入基于 *PowerPC* 的架构领域...

幸运的是，所有这些都可以通过稍微增加顶点缓冲区的大小来轻松解决。在 *Molecule* 中所有蒙皮顶点缓冲区的预期数据类型是 *float4*， 而不是 *float3*。这允许我们将 *SSE* 数据类型直接写入顶点缓冲区内存。此外，*D3D11* 保证缓冲区数据是16字节对齐的，这意味着我们可以使用对齐的存储（[_mm_store_ps, MOVAPS](http://msdn.microsoft.com/de-at/library/s3h4ay6y(v=vs.90).aspx "_mm_store_ps")）。

此外，通过将源数据从 *float3* 更改为 *float4* 并在16字节边界上正确对齐，我们也可以使用对齐的读取（[_mm_load_ps, MOVAPS](http://msdn.microsoft.com/en-us/library/zzd50xxt(v=vs.90).aspx "_mm_load_ps")）。

这意味着我们可以直接从内存加载到 *SIMD* 类型中，对这些数据类型执行所有操作，并将结果写回内存，而无需离开 *SIMD* 管道。此优化将性能数据从4.00ms降低到3.53ms。请注意，即使我们必须读取和写入更多数据（从 *float3* 到 *float4*），性能仍然有所提高。这是为工作选择正确数据布局的一个很好的例子，这并不总是意味着保持数据尽可能小！诚然，在大多数情况下，尽可能减少读取或写入的数据量是个好主意。


## 运行时友好的数据布局 - *Runtime-friendly data layout*
----
请记住，面向数据的设计主要是关于如何读取、转换和写入数据。一种方法是在内存中保持同质连续的数据流，并且只存储尽可能少的数据。这意味着对于某些类型的数据（主要的例子是粒子），以 *SoA* 而不是 *AoS* 的方式存储数据通常是有益的。

在我们的示例中，蒙皮所需的所有数据都已经以一种缓存友好的方式存储：

+ 所有的关节索引和权值都是连续存储的。它们是顺序访问的，不会在内存中跳转。
+ 顶点数据只包含需要蒙皮的数据，未蒙皮的数据已经被踢出。
+ 顶点数据是按顺序读写的，不会在内存中跳转。

在没有任何关于数据本身的额外知识的情况下，可以说内存布局已经尽可能地适合缓存。

但它并不止于此！即使数据已经连续存储，考虑数据包含的内容也是有意义的，这种洞察力允许我们进一步优化内存访问。

请记住，我们在文章开头说过，假设每个顶点都有1到4个骨骼影响。这意味着对于那些具有少于4个骨骼影响的顶点，对应的权重将为零，并且我们在构建加权矩阵时做了大量不必要的工作。

我们怎样才能去掉多余的运算呢？新手程序员通常会试图通过在条件上进行分支来摆脱额外的工作，在条件不成立的情况下不做任何事情。在我们的例子中，我们可以简单地在不同的权重上进行分支检查它们是否真的对加权矩阵有贡献：

```C++
if (weight0 != 0.0f)
{
	weightedMatrix = MatrixAdd(weightedMatrix, MatrixMul(matrixPalette[index0], weight0));
}
 
if (weight1 != 0.0f)
{
	weightedMatrix = MatrixAdd(weightedMatrix, MatrixMul(matrixPalette[index1], weight1));
}
 
if (weight2 != 0.0f)
{
	weightedMatrix = MatrixAdd(weightedMatrix, MatrixMul(matrixPalette[index2], weight2));
}
 
if (weight3 != 0.0f)
{
	weightedMatrix = MatrixAdd(weightedMatrix, MatrixMul(matrixPalette[index3], weight3));
}
```

坏消息是，通过引入这些额外的分支，代码很可能比原始实现运行得更慢！硬件分支预测在检测遵循特定模式的分支方面确实很好，但对于或多或少是随机的分支就不那么好了。此外，在当前基于 *PowerPC* 的主机等按顺序运行的CPU上，分支预测失败的代价大约是24个时钟周期，因此您最好还是进行数学运算。

要考虑的另一种选择是存储每个顶点的骨骼影响数量，并用单独的循环替换构建加权矩阵的代码，类似于以下代码：

```C++
for each influence j
{
	weightedMatrix = MatrixAdd(weightedMatrix, MatrixMul(matrixPalette[index_j], weight_j));
}
```

请注意，我们现在需要存储每个顶点的影响数量，并且每个顶点增加了内部循环的开销。是的，这样的地方的环路确实有可测量的开销。此外，由于需要保存每个顶点的影响数量，我们需要从内存中获取更多数据。如果你的蒙皮网格有很多顶点有3或4个骨骼影响，那么“优化”代码很可能会因为额外的开销和内存获取而运行较慢——所以，我们只是针对边缘情况进行了优化。

我们确实希望代码能够完美地适应输入数据，无论数据是什么。

存储每个顶点的骨骼影响数量已经朝着正确的方向发展，但引入了太多的开销。但是我们可以做一件简单的事情：与其存储每个顶点的影响计数，为什么不对数据进行排序并存储具有一定影响计数的顶点数量呢？我们知道我们有1、2、3或4个影响，所以我们只需要存储4个循环计数。

因此，我们不按任何顺序存储顶点，而是重新排列它们（与索引缓冲区一起），以便源数据首先包含所有仅具有1个骨骼影响的顶点，然后包含所有具有2个影响的顶点，依此类推。除此之外，我们还存储具有1个影响的顶点数量，具有2个影响的顶点数量，以此类推。所有这些都可以在资产管道中方便地完成，所以它不会产生额外的运行时成本。

好消息是，这允许我们通过为每个路径编写专门的代码，为每个顶点只做绝对可能的最小工作。此外，这开辟了新的优化可能性，因为我们现在确切地知道数据包含什么！举个例子，我们不需要存储只受一个骨骼影响的权重，它肯定是1.0f。这样可以节省内存并减少内存访问。

最后，代码看起来如下所示：

```C++
for numVertices1Bone:
{
	uint16_t index0 = jointIndices[i];
	
	vector4_t position = srcVertexData[i];
	vector4_t skinnedPosition = MatrixMul(matrixPalette[index0],position);
	
	dstVertexData[i] = skinnedPosition;
}
 
for numVertices2Bones:
{
	uint16_t index0 = jointIndices[i*2 + 0];
	uint16_t index1 = jointIndices[i*2 + 1];
	
	float weight0 = jointWeights[i*2 + 0];
	float weight1 = jointWeights[i*2 + 1];
	
	matrix4x4_t mat0 = MatrixMul(matrixPalette[index0], weight0);
	matrix4x4_t mat1 = MatrixMul(matrixPalette[index1], weight1);
	
	matrix4x4_t weightedMatrix = MatrixAdd(mat0, mat1);
	
	vector4_t position = srcVertexData[i];
	vector4_t skinnedPosition = MatrixMul(weightedMatrix, position);
	dstVertexData[i] = skinnedPosition;
}
 
// code for 3 and 4 influences omitted
```

请注意，我们只执行绝对需要的操作。没有内循环，没有额外的分支，只是简单的代码。

正如你可能已经猜到的那样，这种优化产生了最大的回报——它将性能数据从3.53ms降到2.00ms。当然，这取决于每个顶点的影响数量，但它在任何情况下都是最优的，而不仅仅是边缘情况。对于5k测试角色，每个顶点的影响百分比大致分布如下：

30%的顶点有4个骨骼影响，25%的顶点有3个，20%有2个，25%只有1个骨骼影响。

## 最终调整 - *Final touches*
----
作为最后的优化，可以重写代码，使其对编译器更友好，这使开销从2.00ms降到1.78ms。因为这些优化是高度特定于编译器的，所以我在这里不详细介绍。*MSVC* 似乎喜欢紧凑的代码，因为这会减少堆栈上的寄存器溢出。*GCC* 似乎更喜欢详细的代码（类似于上面写的代码），它清楚地显示常量和非别名变量。

最后但同样重要的是，构建一个64位可执行文件可以额外加速15%，因为有更多的 *SSE* 寄存器可用，从而减少寄存器溢出。

## 结语 - *Conclusion*
----
在最后，我们通过一些数据转换、一些代码更改和针对64位系统的构建，成功地获得了约2.75倍的速度提升。实现所有这些花了大约半天的时间，这是绝对值得的。利用 *Molecule* 的 *JobSystem* 并完全多线程化进一步提高了性能，几乎完美地（线性地）扩展到所有可用的核心。如上所述，代码仅由4个简单的循环组成，这些循环可以简单地并行化。

最后需要说明的是，对数据进行转换对于基于计算着色器的体系结构也是有益的，因为需要读取的数据更少，需要做的工作也更少。