# GPIO — 通用输入输出

> 芯片：STM32F4 系列 | 总线：AHB1

## 原理

### GPIO 是 MCU 与外部世界交互的最基本接口
每个 GPIO 引脚可配置为 8 种模式：

| 模式 | 用途 | 典型场景 |
|------|------|----------|
| 推挽输出 (Push-Pull) | 输出 0/1，驱动能力强 | 点 LED、控制继电器 |
| 开漏输出 (Open-Drain) | 只能拉低，拉高靠外部上拉 | I2C、电平转换 |
| 上拉输入 (Pull-Up) | 默认高电平 | 按键检测（按下为低） |
| 下拉输入 (Pull-Down) | 默认低电平 | 按键检测（按下为高） |
| 浮空输入 (No Pull) | 高阻态 | ADC 输入 |
| 模拟模式 (Analog) | 模拟信号 | ADC/DAC |
| 复用推挽 (AF PP) | 外设功能输出 | UART TX、SPI MOSI |
| 复用开漏 (AF OD) | 外设功能开漏 | I2C SDA |

### 输出速度
CubeMX 可选 Low / Medium / High / Very High。一般 GPIO 用 Low，SPI 等高速接口用 High。

### BSRR 寄存器（原子操作）
`GPIOx->BSRR`：低 16 位置位（设 1），高 16 位复位（设 0）。一条指令同时改多个引脚，不会被中断打断。

---

## CubeMX 配置

### LED 输出（PA0 = LED）

| 步骤 | 操作 |
|------|------|
| **Pinout** | 点击 PA0 → 选 `GPIO_Output` |
| **GPIO 标签** | 右键 PA0 → `Enter User Label` → 输入 `LED` |
| **Configuration → GPIO** | 点 PA0 那一行 |
| GPIO output level | **Low**（初始灭） |
| GPIO mode | **Output Push Pull** |
| GPIO Pull-up/Pull-down | **No pull-up and no pull-down** |
| Maximum output speed | **Low** |

### 按键输入（PB0 = KEY，上拉）

| 步骤 | 操作 |
|------|------|
| **Pinout** | 点击 PB0 → 选 `GPIO_Input` |
| **GPIO 标签** | 右键 PB0 → `Enter User Label` → 输入 `KEY` |
| **Configuration → GPIO** | 点 PB0 那一行 |
| GPIO mode | **Input mode**（无需改） |
| GPIO Pull-up/Pull-down | **Pull-up**（未按下时读高电平） |

### Clock Configuration 确认
- AHB1 bus 已自动使能（使能了 GPIO 引脚就自动使能总线时钟）

### 生成代码
`Project → Generate Code`，CubeMX 在 `gpio.c` 里自动生成：

```c
// MX_GPIO_Init() — CubeMX 自动生成，不要手动改
static void MX_GPIO_Init(void) {
    GPIO_InitTypeDef GPIO_InitStruct = {0};
    __HAL_RCC_GPIOA_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();

    GPIO_InitStruct.Pin = GPIO_PIN_0;
    GPIO_InitStruct.Mode = GPIO_MODE_OUTPUT_PP;
    GPIO_InitStruct.Pull = GPIO_NOPULL;
    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
    HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);

    GPIO_InitStruct.Pin = GPIO_PIN_0;
    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
    GPIO_InitStruct.Pull = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOB, &GPIO_InitStruct);
}
```

---

## 常用 API（手写代码部分）

| 函数                                                          | 作用                |
| ----------------------------------------------------------- | ----------------- |
| `HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET)`   | 点亮 LED            |
| `HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_RESET)` | 熄灭 LED            |
| `HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin)`                | 翻转 LED            |
| `HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin)`                  | 读按键（返回 SET/RESET） |

> CubeMX 勾选 `Generate peripheral initialization as a pair of .c/.h files` 后，每个引脚自动生成宏：`LED_Pin`、`LED_GPIO_Port`、`KEY_Pin`、`KEY_GPIO_Port`

---

## 示例代码（写在 `main.c` 的 `while(1)` 里）

```c
// CubeMX 已生成 MX_GPIO_Init()，只需在 while(1) 里写应用逻辑

while (1) {
    // 方式 1：按键按下（低电平）→ LED 亮
    if (HAL_GPIO_ReadPin(KEY_GPIO_Port, KEY_Pin) == GPIO_PIN_RESET) {
        HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_SET);
    } else {
        HAL_GPIO_WritePin(LED_GPIO_Port, LED_Pin, GPIO_PIN_RESET);
    }

    // 方式 2：每 500ms 翻转（HAL_Delay 由 CubeMX 的 SysTick 提供）
    // HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
    // HAL_Delay(500);
}
```

### 进阶：BSRR 原子操作
```c
// 同时置位 PA0（亮 LED），复位 PA1 — 一条指令完成
GPIOA->BSRR = LED_Pin | (GPIO_PIN_1 << 16);
```

---

## 注意事项
- ⚠️ CubeMX 选中引脚后总线时钟自动使能，不用手动写 `__HAL_RCC_xxx_CLK_ENABLE()`
- ⚠️ 勾选 User Label 后会自动生成 `xxx_Pin` 和 `xxx_GPIO_Port` 宏，代码更好读
- ⚠️ `HAL_GPIO_WritePin` 内部操作 ODR，非原子；多任务场景用 `BSRR`
- ⚠️ 5V 耐受：F4 部分引脚标有 `FT`（5V tolerant），3.3V 接 5V 必须确认
- ⚠️ 复用功能（UART/SPI/I2C）在 CubeMX 里选对应外设即可自动配置，不要手动设 AF
