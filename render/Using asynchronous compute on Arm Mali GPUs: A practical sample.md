# 在Arm Mali GPU上使用异步计算：附实践代码

- 标题：Using asynchronous compute on Arm Mali GPUs: A practical sample
- https://community.arm.com/developer/tools-software/graphics/b/blog/posts/using-asynchronous-compute-on-arm-mali-gpus
- 作者：Hans-Kristian Arntzen
- 时间：June 16, 2021

异步计算技术确实是效果拔群的性能优化杀器，但难于在项目中落地。异步计算始于上一代[控制台硬件（Console Hardware），就是游戏主机时代](https://www.retroreversing.com/hardware/consoles)，在后来的现代图形 API（如 Vulkan 和 D3D12）上逐步体现，目前异步计算已是图形程序员的常用工具。

本文将展示一个新的 Vulkan 示例，该示例代码也已更新到了 Github KhronosGroup 下的 [Vulkan-Sample](https://github.com/KhronosGroup/Vulkan-Samples) 仓库中，其演示了如何使用异步计算，感兴趣的同学也可以看看这篇 [《使用异步计算技术榨干GPU资源》](https://github.com/KhronosGroup/Vulkan-Samples/blob/master/samples/performance/async_compute/async_compute_tutorial.md)。

严格来说，"异步计算"（Async compute）本身不一种技术，它只是通过一次同时向 GPU 提交多个命令流（multiple streams of commands）来有效利用现代 GPU 可用的硬件资源的策略。咱们后文会具体展开来讲，因为把异步计算用起来，确实需要一定的专业知识。

> 本文的代码示例是基于我 [2018 年博客文章](https://community.arm.com/developer/tools-software/graphics/b/blog/posts/using-compute-post-processing-in-vulkan-on-mali) 的代码改进的，改进后的代码证明了基于计算的后处理可以带来性能提升。
