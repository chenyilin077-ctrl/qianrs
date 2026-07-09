# RCC — 时钟系统

> 芯片：STM32F4 | HSE 8MHz → PLL → 168MHz SYSCLK

## 原理

### 时钟源
| 时钟 | 频率 | 用途 |
|------|------|------|
| **HSI** | 16MHz 内部 RC | 默认启动，精度±1% |
| **HSE** | 8-25MHz 外部晶振 | 精确 PLL 输入 |
| **PLL** | 最高 168MHz | 系统主时钟 |
| **LSI** | 32kHz 内部 RC | 独立看门狗 |
| **LSE** | 32.768kHz 外部晶振 | RTC 实时时钟 |

### 经典 PLL 链路（HSE 8MHz → 168MHz）
```
HSE(8MHz) / M(8) = 1MHz → × N(336) = 336MHz → / P(2) = 168MHz (SYSCLK)
                                              → / Q(7) = 48MHz (USB)
```

### 总线分频
```
SYSCLK(168) → AHB  /1 → 168MHz (HCLK, 内核/内存/DMA)
            → APB1 /4 →  42MHz (TIM2-7, USART2-5, SPI2-3, I2C)
            → APB2 /2 →  84MHz (TIM1/8, USART1/6, SPI1, ADC)
```

> ⚡ APB1 定时器时钟 = APB1×2 = 84MHz（因为 APB1 分频 ≠ 1）

---

## CubeMX 配置

### Clock Configuration 页面

CubeMX 打开 `.ioc` → **Clock Configuration** 标签页，图形化拖拽：

| 步骤 | 值 | 说明 |
|------|-----|------|
| **HSE** | 选 **Crystal/Ceramic Resonator** | 使能外部晶振 |
| Input frequency | **8 MHz** | 填入实际晶振频率 |
| **PLL Source Mux** | 选 **HSE** | PLL 时钟源 |
| **/M** | **8** | HSE/8 = 1MHz |
| **×N** | **336** | 1MHz × 336 = 336MHz VCO |
| **/P** | **2** | 336/2 = 168MHz → SYSCLK |
| **/Q** | **7** | 336/7 = 48MHz → USB |
| **System Clock Mux** | 选 **PLLCLK** | 用 PLL 当主时钟 |
| **AHB Prescaler** | **/1** | HCLK = 168MHz |
| **APB1 Prescaler** | **/4** | PCLK1 = 42MHz |
| **APB2 Prescaler** | **/2** | PCLK2 = 84MHz |

CubeMX 自动检查频率是否超限——**红字 = 超频，蓝字 = 正常**。看到红色的把分频调大。

> 提示：直接拖右侧的 HCLK 滑块到 168，CubeMX 会自动帮你算好 M/N/P/Q！

---

## CubeMX 自动生成代码

```c
// main.c — SystemClock_Config()
void SystemClock_Config(void) {
    RCC_OscInitTypeDef RCC_OscInitStruct = {0};
    RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

    RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSE;
    RCC_OscInitStruct.HSEState = RCC_HSE_ON;
    RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
    RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSE;
    RCC_OscInitStruct.PLL.PLLM = 8;
    RCC_OscInitStruct.PLL.PLLN = 336;
    RCC_OscInitStruct.PLL.PLLP = 2;
    RCC_OscInitStruct.PLL.PLLQ = 7;
    HAL_RCC_OscConfig(&RCC_OscInitStruct);

    RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK | RCC_CLOCKTYPE_SYSCLK
                                | RCC_CLOCKTYPE_PCLK1 | RCC_CLOCKTYPE_PCLK2;
    RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
    RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
    RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV4;
    RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV2;
    HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_5);
}
// Flash Latency: CubeMX 根据频率自动选 FLASH_LATENCY_5（168MHz 需要 5WS）
```

---

## 时钟验证方式

### 方式 1：串口打印（推荐）
```c
printf("SYSCLK = %lu MHz\r\n", HAL_RCC_GetSysClockFreq() / 1000000);
printf("HCLK   = %lu MHz\r\n", HAL_RCC_GetHCLKFreq()    / 1000000);
printf("PCLK1  = %lu MHz\r\n", HAL_RCC_GetPCLK1Freq()   / 1000000);
printf("PCLK2  = %lu MHz\r\n", HAL_RCC_GetPCLK2Freq()   / 1000000);
// 输出: 168 / 168 / 42 / 84
```

### 方式 2：MCO 输出用示波器看
```
CubeMX → RCC → Master Clock Output 1 → 选 HSE
Pinout → PA8 → 自动设为 RCC_MCO
示波器接到 PA8 → 看到 8MHz 正弦波 = HSE 工作正常
```

### 方式 3：LED 1 秒闪（肉眼）
```c
HAL_Delay(1000);  // 1 秒亮灭一次，不准就说明时钟配置有问题
HAL_GPIO_TogglePin(LED_GPIO_Port, LED_Pin);
```

---

## 注意事项
- ⚠️ CubeMX Clock Configuration 里红字 = 超频，必须调回来，蓝字才安全
- ⚠️ APB1 定时器时钟翻倍：CubeMX Clock 图上 TIMx clock 显示的是 84MHz（不是 42MHz）
- ⚠️ FLASH_LATENCY 和频率严格匹配，CubeMX 自动算不用手动改
- ⚠️ 改了 Clock Configuration 后 CubeMX 只重写 `SystemClock_Config()`，不改其他文件
- ⚠️ 没画外部晶振焊 8MHz 就 HSI 启动——CubeMX 默认 HSI，不改也能跑 16MHz
