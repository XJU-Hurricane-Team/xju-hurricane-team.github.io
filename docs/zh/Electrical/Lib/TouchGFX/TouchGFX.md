![logo|637](./Picture/logo.svg)

## 前言

TouchGFX 是一款适用于 STM32 MCU 的嵌入式 UI 框架，提供了便捷的 UI 设计软件，多种帧缓存策略以及详尽的开发文档。本文以 26 年战队使用的遥控器工程为例，简单讲解如何在战队模板的基础上搭建 TouchGFX  工程方便后续队员上手使用。

[TouchGFX文档 ](https://support.touchgfx.com/zh-CN/docs/introduction/welcome)

[26遥控项目仓库](https://github.com/XJU-Hurricane-Team/26RemoteCtrl)

## 硬件选型

### 硬件概述

在开发嵌入式 UI 的过程中首先需要根据硬件条件考虑软件设计，通常需要考虑四个方面：

- 屏幕：驱动芯片，屏幕尺寸，传输接口。
- MCU：MCU资源大小，主频，接口带宽。
- RAM：用于设置帧缓存区，帧缓存区也可能在 MCU 内部 RAM 中。
- FLASH：用于存储字库，图片，视频。

![](Picture/4-main-parts.webp)

如上图所示完整的工作流程为： MCU 负责加载 FLASH 当中的字库，图片等资源，经由 UI 引擎渲染出要显现的内容，MCU 将渲染结果写入 RAM 当中的缓存区，再由通信接口（LTDC，FMC，SPI）将缓存区的内容传输给屏幕完成显示。

### 例程分析

遥控的屏幕采用`240*320`分辨率 SPI 接口 St7789 主控的 LCD 屏，按照色深为 16bpp （RGB565）的情况计算，每一帧画面所需要的存储空间为 150kb 。主控芯片为 STM32F429VET6 （ROM：1024.0KB RAM1：192.0KB RAM2：64.0kb）在没有拓展 SDRAM 的情况下仅使用内部 RAM 作为缓存区实现单帧缓存较为困难，因此采用[部分帧缓存策略](https://support.touchgfx.com/zh-CN/docs/basic-concepts/framebuffer#display-with-gram-partial-buffering)。F4 主频为180MHz，APB2总线的时钟频率为90MHz， SPI 接口在2分频的情况下最高传输速率为`45Mbit/s`，单帧传输事件为 26.7ms，考虑到不可能跑满带宽因此实际的刷新率应该在 30Hz 左右。

![](Picture/gram-spi.webp)

## 初始配置

### CubeMX配置

初始化过程需要在 CubeMX 当中配置，首先需要初始化传输接口，注意需要开启 DMA 和 DMA 中断。
![](Picture/Pasted%20image%2020260406122324.png)

注意 DMA 的`Data Width`要设置为半字宽度。
![](Picture/Pasted%20image%2020260407115857.png)

开启 CRC 和 DMA2D（如果 MCU 有的话）。
![](Picture/Pasted%20image%2020260406122614.png)

![](Picture/Pasted%20image%2020260406122635.png)

添加 TouchGFX 组件，这里使用组件版本为4.26.1。

![](Picture/Pasted%20image%2020260406111430.png)

这里根据前面硬件选型部分的分析进行选择，需要注意接口是SPI 的情况下`interface`选择`Custom`，`Number of Blocks`是指缓存区的块数，`Block Size`是每块缓存区的大小。战队模板当中的 FreeRTOS 是手动添加的因此这里的`Real-Time Operating System`选项我们选择`Custom`后续需要手动进行代码配置，如果想要 CubeMX 直接生成可以选择`CMSIS-V2`。由于我们使用的 LCD 模块没有引出 [TE 引脚](https://support.touchgfx.com/zh-CN/docs/development/scenarios/touchgfx-on-lowcost-hardware#tearing-effect)所以`Partial Framebuffer VSync`选项不开启。

配置成功后即可生成代码，如果使用的是战队模板需要取消`cleanup`脚本的执行。

![](Picture/Pasted%20image%2020260406124956.png)

### 工程配置

点击生成代码后在`\CubeMX\TouchGFX`目录下会生成如下文件，点击红框中的文件即可打开[TouchGFX Designer](https://www.st.com.cn/zh/development-tools/touchgfxdesigner.html) 进行 [UI 设计](https://support.touchgfx.com/zh-CN/docs/category/designer-user-guide)。

![](Picture/Pasted%20image%2020260406125606.png)

在`\CubeMX\Middlewares\ST\touchgfx`路径下会生成如下文件，这些是 TouchGFX 的核心库文件。
![](Picture/Pasted%20image%2020260406130259.png)
打开 EIDE 按照下图添加文件到工程
![](Picture/Pasted%20image%2020260406130902.png)

注意这里`touchgfx_core_wchar32.lib`为 TouchGFX 框架的核心库，仅提供二进制库文件，需要根据工程使用的编译器类型选择适配的库文件，所有库文件都存放在`\CubeMX\Middlewares\ST\touchgfx\lib\core\cortex_m4f`路径下，这里例程使用是 AC6 编译器选择图中的库文件（注意此处选用touchgfx_core_wchar32.lib时需将构建器配置—C/C++下的enum/wchar is short type取消勾选）。

添加包含的目录：
![](Picture/Pasted%20image%2020260406131328.png)

## 框架说明

### 图形引擎

TouchGFX 框架中使用[图形引擎](https://support.touchgfx.com/zh-CN/docs/basic-concepts/graphics-engine)绘制需要显示图形，它的工作方式分为三步循环执行：

- **采集事件**：采集触摸屏事件、物理按钮按下事件和来自后台系统的消息等。
- **更新场景模型**：对采集的事件做出响应，更新模型的位置、动画、色彩和图像等。
- **渲染场景模型**：重绘模型中已更新的部分，并使之显示在显示屏上。

![](Picture/engine-cycle.webp)

由于引擎没有源码其内部对于我们来说是个黑盒，我们只需要关心引擎如何启动，循环执行需要什么条件，以及渲染后的数据如何传递给屏幕即可。

### TouchGFX AL

[TouchGFX抽象层（AL）](https://support.touchgfx.com/zh-CN/docs/development/touchgfx-hal-development/touchgfx-al-development-introduction)是位于用户代码与 TouchGFX 引擎之间的软件组件，分为硬件抽象层（HAL）和操作系统抽象层（OSAL），其主要任务是将引擎分别与底层硬件和操作系统相结合。TouchGFX 抽象层使用 C++ 编写，通过类的继承分为如下三层：

- `HAL`：祖父类，是整个框架的基础，类的声明在`CubeMX\Middlewares\ST\touchgfx\framework\include\touchgfx\hal\HAL.hpp`文件当中，方法的实现编译在 TouchGFX 的核心库当中无法直接查看，这个类无需我们手动修改，由 CubeMX 自动添加。

- `TouchGFXGeneratedHAL` ：父类，继承于`HAL`类，类的声明位于`CubeMX\TouchGFX\target\generated\TouchGFXGeneratedHAL.hpp`文件中，这个类由TouchGFX Generator 根据项目配置自动生成，定义了项目配置所需要的函数，这个类不需要用户手动修改，但其成员函数中调用了需要用户手动实现的函数。

- `TouchGFXHAL`：子类，继承于`TouchGFXGeneratedHAL`，声明于`CubeMX\TouchGFX\target\TouchGFXHAL.hpp`，这个类在第一次由TouchGFX Generator 根据项目配置自动生成，用户可以在此基础上根据自己的需求更改这个类，框架初始化时声明的也是这个类的对象。

![](Picture/code-architecture.webp) 

## 代码配置

### 初始化流程

```text
       MX_TouchGFX_Init()
               ├─ touchgfx_components_init()       // 组件预初始化
               └─ touchgfx_init()
                    ├─ 注册 Bitmap / Texts / Fonts 
                    └─ hal.initialize()（调用点，实际调用TouchGFXHAL::initialize()）
                            └─ TouchGFXHAL::initialize()                 // 板级初始化
                                    ├─ xSemaphoreCreateCounting(8, 0)           // 手动添加
                                    ├─ ST77xx_Init()                            // 手动添加
                                    ├─ TouchGFXGeneratedHAL::initialize()       // 引擎装配
                                    │     └─HAL::initialize()          // 引擎内部初始化（闭源）
                                    │     │ ├─ OSWrappers::initialize()  // 同步对象初始化（手写）
                                    │     │ └─ dma.initialize() → DMA2D
                                    │     ├─ registerEventListener()
                                    │     ├─ setFrameBufferAllocator()
                                    │     └─ setFrameRefreshStrategy()
                                    ├─ HAL_TIM_Base_Start_IT(&htim2)                // 手动添加
                                    └─ 重新注册hdma2d.XferErrorCallback              // 手动添加
    ————————————————————————————————————初始化完成—————————————————————————————————————
       MX_TouchGFX_Process()
               └─ touchgfx_taskEntry()
                       └─ hal.taskEntry()
                               └─ 进入主循环
```

HAL::initialize()闭源，可以确认它会调用用户实现的 OSWrappers::initialize()、并初始化 DMA2D 等，可打断点验证。

图中标注“手动添加”的位置都需要用户实现，详见下文。

### RTOS 设置

`CubeMX\TouchGFX\target\OSWrappers.cpp`这个文件中包含了引擎运行过程中所需要的同步接口。由于在 CubeMX 初始化部分我们选择的 RTOS 为`Custom`，因此文件当中的接口函数实现均为空，需要我们手动添加实现。

当前工程使用 FreeRTOS ，因此采用二值信号量 + 队列来完成 TouchGFX 的同步需求，关键如下函数的实现：

- `OSWrappers::initialize()`：创建并初始化帧缓存信号量 `frame_buffer_sem`，同时创建长度为 1 的 VSync 队列 `vsync_queue`。
- `takeFrameBufferSemaphore()/giveFrameBufferSemaphore()`：用于在渲染与传输阶段互斥访问帧缓存块。
- `waitForVSync()/signalVSync()`：引擎主循环阻塞等待下一次 VSync，定时器中断到来时通过队列唤醒。
- `taskYield()`：在需要短暂让出 CPU 时调用 `taskYIELD()`，避免 UI 线程忙等。

这套封装的作用可以简单理解为：

- **信号量**负责“这个缓存块现在能不能被画/被传”。
- **队列**负责“这一帧什么时候开始（VSync）”。

如果在 CubeMX 中选择 `CMSIS-V2`，这些逻辑会自动生成，如果和例程一样选择手动实现可以参考例程。

### 屏幕传输

为了使框架适用于任意硬件，需要用户手动实现以下在`TouchGFXGeneratedHAL.cpp`中声明的函数：

```c
/* 判断是否正在传输缓存块 */
extern "C" int touchgfxDisplayDriverTransmitActive(); 
/* 传输数据块到屏幕指定位置 */ 
extern "C" void touchgfxDisplayDriverTransmitBlock(const uint8_t* pixels, uint16_t x, uint16_t y, uint16_t w, uint16_t h);
```

同时需要手动在合适位置调用以下在`TouchGFXGeneratedHAL.cpp`中定义的函数：

```c
/* 在传输完成函数中调用启动新一轮调用 */ 
extern "C"
void DisplayDriver_TransferCompleteCallback()
{
    // After completed transmission start new transfer if blocks are ready.
    touchgfx::startNewTransfer();
}
/* 发送引擎同步信号，当渲引擎接收到此信号后启动新一轮渲染 */
extern "C"
void touchgfxSignalVSync(void)
{
    /* VSync has occurred, increment TouchGFX engine vsync counter */
    touchgfx::HAL::getInstance()->vSync();
    /* VSync has occurred, signal TouchGFX engine */
    touchgfx::OSWrappers::signalVSync();
}
```

例程中在屏幕驱动文件`st77xx.c`下调用与实现了上述函数。`DisplayDriver_TransferCompleteCallback()`函数在传输完成回调函数中调用以启动新一轮传输，`touchgfxSignalVSync(void)`函数由定时器中断回调函数定时调用，调用频率约`30Hz`。注意`touchgfxDisplayDriverShouldTransferBlock()`这个函数只有在 CubeMX 中启用 `Partial Framebuffer VSync`选项时才需要实现。

完整传输链路如下：

1. TouchGFX 引擎完成一个块的绘制后，调用 `TouchGFXGeneratedHAL::flushFrameBuffer()`。
2. `flushFrameBuffer()` 内部将块标记为可传输，并在总线空闲时调用 `touchgfxDisplayDriverTransmitBlock()`。
3. 在 `st77xx.c` 中由 `ST77xx_Bitmap()` 设置窗口并启动 `HAL_SPI_Transmit_DMA()`。
4. DMA 传输完成后进入 `HAL_SPI_TxCpltCallback()`，恢复 SPI 配置、拉高 CS，并调用 `DisplayDriver_TransferCompleteCallback()`。
5. `DisplayDriver_TransferCompleteCallback()` 内部调用 `touchgfx::startNewTransfer()`，释放已发送块并继续发送下一个已就绪块。

### 分块缓存与同步

部分帧缓存策略下，引擎并不直接画到屏幕，而是用一个或多个用户定义的内存块作为帧缓存，在“绘制—传输”之间轮转复用。块的流转由 `FrameBufferAllocator` 管理，用户只需要补齐几个同步钩子。

#### 块状态机

```text
EMPTY ──allocateBlock()────────────────► ALLOCATED   引擎拿到空画布，allocateBlock()内将块状态置为ALLOCATED表示正在绘制
ALLOCATED ──markBlockReadyForTransfer()─► DRAWN      一块画完后在flushFrameBuffer()中将块状态置为DRAWN表示可传输
DRAWN ──getBlockForTransfer()──────────► SENDING     总线空闲时flushFrameBuffer()调用getBlockForTransfer()标记块状态为正在传输
SENDING ──freeBlockAfterTransfer()─────► EMPTY       传输完成，归还
```

#### 四个同步函数

| 函数                                         | 类型           | 谁调用                           | 时机             | 上下文    |
| ------------------------------------------ | ------------ | ----------------------------- | -------------- | ------ |
| `FrameBufferAllocatorWaitOnTransfer()`     | `__weak`，需覆盖 | `allocateBlock()`             | 没有 EMPTY 块时    | GUI 任务 |
| `FrameBufferAllocatorSignalBlockDrawn()`   | `__weak`，可选  | `markBlockReadyForTransfer()` | 一块画完           | GUI 任务 |
| `touchgfxNotifyBlockTransferDone()`        | 本工程自定义       | `st77xx.c`                    | 传输启动失败 / 线程上下文 | 任务     |
| `touchgfxNotifyBlockTransferDoneFromISR()` | 本工程自定义       | `st77xx.c`                    | 传输完成 / 中断上下文   | 中断     |

- 前两个是生成代码里的弱函数，默认空实现，本工程在 `TouchGFXHAL.cpp` 中提供强定义；
- 后两个不是 TouchGFX 接口，是本工程为同步自建的函数。

##### 函数实现

```c
namespace touchgfx {
void FrameBufferAllocatorWaitOnTransfer() {
    if (g_touchgfxTransferSem != NULL) {
        // 添加超时保护，避免永久阻塞
        if (xSemaphoreTake(g_touchgfxTransferSem, pdMS_TO_TICKS(500)) !=
            pdPASS) {

            IsTransmittingBlock_ = 0;
        }
    } else {
        taskYIELD();
    }
}

void FrameBufferAllocatorSignalBlockDrawn() {
    /* Block transfer start is handled by generated HAL logic.
     * This hook is kept to satisfy the partial framebuffer contract. */
}
} // namespace touchgfx

extern "C" void touchgfxNotifyBlockTransferDone(void) {
    if (g_touchgfxTransferSem != NULL) {
        (void)xSemaphoreGive(g_touchgfxTransferSem);
    }
}

extern "C" void touchgfxNotifyBlockTransferDoneFromISR(void) {
    if (g_touchgfxTransferSem != NULL) {
        BaseType_t xHigherPriorityTaskWoken = pdFALSE;
        (void)xSemaphoreGiveFromISR(g_touchgfxTransferSem,
                                    &xHigherPriorityTaskWoken);
        portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
    }
}
```

> IsTransmittingBlock_ 是屏幕驱动中的“SPI 正在传输”标志：ST77xx_Bitmap() 置 1，DMA 完成/出错/超时时清 0，由 touchgfxDisplayDriverTransmitActive() 返回给引擎。

#### 同步过程

`g_touchgfxTransferSem` 是计数信号量（上限 8、初值 0），在 `TouchGFXHAL::initialize()` 中创建（见“引擎启动”）。

1. 引擎开始绘制时调用 `allocateBlock()`；若没有 `EMPTY` 块，就调用 `FrameBufferAllocatorWaitOnTransfer()`阻塞等待（例程 500 ms 超时兜底）。
2. **每传输完一块**，`HAL_SPI_TxCpltCallback()` 调用 `touchgfxNotifyBlockTransferDoneFromISR()`，通过 `xSemaphoreGiveFromISR()` 发送一个“完成通知”（用于释放`FrameBufferAllocatorWaitOnTransfer()`阻塞）。
3. `startNewTransfer()` 中的 `freeBlockAfterTransfer()` 把块恢复为 `EMPTY`；等待中的绘制流程被唤醒后即可继续。
4. 传输启动失败或 SPI 忙超时时，驱动走非 ISR 版 `touchgfxNotifyBlockTransferDone()` 做同样通知，避免绘制流程永久阻塞。

> 500 ms 超时只是防止死锁：超时只清除 `IsTransmittingBlock_`，**不会**把块恢复为 `EMPTY`；块的真正回收仍依赖 `freeBlockAfterTransfer()`。

### 引擎启动

CubeMX 生成的代码中会在 main 函数中自动调用`MX_TouchGFX_Init()`去执行 TouchGFX 框架初始化（见初始化流程），想要启动图形引擎只需要创建任务调用`MX_TouchGFX_Process()`函数即可，该函数内部执行引擎循环。

```c
/**
 * @brief TouchGFX_Task: TouchGFX processing task.
 * @param pvParameters Start parameters.
 */

void TouchGFX_Task(void *pvParameters) {
    UNUSED(pvParameters);
    MX_TouchGFX_Process();
}
```

其中 `MX_TouchGFX_Init()` 内部会调用到 `TouchGFXHAL::initialize()`，例程在这里做了五件事情：

1. 创建传输同步信号量 `g_touchgfxTransferSem`。

2. 调用 `ST77xx_Init()` 完成屏幕初始化。

3. 调用`TouchGFXGeneratedHAL::initialize()`初始化 OSAL/DMA2D/LCD，绑定缓存块与刷新策略，注册事件。

4. 启动 `TIM2` 中断，周期性触发 `touchgfxSignalVSync()` 给引擎提供 VSync 节拍。

5. 重新注册DMA2D回调函数避免生成代码在出错时进入 while(1)。

其中屏幕初始化和TIM2与硬件强相关。

```c
void TouchGFXHAL::initialize()
{
    // Calling parent implementation of initialize().
    //
    // To overwrite the generated implementation, omit the call to the parent function
    // and implement the needed functionality here.
    // Please note, HAL::initialize() must be called to initialize the framework.

    if (g_touchgfxTransferSem == NULL) {
        g_touchgfxTransferSem = xSemaphoreCreateCounting(8, 0);
    }
    ST77xx_Init(0, ST7789);                             // 初始化屏幕
    TouchGFXGeneratedHAL::initialize();                  
    HAL_TIM_Base_Start_IT(&htim2);                      // 启动VSync定时器

    // 修复：重新注册DMA2D错误回调，避免生成文件中的死循环
    extern DMA2D_HandleTypeDef hdma2d;
    hdma2d.XferErrorCallback = TouchGFXHAL_DMA2D_ErrorCallback;
}
```

### 注意事项

FreeRTOSConfig.h中的configTOTAL_HEAP_SIZE初始值往往较小，对遥控器工程有些局促，可以适当扩大（如直接扩大至65535）。

## UI 设置

### MVP 架构说明

TouchGFX 使用经典 MVP（Model-View-Presenter）架构。

- **Model（数据层）**：负责采集后端任务发送来的数据，存储 UI 模型的数据。
- **Presenter（中间层）**：负责将 Model 中采集到的数据更新到 View，同时将 View 层中的事件传递给 Model。
- **View（显示层）**：每一个屏幕对应一个 View 对象，存储着所有要显示的内容。

这样的架构带来的好处是：界面资源可以频繁在 Designer 里迭代，而业务逻辑可以稳定留在用户代码中，降低重新生成代码时的冲突风险。

![](Picture/mvp-2.webp)

在例程中 MVP 架构按照下述方式工作：

- 在 `Model.cpp` 负责采集数据中从 `ui_msg_queue` 非阻塞读取底层消息（遥控器、R1、R2 状态），完成数据聚合。
- 在 `screenPresenter.cpp` 中接收 Model 回调，把业务数据写入 View 的显示字段。
- 在`screenView.cpp` 中把字段格式化为文本并 `invalidate()`，触发局部重绘，将需要显示的信息显示在屏幕上。


