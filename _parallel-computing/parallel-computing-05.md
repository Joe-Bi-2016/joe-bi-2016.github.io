---
title: "InfiniBand和NIC在RDMA中工作原理和扮演的角色"
collection: parallel-computing
permalink: /parallel-computing/parallel-computing-05
excerpt: ' '
date: 2025-02-27
citation: 'Joe-Bi. (2025). &quot;InfiniBand和NIC在RDMA中工作原理和扮演的角色.&quot; <i>GitHub Joe-Bi of blog</i>'
---

# 一、InfiniBand在RDMA中的职责
InfiniBand是一种高性能、低延迟的网络通信协议和硬件架构，专为大规模并行计算和数据中心设计。在 RDMA场景中，InfiniBand的核心职责包括：

### 1. 提供高带宽、低延迟的传输通道
- &zwnj;**物理层支持**&zwnj;：通过专用InfiniBand网络（如QSFP光纤）实现高吞吐量（可达400 Gbps）和微秒级延迟  
- &zwnj;**协议栈优化**&zwnj;：InfiniBand 协议栈（包括传输层、网络层、链路层）直接集成硬件加速，绕过传统 TCP/IP 协议栈的开销  

### 2. 支持零拷贝数据传输
- &zwnj;**绕过 CPU 和内核**&zwnj;：InfiniBand的HCA（主机通道适配器）直接访问应用程序内存，无需 CPU 参与数据搬运  
- &zwnj;**内存注册（Memory Registration）**&zwnj;：在通信前，HCA 将应用程序内存注册为可被远程访问的缓冲区（MR，Memory Region），确保安全性和权限控制  

### 3. 管理通信协议
- &zwnj;**队列对（QP，Queue Pair）**&zwnj;：通过发送队列（SQ）和接收队列（RQ）管理 RDMA 操作，支持多种传输模式（可靠/不可靠、连接/非连接）  
- &zwnj;**完成队列（CQ，Completion Queue）**&zwnj;：通知应用程序数据传输完成状态  

### 4. 硬件级流控制与拥塞管理
- &zwnj;**基于信用的流控**&zwnj;：防止接收端缓冲区溢出  
- &zwnj;**自适应路由（Adaptive Routing）**&zwnj;：动态选择最优路径，避免网络拥塞  

---

# 二、NIC（网络接口卡）在RDMA中的职责
在 RDMA场景中，NIC通常指支持RDMA的智能网卡（如 RoCE（RDMA over Converged Ethernet）或 iWARP 网卡）。其核心职责包括：

### 1. 在以太网上实现RDMA
- &zwnj;**RoCE（RDMA over Converged Ethernet）**&zwnj;：  
  - RoCEv1：基于以太网链路层（L2），依赖无损网络（需 PFC 等流控协议）  
  - RoCEv2：基于 UDP/IP（L3），支持路由和跨子网通信  
- &zwnj;**iWARP**&zwnj;：基于 TCP/IP 协议栈，兼容传统网络，但性能低于 RoCE 和 InfiniBand  

### 2. 硬件加速RDMA操作
- &zwnj;**零拷贝传输**&zwnj;：NIC直接读写应用程序内存，避免数据在用户态和内核态间复制  
- &zwnj;**协议卸载（Offload）**&zwnj;：将传输层协议（如TCP/IP或 RDMA协议）的处理卸载到NIC硬件，减少CPU负载  

### 3. 管理队列与内存访问
- &zwnj;**队列对（QP）与工作请求（WR）**&zwnj;：NIC维护发送/接收队列，处理应用程序提交的工作请求（如 RDMA Read/Write）  
- &zwnj;**内存保护与地址转换**&zwnj;：通过内存注册和物理地址转换表（PTE），确保远程访问的安全性  

### 4. 网络流量优化
- &zwnj;**拥塞控制**&zwnj;：在RoCE中实现DCQCN（数据中心量化拥塞通知）等算法  
- &zwnj;**优先级标记（PFC）**&zwnj;：在以太网中划分优先级流量，确保无损传输  

---

# 三、InfiniBand vs. RDMA-enabled NIC 的对比

| 特性                   | InfiniBand                          | RDMA-enabled NIC（如 RoCE）         |
|------------------------|-------------------------------------|--------------------------------------|
| 网络架构               | 专用 InfiniBand 网络（HCA + 交换机） | 基于以太网（兼容现有基础设施）        |
| 协议栈                | 原生 InfiniBand 协议栈              | RDMA over Ethernet（RoCE）或 TCP/IP（iWARP） |
| 延迟                  | 微秒级（1~3 μs）                   | 微秒到亚微秒级（依赖网络配置）         |
| 部署成本              | 高（需专用硬件）                    | 低（利用现有以太网）                  |
| 适用场景              | HPC、超算、金融交易                 | 云数据中心、存储网络（如 NVMe over Fabrics） |
| 流控与拥塞管理        | 硬件级流控（基于信用）              | 依赖 PFC 和 ECN（需配置无损网络）      |

---

# 四、协同工作流程示例（以InfiniBand为例）
1. &zwnj;**初始化**&zwnj;：  
   - 应用程序注册内存区域（MR），创建队列对（QP）  
   - HCA 将 MR 的物理地址和访问权限同步到远程节点  

2. &zwnj;**数据传输**&zwnj;：  
   - 应用程序提交工作请求（WR）到本地 HCA 的发送队列（SQ）  
   - HCA 生成 RDMA 数据包，通过 InfiniBand 网络发送到目标 HCA  
   - 目标 HCA 直接将数据写入远程内存，完成后通过完成队列（CQ）通知应用程序  

3. &zwnj;**完成通知**&zwnj;：  
   - 双方 HCA 通过中断或轮询方式通知应用程序操作完成  

---

# 五、Infiniband与NIC关系

### 1. 广义上的联系
- &zwnj;**硬件角色**&zwnj;：两者都是网络通信的硬件组件，负责连接计算机与网络，管理数据传输‌
- &zwnj;**支持RDMA**&zwnj;：InfiniBand和某些NIC（如支持RoCE的智能网卡）均支持RDMA（远程直接内存访问），实现零拷贝和低延迟传输
- &zwnj;**队列模型**&zwnj;：均使用队列对（QP）管理数据传输请求（发送队列SQ和接收队列RQ）

### 2. 协同场景
- &zwnj;**InfiniBand的专用NIC**&zwnj;：  
  InfiniBand网络中使用的网卡称为 HCA（Host Channel Adapter），本质上是一种专用NIC，专为InfiniBand协议设计
- &zwnj;**以太网中的RDMA**&zwnj;：  
  普通以太网卡通过支持 RoCE（RDMA over Converged Ethernet）或 iWARP 协议，也能实现类似InfiniBand的RDMA功能，但需依赖以太网协议栈

---

### 3. 核心区别‌

| 特性                   | InfiniBand                          | 传统NIC（以太网/RDMA-enabled NIC）  |
|------------------------|-------------------------------------|-------------------------------------|
| 网络架构               | 专用网络协议和硬件（HCA + 交换机）   | 基于以太网（兼容现有网络基础设施）   |
| 协议栈                | 原生InfiniBand协议（物理层到传输层） | 基于TCP/IP（iWARP）或UDP/IP（RoCE） |
| 硬件设计              | 专用HCA芯片，集成RDMA硬件加速        | 智能网卡（如NVIDIA ConnectX）支持RoCE或iWARP |
| 延迟                  | 极低（微秒级，1~3 μs）             | 较低（依赖配置，通常5~10 μs）        |
| 部署成本              | 高（需专用交换机、线缆和HCA）       | 低（利用现有以太网基础设施）         |

---

### 4. 关键差异分析‌

#### 1). 协议栈与网络架构
- &zwnj;**InfiniBand**&zwnj;：  
  端到端专用协议栈，完全绕过TCP/IP，支持原生RDMA操作（如SEND/WRITE/READ）  
- &zwnj;**NIC（以太网）**&zwnj;：  
  依赖以太网协议栈，需通过RoCE或iWARP封装RDMA协议，引入额外包头开销

#### 2). 硬件实现
- &zwnj;**InfiniBand HCA**&zwnj;：  
  硬件直接处理队列对（QP）、内存注册（MR）和传输协议，支持细粒度流控  
- &zwnj;**RDMA-enabled NIC**&zwnj;：  
  通过硬件卸载TCP/IP或RoCE协议栈实现RDMA，依赖软件驱动和操作系统支持

#### 3). 性能与适用场景
- &zwnj;**InfiniBand**&zwnj;：  
  适合超低延迟场景（高频交易、超算），但部署成本高‌
- &zwnj;**以太网NIC**&zwnj;：  
  适合云数据中心和存储网络（如NVMe over Fabrics），依赖无损网络配置‌

---

### 5. 选择建议
- &zwnj;**选择InfiniBand**&zwnj;：需极致性能（延迟/带宽）且预算充足时优先考虑  
- &zwnj;**选择RDMA-enabled NIC**&zwnj;：需平衡成本与性能，且复用现有以太网基础设施时使用  

---

### 6. 小结
   1. &zwnj;**本质关系**&zwnj;：  
   InfiniBand的HCA是专用NIC，以太网NIC通过协议扩展支持RDMA  
   2. &zwnj;**未来趋势**&zwnj;：  
   以太网带宽提升（800G）可能缩小性能差距，但InfiniBand仍主导超低延迟领域  

---

#  <font color="red">六、角色与形象理解</font>

   可以将InfiniBand类比成高速公路网和交通法则，NIC类比成运输工具来理解InfiniBand和NIC。

### 1. InfiniBand：高速公路网与交通法则
- &zwnj;**物理网络结构**&zwnj;  
  通过QSFP光纤链路（EDR/HDR标准）构建高带宽通道，类似高速公路网的物理路基
- &zwnj;**协议与规则**&zwnj;  
  - 流量控制：基于信用的流控机制（类似交通信号灯系统）
  - 虚拟通道管理：划分多个VL（Virtual Lane）实现优先级隔离‌
- &zwnj;**网络管理**&zwnj;  
  子网管理器动态优化路由（类似智能交通调度中心）

### 2. NIC：运输工具与调度系统
- &zwnj;**数据搬运执行**&zwnj;  
  通过HCA的DMA引擎实现零拷贝传输（卡车直接装卸货物）
- &zwnj;**硬件加速能力**&zwnj;  
  卸载协议处理（如RoCEv2封装），降低CPU负载（独立动力引擎）
- &zwnj;**智能调度**&zwnj;  
  管理QP队列对和流量优先级（运输工具的实时路径规划）

---

### 3. 类比的补充说明‌

| 类比维度        | 传统交通系统局限              | InfiniBand/NIC增强特性           |
|----------------|----------------------------|---------------------------------|
| 扩展性          | 路网受物理限制                | 支持万级节点动态扩展（可编程逻辑）   |
| 安全性          | 依赖外部监管                 | 内存注册（MR）与密钥认证（硬件级防护）|
| 协同性          | 车辆与道路系统独立            | 协议栈与硬件深度集成（如QP-CQ联动）|

---

### 4. 典型场景验证‌

#### RDMA Write操作流程（类比物流系统）
   1). &zwnj;**订单生成**&zwnj;  
   应用程序提交WR（工作请求），指定目标内存地址  
   2). &zwnj;**路径规划**&zwnj;  
   子网管理器选择最优VL通道（智能导航）  
   3). &zwnj;**货物运输**&zwnj;  
   HCA直接分片发送数据包（集装箱卡车队）‌  
   4). &zwnj;**签收反馈**&zwnj;  
   接收端生成CQE完成通知（电子回单）‌

---

#  七、关键挑战与解决方案
- &zwnj;**网络拥塞**&zwnj;：  
  - InfiniBand：自适应路由和基于信用的流控  
  - RoCE：DCQCN 算法 + PFC 优先级划分  
- &zwnj;**内存安全**&zwnj;：  
  - 通过内存注册和密钥（L_Key/R_Key）保护远程内存访问  
- &zwnj;**兼容性**&zwnj;：  
  - iWARP 支持传统TCP/IP，但性能受限；RoCEv2通过UDP/IP实现跨子网通信  

---

# 八、总结
1. InfiniBand 是 RDMA 的“黄金标准”，提供极致性能，但依赖专用硬件  
2. RDMA-enabled NIC（如 RoCE）在以太网上实现 RDMA，平衡性能与成本，更适合大规模数据中心  
3. 两者均通过硬件卸载、零拷贝和队列管理实现高效数据传输，但设计哲学和适用场景不同 
4. 在‌纯InfiniBand组网‌中，‌由于InfiniBand包含HCA硬件模块，已完全替代传统NIC‌，成为数据通信的核心硬件载体‌。用户只需部署HCA即可满足网络连接与RDMA需求，无需额外NIC‌ 
