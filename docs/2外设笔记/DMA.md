# DMA — 直接存储器访问

> 芯片：STM32F4 | DMA1（APB1 外设）/ DMA2（APB2 外设）| CPU 不参与数据搬运

## 原理

DMA 是硬件数据搬运工——CPU 配置一次，DMA 自动在外设和内存之间搬数据，搬完通知 CPU。

### F4 DMA 架构
```
DMA1: 8 Stream × 8 Channel → APB1 外设（UART2-5、SPI2-3、I2C）
DMA2: 8 Stream × 8 Channel → APB2 外设（UART1/6、SPI1、ADC）+ M2M
```
- 每个 Stream 只能选 1 个 Channel，一个 Channel 只对应特定外设
- 方向：外设→内存（RX）或 内存→外设（TX）

---

## CubeMX 配置

以 **USART1 TX DMA + USART1 RX DMA** + **ADC1 DMA** 为例。

### USART1 收发 DMA

| 步骤 | 参数 | 值 |
|------|------|-----|
| CubeMX 先配好 **USART1**（Asynchronous 模式） | — | — |
| **USART1 → DMA Settings** | Add → `USART1_RX` | — |
| | Direction | **Peripheral To Memory** |
| | Mode | **Circular**（循环接收） |
| | Priority | **High** |
| | Add → `USART1_TX` | — |
| | Direction | **Memory To Peripheral** |
| | Mode | **Normal**（发完即停） |
| | Priority | **Low** |

### ADC1 DMA

| 步骤 | 参数 | 值 |
|------|------|-----|
| CubeMX 先配好 **ADC1**（Continuous + Scan 多通道） | — | — |
| **ADC1 → DMA Settings** | Add → `ADC1` | — |
| | Direction | **Peripheral To Memory** |
| | Mode | **Circular** |
| | Data Width | **Half Word** |
| | Priority | **High** |

> CubeMX 自动分配 DMA Stream/Channel，NVIC 里 DMA 中断也自动使能

---

## CubeMX 自动生成代码

```c
// dma.c — MX_DMA_Init()
static void MX_DMA_Init(void) {
    // USART1 RX: DMA2_Stream2, CH4
    __HAL_RCC_DMA2_CLK_ENABLE();
    hdma_usart1_rx.Instance = DMA2_Stream2;
    hdma_usart1_rx.Init.Channel = DMA_CHANNEL_4;
    hdma_usart1_rx.Init.Direction = DMA_PERIPH_TO_MEMORY;
    hdma_usart1_rx.Init.Mode = DMA_CIRCULAR;
    // ...
    __HAL_LINKDMA(&huart1, hdmarx, hdma_usart1_rx);

    // ADC1: DMA2_Stream0, CH0
    hdma_adc1.Instance = DMA2_Stream0;
    hdma_adc1.Init.Channel = DMA_CHANNEL_0;
    hdma_adc1.Init.Direction = DMA_PERIPH_TO_MEMORY;
    hdma_adc1.Init.Mode = DMA_CIRCULAR;
    // ...
    __HAL_LINKDMA(&hadc1, DMA_Handle, hdma_adc1);
}
// CubeMX 替你做完了 Stream/Channel 映射和 __HAL_LINKDMA，一个都不用手写
```

---

## 手写业务代码

CubeMX 生成 `MX_DMA_Init()` 后，直接用 HAL 的 DMA 版 API：

| API | 说明 |
|-----|------|
| `HAL_UART_Transmit_DMA(&huart1, buf, len)` | DMA 发送 |
| `HAL_UART_Receive_DMA(&huart1, buf, len)` | DMA 接收 |
| `HAL_ADC_Start_DMA(&hadc1, (uint32_t*)buf, len)` | ADC DMA 采集 |

```c
// UART DMA 发送 + 接收
uint8_t tx_buf[] = "DMA Test\r\n";
uint8_t rx_buf[64];

int main(void) {
    HAL_Init();
    SystemClock_Config();
    MX_GPIO_Init();
    MX_DMA_Init();          // CubeMX 生成
    MX_USART1_UART_Init();  // CubeMX 生成（已绑定 DMA）

    HAL_UART_Transmit_DMA(&huart1, tx_buf, sizeof(tx_buf) - 1);
    HAL_UART_Receive_DMA(&huart1, rx_buf, sizeof(rx_buf));

    while (1);  // DMA 后台工作
}

// DMA 发送完成回调
void HAL_UART_TxCpltCallback(UART_HandleTypeDef *huart) {
    if (huart->Instance == USART1) {
        printf("DMA TX done!\r\n");
    }
}

// DMA 接收半满/全满回调
void HAL_UART_RxHalfCpltCallback(UART_HandleTypeDef *huart) {
    // rx_buf 前一半满了，可以处理
}
```

---

## 注意事项
- ⚠️ CubeMX 里 DMA 的 Stream+Channel**自动分配合适的**，不需要背映射表
- ⚠️ RX 用 **Circular** 模式才持续接收；TX 用 **Normal** 发完一次停
- ⚠️ ADC DMA 记得 `Continuous DMA Requests` 选 **Enabled**
- ⚠️ DMA 传输期间**不要碰缓冲区**——RX 在 Circular 模式下手动读要注意半满/全满回调
- ⚠️ 画 CubeMX 图时 `System Core → DMA` 可以查看所有已配置的 DMA 流
