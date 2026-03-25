STM32 是意法半导体 (STMicroelectronics) 推出的基于 ARM Cortex-M 内核的微控制器系列，有着丰富的外设资源和良好的实时性以及社区开发环境。

ARM Cortex系列：

- Cortex-A—面向性能密集型系统的应用处理器内核。

- Cortex-R—面向实时应用的高性能内核。

- Cortex-M—面向各类嵌入式应用的微控制器内核。


### 系列型号

意法半导体芯片的种类繁多，主要系列如下：

![芯片种类](./Picture/芯片种类.png)

### 命名规则

芯片的命名包含了芯片的基本信息，具体命名规则如下：

MCU

![芯片命名规则](./Picture/芯片命名规则.png)

MPU

![MPU命名规则](./Picture/MPU命名规则.png)

选型手册：[Selection_Guide.pdf](https://static.stmcu.com.cn/upload/Selection_Guide.pdf)

PS.
MPU 和 MCU的最主要的区别就是 MPU 的性能更强具有 MMU（内存管理单元）能够运行 Linux 操作系统，而 MCU 则只能运行 FreeRTOS 这类实时操作系统。