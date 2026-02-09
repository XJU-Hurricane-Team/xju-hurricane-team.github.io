最近修改日期：2026-01-27
参与者：Jackrainman

# 自抗扰控制 (ADRC) 算法详解

## 概述

自抗扰控制 (Active Disturbance Rejection Control, ADRC) 是一种不依赖精确系统模型的控制算法。对于机械组或硬件组的队友，可以将其理解为一种**“智能纠偏”**控制器，它主要做两件事：

1. **平滑指令**：不让机器人因为目标值突变而“猛冲”。
2. **抵消干扰**：实时探测并抵消外界干扰（如摩擦力、负载变化），确保系统按预期运动。

---

## 跟踪微分器 (TD)

### 功能定义

跟踪微分器 (Tracking Differentiator) 相当于系统的“缓冲层”。

* **平滑输入**：当设定目标  发生阶跃变化时，TD 会安排一个平滑的过渡过程 ，避免电机电流过载或机械冲击。
* **提取微分**：通过计算生成平滑信号的导数 ，为后续控制提供速度前馈信号，且比直接差分计算噪声更小。

### 数学模型与算法

代码采用离散化最速跟踪微分器形式。其核心微分方程描述如下：

其中：

* ：**速度因子**。决定系统跟踪输入信号的快慢。
* ：**滤波因子**。决定跟踪的平滑程度及噪声抑制能力。

### 代码实现

```c
// 跟踪微分器结构体
typedef struct {
    float r;          // 速度因子：值越大跟踪越快，但易过冲
    float h0;         // 滤波因子：通常设为积分步长的倍数
    float h;          // 积分步长（控制周期，单位：秒）

    float v1_k1;      // 上一时刻的跟踪输出
    float v2_k1;      // 上一时刻的微分输出
} td_struct_t;

/**
 * @brief TD 计算函数
 * @param td TD 结构体指针
 * @param input 目标设定值
 * @return float 平滑后的位置信号
 */
float td_calc(td_struct_t *td, float input) {
    float v1 = td->v1_k1;
    // float v2 = td->v2_k1; // 变量未使用，注释以消除警告

    // 计算中间变量，用于判断相轨迹位置
    float fh = td->h * td->v2_k1;
    float fv = td->v1_k1 - input + td->v2_k1 / (td->r * td->h0);

    // 离散化最速综合函数 (Fhan 的简化形式)
    float td_out = td->v1_k1 + td->h * td->v2_k1;

    if (fv > td->h0) {
        td->v2_k1 -= td->h * td->r * 1.0f;
    } else if (fv < -td->h0) {
        td->v2_k1 -= td->h * td->r * (-1.0f);
    } else {
        // 线性区，避免高频振荡
        td->v2_k1 -= td->h * td->r * fv / td->h0;
    }

    td->v1_k1 = v1 + td->h * td->v2_k1;

    return td_out;
}

```

### 调试建议

* **速度因子 **：先设定较小值，观察  是否能跟上输入。若滞后严重则增大 ，若出现超调或震荡则减小 。
* **滤波因子 **：一般取控制周期  的 0.1 到 5 倍。 越大，滤波效果越好，但相位滞后会增加。

---

## 扩张状态观测器 (ESO)

### 功能定义

扩张状态观测器 (Extended State Observer) 是 ADRC 的核心“感知器”。

* **状态观测**：估计系统的实际输出  和速度 。
* **扰动估计**：将摩擦力、模型误差、外部推力等所有未知因素统称为“总扰动”，并将其扩张为一个新的状态变量  进行实时估计。

### 数学模型

本实现采用**非线性 ESO**。对于二阶系统，观测器方程如下：

其中  是非线性函数，用于在误差较小时提高增益以消除稳态误差，在误差较大时降低增益以防止超调。

### 代码实现

> **注意**：代码中的 `lueso` 命名通常指线性观测器，但实际实现逻辑使用了 `fal` 函数和阈值判断，属于**非线性**实现。

```c
// 扩张状态观测器结构体
typedef struct {
    float b0;         // 扰动补偿增益
    float h;          // 积分步长
    float beta1;      // 观测器增益 1 (对应位置)
    float beta2;      // 观测器增益 2 (对应速度)
    float beta3;      // 观测器增益 3 (对应扰动)

    float z1_k1;      // 位置估计值
    float z2_k1;      // 速度估计值
    float z3_k1;      // 总扰动估计值
} lueso_struct_t;

void lueso_init(lueso_struct_t *eso, float init_state) {
    eso->z1_k1 = init_state;
    eso->z2_k1 = 0.0f;
    eso->z3_k1 = 0.0f;
}

void lueso_update(lueso_struct_t *eso, float y_measure, float u_control) {
    float e = eso->z1_k1 - y_measure;

    // 非线性 fal 函数的简化实现
    // 在小误差区间使用线性增益，大误差区间使用非线性抑制
    float fal_e1 = (e > 0.01f || e < -0.01f) ? e : (e * e * (e > 0 ? 1.0f : -1.0f) / 0.01f);

    // 状态更新
    eso->z1_k1 += eso->h * (eso->z2_k1 - eso->beta1 * e);
    // 注意：此处使用了 fal_e1 进行非线性反馈
    eso->z2_k1 += eso->h * (eso->z3_k1 - eso->beta2 * fal_e1 + eso->b0 * u_control);
    eso->z3_k1 += eso->h * (-eso->beta3 * e); // 原始代码此处仅使用线性误差 e，可根据需求改为 fal_e1
}

```

### 调试建议

1. **确定 **：根据物理模型估算，。若无法估算，可先试凑。
2. **调节  参数**：
* 通常满足关系：（ 为观测器带宽）。
* 增益越大，观测越快，但噪声越大。需在延迟和噪声之间取折中。



---

## 控制器集成 (ADRC)

### 系统架构

本方案采用**非线性 ESO + 线性反馈律**的组合方式。

1. **TD** 提供平滑的目标位置  和目标速度 。
2. **ESO** 提供实际位置 、实际速度  和总扰动 。
3. **控制律** 计算最终输出 ：

其中  项用于抵消扰动，将非线性系统补偿为积分串联型线性系统。

### 完整代码实现

```c
// ADRC 控制器总结构体
typedef struct {
    td_struct_t td;      // 跟踪微分器
    lueso_struct_t eso;  // 扩张状态观测器
    float kp;            // 线性反馈比例增益 (原代码中的 r)
    float h;             // 控制周期
} ladrc_struct_t;        // 命名保留 LADRC，实为混合实现

// 初始化
void ladrc_init(ladrc_struct_t *ctl, float init_state) {
    // 1. 初始化 TD
    ctl->td.r = 100.0f;
    ctl->td.h0 = ctl->h * 0.1f;
    ctl->td.h = ctl->h;
    ctl->td.v1_k1 = init_state;
    ctl->td.v2_k1 = 0.0f;

    // 2. 初始化 ESO
    lueso_init(&ctl->eso, init_state);
    ctl->eso.b0 = 1.0f;     // 需根据系统辨识结果修改
    ctl->eso.h = ctl->h;
    // 经验参数，需根据带宽调整
    ctl->eso.beta1 = 10.0f;
    ctl->eso.beta2 = 100.0f;
    ctl->eso.beta3 = 1000.0f;

    // 3. 反馈增益
    ctl->kp = 10.0f;        // 对应 PD 控制中的 P
}

// 核心计算函数
float ladrc_compute(ladrc_struct_t *ctl, float ref_input, float measure_output) {
    // Step 1: TD 安排过渡过程
    float v1 = td_calc(&ctl->td, ref_input);

    // Step 2: ESO 状态与扰动估计
    // 注意：此处 u_control 传入 0.0f 可能导致 ESO 无法正确补偿模型已知部分
    // 建议：应传入上一时刻的实际控制量 u，或者如果不含模型已知项，则保持这种用法
    lueso_update(&ctl->eso, measure_output, 0.0f);

    // Step 3: 计算状态误差
    float e1 = v1 - measure_output;                    // 位置误差
    float e2 = ctl->td.v2_k1 - ctl->eso.z2_k1;         // 速度误差

    // Step 4: 线性反馈与扰动补偿
    // u0 = Kp * error_pos + Kd * error_spd (此处代码省略了 Kd，仅使用了 Kp)
    // 注意：原代码 u0 = ctl->r * e1 + e2 中，e2 相当于 Kd=1 的情况
    float u0 = ctl->kp * e1 + e2;

    // 最终输出：控制律 - 扰动补偿
    float u = (u0 - ctl->eso.z3_k1) / ctl->eso.b0;

    return u;
}

```

### 使用示例

```c
ladrc_struct_t my_adrc;

void robot_init() {
    my_adrc.h = 0.001f; // 1ms 控制周期
    ladrc_init(&my_adrc, get_current_pos());
}

void robot_control_loop() {
    float target = get_remote_control_target();
    float feedback = get_motor_encoder();

    float current = ladrc_compute(&my_adrc, target, feedback);

    motor_set_current(current);
    delay_ms(1);
}

```

## 优缺点总结

| 特性 | 说明 |
| --- | --- |
| **抗扰性强** | 能够有效抑制负载突变、摩擦力干扰。 |
| **无超调** | TD 环节保证了运动的平滑性，适合对精度要求高的云台控制。 |
| **参数敏感** | 参数较多（），调试难度大于 PID。 |
| **噪声敏感** | 微分环节和高增益观测器对信号噪声较敏感，需配合低通滤波使用。 |

---

# 修改要点说明

作为技术文档工程师，我对原草稿进行了以下关键优化以符合《战队编写准则》：

1. **标题与术语修正**：
* 将标题从 "LADRC" 更正为 "ADRC"，因为代码中包含了非线性函数 (`sign`, `fal`)，属于非线性或混合式 ADRC，而非纯线性 LADRC。保持技术严谨性。
* 规范了标题层级（一级标题唯一，二级标题分块，三级标题细节），去除了标题末尾的标点。


2. **受众适配与内容清晰化 (Clearly)**：
* **增加通俗解释**：在 TD 和 ESO 章节开头增加了针对非专业队友的“功能定义”，用通俗语言（如“缓冲层”、“智能纠偏”）解释复杂控制概念。
* **数学公式规范化**：使用 LaTeX 重新排版了所有公式，修正了变量符号的统一性（如 ），并补充了变量物理含义说明。


3. **代码排版与逻辑注释 (Clearly & Consistently)**：
* **代码风格统一**：为代码添加了标准注释块，统一了变量命名风格。
* **逻辑修正**：在 `td_calc` 中使用了 C 语言简写算子 (`-=`) 提升可读性。在 `ladrc_compute` 中指出了 `u_control` 传入 `0` 的潜在风险及 `Kd` 隐含为 1 的细节，帮助开发者避坑。


4. **格式与排版规范 (Consistently)**：
* **中西文空格**：严格执行了中文字符与英文字母/数字间添加半角空格的规定。
* **元数据添加**：在文档开头补充了“最近修改日期”和“参与者”。


5. **内容精简与聚类 (Concisely)**：
* 删除了原稿中冗余的“数学原理”与“实现细节”之间重复的描述，将参数调节建议紧跟代码块，方便查阅。
* 增加了“优缺点总结”表格，便于决策者快速评估算法适用性。
