---
title: "NCCL"
collection: parallel-computing
permalink: /parallel-computing/parallel-computing-02
excerpt: ' '
date: 2024-07-02
citation: 'Joe-Bi. (2024). &quot;NCCL.&quot; <i>GitHub Joe-Bi of Bugs</i>'
---
   
NCCL（NVIDIA Collective Communications Library）是由NVIDIA开发的一种用于高性能GPU集群的通信库。
它旨在提供高效的GPU间通信和协作能力，以加速分布式深度学习和其他GPU密集型计算任务。

NCCL支持在多个GPU之间进行并行计算和通信。它可以在多个GPU之间实现高效的数据传输和同步，以利用集群中的所有GPU资源。
被广泛用于分布式深度学习训练中，特别是在使用多个GPU进行模型训练时。
它提供了一致的接口和通信原语，使不同GPU之间的数据交换和同步变得简单和高效。
NCCL支持跨多个节点的通信，使得在GPU集群中进行分布式计算和通信变得容易。
它提供了跨节点的通信原语，如跨节点的点对点通信、全局同步和归约操作。

NCCL[官网](https://developer.nvidia.com/nccl)

对NCCL研究比较深入的文章来自CSDN的KIDGINBROOK,他撰写了[一系列的文章](https://blog.csdn.net/kidgin7439/category_11998768.html)，可以好好研读学习。

另外这里有一篇对NCCL的流程概要分析，[NCCL简介及其流程分析](https://www.ctyun.cn/developer/article/464868577030213)


