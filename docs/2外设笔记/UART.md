# UART / USART — 通用异步收发

> 芯片：STM32F4 | 总线：APB1（USART2-5）/ APB2（USART1、USART6）

## 原理

### 异步串行通信
UART 通过 **TX（发送）** 和 **RX（接收）** 两根线实现全双工，不需要时钟线。

### 数据帧格式
```
┌─ 起始位 ─┬── 数据位（5-9）──┬─ 校验位 ─┬─ 停止位 ─┐
│  1 bit 0  │  LSB ... MSB     │ 可选1bit │ 0.5/1/2bit│
└───────────┴──────────────────┴──────────┴──────────┘
```
- **波特率**：常用 9600、115200
- **起始位**：1 低电平 → 数据位 8bit → 停止位 1bit（最常用 8N1）

### TTL vs USB
STM32 引脚输出 3.3V TTL，接电脑需要 CH340 / CP2102 等 USB-TTL 芯片。

---

## CubeMX 配置

以 **USART1：PA9=TX, PA10=RX, 115200, 8N1** 为例。

### Pinout
| 步骤 | 操作 |
|------|------|
| **Connectivity** → **USART1** | Mode 选 **Asynchronous** |
| PA9 自动设为 USART1_TX, PA10 自动设为 USART1_RX | — |
| 右键 PA9/PA10 → `Enter User Label` | 输入 `USART1_TX` / `USART1_RX` |

### Configuration → USART1
| 参数 | 值 |
|------|-----|
| Baud Rate | **115200 Bits/s** |
| Word Length | **8 Bits (including Parity)** |
| Parity | **None** |
| Stop Bits | **1** |
| Over Sampling | **16 Samples** |

### NVIC（如需中断）
| 步骤 | 操作 |
|------|------|
| **NVIC Settings** 选项卡 | 勾选 **USART1 global interrupt** |

### DMA（如需 DMA）
| 步骤 | 操作 |
|------|------|
| **DMA Settings** 选项卡 | 点 **Add** → 选 `USART1_RX` / `USART1_TX` |
| Mode | RX 选 **Circular**（循环），TX 选 **Normal** |
| Priority | **High**（RX）/ **Low**（TX） |

> ⚠️ 选 DMA 后 NVIC 里 DMA 中断自动使能，不用额外勾选

---

## CubeMX 自动生成的代码

```c
// usart.c — MX_USART1_UART_Init()
static void MX_USART1_UART_Init(void) {
    huart1.Instance = USART1;
    huart1.Init.BaudRate = 115200;
    huart1.Init.WordLength = UART_WORDLENGTH_8B;
    huart1.Init.StopBits = UART_STOPBITS_1;
    huart1.Init.Parity = UART_PARITY_NONE;
    huart1.Init.Mode = UART_MODE_TX_RX;
    huart1.Init.HwFlowCtl = UART_HWCONTROL_NONE;
    huart1.Init.OverSampling = UART_OVERSAMPLING_16;
    HAL_UART_Init(&huart1);
}
// 选 DMA 后还会自动生成 DMA 绑定代码
```

---

## 常用 API（手写代码部分）

| 函数 | 模式 | 说明 |
|------|------|------|
| `HAL_UART_Transmit(&huart1, buf, len, timeout)` | 阻塞 | 发送 |
| `HAL_UART_Receive(&huart1, buf, len, timeout)` | 阻塞 | 接收 |
| `HAL_UART_Transmit_IT(&huart1, buf, len)` | 中断 | 非阻塞发送 |
| `HAL_UART_Receive_IT(&huart1, buf, len)` | 中断 | 非阻塞接收 |
| `HAL_UART_Transmit_DMA(&huart1, buf, len)` | DMA | DMA 发送 |
| `HAL_UART_Receive_DMA(&huart1, buf, len)` | DMA | DMA 接收 |

---

## 示例 1：printf 重定向

在 `main.c` 添加（CubeMX 不生成这段）：

```c
#include <stdio.h>

// 重定向 printf → USART1
int fputc(int ch, FILE *f) {
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}

int main(void) {
    // ... HAL_Init, SystemClock_Config, MX_USART1_UART_Init ...
    printf("Hello STM32! SYSCLK = %lu MHz\r\n",
           HAL_RCC_GetSysClockFreq() / 1000000);
    while (1);
}
```

## 示例 2：中断接收回显

CubeMX 配置：USART1 → NVIC 勾选 `USART1 global interrupt`

```c
uint8_t rx_byte;

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_USART1_UART_Init();

    HAL_UART_Receive_IT(&huart1, &rx_byte, 1);  // 启动中断接收
    while (1);  // 工作都在回调里做
}

// 接收完成回调（重写 HAL 虚函数）
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart->Instance == USART1) {
        HAL_UART_Transmit(&huart1, &rx_byte, 1, 10); // 回显
        HAL_UART_Receive_IT(&huart1, &rx_byte, 1);   // 再次启动
    }
}
```

## 示例 3：IDLE 中断 + DMA 不定长接收

CubeMX 配置：USART1 → DMA 添加 RX（Circular）→ NVIC 勾选 USART1

```c
#define RX_BUF_SIZE 128
uint8_t rx_buf[RX_BUF_SIZE];

int main(void) {
    // ... Init ...
    __HAL_UART_ENABLE_IT(&huart1, UART_IT_IDLE);           // 开启 IDLE 中断
    HAL_UART_Receive_DMA(&huart1, rx_buf, RX_BUF_SIZE);    // 启动 DMA 接收
    while (1);
}

void USART1_IRQHandler(void) {
    HAL_UART_IRQHandler(&huart1);
    if (__HAL_UART_GET_FLAG(&huart1, UART_FLAG_IDLE)) {
        __HAL_UART_CLEAR_IDLEFLAG(&huart1);
        HAL_UART_DMAStop(&huart1);
        uint16_t len = RX_BUF_SIZE - __HAL_DMA_GET_COUNTER(&hdma_usart1_rx);
        // 处理 rx_buf[0..len-1] ...
        HAL_UART_Receive_DMA(&huart1, rx_buf, RX_BUF_SIZE); // 重启
    }
}
```

---

## 注意事项
- ⚠️ TTL 是 3.3V，必须用 USB-TTL 模块（CH340 等）才能接电脑
- ⚠️ 中断接收回调结尾必须**再次调用** `HAL_UART_Receive_IT`，只收一次就停了
- ⚠️ CubeMX 里 NVIC 不勾 = 不进中断服务函数，别漏了
- ⚠️ CubeMX 默认只用 TX/RX 两根线，CTS/RTS 流控一般不需要
