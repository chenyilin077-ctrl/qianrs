# TIM — 定时器 & PWM

> 芯片：STM32F4 | APB1（TIM2-7,12-14）/ APB2（TIM1,8-11）

## 原理

### 三类定时器
| 类型 | 型号 | 通道 | 功能 |
|------|------|------|------|
| 基本 | TIM6/7 | 无输出 | 纯定时中断 |
| 通用 | TIM2-5,9-14 | 4 通道 | PWM 输出/输入、编码器 |
| 高级 | TIM1/8 | 6 通道 | 死区互补、刹车 |

### 工作流程
```
时钟源 → 预分频 PSC → 计数器 CNT（0→ARR 循环）→ 比较 CCR → PWM 输出
```

### 公式
```
计数器频率 = 定时器时钟 / (PSC + 1)
PWM 频率   = 计数器频率 / (ARR + 1)
占空比     = CCR / (ARR + 1) × 100%
```

> ⚡ APB1 定时器时钟 = APB1×2 = 84MHz（APB1 分频 ≠ 1 时）

---

## CubeMX 配置

### 定时中断（TIM2, 1ms）

| 步骤 | 参数 | 值 |
|------|------|-----|
| **Timers** → **TIM2** | Clock Source | **Internal Clock** |
| **Configuration → TIM2** | Prescaler | **8400-1**（84MHz/8400=10kHz） |
| | Counter Period | **10-1**（10kHz/10=1kHz=1ms） |
| **NVIC Settings** | 勾选 **TIM2 global interrupt** | — |

### PWM 输出（TIM3_CH1 = PA6, 呼吸灯）

| 步骤 | 参数 | 值 |
|------|------|-----|
| **Timers** → **TIM3** | Clock Source | **Internal Clock** |
| | Channel 1 | **PWM Generation CH1** |
| PA6 自动设为 TIM3_CH1 | — | — |
| **Configuration → TIM3** | Prescaler | **84-1**（84MHz/84=1MHz） |
| | Counter Period (ARR) | **999**（1MHz/1000=1kHz） |
| **Configuration → TIM3 → CH1** | Pulse (CCR) | **0**（初始占空比 0%） |
| | Mode | **PWM mode 1** |

### 舵机 PWM（TIM3_CH3 = PB0, 50Hz）

| 步骤 | 参数 | 值 |
|------|------|-----|
| Prescaler | **84-1**（1MHz） |
| Counter Period | **20000-1**（1MHz/20000=50Hz） |
| Pulse | **1500**（1.5ms=舵机中位 90°） |

---

## CubeMX 自动生成代码

```c
// tim.c — MX_TIM3_Init()
static void MX_TIM3_Init(void) {
    TIM_ClockConfigTypeDef sClockSourceConfig = {0};
    TIM_MasterConfigTypeDef sMasterConfig = {0};
    TIM_OC_InitTypeDef sConfigOC = {0};

    htim3.Instance = TIM3;
    htim3.Init.Prescaler = 84 - 1;
    htim3.Init.CounterMode = TIM_COUNTERMODE_UP;
    htim3.Init.Period = 999;
    HAL_TIM_PWM_Init(&htim3);

    sConfigOC.OCMode = TIM_OCMODE_PWM1;
    sConfigOC.Pulse = 0;  // 初始占空比
    sConfigOC.OCPolarity = TIM_OCPOLARITY_HIGH;
    HAL_TIM_PWM_ConfigChannel(&htim3, &sConfigOC, TIM_CHANNEL_1);
}
// CubeMX 还会自动生成 MX_TIM2_Init()（定时中断）
```

---

## 手写业务代码

### 呼吸灯（TIM3_CH1）
```c
int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_TIM3_Init();

    HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_1);  // 启动 PWM 输出

    int16_t brightness = 0;
    int8_t  dir = 1;

    while (1) {
        __HAL_TIM_SET_COMPARE(&htim3, TIM_CHANNEL_1, brightness);
        brightness += dir;
        if (brightness >= 999) dir = -1;
        if (brightness <= 0)   dir = 1;
        HAL_Delay(5);
    }
}
```

### 舵机控制（TIM3_CH3）
```c
// 500-2500us → 0-180°
void Servo_SetAngle(TIM_HandleTypeDef *htim, uint8_t angle) {
    if (angle > 180) angle = 180;
    uint16_t pulse = 500 + (uint32_t)2000 * angle / 180;
    __HAL_TIM_SET_COMPARE(htim, TIM_CHANNEL_3, pulse);
}

int main(void) {
    // ... MX_TIM3_Init ...
    HAL_TIM_PWM_Start(&htim3, TIM_CHANNEL_3);

    Servo_SetAngle(&htim3, 0);   HAL_Delay(1000);
    Servo_SetAngle(&htim3, 90);  HAL_Delay(1000);
    Servo_SetAngle(&htim3, 180); HAL_Delay(1000);
    while (1);
}
```

### 定时中断（TIM2, 1ms）
```c
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim) {
    if (htim->Instance == TIM2) {
        // 每 1ms 执行一次
        static uint32_t cnt = 0;
        if (++cnt >= 1000) {  // 1 秒
            cnt = 0;
            HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
        }
    }
}
// 启动：HAL_TIM_Base_Start_IT(&htim2); 放在 main 里 MX_TIM2_Init() 之后
```

---

## 注意事项
- ⚠️ CubeMX 里 Prescaler 和 Counter Period 填 **-1** 后的值（如 84-1=83），不要少减
- ⚠️ PWM 用 `__HAL_TIM_SET_COMPARE` 动态改占空比，比重新配置快得多
- ⚠️ 舵机 ARR=20000-1=19999（50Hz），占空比范围 500-2500
- ⚠️ 定时中断 + CubeMX：NVIC 不勾 = `HAL_TIM_Base_Start_IT` 也不会进中断
- ⚠️ 高级定时器 TIM1/8 的互补输出在 CubeMX 里选 `PWM Generation CHxN`
