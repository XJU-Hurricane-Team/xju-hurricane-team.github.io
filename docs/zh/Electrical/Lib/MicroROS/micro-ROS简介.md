# micro-ROS 简介

## 概述

在现代机器人与智能装备开发中，通常采用 **"上位机（MPU/工控机）+ 下位机（MCU 单片机）"** 的异构计算架构：上位机运行 [ROS 2](https://docs.ros.org/)，负责高算力的激光 SLAM、视觉感知、路径规划与行为决策；而下位机（如意法半导体的 [STM32 系列 32 位微控制器](https://www.st.com/en/microcontrollers-microprocessors/stm32-32-bit-arm-cortex-mcus.html)）则负责强实时性的电机闭环控制、编码器脉冲采集、IMU 姿态解算与底层安全传感器触发。

在传统开发模式中，STM32 与 ROS 2 上位机之间隔着一道天然的"通信鸿沟"——开发者不得不手写串口打包解包协议，并在上位机额外编写"串口转 ROS 话题"的桥接节点。[micro-ROS](https://micro.vulcanexus.org/) 的出现正是为了彻底消除这一鸿沟，将标准 ROS 2 生态无缝延伸至微控制器内部。

由于战队主要使用的是STM32上的开发，所以笔者接下来的文章都围绕着 `STM32 + micro-ROS` 展开。

## micro-ROS 是什么？

[**micro-ROS**](https://micro.vulcanexus.org/docs/overview/features/) 是 [ROS 2](https://docs.ros.org/) 官方针对资源受限微控制器推出的嵌入式扩展框架。由于像 [STM32F4 / STM32H7](https://www.st.com/en/microcontrollers-microprocessors/stm32-high-performance-mcus.html) 这类微控制器通常仅有几十 KB 到几百 KB 的 RAM（[micro-ROS 硬件要求与支持列表](https://micro.vulcanexus.org/docs/overview/hardware/)），无法直接运行标准桌面端 ROS 2 庞大且吃内存的完整 [eProsima Fast DDS](https://fast-dds.docs.eprosima.com/) 协议栈，micro-ROS 采用了一种极其精巧的 **"客户端-代理（Client-Agent）"** 协同架构与轻量化软件栈。

### Client-Agent 协同架构

- **下位机端（micro-ROS Client，运行于 STM32）**：在单片机上运行轻量级的客户端程序，通过串口、USB 或以太网与上位机交互。
- **上位机端（**[**micro-ROS Agent**](https://github.com/micro-ROS/micro-ROS-Agent)**，运行于 Linux 工控机）**：作为桥接代理进程，一方面通过轻量级协议接收来自下位机的请求，另一方面代表下位机在上位机完整的 DDS 全局数据空间中创建并维护对应的节点、发布者与订阅者实体（[eProsima Micro XRCE-DDS 架构总览](https://micro-xrce-dds.docs.eprosima.com/en/stable/introduction.html)）。

### 四层模块化软件栈（STM32 端）

1. **实时操作系统 / 硬件抽象层**：原生支持 [FreeRTOS](https://www.freertos.org/)、[Zephyr RTOS](https://www.zephyrproject.org/)、[Apache NuttX](https://nuttx.apache.org/) 等主流开源实时操作系统，也可在裸机环境下运行，并通过官方工具包 [micro_ros_stm32cubemx_utils](https://github.com/micro-ROS/micro_ros_stm32cubemx_utils) 与 [STM32CubeMX](https://www.st.com/en/development-tools/stm32cubemx.html) 及 [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html) 工程深度集成。
2. **极度资源受限中间件层（**[**eProsima Micro XRCE-DDS**](https://micro-xrce-dds.docs.eprosima.com/en/stable/introduction.html)**）**：遵循 DDS-XRCE 国际标准，专门为微控制器设计，采用静态内存池机制，在完成初始化后可实现**运行期零动态内存分配**，彻底杜绝嵌入式常见的堆内存碎片与内存泄漏问题（[micro-ROS 架构特性](https://micro.vulcanexus.org/docs/overview/features/)）。
3. **ROS 2 客户端支持库与 C 语言接口层（rcl + rclc）**：直接复用标准 ROS 2 的底层客户端支持库 rcl，并针对微控制器开发了纯 C 语言的客户端库 [rclc](https://github.com/ros2/rclc)，不仅省去了 C++ 标准库的高昂闪存与内存开销，还提供了专为实时嵌入式系统定制的确定性执行器（rclc_executor）。
4. **用户应用层**：开发者在 STM32 中直接调用熟悉的 ROS 2 风格 API 编写传感器采集与电机控制业务代码。

## micro-ROS 在 STM32 中做什么

将 micro-ROS 嵌入 STM32 后，单片机不再是一个被动收发字节流的"外设从机"，而是直接成为 ROS 2 分布式计算图中的**一等公民（First-Class ROS 2 Node）**。它在 STM32 中主要承担以下核心任务：

- **原生发布与订阅标准或自定义话题**：STM32 可以直接以标准 ROS 2 消息格式向外发布高频传感器与里程计数据，或直接订阅上位机发出的速度控制指令，无需任何中间协议转译。
- **提供服务端/客户端调用与动作管理**：支持同步/异步的请求-响应服务以及带反馈的动作状态机控制。
- **节点图自发现与生命周期管理**：STM32 内部的节点能够直接被上位机的 ros2 node list、ros2 topic echo、ros2 topic hz、rqt_graph 等标准命令行与可视化工具检索和监控，并支持节点生命周期状态转换（配置、激活、停用、清理）（[micro-ROS 核心特性](https://micro.vulcanexus.org/docs/overview/features/)）。
- **高精度跨设备时间同步**：内置基于 NTP 机制的客户端-代理硬件时钟对齐能力，使 STM32 在采集 IMU 或里程计数据时可以直接打上与上位机严格同步的微秒/毫秒级系统时间戳，极大提升多传感器融合（如激光雷达畸变校正）的精度。
- **硬实时确定性任务调度**：借助 [rclc](https://github.com/ros2/rclc) 执行器与 [FreeRTOS](https://www.freertos.org/) 任务结合，可精确控制多个订阅回调与定时器的触发顺序和优先级，确保底层 PID 控制环路的确定性周期执行。

## micro-ROS 的优势

在机器人底层开发中，过去常见的通信方式主要有两种：一是**传统裸机自定义串口协议**（如自定义帧头 0xAA 0x55 + 数据长度 + 结构体字节流 + CRC 校验），二是 ROS 1 时代的 [rosserial 协议](https://wiki.ros.org/rosserial/Overview/Protocol)。将 [micro-ROS](https://micro.vulcanexus.org/docs/overview/features/) 与这两者对比，其工程优势主要体现在以下六个核心维度：

### 通信方案全方位对比表

| **对比维度**         | **传统自定义串口通信（裸串口协议）**               | **ROS 1 传统桥接（rosserial）**                          | **micro-ROS（基于 Micro XRCE-DDS）**                         |
| -------------------- | -------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| **系统角色定位**     | 透明字节流从机，对 ROS 完全不可见                  | 串口话题转发终端，非真正独立的 ROS 2 节点                | **原生 ROS 2 节点**，在计算图中一等公民可见                  |
| **协议与序列化**     | 需手动定义帧头、帧尾、字节序、大小端与 CRC 校验    | 专有封包格式（0xff 同步头 + 16 位话题 ID + 校验和）      | 遵循国际标准 **DDS-XRCE 与 CDR 序列化**，自动生成代码        |
| **上位机开发工作量** | **高**：必须手写并维护上位机串口解析与话题转发节点 | **中**：依赖特定 Python/C++ 桥接节点，且仅原生面向 ROS 1 | **极低**：直接启动通用 **micro-ROS Agent**，零业务转译代码   |
| **通信模式丰富度**   | 仅原始字节流（需自行区分指令帧与状态帧）           | 仅支持基础 Topic 与简单 Service，不支持生命周期与 QoS    | 完整支持 **Topic、Service、Action、Parameter、Lifecycle**    |
| **QoS控制**          | 无内置机制（丢包重传与流量控制需全部手写）         | 仅依赖串口校验和丢弃损坏包，无分级可靠性策略             | 原生支持 **DDS QoS 策略**                                    |
| **物理传输层解耦**   | 与 UART 硬件逻辑深度绑定，更换通信接口需重写收发层 | 主要局限于 UART 字符设备与简单 TCP 套接字                | **传输层高度抽象**：无缝切换 UART（DMA/中断）、USB CDC、UDP/以太网、Wi-Fi、CAN |
| **内存与实时性保障** | 依赖开发者手写环形缓冲区水平                       | 易出现串口缓冲区溢出、同步丢失与时间漂移                 | **静态内存池（零动态分配）** + [rclc](https://github.com/ros2/rclc) 确定性实时调度 + 硬件级时钟同步 |

### 相比传统串口通信的五大核心优势详解

1. **彻底告别"双端重复造轮子"，大幅降低维护成本**
    - **传统串口痛点**：每当需要新增一个传感器数据或修改一个控制字段时，开发者必须先修改 [STM32](https://www.st.com/en/microcontrollers-microprocessors/stm32-32-bit-arm-cortex-mcus.html) 端的 C 语言打包结构体、数组偏移量和校验计算；随后又必须同步修改 Linux 上位机端的串口读取、状态机解包与 ROS 话题再发布代码。一旦双端字节对齐或浮点精度不一致，就会引发极难排查的解析错乱。
    - **micro-ROS 优势**：采用统一接口定义语言。无论是标准消息还是自定义消息，双端共享同一套数据结构定义，由底层 Micro CDR 自动完成序列化与反序列化，上位机只需运行通用的 **micro-ROS Agent**，完全省去了上位机串口转译节点的编写与维护。
2. **从"黑盒透传"升级为"全链路透明可观测"**
    - **传统串口痛点**：串口线上传输的是十六进制字节流，调试时往往需要拔掉上位机连线、挂接串口助手或逻辑分析仪抓包，或者等上位机解析节点跑通后才能看到数据；一旦通信中断，很难快速定位是单片机死机、串口阻塞还是上位机解析出错。
    - **micro-ROS 优势**：STM32 节点直接暴露在 **ROS 2** 网络中。开发者在上位机终端即可直接使用 `ros2 node info` 查看单片机暴露的所有接口，用 `ros2 topic hz` 实时测量 STM32 的真实发送频率，用 `ros2 param` 动态在线调参（例如在线调节 STM32 内部的电机 PID 参数），调试体验与调试本地 Linux 节点完全一致。
3. **提供工业级 QoS（服务质量）与多语义通信机制**
    - **传统串口痛点**：同一根串口线上既有高频但允许偶发丢包的传感器数据（如 200Hz IMU 数据），又有低频但绝对不能丢失的关键控制指令（如急停、模式切换）。裸串口很难在同一通道上优雅地实现差异化可靠性保障。
    - **micro-ROS 优势**：直接继承了 DDS 的 **QoS策略**与多通信原语。在同一个物理串口（或虚拟串口/网口）通道上，高频遥测数据可配置为 Best Effort（尽力而为，追求最低延迟），而关键参数配置或急停指令则可使用 Reliable（可靠传输，自动握手机制与重传）或直接采用带应答的 **Service / Action** 机制。
4. **底层物理传输介质与上层业务逻辑完全解耦**
    - **传统串口痛点**：早期使用普通 UART 波特率开发，后期发现带宽不够想升级为 USB 虚拟串口（USB CDC）或以太网 UDP 时，往往需要大面积重构收发状态机代码。
    - **micro-ROS 优势**：在 [micro_ros_stm32cubemx_utils](https://github.com/micro-ROS/micro_ros_stm32cubemx_utils) 中，传输层被严格抽象为 4 个标准钩子函数（open、close、write、read）。STM32 工程开箱即支持 **UART + DMA 循环接收**、**UART 中断**、**USB CDC 虚拟串口** 以及 **基于 LwIP 的 Ethernet UDP**。当硬件接口从串口升级为 USB 或百兆以太网时，STM32 上层的节点、话题与控制代码**一行都不用改**。
5. **专为嵌入式设计的静态内存安全与高精度时间同步**
    - **传统串口痛点**：串口接收到上位机发来的指令或上传传感器数据时，若简单使用上位机接收时刻作为时间戳，会引入严重的串口传输延迟与操作系统调度抖动，导致建图与导航算法精度下降。
    - **micro-ROS 优势**：一方面通过静态内存池消除了动态 malloc/free 带来的运行期崩溃隐患；另一方面内置毫秒/微秒级时钟同步协议，让 STM32 直接在硬件中断或实时采样瞬间打上与上位机对齐的精确时间戳。

## 适用边界与工程选型建议

尽管 **micro-ROS** 优势显著，但在实际 **STM32** 项目选型中，仍需遵循合理的架构边界：

- **最适合使用 micro-ROS 的场景**：
    - 硬件选型为中高配 32 位单片机（如 [STM32F4 / STM32F7 / STM32H7](https://www.st.com/en/microcontrollers-microprocessors/stm32-high-performance-mcus.html) 或 [STM32L4](https://www.st.com/en/microcontrollers-microprocessors/stm32-ultra-low-power-mcus.html) 系列，建议可用 RAM 在 64 KB 以上，FreeRTOS 分配给 micro-ROS 任务的栈空间大于 10 KB，参见 [micro_ros_stm32cubemx_utils 说明](https://github.com/micro-ROS/micro_ros_stm32cubemx_utils)）。
    - 移动机器人底盘控制（差速、麦轮、阿克曼转向）、多路编码器里程计解算、IMU 姿态融合、机械臂关节驱动、电源管理监控以及分布式传感器节点。
- **不适合或需谨慎使用的场景**：
    - **极低端超小内存单片机**：如仅有几 KB 到 20 KB RAM 的低端 8 位/32 位芯片（如 STM32F103C6/C8 等极小容量型号），此时精简的裸机自定义串口或 CAN 协议更为轻量。
    - **超大吞吐量原始媒体流**：切勿在 STM32 上通过 micro-ROS 传输未压缩的图像或高密度激光点云，这类大数据包应直接接入上位机处理，STM32 仅负责传输紧凑的状态量与控制指令。
