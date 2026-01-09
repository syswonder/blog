# 以共享内存优化数据传输的相关调研

时间: 2026/01/09

作者: 王善上


## 1. Dora-rs
Dora（dora-rs）是一个新兴的高性能数据流与 AI 管线中间件框架，旨在为跨语言、跨进程的数据传输提供高效支持。与传统机器人中间件（如 ROS 2）相比，Dora 在优化方面做出了以下努力： 
- **零拷贝传输** ：Dora 以 **Apache Arrow** 内存格式为核心，允许不同语言和进程直接共享数据结构，避免重复的序列化与拷贝操作。 
- **跨语言高效互操作** ：通过统一的 Arrow 格式，Dora 能够在 Rust、Python、C++ 等多种语言之间实现零拷贝数据交换，降低跨语言调用的性能损耗。 

**总结**：Dora 的优化努力主要集中在**零拷贝传输**与**跨语言高效互操作**，通过 Apache Arrow 的统一内存表示，它在性能敏感的 AI 与数据流场景中展现出明显优势，但也因此对用户提出了更高的使用复杂度要求。

### 1.1 生态较小

- **发展时间**：
  - 自 ROS 1 延续而来，ROS 2 已发展多年，经历多个 LTS 版本。而 Dora 项目较新，仍处于快速演进阶段。
- **社区规模**：
  - 在 GitHub 中，Dora 的 star 数量和仓库数量远小于 ROS 2。
  - ROS ：2024 年有 5.3 亿的 ROS 包下载量（ROS 2 已成为主流），超过 1250 家公司使用（可观产业支持），在 Robotics Stack Exchange 提问占比高达 93%（社区活跃度高）。[1]
  - Dora ："Although it is much faster than ROS, it is still unstable and has a rather smaller community." [2]

**结论**：ROS 2 生态规模大、用户基数广、应用场景成熟，是工程与科研中的主流选择。Dora 的生态规模明显小于 ROS 2。


### 1.2. 用户使用方式较为复杂

#### 1.2.1 ROS 2：高层抽象，屏蔽序列化细节

ROS 2 的典型用户使用方式具有以下特点：

- 用户通过 **rclcpp / rclpy 等客户端库**直接操作消息对象。
- 消息由 `.msg` 描述文件定义。
- **序列化 / 反序列化由中间件（DDS）自动完成**。



#### 1.2.2 Dora：基于 Arrow 的显式数据构造与反序列化

Dora 的使用方式与 ROS 2 有本质差异：

- 节点间传输的数据以 **Apache Arrow 内存格式**为核心。
- 用户需要：
  - 显式构造 Arrow Array / RecordBatch
  - 定义 schema
  - 在接收端进行 **Arrow → 目标语言对象的反序列化**
- 即使在跨语言零拷贝场景下，用户仍需理解并操作 Arrow 数据结构

这意味着 Dora 的使用更偏向：对性能极端敏感、可接受更高代码复杂度的用户

### 1.3. 小结与研究动机

| 对比维度 | ROS 2 | Dora |
|--------|------|------|
| 生态规模 | 非常大，成熟 | 较小，新兴 |
| 用户抽象层级 | 高（消息语义） | 低（数据结构/内存语义） |
| 序列化感知 | 用户基本无感 | 用户显式处理 |
| 使用复杂度 | 较低 | 较高 |
| 零拷贝能力 | 受限但易用 | 强但使用成本高 |

**关键观察**：

- Dora 通过 Arrow 实现了高效零拷贝，但**将数据格式与反序列化复杂性暴露给了用户**。
- ROS 2 拥有庞大的生态和良好的易用性，但在零拷贝与非 POD 数据传输方面存在性能瓶颈。

---

## 2. ROS2 Loaned Message

### 2.1. 背景
在 ROS2 中，除了共享内存优化外，官方还提供了一种 **Loaned Message** 机制，用于减少消息传输过程中的内存拷贝。其核心思想是：  
- Publisher 在发送消息时，直接“借用”中间件内部的内存空间。  
- Subscriber 在接收消息时，也直接访问这块内存，而不是再进行一次拷贝。  

这样可以显著降低拷贝开销，提升性能。


### 2.2. Loaned Message 的局限性

#### 2.2.1. 仅支持 POD 类型
- Loaned Message 只能用于 **Plain Old Data (POD)** 类型的消息。  
- POD 类型指的是没有复杂构造函数、析构函数、虚函数的简单结构体。  
- **不支持可变长类型**，例如：
  - `std::vector`
  - `std::string`
  - 其他动态分配的容器  

这意味着对于图像、点云等大数据量场景，Loaned Message 的适用性有限。

#### 2.2.2. 强行转为非 POD 的代价
- 如果用户希望传输非 POD 类型数据（如图像的 `vector<uint8_t>`），需要提前定义一个 **固定长度的消息类型**。  
- 例如：定义一个 `uint8[921600]` 的数组来存储 720P 图像。  
- 这样虽然可以绕过 POD 限制，但带来以下问题：
  - **灵活性差**：必须提前固定大小，无法适应不同分辨率或动态数据长度。  
  - **内存浪费**：即使实际数据较小，也会占用整个固定数组空间。  
  - **维护复杂**：需要为不同数据类型和分辨率定义不同的消息结构。


### 2.3. 总结
ROS2 Loaned Message 是一种官方提供的零拷贝优化机制，但它的局限性主要在于：
- 只能用于 POD 类型消息。  
- 无法直接支持 `vector`、`string` 等可变长数据。  
- 若要支持非 POD 数据，必须提前定义固定长度的消息类型，导致灵活性和可维护性下降。  

在实际应用中，Loaned Message 更适合小型、固定格式的数据，而对于图像等大规模可变长数据，Boost.Interprocess 的共享内存优化方式更为合适。

---


## 3. Boost.Interprocess 简介
Boost.Interprocess 是 Boost 库中的一个模块，主要用于 **进程间通信（IPC）**。它提供了多种机制来实现不同进程之间的数据共享与同步，包括：

- **共享内存 (Shared Memory)**：允许多个进程直接访问同一块内存区域，避免数据拷贝。
- **同步原语 (Synchronization Primitives)**：如 `interprocess_mutex`、`interprocess_semaphore`，保证多进程访问共享资源时的安全。
- **容器与分配器 (Containers & Allocators)**：支持在共享内存中创建 STL 风格的容器（如 `vector`、`map`），并通过专用分配器管理。避免创建后再拷贝进共享内存造成性能损失。

---

## 4. Fast DDS 的共享内存管理
- **基本框架**：对数据块（通常是序列化后的消息）采用引用计数，由读取端调用```return_loan()```来减少引用计数，当引用计数为 0 时释放。
- **对异常退出的处理**： 
  - **端口接收时自检**：对每个订阅者维护一个**端口**，表示该订阅者未处理的消息队列，发布者会往订阅者的端口发送消息的 buffer 描述符，触发```Port.try_push()```，此函数会在推送异常时，调用```recover_blocked_processing()```，检测是否为僵尸端口，如果是则清除该订阅者的全部占用。
  - **占用信息维护**：对每个订阅者维护一个```blocked_processing```列表,表示该订阅者正在占用哪些信息.
  - **僵尸端口判定**：```is_zombie()```：会使用端口文件锁_el（exclusive lock）、_sl（shared lock），锁会在初始时创建，崩溃后锁会被内核释放。"An exclusive port is zombie when it has a "_el" file & the file is not locked"


[1]：Open Robotics, ROS Metrics Report 2024, ROS Discourse. https://discourse.openrobotics.org/t/2024-ros-metrics-report/42354
[2]：Proc. 34. Workshop Computational Intelligence, Berlin, 21.-22.11.2024 175
https://library.oapen.org/bitstream/handle/20.500.12657/95710/proceedings-34-workshop-computational-intelligence-berlin-21-22-november-2024.pdf?isAllowed=y&sequence=1