---
title: "RDMA技术原理与实现浅析"
collection: parallel-computing
permalink: /parallel-computing/parallel-computing-04
excerpt: ' '
date: 2025-02-27
citation: 'Joe-Bi. (2025). &quot;RDMA技术原理与实现浅析.&quot; <i>GitHub Joe-Bi of Bugs</i>'
---
   
# RDMA技术原理与实现解析

## 1. RDMA基本概念  
- &zwnj;**定义**&zwnj;：RDMA（Remote Direct Memory Access）允许计算机直接存取另一台计算机内存的技术，绕过操作系统和CPU介入，减少延迟并提升通信效率。  
- &zwnj;**适用场景**&zwnj;：特别适合大规模并行计算机集群，支持高吞吐量和低延迟的网络通信。  

---

## 2. RDMA核心原理  
- &zwnj;**直接内存传输**&zwnj;：应用程序能够直接执行数据传输，在不涉及到网络软件栈的情况下，数据能够被直接发送到缓冲区或者能够直接从缓冲区里接收，而不需要被复制到网络层。  
- &zwnj;**硬件卸载**&zwnj;：通过RDMA网卡（NIC）直接处理数据传输任务，不需要在内核态与用户态之间做上下文切换，从而解放了内存带宽和CPU周期用于改进应用系统性能，释放CPU算力。  

---

## 3. RDMA核心优势  
- &zwnj;**零拷贝（Zero-copy）**&zwnj;：数据直接从应用缓冲区发送到网络，避免内核与用户空间之间的复制。  
- &zwnj;**内核旁路（Kernel Bypass）**&zwnj;：应用程序在用户态完成通信，减少上下文切换延迟。  

---

## 4. RDMA网络协议实现  
目前，RDMA有三种不同的硬件实现方式，分别是InfiniBand、iWarp和RoCE（RDMA over Converged Ethernet）。其中，InfiniBand是一种专为RDMA设计的网络，从硬件级别保证可靠传输；而RoCE和iWARP都是基于以太网的RDMA技术，支持相应的verbs接口。
  
&zwnj;**InfiniBand**&zwnj;：专为RDMA设计的网络协议，硬件级可靠传输。  
&zwnj;**iWARP**&zwnj; ：通过TCP/IP实现RDMA，兼容传统网络设备 。  
&zwnj;**RoCE（RDMA over Converged Ethernet）**&zwnj;：基于以太网的RDMA实现，分为两个版本：  
  - &zwnj;**RoCEv1**&zwnj;：基于以太网链路层，依赖PFC流控技术。  
  -  &zwnj;**RoCEv2**&zwnj; ：基于UDP/IP协议栈，支持IP路由 。  
  
---

## 5. RDMA工作流程 
### 工作过程
当一个应用执行RDMA读或写请求时，不执行任何数据复制。在不需要任何内核内存参与的条件下，RDMA请求从运行在用户空间中的应用中发送到本地NIC（网卡）。
NIC读取缓冲的内容，并通过网络传送到远程NIC。
在网络上传输的RDMA信息包含目标虚拟地址、内存钥匙和数据本身请求，既可以完全在用户空间中处理（通过轮询用户级完成排列），又或者在应用一直睡眠到请求完成时的情况下通过系统中断处理。
目标NIC确认内存钥匙，直接将数据写入应用缓存中，用于操作的远程虚拟内存地址包含在RDMA信息中。

### 通信过程  
1.  &zwnj;**内存注册**&zwnj; ：应用程序将内存区域注册为RDMA可访问区域，生成唯一内存密钥(Memory Key)。  
2.  &zwnj;**队列管理**&zwnj; ：  
   - 创建队列对（QP，Queue Pair），包含发送队列（SQ）和接收队列（RQ） 。  
   - 完成队列（CQ）用于通知请求处理状态 。  
3.  &zwnj;**数据传输**&zwnj; ：  
   - 发送端NIC直接读取用户缓冲区数据，封装目标地址和内存密钥 。  
   - 接收端NIC验证密钥后，将数据写入目标内存区域 。  

### 操作模式  
-  &zwnj;**单边操作（单端发起）**&zwnj; ：如RDMA Write/Read，仅需发送端明确目标地址 。  
-  &zwnj;**双边操作（两端协同）**&zwnj; ：需接收端预先准备缓冲区 。  

---

### 6. 工作细节
RDMA提供了基于消息队列的点对点通信，每个应用都可以直接获取自己的消息，无需操作系统和协议栈的介入。消息服务建立在通信双方的本地与远端应用之间创建的Channel-IO连接之上。当APP需要通信时，就会创建一条Channel连接，每条Channel的首尾端点是两对QueuePairs（QP）。每对QP由SendQueue（SQ）和ReceiveQueue（RQ）构成，这些队列中管理着各种类型的消息。除了QP描述的两种基本队列之外，RDMA还提供一种队列CompleteQueue（CQ），CQ用来知会用户WQ上的消息已经被处理完。

---

## 7. 技术演进与挑战  
-  &zwnj;**性能瓶颈**&zwnj; ：依赖高速网络硬件（如InfiniBand/100G以太网）和低延迟拓扑 。  
-  &zwnj;**安全风险**&zwnj; ：直接内存访问需严格权限控制，防止越权操作 。  
-  &zwnj;**编程复杂度**&zwnj; ：需手动管理内存注册、连接状态等底层细节 。  

---

通过上述机制，RDMA为高性能计算、分布式存储和AI训练等场景提供了显著的性能提升 。

