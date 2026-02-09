# LADRC代码实战

> **最近修改日期**：2026-02-07
> **参与者**：Jackrainman
> **前置知识**：[LADRC.md](../ControlTheory/LADRC.md)
> **注**：ladrc 与 td 已分离，文档尚未更新

---

## 目录

1. [前置知识](#前置知识)
2. [第一部分：跟踪微分器(TD)实现](#第一部分跟踪微分器td实现)
3. [第二部分：一阶LADRC实现](#第二部分一阶ladrc实现)
4. [第三部分：二阶LADRC实现](#第三部分二阶ladrc实现)
5. [第四部分：完整使用示例](#第四部分完整使用示例)
6. [第五部分：调试技巧与常见问题](#第五部分调试技巧与常见问题)

---

## 前置知识

在开始编写代码之前，请确保你已经理解以下内容：

| 知识点           | 说明               | 参考文档                                 |
| ------------- | ---------------- | ------------------------------------ |
| **LADRC基本原理** | 总扰动、LESO、LSEF的概念 | [LADRC.md](../ControlTheory/LADRC.md) |
| **离散化方法**     | 零阶保持法、欧拉法        | 本文相关章节                               |
| **C语言基础**     | 结构体、指针、基本运算      | -                                    |

**本文代码特点**：
- 使用 **C语言** 编写（适用于嵌入式平台如STM32、DSP等）
- 浮点数使用 `float` 类型（可根据需要改为 `double`）
- **v3.1版本采用参数固化模式**：采样周期在初始化时固化到结构体中，消除时间抖动对积分的影响

---

## 第一部分：跟踪微分器(TD)实现

### 1.1 TD的作用

跟踪微分器（Tracking Differentiator, TD）有两个核心功能：

1. **安排过渡过程**：将跳变的输入信号平滑化，避免控制量突变
2. **提取微分信号**：从有噪声的信号中提取可靠的微分信号

**为什么需要TD？**

假设输入信号从0跳变到100，如果不加处理直接输入控制器，会产生巨大的控制量突变，可能导致：
- 执行机构（如电机）过流损坏
- 系统振荡甚至失稳
- 机械结构受到冲击

TD通过安排合理的过渡过程，让信号"平滑地"到达目标值。

### 1.2 TD结构体定义

```c
/**
 * @brief 跟踪微分器结构体
 * @note  二阶TD，用于安排过渡过程和提取微分信号
 *        采用参数固化模式，采样周期在初始化时确定
 */
typedef struct {
    float x1;       // 跟踪信号（位置）- 平滑后的目标值输出
    float x2;       // 微分信号（速度）- 目标值的变化率
    float r;        // 快速跟踪因子（相当于最大加速度）
    float h;        // 积分步长（采样周期）- 【固化参数】RTOS任务周期
    float h0;       // 滤波因子（用于输入滤波，h0 = N * h）
    float max_x2;   // 最大速度限制（0表示不限制）
} td_t;
```

### 1.3 TD初始化函数

```c
/**
 * @brief TD 初始化 - 参数固化模式
 *
 * @param td      TD结构体指针
 * @param r       快速跟踪因子，决定跟踪速度
 * @param dt      固定采样周期(秒) - RTOS任务周期，将固化到结构体中
 * @param n       滤波因子(无量纲) - 建议值1~5，值越大滤波越强但延迟越大
 * @param max_x2  最大速度限制（0表示不限制）
 *
 * @note 参数固化模式的核心逻辑：
 *       1. 固化采样周期：td->h = dt
 *       2. 自动计算内部滤波参数：td->h0 = n * dt
 *       3. 安全检查：如果 n < 1.0，强制设为 1.0，防止数学模型崩溃
 */
void td_init(td_t *td, float r, float dt, float n, float max_x2) {
    /* 安全性检查：滤波因子N不能小于1.0，否则数学模型会崩溃 */
    if (n < 1.0f) {
        n = 1.0f;
    }

    td->r = r;
    td->h = dt;         /* 固化采样周期到结构体中 */
    td->h0 = n * dt;    /* 自动计算内部滤波参数 h0 = N * dt */
    td->max_x2 = max_x2;
    td->x1 = 0.0f;
    td->x2 = 0.0f;
}
```

**参数说明**：

| 参数         | 符号        | 作用              | 调节建议                                                    |
| ---------- | --------- | --------------- | ------------------------------------------------------- |
| **快速跟踪因子** | $r$       | 决定跟踪速度，相当于最大加速度 | $r$ 越大，跟踪越快，但可能超调。建议初始值：$r = 1/T_{settle}$ ，就是 $一秒算多少次$ |
| **滤波因子**   | $N$       | 决定平滑程度（无量纲）     | 建议值1~5。N=1: 滤波最弱响应最快；N=3~5: 典型推荐值；N>10: 强滤波但滞后明显        |
| **最大速度**   | $max\_x2$ | 限制输出速度          | 根据执行器最大速度设置，防止目标变化过快                                    |

**为什么采用参数固化模式？**

在RTOS环境中，任务调度可能存在抖动。如果在每次`td_update`时传入不同的`dt`，会导致积分步长不一致，产生累积误差。参数固化模式将`dt`在初始化时保存到结构体中，确保每个控制周期使用相同的时间基准，提高系统稳定性。

### 1.4 fhan最速控制综合函数

``c
/**
 * @brief fhan 最速控制综合函数（梯形加速度曲线）
 *
 * 这是韩京清教授提出的最速控制综合函数，用于实现时间最优控制。
 * 产生的加速度曲线是梯形的（加速-匀速-减速）。
 *
 * @param x1 位置误差
 * @param x2 速度
 * @param r 快速跟踪因子（相当于最大加速度）
 * @param h 积分步长（采样周期）
 * @param h0 滤波因子（用于输入滤波，典型值为h的5-10倍）
 * @return 加速度输出
 */
float td_fhan(float x1, float x2, float r, float h, float h0) {
    float d = r * h;                                // 单步速度变化量 (delta v)
                                                    // d = r·h 是离散系统能感知的最小速度单位

    float d0 = h0 * d;                              // 线性区宽度
                                                    // 当误差|y| <= d0时，系统进入线性区平滑处理

    float y = x1 + h * x2;                          /* 预测：当前位置 + h×速度 = 不加控制时的未来位置
                                                     * 产生"超前意识"，抵消离散系统的相位滞后，是不超调的第一道防线 */

    /* 安全检查：限制 y 的范围防止 sqrtf 溢出 */
    const float MAX_Y = 1e15f;
    if (fabsf(y) > MAX_Y) {
        y = (y > 0.0f) ? MAX_Y : -MAX_Y;
    }

    float sqrt_arg = d * d + 8.0f * r * fabsf(y);   // 离散刹车曲线方程的核心
                                                    // 源于等差数列求和公式，8 = 4×2 是离散求和系数

    /* 安全检查：确保 sqrtf 参数非负且有限 */
    if (sqrt_arg < 0.0f || !isfinite(sqrt_arg)) {
        sqrt_arg = 0.0f;
    }

    float a0 = sqrtf(sqrt_arg);                     /* 状态解算指标
                                                     * 代表在当前位置误差y下，系统若想最快停下，
                                                     * 理想中应该具备的"速度量级" */

    float a;                                        // 综合切换指标
    if (fabsf(y) <= d0) {                           // 线性区：系统非常靠近目标
        a = x2 + y / h;                             /* 预测下一时刻刚好归零的逻辑
                                                     * 此时fhan退化为PD控制器结构
                                                     * P项: -y/h², D项: -x2/h */
    } else {                                        // 非线性区：系统离目标较远（赶路）
        a = x2 + 0.5f * (a0 - d) * ((y > 0) ? 1.0f : -1.0f);
                                                    /* 0.5*(a0-d) 是基于当前剩余距离y，
                                                     * 计算出的"当前时刻应该具有的临界速度"
                                                     * 系统始终保持在最速控制切换曲线上 */
    }

    float fhan_out;                                 // 加速度输出
    if (fabsf(a) <= d) {                            // 线性插值区（接近停稳）
        fhan_out = -r * a / d;                      /* 比例缩放：加速度随靠近目标而逐渐减小
                                                     * 如果不做这个线性处理，最后一步可能输出过大加速度导致跨过原点
                                                     * 通过a/d比例缩放，最终在目标点刚好减为0 */
    } else {                                        // 饱和区（满速加速/刹车）- Bang-Bang控制
        fhan_out = -r * ((a > 0) ? 1.0f : -1.0f);   /* 只有+r或-r两种状态
                                                     * 保证最快响应，油门踩到底或刹车踩到底 */
    }

    return fhan_out;
}
```

**fhan函数工作原理简述**：

fhan函数是TD的核心，它实现了**离散时间下的最速控制**。算法根据当前位置误差和速度，实时计算最优加速度：

1. **预测未来位置** `y = x1 + h*x2`：考虑离散时间滞后，预测下一时刻的位置
2. **计算刹车曲线** `a0 = sqrt(d² + 8r|y|)`：根据离散系统特性，计算安全刹车所需的速度指标
3. **分区控制**：
   - **非线性区**（远离目标）：Bang-Bang控制，全力加速或全力刹车，追求最速
   - **线性区**（接近目标）：PD控制，平滑收敛，防止超调

**关键切换逻辑**：
- 当 `a < 0`：当前速度小于刹车曲线要求的速度 → **全力加速**
- 当 `a = 0`：当前速度等于刹车曲线要求的极限速度 → **切换时刻，开始刹车**
- 当 `a > 0`：当前速度超过刹车曲线要求的速度 → **全力刹车**

### 1.5 TD更新计算函数

``c
/**
 * @brief TD 更新计算 - 参数固化模式
 *
 * @param td      TD结构体指针
 * @param target  目标值
 * @return        跟踪输出x1（平滑后的目标值）
 *
 * @note 参数固化模式：此函数使用结构体中固化的采样周期td->h，
 *       不再接受外部传入的dt。这适用于RTOS固定频率调用的场景，
 *       可以消除时间抖动对积分的影响。
 */
float td_update(td_t *td, float target) {
    float x1_error = td->x1 - target;               // 计算当前跟踪误差

    /* 使用固化的采样周期td->h进行计算，不再依赖外部传入的dt */
    float fh = td_fhan(x1_error, td->x2, td->r, td->h, td->h0);
                                                    // 调用fhan计算最优加速度

    /* 使用固化的td->h进行积分，确保时间一致性 */
    float new_x2 = td->x2 + td->h * fh;             // 速度积分：v = v0 + a·dt

    /* 速度限制检查 */
    if (td->max_x2 > 0.0f) {
        if (new_x2 > td->max_x2) {
            new_x2 = td->max_x2;
        } else if (new_x2 < -td->max_x2) {
            new_x2 = -td->max_x2;
        }
    }

    td->x2 = new_x2;                                // 更新速度状态
    /* 使用固化的td->h进行位置积分 */
    td->x1 = td->x1 + td->h * td->x2;               // 位置积分：x = x0 + v·dt

    return td->x1;                                  // 返回平滑后的目标值
}
```

**为什么td_update不再传dt？**

在参数固化模式下，`td->h`在初始化时就已经确定。这样做有两个好处：
1. **消除时间抖动**：避免RTOS调度抖动导致积分步长不一致
2. **简化调用**：调用者只需传入目标值，无需关心采样周期

### 1.6 TD重置函数

```c
/**
 * @brief TD 重置状态
 *
 * @param td         TD结构体指针
 * @param init_value 初始值
 *
 * @note 在系统复位、模式切换或故障恢复时调用，
 *       将TD状态重置为指定值，避免历史状态影响新的控制过程
 */
void td_reset(td_t *td, float init_value) {
    td->x1 = init_value;                            // 位置重置为初始值
    td->x2 = 0.0f;                                  // 速度重置为0
}
```

### 1.7 TD使用示例

```c
// 定义TD结构体
td_t my_td;

// 初始化：快速跟踪因子r=100，采样周期dt=0.001s，滤波因子N=5，最大速度限制50
// N = 5 是典型设置，h0 = 5 * dt 会在内部自动计算
td_init(&my_td, 100.0f, 0.001f, 5.0f, 50.0f);

// 主循环（每1ms执行一次）
while(1) {
    float target = Get_Target();                    // 获取目标值
    float smooth_target = td_update(&my_td, target);  // 更新TD（无需传入dt）
    float target_speed = my_td.x2;                  // 获取目标值的变化率

    // 将smooth_target和target_speed用于后续控制...

    Delay_ms(1);                                    // 等待下一个周期
}

// 如果需要重置TD状态（如系统复位时）
td_reset(&my_td, 0.0f);  // 将位置重置为0，速度重置为0
```

---

## 第二部分：一阶LADRC实现

### 2.1 一阶LADRC原理

一阶LADRC适用于被控对象为一阶的系统（如电流控制、简单速度控制等）：

**被控对象模型**：

$$
\dot{x} = f(x, w, t) + b \cdot u
$$

其中 $f(x,w,t)$ 为总扰动（包含模型不确定性和外部扰动），$b$ 为控制增益。

一阶LADRC使用**二阶ESO**（估计状态和总扰动）。

### 2.2 一阶LADRC结构体定义

``c
/**
 * @brief 一阶LADRC结构体
 * @note  适用于电流/简单速度控制等一阶系统
 *        采用组合模式，内嵌TD模块
 */
typedef struct {
    // ESO增益（二阶观测器）
    float beta1;        // ESO增益beta1 - 位置观测带宽
    float beta2;        // ESO增益beta2 - 扰动观测带宽

    // 控制器增益
    float kp;           // 比例增益 - 控制器刚度

    // 系统参数
    float b;            // 控制增益(b0) - 决定控制量的缩放比例
    float dt;           // 采样周期(秒) - 【固化参数】RTOS固定周期

    // 状态估计值
    float x1;           // 估计的系统输出（跟踪测量值）
    float x2;           // 估计的总扰动（包含内部动态和外部扰动）

    // 输出限制
    float max_output;   // 输出限幅值（如PWM最大值）

    // 抗积分饱和
    float k_aw;         // 抗积分饱和增益（0表示不使用，建议值1.0~3.0）
    float pre_out;      // 上一时刻输出（用于计算饱和程度）

    // 输出
    float out;          // 当前输出

    // TD组合模式（v3.1新增）
    td_t td;            // 内嵌TD结构体
    bool use_td;        // 是否使用TD（在初始化时根据td_r参数决定）
} first_order_ladrc_t;
```

### 2.3 一阶LADRC初始化函数

``c
/**
 * @brief 一阶LADRC初始化 - 参数固化+TD组合模式
 *
 * @param fladrc     一阶LADRC结构体指针
 * @param max_output 控制量输出限幅（例如PWM最大值）
 * @param beta1      ESO状态观测器增益1（位置观测带宽）
 * @param beta2      ESO状态观测器增益2（扰动观测带宽）
 * @param kp         控制器比例增益（刚度）
 * @param b          系统增益估计值(b0) - 决定控制量的缩放比例
 * @param dt         RTOS固定采样周期(秒) - 必须与实际任务频率一致
 * @param k_aw       抗积分饱和增益（建议值1.0~3.0，0表示不启用）
 * @param td_r       TD快速跟踪因子(r) - 决定目标值响应速度
 *                   0表示禁用TD（直接透传目标值）
 * @param td_n       TD滤波因子(无量纲，建议值1~5) - 决定噪声过滤能力
 *                   N=1: 滤波最弱响应最快；N=3~5: 典型推荐值；N>10: 强滤波但滞后明显
 * @param td_max_x2  TD最大速度限制(0表示不限制) - 防止设定值跳变过大导致系统冲击
 */
void first_order_ladrc_init(first_order_ladrc_t *fladrc, float max_output,
                           float beta1, float beta2, float kp, float b,
                           float dt, float k_aw,
                           float td_r, float td_n, float td_max_x2) {

    /* 初始化ESO参数 */
    fladrc->beta1 = beta1;
    fladrc->beta2 = beta2;

    /* 初始化控制器参数 */
    fladrc->kp = kp;
    fladrc->b = b;

    /* 初始化状态估计值 */
    fladrc->x1 = 0.0f;                              // 输出估计清零
    fladrc->x2 = 0.0f;                              // 扰动估计清零

    /* 初始化输出限制 */
    fladrc->max_output = max_output;

    /* 初始化抗积分饱和参数 */
    fladrc->k_aw = k_aw;
    fladrc->pre_out = 0.0f;                         // 上一时刻输出清零

    /* 初始化采样周期 - 固化到结构体中 */
    fladrc->dt = dt;                                // 【关键】后续计算都使用这个固化的dt

    /* 初始化输出 */
    fladrc->out = 0.0f;

    /* 初始化TD（组合模式）- 使用参数固化模式 */
    if (td_r > 0.0f) {
        td_init(&fladrc->td, td_r, dt, td_n, td_max_x2);
                                                    // 调用TD初始化，传入滤波因子N
        fladrc->use_td = true;                      // 启用TD
    } else {
        fladrc->use_td = false;                     // 禁用TD，目标值直接透传
    }
}
```

### 2.4 一阶LADRC参数重置函数

``c
/**
 * @brief 一阶LADRC参数重置 - 支持热更新
 *
 * @note 在系统运行过程中动态调整参数，同时重置ESO状态避免瞬态问题
 *       常用于自适应控制、参数调度等场景
 */
void first_order_ladrc_reset(first_order_ladrc_t *fladrc,
                             float beta1, float beta2, float kp, float b,
                             float dt, float k_aw,
                             float td_r, float td_n, float td_max_x2) {
    /* 更新控制参数 */
    fladrc->beta1 = beta1;
    fladrc->beta2 = beta2;
    fladrc->kp = kp;
    fladrc->b = b;
    fladrc->dt = dt;
    fladrc->k_aw = k_aw;

    /* 重置 ESO 状态，避免参数切换时的瞬态问题 */
    fladrc->x1 = 0.0f;
    fladrc->x2 = 0.0f;
    fladrc->pre_out = 0.0f;

    /* 重新配置TD参数 - 使用参数固化模式 */
    if (td_r > 0.0f) {
        /* 安全性检查：滤波因子N不能小于1.0 */
        float n = td_n;
        if (n < 1.0f) {
            n = 1.0f;
        }
        fladrc->td.r = td_r;
        fladrc->td.h = dt;          // 固化采样周期
        fladrc->td.h0 = n * dt;     // 计算内部滤波参数 h0 = N * dt
        fladrc->td.max_x2 = td_max_x2;
        fladrc->use_td = true;
    } else {
        fladrc->use_td = false;
    }
}
```

### 2.5 一阶LADRC计算函数

``c
/**
 * @brief 一阶LADRC计算函数 - 核心控制算法
 *
 * @param fladrc  一阶LADRC结构体指针
 * @param target  目标值
 * @param measure 系统实际测量值
 * @return        控制量输出
 *
 * @note 控制流程：
 *       1. TD平滑目标值（如果启用）
 *       2. 二阶ESO估计系统状态和总扰动
 *       3. P控制器计算虚拟控制量u0
 *       4. 扰动补偿得到实际控制量
 *       5. 输出限幅和抗积分饱和处理
 */
float first_order_ladrc_calc(first_order_ladrc_t *fladrc,
                              float target, float measure) {
    /*
     * 一阶LADRC原理：
     * 被控对象：ẋ = f(x, w, t) + b*u  (一阶系统)
     * 其中f(x,w,t)为总扰动，包含模型不确定性和外部扰动
     */

    /* 步骤0: TD（跟踪微分器）处理目标值 */
    float td_target = target;
    if (fladrc->use_td) {
        /* 参数固化模式：不再传入dt，使用结构体中固化的td->h */
        td_target = td_update(&fladrc->td, target);
                                                    // 获取平滑后的目标值
                                                    // 同时fladrc->td.x2为目标变化率
    }

    /* 步骤1: 执行二阶扩张状态观测器(ESO) */
    /*
     * 一阶系统ESO公式 (二阶观测器):
     * dx1 = x2 + b*u + beta1 * (measure - x1)   <- ẋ1 = x2 + b*u + 修正项
     * dx2 = beta2 * (measure - x1)              <- ẋ2 = 扰动变化率
     *
     * 状态含义:
     * x1 - 系统输出估计（跟踪测量值measure）
     * x2 - 总扰动估计（包含内部动态f(x)和外部扰动w）
     *
     * 关键设计：使用上一时刻的实际输出(限幅后的pre_out)进行ESO更新，
     *          防止积分饱和导致观测器发散
     */

    /* 计算ESO微分方程 - 优化：只计算一次误差 */
    float error = measure - fladrc->x1;             // 观测误差 = 测量值 - 估计值
    float dx1 = fladrc->x2 + fladrc->b * fladrc->pre_out + fladrc->beta1 * error;
                                                    // 输出估计的微分
                                                    // = 扰动估计 + b*控制量 + 修正项
    float dx2 = fladrc->beta2 * error;              // 扰动估计的微分（假设扰动变化缓慢）

    /* 更新状态估计值(欧拉积分，乘以dt) */
    fladrc->x1 += dx1 * fladrc->dt;                 // 离散积分更新输出估计
    fladrc->x2 += dx2 * fladrc->dt;                 // 离散积分更新扰动估计

    /* 步骤2: 计算控制量 */
    /*
     * 一阶LADRC控制律:
     * u0 = kp * (target - x1)        // 名义控制：P控制器
     * u = (u0 - x2) / b              // 扰动补偿：用估计的扰动x2进行前馈补偿
     *
     * 物理意义：通过ESO估计出总扰动x2，在控制量中将其抵消，
     *          使系统变为纯粹的积分器 ẋ = b*u0
     */

    /* 计算名义控制量u0 */
    float u0 = fladrc->kp * (td_target - fladrc->x1);
                                                    // P控制器：u0 = kp * 误差

    /* 抗积分饱和处理 */
    /*
     * 当控制量达到限幅时，ESO中的扰动估计可能会持续累积（积分饱和），
     * 导致系统退出饱和时出现大的超调。
     * 抗积分饱和通过检测饱和误差，调整u0使其退出饱和状态。
     */
    if (fladrc->k_aw > 0.0f && fabsf(fladrc->pre_out) >= fladrc->max_output * 0.99f) {
        float u_ideal = (u0 - fladrc->x2) / fladrc->b;
                                                    // 理论上的理想控制量（无限幅时）
        float saturation_error = fladrc->pre_out - u_ideal;
                                                    // 饱和误差 = 实际输出 - 理想输出
        u0 -= fladrc->k_aw * fladrc->b * saturation_error;
                                                    /* 调整名义控制量u0，使其趋向于退出饱和
                                                     * k_aw越大，退出饱和越快，但可能影响稳态精度 */
    }

    /* 计算理论控制输出u */
    float out_temp;
    if (fabsf(fladrc->b) < 0.0001f) {               // 安全检查：防止除零
        out_temp = 0.0f;
    } else {
        out_temp = (u0 - fladrc->x2) / fladrc->b;   // 扰动补偿：u = (u0 - x2) / b
                                                    // 将估计的扰动x2从控制量中抵消
    }

    /* 输出限幅 */
    if (out_temp > fladrc->max_output) {
        out_temp = fladrc->max_output;
    } else if (out_temp < -fladrc->max_output) {
        out_temp = -fladrc->max_output;
    }

    /* 保存输出用于下一次ESO计算 */
    fladrc->out = out_temp;
    fladrc->pre_out = out_temp;                     // 保存限幅后的输出，用于下一轮ESO更新
                                                    // 【关键】这防止了ESO看到"想要的"控制量，
                                                    // 而是看到"实际的"控制量，避免积分饱和

    return fladrc->out;
}
```

### 2.6 一阶LADRC参数说明

| 参数 | 符号 | 作用 | 整定建议 |
|------|------|------|----------|
| **ESO增益** | $\beta_1, \beta_2$ | 观测器增益 | 基于带宽法：$\beta_1=2\omega_o, \beta_2=\omega_o^2$ |
| **比例增益** | $k_p$ | 误差响应 | 基于带宽法：$k_p = \omega_c$ |
| **控制增益** | $b$ | 控制效率 | 从阶跃响应估计，需与实际系统匹配 |
| **输出限幅** | $max\_output$ | 执行器输出限制 | 根据执行器能力设置 |
| **抗饱和增益** | $k\_{aw}$ | 抑制积分饱和 | 0表示不使用，建议值1.0-3.0 |
| **TD快速跟踪因子** | $td\_r$ | 目标值响应速度 | 0表示禁用TD，建议值根据过渡时间要求设置 |
| **TD滤波因子** | $td\_n$ | 噪声过滤能力 | 无量纲，建议值1~5 |

---

## 第三部分：二阶LADRC实现

### 3.1 二阶LADRC原理

二阶LADRC适用于被控对象为二阶的系统（如位置控制、角度控制等）。其核心组成：

1. **三阶ESO（扩张状态观测器）**：估计位置、速度和总扰动
2. **PD控制器**：根据误差生成虚拟控制量
3. **扰动补偿**：利用估计的总扰动进行补偿

**被控对象模型**：

$$
\ddot{y} = f(x, \dot{x}, d, t) + b \cdot u
$$

其中 $f$ 为总扰动（包含模型不确定性和外部扰动）。

### 3.2 二阶LADRC结构体定义

``c
/**
 * @brief 二阶LADRC结构体
 * @note  适用于位置/角度控制等二阶系统
 *        采用组合模式，内嵌TD模块
 */
typedef struct {
    // ESO增益（三阶观测器）
    float beta1;        // ESO增益beta1 - 位置观测带宽
    float beta2;        // ESO增益beta2 - 速度观测带宽
    float beta3;        // ESO增益beta3 - 扰动观测带宽

    // 控制器增益
    float kp;           // 比例增益（刚度）
    float kd;           // 微分增益（阻尼）- 注意：LADRC中通常设为0，由b0处理，除非需要额外的PD

    // 系统参数
    float b;            // 控制增益(b0) - 决定控制量的缩放比例
    float dt;           // 采样周期(秒) - 【固化参数】RTOS固定周期

    // 状态估计值
    float x1;           // 估计的位置（跟踪测量值）
    float x2;           // 估计的速度
    float x3;           // 估计的总扰动（包含内部动态和外部扰动）

    // 输出限制
    float max_output;   // 输出限幅值（如PWM最大值）

    // 抗积分饱和
    float k_aw;         // 抗积分饱和增益（0表示不使用，建议值1.0~3.0）
    float pre_out;      // 上一时刻输出（用于计算饱和程度）

    // 输出
    float out;          // 当前输出

    // TD组合模式（v3.1新增）
    td_t td;            // 内嵌TD结构体
    bool use_td;        // 是否使用TD（在初始化时根据td_r参数决定）
} ladrc_t;
```

### 3.3 二阶LADRC初始化函数

``c
/**
 * @brief 二阶LADRC初始化 - 参数固化+TD组合模式
 *
 * @param ladrc      二阶LADRC结构体指针
 * @param max_output 控制量输出限幅（例如PWM最大值）
 * @param beta1      ESO状态观测器增益1（位置观测带宽）
 * @param beta2      ESO状态观测器增益2（速度观测带宽）
 * @param beta3      ESO状态观测器增益3（扰动观测带宽）
 * @param kp         控制器比例增益（刚度）
 * @param kd         控制器微分增益（阻尼）- 注意：LADRC中通常设为0，由b0处理
 * @param b          系统增益估计值(b0) - 决定控制量的缩放比例
 * @param dt         RTOS固定采样周期(秒) - 必须与实际任务频率一致，将固化到结构体
 * @param k_aw       抗积分饱和增益（建议值1.0~3.0，0表示不启用）
 * @param td_r       TD快速跟踪因子(r) - 决定目标值响应速度
 *                   0表示禁用TD（直接透传目标值）
 * @param td_n       TD滤波因子(无量纲，建议值1~5) - 决定噪声过滤能力
 *                   N=1: 滤波最弱响应最快；N=3~5: 典型推荐值；N>10: 强滤波但滞后明显
 * @param td_max_x2  TD最大速度限制(0表示不限制) - 防止设定值跳变过大导致系统冲击
 */
void ladrc_init(ladrc_t *ladrc, float max_output,
                float beta1, float beta2, float beta3,
                float kp, float kd, float b,
                float dt, float k_aw,
                float td_r, float td_n, float td_max_x2) {
    /* 1. 初始化 ESO (扩张状态观测器) 参数 */
    ladrc->beta1 = beta1;
    ladrc->beta2 = beta2;
    ladrc->beta3 = beta3;

    /* 2. 初始化 控制器 (State Error Feedback) 参数 */
    ladrc->kp = kp;
    ladrc->kd = kd;                                 // 注意：LADRC中通常设为0，由b0处理
    ladrc->b = b;

    /* 3. 清空 ESO 内部状态 */
    ladrc->x1 = 0.0f;                               // 估计位置清零
    ladrc->x2 = 0.0f;                               // 估计速度清零
    ladrc->x3 = 0.0f;                               // 估计总扰动清零

    /* 4. 输出与抗饱和设置 */
    ladrc->max_output = max_output;
    ladrc->k_aw = k_aw;
    ladrc->pre_out = 0.0f;                          // 上一时刻输出清零
    ladrc->out = 0.0f;

    /* 5. 固化采样周期 (核心设计) */
    // 这个 dt 将被 ESO 和 TD 共同使用，作为时间基准
    // 固化后，update函数不再接受外部dt，消除时间抖动影响
    ladrc->dt = dt;

    /* 6. 初始化 TD (参数固化模式) */
    if (td_r > 0.0f) {
        // 调用 td_init - 使用参数固化模式
        // 传入滤波因子 N (td_n)，内部自动计算 h0 = N * dt
        td_init(&ladrc->td, td_r, dt, td_n, td_max_x2);
        ladrc->use_td = true;                       // 启用TD
    } else {
        // 禁用 TD 模式，目标值直接透传
        ladrc->use_td = false;
        // 为了安全，将 TD 状态清零
        td_reset(&ladrc->td, 0.0f);
    }
}
```

### 3.4 二阶LADRC参数重置函数

``c
/**
 * @brief 二阶LADRC参数重置 - 支持热更新
 *
 * @note 在系统运行过程中动态调整参数，同时重置ESO状态避免瞬态问题
 *       常用于自适应控制、参数调度等场景
 */
void ladrc_reset(ladrc_t *ladrc, float beta1, float beta2, float beta3,
                 float kp, float kd, float b,
                 float dt, float k_aw,
                 float td_r, float td_n, float td_max_x2) {
    /* 更新控制参数 */
    ladrc->beta1 = beta1;
    ladrc->beta2 = beta2;
    ladrc->beta3 = beta3;
    ladrc->kp = kp;
    ladrc->kd = kd;
    ladrc->b = b;
    ladrc->dt = dt;
    ladrc->k_aw = k_aw;

    /* 重置 ESO 状态，避免参数切换时的瞬态问题 */
    ladrc->x1 = 0.0f;
    ladrc->x2 = 0.0f;
    ladrc->x3 = 0.0f;
    ladrc->pre_out = 0.0f;

    /* 重新配置TD参数 - 使用参数固化模式 */
    if (td_r > 0.0f) {
        /* 安全性检查：滤波因子N不能小于1.0 */
        float n = td_n;
        if (n < 1.0f) {
            n = 1.0f;
        }
        ladrc->td.r = td_r;
        ladrc->td.h = dt;           // 固化采样周期
        ladrc->td.h0 = n * dt;      // 计算内部滤波参数 h0 = N * dt
        ladrc->td.max_x2 = td_max_x2;
        ladrc->use_td = true;
    } else {
        ladrc->use_td = false;
    }
}
```

### 3.5 二阶LADRC计算函数

``c
/**
 * @brief 二阶LADRC计算 - 核心控制算法
 *
 * @param ladrc    二阶LADRC结构体指针
 * @param target   目标值
 * @param measure  系统实际测量值（如编码器读数）
 * @return         控制量输出
 *
 * @note 控制流程：
 *       1. TD平滑目标值（如果启用）
 *       2. 三阶ESO估计系统状态（位置、速度）和总扰动
 *       3. PD控制器计算虚拟控制量u0
 *       4. 扰动补偿得到实际控制量
 *       5. 输出限幅和抗积分饱和处理
 */
float ladrc_calc(ladrc_t *ladrc, float target, float measure) {
    /* 步骤0: TD（跟踪微分器）处理目标值 */
    float td_target = target;
    if (ladrc->use_td) {
        /* 参数固化模式：不再传入dt，使用结构体中固化的td->h */
        td_target = td_update(&ladrc->td, target);
                                                    // 获取平滑后的目标值
                                                    // ladrc->td.x1: 平滑目标值
                                                    // ladrc->td.x2: 目标变化率（微分信号）
    }

    /* 步骤1: 执行三阶扩张状态观测器(ESO) */
    /*
     * 二阶LADRC被控对象: ẍ = f + b*u
     * 三阶ESO公式:
     * dx1 = x2 + β1*(y - x1)         <- ẋ1 = x2 (速度)
     * dx2 = x3 + b*u + β2*(y - x1)   <- ẋ2 = x3 + b*u (加速度=扰动+控制)
     * dx3 = β3*(y - x1)              <- ẋ3 = df/dt (扰动变化率，假设变化缓慢)
     *
     * 状态含义:
     * x1 = y (位置估计)
     * x2 = ẏ = v (速度估计)
     * x3 = f(x,ẋ,d) (总扰动估计，包含模型不确定性和外部扰动)
     *
     * 关键设计：使用上一时刻的实际输出(限幅后的pre_out)进行ESO更新，
     *          防止积分饱和导致观测器发散
     */

    /* 计算ESO微分方程 - 优化：只计算一次误差 */
    float error = measure - ladrc->x1;              // 观测误差 = 测量值 - 估计值
    float dx1 = ladrc->x2 + ladrc->beta1 * error;   // 位置估计的微分 = 速度估计 + 修正项
    float dx2 = ladrc->x3 + ladrc->b * ladrc->pre_out + ladrc->beta2 * error;
                                                    // 速度估计的微分 = 扰动估计 + b*控制量 + 修正项
    float dx3 = ladrc->beta3 * error;               // 扰动估计的微分（假设扰动变化缓慢）

    /* 更新状态估计值(欧拉积分，乘以dt) */
    ladrc->x1 += dx1 * ladrc->dt;                   // 离散积分更新位置估计
    ladrc->x2 += dx2 * ladrc->dt;                   // 离散积分更新速度估计
    ladrc->x3 += dx3 * ladrc->dt;                   // 离散积分更新扰动估计

    /* 步骤2: 计算控制量u0 */
    /*
     * 控制律公式:
     * u0 = kp * (r - x1) - kd * x2   // 名义控制：PD控制器
     * u = (u0 - x3) / b              // 扰动补偿：用估计的扰动x3进行前馈补偿
     *
     * 物理意义：通过ESO估计出总扰动x3，在控制量中将其抵消，
     *          使系统变为纯粹的二重积分器 ẍ = b*u0
     */

    /* 计算名义控制量u0 */
    float u0 = ladrc->kp * (td_target - ladrc->x1) - ladrc->kd * ladrc->x2;
                                                    // PD控制器：u0 = kp*误差 - kd*速度
                                                    // 注意：LADRC中kd通常设为0，由b0处理阻尼

    /* 抗积分饱和处理 */
    /*
     * 当控制量达到限幅时，通过调整名义控制量u0使其退出饱和状态
     */
    if (ladrc->k_aw > 0.0f && fabsf(ladrc->pre_out) >= ladrc->max_output * 0.99f) {
        float u_ideal = (u0 - ladrc->x3) / ladrc->b;
                                                    // 理论上的理想控制量（无限幅时）
        float saturation_error = ladrc->pre_out - u_ideal;
                                                    // 饱和误差 = 实际输出 - 理想输出
        u0 -= ladrc->k_aw * ladrc->b * saturation_error;
                                                    // 调整u0使其趋向于退出饱和
    }

    /* 计算理论控制输出u */
    float out_temp;
    if (fabsf(ladrc->b) < 0.0001f) {                // 安全检查：防止除零
        out_temp = 0.0f;
    } else {
        out_temp = (u0 - ladrc->x3) / ladrc->b;     // 扰动补偿：u = (u0 - x3) / b
                                                    // 将估计的扰动x3从控制量中抵消
    }

    /* 输出限幅 */
    if (out_temp > ladrc->max_output) {
        out_temp = ladrc->max_output;
    } else if (out_temp < -ladrc->max_output) {
        out_temp = -ladrc->max_output;
    }

    /* 保存输出用于下一次ESO计算 */
    ladrc->out = out_temp;
    ladrc->pre_out = out_temp;                      // 保存限幅后的输出，用于下一轮ESO更新
                                                    // 【关键】ESO看到"实际的"控制量，避免积分饱和

    return ladrc->out;
}
```

### 3.6 二阶LADRC参数说明

| 参数 | 符号 | 作用 | 整定建议 |
|------|------|------|----------|
| **ESO增益** | $\beta_1, \beta_2, \beta_3$ | 决定观测器跟踪速度和稳定性 | 基于带宽法：$\beta_1=3\omega_o, \beta_2=3\omega_o^2, \beta_3=\omega_o^3$ |
| **比例增益** | $k_p$ | 位置误差响应 | 基于带宽法：$k_p = \omega_c^2$ |
| **微分增益** | $k_d$ | 速度阻尼 | 基于带宽法：$k_d = 2\omega_c$。注意：LADRC中通常设为0，由b0处理 |
| **控制增益** | $b$ | 控制效率 | 从阶跃响应估计，需与实际系统匹配 |
| **输出限幅** | $max\_output$ | 执行器输出限制 | 根据执行器能力设置 |
| **抗饱和增益** | $k\_{aw}$ | 抑制积分饱和 | 0表示不使用，建议值1.0-3.0 |
| **TD快速跟踪因子** | $td\_r$ | 目标值响应速度 | 0表示禁用TD，建议值根据过渡时间要求设置 |
| **TD滤波因子** | $td\_n$ | 噪声过滤能力 | 无量纲，建议值1~5 |

---

## 第四部分：完整使用示例

``c
#include <stdio.h>
#include <math.h>
#include "ladrc.h"

// 模拟被控对象（二阶系统）
float Plant_Update(float u, float h) {
    static float x1 = 0, x2 = 0;    // 位置和速度状态
    static float d = 0;              // 外部扰动

    // 系统动力学：ddot(y) = -a1*dot(y) - a0*y + b*u + d
    float a1 = 2.0f, a0 = 1.0f, b = 10.0f;
    float dx2 = -a1 * x2 - a0 * x1 + b * u + d;

    // 离散积分（欧拉法）
    x2 += h * dx2;      // 更新速度
    x1 += h * x2;       // 更新位置

    return x1;          // 返回输出（位置）
}

int main(void)
{
    // 定义LADRC控制器（v3.1版本，内嵌TD，无需单独声明td_t）
    ladrc_t controller;

    // 系统参数
    float dt = 0.001f;              // 1ms采样周期
    float wo = 100.0f;              // 观测器带宽
    float wc = 25.0f;               // 控制器带宽
    float b = 10.0f;                // 控制增益

    // 计算LADRC参数（带宽法）
    float beta1 = 3.0f * wo;                // 3*wo
    float beta2 = 3.0f * wo * wo;           // 3*wo^2
    float beta3 = wo * wo * wo;             // wo^3
    float kp = wc * wc;                     // wc^2
    float kd = 2.0f * wc;                   // 2*wc

    // 初始化LADRC：max_output=50, k_aw=2.0（启用抗积分饱和）
    // td_r=100启用TD，td_n=5设置滤波因子，td_max_x2=0不限制速度
    ladrc_init(&controller, 50.0f, beta1, beta2, beta3,
               kp, kd, b, dt, 2.0f,          // 基本参数
               100.0f, 5.0f, 0.0f);          // TD参数：r=100, N=5, max_x2=0

    printf("LADRC控制仿真开始...\n");
    printf("时间\t目标\t输出\t控制量\t扰动估计\n");

    // 仿真循环
    float y = 0.0f;                  // 系统输出
    for (int k = 0; k < 2000; k++) {
        float t = k * dt;            // 当前时间

        // 目标值：0-0.5s为0，0.5s后跳变到10
        float ref = (t > 0.5f) ? 10.0f : 0.0f;

        // LADRC控制计算（TD处理已在内部完成）
        float u = ladrc_calc(&controller, ref, y);

        // 更新被控对象（实际系统中，这是物理过程）
        y = Plant_Update(u, dt);

        // 每100ms打印一次
        if (k % 100 == 0) {
            printf("%.3f\t%.2f\t%.2f\t%.2f\t%.2f\n",
                   t, ref, y, u, controller.x3);
        }
    }

    printf("仿真结束\n");
    return 0;
}
```

### 使用要点总结

1. **无需单独声明TD结构体**：TD内嵌在LADRC结构体中，一体化调用
2. **参数固化模式**：`dt`在初始化时固化，调用`ladrc_calc`时无需再传`dt`
3. **TD通过参数启用**：`td_r > 0`启用TD，`td_r = 0`禁用TD（直接透传目标值）
4. **滤波因子改为无量纲N**：`td_n`是1~5的无量纲参数，内部自动计算`h0 = N * dt`

---

## 第五部分：调试技巧与常见问题

### 5.1 调试步骤

LADRC参数整定的一般流程：

```
步骤1: 确定系统阶数
    └─→ 观察被控对象：输入到输出需要几次积分
    └─→ 一阶系统（电流、简单速度）→ 使用first_order_ladrc
    └─→ 二阶系统（位置、角度）→ 使用ladrc

步骤2: 估计b
    ├─→ 施加阶跃输入，记录输出响应
    ├─→ 测量初始加速度：a0 = (y[2] - 2*y[1] + y[0]) / h^2
    └─→ 计算：b = a0 / U（U为阶跃幅值）

步骤3: 设置带宽参数
    ├─→ wc = 期望控制器带宽（如：希望0.1s响应，wc≈10*2π/0.1≈60）
    ├─→ wo = 3~5 * wc（初始比例，观测器带宽）
    └─→ 计算LADRC增益：
        - 一阶：β1=2*wo, β2=wo^2, kp=wc
        - 二阶：β1=3*wo, β2=3*wo^2, β3=wo^3, kp=wc^2, kd=2*wc

步骤4: 设置TD参数（可选）
    ├─→ td_r = 100（TD快速跟踪因子，初始值）
    └─→ td_n = 5（滤波因子，无量纲）

步骤5: 逐步调试wc
    ├─→ 先设较小的wc（如目标值的50%）
    ├─→ 测试响应，如过慢则增大20%
    └─→ 直到响应速度满足要求

步骤6: 调试wo
    ├─→ 固定wc，调整wo/wc比例
    ├─→ 如果估计滞后明显：增大wo
    └─→ 如果噪声敏感：减小wo

步骤7: 微调b
    ├─→ 如果系统振荡：减小b（如减10%）
    └─→ 如果响应迟缓：增大b（如增10%）

步骤8: 优化TD参数
    ├─→ 如果目标跳变时超调大：减小td_r或增大td_n
    └─→ 如果过渡过程过慢：增大td_r
```

### 5.2 常见问题排查

| 现象        | 可能原因          | 解决方案                |
| --------- | ------------- | ------------------- |
| **系统振荡**  | wc过大、b过大、wo过大 | 逐步减小参数，每次减10-20%    |
| **响应迟缓**  | wc过小、b过小      | 增大wc或b              |
| **估计滞后**  | wo过小          | 增大wo，但不要超过5wc       |
| **噪声敏感**  | wo过大、td_n过小   | 减小wo，或增大td_n        |
| **超调严重**  | wc过大、td_r过大   | 减小wc或td_r           |
| **稳态误差**  | b估计不准         | 微调b，或检查观测器是否正常收敛    |
| **启动冲击**  | TD参数不当        | 减小td_r或增大td_n       |
| **饱和后超调** | 未启用抗积分饱和      | 增大k_aw（建议1.0-3.0）   |
| **目标值抖动** | td_n过小或TD禁用   | 增大td_n（建议3~5），或启用TD |
注：现象与可能原因均摘自网络

### 5.3 调试技巧

**1. 实时监控关键变量**

在调试阶段，建议实时输出以下变量：
- 目标值 `target`
- 实际输出 `measure`
- 估计的扰动 `x3`（二阶）或 `x2`（一阶）
- 控制量 `out`
- 虚拟控制量 `u0`
- TD状态（如果启用）

```c
// 调试输出示例（通过串口或日志）
void LADRC_DebugOutput(ladrc_t *ladrc, float target, float measure)
{
    printf("目标=%.2f, 实际=%.2f, 估计位置=%.2f, 估计速度=%.2f, 扰动=%.2f, 输出=%.2f\n",
           target, measure, ladrc->x1, ladrc->x2, ladrc->x3, ladrc->out);
    if (ladrc->use_td) {
        printf("TD: 平滑目标=%.2f, 目标速度=%.2f\n", ladrc->td.x1, ladrc->td.x2);
    }
}
```

**2. 参数调整口诀**

```
响应慢 → 增wc
有振荡 → 减wc或减b
估计慢 → 增wo
噪声大 → 减wo或增td_n
有超调 → 减td_r（TD速度）
目标抖 → 增td_n（TD滤波）
```

**3. 安全限幅**

```c
// ladrc_calc内部已集成输出限幅，只需设置max_output参数
// 如需动态调整限幅值，可直接修改结构体

void LADRC_SetOutputLimit(ladrc_t *ladrc, float max_output)
{
    ladrc->max_output = max_output;
}

// 抗积分饱和通过k_aw参数启用
void LADRC_SetAntiWindup(ladrc_t *ladrc, float k_aw)
{
    ladrc->k_aw = k_aw;  // 建议值1.0-3.0，0表示禁用
}
```

**4. 在线参数调整**

在调试阶段，可以设计通信接口实现参数在线调整：

```c
// 通过串口接收命令调整参数
void LADRC_AdjustParameter(ladrc_t *ladrc, char param, float value)
{
    switch(param) {
        case 'w':  // wo - 重新计算beta增益
            ladrc->beta1 = 3.0f * value;
            ladrc->beta2 = 3.0f * value * value;
            ladrc->beta3 = value * value * value;
            break;
        case 'c':  // wc - 重新计算PD增益
            ladrc->kp = value * value;
            ladrc->kd = 2.0f * value;
            break;
        case 'b':  // b - 控制增益
            ladrc->b = value;
            break;
        case 'r':  // td_r - TD快速跟踪因子
            if (value > 0.0f && !ladrc->use_td) {
                ladrc->use_td = true;
            }
            ladrc->td.r = value;
            break;
        case 'n':  // td_n - TD滤波因子
            if (value < 1.0f) value = 1.0f;
            ladrc->td.h0 = value * ladrc->dt;
            break;
    }
}
```

---

> **总结**：
> - **TD模块**：内嵌在LADRC结构体中，关键参数td_r（快速跟踪因子）和td_n（无量纲滤波因子）
> - **一阶LADRC**：适用于电流/速度控制，包含二阶ESO和P控制器，关键参数β1/β2（观测器增益）、kp（控制器增益）、b（控制增益）
> - **二阶LADRC**：适用于位置/角度控制，包含三阶ESO和PD控制器，关键参数β1/β2/β3（观测器增益）、kp/kd（控制器增益）、b（控制增益）
> - **参数固化模式**：dt在初始化时固化到结构体，消除时间抖动影响
> - **抗积分饱和**：通过k_aw参数启用，调整u0使其退出饱和状态
>
> 建议初学者按照文档顺序先理解每个模块的作用，再逐步调试参数，最终你会体会到LADRC"以简御繁"的设计哲学。
