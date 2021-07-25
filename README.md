# gpu

GPU的核心技术。那要从GPU RTL搞起，梳理出一套知识，给出硬件specs，performance guide等等。怎么设计GPU RTL那一套理论，是各家的专利(要塞进去两个pipeline，Computing, rendering。

硬件GPU跟 Driver，OS那一套「互动机制」，(没有确切的notes可以看，只能自己搜，Intel开源了相关的GPU Driver runtime, compiler source等，学下来那一套，至少对于目前的圈子完全是够的，剩下就是GPU RTL, CUDA kernel这种东西，各种software-hardware co-design framework，manager等等，然后，就是shading那一套理论，也要好好学，至少要学到能够定位game engine的bound，给出profiling report, 整个这么一圈学下来，嗯，emm，算差不多了，哦，GPU CUDA的sass也可以学着hack一把，写写汇编。这么一波下来，嗯，差不多了

 

## Render

- 2021-07-25 [Using asynchronous compute on Arm Mali GPUs: A practical sample](./render/UsingAsynchronousComputeOnArmMaliGPUs:APracticalSample.md)

## Compute
