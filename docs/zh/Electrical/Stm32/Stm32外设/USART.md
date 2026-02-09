# 串口

## 串口实质

串口即串行通信接口，本质是**以位（bit）为单位，将数据沿单一传输线路逐位依次发送 / 接收的硬件通信接口**。

## 串口的同步与异步通信

**异步通信**：无专用时钟线，收发双方提前约定**波特率**、**数据帧格式**实现同步，以 UART 为核心代表；数据按独立帧传输，帧含起始位、数据位、校验位、停止位，帧间可空闲，硬件简单、容错性稍高，但有冗余开销，传输效率较低，适配**低速率短距离**场景。

![](D:\09_Laboratory\04_FileMark\xju-hurricane-team.github.io\docs\zh\Electrical\SerialPort\Pictures\异步通信.png)

**同步通信**：由**统一时钟源**（专用时钟线或内嵌时钟信号）实现严格同步，以 SPI、IIC 为核心代表；数据以连续帧块传输，含同步字符无冗余起始 / 停止位，传输效率极高、速率上限高，但硬件复杂度高，对时钟偏差和布线抗干扰性要求严苛，适配**高速率大数据量**场景。

![](D:\09_Laboratory\04_FileMark\xju-hurricane-team.github.io\docs\zh\Electrical\SerialPort\Pictures\同步通信.png)

## 常见串口分类

1.**UART**：通用异步收发传输器，是**异步串口的底层硬件标准**，仅定义数据帧格式和异步通信逻辑，无专属电气特性和物理接口，需搭配电平标准(TTL、RS-232、RS-422等)使用，核心为 TX（发送）、RX（接收）引脚，是所有异步串口的基础。

![](D:\09_Laboratory\04_FileMark\xju-hurricane-team.github.io\docs\zh\Electrical\SerialPort\Pictures\uart.png)

2.**TTL ：基于 UART 的**电平标准，为单片机 / 嵌入式设备原生串口，电平为 3.3V/5V（高电平表示 1，低电平表示 0），可直接对接同电平 UART 设备，无需电平转换，是嵌入式开发调试的常用串口。

3.**RS232：在 UART 基础上定义**电气特性和物理接口的异步串口标准，采用正负电平（+3~+15V 表示 0，-3~-15V 表示 1），解决 TTL 电平传输距离短的问题，经典物理接口为 DB9，需通过 MAX232 等芯片做 TTL-RS232 电平转换，才能与单片机对接，常见于老式电脑 COM 口、传统外设。

![](D:\09_Laboratory\04_FileMark\xju-hurricane-team.github.io\docs\zh\Electrical\SerialPort\Pictures\RS232接口.png)

-----

串口、COM口是指的物理接口形式(硬件)。而TTL、RS-232、RS-485是指的电平标准(电信号)。 

![](D:\09_Laboratory\04_FileMark\xju-hurricane-team.github.io\docs\zh\Electrical\SerialPort\Pictures\COMS和UART.png)

嵌入式里面说的串口，一般是指UART口。4个pin（Vcc,GND,RX,TX），用TTL电平。

PC中的COM口即串行通讯端口，简称串口。9个Pin，用RS232电平。 

-----

4.**SPI**：串行外设接口，**同步串口协议**，采用四线制（SCLK 时钟、MOSI 主发从收、MISO 主收从发、CS 片选），单主多从架构，全双工通信，速率快，抗干扰性好，适用于主控与高速外设（闪存、显示屏、传感器）的短距离对接。

![](D:\09_Laboratory\04_FileMark\xju-hurricane-team.github.io\docs\zh\Electrical\SerialPort\Pictures\SPI接线.png)

5.**IIC**：集成电路总线，**同步串口协议**，采用二线制（SCL 时钟、SDA 数据线），多主多从架构，半双工通信，布线极简，可实现多设备总线组网，适用于低速外设（温湿度传感器、EEPROM）的近距离对接。

![](D:\09_Laboratory\04_FileMark\xju-hurricane-team.github.io\docs\zh\Electrical\SerialPort\Pictures\IIC总线.png)

## 波特率和比特率

异步串口通信的核心前提之一是波特率一致。

**比特率**：指**单位时间内实际传输的有效二进制位数**，单位为 bps（位 / 秒），反映串口实际数据传输能力。

**波特率**：指**单位时间内传输的信号码元数**，单位为 Baud（波特）；在串口通信中，码元通常与二进制位一一对应（1 个码元表示 1 位），因此**波特率数值上等于比特率**（如 9600 波特率对应 9600bps 比特率），是串口通信中约定速率的核心指标，常见值有 9600、19200、115200 等，速率越高传输越快，对应传输距离越短。

## 数据传输方向：全双工与半双工

**全双工**：支持**同一时间双向同时收发数据**，收发通道相互独立，无需分时占用传输线路；**适配类别**：UART、TTL 、RS232、SPI（TX/RX 通道独立，时钟同步实现同时收发）。

**半双工**：同一时间仅能单向传输数据，收发共用一条传输线路，需分时发送 / 接收；**适配类别**：IIC（SDA 数据线收发共用）、工业常用的 RS485（异步串口，总线型组网，收发共用差分线路）。

# USART/UART 介绍及使用

## 一、USART/UART 基础介绍

### 1. 概念与区别

**UART（通用异步收发传输器）**：仅支持异步通信，通过 TX（发送）、RX（接收）两根信号线实现全双工数据传输，无需时钟同步，依靠波特率、数据位、停止位、校验位等参数保证通信同步。

**USART（通用同步异步收发传输器）**：兼容 UART 功能，额外支持同步通信（需 SCLK 时钟线），本文以更常用的异步模式（UART）为核心讲解。

### 2. 核心通信参数

UART 通信的核心参数决定了数据帧的格式，需通信双方完全一致：

**波特率**：每秒传输的比特数（如 9600、115200），是通信速率的核心指标；

**字长**：一帧数据的有效位（通常 8 位 / 9 位）；

**停止位**：一帧数据结束的标志（1 位 / 1.5 位 / 2 位）；

**奇偶校验位**：用于检错（无校验 / 奇校验 / 偶校验）；

**硬件流控**：可选（无流控 / RTS/CTS），用于控制数据收发节奏，普通场景无需开启。

## 二、STM32 HAL 库 UART 核心结构体

### 1. UART 句柄结构体（UART_HandleTypeDef）

该结构体是 HAL 库操作 UART 的核心，包含外设基地址、配置参数、缓冲区、状态信息、回调函数等：

```
typedef struct __UART_HandleTypeDef
{
  USART_TypeDef                 *Instance;        /*!< UART寄存器基地址        */
  UART_InitTypeDef              Init;             /*!< UART通信参数配置结构体  */
  uint8_t                       *pTxBuffPtr;      /*!< 指向UART发送传输缓冲区的指针 */
  uint16_t                      TxXferSize;       /*!< UART发送传输的总数据长度    */
  __IO uint16_t                 TxXferCount;      /*!< UART发送传输的剩余数据计数器 */
  uint8_t                       *pRxBuffPtr;      /*!< 指向UART接收传输缓冲区的指针 */
  uint16_t                      RxXferSize;       /*!< UART接收传输的总数据长度    */
  __IO uint16_t                 RxXferCount;      /*!< UART接收传输的剩余数据计数器 */
  DMA_HandleTypeDef             *hdmatx;          /*!< UART发送DMA句柄参数        */
  DMA_HandleTypeDef             *hdmarx;          /*!< UART接收DMA句柄参数        */
  HAL_LockTypeDef               Lock;             /*!< 锁对象（用于多任务/中断下的资源保护） */
  __IO HAL_UART_StateTypeDef    gState;           /*!< UART全局状态信息 */
  __IO HAL_UART_StateTypeDef    RxState;          /*!< UART接收操作状态信息 */
  __IO uint32_t                 ErrorCode;        /*!< UART错误码 */
#if (USE_HAL_UART_REGISTER_CALLBACKS == 1)
  void (* TxHalfCpltCallback)(struct __UART_HandleTypeDef *huart);        /*!< 发送完成一半回调 */
  void (* TxCpltCallback)(struct __UART_HandleTypeDef *huart);            /*!< 发送完成回调 */
  void (* RxHalfCpltCallback)(struct __UART_HandleTypeDef *huart);        /*!< 接收完成一半回调 */
  void (* RxCpltCallback)(struct __UART_HandleTypeDef *huart);            /*!< 接收完成回调 */
  void (* ErrorCallback)(struct __UART_HandleTypeDef *huart);             /*!< 错误回调 */
  // 其他回调函数省略
#endif
} UART_HandleTypeDef;
```

### 2. UART 初始化结构体（UART_InitTypeDef）

用于配置 UART 核心通信参数：

```
typedef struct
{
  uint32_t BaudRate;                  /*!< 波特率 */
  uint32_t WordLength;                /*!< 字长（UART_WORDLENGTH_8B/9B） */
  uint32_t StopBits;                  /*!< 停止位（UART_STOPBITS_1/1_5/2） */
  uint32_t Parity;                    /*!< 校验位（UART_PARITY_NONE/ODD/EVEN） */
  uint32_t Mode;                      /*!< 模式（UART_MODE_TX/RX/TX_RX） */
  uint32_t HwFlowCtl;                 /*!< 硬件流控（UART_HWCONTROL_NONE/RTS/CTS/RTS_CTS） */
  uint32_t OverSampling;              /*!< 过采样（UART_OVERSAMPLING_16/8，仅部分芯片支持8倍） */
} UART_InitTypeDef;
```

## 三、UART 配置与使用步骤

### 1. 核心配置流程

1. 初始化 UART 句柄，配置通信参数（波特率、字长等）；
2. 实现底层硬件初始化（GPIO、时钟、中断）；
3. 配置中断优先级并使能中断（中断接收场景）；
4. 编写中断服务函数及回调函数；
5. 实现数据收发逻辑。

### 2. 完整代码示例

#### 1. 头文件与宏定义（usart.h）

```
#ifndef __USART_H
#define __USART_H

#include "stm32f1xx_hal.h"

/* 串口宏定义 */
#define USART_UX           USART1
#define USART_TX_GPIO_PORT GPIOA
#define USART_TX_GPIO_PIN  GPIO_PIN_9
#define USART_RX_GPIO_PORT GPIOA
#define USART_RX_GPIO_PIN  GPIO_PIN_10
#define USART_UX_IRQn      USART1_IRQn

/* 时钟使能宏 */
#define USART_TX_GPIO_CLK_ENABLE() __HAL_RCC_GPIOA_CLK_ENABLE()
#define USART_RX_GPIO_CLK_ENABLE() __HAL_RCC_GPIOA_CLK_ENABLE()
#define USART_UX_CLK_ENABLE()      __HAL_RCC_USART1_CLK_ENABLE()

/* 接收相关配置 */
#define USART_EN_RX        1          /* 使能接收中断 */
#define RXBUFFERSIZE       1          /* 单次中断接收字节数 */
#define USART_REC_LEN      4096       /* 最大接收字节数 */

/* 全局变量声明 */
extern UART_HandleTypeDef g_uart1_handle;
extern uint8_t g_rx_buffer[RXBUFFERSIZE];
extern uint8_t g_usart_rx_buf[USART_REC_LEN];
extern uint16_t g_usart_rx_sta;

/* 函数声明 */
void usart_init(uint32_t baudrate);

#endif
```

#### 2. 串口底层实现（usart.c）

```
#include "usart.h"

/* 全局变量定义 */
UART_HandleTypeDef g_uart1_handle;    /* UART句柄 */
uint8_t g_rx_buffer[RXBUFFERSIZE];    /* 中断接收临时缓冲区 */
uint8_t g_usart_rx_buf[USART_REC_LEN];/* 接收数据缓冲区 */
uint16_t g_usart_rx_sta = 0;          /* 接收状态标志：
                                         bit15：接收完成标志
                                         bit14：接收到回车符(\r)
                                         bit0~13：接收字节数 */

/**
 * @brief  串口初始化函数
 * @param  baudrate：串口波特率
 * @retval 无
 */
void usart_init(uint32_t baudrate)
{
    /* 1. 初始化UART句柄参数 */
    g_uart1_handle.Instance = USART_UX;
    g_uart1_handle.Init.BaudRate = baudrate;                  /* 波特率 */
    g_uart1_handle.Init.WordLength = UART_WORDLENGTH_8B;      /* 8位数据位 */
    g_uart1_handle.Init.StopBits = UART_STOPBITS_1;           /* 1位停止位 */
    g_uart1_handle.Init.Parity = UART_PARITY_NONE;            /* 无校验 */
    g_uart1_handle.Init.HwFlowCtl = UART_HWCONTROL_NONE;      /* 无硬件流控 */
    g_uart1_handle.Init.Mode = UART_MODE_TX_RX;               /* 收发双工 */
    g_uart1_handle.Init.OverSampling = UART_OVERSAMPLING_16;  /* 16倍过采样 */
    HAL_UART_Init(&g_uart1_handle);                           /* 初始化UART */

    /* 2. 开启中断接收（HAL库中断接收为单次触发，需在回调中重新开启） */
    HAL_UART_Receive_IT(&g_uart1_handle, g_rx_buffer, RXBUFFERSIZE);
}

/**
 * @brief  UART底层硬件初始化（由HAL_UART_Init调用）
 * @param  huart：UART句柄指针
 * @retval 无
 */
void HAL_UART_MspInit(UART_HandleTypeDef *huart)
{
    GPIO_InitTypeDef gpio_init_struct;

    if (huart->Instance == USART_UX)
    {
        /* 1. 使能时钟 */
        USART_TX_GPIO_CLK_ENABLE();
        USART_RX_GPIO_CLK_ENABLE();
        USART_UX_CLK_ENABLE();

        /* 2. 配置TX引脚（复用推挽输出） */
        gpio_init_struct.Pin = USART_TX_GPIO_PIN;
        gpio_init_struct.Mode = GPIO_MODE_AF_PP;
        gpio_init_struct.Pull = GPIO_PULLUP;
        gpio_init_struct.Speed = GPIO_SPEED_FREQ_HIGH;
        HAL_GPIO_Init(USART_TX_GPIO_PORT, &gpio_init_struct);

        /* 3. 配置RX引脚（复用输入） */
        gpio_init_struct.Pin = USART_RX_GPIO_PIN;
        gpio_init_struct.Mode = GPIO_MODE_AF_INPUT;
        HAL_GPIO_Init(USART_RX_GPIO_PORT, &gpio_init_struct);

        /* 4. 配置中断（使能+优先级） */
#if USART_EN_RX
        HAL_NVIC_EnableIRQ(USART_UX_IRQn);
        HAL_NVIC_SetPriority(USART_UX_IRQn, 3, 3);  /* 抢占优先级3，子优先级3 */
#endif
    }
}

/**
 * @brief  接收完成回调函数（中断接收完成后触发）
 * @param  huart：UART句柄指针
 * @retval 无
 */
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART_UX)
    {
        if ((g_usart_rx_sta & 0x8000) == 0)  /* 未接收完成 */
        {
            if (g_usart_rx_sta & 0x4000)      /* 已接收到回车符(\r) */
            {
                if (g_rx_buffer[0] != 0x0a)   /* 未接收到换行符(\n)，判定接收异常 */
                {
                    g_usart_rx_sta = 0;       /* 重置状态 */
                }
                else
                {
                    g_usart_rx_sta |= 0x8000; /* 置位接收完成标志 */
                }
            }
            else                              /* 未接收到回车符(\r) */
            {
                if (g_rx_buffer[0] == 0x0d)
                {
                    g_usart_rx_sta |= 0x4000; /* 标记接收到回车符 */
                }
                else
                {
                    /* 存储有效数据 */
                    g_usart_rx_buf[g_usart_rx_sta & 0x3FFF] = g_rx_buffer[0];
                    g_usart_rx_sta++;

                    /* 防止缓冲区溢出 */
                    if (g_usart_rx_sta > (USART_REC_LEN - 1))
                    {
                        g_usart_rx_sta = 0;
                    }
                }
            }
        }

        /* 重新开启中断接收，保证连续接收 */
        HAL_UART_Receive_IT(&g_uart1_handle, g_rx_buffer, RXBUFFERSIZE);
    }
}

/**
 * @brief  串口中断服务函数
 * @retval 无
 */
void USART_UX_IRQHandler(void)
{
    /* 调用HAL库中断处理函数，解析中断类型并触发对应回调 */
    HAL_UART_IRQHandler(&g_uart1_handle);
}
```

#### 3. 主函数测试（main.c）

```
#include "./stm32f1xx_it.h"
#include "./SYSTEM/sys/sys.h"
#include "./SYSTEM/usart/usart.h"
#include "./SYSTEM/delay/delay.h"
#include "./BSP/LED/led.h"

int main(void)
{
    uint8_t len;
    uint16_t times = 0;

    /* 系统初始化 */
    HAL_Init();                              /* HAL库初始化 */
    sys_stm32_clock_init(RCC_PLL_MUL9);      /* 配置系统时钟为72MHz */
    delay_init(72);                          /* 延时初始化 */
    usart_init(115200);                      /* 串口初始化：115200波特率 */
    led_init();                              /* LED初始化（状态指示） */

    /* 主循环 */
    while (1)
    {
        if (g_usart_rx_sta & 0x8000)         /* 检测到完整数据接收 */
        {
            len = g_usart_rx_sta & 0x3FFF;   /* 提取接收数据长度 */
            printf("\r\n你发送的消息为:\r\n");
            
            /* 回显接收到的数据 */
            HAL_UART_Transmit(&g_uart1_handle, g_usart_rx_buf, len, 1000);
            while(__HAL_UART_GET_FLAG(&g_uart1_handle, UART_FLAG_TC) != SET); /* 等待发送完成 */
            
            printf("\r\n\r\n");
            g_usart_rx_sta = 0;              /* 重置接收状态，准备下次接收 */
        }
        else
        {
            times++;
            /* 定时打印提示信息 */
            if (times % 5000 == 0)
            {
                printf("\r\nSTM32F1 USART 实验\r\n");
                printf("请输入数据，以回车键结束\r\n");
            }
            /* LED翻转，指示系统运行 */
            if (times % 30 == 0)
            {
                LED0_TOGGLE();
            }
            delay_ms(10);                    /* 降低CPU占用 */
        }
    }
}
```

## 四、代码关键说明

### 1. 中断接收逻辑

HAL 库的`HAL_UART_Receive_IT`为**单次中断接收**，即接收 1 字节数据后中断触发一次，需在`HAL_UART_RxCpltCallback`中重新调用该函数，实现连续接收。

### 2. 接收状态标志（g_usart_rx_sta）

`bit15`：接收完成（检测到`\r+\n`）；

`bit14`：检测到`\r`（回车符）；

`bit0~13`：记录接收字节数（最大 4095）；

若接收数据溢出或异常，重置状态为 0，重新接收。

## 五、常见使用场景与注意事项

### 1. 常用 API 说明

|           函数名            |            功能            |
| :-------------------------: | :------------------------: |
|      `HAL_UART_Init()`      |      初始化 UART 外设      |
|    `HAL_UART_Transmit()`    |       阻塞式发送数据       |
|   `HAL_UART_Receive_IT()`   |        开启中断接收        |
|   `HAL_UART_IRQHandler()`   |      中断处理核心函数      |
| `HAL_UART_RxCpltCallback()` | 接收完成回调（需用户实现） |

### 2. 注意事项

1. 通信双方参数必须一致（波特率、字长、停止位、校验位）；
2. GPIO 配置需匹配：TX 为`GPIO_MODE_AF_PP`（复用推挽），RX 为`GPIO_MODE_AF_INPUT`（复用输入）；
3. 中断优先级配置需合理，避免与其他高优先级中断冲突；
4. 接收缓冲区需预留足够空间，防止数组越界；
5. 若使用 DMA 收发，需额外配置 DMA 句柄及相关中断。

## 六、扩展说明

**DMA 模式**：大数量数据收发建议使用 DMA（`HAL_UART_Transmit_DMA`/`HAL_UART_Receive_DMA`），降低 CPU 占用；

**错误处理**：可实现`ErrorCallback`回调函数，处理帧错误、奇偶校验错误等异常；

**多串口扩展**：复制上述代码，修改外设宏定义（如 USART2、USART3），即可实现多串口配置。