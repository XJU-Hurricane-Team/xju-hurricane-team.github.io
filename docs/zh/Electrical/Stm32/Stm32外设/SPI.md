# SPI 总线介绍与使用指南

## 一、SPI 总线概述

SPI（Serial Peripheral Interface，串行外设接口）是一种由摩托罗拉公司开发的同步串行通信协议，主要用于短距离、高速率的芯片间通信。SPI 采用主从（Master-Slave）架构，支持全双工通信，广泛应用于 MCU 与外设（如传感器、Flash、显示屏、ADC/DAC 等）之间的数据交互。

### 1.1 核心特性

**同步通信**：依赖时钟信号同步数据传输，无起始 / 停止位，传输效率高；

**全双工**：主从设备可同时收发数据；

**多从机支持**：通过片选（CS/SS）信号实现多个从机挂载；

**灵活的时钟配置**：支持时钟极性（CPOL）和时钟相位（CPHA）的组合配置，适配不同外设；

**无地址机制**：通过片选信号指定通信从机，协议简单。

### 1.2 硬件引脚定义

SPI 总线核心引脚（4 线制，部分场景可简化为 3 线）：

| 引脚名称 |         英文全称         |                        功能描述                        |
| :------: | :----------------------: | :----------------------------------------------------: |
|   SCK    |       Serial Clock       |          时钟信号，由主设备产生，控制通信时序          |
|   MOSI   |   Master Out Slave In    |               主设备发送、从设备接收数据               |
|   MISO   |   Master In Slave Out    |               主设备接收、从设备发送数据               |
|  CS/SS   | Chip Select/Slave Select | 片选信号，主设备拉低选中对应从机，低电平有效（可配置） |

## 二、SPI 通信原理

### 2.1 时钟极性与相位（CPOL/CPHA）

SPI 的通信时序由 CPOL 和 CPHA 两个参数决定，共 4 种组合模式，需与外设手册匹配：

**CPOL（时钟极性）**：定义 SCK 空闲时的电平

CPOL=0：空闲时 SCK 为低电平；

CPOL=1：空闲时 SCK 为高电平。

**CPHA（时钟相位）**：定义数据采样的时机

CPHA=0：在 SCK 第一个跳变沿（上升 / 下降）采样数据；

CPHA=1：在 SCK 第二个跳变沿（上升 / 下降）采样数据。

### 2.2 通信流程

1. 主设备拉低目标从机的 CS 引脚，选中该从机；
2. 主设备产生 SCK 时钟信号，同步发送 / 接收数据：主设备通过 MOSI 逐位发送数据；从设备通过 MISO 逐位返回数据；
3. 通信完成后，主设备拉高 CS 引脚，释放从机。

## 三、工程中 SPI 的使用（以 STM32 为例）

### 3.1 SPI 初始化配置（STM32 HAL 库）

#### 1. 引脚初始化（gpio.c）

```
#include "gpio.h"

void MX_GPIO_Init(void)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};

  /* 使能 SPI1 引脚时钟 */
  __HAL_RCC_GPIOA_CLK_ENABLE();

  /* SPI1 SCK: PA5, MOSI: PA7, MISO: PA6 */
  GPIO_InitStruct.Pin = GPIO_PIN_5 | GPIO_PIN_7 | GPIO_PIN_6;
  GPIO_InitStruct.Mode = GPIO_MODE_AF_PP; // 复用推挽输出（SCK/MOSI）
  GPIO_InitStruct.Pull = GPIO_NOPULL;
  GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  /* CS 引脚: PA4（手动控制） */
  GPIO_InitStruct.Pin = GPIO_PIN_4;
  GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
  HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

  /* 初始拉高 CS，未选中从机 */
  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET);
}
```

#### 2. SPI 外设初始化（spi.c）

```
#include "spi.h"

SPI_HandleTypeDef hspi1;

void MX_SPI1_Init(void)
{
  /* SPI 基本配置 */
  hspi1.Instance = SPI1;
  hspi1.Init.Mode = SPI_MODE_MASTER;          // 主设备模式
  hspi1.Init.Direction = SPI_DIRECTION_2LINES;// 全双工（2 线）
  hspi1.Init.DataSize = SPI_DATASIZE_8BIT;    // 8 位数据宽度
  hspi1.Init.CLKPolarity = SPI_POLARITY_LOW;  // CPOL=0（空闲低电平）
  hspi1.Init.CLKPhase = SPI_PHASE_1EDGE;      // CPHA=0（第一个边沿采样）
  hspi1.Init.NSS = SPI_NSS_SOFT;              // 软件控制 NSS（CS）
  hspi1.Init.BaudRatePrescaler = SPI_BAUDRATEPRESCALER_16; // 时钟分频（APB2=72MHz → 4.5MHz）
  hspi1.Init.FirstBit = SPI_FIRSTBIT_MSB;     // 高位先行
  hspi1.Init.TIMode = SPI_TIMODE_DISABLE;
  hspi1.Init.CRCCalculation = SPI_CRCCALCULATION_DISABLE;
  if (HAL_SPI_Init(&hspi1) != HAL_OK)
  {
    Error_Handler(); // 初始化失败处理
  }
}

/* SPI 底层硬件初始化（MSP） */
void HAL_SPI_MspInit(SPI_HandleTypeDef* spiHandle)
{
  if(spiHandle->Instance==SPI1)
  {
    __HAL_RCC_SPI1_CLK_ENABLE(); // 使能 SPI1 时钟
  }
}
```

### 3.2 SPI 读写函数封装（spi.c）

```
#include "spi.h"
#include "gpio.h"

/**
 * @brief  SPI 写字节到外设
 * @param  data: 要发送的字节
 * @retval 无
 */
void SPI_WriteByte(uint8_t data)
{
  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET); // 拉低 CS，选中从机
  HAL_SPI_Transmit(&hspi1, &data, 1, 100); // 发送 1 字节，超时 100ms
  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET); // 拉高 CS，释放从机
}

/**
 * @brief  SPI 从外设读取字节
 * @retval 读取到的字节
 */
uint8_t SPI_ReadByte(void)
{
  uint8_t data = 0;
  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET); // 拉低 CS
  HAL_SPI_Receive(&hspi1, &data, 1, 100); // 接收 1 字节
  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET); // 拉高 CS
  return data;
}

/**
 * @brief  SPI 读写数据（全双工）
 * @param  tx_buf: 发送缓冲区
 * @param  rx_buf: 接收缓冲区
 * @param  len: 数据长度
 * @retval HAL 状态码
 */
HAL_StatusTypeDef SPI_ReadWrite(uint8_t* tx_buf, uint8_t* rx_buf, uint16_t len)
{
  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_RESET);
  HAL_StatusTypeDef status = HAL_SPI_TransmitReceive(&hspi1, tx_buf, rx_buf, len, 100);
  HAL_GPIO_WritePin(GPIOA, GPIO_PIN_4, GPIO_PIN_SET);
  return status;
}
```

### 3.3 主函数调用示例（main.c）

```
#include "main.h"
#include "spi.h"
#include "gpio.h"

int main(void)
{
  /* 初始化硬件层 */
  HAL_Init();
  SystemClock_Config();
  MX_GPIO_Init();
  MX_SPI1_Init();

  uint8_t tx_data = 0x5A;
  uint8_t rx_data = 0;

  while (1)
  {
    /* 写数据到 SPI 外设 */
    SPI_WriteByte(tx_data);

    /* 从 SPI 外设读数据 */
    rx_data = SPI_ReadByte();

    /* 全双工读写示例 */
    uint8_t tx_buf[4] = {0x01, 0x02, 0x03, 0x04};
    uint8_t rx_buf[4] = {0};
    SPI_ReadWrite(tx_buf, rx_buf, 4);

    HAL_Delay(1000); // 延时 1s
  }
}
```

## 四、SPI 使用注意事项

1. **时钟速率匹配**：主设备 SCK 速率需低于外设支持的最大速率（通常外设手册会标注）；
2. **CPOL/CPHA 配置**：必须与外设的 SPI 时序要求一致，否则数据读写错误；
3. **CS 信号控制**：每次通信前拉低 CS，结束后拉高；多从机场景需确保同一时刻仅一个从机的 CS 被拉低；
4. **数据位宽**：根据外设要求配置 8 位 / 16 位数据宽度；
5. **总线冲突**：SPI 无总线仲裁机制，多主设备场景需额外设计冲突避免逻辑；
6. **硬件兼容性**：部分外设仅支持半双工，需配置 `SPI_DIRECTION_1LINE` 模式；
7. **中断 / DMA 模式**：高频 / 大数据量传输建议使用中断或 DMA 模式，避免阻塞主循环（HAL 库接口：`HAL_SPI_Transmit_IT`/`HAL_SPI_Transmit_DMA`）。