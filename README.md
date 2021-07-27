# gpu


> 硬件 GPU 跟 Driver 、 OS 那一套「互动机制」，没有确切的笔记可看，只能自己搜，Intel开源了相关的 GPU Driver runtime 、 compiler source 等，学下来那一套，至少对于目前的圈子完全是够的，剩下就是 GPU RTL 、 CUDA kernel 这种东西，各种 software-hardware co-design framework 、 manager等等，然后是 shading 那一套理论，也要好好学，至少要学到能够定位 game engine 的 bound ，给出 profiling report ，整个这么一圈学下来，算差不多了。哦， GPU CUDA 的 sass 也可以学着 hack 一把，写写汇编。这么一波下来，嗯，差不多了。  
> 而 GPU 的核心技术。那要从 GPU RTL 搞起，梳理出一套知识，给出硬件 specs 、 performance guide 等等。怎么设计 GPU RTL 那一套理论，是各家的专利：要塞进去两个 pipeline 、 Computing 、 rendering 。


 

## Render

- 2021-07-25 [Using asynchronous compute on Arm Mali GPUs: A practical sample](./render/UsingAsynchronousComputeOnArmMaliGPUs:APracticalSample.md)

## Compute
