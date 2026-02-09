- # IIC介绍与OLED屏驱动使用

  ## 一、IIC (I2C) 基础介绍

  ### 1. 概念与核心特性

  IIC（Inter-Integrated Circuit，集成电路总线），也写作 I2C，是**半双工同步串行通信协议**，由飞利浦公司开发，主打**极简布线、多主多从架构**，仅需 SCL（时钟线）、SDA（数据线）两根信号线即可实现多设备间双向数据通信，广泛应用于嵌入式系统中低速外设的近距离对接（如 OLED、传感器、EEPROM 等）。

  核心特性：

  同步通信：通过 SCL 时钟线实现收发双方严格同步，无时钟偏差问题；
  
  半双工传输：收发共用 SDA 数据线，同一时间仅能单向传输数据；
  
  多主多从：总线支持多个主设备和从设备，通过**唯一 7 位 / 10 位设备地址**区分；
  
  硬件极简：仅 2 根信号线，开漏输出设计，需外接上拉电阻（典型 4.7KΩ）；
  
  速率分级：标准模式 100kbps、快速模式 400kbps，满足低速外设传输需求；
  
  线与特性：SCL/SDA 总线空闲时为高电平，支持多设备挂载无冲突。
  
  ### 2. 核心通信规则
  
  IIC 无专用帧起始 / 停止位，通过**总线时序信号**定义通信阶段，主从设备需严格遵循：
  
  1. **起始条件（S）**：SCL 高电平时，SDA 由高拉低，触发总线通信开始；
  2. **停止条件（P）**：SCL 高电平时，SDA 由低拉高，结束通信并释放总线；
  3. **数据传输**：SCL 低电平时 SDA 可修改（准备数据），SCL 高电平时 SDA 保持稳定（有效数据），每帧 8 位、高位先行；
  4. **应答信号（ACK）**：接收方收完 8 位数据后，在第 9 个 SCL 周期将 SDA 拉低，反馈 “接收成功”；SDA 为高则为非应答（NACK），表示传输结束 / 失败；
  5. **地址寻址**：通信开始后主设备先发送 7 位 / 10 位从设备地址，紧跟 1 位**读写位**（0 = 写，1 = 读），被寻址从设备返回 ACK 后进入数据阶段。
  
  ### 3. 与 UART/SPI 的核心区别
  
  |  对比维度  |              IIC(I2C)              |              UART               |               SPI               |
  | :--------: | :--------------------------------: | :-----------------------------: | :-----------------------------: |
  |  同步方式  |         同步（SCL 时钟线）         |   异步（波特率约定，无时钟）    |       同步（SCLK 时钟线）       |
  |  传输方式  |         半双工（SDA 共用）         |      全双工（TX/RX 独立）       |    全双工（MOSI/MISO 独立）     |
  | 信号线数量 |          2 根（SCL/SDA）           |       2-3 根（TX/RX/GND）       |   3-4 根（SCLK/MOSI/MISO/CS）   |
  |  设备寻址  |        7/10 位地址，无片选         |       无地址，一对一为主        |   片选线（CS）寻址，单主多从    |
  |  总线架构  |              多主多从              | 点对点 / 一对多（需自定义协议） |            单主多从             |
  |  典型速率  |          100kbps/400kbps           |        9600bps~115200bps        |       几 Mbps~ 几十 Mbps        |
  |  适用场景  | 低速外设（OLED / 传感器 / EEPROM） |      设备调试 / 远距离通信      | 高速外设（闪存 / 显示屏 / ADC） |
  
  ## 二、STM32 HAL 库 IIC 核心结构体
  
  STM32 HAL 库对 IIC 外设进行高度封装，核心通过**I2C_HandleTypeDef**（IIC 句柄）管理所有资源，**I2C_InitTypeDef**配置通信参数，所有 IIC 操作均基于句柄实现，无需直接操作寄存器。
  
  ### 1. I2C_HandleTypeDef 结构体（IIC 句柄结构体）
  
  封装 IIC 外设寄存器基地址、初始化参数、传输缓冲区、状态信息、DMA 句柄、回调函数等，是 HAL 库操作 IIC 的核心载体。
  
  ```
  typedef struct __I2C_HandleTypeDef
  {
    I2C_TypeDef                 *Instance;        /*!< IIC外设寄存器基地址 */
    I2C_InitTypeDef             Init;             /*!< IIC通信参数配置结构体 */
    uint8_t                     *pTxBuffPtr;      /*!< 指向IIC发送缓冲区的指针 */
    uint16_t                    TxXferSize;       /*!< IIC发送的总数据长度 */
    __IO uint16_t               TxXferCount;      /*!< IIC发送的剩余数据计数器 */
    uint8_t                     *pRxBuffPtr;      /*!< 指向IIC接收缓冲区的指针 */
    uint16_t                    RxXferSize;       /*!< IIC接收的总数据长度 */
    __IO uint16_t               RxXferCount;      /*!< IIC接收的剩余数据计数器 */
    __IO uint32_t               State;            /*!< IIC全局工作状态（@ref HAL_I2C_StateTypeDef） */
    __IO uint32_t               ErrorCode;        /*!< IIC错误码（记录超时/应答错误等） */
    DMA_HandleTypeDef           *hdmatx;          /*!< IIC发送DMA句柄参数 */
    DMA_HandleTypeDef           *hdmarx;          /*!< IIC接收DMA句柄参数 */
    HAL_LockTypeDef             Lock;             /*!< 锁对象（多任务/中断下资源保护） */
  #if (USE_HAL_I2C_REGISTER_CALLBACKS == 1)
    void (* MasterTxCpltCallback)(struct __I2C_HandleTypeDef *hi2c); /*!< 主机发送完成回调 */
    void (* MasterRxCpltCallback)(struct __I2C_HandleTypeDef *hi2c); /*!< 主机接收完成回调 */
    void (* ErrorCallback)(struct __I2C_HandleTypeDef *hi2c);        /*!< IIC错误回调 */
    void (* MspInitCallback)(struct __I2C_HandleTypeDef *hi2c);      /*!< IIC底层硬件初始化回调 */
  #endif
  } I2C_HandleTypeDef;
  ```
  
  ### 2. I2C_InitTypeDef 结构体（IIC 初始化配置结构体）
  
  用于配置 IIC 核心通信参数，主从设备参数必须完全一致，配置完成后传入`HAL_I2C_Init()`完成外设初始化。
  
  ```
  typedef struct
  {
    uint32_t ClockSpeed;         /*!< IIC时钟频率（SCL），≤100kHz(标准)/≤400kHz(快速) */
    uint32_t DutyCycle;          /*!< 时钟占空比，I2C_DUTYCYCLE_2(50%)/I2C_DUTYCYCLE_16_9 */
    uint16_t OwnAddress1;        /*!< 本机地址（主设备可任意设，从设备需唯一） */
    uint32_t AddressingMode;     /*!< 寻址模式，I2C_ADDRESSINGMODE_7BIT(常用)/10BIT */
    uint32_t DualAddressMode;    /*!< 双地址模式，一般禁用（I2C_DUALADDRESS_DISABLE） */
    uint16_t OwnAddress2;        /*!< 第二个本机地址，双地址模式下使用 */
    uint32_t GeneralCallMode;    /*!< 广播模式，一般禁用（I2C_GENERALCALL_DISABLE） */
    uint32_t NoStretchMode;      /*!< 时钟拉伸，建议使能（I2C_NOSTRETCH_DISABLE） */
  } I2C_InitTypeDef;
  ```
  
  ## 三、IIC 驱动 OLED 屏（0.96 寸 SSD1306）硬件说明
  
  ### 1. 0.96 寸 OLED 屏基本参数
  
  - 驱动芯片：**SSD1306**（主流 IIC OLED 屏驱动芯片，固定 IIC 地址）；
  - 分辨率：128×64（128 列，64 行）；
  - 显示颜色：单色（蓝 / 白，根据屏体而定）；
  - IIC 地址：**0x3C** 或 **0x3D**（主流为 0x3C，7 位地址，无需修改）；
  - 供电电压：3.3V（推荐，兼容 5V）；
  - 通信方式：IIC（仅 SCL/SDA 两根线，无需额外片选线）。
  
  ### 2. 硬件接线要求
  
  IIC 总线为开漏输出特性，**SCL/SDA 必须外接 4.7KΩ 上拉电阻**（核心要求，无拉电阻会导致通信失败），STM32 与 OLED 接线如下（3.3V 供电）：
  
  | STM32 引脚 | OLED 引脚 |                       备注                       |
  | :--------: | :-------: | :----------------------------------------------: |
  |  IIC_SCL   |    SCL    |            外接 4.7KΩ 上拉电阻到 3.3V            |
  |  IIC_SDA   |    SDA    |            外接 4.7KΩ 上拉电阻到 3.3V            |
  |    3.3V    |    VCC    | 屏体供电，禁止接 5V（部分屏体可兼容，建议 3.3V） |
  |    GND     |    GND    |            必须共地，保证电平参考一致            |
  
  ### 3. 接线示例（STM32F103C8T6）
  
  以 I2C1 为例，推荐引脚（可根据硬件修改，需对应修改代码宏定义）：

  STM32 GPIOB6 → OLED SCL（上拉 4.7KΩ 到 3.3V）

  STM32 GPIOB7 → OLED SDA（上拉 4.7KΩ 到 3.3V）

  STM32 3.3V → OLED VCC

  STM32 GND → OLED GND

  ## 四、IIC 配置与 OLED 驱动完整代码示例（STM32F103，HAL 库）
  
  以 STM32F103C8T6 的 I2C1 为例，实现**IIC 底层初始化 + SSD1306 OLED 屏驱动**，支持显示字符、数字、字符串、清屏、光标定位等基础功能，代码包含**头文件宏定义、IIC 底层实现、OLED 驱动封装、主函数测试**。
  
  ### 1. 头文件与宏定义（i2c_oled.h）
  
  包含 IIC 外设、OLED 屏体、显示参数的宏定义，以及所有函数声明，统一管理配置项。
  
  ```
  #ifndef __I2C_OLED_H
  #define __I2C_OLED_H
  
  #include "stm32f1xx_hal.h"
  #include <stdint.h>
  #include <string.h>
  #include <stdio.h>
  
  /************************** IIC外设宏定义（可根据硬件修改） **************************/
  #define I2C_UX            I2C1                    /* 选用I2C1 */
  #define I2C_SCL_GPIO_PORT GPIOB                   /* SCL引脚端口 */
  #define I2C_SCL_GPIO_PIN  GPIO_PIN_6              /* SCL引脚号 */
  #define I2C_SDA_GPIO_PORT GPIOB                   /* SDA引脚端口 */
  #define I2C_SDA_GPIO_PIN  GPIO_PIN_7              /* SDA引脚号 */
  
  /************************** IIC时钟使能宏定义 **************************/
  #define I2C_SCL_GPIO_CLK_ENABLE() __HAL_RCC_GPIOB_CLK_ENABLE()
  #define I2C_SDA_GPIO_CLK_ENABLE() __HAL_RCC_GPIOB_CLK_ENABLE()
  #define I2C_UX_CLK_ENABLE()       __HAL_RCC_I2C1_CLK_ENABLE()
  
  /************************** IIC通信参数宏定义 **************************/
  #define I2C_CLOCK_SPEED    400000U                /* IIC时钟频率，400KHz（快速模式） */
  #define I2C_OWN_ADDRESS1   0x01U                  /* 主机本机地址（任意有效地址即可） */
  #define I2C_TIMEOUT        500U                   /* IIC通信超时时间，ms */
  
  /************************** OLED屏宏定义（SSD1306） **************************/
  #define OLED_I2C_ADDR      0x3C                   /* OLED 7位IIC地址（主流0x3C，部分为0x3D） */
  #define OLED_WIDTH         128                    /* OLED宽度：128列 */
  #define OLED_HEIGHT        64                     /* OLED高度：64行 */
  #define OLED_PAGE_NUM      8                      /* 64行分为8页，每页8行 */
  #define OLED_CMD_MODE      0x00                   /* OLED写命令模式 */
  #define OLED_DATA_MODE     0x40                   /* OLED写数据模式 */
  
  /************************** 全局变量声明 **************************/
  extern I2C_HandleTypeDef g_i2c1_handle;          /* IIC1句柄 */
  
  /************************** 函数声明 **************************/
  /* IIC底层函数 */
  void i2c_init(void);                                                                 /* IIC外设初始化 */
  uint8_t i2c_write_byte(I2C_HandleTypeDef *hi2c, uint16_t dev_addr, uint8_t reg, uint8_t data); /* IIC写1字节 */
  
  /* OLED驱动函数 */
  void oled_init(void);                                                                 /* OLED初始化 */
  void oled_clear(void);                                                                 /* OLED清屏 */
  void oled_refresh(void);                                                               /* OLED刷新显示 */
  void oled_draw_point(uint8_t x, uint8_t y, uint8_t state);                             /* 画点：x列，y行，state=1亮/0灭 */
  void oled_show_char(uint8_t x, uint8_t y, uint8_t ch, uint8_t size);                     /* 显示字符：x列，y行，字符，字号(8/16) */
  void oled_show_string(uint8_t x, uint8_t y, char *str, uint8_t size);                   /* 显示字符串：x列，y行，字符串，字号 */
  void oled_show_num(uint8_t x, uint8_t y, uint32_t num, uint8_t len, uint8_t size);      /* 显示数字：x列，y行，数字，位数，字号 */
  
  #endif /* __I2C_OLED_H */
  ```
  
  ### 2. 字模文件（oled_font.h）
  
  包含 8×8、16×16 ASCII 字符字模（取模方式：**列行式、高位先行、顺向取模**，适配 SSD1306），是 OLED 显示字符的基础，需与`i2c_oled.h`同目录。
  
  ```
  #ifndef __OLED_FONT_H
  #define __OLED_FONT_H
  
  // 8×8 ASCII字符字模（仅数字、字母、常用符号，0x20~0x7F）
  extern const uint8_t oled_font8x8[96][8];
  // 16×16 ASCII字符字模（仅数字、字母、常用符号，0x20~0x7F）
  extern const uint8_t oled_font16x16[96][32];
  
  #endif /* __OLED_FONT_H */
  ```

  **字模说明**：可通过「PCtoLCD2002」取模软件生成，取模参数需严格匹配：

  取模方式：列行式

  字节顺序：高位先行
  
  取模方向：顺向
  
  字体大小：8×8/16×16
  
  编码格式：ASCII
  
  ### 3. IIC 底层与 OLED 驱动实现（i2c_oled.c）
  
  包含 IIC 底层硬件初始化、IIC 通用写函数、SSD1306 初始化、OLED 基础显示函数封装，基于 HAL 库阻塞式 API 开发，稳定易调试。
  
  ```
  #include "i2c_oled.h"
  #include "oled_font.h"
  
  /************************** 全局变量定义 **************************/
  I2C_HandleTypeDef g_i2c1_handle;  /* IIC1句柄 */
  uint8_t g_oled_buf[OLED_WIDTH * OLED_PAGE_NUM] = {0}; /* OLED显示缓冲区（128*8=1024字节） */
  
  /************************** IIC底层硬件初始化（由HAL_I2C_Init自动调用） **************************/
  void HAL_I2C_MspInit(I2C_HandleTypeDef *hi2c)
  {
      GPIO_InitTypeDef gpio_init_struct = {0};
      if (hi2c->Instance == I2C_UX)
      {
          /* 1. 使能GPIO和IIC外设时钟 */
          I2C_SCL_GPIO_CLK_ENABLE();
          I2C_SDA_GPIO_CLK_ENABLE();
          I2C_UX_CLK_ENABLE();
  
          /* 2. 配置SCL/SDA引脚：复用开漏输出 + 上拉 + 高速（IIC标准配置） */
          gpio_init_struct.Pin = I2C_SCL_GPIO_PIN;
          gpio_init_struct.Mode = GPIO_MODE_AF_OD;        /* 复用开漏输出，保证IIC线与特性 */
          gpio_init_struct.Pull = GPIO_PULLUP;            /* 上拉电阻，总线空闲为高电平 */
          gpio_init_struct.Speed = GPIO_SPEED_FREQ_HIGH;  /* 高速模式，适配400KHz */
          HAL_GPIO_Init(I2C_SCL_GPIO_PORT, &gpio_init_struct);
  
          gpio_init_struct.Pin = I2C_SDA_GPIO_PIN;
          HAL_GPIO_Init(I2C_SDA_GPIO_PORT, &gpio_init_struct);
      }
  }
  
  /************************** IIC底层硬件反初始化 **************************/
  void HAL_I2C_MspDeInit(I2C_HandleTypeDef *hi2c)
  {
      if (hi2c->Instance == I2C_UX)
      {
          I2C_UX_CLK_ENABLE(); /* 禁用IIC外设时钟 */
          HAL_GPIO_DeInit(I2C_SCL_GPIO_PORT, I2C_SCL_GPIO_PIN); /* 释放SCL引脚 */
          HAL_GPIO_DeInit(I2C_SDA_GPIO_PORT, I2C_SDA_GPIO_PIN); /* 释放SDA引脚 */
      }
  }
  
  /************************** IIC外设初始化函数 **************************/
  void i2c_init(void)
  {
      /* 配置IIC初始化参数 */
      g_i2c1_handle.Instance = I2C_UX;
      g_i2c1_handle.Init.ClockSpeed = I2C_CLOCK_SPEED;
      g_i2c1_handle.Init.DutyCycle = I2C_DUTYCYCLE_2;         /* 时钟占空比50%，推荐 */
      g_i2c1_handle.Init.OwnAddress1 = I2C_OWN_ADDRESS1;
      g_i2c1_handle.Init.AddressingMode = I2C_ADDRESSINGMODE_7BIT; /* 7位寻址，常用 */
      g_i2c1_handle.Init.DualAddressMode = I2C_DUALADDRESS_DISABLE;
      g_i2c1_handle.Init.OwnAddress2 = 0x00;
      g_i2c1_handle.Init.GeneralCallMode = I2C_GENERALCALL_DISABLE;
      g_i2c1_handle.Init.NoStretchMode = I2C_NOSTRETCH_DISABLE;     /* 使能时钟拉伸 */
  
      /* 初始化IIC外设，失败则死循环 */
      if (HAL_I2C_Init(&g_i2c1_handle) != HAL_OK)
      {
          while(1);
      }
  }
  
  /************************** IIC写1字节数据（OLED专用，适配命令/数据写入） **************************/
  uint8_t i2c_write_byte(I2C_HandleTypeDef *hi2c, uint16_t dev_addr, uint8_t reg, uint8_t data)
  {
      uint8_t buf[2] = {reg, data};
      /* HAL库IIC地址为8位（7位地址+1位读写位），故7位地址左移1位 + 写位(0) */
      return HAL_I2C_Master_Transmit(hi2c, (dev_addr << 1) | 0x00, buf, 2, I2C_TIMEOUT);
  }
  
  /************************** OLED写命令/数据 **************************/
  static void oled_write_cmd(uint8_t cmd)
  {
      i2c_write_byte(&g_i2c1_handle, OLED_I2C_ADDR, OLED_CMD_MODE, cmd);
  }
  static void oled_write_data(uint8_t data)
  {
      i2c_write_byte(&g_i2c1_handle, OLED_I2C_ADDR, OLED_DATA_MODE, data);
  }
  
  /************************** OLED初始化（SSD1306驱动芯片初始化序列） **************************/
  void oled_init(void)
  {
      HAL_Delay(100); /* 上电延时，保证屏体稳定 */
  
      /* SSD1306标准初始化命令序列 */
      oled_write_cmd(0xAE); /* 关闭显示 */
      oled_write_cmd(0xD5); /* 设置时钟分频因子/震荡频率 */
      oled_write_cmd(0x80); /* 分频因子=1，震荡频率默认 */
      oled_write_cmd(0xA8); /* 设置多路复用率 */
      oled_write_cmd(0x3F); /* 64行显示，复用率=63 */
      oled_write_cmd(0xD3); /* 设置显示偏移 */
      oled_write_cmd(0x00); /* 偏移0，无位移 */
      oled_write_cmd(0x40); /* 设置显示开始行 */
      oled_write_cmd(0x8D); /* 使能电荷泵 */
      oled_write_cmd(0x14); /* 开启电荷泵（必须开启，否则无显示） */
      oled_write_cmd(0x20); /* 设置内存地址模式 */
      oled_write_cmd(0x02); /* 页地址模式（常用） */
      oled_write_cmd(0xA1); /* 设置段重映射，0xA1=正常，0xA0=左右翻转 */
      oled_write_cmd(0xC8); /* 设置COM扫描方向，0xC8=正常，0xC0=上下翻转 */
      oled_write_cmd(0xDA); /* 设置COM引脚硬件配置 */
      oled_write_cmd(0x12); /* 交替COM引脚配置 */
      oled_write_cmd(0x81); /* 设置对比度 */
      oled_write_cmd(0xCF); /* 对比度值，0x00~0xFF，越大越亮 */
      oled_write_cmd(0xD9); /* 设置预充电周期 */
      oled_write_cmd(0xF1); /* 预充电周期=15DCLK+1DCLK */
      oled_write_cmd(0xDB); /* 设置VCOMH取消选择级别 */
      oled_write_cmd(0x30); /* VCOMH=0.83*VCC */
      oled_write_cmd(0xA4); /* 全局显示开启，0xA4=正常显示，0xA5=全屏亮 */
      oled_write_cmd(0xA6); /* 正常显示，0xA6=正常，0xA7=反显 */
      oled_write_cmd(0xAF); /* 开启显示 */
  
      oled_clear();  /* 清屏 */
      oled_refresh();/* 刷新显示 */
  }
  
  /************************** OLED清屏（清空显示缓冲区，全灭） **************************/
  void oled_clear(void)
  {
      memset(g_oled_buf, 0x00, sizeof(g_oled_buf)); /* 缓冲区置0，所有点灭 */
  }
  
  /************************** OLED刷新显示（将缓冲区数据写入屏体） **************************/
  void oled_refresh(void)
  {
      uint8_t page, col;
      for (page = 0; page < OLED_PAGE_NUM; page++)
      {
          /* 设置页地址和列地址 */
          oled_write_cmd(0xB0 + page); /* 设置页起始地址（0xB0~0xB7） */
          oled_write_cmd(0x00);        /* 设置列起始地址低4位 */
          oled_write_cmd(0x10);        /* 设置列起始地址高4位 */
          /* 写入当前页的128列数据 */
          for (col = 0; col < OLED_WIDTH; col++)
          {
              oled_write_data(g_oled_buf[page * OLED_WIDTH + col]);
          }
      }
  }
  
  /************************** OLED画点（基础函数，为字符显示提供支撑） **************************/
  void oled_draw_point(uint8_t x, uint8_t y, uint8_t state)
  {
      if (x >= OLED_WIDTH || y >= OLED_HEIGHT) return; /* 超出范围，直接返回 */
      uint8_t page = y / 8;    /* 计算点所在的页（每页8行） */
      uint8_t bit = y % 8;     /* 计算点在页内的位（0~7） */
      if (state)
      {
          g_oled_buf[page * OLED_WIDTH + x] |= (1 << bit); /* 置1，点亮 */
      }
      else
      {
          g_oled_buf[page * OLED_WIDTH + x] &= ~(1 << bit);/* 置0，熄灭 */
      }
  }
  
  /************************** OLED显示单个字符（8×8/16×16字号） **************************/
  void oled_show_char(uint8_t x, uint8_t y, uint8_t ch, uint8_t size)
  {
      if (x >= OLED_WIDTH || y >= OLED_HEIGHT || (size != 8 && size != 16)) return;
      ch -= 0x20; /* 字模从0x20（空格）开始，偏移校正 */
      uint8_t i, j;
      if (size == 8)
      {
          /* 8×8字号，1行8列 */
          for (i = 0; i < 8; i++)
          {
              uint8_t dat = oled_font8x8[ch][i];
              for (j = 0; j < 8; j++)
              {
                  oled_draw_point(x + j, y + i, (dat >> j) & 0x01);
              }
          }
      }
      else
      {
          /* 16×16字号，2行16列 */
          for (i = 0; i < 16; i++)
          {
              uint8_t dat = oled_font16x16[ch][i];
              for (j = 0; j < 8; j++)
              {
                  oled_draw_point(x + j, y + i, (dat >> j) & 0x01);
              }
              dat = oled_font16x16[ch][i + 16];
              for (j = 0; j < 8; j++)
              {
                  oled_draw_point(x + 8 + j, y + i, (dat >> j) & 0x01);
              }
          }
      }
  }
  
  /************************** OLED显示字符串（基于字符显示封装） **************************/
  void oled_show_string(uint8_t x, uint8_t y, char *str, uint8_t size)
  {
      uint8_t x0 = x;
      while (*str)
      {
          if (x >= OLED_WIDTH) /* 超出列范围，换行 */
          {
              x = 0;
              y += size;
              if (y >= OLED_HEIGHT) break; /* 超出行范围，退出 */
          }
          oled_show_char(x, y, *str++, size);
          x += size; /* 字符间距=字号，可自定义调整 */
      }
  }
  
  /************************** OLED显示数字（支持无符号整数，任意位数） **************************/
  void oled_show_num(uint8_t x, uint8_t y, uint32_t num, uint8_t len, uint8_t size)
  {
      uint8_t i, digit;
      uint8_t buf[16] = {0};
      /* 数字转字符数组，逆序存储 */
      for (i = 0; i < len; i++)
      {
          buf[i] = num % 10 + '0';
          num /= 10;
      }
      /* 正序显示 */
      for (i = len; i > 0; i--)
      {
          oled_show_char(x + (len - i) * size, y, buf[i - 1], size);
      }
  }
  ```
  
  ### 4. 主函数测试（main.c）
  
  实现系统初始化、IIC 初始化、OLED 初始化，并测试 OLED 的**字符、字符串、数字显示**功能，包含 LED 状态指示，验证程序正常运行。
  
  ```
  #include "stm32f1xx_hal.h"
  #include "i2c_oled.h"
  #include "led.h"
  #include <stdio.h>
  
  /************************** printf重定向（串口调试，可选） **************************/
  // 若需串口打印，需提前实现usart_init()，此处为示例
  // int fputc(int ch, FILE *f)
  // {
  //     HAL_UART_Transmit(&g_uart1_handle, (uint8_t *)&ch, 1, 0xFFFF);
  //     return ch;
  // }
  
  /************************** 主函数 **************************/
  int main(void)
  {
      uint32_t cnt = 0; /* 计数变量，用于测试数字显示 */
  
      /* 1. 系统底层初始化 */
      HAL_Init();                              /* HAL库初始化（定时器、中断等） */
      sys_stm32_clock_init(RCC_PLL_MUL9);      /* 配置系统时钟为72MHz（STM32F103标配） */
      delay_init(72);                          /* 延时函数初始化，入参为主频72MHz */
      led_init();                              /* LED初始化（状态指示，可选） */
      i2c_init();                              /* IIC1初始化（400KHz） */
      oled_init();                             /* OLED屏初始化（SSD1306） */
  
      /* 2. OLED显示初始化信息 */
      oled_show_string(0, 0, "STM32 IIC OLED", 16);  /* 第一行：字符串，16×16字号 */
      oled_show_string(0, 20, "128*64 SSD1306", 16); /* 第二行：字符串，16×16字号 */
      oled_show_string(0, 40, "Count: ", 16);         /* 第三行：固定字符串 */
      oled_refresh();                               /* 刷新显示到屏体 */
  
      /* 主循环：持续更新数字显示，LED翻转指示系统运行 */
      while (1)
      {
          oled_show_num(64, 40, cnt, 5, 16); /* 显示计数，5位数字，16×16字号 */
          oled_refresh();                    /* 刷新显示 */
          cnt++;                             /* 计数自增 */
          if (cnt > 99999) cnt = 0;          /* 溢出清零 */
  
          LED0_TOGGLE(); /* LED翻转（如PA8），指示系统正常运行 */
          HAL_Delay(500);/* 延时500ms，数字每秒更新2次 */
      }
  }
  ```
  
  ## 五、核心 HAL 库 IIC API 说明
  
  本文基于 HAL 库**阻塞式 IIC API**开发，该类 API 简单稳定、易调试，无需处理中断 / DMA，适配 OLED、传感器等低速外设场景，核心常用 API 如下：
  
  |           函数名            |                   功能描述                    |                适用场景                 |
  | :-------------------------: | :-------------------------------------------: | :-------------------------------------: |
  |      `HAL_I2C_Init()`       | 初始化 IIC 外设，配置通信参数（时钟、地址等） |           IIC 外设初始化阶段            |
  |     `HAL_I2C_MspInit()`     |   IIC 底层硬件初始化（GPIO / 时钟 / 中断）    |  被`HAL_I2C_Init()`自动调用，用户实现   |
  | `HAL_I2C_Master_Transmit()` |         主机向从设备发送指定长度数据          | IIC 通用写操作（如 OLED 写命令 / 数据） |
  | `HAL_I2C_Master_Receive()`  |         主机从从设备接收指定长度数据          |   IIC 通用读操作（如传感器数据读取）    |
  |  `HAL_I2C_Check_Address()`  |      检测总线上指定地址的从设备是否存在       |       设备挂载检测、IIC 地址调试        |
  |     `HAL_I2C_DeInit()`      |          反初始化 IIC 外设，释放资源          |          外设停用、低功耗配置           |
  
  ### 关键地址处理说明
  
  HAL 库中 IIC 的设备地址为**8 位格式**（7 位设备地址 + 1 位读写位），而手册中给出的 OLED 地址为 7 位（如 0x3C），因此使用时需做如下转换：
  
  **写操作**：7 位地址 << 1 + 0x00（写位为 0），例：0x3C << 1 = 0x78；
  
  **读操作**：7 位地址 << 1 + 0x01（读位为 1），例：0x3C << 1 | 0x01 = 0x79；
  
  本文代码中`i2c_write_byte`函数已自动处理该转换，用户只需传入 7 位 OLED 地址（0x3C/0x3D）即可。