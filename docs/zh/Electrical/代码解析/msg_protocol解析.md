> 更新日期：2026/2/1
>
> 参与者：KkarinL15
>
> 源码地址：[message-protocol](https://github.com/XJU-Hurricane-Team/message-protocol)

### 函数调用

在初始任务中

- 调用`message_register_send_uart(msg_id_t msg_id, UART_HandleTypeDef *huart,uint32_t buf_size)`注册串口消息通讯句柄；
- 调用`message_register_polling_uart(msg_id_t msg_id, UART_HandleTypeDef *huart,uint32_t buf_size, uint32_t fifo_size)`注册数据接收串口句柄；
- 调用`message_register_recv_callback(msg_id_t msg_id,msg_recv_callback_t msg_callback)`注册接收回调函数指针；
- 完成以上注册后创建`message_polling_task`任务为后续轮询函数做准备。

调用`message_send_data`函数发送数据。

在`message_polling_task`任务中调用`message_polling_data`函数轮询消息队列读取数据。

### 数据结构体

#### `msg_fifo_t`

环形缓冲区

```
typedef struct {
    uint32_t size;          /*!< 缓冲区大小 */
    uint32_t mask;          /*!< 大小掩码 */
    uint8_t frame_len;      /*!< 帧长度 */
    bool new_frame;         /*!< 是否是新的一帧 (写长度用) */
    volatile uint32_t head; /*!< 头指针 */
    volatile uint32_t tail; /*!< 尾指针 */
    uint8_t buf[0];         /*!< 缓冲区 */
} msg_fifo_t;
```

#### `msg_instance`

消息队列管理结构

```
struct msg_instance {
    msg_recv_callback_t recv_callback; /*!< 接收回调函数 */
    UART_HandleTypeDef *send_uart;     /*!< 发送串口句柄 */
    UART_HandleTypeDef *recv_uart;     /*!< 接收串口句柄 */

    uint8_t *send_buf;     /*!< 发送缓冲区 */
    uint32_t send_buf_len; /*!< 发送缓冲区大小 */

#if MSG_ENABLE_RTOS
    SemaphoreHandle_t send_buf_semp; /*!< 发送缓冲区二值型号量 */
#endif                               /* MSG_ENABLE_RTOS */

    uint8_t *recv_buf;      /*!< 接收缓冲区 */
    uint32_t recv_buf_size; /*!< 接收缓冲区大小 */

#ifdef MSG_ESC
    bool escape; /*!< 是否要将下一个字符转义 */
#endif           /* MSG_ESC */

    msg_fifo_t *fifo;          /*!< 接收缓冲区 */
    uint32_t fifo_element_len; /*!< 当前队列元素个数 */

#if MSG_ENABLE_STATISTICS
    uint32_t send_count; /*!< 发送计数 */

    uint32_t recv_success; /*!< 接收成功计数 */
    uint32_t recv_error;   /*!< 接收错误计数 */
#if MSG_ENABLE_CRC8
    uint32_t crc_check_error; /*!< CRC 校验错误计数 */
#endif                        /* MSG_ENABLE_CRC8 */

    uint32_t max_fifo_element_len; /*!< 最大队列元素个数 */
    uint32_t fifo_overflow;        /*!< 队列溢出清空计数 */
#endif                             /* MSG_ENABLE_STATISTICS */
};
```

### 各项辅助功能

- 启用线程安全处理：`MSG_ENABLE_RTOS`

  启用后会使用互斥信号量来保护发送缓冲区（仅支持`FreeRTOS`）。

- 启用转义标识：`MSG_ESC`

  为了做到透传（即原样、完整地将数据从发送端转发至接收端），消息会对内容转义。定义`MSG_ESC`可以选择转义字符（建议选择出现频次低的字节）。

- 启用始能统计：`MSG_ENABLE_STATISTICS`

  统计每种消息的接收情况（接收成功、错误计数，内存分配失败计数，队列长度与最大深度等）。

- 启用`CRC`校验：`MSG_ENABLE_CRC8`

  检验传入数据和数据长度。（需接收方与发送方同时开启，否则数据解析时数据会被误判为脏数据）

  `CRC8`和`CRC16`的区别在于校验长度有区别和返回的校验值位数不同；`CRC8`的校验码为`1byte`，`CRC16`的校验码为`2byte`。

#### 辅助功能的启用

宏定义`MSG_ENABLE_RTOS`为1

- 在`message_register_send_uart`中使用信号量保护发送缓存区

  ```
  struct msg_instance *msg = msg_list[msg_id];
  if (msg->send_buf_semp == NULL) {
          msg->send_buf_semp = xSemaphoreCreateMutex();
      }
  ```

- 在`message_send_data`中

  ```
  xSemaphoreTake(msg->send_buf_semp, portMAX_DELAY);//用信号量等待获取发送权限
  									.
  									.
  									.
  xSemaphoreGive(msg->send_buf_semp);//释放信号量
  ```

宏定义`MSG_ENABLE_CRC8`为1

- 在`message_send_data`中

  得到`CRC8`校验值

  ```
  #if MSG_ENABLE_CRC8
      /* CRC8 校验结果 */
      uint8_t crc8_value = calc_crc8(data, data_len);
  #endif /* MSG_ENABLE_CRC8 */
  ```

  将`CRC8`校验值拆分为两个字节存入数据帧

  ```
  #if MSG_ENABLE_CRC8
      /* 添加 CRC8 帧校验数据, 拆成两个字节, 每个字节小于 0x10, 这样可以避免转义 */
      send_buf[buf_idx] = (crc8_value >> 4) & 0x0F;
      ++buf_idx;
      send_buf[buf_idx] = (crc8_value & 0x0F);
      ++buf_idx;
  #endif /* MSG_ENABLE_CRC8 */
  ```

- 在`message_data_dequeue`中

  定义`CRC8`校验值变量

  ```
  #if MSG_ENABLE_CRC8
      /* 接收到的 CRC8 校验值 */
      uint8_t crc_recv;
      /* 计算得到的 CRC8 校验值 */
      uint8_t crc_value;
  #endif /* MSG_ENABLE_CRC8 */
  ```

  检验数据包长度与实际接收长度是否一致

  ```
  /* 验证数据包长度与实际接收长度是否一致, 数据包第二个字节是长度 */
  #if MSG_ENABLE_CRC8
          /* 1 byte 标识, 1 byte 长度, 1 byte 结束符, 1 byte FIFO 元素大小
           * 2 byte CRC8 校验值
           * 总共 6 byte. */
          if ((frame_len - 6) != fifo->buf[(fifo->head + 2) & fifo->mask]) {
  #else  /* MSG_ENABLE_CRC8 */
          /* 1 byte 标识, 1 byte 长度, 1 byte 结束符, 1 byte FIFO 元素大小
           * 总共 4 byte. */
          if ((frame_len - 4) != fifo->buf[(fifo->head + 2) & fifo->mask]) {
  #endif /* MSG_ENABLE_CRC8 */
  ```

  当数据被分成两段时如下处理（再进行校验）

  ```
  #if MSG_ENABLE_CRC8
                  call_len = frame_len - 6;
  #else  /* MSG_ENABLE_CRC8 */
                  call_len = frame_len - 4;
  #endif /* MSG_ENABLE_CRC8 */
  ```

  校验`CRC8`

  ```
  #if MSG_ENABLE_CRC8
          /* 校验 CRC8 */
          crc_value = calc_crc8(call_data, call_len);
          /* 接收到的 CRC8 校验值 */
          crc_recv =
              (call_data[frame_len - 5] & 0x0F) | (call_data[frame_len - 6] << 4);
          if (crc_value != crc_recv) {
              /* 校验结果不一致, 出队到下一个 */
              fifo->head += frame_len;
              --msg->fifo_element_len;
  #if MSG_ENABLE_STATISTICS
              ++msg->crc_check_error;
  #endif /* MSG_ENABLE_STATISTICS */
              continue;
          }
  #endif /* MSG_ENABLE_CRC8 */			
  ```

### 数据发送

流程如下：

![](.\Picture\msg_send.png)

#### 扩缩缓存区

```
if (msg->send_buf_len <= data_len + 5) {
        /* 不够 */
        uint8_t *new_buf = (uint8_t *)MSG_REALLOC(msg->send_buf, data_len * 2);
        if (new_buf == NULL) {
            return;
        }

        msg->send_buf = new_buf;
        msg->send_buf_len = data_len * 2;
    } else if ((data_len + 3) * 3 <= msg->send_buf_len) {
        /* 有余*/
        uint8_t *new_buf =
            (uint8_t *)MSG_REALLOC(msg->send_buf, msg->send_buf_len / 2);
        if (new_buf == NULL) {
            return;
        }

        msg->send_buf = new_buf;
        msg->send_buf_len = msg->send_buf_len / 2;
    }
```

- 不够（缓存区长度 <= 数据长度+5）：将缓存区的长度扩为当前数据的两倍。
- 有余（（数据长度+3）*3 <= 缓存区长度）：将缓存区的长度缩到原来的1/2。

#### 数据拆分

缓存区数组（`uint8_t`类型）第一字节高四位标记数据`ID`，低四位标记数据类型；第二字节标记数据长度；第三字节开始将数据复制到字节流；最后一个字节用帧结束标志`MSG_EOF`结尾。

### 轮询接收的数据

流程如下：

![](.\Picture\msg-rec.png)

#### 消息数据入队

把从串口读到的字节写入环形 `FIFO`，遇到 `MSG_EOF` 完成一帧并在帧前预留的位置写入该帧在 `FIFO` 中的总长度。

```
if (fifo->new_frame) {
            /* 空一个字节写长度, 长度写 0 */
            fifo->buf[fifo->tail & fifo->mask] = 0;
            ++fifo->tail;
            fifo->new_frame = false;
        }

        /* 将数据写入队列 */
        fifo->buf[fifo->tail & fifo->mask] = msg->recv_buf[i];
        ++fifo->tail;
        ++fifo->frame_len;

        if (fifo->size == (fifo->tail - fifo->head)) {
            /* FIFO 已满, 先覆盖旧数据, 有新的帧直接清空 FIFO, 从头开始存 */
            fifo->head = 0;
            fifo->tail = 0;
            msg->fifo_element_len = 0;
            fifo->frame_len = 0;
```

环形缓冲区，注册数据接口时传入的队列大小参数必须为2的幂次方。代码里用位掩码替代取模来做索引环绕：`msg_fifo_init` 把 `mask = fifo_size - 1`，然后在读写时用 `index & mask` 访问缓冲区（例如 `fifo->buf[fifo->tail & fifo->mask]`）。只有当 `fifo_size` 是 2 的幂时，`fifo_size-1` 的二进制是低位全 1，`index & mask` 等价于 `index % fifo_size`，才能正确、快速地做环绕索引。若不是 2 的幂，`& mask` 不再等同于取模，会导致索引错误和数据混乱。

#### 回调函数

回调函数参数形式必须是`void func(uint32_t, uint8_t, uint8_t*)`第一个参数是消息长度, 第二个参数是消息标识 (高四位是 ID, 低四位是数据类型)，第三个参数是数据区内容, 无返回值。

#### 消息数据出队

从 `FIFO`按帧读取数据，校验帧长度（和 `CRC8` 可选），把完整帧拷贝到 `recv_buf`（处理跨界情形），并在校验通过时调用注册的回调 `msg->recv_callback`对消息`ID`调用相应回调函数。
