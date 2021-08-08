# 在Arm Mali GPU上使用异步计算：附实践代码

- 标题：Using asynchronous compute on Arm Mali GPUs: A practical sample
- https://community.arm.com/developer/tools-software/graphics/b/blog/posts/using-asynchronous-compute-on-arm-mali-gpus
- 作者：Hans-Kristian Arntzen
- 时间：June 16, 2021

异步计算技术确实是效果拔群的性能优化杀器，但难于在项目中落地。异步计算始于上一代[控制台硬件（Console Hardware），就是游戏主机时代](https://www.retroreversing.com/hardware/consoles)，在后来的现代图形 API（如 Vulkan 和 D3D12）上逐步体现，目前异步计算已是图形程序员的常用工具。

本文将展示一个新的 Vulkan 示例，该示例代码也已更新到了 Github KhronosGroup 下的 [Vulkan-Sample](https://github.com/KhronosGroup/Vulkan-Samples) 仓库中，其演示了如何使用异步计算，感兴趣的同学也可以看看这篇 [《使用异步计算技术榨干GPU资源》](https://github.com/KhronosGroup/Vulkan-Samples/blob/master/samples/performance/async_compute/async_compute_tutorial.md)。

严格来说，"异步计算"（Async compute）本身不一种技术，它只是通过一次同时向 GPU 提交多个命令流（multiple streams of commands）来有效利用现代 GPU 可用的硬件资源的策略。咱们后文会具体展开来讲，因为把异步计算用起来，确实需要一定的专业知识。

> 本文的代码示例是基于我 [2018 年博客文章](https://community.arm.com/developer/tools-software/graphics/b/blog/posts/using-compute-post-processing-in-vulkan-on-mali) 的代码改进的，改进后的代码证明了基于计算的后处理可以带来性能提升。


# 异步计算的意义

现代 GPU 有多个队列，通过队列提交任务给着色器（Shader Core）处理。桌面级 GPU 和 [Arm Mali GPU](https://developer.arm.com/ip-products/graphics-and-multimedia/mali-gpus) 在队列执行拓补结构上不同，这也导致了在异步方法的实现上二者的差异。下面，我们先分析传统桌面 GPU 和 Arm Mali GPU 的架构差异。

## 桌面级 GPU 的队列任务流程

传统桌面 GPU 架构的渲染模式属于 **立即模式 （Immediate Mode）**，立即模式下渲染流程会严格地按照命令队列的方式执行：在每个图元（primitive）上的每次绘制调用（draw call）中，以顺序地方式执行顶点和片段着色器（vertex and fragment shaders）的工作负载。

> 渲染是从 2D 或 3D 模型借助计算机程序生成现实世界的真实或非非真实图像的过程。渲染也是图形流水线的最后一个主要步骤，让模型或者动画得到最终的外观呈现。

这个流程以伪代码的形式表示即为：

```python
for draw in renderPass:
    for primitive in draw:
        for vertex in primitive:
            execute_vertex_shader(vertex)
        if primitive not culled:
            for fragment in primitive:
                execute_fragment_shader(fragment)
```

这里的**绘制调用**（draw call）为 GL 的描绘次数，也可称为一条用来渲染网格的命令，这条命令由 CPU 发出，并被 GPU 接收。GL 绘图一般次序为：设置颜色——绘图方式——顶点坐标——绘制——结束。每帧会重复该过程，该过程就是一次`draw call`。即上述伪代码中的 `renderPass`的一次`draw`。

> 网格（Mesh）代表一个可绘制实体，一般来说，Mesh 指 3D 模型的网格，由多边形拼接而成的，复杂多边形由多个三角面拼接而成的。所以一个 3D 模型的表面是由多个彼此相连的三角面构成的，三维空间中，构成这些三角面的点以及三角形的边的集合就是 Mesh，在 OpenGL 中我们可以定义网格（Mesh）类结构体，其成员可以是多个顶点（Vertex）结构体的组合，而顶点（Vertex）是包含了位置、纹理坐标等信息的结构体。

`draw call`命令发出后，GPU 使用 Render State（材质、纹理、着色器）和所有顶点数据（这些数据就是下图中的`Atrtibutes`和`Textures`），通过代码魔法将这些信息转换为屏幕上的彩色像素，这个转换过程也被称为 **Pipeline**（流水线，或者别扭的直译叫法“管线”，我不喜欢叫“管线”:p） 。该过程简化用下图表示，中间如栅格化等操作略过（栅格化—— rasterization，表示将计算机图形学中的[向量图形](https://zh.wikipedia.org/wiki/向量圖形)转换成像素阵列，即[位图](https://zh.wikipedia.org/wiki/位图)的过程）：

![img](https://developer.arm.com/-/media/developer/Graphics%20and%20Multimedia/Developer%20Guide%20Article%20Inline%20Images/Tile-based%20rendering/imr.svg?revision=c0aeae87-6090-48b5-a797-31ced10e01f3&la=en&hash=23B87C30910C78519B6E9E4111BA8080C6C66571)

上图分为三部分，第一行为 GPU 这一侧的处理，第三行为主机端数据，中间的第二行为数据交互流向。其中，蓝色为硬件单元，橘色为数据结构，绿色为数据。渲染流程为：主机端将图元的属性信息数据通过 DDR 发给 GPU，并交给**顶点着色器处理，用于处理如几何变换、灯光等如一个矩形的四个顶点即顶点着色器会被调用四次**，其处理结果以先进先出的队列数据结构保存，并交给**片段着色器（Fragment Shader）处理，Fragment Shader 的工作内容很多，包括不限于每个像素的最终颜色等属性。Fragment 是可被渲染到屏幕上的像素点**。

总之，上面，在桌面级GPU硬件上，渲染队列通常是两种组成。**因为文中说的不清楚，我的理解是：**有两个串行的队列，分别有其硬件实现，对应是上图的 Vertex Shader 和 Fragment Shader。Vertex Shader 是处理计算负载任务，即COMPUTE Queue，而 Fragment Shader 是 GRAPHICS Queue：

- 1个单独（lone）的**图形 （Grahpics）** 队列：什么类型的任务都可以做，来者不拒，这个队列很可能很长，**在图上表示为第一行（GPU）的 Fragment Shader的工作**；
- N个（许多个）仅**计算（Compute）的队列**：只能做计算任务负载，**在图上表示为第一行（GPU）Vertex Shader的工作**。

这么看，传统桌面级 GPU 两个串行队列的立即模式下，非常低效。这里也[科普一下这种最早的**立即模式渲染**（Immediate Mode Rendering，IMR）](https://www.igao7.com/news/201406/1217-vv-gpu.html)。

> 传统桌面 GPU（nVIDIA，AMD）都是 IMR 架构，在移动领域，nVIDIA 的 GeForce ULP 和 Vivante 的 GC 系列 GPU 都是属于 IMR 架构。IMR 架构的 GPU 渲染完物体后，都会把结果写到系统内存中的帧缓存里（FrameBuffer，见上图的Framebuffer Working Set），因此就可能出现 GPU 花了大量的时间渲染了一个被遮挡的看不见的物体，而最后这些结果在渲染完遮挡物后被覆盖，做了无用功。这个问题称之为 Overdraw。
>
> 虽然现代的 IMR 架构 GPU 在一定程度上可以避免这个问题，因为后续又有了 TBR （Tlie Based Rendering）架构，但要求应用程序将场景里的三角形按照严格的从前往后的顺序提交给 GPU，要完全避免 Overdraw 还是很困难的，当然后续的TBDR（Tile Based Deferred Rendering）架构完全避免了这个问题，是后话了。

## 移动端 Arm Mali GPU 的队列任务流程

Arm Mali 是 **Tile Based GPU**，其实这个架构不仅在 Mali 上有用到，TBG 是很多移动端 GPU 早起的通用架构，其特点是相比传统的桌面 GPU 立即模式，TBG 以大小为 16 x 16 的 tile 解决了在 Fragment Shader 计算过程中，与 DDR 反复读取数据的带宽问题， 且能被进一步压缩减少 DDR 传输开销，**说白了就是更省带宽**。分块也因此带来了相应的分块队列布局，也就是 Tiler 数组，Arm Mali GPU 其简化的渲染流程如下图。

![hardware flow](https://developer.arm.com/-/media/developer/Graphics%20and%20Multimedia/Developer%20Guide%20Article%20Inline%20Images/Tile-based%20rendering/tbr.svg?revision=0d2147a0-2d78-456d-949e-94c3731bd6ba&hash=CD76FA018D89460B8A5F1DACAE6C338F4F7E404C&la=en)

上图中，渲染流程（pipeline）从硬件处理角度来说，被中间的虚线一分为二，我们简单说一下：

1. 第一条路径—— Attributes(DDR) -> Vertex Shader -> Tiler -> Geometry Working Set(DDR)：其中 Vertex Shader 和 Tiler 的过程构成第一个硬件任务处理队列，这里与桌面 GPU 的不同是，中间结果 Geometry Working Set 会存储到 DDR 上，估计也是对 On-Chip memory 大小的考虑；
2. 第二条路径—— Geometry Working Set(DDR) / Textures(DDR) -> Fragment Shader <-> Local Tile Memory(GPU) -> Compress Framebuffer(DDR)：Fragment Shader 则是第二个硬件队列的起始处理节点，与 Fragment Shader 交互的在桌面级 GPU 只有均位于 DDR 上的 Texture 和 FrameBuffer Working Set，前者是输入，后者是输出和临时结果暂存；而移动端 GPU 与 Fragment Shader 交互的有三个：位于 DDR 上的 Texture 和 Geometry Working Set 是输入，**而 Local Tiled Memory 是位于 On-Chip 的，这是与桌面级 GPU 的极大不同，是对性能和片上内存的综合考虑**，其经过压缩进而变为 DDR 上体积更小的 Compressed FrameBuffer。

计算任务会随着 Vertex Shader 和 Tiler 的处理而进行，其实也可以把 Vertex Shader 看成是 Compute Shader ，因为它做的都是计算。这两个硬件处理流程对应的伪代码如下：

```cpp
# Pass one
for draw in renderPass:
    for primitive in draw:
        for vertex in primitive:
            execute_vertex_shader(vertex)
        append_tile_list(primitive)

# Pass two
for tile in renderPass:
    for primitive in tile:
        for fragment in primitive:
            execute_fragment_shader(fragment)
```

说到这里的硬件队列，不得不说软件逻辑角度的队列，即 Arm Mali Vulkan 的 `VkQueue` API了，这是个对底层实现不咋透明的 API（我后文会具体讲），根据 [Khronos](https://www.khronos.org/registry/vulkan/specs/1.2-extensions/man/html/VkQueue.html) 和 [Arm](https://arm-software.github.io/vulkan-sdk/class_mali_s_d_k_1_1_platform.html#a8f3c9ad890211644f78e5307e195935e) 的文档，`VkQueue`是 Vulkan device queue，在创建获取到逻辑硬件后，与之相关 queues 也被创建，与之相关的 API 有：

- 获取该对象的接口 `getGraphicsQueue()`，用于获取当前 Vulkan Graphics queue；
- 获取当前 Vulkan Graphics family index的接口`getGraphicsQueueIndex()`。

Vulkan 与 OpenCL 的纯计算 Queue 有着明显不同，[Vulkan 的 Queue 有 4 种类型的 Flag](https://stackoverflow.com/questions/55272626/what-is-actually-a-queue-family-in-vulkan)：Graphics、Compute、Transfer、Sparse，每种 Queue 可以同时有多个 Flag ，然后 Queue Family 是指有一系列相同 Flag 的 Queue，也就是说一个 Family 内所有Queue都具有相同的属性。不过要真正理解 Queue Famliy 需要先理解 Vulkan 的 Queue：

命令缓冲区会被提交给队列，提交到一个队列的命令缓冲区任务，是按顺序挨个执行的。提交到不同队列的命令缓冲区任务，除非显式地使用`VkSemaphore`做同步，默认情况下，彼此间的任务顺序是无序的。**一个时间点上，只能从一个线程向一个队列提交工作**，但**不同的线程可以同时向不同的队列提交工作**。

每个队列只能执行某些（一种或者多种）固定 Flag 类型的操作。

- Graphics 队列可以运行由`vkCmdDraw*`命令启动的图形流水线（Graphics pipeline）；
- Computer 队列可以运行由`vkCmdDispatch*`启动的计算流水线（Compute Pipeline）；
- Transfer 队列可以运行从`vkCmdCopy*`启动的传输拷贝操作；
- Sparse Binding 队列可以通过使用`vkQueueBindSparse`，改变稀疏资源与内存的绑定。注意这是直接提交到队列的操作，而不是命令缓冲区中的命令。

一些队列可以执行多种操作。在规范中，可以提交到队列的每个命令都有一个“命令属性”表，这些“命令属性”就是前面所说的 Flag，其列出了可执行命令的队列类型。下面这段代码展示了查询 Queue Family 队列类型的过程：


```cpp
vector<vk::QueueFamilyProperties> queue_families = device.getQueueFamilyProperties();
for (auto &q_family : queue_families)
{
    cout << "Queue number: "  + to_string(q_family.queueCount) << endl;
    cout << "Queue flags: " + to_string(q_family.queueFlags) << endl;
}

// print result as below:
// Queue number: 16
// Queue flags: {Graphics | Compute | Transfer | SparseBinding}
// Queue number: 1
// Queue flags: {Transfer}
// Queue number: 8
// Queue flags: {Compute}
```

通过获取当前的队列族（Queue Family），可以看到上述代码对应的当前设备支持三种队列：

1. 可以进行**图形、计算、传输和稀疏绑定**（即同时支持三种 Flag ）的操作，最多可以创建 16 个该类型的队列；
2. **只能做传输操作**，这种队列只能创建一个。通常是独立显卡（GPU）与主机之间异步 DMA 传输数据的，即传输过程独立于图形或计算操作，异步完成传输；
3. **只能计算的队列**，最多8个。

**一些队列可能只是主机侧虚拟出来的，即逻辑的，并非有硬件的实际队列中对应，有的则相反，可能存在硬件队列与软件编程时对应。**

例如，许多 GPU 只有一个硬件图形队列（Graphics Queue），那么即使从有 Graphics Flag 的 Queue family 中创建两个 VkQueue，内核驱动程序提交这些队列到命令缓冲区的处理过程，仍会转为某种串行方式执行在 GPU 上。但有些 GPU 有多个仅能计算 （Compute）的硬件队列，因此一个只计算 queue Family 的两个 VkQueue ，实际上这两个逻辑队列，可能会在整个 GPU 中没有依赖且独立并发地执行。**Vulkan 不会公开这一点，所以前面我说，`vkQueue` 对底层实现不咋透明**。

> 实际使用中，完全可以根据当前设备拥有的并发量来决定使用多少个队列。对于许多应用程序来说，一个“通用”队列就能满足使用。更高级的用法可能是一个图形+计算队列、一个单独的用于异步计算工作的仅计算队列和一个用于异步 DMA 的传输队列。
>
> 此外，这些队列的执行是顺序开始，但是运行起来后就独立，即可能以乱序的方式结束执行。

## 保证硬件队列一直处于计算状态

基于分块的 GPU 架构会导致渲染流水线被一分为二，但实际中，这会导致流水线停顿，但我们希望 Fragment queue 时刻处于繁忙状态，即下图的 Texture Shader。

> ### Texture Shader、Fragment shader、Pixel Shader这三个是一个东西嘛
> Pixel和Fragment shader有些gamedev说是一样的，只是同义词。而且Pixel Shader是以前很早就流传下来的，而且“像素着色器”（Pixel shader）的“像素”（Pixel）是用词不当，因为像素着色器不直接操作像素。像素着色器操作的是片段（Fragment）,往往可能不是最终的实际像素，因为还取决于几个pixel shader之外的因素。（https://www.gamedev.net/forums/topic/388566-pixel-shader-vs-fragment-shader/）
> 【】【】【】

![image](https://user-images.githubusercontent.com/7320657/124752727-ca5bda00-df5a-11eb-9a17-39378d0e4eb3.png)
**图：TBR Pipeline**（来自 PowerVR 的 TBR 架构）

上图是前文基于分块的渲染流水线的更细节的版本，大的来说是两阶段：Vertex 开始计算的第一阶段，以及从逐个 Tile 的光栅化开始的第二阶段，由于每次处理 Tile 大小的分块如16x16或者32x32，片上缓冲区（on-chip buffers）也可调整为相应大小，在这之后，图形硬件使用片上缓冲区进行颜色、深度和模具缓冲区读写操作（colour, depth and stencil buffer Read-Modify-Write operation）。通过对片上缓冲区的使用，避免了系统内存的传输操作（尽管TBR方法改进了传统IMR设计，但它并没有减少 Over Draw，这个问题我下一篇文章再写吧）。

![Rendering Pipeline Flowchart](https://www.khronos.org/opengl/wiki_opengl/images/RenderingPipeline.png)

[图：渲染流水线，蓝色方块为可编程着色器阶段](https://www.khronos.org/opengl/wiki/Rendering_Pipeline_Overview)

因为 TBR 这个过程中，**每个线程占用大量带宽，导致 Vertex 和 Tiling 的工作效率较低最低**。**这是本文要解决的问题**，而**答案的核心是 Fragment queue 可以单独工作**（在 Arm Mali 架构图上是 Fragment Shader，在 PowerVR 的上图我理解的对应是 Texture & Shade），片段队列能够继续工作是很重要的。将着色器核心填满几何体工作，可能导致外部带宽传输工作的暂停。这里的几何体工作并没有在上图体现出来，一般几何体工作指的是几何着色器（Geometry Shader）的任务，几何着色器通常位于第一阶段的流程中：顶点着色器->曲面细分着色器->几何着色器->裁剪，甚至有些地方把第一阶段整体称为几何阶段（Geometry Stage）。

![GPU](https://www.programmersought.com/images/192/ece191e7e8e5bf265339d03f5d4a1060.JPEG)

[图：GPU渲染流程分为几何阶段与光栅化阶段](https://www.programmersought.com/article/66693616460/)

其实在整个流水线过程中，从 Vertex / Compute → Fragment 的依赖是很好的，因为刚我们才说过 Fragment Queue 可以单独工作。但现在问题出现在 Fragment → Vertex / Compute 的依赖关系的计算，这是本示例将探讨和解决的问题。

## 案例研究：后期效果的计算

使用计算 Shader 来做**后期效果**变得越来越流行，现代游戏引擎正在向这样的情形发展——即主要 pass 的光栅化（rasterization）在渲染总体任务中所占的比例越来越小。所谓**后期效果，是指依赖于当前帧的任何片段着色（Fragment shader）的相关计算过程**，例如，高动态范围（HDR）bloom、景深、模糊效果等。

传统上，后期效果以一连串的渲染过程实现，但有些情况如高动态范围（HDR）中的约简操作（Reduction Operation），**用计算着色器相比片段着色器实现更有吸引力**：

1. 毕竟若用片段着色器计算且最终一连串渲染过程得到的仅仅是一个很小尺寸的渲染结果，那可就真不划算了；
2. 此外，Render Target，是现代GPU有的一个特性，指的是即将被渲染为即时内存缓冲（Intermediate memory buffer），或者渲染目标纹理（Render Target Texture）的一个3D场景。compte shader比pixel shader的优势在两个地方：一是不用切换渲染目标（render target，如这次渲染是渲染到颜色贴图、还是深度贴图）避免切换开销，因为每次切换目标在GPU层都会有一系列的设置状态带来的消耗；

二是可以写到别的像素，，此外，在移动端TBDR中一般还会导致大量的带宽消耗，

### 流水线的中断

虽然知道了使用计算着色器比片段着色器好，但实际计算后期效果时，我们很容易陷入流水线中断，并由此导致性能降低的情况。再回到刚提到的流水线计算流程：

VERTEX → FRAGMENT（场景渲染）→ COMPUTE（后期效果计算如HRD）→ 如何进入屏幕？

为了进入屏幕，我们最终必须在 FRAGMENT 中做一些事情，我们得到可怕的 FRAGMENT → COMPUTE → FRAGMENT。有了这个屏障，我们就无法使用 FRAGMENT 着色，这是我们不想要的。

## 能否将计算与结果的屏幕呈现整合统一？

理论上，在 Vulkan 中是可以的，并有几个桌面游戏也确实这么做的。但是移动设备的一个重要绊脚石是我们将如何处理 UI 渲染。在渲染通道中渲染 UI，只是将其写回内存，然后在计算通道中合成，从带宽的角度来看是非常浪费的。如果可以，我们绝对应该避免这种情况。

### 样品起点

![ ](https://community.arm.com/resized-image/__size/1040x0/__key/communityserver-blogs-components-weblogfiles/00-00-00-20-66/Async-compute-image2-_2800_final_2900_.jpg)

这是我们开始的示例。场景组合非常简单，可作为大型计算后期繁重应用程序的代理。提高分辨率以更容易看到性能差异：

- 阴影贴图，8K（顶点/片段）
- 主通道渲染，前向着色，4K（VERTEX / FRAGMENT）
- Threshold + Bloom blur (COMPUTE) – 复杂后期效果的代理
- Tonemap + UI (VERTEX / FRAGMENT) – 代表我们如何在片段中结束帧

正如我们从性能指标中看到的那样，存在问题。GPU 以 787 M 周期/秒的速度处于活动状态，但片段着色仅以 600 M 周期/秒的速度处于活动状态。如果我们不受 CPU 限制，并且没有达到垂直同步，这是一个好兆头，我们有一个泡沫要破灭。它还说明，当 Vertex Compute 周期猛增时，Fragment 下降。此下降是阈值 + Bloom 模糊通道。

### 你如何获得这些硬件统计信息？

对于 Arm Mali，有这个[GitHub 链接](https://github.com/ARM-software/HWCPipe)。 Vulkan Samples 框架可以利用这个库来实时读取硬件计数器——确实非常棒。这些是[Arm Mobile Studio](https://www.arm.com/products/development-tools/graphics/arm-mobile-studio) 会给你的计数器 。

### 异步

![ ](https://community.arm.com/resized-image/__size/1040x0/__key/communityserver-blogs-components-weblogfiles/00-00-00-20-66/Async-compute-image3-_2800_final_2900_.jpg)

在这里，我们使用 async，它使我们能够破除泡沫。最后，我们看到了一个漂亮的、完全饱和的 Fragment 队列。

我们在这里获得不错收益的主要原因是我们现在可以并行运行两件事：

- 下一帧的阴影贴图 (FRAGMENT)
- 绽放（计算）

渲染阴影贴图是非常受光栅化限制的，即大量使用固定功能硬件，并且着色器核心大多是摆弄大拇指。这是我们注入一些计算工作负载的最佳时机。顶点工作负载在这里也能很好地工作，但我们不一定有足够的顶点着色工作来保持 GPU 忙碌。将一些片段工作转移到计算上是有意义的。

在这个特定的示例中，我们在[Mali-G77 GPU](https://developer.arm.com/ip-products/graphics-and-multimedia/mali-gpus/mali-g77-gpu)上获得了约 5% 的 FPS 增益，但这些结果非常特定于内容。需要注意的是，即使 Fragment 周期上升，性能也不会线性扩展，因为 Vertex 和 Fragment 仍然共享相同的着色器核心。通过具有活动周期，这仅意味着如果着色器核心上有空闲线程，则 GPU 已准备好立即开始分派工作。着色器核心调度程序可以填充活动中的任何下降。

### 技术

这里的想法是要意识到，如果没有管道，我们可以利用多个 VkQueue 的力量召唤一个管道存在。因此，我们不只是在做*异步计算*，我们也在做*异步图形*。

### 实施细则

该技术将利用一些想法：

- 可以在 Arm Mali 上使用队列优先级，较高优先级的队列可以抢占较低优先级的队列。我们可以感谢 VR 使这个功能成为现实。
- 队列打破了 Vulkan API 中的依赖链。

要解释队列如何分解依赖链，我们必须首先了解屏障在 Vulkan 中的工作原理。管道屏障将所有命令分为两部分，之前的和之后的。然后根据舞台掩码对这两半进行排序。信号量也以类似的想法运行，当前面的一切都完成时，信号量就会发出信号。等待意味着信号量在信号量上被阻塞后的一切，受阶段掩码的限制。

FRAGMENT → COMPUTE → FRAGMENT 障碍会造成无法避免管道气泡的情况。Barriers 只影响单个 VkQueue 内的排序。这里的关键是将帧分成两部分，并改为管道：

- 渲染主通道所需的所有工作 → VkQueue `#1`（低优先级）
- VkQueue `#1` 中的信号量，在 VkQueue `#0` 中等待
- 主渲染通道 + 呈现之后的所有工作 → VkQueue `#0`（更高的优先级）

在这个方案中，我们从未在 VkQueue `#1` 中观察到可怕的 FRAGMENT → COMPUTE 屏障，所以当 VkQueue `#0` 忙于完成呈现的帧时，VkQueue `#1` 可以愉快地通电并开始渲染下一帧。这样我们就可以实现正确的流水线操作。

最后一个技巧是使用队列优先级。VkQueue` #0` 需要比 `#1` 具有更高的优先级，因为队列 `#0` 总是更接近于拥有一个完整的帧，我们真的不希望队列` #`1 阻止` #0` 工作。如果发生这种情况，我们就有可能错过 V-Blank。

队列优先级必须在 Vulkan 中预先声明。这是在设备创建期间完成的：

```cpp
VkDeviceCreateInfo device_info = { VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO };
VkDeviceQueueCreateInfo queue_info = { VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO };
device_info.queueCreateInfoCount = 1;
device_info.pQueueCreateInfos = &queue_info;
queue_info.queueFamilyIndex = 0; // 使用 vkGetPhysicalDeviceQueueFamilyProperties 查询
静态常量浮点优先级 [] = { 1.0f, 0.5f };
queue_info.pQueuePriorities = 优先级；
queue_info.queueCount = 2; // 使用 vkGetPhysicalDeviceQueueFamilyProperties 排队
vkCreateDevice(gpu, &device_info, nullptr, &device);
vkGetDeviceQueue(device, 0, 0, &high_prio_queue);
vkGetDeviceQueue(device, 0, 1, &normal_prio_queue);
```

### 如果我改为跨帧重新排序提交怎么办？

这当然是可能的，但这意味着保留帧以便可以进行重新排序，这通常会使输入延迟增加一帧。这对于交互式应用程序（例如游戏）来说是不可取的。

## 计算工作负载的缺点

并不是说，知道以上就可以立马着手开始计算了。在此之前还需要考虑一些问题。 假设计算线程（compute thread）和片段线程（fragment thread）的工作量一样，由于下面的原因，通常**片段线程（fragment thread）会比计算线程（compute thread）效率更高**：【】【】这里因果关系不是很明确。【】【】】

- 帧缓冲压缩丢失（Loss of framebuffer compression）：对于存储图像，AFBC（Arm 帧缓冲压缩https://www.arm.com/why-arm/technologies/graphics-technologies/arm-frame-buffer-compression）会丢失，这意味着带宽受到的打击比应有的要大一些。
- 事务消除损失（Loss of transactional elimination）：另一个带宽节省功能是消除冗余图块写回，它不能与存储图像一起使用。
- 片段着色器的间接饥饿（Indirect starvation of fragment shaders）：作者很早提到 VERTEX 和 COMPUTE 工作负载使用相同的硬件队列时，若 COMPUTE 工作负载很多，会导致 VERTEX 没有可用的【数据 OR 计算资源？????】，继而导致流水线上的下一个环节，FRAGEMENT 工作任务也没法处理。

## 最佳实践与结论

也推荐看到这里的同学，看看这篇关于图像计算处理的文章[《Arm Mali GPU Best Practices Developer Guide》][https://developer.arm.com/documentation/101897/v2-2/Compute/Image-processing?_ga=2.6604722.1288974174.1624198760-383027749.1605017916]，不要使用计算来处理由片段着色生成的图像。这样做会造成向后依赖，从而导致气泡。如果片段着色器输出被后续渲染通道的片段着色器消耗，则渲染通道会更干净地通过管道。

这项研究的目的是证明了：使用 Vulkan API 是多么幸运，因为 OpenGL ES API 没有多队列的概念。

---

本文的示例展示了，有多种方式充分发挥计算着色器的能力。但使用前需要深入思考并评估预期结果，异步计算可以榨取最后那百分之几的性能，快来试试吧！

# 参考

- https://developer.arm.com/solutions/graphics-and-gaming/developer-guides/learn-the-basics/tile-based-rendering/single-page
- https://community.arm.com/developer/tools-software/graphics/b/blog/posts/using-asynchronous-compute-on-arm-mali-gpus

