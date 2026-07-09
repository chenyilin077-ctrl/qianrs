# NVIC & EXTI — 中断系统

> 芯片：STM32F4 | Cortex-M4 | 优先级分组 + 边沿触发

## 原理

### 中断处理链
```
外设/GPIO 触发 → EXTI 边沿检测 → NVIC 优先级排队 → CPU 跳转 ISR
```

### 优先级
- 抢占优先级（Preemption）：数值越小越高，可打断低优先级的
- 子优先级（Subpriority）：抢占相同时同时触发，数字小的先响应
- CubeMX 默认分组 `NVIC_PRIORITYGROUP_4`（16 级全抢占）

### EXTI 线
- 16 条 GPIO 线：EXTI0~15 对应 PIN_0~PIN_15
- ⚠️ 同一编号只能选 1 个 GPIO：PA0 和 PB0 不能同时做 EXTI0

---

## CubeMX 配置

### 按键中断（PA0 = KEY，下降沿触发）

| 步骤 | 参数 | 值 |
|------|------|-----|
| **Pinout** | PA0 | 选 **GPIO_EXTI0** |
| | 右键 → `Enter User Label` | `KEY` |
| **Configuration → GPIO** | GPIO mode | **External Interrupt Mode with Falling edge trigger** |
| | GPIO Pull-up/Pull-down | **Pull-up** |
| **NVIC** 选项卡 | 勾选 **EXTI line0 interrupt** | — |
| | Preemption Priority | 2（或任意） |

### UART IDLE 中断（不定长接收）

| 步骤 | 参数 | 值 |
|------|------|-----|
| USART1 配好 Asynchronous + DMA RX（Circular） | — | — |
| **NVIC** 选项卡 | 自动勾选 USART1 + DMA | 不额外操作 |
| 手写代码部分 | `__HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE)` | CubeMX 不自动配 IDLE |

---

## CubeMX 自动生成代码

```c
// gpio.c — MX_GPIO_Init()
// PA0 自动配为 EXTI0，CubeMX 生成：
HAL_NVIC_SetPriority(EXTI0_IRQn, 2, 0);
HAL_NVIC_EnableIRQ(EXTI0_IRQn);

// stm32f4xx_it.c — CubeMX 自动生成中断入口
void EXTI0_IRQHandler(void) {
    HAL_GPIO_EXTI_IRQHandler(KEY_Pin);
}
```

---

## 手写业务代码

### 按键中断 + 软件去抖

```c
volatile uint32_t last_tick = 0;

// CubeMX 生成 EXTI0_IRQHandler → HAL_GPIO_EXTI_IRQHandler → 下面这个回调
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) {
    if (GPIO_Pin == KEY_Pin) {
        uint32_t now = HAL_GetTick();
        if (now - last_tick > 50) {  // 50ms 去抖
            last_tick = now;
            HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
        }
    }
}
```

### UART IDLE + DMA（不定长帧接收）

CubeMX 已经生成 `USART1_IRQHandler`，在 `stm32f4xx_it.c` 里找到它，手动加 IDLE 处理：

```c
// stm32f4xx_it.c — USART1_IRQHandler（CubeMX 已生成框架，往里加）
void USART1_IRQHandler(void) {
    HAL_UART_IRQHandler(&huart1);  // CubeMX 已有的

    // 下面这段手写：
    if (__HAL_UART_GET_FLAG(&huart1, UART_FLAG_IDLE)) {
        __HAL_UART_CLEAR_IDLEFLAG(&huart1);
        HAL_UART_DMAStop(&huart1);
        uint16_t rx_len = RX_BUF_SIZE - __HAL_DMA_GET_COUNTER(&hdma_usart1_rx);
        printf("Got %d bytes: %.*s\r\n", rx_len, rx_len, rx_buf);
        HAL_UART_Receive_DMA(&huart1, rx_buf, RX_BUF_SIZE);  // 重启
    }
}

// main.c — 初始化
int main(void) {
    // ... Init ...
    __HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE);           // 开启 IDLE 中断
    HAL_UART_Receive_DMA(&huart1, rx_buf, RX_BUF_SIZE);    // 启动 DMA
    while (1);
}
```

---

## CubeMX NVIC 配置一览

| 中断源 | NVIC 勾选 | 常见优先级 | 用途 |
|--------|----------|-----------|------|
| EXTI line0-4 | 每个单独勾 | 2 | 独立按键 |
| EXTI line[9:5] | 一条线对应 PIN5-9 | 2 | 多按键共享 |
| USART1 | 勾选 | 3 | 串口收发 |
| DMA1/2 stream | CubeMX 自动勾 | 看情况 | DMA 完成通知 |
| TIM2 | 勾选 | 1 | 定时中断 |
| SysTick | CubeMX 默认配 | 0 | HAL 时基 |

---

## 注意事项
- ⚠️ EXTI 同一编号不能同时选多个 GPIO——CubeMX 里 PA0 和 PB0 不能都配 EXTI0
- ⚠️ 中断回调用时尽量短，耗时操作放主循环用标志位触发
- ⚠️ 按键中断必须软件去抖 50ms，硬件抖动一次触发几次
- ⚠️ IDLE 中断不会在 CubeMX 里自动开启，必须手写 `__HAL_UART_ENABLE_IT(..., UART_IT_IDLE)`
- ⚠️ 中断函数名必须和 `startup_stm32f4xx.s` 一致，CubeMX 产生的固件库已经配好
